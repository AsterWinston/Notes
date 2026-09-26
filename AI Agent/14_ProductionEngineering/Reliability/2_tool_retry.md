# 1. 重试的前提：幂等

结论先行：重试（Retry）是应对临时抖动的常用手段，但它成立的前提是操作幂等（Idempotent）；对非幂等的写操作重试，等于制造重复副作用。

```
重试必须建立在幂等之上。
非幂等的写操作重试 = 重复下单、重复扣款、重复发消息。
```

> 只读操作可放心重试；写操作必须先保证幂等（见 `3_idempotency.md`）。

## 1.1 为什么重试必须幂等

结论先行：重试的本质是"把同一个请求再发一次"，如果请求本身带副作用，多发一次就多一次副作用。

机制拆解：

1. 请求发出后，服务端可能已执行成功，只是响应在返回途中丢失。
2. 调用方看到超时，认为失败，发起重试。
3. 服务端第二次收到相同请求，若无幂等保护，会再次执行。
4. 结果：重复下单、重复扣款。

```
客户端                服务端
  │ ──请求1──▶       │ 执行成功
  │   ✗ 响应丢失      │
  │ ──请求2──▶       │ 再次执行 ← 副作用翻倍
```

## 1.2 只读与写操作的重试差异

结论先行：只读操作天然幂等，可放心重试；写操作必须先实现幂等再重试。

| 操作类型 | 天然幂等 | 可否直接重试 |
|---------|:--------:|:-----------:|
| 查询（GET） | 是 | 是 |
| 覆盖更新 | 是 | 是 |
| 插入 | 否 | 否（需幂等键） |
| 累加 | 否 | 否（需幂等键） |
| 转账 | 否 | 否（需幂等键） |

## 1.3 重试与其他可靠性机制的关系

结论先行：重试不是万能药，它必须与超时、熔断、限流、幂等配合，否则会放大故障。

| 机制 | 与重试的关系 |
|------|-------------|
| 超时 | 超时是重试的触发源之一 |
| 幂等 | 重试成立的前提 |
| 熔断 | 熔断打开时禁止重试 |
| 限流 | 限制重试带来的额外流量 |

# 2. 什么错误该重试

结论先行：只有"临时性、可恢复"的错误才该重试；对确定性错误重试只会浪费资源并放大问题。

## 2.1 错误分类

### 2.1.1 可重试错误

结论先行：网络抖动、服务端临时故障、限流属于可重试错误。

| 错误类型 | 例子 |
|----------|------|
| 网络超时 | 连接超时、读超时 |
| 连接重置 | Connection reset |
| 服务端 5xx | 500、502、503、504 |
| 限流 429 | Too Many Requests |

### 2.1.2 不可重试错误

结论先行：客户端错误与业务错误属于确定性失败，重试不会改变结果。

| 错误类型 | 例子 |
|----------|------|
| 客户端 4xx | 400、401、403、404 |
| 参数错误 | 校验失败 |
| 业务失败 | 余额不足 |

## 2.2 判断实现

结论先行：用一个集中式的可重试判断函数，避免散落各处的魔法判断。

| 错误类型 | 例子 | 是否重试 |
|----------|------|:--------:|
| 网络超时 | 连接超时、读超时 | 是 |
| 连接重置 | Connection reset | 是 |
| 服务端 5xx | 500、502、503、504 | 是 |
| 限流 429 | Too Many Requests | 是（退避） |
| 客户端 4xx | 400、401、403、404 | 否 |
| 参数错误 | 校验失败 | 否 |
| 业务失败 | 余额不足 | 否 |

```java
import java.net.ConnectException;
import java.net.SocketTimeoutException;
import org.springframework.web.client.HttpStatusCodeException;

public boolean isRetryable(Exception e) {
    if (e instanceof SocketTimeoutException) return true;
    if (e instanceof ConnectException) return true;

    if (e instanceof HttpStatusCodeException http) {
        int code = http.getStatusCode().value();
        return code == 429 || code >= 500;
    }
    return false;
}
```

> 重试不该重试的错误，只会放大问题（如重试 401 会触发风控）。

## 2.3 边界与反例

结论先行：有些错误表面可重试，实际重试会造成更严重后果。

