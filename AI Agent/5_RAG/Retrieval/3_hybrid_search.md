# 1. 为什么需要混合检索

混合检索（Hybrid Search）= **稠密检索（语义）+ 稀疏检索（词面）**。它同时跑两路召回，再融合排序，用两者的互补性弥补单路检索的短板。当前它是**性价比最高的检索方案**，几乎是生产标配。

## 1.1 稠密与稀疏的互补

- **稠密检索**：把查询与文档映射到向量空间，擅长"同义改写、语义泛化"，但容易漏掉精确的专有名词与数字；
- **稀疏检索**（BM25 等）：基于词频与逆文档频率，擅长"精确匹配、专有名词、型号、条文"，但无法理解同义表达。

两者的失败模式几乎正交，因此融合后能显著扩大召回覆盖面。

## 1.2 典型查询对照

| 查询类型 | 稠密 | 稀疏 | 混合 |
|----------|:----:|:----:|:----:|
| 同义改写（"怎么报销"→"费用申请流程"） | ✅ | ❌ | ✅ |
| 专有名词（"A123 订单"） | ❌ | ✅ | ✅ |
| 产品型号（"iPhone 15 Pro Max"） | ⚠️ | ✅ | ✅ |
| 语义泛化（"合同到期怎么办"） | ✅ | ⚠️ | ✅ |
| 精确条文（"民法典第 1024 条"） | ❌ | ✅ | ✅ |
| 错别字/口语（"咋退钱"） | ✅ | ⚠️ | ✅ |

> 结论：单路总有一类查询会失手，混合是"覆盖面优先"的工程选择。

## 1.3 什么时候不需要混合

并非所有场景都必须混合：

- 纯语义问答、文档用词规范统一 → 单路稠密可能已足够；
- 纯关键词/编号检索（如日志、代码搜索）→ 单路 BM25 更直接；
- 极端低延迟要求且召回已达标 → 可先上线单路，按需再加。

判断标准只有一个：**用标注集评测召回率，不足则补一路**。

# 2. 整体流程

混合检索的标准流程是"**两路召回 → 融合 → 重排**"。

```
                    ┌──────────────┐
查询 ──┬──────────▶│ 稠密检索      │──▶ 候选列表 A（按相似度排序）
       │           └──────────────┘
       │           ┌──────────────┐
       └──────────▶│ 稀疏检索 BM25 │──▶ 候选列表 B（按 BM25 排序）
                   └──────────────┘
                          │
                          ▼
                   ┌──────────────┐
                   │  融合（RRF）  │──▶ 合并候选
                   └──────┬───────┘
                          ▼
                   ┌──────────────┐
                   │  重排 Rerank  │──▶ 最终 Top-N
                   └──────────────┘
```

## 2.1 两路召回

- 两路都要**并行**执行，避免延迟叠加；
- 各自 topK 要偏大（20~50），因为融合是"取长补短"，召回不足则融合无意义。

## 2.2 融合

融合的核心问题是：**两路分数不可比**（余弦相似度 vs BM25 分数）。解决思路有两条：

- **只比排名**：RRF（推荐）；
- **先归一化再加权**：加权分数融合。

## 2.3 重排

融合后候选通常仍有 20~30 条，用 Cross-Encoder 重排器（Rerank）精排，取 Top-N（3~5）喂给模型。重排能显著提升精度。

> 重排细节见 `5_RAG/Retrieval/4_reranking.md`。

## 2.4 完整数据流

```
question
  ├─ 向量化 ─▶ 向量库 ─▶ A[(doc,score)]
  └─ 分词   ─▶ BM25   ─▶ B[(doc,score)]
                              │
                    RRF 融合 ─┘
                              │
                      重排 ───┘
                              │
                        Top-N ─▶ 组装上下文
```

# 3. 融合方法一：RRF（推荐）

## 3.1 公式与直觉

RRF（Reciprocal Rank Fusion）只依赖**排名**，不依赖分数，因此天然解决"两种检索分数不可比"的问题。

```
RRF(d) = Σ   1 / (k + rank_i(d))
        i
```

| 符号 | 含义 |
|------|------|
| `rank_i(d)` | 文档 d 在第 i 个检索结果中的排名（从 1 开始） |
| `k` | 平滑常数，通常取 60 |

直觉：**排名越靠前贡献越大，且贡献随排名衰减**；两路都靠前的文档会获得叠加分数。

### 3.1.1 为什么只看排名

余弦相似度（0~1）与 BM25 分数（无上界）量纲不同，直接相加会失真。RRF 抛弃分数、只保留排名，从而**天然免疫量纲差异**，代价是忽略分数差距。

## 3.2 Java 实现

### 3.2.1 基础版

