数据库测试最容易变成"在我机器上能过"的重灾区：测试之间互相污染、并行跑就报唯一冲突、CI 里没有数据库直接跳过。SQLx 提供了专门的测试属性和迁移集成，配合事务隔离或容器化，可以把这些问题一次性解决。

# 1. 三种隔离策略

## 1.1 先想清楚要隔离什么

| 策略 | 隔离手段 | 速度 | 保真度 |
|------|----------|------|--------|
| 内存 SQLite | 每个测试独立内存库 | 最快 | 低，方言不同 |
| 每测试一库 | `#[sqlx::test]` 自动建库 | 中 | 高，用真实数据库 |
| 事务回滚 | 测试结束 rollback | 快 | 高，但要求代码接受连接 |
| Testcontainers | 容器化数据库 | 慢 | 最高，环境完全可控 |

★ **关键认知**：隔离的目标不是"让测试不互相影响"这么简单，而是**让每个测试都能独立重复执行**。同一个测试连跑十次结果一致、换个顺序也一致，才叫隔离。⚠ 只靠"测试结束时删数据"往往会漏表、漏序列、漏外键顺序，最终还是要靠独立库或事务回滚。

## 1.2 选择建议

单元测试和快速反馈用内存 SQLite 或事务回滚；集成测试和方言相关逻辑用 `#[sqlx::test]` 每测试一库；验收测试用 Testcontainers 起与生产同版本的数据库。

# 2. #[sqlx::test]

## 2.1 基本用法

```rust
// 需要 sqlx 的 macros 与 migrate feature
#[sqlx::test]
async fn create_user_works(pool: sqlx::PgPool) {
    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("张三").bind("zhangsan@example.com")
        .execute(&pool).await.unwrap();

    let count: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users")
        .fetch_one(&pool).await.unwrap();

    assert_eq!(count, 1);
}
```

这个属性会为每个测试读取 `DATABASE_URL`，创建一个带随机后缀的新数据库，执行 `migrations/` 下的全部迁移，把 `PgPool` 注入测试函数，并在测试结束后删除该数据库。

## 2.2 参数形式

测试函数的参数决定了注入什么：写 `pool: sqlx::PgPool` 注入连接池；写 `mut conn: sqlx::PgConnection` 注入单条连接，事务测试常用；两者都写也可以同时拿到。函数签名里的参数名和类型由 `#[sqlx::test]` 识别，不需要额外配置。

## 2.3 指定迁移目录与 fixtures

```rust
// 指定迁移目录
#[sqlx::test(migrations = "./migrations")]
async fn custom_migrations(pool: sqlx::PgPool) {}

// 关闭自动迁移，完全自己建表
#[sqlx::test(migrations = false)]
async fn no_migrations(pool: sqlx::PgPool) {}

// 每个测试前导入 fixtures 下的固定数据
#[sqlx::test(fixtures(path = "./fixtures", scripts("users")))]
async fn with_fixtures(pool: sqlx::PgPool) {}
```

★ **关键认知**：`#[sqlx::test]` 每测试建库的开销主要来自**迁移**和**建库 DDL**，不是查询。迁移越多、测试越多，总耗时越明显。⚠ 优化方向是减少测试库数量（用事务回滚）或让迁移足够轻，而不是把多个测试塞进一个库——那会重新引入污染。

## 2.4 前提条件

`DATABASE_URL` 必须指向一个**能创建数据库**的账号（PostgreSQL 需要 `CREATEDB`），迁移目录默认从项目根目录找。每个测试一个库，天然支持并行。⚠ 云数据库通常不允许随便建库，这时改用事务回滚或 Testcontainers。

# 3. 事务回滚做隔离

## 3.1 思路

一个测试内开启事务，所有操作都在事务里执行，测试结束直接 rollback。数据库回到测试前状态，不需要清理。

```rust
#[tokio::test]
async fn test_create_user(pool: PgPool) {
    let mut tx = pool.begin().await.unwrap();

    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("张三").bind("zhangsan@example.com")
        .execute(&mut *tx).await.unwrap();

    let count: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users")
        .fetch_one(&mut *tx).await.unwrap();

    assert_eq!(count, 1);
    tx.rollback().await.unwrap(); // 数据不落库
}
```

## 3.2 对代码结构的要求

