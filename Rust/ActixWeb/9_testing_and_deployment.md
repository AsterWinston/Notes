写完业务代码只是前半程。一个能上线的 Actix Web 服务还需要三样东西：可重复的测试、分环境的配置、可预测的退出与部署方式。本篇把这三件事收束起来，作为这个系列的最后一站。

# 1. 测试分层

| 层级 | 测什么 | 工具 |
|------|--------|------|
| 单元测试 | 纯函数、校验逻辑、错误映射 | `#[test]` / `#[tokio::test]` |
| 接口测试 | 路由、提取器、状态码、响应体 | `actix_web::test` |

★ **关键认知**：`actix_web::test` 不需要真的监听端口。`init_service` 直接构建一个内存中的 `App`，`TestRequest` 构造请求，`call_service` 调用它。测试跑得和单元测试一样快，却覆盖了路由、中间件、提取器和序列化的完整链路。

在 `[dev-dependencies]` 里加上 `actix-web`、`tokio`（`macros` + `rt-multi-thread`）、`serde_json`，测试数据库再按需加 `sqlx`。

# 2. actix_web::test 基础

## 2.1 第一个接口测试

```rust
#[cfg(test)]
mod tests {
    use actix_web::{test, web, App};

    #[actix_web::test]
    async fn index_returns_ok() {
        // 1. 构建应用，不绑定端口
        let app = test::init_service(
            App::new().route("/", web::get().to(|| async { "ok" })),
        )
        .await;

        // 2. 构造请求并调用
        let req = test::TestRequest::get().uri("/").to_request();
        let resp = test::call_service(&app, req).await;
        assert!(resp.status().is_success());

        // 3. 读响应体
        assert_eq!(test::read_body(resp).await, "ok");
    }
}
```

⚠ 必须用 `#[actix_web::test]`，写成普通 `#[test]` 再 `.await` 会编译失败。

## 2.2 常用 API

| API | 作用 |
|-----|------|
| `test::init_service(App)` | 构建测试用 service |
| `TestRequest::get()` | 构造 GET 请求 |
| `.set_json(&value)` | 设置 JSON body 与 Content-Type |
| `.insert_header((k, v))` | 添加请求头 |
| `test::call_service(&app, req)` | 执行请求 |
| `test::read_body_json(resp)` | 读取并反序列化 JSON |

## 2.3 测试 JSON 接口

```rust
#[actix_web::test]
async fn create_user_returns_201() {
    let app = test::init_service(
        App::new().route("/users", web::post().to(create_user)),
    )
    .await;

    let payload = serde_json::json!({
        "name": "Tom",
        "email": "tom@example.com"
    });

    let req = test::TestRequest::post()
        .uri("/users")
        .set_json(&payload) // 自动带上 application/json
        .to_request();

    let resp = test::call_service(&app, req).await;
    assert_eq!(resp.status(), 201);

    let body: serde_json::Value = test::read_body_json(resp).await;
    assert_eq!(body["name"], "Tom");
}
```

★ **关键认知**：`set_json` 不只是序列化，还会设置 `Content-Type: application/json`。如果手写 `.set_payload(...)` 却忘了头，`web::Json` 提取器会直接返回 400，测试就会以"看起来像业务 bug"的方式失败。

## 2.4 测试共享状态

```rust
#[actix_web::test]
async fn counter_increases() {
    let state = web::Data::new(AppState::default());
    let app = test::init_service(
        App::new().app_data(state).route("/hit", web::get().to(hit)),
    )
    .await;

    for expected in 1..=3 {
        let req = test::TestRequest::get().uri("/hit").to_request();
        let resp = test::call_service(&app, req).await;
        assert_eq!(test::read_body(resp).await, format!("{}", expected));
    }
}
```

每次测试都要新建 `state`，不要在测试之间共享全局状态，否则用例顺序会影响结果。鉴权测试同理：先不带 `Authorization` 断言 401，再用 `.insert_header(("Authorization", "Bearer test-token"))` 断言 200。

# 3. 测试数据库

## 3.1 内存 SQLite 的坑

```rust
use sqlx::sqlite::SqlitePoolOptions;

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

## 3.2 每个测试独立建库

用 `test_pool()` 建库、跑迁移，再把 `AppState { db: pool.clone() }` 注入 `web::Data`。断言完调用 `pool.close().await` 及时释放连接。`tests/` 目录下的文件会被当独立 crate 编译，测试辅助函数放 `tests/common/mod.rs`，避免被当成测试用例。

# 4. 配置管理

```rust
#[derive(Clone, Debug)]
pub struct Config {
    pub bind: String,
    pub database_url: String,
    pub jwt_secret: String,
    pub workers: usize,
}

