真实服务里没有"无限等待"这回事：请求要超时、任务要能被叫停、多个事件源要抢着响应。Tokio 用 `select!` 做竞速，用 `timeout` 包住任意操作，而这一切都建立在"取消 = drop Future"这个简单却危险的语义上。这一篇讲清楚它们怎么用、取消安全为什么是生死线，以及超时后如何正确地重试。

# 1. select!：让多个 Future 赛跑

## 1.1 基本形态

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (_tx, mut rx) = mpsc::channel::<String>(8);

    tokio::select! {
        msg = rx.recv() => {
            println!("收到消息：{:?}", msg);
        }
        _ = sleep(Duration::from_millis(500)) => {
            println!("等太久了，超时");
        }
    }
}
```

`select!` 同时 poll 所有分支，**哪个先就绪就执行哪个**，然后丢弃其余分支的 Future。

★ **关键认知**：`select!` 是**公平的**（默认随机轮询顺序），不会因为写在前面的分支就先执行；它也不是并行——所有分支仍在同一个任务里被 poll。⚠ 最重要的一点：**没被选中的分支，其 Future 会被立即 drop**。这正是"取消"发生的地方，也是所有坑的源头。

## 1.2 带返回值的 select

```rust
let result = tokio::select! {
    v = rx.recv() => format!("消息 {:?}", v),
    _ = sleep(Duration::from_millis(100)) => "超时".to_string(),
};
println!("{}", result);
```

## 1.3 biased 与 else

```rust
// biased：严格按书写顺序检查，前面的优先
tokio::select! {
    biased;
    _ = high_priority() => println!("高优先级先处理"),
    _ = low_priority() => println!("低优先级"),
}
```

```rust
// else：所有分支都 disabled 时执行（少见，用于全部条件关闭）
tokio::select! {
    _ = rx.recv() => {}
    else => println!("所有分支都不可用"),
}
```

★ **关键认知**：默认随机化是为了**防止饿死**：如果永远优先第一个分支，高并发下后面的分支可能一直轮不到。`biased;` 把它变成确定性顺序，适合"必须先检查关闭信号"的场景。⚠ 用 `biased;` 时要想清楚：排在前面的分支如果长期就绪，后面的会被饿死。

## 1.4 用 select 实现"任一完成即返回"

把两个 `fetch` 放进 `select!`，谁先完成就取谁，另一个被 drop——这是"**对冲请求（hedged request）**"的原型。⚠ 被丢弃的 Future 真的被**取消**了，如果它已经写了半条数据就会出问题，所以要区分"纯读取"和"有副作用"的操作。

# 2. timeout：给任意 Future 加时间上限

## 2.1 两种写法

```rust
use tokio::time::{timeout, sleep, Duration};

// 写法 1：timeout 包住任意 Future，返回 Result<T, Elapsed>
match timeout(Duration::from_millis(100), sleep(Duration::from_secs(1))).await {
    Ok(()) => println!("按时完成"),
    Err(_) => println!("超时了"),   // Elapsed
}
```

```rust
// 写法 2：手写 select 版本，语义等价
tokio::select! {
    _ = sleep(Duration::from_secs(1)) => println!("完成"),
    _ = sleep(Duration::from_millis(100)) => println!("超时"),
}
```

★ **关键认知**：`timeout(dur, fut)` 返回 `Result<F::Output, Elapsed>`。它是**取消的一等公民**：超时发生时，内部 Future 被 drop，你得接受"操作可能只做了一半"。⚠ `timeout` 不会强杀任何东西，它只是"不再 poll 那个 Future"，如果那 Future 在做同步计算，timer 到点了也插不进去（因为没有 await 点）。

## 2.2 超时 + 业务错误的两层 Result

```rust
async fn query() -> Result<String, String> {
    Ok("data".to_string())
}

