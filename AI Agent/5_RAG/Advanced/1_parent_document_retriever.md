# 1. 核心矛盾与解法

结论先行：RAG（Retrieval-Augmented Generation，检索增强生成）中"块小则检索准、块大则信息全"是一对根本矛盾，父子分块（Parent-Child / Small-to-Big）用"小子块检索、大父块生成"同时满足两端。

## 1.1 块大小的两难

### 1.1.1 检索视角

```
块小 → 检索准（语义聚焦），但喂给模型的信息不完整
块大 → 信息完整，但检索不准（语义被稀释）
```

从向量视角看，块越小，嵌入向量越接近单一主题；块越大，多个主题被平均，向量落在"语义重心"，导致与具体问题都不够近。

### 1.1.2 生成视角

LLM（Large Language Model，大语言模型）需要足够上下文才能作答。若只喂一个 300 字的片段，模型往往缺少前置定义、条件或结论，产生"看得见却不完整"的答案。

| 块大小 | 检索精度 | 生成完整性 | 典型问题 |
|--------|:--------:|:----------:|----------|
| 200 字 | 高 | 低 | 答案缺前提 |
| 800 字 | 中 | 中 | 折中 |
| 2000 字 | 低 | 高 | 命中不准 |

### 1.1.3 权衡的量化

用信息密度与语义聚焦度可粗略量化这一矛盾：

```
语义聚焦度 ≈ 1 / 块内主题数
信息完整度 ≈ 块内主题覆盖率
```

| 指标 | 小块 | 大块 |
|------|------|------|
| 主题数 | 少 | 多 |
| 向量聚焦 | 高 | 低 |
| 答案完整率 | 低 | 高 |

父子分块让两个指标在不同粒度上各自最优，从而绕开单一块大小的零和。

## 1.2 父子分块的核心思想

```
小块用于"检索"（精准命中），大块用于"生成"（信息完整）
```

检索在子块空间完成，命中后回溯到父块，把完整上下文交给模型。这一模式又称 Small-to-Big（由小到大）检索。

## 1.3 与其他分块策略对比

| 策略 | 检索粒度 | 生成粒度 | 复杂度 |
|------|----------|----------|--------|
| 固定分块 | 块 | 同块 | 低 |
| 父子分块 | 子块 | 父块 | 中 |
| 句子窗口 | 句子 | 窗口 | 中 |
| 语义分块 | 语义段 | 语义段 | 中 |

# 2. 结构设计

结论先行：父子结构本质是一棵两层树，父块是"生成单元"，子块是"检索单元"，两者通过 parentId 关联。

## 2.1 父子层级模型

```
父块 P1（完整章节，约 1500 字）
├── 子块 C1（约 300 字）→ 向量化，用于检索
├── 子块 C2（约 300 字）→ 向量化，用于检索
└── 子块 C3（约 300 字）→ 向量化，用于检索

父块 P2
├── 子块 C4
└── 子块 C5
```

检索命中 C2 → 返回 P1 的完整内容给模型。

### 2.1.1 结构不变量

| 不变量 | 含义 |
|--------|------|
| 每个子块恰属一个父块 | 便于回溯 |
| 子块文本 ⊆ 父块文本 | 避免信息丢失 |
| parentId 唯一 | 去重依据 |

维护这三条不变量，回溯逻辑就永远不需要处理歧义。

## 2.2 分块边界策略

### 2.2.1 语义/结构分块

优先按文档结构切分：标题、段落、列表、表格。结构边界天然是语义边界，优于按字符数硬切。

### 2.2.2 重叠与切分器

子块之间可设置 10% ~ 20% 重叠，避免答案恰好跨边界被截断。父块通常不重叠，或重叠极小。

| 切分方式 | 优点 | 缺点 |
|----------|------|------|
| 定长切分 | 简单、块大小均匀 | 破坏语义 |
| 结构切分 | 语义完整 | 块大小不均 |
| 递归切分 | 兼顾两者 | 实现稍复杂 |

### 2.2.3 切分器选型

