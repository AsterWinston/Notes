写服务器程序时，最先撞上的墙往往不是语法，而是"等"。一次网络请求要等几十毫秒，一次磁盘读要等几毫秒，如果每个"等"都让线程原地停住，CPU 就在空转烧钱。异步编程要解决的就是这件事：**把"等待"从"占用线程"里剥离出来**。本篇从同步与异步的差别讲起，落到 Rust 的 `async/await` 与 `Future`，先建立正确的心智模型，再去看后面几篇的实现细节。

# 1. 同步与异步：先把问题说清楚

## 1.1 一个同步服务器的瓶颈

先看最朴素的写法：每来一个连接就开一个线程，线程里阻塞地读写。

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};
use std::thread;

fn handle(mut stream: TcpStream) {
    let mut buf = [0u8; 1024];
    // read 是阻塞调用：没有数据时，整个线程原地睡眠
    let n = stream.read(&mut buf).unwrap();
    stream.write_all(&buf[..n]).unwrap();
}

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        thread::spawn(move || handle(stream));   // 每连接一线程
    }
}
```

这段代码能跑，但有两个硬伤：

1. **线程很贵**。Linux 上一个线程默认栈 8 MB，一万个并发连接就是 80 GB 虚拟内存；线程切换还要陷入内核。
2. **大部分线程在睡觉**。连接空闲时线程被 `read` 挂起，什么也不做，却依然占着栈和调度资源。

★ **关键认知**：同步阻塞模型的根本矛盾不是"慢"，而是**等待期间资源被独占**。线程数决定了并发上限，而真正干活的 CPU 核心可能只有 8 个。异步模型的目标是用**少量线程**承载**大量并发任务**，让线程只在"有活可干"时才运行。

## 1.2 阻塞与非阻塞

这两个词描述的是**一次调用会不会让调用方停下来**：

```text
阻塞调用   read(fd)：没数据就一直睡，直到有数据或出错才返回
非阻塞调用 read(fd, O_NONBLOCK)：没数据立刻返回 EWOULDBLOCK，不等
```

非阻塞解决了"不睡"，但带来新问题：**你得反复问"好了没"**。这就是轮询（polling），空转同样浪费 CPU。真正优雅的方案是"注册一个回调，等数据来了再通知我"——这正是事件驱动 I/O（epoll/kqueue/IOCP）与异步运行时的立足点。

## 1.3 同步/异步 与 阻塞/非阻塞 不是一回事

初学者常把这四个词混为一谈。它们的维度不同：

| 维度 | 关注点 | 取值 |
|------|--------|------|
| 同步 / 异步 | 结果怎么拿：调用方主动等，还是被通知 | 同步、异步 |
| 阻塞 / 非阻塞 | 调用期间线程状态：睡着，还是立刻返回 | 阻塞、非阻塞 |

于是有四种组合，其中"异步 + 阻塞"几乎没人用：

```text
                阻塞                    非阻塞
同步     普通 read/write          read + 轮询（忙等）
异步     少见（伪异步）           回调 / Future（Tokio）
```

★ **关键认知**：Rust 的 `async/await` 属于**异步非阻塞**——`.await` 看起来像"等"，但它不会阻塞线程，只是把当前任务挂起、把线程还给执行器去跑别的任务。⚠ 但如果你在 `async` 函数里调用了一个阻塞函数（比如 `std::thread::sleep`），那整个线程照样被卡住，异步的好处瞬间归零。这条坑会贯穿整个模块。

## 1.4 三种并发模型对比

| 模型 | 并发单位 | 优点 | 缺点 |
|------|---------|------|------|
| 多线程 | OS 线程 | 编程直观，能用多核 | 内存与切换开销大，万级连接吃力 |
| 回调 | 回调函数 | 单线程高并发 | 回调地狱，错误处理与控制流难写 |
| async/await | 任务（Future） | 写法接近同步，开销小 | 有染色效应，生态需异步版本 |

异步的本质是**用户态调度**：运行时维护一堆任务，哪个任务的 I/O 就绪了就推进哪个，用一个或几个线程轮转执行。任务切换发生在用户态，成本远低于线程切换。

# 2. Rust 的异步模型

## 2.1 async fn 返回的是 Future

在 Rust 里，`async fn` 不会立刻执行函数体，而是返回一个实现了 `Future` 的值：

```rust
async fn hello() -> String {
    String::from("hello")
}

