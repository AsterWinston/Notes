# 1. 评测框架概览

评测框架（Evaluation Framework）把「数据集 + 执行器 + 指标 + 报告」串成流水线，让评测可重复、可对比、可自动化。

## 1.1 主流框架一览

| 框架 | 语言 | 定位 | 特点 |
|------|------|------|------|
| RAGAS | Python | RAG 评测 | RAG 评测事实标准，指标齐全 |
| DeepEval | Python | 单元测试式评测 | 断言风格，易接入 pytest |
| promptfoo | Node | Prompt 对比 | 声明式配置，CI 友好 |
| Langfuse | 多语言 | 平台 + 评测 | 数据集、Trace、人工标注一体 |
| OpenAI Evals | Python | 通用评测框架 | 官方出品，注册式评测 |
| TruLens | Python | RAG + Agent 评测 | 反馈函数（Feedback Function） |

> 多数评测框架是 Python 生态，Java 侧可**通过 API 集成或自建**。

## 1.2 Java 接入路径

| 路径 | 做法 | 优点 | 代价 |
|------|------|------|------|
| 纯 Python | 用 Python 跑评测 | 生态最全 | 数据要导出，链路割裂 |
| API 集成 | Java 调平台 API 上报/取数 | 侵入小 | 依赖平台能力 |
| 自建 | Java 实现指标与编排 | 与业务同栈 | 需自己维护 |

- 团队以 Python 为主：优先 RAGAS + promptfoo，Java 只负责导出样本。
- 已有可观测平台：优先 Langfuse，Java 侧只做上报。
- 纯 Java 团队、指标需贴业务：自建轻量框架，复刻 RAGAS 逻辑。

## 1.3 通用分层

```
+-----------+     +-----------+     +-----------+
|  Dataset  | --> |  Runner   | --> |  Metric   |
+-----------+     +-----------+     +-----------+
                        |                 ^
                        v                 |
                  +-----------+     +-----------+
                  |  Report   | <-- |   Judge   |
                  +-----------+     +-----------+
```

| 层 | 职责 | 关键问题 |
|----|------|----------|
| Dataset | 提供样本与标准答案 | 覆盖是否充分 |
| Runner | 驱动被测系统 | 是否可并发、可缓存 |
| Metric | 计算分数 | 公式是否稳定 |
| Judge | 提供 LLM 评审 | 是否校准过 |

> 四层解耦后，换模型、换 Judge、换指标互不影响，这是自建框架的核心收益。

# 2. RAGAS 指标

RAGAS（Retrieval Augmented Generation Assessment）定义了 RAG 评测的标准指标集，是跨团队对齐口径的事实标准。

## 2.1 核心指标

| 指标 | 说明 | 关注对象 |
|------|------|----------|
| Faithfulness | 忠实度 | 答案 vs 上下文 |
| Answer Relevancy | 答案相关性 | 答案 vs 问题 |
| Context Precision | 上下文精确率 | 检索结果排序质量 |
| Context Recall | 上下文召回率 | 检索是否覆盖答案所需 |
| Answer Correctness | 答案正确性 | 答案 vs 标准答案 |

### 2.1.1 指标公式

```
faithfulness       = 有依据的论断数 / 论断总数
answer_relevancy   = mean(cos_sim(生成问题_i, 原问题))
context_precision  = Σ (precision@k × rel_k) / 相关文档总数
context_recall     = 检索到的相关句数 / 标准答案相关句总数
answer_correctness = w1 × 事实一致 + w2 × 语义相似
```

## 2.2 使用示例

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy

result = evaluate(dataset, metrics=[faithfulness, answer_relevancy])
```

Java 侧可复刻其计算逻辑（用 LLM-as-Judge）。

## 2.3 边界与失效场景

| 场景 | 失效原因 | 应对 |
|------|----------|------|
| 无标准答案 | recall/correctness 无法算 | 只用 faithfulness |
| 上下文为空 | faithfulness 无依据可查 | 单独统计检索失败率 |
| 答案过长 | 论断多，忠实度被稀释 | 先做答案压缩 |
| 多语言混排 | 语义相似度失真 | 用同语言 Judge |
| 领域专有名词 | 通用模型误判 | 领域微调 Judge |

> 指标只是代理（Proxy），不是目标本身；先看失败样本，再看总分。

# 3. Java 自建评测框架

自建框架的核心是三个抽象：`Metric`（怎么算）、`EvalSample`（算谁）、`RagEvaluator`（怎么编排）。

## 3.1 领域模型

### 3.1.1 Metric 接口

```java
public interface Metric {

    String name();

    MetricResult compute(EvalSample sample);