| 切分器 | 适用 | 备注 |
|--------|------|------|
| 字符切分 | 日志、代码 | 简单 |
| 递归切分 | 通用文本 | 首选 |
| Markdown 切分 | 结构化文档 | 保留标题层级 |
| Token 切分 | 严格预算 | 需分词器 |

直觉：父块用结构/递归切分，子块在父块内部再切，保证子块不跨父块边界。

## 2.3 元数据设计

| 字段 | 用途 |
|------|------|
| `parentId` | 回溯父块 |
| `source` | 引用溯源 |
| `tenantId` | 多租户隔离 |
| `chunkIndex` | 顺序与窗口扩展 |
| `docVersion` | 版本一致性 |

### 2.3.1 元数据构造示例

```java
Map<String, Object> meta = new HashMap<>();
meta.put("parentId", parentId);
meta.put("source", source);
meta.put("chunkIndex", index);
meta.put("docVersion", version);
TextSegment child = TextSegment.from(text, Metadata.from(meta));
```

# 3. 存储设计

结论先行：父子分块需要两套存储——向量库存子块，关系库或文档库存父块，二者通过 parentId 关联。

## 3.1 双存储架构

```
┌──────────────┐        ┌──────────────┐
│   向量库      │        │  文档/关系库  │
│  子块向量     │        │   父块全文    │
│  + parentId   │        │  + 元数据     │
└──────┬───────┘        └──────┬───────┘
       │  parentId 关联         │
       └───────────┬───────────┘
                   ▼
                检索服务
```

| 存储 | 内容 |
|------|------|
| 向量库 | 子块（向量 + parentId 元数据） |
| 关系库/文档库 | 父块（完整内容） |

## 3.2 Java 示例

```java
// 子块：存向量，元数据记录父块 ID
TextSegment child = TextSegment.from(childText,
        Metadata.from(Map.of(
                "parentId", parentId,
                "source", source,
                "tenantId", tenantId)));
embeddingStore.add(embeddingModel.embed(child).content(), child);

// 父块：单独存储（数据库 / 文档存储）
parentStore.save(new ParentDocument(parentId, parentText, metadata));
```

## 3.3 一致性与事务

- 父子块应在同一逻辑事务中写入，避免"有子无父"。
- 向量库与关系库通常异构，无法强事务，需补偿机制。
- 更新父块时，先写父块新版本，再原子切换子块的 parentId 指向。

### 3.3.1 补偿式写入示例

```java
public void writeParentChild(ParentDocument parent, List<TextSegment> kids) {
    parentStore.save(parent);
    try {
        kids.forEach(c -> store.add(embed(c), c));
    } catch (Exception e) {
        parentStore.delete(parent.id());
        throw e;
    }
}
```

先写父块，子块失败则回滚父块，避免产生"有子无父"的孤儿子块。

# 4. 检索流程

结论先行：检索分四步——子块召回、取 parentId、父块去重、按预算裁剪注入。

## 4.1 四步流程

```
查询
  │
  ▼
① 子块向量检索 → Top-K 子块
  │
  ▼
② 取子块的 parentId → 查父块内容
  │
  ▼
③ 父块去重（多个子块可能属于同一父块）
  │
  ▼
④ 按父块大小裁剪 → 注入提示
```

## 4.2 Java 完整示例

### 4.2.1 子块检索

```java
public List<String> retrieveParents(String query, int topK) {
    // 1. 子块检索
    List<EmbeddingMatch<TextSegment>> matches = embeddingStore.search(
            EmbeddingSearchRequest.builder()
                    .queryEmbedding(embeddingModel.embed(query).content())
                    .maxResults(topK)
                    .build()).matches();
    // ...
}
```

### 4.2.2 父块加载与去重

```java
    // 2. 取父块 ID（去重）
    Set<String> parentIds = matches.stream()
            .map(m -> m.embedded().metadata().get("parentId").toString())
            .collect(Collectors.toCollection(LinkedHashSet::new));

    // 3. 加载父块内容
    return parentIds.stream()
            .map(parentStore::getContent)
            .toList();
}
```

