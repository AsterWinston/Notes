# 1. 可靠性五件套总览

## 1.1 六道防线

结论先行：LLM 调用的可靠性不是靠"重试一下"，而是由超时、重试、幂等、限流、熔断、降级六道防线协同组成，缺一不可。

机制：请求依次经过"限流保护系统、超时兜住挂起、重试吸收瞬时故障、熔断阻断持续故障、降级提供兜底、幂等保证重试安全"。

### 1.1.1 各防线职责

```
超时 → 重试 → 幂等 → 限流 → 熔断 → 降级
  │      │      │      │      │      │
  │      │      │      │      │      └─ 兜底可用
  │      │      │      │      └─ 持续故障快速失败
  │      │      │      └─ 保护下游与配额
  │      │      └─ 保证重试不产生副作用
  │      └─ 吸收瞬时抖动
  └─ 避免线程无限挂起
```

六道防线各司其职：限流在入口保护系统，熔断在下游持续故障时快速失败，重试吸收瞬时抖动，超时兜住挂起，幂等保证重试安全，降级提供最终兜底。

### 1.1.2 调用链

```
用户请求
   │
   ▼
[限流] ──超限──► 拒绝
   │ 通过
   ▼
[熔断] ──打开──► 直接降级
   │ 关闭
   ▼
[重试] ──成功──► 返回
   │ 耗尽
   ▼
[降级链] ──► 备用/缓存/静态兜底
```

## 1.2 失败分类

### 1.2.1 可重试与不可重试

| 可重试 | 不可重试 |
|--------|----------|
| 超时 | 400 参数错误 |
| 429 限流 | 401 认证失败 |
| 5xx | 403 无权限 |
| 连接重置 | 404 不存在 |

结论：重试的前提是"错误可能自愈且操作幂等"，否则重试只会放大问题。

### 1.2.2 失败来源

| 来源 | 典型表现 | 应对 |
|------|----------|------|
| 网络 | 连接超时、重置 | 重试 |
| 厂商限流 | 429 | 退避重试 |
| 厂商故障 | 5xx、宕机 | 熔断 + 降级 |
| 输入错误 | 400 | 不重试，修正输入 |
| 内容安全 | 拦截 | 降级或提示 |
| 本地资源 | 线程池满 | 限流 |

> 详细机制见 `14_ProductionEngineering/Reliability`，本文聚焦 LLM 工程视角。

# 2. 超时

## 2.1 超时分层

### 2.1.1 各层超时

结论先行：超时必须分层设置，且"内层超时之和"要小于外层总超时，否则外层会先被触发。

| 层级 | 建议 |
|------|------|
| 连接 | 1~3s |
| 模型调用 | 30~60s |
| 工具调用 | 3~10s |
| 总请求 | 60~120s |

机制：连接超时控制建连，读超时控制等待响应，总超时用截止时间贯穿全链路。

```
总超时预算 ≥ Σ(每次调用超时 × 最大重试次数) + 退避等待之和
```

若不等式不成立，外层会在重试完成前先行超时，重试投入全部白费。

### 2.1.2 配置示例

```java
ChatModel model = OpenAiChatModel.builder()
        .apiKey(key)
        .modelName("gpt-4o-mini")
        .timeout(Duration.ofSeconds(30))         // 请求超时
        .maxRetries(2)                            // 内置重试
        .build();
```

结论：厂商 SDK 的内置重试与框架层重试容易叠加，需显式约定由哪一层负责重试。

## 2.2 超时边界

### 2.2.1 流式场景

结论：流式响应不能只设"首个 token 超时"，还要设"token 间隔超时"和"总时长上限"。

生产建议：对流式接口设置首包超时（如 10s）、空闲超时（如 30s）与总时长（如 180s）。

### 2.2.2 常见坑

- 只设连接超时，不设读超时，线程仍会长时间挂起。
- 外层网关超时短于内层重试总耗时，用户先看到 504。
- 超时后未取消底层请求，仍占用配额。

生产建议：用统一的 `deadline` 贯穿调用链，任一环节发现已超时立即中止，避免"僵尸请求"。

## 2.3 超时的可取消性

### 2.3.1 取消传播

结论：超时触发后必须向底层传播取消信号，否则被放弃的请求仍在消耗厂商配额。

```java
// 用 CompletableFuture 的取消能力贯穿超时
public String chatWithDeadline(String prompt, Duration deadline) {
    CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> model.chat(prompt), executor);
    try {
        return future.get(deadline.toMillis(), TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        future.cancel(true);                 // 传播取消
        throw new LlmTimeoutException("调用超时", e);
    } catch (Exception e) {
        throw new LlmException("调用失败", e);
    }
}
```

