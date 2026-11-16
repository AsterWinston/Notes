gRPC 的性能优势不是自动获得的。消息过大、channel 滥用、线程池配置不当、背压缺失，都会把二进制协议的优势抵消掉。本篇从成本模型出发，逐项讲清消息、连接、压缩、并发、背压、压测和监控。

# 1. 性能从哪里来

## 1.1 一次调用的开销

```text
客户端
  │ 1. 序列化 protobuf
  │ 2. 写 HTTP/2 DATA 帧
  │ 3. TLS 加密（如启用）
  ▼
网络
  │ 4. 传输、路由、可能经过代理
  ▼
服务端
  │ 5. TLS 解密
  │ 6. 解析 HTTP/2 帧
  │ 7. 反序列化 protobuf
  │ 8. 业务处理（DB、缓存、下游 RPC）
  │ 9. 序列化响应
  ▼
返回客户端
```

gRPC 优化的是第 1 到 7 步：二进制编码比 JSON 紧凑，HTTP/2 多路复用减少连接数，HPACK 减少头部开销。但真正决定延迟的往往是第 8 步的业务处理，以及第 4 步的网络和代理。不要把所有性能问题都归因于 gRPC。

## 1.2 先看哪几个指标

| 指标 | 说明 | 关注点 |
|------|------|--------|
| QPS | 每秒请求数 | 容量规划 |
| P50/P95/P99 延迟 | 分位延迟 | P99 才能暴露长尾 |
| 错误率 | 按状态码分组 | `UNAVAILABLE`、`DEADLINE_EXCEEDED` |
| 在途请求数 | 当前并发 | 是否接近线程池上限 |
| 连接数 | TCP/HTTP2 连接 | 是否频繁重建 |
| CPU/内存/GC | 资源使用 | 序列化与压缩的代价 |
| 消息大小 | 平均与 P99 | 是否触发 4 MiB 限制 |

**设计考量**：只看平均延迟会掩盖长尾。gRPC 的流控、线程池排队、GC 停顿都更容易在 P99 上体现。压测报告至少要有 P95 和 P99，并注明 payload 大小与并发数。

# 2. 消息大小

## 2.1 默认 4 MiB 限制

gRPC Java 默认单条入站消息上限是 **4 MiB**。超过会返回：

```text
RESOURCE_EXHAUSTED: gRPC message exceeds maximum size 4194304
```

调整方式：

```java
// 服务端
Server server = ServerBuilder.forPort(50051)
        .addService(new UserServiceImpl())
        .maxInboundMessageSize(16 * 1024 * 1024)
        .build()
        .start();
// 客户端
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 50051)
        .usePlaintext()
        .maxInboundMessageSize(16 * 1024 * 1024)
        .build();
```

调大上限只是"让请求能过"，并没有解决大消息的代价。真正的做法是拆分或流式传输。

## 2.2 大消息的代价

| 代价 | 说明 |
|------|------|
| 内存分配 | 序列化、反序列化都会产生临时对象 |
| 拷贝 | `ByteString.copyFrom`、TLS、压缩都可能复制字节 |
| GC 压力 | 大对象进入老年代，触发 Full GC |
| 流控阻塞 | 大消息占满 HTTP/2 窗口，阻塞同连接其他流 |
| 代理限制 | Nginx、Envoy 往往有自己的 body 上限 |
| 失败重传 | 一条大消息失败，整个请求要重来 |

**注意**：`bytes` 字段在 Java 中是 `ByteString`。`ByteString.copyFrom(byte[])` 会复制一份；如果确定字节数组之后不再修改，可以用 `ByteString.wrap(byte[])` 避免拷贝，但调用方必须保证不再改动原数组。

## 2.3 拆分与流式

| 数据规模 | 建议 |
|----------|------|
| 小于 1 MiB | 普通 Unary 即可 |
| 1 到 4 MiB | 评估是否真的需要一次传完 |
| 大于 4 MiB | 分块 + Client Streaming |
| 大结果集 | Server Streaming 分批下发 |
| 文件 | 固定大小分块，例如 64 KiB 到 1 MiB |

