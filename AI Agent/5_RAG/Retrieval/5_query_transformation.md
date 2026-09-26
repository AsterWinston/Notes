# 1. 为什么改写查询

**结论先行**：检索系统匹配的是"查询"与"文档"，而用户输入往往不是一条好的查询。查询改写（Query Transformation，查询变换）的核心任务，是把口语化、带指代、多意图的自然语言问题，转换成检索友好的表述，从而缩小"用户表达"与"文档表述"之间的鸿沟。

## 1.1 用户表达与文档表述的鸿沟

### 1.1.1 口语化与指代

```
用户问："那个报销的，就是出差花的钱，怎么弄？"
问题：口语化、有指代、信息散乱
检索友好："差旅费报销流程"
```

口语化词汇（"咋弄""那个"）在文档里几乎不出现，向量与关键词都难以命中。

### 1.1.2 多意图与上下文依赖

一句"报销标准和流程是什么"其实包含两个意图；"那这个呢"完全依赖上文。直接检索会顾此失彼或完全跑偏。

### 1.1.3 表述不对称的数学直觉

检索本质上是最大化 `sim(q, d)`。若 `q` 是问句、`d` 是陈述句，二者在向量空间中的分布存在系统性偏移：

```
问题向量通常聚集在"疑问/祈使"区域
文档向量通常聚集在"陈述/说明"区域
两个区域重叠度低 → 直接匹配分数低
改写把 q 投影到文档区域 → 分数升高
```

这解释了为什么"把问题改写成陈述式查询"往往立竿见影。

## 1.2 常见问题类型

| 问题 | 例子 |
|------|------|
| 口语化 | "咋报销啊" |
| 指代不明 | "它怎么弄" |
| 多意图 | "报销标准和流程是什么" |
| 上下文依赖 | "那这个呢"（依赖前文） |
| 过短 | "报销" |
| 过长/冗余 | 粘贴一大段需求再问一句 |

## 1.3 改写带来的收益边界

改写能提升召回率，但**不能凭空创造知识**：若知识库没有相关内容，再好的改写也检索不到。此外，改写本身可能引入错误（丢实体、改语义），因此必须有评测兜底。

| 改写能解决 | 改写不能解决 |
|------------|--------------|
| 表述不匹配 | 知识库缺失 |
| 召回不全 | 分块不合理 |
| 指代悬空 | embedding 不适配 |
| 多意图混淆 | 排序错误 |

# 2. 四类查询改写技术

**结论先行**：查询重写、查询扩展、查询分解、HyDE 是四类互补技术，分别解决"表述不匹配""召回不全""多意图""问答不对称"四类问题。

## 2.1 技术总览

| 技术 | 作用 |
|------|------|
| 查询重写 | 口语 → 规范表述 |
| 查询扩展 | 一个查询 → 多个查询 |
| 查询分解 | 复杂查询 → 多个子查询 |
| HyDE | 生成假设答案再检索 |

## 2.2 各技术适用场景

| 技术 | 最适合 | 不适合 |
|------|--------|--------|
| 重写 | 多轮对话、口语输入 | 输入本已规范 |
| 扩展 | 同义表述多、召回不全 | 精确术语查询 |
| 分解 | 多意图复合问题 | 单一事实问答 |
| HyDE | 问题与文档表述差异大 | 事实性极强的精确查询 |

## 2.3 组合原则

**先便宜后昂贵、先简单后复杂**：先做重写（一次调用、一次检索），若召回仍不足，再叠加扩展或 HyDE。避免无脑全上导致成本与延迟失控。

```
查询复杂度评估
      │
  ┌───┴────┬─────────┬──────────┐
  ▼        ▼         ▼          ▼
简单      多轮      多意图     表述差异大
不处理    重写      重写+分解  重写+HyDE
```

# 3. 查询重写

**结论先行**：查询重写（Query Rewriting，查询重写）是最基础、性价比最高的一步，用一次 LLM 调用把问题规范化，并在多轮场景中消解指代。

## 3.1 机制

```
原始问题 + 对话历史
        │
        ▼
   LLM 改写（保留实体，去口语）
        │
        ▼
   规范查询 → 检索
```

多轮对话中，重写必须把"它""这个"替换为历史中出现的具体实体，否则单看当前问题无法检索。

## 3.2 LangChain4j 实现

```java
interface QueryRewriter {
    @UserMessage("""
            把用户问题改写为适合检索的查询语句。
            要求：保留关键信息，去掉口语化表达，输出一行查询。
            对话历史：{{history}}
            用户问题：{{question}}
            改写后：
            """)
    String rewrite(@V("history") String history, @V("question") String question);
}
```

