# 1. 压测端性能优化目标

## 1.1 为什么要优化压测端

**压测端（施压机）自身的瓶颈会让测试结果系统性失真**：你以为测出了被测系统的极限，实际测出的是 k6 进程或施压机内核的极限。典型症状是 VU 加到一定数量后 RPS 不再增长、`iteration_duration` 持续膨胀，而服务端 CPU 才 30%——压力根本没打出去。这类失真比测不出来更危险，因为它会得出"系统容量只有 X"的错误结论，导致过度扩容或错误的架构决策。

优化的目标可以一句话概括：**确保观察到的每一个延迟和错误都来自被测系统，而不是施压链路**。为此需要在测试前评估压测端容量、测试中监控压测端资源、测试后校验数据可信度（详见 5.2）。

## 1.2 资源模型

k6 施压机的资源开销集中在三处：

- **CPU**：k6 用 Go 编写的 JS 解释器 Goja 执行脚本，**Goja 没有 JIT 编译**，纯解释执行的 JS 比 V8 慢一个量级；此外 TLS 握手的加解密、大响应体的 JSON 解析都是 CPU 密集操作。CPU 通常是施压机的第一瓶颈。
- **内存**：每个 VU 持有一个独立的 Goja 运行时实例（isolate），外加脚本 init 阶段加载的数据副本、未丢弃的响应体缓存。VU 数量与单 VU 内存占用的乘积决定总内存。
- **网络**：带宽、TCP 连接数、临时端口（ephemeral port）范围、文件描述符上限。高并发短连接场景下，fd 与端口耗尽比带宽耗尽更常见。

```text
┌───────────────────────── k6 进程 ─────────────────────────┐
│                                                            │
│  VU 1              VU 2              VU N                  │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │ Goja 运行时│     │ Goja 运行时│     │ Goja 运行时│  ← 每 VU 一份 │
│  │ init 数据副本│    │ init 数据副本│    │ init 数据副本│ ← 无 SharedArray │
│  │ 响应体缓存  │     │ 响应体缓存  │     │ 响应体缓存  │ ← 未 discard 时 │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘            │
│       │ goroutine 调度到 GOMAXPROCS 个 OS 线程               │
│  ─────▼────────────────▼────────────────▼─────              │
│  netpoller(epoll) → TCP 连接 × N → 网卡                      │
└────────────────────────────────────────────────────────────┘
```

# 2. VU 与资源关系

## 2.1 VU 成本评估

**一个 VU = 一个 goroutine + 一个专属的 Goja JS 运行时实例**。goroutine 本身很便宜（初始栈 2KB 级），真正的成本在 Goja 运行时和脚本加载的数据：最小化脚本（只发 HTTP 请求）单 VU 常驻内存约 1~2MB，引入大型 JS 库或大 JSON 数据文件后单 VU 可轻松超过 10MB。粗略估算公式：

```text
压测端内存 ≈ VU 数 × (Goja 基础开销 + init 数据副本 + 响应体缓存)
```

其中最容易被忽视的是 **init 数据副本**：k6 默认为每个 VU 独立执行一遍 init 上下文，`JSON.parse(open('data.json'))` 的结果在每个 VU 里各存一份。1 万个 VU 加载 5MB 的 JSON 就是 50GB 内存——这正是 SharedArray 要解决的问题（见 3.4）。CPU 方面，Goja 解释执行的开销与脚本复杂度成正比：纯 `http.get` 循环单机几千 VU 没问题，脚本里做大量字符串处理、JSON 操作时，同样机器可能撑不住几百 VU。

## 2.2 资源瓶颈识别

**"吞吐上不去但服务端很闲"是压测端瓶颈的决定性信号**。逐项排查：

| 观察点 | 命令 / 指标 | 瓶颈判定 |
|--------|-------------|----------|
| k6 进程 CPU | `top -p $(pgrep k6)` | 接近 核数×100% 即 CPU 打满 |
| 连接阻塞时间 | `http_req_blocked` | 持续升高 = 客户端建连能力不足（fd/端口/连接池） |
| 文件描述符 | `ls /proc/$(pgrep k6)/fd \| wc -l` 对比 `cat /proc/$(pgrep k6)/limits` | 接近 `Max open files` 即 fd 耗尽 |
| socket 状态分布 | `ss -s`、`ss -tan state time-wait \| wc -l` | TIME_WAIT 数万级 = 短连接端口耗尽 |
| 网卡吞吐 | `sar -n DEV 1` | 接近网卡带宽 = 网络打满 |
| 内存 | `ps -o rss= -p $(pgrep k6)` | 持续增长不回落 = 疑似泄漏或响应体堆积 |

