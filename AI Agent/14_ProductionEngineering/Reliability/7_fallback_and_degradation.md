# 1. 降级的目标与定位

**结论先行**：降级的核心目标不是"永远成功"，而是"永远有响应"——当依赖失效时，系统仍能给出一个可用、可信或明确的答复，而不是挂起、超时或崩溃。

## 1.1 什么是降级

降级（Degradation）指当系统依赖（模型、检索、工具、下游服务）不可用或资源紧张时，主动切换到一个功能较弱但仍可用的替代路径，以保证整体可用性。它牺牲的是"最好结果"，保住的是"有结果"。

### 1.1.1 降级与容错的边界

容错（Fault Tolerance）是更宽的概念，包含重试（Retry）、熔断（Circuit Breaker）、隔离（Bulkhead）、降级（Fallback）等。降级只是其中"降低功能以保可用"的一环，不能替代其他手段。

| 手段 | 解决的问题 | 是否降低功能 | 典型实现 |
|------|------------|--------------|----------|
| 重试 | 瞬时抖动 | 否 | Resilience4j Retry |
| 熔断 | 依赖持续故障 | 否（快速失败） | Resilience4j CircuitBreaker |
| 隔离 | 故障扩散 | 否 | 线程池/信号量舱壁 |
| 超时 | 慢依赖拖垮 | 否 | TimeLimiter |
| 降级 | 依赖不可用时的替代 | 是 | Fallback 链 |

### 1.1.2 降级与熔断、限流的关系

熔断负责"发现依赖坏了就别再打"，限流（Rate Limiting）负责"别把依赖打垮"，降级负责"打不了怎么办"。三者串联在同一个调用链上。

```
请求 → 限流(放过?) → 熔断(允许调用?) → 调用依赖
                                          │ 失败
                                          ▼
                                       降级路径
```

## 1.2 核心原则

### 1.2.1 永远有响应

无论依赖如何失效，用户都应在可接受时间内收到某种响应。这是可用性（Availability）的底线。

```
正常：主模型 + 完整 RAG + 全部工具
降级：备用模型 / 缓存结果 / 简化流程 / 明确失败
```

### 1.2.2 宁可明确失败不可编造

> **宁可明确失败，不可编造答案。**

降级绝不能以牺牲正确性为代价。当无法给出可信答案时，应明确告知失败或转人工，而不是用低质量模型"编一个看起来对的答案"。

## 1.3 降级的代价

### 1.3.1 质量代价

每一级降级都伴随质量下降，必须量化：备用模型准确率、缓存陈旧率、无 RAG 的幻觉率。没有量化的降级是盲降。

| 级别 | 质量影响 | 可接受度 |
|------|----------|----------|
| 备用模型 | 准确率降 5~15% | 一般问答可接受 |
| 缓存 | 可能过期 | 仅时效性弱的问题 |
| 无 RAG | 幻觉率上升 | 需明确提示 |

### 1.3.2 复杂度代价

降级路径也是代码，也需要测试与维护。降级链越长，分支越多，出 bug 的概率越高。建议把降级逻辑抽成独立组件，而非散落在业务代码里的 try-catch。

# 2. 降级触发条件

**结论先行**：降级必须由明确的信号触发，而非拍脑袋；触发条件要可观测、可配置、可回放。

## 2.1 触发信号分类

### 2.1.1 依赖类触发

依赖自身返回错误或超时。

| 触发 | 说明 |
|------|------|
| 模型不可用 | 超时、熔断、限流 |
| 检索不可用 | 向量库故障 |
| 工具不可用 | 下游故障 |
| 认证失效 | API Key 过期 |

### 2.1.2 资源类触发

系统自身资源或配额受限。

| 触发 | 说明 |
|------|------|
| 并发已满 | 舱壁（Bulkhead）拒绝 |
| 成本超预算 | 预算耗尽 |
| 延迟超阈值 | 保证响应时间 |
| 队列积压 | 背压（Backpressure）触发 |

### 2.1.3 业务类触发

业务规则要求切换路径，例如低优先级租户在高峰期降级、灰度期间仅对部分流量启用新模型。

## 2.2 触发判定机制

### 2.2.1 熔断器状态

熔断器（Circuit Breaker）是最常用的自动触发源，其状态机为 CLOSED → OPEN → HALF_OPEN。

