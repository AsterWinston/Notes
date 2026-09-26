# 1. 错误处理的目标与原则

## 1.1 目标：不中断 Agent

**结论先行**：错误处理的核心目标是：不让单个工具失败中断整个 Agent（智能体）循环，而是把错误变成"可被模型理解并处理"的信息。

## 1.2 从异常到信息

### 1.2.1 异常中断的代价

当工具直接抛出异常时，框架通常会把异常向上抛，导致整轮对话失败。用户看到的是"系统错误"，模型没有机会换参数、换工具或告知用户。

### 1.2.2 结构化错误的收益

| 维度 | 抛异常 | 结构化错误 |
|------|--------|------------|
| Agent 循环 | 中断 | 继续 |
| 模型纠错 | 无机会 | 可调整 |
| 可观测 | 差 | 按错误码统计 |
| 用户体验 | 差 | 可解释 |

## 1.3 设计原则

**结论先行**：工具错误应满足"结构化、可操作、不泄露、不冗长、一致"五条原则。

| 原则 | 说明 |
|------|------|
| 结构化 | 有 code、message、retryable |
| 可操作 | 给出建议（suggestion） |
| 不泄露 | 不暴露内部实现细节 |
| 不冗长 | 简短明确 |
| 一致 | 同类错误统一格式 |

## 1.4 边界与生产建议

**边界与坑**：

- 把所有异常都吞掉会让真实缺陷被掩盖，必须保留日志。
- 结构化错误不等于丢弃堆栈，堆栈只进日志，不进模型上下文。

**生产建议**：

- 在工具执行器统一兜底，避免每个工具各写一套。
- 为错误处理定义验收用例，覆盖每类错误。

# 2. 错误分类

## 2.1 分类维度

**结论先行**：错误按"是否可重试"和"模型能否处理"两个维度分类，决定了后续的重试、降级与提示策略。

### 2.1.1 可重试性

可重试错误通常来自瞬时故障（超时、限流、下游 5xx），重试有较大概率成功。

### 2.1.2 模型可处理性

模型可处理的错误能通过改参数、换查询或换工具解决；不可处理的错误应直接告知用户。

## 2.2 完整分类表

| 类别 | 例子 | 可重试 | 模型能否处理 |
|------|------|:------:|:------------:|
| 参数错误 | 格式不对、缺字段 | 否 | 能（改参数） |
| 权限错误 | 无权限 | 否 | 有限（换方式） |
| 资源不存在 | 订单不存在 | 否 | 能（换查询） |
| 超时 | 网络慢 | 是 | 能（重试） |
| 限流 | 调用过频 | 是 | 能（等待） |
| 下游故障 | 服务 500 | 是 | 有限（换工具） |
| 业务失败 | 余额不足 | 否 | 能（告知用户） |
| 未知错误 | 系统异常 | 视情况 | 否 |

## 2.3 分类决策树

```
错误发生
   |
   +-- 参数问题? ---- 是 --> INVALID_ARGUMENT（不可重试）
   |
   +-- 资源不存在? -- 是 --> NOT_FOUND（不可重试）
   |
   +-- 瞬时故障? ---- 是 --> TIMEOUT/RATE_LIMITED（可重试）
   |
   +-- 业务规则? ---- 是 --> BUSINESS_FAILED（不可重试）
   |
   +-- 其他 ---------------> INTERNAL_ERROR（记录并告警）
```

## 2.4 边界与生产建议

**边界与坑**：

- 同一异常在不同上下文可能属于不同类别，需结合工具语义判断。
- "未知错误"要默认不可重试，避免放大问题。

**生产建议**：

- 把分类规则固化为枚举与映射表，避免散落各处。
- 定期复盘错误码分布，优化分类。

# 3. 结构化错误模型

## 3.1 ToolError 与 ToolResult

**结论先行**：用 `ToolError` 描述错误，用 `ToolResult<T>` 承载成功或失败，是所有工具的统一返回契约。