> 多轮对话中，重写必须结合历史，解决指代问题。

若历史很长，应先做历史摘要再传入，避免提示词超长：

```java
String condensed = historySummarizer.summarize(rawHistory);
String query = rewriter.rewrite(condensed, question);
```

## 3.3 边界与坑

| 坑 | 说明 |
|----|------|
| 丢失关键实体 | 模型"顺口"改写时删掉专有名词 |
| 过度规范化 | 把用户特有的术语替换成通用词，反而失配 |
| 忽略历史 | 多轮场景不做指代消解，查询残缺 |
| 输出多余解释 | 未约束输出格式，返回"好的，改写为：..." |
| 语言漂移 | 中英混合输入被改写成单一语言，漏召回 |

## 3.4 生产建议

- 提示词显式要求"保留所有专有名词与数字"；
- 约束输出为**单行查询**，必要时用结构化输出；
- 改写前后都保留原查询，可同时检索融合，降低改写风险；
- 对改写结果做长度与内容校验，异常时回退原查询。

```java
String rewritten = rewriter.rewrite(history, question);
if (rewritten == null || rewritten.isBlank() || rewritten.length() > 200) {
    rewritten = question; // 校验失败回退
}
```

# 4. 查询扩展（多查询）

**结论先行**：查询扩展（Query Expansion，查询扩展）把一个查询变成多个语义相近的查询，分别检索后合并，目的是提升召回覆盖率，代价是更多 LLM 调用与检索次数。

## 4.1 机制

```
       原始查询
          │
   ┌──────┼──────┐
   ▼      ▼      ▼
 变体1  变体2  变体3
   │      │      │
   └──────┼──────┘
          ▼
   多路检索 → 合并去重
```

## 4.2 实现

### 4.2.1 LangChain4j AI Service

```java
interface QueryExpander {
    @UserMessage("""
            为下面的问题生成 3 个不同角度的检索查询，输出 JSON 数组：
            {{it}}
            """)
    List<String> expand(String question);
}
```

### 4.2.2 多路检索与合并

```java
public List<Document> multiQueryRetrieve(String question) {
    List<String> queries = new ArrayList<>();
    queries.add(question);
    queries.addAll(expander.expand(question));

    Set<Document> merged = new LinkedHashSet<>();
    for (String q : queries) {
        merged.addAll(vectorStore.similaritySearch(
                SearchRequest.builder().query(q).topK(10).build()));
    }
    return new ArrayList<>(merged);
}
```

**优点**：提升召回覆盖率；**缺点**：增加 LLM 调用与检索次数。

### 4.2.3 按 ID 去重与 RRF 融合

多路结果的正确合并方式是 RRF（Reciprocal Rank Fusion，倒数排名融合）而非简单并集：对每个文档累加各路 `1/(k+rank)` 分，再按总分排序。

## 4.3 边界与坑

- 变体高度重复：扩展没带来新视角，白花成本；
- 扩展过多引入噪声：召回池变大，稀释正确文档；
- 用 `Set` 去重依赖 `Document` 的 equals，若未实现会去重失效，应显式按 ID 去重；
- 合并后未重排：多路结果顺序混乱，需 RRF 或重排整理。

## 4.4 生产建议

- 限制扩展数量（2~3 个）；
- 提示词要求"不同角度"而非同义改写；
- 合并后接 RRF 融合或重排；
- 监控扩展查询与原始查询的**重合度**，过低说明扩展跑偏。

# 5. 查询分解

**结论先行**：查询分解（Query Decomposition，查询分解）把复杂问题拆成独立子问题分别检索，解决"单次检索只命中部分内容"的问题。

## 5.1 机制

```
问题："报销标准和流程是什么？"

分解：
  子查询1："差旅费报销标准"
  子查询2："差旅费报销流程"
       │
       ▼
  分别检索 → 合并去重 → 生成综合回答
```

> 适合"多意图"问题，避免单次检索只命中部分内容。

## 5.2 实现

```java
interface QueryDecomposer {
    @UserMessage("""
            把复杂问题拆解为独立的子问题，输出 JSON 数组：
            {{it}}
            """)
    List<String> decompose(String question);
}

public List<Document> decomposeRetrieve(String question) {
    List<String> subs = decomposer.decompose(question);
    Set<String> seen = new HashSet<>();
    List<Document> result = new ArrayList<>();
    for (String sub : subs) {
        for (Document d : vectorStore.similaritySearch(
                SearchRequest.builder().query(sub).topK(10).build())) {
            if (seen.add(d.id())) {
                result.add(d);
            }
        }
    }
    return result;
}
```

## 5.3 边界与坑

