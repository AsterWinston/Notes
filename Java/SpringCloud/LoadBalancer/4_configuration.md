
# 1. 全局配置属性

Spring Cloud LoadBalancer 的所有配置属性前缀为 `spring.cloud.loadbalancer`：

```yaml
spring:
  cloud:
    loadbalancer:
      # 启用/禁用全局负载均衡
      enabled: true

      # 服务发现集成
      discovery:
        enabled: true

      # 缓存配置
      cache:
        enabled: true
        ttl: 35s                 # 实例列表缓存存活时间（Duration）
        capacity: 256            # 缓存最大容量（服务数量）

      # 重试配置（需引入 spring-retry）
      retry:
        enabled: false
        retry-on-all-operations: false   # 是否对所有操作重试
        max-retries-on-same-service-instance: 0  # 相同实例最大重试
        max-retries-on-next-service-instance: 1  # 下一个实例最大重试
        retryable-status-codes:           # 可重试的状态码
          - 502
          - 503
          - 504

      # 区域配置
      zone: zone1

      # 健康检查配置
      health-check:
        interval: 25s            # 健康检查间隔
        refetch-instances-interval: 25s  # 刷新实例间隔
        repeat-health-check: true        # 是否重复检查

      # 粘滞会话配置
      sticky-session:
        instance-id-cookie-name: sc-lb-instance-id
        add-service-instance-cookie: false

      # 无可用实例时的行为
      use-raw-status-code: false  # true 时不包装 503
```

# 2. LoadBalancerProperties 源码

```java
@ConfigurationProperties("spring.cloud.loadbalancer")
public class LoadBalancerProperties {

    private boolean enabled = true;
    private boolean useRawStatusCode = false;

    // 重试配置
    private Retry retry = new Retry();

    // 区域
    private String zone;

    // 健康检查
    private HealthCheck healthCheck = new HealthCheck();

    // 粘滞会话
    private StickySession stickySession = new StickySession();

    // 提示（Hint）用于扩展
    private Map<String, String> hints = new HashMap<>();

    public static class Retry {
        private boolean enabled = false;
        private boolean retryOnAllOperations = false;
        private int maxRetriesOnSameServiceInstance = 0;
        private int maxRetriesOnNextServiceInstance = 1;
        private Set<Integer> retryableStatusCodes = new HashSet<>();
    }

    public static class HealthCheck {
        private Duration interval = Duration.ofSeconds(25);
        private Duration refetchInstancesInterval = Duration.ofSeconds(25);
        private boolean repeatHealthCheck = true;
    }

    public static class StickySession {
        private String instanceIdCookieName = "sc-lb-instance-id";
        private boolean addServiceInstanceCookie = false;
    }
}
```

# 3. 按服务配置

## 3.1 YAML 按服务配置

```yaml
spring:
  cloud:
    loadbalancer:
      # 以下配置仅对 user-service 生效
      configurations:
        user-service:
          strategy: round-robin
        order-service:
          strategy: random
```

## 3.2 Java 单个服务配置

```java
@Configuration
@LoadBalancerClient(
    name = "user-service",
    configuration = UserServiceLoadBalancerConfig.class
)
public class UserServiceLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> userServiceLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }

    @Bean
    public ServiceInstanceListSupplier userServiceSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withCaching()        // 缓存 35s
                .withZonePreference() // 同 zone 优先
                .build(context);
    }
}
```

## 3.3 多个服务集中配置

```java
@Configuration
@LoadBalancerClients({
    @LoadBalancerClient(name = "user-service",
            configuration = UserConfig.class),
    @LoadBalancerClient(name = "order-service",
            configuration = OrderConfig.class)
})
public class GlobalLoadBalancerConfig {
}

@Configuration
class UserConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> userServiceBalancer(...) {
        return new RoundRobinLoadBalancer(...);
    }
}

@Configuration
class OrderConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> orderServiceBalancer(...) {
        return new RandomLoadBalancer(...);
    }
}
```

# 4. 实例供应器（Supplier）配置

## 4.1 Builder 方式构建链

```java
@Bean
public ServiceInstanceListSupplier customSupplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withBlockingDiscoveryClient()           // ① 阻塞式服务发现
            // .withDiscoveryClient()                // ① 响应式服务发现（二选一）
            .withCaching()                           // ② 开启缓存
            .withZonePreference()                    // ③ 同 Zone 优先
            .withHealthChecks()                      // ④ 健康检查过滤
            .withRetryAwareness()                    // ⑤ 重试感知
            .withHints()                             // ⑥ 透传 Hint
            .withBase(originalSupplier)              // ⑦ 包装自定义 Supplier
            .build(context);
}
```