#[tokio::main]
async fn main() {
    let result = timeout(Duration::from_millis(100), query()).await;

    match result {
        Ok(Ok(data)) => println!("成功：{}", data),     // 按时 + 业务成功
        Ok(Err(e)) => println!("业务错误：{}", e),      // 按时 + 业务失败
        Err(_) => println!("超时"),                     // 超时
    }
}
```

★ **关键认知**：`Result<Result<T, E>, Elapsed>` 是超时场景的标准嵌套。⚠ 用 `?` 时要注意解包顺序：`timeout(...).await??` 会先传播 `Elapsed` 再传播业务错误，两者语义不同，别糊在一起。

## 2.3 用 timeout 防止无限等待

任何时候等待外部事件（通道、锁、网络）都应包一层 `timeout`，把"可能永远 Pending"变成"有限时间内返回"。⚠ 超时后要明确是放弃、重试还是降级，别让错误静默吞掉。

# 3. 取消安全：select 的生死线

## 3.1 取消到底做了什么

当 `select!` 选中某个分支，其余分支的 Future 被 **drop**。drop 一个 Future 的语义是：**丢弃它的状态机，不再 poll**。它不是"发一个取消信号"，而是"就地销毁"。

```text
select! 内部过程：

  同时 poll 分支 A、B、C
        │
        ├─ B 返回 Ready
        │
        ├─ drop A 的 Future
        ├─ drop C 的 Future
        └─ 执行 B 的处理分支
```

★ **关键认知**：Rust 没有"线程中断"式的取消，**取消 = drop**。这带来一个关键性质：**任何 Future 只要在 `.await` 点被 drop，就发生在那个 await 点**；如果没有 await 点，就拿它没办法。⚠ 因此"取消的粒度"取决于代码里 await 的密度。

## 3.2 取消安全与不安全

```text
取消安全（cancel-safe）：
  tokio::time::sleep            丢弃即可，无残留
  mpsc::Receiver::recv          未取到的消息仍在队列，可重试
  tokio::net::TcpStream::read   底层缓冲区语义由实现保证
  watch::Receiver::changed      只是等待，无副作用

取消不安全（cancel-unsafe）：
  AsyncReadExt::read_exact      已读的字节可能丢失
  AsyncWriteExt::write_all      可能只写了一部分
  tokio::io::copy               中断会留下不完整状态
  StreamExt::next 的某些组合    取决于流实现
```

★ **关键认知**：**"取消安全"不是"可以安全取消"，而是"取消后仍可重试且不丢数据"**。一个操作如果中途被 drop 会让状态不一致，它就是取消不安全的，不能放进 `select!` 的循环里反复创建。⚠ 最隐蔽的 bug 是：代码能编译、能跑、大部分时候对，只有在特定竞态下丢一条消息。

## 3.3 正确与错误示范

```rust
// ✗ 危险：read_exact 在循环里被 select 反复取消
loop {
    tokio::select! {
        r = reader.read_exact(&mut buf) => { /* 可能丢字节 */ }
        _ = shutdown.changed() => break,
    }
}
```

```rust
// ✓ 正确：用 read 累加，或用能记录进度的封装
loop {
    let n = tokio::select! {
        r = reader.read(&mut buf[cursor..]) => r?,
        _ = shutdown.changed() => break,
    };
    cursor += n;                       // 进度自己保存，不怕取消
    if cursor == buf.len() { break; }
}
```

★ **关键认知**：对抗取消不安全的通用手法是**把进度保存在 Future 之外**——每次只读一小段、自己累加 offset，这样即使某次调用被取消，进度也不会丢。⚠ `read_exact`、`write_all` 这类"要么全做要么全不做"的 API，本质上不适合被反复取消。

# 4. 结合 CancellationToken 做结构化取消

跨任务、跨层级的取消用 `CancellationToken`（`tokio-util`）：把它克隆给每个任务，任务在 `select!` 里监听 `token.cancelled()`，父级调用 `token.cancel()` 一次广播给所有克隆（`tokio-util = "0.7"`）。

★ **关键认知**：`CancellationToken` 是可克隆的**取消广播**：`cancel()` 后所有 `cancelled()` 的等待者同时就绪。它比 `watch::channel(bool)` 语义更明确，也支持父子 token（`child_token`）做层级取消。⚠ 它同样是**协作式**——任务必须在 `select!` 里监听 `cancelled()`，否则不会被中断。

# 5. 超时与重试

## 5.1 一个朴素的超时重试

```rust
use tokio::time::{sleep, timeout, Duration};
use std::future::Future;

