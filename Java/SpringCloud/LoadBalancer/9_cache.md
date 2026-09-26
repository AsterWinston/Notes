
# 1. 为什么需要缓存

每次请求都去注册中心查询实例列表会带来严重问题：

| 问题 | 说明 |
|------|------|
| **性能下降** | 注册中心 RPC 调用开销，降低吞吐 |
| **注册中心压力** | 高并发场景下打爆注册中心 |
| **可用性风险** | 注册中心短暂故障导致所有调用失败 |

# 2. CachingServiceInstanceListSupplier

## 2.1 源码分析

```java
public class CachingServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    // Flux 缓存实现（基于 Reactor 的 CacheMono / CacheFlux）
    private final Flux<List<ServiceInstance>> serviceInstances;

    // 缓存配置
    private final Duration cacheDuration;      // TTL
    private final int maxCacheSize;            // 最大条目数

    public CachingServiceInstanceListSupplier(
            ServiceInstanceListSupplier delegate,
            Duration cacheDuration,
            int maxCacheSize) {
        super(delegate);

        this.cacheDuration = cacheDuration;
        this.maxCacheSize = maxCacheSize;

        // 构建带缓存的 Flux：
        // ① 调用 delegate.get() 获取实例列表
        // ② 缓存 cacheDuration 时长
        // ③ 过期后重新调用 delegate.get()
        this.serviceInstances = CacheFlux
                .lookup(key -> {
                    // 查缓存
                    List<ServiceInstance> cached = cache.get(key);
                    if (cached != null) {
                        return Mono.justOrEmpty(cached);
                    }
                    return Mono.empty();
                }, getServiceId())
                .onCacheMissResume(() ->
                        // 缓存未命中：从 delegate 获取 + 写入缓存
                        delegate.get()
                                .collectList()
                                .doOnNext(instances ->
                                        cache.put(getServiceId(), instances))
                )
                .andWriteWith((key, value) ->
                        // 写入缓存
                        Mono.fromRunnable(() ->
                                cache.put(getServiceId(), value))
                                .then(Mono.just(value))
                );
    }

    @Override
    public Flux<ServiceInstance> get() {
        return serviceInstances.flatMapMany(Flux::fromIterable);
    }

    @Override
    public Flux<ServiceInstance> get(Request request) {
        return get();  // 缓存不与请求上下文关联
    }
}
```

## 2.2 缓存生命周期

```
t=0s     首次请求 → delegate.get() → 查询注册中心 → 返回 3 个实例 → 写入缓存
t=5s     请求 → 缓存命中 → 直接返回（不查注册中心）
t=10s    请求 → 缓存命中 → 直接返回
t=34s    请求 → 缓存命中 → 直接返回
t=35s    请求 → 缓存过期 → delegate.get() → 查询注册中心 → 刷新缓存
t=36s    请求 → 缓存命中 → 直接返回
```

## 2.3 缓存配置

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s          # 缓存生存时间
        capacity: 256     # 最大缓存服务数
```

等价代码配置：

```java
@Bean
public ServiceInstanceListSupplier supplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withCaching()       // 等效于 CachingServiceInstanceListSupplier
            .build(context);
}

// 自定义 TTL
@Bean
public ServiceInstanceListSupplier customCacheSupplier(
        ConfigurableApplicationContext context) {
    DiscoveryClientServiceInstanceListSupplier base =
            new DiscoveryClientServiceInstanceListSupplier(
                    context.getBean(DiscoveryClient.class),
                    context.getEnvironment());
    return new CachingServiceInstanceListSupplier(base,
            Duration.ofSeconds(10),   // TTL=10s
            512);                     // 最大缓存 512 个服务
}
```

# 3. 缓存更新触发条件

| 条件 | 说明 |
|------|------|
| **TTL 过期** | 达到 `spring.cloud.loadbalancer.cache.ttl` 设置的时长 |
| **首次请求** | 缓存中无该服务的数据 |
| **显式清除** | 通过 Actuator endpoint 或编程方式清除 |

# 4. 主动刷新缓存

## 4.1 通过 Actuator 刷新

```yaml
management:
  endpoints:
    web:
      exposure:
        include: loadbalancer-caches
