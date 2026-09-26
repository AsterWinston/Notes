
# 1. 架构设计

## 1.1 服务名命名规范

```
服务名: {业务域}-{服务职责}

推荐:
  user-service
  order-service
  payment-gateway
  inventory-sync

避免:
  service1          （无意义）
  com.example.order （包含包名）
  User-Service      （包含大写）
  user_service      （下划线分隔）
```

## 1.2 配置分层

```yaml
# ① 全局默认配置（application.yml）
spring:
  cloud:
    loadbalancer:
      enabled: true
      cache:
        ttl: 30s
        capacity: 512
      retry:
        enabled: true
        max-retries-on-next-service-instance: 2

---
# ② 环境级覆盖（application-prod.yml）
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 10s          # 生产缩短缓存，更快感知实例变化

---
# ③ 服务级配置（@LoadBalancerClient）
@LoadBalancerClient(name = "user-service",
        configuration = UserServiceLBConfig.class)
```

## 1.3 命名空间（Nacos）

```
推荐命名空间划分：
  - public (保留)
  - dev      (开发)
  - test     (测试)
  - staging  (预发)
  - prod     (生产)
```

# 2. 负载均衡策略选择

```java
// 推荐配置
@Configuration
@LoadBalancerClients(defaultConfiguration = DefaultLBConfig.class)
public class LoadBalancerGlobalConfig {
}

@Configuration(proxyBeanMethods = false)
class DefaultLBConfig {

    @Bean
    public ServiceInstanceListSupplier supplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()           // 必须
                .withCaching()                   // 必须（性能）
                .withZonePreference()            // 推荐（多机房）
                .withHealthChecks()              // 推荐（容错）
                .build(context);
    }

    @Bean
    public ReactorLoadBalancer<ServiceInstance> balancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                factory.getLazyProvider(name,
                        ServiceInstanceListSupplier.class), name);
    }
}
```

# 3. 缓存配置最佳实践

| 场景 | TTL | capacity | 说明 |
|------|-----|----------|------|
| 生产环境（服务稳定） | 30-60s | 512 | 降低注册中心压力 |
| 生产环境（频繁变更） | 10-20s | 512 | 快速感知变化 |
| 灰度发布期间 | 3-5s | 默认 | 及时感知实例上下线 |
| 开发环境 | 5-10s | 64 | 快速调试 |
| 大规模（>1000 服务） | 60-120s | 2000 | 服务多 TTL 要大 |

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: ${LB_CACHE_TTL:35s}
        capacity: ${LB_CACHE_CAPACITY:512}
```

# 4. 重试配置最佳实践

```yaml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
        retry-on-all-operations: false   # 仅 GET 重试
        max-retries-on-same-service-instance: 0   # 不重试同实例
        max-retries-on-next-service-instance: 2   # 最多换 2 个
        retryable-status-codes:
          - 502
          - 503
          - 504
        backoff:
          enabled: true
          min-backoff: 100ms
          max-backoff: 500ms
          multiplier: 1.5

feign:
  client:
    config:
      default:
        retryer: feign.Retryer.NEVER_RETRY   # 重试交给 LB 层
```

# 5. 健康检查配置

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        interval: 15s                    # 每 15s 检查一次
        refetch-instances-interval: 30s  # 每 30s 全量刷新
        repeat-health-check: true        # 不健康实例持续检查

management:
  endpoints:
    web:
      exposure:
        include: health                  # 暴露健康端点
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
```

# 6. HTTP 客户端配置

## 6.1 RestTemplate 连接池

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .setConnectTimeout(Duration.ofSeconds(3))
                .setReadTimeout(Duration.ofSeconds(5))
                .build();
    }
}
```

## 6.2 WebClient 连接池

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    ConnectionProvider provider = ConnectionProvider.builder("lb-pool")
            .maxConnections(500)
            .pendingAcquireMaxCount(1000)
            .maxIdleTime(Duration.ofSeconds(30))
            .build();

    HttpClient httpClient = HttpClient.create(provider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            .responseTimeout(Duration.ofSeconds(5));

    return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient));
}
```

# 7. 多 Zone / 多集群部署

```yaml
# 同 Zone 优先 + 降级策略
spring:
  cloud:
    loadbalancer:
      zone: ${POD_ZONE:default}          # 通过环境变量注入

# 实例注册时携带 Zone 信息
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          zone: ${POD_ZONE:SH}
```

```java
// 确保 Zone 感知的 Supplier 被使用
@Bean
public ServiceInstanceListSupplier supplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withZonePreference()
            .withHealthChecks()     // 健康检查放在 Zone 过滤之后
            .build(context);
}
```

# 8. 监控与可观测性

```java
@Component
public class LoadBalancerMetrics {

    @Bean
    public LoadBalancerLifecycle metricsLifecycle(
            MeterRegistry meterRegistry) {
        return new LoadBalancerLifecycle<
                RequestDataContext, Object, ServiceInstance>() {

            @Override
            public void onStart(Request<RequestDataContext> request) {
                meterRegistry.counter("lb.choose.start").increment();
            }

            @Override
            public void onComplete(
                    CompletionContext<Object, ServiceInstance,
                            RequestDataContext> context) {
                if (context.getClientResponse() != null) {
                    ServiceInstance instance = context.getClientResponse()
                            .getServer();
                    meterRegistry.counter("lb.choose.success",
                            "service", instance.getServiceId()).increment();
                } else {
                    meterRegistry.counter("lb.choose.failed").increment();
                }
            }
        };
    }
}
```

# 9. 安全检查清单

| 检查项 | 说明 |
|--------|------|
| 是否关闭了 Ribbon | 移除 `spring-cloud-starter-netflix-ribbon` 依赖 |
| 是否开启缓存 | `spring.cloud.loadbalancer.cache.enabled=true` |
| TTL 是否合理 | 根据环境调整，生产 10-60s |
| 是否启用健康检查 | `withHealthChecks()` 过滤不健康实例 |
| 是否配置重试 | 引入 spring-retry + 配置 `max-retries` |
| 是否关闭 Feign 重试 | `feign.client.config.default.retryer=NEVER_RETRY` |
| 是否暴露 Actuator | `management.endpoints.web.exposure.include=loadbalancer-caches,health` |
| 是否设置超时 | RestTemplate/WebClient/Feign 超时 < 5s |
| 是否配置连接池 | WebClient 连接池和 RestTemplate 连接池 |
| 是否做好降级 | Feign fallback + WebClient onErrorResume |

# 10. 常见反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| 缓存 TTL 设为 0 | 每次查注册中心，打爆它 | TTL >= 5s |
| 重试所有方法（含 POST） | 可能重复创建资源 | `retry-on-all-operations=false` |
| 同一个实例重试多次 | 浪费资源，治标不治本 | `max-retries-on-same-service-instance=0` |
| 不配置超时 | 请求可能无限等待 | 每个客户端都设超时 |
| 不暴露健康端点 | 无法剔除不健康实例 | 集成 Actuator |
| 不使用连接池 | 每次新建 TCP 连接 | 配置 WebClient/RestTemplate 连接池 |
| `@LoadBalancerClient` 配置类加 `@Configuration` | 全局生效导致混乱 | 去掉 `@Configuration` |
