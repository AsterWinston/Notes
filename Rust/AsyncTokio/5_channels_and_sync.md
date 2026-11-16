任务一旦被 spawn 出去，就各自为政了。它们之间怎么传数据、怎么共享状态、怎么限制并发数，是写异步服务绕不开的问题。Tokio 提供了四种风格迥异的通道和三种常用同步原语，选错了不仅性能差，还会引入死锁。这一篇把它们讲清楚，并给出选型表和死锁的典型形态。

# 1. 四种通道：先看选型表

| 通道 | 方向 | 容量 | 典型场景 |
|------|------|------|---------|
| `mpsc` | 多生产者 → 单消费者 | 有界 / 无界 | 任务队列、事件汇聚 |
| `oneshot` | 单生产者 → 单消费者 | 1 | 请求-响应、一次性结果回传 |
| `broadcast` | 多生产者 → 多消费者 | 有界 | 事件广播、配置下发 |
| `watch` | 单生产者 → 多消费者 | 1（只留最新） | 状态订阅、优雅关闭信号 |

★ **关键认知**：选通道的第一问是"**消费者有几个、要不要保留历史**"。单消费者、每条消息都要处理 → `mpsc`；一次性回传结果 → `oneshot`；多个订阅者都要收到每一条 → `broadcast`；订阅者只关心"最新值" → `watch`。⚠ 用 `broadcast` 存状态、用 `watch` 传事件，都是典型的错配，会在慢消费者或丢消息上付出代价。

# 2. mpsc：最常用的任务通道

## 2.1 基本用法

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // 有界通道：缓冲区满时 send 会等待（形成背压）
    let (tx, mut rx) = mpsc::channel::<String>(8);

    tokio::spawn(async move {
        for i in 0..3 {
            tx.send(format!("消息 {}", i)).await.unwrap();
        }
        // tx 在这里 drop，接收端会收到 None，循环自然结束
    });

    while let Some(msg) = rx.recv().await {
        println!("收到：{}", msg);
    }
}
```

★ **关键认知**：`mpsc::channel(n)` 的 `n` 是**缓冲区容量**，`send` 在缓冲满时会 `await` 等待空位——这就是**背压**：生产快于消费时，生产者被自然限速，而不是把内存吃爆。⚠ 无界通道 `mpsc::unbounded_channel()` 的 `send` 永不阻塞，看似方便，实则是内存泄漏的温床，生产环境应优先用有界通道。

## 2.2 多生产者与关闭语义

```rust
let (tx, mut rx) = mpsc::channel::<i32>(16);

for id in 0..3 {
    let tx = tx.clone();          // 每个生产者克隆一份发送端
    tokio::spawn(async move {
        tx.send(id * 10).await.unwrap();
    });
}
drop(tx);                          // 丢掉原始 tx，否则 rx 永远等不到 None

while let Some(v) = rx.recv().await {
    println!("{}", v);
}
```

★ **关键认知**：接收端在所有发送端都被 drop 后才会返回 `None`。忘掉 `drop(tx)` 是"循环不退出"的头号原因。⚠ 克隆发送端很便宜，但每个克隆都持有通道一半的所有权；只要还有任何一个 `Sender` 活着，通道就不会关闭。

## 2.3 发送端的几种变体

```text
tx.send(v).await      // 缓冲区满时等待（异步，有背压）
tx.try_send(v)        // 不等待，满则返回 TrySendError::Full
tx.reserve().await    // 先占一个位置，再在需要时 send（适合批量）
tx.blocking_send(v)   // 同步阻塞，只能在非 async 上下文用
```

★ **关键认知**：`try_send` 适合"丢旧留新"或"满了就降级"的场景；`reserve` 适合先拿到许可再构造消息，避免"构造完才发现发不出去"。⚠ 在 `async` 函数里调用 `blocking_send` 会卡住 worker，和直接用阻塞调用一样危险。

# 3. oneshot：一次性的结果回传

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<u32>();

    tokio::spawn(async move {
        let result = 21 * 2;
        let _ = tx.send(result);      // 只能发一次，发完通道关闭
    });

    let value = rx.await.unwrap();    // 接收端 await 拿结果
    println!("{}", value);
}
```

★ **关键认知**：`oneshot` 是"**把一个任务的结果送回等待方**"的标准工具，语义接近"Future 的手动版"。常用于请求-响应、后台任务完成通知。⚠ `rx.await` 返回 `Result<T, RecvError>`，发送端被 drop 而未发送时会得到 `Err`，必须处理，否则 `.unwrap()` 会在发送端提前退出时 panic。

