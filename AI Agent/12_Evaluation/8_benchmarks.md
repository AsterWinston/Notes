# 1. 什么是基准测试

**结论先行：基准（Benchmark）是公开的标准化评测集，用于横向对比模型/Agent 能力，但它只能参考，不能替代自有评测。**

## 1.1 定义与作用

### 1.1.1 三大作用

```
作用：
① 选型参考（哪个模型更强）
② 追踪进展（新版本是否提升）
③ 发现问题（短板在哪）
```

| 作用 | 说明 | 局限 |
|------|------|------|
| 选型参考 | 横向对比候选模型 | 通用分不等于业务分 |
| 追踪进展 | 版本迭代看趋势 | 受污染影响可能虚高 |
| 发现问题 | 定位能力短板 | 粒度粗，难归因 |

### 1.1.2 基准的构成要素

一个可用的基准通常包含四部分：任务集合、参考答案或验证器、评测脚本、以及报告规范。缺少任意一项，结果都难以复现。

| 要素 | 说明 |
|------|------|
| 任务集 | 输入样本集合 |
| 验证器 | 判定正确性的程序 |
| 运行器 | 统一执行与计分 |
| 报告 | 指标与运行环境记录 |

## 1.2 基准与自有评测的关系

### 1.2.1 初筛与终选

基准回答"模型在公共任务上表现如何"，自有评测回答"模型在我的业务上表现如何"。二者是**初筛与终选**的关系，不可互相替代。

> **基准只能参考，不能替代自有评测**。

# 2. 通用能力基准

**结论先行：通用能力基准衡量知识与推理，是选型的粗筛工具，但对具体 Agent 场景的预测力有限。**

## 2.1 知识类基准

### 2.1.1 MMLU 与 GPQA

| 基准 | 考察 | 形式 |
|------|------|------|
| MMLU | 多学科知识 | 选择题 |
| GPQA | 研究生级科学 | 选择题 |

MMLU 覆盖 57 个学科，是最广泛使用的知识基准；GPQA 难度更高，用于区分顶尖模型。

## 2.2 推理类基准

### 2.2.1 HellaSwag、ARC 与 BBH

| 基准 | 考察 | 形式 |
|------|------|------|
| HellaSwag | 常识推理 | 选择 |
| ARC | 科学推理 | 选择 |
| BBH | 复杂推理 | 多样 |

## 2.3 对话类基准

### 2.3.1 MT-Bench

| 基准 | 考察 | 形式 |
|------|------|------|
| MT-Bench | 对话质量 | LLM 评分 |

MT-Bench 使用强模型作为裁判（LLM-as-a-Judge），需注意裁判模型自身的偏好偏差，如倾向于更长的回答。

## 2.4 多模态与长上下文

### 2.4.1 MMMU 与 RULER

| 基准 | 考察 | 说明 |
|------|------|------|
| MMMU | 多模态理解 | 图文混合题 |
| RULER | 长上下文 | 检索与聚合 |

长上下文基准常通过"大海捞针（Needle in a Haystack）"检验模型在超长输入中定位信息的能力。

# 3. 编码能力基准

**结论先行：编码基准从函数级走向仓库级，越贴近真实工程，越能反映 Agent 的实际能力。**

## 3.1 函数级基准

### 3.1.1 HumanEval 与 MBPP

| 基准 | 考察 |
|------|------|
| HumanEval | 函数级代码生成 |
| MBPP | 基础编程 |

## 3.2 仓库级基准

### 3.2.1 SWE-bench

SWE-bench 基于真实 GitHub Issue，要求 Agent 修改仓库代码并通过仓库自带测试，是最接近真实软件工程的基准之一。

## 3.3 无污染基准

### 3.3.1 LiveCodeBench

| 基准 | 考察 |
|------|------|
| LiveCodeBench | 无污染的编码题 |

LiveCodeBench 持续从新比赛中取题，降低训练数据污染风险。

# 4. Agent 能力基准

**结论先行：Agent 基准强调多步工具使用与环境交互，是最贴近本笔记主题的一类基准。**