## 4.2 手动链式组装

```java
@Bean
public ServiceInstanceListSupplier customSupplier(
        ConfigurableApplicationContext context) {

    // ① 基础：从注册中心获取
    DiscoveryClientServiceInstanceListSupplier baseSupplier =
            new DiscoveryClientServiceInstanceListSupplier(
                    context.getBean(DiscoveryClient.class),
                    context.getEnvironment());

    // ② 叠加缓存（35 秒 TTL）
    CachingServiceInstanceListSupplier cachingSupplier =
            new CachingServiceInstanceListSupplier(baseSupplier,
                    Duration.ofSeconds(35));

    // ③ 叠加健康检查
    HealthCheckServiceInstanceListSupplier healthSupplier =
            new HealthCheckServiceInstanceListSupplier(cachingSupplier,
                    Duration.ofSeconds(10), null);

    return healthSupplier;
}
```

# 5. 自定义配置类注意事项

```java
@Configuration(proxyBeanMethods = false)
// ① 不要添加 @Configuration 注解（避免全局生效）
// ② 使用 proxyBeanMethods = false 提高性能
// ③ 不要被 @ComponentScan 扫描到
public class CustomLoadBalancerConfiguration {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> customLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);
        return new MyLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

**关键规则**：

| 规则 | 说明 |
|------|------|
| 不能加 `@Configuration` | 否则对所有服务生效 |
| 必须用 `@LoadBalancerClient` 引用 | 域名隔离 |
| `proxyBeanMethods = false` | 避免 CGLIB 代理开销 |
| 不要被 `@ComponentScan` 扫描 | 放在独立包中 |

# 6. 缓存配置详解

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s           # 默认 35 秒
        capacity: 256      # 默认 256 个服务

        # 等价于旧 Ribbon 配置
        # ribbon.ServerListRefreshInterval=35000
```

缓存工作流程：

```
请求 serviceId = "user-service"
  │
  ▼
CachingServiceInstanceListSupplier
  │
  ├── 缓存命中（TTL 未过期）→ 直接返回缓存列表
  │
  └── 缓存未命中 → 调用 delegate.get()
        │
        ├── DiscoveryClient.getInstances("user-service")  ← 查注册中心
        │
        └── 写入缓存（key = serviceId, TTL = 35s）
```

# 7. 重试配置详解

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
        retry-on-all-operations: true   # 对所有 HTTP 方法重试
        max-retries-on-same-service-instance: 1   # 同一个实例最多重试 1 次
        max-retries-on-next-service-instance: 2   # 最多尝试 2 个不同实例
        retryable-status-codes:
          - 500
          - 502
          - 503
          - 504
```

**重试流程**：

```
① serviceId = "user-service"
② choose() → 实例 A
③ 请求 实例 A → 返回 503
④ retryOnSame → 再次请求 实例 A → 返回 503
⑤ choose() → 实例 B
⑥ 请求 实例 B → 返回 503
⑦ choose() → 实例 C
⑧ 请求 实例 C → 返回 200 ✓
```

# 8. 关闭特定功能的配置

```yaml
# 禁用缓存
spring.cloud.loadbalancer.cache.enabled: false

# 禁用 Zone 偏好
# 方式：不声明 withZonePreference() Supplier

# 禁用健康检查
# 方式：不声明 withHealthChecks() Supplier

# 全局禁用 LoadBalancer
spring.cloud.loadbalancer.enabled: false

# 局部禁用（指定 URL 绕过）
@FeignClient(name = "user-service", url = "http://localhost:8081")
```

# 9. Ribbon 配置迁移对照

| Ribbon 配置 | LoadBalancer 配置 |
|------------|-------------------|
| `ribbon.eureka.enabled` | `spring.cloud.loadbalancer.discovery.enabled` |
| `ribbon.ServerListRefreshInterval` | `spring.cloud.loadbalancer.cache.ttl` |
| `ribbon.MaxAutoRetries` | `spring.cloud.loadbalancer.retry.max-retries-on-same-service-instance` |
| `ribbon.MaxAutoRetriesNextServer` | `spring.cloud.loadbalancer.retry.max-retries-on-next-service-instance` |
| `ribbon.OkToRetryOnAllOperations` | `spring.cloud.loadbalancer.retry.retry-on-all-operations` |
| `ribbon.retryableStatusCodes` | `spring.cloud.loadbalancer.retry.retryable-status-codes` |
| `<service>.ribbon.NFLoadBalancerRuleClassName` | `@LoadBalancerClient(configuration=...)` |
| `<service>.ribbon.listOfServers` | 不再支持静态配置，必须走注册中心 |
