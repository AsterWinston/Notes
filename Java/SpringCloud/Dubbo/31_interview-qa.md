
# 1. 基础概念

## 1.1 什么是 RPC？与 HTTP 有什么区别？

**RPC（Remote Procedure Call）** 是一种进程间通信方式，允许像调用本地方法一样调用远程服务。

| 维度 | RPC（Dubbo） | HTTP（Spring Cloud Feign） |
|------|-------------|---------------------------|
| 传输协议 | TCP 长连接，二进制协议 | HTTP 1.1/2，文本协议 |
| 序列化 | Hessian2/Protobuf/Kryo | JSON/XML |
| 性能 | 高（二进制，IO 多路复用） | 中（HTTP 报文头开销大） |
| 服务治理 | 内置丰富（路由/限流/降级） | 需借助额外组件 |
| 跨语言 | Java 为主 | 天然多语言 |
| 适用场景 | 内部微服务高频调用 | 对外 API、跨语言调用、BFF |

## 1.2 Dubbo 与 Spring Cloud 的关系？

Dubbo 和 Spring Cloud **不是替代关系，而是互补关系**：

- Spring Cloud 是微服务**全家桶生态**（网关、配置、调用、熔断、追踪）
- Dubbo 是 Spring Cloud 体系中的 **RPC 通信组件**，可替换 Feign
- Dubbo 通过 Spring Cloud Alibaba 融入 Spring Cloud 生态

```
Spring Cloud 生态
├── 注册中心: Nacos / Eureka / Consul
├── RPC 通信: Feign (HTTP) / Dubbo (RPC)     ← Dubbo 在这里
├── 网关: Gateway / Zuul
├── 熔断: Sentinel / Resilience4j
├── 配置: Nacos Config / Spring Cloud Config
└── 追踪: SkyWalking / Zipkin
```

## 1.3 Dubbo 支持哪些注册中心？

| 注册中心 | 生产推荐度 | 特点 |
|---------|-----------|------|
| **Nacos** | ★★★★★ | 阿里出品，同时支持注册和配置中心，功能全面 |
| **Zookeeper** | ★★★★☆ | 成熟稳定，CP 模型，大规模场景需优化 |
| **Redis** | ★★☆☆☆ | 仅适用于轻量场景 |
| **Kubernetes API** | ★★★★☆ | 云原生场景下推荐，无需额外部署注册中心 |
| **Multicast** | ☆☆☆☆☆ | 测试环境，不可用于生产 |

## 1.4 Dubbo 的负载均衡策略有哪些？

| 策略 | 全限定类名 | 原理 | 适用场景 |
|------|----------|------|---------|
| **Random** | `RandomLoadBalance` | 加权随机 | 通用场景（**默认**） |
| **RoundRobin** | `RoundRobinLoadBalance` | 加权轮询 | 请求量均匀分布 |
| **LeastActive** | `LeastActiveLoadBalance` | 最小活跃调用数 | 长连接、耗时差异大 |
| **ConsistentHash** | `ConsistentHashLoadBalance` | 一致性哈希 | 有状态服务、缓存命中 |
| **ShortestResponse** | `ShortestResponseLoadBalance` | 最短响应时间 | 异构集群、性能差异大 |

## 1.5 Dubbo 的集群容错策略有哪些？

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **Failover** | 失败自动重试其他节点（**默认**） | 幂等读操作 |
| **Failfast** | 快速失败，立即报错 | 非幂等写操作 |
| **Failsafe** | 失败直接忽略，返回空 | 日志/监控等旁路操作 |
| **Failback** | 失败后台定时重试 | 消息通知等最终一致性 |
| **Forking** | 并行调用多个节点，取最快响应 | 实时性要求极高的读 |
| **Broadcast** | 逐个调用所有节点，任一失败即失败 | 缓存刷新、配置更新 |

# 2. 核心原理

## 2.1 描述 Dubbo 的服务导出流程

Provider 启动时的服务导出流程（以 Dubbo 协议为例）：

