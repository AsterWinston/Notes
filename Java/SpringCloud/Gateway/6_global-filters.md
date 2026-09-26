
# 1. 全局过滤器体系

## 1.1 接口定义

```java
@FunctionalInterface
public interface GlobalFilter {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

`GlobalFilter` 与 `GatewayFilter` 接口签名完全相同，区别在于：
- `GlobalFilter`：全局生效，自动作用于所有路由
- `GatewayFilter`：路由级生效，需在路由配置中显式引用

## 1.2 合并机制 — FilteringWebHandler 源码

```java
public class FilteringWebHandler implements WebHandler {
    private final List<GlobalFilter> globalFilters;

    public FilteringWebHandler(List<GlobalFilter> globalFilters) {
        this.globalFilters = globalFilters;
    }

    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        // 从 exchange 中获取匹配到的路由
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);

        // 合并全局过滤器 + 路由级过滤器
        List<GatewayFilter> allFilters = new ArrayList<>(this.globalFilters);
        allFilters.addAll(route.getGatewayFilters());

        // 统一排序
        AnnotationAwareOrderComparator.sort(allFilters);

        // 创建并执行过滤器链
        return new DefaultGatewayFilterChain(allFilters).filter(exchange);
    }
}
```

## 1.3 内置全局过滤器总览

| 类名 | Order | 职责 |
|------|-------|------|
| `AdaptCachedBodyGlobalFilter` | `MAX_VALUE - 3` | 缓存请求体供后续过滤器使用 |
| `RouteToRequestUrlFilter` | 10000 | 构建下游请求 URL |
| `ReactiveLoadBalancerClientFilter` | 10100 | 解析 `lb://` 协议，负载均衡 |
| `ForwardRoutingFilter` | `MAX_VALUE` | 处理 `forward://` 本地转发 |
| `ForwardPathFilter` | 0 | 为 `forward://` 路径添加前缀 |
| `NettyRoutingFilter` | `MAX_VALUE - 1` | 通过 Netty HttpClient 发送 HTTP 请求 |
| `NettyWriteResponseFilter` | -1 | 将后端响应写回客户端 |
| `WebsocketRoutingFilter` | `MAX_VALUE - 2` | WebSocket 协议转发 |
| `GatewayMetricsFilter` | 0 | 记录请求指标 |
| `RemoveCachedBodyFilter` | `MAX_VALUE` | 清理缓存的请求体 |
| `WeightCalculatorWebFilter` | - | 权重路由分组选择 |

# 2. 核心全局过滤器源码解析

## 2.1 RouteToRequestUrlFilter

**Order**: 10000

**职责**：将路由定义的 URI 与当前请求的路径结合，构造最终的下游请求 URL。

```java
public class RouteToRequestUrlFilter implements GlobalFilter, Ordered {
    public static final String ROUTE_TO_URL_FILTER_ORDER = 10000;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route == null) return chain.filter(exchange);

        URI routeUri = route.getUri();
        String scheme = routeUri.getScheme();

        if (ServerWebExchangeUtils.isAlreadyRouted(exchange)) return chain.filter(exchange);

        // 处理带有 lb 协议前缀的 URI
        if (routeUri.getHost() == null && routeUri.getRawPath() == null) {
            throw new IllegalArgumentException("Invalid route URI: " + routeUri);
        }

        // 构建请求 URL：routeUri + request path
        URI requestUrl = UriComponentsBuilder
            .fromUri(routeUri)
            .path(exchange.getRequest().getPath().pathWithinApplication().value())
            .queryParams(exchange.getRequest().getQueryParams())
            .build()
            .toUri();

        // 保存到 exchange 属性
        exchange.getAttributes().put(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR, requestUrl);

        // 如果 scheme 前缀有 lb，保存原始 scheme 信息
        if (routeUri.getScheme() != null && routeUri.getScheme().contains("lb")) {
            exchange.getAttributes().put(
                ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR,
                routeUri.getScheme().substring(0, routeUri.getScheme().indexOf("lb")));
        }

        return chain.filter(exchange);
    }
}
```

**处理流程**：

```
路由 URI: lb://user-service
请求路径: /api/user/profile?id=1

↓ RouteToRequestUrlFilter

GATEWAY_REQUEST_URL_ATTR = lb://user-service/api/user/profile?id=1
```

## 2.2 ReactiveLoadBalancerClientFilter

**Order**: 10100

