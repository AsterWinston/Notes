# 1. 为什么需要重排

**结论先行**：向量检索是"粗排"，它用双塔模型（Bi-Encoder，双编码器）分别编码查询与文档，速度快但缺少 token 级交互；重排（Reranking，重排序）用交叉编码器（Cross-Encoder，交叉编码器）把查询与文档一起编码，精度高但慢。只有把"快"与"准"分两阶段组合，才能同时兼顾召回率与最终精度。

## 1.1 双塔与交叉编码的本质差异

重排之所以有效，根源在于两种编码范式的结构差异，而非"换了个更大的模型"。

### 1.1.1 双塔（Bi-Encoder）机制

双塔模型把查询和文档**分别**送入编码器，得到两个独立向量，再用余弦相似度或点积计算分数。文档向量可以离线预计算并建立索引，因此在线检索只需编码查询一次，再做近似最近邻（ANN，Approximate Nearest Neighbor）搜索。

```
双塔（Bi-Encoder）：
  query → [E] → q_vec ┐
                       ├─ cosine(q_vec, d_vec) → score
  doc   → [E] → d_vec ┘
  文档向量可预计算，查询仅编码一次
```

关键直觉：查询与文档之间**没有任何直接交互**，全部语义压缩进两个固定维度向量，信息必然有损。

### 1.1.2 交叉编码（Cross-Encoder）机制

交叉编码器把查询与文档**拼接后一起**送入编码器，让自注意力（Self-Attention，自注意力）在两者 token 之间充分交互，最后输出一个相关性分数。它无法预计算文档向量，每对 (query, doc) 都要完整前向一次。

```
交叉编码（Cross-Encoder）：
  [query + doc] → [E] → score
  两者一起编码，token 级充分交互
  精度高，但需逐对计算，无法预计算
```

关键直觉：注意力矩阵能看到"查询的哪个词对应文档的哪个词"，因而对否定、条件、数字等细节更敏感。

### 1.1.3 数学形式对比

双塔的相似度可以写成：

```
s(q, d) = cos(E_q(q), E_d(d)) = <E_q(q), E_d(d)> / (||E_q(q)||·||E_d(d)||)
```

交叉编码则直接建模条件概率或回归分数：

```
s(q, d) = f_θ([CLS] q [SEP] d [SEP])
```

其中 `f_θ` 是带分类头的 Transformer。差别在于：前者把交互推迟到点积，后者在编码阶段就完成交互。这解释了为什么交叉编码更准、也更慢。

## 1.2 速度与精度的权衡

| 维度 | 双塔（召回） | 交叉编码（重排） |
|------|:------------:|:----------------:|
| 速度 | 快（文档可预计算） | 慢（逐对计算） |
| 精度 | 中 | 高 |
| 索引 | 支持 ANN 索引 | 无索引，线性扫描 |
| 复杂度 | O(1) 查询编码 | O(N) 逐对前向 |
| 用法 | 召回 Top-100 | 精排 Top-100 → Top-5 |

工程上的典型量级：

| 阶段 | 处理规模 | 典型延迟 |
|------|----------|----------|
| 双塔召回 | 百万级文档 | 10~50ms |
| 交叉编码重排 | 50 个候选 | 100~500ms |
| 生成 | 5 个片段 | 1~5s |

> 标准做法：**召回多（20~100）→ 重排 → 取少（3~5）**。

## 1.3 召回与精排的分工

召回阶段追求**高召回率**：宁可放进一些噪声，也不能漏掉正确文档，因为它无法补回没召回的文档。重排阶段追求**高精度**：在候选集里把真正相关的排到前面。二者目标不同，因此评测指标也不同（召回用 Recall@K，重排用 nDCG/MRR）。

用一句话概括：**召回决定上限，重排逼近上限**。

## 1.4 何时重排收益最大

