# 1. LLM 应用的缓存层次

**结论先行**：LLM（Large Language Model，大语言模型）应用的缓存不是一个点，而是一个从「结果」到「Prompt（提示词）」的五层体系。层次越靠上越省，但风险越高；越靠下越安全，但收益越小。

## 1.1 五层缓存全景

### 1.1.1 层次结构

```
┌─────────────────────────────────────────┐
│ ① 结果缓存：相同问题直接返回            │  最省
├─────────────────────────────────────────┤
│ ② 语义缓存：相似问题返回缓存答案        │
├─────────────────────────────────────────┤
│ ③ 检索缓存：相同查询返回检索结果        │
├─────────────────────────────────────────┤
│ ④ 向量缓存：相同文本不重复向量化        │
├─────────────────────────────────────────┤
│ ⑤ Prompt 缓存：模型侧缓存提示前缀       │  省钱
└─────────────────────────────────────────┘
```

### 1.1.2 收益与风险

| 层 | 收益 | 风险 |
|----|:----:|:----:|
| 结果缓存 | 极高 | 高（个性化 / 陈旧） |
| 语义缓存 | 高 | 高（误命中） |
| 检索缓存 | 中 | 低 |
| 向量缓存 | 中 | 极低 |
| Prompt 缓存 | 中 | 低 |

## 1.2 缓存决策原则

### 1.2.1 确定性优先

只缓存确定性高、无副作用的内容；个性化、实时、含隐私的数据一律不缓存。

### 1.2.2 就近失效

知识更新时，优先失效受影响的缓存，而非全量清空。

> 生产建议：从最安全的向量缓存与检索缓存做起，再逐步向上引入语义与结果缓存。

# 2. 结果缓存

**结论先行**：结果缓存直接返回已生成的答案，收益最高，但也最容易出错，必须严格限定缓存条件。

## 2.1 实现

### 2.1.1 基础代码

```java
public String chatWithCache(String question, UserContext user) {
    String key = cacheKey(question, user);

    String cached = redis.opsForValue().get(key);
    if (cached != null) {
        return cached;
    }

    String answer = chatClient.prompt().user(question).call().content();

    // 只缓存确定性高、无个性化的回答
    if (isCacheable(question)) {
        redis.opsForValue().set(key, answer, Duration.ofHours(1));
    }
    return answer;
}

private String cacheKey(String question, UserContext user) {
    return "chat:%s:%s".formatted(user.tenantId(), sha256(question));
}
```

### 2.1.2 key 设计

key 至少应包含租户、问题哈希与版本三部分：

```
chat:{tenantId}:{promptVersion}:{sha256(question)}
```

## 2.2 什么可以缓存

### 2.2.1 可缓存与不可缓存

| 可缓存 | 不可缓存 |
|--------|----------|
| 通用知识问答 | 个性化推荐 |
| 政策解释 | 实时数据 |
| 固定流程说明 | 含用户隐私 |
| 静态文档摘要 | 有副作用的操作 |

### 2.2.2 可缓存性判定

判定逻辑应集中在一处，避免散落各处导致不一致：

```java
private boolean isCacheable(String question) {
    if (containsPii(question)) return false;        // 含隐私
    if (isRealtime(question)) return false;         // 实时数据
    if (isUserScoped(question)) return false;       // 个性化
    return question.length() >= 8;                  // 过滤过短问题
}
```

## 2.3 边界与坑

> **key 必须包含租户**，否则会串数据。

- 缓存穿透：大量不存在的 key 直击后端。
- 陈旧数据：知识更新后旧答案仍在返回。
- 隐私泄露：把用户私有内容写入共享缓存。

> 生产建议：为结果缓存配置独立的命中率与陈旧率监控。

# 3. 语义缓存

**结论先行**：语义缓存用向量相似度匹配「换一种说法」的相同问题，能显著提升命中率，但存在误命中风险，阈值必须保守。

## 3.1 原理