```java
public record ToolError(
        String code,           // 机器可读的错误码
        String message,        // 人类可读的描述
        boolean retryable,     // 是否可重试
        String suggestion      // 给模型的建议
) {}

public record ToolResult<T>(
        boolean success,
        T data,
        ToolError error
) {
    public static <T> ToolResult<T> ok(T data) {
        return new ToolResult<>(true, data, null);
    }

    public static <T> ToolResult<T> fail(String code, String message,
                                         boolean retryable, String suggestion) {
        return new ToolResult<>(false, null,
                new ToolError(code, message, retryable, suggestion));
    }
}
```

## 3.2 泛型结果设计

**结论先行**：`ToolResult<T>` 用泛型保留数据类型，避免全部退化为 `Map`。

| 设计点 | 说明 |
|--------|------|
| 成功数据 | 泛型 `T`，编译期类型安全 |
| 失败信息 | 统一 `ToolError` |
| 序列化 | 需可 JSON 化，避免循环引用 |

## 3.3 错误码规范

| 前缀 | 类别 | 示例 |
|------|------|------|
| INVALID_ | 参数 | INVALID_ARGUMENT |
| NOT_ | 资源 | NOT_FOUND |
| FORBIDDEN | 权限 | FORBIDDEN |
| TIMEOUT | 超时 | TIMEOUT |
| RATE_ | 限流 | RATE_LIMITED |
| INTERNAL_ | 内部 | INTERNAL_ERROR |

## 3.4 边界与生产建议

**边界与坑**：

- `code` 一旦对外发布就应保持稳定，不能随意改名。
- `suggestion` 为空会让模型无从下手，宁可不给也不要给错误引导。

**生产建议**：

- 错误码集中定义在枚举中，工具与监控共用。
- 为每个错误码编写文档说明与示例。

# 4. 错误处理实现

## 4.1 工具内 try-catch

**结论先行**：每个工具都应捕获异常并转换为结构化错误，绝不向框架抛出原始异常。

### 4.1.1 分类型捕获

按业务语义分类型捕获，映射到不同错误码与建议。

### 4.1.2 兜底捕获

最后的 `catch (Exception e)` 捕获所有未预期异常，记录日志并返回 `INTERNAL_ERROR`。

## 4.2 完整示例

```java
@Tool("查询订单")
public ToolResult queryOrder(@ToolParam(description = "订单号") String orderId) {
    try {
        Order order = orderService.get(orderId);
        return ToolResult.ok(toSummary(order));

    } catch (IllegalArgumentException e) {
        // 参数错误 → 建议改参数
        return ToolResult.fail("INVALID_ARGUMENT",
                "订单号格式错误", false,
                "订单号应为字母+数字格式，如 A123");

    } catch (NotFoundException e) {
        // 资源不存在 → 建议确认
        return ToolResult.fail("NOT_FOUND",
                "订单不存在：" + orderId, false,
                "请确认订单号是否正确");

    } catch (AccessDeniedException e) {
        return ToolResult.fail("FORBIDDEN",
                "无权限查询该订单", false,
                "该订单不属于当前用户，无法查询");

    } catch (TimeoutException e) {
        return ToolResult.fail("TIMEOUT",
                "查询超时", true,
                "可稍后重试");

    } catch (RateLimitException e) {
        return ToolResult.fail("RATE_LIMITED",
                "调用过于频繁", true,
                "请稍等几秒后重试");

    } catch (Exception e) {
        log.error("工具执行异常: {}", orderId, e);
        return ToolResult.fail("INTERNAL_ERROR",
                "系统异常", false,
                "该功能暂时不可用，请换用其他方式");
    }
}
```

## 4.3 全局异常处理器

```java
@Component
@RequiredArgsConstructor
public class SafeToolExecutor {

    public ToolResult execute(ToolCall call) {
        try {
            return call.invoke();
        } catch (Exception e) {
            log.error("工具 {} 执行失败", call.name(), e);
            return ToolResult.fail("INTERNAL_ERROR",
                    "系统异常", false, "请稍后重试或换用其他方式");
        }
    }
}
```

## 4.4 边界与生产建议

**边界与坑**：

- 捕获顺序必须从具体到宽泛，否则子类异常会被父类提前捕获。
- `catch (Throwable t)` 会吞掉 `OutOfMemoryError` 等致命错误，应避免。

**生产建议**：

- 工具层与执行器层双重兜底，避免遗漏。
- 兜底日志包含工具名、参数摘要与追踪 ID。

