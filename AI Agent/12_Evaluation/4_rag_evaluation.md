# 1. RAG 评测的两个层次

结论先行：RAG（Retrieval-Augmented Generation，检索增强生成）系统的质量由"检索"与"生成"两段共同决定，必须分层评测，否则出现问题时无法判断是"没找到"还是"找到了但答错"。

```
① 检索评测：能否召回正确答案所在的文档
② 生成评测：基于检索内容，回答是否忠实、相关、正确
```

> 先分层评测，再端到端评测。

## 1.1 为什么要分层

### 1.1.1 两类失败模式

RAG 的失败模式（failure mode）可归为两类：检索失败与生成失败。若只做端到端评测，一个低分会掩盖真实原因。

- 检索失败：正确文档根本没进上下文，生成模型再强也无能为力。
- 生成失败：正确文档已在上下文中，但模型忽略、误解或编造。

分层评测把这两类问题解耦，使指标可归因、优化有方向。

## 1.2 评测流水线

一条完整的 RAG 评测流水线包含：数据集、检索器、生成器、判官（Judge）、报告五部分。

```
 评测集(问题+标准答案+相关文档ID)
            │
            ▼
    ┌───────────────┐
    │  检索器        │──► 检索指标(Recall/MRR/nDCG)
    └───────┬───────┘
            │ 上下文
            ▼
    ┌───────────────┐
    │  生成器        │──► 生成指标(忠实/相关)
    └───────┬───────┘
            │ 答案
            ▼
    ┌───────────────┐
    │  端到端评测    │──► 正确率/拒答/幻觉
    └───────────────┘
```

评测集的最小字段：

```java
public record EvalCase(
        String id,
        String question,
        String groundTruth,          // 标准答案，可为 null（无答案题）
        List<String> relevantDocIds, // 相关文档 ID，检索指标依赖它
        List<String> requiredFacts   // 可选：答案必须包含的事实点
) {}
```

边界与坑：`relevantDocIds` 缺失时检索指标无法计算，必须在建集阶段标注；`groundTruth` 为 `null` 表示"应拒答"的负样本。

生产建议：把评测集版本化（如 `eval-v1.json`），每次改动检索或提示都跑同一套集，保证可比。

# 2. 检索指标

结论先行：检索指标衡量"正确文档是否被排到前面"，核心是 Recall@K、Precision@K、MRR、nDCG@K 四个。

| 指标 | 定义 | 公式 |
|------|------|------|
| Recall@K | 正确文档在前 K 个中 | 命中数 / 总数 |
| Precision@K | 前 K 个中相关占比 | 相关数 / K |
| MRR | 首个相关结果排名倒数均值 | mean(1/rank) |
| nDCG@K | 考虑位置的加权 | 见下 |

## 2.1 Recall@K 与 Precision@K

### 2.1.1 定义与权衡

机制：Recall@K 关心"有没有漏"，Precision@K 关心"有没有混入噪声"。二者常此消彼长：K 越大召回越高、精确越低。

### 2.1.2 Java 实现

```java
public double recallAtK(List<EvalCase> cases, int k, Retriever retriever) {
    int hit = 0;
    for (EvalCase c : cases) {
        List<Document> retrieved = retriever.retrieve(c.question(), k);
        if (retrieved.stream().anyMatch(d ->
                c.relevantDocIds().contains(d.getId()))) {
            hit++;
        }
    }
    return hit / (double) cases.size();
}

public double precisionAtK(List<EvalCase> cases, int k, Retriever retriever) {
    double sum = 0;
    for (EvalCase c : cases) {
        List<Document> retrieved = retriever.retrieve(c.question(), k);
        long relevant = retrieved.stream()
                .filter(d -> c.relevantDocIds().contains(d.getId()))
                .count();
        sum += relevant / (double) k;
    }
    return sum / cases.size();
}
```

边界与坑：当相关文档数大于 1 时，`anyMatch` 的 Recall@K 会高估"全召回"；严格版本应计算 `召回的相关文档数 / 相关文档总数`。

