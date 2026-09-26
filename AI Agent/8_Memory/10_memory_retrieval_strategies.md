# 1. 召回的目标

记忆召回（memory retrieval）是在有限的 Token 预算（token budget）下，
从海量记忆条目中挑出"最该给模型看"的那一部分。

## 1.1 什么是记忆召回

结论先行：召回不是"把所有相关记忆都塞进上下文"，
而是在预算约束下做一次带权重的排序与截断。

### 1.1.1 召回的位置

机制上，召回位于"记忆写入"与"上下文组装"之间：

```
写入 ──▶ [记忆库] ──▶ 召回 ──▶ 去重/重排 ──▶ 预算裁剪 ──▶ 上下文
                        ▲
                    当前 query
```

### 1.1.2 召回四阶段

| 阶段 | 作用 | 典型手段 |
|------|------|----------|
| 候选生成 | 先扩大候选集 | 多路召回 |
| 融合去重 | 合并、去除重复 | 归一化 + 去重 |
| 打分排序 | 决定优先级 | 综合评分 |
| 裁剪输出 | 控制 Token | 预算裁剪 |

## 1.2 五大目标

| 目标 | 说明 |
|------|------|
| 相关性 | 与当前问题相关 |
| 重要性 | 关键信息优先 |
| 时效性 | 近期信息优先 |
| 多样性 | 避免重复 |
| 预算内 | 不超 Token |

这五个目标往往互相冲突：最相关的不一定最重要，最新的不一定最相关。
召回的本质是**在冲突中做加权折中**。

# 2. 召回方式

## 2.1 单路召回方式

结论先行：不同记忆类型适用不同召回方式，生产上通常组合使用。

| 方式 | 说明 | 适用 |
|------|------|------|
| 全量 | 全部带入 | 核心记忆（小） |
| 时间窗口 | 最近 N 条 | 近期对话 |
| 语义检索 | 向量相似度 | 历史事实 |
| 结构化查询 | 按条件筛选 | 用户偏好 |
| 混合 | 多路合并 | 生产推荐 |

## 2.2 语义召回实现

结论先行：语义召回用向量相似度（vector similarity）找"意思相近"的记忆，
适合"事实型"记忆，但对数字、ID 等精确信息不可靠。

### 2.2.1 向量检索实现

```java
// LangChain4j 风格：基于 EmbeddingStore 的语义召回
public List<MemoryEntry> semanticRecall(String userId, String query, int topK) {
    Embedding queryEmbedding = embeddingModel.embed(query).content();

    EmbeddingSearchRequest request = EmbeddingSearchRequest.builder()
            .queryEmbedding(queryEmbedding)
            .maxResults(topK)
            .minScore(0.6)               // 相似度下限，过滤噪声
            .filter(metadataKey("userId").isEqualTo(userId))
            .build();

    return embeddingStore.search(request).matches().stream()
            .map(m -> toMemoryEntry(m.embedded()))
            .toList();
}
```

### 2.2.2 边界与坑

- 相似度阈值（minScore）过低会引入大量噪声，过高会漏召。
- 向量模型对领域术语区分度差，需配合关键词召回；过滤必须带租户/用户。

## 2.3 方式对比与选型

| 方式 | 召回率 | 精度 | 延迟 | 成本 | 典型场景 |
|------|:------:|:----:|:----:|:----:|----------|
| 全量 | 高 | 低 | 低 | 高 | 少量核心记忆 |
| 时间窗口 | 中 | 中 | 低 | 低 | 近期对话 |
| 语义检索 | 高 | 中 | 中 | 中 | 历史事实 |
| 结构化查询 | 低 | 高 | 低 | 低 | 用户偏好 |
| 混合 | 高 | 高 | 高 | 中 | 生产系统 |

生产建议：默认走混合召回，用"结构化 + 时间窗口"保证精确性，
用"语义检索"兜底长尾，再统一打分裁剪。

# 3. 综合评分

## 3.1 评分公式拆解

结论先行：单一相似度不够，需把"相似度、重要性、时效、频率"
合成一个 0~1 的分数用于排序。

```java
public double score(MemoryEntry memory, String query, Instant now) {
    // ① 语义相似度（0~1）
    double similarity = cosineSimilarity(
            embed(query), memory.embedding());

    // ② 重要性（0~1）
    double importance = memory.importance();

    // ③ 时间新鲜度（0~1，30 天半衰期）
    long daysOld = ChronoUnit.DAYS.between(memory.createdAt(), now);
    double recency = Math.exp(-daysOld / 30.0);

    // ④ 访问频率（常被访问的更相关）
    double frequency = Math.min(1.0, memory.accessCount() / 10.0);

    // 加权组合
    return similarity * 0.5
            + importance * 0.2
            + recency * 0.2
            + frequency * 0.1;
}
```

