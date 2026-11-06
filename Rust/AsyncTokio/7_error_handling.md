异步代码的错误处理比同步多出两道坎：一是错误要跨越 `.await` 和任务边界，二是 `JoinError`、`Elapsed`、`RecvError` 这些"基础设施错误"会跟业务错误混在一起。这一篇讲清楚 `Result` 与 `?` 在 async 里的行为、`anyhow` 和 `thiserror` 的分工、上下文怎么叠加，以及错误如何跨任务传播而不丢失。

# 1. Result 与 ? 在异步中的行为

## 1.1 老规矩没变

```rust
use std::fs;

fn read_config() -> Result<String, std::io::Error> {
    fs::read_to_string("config.toml")
}

async fn read_config_async() -> Result<String, std::io::Error> {
    tokio::fs::read_to_string("config.toml").await   // async 版本
}
```

★ **关键认知**：`?` 在 `async fn` 里的语义与同步完全一致：遇到 `Err` 就**提前返回**，用 `From` 转换错误类型。区别只在于展开后的状态机多了一个"错误出口"。⚠ 所以 `?` 只能用在返回 `Result`（或 `Option`）的函数里，`async` 也不例外；在返回 `()` 的 `async` 里用 `?` 会编译报错。

## 1.2 ? 与 From 的配合

`?` 在遇到 `Err` 时会调用 `From::from` 做类型转换。因此只要为目标错误类型实现了 `From<源错误>`，`?` 就能把 `io::Error`、`ParseIntError` 自动转成你的 `AppError`。手写这些 `From` 很啰嗦，`thiserror` 的 `#[from]` 正是为此而生（见 2.2）。

# 2. anyhow 与 thiserror 的分工

## 2.1 一句话区分

| 库 | 面向 | 用途 |
|----|------|------|
| `thiserror` | **库作者** | 定义具体的、可被匹配的错误枚举 |
| `anyhow` | **应用作者** | 快速聚合各种错误，带上下文，不关心具体类型 |

★ **关键认知**：**库要能被调用方 `match`，所以用 `thiserror` 定义精确的错误类型；应用只关心"报错并记录"，所以用 `anyhow::Error` 兜住一切**。⚠ 不要在自己的库的公开 API 里返回 `anyhow::Error`——调用方无法匹配具体错误，也没法做重试决策。

## 2.2 thiserror 示例

```toml
[dependencies]
thiserror = "2"
```

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("读取配置文件失败：{0}")]
    Io(#[from] std::io::Error),          // #[from] 自动生成 From，? 可直接用

    #[error("配置格式错误：{message}")]
    Parse { message: String },

    #[error("缺少必填字段：{field}")]
    MissingField { field: String },
}

pub fn load_config(path: &str) -> Result<String, ConfigError> {
    let text = std::fs::read_to_string(path)?;   // io::Error 自动转 ConfigError::Io
    if text.is_empty() {
        return Err(ConfigError::MissingField {
            field: "content".to_string(),
        });
    }
    Ok(text)
}
```

★ **关键认知**：`#[error("...")]` 定义 `Display`，`#[from]` 定义 `From`，`#[source]` 标记错误链的下一环。这让"定义错误类型"从几十行样板变成几个属性。⚠ `#[from]` 只能用于**单一来源**——同一个枚举里不能有两个变体都 `#[from]` 同一个类型，否则 `From` 实现冲突。

## 2.3 anyhow 示例

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{Context, Result, bail};

async fn load_user(id: u64) -> Result<String> {
    let path = format!("users/{}.json", id);
    let text = tokio::fs::read_to_string(&path)
        .await
        .with_context(|| format!("读取用户 {} 的文件失败：{}", id, path))?;

    if text.trim().is_empty() {
        bail!("用户 {} 的文件为空", id);   // 相当于 return Err(anyhow!(...))
    }
    Ok(text)
}