## 2.2 MRR（Mean Reciprocal Rank，平均倒数排名）

机制：只奖励"第一个相关结果出现得早"，适合问答类只取 Top-1 的场景。

```java
public double mrr(List<EvalCase> cases, Retriever retriever) {
    double sum = 0;
    for (EvalCase c : cases) {
        List<Document> retrieved = retriever.retrieve(c.question(), 10);
        for (int i = 0; i < retrieved.size(); i++) {
            if (c.relevantDocIds().contains(retrieved.get(i).getId())) {
                sum += 1.0 / (i + 1);
                break;
            }
        }
    }
    return sum / cases.size();
}
```

| 首个相关位置 | 倒数排名贡献 |
|:---:|:---:|
| 1 | 1.000 |
| 2 | 0.500 |
| 3 | 0.333 |
| 5 | 0.200 |
| 未命中 | 0.000 |

## 2.3 nDCG@K（Normalized Discounted Cumulative Gain，归一化折损累计增益）

机制：相关性有等级（0/1/2/3），越靠前权重越大，再做理想排序归一化。

```
DCG@K  = Σ_{i=1..K} (2^rel_i - 1) / log2(i + 1)
IDCG@K = 理想排序下的 DCG@K
nDCG@K = DCG@K / IDCG@K
```

```java
private double dcg(List<Integer> gains, int k) {
    double dcg = 0;
    for (int i = 0; i < Math.min(k, gains.size()); i++) {
        dcg += (Math.pow(2, gains.get(i)) - 1) / (Math.log(i + 2) / Math.log(2));
    }
    return dcg;
}

public double ndcgAtK(List<Integer> gains, int k) {
    List<Integer> ideal = gains.stream()
            .sorted(Comparator.reverseOrder())
            .toList();
    double idcg = dcg(ideal, k);
    return idcg == 0 ? 0 : dcg(gains, k) / idcg;
}
```

边界与坑：nDCG 的绝对值依赖相关性标注标准，跨数据集不可直接比较；只应在同一评测集内比较不同版本。

## 2.4 指标选择建议

单文档问答选 MRR，多文档综述选 Recall@K，带等级标注选 nDCG@K，噪声敏感选 Precision@K。

生产建议：同时上报 Recall@K 与 Precision@K，观察二者的权衡曲线，再决定 K 与阈值。

# 3. 生成指标（RAGAS 体系）

结论先行：生成质量用 RAGAS（Retrieval Augmented Generation Assessment，检索增强生成评估）四指标刻画：忠实度、答案相关性、上下文精确率、上下文召回率。

| 指标 | 含义 | 计算方式 |
|------|------|----------|
| Faithfulness | 回答是否完全基于上下文 | LLM 判断 |
| Answer Relevance | 回答是否切题 | LLM 判断 |
| Context Precision | 上下文中相关内容占比 | LLM 判断 |
| Context Recall | 回答所需信息是否都在上下文 | LLM 判断 |

```java
public record RagScores(
        double faithfulness,      // 忠实度
        double answerRelevance,   // 答案相关性
        double contextPrecision,  // 上下文精确率
        double contextRecall      // 上下文召回率
) {}
```

## 3.1 忠实度计算

### 3.1.1 论断拆解与判定

机制：先把答案拆成若干独立论断（claim），再逐条判断能否从上下文找到依据，最后取支持比例。

```java
interface FaithfulnessJudge {
    @UserMessage("""
            把模型回答拆解为若干独立论断，判断每个论断是否能从参考资料中找到依据。

            参考资料：
            {{context}}

            模型回答：
            {{answer}}

            输出 JSON：
            {"claims":[{"text":"","supported":true}]}
            """)
    ClaimsResult decompose(@V("context") String ctx, @V("answer") String ans);
}

public double faithfulness(String context, String answer) {
    ClaimsResult result = judge.decompose(context, answer);
    long supported = result.claims().stream()
            .filter(Claim::supported).count();
    return supported / (double) result.claims().size();
}
```

