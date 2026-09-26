# 1. 知识点
- **网关核心功能**：路由转发、统一鉴权认证、限流熔断、日志采集、灰度发布、跨域处理、负载均衡、协议转换、请求聚合。
- **Route路由**：网关的基本构建块，由ID、目标URI、Predicate集合、Filter集合组成，定义了请求匹配和转发规则。
- **Predicate断言**：Java 8函数式接口，用于匹配HTTP请求参数（如路径、Header、参数、时间等），决定请求是否匹配该路由。
- **Filter过滤器**：Spring WebFlux的WebFilter实例，对请求和响应做修改（添加Header、重写路径、限流等），分为GatewayFilter和GlobalFilter。
- **基于Netty运行**：Gateway构建在Spring WebFlux之上，使用Netty作为底层网络引擎，支持异步非阻塞I/O，高并发性能优秀。
- **工作流程**：GatewayHandlerMapping匹配路由 → GatewayWebHandler执行前置过滤器链 → 请求转发到后端微服务 → 后端返回响应 → 执行后置过滤器链 → 返回客户端。
- **GatewayFilter**：局部过滤器，只对特定路由生效，通过路由配置中的filters指定，Spring提供了丰富的内置过滤器工厂。
- **GlobalFilter**：全局过滤器，对所有路由生效，实现了GlobalFilter接口，如鉴权、日志、限流等全局功能通常用GlobalFilter实现。
- **过滤器链**：FilterChain将所有GatewayFilter串联执行，每个Filter可拦截请求进入和响应返回两个阶段。
- **Path路由断言**：根据请求路径匹配路由，如`Path=/api/user/**`匹配所有`/api/user/`开头的请求。
- **After/Before/Between路由断言**：根据请求时间匹配路由，可设置路由的生效时间段，用于限时活动等场景。
- **Cookie路由断言**：根据请求中的Cookie键值对匹配路由，适用于基于Cookie的灰度路由。
- **Header路由断言**：根据请求Header中的属性值匹配路由，常用于基于版本号、灰度标签的流量路由。
- **Host路由断言**：根据请求的Host头匹配路由，支持多域名路由到不同的后端服务。
- **Method路由断言**：根据HTTP请求方法（GET/POST/PUT/DELETE）匹配路由。
- **Query路由断言**：根据URL查询参数匹配路由，支持变量名和可选的正则表达式值。
- **RemoteAddr路由断言**：根据请求来源IP地址（CIDR表示法）匹配路由，如内网IP访问特定路由。
- **Weight权重路由断言**：将流量按权重比例分配到不同服务，用于灰度发布，需要配合group分组。
- **AddRequestHeader过滤器**：在请求转发前向Header中添加指定键值对，常用于向后端服务传递网关标识。
- **AddResponseHeader过滤器**：在响应返回前向Header中添加指定键值对。
- **AddRequestParameter过滤器**：在请求转发前向URL添加查询参数。
- **PrefixPath过滤器**：在请求路径前添加前缀，如`/api/user`变为`/gateway/api/user`。
- **StripPrefix过滤器**：移除请求路径的前缀部分，如`/api/user/list`移除`/api`后变为`/user/list`。
- **RedirectTo过滤器**：将请求重定向到指定URL，支持HTTP状态码指定（301/302）。
- **RewritePath过滤器**：使用正则表达式重写请求路径，支持捕获组替换。
- **Retry过滤器**：配置请求失败后的重试策略（重试次数、重试状态码、重试方法等），提高可用性。
- **RequestRateLimiter过滤器**：基于Redis实现令牌桶算法的请求限流，需实现KeyResolver定义限流维度。
- **自定义全局过滤器**：实现`GlobalFilter`和`Ordered`接口，通过`ServerWebExchange`获取和修改请求响应，控制过滤器的执行顺序。
- **Sentinel集成**：使用`sentinel-spring-cloud-gateway-adapter`将Gateway路由接入Sentinel管控，支持API分组、流控、降级。
- **跨域配置**：通过`spring.cloud.gateway.globalcors`配置全局CORS策略，支持配置允许的源、方法、头等。
- **负载均衡集成**：默认集成Spring Cloud LoadBalancer，URI中使用`lb://服务名`即可实现负载均衡转发。
- **灰度发布实现**：通过Weight权重路由（按比例分流）、Header匹配（按请求特征分流）、参数匹配（按客户端版本分流）实现金丝雀发布。

# 2. 面试题