```
CLOSED ──错误率超阈──▶ OPEN ──等待窗口──▶ HALF_OPEN
   ▲                                          │
   └──────────── 探测成功 ◀───────────────────┘
```

### 2.2.2 阈值与迟滞

阈值（Threshold）应带迟滞（Hysteresis），避免在临界点反复抖动。建议错误率阈值 50%、最小样本数 20、OPEN 保持 30 秒。

```yaml
resilience4j:
  circuitbreaker:
    instances:
      primaryModel:
        slidingWindowSize: 20
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
```

## 2.3 手动与自动触发

### 2.3.1 自动触发

由熔断、限流、超时、预算等运行时指标自动判定，响应快、无人工介入。

### 2.3.2 人工开关

运维可通过配置中心强制切换降级，用于已知故障（如厂商公告故障）时提前降级，避免真实流量受损。

```java
if (configCenter.getBoolean("ai.forceFallback", false)) {
    return fallbackModel.answer(question, user);
}
```

# 3. 降级链设计

**结论先行**：降级不是一个点，而是一条"链"——从最强能力逐级退到最弱兜底，每一级都应有明确的进入条件与退出提示。

## 3.1 五级降级链

### 3.1.1 逐级降级流程

```
① 主模型
   ↓ 失败
② 备用模型（其他厂商/小模型）
   ↓ 失败
③ 缓存结果（历史相同问题）
   ↓ 无缓存
④ 简化流程（跳过 RAG/工具，直接回答）
   ↓ 失败
⑤ 静态兜底（"服务繁忙，请稍后重试"）
```

### 3.1.2 完整实现

```java
public String answerWithFallback(String question, UserContext user) {
    // ① 主模型
    try {
        return primaryModel.answer(question, user);
    } catch (Exception e) {
        log.warn("主模型失败，降级", e);
    }

    // ② 备用模型
    try {
        return fallbackModel.answer(question, user);
    } catch (Exception e) {
        log.warn("备用模型失败，降级到缓存", e);
    }

    // ③ 缓存
    String cached = cache.get(cacheKey(question, user));
    if (cached != null) {
        return cached + "\n\n（注：当前为缓存结果）";
    }

    // ④ 静态兜底
    return "服务当前繁忙，请稍后重试。";
}
```

## 3.2 降级链的边界

### 3.2.1 链长控制

降级链过长会累积延迟。一般控制在 3~4 级；每级都设独立超时，避免"降级反而更慢"。

### 3.2.2 短路与快速失败

对高风险链路，某些级应"短路"直接失败，而不是继续降级。可用策略对象显式声明允许的降级级别。

```java
Set<FallbackLevel> allowed = switch (riskLevel) {
    case HIGH -> Set.of();                         // 不允许降级
    case MEDIUM -> Set.of(FallbackLevel.MODEL);    // 仅换模型
    case LOW -> Set.of(FallbackLevel.values());    // 全部允许
};
```

## 3.3 降级链的抽象

### 3.3.1 策略接口

把每一级抽象为统一接口，便于组合与测试。

```java
public interface FallbackStrategy {
    FallbackLevel level();
    boolean supports(Request req);
    String execute(Request req) throws Exception;
}
```

### 3.3.2 组合器

```java
public class FallbackChain {
    private final List<FallbackStrategy> strategies;

    public String execute(Request req) {
        for (FallbackStrategy s : strategies) {
            if (!s.supports(req)) continue;
            try {
                return s.execute(req);
            } catch (Exception e) {
                meterRegistry.counter("ai.degradation",
                        "level", s.level().name()).increment();
            }
        }
        throw new AllFallbacksFailedException(req);
    }
}
```

# 4. 各类降级策略

**结论先行**：不同依赖失效需要不同降级策略，核心是"替换能力"而非"忽略错误"。

## 4.1 模型降级

### 4.1.1 模型映射

| 降级 | 影响 |
|------|------|
| 大模型 → 小模型 | 质量略降，可用性保 |
| 云端 → 本地 | 质量降，成本降 |
| 多模态 → 纯文本 | 丢失图像能力 |
| 强推理 → 快模型 | 复杂任务质量降 |

### 4.1.2 实现与配置

```java
public String chat(String prompt) {
    if (primaryBreaker.isOpen()) {
        return smallModel.chat(prompt);        // 换小模型
    }
    return primaryModel.chat(prompt);
}
```

```yaml
llm:
  primary: gpt-4o
  fallback:
    - qwen-max
    - local-llama-8b
  perCallTimeout: 20s
  maxFallbackLevels: 2
```

