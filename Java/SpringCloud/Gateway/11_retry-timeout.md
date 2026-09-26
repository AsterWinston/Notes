
# 1. RetryGatewayFilter 源码解析

## 1.1 过滤器工厂

```java
public class RetryGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RetryGatewayFilterFactory.RetryConfig> {

    public RetryGatewayFilterFactory() {
        super(RetryConfig.class);
    }

    @Override
    public GatewayFilter apply(RetryConfig config) {
        return new OrderedGatewayFilter(new RetryFilter(config), 10002);
    }

    private static class RetryFilter implements GatewayFilter {
        private final RetryConfig config;

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            return Retry.create(config.getRetryTemplate())
                .executeComposable(context -> chain.filter(exchange)
                    .doOnError(throwable -> {
                        // 检查是否属于可重试的异常
                        if (!shouldRetry(throwable, config)) {
                            // 不可重试的异常，直接传播
                            throw RetryExhaustedException.wrap(throwable);
                        }
                    })
                    .flatMap(response -> {
                        // 检查响应状态码是否属于可重试范围
                        HttpStatus status = exchange.getResponse().getStatusCode();
                        if (status != null && !config.getStatuses().contains(status)
                                && !config.getSeries().contains(status.series())) {
                            return Mono.error(new NonRetryableException(status));
                        }
                        return Mono.just(response);
                    })
                );
        }
    }
}
```

## 1.2 RetryConfig 配置类

```java
public static class RetryConfig {
    private int retries = 3;                              // 重试次数
    private List<HttpStatus> statuses = new ArrayList<>(); // 可重试的状态码
    private List<HttpMethod> methods = new ArrayList<>(    // 可重试的 HTTP 方法
        Arrays.asList(HttpMethod.GET));
    private List<HttpStatus.Series> series = new ArrayList<>( // 可重试的状态码系列
        Arrays.asList(HttpStatus.Series.SERVER_ERROR));
    private List<Class<Throwable>> exceptions = new ArrayList<>( // 可重试的异常
        Arrays.asList(IOException.class, TimeoutException.class));
    private BackoffConfig backoff = new BackoffConfig();  // 退避策略

    // 内部退避配置
    public static class BackoffConfig {
        private Duration firstBackoff = Duration.ofMillis(100); // 首次退避时间
        private Duration maxBackoff = Duration.ofMillis(10000); // 最大退避时间
        private int factor = 2;              // 指数退避因子
        private boolean basedOnPreviousValue = false; // 基于前一次的重试时间
    }
}
```

# 2. Retry 过滤器配置