| 反例 | 问题 |
|------|------|
| 重试 401 | 触发账号风控、锁定 |
| 重试 400 | 参数永远错误，纯浪费 |
| 重试非幂等写 | 重复副作用 |
| 重试超长任务 | 叠加占用资源 |

生产建议：可重试判断应可配置，并按工具维度覆盖默认值。

# 3. 退避策略

结论先行：退避（Backoff）决定两次重试之间的等待时间；固定退避易形成重试风暴，指数退避加抖动是最佳实践。

```
固定退避：     1s, 1s, 1s, 1s           ← 可能形成重试风暴
指数退避：     1s, 2s, 4s, 8s           ← 常用
指数 + 抖动：  1.1s, 1.9s, 4.2s, 7.8s   ← 最佳，避免同步重试
```

## 3.1 三种退避对比

结论先行：固定退避简单但危险，指数退避缓解压力，指数加抖动在缓解压力同时打散同步。

| 策略 | 优点 | 缺点 |
|------|------|------|
| 固定 | 简单 | 易形成风暴 |
| 指数 | 缓解压力 | 尾部等待长 |
| 指数+抖动 | 最佳 | 实现稍复杂 |

## 3.2 抖动的重要性

### 3.2.1 重试风暴的形成

结论先行：当大量客户端在同一时刻失败，若退避时间相同，它们会再次同时重试，形成周期性冲击。

```
无抖动（同步重试）
客户端 A ──fail──1s──▶ 重试
客户端 B ──fail──1s──▶ 重试   ← 同一时刻再次冲击
客户端 C ──fail──1s──▶ 重试

有抖动（打散）
客户端 A ──fail──1.1s──▶ 重试
客户端 B ──fail──1.9s────▶ 重试
客户端 C ──fail──0.8s─▶ 重试
```

### 3.2.2 抖动实现

结论先行：在指数退避的基础上，乘以一个随机因子，把重试时间打散。

```java
import java.time.Duration;

public Duration backoff(int attempt, Duration base, Duration max) {
    // 指数退避
    long exp = base.toMillis() * (1L << attempt);
    long capped = Math.min(exp, max.toMillis());

    // 抖动：±20%
    double jitter = 0.8 + Math.random() * 0.4;
    return Duration.ofMillis((long) (capped * jitter));
}
```

## 3.3 退避参数选择

结论先行：初始等待、倍数、上限三个参数共同决定退避曲线，需与超时预算匹配。

| 参数 | 建议值 | 说明 |
|------|-------|------|
| 初始等待 | 200~500ms | 快速重试首次 |
| 倍数 | 2 | 标准指数 |
| 抖动 | 0.2~0.5 | 打散同步 |
| 单次上限 | 5~10s | 避免尾部过长 |

生产建议：退避总时长不得超过上游超时预算。

# 4. 重试实现

结论先行：可用手写、Spring Retry 或 Resilience4j 实现重试，生产环境推荐使用成熟库以统一治理。

## 4.1 手写重试

结论先行：手写重试便于理解机制，但需自己处理分类、退避、次数与异常。

```java
import java.time.Duration;
import java.util.function.Supplier;

public <T> T retry(int maxAttempts, Supplier<T> action) {
    RuntimeException last = null;

    for (int attempt = 0; attempt < maxAttempts; attempt++) {
        try {
            return action.get();
        } catch (Exception e) {
            last = new RuntimeException(e);

            if (!isRetryable(e) || attempt == maxAttempts - 1) {
                throw last;
            }
            sleep(backoff(attempt, Duration.ofMillis(500),
                    Duration.ofSeconds(5)));
        }
    }
    throw last;
}
```

## 4.2 Spring Retry

结论先行：用注解声明重试与恢复，代码侵入小，适合 Spring 项目。

```java
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import java.net.SocketTimeoutException;
import org.springframework.web.client.HttpServerErrorException;

@Retryable(
        retryFor = {SocketTimeoutException.class, HttpServerErrorException.class},
        noRetryFor = {IllegalArgumentException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 500, multiplier = 2, random = true))
public String callTool(String args) {
    return tool.execute(args);
}

@Recover
public String recover(Exception e, String args) {
    return "工具暂时不可用，请稍后重试";
}
```

## 4.3 Resilience4j Retry

结论先行：Resilience4j 的重试可与熔断、限流、超时统一组合，适合复杂治理场景。

