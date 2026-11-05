能运行的接口不等于可维护的接口，测试是初学者从"写完就算"走向工程化的分水岭。本篇用标准库 `testing` 和 `httptest` 测试 Gin 的 Handler、路由与中间件，并说明如何替换数据库等外部依赖。

# 1. 测试基础

## 1.1 Go 测试约定

- 测试文件以 `_test.go` 结尾。
- 测试函数以 `Test` 开头，参数为 `*testing.T`。
- 基准测试以 `Benchmark` 开头，参数为 `*testing.B`。
- 测试文件和被测代码放在同一个包中，便于访问未导出成员；也可以使用 `_test` 包做黑盒测试。

```go
func TestAdd(t *testing.T) {
    got := Add(1, 2)
    want := 3
    if got != want {
        t.Fatalf("Add(1, 2) = %d, want %d", got, want)
    }
}
```

运行：

```bash
go test ./...
go test -v ./...
go test -run TestAdd ./...
```

## 1.2 httptest 简介

`net/http/httptest` 可以在不启动真实端口的情况下构造请求和记录响应：

```go
req := httptest.NewRequest(http.MethodGet, "/ping", nil)
rec := httptest.NewRecorder()

r := gin.New()
r.GET("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "pong")
})
r.ServeHTTP(rec, req)

if rec.Code != http.StatusOK {
    t.Fatalf("status = %d, want %d", rec.Code, http.StatusOK)
}
```

- `NewRequest` 创建请求对象。
- `NewRecorder` 实现 `http.ResponseWriter`，把响应记录在内存中。
- `r.ServeHTTP` 让 Gin 处理请求，不经过真实网络。

★ **关键认知**：`httptest` 测试的是 HTTP 层行为，不测试网络连接本身。端口占用、TLS、超时等属于集成测试或端到端测试的范围。

# 2. Handler 测试

## 2.1 构造请求

带路径参数：

```go
req := httptest.NewRequest(http.MethodGet, "/users/42", nil)
```

Gin 只有在 `ServeHTTP` 匹配路由后才会填充 `c.Param`，所以测试路径参数时要用真实路由，而不是直接调用 Handler。

带 JSON 请求体：

```go
body := strings.NewReader(`{"name":"alice","email":"alice@example.com"}`)
req := httptest.NewRequest(http.MethodPost, "/users", body)
req.Header.Set("Content-Type", "application/json")
```

带查询参数：

```go
req := httptest.NewRequest(http.MethodGet, "/users?page=2&size=10", nil)
```

## 2.2 断言响应

```go
func TestCreateUser(t *testing.T) {
    r := setupRouter()

    body := strings.NewReader(`{"name":"alice","email":"alice@example.com"}`)
    req := httptest.NewRequest(http.MethodPost, "/users", body)
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    r.ServeHTTP(rec, req)

    if rec.Code != http.StatusCreated {
        t.Fatalf("status = %d, body = %s", rec.Code, rec.Body.String())
    }

    var resp struct {
        Name string `json:"name"`
    }
    if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
        t.Fatalf("decode response: %v", err)
    }
    if resp.Name != "alice" {
        t.Fatalf("name = %q, want %q", resp.Name, "alice")
    }
}
```

⚠ 不要用字符串比较整个 JSON。字段顺序、空格、转义都可能变化，应该先反序列化再比较字段。浮点数和时间字段还要考虑精度与时区。

# 3. 路由与中间件测试

```go
func TestAuthMiddleware(t *testing.T) {
    r := gin.New()
    r.Use(Auth())
    r.GET("/profile", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"userID": c.GetInt64("userID")})
    })

    t.Run("missing token", func(t *testing.T) {
        req := httptest.NewRequest(http.MethodGet, "/profile", nil)
        rec := httptest.NewRecorder()
        r.ServeHTTP(rec, req)

        if rec.Code != http.StatusUnauthorized {
            t.Fatalf("status = %d, want 401", rec.Code)
        }
    })

    t.Run("valid token", func(t *testing.T) {
        token, _ := SignToken(42, "user")
        req := httptest.NewRequest(http.MethodGet, "/profile", nil)
        req.Header.Set("Authorization", "Bearer "+token)
        rec := httptest.NewRecorder()
        r.ServeHTTP(rec, req)

        if rec.Code != http.StatusOK {
            t.Fatalf("status = %d, want 200", rec.Code)
        }
    })
}
```

