Handler 是业务真正落地的地方，提取器决定"请求里的数据以什么形态进入 Handler"。Actix Web 的类型驱动提取器很舒服，但规则比表面严格：什么能提取、谁先执行、请求体能不能读第二次，都有明确边界。

# 1. Handler 的本质

## 1.1 一个 Handler 长什么样

```rust
use actix_web::{get, web, HttpResponse, Responder};

#[get("/hello/{name}")]
async fn hello(name: web::Path<String>) -> impl Responder {
    HttpResponse::Ok().body(format!("hello, {}", name))
}
```

```text
async fn handler(提取器参数...) -> 返回值
                 │                │
                 │                └── 必须实现 Responder
                 └── 每个参数都必须实现 FromRequest
```

★ **关键认知**：Actix Web 没有显式的"Handler trait 对象"。只要 `async fn` 的所有参数实现了 `FromRequest`、返回值实现了 `Responder`，它就是合法 Handler。写错时编译器会精确指出哪个参数不满足约束。

## 1.2 返回值可以是什么

| 返回类型 | 说明 |
|----------|------|
| `HttpResponse` | 完全控制状态码与响应头 |
| `impl Responder` | 编译器推断具体类型，写法灵活 |
| `String` / `&'static str` | 自动作为 `text/plain` |
| `web::Bytes` | 二进制响应体 |
| `Result<T, E>` | `T: Responder` 且 `E: Into<Error>` |
| `Option<T>` | `Some` 正常返回，`None` 自动 404 |
| `()` | 空响应体，状态 200 |

```rust
async fn a() -> &'static str { "plain text" }
async fn b() -> String { "owned".to_string() }
async fn c() -> Option<&'static str> { None } // 自动 404
async fn d() -> Result<HttpResponse, actix_web::Error> {
    Ok(HttpResponse::Ok().finish())
}
```

⚠ `impl Responder` 只适合单一返回类型。不同分支返回不同具体类型（如有时 `HttpResponse`、有时 `String`）时，编译器无法推断统一类型，要么统一转成 `HttpResponse`，要么用 `Either` 或 `Result` 包起来。

## 1.3 用 Result 写可失败逻辑

```rust
use actix_web::{error, web, HttpResponse, Result};

async fn divide(q: web::Query<std::collections::HashMap<String, f64>>) -> Result<HttpResponse> {
    let a = q.get("a").copied().unwrap_or(0.0);
    let b = q.get("b").copied().unwrap_or(1.0);

    if b == 0.0 {
        return Err(error::ErrorBadRequest("b 不能为 0")); // 直接变成 400
    }
    Ok(HttpResponse::Ok().json(serde_json::json!({ "result": a / b })))
}
```

`actix_web::Result<T>` 是 `Result<T, actix_web::Error>` 的别名。`?` 能把实现了 `ResponseError` 的错误自动转成 `actix_web::Error`，这是第 6 篇的主线。

# 2. 常用提取器

## 2.1 总览

| 提取器 | 数据来源 | 失败行为 |
|--------|----------|----------|
| `web::Path<T>` | 路径段 | 400 |
| `web::Query<T>` | 查询字符串 | 400 |
| `web::Json<T>` | JSON 请求体 | 400 |
| `web::Form<T>` | 表单请求体 | 400 |
| `web::Data<T>` | `app_data` 注册的共享状态 | 500（未注册） |
| `HttpRequest` | 整个请求 | 不会失败 |
| `web::Bytes` | 原始请求体 | 400 |

## 2.2 web::Json

```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize)]
struct UserView {
    id: i64,
    name: String,
    email: String,
}

#[actix_web::post("/users")]
async fn create_user(body: web::Json<CreateUser>) -> impl actix_web::Responder {
    let view = UserView {
        id: 1,
        name: body.name.clone(),
        email: body.email.clone(),
    };
    actix_web::HttpResponse::Created().json(view)
}
```

