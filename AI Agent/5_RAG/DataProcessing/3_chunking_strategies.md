# 1. 为什么分块如此重要

分块（Chunking）是把长文档切成检索单元的过程，是 RAG 流水线里影响效果最大的单一环节。同样的模型与向量库，换分块策略效果可能天差地别。

## 1.1 分块与检索的耦合关系

向量检索的基本单位是块，不是文档。用户问题与块做相似度比较，块的质量直接决定召回内容的质量。块太大包含多个主题，向量被"平均"后指向模糊；块太小语义不完整，答案被拆散。

```
用户问题 ──► 相似度 ──► 命中块 ──► 拼接 ──► LLM 生成
                         ▲
                    分块质量决定这一步的成败
```

## 1.2 大小的两难

### 1.2.1 过大

```
分块太大：噪声多、检索不准、Token 浪费
```

一个块里混了多个主题，相似度被稀释，检索到的是"相关但答非所问"的内容；同时占用大量上下文窗口，挤掉其他有用块。

### 1.2.2 过小

```
分块太小：语义不完整、答案被拆散、召回碎片化
```

一个完整答案被切成三四块，只召回其中一块，LLM 拿不到完整信息，回答残缺。

## 1.3 边界与反例

- 反例一：全库统一用 500 字符固定分块，法条被从中间切断，条款编号与正文分离。
- 反例二：块大小设为 4000 字符，检索命中的块包含答案也包含大量无关内容，LLM 被噪声干扰。
- 边界：分块没有"正确值"，只有"在评测集上更优的值"。

> **分块是 RAG 效果的最大单一变量**。

# 2. 五种分块策略

## 2.1 策略总览

| 策略 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 固定长度 | 按字符/Token 切 | 简单、可预测 | 切断语义 |
| 递归 | 段落→句子→词逐级 | 尽量保持语义 | 块长不均 |
| 按结构 | 按标题/章节切 | 尊重文档结构 | 依赖格式规范 |
| 语义 | 按语义相似度断点 | 语义最完整 | 计算成本高 |
| 父子 | 小块检索、大块喂模型 | 兼顾精度与完整 | 存储翻倍 |

## 2.2 策略对比表

| 维度 | 固定 | 递归 | 结构 | 语义 | 父子 |
|------|------|------|------|------|------|
| 实现复杂度 | 低 | 低 | 中 | 高 | 高 |
| 语义完整性 | 差 | 中 | 好 | 最好 | 好 |
| 计算成本 | 低 | 低 | 低 | 高 | 中 |
| 依赖文档格式 | 否 | 否 | 是 | 否 | 否 |

## 2.3 选型决策树

```
文档有规范标题层级？ ──是──► 按结构分块
        │否
        ▼
追求最高语义质量？ ──是──► 语义 / 父子分块
        │否
        ▼
默认：递归分块（+ 重叠）
```

结论先行：**默认递归，结构规范用按结构，追求质量用语义或父子。**

## 2.4 混合策略

真实文档很少只用一种策略。常见组合是"结构优先 + 递归兜底 + 重叠"。

```
按标题切章节
   │ 章节仍超长？
   ▼
对超长章节做递归切分
   │ 再加 10%~15% 重叠
   ▼
得到最终块
```

```java
// 伪代码：结构 + 递归混合
List<TextSegment> sections = structureSplitter.split(document);
List<TextSegment> chunks = new ArrayList<>();
for (TextSegment section : sections) {
    if (section.text().length() <= maxSize) {
        chunks.add(section);
    } else {
        chunks.addAll(recursiveSplitter.split(section));
    }
}
```

边界：混合策略会引入两套参数，调试复杂度上升，需用评测集统一验证。

# 3. 固定长度分块

## 3.1 原理

按固定字符数或 Token 数切分，不考虑语义边界。实现最简单，行为最可预测，是所有方案的兜底。

## 3.2 参数详解

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
        .withChunkSize(800)
        .withMinChunkSizeChars(350)
        .withMinChunkLengthToEmbed(5)
        .withMaxNumChunks(10000)
        .withKeepSeparator(true)
        .build();