测试 404 和 405 时，要确保路由和 `NoRoute`/`NoMethod` 已注册：

```go
t.Run("not found", func(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/missing", nil)
    rec := httptest.NewRecorder()
    r.ServeHTTP(rec, req)

    if rec.Code != http.StatusNotFound {
        t.Fatalf("status = %d, want 404", rec.Code)
    }
})
```

# 4. 依赖替换

## 4.1 接口与假实现

Handler 依赖接口，而不是具体数据库：

```go
type UserRepository interface {
    FindByID(ctx context.Context, id int64) (*User, error)
}

type fakeUserRepository struct {
    users map[int64]*User
}

func (f *fakeUserRepository) FindByID(_ context.Context, id int64) (*User, error) {
    user, ok := f.users[id]
    if !ok {
        return nil, ErrUserNotFound
    }
    return user, nil
}
```

在测试中注入 fake：

```go
repo := &fakeUserRepository{
    users: map[int64]*User{42: {ID: 42, Name: "alice"}},
}
handler := NewUserHandler(repo)
r := gin.New()
r.GET("/users/:id", handler.GetByID)
```

★ **关键认知**：可测试性往往反映设计质量。如果 Handler 里直接 `sql.Open` 或调用全局单例，就很难替换依赖。把外部依赖抽象成接口，是解耦和测试的共同基础。

## 4.2 数据库测试

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| `sqlmock` | 快、无需真实数据库 | 只验证 SQL 调用，不验证真实行为 | 单元测试 |
| 临时数据库 | 真实 SQL 行为 | 启动慢、隔离复杂 | 集成测试 |
| Testcontainers | 环境一致、可复现 | 依赖 Docker、启动较慢 | CI 集成测试 |

初学阶段可以先用 fake 测 Handler，再用真实数据库测 repository。不要为了追求"全真实"让单元测试跑几分钟。

# 5. 基准测试

```go
func BenchmarkPing(b *testing.B) {
    r := gin.New()
    r.GET("/ping", func(c *gin.Context) {
        c.String(http.StatusOK, "pong")
    })

    req := httptest.NewRequest(http.MethodGet, "/ping", nil)
    rec := httptest.NewRecorder()

    b.ResetTimer() // 排除准备阶段的耗时
    for i := 0; i < b.N; i++ {
        r.ServeHTTP(rec, req)
    }
}
```

运行：

```bash
go test -bench=BenchmarkPing -benchmem
```

⚠ 复用同一个 `rec` 会累积响应内容，真实基准测试中应根据场景重置；同时避免在循环里做与测试目标无关的初始化。

# 6. 常见错误

| 问题 | 后果 | 建议 |
|------|------|------|
| 测试依赖全局状态 | 顺序不同结果不同 | 每个测试独立初始化 |
| 只断言状态码 | 响应内容错误无法发现 | 同时断言关键字段 |
| 用字符串比较 JSON | 字段顺序变化导致误报 | 反序列化后比较 |
| 测试真实外部服务 | 慢且不稳定 | 用 fake 或容器化依赖 |
| 并发测试共享数据库 | 数据互相污染 | 每个测试独立事务或独立库 |
| 忘记 `t.Parallel` 的共享资源 | 数据竞争 | 只在真正独立时并行 |

# 7. 动手练习

为用户查询接口补全测试：

1. 成功：`GET /users/42` 返回 200 和正确用户。
2. 参数错误：`GET /users/abc` 返回 400。
3. 资源不存在：`GET /users/999` 返回 404。
4. 未授权：没有 Token 访问受保护接口返回 401。
5. 使用 `t.Run` 组织表驱动测试，并让每个用例独立。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 测试文件 | `_test.go`，函数以 `Test` 开头 |
| httptest | 不启动端口，直接测试 HTTP 行为 |
| 断言 | 反序列化后比较字段，不比较整段 JSON 字符串 |
| 依赖 | 用接口隔离数据库，测试注入 fake |
| 数据库测试 | 单元测试用 mock，集成测试用真实库或容器 |
| 基准测试 | 用 `b.ResetTimer`，避免准备阶段干扰结果 |

下一篇进入性能与生产实践，讲基准测试、pprof、优雅关闭、路由树和 Context 池。
