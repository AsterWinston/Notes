# 1. 概述

## 1.1 定位与默认值

Feign 内置了 `Retryer` 接口，控制在**特定可重试异常**发生时是否自动重试。默认行为是**不重试**（`Retryer.NEVER_RETRY`）。

```java
public interface Retryer extends Cloneable {

    /** 是否继续重试，或抛出异常 */
    void continueOrPropagate(RetryableException e);

    /** 每次调用 clone() 返回新实例，承载重试状态 */
    Retryer clone();
}
```

"默认不重试"是 Spring Cloud 化之后的策略（OpenFeign 原生默认反而是 `Retryer.Default` 5 次）：远程调用的重试在微服务里是**放大器**，必须显式决策。也因此排查"Feign 怎么自动重试了"的问题，第一站永远是确认有人把 Retryer Bean 换掉了。

## 1.2 重试循环在管线中的位置

重试发生在 `SynchronousMethodHandler.executeAndDecode` 外层的 `while(true)` 里（源码见 §5 与 `7_interceptor.md` §8.1）：一次完整尝试 = 拦截器 → 编码 → 发送 → 解码。这意味着**每次重试都会重新执行拦截器**，并且拿到一个全新的 `Response`——重试不是"重放报文"，而是"从模板重建请求再走一遍"。

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

## 2.1 三个容易误读的细节

1. **maxAttempts 是"总尝试数"而非"重试数"**：`attempt` 从 1 起、先比较后自增，默认 5 = 首次 + 4 次重试。配置表（§9）按此口径换算。
2. **clone() 是线程安全的实现方式**：Retryer 单例作为配置 Bean，每次 `invoke` 先 `this.retryer.clone()` 得到**本次调用专属**的状态实例（§5 源码可见），`attempt`/`sleptForMillis` 的读写因此不需要锁。自定义 Retryer 必须正确实现 clone——返回 `this` 会让并发调用互相污染计数。
3. **Retry-After 优先于退避**：服务端给了明确的恢复时间提示时尊重它（并截到 maxPeriod），这是 429/503 治理里"调用方配合服务端削峰"的接口（构造带 retryAfter 的异常见 §6）。另外 `Thread.sleep` 在调用线程上执行——重试等待占用业务线程池额度，高并发下 maxAttempts 与 maxPeriod 的组合可能把池子睡穿（与 `11_timeout-connection.md`、`26_performance-tuning.md` 关联）。

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

yml 按无参构造反射创建，`MyCustomRetryer` 必须提供 public 无参构造；需要注入配置（间隔来自配置中心之类）就只能走 §3.1 的 @Bean 路线（两入口的覆盖关系见 `4_configuration.md` §6）。

## 3.3 禁用重试

```java
@Bean
public Retryer feignRetryer() {
    return Retryer.NEVER_RETRY;  // 默认值
}
```

需要"全局默认关、个别客户端开"时，用 `default: NEVER_RETRY` + 专属配置类注册 `Retryer.Default` 的组合，比"全局开、个别关"安全：后者总有一个新客户端忘关就带上重试上线。

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

注意 `e.status() >= 500` 的分支对 status=-1（网络失败）永远不成立——这个实现里超时/断连反而不重试，若期望"网络抖动重试、确定性 5xx 不重试"，条件要显式写成 `(e.status() == -1 || e.status() == 503)`。基于状态码过滤的前提是状态码能进 Retryer：默认 ErrorDecoder 不产出 RetryableException（§5），所以此类策略必须与 §6 的 ErrorDecoder 配套，或者干脆退一步——只对 IOException 家族（status=-1）重试，不需要任何翻译层配合。

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

表里"是"的三行有一个共同来源：`Client.execute` 抛出的 `IOException` 会被 `executeAndDecode` 捕获并包装成 `RetryableException`（携带原异常与重试时刻），ErrorDecoder 产出的是第二类。由此也能推出一个反直觉结论：**DNS 失败、连接池耗尽等"没到服务端"的失败都可重试，而服务端明确回了 500 的失败默认一次都不重试**——默认策略的设计意图是让"可安全重放的失败"有容错、让"确定性业务失败"快速失败。

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

