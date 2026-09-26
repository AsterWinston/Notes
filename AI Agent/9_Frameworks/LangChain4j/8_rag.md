# 1. RAG 全链路

结论先行：RAG（Retrieval-Augmented Generation，检索增强生成）分两个阶段——离线把文档切成片段并向量化入库，在线把用户问题向量化后检索相关片段，注入提示再交给模型生成。LangChain4j 为每个环节都提供了可替换的组件。

## 1.1 离线索引阶段

### 1.1.1 加载与切分

加载器把 PDF、Word、HTML、数据库等统一转成 `Document`；切分器（Splitter）把长文档拆成 `TextSegment`。切分粒度直接决定检索质量：太粗会混入噪声，太细会丢上下文。

### 1.1.2 向量化与入库

`EmbeddingModel` 把每个 `TextSegment` 转成向量，`EmbeddingStore` 负责持久化向量与原文及元数据。离线阶段通常只跑一次，或按增量更新。

## 1.2 在线查询阶段

### 1.2.1 检索

用户问题经同一个 `EmbeddingModel` 向量化，再在 `EmbeddingStore` 中做相似度搜索，返回若干 `Content`（内容 + 元数据 + 分数）。

### 1.2.2 增强与生成

检索结果经 `ContentAggregator` 合并、去重、重排，再由 `ContentInjector` 注入到提示模板，最后交给 `ChatModel` 生成答案。

## 1.3 全链路图

```
离线索引：
  Document ─DocumentSplitter─▶ TextSegment ─EmbeddingModel─▶ Embedding
                                                                 │
                                                                 ▼
                                                          EmbeddingStore

在线查询：
  用户问题 ─EmbeddingModel─▶ 查询向量 ─ContentRetriever─▶ List<Content>
                                                              │
                                                              ▼
                                                     RetrievalAugmentor
                                                              │
                                                              ▼
                                                      注入 Prompt → ChatModel
```

一句话概括：**离线建索引，在线检索 + 注入**。任何 RAG 优化都是对这两条链路中某个环节的改进。

# 2. 最小可用 RAG

结论先行：先用最少依赖跑通闭环，再逐步替换组件。LangChain4j 提供了从"一行式 Easy RAG"到"完全手动"的完整光谱。

## 2.1 依赖

### 2.1.1 核心依赖

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j</artifactId>
</dependency>
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
</dependency>
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-easy-rag</artifactId>   <!-- 可选：默认 RAG 组件 -->
</dependency>
```

### 2.1.2 Easy RAG 的作用

`langchain4j-easy-rag` 打包了一套默认实现：默认切分器、默认向量模型、内存向量库与默认检索器。它的目标是"零配置可用"，但不适合生产。

## 2.2 一行式 RAG（Easy RAG）

```java
// 加载 → 切分 → 向量化 → 存储，全部自动
EmbeddingStoreContentRetriever retriever =
        EmbeddingStoreContentRetriever.from(
                embeddingModel,
                embeddingStore);

Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .contentRetriever(retriever)
        .build();
```

边界与坑：Easy RAG 使用内存向量库，重启即丢，且无法做租户过滤。生产必须替换为持久化向量库并加元数据过滤。

## 2.3 完整手动流程

### 2.3.1 加载文档

```java
// 1. 加载文档
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/handbook.pdf"),
        new ApachePdfBoxDocumentParser());
```

### 2.3.2 切分文档

```java
// 2. 切分（递归切分：片段 800 字符，重叠 100 字符）
DocumentSplitter splitter = DocumentSplitters.recursive(800, 100);
List<TextSegment> segments = splitter.split(document);
```

### 2.3.3 向量化、入库与检索

```java
// 3. 向量化并入库
List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
embeddingStore.addAll(embeddings, segments);

// 4. 构建检索器
ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
        .embeddingStore(embeddingStore)
        .embeddingModel(embeddingModel)
        .maxResults(5)
        .minScore(0.7)
        .build();

// 5. 接入 AI Service
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .contentRetriever(retriever)
        .build();
```

## 2.4 EmbeddingStoreIngestor（推荐）

`EmbeddingStoreIngestor` 把"切分 → 向量化 → 入库"封装成一个组件，是生产中最常用的入口：

```java
EmbeddingStoreIngestor ingestor = EmbeddingStoreIngestor.builder()
        .documentSplitter(DocumentSplitters.recursive(800, 100))
        .embeddingModel(embeddingModel)
        .embeddingStore(embeddingStore)
        .build();