#[tokio::main]
async fn main() -> Result<()> {
    let u = load_user(42).await?;          // main 可以返回 Result
    println!("{}", u);
    Ok(())
}
```

★ **关键认知**：`anyhow::Context` 是最有价值的 API——它让每个 `?` 都能附上一句"当时在干什么"，错误打印出来是一串因果链，而不是孤零零的 "No such file"。⚠ `anyhow::Error` 是类型擦除的，你**不能 `match` 它的具体类型**；如果某个错误需要被程序逻辑区分处理，就必须在**产生它的那一层**用 `thiserror` 定义好。

## 2.4 混用：库用 thiserror，应用用 anyhow

```rust
// 库：精确类型，可匹配
pub fn parse_port(s: &str) -> Result<u16, ConfigError> {
    s.parse().map_err(|_| ConfigError::Parse { message: s.into() })
}

// 应用：anyhow 包住，再补上下文
use anyhow::Context;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let port = parse_port("abc")
        .context("解析监听端口失败")?;   // thiserror 错误 → anyhow，并加一层上下文
    println!("{}", port);
    Ok(())
}
```

★ **关键认知**：`thiserror` 的错误实现了 `std::error::Error`，所以可以**自动转成 `anyhow::Error`**（`anyhow` 对所有 `E: Error + Send + Sync + 'static` 有 blanket impl）。这就是"库精确、应用宽松"能无缝衔接的底层原因。⚠ 反向不成立：`anyhow::Error` 转不回具体类型（除非用 `downcast`，且要求原始错误 `'static`）。

# 3. 错误上下文：让日志能定位问题

## 3.1 没有上下文 vs 有上下文

```text
没有上下文：
  Error: No such file or directory (os error 2)

有上下文：
  Error: 加载应用配置失败
    Caused by:
      读取配置文件 failed
    Caused by:
      No such file or directory (os error 2)
```

★ **关键认知**：**上下文不是给机器看的，是给凌晨三点排障的人看的**。一个好的错误链应该能回答"哪一步、哪个输入、为什么"。⚠ 常见反模式是每层都 `map_err(|e| e.to_string())`，把结构化错误压成字符串——错误链断裂，`source()` 再也拿不到根因。

## 3.2 在 await 之间保留上下文

```rust
use anyhow::{Context, Result};

async fn fetch_and_parse(url: &str) -> Result<Vec<u8>> {
    let resp = reqwest::get(url)
        .await
        .with_context(|| format!("请求 {} 失败", url))?
        .bytes()
        .await
        .context("读取响应体失败")?;      // 每个 await 都是一次可失败的边界
    Ok(resp.to_vec())
}
```

★ **关键认知**：异步的每个 `.await` 都是一个潜在的失败点，因此**每个 await 后面都值得考虑加一层上下文**。⚠ 但别过度：给不会失败的 await（比如 `sleep`）加 context 是噪音，反而淹没真正的错误。

# 4. 跨任务边界的错误传播

## 4.1 JoinError：任务层的失败

```rust
use anyhow::{anyhow, Context, Result};

