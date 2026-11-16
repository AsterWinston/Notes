很多初学者把 Node.js 当成"另一门 JavaScript"，真正上手后却被 `setTimeout` 和 `setImmediate` 的执行顺序、一个死循环卡住整个服务、`process.nextTick` 插队这些现象绕晕。问题不在语法，而在运行时模型。本篇从 V8 与 libuv 的分工讲起，把事件循环的六个阶段、微任务、阻塞与非阻塞一次讲透。

# 1. Node.js 是什么

## 1.1 运行时不是语言

**JavaScript 是语言，Node.js 是运行时**。语言规定了语法、类型、对象，运行时则提供"让代码跑起来"的环境：引擎、内存管理、文件系统、网络、定时器。

浏览器和 Node.js 都能执行 JavaScript，但给它的能力完全不同。浏览器给你 `document`、`window`、`fetch`；Node.js 给你 `fs`、`path`、`http`、`process`。所以判断一段代码能否在 Node 里跑，先问一句：**它依赖的是语言本身，还是某个宿主环境提供的 API？**

★ **关键认知**：Node.js 不是框架，也不是语言，而是一个 **JavaScript 运行时**。它把 V8 引擎、libuv 事件循环和一套标准库拼在一起，让 JavaScript 能脱离浏览器做服务端的事情。

## 1.2 三层结构

```text
┌─────────────────────────────────────────────┐
│  你的代码 + Node 标准库 (fs / http / net …) │
├─────────────────────────────────────────────┤
│  V8 引擎:解析、编译、执行 JS,管理堆内存     │
├─────────────────────────────────────────────┤
│  libuv:事件循环、线程池、异步 I/O、跨平台    │
├─────────────────────────────────────────────┤
│  操作系统:文件系统、网络协议栈、CPU 调度     │
└─────────────────────────────────────────────┘
```

### 1.2.1 V8 负责什么

V8 是 Google 为 Chrome 写的 JavaScript 引擎，Node.js 直接复用。它负责：

- 把 JavaScript 源码解析成抽象语法树，再编译成机器码；
- 管理调用栈与堆内存，执行垃圾回收；
- 提供 `Promise`、`Map`、`Symbol` 等语言级能力。

V8 只懂 JavaScript，不懂文件、网络、定时器。`fs.readFile` 这样的函数并不在 V8 里，而是 Node 用 C++ 实现后暴露给 JavaScript 的。

### 1.2.2 libuv 负责什么

libuv 是 Node.js 的异步基石，它负责：

- 维护**事件循环**，不断检查有没有待处理的事件；
- 把耗时的系统调用交给**线程池**(默认 4 个线程)或操作系统的异步接口；
- 在任务完成后，把回调重新排进 JavaScript 的队列。

★ **关键认知**：JavaScript 执行是单线程的，但 Node 进程不是单线程的。libuv 的线程池、操作系统的网络轮询、V8 的垃圾回收线程都在背后工作。**"单线程"指的是你的 JS 代码同一时刻只有一段在执行**，不是整个进程只有一个线程。

### 1.2.3 Node 标准库

标准库是架在 V8 与 libuv 之上的 JavaScript 接口层，例如 `fs`、`path`、`os`、`http`、`stream`、`crypto`。它们大多遵循同一套模式：**同步版本 + 回调版本 + Promise 版本**(如 `readFileSync`、`readFile`、`fs/promises.readFile`)。

## 1.3 最小示例

新建 `server.js`：

```javascript
const http = require("node:http");

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain; charset=utf-8" });
  res.end("你好,Node.js");
});

server.listen(3000, () => {
  console.log("服务已启动: http://localhost:3000");
});
```

运行：

```bash
node server.js
```

这段代码能同时处理成千上万个连接，靠的不是多线程，而是**事件循环**：每个请求到达时触发一个回调，回调里不阻塞，处理完立刻返回，事件循环再去处理下一个。

# 2. 事件循环的整体模型

## 2.1 单线程的含义

你的 JavaScript 代码运行在**一个调用栈**上。函数调用压栈，返回出栈。栈空之后，事件循环才有机会把队列里的回调送进来执行。

```text
调用栈                任务队列
┌──────────┐        ┌──────────────────────┐
│ main()   │        │ 定时器回调            │
│  fn()    │  ←──   │ I/O 回调              │
└──────────┘        │ 微任务                │
                    └──────────────────────┘
```

★ **关键认知**：所谓"阻塞"，就是一段同步代码长时间占着调用栈不返回。调用栈被占住，事件循环就无法把其他回调送进来，整个进程看起来像卡死了。Node 的并发能力来自"快速处理并尽快归还调用栈"，而不是"同时执行多段 JS"。

## 2.2 调用栈、任务队列与事件循环