| 坑 | 说明 |
|----|------|
| 拆得过细 | 子问题过多，延迟与成本上升 |
| 子问题不独立 | 存在依赖关系时无法并行检索 |
| 漏拆 | 模型误判为简单问题，仍只检索一次 |
| 合并顺序丢失 | 综合回答时子答案顺序错乱 |
| 子问题越界 | 拆出原文没有覆盖的新问题，检索为空 |

## 5.4 生产建议

- 先用轻量判断器判定"是否需要分解"，简单问题跳过；
- 限制子问题数量（2~4）；
- 子问题并行检索以控制延迟；
- 生成综合答案时按子问题顺序组织，保证逻辑连贯；
- 用 `CompletableFuture` 并行检索各子问题，避免串行叠加延迟。

# 6. HyDE（Hypothetical Document Embeddings）

**结论先行**：HyDE（Hypothetical Document Embeddings，假设文档嵌入）先生成一段"假想答案"，再用假想答案去检索。它解决的是"问题与文档表述形式不对称"的问题。

## 6.1 机制与直觉

```
问题 → LLM 生成假设答案 → 对假设答案向量化 → 检索
```

为什么有效：假设答案与真实文档在**表述形式**上更接近（都是陈述句），比问题更容易命中。即使假设答案内容不完全正确，其"形式"和"关键词分布"也更贴近目标文档。

```
问题向量：  "差旅报销的时限是多少？"   → 疑问区
假设答案：  "差旅报销需在费用发生后 30 天内提交。" → 陈述区
真实文档：  "报销需在费用发生后 30 天内提交。"     → 陈述区
```

假设答案与真实文档同处陈述区，距离更近。

## 6.2 实现

```java
interface HypotheticalGenerator {
    @UserMessage("""
            请根据你的知识，写一段可能回答下面问题的内容（100 字左右）。
            即使不确定也请尽量写得像文档片段：
            {{it}}
            """)
    String generate(String question);
}

public List<Document> hydeRetrieve(String question) {
    String hypothetical = generator.generate(question);
    return vectorStore.similaritySearch(
            SearchRequest.builder().query(hypothetical).topK(10).build());
}
```

**优点**：解决"问题与文档表述差异大"的问题；**缺点**：多一次 LLM 调用，且假设答案可能误导。

### 6.2.1 多假设版本

生成多个假设答案并融合，可降低单个假设跑偏的风险：

```java
public List<Document> multiHydeRetrieve(String question, int n) {
    List<List<Document>> lists = new ArrayList<>();
    for (int i = 0; i < n; i++) {
        String hypo = generator.generate(question);
        lists.add(vectorStore.similaritySearch(
                SearchRequest.builder().query(hypo).topK(10).build()));
    }
    return mergeByRrf(lists, 60);
}
```

## 6.3 边界与坑

- 幻觉扩散：假设答案编造了错误实体，检索到错误文档；
- 领域偏差：模型不熟悉领域时假设答案质量差；
- 延迟翻倍：生成答案本身是 LLM 调用；
- 精确查询失效：编号、代码等事实性查询被"改写"成相似但错误的表述。

## 6.4 生产建议

- 与原始查询**一起检索融合**，而不是完全替换；
- 对事实性极强的精确查询（如编号、代码）不启用 HyDE；
- 生成时限制长度，避免拖慢向量化；
- 监控假设答案与最终命中文档的一致性。

# 7. 组合使用

**结论先行**：四类技术可流水线组合，但组合越多成本越高，应按查询复杂度动态决定启用哪些环节。

## 7.1 组合流水线

```
用户问题
   │
   ▼
① 重写（结合历史）→ 规范查询
   │
   ▼
② 分解（若复杂）→ 子查询
   │
   ▼
③ 扩展 + HyDE → 多路查询
   │
   ▼
④ 多路检索 → RRF 融合 → 重排
```

## 7.2 实现

```java
public List<Document> advancedRetrieve(String question, String history) {
    String rewritten = rewriter.rewrite(history, question);

    List<String> subQueries = decomposer.decompose(rewritten);

    List<List<String>> rankedLists = new ArrayList<>();
    for (String sub : subQueries) {
        rankedLists.add(denseSearch(sub, 20));
        rankedLists.add(denseSearch(hyde.generate(sub), 20));
        rankedLists.add(sparseSearch(sub, 20));
    }

    return rrfFuse(rankedLists, 60);
}
```

## 7.3 边界与坑

- 环节串联导致延迟叠加，可能超过预算；
- 某环节失败会拖垮整条链路，需独立降级；
- 多路结果未重排，融合顺序不稳定；
- 组合爆炸：子查询 × 扩展 × HyDE 会让检索次数成倍增长。

## 7.4 生产建议