## 2.1 Spring Cloud Gateway 和 Zuul 的区别是什么？
| 维度 | Spring Cloud Gateway | Zuul 1.x |
|------|---------------------|----------|
| 架构 | 异步非阻塞（Reactor） | 同步阻塞（Servlet） |
| 底层 | Netty | Tomcat/Servlet容器 |
| 线程模型 | 少量线程处理大量请求 | 一个请求一个线程 |
| 性能 | 高（适合高并发） | 较低（并发瓶颈明显） |
| API支持 | WebFlux + WebSocket | Servlet + ZuulFilter |
| 功能丰富度 | Router/Filter/Predicate | ZuulFilter |
| 长连接支持 | WebSocket天然支持 | 需要额外配置 |
| 维护状态 | 活跃维护 | 已停更 |
| Spring Boot | 2.x及以上 | 1.x/2.x |

Zuul 2.x研发停滞未发布正式版，新项目应选择Spring Cloud Gateway。

## 2.2 Spring Cloud Gateway 的核心组件是什么？
**三大核心组件**：

1. **Route（路由）**：
   - 由唯一ID标识
   - 包含目标URI（转发地址）
   - 包含一组Predicate（断言）和一组Filter（过滤器）
   - 如果所有Predicate匹配成功，请求通过该路由转发
   - 示例：`id: order-route, uri: lb://order-service, predicates: [Path=/order/**]`

2. **Predicate（断言）**：
   - 基于Java 8 Predicate函数式接口
   - 匹配HTTP请求的各种条件（路径、Header、参数、时间等）
   - 通过断言工厂（如PathRoutePredicateFactory）创建
   - 可通过`-`前缀配置多个断言，"与"逻辑

3. **Filter（过滤器）**：
   - GatewayFilter：针对特定路由的过滤器
   - GlobalFilter：全局过滤器，所有路由生效
   - 可修改请求（前置过滤）和响应（后置过滤）
   - 通过过滤器工厂创建或直接实现接口

## 2.3 Spring Cloud Gateway 的工作流程详解
1. 客户端请求到达Gateway的Netty服务器
2. `RoutePredicateHandlerMapping`收到请求：
   - 遍历所有Route定义
   - 对每个Route执行所有Predicate断言
   - 找到第一个所有断言都匹配的Route
   - 如果未匹配任何Route，返回404
3. `FilteringWebHandler`获取匹配Route的所有GatewayFilter和全局GlobalFilter
4. 构建过滤器链（`DefaultGatewayFilterChain`）：
   - 按`Ordered`接口的order值排序
   - GlobalFilter和GatewayFilter混合排序
5. 前置过滤器依次执行（修改请求Header、路径重写、鉴权、限流等）
6. 过滤器链执行到`NettyRoutingFilter`：
   - 通过LoadBalancer选择目标服务实例
   - 构建转发请求，通过Netty客户端发送
7. 后端微服务处理请求并返回响应
8. 后置过滤器依次执行（修改响应Header、日志记录等）
9. 最终响应返回给客户端

## 2.4 Spring Cloud Gateway 如何实现动态路由？
**方式一：基于Nacos配置中心动态路由**
```yaml
# Nacos配置
spring:
  cloud:
    gateway:
      routes:
        - id: dynamic-route
          uri: lb://new-service
          predicates:
            - Path=/new-service/**
```
结合`@RefreshScope`或`RefreshRoutesEvent`实现热更新。

**方式二：编程式动态路由**
```java
@Component
public class DynamicRouteService {
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    @Autowired
    private ApplicationEventPublisher publisher;

    public void addRoute(RouteDefinition definition) {
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }

    public void deleteRoute(String routeId) {
        routeDefinitionWriter.delete(Mono.just(routeId)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
}
```

**方式三：数据库+监听变更**
1. 将路由配置存储在数据库（MySQL/Redis）
2. 通过定时任务或监听机制检测路由变更
3. 变更后调用`RouteDefinitionWriter`更新路由并发布`RefreshRoutesEvent`

## 2.5 Spring Cloud Gateway 的过滤器体系是怎样的？
**过滤器类型**：

1. **GatewayFilter（局部过滤器）**：
   - 只作用于配置了该过滤器的特定路由
   - 通过路由配置文件的`filters`属性指定
   - 示例：`StripPrefix=1`、`AddRequestHeader=X-From, gateway`

2. **GlobalFilter（全局过滤器）**：
   - 作用于所有路由
   - 实现`GlobalFilter`和`Ordered`接口
   - 注入Spring容器即可自动生效
   - 示例：`LoadBalancerClientFilter`、`NettyRoutingFilter`