两个协同细节：`RetryableException` 的最后一个参数是触发它的 `Request`，排障时靠它还原"哪次尝试的哪个报文"；`retryAfter` 传的是**绝对时间点**（Date），示例的换算写法正确，但 `Long.parseLong` 裸解析遇到 HTTP 日期格式（`Retry-After: Wed, 21 Oct 2025 ...`）会 NumberFormatException——健壮实现要么双格式解析，要么解析失败退回默认间隔。更重要的协同警告：这套 Feign 级重试与 Resilience4j 的 Retry 装饰器**会叠加放大**（总请求次数 = feignAttempts × r4jAttempts），两边同时开启是最常见的重试风暴根因，二选一并在文档里注明（`14_circuit-breaker.md`）。

# 7. 与 Spring Retry 对比

| 维度 | Feign Retryer | Spring Retry |
|------|--------------|--------------|
| **触发条件** | `RetryableException` 或 I/O 异常 | 可配置异常类型白名单 |
| **退避策略** | 指数退避（内置） | Fixed / Exponential Random / Custom |
| **状态性** | 简单计数 | 重试上下文、Recovery 回调 |
| **熔断集成** | 无 | 可配合 Circuit Breaker |

**建议**：重试需求简单时用 Feign `Retryer`；需要更复杂的重试策略（如按异常类型、重试恢复回调），用 Spring Retry + Feign Builder 手动构建。

对比表外还有第三个主流选项——Resilience4j Retry：它工作在 Feign 调用外层（装饰器/切面），能拿到**最终结果**而非单次尝试异常，支持按异常类型/结果谓词决策、监听器可观测，且与熔断同族配置统一。三种机制的层次记忆：Retryer = 客户端内部每次尝试的容错；R4j Retry = 面向结果的调用策略；Spring Retry = 任意方法级的 AOP 重试。它们两两都可能叠加，**同一个链路里只选一层做重试**。

# 8. 重试对幂等性的影响

```text
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

## 8.1 读超时重试的真实危险：第一次可能已经成功

"连接还没建立就失败"确实可安全重放；但**读超时**意味着请求很可能已经到达服务端并执行完毕，只是响应没有及时回来。对不幂等的 POST，重试会创建两笔订单——这比"根本没发出去"糟糕得多，且生产事故里读超时恰恰是网络抖动的最常见形态。因此"按方法排除 POST"是底线而非高线：**对写操作，重试必须配服务端幂等键**（请求头带业务唯一 ID，提供方查重返回首次结果，`25_best-practices.md` 有展开）。PUT 的"幂等取决于实现"也危险：`update(user)` 全量覆盖是幂等的，`increment(balance)` 这类语义化接口哪怕挂在 PUT 上也绝不幂等——判断依据是接口语义，不是 HTTP 动词。

# 9. 配置建议

| 场景 | period | maxPeriod | maxAttempts |
|------|--------|-----------|-------------|
| **快速失败（默认）** | — | — | NEVER_RETRY |
| **网络波动容忍** | 500 | 5000 | 3 |
| **临时故障等恢复** | 1000 | 10000 | 5 |
| **极力保证成功** | 500 | 30000 | 10 |

## 9.1 先算时间预算，再谈重试参数

重试参数的上限由**调用链总预算**倒推，不是独立决策：`总耗时 ≈ maxAttempts × read-timeout + Σ退避间隔`。以"网络波动容忍"行为例：3 × 5s 读超时 + (0.5+1) 退避 ≈ 16.5s——如果上游给本服务的超时只有 10s，重试到第二次就会以上游超时告终，反而把故障放大给更上层。工程约定：**本层总预算 ≤ 调用方预算的 70%**，并让 connect-timeout 短（建立失败快速重试有意义）、read-timeout 与 maxPeriod 相互制衡；"极力保证成功"档基本只适用于离线任务线程，同步 Web 线程上睡 30s 等价的是一次小型故障演练。
