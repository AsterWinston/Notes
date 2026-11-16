中间件让鉴权、日志、恢复、限流等横切逻辑从业务代码中剥离出来，是 Gin 工程化的关键。本篇解释请求处理链、执行顺序和 `c.Next`/`c.Abort` 的真实含义，并给出可直接改造的自定义中间件。

# 1. 中间件是什么

## 1.1 请求处理链

一次请求进入 Gin 后，会按注册顺序经过中间件，最后到达业务 Handler：

```text
请求
  │
  ▼
┌─────────────┐
│ Logger      │
└──────┬──────┘
       ▼
┌─────────────┐
│ Auth        │
└──────┬──────┘
       ▼
┌─────────────┐
│ Handler     │
└──────┬──────┘
       ▼
响应按相反方向返回
```

中间件不是"请求前执行的钩子"这么简单。它可以在调用 `c.Next()` 之后继续执行，因此同一个中间件既能在请求前做事，也能在响应后做事。

## 1.2 Gin 中间件函数签名

```go
func MyMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 请求前逻辑
        c.Next()
        // 响应后逻辑
    }
}
```

- 外层函数用于接收配置，返回真正的中间件函数。
- `gin.HandlerFunc` 本质是 `func(*gin.Context)`。
- 中间件函数返回时，不代表请求结束；如果没有调用 `c.Next()`，后续 Handler 可能不会执行。

★ **关键认知**：Gin 的中间件链是一个"洋葱模型"。进入时从外到内，返回时从内到外。理解这一点，才能正确计算耗时、捕获 panic 和修改响应。

# 2. 中间件执行顺序

## 2.1 全局中间件

```go
r := gin.New()
r.Use(Logger())
r.Use(Recovery())

r.GET("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "pong")
})
```

执行顺序是：`Logger` 进入 → `Recovery` 进入 → Handler → `Recovery` 返回 → `Logger` 返回。

## 2.2 分组中间件

```go
api := r.Group("/api")
api.Use(Auth())
{
    api.GET("/profile", profile)
}
```

全局中间件先执行，再执行分组中间件。分组中间件只作用于该分组及其子分组，不会影响其他路由。

## 2.3 c.Next 与 c.Abort

```go
func Demo() gin.HandlerFunc {
    return func(c *gin.Context) {
        fmt.Println("1. 进入中间件")

        c.Next() // 暂停当前中间件，执行后续中间件和 Handler

        fmt.Println("4. 回到中间件")
    }
}

r.Use(Demo())
r.GET("/demo", func(c *gin.Context) {
    fmt.Println("2. 进入 Handler")
    c.String(http.StatusOK, "ok")
    fmt.Println("3. Handler 返回")
})
```

输出顺序：

```text
1. 进入中间件
2. 进入 Handler
3. Handler 返回
4. 回到中间件
```

`c.Abort()` 会阻止后续 Handler 和中间件继续执行，但**不会自动写响应**：

```go
func Auth() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "code":    "UNAUTHORIZED",
                "message": "缺少认证信息",
            })
            return // 写完响应后立即返回，避免继续执行
        }

        c.Next()
    }
}
```

⚠ 只用 `c.Abort()` 而不写响应，客户端可能收到空的 200。鉴权失败通常应使用 `AbortWithStatusJSON` 或先写响应再 `Abort`。

# 3. 自定义中间件

## 3.1 请求日志

```go
func RequestLogger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        c.Next() // 先让请求走完，才能拿到状态码和耗时

        log.Printf(
            "method=%s path=%s status=%d cost=%s",
            c.Request.Method,
            c.Request.URL.Path,
            c.Writer.Status(),
            time.Since(start),
        )
    }
}
```

`c.Writer.Status()` 在 Handler 执行后才有意义。如果在 `c.Next()` 之前读取，通常得到默认值 200。

## 3.2 鉴权

下面是一个简化的 Token 校验中间件：

```go
func Auth() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "code":    "UNAUTHORIZED",
                "message": "请先登录",
            })
            return
        }

        userID, err := parseToken(token) // 伪代码：验证签名与过期时间
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "code":    "INVALID_TOKEN",
                "message": "登录已失效",
            })
            return
        }

        c.Set("userID", userID) // 把用户信息传给后续 Handler
        c.Next()
    }
}
```

## 3.3 异常恢复

Gin 自带 `Recovery()`，可以捕获 Handler 中的 panic 并返回 500：

```go
r := gin.New()
r.Use(gin.Logger(), gin.Recovery())
```

生产中通常需要自定义恢复逻辑，记录请求信息和堆栈，但不要把这些内部信息返回给客户端：

```go
func Recovery() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack()) // 堆栈只写日志
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "code":    "INTERNAL_ERROR",
                    "message": "服务内部错误",
                })
            }
        }()

        c.Next()
    }
}
```

# 4. 中间件间传值

```go
c.Set("userID", int64(42))

value, exists := c.Get("userID")
if !exists {
    // 没有设置过
}

userID, ok := value.(int64) // 存储的是 any，需要类型断言
```

建议把键名定义成常量，避免字符串拼写错误：

```go
const ContextUserID = "userID"

c.Set(ContextUserID, userID)
```

⚠ 不要把 `gin.Context` 传给异步 goroutine。请求结束后 Context 会被复用，异步任务应复制需要的数据，或者使用 `c.Copy()` 创建只读副本。

# 5. 常见错误与性能注意

| 问题 | 后果 | 正确做法 |
|------|------|----------|
| 鉴权失败只 `Abort` 不写响应 | 客户端收到空响应或 200 | 使用 `AbortWithStatusJSON` |
| 重复调用 `c.Next()` | 后续 Handler 执行多次 | 每个中间件最多调用一次 |
| 在中间件里做慢查询 | 所有请求变慢 | 只做必要校验，重逻辑放业务层 |
| 把 Context 传给 goroutine | 数据竞争、读到其他请求的数据 | 复制数据或使用 `c.Copy()` |
| 中间件顺序错误 | 日志拿不到用户信息、恢复捕获不到 panic | 按依赖关系安排顺序 |

# 6. 动手练习

实现一个请求 ID 中间件，要求：

1. 优先读取请求头 `X-Request-ID`，没有则生成 UUID。
2. 把请求 ID 写入 `c.Set`，同时写入响应头。
3. 请求日志中打印请求 ID、方法、路径、状态码和耗时。
4. 鉴权失败和 panic 响应中带上请求 ID，方便排查。

验收标准：同一次请求的日志、响应头和错误响应中的请求 ID 完全一致。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 中间件模型 | 洋葱模型，`c.Next` 前是请求阶段，后是响应阶段 |
| 执行顺序 | 全局中间件 → 分组中间件 → Handler → 反向返回 |
| 中断 | `c.Abort()` 停止后续处理，但响应要自己写 |
| 传值 | `c.Set` / `c.Get`，注意类型断言和键名管理 |
| 常见风险 | 忘记 Abort、重复 Next、跨 goroutine 使用 Context |

下一篇进入参数绑定与校验，讲清 Query、Form、JSON、Path 四类参数来源以及 `ShouldBind` 系列方法的差异。