## 2.1 基本配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: retry_route
          uri: http://backend-service
          predicates:
            - Path=/api/**
          filters:
            - name: Retry
              args:
                retries: 3                        # 重试次数
                statuses:                         # 响应状态码触发的重试
                  - BAD_GATEWAY
                  - SERVICE_UNAVAILABLE
                  - GATEWAY_TIMEOUT
                methods: GET                      # 仅对 GET 请求重试
                series: SERVER_ERROR               # 5xx 系列触发重试
                exceptions:                       # 异常类型触发的重试
                  - java.io.IOException
                  - java.util.concurrent.TimeoutException
                backoff:
                  firstBackoff: 100ms             # 首次等待 100ms
                  maxBackoff: 5s                  # 最长等待 5s
                  factor: 2                       # 指数因子：100ms → 200ms → 400ms
                  basedOnPreviousValue: false     # 使用原始退避配置
```

## 2.2 带重试的降级方案

结合 `Retry` 和 `CircuitBreaker` 实现多层容错：

```yaml
filters:
  - name: Retry
    args:
      retries: 2
      statuses:
        - BAD_GATEWAY
        - SERVICE_UNAVAILABLE
        - GATEWAY_TIMEOUT
      backoff:
        firstBackoff: 200ms
        maxBackoff: 2s
        factor: 2
  - name: CircuitBreaker
    args:
      name: backendCB
      fallbackUri: forward:/fallback
      statusCodes:
        - 500
        - 502
        - 503
        - 504
```

**处理流程**：
1. 第一次请求失败
2. 重试机制：等 200ms → 重试 → 等 400ms → 重试
3. 如果重试耗尽，进入 CircuitBreaker
4. CircuitBreaker 记录失败并执行降级

# 3. 超时控制

## 3.1 全局超时配置

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 5000          # 连接超时（毫秒）
        response-timeout: 10s          # 响应超时（Duration 格式）
        max-header-size: 16384
        max-initial-line-length: 4096
```

## 3.2 按路由超时配置

通过路由元数据单独设置超时：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: fast_service
          uri: lb://fast-service
          predicates:
            - Path=/fast/**
          metadata:
            response-timeout: 200       # 200ms
            connect-timeout: 200        # 200ms

        - id: slow_service
          uri: lb://slow-service
          predicates:
            - Path=/slow/**
          metadata:
            response-timeout: 30000     # 30s
            connect-timeout: 10000      # 10s

        - id: no_timeout_service
          uri: lb://no-timeout
          predicates:
            - Path=/no-timeout/**
          metadata:
            response-timeout: -1        # 负值表示不设超时
```

## 3.3 超时处理源码分析

```java
// NettyRoutingFilter 中的超时处理
public class NettyRoutingFilter implements GlobalFilter, Ordered {

    private final HttpClient httpClient;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取响应超时配置（优先路由元数据，其次全局）
        Long responseTimeout = exchange.getAttribute(
            ServerWebExchangeUtils.CLIENT_RESPONSE_TIMEOUT_ATTR);

        Mono<HttpClientResponse> responseMono = this.httpClient
            .request(method)
            .uri(requestUrl.toString())
            .send(...)
            .responseConnection(...);

        if (responseTimeout != null && responseTimeout > 0) {
            responseMono = responseMono.timeout(Duration.ofMillis(responseTimeout))
                .onErrorMap(TimeoutException.class,
                    e -> new ResponseStatusException(HttpStatus.GATEWAY_TIMEOUT,
                        "Response took longer than " + responseTimeout + "ms", e));
        }

        return responseMono.flatMap(responseBody -> ...);
    }
}
```

## 3.4 自定义超时过滤器

```java
@Component
public class TimeoutGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route != null && route.getMetadata().containsKey("response-timeout")) {
            Number timeout = (Number) route.getMetadata().get("response-timeout");
            long timeoutMs = timeout.longValue();

            if (timeoutMs > 0) {
                // 设置超时
                exchange.getAttributes().put(
                    ServerWebExchangeUtils.CLIENT_RESPONSE_TIMEOUT_ATTR, timeoutMs);
            } else if (timeoutMs < 0) {
                // 负值：禁用超时
                exchange.getAttributes().remove(
                    ServerWebExchangeUtils.CLIENT_RESPONSE_TIMEOUT_ATTR);
            }
        }

        if (route != null && route.getMetadata().containsKey("connect-timeout")) {
            Number connectTimeout = (Number) route.getMetadata().get("connect-timeout");
            exchange.getAttributes().put(
                ServerWebExchangeUtils.CLIENT_CONNECT_TIMEOUT_ATTR,
                connectTimeout.longValue());
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return RouteToRequestUrlFilter.ROUTE_TO_URL_FILTER_ORDER - 1;
    }
}
```

# 4. 指数退避策略

## 4.1 退避公式

```
基于前一次退避（basedOnPreviousValue = false）：
  delay = min(firstBackoff × factor^(attempt - 1), maxBackoff)

基于当前退避（basedOnPreviousValue = true）：
  delay = min(previousDelay × factor, maxBackoff)
```

## 4.2 退避示例

```yaml
filters:
  - name: Retry
    args:
      retries: 4
      backoff:
        firstBackoff: 1s
        maxBackoff: 30s
        factor: 3
        basedOnPreviousValue: false
```

| 重试次数 | 等待时间 |
|----------|----------|
| 第 1 次 | 1s = 1 × 3^0 |
| 第 2 次 | 3s = 1 × 3^1 |
| 第 3 次 | 9s = 1 × 3^2 |
| 第 4 次 | 27s = min(1 × 3^3, 30) |

## 4.3 带随机抖动的退避

```java
@Component
public class JitterRetryFilter implements GatewayFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return Retry.withThrowable(
            Retry.anyOf(IOException.class, TimeoutException.class)
                .retryMax(3)
                .doOnRetry(ctx -> log.warn("Retry: attempt {}", ctx.currentRetry()))
                .backoff(Schedulers.parallel(), Duration.ofMillis(100))
                .jitter(0.5) // 50% 的随机抖动
        ).apply(chain.filter(exchange));
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.LOWEST_PRECEDENCE;
    }
}
```

# 5. 重试的幂等性

## 5.1 重试安全指南

```yaml
filters:
  - name: Retry
    args:
      methods: GET,HEAD,OPTIONS       # 幂等方法
      # methods: GET                  # 建议仅对 GET 使用重试
      statuses:
        - BAD_GATEWAY
        - SERVICE_UNAVAILABLE
        - GATEWAY_TIMEOUT
      series: SERVER_ERROR             # 5xx
```

## 5.2 自定义重试条件

```java
@Component
public class IdempotentRetryFilterFactory
        extends AbstractGatewayFilterFactory<RetryGatewayFilterFactory.RetryConfig> {

    public IdempotentRetryFilterFactory() {
        super(RetryGatewayFilterFactory.RetryConfig.class);
    }

    @Override
    public GatewayFilter apply(RetryConfig config) {
        return (exchange, chain) -> {
            HttpMethod method = exchange.getRequest().getMethod();

            // 只对幂等方法启用重试
            if (!isIdempotent(method)) {
                return chain.filter(exchange);
            }

            return Retry.create(config.getRetryTemplate())
                .executeComposable(context -> chain.filter(exchange)
                    .onErrorResume(throwable -> {
                        if (shouldRetry(throwable, config)) {
                            return Mono.error(throwable); // 继续重试
                        }
                        return Mono.error(throwable);     // 不重试
                    }));
        };
    }

    private boolean isIdempotent(HttpMethod method) {
        return method == HttpMethod.GET
            || method == HttpMethod.HEAD
            || method == HttpMethod.OPTIONS
            || method == HttpMethod.PUT
            || method == HttpMethod.DELETE;
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("retries");
    }
}
```

# 6. 最佳实践

## 6.1 参数推荐

| 场景 | retries | firstBackoff | maxBackoff | factor | methods |
|------|---------|-------------|------------|--------|---------|
| 轻量查询 | 2 | 100ms | 2s | 2 | GET |
| 批量查询 | 3 | 500ms | 5s | 2 | GET |
| 关键写入 | 1 | 1s | 5s | 2 | PUT（需幂等） |
| 实时服务 | 0 | - | - | - | 全部 |

## 6.2 超时设置原则

```yaml
# 各层超时时间关系
# timelimiter.timeout-duration < response-timeout < 客户端超时

resilience4j:
  timelimiter:
    instances:
      service:
        timeout-duration: 5s    # 熔断超时

spring:
  cloud:
    gateway:
      httpclient:
        response-timeout: 8s    # 网关超时

# 客户端侧超时建议设为 10s
```

## 6.3 重试风暴防护

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: critical_service
          uri: lb://critical-service
          predicates:
            - Path=/api/critical/**
          filters:
            # 限制重试次数和并发
            - name: Retry
              args:
                retries: 2                # 最多重试 2 次
                backoff:
                  firstBackoff: 500ms
                  maxBackoff: 2s
                  factor: 2
```

## 6.4 监控与告警

```yaml
logging:
  level:
    reactor.retry: DEBUG
    org.springframework.cloud.gateway.filter.factory.RetryGatewayFilterFactory: DEBUG
```

重试相关指标：

| 指标 | 说明 |
|------|------|
| 重试次数 | 过多的重试表示后端服务不稳定 |
| 重试成功率 | 重试后成功的比例 |
| 超时次数 | 超时是重试和熔断的重要触发条件 |

## 6.5 常见坑

1. **超时与重试的关系**：重试会重置超时计时器，每次重试都是独立请求
2. **请求体缓存**：重试需要重新发送请求体，确保 `AdaptCachedBodyGlobalFilter` 已缓存 Body
3. **非幂等操作**：POST/PATCH 等非幂等方法重试可能导致数据重复
4. **重试次数过多**：叠加的重试会放大对后端服务的压力（重试风暴）
5. **退避时间过长**：指数退避可能导致部分请求等待时间过长
