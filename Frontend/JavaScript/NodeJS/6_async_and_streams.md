Node 的代码几乎处处异步：读文件、查数据库、调接口。异步写错，轻则结果顺序混乱，重则请求永不返回、内存持续上涨。流则是异步 I/O 的延伸，把大数据切成小块处理。本篇把异步写法、并发控制、流与背压串起来讲。

# 1. 三种异步写法

## 1.1 回调

回调是把"完成后要做什么"作为参数传进去：

```javascript
import fs from "node:fs";

fs.readFile("./a.txt", "utf8", (err, data) => {
  if (err) {
    console.error("读取失败", err);
    return;
  }
  console.log(data);
});
```

错误优先回调的约定是：**第一个参数是错误，第二个才是结果**。回调的问题是嵌套深、错误处理重复，也就是常说的"回调地狱"。

## 1.2 Promise

```javascript
import { readFile } from "node:fs/promises";

readFile("./a.txt", "utf8")
  .then((data) => {
    console.log(data);
  })
  .catch((err) => {
    console.error("读取失败", err);
  });
```

Promise 把"结果"和"错误"统一成一条链，可以用 `.then` 串联、用 `.catch` 兜底。它解决了嵌套，但链太长时仍然不够直观。

## 1.3 async/await

```javascript
import { readFile } from "node:fs/promises";

async function main() {
  try {
    const data = await readFile("./a.txt", "utf8");
    console.log(data);
  } catch (err) {
    console.error("读取失败", err);
  }
}

main();
```

`async` 函数总是返回 Promise，`await` 会暂停当前函数的执行，但**不会阻塞事件循环**——它只是把后续代码注册成微任务。

★ **关键认知**：`await` 的语义是"在这里等结果，同时把线程让给别人"，不是"阻塞整个进程"。一个 `await` 中的慢查询不会卡住其他请求，这和 `readFileSync` 有本质区别。

## 1.4 常见错误

**错误一：在 `forEach` 里 `await`**

```javascript
// 错误:forEach 不会等待回调
[1, 2, 3].forEach(async (id) => {
  await save(id);
});
console.log("全部完成"); // 实际还没完成
```

改用 `for...of`：

```javascript
for (const id of [1, 2, 3]) {
  await save(id); // 串行,逐个等待
}
```

或者需要并行时用 `Promise.all`：

```javascript
await Promise.all([1, 2, 3].map((id) => save(id)));
```

**错误二：忘记 `return` 导致 Promise 丢失**

```javascript
// 错误:外层拿不到内层 Promise
async function load() {
  fetchData(); // 没有 await,也没有 return
}
```

**错误三：吞掉错误**

```javascript
// 错误:没有 catch,失败会变成未处理的 rejection
fetchData().then((data) => console.log(data));
```

**错误四：在循环里串行执行本可并行的任务**

```javascript
// 慢:一个接一个
for (const url of urls) {
  results.push(await fetch(url));
}

// 快:同时发起
const results = await Promise.all(urls.map((url) => fetch(url)));
```

# 2. 并发控制

## 2.1 串行与并行

```text
串行:[任务1]──►[任务2]──►[任务3]──►   总耗时 = 三者之和
并行:[任务1]──┐
      [任务2]──┼──►                    总耗时 ≈ 最慢的那个
      [任务3]──┘
```

`Promise.all` 让任务同时开始，适合互不依赖的 I/O。串行适合后一个任务依赖前一个结果。

## 2.2 all、allSettled、race、any

| 方法 | 成功条件 | 失败条件 | 典型用途 |
|------|---------|---------|---------|
| `Promise.all` | 全部成功 | 任意一个失败即失败 | 依赖全部结果 |
| `Promise.allSettled` | 全部结束 | 永不失败 | 批量任务，允许部分失败 |
| `Promise.race` | 第一个结束 | 第一个失败即失败 | 超时控制 |
| `Promise.any` | 第一个成功 | 全部失败才失败 | 多源竞速 |