# 5. 错误信息设计原则

## 5.1 五条原则

**结论先行**：错误信息是给模型看的，必须短、准、可操作，且不泄露内部实现。

```java
// 错误：泄露内部细节
return ToolResult.fail("java.sql.SQLException: Table 'orders' doesn't exist at line 42");

// 正确：面向模型
return ToolResult.fail("INTERNAL_ERROR", "查询服务暂时不可用", false, "请稍后重试");
```

## 5.2 脱敏与防泄露

### 5.2.1 泄露的内部细节

| 泄露内容 | 风险 |
|----------|------|
| 堆栈信息 | 暴露技术栈与文件路径 |
| SQL 语句 | 暴露表结构 |
| 内部 IP/主机名 | 暴露网络拓扑 |
| 密钥片段 | 直接安全事故 |

### 5.2.2 脱敏实现

```java
private String sanitize(String raw) {
    return raw
            .replaceAll("(?i)(password|token|secret)=\\S+", "$1=***")
            .replaceAll("\\b\\d{11}\\b", "***********");
}
```

## 5.3 边界与生产建议

**边界与坑**：

- 过度脱敏会让错误无法排查，脱敏规则要可配置。
- 拼接用户输入到错误信息时，注意防止注入到下游系统。

**生产建议**：

- 统一脱敏工具，所有对外错误信息都经过它。
- 脱敏规则纳入安全评审。

# 6. 模型如何利用错误

## 6.1 错误反馈循环

**结论先行**：错误返回给模型后，模型会尝试自我纠正，`suggestion` 是引导纠正的关键。

```
模型收到：{"success":false,"error":{"code":"NOT_FOUND","message":"订单不存在：A123",
          "suggestion":"请确认订单号是否正确"}}

模型反应：
① 可能意识到订单号错了 -> 询问用户
② 可能尝试其他订单号
③ 可能换用其他工具
```

> **给出 suggestion 能显著提升模型的自我纠错率。**

## 6.2 suggestion 的作用

| suggestion | 模型行为 |
|------------|----------|
| 修正参数 | 重新调用同工具 |
| 换工具 | 调用替代工具 |
| 告知用户 | 生成解释性回复 |
| 稍后重试 | 延迟后重试 |

## 6.3 边界与生产建议

**边界与坑**：

- 无限纠错会消耗大量 Token，应设置最大迭代次数。
- `suggestion` 若与 `retryable` 矛盾，模型行为会混乱。

**生产建议**：

- 提示词中明确"根据错误信息决定下一步"。
- 记录模型的纠错路径，用于优化 suggestion 质量。

# 7. 自动重试

## 7.1 重试前提

**结论先行**：只有可重试错误才自动重试，且必须配合退避、上限与幂等，否则会放大故障。

## 7.2 退避策略

### 7.2.1 指数退避

每次重试等待时间按倍数增长，给下游恢复时间。

### 7.2.2 抖动

在退避时间上加入随机抖动，避免大量请求同时重试形成风暴。

```java
private long backoff(int attempt) {
    long base = Math.min(200L << attempt, 2000L);
    return base + ThreadLocalRandom.current().nextLong(0, 100);
}
```

## 7.3 重试执行器

```java
public class RetryingToolExecutor {

    public ToolResult execute(ToolCall call) {
        ToolResult result = doExecute(call);

        // 只对可重试错误重试
        if (!result.success() && result.error().retryable()) {
            for (int i = 0; i < MAX_RETRIES; i++) {
                sleep(backoff(i));
                result = doExecute(call);
                if (result.success() || !result.error().retryable()) {
                    break;
                }
            }
        }
        return result;
    }
}
```

> 详见 `14_ProductionEngineering/Reliability/2_tool_retry.md`。

## 7.4 边界与生产建议

**边界与坑**：

- 重试写操作若无幂等键，会造成重复扣款、重复下单。
- 重试总耗时必须受整体超时预算约束。

**生产建议**：

- 重试次数 2–3 次即可，过多收益递减。
- 重试指标接入监控，异常上升时告警。

# 8. 熔断与降级

## 8.1 熔断器状态机

**结论先行**：熔断器在错误率超阈值时打开，快速失败，避免线程被拖垮；冷却后进入半开探测。

