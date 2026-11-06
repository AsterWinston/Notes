Axum 是 Tokio 团队维护的 Web 框架，也是 Rust 服务端开发里"概念最少、和异步生态贴得最近"的一个选择。本篇先把它放回坐标系，讲清它与 Actix Web 的取舍，再跑通第一个服务，最后落到可维护的项目结构。

# 1. Axum 是什么

## 1.1 定位与核心特点

Axum 不是一个大而全的框架，而是一层薄薄的"路由 + 提取器 + 响应"封装，底下直接站着 Tokio、Hyper 和 Tower。

| 特征 | 含义 |
|------|------|
| 运行时 | 直接使用 Tokio，没有自建运行时 |
| HTTP 底座 | Hyper，连接管理与协议解析交给它 |
| 抽象核心 | `tower::Service`，中间件即 Layer |
| Handler | 普通 `async fn`，参数由提取器解析 |
| 状态 | `State` 提取器 + `with_state`，编译期检查 |

★ **关键认知**：Axum 的设计哲学是"**组合而不是内置**"。它只负责把请求分发给 Handler，其余能力（日志、CORS、超时、压缩、静态文件）都通过 Tower 中间件拼装。好处是生态统一、每一层都可替换；代价是必须理解 Tower 的 `Service` / `Layer` 模型，否则中间件顺序会写反。

## 1.2 版本与依赖边界

本系列以 **Axum 0.8 + Tokio 1.x + Rust 2021** 为准。`axum` 是框架本体，`tokio` 提供运行时与监听，`tower` / `tower-http` 提供中间件，`serde` 负责序列化，`hyper` 一般不用直接依赖。

Axum 0.8 的几处关键变化：

| 变化 | 0.7 及以前 | 0.8 |
|------|-----------|-----|
| 路径参数 | `/:id` | `/{id}` |
| 通配符 | `/*rest` | `/{*rest}` |
| 自定义提取器 | 需要 `#[async_trait]` | 原生 `async fn` |
| WebSocket 文本帧 | `Message::Text(String)` | `Message::Text(Utf8Bytes)` |
| 启动方式 | `axum::Server::bind` | `axum::serve(TcpListener, app)` |

⚠ 从旧教程抄代码时，最常见的报错就是路径写成了 `:id`。0.8 不再把 `:id` 当参数，迁移时需要全局替换成 `{id}`。

## 1.3 与 Actix Web 的对比

| 维度 | Actix Web | Axum |
|------|-----------|------|
| 运行时 | actix-rt（封装 Tokio） | 直接 Tokio |
| 抽象核心 | `Service` + `Transform` + actor 传统 | `tower::Service` + 函数式提取器 |
| 性能 | 极高，基准常年领先 | 很高，同一梯队 |
| 学习曲线 | 较陡，中间件类型签名复杂 | 较平缓，概念更少 |
| 中间件生态 | 自带一批，部分可接 `tower-http` | 直接复用整个 `tower` 生态 |
| 状态注入 | `web::Data`，运行期按类型查找 | `State`，编译期检查 |

★ **关键认知**：选型不要只看跑分。Actix Web 的优势是"框架把常见能力配齐了"，代价是类型签名更绕；Axum 的优势是"和 Tower 生态无缝"，代价是自己拼装更多。两者在真实业务里的性能差距，通常被数据库和网络淹没。⚠ 如果团队里没人熟悉 `tower::Service`，Axum 的中间件会变成主要学习成本；反过来，如果已经有大量 Tower 组件，Axum 的集成成本最低。

# 2. 第一个服务

## 2.1 创建项目

```bash
cargo new axum-demo
cd axum-demo
```

```toml
[package]
name = "axum-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

## 2.2 最小 main.rs

```rust
use axum::{routing::get, Json, Router};
use serde_json::{json, Value};

async fn index() -> &'static str {
    "Hello, Axum!"
}

