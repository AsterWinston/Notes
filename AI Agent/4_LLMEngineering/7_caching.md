# 1. 缓存层次与价值

## 1.1 五层缓存全景

结论先行：LLM 应用的缓存不是单一开关，而是从"最终答案"到"模型内部前缀"的五层结构；越靠近结果层越省成本，但正确性风险也越高。

机制：一次用户请求自上而下依次经过五层，前三层由应用侧完全控制，第五层由模型厂商侧透明实现。任何一层命中都会短路其后的全部计算。

### 1.1.1 各层定义

```
① 结果缓存    相同问题直接返回答案        应用侧  最省，风险最高
② 语义缓存    相似问题返回缓存答案        应用侧  省，有误命中风险
③ 检索缓存    相同查询返回检索结果        应用侧  安全
④ 向量缓存    相同文本不重复向量化        应用侧  安全
⑤ Prompt 缓存 模型侧缓存提示前缀          厂商侧  透明
```

### 1.1.2 命中路径

```
用户问题
   │
   ▼
[① 结果缓存] ──命中──► 返回
   │ 未命中
   ▼
[② 语义缓存] ──命中──► 返回
   │ 未命中
   ▼
[④ 向量缓存] ◄──► [③ 检索缓存]
   │ 未命中
   ▼
[⑤ Prompt 缓存] ──► 模型生成 ──► 逐级回填
```

## 1.2 收益与代价

### 1.2.1 成本收益

| 层次 | 典型命中率 | 单次节省 | 主要代价 |
|------|:----------:|:--------:|----------|
| 结果缓存 | 10%~30% | 一次完整调用 | 陈旧、串租户 |
| 语义缓存 | 20%~40% | 一次完整调用 | 误命中 |
| 检索缓存 | 30%~60% | 一次检索 | 索引更新失效 |
| 向量缓存 | 50%~90% | 一次 embedding | 内存占用 |
| Prompt 缓存 | 视前缀而定 | 输入 Token | 前缀需稳定 |

收益评估公式（近似）：

```
单次节省成本 = 命中率 × 单次调用成本
净收益      = 单次节省成本 - 缓存维护成本 - 误命中返工成本
```

结论：只有当命中率足够高、且误命中返工成本可控时，缓存才真正划算。

### 1.2.2 风险总览

结论：缓存收益与风险成正比。结果层最省但最容易出错，向量层最安全，落地时应从安全层做起。

> 详细策略见 `14_ProductionEngineering/Performance/2_caching.md`，本文聚焦 LLM 工程实现。

# 2. 结果缓存

## 2.1 精确匹配结果缓存

### 2.1.1 基础实现

结论先行：结果缓存投入产出比最高，但只在"问题到答案"映射稳定时才安全。

机制：对归一化后的问题计算 key，命中即返回；未命中则调用模型并回填。

```java
@Service
public class CachedChatService {

    private final ChatClient chatClient;
    private final StringRedisTemplate redis;
    private final MeterRegistry metrics;

    public String chat(String question, UserContext user) {
        String key = cacheKey(question, user);

        String cached = redis.opsForValue().get(key);
        if (cached != null) {
            metrics.counter("cache.hit", "type", "result").increment();
            return cached;
        }
        metrics.counter("cache.miss", "type", "result").increment();

        String answer = chatClient.prompt().user(question).call().content();

        if (isCacheable(question)) {
            redis.opsForValue().set(key, answer, jitteredTtl());
        }
        return answer;
    }

    private String cacheKey(String question, UserContext user) {
        // key 必须含租户，避免串数据
        return "chat:%s:%s".formatted(user.tenantId(), sha256(normalize(question)));
    }

    private String normalize(String q) {
        return q.strip().replaceAll("\\s+", " ").toLowerCase();
    }
}
```

### 2.1.2 key 设计

| 要素 | 是否入 key | 说明 |
|------|:----------:|------|
| 归一化问题 | 是 | 去空白、统一大小写 |
| 租户 ID | 是 | 隔离数据，必须 |
| Prompt 版本 | 是 | 模板变更自动失效 |
| 模型名 | 是 | 不同模型答案不同 |
| 用户 ID | 视情况 | 个性化场景必须 |
| 时间戳 | 否 | 用 TTL 替代 |

结论：key 每少一个隔离维度，就多一类串数据事故。

```java
// 可复用的 key 构造器，统一拼装隔离维度
public String buildKey(CacheScope scope, String tenantId,
                       String promptVersion, String model, String payload) {
    return "%s:v%s:%s:%s:%s".formatted(
            scope.name().toLowerCase(), promptVersion,
            tenantId, model, sha256(payload));
}
```

## 2.2 可缓存判断

### 2.2.1 判断规则