事件循环可以粗略理解为下面这个循环：

```text
while (还有待处理任务) {
  1. 清空当前阶段的回调队列
  2. 执行过程中产生的微任务全部清空
  3. 进入下一个阶段
}
```

每个阶段有自己的队列。事件循环按固定顺序走完一轮，再回到开头。一轮里可能处理多个回调，但**微任务在阶段切换时会被优先清空**。

## 2.3 六个阶段

Node.js 的事件循环分为六个阶段，顺序固定：

| 阶段 | 名称 | 主要工作 |
|------|------|---------|
| 1 | timers | 执行到期的 `setTimeout`、`setInterval` 回调 |
| 2 | pending callbacks | 执行上一轮延迟的系统回调，如 TCP 错误 |
| 3 | idle， prepare | 仅供 Node 内部使用 |
| 4 | poll | 等待并执行 I/O 回调，如文件、网络数据到达 |
| 5 | check | 执行 `setImmediate` 回调 |
| 6 | close callbacks | 执行 `close` 事件回调，如 `socket.on("close")` |

其中 **poll 阶段最特殊**：如果队列为空且没有定时器到期，事件循环会在这里等待，直到有新的 I/O 事件或定时器到期。这也是 Node 进程"没事做时不会空转烧 CPU"的原因。

## 2.4 一轮循环的完整顺序

```text
        ┌───────────────────────────┐
        │           timers           │  setTimeout / setInterval
        └─────────────┬─────────────┘
                      ▼
        ┌───────────────────────────┐
        │     pending callbacks      │  延迟的系统回调
        └─────────────┬─────────────┘
                      ▼
        ┌───────────────────────────┐
        │      idle, prepare         │  Node 内部
        └─────────────┬─────────────┘
                      ▼
        ┌───────────────────────────┐
        │           poll             │  I/O 回调,可能在此等待
        └─────────────┬─────────────┘
                      ▼
        ┌───────────────────────────┐
        │           check            │  setImmediate
        └─────────────┬─────────────┘
                      ▼
        ┌───────────────────────────┐
        │      close callbacks       │  close 事件
        └─────────────┬─────────────┘
                      │
                      └──────── 回到 timers
```

★ **关键认知**：`setTimeout(fn, 0)` 并不等于"立刻执行"，它只是"尽快进入 timers 阶段"；`setImmediate` 则固定在 check 阶段。**两者在同一个阶段里的相对顺序不保证**，但在 I/O 回调内部，`setImmediate` 通常先于 `setTimeout`。

# 3. 微任务与 nextTick

## 3.1 微任务队列

微任务(Microtask)是"当前操作结束后立刻执行"的任务，优先级高于事件循环的下一阶段。主要来源：

- `Promise.then` / `catch` / `finally`；
- `queueMicrotask`；
- `process.nextTick`(严格说它是独立队列，但表现接近微任务)。

事件循环每执行完一个回调、切换阶段之前，都会**清空微任务队列**。如果微任务里又产生微任务，会一直清下去，直到队列为空。

```javascript
Promise.resolve().then(() => {
  console.log("微任务 1");
  Promise.resolve().then(() => console.log("微任务 2"));
});

console.log("同步代码");
```

输出：

```text
同步代码
微任务 1
微任务 2
```

## 3.2 process.nextTick 的特殊优先级

`process.nextTick` 的回调放在一个独立队列里，优先级**高于 Promise 微任务**。它在当前操作结束后、Promise 队列之前执行。

★ **关键认知**：`process.nextTick` 不是"下一轮事件循环"，而是"当前操作结束后马上执行"。名字有误导性，它比 `setTimeout(..., 0)` 快得多。⚠ 递归调用 `process.nextTick` 会**饿死 I/O**：微任务队列永远清不完，事件循环无法进入下一个阶段。

```javascript
function loop() {
  process.nextTick(loop); // 危险:永远清不完的队列
}
loop();
```

## 3.3 经典输出顺序题

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

process.nextTick(() => console.log("4"));

console.log("5");
```

输出顺序：

```text
1
5
4
3
2
```

逐段解释：

1. `1` 和 `5` 是同步代码，最先执行；
2. 同步代码结束后，先清空 `nextTick` 队列，输出 `4`；
3. 再清空 Promise 微任务队列，输出 `3`；
4. 最后进入事件循环的 timers 阶段，输出 `2`。

★ **关键认知**：记忆口诀是 **同步 → nextTick → Promise 微任务 → 宏任务**。⚠ 不要把它当成绝对公式：微任务的清空发生在"每个回调之后、每个阶段之间"，嵌套的微任务会插在中间执行，实际顺序要结合代码逐步分析。

# 4. 阻塞与非阻塞

## 4.1 同步 API 为什么会阻塞

Node 提供同步 API，例如 `fs.readFileSync`。它会**一直占用调用栈**，直到文件读完。

```javascript
const fs = require("node:fs");