**职责**：解析 `lb://` 协议的 URI，通过 Spring Cloud LoadBalancer 选择服务实例。

```java
public class ReactiveLoadBalancerClientFilter implements GlobalFilter, Ordered {
    private static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;
    private final ReactorLoadBalancer<ServiceInstance> loadBalancer;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String schemePrefix = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR);

        // 检查是否使用 lb 协议
        if (url == null || (!"lb".equals(url.getScheme())
                && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }

        // 从 lb:// 协议中提取 serviceId
        URI lbUri = url;
        if ("lb".equals(schemePrefix)) {
            // 处理 lb:http:// 格式
            lbUri = URI.create(url.getSchemeSpecificPart());
        }

        String serviceId = lbUri.getHost();
        Set<LoadBalancerLifecycle> lifecycle = getLoadBalancerLifecycle();
        return choose(serviceId, exchange)  // 选择服务实例
            .flatMap(response -> {
                ServiceInstance instance = response.getServer();
                if (instance == null) {
                    throw new NotFoundException("Unable to find instance for " + serviceId);
                }

                // 替换 URI 中的 host:port 为实际实例地址
                URI requestUrl = reconstructURI(instance, lbUri);
                exchange.getAttributes().put(
                    ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR, requestUrl);

                // 设置是否 HTTPS
                if (instance.isSecure()) {
                    exchange.getAttributes().put(
                        ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR, "https");
                }

                return chain.filter(exchange);
            });
    }

    private Mono<Response<ServiceInstance>> choose(String serviceId,
            ServerWebExchange exchange) {
        // 委托给 LoadBalancer 选择实例
        return this.loadBalancer.choose(request);
    }

    private URI reconstructURI(ServiceInstance instance, URI original) {
        // 用实际实例地址替换 lb://serviceId
        return LoadBalancerUriTools.reconstructURI(instance, original);
    }
}
```

## 2.3 NettyRoutingFilter

**Order**: `MAX_VALUE - 1`（Integer.MAX_VALUE - 1）

**职责**：通过 Netty HttpClient 发送 HTTP/HTTPS 请求到后端服务。

```java
public class NettyRoutingFilter implements GlobalFilter, Ordered {
    private final HttpClient httpClient;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI requestUrl = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);

        // 只处理 http/https 协议
        String scheme = requestUrl.getScheme();
        if (!"http".equals(scheme) && !"https".equals(scheme)) {
            return chain.filter(exchange);
        }

        // 标记已经路由，防止重复路由
        ServerWebExchangeUtils.setAlreadyRouted(exchange);

        ServerHttpRequest request = exchange.getRequest();
        HttpMethod method = request.getMethod();

        // 构建 Netty 请求
        HttpHeaders headers = request.getHeaders();
        Flux<HttpData> body = request.getBody()
            .map(dataBuffer -> ByteBufHttpData
                .newInstance(dataBuffer.asByteBuffer()));

        // 发送请求
        Flux<HttpClientResponse> responseFlux = this.httpClient
            .request(method)
            .uri(requestUrl.toString())
            .send((req, nettyOutbound) -> {
                // 写入请求头和请求体
                headers.forEach((key, values) ->
                    values.forEach(value -> req.header(key, value)));
                return nettyOutbound.send(body.map(data -> data));
            })
            .responseConnection((res, connection) -> {
                // 处理响应
                ServerHttpResponse delegate = exchange.getResponse();
                // 构建 Flux<DataBuffer> 响应
                Flux<DataBuffer> responseBody = connection.inbound()
                    .receive()
                    .map(byteBuf -> {
                        DataBuffer buffer = delegate.bufferFactory()
                            .wrap(byteBuf.retain());
                        return buffer;
                    });

                // 设置响应头
                res.responseHeaders().forEach((name, values) ->
                    delegate.getHeaders().addAll(name, values));

                return Mono.just(responseBody);
            });

        // 将响应写回 exchange
        return responseFlux.flatMap(responseBody ->
            exchange.getResponse().writeWith(responseBody));
    }
}
```

## 2.4 NettyWriteResponseFilter

**Order**: -1

**职责**：将后端服务的响应写回到客户端。Order 值很小以确保在 POST 阶段最后执行。