```
开始
  │
  ▼
召回 Top-K 是否包含正确文档？ ── 否 ──► 修召回，重排无用
  │ 是
  ▼
候选数是否 > 10？ ── 否 ──► 收益有限
  │ 是
  ▼
排序是否正确？ ── 否 ──► 重排收益显著
  │ 是
  ▼
收益有限，可考虑跳过重排
```

- 候选多且噪声大：重排能显著提纯；
- 查询与文档表述差异大：交叉编码能捕捉语义对应；
- 需要严格排序（如只喂 3 条给模型）：排序质量直接决定答案质量；
- 召回已经很准且候选很少：重排收益有限，甚至不值得多一次调用。

# 2. 两阶段检索架构

**结论先行**：生产级 RAG 检索几乎都是"粗排召回 → 精排重排 → 少量喂给生成模型"的两阶段（或三阶段）架构。理解每一阶段的输入输出与目标，是排查检索问题的前提。

## 2.1 整体流程

```
查询
  │
  ▼
① 召回（双塔 / BM25 / 混合）→ 100 个候选      ← 要"全"
  │
  ▼
② 重排（交叉编码）→ 打分排序 → 取 5 个         ← 要"准"
  │
  ▼
③ 喂给模型
```

## 2.2 召回阶段

召回可以来自稠密检索（向量）、稀疏检索（BM25，Best Matching 25）、或二者混合。目标是把正确文档"捞进"候选池。候选数常用 20~100，取决于重排预算。

召回阶段的常见问题：

| 症状 | 可能原因 |
|------|----------|
| 正确文档不在候选池 | embedding 不适配、分块不合理 |
| 候选池全是近义噪声 | 向量维度/归一化问题 |
| 中文检索差 | 未用中文或多语言模型 |

## 2.3 重排阶段

重排对每个候选与查询计算相关性分数，按分排序后取 Top-N。可选加阈值（minScore）过滤明显不相关的结果。此阶段是延迟与费用的主要来源。

重排的输入是"候选列表"，输出是"有序列表 + 分数"。分数是相对值，**不要跨模型、跨查询直接比较绝对值**。

## 2.4 生成阶段

把 Top-N 片段拼进提示词（Prompt，提示词）交给大模型。片段越少越省 token、噪声越低，但太少可能缺信息，通常 3~5 条。

拼接时建议为每个片段标注来源，便于模型引用与用户溯源：

```java
String context = segments.stream()
        .map(s -> "【来源：" + s.metadata().getString("source") + "】\n" + s.text())
        .collect(Collectors.joining("\n\n"));
```

# 3. 重排模型选择

**结论先行**：选型时优先考虑"语言覆盖、部署方式（云/本地）、与召回 embedding 的匹配度"，而不是只看榜单分数。

## 3.1 商业 API 模型

| 模型 | 说明 |
|------|------|
| Cohere Rerank | 商业 API，效果好，多语言 |
| Jina Reranker | 商业/开源双版本 |
| Voyage Rerank | 商业，长文档友好 |
| 各家云厂商重排 | 与自家检索栈集成 |

商业 API 优点：开箱即用、免运维；缺点：数据出网、按量计费、有单点风险。

## 3.2 开源模型

| 模型 | 说明 |
|------|------|
| BGE Reranker | 开源，中文友好 |
| bge-reranker-v2-m3 | 多语言，可本地部署 |
| bge-reranker-large | 精度更高，算力更大 |
| Cross-Encoder/ms-marco | 英文经典基线 |

开源模型可内网部署、数据不出域，但需要 GPU 与运维能力。

## 3.3 生成式重排（MonoT5 / RankGPT）

| 模型 | 思路 |
|------|------|
| MonoT5 | 把重排转为"相关/不相关"生成任务 |
| RankGPT | 让 LLM 直接对候选列表排序 |
| RankZephyr | RankGPT 的开源复现 |

RankGPT 的提示词思路（伪代码）：

```java
// 伪代码：让 LLM 输出候选的排序编号
String prompt = """
        给定问题：%s
        请按相关性从高到低排列下列候选文档，只输出编号列表：
        [1] %s
        [2] %s
        ...
        """.formatted(query, doc1, doc2);

// 解析返回的编号列表，重排候选
```

