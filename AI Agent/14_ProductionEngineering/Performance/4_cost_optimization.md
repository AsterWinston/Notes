# 1. 成本构成

**结论先行**：LLM 应用的总成本由"模型调用 + 检索链路 + 基础设施"三部分组成，其中模型调用是绝对大头，优化重点必须放在这里。

## 1.1 成本分解模型

```
总成本 = 模型调用 + 向量化 + 重排 + 向量库 + 存储 + 基础设施
```

### 1.1.1 模型调用成本

模型调用又分输入（prompt）与输出（completion），两者单价不同，输出通常更贵。

```java
public record ModelPrice(BigDecimal inputPer1M, BigDecimal outputPer1M) {

    public BigDecimal calculate(int inputTokens, int outputTokens) {
        BigDecimal in = inputPer1M
                .multiply(BigDecimal.valueOf(inputTokens))
                .divide(BigDecimal.valueOf(1_000_000));
        BigDecimal out = outputPer1M
                .multiply(BigDecimal.valueOf(outputTokens))
                .divide(BigDecimal.valueOf(1_000_000));
        return in.add(out);
    }
}
```

### 1.1.2 检索链路成本

检索链路包含向量化（embedding）、重排（rerank）、向量库查询与存储，虽单项占比不高，但调用频繁，累积可观。

## 1.2 成本占比与优化空间

| 项 | 占比（典型） | 优化空间 |
|----|:------------:|:--------:|
| 模型调用（输入） | 30%~50% | 大 |
| 模型调用（输出） | 20%~40% | 大 |
| 向量化 | 5%~15% | 中 |
| 重排 | 5%~10% | 中 |
| 向量库/存储 | 5%~10% | 中 |
| 基础设施 | 5%~10% | 小 |

> **模型调用是大头**，优化重点在此。

## 1.3 单位经济性

**结论先行**：技术成本必须折算成"每请求成本 / 每用户成本 / 每转化成本"，才能判断是否可持续。

| 指标 | 公式 |
|------|------|
| 每请求成本 | 总成本 / 请求数 |
| 每活跃用户成本 | 总成本 / DAU |
| 每转化成本 | 总成本 / 成交数 |
| 毛利率 | (收入 - 成本) / 收入 |

> 生产建议：任何优化都要能回答"它把单位经济性改善了多少"。

---

# 2. 优化手段总览

**结论先行**：优化手段众多，应按"收益/代价比"排序，先做收益大、风险低的手段。

## 2.1 手段矩阵

| 手段 | 节省 | 代价 |
|------|:----:|------|
| 模型分层路由 | 30%~70% | 实现复杂度 |
| 缓存（结果/语义） | 10%~40% | 一致性风险 |
| Prompt 缓存 | 20%~50%（输入） | 需设计提示 |
| 上下文裁剪 | 10%~30% | 可能丢信息 |
| 批处理 | 10%~30% | 延迟增加 |
| 本地 Embedding | 显著 | 运维成本 |
| 减少重排候选 | 中 | 精度略降 |
| 输出长度限制 | 中 | 完整性 |

## 2.2 收益与代价权衡

```
高收益 / 低代价  →  立即做（路由、Prompt 缓存）
高收益 / 高代价  →  规划做（本地 Embedding）
低收益 / 低代价  →  顺手做（输出限制）
低收益 / 高代价  →  不做
```

## 2.3 优化原则

- 先可见，后优化：没有成本数据不做决策。
- 先质量，后成本：任何优化都要过评测。
- 先高收益，后细枝末节。

---

# 3. 模型分层路由（收益最大）

**结论先行**：把简单任务交给小模型、复杂任务交给大模型，是性价比最高的优化手段。

## 3.1 路由决策

### 3.1.1 规则分类

```java
public enum TaskComplexity { SIMPLE, STANDARD, COMPLEX }

// 用规则先过滤（零成本）
public TaskComplexity classify(String q) {
    if (q.length() < 20 && isSimpleIntent(q)) {
        return TaskComplexity.SIMPLE;
    }
    if (containsComplexMarkers(q)) {
        return TaskComplexity.COMPLEX;
    }
    return TaskComplexity.STANDARD;
}
```

