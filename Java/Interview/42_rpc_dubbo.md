# 1. 知识点
- **RPC（Remote Procedure Call）**：远程过程调用，让客户端调用远程服务就像调用本地方法一样透明。核心屏蔽网络通信、序列化、协议等底层细节。
- **RPC 核心流程**：Client 通过代理（Proxy）调用 → 序列化（Serialize）参数 → 网络传输 → Server 反序列化 → 调用本地实现（Stub）→ 序列化返回值 → 网络传输 → Client 反序列化返回值。
- **Dubbo 架构**：Provider（服务提供者）暴露服务注册到 Registry，Consumer 从 Registry 订阅服务地址并调用 Provider 暴露的端口。Monitor 监控统计调用数据。Container 为服务运行容器（Spring 容器）。
- **Dubbo 协议**：
  - `dubbo://`（默认）：单长连接 + NIO 异步传输，TCP 连接复用减少握手。适合小数据量（<100K）、高并发场景。序列化默认为 hessian2。
  - `http+json://`：HTTP 协议 + JSON 序列化，适合跨语言调用。
  - `grpc://`：基于 gRPC 框架的协议。
  - `rest://`：基于 REST 风格（JAX-RS），适合对外 API。
  - `rmi://`：Java RMI 协议。
  - `hessian://`：Hessian 序列化协议。
- **注册中心**：Dubbo 支持 ZooKeeper（最常用）、Nacos、Redis、MultiCast 等。Provider 启动时将服务 URL 注册到注册中心，Consumer 订阅对应接口并注册 Watcher，服务提供者变更时通知 Consumer 刷新 Invoker 列表。
- **负载均衡**：
  - `Random`（默认）：随机分配 + 权重（Weight）。
  - `RoundRobin`：轮询分配 + 权重。
  - `LeastActive`：最少活跃调用数（最少并发），优先分配给处理速度快的 Provider。
  - `ConsistentHash`：一致性哈希（同参数同 Provider），适合含状态服务。
- **集群容错**：
  - `Failover`（默认）：失败自动切换，自动重试其他 Provider（`retries` 默认 2 次）。
  - `Failfast`：快速失败，立即报异常，适合非幂等操作。
  - `Failsafe`：安全失败，记录日志但忽略异常。
  - `Failback`：失败后台定时重试，适合通知类操作。
  - `Forking`：并行调用多个 Provider，取第一个成功返回。
  - `Broadcast`：逐个调用所有 Provider，任一失败即报告异常。
- **服务分组（Group）**：同一接口的不同实现通过 `group` 区分，Consumer 指定 `group` 调用对应实现。
- **多版本（Version）**：灰度发布手段，Provider 同时暴露多个版本（`1.0.0`、`2.0.0`），Consumer 指定版本调用，逐步迁移流量。
- **参数验证**：Controller 层 Bean Validation（JSR-303），Dubbo 端通过 `validation` 属性开启参数校验。
- **结果缓存（Cache）**：`cache=true` 开启调用结果缓存（LRU/JCache），减少重复调用。
- **泛化调用（GenericService）**：不依赖服务端接口类，动态指定服务名、方法名和参数调用远程服务。适合网关和测试工具。
- **SPI 机制**：Dubbo 的核心扩展机制，基于 JDK SPI 改进。特点：`ExtensionLoader` 加载扩展实现，`@SPI` 指定默认扩展，`@Adaptive` 动态生成适配类，`@Activate` 条件激活。支持 IOC（注入）和 AOP（包装类）。与 Java SPI 区别：Dubbo SPI 可按 Key 获取指定实现且具备 IOC 和 AOP 能力。
- **Dubbo 线程模型**：通过 Dispatcher 调度策略控制事件分发：
  - `direct`：所有方法都在 IO 线程执行（不推荐）。
  - `message`：请求/响应消息的收发只在 IO 线程，消息体处理分配业务线程池执行。
  - `execution`：只有对性能无影响的连接/断开等事件在 IO 线程执行，其他全在业务线程池。
  - `connection`：在 IO 线程上把连接断开与心跳放到线程池（5.2+）。