```java
public List<String> rrfFuse(List<List<String>> rankedLists, int k) {
    Map<String, Double> scores = new HashMap<>();

    for (List<String> list : rankedLists) {
        for (int rank = 0; rank < list.size(); rank++) {
            scores.merge(list.get(rank), 1.0 / (k + rank + 1), Double::sum);
        }
    }
    return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .map(Map.Entry::getKey)
            .toList();
}
```

### 3.2.2 支持权重的加权 RRF

当两路可信度不同（如稠密更准），可给每路一个权重：

```java
public List<String> weightedRrf(List<List<String>> rankedLists,
                                List<Double> weights, int k) {
    Map<String, Double> scores = new HashMap<>();
    for (int i = 0; i < rankedLists.size(); i++) {
        List<String> list = rankedLists.get(i);
        double w = weights.get(i);
        for (int rank = 0; rank < list.size(); rank++) {
            scores.merge(list.get(rank), w / (k + rank + 1), Double::sum);
        }
    }
    return sortDesc(scores);
}
```

## 3.3 k 值的选择

| k | 效果 |
|---|------|
| 小（如 1~10） | 头部排名权重极大，尾部几乎无贡献 |
| 60（默认） | 平衡，业界经验值 |
| 大（>100） | 各排名差异被拉平，接近"投票计数" |

> 无特殊需求就用 60；若两路质量差异大，优先用加权 RRF 而非改 k。

## 3.4 优缺点

**优点**：无需分数归一化，鲁棒，实现简单，几乎无调参。
**缺点**：忽略分数差距（第 1 名与第 2 名差距可能很大，但 RRF 只看排名）。

# 4. 融合方法二：加权分数融合

## 4.1 公式

```
finalScore = α × normalize(denseScore) + (1-α) × normalize(sparseScore)
```

```java
public List<Result> weightedFuse(List<ScoredDoc> dense,
                                 List<ScoredDoc> sparse,
                                 double alpha) {
    Map<String, Double> scores = new HashMap<>();
    dense.forEach(d -> scores.merge(d.id(),
            alpha * minMaxNormalize(d.score(), dense), Double::sum));
    sparse.forEach(d -> scores.merge(d.id(),
            (1 - alpha) * minMaxNormalize(d.score(), sparse), Double::sum));

    return sortDesc(scores);
}
```

## 4.2 归一化方法对比

### 4.2.1 min-max 归一化

```
x' = (x - min) / (max - min)
```

简单直观，但对**离群值敏感**（一个极高分会让其余分数被压扁）。

### 4.2.2 z-score 标准化

```
x' = (x - mean) / std
```

对分布更稳定，但要求分数近似正态；若分布长尾，效果一般。

| 方法 | 优点 | 缺点 |
|------|------|------|
| min-max | 实现简单、范围固定 [0,1] | 对离群值敏感 |
| z-score | 抗离群 | 范围不固定，可能为负 |

## 4.3 α 调优

- α = 0.5：两路等权；
- α = 0.6~0.7：稠密略高（多数语义场景的经验值）；
- α 应按**标注集网格搜索**确定，而非拍脑袋。

## 4.4 优缺点

**优点**：可利用分数差距做精细排序。
**缺点**：归一化方式与 α 都需调优，且两路分数分布差异大时不稳定。

> 实践建议：**优先 RRF**，需要精细控制时再用加权融合。

# 5. 三路及以上融合

融合框架天然支持多路。常见第三路是 **HyDE**：先让模型生成一段"假设答案"，再用它去检索，提升语义召回。

## 5.1 HyDE 扩展

```java
String hydeDoc = llm.generate("请写一段可能回答该问题的文字：" + question);
```

## 5.2 多路融合实现

```java
List<List<String>> rankedLists = List.of(
        denseRetrieve(question, 20),      // 向量
        sparseRetrieve(question, 20),     // BM25
        denseRetrieve(hydeDoc, 20)        // HyDE 假设文档
);

List<String> fused = rrfFuse(rankedLists, 60);
```

> 路数不是越多越好：每加一路都会增加延迟与调参成本，按评测收益决定。

# 6. Spring AI 中的混合检索

## 6.1 手动融合

Spring AI 的 `VectorStore` 提供稠密检索，稀疏检索通常借助外部引擎（ES / Lucene）实现，再手动融合：

```java
public List<Document> hybridSearch(String question, int topK) {
    List<Document> dense = vectorStore.similaritySearch(
            SearchRequest.builder().query(question).topK(20).build());

    List<Document> sparse = fullTextSearch(question, 20);

    return rrfFuse(dense, sparse).stream().limit(topK).toList();
}
```

## 6.2 完整示例

