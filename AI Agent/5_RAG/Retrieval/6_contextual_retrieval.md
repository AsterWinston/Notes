# 1. 问题：分块丢失上下文

**结论先行**：传统分块（Chunking，分块）把文档切成孤立片段，片段脱离原始语境后，既丢失了标题、章节等结构信息，也丢失了指代与省略的语义线索，导致检索命中率下降。上下文检索（Contextual Retrieval，上下文检索）就是为每个片段补回这些语境。

## 1.1 分块为何丢失上下文

### 1.1.1 结构信息丢失

```
原文：
  "第三章 差旅报销
   1. 员工出差前需提交申请。
   2. 报销需在费用发生后 30 天内提交。"   ← 被切分的片段

切分后片段：
  "报销需在费用发生后 30 天内提交。"      ← 不知道是"差旅报销"
```

标题、章节号、文档名都在切分时被剥离，片段自身无法表达"它属于哪个主题"。

### 1.1.2 指代与省略

文档中大量出现"该流程""上述费用""本公司"等指代与省略，脱离上下文后无法解析其指向。检索"差旅报销时限"时，片段可能因缺少"差旅"关键词而命中率低。

### 1.1.3 语义漂移

即使不涉及指代，孤立片段也容易产生语义漂移：一个讲"审批权限"的片段若没有"差旅"前缀，其向量可能更接近"通用审批"而非"差旅审批"，从而在排序中掉队。

## 1.2 上下文丢失的表现

| 表现 | 后果 |
|------|------|
| 关键词缺失 | 稀疏检索（BM25）命中率低 |
| 主题不明 | 稠密向量语义漂移 |
| 指代悬空 | 生成答案引用错误 |
| 边界模糊 | 相邻片段内容混淆 |

## 1.3 影响范围

上下文丢失对**结构化文档**（手册、合同、法规）影响最大，对短文本、独立问答对影响较小。因此方案选择要与文档形态匹配。

| 文档类型 | 受影响程度 |
|----------|:----------:|
| 长手册/法规 | 高 |
| 合同/制度 | 高 |
| 技术文档 | 中高 |
| FAQ 问答对 | 低 |
| 短评论 | 低 |

# 2. 解决方案总览

**结论先行**：补回上下文有五类方案，成本从"零成本"到"每块一次 LLM 调用"不等，应按文档价值与预算分层使用。

## 2.1 五类方案

| 方案 | 思路 |
|------|------|
| 上下文拼接 | 把标题/章节拼进片段 |
| 上下文检索（Contextual Retrieval） | 用 LLM 为每块生成上下文说明 |
| 父子分块 | 小块检索、大块喂模型 |
| 句子窗口 | 命中句子后扩展前后文 |
| 元数据增强 | 把章节信息写入元数据 |

## 2.2 成本与效果梯度

```
成本 低 ────────────────────────────────────► 高
     标题拼接   元数据增强   句子窗口   父子分块   上下文检索
效果 中         中           中高       高         高
```

## 2.3 选型路径

> 推荐路径：**标题拼接（默认）→ 父子分块 → 上下文检索（高价值场景）**。

先上零成本方案，用评测确认收益后再逐级加码。切忌一上来就对全量文档做 LLM 上下文增强，成本会失控。

# 3. 方案一：标题拼接

**结论先行**：标题拼接（Title Prefixing，标题前缀）是成本最低、见效最快的方案，应作为默认做法。

## 3.1 机制

在切分时记录每个片段所属的层级标题，拼接进片段文本后再向量化与建索引。这样片段同时携带了"它是什么主题"的信息。

```
文档结构解析
   │
   ▼
遍历标题层级，维护"当前标题路径"
   │
   ▼
每个片段 = 【标题路径】 + 正文
```

## 3.2 实现

```java
// 切分时把层级标题拼进块内容
String enriched = """
        【员工手册 > 第三章 差旅报销】
        %s
        """.formatted(chunkText);
```

> 零成本、立竿见影，**应作为默认做法**。

结合分块器维护标题路径的实现思路：

```java
String chapterPath = "员工手册 > 第三章 差旅报销";
for (String chunkText : splitByParagraph(section)) {
    String enriched = "【" + chapterPath + "】\n" + chunkText;
    store.add(TextSegment.from(enriched, metadataOf(chapterPath)));
}
```

## 3.3 边界与坑

| 坑 | 说明 |
|----|------|
| 标题过长 | 拼接后片段变长，稀释正文权重 |
| 层级缺失 | 解析器未识别标题结构，拼了空标题 |
| 标题与内容不符 | 文档标题层级混乱，误拼 |
| 元数据未同步 | 只改了文本，未写入元数据 |