## 4.2 检索降级

### 4.2.1 无检索回退

RAG（Retrieval-Augmented Generation，检索增强生成）失效时，可退回无检索的纯模型回答，但必须明确告知用户。

```java
public String ragAnswer(String question) {
    try {
        List<Document> docs = retriever.retrieve(question);
        return generateWithContext(question, docs);
    } catch (Exception e) {
        // 检索不可用 → 不用 RAG，但要明确告知
        return "知识库暂时不可用，以下回答未基于内部资料：\n"
                + model.chat(question);
    }
}
```

> RAG 降级时必须**明确告知用户**回答未基于资料，避免误导。

### 4.2.2 降级检索

也可换用较弱但更稳的检索通道，例如从向量检索退回关键词（BM25）检索，仍保留引用。

| 主检索 | 降级检索 | 影响 |
|--------|----------|------|
| 向量 + 重排 | 仅向量 | 精度降 |
| 向量 | BM25 关键词 | 召回降 |
| 多路融合 | 单路 | 覆盖降 |

## 4.3 工具降级

### 4.3.1 缓存回退

工具调用失败时，可退回缓存值（标注陈旧）或明确返回失败。

```java
@Tool("查询实时库存")
ToolResult queryStock(@P("商品编码") String sku) {
    try {
        return ToolResult.ok(stockApi.query(sku));
    } catch (Exception e) {
        // 降级到缓存（可能不是最新）
        Integer cached = cache.get(sku);
        if (cached != null) {
            return ToolResult.ok(cached + "（缓存数据，可能不是最新）");
        }
        return ToolResult.fail("库存服务暂时不可用");
    }
}
```

### 4.3.2 只读降级

写操作不可降级；读操作可降级。对写工具失败，必须明确失败并禁止 Agent 继续假设成功。

## 4.4 流程降级

复杂 Agentic 流程失败时，退回单轮或传统流程。

```java
public String handleComplex(String question) {
    try {
        return agenticRag(question);           // 复杂流程
    } catch (Exception e) {
        return traditionalRag(question);        // 简化流程
    }
}
```

# 5. 缓存兜底

**结论先行**：热点问题的缓存命中率很高，是性价比最高的兜底手段；但缓存结果必须标注来源与时效。

## 5.1 精确缓存与语义缓存

| 类型 | 匹配方式 | 命中率 | 风险 |
|------|----------|--------|------|
| 精确缓存 | 问题文本哈希 | 低 | 低 |
| 语义缓存 | 向量相似度 | 高 | 可能答非所问 |

```java
public String answerWithCache(String question) {
    String key = normalize(question);

    // 语义缓存：找相似的历史问题
    String similar = semanticCache.findSimilar(key, 0.95);
    if (similar != null) {
        return semanticCache.get(similar);
    }
    return model.chat(question);
}
```

## 5.2 缓存兜底的坑

> 热点问题（"如何报销"）的缓存命中率可能很高，是有效的兜底。

- 相似度阈值过低会把不同问题判为相同，导致答非所问；
- 缓存可能包含已过期信息（价格、政策），必须标注时效；
- 涉及个人数据时，缓存键必须包含租户/用户维度，防止越权。

## 5.3 缓存的淘汰与刷新

### 5.3.1 TTL 与主动失效

缓存必须设 TTL（Time To Live，存活时间），并在数据源变更时主动失效。

```java
public void onPolicyUpdated(String policyId) {
    semanticCache.invalidateByTag("policy:" + policyId);
}
```

# 6. 优雅降级 vs 快速失败

**结论先行**：是否降级取决于"错误代价"——用户体验优先则优雅降级，正确性优先则快速失败。

## 6.1 决策矩阵

| 策略 | 适用 | 说明 |
|------|------|------|
| 优雅降级 | 用户体验优先 | 尽力给出可用结果 |
| 快速失败 | 正确性优先 | 高风险场景不将就 |

| 维度 | 优雅降级 | 快速失败 |
|------|----------|----------|
| 可用性 | 高 | 低 |
| 正确性 | 可能下降 | 保持 |
| 用户感知 | 有结果 | 明确错误 |
| 适用场景 | 一般问答 | 金融/医疗/交易 |

## 6.2 高风险场景