```java
@Service
public class HybridRetriever {

    private final VectorStore vectorStore;
    private final FullTextSearcher fullTextSearcher;

    public List<Document> retrieve(String question, int topK) {
        // 两路并行召回
        CompletableFuture<List<Document>> denseF =
                CompletableFuture.supplyAsync(() -> vectorStore.similaritySearch(
                        SearchRequest.builder().query(question).topK(30).build()));
        CompletableFuture<List<Document>> sparseF =
                CompletableFuture.supplyAsync(() -> fullTextSearcher.search(question, 30));

        List<Document> dense = denseF.join();
        List<Document> sparse = sparseF.join();

        // RRF 融合
        List<String> fused = rrfFuse(
                List.of(ids(dense), ids(sparse)), 60);

        return topN(fused, topK, union(dense, sparse));
    }
}
```

### 6.2.1 异常处理

两路并行时，任一路失败应**降级为单路结果**而非整体失败：

```java
List<Document> dense = safeJoin(denseF, List.of());
List<Document> sparse = safeJoin(sparseF, List.of());
if (dense.isEmpty() && sparse.isEmpty()) {
    throw new RetrievalException("检索全部失败");
}
```

## 6.3 与 VectorStore 配合

- `topK` 设大（20~50）保证召回；
- 融合前先按 id 建立 `Map<id, Document>` 便于回查原文；
- 融合后交给 Rerank Advisor 或手动重排。

> ES 原生支持 RRF Retriever，可直接在引擎侧融合，减少应用层代码。

# 7. LangChain4j 中的混合检索

## 7.1 组合检索器

LangChain4j 通过 `RetrievalAugmentor` 组合多个检索器：

```java
RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
        .contentRetriever(new DefaultContentRetriever(
                List.of(vectorRetriever, fullTextRetriever)))
        .contentAggregator(new ReRankingContentAggregator(scoringModel))
        .build();
```

## 7.2 融合与重排

`DefaultContentRetriever` 支持多检索器合并，`ReRankingContentAggregator` 负责重排。若需要自定义融合策略，可实现 `ContentAggregator` 接口。

# 8. 参数调优

## 8.1 关键参数

| 参数 | 建议 | 说明 |
|------|------|------|
| 各检索器 topK | 20~50 | 召回要足 |
| RRF `k` | 60 | 常用默认值 |
| 融合后候选数 | 20~30 | 供重排使用 |
| 最终 topN | 3~5 | 喂给模型 |
| 加权 α | 0.5~0.7 | 稠密通常略高 |

## 8.2 调优方法

1. 准备带标注的查询集（query → 相关文档 id）；
2. 固定重排器，网格搜索各 topK、`k`、α；
3. 以 Recall@N、nDCG@N 为目标，选最优组合；
4. 在留出集上验证，防止过拟合。

### 8.2.1 防止过拟合

调参只在**训练集**上搜索，最终指标以**留出集**为准；若两集差距过大，说明参数过拟合，应回退到更保守的默认值（如 `k=60`、α=0.5）。

## 8.3 评测指标

| 指标 | 衡量 |
|------|------|
| Recall@N | 相关文档是否被召回 |
| nDCG@N | 排序质量 |
| MRR | 首个相关结果的位置 |
| 延迟 P99 | 工程约束 |

# 9. 效果对比（经验值）

| 方案 | 相对召回率 |
|------|:----------:|
| 纯稠密 | 基线 |
| 纯 BM25 | 基线 ± |
| 混合（RRF） | +10%~25% |
| 混合 + 重排 | +15%~35% |

> 具体提升幅度取决于数据与查询类型，务必用自有数据评测。

# 10. 生产注意事项

## 10.1 索引同步

稠密与稀疏索引必须**一起更新**，否则会出现"一路召回新文档、另一路没有"的偏差。建议用同一份文档管道同时写入两个索引。

### 10.1.1 同步实现方式

- **同管道双写**：解析后同时写入向量库与搜索引擎，用同一 `docId`；
- **变更补偿**：删除/更新时按 `docId` 双删双改，失败进入重试队列；
- **一致性校验**：定期对账两边的文档数量与版本。

## 10.2 延迟与并发

- 两路检索**并行**，总延迟取较慢一路；
- 为每路设超时，一路失败时可降级为单路结果；
- 向量库与搜索引擎的连接池要分别配置。

## 10.3 常见坑

| 风险 | 建议 |
|------|------|
| 分数直接相加 | 用 RRF，避免归一化陷阱 |
| 只做一路召回 | 至少稠密 + 稀疏 |
| 融合后不重排 | 加 Rerank 提升精度 |
| 索引不同步 | 稠密与稀疏索引一起更新 |
| 无评测 | 用标注集验证提升 |
| 延迟增加 | 两路并行检索 |
| 融合丢原文 | 维护 id→文档映射 |

# 11. 小结

- 混合检索 = 稠密（语义）+ 稀疏（词面），两者失败模式正交，互补短板。
- 融合首选 **RRF**，天然规避分数不可比问题，几乎免调参。
- 需要精细控制时用加权融合，但要处理归一化并调 α。
- 支持多路融合（如加 HyDE），但按评测收益决定是否加路。
- 混合 + 重排是当前性价比最高的组合，生产几乎必用。
