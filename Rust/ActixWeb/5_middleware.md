中间件负责日志、鉴权、CORS、压缩这些横切关注点。Actix Web 的中间件模型比 Gin 的 `c.Next()` 更重：它建立在 `Transform` 和 `Service` 两个 trait 上，类型签名一开始会劝退不少人。抓住"洋葱"和"最后 wrap 的最先执行"这两点，剩下都是模板。

# 1. 中间件模型

## 1.1 洋葱模型

```text
请求
  │
  ▼
┌──────────────────┐
│ 最外层中间件      │  ← 最后 wrap 的
│  ┌────────────┐  │
│  │ 内层中间件  │  │
│  │  ┌──────┐  │  │
│  │  │Handler│  │  │
│  │  └──────┘  │  │
│  └────────────┘  │
└──────────────────┘
  │
  ▼
响应按相反方向穿出
```

进入时从外到内，返回时从内到外，所以同一个中间件既能做请求前逻辑，也能做响应后逻辑。

## 1.2 Transform 与 Service

| trait | 职责 | 调用时机 |
|-------|------|----------|
| `Transform` | 工厂，把内层 service 包成新 service | 初始化时一次 |
| `Service` | 实际的请求处理逻辑 | 每个请求一次 |

```text
Transform::new_transform(inner) -> Middleware(inner)
Middleware::call(req) -> 前置 -> inner.call(req).await -> 后置
```

★ **关键认知**：`Transform` 只在**初始化时调用一次**，`Service::call` 才是**每个请求调用一次**。把"每个请求都要重新算"的逻辑写进 `new_transform`，是常见的性能错误。

## 1.3 wrap 的顺序

```rust
App::new()
    .wrap(A)   // 先注册
    .wrap(B)   // 后注册
```

实际结构是 `B(A(handler))`，所以 **B 在最外层，先执行**。

⚠ **关键认知**：官方明确说 "if you use `wrap()` or `wrap_fn()` multiple times, the last occurrence will be executed first"。这和很多框架"先注册先执行"的直觉相反，调试时打一行日志最可靠。

层级关系：`App::wrap` 最外层，其次是 `scope::wrap`，再到 `resource::wrap`，最后是 Handler。

# 2. 执行顺序实战

用两个只打印进入/返回的 `wrap_fn` 注册，A 先注册、B 后注册：

```text
.wrap_fn(A)  → 内层
.wrap_fn(B)  → 外层（最后注册）

输出：
B 进入: /
A 进入: /
A 返回
B 返回
```

后注册的 B 先执行。

| 中间件 | 建议位置 | 原因 |
|--------|----------|------|
| 请求 ID | 最外层 | 后续日志和错误都要带上 |
| 日志 | 最外层或次外层 | 想记录 401/403 就必须在外层 |
| CORS | 最外层 | 预检请求要在鉴权之前处理 |
| 鉴权 | 内层 | 在请求 ID、日志之后 |

★ **关键认知**：因为"最后 wrap 的在最外层"，这张表翻译成代码时，**越靠外的越往后写**。想让 CORS 在最外层，就把它放在 `.wrap()` 链的最后。

# 3. 内置中间件

## 3.1 Logger

```rust
use actix_web::middleware::Logger;

env_logger::init_from_env(
    env_logger::Env::default().default_filter_or("info"),
);

App::new().wrap(Logger::new("%a %r %s %b %Dms \"%{User-Agent}i\""))
```

| 占位符 | 含义 |
|--------|------|
| `%a` | 客户端 IP |
| `%r` | 请求首行 |
| `%s` | 状态码 |
| `%D` | 耗时（毫秒） |

⚠ `Logger` 依赖 `log` crate，没初始化 `env_logger` 就什么都看不到。检查 `RUST_LOG=info` 是否生效。

## 3.2 其余常用中间件

```rust
use actix_web::middleware::{Compress, DefaultHeaders, NormalizePath};

App::new()
    .wrap(Compress::default())        // 按 Accept-Encoding 压缩
    .wrap(NormalizePath::trim())      // 去掉尾部斜杠再匹配
    .wrap(
        DefaultHeaders::new()         // 只在响应没有同名头时添加
            .add(("X-Content-Type-Options", "nosniff"))
            .add(("X-Frame-Options", "DENY")),
    )
```

`Compress` 做响应压缩，`NormalizePath::trim()` 统一尾部斜杠（会改写 URI），`DefaultHeaders` 添加统一安全头，`ErrorHandlers` 用来改写框架默认错误响应。

# 4. 自定义中间件

## 4.1 完整 Transform + Service 模板

下面实现记录耗时的中间件：

```rust
use std::future::{ready, Ready};
use std::time::Instant;

use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::LocalBoxFuture;

pub struct Timing;

impl<S, B> Transform<S, ServiceRequest> for Timing
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = TimingMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(TimingMiddleware { service })) // 初始化：只包一层
    }
}

pub struct TimingMiddleware<S> { service: S }

impl<S, B> Service<ServiceRequest> for TimingMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service); // 把 ready 状态转发给内层

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let start = Instant::now();
        let fut = self.service.call(req);

        Box::pin(async move {
            let res = fut.await?;
            log::info!("{} 耗时 {:?}", res.request().path(), start.elapsed());
            Ok(res)
        })
    }
}
```

注册 `App::new().wrap(Timing)`，并添加 `futures-util = "0.3"` 依赖。

★ **关键认知**：模板里的 `S`、`B`、`LocalBoxFuture` 不是装饰，而是"中间件对任意内层 service 和响应体类型都成立"的泛型约束。照抄模板、只改 `call` 里的前后置逻辑，是上手最快的方式。

