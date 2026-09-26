
# 1. 网关在微服务中的角色

API 网关是微服务架构的入口层，承担着路由转发、请求过滤、协议转换、安全认证、限流熔断、日志监控等核心职责。在微服务架构中，网关作为系统的统一入口，屏蔽了后端服务的复杂性，为客户端提供了简洁的 API 接口。

## 1.1 网关的核心功能

- **路由转发**：根据请求路径、Header、参数等条件将请求路由到对应的后端服务
- **请求过滤**：在请求进入后端服务前进行预处理（鉴权、校验、改写等）
- **响应过滤**：在后端服务返回后对响应进行后处理（改写 Header、压缩等）
- **安全防护**：统一认证授权、IP 黑白名单、CORS、防 XSS/CSRF
- **流量控制**：限流、熔断、降级、重试
- **协议转换**：HTTP ↔ gRPC、HTTP ↔ WebSocket 等
- **可观测性**：统一日志、监控指标、链路追踪

# 2. 网关技术演进：Zuul 1.x → Zuul 2.x → Spring Cloud Gateway

## 2.1 Zuul 1.x（Netflix）

Zuul 1.x 基于 Servlet 2.5，使用阻塞 I/O（BIO）模型。每个请求占用一个线程，线程在等待后端响应时被阻塞。在高并发场景下，线程耗尽导致系统吞吐量急剧下降。

```java
// Zuul 1.x 的核心：Servlet 阻塞模型
public class ZuulServlet extends HttpServlet {
    @Override
    public void service(HttpServletRequest req, HttpServletResponse res) {
        try {
            // 每个请求占用一个线程，阻塞等待
            ZuulRunner zuulRunner = new ZuulRunner();
            zuulRunner.init(req, res);
            zuulRunner.preRoute();
            zuulRunner.route();      // 这里会阻塞等待后端响应
            zuulRunner.postRoute();
        } catch (ZuulException e) {
            zuulRunner.error();
        }
    }
}
```

**缺点**：
- 线程模型：同步阻塞，每个请求占一个线程
- 性能瓶颈：线程上下文切换开销大，连接数受限
- 不支持长连接（WebSocket）
- 不支持响应式编程

## 2.2 Zuul 2.x（Netflix）

Zuul 2.x 基于 Netty 实现了异步非阻塞模型，解决了线程阻塞问题。但 Netflix 没有提供 Spring Cloud 集成，社区生态薄弱，落地成本高。

## 2.3 Spring Cloud Gateway

Spring Cloud Gateway 基于 Spring WebFlux + Reactor Netty，采用响应式编程模型，实现了异步非阻塞架构。

```java
// 核心配置：基于非阻塞事件驱动
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("path_route", r -> r.path("/get")
            .uri("http://httpbin.org"))
        .build();
}
```

**核心优势**：
- 异步非阻塞：基于 Reactor Netty 事件循环，少量线程处理大量连接
- 性能卓越：吞吐量远超 Zuul 1.x，接近 Zuul 2.x
- 生态完善：Spring Cloud 原生集成，配置简单
- 功能丰富：30+ 内置过滤器工厂，轻松扩展
- 支持 WebSocket、gRPC 等协议
- 支持响应式编程，与 Spring WebFlux 无缝对接

## 2.4 性能对比

| 指标 | Zuul 1.x | Zuul 2.x | Spring Cloud Gateway |
|------|----------|----------|---------------------|
| 线程模型 | BIO（阻塞） | NIO（非阻塞） | NIO（非阻塞） |
| 底层框架 | Servlet | Netty | Reactor Netty |
| 编程模型 | 同步 | 异步 | 响应式 |
| 吞吐量（基准） | 低 | 高 | 高 |
| WebSocket | 不支持 | 支持 | 支持 |
| gRPC | 不支持 | 不支持 | 支持 |
| Spring Cloud 集成 | 原生 | 需自行集成 | 原生 |
| 社区维护 | 维护模式 | 维护模式 | 活跃 |

# 3. 核心概念：Route、Predicate、Filter

## 3.1 Route（路由）

路由是网关的基本组成单元，包含以下要素：

```java
public class Route implements Ordered {
    private final String id;                    // 路由唯一标识
    private final URI uri;                      // 目标 URI
    private final int order;                    // 路由优先级（值越小优先级越高）
    private final AsyncPredicate<ServerWebExchange> predicate; // 断言
    private final List<GatewayFilter> gatewayFilters;          // 过滤器列表
    private final Map<String, Object> metadata; // 元数据
}
```

**路由的三种配置方式**：