用 `LinkedHashSet` 保持相关度顺序，避免去重后顺序错乱。

### 4.2.3 按预算裁剪

```java
public List<String> trim(List<String> parents, int tokenBudget) {
    List<String> out = new ArrayList<>();
    int used = 0;
    for (String p : parents) {
        int t = countTokens(p);
        if (used + t > tokenBudget) {
            break;
        }
        out.add(p);
        used += t;
    }
    return out;
}
```

## 4.3 边界与坑

- 多个子块命中同一父块，若不去重会重复注入浪费 Token。
- 父块过大时直接拼接会超出上下文窗口。
- parentId 缺失或类型错误会导致回溯失败。

# 5. Spring AI 与 LangChain4j 实现

结论先行：Spring AI 提供内置 `ParentDocumentRetriever`，LangChain4j 需自建"子块检索 + 父块回溯"流程。

## 5.1 Spring AI ParentDocumentRetriever

```java
// Spring AI 内置 ParentDocumentRetriever
ParentDocumentRetriever retriever = ParentDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .documentStore(documentStore)
        .build();

List<Document> parents = retriever.retrieve(
        new Query(question)).stream()
        .map(Document::getText)
        .toList();
```

> 需要配置 `DocumentStore`（存储父块）与子块切分器。若版本差异导致 API 不同，以官方文档为准。

### 5.1.1 关键配置项

| 配置 | 作用 |
|------|------|
| vectorStore | 子块向量存储 |
| documentStore | 父块存储 |
| childSplitter | 子块切分器 |
| parentSplitter | 父块切分器 |

## 5.2 LangChain4j 自建实现

```java
public interface ParentStore {
    void save(String parentId, String content, Map<String, Object> meta);
    String getContent(String parentId);
}

public class ParentChildRetriever {
    private final EmbeddingStore<TextSegment> store;
    private final EmbeddingModel model;
    private final ParentStore parentStore;

    public List<String> retrieve(String query, int topK) {
        Embedding q = model.embed(query).content();
        List<EmbeddingMatch<TextSegment>> hits =
                store.search(EmbeddingSearchRequest.builder()
                        .queryEmbedding(q).maxResults(topK).build()).matches();
        return hits.stream()
                .map(h -> h.embedded().metadata().getString("parentId"))
                .distinct()
                .map(parentStore::getContent)
                .toList();
    }
}
```

## 5.3 伪代码说明

若框架版本不提供 `ParentDocumentRetriever`，可用下述伪代码表达等价语义：

```
function retrieve(query):
    children = vectorSearch(query, topK)
    parents  = orderedSet()
    for c in children:
        parents.add(lookupParent(c.parentId))
    return trim(parents, tokenBudget)
```

# 6. 参数设计

结论先行：父块以"信息完整"为纲，子块以"检索精准"为纲，topK 分层设置。

## 6.1 参数表

| 参数 | 建议 | 说明 |
|------|------|------|
| 父块大小 | 1000 ~ 2000 字 | 保证信息完整 |
| 子块大小 | 200 ~ 400 字 | 保证检索精准 |
| 子块重叠 | 10% ~ 20% | 避免边界截断 |
| 检索 topK | 10 ~ 20 | 子块级召回 |
| 父块去重后 | 3 ~ 5 | 喂给模型 |

## 6.2 调参直觉

```
父块/子块比 ≈ 4 ~ 6 倍
父块去重后数量 ≈ 上下文预算 / 父块 Token 数
```

### 6.2.1 预算分配示例

```
上下文窗口 8K Token
预留系统提示与答案 2K
可用上下文 6K
父块平均 800 Token → 去重后约 6 ~ 7 个父块
```

## 6.3 边界与坑

- 父块过大导致注入数量被迫压缩，反而丢信息。
- 子块过小导致语义碎片化，召回噪声增大。
- topK 太小可能全落在同一父块，去重后只剩一条。

# 7. 变体：句子窗口

结论先行：句子窗口（Sentence Window）把粒度降到句子，命中后扩展前后若干句，适合长文档问答。

