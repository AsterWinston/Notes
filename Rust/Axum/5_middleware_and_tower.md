Axum 的中间件不是框架自己发明的，而是直接复用 Tower 的 `Service` 和 `Layer`。理解了这两个 trait，日志、CORS、超时、鉴权就都是同一套拼装动作；不理解的代价是顺序写反、类型签名看不懂、错误处理接不上。本篇把 Tower 模型和 Axum 的挂载方式一次讲清。

# 1. Tower 模型

## 1.1 Service

Tower 的核心抽象是"一个可以调用、可能还没准备好的异步函数"：

```rust
use std::{
    future::Future,
    task::{Context, Poll},
};

pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // 服务是否已经准备好接收请求
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    // 真正处理请求
    fn call(&mut self, request: Request) -> Self::Future;
}
```

Axum 的 Router 本身就是一个 `Service<Request<Body>>`，错误类型是 `Infallible`。Handler 被包装成 Service，中间件也是 Service，层层套下去。

## 1.2 Layer

`Layer` 是"把一个 Service 包装成另一个 Service"的工厂：

```rust
pub trait Layer<S> {
    type Service;

    fn layer(&self, inner: S) -> Self::Service;
}
```

`Layer` 只负责创建包装层，不负责调用；调用发生在 `Service::call`。这个分离让中间件可以持有配置（超时时间、CORS 规则），而每次请求创建独立的 future。

## 1.3 Axum 与 Tower 的关系

| Axum 概念 | 对应的 Tower 概念 |
|-----------|-------------------|
| `Router` | `Service<Request<Body>>` |
| `Handler` | 被包装成 `Service` 的异步函数 |
| `.layer(...)` | 对 Router 应用 `Layer` |
| `.route_layer(...)` | 只对路由应用 `Layer` |
| `tower-http` 中间件 | 现成的 `Layer` / `Service` |

★ **关键认知**：Axum 没有自己的中间件协议，它只是"把 Tower 接到 HTTP 上"。所以任何 `tower` / `tower-http` 组件都能直接用；反过来，自己写中间件时也要遵守 Tower 的 `poll_ready` / `call` 契约，而不是随手写个函数。

# 2. 在 Axum 中挂中间件

## 2.1 layer 与 route_layer

```rust
use axum::{routing::get, Router};

let app = Router::new()
    .route("/", get(index))
    .route("/health", get(health))
    .layer(log_layer);        // 作用于所有路由 + fallback
```

```rust
let app = Router::new()
    .route("/admin", get(admin))
    .route_layer(auth_layer); // 只作用于已注册的路由
```

| API | 作用范围 | 备注 |
|-----|----------|------|
| `.layer(...)` | 路由 + fallback | 最常用 |
| `.route_layer(...)` | 仅路由 | Router 没有路由时会 panic |

★ **关键认知**：`.route_layer` 的价值在于"只有请求命中路由才执行"。比如鉴权中间件，如果放在 `.layer`，连 404 请求都会先过一遍鉴权，未登录用户访问不存在的路径会得到 401 而不是 404，语义就错了。

## 2.2 from_fn

最轻量的写法是把普通 `async fn` 转成中间件：

```rust
use axum::{
    extract::Request,
    middleware::Next,
    response::Response,
};

async fn log(request: Request, next: Next) -> Response {
    let method = request.method().clone();
    let uri = request.uri().clone();
    let start = std::time::Instant::now();

    let response = next.run(request).await;

    tracing::info!(
        %method,
        %uri,
        status = %response.status(),
        elapsed = ?start.elapsed(),
        "request finished"
    );
    response
}

let app = Router::new()
    .route("/", get(index))
    .layer(axum::middleware::from_fn(log));
```

参数顺序有讲究：前面的参数是提取器，最后两个固定是 `Request` 和 `Next`。`next.run(request)` 返回 `Response`，中间件可以在它前后加逻辑。

## 2.3 from_fn_with_state

需要共享状态时用 `from_fn_with_state`：

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response},
};

async fn auth(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Response {
    let ok = request
        .headers()
        .get("x-token")
        .and_then(|value| value.to_str().ok())
        .map(|token| token == state.token)
        .unwrap_or(false);

    if !ok {
        return (StatusCode::UNAUTHORIZED, "unauthorized").into_response();
    }

    request.extensions_mut().insert(AuthUser { id: "u1".into() });
    next.run(request).await
}

let app = Router::new()
    .route("/me", get(me))
    .layer(axum::middleware::from_fn_with_state(state.clone(), auth))
    .with_state(state);
```

★ **关键认知**：`from_fn_with_state` 的状态是在**创建 Layer 时**传入的，不会自动读取 Router 的 `with_state`。所以要先 `state.clone()` 给中间件，再把原状态交给 Router。⚠ 两边类型不一致时不会报"状态缺失"，而是编译错误指向泛型约束，排查时先看两处 `State<T>` 是不是同一个 `T`。

# 3. tower-http 常用中间件

## 3.1 TraceLayer

```toml
[dependencies]
tower-http = { version = "0.6", features = ["trace", "cors", "timeout", "compression-gzip"] }
```

```rust
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/", get(index))
    .layer(TraceLayer::new_for_http());
```

它会为每个请求生成 span，记录方法、URI、状态码、耗时。配合 `tracing-subscriber` 使用，不要同时再手写一份重复的日志中间件。

## 3.2 CorsLayer

```rust
use axum::http::{HeaderValue, Method};
use tower_http::cors::{Any, CorsLayer};

let cors = CorsLayer::new()
    .allow_origin("http://localhost:5173".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST])
    .allow_headers(Any);