`http_req_blocked` 需要特别解释：它统计的是"请求发出前被阻塞的时间"（获取连接、DNS 解析、TCP/TLS 握手前的排队）。服务端慢会体现在 `http_req_waiting` 上；**`http_req_blocked` 升高几乎总是客户端侧的问题**——连接池拿不到连接、fd 上限、DNS 解析慢。

## 2.3 VU 数量建模与到达率换算

**VU 数量不是拍脑袋定的，用 Little's Law 从目标到达率反推**：

```text
并发数 L = 到达率 λ × 平均响应时间 W
```

要模拟 500 RPS、目标接口平均响应 200ms，则最少需要 `500 × 0.2 = 100` 个 VU；考虑响应时间毛刺（P95 可能是均值的 2~3 倍）和脚本里的 `sleep`，实际配置留 30%~50% 余量，即 130~150 VU。反过来，封闭模型（`per-vu-iterations` / `constant-vus`）下 RPS 由公式 `RPS = VU 数 ÷ (思考时间 + 响应时间)` 决定——响应时间一旦变慢，封闭模型的 RPS 自动下降，这正是它产生**协调遗漏**（coordinated omission）的原因，需要精确控制到达率时应使用开放模型的 `constant-arrival-rate` 执行器：

```javascript
export const options = {
  scenarios: {
    api_load: {
      executor: 'constant-arrival-rate',
      rate: 500,                 // 目标到达率：500 次迭代/秒
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 150,      // Little's Law：500 × 0.2 × 1.5 余量
      maxVUs: 400,               // 服务端变慢时允许临时扩容的上限
    },
  },
};
```

> **设计考量**：`preAllocatedVUs` 与 `maxVUs` 的差值是开放模型的"缓冲池"。服务端响应变慢时 k6 会从缓冲池借 VU 维持到达率；`maxVUs` 耗尽后到达率开始掉，此时 summary 里的 `dropped_iterations` 指标大于 0——**看到 dropped_iterations 就说明要么服务端撑不住了，要么 maxVUs 配小了**，两者要靠服务端指标区分。

# 3. 脚本优化策略

## 3.1 减少不必要的解析

**响应体是压测端最大的内存来源，默认全部缓存在 VU 内存里**。k6 的 `discardResponseBodies` 选项默认为 `false`，即所有响应体都会被读进内存以支持 `res.json()`、`res.body` 访问。大流量测试中应全局丢弃响应体，只对确需断言内容的请求单独开启：

```javascript
export const options = {
  discardResponseBodies: true,   // 全局丢弃，内存占用大幅下降
};

export default function () {
  // 普通请求：只关心状态码，无响应体开销
  http.get(`${BASE}/api/items`);

  // 个别请求需要校验内容：单独声明 responseType
  const res = http.get(`${BASE}/api/items/1`, { responseType: 'text' });
  check(res, { 'body ok': (r) => r.json('data.id') === 1 });
}
```

解析同理：不要对 2MB 的 JSON 响应做 `res.json()` 全量解析却只用一个字段，`res.json('data.items.0.id')` 按路径提取，或者干脆在 `discardResponseBodies: true` 下只断言 `status === 200`。

## 3.2 控制日志与调试

**`console.log` 是热路径上的性能杀手**：每条日志都要跨 JS↔Go 边界序列化，再同步写入输出流。几千个 VU 每次迭代打一条日志，吞吐能掉一个数量级，还会让 CI 日志无法阅读。正确姿势是调试与压测分离：

```bash
# 调试阶段：单 VU、日志输出到文件，观察完整行为
k6 run --vus 1 --iterations 3 --console-output debug.log script.js

# 压测阶段：脚本内不带 console.log，或用 --quiet 抑制非必要输出
k6 run --quiet script.js
```

如果脚本里必须保留日志能力，用环境变量做开关：`if (__ENV.DEBUG === '1') console.log(...)`，压测时不注入即零开销（条件判断本身可忽略）。

## 3.3 合理的思考时间

