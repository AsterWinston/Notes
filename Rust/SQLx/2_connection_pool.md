连接池是数据库访问里最容易被忽视、又最容易出事故的一块。池子配小了请求排队，配大了数据库被打爆，超时没设好则整个服务卡在"等连接"上。本篇把 SQLx 的 `Pool` 拆开看：它怎么复用连接、每个参数控制什么、多数据库与优雅关闭怎么做。

# 1. Pool 到底是什么

## 1.1 没有连接池会怎样

每次查询都新建一条 TCP 连接，代价包括 TCP 三次握手、TLS 协商、数据库认证、分配会话内存。一个 QPS 一千的服务，每秒就要重复上千次这些动作，数据库的连接数也会瞬间打满。

```text
无池：每次请求都新建连接
┌────────┐   TCP+认证   ┌──────────┐
│ 请求 1 │ ───────────► │          │
└────────┘              │          │
┌────────┐   TCP+认证   │  数据库  │
│ 请求 2 │ ───────────► │          │
└────────┘              │          │
┌────────┐   TCP+认证   │          │
│ 请求 3 │ ───────────► │          │
└────────┘              └──────────┘
```

有池之后，连接被创建一次、反复借还：

```text
有池：连接复用
┌────────┐   借    ┌─────────────────┐   复用   ┌──────────┐
│ 请求 1 │ ──────► │ Pool            │ ───────► │ 连接 A   │
└────────┘         │  ┌───────────┐  │          └──────────┘
┌────────┐   借    │  │ 空闲队列  │  │   复用   ┌──────────┐
│ 请求 2 │ ──────► │  │ A B C ... │  │ ───────► │ 连接 B   │
└────────┘         │  └───────────┘  │          └──────────┘
                   └─────────────────┘
```

## 1.2 Pool 内部是 Arc

```rust
use sqlx::PgPool;

let pool = PgPoolOptions::new().connect(&url).await?;
let pool2 = pool.clone(); // 只增加引用计数，不新建连接
```

★ **关键认知**：`Pool` 内部是 `Arc<PoolInner>`，`clone()` 是**廉价的引用计数操作**。你可以在 `AppState`、每个 handler、每个 repo 里各存一份，它们共享同一个连接池。⚠ 不要为了"每个请求一个池"去反复 `PgPoolOptions::connect`，那等于关掉了连接池，还会迅速耗尽数据库的 `max_connections`。

## 1.3 借与还的生命周期

```rust
// 借用一条连接，离开作用域自动归还
let mut conn = pool.acquire().await?;
sqlx::query("SELECT 1").execute(&mut *conn).await?;
// conn 在这里 drop，连接回到池里
```

```rust
// 直接把 &Pool 交给查询，内部临时借用并归还
sqlx::query("SELECT 1").execute(&pool).await?;
```

两种写法效果相同，区别只在"要不要在多次操作间持有同一条连接"。事务必须用前者，因为事务的所有语句必须在同一连接上。

# 2. 创建池与关键参数

## 2.1 PgPoolOptions

```rust
use std::time::Duration;
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(20)                          // 并发上限
    .min_connections(2)                           // 常驻空闲连接
    .acquire_timeout(Duration::from_secs(5))      // 借连接的最长等待
    .idle_timeout(Duration::from_secs(600))       // 空闲多久后关闭
    .max_lifetime(Duration::from_secs(1800))      // 连接最长存活时间
    .test_before_acquire(true)                    // 借出前探活
    .connect(&url)
    .await?;
```

## 2.2 参数一览

| 参数 | 默认值 | 控制什么 | 调优建议 |
|------|--------|----------|----------|
| `max_connections` | 10 | 池内连接上限 | 按数据库连接预算分配 |
| `min_connections` | 0 | 空闲时保留的连接 | 服务启动后想预热，设 2-5 |
| `acquire_timeout` | 30 秒 | 借不到连接等多久 | 服务端设 3-5 秒，快速失败 |
| `idle_timeout` | 10 分钟 | 空闲连接多久回收 | 与数据库侧 `idle_session_timeout` 配合 |
| `max_lifetime` | 30 分钟 | 连接最长寿命 | 短于中间件/数据库的强制断连时间 |
| `test_before_acquire` | true | 借出前发探活语句 | 有负载均衡时可关掉换性能 |
| `after_connect` | 无 | 新连接建立后的初始化回调 | 设时区、`search_path`、语句超时 |