### 2.3.2 边界

反例：取消信号发出后，部分 HTTP 客户端并不会真正断开连接，需要确认客户端支持取消。

# 3. 重试

## 3.1 重试实现

### 3.1.1 Spring Retry

结论先行：重试要用声明式配置，明确"哪些异常重试、重试几次、如何退避"。

```java
@Retryable(
        retryFor = {SocketTimeoutException.class, HttpServerErrorException.class},
        noRetryFor = {IllegalArgumentException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 500, multiplier = 2, random = true))
public String chat(String prompt) {
    return chatClient.prompt().user(prompt).call().content();
}

@Recover
public String recover(Exception e, String prompt) {
    log.warn("重试耗尽，降级", e);
    return fallback(prompt);
}
```

### 3.1.2 幂等前提

结论：只有幂等操作才能安全重试。LLM 调用通常无副作用，但"工具调用"可能有。

生产建议：把有副作用的工具调用（下单、发邮件）纳入幂等键管理，重试前先查是否已执行。

## 3.2 只重试可重试错误

### 3.2.1 错误分类表

| 错误类型 | 是否重试 | 原因 |
|----------|:--------:|------|
| 超时 | 是 | 可能瞬时抖动 |
| 429 | 是 | 退避后可能恢复 |
| 5xx | 是 | 服务端临时故障 |
| 400 | 否 | 输入问题，重试无用 |
| 401/403 | 否 | 凭证问题 |
| 内容安全拦截 | 否 | 重试结果相同 |

### 3.2.2 反例

反例：对 401 认证失败重试三次，只会浪费三次配额并延迟错误暴露；对参数错误的 400 重试，结果完全一致。

生产建议：维护一份异常分类映射，新增异常时先归类再决定是否重试。

# 4. 退避策略

## 4.1 退避算法

### 4.1.1 指数退避

结论先行：退避必须"指数增长 + 随机抖动"，否则重试会形成同步风暴。

```java
public Duration backoff(int attempt) {
    long exp = 500L * (1L << attempt);              // 指数
    long capped = Math.min(exp, 5000);              // 上限
    double jitter = 0.8 + Math.random() * 0.4;      // ±20% 抖动
    return Duration.ofMillis((long) (capped * jitter));
}
```

### 4.1.2 抖动类型

| 抖动方式 | 说明 |
|----------|------|
| 全抖动 | 随机范围 [0, cap] |
| 等抖动 | [cap/2, cap] |
| 比例抖动 | cap × (1 ± r) |

结论：全抖动防同步效果最好，但平均等待更短；比例抖动更易预测。

下表为 `delay=500ms`、上限 5s 的退避序列示例：

| 第几次 | 理论间隔 | 抖动后范围 |
|:------:|:--------:|:----------:|
| 1 | 500ms | 400~600ms |
| 2 | 1000ms | 800~1200ms |
| 3 | 2000ms | 1600~2400ms |
| 4 | 4000ms | 3200~4800ms |
| 5 | 5000ms（封顶） | 4000~6000ms |

## 4.2 退避边界

### 4.2.1 上限与预算

结论：退避必须有上限，且"总重试耗时"不得超过外层超时预算。

> 指数 + 抖动，避免重试风暴。

### 4.2.2 重试风暴

反例：所有实例在 429 后固定等待 1 秒重试，会在同一时刻再次打满限流，形成周期性雪崩。

生产建议：设置最大退避（如 5s），并加入全局并发重试上限。

# 5. 降级链

## 5.1 降级层级

### 5.1.1 五级降级

结论先行：降级不是单一兜底，而是"主模型到静态文案"的多级链路，越靠前质量越高。

```
① 主模型
   ↓ 失败
② 备用模型（其他厂商/小模型）
   ↓ 失败
③ 缓存结果
   ↓ 无缓存
④ 简化流程（不用 RAG/工具）
   ↓ 失败
⑤ 静态兜底（"服务繁忙，请稍后重试"）
```

### 5.1.2 代码实现

```java
public String answerWithFallback(String question, UserContext user) {
    // ① 主模型
    try {
        return primaryChat(question, user);
    } catch (Exception e) {
        log.warn("主模型失败，降级", e);
    }
    // ② 备用模型
    try {
        return fallbackChat(question, user);
    } catch (Exception e) {
        log.warn("备用模型失败，降级到缓存", e);
    }
    // ③ 缓存
    String cached = cache.get(cacheKey(question, user));
    if (cached != null) return cached + "\n（缓存结果）";

    // ④ 简化流程省略
    // ⑤ 静态兜底
    return "服务当前繁忙，请稍后重试。";
}
```