生成式重排灵活但延迟高，适合离线或低并发场景。

## 3.4 选型对比与建议

| 维度 | 商业 API | 开源本地 |
|------|:--------:|:--------:|
| 上手速度 | 快 | 慢 |
| 数据合规 | 需评估 | 可控 |
| 单次成本 | 按量 | 算力折旧 |
| 延迟稳定性 | 受网络影响 | 可控 |
| 可定制 | 低 | 高（可微调） |

建议：中小团队先用商业 API 验证收益，收益确认且量级上来后再评估自建。选型时务必让**重排模型与 embedding 模型做一次对比评测**，同系列通常更协调。

# 4. 实现方式

**结论先行**：LangChain4j 通过 `ScoringModel` + `ReRankingContentAggregator` 提供开箱重排；若框架不满足需求，可手动实现"逐对打分 → 排序 → 截断"三步。

## 4.1 LangChain4j 的 ScoringModel

### 4.1.1 Cohere 集成

```java
ScoringModel scoringModel = CohereScoringModel.builder()
        .apiKey(System.getenv("COHERE_API_KEY"))
        .modelName("rerank-multilingual-v3.0")
        .build();
```

### 4.1.2 ReRankingContentAggregator

```java
ContentAggregator aggregator = ReRankingContentAggregator.builder()
        .scoringModel(scoringModel)
        .minScore(0.7)
        .build();

RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
        .contentRetriever(retriever)
        .contentAggregator(aggregator)
        .build();
```

`ReRankingContentAggregator` 会在检索后自动调用 `ScoringModel`，按分数重排并过滤低于 `minScore` 的结果。`minScore` 需用标注数据校准，拍脑袋设定容易误杀。

## 4.2 手动重排

当框架版本不支持某能力时，手动实现最直观：

```java
public List<Document> rerank(String query, List<Document> candidates, int topN) {
    List<ScoredDocument> scored = new ArrayList<>();

    for (Document doc : candidates) {
        double score = rerankModel.score(query, doc.getText());
        scored.add(new ScoredDocument(doc, score));
    }

    return scored.stream()
            .sorted(Comparator.comparingDouble(ScoredDocument::score).reversed())
            .limit(topN)
            .map(ScoredDocument::document)
            .toList();
}

record ScoredDocument(Document document, double score) {}
```

## 4.3 批量重排（性能）

逐对调用会放大网络往返，批量接口能显著降低延迟：

```java
// 一次请求重排多个文档对，减少网络往返
List<Double> scores = rerankModel.scoreBatch(query,
        candidates.stream().map(Document::getText).toList());
```

批量时注意 API 的**单请求文档数上限**，超限需分片并发，并控制并发度避免触发限流。

## 4.4 Spring AI 风格实现（伪代码）

Spring AI 截至本文暂无统一的 Rerank 抽象，下面用伪代码示意接入方式（API 名称以官方为准）：

```java
// 伪代码：Spring AI 未提供统一 Rerank 接口，需自行封装
public class RerankService {

    private final RestClient restClient;

    public RerankService(RestClient.Builder builder, String apiKey) {
        this.restClient = builder
                .baseUrl("https://api.example-rerank.com")
                .defaultHeader("Authorization", "Bearer " + apiKey)
                .build();
    }

    public List<Document> rerank(String query, List<Document> docs, int topN) {
        RerankRequest req = new RerankRequest(query,
                docs.stream().map(Document::getText).toList(), topN);

        RerankResponse resp = restClient.post()
                .uri("/v1/rerank")
                .body(req)
                .retrieve()
                .body(RerankResponse.class);

        return resp.results().stream()
                .map(r -> docs.get(r.index()))
                .toList();
    }
}
```

若走 Spring AI 的 `ChatClient` 做 LLM 重排，可用结构化输出约束返回排序编号，但成本与延迟都远高于专用重排模型，仅适合兜底。

# 5. 本地重排模型（ONNX）

