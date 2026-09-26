# 1. 为什么用向量存记忆

## 1.1 窗口记忆的局限

**结论先行**：消息窗口只能保留最近 N 条，早期信息会丢失；向量记忆按语义相关性召回任意历史，不受位置限制。

### 1.1.1 问题

```
消息窗口：只能保留最近 N 条，早期信息丢失
向量记忆：按相关性召回任意历史，不受位置限制
```

### 1.1.2 向量优势

- 召回不依赖位置，只看语义相似。
- 容量远大于窗口。
- 可跨会话、跨时间召回早期信息。

## 1.2 窗口与向量对比

### 1.2.1 对比表

| 维度 | 窗口记忆 | 向量记忆 |
|------|----------|----------|
| 召回 | 按时间 | 按语义相关性 |
| 容量 | 有限 | 大 |
| 早期信息 | 丢失 | 可召回 |
| 成本 | 低 | 需向量化 |
| 精度 | 高（近因） | 依赖检索质量 |

### 1.2.2 适用场景

| 场景 | 推荐 |
|------|------|
| 短期多轮对话 | 窗口记忆 |
| 跨会话用户偏好 | 向量 + 结构化 |
| 长文档问答 | 向量检索 |
| 实时状态跟踪 | 窗口/内存 |

> 生产常用**窗口 + 向量**组合：近期全量，远期按需召回。

## 1.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 全部依赖向量 | 近因信息可能被淹 | 保留窗口 |
| 不做过滤 | 越权 | userId/tenantId |
| 全量向量化 | 成本高 | 阈值筛选 |
| 忽略时效 | 召回过期 | 时间衰减 |

# 2. 存储结构

## 2.1 MemoryEntry 定义

**结论先行**：每条记忆既要有内容，也要有用户、类型、重要性、时间等元数据，用于过滤与排序。

### 2.1.1 字段说明

```java
public record MemoryEntry(
        String id,
        String userId,
        String sessionId,
        String content,
        MemoryType type,
        double importance,
        Instant createdAt,
        Instant lastAccessedAt
) {}
```

| 字段 | 用途 |
|------|------|
| id | 唯一定位，更新/删除用 |
| userId | 用户隔离 |
| sessionId | 会话归属 |
| content | 记忆正文 |
| type | 类型分类 |
| importance | 排序权重 |
| createdAt | 时间衰减基准 |
| lastAccessedAt | 访问衰减基准 |

### 2.1.2 构造示例

```java
MemoryEntry entry = new MemoryEntry(
        UUID.randomUUID().toString(),
        user.userId(),
        sessionId,
        "用户偏好简洁回答",
        MemoryType.PREFERENCE,
        0.8,
        Instant.now(),
        Instant.now());
```

## 2.2 向量化存储

### 2.2.1 代码

```java
// 向量化并存储
public void store(MemoryEntry entry) {
    TextSegment segment = TextSegment.from(
            entry.content(),
            Metadata.from(Map.of(
                    "memoryId", entry.id(),
                    "userId", entry.userId(),
                    "tenantId", entry.tenantId(),
                    "type", entry.type().name(),
                    "importance", entry.importance(),
                    "createdAt", entry.createdAt().toString())));

    Embedding embedding = embeddingModel.embed(segment).content();
    memoryStore.add(embedding, segment);
}
```

### 2.2.2 元数据

| 元数据键 | 用途 | 是否必需 |
|----------|------|----------|
| memoryId | 回写/删除定位 | 是 |
| userId | 用户隔离 | 是 |
| tenantId | 租户隔离 | 是 |
| type | 类型筛选 | 是 |
| importance | 加权排序 | 否 |
| createdAt | 时间衰减 | 否 |

## 2.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 元数据缺失 | 无法过滤/排序 | 入库必带 |
| 内容过长 | 向量语义稀释 | 摘要 |
| 维度不匹配 | 检索报错 | 固定模型版本 |
| 无租户字段 | 跨租户泄露 | 强制 tenantId |

# 3. 召回

## 3.1 基本召回

### 3.1.1 流程

**结论先行**：召回 = 问题向量化 + 相似度检索 + 元数据过滤 + 阈值裁剪。

```
query ──embed──> 向量 ──search──> 候选 ──filter──> 结果
```