```java
import io.github.resilience4j.core.IntervalFunction;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import java.net.SocketTimeoutException;
import java.time.Duration;
import org.springframework.web.client.HttpServerErrorException;

Retry retry = Retry.of("tool", RetryConfig.custom()
        .maxAttempts(3)
        .waitDuration(Duration.ofMillis(500))
        .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                500, 2.0, 0.2))                 // 指数 + 抖动
        .retryExceptions(SocketTimeoutException.class,
                HttpServerErrorException.class)
        .ignoreExceptions(IllegalArgumentException.class)
        .build());

public String callTool(String args) {
    return Retry.decorateSupplier(retry, () -> tool.execute(args)).get();
}
```

## 4.4 三种实现对比

结论先行：按团队规模与治理需求选择，小项目手写即可，中大型用库。

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 手写 | 无依赖、透明 | 易遗漏 | 简单场景 |
| Spring Retry | 注解声明、易用 | 与 WebFlux 配合弱 | Spring MVC |
| Resilience4j | 组合能力强 | 学习成本高 | 复杂治理 |

# 5. 重试与幂等键

结论先行：写操作重试必须携带同一个幂等键（Idempotency Key），由 Agent 生成一次并在重试时复用。

## 5.1 幂等键的复用

结论先行：幂等键标识"一次业务意图"，重试属于同一次意图，因此键必须保持不变。

```
第一次尝试 ──key=abc──▶ 服务端（可能已成功）
重试       ──key=abc──▶ 服务端（识别为同一意图，返回已有结果）
```

## 5.2 工具层实现

结论先行：在工具签名中显式暴露幂等键参数，并说明重试时保持不变。

```java
@Tool("创建订单")
ToolResult createOrder(
        @P("幂等键，重试时必须相同") String idempotencyKey,
        @P("商品编码") String sku) {

    // 服务端：同一 key 只创建一次
    return orderService.createIdempotent(idempotencyKey, sku);
}
```

> 幂等键由 Agent 生成一次，重试时复用，不要每次重试都生成新的。

## 5.3 常见错误

结论先行：最常见的错误是"每次重试生成新键"，导致幂等保护形同虚设。

| 错误 | 后果 |
|------|------|
| 每次重试新键 | 幂等失效，重复副作用 |
| 键不含用户 | 跨用户冲突 |
| 键可预测且不唯一 | 误合并不同请求 |

# 6. 重试与熔断配合

结论先行：重试处理临时抖动，熔断处理持续故障；两者必须按正确顺序组合，否则重试会放大故障。

```
重试：处理"临时抖动"
熔断：处理"持续故障"
```

## 6.1 职责区分

结论先行：先分清两类故障，才能决定由谁处理。

| 机制 | 触发条件 | 处理对象 |
|------|---------|---------|
| 重试 | 单次失败 | 临时抖动 |
| 熔断 | 失败率超阈 | 持续故障 |

## 6.2 组合顺序

结论先行：熔断在最外层、重试在内层，使持续故障被快速失败拦截，不再重试。

```
请求 → 熔断器 → 限流器 → 重试 → 实际调用
```

```java
import io.github.resilience4j.decorators.Decorators;
import java.util.function.Supplier;

Supplier<String> decorated = Decorators.ofSupplier(() -> tool.execute(args))
        .withCircuitBreaker(circuitBreaker)     // 最外层：持续故障快速失败
        .withRetry(retry)                        // 内层：临时失败重试
        .withTimeLimiter(timeLimiter)
        .decorate();
```

## 6.3 熔断打开时的行为

结论先行：熔断打开时不应重试，否则重试也会立即失败，白白浪费资源。

> 熔断打开时不应重试，否则重试也会立即失败，浪费资源。

| 熔断状态 | 重试行为 |
|---------|---------|
| CLOSED | 正常重试 |
| OPEN | 立即失败，不重试 |
| HALF_OPEN | 放行少量探测请求 |

# 7. 重试风暴的防护

结论先行：重试会放大流量，必须用多道防线把重试总量限制在可控范围。

## 7.1 五道防线

结论先行：次数上限、退避抖动、熔断、限流、重试预算共同构成防护网。

```java
// ① 限制重试次数（2~3 次）
// ② 指数退避 + 抖动
// ③ 熔断：失败率高时快速失败
// ④ 限流：限制并发
// ⑤ 重试预算：整个请求的重试总次数上限
```

