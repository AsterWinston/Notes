# 1. 三种记忆类型

借鉴认知科学，Agent 记忆分为三类：情景记忆、语义记忆、程序性记忆。

## 1.1 认知科学来源

**结论先行**：人类记忆不是单一存储，Agent 记忆同样应分类，因为三类记忆的写入时机、检索方式与生命周期完全不同。

### 1.1.1 三类定义

```
① 情景记忆（Episodic）   "发生过什么"   —— 具体事件与交互
② 语义记忆（Semantic）   "知道什么"     —— 事实与知识
③ 程序性记忆（Procedural）"怎么做"      —— 技能与流程
```

### 1.1.2 对比表

| 类型 | 内容 | 例子 | 存储 |
|------|------|------|------|
| 情景 | 具体事件 | "上次用户问过报销，我答了 30 天" | 时间序列 |
| 语义 | 事实知识 | "用户是后端工程师" | 结构化/向量 |
| 程序性 | 技能流程 | "这类问题应该先查政策再回答" | 规则/提示 |

## 1.2 三种记忆的关系

### 1.2.1 协作

**结论先行**：三者不是互斥，而是从"具体经验"到"抽象规律"再到"行动策略"的递进。

```
情景（具体） ──归纳──> 语义（规律） ──提炼──> 程序性（行动）
```

### 1.2.2 边界

- 情景记忆回答"当时发生了什么"，语义记忆回答"普遍是什么"，程序性记忆回答"应该怎么做"。
- 同一信息可能在不同层级出现，需明确主存储，避免三处都写导致冲突。

# 2. 情景记忆

## 2.1 定义与结构

**结论先行**：情景记忆记录"发生了什么"，按时间组织，包含事件、结果、所用工具。

### 2.1.1 record 定义

```java
public record EpisodicMemory(
        String id,
        String userId,
        String sessionId,
        String event,                  // 事件描述
        String outcome,                // 结果
        List<String> toolsUsed,        // 用到的工具
        Instant timestamp,
        double importance
) {}
```

### 2.1.2 记录时机

| 时机 | 是否记录 | 说明 |
|------|----------|------|
| 任务成功完成 | 是 | 可复用经验 |
| 任务失败 | 是（低分） | 避免重蹈 |
| 普通寒暄 | 否 | 无价值 |
| 用户纠正 | 是 | 高价值 |

## 2.2 记录代码

### 2.2.1 同步记录实现

**结论先行**：记录应在任务完成后立即写入，但写入动作要与主流程解耦。

```java
// 记录一次成功的问题解决
public void recordEpisode(UserContext user, AgentTrace trace) {
    if (trace.status() != COMPLETED) return;

    store.save(new EpisodicMemory(
            UUID.randomUUID().toString(),
            user.userId(),
            trace.sessionId(),
            "用户询问：" + trace.goal(),
            "解决方案：" + trace.finalAnswer(),
            trace.steps().stream().map(Step::toolName).toList(),
            Instant.now(),
            trace.wasSuccessful() ? 0.8 : 0.5));
}
```

### 2.2.2 异步与幂等

```java
public void recordAsync(UserContext user, AgentTrace trace) {
    if (trace.status() != COMPLETED) return;

    // 幂等键：sessionId + goal 哈希，避免重复记录
    String key = trace.sessionId() + ":" + trace.goal().hashCode();
    if (dedupCache.putIfAbsent(key, true) != null) return;

    episodeQueue.offer(toMemory(user, trace));   // 异步消费落库
}
```

## 2.3 检索

情景记忆的检索通常"时间倒序 + 语义相似"结合：先按用户过滤，再按相似度召回最近若干条。

### 2.3.1 时间倒序

```java
public List<EpisodicMemory> recent(String userId, int limit) {
    return store.findByUser(userId).stream()
            .sorted(Comparator.comparing(EpisodicMemory::timestamp).reversed())
            .limit(limit)
            .toList();
}
```

### 2.3.2 语义相似

```java
public List<EpisodicMemory> findSimilar(String userId, String query, int k) {
    return episodicStore.search(
            EmbeddingSearchRequest.builder()
                    .queryEmbedding(embeddingModel.embed(query).content())
                    .maxResults(k)
                    .filter(metadataKey("userId").isEqualTo(userId))
                    .build()).matches().stream()
            .map(m -> toEpisodic(m.embedded()))
            .sorted(Comparator.comparing(EpisodicMemory::timestamp).reversed())
            .toList();
}
```

