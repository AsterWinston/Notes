
# 1. 概述

Spring Cloud LoadBalancer 通过 `HealthCheckServiceInstanceListSupplier` 对 `ServiceInstanceListSupplier` 进行装饰，在实例列表返回前剔除不健康的实例，确保流量只路由到健康的后端节点。

# 2. HealthCheckServiceInstanceListSupplier

## 2.1 源码分析

```java
public class HealthCheckServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier
        implements InitializingBean, DisposableBean {

    private final ReactiveLoadBalancer.Factory<ServiceInstance> factory;
    private final Duration healthCheckInterval;
    private final Duration refetchInstancesInterval;
    private final String serviceId;

    // 缓存上次检查的健康实例列表
    private Map<String, List<ServiceInstance>> healthCheckResults = new ConcurrentHashMap<>();

    @Override
    public Flux<ServiceInstance> get() {
        String serviceId = getServiceId();
        List<ServiceInstance> checked = healthCheckResults.get(serviceId);
        if (checked != null) {
            return Flux.fromIterable(checked);
        }
        // 首次调用或缓存过期：重新检查
        return refreshAndGet(serviceId);
    }

    private Flux<ServiceInstance> refreshAndGet(String serviceId) {
        return getDelegate().get()
                .collectList()
                .flatMapMany(instances -> {
                    // 对每个实例执行健康检查
                    return Flux.fromIterable(instances)
                            .filterWhen(instance -> {
                                // 构建健康检查 URL
                                String healthUrl = instance.getUri() + "/actuator/health";
                                return WebClient.create()
                                        .get().uri(healthUrl)
                                        .retrieve()
                                        .toBodilessEntity()
                                        .map(response -> response.getStatusCode().is2xxSuccessful())
                                        .onErrorReturn(false);
                            })
                            .collectList()
                            .doOnNext(healthy -> healthCheckResults.put(serviceId, healthy))
                            .flatMapMany(Flux::fromIterable);
                });
    }

    @Override
    public void afterPropertiesSet() {
        // 定时刷新健康检查结果
        Schedulers.single().schedulePeriodically(
                () -> healthCheckResults.clear(), // 清空缓存触发重新检查
                healthCheckInterval.toMillis(),
                healthCheckInterval.toMillis(),
                TimeUnit.MILLISECONDS);
    }
}
```

## 2.2 健康检查流程图

```
请求实例列表
    │
    ▼
HealthCheckServiceInstanceListSupplier
    │
    ├── 缓存有效 → 返回缓存的健康列表
    │
    └── 缓存失效
          │
          ▼
    ① 调用 delegate.get() 获取全量实例
          │
          ▼
    ② 对每个实例发起 HTTP 健康检查
       GET http://192.168.1.10:8080/actuator/health
          │
          ├── 200 / 2xx → 标记健康
          └── 失败/超时 → 标记不健康（过滤掉）
          │
          ▼
    ③ 将健康实例列表写入缓存
          │
          ▼
    ④ 返回 Flux<ServiceInstance>
```

# 3. 配置方式

## 3.1 Builder 链式配置

```java
@Bean
public ServiceInstanceListSupplier supplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withHealthChecks()
            .withCaching()
            .build(context);
}
```

## 3.2 配置文件调整

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        interval: 15s              # 健康检查间隔
        refetch-instances-interval: 25s  # 全量刷新实例间隔
        repeat-health-check: true  # 是否重复检查失败实例
```

## 3.3 自定义健康检查实现

```java
public class CustomHealthCheckServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final Map<String, HealthStatus> healthCache = new ConcurrentHashMap<>();
    private final WebClient webClient;

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .filterWhen(this::isInstanceHealthy);
    }

    private Mono<Boolean> isInstanceHealthy(ServiceInstance instance) {
        String key = instance.getHost() + ":" + instance.getPort();

        // 缓存命中：直接返回
        HealthStatus cached = healthCache.get(key);
        if (cached != null && !cached.isExpired()) {
            return Mono.just(cached.isHealthy());
        }

        // 执行健康检查
        return webClient.get()
                .uri(instance.getUri() + "/actuator/health")
                .retrieve()
                .bodyToMono(HealthResponse.class)
                .map(resp -> {
                    boolean healthy = "UP".equals(resp.getStatus());
                    healthCache.put(key, new HealthStatus(healthy, Instant.now()));
                    return healthy;
                })
                .onErrorReturn(false);
    }

    @Data
    @AllArgsConstructor
    private static class HealthStatus {
        private boolean healthy;
        private Instant checkTime;

        boolean isExpired() {
            return Duration.between(checkTime, Instant.now()).getSeconds() > 15;
        }
    }
}
```

# 4. 集成 Actuator 健康端点

## 4.1 暴露健康端点（被调用方）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true   # Kubernetes 就绪/存活探针
```

