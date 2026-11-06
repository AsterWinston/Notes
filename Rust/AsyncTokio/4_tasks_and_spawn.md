`join!` 能把几个 Future 并发起来，但它们仍在**同一个任务**里、共用一份调度上下文。想要"扔出去、各跑各的、回头再收结果"，就需要 `tokio::spawn`。这一篇讲清楚任务与 Future 的区别、`JoinHandle` 的用法、`Send + 'static` 为什么是硬性要求、任务取消到底发生了什么，以及如何把阻塞代码安全地踢出异步世界。

# 1. 任务与 Future 的区别

## 1.1 任务是被 spawn 出去的 Future

```rust
#[tokio::main]
async fn main() {
    // 直接 await：当前任务继续跑，直到这个 Future 完成
    let a = work(1).await;

    // spawn：把 Future 变成独立任务，立刻返回 JoinHandle
    let handle = tokio::spawn(work(2));

    let b = handle.await.unwrap();   // 稍后再收结果
    println!("{} {}", a, b);
}

async fn work(n: u32) -> u32 {
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    n * 10
}
```

★ **关键认知**：`spawn` 的意义不是"开启并发"（`join!` 也能并发），而是**让任务脱离当前调用栈独立调度**：它可以跑在另一个 worker 线程上、可以在当前函数返回后继续存活、可以在任意时刻被单独取消。⚠ 反过来说，`spawn` 出来的任务**不保证执行顺序、不保证同一线程、也不保证一定执行完**——运行时不会替你等待它。

## 1.2 任务的生命周期

```text
tokio::spawn(fut)
   │
   ├─ 把 fut 装箱，分配一个 Task 结构
   ├─ 生成 JoinHandle（持有一份"结果通道"）
   ├─ Task 推入就绪队列
   ▼
worker 线程 poll 它
   │
   ├─ Ready(v) ──▶ 结果写入 JoinHandle，任务销毁
   └─ Pending  ──▶ 等 Waker，稍后继续

如果 JoinHandle 被 drop 而任务还没完成：
   → 任务默认继续运行（detached），结果被丢弃
```

★ **关键认知**：**drop `JoinHandle` 不会取消任务**，只是放弃拿结果。任务会成为"游离任务（detached task）"继续跑，直到完成或进程退出。这与很多语言"句柄释放即取消"的直觉相反。⚠ 想取消任务，必须显式调用 `handle.abort()` 或让 Future 被 drop（见第 4 节）。

# 2. JoinHandle 与结果

## 2.1 await 拿到的是两层 Result

```rust
let handle: tokio::task::JoinHandle<u32> = tokio::spawn(async {
    tokio::time::sleep(std::time::Duration::from_millis(50)).await;
    7
});

// JoinError 表示"任务没正常跑完"：panic 或被取消
match handle.await {
    Ok(value) => println!("结果 {}", value),
    Err(e) if e.is_panic() => println!("任务 panic 了"),
    Err(e) if e.is_cancelled() => println!("任务被取消"),
    Err(e) => println!("其他错误: {}", e),
}
```

★ **关键认知**：`JoinHandle::await` 返回 `Result<T, JoinError>`，其中外层 `Err` 只表达**任务级别的失败**（panic / 取消 / 运行时关闭），**不表达业务错误**。业务错误应该包在 `T` 里，比如 `T = Result<Data, MyError>`。⚠ 初学者常把"任务 panic"和"业务返回 Err"混在一起处理，导致把可恢复错误当成崩溃。

## 2.2 panic 会被捕获

```rust
let handle = tokio::spawn(async {
    panic!("任务内部崩了");
});

let err = handle.await.unwrap_err();
println!("是 panic 吗：{}", err.is_panic());   // true
// 进程本身不会因为子任务 panic 而崩溃
```

★ **关键认知**：Tokio 会捕获任务内的 panic 并转成 `JoinError`，**不会直接终止进程**（除非 panic 策略配置为 abort）。这是任务隔离的好处：一个任务的崩溃不会拖垮整个服务。⚠ 但 panic 依然会中止该任务的正常清理逻辑，因此不要把"预期内的失败"写成 `panic!`，那会污染错误链路。

## 2.3 多个句柄一起等

```rust
use tokio::task::JoinSet;

let mut set = JoinSet::new();
for i in 0..5 {
    set.spawn(async move {
        tokio::time::sleep(std::time::Duration::from_millis(100 * i)).await;
        i
    });
}

// 按完成顺序取结果，而不是按 spawn 顺序
while let Some(res) = set.join_next().await {
    println!("完成：{:?}", res.unwrap());
}
```

