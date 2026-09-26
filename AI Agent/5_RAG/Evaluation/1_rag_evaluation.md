# 1. 为什么 RAG 必须评测

> "感觉回答更好了" ≠ "确实更好了"

RAG（Retrieval-Augmented Generation，检索增强生成）链路长、变量多，任何改动都可能"改好一处、改坏另一处"。**没有评测的优化都是盲调。**

## 1.1 感觉优化与真实优化的差距

人在读几条样例时会天然关注"答得漂亮"的那一条，忽略"悄悄变差"的十条。这种直觉偏差在 RAG 里尤其危险，因为检索质量的变化常常是隐性的：把块切小一点，某些问题的召回立刻变好，另一些问题的答案却被切断。

- 主观感受：抽样 5 条，觉得"挺像样"。
- 客观评测：同一批 200 条，Recall@5 从 0.86 掉到 0.79。
- 差距来源：样本偏差、近因效应、对失败样例的天然遗忘。

只有把判断交给固定评测集与固定指标，才能把"感觉"翻译成"数字"。

## 1.2 链路长带来的变量爆炸

一条典型的 RAG 链路可以拆成离线索引与在线检索生成两段：

```
离线索引                          在线服务
─────────────                     ─────────────
文档采集 ─► 解析 ─► 分块          问题 ─► 改写 ─► 检索 ─► 重排
   │         │       │                              │
   ▼         ▼       ▼                              ▼
 元数据    清洗    向量化 ─► 入库              上下文拼装 ─► 生成
```

上图里每一个箭头都是一个可调参数，参数组合的数量是乘法级增长的。

### 1.2.1 检索侧变量

| 环节 | 典型变量 | 影响 |
|------|----------|------|
| 解析 | 解析器、OCR 开关、表格策略 | 内容是否完整可读 |
| 分块 | 块大小、重叠、边界规则 | 召回上限 |
| 向量化 | Embedding 模型、维度、归一化 | 语义匹配能力 |
| 索引 | 距离度量、HNSW 参数、分区 | 召回率与延迟 |
| 检索 | topK、阈值、过滤条件 | 候选质量 |
| 重排 | Rerank 模型、截断数量 | 精度上限 |

### 1.2.2 生成侧变量

| 环节 | 典型变量 | 影响 |
|------|----------|------|
| 提示 | 系统提示、约束条款、示例 | 忠实度 |
| 上下文 | 片段数量、顺序、长度 | 是否遗漏 |
| 模型 | 模型版本、温度、最大 token | 稳定性 |
| 输出 | 结构化格式、引用编号 | 可校验性 |

变量一多，"改一个参数全局观察"就退化为随机游走。评测是把随机游走变成可收敛优化的唯一手段。

## 1.3 没有评测的优化都是盲调

### 1.3.1 盲调的典型表现

- 只凭个例修改提示，改完不做回归。
- 换了 Embedding 模型却不知道召回是升是降。
- 上线后收到零星投诉，无法判断是偶发还是系统性退化。

### 1.3.2 评测作为工程护栏

评测集加阈值断言，等于给 RAG 装了一道"回归护栏"：任何 Prompt、分块、模型的变更，只要越过阈值就会被 CI 拦下。这道护栏让团队敢于持续迭代，而不是每次改动都提心吊胆。

# 2. 评测的两个层次

```
① 检索评测：能不能检索到正确答案所在的文档
② 生成评测：基于检索内容，回答是否忠实、相关、正确
```

| 层次 | 关注 | 失败表现 |
|------|------|----------|
| 检索 | 召回率、精确率 | 答案没被召回 |
| 生成 | 忠实度、相关性 | 有资料但答错/编造 |

## 2.1 检索评测

### 2.1.1 检索评测关注什么

检索评测回答一个问题：**正确答案所在的那块资料，有没有出现在候选列表里？** 它只看文档与排序，不看模型怎么措辞。核心是"覆盖"与"排序"两件事：覆盖看有没有召回，排序看排得靠不靠前。

### 2.1.2 检索失败的表现

- 相关文档完全没进候选 → 召回失败。
- 相关文档进了候选但排在 30 名开外 → 排序失败。
- 候选里塞了大量无关块 → 精确率低，浪费上下文预算。

## 2.2 生成评测

### 2.2.1 生成评测关注什么

生成评测在"给定正确上下文"的前提下，判断回答是否忠实、切题、正确。它隔离了检索变量，能单独衡量提示与模型的能力。

### 2.2.2 生成失败的表现

- 上下文里有答案，模型却编造了别的数字 → 忠实度低。
- 答得流畅但偏离问题 → 相关性低。
- 只用了上下文的一部分，漏掉关键约束 → 覆盖率低。