**结论先行**：ONNX（Open Neural Network Exchange，开放神经网络交换格式）让重排模型可脱离 Python 运行时，在 JVM 内推理，适合内网与低延迟场景。

## 5.1 ONNX 加载

```java
ScoringModel scoringModel = new OnnxScoringModel(
        Path.of("models/bge-reranker-base.onnx"),
        Path.of("models/tokenizer.json"));
```

优点：无网络依赖、数据不出内网；缺点：需算力、需自行管理模型与分词器版本。

## 5.2 分词与推理细节

本地推理必须保证**分词器与训练时一致**，否则分数会漂移。常见坑：特殊 token、最大长度截断、padding 策略不一致。

```java
// 伪代码：确认 tokenizer 配置与模型配套
HuggingFaceTokenizer tokenizer = HuggingFaceTokenizer.newInstance(
        Path.of("models/tokenizer.json"));

// 截断到模型支持的最大长度，避免越界
Encoding enc = tokenizer.encode(query, doc,
        true, true); // addSpecialTokens, addPairTokens
```

建议把分词配置与模型文件一起版本化。

## 5.3 部署与显存

| 方式 | 优点 | 缺点 |
|------|------|------|
| CPU | 部署简单 | 延迟高、吞吐低 |
| GPU | 吞吐高 | 显存成本高 |
| INT8 量化 | 省显存 | 少量精度损失 |
| FP16 | 平衡 | 需硬件支持 |

- CPU 推理：低并发可用，但延迟高；
- GPU 推理：吞吐高，注意显存与批大小；
- 量化（INT8/FP16）：降低显存与延迟，但可能损失少量精度，需评测。

# 6. 重排的价值

**结论先行**：重排的收益取决于"召回候选的质量分布"，它能把好文档提前，但不能创造没被召回的好文档。

## 6.1 收益场景

| 场景 | 提升 |
|------|------|
| 召回结果多且噪声大 | 显著（精度提升明显） |
| 查询与文档表述差异大 | 显著 |
| 需要严格 Top-K 排序 | 显著 |
| 多语言/跨域查询 | 较明显 |

## 6.2 无收益场景

- 召回已经很准：重排后排序几乎不变；
- 候选只有 3~5 条：排序空间小；
- 正确文档根本没被召回：重排无能为力。

> **召回质量差时，重排救不回来**。先修召回，再谈重排。

## 6.3 量化收益的指标

| 指标 | 含义 |
|------|------|
| nDCG@K | 归一化折损累计增益，关注排序质量 |
| MRR | 平均倒数排名，关注首个正确项 |
| Recall@K | 前 K 是否包含正确文档 |
| Hit Rate | 命中率 |

nDCG 的核心思想是"越靠前的相关文档权重越高"：

```
DCG@K  = Σ  (2^rel_i - 1) / log2(i + 1)
        i=1..K
nDCG@K = DCG@K / IDCG@K     （IDCG 为理想排序的 DCG）
```

重排主要改善 nDCG 与 MRR，而非 Recall。

# 7. 重排的成本

**结论先行**：重排是"用延迟和金钱换精度"，必须把成本计入预算，否则会在高并发下拖垮系统。

## 7.1 延迟成本

| 成本 | 说明 |
|------|------|
| 延迟 | 交叉编码逐对计算，100 对可能 100~500ms |
| 费用 | 商业 API 按量计费 |
| 算力 | 本地模型需 GPU |

## 7.2 费用成本

商业 API 按"文档数 × 查询数"计费。若每次重排 50 个文档、每天 10 万次查询，费用会迅速累积。务必先估算再上线。

粗略估算公式：

```
日费用 ≈ 日均查询数 × 每次文档数 × 单文档单价
```

## 7.3 算力成本

本地模型的一次前向计算量与文档长度正相关。长文档重排既慢又贵，可考虑先截断到关键段落。

## 7.4 成本优化手段