### 3.1.2 代码

```java
public List<MemoryEntry> recall(UserContext user, String query, int topK) {
    Embedding queryEmbedding = embeddingModel.embed(query).content();

    List<EmbeddingMatch<TextSegment>> matches = memoryStore.search(
            EmbeddingSearchRequest.builder()
                    .queryEmbedding(queryEmbedding)
                    .maxResults(topK)
                    .minScore(0.7)
                    .filter(metadataKey("userId").isEqualTo(user.userId()))   // 隔离
                    .build()).matches();

    return matches.stream()
            .map(m -> toMemoryEntry(m.embedded()))
            .toList();
}
```

## 3.2 过滤隔离

> **必须按 userId/tenantId 过滤**，否则会召回他人记忆。

### 3.2.1 过滤项

| 过滤项 | 作用 | 缺失后果 |
|--------|------|----------|
| userId | 用户隔离 | 越权 |
| tenantId | 租户隔离 | 跨租户泄露 |
| type | 类型筛选 | 结果混杂 |

### 3.2.2 阈值

| minScore | 效果 | 建议 |
|----------|------|------|
| 0.5 | 召回多、噪声大 | 探索 |
| 0.7 | 平衡 | 默认 |
| 0.85 | 精准、可能漏 | 严格场景 |

## 3.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 无用户过滤 | 召回他人 | 强制过滤 |
| minScore 过低 | 噪声 | 调阈值 |
| topK 过大 | 超预算 | 限制 |
| 查询为空 | 报错 | 参数校验 |

# 4. 重要性与时间加权

## 4.1 评分公式

**结论先行**：纯相似度可能召回无关内容，需结合重要性与时效综合排序。

### 4.1.1 公式

```
score = similarity * 0.6 + importance * 0.25 + recency * 0.15
recency = exp(-daysOld / 30)   # 30 天半衰期
```

### 4.1.2 参数含义

| 参数 | 含义 | 取值建议 |
|------|------|----------|
| similarity | 向量相似度 | 0~1 |
| importance | 记忆重要性 | 0~1 |
| recency | 时间新鲜度 | 0~1 |
| 30 | 半衰期天数 | 按业务 |

## 4.2 代码实现

### 4.2.1 score

```java
public double score(double similarity, double importance, Instant createdAt) {
    // 时间衰减：越久远权重越低
    long daysOld = ChronoUnit.DAYS.between(createdAt, Instant.now());
    double recency = Math.exp(-daysOld / 30.0);      // 30 天半衰期

    // 综合评分
    return similarity * 0.6 + importance * 0.25 + recency * 0.15;
}
```

### 4.2.2 recallRanked

```java
public List<MemoryEntry> recallRanked(UserContext user, String query, int topK) {
    List<MemoryEntry> candidates = recall(user, query, topK * 3);

    return candidates.stream()
            .sorted(Comparator.comparingDouble(
                    (MemoryEntry m) -> score(m.similarity(), m.importance(),
                            m.createdAt())).reversed())
            .limit(topK)
            .toList();
}
```

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 权重拍脑袋 | 排序不优 | A/B 调参 |
| 半衰期过短 | 旧记忆消失 | 按业务调 |
| 只召回不重排 | 质量差 | 先多召回再重排 |
| similarity 缺失 | 无法计算 | 检索时保留分数 |

# 5. 去重

## 5.1 相似度去重

### 5.1.1 原理

**结论先行**：相似记忆不应重复存储，写入前用高阈值查重。

```
新记忆 ──embed──> 检索相似 ──命中──> 更新
                          └─未命中──> 新增
```

### 5.1.2 代码

```java
public void storeWithDedup(MemoryEntry entry) {
    // 查找相似记忆
    List<EmbeddingMatch<TextSegment>> similar = memoryStore.search(
            EmbeddingSearchRequest.builder()
                    .queryEmbedding(embeddingModel.embed(entry.content()).content())
                    .maxResults(1)
                    .minScore(0.92)                       // 高阈值
                    .filter(metadataKey("userId").isEqualTo(entry.userId()))
                    .build()).matches();

    if (!similar.isEmpty()) {
        // 已存在相似 → 更新（提高重要性、刷新时间）
        String existingId = similar.get(0).embedded().metadata()
                .get("memoryId").toString();
        updateImportance(existingId, entry.importance());
        touch(existingId);
        return;
    }
    store(entry);
}
```

