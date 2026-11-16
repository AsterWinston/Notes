数据库错误是服务里最需要"读懂"的一类错误：唯一键冲突要返回 409，连接池超时要返回 503，找不到行要返回 404，而语法错误应该直接 500 并告警。SQLx 把底层错误收敛成 `sqlx::Error`，但它只是一个枚举，真正的难点是**分类**与**映射**。

# 1. sqlx::Error 的结构

## 1.1 主要变体

```rust
pub enum Error {
    Database(Box<dyn DatabaseError>), // 数据库返回的错误
    Io(std::io::Error),              // 网络读写失败
    Protocol(String),                // 协议错误，如连接被意外关闭
    RowNotFound,                     // fetch_one 没取到行
    PoolTimedOut,                    // 借连接超时
    PoolClosed,                      // 池已关闭
    Migrate(Box<MigrateError>),      // 迁移失败
    // 还有 Configuration、Tls、Decode、Encode 等变体
}
```

★ **关键认知**：`sqlx::Error` 是**分层**的。`Database` 包裹数据库自己返回的错误（唯一约束、语法错误、死锁），`Io`/`Protocol` 是连接层问题，`PoolTimedOut` 是池层问题。映射之前先判断"错在哪一层"，否则很容易把所有错误都当成 500。⚠ 不要用 `err.to_string()` 做字符串匹配来分类，错误信息会随数据库版本变化；用类型和错误码。

## 1.2 常用分类

| 变体 | 语义 | 通常映射 |
|------|------|----------|
| `RowNotFound` | 查询要求一行但没有 | 404 |
| `Database` + 唯一约束 | 违反唯一索引 | 409 |
| `Database` + 外键约束 | 引用了不存在的父行 | 409 / 422 |
| `Database` + check 约束 | 业务规则不满足 | 422 |
| `PoolTimedOut` | 等连接超时 | 503 |
| `PoolClosed` | 服务正在关闭 | 503 |
| `Io` / `Protocol` | 连接断开 | 503 或重试 |
| `Configuration` | 配置错误 | 500，启动时失败 |
| `Migrate` | 迁移失败 | 500，阻止启动 |

# 2. 数据库错误的细分

## 2.1 DatabaseError trait

```rust
use sqlx::error::DatabaseError;

if let sqlx::Error::Database(db) = &err {
    println!("码：{:?}", db.code());            // 数据库原始码
    println!("约束：{:?}", db.constraint());    // 违反的约束名
    println!("唯一：{}", db.is_unique_violation());
    println!("外键：{}", db.is_foreign_key_violation());
}
```

`DatabaseError` 提供了一组跨数据库的语义判断：

| 方法 | 含义 |
|------|------|
| `is_unique_violation()` | 唯一索引冲突 |
| `is_foreign_key_violation()` | 外键约束冲突 |
| `is_check_violation()` | check 约束冲突 |
| `code()` | 数据库原始错误码 |
| `constraint()` | 冲突的约束名 |
| `kind()` | 归一化后的 `ErrorKind` |

★ **关键认知**：优先用 `is_xxx_violation()` 而不是错误码。SQLx 会在各驱动里把 PostgreSQL 的 `23505`、MySQL 的 `1062`、SQLite 的 `2067` 统一归一到唯一冲突，跨库代码只写一份判断即可。⚠ 但 `constraint()` 在部分数据库或版本下可能为 `None`，不要假设它一定存在。

## 2.2 原始错误码对照

| 场景 | PostgreSQL | MySQL | SQLite |
|------|------------|-------|--------|
| 唯一约束冲突 | `23505` | `1062` | `2067` |
| 外键约束冲突 | `23503` | `1452` / `1451` | `787` |
| check 约束冲突 | `23514` | `3819` | `275` |
| 非空约束冲突 | `23502` | `1048` | `1299` |
| 序列化失败（可重试） | `40001` | `1213`（死锁） | `5`（busy） |

需要区分具体唯一键时，用 `match db.constraint()` 把 `users_email_key`、`users_name_key` 分别映射到不同业务错误，其余落到通用 `Conflict`。

## 2.3 `is_unique_violation` 的边界

`is_unique_violation()` 只回答"是不是唯一冲突"，不回答"哪个字段冲突"。一个表有多个唯一索引时，必须配合 `constraint()` 或错误码里的详情才能精确映射。

⚠ 约束名是数据库生成的，可能随迁移变化（比如 `users_email_key` 改了索引名就变了）。对稳定性要求高的业务，建议在迁移里给约束显式命名，例如 `ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);`。

# 3. RowNotFound 与 Option

## 3.1 两种"没有数据"

