SQLx 是 Rust 生态里最常用的异步数据库访问库。它不隐藏 SQL，也不替你生成表结构，而是把"连接池 + 参数绑定 + 行映射 + 可选的编译期校验"这几件事做扎实。本篇从定位讲起，带你装好依赖、连上 PostgreSQL，并跑通第一条查询。

# 1. SQLx 是什么

## 1.1 四个核心能力

SQLx 提供的功能可以概括为四层：

| 能力 | 具体内容 | 对应 API |
|------|----------|----------|
| 连接池 | 复用物理连接，限制并发上限 | `PgPool`、`PgPoolOptions` |
| 参数绑定 | 用占位符传值，走数据库预处理协议 | `.bind()` |
| 行映射 | 把查询结果映射成 Rust 结构体 | `query_as`、`FromRow` |
| 编译期校验 | 编译时连库校验 SQL 与表结构 | `query!`、`query_as!` |

★ **关键认知**：SQLx 的定位是"**异步 SQL 工具库**"，而不是 ORM。它把 SQL 原样交给你写，只负责安全地执行和映射。这个选择决定了后面所有用法：你写 SQL、你写迁移、你处理方言差异，换来的是完全透明、没有魔法。

## 1.2 与其他方案对比

| 方案 | 是否异步 | 是否手写 SQL | 编译期检查 | 迁移 | 适合场景 |
|------|----------|--------------|------------|------|----------|
| SQLx | 是 | 是 | 可选 | 内置 | 想掌控 SQL、接受手写映射 |
| Diesel | 否（同步） | 否，用 DSL | 强类型编译期 | 内置 | 强类型查询构造器、复杂表达式 |
| SeaORM | 是 | 否，基于 SeaQuery | 类型约束 | 内置 | 想要 ActiveRecord 风格的 ORM |
| tokio-postgres | 是 | 是 | 无 | 无 | 只要驱动、自己造轮子 |

## 1.3 支持的数据库

SQLx 0.8 官方支持 PostgreSQL、MySQL、MariaDB、SQLite，以及实验性的 MSSQL。每接一种数据库都要开对应的 feature，否则 `connect` 会直接编译不过。

★ **关键认知**：SQLx 不同数据库的驱动是**编译期选定的**，不是运行时插件。一个二进制里可以同时装多种驱动（比如 PostgreSQL 加 SQLite 做测试），但不能像 JDBC 那样在运行时加载任意驱动。

# 2. 安装与 feature

## 2.1 最小 Cargo.toml

```toml
[package]
name = "sqlx-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = [
    "runtime-tokio",      # 绑定 Tokio 运行时
    "tls-rustls",         # 可选：需要 TLS 时打开
    "postgres",           # 选择数据库驱动
    "macros",             # 启用 query! 系列宏
    "migrate",            # 启用迁移功能
    "chrono",             # 可选：时间类型映射
    "uuid",               # 可选：UUID 类型映射
    "json",               # 可选：JSON/JSONB 映射
] }
```

## 2.2 feature 说明

| feature | 作用 | 不开会怎样 |
|---------|------|------------|
| `runtime-tokio` | 绑定 Tokio 运行时 | 没有运行时，池无法工作 |
| `runtime-async-std` | 绑定 async-std 运行时 | 与 Tokio 二选一 |
| `postgres` / `mysql` / `sqlite` | 编译对应驱动 | 相关类型不存在 |
| `macros` | `query!` 等宏 | 宏找不到 |
| `migrate` | `sqlx::migrate!` 与迁移执行 | 无法在代码里跑迁移 |
| `tls-rustls` / `tls-native-tls` | 加密连接 | 云数据库常连不上 |
| `chrono` / `time` | 日期时间映射 | `NaiveDateTime` 不实现 `Decode` |
| `uuid` | UUID 映射 | `Uuid` 不实现 `Decode` |
| `json` | `serde_json::Value` 映射 | JSONB 字段无法直接解 |
| `any` | 运行时可切换数据库 | 不能使用 `AnyPool` |

## 2.3 运行时怎么选