```

| 参数 | 含义 |
|------|------|
| `chunkSize` | 每块目标 Token 数 |
| `minChunkSizeChars` | 小于此字符数不单独成块 |
| `keepSeparator` | 是否保留分隔符 |

## 3.3 适用场景

**适用**：格式不固定、结构混乱的文本，作为兜底方案。例如日志、OCR 结果、拼接文本。

## 3.4 边界与坑

- 坑一：按 Token 切比按字符切更贴近模型限制，但 Token 与字符不成固定比例，中文尤甚。
- 坑二：`keepSeparator=false` 会让块首尾缺失标点，拼接时读不通。
- 边界：固定长度对代码、表格极不友好，这类内容应单独处理。

# 4. 递归分块（推荐默认）

## 4.1 原理：分隔符优先级

递归分块先尝试用"最大语义单位"的分隔符切分，若块仍超长，再降级用更小的分隔符，直到满足大小限制。

递归顺序：`\n\n`（段落）→ `\n`（行）→ `.` `。`（句子）→ 空格 → 字符。

```
第一级：段落   [========|========|========]
第二级：行     [====|====|====|====]
第三级：句子   [==|==|==|==|==|==]
```

## 4.2 代码示例

```java
DocumentSplitter splitter = DocumentSplitters.recursive(
        800,    // 每段最大字符数
        100);   // 重叠字符数

List<TextSegment> segments = splitter.split(document);
```

**优点**：优先在语义边界切分，块大小尽量均匀。

## 4.3 分隔符定制

不同语言的句子边界不同。中文用 `。！？`，英文用 `.?!`，代码用 `\n` 与 `{}`。可自定义分隔符列表以适配语料。

```java
// 伪代码：自定义分隔符优先级
List<String> separators = List.of("\n\n", "\n", "。", "！", "？", "；", " ");
```

## 4.4 边界与坑

- 坑一：块长不均，遇到超长无标点文本（如 Base64）会退化到按字符切。
- 坑二：递归只看分隔符，不理解语义，跨段落的完整论述仍可能被拆开。
- 边界：重叠参数要与递归配合，否则边界处信息仍会丢失。

# 5. 按结构分块

## 5.1 结构来源

结构可来自 Markdown 标题、HTML 标签、Word 大纲、PDF 书签。只要文档有稳定结构，按结构切分能最大程度保持语义完整。

## 5.2 Markdown 标题分块

```java
// Markdown 按标题层级
MarkdownDocumentReader reader = new MarkdownDocumentReader(resource,
        MarkdownDocumentReaderConfig.builder()
                .withHorizontalRuleCreateDocument(true)
                .withIncludeCodeBlock(false)
                .build());
```

其他按结构的切分器：

```java
// 按段落 / 行 / 句子
DocumentByParagraphSplitter paragraphSplitter = new DocumentByParagraphSplitter(1000, 100);
DocumentByLineSplitter lineSplitter = new DocumentByLineSplitter(1000, 100);
DocumentBySentenceSplitter sentenceSplitter = new DocumentBySentenceSplitter(500, 50);
```

**适用**：结构规范的文档（手册、政策、技术文档）。

## 5.3 标题上下文注入

> 按结构切分时，**务必把标题写入元数据或拼接进块内容**，否则块会失去上下文。

### 5.3.1 拼进块内容

```java
// 把章节标题拼进块内容，提升检索命中
String enriched = "【" + sectionTitle + "】\n" + chunkText;
```

### 5.3.2 写入元数据

```java
// 伪代码：标题入元数据，检索时可回填
Map<String, Object> meta = Map.of(
        "section", sectionTitle,
        "level", headingLevel);
```

## 5.4 边界与坑

- 坑一：某个章节过长（超过模型上限）时仍需二次切分，结构分块不能单独使用。
- 坑二：标题层级不规范的文档（用加粗代替标题）会切不出结构。
- 边界：结构分块应与递归分块组合——先按结构切，再对超长章节递归切。

## 5.5 标题路径的完整保留

只拼一级标题会丢失层级关系。更好的做法是拼接"标题路径"，形如"手册 > 第三章 > 3.2 报销流程"。

```java
// 伪代码：维护标题栈，生成完整路径
Deque<String> headingStack = new ArrayDeque<>();

