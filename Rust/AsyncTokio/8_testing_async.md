异步代码的测试有三类典型故障：测试函数不会跑（没有运行时）、测试偶发通过（依赖时序）、测试偶尔卡死（等一个永不到来的消息）。这一篇把 `#[tokio::test]` 的机制讲清楚，并给出超时、并发、mock、确定性四方面的实操套路，让异步测试从"跑三次过一次"变成"稳定可复现"。

# 1. #[tokio::test] 做了什么

## 1.1 最小示例与展开

```rust
async fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[tokio::test]
async fn test_add() {
    assert_eq!(add(1, 2).await, 3);
}
```

```toml
[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

它展开后相当于一个 `#[test]`，内部用 `Builder::new_current_thread().enable_all().build()` 建运行时再 `block_on`。

★ **关键认知**：`#[tokio::test]` 是属性宏，展开后为每个测试函数**新建一个 current-thread 运行时**并 `block_on` 测试体。它是同步 `#[test]` 的异步版本——没有它，`async fn` 测试根本无法被测试框架驱动。⚠ 每个测试默认独立运行时、独立线程，所以测试之间不会共享运行时状态；这是好事，但也意味着不能靠"上一个测试初始化了什么"。

★ **关键认知**：默认用 `current_thread` 是有意的——测试追求**确定性**，单线程下任务不会真并行，时序更容易复现。⚠ 如果测试的正是多线程调度行为，显式指定 `#[tokio::test(flavor = "multi_thread", worker_threads = 4)]`。

## 1.2 常用变体

```rust
// 多线程运行时
#[tokio::test(flavor = "multi_thread", worker_threads = 4)]
async fn test_multi() { /* ... */ }

// 测试函数返回 Result，用 ? 传播错误
#[tokio::test]
async fn test_result() -> Result<(), Box<dyn std::error::Error>> {
    let v: i32 = "42".parse()?;
    assert_eq!(v, 42);
    Ok(())
}
```

★ **关键认知**：测试函数返回 `Result` 时，`Err` 会让测试失败并打印错误，这比到处 `.unwrap()` 更适合"被测代码本身返回 Result"的场景。⚠ `#[tokio::test]` 与 `#[should_panic]` 组合时，返回 `Result` 的测试在 panic 语义上有细微差别，多数情况下优先用断言。

# 2. 给测试加超时：防止挂死

## 2.1 问题的来源

```rust
#[tokio::test]
async fn test_hangs() {
    // 如果被测代码等一个永不到来的消息，测试会一直挂着
    let (_tx, mut rx) = tokio::sync::mpsc::channel::<i32>(1);
    let v = rx.recv().await;      // 永远 Pending，CI 卡死
    assert!(v.is_none());
}
```

★ **关键认知**：`cargo test` 默认**没有超时**，一个挂住的异步测试会让整个 CI 卡到人工介入。⚠ 异步测试必须自带超时兜底，这不是可选项——尤其是在等待通道、锁、网络 mock 的场景。

## 2.2 两种超时方案

```rust
use tokio::time::{timeout, Duration};

#[tokio::test]
async fn test_with_timeout() {
    let fut = async {
        tokio::time::sleep(Duration::from_millis(10)).await;
        42
    };

    let v = timeout(Duration::from_secs(1), fut)
        .await
        .expect("测试超时，可能有死锁或未唤醒");
    assert_eq!(v, 42);
}
```

```toml
# 方案 2：给整个测试二进制设置超时（nextest 支持更好）
# .config/nextest.toml
# [profile.default]
# slow-timeout = { period = "60s", terminate-after = 2 }
```

★ **关键认知**：**用 `tokio::time::timeout` 而不是自己造看门狗线程**，因为测试运行在 Tokio 虚拟时间上时，timeout 可以被加速（见 4.2）。⚠ 如果整个测试体都需要保护，把测试体包进 `async {}` 再 `timeout` 包住最省事。

# 3. 并发测试

## 3.1 验证"真的并发了"