## 5.2 更新策略

### 5.2.1 策略表

| 场景 | 处理 |
|------|------|
| 完全重复 | 刷新访问时间 |
| 相似但更完整 | 删除旧 + 写新 |
| 相似且矛盾 | 以新为准 |
| 互补 | 合并后重写 |

### 5.2.2 说明

- 去重阈值应显著高于召回阈值（去重用 0.92，召回用 0.7）。
- 去重必须先按 userId 过滤，否则会跨用户合并。

## 5.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 阈值过低 | 误判重复 | 提高阈值 |
| 阈值过高 | 重复堆积 | 结合精确键 |
| 去重跨用户 | 越权 | 过滤 userId |
| 忽略元数据 | 误删 | 校验 memoryId |

# 6. 更新与删除

## 6.1 更新

### 6.1.1 原理

**结论先行**：多数向量库不支持原地更新，更新等于"删除 + 重写"。

### 6.1.2 代码

```java
// 向量库通常不支持原地更新 → 删除 + 重写
public void update(MemoryEntry entry) {
    memoryStore.remove(entry.id());
    store(entry);
}
```

## 6.2 删除

### 6.2.1 单条

```java
// 删除单条
public void delete(String memoryId) {
    memoryStore.remove(memoryId);
}
```

### 6.2.2 批量

```java
// 删除用户全部记忆
public void deleteByUser(String userId) {
    List<String> ids = findByUser(userId).stream()
            .map(MemoryEntry::id).toList();
    memoryStore.removeAll(ids);
}
```

## 6.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 更新非原子 | 中间态丢失 | 先写后删或版本号 |
| 删除遗漏元数据 | 残留 | 同时删结构化 |
| 未校验归属 | 越权删 | 校验 userId |
| 大批量删除慢 | 超时 | 分批 |

# 7. 访问追踪（用于衰减）

## 7.1 记录访问

### 7.1.1 原理

**结论先行**：只有记录"何时被访问"，才能实现基于使用的重要性衰减。

### 7.1.2 代码

```java
public List<MemoryEntry> recallWithTouch(UserContext user, String query, int topK) {
    List<MemoryEntry> results = recall(user, query, topK);

    // 记录访问，用于后续重要性衰减
    results.forEach(m -> {
        memoryStore.touch(m.id());
        metaStore.updateLastAccessed(m.id(), Instant.now());
    });
    return results;
}
```

## 7.2 用于衰减

### 7.2.1 衰减公式

```
importance_new = importance_old * 0.95 ^ daysSinceLastAccess
```

### 7.2.2 代码

```java
@Scheduled(cron = "0 0 3 * * *")
public void decay() {
    metaStore.findAll().forEach(m -> {
        long days = ChronoUnit.DAYS.between(m.lastAccessedAt(), Instant.now());
        double factor = Math.pow(0.95, days);
        metaStore.updateImportance(m.id(), m.importance() * factor);
    });
}
```

## 7.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 不追踪 | 无法衰减 | 每次召回 touch |
| 写放大 | 高频更新 | 异步批量 |
| 衰减过快 | 丢有用记忆 | 调因子 |
| 未持久化 | 重启丢失 | 落库 |

# 8. 混合记忆架构

## 8.1 架构图

### 8.1.1 图

**结论先行**：生产用"窗口 + 向量"混合：近期全量带入，远期按相关性召回。

```
当前会话：
  ├── 窗口记忆（最近 20 条）    ← 全量带入
  └── 向量记忆（早期重要内容）  ← 按相关性召回
```

### 8.1.2 说明

- 窗口负责"连贯性"，向量负责"长期信息"。
- 两者合并前需去重，避免同一内容出现两次。

## 8.2 上下文构建

### 8.2.1 代码

```java
public List<Message> buildContext(String sessionId, String question) {
    List<Message> context = new ArrayList<>();

    // ① 窗口：最近消息全量
    List<Message> recent = conversationStore.recent(sessionId, 20);
    context.addAll(recent);

    // ② 向量：早期重要内容按相关性召回
    List<MemoryEntry> old = vectorMemory.recall(sessionId, question, 5);
    if (!old.isEmpty()) {
        String recalled = formatMemories(old);
        context.add(SystemMessage.from("相关历史：\n" + recalled));
    }

    // ③ 当前问题
    context.add(UserMessage.from(question));
    return context;
}
```

