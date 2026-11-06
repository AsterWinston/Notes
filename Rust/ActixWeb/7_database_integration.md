Rust 的数据库生态主要分两派：SQLx 走"异步 + 接近裸 SQL"，Diesel 走"同步 + 强类型 ORM"。选哪个，比"哪个性能高"更重要的，是你的团队更愿意写 SQL 还是更愿意和类型系统搏斗。本篇以 SQLx 为主线，串起连接池、CRUD、事务和错误处理，最后给出 Diesel 的接入方式。

# 1. SQLx 与 Diesel 怎么选

| 维度 | SQLx | Diesel |
|------|------|--------|
| 执行模型 | 原生异步 | 同步（需 `web::block`） |
| SQL 写法 | 手写 SQL，`query!` 可编译期校验 | DSL 构建查询 |
| 学习曲线 | 会 SQL 就能上手 | 需要理解 DSL 与 trait |
| 编译期检查 | `query!` 连接数据库校验 | 类型系统强校验 |
| 适合场景 | 异步服务、复杂 SQL、快速迭代 | 强类型领域模型、同步团队 |

★ **关键认知**：Actix Web 是异步框架，SQLx 与它天然匹配；Diesel 本身同步，必须用 `web::block` 把查询挪到阻塞线程池，否则会堵住 worker。**新项目优先 SQLx**，除非团队已有成熟的 Diesel 代码与经验。

## 1.1 依赖

```toml
[dependencies]
actix-web = "4"
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "macros", "migrate"] }
serde = { version = "1", features = ["derive"] }
thiserror = "2"
anyhow = "1"
```

用 PostgreSQL 把 `"sqlite"` 换成 `"postgres"`，MySQL 换成 `"mysql"`。`runtime-tokio` 必须显式开启。

# 2. 接入 SQLx

## 2.1 建立连接池

```rust
use sqlx::sqlite::SqlitePoolOptions;
use std::time::Duration;

pub async fn init_pool(database_url: &str) -> Result<sqlx::SqlitePool, sqlx::Error> {
    SqlitePoolOptions::new()
        .max_connections(10)                     // 池内最多 10 条连接
        .min_connections(2)                      // 预热 2 条，减少首请求抖动
        .acquire_timeout(Duration::from_secs(5)) // 拿不到连接最多等 5 秒
        .idle_timeout(Duration::from_secs(600))  // 空闲 10 分钟回收
        .max_lifetime(Duration::from_secs(1800)) // 连接最长活 30 分钟
        .connect(database_url)
        .await
}
```

PostgreSQL 对应 `sqlx::postgres::PgPoolOptions`。

## 2.2 注入 AppState

```rust
use actix_web::{web, App, HttpServer};

#[derive(Clone)]
pub struct AppState {
    pub db: sqlx::SqlitePool, // Pool 内部是 Arc，clone 很便宜
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let db = init_pool("sqlite://app.db?mode=rwc")
        .await
        .expect("数据库连接失败");

    let state = web::Data::new(AppState { db });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .route("/users", web::get().to(list_users))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

`sqlite://app.db?mode=rwc` 表示文件不存在时创建。⚠ 不写 `mode=rwc` 时，首次启动可能报 "unable to open database file"。

★ **关键认知**：连接池是**进程级共享**的，不是每个 worker 一份。4 个 worker 共用 `max_connections = 10` 的池，总连接数仍是 10。把池放进 `web::Data` 后，所有 Handler 共享同一个池，这才是正确姿势。

## 2.3 连接池参数

| 参数 | 作用 | 经验值 |
|------|------|--------|
| `max_connections` | 池上限 | 数据库上限 / 实例数，再留 20% 余量 |
| `acquire_timeout` | 等待超时 | 3 到 10 秒，宁可快速失败 |

⚠ `max_connections` 不是越大越好。PostgreSQL 每个连接都占内存，超过数据库上限会直接拒绝新连接。多个实例共享数据库时，要按"实例数 × 单实例上限"算总账。

# 3. CRUD

## 3.1 查询多行