## 2.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 记录所有事件 | 噪声大 | 只记成功/失败/纠正 |
| 无结果字段 | 无法复用 | 必存 outcome |
| 无时间排序 | 召回旧事件 | 时间衰减 |
| 原样存全文 | 成本高 | 摘要 |

# 3. 语义记忆

## 3.1 定义与结构

**结论先行**：语义记忆记录"知道的事实"，与时间无关，通常用三元组表示。

### 3.1.1 record 定义

```java
public record SemanticMemory(
        String id,
        String userId,
        String subject,                // 主体
        String predicate,              // 关系
        String object,                 // 客体
        double confidence,
        Instant updatedAt
) {}
```

### 3.1.2 三元组示例

```
(用户, 职位, 后端工程师)
(用户, 偏好回答风格, 简洁)
(用户公司, 技术栈, Java 21)
```

## 3.2 存储与检索

**结论先行**：语义记忆同时写结构化表（按 subject/predicate 精确查）与向量库（按语义查），两者互补。

### 3.2.1 双写实现

```java
public void save(SemanticMemory memory) {
    semanticTable.upsert(memory);                       // 结构化
    TextSegment seg = TextSegment.from(
            "%s %s %s".formatted(memory.subject(),
                    memory.predicate(), memory.object()),
            Metadata.from(Map.of("userId", memory.userId())));
    vectorStore.add(embeddingModel.embed(seg).content(), seg);
}
```

### 3.2.2 两类检索

```java
// 精确查：主体 + 关系
public Optional<SemanticMemory> findExact(String userId, String subject, String predicate) {
    return semanticTable.find(userId, subject, predicate);
}

// 语义查：自然语言问题
public List<SemanticMemory> findRelevant(String userId, String question, int k) {
    return vectorStore.search(EmbeddingSearchRequest.builder()
                    .queryEmbedding(embeddingModel.embed(question).content())
                    .maxResults(k)
                    .filter(metadataKey("userId").isEqualTo(userId))
                    .build()).matches().stream()
            .map(m -> toSemantic(m.embedded()))
            .toList();
}
```

用途：

- 用户画像
- 事实性知识
- 个性化回答

## 3.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 三元组过窄 | 复杂事实难表达 | 允许 object 为句子 |
| 无置信度 | 谣言当事实 | confidence 过滤 |
| 矛盾并存 | 新旧冲突 | 以新为准 |
| 只写一处 | 检索不全 | 结构化 + 向量 |

# 4. 程序性记忆

## 4.1 定义与结构

**结论先行**：程序性记忆记录"怎么做"，通常是触发条件 + 处理流程的规则。

```java
public record ProceduralMemory(
        String id,
        String userId,                 // 可为 null（全局）
        String trigger,                // 触发条件
        String procedure,              // 处理流程
        double successRate,
        int useCount
) {}
```

```
触发：用户询问报销问题
流程：先查《报销政策》文档 → 提取关键条款 → 附来源回答

触发：用户表达不满
流程：先致歉 → 询问具体问题 → 提供解决方案 → 必要时转人工
```

用途：

- 复用成功的处理模式
- 避免重复试错
- 形成"经验"

## 4.2 触发匹配

### 4.2.1 语义匹配

```java
public List<ProceduralMemory> match(String userId, String question) {
    Embedding q = embeddingModel.embed(question).content();
    return proceduralStore.search(
            EmbeddingSearchRequest.builder()
                    .queryEmbedding(q)
                    .maxResults(3)
                    .minScore(0.75)
                    .build()).matches().stream()
            .map(m -> toProcedural(m.embedded()))
            .filter(p -> p.userId() == null || p.userId().equals(userId))
            .toList();
}
```

### 4.2.2 规则匹配兜底

```java
// 关键词规则优先，命中则直接返回，避免语义误匹配
public Optional<ProceduralMemory> matchByRule(String userId, String question) {
    return ruleTable.findByUser(userId).stream()
            .filter(r -> question.contains(r.keyword()))
            .findFirst();
}
```

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 流程未验证 | 错误经验固化 | 多次成功才固化 |
| 触发过宽 | 到处命中 | 提高匹配阈值 |
| 无成功率 | 无法淘汰 | 记录 successRate |
| 全局与私有混淆 | 越权/污染 | userId 区分 |