```protobuf
service FileService {
  rpc Upload (stream Chunk) returns (UploadReply);
}
message Chunk {
  bytes data = 1;
  int64 offset = 2;   // 便于断点续传与去重
}
```

分块大小不是越小越好：块太小会增加帧头和调用次数，块太大又回到大消息问题。通常从 64 KiB 到 1 MiB 之间压测选值。

# 3. 连接复用

## 3.1 Channel 是昂贵资源

一个 `ManagedChannel` 背后可能包含：

- DNS 解析与名称解析器。
- 一条或多条 TCP 连接。
- TLS 会话。
- HTTP/2 流控窗口。
- 负载均衡策略与子 channel。
- 重试、keepalive、指标统计。

每次请求新建 channel，相当于每次重新握手、重新建连，延迟和资源都会爆炸。正确做法是**按目标复用**：

```java
// 进程级单例，注入到各个 service
@Bean(destroyMethod = "shutdown")
ManagedChannel userChannel() {
    return ManagedChannelBuilder.forTarget("dns:///user-service:50051")
            .usePlaintext()
            .defaultLoadBalancingPolicy("round_robin")
            .build();
}
```

## 3.2 多路复用与连接数

HTTP/2 允许一条连接上并发多个流。理论上一个 channel 就能承载大量并发 RPC，但实际受限于：

- 服务端 `maxConcurrentStreams`。
- 流控窗口大小。
- 单条 TCP 连接的带宽与丢包影响。
- 负载均衡需要多条连接分散到不同实例。

| 场景 | 连接策略 |
|------|----------|
| 单实例、低并发 | 一个 channel 足够 |
| 多实例、需要负载均衡 | 一个 channel + `round_robin`，由 gRPC 管理子连接 |
| 超大吞吐 | 多个 channel 分片，避免单连接窗口成为瓶颈 |
| 跨地域 | 按地域建 channel，避免跨区流量 |

**注意**：不是连接越多越好。连接过多会增加文件描述符、TLS 握手、服务端线程与内存开销，还会让负载均衡更难做。

## 3.3 Keepalive 与代理超时

长时间空闲的连接可能被 NAT、防火墙、负载均衡器静默断开。客户端下次复用时才发现在连不上，表现为间歇性的 `UNAVAILABLE`。

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress("user-service", 50051)
        .usePlaintext()
        .keepAliveTime(60, TimeUnit.SECONDS)
        .keepAliveTimeout(10, TimeUnit.SECONDS)
        .build();
```

Keepalive 间隔要小于中间设备的空闲超时，但不能太激进，否则可能被判定为滥用。服务端也可以通过 `permitKeepAliveTime` 限制客户端的最小探测间隔。

# 4. 压缩

## 4.1 开启 gzip

gRPC 支持消息级压缩。Java 默认注册了 gzip：

```java
// 客户端：只对这次调用压缩
UserReply reply = stub
        .withCompression("gzip")
        .getUser(request);