## 3.4 生产建议

- 拼接时保留完整层级路径（文档 > 章 > 节）；
- 控制标题长度，必要时截断；
- 同时写入元数据，便于过滤与展示；
- 对无标题文档做兜底（用文件名或首句代替）。

# 4. 方案二：上下文检索（Contextual Retrieval）

**结论先行**：上下文检索用 LLM 为每个片段生成一句"位置与主题说明"，再拼接到片段前。它效果显著，但成本最高，适合高价值文档。

## 4.1 原理

Anthropic 提出的方法：**用 LLM 为每个块生成一段上下文说明，再拼接**。

```
原始块："报销需在费用发生后 30 天内提交。"

LLM 生成上下文："本段来自《员工手册》第三章差旅报销，说明差旅费报销的时限要求。"

增强后块："本段来自《员工手册》第三章差旅报销，说明差旅费报销的时限要求。
          报销需在费用发生后 30 天内提交。"
```

关键约束：让模型**只说明位置与主题，不回答问题、不补充事实**，避免引入幻觉。

## 4.2 实现

### 4.2.1 LangChain4j AI Service

```java
interface Contextualizer {
    @UserMessage("""
            下面是整篇文档和其中一个片段。
            请用一句话说明这个片段在文档中的位置和主题，不要回答问题。
            文档：{{document}}
            片段：{{chunk}}
            上下文说明：
            """)
    String contextualize(@V("document") String doc, @V("chunk") String chunk);
}
```

### 4.2.2 增强片段构造

```java
public TextSegment contextualize(String fullDoc, TextSegment chunk) {
    String context = contextualizer.contextualize(fullDoc, chunk.text());
    String enriched = context + "\n" + chunk.text();

    return TextSegment.from(enriched, chunk.metadata());
}
```

### 4.2.3 批量增强

逐块调用成本高，批量接口一次处理多个块：

```java
public List<TextSegment> contextualizeAll(String fullDoc, List<TextSegment> chunks) {
    List<TextSegment> result = new ArrayList<>();
    for (TextSegment chunk : chunks) {
        result.add(contextualize(fullDoc, chunk));
    }
    return result;
}
```

## 4.3 成本优化

对每个块都调用 LLM 成本高。优化：

| 手段 | 说明 |
|------|------|
| 提示缓存 | 同一文档的块共享文档内容缓存 |
| 小模型 | 用便宜模型生成上下文 |
| 批处理 | 一次请求处理多个块 |
| 仅关键文档 | 只对高价值文档做 |

```java
// 一次处理多个块，减少调用次数
List<String> contexts = contextualizer.contextualizeBatch(fullDoc, chunks);
```

提示缓存（Prompt Caching，提示缓存）是关键：同一文档的多个块共用"整篇文档"前缀，可大幅降低重复计费。

## 4.4 效果与评测

Anthropic 报告：Contextual Retrieval 可将检索失败率降低约 **35%**，配合 BM25 混合检索降低约 **49%**（具体因数据而异，需自行评测）。

评测做法：构造"已知答案"的查询集，对比增强前后正确文档是否进入 Top-K，统计失败率变化。

```java
public void evaluateContextualization(List<EvalCase> cases) {
    double before = failureRate(cases, /* usePlainChunks= */ true);
    double after = failureRate(cases, /* usePlainChunks= */ false);
    log.info("failure before={} after={} reduction={}%",
            before, after, (before - after) / before * 100);
}
```

## 4.5 边界与坑

| 坑 | 说明 |
|----|------|
| 上下文与原块矛盾 | 模型补充了错误信息 |
| 上下文过长 | 稀释原块语义，检索反而变差 |
| 成本失控 | 每块一次调用，文档多时费用惊人 |
| 缓存失效 | 提示前缀不一致导致缓存不命中 |
| 元数据未更新 | 增强文本与元数据不一致 |

## 4.6 生产建议

- 提示词明确"只描述位置与主题"；
- 控制上下文说明在 1~2 句；
- 优先对高价值、高频查询文档启用；
- 全流程留痕，便于回溯与评测；
- 增强前先做一次小规模 A/B，确认收益再全量。

# 5. 方案三：父子分块

**结论先行**：父子分块（Parent-Child Chunking，父子分块）用小块检索、大块喂模型，兼顾检索精度与上下文完整性。

## 5.1 机制