```javascript
const results = await Promise.allSettled(urls.map((url) => fetch(url)));

for (const result of results) {
  if (result.status === "fulfilled") {
    console.log("成功", result.value.status);
  } else {
    console.log("失败", result.reason.message);
  }
}
```

超时控制：

```javascript
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => reject(new Error("超时")), ms);
  });
  return Promise.race([promise, timeout]);
}
```

⚠ **关键认知**：`Promise.race` 超时后，**原任务并不会被取消**，它仍在后台执行。真正的取消需要 `AbortController`，把 `signal` 传给 `fetch` 等支持取消的 API。

## 2.3 手写并发池

一次发起 1000 个请求会打爆下游服务。更好的做法是限制同时进行的数量：

```javascript
async function runWithConcurrency(tasks, limit) {
  const results = new Array(tasks.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < tasks.length) {
      const index = nextIndex++;      // 原子地取下一个任务
      results[index] = await tasks[index]();
    }
  }

  const workers = Array.from(
    { length: Math.min(limit, tasks.length) },
    () => worker()
  );

  await Promise.all(workers);
  return results;
}
```

使用：

```javascript
const tasks = urls.map((url) => () => fetch(url));
const responses = await runWithConcurrency(tasks, 5);
```

★ **关键认知**：并发限制的本质是**同时只有 `limit` 个 worker 在取任务**，而不是先分片再串行。每个 worker 完成一个就取下一个，整体吞吐稳定，也不会因为某个分片特别慢而拖尾。

# 3. Stream 基础

## 3.1 四种流

| 类型 | 作用 | 示例 |
|------|------|------|
| Readable | 可读流 | `fs.createReadStream`、`req` |
| Writable | 可写流 | `fs.createWriteStream`、`res` |
| Duplex | 可读可写 | `net.Socket` |
| Transform | 读入后转换再输出 | `zlib.createGzip` |

## 3.2 可读流与可写流事件

```javascript
import { createReadStream } from "node:fs";

const stream = createReadStream("./big.log", { encoding: "utf8" });

stream.on("data", (chunk) => {
  console.log("收到一块", chunk.length);
});

stream.on("end", () => {
  console.log("读完了");
});

stream.on("error", (err) => {
  console.error("读取出错", err);
});
```

可写流：

```javascript
import { createWriteStream } from "node:fs";

const out = createWriteStream("./out.log");

out.write("第一行\n");
out.write("第二行\n");
out.end(); // 必须调用,否则文件不会正常关闭
```

## 3.3 背压

生产者比消费者快时，数据会在内存里堆积，这就是背压问题。

```text
读得快:  [chunk][chunk][chunk][chunk] ──►
写得慢:  [chunk] ...                   缓冲区越来越大
```

`write()` 返回 `false` 表示内部缓冲区已满：

```javascript
import { createReadStream, createWriteStream } from "node:fs";

const reader = createReadStream("./big.log");
const writer = createWriteStream("./copy.log");

reader.on("data", (chunk) => {
  if (!writer.write(chunk)) {
    reader.pause();                         // 暂停读取
    writer.once("drain", () => reader.resume()); // 缓冲排空后恢复
  }
});

reader.on("end", () => writer.end());
```

★ **关键认知**：`write()` 返回 `false` 不是"写入失败"，而是"缓冲区满了，请先停一停"。忽略这个返回值，内存会随着读取速度持续上涨。⚠ 手写背压容易漏掉 `error`、`close` 等分支，生产代码优先用 `pipeline`。

# 4. 管道与组合

## 4.1 pipe 的问题

```javascript
reader.pipe(gzip).pipe(writer);
```

`pipe` 会自动处理背压，但**不会在出错时销毁整条链**，也不会把错误传给最终的回调。源流出错时，下游可能一直挂着。

## 4.2 pipeline

```javascript
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

await pipeline(
  createReadStream("./access.log"),
  createGzip(),
  createWriteStream("./access.log.gz")
);
```

`pipeline` 的优点：