YAML 配置：
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service
          uri: lb://user-service
          predicates:
            - Path=/user/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Request-Source, Gateway
```

Java DSL 配置：
```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("user_service", r -> r
            .path("/user/**")
            .filters(f -> f
                .stripPrefix(1)
                .addRequestHeader("X-Request-Source", "Gateway"))
            .uri("lb://user-service"))
        .build();
}
```

Bean 方式配置：
```java
@Bean
public RouteDefinition userRouteDefinition() {
    RouteDefinition definition = new RouteDefinition();
    definition.setId("user_service");
    definition.setUri(URI.create("lb://user-service"));

    PathRoutePredicateFactory.Config pathConfig = new PathRoutePredicateFactory.Config();
    pathConfig.setPatterns(List.of("/user/**"));
    definition.getPredicates().add(new PredicateDefinition("Path=" + String.join(",", pathConfig.getPatterns())));

    FilterDefinition filterDef = new FilterDefinition();
    filterDef.setName("StripPrefix");
    filterDef.addArg("parts", "1");
    definition.getFilters().add(filterDef);

    return definition;
}
```

## 3.2 Predicate（断言）

断言是 Java 8 的 `Predicate<ServerWebExchange>`，用于匹配请求条件。网关内置了 14 种断言工厂：

```java
// 断言接口定义
@FunctionalInterface
public interface AsyncPredicate<T> {
    Publisher<Boolean> apply(T t);
}

// 使用示例：组合多个断言
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("complex_route", r -> r
            .path("/api/**")                             // Path 断言
            .and().method("GET", "POST")                  // Method 断言
            .and().header("X-Auth-Token", "\\w+")        // Header 断言
            .and().cookie("session", "\\w+")              // Cookie 断言
            .and().query("page", "\\d+")                  // Query 断言
            .uri("lb://api-service"))
        .build();
}
```

## 3.3 Filter（过滤器）

过滤器分为 `GatewayFilter`（路由级）和 `GlobalFilter`（全局级）：

```java
// GatewayFilter 接口
public interface GatewayFilter {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// GlobalFilter 接口
public interface GlobalFilter {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// 过滤器链实现：责任链模式
public class DefaultGatewayFilterChain implements GatewayFilterChain {
    private final List<GatewayFilter> filters;
    private final int index;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange) {
        if (this.index < filters.size()) {
            GatewayFilter filter = filters.get(this.index);
            DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this, this.index + 1);
            return filter.filter(exchange, chain);
        }
        return Mono.empty(); // 链结束，转发请求
    }
}
```

# 4. 请求处理流程源码解析

## 4.1 完整请求链路

```
客户端请求
    │
    ▼
DispatcherHandler（WebFlux 前端控制器）
    │
    ▼
RoutePredicateHandlerMapping.lookupRoute()
    │  └─ 遍历所有 Route，逐个执行 Predicate 匹配
    │  └─ 返回匹配的 Route（按 order 排序）
    ▼
FilteringWebHandler.handle()
    │  └─ 合并 GlobalFilter + 路由级 GatewayFilter
    │  └─ 按 order 排序形成过滤器链
    │  └─ 执行 chain.filter(exchange)
    ▼
GatewayFilterChain 执行
    │  ├─ PRE 阶段：前置过滤器（如鉴权、改写请求）
    │  ├─ ROUTE 阶段：转发到后端服务（由 NettyRoutingFilter 执行）
    │  └─ POST 阶段：后置过滤器（如改写响应、记录日志）
    ▼