fn main() {
    let fut = hello();          // 只是构造 Future，函数体一行都没跑
    // fut 此刻什么也没发生
    let s = futures::executor::block_on(fut);   // 被驱动后才真正执行
    println!("{}", s);
}
```

编译器的魔法在于：`async fn` 会被展开成一个**状态机**，每个 `.await` 是一个可能暂停的状态点，局部变量被存进状态机内部。这也是为什么异步函数的栈帧大小在编译期就确定。

★ **关键认知**：`async fn` 是**惰性**的。调用它只得到一张"待办清单"（Future），不会执行任何逻辑；必须有执行器（executor）去 `poll` 它，它才会往前走。这与 JavaScript 的 Promise 一创建就开跑不同，是 Rust 零成本抽象的直接后果。

## 2.2 .await 是暂停点，不是阻塞点

```rust
async fn fetch_two() -> (String, String) {
    let a = fetch("a").await;     // 若 a 未就绪，挂起当前任务，交还线程
    let b = fetch("b").await;     // 同理
    (a, b)
}
```

`.await` 的语义是：**poll 这个子 Future；如果它返回 `Pending`，就把当前任务注册进唤醒队列并让出线程**。等子 Future 就绪，运行时会重新 poll 父 Future，从上次暂停的位置继续。整条链路没有一个线程在"睡"。
```text
任务 A 执行 fetch("a").await
        │
        ▼
  子 Future 返回 Pending
        │
        ▼
  任务 A 挂起，线程去跑任务 B
        │  （I/O 就绪，Waker 被调用）
        ▼
  任务 A 重新入队，从暂停点继续
```

## 2.3 Future 是"可被拉取的"，不是"主动推送的"

Rust 的 Future 采用 **pull 模型**：执行器主动 `poll`，Future 被动响应。这带来一个反直觉的结论——**光创建 Future 而不 poll，什么都不会发生**：

```rust
async fn work() {
    println!("这行永远不会打印");
}

fn main() {
    let _f = work();    // 没有执行器 poll 它
    println!("main 结束");   // 输出：main 结束
}
```

★ **关键认知**：正因为 pull 模型，Rust 才能做到"零成本"——不需要为每个 Future 分配堆内存、不需要内置调度器。Future 就是一个可以内联展开的结构体，而调度策略完全交给运行时（Tokio、async-std、smol）决定。⚠ 代价是标准库只定义了 `Future` trait，不提供运行时，所以你必须显式引入 Tokio 之类的执行器。

## 2.4 谁来推进 Future

执行器做三件事：

```text
1. 保存所有待执行的任务（就绪队列）
2. 循环取出任务，调用 poll
3. 任务返回 Pending 时，把它的 Waker 存起来；Waker 触发后重新入队
```

这套机制会在第二篇《Future 与 poll》里拆开讲。现在只需记住：**Future 描述"做什么"，执行器决定"何时做"**。

# 3. 最小可运行示例

## 3.1 加入 Tokio

```toml
[package]
name = "async-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }   # 学习期开 full 最省心
```

```bash
cargo run
```

## 3.2 串行版：两个 sleep 相加

```rust
use tokio::time::{sleep, Duration};

async fn task(name: &str, ms: u64) -> String {
    sleep(Duration::from_millis(ms)).await;   // 异步睡眠，不阻塞线程
    format!("{} 完成", name)
}

#[tokio::main]
async fn main() {
    let start = std::time::Instant::now();

    let a = task("A", 500).await;   // 先等 A 完成
    let b = task("B", 500).await;   // 再等 B 完成
    println!("{}, {}", a, b);
    println!("耗时 {:?}", start.elapsed());   // 约 1000ms
}
```

## 3.3 并发版：用 join! 同时推进

```rust
use tokio::time::{sleep, Duration};

