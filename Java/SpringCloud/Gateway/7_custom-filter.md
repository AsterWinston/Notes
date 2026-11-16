
# 1. 自定义过滤器的方式

Spring Cloud Gateway 提供了三种自定义过滤器的方式：

| 方式 | 作用域 | 配置方式 | 适用场景 |
|------|--------|----------|----------|
| 自定义 `GatewayFilterFactory` | 路由级 | 通过 YAML/DSL 按路由配置 | 需要参数化配置的过滤器 |
| 自定义 `GlobalFilter` | 全局 | Bean 自动注入 | 全局通用逻辑（认证、日志等） |
| 自定义 `GatewayFilter` | 路由级 | 通过 Java DSL 的 `.filter()` 方法 | 简单的一次性过滤器 |

# 2. 自定义 GatewayFilterFactory

## 2.1 命名规范

工厂类名必须以 `GatewayFilterFactory` 结尾，配置时使用前缀部分：

```java
public class AuthGatewayFilterFactory extends AbstractGatewayFilterFactory<...> { }
// 配置: filters: - Auth=...
```

## 2.2 抽象基类

```java
public abstract class AbstractGatewayFilterFactory<C>
        implements GatewayFilterFactory<C>, ApplicationEventPublisherAware {

    private final Class<C> configClass;

    public AbstractGatewayFilterFactory(Class<C> configClass) {
        this.configClass = configClass;
    }

    @Override
    public C newConfig() {
        return BeanUtils.instantiateClass(configClass);
    }
}
```

## 2.3 带配置参数的过滤器工厂

```java
@Component
public class RateLimitByIpGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RateLimitByIpGatewayFilterFactory.Config> {

    private final RateLimiter rateLimiter;

    public RateLimitByIpGatewayFilterFactory(RateLimiter rateLimiter) {
        super(Config.class);
        this.rateLimiter = rateLimiter;
    }

    @Override
    public List<String> shortcutFieldOrder() {
        // 快捷配置参数顺序: "capacity,refillRate"
        return Arrays.asList("capacity", "refillRate");
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String ip = exchange.getRequest().getRemoteAddress()
                .getAddress().getHostAddress();

            return rateLimiter.isAllowed(ip, config.getCapacity(), config.getRefillRate())
                .flatMap(allowed -> {
                    if (!allowed) {
                        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                        return exchange.getResponse().setComplete();
                    }
                    return chain.filter(exchange);
                });
        };
    }

    public static class Config {
        private int capacity = 100;    // 桶容量
        private int refillRate = 10;   // 每秒补充速率

        public int getCapacity() { return capacity; }
        public void setCapacity(int capacity) { this.capacity = capacity; }
        public int getRefillRate() { return refillRate; }
        public void setRefillRate(int refillRate) { this.refillRate = refillRate; }
    }
}
```

**YAML 配置**：

```yaml
filters:
  # 快捷方式
  - RateLimitByIp=200, 20
  # 完整方式
  - name: RateLimitByIp
    args:
      capacity: 200
      refillRate: 20
```

## 2.4 使用 ShortcutType

```java
@Component
public class HeaderInjectionGatewayFilterFactory
        extends AbstractGatewayFilterFactory<HeaderInjectionGatewayFilterFactory.Config> {

    public HeaderInjectionGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public ShortcutType shortcutType() {
        // GATHER_LIST: 将剩余参数作为列表
        return ShortcutType.GATHER_LIST;
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList("headers");
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest().mutate()
                .headers(headers -> config.getHeaders().forEach(h -> {
                    String[] parts = h.split(":", 2);
                    if (parts.length == 2) {
                        headers.add(parts[0].trim(), parts[1].trim());
                    }
                }))
                .build();
            return chain.filter(exchange.mutate().request(request).build());
        };
    }

    public static class Config {
        private List<String> headers;

        public List<String> getHeaders() { return headers; }
        public void setHeaders(List<String> headers) { this.headers = headers; }
    }
}
```

**YAML 配置**：

```yaml
filters:
  - HeaderInjection=X-Custom:value1,X-Env:prod
```

## 2.5 继承 AbstractNameValueGatewayFilterFactory

对于简单的名值对配置，可以继承 `AbstractNameValueGatewayFilterFactory`：

