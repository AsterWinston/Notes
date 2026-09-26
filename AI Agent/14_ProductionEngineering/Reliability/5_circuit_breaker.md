# 1. 为什么需要熔断

熔断器（circuit breaker）是应对依赖持续故障的保护模式。结论先行：**当依赖已经持续故障时，继续调用只会消耗自身资源并放大故障，熔断让系统快速失败，把资源留给可用路径**。

## 1.1 故障级联

```
依赖持续故障时，如果继续调用：
- 每个请求都要等超时（浪费线程）
- 大量重试放大故障
- 级联失败，整个服务不可用
```

### 1.1.1 线程被耗尽

若依赖平均响应 10 秒、超时设 10 秒，则每个请求会占住一个线程 10 秒。当并发请求数超过线程池容量时，新请求排队，最终整个服务无可用线程。

### 1.1.2 重试放大故障

重试（retry）在依赖故障时会把请求量放大数倍。原本已过载的依赖收到更多请求，恢复更加困难，形成正反馈。

### 1.1.3 级联失败

上游服务因下游卡住而排队，上游的上游又因上游排队而卡住，故障沿调用链向上蔓延，最终整个系统不可用。

## 1.2 快速失败

熔断让系统在依赖故障时**快速失败**，而不是耗死自己。快速失败有三个好处：立即释放资源、给依赖恢复时间、向上游明确暴露问题。

## 1.3 熔断与超时/重试的关系

| 机制 | 触发条件 | 作用范围 |
|------|----------|----------|
| 超时 | 单次调用超时 | 单请求 |
| 重试 | 单次失败 | 单请求 |
| 熔断 | 失败率超阈值 | 一段时间内所有请求 |

熔断是「群体性」保护，超时与重试是「个体性」保护。三者组合才完整。

## 1.4 边界与反例

- **反例：熔断器包住了整个应用**。若所有依赖共用一个熔断器，任一依赖故障会熔断全部调用。
- **边界：熔断不解决依赖本身的故障**。它只是止损，依赖恢复仍需人工或自动修复。
- **边界：首次故障无保护**。熔断需要累积统计，冷启动阶段没有历史数据，可能误判。

# 2. 熔断器状态机

结论先行：**熔断器是有限状态机，CLOSED 正常放行并统计，OPEN 直接拒绝，HALF_OPEN 试探恢复，三态循环**。

## 2.1 三态定义

```
               失败率超阈值
    ┌────────┐ ──────────────▶ ┌────────┐
    │ CLOSED │                 │  OPEN  │
    │ 正常    │ ◀────────────── │ 熔断   │
    └────────┘   成功率达到阈值  └────┬───┘
         ▲                           │ 等待时间到
         │                           ▼
         │                   ┌────────────┐
         └───────────────────│ HALF_OPEN  │
              试探成功        │  半开试探   │
                             └────────────┘
                                   │ 试探失败
                                   ▼
                                （回到 OPEN）
```

| 状态 | 行为 |
|------|------|
| CLOSED | 正常调用，统计失败率 |
| OPEN | 直接拒绝，不调用依赖 |
| HALF_OPEN | 放行少量请求试探 |

### 2.1.1 CLOSED

CLOSED（闭合）是正常状态。所有请求放行到依赖，同时记录成功/失败/慢调用，滚动计算失败率。当失败率超过阈值且满足最小请求数时，转入 OPEN。

### 2.1.2 OPEN

OPEN（断开）是熔断状态。所有请求立即被拒绝，抛出 `CallNotPermittedException`，不产生任何网络调用。经过 `waitDurationInOpenState` 后自动转入 HALF_OPEN。

### 2.1.3 HALF_OPEN

HALF_OPEN（半开）是试探状态。只放行 `permittedNumberOfCallsInHalfOpenState` 个请求。若这些请求成功率达标，转回 CLOSED；否则回到 OPEN 并重新计时。

## 2.2 状态转换条件

| 转换 | 条件 |
|------|------|
| CLOSED → OPEN | 失败率 ≥ 阈值 且 请求数 ≥ 最小请求数 |
| OPEN → HALF_OPEN | 经过 waitDurationInOpenState |
| HALF_OPEN → CLOSED | 试探请求成功率达标 |
| HALF_OPEN → OPEN | 试探请求中出现失败 |

