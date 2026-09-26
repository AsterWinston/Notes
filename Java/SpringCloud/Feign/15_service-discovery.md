
# 1. 概述

Spring Cloud OpenFeign 与**服务注册中心**（Nacos、Eureka、Consul、Zookeeper）深度集成。核心链路：

```
@FeignClient(name = "user-service")
    │
    │  1. 从注册中心获取实例列表
    ▼
DiscoveryClient (NacosDiscoveryClient / EurekaDiscoveryClient / ...)
    │
    │  2. 传递实例列表给 LoadBalancer
    ▼
ServiceInstanceListSupplier (DiscoveryClientServiceInstanceListSupplier)
    │
    │  3. LoadBalancer 选择实例
    ▼
FeignBlockingLoadBalancerClient
    │
    │  4. 发送 HTTP 到实际 IP:Port
    ▼
目标服务实例
```

# 2. Nacos 集成

## 2.1 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

## 2.2 配置

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: dev              # 命名空间（隔离环境）
        group: DEFAULT_GROUP        # 分组
        cluster-name: beijing       # 集群名（同集群优先调用）
        weight: 10                  # 权重（配合 NacosLoadBalancer）
        ephemeral: true             # 是否临时实例
        metadata:
          version: v2.0             # 自定义元数据（灰度/版本路由）
```

## 2.3 使用

```java
@FeignClient(name = "user-service")  // name = 注册中心中的服务名
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id);
}
```

无需任何额外代码，Feign 自动通过 Nacos 发现 `user-service` 的实例。

# 3. Eureka 集成

## 3.1 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

## 3.2 配置

```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    metadata-map:
      zone: zone1
      version: v2.0
```

# 4. Consul 集成

## 4.1 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

## 4.2 配置

```yaml
spring:
  application:
    name: order-service
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        instance-id: ${spring.application.name}:${random.value}
        health-check-path: /actuator/health
        health-check-interval: 15s
        tags:
          - version=v2.0
```

# 5. DiscoveryClient 接口

```java
public interface DiscoveryClient {

    String description();

    // 获取所有服务实例
    List<ServiceInstance> getInstances(String serviceId);

    // 获取所有注册的服务名
    List<String> getServices();

    // 从注册中心获取服务实例（响应式，LoadBalancer 使用）
    default Flux<ServiceInstance> getInstancesReactive(String serviceId) {
        return Flux.fromIterable(getInstances(serviceId));
    }
}
```

各注册中心的实现：

| 注册中心 | DiscoveryClient 实现 |
|----------|---------------------|
| Nacos | `NacosDiscoveryClient` |
| Eureka | `EurekaDiscoveryClient` |
| Consul | `ConsulDiscoveryClient` |
| Zookeeper | `ZookeeperDiscoveryClient` |

# 6. ServiceInstanceListSupplier 链

```

Request
  │
  ▼
DiscoveryClientServiceInstanceListSupplier   (从注册中心获取实例)
  │
  ▼
CachingServiceInstanceListSupplier           (缓存实例列表)
  │
  ▼
ZonePreferenceServiceInstanceListSupplier    (同 Zone 过滤)
  │
  ▼
HealthCheckServiceInstanceListSupplier       (健康检查过滤，部分注册中心)
  │
  ▼
ReactorLoadBalancer.choose()
```

## 6.1 自定义 Supplier 链

```java
@Configuration
public class CustomSupplierConfig {

    @Bean
    public ServiceInstanceListSupplier discoveryClientSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withCaching()
                .withZonePreference()
                .withHealthChecks()
                .withHints()       // 基于 Hint 过滤（灰度场景）
                .build(context);
    }
}
```

# 7. 灰度发布 / 多版本路由

通过注册中心的自定义元数据 + 自定义 LoadBalancer 实现：

```java
@Component
public class GrayRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 传入版本标记
        String version = getRequestVersion();
        if (version != null) {
            template.header("X-Service-Version", version);
        }
    }
}
```

```java
public class VersionBasedLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // 从 RequestDataContext 读取 Hint
        RequestDataContext ctx = (RequestDataContext) request.getContext();
        String version = ctx.getClientRequest().getHeaders()
                .getFirst("X-Service-Version");

        return supplier.get(request).next().map(instances -> {
            List<ServiceInstance> matched = instances.stream()
                    .filter(i -> version == null ||
                            version.equals(i.getMetadata().get("version")))
                    .collect(Collectors.toList());
            if (matched.isEmpty()) matched = instances; // fallback
            int idx = ThreadLocalRandom.current().nextInt(matched.size());
            return new DefaultResponse(matched.get(idx));
        });
    }
}
```

# 8. 注册中心健康检查与下线

| 注册中心 | 检查机制 | 下线感知延迟 |
|----------|----------|-------------|
| Nacos | 临时实例心跳 (5s) + 持久实例主动探测 | 15s 内 |
| Eureka | 客户端心跳 (30s) + 自我保护 | 90s（默认） |
| Consul | Agent 健康检查 | 及时 |

**影响**：服务下线后，在 `CachingServiceInstanceListSupplier` 缓存过期前仍可能路由到已下线实例。建议：

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 5s     # 缩短缓存，加速感知
      retry:
        enabled: true
        retry-on-all-operations: false  # 仅 GET 重试
    nacos:
      discovery:
        watch-delay: 1000  # Nacos Watch 延迟
```

# 9. 多注册中心

```java
@Configuration
public class MultiRegistryLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> multiRegistryBalancer(
            Environment env,
            LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);

        // 从 Nacos 获取实例
        ServiceInstanceListSupplier nacosSupplier = nacosSupplier(name);
        // 从 Eureka 获取实例
        ServiceInstanceListSupplier eurekaSupplier = eurekaSupplier(name);

        // 合并两个注册中心的实例
        ServiceInstanceListSupplier merged = new MergedSupplier(nacosSupplier, eurekaSupplier);

        return new RoundRobinLoadBalancer(
                ObjectProvider.of(merged), name
        );
    }
}
```

# 10. 常见问题

**Q：服务名能直接解析但一直调不通？**

检查注册中心的实例是否健康（`status: UP` / `healthy: true`）。在 Nacos 控制台查看服务详情。

**Q：@FeignClient name 与服务提供方的 `spring.application.name` 不一致？**

必须完全一致，Feign 通过 name 从 DiscoveryClient 查找服务实例。

**Q：如何指定调用特定实例（调试场景）？**

在 `@FeignClient` 中指定 `url` 跳过注册中心：

```java
@FeignClient(name = "user-service", url = "http://192.168.1.10:8080")
```
