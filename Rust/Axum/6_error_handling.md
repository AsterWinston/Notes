错误处理是 Web 服务的骨架：正常路径只占一小部分，剩下全是"参数不对、没登录、资源不存在、数据库挂了"。Axum 把错误出口统一到 `IntoResponse`，只要自定义错误类型实现了它，Handler 就可以直接返回 `Result<T, AppError>`，不需要到处手写状态码。

# 1. IntoResponse 是错误处理的基石

## 1.1 什么是 IntoResponse

`IntoResponse` 是"任何能变成 HTTP 响应的东西"：

```rust
pub trait IntoResponse {
    fn into_response(self) -> Response;
}
```

Axum 为大量类型实现了它：

| 类型 | 变成什么 |
|------|----------|
| `&'static str` / `String` | 200 + 文本 |
| `StatusCode` | 只有状态码 |
| `Json<T>` | 200 + JSON |
| `(StatusCode, T)` | 指定状态码 + 响应体 |
| `(StatusCode, HeaderMap, T)` | 再加响应头 |
| `Result<T, E>` | `Ok` / `Err` 分别转换 |

★ **关键认知**：`Result<T, E>` 本身也实现 `IntoResponse`，前提是 `T` 和 `E` 都实现。所以 Handler 返回 `Result<Json<User>, AppError>` 是惯用写法：成功走 `Json`，失败走 `AppError`，框架自动二选一。⚠ 如果 `E` 没实现 `IntoResponse`，报错会指向 `Result` 的 trait bound，而不是直接说"你的错误类型缺实现"。

## 1.2 元组与状态码

```rust
use axum::{
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use serde_json::json;

async fn create() -> impl IntoResponse {
    (
        StatusCode::CREATED,
        Json(json!({ "id": 1, "name": "Tom" })),
    )
}
```

元组是最轻量的"状态码 + 响应体"组合。需要加响应头时用三元组 `(StatusCode, HeaderMap, body)`。

# 2. 自定义错误类型

## 2.1 枚举错误

```rust
#[derive(Debug)]
pub enum AppError {
    NotFound,
    Unauthorized,
    BadRequest(String),
    Conflict(String),
    ServiceUnavailable,
    Database(sqlx::Error),
}
```

用枚举而不是 `Box<dyn Error>` 的好处是：错误分支显式、状态码映射集中、编译器会提醒你处理新增分支。

## 2.2 实现 IntoResponse

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code, message) = match self {
            AppError::NotFound => (
                StatusCode::NOT_FOUND,
                "NOT_FOUND",
                "资源不存在".to_string(),
            ),
            AppError::Unauthorized => (
                StatusCode::UNAUTHORIZED,
                "UNAUTHORIZED",
                "未认证".to_string(),
            ),
            AppError::BadRequest(message) => {
                (StatusCode::BAD_REQUEST, "BAD_REQUEST", message)
            }
            AppError::Conflict(message) => {
                (StatusCode::CONFLICT, "CONFLICT", message)
            }
            AppError::ServiceUnavailable => (
                StatusCode::SERVICE_UNAVAILABLE,
                "SERVICE_UNAVAILABLE",
                "服务暂时不可用".to_string(),
            ),
            AppError::Database(error) => {
                // 数据库细节只进日志，不返回给客户端
                tracing::error!(error = %error, "数据库错误");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "INTERNAL",
                    "服务器内部错误".to_string(),
                )
            }
        };

        (status, Json(json!({ "code": code, "message": message }))).into_response()
    }
}
```

Handler 直接返回：

```rust
async fn get_user(Path(id): Path<i64>) -> Result<Json<Value>, AppError> {
    if id == 0 {
        return Err(AppError::NotFound);
    }
    Ok(Json(json!({ "id": id })))
}
```

★ **关键认知**：`IntoResponse` 是错误处理的唯一出口。把"状态码、业务码、日志、脱敏"全部收敛在 `into_response` 里，Handler 就只需要表达"发生了什么错误"，不需要关心怎么呈现。⚠ 不要在 Handler 里同时 `tracing::error!` 又在 `IntoResponse` 里打一遍，否则同一个错误会产生两条日志。

## 2.3 From 转换

有了 `From`，就能在 Handler 里用 `?`：

```rust
use axum::extract::rejection::JsonRejection;

impl From<sqlx::Error> for AppError {
    fn from(error: sqlx::Error) -> Self {
        match error {
            sqlx::Error::RowNotFound => AppError::NotFound,
            other => AppError::Database(other),
        }
    }
}

impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        AppError::BadRequest(rejection.body_text())
    }
}
```

```rust
use axum::extract::{rejection::JsonRejection, Json};