```

```bash
# 查看所有缓存
curl http://localhost:8080/actuator/loadbalancer-caches

# 清除指定服务缓存
curl -X DELETE http://localhost:8080/actuator/loadbalancer-caches/user-service

# 清除所有缓存
curl -X DELETE http://localhost:8080/actuator/loadbalancer-caches
```

## 4.2 编程方式清除

```java
@Service
public class LoadBalancerCacheService {

    @Autowired
    private LoadBalancerCacheManager cacheManager;

    public void evictCache(String serviceId) {
        Cache cache = cacheManager.getCache(serviceId);
        if (cache != null) {
            cache.evict(serviceId);
        }
    }

    public void evictAll() {
        cacheManager.getCacheNames()
                .forEach(name -> {
                    Cache cache = cacheManager.getCache(name);
                    if (cache != null) {
                        cache.clear();
                    }
                });
    }
}
```

# 5. LoadBalancerCacheManager

```java
public interface LoadBalancerCacheManager {

    // 获取指定服务的缓存
    Cache getCache(String serviceId);

    // 获取所有缓存名称
    Collection<String> getCacheNames();
}

// 默认实现：基于 ConcurrentHashMap
public class DefaultLoadBalancerCacheManager implements LoadBalancerCacheManager {

    private final CacheManager delegate;  // Spring Cache Abstraction

    @Override
    public Cache getCache(String serviceId) {
        return delegate.getCache(CACHE_NAME_PREFIX + serviceId);
    }
}
```

# 6. 区域感知缓存（Zone-Aware Caching）

```java
// 不同 Zone 使用不同缓存 key
public class ZoneAwareCachingSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final String zone;
    private final Map<String, List<ServiceInstance>> zoneCache = new ConcurrentHashMap<>();

    @Override
    public Flux<ServiceInstance> get() {
        String key = getServiceId() + ":" + zone;
        List<ServiceInstance> cached = zoneCache.get(key);
        if (cached != null) {
            return Flux.fromIterable(cached);
        }
        return getDelegate().get()
                .collectList()
                .doOnNext(list -> zoneCache.put(key, list))
                .flatMapMany(Flux::fromIterable);
    }
}
```

# 7. 缓存最佳实践

## 7.1 TTL 设置指南

| 场景 | 推荐 TTL | 原因 |
|------|---------|------|
| 生产环境（稳定服务） | 30-60s | 降低注册中心压力 |
| 生产环境（频繁变更） | 5-15s | 快速感知实例变化 |
| 开发环境 | 5-10s | 快速调试 |
| 灰度/金丝雀发布期间 | 3-5s | 及时切换流量 |
| 弹性扩缩容 | 5-15s | 快速感知新实例 |

## 7.2 缓存与失败的平衡

```java
// 健康检查 + 缓存组合：
// 缓存 TTL = 30s（降低注册中心压力）
// 健康检查间隔 = 10s（快速剔除不健康实例）
@Bean
public ServiceInstanceListSupplier balancedSupplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withCaching()                    // 缓存 35s
            .withHealthChecks()               // 10s 健康检查
            .build(context);
}
```

**工作机制**：
- 缓存保存全量实例列表，35s 不查注册中心
- 健康检查每 10s 过滤一次，剔除不健康实例
- 即使注册中心挂了，缓存的实例 + 健康检查仍能保证可用性

## 7.3 大规模微服务场景

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 60s         # 服务多、变更少，TTL 可以更长
        capacity: 1024   # 1000+ 服务
```

如果服务数量超过 `capacity`，旧的缓存条目会被 LRU 淘汰。

## 7.4 避免缓存雪崩

```java
// 给每个服务的 TTL 加上随机抖动，避免同时过期
public class JitterCachingSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final ThreadLocalRandom random = ThreadLocalRandom.current();
    private final Duration baseTtl;
    private final Duration jitter;

    private Duration getTtl() {
        long jitterMs = random.nextLong(-jitter.toMillis(), jitter.toMillis());
        return baseTtl.plusMillis(jitterMs);
    }
}
```
