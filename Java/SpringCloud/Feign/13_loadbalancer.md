
# 1. 概述

Spring Cloud OpenFeign 默认集成了 **Spring Cloud LoadBalancer**，可以将 `@FeignClient` 中的服务名（`name`）自动解析为实际的服务实例地址，并实现客户端负载均衡。

```
服务名 (user-service)
    │
    │  Spring Cloud LoadBalancer
    ▼
实例列表 (192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080)
    │
    │  负载均衡算法
    ▼
选定实例 (192.168.1.11:8080)
```

# 2. 版本演进：Ribbon → Spring Cloud LoadBalancer

| | Ribbon | Spring Cloud LoadBalancer |
|------|-------|--------------------------|
| **状态** | 已进入维护模式 | 官方推荐 |
| **Spring Cloud 版本** | Hoxton 及之前 | Hoxton.SR1+ (2020.0+) |
| **响应式支持** | 不支持 | 支持 WebFlux |
| **缓存** | ServerList 缓存 | CachingServiceInstanceListSupplier |
| **API 风格** | 命令式 | 响应式 + 阻塞桥接 |

**迁移注意**（2020.0.x 起）：
- `spring-cloud-starter-netflix-ribbon` → 移除
- `spring-cloud-starter-loadbalancer` → 自动引入（`spring-cloud-starter-openfeign` 已包含）
- `ribbon.xxx` 配置 → `spring.cloud.loadbalancer.xxx`

# 3. 核心配置

```yaml
spring:
  cloud:
    loadbalancer:
      # 是否启用
      enabled: true
      # 服务发现集成
      discovery:
        enabled: true
      # 缓存配置
      cache:
        enabled: true
        ttl: 35s          # 实例列表缓存 TTL
        capacity: 256     # 缓存容量
      # 重试配置（需引入 spring-retry）
      retry:
        enabled: false
      # Zone 区域感知
      zone: zone1
```

# 4. 负载均衡算法

## 4.1 默认：RoundRobin（轮询）

```java
// 默认使用 RoundRobinLoadBalancer，无需额外配置
```

## 4.2 切换到随机算法

```java
@Configuration
public class LoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name
        );
    }
}
```

## 4.3 切换到 Nacos 加权轮询

```java
@Bean
public ReactorLoadBalancer<ServiceInstance> nacosWeightedLoadBalancer(
        Environment environment,
        LoadBalancerClientFactory loadBalancerClientFactory) {
    String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new NacosLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
    );
}
```

# 5. 自定义负载均衡策略

```java
@Configuration
public class GrayLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> grayLoadBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new GrayRoundRobinLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name
        );
    }
}

public class GrayRoundRobinLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;
    private final AtomicInteger position = new AtomicInteger(0);

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = supplierProvider.getIfAvailable();

        // 获取请求上下文中的灰度标记
        RequestDataContext context = (RequestDataContext) request.getContext();
        HttpHeaders headers = context.getClientRequest().getHeaders();
        boolean gray = Boolean.parseBoolean(headers.getFirst("X-Gray"));

        return supplier.get(request).next().map(instances -> {
            // 根据灰度标记过滤实例
            List<ServiceInstance> filtered = instances.stream()
                    .filter(i -> gray == isGrayInstance(i))
                    .collect(Collectors.toList());

            if (filtered.isEmpty()) {
                return new EmptyResponse();
            }

            // 轮询
            int pos = Math.abs(position.incrementAndGet()) % filtered.size();
            return new DefaultResponse(filtered.get(pos));
        });
    }

    private boolean isGrayInstance(ServiceInstance instance) {
        String gray = instance.getMetadata().get("gray");
        return Boolean.parseBoolean(gray);
    }
}
```

# 6. @LoadBalancerClient 注解

```java
@Configuration
@LoadBalancerClient(
    name = "user-service",
    configuration = UserServiceLoadBalancerConfig.class
)
public class LoadBalancerAutoConfig { }
```

声明某个服务的**独立负载均衡配置**，不与其他服务共享。

# 7. FeignBlockingLoadBalancerClient 源码解析

```java
public class FeignBlockingLoadBalancerClient implements Client {

    private final Client delegate;          // 底层 HTTP 引擎
    private final LoadBalancerClient loadBalancerClient;

    @Override
    public Response execute(Request request, Request.Options options) throws IOException {
        // 1. 提取 serviceId（从 URL host 部分）
        URI uri = URI.create(request.url());
        String serviceId = uri.getHost();

        // 2. 通过 LoadBalancer 选择实例
        ServiceInstance instance = loadBalancerClient.choose(serviceId);

        // 3. 重建 URL（替换 host:port）
        String newUrl = request.url().replace(
                uri.getHost() + ":" + uri.getPort(),
                instance.getHost() + ":" + instance.getPort()
        );

        // 4. 委托给真正的 HTTP 客户端执行
        Request newRequest = Request.create(..., newUrl, ...);
        return delegate.execute(newRequest, options);
    }
}
```

**关键流程**：
1. `request.url()` → 提取 host 作为 serviceId
2. `loadBalancerClient.choose(serviceId)` → 选一个实例
3. 将 URL 中的 host:port 替换为实际实例地址
4. 委托 `delegate`（如 `ApacheHttpClient`）执行

# 8. 不经过负载均衡的调用

3 种方式直连指定 URL：

**方式 1：@FeignClient 指定 url**

```java
@FeignClient(name = "external-api", url = "https://api.example.com")
public interface ExternalApiClient { }
```

**方式 2：Feign.Builder 直接 target URL**

```java
UserClient client = Feign.builder()
        .target(UserClient.class, "http://192.168.1.100:8080");
```

**方式 3：不引入 spring-cloud-starter-loadbalancer（或排除依赖）**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

# 9. LoadBalancer 生命周期

```
首次请求某个服务
    │
    ├── ServiceInstanceListSupplier 获取实例列表
    │       │
    │       ├── DiscoveryClientServiceInstanceListSupplier (从注册中心获取)
    │       ├── CachingServiceInstanceListSupplier (缓存 N 秒)
    │       └── ZonePreferenceServiceInstanceListSupplier (同 Zone 优先)
    │
    ├── ReactorLoadBalancer.choose() 选择实例
    │
    ├── 请求执行
    │
    └── 失败时可能触发重试（选择下一个实例）
```

# 10. 常见问题

**Q：服务下线后还能调通？**

实例列表缓存 TTL 默认 35s，在缓存失效前仍可能路由到已下线的实例。配合重试机制可缓解。

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 5s  # 缩短缓存时间
      retry:
        enabled: true
```

**Q：如何指定客户端不经过 LoadBalancer？**

```java
@FeignClient(name = "user-service", url = "http://localhost:8080")
```

或设置：

```yaml
spring:
  cloud:
    loadbalancer:
      enabled: false  # 全局禁用
```
