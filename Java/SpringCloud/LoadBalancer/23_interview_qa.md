
# 1. 基础篇

## 1.1 Q1: 什么是 Spring Cloud LoadBalancer？为什么替代 Ribbon？

Spring Cloud LoadBalancer 是 Spring Cloud 官方提供的客户端负载均衡器，从 2020.0 版本开始取代进入维护模式的 Netflix Ribbon。

**替代原因**：
- Ribbon 于 2018 年停止维护，不再有新功能
- Ribbon 基于阻塞 I/O，不支持响应式编程
- Spring Cloud 需要统一的负载均衡抽象，与 WebFlux/WebClient/Gateway 生态一致
- LoadBalancer 提供了更精简的函数式 API、懒加载子容器隔离、装饰器模式的服务实例供应链

## 1.2 Q2: LoadBalancer 的核心设计模式有哪些？

| 模式 | 应用 |
|------|------|
| **装饰器** | `ServiceInstanceListSupplier` 链（缓存/Zone/健康检查层层包装） |
| **策略** | `ReactorLoadBalancer` 实现（轮询/随机/自定义算法） |
| **工厂方法** | `LoadBalancerClientFactory` 为每个服务创建独立子容器 |
| **模板方法** | `NamedContextFactory` 管理子容器生命周期 |
| **观察者** | `LoadBalancerLifecycle` 提供选择完成回调 |

## 1.3 Q3: @LoadBalanced 注解的原理是什么？

`@LoadBalanced` 是一个标记注解（本质是 `@Qualifier`），配合 `LoadBalancerAutoConfiguration` 工作：

1. 自动配置扫描所有标注了 `@LoadBalanced` 的 `RestTemplate` / `WebClient.Builder`
2. 对 `RestTemplate`：注入 `LoadBalancerInterceptor`，拦截请求并将服务名替换为实际实例地址
3. 对 `WebClient.Builder`：注入 `ReactorLoadBalancerExchangeFilterFunction`，在过滤器链中替换 URI

## 1.4 Q4: 负载均衡算法有哪些？默认是什么？

内置算法：
- **RoundRobinLoadBalancer**（默认，轮询）— 基于 `AtomicInteger` 自增取模
- **RandomLoadBalancer**（随机）— 基于 `ThreadLocalRandom` 随机选取

扩展方式：实现 `ReactorServiceInstanceLoadBalancer` 接口，如 Nacos 的 `NacosLoadBalancer`（加权轮询）。

## 1.5 Q5: RestTemplate 和 WebClient 的负载均衡有什么区别？

| 维度 | RestTemplate | WebClient |
|------|-------------|-----------|
| 线程模型 | 阻塞（同步） | 非阻塞（响应式） |
| LB 入口 | `LoadBalancerInterceptor` | `ReactorLoadBalancerExchangeFilterFunction` |
| 实例选择 | `choose().block()` 阻塞等待 | `choose()` 异步返回 Mono |
| 吞吐量 | 受线程池限制 | 受事件循环限制（更高） |
| 超时控制 | `RestTemplateBuilder` 配置 | `HttpClient` 配置 |

# 2. 进阶篇

## 2.1 Q6: LoadBalancerClientFactory 的子容器机制是怎样的？

`LoadBalancerClientFactory` 继承自 `NamedContextFactory`，为每个 `serviceId` 创建独立的 `AnnotationConfigApplicationContext` 子容器：

```
LoadBalancerClientFactory (父容器)
  ├── "user-service" → RoundRobin + UserSupplier
  ├── "order-service" → Random + OrderSupplier
  └── "default" → RoundRobin + DefaultSupplier
```

- 每个子容器可以有自己的 `ReactorLoadBalancer`、`ServiceInstanceListSupplier` 配置
- 通过 `@LoadBalancerClient(name="xxx", configuration=...)` 指定服务级配置
- 懒加载：首次调用该服务时才创建子容器

## 2.2 Q7: ServiceInstanceListSupplier 的装饰链是如何工作的？

采用装饰器模式，链式包装：

```
DiscoveryClientSupplier（从注册中心获取原始列表）
  → CachingSupplier（缓存 N 秒）
  → ZonePreferenceSupplier（同 Zone 过滤）
  → HealthCheckSupplier（健康检查过滤）
  → 自定义Supplier...
```

每个 Supplier 继承 `DelegatingServiceInstanceListSupplier`，在 `get()` 方法中调用 `getDelegate().get()` 获取上游结果，然后进行自己的处理后返回。

## 2.3 Q8: 如何实现灰度发布负载均衡？

实现 `ReactorServiceInstanceLoadBalancer`，在 `choose()` 中：
1. 从 `Request` 的上下文获取灰度标记（如 Header `X-Gray-Version`）
2. 按实例的 `metadata.version` 过滤出灰度/非灰度实例
3. 在候选列表中执行轮询或随机选择

```java
// 核心逻辑
String grayTag = headers.getFirst("X-Gray-Version");
List<ServiceInstance> candidates = instances.stream()
    .filter(i -> grayTag == null
        ? !"gray".equals(i.getMetadata().get("version"))
        : grayTag.equals(i.getMetadata().get("version")))
    .collect(Collectors.toList());
```

## 2.4 Q9: 如何保证幂等操作不因重试产生副作用？

1. 默认 `retry-on-all-operations=false`，仅对 GET 方法重试
2. `max-retries-on-same-service-instance=0`，换实例重试而非同一个
3. 如果业务确需重试 POST 等操作，需保证服务端幂等（如唯一键去重）
4. 建议在 LoadBalancer 层做重试（可换实例），关闭 Feign 自带重试

