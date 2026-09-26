# 1. Token 决定一切

Token（词元）是模型计费与运算的基本单位。输入与输出都以 Token 计量，因此 Token 数直接决定成本、延迟、上下文容量与限流额度。

## 1.1 Token 的四重影响

结论先行：Token 是 LLM 应用中唯一同时影响成本、性能与容量的变量，必须作为一等指标治理。

### 1.1.1 成本

多数厂商按输入、输出分别计费，输出通常比输入贵 2~4 倍。

### 1.1.2 延迟

输入越长，首 Token 延迟（TTFT）越高；输出越长，总耗时越长。

### 1.1.3 上下文容量

上下文窗口（Context Window）有限，Token 超限会报错或被截断。

### 1.1.4 限流

厂商按 Token 每分钟（TPM）与请求每分钟（RPM）限流，Token 突增会触发 429。

```
Token → 成本 + 延迟 + 上下文容量 + 限流
```

## 1.2 输入与输出的不对称

| 维度 | 输入 Token | 输出 Token |
|------|-----------|-----------|
| 单价 | 较低 | 较高 |
| 可控性 | 高（可裁剪） | 中（受提示影响） |
| 缓存 | 支持 Prompt 缓存 | 通常不支持 |

## 1.3 本文范围

> 详细策略见 `14_ProductionEngineering/Performance/3_token_budget_management.md` 与 `4_cost_optimization.md`，本文聚焦工程实现。

# 2. Token 估算

结论先行：生产环境用精确 tokenizer（分词器）估算，粗估仅用于快速判断。

## 2.1 粗估

经验规则：中文约 1 Token/字，英文约 1 Token/4 字符。

## 2.2 精确估算

### 2.2.1 jtokkit

```java
public class TokenEstimator {

    // 粗估：中文 ≈ 1 token/字，英文 ≈ 1 token/4 字符
    public static int estimate(String text) {
        int chinese = countChinese(text);
        int other = text.length() - chinese;
        return chinese + (other / 4) + 1;
    }

    // 精确：用模型对应的 tokenizer（如 jtokkit）
    public static int exact(String text, String model) {
        Encoding encoding = Encodings.newDefaultEncodingRegistry()
                .getEncodingForModel(model);
        return encoding.countTokens(text);
    }
}
```

### 2.2.2 其他模型

非 OpenAI 系模型需使用对应 tokenizer，或按厂商提供的换算系数近似。