- 用"复杂度分类器"决定启用哪些环节；
- 各环节独立超时与降级；
- 融合后接重排，保证最终顺序质量；
- 对总检索次数设上限，防止组合爆炸。

# 8. 成本与收益

**结论先行**：每增加一种改写技术，都会增加 LLM 调用与检索次数，必须用收益评测决定是否值得。

## 8.1 成本对比表

| 技术 | LLM 调用 | 检索次数 | 收益 |
|------|:--------:|:--------:|------|
| 查询重写 | 1 | 1 | 中高 |
| 多查询扩展 | 1 | N | 高 |
| 查询分解 | 1 | N | 中高 |
| HyDE | 1 | 1~N | 中高 |
| 全组合 | 2+ | 3N | 高但贵 |

## 8.2 收益评估

用标注集对比"改写前 / 改写后"的 Recall@K 与 nDCG@K。若提升不足 2~3 个百分点，可能不值得多付的延迟与费用。

```java
public void evaluateTransformation(List<EvalCase> cases) {
    double before = recallAtK(cases, q -> retrieve(q));
    double after = recallAtK(cases, q -> advancedRetrieve(q, ""));
    log.info("Recall before={} after={} gain={}", before, after, after - before);
}
```

## 8.3 按需启用策略

> **按需使用**：简单查询不必全上；先做重写，不够再加扩展/HyDE。

| 查询类型 | 建议启用 |
|----------|----------|
| 单轮简单 | 不启用 |
| 多轮对话 | 重写 |
| 召回不足 | 重写 + 扩展 |
| 多意图 | 重写 + 分解 |
| 表述差异大 | 重写 + HyDE |

# 9. 缓存与延迟

**结论先行**：改写结果可缓存，且必须计入延迟预算；否则会在高并发下成为瓶颈。

## 9.1 缓存设计

```java
// 改写结果可缓存（相同问题+历史）
String cacheKey = sha256(question + history);
String rewritten = rewriteCache.get(cacheKey, () -> rewriter.rewrite(history, question));
```

缓存键应包含**历史摘要**，否则多轮场景会命中错误缓存。可先对历史做摘要再拼 key，以控制 key 长度与稳定性。

## 9.2 延迟预算

> 查询改写增加一次 LLM 调用（~200~500ms），需评估延迟预算。

| 环节 | 典型延迟 |
|------|----------|
| 重写 | 200~500ms |
| 扩展 | 200~500ms |
| HyDE | 300~800ms |
| 检索 | 10~100ms |
| 重排 | 100~500ms |

端到端预算示例：若总预算 2s，则"重写 + 检索 + 重排"尚可，叠加 HyDE 可能超预算。

## 9.3 并发与超时

- 多路检索并行执行，避免串行叠加；
- 每个 LLM 环节设置独立超时，超时即降级为原始查询；
- 缓存命中率作为核心监控指标；
- 用 `CompletableFuture.orTimeout` 给改写环节设超时，超时即回退原查询。

# 10. 生产注意事项

**结论先行**：查询改写的风险主要是"信息丢失、噪声引入、延迟超预算、无评测"，需通过提示约束、数量限制、缓存与评测来治理。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 改写丢失关键信息 | 提示强调"保留关键实体" |
| 多轮指代未处理 | 结合对话历史改写 |
| 过度扩展引入噪声 | 限制扩展数量（2~3） |
| HyDE 误导 | 与原始查询一起检索融合 |
| 延迟超预算 | 缓存 + 按需启用 |
| 无评测 | 用标注集验证改写收益 |

## 10.2 监控与评测

| 指标 | 用途 |
|------|------|
| 改写成功率 | 发现格式解析失败 |
| 缓存命中率 | 评估成本与延迟 |
| 各环节延迟 | 定位瓶颈 |
| 改写前后 Recall | 量化收益 |
| 回退触发率 | 评估稳定性 |

## 10.3 降级策略

任一改写环节失败时，应回退到"直接用原始查询检索"，保证可用性优先。降级需记录日志与告警，便于后续分析。

```java
public String safeRewrite(String history, String question) {
    try {
        String r = rewriter.rewrite(history, question);
        return (r == null || r.isBlank()) ? question : r;
    } catch (Exception e) {
        log.warn("rewrite failed, fallback", e);
        return question;
    }
}
```

# 11. 小结

- 查询改写解决"用户表达 ≠ 文档表述"的鸿沟。
- 四类技术：重写、扩展、分解、HyDE。
- 多轮对话必须结合历史改写，解决指代。
- 组合使用效果最好，但成本也最高，按需启用。
- 改写结果可缓存，注意延迟预算。
- 一切改写都要用标注集验证收益，避免为负收益买单。