## 4.2 自定义健康指示器

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(3)) {
                return Health.up()
                        .withDetail("database", "MySQL")
                        .withDetail("status", "connected")
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withDetail("database", "MySQL")
                    .withDetail("error", e.getMessage())
                    .build();
        }
        return Health.down().build();
    }
}
```

# 5. SameInstancePreferenceServiceInstanceListSupplier

粘滞会话（Sticky Session）的实现：

```java
public class SameInstancePreferenceServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private ServiceInstance previouslySelectedInstance;

    public SameInstancePreferenceServiceInstanceListSupplier(
            ServiceInstanceListSupplier delegate) {
        super(delegate);
    }

    @Override
    public Flux<ServiceInstance> get() {
        // 将上次选中的实例放到列表最前面（优先选择）
        return getDelegate().get()
                .collectList()
                .flatMapMany(instances -> {
                    if (previouslySelectedInstance != null) {
                        List<ServiceInstance> reordered = new ArrayList<>(instances);
                        reordered.remove(previouslySelectedInstance);
                        reordered.add(0, previouslySelectedInstance);
                        return Flux.fromIterable(reordered);
                    }
                    return Flux.fromIterable(instances);
                });
    }

    public void setPreviouslySelectedInstance(ServiceInstance instance) {
        this.previouslySelectedInstance = instance;
    }
}
```

# 6. 区域感知过滤（ZonePreferenceServiceInstanceListSupplier）

```java
public class ZonePreferenceServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final String zone;

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .collectList()
                .flatMapMany(allInstances -> {
                    // 筛选同 Zone 实例
                    List<ServiceInstance> sameZone = allInstances.stream()
                            .filter(i -> zone.equals(i.getMetadata().get("zone")))
                            .collect(Collectors.toList());

                    // 有同 Zone 实例：只用同 Zone
                    // 无同 Zone 实例：全部实例
                    if (!sameZone.isEmpty()) {
                        return Flux.fromIterable(sameZone);
                    }
                    return Flux.fromIterable(allInstances);
                });
    }
}
```

# 7. 实例过滤链完整示例

```java
@Configuration
public class ProductionLoadBalancerConfig {

    @Bean
    public ServiceInstanceListSupplier productionSupplier(
            ConfigurableApplicationContext context) {

        DiscoveryClientServiceInstanceListSupplier base =
                new DiscoveryClientServiceInstanceListSupplier(
                        context.getBean(DiscoveryClient.class),
                        context.getEnvironment());

        // ① Zone 过滤 → ② 健康检查 → ③ 缓存
        ZonePreferenceServiceInstanceListSupplier zoneFilter =
                new ZonePreferenceServiceInstanceListSupplier(base);

        HealthCheckServiceInstanceListSupplier healthFilter =
                new HealthCheckServiceInstanceListSupplier(zoneFilter, ...);

        CachingServiceInstanceListSupplier cached =
                new CachingServiceInstanceListSupplier(
                        healthFilter,
                        Duration.ofSeconds(10));

        return cached;
    }
}
```

**过滤顺序**：

```
DiscoveryClient（全量 10 个实例）
    │
    ▼ ZonePreference（同 Zone：6 个）
    │
    ▼ HealthCheck（健康：4 个）
    │
    ▼ Caching（缓存 10s）
    │
    ▼ ReactorLoadBalancer.choose()（轮询/随机选 1 个）
```

# 8. 常见问题与最佳实践

| 问题 | 原因 | 解决 |
|------|------|------|
| 健康检查太频繁 | `interval` 过小 | 适当增大（如 15-30s） |
| 不健康实例仍被调用 | 缓存未更新 | 缩短 `refetch-instances-interval` |
| Actuator endpoint 泄露 | `show-details: always` | 生产环境用 `when-authorized` |
| 健康检查阻塞 | 同步 HTTP 调用 | 使用 WebClient 异步检查 |
| 大量实例健康检查 OOM | 无上限并发 | 限制 `.filterWhen` 并发数 |

**最佳实践**：

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        interval: 15s                      # 15s 检查一次
        refetch-instances-interval: 30s    # 30s 全量刷新
        repeat-health-check: true          # 重复检查（直到恢复）

management:
  endpoint:
    health:
      probes:
        enabled: true                      # K8s 就绪探针
      show-details: when-authorized        # 生产安全
```