### 3.1.1 相似问题命中

```
"如何报销差旅费"  → 缓存命中
"差旅费用怎么报"  → 语义相似度 0.96 → 命中同一缓存
```

### 3.1.2 阈值与效果

| 阈值 | 效果 |
|:----:|------|
| 0.99 | 几乎只有完全相同才命中 |
| 0.95 | 高度相似命中（推荐） |
| 0.90 | 较宽松，有误命中风险 |

## 3.2 实现

```java
public class SemanticCache {

    private final EmbeddingModel embeddingModel;
    private final EmbeddingStore<TextSegment> cacheStore;
    private final Map<String, String> answerMap;

    public Optional<String> get(String question) {
        Embedding queryEmb = embeddingModel.embed(question).content();

        List<EmbeddingMatch<TextSegment>> matches = cacheStore.search(
                EmbeddingSearchRequest.builder()
                        .queryEmbedding(queryEmb)
                        .maxResults(1)
                        .minScore(0.95)              // 高阈值，避免误命中
                        .build()).matches();

        if (matches.isEmpty()) return Optional.empty();

        String cachedQuestionId = matches.get(0).embedded().id();
        return Optional.ofNullable(answerMap.get(cachedQuestionId));
    }

    public void put(String question, String answer) {
        Embedding emb = embeddingModel.embed(question).content();
        String id = UUID.randomUUID().toString();
        cacheStore.add(emb, TextSegment.from(question,
                Metadata.from(Map.of("id", id))));
        answerMap.put(id, answer);
    }
}
```

## 3.3 边界与坑

> **语义缓存有误命中风险**（把不同问题的答案返回），阈值要保守。

- 否定句极易误命中：「可以退款」与「不可以退款」向量很近。
- 多语言混合会降低相似度判定质量。
- 缓存库本身要设 TTL 与容量上限。

> 生产建议：对语义缓存记录「相似度分布」，定期抽样人工复核误命中。

# 4. 检索缓存

**结论先行**：检索缓存缓存的是「问题 → 文档列表」，即使最终答案不同，检索结果通常也可复用，因此比结果缓存更安全。

## 4.1 实现

```java
public List<Document> retrieveWithCache(String question, String tenantId) {
    String key = "retrieval:%s:%s".formatted(tenantId, sha256(question));

    List<Document> cached = retrievalCache.get(key);
    if (cached != null) return cached;

    List<Document> docs = doRetrieve(question, tenantId);
    retrievalCache.put(key, docs, Duration.ofMinutes(30));
    return docs;
}
```

## 4.2 适用与不适用

### 4.2.1 场景匹配

| 适用 | 不适用 |
|------|--------|
| 高频重复问题 | 实时数据检索 |
| 静态知识库 | 强权限隔离场景 |
| 检索开销大的场景 | 数据秒级变化 |

### 4.2.2 权限感知的 key

当检索结果受权限过滤影响时，key 必须包含权限维度：

```java
private String retrievalKey(String question, String tenantId, Set<String> scopes) {
    String scopeSig = sha256(String.join(",", new TreeSet<>(scopes)));
    return "retrieval:%s:%s:%s".formatted(tenantId, scopeSig, sha256(question));
}
```

## 4.3 边界与坑

> 检索缓存比结果缓存**更安全**（即使答案不同，检索结果通常可复用）。

- 检索结果含权限过滤时，key 必须包含权限维度。
- 文档更新后要主动失效相关检索缓存。
- 缓存的是引用而非内容，注意文档 ID 失效问题。

> 生产建议：检索缓存 TTL 应短于文档更新周期。

# 5. 向量缓存

**结论先行**：向量缓存避免对相同文本重复调用嵌入模型，是收益最稳定、风险最低的一层。

## 5.1 实现

```java
public Embedding embedWithCache(String text) {
    String hash = sha256(text);

    Embedding cached = vectorCache.get(hash);
    if (cached != null) return cached;

    Embedding embedding = embeddingModel.embed(text).content();
    vectorCache.put(hash, embedding);
    return embedding;
}
```