结论：每一级都要记录降级原因，否则无法定位质量下降的根因。

| 级别 | 质量 | 成本 | 适用 |
|------|:----:|:----:|------|
| 主模型 | 高 | 高 | 默认 |
| 备用模型 | 中 | 中 | 主模型故障 |
| 缓存 | 中 | 低 | 可缓存问题 |
| 简化流程 | 低 | 低 | 非强依赖 RAG |
| 静态兜底 | 无 | 无 | 全部失败 |

## 5.2 降级内容设计

### 5.2.1 缓存降级

结论：缓存降级要明确标注"这是缓存结果"，避免用户误以为是最新答案。

### 5.2.2 简化流程

结论：简化流程指去掉 RAG 与工具调用，用纯模型回答，质量下降但可用性提升。

生产建议：为每条降级路径定义质量预期与适用场景，避免"降级后胡说"。

# 6. 熔断

## 6.1 熔断器

### 6.1.1 状态机

结论先行：熔断器在"持续故障"时快速失败，避免把资源浪费在必然失败的调用上。

```
关闭 ──失败率超阈值──► 打开 ──等待窗口──► 半开
  ▲                                        │
  └──────── 探测成功 ◄─────────────────────┘
```

### 6.1.2 配置

```java
CircuitBreaker breaker = CircuitBreaker.of("llm",
        CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .slidingWindowSize(20)
                .minimumNumberOfCalls(10)
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .build());

public String chat(String prompt) {
    return CircuitBreaker.decorateSupplier(breaker,
            () -> model.chat(prompt)).get();
}
```

结论：阈值、窗口、半开探测次数需按实际 QPS 调优。

## 6.2 熔断与重试

### 6.2.1 打开时不重试

> 熔断打开时**不应重试**，直接降级。

机制：熔断打开意味着已知下游不可用，重试只会增加无效负载。

### 6.2.2 半开探测

结论：半开状态只放行少量请求探测，成功则关闭，失败则继续打开。

生产建议：半开探测请求与正常请求隔离，避免探测失败影响真实流量。

## 6.3 熔断指标

### 6.3.1 状态指标

结论：熔断器的状态变化必须暴露为指标，否则无法区分"正常"与"被熔断"。

```java
registry.gauge("llm.circuit.state", breaker,
        b -> b.getState() == CircuitBreaker.State.CLOSED ? 0
         : b.getState() == CircuitBreaker.State.HALF_OPEN ? 1 : 2);
```

### 6.3.2 告警

结论：熔断打开应立即告警，因为它通常意味着下游厂商或网络出现较严重问题。

# 7. 限流与幂等

## 7.1 限流

### 7.1.1 限流维度

| 维度 | 目的 |
|------|------|
| 全局 QPS | 保护下游 |
| 租户 QPS | 防单租户挤占 |
| 并发数 | 保护线程池 |
| Token 速率 | 控制成本 |

### 7.1.2 实现

```java
RateLimiter limiter = RateLimiter.of("llm", RateLimiterConfig.custom()
        .limitForPeriod(100)
        .limitRefreshPeriod(Duration.ofSeconds(1))
        .timeoutDuration(Duration.ofMillis(0))
        .build());

Supplier<String> guarded = RateLimiter.decorateSupplier(limiter,
        () -> model.chat(prompt));
```

结论：限流被拒后应快速失败或排队，不应立即重试。

生产建议：限流拒绝应返回明确的"稍后重试"信号，并在客户端做全抖动退避，而非立即重放。

## 7.2 幂等

### 7.2.1 幂等键

结论：重试安全依赖幂等，幂等依赖唯一业务键。

```java
public String createOrder(String idempotencyKey, OrderRequest req) {
    String existing = redis.opsForValue().get("idem:" + idempotencyKey);
    if (existing != null) return existing;          // 已执行，直接返回
    String result = doCreateOrder(req);
    redis.opsForValue().set("idem:" + idempotencyKey, result,
            Duration.ofHours(24));
    return result;
}
```

### 7.2.2 副作用防护

结论：凡有副作用的工具调用，必须先做幂等判断再执行，且幂等键由调用方生成。

```java
// 工具调用侧：用幂等键包裹有副作用的操作
public ToolResult invokeTool(String toolName, JsonNode args,
                             String idempotencyKey) {
    String key = "tool:%s:%s".formatted(toolName, idempotencyKey);
    String done = redis.opsForValue().get(key);
    if (done != null) return ToolResult.fromJson(done);
    ToolResult result = toolExecutor.execute(toolName, args);
    redis.opsForValue().set(key, result.toJson(), Duration.ofHours(24));
    return result;
}
```