let app = Router::new().route("/", get(index)).layer(cors);
```

★ **关键认知**：CORS 是浏览器行为，服务端只是返回响应头。⚠ `allow_origin(Any)` 不能和 `allow_credentials(true)` 同时用，浏览器会拒绝 `Access-Control-Allow-Origin: *` 加凭据的组合。生产环境要明确列出来源，不要图省事用 `CorsLayer::permissive()`。

## 3.3 TimeoutLayer

```rust
use std::time::Duration;
use tower_http::timeout::TimeoutLayer;

let app = Router::new()
    .route("/", get(index))
    .layer(TimeoutLayer::new(Duration::from_secs(10)));
```

超时后返回 **408 Request Timeout**。⚠ `tower::timeout::TimeoutLayer` 的行为不同，它产生的是 Service 错误，需要 `HandleErrorLayer` 转成响应；在 Axum 里优先用 `tower_http::timeout`。

## 3.4 其他常用组件

| 组件 | 作用 |
|------|------|
| `CompressionLayer` | gzip / br 压缩响应 |
| `CatchPanicLayer` | 捕获 Handler panic，转 500 |
| `RequestBodyLimitLayer` | 限制请求体大小 |
| `ServeDir` | 提供静态文件服务 |
| `SetRequestIdLayer` | 生成请求 ID |

这些组件都是 Layer / Service，挂载方式与前面一致。

# 4. 执行顺序

## 4.1 Router::layer 的顺序

`Router::layer` 每次都把当前 Router 包一层，所以**后加的在外层，先执行**：

```rust
let app = Router::new()
    .route("/", get(index))
    .layer(timeout) // 先加 → 内层
    .layer(cors)    // 后加 → 外层
    .layer(trace);  // 最后加 → 最外层
```

请求顺序是 `trace → cors → timeout → handler`，响应按相反方向返回。

## 4.2 ServiceBuilder 的顺序

`ServiceBuilder` 相反，**先加的在最外层**：

```rust
use tower::ServiceBuilder;

let app = Router::new()
    .route("/", get(index))
    .layer(
        ServiceBuilder::new()
            .layer(trace)   // 先加 → 最外层
            .layer(cors)
            .layer(timeout), // 后加 → 最内层
    );
```

两种写法表达的是同一件事，但方向相反。团队里最好只选一种，否则 review 时很容易看反。

## 4.3 请求与响应流向

```text
请求 → Trace（最外层）→ CORS → Timeout（最内层）→ Router → Handler
响应 ←────────────────────────────────────────────────────────────
```

★ **关键认知**：中间件是"洋葱"，请求从外向内，响应从内向外。⚠ 顺序直接影响正确性：CORS 放在 Timeout 内层，超时响应就不会带 CORS 头；鉴权放在日志外层，未授权请求就不会被日志记录。排顺序时先问"这一层需要看到什么、需要给什么加头"。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 中间件完全不生效 | 在 `.route()` 之前 `.layer()` | 先注册路由再挂层 |
| 404 也返回 401 | 鉴权用了 `.layer` | 改用 `.route_layer` |
| CORS 预检 404 | 没有处理 `OPTIONS` | 用 `CorsLayer`，不要手写 |
| 超时后没日志 | 日志层在超时层内层 | 把 Trace 放最外层 |
| `from_fn_with_state` 编译失败 | 状态类型与 Router 不一致 | 对齐 `AppState` |
| Layer 类型不满足 | 缺少 `tower` 依赖或 feature | 补依赖、开 feature |

⚠ 中间件报错经常表现为"路由 500 但 Handler 没执行"。先看 Layer 是否把错误吞掉，再看 `poll_ready` 是否一直返回 `Pending`。

# 6. 动手练习

1. 用 `from_fn` 写一个日志中间件，打印方法、URI、状态码和耗时，并验证它在 404 请求上也会执行。
2. 给 `/admin` 路由加一个基于 `X-Token` 的鉴权中间件，要求未命中路由时仍返回 404 而不是 401。
3. 组合 `TraceLayer`、`CorsLayer`、`TimeoutLayer`，用 `ServiceBuilder` 和链式 `.layer` 各写一遍，确认执行顺序一致。

验收标准：日志能看到完整请求链路，CORS 预检通过，超时响应带 CORS 头且被日志记录。

# 7. 小结

| 概念 | 要点 |
|------|------|
| `Service` | 可调用的异步处理单元，有 `poll_ready` 与 `call` |
| `Layer` | 把 Service 包装成新 Service 的工厂 |
| `Router` | 本身就是 `Service<Request<Body>>` |
| `.layer` | 作用于路由 + fallback |
| `.route_layer` | 只作用于路由，适合鉴权 |
| `from_fn` | 用普通函数写中间件 |
| `from_fn_with_state` | 带共享状态的中间件 |
| 顺序 | `Router::layer` 后加先执行，`ServiceBuilder` 先加先执行 |

下一篇进入错误处理，讲清楚 `IntoResponse` 如何统一错误出口、自定义错误类型怎么设计、状态码与业务码如何分离，以及 panic 该由谁兜底。