## 5.2 增量索引中的应用

### 5.2.1 收益对比

> 增量索引时，未变更文档的向量可直接复用，**大幅降低向量化成本**。

| 场景 | 无缓存 | 有缓存 |
|------|--------|--------|
| 全量重建 100 万文档 | 100 万次嵌入 | 首次 100 万次 |
| 增量更新 1% | 1 万次 | 0（未变更） |

### 5.2.2 增量流程

```java
public Embedding embedOrReuse(Document doc, Map<String, String> hashIndex) {
    String hash = sha256(doc.text());
    if (hash.equals(hashIndex.get(doc.id()))) {
        return vectorCache.get(hash);   // 内容未变，直接复用
    }
    Embedding emb = embeddingModel.embed(doc.text()).content();
    vectorCache.put(hash, emb);
    hashIndex.put(doc.id(), hash);
    return emb;
}
```

## 5.3 边界与坑

- 文本哈希要包含模型标识，换模型后旧向量不可复用。
- 哈希碰撞概率极低但非零，可用 SHA-256。
- 向量体积大，缓存容量与淘汰策略要合理。

> 生产建议：向量缓存持久化到磁盘或对象存储，跨重启复用。

# 6. Prompt 缓存（模型侧）

**结论先行**：部分厂商支持在模型侧缓存提示前缀，重复前缀按更低价格计费。设计提示时把稳定内容前置，可显著提升命中率。

## 6.1 原理

### 6.1.1 厂商差异

| 厂商 | 机制 |
|------|------|
| OpenAI | 自动缓存长前缀 |
| Anthropic | 需显式 `cache_control` 标记 |
| 其他 | 视厂商支持情况而定 |

### 6.1.2 提示布局

```java
// OpenAI 自动缓存长前缀；Anthropic 需显式标记 cache_control
// 关键：把稳定内容（system、知识）放前面，易变内容放后面
String prompt = """
        [稳定的系统提示与参考资料]      ← 可缓存
        ...
        [用户问题]                      ← 每次不同
        """;
```

## 6.2 命中率优化

| 手段 | 说明 |
|------|------|
| 稳定前置 | system / 知识放最前 |
| 减少变动 | 避免在系统提示里塞时间戳 |
| 前缀复用 | 同一模板跨请求共享前缀 |

## 6.3 边界与坑

- 前缀中插入动态内容（如当前时间）会让缓存整段失效。
- 缓存通常有最短长度门槛，过短前缀不生效。
- 计费与缓存时长因厂商而异，需实测确认。

> 生产建议：对 Prompt 缓存单独统计「缓存读取 Token 占比」，量化省钱效果。

# 7. 缓存一致性

**结论先行**：缓存与数据源必然存在时间差，关键是用主动失效、TTL 与版本号把不一致控制在可接受范围。

## 7.1 一致性问题

| 问题 | 对策 |
|------|------|
| 知识更新后缓存陈旧 | 知识变更时主动失效相关缓存 |
| 缓存与数据不一致 | 设置合理 TTL |
| 版本切换 | key 中带 Prompt / 模型版本 |

## 7.2 版本化 key

```java
// key 带版本，Prompt 变更后自动失效旧缓存
String key = "chat:v%s:%s:%s".formatted(promptVersion, tenantId, hash);
```

## 7.3 失效策略对比

| 策略 | 一致性 | 复杂度 |
|------|:------:|:------:|
| 纯 TTL | 弱 | 低 |
| 主动失效 | 强 | 中 |
| 版本号 | 强 | 低 |
| 事件驱动失效 | 强 | 高 |

## 7.4 边界与坑

- 主动失效要能定位「哪些缓存受影响」，否则只能全量清空。
- 多级缓存（本地 + Redis）失效要同时覆盖两层。
- 版本号方案会产生旧数据垃圾，需配合 TTL 清理。