# 5. 三者的协作

## 5.1 协作流程

**结论先行**：回答前依次查程序性（怎么做）、语义（知道什么）、情景（以前怎么做），组装成上下文。

### 5.1.1 顺序

```
用户提问
   │
   ▼
① 程序性记忆：有没有处理这类问题的流程？
   │
   ▼
② 语义记忆：关于用户/主题我知道什么？
   │
   ▼
③ 情景记忆：以前类似情况是怎么处理的？
   │
   ▼
组装上下文 → 模型生成
```

### 5.1.2 组装原则

- 程序性记忆给"方法"，语义记忆给"事实"，情景记忆给"参考"。
- 三者在提示中分区标注，避免模型混淆。

## 5.2 代码

```java
public String buildMemoryContext(UserContext user, String question) {
    StringBuilder sb = new StringBuilder();

    // ① 程序性
    List<ProceduralMemory> procedures = proceduralStore.match(
            user.userId(), question);
    if (!procedures.isEmpty()) {
        sb.append("【建议处理流程】\n")
          .append(procedures.get(0).procedure()).append("\n\n");
    }

    // ② 语义
    List<SemanticMemory> facts = semanticStore.findRelevant(
            user.userId(), question, 5);
    if (!facts.isEmpty()) {
        sb.append("【已知信息】\n")
          .append(formatFacts(facts)).append("\n\n");
    }

    // ③ 情景
    List<EpisodicMemory> episodes = episodicStore.findSimilar(
            user.userId(), question, 3);
    if (!episodes.isEmpty()) {
        sb.append("【类似历史】\n")
          .append(formatEpisodes(episodes)).append("\n\n");
    }

    return sb.toString();
}
```

## 5.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 顺序颠倒 | 事实淹没方法 | 程序性优先 |
| 不分区标注 | 模型混淆 | 明确标题 |
| 全量注入 | 超预算 | 每类限量 |
| 无用户过滤 | 越权 | userId 过滤 |

# 6. 程序性记忆的生成

## 6.1 从成功经验提炼

**结论先行**：程序性记忆不应凭空产生，而应从多次成功的执行轨迹中归纳。

```java
interface ProcedureLearner {
    @UserMessage("""
            分析下面成功的任务执行轨迹，提炼一个可复用的处理流程。

            任务：{{goal}}
            执行轨迹：{{trace}}
            结果：成功

            输出：
            {"trigger":"触发条件","procedure":"处理流程"}
            """)
    LearnedProcedure learn(@V("goal") String goal,
                           @V("trace") String trace);
}
```

## 6.2 固化条件

### 6.2.1 固化阈值

| 条件 | 阈值建议 | 理由 |
|------|----------|------|
| 相似成功次数 | ≥ 3 | 排除偶然 |
| 成功率 | ≥ 0.8 | 保证可靠 |
| 流程长度 | 适中 | 过长难复用 |

### 6.2.2 固化实现

```java
// 多次成功后固化
public void maybeLearn(UserContext user, AgentTrace trace) {
    if (!trace.wasSuccessful()) return;

    List<AgentTrace> similar = findSimilarSuccessful(trace, 3);
    if (similar.size() >= 3) {                     // 多次成功才固化
        LearnedProcedure proc = learner.learn(trace.goal(), trace.toString());
        proceduralStore.save(toMemory(user, proc));
    }
}
```

## 6.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 一次成功就固化 | 偶然当规律 | 多次成功 |
| 提炼过泛 | 无指导意义 | 限制触发条件 |
| 从不淘汰 | 旧流程过期 | successRate 淘汰 |
| 未人工复核 | 错误流程上线 | 灰度/复核 |

# 7. 记忆类型的转换

## 7.1 转换路径

**结论先行**：情景记忆多次重复可归纳为语义记忆，语义规律多次验证可提炼为程序性记忆。

```
情景记忆（多次相似事件）
        │ 归纳
        ▼
语义记忆（事实/规律）

语义记忆（多次验证的规律）
        │ 提炼
        ▼
程序性记忆（处理流程）
```

## 7.2 归纳代码

