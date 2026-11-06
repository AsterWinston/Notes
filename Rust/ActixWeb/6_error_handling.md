错误处理是区分 demo 和可维护服务的关键。Actix Web 的错误体系围绕 `ResponseError` 展开：只要错误类型实现了它，就能用 `?` 直接向上抛，并由框架转成 HTTP 响应。难点不在语法，而在"哪些信息给用户、哪些只进日志"。

# 1. Actix 的错误模型

## 1.1 actix_web::Error 是什么

`actix_web::Error` 是一个**类型擦除的错误容器**，内部持有 `Box<dyn ResponseError>`。它让 Handler 可以用统一的 `Result<T, Error>` 返回，并在框架边界把错误转成 `HttpResponse`。

```rust
use actix_web::{error, Result};

async fn demo() -> Result<&'static str> {
    Err(error::ErrorBadRequest("参数错误")) // 直接变成 400
}
```

## 1.2 没有处理会怎样

```rust
async fn boom() -> &'static str {
    panic!("something wrong");
}
```

Handler 里的 panic 会被 worker 捕获，记录日志并返回 **500**，但响应体是框架默认的，不受你控制。

★ **关键认知**：Actix Web 把"可预期错误"和"程序 bug"分开处理。可预期错误走 `Result` + `ResponseError`，由你决定状态码和响应体；`panic` 只留给真正的 bug，不要指望它被业务错误格式化逻辑接住。

## 1.3 `?` 为什么能用

`?` 做了两步：先把错误通过 `From` 转成 `actix_web::Error`，再提前返回。Actix 为所有 `ResponseError + 'static` 类型提供了这个转换。

★ **关键认知**：只要错误类型实现 `ResponseError`，就能直接参与 `?` 传播，不需要手写 `map_err`。这是统一错误类型设计的核心收益。

# 2. ResponseError trait

## 2.1 必须实现什么

```rust
pub trait ResponseError: fmt::Debug + fmt::Display {
    fn status_code(&self) -> StatusCode;

    fn error_response(&self) -> HttpResponse {
        // 默认实现：状态码 + Display 文本
    }
}
```

两个硬性要求：`Debug` 供框架日志使用，`Display` 供默认响应体使用。通常用 `thiserror` 派生这两个，然后只实现 `status_code` 和 `error_response` 两个方法。

# 3. 统一错误类型

## 3.1 用 thiserror 定义

```toml
[dependencies]
thiserror = "2"
anyhow = "1"
```

```rust
use actix_web::{http::StatusCode, HttpResponse, ResponseError};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("资源不存在: {0}")]
    NotFound(String),

    #[error("请求参数错误: {0}")]
    BadRequest(String),

    #[error("未认证")]
    Unauthorized,

    #[error("无权访问")]
    Forbidden,

    #[error("资源冲突: {0}")]
    Conflict(String),

    #[error("校验失败: {0}")]
    Validation(String),

    #[error("数据库错误: {0}")]
    Database(#[from] sqlx::Error),

    #[error("内部错误: {0}")]
    Internal(#[from] anyhow::Error),
}
```

`#[from]` 自动生成 `From<sqlx::Error>` 和 `From<anyhow::Error>`，于是 `?` 可以直接把底层错误转成 `AppError`。

## 3.2 实现 ResponseError

```rust
impl ResponseError for AppError {
    fn status_code(&self) -> StatusCode {
        match self {
            AppError::NotFound(_) => StatusCode::NOT_FOUND,
            AppError::BadRequest(_) => StatusCode::BAD_REQUEST,
            AppError::Unauthorized => StatusCode::UNAUTHORIZED,
            AppError::Forbidden => StatusCode::FORBIDDEN,
            AppError::Conflict(_) => StatusCode::CONFLICT,
            AppError::Validation(_) => StatusCode::UNPROCESSABLE_ENTITY,
            AppError::Database(_) | AppError::Internal(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }

    fn error_response(&self) -> HttpResponse {
        let (code, message) = match self {
            AppError::NotFound(m) => ("NOT_FOUND", m.clone()),
            AppError::BadRequest(m) => ("BAD_REQUEST", m.clone()),
            AppError::Unauthorized => ("UNAUTHORIZED", "请先登录".to_string()),
            AppError::Forbidden => ("FORBIDDEN", "无权访问".to_string()),
            AppError::Conflict(m) => ("CONFLICT", m.clone()),
            AppError::Validation(m) => ("VALIDATION_ERROR", m.clone()),
            // 内部错误不把细节返回给客户端
            AppError::Database(_) | AppError::Internal(_) => {
                ("INTERNAL_ERROR", "服务内部错误".to_string())
            }
        };

        if matches!(self.status_code(), StatusCode::INTERNAL_SERVER_ERROR) {
            log::error!("internal error: {:?}", self); // 完整信息只进日志
        } else {
            log::warn!("client error: {}", self);
        }

        HttpResponse::build(self.status_code()).json(serde_json::json!({
            "code": code,
            "message": message
        }))
    }
}
```