★ **关键认知**：`max_connections` 不是"越大越快"。数据库总连接数是全局资源，要按下面的公式倒推：

```text
数据库允许连接数 >= 实例副本数 × 每个实例的 max_connections × 1.2
```

⚠ 留 20% 余量给运维、迁移、监控账号。如果数据库上限是 100，跑 4 个副本，每个实例最多给 20，别给 30。

## 2.3 三个超时的区别

| 超时 | 管的是 | 触发后的错误 |
|------|--------|--------------|
| `acquire_timeout` | 从池里借连接 | `Error::PoolTimedOut` |
| `idle_timeout` | 连接空闲多久被回收 | 无错误，后台静默关闭 |
| `max_lifetime` | 连接从创建到强制关闭 | 无错误，后台静默替换 |
| 语句超时 | 单条 SQL 执行时长 | `Error::Database`，由数据库报错 |

★ **关键认知**：`acquire_timeout` 只覆盖"排队等连接"，**不覆盖 SQL 执行时间**。一条慢查询会把连接占住，池子被慢慢抽干，最后所有人都卡在 `PoolTimedOut` 上——看起来像连接池问题，根因却是慢 SQL。⚠ 一定要在数据库侧或连接初始化时设置语句超时：

```rust
// PostgreSQL：新连接统一设置 5 秒语句超时
let pool = PgPoolOptions::new()
    .after_connect(|conn, _meta| {
        Box::pin(async move {
            sqlx::query("SET statement_timeout = '5s'")
                .execute(conn)
                .await?;
            Ok(())
        })
    })
    .connect(&url)
    .await?;
```

# 3. 获取连接与健康检查

## 3.1 acquire 与 PoolConnection

```rust
let mut conn = pool.acquire().await?;

// 连接自身也能开事务
let mut tx = conn.begin().await?;
sqlx::query("UPDATE users SET name = $1 WHERE id = $2")
    .bind("新名字")
    .bind(1_i64)
    .execute(&mut *tx)
    .await?;
tx.commit().await?;
```

`PoolConnection<Postgres>` 在 drop 时归还连接。如果它是在事务中途被 drop，SQLx 会先回滚再归还，避免把"脏连接"放回池里。

## 3.2 test_before_acquire

开启后，每次借出连接前会发一条轻量探活语句（PostgreSQL 是 `SELECT 1`）。好处是能剔除已被数据库或中间件断开的死连接，坏处是每次借出多一次往返。

```rust
// 有 PgBouncer、云数据库自动断连时，建议开启
.test_before_acquire(true)
```

★ **关键认知**：池能做的是"发现坏连接并丢弃"，不是"修复连接"。如果连接因网络抖动断开，探活会失败，SQLx 关闭它并尝试下一条。⚠ 探活也会消耗数据库资源，高 QPS 下如果连接本身很稳定，可以关掉，靠 `max_lifetime` 主动轮换。

## 3.3 连接泄漏

连接泄漏指借出后长期不归还，常见原因：

| 原因 | 表现 | 排查 |
|------|------|------|
| 把 `PoolConnection` 存进全局或长生命周期结构 | 池逐渐耗尽 | 搜索 `acquire` 的存放位置 |
| 在持有连接时 await 外部 IO | 连接被长时间占用 | 缩短持有范围 |
| 事务忘记 commit/rollback | 连接卡在事务中 | 检查所有分支 |
| 流式查询没读完 | 连接被游标占用 | `fetch` 要消费完或显式 drop |

⚠ SQLx 没有"连接泄漏检测"开关，但可以打开日志观察 `acquire` 与 `release` 是否成对：

```rust
// 需要开启 sqlx 的 logging feature，并初始化 tracing subscriber
tracing_subscriber::fmt()
    .with_max_level(tracing::Level::DEBUG)
    .init();
```

# 4. 多数据库

## 4.1 同时连多个库

```rust
use sqlx::postgres::PgPoolOptions;
use sqlx::mysql::MySqlPoolOptions;

// 两个独立的池，类型不同，互不影响
let pg = PgPoolOptions::new().max_connections(10).connect(&pg_url).await?;
let my = MySqlPoolOptions::new().max_connections(5).connect(&my_url).await?;
```

适合读写分离、从旧库迁移、跨库报表。⚠ 两个池就是两套连接预算，要分别计入数据库的连接上限。

## 4.2 AnyPool：运行时切换