| 方式 | 结果类型 | 适用 |
|------|---------|------|
| `JoinHandle` | `Result<T, JoinError>` | 你 spawn 的任务，想拿到返回值/感知 panic |
| `oneshot` | `Result<T, RecvError>` | 任务不由你 spawn，或只想发一次值 |
| `mpsc` | 流式多条 | 持续产出 |

# 4. broadcast：一对多广播

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, _rx0) = broadcast::channel::<String>(16);

    // 每个订阅者调用 subscribe 得到独立接收端
    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    tokio::spawn(async move {
        tx.send("事件 A".to_string()).unwrap();
        tx.send("事件 B".to_string()).unwrap();
    });

    println!("订阅者 1：{:?}", rx1.recv().await);
    println!("订阅者 2：{:?}", rx2.recv().await);
}
```

★ **关键认知**：`broadcast` 给每个订阅者维护一个**独立的游标**，各自从头读，互不影响。但缓冲区是环形的——**慢订阅者会落后**，当它落后超过容量时，`recv` 会返回 `RecvError::Lagged(n)`，表示"你错过了 n 条"。⚠ 这不是 bug，是广播的固有权衡：要么加大缓冲，要么把慢消费者踢掉，不能指望它不丢消息。

接收端必须 `match` 出 `Lagged(n)` 并明确丢弃策略，否则会把"漏读"误判成"没消息"。

# 5. watch：只关心最新值

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel(0);   // 初始值 0

    tokio::spawn(async move {
        for i in 1..=3 {
            tx.send(i).unwrap();            // 覆盖式更新，只保留最新
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
        }
    });

    // changed() 等待"值发生了变化"
    while rx.changed().await.is_ok() {
        println!("最新值：{}", *rx.borrow());
    }
}
```

★ **关键认知**：`watch` 的接收端**不会排队**，它永远只看到最新值；中间值可能被跳过。这正适合"配置热更新""关闭信号""进度百分比"这类**状态**，而不适合"每一条都要处理"的**事件**。⚠ 用 `watch` 传任务队列会丢任务，这是选型时最容易犯的错。

## 5.1 优雅关闭的经典组合

把 `watch::channel(false)` 的接收端克隆给每个任务，任务在 `select!` 里等 `rx.changed()`；关闭时 `shutdown_tx.send(true)` 一次通知全部订阅者。要点是**广播一个状态变化**，而不是给每个任务单独发消息。⚠ 接收端要先核对当前值，避免"订阅前已经关闭"导致漏判。

# 6. Mutex 与 RwLock：异步版有什么不同

## 6.1 为什么不能用 std::sync::Mutex 跨 await

```rust
// ✗ 危险：std Mutex 的 guard 不是 Send，跨 await 持有会编译失败
async fn bad(m: &std::sync::Mutex<i32>) {
    let mut guard = m.lock().unwrap();
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    *guard += 1;
}

// ✓ 正确：tokio::sync::Mutex 的 guard 可以跨 await
async fn good(m: &tokio::sync::Mutex<i32>) {
    let mut guard = m.lock().await;
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    *guard += 1;
}
```

★ **关键认知**：`std::sync::Mutex` 的 `lock()` 是阻塞的，它的 guard 跨 `await` 会违反 `Send`（编译器会拦下）。`tokio::sync::Mutex` 的 `lock().await` 在锁被占用时**挂起任务而非阻塞线程**，因此适合"持锁期间要 await"的场景。⚠ 但这不代表 tokio Mutex 更好：**不跨 await 的临界区，用 std Mutex 更快**（无异步开销）。

| 锁 | `lock` 形态 | 适用 |
|----|------------|------|
| `std::sync::Mutex` | 同步，阻塞 | 临界区无 await，纯内存操作 |
| `tokio::sync::Mutex` | 异步，挂起 | 临界区必须跨 await |
| `std::sync::RwLock` | 同步 | 读多写少，无 await |
| `tokio::sync::RwLock` | 异步 | 读多写少，跨 await |

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    handles.push(tokio::spawn(async move {
        let mut guard = counter.lock().await;
        *guard += 1;
    }));
}

