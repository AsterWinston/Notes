
# 1. 概述

Spring Cloud LoadBalancer 的重试机制基于 Spring Retry，当请求失败时自动选择**下一个**服务实例重试，提高调用成功率。

```
请求 user-service
  │
  ▼
负载均衡 → 选择 实例A
  │
  ├── 请求成功 (200) ✓ → 返回
  │
  └── 请求失败 (503)
        │
        ▼
      重试 → 选择 实例B
        │
        ├── 请求成功 (200) ✓ → 返回
        │
        └── 请求失败 (503)
              │
              ▼
            重试 → 选择 实例C
              │
              └── 请求成功 (200) ✓ → 返回
```

# 2. 依赖与配置

## 2.1 添加依赖

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<!-- 可选：AOP 支持便于使用 @Retryable -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

## 2.2 配置属性

```yaml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
        retry-on-all-operations: false    # 默认仅对 GET 重试
        max-retries-on-same-service-instance: 0   # 同一实例不重试
        max-retries-on-next-service-instance: 2   # 最多尝试 2 个新实例
        retryable-status-codes:           # 触发重试的 HTTP 状态码
          - 500
          - 502
          - 503
          - 504
        backoff:
          enabled: true
          min-backoff: 100ms              # 最小退避时间
          max-backoff: 1000ms             # 最大退避时间
          multiplier: 2.0                 # 退避倍数
```

# 3. 重试机制源码解析

## 3.1 RetryableLoadBalancerExchangeFilterFunction（WebClient）

```java
public class RetryableLoadBalancerExchangeFilterFunction
        extends LoadBalancerExchangeFilterFunction {

    private final LoadBalancerRetryPolicy retryPolicy;
    private final LoadBalancerProperties properties;

    @Override
    public Mono<ClientResponse> filter(
            ClientRequest request, ExchangeFunction next) {
        URI originalUrl = request.url();
        String serviceId = originalUrl.getHost();

        // 构建重试监听器
        RetryListener retryListener = getRetryListener(serviceId);

        // 构建重试模板
        RetryTemplate retryTemplate = createRetryTemplate(serviceId);

        return Mono.fromCallable(() -> {
                    // 执行带重试的请求
                    return retryTemplate.execute(context -> {
                        // ① 选择实例
                        ServiceInstance instance = choose(serviceId, request);

                        // ② 替换 URL
                        ClientRequest newRequest = ClientRequest.from(request)
                                .url(reconstructURI(instance, originalUrl))
                                .build();

                        // ③ 发出请求
                        ClientResponse response = next.exchange(newRequest)
                                .block();

                        // ④ 判断是否需要重试
                        if (shouldRetry(response)) {
                            throw new RetryableStatusCodeException(
                                    response.rawStatusCode());
                        }

                        return response;
                    });
                })
                .subscribeOn(Schedulers.boundedElastic());
    }
}
```

## 3.2 RetryableLoadBalancerInterceptor（RestTemplate）

```java
public class RetryableLoadBalancerInterceptor
        implements ClientHttpRequestInterceptor {

    private final LoadBalancerClient loadBalancerClient;
    private final LoadBalancerRetryPolicy retryPolicy;
    private final LoadBalancerProperties properties;

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        URI originalUri = request.getURI();
        String serviceId = originalUri.getHost();

        RetryTemplate retryTemplate = createRetryTemplate(serviceId);

        return retryTemplate.execute(context -> {
            // ① 选择实例
            ServiceInstance instance = loadBalancerClient.choose(serviceId);

            // ② 重建请求
            HttpRequest newRequest = new ServiceRequestWrapper(request, instance);

            // ③ 执行
            ClientHttpResponse response = execution.execute(newRequest, body);

            // ④ 判断重试
            if (shouldRetry(response)) {
                throw new RetryableStatusCodeException(
                        response.getRawStatusCode());
            }

            return response;
        });
    }
}
```

# 4. 重试策略详解

## 4.1 RetryTemplate 配置

```java
private RetryTemplate createRetryTemplate(String serviceId) {
    RetryTemplate template = new RetryTemplate();

    // 退避策略
    ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
    backOffPolicy.setInitialInterval(properties.getRetry()
            .getBackoff().getMinBackoff().toMillis());
    backOffPolicy.setMaxInterval(properties.getRetry()
            .getBackoff().getMaxBackoff().toMillis());
    backOffPolicy.setMultiplier(properties.getRetry()
            .getBackoff().getMultiplier());
    template.setBackOffPolicy(backOffPolicy);

    // 重试策略
    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
    retryPolicy.setMaxAttempts(
            properties.getRetry().getMaxRetriesOnNextServiceInstance() + 1);
    template.setRetryPolicy(retryPolicy);

    return template;
}
```

