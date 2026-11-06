提取器是 Axum 最有辨识度的设计：Handler 的参数不是手动从请求里抠出来的，而是声明类型，框架负责解析、校验、失败时返回合适的错误。理解"哪些提取器读请求行、哪些读请求体"，就理解了 Axum 一半的用法。

# 1. 提取器是什么

## 1.1 两类 trait

Axum 的提取器分成两族，区别只有一个：**是否消耗请求体**。

| trait | 读取内容 | 是否消耗 body | 典型类型 |
|-------|----------|---------------|----------|
| `FromRequestParts` | 方法、URI、头、扩展 | 否 | `Path`、`Query`、`State`、`HeaderMap` |
| `FromRequest` | 整个请求，包括 body | 是 | `Json`、`Form`、`Bytes`、`String` |

两者的签名只差一个方法名：`FromRequestParts::from_request_parts(&mut Parts, &S)` 拿到的是请求的"前半部分"，`FromRequest::from_request(Request, &S)` 拿到的是整个请求。前者可以重复调用，后者会把 body 的所有权拿走。

★ **关键认知**：请求体在 HTTP 里是一条单向流，读完就没了。所以 `FromRequest` 的提取器**最多出现一次，且必须放在参数列表最后**。这不是框架的偏好，而是"流只能消费一次"的物理约束。

## 1.2 顺序规则

```rust
use axum::{
    extract::{Path, Query, State},
    Json,
};

async fn ok(
    State(state): State<AppState>,
    Path(id): Path<i64>,
    Query(q): Query<PageQuery>,
    Json(body): Json<CreateUser>, // 读 body 的放最后
) {
    // 这里写业务逻辑
}
```

反过来写会直接编译失败：

```rust
async fn bad(
    Json(body): Json<CreateUser>, // 已经拿走了整个请求
    Path(id): Path<i64>,          // ✗ 编译错误：Json 不是 FromRequestParts
) {
    // 不会执行到这里
}
```

⚠ 报错通常长这样：`the trait bound Json<CreateUser>: FromRequestParts<_> is not satisfied`。看到 `FromRequestParts` 就往回看参数顺序，而不是去改类型。

# 2. 常用提取器

## 2.1 Path

```rust
use axum::extract::Path;

async fn get_user(Path(id): Path<i64>) -> String {
    format!("user {id}")
}

let app = Router::new().route("/users/{id}", get(get_user));
```

`Path<T>` 支持单值、元组、结构体和 `HashMap<String, String>`。解析失败返回 400，Handler 不执行。

## 2.2 Query

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct PageQuery {
    page: Option<u32>,
    size: Option<u32>,
}

async fn list(Query(q): Query<PageQuery>) -> String {
    let page = q.page.unwrap_or(1);
    let size = q.size.unwrap_or(20).clamp(1, 100); // 顺手做上限保护
    format!("page={page} size={size}")
}
```

★ **关键认知**：`Query<T>` 依赖 `serde_urlencoded`，**非 `Option` 字段缺失时直接返回 400**。可选参数必须显式写成 `Option<T>`，不能指望 `#[serde(default)]` 之外的东西兜底；`#[serde(default)]` 只在该字段完全缺失时生效，类型错误仍然 400。

## 2.3 Json

```rust
use axum::Json;
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

async fn create(Json(payload): Json<CreateUser>) -> Json<CreateUser> {
    Json(payload)
}
```

`Json<T>` 要求请求头是 `Content-Type: application/json`，缺了返回 **415**，JSON 语法错误返回 **400**，字段类型不匹配返回 **422**。⚠ 用 `curl -d '{...}'` 时默认是表单类型，必须显式加 `-H "Content-Type: application/json"`。

## 2.4 Form

```rust
use axum::extract::Form;

#[derive(Deserialize)]
struct LoginForm {
    username: String,
    password: String,
}

async fn login(Form(form): Form<LoginForm>) -> String {
    format!("login as {}", form.username)
}
```

`Form<T>` 解析 `application/x-www-form-urlencoded`。⚠ 它**不支持 multipart**；文件上传要用 `axum::extract::Multipart`，并且要注意字段顺序和大小限制。

## 2.5 State

```rust
use axum::extract::State;

async fn info(State(state): State<AppState>) -> String {
    state.app_name.clone()
}
```

`State<T>` 是编译期注入的应用状态，永远不会解析失败。它只要求 `T: Clone + Send + Sync + 'static`，以及 Router 调用过 `with_state`。

# 3. 拒绝响应

## 3.1 默认拒绝

| 提取器 | 失败原因 | 默认状态码 |
|--------|----------|------------|
| `Path` | 类型转换失败 | 400 |
| `Query` | 反序列化失败 | 400 |
| `Json` | 缺少 JSON Content-Type | 415 |
| `Json` | JSON 语法错误 | 400 |
| `Json` | 字段类型不匹配 | 422 |
| `Form` | Content-Type 不对 | 415 |
| `Form` | 字段解析失败 | 422 |
| `Extension` | 扩展不存在 | 500 |