## 2.5 Q10: Zone 感知路由的原理与降级策略？

**原理**：`ZonePreferenceServiceInstanceListSupplier` 从注册中心获取所有实例后，按 `metadata.zone` 过滤出与当前实例同 Zone 的实例：

- 同 Zone 有实例 → 仅返回同 Zone 实例（完全隔离）
- 同 Zone 无实例 → 降级返回所有实例（跨 Zone 路由）

**优化**：可以实现多级 Zone 优先级降级（SH → BJ → GZ → 全球），而不是直接暴露全部。

# 3. 实战篇

## 3.1 Q11: Ribbon 迁移到 LoadBalancer 需要做哪些工作？

| 步骤 | 操作 |
|------|------|
| 1. 升级版本 | Spring Cloud 升级到 2020.0.x 或更高 |
| 2. 移除依赖 | 排除 `spring-cloud-starter-netflix-ribbon` |
| 3. 更新配置 | `ribbon.*` → `spring.cloud.loadbalancer.*` |
| 4. 替换注解 | `@RibbonClient` → `@LoadBalancerClient` |
| 5. 迁移规则 | `IRule` → `ReactorLoadBalancer<ServiceInstance>` |
| 6. 迁移过滤 | `ServerListFilter` → `ServiceInstanceListSupplier` 装饰器 |
| 7. 关闭 Feign 重试 | `feign.client.config.default.retryer=NEVER_RETRY` |
| 8. 开启 LB 重试 | 引入 spring-retry + 配置 retry |

## 3.2 Q12: 生产环境 LoadBalancer 最佳实践配置是什么？

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 15s                  # 生产建议 10-30s
        capacity: 512
      retry:
        enabled: true
        max-retries-on-same-service-instance: 0
        max-retries-on-next-service-instance: 2
        retry-on-all-operations: false
        retryable-status-codes: [502, 503, 504]
      zone: ${ZONE:default}
      health-check:
        interval: 15s
        refetch-instances-interval: 30s
```

配合：
- Actuator health endpoint 暴露
- 注册中心健康检查 + 心跳
- 优雅上下线
- 连接池配置

## 3.3 Q13: 高并发下 RoundRobinLoadBalancer 的性能瓶颈是什么？

`AtomicInteger.incrementAndGet()` 在高并发下会产生 CAS 自旋竞争。虽然每个 CAS 操作很快（~50ns），但在极高并发（>100k QPS）下仍可能成为瓶颈。

**优化方案**：
- 使用 `LongAdder`（JDK 8+），分段累加，写入无需 CAS
- 或使用 `ThreadLocal` 计数器 + 定期汇总

## 3.4 Q14: 为什么 FeignClient 指定了 url 就不走 LoadBalancer？

`FeignBlockingLoadBalancerClient` 在执行前检查请求 URL 的 host：
- 如果 host 匹配已知的 `serviceId` → 走负载均衡
- 如果 host 是 IP 地址或配置了 url → 直接透传到 delegate HTTP 客户端
- 判断逻辑：`if (serviceId == null || isNotServiceName(serviceId))` → 直连

## 3.5 Q15: 如何处理缓存导致的实例感知延迟？

| 方案 | 适用场景 |
|------|---------|
| 缩短 TTL（5-10s） | 频发变更/灰度发布期间 |
| 健康检查辅助 | 缓存实例但健康检查过滤不健康节点 |
| 主动清除缓存 | 通过 Actuator endpoint 在实例上下线时主动清除 |
| 推送通知刷新 | 监听注册中心变更事件，主动 evict 缓存 |
| 预热策略 | 新实例上线后延迟一段时间再接流量 |

# 4. 对比篇

## 4.1 Q16: Ribbon vs LoadBalancer 的核心差异？

| 维度 | Ribbon | LoadBalancer |
|------|--------|-------------|
| 维护 | Netflix 停止维护 | Spring 官方活跃 |
| 响应式 | 不支持 | 原生 Reactor |
| API | `ILoadBalancer`/`IRule`/`IPing` | `ReactorLoadBalancer`/`ServiceInstanceListSupplier` |
| 配置隔离 | `@RibbonClient` + 命名空间 | `NamedContextFactory` 子容器 |
| 扩展 | 替换 Rule/Ping/Filter | 装饰器链 + 自定义 Balancer |
| 缓存 | ServerList 缓存 | `CachingServiceInstanceListSupplier` |
| 注册中心 | Eureka 原生绑定 | `DiscoveryClient` 抽象 |

## 4.2 Q17: 客户端负载均衡 vs 服务端负载均衡？

| 维度 | 客户端 LB（LoadBalancer） | 服务端 LB（Nginx/K8s Service） |
|------|--------------------------|-------------------------------|
| 位置 | 调用方进程内 | 独立进程/集群 |
| 延迟 | 低（本地选择） | 多一跳 |
| 实例感知 | 直接查询注册中心 | 依赖服务发现插件 |
| 策略控制 | 灵活（每个调用方可配不同策略） | 全局统一 |
| 故障感知 | 快速 | 需健康检查等待 |
| 复杂度 | 集成在应用内 | 独立组件 |
| 单点风险 | 无（每个实例独立） | 有（LB 本身需高可用） |

**推荐组合**：K8s Service（L4 负载均衡）+ Spring Cloud LoadBalancer（L7 客户端负载均衡），兼顾性能和灵活性。
