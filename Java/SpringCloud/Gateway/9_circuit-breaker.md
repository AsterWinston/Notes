
# 1. 熔断降级概述

## 1.1 什么是熔断

熔断（Circuit Breaker）是一种服务保护机制，当后端服务出现故障（超时、异常、拒绝连接等）时，断路器打开，后续请求直接返回降级响应，避免请求堆积和对故障服务的持续调用，防止级联故障（雪崩效应）。

## 1.2 断路器状态机

```
  正常（CLOSED）
      │
      │ 失败率达到阈值
      ▼
  打开（OPEN） ─── 等待超时 ──→ HALF_OPEN
      │                           │
      │                           ├── 请求成功 → CLOSED
      │                           └── 请求失败 → OPEN
      │
      └── 手动重置 → CLOSED
```

| 状态 | 说明 |
|------|------|
| CLOSED | 正常状态，请求正常通过 |
| OPEN | 熔断状态，请求直接降级 |
| HALF_OPEN | 半开状态，允许少量探测请求通过 |

# 2. Spring Cloud Circuit Breaker 集成

## 2.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

## 2.2 核心过滤器源码

`SpringCloudCircuitBreakerFilterFactory` 是网关熔断的核心实现：

```java
public class SpringCloudCircuitBreakerFilterFactory
        extends AbstractGatewayFilterFactory<SpringCloudCircuitBreakerFilterFactory.Config> {

    private final ReactiveCircuitBreakerFactory circuitBreakerFactory;

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 获取断路器名称
            String circuitName = config.getName();
            // 创建断路器
            ReactiveCircuitBreaker circuitBreaker =
                circuitBreakerFactory.create(circuitName);

            // 定义路由标识符
            exchange.getAttributes().put(
                ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR, true);

            // 使用断路器包装请求
            return circuitBreaker.run(
                // 主逻辑：转发请求
                chain.filter(exchange)
                    .doOnSuccess(response -> {
                        // 检查响应状态码是否需要断路
                        HttpStatus status = exchange.getResponse().getStatusCode();
                        if (config.getStatusCodes().contains(status.value())) {
                            throw new CircuitBreakerStatusCodeException(status);
                        }
                    }),
                // 降级逻辑：请求失败时的处理
                throwable -> {
                    // 记录异常信息到 exchange 属性
                    exchange.getAttributes().put(
                        ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR,
                        throwable);

                    // 处理降级请求
                    URI fallbackUri = config.getFallbackUri();
                    if (fallbackUri != null) {
                        return handleFallback(exchange, throwable, fallbackUri);
                    }

                    // 无降级 URI 时直接返回错误
                    return Mono.error(throwable);
                }
            );
        };
    }

    private Mono<Void> handleFallback(ServerWebExchange exchange,
            Throwable throwable, URI fallbackUri) {
        // 如果降级 URI 是以 forward 开头的内部转发
        if (fallbackUri.getScheme().equals("forward")) {
            // 设置内部转发 URL
            exchange.getAttributes().put(
                ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR, fallbackUri);
            ServerWebExchangeUtils.setAlreadyRouted(exchange);

            // 通过 DispatcherHandler 内部转发
            return exchange.getRequest().mutate()
                .path(fallbackUri.getPath())
                .build()
                .let(newRequest -> exchange.mutate().request(newRequest).build())
                .let(mutatedExchange -> {
                    // 获取 ForwardRoutingFilter 处理
                    return Flux.deferContextual(ctx -> {
                        // 省略 DispatcherHandler 内部转发细节
                        return Mono.empty();
                    });
                });
        }
        throw new IllegalArgumentException("Only forward scheme supported for fallback");
    }
}
```

## 2.3 YAML 配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service_with_cb
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/user
                statusCodes:
                  - 500
                  - 502
                  - SERVICE_UNAVAILABLE
            - StripPrefix=2