// 服务端：对响应压缩（在拦截器或方法内）
if (responseObserver instanceof ServerCallStreamObserver<?> observer) {
    ((ServerCallStreamObserver<?>) observer).setCompression("gzip");
}
```

`ServerBuilder` 与 `ManagedChannelBuilder` 都可以配置压缩器注册表，但默认已经包含 gzip，通常不需要额外设置。

## 4.2 压缩的取舍

| 维度 | 不压缩 | gzip |
|------|--------|------|
| 网络带宽 | 高 | 低 |
| CPU | 低 | 高 |
| 延迟 | 小消息更快 | 大文本更划算 |
| 已压缩数据 | 无影响 | 几乎无收益，反而浪费 CPU |

**设计考量**：压缩适合大段文本、JSON 字符串、日志；不适合图片、视频、zip 等已经压缩过的二进制。压缩比和 CPU 开销要在真实 payload 上压测，不要凭感觉开启全局压缩。

# 5. 线程池与并发

## 5.1 服务端线程模型

gRPC Java 服务端默认使用共享的缓存线程池。请求量大时线程数可能持续增长，带来上下文切换和内存开销。生产环境建议显式配置：

```java
ExecutorService businessPool = new ThreadPoolExecutor(
        16, 64,
        60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000),
        new ThreadFactoryBuilder().setNameFormat("grpc-biz-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
Server server = ServerBuilder.forPort(50051)
        .addService(new UserServiceImpl())
        .executor(businessPool)
        .build()
        .start();
```

线程池大小取决于业务是 CPU 密集还是 IO 密集：

| 业务类型 | 初始大小 | 说明 |
|----------|----------|------|
| CPU 密集 | 接近 CPU 核数 | 避免过多上下文切换 |
| IO 密集（DB/缓存） | 大于核数，按压测调 | 等待 IO 时线程可让出 |
| 混合 | 分离线程池 | 避免慢下游拖垮核心接口 |

**注意**：不要在 gRPC 业务线程里调用阻塞的下游 RPC，否则线程池会被慢下游占满，形成级联故障。异步调用或舱壁隔离更安全。

## 5.2 directExecutor 的边界

```java
Server server = ServerBuilder.forPort(50051)
        .addService(new UserServiceImpl())
        .directExecutor() // 在 gRPC 传输线程上直接执行
        .build()
        .start();
```

`directExecutor` 省掉了线程切换，适合极轻量的内存计算。但只要业务里有任何阻塞（DB、HTTP、文件、锁），就会堵住传输线程，吞吐骤降甚至死锁。除非你非常确定处理器是纯 CPU 且极快，否则不要用。

## 5.3 Java 21 的虚拟线程

Java 17 没有虚拟线程，只能靠平台线程池。升级到 Java 21 后，可以为 IO 密集型服务使用虚拟线程执行器：

```java
// Java 21+ 才能编译
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

虚拟线程适合大量阻塞 IO，但不是银弹：数据库连接池、下游限流、CPU 核数仍然是瓶颈。迁移前先确认依赖库对虚拟线程友好，避免 synchronized 导致载体线程 pinned。

# 6. 背压

## 6.1 HTTP/2 流控

HTTP/2 有连接级和流级的窗口。接收方通过 `WINDOW_UPDATE` 告诉发送方还能发多少字节。gRPC 在这层之上又提供了应用级可写性通知。

## 6.2 服务端生产数据

```java
ServerCallStreamObserver<RangeReply> observer =
        (ServerCallStreamObserver<RangeReply>) responseObserver;
observer.setOnReadyHandler(() -> {
    // 只有可写时才生产，避免消息在内存里堆积
    while (observer.isReady() && cursor <= end) {
        observer.onNext(RangeReply.newBuilder().setValue(cursor++).build());
    }
    if (cursor > end) {
        observer.onCompleted();
    }
});
```

如果生产逻辑很复杂，可以用一个有界队列加单独的生产线程，`onReady` 只负责从队列取数据。**不要**用无界队列，否则客户端消费慢时内存会持续增长。

## 6.3 客户端控制拉取

```java
ClientCallStreamObserver<RangeReply> observer =
        (ClientCallStreamObserver<RangeReply>) responseObserver;
observer.disableAutoInboundFlowControl();
observer.request(1); // 每次只拉一条
// 在 onNext 中处理完后再 request(1)，形成拉模式
```

默认情况下 gRPC 会自动请求消息，应用层感知不到背压。数据量大、处理慢时，手动流控能防止消息堆积，但会增加代码复杂度。

# 7. 压测与监控

## 7.1 工具

| 工具 | 用途 |
|------|------|
| `ghz` | gRPC 压测，支持 proto、并发、QPS、延迟分布 |
| `grpcurl` | 功能验证、反射调试 |
| `k6` | 脚本化压测，支持 gRPC |
| JMH | 序列化、反序列化微基准 |
| async-profiler | CPU/内存/锁火焰图 |
| JFR | JVM 运行时事件记录 |
| Arthas | 线上诊断、线程与调用追踪 |

`ghz` 示例：

```bash
ghz --insecure \
  --proto user.proto \
  --call user.v1.UserService/GetUser \
  -d '{"id":1}' \
  -n 10000 -c 50 \
  localhost:50051
```

参数含义：`-n` 总请求数，`-c` 并发数，`-d` 请求体。压测时要使用真实的 payload 大小和分布，否则结果没有参考价值。

## 7.2 监控指标

| 类别 | 指标 |
|------|------|
| 流量 | QPS、消息大小、并发流数 |
| 延迟 | P50/P95/P99、按方法拆分 |
| 错误 | 按 `grpc-status` 分组、超时率、取消率 |
| 资源 | CPU、内存、GC、线程池活跃数与队列长度 |
| 连接 | 连接数、重建次数、握手失败数 |
| 下游 | DB/缓存/其他 RPC 的延迟与错误 |

Spring Boot 可以接入 Micrometer + OpenTelemetry，把 gRPC 指标和 Trace 汇总到 Prometheus/Grafana。`grpc-services` 还提供 Channelz，用于查看连接与流的状态。

## 7.3 排查顺序

```text
延迟升高
  │
  ├── 先看 P99 是否集中在某个方法
  ├── 看线程池是否排队、CPU 是否饱和
  ├── 看 GC 是否有长停顿
  ├── 看下游 RPC/DB 是否变慢
  ├── 看网络是否有重传、连接是否频繁重建
  └── 最后才怀疑序列化或 gRPC 框架本身
```

**行业实践**：性能问题优先用数据定位，不要靠"感觉"。一次完整的压测应记录版本、机器规格、并发数、payload、P99 和错误率，否则优化前后无法对比。

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `RESOURCE_EXHAUSTED` | 消息超过 4 MiB | 拆分、流式或调大上限 |
| 内存持续上涨 | 无界队列或无节制 `onNext` | 有界队列 + `isReady` 背压 |
| 线程数暴涨 | 默认缓存线程池 + 阻塞业务 | 固定池、异步、隔离 |
| 吞吐上不去 | 单连接流控窗口或线程池太小 | 调窗口、加连接、扩池并压测 |
| 压缩后更慢 | 小消息或已压缩数据 | 只对大文本开启 |
| 间歇性 `UNAVAILABLE` | 空闲连接被代理断开 | 配 keepalive，匹配代理超时 |
| 连接数过多 | 每次请求新建 channel | 复用 channel |
| GC 频繁 | 大对象、频繁序列化 | 减小消息、复用 buffer、换 G1/ZGC |

# 9. 练习与自检

1. 用 `ghz` 对比 1 KiB 与 1 MiB payload 的 P99 延迟，观察消息大小对吞吐的影响。
2. 把服务端线程池设为固定 8 个线程，在方法里模拟 200ms 下游延迟，逐步增加并发，找出吞吐拐点并解释原因。
3. 给服务端流加上 `isReady` 背压，用慢客户端消费，观察服务端内存与发送速率的变化。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 消息 | 默认 4 MiB，大消息拆分或流式 |
| 连接 | channel 按目标复用，连接不是越多越好 |
| 压缩 | 大文本划算，已压缩数据无收益 |
| 线程 | 显式线程池，避免阻塞传输线程 |
| 背压 | `isReady`、`request(n)`、有界队列 |
| 压测 | 真实 payload，关注 P99 与错误码 |
| 监控 | 方法级延迟、线程池、连接、GC 一起看 |

下一篇是模块收尾，对比 Java gRPC、Go gRPC、REST 与 Dubbo，整理高频自检题和生产检查清单。
