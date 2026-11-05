gRPC 的性能优势不是"用了 Protobuf 就快"，而是 HTTP/2 多路复用、二进制编码、连接复用、流式传输共同作用的结果。用错其中任何一环，性能可能还不如 REST。本篇把性能关注点、常见故障和面试高频问题集中收束，作为整个模块的收尾。

# 1. 性能关注点总览

## 1.1 一次调用的开销构成

```text
客户端
  ├── 序列化（Protobuf 编码）
  ├── 压缩（可选）
  ├── HTTP/2 帧封装、流控
  ├── 网络传输
  ▼
服务端
  ├── 解帧、解压
  ├── 反序列化
  ├── 拦截器链
  ├── 业务处理
  └── 下游依赖（DB、其他 RPC）
```

★ **关键认知**：绝大多数 gRPC 服务的瓶颈**不在 gRPC 框架本身**，而在业务逻辑、数据库和下游调用。先做 profiling 找到真正热点，再谈序列化优化。上来就换压缩算法、调大连接数，通常是在优化 5% 的部分。

## 1.2 优化优先级

| 优先级 | 关注点 | 收益 |
|--------|--------|------|
| 高 | 减少调用次数、批量化 | 数量级 |
| 高 | 连接复用、连接池 | 显著 |
| 高 | 消息大小控制 | 显著 |
| 中 | 并发流、超时设置 | 中等 |
| 中 | 压缩 | 视数据而定 |
| 低 | 序列化库替换 | 通常很小 |

# 2. 消息大小

## 2.1 默认限制

gRPC-Go 默认**单条接收消息上限 4MB**，超过会返回 `ResourceExhausted`：

```text
rpc error: code = ResourceExhausted desc = grpc: received message larger than
max (5242880 vs. 4194304)
```

调整方式：

```go
// 服务端
grpc.NewServer(
    grpc.MaxRecvMsgSize(16*1024*1024), // 接收上限 16MB
    grpc.MaxSendMsgSize(16*1024*1024), // 发送上限
)

// 客户端
grpc.NewClient(addr,
    grpc.WithDefaultCallOptions(
        grpc.MaxCallRecvMsgSize(16*1024*1024),
        grpc.MaxCallSendMsgSize(16*1024*1024),
    ),
)
```

⚠ **关键认知**：调大上限只是"让它能过"，不是"让它变快"。一条 50MB 的消息会占满 HTTP/2 流控窗口、拉长 GC 停顿、阻塞同连接上的其他流。正确做法是**分块传输**（客户端流）或**分页查询**，而不是无限放大限制。

## 2.2 控制消息大小的手段

| 手段 | 说明 |
|------|------|
| 分页 | `page_size` + `page_token`，避免一次拉全量 |
| 流式分块 | 大文件用 Client Streaming 切片 |
| 字段裁剪 | 列表接口不返回大文本、二进制 |
| 避免冗余 | 不要同时传 id 和整个对象 |
| 用 `bytes` 而非 base64 字符串 | 二进制直接传，省 33% 膨胀 |

★ **关键认知**：Protobuf 的字段名不上线，但**字段编号和值仍然占空间**。一个包含 1000 条记录的列表，即使每条只有几个字段，累积起来也可能超过 4MB。列表接口要默认分页，并且给客户端明确的 `next_page_token`。

# 3. 连接复用

## 3.1 连接是并发安全的

```go
// 正确：全局复用一个 conn
var conn *grpc.ClientConn

func init() {
    var err error
    conn, err = grpc.NewClient(addr, opts...)
    if err != nil {
        panic(err)
    }
}

func GetUser(ctx context.Context, id int64) (*pb.User, error) {
    return pb.NewUserServiceClient(conn).GetUser(ctx, &pb.GetUserRequest{Id: id})
}
```