★ **关键认知**：固定数量、需要一起等 → `tokio::join!`；动态数量、按完成顺序处理 → `JoinSet`；只想"发出去不管" → 直接 drop 句柄（但要考虑任务泄漏，见下一篇）。⚠ `join!` 在任一分支一直 Pending 时才 Pending，而 `JoinSet::join_next` 是"谁先好先返回谁"，适合任务池、爬虫抓取这类场景。

# 3. 为什么要求 Send + 'static

## 3.1 'static：任务可能活得比调用者久

```rust
async fn spawn_with_borrow(data: &[u8]) {
    // ✗ 编译错误：borrowed data escapes outside of function
    tokio::spawn(async move {
        println!("{:?}", data);
    });
}
```

`spawn` 出来的任务会被放到运行时的任务队列，**可能在你这个函数返回之后才被执行**。如果它借用了栈上的 `data`，函数返回后引用就悬垂了。因此编译器要求 Future 满足 `'static`：不能持有短生命周期的借用（`'static` 借用或所有权都行）。

```rust
// 正确：转移所有权
async fn spawn_ok(data: Vec<u8>) {
    tokio::spawn(async move {
        println!("{:?}", data);   // data 已被 move 进任务
    });
}
```

★ **关键认知**：`'static` 不是"必须活到程序结束"，而是"**不包含短于 'static 的借用**"。把 `Arc` 复制一份进去是常见解法：`let data = Arc::clone(&data); tokio::spawn(async move { ... });`。

## 3.2 Send：任务可能被搬到别的线程

多线程运行时会把任务在 worker 之间迁移，所以 Future 必须 `Send`。这排除了持有 `Rc`、`RefCell` 等在多线程下不安全的类型。

```rust
use std::rc::Rc;

async fn bad() {
    let r = Rc::new(1);
    tokio::spawn(async move {   // ✗ Rc 不是 Send
        println!("{}", r);
    });
}
```

```rust
use std::sync::Arc;

async fn good() {
    let r = Arc::new(1);
    tokio::spawn(async move {   // ✓ Arc 是 Send + Sync
        println!("{}", r);
    });
}
```

★ **关键认知**：`Send + 'static` 不是 Tokio 故意刁难，而是**任务可能跨线程、跨栈帧存活**这两个事实的直接推论。⚠ 用 `current_thread` 运行时并不免除 `Send` 约束——`spawn` 的签名始终要求它；只有 `spawn_local`（需 `LocalSet`）才能跑非 `Send` 的 Future。

★ **关键认知**：若函数参数是借用且要 spawn，先在函数内做一次"借用 → 拥有"的转换（比如 `s.to_string()`），把所有权带进任务，就能解耦生命周期。这类"提前克隆"是异步代码里非常常见的取舍。

# 4. 任务取消

## 4.1 abort：强制中止

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        tokio::time::sleep(Duration::from_secs(10)).await;
        println!("不该看到这行");
    });

    tokio::time::sleep(Duration::from_millis(100)).await;
    handle.abort();                     // 请求取消

    match handle.await {
        Err(e) if e.is_cancelled() => println!("已取消"),
        _ => println!("其他情况"),
    }
}
```

★ **关键认知**：`abort()` 是**协作式的"下一次 poll 时丢弃"**，不是"立刻杀线程"。运行时会在任务下一次被调度时直接 drop 它，防止它再往前推进。⚠ 如果任务正处于一段**没有 await 的同步计算**中，abort 无法打断它，必须等它跑到下一个 await 点。这正是"长同步计算必须让出（或 spawn_blocking）"的又一个理由。

## 4.2 drop 也是取消

```rust
async fn scoped() {
    let handle = tokio::spawn(async { /* 长任务 */ });
    // handle 在这里被 drop，但任务继续跑（detached）
}
```

```rust
// 用 JoinSet 时，drop 整个 set 会 abort 其中的任务
let mut set = JoinSet::new();
set.spawn(async { /* ... */ });
// set 离开作用域 → 所有任务被 abort
```

★ **关键认知**：取消的粒度取决于"谁持有任务"。`JoinHandle` 被 drop 不会取消；`JoinSet` 被 drop 会 abort 全部。⚠ 更通用的取消机制是**结合 `select!` 与 `CancellationToken`**（下一篇细讲），它让任务自己在安全点退出，而不是被粗暴丢弃。

## 4.3 取消安全（cancellation safety）

不是所有 Future 都能被安全地中途丢弃：

```text
取消安全的例子：
  tokio::time::sleep       被 drop 即放弃计时，无副作用
  mpsc::Receiver::recv     丢掉后消息仍在队列，可重试