```

# 3. Resilience4j 配置详解

## 3.1 完整配置

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10                # 滑动窗口大小
        minimum-number-of-calls: 5              # 最少调用次数（达到后才开始计算）
        failure-rate-threshold: 50              # 失败率阈值（百分比）
        wait-duration-in-open-state: 10s        # 熔断打开持续时间
        permitted-number-of-calls-in-half-open-state: 3  # 半开状态允许的请求数
        sliding-window-type: COUNT_BASED        # 滑动窗口类型
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.BusinessException

    instances:
      userServiceCB:
        base-config: default
        failure-rate-threshold: 30              # 更严格的阈值

  timelimiter:
    configs:
      default:
        timeout-duration: 5s                    # 超时时间
        cancel-running-future: true
      userServiceCB:
        timeout-duration: 3s
```

## 3.2 配置参数详解

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `sliding-window-size` | 100 | 滑动窗口大小（请求数） |
| `sliding-window-type` | COUNT_BASED | 计数窗口 / 时间窗口（TIME_BASED） |
| `minimum-number-of-calls` | 100 | 达到此数量才开始计算失败率 |
| `failure-rate-threshold` | 50 | 失败率阈值（百分比），超过则熔断 |
| `wait-duration-in-open-state` | 60s | OPEN 状态持续后自动转为 HALF_OPEN |
| `permitted-number-of-calls-in-half-open-state` | 10 | HALF_OPEN 状态下允许通过的请求数 |
| `automatic-transition-from-open-to-half-open-enabled` | true | OPEN 到 HALF_OPEN 的自动转换 |
| `record-exceptions` | 空 | 指定哪些异常计入失败 |
| `ignore-exceptions` | 空 | 指定哪些异常不计入失败 |

## 3.3 时间窗口配置

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: TIME_BASED
        sliding-window-size: 10        # 10 秒
        minimum-number-of-calls: 5
```

# 4. 降级处理

## 4.1 forward 降级

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: userServiceCB
      fallbackUri: forward:/fallback/user
```

定义降级路由：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_fallback
          uri: forward:/fallback/user
          predicates:
            - Path=/fallback/user
```

降级处理 Controller：

```java
@RestController
public class FallbackController {

    @PostMapping("/fallback/user")
    public Mono<ServerResponse> userFallback(ServerWebExchange exchange) {
        // 获取熔断异常信息
        Throwable exception = exchange.getAttribute(
            ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR);

        return ServerResponse.status(HttpStatus.SERVICE_UNAVAILABLE)
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new FallbackResponse(
                503,
                "Service temporarily unavailable",
                exception != null ? exception.getMessage() : "Unknown error"
            ));
    }

    record FallbackResponse(int code, String message, String detail) {}
}
```

## 4.2 FallbackHeaders 过滤器

将熔断异常信息以 Header 形式传递到降级端点：

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: userServiceCB
      fallbackUri: forward:/fallback/user
  - name: FallbackHeaders
    args:
      executionExceptionTypeHeaderName: X-Exception-Type
      executionExceptionMessageHeaderName: X-Exception-Message
      rootCauseExceptionTypeHeaderName: X-Root-Cause-Type
      rootCauseExceptionMessageHeaderName: X-Root-Cause-Message
```

```java
@RestController
public class FallbackController {

    @GetMapping("/fallback/user")
    public Mono<ServerResponse> userFallback(ServerRequest request) {
        String exceptionType = request.headers()
            .firstHeader("X-Exception-Type");
        String exceptionMessage = request.headers()
            .firstHeader("X-Exception-Message");
        String rootCauseType = request.headers()
            .firstHeader("X-Root-Cause-Type");
        String rootCauseMessage = request.headers()
            .firstHeader("X-Root-Cause-Message");

        return ServerResponse.status(HttpStatus.SERVICE_UNAVAILABLE)
            .bodyValue(Map.of(
                "code", 503,
                "exceptionType", exceptionType,
                "exceptionMessage", exceptionMessage,
                "rootCauseType", rootCauseType,
                "rootCauseMessage", rootCauseMessage
            ));
    }
}
```

# 5. 熔断与重试配合