```java
@Component
public class AddUuidHeaderGatewayFilterFactory
        extends AbstractNameValueGatewayFilterFactory {

    @Override
    public GatewayFilter apply(NameValueConfig config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest().mutate()
                .header(config.getName(), config.getValue())
                .build();
            return chain.filter(exchange.mutate().request(request).build());
        };
    }
}
```

**YAML 配置**：

```yaml
filters:
  - AddUuidHeader=X-Trace-Id, #{T(java.util.UUID).randomUUID()}
```

# 3. 自定义 GlobalFilter

## 3.1 认证过滤器

```java
@Component
@Order(-200)
public class AuthenticationGlobalFilter implements GlobalFilter {

    private static final Set<String> WHITE_LIST = Set.of("/login", "/register", "/actuator");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 白名单放行
        if (WHITE_LIST.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        // 检查 Token
        String token = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (token == null || !token.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid token");
        }

        return verifyToken(token.substring(7))
            .flatMap(claims -> {
                // 将用户信息放入请求头传递给下游
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Name", (String) claims.get("name"))
                    .build();
                return chain.filter(exchange.mutate().request(request).build());
            })
            .onErrorResume(e -> unauthorized(exchange, e.getMessage()));
    }

    private Mono<Claims> verifyToken(String jwt) {
        // JWT 验证逻辑
        return Mono.fromCallable(() -> Jwts.parser()
            .setSigningKey(secretKey)
            .parseClaimsJws(jwt)
            .getBody());
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders()
            .setContentType(MediaType.APPLICATION_JSON);
        byte[] body = ("{\"code\":401,\"message\":\"" + message + "\"}").getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

## 3.2 请求日志过滤器

```java
@Component
@Order(-100)
public class RequestLoggingGlobalFilter implements GlobalFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingGlobalFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // PRE 阶段
        ServerHttpRequest request = exchange.getRequest();
        String method = request.getMethod().name();
        String path = request.getPath().value();
        String query = request.getURI().getRawQuery();
        String ip = request.getRemoteAddress().getAddress().getHostAddress();

        long startTime = System.currentTimeMillis();

        StringBuilder msg = new StringBuilder();
        msg.append("→ ").append(method).append(" ").append(path);
        if (query != null) msg.append("?").append(query);
        msg.append(" from ").append(ip);
        log.info(msg.toString());

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // POST 阶段
            long elapsed = System.currentTimeMillis() - startTime;
            HttpStatus status = exchange.getResponse().getStatusCode();
            log.info("← {} {} ({}ms)", status, path, elapsed);
        }));
    }
}
```

## 3.3 CORS 过滤器

```java
@Component
@Order(-300)
public class CorsGlobalFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        // 处理预检请求
        if (request.getMethod() == HttpMethod.OPTIONS) {
            response.getHeaders().add("Access-Control-Allow-Origin", "*");
            response.getHeaders().add("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
            response.getHeaders().add("Access-Control-Allow-Headers", "Content-Type,Authorization");
            response.getHeaders().add("Access-Control-Max-Age", "3600");
            response.setStatusCode(HttpStatus.OK);
            return response.setComplete();
        }

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            response.getHeaders().add("Access-Control-Allow-Origin", "*");
            response.getHeaders().add("Access-Control-Expose-Headers", "X-Custom-Header");
        }));
    }
}
```

# 4. 自定义 GatewayFilter（非工厂模式）

## 4.1 通过 Java DSL 添加

```java
@Bean
public RouteLocator customRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("custom_filter_route", r -> r
            .path("/custom/**")
            .filters(f -> f.filter((exchange, chain) -> {
                // 自定义过滤逻辑
                System.out.println("Custom filter executed");
                return chain.filter(exchange);
            }))
            .uri("http://localhost:8081"))
        .build();
}
```

## 4.2 可复用的 OrderedGatewayFilter

```java
public class TimingGatewayFilter implements GatewayFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long start = System.nanoTime();
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long elapsed = System.nanoTime() - start;
            exchange.getResponse().getHeaders()
                .add("X-Elapsed-Time", elapsed / 1_000_000 + "ms");
        }));
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.LOWEST_PRECEDENCE;
    }
}