## 2.3 边界与反例

- **反例：HALF_OPEN 放行过多**。若试探数等于窗口大小，等于直接恢复全量流量，依赖可能再次被打垮。
- **边界：HALF_OPEN 期间新请求**。多数实现会拒绝 HALF_OPEN 之外的请求，避免试探流量被淹没。
- **边界：OPEN 期间计时基于本地时钟**。多实例各自计时，恢复时间不同步。

# 3. 关键参数

结论先行：**参数决定熔断灵敏度：阈值过高则形同虚设，过低则频繁误熔断；最小请求数与窗口大小用于过滤小样本噪声**。

## 3.1 参数表

| 参数 | 说明 | 建议 |
|------|------|------|
| 失败率阈值 | 触发熔断的失败比例 | 50% |
| 滑动窗口 | 统计窗口大小 | 10~100 次 |
| 最小请求数 | 窗口内最少请求数 | 10 |
| 熔断时长 | OPEN 持续时间 | 30~60s |
| 半开试探数 | HALF_OPEN 放行数 | 3~10 |

### 3.1.1 参数调优方法

参数调优应基于依赖的故障特征与恢复能力：

- **恢复快的依赖**（如可快速扩容的下游）：熔断时长可短，快速试探恢复。
- **恢复慢的依赖**（如人工修复的第三方）：熔断时长宜长，避免反复冲击。
- **错误容忍度低的业务**：失败率阈值应低，尽早熔断。
- **冷启动或低频依赖**：改用时间窗口并降低最小请求数，避免熔断永不触发。

## 3.2 窗口类型

### 3.2.1 计数窗口

`COUNT_BASED` 按请求次数统计，例如最近 20 次请求。适合请求量稳定的场景，不受时间影响。

### 3.2.2 时间窗口

`TIME_BASED` 按时间统计，例如最近 60 秒。适合请求量波动大的场景，反映真实时间维度。

| 窗口类型 | 统计依据 | 适用 |
|----------|----------|------|
| COUNT_BASED | 请求数 | 流量稳定 |
| TIME_BASED | 时间 | 流量波动 |

## 3.3 慢调用熔断

除失败率外，还应监控慢调用率（slow call rate）。依赖虽未失败但响应极慢时，同样会拖垮系统。

```java
CircuitBreakerConfig.custom()
        .slowCallRateThreshold(80)                       // 慢调用率 80%
        .slowCallDurationThreshold(Duration.ofSeconds(10)) // 超过 10s 算慢
        .build();
```

## 3.4 边界与反例

- **反例：最小请求数过小**。设为 1 时，单次失败即熔断，误判率极高。
- **边界：失败率阈值 100%**。只有全部失败才熔断，几乎失去意义，等于依赖已完全不可用。
- **边界：窗口内请求稀疏**。低流量场景下，计数窗口可能长时间凑不齐最小请求数，导致熔断失效，此时应改用时间窗口。

# 4. Resilience4j 实现

结论先行：**Resilience4j 是 JVM 上最成熟的熔断实现，通过装饰器（decorator）无侵入地包裹调用，并提供事件与指标**。

## 4.1 基础用法

```java
CircuitBreaker circuitBreaker = CircuitBreaker.of("llm",
        CircuitBreakerConfig.custom()
                .failureRateThreshold(50)                    // 失败率 50%
                .slidingWindowType(SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(20)                       // 窗口 20 次
                .minimumNumberOfCalls(10)                    // 最少 10 次才统计
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .permittedNumberOfCallsInHalfOpenState(5)
                .slowCallRateThreshold(80)                   // 慢调用率
                .slowCallDurationThreshold(Duration.ofSeconds(10))
                .build());

public String call(String prompt) {
    return CircuitBreaker.decorateSupplier(circuitBreaker,
            () -> model.chat(prompt)).get();
}
```

## 4.2 配置详解

