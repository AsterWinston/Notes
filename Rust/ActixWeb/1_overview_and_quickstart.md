Actix Web 是 Rust 生态里性能最激进的 Web 框架之一，也是从"会写 Rust"跨到"能用 Rust 写服务"的常用起点。本篇先把它放回坐标系，讲清与 Axum 的取舍，再跑通第一个服务，最后给出可维护的项目结构。

# 1. Actix Web 是什么

## 1.1 定位与核心特点

Actix Web 建立在 **actor 模型思想 + Tokio 异步运行时**之上，招牌是吞吐量，在 TechEmpower 等基准里长期处于第一梯队。

| 特征 | 含义 |
|------|------|
| 异步优先 | Handler 是 `async fn`，底层跑在 Tokio 上 |
| 类型安全提取器 | 路径、查询、JSON、表单由类型驱动解析 |
| 中间件洋葱模型 | 基于 `Transform` + `Service`，可组合 |
| 多 worker | `HttpServer` 默认按 CPU 核数起 worker |

## 1.2 版本与依赖边界

当前主流是 **Actix Web 4.x** + Rust 2021 edition。几个名字容易混的 crate：

| crate | 作用 |
|-------|------|
| `actix-web` | Web 框架本体 |
| `actix-web-actors` | 在 Actix Web 里接入 WebSocket |
| `actix-cors` | CORS 中间件 |
| `actix-session` | 会话中间件 |
| `actix` | 通用 actor 框架，WebSocket 里直接用到 |

⚠ `actix-web` 和 `actix` 不是一回事。普通 HTTP 接口只需要 `actix-web`；做 WebSocket actor 时才显式引入 `actix`。

## 1.3 与 Axum 的对比

| 维度 | Actix Web | Axum |
|------|-----------|------|
| 运行时 | actix-rt（封装 Tokio） | 直接 Tokio |
| 抽象核心 | `Service` + `Transform` + actor 传统 | `tower::Service` + 函数式提取器 |
| 性能 | 极高，基准常年领先 | 很高，同一梯队 |
| 学习曲线 | 较陡，中间件类型签名复杂 | 较平缓，概念更少 |
| 中间件生态 | 自带一批，部分可接 `tower-http` | 直接复用整个 `tower` 生态 |
| 路由写法 | 宏路由 + builder | 以 `Router` builder 为主 |
| WebSocket | `actix-web-actors`，actor 模型 | 原生 `WebSocketUpgrade`，更轻 |

★ **关键认知**：选型不要只看跑分。Actix Web 的优势是框架已把常见能力配齐，代价是类型签名更绕；Axum 的优势是与 `tower` 生态无缝，代价是自己拼装更多。两者性能差距在真实业务里通常被数据库和网络淹没。

# 2. 第一个服务

## 2.1 创建项目

```bash
cargo new actix-demo
cd actix-demo
```

```toml
[package]
name = "actix-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
env_logger = "0.11"
log = "0.4"
```

## 2.2 最小 main.rs

```rust
use actix_web::{get, App, HttpResponse, HttpServer, Responder};

#[get("/")]
async fn index() -> impl Responder {
    HttpResponse::Ok().body("Hello, Actix Web!")
}

#[get("/health")]
async fn health() -> impl Responder {
    // 健康检查返回 JSON，方便探活脚本解析
    HttpResponse::Ok().json(serde_json::json!({ "status": "ok" }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(
        env_logger::Env::default().default_filter_or("info"), // 默认 info 级别
    );

    HttpServer::new(|| {
        App::new().service(index).service(health) // 注册宏路由
    })
    .bind(("127.0.0.1", 8080))? // 绑定失败直接返回 io::Error
    .run()
    .await
}
```

## 2.3 运行与验证

```bash
cargo run
curl http://127.0.0.1:8080/
curl http://127.0.0.1:8080/health
```

★ **关键认知**：`#[actix_web::main]` 把 `async fn main` 包装成运行时入口。忘记它，`main` 里就不能 `.await`；而 `HttpServer::new` 的闭包必须是 `Fn`，因为每个 worker 都要调用一次。

## 2.4 不用宏的等价写法

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};

