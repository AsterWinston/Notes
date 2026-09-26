# 1. 为什么必须监控

**结论先行**：LLM 应用的成本是按量计费乘以不可预测的用量，用量又由模型、上下文、检索与用户行为共同决定。没有监控，账单会在无人察觉时暴涨，且无法归因到具体租户或功能。

```
LLM 成本 = 按量计费 × 不可预测的用量
不监控的后果：一夜之间账单暴涨，且不知道钱花在哪
```

## 1.1 成本结构

一次调用的成本由输入 Token、输出 Token 与模型单价决定。其中输入 Token 往往被低估：系统提示、历史消息、检索片段都会反复计入每一次调用。

| 成本来源 | 是否可预测 | 优化手段 |
|----------|:----------:|----------|
| 系统提示 | 高 | 精简 |
| 历史消息 | 中 | 截断/摘要 |
| 检索片段 | 中 | 控制条数与长度 |
| 输出长度 | 低 | 限制 max_tokens |
| 模型单价 | 高 | 分级选型 |

## 1.2 典型风险

| 风险 | 说明 |
|------|------|
| 用量突增 | 恶意刷接口、死循环 |
| 模型误用 | 简单任务用大模型 |
| 上下文膨胀 | 历史与检索无节制 |
| 无法归因 | 不知道哪个租户/功能消耗大 |

## 1.3 监控目标

| 目标 | 对应能力 |
|------|----------|
| 看得见 | 采集 Token 与成本 |
| 分得清 | 按维度归因 |
| 管得住 | 预算与配额 |
| 反应快 | 告警与自动降级 |

# 2. 成本计算

## 2.1 定价模型

### 2.1.1 缓存与推理价格

部分厂商对缓存命中输入、批量推理给出折扣，定价不能只有"输入/输出"两档。若使用缓存，应按命中与未命中分别计价。

### 2.1.2 定价注册表

```java
public class PricingRegistry {

    private static final Map<String, ModelPricing> PRICING = Map.of(
            "gpt-4o-mini",   new ModelPricing(of("0.15"), of("0.60")),
            "gpt-4o",        new ModelPricing(of("2.50"), of("10.00")),
            "qwen-plus",     new ModelPricing(of("0.80"), of("2.00"))
    );

    public BigDecimal cost(String model, long input, long output) {
        ModelPricing pricing = PRICING.get(model);
        if (pricing == null) {
            log.warn("未知模型定价: {}", model);
            return BigDecimal.ZERO;
        }
        return pricing.cost(input, output);
    }
}
```

> 定价需**定期更新**（厂商调价频繁），建议配置化。

## 2.2 计算实现

```java
public record ModelPricing(
        BigDecimal inputPricePerMillion,
        BigDecimal outputPricePerMillion
) {
    public BigDecimal cost(long inputTokens, long outputTokens) {
        return inputPricePerMillion
                .multiply(BigDecimal.valueOf(inputTokens))
                .add(outputPricePerMillion
                        .multiply(BigDecimal.valueOf(outputTokens)))
                .divide(BigDecimal.valueOf(1_000_000), 6, RoundingMode.HALF_UP);
    }
}
```

单价以"每百万 Token"计价，因此需除以 1,000,000。使用 `BigDecimal` 避免浮点误差累积。

## 2.3 边界与坑

| 坑 | 后果 | 规避 |
|----|------|------|
| 用 double 累加 | 金额漂移 | 用 BigDecimal |
| 未知模型返回 0 | 成本被低估 | 告警并兜底 |
| 单价硬编码 | 调价后失准 | 配置化 |
| 忽略缓存价 | 成本偏差 | 分档计价 |

# 3. 成本采集

## 3.1 采集点

成本应在"拿到 Usage 之后、返回给调用方之前"采集。Spring AI 通过 Advisor 提供了统一的拦截点，可覆盖所有模型调用。

## 3.2 Advisor 实现

### 3.2.1 拦截链

Advisor 按 `getOrder()` 排序组成链，请求依次经过，响应逆序返回。成本采集通常放在靠后的位置，确保能拿到最终 Usage。

### 3.2.2 完整代码