```java
public class NettyWriteResponseFilter implements GlobalFilter, Ordered {
    public static final int WRITE_RESPONSE_FILTER_ORDER = -1;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 先执行过滤器链（包括路由请求到后端）
        return chain.filter(exchange).then(Mono.defer(() -> {
            // 获取后端响应数据
            ServerHttpResponse response = exchange.getResponse();

            // 如果已有数据，无需额外处理
            if (response.isCommitted()) {
                return Mono.empty();
            }

            // 某些情况下需要获取缓存的响应体
            DataBuffer body = exchange.getAttribute(
                ServerWebExchangeUtils.CACHED_RESPONSE_BODY_ATTR);
            if (body != null) {
                return response.writeWith(Mono.just(body));
            }

            return Mono.empty();
        }));
    }
}
```

## 2.5 ForwardRoutingFilter

**Order**: `MAX_VALUE`（最低优先级）

**职责**：处理 `forward://` 协议的 URI，在网关内部转发请求。

```java
public class ForwardRoutingFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI requestUrl = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        if (requestUrl == null || !"forward".equals(requestUrl.getScheme())) {
            return chain.filter(exchange);
        }

        // 标记已路由
        ServerWebExchangeUtils.setAlreadyRouted(exchange);

        // 构建转发路径
        String path = requestUrl.getPath();
        // 通过 DispatcherHandler 在网关内部转发
        return this.dispatcherHandler.handle(
            exchange.mutate()
                .request(exchange.getRequest().mutate()
                    .path(path)
                    .build())
                .build())
            .then(Mono.empty());
    }
}
```

**典型应用**：熔断降级的 fallback 处理。

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: myCB
      fallbackUri: forward:/fallback

routes:
  - id: fallback_route
    uri: forward:/fallback
    predicates:
      - Path=/fallback
```

## 2.6 WebsocketRoutingFilter

**Order**: `MAX_VALUE - 2`

**职责**：处理 `ws://` 和 `wss://` 协议的 WebSocket 请求。

详见 [14_websocket.md] 的详细说明。

## 2.7 AdaptCachedBodyGlobalFilter

**Order**: `MAX_VALUE - 3`

**职责**：缓存请求体，供需要读取 Body 的断言或过滤器使用（如 `ReadBodyRoutePredicateFactory`、`ModifyRequestBodyGatewayFilterFactory`）。

```java
public class AdaptCachedBodyGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 检查是否已经有缓存的 body
        DataBuffer cachedBody = exchange.getAttribute(
            ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
        if (cachedBody != null) {
            return chain.filter(exchange);
        }

        // 读取并缓存请求体
        return ServerWebExchangeUtils.cacheRequestBody(exchange, (serverHttpRequest) -> {
            return chain.filter(exchange.mutate()
                .request(serverHttpRequest)
                .build());
        });
    }
}
```

## 2.8 RemoveCachedBodyGlobalFilter

**Order**: `MAX_VALUE`（最后执行）

**职责**：清理缓存的请求体数据，防止内存泄漏。

```java
public class RemoveCachedBodyGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // POST 阶段移除缓存的请求体
            DataBuffer oldBody = exchange.getAttribute(
                ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
            if (oldBody != null) {
                DataBufferUtils.release(oldBody);
                exchange.getAttributes()
                    .remove(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
            }
        }));
    }
}
```

## 2.9 GatewayMetricsFilter

**Order**: 0

**职责**：记录请求指标（路由 ID、URI、状态码、耗时等）。

详见 [17_monitoring.md] 的详细说明。

# 3. 全局过滤器执行顺序详解

## 3.1 排序规则

```
Order 值    →   PRE 阶段执行顺序    →   POST 阶段执行顺序
负值         先执行                   后执行
0           中间                     中间
正值         后执行                   先执行
```

## 3.2 完整的执行序列

