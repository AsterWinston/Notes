
# 1. 集成概述

Spring Cloud Gateway 集成 Nacos 后，可以实现**动态路由**——路由规则存储在 Nacos 配置中心，修改路由无需重启 Gateway 即可实时生效。

## 1.1 核心能力

- 动态路由：路由规则从 Nacos 配置中心实时获取
- 灰度路由：基于 Nacos 实例元数据实现流量灰度
- 动态负载均衡：基于 Nacos 权重自动调整流量分配
- 服务发现集成：Gateway 直接通过 Nacos 发现下游服务

# 2. 基础集成配置

## 2.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

## 2.2 应用配置

```yaml
# bootstrap.yml
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev
      config:
        server-addr: 127.0.0.1:8848
        namespace: dev
        file-extension: yaml
    gateway:
      discovery:
        locator:
          enabled: true          # 开启服务发现路由
          lower-case-service-id: true  # 服务名小写
      routes:
        - id: order-route
          uri: lb://order-service  # 负载均衡方式
          predicates:
            - Path=/order/**
          filters:
            - StripPrefix=1
        - id: user-route
          uri: lb://user-service
          predicates:
            - Path=/user/**
          filters:
            - StripPrefix=1
```

# 3. 动态路由实现

## 3.1 Nacos 存储路由配置

在 Nacos 中创建配置 `gateway-routes.json`：

```json
[
  {
    "id": "order-service-route",
    "uri": "lb://order-service",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "pattern": "/order/**"
        }
      }
    ],
    "filters": [
      {
        "name": "StripPrefix",
        "args": {
          "parts": 1
        }
      },
      {
        "name": "RequestRateLimiter",
        "args": {
          "redis-rate-limiter.replenishRate": 10,
          "redis-rate-limiter.burstCapacity": 20
        }
      }
    ],
    "metadata": {
      "version": "v2",
      "gray": "true"
    }
  },
  {
    "id": "user-service-route",
    "uri": "lb://user-service",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "pattern": "/user/**"
        }
      }
    ],
    "filters": [
      {
        "name": "StripPrefix",
        "args": {
          "parts": 1
        }
      }
    ],
    "order": 0
  }
]
```

## 3.2 动态路由监听实现

```java
@Component
public class DynamicRouteService implements CommandLineRunner {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    @Autowired
    private NacosConfigManager nacosConfigManager;

    // Nacos 路由配置的 Data ID
    private static final String ROUTE_DATA_ID = "gateway-routes.json";
    private static final String ROUTE_GROUP = "DEFAULT_GROUP";

    @Override
    public void run(String... args) {
        // 1. 首次加载路由
        String routesJson = nacosConfigManager.getConfigService()
            .getConfig(ROUTE_DATA_ID, ROUTE_GROUP, 5000);
        refreshRoutes(routesJson);

        // 2. 监听路由变更
        nacosConfigManager.getConfigService().addListener(
            ROUTE_DATA_ID, ROUTE_GROUP, new Listener() {
                @Override
                public Executor getExecutor() {
                    return null;
                }

                @Override
                public void receiveConfigInfo(String configInfo) {
                    // 路由变更时自动刷新
                    refreshRoutes(configInfo);
                }
            });
    }

    private void refreshRoutes(String routesJson) {
        List<RouteDefinition> routeDefinitions = JSON.parseArray(
            routesJson, RouteDefinition.class);

        // 清空现有路由并添加新路由
        reactorRouteDefinitionWriter.getRouteDefinitions()
            .map(RouteDefinition::getId)
            .flatMap(reactorRouteDefinitionWriter::delete)
            .thenMany(Flux.fromIterable(routeDefinitions))
            .flatMap(reactorRouteDefinitionWriter::save)
            .subscribe();
    }
}
```

## 3.3 使用 Spring Cloud Gateway 官方动态路由

```java
@Configuration
public class GatewayDynamicRouteConfig {

    @Autowired
    private NacosConfigManager nacosConfigManager;

    @Bean
    @RefreshScope
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // 基础路由
            .route("order-route", r -> r
                .path("/order/**")
                .uri("lb://order-service"))
            // Nacos 权重负载均衡
            .route("user-route", r -> r
                .path("/user/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Request-Source", "gateway"))
                .uri("lb://user-service"))
            .build();
    }
}
```

# 4. 灰度路由

## 4.1 基于元数据的灰度策略

```java
@Component
public class GrayLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        HttpHeaders headers = ((ReactiveRequestContext) request.getContext())
            .getExchange().getRequest().getHeaders();

        // 1. 从请求头获取灰度标识
        String grayTag = headers.getFirst("X-Gray-Tag");
        String version = headers.getFirst("X-Version");

        // 2. 获取所有服务实例
        List<ServiceInstance> instances = discoveryClient
            .getInstances(((DefaultRequest) request).getServiceId());

        // 3. 灰度匹配
        List<ServiceInstance> matchedInstances = instances.stream()
            .filter(instance -> {
                if (StringUtils.isNotBlank(version)) {
                    // 版本路由
                    return version.equals(instance.getMetadata().get("version"));
                }
                if ("true".equals(grayTag)) {
                    // 灰度标记：路由到灰度实例
                    return "true".equals(instance.getMetadata().get("gray"));
                }
                // 非灰度请求：只路由到非灰度实例
                return !"true".equals(instance.getMetadata().get("gray"));
            })
            .collect(Collectors.toList());

        // 4. 如果没有匹配的实例，回退到普通实例
        if (matchedInstances.isEmpty()) {
            matchedInstances = instances;
        }

        // 5. 加权随机选择一个实例
        ServiceInstance selected = weightedRandom(matchedInstances);
        return Mono.just(new DefaultResponse(selected));
    }

    private ServiceInstance weightedRandom(List<ServiceInstance> instances) {
        double totalWeight = instances.stream()
            .mapToDouble(i -> Double.parseDouble(
                i.getMetadata().getOrDefault("nacos.weight", "1.0")))
            .sum();
        double random = ThreadLocalRandom.current().nextDouble(totalWeight);

        for (ServiceInstance instance : instances) {
            random -= Double.parseDouble(
                instance.getMetadata().getOrDefault("nacos.weight", "1.0"));
            if (random <= 0) {
                return instance;
            }
        }
        return instances.get(0);
    }
}
```