```java
public class CostTrackingAdvisor implements CallAdvisor {

    private final MeterRegistry registry;
    private final PricingRegistry pricing;
    private final CostStore costStore;

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);

        Usage usage = response.chatResponse().getMetadata().getUsage();
        if (usage == null) return response;

        String model = extractModel(response);
        String tenant = TenantContext.current();
        String userId = UserContextHolder.current().userId();

        BigDecimal cost = pricing.cost(model,
                usage.getPromptTokens(), usage.getCompletionTokens());

        // ① 指标
        registry.counter("ai.cost.total", "model", model, "tenant", tenant)
                .increment(cost.doubleValue());
        registry.summary("ai.tokens.input", "model", model)
                .record(usage.getPromptTokens());
        registry.summary("ai.tokens.output", "model", model)
                .record(usage.getCompletionTokens());

        // ② 持久化（用于计费与归因）
        costStore.record(new CostRecord(
                tenant, userId, model,
                usage.getPromptTokens(), usage.getCompletionTokens(),
                cost, Instant.now(), MDC.get("traceId")));

        // ③ 预算检查
        if (costStore.dailyCost(tenant)
                .compareTo(budget(tenant)) > 0) {
            alertService.warn("租户成本超预算: " + tenant);
        }

        return response;
    }

    @Override
    public String getName() { return "CostTrackingAdvisor"; }

    @Override
    public int getOrder() { return 210; }
}
```

## 3.3 流式场景

流式响应（Streaming）的 Usage 常在最后一个 chunk 才给出。此时应在流结束时采集，且不能让采集逻辑阻塞数据流。

```java
// 伪代码：流结束时采集
return chatClient.prompt()
        .stream()
        .doOnComplete(() -> collector.flush())
        .doOnError(e -> collector.markError());
```

## 3.4 采集的边界与坑

| 坑 | 后果 | 规避 |
|----|------|------|
| 中途异常无 Usage | 漏记 | 失败也记 0 成本 |
| 多模态计价不同 | 低估 | 按模态分档 |
| 重试重复计费 | 虚高 | traceId 去重 |
| 采集阻塞响应 | 延迟上升 | 异步落库 |

# 4. 成本归因维度

## 4.1 维度清单

| 维度 | 用途 |
|------|------|
| 租户 | 计费、配额 |
| 用户 | 防滥用 |
| 模型 | 选型决策 |
| 接口/功能 | 定位高消耗功能 |
| 会话/任务 | 单次成本分析 |
| 时间 | 趋势与异常 |

## 4.2 聚合查询

```sql
-- 按租户统计日成本
SELECT tenant_id, DATE(ts) AS day, SUM(cost) AS total_cost
FROM cost_record
WHERE ts >= now() - INTERVAL '7 days'
GROUP BY tenant_id, DATE(ts)
ORDER BY day DESC, total_cost DESC;
```

## 4.3 归因陷阱

| 陷阱 | 后果 | 规避 |
|------|------|------|
| 维度缺失 | 无法归因 | 强制传租户 |
| 维度高基数 | 查询慢 | 预聚合 |
| 时钟不一致 | 跨日错位 | 统一 UTC |
| 重复计数 | 成本虚高 | 幂等写入 |

## 4.4 归因示例

```java
// 伪代码：一次请求写入多条归因记录
recordCost("tenant", tenant, cost);
recordCost("user", userId, cost);
recordCost("feature", feature, cost);
```

同一笔成本可写入多个维度，查询时按需聚合。明细表按天分区，聚合表按维度预计算。

# 5. Token 监控

## 5.1 指标设计

```java
// 单任务 Token 异常
registry.summary("agent.tokens.per.task")
        .record(taskTokens);
```

## 5.2 异常信号

| 指标 | 异常信号 |
|------|----------|
| 单任务 Token 突增 | 循环或上下文膨胀 |
| 输入/输出比下降 | 检索或历史过多 |
| 平均 Token/请求上升 | 提示变长 |

## 5.3 输入输出比

```java
// 输入输出比异常（提示膨胀信号）
double ratio = outputTokens / (double) inputTokens;
if (ratio < 0.05) {
    log.warn("输入远大于输出，可能提示膨胀: ratio={}", ratio);
}
```

比值本身不是绝对值指标，应结合业务类型判断：摘要类任务输出本就短，不能一概而论。

## 5.4 与成本的联动

Token 指标是成本的先行信号：在定价已知时，Token 突增往往先于成本突增被观测到。因此可对 Token 设置更敏感的告警阈值，作为成本的早期预警。

# 6. 预算与配额

## 6.1 预算模型

| 层级 | 周期 | 动作 |
|------|------|------|
| 平台 | 日/月 | 总量预警 |
| 租户 | 日/月 | 超限拒绝或降级 |
| 用户 | 日 | 限流 |
| 会话 | 单次 | 上限截断 |

## 6.2 前置拦截

### 6.2.1 预估

在调用前用输入长度与期望输出预估成本，作为准入判断依据。

### 6.2.2 拒绝与排队