### 3.1.2 小模型分类

规则无法判定时，用便宜的小模型做意图/难度分类。

```java
// 伪代码：用小模型输出枚举，成本极低
TaskComplexity c = smallModel.classify(
        "判断难度，仅输出 SIMPLE/STANDARD/COMPLEX：" + q);
```

### 3.1.3 置信度回退

当分类置信度低或小模型回答质量差时，回退到大模型。

```java
String answer = small.chat(q);
if (lowConfidence(answer) || judge(answer)) {
    answer = large.chat(q);   // 回退
}
```

## 3.2 路由实现

```java
public class ModelRouter {

    private final ChatModel small;    // 便宜
    private final ChatModel medium;
    private final ChatModel large;    // 贵

    public String route(String question) {
        TaskComplexity c = classifier.classify(question);

        return switch (c) {
            case SIMPLE   -> small.chat(question);
            case STANDARD -> medium.chat(question);
            case COMPLEX  -> large.chat(question);
        };
    }
}
```

| 任务 | 模型 |
|------|------|
| 分类、抽取、改写 | 小模型 |
| 常规问答、RAG 生成 | 中模型 |
| 复杂推理、代码 | 大模型 |

## 3.3 路由的坑

- 误判复杂任务为简单任务，会导致答案质量崩塌。
- 分类本身也有成本，需与节省相抵。
- 路由规则要有线上可观测的命中分布。

> 先用规则（零成本），规则判断不了的再用小模型分类。

---

# 4. 缓存

**结论先行**：缓存是最直接的省钱手段，但必须接受"可能返回过期内容"的一致性代价。

## 4.1 缓存类型

```java
// 结果缓存：热点问题直接返回
// 语义缓存：相似问题复用
// 检索缓存：相同查询复用检索
// 向量缓存：相同文本不重复向量化
```

| 类型 | 命中条件 | 收益 |
|------|----------|------|
| 结果缓存 | 完全相同 | 最高 |
| 语义缓存 | 语义相似 | 高 |
| 检索缓存 | 相同查询 | 中 |
| 向量缓存 | 相同文本 | 中 |

## 4.2 缓存实现

```java
// 伪代码：结果缓存
String key = hash(normalize(question) + "|" + tenantId);
String cached = cache.get(key);
if (cached != null) {
    return cached;
}
String answer = model.chat(question);
cache.put(key, answer, Duration.ofHours(1));
return answer;
```

## 4.3 一致性风险

- 知识库更新后，缓存可能返回旧答案。
- 语义缓存阈值过松会"张冠李戴"。
- 缓存键必须包含租户、模型、提示版本等维度。

> 生产建议：合理 TTL（存活时间）+ 主动失效 + 版本化缓存键。

---

# 5. Prompt 缓存

**结论先行**：把稳定内容放在提示前缀，能命中厂商侧的 Prompt 缓存，直接降低输入单价。

## 5.1 前缀缓存机制

多数厂商按"最长公共前缀"做缓存，前缀不变的部分按折扣价计费。

```
[稳定前缀：系统提示 + 知识库] ← 命中缓存，折扣计费
[可变后缀：用户问题]          ← 全价
```

## 5.2 提示结构设计

```java
// 稳定内容前置，提升厂商侧 Prompt 缓存命中
String prompt = """
        %s        ← 系统提示（稳定）
        %s        ← 知识库上下文（半稳定）
        %s        ← 用户问题（每次都变）
        """.formatted(systemPrompt, context, question);
```

部分厂商对缓存前缀按 **10%~25%** 价格计费，收益显著。

## 5.3 边界与坑

- 前缀中夹入时间戳、随机数会让缓存永远失效。
- 知识库内容频繁变动会破坏前缀稳定性。
- 不同厂商缓存策略不同，需实测命中率。

> 生产建议：把"最稳定的内容放最前面"，把"每次都变的内容放最后面"。