| 方法 | 作用 | 默认 |
|------|------|------|
| `failureRateThreshold` | 失败率阈值 | 50 |
| `slidingWindowType` | 窗口类型 | COUNT_BASED |
| `slidingWindowSize` | 窗口大小 | 100 |
| `minimumNumberOfCalls` | 最小请求数 | 100 |
| `waitDurationInOpenState` | 熔断时长 | 60s |
| `permittedNumberOfCallsInHalfOpenState` | 半开试探数 | 10 |
| `recordExceptions` | 计入失败的异常 | 空 |
| `ignoreExceptions` | 忽略的异常 | 空 |

### 4.2.1 异常记录策略

默认所有异常都计入失败。生产上应区分「业务异常」与「系统异常」：参数校验失败不应触发熔断。

```java
CircuitBreakerConfig.custom()
        .recordExceptions(IOException.class, TimeoutException.class)
        .ignoreExceptions(BusinessException.class, ValidationException.class)
        .build();
```

### 4.2.2 注册表

`CircuitBreakerRegistry` 用于集中管理多个熔断器，可按名称获取或创建。

```java
CircuitBreakerRegistry registry = CircuitBreakerRegistry.ofDefaults();
CircuitBreaker llmBreaker = registry.circuitBreaker("llm");
CircuitBreaker toolBreaker = registry.circuitBreaker("tool.search");
```

## 4.3 事件监听

### 4.3.1 事件类型

Resilience4j 会发布状态转换与调用结果事件：`StateTransition`、`Success`、`Error`、`CallNotPermitted`、`IgnoredError`、`SlowCall`。

### 4.3.2 监听器

```java
circuitBreaker.getEventPublisher()
        .onStateTransition(event ->
                log.warn("熔断状态变更: {} -> {}",
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState()))
        .onCallNotPermitted(event ->
                metrics.counter("cb.rejected", "name", "llm").increment());
```

## 4.4 边界与反例

- **反例：不区分异常类型**。用户输入错误导致的 `IllegalArgumentException` 计入失败会误熔断。
- **边界：装饰器只包裹同步调用**。异步或响应式（reactive）调用需使用对应的操作符，如 `transformDeferred`。
- **边界：全局注册表内存**。按动态 key 创建熔断器时需控制数量，避免内存泄漏。

# 5. 与重试组合

结论先行：**装饰顺序决定语义，正确顺序是「熔断（外）→ 限流 → 重试（内）→ 超时 → 实际调用」，熔断打开时不应重试**。

## 5.1 装饰顺序

```
正确顺序：熔断（外）→ 限流 → 重试（内）→ 实际调用
```

为什么熔断在最外层？因为熔断打开时需要立即拒绝，不进入限流与重试逻辑。若熔断在内层，重试会先消耗次数再被熔断拒绝，浪费资源。

```
调用流:
  熔断 ──OPEN?──▶ 拒绝
   │CLOSED
   ▼
  限流 ──超限?──▶ 拒绝
   │放行
   ▼
  重试 ──失败?──▶ 重试（最多 N 次）
   │成功
   ▼
  实际调用
```

## 5.2 代码

```java
Supplier<String> decorated = Decorators.ofSupplier(() -> model.chat(prompt))
        .withCircuitBreaker(circuitBreaker)
        .withRateLimiter(rateLimiter)
        .withRetry(retry)
        .withTimeLimiter(timeLimiter)
        .decorate();
```

> **熔断打开时不重试**：重试会立即被熔断器拒绝，浪费资源。

## 5.3 边界与反例

- **反例：重试包在熔断外**。每次重试都重新判定熔断，看似合理，实则熔断已打开时仍会执行重试循环，产生无谓开销。
- **边界：重试计数与熔断窗口**。重试的失败会被熔断器计为多次失败，加速熔断，需评估是否符合预期。
- **边界：超时与熔断时长**。单次超时应小于熔断时长，否则 HALF_OPEN 试探也会超时失败。

# 6. 熔断后的降级

结论先行：**熔断打开时必须提供降级路径，否则熔断只是把「慢失败」变成「快失败」，用户仍得不到服务**。

## 6.1 降级链

```
主模型 → 备用模型 → 缓存结果 → 静态兜底 → 明确失败
```