// 在 DSL 中使用
@Bean
public RouteLocator routes(RouteLocatorBuilder builder, TimingGatewayFilter timingFilter) {
    return builder.routes()
        .route("timed_route", r -> r
            .path("/api/**")
            .filters(f -> f.filter(timingFilter))
            .uri("http://backend"))
        .build();
}
```

# 5. 过滤器中的 PRE 与 POST 模式

## 5.1 纯 PRE 过滤器

```java
// 在请求转发前执行，不处理响应
@Override
public GatewayFilter apply(Config config) {
    return (exchange, chain) -> {
        // PRE: 修改请求
        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-Custom", "value")
            .build();
        return chain.filter(exchange.mutate().request(request).build());
        // 没有 then() 块，不处理 POST
    };
}
```

## 5.2 纯 POST 过滤器

```java
// 只处理响应，不修改请求
@Override
public GatewayFilter apply(Config config) {
    return (exchange, chain) -> chain.filter(exchange)
        .then(Mono.fromRunnable(() -> {
            // POST: 修改响应
            exchange.getResponse().getHeaders()
                .add("X-Processed-By", "Gateway");
        }));
}
```

## 5.3 PRE + POST 过滤器

```java
@Override
public GatewayFilter apply(Config config) {
    return (exchange, chain) -> {
        // PRE: 请求处理前
        long start = System.currentTimeMillis();

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // POST: 响应处理后
            long elapsed = System.currentTimeMillis() - start;
            exchange.getResponse().getHeaders()
                .add("X-Elapsed", String.valueOf(elapsed));
        }));
    };
}
```

# 6. 过滤器中获取 Route 信息

```java
@Component
public class RouteInfoFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取当前匹配的路由
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route != null) {
            System.out.println("Route ID: " + route.getId());
            System.out.println("Route URI: " + route.getUri());
            System.out.println("Route Order: " + route.getOrder());
            System.out.println("Route Metadata: " + route.getMetadata());
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.HIGHEST_PRECEDENCE;
    }
}
```

# 7. 过滤器中的异常处理

```java
@Component
public class ExceptionHandlingFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange)
            .onErrorResume(throwable -> {
                // 统一异常处理
                log.error("Request failed: {}", throwable.getMessage());

                if (throwable instanceof TimeoutException) {
                    exchange.getResponse().setStatusCode(HttpStatus.GATEWAY_TIMEOUT);
                } else if (throwable instanceof NotFoundException) {
                    exchange.getResponse().setStatusCode(HttpStatus.NOT_FOUND);
                } else {
                    exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                }

                // 写入错误信息
                byte[] body = ("{\"error\":\"" + throwable.getMessage() + "\"}").getBytes();
                DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
                return exchange.getResponse().writeWith(Mono.just(buffer));
            });
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.HIGHEST_PRECEDENCE + 100;
    }
}
```

# 8. 过滤器中的异步操作

```java
@Component
public class AsyncProcessingFilter implements GlobalFilter, Ordered {

    private final WebClient webClient;

    public AsyncProcessingFilter(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder.build();
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 异步调用外部服务
        return webClient.get()
            .uri("http://auth-service/verify")
            .header("Authorization", exchange.getRequest().getHeaders()
                .getFirst(HttpHeaders.AUTHORIZATION))
            .exchangeToMono(response -> {
                if (response.statusCode().is2xxSuccessful()) {
                    return chain.filter(exchange);
                }
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }

    @Override
    public int getOrder() {
        return -300;
    }
}
```

# 9. 过滤器的测试

```java
@SpringBootTest
@AutoConfigureMockMvc
class CustomFilterTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void testCustomFilter() {
        ResponseEntity<String> response = restTemplate.exchange(
            RequestEntity.get(URI.create("/api/test"))
                .header("X-Version", "v1")
                .build(),
            String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getHeaders().containsKey("X-Custom-Header")).isTrue();
    }
}
```

# 10. 最佳实践

## 10.1 过滤器命名和配置

- 工厂类名以 `GatewayFilterFactory` 结尾
- 配置时使用去掉 `GatewayFilterFactory` 的前缀
- 优先使用 `AbstractGatewayFilterFactory` 而非直接实现接口

## 10.2 性能考量

- 避免在过滤器链中使用阻塞操作
- 异步操作避免在 EventLoop 线程上执行耗时任务
- 合理使用缓存减少重复计算（如 Token 验证结果）

## 10.3 配置参数

- 提供合理的默认值
- 使用 `shortcutFieldOrder()` 定义快捷配置参数顺序
- 复杂的配置使用完整配置方式（name + args）

## 10.4 错误处理

- 不要在过滤器中吞掉异常
- 使用 `onErrorResume` 统一处理错误
- 明确区分客户端错误（4xx）和服务器错误（5xx）