`web::Json<T>` 有两个硬性条件：`Content-Type` 必须是 `application/json`；请求体必须是合法且能反序列化成 `T` 的 JSON。任一不满足都返回 **400**，Handler 不执行。

★ **关键认知**：`web::Json` 不只是"解析 JSON"，它同时承担**内容协商校验**。客户端忘了带 `Content-Type: application/json` 时，框架不会猜，而是直接拒绝。这是 `curl` 调试失败的常见根因。

自定义 JSON 错误响应：

```rust
App::new().app_data(
    web::JsonConfig::default()
        .limit(1_048_576) // 限制 1MB，防止超大请求体
        .error_handler(|err, _req| {
            actix_web::error::InternalError::from_response(
                err,
                actix_web::HttpResponse::BadRequest().json(serde_json::json!({
                    "code": "INVALID_JSON",
                    "message": "请求体不是合法 JSON"
                })),
            )
            .into()
        }),
)
```

## 2.3 web::Form

```rust
#[derive(serde::Deserialize)]
struct LoginForm {
    username: String,
    password: String,
}

#[actix_web::post("/login")]
async fn login(form: web::Form<LoginForm>) -> impl actix_web::Responder {
    format!("login: {}", form.username)
}
```

`web::Form` 对应 `application/x-www-form-urlencoded`，浏览器原生表单提交就是这种格式。⚠ 它**不支持** `multipart/form-data`，文件上传需要 `actix-multipart`。

## 2.4 Path、Query 与 Data

```rust
#[actix_web::get("/users/{id}")]
async fn detail(id: web::Path<i64>) -> impl actix_web::Responder {
    format!("id = {}", id.into_inner()) // 也可以 *id 解引用
}

#[derive(serde::Deserialize)]
struct ListQuery {
    page: Option<u32>,
    size: Option<u32>,
}

#[actix_web::get("/users")]
async fn list(q: web::Query<ListQuery>) -> impl actix_web::Responder {
    format!("page={:?} size={:?}", q.page, q.size)
}

#[actix_web::get("/count")]
async fn count(state: web::Data<std::sync::atomic::AtomicUsize>) -> impl actix_web::Responder {
    format!("count = {}", state.load(std::sync::atomic::Ordering::Relaxed))
}
```

`web::Data` 是 `Arc` 的薄封装，未注册就提取会返回 **500**，启动期不会报错。

`HttpRequest` 不消费请求体，可与其他提取器同时出现；需要原始字节时用 `web::Bytes`。

# 3. 提取器的执行规则

## 3.1 按参数顺序执行

```rust
async fn handler(
    req: HttpRequest,        // 1. 先执行
    path: web::Path<i64>,    // 2. 再执行
    body: web::Json<CreateUser> // 3. 最后执行
) -> impl actix_web::Responder {
    format!("{} {:?} {:?}", req.path(), path.into_inner(), body.0)
}
```

★ **关键认知**：提取器按从左到右的顺序运行，任意一个失败就立即返回对应的 4xx/5xx，后面的提取器不再执行。把最便宜的校验放前面，可以避免无谓的 JSON 解析开销。

## 3.2 请求体只能读一次

请求体是**流**，只能被消费一次。`web::Json`、`web::Form`、`web::Bytes`、`web::Payload` 都属于消费型提取器，一个 Handler 里最多出现一个，并建议放在参数列表最后。

```text
请求 POST /users
Content-Type: text/plain
Body: {"name":"Tom"}
      │
      ▼
web::Json<CreateUser>
      │ Content-Type 不匹配
      ▼
400 Bad Request（Handler 一行都没执行）
```

⚠ 需要"先记录原始 body 再解析"时，应该用 `web::Bytes` 取原始字节，再手动 `serde_json::from_slice`，而不是放两个 `Json` 提取器。

# 4. 自定义提取器

## 4.1 实现 FromRequest

一个常见需求：从请求头取 API Key，校验后直接作为参数注入。