```rust
// 场景 A：按主键查，可能不存在 —— 用 fetch_optional
let user: Option<User> = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE id = $1",
)
.bind(id).fetch_optional(pool).await?;

// 场景 B：插入并 RETURNING，必须返回一行 —— 用 fetch_one
let created: User = sqlx::query_as::<_, User>(
    "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email",
)
.bind(name).bind(email).fetch_one(pool).await?;
```

★ **关键认知**：`RowNotFound` 是**调用方选错了取行方式**的信号，不是数据库故障。按主键查询用 `fetch_optional` 得到 `Option`，业务里再决定 404；只有"逻辑上必须有返回"的场景才用 `fetch_one`，此时 `RowNotFound` 意味着数据或代码异常，应该 500 并告警。⚠ 把 `RowNotFound` 一律映射成 404 会让真正的 bug（比如 `RETURNING` 被误删）伪装成"资源不存在"。

## 3.2 用 Option 表达可空列

```rust
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    nickname: Option<String>, // 列可为 NULL
}
```

注意区分两件事：

| 情况 | 类型 | 含义 |
|------|------|------|
| 行不存在 | `Option<User>`（来自 `fetch_optional`） | 查无此记录 |
| 列值为 NULL | `Option<String>`（字段类型） | 记录存在，但字段为空 |

⚠ 常见错误是把 `NOT NULL` 列写成 `Option<T>`，代码里到处 `unwrap`，既没表达约束，又埋了 panic。

# 4. 连接层与池层错误

## 4.1 哪些是可重试的

先按"错误来自哪一层"分类，再看是否值得重试：

| 错误 | 可重试 | 原因 |
|------|--------|------|
| 唯一冲突 | 否 | 数据本身冲突，重试仍冲突 |
| 外键冲突 | 否 | 父行不存在，除非上游先补数据 |
| `RowNotFound` | 否 | 查询条件决定，不会自己出现 |
| `PoolTimedOut` | 是（退避） | 可能是瞬时高负载 |
| `Io` / `Protocol` | 是（有限次） | 网络抖动 |
| 序列化失败 / 死锁 | 是 | 数据库建议重试整个事务 |
| `Configuration` | 否 | 配置错，重试无意义 |

★ **关键认知**：重试的前提是**操作幂等**。读操作天然幂等；写操作要么本身幂等（`UPDATE ... WHERE id = ?` 设绝对值），要么靠唯一键/去重表保证重复执行不出错。⚠ 对一个"插入订单"盲目重试，可能生成两笔订单。

# 5. 映射到业务错误

## 5.1 定义 AppError

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("资源不存在")]
    NotFound,
    #[error("记录已存在：{0}")]
    Conflict(String),
    #[error("参数不合法：{0}")]
    Invalid(String),
    #[error("服务暂时不可用")]
    Unavailable,
    #[error(transparent)]
    Database(#[from] sqlx::Error),
}
```

`#[from]` 让 `?` 自动把 `sqlx::Error` 转成 `AppError::Database`，但这样所有数据库错误都会落到一个笼统变体里。更好的做法是手写分类：

```rust
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        match err {
            sqlx::Error::RowNotFound => AppError::NotFound,
            sqlx::Error::PoolTimedOut | sqlx::Error::PoolClosed => AppError::Unavailable,

            sqlx::Error::Database(ref db) if db.is_unique_violation() => {
                AppError::Conflict("记录已存在".into())
            }
            sqlx::Error::Database(ref db) if db.is_foreign_key_violation() => {
                AppError::Invalid("关联的数据不存在".into())
            }
            sqlx::Error::Database(ref db) if db.is_check_violation() => {
                AppError::Invalid("数据不满足约束".into())
            }

            // 连接层的瞬时错误也可以归为不可用
            sqlx::Error::Io(_) | sqlx::Error::Protocol(_) => AppError::Unavailable,

            other => AppError::Database(other),
        }
    }
}
```

## 5.2 HTTP 状态码映射

`status_code` 可以直接用 `match self` 返回 404、409、422、503、500，下面的表是同一份映射的可读版本。

| AppError | 状态码 | 客户端该做什么 |
|----------|--------|----------------|
| `NotFound` | 404 | 不要重试 |
| `Conflict` | 409 | 换唯一值或提示已存在 |
| `Invalid` | 422 | 修正参数 |
| `Unavailable` | 503 | 退避后重试 |
| `Database` | 500 | 不暴露细节，服务端告警 |

★ **关键认知**：**不要把 `sqlx::Error` 的原始信息返回给客户端**。它可能包含表名、列名、SQL 片段甚至连接串。对外只给稳定的业务文案，完整错误写日志并带 trace id。⚠ 但日志里也要注意脱敏，参数值可能含手机号、密码。

## 5.3 分层处理

Handler 只负责把 `AppError` 转成 HTTP 响应；Repo 把 `sqlx::Error` 分类成 `AppError` 并记录上下文；Service 组合多个 repo，决定是否重试、是否补偿。

