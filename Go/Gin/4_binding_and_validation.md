从请求中拿到正确、可信、可校验的参数，是接口开发的第一步。本篇区分 Query、Form、JSON、Path 四类参数来源，讲清 `ShouldBind` 系列方法的差异，并给出适合初学者的校验与错误返回方式。

# 1. 参数来源

## 1.1 Query、Form、JSON、Path

HTTP 请求可以把参数放在不同位置。先判断参数从哪里来，再选择对应的读取方式：

| 来源 | 示例 | 常见 Content-Type | 适用场景 |
|------|------|-------------------|----------|
| Path | `/users/42` | 无 | 资源 ID、层级路径 |
| Query | `/users?page=1&size=10` | 无 | 分页、搜索、过滤 |
| Form | `username=alice&age=18` | `application/x-www-form-urlencoded` | 普通表单提交 |
| JSON | `{"name":"alice"}` | `application/json` | 前后端分离接口 |
| Multipart | 文件 + 字段 | `multipart/form-data` | 文件上传 |

★ **关键认知**：参数来源错误时，绑定往往不会报"参数放错位置"，而是得到一个零值。例如把 JSON 字段放在 Query 里，结构体里对应字段会是空字符串。排错时先看请求的实际格式。

## 1.2 ShouldBind 系列方法区别

Gin 的绑定方法主要区别在**如何报告错误**：

| 方法 | 绑定失败时 | 是否自动写响应 |
|------|------------|----------------|
| `ShouldBind` | 返回 error | 否 |
| `ShouldBindJSON` | 返回 error，只绑定 JSON | 否 |
| `Bind` | 自动写 400 并中止 | 是 |
| `BindJSON` | 自动写 400 并中止 | 是 |

初学者建议统一使用 `ShouldBind` 系列，然后自己返回结构化错误。`Bind` 的自动响应格式通常不符合项目的统一响应规范。

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{
        "code":    "INVALID_ARGUMENT",
        "message": "请求参数格式错误",
        "detail":  err.Error(),
    })
    return
}
```

# 2. 结构体绑定

## 2.1 标签与字段映射

结构体标签告诉 Gin 如何把请求数据映射到字段：

```go
type CreateUserRequest struct {
    Name  string `json:"name" form:"name" uri:"name"`
    Email string `json:"email" form:"email"`
    Age   int    `json:"age" form:"age"`
}
```

- `json`：JSON 请求体字段名。
- `form`：表单或 Query 字段名。
- `uri`：路径参数名。
- 字段必须是**可导出**的，也就是首字母大写，否则反射无法赋值。

⚠ Go 的 JSON 解码默认不区分字段名大小写，但为了契约清晰，建议客户端和服务端使用同一种命名风格，例如统一 `snake_case` 或统一 `camelCase`。

## 2.2 绑定 JSON 请求体

```go
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=2,max=20"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,gte=1,lte=120"`
}

func CreateUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "code":    "INVALID_ARGUMENT",
            "message": "参数校验失败",
            "detail":  err.Error(),
        })
        return
    }

    c.JSON(http.StatusCreated, gin.H{
        "name":  req.Name,
        "email": req.Email,
        "age":   req.Age,
    })
}
```

对应的请求：

```bash
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"alice","email":"alice@example.com","age":18}'
```

如果缺少 `email`，或者 `age` 是字符串，`ShouldBindJSON` 会返回错误。缺失字段和类型不匹配都会导致绑定失败，但错误文本不同，调试时可以打印 `err`。

# 3. 参数校验

## 3.1 binding 标签

Gin 使用 `go-playground/validator` 的标签语法：

| 标签 | 含义 | 示例 |
|------|------|------|
| `required` | 必填且非零值 | `binding:"required"` |
| `min` / `max` | 字符串长度或数字范围 | `binding:"min=2,max=20"` |
| `gte` / `lte` | 大于等于 / 小于等于 | `binding:"gte=1,lte=120"` |
| `email` | 邮箱格式 | `binding:"email"` |
| `oneof` | 枚举值之一 | `binding:"oneof=admin user"` |
| `url` | URL 格式 | `binding:"url"` |

⚠ `required` 对数字字段判断的是"非零值"，所以 `age=0` 会被当成未填写。如果 0 是合法业务值，应使用指针字段或自定义校验。

## 3.2 自定义校验