- 控制候选数（20~50，不要 100+）；
- 批量调用；
- 只在必要时重排（如结果分数接近时）；
- 缓存重复查询的重排结果；
- 分级：简单查询不重排，复杂查询才重排；
- 惰性重排：仅当召回分数接近（难分伯仲）时才重排，否则沿用召回顺序。

# 8. 与混合检索配合

**结论先行**：混合检索（稠密 + 稀疏）负责"捞全"，重排负责"排准"，二者组合是当前工程上最优的检索链路。

## 8.1 RRF 融合

RRF（Reciprocal Rank Fusion，倒数排名融合）只依赖排名而非分数，天然规避了不同检索器分数量纲不一致的问题：

```
score(d) = Σ  1 / (k + rank_i(d))
          i
```

其中 `k` 通常取 60，用于平滑高排名项的权重。实现时对每个文档累加各路倒数排名分，再按总分降序取 Top-K。

## 8.2 完整链路

```
稠密检索 Top-20  ┐
                  ├─ RRF 融合 → Top-20 → 重排 → Top-5 → 模型
稀疏检索 Top-20  ┘
```

## 8.3 常见误区

- 直接相加稠密与稀疏分数：量纲不同，需归一化或用 RRF；
- 融合后再重排却仍用融合分数：重排应重新打分，不要混用旧分；
- 忽略去重：同一文档可能被两路召回，需按 ID 去重。

# 9. 参数调优

**结论先行**：召回候选数、重排 topN、minScore 三个参数必须用标注集评测驱动，不能凭经验拍定。

## 9.1 关键参数

| 参数 | 建议 |
|------|------|
| 召回候选数 | 20~50 |
| 重排后 topN | 3~5 |
| minScore | 用标注数据校准 |
| 批量大小 | 按 API 限制 |
| RRF k | 60（默认起点） |

## 9.2 评测驱动调参

```java
public void tuneRerank(List<EvalCase> cases) {
    for (int candidates : new int[]{10, 20, 50}) {
        for (int topN : new int[]{3, 5, 8}) {
            double ndcg = evaluate(cases, candidates, topN);
            log.info("candidates={} topN={} nDCG={}", candidates, topN, ndcg);
        }
    }
}
```

## 9.3 阈值校准

`minScore` 过高会误杀（漏掉正确文档），过低则失去过滤意义。做法：在标注集上画 Precision-Recall 曲线，扫描阈值并选 F1 最优点。

# 10. 生产注意事项

**结论先行**：重排的风险集中在"延迟、成本、可用性、期望管理"四方面，需通过监控与降级预案控制。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 候选过多导致延迟高 | 控制 20~50 |
| 重排模型与 Embedding 不匹配 | 选同系列或评测对比 |
| 无阈值 | 用标注数据校准 |
| 单点 API | 备用重排模型 |
| 忽略成本 | 监控调用量与费用 |
| 期望过高 | 召回差时先修召回 |

## 10.2 监控指标

| 指标 | 用途 |
|------|------|
| 重排延迟 P50/P95/P99 | 发现性能退化 |
| 调用量 / 费用 | 成本控制 |
| 分数分布 | 判断阈值是否合理 |
| Top-N 命中率 | 评估排序质量 |

## 10.3 降级策略

当重排 API 超时或报错时，应回退到"按召回分数直接取 Top-N"，保证主链路可用。降级要有开关与告警，而不是静默失败。

```java
public List<Document> safeRerank(String query, List<Document> candidates, int topN) {
    try {
        return rerankService.rerank(query, candidates, topN);
    } catch (Exception e) {
        log.warn("rerank failed, fallback to recall order", e);
        metrics.counter("rerank.fallback").increment();
        return candidates.stream().limit(topN).toList();
    }
}
```

# 11. 小结

- 重排用交叉编码精排，精度显著高于双塔。
- 标准流程：召回多 → 重排 → 取少。
- 重排能提升精度，但**救不了糟糕的召回**。
- 候选数、topN、阈值都要评测调优。
- 与混合检索配合是当前最优组合。
- 生产上要监控延迟与成本，并准备降级方案。