后端服务响应返回给客户端
```

## 4.2 RoutePredicateHandlerMapping 源码分析

```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {
    private final RouteLocator routeLocator;

    @Override
    protected Mono<Object> getHandlerInternal(ServerWebExchange exchange) {
        // 设置当前请求的 GatewayContext
        exchange.getAttributes().put(ServerWebExchangeUtils.GATEWAY_HANDLER_MAPPER_ATTR, getClass().getSimpleName());
        return lookupRoute(exchange)
                .flatMap((Function<Route, Mono<?>>) r -> {
                    // 设置匹配到的路由到 exchange 属性
                    exchange.getAttributes().put(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR, r);
                    return Mono.just(r);
                })
                .switchIfEmpty(Mono.error(() -> new NotFoundException("No matching route")));
    }

    protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
        return this.routeLocator.getRoutes()
                .concatMap(route -> Mono.just(route)
                        .filterWhen(r -> r.getPredicate().apply(exchange))
                        .doOnError(e ->  ...)
                        .onErrorResume(e -> Mono.empty()))
                .next() // 只取第一个匹配的路由（按 order 排序后的第一个）
                .map(route -> {
                    // 记录路由匹配日志
                    if (logger.isDebugEnabled()) { ... }
                    return route;
                });
    }
}
```

**关键要点**：
1. `concatMap` 保证路由按定义顺序执行
2. `filterWhen` 执行异步断言匹配
3. `.next()` 取第一个匹配的路由后就终止
4. 路由的 `order` 值决定了匹配顺序，值越小越优先

## 4.3 FilteringWebHandler 源码分析

```java
public class FilteringWebHandler implements WebHandler {
    private final List<GlobalFilter> globalFilters;

    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        // 获取匹配到的路由
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);

        // 合并全局过滤器 + 路由级过滤器
        List<GatewayFilter> allFilters = combine(route);

        // 创建过滤器链并执行
        return new DefaultGatewayFilterChain(allFilters).filter(exchange);
    }

    private List<GatewayFilter> combine(Route route) {
        List<GatewayFilter> allFilters = new ArrayList<>();

        // 添加全局过滤器
        allFilters.addAll(this.globalFilters);

        // 添加路由级过滤器（用 OrderedGatewayFilter 包装以支持排序）
        allFilters.addAll(route.getFilters());

        // 按 order 升序排序
        allFilters.sort(AnnotationAwareOrderComparator.INSTANCE);

        return allFilters;
    }
}
```

**Order 排序规则**：
- 值越小越靠前执行（PRE 阶段先执行）
- POST 阶段执行顺序与 PRE 相反（值越大越先执行）
- 路由级过滤器默认 order = 0

# 5. Reactor Netty 线程模型

Spring Cloud Gateway 底层基于 Reactor Netty，采用事件驱动模型：

```yaml
# Netty 线程模型配置
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          type: fixed       # 连接池类型：fixed / elastic
          max-connections: 500
          acquire-timeout: 45000
          max-idle-time: 3000
```

## 5.1 Event Loop 模型

```
                    ┌─────────────────┐
                    │   Boss Event    │
                    │   Loop Group    │
                    │   (1-N threads) │
                    └────────┬────────┘
                             │ 连接接受
                             ▼
                    ┌─────────────────┐
                    │   Work Event    │
                    │   Loop Group    │
                    │   (N threads=   │
                    │    CPU cores*2) │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Channel  │  │ Channel  │  │ Channel  │
        │ Handler  │  │ Handler  │  │ Handler  │
        └──────────┘  └──────────┘  └──────────┘
```

**核心特性**：
- 少量线程处理大量连接（典型配置：CPU 核心数 × 2）
- 避免线程上下文切换开销
- 零拷贝（Direct Buffer）
- 背压支持（Reactive Streams）

## 5.2 阻塞操作注意事项

在响应式编程中，绝不能阻塞事件循环线程：

```java
// ❌ 错误：阻塞事件循环线程
public class BadGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Thread.sleep(1000); // 阻塞事件循环线程！
        return chain.filter(exchange);
    }
}

// ✅ 正确：使用 subscribeOn 切换到其他调度器
public class GoodGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return Mono.fromRunnable(() -> {
            // 阻塞操作
            Thread.sleep(1000);
        }).subscribeOn(Schedulers.boundedElastic())
          .then(chain.filter(exchange));
    }
}
```

# 6. 对比其他网关产品

| 特性 | Spring Cloud Gateway | Kong Kong（OpenResty） | Traefik | Nginx + Lua |
|------|---------------------|----------------------|---------|-------------|
| 语言生态 | Java（Spring） | Lua/C | Go | C/Lua |
| 架构模型 | 响应式 Netty | NIO + Lua VM | 响应式 | 事件驱动 |
| 配置方式 | YAML/Java DSL/API | REST API + DB | 标签/文件 | 文件配置 |
| 服务发现 | 原生集成 | 需插件 | 原生集成 | 需 Lua 实现 |
| 动态路由 | 支持（Actuator + 事件刷新） | 支持 | 支持 | 需 reload |
| 扩展性 | Java/Filters | Lua 插件 | Go 中间件 | Lua 脚本 |
| 学习成本 | 中等（需了解响应式编程） | 高（Lua + Nginx） | 低 | 高 |

# 7. 适用场景

- **Java 技术栈的微服务架构**：Spring Cloud 生态的最佳选择
- **需要快速集成服务发现**：与 Nacos/Eureka/Consul 原生集成
- **丰富的内置过滤器**：30+ 开箱即用的过滤器工厂
- **需要自定义扩展**：通过 Java API 灵活扩展 Predicate 和 Filter
- **高并发场景**：异步非阻塞架构，吞吐量优于传统 Servlet 网关
