接口返回什么结构、错误如何表达，直接决定前端和调用方是否好用。本篇从状态码、JSON 响应讲到统一响应结构和全局错误处理，帮助初学者写出可预测、可排错的 API。

# 1. 响应基础

## 1.1 JSON、XML 与 HTML

Gin 提供多种响应方法：

```go
r.GET("/json", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"message": "ok"}) // application/json
})

r.GET("/xml", func(c *gin.Context) {
    c.XML(http.StatusOK, gin.H{"message": "ok"}) // application/xml
})

r.GET("/html", func(c *gin.Context) {
    c.HTML(http.StatusOK, "<h1>Hello</h1>") // text/html
})
```

| 方法 | Content-Type | 适用场景 |
|------|--------------|----------|
| `c.JSON` | `application/json` | 前后端分离 API、移动端接口 |
| `c.XML` | `application/xml` | 需要兼容 XML 的老系统 |
| `c.HTML` | `text/html` | 服务端渲染页面 |
| `c.String` | `text/plain` | 简单文本、健康检查 |
| `c.Data` | 自定义 | 图片、文件等二进制响应 |

现代 Web API 通常统一使用 JSON。不要同一个接口根据参数在不同格式之间切换，除非有明确的兼容需求。

## 1.2 状态码

状态码表达的是**请求结果的大类**，业务错误码表达的是**具体业务原因**，二者不能互相替代。

| 状态码 | 含义 | 常见场景 |
|--------|------|----------|
| `200 OK` | 请求成功 | 查询、更新成功 |
| `201 Created` | 资源创建成功 | POST 创建用户 |
| `204 No Content` | 成功但无响应体 | DELETE 成功 |
| `400 Bad Request` | 请求格式或参数错误 | JSON 解析失败、校验失败 |
| `401 Unauthorized` | 未认证 | 缺少 Token、Token 失效 |
| `403 Forbidden` | 已认证但无权限 | 普通用户访问管理员接口 |
| `404 Not Found` | 资源不存在 | 用户 ID 不存在 |
| `409 Conflict` | 资源冲突 | 用户名已存在 |
| `429 Too Many Requests` | 请求过于频繁 | 限流 |
| `500 Internal Server Error` | 服务端未预期错误 | 空指针、数据库异常 |

★ **关键认知**：不要所有响应都返回 200。状态码是 HTTP 层的通用语言，网关、监控、客户端库都会依赖它。业务错误码放在响应体中，用于区分同一状态码下的不同原因。

# 2. 统一响应结构

## 2.1 成功响应

推荐结构：

```go
type Response struct {
    Code    string `json:"code"`              // 业务码，成功时固定为 OK
    Message string `json:"message"`           // 人类可读信息
    Data    any    `json:"data,omitempty"`    // 业务数据，可为空
}
```

```go
func OK(c *gin.Context, data any) {
    c.JSON(http.StatusOK, Response{
        Code:    "OK",
        Message: "success",
        Data:    data,
    })
}
```

为什么同时保留 HTTP 状态码和业务码：

- HTTP 状态码给通用客户端、网关和监控看。
- 业务码给前端做精确分支，例如 `USER_NOT_FOUND` 和 `INVALID_PASSWORD` 都可能是 400，但处理方式不同。
- `message` 面向人，`code` 面向程序，不要用 message 做条件判断。

## 2.2 错误响应

```go
type ErrorResponse struct {
    Code      string      `json:"code"`
    Message   string      `json:"message"`
    Details   []FieldError `json:"details,omitempty"`
    RequestID string      `json:"requestId,omitempty"`
}

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}
```

错误分类建议：

| 类型 | HTTP 状态码 | 业务码示例 | 是否返回详情 |
|------|-------------|------------|--------------|
| 参数错误 | 400 | `INVALID_ARGUMENT` | 返回字段级错误 |
| 未认证 | 401 | `UNAUTHORIZED` | 不返回内部细节 |
| 无权限 | 403 | `FORBIDDEN` | 不返回内部细节 |
| 资源不存在 | 404 | `NOT_FOUND` | 可返回资源类型 |
| 业务冲突 | 409 | `CONFLICT` | 返回可操作提示 |
| 系统错误 | 500 | `INTERNAL_ERROR` | 只返回 requestId |