| 分量 | 取值 | 作用 | 失效场景 |
|------|------|------|----------|
| similarity | 0~1 | 相关性 | 无 embedding 时退化为 0 |
| importance | 0~1 | 重要性 | 写入时未标注则恒定 |
| recency | 0~1 | 新鲜度 | 长期有效知识被压制 |
| frequency | 0~1 | 热度 | 高频噪声被放大 |

## 3.2 完整 Java 实现

结论先行：把评分封装为可配置的评分器，权重外置，便于按场景调优。

### 3.2.1 评分器

```java
public class MemoryScorer {

    private final ScoreWeights weights;

    public MemoryScorer(ScoreWeights weights) {
        this.weights = weights;
    }

    public double score(MemoryEntry m, double[] queryEmbedding, Instant now) {
        double similarity = m.embedding() == null
                ? 0.0 : cosine(queryEmbedding, m.embedding());

        double importance = clamp01(m.importance());

        double daysOld = Math.max(0,
                ChronoUnit.DAYS.between(m.createdAt(), now));
        double recency = Math.exp(-daysOld / weights.halfLifeDays());

        double frequency = Math.min(1.0, m.accessCount() / 10.0);

        return similarity * weights.similarity()
                + importance * weights.importance()
                + recency * weights.recency()
                + frequency * weights.frequency();
    }

    private static double clamp01(double v) {
        return Math.max(0.0, Math.min(1.0, v));
    }
}
```

### 3.2.2 权重配置

```java
public record ScoreWeights(
        double similarity,
        double importance,
        double recency,
        double frequency,
        double halfLifeDays) {}
```

## 3.3 权重调优与边界

| 场景 | similarity | importance | recency | frequency |
|------|:----------:|:----------:|:-------:|:---------:|
| 客服对话 | 0.5 | 0.2 | 0.2 | 0.1 |
| 长期知识库 | 0.6 | 0.3 | 0.05 | 0.05 |
| 实时助手 | 0.4 | 0.1 | 0.4 | 0.1 |
| 任务型 Agent | 0.4 | 0.3 | 0.1 | 0.2 |

边界与坑：

- 权重之和不必为 1，但应保持量纲一致；recency 对"长期有效"知识是负作用，应分类处理。
- frequency 会形成"马太效应"：越常被召回越常被召回，需上限约束。

生产建议：权重做成配置项，配合离线评测做网格搜索或贝叶斯优化。

# 4. 多路召回

## 4.1 召回管线设计

结论先行：多路召回（hybrid recall）并行发起多路查询，
合并候选后再统一去重、打分、裁剪。

```
        ┌─ 核心记忆（全量）──────┐
query ──┼─ 近期对话（窗口 10）───┼─▶ 合并 ─▶ 去重 ─▶ 评分 ─▶ 裁剪
        ├─ 语义召回（TopK 10）───┤
        └─ 结构化（偏好）───────┘
```

## 4.2 完整代码实现

```java
public List<MemoryEntry> hybridRecall(
        UserContext user, String query, int budget) {
    List<MemoryEntry> candidates = new ArrayList<>();

    // ① 核心记忆（全量，小）
    candidates.addAll(coreMemory.all(user.userId()));

    // ② 近期对话（时间窗口）
    candidates.addAll(conversationStore.recent(user.sessionId(), 10));

    // ③ 语义召回（向量）
    candidates.addAll(vectorMemory.recall(user.userId(), query, 10));

    // ④ 结构化召回（偏好类）
    candidates.addAll(structuredMemory.findByType(
            user.userId(), MemoryType.PREFERENCE));

    // 去重 + 评分排序 + 预算裁剪
    return deduplicate(candidates).stream()
            .sorted(Comparator.comparingDouble(m ->
                    score(m, query, Instant.now())).reversed())
            .limit(budget)
            .toList();
}
```

并行版本（生产推荐）：

```java
public List<MemoryEntry> hybridRecallParallel(UserContext user, String query) {
    CompletableFuture<List<MemoryEntry>> core = CompletableFuture.supplyAsync(
            () -> coreMemory.all(user.userId()));
    CompletableFuture<List<MemoryEntry>> sem = CompletableFuture.supplyAsync(
            () -> vectorMemory.recall(user.userId(), query, 10));

    return Stream.of(core, sem).map(CompletableFuture::join)
            .flatMap(List::stream).toList();
}
```

## 4.3 边界与反例