```
    错误率超阈值          冷却时间到
CLOSED ---------> OPEN -----------> HALF_OPEN
   ^                                  |
   |        探测成功                   |
   +----------------------------------+
                     探测失败 -> OPEN
```

## 8.2 代码实现

```java
public ToolResult executeWithCircuitBreaker(ToolCall call) {
    if (circuitBreaker.isOpen(call.name())) {
        return ToolResult.fail("SERVICE_UNAVAILABLE",
                "该工具暂时不可用", true,
                "请换用其他方式或稍后重试");
    }
    try {
        ToolResult result = doExecute(call);
        circuitBreaker.recordSuccess(call.name());
        return result;
    } catch (Exception e) {
        circuitBreaker.recordFailure(call.name());
        throw e;
    }
}
```

## 8.3 降级策略

| 策略 | 说明 |
|------|------|
| 返回缓存 | 使用上次成功结果 |
| 换工具 | 使用替代实现 |
| 部分结果 | 返回已知可用部分 |
| 明确失败 | 告知模型暂时不可用 |

## 8.4 边界与生产建议

**边界与坑**：

- 熔断粒度按工具或下游服务划分，过粗会误伤健康工具。
- 半开状态要限制探测并发，避免瞬间打满。

**生产建议**：

- 熔断阈值与恢复时间需结合压测确定。
- 熔断事件必须记录并告警。

# 9. 监控与告警

## 9.1 关键指标

**结论先行**：按工具与错误码维度统计错误率、重试率与熔断次数，才能快速定位问题。

```java
registry.counter("tool.errors",
        "tool", call.name(),
        "code", error.code()).increment();
```

| 指标 | 告警 |
|------|------|
| 工具错误率 | 大于 10% |
| 特定错误码突增 | 突增 |
| 重试率 | 上升 |
| 熔断次数 | 任意 |

## 9.2 埋点实现

```java
@Around("execution(* com.example.tools..*(..))")
public Object track(ProceedingJoinPoint pjp) throws Throwable {
    String tool = pjp.getSignature().getName();
    Timer.Sample sample = Timer.start(registry);
    try {
        Object result = pjp.proceed();
        sample.stop(registry.timer("tool.latency", "tool", tool));
        return result;
    } catch (Throwable t) {
        registry.counter("tool.errors", "tool", tool, "code", "UNKNOWN").increment();
        throw t;
    }
}
```

## 9.3 告警阈值

| 场景 | 阈值 | 动作 |
|------|------|------|
| 错误率 | 大于 10% | 通知值班 |
| 熔断 | 任意 | 立即通知 |
| 连续超时 | 5 次 | 通知 |
| 未知错误 | 突增 | 排查 |

## 9.4 边界与生产建议

**边界与坑**：

- 指标基数过高（如把参数值作为标签）会拖垮监控系统。
- 告警过滥会导致疲劳，阈值需要调优。

**生产建议**：

- 标签只保留工具名与错误码，避免高基数。
- 建立错误码到责任人的映射，加速处置。

# 10. 常见坑、生产注意事项与小结

## 10.1 常见坑

| 坑 | 后果 | 解决 |
|----|------|------|
| 抛异常中断循环 | Agent 失败 | 返回结构化错误 |
| 错误信息过长 | 浪费 Token | 精简 |
| 泄露内部细节 | 安全风险 | 脱敏 |
| 无 suggestion | 模型无法纠错 | 给出建议 |
| 重试不可重试错误 | 放大问题 | 分类判断 |
| 错误静默 | 无法排查 | 记录日志 |
| 无监控 | 无法发现 | 指标告警 |

## 10.2 生产注意事项

| 风险 | 建议 |
|------|------|
| 异常中断 | 返回结构化错误 |
| 无错误码 | 定义错误码 |
| 无建议 | 给模型 suggestion |
| 信息泄露 | 脱敏 |
| 重试无差别 | 只重试可重试 |
| 无监控 | 按工具与错误码统计 |

## 10.3 小结

- 工具错误要变成"模型能理解并处理"的信息。
- 结构化错误：code + message + retryable + suggestion。
- 不泄露内部细节，给出可操作建议。
- 只对可重试错误自动重试，配合熔断降级。
- 错误必须监控与告警。