默认响应体是纯文本，和业务统一的 JSON 错误结构不一致。要么在网关层统一包装，要么在 Handler 里捕获 rejection。

## 3.2 捕获 rejection

```rust
use axum::{
    extract::{rejection::JsonRejection, Json},
    http::StatusCode,
    response::IntoResponse,
};

async fn create(payload: Result<Json<CreateUser>, JsonRejection>) -> impl IntoResponse {
    match payload {
        Ok(Json(user)) => (StatusCode::CREATED, Json(user)).into_response(),
        Err(rejection) => (rejection.status(), rejection.body_text()).into_response(),
    }
}
```

把提取器包进 `Result<T, Rejection>`，Handler 就能自己决定错误格式。⚠ 不要把所有 rejection 都改成 400，`status()` 里已经区分了 415、422，丢掉它会让客户端无法判断该改请求头还是改字段。

## 3.3 WithRejection

```rust
use axum::extract::{rejection::JsonRejection, WithRejection};

async fn create(
    WithRejection(Json(payload), _): WithRejection<Json<CreateUser>, AppError>,
) -> Result<StatusCode, AppError> {
    // payload 已解析成功；失败会自动转成 AppError
    Ok(StatusCode::CREATED)
}
```

只要给 `AppError` 实现 `From<JsonRejection>`，`WithRejection` 就能把提取失败接进统一错误体系，比每个 Handler 手写 `match` 干净。

# 4. 自定义提取器

## 4.1 从请求头提取 API Key

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

struct ApiKey(String);

impl<S> FromRequestParts<S> for ApiKey
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let key = parts
            .headers
            .get("x-api-key")
            .and_then(|value| value.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "缺少 X-API-Key"))?;
        Ok(ApiKey(key.to_string()))
    }
}
```

```rust
async fn protected(_key: ApiKey) -> &'static str {
    "ok"
}
```

★ **关键认知**：0.8 的自定义提取器不需要 `#[async_trait]`，直接写 `async fn`。⚠ 提取器里不要做重 I/O：它运行在请求处理的关键路径上，慢查询会把整个连接拖住。

## 4.2 依赖其他提取器

提取器可以复用别的 `FromRequestParts`：

```rust
#[derive(Clone)]
struct Claims {
    sub: String,
}

struct CurrentUser {
    id: String,
}

impl<S> FromRequestParts<S> for CurrentUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // 假设鉴权中间件已经把 Claims 写进 extensions
        parts
            .extensions
            .get::<Claims>()
            .map(|claims| CurrentUser { id: claims.sub.clone() })
            .ok_or((StatusCode::UNAUTHORIZED, "未登录"))
    }
}
```

这种写法把"从哪拿身份"收敛到一个类型里，Handler 只声明 `CurrentUser`，不关心中间件细节。

# 5. 常见错误

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| 编译报 `FromRequestParts` 不满足 | 读 body 的提取器不在最后 | 调整参数顺序 |
| JSON 接口一直 415 | 请求头不是 `application/json` | 检查 `Content-Type` |
| 可选查询参数 400 | 字段没写 `Option` | 改成 `Option<T>` |
| 路径参数 400 | 用了 `:id` 或类型不匹配 | 检查路由与 `Path<T>` |
| 自定义提取器 500 | 直接 panic 而不是返回 `Rejection` | 用 `Result` 表达失败 |
| Form 拿不到文件 | `Form` 不支持 multipart | 改用 `Multipart` |

⚠ 提取器的错误发生在 Handler 之前，业务日志里往往只看到"请求没进来"。排查 400/415 时先看提取器，不要一头扎进业务代码。

# 6. 动手练习

1. 实现 `GET /users?page=2&size=50`，缺省值为 1 和 20，`size` 上限 100，非数字返回 400。
2. 实现 `POST /users`，要求 JSON 请求体，手动捕获 `JsonRejection` 并返回统一的 `{"code","message"}`。
3. 写一个 `ApiKey` 提取器，从 `X-API-Key` 读值，缺失返回 401，并在一个受保护接口上使用它。

验收标准：四种失败场景（缺 Content-Type、坏 JSON、字段类型错、缺 API Key）都返回预期状态码与 JSON 结构。

# 7. 小结

| 概念 | 要点 |
|------|------|
| `FromRequestParts` | 只读请求头与 URI，可多次调用 |
| `FromRequest` | 消耗 body，只能有一个且必须最后 |
| `Path` / `Query` | 解析失败默认 400 |
| `Json` | 缺 Content-Type 415，语法 400，类型 422 |
| `Form` | 只支持 urlencoded，不支持 multipart |
| `State` | 编译期注入，不会解析失败 |
| rejection | 用 `Result<T, Rejection>` 或 `WithRejection` 接管 |
| 自定义提取器 | 0.8 直接写 `async fn`，不需要宏 |

下一篇进入状态与共享数据，讲清楚 `State` 为什么要求 `Clone + Send + Sync`、`Arc` 与连接池怎么放、`Mutex` 与 `RwLock` 在异步里有哪些边界，以及跨 `.await` 持锁为什么会编译失败。
