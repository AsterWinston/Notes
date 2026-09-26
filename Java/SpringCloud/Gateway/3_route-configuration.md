
# 1. 路由定义模型

## 1.1 Route 核心模型

`Route` 是网关处理请求的基本单位，定义于 `org.springframework.cloud.gateway.route.Route`：

```java
public class Route implements Ordered {
    private final String id;                          // 路由 ID
    private final URI uri;                            // 目标 URI
    private final int order;                          // 优先级
    private final AsyncPredicate<ServerWebExchange> predicate; // 断言
    private final List<GatewayFilter> gatewayFilters;           // 过滤器
    private final Map<String, Object> metadata;       // 元数据
}
```

`RouteDefinition` 是 Route 的可序列化版本，用于配置解析：

```java
public class RouteDefinition {
    private String id;
    private URI uri;
    private int order;
    private List<PredicateDefinition> predicates = new ArrayList<>();
    private List<FilterDefinition> filters = new ArrayList<>();
    private Map<String, Object> metadata = new HashMap<>();
}
```

## 1.2 PredicateDefinition

```java
public class PredicateDefinition {
    private String name;                              // 断言工厂名称
    private Map<String, String> args = new LinkedHashMap<>(); // 参数

    public PredicateDefinition(String text) {
        int eqIdx = text.indexOf('=');
        if (eqIdx <= 0) {
            throw new IllegalArgumentException(...);
        }
        this.name = text.substring(0, eqIdx);
        String argsText = text.substring(eqIdx + 1);
        // 解析逗号分隔的参数，支持转义逗号
        this.args = parseArgs(argsText);
    }
}
```

## 1.3 FilterDefinition

```java
public class FilterDefinition {
    private String name;                              // 过滤器工厂名称
    private Map<String, String> args = new LinkedHashMap<>(); // 参数
}
```

# 2. 路由定义方式

## 2.1 YAML 配置方式

最常用的配置方式，通过 `application.yml` 定义：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
            - Method=GET
          filters:
            - StripPrefix=2
            - AddRequestHeader=X-Source, gateway
          order: 0
          metadata:
            response-timeout: 5000

        - id: order_service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@userKeyResolver}"
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

**支持的 URI 协议**：

| 协议 | 说明 | 示例 |
|------|------|------|
| `http://` | HTTP 转发 | `http://localhost:8081` |
| `https://` | HTTPS 转发 | `https://api.example.com` |
| `lb://` | 负载均衡转发 | `lb://user-service` |
| `ws://` | WebSocket 转发 | `ws://localhost:8082` |
| `wss://` | WebSocket TLS | `wss://websocket.example.com` |
| `forward://` | 本地转发（不经过网络） | `forward:/fallback` |

## 2.2 Java DSL 配置方式

通过 `RouteLocatorBuilder` 以 Java DSL 风格配置：

```java
@Configuration
public class RouteConfiguration {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // 基础路由
            .route("user_service", r -> r
                .path("/api/user/**")
                .and().method("GET")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Source", "gateway"))
                .uri("lb://user-service"))

            // 带元数据和 order 的路由
            .route("order_service", r -> r
                .path("/api/order/**")
                .filters(f -> f
                    .circuitBreaker(config -> config
                        .setName("orderCB")
                        .setFallbackUri("forward:/fallback")))
                .order(1)
                .metadata("response-timeout", 5000)
                .uri("lb://order-service"))

            // 权重路由
            .route("canary_v1", r -> r
                .weight("canary", 80)
                .path("/api/**")
                .uri("http://v1.example.com"))
            .route("canary_v2", r -> r
                .weight("canary", 20)
                .path("/api/**")
                .uri("http://v2.example.com"))

            .build();
    }
}
```

**Java DSL 风格的方法链**：