# 8. 组合顺序

## 8.1 装饰器顺序

### 8.1.1 正确顺序

结论先行：装饰器顺序决定语义，"快速失败在外、重试在内"是唯一正确姿势。

```java
Supplier<String> decorated = Decorators.ofSupplier(() -> model.chat(prompt))
        .withCircuitBreaker(breaker)      // 外：持续故障快速失败
        .withRateLimiter(rateLimiter)     // 限流
        .withRetry(retry)                 // 内：临时失败重试
        .withTimeLimiter(timeLimiter)
        .decorate();
```

```
熔断（外）→ 限流 → 重试（内）→ 实际调用
```

### 8.1.2 错误顺序

反例：把重试放在熔断外层，会导致熔断打开后仍不断重试，熔断形同虚设。

## 8.2 组合原则

### 8.2.1 快速失败在外

结论：越"便宜"、越"能快速否决"的防护越应放外层，减少无效内层调用。

### 8.2.2 重试在内

结论：重试只应包裹"实际调用"，不应包裹熔断或限流，否则会绕过它们。

生产建议：用统一工厂方法封装组合，避免各业务线顺序不一致。

# 9. 降级的可见性与高风险场景

## 9.1 可观测

### 9.1.1 指标与日志

结论先行：降级必须可观测，否则"静默降级"会让质量问题长期潜伏。

```java
registry.counter("ai.degradation",
        "level", level,            // MODEL / CACHE / STATIC
        "reason", reason).increment();
log.warn("触发降级 level={} reason={} traceId={}",
        level, reason, MDC.get("traceId"));
```

### 9.1.2 用户提示

结论：降级应对用户适度提示，尤其当答案是缓存或简化结果时。

| 降级级别 | 用户提示建议 |
|----------|--------------|
| 备用模型 | 无需提示，用户无感 |
| 缓存结果 | 标注"以下为缓存结果" |
| 简化流程 | 提示"当前为简版回答" |
| 静态兜底 | 明确"服务繁忙，请稍后重试" |

> 降级必须可观测，且对用户适度提示。

## 9.2 高风险场景

### 9.2.1 宁可失败

结论：医疗、金融等高风险场景，"降级 = 明确失败 + 转人工"，绝不能用缓存或小模型冒充。

```java
// 高风险场景：宁可失败，不降级
public String financialAdvice(String question) {
    if (primaryUnavailable()) {
        throw new ServiceUnavailableException(
                "金融建议服务暂不可用，请稍后重试");
    }
    return primaryChat(question);
}
```

### 9.2.2 转人工

生产建议：高风险场景提供"转人工"通道，并记录未答复请求以便回访。

# 10. 生产注意事项

## 10.1 风险清单

### 10.1.1 风险表

| 风险 | 建议 |
|------|------|
| 无超时 | 必设 |
| 重试不可重试错误 | 分类判断 |
| 固定间隔重试 | 指数 + 抖动 |
| 熔断打开仍重试 | 组合顺序正确 |
| 无降级 | 必须有兜底 |
| 降级静默 | 指标 + 日志 |
| 高风险降级 | 明确失败 |

### 10.1.2 反模式

- 无限重试：耗尽配额并拖垮线程池。
- 只重试不降级：重试耗尽后直接抛错给用户。
- 降级无提示：用户拿到过期缓存却不知情。

## 10.2 生产建议

### 10.2.1 检查清单

- 每层是否都有超时？
- 重试是否只针对可重试错误？
- 退避是否指数加抖动？
- 熔断与重试顺序是否正确？
- 降级是否可观测？

### 10.2.2 压测与演练

结论：定期做故障演练（注入超时、429、5xx），验证降级链是否按预期生效。

演练清单：

| 故障注入 | 期望行为 |
|----------|----------|
| 主模型超时 | 重试后转备用模型 |
| 持续 5xx | 熔断打开并降级 |
| 429 限流 | 退避重试，不风暴 |
| 全部失败 | 静态兜底并告警 |
| 高风险场景 | 明确失败转人工 |

# 11. 小结

## 11.1 核心要点

- 可靠性五件套：超时、重试、幂等、限流、熔断、降级。
- 只重试可重试错误，用指数加抖动退避。
- 降级链：主模型、备用、缓存、简化、兜底。
- 组合顺序：熔断（外）、限流、重试（内）。
- 高风险场景宁可失败，不可降级。

## 11.2 落地清单

- 先补超时与幂等，再加重试与退避。
- 用统一工厂封装防护组合顺序。
- 降级全程打点，接入告警。
- 高风险场景单独定义"不降级"策略。
