走到这里，gRPC 的契约、调用、流式、拦截器、Spring 集成、错误、安全、性能都已经过了一遍。最后一篇做三件事：横向对比 Java gRPC、Go gRPC、REST 与 Dubbo，整理高频自检题，给出一份能直接拿去用的生产检查清单。

# 1. 横向对比

## 1.1 Java gRPC 与 Go gRPC

两者共享同一套协议与 proto 契约，差异主要在语言运行时和 API 风格：

| 维度 | Java gRPC | Go gRPC |
|------|-----------|---------|
| 生成代码 | `GreeterGrpc` 内部类 | `pb.go` + `grpc.pb.go` |
| 客户端 | Blocking/Async/Future 三种 stub | 统一的 Client 接口 |
| 服务端 | 继承 `XxxImplBase` | 实现 `XxxServer` 接口 |
| 流式 API | `StreamObserver` 回调 | `stream.Recv()` / `stream.Send()` |
| 结束信号 | `onCompleted` / `onError` | `io.EOF` 表示正常结束 |
| 上下文 | `io.grpc.Context` + `Deadline` | `context.Context` |
| 拦截器 | `ClientInterceptor` / `ServerInterceptor` | Unary/Stream 函数式拦截器 |
| 并发模型 | 线程池 | goroutine |
| 错误 | `StatusRuntimeException` | `status.Error` |
| 停机 | `shutdown()` + `awaitTermination` | `GracefulStop()` |

**最容易混淆的三点**：

1. Java 的服务端流可以用阻塞 stub 返回 `Iterator`，Go 必须自己写 `Recv` 循环。
2. Java 用 `onCompleted` 表示结束，Go 用 `io.EOF`；把 EOF 当错误是 Go 侧常见 bug，Java 侧则是忘记 `onCompleted`。
3. Java 的 `StreamObserver` 不保证你调用 `onNext` 的线程安全，Go 的 `Send` 也不允许并发，但两者的错误表现不同。

## 1.2 gRPC 与 REST

| 维度 | REST/JSON | gRPC |
|------|-----------|------|
| 契约 | OpenAPI，常见后补 | `.proto`，契约先行 |
| 数据格式 | 文本 JSON | 二进制 Protobuf |
| 传输 | HTTP/1.1 或 HTTP/2 | HTTP/2 |
| 浏览器 | 原生支持 | 需要 grpc-web/网关 |
| 流式 | SSE/WebSocket 另做 | 原生四种流 |
| 缓存 | HTTP 缓存成熟 | 应用层自己做 |
| 调试 | 抓包可读 | 需要 grpcurl/反射 |
| 性能 | 一般 | 高，尤其内部服务 |
| 适用 | 开放 API、BFF、前端 | 服务间、低延迟、强类型 |

**结论**：gRPC 不是 REST 的全面替代。面向浏览器和第三方的接口，REST 的通用性更好；内部服务之间，gRPC 的类型安全、性能和流式能力更划算。生产系统常见"边缘 REST，内部 gRPC"。

## 1.3 gRPC 与 Dubbo

| 维度 | gRPC | Dubbo 3 |
|------|------|---------|
| 协议 | HTTP/2 + gRPC 帧 | Dubbo 协议 / Triple（HTTP/2） |
| 序列化 | Protobuf | Hessian2、Protobuf 等 |
| 契约 | `.proto` 跨语言 | Java 接口为主，Triple 支持 proto |
| 服务发现 | 需接 DNS/xDS/注册中心 | 内置注册中心与治理 |
| 治理 | 依赖外部组件 | 负载均衡、路由、限流、熔断内置 |
| 流式 | 原生四种流 | Triple 支持流式 |
| 跨语言 | 强 | 以 Java 为主，Triple 改善 |
| Spring 集成 | Spring gRPC | Spring Cloud Alibaba 成熟 |

**设计考量**：Dubbo 更像"服务治理框架 + RPC"，gRPC 更像"跨语言 RPC 协议 + 工具链"。如果团队已经在 Dubbo 生态里，且有成熟的注册中心、治理面板，继续用 Dubbo 3 的 Triple 协议是合理的；如果是多语言、云原生、需要和 Envoy/Kubernetes 集成，gRPC 更自然。

## 1.4 选型建议

| 场景 | 建议 |
|------|------|
| 浏览器直连、开放 API | REST/JSON |
| Java 内部服务、已有 Dubbo 治理 | Dubbo 3 Triple |
| 多语言微服务、Service Mesh | gRPC |
| 实时推送、大文件、双向流 | gRPC Streaming |
| 简单脚本、一次性调用 | REST 或 gRPC 均可，优先简单 |

# 2. 高频自检题

## 2.1 gRPC 为什么基于 HTTP/2？

1) 多路复用让一条连接并发多个 RPC，避免连接数爆炸；2) HPACK 压缩重复头部；3) Trailers 能在响应体后携带 `grpc-status`；4) 流控防止发送方压垮接收方；5) 原生 stream 支撑四种调用模式。代价是链路上每一跳都必须懂 HTTP/2，普通 HTTP/1.1 代理会直接破坏帧格式。

