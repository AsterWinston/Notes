数据库是大多数服务真正的瓶颈，也是错误最集中的地方：连接池打满、事务没提交、唯一约束冲突、测试库互相污染。Axum 本身不包数据库，但和 SQLx 的组合非常顺：异步、编译期 SQL 检查、连接池都直接可用。

# 1. SQLx 与连接池

## 1.1 为什么选 SQLx

| 方案 | 特点 | 适合 |
|------|------|------|
| SQLx | 异步、手写 SQL、编译期检查 | 想掌控 SQL、接受手写映射 |
| Diesel | 同步 ORM、类型安全查询构造器 | 强类型、复杂查询构建 |
| SeaORM | 异步 ORM、基于 SeaQuery | 想要 ActiveRecord 风格 |

★ **关键认知**：SQLx 不是 ORM，它不替你生成 SQL，也不隐藏数据库差异。它提供的是"异步连接池 + 参数绑定 + 行映射 + 可选的编译期校验"。⚠ 选 SQLx 意味着你要自己写迁移、自己写查询、自己处理方言差异；换来的是透明和可控。

## 1.2 创建连接池

```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "macros", "migrate"] }
```

```rust
use std::time::Duration;
use sqlx::sqlite::SqlitePoolOptions;

let pool = SqlitePoolOptions::new()
    .max_connections(10) // 明确上限，别让数据库被打爆
    .acquire_timeout(Duration::from_secs(5)) // 拿不到连接就快速失败
    .connect("sqlite://app.db?mode=rwc") // 文件不存在时自动创建
    .await?;

sqlx::migrate!("./migrations").run(&pool).await?;
```

★ **关键认知**：连接池必须在启动时创建一次，放进 `AppState` 共享。`Pool` 内部是 `Arc`，`clone()` 只增加引用计数，不会新建连接。⚠ `max_connections` 不是越大越好：数据库的连接数上限、实例副本数、连接池总和要一起算，通常留 20% 余量。

# 2. 查询

## 2.1 query_as 与 FromRow

```rust
#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
}

async fn list_users(pool: &sqlx::SqlitePool) -> Result<Vec<User>, sqlx::Error> {
    sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users ORDER BY id",
    )
    .fetch_all(pool)
    .await
}
```

`query_as` 按列名映射到结构体字段。列名和字段名不一致时用 `#[sqlx(rename = "...")]`，不要靠位置猜。

## 2.2 绑定参数

```rust
let user = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE id = ?",
)
.bind(id)
.fetch_optional(pool)
.await?;
```

| 数据库 | 占位符 | 示例 |
|--------|--------|------|
| SQLite | `?` | `WHERE id = ?` |
| MySQL | `?` | `WHERE id = ?` |
| PostgreSQL | `$1`、`$2` | `WHERE id = $1` |

★ **关键认知**：永远用 `.bind()`，不要把用户输入拼进 SQL 字符串。SQLx 的参数绑定会走数据库的预处理协议，拼接则会把注入风险直接带进生产。⚠ `fetch_optional` 返回 `Option<T>`，`fetch_one` 在无行时返回 `Error::RowNotFound`，`fetch_all` 返回 `Vec<T>`，三者不要混用。

## 2.3 编译期检查

```rust
let user = sqlx::query_as!(
    User,
    "SELECT id, name, email FROM users WHERE id = ?",
    id
)
.fetch_optional(&pool)
.await?;
```

`query_as!` 会在编译时连接数据库，检查 SQL 语法、表结构、列类型。代价是构建机必须能访问数据库，或者使用离线模式：

```bash
cargo sqlx prepare   # 生成 .sqlx 目录，提交到仓库
```

之后构建时设置 `SQLX_OFFLINE=true`，就不再需要数据库。⚠ 改了 schema 记得重新 `prepare`，否则 CI 会用旧元数据通过编译，运行时才发现列不存在。

# 3. 事务

