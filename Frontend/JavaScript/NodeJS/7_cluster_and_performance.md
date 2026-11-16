Node 单进程只能用一个 CPU 核心，一旦遇到 CPU 密集任务，再多的并发请求也只能排队。要榨干多核机器，得先分清"I/O 密集"和"CPU 密集"，再选择 cluster 或 worker_threads。本篇讲清楚两条路线、负载均衡方式，以及如何定位内存与 CPU 问题。

# 1. 为什么单进程不够

## 1.1 I/O 与 CPU 的瓶颈

```text
I/O 密集:  等数据库、等网络、等磁盘
           → 事件循环足够,单进程可支撑大量并发

CPU 密集:  加密、压缩、图像处理、大计算
           → 主线程被占满,事件循环停摆,其他请求全部排队
```

★ **关键认知**：多开进程**不会让单个 CPU 密集任务变快**，它只是让多个任务能同时跑在不同核心上。如果单个请求就要算 10 秒，cluster 只能让 4 个这样的请求并行，不能把 10 秒变成 2.5 秒。

## 1.2 多核利用的两条路

| 方案 | 隔离级别 | 通信成本 | 适用场景 |
|------|---------|---------|---------|
| `cluster` | 进程级 | 较高，需 IPC | 扩展 HTTP 服务，利用多核 |
| `worker_threads` | 线程级 | 较低，可共享内存 | 单个服务内的 CPU 密集任务 |

# 2. cluster 模块

## 2.1 工作原理

`cluster` 会 fork 出多个子进程，每个子进程都是独立的 Node 实例，各自有独立的内存与事件循环。主进程负责创建、监控和重启它们，并可以选择由谁来接收新连接。

```text
        ┌──────────────────┐
        │   primary 进程    │  监听端口、管理 worker
        └────────┬─────────┘
      ┌──────────┼──────────┐
      ▼          ▼          ▼
  ┌───────┐  ┌───────┐  ┌───────┐
  │worker1│  │worker2│  │worker3│
  └───────┘  └───────┘  └───────┘
      各自独立的事件循环与内存
```

★ **关键认知**：cluster 的 worker **不共享内存**。在主进程里定义的变量、缓存、连接池，worker 各自有一份。⚠ 用内存存会话或限流计数时，cluster 下每个 worker 各算各的，总数会不准。

## 2.2 最小示例

```javascript
// server.js
import cluster from "node:cluster";
import http from "node:http";
import os from "node:os";

const workers = os.availableParallelism?.() ?? os.cpus().length;

if (cluster.isPrimary) {
  console.log(`主进程 ${process.pid},启动 ${workers} 个 worker`);

  for (let i = 0; i < workers; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker, code, signal) => {
    console.error(`worker ${worker.process.pid} 退出,重启`);
    cluster.fork(); // 简单重启策略
  });
} else {
  http.createServer((req, res) => {
    res.end(`处理进程 ${process.pid}\n`);
  }).listen(3000);

  console.log(`worker ${process.pid} 已启动`);
}
```

运行后反复请求，会看到响应里的 PID 在不同 worker 之间轮换。

## 2.3 负载均衡策略

Node 在非 Windows 平台上默认使用**轮询(round-robin)**：主进程接受连接，再分发给 worker。

```text
连接 ──► primary ──► worker1
                  ├─► worker2
                  └─► worker3
```

Windows 上默认使用操作系统调度：worker 自己竞争接受连接，可能出现分配不均。

也可以手动控制：

```javascript
cluster.schedulingPolicy = cluster.SCHED_RR;  // 轮询
cluster.schedulingPolicy = cluster.SCHED_NONE; // 交给操作系统
```

★ **关键认知**：轮询的优点是**分配均匀**，缺点是主进程要参与每个连接的接收，略增开销。操作系统的竞争调度吞吐可能更高，但短连接多时容易倾斜。⚠ 如果按连接分发，长连接(如 WebSocket)会让某些 worker 长期更忙，必要时改用按请求分发或独立网关。