```rust
use std::time::Instant;
use tokio::time::{sleep, Duration};

#[tokio::test]
async fn test_concurrent_not_serial() {
    let start = Instant::now();

    let (a, b, c) = tokio::join!(
        sleep(Duration::from_millis(200)),
        sleep(Duration::from_millis(200)),
        sleep(Duration::from_millis(200)),
    );

    let elapsed = start.elapsed();
    assert!(elapsed < Duration::from_millis(500), "本应并发，实际串行：{:?}", elapsed);
}
```

★ **关键认知**：并发测试的断言要留足**余量**（这里 200×3=600ms，断言 <500ms），因为 CI 机器慢、调度有抖动。⚠ 用 `Instant::now()` 做严格计时断言是**不稳定测试**的常见来源；更稳的做法是用虚拟时间，或用计数/信号验证顺序，而不是卡死具体毫秒。

## 3.2 测试锁与竞态

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::test(flavor = "multi_thread", worker_threads = 4)]
async fn test_shared_counter() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..100 {
        let c = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            let mut g = c.lock().await;
            *g += 1;
        }));
    }

    for h in handles {
        h.await.unwrap();
    }

    assert_eq!(*counter.lock().await, 100);
}
```

★ **关键认知**：测并发正确性要**放大竞态窗口**：任务数够多、worker 够多、临界区里加点 await，才能把"忘记加锁"暴露出来。⚠ 单次运行没挂不代表没问题——这类测试要多跑几轮，或循环 1000 次，才能提高命中率。

# 4. 确定性：让异步测试可复现

## 4.1 用虚拟时间消除 sleep 抖动

```rust
#[tokio::test(start_paused = true)]
async fn test_virtual_time() {
    use std::time::Instant;
    use tokio::time::{sleep, Duration};

    let start = Instant::now();
    sleep(Duration::from_secs(3600)).await;   // 一秒内"睡"完一小时
    assert!(start.elapsed() >= Duration::from_secs(3600));
}
```

★ **关键认知**：`start_paused = true` 让 Tokio 使用**虚拟时钟**：`sleep` 不真的等，而是"当所有任务都空闲时快进到下一个定时器"。这让"等 1 小时"的测试在毫秒级完成，且**完全确定**。⚠ 虚拟时间只对 `tokio::time` 生效，对 `std::thread::sleep`、真实 I/O 无效——测试里必须用 Tokio 的版本。

## 4.2 虚拟时间配合 timeout

```rust
#[tokio::test(start_paused = true)]
async fn test_timeout_fires() {
    use tokio::time::{timeout, sleep, Duration};

    let res = timeout(Duration::from_millis(100), async {
        sleep(Duration::from_secs(5)).await;
    })
    .await;

    assert!(res.is_err(), "应当触发超时");
}
```

## 4.3 三条确定性纪律

```text
1. 不用固定 sleep 来"等结果"——用通道/Notify/条件变量等信号
2. 不用真实网络/磁盘——用 mock 或临时目录
3. 断言时间用范围或虚拟时间，不用精确毫秒
```

等待结果时用 `Notify` 或通道发信号，`notify.notified().await` 会精确地在事件发生时返回，不受机器快慢影响。

# 5. Mock 外部依赖

## 5.1 用 trait 抽象依赖

```rust
use async_trait::async_trait;

#[async_trait]
pub trait UserRepo {
    async fn find(&self, id: u64) -> Result<String, String>;
}

pub struct Service<R: UserRepo> {
    repo: R,
}

impl<R: UserRepo> Service<R> {
    pub async fn greeting(&self, id: u64) -> Result<String, String> {
        let name = self.repo.find(id).await?;
        Ok(format!("你好，{}", name))
    }
}
```

```toml
[dependencies]
async-trait = "0.1"
```

★ **关键认知**：异步 trait 方法在稳定 Rust 里还不能直接用于 `dyn`，`#[async_trait]` 宏把它展开成返回 `Pin<Box<dyn Future>>` 的形式，代价是一次装箱。⚠ 如果不需要 `dyn`，优先用泛型 `R: UserRepo`（静态分发，零开销）；需要 `Arc<dyn UserRepo>` 时才必须 `async_trait`。