```java
// 高风险场景：宁可失败
public String financialAdvice(String question) {
    if (primaryModelUnavailable()) {
        throw new ServiceUnavailableException("金融建议服务暂不可用，请稍后重试");
    }
    return primaryModel.chat(question);
}
```

## 6.3 混合策略

### 6.3.1 按请求分级

同一系统可对不同请求采用不同策略：一般问答优雅降级，交易类快速失败。

| 请求类型 | 策略 | 理由 |
|----------|------|------|
| 知识问答 | 优雅降级 | 有结果优于无结果 |
| 内容生成 | 优雅降级 | 可人工复核 |
| 资金操作 | 快速失败 | 错误代价高 |
| 医疗建议 | 快速失败 | 合规要求 |

### 6.3.2 策略路由

```java
public String handle(Request req) {
    RiskLevel risk = riskClassifier.classify(req);
    return risk == RiskLevel.HIGH
            ? failFast(req)
            : fallbackChain.execute(req);
}
```

# 7. 降级的可见性

**结论先行**：不可观测的降级等于没有降级——它会让系统"看起来正常"，实则质量持续下滑。

## 7.1 指标与日志

降级必须**可观测**：

```java
meterRegistry.counter("ai.degradation",
        "level", level,          // MODEL / RAG / TOOL / STATIC
        "reason", reason).increment();

log.warn("触发降级 level={} reason={} traceId={}",
        level, reason, MDC.get("traceId"));
```

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| ai.degradation{level} | 各级降级次数 | 5 分钟 > 100 |
| ai.fallback.latency | 降级链耗时 | P99 > 3s |
| ai.static.fallback | 静态兜底占比 | > 5% |

## 7.2 用户提示

对用户也应**适度提示**：

```
"（当前为缓存结果）"
"（知识库暂时不可用）"
"（服务繁忙，已简化处理）"
```

## 7.3 告警

当静态兜底或不可补偿的降级占比超过阈值时，应立即告警，因为这通常意味着主依赖已整体失效。

# 8. 降级与业务语义

**结论先行**：降级可能悄悄改变业务语义，这是最危险的坑——代码"成功"了，业务却错了。

## 8.1 语义改变风险

| 场景 | 降级风险 |
|------|----------|
| 金融建议 | 质量下降可能误导 |
| 医疗建议 | 必须拒答而非降级 |
| 订单操作 | 不能降级到"假设成功" |
| 一般问答 | 可降级到通用回答 |

## 8.2 高风险领域策略

> **高风险领域：降级 = 明确失败 + 转人工**，而不是给一个"差不多"的答案。

```
高风险请求 ──主链路失败──▶ 明确失败
                          └─▶ 转人工队列
一般请求   ──主链路失败──▶ 降级链 ──▶ 有响应的可用结果
```

# 9. 生产注意事项

**结论先行**：降级是一套需要设计、演练、监控的工程能力，而不是零散的 try-catch。

## 9.1 配置与演练

- 降级链、阈值、文案应配置化，便于热更新；
- 定期做故障演练（Chaos），验证降级真的生效；
- 降级文案需产品/法务审核，避免误导；
- 为降级路径编写与主链路同级的单元测试。

## 9.2 风险与反例

| 风险 | 建议 |
|------|------|
| 降级无提示 | 明确告知用户 |
| 降级静默 | 记录指标与日志 |
| 降级改变语义 | 高风险场景不降级 |
| 无兜底 | 必须有静态兜底 |
| 缓存数据陈旧 | 标注"缓存数据" |
| 降级链过长 | 控制在 3~4 级 |

反例：降级代码自身抛异常导致请求 500；降级路径未设超时，比主链路还慢；降级后未记录，故障期间无人察觉；把写操作降级为"假设成功"。

## 9.3 上线检查清单

| 检查项 | 要求 |
|--------|------|
| 每级超时 | 已设置且小于上游超时 |
| 静态兜底 | 存在且文案已审核 |
| 指标埋点 | 各级降级均有计数 |
| 告警规则 | 静态兜底占比已配置 |
| 故障演练 | 每季度至少一次 |
| 高风险链路 | 已确认禁止降级 |

# 10. 小结

- 降级目标是"永远有响应"，但宁可明确失败不可编造。
- 降级链：主模型 → 备用模型 → 缓存 → 简化流程 → 静态兜底。
- RAG/工具降级必须明确告知用户。
- 降级必须可观测（指标 + 日志）。
- 高风险场景用快速失败而非优雅降级。