反例一：所有路都返回大量候选，合并后去重排序开销大。
应给每路设置 `topK` 上限，控制候选总量。

反例二：某一路超时拖垮整体延迟。
应为每路设置独立超时与降级（返回空列表）。

| 风险 | 后果 | 对策 |
|------|------|------|
| 候选爆炸 | 排序慢、内存高 | 每路 TopK 上限 |
| 单路超时 | 整体延迟升高 | 独立超时 + 降级 |
| 重复召回 | 浪费预算 | 归一化去重 |
| 跨租户泄漏 | 安全事故 | 强制租户过滤 |

# 5. 去重

## 5.1 去重策略

结论先行：去重先做文本归一化（normalize），再按内容键合并，
冲突时保留"更重要"的那条。

| 策略 | 说明 | 适用 |
|------|------|------|
| 精确去重 | 内容完全一致 | 完全重复 |
| 归一化去重 | 去空格/标点后一致 | 格式差异 |
| 语义去重 | 向量相似度 > 阈值 | 同义改写 |

## 5.2 代码实现

```java
public List<MemoryEntry> deduplicate(List<MemoryEntry> memories) {
    Map<String, MemoryEntry> unique = new LinkedHashMap<>();

    for (MemoryEntry m : memories) {
        String key = normalize(m.content());
        MemoryEntry existing = unique.get(key);

        if (existing == null || m.importance() > existing.importance()) {
            unique.put(key, m);            // 保留更重要的
        }
    }
    return new ArrayList<>(unique.values());
}

private String normalize(String content) {
    // 归一化：去空格、统一标点、小写
    return content.replaceAll("\\s+", "")
            .replaceAll("[，。！？]", "")
            .toLowerCase();
}
```

## 5.3 边界与坑

- 语义去重伪代码：若已选集合中存在相似度 > 0.95 的条目，
  则保留分数更高者，否则加入。
- 归一化过狠会把不同内容合并（如 "A-123" 与 "A123"）；含 ID、金额的记忆应避免删除数字与连字符。

# 6. 多样性

## 6.1 多样性控制

结论先行：多样性控制避免召回大量"意思重复"的内容，
让上下文覆盖不同信息面。

```java
public List<MemoryEntry> enforceDiversity(List<MemoryEntry> memories, int max) {
    List<MemoryEntry> selected = new ArrayList<>();

    for (MemoryEntry candidate : memories) {
        // 与已选内容相似度过高则跳过
        boolean tooSimilar = selected.stream().anyMatch(s ->
                cosineSimilarity(s.embedding(), candidate.embedding()) > 0.9);

        if (!tooSimilar) {
            selected.add(candidate);
            if (selected.size() >= max) break;
        }
    }
    return selected;
}
```

## 6.2 MMR 实现

结论先行：MMR（Maximal Marginal Relevance，最大边际相关性）
在"相关性"与"多样性"之间显式加权，比硬阈值更平滑。

```
MMR = argmax [ λ·sim(d, q) − (1−λ)·max sim(d, s) ]
                   相关          与已选的最大相似
```

```java
public List<MemoryEntry> mmr(List<MemoryEntry> candidates,
                             double[] queryEmbedding,
                             int k, double lambda) {
    List<MemoryEntry> selected = new ArrayList<>();
    List<MemoryEntry> pool = new ArrayList<>(candidates);

    while (selected.size() < k && !pool.isEmpty()) {
        MemoryEntry best = null;
        double bestScore = Double.NEGATIVE_INFINITY;

        for (MemoryEntry d : pool) {
            double rel = cosine(queryEmbedding, d.embedding());
            double red = selected.stream().mapToDouble(s ->
                    cosine(s.embedding(), d.embedding())).max().orElse(0.0);
            double mmr = lambda * rel - (1 - lambda) * red;
            if (mmr > bestScore) { bestScore = mmr; best = d; }
        }
        selected.add(best);
        pool.remove(best);
    }
    return selected;
}
```

| lambda | 行为 | 适用 |
|:------:|------|------|
| 1.0 | 纯相关性 | 只求最相关 |
| 0.7 | 相关为主 | 通用 |
| 0.5 | 平衡 | 信息面广 |

# 7. 时效处理

## 7.1 硬过期

结论先行：硬过期（hard expiration）对超期记忆直接过滤，
适用于有明确有效期的信息（如临时令牌、活动信息）。

```java
public boolean isExpired(MemoryEntry m) {
    return m.expiresAt() != null && m.expiresAt().isBefore(Instant.now());
}
```

## 7.2 软衰减