```rust
use sqlx::any::AnyPoolOptions;

// 需要 any feature
sqlx::any::install_default_drivers();

let pool = AnyPoolOptions::new()
    .max_connections(10)
    .connect(&url)
    .await?;
```

`AnyPool` 根据连接串前缀选择驱动，适合"同一份代码支持多种数据库"的工具或测试框架。

| 方案 | 类型安全 | 占位符 | 适合 |
|------|----------|--------|------|
| 具体池（`PgPool` 等） | 强，编译期检查 | 各自方言 | 生产服务首选 |
| `AnyPool` | 弱，运行时判断 | 统一用 `?` | 多数据库兼容层、测试工具 |

★ **关键认知**：`AnyPool` 牺牲了编译期类型检查和方言差异（占位符统一为 `?`），只适合"确实需要运行时切换"的场景。⚠ 生产服务不要为了"以后可能换库"提前上 `AnyPool`，那会让所有查询失去编译期校验。

# 5. 生命周期与优雅关闭

## 5.1 close 做了什么

```rust
pool.close().await;
```

`close()` 会：停止接受新的借用请求、等待已借出的连接归还、关闭所有物理连接。已经在执行的查询会跑完，不会被打断。

```rust
// 带超时的关闭，避免无限等待
tokio::time::timeout(Duration::from_secs(10), pool.close())
    .await
    .ok();
```

## 5.2 与 Axum 集成

```rust
use axum::{extract::State, routing::get, Router};
use sqlx::PgPool;

#[derive(Clone)]
struct AppState {
    pool: PgPool, // clone 只是 Arc 计数 +1
}

async fn health(State(state): State<AppState>) -> &'static str {
    match sqlx::query("SELECT 1").execute(&state.pool).await {
        Ok(_) => "ok",
        Err(_) => "db down",
    }
}

let state = AppState { pool };
let app = Router::new().route("/health", get(health)).with_state(state);

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
axum::serve(listener, app)
    .with_graceful_shutdown(async {
        tokio::signal::ctrl_c().await.ok();
    })
    .await?;
```

★ **关键认知**：关闭顺序应该是"先停止接收新请求，再关池"。如果先 `pool.close()`，还在处理中的请求会拿到 `PoolClosed` 错误。⚠ Axum 的 `with_graceful_shutdown` 只保证请求处理完，池的关闭要放在 `serve` 返回之后：

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await?;

pool.close().await; // serve 返回后，所有请求都已结束
```

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `PoolTimedOut` 频繁出现 | 池太小、有慢查询、连接泄漏 | 查慢 SQL，核对连接预算 |
| 启动后第一批请求很慢 | `min_connections` 为 0，冷启动 | 设 2-5 预热 |
| 云数据库每隔几分钟报连接断开 | 中间件有闲置超时 | `max_lifetime` 设短于它，开启探活 |
| `PoolClosed` | 池已关闭仍在查询 | 检查关闭顺序 |
| 多个副本把库打满 | 每实例连接数 × 副本数超限 | 按公式倒推 `max_connections` |
| `too many connections` | 忘了关旧池或重复建池 | 全局只建一次 |

# 7. 动手练习

1. 把 `max_connections` 设为 1，写一个并发 10 的压测脚本，观察 `acquire_timeout` 触发时的错误类型；再把超时改成 100 毫秒对比现象。
2. 用 `after_connect` 给每个新连接设置 `application_name`，然后在 PostgreSQL 的 `pg_stat_activity` 里确认能看到该标识。
3. 实现一个 Axum 服务，`/health` 检查数据库，按 Ctrl+C 后验证日志顺序是"停止接收请求 → 查询跑完 → 关闭池"。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Pool 本质 | `Arc` 包裹，clone 廉价，全局建一次 |
| 上限 | 按副本数与数据库上限倒推，留 20% 余量 |
| 三种超时 | 借连接、空闲回收、最长寿命，各管一段 |
| 慢查询 | 不在 `acquire_timeout` 覆盖范围内，要单独设语句超时 |
| 探活 | `test_before_acquire` 能剔除死连接，但有开销 |
| 多库 | 具体池类型安全，`AnyPool` 灵活但弱校验 |
| 关闭 | 先停服务，再 `pool.close().await` |

下一篇进入查询 API，讲清楚 `query`、`query_as`、`query_scalar` 的分工，`.bind()` 为什么能防注入，`FromRow` 的字段映射规则，以及动态 SQL 的安全拼装方式。