融合 `CircuitBreaker` 和 `Retry` 过滤器可以实现更完善的容错策略：

```yaml
filters:
  - name: Retry
    args:
      retries: 2
      statuses: BAD_GATEWAY
      methods: GET
      backoff:
        firstBackoff: 200ms
        maxBackoff: 2s
        factor: 2
  - name: CircuitBreaker
    args:
      name: userServiceCB
      fallbackUri: forward:/fallback/user
      statusCodes:
        - 500
        - 502
        - 503
        - 504
```

**执行顺序**：
1. 先执行 Retry（重试 2 次）
2. 若重试失败，进入 CircuitBreaker
3. CircuitBreaker 执行降级逻辑

# 6. 熔断监控

## 6.1 Actuator 端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: circuitbreakers,circuitbreakerevents
```

```bash
# 查看所有断路器状态
curl http://localhost:8080/actuator/circuitbreakers

# 查看断路器事件
curl http://localhost:8080/actuator/circuitbreakerevents
```

## 6.2 Metrics 指标

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
```

自动暴露的 Resilience4j 指标：

| 指标 | 类型 | 说明 |
|------|------|------|
| `resilience4j.circuitbreaker.state` | Gauge | 断路器状态（0=CLOSED, 1=OPEN, 2=HALF_OPEN） |
| `resilience4j.circuitbreaker.calls` | Timer | 调用耗时 |
| `resilience4j.circuitbreaker.buffered` | Gauge | 滑动窗口中的缓冲调用数 |
| `resilience4j.circuitbreaker.failure_rate` | Gauge | 当前失败率 |
| `resilience4j.circuitbreaker.slow_calls` | Timer | 慢调用耗时 |

# 7. 最佳实践

## 7.1 合理的熔断参数

| 场景 | slidingWindowSize | failureRateThreshold | waitDurationInOpenState | permittedCallsInHalfOpen |
|------|------------------|---------------------|------------------------|-------------------------|
| 核心服务 | 20 | 30 | 15s | 5 |
| 非核心服务 | 10 | 50 | 10s | 3 |
| 外部依赖 | 30 | 40 | 30s | 5 |

## 7.2 超时与熔断配合

```yaml
resilience4j:
  timelimiter:
    instances:
      userServiceCB:
        timeout-duration: 3s
  circuitbreaker:
    instances:
      userServiceCB:
        sliding-window-size: 10
        failure-rate-threshold: 50
```

**注意**：`timeout-duration` 必须小于网关的 `response-timeout`，否则网关超时优先于熔断超时。

## 7.3 降级响应的设计

```java
@RestController
public class UnifiedFallbackController {

    @RequestMapping("/fallback/{service}")
    public Mono<Map<String, Object>> fallback(
            @PathVariable String service,
            ServerWebExchange exchange) {

        Throwable exception = exchange.getAttribute(
            ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR);

        Map<String, Object> result = new HashMap<>();
        result.put("code", 503);
        result.put("message", service + " is temporarily unavailable");
        result.put("timestamp", Instant.now().toString());
        result.put("path", exchange.getRequest().getPath().value());

        if (exception != null) {
            result.put("exception", exception.getClass().getSimpleName());
            result.put("detail", exception.getMessage());
        }

        return Mono.just(result);
    }
}
```

## 7.4 注意事项

1. **熔断粒度**：按服务或按接口维度配置断路器，避免不同接口相互影响
2. **降级响应**：降级响应应包含足够信息便于排查，但不要暴露敏感信息
3. **超时配置**：确保 `timelimiter.timeout-duration` < 网关的 `response-timeout`
4. **状态码触发**：除 5xx 外，也可以配置特定 4xx 状态码触发熔断
5. **慢调用熔断**：配置 `slow-call-duration-threshold` 和 `slow-call-rate-threshold` 实现慢调用熔断
6. **noop 断路器**：测试环境中可以使用 `CircuitBreakerFactory.NoOpCircuitBreaker` 禁用熔断
