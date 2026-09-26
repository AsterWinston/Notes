
# 1. 负载均衡在网关中的角色

API 网关作为微服务流量的统一入口，负载均衡是其核心功能之一。Spring Cloud Gateway 通过 `lb://` 协议前缀与 Spring Cloud LoadBalancer 集成，实现客户端负载均衡。

```
客户端请求 → Gateway → LoadBalancer → ServiceInstance A
                                    → ServiceInstance B
                                    → ServiceInstance C
```

# 2. 核心组件源码解析

## 2.1 ReactiveLoadBalancerClientFilter

```java
public class ReactiveLoadBalancerClientFilter implements GlobalFilter, Ordered {

    public static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;

    private final LoadBalancerClientFactory clientFactory;
    private final LoadBalancerProperties properties;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String schemePrefix = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR);

        // 非 lb 协议直接跳过
        if (url == null || (!"lb".equals(url.getScheme())
                && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }

        // 将 lb://service-id 与原始 URI 合并后保存
        URI originalUri = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR);

        // 提取 serviceId
        String serviceId = url.getHost();
        if (serviceId == null) {
            throw new IllegalArgumentException("...");
        }

        // 设置 lb 相关的生命周期
        Set<LoadBalancerLifecycle> supportedLifecycleCallbacks = ...;

        // 选择服务实例
        return choose(serviceId, exchange)
            .flatMap(response -> {
                ServiceInstance instance = response.getServer();
                if (instance == null) {
                    throw new NotFoundException("Unable to find instance for " + serviceId);
                }

                // 重构 URI：将 host:port 替换为实际实例
                URI requestUrl = reconstructURI(instance, url);
                exchange.getAttributes().put(
                    ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR, requestUrl);

                // 处理 HTTPS
                if (instance.isSecure()) {
                    exchange.getAttributes().put(
                        ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR, "https");
                }

                return chain.filter(exchange);
            });
    }

    private URI reconstructURI(ServiceInstance instance, URI original) {
        return LoadBalancerUriTools.reconstructURI(instance, original);
    }
}
```

**处理流程**：

1. 从 `GATEWAY_REQUEST_URL_ATTR` 获取目标 URL（如 `lb://user-service/api/users`）
2. 检查 scheme 是否为 `lb` 或包含 `lb` 前缀
3. 从 URL 的 host 部分提取 serviceId
4. 通过 `ReactorLoadBalancer` 选择服务实例
5. 用实际的 `host:port` 替换 `lb://serviceId`
6. 继续过滤器链，由 `NettyRoutingFilter` 发送请求

## 2.2 Spring Cloud LoadBalancer 核心接口

```java
// 核心负载均衡器接口
public interface ReactorLoadBalancer<T> {
    Mono<Response<T>> choose(Request request);
    Mono<Response<T>> choose();
}

// 服务实例接口
public interface ServiceInstance {
    String getServiceId();
    String getHost();
    int getPort();
    boolean isSecure();
    URI getUri();
    Map<String, String> getMetadata();
}

// 请求上下文
public class DefaultRequest<T> implements Request<T> {
    private final T context;
    private final Map<String, Object> hints;
}

// 响应
public class Response<T> {
    private final T server;
    private final boolean hasServer;
}
```

## 2.3 内置负载均衡策略

```java
// 轮询策略（默认）
public class RoundRobinLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {
    private final AtomicInteger position = new AtomicInteger(0);

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // 从服务发现获取服务实例列表
        Flux<ServiceInstance> serviceInstances = serviceInstanceListSupplier
            .get(request);
        return serviceInstances
            .collectList()
            .map(instances -> {
                if (instances.isEmpty()) return new Response<>(null);
                int pos = Math.abs(this.position.incrementAndGet());
                return new Response<>(instances.get(pos % instances.size()));
            });
    }
}

// 随机策略
public class RandomLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {
    private final Random random = new Random();

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplier.get(request)
            .collectList()
            .map(instances -> {
                if (instances.isEmpty()) return new Response<>(null);
                return new Response<>(instances.get(random.nextInt(instances.size())));
            });
    }
}
```

# 3. 负载均衡策略配置

## 3.1 配置文件方式

```yaml
spring:
  cloud:
    loadbalancer:
      # 全局策略
      client:
        enabled: true
      # 按服务配置策略
      ribbon:
        enabled: false          # 关闭 Ribbon（Spring Cloud 2020+ 默认关闭）

      # 为特定服务指定策略
      configurations:
        user-service:
          strategy: random       # random / round-robin
        order-service:
          strategy: round-robin
```

## 3.2 Java 代码配置

```java
@Configuration
public class LoadBalancerConfiguration {

    // 全局随机策略
    @Bean
    public ReactorLoadBalancer<ServiceInstance> reactorLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}

// 按服务分别配置策略
@Configuration
@LoadBalancerClient(name = "user-service", configuration = UserServiceLBConfiguration.class)
public class UserServiceLBConfiguration {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> userServiceLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}
```

# 4. 自定义负载均衡策略

## 4.1 权重负载均衡器

```java
public class WeightedLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
    private final String serviceId;
    private final Random random = new Random();

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplierProvider.get(request)
            .collectList()
            .map(this::weightedChoose);
    }

    private Response<ServiceInstance> weightedChoose(List<ServiceInstance> instances) {
        if (instances.isEmpty()) return new Response<>(null);

        // 从实例的 metadata 中读取权重
        int totalWeight = instances.stream()
            .mapToInt(this::getWeight)
            .sum();

        if (totalWeight <= 0) {
            // 没有权重配置，退化为随机
            return new Response<>(instances.get(random.nextInt(instances.size())));
        }

        int target = random.nextInt(totalWeight);
        int cumulative = 0;
        for (ServiceInstance instance : instances) {
            cumulative += getWeight(instance);
            if (target < cumulative) {
                return new Response<>(instance);
            }
        }
        return new Response<>(instances.get(instances.size() - 1));
    }

    private int getWeight(ServiceInstance instance) {
        String weight = instance.getMetadata().get("weight");
        return weight != null ? Integer.parseInt(weight) : 1;
    }
}
```