    default boolean requiresGroundTruth() {
        return false;
    }
}
```

### 3.1.2 EvalSample 与 MetricResult

```java
public record EvalSample(
        String id, String question, String answer, String groundTruth,
        List<String> contexts, List<String> retrievedDocIds,
        List<String> relevantDocIds, long latencyMs, BigDecimal cost) {

    public boolean hasGroundTruth() {
        return groundTruth != null && !groundTruth.isBlank();
    }
}

public record MetricResult(String metricName, double score, String reason) {
    public static MetricResult of(String name, double score) {
        return new MetricResult(name, score, "");
    }
}
```

> 用 record 承载样本与结果，天然不可变、易序列化，适合落盘做基线。

## 3.2 编排器

```java
public class RagEvaluator {

    private final List<Metric> metrics;

    public RagEvaluator(JudgeClient judge) {
        this.metrics = List.of(
                new FaithfulnessMetric(judge),
                new AnswerRelevancyMetric(judge),
                new ContextPrecisionMetric(judge),
                new ContextRecallMetric(judge),
                new RecallAtKMetric(5));
    }

    public EvalReport evaluate(List<EvalSample> samples) {
        Map<String, List<MetricResult>> details = new LinkedHashMap<>();
        Map<String, List<Double>> scores = new HashMap<>();
        for (EvalSample sample : samples) {
            for (Metric metric : metrics) {
                if (metric.requiresGroundTruth() && !sample.hasGroundTruth()) {
                    continue;
                }
                MetricResult result = metric.compute(sample);
                scores.computeIfAbsent(metric.name(), k -> new ArrayList<>())
                        .add(result.score());
                details.computeIfAbsent(sample.id(), k -> new ArrayList<>())
                        .add(result);
            }
        }
        return EvalReport.from(samples.size(), scores, details);
    }
}
```

## 3.3 并发与缓存

LLM Judge 是 IO 密集且昂贵，串行跑几百条样本会很慢，需要并发与缓存。

| 手段 | 目的 | 注意 |
|------|------|------|
| 并发池 | 提升吞吐 | 限流防触发配额 |
| 重试退避 | 容忍 429/超时 | 幂等，避免重复计费 |
| 结果缓存 | 相同输入复用分数 | 键含模型与提示版本 |
| 断点续跑 | 大批量任务可恢复 | 落盘中间结果 |

> 缓存键必须包含「模型 ID + 提示版本 + 样本内容」，否则会错误复用旧分数。

# 4. 指标实现示例

## 4.1 忠实度指标

忠实度（Faithfulness）衡量答案是否「有据可依」。

### 4.1.1 论断分解

```
答案: "差旅费需 30 天内报销，需发票。"
        |
        +- Claim1: 差旅费需 30 天内报销 -> 上下文有 -> 支持
        +- Claim2: 需发票              -> 上下文无 -> 不支持
faithfulness = 1 / 2 = 0.5
```

### 4.1.2 计算逻辑

```java
public class FaithfulnessMetric implements Metric {

    private final JudgeClient judge;

    public FaithfulnessMetric(JudgeClient judge) { this.judge = judge; }

    @Override
    public String name() { return "faithfulness"; }

    @Override
    public MetricResult compute(EvalSample sample) {
        String context = String.join("\n---\n", sample.contexts());
        ClaimsResult result = judge.decomposeClaims(context, sample.answer());
        if (result.claims().isEmpty()) {
            return new MetricResult(name(), 0.0, "无有效论断");
        }
        long supported = result.claims().stream()
                .filter(Claim::supported).count();
        int total = result.claims().size();
        double score = supported / (double) total;
        return new MetricResult(name(), score,
                "%d/%d 条论断有依据".formatted(supported, total));
    }
}
```

> 边界：答案为「我不知道」时无论断，应记 0 分而非 1 分，否则会奖励拒答。

## 4.2 检索指标

### 4.2.1 Recall@K

```java
public class RecallAtKMetric implements Metric {

    private final int k;

    public RecallAtKMetric(int k) { this.k = k; }

    @Override
    public String name() { return "recall@" + k; }