边界与坑：论断数为 0 时应返回 1.0（空答案不产生幻觉）还是拒绝评测，需明确；建议返回 1.0 但在报告中标记"空答案"。

## 3.2 答案相关性

机制：让判官仅看"问题"与"答案"，判断答案是否直接回答了问题，不看上下文。

```java
public double answerRelevance(String question, String answer) {
    String verdict = judge.judgeRelevance(question, answer);
    return "YES".equalsIgnoreCase(verdict.trim()) ? 1.0 : 0.0;
}
```

| 答案形态 | 相关性 | 说明 |
|----------|:------:|------|
| 直接命中 | 高 | 正面回答问题 |
| 部分相关 | 中 | 只答了一半 |
| 跑题 | 低 | 答非所问 |
| 拒答 | 视情况 | 无答案题拒答应记满分 |

## 3.3 上下文精确率与召回率

机制：上下文精确率衡量检索到的片段里有多少与答案相关；上下文召回率衡量回答所需信息有多少出现在上下文中。

```
Context Precision = 相关片段数 / 检索片段总数
Context Recall    = 被上下文覆盖的答案句数 / 答案句总数
```

```java
public double contextRecall(String context, String groundTruth) {
    List<String> sentences = splitSentences(groundTruth);
    long covered = sentences.stream()
            .filter(s -> judge.isSupported(context, s))
            .count();
    return covered / (double) sentences.size();
}
```

边界与坑：Judge 判断有成本与噪声，逐句调用会放大延迟；建议批量传入一次判定，并抽样人工校验。

生产建议：忠实度是最重要的红线指标（低于阈值即视为不可上线），相关性次之，上下文指标用于指导检索优化。

# 4. 端到端指标

结论先行：端到端指标反映用户真实体验，包含正确率、拒答准确率、引用准确率与幻觉率。

| 指标 | 说明 |
|------|------|
| 答案正确率 | 与标准答案一致 |
| 拒答准确率 | 无答案时正确拒答 |
| 引用准确率 | 引用来源真实存在 |
| 幻觉率 | 编造内容比例 |

## 4.1 正确率与拒答

机制：对正样本比对标准答案，对负样本检查是否拒答。

```java
public EndToEndReport evaluateEndToEnd(List<EvalCase> cases) {
    int correct = 0, hallucinated = 0, correctRefusal = 0;

    for (EvalCase c : cases) {
        RAGResult result = rag.ask(c.question());
        String answer = result.answer();

        if (c.groundTruth() == null) {
            // 无答案问题：应拒答
            if (isRefusal(answer)) correctRefusal++;
        } else {
            if (matches(answer, c.groundTruth())) correct++;
            if (containsHallucination(answer, result.context())) hallucinated++;
        }
    }
    return new EndToEndReport(
            correct / (double) cases.size(),
            correctRefusal,
            hallucinated / (double) cases.size());
}
```

边界与坑：字符串精确匹配对措辞敏感，建议用"要点覆盖"或 LLM 判定替代；`matches` 应对大小写、数字格式做归一化。

## 4.2 引用与幻觉

机制：引用准确率检查答案中每个 `[n]` 是否指向真实存在的来源；幻觉率可用忠实度的补数近似。

| 检查项 | 判定 | 处置 |
|--------|------|------|
| 引用编号越界 | 不存在来源 | 计为错误 |
| 引用与内容不符 | 张冠李戴 | 计为错误 |
| 无引用却有结论 | 疑似幻觉 | 交判官复核 |

生产建议：引用准确率与幻觉率应作为上线门禁（gate），任一超标即回滚。

# 5. 分块策略评测

结论先行：分块（chunking）直接决定召回上限，必须通过对比实验选择，而不是拍脑袋定参数。

```
方法：准备问答对，标注"答案所在块"
对比不同分块策略下"答案块是否被召回"
```

## 5.1 分块策略与参数

| 策略 | 切分依据 | 适用 |
|------|----------|------|
| 固定长度 | 字符/token 数 | 结构松散文本 |
| 递归切分 | 段落→句子层级 | 通用文档 |
| 语义切分 | 语义相似度断点 | 长文与对话 |
| 结构化切分 | 标题/Markdown | 手册与规范 |

