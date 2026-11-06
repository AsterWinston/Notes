前面八篇把 Tokio 的零件拆了个遍，这一篇做两件事：把高频的坑汇总成一份可对照的清单，再把"怎么写出能上生产的异步服务"讲成几条可执行的原则。内容偏经验，但每条背后都对应前面讲过的机制。

# 1. 绝不阻塞运行时

## 1.1 阻塞的四种来源

```text
1. 标准库同步 IO        std::fs / std::net / std::process
2. 同步睡眠             std::thread::sleep
3. 同步锁长时间持有      std::sync::Mutex 跨 await（编译期会拦）
4. CPU 密集计算         大循环、哈希、加解密、图像处理
```

★ **关键认知**：异步运行时的 worker 数量是有限的（默认 = CPU 核数）。**一个 worker 被阻塞，就少一份并行度**；所有 worker 都被阻塞，整个服务停摆——包括心跳、超时、关闭信号。⚠ 最危险的是"偶发才阻塞"的代码：平时看不出来，流量高峰时集中爆发，表现为"延迟突然飙升但 CPU 不高"。

## 1.2 正确的隔离方式

| 阻塞类型 | 方案 |
|---------|------|
| 文件 / 同步网络 IO | `tokio::fs` / `tokio::net`，或 `spawn_blocking` |
| 同步睡眠 | `tokio::time::sleep` |
| CPU 密集 | `spawn_blocking` 或 rayon 线程池 |
| 长同步锁 | 缩短临界区，或换 `tokio::sync::Mutex` |

```rust
// 反例：CPU 密集任务直接放在 async fn 里
async fn bad_hash(data: Vec<u8>) -> u64 {
    let mut h = 0u64;
    for b in data { h = h.wrapping_mul(31).wrapping_add(b as u64); }   // 卡住 worker
    h
}

// 正例：交给阻塞线程池
async fn good_hash(data: Vec<u8>) -> u64 {
    tokio::task::spawn_blocking(move || {
        let mut h = 0u64;
        for b in data { h = h.wrapping_mul(31).wrapping_add(b as u64); }
        h
    })
    .await
    .unwrap()
}
```

★ **关键认知**：`spawn_blocking` 的线程池默认上限 512，且**不参与异步调度**。它适合"少量、粒度粗"的阻塞操作；如果每天要跑几万个短小的 CPU 任务，更适合用 rayon 的 work-stealing 线程池，而不是把阻塞池打满。⚠ 注意阻塞池里的任务不能被 abort，取消只能靠任务自己检查标志位。

## 1.3 一个自动检测手段

安装 `cargo install tokio-console`，开启 `tokio` 的 `tracing` feature 与 `console-subscriber`，在 `main` 里调用 `console_subscriber::init()`，即可实时观察任务耗时与等待。

★ **关键认知**：阻塞 bug 靠读代码很难发现，**靠观测才抓得住**。`tokio-console` 能实时看到哪些任务长时间占用 worker、哪些任务一直 Pending，是把"玄学延迟"变成"可定位问题"的关键工具。

# 2. 任务泄漏

## 2.1 什么叫泄漏

```text
任务泄漏：spawn 出去的任务永远不会结束，也没人取消它，
          它会一直占着内存、文件描述符、连接，直到进程退出。
```

```rust
// 典型泄漏：无限循环 + 没人持有句柄
fn start_background() {
    tokio::spawn(async {
        loop {
            tokio::time::sleep(Duration::from_secs(1)).await;
            // 永远跑下去
        }
    });   // 句柄被丢弃，无法取消
}
```

★ **关键认知**：`spawn` 出来的任务是**detached** 的——drop `JoinHandle` 不会停止它。所以"后台任务"必须有一个明确的**生命周期所有者**：要么由某个结构体持有 `JoinHandle` 并在 drop 时 `abort`，要么监听统一的 `CancellationToken`。⚠ 一层循环里反复 `spawn` 而不取消，是最常见的泄漏形态，进程内存会缓慢上涨。

## 2.2 三种治理方式

```rust
// 方式 1：用结构体持有句柄，Drop 时 abort
struct Worker {
    handle: tokio::task::JoinHandle<()>,
}

impl Drop for Worker {
    fn drop(&mut self) {
        self.handle.abort();
    }
}
```

```rust
// 方式 2：CancellationToken 统一取消
let token = CancellationToken::new();
let t = token.clone();
tokio::spawn(async move {
    tokio::select! {
        _ = t.cancelled() => {}
        _ = run_forever() => {}
    }
});
// 关闭时
token.cancel();
```

```rust
// 方式 3：JoinSet 统一管理，drop 即全部 abort
let mut set = JoinSet::new();
set.spawn(async { /* ... */ });
// set 离开作用域，所有任务被取消
```