## 4.2 重试决策流程

```
请求失败 (503)
  │
  ▼
shouldRetry(response)
  │
  ├── 状态码在 retryable-status-codes 中？
  │     ├── 是 → 进入重试流程
  │     └── 否 → 直接返回失败
  │
  ▼
retryOnAllOperations = true？
  │
  ├── 是 → 所有方法都可重试
  │
  └── 否 → 仅 GET 方法可重试（防止重复提交 POST/PUT/DELETE）
```

## 4.3 幂等性注意事项

```java
// ⚠️ POST 请求默认不重试（避免重复创建资源）
// 如果业务保证幂等，可以开启对所有操作重试：

@FeignClient(name = "order-service")
public interface OrderClient {

    // 此接口操作幂等，可以安全重试
    @PostMapping("/api/orders")
    Order createOrder(@RequestBody CreateOrderRequest req);
}

// 配置：
spring.cloud.loadbalancer.retry.retry-on-all-operations=true
```

# 5. 重试 + 新一代实例选择

```java
public class NextInstanceRetryPolicy implements RetryPolicy {

    private final Map<String, Set<String>> triedInstances = new ConcurrentHashMap<>();

    @Override
    public boolean canRetry(RetryContext context) {
        String serviceId = context.getAttribute("serviceId");
        String lastInstance = context.getAttribute("lastInstance");

        // 记录已尝试的实例
        triedInstances.computeIfAbsent(serviceId, k -> new HashSet<>())
                .add(lastInstance);

        // 判断是否还有新实例可尝试
        List<ServiceInstance> allInstances = getAllInstances(serviceId);
        long untried = allInstances.stream()
                .filter(i -> !triedInstances.get(serviceId)
                        .contains(getKey(i)))
                .count();

        return untried > 0;
    }

    private String getKey(ServiceInstance instance) {
        return instance.getHost() + ":" + instance.getPort();
    }
}
```

# 6. Feign 重试 vs LoadBalancer 重试

| 维度 | LoadBalancer 重试 | Feign 自带重试 |
|------|-------------------|---------------|
| **粒度** | 负载均衡层（换实例重试） | HTTP 客户端层 |
| **实例切换** | ✓ 自动换下一个实例 | ✗ 同一实例 |
| **Spring Retry** | ✓ 基于 Spring Retry | ✗ |
| **配置位置** | `spring.cloud.loadbalancer.retry.*` | `feign.client.config.*.retryer` |
| **推荐** | 生产环境使用 LoadBalancer 重试 | 不推荐（已废弃） |

```yaml
# 推荐：关闭 Feign 自带重试，使用 LoadBalancer 重试
feign:
  client:
    config:
      default:
        retryer: feign.Retryer.NEVER_RETRY
```

# 7. 重试监控

```java
@Component
public class RetryMetricsCollector implements RetryListener {

    private final MeterRegistry meterRegistry;

    @Override
    public <T, E extends Throwable> boolean open(
            RetryContext context, RetryCallback<T, E> callback) {
        meterRegistry.counter("loadbalancer.retry.attempt").increment();
        return true;
    }

    @Override
    public <T, E extends Throwable> void onError(
            RetryContext context, RetryCallback<T, E> callback,
            Throwable throwable) {
        meterRegistry.counter("loadbalancer.retry.error",
                "exception", throwable.getClass().getSimpleName()).increment();
    }

    @Override
    public <T, E extends Throwable> void close(
            RetryContext context, RetryCallback<T, E> callback,
            Throwable throwable) {
        if (context.getRetryCount() > 0) {
            meterRegistry.summary("loadbalancer.retry.count")
                    .record(context.getRetryCount());
        }
    }
}
```

# 8. 最佳实践

```yaml
# 生产环境推荐配置
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
        retry-on-all-operations: false     # 仅 GET 重试
        max-retries-on-same-service-instance: 0   # 不重试同一实例
        max-retries-on-next-service-instance: 2   # 最多换 2 个实例
        retryable-status-codes:
          - 502
          - 503
          - 504
        backoff:
          enabled: true
          min-backoff: 100ms
          max-backoff: 500ms
          multiplier: 1.5

# 关闭 Feign 自带重试
feign:
  client:
    config:
      default:
        retryer: feign.Retryer.NEVER_RETRY
```

**关键原则**：

| 原则 | 说明 |
|------|------|
| 只重试幂等操作 | POST/PUT/DELETE 默认不重试 |
| 必须换实例重试 | 同一实例重试无意义 |
| 合理退避 | 避免重试风暴，指数退避 |
| 限制重试次数 | 防止无限重试耗尽资源 |
| 监控告警 | 重试率突增通常是后端故障信号 |