```
父块（完整章节，1000~2000 字）
├── 子块（200~400 字）→ 用于检索
├── 子块
└── 子块
```

检索命中子块后，返回**父块内容**给模型，保证上下文完整。

> 详见 `Advanced/1_parent_document_retriever.md`。

## 5.2 实现

```java
public List<TextSegment> parentChildRetrieve(String query, ParentStore store) {
    List<TextSegment> hits = vectorStore.search(
            SearchRequest.builder().query(query).topK(20).build());

    Set<String> parentIds = hits.stream()
            .map(s -> s.metadata().getString("parentId"))
            .collect(Collectors.toSet());

    return parentIds.stream()
            .map(store::get)
            .flatMap(Optional::stream)
            .toList();
}
```

入库阶段需先建立父子映射：

```java
String parentId = UUID.randomUUID().toString();
parentStore.put(parentId, parentSegment);
for (TextSegment child : splitIntoChildren(parentSegment)) {
    Metadata meta = child.metadata().copy().put("parentId", parentId);
    vectorStore.add(TextSegment.from(child.text(), meta));
}
```

## 5.3 边界与坑

| 坑 | 说明 |
|----|------|
| 父块过大 | 超出模型上下文窗口，浪费 token |
| 父块重复 | 多个子块命中同一父块，需去重 |
| 父子映射丢失 | 存储时未记录 parentId |
| 更新不一致 | 父块更新后子块未重建 |
| 父子粒度失衡 | 子块太小导致语义不完整 |

## 5.4 生产建议

- 父块大小控制在模型窗口的合理比例内；
- 按父块 ID 去重后再喂模型；
- 建立父子映射的同步更新机制；
- 对返回父块再做一次截断，避免超长。

# 6. 方案四：句子窗口

**结论先行**：句子窗口（Sentence Window，句子窗口）以句子为检索单位，命中后向两侧扩展窗口，用最小存储成本换取局部上下文。

## 6.1 机制

```
句子索引:  [s1] [s2] [s3] [s4] [s5]
命中 s3，窗口=1
返回:            [s2] [s3] [s4]
```

## 6.2 实现

```java
public String sentenceWindow(Document doc, int hitSentenceIndex, int window) {
    List<String> sentences = splitSentences(doc.getText());
    int start = Math.max(0, hitSentenceIndex - window);
    int end = Math.min(sentences.size(), hitSentenceIndex + window + 1);
    return String.join("", sentences.subList(start, end));
}
```

实现要点：

- 存储时以"句子"为单位向量化，元数据记录其所在段落/文档；
- 检索命中后按元数据取回窗口内容。

## 6.3 边界与坑

- 句子切分对中文不友好（标点、换行），需专门的分句器；
- 窗口过大等于退化为段落检索；
- 跨段落扩展会引入无关内容；
- 元数据缺失导致无法定位窗口。

## 6.4 生产建议

- 用成熟的中文分句库，避免简单按句号切分；
- 窗口大小按查询类型调优（1~2 句起步）；
- 窗口内容与原句一并保留，便于溯源；
- 把窗口扩展逻辑放在检索后处理阶段，不污染索引。

# 7. 方案五：元数据增强

**结论先行**：元数据增强（Metadata Enrichment，元数据增强）不改变片段文本，而是把章节、来源、时间等信息写入元数据，用于过滤与重排加权。

## 7.1 机制

| 元数据字段 | 用途 |
|------------|------|
| 文档名 | 展示来源 |
| 章节路径 | 过滤与加权 |
| 页码 | 定位原文 |
| 更新时间 | 时效性排序 |
| 权限标签 | 访问控制 |

## 7.2 实现

```java
Metadata meta = new Metadata();
meta.put("source", "员工手册.pdf");
meta.put("chapter", "第三章 差旅报销");
meta.put("page", 12);
meta.put("updatedAt", "2025-01-01");

TextSegment segment = TextSegment.from(chunkText, meta);
```

检索时可用元数据做前置过滤：

```java
SearchRequest req = SearchRequest.builder()
        .query(query)
        .topK(20)
        .filterExpression("chapter = '第三章 差旅报销'")
        .build();
```

## 7.3 边界与坑

- 元数据不参与向量语义，无法直接提升语义匹配；
- 字段命名不统一，过滤逻辑易错；
- 元数据过多会增加存储与传输开销；
- 过滤过严会直接排除正确文档。

## 7.4 生产建议

- 统一元数据 schema；
- 把元数据用于**过滤前置**（先按权限/时间过滤再检索）；
- 与标题拼接配合，文本与元数据双保险；
- 元数据字段做索引，保证过滤性能。