`sleep` 在负载测试中有双重作用：**模拟真实用户节奏**（真人不会零间隔连续点击），以及**给压测端自身降温**（sleep 期间 VU 的 goroutine 挂起，几乎不占 CPU）。做容量摸底的 stress 测试可以去掉 sleep 打最大吞吐，但要盯紧 2.2 的客户端指标——没有 sleep 的脚本对压测端的压力成倍增加，测出的"极限"可能先是施压机的极限。

## 3.4 减少 per-iteration 分配与 SharedArray

**init 上下文里的代码每个 VU 只执行一次，default 函数里的代码每次迭代都执行**——把不变的对象构造全部提升到 init 阶段：

```javascript
// 反例：每次迭代都新建 headers 对象和 URL 拼接（GC 压力大）
export default function () {
  http.get(`${BASE}/api/items`, { headers: { Authorization: `Bearer ${TOKEN}` } });
}

// 正例：params 对象在 init 阶段构造一次，迭代内复用
const PARAMS = { headers: { Authorization: `Bearer ${__ENV.API_TOKEN}` } };
export default function () {
  http.get(`${BASE}/api/items`, PARAMS);
}
```

Goja 没有分代 GC 那种精细优化，高频小对象分配会显著推高 CPU。大数据文件则必须用 `SharedArray`：

```javascript
import { SharedArray } from 'k6/data';

// 数据只在进程内加载解析一次，所有 VU 共享同一份底层数组
const users = new SharedArray('users', () => JSON.parse(open('./users.json')));

export default function () {
  const u = users[Math.floor(Math.random() * users.length)];
  http.post(`${BASE}/login`, JSON.stringify(u));
}
```

> **注意**：SharedArray 共享的是底层存储，**每次读取元素时仍会拷贝一份返回**（防止 VU 间互相污染），所以访问比普通数组略慢。取舍原则：数据量大（MB 级以上）或 VU 数多时用 SharedArray 换内存；数据只有几十条、访问极频繁的小热点数据，直接用普通数组反而更快，内存重复的代价可以忽略。

## 3.5 --compatibility-mode 取舍

k6 的 `--compatibility-mode` 决定脚本如何被编译进 Goja：

| 模式 | 行为 | 代价 | 适用 |
|------|------|------|------|
| `full`（默认） | 用 Babel 把 ES6+ 转译为 ES5.1 再执行 | init 阶段慢、每 VU 内存更高 | 依赖老旧 npm 库、遇到 Goja 语法报错时 |
| `base` | 跳过转译，直接用 Goja 原生 ES6+ 支持 | 部分新语法（如某些 ES2020+ 特性）不支持 | TypeScript 编译产物、现代 JS 脚本 |

**压测大 VU 数时推荐 `--compatibility-mode=base`**：省去 Babel 转译能明显缩短启动时间、降低每 VU 内存。用 TypeScript + esbuild 构建的脚本（产物已是标准 ES 代码）几乎都应跑 base 模式；只有当遇到 `SyntaxError: Unexpected token` 这类 Goja 原生解析不了的语法时才退回 full。

# 4. 网络与连接优化

## 4.1 连接复用

**k6 默认每个 VU 复用 HTTP 连接（keep-alive），这是压测端最重要的默认优化**。`noConnectionReuse: true` 会让每个请求都重新握手，CPU（TLS 加解密）与 TIME_WAIT 数量暴涨，仅在专门测试握手性能时开启。相关的还有 `noVUConnectionReuse`（默认 false）：设为 true 时同一 VU 的迭代之间也不复用连接。压测 HTTPS 时注意 TLS 握手成本远高于 HTTP，1 万个 VU 同时冷启动建连的瞬间 CPU 尖峰可能打满施压机——用 `stages` 渐进加压（ramping）而不是瞬间拉满，既贴近真实流量，也保护压测端。

## 4.2 超时设置

超时是双刃剑：**过短会把正常网络抖动记成失败放大错误率，过长会让慢响应长时间占住 VU**——封闭模型下 VU 被挂住直接意味着吞吐下降，你会误以为系统 RPS 上不去，其实是自己的超时配置在限流。k6 全局默认超时 60s，对大多数 API 太长；按业务 SLO 设置为"P99 响应时间的 2~3 倍"较合理：