console.log("开始读取");
const data = fs.readFileSync("big.log", "utf8"); // 阻塞:CPU 空等磁盘
console.log("读完,长度", data.length);
console.log("这行要等文件读完才执行");
```

在服务端，这种阻塞意味着：**这个请求读取文件期间，其他所有请求都得排队**。文件小、启动脚本里用一下问题不大，但在请求处理函数里用同步 API 是事故高发点。

## 4.2 异步 I/O 的本质

异步版本把"等待"交给 libuv：

```javascript
const fs = require("node:fs");

console.log("开始读取");

fs.readFile("big.log", "utf8", (err, data) => {
  if (err) {
    console.error("读取失败", err);
    return;
  }
  console.log("读完,长度", data.length);
});

console.log("这行会先执行");
```

流程如下：

```text
1. 主线程调用 fs.readFile,把任务交给 libuv 线程池
2. 主线程立刻返回,继续执行后面的同步代码
3. 线程池读完文件,把回调排入 poll 阶段队列
4. 事件循环在 poll 阶段执行回调
```

★ **关键认知**：异步 I/O 不是"JavaScript 同时做了两件事"，而是 **JavaScript 把等待外包给了别的线程，自己先去做别的事**。回调最终仍然回到那唯一的主线程执行。

## 4.3 常见阻塞来源

| 来源 | 例子 | 后果 |
|------|------|------|
| CPU 密集计算 | 大数组排序、加密、图片处理 | 主线程长时间被占，所有请求卡住 |
| 同步文件 API | `readFileSync`、`readdirSync` | 磁盘慢时整个服务停顿 |
| 同步网络调用 | 老式同步 DNS、阻塞式 socket | 网络抖动直接拖垮服务 |
| 超大 JSON 解析 | `JSON.parse` 几十 MB 字符串 | 解析期间无法响应 |
| 死循环 / 正则回溯 | `while(true)`、灾难性回溯 | 进程完全无响应 |

⚠ **关键认知**：CPU 密集任务不会因为"写成 async"就变快。`async function` 里放一个死循环，照样阻塞主线程。要真正利用多核，得用 `worker_threads` 或 `cluster`，这在第 7 篇展开。

# 5. 常见错误与练习

## 5.1 常见错误

**错误一：在请求处理里用同步 API**

```javascript
app.get("/report", (req, res) => {
  const data = fs.readFileSync("report.csv", "utf8"); // 错误:阻塞所有请求
  res.send(data);
});
```

正确做法是使用 `fs/promises` 或流：

```javascript
app.get("/report", async (req, res) => {
  const data = await fs.promises.readFile("report.csv", "utf8"); // 正确:不阻塞
  res.send(data);
});
```

**错误二：以为 setTimeout(fn， 0) 一定比 setImmediate 先**

```javascript
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
```

在主模块里，两者顺序**不保证**。但在 I/O 回调里：

```javascript
fs.readFile(__filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate")); // 通常先输出
});
```

**错误三：用 nextTick 做递归轮询**

递归 `nextTick` 会让事件循环永远进不到 poll 阶段，网络请求全部得不到处理。需要轮询时用 `setImmediate` 或 `setTimeout`。

## 5.2 练习

1. 写出下面代码的输出顺序，并解释原因：

   ```javascript
   setTimeout(() => console.log("A"), 0);
   Promise.resolve().then(() => console.log("B"));
   process.nextTick(() => console.log("C"));
   console.log("D");
   ```

2. 写一个脚本，用异步方式读取一个 100 MB 文件并统计行数，要求读取期间进程仍能响应一个 `setInterval` 打印的心跳。

3. 解释为什么下面代码会让服务"卡死"，并给出两种修复思路：

   ```javascript
   app.get("/hash", (req, res) => {
     let sum = 0;
     for (let i = 0; i < 1e10; i++) sum += i;
     res.send(String(sum));
   });
   ```

# 6. 小结

| 概念 | 要点 |
|------|------|
| Node 运行时 | V8 + libuv + 标准库，不是语言 |
| 单线程 | 指 JS 执行单线程，进程内仍有多个线程 |
| 事件循环 | timers → pending → idle → poll → check → close |
| 微任务 | 每个回调后、阶段切换前清空 |
| nextTick | 优先于 Promise 微任务，递归会饿死 I/O |
| 阻塞 | 同步占住调用栈，异步把等待外包给 libuv |

下一篇进入模块系统，讲清楚 CommonJS 与 ESM 的加载机制、`exports` 与 `module.exports` 的区别、循环依赖，以及 `package.json` 的 `type` 字段如何决定文件按哪种格式解析。