    @Override
    public MetricResult compute(EvalSample sample) {
        Set<String> retrieved = new HashSet<>(sample.retrievedDocIds());
        boolean hit = sample.relevantDocIds().stream()
                .anyMatch(retrieved::contains);
        return new MetricResult(name(), hit ? 1.0 : 0.0,
                hit ? "命中" : "未命中");
    }
}
```

### 4.2.2 Precision@K 与 MRR

```
precision@k = |相关 ∩ 前k| / k
mrr         = 1 / 第一个相关文档的排名
```

| 指标 | 衡量 | 偏好 |
|------|------|------|
| Recall@K | 覆盖度 | 越高越好 |
| Precision@K | 纯净度 | 越高越好 |
| MRR | 首个相关的排名 | 越高越好 |
| NDCG | 分级相关 + 位置折损 | 越高越好 |

## 4.3 相关性与正确性

```java
public MetricResult compute(EvalSample sample) {
    double sim = judge.semanticSimilarity(sample.question(), sample.answer());
    return new MetricResult("answer_relevancy", sim, "问题-答案语义相似度");
}
```

正确性（Answer Correctness）需要标准答案，通常由「事实一致 + 长度惩罚」加权：

```java
double sim = judge.semanticSimilarity(sample.groundTruth(), sample.answer());
double score = 0.7 * sim + 0.3 * lengthPenalty(sample);
```

## 4.4 指标实现反例

| 反例 | 问题 | 修正 |
|------|------|------|
| 拒答算满分 | 奖励「我不知道」 | 无论断记 0 |
| 用字数当相关性 | 长答案占便宜 | 用语义判断 |
| 只看总分 | 掩盖局部失败 | 保留每条详情 |
| 指标无版本 | 分数不可比 | 指标带版本号 |
| Judge 未固定 | 换模型分数漂移 | 锁定 Judge 版本 |

# 5. 报告输出

## 5.1 报告结构

```java
public record EvalReport(
        int total,
        Map<String, Double> metricAverages,
        Map<String, List<MetricResult>> details,
        long avgLatencyMs,
        BigDecimal avgCost
) {
    public static EvalReport from(int total,
                                  Map<String, List<Double>> scores,
                                  Map<String, List<MetricResult>> details) {
        Map<String, Double> averages = new LinkedHashMap<>();
        scores.forEach((name, values) -> averages.put(name, mean(values)));
        return new EvalReport(total, averages, details,
                (long) averageLatency(details), averageCost(details));
    }
}
```

| 字段 | 含义 | 用途 |
|------|------|------|
| total | 样本数 | 判断是否达最小样本量 |
| metricAverages | 各指标均值 | 与基线对比 |
| details | 每样本每条指标 | 失败归因 |
| avgLatencyMs | 平均延迟 | 性能门禁 |
| avgCost | 平均成本 | 成本门禁 |

## 5.2 打印与导出

### 5.2.1 print

```java
public void print() {
    System.out.println("=== 评测报告 ===");
    System.out.println("样本数: " + total);
    metricAverages.forEach((name, avg) ->
            System.out.printf("%-20s %.3f%n", name, avg));
    System.out.printf("%-20s %d ms%n", "avgLatency", avgLatencyMs);
    System.out.printf("%-20s $%.4f%n", "avgCost", avgCost);
}
```

### 5.2.2 saveJson

```java
public void saveJson(Path path) throws IOException {
    new ObjectMapper().registerModule(new JavaTimeModule())
            .writerWithDefaultPrettyPrinter()
            .writeValue(path.toFile(), this);
}
```

## 5.3 报告可比性

- 报告需带 `runId`、`timestamp`、`model`、`promptVersion`、`metricVersion`。
- 只有「同口径」的报告才能比：样本集、指标版本、Judge 版本一致。
- 对比时看「差值 + 置信区间」，而非单点数值。

# 6. 与 Langfuse 集成

Langfuse 提供数据集（Dataset）与评测（Score）能力，Java 可通过 API 上报。

## 6.1 上报机制

- Trace：一次请求的完整链路。
- Score：挂到 trace 上的评分，可来自人工或模型。
- Dataset：可版本化的样本集合，支持回归对比。

## 6.2 上报实现

```java
public void report(EvalReport report) {
    for (var entry : report.details().entrySet()) {
        for (MetricResult result : entry.getValue()) {
            client.score(ScoreRequest.builder()
                    .traceId(entry.getKey())
                    .name(result.metricName())
                    .value(result.score())
                    .comment(result.reason())
                    .build());
        }
    }
}
```

## 6.3 集成注意事项

| 事项 | 说明 |
|------|------|
| 幂等 | 同一 runId 重复上报要覆盖而非累加 |
| 批量 | 合并请求，减少网络往返 |
| 限流 | 遵守平台速率限制，失败重试 |
| 隐私 | 上报前脱敏，避免泄露用户数据 |
| 失败降级 | 上报失败不阻断主评测流程 |

> 平台是「数据存放与展示」层，指标计算仍应在可控代码中完成。

# 7. CI 集成脚本

## 7.1 Runner 主流程

```java
public static void main(String[] args) {
    List<EvalCase> cases = EvalCaseLoader.load("eval/cases.json");
    EvalReport report = buildEvaluator().evaluate(toSamples(cases));
    report.print();

    EvalReport baseline = EvalReport.load("eval/baseline.json");
    List<String> failures = compare(report, baseline);
    if (!failures.isEmpty()) {
        failures.forEach(System.err::println);
        System.exit(1);
    }
    System.out.println("评测通过");
}
```

## 7.2 基线与门禁

### 7.2.1 对比逻辑

```java
static List<String> compare(EvalReport now, EvalReport base) {
    List<String> failures = new ArrayList<>();
    double tolerance = 0.02;
    now.metricAverages().forEach((name, value) -> {
        double before = base.metricAverages().getOrDefault(name, 0.0);
        if (value < before - tolerance) {
            failures.add("指标下降 %s: %.3f -> %.3f"
                    .formatted(name, before, value));
        }
    });
    if (now.avgLatencyMs() > base.avgLatencyMs() * 1.2) {
        failures.add("延迟超标: " + now.avgLatencyMs());
    }
    return failures;
}
```

| 门禁项 | 阈值示例 | 失败动作 |
|--------|----------|----------|
| 忠实度 | < 0.90 | 阻断合并 |
| Recall@5 | 较基线降 > 2% | 阻断合并 |
| 平均延迟 | 较基线增 > 20% | 警告/阻断 |
| 平均成本 | 较基线增 > 10% | 警告 |

## 7.3 退出码与产物

```yaml
- name: Eval regression
  if: github.event_name == 'pull_request'
  env:
    OPENAI_API_KEY: ${{ secrets.EVAL_KEY }}
  run: ./mvnw -q exec:java -Dexec.mainClass=EvalRunner