被测试的函数必须接受 `&mut PgConnection`（或泛型 `E: Executor`），而不是 `&PgPool`：

```rust
// 可测试：接受执行器
async fn create_user(
    executor: impl sqlx::PgExecutor<'_>,
    name: &str,
    email: &str,
) -> Result<User, sqlx::Error> {
    sqlx::query_as::<_, User>(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email",
    )
    .bind(name).bind(email).fetch_one(executor).await
}
```

测试里传 `&mut *tx`，生产里传 `&pool`，同一个函数两处复用。

★ **关键认知**：事务回滚隔离能否用起来，取决于**repo 层是否接受 `Executor`**。如果每个函数都写死 `&PgPool`，就只能在测试里真的写库再清理。⚠ 这属于"为可测试性调整设计"，不是过度设计——它同时让事务复用变得自然。

## 3.3 局限

| 局限 | 说明 |
|------|------|
| 无法测试提交后的行为 | rollback 后看不到落库结果 |
| 无法测试跨连接可见性 | 事务内的数据别的连接看不到 |
| 序列不会回滚 | `BIGSERIAL` 消耗的序列值不会退 |
| DDL 可能隐式提交 | MySQL 的 DDL 会破坏事务边界 |

⚠ 序列不回滚意味着测试里插入的行 id 会持续增长，断言"id 等于 1"必然失败。要断言就断言业务字段，别断言自增 id。

# 4. 内存 SQLite 的坑

## 4.1 按连接隔离

```rust
// 错误示范：默认多连接，迁移在 A 上建表，查询可能落到 B
let pool = SqlitePoolOptions::new()
    .connect("sqlite::memory:")
    .await?;
```

★ **关键认知**：SQLite 的 `:memory:` 数据库是**按连接隔离**的。连接池开几条连接，就有几个互不相干的空库。⚠ 这不是 SQLx 的 bug，是 SQLite 的语义。两种解法：

```rust
// 解法 1：把连接数限制为 1
let pool = SqlitePoolOptions::new()
    .max_connections(1)
    .connect("sqlite::memory:")
    .await?;

// 解法 2：共享缓存，多个连接访问同一个内存库
let pool = SqlitePoolOptions::new()
    .connect("sqlite:file:testdb?mode=memory&cache=shared")
    .await?;
```

解法 2 需要额外注意：只要还有连接存活，共享内存库就存在；所有连接都关闭后数据消失。测试里要保持至少一条连接常驻。

# 5. Testcontainers

## 5.1 它解决什么问题

本地没装 PostgreSQL、CI 环境不固定、需要特定版本与扩展时，用容器起一个真实数据库，测试结束自动销毁。

```text
cargo test
   ├── 启动 postgres:16 容器
   ├── 建池 + 跑迁移 + 执行测试
   └── 容器 drop，自动删除
```

## 5.2 依赖

```toml
[dev-dependencies]
tokio = { version = "1", features = ["full"] }
testcontainers = "0.23"
testcontainers-modules = { version = "0.11", features = ["postgres"] }
```

## 5.3 示例

```rust
use sqlx::PgPool;
use testcontainers_modules::postgres::Postgres;
use testcontainers_modules::testcontainers::runners::AsyncRunner;
use testcontainers_modules::testcontainers::ContainerAsync;

async fn start_postgres() -> (ContainerAsync<Postgres>, PgPool) {
    let container = Postgres::default().start().await.unwrap();
    let port = container.get_host_port_ipv4(5432).await.unwrap();
    let url = format!("postgres://postgres:postgres@127.0.0.1:{}/postgres", port);

    let pool = PgPool::connect(&url).await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();
    (container, pool)
}

#[tokio::test]
async fn test_with_real_postgres() {
    let (_container, pool) = start_postgres().await;
    let one: i32 = sqlx::query_scalar("SELECT 1").fetch_one(&pool).await.unwrap();
    assert_eq!(one, 1);
}
```

★ **关键认知**：`container` 必须活到测试结束，否则它被 drop 时容器会被删掉，池立刻连不上。⚠ 用 `_container` 绑定（前缀下划线但仍持有值），不要写成 `_`，`let _ = ...` 会立即丢弃。

## 5.4 多个测试共享容器