```text
.predicate()  ← 断言集合
    .path()                     PathRoutePredicateFactory
    .method()                   MethodRoutePredicateFactory
    .header()                   HeaderRoutePredicateFactory
    .cookie()                   CookieRoutePredicateFactory
    .query()                    QueryRoutePredicateFactory
    .host()                     HostRoutePredicateFactory
    .remoteAddr()               RemoteAddrRoutePredicateFactory
    .weight()                   WeightRoutePredicateFactory
    .after()                    AfterRoutePredicateFactory
    .before()                   BeforeRoutePredicateFactory
    .between()                  BetweenRoutePredicateFactory
    .and()                      AND 组合
    .or()                       OR 组合
    .negate()                   取反

.filters()  ← 过滤器集合（类型安全）
    .addRequestHeader()
    .addRequestParameter()
    .addResponseHeader()
    .circuitBreaker()
    .dedupeResponseHeader()
    .fallbackHeaders()
    .mapRequestHeader()
    .modifyRequestBody()
    .modifyResponseBody()
    .prefixPath()
    .preserveHostHeader()
    .redirect()
    .removeRequestHeader()
    .removeRequestParameter()
    .removeResponseHeader()
    .requestRateLimiter()
    .requestSize()
    .retry()
    .rewritePath()
    .rewriteResponseHeader()
    .saveSession()
    .secureHeaders()
    .setPath()
    .setRequestHeader()
    .setRequestHostHeader()
    .setResponseHeader()
    .setStatus()
    .stripPrefix()
    .filter()                  添加自定义 GatewayFilter
```

## 2.3 Bean 配置方式

通过 `RouteDefinitionWriter` 编程式注册路由：

```java
@Component
public class RouteRegistrationBean implements ApplicationRunner {
    private final RouteDefinitionWriter routeDefinitionWriter;

    public RouteRegistrationBean(RouteDefinitionWriter routeDefinitionWriter) {
        this.routeDefinitionWriter = routeDefinitionWriter;
    }

    @Override
    public void run(ApplicationArguments args) {
        // 构建路由定义
        RouteDefinition definition = new RouteDefinition();
        definition.setId("dynamic_route");
        definition.setUri(URI.create("lb://dynamic-service"));

        // 添加断言
        PredicateDefinition pathPredicate = new PredicateDefinition();
        pathPredicate.setName("Path");
        pathPredicate.addArg("pattern", "/dynamic/**");
        definition.getPredicates().add(pathPredicate);

        PredicateDefinition methodPredicate = new PredicateDefinition();
        methodPredicate.setName("Method");
        methodPredicate.addArg("method", "GET,POST");
        definition.getPredicates().add(methodPredicate);

        // 添加过滤器
        FilterDefinition stripFilter = new FilterDefinition();
        stripFilter.setName("StripPrefix");
        stripFilter.addArg("parts", "1");
        definition.getFilters().add(stripFilter);

        // 注册路由
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
    }
}
```

# 3. RouteLocator 体系源码解析

## 3.1 类层次结构

```
RouteLocator（接口：获取路由列表）
    │
    ├── CachingRouteLocator（缓存装饰器）
    │       └── 包装其他 RouteLocator，提供缓存和刷新能力
    │
    ├── RouteDefinitionRouteLocator（核心实现）
    │       └── 将 RouteDefinition 转换为 Route
    │       └── 使用 RoutePredicateFactory 创建断言
    │       └── 使用 GatewayFilterFactory 创建过滤器
    │
    ├── DiscoveryClientRouteDefinitionLocator
    │       └── 从服务发现中心自动创建 RouteDefinition
    │
    └── CompositeRouteDefinitionLocator（组合模式）
            └── 组合多个 RouteDefinitionLocator
```

## 3.2 RouteDefinitionRouteLocator 核心源码