结论先行：不是所有问题都能缓存，"能否缓存"必须在写入前判断，而不是命中后补救。

机制：把业务约束抽象为四类否决条件，任一命中即拒绝写入。

```java
public boolean isCacheable(String question) {
    // 不可缓存的场景
    if (containsPersonalInfo(question)) return false;      // 含隐私
    if (isTimeSensitive(question)) return false;           // 时效性
    if (requiresPersonalization(question)) return false;   // 需个性化
    if (hasSideEffect(question)) return false;             // 有副作用
    return true;
}
```

### 2.2.2 边界与反例

| 可缓存 | 不可缓存 |
|--------|----------|
| 通用知识问答 | 个性化推荐 |
| 政策解释 | 实时数据 |
| 固定流程说明 | 含隐私 |
| 静态摘要 | 有副作用操作 |

反例：把"今天天气如何"缓存一小时，用户下午再问会拿到上午的答案；把"帮我下单"缓存，则可能重复下单。这类问题即便文本完全相同，也绝不能命中结果缓存。

生产建议：把 `isCacheable` 做成可配置规则集，新增业务时先过"时效性、隐私、个性化、副作用"四问。

# 3. 语义缓存

## 3.1 语义缓存机制

### 3.1.1 向量相似检索

结论先行：语义缓存用向量相似度替代字符串相等，能覆盖同义改写，但引入了误命中这一全新风险类别。

机制：把问题向量化后写入向量库，查询时检索最近邻，若相似度超过阈值则返回其答案。

### 3.1.2 阈值选择

| 阈值 | 效果 |
|:----:|------|
| 0.99 | 几乎只有完全相同命中 |
| 0.95 | 高度相似命中（推荐） |
| 0.90 | 较宽松，有误命中风险 |
| <0.90 | 不推荐，误命中率高 |

结论：阈值越低保命率越高，但误命中率也越高；生产建议从 0.95 起步，用线上误命中反馈微调。

## 3.2 语义缓存实现

### 3.2.1 读取实现

```java
@Service
public class SemanticCache {

    private final EmbeddingModel embeddingModel;
    private final EmbeddingStore<TextSegment> store;
    private final Map<String, String> answers;

    public Optional<String> get(String question, String tenantId) {
        Embedding qe = embeddingModel.embed(question).content();

        List<EmbeddingMatch<TextSegment>> matches = store.search(
                EmbeddingSearchRequest.builder()
                        .queryEmbedding(qe)
                        .maxResults(1)
                        .minScore(0.95)                     // 高阈值
                        .filter(metadataKey("tenantId").isEqualTo(tenantId))
                        .build()).matches();

        if (matches.isEmpty()) return Optional.empty();

        String id = matches.get(0).embedded().metadata().get("id").toString();
        return Optional.ofNullable(answers.get(id));
    }
}
```

写入侧与读取侧使用同一 embedding 模型，并显式记录写入时间与租户：

```java
public void put(String question, String answer, String tenantId) {
    String id = UUID.randomUUID().toString();
    Embedding e = embeddingModel.embed(question).content();
    store.add(e, TextSegment.from(question,
            Metadata.from(Map.of("id", id, "tenantId", tenantId))));
    answers.put(id, answer);
}
```

### 3.2.2 误命中防护

结论：语义缓存的头号风险是"语义相近但答案不同"，必须用多重手段收敛。

- 阈值保守：从 0.95 起，宁可少命中。
- 租户隔离：`filter` 中强制 `tenantId`，杜绝跨租户命中。
- 写入审慎：含数字、否定词、条件的问题不写入。
- 双校验：高相似度时再用小模型判断是否同一意图。

> **语义缓存有误命中风险**，阈值要保守，且必须按租户隔离。

# 4. 检索缓存与向量缓存

## 4.1 检索结果缓存

### 4.1.1 实现

结论先行：检索缓存是"安全缓存"，因为检索结果由知识库决定，不涉及生成歧义。

```java
@Service
public class CachedRetriever {

    private final Retriever retriever;
    private final StringRedisTemplate redis;

    public List<Document> retrieve(String query, String tenantId) {
        String key = "retrieval:%s:%s".formatted(tenantId, sha256(query));
        String cached = redis.opsForValue().get(key);
        if (cached != null) {
            return JsonUtils.readList(cached, Document.class);
        }
        List<Document> docs = retriever.retrieve(query);
        redis.opsForValue().set(key, JsonUtils.write(docs), jitteredTtl());
        return docs;
    }
}
```

### 4.1.2 失效

结论：检索缓存必须在知识库变更时失效，否则会持续返回过期文档。

生产建议：知识更新事件中按租户或文档 ID 删除相关 key，或直接在 key 中携带索引版本号。