## 2.3 两层的组合与边界

### 2.3.1 层间依赖

两层不是并列关系，而是串联关系：

```
检索层 ──► 生成层 ──► 端到端
  │            │
  └─ 决定上限  └─ 决定实现程度
```

检索没召回，生成再强也无从下手；检索召回正确，生成仍可能答错。因此诊断时必须先分层，再归因。

### 2.3.2 评测的边界与反例

- **边界一**：检索评测依赖 `relevantDocIds` 标注，标注不全时会低估召回。
- **边界二**：生成评测依赖 Judge，Judge 本身有偏差，需校准。
- **反例**：某次改动让 Recall 上升但忠实度下降，端到端持平——只看单一层次会误判为"没变化"。

# 3. 核心指标

## 3.1 检索指标

### 3.1.1 指标定义与公式

| 指标 | 定义 | 公式 |
|------|------|------|
| Recall@K | 正确文档是否在前 K 个结果中 | 命中查询数 / 查询总数 |
| Precision@K | 前 K 个结果中相关文档占比 | 相关数 ∩ K / K |
| MRR | 第一个相关结果排名的倒数均值 | mean(1 / rank) |
| nDCG@K | 考虑排序位置的加权指标 | DCG@K / IDCG@K |

其中 DCG 的常见形式为：

```
DCG@K  = Σ (2^rel_i - 1) / log2(i + 1)
IDCG@K = 理想排序下的 DCG@K
nDCG@K = DCG@K / IDCG@K
```

### 3.1.2 Java 实现

```java
public record EvalCase(
        String id,
        String question,
        String groundTruth,
        List<String> relevantDocIds,
        List<String> mustContain,
        List<String> mustNotContain
) {}

public double recallAtK(List<EvalCase> cases, int k) {
    int hit = 0;
    for (EvalCase c : cases) {
        List<String> retrieved = retrieve(c.question(), k);
        if (!Collections.disjoint(retrieved, c.relevantDocIds())) {
            hit++;
        }
    }
    return hit / (double) cases.size();
}

public double mrr(List<EvalCase> cases) {
    double sum = 0;
    for (EvalCase c : cases) {
        List<String> retrieved = retrieve(c.question(), 10);
        for (int i = 0; i < retrieved.size(); i++) {
            if (c.relevantDocIds().contains(retrieved.get(i))) {
                sum += 1.0 / (i + 1);
                break;
            }
        }
    }
    return sum / cases.size();
}
```

### 3.1.3 边界与陷阱

- 一个查询可能有多个相关文档，Recall@K 应做"任一命中"而非"全部命中"。
- topK 越大 Recall 越高但 Precision 越低，两者必须一起看。
- nDCG 需要分级相关性（rel 可为 2、1、0），二值标注时应退化用 MRR。

## 3.2 生成指标（RAGAS 体系）

### 3.2.1 指标含义

| 指标 | 含义 | 方向 |
|------|------|------|
| Faithfulness（忠实度） | 回答是否完全基于检索内容 | 越高越好 |
| Answer Relevance（答案相关性） | 回答是否切题 | 越高越好 |
| Context Precision（上下文精确度） | 检索上下文中相关内容的占比 | 越高越好 |
| Context Recall（上下文召回） | 回答所需信息是否都在上下文中 | 越高越好 |

### 3.2.2 公式拆解

```
Faithfulness      = 可被上下文支撑的断言数 / 回答总断言数
Context Precision = 相关片段数 / 上下文片段总数（可加位置权重）
Context Recall    = 被上下文覆盖的标准答案信息点 / 全部信息点
Answer Relevance  = 由回答反推的问题 与 原问题 的语义相似度
```

### 3.2.3 Java 实现

```java
public record RagScores(
        double faithfulness,
        double answerRelevance,
        double contextPrecision,
        double contextRecall
) {}
```

## 3.3 端到端指标

### 3.3.1 指标说明

| 指标 | 说明 |
|------|------|
| 答案正确率 | 与标准答案一致的比例 |
| 拒答率 | 无资料时正确拒答的比例 |
| 引用准确率 | 引用来源是否真实存在 |

### 3.3.2 组合指标

端到端指标反映用户真实体感，但粒度粗、难定位。推荐"端到端看趋势，分层指标定归因"：先看端到端是否退化，再看分层检索与生成指标谁在下降。

# 4. 评测集构建

```json
[
  {
    "id": "q1",
    "question": "差旅费报销时限是多久？",
    "groundTruth": "费用发生后 30 天内",
    "relevantDocIds": ["doc-handbook-3-2"],
    "mustContain": ["30 天"],
    "mustNotContain": ["我不知道", "无法确定"]
  }
]
```