结论先行：软衰减（soft decay）不删除记忆，只降低其权重，
时间越久权重越低（见第 3 节 score 的 recency 项）。

## 7.3 冲突解决

结论先行：同一键（key）的新旧记忆冲突时，默认以新覆盖旧，
但需保留历史版本以便审计。

```java
public MemoryEntry resolveConflict(MemoryEntry old, MemoryEntry fresh) {
    if (old.key() != null && old.key().equals(fresh.key())) {
        return fresh.createdAt().isAfter(old.createdAt()) ? fresh : old;
    }
    return fresh;
}
```

边界与坑：

- "新覆盖旧"对"用户改主意"正确，对"误写入"会污染。
- 时间戳可能因时钟漂移乱序，应使用单调版本号；冲突解决需保留 `supersededBy` 链。

生产建议：对偏好类记忆用"新覆盖旧 + 版本链"，
对事实类记忆用"多版本共存 + 时效加权"。

# 8. 预算裁剪

## 8.1 Token 预算模型

结论先行：召回结果必须塞进固定的 Token 预算，
超出的按分数从低到高丢弃。预算来自"总窗口减去系统提示与输出预留"。

## 8.2 裁剪实现

```java
public String renderWithinBudget(List<MemoryEntry> memories, int maxTokens) {
    StringBuilder sb = new StringBuilder();
    int used = 0;

    for (MemoryEntry m : memories) {
        String line = "- " + m.content() + "\n";
        int tokens = estimateTokens(line);

        if (used + tokens > maxTokens) break;

        sb.append(line);
        used += tokens;
    }
    return sb.toString();
}
```

## 8.3 边界与坑

- `estimateTokens` 只是估算，需留安全余量（如 10%）；单条超预算会导致后续被跳过，可对单条截断。
- 中文按字估算通常偏高，英文按 4 字符 ≈ 1 token 偏低。

# 9. 召回效果评测

## 9.1 评测指标

结论先行：记忆召回和检索一样需要评测，否则无法知道是否变好。

| 指标 | 说明 |
|------|------|
| 召回率 | 该召回的是否召回 |
| 精确率 | 召回的是否都相关 |
| 排序质量 | 相关的排前面 |
| Token 效率 | 每 Token 的信息量 |

## 9.2 评测实现

### 9.2.1 命中率

```java
public double recallQuality(List<MemoryEvalCase> cases) {
    int hit = 0;

    for (MemoryEvalCase c : cases) {
        List<MemoryEntry> recalled = recall(c.user(), c.query(), 10);

        // 期望召回的记忆是否命中
        if (recalled.stream()
                .anyMatch(m -> c.expectedMemoryIds().contains(m.id()))) {
            hit++;
        }
    }
    return hit / (double) cases.size();
}
```

### 9.2.2 NDCG

```java
public double ndcg(List<MemoryEntry> ranked, Set<String> relevant, int k) {
    double dcg = 0, idcg = 0;
    for (int i = 0; i < k; i++) {
        double gain = 1.0 / (Math.log(i + 2) / Math.log(2));
        if (i < ranked.size() && relevant.contains(ranked.get(i).id()))
            dcg += gain;
        if (i < relevant.size()) idcg += gain;
    }
    return idcg == 0 ? 0 : dcg / idcg;
}
```

边界与坑：评测集需覆盖长尾与对抗查询，只用热门查询会高估效果。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 只按相似度 | 综合重要性/时效 |
| 无去重 | 归一化去重 |
| 内容雷同 | 多样性控制 |
| 过期未过滤 | 硬过期 + 软衰减 |
| 超预算 | 按预算裁剪 |
| 无评测 | 建记忆召回评测 |

## 10.2 生产建议

结论先行：召回是"读路径"上最频繁的操作，
必须同时保证正确性、延迟与成本。

- 加缓存：对同一 (user, query) 的召回结果做短 TTL 缓存。
- 加降级：向量服务不可用时退回关键词/结构化召回。
- 加监控与护栏：记录候选数、耗时、命中率、Token，并强制租户隔离。

```java
Timer.Sample sample = Timer.start(registry);
List<MemoryEntry> result = hybridRecall(user, query, budget);
sample.stop(registry.timer("recall.latency"));
registry.counter("recall.hit").increment(result.size());
```

# 11. 小结

## 11.1 核心要点

- 召回目标：相关、重要、新鲜、多样、预算内。
- 综合评分：相似度 + 重要性 + 时效 + 频率。
- 多路召回后去重、多样性控制、预算裁剪。

## 11.2 工程要点

- 时效用"硬过期 + 软衰减"。
- 记忆召回也需要评测。