## 2.4 零停机重启

```javascript
if (cluster.isPrimary) {
  for (let i = 0; i < workers; i++) cluster.fork();

  process.on("SIGUSR2", () => {
    const oldWorkers = Object.values(cluster.workers);
    let remaining = oldWorkers.length;

    for (const worker of oldWorkers) {
      const replacement = cluster.fork(); // 先起新的
      replacement.on("listening", () => {
        worker.disconnect();             // 再断旧的
      });
      worker.on("exit", () => {
        if (--remaining === 0) console.log("滚动重启完成");
      });
    }
  });
}
```

流程是"先起新、再停旧"，保证任意时刻都有 worker 在服务。生产环境通常交给 PM2 或容器编排完成。

## 2.5 进程管理与重启

- 区分**正常退出**与**异常退出**，异常退出要限速重启，避免崩溃循环；
- 记录 worker 的退出码与信号；
- 关闭时先停止接收新连接，再等待存量请求完成；
- 用 `disconnect()` 而非直接 `kill()` 来优雅下线。

# 3. worker_threads

## 3.1 与 cluster 的区别

| 维度 | cluster | worker_threads |
|------|---------|----------------|
| 隔离 | 独立进程 | 同一进程内的线程 |
| 内存 | 完全隔离 | 可通过 `SharedArrayBuffer` 共享 |
| 启动开销 | 较高 | 较低 |
| 通信 | IPC，结构化克隆 | `postMessage`，结构化克隆 |
| 崩溃影响 | 不影响其他进程 | 未捕获异常可能影响整个进程 |

## 3.2 最小示例

`worker.js`：

```javascript
import { parentPort } from "node:worker_threads";

parentPort.on("message", (n) => {
  let sum = 0;
  for (let i = 0; i < n; i++) sum += i;
  parentPort.postMessage(sum);
});
```

`main.js`：

```javascript
import { Worker } from "node:worker_threads";

function runTask(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL("./worker.js", import.meta.url));

    worker.once("message", resolve);
    worker.once("error", reject);
    worker.once("exit", (code) => {
      if (code !== 0) reject(new Error(`worker 退出码 ${code}`));
    });

    worker.postMessage(n);
  });
}

console.log(await runTask(1e9)); // 主线程保持可响应
```

★ **关键认知**：CPU 密集任务应放进 worker，主线程只负责接收请求和调度。⚠ 每次任务都新建 worker 开销不小，高频场景要用**线程池**复用 worker，而不是来一个任务 `new Worker` 一次。

## 3.3 共享内存

```javascript
// 主线程
const shared = new SharedArrayBuffer(4);
const view = new Int32Array(shared);

const worker = new Worker(new URL("./counter.js", import.meta.url), {
  workerData: shared,
});

Atomics.add(view, 0, 1); // 原子加一
```

`SharedArrayBuffer` 能在多个线程间共享字节，`Atomics` 保证读写原子性。⚠ 共享内存需要自己处理同步，复杂逻辑容易写出竞态；大多数场景用 `postMessage` 传数据更安全。

# 4. 内存与性能分析

## 4.1 内存指标

```javascript
const usage = process.memoryUsage();

console.log({
  rss: `${(usage.rss / 1024 / 1024).toFixed(1)} MB`,       // 进程总内存
  heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(1)} MB`, // V8 堆总量
  heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(1)} MB`,   // V8 已用堆
  external: `${(usage.external / 1024 / 1024).toFixed(1)} MB`,   // C++ 对象
});
```

| 指标 | 含义 | 关注点 |
|------|------|--------|
| `rss` | 常驻内存 | 持续上涨可能泄漏 |
| `heapUsed` | JS 堆已用 | 与 `heapTotal` 差距过大需关注 |
| `external` | Buffer 等外部内存 | 大量 Buffer 时上涨 |
| `arrayBuffers` | ArrayBuffer 占用 | 流与二进制处理相关 |