★ **关键认知**：**"谁能 spawn，谁就要负责收尸"**。把任务句柄集中到一个作用域（`JoinSet`、结构体字段、token 的拥有者），取消路径才是完整可审计的。⚠ 不要依赖"进程退出时会清理"——长驻服务不会退出，泄漏会一直累积。

★ **关键认知**：给长期任务套一层 `timeout` 是最后一道保险——即使忘了取消，也能在超时后让任务自然结束，避免永久占用资源。

# 3. 背压：别让生产者冲垮消费者

## 3.1 背压是什么

```text
无背压：
  生产者 ──快速──▶ [队列无限增长] ──慢速──▶ 消费者
                          ↑
                    内存持续上涨，最终 OOM

有背压：
  生产者 ──send().await──▶ [有界队列]
              ↑ 满了就等      │
              └───────────────┘ 消费者腾出空位
```

★ **关键认知**：背压的本质是**把"压力"传回上游，让上游慢下来**，而不是把压力转成内存。异步的 `await` 正好天然支持这一点：`send` 在有界队列满时挂起，生产者自动降速。⚠ 无界队列（`unbounded_channel`）是把背压关掉，只适合"生产速率恒定且可证明有界"的场景，否则就是内存炸弹。

## 3.2 背压的三个层次

| 层次 | 手段 | 效果 |
|------|------|------|
| 队列 | 有界 `mpsc` | 缓冲满则阻塞生产者 |
| 并发 | `Semaphore` | 限制同时进行的操作数 |
| 请求 | 超时 + 限流 | 拒绝或降级超额请求 |

```rust
// 队列 + 并发双重限流
let (tx, mut rx) = mpsc::channel::<Job>(100);        // 最多排队 100
let sem = Arc::new(Semaphore::new(16));              // 同时处理 16 个

tokio::spawn(async move {
    while let Some(job) = rx.recv().await {
        let permit = sem.clone().acquire_owned().await.unwrap();
        tokio::spawn(async move {
            process(job).await;
            drop(permit);
        });
    }
});
```

★ **关键认知**：**队列长度限制"等待"，信号量限制"在跑"**，两者解决不同问题。只有队列没信号量，会同时跑起一堆任务把下游打爆；只有信号量没队列，生产者在 `acquire` 处排队，压力照样传回上游——通常两者都要。⚠ 背压反噬时要给"等待"加超时，否则上游会无限期卡住。

## 3.3 丢弃策略

队列满时三种选择：阻塞生产者（最常用）、丢弃新消息（可丢的事件流）、丢弃旧消息（只关心最新状态）。

# 4. 可观测性

## 4.1 异步服务要观测什么

| 指标 | 说明 | 异常信号 |
|------|------|---------|
| 任务数 | 活跃任务总量 | 持续上涨 = 泄漏 |
| 任务等待时长 | 从入队到被 poll | 变长 = worker 不足或阻塞 |
| 队列深度 | 通道积压 | 上涨 = 消费跟不上 |
| 延迟分位 | P50/P99/P999 | P99 飙升 = 有阻塞或锁竞争 |
| 取消率 | abort / 超时占比 | 上涨 = 下游变慢 |

★ **关键认知**：异步系统的故障常常表现为"**某个指标的天花板**"——比如 worker 全被阻塞时，CPU 不高但延迟爆表。所以观测要成对看：**延迟 + CPU + 队列深度**，单看一个指标会误判。⚠ P99 比平均值重要得多：异步的尾延迟会被调度、锁、GC 式的停顿放大。

## 4.2 tracing：结构化日志

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tracing::{info, instrument};

#[instrument(skip(db))]
async fn handle_request(id: u64, db: &Db) -> Result<String, String> {
    info!("开始处理请求");
    let user = db.find(id).await.map_err(|e| e.to_string())?;
    info!(user = %user, "处理完成");
    Ok(user)
}
```

★ **关键认知**：`#[instrument]` 会自动把函数参数、执行时长、错误记录进 span，异步调用链因此可以"串起来"——这是异步排障的核心能力。⚠ 异步任务的日志容易交错，必须靠 span 关联；纯 `println!` 在多任务下会变成一团乱麻。

## 4.3 用 span 跟踪跨任务链路

★ **关键认知**：`spawn` 会**丢失当前 span**（因为换了任务），必须用 `.instrument(tracing::info_span!("job", id = 1))` 手动把上下文带过去。⚠ 这又是一个"任务边界切断上下文"的例子，和错误传播、取消是同一类问题。

# 5. 常见坑速查表

| 坑 | 症状 | 修法 |
|----|------|------|
| 异步里跑阻塞代码 | 延迟高、CPU 低 | `spawn_blocking` / 异步 API |
| 忘了 `.await` | 逻辑不执行 | 加 await 或 spawn |
| `std::sync::Mutex` 跨 await | 编译错误 | 缩小临界区 / tokio Mutex |
| `unbounded_channel` | 内存涨 | 换有界通道 |
| drop 句柄当取消 | 任务继续跑 | `abort` / token |
| `read_exact` 放 select 循环 | 偶发丢数据 | 存进度 / 换 API |
| `timeout` 想中断同步计算 | 超时不生效 | `spawn_blocking` |
| 重试非幂等操作 | 重复写入 | 幂等键 |
| 测试用真实 sleep | CI 偶发失败 | `start_paused` / 信号 |
| spawn 后丢 span | 日志断链 | `.instrument()` |
| 运行时嵌套 | panic | 不在 async 里 block_on |
| 忘记 drop(tx) | 接收循环不结束 | 显式 drop |