- 自动处理背压；
- 任一环节出错时销毁所有流；
- 返回 Promise，可直接 `await` 并 `try/catch`。

## 4.3 转换流

内置的 `Transform` 可以自定义处理逻辑：

```javascript
import { Transform } from "node:stream";

const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  },
});
```

组合一个"读日志 → 转大写 → 写文件"的管道：

```javascript
await pipeline(
  createReadStream("./input.txt"),
  upperCase,
  createWriteStream("./output.txt")
);
```

★ **关键认知**：流式处理的关键收益是**内存占用与文件大小解耦**。处理 10 GB 文件时，内存里通常只有几十 KB 到几 MB 的块，而不是 10 GB。⚠ 但不要把流的结果再拼成一个巨大字符串，那等于把省下的内存又还回去了。

# 5. 常见泄漏与坑

## 5.1 忘记销毁流

```javascript
// 错误:出错后没有销毁,文件描述符可能泄漏
reader.pipe(writer);
reader.on("error", (err) => console.error(err));
```

用 `pipeline` 或显式 `destroy()`：

```javascript
reader.on("error", (err) => {
  console.error(err);
  reader.destroy();
  writer.destroy();
});
```

## 5.2 未处理的 error 事件

可读流触发 `error` 时，如果没有监听器，会抛出未捕获异常，直接终止进程：

```javascript
stream.on("error", (err) => {
  console.error("流错误", err);
});
```

## 5.3 常见内存泄漏

| 来源 | 表现 | 排查 |
|------|------|------|
| 全局数组只增不减 | 内存随运行时间线性上涨 | 检查缓存与队列 |
| 事件监听器未移除 | 监听器数量持续增长 | `emitter.listenerCount()` |
| 定时器未清理 | 回调持续执行并持有引用 | `clearInterval` |
| 闭包持有大对象 | 快照里大量对象无法回收 | 堆快照对比 |
| 无上限并发 | 同时挂起成千上万个 Promise | 加并发限制 |

```javascript
// 错误:每次请求都注册监听器,却从不移除
app.get("/listen", (req, res) => {
  bus.on("event", handler); // 越积越多
  res.end("ok");
});
```

★ **关键认知**：内存泄漏在 Node 里最常见的形态不是"对象没释放"，而是**某个长期存活的对象(全局缓存、事件总线、定时器)一直持有短生命周期对象的引用**。排查时先找"谁在长期持有它"。

## 5.4 未处理的 rejection

```javascript
process.on("unhandledRejection", (reason) => {
  console.error("未处理的 rejection", reason);
});
```

⚠ **关键认知**：监听 `unhandledRejection` 只是兜底，不是修复。它说明某处 `await` 没有 `try/catch`，或者 Promise 链缺少 `.catch`。生产环境应把它当成需要修复的缺陷，而不是静默忽略。

# 6. 练习

1. 用 `Promise.allSettled` 实现一个批量下载器：传入 20 个 URL，最多同时进行 5 个，返回每个 URL 的成功或失败结果。

2. 用 `pipeline` 实现一个命令行工具，读取一个大文本文件，过滤掉空行，再写入新文件。

3. 写一个最小示例，复现"在 `forEach` 里 `await` 导致顺序错误"的问题，再分别用 `for...of` 和 `Promise.all` 修复，比较两者耗时。

# 7. 小结

| 概念 | 要点 |
|------|------|
| async/await | 让出线程而非阻塞，错误用 try/catch |
| forEach + await | 不会等待，改用 for...of 或 Promise.all |
| allSettled | 等待全部结束，适合允许部分失败 |
| 并发池 | 固定 worker 数，稳定吞吐 |
| 背压 | `write()` 返回 false 时暂停读取 |
| pipeline | 自动背压 + 错误销毁，优于 pipe |
| 泄漏 | 长期对象持有短生命周期引用是主因 |

下一篇进入 cluster 与性能，讲清楚如何利用多核、`cluster` 与 `worker_threads` 的分工、CPU 密集任务的处理，以及内存与 CPU 的性能分析。
