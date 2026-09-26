
# 1. 概述

Spring Cloud Gateway 通过 `lb://` 协议前缀集成 Spring Cloud LoadBalancer，在路由转发时自动将服务名解析为实际实例地址。

# 2. 核心组件

## 2.1 ReactiveLoadBalancerClientFilter

```java
public class ReactiveLoadBalancerClientFilter
        implements GlobalFilter, Ordered {

    public static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;

    private final LoadBalancerClientFactory clientFactory;
    private final LoadBalancerProperties properties;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String schemePrefix = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR);

        // ① 非 lb 协议：跳过
        if (url == null || (!"lb".equals(url.getScheme())
                && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }

        // ② 提取 serviceId
        String serviceId = url.getHost();

        // ③ 收集 LoadBalancerLifecycle 回调
        Set<LoadBalancerLifecycle> lifecycles =
                LoadBalancerLifecycleValidator
                        .getSupportedLifecycleProcessors(
                                clientFactory.getInstances(serviceId,
                                        LoadBalancerLifecycle.class),
                                RequestDataContext.class,
                                Object.class,
                                ServiceInstance.class);

        // ④ 构建请求上下文
        RequestDataContext requestContext = buildRequestContext(exchange, url);

        // ⑤ 执行负载均衡选择
        return choose(serviceId, requestContext)
                .flatMap(response -> {
                    if (!response.hasServer()) {
                        return Mono.error(
                                new NotFoundException(
                                    "Unable to find instance for " + serviceId));
                    }

                    ServiceInstance instance = response.getServer();

                    // ⑥ 重建 URI
                    URI requestUrl = LoadBalancerUriTools.reconstructURI(
                            instance, url);
                    exchange.getAttributes().put(
                            ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR,
                            requestUrl);

                    // ⑦ 如果是 HTTPS 实例
                    if (instance.isSecure()) {
                        exchange.getAttributes().put(
                                ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR,
                                "https");
                    }

                    // ⑧ 触发 Lifecycle 完成回调
                    lifecycles.forEach(lifecycle -> lifecycle.onComplete(
                            new CompletionContext<>(...)));

                    return chain.filter(exchange);
                });
    }
}
```

# 3. 路由配置

## 3.1 手动路由 + LB

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service          # ← lb:// 协议前缀
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Source, Gateway
```

## 3.2 自动服务发现路由

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true                  # 自动为每个注册的服务创建路由
          lower-case-service-id: true    # 服务名转小写
          url-expression: "'lb://' + serviceId"
          filters:
            - StripPrefix=1
```

**自动路由效果**：Nacos 中注册了 `user-service`，Gateway 自动生成：

```
Path=/user-service/** → lb://user-service
```

# 4. 完整请求链路

```
客户端请求: GET http://gateway:8080/api/users/1

① DispatcherHandler 接收请求

② RoutePredicateHandlerMapping.lookupRoute()
   匹配路由: {id: user-service, uri: lb://user-service}

③ RouteToRequestUrlFilter
   设置 GATEWAY_REQUEST_URL_ATTR = lb://user-service/api/users/1

④ ReactiveLoadBalancerClientFilter (order=10100)
   │
   ├── 检查 scheme == "lb"
   ├── serviceId = "user-service"
   │
   ├── ReactorLoadBalancer.choose()
   │     ├── ServiceInstanceListSupplier.get()
   │     │     └── DiscoveryClient.getInstances("user-service")
   │     └── RoundRobinLoadBalancer.choose()
   │           └── → 192.168.1.11:8080
   │
   ├── 重建 URI: http://192.168.1.11:8080/api/users/1
   └── 更新 GATEWAY_REQUEST_URL_ATTR

⑤ NettyRoutingFilter (order=Integer.MAX_VALUE)
   发送实际 HTTP 请求到 http://192.168.1.11:8080/api/users/1

⑥ 返回响应给客户端
```

# 5. 过滤器顺序