3. **内置全局过滤器**（按执行顺序）：
   - `RemoveCachedBodyFilter`：移除缓存Body
   - `AdaptCachedBodyGlobalFilter`：适配缓存Body
   - `NettyWriteResponseFilter`：Netty响应写入
   - `ForwardPathFilter`：路径转发
   - `RouteToRequestUrlFilter`：路由转URL
   - `ReactiveLoadBalancerClientFilter`：负载均衡
   - `LoadBalancerServiceInstanceCookieFilter`：负载均衡Cookie（WebSocket）
   - `NettyRoutingFilter`：HTTP请求转发
   - `ForwardRoutingFilter`：Forward转发

## 2.6 如何自定义全局过滤器？请写一个鉴权过滤器示例
```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    // 白名单路径（不需要鉴权）
    private static final List<String> WHITE_LIST = Arrays.asList(
        "/auth/login", "/auth/register", "/public/**"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();

        // 白名单路径直接放行
        if (isWhitePath(path)) {
            return chain.filter(exchange);
        }

        // 检查Token
        String token = request.getHeaders().getFirst("Authorization");
        if (StringUtils.isEmpty(token)) {
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            return response.setComplete(); // 401未授权
        }

        try {
            // 验证Token有效性，解析用户信息
            String userId = JwtUtil.parseToken(token);
            // 将用户信息传递给下游服务（通过Header）
            ServerHttpRequest newRequest = request.mutate()
                .header("X-User-Id", userId)
                .build();
            return chain.filter(exchange.mutate().request(newRequest).build());
        } catch (Exception e) {
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.FORBIDDEN);
            return response.setComplete(); // 403 Token无效
        }
    }

    @Override
    public int getOrder() {
        return -100; // 数值越小优先级越高
    }
}
```

## 2.7 如何使用 Spring Cloud Gateway 实现限流？
**基于RequestRateLimiter + Redis的令牌桶限流**：

```java
@Configuration
public class RateLimiterConfig {
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            Objects.requireNonNull(
                exchange.getRequest().getRemoteAddress()
            ).getAddress().getHostAddress()
        );
    }
    // 也可按用户限流、按接口限流等
}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: rate-limit-route
          uri: lb://order-service
          predicates:
            - Path=/order/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10   # 每秒令牌填充速率
                redis-rate-limiter.burstCapacity: 20   # 令牌桶容量
                redis-rate-limiter.requestedTokens: 1  # 每次请求消耗令牌数
                key-resolver: "#{@ipKeyResolver}"      # 限流维度Key
  redis:
    host: localhost
    port: 6379
```

**限流维度**（KeyResolver）：
- IP限流：按请求来源IP
- 用户限流：按URL参数中的userId或Header中的Token
- 接口限流：按请求路径
- 组合限流：IP+接口、用户+接口

## 2.8 Spring Cloud Gateway 如何集成 Sentinel 实现网关限流？
**步骤一**：添加依赖
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-cloud-gateway-adapter</artifactId>
</dependency>
```

**步骤二**：配置
```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      filter:
        enabled: true  # 开启Gateway Sentinel Filter
      scg:
        fallback:
          mode: response  # 限流降级时返回自定义响应
          response-body: '{"code":429,"message":"请求过多，请稍后再试"}'
          response-status: 429
          content-type: application/json
```

**步骤三**：可在Sentinel控制台配置：
- **API分组**：将多个路由路径组合为一个API组，统一配置流控规则
- **网关流控规则**：QPS阈值、针对特定路由/API组
- **降级规则**：慢调用比例、异常比例、异常数

**与普通Sentinel的区别**：
- 网关资源以Route ID或API Group为粒度
- 流控规则在网关层面统一管理，避免在每个微服务配置
- 拦截发生在网关层，降低后端服务的无效请求压力

## 2.9 如何使用 Spring Cloud Gateway 实现灰度发布？
**方式一：Weight权重路由**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-v1
          uri: lb://order-service
          predicates:
            - Path=/order/**
            - Weight=group1, 90        # 90%流量走v1
        - id: order-service-v2
          uri: lb://order-service-canary
          predicates:
            - Path=/order/**
            - Weight=group1, 10        # 10%流量走v2（灰度版本）
```

**方式二：Header匹配灰度**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-canary
          uri: lb://order-service-v2
          predicates:
            - Path=/order/**
            - Header=version, gray     # Header中version=gray的请求走灰度
        - id: order-normal
          uri: lb://order-service
          predicates:
            - Path=/order/**
```

**方式三：参数匹配灰度**
```yaml
predicates:
  - Path=/order/**
  - Query=version, gray   # URL参数version=gray的请求走灰度
```

**全链路灰度的关键**：网关通过Header/参数识别灰度流量后，需要将灰度标签透传到所有下游服务（通过RequestInterceptor或自定义GlobalFilter设置Header），各服务根据灰度标签路由到对应实例。