## 5.2 对比实验

### 5.2.1 实验代码

```java
public void compareChunkingStrategies(List<EvalCase> cases) {
    List<ChunkingStrategy> strategies = List.of(
            ChunkingStrategy.fixed(500, 50),
            ChunkingStrategy.fixed(1000, 100),
            ChunkingStrategy.recursive(800, 100),
            ChunkingStrategy.semantic());

    for (ChunkingStrategy strategy : strategies) {
        double recall = evaluateWithStrategy(cases, strategy);
        log.info("策略 {}: Recall@5 = {}", strategy.name(), recall);
    }
}
```

| 策略 | 块大小 | 重叠 | Recall@5（示例） |
|------|:---:|:---:|:---:|
| fixed | 500 | 50 | 0.71 |
| fixed | 1000 | 100 | 0.78 |
| recursive | 800 | 100 | 0.84 |
| semantic | 自适应 | 自适应 | 0.86 |

边界与坑：块太小会丢上下文，块太大会稀释相关信号并推高成本；重叠过大导致重复内容占据上下文。

生产建议：分块变更需重新评测并重建索引，二者必须同步，否则指标失真。

# 6. 检索参数评测

结论先行：topK 与相似度阈值是检索质量的两个旋钮，应通过网格搜索（grid search）在评测集上寻优。

## 6.1 topK 与阈值

- topK 越大，召回越高、噪声越多、延迟越高。
- 阈值越高，噪声越少、可能漏召回。

## 6.2 网格搜索

```java
public void tuneRetrieval(List<EvalCase> cases) {
    for (int topK : new int[]{5, 10, 20, 50}) {
        for (double threshold : new double[]{0.5, 0.6, 0.7, 0.8}) {
            double recall = evaluate(cases, topK, threshold);
            log.info("topK={} threshold={} recall={}", topK, threshold, recall);
        }
    }
}
```

边界与坑：在评测集上过拟合参数会损害线上泛化，建议留出独立验证集，并优先选择"平台区"参数而非峰值。

生产建议：把参数写入配置中心，支持灰度切换与回滚。

# 7. 混合检索与重排评测

结论先行：稠密检索、稀疏检索（BM25）、混合与重排各有所长，应逐一评测再决定组合。

## 7.1 检索策略对比

```java
public void compareRetrievalStrategies(List<EvalCase> cases) {
    Map<String, Double> results = new LinkedHashMap<>();

    results.put("dense",      evalRecall(cases, denseRetriever));
    results.put("bm25",       evalRecall(cases, bm25Retriever));
    results.put("hybrid",     evalRecall(cases, hybridRetriever));
    results.put("hybrid+rerank", evalRecall(cases, rerankedRetriever));

    results.forEach((name, recall) ->
            log.info("{}: Recall@5 = {}", name, recall));
}
```

经验提升（因数据而异）：

| 方案 | 相对提升 |
|------|:--------:|
| 纯稠密 | 基线 |
| 混合 | +10%~25% |
| 混合 + 重排 | +15%~35% |

## 7.2 融合与重排机制

### 7.2.1 融合与重排

混合检索常用 RRF（Reciprocal Rank Fusion，倒数排名融合）：

```
RRF(d) = Σ_r 1 / (k + rank_r(d))     k 常取 60
```

重排（rerank）则用交叉编码器（cross-encoder）对候选重新打分，精度高但延迟大。

边界与坑：重排只对已召回的候选有效，若正确文档不在候选集内，重排无法补救；因此重排前必须保证召回充足。

生产建议：先做混合召回（召回优先），再用重排精排（精度优先），并用延迟预算约束重排候选数。

# 8. 评测报告

结论先行：评测结果应固化为结构化报告，包含检索、生成、端到端、性能与成本五类字段，便于版本对比。

## 8.1 报告字段

