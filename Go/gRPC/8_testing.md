gRPC 的测试比 HTTP 麻烦一点：没有现成的 `httptest`，流式接口还要处理收发时序。但 Go 生态给了 `bufconn`——在内存里建一条假连接，既不占端口，也不走网卡，速度接近单元测试。本篇从最小测试骨架讲到流式接口和错误断言。

# 1. 测试分层

## 1.1 三层测试

| 层次 | 测什么 | 工具 | 速度 |
|------|--------|------|------|
| 单元测试 | 业务逻辑、参数校验 | 纯 Go + mock | 毫秒 |
| 集成测试 | 序列化、拦截器、服务注册 | `bufconn` | 毫秒级 |
| 端到端 | 真实网络、TLS、代理 | 真实端口 + 容器 | 秒级 |

```text
        ┌─────────────────────────┐
        │  E2E：真实端口 + TLS      │  少而慢
        ├─────────────────────────┤
        │  集成：bufconn 内存连接   │  主力
        ├─────────────────────────┤
        │  单元：纯函数 + mock      │  多而快
        └─────────────────────────┘
```

★ **关键认知**：大多数 gRPC 测试应该用 `bufconn`，而不是真的监听端口。它跑的是完整的 gRPC 栈（编解码、拦截器、流控），只是把 TCP 换成了内存管道。既保留了真实性，又避免了端口冲突和清理问题。

## 1.2 bufconn 是什么

`google.golang.org/grpc/test/bufconn` 提供一个实现了 `net.Listener` 的内存连接：

```text
grpc.NewClient
    │  WithContextDialer 返回 lis.DialContext 建立的连接
    ▼
┌──────────────────────────────┐
│ bufconn.Listener（内存管道）  │
└──────────────────────────────┘
    ▲
    │  Serve(lis)
grpc.Server
```

# 2. 最小测试骨架

## 2.1 测试辅助函数

```go
package server_test

import (
    "context"
    "net"
    "testing"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/test/bufconn"

    demov1 "example.com/streamdemo/gen"
)

func startTestServer(t *testing.T, register func(*grpc.Server)) *grpc.ClientConn {
    t.Helper()

    lis := bufconn.Listen(1024 * 1024) // 1MB 缓冲，够大多数测试
    s := grpc.NewServer()
    register(s) // 由调用方决定注册哪些服务

    go func() {
        if err := s.Serve(lis); err != nil && err != grpc.ErrServerStopped {
            t.Errorf("serve: %v", err) // 只报告，不要在 goroutine 里 Fatal
        }
    }()

    t.Cleanup(s.Stop) // 测试结束一定关服务端，否则 goroutine 泄漏

    conn, err := grpc.NewClient("passthrough:///bufnet",
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
            return lis.DialContext(ctx) // 不走网络，直接拿内存连接
        }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        t.Fatalf("new client: %v", err)
    }
    t.Cleanup(func() { _ = conn.Close() })

    return conn
}
```

⚠ **关键认知**：`NewClient` 的 target 必须带 `passthrough:///` 或随意一个名字，关键在 `WithContextDialer`。如果忘了这个 option，客户端会真的去解析 `bufnet` 域名，测试直接失败。

## 2.2 第一个测试

```go
func TestAdd(t *testing.T) {
    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, &demoServer{})
    })

    client := demov1.NewStreamDemoClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    resp, err := client.Add(ctx, &demov1.AddRequest{A: 2, B: 3})
    if err != nil {
        t.Fatalf("Add: %v", err)
    }
    if resp.GetSum() != 5 {
        t.Errorf("sum = %d, want 5", resp.GetSum())
    }
}
```

★ **关键认知**：集成测试里也要设超时。如果服务端逻辑死锁，没有超时的测试会一直挂到 `go test` 的全局超时（默认 10 分钟），CI 上非常难排查。

# 3. 表驱动测试

## 3.1 正常与错误一起覆盖

```go
func TestAddTable(t *testing.T) {
    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, &demoServer{})
    })
    client := demov1.NewStreamDemoClient(conn)

    tests := []struct {
        name     string
        req      *demov1.AddRequest
        wantSum  int64
        wantCode codes.Code
    }{
        {name: "正数相加", req: &demov1.AddRequest{A: 1, B: 2}, wantSum: 3, wantCode: codes.OK},
        {name: "负数相加", req: &demov1.AddRequest{A: -1, B: -2}, wantSum: -3, wantCode: codes.OK},
        {name: "零值非法", req: &demov1.AddRequest{A: 0, B: 0}, wantCode: codes.InvalidArgument},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // 每个子测试独立，可并行

            ctx, cancel := context.WithTimeout(context.Background(), time.Second)
            defer cancel()

            resp, err := client.Add(ctx, tt.req)

            if got := status.Code(err); got != tt.wantCode {
                t.Fatalf("code = %v, want %v (err=%v)", got, tt.wantCode, err)
            }
            if tt.wantCode != codes.OK {
                return // 错误用例不检查响应体
            }
            if resp.GetSum() != tt.wantSum {
                t.Errorf("sum = %d, want %d", resp.GetSum(), tt.wantSum)
            }
        })
    }
}
```