★ **关键认知**：`heapUsed` 高不一定是泄漏，可能只是垃圾还没回收。判断泄漏要看**多次 GC 后仍然持续上涨**的趋势，而不是某一时刻的绝对值。

## 4.2 CPU 密集任务的处理

```javascript
// 错误:在主线程里做重计算
app.get("/hash", (req, res) => {
  const result = heavyHash(req.query.input); // 阻塞所有请求
  res.json({ result });
});
```

正确做法：

```javascript
app.get("/hash", async (req, res) => {
  const result = await pool.run(req.query.input); // 交给 worker 线程池
  res.json({ result });
});
```

## 4.3 性能分析工具

| 工具 | 用途 |
|------|------|
| `node --inspect` | 连接 Chrome DevTools 调试与采样 |
| `node --cpu-prof` | 生成 CPU profile，分析热点函数 |
| `node --heap-prof` | 生成堆分配 profile |
| `clinic` | 一体化诊断 CPU、内存、事件循环 |
| `0x` | 生成火焰图，定位 CPU 热点 |
| `process.hrtime.bigint()` | 高精度计时，定位慢操作 |

```bash
node --cpu-prof --cpu-prof-dir=./profiles server.js
```

运行一段时间后，在 Chrome DevTools 的 Performance 面板加载生成的 `.cpuprofile`，查看火焰图。

## 4.4 事件循环延迟

```javascript
import { monitorEventLoopDelay } from "node:perf_hooks";

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  console.log("事件循环延迟 p99(ms)", histogram.percentile(99) / 1e6);
}, 5000);
```

★ **关键认知**：事件循环延迟是 Node 服务最值得监控的指标之一。它直接反映"主线程有多忙"：延迟升高说明有同步阻塞或 CPU 密集任务，此时加机器只能缓解，根治要靠拆任务或上 worker。

# 5. 常见错误与练习

## 5.1 常见错误

**错误一：在 cluster 里用内存做限流**

```javascript
const counts = new Map(); // 每个 worker 各一份,总数不准
```

应改用 Redis 等共享存储，或由网关统一限流。

**错误二：worker 崩溃无人处理**

```javascript
const worker = new Worker("./worker.js");
// 没有监听 error / exit,崩溃后任务永远不返回
```

必须监听 `error` 与 `exit`，并给任务加超时。

**错误三：无限制地创建 worker**

```javascript
for (const task of tasks) {
  new Worker("./worker.js"); // 瞬间创建上千线程,反而更慢
}
```

应使用固定大小的线程池。

**错误四：只看内存绝对值判断泄漏**

刚跑完大批量任务，`heapUsed` 高是正常的。要连续观察 GC 后的基线是否抬升。

## 5.2 练习

1. 用 `cluster` 启动 4 个 worker，客户端连续请求 20 次，统计每个 PID 被命中的次数，验证轮询效果。

2. 写一个 CPU 密集函数(如计算斐波那契数列)，分别在主线程和 `worker_threads` 中执行，同时用 `setInterval` 观察事件循环是否被阻塞。

3. 用 `node --cpu-prof` 跑一段包含排序与 JSON 解析的脚本，在 DevTools 中找出耗时最高的函数。

# 6. 小结

| 概念 | 要点 |
|------|------|
| cluster | 多进程利用多核，内存不共享 |
| 负载均衡 | 非 Windows 默认轮询，Windows 交给系统 |
| 零停机 | 先起新 worker，再断开旧的 |
| worker_threads | 线程级并行，适合 CPU 密集任务 |
| 共享内存 | `SharedArrayBuffer` + `Atomics`，需自行同步 |
| 事件循环延迟 | 反映主线程繁忙程度的关键指标 |
| 性能分析 | `--cpu-prof`、`--heap-prof`、火焰图 |

下一篇进入测试与调试，讲清楚 `node:test` 与 Jest 的用法、断点调试、结构化日志、CPU profile 与内存快照的实战。