★ **关键认知**：`error_response` 是"用户可见"和"日志可见"的分界线。`Display` 可以写详细原因供排查，但响应体里对 5xx 必须换成通用文案。⚠ 直接把 `sqlx::Error` 的 `Display` 返回给前端，可能泄露表名、SQL 片段和数据库地址。

## 3.3 Handler 里直接返回

```rust
use actix_web::{web, Result};

async fn get_user(
    id: web::Path<i64>,
    state: web::Data<AppState>,
) -> Result<web::Json<UserView>> {
    let user = sqlx::query_as::<_, UserView>("SELECT id, name FROM users WHERE id = ?")
        .bind(id.into_inner())
        .fetch_optional(&state.db)
        .await? // sqlx::Error → AppError::Database
        .ok_or_else(|| AppError::NotFound("用户不存在".into()))?; // 自定义 404

    Ok(web::Json(user))
}
```

`Result<T, AppError>` 能直接作为 Handler 返回值，因为 `AppError: ResponseError`，而 `Result<T, E>` 实现了 `Responder`。

# 4. 状态码怎么选

| 场景 | 状态码 | 错误变体 |
|------|--------|----------|
| 参数格式错误 | 400 | `BadRequest` |
| 未登录 / 令牌无效 | 401 | `Unauthorized` |
| 已登录但无权限 | 403 | `Forbidden` |
| 资源不存在 | 404 | `NotFound` |
| 唯一键冲突 / 状态冲突 | 409 | `Conflict` |
| 语义校验失败 | 422 | `Validation` |
| 未预期内部错误 | 500 | `Internal` / `Database` |
| 依赖服务不可用 | 503 | 单独变体 |

400 与 422 的区别容易被忽略：400 更偏"请求本身不合法"（JSON 解析失败、类型不对），422 更偏"格式合法但业务校验不过"。团队内统一即可，但不要同一类问题一会儿 400 一会儿 422。

# 5. 错误转换

```rust
async fn create(
    body: web::Json<CreateUser>,
    state: web::Data<AppState>,
) -> Result<HttpResponse> {
    if body.name.trim().is_empty() {
        return Err(AppError::BadRequest("name 不能为空".into()).into());
    }

    let id = sqlx::query("INSERT INTO users(name) VALUES (?)")
        .bind(&body.name)
        .execute(&state.db)
        .await? // 自动 From<sqlx::Error>
        .last_insert_rowid();

    Ok(HttpResponse::Created().json(serde_json::json!({ "id": id })))
}
```

当底层错误需要额外上下文、或不能实现 `From` 时（孤儿规则限制），用 `map_err` 手动映射；也可以直接 `map_err(ErrorInternalServerError)` 把任意错误变成 500。

★ **关键认知**：`?` 的前提是"错误类型能转成目标错误类型"。项目里应该只有**一个**应用级错误枚举，所有底层错误通过 `#[from]` 或 `map_err` 汇入它。Handler 里到处写 `Box<dyn Error>` 会让状态码和响应格式失控。

# 6. panic 与恢复

## 6.1 panic 不是错误处理

```rust
async fn bad(state: web::Data<AppState>) -> String {
    let user = sqlx::query("SELECT ...")
        .fetch_one(&state.db)
        .await
        .unwrap(); // 数据库一抖动就 panic
    format!("{:?}", user)
}
```

