一次 gRPC 调用除了业务消息，还带着一整套"上下文"：谁在调用、还剩多少时间、失败原因是什么。这些信息不放在请求体里，而是通过 metadata、status、deadline 三条独立通道传递。搞不清这三者，就会出现"鉴权信息传丢了""超时没生效""错误码永远 Unknown"这类问题。

# 1. metadata：跨进程的键值对

## 1.1 本质与限制

**metadata** 是 gRPC 的请求头/响应头抽象，类型是：

```go
type MD map[string][]string // 一个 key 可以对应多个 value
```

关键规则：

- key 一律**小写**，HTTP/2 头部不区分大小写，gRPC 统一转小写。
- 普通 key 只能包含 `a-z 0-9 - _ .`。
- 以 `-bin` 结尾的 key 表示二进制值，传输时会做 base64 编码。
- 总大小默认受 HTTP/2 头部限制（通常 8KB 左右），不要往里塞大对象。

★ **关键认知**：metadata 适合放**轻量、跨切面**的信息：trace id、token、租户 id、灰度标记。业务数据放 message，不要塞 metadata。metadata 既不参与 schema，也没有类型检查，塞多了会变成"隐式协议"。

## 1.2 客户端发送

```go
import "google.golang.org/grpc/metadata"

// 方式一：一次性构造
md := metadata.Pairs(
    "authorization", "Bearer xxx",
    "x-request-id", "req-123",
    "x-tenant-id", "t-42",
)
ctx := metadata.NewOutgoingContext(context.Background(), md)

// 方式二：在已有 metadata 上追加（推荐，不会覆盖调用方设置的值）
ctx = metadata.AppendToOutgoingContext(ctx, "x-trace-id", "trace-9")

resp, err := client.SayHello(ctx, req)
```

⚠ **关键认知**：`NewOutgoingContext` 会**整体替换**已有 metadata，而 `AppendToOutgoingContext` 是追加。在拦截器里统一注入 token 时用后者，否则业务代码自己设的 header 会被悄悄抹掉。

## 1.3 服务端读取

```go
func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Internal, "missing metadata")
    }

    // Get 返回 []string，取第一个前要判空
    if ids := md.Get("x-request-id"); len(ids) > 0 {
        log.Println("request id:", ids[0])
    }

    return &pb.HelloReply{Message: "ok"}, nil
}
```

## 1.4 响应头与 Trailer

```go
// 服务端：在响应体之前发送 header
grpc.SetHeader(ctx, metadata.Pairs("x-server-version", "1.2.3"))
grpc.SendHeader(ctx, metadata.Pairs("x-early", "1")) // 立即发送

// 服务端：在响应之后发送 trailer
grpc.SetTrailer(ctx, metadata.Pairs("x-cost-ms", "12"))

// 客户端：Unary 调用后读取
var header, trailer metadata.MD
resp, err := client.SayHello(ctx, req, grpc.Header(&header), grpc.Trailer(&trailer))

// 客户端：流式调用结束后读取
stream.Trailer() // 必须在 Recv 返回 io.EOF 之后调用才有值
```

★ **关键认知**：Header 在响应体之前到，Trailer 在响应之后到。gRPC 的 `grpc-status` 就放在 Trailer 里，所以**流式调用必须读到流结束，才能拿到最终状态**。中途只看 `Recv` 的 error 是不够的。

# 2. 状态码：gRPC 的错误语言

## 2.1 常用状态码

| 码 | 数值 | 语义 | 典型场景 |
|----|------|------|----------|
| `OK` | 0 | 成功 | 正常返回 |
| `Canceled` | 1 | 调用方取消 | 客户端主动 cancel |
| `Unknown` | 2 | 未知错误 | 未包装的 error |
| `InvalidArgument` | 3 | 参数非法 | 校验失败 |
| `DeadlineExceeded` | 4 | 超时 | ctx 超时 |
| `NotFound` | 5 | 资源不存在 | 查不到记录 |
| `AlreadyExists` | 6 | 已存在 | 重复创建 |
| `PermissionDenied` | 7 | 无权限 | 鉴权通过但无权 |
| `ResourceExhausted` | 8 | 资源耗尽 | 限流、配额 |
| `FailedPrecondition` | 9 | 前置条件不满足 | 状态机非法流转 |
| `Aborted` | 10 | 并发冲突 | 乐观锁失败 |
| `OutOfRange` | 11 | 越界 | 分页参数越界 |
| `Unimplemented` | 12 | 未实现 | 方法未注册 |
| `Internal` | 13 | 内部错误 | panic、依赖故障 |
| `Unavailable` | 14 | 暂时不可用 | 连接断开、服务重启 |
| `DataLoss` | 15 | 数据损坏 | 解码失败 |
| `Unauthenticated` | 16 | 未认证 | token 缺失或过期 |