async fn retry_with_timeout<F, Fut, T, E>(
    mut op: F,
    attempts: u32,
    per_try: Duration,
) -> Result<T, String>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut last_err = String::from("未执行");
    for i in 0..attempts {
        match timeout(per_try, op()).await {
            Ok(Ok(v)) => return Ok(v),                 // 成功
            Ok(Err(e)) => last_err = e.to_string(),    // 业务失败，可重试
            Err(_) => last_err = "超时".to_string(),   // 超时，可重试
        }
        // 指数退避：100ms, 200ms, 400ms ...
        let backoff = Duration::from_millis(100 * 2u64.pow(i));
        sleep(backoff).await;
    }
    Err(last_err)
}
```

★ **关键认知**：重试有两个前提：**（1）操作是幂等的**（重复执行不会产生副作用），**（2）失败是暂时的**（网络抖动、限流）。对"扣款""下单"这类非幂等操作，盲目重试会造成重复扣款。⚠ 重试还要有**退避 + 抖动**，否则大量客户端同时重试会把下游打垮（惊群）。

## 5.2 只对特定错误重试

```rust
#[derive(Debug)]
enum MyError {
    Transient(String),   // 可重试：超时、连接重置
    Permanent(String),   // 不可重试：参数错误、404
}

fn is_retryable(e: &MyError) -> bool {
    matches!(e, MyError::Transient(_))
}
```

```text
重试决策表：

  错误类型            重试？      说明
  连接超时            是          典型瞬时故障
  连接被重置          是          对端重启/网络抖动
  429 限流            是（退避）  必须尊重 Retry-After
  400 参数错误        否          重试也一样
  401/403 鉴权失败    否          除非刷新了凭证
  500 服务端错误      视情况      需结合幂等性判断
```

★ **关键认知**：**"重试所有错误"是反模式**。对永久性错误重试只会放大故障、浪费配额。⚠ 重试层应该放在"知道错误语义"的地方，而不是无脑包在最外层。

## 5.3 用 select 做总超时（跨多次重试）

单次尝试有超时还不够：多次重试累加起来仍可能失控。做法是在最外层再包一个 `timeout(总时长, async { 重试循环 })`，形成"单次超时 + 总超时"的双保险。

# 6. 常见错误

```text
错误 1：把取消不安全的 Future 放进 select 循环
  现象：偶发丢数据、重复写入
  解决：改用取消安全 API，或把进度存到 Future 外

错误 2：以为 timeout 能中断同步计算
  现象：设了 100ms，任务跑了 10s 才返回
  原因：没有 await 点，drop 插不进去
  解决：spawn_blocking 或拆分 await

错误 3：重试非幂等操作
  现象：重复扣款、重复下单
  解决：先做幂等（幂等键/去重表），再谈重试

错误 4：用 biased; 后某分支饿死
  解决：确认分支优先级合理，或去掉 biased

错误 5：忘记处理 timeout 的 Elapsed
  现象：用 ? 把超时当成业务错误传播
  解决：显式 match 两层 Result
```

# 7. 练习

1. 用 `select!` 让两个不同耗时的 HTTP 请求赛跑（可用 `sleep` 模拟），取最快者，并思考被丢弃分支的副作用。
2. 写一个"最多重试 3 次、每次 200ms 超时、指数退避"的函数，故意让第 2 次成功，打印每次尝试的结果。
3. 用 `CancellationToken` 起 3 个任务，200ms 后统一取消；再把其中一个任务换成"纯 CPU 循环、无 await"，观察它是否响应取消。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `select!` | 多 Future 竞速，未选中者被 drop |
| 公平性 | 默认随机，`biased;` 变确定性 |
| `timeout` | `Result<T, Elapsed>`，取消内部 Future |
| 取消语义 | 取消 = drop，只在 await 点生效 |
| 取消安全 | drop 后不丢数据、可重试 |
| `CancellationToken` | 可克隆的取消广播，协作式 |
| 重试 | 需幂等 + 退避 + 只重试瞬时错误 |
| 总超时 | 外层再包一层 timeout |

下一篇进入错误处理：`Result` 与 `?` 在异步函数里的行为、`anyhow` 与 `thiserror` 如何分工、错误上下文的叠加、跨任务边界的错误传播，以及如何设计一个既好调试又好分类的错误类型。