```rust
use actix_web::{web, Result};
use serde::Serialize;
use sqlx::FromRow;

#[derive(Debug, Serialize, FromRow)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
}

pub async fn list_users(state: web::Data<AppState>) -> Result<web::Json<Vec<User>>> {
    let users = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users ORDER BY id",
    )
    .fetch_all(&state.db)
    .await?; // sqlx::Error → actix_web::Error（依赖 AppError 的 From）

    Ok(web::Json(users))
}
```

`FromRow` 按**列名**映射到字段，所以 SQL 别名要和结构体字段一致，例如 `SELECT name AS display_name` 对应 `display_name: String`。

## 3.2 查询单行

```rust
pub async fn get_user(
    id: web::Path<i64>,
    state: web::Data<AppState>,
) -> Result<web::Json<User>> {
    let user = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users WHERE id = ?",
    )
    .bind(id.into_inner())
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| crate::errors::AppError::NotFound("用户不存在".into()))?;

    Ok(web::Json(user))
}
```

| 方法 | 无结果时 | 多条结果时 |
|------|----------|------------|
| `fetch_one` | 返回 `RowNotFound` 错误 | 返回第一条 |

★ **关键认知**："查不到"不是数据库错误。用 `fetch_optional` + `ok_or_else(AppError::NotFound)` 才能得到 404；直接用 `fetch_one` 会把"没找到"变成 500。

## 3.3 插入、更新与删除

```rust
let id = sqlx::query("INSERT INTO users (name, email) VALUES (?, ?)")
    .bind(&body.name).bind(&body.email)
    .execute(&state.db).await?
    .last_insert_rowid();

let result = sqlx::query("UPDATE users SET name = ? WHERE id = ?")
    .bind(&body.name).bind(id)
    .execute(&state.db).await?;

if result.rows_affected() == 0 {
    return Err(AppError::NotFound("用户不存在".into()).into());
}
```

PostgreSQL 没有 `last_insert_rowid`，要用 `RETURNING id` 配合 `fetch_one` 或 `query_scalar`。⚠ 更新不检查 `rows_affected()` 时，改不存在的 ID 也会返回 200。

## 3.4 编译期校验的 query!

`query_as!(User, "SELECT ...")` 会在**编译期**连接数据库，检查 SQL 语法、表结构、字段类型。编译机无法访问数据库时，用 `cargo sqlx prepare` 生成 `.sqlx/` 离线元数据，并在 CI 设置 `SQLX_OFFLINE=true`。团队协作时 `.sqlx/` 要提交到仓库。

# 4. 事务

```rust
pub async fn transfer(
    state: web::Data<AppState>,
    from: i64,
    to: i64,
    amount: i64,
) -> Result<(), AppError> {
    let mut tx = state.db.begin().await?;

    let updated = sqlx::query(
        "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?",
    )
    .bind(amount)
    .bind(from)
    .bind(amount)
    .execute(&mut *tx) // 解引用，把 &mut Transaction 当作执行器
    .await?
    .rows_affected();

    if updated == 0 {
        return Err(AppError::BadRequest("余额不足".into())); // tx 被 drop，自动回滚
    }

    sqlx::query("UPDATE accounts SET balance = balance + ? WHERE id = ?")
        .bind(amount)
        .bind(to)
        .execute(&mut *tx)
        .await?;

    tx.commit().await?; // 显式提交
    Ok(())
}
```

★ **关键认知**：`Transaction` 的 `Drop` 行为是**回滚**。任何提前 `return`、`?` 传播错误、panic，只要没走到 `commit()`，事务都会回滚。这让"忘记回滚"不再是问题，但也意味着**忘记 commit 就等于白写**。

⚠ 事务绑定单条连接，不要把它存进全局状态跨请求复用，否则会导致锁长期不释放、并发串行化甚至死锁。事务必须在**一个 Handler 内**开始并结束。

# 5. Diesel 的接入方式

Diesel 查询是阻塞的，必须用 `web::block` 挪到阻塞线程池：