## 3.2 错误断言正确姿势

```go
// 正确：比较状态码
if status.Code(err) != codes.NotFound { ... }

// 正确：比较错误消息中的关键字（脆弱但有时必要）
st, _ := status.FromError(err)
if !strings.Contains(st.Message(), "user") { ... }

// 错误：直接比较字符串
if err.Error() != "rpc error: code = NotFound desc = user not found" { ... }
```

⚠ **关键认知**：`err.Error()` 的格式会随 gRPC 版本变化，而且包含地址、时间等不稳定信息。**断言状态码，不断言整串错误**。确实要验证消息时，用 `status.FromError` 取 `Message()` 再做包含判断。

# 4. Mock 服务端

## 4.1 用函数字段替换实现

当你要测试**客户端行为**（重试、超时、错误处理）时，不需要真实业务逻辑，构造一个可控的假服务端：

```go
type fakeGreeter struct {
    demov1.UnimplementedStreamDemoServer
    addFn func(ctx context.Context, req *demov1.AddRequest) (*demov1.AddReply, error)
}

func (f *fakeGreeter) Add(ctx context.Context, req *demov1.AddRequest) (*demov1.AddReply, error) {
    return f.addFn(ctx, req) // 把行为注入进来
}
```

```go
func TestClientHandlesUnavailable(t *testing.T) {
    fake := &fakeGreeter{
        addFn: func(ctx context.Context, req *demov1.AddRequest) (*demov1.AddReply, error) {
            return nil, status.Error(codes.Unavailable, "try later")
        },
    }

    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, fake)
    })
    client := demov1.NewStreamDemoClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    _, err := client.Add(ctx, &demov1.AddRequest{A: 1, B: 2})
    if status.Code(err) != codes.Unavailable {
        t.Fatalf("code = %v, want Unavailable", status.Code(err))
    }
}
```

★ **关键认知**：`UnimplementedXxxServer` 不只用于生产兼容，也是测试的好帮手。它给所有方法提供了返回 `Unimplemented` 的默认实现，你只需要覆写关心的那几个方法，不用实现整个接口。

## 4.2 什么时候用 mock 框架

| 场景 | 建议 |
|------|------|
| 假服务端返回固定错误 | 手写 fake struct |
| 需要断言调用次数/参数 | 手写计数或 gomock |
| 下游依赖多、行为复杂 | gomock / mockery 生成 |
| 只测序列化与拦截器 | bufconn + 真实现 |

# 5. 流式接口测试

## 5.1 服务端流

```go
func TestRange(t *testing.T) {
    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, &demoServer{})
    })
    client := demov1.NewStreamDemoClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    stream, err := client.Range(ctx, &demov1.RangeRequest{Start: 1, End: 3})
    if err != nil {
        t.Fatalf("Range: %v", err)
    }

    var got []int32
    for {
        reply, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            break // 正常结束
        }
        if err != nil {
            t.Fatalf("Recv: %v", err)
        }
        got = append(got, reply.GetValue())
    }

    want := []int32{1, 2, 3}
    if !slices.Equal(got, want) {
        t.Errorf("got %v, want %v", got, want)
    }
}
```

## 5.2 客户端流

```go
func TestUpload(t *testing.T) {
    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, &demoServer{})
    })
    client := demov1.NewStreamDemoClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    stream, err := client.Upload(ctx)
    if err != nil {
        t.Fatalf("Upload: %v", err)
    }

    for _, part := range []string{"ab", "cd", "ef"} {
        if err := stream.Send(&demov1.Chunk{Data: []byte(part)}); err != nil {
            t.Fatalf("Send: %v", err)
        }
    }

    reply, err := stream.CloseAndRecv()
    if err != nil {
        t.Fatalf("CloseAndRecv: %v", err)
    }
    if reply.GetSize() != 6 {
        t.Errorf("size = %d, want 6", reply.GetSize())
    }
}
```

## 5.3 双向流

双向流最容易写出死锁。原则是：**发送和接收分开，至少一个在 goroutine 里**。