ingestor.ingest(document);   // 加载 → 切分 → 向量化 → 存储
```

边界与坑：`ingest` 默认不带元数据，多租户场景需在 `TextSegment` 上追加 `Metadata` 后再入库。

# 3. ContentRetriever

结论先行：`ContentRetriever` 是"问题 → 相关内容列表"的抽象。最常见实现是向量检索，此外还有联网、SQL、云端搜索等。

## 3.1 内置检索器

| 检索器 | 说明 |
|--------|------|
| `EmbeddingStoreContentRetriever` | 向量检索（最常用） |
| `WebSearchContentRetriever` | 联网搜索 |
| `SqlDatabaseContentRetriever` | 数据库查询（实验性） |
| `AzureAiSearchContentRetriever` | Azure AI Search |

## 3.2 EmbeddingStoreContentRetriever 参数

### 3.2.1 maxResults 与 minScore

```java
ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
        .embeddingStore(store)
        .embeddingModel(embeddingModel)
        .maxResults(20)          // 召回多
        .minScore(0.6)           // 相似度下限
        .dynamicFilter(query -> metadataFilter(query))   // 动态过滤
        .build();
```

- `maxResults`：召回条数，越大越全但注入 token 越多。
- `minScore`：相似度阈值，过滤掉不相关内容，但可能因模型不同而含义不同。

### 3.2.2 dynamicFilter

`dynamicFilter` 接收当前查询，返回一个 `Filter`。它是多租户与权限控制的挂载点，可在运行时按用户上下文动态决定过滤条件。

# 4. 元数据过滤与多租户

结论先行：RAG 检索默认是"全库相似度搜索"，如果不加过滤，A 租户的问题会检索到 B 租户的资料。生产 RAG 必须做租户隔离。

## 4.1 写入时打标签

```java
// 写入时打标签
TextSegment segment = TextSegment.from(
        text,
        Metadata.from(Map.of(
                "tenantId", tenantId,
                "department", dept)));
```

元数据必须在写入时确定，事后补需要重建整个索引。

### 4.1.1 元数据类型

| 类型 | 示例 | 用途 |
|------|------|------|
| 归属 | `tenantId`、`userId` | 隔离与权限 |
| 来源 | `source`、`url`、`page` | 引用与溯源 |
| 时效 | `updatedAt`、`version` | 时效过滤 |
| 分类 | `department`、`category` | 范围检索 |

## 4.2 检索时强制过滤

```java
// 检索时强制过滤
EmbeddingStoreContentRetriever.builder()
        .embeddingStore(store)
        .embeddingModel(embeddingModel)
        .dynamicFilter(query -> {
            String tenant = TenantContext.currentTenant();
            return metadataKey("tenantId").isEqualTo(tenant);
        })
        .build();
```

> 生产 RAG **必须做租户隔离**，否则会检索到其他租户的数据。详见 `5_RAG/DataProcessing/4_metadata_and_filtering.md`。

## 4.3 隔离方案对比

| 方案 | 隔离强度 | 运维成本 | 适用 |
|------|----------|----------|------|
| 元数据过滤（同表） | 中 | 低 | 租户多、数据量小 |
| 独立表/集合 | 高 | 中 | 租户少、合规要求高 |
| 独立实例 | 最高 | 高 | 强隔离、大客户 |

边界与坑：过滤条件必须来自服务端可信上下文，绝不能直接用客户端传入的 `tenantId`，否则过滤形同虚设。

# 5. RetrievalAugmentor（高级编排）

结论先行：默认检索流程可以拆成"查询改写 → 多路召回 → 聚合重排 → 注入"四步，用 `DefaultRetrievalAugmentor` 可以逐段替换。

## 5.1 四步流程

```
Query ─QueryTransformer─▶ 多个查询
          │
          ▼
     ContentRetriever（可多个）→ 多路召回
          │
          ▼
     ContentAggregator（合并去重）
          │
          ▼
     ContentInjector（注入提示）
```

## 5.2 DefaultRetrievalAugmentor 配置

```java
RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
        .queryTransformer(new ExpandingQueryTransformer(model, 3))
        .contentRetriever(retriever)
        .contentAggregator(ReRankingContentAggregator.builder()
                .scoringModel(scoringModel)
                .minScore(0.7)
                .build())
        .contentInjector(DefaultContentInjector.builder()
                .promptTemplate(PromptTemplate.from("""
                        依据以下资料回答问题，资料不足时明确说明：
                        {{contents}}
                        问题：{{userMessage}}
                        """))
                .build())
        .build();
