context 是 Go 并发编程里传递"取消信号、超时、请求元数据"的标准机制。几乎每个涉及网络、RPC、数据库的服务端代码都会用到它。理解 context 的设计意图和底层实现，是写出优雅可取消代码的关键。

# 1. context 解决的问题

## 1.1 为什么需要 context

★ **关键认知**：在服务端，一个请求往往触发一串调用链（handler → service → 数据库/RPC）。当请求被取消（客户端断开、超时），需要**把取消信号传播给整条调用链**，让所有相关的 goroutine 停止工作、释放资源：

```
请求处理链：
client → handler → service → DB 查询
                    └→ RPC 调用 → 下游服务

问题：client 断开后，如何让 DB 查询和 RPC 调用都及时停止？
答案：用 context 沿调用链传递取消信号。
```

## 1.2 context 接口

```go
// context.Context 接口
type Context interface {
    Deadline() (deadline time.Time, ok bool)   // 截止时间
    Done() <-chan struct{}                     // 取消信号 channel
    Err() error                                // 取消原因
    Value(key any) any                         // 值传递
}
```

★ **关键认知**：context 的核心是 `Done()` 返回的 channel——当 context 被取消时，这个 channel 会被关闭。调用方通过 `<-ctx.Done()` 感知取消。

# 2. 四种创建方式

## 2.1 根 context

```go
ctx := context.Background()    // 根 context，永不取消（进程级）
ctx := context.TODO()          // 占位符，还没想好用什么
```

★ **关键认知**：`Background` 和 `TODO` 本质相同（都是空 context），区别只是语义——`Background` 是"程序的根"，`TODO` 是"将来再补"。两者都永不取消、无值。

## 2.2 派生 context

```go
// 1. 可取消的 context
ctx, cancel := context.WithCancel(parent)
defer cancel()          // 记得调用 cancel 释放资源

// 2. 带超时的 context
ctx, cancel := context.WithTimeout(parent, 3*time.Second)

// 3. 带截止时间的 context
ctx, cancel := context.WithDeadline(parent, time.Now().Add(time.Hour))

// 4. 携带值的 context
ctx := context.WithValue(parent, key, value)
```

# 3. 取消传播机制

## 3.1 父取消，子自动取消

★ **关键认知**：context 是**树形结构**，父 context 取消时，所有子 context 自动取消：

```
        Background
            │
        WithCancel
        /        \
   子 context   子 context     ← 父取消，所有子都取消
```

```go
ctx, cancel := context.WithCancel(context.Background())
childCtx, _ := context.WithCancel(ctx)

cancel()                 // 取消父
<-childCtx.Done()        // 子 context 也被取消
```

## 3.2 取消的实现

★ **关键认知**：context 的取消实现依赖 `cancelCtx` 结构（`context/context.go`）：

```go
type cancelCtx struct {
    Context
    done     chan struct{}   // 取消时关闭这个 channel
    mu       sync.Mutex
    children map[canceler]struct{}   // 子 context 集合
    err      error
}
```

```
取消流程：
1. cancel() 被调用 → 关闭 done channel
2. 遍历 children，递归取消所有子 context
3. 所有 <-ctx.Done() 的地方都会收到关闭信号
```

★ **关键认知**：关闭 channel 是 Go 里广播信号的惯用手段——**关闭一个 channel 会让所有等待它的接收者立即返回**。context 正是用"关闭 done channel"实现一对多的取消广播。

# 4. 正确使用姿势

## 4.1 规则

★ **关键认知**：官方 context 使用规范：

```
1. context 作为第一个参数传递，命名通常为 ctx
2. 不要把 context 存进结构体（作为字段），要显式传递
3. context 是并发安全的（可同时被多个 goroutine 使用）
4. 不要传 nil context（不确定就用 TODO）
5. 只传递请求级数据，不用 context 传可选参数
```

## 4.2 配合 select 使用

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():        // 感知取消
            fmt.Println("cancelled:", ctx.Err())
            return
        default:
            // 正常工作
        }
    }
}
```

★ **关键认知**：context 和 select 是黄金搭档。`ctx.Done()` 返回的 channel 放进 select，让 goroutine 能同时响应"工作事件"和"取消事件"。这是可取消 goroutine 的标准写法。

# 5. 超时与截止时间

## 5.1 超时控制

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

result, err := doWork(ctx)   // doWork 内部会在 3 秒后被取消
```

★ **关键认知**：`WithTimeout` 会在指定时间后自动调用 cancel。**必须 `defer cancel()`** 及时释放 context 资源（否则定时器一直存活，导致内存泄漏）。

## 5.2 传递超时给下游

```go
func handler(ctx context.Context) {
    // 把 ctx 传给数据库查询、RPC 调用
    rows, err := db.QueryContext(ctx, "SELECT ...")   // 数据库感知取消
    resp, err := client.Call(ctx, req)                // RPC 感知取消
}
```

★ **关键认知**：Go 标准库和主流框架的 I/O 方法几乎都支持 context（`QueryContext`、`http.NewRequestWithContext` 等）。把 ctx 一路传下去，才能让取消信号贯穿整个调用链，避免"上层取消了、下层还在傻等"。

# 6. context 传值的正确用法

## 6.1 WithValue 的使用

```go
type keyType string
const userKey keyType = "user"

ctx := context.WithValue(context.Background(), userKey, "tom")
user := ctx.Value(userKey).(string)
```

## 6.2 传值的注意事项

★ **关键认知**：context 传值**只用于请求级元数据**（如用户 ID、trace ID），且必须：

```
1. key 使用自定义类型（避免和其他包的字符串 key 冲突）
2. 不要传可变对象（避免并发问题）
3. 不要用 context 传"可选参数"（应显式传参）
```

⚠ **关键认知**：context 传值是**类型不安全的**（Value 返回 any，需断言），且丢失编译期检查。能用显式参数表达的，就不要塞进 context。滥用 WithValue 会让代码难以追踪和维护。

# 7. 常见陷阱

★ **关键认知**：context 使用中的常见错误：

```
1. 忘记 defer cancel() → 定时器/goroutine 泄漏
2. 把 context 存进结构体字段 → 违背显式传递原则
3. 用 string 作 WithValue 的 key → 可能和其他包冲突
4. 在循环里创建 context 而不复用 → 性能浪费
5. 忽略 ctx.Err()，只判断 Done 不读错误原因
```

# 8. 小结

| 概念 | 要点 |
|------|------|
| 作用 | 取消/超时/值传递沿调用链传播 |
| Done() | 返回 channel，取消时关闭 |
| WithCancel | 手动取消 |
| WithTimeout | 自动超时 |
| 父子传播 | 父取消子自动取消 |
| 使用 | 第一参数、显式传递、defer cancel |
| WithValue | 仅请求级元数据，自定义 key |

下一篇进入标准库深入，讲清楚 net/http、io、bufio、encoding/json 等核心包的内部实现，以及它们如何体现 Go 的设计哲学。