```java
public class BudgetEnforcer {

    public void checkBefore(String tenantId, int estimatedTokens) {
        BigDecimal estimatedCost = pricing.estimateCost(tenantId, estimatedTokens);
        BigDecimal used = costStore.dailyCost(tenantId);
        BigDecimal budget = budgetService.dailyBudget(tenantId);

        if (used.add(estimatedCost).compareTo(budget) > 0) {
            throw new BudgetExceededException(
                    "租户日预算已用尽: %s/%s".formatted(used, budget));
        }

        // 预警
        if (used.compareTo(budget.multiply(of("0.8"))) > 0) {
            alertService.warn("租户成本接近预算: " + tenantId);
        }
    }
}
```

## 6.3 配额维度

配额可按请求数、Token 数或金额设置。金额配额最直观，但需要预估；Token 配额更稳定，适合做限流。

## 6.4 分级预算

| 场景 | 预算策略 |
|------|----------|
| 免费用户 | 低日额度，超限排队 |
| 付费用户 | 按套餐额度 |
| 内部服务 | 高额度，仅告警 |
| 压测环境 | 独立额度，隔离 |

预算策略应可配置、可热更新，避免为调额而发版。

# 7. 告警规则

## 7.1 规则清单

| 告警 | 条件 | 级别 |
|------|------|------|
| 租户日成本超预算 | > 100% | P1 |
| 租户成本接近预算 | > 80% | P2 |
| 单任务 Token 异常 | > 100k | P2 |
| 全局日成本突增 | 环比 > 50% | P1 |
| 单用户高频调用 | > 阈值 | P2 |

## 7.2 Prometheus 配置

```yaml
- alert: TenantBudgetExceeded
  expr: sum by (tenant) (increase(ai_cost_total[1d])) > 100
  labels: { severity: critical }
```

### 7.2.1 表达式要点

`increase` 需搭配足够长的窗口与稳定的抓取间隔；成本计数器若被重置（应用重启），`increase` 会自动处理重置，无需额外处理。

## 7.3 降噪

成本告警容易在月初或促销期误报。建议用滑动窗口、设置 `for` 持续时间，并对已知活动加静默（Silence）。

# 8. 成本看板

## 8.1 布局

```
┌────────────────────────────────────────────┐
│ 今日总成本 / 环比 / 预算使用率              │
├────────────────────────────────────────────┤
│ 成本趋势（按天）                            │
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

## 8.2 关键指标

| 指标 | 回答的问题 |
|------|-----------|
| 预算使用率 | 还能撑多久 |
| 环比 | 是否异常 |
| Top 租户 | 谁在消耗 |
| 模型占比 | 选型是否合理 |

## 8.3 看板实现

看板应区分"实时"与"对账"两类视图：实时视图用于发现异常，容忍少量延迟；对账视图用于计费，要求与明细一致。两者数据源可不同，但口径必须统一。

# 9. 降本联动

## 9.1 自动降级

监控发现异常后，自动或手动触发：

```java
// 成本超预算 → 自动降级
if (costStore.dailyCost(tenant).compareTo(hardLimit) > 0) {
    // 切换到小模型
    tenantConfigService.setModelTier(tenant, Tier.FAST);
    alertService.warn("租户已自动降级到小模型: " + tenant);
}
```

## 9.2 缓存与裁剪

| 手段 | 收益 | 风险 |
|------|------|------|
| 语义缓存 | 高 | 命中错误 |
| 提示精简 | 中 | 效果下降 |
| 检索裁剪 | 中 | 召回下降 |
| 结果复用 | 中 | 时效性 |

## 9.3 闭环

降级动作本身也要被监控，否则可能长期停留在大模型或小模型而无人发现。建议对"降级状态"设指标与看板。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 无成本采集 | Token/成本一等指标 |
| 定价未更新 | 配置化 + 定期更新 |
| 无归因维度 | 按租户/用户/模型 |
| 无预算 | 按租户设预算 |
| 无告警 | 超预算与突增告警 |
| 只监控不联动 | 超限自动降级 |

## 10.2 数据一致性

成本记录应在事务内或幂等写入，避免重试导致重复计费。以请求 ID 或 traceId 作为去重键。

### 10.2.1 去重实现

```sql
-- 以 request_id 唯一约束实现幂等写入
INSERT INTO cost_record(request_id, tenant_id, cost, ts)
VALUES (?, ?, ?, ?)
ON CONFLICT (request_id) DO NOTHING;
```

## 10.3 治理

| 事项 | 做法 |
|------|------|
| 定价维护 | 配置中心 + 变更审计 |
| 数据保留 | 明细保留 90 天，聚合长期 |
| 对账 | 与厂商账单定期核对 |

# 11. 小结

- 成本监控是 LLM 应用的一等需求。
- 采集 Token + 计算成本 + 按维度归因。
- 预算化：预估、预警、超限拒绝或降级。
- 关注单任务 Token 与输入输出比等异常信号。
- 监控要与降级联动，形成闭环。
- 定价配置化、写入幂等、账单定期对账，是长期稳定的保障。