impl Config {
    pub fn from_env() -> Self {
        Self {
            bind: std::env::var("BIND").unwrap_or_else(|_| "0.0.0.0:8080".into()),
            database_url: std::env::var("DATABASE_URL").expect("DATABASE_URL 未设置"),
            jwt_secret: std::env::var("JWT_SECRET").expect("JWT_SECRET 未设置"),
            workers: std::env::var("WORKERS").ok()
                .and_then(|v| v.parse().ok()).unwrap_or(4),
        }
    }
}
```

| 配置 | 开发 | 生产 |
|------|------|------|
| `RUST_LOG` | `debug` | `info` |
| 数据库 | 本地 SQLite | PostgreSQL 集群 |
| `JWT_SECRET` | `.env` 随机值 | 密钥管理服务 |

★ **关键认知**：配置缺失要在**启动期失败**，不要给密钥类配置设默认值。`JWT_SECRET` 用默认值意味着所有部署共用同一个密钥，等于没有密钥。本地用 `dotenvy::dotenv().ok()` 读取 `.env`，生产直接用环境变量。⚠ `.env` 必须写进 `.gitignore`，仓库只提交 `.env.example`。

# 5. 日志

```rust
env_logger::init_from_env(
    env_logger::Env::default().default_filter_or("info"),
);
```

```bash
RUST_LOG=info,actix_web=debug,actix_server=info cargo run
```

| 目标 | 级别建议 |
|------|----------|
| 业务代码 | `info` |
| `actix_web` | `info` 或 `debug` |
| `sqlx` | `warn`（SQL 日志很吵） |
| 第三方库 | `warn` |

生产环境更适合 JSON 日志，便于采集与检索。可以用 `tracing` + `tracing-subscriber` 的 `json()` 输出，并在日志里带 `request_id`：

```rust
tracing::info!(request_id = %id, path = %req.path(), "request finished");
```

★ **关键认知**：没有 `request_id`，并发下根本串不起一次请求。请求 ID 中间件生成、写入扩展和响应头，日志与错误响应都引用同一个值，排查时才能一条线拉到底。

# 6. 优雅关闭

```rust
use actix_web::{web, App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let server = HttpServer::new(|| {
        App::new().route("/", web::get().to(|| async { "ok" }))
    })
    .bind(("0.0.0.0", 8080))?
    .workers(4)
    .shutdown_timeout(30) // 最多等 30 秒处理完在途请求
    .run();

    let handle = server.handle();
    actix_web::rt::spawn(async move {
        if tokio::signal::ctrl_c().await.is_ok() {
            log::info!("收到退出信号，开始优雅关闭");
            handle.stop(true).await; // true 表示等待在途请求
        }
    });

    server.await?;
    // 服务停止后执行清理，例如 state.db.close().await
    Ok(())
}
```

关闭流程：收到 `SIGTERM`/`Ctrl+C` → 停止接受新连接 → 等待在途请求完成（最多 `shutdown_timeout`）→ 全部完成正常退出，超时则强制关闭剩余连接。

⚠ `shutdown_timeout` 必须**小于**容器编排的终止宽限期（Kubernetes 默认 30 秒）。设成 60 秒而平台 30 秒就杀进程，等于没有优雅关闭，还会丢掉正在写的请求。示例里的 `tokio::signal` 需要在依赖里开启 `tokio = { version = "1", features = ["signal"] }`。

# 7. 部署

```bash
cargo build --release
```

```toml
[profile.release]
lto = "thin"     # 链接时优化，体积更小
strip = true     # 去掉符号表
codegen-units = 1
```

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/actix-demo /usr/local/bin/actix-demo
ENV RUST_LOG=info BIND=0.0.0.0:8080
EXPOSE 8080
USER 10001
CMD ["actix-demo"]
```

★ **关键认知**：容器里必须绑 `0.0.0.0`，绑 `127.0.0.1` 时端口映射进不来。这是容器部署最常见的"本地能跑、镜像里访问不了"原因。

| 项目 | 检查点 |
|------|--------|
| 监听地址 | `0.0.0.0`，端口与编排一致 |
| 配置 | 密钥来自环境/密钥管理，无默认值 |
| 数据库 | 连接池上限、迁移已执行 |
| 日志 | 输出到 stdout，级别合理 |
| 健康检查 | 提供 `/health`，不依赖重查询 |
| 优雅关闭 | 超时小于平台宽限期 |

反向代理还需要转发 `X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto`；WebSocket 要额外转发 `Upgrade` 与 `Connection: upgrade`。

# 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 测试编译报 await 错误 | 用了 `#[test]` | 改 `#[actix_web::test]` |
| 测试读不到 body | 只读了状态码就丢弃响应 | 先 `read_body` 再断言 |
| 内存库表不存在 | `:memory:` 多连接各自独立 | `max_connections(1)` 或共享缓存 |
| 容器内访问不到 | 绑了 `127.0.0.1` | 改 `0.0.0.0` |
| 发布时连接被重置 | 没等优雅关闭 | 配 `shutdown_timeout` 与信号处理 |

# 9. 动手练习

1. 给 `GET /users` 写接口测试，覆盖空列表、正常列表、数据库错误三种情况。
2. 用内存 SQLite 建测试库，跑迁移并测试创建用户的完整链路，确保用例之间互不影响。

验收标准：`cargo test` 全绿且可重复执行，容器收到 `SIGTERM` 后 30 秒内正常退出。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 接口测试 | `init_service` + `TestRequest` + `call_service` |
| 异步测试 | 必须用 `#[actix_web::test]` |
| 状态注入 | 每个用例新建 `web::Data` |
| 测试数据库 | 内存 SQLite 注意连接隔离 |
| 配置 | 启动期校验，密钥无默认值 |
| 日志 | `RUST_LOG` + 请求 ID + 结构化输出 |
| 优雅关闭 | 停止接新连接，等待在途请求 |
| 部署 | 绑 `0.0.0.0`、非 root、健康检查、反向代理 |

至此，Actix Web 这条线从框架定位、路由、提取器、状态、中间件、错误、数据库、鉴权与 WebSocket，一直走到测试与部署。下一步建议挑一个真实的小项目完整实现一遍：先定 API 契约，再写错误类型与数据层，最后补测试和部署脚本。把九篇里的片段连成一条能跑通的链路，比继续读更多概念更有价值。