```javascript
export const options = {
  timeout: '10s',              // 全局默认，作用于所有请求
  insecureSkipTLSVerify: false, // 不要为了省事全局跳过证书校验，会掩盖 TLS 配置问题
};
```

## 4.3 Linux 内核参数调优

单机上万连接时，默认内核参数会先于 k6 成为瓶颈。施压机（Linux）标准调优：

```bash
# /etc/sysctl.d/99-k6.conf —— 施压机内核调优
net.core.somaxconn = 65535              # 监听队列上限（本机起 mock/代理服务时必须调）
net.ipv4.ip_local_port_range = 1024 65535  # 临时端口范围，决定单机可建连接数上限
net.ipv4.tcp_tw_reuse = 1               # 允许复用 TIME_WAIT 端口（仅对客户端安全）
net.ipv4.tcp_fin_timeout = 15           # 缩短 FIN_WAIT 时间
net.netfilter.nf_conntrack_max = 262144 # 走 NAT/防火墙时必调，否则 conntrack 表满丢包
fs.file-max = 2097152                   # 系统级 fd 上限

# 进程级 fd 上限：每个 TCP 连接占 1 个 fd，默认 1024 远远不够
ulimit -n 250000
# systemd 服务方式运行 k6 时，在 unit 里配 LimitNOFILE=250000
```

> **行业实践**：调优后先做一次"施压能力标定"——用同一脚本对已知容量的靶机（或本地 nginx）压一轮，记录本机能稳定打出的最大 RPS 和 VU 数，作为后续所有测试的客户端容量基线。正式压测时把负载控制在标定值的 70% 以内，超出就加机器（见 5.1）。

# 5. 客户端扩展策略

## 5.1 水平扩展

**单机达到瓶颈（CPU 70%+ 或网络打满）后，加 VU 只会污染数据，唯一出路是水平扩展施压机**。v0.5x 的三条路径：原生分布式执行（k6 v0.49+ 引入，控制节点把负载切分给多个 instance 节点，`k6 run --execution-mode=distributed` 配合 `K6_INSTANCES` 指定节点列表，仍属较新功能，生产使用前充分验证）、Kubernetes 上的 k6-operator（把 instance 跑成 Pod，弹性扩缩，适合云原生团队）、Grafana Cloud k6（全托管，按负载付费，免运维但数据出内网需评估合规）。选型细节见第 15 篇分布式专题。

## 5.2 监控客户端

**压测报告必须附带压测端自身的资源数据，否则结果不可信**。三层监控：

- **k6 内置指标**：`vus` / `vus_max`（实际并发是否达到配置值）、`data_received` / `data_sent`（带宽速率）、`dropped_iterations`（开放模型下 VU 不够的直接证据）。使用 `--out experimental-prometheus-rw` 输出时还会带上 k6 进程的 Go 运行时指标（`k6_go_goroutines`、内存统计），可直接在 Grafana 建面板。
- **进程级**：`pidstat -p $(pgrep k6) 1` 采 CPU/内存曲线，与测试时间轴对齐。
- **系统级**：node_exporter 或云监控采网卡吞吐、TCP 重传（`netstat -s | grep -i retrans`）、fd 使用量。

复盘时的判定规则：压测端 CPU 峰值 < 70%、无 fd/端口告警、`http_req_blocked` P95 < 10ms，则本次数据的瓶颈结论可以采信；任何一项不满足，先修压测端再重测。

# 6. 小结

压测端优化的主线是"识别瓶颈 → 建模容量 → 脚本减负 → 系统调优 → 水平扩展"。k6 的资源模型核心是每 VU 一个 Goja 解释器实例：CPU 开销来自无 JIT 的 JS 解释和 TLS/JSON 处理，内存开销来自运行时、init 数据副本与响应体缓存，网络开销受 fd、临时端口和 somaxconn 约束。VU 数量用 Little's Law（L = λ × W）从目标到达率反推并留余量，精确控制到达率用开放模型执行器并盯住 dropped_iterations。脚本层面的四大杠杆：discardResponseBodies 丢响应体、console.log 移出热路径、不变对象提升到 init 上下文、大数据用 SharedArray 共享；再配合 `--compatibility-mode=base` 和内核参数调优，单机可稳定支撑的 VU 数能提升数倍。最后，任何压测报告都要附带压测端自身的 CPU、内存、网络、blocked 时间数据——压测端不干净的结果，宁可不要。