```go
// 错误示范：每次请求新建连接
func GetUser(ctx context.Context, id int64) (*pb.User, error) {
    conn, _ := grpc.NewClient(addr, opts...) // 每次都握手、TLS、建流
    defer conn.Close()
    return pb.NewUserServiceClient(conn).GetUser(ctx, &pb.GetUserRequest{Id: id})
}
```

★ **关键认知**：`grpc.ClientConn` 是**并发安全**的，内部维护连接池、名称解析、负载均衡和重连。一个进程对一个目标通常只需要一个 `ClientConn`。每次请求 `NewClient` 会反复做 DNS、TLS 握手，延迟和资源都会爆炸。

## 3.2 什么时候需要多个连接

| 场景 | 是否需要多连接 |
|------|----------------|
| 单一后端服务 | 一个 conn 足够 |
| 多租户、不同凭据 | 按租户隔离 conn |
| 需要突破单连接流上限 | 连接池（少见） |
| 不同目标地址 | 每个目标一个 conn |

⚠ **关键认知**：HTTP/2 单连接默认允许的并发流很多（服务端可配 `MaxConcurrentStreams`），一般业务根本打不满。只有极端高并发或服务端把上限设得很低时，才需要考虑连接池。盲目建池会增加 LB 复杂度和资源占用。

## 3.3 连接泄漏

```go
// 常见泄漏：创建后没有 Close
conn, _ := grpc.NewClient(addr, opts...)
client := pb.NewUserServiceClient(conn)
// 函数结束，conn 仍然持有 goroutine 和 TCP 连接
```

用 `pprof` 看 goroutine：

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

gRPC 的连接会启动 resolver、balancer、health check 等多个 goroutine，泄漏后表现为 goroutine 数持续上涨。

# 4. 压缩

## 4.1 开启 gzip

```go
import (
    "google.golang.org/grpc/encoding/gzip"
)

// 客户端：对单次调用启用
resp, err := client.GetUser(ctx, req, grpc.UseCompressor(gzip.Name))

// 客户端：全局默认
grpc.WithDefaultCallOptions(grpc.UseCompressor(gzip.Name))
```

服务端无需额外注册，gRPC-Go 内置了 gzip。收到压缩请求会自动解压，响应是否压缩取决于服务端是否设置了 compressor。

## 4.2 压缩的取舍

| 数据类型 | 压缩收益 | 建议 |
|----------|----------|------|
| 大量重复文本、JSON、日志 | 高（5~10 倍） | 开启 |
| 图片、视频、已压缩数据 | 接近 0 | 不要开 |
| 小消息（< 1KB） | 可能负收益 | 不要开 |
| 高 CPU 敏感场景 | 需权衡 | 用更快的算法 |

★ **关键认知**：压缩是**用 CPU 换带宽**。内网高带宽、低延迟的场景，压缩往往得不偿失；跨公网、按流量计费的场景才值得。gzip 压缩率好但 CPU 高，大流量服务可以考虑 zstd 等更现代的算法（需要注册第三方 compressor）。

# 5. 并发流与流控

## 5.1 单连接多流

```text
ClientConn
  └── HTTP/2 连接
        ├── stream 1 ── RPC A
        ├── stream 2 ── RPC B
        ├── stream 3 ── RPC C
        └── ...
```

所有流共享一条 TCP 连接。HTTP/2 的流控窗口防止某一方发送过快压垮另一方。

## 5.2 服务端并发上限

```go
grpc.NewServer(
    grpc.MaxConcurrentStreams(1000), // 单连接最多 1000 个并发流
)
```

超过上限时，客户端的新流会等待，表现为延迟升高而不是直接报错。

⚠ **关键认知**：HTTP/2 解决了应用层队头阻塞，但**没有解决 TCP 层队头阻塞**。一个流的大量数据丢包，会阻塞同连接上其他流的重传。超大消息和高并发小请求混跑时，建议隔离到不同连接或不同服务。

## 5.3 批量与并发