for h in handles {
    h.await.unwrap();
}
println!("{}", *counter.lock().await);   // 10
```

★ **关键认知**：`tokio::sync::RwLock` 默认**写者优先**（避免读者饿死写者），而 `std::sync::RwLock` 的平台实现差异较大。⚠ `tokio` 的 `RwLock` 在"只读、不 await"场景下比 std 版慢不少，别默认全用 tokio 版。

## 6.2 用 std Mutex 的正确姿势：缩小临界区

把 guard 关进 `{}` 块，让它在 `await` 之前 drop，是"既要 std Mutex 性能、又要 await"的标准手法。⚠ 要注意"检查-然后-操作"的竞态：两个任务可能同时未命中、同时 fetch，最后都写入——这属于逻辑竞态，锁本身救不了，需要额外去重。

# 7. Semaphore：并发限流

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let sem = Arc::new(Semaphore::new(3));   // 同时最多 3 个任务
    let mut handles = vec![];

    for i in 0..10 {
        let permit = sem.clone().acquire_owned().await.unwrap();
        handles.push(tokio::spawn(async move {
            println!("任务 {} 开始", i);
            tokio::time::sleep(std::time::Duration::from_millis(200)).await;
            drop(permit);                     // 释放许可，允许下一个进入
        }));
    }

    for h in handles {
        h.await.unwrap();
    }
}
```

★ **关键认知**：`Semaphore` 是**计数信号量**，`acquire` 拿到一个许可（permit）才能继续，permit 被 drop 时自动归还。`acquire_owned` 返回 `OwnedSemaphorePermit`，可以 move 进 `spawn` 的任务。⚠ 忘记 drop permit（或把它存进长生命周期结构）会导致许可泄漏，最终所有任务卡在 `acquire`。

| 手段 | 限的是什么 | 场景 |
|------|-----------|------|
| 有界 `mpsc` | 队列里待处理的消息数 | 生产者-消费者解耦 |
| `Semaphore` | 同时进行的操作数 | 限流、连接池、并发下载 |

# 8. 死锁的典型形态

```text
形态 1：同一任务重复锁同一把非重入锁
  let g1 = m.lock().await;
  let g2 = m.lock().await;   // ✗ 永远等不到

形态 2：两个任务以相反顺序拿两把锁
  任务 A: lock(x) → lock(y)
  任务 B: lock(y) → lock(x)   // ✗ 循环等待

形态 3：持锁 await 一个依赖该锁的任务
  持锁 → await 需要同一把锁的任务   // ✗ 自锁

形态 4：通道互等
  任务 A 等 B 的消息，B 等 A 的消息，双方都不发送
```

★ **关键认知**：异步死锁比同步更难发现，因为任务不是"卡在系统调用里"，而是"卡在 `Pending`"，CPU 看起来空闲、程序也不报错。⚠ 三条纪律能挡掉大部分问题：**（1）固定加锁顺序；（2）持锁期间不做 await（除非必要）；（3）用 `try_lock` + 超时做兜底探测**。

# 9. 常见错误

```text
错误 1：无界通道导致内存暴涨
  解决：优先有界通道，形成背压

错误 2：忘记 drop(tx)，接收循环永不结束
  解决：显式 drop 或让 tx 在作用域结束时自然释放

错误 3：用 broadcast 却不处理 Lagged
  解决：match 三种 RecvError，明确丢弃策略

错误 4：用 watch 当消息队列，任务丢失
  解决：watch 只传状态，事件用 mpsc
```

# 10. 练习

1. 用有界 `mpsc(2)` 搭一个生产者（发 100 条）与消费者（每条睡 10ms），打印生产者在何处被背压卡住。
2. 用 `broadcast(4)` 起一个快订阅者和一个慢订阅者，让慢的睡久一点，观察 `Lagged` 错误并打印丢失条数。
3. 用 `Semaphore(5)` 模拟 20 个并发下载，统计同时在跑的任务数不超过 5；再故意漏掉 drop permit，观察后续任务为何全部卡死。

# 11. 小结

| 原语 | 关键点 |
|------|--------|
| `mpsc` | 多对一，有界形成背压 |
| `oneshot` | 一次性结果回传 |
| `broadcast` | 一对多，慢订阅者会 Lagged |
| `watch` | 只留最新值，适合状态/关闭信号 |
| `tokio::Mutex` | 跨 await 持锁时用，否则 std 更快 |
| `RwLock` | 读多写少，注意写者优先策略 |
| `Semaphore` | 并发限流，permit 自动归还 |
| 死锁 | 固定顺序、缩短持锁、加超时兜底 |

下一篇进入超时与取消：`select!` 如何在多个 Future 间竞速、`timeout` 怎样包住任意操作、为什么"取消安全"决定了 `select!` 能否重试，以及超时后如何做退避重试。通道与取消结合，才能写出真正健壮的服务。