> 生产建议：优先用「版本号 + TTL」组合，简单且足够可靠。

# 8. 缓存穿透与雪崩

**结论先行**：缓存三大经典故障——穿透、雪崩、击穿——在 LLM 场景同样成立，且因成本高而后果更严重。

## 8.1 三类故障

| 问题 | 说明 | 对策 |
|------|------|------|
| 穿透 | 大量不存在的 key 直击后端 | 空值缓存 + 布隆过滤器 |
| 雪崩 | 大量 key 同时过期 | TTL 加随机抖动 |
| 击穿 | 热点 key 过期瞬间高并发 | 互斥锁 + 逻辑过期 |

## 8.2 代码实现

### 8.2.1 TTL 抖动

```java
// TTL 抖动，避免同时过期
Duration ttl = Duration.ofMinutes(30)
        .plusSeconds(ThreadLocalRandom.current().nextInt(300));
```

### 8.2.2 空值缓存

```java
// 查询为空也缓存一个短 TTL 的哨兵值，防止穿透
redis.opsForValue().set(key, "__NULL__", Duration.ofMinutes(1));
```

## 8.3 边界与坑

- 空值缓存要设短 TTL，避免数据写入后仍返回空。
- 布隆过滤器有误判率，且不支持删除。
- 击穿的互斥锁要设超时，避免死锁。

> 生产建议：对三类故障分别配置监控指标，并做故障演练。

# 9. 缓存指标

**结论先行**：没有指标就无法判断缓存是否有效，也无法发现误命中与雪崩。

## 9.1 核心指标

```java
registry.counter("cache.hit", "type", "semantic").increment();
registry.counter("cache.miss", "type", "semantic").increment();
```

| 指标 | 说明 |
|------|------|
| 命中率 | 目标 > 30%（热点场景更高） |
| 平均节省 Token | 衡量收益 |
| 误命中率 | 语义缓存需监控 |
| 缓存大小 | 容量规划 |
| 淘汰率 | 判断容量是否不足 |

## 9.2 指标采集示例

```java
public Optional<String> getWithMetrics(String q) {
    Optional<String> hit = semanticCache.get(q);
    if (hit.isPresent()) {
        registry.counter("cache.hit", "type", "semantic").increment();
    } else {
        registry.counter("cache.miss", "type", "semantic").increment();
    }
    return hit;
}
```

## 9.3 边界与坑

- 只看命中率不看收益，可能缓存了低价值内容。
- 误命中率无法自动计算，需抽样或人工标注。
- 指标维度要带 `type`，否则多层缓存混在一起无法定位。

> 生产建议：命中率与节省 Token 同时监控，二者背离说明缓存策略需调整。

# 10. 生产注意事项

**结论先行**：缓存的坑集中在 key 设计、阈值、失效与隐私四方面，任何一处疏忽都会放大后果。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| key 不含租户 | 串数据风险 |
| 语义缓存阈值过低 | 误命中 |
| TTL 无抖动 | 雪崩 |
| 知识更新不失效 | 陈旧答案 |
| 缓存个性化内容 | 隐私泄露 |
| 无版本控制 | 版本切换后脏数据 |
| 无命中率监控 | 无法评估收益 |

## 10.2 落地检查表

- 所有缓存 key 是否含租户与版本？
- 语义缓存阈值是否经过验证？
- TTL 是否带随机抖动？
- 知识更新是否有失效链路？
- 是否监控命中率、误命中与节省 Token？

> 生产建议：缓存上线前必须完成「隐私审查 + 误命中抽样」两道门禁。

# 11. 小结

- 缓存分五层：结果、语义、检索、向量、Prompt。
- 结果缓存最省但最需谨慎；检索与向量缓存最安全。
- 语义缓存有误命中风险，阈值要保守。
- key 必须含租户与版本。
- 提示设计「稳定前置」以提升 Prompt 缓存命中。
- 穿透、雪崩、击穿需分别设防。