```

- 退出码 0：通过；非 0：门禁失败，CI 变红。
- 产物：`report.json` 上传为 Artifact，便于回溯。
- 失败详情：写入 PR 评论或日志，指向具体样本 ID。

# 8. 框架选型

## 8.1 按需求选型

| 需求 | 推荐 |
|------|------|
| RAG 评测 | RAGAS 指标（自建或 Python） |
| Prompt 对比 | promptfoo |
| 全流程平台 | Langfuse |
| 纯 Java 团队 | 自建轻量框架 |
| Agent 评测 | 自建（轨迹评测） |
| 单元断言式 | DeepEval |

## 8.2 混合架构

- 离线用 Python 跑基线并导出 JSON，线上用 Java Runner 做门禁并上报 Langfuse。

## 8.3 选型反模式

| 反模式 | 问题 |
|--------|------|
| 为框架而框架 | 指标不贴业务 |
| 全盘照搬指标 | 与场景不符 |
| 只用平台不建集 | 无标准样本 |
| 只自建不校准 | Judge 偏差累积 |
| 换框架丢历史 | 分数不可比 |

# 9. 生产注意事项

## 9.1 常见风险与建议

| 风险 | 建议 |
|------|------|
| 直接套用框架 | 按业务调整指标 |
| Judge 未校准 | 人工校准 |
| 报告不可比 | 统一格式 |
| 不接 CI | 接入门禁 |
| 无失败详情 | 保留每条详情 |
| 成本高 | 抽样 + 小模型 Judge |

## 9.2 Judge 校准与成本控制

### 9.2.1 抽样策略

```java
public List<EvalSample> sample(List<EvalSample> all, int maxSize) {
    if (all.size() <= maxSize) {
        return all;
    }
    List<EvalSample> copy = new ArrayList<>(all);
    Collections.shuffle(copy, new Random(42));
    return copy.subList(0, maxSize);
}
```

| 策略 | 场景 | 成本 |
|------|------|------|
| 全量 | 发版前 | 高 |
| 分层抽样 | 每日回归 | 中 |
| 小模型预筛 | 大流量 | 低 |
| 只评失败样本 | 迭代优化 | 最低 |

> 校准：抽 30~50 条人工打分，算 Judge 与人工的一致性（如 Cohen's Kappa），低于阈值先修提示。

# 10. 小结

## 10.1 核心结论

- 评测框架多为 Python，Java 可自建或通过 API 集成。
- RAGAS 定义了 RAG 评测标准指标，可复刻其逻辑。
- 自建框架：Metric 接口 + Evaluator + Report。
- Langfuse 提供数据集与评测平台能力。
- 评测必须接入 CI 并保留失败详情。

## 10.2 落地清单

| 项 | 检查 |
|----|------|
| 指标 | 覆盖忠实度/相关性/检索/成本 |
| Judge | 已用人工样本校准 |
| 样本 | 50~200 条起步，版本化 |
| 报告 | 含 runId/版本，可 JSON 导出 |
| 门禁 | CI 阻断式，有退出码 |
| 详情 | 每条样本保留理由 |