| 字段 | 用途 |
|------|------|
| `question` | 输入 |
| `groundTruth` | 标准答案 |
| `relevantDocIds` | 用于检索评测 |
| `mustContain` | 关键信息校验 |
| `mustNotContain` | 拒答/幻觉校验 |

## 4.1 评测集结构

### 4.1.1 字段设计

- `id`：稳定主键，跨版本追踪同一条用例。
- `relevantDocIds`：允许一对多，便于评估多跳问题。
- `mustContain` / `mustNotContain`：低成本、可解释的确定性校验。
- 可选 `category`：按场景分组，便于分场景统计。

### 4.1.2 Java 建模与加载

```java
public record EvalCase(
        String id,
        String question,
        String groundTruth,
        List<String> relevantDocIds,
        List<String> mustContain,
        List<String> mustNotContain
) {}

public List<EvalCase> loadCases(String path) {
    return objectMapper.readValue(
            new File(path),
            new TypeReference<List<EvalCase>>() {});
}
```

## 4.2 评测集来源

### 4.2.1 来源对比

| 来源 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| 真实用户问题 | 最贴近实际 | 真实分布 | 需脱敏、噪声大 |
| 业务专家编写 | 质量高、覆盖全 | 权威 | 成本高 |
| LLM 生成 | 成本低、规模大 | 快 | 需人工抽检 |
| 失败案例回流 | 针对性最强 | 直击痛点 | 数量有限 |

### 4.2.2 规模与配比

> 建议规模：**50~200 条**起步，覆盖主要场景与边界。

推荐配比：正常问题 60%、边界问题 25%、无答案问题 15%。无答案样本用来衡量拒答能力，最容易被忽略。

# 5. LLM-as-Judge 评测

## 5.1 Judge 的基本形态

### 5.1.1 Prompt 设计

Judge 的本质是"让模型按 rubric（评分细则）打分"。关键是把评分维度拆清、给出 0-1 的锚点、要求输出结构化 JSON。

```java
interface Judge {
    @UserMessage("""
            评估回答质量，从以下维度打分（0-1）并说明理由：
            1. faithfulness：回答是否完全基于【参考资料】
            2. relevance：回答是否切题

            问题：{{question}}
            参考资料：{{context}}
            回答：{{answer}}

            输出 JSON：{"faithfulness":0.0,"relevance":0.0,"reason":""}
            """)
    JudgeResult evaluate(@V("question") String q,
                         @V("context") String ctx,
                         @V("answer") String ans);
}

public record JudgeResult(
        double faithfulness,
        double relevance,
        String reason
) {}
```

### 5.1.2 参考实现（Spring AI 风格）

```java
@Component
public class SpringAiJudge implements Judge {

    private final ChatClient chatClient;

    public SpringAiJudge(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @Override
    public JudgeResult evaluate(String q, String ctx, String ans) {
        String raw = chatClient.prompt()
                .user(u -> u.text("问题：{q}\n资料：{ctx}\n回答：{ans}\n只输出 JSON。")
                        .param("q", q).param("ctx", ctx).param("ans", ans))
                .call().content();
        return objectMapper.readValue(raw, JudgeResult.class);
    }
}
```

## 5.2 偏差与校准

### 5.2.1 常见偏差

| 偏差 | 表现 | 缓解 |
|------|------|------|
| 位置偏差 | 偏爱靠前/靠后的候选 | 交换顺序取平均 |
| 自偏好 | 偏爱同族模型输出 | 换模型或交叉评审 |
| 长度偏差 | 偏爱长回答 | 长度归一化、明确 rubric |
| 宽松偏差 | 普遍给高分 | 提供评分锚点样例 |

### 5.2.2 校准方法

- 用人工标注的小样本（如 30 条）与 Judge 分数做相关性校验。
- 要求 Judge 先给理由再给分，提升一致性。
- 多次采样取均值，降低单次波动。

> LLM-as-Judge 便宜高效，但有位置偏差与自偏好，需校准。详见 `12_Evaluation/3_llm_as_judge.md`。

# 6. 自动化评测流程

## 6.1 单次评测流程

### 6.1.1 流程拆解

```
读取评测集
   │
   ▼
逐条：检索 ─► 生成 ─► 评审
   │
   ▼
聚合指标 ─► 生成报告 ─► 阈值断言
```

### 6.1.2 Java 实现

