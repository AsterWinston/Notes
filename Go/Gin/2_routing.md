路由决定一个 HTTP 请求交给哪个处理函数，是 Gin 使用中最先要掌握的部分。本篇从路径参数、查询参数讲到路由分组、匹配优先级和冲突排错，帮助初学者建立清晰的路由模型。

# 1. 路由基础

## 1.1 HTTP 方法与路由注册

一个完整的路由由**方法 + 路径 + 处理函数**组成：

```go
r := gin.Default()

r.GET("/users", listUsers)       // 查询用户列表
r.POST("/users", createUser)     // 创建用户
r.PUT("/users/:id", updateUser)  // 整体更新
r.PATCH("/users/:id", patchUser) // 部分更新
r.DELETE("/users/:id", deleteUser)
```

同一个路径可以注册多个方法，Gin 会分别匹配。例如 `GET /users` 和 `POST /users` 不会冲突。

也支持一次注册多个方法：

```go
r.Any("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "pong") // 任意方法都返回 pong
})
```

★ **关键认知**：路由方法不是"给函数贴标签"，而是匹配条件的一部分。用错方法时，路径再正确也不会进入处理函数，通常表现为 404 或 405。

## 1.2 路径参数

路径参数用 `:` 声明，适合表达资源标识：

```go
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id") // 得到字符串形式的 id

    userID, err := strconv.ParseInt(id, 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "id 必须是数字"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"id": userID})
})
```

请求 `/users/42` 时，`id` 为 `"42"`。请求 `/users/abc` 时仍会进入同一个 Handler，因此**类型转换和合法性校验必须自己做**。

多个路径参数：

```go
r.GET("/users/:uid/orders/:oid", func(c *gin.Context) {
    uid := c.Param("uid")
    oid := c.Param("oid")
    c.JSON(http.StatusOK, gin.H{"uid": uid, "oid": oid})
})
```

⚠ `c.Param` 返回空字符串时，通常说明参数名写错或路由没有匹配到该参数段，不要直接把空值当合法参数使用。

## 1.3 查询参数

查询参数来自 URL 中 `?` 后面的键值对，例如 `/search?q=gin&page=2`。

```go
r.GET("/search", func(c *gin.Context) {
    q := c.Query("q")                    // 不存在时返回空字符串
    page := c.DefaultQuery("page", "1")  // 不存在时返回默认值
    size, ok := c.GetQuery("size")       // ok 表示参数是否真的存在

    if !ok {
        size = "10" // 手动设置默认值
    }

    c.JSON(http.StatusOK, gin.H{
        "q":    q,
        "page": page,
        "size": size,
    })
})
```

| 方法 | 参数不存在时 | 适合场景 |
|------|--------------|----------|
| `Query` | 返回 `""` | 能接受空字符串 |
| `DefaultQuery` | 返回指定默认值 | 分页、排序等有默认值的参数 |
| `GetQuery` | 返回 `("", false)` | 需要区分"没传"和"传了空值" |

# 2. 路由分组

## 2.1 Group 的作用

分组用于复用路径前缀和中间件：

```go
api := r.Group("/api/v1")
{
    api.GET("/users", listUsers)
    api.POST("/users", createUser)
}
```

实际注册的路径是 `/api/v1/users`。分组还可以挂载中间件：

```go
auth := api.Group("/admin")
auth.Use(authMiddleware) // 只作用于 /api/v1/admin 下的路由
{
    auth.GET("/stats", getStats)
}
```

## 2.2 嵌套分组与版本管理

```go
v1 := r.Group("/api/v1")
v2 := r.Group("/api/v2")

adminV1 := v1.Group("/admin")
adminV1.Use(adminOnly)
{
    adminV1.GET("/users", listAllUsers)
}
```

★ **关键认知**：版本分组的价值在于让旧接口继续可用。不要直接改 `/api/v1` 的语义；不兼容变更应放到 `/api/v2`，并在文档中说明迁移方式。

# 3. 路由匹配规则

## 3.1 静态路由与参数路由优先级