★ **关键认知**：这张表里的每一条，前八篇都展开讲过。**异步的坑几乎都来自同一件事：任务边界**——边界会切断所有权、切断借用、切断错误链、切断日志上下文、切断取消。凡是要"跨边界"的东西，都得显式处理。

# 6. 一套可复用的工程结构

## 6.1 优雅关闭骨架

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let token = tokio_util::sync::CancellationToken::new();
    let server = tokio::spawn(run_server(token.clone()));

    tokio::signal::ctrl_c().await?;       // 等 Ctrl+C
    token.cancel();                       // 通知所有任务停止接收

    // 给在途请求最多 10 秒收尾
    let _ = tokio::time::timeout(std::time::Duration::from_secs(10), server).await;
    Ok(())
}
```

★ **关键认知**：优雅关闭的要点是**分层**：先停接收（不再接新请求），再等在途请求完成，最后强制超时。⚠ 别一上来就 `abort` 所有任务——那会把正在写的数据截断，比不关闭还糟。给"在途"留时间，给"总时长"设上限。

# 7. 设计检查清单

```text
启动期
  [ ] 运行时 flavor 明确（多线程 / 单线程）
  [ ] tracing / metrics 已初始化
  [ ] 配置错误能在启动时暴露，而不是运行期

运行期
  [ ] 所有阻塞调用都有出处（异步 API 或 spawn_blocking）
  [ ] 每个 spawn 都有生命周期所有者
  [ ] 队列有界，并发有信号量
  [ ] 每个外部调用有超时
  [ ] 重试幂等 + 退避 + 只重试瞬时错误

关闭期
  [ ] 统一取消信号（token），先停接收、再等在途、最后硬超时
  [ ] 资源（连接、文件）显式释放
```

★ **关键认知**：异步代码的正确性很难靠"读一遍"保证，靠的是**把这套清单变成 review 时的检查项**。⚠ 清单不必一次全做到，但每一项缺失都要能说清"为什么可以缺"，而不是"忘了"。

# 8. 练习

1. 找一段在 async 里做同步文件读的代码，改成 `tokio::fs` 或 `spawn_blocking`，用 `tokio-console` 前后对比任务耗时。
2. 写一个会泄漏的后台循环，改用 `CancellationToken` + 结构体持有句柄，验证进程退出前任务能被干净取消。
3. 给一个"生产者-队列-消费者"链路同时加上有界 `mpsc(100)` 和 `Semaphore(16)`，压测观察内存与延迟是否平稳。

# 9. 小结

| 原则 | 一句话 |
|------|--------|
| 不阻塞 | 阻塞代码一律隔离到专用线程池 |
| 任务有主 | 谁 spawn，谁持有句柄与取消权 |
| 背压 | 有界队列 + 信号量，把压力传回上游 |
| 可观测 | 延迟、CPU、队列、任务数成对看 |
| span 贯穿 | 跨任务用 `.instrument()` 带上下文 |
| 优雅关闭 | 停接收 → 等在途 → 硬超时 |
| 边界意识 | 所有权、错误、取消、日志都要显式跨边界 |

## 9.1 全模块回顾

```text
1  async 基础     同步 vs 异步、Future 惰性、await 暂停
2  Future/poll    poll、Waker、Pin、executor/reactor
3  运行时         #[tokio::main]、flavor、block_on、边界
4  任务           spawn、JoinHandle、Send+'static、abort
5  通道/同步       mpsc/oneshot/broadcast/watch、锁、信号量
6  超时/取消       select!、timeout、取消安全、重试
7  错误           Result/?、anyhow/thiserror、跨任务传播
8  测试           #[tokio::test]、超时、并发、mock、确定性
9  工程实践       不阻塞、防泄漏、背压、观测、关闭
```

★ **关键认知**：学完这九篇，你真正该带走的能力不是"记住 API"，而是**看到一段异步代码时能问出正确的问题**：它在哪个运行时上跑？哪里会阻塞？任务的生命周期归谁？取消发生在哪个 await 点？错误和日志能不能跨任务追溯？这些问题问对了，坑就少了一大半。

⚠ 最后提醒一句：**Tokio 不是"异步"的同义词**。异步是语言层的 `Future` 模型，Tokio 只是最流行的一种运行时实现。理解了 `poll`/`Waker`/`Pin`，即使将来换 async-std、smol，或写自己的执行器，底层心智模型依然通用。