# 3. 错误处理

## 3.1 错误分类

在业务层定义可被识别的错误：

```go
type AppError struct {
    Status  int
    Code    string
    Message string
    Err     error // 内部原因，不直接返回给客户端
}

func (e *AppError) Error() string {
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}
```

常见错误：

```go
var ErrUserNotFound = &AppError{
    Status:  http.StatusNotFound,
    Code:    "USER_NOT_FOUND",
    Message: "用户不存在",
}
```

★ **关键认知**：业务错误是"预期内的失败"，例如用户不存在、余额不足；系统错误是"预期外的失败"，例如数据库连接断开、空指针。两者都应记录日志，但只有前者适合把原因返回给用户。

## 3.2 全局错误处理中间件

Gin 的 Handler 没有统一错误返回值，常见做法是把错误写入 Context，再由中间件统一处理：

```go
const ContextError = "appError"

func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) == 0 {
            return
        }

        err := c.Errors.Last().Err
        var appErr *AppError
        if errors.As(err, &appErr) {
            c.JSON(appErr.Status, ErrorResponse{
                Code:      appErr.Code,
                Message:   appErr.Message,
                RequestID: c.GetString("requestID"),
            })
            return
        }

        log.Printf("unexpected error: %v", err) // 内部错误写日志
        c.JSON(http.StatusInternalServerError, ErrorResponse{
            Code:      "INTERNAL_ERROR",
            Message:   "服务内部错误",
            RequestID: c.GetString("requestID"),
        })
    }
}
```

Handler 中：

```go
func GetUser(c *gin.Context) {
    user, err := userService.Find(c.Param("id"))
    if err != nil {
        _ = c.Error(err) // 交给统一错误处理
        return
    }

    OK(c, user)
}
```

⚠ 不要在 Handler 里既 `c.Error(err)` 又直接写响应，否则中间件可能重复写 Header，导致 `superfluous response.WriteHeader call`。

# 4. 日志与请求 ID

一次请求可能经过多个服务和多个日志文件，请求 ID 用来把它们串起来：

```go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := c.GetHeader("X-Request-ID")
        if id == "" {
            id = uuid.NewString()
        }

        c.Set("requestID", id)
        c.Header("X-Request-ID", id)
        c.Next()
    }
}
```

错误日志至少包含：

- 请求 ID。
- 请求方法、路径、状态码。
- 用户 ID（如果已认证）。
- 错误类型和内部原因。
- 发生时间。

★ **关键认知**：日志是给排查问题的人看的，响应是给调用方看的。内部错误栈、SQL 语句、文件路径应写日志，不应出现在响应体中。

# 5. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|----------|
| 所有错误返回 200 | 网关和监控无法识别失败 | 使用正确状态码 |
| 直接返回 `err.Error()` | 泄露 SQL、路径、内部类型 | 转换为统一错误结构 |
| 响应结构前后不一致 | 前端需要写多套解析逻辑 | 固定成功/错误结构 |
| 重复写响应 | 日志出现 `superfluous WriteHeader` | 每个请求只写一次响应 |
| 没有请求 ID | 多服务日志无法关联 | 入口生成并透传请求 ID |
| 错误日志没有上下文 | 只能看到错误文本 | 记录方法、路径、用户、参数摘要 |

# 6. 动手练习

把上一篇的用户注册接口改造成统一响应结构：

1. 成功时返回 `201`，结构为 `{code, message, data}`。
2. 参数错误返回 `400`，包含字段级 `details`。
3. 邮箱已存在返回 `409`，业务码为 `EMAIL_EXISTS`。
4. 未预期错误返回 `500`，只暴露 `requestId`。
5. 补全错误码表，说明每个错误码对应的 HTTP 状态码和前端处理方式。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 状态码 | 表达结果大类，业务码表达具体原因 |
| 成功响应 | `code` + `message` + `data`，结构稳定 |
| 错误响应 | 分类处理，内部细节只进日志 |
| 全局错误 | Handler 返回错误，中间件统一写响应 |
| 请求 ID | 串联日志、响应和排错链路 |
| 常见风险 | 全 200、泄露错误栈、重复写响应 |

下一篇进入静态文件与上传，讲清静态目录、单页应用回退、文件上传下载以及路径穿越等安全问题。