**实例注册时的元数据配置**（Nacos）：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          weight: 10
```

## 4.2 最小连接数负载均衡器

```java
public class LeastConnectionLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    private final Map<String, AtomicInteger> connectionCounts = new ConcurrentHashMap<>();

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplierProvider.get(request)
            .collectList()
            .map(instances -> {
                if (instances.isEmpty()) return new Response<>(null);
                // 选择连接数最少的实例
                return instances.stream()
                    .min(Comparator.comparingInt(
                        inst -> connectionCounts
                            .computeIfAbsent(getKey(inst), k -> new AtomicInteger(0))
                            .get()))
                    .map(instance -> {
                        connectionCounts.computeIfAbsent(getKey(instance), k -> new AtomicInteger(0))
                            .incrementAndGet();
                        return new Response<>(instance);
                    })
                    .orElse(new Response<>(null));
            });
    }

    // 需要在 POST 阶段释放连接计数：connectionCounts.get(key).decrementAndGet()

    private String getKey(ServiceInstance instance) {
        return instance.getHost() + ":" + instance.getPort();
    }
}
```

# 5. 服务发现集成

## 5.1 Nacos 集成

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev
        group: DEFAULT_GROUP
        metadata:
          version: v1
```

## 5.2 Eureka 集成

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

## 5.3 Consul 集成

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        prefer-ip-address: true
        health-check-path: /actuator/health
```

# 6. 服务发现自动路由

## 6.1 配置

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          # 服务 ID 转小写
          lower-case-service-id: true
          # 路由路径表达式
          url-expression: "'lb://'+serviceId"
          # 自动为每个服务添加的过滤器
          filters:
            - StripPrefix=1
```

## 6.2 实现原理

```java
public class DiscoveryClientRouteDefinitionLocator
        implements RouteDefinitionLocator, Ordered {

    private final DiscoveryClient discoveryClient;
    private final GatewayDiscoveryClientAutoConfiguration.DiscoveryLocatorProperties properties;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(discoveryClient.getServices())
            .map(serviceId -> {
                RouteDefinition routeDef = new RouteDefinition();
                routeDef.setId("ReactiveCompositeDiscoveryClient_" + serviceId);

                // 自动创建路径断言
                String lowerCaseId = properties.isLowerCaseServiceId()
                    ? serviceId.toLowerCase() : serviceId;
                routeDef.getPredicates().add(new PredicateDefinition(
                    "Path=" + lowerCaseId + "/**"));

                // 自动设置 lb:// 协议
                routeDef.setUri(URI.create(properties.getUrlExpression()));
                // "lb://" + serviceId

                // 自动添加过滤器
                if (properties.getFilters() != null) {
                    for (FilterDefinition filter : properties.getFilters()) {
                        routeDef.getFilters().add(filter);
                    }
                }

                return routeDef;
            });
    }
}
```

**自动生成的路由**：

```yaml
# 自动创建的路由等价于以下配置
- id: ReactiveCompositeDiscoveryClient_user-service
  uri: lb://user-service
  predicates:
    - Path=/user-service/**
  filters:
    - StripPrefix=1
```

# 7. 负载均衡的请求流程

```
请求: GET /api/users

1. RoutePredicateHandlerMapping 匹配路由
   Route = { id: user_route, uri: lb://user-service }

2. RouteToRequestUrlFilter 构建 URL
   GATEWAY_REQUEST_URL_ATTR = lb://user-service/api/users

3. ReactiveLoadBalancerClientFilter 负载均衡
   serviceId = "user-service"
   choose() → 返回 ServiceInstance { host: 192.168.1.10, port: 8080 }

4. 重构 URI
   GATEWAY_REQUEST_URL_ATTR = http://192.168.1.10:8080/api/users

5. NettyRoutingFilter 发送 HTTP 请求到 192.168.1.10:8080
```

# 8. 负载均衡的最佳实践

## 8.1 连接池配置

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          type: fixed
          max-connections: 500          # 最大连接数
          max-idle-time: 3000           # 最大空闲时间（ms）
          max-life-time: 60000          # 连接最大生命周期（ms）
          acquire-timeout: 45000        # 获取连接超时（ms）
```

## 8.2 健康检查与实例排除

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        interval: 30s                   # 健康检查间隔
        path: /actuator/health          # 健康检查路径
      cache:
        enabled: true
        ttl: 5s                         # 实例列表缓存时间
```

## 8.3 服务实例元数据的利用

实例元数据可以携带环境、版本、区域等信息，用于精细化路由：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: v2
          region: us-east-1
          canary: true
```

```java
// 在自定义负载均衡策略中读取元数据
ServiceInstance instance = ...;
String version = instance.getMetadata().get("version");
String region = instance.getMetadata().get("region");
```

## 8.4 注意事项

1. **服务名大小写**：建议统一使用小写服务名
2. **404 vs 503**：`loadbalancer.use404` 配置控制无可用实例时的返回状态码
3. **预热与缓存**：配置合理的服务实例列表缓存时间
4. **优雅上下线**：配合服务发现的健康检查和权重调整实现优雅上下线