```java
public RagReport evaluate(List<EvalCase> cases) {
    List<Double> faithfulness = new ArrayList<>();
    List<Double> recall = new ArrayList<>();

    for (EvalCase c : cases) {
        // 检索
        List<Document> docs = retriever.retrieve(c.question());
        recall.add(recallAt(docs, c.relevantDocIds()));

        // 生成
        String answer = generate(c.question(), docs);

        // 评审
        JudgeResult judge = judge.evaluate(
                c.question(), joinText(docs), answer);
        faithfulness.add(judge.faithfulness());
    }
    return new RagReport(avg(recall), avg(faithfulness));
}
```

## 6.2 报告与可视化

### 6.2.1 报告结构

| 区块 | 内容 |
|------|------|
| 概览 | 总分、样本数、运行耗时 |
| 分层指标 | 检索指标、生成指标 |
| 分场景 | 按 category 分组统计 |
| 失败清单 | 低于阈值的用例明细 |

### 6.2.2 结果解读

报告不是"看一个总分"，而是"找下降的维度"。建议每次运行保存历史快照，做同比与环比。

# 7. 回归测试

## 7.1 阈值断言

### 7.1.1 断言示例

```java
@Test
void rag_regression() {
    RagReport report = evaluator.evaluate(loadCases("eval/rag-cases.json"));

    assertTrue(report.recallAt5() >= 0.85,
            "召回率下降: " + report.recallAt5());
    assertTrue(report.faithfulness() >= 0.90,
            "忠实度下降: " + report.faithfulness());
}
```

### 7.1.2 阈值设定

阈值不能拍脑袋，应基于基线运行结果的分布设定，留出合理波动空间：

| 指标 | 基线 | 告警阈值 | 阻断阈值 |
|------|:----:|:--------:|:--------:|
| Recall@5 | 0.88 | 0.85 | 0.80 |
| Faithfulness | 0.93 | 0.90 | 0.85 |
| 拒答率 | 0.90 | 0.85 | 0.80 |

## 7.2 CI 集成

### 7.2.1 流水线设计

```
提交代码
   │
   ▼
构建 ─► 单元测试 ─► RAG 评测 ─► 阈值断言
                          │
                          ├─ 通过 ─► 合并
                          └─ 失败 ─► 阻断并出报告
```

### 7.2.2 失败处理

> 接入 CI，任何 Prompt/分块/模型变更都必须通过阈值。

- 评测用例与代码同仓、同版本化。
- 失败时上传报告制品，便于回看是哪几条退化。
- 允许标注"已知抖动"用例，避免噪声频繁阻断。

# 8. 指标诊断表

## 8.1 症状到结论

### 8.1.1 诊断矩阵

| 症状 | 检索指标 | 生成指标 | 结论 |
|------|:--------:|:--------:|------|
| 答不出 | 低召回 | — | 检索问题 |
| 答错 | 高召回 | 低忠实度 | 生成问题 |
| 答偏 | 高召回 | 低相关性 | 提示/上下文问题 |
| 噪声多 | 低精确率 | — | 阈值/重排问题 |
| 编造 | 高召回 | 低忠实度 | 提示未约束 |

### 8.1.2 使用方式

先用端到端指标发现"有退化"，再用本表定位到"哪一层"，最后用失败分析细化到具体子类（见 `2_failure_analysis.md`）。

## 8.2 诊断的反例

### 8.2.1 误判场景

- 高召回但低忠实度：可能不是提示问题，而是上下文过长导致关键句被淹没。
- 低精确率但端到端正常：可能重排把噪声压到了末尾，无需处理。

### 8.2.2 交叉验证

任何诊断结论都应回到原始样例人工确认，避免只看聚合数字下结论。

# 9. 生产注意事项

## 9.1 评测集治理

### 9.1.1 过拟合与版本

| 风险 | 建议 |
|------|------|
| 评测集过拟合 | 保留未参与调优的测试集 |
| 无版本记录 | 评测集与代码一起版本化 |

### 9.1.2 样本规模

| 风险 | 建议 |
|------|------|
| 样本太少 | 50 条起步 |
| 只测 Happy Path | 覆盖边界、无答案、多意图 |

## 9.2 评测运行治理

### 9.2.1 稳定性与成本

| 风险 | 建议 |
|------|------|
| 评测不稳定 | 多次采样取均值 |
| 评测成本高 | 抽样 + 小模型 Judge |

### 9.2.2 覆盖度

- 定期用真实流量采样补充评测集。
- 把线上失败案例回流进评测集，形成闭环。

# 10. 小结

- RAG 评测分检索与生成两层。
- 核心指标：Recall@K、MRR、忠实度、相关性、上下文精确/召回。
- 评测集建议 50~200 条，覆盖主场景与边界。
- LLM-as-Judge 是自动化评测主力，但要防偏差。
- 评测必须接入 CI，防止退化。