## 4.2 wrap_fn 与 from_fn

只做简单逻辑时，不必写两个 struct：`wrap_fn(|req, srv| { ... })` 直接拿到 `ServiceRequest` 和内层 service，适合原型；Actix Web 4 还提供 `middleware::from_fn`，用 `async fn(ServiceRequest, Next<impl MessageBody>) -> Result<ServiceResponse<impl MessageBody>, Error>` 的形状表达，可读性更好，推荐作为简单中间件的首选。两者注册时同样遵守"最后 wrap 的最先执行"。

# 5. 鉴权中间件

```rust
use actix_web::{
    body::MessageBody,
    dev::{ServiceRequest, ServiceResponse},
    error::ErrorUnauthorized,
    middleware::{from_fn, Next},
    Error, HttpRequest,
};

#[derive(Clone, Copy)]
pub struct UserId(pub i64);

async fn auth(
    req: ServiceRequest,
    next: Next<impl MessageBody>,
) -> Result<ServiceResponse<impl MessageBody>, Error> {
    let token = req.headers().get("Authorization").and_then(|v| v.to_str().ok());

    match token {
        Some(t) if t.starts_with("Bearer ") => {
            let user_id = parse_token(&t[7..])?; // 伪代码：验签
            req.extensions_mut().insert(UserId(user_id));
            next.call(req).await
        }
        Some(_) => Err(ErrorUnauthorized("令牌格式错误")),
        None => Err(ErrorUnauthorized("缺少令牌")),
    }
}

fn parse_token(token: &str) -> Result<i64, Error> {
    if token.is_empty() { return Err(ErrorUnauthorized("令牌为空")); }
    Ok(1) // 示例：真实实现见鉴权篇
}

async fn me(req: HttpRequest) -> impl actix_web::Responder {
    match req.extensions().get::<UserId>() {
        Some(uid) => format!("user = {}", uid.0),
        None => "anonymous".to_string(),
    }
}
```

注册时注意顺序：`.wrap(from_fn(auth))` 要在请求 ID 之后（也就是写在它前面）；只想保护部分路由时，把 `auth` 挂到对应 `scope`，不要全局挂。不想继续往内走时，用 `return Ok(req.into_response(HttpResponse::Forbidden().body("blocked")))` 直接返回响应，别再调用 `next`。

★ **关键认知**：`return Ok(req.into_response(...))` 是"不再往内走"的正确写法；直接 `Err(...)` 也可以，但会走错误处理链。前者是"正常返回响应"，后者是"产生一个错误"。

# 6. CORS

```toml
[dependencies]
actix-cors = "0.7"
```

开发环境图省事，生产环境收紧：

```rust
use actix_cors::Cors;
use actix_web::http::header;

// 生产：明确来源、方法、头
let cors = Cors::default()
    .allowed_origin("https://example.com")
    .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
    .allowed_headers(vec![header::AUTHORIZATION, header::CONTENT_TYPE])
    .max_age(3600);

App::new().wrap(cors)
```

★ **关键认知**：CORS 必须放在**鉴权外层**，否则浏览器的 `OPTIONS` 预检请求还没带鉴权头就被 401 拦下，前端会看到莫名其妙的跨域错误。由于"最后 wrap 的最先执行"，把 `Cors` 写在整个 `.wrap()` 链的最后。

⚠ `Cors::permissive()` 允许任意来源，只能本地开发用。需要携带 Cookie 时还要 `.supports_credentials()`，且不能与 `allowed_origin("*")` 同时使用。

# 7. 常见坑

| 问题 | 后果 | 正确做法 |
|------|------|----------|
| 以为先 wrap 先执行 | 日志/鉴权顺序全错 | 记住最后 wrap 的最外层 |
| CORS 放在鉴权内层 | 预检请求 401 | CORS 放最外层 |
| 在 `new_transform` 里做业务 | 逻辑只执行一次 | 业务放 `call` |
| 重复消费 body | 后续提取器失败 | 中间件只读 header/扩展 |
| 中间件里阻塞 | 整个 worker 卡住 | 慢操作异步化或 `web::block` |
| 忘记初始化日志 | `Logger` 无输出 | 初始化 `env_logger` |

⚠ 中间件是**每个 App、每个 scope、每个 resource 分别注册**的。给一个 `scope` 挂了中间件，不会自动作用到另一个 `scope`；`HttpServer` 层没有"全局中间件"这个概念。

# 8. 动手练习

1. 写请求 ID 中间件：优先读 `X-Request-ID`，没有就生成一个，写入 `extensions` 和响应头。
2. 写鉴权中间件，只作用于 `/api/admin` scope，未登录返回 401。
3. 组合 `Logger`、CORS、请求 ID、鉴权，画出执行顺序并用日志验证。

验收标准：预检请求不返回 401，401 响应里带请求 ID，日志顺序与设计一致。

# 9. 小结

| 概念 | 要点 |
|------|------|
| 洋葱模型 | 进入由外到内，返回由内到外 |
| `Transform` | 工厂，初始化时调用一次 |
| `Service` | 每请求调用，前后置逻辑写这里 |
| wrap 顺序 | 最后注册的最先执行 |
| 层级 | App → scope → resource |
| 轻量中间件 | `wrap_fn`、`from_fn` |
| CORS | 放最外层，生产禁用 permissive |
| 鉴权 | 校验后写入 `extensions`，短路用 `into_response` |

下一篇进入错误处理，讲清楚 `ResponseError` trait、如何把数据库错误和校验错误统一成一种响应结构、`?` 的自动转换，以及 panic 与 `ErrorHandlers` 的边界。