降级链（fallback chain）逐级尝试，尽量给出可用结果；全部失败时明确返回错误，而非静默失败。

## 6.2 代码

```java
public String chatWithFallback(String prompt) {
    try {
        return CircuitBreaker.decorateSupplier(circuitBreaker,
                () -> primaryModel.chat(prompt)).get();

    } catch (CallNotPermittedException e) {
        // 熔断打开 → 降级
        log.warn("主模型熔断，降级到备用模型");
        return fallbackModel.chat(prompt);

    } catch (Exception e) {
        // 其他异常 → 兜底
        return "服务暂时不可用，请稍后重试";
    }
}
```

## 6.3 边界与反例

- **反例：降级路径也依赖同一故障依赖**。备用模型若与主模型同厂商同区域，会一起故障。
- **边界：降级质量**。缓存或兜底结果可能过时或不完整，需标注来源与时效。
- **边界：降级路径无限递归**。降级调用自身也需限流与熔断，避免降级层被打垮。

# 7. Agent 场景的熔断对象

结论先行：**Agent 依赖多样，应对模型、工具、向量库、重排、MCP Server 分别熔断，实现故障隔离**。

## 7.1 对象表

| 对象 | 说明 |
|------|------|
| 模型 | 某个模型不可用 |
| 工具 | 下游服务故障 |
| 向量库 | 检索不可用 |
| 重排服务 | 重排不可用（可跳过） |
| MCP Server | 外部工具故障 |

## 7.2 每工具独立熔断

```java
// 每个工具独立熔断
private final Map<String, CircuitBreaker> toolBreakers = new ConcurrentHashMap<>();

public ToolResult execute(ToolCall call) {
    CircuitBreaker breaker = toolBreakers.computeIfAbsent(
            call.name(), n -> CircuitBreaker.of(n, defaultConfig()));

    try {
        String result = CircuitBreaker.decorateSupplier(breaker,
                () -> doExecute(call)).get();
        return ToolResult.ok(result);
    } catch (CallNotPermittedException e) {
        return ToolResult.fail("工具暂时不可用（熔断中）");
    }
}
```

### 7.2.1 熔断器生命周期

动态工具名会导致熔断器无限增长，需设置容量上限与淘汰策略：

| 问题 | 方案 |
|------|------|
| 数量膨胀 | 按工具组而非工具名建熔断器 |
| 内存泄漏 | 使用 LRU 或定时清理 |
| 冷启动误判 | 降低最小请求数或预热 |

## 7.3 结构化错误

> 工具熔断后返回**结构化错误**给模型，让模型换策略。

不要只返回纯文本错误。结构化错误应包含错误码、是否可重试、建议动作，便于模型或编排器决策。

```java
public record ToolError(String code, boolean retryable, String suggestion) {
    public static ToolError circuitOpen(String tool) {
        return new ToolError("TOOL_CIRCUIT_OPEN", false,
                "请改用其他工具或稍后重试");
    }
}
```

## 7.4 边界与反例

- **反例：全局一个熔断器**。一个工具故障导致所有工具被熔断，丧失隔离意义。
- **边界：工具熔断的粒度**。同名工具的不同租户/环境是否共享熔断器需明确，通常按工具名共享。
- **边界：可跳过的依赖**。重排服务熔断时应跳过而非失败，需在编排层处理。

# 8. 分布式熔断

结论先行：**单机熔断器无法感知集群整体故障，大规模部署下应评估共享状态熔断或网关熔断**。

## 8.1 单机局限

单机熔断器无法感知集群整体故障。方案：

| 方案 | 说明 |
|------|------|
| 本地熔断 | 每实例独立（简单，有延迟） |
| 共享状态熔断 | 用 Redis 共享计数 |
| 网关熔断 | 在网关层统一熔断 |
| 服务网格 | Istio 等提供 |

## 8.2 方案对比

| 方案 | 一致性 | 复杂度 | 适用 |
|------|:------:|:------:|------|
| 本地熔断 | 弱 | 低 | 依赖故障快速传播 |
| 共享状态 | 强 | 高 | 需全局一致 |
| 网关熔断 | 中 | 中 | 南北向流量 |
| 服务网格 | 中 | 中 | 已用 Istio |

