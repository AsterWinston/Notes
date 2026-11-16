gRPC 的四种调用方式是它区别于 REST 的核心能力。很多人只会在 Unary 里打转，遇到大文件传输、实时推送、批量上报就退回 HTTP。本篇用一套完整示例把四种模式跑通，并说清楚每种模式在什么场景下才真正划算。

# 1. 四种调用方式总览

## 1.1 一张表看懂

| 模式 | 客户端发送 | 服务端返回 | 典型场景 |
|------|-----------|-----------|----------|
| Unary（一元） | 1 条消息 | 1 条消息 | 查询、创建、常规 RPC |
| Server Streaming | 1 条消息 | N 条消息 | 日志推送、大结果集、订阅 |
| Client Streaming | N 条消息 | 1 条消息 | 文件上传、批量聚合 |
| Bidirectional Streaming | N 条消息 | N 条消息 | 实时聊天、双向同步、流式处理 |

```text
Unary:        Client ──req──▶ Server ──resp──▶ Client

Server Stream: Client ──req──▶ Server ──resp1──▶
                                  ──resp2──▶
                                  ──resp3──▶ Client

Client Stream: Client ──req1──▶
                       ──req2──▶
                       ──req3──▶ Server ──resp──▶ Client

Bidi Stream:  Client ◀──resp1── Server
              Client ──req1──▶
              Client ◀──resp2── Server
              Client ──req2──▶
```

★ **关键认知**：四种模式的底层都是"一个 HTTP/2 stream 上收发多个 DATA 帧"，区别只在于**谁先结束、谁发几条**。理解这一点，你就不会把流式当成"多条独立请求"，而是"一次调用内的消息序列"。

## 1.2 示例 proto

```protobuf
syntax = "proto3";

package demo.v1;

option go_package = "example.com/streamdemo/gen;demov1";

service StreamDemo {
  rpc Add (AddRequest) returns (AddReply);                          // Unary
  rpc Range (RangeRequest) returns (stream RangeReply);             // 服务端流
  rpc Upload (stream Chunk) returns (UploadReply);                  // 客户端流
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);       // 双向流
}

message AddRequest {
  int64 a = 1;
  int64 b = 2;
}

message AddReply {
  int64 sum = 1;
}

message RangeRequest {
  int32 start = 1;
  int32 end = 2;
}

message RangeReply {
  int32 value = 1;
}

message Chunk {
  bytes data = 1;
}

message UploadReply {
  int64 size = 1;
}

message ChatMessage {
  string user = 1;
  string text = 2;
}
```

# 2. Unary：最常用的一问一答

## 2.1 服务端实现

```go
type demoServer struct {
    demov1.UnimplementedStreamDemoServer
}

func (s *demoServer) Add(ctx context.Context, req *demov1.AddRequest) (*demov1.AddReply, error) {
    // ctx 携带 deadline、metadata、取消信号，必须透传给下游
    if req.GetA() == 0 && req.GetB() == 0 {
        return nil, status.Error(codes.InvalidArgument, "两个操作数不能同时为 0")
    }
    return &demov1.AddReply{Sum: req.GetA() + req.GetB()}, nil
}
```

## 2.2 客户端调用

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.Add(ctx, &demov1.AddRequest{A: 1, B: 2})
if err != nil {
    st, ok := status.FromError(err)   // 取出 gRPC 状态码
    if ok && st.Code() == codes.InvalidArgument {
        log.Println("参数错误：", st.Message())
        return
    }
    log.Fatalf("Add: %v", err)
}
log.Println("sum =", resp.GetSum())
```

★ **关键认知**：Unary 的 `ctx` 是整条调用链的"生命线"。超时、取消、metadata 都挂在它上面。服务端**不要**把 `ctx` 存到结构体或后台 goroutine 里长期使用，请求结束后它就会被取消。

# 3. Server Streaming：一次请求，多次响应

## 3.1 服务端

```go
func (s *demoServer) Range(req *demov1.RangeRequest, stream demov1.StreamDemo_RangeServer) error {
    // 注意第二个参数是 stream，不是 ctx；ctx 通过 stream.Context() 获取
    if req.GetEnd() < req.GetStart() {
        return status.Error(codes.InvalidArgument, "end 必须大于等于 start")
    }

    for i := req.GetStart(); i <= req.GetEnd(); i++ {
        // 每次 Send 写一个消息帧；客户端可能随时取消
        if err := stream.Send(&demov1.RangeReply{Value: i}); err != nil {
            return err
        }

        // 模拟耗时，同时响应取消信号
        select {
        case <-stream.Context().Done():
            return status.FromContextError(stream.Context().Err()).Err()
        case <-time.After(10 * time.Millisecond):
        }
    }
    return nil
}
```

## 3.2 客户端

```go
stream, err := client.Range(ctx, &demov1.RangeRequest{Start: 1, End: 5})
if err != nil {
    log.Fatalf("Range: %v", err)
}