Gin 使用基数树（radix tree）组织路由。静态路径通常优先于参数路径：

```go
r.GET("/users/me", getCurrentUser) // 静态路径
r.GET("/users/:id", getUserByID)   // 参数路径
```

请求 `/users/me` 会命中第一个，而不是把 `me` 当成 id。

但如果同时注册 `/users/:id` 和 `/users/:name`，就会产生冲突，因为同一位置出现两个不同的参数名。Gin 在启动时通常会直接 panic，而不是等到请求时才报错。

⚠ 路由冲突是"启动期错误"，不要用 recover 掩盖；应根据报错信息调整路由设计。

## 3.2 通配符与冲突

Gin 支持两种特殊段：

| 写法 | 含义 | 示例 |
|------|------|------|
| `:param` | 匹配单个路径段 | `/users/:id` 匹配 `/users/1`，不匹配 `/users/1/orders` |
| `*filepath` | 匹配剩余所有路径 | `/static/*filepath` 匹配 `/static/css/app.css` |

```go
r.GET("/static/*filepath", func(c *gin.Context) {
    p := c.Param("filepath") // 注意：通常以 / 开头
    c.String(http.StatusOK, p)
})
```

⚠ 通配符只能出现在路径末尾附近的固定位置，且与已有路由可能冲突。设计时优先使用明确的资源路径，避免一个通配符覆盖过多接口。

# 4. 404 与 405 处理

默认情况下，未匹配的路径返回 404。可以用 `NoRoute` 统一处理：

```go
r.NoRoute(func(c *gin.Context) {
    c.JSON(http.StatusNotFound, gin.H{
        "code":    "NOT_FOUND",
        "message": "接口不存在",
        "path":    c.Request.URL.Path,
    })
})
```

`NoMethod` 用于处理方法不匹配：

```go
r.NoMethod(func(c *gin.Context) {
    c.JSON(http.StatusMethodNotAllowed, gin.H{
        "code":    "METHOD_NOT_ALLOWED",
        "message": "请求方法不支持",
    })
})
```

★ **关键认知**：404 是"找不到资源或路由"，405 是"路由存在但不支持该方法"。两者语义不同，前端和监控系统会据此判断问题类型。

# 5. 常见错误与排错

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| 路径参数取不到 | 参数名写错、路由未匹配 | 打印 `c.FullPath()` 和 `c.Params` |
| 查询参数总是空 | 参数放在路径或请求体里 | 确认 URL 中是否有 `?key=value` |
| 启动时 panic | 路由冲突或非法通配符 | 阅读 panic 中的冲突路径 |
| 带尾斜杠 404 | 注册路径与请求路径不一致 | 统一约定是否使用尾斜杠 |
| 分组中间件没生效 | 在 `Use` 前注册了路由 | 先 `Use`，再注册该组路由 |

# 6. 动手练习

设计一组用户资源路由，并写明每个路由的方法、路径和参数来源：

1. `GET /users`：分页查询，参数 `page`、`size` 来自 Query。
2. `GET /users/:id`：查询单个用户，`id` 来自 Path。
3. `POST /users`：创建用户，JSON 请求体。
4. `PUT /users/:id`：整体更新，`id` 来自 Path，字段来自 JSON。
5. `DELETE /users/:id`：删除用户。

进一步要求：把所有用户路由放进 `/api/v1` 分组，并为 `/api/v1/admin` 分组预留鉴权中间件。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 路由组成 | 方法 + 路径 + Handler |
| 路径参数 | `:param` 匹配单段，取到的是字符串，需要自行校验 |
| 查询参数 | `Query` / `DefaultQuery` / `GetQuery` 的区别在缺省行为 |
| 分组 | 复用前缀和中间件，适合版本与权限隔离 |
| 匹配优先级 | 静态路由优先于参数路由，冲突会在启动期暴露 |
| 404/405 | 用 `NoRoute`、`NoMethod` 返回统一错误结构 |

下一篇进入中间件，解释请求处理链、执行顺序以及 `c.Next` 与 `c.Abort` 的真实行为。