```
NettyWriteResponseFilter (-1)       ← POST: 写响应
     │
GatewayMetricsFilter (0)            ← PRE: 开始计时
ForwardPathFilter (0)               ← PRE: forward 路径处理
     │
RouteToRequestUrlFilter (10000)     ← PRE: 构建目标 URL
     │
ReactiveLoadBalancerClientFilter (10100) ← PRE: 负载均衡
     │
     ▼
NettyRoutingFilter (MAX_VALUE - 1)  ← PRE: 发送 HTTP 请求到后端
WebsocketRoutingFilter (MAX_VALUE - 2) ← PRE: WebSocket 转发
AdaptCachedBodyGlobalFilter (MAX_VALUE - 3) ← PRE: 缓存请求体
ForwardRoutingFilter (MAX_VALUE)    ← PRE: forward 转发
RemoveCachedBodyGlobalFilter (MAX_VALUE) ← POST: 清理缓存
     │
     ▼
后端服务
     │
     ▼（响应返回）
     │
RemoveCachedBodyGlobalFilter (MAX_VALUE) ← POST: 清理缓存
ForwardRoutingFilter (MAX_VALUE)    ← POST: forward 完成
AdaptCachedBodyGlobalFilter (MAX_VALUE - 3) ← POST: 完成
WebsocketRoutingFilter (MAX_VALUE - 2) ← POST: 完成
NettyRoutingFilter (MAX_VALUE - 1)  ← POST: HTTP 请求完成
     │
ReactiveLoadBalancerClientFilter (10100) ← POST: 负载均衡完成
RouteToRequestUrlFilter (10000)     ← POST: URL 构建完成
     │
ForwardPathFilter (0)               ← POST: 路径处理完成
GatewayMetricsFilter (0)            ← POST: 记录指标
     │
NettyWriteResponseFilter (-1)       ← POST: 写响应（最先执行 POST）
```

# 4. 自定义全局过滤器

## 4.1 最简单的全局过滤器

```java
@Component
public class LoggingGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // PRE 阶段
        String path = exchange.getRequest().getPath().value();
        String method = exchange.getRequest().getMethod().name();
        System.out.println("→ " + method + " " + path);

        // 记录开始时间
        long startTime = System.currentTimeMillis();
        exchange.getAttributes().put("startTime", startTime);

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // POST 阶段
            long elapsed = System.currentTimeMillis() - startTime;
            HttpStatus status = exchange.getResponse().getStatusCode();
            System.out.println("← " + status + " (" + elapsed + "ms)");
        }));
    }

    @Override
    public int getOrder() {
        return -100; // 高优先级
    }
}
```

## 4.2 认证全局过滤器

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    private static final Set<String> PUBLIC_PATHS = Set.of(
        "/login", "/register", "/actuator/health");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 白名单路径跳过认证
        if (PUBLIC_PATHS.contains(path)) {
            return chain.filter(exchange);
        }

        // 检查 Token
        String token = exchange.getRequest().getHeaders()
            .getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            // 返回 401
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // 验证 Token（异步调用认证服务）
        return validateToken(token.substring(7))
            .flatMap(valid -> {
                if (!valid) {
                    exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                    return exchange.getResponse().setComplete();
                }
                return chain.filter(exchange);
            });
    }

    private Mono<Boolean> validateToken(String jwt) {
        // 调用认证服务验证 JWT
        return Mono.just(true); // 简化示例
    }

    @Override
    public int getOrder() {
        return -200; // 最高优先级，最先执行
    }
}
```

## 4.3 Bean 方式注册全局过滤器

```java
@Configuration
public class GlobalFilterConfiguration {

    @Bean
    public GlobalFilter timingFilter() {
        return new OrderedGatewayFilter((exchange, chain) -> {
            long start = System.nanoTime();
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                long elapsed = System.nanoTime() - start;
                System.out.println("Time: " + elapsed / 1_000_000 + "ms");
            }));
        }, -50);
    }
}
```

# 5. 全局过滤器的最佳实践

## 5.1 Order 规划

建议为自定义全局过滤器预留 Order 区间：

| Order 区间 | 用途 | 示例 |
|-----------|------|------|
| `-∞ ~ -1000` | 安全认证、IP 黑白名单 | AuthGlobalFilter |
| `-999 ~ -100` | 请求预处理、日志 | LoggingFilter |
| `-99 ~ 10000` | 业务相关 | RateLimitFilter |
| `10001 ~ MAX` | 路由转发、后处理 | 默认系统过滤器 |

## 5.2 性能注意事项

1. **避免阻塞**：不要在过滤器链中使用阻塞操作
2. **尽早返回**：无法处理的请求尽早返回（如鉴权失败）
3. **注意内存**：`AdaptCachedBodyGlobalFilter` 会缓存 Body，大请求体时注意内存
4. **复用连接**：配置合理的连接池参数

## 5.3 注意事项

- 全局过滤器不能使用 `@ConditionalOnMissingBean` 替换内置实现
- 自定义全局过滤器需实现 `Ordered` 接口或使用 `@Order` 注解
- 全局过滤器中抛出异常会被 `ExceptionHandlingWebHandler` 捕获
- 标记 `setAlreadyRouted()` 可以防止请求被多次路由