```
1. ServiceConfig.doExport()
   ├── 检查/组装配置参数
   └── doExportUrls()
       │
       ├── loadRegistries() — 获取注册中心地址列表
       │
       └── 遍历每个协议 × 每个注册中心
           └── doExportUrlsFor1Protocol()
               │
               ├── 组装 Dubbo URL（携带所有配置参数）
               │
               ├── proxyFactory.getInvoker(ref, interface, url)
               │   └── 通过 Javassist/JDK 动态代理包装 ServiceImpl
               │       → 生成 AbstractProxyInvoker
               │
               └── protocol.export(invoker)
                   │
                   ├── RegistryProtocol.export()
                   │   ├── doLocalExport() — 本地暴露
                   │   │   └── DubboProtocol.export()
                   │   │       ├── openServer() — 启动 Netty Server
                   │   │       └── exporterMap.put(key, exporter)
                   │   │
                   │   ├── register() — 注册到注册中心
                   │   │   └── registry.register(url)
                   │   │
                   │   └── subscribe() — 订阅配置覆盖
                   │
                   └── 返回 Exporter（持有 invoker 和 unexport 回调）
```

**关键点**：
- Provider 先本地暴露（启动 Netty Server 监听端口），再注册到注册中心
- 遍历每个协议和每个注册中心的笛卡尔积
- URL 作为配置总线贯穿全程

## 2.2 描述 Dubbo 的服务引用流程

Consumer 端的服务引用流程：

```
1. ReferenceConfig.get()
   │
   ├── 检查/组装配置参数
   │
   └── createProxy()
       │
       ├── 组装 Dubbo URL（携带 Consumer 端配置）
       │
       └── protocol.refer(interface, urls)
           │
           └── RegistryProtocol.refer()
               │
               ├── doRefer() — 创建 ClusterInvoker
               │   ├── 创建 RegistryDirectory（服务目录）
               │   │   └── 订阅注册中心，获取 Provider URL 列表
               │   │
               │   └── cluster.join(directory)
               │       └── 将 Directory + Router + LoadBalance 组装成 ClusterInvoker
               │
               └── proxyFactory.getProxy(invoker)
                   └── 动态代理生成接口实现
                       └── 调用时 → InvokerInvocationHandler.invoke()
                           → MockClusterInvoker → ClusterInvoker
                           → Directory.list() → Router → LoadBalance
                           → 选定 Provider → 发起 RPC 调用
```

**关键点**：
- Consumer 端通过 `RegistryDirectory` 订阅注册中心变更
- `ClusterInvoker` 封装了 Directory、Router、LoadBalance 的组合决策
- 每次 RPC 调用时执行路由和负载均衡（而非启动时固化）

## 2.3 Dubbo SPI 与 Java SPI 的核心区别？

| 特性 | Java SPI | Dubbo SPI |
|------|---------|-----------|
| 配置文件位置 | `META-INF/services/` | `META-INF/dubbo/`、`META-INF/dubbo/internal/` |
| 加载方式 | 全量加载（所有实现类都实例化） | **按需加载**（使用时才实例化） |
| 默认实现 | 不支持 | `@SPI("default")` 指定默认扩展名 |
| IoC 注入 | 不支持 | 支持扩展点自动注入 |
| AOP 增强 | 不支持 | `Wrapper` 类自动包装（装饰器模式） |
| 自适应扩展 | 不支持 | `@Adaptive` 动态生成适配类 |
| 扩展激活 | 不支持 | `@Activate` 按条件（group/value）自动激活 |

**源码核心类**：`ExtensionLoader<T>`，位于 `org.apache.dubbo.common.extension` 包。

## 2.4 Invoker 模型是什么？为什么这样设计？

Invoker 是 Dubbo 中**最核心的领域模型**，代表一个"可执行的对象"：

```java
public interface Invoker<T> {
    Class<T> getInterface();
    Result invoke(Invocation invocation) throws RpcException;
    URL getUrl();
    boolean isAvailable();
    void destroy();
}
```