## 4.1 通用助手

### 4.1.1 GAIA

| 基准 | 考察 | 说明 |
|------|------|------|
| GAIA | 通用助手任务 | 多步工具使用 |

## 4.2 网页与桌面

### 4.2.1 WebArena 与 OSWorld

| 基准 | 考察 | 说明 |
|------|------|------|
| WebArena | 网页操作 | 真实网站环境 |
| OSWorld | 计算机操作 | 桌面任务 |

## 4.3 工具与多环境

### 4.3.1 τ-bench 与 AgentBench

| 基准 | 考察 | 说明 |
|------|------|------|
| AgentBench | 多环境 Agent | 综合 |
| τ-bench | 工具使用 | 对话 + 工具 |
| SWE-bench | 软件工程 | 代码修复 |

# 5. RAG 能力基准

**结论先行：RAG 基准分别从检索、表示、鲁棒性三个角度评估，选型时应组合使用。**

## 5.1 检索泛化

### 5.1.1 BEIR

| 基准 | 考察 |
|------|------|
| BEIR | 检索泛化 |

BEIR 由多个异构检索数据集组成，用于检验检索器的零样本泛化能力。

## 5.2 表示与鲁棒性

### 5.2.1 MTEB、RGB 与 CRUD-RAG

| 基准 | 考察 |
|------|------|
| MTEB | Embedding 综合 |
| RGB | RAG 鲁棒性 |
| CRUD-RAG | 中文 RAG |

## 5.3 检索指标公式

### 5.3.1 Recall@k 与 nDCG

```
Recall@k = |相关文档 ∩ 前k结果| / |相关文档|
Precision@k = |相关文档 ∩ 前k结果| / k
MRR = (1/|Q|) * Σ 1 / rank_i
nDCG@k = DCG@k / IDCG@k
DCG@k = Σ (2^rel_i - 1) / log2(i + 1)
```

| 指标 | 关注点 | 适用 |
|------|--------|------|
| Recall@k | 是否召回 | 检索阶段 |
| Precision@k | 是否精准 | 上下文有限时 |
| MRR | 首个正确位置 | 问答 |
| nDCG@k | 排序质量 | 分级相关 |

> 生产建议：RAG 选型至少同时看 Recall@k 与 nDCG@k，单一指标易失真。

# 6. 基准的局限

**结论先行：基准存在数据污染、与业务脱节、可被优化、静态、只看分数五类局限，必须警惕。**

## 6.1 污染与脱节

| 局限 | 说明 |
|------|------|
| 数据污染 | 训练集可能包含测试题 |
| 与业务脱节 | 通用基准 ≠ 你的场景 |

### 6.1.1 污染检测

污染检测常用手段：对测试样本做改写后重测、检查模型对题目的困惑度、用时间切分验证。若改写后分数骤降，往往说明存在记忆而非推理。

## 6.2 可被优化、静态与只看分数

| 局限 | 说明 |
|------|------|
| 可被优化 | 针对基准调优 |
| 静态 | 不反映最新需求 |
| 只看分数 | 忽略延迟与成本 |

### 6.2.1 Goodhart 定律

当指标变成目标，它就不再是好指标。针对基准过拟合会牺牲真实泛化能力，因此必须用未公开的自有评测集兜底。

> **不要用基准分数代替业务评测。**

# 7. 使用建议与选型流程

**结论先行：基准用于初筛，自有评测用于终选，最后结合延迟与成本综合判断。**

## 7.1 从初筛到终选

```
① 用基准做初筛（选 2~3 个候选模型）
② 用自有评测集做终选
③ 结合延迟与成本综合判断
```

## 7.2 Java 实现

### 7.2.1 选型报告

```java
public ModelSelectionReport selectModel(List<String> candidates) {
    return candidates.stream().map(model -> {
        // ① 基准分（参考）
        double benchmarkScore = benchmarkRegistry.score(model, "MTEB");

        // ② 自有评测（决定）
        EvalReport own = evaluateWithModel(model, ownEvalSet);

        // ③ 成本与延迟
        long latency = measureLatency(model);
        BigDecimal cost = measureCost(model);

        return new ModelSelectionReport(model, benchmarkScore, own,
                latency, cost);
    }).toList();
}
```