## 4.2 向量缓存

### 4.2.1 实现

```java
@Service
public class CachedEmbeddingModel {

    private final EmbeddingModel delegate;
    private final Cache<String, Embedding> cache = Caffeine.newBuilder()
            .maximumSize(100_000)
            .expireAfterAccess(Duration.ofHours(6))
            .recordStats()
            .build();

    public Embedding embed(String text) {
        return cache.get(sha256(text), k -> delegate.embed(text).content());
    }

    public List<Embedding> embedAll(List<String> texts) {
        return texts.stream().map(this::embed).toList();
    }
}
```

### 4.2.2 增量索引复用

结论：向量缓存的最大价值在批量索引场景——未变更文档的向量可直接复用。

> 增量索引时，**未变更文档的向量可直接复用**，大幅降低成本。

边界：向量与 embedding 模型绑定，换模型后缓存必须整体失效，因此 key 中应包含模型标识。

| 场景 | 复用向量 | 说明 |
|------|:--------:|------|
| 文档内容未变 | 是 | 直接命中 |
| 文档轻微变更 | 否 | 重新向量化 |
| 换 embedding 模型 | 否 | 全部重建 |
| 新增文档 | 否 | 仅新文档计算 |

# 5. Prompt 缓存

## 5.1 厂商侧前缀缓存

### 5.1.1 OpenAI 自动缓存

结论先行：部分厂商会自动缓存稳定前缀，命中后输入 Token 成本显著下降，但要求前缀足够长且完全一致。

机制：服务端对请求的前缀做哈希，若与近期请求前缀一致则复用其 KV 缓存，跳过重复的注意力计算。

### 5.1.2 Anthropic 显式标记

```
厂商侧缓存提示前缀：
- OpenAI：自动缓存长前缀（约 1024 token 以上）
- Anthropic：显式标记 cache_control
```

机制：缓存以"前缀"为单位，任何一处字节变化都会导致其后全部失效。

## 5.2 Prompt 结构优化

### 5.2.1 稳定前置

结论：把稳定内容放前面、易变内容放后面，是 Prompt 缓存的唯一结构原则。

```java
String prompt = """
        [系统提示与知识库]    ← 稳定，可缓存
        ...
        [用户问题]            ← 每次不同
        """;
```

### 5.2.2 边界与坑

反例：在系统提示中插入当前时间戳或请求 ID，会让前缀每次都不同，缓存命中率直接归零。

生产建议：把时间、用户名、随机数等动态字段统一放到消息末尾或 user 消息中。

# 6. 缓存失效

## 6.1 主动失效

### 6.1.1 事件驱动

结论先行：主动失效保证"知识一变、缓存即清"，是避免陈旧答案的关键。

机制：监听领域事件，按租户或文档维度批量删除相关 key。

```java
// ① 知识更新时主动失效
@EventListener
public void onKnowledgeUpdated(KnowledgeUpdatedEvent event) {
    redis.delete(pattern("chat:%s:*".formatted(event.tenantId())));
}
```

### 6.1.2 版本号失效

```java
// ② key 带版本，Prompt 变更后自动失效
String key = "chat:v%s:%s:%s".formatted(promptVersion, tenantId, hash);
```

结论：版本号是最优雅的失效方式，无需删除历史数据，新版本自然 miss。

## 6.2 被动失效

### 6.2.1 TTL 与抖动

```java
// ③ TTL 加抖动，防雪崩
Duration ttl = Duration.ofMinutes(30)
        .plusSeconds(ThreadLocalRandom.current().nextInt(300));
```

### 6.2.2 雪崩防护

结论：固定 TTL 会让大批 key 同时过期，形成瞬时流量洪峰。

生产建议：所有 TTL 统一叠加随机抖动；热点数据加长 TTL 或永不过期并配合主动失效。

# 7. 穿透、击穿与雪崩

## 7.1 穿透与击穿

### 7.1.1 概念区分

| 问题 | 原因 | 表现 |
|------|------|------|
| 穿透 | 查询不存在的数据 | 每次都打到后端 |
| 击穿 | 热点 key 过期瞬间 | 并发全部打到后端 |
| 雪崩 | 大批 key 同时过期 | 后端整体过载 |

三者常被混用，但根因不同：穿透是"数据本不存在"，击穿是"单点热点失效"，雪崩是"集体失效"。

### 7.1.2 互斥锁防击穿