- **调用过程**：Consumer 代理 → 路由 Router 过滤 → 负载均衡选中 Invoker → Filter 链处理 → Netty 发送请求 → Provider Netty 接收 → 业务线程池处理 → Filter 链 → 返回结果。
- **Filter 机制**：责任链模式，Consumer 端 Filter（如 TpsLimitFilter）和 Provider 端 Filter（如 ExceptionFilter）。用户可自定义 Filter 实现调用拦截（日志、限流、鉴权等）。
- **服务导出**：`ServiceBean` 触发 `ServiceConfig.export()`，创建 Netty Server 监听端口，将服务 URL（`dubbo://ip:port/interface?parameters`）注册到注册中心。
- **服务引入**：`ReferenceBean` 创建代理对象，通过 `InvokerInvocationHandler` 代理方法调用。Consumer 启动 Netty Client 连接 Provider，向注册中心订阅服务地址。
- **服务降级**：`mock` 参数强制返回 null（`force:return null`）或失败后 Mock（`fail:return null`），在 Provider 不可用时兜底。
- **Dubbo 3.x 新特性**：
  - **应用级服务发现**：从接口级注册切换到应用级注册，减少注册数据量和连接数。
  - **Triple 协议**：基于 HTTP/2 和 gRPC 构建，支持流式调用、HTTP1 兼容、跨语言通信。
  - **云原生**：适配 Kubernetes（Service 映射、Proxyless 服务网格）。
  - **Spring Cloud 适配**：Dubbo 作为 Spring Cloud 体系的服务调用方案。
- **gRPC**：Google 开源的高性能 RPC 框架，基于 HTTP/2（支持多路复用和双向流），使用 Protocol Buffers 作为接口定义语言和序列化协议。
- **gRPC 四种模式**：
  - **Unary RPC（一元）**：客户端单请求 → 服务端单响应（传统 RPC）。
  - **Server Streaming RPC**：客户端单请求 → 服务端流式多响应。
  - **Client Streaming RPC**：客户端流式多请求 → 服务端单响应。
  - **Bidirectional Streaming RPC**：双向流，客户端和服务端独立异步流式读写。
- **Protocol Buffers（Protobuf）**：二进制序列化协议，文件 `.proto` 定义消息格式和 RPC 方法，通过 `protoc` 生成客户端 Stub 和服务端骨架代码。
- **Feign vs Dubbo**：Feign 是声明式 HTTP 客户端（RestTemplate 封装），适合对外 API 调用和跨语言交互。Dubbo 是 TCP 直连高性能 RPC，适合内部微服务间高效通信。Dubbo 比 Feign 性能更高但对外不友好（需 Dubbo 客户端）。

# 2. 面试题

## 2.1 什么是 RPC？RPC 的核心流程是怎样的？
RPC（Remote Procedure Call）是远程过程调用协议，目的是让客户端调用远程服务像调用本地方法一样透明。核心流程：
1. Client 调用本地代理（Proxy/Stub），封装方法名、参数等。
2. 序列化参数为二进制数据。
3. 通过网络协议发送请求到 Server。
4. Server 收到请求后反序列化，找到对应的本地实现方法并执行。
5. 序列化返回值发回 Client。
6. Client 反序列化返回结果给调用方。

## 2.2 Dubbo 的架构包含哪些核心组件？
- **Provider**：暴露服务的服务提供方，启动注册到 Registry。
- **Consumer**：调用远程服务的使用方，从 Registry 订阅服务地址。
- **Registry**：服务注册与发现中心（ZooKeeper/Nacos/Redis），记录 Provider 地址信息。
- **Monitor**：监控中心，统计调用次数、耗时等，Provider 和 Consumer 定时上报。
- **Container**：服务运行容器（Spring 容器），负责 Provider 的生命周期管理。

## 2.3 Dubbo 的负载均衡策略有哪些？
1. **Random（随机 + 权重）**：默认策略，按照权重比例随机分配。权重越大被分到概率越高。适合通常情况。
2. **RoundRobin（轮询 + 权重）**：按权重轮询分配，循环调用 Provider。
3. **LeastActive（最少活跃调用数）**：统计每个 Provider 正在处理的请求数（活跃数），活跃数越小代表处理能力越好，优先分配。请求耗时不均场景高效。
4. **ConsistentHash（一致性哈希）**：同参数的请求总是发到同一 Provider，适合需要请求粘性的状态服务。