**设计的核心价值**：

1. **统一抽象**：无论是本地执行还是远程调用，对外都表现为 `Invoker`
2. **组合模式**：Filter 链、Cluster、Router 都通过包装 Invoker 实现
3. **责任链解耦**：每个 Invoker 只关注自己的职责，通过链式组合实现复杂功能

```
Proxy (用户看到的接口)
  └── InvokerInvocationHandler
      └── MockClusterInvoker
          └── ClusterInvoker (包含 Directory/Router/LoadBalance)
              └── 经过 Filter 链
                  └── DubboInvoker
                      └── ExchangeClient
                          └── Netty Channel
```

## 2.5 Dubbo URL 为什么是配置总线的核心？

Dubbo 使用 `org.apache.dubbo.common.URL`（非 HTTP URL）作为配置的传递载体：

```
dubbo://192.168.1.100:20880/com.example.UserService
  ?timeout=3000
  &version=1.0.0
  &loadbalance=random
  &retries=2
  &side=provider
```

**设计理由**：

1. **不可变对象**：线程安全，天然适合多线程场景
2. **统一参数格式**：所有配置通过 key-value 对表达，易于解析和扩展
3. **全生命周期闭环**：
   - Provider 导出时生成 URL → 注册中心存储 URL
   - Consumer 发现时拉取 URL → Router/LoadBalance 消费 URL
   - RPC 调用时携带 URL → Provider 端解析 URL 参数

## 2.6 Dubbo Filter 链的执行机制是什么？

Filter 链基于**责任链模式 + 装饰器模式**实现：

```java
// 构建伪代码（实际由 ExtensionLoader 自动生成）
Invoker<T> filterChain = new Filter1(
    new Filter2(
        new Filter3(
            actualInvoker
        )
    )
);

// 调用时层层传递
filterChain.invoke(invocation);
// → Filter1.invoke() → Filter2.invoke() → Filter3.invoke() → actualInvoker.invoke()
```

**内置核心 Filter**：

| Filter | 端 | 职责 |
|--------|----|------|
| `MonitorFilter` | Consumer/Provider | 上报调用次数和耗时 |
| `TimeoutFilter` | Provider | 记录超时日志 |
| `ExceptionFilter` | Provider | 异常包装 |
| `TokenFilter` | Provider | 服务鉴权 token 验证 |
| `ActiveLimitFilter` | Consumer | 限制并发调用数 |
| `TpsLimitFilter` | Provider | 限制每秒调用次数 |
| `ExecuteLimitFilter` | Provider | 限制并发执行数 |
| `ContextFilter` | Consumer | RpcContext 上下文传递 |

# 3. 工程实践

## 3.1 超时与重试如何配置？为什么要谨慎使用重试？

```yaml
dubbo:
  provider:
    timeout: 3000      # Provider 端超时
    retries: 0         # Provider 端不重试（避免重复执行）
  consumer:
    timeout: 3000      # Consumer 端超时（实际生效值）
    retries: 2         # Consumer 端重试次数
```

**重试风险**：
- 非幂等写操作重试会导致**数据重复**（如扣款、下单）
- 超时重试 + 数据库事务可能导致**雪崩**
- 建议：写操作设置 `retries=0`，读操作可设置 `retries=1~2`

**超时传播**：`Consumer timeout > Provider timeout > 实际方法执行时间`

## 3.2 Dubbo 服务的版本号与分组如何设计？

```yaml
dubbo:
  provider:
    version: 2.0.0           # 服务版本
    group: stable            # 服务分组
```

**版本号策略**：

| 版本 | 含义 | 场景 |
|------|------|------|
| `1.0.0` | 正式版 | 生产环境 |
| `2.0.0-SNAPSHOT` | 开发版 | 联调环境 |
| `*` | 通配符 | Consumer 消费任意版本 |

**分组策略**：