如果你的项目已经用了 Axum、Actix 或纯 Tokio，就选 `runtime-tokio`。两者混用会导致"没有 reactor"的运行时错误，因为 SQLx 的连接任务必须挂在你实际使用的运行时上。

★ **关键认知**：`runtime-tokio` 和 `runtime-async-std` **只能选一个**。Cargo 默认会合并 feature，一旦两个都被别的依赖打开，编译会报冲突。检查方式是 `cargo tree -e features | Select-String sqlx`，确认只有一个 runtime feature 生效。

## 2.4 环境变量与 .env

SQLx 约定从 `DATABASE_URL` 读取连接串，`query!` 宏和 `sqlx-cli` 都依赖它：

```bash
# .env 文件（不要提交到仓库）
DATABASE_URL=postgres://postgres:postgres@localhost:5432/demo
```

```rust
// main.rs 开头加载 .env，缺失时忽略错误
dotenvy::dotenv().ok();
```

```toml
[dependencies]
dotenvy = "0.15"
```

连接串格式：

```text
postgres://用户:密码@主机:端口/数据库名?参数
mysql://用户:密码@主机:3306/数据库名
sqlite://文件路径.db?mode=rwc
```

⚠ 把 `.env` 加进 `.gitignore`。连接串里带密码，一旦提交就是安全事故；生产环境用环境变量或密钥管理服务注入。

# 3. 准备数据库

## 3.1 建库

```sql
-- 在 psql 中执行
CREATE DATABASE demo;
```

## 3.2 建表

```sql
-- 一张最小的用户表
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 3.3 插入几条数据

```sql
INSERT INTO users (name, email) VALUES
    ('张三', 'zhangsan@example.com'),
    ('李四', 'lisi@example.com');
```

# 4. 第一条连接

## 4.1 用连接串直接连

```rust
use sqlx::PgPool;

let pool = PgPool::connect("postgres://postgres:postgres@localhost:5432/demo").await?;
```

`PgPool::connect` 会立即建立至少一条连接，失败就返回 `Err`。适合脚本和小工具；服务里更推荐 `PgPoolOptions`，可以精细控制参数，下一篇会专门讲。

## 4.2 完整可运行示例

```rust
use sqlx::postgres::PgPoolOptions;
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    dotenvy::dotenv().ok();

    // 1. 建池：最多 5 条连接，获取超时 5 秒
    let pool: PgPool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&std::env::var("DATABASE_URL").expect("缺少 DATABASE_URL"))
        .await?;

    // 2. 查询：用 $1 占位符，绑定参数
    let users: Vec<User> = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users ORDER BY id",
    )
    .fetch_all(&pool)
    .await?;

    // 3. 打印结果
    for user in &users {
        println!("#{} {} <{}>", user.id, user.name, user.email);
    }

    // 4. 关闭连接池，等待在途查询结束
    pool.close().await;
    Ok(())
}
```

## 4.3 逐段说明

| 代码 | 作用 | 注意点 |
|------|------|--------|
| `PgPoolOptions::new()` | 构造池配置 | 后续 `.connect()` 才真正建池 |
| `.max_connections(5)` | 并发连接上限 | 不是越大越好 |
| `query_as::<_, User>` | 声明结果映射类型 | 泛型第一个参数是数据库类型，用 `_` 让编译器推断 |
| `.fetch_all(&pool)` | 取全部行 | 结果为空时返回空 `Vec`，不报错 |
| `pool.close().await` | 优雅关闭 | 会等待已借出的连接归还 |

## 4.4 运行

```bash
cargo run
```

输出：

```text
#1 张三 <zhangsan@example.com>
#2 李四 <lisi@example.com>
```

## 4.5 为什么是 `$1` 而不是 `?`

| 数据库 | 占位符 | 示例 |
|--------|--------|------|
| PostgreSQL | `$1`、`$2` | `WHERE id = $1 AND name = $2` |
| MySQL | `?` | `WHERE id = ? AND name = ?` |
| SQLite | `?` | `WHERE id = ? AND name = ?` |

★ **关键认知**：占位符只是**位置**，真正的值通过 `.bind()` 单独传给驱动。绑定走的是数据库的预处理协议，值永远不会被当成 SQL 语法解析，所以用户输入里带 `'; DROP TABLE` 也只是普通字符串。⚠ 反过来，任何 `format!` 拼进 SQL 的"动态条件"都会绕过这层保护，是注入的唯一入口。