String path = String.join(" > ", headingStack);
String enriched = path + "\n" + chunkText;
```

| 保留方式 | 检索效果 | 说明 |
|----------|----------|------|
| 仅当前标题 | 中 | 缺层级 |
| 标题路径 | 好 | 上下文完整 |
| 仅元数据 | 低 | 不进向量，命中率低 |

边界：标题路径过长会稀释正文语义，建议最多保留 3 级。

# 6. 语义分块

## 6.1 原理

```
把文本切成句子 → 计算相邻句子相似度 → 相似度骤降处作为断点
```

语义分块用向量相似度判断"话题是否切换"，在话题边界处切开，语义最完整。

## 6.2 代码示例

```java
public List<String> semanticChunk(List<String> sentences, double threshold) {
    List<String> chunks = new ArrayList<>();
    List<String> current = new ArrayList<>();
    double[] embeddings = embed(sentences);

    for (int i = 0; i < sentences.size(); i++) {
        current.add(sentences.get(i));
        if (i + 1 < sentences.size()
                && cosine(embeddings[i], embeddings[i + 1]) < threshold) {
            chunks.add(String.join("", current));
            current.clear();
        }
    }
    if (!current.isEmpty()) chunks.add(String.join("", current));
    return chunks;
}
```

**优点**：语义完整；**缺点**：需额外 Embedding 计算。

## 6.3 阈值与实现变体

| 变体 | 断点判据 |
|------|----------|
| 绝对阈值 | 相邻相似度 < 固定值 |
| 百分位 | 相似度最低的 N% 处 |
| 标准差 | 低于均值 - k×标准差 |

百分位法比绝对阈值更稳健，因为相似度分布随语料变化。

## 6.4 边界与坑

- 坑一：每个句子都要 Embedding，长文档成本高，需先用递归粗切再语义细分。
- 坑二：相似度阈值难调，不同领域差异大，必须用评测集确定。
- 边界：短文档（少于 10 句）语义分块收益有限，不值得引入成本。

# 7. 父子分块

## 7.1 原理

```
文档
├── 父块（如整节，1000~2000 字）  ← 喂给模型
│   ├── 子块（如段落，200~400 字） ← 用于检索
│   ├── 子块
│   └── 子块
└── 父块
    ├── 子块
    └── 子块
```

用小块做检索（精度高），命中后回取父块内容喂给模型（上下文完整），兼顾两者优点。

## 7.2 代码示例

```java
// 子块存向量，元数据里记录父块 ID
TextSegment child = TextSegment.from(childText,
        Metadata.from(Map.of("parentId", parentId)));
store.add(embeddingModel.embed(child).content(), child);

// 检索命中子块 → 取父块内容注入
String parentText = parentStore.get(match.embedded().metadata().get("parentId").toString());
```

**优点**：检索精度高（小块），上下文完整（大块）。

## 7.3 与句子窗口的关系

句子窗口（Sentence Window）是父子分块的变体：以句子为子块检索，命中后返回前后 N 句作为父内容。区别在于父内容是否预先固化。

## 7.4 边界与坑

- 坑一：父块若过大，超出上下文窗口，需限制父块上限。
- 坑二：父块内容被多子块命中时会重复注入，需去重。
- 边界：存储翻倍，需评估向量库容量。

> 详见 `Advanced/1_parent_document_retriever.md`。

# 8. 重叠（Overlap）

## 8.1 作用与代价

```
块1: [==========]
块2:       [==========]     ← 重叠部分
块3:             [==========]
```

- 目的：防止语义在边界被切断。
- 建议：块大小的 **10%~20%**。
- 代价：存储与向量化成本增加。

## 8.2 重叠比例

| 内容类型 | 建议重叠 |
|----------|----------|
| 叙述性文本 | 10%~15% |
| 技术文档 | 15%~20% |
| 结构化条目 | 0（按条目切） |

## 8.3 边界与坑

- 坑一：重叠过大导致相邻块高度相似，检索返回重复内容。
- 坑二：结构化内容（法条、FAQ）加重叠反而破坏条目完整性。
- 边界：重叠是"缓解"而非"解决"边界切断，语义分块能从根本减少该问题。

# 9. 参数选择指南

## 9.1 场景对照表

| 场景 | 块大小 | 重叠 |
|------|--------|------|
| 通用问答 | 500~800 | 10% |
| 技术文档 | 800~1200 | 15% |
| 法条/条款 | 按条款 | 0~1 句 |
| 对话记录 | 按轮次 | 1 轮 |
| 代码 | 按函数/类 | 0 |

## 9.2 参数搜索方法

> **没有万能值**。做法：准备 3~5 组参数，用评测集对比。

```java
// 伪代码：网格搜索
int[] sizes = {400, 600, 800, 1200};
int[] overlaps = {0, 50, 100};
for (int size : sizes)
    for (int overlap : overlaps)
        score(size, overlap, evalSet);