★ **关键认知**：区分 `Unauthenticated`（你是谁不知道）和 `PermissionDenied`（知道你是谁，但你不能做）。前者应该让客户端去登录，后者不应该。混用会导致客户端做出错误的重试或跳转行为。

## 2.2 构造与解析

```go
// 服务端：构造带状态码的错误
return nil, status.Error(codes.NotFound, "user 42 not found")

// 服务端：带格式化
return nil, status.Errorf(codes.InvalidArgument, "invalid age: %d", age)

// 客户端：解析
st, ok := status.FromError(err)
if !ok {
    // 不是 gRPC status（比如 ctx 被取消前的本地错误）
    log.Println("non-grpc error:", err)
    return
}

switch st.Code() {
case codes.NotFound:
    log.Println("用户不存在")
case codes.Unavailable:
    log.Println("服务暂时不可用，可重试")
default:
    log.Printf("code=%s msg=%s", st.Code(), st.Message())
}
```

## 2.3 错误详情

只传一个字符串不够时，用 `WithDetails` 附带结构化信息：

```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/status"
)

func validate(req *pb.CreateUserRequest) error {
    st := status.New(codes.InvalidArgument, "invalid request")

    detail := &errdetails.BadRequest{
        FieldViolations: []*errdetails.BadRequest_FieldViolation{
            {Field: "email", Description: "不是合法邮箱"},
            {Field: "age", Description: "必须大于 0"},
        },
    }

    withDetails, err := st.WithDetails(detail)
    if err != nil {
        return st.Err() // 附加失败时退回普通 status
    }
    return withDetails.Err()
}
```

客户端解析：

```go
st, _ := status.FromError(err)
for _, d := range st.Details() {
    switch info := d.(type) {
    case *errdetails.BadRequest:
        for _, v := range info.GetFieldViolations() {
            log.Printf("字段 %s: %s", v.GetField(), v.GetDescription())
        }
    case *errdetails.RetryInfo:
        log.Println("建议重试间隔：", info.GetRetryDelay())
    }
}
```

★ **关键认知**：错误详情在线上序列化成 `google.protobuf.Any`，**接收方必须有对应的 protobuf 类型才能解出来**。所以自定义错误详情要放在双方共享的 proto 里，不能只在服务端定义。

# 3. deadline 与超时

## 3.1 客户端设置

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

resp, err := client.SayHello(ctx, req)
```

gRPC 会把 deadline 转成 `grpc-timeout` 头部发给服务端，服务端拿到的 `ctx` 就带有**同一个绝对截止时间**（已扣除网络传输耗时）。

```text
客户端 ctx deadline = T
    │  grpc-timeout: 1987m（剩余毫秒）
    ▼
服务端 ctx deadline ≈ T - 网络耗时
    │  继续透传给数据库/下游 RPC
    ▼
下游 ctx deadline 更早
```

★ **关键认知**：deadline 是**绝对时间**，会沿着调用链逐跳缩短，这是它和"每跳固定超时"最大的区别。下游拿到的剩余时间是"总预算减去已经花掉的"，天然防止"每层都等 3 秒，最后 9 秒才失败"。

## 3.2 服务端处理

```go
func (s *server) Slow(ctx context.Context, req *pb.Req) (*pb.Reply, error) {
    select {
    case <-time.After(5 * time.Second):
        return &pb.Reply{}, nil
    case <-ctx.Done():
        // ctx.Err() 是 context.DeadlineExceeded 或 context.Canceled
        return nil, status.FromContextError(ctx.Err()).Err()
    }
}
```

`status.FromContextError` 会把 `context.DeadlineExceeded` 映射成 `codes.DeadlineExceeded`，把 `context.Canceled` 映射成 `codes.Canceled`。

⚠ **关键认知**：服务端**无法拒绝**一个已经超时的请求——请求可能已经到达。服务端能做的是：在耗时操作前后检查 `ctx.Done()`，并把 ctx 继续传给数据库和下游，让整条链路一起放弃。

## 3.3 超时的层次

| 层次 | 设置方式 | 作用 |
|------|----------|------|
| 单次调用 | `context.WithTimeout` | 精确控制某个 RPC |
| 客户端默认 | 拦截器统一注入 | 防止漏设超时 |
| 服务端处理 | 从 ctx 读取剩余时间 | 判断是否值得继续 |
| 连接层 | `grpc.WithConnectParams` | 建连超时、backoff |
| 空闲连接 | keepalive | 探测死连接 |

# 4. 重试

## 4.1 服务配置式重试

gRPC 内置重试通过 service config 声明，不需要写代码：

```go
const retryPolicy = `{
  "methodConfig": [{
    "name": [{"service": "demo.v1.UserService", "method": "GetUser"}],
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2,
      "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    }
  }]
}`