有些规则无法用内置标签表达，例如"用户名不能是保留字"：

```go
var reservedNames = map[string]bool{
    "admin": true,
    "root":  true,
}

func ValidateUsername(fl validator.FieldLevel) bool {
    name := fl.Field().String()
    return !reservedNames[name]
}

func main() {
    r := gin.Default()

    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        if err := v.RegisterValidation("username", ValidateUsername); err != nil {
            panic(err) // 注册失败通常意味着标签名重复或函数签名错误
        }
    }

    r.POST("/users", createUser)
    r.Run()
}
```

使用：

```go
type CreateUserRequest struct {
    Name string `json:"name" binding:"required,username"`
}
```

自定义校验适合简单、可复用的规则。涉及数据库查询的校验（例如用户名是否已存在）不建议放在 validator 中，避免校验层与数据层耦合。

## 3.3 错误信息国际化

`err.Error()` 适合开发调试，不适合直接返回给终端用户。建议转换成统一结构：

```go
func BindError(err error) gin.H {
    var validationErrors validator.ValidationErrors
    if errors.As(err, &validationErrors) {
        details := make([]gin.H, 0, len(validationErrors))
        for _, fieldError := range validationErrors {
            details = append(details, gin.H{
                "field":   fieldError.Field(),
                "tag":     fieldError.Tag(),
                "message": fieldMessage(fieldError),
            })
        }
        return gin.H{
            "code":    "INVALID_ARGUMENT",
            "message": "参数校验失败",
            "details": details,
        }
    }

    return gin.H{
        "code":    "INVALID_ARGUMENT",
        "message": "请求参数格式错误",
    }
}

func fieldMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return "该字段必填"
    case "email":
        return "邮箱格式不正确"
    case "min", "gte":
        return "数值或长度过小"
    case "max", "lte":
        return "数值或长度过大"
    default:
        return "字段不合法"
    }
}
```

★ **关键认知**：对外错误信息要稳定、可读，不要把内部类型名、结构体名直接暴露给用户。错误码用于程序判断，message 用于人阅读，二者不要混在一起。

# 4. 绑定边界与默认值

零值问题是最常见的绑定陷阱。对于 `int` 字段，`0` 既可能是用户传入的值，也可能是没有传：

```go
type UpdateRequest struct {
    Age *int `json:"age"` // 指针可以区分 nil 和 0
}
```

判断逻辑：

```go
if req.Age != nil {
    // 用户明确传了 age，包括 0
}
```

默认值建议在绑定后显式设置：

```go
page := 1
if req.Page != nil {
    page = *req.Page
}
```

⚠ 不要通过 `binding:"required"` 给字段设置默认值，它只负责校验，不负责补值。

# 5. 常见错误与安全注意

| 问题 | 后果 | 建议 |
|------|------|------|
| 忽略 `ShouldBind` 的 error | 使用零值继续业务 | 绑定失败立即返回 |
| 允许未知字段 | 客户端拼错字段也被静默忽略 | 需要严格模式时使用 `DisallowUnknownFields` |
| 请求体过大 | 内存被占满 | 限制 `MaxBytesReader` 或反向代理大小 |
| 直接把 `err.Error()` 返回 | 泄露内部结构 | 转成统一错误结构 |
| 路径参数不校验 | `abc` 被当作 ID | 显式转换并检查范围 |

# 6. 动手练习

实现用户注册接口：

1. 请求体包含 `username`、`email`、`age`、`role`。
2. `username` 长度 3-20，不能是 `admin`。
3. `email` 必须是合法邮箱。
4. `age` 范围 1-120，且必须显式传入。
5. `role` 只能是 `admin` 或 `user`。
6. 绑定失败时返回统一错误结构，包含字段级错误。

进阶：把错误信息改成中文，并为每个字段写一条测试。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 参数来源 | Path、Query、Form、JSON、Multipart，先判断来源再绑定 |
| 绑定方法 | 优先用 `ShouldBind` 系列，自己控制错误响应 |
| 字段映射 | 标签决定字段名，字段必须可导出 |
| 校验 | `binding` 标签覆盖常用规则，复杂规则用自定义校验 |
| 零值 | 用指针区分"没传"和"传了 0" |
| 错误 | 对外返回稳定结构，不泄露内部错误 |

下一篇进入响应与错误处理，讲清状态码语义、统一响应结构和全局错误处理中间件。