### 7.2.2 加权综合打分

```java
public double compositeScore(ModelSelectionReport r) {
    // 业务分权重最高，基准分仅作参考
    return 0.60 * r.own().qualityScore()
            + 0.20 * r.benchmarkScore()
            + 0.10 * latencyScore(r.latency())
            + 0.10 * costScore(r.cost());
}
```

## 7.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 基准分权重过高 | 选到业务不佳模型 | 业务分主导 |
| 忽略长尾 | 只看平均分 | 看最差场景 |
| 未锁定版本 | 模型更新致漂移 | 记录模型版本 |

> 生产建议：选型报告应同时给出质量、延迟、成本三维数据，而非单一分数。

# 8. SWE-bench 类评测要点

**结论先行：SWE-bench 类评测以"真实仓库 + 真实 Issue + 自动化测试"为核心，可复现性是关键。**

## 8.1 基本特点

```
特点：真实仓库 + 真实 Issue + 自动化测试验证
评测：能否让测试通过
```

## 8.2 核心要素

### 8.2.1 环境隔离与测试验证

| 要点 | 说明 |
|------|------|
| 环境隔离 | 每个任务独立容器 |
| 测试验证 | 用仓库测试判断 |
| 结果 | 通过率（resolve rate） |

### 8.2.2 失败归因

失败常见于：定位错文件、修改不完整、引入回归、无法运行测试。评测应记录失败类型分布，而非只看通过率。

## 8.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 环境不一致 | 本地能过 CI 不过 | 容器化固定依赖 |
| 测试泄漏 | Agent 看到答案 | 隔离测试文件 |
| 单次判定 | 随机导致抖动 | 多次运行 |

> 生产建议：自建软件工程基准时，务必固定依赖版本并隔离测试用例。

# 9. 自建领域基准

**结论先行：当通用基准不适用时，构建领域基准是唯一能反映真实业务质量的手段。**

## 9.1 何时自建

通用基准与业务脱节、数据敏感无法外流、需要特定工具链时，应自建领域基准。

## 9.2 Java 实现

### 9.2.1 DomainBenchmark

```java
public class DomainBenchmark {

    public BenchmarkReport run(List<DomainTask> tasks) {
        int passed = 0;
        for (DomainTask task : tasks) {
            AgentTrace trace = agent.runWithTrace(task.goal());
            if (task.validator().validate(trace)) {
                passed++;
            }
        }
        return new BenchmarkReport(
                tasks.size(),
                passed,
                passed / (double) tasks.size());
    }
}
```

### 9.2.2 自动验证器

验证器应尽量基于可判定的信号，而非 LLM 主观评分：

```java
public interface Validator {
    boolean validate(AgentTrace trace);
}

// 结构化验证：检查最终答案字段
public class JsonFieldValidator implements Validator {
    private final String field;
    private final Object expected;

    public JsonFieldValidator(String field, Object expected) {
        this.field = field;
        this.expected = expected;
    }

    @Override
    public boolean validate(AgentTrace trace) {
        Map<String, Object> answer = parseJson(trace.finalAnswer());
        return Objects.equals(answer.get(field), expected);
    }
}
```

## 9.3 领域基准的要素

| 要素 | 说明 |
|------|------|
| 真实任务 | 来自业务 |
| 自动验证 | 可程序判定 |
| 环境固定 | 可复现 |
| 版本化 | 可对比 |

## 9.4 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 任务过拟合 | 模型"背"答案 | 定期换题 |
| 验证器过松 | 假通过 | 人工抽检 |
| 无版本 | 无法对比 | 基准版本化 |

> 生产建议：领域基准应像代码一样纳入版本管理，每次发版记录基准版本号。

# 10. 结果解读与统计

**结论先行：小差距应视为持平，波动大必须多次运行取均值，与业务不符时以业务评测为准。**

