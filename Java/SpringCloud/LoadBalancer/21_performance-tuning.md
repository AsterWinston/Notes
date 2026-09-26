
# 1. 性能基准

## 1.1 单次 choose() 延迟

| 场景 | 延迟 | 说明 |
|------|------|------|
| 缓存命中 | 10-20μs | 纯内存操作 |
| 缓存未命中 | 5-50ms | 查注册中心 |
| 健康检查开启 | +1-3ms/实例 | 需额外 HTTP 调用 |

## 1.2 吞吐量（理论值）

| 场景 | 理论 QPS | 限制因素 |
|------|---------|---------|
| 纯内存 | 1,000,000+ | CPU |
| 缓存命中 | 500,000+ | CAS 竞争 |
| 注册中心 | 取决于注册中心 QPS 上限 | 网络 I/O |

# 2. 缓存优化

## 2.1 TTL 调优

```yaml
# 核心原则：TTL 越长性能越好，但感知变化越慢
# 需要在性能和数据新鲜度之间权衡

spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 35s          # 默认值
        capacity: 256     # 服务数量
```

TTL 对注册中心 QPS 的影响：

| TTL | 单服务调用 QPS | 注册中心承受 QPS | 实例感知延迟 |
|-----|---------------|-----------------|-------------|
| 1s | 10,000 | 10 | 1s |
| 5s | 10,000 | 2 | 5s |
| 35s | 10,000 | 0.3 | 35s |
| 60s | 10,000 | 0.17 | 60s |

## 2.2 主动刷新策略

```java
// 结合推送通知主动刷新缓存，避免长 TTL 导致感知延迟

@NacosConfigListener(dataId = "instance-change", groupId = "SERVICE_GROUP")
public void onInstanceChange(String changeEvent) {
    String serviceId = parseServiceId(changeEvent);
    cacheManager.getCache(serviceId).evict(serviceId);
    log.info("Evicted cache for {} due to instance change", serviceId);
}
```

# 3. 健康检查优化

## 3.1 并发健康检查

```java
public class ParallelHealthCheckSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final int concurrency = 10;  // 并发检查数

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .collectList()
                .flatMapMany(instances -> {
                    // 并行健康检查（限制并发数）
                    return Flux.fromIterable(instances)
                            .flatMap(instance ->
                                    checkHealth(instance)
                                            .filter(healthy -> healthy)
                                            .map(h -> instance),
                                    concurrency);   // 限制同时检查 10 个
                });
    }

    private Mono<Boolean> checkHealth(ServiceInstance instance) {
        return WebClient.create()
                .get()
                .uri(instance.getUri() + "/actuator/health")
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofSeconds(2)) // 单个检查最多 2s
                .map(r -> r.getStatusCode().is2xxSuccessful())
                .onErrorReturn(false);
    }
}
```

## 3.2 轻量级健康端点

```java
// 自定义轻量级健康端点，只检查关键依赖
@Component
public class LightweightHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // 仅检查内存和线程池状态（不查 DB/Redis）
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();

        if (usedMemory > maxMemory * 0.9) {
            return Health.down()
                    .withDetail("reason", "High memory usage")
                    .build();
        }
        return Health.up().build();
    }
}
```

# 4. 连接池优化

## 4.1 RestTemplate 连接池

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    // 使用 Apache HttpClient 5 连接池
    PoolingHttpClientConnectionManager cm =
            new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(200);               // 总连接数
    cm.setDefaultMaxPerRoute(50);      // 每个路由最大连接

    CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(cm)
            .setConnectionTimeToLive(30, TimeUnit.SECONDS)
            .evictIdleConnections(30, TimeUnit.SECONDS)
            .build();

    return new RestTemplateBuilder()
            .requestFactory(() -> new HttpComponentsClientHttpRequestFactory(httpClient))
            .setConnectTimeout(Duration.ofSeconds(3))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();
}
```

## 4.2 WebClient 连接池（Reactor Netty）

```java
ConnectionProvider provider = ConnectionProvider.builder("lb-pool")
        .maxConnections(500)                      // 最多 500 连接
        .pendingAcquireMaxCount(2000)             // 等待队列最长 2000
        .pendingAcquireTimeout(Duration.ofSeconds(45))
        .maxIdleTime(Duration.ofSeconds(30))      // 30s 空闲释放
        .maxLifeTime(Duration.ofSeconds(60))      // 60s 强制回收
        .lifo()                                   // LIFO 策略（复用热连接）
        .metrics(true)                            // 启用 Micrometer 指标
        .build();