```
group: prod      → 正式环境
group: dev       → 开发环境
group: stable    → 稳定节点
group: canary    → 金丝雀节点
```

## 3.3 如何实现灰度发布与流量染色？

方案一：基于版本号升级：

```yaml
# 发布 v2 版本到灰度节点
dubbo:
  provider:
    version: 2.0.0

# Consumer 切换到 v2
dubbo:
  consumer:
    version: 2.0.0
```

方案二：基于路由规则 + 参数：

```yaml
# Dubbo Admin 创建条件路由规则
# 请求头 env=gray → 路由到灰度节点
enabled: true
conditions:
  - "method=getUser => env=gray"
  - "env=gray => 192.168.1.200:20880"  # 灰度节点
```

## 3.4 大接口设计问题如何解决？

**问题**：单个服务接口方法过多（如 100+ 方法），导致注册数据膨胀、版本管理困难。

**解决方案**：
1. **接口拆分**：按业务领域拆分（UserService、UserAuthService、UserProfileService）
2. **版本隔离**：不同版本独立部署
3. **泛化调用替代**：对不稳定的接口使用泛化调用，避免强依赖

## 3.5 Dubbo 服务如何做到优雅上下线？

```yaml
dubbo:
  application:
    shutdown-wait-time: 10000   # 优雅关闭等待时间(ms)
    register-mode: instance

  provider:
    warmup: 120000              # 预热时间(ms)，期间降低权重
```

**上线流程**：
1. Provider 启动 → 延迟注册（warmup）
2. 预热期间权重逐步从 0 升至 100%
3. 预热完成 → 全权重接收流量

**下线流程**：
1. 从注册中心反注册（Consumer 停止路由新请求到本机）
2. 等待 inflight 请求处理完成（shutdown-wait-time）
3. 关闭 Netty Server → 关闭 Spring 容器

## 3.6 如何排查 Dubbo 调用超时？

```
排查顺序：
1. Provider 日志 → 确认请求是否到达
2. Provider 线程池 → 是否满（active == max）
3. 业务方法耗时 → 是否有慢查询/慢缓存
4. GC 日志 → 是否有长时间 Full GC
5. 网络延迟 → ping/traceroute Provider
6. Consumer 端超时设置 → 是否过短

常用命令：
# Provider 线程池状态
curl http://localhost:22222/threadpool

# JVM 状态
curl http://localhost:22222/jvm

# 查看服务列表
curl http://localhost:22222/ls
```

# 4. 架构设计

## 4.1 为什么 Dubbo 采用微内核 + 插件化架构？

**核心思想**：框架内核保持最小化，所有功能模块通过 SPI 扩展实现。

**收益**：
1. **可扩展性**：新增协议/注册中心/序列化方式只需实现 SPI 接口
2. **按需加载**：只实例化依赖的扩展点，减少内存和启动开销
3. **隔离性**：扩展点之间无硬编码依赖，替换实现无需修改核心代码
4. **易测试**：每个扩展点可独立单元测试

```
Dubbo Core (Microkernel)
├── SPI: Protocol      → DubboProtocol / TripleProtocol / HttpProtocol
├── SPI: Registry      → NacosRegistry / ZookeeperRegistry / K8sRegistry
├── SPI: Serialization → Hessian2 / FastJSON2 / Kryo / Protobuf
├── SPI: LoadBalance   → Random / RoundRobin / LeastActive / ConsistentHash
├── SPI: Filter        → MonitorFilter / TimeoutFilter / TokenFilter / ...
└── SPI: Router        → ConditionRouter / TagRouter / MeshRouter / ...
```

## 4.2 Dubbo URL 作为配置总线的优缺点？

**优点**：
- 全生命周期统一的数据载体
- 天然支持 SPI 扩展（通过参数选择具体实现）
- 不可变对象，线程安全

**缺点**：
- URL 字符串膨胀（参数过多时可达数百字符）
- 缺少强类型约束（参数解析依赖字符串）
- 不适合大数据量传输（应使用 MetadataService）