async fn health() -> Json<Value> {
    // 健康检查返回 JSON，方便探活脚本解析
    Json(json!({ "status": "ok" }))
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    tracing_subscriber::fmt().init(); // 开发期默认输出到 stdout

    // Router 只描述"请求怎么分"，不负责监听端口
    let app = Router::new()
        .route("/", get(index))
        .route("/health", get(health));

    // Listener 负责监听，serve 把两者接起来
    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await?;
    tracing::info!("listening on http://{}", listener.local_addr()?);
    axum::serve(listener, app).await?;
    Ok(())
}
```

## 2.3 运行与验证

```bash
cargo run
curl http://127.0.0.1:8080/
```

★ **关键认知**：Axum 的 Handler 就是返回 `impl IntoResponse` 的 `async fn`。`&'static str`、`String`、`Json<T>`、`StatusCode`、`(StatusCode, Json<T>)` 都实现了 `IntoResponse`，所以不需要像 Actix Web 那样显式构造 `HttpResponse`。⚠ Handler 返回 `Result<T, E>` 时，`T` 和 `E` 都要实现 `IntoResponse`，否则编译错误会指向一个很难读的 trait bound。

## 2.4 一次请求经历了什么

```text
客户端 → TcpListener（监听端口、接受连接）
        ↓
axum::serve（把连接交给 Hyper）
        ↓
Router（路径匹配 + 方法匹配）
        ↓
Tower 中间件（外层到内层）
        ↓
Handler（提取器 + 业务逻辑）
        ↓
IntoResponse（HTTP 响应）
```

★ **关键认知**：`Router` 负责"怎么分"，`TcpListener` 负责"怎么收"，Handler 负责"怎么处理"。排错先定位层级：端口连不上看 Listener，404 看 Router，400 看提取器，500 看 Handler 和中间件。

# 3. Router 与 Listener

## 3.1 Router 的职责

`Router` 是一棵路由树，内部按路径和方法组织 Handler。它本身不是服务器，也不持有端口。

```rust
let app: Router = Router::new()
    .route("/", get(index))
    .route("/users", get(list_users).post(create_user));
```

带状态的 Router 类型是 `Router<AppState>`，只有调用 `with_state(state)` 之后才变成可以直接交给 `serve` 的 `Router<()>`。

★ **关键认知**：`Router<S>` 里的 `S` 是"还缺什么状态"的类型标记。带 `State<AppState>` 参数的 Handler 会让 Router 变成 `Router<AppState>`，此时直接 `axum::serve(listener, app)` 会编译失败，必须先 `with_state`。这是 Axum 把"状态注入"从运行期错误提前到编译期的关键设计。

## 3.2 Listener 与 serve

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
axum::serve(listener, app)
    .with_graceful_shutdown(async {
        tokio::signal::ctrl_c().await.ok();
    })
    .await?;
```

`axum::serve` 接受任何 `TcpListener` 和任何实现了 `Service<Request<Body>>` 的东西。Router 正好实现了它，所以大多数时候不需要关心 Hyper 的连接细节。⚠ 容器部署必须绑 `0.0.0.0`；绑 `127.0.0.1` 时端口映射进不了容器，现象是"本地能跑，镜像里访问不了"。

## 3.3 从 main 到业务的分层

```text
main.rs
  ├── 读取配置、初始化日志
  ├── 创建连接池 / 共享状态
  ├── 组装 Router
  └── 启动 serve
        ↓