```java
public class RouteDefinitionRouteLocator implements RouteLocator, BeanFactoryAware {
    private final RouteDefinitionLocator routeDefinitionLocator;
    private final Map<String, RoutePredicateFactory> predicates;
    private final Map<String, GatewayFilterFactory> gatewayFilterFactories;

    @Override
    public Flux<Route> getRoutes() {
        return this.routeDefinitionLocator.getRouteDefinitions()
            .map(this::convertToRoute)            // RouteDefinition → Route
            .onErrorContinue(...);                 // 单个路由异常不影响其他路由
    }

    private Route convertToRoute(RouteDefinition definition) {
        // 1. 将 PredicateDefinition → AsyncPredicate
        AsyncPredicate<ServerWebExchange> predicate = combinePredicates(definition);

        // 2. 将 FilterDefinition → GatewayFilter
        List<GatewayFilter> filters = getFilters(definition);

        // 3. 构建 Route
        return Route.async(definition.getId(), definition.getUri(), definition.getOrder(),
            predicate, filters, definition.getMetadata());
    }

    private AsyncPredicate<ServerWebExchange> combinePredicates(RouteDefinition definition) {
        // 将所有断言用 AND 组合
        List<PredicateDefinition> predicateDefinitions = definition.getPredicates();
        AsyncPredicate<ServerWebExchange> predicate = AsyncPredicate.from(exchange -> true);

        for (PredicateDefinition predicateDefinition : predicateDefinitions) {
            RoutePredicateFactory<Object> factory = lookup(predicateDefinition);
            // 创建断言配置
            Object config = factory.newConfig();
            Map<String, String> args = predicateDefinition.getArgs();
            if (args != null && !args.isEmpty()) {
                factory.beforeApply(config, () -> args); // Shortcut 配置处理
            }
            // 应用工厂创建断言
            predicate = predicate.and(factory.applyAsync(config));
        }
        return predicate;
    }

    private List<GatewayFilter> getFilters(RouteDefinition definition) {
        List<GatewayFilter> filters = new ArrayList<>();
        for (FilterDefinition filterDefinition : definition.getFilters()) {
            GatewayFilterFactory<Object> factory = lookup(filterDefinition);
            Object config = factory.newConfig();
            Map<String, String> args = filterDefinition.getArgs();
            if (args != null && !args.isEmpty()) {
                for (Map.Entry<String, String> entry : args.entrySet()) {
                    factory.beforeApply(config, entry.getKey(), entry.getValue());
                }
            }
            filters.add(factory.apply(config));
        }
        return filters;
    }
}
```

## 3.3 CachingRouteLocator 源码

```java
public class CachingRouteLocator implements RouteLocator {
    private final RouteLocator delegate;
    private Flux<Route> routes;
    private final EventBus eventBus;

    public CachingRouteLocator(RouteLocator delegate) {
        this.delegate = delegate;
        this.eventBus = new EventBus();
        // 监听刷新事件
        this.eventBus.register(this, RefreshRoutesEvent.class,
            (event, subscriber) -> refresh());
    }

    @Override
    public Flux<Route> getRoutes() {
        // 双重检查锁定模式实现缓存
        if (this.routes == null) {
            synchronized (this) {
                if (this.routes == null) {
                    this.routes = this.delegate.getRoutes().cache();
                }
            }
        }
        return this.routes;
    }

    public void refresh() {
        synchronized (this) {
            this.routes = null; // 清空缓存，下次访问时重新加载
        }
    }
}
```

# 4. 路由匹配原理

## 4.1 RoutePredicateHandlerMapping 匹配流程

```java
public Mono<Route> lookupRoute(ServerWebExchange exchange) {
    return this.routeLocator.getRoutes()                    // 获取所有路由
        .concatMap(route -> Mono.just(route)
            .filterWhen(r -> r.getPredicate().apply(exchange)) // 断言匹配
            .doOnError(e -> logger.debug("...", e))
            .onErrorResume(e -> Mono.empty()))             // 单个断言异常跳过
        .next()                                             // 取第一个匹配结果
        .switchIfEmpty(Mono.error(...));                    // 无匹配则返回 404
}
```