---

# 6. 上下文裁剪

**结论先行**：放入模型的片段越少，输入 Token 越少，且往往更准。

## 6.1 片段数量控制

```java
// 只放最相关的 3~5 个片段，而不是 10 个
SearchRequest.builder()
        .query(question)
        .topK(20)                  // 召回多
        .build();
// → 重排后取 topN=3
```

| 做法 | 输入 Token |
|------|:----------:|
| 直接放 10 个片段 | 高 |
| 重排后放 3 个 | 低（且更准） |

## 6.2 重排后再裁剪

先多召回（保证覆盖），再重排（保证精度），最后只保留 topN（控制成本）。

```
召回 topK=50 → 重排 → 取 topN=3~5 → 拼入提示
```

## 6.3 质量与成本平衡

- 片段过少会漏关键信息，需评测确认。
- 每个片段长度也要限制，避免单片段占满预算。
- 裁剪策略应与 Token 预算模块共享配置。

> 生产建议：把"片段数"与"每片段长度"都做成可调参数，用评测驱动调优。

---

# 7. 输出控制

**结论先行**：输出 Token 通常比输入贵，限制输出长度是"低风险高收益"的手段。

## 7.1 maxTokens 限制

```java
ChatOptions options = ChatOptions.builder()
        .maxTokens(500)            // 限制输出长度
        .build();
```

## 7.2 提示约束

```java
// 提示中明确要求简洁
"用不超过 100 字回答。"
```

## 7.3 结构化输出

- 要求 JSON 时，应给字段模板，避免冗长自然语言。
- 结构化输出要预留解析失败的兜底。
- 过度限制长度可能截断 JSON，需权衡。

> 生产建议：按接口设置不同的 `maxTokens` 默认值，而非全局一刀切。

---

# 8. 批处理

**结论先行**：把多个独立请求合并为一次调用，可摊薄系统提示与网络开销。

## 8.1 合并调用

```java
// 合并多个独立请求为一次调用
String prompt = """
        请分别回答以下 %d 个问题，按顺序编号输出：
        %s
        """.formatted(questions.size(), formatQuestions(questions));

// 一次调用替代 N 次调用，节省系统提示的重复开销
```

## 8.2 Batch API

| 场景 | 说明 |
|------|------|
| 批量分类 | 一次处理多条 |
| 批量摘要 | 合并请求 |
| 离线任务 | 用 Batch API（更便宜） |

```java
// 伪代码：异步 Batch API
BatchJob job = batchApi.submit(requests);
BatchResult result = batchApi.poll(job.id());
```

> 部分厂商提供 Batch API，价格可低至 **50%**，适合非实时任务。

## 8.3 适用边界

- 合并请求不能有相互依赖。
- 批量过大时单次响应可能被截断，需分片。
- 实时交互场景不宜批处理（延迟敏感）。

> 生产建议：实时路径用 Prompt 缓存，离线路径用 Batch API，各取所长。

---

# 9. Embedding 成本优化

**结论先行**：Embedding 成本可通过去重、缓存、本地化与增量索引大幅降低。

## 9.1 去重与缓存

| 手段 | 说明 |
|------|------|
| 内容去重 | 相同内容不重复向量化 |
| 向量缓存 | 未变更文档复用向量 |

```java
// 增量：只向量化变更文档
if (!fingerprintStore.isUnchanged(doc)) {
    vectorStore.add(embed(doc));
}
```

## 9.2 本地模型

| 手段 | 说明 |
|------|------|
| 本地模型 | 一次性部署，零边际成本 |
| 降维 | Matryoshka 降维 |

- 本地模型省边际成本，但增加运维与硬件成本。
- 本地模型质量需与云端对比评测。

## 9.3 降维与增量

| 手段 | 说明 |
|------|------|
| 降维 | 减少向量维度，降低存储与计算 |
| 增量索引 | 只处理变更文档 |

> 生产建议：文档级指纹（fingerprint）是增量索引与缓存的基础设施。

---

# 10. 重排与检索优化