# 6. 重试

## 6.1 只重试可重试的错误

```rust
fn is_retryable(err: &sqlx::Error) -> bool {
    match err {
        sqlx::Error::PoolTimedOut => true,
        sqlx::Error::Io(_) | sqlx::Error::Protocol(_) => true,
        sqlx::Error::Database(db) => {
            // PostgreSQL 序列化失败 40001、死锁 40P01；MySQL 死锁 1213
            matches!(db.code().as_deref(), Some("40001") | Some("40P01") | Some("1213"))
        }
        _ => false,
    }
}
```

## 6.2 指数退避

```rust
use std::time::Duration;

async fn with_retry<T, F, Fut>(mut op: F) -> Result<T, sqlx::Error>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, sqlx::Error>>,
{
    let mut attempt = 0;
    loop {
        match op().await {
            Ok(value) => return Ok(value),
            Err(err) if is_retryable(&err) && attempt < 3 => {
                attempt += 1;
                // 100ms、200ms、400ms，加抖动避免同时重试
                let wait = 100u64 * (1 << attempt) + (attempt * 37) % 50;
                tokio::time::sleep(Duration::from_millis(wait)).await;
            }
            Err(err) => return Err(err),
        }
    }
}
```

★ **关键认知**：重试必须满足三个条件——**错误可重试**、**操作幂等**、**有次数上限**。三者缺一，重试就会从"提高可用性"变成"放大故障"。⚠ 没有抖动的重试会让所有客户端在同一时刻再次打向数据库，形成"重试风暴"。

## 6.3 重试要放在事务外层

正确的做法是把**整个事务**作为一个整体重试：`with_retry(|| async { let mut tx = pool.begin().await?; /* ... */ tx.commit().await?; Ok(()) }).await?`。

⚠ 不要在事务内部对单条语句重试。序列化失败意味着整个事务要重来，只重试其中一条会让事务处于不一致状态。

# 7. anyhow 与 thiserror 的分工

| 库 | 用途 | 场景 |
|----|------|------|
| `thiserror` | 定义**库或应用的错误类型** | `AppError`、领域错误 |
| `anyhow` | 快速包裹**任意错误**并加上下文 | 脚本、bin、内部工具 |

应用入口处可以用 `anyhow::Context` 串上下文，例如 `sqlx::PgPool::connect(&url).await.context("连接数据库失败，请检查 DATABASE_URL")?`。

★ **关键认知**：`anyhow::Error` 适合"不需要被上层分类处理"的场合。一旦调用方要按错误类型决定 404/409/503，就必须用 `thiserror` 定义枚举，因为 `anyhow` 把类型擦除了，`downcast` 既啰嗦又容易漏。

# 8. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 唯一冲突返回 500 | 没分类 `sqlx::Error` | 判断 `is_unique_violation` 映射 409 |
| 所有错误都返回 404 | 把 `RowNotFound` 当通用错误 | 只对"查无"场景用 `fetch_optional` |
| 重试后产生重复订单 | 写操作不幂等 | 唯一键去重或幂等 token |
| 客户端看到 SQL 片段 | 直接把错误 `to_string()` 返回 | 对外用固定文案 |
| 匹配错误码失效 | 用字符串匹配错误信息 | 用 `code()` 或 `is_xxx_violation()` |
| 迁移失败但服务启动 | 没检查 `migrate!` 结果 | 迁移失败直接退出 |

# 9. 动手练习

1. 给 `users` 表加唯一索引，写一个创建用户的函数，把唯一冲突映射成 409、外键冲突映射成 422，并用两个并发请求验证只有一个成功。
2. 实现 `with_retry`，用 `PoolTimedOut` 和唯一冲突两种错误测试，确认前者重试、后者立即返回。
3. 把 `AppError` 接入 Axum 的 `IntoResponse`，要求 500 响应里不包含数据库原始信息，但日志里能看到完整错误和请求 id。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 分层 | 语义错误、池错误、连接错误分开处理 |
| 唯一冲突 | `is_unique_violation()`，需要精确时看 `constraint()` |
| `RowNotFound` | 用 `fetch_optional` 从源头避免 |
| 可重试 | 池超时、IO、序列化失败、死锁 |
| 不可重试 | 约束冲突、`RowNotFound`、配置错误 |
| 映射 | `thiserror` 定义 `AppError`，对外脱敏 |
| 重试 | 幂等 + 分类 + 上限 + 抖动 |

下一篇进入测试，讲如何为每个测试准备独立数据库、`#[sqlx::test]` 的迁移机制、用事务回滚做隔离、Testcontainers 起真实 PostgreSQL，以及 CI 里怎么配置数据库服务。