> 用 [jtokkit](https://github.com/knuddelsgmbh/jtokkit) 在 Java 中精确计算 OpenAI 系模型的 Token。

## 2.3 边界与坑

- 粗估在代码、JSON、表情符号上误差较大。
- 不同模型分词方式不同，跨模型不可直接套用。
- 消息本身（role、分隔符）也会计入 Token。

## 2.4 生产建议

- 预算校验用精确值，实时监控可用粗估。
- 对高频文本做估算结果缓存。
- 预留 10%~20% 安全余量。

# 3. 成本计算

结论先行：成本 = 输入 Token × 输入单价 + 输出 Token × 输出单价，单价必须配置化。

## 3.1 定价模型

```java
public record ModelPricing(
        BigDecimal inputPerMillion,
        BigDecimal outputPerMillion) {

    public BigDecimal cost(long inputTokens, long outputTokens) {
        return inputPerMillion.multiply(BigDecimal.valueOf(inputTokens))
                .add(outputPerMillion.multiply(BigDecimal.valueOf(outputTokens)))
                .divide(BigDecimal.valueOf(1_000_000), 6, RoundingMode.HALF_UP);
    }
}
```

## 3.2 Java 实现

### 3.2.1 定价注册表

```java
@Component
public class PricingRegistry {

    private final Map<String, ModelPricing> pricing;    // 从配置加载

    public BigDecimal cost(String model, Usage usage) {
        ModelPricing p = pricing.get(model);
        if (p == null) {
            log.warn("未知模型定价: {}", model);
            return BigDecimal.ZERO;
        }
        return p.cost(usage.getPromptTokens(), usage.getCompletionTokens());
    }
}
```

### 3.2.2 配置示例

```yaml
ai:
  pricing:
    gpt-4o-mini:
      input-per-million: 0.15
      output-per-million: 0.60
```

## 3.3 缓存与批量的定价差异

| 类型 | 计费 |
|------|------|
| 普通输入 | 标准单价 |
| 缓存命中输入 | 通常 10%~50% 单价 |
| 缓存写入 | 可能加价 |
| 批量 API | 常见 50% 折扣 |

> 定价**配置化**，便于厂商调价时快速更新。

## 3.4 生产建议

- 定价变更记录生效时间，支持历史回溯。
- 未知模型必须告警，避免静默零成本。
- 定期与厂商账单对账。

# 4. 成本采集

结论先行：成本必须在调用链中自动采集，不能依赖人工统计。

## 4.1 Advisor 拦截

Spring AI 的 `CallAdvisor` 可在调用前后拦截，读取 `Usage` 并记录成本。

## 4.2 完整实现

### 4.2.1 指标与持久化

```java
public class CostTrackingAdvisor implements CallAdvisor {

    private final PricingRegistry pricing;
    private final CostStore costStore;
    private final MeterRegistry registry;

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);

        Usage usage = response.chatResponse().getMetadata().getUsage();
        if (usage == null) return response;

        String model = extractModel(response);
        String tenant = TenantContext.current();
        String userId = UserContextHolder.current().userId();

        BigDecimal cost = pricing.cost(model, usage);

        // 指标
        registry.counter("ai.cost", "model", model, "tenant", tenant)
                .increment(cost.doubleValue());
        registry.summary("ai.tokens.input", "model", model)
                .record(usage.getPromptTokens());
        registry.summary("ai.tokens.output", "model", model)
                .record(usage.getCompletionTokens());

        // 持久化（计费与归因）
        costStore.record(new CostRecord(tenant, userId, model,
                usage.getPromptTokens(), usage.getCompletionTokens(),
                cost, Instant.now(), MDC.get("traceId")));

        return response;
    }

    @Override
    public String getName() { return "CostTrackingAdvisor"; }

    @Override
    public int getOrder() { return 210; }
}
```

### 4.2.2 关键字段

| 字段 | 用途 |
|------|------|
| tenant | 多租户归因 |
| userId | 用户级限流 |
| model | 模型分层分析 |
| traceId | 与链路日志关联 |

## 4.3 归因维度

常见归因维度：租户、用户、模型、功能模块、会话。

## 4.4 生产建议

- 采集异步化，避免阻塞主链路。
- 落库失败要重试或降级到日志。
- 区分计费口径与内部核算口径。

# 5. 预算控制

结论先行：预算要"事前拦截 + 事中告警 + 事后对账"三段式。

## 5.1 事前校验

```java
public class BudgetEnforcer {

    public void checkBefore(UserContext user, int estimatedTokens) {
        String tenant = user.tenantId();

        // ① 预估成本
        BigDecimal estimated = pricing.estimateCost(tenant, estimatedTokens);

        // ② 已用成本
        BigDecimal used = costStore.dailyCost(tenant);

        // ③ 预算
        BigDecimal budget = budgetService.dailyBudget(tenant);

        if (used.add(estimated).compareTo(budget) > 0) {
            throw new BudgetExceededException("日预算已用尽");
        }
        // 预警
        if (used.compareTo(budget.multiply(new BigDecimal("0.8"))) > 0) {
            alertService.warn("租户成本接近预算: " + tenant);
        }
    }
}
```

## 5.2 分级预算

### 5.2.1 租户预算

按租户设置日/月预算，是主要的成本闸门。

### 5.2.2 用户预算

防止单用户刷量，常用每分钟 Token 上限。

### 5.2.3 单请求预算

限制单次请求最大 Token，防止超长输入拖垮预算。

## 5.3 边界与坑

- 并发下"检查—使用"存在竞态，需原子计数。
- 预估不准会导致实际超支，需留余量。
- 预算用尽要有友好降级而非直接报错。

## 5.4 生产建议

- 用 Redis 原子操作实现分布式计数。
- 预算阈值与告警分级配置化。
- 超预算自动切换到更小模型。

# 6. 降本手段

结论先行：优先做模型分层、缓存与上下文裁剪，这三项收益最大且改动可控。

## 6.1 手段总览

| 手段 | 节省 | 说明 |
|------|:----:|------|
| 模型分层 | 30%~70% | 简单任务用小模型 |
| 结果缓存 | 10%~40% | 热点问题复用 |
| Prompt 缓存 | 20%~50%（输入） | 稳定前缀前置 |
| 上下文裁剪 | 10%~30% | 减少检索片段 |
| 输出限制 | 中 | 限制 maxTokens |
| 批处理 | 10%~30% | 合并请求 |
| 本地 Embedding | 显著 | 零边际成本 |

## 6.2 模型分层

```java
// 伪代码：按任务复杂度路由模型
String model = complexity.classify(question) > THRESHOLD
        ? "gpt-4o"
        : "gpt-4o-mini";
```

## 6.3 结果缓存

```java
// 伪代码：对确定性问答做语义或精确缓存
String cached = cache.get(hash(question));
if (cached != null) return cached;
```

缓存需注意时效性，事实类问答不宜长缓存。

## 6.4 生产建议

- 分层路由先用规则，稳定后再上模型判断。
- 缓存键要包含模型与提示版本。
- 输出限制与任务需求平衡，避免截断。

# 7. 上下文裁剪

结论先行：只放最相关的片段，Token 省下来的是成本与延迟。

## 7.1 裁剪策略

按相关性排序后，在 Token 预算内贪心填充。

## 7.2 Java 实现

```java
public String buildContext(List<Document> docs, int maxTokens) {
    StringBuilder sb = new StringBuilder();
    int used = 0;

    for (Document doc : docs) {
        int tokens = TokenEstimator.estimate(doc.getText());
        if (used + tokens > maxTokens) break;

        sb.append(doc.getText()).append("\n---\n");
        used += tokens;
    }
    return sb.toString();
}
```

> 配合重排，**只放最相关的 3~5 个片段**。

## 7.3 边界与坑

- 贪心可能错过"短而关键"的片段。
- 截断单片段会破坏语义完整性。
- 去重不足会浪费预算。

## 7.4 生产建议

- 先重排再裁剪，提升单位 Token 信息量。
- 对片段做去重与压缩。
- 记录裁剪前后 Token，量化收益。

# 8. Prompt 缓存

结论先行：把稳定内容前置，命中厂商侧缓存可大幅降低输入成本。

## 8.1 原理

多数厂商对"相同前缀"缓存其 KV（键值）状态，命中后按更低单价计费。

## 8.2 Java 实现

```java
// 稳定内容前置，提高厂商侧缓存命中
String prompt = """
        %s        ← 系统提示（稳定）
        %s        ← 知识库上下文（半稳定）
        %s        ← 用户问题（每次都变）
        """.formatted(system, context, question);
```

## 8.3 边界与坑

- 前缀中任何变化都会导致缓存失效。
- 缓存有最短长度门槛。
- 缓存有过期时间，长期不用会失效。

## 8.4 生产建议

- 系统提示、工具定义保持字节级稳定。
- 避免在稳定区插入时间戳、随机 ID。
- 监控缓存命中率与节省金额。

# 9. 成本监控看板

结论先行：看板要能回答"谁在花钱、花在哪个模型、趋势如何"。

## 9.1 看板布局

```
┌────────────────────────────────────────────┐
│ 今日总成本 / 环比 / 预算使用率              │
├────────────────────────────────────────────┤
│ Top 10 租户成本                             │
├────────────────────────────────────────────┤
│ 按模型成本占比                              │
├────────────────────────────────────────────┤
│ Token 趋势（输入/输出）                      │
├────────────────────────────────────────────┤
│ 单任务 Token 分布（P50/P95/P99）             │
└────────────────────────────────────────────┘
```

## 9.2 关键指标

| 指标 | 含义 |
|------|------|
| 总成本 | 日/周/月累计 |
| 预算使用率 | 已用 / 预算 |
| 单请求成本 | 均值与 P99 |
| 缓存命中率 | 缓存节省比例 |

## 9.3 告警联动

- 使用率超 80% 预警、超 100% 拦截。
- 环比突增触发排查。
- 单请求成本异常触发采样分析。

## 9.4 生产建议

- 看板与告警同源，避免口径不一致。
- 支持按租户下钻定位。
- 保留历史数据用于趋势分析。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 无成本采集 | Token/成本一等指标 |
| 定价硬编码 | 配置化 |
| 无预算 | 按租户设预算 |
| 无告警 | 超预算告警 |
| 只监控不降级 | 超限自动降级 |
| 上下文无节制 | 预算 + 裁剪 |

## 10.2 降级策略

```java
// 伪代码：超预算时逐级降级
if (budgetExceeded(tenant)) {
    model = "gpt-4o-mini";
    maxTokens = Math.min(maxTokens, 512);
    context = context.substring(0, CONTEXT_HARD_LIMIT);
}
```

## 10.3 对账

- 定期将内部统计与厂商账单比对。
- 差异超过阈值触发告警。
- 记录对账差异原因（缓存、批量、失败重试）。

# 11. 小结

## 11.1 要点回顾

- Token 决定成本、延迟、容量与限流。
- 用 jtokkit 精确计算，或粗估快速判断。
- 成本必须采集、归因、预算化。
- 降本优先：模型分层、缓存、上下文裁剪。
- 监控要与降级联动。

## 11.2 落地清单

- Token 与成本作为一等指标采集。
- 定价配置化并支持历史回溯。
- 按租户、用户、单请求设预算。
- 稳定前缀前置以提升缓存命中。
- 超预算自动降级并对账。