async fn run_task() -> Result<i32> {
    let handle = tokio::spawn(async {
        // 业务逻辑，返回 Result
        Ok::<i32, String>(42)
    });

    // 三层结构：JoinError / 业务 Err / Ok
    let value = handle
        .await
        .context("任务 join 失败")?      // JoinError（panic/取消）
        .map_err(|e| anyhow!("业务错误：{}", e))?;   // 业务错误
    Ok(value)
}
```

★ **关键认知**：`spawn` 之后错误变成三层：`Result<Result<T, E>, JoinError>`。外层是"任务有没有跑完"，内层是"业务成没成功"。⚠ 一定要区分 `is_panic()` 和 `is_cancelled()`：panic 说明代码有 bug，取消往往不是错误（比如正常关闭），不该一视同仁地报警。

## 4.2 用 JoinSet 收集多个任务的错误

`JoinSet::join_next()` 逐条返回 `Result<Result<T, E>, JoinError>`，外层仍是任务层、内层是业务层，匹配方式与 4.1 相同。

★ **关键认知**：`anyhow` 的 `{:#}` 格式化会打印**完整错误链**（每层 `Caused by`），而 `{}` 只打印最外层。日志里请用 `{:#}` 或 `{:?}`，否则你把上下文全丢了。⚠ `JoinSet` 的错误是逐条返回的，不会因为一个失败就中止其它任务——需要"一个失败就全部停止"时，得自己 `abort_all()`。

## 4.3 错误与取消要分开建模

★ **关键认知**：把 `Timeout`、`Cancelled`、`Upstream` 建成三个独立变体，而不是全塞进一个 `Io(String)`。**取消通常不是错误**。优雅关闭时任务被 abort 属于正常流程，如果把它混进错误率统计，监控会误报。⚠ 实践上把 `Cancelled` 单列，并在日志级别上降级（debug 而非 error），是成熟服务的常见做法。

# 5. 一个完整的错误设计

## 5.1 分层结构与 main 的返回类型

```text
┌────────────────────────────────────┐
│  应用层（main / handler）           │  anyhow::Result，补上下文
├────────────────────────────────────┤
│  服务层（业务编排）                 │  thiserror 定义 ServiceError
├────────────────────────────────────┤
│  基础设施层（DB / HTTP / 文件）     │  thiserror 定义具体错误 + #[from]
└────────────────────────────────────┘
```

```rust
// 基础设施错误
#[derive(Debug, thiserror::Error)]
pub enum RepoError {
    #[error("数据库连接失败")]
    Connection(#[source] sqlx::Error),
    #[error("记录不存在：{id}")]
    NotFound { id: u64 },
}

// 服务错误：包装基础设施错误
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("仓储层失败")]
    Repo(#[from] RepoError),          // 底层错误向上转
    #[error("参数非法：{0}")]
    Invalid(String),
}

// 应用层用 anyhow + context 补请求上下文
async fn handler(id: u64) -> anyhow::Result<String> {
    let user = fetch_user(id).await
        .with_context(|| format!("处理请求 id={}", id))?;
    Ok(user)
}

async fn fetch_user(id: u64) -> Result<String, ServiceError> {
    if id == 0 {
        return Err(ServiceError::Invalid("id 不能为 0".into()));
    }
    Ok(format!("user-{}", id))
}
```

★ **关键认知**：分层的价值在于**每层只回答自己该回答的问题**：基础设施层说"什么操作失败了"，服务层说"业务上意味着什么"，应用层说"这次请求的上下文是什么"。⚠ 很多项目的错误处理烂，不是因为没选对库，而是因为所有层都在 `e.to_string()`，把层次压平了。

★ **关键认知**：`main` 可以返回 `anyhow::Result<()>`，出错时以非零码退出并打印完整错误链。⚠ 但如果退出前要清理（关闭连接、flush 日志），别依赖自动退出——显式 `match` 处理完再返回。

# 6. 常见错误

```text
错误 1：库 API 返回 anyhow::Error
  后果：调用方无法 match、无法重试
  解决：库用 thiserror 定义具体类型

错误 2：到处 map_err(|e| e.to_string())
  后果：错误链断裂，排障困难
  解决：用 #[source] / Context 保留结构

错误 3：把 JoinError 和业务错误一视同仁
  后果：取消被当成故障报警
  解决：区分 is_panic / is_cancelled，取消单独建模

错误 4：日志用 {} 而非 {:#}
  后果：只看到最外层"操作失败"
  解决：anyhow 用 {:#} 或 {:?} 打印完整链

错误 5：async fn 里用 ? 但返回 ()
  解决：返回 Result，或用 match 显式处理
```

# 7. 练习

1. 用 `thiserror` 定义一个含 3 个变体的错误枚举，其中两个带 `#[from]`，在两个不同的 `?` 场景验证自动转换。
2. 用 `anyhow::Context` 给一段包含两次 `.await` 的代码加两层上下文，用 `{:#}` 打印，观察 `Caused by` 链。
3. 写一个 `spawn` 任务，分别制造"业务 Err""panic""abort"三种结局，用 `JoinHandle::await` 逐一区分并打印。

# 8. 小结

| 主题 | 要点 |
|------|------|
| `?` | 提前返回 + `From` 转换，async 中语义不变 |
| `thiserror` | 库作者用，定义可匹配的错误枚举 |
| `anyhow` | 应用作者用，类型擦除 + 上下文 |
| `Context` | 给每个失败点补"当时在干什么" |
| `#[source]` | 保留底层错误，维持错误链 |
| `JoinError` | 任务层失败，区分 panic / 取消 |
| `{:#}` | 打印完整错误链，日志必备 |
| 取消 | 不是错误，应单独建模 |

下一篇进入异步测试：`#[tokio::test]` 如何创建运行时、怎样给测试加超时防止挂死、并发测试怎么写、外部依赖怎么 mock，以及如何让异步测试保持确定性。