```java
public record RagEvalReport(
        double recallAt5,
        double mrr,
        double faithfulness,
        double answerRelevance,
        double contextPrecision,
        double contextRecall,
        double endToEndAccuracy,
        long avgLatencyMs,
        BigDecimal avgCost
) {
    public void print() {
        System.out.printf("Recall@5:         %.3f%n", recallAt5);
        System.out.printf("MRR:              %.3f%n", mrr);
        System.out.printf("Faithfulness:     %.3f%n", faithfulness);
        System.out.printf("AnswerRelevance:  %.3f%n", answerRelevance);
        System.out.printf("ContextPrecision: %.3f%n", contextPrecision);
        System.out.printf("ContextRecall:    %.3f%n", contextRecall);
        System.out.printf("EndToEnd:         %.3f%n", endToEndAccuracy);
        System.out.printf("Latency:          %d ms%n", avgLatencyMs);
        System.out.printf("Cost:             $%.4f%n", avgCost);
    }
}
```

## 8.2 版本对比

| 版本 | Recall@5 | Faithfulness | 延迟 | 成本 | 结论 |
|------|:---:|:---:|:---:|:---:|------|
| v1 基线 | 0.78 | 0.85 | 1.2s | $0.006 | 基线 |
| v2 混合检索 | 0.86 | 0.86 | 1.6s | $0.007 | 召回提升 |
| v3 加重排 | 0.90 | 0.91 | 2.1s | $0.010 | 精度提升、成本上升 |

边界与坑：只看单次结果会受随机性干扰，应多次运行取均值与方差。

生产建议：在 CI 中加入评测门禁，指标回退超过阈值即阻断合并。

# 9. 诊断矩阵

结论先行：把"症状"映射到"检索指标 × 生成指标"的组合，即可快速定位故障层。

| 症状 | 检索指标 | 生成指标 | 结论 |
|------|:--------:|:--------:|------|
| 答不出 | 低召回 | — | 检索问题 |
| 答错 | 高召回 | 低忠实度 | 生成问题 |
| 答偏 | 高召回 | 低相关性 | 提示/上下文 |
| 噪声多 | 低精确率 | — | 阈值/重排 |
| 编造 | 高召回 | 低忠实度 | 提示未约束 |

## 9.1 症状到根因

### 9.1.1 症状映射规则

```
答不出 ──► 查 Recall@K 低 ──► 分块/检索参数/索引
答错 ────► 查 Faithfulness 低 ──► 提示约束/上下文顺序
答偏 ────► 查 Relevance 低 ──► 提示词/问题改写
噪声多 ──► 查 Precision 低 ──► 阈值/重排
```

## 9.2 处置顺序

先修检索（召回优先），再修生成（忠实优先）。检索不到位时调生成提示收益有限。

生产建议：把诊断矩阵做成看板，指标异常自动高亮对应处置项。

# 10. 生产注意事项

结论先行：RAG 评测的常见风险集中在"只测端到端、缺标注、Judge 未校准、缺基线与成本度量"。

## 10.1 评测集与标注

| 风险 | 建议 |
|------|------|
| 只测端到端 | 分层评测 |
| 无 relevantDocIds | 无法测检索 |
| Judge 未校准 | 人工校准 |
| 只测一种策略 | 对比多种 |

## 10.2 基线与成本

| 风险 | 建议 |
|------|------|
| 无版本对比 | 保留基线 |
| 忽略延迟成本 | 一并度量 |
| 评测集泄露 | 训练/评测隔离 |
| 指标漂移 | 定期重跑基线 |

边界与坑：LLM 判官自身会随模型升级而变化，跨时间比较需固定判官版本。

生产建议：建立"黄金集"（golden set）小样本用于每日快速回归，全量集用于发布前评估。

# 11. 小结

## 11.1 核心要点

- RAG 评测分检索与生成两层。
- 检索指标：Recall@K、MRR、nDCG。
- 生成指标：忠实度、相关性、上下文精确/召回。

## 11.2 落地清单

- 分块与检索参数应通过对比实验选择。
- 用诊断矩阵定位"检索失败"还是"生成失败"。
- 报告应包含性能与成本，并保留基线做版本对比。