## 5.2 写一个假实现

```rust
struct FakeRepo;

#[async_trait]
impl UserRepo for FakeRepo {
    async fn find(&self, id: u64) -> Result<String, String> {
        if id == 1 {
            Ok("Alice".to_string())
        } else {
            Err(format!("用户 {} 不存在", id))
        }
    }
}

#[tokio::test]
async fn test_greeting() {
    let svc = Service { repo: FakeRepo };
    assert_eq!(svc.greeting(1).await.unwrap(), "你好，Alice");
    assert!(svc.greeting(2).await.is_err());
}
```

★ **关键认知**：手写 fake 往往比引入 mock 框架更清晰——它的行为完全由你控制，且没有宏和动态代理的复杂度。⚠ 但 fake 要**只模拟被测代码真正用到的行为**，模拟过头会让测试与实现耦合，实现一变测试就碎。

## 5.3 用 mpsc 模拟"会失败的下游"

让 fake 持有一个 `mpsc::Receiver<Result<T, E>>`，每次 `find` 从通道取一条预设结果，就能逐次控制"失败一次再成功"的重试路径，时序完全由测试掌握。⚠ 记得给测试加 `timeout`，否则 fake 没发消息时测试会挂住。

# 6. 测试组织与实践

| 位置 | 用途 | 可见性 |
|------|------|--------|
| `src/*.rs` 内 `#[cfg(test)] mod tests` | 单元测试，可访问私有项 | 同模块 |
| `tests/*.rs` | 集成测试，只访问公开 API | 作为外部 crate |

★ **关键认知**：`#[tokio::test]` 在两种位置都能用，但**每个测试都建一个运行时**，测试多了会累积线程创建开销。⚠ `tokio` 的 `features = ["full"]` 放在 dev-dependencies 里没问题，但别带进生产依赖——会显著增加编译时间。

# 7. 常见错误

```text
错误 1：async fn 测试没加 #[tokio::test]
  现象：编译报错，或测试根本没跑
  解决：加属性宏

错误 2：用真实 sleep 等结果
  现象：CI 上偶发失败
  解决：改用 Notify/通道信号，或 start_paused

错误 3：测试无超时，CI 卡死
  解决：timeout 包住测试体，或用 nextest slow-timeout

错误 4：断言精确耗时
  现象：慢机器上失败
  解决：留余量，或用虚拟时间

错误 5：mock 过度，测试与实现耦合
  解决：只 mock 行为契约，不 mock 内部调用顺序
```

# 8. 练习

1. 给一个 `async fn` 写测试：先用真实 `sleep` 版本，再改成 `start_paused = true`，对比运行时间。
2. 写一个测试，故意让被测函数等一个永不到来的通道消息，用 `timeout` 让它 1 秒内失败并打印原因。
3. 用 `async_trait` 定义一个 `Storage` trait，写两个实现（内存版、可注入失败的版本），测试"失败后重试成功"的路径。

# 9. 小结

| 主题 | 要点 |
|------|------|
| `#[tokio::test]` | 每个测试建 current-thread 运行时并 block_on |
| `flavor` | 默认单线程，需要并行调度时显式多线程 |
| 超时 | 测试必须自带，`timeout` 包测试体 |
| 并发断言 | 留余量，别赌精确毫秒 |
| `start_paused` | 虚拟时间，秒级完成且确定 |
| 确定性 | 用信号而非 sleep，用虚拟时间而非真实时钟 |
| mock | 优先 trait + 手写 fake，通道控制逐次返回 |

下一篇是本模块收尾：把前面八篇的坑汇总成一份清单——如何避免阻塞运行时、如何发现和治理任务泄漏、背压怎么设计、观测怎么做，以及面试和实战里最高频的几个陷阱与总结。