## 3.1 begin/commit/rollback

```rust
let mut tx = pool.begin().await?;

sqlx::query("UPDATE accounts SET balance = balance - ? WHERE id = ?")
    .bind(amount)
    .bind(from)
    .execute(&mut *tx)
    .await?;

sqlx::query("UPDATE accounts SET balance = balance + ? WHERE id = ?")
    .bind(amount)
    .bind(to)
    .execute(&mut *tx)
    .await?;

tx.commit().await?;
```

注意 `execute(&mut *tx)`：`Transaction` 需要解引用成连接。事务对象离开作用域而没有 `commit()` 时，SQLx 会回滚。

★ **关键认知**：事务的边界应该包住"要么全成功、要么全失败"的最小业务单元，而不是整个请求。⚠ 事务里不要做 HTTP 请求、发消息、读大文件，长事务会持锁并拖垮连接池。

## 3.2 事务中的错误

```rust
async fn transfer(pool: &SqlitePool, from: i64, to: i64, amount: i64) -> Result<(), AppError> {
    let mut tx = pool.begin().await?;

    let result = sqlx::query("UPDATE accounts SET balance = balance - ? WHERE id = ?")
        .bind(amount)
        .bind(from)
        .execute(&mut *tx)
        .await?;

    if result.rows_affected() == 0 {
        return Err(AppError::NotFound); // tx 在这里 drop，自动回滚
    }

    sqlx::query("UPDATE accounts SET balance = balance + ? WHERE id = ?")
        .bind(amount)
        .bind(to)
        .execute(&mut *tx)
        .await?;

    tx.commit().await?;
    Ok(())
}
```

`?` 提前返回时 `tx` 被 drop，SQLx 会安排回滚。⚠ 但 drop 触发的回滚是"尽力而为"，如果连接已经断开就无能为力；关键业务可以显式 `tx.rollback().await` 并记录结果。

# 4. 错误映射

## 4.1 sqlx::Error

```rust
impl From<sqlx::Error> for AppError {
    fn from(error: sqlx::Error) -> Self {
        match error {
            sqlx::Error::RowNotFound => AppError::NotFound,
            sqlx::Error::PoolTimedOut => AppError::ServiceUnavailable,
            sqlx::Error::Database(db) if db.is_unique_violation() => {
                AppError::Conflict("记录已存在".into())
            }
            other => AppError::Database(other),
        }
    }
}
```

| SQLx 错误 | 建议状态码 |
|-----------|------------|
| `RowNotFound` | 404 |
| 唯一约束冲突 | 409 |
| 外键约束冲突 | 409 / 422 |
| `PoolTimedOut` | 503 |
| 连接失败 / 协议错误 | 500 |

## 4.2 唯一约束与 404

```rust
async fn create_user(pool: &SqlitePool, name: &str, email: &str) -> Result<User, AppError> {
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id, name, email",
    )
    .bind(name)
    .bind(email)
    .fetch_one(pool)
    .await?; // 唯一约束冲突会自动转成 AppError::Conflict
    Ok(user)
}
```

★ **关键认知**：把数据库约束当成业务规则的最后一道防线。应用层可以先查重，但并发下仍可能撞唯一索引；最终必须依赖数据库约束，并把冲突错误映射成 409。⚠ 不要捕获错误后返回 200 加一段"创建失败"，那会让客户端和监控都误判。

# 5. 测试数据库

## 5.1 内存 SQLite 的坑

```rust
async fn test_pool() -> sqlx::SqlitePool {
    let pool = SqlitePoolOptions::new()
        .max_connections(1) // 关键：:memory: 每条连接是独立的库
        .connect("sqlite::memory:")
        .await
        .expect("创建测试库失败");

    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("迁移失败");
    pool
}
```