## 4.2 灰度配置示例

```yaml
# 注册灰度实例（特定版本）
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        metadata:
          version: v2.0.0    # 版本标识
          gray: "true"       # 灰度标识
          zone: gray-zone    # 灰度可用区
```

## 4.3 Gateway 过滤器实现灰度

```java
@Component
public class GrayGatewayFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // 根据请求特征判断是否灰度用户
        String userId = request.getHeaders().getFirst("X-User-Id");
        boolean isGrayUser = isGrayUser(userId);

        ServerHttpRequest mutatedRequest = request.mutate()
            .header("X-Gray-Tag", isGrayUser ? "true" : "false")
            .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }

    private boolean isGrayUser(String userId) {
        // 从 Nacos 配置中获取灰度用户白名单
        // 或根据 userId hash 百分比判断
        if (userId == null) return false;
        int hash = Math.abs(userId.hashCode() % 100);
        return hash < 20; // 20% 的用户走灰度
    }

    @Override
    public int getOrder() {
        return -100; // 最高优先级
    }
}
```

# 5. 限流集成

## 5.1 Gateway + Nacos + Sentinel 限流

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080
      datasource:
        gw-flow:
          nacos:
            server-addr: 127.0.0.1:8848
            data-id: gw-flow-rules.json
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: gw-flow
        gw-api-group:
          nacos:
            server-addr: 127.0.0.1:8848
            data-id: gw-api-group-rules.json
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: gw-api-group
```

## 5.2 Nacos 中的限流规则

```json
[
  {
    "resource": "order-api",
    "resourceMode": 0,
    "grade": 1,
    "count": 100,
    "intervalSec": 1,
    "controlBehavior": 0,
    "burst": 0,
    "maxQueueingTimeoutMs": 500
  },
  {
    "resource": "user-api",
    "resourceMode": 0,
    "grade": 1,
    "count": 200,
    "intervalSec": 1
  }
]
```

# 6. 跨域处理

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders:
              - Content-Type
              - Authorization
              - X-Gray-Tag
            allowCredentials: true
            maxAge: 3600
```

# 7. 请求日志与追踪

```java
@Component
public class RequestLoggingFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String traceId = request.getHeaders().getFirst("X-Trace-Id");

        if (traceId == null) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }

        long startTime = System.currentTimeMillis();

        // 记录请求日志
        log.info("Gateway Request: method={}, path={}, traceId={}",
            request.getMethod(), request.getPath(), traceId);

        // 添加 traceId 到请求头
        ServerHttpRequest mutatedRequest = request.mutate()
            .header("X-Trace-Id", traceId)
            .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build())
            .then(Mono.fromRunnable(() -> {
                long duration = System.currentTimeMillis() - startTime;
                HttpStatus status = exchange.getResponse().getStatusCode();
                log.info("Gateway Response: path={}, status={}, duration={}ms, traceId={}",
                    request.getPath(), status, duration, traceId);
            }));
    }

    @Override
    public int getOrder() {
        return -200;
    }
}
```

# 8. 高可用网关架构

```
              ┌─────────────┐
              │    DNS/VIP   │
              └──────┬──────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │Gateway-1 │ │Gateway-2 │ │Gateway-3 │
  │ (Nacos   │ │ (Nacos   │ │ (Nacos   │
  │  Client) │ │  Client) │ │  Client) │
  └────┬─────┘ └────┬─────┘ └────┬─────┘
       └────────────┼────────────┘
                    │
         ┌──────────▼──────────┐
         │   Nacos Cluster     │
         │  (动态路由 + 服务发现) │
         └──────────┬──────────┘
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
┌────────┐    ┌──────────┐   ┌──────────┐
│Order   │    │User      │   │Payment   │
│Service │    │Service   │   │Service   │
└────────┘    └──────────┘   └──────────┘
```

# 9. 生产环境建议

1. **Gateway 独立部署**：Gateway 作为一个独立微服务部署，与业务服务分离
2. **路由配置管理**：使用 Nacos 配置中心统一管理路由规则，实现动态路由
3. **限流熔断**：在 Gateway 层实现全局限流，使用 Sentinel + Nacos 持久化规则
4. **灰度发布**：利用 Nacos 元数据实现灰度策略，结合 Gateway 过滤器做流量染色
5. **监控告警**：Gateway 接入 Prometheus + Grafana，监控 QPS、延迟、错误率
6. **高可用**：Gateway 集群部署（3+ 节点），Nginx 做四层/七层负载均衡