**匹配策略**：
1. 获取所有已加载的路由列表
2. 按 `order` 升序排列（未设置 order 则按定义顺序）
3. 依次对每个路由执行断言组合
4. 取第一个断言结果为 `true` 的路由
5. 匹配失败返回 404

## 4.2 断言组合逻辑

多个断言是 AND 关系：

```yaml
predicates:
  - Path=/api/**          # 路径匹配
  - Method=GET            # 方法匹配
  - Header=X-Version, v1  # Header 匹配
```

等价于 Java 表达式：

```java
Predicate<ServerWebExchange> predicate = pathPredicate
    .and(methodPredicate)
    .and(headerPredicate);
```

## 4.3 路由优先级控制

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: specific
          uri: http://specific
          predicates:
            - Path=/api/v2/**
          order: 0

        - id: general
          uri: http://general
          predicates:
            - Path=/api/**
          order: 1
```

访问 `/api/v2/users`：
1. 先匹配 `specific`（order=0），Path 匹配成功 → 路由到 specific
2. 不会继续匹配 `general`

访问 `/api/v1/users`：
1. specific 的 Path 不匹配（`/api/v2/**`）
2. general 匹配成功 → 路由到 general

# 5. 默认过滤器

## 5.1 default-filters 配置

对所有路由生效的过滤器：

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - AddResponseHeader=X-Gateway-Timestamp, #{T(java.time.Instant).now()}
        - DedupeResponseHeader=Access-Control-Allow-Origin, RETAIN_LAST
      routes:
        - id: service_a
          uri: http://localhost:8081
          predicates:
            - Path=/a/**
```

**执行顺序**：default-filters 中的过滤器与路由级过滤器合并后统一排序执行。

## 5.2 源码分析

```java
public class FilteringWebHandler implements WebHandler {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        List<GatewayFilter> allFilters = new ArrayList<>();
        allFilters.addAll(globalFilters);    // 全局过滤器
        allFilters.addAll(route.getFilters()); // 路由级过滤器
        allFilters.sort(AnnotationAwareOrderComparator.INSTANCE);
        return new DefaultGatewayFilterChain(allFilters).filter(exchange);
    }
}
```

# 6. 路由元数据的应用

## 6.1 在过滤器中读取元数据

```java
@Component
public class MetadataAwareFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route != null) {
            // 读取元数据中的超时配置
            Map<String, Object> metadata = route.getMetadata();
            Number timeout = (Number) metadata.get("response-timeout");
            if (timeout != null) {
                exchange.getAttributes().put(
                    ServerWebExchangeUtils.CLIENT_RESPONSE_TIMEOUT_ATTR,
                    timeout.longValue());
            }
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE;
    }
}
```

## 6.2 内置元数据键

Spring Cloud Gateway 识别以下元数据键：

| 元数据键 | 类型 | 说明 |
|----------|------|------|
| `response-timeout` | Number | 按路由的响应超时时间（ms），负值禁用全局超时 |
| `connect-timeout` | Number | 按路由的连接超时时间（ms） |
| `cors` | CorsConfiguration | 按路由的 CORS 配置 |

# 7. 配置最佳实践

## 7.1 配置分离

将路由配置与主配置分离：

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
# application-dev.yml
spring:
  cloud:
    gateway:
      routes:
        - id: dev_route
          uri: http://localhost:8081
          predicates:
            - Path=/dev/**
```

## 7.2 路由命名规范

```
{service-name}_{function}_{version}
```

```yaml
- id: user_query_v1
- id: order_create_v2
- id: payment_callback
```

## 7.3 服务发现路由自动配置

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
          url-expression: "'lb://'+serviceId"
          filters:
            - StripPrefix=0
```

自动为已注册的服务创建路由：`/SERVICE-ID/**` → `lb://service-id/**`

## 7.4 路由数量与性能

- 路由数量控制在合理范围内（建议 < 1000 条）
- 利用 `order` 将高频路由放在前面
- 使用 `cache` 减少路由解析开销
- 避免过于复杂的断言条件