```rust
use actix_web::{dev::Payload, error, web, Error, FromRequest, HttpRequest};
use std::future::{ready, Ready};

pub struct ApiKey(pub String);

impl FromRequest for ApiKey {
    type Error = Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        match req.headers().get("X-API-KEY").and_then(|v| v.to_str().ok()) {
            Some(key) if key == "secret" => ready(Ok(ApiKey(key.to_owned()))),
            Some(_) => ready(Err(error::ErrorUnauthorized("API Key 无效"))),
            None => ready(Err(error::ErrorUnauthorized("缺少 API Key"))),
        }
    }
}
```

```rust
async fn protected(_key: ApiKey) -> impl actix_web::Responder { "authorized" }
```

★ **关键认知**：自定义提取器把"鉴权参数解析"变成了类型系统的一部分。函数签名里出现 `ApiKey`，读代码的人立刻知道这个接口需要鉴权，比在函数体里手写 header 检查更不容易漏。

如果提取过程需要 `.await`（查缓存、验签），把 `Future` 换成 `LocalBoxFuture`，在 `Box::pin(async move { ... })` 里写异步逻辑即可，其余 trait 形状不变。

# 5. 错误响应

```rust
use actix_web::{error, web, Result};

async fn check(id: web::Path<i64>) -> Result<String> {
    if *id < 0 {
        return Err(error::ErrorBadRequest("id 不能为负")); // 400
    }
    if *id == 0 {
        return Err(error::ErrorNotFound("用户不存在")); // 404
    }
    Ok(format!("ok {}", *id))
}
```

常用构造函数：`ErrorBadRequest`、`ErrorUnauthorized`、`ErrorForbidden`、`ErrorNotFound`、`ErrorConflict`、`ErrorInternalServerError`。

★ **关键认知**：错误响应建议统一为 `code` + `message` + 可选的 `request_id`。`code` 给前端做分支，`message` 给人看，`request_id` 用于和日志对齐。不要把 `Debug` 格式的内部错误直接返回，可能泄露 SQL、路径和堆栈。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 接口 400 但 Handler 没进 | `Content-Type` 不是 `application/json` | 请求头补上 |
| JSON 字段总是默认值 | 字段名与 JSON 不一致 | 加 `#[serde(rename = "...")]` |
| 两个 body 提取器编译失败 | 请求体只能消费一次 | 只保留一个，或先取 `Bytes` |
| `Data` 提取返回 500 | 没有 `app_data` 注册 | 启动时统一注册 |
| 自定义提取器不生效 | 没实现 `FromRequest` | 检查 trait 与参数类型 |

⚠ `serde` 默认忽略未知字段，但对缺失字段会报错（除非 `Option` 或 `#[serde(default)]`）。前后端字段对不上时，往往表现成 400。

# 7. 动手练习

1. 实现 `POST /echo`，接收 `web::Json<serde_json::Value>` 原样返回，并统计请求体字节数（提示：先取 `web::Bytes` 再手动解析）。
2. 写一个 `BearerToken` 提取器，从 `Authorization: Bearer xxx` 取出 token，缺失或格式错误返回 401。
3. 让 `POST /users` 在 `name` 为空时返回 422，并给出统一错误结构。

验收标准：错误响应都包含 `code` 与 `message`，且不会把内部错误原文暴露出去。

# 8. 小结

| 概念 | 要点 |
|------|------|
| Handler | 参数实现 `FromRequest`，返回值实现 `Responder` |
| `web::Json` | 要求 JSON 内容类型，失败 400 |
| `web::Form` | 只处理 urlencoded，不支持文件 |
| `web::Path`/`Query` | 类型驱动解析，失败即 400 |
| 提取顺序 | 从左到右，先失败先返回 |
| 请求体 | 只能消费一次，消费型提取器放最后 |
| 自定义提取器 | 实现 `FromRequest`，把校验变成类型 |

下一篇进入共享状态，讲清楚 `web::Data` 与 `Arc` 的关系、为什么 `HttpServer::new` 的闭包要反复 clone、`Mutex`/`RwLock` 在异步 Handler 里的正确用法，以及连接池的生命周期。