Gateway 内置过滤器的执行顺序：

| Order | 过滤器 | 职责 |
|-------|--------|------|
| -1 | `AdaptCachedBodyGlobalFilter` | 缓存请求体 |
| 0 | `RemoveCachedBodyFilter` | 清理缓存 |
| 10000 | `ForwardRoutingFilter` | 转发请求 |
| **10100** | **ReactiveLoadBalancerClientFilter** | **负载均衡** |
| 10150 | `WebsocketRoutingFilter` | WebSocket 路由 |
| MAX | `NettyRoutingFilter` | Netty 发出请求 |

# 6. 自定义 Gateway LB 过滤器

```java
@Component
public class CustomLoadBalancerFilter
        implements GlobalFilter, Ordered {

    private final LoadBalancerClientFactory clientFactory;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String schemePrefix = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR);

        if (url == null || (!"lb".equals(url.getScheme())
                && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }

        String serviceId = url.getHost();

        // 获取自定义 LoadBalancer
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                clientFactory.getInstance(serviceId,
                        ReactorServiceInstanceLoadBalancer.class);

        // 构建带 Hint 的请求
        DefaultRequest<RequestDataContext> lbRequest =
                new DefaultRequest<>(new RequestDataContext(
                        new RequestData(
                                exchange.getRequest().getMethodValue(),
                                url,
                                exchange.getRequest().getHeaders(),
                                null),
                        Collections.emptyMap()));

        return loadBalancer.choose(lbRequest)
                .flatMap(response -> {
                    ServiceInstance instance = response.getServer();
                    URI newUrl = LoadBalancerUriTools.reconstructURI(
                            instance, url);
                    exchange.getAttributes().put(
                            ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR,
                            newUrl);
                    return chain.filter(exchange);
                });
    }

    @Override
    public int getOrder() {
        return ReactiveLoadBalancerClientFilter
                .LOAD_BALANCER_CLIENT_FILTER_ORDER - 1;  // 在原过滤器之前
    }
}
```

# 7. WebSocket 负载均衡

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: websocket-route
          uri: lb:ws://chat-service       # lb:ws:// 协议
          predicates:
            - Path=/ws/chat/**
```

# 8. 区域感知路由

```yaml
spring:
  cloud:
    loadbalancer:
      zone: SH          # 当前 Gateway 所在 Zone

    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
```

结合 `ZonePreferenceServiceInstanceListSupplier`，Gateway 会优先将请求路由到同 Zone 的后端实例。

# 9. 动态路由刷新

```java
@RestController
@RequestMapping("/admin/routes")
public class RouteRefreshController {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    @Autowired
    private ApplicationEventPublisher publisher;

    @PostMapping("/add")
    public Mono<String> addRoute(@RequestBody RouteDefinition definition) {
        return routeDefinitionWriter.save(Mono.just(definition))
                .then(Mono.fromRunnable(() ->
                        publisher.publishEvent(
                                new RefreshRoutesEvent(this))))
                .thenReturn("Route added");
    }

    @DeleteMapping("/delete/{routeId}")
    public Mono<String> deleteRoute(@PathVariable String routeId) {
        return routeDefinitionWriter.delete(Mono.just(routeId))
                .then(Mono.fromRunnable(() ->
                        publisher.publishEvent(
                                new RefreshRoutesEvent(this))))
                .thenReturn("Route deleted");
    }
}
```

# 10. 最佳实践

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
          metadata:
            response-timeout: 5000
            connect-timeout: 3000

    loadbalancer:
      cache:
        ttl: 10s          # Gateway 层缓存稍短，更快感知实例变化
      retry:
        enabled: true
        max-retries-on-next-service-instance: 2
      zone: ${ZONE:default}

  cloud:
    gateway:
      httpclient:
        pool:
          type: fixed
          max-connections: 500
          max-idle-time: 30000
          acquire-timeout: 45000
```