for {
    reply, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        break // 服务端正常结束，Recv 返回 io.EOF
    }
    if err != nil {
        log.Fatalf("Recv: %v", err) // 这里的 err 已经是 gRPC status
    }
    log.Println(reply.GetValue())
}
```

★ **关键认知**：服务端流的结束信号是 **`Recv()` 返回 `io.EOF`**，不是返回 `nil`。把 `io.EOF` 当错误上报是最常见的写法 bug。

## 3.3 适用场景

- 大结果集分页推送：一次请求，服务端按批 `Send`，避免一次性构造巨大响应。
- 实时订阅：日志、行情、进度条。
- 服务端处理时间长：先回部分结果，降低客户端等待焦虑。

⚠ **关键认知**：服务端流不会自动做"背压"提示，但 HTTP/2 有流控窗口。如果客户端 `Recv` 很慢，`Send` 最终会阻塞。不要在一个 `Send` 循环里无限制地生产数据，必要时用带缓冲的 channel 解耦。

# 4. Client Streaming：多次请求，一次响应

## 4.1 服务端

```go
func (s *demoServer) Upload(stream demov1.StreamDemo_UploadServer) error {
    var total int64

    for {
        chunk, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            // 客户端半关闭，返回唯一响应
            return stream.SendAndClose(&demov1.UploadReply{Size: total})
        }
        if err != nil {
            return err // 客户端取消或网络中断
        }

        total += int64(len(chunk.GetData()))
    }
}
```

## 4.2 客户端

```go
stream, err := client.Upload(ctx)
if err != nil {
    log.Fatalf("Upload: %v", err)
}

for _, part := range [][]byte{[]byte("hello "), []byte("grpc "), []byte("world")} {
    if err := stream.Send(&demov1.Chunk{Data: part}); err != nil {
        log.Fatalf("Send: %v", err)
    }
}

reply, err := stream.CloseAndRecv() // 关闭发送方向，等待服务端响应
if err != nil {
    log.Fatalf("CloseAndRecv: %v", err)
}
log.Println("received bytes =", reply.GetSize())
```

★ **关键认知**：客户端流的收尾动作是 **`CloseAndRecv()`**，它会发送 END_STREAM 并等待服务端唯一响应。如果只 `Send` 不 `CloseAndRecv`，服务端会一直等 `io.EOF`，最后被 deadline 杀掉。

## 4.3 适用场景

- 文件分块上传：边读边发，内存占用恒定。
- 批量指标上报：客户端攒批发送，服务端聚合。
- 需要"先攒后算"的场景：例如求平均、去重、聚合统计。

# 5. Bidirectional Streaming：双向实时

## 5.1 服务端（回声 + 广播式处理）

```go
func (s *demoServer) Chat(stream demov1.StreamDemo_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            return nil // 客户端关闭发送，服务端也结束
        }
        if err != nil {
            return err
        }

        // 回一条处理后的消息；收发在同一个 stream 上交错进行
        if err := stream.Send(&demov1.ChatMessage{
            User: "server",
            Text: "echo: " + msg.GetText(),
        }); err != nil {
            return err
        }
    }
}
```

## 5.2 客户端：收发分离到两个 goroutine

```go
stream, err := client.Chat(ctx)
if err != nil {
    log.Fatalf("Chat: %v", err)
}