## 2.4 Dubbo 的集群容错策略有哪些？
- **Failover（失败自动切换）**：默认策略，失败后重试其他 Provider，可设 `retries` 重试次数。适用于幂等操作。
- **Failfast（快速失败）**：失败后立即报错，不重试。适用于非幂等操作（如扣款）。
- **Failsafe（安全失败）**：失败后记录日志忽略异常。适用于日志记录等不影响核心逻辑的操作。
- **Failback（失败定时重试）**：失败后进入重试队列，定时后台重试。适用于通知类操作。
- **Forking（并行调用）**：并行调用多个 Provider，取第一个成功返回的结果。适合实时性极高场景，浪费资源。
- **Broadcast（广播调用）**：逐个调用所有 Provider，任一失败即报异常。适用于通知所有 Provider 的配置更新等场景。

## 2.5 Dubbo 的 SPI 机制和 Java SPI 有什么区别？
| | Java SPI | Dubbo SPI |
|------|----------|-----------|
| 配置文件位置 | `META-INF/services/接口名` | `META-INF/dubbo/`、`META-INF/dubbo/internal/`等 |
| 配置文件格式 | 全限定类名列表 | Key=全限定类名（Key 可标识扩展名） |
| 加载方式 | `ServiceLoader` 加载所有实现 | `ExtensionLoader.getExtensionLoader()` |
| 按需加载 | 加载全部实现，不支持按 Key 获取 | 支持按 Key 获取指定实现 |
| IOC / AOP | 不支持 | 支持（注入其他扩展、包装类 AOP） |
| @SPI | 无 | `@SPI` 指定默认扩展名 |
| @Adaptive | 无 | `@Adaptive` 动态生成适配类 |
| @Activate | 无 | `@Activate` 条件激活（如指定 group 时激活） |

## 2.6 Dubbo 的调用过程是怎样的？
1. Consumer 端通过代理（`InvokerInvocationHandler`）拦截方法调用。
2. 通过路由器（Router）过滤出符合条件的 Provider 列表。
3. 负载均衡（LoadBalance）从中选出目标 Provider 的 Invoker。
4. 执行 Consumer Filter 链（责任链模式，如 MonitorFilter、TpsLimiter 等）。
5. 通过 Netty Client 发送序列化后的请求到目标 Provider。
6. Provider 的 Netty Server 接收请求，根据 Dispatcher 策略分派到业务线程池。
7. 执行 Provider Filter 链。
8. 通过反射调用服务实现类的方法。
9. 结果经过 Filter 链 → 序列化 → Netty 返回给 Consumer。
10. Consumer 反序列化结果，经过 Consumer Filter 链返回给调用方。

## 2.7 Dubbo 3.x 的主要新特性是什么？
1. **应用级服务发现**：服务注册粒度由接口级改为应用级，减少注册中心数据量和连接数。接口与 IP 映射存储在元数据中心。
2. **Triple 协议**：基于 HTTP/2 和 gRPC 的新协议，支持流式调用（Server Stream/Client Stream/Bidirectional Stream）、HTTP/1 兼容、跨语言互操作。
3. **云原生支持**：Kubernetes Service 映射为 Dubbo Service，支持 Proxyless Service Mesh（无 Sidecar 模式）。
4. **多协议支持**：`dubbo://`、`triple://`、`rest://` 在同一端口共存。

## 2.8 gRPC 是什么？有哪些调用模式？
gRPC 是 Google 的高性能开源 RPC 框架，基于 HTTP/2（多路复用、双向流、头部压缩），使用 Protocol Buffers 作为 IDL 和序列化协议。

四种调用模式：
1. **Unary RPC（一元）**：客户端一个请求，服务端一个响应。`rpc Method(Request) returns (Response);`
2. **Server Streaming RPC**：客户端一个请求，服务端返回多个响应的流。`rpc Method(Request) returns (stream Response);`
3. **Client Streaming RPC**：客户端发送多个请求的流，服务端汇总后返回一个响应。`rpc Method(stream Request) returns (Response);`
4. **Bidirectional Streaming RPC**：客户端和服务端各自独立流式读写。`rpc Method(stream Request) returns (stream Response);`