```

## 5.3 自定义组件

每个环节都是接口，可自行实现：

| 接口 | 职责 | 可替换为 |
|------|------|----------|
| `QueryTransformer` | 改写/扩展查询 | 扩展、压缩、HyDE |
| `ContentRetriever` | 召回 | 向量、全文、混合 |
| `ContentAggregator` | 合并重排 | 去重、重排、截断 |
| `ContentInjector` | 注入提示 | 自定义模板 |

生产建议：先只替换"召回 + 重排"两个环节，收益最大；查询改写与注入模板留待评测后再调。

# 6. 查询改写（Query Transformation）

结论先行：用户问题往往口语化、含指代或信息不足，直接拿去检索召回差。查询改写把"用户问题"变成"适合检索的查询"。

## 6.1 扩展查询

```java
// 扩展查询：生成多个相关查询，提升召回
QueryTransformer transformer = new ExpandingQueryTransformer(model, 3);
```

它让模型生成 N 个语义相关的查询，分别检索后合并，缓解"单一措辞召回不全"。

## 6.2 压缩查询

```java
// 压缩查询：从对话历史中提取真正的检索意图
QueryTransformer compressor = new CompressingQueryTransformer(model);
```

多轮对话中，"那它的价格呢？"本身无法检索。压缩器结合历史，把它改写成"产品 X 的价格是多少"。

## 6.3 技术对比

| 技术 | 作用 | 成本 |
|------|------|------|
| 查询扩展 | 多角度召回 | 多次检索 |
| 查询压缩 | 多轮对话中提取核心问题 | 一次 LLM 调用 |
| HyDE | 先生成假设答案再检索 | 一次生成 + 一次检索 |

> 详见 `5_RAG/Retrieval/5_query_transformation.md`。

边界与坑：改写会引入延迟与额外成本；改写质量依赖模型，弱模型可能把查询改坏。生产建议对改写做 A/B 评测，并设置超时降级为原始查询。

# 7. 重排（Re-ranking）

结论先行：向量检索是"粗召回"，重排是"精排序"。策略是召回多、重排后取少，既保证召回率又控制注入量。

## 7.1 为什么要重排

向量相似度是"双塔"近似，查询与文档分别编码，无法捕捉细粒度交互。Cross-Encoder（交叉编码器）把查询与文档一起编码，精度更高但更慢，因此只用于少量候选。

## 7.2 实现

```java
ScoringModel scoringModel = new CohereScoringModel(...);   // 或本地 Cross-Encoder

ContentAggregator aggregator = ReRankingContentAggregator.builder()
        .scoringModel(scoringModel)
        .minScore(0.7)
        .build();
```

策略：**召回多（20）→ 重排 → 取少（5）喂给模型**。

## 7.3 边界与坑

- 重排模型与向量模型需配套评测，重排分数与向量分数不可直接比较。
- 重排增加一次推理，注意延迟预算。
- 重排后仍需 `minScore` 兜底，避免"矮子里拔将军"。

## 7.4 重排模型选型

| 类型 | 示例 | 特点 |
|------|------|------|
| 云端 API | Cohere Rerank | 开箱即用，按量计费 |
| 本地模型 | BGE Reranker | 数据不出域，需 GPU |
| LLM 打分 | 用 ChatModel 打分 | 灵活但慢且贵 |

# 8. 父子文档检索

结论先行：小块检索精度高但上下文不足，大块上下文全但检索噪声大。父子检索"用小块命中、用大块喂模型"，兼顾两者。

## 8.1 原理

```
入库：父块 P（完整段落）+ 子块 C1,C2,C3（句子级）
      C1,C2,C3 带 metadata: parentId=P

检索：命中 C2
      │
      ▼
      取 parentId=P 的完整父块内容
      │
      ▼
      注入 P（而非 C2）→ 上下文完整
```

## 8.2 实现要点

```java
// 存储时：父子块都存，用元数据关联
// 检索时：命中子块 → 取父块内容注入
```

生产建议：父块大小控制在模型窗口的合理比例内；父子关系用稳定 ID 关联，避免重建索引后错位。

> 详见 `5_RAG/Advanced/1_parent_document_retriever.md`。

# 9. 混合检索

结论先行：向量检索擅长语义，全文检索擅长关键词与专有名词。混合检索同时用两者，再融合排序，覆盖更全。

## 9.1 原理

```
         ┌─ 向量检索（语义）─┐
Query ───┤                    ├─▶ 融合（RRF/加权）─▶ Top-K
         └─ 全文检索（关键词）┘
```

## 9.2 实现

LangChain4j 提供 `EmbeddingStoreContentRetriever` 与全文检索结合的模式，或用多个检索器 + 聚合器：

```java
RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
        .contentRetriever(new DefaultContentRetriever(   // 多路
                List.of(vectorRetriever, fullTextRetriever)))
        .build();