```go
func TestChat(t *testing.T) {
    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, &demoServer{})
    })
    client := demov1.NewStreamDemoClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    stream, err := client.Chat(ctx)
    if err != nil {
        t.Fatalf("Chat: %v", err)
    }

    go func() {
        defer stream.CloseSend()
        for _, text := range []string{"a", "b"} {
            if err := stream.Send(&demov1.ChatMessage{Text: text}); err != nil {
                return // 错误留给 Recv 侧暴露
            }
        }
    }()

    var replies int
    for {
        _, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            break
        }
        if err != nil {
            t.Fatalf("Recv: %v", err)
        }
        replies++
    }

    if replies != 2 {
        t.Errorf("replies = %d, want 2", replies)
    }
}
```

⚠ **关键认知**：在测试 goroutine 里**不要调用 `t.Fatalf`**，它只会退出那个 goroutine，主测试可能永远阻塞。goroutine 内用 `t.Errorf` 或把错误发到 channel，让主 goroutine 统一判断。

# 6. 超时与取消的断言

## 6.1 验证 DeadlineExceeded

```go
func TestSlowCallTimesOut(t *testing.T) {
    slow := &fakeGreeter{
        addFn: func(ctx context.Context, req *demov1.AddRequest) (*demov1.AddReply, error) {
            select {
            case <-time.After(2 * time.Second):
                return &demov1.AddReply{Sum: 1}, nil
            case <-ctx.Done():
                return nil, status.FromContextError(ctx.Err()).Err()
            }
        },
    }

    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, slow)
    })
    client := demov1.NewStreamDemoClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()

    _, err := client.Add(ctx, &demov1.AddRequest{A: 1, B: 1})
    if status.Code(err) != codes.DeadlineExceeded {
        t.Fatalf("code = %v, want DeadlineExceeded", status.Code(err))
    }
}
```

★ **关键认知**：超时测试的时间参数要留足余量。设 100ms 超时、服务端 200ms 返回，在慢 CI 上可能因为调度延迟变成不稳定测试。原则是**超时明显短于业务耗时**（比如 100ms vs 2s），而不是卡在边界。

## 6.2 验证取消传播

```go
func TestCancelPropagates(t *testing.T) {
    cancelled := make(chan struct{})

    slow := &fakeGreeter{
        addFn: func(ctx context.Context, req *demov1.AddRequest) (*demov1.AddReply, error) {
            <-ctx.Done()
            close(cancelled) // 服务端感知到取消
            return nil, status.FromContextError(ctx.Err()).Err()
        },
    }

    conn := startTestServer(t, func(s *grpc.Server) {
        demov1.RegisterStreamDemoServer(s, slow)
    })

    ctx, cancel := context.WithCancel(context.Background())
    go func() {
        time.Sleep(100 * time.Millisecond)
        cancel()
    }()

    _, _ = demov1.NewStreamDemoClient(conn).Add(ctx, &demov1.AddRequest{A: 1})

    select {
    case <-cancelled:
    case <-time.After(time.Second):
        t.Fatal("服务端没有收到取消信号")
    }
}
```

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 测试卡住不返回 | 流没读完/服务端没停 | 加超时、`t.Cleanup(s.Stop)` |
| `bufnet` 解析失败 | 忘了 `WithContextDialer` | 补 dialer option |
| `connection refused` | target 用了真实地址 | 统一用 `passthrough:///bufnet` |
| 错误断言偶发失败 | 比较了错误字符串 | 改为比较状态码 |
| `-race` 报数据竞争 | 多 goroutine 共享切片/计数器 | 加锁或用 channel |
| goroutine 内 Fatal 后测试挂起 | `t.Fatalf` 只退出当前 goroutine | goroutine 内用 `t.Errorf` |
| 并行测试互相干扰 | 共享全局 server/端口 | 每个测试独立 bufconn |

# 8. 练习与自检

1. 给 `Upload` 写一个"发送中途出错"的测试，验证客户端能拿到对应状态码。
2. 用 `fakeGreeter` 模拟前两次 `Unavailable`、第三次成功，配合 service config 验证重试生效。
3. 给 `Chat` 测试加 `-race`，检查发送 goroutine 与接收循环之间是否有共享变量竞争。

# 9. 小结

| 主题 | 要点 |
|------|------|
| bufconn | 内存连接，跑完整 gRPC 栈，不占端口 |
| 测试骨架 | `Listen` → `Serve` → `WithContextDialer` → `Cleanup` |
| 表驱动 | 正常与错误用例放同一张表，断言状态码 |
| mock | 内嵌 `UnimplementedXxxServer`，只覆写关心的方法 |
| 流式 | `io.EOF` 判结束，收发分离防死锁 |
| 超时 | 参数留余量，避免边界抖动 |
| 断言 | 用 `status.Code`，不用整串错误字符串 |

下一篇进入性能与面试，讲清消息大小、连接复用、压缩、并发流、常见故障，以及 gRPC 与 REST/Thrift 的对比和自检问题。