**结论先行**：重排是"质量放大器"，但候选越多越贵，需在精度与成本间取平衡。

## 10.1 候选数量

```java
// 重排候选从 100 降到 20~30
// 相似度阈值提高，减少无效候选
EmbeddingStoreContentRetriever.builder()
        .maxResults(20)            // 而不是 100
        .minScore(0.7)             // 而不是 0.5
        .build();
```

## 10.2 阈值

- 阈值过低会引入噪声，增加下游 Token。
- 阈值过高会漏召回，需评测确认。
- 阈值应随语料变化定期校准。

## 10.3 混合检索

```
稠密检索(语义) + 稀疏检索(关键词) → RRF 融合 → 重排 → topN
```

> 生产建议：召回阶段宽、重排阶段严，是控制成本的通用范式。

---

# 11. 成本监控与预算

**结论先行**：没有成本监控就无法优化；监控要能按维度拆解并触发预算告警。

## 11.1 成本追踪

```java
public class CostTracker {

    public void record(String tenantId, String model, Usage usage) {
        double cost = price(model)
                .calculate(usage.inputTokens(), usage.outputTokens());

        registry.counter("ai.cost", "tenant", tenantId, "model", model)
                .increment(cost);
    }
}
```

## 11.2 预算告警

```java
// 超预算告警
if (dailyCost(tenantId) > budget(tenantId) * 0.8) {
    alertService.warn("租户成本接近预算上限: " + tenantId);
}
```

## 11.3 多维归因

| 监控维度 | 用途 |
|----------|------|
| 按租户 | 计费与配额 |
| 按模型 | 选型决策 |
| 按接口 | 定位高消耗功能 |
| 按用户 | 防滥用 |

> 生产建议：成本看板与 Token 看板共用同一数据源，避免口径不一致。

---

# 12. 优化顺序

**结论先行**：优化有明确先后顺序，先建可见性，再做高收益手段。

## 12.1 分阶段落地

```
1. 建立成本监控（先能看见）
2. 模型分层路由（收益最大）
3. 缓存（结果 + 语义 + 向量）
4. Prompt 缓存（稳定前置）
5. 上下文裁剪（减少片段数）
6. 输出长度限制
7. 批处理（离线任务）
8. 本地 Embedding
9. 减少重排候选
```

## 12.2 收益评估

| 阶段 | 预期节省 | 风险 |
|------|:--------:|------|
| 监控 | 0（基础设施） | 低 |
| 路由 | 30%~70% | 中 |
| 缓存 | 10%~40% | 中 |
| Prompt 缓存 | 20%~50% | 低 |
| 裁剪/输出 | 20%~40% | 中 |

> 生产建议：每上一项优化都记录"优化前/后单位成本"，形成可复盘的收益账本。

---

# 13. 生产注意事项

**结论先行**：成本优化的最大风险是"省了钱、丢了质量"，必须用评测与监控兜底。

## 13.1 风险清单

| 风险 | 建议 |
|------|------|
| 无成本可见性 | 先建监控 |
| 一刀切用大模型 | 分层路由 |
| 缓存不一致 | 合理 TTL + 失效 |
| 过度裁剪 | 评测确认质量不降 |
| 无预算告警 | 按租户设预算 |
| 只看总量 | 按维度拆解定位 |

## 13.2 检查表

- [ ] 成本按租户/模型/接口/用户四维可查
- [ ] 有预算告警与降级策略
- [ ] 路由有命中分布与质量回退
- [ ] 缓存键版本化、TTL 合理
- [ ] 优化前后有质量评测对比

> 生产建议：把"成本"与"质量"放在同一张看板上，任何优化都不允许质量指标劣化。

---

# 14. 小结

- 模型调用是成本大头，优化重点在此。
- 模型分层路由收益最大，配合规则分类。
- 缓存（结果/语义/向量）与 Prompt 缓存收益显著。
- 上下文裁剪 + 输出限制能省下可观 Token。
- 先建立成本监控，再谈优化。
- 任何优化都要以评测确认质量不降为前提。