```

## 9.3 常见误区

- 误区一：直接照搬别人的参数，忽略语料差异。
- 误区二：只调块大小不调重叠，效果提升有限。
- 误区三：用检索指标调参却忘了生成质量才是最终目标。

## 9.4 参数配置化与版本化

分块参数应作为配置管理，并带上版本号，便于回溯"某次效果变化是哪个参数导致的"。

```java
public record ChunkConfig(
        String version,
        int chunkSize,
        int overlap,
        String strategy) {

    public static ChunkConfig defaultConfig() {
        return new ChunkConfig("v1", 800, 100, "recursive");
    }
}
```

| 配置项 | 作用 | 变更影响 |
|--------|------|----------|
| `strategy` | 分块策略 | 需全量重建 |
| `chunkSize` | 块大小 | 需全量重建 |
| `overlap` | 重叠 | 需全量重建 |

边界：任何分块参数变更都会使已有索引失效，必须走"新索引 + 评测 + 别名切换"流程。

# 10. 分块效果评测

## 10.1 指标

```
指标：召回率（正确块是否被检索到）
```

| 指标 | 含义 |
|------|------|
| 块级召回率 | 答案所在块是否被召回 |
| 上下文精确率 | 召回块中相关块占比 |
| 端到端得分 | 最终回答是否正确 |

## 10.2 评测流程

```
方法：
1. 准备问答对，标注"答案所在块"
2. 用不同分块策略索引
3. 对比"答案块的召回率"
```

## 10.3 边界与坑

- 坑一：标注"答案所在块"依赖分块结果，换分块后标注失效，需按字符区间标注。
- 坑二：只看召回不看精确，会偏爱超大块。
- 边界：评测集要覆盖典型问题类型，样本太少结论不可靠。

> 详见 `Evaluation/1_rag_evaluation.md`。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 块太小丢语义 | 提高块大小或父子分块 |
| 块太大引入噪声 | 降低块大小 + 重排 |
| 表格被切断 | 表格整体成块 |
| 标题与内容分离 | 标题拼进块内容 |
| 参数拍脑袋 | 评测驱动 |
| 全量重切 | 用新索引 + 别名切换 |

## 11.2 索引重建与别名切换

换分块策略意味着全量重建索引。做法：新建索引写入新块，评测通过后用别名（alias）原子切换到新索引，避免停机与检索中断。

```java
// 伪代码：别名切换
client.indices().create(c -> c.index("knowledge_v2"));
reindex("knowledge_v1", "knowledge_v2");
client.indices().updateAliases(a -> a
        .remove(r -> r.index("knowledge_v1").alias("knowledge"))
        .add(x -> x.index("knowledge_v2").alias("knowledge")));
```

## 11.3 可观测性

记录分块指标：块总数、平均块长、块长分布、重叠比例、空块数。块长分布异常（大量极短块）往往意味着分块参数或解析质量有问题。

## 11.4 分块流水线的幂等

分块是确定性操作，同一文档同一配置应产出完全相同的块。因此块 ID 可用"文档 ID + 块序号 + 配置版本"稳定生成，便于增量更新与去重。

```java
// 伪代码：稳定块 ID
String chunkId = docId + "#" + index + "@" + config.version();
```

| 场景 | 期望行为 |
|------|----------|
| 文档未变 | 块 ID 不变，跳过重写 |
| 文档变更 | 仅变更块重写 |
| 配置变更 | 全量重写，新索引 |

边界：块 ID 稳定依赖分块确定性，若分块器内部有随机性（如语义分块的采样），需固定随机种子。

# 12. 小结

- 分块是 RAG 效果的最大变量。
- 五种策略：固定、递归、结构、语义、父子。
- 默认用递归，结构规范用按结构，追求质量用语义/父子。
- 重叠 10%~20%，防止语义切断。
- 参数必须评测驱动，不要拍脑袋。
- 换策略要全量重建并用别名原子切换。