★ **关键认知**：SQLite 的 `:memory:` 数据库是**按连接隔离**的。连接池开两条连接，就会得到两个不同的空库，测试会出现"表不存在"的诡异错误。要么把连接数限制为 1，要么用 `file:test.db?mode=memory&cache=shared` 让连接共享同一份内存库。

## 5.2 sqlx::test

```rust
#[sqlx::test]
async fn create_user_works(pool: sqlx::SqlitePool) {
    sqlx::query("INSERT INTO users (name, email) VALUES (?, ?)")
        .bind("Tom")
        .bind("tom@example.com")
        .execute(&pool)
        .await
        .unwrap();

    let count: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users")
        .fetch_one(&pool)
        .await
        .unwrap();

    assert_eq!(count, 1);
}
```

`#[sqlx::test]` 会为每个测试创建独立数据库并自动跑迁移，测试之间互不影响。它依赖 `DATABASE_URL` 和 `migrations/` 目录，CI 里要设置好环境变量。

# 6. 项目实践

```text
src/
├── db.rs             // 建池、跑迁移
├── models/user.rs    // FromRow 结构体
├── repo/users.rs     // 查询与事务
└── handlers/users.rs // 只做参数提取与响应
```

| 层 | 职责 | 不应该做 |
|----|------|----------|
| Handler | 提取参数、调用 repo、返回响应 | 写 SQL |
| Repo | 查询、事务、错误映射 | 处理 HTTP |
| Model | 数据结构与映射 | 依赖 axum |

★ **关键认知**：Repo 层返回 `Result<T, AppError>`，Handler 只负责把结果变成 HTTP。这样数据库逻辑可以脱离 Axum 单独测试，事务边界也更清晰。⚠ 不要在 Handler 里直接 `pool.begin()`，事务应该由业务服务或 repo 统一管理。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 测试报"表不存在" | `:memory:` 多连接各自独立 | `max_connections(1)` 或共享缓存 |
| 编译时连不上数据库 | `query!` 需要在线元数据 | 用 `cargo sqlx prepare` 离线构建 |
| 连接池耗尽 | 泄漏连接或长事务 | 检查所有路径是否释放、缩短事务 |
| 唯一约束 500 | 错误没细分 | 映射 `is_unique_violation` 为 409 |
| 事务没生效 | 忘了 `commit()` | 检查是否所有路径都提交 |
| `last_insert_rowid` 为 0 | 用错返回类型 | 确认执行的是同一条插入 |

⚠ SQLx 的编译期检查只校验 SQL 与 schema，不校验业务语义。"余额不能为负"这类规则仍要写在代码或数据库约束里。

# 8. 动手练习

1. 建一张 `users` 表，写迁移，实现列表、详情、创建三个接口，全部通过 `State` 注入连接池。
2. 实现转账事务，要求余额不足时返回 422，账户不存在返回 404，并验证失败后两边余额都没变。
3. 用内存 SQLite 写两个测试，确认它们互不影响；再故意把连接数改成 2，观察"表不存在"的错误。

验收标准：`cargo test` 可重复执行，事务失败能回滚，唯一约束返回 409。

# 9. 小结

| 概念 | 要点 |
|------|------|
| 连接池 | 启动时建一次，`Pool` clone 廉价 |
| `query_as` | 按列名映射到 `FromRow` 结构体 |
| 参数绑定 | 永远 `.bind()`，不要拼 SQL |
| `query!` | 编译期检查，需在线或离线元数据 |
| 事务 | `begin` → `execute(&mut *tx)` → `commit` |
| 回滚 | drop 触发尽力回滚，关键路径显式 rollback |
| 错误映射 | `RowNotFound` 404、唯一约束 409、池超时 503 |
| 测试库 | `:memory:` 连接隔离，或 `#[sqlx::test]` |

下一篇进入鉴权与 WebSocket，讲清楚 JWT 的签发与校验、鉴权中间件如何把用户信息传给 Handler、WebSocket 的升级与心跳，以及用广播通道做多连接推送。