每个测试起一个容器太慢，可以用 `tokio::sync::OnceCell` 缓存一个池。⚠ 共享容器加共享库会重新引入污染，要么每个测试用独立 schema，要么每个测试建独立 database。

# 6. 测试组织

## 6.1 目录结构

把建池、跑迁移、创建测试数据的公共逻辑放进 `tests/common/mod.rs`，每个集成测试文件通过 `mod common;` 复用。repo 函数放在 `src/repo/`，只接受 `impl Executor`，不直接依赖池。

## 6.2 测试数据工厂

```rust
async fn insert_user(executor: impl sqlx::PgExecutor<'_>, name: &str) -> i64 {
    sqlx::query_scalar(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
    )
    .bind(name)
    .bind(format!("{}@example.com", name))
    .fetch_one(executor)
    .await
    .unwrap()
}
```

★ **关键认知**：测试数据应该由**工厂函数**创建，而不是从固定的 seed 文件里猜 id。固定 id 会让测试依赖插入顺序，一改迁移就全挂。⚠ 测试之间不要共享可变状态，包括"第 1 条记录"这种隐含假设。

## 6.3 并行测试

`cargo test` 默认多线程并行。用 `#[sqlx::test]` 时每个测试独立库，天然安全；用共享池 + 事务回滚时也安全；但如果共享池直接写库，就必须 `--test-threads=1` 或自己加锁。

# 7. CI 配置

## 7.1 GitHub Actions 服务容器

```yaml
name: test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: demo
        ports:
          - 5432:5432
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/demo
      SQLX_OFFLINE: "true"
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --all-features
```

## 7.2 迁移检查

在空库上执行 `cargo sqlx migrate run`，确认迁移能从零跑通；再执行 `cargo sqlx prepare --check`，确认离线元数据与当前 schema 一致。

★ **关键认知**：CI 里至少要跑两类检查——**迁移能从零跑通**，以及**`query!` 的离线元数据与 schema 一致**。前者防"迁移顺序错"，后者防"改了表忘了 prepare"。⚠ `SQLX_OFFLINE=true` 只影响编译期宏，不影响运行时连接；`#[sqlx::test]` 仍然需要真实的 `DATABASE_URL`。

# 8. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 测试报 `no such table` | SQLite `:memory:` 多连接各自独立 | 限制为 1 条或共享缓存 |
| 测试并行时唯一冲突 | 共享库且数据未隔离 | 每测试一库或事务回滚 |
| `#[sqlx::test]` 建库失败 | 账号没有 `CREATEDB` 权限 | 换账号或用 Testcontainers |
| 测试结束数据还在 | 只开事务没 rollback | 显式 `tx.rollback()` |
| 断言 id 等于 1 失败 | 序列不回滚 | 断言业务字段，不断言自增 id |
| 容器测试莫名连不上 | container 被提前 drop | 绑定到具名变量并活到测试结束 |
| CI 编译报列不存在 | 忘了 `cargo sqlx prepare` | 重新生成并提交 `.sqlx/` |

# 9. 动手练习

1. 用 `#[sqlx::test]` 写三个测试：创建用户、重复邮箱冲突、查询不存在用户，确认它们可以并行重复执行。
2. 把 repo 函数改成接受 `impl PgExecutor`，用"开事务 + rollback"的方式写一个不落库的测试，并验证测试后表里没有数据。
3. 用 Testcontainers 起 PostgreSQL 16，跑通迁移和一条查询；再故意把 `_container` 改成 `_`，观察连接失败的现象。

验收标准：`cargo test` 连续跑三次结果一致；第 3 题能解释容器生命周期与池的关系。

# 10. 小结

| 主题 | 要点 |
|------|------|
| `#[sqlx::test]` | 每测试建独立库 + 自动迁移 |
| 事务回滚 | 快，但要求 repo 接受 `Executor` |
| 内存 SQLite | `:memory:` 按连接隔离，限制连接或共享缓存 |
| Testcontainers | 真实数据库，容器要活到测试结束 |
| 并行 | 独立库天然并行，共享库需串行或加锁 |
| CI | 跑迁移 + `prepare --check` + 真实 `DATABASE_URL` |

下一篇是模块收尾，讲查询如何分层组织、类型映射的取舍、N+1 与批量写入的性能优化、安全清单，以及把前面七篇串起来的常见坑与项目检查表。