## 8.3 共享状态实现

```java
// Resilience4j 支持共享 CircuitBreakerRegistry
CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
// 多实例可通过自定义 StateRepository 共享状态
```

共享状态需权衡：Redis 往返增加延迟，且 Redis 本身成为故障点。多数场景下本地熔断已足够，因为依赖故障会同时影响所有实例，各实例会几乎同时熔断。

## 8.4 边界与反例

- **反例：过早引入共享状态**。增加复杂度与延迟，收益有限。
- **边界：状态同步延迟**。共享状态有网络延迟，极端情况下仍有窗口期超发。
- **边界：熔断风暴**。共享状态熔断后，所有实例同时进入 HALF_OPEN，会同时试探，需加随机抖动。

# 9. 监控与告警

结论先行：**熔断必须可观测：状态、失败率、慢调用率、拒绝数都应有指标，进入 OPEN 应触发告警**。

## 9.1 指标

```java
registry.counter("circuit.state", "name", name, "state", state).increment();
```

| 指标 | 告警 |
|------|------|
| 熔断器状态 | 进入 OPEN 告警 |
| 失败率 | > 50% 告警 |
| 慢调用率 | > 80% 告警 |
| 熔断时长 | 长时间未恢复告警 |

## 9.2 事件驱动

推荐用事件监听代替轮询指标，实时性更好：

```java
circuitBreaker.getEventPublisher()
        .onStateTransition(e -> {
            if (e.getStateTransition().getToState() == State.OPEN) {
                alertService.fire("熔断打开", circuitBreaker.getName());
            }
        });
```

## 9.3 边界与反例

- **反例：只告警不追踪恢复**。需监控从 OPEN 恢复的时长，长时间不恢复说明依赖未修复。
- **边界：告警风暴**。大量实例同时熔断会产生告警风暴，需聚合去重。

# 10. 熔断 vs 限流 vs 重试

结论先行：**四者解决不同问题、组合使用：限流防过载、重试治抖动、熔断止损、超时防卡死**。

## 10.1 对比表

| 机制 | 解决的问题 | 触发条件 |
|------|------------|----------|
| 限流 | 流量过大 | 超过速率阈值 |
| 重试 | 临时抖动 | 单次失败 |
| 熔断 | 持续故障 | 失败率超阈值 |
| 超时 | 单次卡住 | 超过时间 |

> 四者组合使用，缺一不可。

## 10.2 组合顺序

```
请求
  │
  ▼
熔断 ──OPEN?──▶ 降级
  │CLOSED
  ▼
限流 ──超限?──▶ 拒绝
  │放行
  ▼
超时
  │
  ▼
重试 ──失败?──▶ 重试
  │
  ▼
实际调用
```

## 10.3 边界与反例

- **反例：只加熔断不加超时**。熔断统计依赖超时判定失败，没有超时则慢调用不会被计为失败。
- **边界：重试与限流的叠加**。重试会放大流量，限流阈值需考虑重试倍数。

# 11. 生产注意事项

## 11.1 常见风险表

| 风险 | 建议 |
|------|------|
| 阈值拍脑袋 | 按依赖特性调优 |
| 窗口太小 | 易误熔断 |
| 熔断后无降级 | 必须有兜底 |
| 熔断打开仍重试 | 组合顺序正确 |
| 无监控 | 监控状态与失败率 |
| 单机熔断失效 | 考虑共享状态 |
| 不区分异常 | 忽略业务异常 |
| 半开试探过多 | 试探数宜小 |

## 11.2 配置清单

```yaml
resilience4j:
  circuitbreaker:
    instances:
      llm:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 10s
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        minimum-number-of-calls: 10
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
```

# 12. 小结

- 熔断在依赖持续故障时快速失败，防止级联崩溃。
- 三态：CLOSED / OPEN / HALF_OPEN。
- 组合顺序：熔断（外）→ 限流 → 重试（内）。
- 熔断必须配合降级：备用模型、缓存、兜底。
- 熔断打开时不应重试。
- 阈值按依赖特性调优，并区分业务异常与系统异常。