```rust
let users = web::block(move || {
    let mut conn = pool.get().expect("获取连接失败"); // demo 简化
    users::table.load::<User>(&mut conn)
})
.await
.map_err(ErrorInternalServerError)??; // BlockingError 与查询错误都 → 500
```

`web::block` 返回 `Result<Result<T, DieselError>, BlockingError>`，所以常见写法是两次 `?` 或两次 `map_err`。⚠ 只处理一层会把内层错误直接 `unwrap`，线上数据库一报错就 panic。

| 关注点 | SQLx | Diesel |
|--------|------|--------|
| Handler 写法 | 直接 `.await` | `web::block` 包一层 |

# 6. 数据库错误接入 AppError

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("资源不存在: {0}")]
    NotFound(String),
    #[error("唯一键冲突: {0}")]
    Conflict(String),
    #[error("数据库错误: {0}")]
    Database(#[from] sqlx::Error),
    #[error("内部错误: {0}")]
    Internal(#[from] anyhow::Error),
}
```

在 Handler 里把"用户可修复"的错误进一步分类：

```rust
fn map_db_error(e: sqlx::Error) -> AppError {
    match &e {
        sqlx::Error::Database(db) if db.is_unique_violation() => {
            AppError::Conflict("邮箱已被注册".into())
        }
        sqlx::Error::RowNotFound => AppError::NotFound("记录不存在".into()),
        _ => AppError::Database(e),
    }
}
// 调用：.await.map_err(map_db_error)?
```

★ **关键认知**：`#[from] sqlx::Error` 让 `?` 能工作，但它把所有数据库错误都归为 500。唯一键冲突、外键约束这类"用户可修复"的错误应该显式分类成 409 或 400，否则前端只能显示"服务内部错误"。

# 7. 迁移与常见坑

```bash
cargo install sqlx-cli --no-default-features --features sqlite
sqlx migrate add create_users   # 新建迁移
sqlx migrate run                # 执行迁移
```

迁移文件里写建表 SQL，启动时可调用 `sqlx::migrate!("./migrations").run(&db).await?`。⚠ 多实例部署时每个实例都跑迁移会竞争，更稳的做法是把迁移放在发布流程的单独一步。

| 现象 | 原因 | 解决 |
|------|------|------|
| 启动报无法打开数据库 | SQLite 文件不存在且没写 `mode=rwc` | URL 加参数或预创建文件 |
| 查询总是 500 | 用了 `fetch_one` 查不到就报错 | 改 `fetch_optional` |
| 更新返回 200 但没改 | 没检查 `rows_affected` | 为 0 返回 404 |

★ **关键认知**：SQL 注入的唯一可靠防线是**参数绑定**。`format!` 拼 SQL 再执行，即使输入来自"可信"的内部服务，也会在日志、审计、二次开发中变成漏洞。所有变量一律 `.bind()`。

# 8. 动手练习

1. 建一张 `users` 表，实现列表、详情、创建、删除四个接口，全部走 SQLx。
2. 给创建接口加唯一键冲突处理：重复邮箱返回 409，而不是 500。

验收标准：查不到返回 404，重复邮箱返回 409，事务失败后两边余额都不变。

# 9. 小结

| 概念 | 要点 |
|------|------|
| SQLx | 异步、手写 SQL、`query!` 可编译期校验 |
| Diesel | 同步 ORM，必须 `web::block` |
| 连接池 | 进程级共享，`max_connections` 按数据库上限算 |
| 查询 | `fetch_optional` 处理"查不到" |
| 事务 | `commit` 才生效，`Drop` 自动回滚 |
| 错误分类 | 唯一键冲突映射 409，其余 500 |
| 迁移 | 发布流程单独执行，避免多实例竞争 |
| 安全 | 一律参数绑定，绝不拼 SQL |

下一篇进入鉴权与 WebSocket，讲清楚 JWT 的签发与校验、鉴权中间件与提取器如何配合、Actix actor 模型下的 WebSocket 连接、心跳与广播，以及常见的令牌泄漏与连接泄漏问题。