// 发送 goroutine
go func() {
    defer stream.CloseSend() // 发送完毕，半关闭
    for _, text := range []string{"hi", "how are you", "bye"} {
        if err := stream.Send(&demov1.ChatMessage{User: "client", Text: text}); err != nil {
            log.Println("send:", err)
            return
        }
        time.Sleep(100 * time.Millisecond)
    }
}()

// 接收循环留在主 goroutine
for {
    reply, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        log.Fatalf("recv: %v", err)
    }
    log.Printf("%s: %s", reply.GetUser(), reply.GetText())
}
```

★ **关键认知**：双向流的 `Send` 和 `Recv` **必须并发进行**。gRPC 允许一个 goroutine 读、一个 goroutine 写，但**不允许两个 goroutine 同时 `Send`**，也不允许两个同时 `Recv`。否则会触发 `concurrent send` 或数据竞争。

⚠ **关键认知**：客户端取消时，服务端的 `Recv` 会返回 `context.Canceled` 包装的 status。服务端应把 `Recv` 的错误原样返回，不要吞掉，否则上层无法感知取消。

# 6. 流式调用的关键细节

## 6.1 错误从哪里来

| 位置 | 错误类型 | 处理方式 |
|------|----------|----------|
| Unary 返回 | `error` | 用 `status.FromError` 取码 |
| 服务端流 `Recv` | `io.EOF` 或 status | EOF 表示正常结束 |
| 客户端流 `Recv` | `io.EOF` 或 status | EOF 表示客户端半关闭 |
| 双向流 `Recv` | `io.EOF` 或 status | EOF 表示对端结束发送 |
| `Send` | status | 通常是对端取消或连接断开 |

## 6.2 上下文与生命周期

```text
客户端 ctx 取消
    │ 发送 RST_STREAM
    ▼
服务端 stream.Context().Done() 关闭
    │
    ├── 阻塞中的 Recv 返回 Canceled
    └── 阻塞中的 Send 返回 Canceled
```

★ **关键认知**：流式接口里**不要忽略 `stream.Context()`**。长时间运行的循环应该定期检查 `Done()`，否则客户端早已离开，服务端还在空转烧 CPU。

## 6.3 何时不该用流

| 场景 | 建议 |
|------|------|
| 简单 CRUD | 用 Unary，不要为了"高级"上流 |
| 需要跨请求恢复 | 流是有状态的，断线要重放，REST + 分页更简单 |
| 浏览器直连 | 需要 grpc-web 或网关，流式支持有限 |
| 请求之间需要独立重试 | 流内消息无法单独重试，考虑拆成 Unary |

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `Recv` 一直阻塞 | 对端没 `CloseSend`/没结束 | 确认发送方逻辑与 deadline |
| `io.EOF` 被当错误 | 未区分正常结束 | 用 `errors.Is(err, io.EOF)` |
| `Send on closed stream` | 两端都关闭后继续发 | 检查收发 goroutine 生命周期 |
| `concurrent send` | 多个 goroutine 同时 `Send` | 用单一发送 goroutine 或加锁 |
| 流被 deadline 杀掉 | 单条消息超时设置过短 | 流式调用用更长的总超时 |
| 内存暴涨 | 服务端无节制 `Send`/缓存 | 控制批次，依赖 HTTP/2 流控 |

# 8. 练习与自检

1. 实现一个 `Countdown(n)` 服务端流，从 n 递减到 0，客户端每收到一个值打印一次。
2. 实现客户端流版的"求平均值"：客户端发多条数字，服务端返回平均值。
3. 把双向流示例改成"客户端 3 秒后主动取消"，观察服务端 `Recv` 返回什么错误码。

# 9. 小结

| 模式 | 关键 API | 记住一点 |
|------|----------|----------|
| Unary | `client.Method(ctx, req)` | ctx 管超时与取消 |
| Server Streaming | `stream.Recv()` 循环 | 结束是 `io.EOF` |
| Client Streaming | `Send` + `CloseAndRecv` | 必须关闭发送方向 |
| Bidirectional | 收发生命周期分离 | 单读单写，不能并发 Send |

下一篇进入拦截器，讲清 Unary 与 Stream 两套拦截器、链式执行顺序，以及日志、鉴权、恢复、超时在客户端与服务端的差异。