# 8. 方案对比

**结论先行**：五类方案在成本、效果、复杂度上各有取舍，没有银弹，应按文档价值分层组合。

## 8.1 对比表

| 方案 | 成本 | 效果 | 复杂度 |
|------|:----:|:----:|:------:|
| 标题拼接 | 极低 | 中 | 低 |
| 元数据增强 | 低 | 中 | 低 |
| 上下文检索 | 高 | 高 | 中 |
| 父子分块 | 中 | 高 | 中 |
| 句子窗口 | 中 | 中高 | 中 |

## 8.2 组合建议

> 推荐路径：**标题拼接（默认）→ 父子分块 → 上下文检索（高价值场景）**。

| 场景 | 建议组合 |
|------|----------|
| 一般文档 | 标题拼接 |
| 长手册/合同 | 标题拼接 + 父子分块 |
| 高价值知识库 | 标题拼接 + 父子分块 + 上下文检索 |
| 强时效/权限 | 元数据增强 + 过滤 |

## 8.3 决策流程

```
文档是否结构化？ ── 否 ──► 元数据增强即可
  │ 是
  ▼
是否长文档？ ── 是 ──► 标题拼接 + 父子分块
  │ 否
  ▼
是否高价值/高频查询？ ── 是 ──► 再叠加上下文检索
  │ 否
  ▼
标题拼接即可
```

# 9. 组合使用

**结论先行**：多种方案可以叠加，形成"分块时增强文本、存储时保留父子、检索时返回大块"的完整链路。

## 9.1 完整流水线

```
① 分块时拼接标题（零成本）
② 用 LLM 生成上下文说明（高价值文档）
③ 父子分块存储
④ 检索命中子块 → 返回父块 + 上下文说明
```

## 9.2 实现要点

```java
// 入库阶段
TextSegment chunk = TextSegment.from(chunkText, meta);
TextSegment enriched = contextualize(fullDoc, chunk);
TextSegment withTitle = TextSegment.from(
        "【" + chapterPath + "】\n" + enriched.text(), enriched.metadata());

// 检索阶段：命中子块 → 取父块
List<TextSegment> parents = parentChildRetrieve(query, store);
```

## 9.3 边界与坑

- 增强叠加导致片段过长，需控制总长度；
- 多方案重复注入相同信息，浪费 token；
- 检索阶段返回父块后可能超出上下文窗口；
- 任一增强环节失败会导致数据不一致。

## 9.4 生产建议

- 对增强后的片段长度设硬上限，超出则截断；
- 记录每段启用了哪些增强，便于回溯；
- 增强流程做成幂等，支持重跑。

# 10. 生产注意事项

**结论先行**：上下文增强的风险集中在"增强内容失真、成本失控、元数据不同步、无评测"四方面，需通过提示约束、成本控制与评测治理。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 增强内容与原块矛盾 | 提示要求"只说明位置与主题" |
| 成本失控 | 小模型 + 缓存 + 仅高价值文档 |
| 增强后块过长 | 控制上下文说明长度（1~2 句） |
| 元数据未同步 | 增强信息写入元数据 |
| 无评测 | 对比增强前后的召回率 |

## 10.2 成本控制

| 手段 | 说明 |
|------|------|
| 提示缓存 | 同文档块共享前缀 |
| 小模型 | 生成上下文用便宜模型 |
| 批处理 | 一次处理多块 |
| 增量增强 | 仅对新文档做 |
| 长度限制 | 限制上下文说明字数 |

## 10.3 评测方法

- 构造带标注的查询集，统计增强前后 Recall@K；
- 抽样人工评估增强说明是否准确；
- 监控平均片段长度与 token 消耗变化。

```java
public void compareEnhancement(List<EvalCase> cases) {
    double plain = recallAtK(cases, plainRetrieve);
    double enriched = recallAtK(cases, contextualRetrieve);
    double avgLen = averageChunkLength();
    log.info("Recall plain={} enriched={} avgLen={}",
            plain, enriched, avgLen);
}
```

# 11. 小结

- 分块会丢失上下文，导致检索命中率下降。
- 标题拼接是零成本默认做法。
- 上下文检索（LLM 生成块说明）效果显著但有成本。
- 父子分块与句子窗口兼顾精度与完整性。
- 推荐组合：标题拼接 + 父子分块 + （高价值场景）上下文检索。
- 一切增强都要用评测验证，并监控成本与片段长度。