### 7.2.1 情景归纳为语义

```java
@Scheduled(cron = "0 0 3 * * SUN")
public void consolidate() {
    // 每周归纳：把重复出现的情景记忆归纳为语义记忆
    Map<String, List<EpisodicMemory>> grouped = groupSimilarEpisodes();

    grouped.forEach((pattern, episodes) -> {
        if (episodes.size() >= 5) {
            SemanticMemory semantic = induce(episodes);
            semanticStore.save(semantic);
        }
    });
}
```

### 7.2.2 语义提炼为程序性

```java
// 当同一语义规律被多次验证（confidence 高、引用次数多）时提炼为流程
public void refineToProcedural() {
    semanticStore.findHighConfidence(0.9).forEach(semantic -> {
        if (semantic.supportCount() >= 5) {
            LearnedProcedure proc = learner.learn(semantic.content(), "多次验证");
            proceduralStore.save(toMemory(null, proc));   // 全局流程
        }
    });
}
```

## 7.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 归纳过早 | 样本不足 | 设最小样本数 |
| 不删原记忆 | 重复冗余 | 标记已归纳 |
| 归纳错误 | 以偏概全 | 置信度阈值 |
| 从不归纳 | 情景堆积 | 定期 consolidation |

# 8. 各类型的检索方式

## 8.1 检索对照

**结论先行**：三类记忆的检索键不同，必须分别设计，不能统一用向量。

| 类型 | 检索 | 说明 |
|------|------|------|
| 情景 | 时间 + 语义 | 按时间倒序 + 相似度 |
| 语义 | 结构化 + 语义 | 按主体/关系 + 向量 |
| 程序性 | 触发条件匹配 | 规则匹配 + 语义 |

## 8.2 混合检索

### 8.2.1 聚合代码

```java
public MemoryContext retrieve(UserContext user, String question) {
    return new MemoryContext(
            proceduralStore.match(user.userId(), question),
            semanticStore.findRelevant(user.userId(), question, 5),
            episodicStore.findSimilar(user.userId(), question, 3));
}
```

### 8.2.2 排序与裁剪

```java
public String render(MemoryContext ctx, int tokenBudget) {
    StringBuilder sb = new StringBuilder();
    // 程序性给 1 条即可，语义给 5 条，情景给 3 条，按预算裁剪
    appendSection(sb, "建议处理流程", ctx.procedures(), 1);
    appendSection(sb, "已知信息", ctx.facts(), 5);
    appendSection(sb, "类似历史", ctx.episodes(), 3);
    return truncateToBudget(sb.toString(), tokenBudget);
}
```

## 8.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 统一用向量 | 精确事实查不准 | 结构化兜底 |
| 不做用户过滤 | 越权 | 强制 userId |
| 不做类型区分 | 结果混杂 | 分区返回 |
| 无排序策略 | 质量不稳 | 分类内排序 |

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 混淆类型 | 明确分类 |
| 只记情景 | 归纳为语义/程序性 |
| 程序性未验证 | 多次成功才固化 |
| 语义记忆冲突 | 以新为准 + 置信度 |
| 无归纳 | 定期 consolidation |
| 检索不加类型 | 按类型分别检索 |

## 9.2 监控

### 9.2.1 指标

| 指标 | 含义 | 告警 |
|------|------|------|
| 各类型记忆量 | 结构健康 | 情景暴涨 |
| 归纳产出数 | consolidation 有效性 | 长期为 0 |
| 程序性成功率 | 质量 | 下降 |
| 检索命中率 | 覆盖 | 过低 |

### 9.2.2 告警建议

- 情景记忆量持续增长而语义/程序性不增：说明归纳任务失效。
- 程序性成功率跌破阈值：应自动停用并转人工复核。

## 9.3 架构建议

- 三类记忆分表/分集合存储，统一通过 `MemoryContext` 聚合。
- consolidation 任务离线跑，避免影响在线。
- 程序性记忆上线前建议人工复核或灰度。

# 10. 小结

- 三类记忆：情景（发生什么）、语义（知道什么）、程序性（怎么做）。
- 情景按时间，语义按事实，程序性按触发条件。
- 三者协作组装上下文。
- 记忆可从情景归纳为语义，再提炼为程序性。
- 分类存储与检索，避免混淆。