conn, err := grpc.NewClient(addr,
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithDefaultServiceConfig(retryPolicy),
)
```

★ **关键认知**：**只有幂等操作才能重试**。`GetUser` 可以，`CreateOrder` 不行——重试可能造成重复下单。gRPC 的重试只对 Unary 和 Server Streaming 生效，客户端流和双向流因为已经发送了部分数据，无法安全重放。

## 4.2 重试的边界

| 情况 | 是否重试 | 原因 |
|------|----------|------|
| `Unavailable` | 是 | 连接问题，换实例可能成功 |
| `DeadlineExceeded` | 谨慎 | 总预算可能已经耗尽 |
| `ResourceExhausted` | 是，带退避 | 限流，等一会再试 |
| `InvalidArgument` | 否 | 参数错误，重试多少次都一样 |
| `Internal` | 否 | 通常是代码 bug |
| `Unauthenticated` | 否 | token 问题，重试无意义 |

⚠ **关键认知**：重试要有**总预算**。`maxAttempts=4` 配合每次 1 秒超时，最坏要 4 秒。如果没有外层 deadline，重试会把延迟放大，甚至引发雪崩。重试 + 熔断 + 限流要一起考虑。

# 5. 取消传播

## 5.1 客户端取消

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    time.Sleep(500 * time.Millisecond)
    cancel() // 主动放弃
}()

_, err := client.Slow(ctx, req)
// err 的 code 是 Canceled
```

取消会通过 HTTP/2 的 `RST_STREAM` 发给服务端，服务端的 `ctx.Done()` 立即关闭。

## 5.2 服务端感知

```go
func (s *server) Process(ctx context.Context, req *pb.Req) (*pb.Reply, error) {
    for i := 0; i < 100; i++ {
        select {
        case <-ctx.Done():
            // 客户端已离开，停止无意义的计算
            return nil, status.FromContextError(ctx.Err()).Err()
        default:
        }

        doWork(i) // 每轮都检查，而不是只在开头检查一次
    }
    return &pb.Reply{}, nil
}
```

★ **关键认知**：取消传播的价值是**省资源**。客户端走了，服务端还在跑 30 秒的查询、还在写日志、还在烧 CPU，这些全是浪费。长循环、批量任务、流式处理都必须定期检查 `ctx.Done()`。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| metadata 读不到 | key 大写或用了 `NewOutgoingContext` 覆盖 | 统一小写，用 `AppendToOutgoingContext` |
| 错误码总是 `Unknown` | 直接 `errors.New` 返回 | 用 `status.Error` 包装 |
| 详情解不出来 | 客户端缺少对应 proto 类型 | 共享错误定义 |
| 超时没传到服务端 | 用了 `context.Background()` 新建 ctx | 透传调用方 ctx |
| 重试导致重复写 | 对非幂等方法开了重试 | 只重试幂等操作 |
| Trailer 为空 | 流没读完 | `Recv` 到 `io.EOF` 后再读 |
| `context canceled` 满天飞 | 上层提前 cancel | 检查 cancel 时机与 defer |

# 7. 练习与自检

1. 客户端通过 metadata 传 `x-request-id`，服务端在日志里打印，验证同一次调用两端 id 一致。
2. 服务端返回带 `BadRequest` 详情的 `InvalidArgument`，客户端解析出所有字段错误。
3. 给一个 3 秒的慢接口设 1 秒超时，观察客户端错误码，以及服务端是否在 1 秒左右收到取消。

# 8. 小结

| 主题 | 要点 |
|------|------|
| metadata | 小写 key、`[]string` 值、轻量上下文信息 |
| Header/Trailer | Header 在前、Trailer 在后，状态在 Trailer |
| 状态码 | 用 `status.Error` 构造，区分认证与授权 |
| 错误详情 | `WithDetails` + `Any`，双方要共享类型 |
| deadline | 绝对时间、逐跳缩短、沿 ctx 传播 |
| 重试 | 只对幂等操作，必须有总预算 |
| 取消 | `RST_STREAM` 通知服务端，长任务要定期检查 |

下一篇进入 TLS 与鉴权，讲清证书加载、mTLS、Token/JWT 校验、拦截器集成和密钥管理，把"能通"升级到"敢上线"。