```java
public String chatWithCache(String question, UserContext user) {
    String key = cacheKey(question, user);

    String cached = redis.opsForValue().get(key);
    if (cached != null) return cached;

    // 互斥锁，防击穿
    RLock lock = redisson.getLock("cache:lock:" + key);
    if (!lock.tryLock(3, 10, TimeUnit.SECONDS)) {
        throw new TooManyRequestsException("请求处理中");
    }
    try {
        // 二次检查
        cached = redis.opsForValue().get(key);
        if (cached != null) return cached;

        String answer = doChat(question, user);
        redis.opsForValue().set(key, answer, jitteredTtl());
        return answer;
    } finally {
        lock.unlock();
    }
}
```

结论：拿到锁的线程负责回填，其余线程快速失败或短暂等待，避免重复计算。

## 7.2 雪崩与穿透防护

### 7.2.1 抖动与预热

生产建议：抖动 TTL、启动预热热点 key、必要时用逻辑过期（value 中带过期时间，异步刷新）。

### 7.2.2 空值缓存防穿透

结论：对确定不存在的结果写入短 TTL 空值，可防止穿透反复打后端。

# 8. 多级缓存架构

## 8.1 本地加分布式

### 8.1.1 两级结构

```
请求 ──► [Caffeine 本地] ──miss──► [Redis 分布式] ──miss──► 模型
              ▲                          ▲
              └──── 回填 ────────────────┘
```

### 8.1.2 一致性

结论：本地缓存与分布式缓存存在一致性窗口，须用短 TTL 或广播失效消息收敛。

```java
// 两级查询：本地未命中再查 Redis，并回填本地
public Optional<String> get(String key) {
    String local = localCache.getIfPresent(key);
    if (local != null) return Optional.of(local);
    String remote = redis.opsForValue().get(key);
    if (remote != null) localCache.put(key, remote);
    return Optional.ofNullable(remote);
}
```

## 8.2 选型对比

### 8.2.1 存储对比

| 维度 | Caffeine | Redis |
|------|----------|-------|
| 位置 | JVM 内 | 独立服务 |
| 延迟 | 纳秒级 | 亚毫秒级 |
| 容量 | 受限 | 大 |
| 多实例一致 | 否 | 是 |
| 适用 | 向量、热点 | 结果、语义 |

### 8.2.2 适用场景

结论：向量缓存放本地，结果与语义缓存放 Redis，两级组合兼顾延迟与容量。

# 9. 缓存指标与监控

## 9.1 核心指标

### 9.1.1 命中率

```java
registry.counter("cache.hit", "type", "result").increment();
registry.counter("cache.miss", "type", "result").increment();
registry.counter("cache.hit", "type", "semantic").increment();
registry.counter("cache.hit", "type", "embedding").increment();
```

### 9.1.2 指标目标

| 指标 | 目标 |
|------|------|
| 命中率 | > 30%（热点场景更高） |
| 误命中率 | 语义缓存需监控 |
| 节省 Token | 衡量收益 |
| 平均延迟下降 | 验证体验 |

## 9.2 监控告警

### 9.2.1 采集维度

按租户、模型、缓存类型分别打点，便于定位问题租户。

### 9.2.2 告警规则

结论：命中率骤降、误命中上升、缓存延迟升高都应触发告警。

生产建议：把误命中反馈做成人工抽检通道，定期评估语义缓存质量。

# 10. 生产注意事项与反模式

## 10.1 风险清单

### 10.1.1 风险表

| 风险 | 建议 |
|------|------|
| key 不含租户 | 串数据 |
| 语义阈值过低 | 误命中 |
| TTL 无抖动 | 雪崩 |
| 知识更新不失效 | 陈旧答案 |
| 缓存个性化内容 | 隐私 |
| 无版本控制 | 脏数据 |
| 无指标 | 无法评估收益 |

### 10.1.2 反模式

- 缓存一切：把有副作用、实时、隐私内容也缓存。
- 只设 TTL 不失效：知识更新后仍返回旧答案。
- 语义阈值拍脑袋：未经评测直接设 0.8。

## 10.2 生产建议

### 10.2.1 检查清单

- key 是否含租户与版本？
- TTL 是否带抖动？
- 是否有主动失效？
- 是否监控命中率与误命中率？

### 10.2.2 落地顺序

结论：先做向量缓存与检索缓存（低风险），再做结果缓存，最后做语义缓存（高风险）。

# 11. 小结

## 11.1 核心要点

- 五层缓存：结果、语义、检索、向量、Prompt。
- 结果缓存最省但需谨慎；检索与向量缓存最安全。
- 语义缓存阈值要保守，必须按租户隔离。
- key 带租户与版本，TTL 加抖动。
- 必须监控命中率与误命中率。

## 11.2 落地清单

- 先低风险层，后高风险层。
- 主动失效、版本号、TTL 抖动三件套齐备。
- 每层独立打点，按租户监控。
- 语义缓存上线前先做离线误命中评测。
