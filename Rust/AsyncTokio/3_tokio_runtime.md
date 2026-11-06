标准库只给了 `Future` 的定义，没给执行器。要把 `async` 代码真正跑起来，你得选一个运行时，而 Tokio 是事实标准。这一篇讲清楚：`#[tokio::main]` 到底替你做了什么、多线程与 current-thread 两种运行时如何取舍、`block_on` 处在哪一层，以及那个让无数人踩坑的"运行时边界"。

# 1. 运行时是什么

## 1.1 运行时 = 执行器 + 反应器 + 定时器 + 线程池

Tokio 运行时不是一个"函数"，而是一整套基础设施：

```text
┌─────────────────────────────────────────────┐
│                 Tokio Runtime               │
│                                             │
│  ┌────────────┐   ┌──────────────────────┐  │
│  │  Executor  │   │  Reactor (epoll/IOCP)│  │
│  │  poll 任务  │◀──│  I/O 就绪 → Waker    │  │
│  └─────┬──────┘   └──────────────────────┘  │
│        │                                    │
│  ┌─────▼──────┐   ┌──────────────────────┐  │
│  │ Timer 轮   │   │ Blocking Pool        │  │
│  │ 定时唤醒   │   │ spawn_blocking 线程  │  │
│  └────────────┘   └──────────────────────┘  │
└─────────────────────────────────────────────┘
```

★ **关键认知**：**"引入 Tokio"不等于"用了异步"**，真正让 Future 前进的是运行时。没有运行时，`async fn` 只是造了个状态机；有了运行时，才有 poll 循环、才有 I/O 就绪通知、才有 `sleep` 的定时唤醒。⚠ 所以任何 `async` 代码的运行环境里，必须**恰好有一个**处于活动状态的运行时（或在运行时内部），否则会触发下一篇要讲的边界 panic。

## 1.2 两种运行时 flavor

| Flavor | 构造方式 | 工作线程 | 适用 |
|--------|---------|---------|------|
| `multi_thread` | 默认，`#[tokio::main]` | 多个（默认 = CPU 核数） | 生产服务，要吃满多核 |
| `current_thread` | `#[tokio::main(flavor = "current_thread")]` | 1 个 | 测试、CLI、单线程嵌入 |

```rust
// 多线程运行时（默认）：任务可被多个 worker 并行执行
#[tokio::main]
async fn main() {
    println!("多线程");
}

// 单线程运行时：所有任务在同一个线程上并发（不并行）
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("单线程");
}
```

★ **关键认知**：两种 flavor 对 `async/await` 代码是**透明**的——同一份任务代码既能跑多线程也能跑单线程，差别只在调度。`current_thread` 不是"降级版"，它在测试里反而更好：行为确定、不需要 `Send`、调试简单。⚠ 但注意：`current_thread` 下如果某个任务阻塞了线程，整个运行时都会停摆，没有别的 worker 来救场。

## 1.3 手动构建运行时

`#[tokio::main]` 是语法糖，展开后大致是：

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()          // 打开 I/O 与定时器驱动
        .build()
        .unwrap()
        .block_on(async {
            // 你写的 main 函数体
        })
}
```

等价的显式写法：

```rust
use tokio::runtime::Runtime;

fn main() {
    let rt = Runtime::new().unwrap();          // 等价于多线程 + enable_all
    rt.block_on(async {
        println!("在显式运行时里执行");
    });
}
```

★ **关键认知**：`block_on` 的作用是"**用当前线程阻塞地驱动一个 Future 直到完成**"。它不是异步，而是同步等待异步结果——它是异步世界与同步世界的桥。⚠ `block_on` 只能在运行时外部调用；在异步任务里调用 `block_on` 会 panic（"Cannot start a runtime from within a runtime"）。

## 1.4 自定义 worker 数量

```rust
use tokio::runtime::Builder;

fn main() {
    let rt = Builder::new_multi_thread()
        .worker_threads(4)                 // 固定 4 个 worker
        .max_blocking_threads(64)          // 阻塞线程池上限
        .thread_name("my-worker")          // 便于排查线程栈
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        // ...
    });
}
```

★ **关键认知**：`worker_threads` 默认等于 CPU 核数，但**不是越大越好**。异步任务的切换成本虽低，worker 过多会增加调度竞争；对 I/O 密集服务，通常保持默认或略小即可。⚠ 真正的并行计算应该走 `spawn_blocking` 或 rayon，而不是靠加 worker 数量。

# 2. #[tokio::main] 展开细节

## 2.1 它改写了什么

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ...
    Ok(())
}
```

展开后：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            // 原来的函数体
            Ok(())
        })
}
```

★ **关键认知**：`#[tokio::main]` 要求函数是 `async` 且返回类型能作为 `block_on` 的 `Output`。它会把 `async fn` 变成同步 `fn`，因此**不能在 `#[tokio::main]` 上再叠加会冲突的属性宏**（比如另一个运行时宏）。⚠ 一个进程里可以有多个运行时，但同时只能有一个在 `block_on`，且不要嵌套。

## 2.2 main 之外的地方怎么用

```rust
// 场景 1：同步 main 里手动建运行时
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async { do_work().await });
}

// 场景 2：测试
#[tokio::test]
async fn test_something() {
    assert_eq!(1 + 1, 2);
}

// 场景 3：在异步代码里不要 block_on！
#[tokio::main]
async fn main() {
    // let rt = Runtime::new().unwrap();
    // rt.block_on(...);   // ✗ panic：运行时嵌套
}
```