| 防线 | 作用 |
|------|------|
| 次数上限 | 单次调用不无限重试 |
| 退避抖动 | 打散同步重试 |
| 熔断 | 持续故障快速失败 |
| 限流 | 限制并发总量 |
| 重试预算 | 限制请求级总重试 |

## 7.2 重试预算

结论先行：重试预算（Retry Budget）限制一个请求在整条链路上的重试总次数，防止重试层层放大。

```java
import java.util.concurrent.atomic.AtomicInteger;

public class RetryBudget {
    private final AtomicInteger remaining;

    public RetryBudget(int total) {
        this.remaining = new AtomicInteger(total);
    }

    public boolean tryConsume() {
        return remaining.getAndUpdate(v -> Math.max(0, v - 1)) > 0;
    }
}
```

## 7.3 自适应限流

结论先行：当失败率升高时主动降低重试比例，让系统有恢复空间。

| 失败率 | 重试策略 |
|--------|---------|
| < 1% | 正常重试 |
| 1%~10% | 降低重试比例 |
| > 10% | 暂停重试，等待恢复 |

# 8. 工具返回错误 vs 重试

结论先行：对 Agent 工具，重试后无论成败，都应返回结构化结果，让模型能感知并调整策略。

## 8.1 三种处理方式

结论先行：按错误性质选择内部重试、结构化错误或直接返回。

| 处理 | 场景 |
|------|------|
| 内部重试后返回成功 | 临时抖动 |
| 重试失败后返回结构化错误 | 让模型换策略 |
| 直接返回错误 | 不可重试错误 |

## 8.2 结构化错误

结论先行：把失败信息包装成模型可读的结构，而不是抛出异常。

```java
@Tool("查询物流")
ToolResult queryLogistics(@P("运单号") String waybill) {
    try {
        String result = retry(3, () -> logisticsApi.query(waybill));
        return ToolResult.ok(result);
    } catch (Exception e) {
        return ToolResult.fail("物流查询失败：" + e.getMessage());
    }
}
```

## 8.3 与提示词的配合

结论先行：在系统提示中说明重试语义与可重试边界，减少模型重复调用。

- 告知模型"相同参数的工具不要重复调用"。
- 返回结构化错误时附上建议动作。
- 对不可重试错误明确标注"请勿重试"。

# 9. 监控

结论先行：重试的收益与代价都需量化，否则无法判断重试是否有效。

## 9.1 关键指标

结论先行：至少监控重试次数、重试成功率与重试后的最终失败率。

| 指标 | 含义 |
|------|------|
| `tool.retry.count` | 重试次数 |
| `tool.retry.success` | 重试成功次数 |
| `tool.retry.rate` | 重试率 |
| `tool.failure.final` | 重试后仍失败 |

## 9.2 告警

结论先行：重试率突增通常意味着下游恶化，应尽快告警。

| 指标 | 告警条件 |
|------|---------|
| 重试率 | > 5% |
| 重试后失败率 | > 1% |
| 重试次数/请求 | 突增 |

# 10. 生产注意事项

结论先行：重试相关的生产事故多源于"重试了不该重试的东西"或"重试没有上限"，需按清单检查。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 重试非幂等写 | 先做幂等 |
| 重试不可重试错误 | 分类判断 |
| 固定间隔 | 指数 + 抖动 |
| 无次数上限 | 2~3 次 |
| 熔断打开仍重试 | 组合顺序正确 |
| 无监控 | 监控重试率与成功率 |
| 重试放大故障 | 熔断 + 限流 + 重试预算 |

## 10.2 检查清单

结论先行：上线前逐项确认，可避免绝大多数重试事故。

- 写操作是否已实现幂等。
- 重试是否携带同一幂等键。
- 是否区分可重试与不可重试错误。
- 是否使用指数退避加抖动。
- 是否设置次数上限与重试预算。
- 熔断与重试的顺序是否正确。
- 是否采集重试相关指标与告警。

# 11. 小结

## 11.1 要点回顾

- 重试的前提是幂等，写操作必须带幂等键。
- 只重试可重试错误（超时、5xx、429）。
- 退避用"指数 + 抖动"，避免重试风暴。

## 11.2 落地行动

- 熔断在外、重试在内；熔断打开时不重试。
- 重试要有次数与预算上限，并监控重试率。