## 10.1 分数解读

| 分数 | 解读 |
|------|------|
| 差距大 | 明确优劣 |
| 差距小（<2%） | 视为持平，看成本 |
| 波动大 | 多次运行取均值 |
| 与业务不符 | 以业务评测为准 |

## 10.2 统计显著性

### 10.2.1 多次运行与置信区间

```java
public StatResult evaluateWithConfidence(List<EvalCase> cases, int runs) {
    List<Double> scores = new ArrayList<>();
    for (int i = 0; i < runs; i++) {
        scores.add(evaluate(cases));
    }
    double mean = scores.stream().mapToDouble(Double::doubleValue).average()
            .orElse(0.0);
    double std = Math.sqrt(scores.stream()
            .mapToDouble(s -> Math.pow(s - mean, 2)).average().orElse(0.0));
    return new StatResult(mean, std, 1.96 * std / Math.sqrt(runs));
}
```

### 10.2.2 配对比较

对同一组用例比较两个模型时，应使用配对差异而非各自均值，减少用例难度带来的方差。

```java
public double pairedDiff(List<EvalCase> cases, Model a, Model b) {
    double sum = 0;
    for (EvalCase c : cases) {
        sum += score(a, c) - score(b, c);
    }
    return sum / cases.size();
}
```

## 10.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 单次定论 | 随机抖动 | 多次取均值 |
| 忽略方差 | 只看均值 | 报标准差 |
| 样本过少 | 不显著 | 扩样本 |

> 生产建议：报告分数时必须附运行次数与标准差，否则不可信。

# 11. 评测框架与工程化

**结论先行：基准要真正落地，必须有统一的运行器、结果存储与可视化看板。**

## 11.1 统一运行器

### 11.1.1 Java Runner

```java
public class BenchmarkRunner {

    private final Map<String, Benchmark> registry = new HashMap<>();

    public void register(String name, Benchmark benchmark) {
        registry.put(name, benchmark);
    }

    public BenchmarkReport runAll(List<EvalCase> cases) {
        Map<String, Double> scores = new LinkedHashMap<>();
        for (Map.Entry<String, Benchmark> e : registry.entrySet()) {
            scores.put(e.getKey(), e.getValue().score(cases));
        }
        return new BenchmarkReport(scores, Instant.now());
    }
}
```

## 11.2 结果存储与看板

### 11.2.1 结果 schema

```json
{
  "runId": "2026-09-21-001",
  "model": "gpt-x",
  "benchmarks": {"MTEB": 0.72, "GAIA": 0.41},
  "runs": 5,
  "std": 0.013,
  "latencyMs": 1850,
  "cost": 0.018
}
```

> 生产建议：每次运行记录模型版本、运行次数、标准差，便于趋势对比与回归。

# 12. 生产注意事项

**结论先行：基准使用要避免迷信、污染、忽略成本、单次运行、无自有基准、不追踪六类风险。**

## 12.1 风险清单

| 风险 | 建议 |
|------|------|
| 迷信基准 | 业务评测为准 |
| 数据污染 | 用无污染基准 |
| 忽略成本 | 综合判断 |
| 单次运行 | 多次取均值 |
| 无自有基准 | 建领域基准 |
| 不追踪 | 定期复测 |

## 12.2 落地建议

- 基准只用于初筛，业务评测决定最终选型。
- 关注无污染基准（如 LiveCodeBench）。
- 选型报告包含质量、延迟、成本三维。
- 自建领域基准并版本化管理。
- 定期复测，追踪模型版本漂移。

# 13. 小结

## 13.1 核心结论

- 基准用于初筛与追踪，不能替代业务评测。
- 主要基准：MMLU、SWE-bench、GAIA、WebArena、MTEB。
- 基准有数据污染、与业务脱节等局限。
- 领域基准应自建，具备真实任务与自动验证。

## 13.2 落地清单

- 选型结合基准、自有评测、延迟与成本综合判断。
- 小差距视为持平，波动大需多次运行取均值。
