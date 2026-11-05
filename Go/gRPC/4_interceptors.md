gRPC 拦截器（interceptor）就是 gRPC 版的中间件：鉴权、日志、链路追踪、panic 恢复、限流这些横切逻辑都靠它。它的坑不在 API，而在**链式顺序**和**流式接口的特殊性**——顺序写反，日志会记错耗时；只写 Unary 拦截器，流式接口就完全裸奔。

# 1. 拦截器的两种形态

## 1.1 为什么分成 Unary 和 Stream

gRPC 的调用分一元和流式两类，函数签名不同，所以拦截器也要分两套：

| 类型 | 服务端注册 | 客户端注册 |
|------|-----------|-----------|
| Unary | `grpc.UnaryInterceptor` | `grpc.WithUnaryInterceptor` |
| Stream | `grpc.StreamInterceptor` | `grpc.WithStreamInterceptor` |
| 多个 Unary | `grpc.ChainUnaryInterceptor` | `grpc.WithChainUnaryInterceptor` |
| 多个 Stream | `grpc.ChainStreamInterceptor` | `grpc.WithChainStreamInterceptor` |

★ **关键认知**：服务端只注册 Unary 拦截器时，所有流式 RPC **完全绕过**它。鉴权如果只写在 Unary 里，攻击者用流式接口就能直接访问业务逻辑。生产环境两套都要有。

## 1.2 四种签名

```go
// 服务端一元：handler 是真正的业务方法
type UnaryServerInterceptor func(
    ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler,
) (any, error)

// 服务端流式：handler 负责执行整个流
type StreamServerInterceptor func(
    srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler,
) error

// 客户端一元：invoker 是真正发请求的函数
type UnaryClientInterceptor func(
    ctx context.Context, method string, req, reply any,
    cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption,
) error

// 客户端流式：streamer 负责创建流
type StreamClientInterceptor func(
    ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string,
    streamer grpc.Streamer, opts ...grpc.CallOption,
) (grpc.ClientStream, error)
```

# 2. 链式顺序

## 2.1 注册顺序 = 洋葱从外到内

```go
grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        RecoveryInterceptor(), // 最外层：能兜住后面所有 panic
        LoggingInterceptor(),  // 第二层：记录真实耗时
        AuthInterceptor(),     // 最内层：接近业务
    ),
)
```

执行顺序：

```text
请求进入
  │
  ▼
┌──────────────────────────────────┐
│ Recovery（defer recover）         │
│ ┌──────────────────────────────┐ │
│ │ Logging（start / defer end）  │ │
│ │ ┌──────────────────────────┐ │ │
│ │ │ Auth（校验 token）        │ │ │
│ │ │ ┌──────────────────────┐ │ │ │
│ │ │ │ 业务 Handler          │ │ │ │
│ │ │ └──────────────────────┘ │ │ │
│ │ └──────────────────────────┘ │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
  │
  ▼
响应返回（顺序反过来）
```

★ **关键认知**：`ChainUnaryInterceptor` 里**先注册的是最外层**。所以恢复必须放第一个（否则内层 panic 它接不到），日志放恢复之后（否则 panic 的请求记不到耗时），鉴权放靠内（先确认是合法请求，再进业务）。

## 2.2 客户端顺序

```go
grpc.WithChainUnaryInterceptor(
    ClientLogging(),   // 最外层，记录总耗时
    ClientMetrics(),   // 统计
    ClientRetry(),     // 内层，重试会重复触发内层拦截器
)
```

⚠ **关键认知**：客户端重试拦截器如果放在日志外层，一次逻辑调用会打印多条"请求日志"；如果放在内层，日志只记一次但看不到每次重试。**想清楚你要的是"调用次数"还是"网络请求次数"**，再决定顺序。

# 3. 服务端实战

## 3.1 日志拦截器

```go
func LoggingInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (any, error) {

        start := time.Now()

        // 从 metadata 里取 trace id；没有就生成一个
        md, _ := metadata.FromIncomingContext(ctx)
        traceID := ""
        if v := md.Get("x-trace-id"); len(v) > 0 {
            traceID = v[0]
        }

        resp, err := handler(ctx, req) // 调用内层

        st, _ := status.FromError(err)
        log.Printf("trace=%s method=%s code=%s cost=%s",
            traceID, info.FullMethod, st.Code(), time.Since(start))
        return resp, err
    }
}
```

## 3.2 鉴权拦截器

```go
var errUnauthenticated = status.Error(codes.Unauthenticated, "missing or invalid token")

func AuthInterceptor(verify func(string) (string, error)) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (any, error) {

        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
            return nil, errUnauthenticated
        }

        tokens := md.Get("authorization") // 小写，HTTP/2 header 不区分大小写
        if len(tokens) == 0 {
            return nil, errUnauthenticated
        }

        userID, err := verify(strings.TrimPrefix(tokens[0], "Bearer "))
        if err != nil {
            return nil, errUnauthenticated
        }

        // 把解析出的身份塞进 ctx，业务层可以直接取
        ctx = context.WithValue(ctx, userIDKey{}, userID)
        return handler(ctx, req)
    }
}
```

★ **关键认知**：鉴权结果通过 `context.WithValue` 往下传，**不要用全局变量**。每个请求的 ctx 是独立的，天然隔离并发。key 用自定义空结构体类型，避免和其他包的 key 冲突。

## 3.3 panic 恢复拦截器

```go
func RecoveryInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (resp any, err error) {

        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic in %s: %v\n%s", info.FullMethod, r, debug.Stack())
                // 对外只给通用错误，不要把 panic 细节泄露给客户端
                err = status.Error(codes.Internal, "internal server error")
            }
        }()

        return handler(ctx, req)
    }
}
```

