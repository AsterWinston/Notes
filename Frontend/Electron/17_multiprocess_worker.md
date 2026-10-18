主进程既要管窗口、又要处理业务,一旦碰上**重计算**(视频处理、大文件、CPU 密集任务),就会把整个应用拖卡。解法是**再开进程/线程**把重活挪出去。本篇讲清 Electron 的三种"多任务"手段——`utilityProcess`、`child_process`、`worker_threads`,及其选型。

# 1. 为什么要再开进程

## 1.1 主进程不能卡

```
主进程职责:窗口、IPC、系统 API —— 它对"卡顿"最敏感(一卡全应用掉帧)
重活(压缩、加密、解析大文件)若放主进程,UI 直接冻结
```

★ **关键认知**：主进程**一旦被重计算占满,整个应用(所有窗口)都卡**。⚠ 所以重活要么在渲染进程(前端 worker),要么**再开一个独立的进程/线程**跑——这是"保持主进程轻快"的必然要求。

# 2. utilityProcess(Electron 推荐)

## 2.1 派生子进程跑重活

```javascript
const { utilityProcess } = require('electron')

const child = utilityProcess.fork('./worker.js')   // 派生子进程,跑 worker.js

// 子进程之间通信(MessagePort)
child.postMessage({ task: 'process', data })       // 发任务
child.on('message', (msg) => { /* 收结果 */ })

child.kill()                                        // 结束
```

★ **关键认知**：`utilityProcess.fork` 派生一个**独立的子进程**(跑你自己的 JS),通过 `postMessage/on('message')` 通信。⚠ 它是 Electron 官方为"**重活子进程**"设计的方案——比 `child_process` 更贴合 Electron 环境(进程类型正确),适合"主进程要跑重活却不想自己卡"。

## 2.2 为什么要独立进程而不是线程

★ **关键认知**：**进程(process)** 比**线程(thread)** 隔离更彻底——子进程崩了不影响主进程,内存独立。⚠ Electron 里重活用**进程**(utilityProcess/child_process)更稳;`worker_threads`(线程)轻但共享进程,某些场景(CPU 密集、无 IO 阻塞)才用线程。

# 3. child_process(Node 原生)

## 3.1 运行外部程序/脚本

```javascript
const { exec, spawn, execFile } = require('child_process')

// 执行一个命令(如调用 ffmpeg 压缩视频)
exec('ffmpeg -i in.mp4 out.mp4', (err, stdout, stderr) => { /* 结果 */ })

// spawn:流式、适合大量输出/长驻
const child = spawn('ffmpeg', ['-i', 'in.mp4', 'out.mp4'])
child.stdout.on('data', d => console.log(d.toString()))
```

★ **关键认知**：`child_process` 是 Node 原生能力——**运行外部程序**(命令、脚本、其他可执行文件)。⚠ `exec` 适合"一次性拿结果"、`spawn` 适合"流式/长驻/大量输出";用 Electron 调 `ffmpeg`/`git` 这类系统工具,就是 child_process 的典型场景。

## 3.2 与 utilityProcess 的区别

| 维度 | utilityProcess | child_process |
|------|----------------|---------------|
| 跑什么 | 你的 JS(子进程) | 外部命令/脚本 |
| 通信 | postMessage | stdout/事件 |
| 定位 | Electron 重活子进程 | 调用系统工具 |
| 复杂度 | 简单 | 略底 |

★ **关键认知**：**utilityProcess 跑"你自己的重活 JS",child_process 运行"外部程序/命令"**。⚠ 要"把主进程的重计算搬出去"用 utilityProcess;要"调用系统里的 ffmpeg/git"用 child_process,二者定位不同。

# 4. worker_threads(线程)

## 4.1 CPU 密集用线程

```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads')

// 主线程:worker 跑 cpu-task.js
const worker = new Worker('./cpu-task.js')
worker.on('message', result => console.log(result))
worker.postMessage({ data })

// cpu-task.js 里:
// parentPort.on('message', ...) 处理,parentPort.postMessage(结果)
```

★ **关键认知**：`worker_threads` 在**同进程内开线程**,适合**CPU 密集**的计算(比进程轻,能共享内存)。⚠ 线程与进程的取舍:线程**轻、快、可共享内存**,但**崩了会连累进程**;进程**重、稳、隔离**。CPU 密集短任务可选线程,有崩溃风险/长驻的用进程。

# 5. 选型与协作模式

| 场景 | 用 |
|------|-----|
| 主进程要跑重活 JS | utilityProcess |
| 调用系统工具(ffmpeg/git) | child_process |
| CPU 密集、短任务 | worker_threads |
| 渲染进程内的重计算 | Web Worker |

★ **关键认知**：四个层次——**渲染进程重计算用 Web Worker,主进程重活用 utilityProcess,调系统工具用 child_process,CPU 密集短任务用 worker_threads**。⚠ 共同目标都是同一个:**不让"该响应用户的进程"被重活占满**。

## 5.1 任务分发的心智

```
主进程(轻,只调度) ──派发──▶ 子进程/线程(重,只计算) ──结果──▶ 主进程 → 推给渲染
```

★ **关键认知**：理想形态是"**主进程只调度、重活进程只计算**"——主进程发任务、收结果、更新 UI,重活完全不占主进程。⚠ 这是"多进程架构"的价值:把计算与响应分离开,应用始终流畅。

# 6. 本篇小结与下一步

★ **关键认知**：重活别放主进程。三层工具——utilityProcess(跑自己的 JS 子进程)、child_process(调系统命令)、worker_threads(线程);渲染层用 Web Worker。共同目标:让响应进程保持轻快。

下一步第 18 篇:原生模块与 Node 集成——native addons、ffi,以及 Electron 里 Node 能力的边界。