```go
// 串行：N 次 RTT
for _, id := range ids {
    user, _ := client.GetUser(ctx, &pb.GetUserRequest{Id: id})
    users = append(users, user)
}

// 更好：批量接口，一次 RTT
resp, _ := client.BatchGetUsers(ctx, &pb.BatchGetUsersRequest{Ids: ids})
```

★ **关键认知**：减少 RTT 的最有效手段是**批量接口**，而不是把并发数调高。100 个串行请求改成 1 个批量请求，收益远大于开 100 个 goroutine。并发只是把延迟重叠，批量是直接消灭调用次数。

# 6. 常见故障

## 6.1 错误码对照

| 错误码 | 常见原因 | 排查方向 |
|--------|----------|----------|
| `Unavailable` | 连接断开、服务重启、DNS 失败 | 检查服务端存活、网络、resolver |
| `DeadlineExceeded` | 超时太短、下游慢 | 看链路耗时分布，调预算 |
| `ResourceExhausted` | 消息过大、限流、并发超限 | 查消息大小、配额、流上限 |
| `Internal` | 服务端 panic、序列化失败 | 查服务端日志、恢复拦截器 |
| `Unimplemented` | 方法未注册、版本不匹配 | 检查 proto 与服务注册 |
| `Unauthenticated` | token 缺失/过期 | 检查 metadata 与时钟 |
| `Canceled` | 上层主动取消 | 检查 cancel 调用时机 |

## 6.2 典型故障链

```text
现象：偶发 Unavailable
  │
  ├── 服务端是否重启/扩缩容？ → 看发布记录
  ├── keepalive 是否被中间设备断开？ → 看 GOAWAY 日志
  ├── 连接是否长期不重连？ → 看 resolver 与 balancer 状态
  └── 是否 LB 把流量打到已下线实例？ → 看健康检查
```

## 6.3 排查工具

```bash
# 1. 看连接与状态（服务端开启 reflection 后可用 grpcurl）
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 describe hello.v1.Greeter

# 2. 直接调一次，看错误码
grpcurl -plaintext -d '{"name":"x"}' localhost:50051 hello.v1.Greeter/SayHello

# 3. 看 goroutine / 内存
go tool pprof http://localhost:6060/debug/pprof/goroutine
go tool pprof http://localhost:6060/debug/pprof/heap

# 4. TLS 握手
openssl s_client -connect localhost:50051 -servername localhost -CAfile ca.crt
```

# 7. gRPC 与 REST、Thrift 对比

## 7.1 三方对比

| 维度 | gRPC | REST/JSON | Thrift |
|------|------|-----------|--------|
| IDL | Protobuf | OpenAPI（可选） | Thrift IDL |
| 传输 | HTTP/2 | HTTP/1.1、HTTP/2 | 自定义 TCP/HTTP |
| 编码 | Protobuf 二进制 | JSON 文本 | Binary/Compact |
| 流式 | 原生四种 | 弱 | 部分支持 |
| 浏览器 | 需 gateway/grpc-web | 原生 | 需代理 |
| 生态 | 云原生、可观测性好 | 最广 | 较老，Facebook 系 |
| 学习成本 | 中 | 低 | 中 |

★ **关键认知**：Thrift 和 gRPC 都是 IDL 驱动的 RPC，差别主要在**传输层与生态**。gRPC 押注 HTTP/2，天然获得多路复用、流控、Trailers 和代理兼容；Thrift 的传输可插拔，但跨语言生态和云原生工具链不如 gRPC。新项目除非有历史包袱，一般优先 gRPC。

## 7.2 何时不该用 gRPC

| 场景 | 更适合 |
|------|--------|
| 公开 API、第三方接入 | REST/JSON |
| 浏览器直连 | REST 或 grpc-web |
| 极简内部脚本 | 直接 HTTP |
| 需要人类可读的调试 | REST/JSON |
| 消息队列场景 | Kafka/NATS |

# 8. 面试自检

