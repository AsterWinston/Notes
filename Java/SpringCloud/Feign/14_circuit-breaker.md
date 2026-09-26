
# 1. 概述

Spring Cloud OpenFeign 从**2020.0.x 版本**开始，将熔断支持从 Hystrix 迁移到 **Resilience4j**。当 Feign 调用失败率达到阈值时，熔断器打开，直接走降级逻辑，避免雪崩效应。

```
正常状态 (CLOSED): 请求正常通过 → 统计失败率
    │
    │ 失败率 >= 阈值 (如 50%)
    ▼
熔断状态 (OPEN): 直接走 fallback → 等待冷却时间
    │
    │ 冷却时间到
    ▼
半开状态 (HALF_OPEN): 允许少量请求通过
    │
    ├── 成功 → 恢复到 CLOSED
    └── 失败 → 重新进入 OPEN
```

# 2. 依赖引入

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<!-- 如果需要响应式 API -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
</dependency>
```

# 3. 基本配置

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
        alphanumeric-ids:
          enabled: true          # Group 名只允许字母数字

resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED   # 滑动窗口类型
        sliding-window-size: 10            # 窗口大小
        minimum-number-of-calls: 5         # 最小调用次数才开始统计
        failure-rate-threshold: 50         # 失败率阈值(%)
        wait-duration-in-open-state: 10000ms      # OPEN 持续时间
        permitted-number-of-calls-in-half-open-state: 5   # HALF_OPEN 允许请求数
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:                 # 记录为失败的异常
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.io.IOException
        ignore-exceptions:                 # 忽略的异常（不计入失败统计）
          - com.example.exception.BusinessException
    instances:
      user-service:
        base-config: default
        sliding-window-size: 20
        wait-duration-in-open-state: 15000ms

  timelimiter:
    configs:
      default:
        timeout-duration: 5s               # 调用超时熔断
        cancel-running-future: true        # 超时后取消任务
    instances:
      user-service:
        timeout-duration: 3s
```

# 4. FeignClient 使用 Fallback

```java
@FeignClient(name = "user-service",
             fallback = UserClientFallback.class)
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
}
```

```java
@Component
public class UserClientFallback implements UserClient {

    @Override
    public User getUser(Long id) {
        // 降级逻辑：返回缓存数据或兜底值
        return new User(id, "N/A", 0);
    }
}
```

Spring Cloud OpenFeign 会自动将 fallback 与 Resilience4j 熔断器绑定。

# 5. FallbackFactory（获取异常）

```java
@FeignClient(name = "user-service",
             fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient { }
```

```java
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public User getUser(Long id) {
                log.error("调用 user-service 失败, userId={}", id, cause);
                if (cause instanceof FeignException.NotFound) {
                    return null;
                }
                return new User(id, "N/A", 0);
            }
        };
    }
}
```

# 6. 配置作用于全局/特定服务

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        failure-rate-threshold: 50
    instances:
      user-service:           # 对应 @FeignClient name
        failure-rate-threshold: 30
        sliding-window-size: 20
      order-service:
        failure-rate-threshold: 40
```

# 7. CircuitBreaker + TimeLimiter 协同

```
Feign 调用
    │
    ├── TimeLimiter  (超时控制)
    │       └── 超时后抛出 TimeoutException → 触发 CircuitBreaker
    │
    ├── CircuitBreaker  (状态机)
    │       ├── CLOSED → 放行请求
    │       └── OPEN → 直接 fallback
    │
    └── Fallback / FallbackFactory
```

**重要**：TimeLimiter 超时时间应 **小于** Feign readTimeout，否则 Feign 的读取超时可能先触发，导致 TimeLimiter 无法生效。

```yaml
spring.cloud.openfeign.client.config.user-service.read-timeout: 8000  # 8s
resilience4j.timelimiter.instances.user-service.timeout-duration: 5s   # 5s < 8s ✓
```

# 8. Bulkhead（信号量隔离）

```yaml
resilience4j:
  bulkhead:
    configs:
      default:
        max-concurrent-calls: 20              # 最大并发数
        max-wait-duration: 500ms             # 等待超时
    instances:
      user-service:
        max-concurrent-calls: 10
```

**SemaphoreBulkhead**（信号量隔离）是 Spring Cloud + Feign 场景的默认选择（因为异步线程池隔离与 WebMVC 冲突）。

# 9. 自定义 CircuitBreaker 工厂

```java
@Configuration
public class FeignCircuitBreakerConfig {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> {
            return new Resilience4JConfigBuilder(id)
                    .circuitBreakerConfig(CircuitBreakerConfig.custom()
                            .slidingWindowType(SlidingWindowType.COUNT_BASED)
                            .slidingWindowSize(10)
                            .failureRateThreshold(50)
                            .build())
                    .timeLimiterConfig(TimeLimiterConfig.custom()
                            .timeoutDuration(Duration.ofSeconds(5))
                            .build())
                    .build();
        });
    }
}
```

# 10. 手动创建 CircuitBreaker FeignClient

```java
@Bean
public UserClient userClient(
        Resilience4JCircuitBreakerFactory circuitBreakerFactory,
        ObjectFactory<HttpMessageConverters> converters) {

    CircuitBreaker circuitBreaker = circuitBreakerFactory.create("user-service");

    UserClient directClient = Feign.builder()
            .encoder(new SpringEncoder(converters))
            .decoder(new SpringDecoder(converters))
            .target(UserClient.class, "http://user-service");

    // 包装为带熔断的代理
    return FeignCircuitBreaker.builder()
            .factory(circuitBreakerFactory)
            .target(directClient, UserClient.class);
}
```

# 11. Hystrix 迁移指南

| Hystrix | Resilience4j 替代 |
|---------|------------------|
| `hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds` | `resilience4j.timelimiter.configs.default.timeout-duration` |
| `hystrix.command.default.circuitBreaker.requestVolumeThreshold` | `resilience4j.circuitbreaker.configs.default.sliding-window-size` |
| `hystrix.command.default.circuitBreaker.errorThresholdPercentage` | `resilience4j.circuitbreaker.configs.default.failure-rate-threshold` |
| `hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds` | `resilience4j.circuitbreaker.configs.default.wait-duration-in-open-state` |
| 线程池隔离 | `resilience4j.bulkhead` (信号量) 或 `resilience4j-threadpoolbulkhead` |

# 12. 熔断指标监控

Resilience4j 内置 Micrometer 支持，可直接接入 Prometheus + Grafana：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}

resilience4j:
  circuitbreaker:
    metrics:
      legacy:
        enabled: false
      enable-micrometer: true
```

可观测的核心指标：
- `resilience4j_circuitbreaker_state` — 熔断器状态 (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
- `resilience4j_circuitbreaker_calls_seconds` — 调用耗时
- `resilience4j_circuitbreaker_failure_rate` — 失败率
- `resilience4j_circuitbreaker_slow_call_rate` — 慢调用率

# 13. 最佳实践

1. **TimeLimiter 超时 < Feign readTimeout** — 让熔断器超时先于底层 HTTP 超时
2. **降级逻辑要轻量** — Fallback 中避免再调用其他 Feign 接口
3. **配置 `record-exceptions`** — 明确哪些异常应计入失败
4. **配合 Bulkhead** — 在高并发场景限制并发数，防止资源耗尽
5. **HALF_OPEN 允许多个请求** — `permitted-number-of-calls-in-half-open-state` 不宜过小