⚠ **关键认知**：gRPC 服务端**不会**自动 recover 业务 handler 里的 panic。没有恢复拦截器，一次空指针就能让整个进程退出。客户端看到的是 `Unavailable`，日志里却什么都没有——这类事故非常常见。

## 3.4 流式版本要包一层 ServerStream

流式拦截器拿不到 `ctx` 参数，只能通过 `ss.Context()` 取。如果要在流式链路里注入值，需要包装 `grpc.ServerStream`：

```go
type wrappedStream struct {
    grpc.ServerStream
    ctx context.Context
}

func (w *wrappedStream) Context() context.Context { return w.ctx }

func AuthStreamInterceptor(verify func(string) (string, error)) grpc.StreamServerInterceptor {
    return func(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo,
        handler grpc.StreamHandler) error {

        md, _ := metadata.FromIncomingContext(ss.Context())
        tokens := md.Get("authorization")
        if len(tokens) == 0 {
            return errUnauthenticated
        }

        userID, err := verify(strings.TrimPrefix(tokens[0], "Bearer "))
        if err != nil {
            return errUnauthenticated
        }

        ctx := context.WithValue(ss.Context(), userIDKey{}, userID)
        return handler(srv, &wrappedStream{ServerStream: ss, ctx: ctx})
    }
}
```

★ **关键认知**：`wrappedStream` 只重写 `Context()`，其余 `SendMsg`/`RecvMsg` 通过内嵌透传。这是 gRPC 流式拦截器的标准写法——**嵌入接口 + 覆写需要改的方法**。

# 4. 客户端拦截器

## 4.1 统一注入 token

```go
func ClientAuth(token string) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any,
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        // 追加到已有 metadata，不要覆盖调用方自己设置的
        ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

## 4.2 客户端超时

```go
func ClientTimeout(d time.Duration) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any,
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        // 如果调用方已经设置了更短的 deadline，不要延长
        if _, ok := ctx.Deadline(); ok {
            return invoker(ctx, method, req, reply, cc, opts...)
        }

        ctx, cancel := context.WithTimeout(ctx, d)
        defer cancel()
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

⚠ **关键认知**：客户端拦截器里创建的 `cancel` **必须在返回前调用**，否则 timer 泄漏、ctx 泄漏。`defer cancel()` 是硬性要求，即使调用成功也要调。

## 4.3 服务端与客户端拦截器的差异

| 维度 | 服务端拦截器 | 客户端拦截器 |
|------|-------------|-------------|
| 目标 | 校验入站请求、保护服务 | 注入凭据、统一超时、重试、埋点 |
| 拿到的东西 | `info.FullMethod`、`req` | `method`、`cc`、`opts` |
| 能否改响应 | 能，包装后返回 | 能，修改 `reply` |
| 重试语义 | 不涉及 | 在内层会重复执行 |
| 常见错误 | 只做 Unary 漏掉 Stream | 忘记 `defer cancel` |

★ **关键认知**：客户端的 `invoker`/`streamer` 是"下一层"的调用入口。拦截器不调用它，请求就发不出去；调用两次，请求就发两次。这既是重试的实现基础，也是误用的来源。

# 5. 流式拦截器的常见需求

## 5.1 统计流的消息数

```go
type countingStream struct {
    grpc.ServerStream
    recv int
    send int
}

func (c *countingStream) RecvMsg(m any) error {
    err := c.ServerStream.RecvMsg(m)
    if err == nil {
        c.recv++
    }
    return err
}

func (c *countingStream) SendMsg(m any) error {
    err := c.ServerStream.SendMsg(m)
    if err == nil {
        c.send++
    }
    return err
}
```

## 5.2 流式 panic 恢复

```go
func RecoveryStreamInterceptor() grpc.StreamServerInterceptor {
    return func(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo,
        handler grpc.StreamHandler) (err error) {

        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic in stream %s: %v", info.FullMethod, r)
                err = status.Error(codes.Internal, "internal server error")
            }
        }()

        return handler(srv, ss)
    }
}
```

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 流式接口没鉴权 | 只注册了 Unary 拦截器 | 补 `StreamInterceptor` |
| panic 导致进程退出 | 没有恢复拦截器 | 加 `Recovery` 并放最外层 |
| 日志耗时偏大 | 日志放在重试/恢复外层 | 调整链式顺序 |
| 客户端超时没生效 | 拦截器里没 `defer cancel` | 补 cancel |
| metadata 取不到 | header 名大小写写错 | 统一小写 |
| 鉴权通过但业务取不到用户 | ctx 没往下传 | `handler(ctx, req)` 传新 ctx |
| 流式注入值无效 | 没包装 ServerStream | 覆写 `Context()` |

# 7. 练习与自检

1. 写一个服务端 Unary 拦截器，统计每个方法的 QPS 和 P99 耗时（可用简单 map + 互斥锁）。
2. 把鉴权逻辑同时接到 Unary 与 Stream 两套拦截器，验证流式接口未带 token 时返回 `Unauthenticated`。
3. 调整链式顺序，让 Recovery 放到 Logging 内层，制造一次 panic，观察日志是否记录到该请求。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 两套拦截器 | Unary 与 Stream 必须都注册，否则流式裸奔 |
| 链式顺序 | 先注册在最外层；Recovery → Logging → Auth |
| 服务端 | 鉴权、恢复、日志、限流 |
| 客户端 | 注入 token、超时、重试、埋点 |
| 流式 | 包装 `grpc.ServerStream` 才能改 ctx |
| 硬性要求 | `defer cancel`、单 goroutine Send |

下一篇进入 metadata、状态码、错误详情与超时，讲清一条请求的"上下文信息"如何跨进程传递，以及 deadline 和取消如何沿调用链传播。