```

## 4.3 Feign 连接池

```yaml
feign:
  httpclient:
    hc5:
      enabled: true
    connection-timeout: 3000
    connection-timer-repeat: 3000
  client:
    config:
      default:
        connectTimeout: 3000
        readTimeout: 5000
        loggerLevel: HEADERS
```

# 5. 减少 AtomicInteger CAS 竞争

```java
// RoundRobinLoadBalancer 使用 AtomicInteger 在高并发下会产生 CAS 竞争
// 优化方案：使用 LongAdder 或分段计数器

public class OptimizedRoundRobinLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    // LongAdder 在高并发下性能优于 AtomicInteger（JDK 8+）
    private final LongAdder position = new LongAdder();
    private final int vmId = new Random().nextInt(1000);

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }

        position.increment();  // LongAdder 写入无需 CAS 循环
        int pos = Math.abs((int) (position.sum() % Integer.MAX_VALUE));
        // ↑ 注意：LongAdder.sum() 不是原子快照，但轮询允许少量不精确

        ServiceInstance instance = instances.get(pos % instances.size());
        return new DefaultResponse(instance);
    }
}
```

# 6. 内存优化

## 6.1 子容器数量控制

```java
// LoadBalancerClientFactory 为每个 serviceId 创建独立子容器
// 大量服务时可能导致内存膨胀

// 监控子容器数量
@EventListener
public void onContextRefreshed(ContextRefreshedEvent event) {
    if (event.getApplicationContext() instanceof AnnotationConfigApplicationContext) {
        log.info("LoadBalancer sub-context created");
    }
}

// 不需要时释放
@PreDestroy
public void destroy() {
    // LoadBalancerClientFactory 实现了 DisposableBean，自动清理
}
```

## 6.2 实例列表缓存大小

```yaml
# capacity 限制缓存的服务数量
spring:
  cloud:
    loadbalancer:
      cache:
        capacity: 256   # 最多缓存 256 个服务，超出 LRU 淘汰
```

# 7. 调优检查清单

| 优化项 | 默认值 | 推荐值 | 说明 |
|--------|--------|--------|------|
| 缓存 TTL | 35s | 10-60s | 按环境调整 |
| 缓存容量 | 256 | 按服务数+30% | 避免 LRU 频繁淘汰 |
| 健康检查间隔 | 25s | 15-30s | 不宜太短 |
| 重试次数 | 2 | 1-3 | 多了浪费 |
| 重试退避 | 100ms | 100-500ms | 避免重试风暴 |
| 连接池大小 | 默认 | 200-500 | 按 QPS 估算 |
| 连接超时 | 无 | 3s | 快速失败 |
| 读取超时 | 无 | 5s | 根据 SLA |
| 空闲连接回收 | 无 | 30s | 及时释放资源 |

# 8. JVM 参数建议

```bash
# 高并发负载均衡场景
java \
  -Xms2g -Xmx2g \                         # 固定堆大小避免 GC 波动
  -XX:+UseG1GC \                           # G1 更适合大堆
  -XX:MaxGCPauseMillis=100 \               # 控制 GC 停顿
  -XX:+AlwaysPreTouch \                    # 预触内存减少运行时延迟
  -Dreactor.netty.pool.leasingStrategy=lifo \  # LIFO 复用热连接
  -jar app.jar
```

# 9. 压测验证

```java
// 使用 JMH 验证性能
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
public class LoadBalancerBenchmark {

    private RoundRobinLoadBalancer balancer;
    private List<ServiceInstance> instances;

    @Setup
    public void setup() {
        instances = IntStream.range(0, 10)
                .mapToObj(i -> new DefaultServiceInstance(
                        "svc-" + i, "192.168.1." + i, 8080, false))
                .collect(Collectors.toList());

        ObjectProvider<ServiceInstanceListSupplier> provider =
                new ObjectProvider<>() {
                    @Override
                    public ServiceInstanceListSupplier getIfAvailable() {
                        return new TestSupplier(instances);
                    }
                };

        balancer = new RoundRobinLoadBalancer(provider, "test");
    }

    @Benchmark
    public Response<ServiceInstance> benchmarkChoose() {
        return balancer.choose(new DefaultRequest<>(
                new RequestDataContext(null, null)))
                .block();
    }
}
```
