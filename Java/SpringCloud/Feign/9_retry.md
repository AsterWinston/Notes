
# 1. 概述

Feign 内置了 `Retryer` 接口，控制在**特定可重试异常**发生时是否自动重试。默认行为是**不重试**（`Retryer.NEVER_RETRY`）。

```java
public interface Retryer extends Cloneable {

    /** 是否继续重试，或抛出异常 */
    void continueOrPropagate(RetryableException e);

    /** 每次调用 clone() 返回新实例，承载重试状态 */
    Retryer clone();
}
```

# 2. Retryer.Default 源码解析

```java
public static class Default implements Retryer {

    private final long period;       // 初始重试间隔 (ms)
    private final long maxPeriod;    // 最大重试间隔 (ms)
    private final int maxAttempts;   // 最大重试次数
    private int attempt;             // 当前尝试次数
    private long sleptForMillis;     // 已睡眠总时长达

    public Default() {
        this(100, TimeUnit.SECONDS.toMillis(1), 5);  // 默认
    }

    public Default(long period, long maxPeriod, int maxAttempts) {
        this.period = period;
        this.maxPeriod = maxPeriod;
        this.maxAttempts = maxAttempts;
        this.attempt = 1;
    }

    @Override
    public void continueOrPropagate(RetryableException e) {
        if (attempt++ >= maxAttempts) {
            throw e;   // 达到最大次数，抛出异常
        }
        long interval;
        if (e.retryAfter() != null) {
            // 优先使用服务端返回的 Retry-After
            interval = e.retryAfter().getTime() - System.currentTimeMillis();
            if (interval > maxPeriod) {
                interval = maxPeriod;
            }
        } else {
            // 指数退避: period * 2^(attempt-1)
            interval = nextMaxInterval();
        }
        try {
            Thread.sleep(interval);
        } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
            throw e;
        }
        sleptForMillis += interval;
    }

    long nextMaxInterval() {
        long interval = (long) (period * Math.pow(2, attempt - 1));
        return Math.min(interval, maxPeriod);
    }

    @Override
    public Retryer clone() {
        return new Default(period, maxPeriod, maxAttempts);
    }
}
```

**默认行为**：
- 初始间隔：100ms
- 最大间隔：1000ms (1s)
- 最大次数：5 次
- 退避策略：**指数退避**（100ms → 200ms → 400ms → 800ms → 1000ms）

# 3. 自定义 Retryer

## 3.1 代码方式

```java
public class UserFeignConfig {
    @Bean
    public Retryer feignRetryer() {
        // 初始间隔 200ms，最大间隔 3000ms，最多重试 3 次
        return new Retryer.Default(200, 3000, 3);
    }
}
```

## 3.2 YAML 方式

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:
            retryer: com.example.feign.MyCustomRetryer
```

## 3.3 禁用重试

```java
@Bean
public Retryer feignRetryer() {
    return Retryer.NEVER_RETRY;  // 默认值
}
```

# 4. 自定义重试策略

```java
public class ConditionalRetryer implements Retryer {

    private final int maxAttempts;
    private int attempt;

    public ConditionalRetryer(int maxAttempts) {
        this.maxAttempts = maxAttempts;
        this.attempt = 1;
    }

    @Override
    public void continueOrPropagate(RetryableException e) {
        // 仅对 5xx 且非 501/505 的错误重试
        if (e.status() >= 500 && e.status() != 501 && e.status() != 505) {
            if (attempt++ >= maxAttempts) {
                throw e;
            }
            try {
                Thread.sleep(200L * attempt);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
                throw e;
            }
        } else {
            throw e;
        }
    }

    @Override
    public Retryer clone() {
        return new ConditionalRetryer(maxAttempts);
    }
}
```

# 5. 重试触发条件

Feign 的 `Retryer` **只对 `RetryableException` 响应**。以下情况会抛出 `RetryableException`：

| 场景 | 是否重试 |
|------|----------|
| **连接超时 / 拒绝** (`IOException`) | 是 |
| **读取超时** (`SocketTimeoutException`) | 是 |
| **ErrorDecoder 抛出 `RetryableException`** | 是 |
| **4xx 错误** (默认 ErrorDecoder) | 否 |
| **5xx 错误** (默认 ErrorDecoder) | 否 |

**重点**：默认 ErrorDecoder 不抛出 `RetryableException`，所以默认只有网络 I/O 异常才会触发重试。如果需要基于 HTTP 状态码重试，需自定义 ErrorDecoder。

# 6. ErrorDecoder + Retryer 协同

```java
public class RetryableErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        // 503 Service Unavailable → 可重试
        if (response.status() == 503) {
            return new RetryableException(
                response.status(),
                "Service Unavailable, retry later",
                response.request().httpMethod(),
                null,  // retryAfter (Date)
                response.request()
            );
        }
        // 429 Too Many Requests → 可重试（带 Retry-After）
        if (response.status() == 429) {
            String retryAfter = response.headers()
                    .getOrDefault("Retry-After", Collections.emptyList())
                    .stream().findFirst().orElse("5");
            Date retryAfterDate = new Date(
                System.currentTimeMillis() + Long.parseLong(retryAfter) * 1000
            );
            return new RetryableException(
                response.status(),
                "Rate limited",
                response.request().httpMethod(),
                retryAfterDate,
                response.request()
            );
        }
        return new ErrorDecoder.Default().decode(methodKey, response);
    }
}
```

# 7. 与 Spring Retry 对比

| 维度 | Feign Retryer | Spring Retry |
|------|--------------|--------------|
| **触发条件** | `RetryableException` 或 I/O 异常 | 可配置异常类型白名单 |
| **退避策略** | 指数退避（内置） | Fixed / Exponential Random / Custom |
| **状态性** | 简单计数 | 重试上下文、Recovery 回调 |
| **熔断集成** | 无 | 可配合 Circuit Breaker |

**建议**：重试需求简单时用 Feign `Retryer`；需要更复杂的重试策略（如按异常类型、重试恢复回调），用 Spring Retry + Feign Builder 手动构建。

# 8. 重试对幂等性的影响

```
重试风险：
  POST /orders → 创建订单  → 不幂等！重试可能导致重复创建
  GET  /users/1  → 查询用户  → 幂等，安全重试
  PUT  /users/1  → 更新用户  → 幂等（取决于实现）
```

**建议**：
1. 仅在 **GET / HEAD / OPTIONS** 上启用重试
2. POST 请求如必须重试，需在服务端做**幂等性保证**（唯一请求 ID 去重）
3. 在自定义 `Retryer` 中根据 HTTP 方法决定是否重试：

```java
@Override
public void continueOrPropagate(RetryableException e) {
    if ("GET".equals(e.request().httpMethod().name())) {
        // 仅 GET 重试...
    } else {
        throw e;  // 非 GET 不重试
    }
}
```

# 9. 配置建议

| 场景 | period | maxPeriod | maxAttempts |
|------|--------|-----------|-------------|
| **快速失败（默认）** | — | — | NEVER_RETRY |
| **网络波动容忍** | 500 | 5000 | 3 |
| **临时故障等恢复** | 1000 | 10000 | 5 |
| **极力保证成功** | 500 | 30000 | 10 |