## 2.9 Dubbo 和 gRPC、Spring Cloud Feign 有什么区别？
| 维度 | Dubbo | gRPC | Feign |
|------|-------|------|-------|
| 协议 | TCP 自定义协议 + HTTP/2（Triple） | HTTP/2 | HTTP/1.x |
| 序列化 | Hessian2 / Protobuf（Triple） | Protocol Buffers | JSON / XML |
| IDL | Java Interface | .proto 文件 | Java Interface + 注解 |
| 跨语言 | Triple 协议支持 | 原生支持多种语言 | HTTP 通用（跨语言但需实现 REST） |
| 性能 | 高（TCP 直连 / HTTP2） | 高（HTTP/2 + Protobuf） | 较低（HTTP/1.x + JSON） |
| 服务治理 | 丰富（Docker 注册/负载/容错/限流） | 需额外组件 | 依赖 Spring Cloud 生态 |
| 适用场景 | 内部微服务高效调用 | 跨语言高性能微服务 | 对外 API 调用 / 内部简单 REST 调用 |

## 2.10 Dubbo 的多版本（Version）和服务分组（Group）有什么用？
- **Version（版本）**：灰度发布和服务升级。同时部署 `1.0.0` 和 `2.0.0` 版本的 Provider，Consumer 通过 `version="2.0.0"` 指定消费新版本服务，逐步迁移流量验证新版本稳定性，验证完成后就可以去掉旧版本。
- **Group（服务分组）**：同一接口不同实现（互斥环境的多实例），如 `group="prod"` 和 `group="test"`。Consumer 通过指定 `group` 区分调用哪个实现的服务。

## 2.11 Dubbo 的 Filter 机制是什么？有哪些常用 Filter？
Filter 是 Dubbo 的过滤拦截器链，基于责任链模式在 Provider 端和 Consumer 端提供拦截调用处理的能力。用户可实现 `Filter` 接口自定义拦截。
- **Consumer 端 Filter**：`ConsumerContextFilter`（传递隐式参数）、`ActiveLimitFilter`（客户端限流）、`TpsLimitFilter`（TPS 限制）。
- **Provider 端 Filter**：`ExceptionFilter`（异常处理）、`AccessLogFilter`（访问日志）、`ExecuteLimitFilter`（服务端并发限制）、`TokenFilter`（Token 鉴权）。
- 通过 META-INF/dubbo/ 下 SPI 配置文件注册，按 `@Activate` 条件激活。

## 2.12 什么是 Dubbo 的泛化调用（GenericService）？
泛化调用是在不依赖服务接口类（.class 文件）的情况下进行 RPC 调用。Consumer 不引入 Provider 接口包，直接通过 `GenericService` 接口，传入服务名 + 方法名 + 参数类型数组 + 参数值数组进行调用。典型场景：Dubbo Admin 测试服务、API 网关。

```java
GenericService genericService = (GenericService) reference.get();
Object result = genericService.$invoke("sayHello", new String[]{"java.lang.String"}, new Object[]{"world"});
```

## 2.13 Dubbo 的超时和重试机制是怎样的？
- **超时（timeout）**：Consumer 调用 Provider 的超时时间，默认 1000ms。超时后抛出 `TimeoutException`。可在服务级或方法级配置 `timeout` 参数。
- **重试（retries）**：`Failover` 集群容错模式下的失败重试次数，默认 2 次（含首次调用共 3 次）。其他容错模式（Failfast/Failsafe 等）不重试。重试次数过多可能放大故障，建议非幂等操作设为 0。

## 2.14 Dubbo 使用 ZooKeeper 作为注册中心的流程？
1. Provider 启动后，`ServiceConfig.export()` 将服务 URL 注册到 ZK 目录 `/dubbo/com.xxx.xxxInterface/providers/dubbo://ip:port/...`（创建临时节点）。
2. Consumer 启动时，`ReferenceConfig.get()` 订阅 ZK 目录 `/dubbo/com.xxx.xxxInterface/providers/` 并注册 Watcher。
3. 当 Provider 上下线时 ZK 子节点变化，Watcher 通知 Consumer 刷新本地 Invoker 缓存。
4. Consumer 通过负载均衡选择一个 Provider 发起 Netty 直连调用（不经过 ZK）。
5. Monitor（若配置）则为单独的统计中心，定时上报调用数据。

## 2.15 如何实现 Dubbo 的 Mock 服务降级？
通过 `mock` 参数实现：
- `mock=true`：使用默认 Mock 类（接口名 + Mock 后缀，需在同包下实现）。
- `mock=com.xxx.MockXxxService`：指定 Mock 实现类。
- `mock=force:return null`：强制 Mock 所有调用返回 null（不发起远程调用抛异常也 Mock）。
- `mock=fail:return null`：只在远程调用失败后 Mock 返回 null。

用于 Provider 不可用时保障核心链路不中断。