取消不安全的例子：
  AsyncReadExt::read_exact 已读部分数据可能丢失
  tokio::io::copy          中途取消会留下不完整状态
```

★ **关键认知**：**"取消安全"指 Future 被 drop 后不丢失数据、不破坏状态，可以安全重试**。在 `select!` 里反复创建又丢弃 Future 时，必须确认它是取消安全的，否则会静默丢数据。⚠ 这是异步编程最隐蔽的坑之一，下一篇会用专门一节展开。

# 5. spawn_blocking

## 5.1 为什么需要它

```rust
#[tokio::main]
async fn main() {
    // ✗ 反例：在异步任务里做同步阻塞 IO
    // 会把一个 worker 线程卡住，其他任务无法推进
    // let data = std::fs::read_to_string("big.txt").unwrap();

    // ✓ 正确：交给阻塞线程池
    let data = tokio::task::spawn_blocking(|| {
        std::fs::read_to_string("big.txt").unwrap()
    })
    .await
    .unwrap();

    println!("{} 字节", data.len());
}
```

★ **关键认知**：`spawn_blocking` 把闭包交给**专用的阻塞线程池**（默认上限 512），当前异步任务只是 `.await` 它的 `JoinHandle`，worker 线程立即被释放去跑别的任务。⚠ 阻塞线程池不是无限的：大量 `spawn_blocking` 会排队；对 CPU 密集任务，更好的选择是 rayon 或自建线程池，因为阻塞池的线程数与调度并不为计算优化。

## 5.2 三条纪律

```text
1. 只有真正的阻塞调用才 spawn_blocking（文件、同步网络、CPU 重活）
2. 不要在 spawn_blocking 里再 await（闭包是同步的）
3. 提交后尽早 await，避免无限堆积任务
```

★ **关键认知**：阻塞闭包必须是**同步**的，返回普通值；需要异步逻辑就留在外面。提交后尽快 `await`，否则任务会在阻塞池里越积越多，最终排队到超时。⚠ 阻塞池的线程数与异步 worker 是两套资源，别把 CPU 密集任务全塞进去，否则会把本该留给文件 IO 的线程占满。

# 6. 常见错误

```text
错误 1：spawn 后不 await，任务静默消失
  原因：进程退出或运行时关闭，任务被丢弃
  解决：需要结果就 await；不需要也要考虑 join 或记录

错误 2：借用了局部变量导致 'static 报错
  解决：move + Arc/克隆，把所有权带进任务

错误 3：用 Rc/RefCell 捕获进 spawn
  解决：换 Arc/Mutex，或改用 spawn_local + LocalSet

错误 4：长同步计算卡住 worker
  现象：其他任务延迟飙升，abort 也不生效
  解决：拆成小块 await，或 spawn_blocking
```

# 7. 练习

1. 用 `tokio::spawn` 启动 3 个不同耗时的任务，用 `JoinSet::join_next` 按完成顺序打印，体会"完成序 ≠ 启动序"。
2. 写一个会 panic 的任务，用 `handle.await` 捕获 `JoinError`，验证主进程不崩溃，并打印 `is_panic()`。
3. 起一个 10 秒任务，100ms 后 `abort()`，在任务里加 `println!` 观察它是否还会输出；再把它换成一段 10 秒的纯 CPU 循环，看 abort 是否立刻生效。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `spawn` | Future → 独立任务，立刻返回句柄 |
| `JoinHandle` | 外层 `JoinError` 表示 panic/取消，业务错误在 `T` 内 |
| `Send + 'static` | 任务可跨线程、可活得比调用者久 |
| drop 句柄 | 默认不取消，任务 detached |
| `abort` | 协作式取消，下一次调度时丢弃 |
| `JoinSet` | 动态任务池，按完成顺序收结果 |
| `spawn_blocking` | 阻塞代码隔离到专用线程池 |
| 取消安全 | drop 不丢数据、可重试，`select!` 前必查 |

下一篇进入通道与同步原语：`mpsc`/`oneshot`/`broadcast`/`watch` 四种通道怎么选，`Mutex`/`RwLock` 在异步里为什么长得不一样，`Semaphore` 如何做并发限流，以及死锁最常见的几种形态。
