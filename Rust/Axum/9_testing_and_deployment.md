写完业务代码只是前半程。一个能上线的 Axum 服务还需要三样东西：可重复的测试、能串起请求的日志、可预测的退出与部署方式。本篇把这三件事收束起来，作为这个系列的最后一站。

# 1. 测试分层

| 层级 | 测什么 | 工具 |
|------|--------|------|
| 单元测试 | 纯函数、校验逻辑、错误映射 | `#[test]` / `#[tokio::test]` |
| 接口测试 | 路由、提取器、状态码、响应体 | `Router::oneshot` |
| 集成测试 | 数据库、迁移、事务 | `sqlx::test` + 测试库 |

★ **关键认知**：测 Router 不需要真的监听端口。`tower::ServiceExt::oneshot` 直接调用 Router 的 `Service` 实现，请求在内存里走完路由、中间件、提取器和 Handler。测试跑得和单元测试一样快，却覆盖了完整链路。

# 2. Router 测试

## 2.1 oneshot

```toml
[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
```

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        http::{Request, StatusCode},
    };
    use tower::ServiceExt;

    #[tokio::test]
    async fn index_ok() {
        let router = build_app(test_state());

        let response = router
            .oneshot(Request::builder().uri("/").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);
    }
}
```

`oneshot` 会消费 Router；同一个测试里要发多个请求，先用 `router.clone()`。Router 的 clone 很便宜，内部是共享的路由表。

## 2.2 构造请求

```rust
let request = Request::builder()
    .method("POST")
    .uri("/users")
    .header("content-type", "application/json")
    .body(Body::from(r#"{"name":"Tom","email":"tom@example.com"}"#))
    .unwrap();

let response = router.clone().oneshot(request).await.unwrap();
assert_eq!(response.status(), StatusCode::CREATED);
```

带鉴权头时直接加 `authorization` 头即可。

★ **关键认知**：`oneshot` 只负责调用 Service，不会帮你补 `Content-Type`。手写 JSON 请求却忘了这个头，`Json` 提取器会返回 415，测试失败看起来像业务 bug。⚠ 用 `.body(Body::from(...))` 时，body 的所有权被移走，后续不能再复用同一个 `Request`。

## 2.3 读取响应

```rust
let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
    .await
    .unwrap();
let value: serde_json::Value = serde_json::from_slice(&bytes).unwrap();
assert_eq!(value["name"], "Tom");
```

`to_bytes` 的第二个参数是大小上限。测试里用 `usize::MAX` 没问题，生产代码要设合理上限，避免把内存打爆。

# 3. 测试状态与数据库

## 3.1 注入 State

```rust
async fn test_state() -> AppState {
    AppState {
        app_name: "test".into(),
        counter: Arc::new(AtomicUsize::new(0)),
        db: test_pool().await,
    }
}
```

每个测试都新建状态，不要在测试之间共享全局 `static`，否则用例顺序会影响结果。鉴权测试同理：先不带 `Authorization` 断言 401，再带上 token 断言 200。

## 3.2 测试数据库

```rust
async fn test_pool() -> sqlx::SqlitePool {
    let pool = sqlx::SqlitePoolOptions::new()
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

```rust
#[tokio::test]
async fn list_users_returns_empty() {
    let state = test_state().await; // 内部建库并跑迁移
    let pool = state.db.clone();
    let router = build_app(state);

    let response = router
        .oneshot(Request::builder().uri("/users").body(Body::empty()).unwrap())
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);
    pool.close().await; // 及时释放连接
}
```

★ **关键认知**：SQLite 的 `:memory:` 数据库**按连接隔离**。连接池开两条连接就会得到两个空库，测试报"表不存在"。要么 `max_connections(1)`，要么用共享缓存。⚠ `tests/` 目录下的每个文件会被编译成独立 crate，公共辅助函数放 `tests/common/mod.rs`，否则会被当成测试用例收集。

# 4. 日志

## 4.1 tracing

```rust
use tracing_subscriber::EnvFilter;

let filter = EnvFilter::try_from_default_env()
    .unwrap_or_else(|_| EnvFilter::new("info"));
tracing_subscriber::fmt().with_env_filter(filter).init();
```

```bash
RUST_LOG=info,axum=debug,tower_http=debug,sqlx=warn cargo run
```

| 目标 | 建议级别 |
|------|----------|
| 业务代码 | `info` |
| `axum` | `info` 或 `debug` |
| `tower_http` | `debug`（请求链路） |
| `sqlx` | `warn`（SQL 日志很吵） |

## 4.2 请求 ID

```rust
use axum::{
    extract::Request,
    http::HeaderValue,
    middleware::Next,
    response::Response,
};

async fn request_id(mut request: Request, next: Next) -> Response {
    let id = uuid::Uuid::new_v4().to_string();
    request.extensions_mut().insert(RequestId(id.clone()));

    let mut response = next.run(request).await;
    response.headers_mut().insert(
        "x-request-id",
        HeaderValue::from_str(&id).unwrap(),
    );
    response
}
```

```rust
#[derive(Clone)]
struct RequestId(String);
```

★ **关键认知**：没有 `request_id`，并发下根本串不起一次请求。中间件生成 ID，写入请求扩展和响应头，日志与错误响应都引用同一个值，排查时才能一条线拉到底。⚠ 不要只在入口打印一次 ID，Handler、数据库错误、下游调用都要带上它。

# 5. 优雅关闭

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("监听 Ctrl+C 失败");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("监听 SIGTERM 失败")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
    tracing::info!("收到退出信号，开始优雅关闭");
}

axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await?;
```

关闭流程：收到 `SIGTERM` / `Ctrl+C` → 停止接受新连接 → 等待在途请求完成 → 退出。Axum 没有单独的 `shutdown_timeout` 参数，超时通常由编排平台或外层信号处理控制。

⚠ 容器平台通常在 30 秒后强制杀进程。如果你的请求可能跑更久，要么缩短处理时间，要么在收到信号后主动给请求加截止时间，不要指望平台无限等待。

# 6. 部署

## 6.1 构建

```bash
cargo build --release
```

```toml
[profile.release]
lto = "thin"     # 链接时优化，体积更小
strip = true     # 去掉符号表
codegen-units = 1
```

## 6.2 容器

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
COPY --from=builder /app/target/release/axum-demo /usr/local/bin/axum-demo
ENV RUST_LOG=info BIND=0.0.0.0:8080
EXPOSE 8080
USER 10001
CMD ["axum-demo"]
```

★ **关键认知**：容器里必须绑 `0.0.0.0`。绑 `127.0.0.1` 时端口映射进不来，这是"本地能跑、镜像里访问不了"最常见的原因。⚠ 不要把 `.env` 打进镜像；密钥由编排平台注入，镜像只带非敏感默认值。

## 6.3 检查清单

| 项目 | 检查点 |
|------|--------|
| 监听地址 | `0.0.0.0`，端口与编排一致 |
| 配置 | 密钥来自环境/密钥管理，无默认值 |
| 数据库 | 连接池上限、迁移已执行 |
| 日志 | 输出到 stdout，级别合理，带 request_id |
| 健康检查 | 提供 `/health`，不依赖重查询 |
| 优雅关闭 | 处理 SIGTERM，等待在途请求 |
| 反向代理 | 转发 `X-Forwarded-For`、`X-Forwarded-Proto` |

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 测试报 await 错误 | 用了 `#[test]` | 改 `#[tokio::test]` |
| 测试读不到 body | 只读状态码就丢弃响应 | 先 `to_bytes` 再断言 |
| JSON 测试 415 | 手写请求忘了 Content-Type | 加 `content-type` 头 |
| 内存库表不存在 | `:memory:` 多连接各自独立 | `max_connections(1)` 或共享缓存 |
| 容器内访问不到 | 绑了 `127.0.0.1` | 改 `0.0.0.0` |
| 发布时连接被重置 | 没等优雅关闭 | 处理 SIGTERM，留出排空时间 |

⚠ 测试里不要用 `unwrap()` 把错误信息吞掉。用 `expect("...")` 写清上下文，失败时才能一眼看出是建库、迁移还是请求构造出了问题。

# 8. 动手练习

1. 给 `GET /users` 写接口测试，覆盖空列表、正常列表、数据库错误三种情况。
2. 用内存 SQLite 建测试库，跑迁移并测试创建用户的完整链路，确保用例之间互不影响。
3. 给服务加 `request_id` 中间件和优雅关闭，用 `docker stop` 验证进程能在排空后退出。

验收标准：`cargo test` 全绿且可重复执行，日志能按 request_id 串起一次请求，容器收到 SIGTERM 后正常退出。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 接口测试 | `Router::oneshot` + `Request::builder` |
| 异步测试 | `#[tokio::test]` |
| 状态注入 | 每个用例新建 `AppState` |
| 测试数据库 | 内存 SQLite 注意连接隔离 |
| 日志 | `tracing` + `EnvFilter` + `request_id` |
| 优雅关闭 | `with_graceful_shutdown` 处理 SIGTERM |

至此，Axum 这条线从框架定位、路由、提取器、状态、中间件、错误、数据库、鉴权与 WebSocket，一直走到测试与部署。下一步建议挑一个真实的小项目完整实现一遍：先定 API 契约，再写错误类型与数据层，最后补测试和部署脚本。把九篇里的片段连成一条能跑通的链路，比继续读更多概念更有价值。