`unwrap`、`expect`、数组越界、除零都会 panic。worker 会捕获并返回 500，但响应体不受 `ResponseError` 控制，连接上的请求会异常结束，频繁 panic 还会污染日志。

## 6.2 自定义 500 响应

用 `ErrorHandlers` 统一改写 5xx：

```rust
use actix_web::middleware::{ErrorHandlerResponse, ErrorHandlers};
use actix_web::{dev, http::StatusCode, HttpResponse, Result};

fn json_500<B>(res: dev::ServiceResponse<B>) -> Result<ErrorHandlerResponse<B>> {
    let (req, res) = res.into_parts();
    let body = serde_json::json!({
        "code": "INTERNAL_ERROR",
        "message": "服务内部错误"
    });
    let new_res = HttpResponse::InternalServerError().json(body).map_into_right_body();
    Ok(ErrorHandlerResponse::Response(
        dev::ServiceResponse::new(req, new_res),
    ))
}

App::new().wrap(
    ErrorHandlers::new().handler(StatusCode::INTERNAL_SERVER_ERROR, json_500),
)
```

记录 panic：

```rust
std::panic::set_hook(Box::new(|info| {
    log::error!("panic: {}", info); // 记录位置与消息，便于定位
}));
```

⚠ 不要用 `std::panic::catch_unwind` 包 async Handler。异步代码的 panic 可能发生在 `.await` 之后的其他任务上下文里，`catch_unwind` 接不住。能返回错误的地方就不要 panic。如果 release 下配置 `panic = "abort"`，panic 会直接终止进程，不再有 worker 级恢复，需配合进程守护自动重启。

# 7. 错误响应设计

推荐结构包含 `code`、`message`、可选的 `details` 和 `request_id`，例如 `{"code":"VALIDATION_ERROR","message":"邮箱格式不正确"}`。

| 字段 | 是否必须 | 说明 |
|------|----------|------|
| `code` | 是 | 稳定错误码，前端分支依据 |
| `message` | 是 | 面向用户的简短说明 |
| `details` | 否 | 字段级校验信息 |
| `request_id` | 建议 | 与日志对齐，便于排查 |

⚠ `code` 一旦对外发布就要保持稳定。改 `code` 等同于破坏 API 契约，和改字段名一样严重。

# 8. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 所有错误都变 500 | 错误类型没实现 `ResponseError` | 补实现或用 `ErrorInternalServerError` |
| `?` 编译不过 | 缺少 `From` 转换 | `#[from]` 或 `map_err` |
| 状态码与响应体不一致 | `error_response` 里硬编码状态码 | 用 `self.status_code()` |
| 前端看到 SQL 细节 | 500 返回了 `Display` 原文 | 5xx 用通用文案 |
| panic 后没有 JSON | panic 不经过 `ResponseError` | `ErrorHandlers` 兜底 |
| 404 被写成 400 | `ok_or` 时选错变体 | 按语义映射状态码 |

# 9. 动手练习

1. 定义 `AppError`，至少包含 `NotFound`、`BadRequest`、`Unauthorized`、`Database`，并实现 `ResponseError`。
2. 写一个接口，把 `sqlx::Error` 通过 `?` 转成 500，把"用户不存在"转成 404，验证两者响应体不同。
3. 用 `ErrorHandlers` 把 panic 产生的 500 替换成统一 JSON，并确保日志里能看到 panic 位置。

验收标准：客户端永远只看到 `code` + `message`，内部细节只出现在日志里。

# 10. 小结

| 概念 | 要点 |
|------|------|
| `actix_web::Error` | 类型擦除容器，实现 `ResponseError` 即可转入 |
| `ResponseError` | 要求 `Debug + Display`，提供状态码与响应 |
| `#[from]` | 让 `?` 自动转换底层错误 |
| 5xx 文案 | 通用化，细节只进日志 |
| 状态码 | 按语义映射，团队内保持一致 |
| panic | 不是错误处理，用 `ErrorHandlers` 兜底 |
| 错误结构 | `code` + `message` + 可选 `details`/`request_id` |

下一篇进入数据库集成，讲清楚 SQLx 与 Diesel 的选型差异、连接池配置、异步查询与事务写法，以及如何把数据库错误接进这套 `AppError` 体系。