### 8.2.2 预算分配

| 部分 | 预算占比 | 说明 |
|------|----------|------|
| 窗口 | 60% | 近期对话 |
| 向量历史 | 25% | 长期信息 |
| 系统提示 | 15% | 指令与格式 |

## 8.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 窗口与向量重复 | 冗余 | 去重 |
| 向量召回挤占窗口 | 超预算 | 分配预算 |
| 历史注入位置错 | 模型混淆 | 系统消息区 |
| 不隔离会话/用户 | 串话 | 过滤 |

# 9. 成本控制

## 9.1 手段

### 9.1.1 手段表

**结论先行**：向量记忆的主要成本是 Embedding 与存储，必须通过筛选、去重、摘要、本地化来控制。

| 手段 | 说明 |
|------|------|
| 只向量化重要内容 | 不重要内容不入库 |
| 去重 | 避免重复向量化 |
| 摘要后入库 | 长内容先摘要 |
| 本地 Embedding | 降边际成本 |
| 定期清理 | 删除过期/低价值记忆 |

### 9.1.2 入库阈值

```java
// 只向量化"值得记住"的内容
public boolean shouldStore(String content, double importance) {
    return importance >= 0.6 && content.length() >= 20;
}
```

## 9.2 代码

### 9.2.1 摘要后入库

```java
public MemoryEntry summarizeBeforeStore(MemoryEntry entry) {
    if (entry.content().length() <= 200) return entry;
    String summary = summarizer.summarize(entry.content());   // LLM 摘要
    return entry.withContent(summary);
}
```

### 9.2.2 定期清理

```java
@Scheduled(cron = "0 0 4 * * *")
public void cleanup() {
    memoryStore.removeAll(findLowValue(0.2, 90));   // 低价值且 90 天未访问
}
```

## 9.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 阈值过高 | 漏记有用信息 | 调阈值 |
| 摘要失真 | 语义丢失 | 保留要点 |
| 本地模型弱 | 检索差 | 评估质量 |
| 从不清理 | 成本累积 | 定期任务 |

# 10. 生产注意事项

## 10.1 风险清单

### 10.1.1 风险表

| 风险 | 建议 |
|------|------|
| 无用户过滤 | 必按 userId/tenantId |
| 记忆爆炸 | 重要性阈值 + 清理 |
| 重复存储 | 相似度去重 |
| 无时效权重 | 加时间衰减 |
| 全量向量化 | 只存重要的 |
| 无访问追踪 | 记录 lastAccessed |
| 更新不当 | 删除 + 重写 |

### 10.1.2 优先级

| 优先级 | 事项 | 理由 |
|--------|------|------|
| P0 | 用户/租户过滤 | 安全底线 |
| P0 | 删除能力 | 合规 |
| P1 | 去重 + 清理 | 成本 |
| P1 | 时效加权 | 质量 |
| P2 | 访问追踪 | 优化 |

## 10.2 监控

### 10.2.1 指标

| 指标 | 含义 | 告警 |
|------|------|------|
| 向量条数/用户 | 膨胀 | 超上限 |
| 召回命中率 | 质量 | 过低 |
| Embedding 调用量 | 成本 | 突增 |
| 平均召回延迟 | 性能 | 超时 |

### 10.2.2 告警建议

- Embedding 调用量突增通常意味着去重失效或写入失控。
- 召回延迟上升优先排查向量库索引与 topK 设置。

## 10.3 架构建议

- 向量库按租户分集合或强制过滤。
- 召回"先粗后精"：多召回再重排。
- 结构化元数据与向量分离存储，便于更新与审计。

# 11. 小结

- 向量记忆按语义召回，突破窗口限制。
- 必须按用户/租户过滤，防越权。
- 结合相似度、重要性、时效综合排序。
- 去重、更新（删+写）、访问追踪是必备。
- 生产用"窗口 + 向量"混合架构。
- 成本控制靠阈值筛选、摘要与定期清理。