## 2.2 四种调用方式的本质区别是什么？

底层都是"一个 HTTP/2 stream 上收发多个 DATA 帧"，区别只在谁先结束、谁发几条。Unary 是 1 发 1 收；Server Streaming 是 1 发 N 收；Client Streaming 是 N 发 1 收；Bidirectional 是 N 发 N 收。不要把流式理解成"多条独立请求"，它是一次调用内的消息序列。

## 2.3 Java 的三种 Stub 怎么选？

BlockingStub 同步返回，适合普通业务主流程；Stub 用 `StreamObserver` 回调，适合流式和需要非阻塞的场景；FutureStub 返回 `ListenableFuture`，适合异步编排和批量扇出。三者都线程安全，可以复用。

## 2.4 StreamObserver 是线程安全的吗？

不是。gRPC 会串行化响应回调，但你调用 `requestObserver.onNext` 的线程安全由你自己负责。多个线程同时发送时必须加锁或收敛到单一发送线程。服务端流同样不能在多个线程里并发 `onNext`。

## 2.5 Deadline 和普通超时有什么区别？

Deadline 是绝对时间点，会沿调用链传播，下游可以拿到剩余时间；普通读超时只作用于某一次 IO。gRPC 用 Deadline 统一表达"这次调用最晚什么时候必须结束"，到期会取消调用并返回 `DEADLINE_EXCEEDED`。

## 2.6 为什么客户端经常收到 UNKNOWN？

最常见的原因是服务端抛了未被转换的 `RuntimeException`，gRPC 只能映射成 `UNKNOWN`。另一个常见原因是 FutureStub 的 `ExecutionException` 没有解包，直接对 `ExecutionException` 取 Status 也会得到 `UNKNOWN`。正确做法是服务端抛 `StatusRuntimeException`，客户端用 `Status.fromThrowable(e.getCause())`。

## 2.7 拦截器顺序怎么排？

服务端建议 Recovery 最外层、Logging 其次、Auth 靠内。Java 原生 API 里 `intercept` 是列表倒序执行，`interceptForward` 是正序执行，推荐后者。Spring gRPC 用 `@Order`，数值越小越靠外。

## 2.8 gRPC 如何做鉴权？

传输层用 TLS/mTLS 保证通道安全；应用层把 token 放在 Metadata 的 `authorization` 头，服务端拦截器校验后写入 `Context`，业务层从 Context 读身份。JWT 要校验签名、`exp`、`iss`、`aud`，并且必须走 TLS，否则 token 会被明文传输。

## 2.9 大文件怎么传？

用 Client Streaming 分块上传，块大小通常 64 KiB 到 1 MiB，消息里带 `offset` 便于断点续传。服务端要校验总大小、限速、做幂等键去重。不要简单调大 `maxInboundMessageSize` 后一次传几百 MB，那会带来内存、GC、流控和代理限制问题。

## 2.10 gRPC 的负载均衡怎么做？

不要每次请求新建连接。复用 `ManagedChannel`，配置 `dns:///service:port` 与 `round_robin`，让 gRPC 维护子连接；Kubernetes 环境可结合 xDS 或 Service Mesh。服务端要能感知实例变化，客户端要处理连接失败与重连退避。

## 2.11 线上出现 UNAVAILABLE 怎么排查？

按顺序查：1) 服务端是否启动、端口是否监听；2) 客户端与服务端 TLS 配置是否一致；3) DNS 是否解析到正确实例；4) 中间代理是否支持 HTTP/2 与 Trailers；5) 空闲连接是否被 NAT/网关断开，是否需要 keepalive；6) 服务端线程池是否耗尽、是否在发布重启。

## 2.12 如何做优雅停机？

先停止接收新请求，再从注册中心摘除实例，然后调用 `Server.shutdown()` 等待在途请求，超时后 `shutdownNow()`。Kubernetes 下要让 readiness 先失败、`terminationGracePeriodSeconds` 大于关闭等待时间，避免客户端看到一批 `UNAVAILABLE`。

## 2.13 重试怎样才能安全？

只重试幂等操作；创建类操作必须带幂等键；只对 `UNAVAILABLE`、`ABORTED` 等可重试状态码开启；使用指数退避加随机抖动；设置最大尝试次数与重试预算，避免重试风暴。流式调用不要依赖框架自动重试，断点续传要业务层自己做。

## 2.14 流式接口为什么收不到结束？

服务端流要调用 `onCompleted`，客户端流要调用 `onCompleted` 半关闭，双向流两端都要正确结束。任何一端只发不收尾，另一端都会一直等，直到 deadline 触发。排查时先看发送方有没有半关闭，再看 deadline 是否过长。

# 3. 生产检查清单

## 3.1 契约与代码生成