```

## 9.3 融合策略

| 策略 | 说明 |
|------|------|
| RRF（倒数排名融合） | 按排名倒数求和，无需归一化 |
| 加权分数 | 需将两路分数归一化到同一区间 |
| 去重合并 | 简单，但忽略排序 |

> 详见 `5_RAG/Retrieval/3_hybrid_search.md`。

# 10. RAG 质量诊断

结论先行：RAG 出错只有两类——检索没召回，或召回了但模型答错。先定位环节，再针对性优化，避免盲目调参。

## 10.1 诊断流程

```
用户问 → 检索 → 生成
          │       │
          │       └─ 生成失败：有资料但答错 → 优化提示/换模型
          └─ 检索失败：资料没召回 → 优化分块/Embedding/查询改写/提高 topK
```

排查步骤：

1. 打印检索命中的 `Content`，看答案是否在其中；
2. 不在 → 检索问题；
3. 在但答错 → 生成问题。

## 10.2 常见问题

| 现象 | 可能原因 | 手段 |
|------|----------|------|
| 检索为空 | 切分过细、阈值过高 | 降低 minScore、加大片段 |
| 命中无关 | 切分过粗、模型不匹配 | 调整切分、换 Embedding |
| 多轮指代失败 | 未做查询压缩 | 启用压缩改写 |
| 答非所问 | 注入模板差 | 明确"资料不足则拒答" |
| 引用错误 | 未要求标注来源 | 提示中强制引用 |

## 10.3 评测指标

| 指标 | 含义 |
|------|------|
| 召回率 Recall@K | 正确答案是否在 Top-K |
| 精确率 Precision@K | Top-K 中相关比例 |
| 忠实度 Faithfulness | 答案是否基于检索内容 |
| 答案相关性 | 是否回答了问题 |

生产建议：建立黄金集（golden set），每次改动跑回归，用数据说话。

## 10.4 黄金集构建

黄金集是"问题 + 标准答案 + 应命中的片段"的集合：

| 字段 | 说明 |
|------|------|
| question | 测试问题，覆盖常见与边界 |
| expectedAnswer | 标准答案要点 |
| expectedSources | 应命中的文档/片段 ID |
| tenant | 租户，用于验证隔离 |

建议规模从几十条起步，逐步补充"线上答错"的案例，形成回归闭环。

# 11. 生产注意事项

结论先行：RAG 的生产风险集中在上下文超限、越权、幻觉引用、知识过期与延迟。逐项设防并配套评测。

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 上下文超限 | 控制 `maxResults` 与片段长度 |
| 检索越权 | 元数据强制过滤 |
| 幻觉引用 | 要求标注来源，无法引用则拒答 |
| 知识过期 | 增量更新 + 版本管理 |
| 无评测 | 建黄金集，度量召回与忠实度 |
| 检索延迟 | 向量库选型 + 缓存热点查询 |

## 11.2 性能优化

- 缓存：对高频问题缓存"检索结果"甚至"最终答案"；
- 并行：多路召回并行执行，减少串行延迟；
- 预过滤：优先用元数据缩小候选集，再算向量；
- 预热：启动时预热 Embedding 模型与连接池。

## 11.3 增量更新

文档会更新，索引必须能增量维护：

```java
// 以稳定 ID 标识片段：source + page + hash
// 更新时先删除同 source 的旧片段，再写入新片段
store.removeAll(idsOfSource("handbook.pdf"));
ingestor.ingest(newDocument);
```

边界与坑：若 ID 不稳定，重复入库会产生大量近重复片段，稀释检索质量。生产建议入库前做去重（文本哈希）。

# 12. 小结

## 12.1 核心要点

- LangChain4j RAG 组件齐全：`DocumentSplitter`、`EmbeddingStore`、`ContentRetriever`、`RetrievalAugmentor`。
- `EmbeddingStoreIngestor` 一行完成"加载→切分→向量化→入库"。
- 高级编排靠 `RetrievalAugmentor`：查询改写 → 多路召回 → 重排 → 注入。
- 元数据过滤是租户隔离与权限控制的关键。

## 12.2 实践清单

| 检查项 | 目标 |
|--------|------|
| 是否持久化向量库 | 重启不丢 |
| 是否做租户过滤 | 不越权 |
| 是否配置重排 | 提升精度 |
| 是否做查询改写 | 提升召回 |
| 是否要求引用 | 降低幻觉 |
| 是否有评测集 | 可回归 |