async fn task(name: &str, ms: u64) -> String {
    sleep(Duration::from_millis(ms)).await;
    format!("{} 完成", name)
}

#[tokio::main]
async fn main() {
    let start = std::time::Instant::now();

    // join! 并发等待多个 Future：两个 sleep 同时计时
    let (a, b) = tokio::join!(
        task("A", 500),
        task("B", 500),
    );
    println!("{}, {}", a, b);
    println!("耗时 {:?}", start.elapsed());   // 约 500ms
}
```

★ **关键认知**：`join!` 只是把多个 Future 放进同一个任务里并发 poll，**并不创建新线程**。两个 `sleep` 在此期间交替挂起、交替被唤醒，因此总耗时是较慢者而非两者之和。⚠ 换成普通的 `let a = ...await; let b = ...await;` 就是串行，这是初学者最常见的性能事故。

# 4. 常见错误

```text
错误 1：`async fn` 里调用阻塞函数
  现象：并发数上不去，CPU 空闲但请求排队
  原因：std::thread::sleep / 同步文件 IO / 同步锁 占住了执行器线程
  解决：用 tokio::time::sleep、tokio::fs，阻塞操作放 spawn_blocking

错误 2：忘记 .await，任务根本没跑
  现象：编译告警 unused must_use，或逻辑静默不执行
  原因：Future 惰性，不 poll 不动
  解决：加 .await，或交给 tokio::spawn

错误 3：以为 async 会自动并行
  现象：for 循环里逐个 await，耗时线性累加
  解决：用 join! / try_join! / FuturesUnordered 并发推进

错误 4：在 main 里直接调用 async fn
  现象：error[E0728] 或类型不匹配
  原因：没有执行器驱动 Future
  解决：加 #[tokio::main]，或用 runtime.block_on
```

# 5. 什么时候该用异步

异步不是银弹。选型时看负载特征：

| 场景 | 推荐 | 理由 |
|------|------|------|
| 高并发网络服务（网关、IM、爬虫） | 异步 | 大量连接、I/O 密集，线程模型撑不住 |
| CPU 密集型计算（图像处理、编解码） | 多线程 / rayon | 异步不加速计算，反而增加调度开销 |
| 简单 CLI、脚本 | 同步 | 引入运行时不划算 |
| 混合负载 | 异步 + spawn_blocking | 用阻塞线程池隔离 CPU 重活 |

★ **关键认知**：异步的收益来自**I/O 等待的重叠**，不是来自"更快地算"。如果你的程序瓶颈是 CPU，异步只会让你多写一层 `async` 关键字。⚠ 判断标准很简单：**线程大部分时间在等 I/O 吗？** 是，异步有价值；否，先考虑多线程。

# 6. 练习

1. 把 3.2 的串行示例改成 3 个各 300ms 的任务，分别用"顺序 await"和 `join!` 运行，打印两种耗时，验证并发确实重叠。
2. 写一个 `async fn`，其中用 `std::thread::sleep` 代替 `tokio::time::sleep`，同时跑 3 个任务并计时。观察耗时为什么仍接近串行，理解"阻塞调用会卡住执行器"。
3. 用 `tokio::time::timeout` 给一个 `sleep(2s)` 的任务加 500ms 上限，观察返回的 `Err`，体会异步任务可以被"外部打断"。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 同步 / 异步 | 结果获取方式：主动等 vs 被通知 |
| 阻塞 / 非阻塞 | 调用期间线程是否被占用 |
| async fn | 返回 Future，惰性，不 poll 不执行 |
| .await | 暂停点，挂起任务、交还线程 |
| Future | pull 模型，由执行器驱动 |
| 并发 vs 并行 | `join!` 是并发，不必然多线程 |
| 异步适用 | I/O 密集、高并发；CPU 密集另寻他法 |

下一篇进入 `Future` 的内部：`poll` 到底返回什么、`Waker` 如何唤醒任务、`Pin` 为什么必须存在，以及执行器的循环是怎样一圈圈转起来的。理解了那一层，`async/await` 就不再是黑盒。