## 8.1 高频问题

**问：gRPC 为什么比 REST 快？**

答：三个层面。编码上 Protobuf 二进制比 JSON 更小、解析更快；传输上 HTTP/2 多路复用，单连接并发且头部压缩；工程上强类型 IDL 减少胶水代码和运行时校验。但要注意，业务瓶颈通常在数据库，不在协议。

**问：四种流分别怎么用？**

答：Unary 一问一答；Server Streaming 一次请求多次响应（推送、大结果集）；Client Streaming 多次请求一次响应（上传、聚合）；Bidirectional 双向实时（聊天、同步）。底层都是一个 HTTP/2 stream 上的多帧收发。

**问：拦截器链的执行顺序？**

答：`ChainUnaryInterceptor` 先注册的在最外层，请求先进后出。恢复放最外层兜 panic，日志放恢复内层记真实耗时，鉴权靠内层。流式要单独注册 Stream 拦截器，并包装 `ServerStream` 才能注入 ctx。

**问：deadline 如何传播？**

答：客户端 deadline 转成 `grpc-timeout` 头部，服务端 ctx 带同一个绝对截止时间，并继续透传给下游。每跳剩余时间递减，形成总预算，避免逐层叠加超时。

**问：gRPC 长连接怎么做负载均衡？**

答：默认 `pick_first` 会全压一个实例。要用 `dns:///` 拿到多地址并显式配 `round_robin`，配合 headless Service；或者用 Envoy/Istio 做 L7 负载均衡。K8s 默认 Service 是 L4，按连接分流，对长连接不友好。

**问：大消息怎么处理？**

答：默认接收上限 4MB。不要简单调大，而是分页、流式分块、裁剪字段。超大消息会占满流控窗口、拉长 GC、阻塞同连接其他流。

**问：如何做鉴权？**

答：传输层用 TLS/mTLS 保证链路与服务身份，应用层用 metadata 携带 JWT，在拦截器里验签、验 exp、验 iss/aud、锁定算法。认证用 `Unauthenticated`，授权用 `PermissionDenied`。

## 8.2 快速自检清单

- [ ] 能画出一次 gRPC 调用在 HTTP/2 上的帧结构
- [ ] 能解释 `grpc-status` 为什么在 Trailer 里
- [ ] 能说清 Unary 与 Stream 拦截器的签名差异
- [ ] 能解释 metadata 与 message 的边界
- [ ] 能说出 deadline 与固定超时的区别
- [ ] 能解释为什么 gRPC 需要客户端负载均衡
- [ ] 能列出至少 5 个常见错误码及排查方向

# 9. 练习与自检

1. 用 `go test -bench` 对比同一数据结构下 Protobuf 与 JSON 的序列化耗时和体积，记录结论。
2. 构造一条 5MB 的响应，观察默认限制下的错误码，再改成分块流式传输。
3. 用 pprof 观察每次请求新建 `ClientConn` 与复用连接时的 goroutine 数量差异。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 性能优先级 | 减少调用次数 > 连接复用 > 消息大小 > 压缩 |
| 消息大小 | 默认 4MB，优先分页/分块而非调大上限 |
| 连接复用 | `ClientConn` 并发安全，一个目标一个连接 |
| 压缩 | 用 CPU 换带宽，小消息和已压缩数据不要开 |
| 并发流 | HTTP/2 多路复用，但 TCP 层仍有队头阻塞 |
| 故障排查 | 先看错误码，再看连接状态、超时、日志 |
| 技术选型 | 内部服务 gRPC，对外 API REST，老系统可能 Thrift |

到这里 gRPC 模块告一段落。从 proto 契约、四种调用、拦截器、metadata 与错误、TLS 鉴权，到网关、负载均衡、测试和性能，已经覆盖了从"跑通"到"上线"的主干路径。下一步建议结合 OpenTelemetry 模块，把链路追踪和指标接入这条调用链。