async fn create(
    payload: Result<Json<CreateUser>, JsonRejection>,
) -> Result<StatusCode, AppError> {
    let Json(_payload) = payload?; // 失败时 ? 自动调用 From<JsonRejection>
    Ok(StatusCode::CREATED)
}
```

⚠ `From<sqlx::Error>` 不要把所有数据库错误都映射成 500。`RowNotFound` 是 404，唯一约束冲突是 409，连接池超时才是 503/500。映射越细，客户端越能做出正确反应。

# 3. 统一错误结构

## 3.1 错误码设计

推荐三层：HTTP 状态码表达"哪一类"，业务码表达"具体是什么"，message 给人看。

```json
{
  "code": "USER_NOT_FOUND",
  "message": "用户不存在",
  "request_id": "8f14e45f"
}
```

| 层 | 示例 | 给谁看 |
|----|------|--------|
| HTTP 状态码 | 404 | 网关、监控、客户端框架 |
| 业务码 | `USER_NOT_FOUND` | 客户端逻辑 |
| message | 用户不存在 | 人、日志 |
| request_id | `8f14e45f` | 排查链路 |

★ **关键认知**：HTTP 状态码不要全部用 200。把业务码塞进 200 响应体，会让网关、重试策略、监控全部失效；反过来，也不要为每个业务错误发明一个新的 HTTP 状态码，标准状态码只有那几十个。

## 3.2 生产与开发环境

| 内容 | 开发 | 生产 |
|------|------|------|
| 错误细节 | 可以返回 | 只进日志 |
| 数据库错误 | 打印完整信息 | 返回通用文案 |
| 堆栈 | 可选 | 绝不返回 |
| request_id | 可选 | 必须带 |

```rust
let message = if cfg!(debug_assertions) {
    format!("数据库错误: {error}")
} else {
    "服务器内部错误".to_string()
};
```

⚠ 即使在开发环境，也不要把密码、token、连接串打进响应体；日志同样要脱敏。

# 4. panic 与恢复

## 4.1 CatchPanicLayer

```rust
use tower_http::catch_panic::CatchPanicLayer;

let app = Router::new()
    .route("/", get(index))
    .layer(CatchPanicLayer::new());
```

```toml
tower-http = { version = "0.6", features = ["catch-panic"] }
```

它捕获 Handler 及内层 Service 的 panic，转成 500，避免整个连接任务被 unwind 掉。

## 4.2 panic 的边界

★ **关键认知**：panic 只适合"程序 bug"，不适合"预期内的失败"。用户输入不合法、资源不存在、下游超时，都应该用 `Result` 表达。⚠ `CatchPanicLayer` 不是万能的：

| 场景 | 能否捕获 |
|------|----------|
| Handler 内 panic | 能 |
| 提取器内 panic | 能（在 Service 调用链上） |
| 后台 `tokio::spawn` 任务 panic | 不能，需自己 await `JoinHandle` |
| `panic = "abort"` 构建 | 不能，进程直接退出 |

发布构建如果设置了 `panic = "abort"`，捕获层形同虚设。需要兜底就保持默认的 unwind 策略，并配置 panic hook 记录日志。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| Handler 返回 `Result` 编译失败 | 错误类型没实现 `IntoResponse` | 给 `AppError` 补实现 |
| 所有错误都是 500 | `From` 里统一映射成内部错误 | 按变体细分状态码 |
| 客户端拿不到业务码 | 只返回了纯文本 | 统一 JSON 结构 |
| 错误日志重复 | Handler 与 `IntoResponse` 都打日志 | 只在一处记录 |
| panic 后连接断开 | 没挂 `CatchPanicLayer` | 加捕获层并检查 panic 策略 |
| 生产泄露 SQL | 直接把 `sqlx::Error` 序列化 | 细节进日志，响应脱敏 |

⚠ `AppError` 里如果直接 `#[derive(Serialize)]` 并返回，容易把内部字段（表名、SQL、路径）带出去。错误响应应该是**手工构造的白名单结构**，不是错误类型的自动序列化。

# 6. 动手练习

1. 定义 `AppError`，至少包含 `NotFound`、`BadRequest`、`Unauthorized`、`Database` 四个变体，并实现统一的 JSON 响应。
2. 给 `From<sqlx::Error>` 加上 `RowNotFound → 404`、唯一约束 → 409 的映射，用一个重复插入的接口验证。
3. 挂上 `CatchPanicLayer`，写一个会 panic 的调试接口，确认客户端收到 500 且进程不退出。

验收标准：四类错误返回正确的状态码与业务码，日志里能看到数据库细节，响应体里看不到。

# 7. 小结

| 概念 | 要点 |
|------|------|
| `IntoResponse` | 所有响应的统一出口 |
| `Result<T, E>` | 成功与失败都实现 `IntoResponse` 即可直接返回 |
| 自定义错误 | 枚举 + 集中映射状态码与业务码 |
| `From` | 让 `?` 自动转换，Handler 更干净 |
| 错误结构 | HTTP 状态码 + 业务码 + message + request_id |
| 脱敏 | 数据库细节进日志，响应只给通用文案 |
| panic | 只用于程序 bug，用 `CatchPanicLayer` 兜底 |
| 日志 | 一个错误只记录一次 |

下一篇进入数据库集成，讲清楚 SQLx 连接池怎么建、`query_as` 与编译期检查怎么选、事务的提交与回滚边界，以及内存 SQLite 做测试时最容易踩的连接隔离问题。