## 4.3 接口级发现 vs 应用级发现的权衡？

| 维度 | 接口级发现（2.x 默认） | 应用级发现（3.x 默认） |
|------|----------------------|----------------------|
| 注册粒度 | 每个接口一个注册节点 | 每个应用一个注册节点 |
| 注册中心压力 | 大（接口数 × 实例数） | 小（仅实例数） |
| 推送效率 | 低（变更触发全量重算） | 高（仅通知应用变更） |
| 元数据存储 | URL 内嵌 | 元数据中心（Nacos/ZK） |
| 扩展性 | 差（不支持多协议） | 好（服务与实例解耦） |
| 迁移成本 | — | 需配置 metadata-report |

## 4.4 Triple 协议相比 Dubbo 协议的优势？

| 特性 | Dubbo 协议 | Triple 协议 |
|------|-----------|------------|
| 协议基础 | 自定义二进制协议 | 基于 HTTP/2 + gRPC |
| 跨语言 | 有限（需各语言实现 Dubbo SDK） | 天然（gRPC 生态多语言） |
| 服务定义 | Java 接口 | Protobuf IDL |
| 流式调用 | 不支持 | 支持 Client/Server/Bidirectional Stream |
| 浏览器访问 | 不支持 | 支持（HTTP/2） |
| 生态工具 | Dubbo 专用 | gRPC-Curl、grpcurl、Postman |
| 穿透网关 | 需要协议转换 | 直接通过 HTTP 网关 |
| 拦截器 | Dubbo Filter | 兼容 gRPC Interceptor + Dubbo Filter |

# 5. 性能优化

## 5.1 Dubbo 性能优化的关键维度？

```
性能优化金字塔
▲ ┌─────────┐
│ │ 业务优化  │ 减少调用次数、批量接口、缓存
│ ├─────────┤
│ │ 序列化   │ Kryo > Protobuf > Hessian2 > FastJSON2 > JDK
│ ├─────────┤
│ │ 线程模型  │ IO 线程、业务线程池、Dispatcher 策略
│ ├─────────┤
│ │ 网络传输  │ 连接数、Netty 参数、TCP 参数
│ ├─────────┤
│ │ JVM/OS  │ GC 策略、堆内存、文件描述符、内核参数
│ └─────────┘
```

## 5.2 一次 Dubbo 调用经历了多少次拷贝？

```
业务线程（堆内存）
  → 序列化为 byte[]（堆内存）
    → Netty 写缓冲（堆外内存，DirectBuffer）
      → Socket 发送缓冲区（内核空间）
        → 网卡 DMA

共 4 次数据拷贝（可优化方向：使用零拷贝）
```

# 6. Dubbo 3 新特性

## 6.1 Dubbo 3 的核心升级有哪些？

1. **应用级服务发现**：降低注册中心压力，解决推送风暴
2. **Triple 协议**：HTTP/2 + gRPC 兼容，支持流式调用
3. **Mesh 化支持**：Proxyless 模式，直接对接 xDS 协议
4. **JDK 17/21 + 虚拟线程**：Project Loom 集成
5. **Spring Boot 3 原生支持**：Jakarta EE 9+ 迁移
6. **统一路由规则**：ConditionRouter + TagRouter + MeshRouter

## 6.2 Dubbo 3 的虚拟线程支持如何配置？

```yaml
# JDK 21+
dubbo:
  application:
    thread-pool-type: virtual-thread  # 使用虚拟线程替换平台线程池

  protocol:
    threadpool: virtual               # Netty IO 线程也可使用虚拟线程
```

**适用场景**：IO 密集型、大量并发连接、长耗时调用等场景。

**注意事项**：
- 虚拟线程不适合 CPU 密集型任务（会导致频繁的 carrier thread 切换）
- 避免在虚拟线程内使用 `synchronized` 块（会 pin 住 carrier thread）
- 数据库连接池、Redis 连接池的大小可能需要重新评估