# 3. 运行时边界：最常见的 panic

## 3.1 什么是"运行时边界"

**运行时边界**指"当前线程是否处在一个正在运行的运行时上下文里"。以下几个操作只有**在运行时内部**才合法：

```text
必须在运行时内（async 上下文）：
  - tokio::spawn
  - tokio::time::sleep
  - tokio::net / tokio::fs 的异步 IO
  - 任何依赖 reactor / timer 的 API

必须在运行时外：
  - Runtime::block_on
```

★ **关键认知**：你看到的两类经典 panic 其实是同一个边界的两个方面：

```text
A. "there is no reactor running, must be called from the context of a Tokio 1.x runtime"
   → 在运行时外调用了需要 reactor 的 API

B. "Cannot start a runtime from within a runtime"
   → 在运行时内调用了 block_on
```

⚠ 前者常见于"在普通 `fn main` 里直接调 `tokio::spawn`"；后者常见于"在 `async` 函数里为了拿同步结果而 `block_on`"。

## 3.2 复现与修复

```rust
// 错误示范：普通 main 里直接 spawn
fn main() {
    tokio::spawn(async {   // ✗ panic：no reactor running
        println!("hi");
    });
}
```

```rust
// 正确做法 1：先建运行时再进入
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        tokio::spawn(async {   // ✓ 此时在运行时内
            println!("hi");
        })
        .await
        .unwrap();
    });
}
```

```rust
// 正确做法 2：直接用 #[tokio::main]
#[tokio::main]
async fn main() {
    tokio::spawn(async { println!("hi"); }).await.unwrap();
}
```

```rust
// 错误示范：异步里 block_on
#[tokio::main]
async fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {});   // ✗ Cannot start a runtime from within a runtime
}
```

## 3.3 从异步里调用同步代码

如果确实需要在异步函数里执行阻塞逻辑，正确姿势是**交出去**，而不是自己 `block_on`：

```rust
#[tokio::main]
async fn main() {
    // 把阻塞工作交给阻塞线程池，await 等结果
    let result = tokio::task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(1));
        42
    })
    .await
    .unwrap();

    println!("{}", result);
}
```

★ **关键认知**：`spawn_blocking` 与 `block_on` 的区别是"**谁等**"。`spawn_blocking` 把阻塞代码丢到专用线程池执行，当前异步任务只是 `.await` 一个句柄，不会卡住 worker；`block_on` 则是让当前线程真停下来，在异步上下文中会直接 panic。⚠ 判断口诀：**异步里想要同步结果 → await 一个 `spawn_blocking`；同步里想跑异步 → `block_on`**。

# 4. 运行时选择实战

## 4.1 一张决策表

| 需求 | 选择 |
|------|------|
| 生产 Web 服务，多核 | `multi_thread`（默认） |
| 单元测试 | `#[tokio::test]`（默认 current_thread） |
| 单线程嵌入式 / WASM | `current_thread` |
| 需要固定线程数做基准测试 | `Builder::worker_threads(n)` |
| 有大量阻塞 IO | 运行时 + `spawn_blocking` 扩容 |

# 5. 常见错误

```text
错误 1：no reactor running
  原因：在运行时外调用 tokio::spawn / sleep / net
  解决：用 #[tokio::main] 或 Runtime::block_on 包起来

错误 2：Cannot start a runtime from within a runtime
  原因：异步里调用 block_on
  解决：改成 .await；需要阻塞就用 spawn_blocking

错误 3：current_thread 下任务互相饿死
  原因：某任务长时间不让出线程（同步阻塞、死循环）
  解决：避免阻塞；必要时换 multi_thread 或 spawn_blocking

错误 4：运行时提前 drop，spawn 的任务全没了
  原因：block_on 返回后主函数结束，Runtime 析构
  解决：在 block_on 内 await 句柄，或显式 shutdown().await
```

# 6. 练习

1. 分别用 `#[tokio::main]` 和手动 `Runtime::new().block_on(...)` 跑同一段异步代码，确认两种写法等价。
2. 写一个程序，在 `#[tokio::main]` 内再 `Runtime::new().block_on(...)`，捕获 panic 信息，并改成 `spawn_blocking` 修复。
3. 用 `Builder::new_multi_thread().worker_threads(2)` 跑 8 个 `sleep(100ms)` 任务，配合 `tokio::time::Instant` 观察总耗时，理解 worker 数与并发的区别。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 运行时 | executor + reactor + timer + blocking pool |
| `#[tokio::main]` | 语法糖，建多线程运行时并 block_on |
| multi_thread | 多 worker，吃多核，生产默认 |
| current_thread | 单线程并发，测试/嵌入式友好 |
| block_on | 同步等异步，只能在运行时外 |
| 运行时边界 | spawn/sleep/net 必须在运行时内 |
| spawn_blocking | 把阻塞代码隔离到专用线程池 |

下一篇进入任务系统：`tokio::spawn` 如何把 Future 变成独立任务、`JoinHandle` 怎么拿结果、为什么要求 `Send + 'static`、任务取消是怎么发生的，以及 `spawn_blocking` 的正确用法与坑。