| 检查项 | 标准 |
|--------|------|
| proto 版本 | 明确 `syntax = "proto3"` |
| 包名 | 带版本号，如 `user.v1` |
| 字段编号 | 上线后不改；删除字段用 `reserved` |
| 枚举零值 | 使用 `XXX_UNSPECIFIED` |
| 兼容性 | CI 跑 `buf breaking` 或等价检查 |
| 生成代码 | 不手改，构建时重新生成 |

## 3.2 服务端

| 检查项 | 标准 |
|--------|------|
| 服务注册 | 每个 `BindableService` 都已绑定 |
| 线程池 | 显式配置，队列有界，拒绝策略明确 |
| 消息上限 | 明确 `maxInboundMessageSize` |
| 超时 | 长任务检查 `Context.isCancelled()` |
| 错误 | 业务异常转 `Status`，不泄露堆栈 |
| 停机 | `shutdown()` + 等待 + 摘流 |
| 健康检查 | 暴露 gRPC Health 服务 |

## 3.3 客户端

| 检查项 | 标准 |
|--------|------|
| Channel | 按目标复用，不随请求创建 |
| Deadline | 每次调用都设置 |
| 重试 | 只对幂等操作，配置退避 |
| 负载均衡 | 多实例场景配置 `round_robin`/xDS |
| 关闭 | 应用退出时 `shutdown` + `awaitTermination` |
| 回调 | 不在 gRPC 回调里做长时间阻塞 |

## 3.4 安全

| 检查项 | 标准 |
|--------|------|
| 传输 | 生产环境必须 TLS |
| 服务间 | 评估 mTLS 或 Service Mesh |
| Token | 短有效期、校验 `aud`、不写日志 |
| 证书 | 监控到期时间，支持轮换 |
| 私钥 | 不进 Git、不打进镜像 |
| 网关 | 确认支持 HTTP/2 与 Trailers |

## 3.5 可观测性

| 检查项 | 标准 |
|--------|------|
| 指标 | QPS、P99、错误码、在途数、线程池 |
| 日志 | 含 trace id、方法名、状态码、耗时 |
| 追踪 | gRPC 调用链接入 OpenTelemetry |
| 告警 | `UNAVAILABLE`、`DEADLINE_EXCEEDED` 突增 |
| 反射 | 测试环境开启，生产按需关闭 |

## 3.6 发布与容量

| 检查项 | 标准 |
|--------|------|
| 压测 | 真实 payload、P99、错误率 |
| 容量 | 线程池、连接数、消息大小有上限 |
| 灰度 | 新版本先小流量验证 |
| 回滚 | proto 向后兼容，旧客户端可继续调用 |
| 发布 | 先摘流再停机，避免抖动 |

# 4. 综合练习与自检

1. 画出一条 gRPC 调用从客户端 stub 到服务端业务方法的完整链路，标出序列化、HTTP/2 帧、拦截器、Context、线程池分别在哪一层。
2. 给一个已有 REST 接口设计对应的 proto，说明哪些字段需要 `optional`、哪些错误码要映射成 `Status`，以及是否需要流式。
3. 写一份上线前的检查表，至少覆盖契约、超时、TLS、线程池、停机、监控六个方面，并说明每项的验证方法。

# 5. 模块总结

| 篇目 | 主题 | 一句话 |
|------|------|--------|
| 1 | 概览与 Protobuf | 理解 RPC 定位、HTTP/2 地基与代码生成 |
| 2 | 服务端与客户端 | 跑通 ImplBase、三种 stub、连接与停机 |
| 3 | 流式模式 | 掌握四种调用与背压、取消 |
| 4 | 拦截器 | 理清顺序、鉴权、日志与 Context |
| 5 | Spring gRPC | 把服务与客户端接入 Spring 生命周期 |
| 6 | 错误与超时 | 用 Status、Deadline、重试处理失败 |
| 7 | TLS 与认证 | 配好证书、mTLS 与 JWT |
| 8 | 性能 | 控制消息、连接、线程与背压 |
| 9 | 对比与总结 | 选型、自检与生产清单 |

# 6. 小结

gRPC 的学习曲线不在"写一个 Hello World"，而在工程化细节：契约怎么演进、超时怎么传播、流怎么结束、拦截器怎么排序、证书怎么轮换、线程池怎么隔离。把这些问题逐个落实，gRPC 才会从"能跑"变成"敢上线"。

如果只记住三句话：

- **契约先行**：`.proto` 是唯一事实来源，编号一旦发布就不能改。
- **超时必设**：所有调用都要有 deadline，并且向下游传播。
- **按场景选型**：内部服务用 gRPC，浏览器与开放 API 用 REST，已有 Dubbo 治理体系可继续用 Triple。

至此，Java Network 的 gRPC 模块结束。下一步可以回到 HTTP 模块补网络基础，或进入 Spring Cloud 模块看服务发现、网关与可观测性如何与 gRPC 配合。