## 7.1 机制

```
以"句子"为单位向量化，元数据记录所在段落
命中句子 → 扩展前后 N 句 → 返回窗口内容
```

### 7.1.1 窗口扩展示例

```java
public String expandWindow(int sentIndex, List<String> sentences, int n) {
    int from = Math.max(0, sentIndex - n);
    int to = Math.min(sentences.size(), sentIndex + n + 1);
    return String.join("", sentences.subList(from, to));
}
```

## 7.2 对比表

| 方案 | 粒度 | 上下文 | 存储开销 |
|------|------|--------|----------|
| 父子分块 | 段落/章节 | 完整章节 | 翻倍 |
| 句子窗口 | 句子 | 前后若干句 | 较小 |
| 全文注入 | 文档 | 整篇 | 无额外 |

## 7.3 生产建议

- 句子切分需处理缩写、小数点、代码等误切。
- 窗口大小 N 建议 2 ~ 3，过大则退化为段落检索。
- 元数据存段落原文，命中后无需二次查询。

# 8. 优缺点

结论先行：父子分块以存储翻倍和实现复杂度换取检索与生成的"双赢"，在多数场景收益大于代价。

## 8.1 优点

| 优点 | 说明 |
|------|------|
| 检索精准 | 小块语义聚焦 |
| 上下文完整 | 大块信息齐全 |
| 引用可定位 | 可回溯父块来源 |
| 实现相对简单 | 两套存储 + 一次回溯 |

## 8.2 缺点与代价

| 缺点 | 说明 |
|------|------|
| 存储翻倍 | 父块与子块都要存 |
| 实现复杂度增加 | 需维护两套存储 |
| 父块可能超 Token | 需裁剪或分段 |
| 一致性要求 | 父子块需同步更新 |

## 8.3 反例

- 反例一：FAQ 场景问题与答案都是一小段，父子分块无收益。
- 反例二：文档本身极短（<500 字），父子块几乎相同，纯属浪费。
- 反例三：只做检索不做生成（如搜索引擎），无需父块回溯。

# 9. 生产注意事项

结论先行：父子分块上线后最易出问题的是"存储不同步"与"父块超预算"，需在写入与注入两端设防。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 父块超 Token | 按预算裁剪或分段 |
| 存储不同步 | 父子块同事务写入 |
| 重复父块 | 按 parentId 去重 |
| 父块内容陈旧 | 与子块一起更新 |
| 无引用信息 | 元数据保留来源 |

## 9.2 监控

| 指标 | 含义 | 告警示例 |
|------|------|----------|
| 父块命中率 | 去重后父块数 / topK | 长期接近 1 说明 topK 过大 |
| 注入 Token 数 | 实际喂给模型 | 超预算告警 |
| 回溯失败率 | 查不到父块的比例 | 大于 0 告警 |
| 平均父块大小 | 监控漂移 | 突增需检查切分器 |

### 9.2.1 埋点示例

```java
record RetrievalMetrics(int childHits, int parentUnique,
                        int injectTokens, double missRate) {}

RetrievalMetrics m = new RetrievalMetrics(
        matches.size(), parents.size(), tokenCount,
        missCount / (double) matches.size());
metrics.record(m);
```

## 9.3 运维

- 建立父子块一致性校验任务，定期扫描孤儿子块。
- 文档更新采用版本号，避免新旧父子混用。
- 变更切分器后需全量重建并回归评测。

# 10. 小结

## 10.1 关键结论

- 父子分块解决"检索精准 vs 信息完整"的矛盾。
- 子块存向量用于检索，父块存内容用于生成。
- 检索后按 parentId 取父块并去重。
- Spring AI 内置 `ParentDocumentRetriever`。
- 存储翻倍是主要代价。

## 10.2 落地清单

```
1. 定义父/子切分器（父 1000~2000 字，子 200~400 字）
2. 子块写入向量库并带 parentId
3. 父块写入文档库
4. 检索 → 去重 → 裁剪 → 注入
5. 监控回溯失败率与注入 Token 数
```