async fn index() -> impl Responder {
    HttpResponse::Ok().body("Hello, Actix Web!")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/", web::get().to(index)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

## 2.5 一次请求经历了什么

```text
客户端 TCP 连接
      │
      ▼
┌─────────────────────────┐
│ HttpServer              │  监听端口、管理 worker 与连接
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ App                     │  路由表 + 中间件链 + app_data
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 中间件（洋葱，逐层进入） │
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ Handler                 │  提取器解析参数 → 业务逻辑
└───────────┬─────────────┘
            ▼
      Responder → HttpResponse
```

★ **关键认知**：`HttpServer` 负责"怎么收"，`App` 负责"怎么分"，Handler 负责"怎么处理"。排错先定位层级：端口不通看 `HttpServer`，404 看路由表，500 看 Handler 与提取器。

# 3. HttpServer 与 App

## 3.1 职责划分

```rust
HttpServer::new(|| App::new() /* 路由、中间件、数据 */)
    .workers(4)             // 手动指定 worker 数量
    .shutdown_timeout(30)   // 优雅关闭最长等待秒数
    .bind(("0.0.0.0", 8080))?
    .run()
    .await
```

`App` 是每个 worker 内部独立构建的，持有路由表、中间件链、`app_data` 与应用配置；`HttpServer` 是进程级外壳，负责监听、worker 和连接管理。

## 3.2 worker 与共享状态

```text
进程
├── worker 0 ── 独立 App 实例 ── 共享 web::Data（Arc）
├── worker 1 ── 独立 App 实例 ── 共享 web::Data（Arc）
└── worker 2 ── 独立 App 实例 ── 共享 web::Data（Arc）
```

★ **关键认知**：`App` 里的普通字段是每个 worker 各一份；只有 `web::Data` 包装后才是跨 worker 共享。⚠ 不要在 `HttpServer::new` 闭包里 `move` 不可克隆的独占资源，否则闭包不满足 `Fn`，编译会报 "cannot move out of captured variable"。

# 4. 项目结构

## 4.1 推荐目录

```text
actix-demo/
├── Cargo.toml
├── .env
├── migrations/
└── src/
    ├── main.rs          // 启动、配置、组装
    ├── config.rs        // 环境变量与配置结构
    ├── routes/
    │   ├── mod.rs       // 统一注册
    │   └── users.rs     // 用户路由
    ├── handlers/users.rs
    ├── models/user.rs
    ├── db.rs            // 连接池初始化
    └── errors.rs        // 统一错误类型
```

## 4.2 用 configure 拆分路由

```rust
// src/routes/users.rs
use actix_web::web;

pub fn config(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/users")
            .route("", web::get().to(list_users))
            .route("/{id}", web::get().to(get_user)),
    );
}

async fn list_users() -> impl actix_web::Responder { "user list" }

async fn get_user(path: web::Path<i64>) -> impl actix_web::Responder {
    format!("user {}", path.into_inner())
}
```

```rust
// src/routes/mod.rs
use actix_web::web;

pub mod users;

pub fn config(cfg: &mut web::ServiceConfig) {
    cfg.configure(users::config); // 把子模块路由挂进来
}
```

```rust
// src/main.rs
use actix_web::{App, HttpServer};
mod routes;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().configure(routes::config))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

★ **关键认知**：`configure` 让路由按业务域拆分，`main.rs` 只负责组装。项目越大，"注册与实现分离"越重要，否则所有路由堆在一个文件里，改一行都要重新读懂全局。

## 4.3 配置外置

```rust
#[derive(Clone)]
pub struct Config {
    pub bind: String,
    pub database_url: String,
}

impl Config {
    pub fn from_env() -> Self {
        Self {
            bind: std::env::var("BIND").unwrap_or_else(|_| "127.0.0.1:8080".into()),
            database_url: std::env::var("DATABASE_URL")
                .expect("DATABASE_URL 必须设置"), // 启动期失败优于运行期失败
        }
    }
}
```

# 5. 常见错误

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| 编译报 `main` 不能 await | 漏了 `#[actix_web::main]` | 检查入口属性 |
| 启动报地址被占用 | 端口被其他进程占用 | 换端口或结束占用进程 |
| 404 但路径看着没错 | 路由未注册或方法不匹配 | 确认 `web::get()` 与路径 |
| 闭包编译不过 | 捕获了不可克隆值 | 用 `web::Data` + `clone` |
| 日志一片空白 | 没初始化 `env_logger` | 检查 `RUST_LOG` 与初始化 |
| `.bind()` 类型不匹配 | 主机名未解析 | 用 `("127.0.0.1", 8080)` |

⚠ `bind` 失败会直接让 `main` 返回错误，看起来像"程序秒退"。启动脚本要打印错误上下文，否则线上只能看到退出码。

# 6. 动手练习

1. 新建项目，实现 `/`、`/health`、`/version` 三个接口，`/version` 返回 `{"version": "0.1.0"}`。
2. 把三个接口拆到 `routes/` 下，用 `configure` 注册，`main.rs` 只保留启动逻辑。
3. 用环境变量 `BIND` 控制监听地址，默认 `127.0.0.1:8080`，并用 `curl` 验证两个端口。

验收标准：`cargo run` 无警告，三个接口返回预期内容，`main.rs` 不超过 30 行。

# 7. 小结

| 概念 | 要点 |
|------|------|
| Actix Web | 基于 Tokio 的高性能异步 Web 框架 |
| `HttpServer` | 进程级监听与 worker 管理 |
| `App` | 每个 worker 独立的路由与状态容器 |
| Handler | 异步函数，输入提取器、输出 `Responder` |
| `#[actix_web::main]` | 异步入口宏，别漏 |
| 项目结构 | 按业务域拆 `routes`，用 `configure` 组装 |

下一篇进入路由系统，讲清楚 `{id}` 路径参数、`web::Query`、`scope` 与 `resource` 的差别，以及 Actix Web 里"注册顺序决定匹配顺序"这个和很多框架不一样的关键点。