routes/*.rs（路径到 Handler）
        ↓
handlers / services（业务逻辑）
        ↓
db / models（数据访问）
```

★ **关键认知**：`main.rs` 应该只做"组装"，不写业务。这样测试可以直接调用 `app(state)` 构造 Router，而不需要启动端口；部署时也只换配置，不动代码。

# 4. 项目结构

## 4.1 推荐目录

```text
axum-demo/
├── Cargo.toml
├── .env
├── migrations/
└── src/
    ├── main.rs          // 启动、配置、组装
    ├── lib.rs           // 暴露 app()，方便测试
    ├── config.rs        // 环境变量与配置结构
    ├── state.rs         // 应用状态
    ├── error.rs         // 统一错误类型
    └── routes/
        ├── mod.rs       // 统一注册
        ├── health.rs
        └── users.rs
```

把 `app()` 放进 `lib.rs` 是一个很实用的习惯：集成测试可以 `use axum_demo::app;`，不用把整个 `main.rs` 复制一遍。

## 4.2 按业务域拆分 Router

```rust
// src/routes/users.rs
use axum::{routing::get, Router};
use crate::state::AppState;

pub fn router() -> Router<AppState> {
    Router::new()
        .route("/users", get(list_users))
        .route("/users/{id}", get(get_user)) // 0.8 用花括号
}

// list_users / get_user 的实现在 handlers 目录，本篇不展开
```

```rust
// src/lib.rs
pub mod config;
pub mod error;
pub mod routes;
pub mod state;

use axum::Router;
use state::AppState;

pub fn app(state: AppState) -> Router {
    Router::new()
        .merge(routes::health::router())
        .merge(routes::users::router())
        .with_state(state) // 所有子 Router 共享同一个状态
}
```

★ **关键认知**：子模块的 `router()` 返回 `Router<AppState>`，在 `lib.rs` 里 `merge` 之后再统一 `with_state`。⚠ 不要在每个子模块里各自 `with_state`，那会得到多个互不相干的 `Router<()>`，再合并时状态类型对不上，编译报错会非常难读。

## 4.3 配置外置

```rust
// src/config.rs
#[derive(Clone, Debug)]
pub struct Config {
    pub bind: String,
    pub app_name: String,
}

impl Config {
    pub fn from_env() -> Self {
        Self {
            bind: std::env::var("BIND").unwrap_or_else(|_| "127.0.0.1:8080".into()),
            app_name: std::env::var("APP_NAME").unwrap_or_else(|_| "axum-demo".into()),
        }
    }
}
```

```rust
// src/state.rs
#[derive(Clone)]
pub struct AppState {
    pub app_name: String,
}
```

`AppState` 要求 `Clone + Send + Sync + 'static`，因为它会被复制进每个请求。字符串克隆很便宜，但连接池、缓存客户端这类资源要放进 `Arc` 或本身就是 `Clone` 的池对象里，避免每个请求复制昂贵资源。

⚠ 密钥类配置不要给默认值。`JWT_SECRET` 用 `unwrap_or` 兜底，等于所有部署共用同一个密钥，比启动失败更危险。开发环境用 `.env`，生产环境用平台注入的环境变量。

# 5. 常见错误

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| 编译报 Handler trait 不满足 | 返回值没实现 `IntoResponse` | 用 `Json`、`StatusCode` 或元组 |
| `serve` 类型不匹配 | Router 还带着状态没有 `with_state` | 检查 `Router<AppState>` |
| 路径参数拿不到 | 用了 0.7 的 `:id` 语法 | 改成 `{id}` |
| 启动报地址被占用 | 端口被其他进程占用 | 换端口或结束占用进程 |
| 容器内访问不到 | 绑了 `127.0.0.1` | 改 `0.0.0.0` |

⚠ `TcpListener::bind` 失败会让 `main` 直接返回错误，看起来像"程序秒退"。启动脚本要打印错误上下文，否则线上只能看到退出码。

# 6. 动手练习

1. 新建项目，实现 `/`、`/health`、`/version` 三个接口，`/version` 返回 `{"version": "0.1.0"}`。
2. 把三个接口拆到 `routes/` 下，用 `merge` 组装，`main.rs` 只保留启动逻辑。
3. 把 `app()` 移到 `lib.rs`，写一个 `cargo test` 能调用的最小测试，确认不启动端口也能构造 Router。

验收标准：`cargo run` 无警告，三个接口返回预期内容，`main.rs` 不超过 30 行。

# 7. 小结

| 概念 | 要点 |
|------|------|
| Axum | 建立在 Tokio + Hyper + Tower 之上的路由与提取器层 |
| `Router` | 路由树，不负责监听；`Router<S>` 表示还缺状态 |
| `TcpListener` + `serve` | 监听端口并把连接交给 Router |
| Handler | `async fn`，输入提取器、输出 `IntoResponse` |
| 0.8 语法 | 路径参数是 `{id}`，不是 `:id` |
| 项目结构 | `lib.rs` 暴露 `app()`，`main.rs` 只做组装 |

下一篇进入路由系统，讲清楚 `{id}` 路径参数、嵌套 Router 的 `nest` 与 `merge`、`fallback` 的兜底逻辑，以及 Axum 与 Actix Web 在匹配顺序上的根本差别。