# 5. 编译期检查初体验

## 5.1 query! 宏

```rust
// 编译时会真实连接数据库，校验 SQL 与列类型
let row = sqlx::query!(
    "SELECT id, name FROM users WHERE id = $1",
    id
)
.fetch_one(&pool)
.await?;

println!("{} {}", row.id, row.name);
```

`query!` 返回匿名结构体，字段由 SQL 列名推导。`query_as!` 则映射到你自己的类型：

```rust
let user = sqlx::query_as!(
    User,
    "SELECT id, name, email FROM users WHERE id = $1",
    id
)
.fetch_optional(&pool)
.await?;
```

## 5.2 离线模式

`query!` 需要编译时能访问数据库。CI 往往连不上，这时用离线元数据：

```bash
cargo install sqlx-cli --no-default-features --features postgres,rustls
cargo sqlx prepare            # 生成 .sqlx/ 目录，提交到仓库
```

之后构建时设置：

```bash
# CI 环境变量
SQLX_OFFLINE=true
```

★ **关键认知**：`.sqlx/` 是**编译期查询的缓存**，不是数据库快照。改了 schema 或 SQL 后必须重新 `cargo sqlx prepare`，否则 CI 会拿旧元数据编译通过，运行时才发现列不存在。⚠ 这个坑非常隐蔽，建议在 CI 里加一步"重新 prepare 后检查 git diff 是否为空"。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `error: no reactor running` | 运行时 feature 没选对，或在非异步上下文用了异步 API | 选 `runtime-tokio`，并在 `#[tokio::main]` 内调用 |
| `error: trait bound ... Decode ... not satisfied` | 缺少类型映射 feature，或字段类型不匹配 | 打开 `chrono`/`uuid`/`json`，核对 SQL 列类型 |
| `error: unsupported `$1` syntax` | 对 MySQL 用了 `$1` | MySQL 一律 `?` |
| 编译时连不上数据库 | `query!` 需要在线元数据 | 设 `DATABASE_URL` 或 `SQLX_OFFLINE=true` |
| `password authentication failed` | 连接串账号密码错误 | 检查 `.env` 与数据库实际配置 |
| `relation "users" does not exist` | 没建表或连错库 | 确认库名、schema、是否跑过迁移 |

⚠ 最后一个错误还有一个常见变体：本机连的是 `demo` 库，`query!` 编译时读的 `DATABASE_URL` 却指向另一个库，于是"明明有表却报不存在"。先 `echo $env:DATABASE_URL` 确认再排查。

# 7. 动手练习

1. 建一张 `tasks` 表（id、title、done、created_at），用 `query_as` 查出所有未完成项并按创建时间倒序排列。
2. 把示例改成 MySQL 版本：换 feature、换占位符、换连接串，确认代码能跑通。
3. 故意把 `SELECT id, name` 改成 `SELECT id, title`，分别观察 `query_as`（运行时）和 `query_as!`（编译时）报错时机的差异。

验收标准：`cargo run` 能打印新建的数据；第 3 题能说清楚两种宏的检查时机差别。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 异步 SQL 工具库，不是 ORM |
| feature | runtime 二选一，数据库驱动按需开 |
| 连接串 | `DATABASE_URL`，格式 `postgres://user:pass@host/db` |
| 查询 | `query_as` 运行时映射，`query_as!` 编译期校验 |
| 参数 | PostgreSQL `$1`，MySQL/SQLite `?`，值一律 `.bind()` |
| 离线构建 | `cargo sqlx prepare` + `SQLX_OFFLINE=true` |

下一篇进入连接池，讲清楚 `Pool` 为什么可以随意 clone、`max_connections` 与超时参数怎么配、连接复用与健康检查的边界，以及多数据源和优雅关闭的正确姿势。
