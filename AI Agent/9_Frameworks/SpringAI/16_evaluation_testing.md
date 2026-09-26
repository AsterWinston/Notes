# 1. LLM 应用测试的特殊性

LLM（Large Language Model，大语言模型）应用的测试不能照搬传统软件的"输入确定、输出确定"模型，必须先理解其概率性与外部依赖特征，才能设计出可靠的测试策略。

## 1.1 从确定性到概率性

传统测试的前提是"同样的输入必然得到同样的输出"，而 LLM 应用打破了这一前提。

| 传统测试 | LLM 应用测试 |
|----------|--------------|
| 输入确定 → 输出确定 | 同输入可能不同输出 |
| 断言精确相等 | 需语义/相似度/规则断言 |
| 用例数量有限 | 需要评测集与统计指标 |
| 无外部依赖 | 依赖模型 API（慢、贵、可能失败） |

### 1.1.1 非确定性的来源

- **采样解码**：`temperature`、`top_p` 等参数让解码带有随机性。
- **模型版本漂移**：供应商静默升级模型，行为可能改变。
- **上下文敏感**：检索片段、记忆内容、工具结果的微小变化都会影响输出。
- **网络与超时**：外部 API 抖动导致失败或截断。

### 1.1.2 对测试设计的影响

因为输出不唯一，断言方式必须从"相等"转向"满足约束"：

- 能用规则判断的，用规则（包含/不包含、正则、JSON Schema）。
- 能用相似度判断的，用相似度阈值（Jaccard、余弦、编辑距离）。
- 只能主观判断的，用 LLM-as-Judge 或人工抽样。

## 1.2 三层测试体系

LLM 应用需要**三层测试**，逐层加码、逐层更接近真实：

```
单元测试（Mock 模型） → 集成测试（真实模型小样例） → 评测（评测集 + 指标）
```

### 1.2.1 各层职责

| 层级 | 隔离对象 | 验证目标 | 是否调用真实模型 |
|------|----------|----------|------------------|
| 单元 | 模型、网络 | 业务逻辑、容错、编排 | 否 |
| 集成 | 网络抖动 | 真实链路、结构化输出、工具联通 | 是（小样例） |
| 评测 | 无 | 质量、回归、统计指标 | 是（完整评测集） |

### 1.2.2 边界与反例

- 反例：把"模型回答得好不好"塞进单元测试，导致测试慢、贵、不稳定。
- 反例：只做评测、不做单元测试，业务逻辑 bug 被模型能力掩盖。
- 反例：只做单元测试，无法发现 Prompt 与真实模型不匹配。

## 1.3 评测驱动的开发闭环

测试与评测不是收尾工作，而是贯穿开发的闭环：

```
改 Prompt / 换模型 / 调分块
        │
        ▼
   跑评测集 ──► 指标对比基线 ──► 达标合并 / 不达标回退
        │
        └──────────────► 持续迭代
```

**结论先行**：先建立基线指标，再谈优化；没有基线的"优化"只是主观感觉。

# 2. 单元测试：Mock 模型

单元测试的核心是把模型调用隔离出去，只测试你写的业务逻辑。

## 2.1 为什么要 Mock

真实模型调用在单元测试中是**反模式**：慢（秒级）、贵（按 Token 计费）、不稳定（网络与限流）。Mock 后测试可在毫秒级完成，并可在 CI 中无限次运行。

## 2.2 Mock ChatModel

### 2.2.1 基础 Mock

```java
@Test
void should_return_fallback_when_model_fails() {
    ChatModel mockModel = mock(ChatModel.class);
    when(mockModel.call(any(Prompt.class)))
            .thenThrow(new RuntimeException("模型不可用"));

    ChatClient client = ChatClient.builder(mockModel).build();
    ChatService service = new ChatService(client);

    String result = service.ask("你好");
    assertEquals("服务暂时不可用，请稍后再试", result);
}
```

### 2.2.2 异常分支覆盖

业务容错逻辑必须逐条覆盖，常见异常分支：

| 异常场景 | 期望行为 | 测试要点 |
|----------|----------|----------|
| 超时 | 快速失败并降级 | 断言降级文案/备用模型 |
| 限流 429 | 退避重试或排队 | 断言重试次数与最终结果 |
| 格式错误 | 重试或兜底 | 断言解析失败处理 |
| 空响应 | 兜底提示 | 断言不抛空指针 |

### 2.2.3 流式 Mock

流式（streaming）场景需要 Mock `Stream<ChatResponse>`，并测试"中途失败"与"提前取消"：

```java
@Test
void stream_should_emit_partial_then_complete() {
    ChatModel mockModel = mock(ChatModel.class);
    when(mockModel.stream(any(Prompt.class)))
            .thenReturn(Flux.just(chunk("你"), chunk("好"), chunk("！")));

    ChatClient client = ChatClient.builder(mockModel).build();
    List<String> parts = client.prompt().user("打招呼")
            .stream().content().collectList().block();

    assertEquals("你好！", String.join("", parts));
}
```

> 说明：`chunk(...)` 为构造 `ChatResponse` 的辅助方法，实际字段名随 Spring AI 版本可能不同，此处为伪代码示意。

## 2.3 Mock 的边界与反例

- 不要让单元测试真的调用模型（慢、贵、不稳定）。
- 用 Mock 覆盖异常分支：超时、限流、格式错误。
- 测试重点是**业务逻辑与容错**，不是模型能力。
- 反例：Mock 返回的 JSON 是手写的、与真实模型格式不一致，测试通过但线上解析失败。

**生产建议**：Mock 的响应样本应从真实调用中采集（脱敏后）固化为夹具（fixture），保证格式贴近真实。

## 2.4 测试替身的选择

| 替身 | 用途 | 风险 |
|------|------|------|
| Mock | 验证交互、模拟异常 | 与真实行为脱节 |
| Stub | 返回固定响应 | 覆盖不足 |
| Fake | 轻量真实实现 | 维护成本 |
| 真实模型 | 集成/评测 | 慢、贵、不稳定 |

原则：**单元测试优先用 Mock/Stub，集成与评测用真实模型**，不要让真实模型渗透进快速测试。

# 3. 契约测试：结构化输出

结构化输出是 LLM 与代码之间的"接口契约"，契约测试验证解析结果是否满足约定。

## 3.1 契约测试的目标

- 字段存在且类型正确；
- 取值落在允许范围；
- 关键字段用枚举而非自由文本；
- 解析失败时有兜底路径。

## 3.2 结构化输出断言

### 3.2.1 基础断言

```java
@Test
void structured_output_should_match_schema() {
    ActorFilms films = chatClient.prompt()
            .user("列出成龙的三部电影")
            .call()
            .entity(ActorFilms.class);

    assertNotNull(films.actor());
    assertFalse(films.movies().isEmpty());
    assertTrue(films.movies().size() <= 5);
}
```

### 3.2.2 Schema 校验

对复杂结构，解析后再用 Bean Validation 或 JSON Schema 二次校验：

```java
public record Invoice(
        @NotBlank String number,
        @Positive BigDecimal amount,
        @NotNull Currency currency) {}

Invoice invoice = chatClient.prompt().user(text).call().entity(Invoice.class);

Set<ConstraintViolation<Invoice>> violations =
        Validation.buildDefaultValidatorFactory().getValidator().validate(invoice);
assertTrue(violations.isEmpty(), "结构校验失败: " + violations);
```

## 3.3 集成测试标记与运行

结构化输出测试依赖真实模型，适合做**集成测试**：用小样例、低成本模型，标记 `@Tag("integration")` 单独运行。

```java
@Tag("integration")
class StructuredOutputIT { /* ... */ }
```

```bash
# 仅运行集成测试
mvn test -Dgroups=integration
# 运行全部（含集成）
mvn verify
```

> 边界：结构化输出并非 100% 可靠，尤其在复杂嵌套结构下。测试应同时覆盖"解析失败"分支。

## 3.4 契约版本管理

结构化输出的 record 一旦被下游消费，就成为契约：字段改名、删除、类型变更都属于破坏性变更，需要版本化或兼容处理。

- 新增可选字段：通常兼容；
- 删除/重命名字段：破坏性，需新版本；
- 枚举新增值：下游需能容错。

# 4. Spring AI 的 Evaluator

Spring AI 提供评测抽象（`Evaluator`），用于自动化评估输出质量，把主观判断转化为可统计的分数。

## 4.1 评测抽象与流程

```
测试数据 → 调用模型 → Evaluator 打分 → 汇总指标
```

常见评测维度：

| 维度 | 说明 |
|------|------|
| 相关性（Relevance） | 回答是否切题 |
| 忠实度（Faithfulness） | 是否忠于给定资料（RAG） |
| 正确性（Correctness） | 与标准答案是否一致 |
| 简洁性 | 是否啰嗦 |

> 说明：Spring AI 各版本中 `Evaluator` 的具体接口名与方法签名不同，下文示例为示意，落地时请以所依赖版本的 API 为准。

## 4.2 基于规则/相似度的评测

### 4.2.1 相似度指标

```java
double similarity = new JaccardSimilarityEvaluator()
        .evaluate(expectedAnswer, actualAnswer);
```

常用相似度及其适用场景：

| 指标 | 计算 | 适用 |
|------|------|------|
| Jaccard | 词集合交并比 | 短答案、关键词覆盖 |
| 余弦相似度 | 向量夹角 | 语义相近的长文本 |
| 编辑距离 | 字符级差异 | 格式化文本、代码 |
| ROUGE/BLEU | n-gram 重叠 | 摘要、翻译（参考） |

### 4.2.2 规则断言

相似度之外，业务上常用规则断言，简单且可解释：

```java
assertAll(
    () -> assertTrue(actual.contains("30 天")),
    () -> assertFalse(actual.contains("我不知道")),
    () -> assertTrue(actual.length() < 800)
);
```

### 4.2.3 自定义 Evaluator

```java
public class KeywordCoverageEvaluator implements Evaluator {
    @Override
    public EvaluationResult evaluate(EvaluationRequest req) {
        List<String> missing = req.mustContain().stream()
                .filter(k -> !req.actual().contains(k))
                .toList();
        double score = 1.0 - (double) missing.size() / req.mustContain().size();
        return new EvaluationResult(score, "缺失关键词: " + missing);
    }
}
```

> 说明：`EvaluationRequest`、`EvaluationResult` 为示意类型，实际接口以所用 Spring AI 版本为准。

## 4.3 基于 LLM 的评测（LLM-as-Judge）

用一个大模型当"裁判"给另一个模型的回答打分，是自动化评测的主力手段。

### 4.3.1 Judge 提示设计

```java
String judgePrompt = """
        请评估下面回答的质量，从 1-5 分打分，并说明理由。
        问题：%s
        参考答案：%s
        模型回答：%s
        输出 JSON：{"score": number, "reason": "string"}
        """.formatted(question, expected, actual);

JudgeResult result = judgeChatClient.prompt()
        .user(judgePrompt)
        .call()
        .entity(JudgeResult.class);
```

设计要点：

- **评分标准要写死**（1-5 各档含义），否则分数不可比；
- **要求输出结构化 JSON**，便于聚合；
- **提供参考答案**（如有），减少主观漂移；
- **控制随机性**：Judge 用较低 `temperature`。

### 4.3.2 偏差与校准

LLM-as-Judge 便宜高效，但有位置偏差、自偏好等风险，需校准：

| 偏差 | 表现 | 缓解 |
|------|------|------|
| 位置偏差 | 更偏好靠前的答案 | 交换顺序各评一次取平均 |
| 自偏好 | 偏好自家模型的输出 | 用第三方模型做裁判 |
| 长度偏差 | 长回答得分更高 | 显式惩罚冗余 |
| 分数漂移 | 不同批次标准不一 | 固定 Judge 版本 + 校准集 |

> LLM-as-Judge 详见 `12_Evaluation/3_llm_as_judge.md`。

# 5. 评测集与回归测试

评测集是"质量基线"的载体，回归测试是防止质量退化的闸门。

## 5.1 构建评测集

### 5.1.1 用例结构

```json
[
  {
    "id": "q1",
    "question": "报销流程是怎样的？",
    "expected": "员工需在费用发生后 30 天内提交申请……",
    "mustContain": ["30 天", "提交申请"],
    "mustNotContain": ["我不知道"]
  }
]
```

### 5.1.2 覆盖设计

好的评测集应覆盖：

- **正常场景**：典型问题，占多数；
- **边界场景**：空输入、超长输入、多语言；
- **异常场景**：模型无法回答、资料缺失、工具失败；
- **对抗场景**：提示注入、诱导越权、恶意输入。

### 5.1.3 评测数据来源

- 真实用户问题（脱敏）——最有价值；
- 领域专家构造——覆盖长尾与专业场景；
- 线上失败案例回流——直接修复薄弱点；
- 合成数据（LLM 生成）——快速扩量，但需人工抽检。

## 5.2 回归测试

每次改 Prompt / 换模型 / 改分块，都跑一遍评测集：

### 5.2.1 阈值断言

```java
@Test
void prompt_regression() {
    List<EvalCase> cases = loadCases("eval/qa-cases.json");
    EvalReport report = evaluator.evaluate(cases, ragChatClient);

    assertTrue(report.accuracy() >= 0.85,
            "准确率下降: " + report.accuracy());
    assertTrue(report.avgLatencyMs() < 3000,
            "延迟超标: " + report.avgLatencyMs());
}
```

### 5.2.2 接入 CI

```yaml
# .github/workflows/eval.yml（示意）
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn -q -Dgroups=evaluation test
```

> 把评测接入 CI，Prompt 变更必须通过阈值才能合并，是**防止退化**的关键。

# 6. RAG 专项评测

RAG（Retrieval-Augmented Generation，检索增强生成）的评测要**把检索和生成拆开**，否则无法定位问题出在哪一环。

## 6.1 检索与生成分离

| 环节 | 输入 | 输出 | 评测关注 |
|------|------|------|----------|
| 检索 | 问题 | 文档片段 | 召回率、精确率 |
| 生成 | 问题 + 片段 | 回答 | 忠实度、相关性 |

## 6.2 指标详解

| 指标 | 说明 |
|------|------|
| 上下文召回率 | 正确文档是否被检索到 |
| 上下文精确率 | 检索结果中有多少是相关的 |
| 忠实度 | 回答是否基于检索内容 |
| 答案相关性 | 回答是否切题 |

## 6.3 常见问题诊断

- 召回率低 → 分块/Embedding/查询改写问题；
- 召回率高但忠实度低 → 生成阶段幻觉，提示或模型问题；
- 忠实度高但相关性低 → 检索到了无关资料。

## 6.4 检索指标计算示例

```
召回率 = 命中的相关文档数 / 全部相关文档数
精确率 = 命中的相关文档数 / 检索返回的文档数
```

先标注"哪些文档是相关的"（ground truth，标准答案），再对每次检索结果逐条计算，最后取平均。

> 详见 `5_RAG/Evaluation/1_rag_evaluation.md`。

# 7. Agent 评测

Agent（智能体）评测要看**结果**与**过程**，只看最终答案会掩盖大量问题。

## 7.1 结果与过程双维度

| 维度 | 指标 |
|------|------|
| 结果 | 任务成功率、答案正确率 |
| 过程 | 工具调用次数、是否走弯路、是否死循环 |
| 成本 | 总 Token、总耗时 |
| 鲁棒性 | 异常输入下的表现 |

## 7.2 轨迹评测

轨迹（trajectory）是 Agent 的完整调用序列，评测点包括：

```
Trace: 用户提问
├── 模型调用（决定工具）
├── 工具调用 getOrder("A123")
├── 模型调用（生成回答）
```

- **步数**：是否用了超过预期的轮数；
- **路径**：是否调用了无关工具；
- **终止**：是否正确结束，而非死循环。

## 7.3 成本与鲁棒性

- 统计每任务的平均 Token 与耗时，设置上限；
- 注入异常输入（空、乱码、超长、越权意图）观察退化行为。

## 7.4 自动评测与人工评审结合

- 自动评测负责回归与大规模筛选；
- 人工评审负责校准标准、发现新问题；
- 二者结果不一致的样本，是最有价值的分析对象。

> 详见 `12_Evaluation/5_agent_evaluation.md` 与 `6_trajectory_evaluation.md`。

# 8. 测试分层与 CI 策略

三层测试需要在 CI 中按不同频率运行，平衡成本与保护力度。

## 8.1 分层金字塔

```
快速单元测试（每次提交，Mock 模型）
        │
        ▼
集成测试（每日/合并前，真实模型小样例）
        │
        ▼
评测回归（发版前，完整评测集）
```

## 8.2 频率与成本权衡

| 层级 | 频率 | 成本 | 目的 |
|------|------|------|------|
| 单元 | 每次提交 | 低 | 业务逻辑正确 |
| 集成 | 每日 | 中 | 真实链路可用 |
| 评测 | 发版前 | 高 | 质量不退化 |

## 8.3 CI 流水线示例

```yaml
# 分层执行（示意）：unit 每次 push，integration 每日，evaluation 发版前
stages: [unit, integration, evaluation]
```

> 边界：评测集调用真实模型有成本，应设置并发上限与超时，避免 CI 长时间占用；偶发失败自动重试，稳定失败视为质量退化并阻断合并。

# 9. 生产注意事项

## 9.1 评测质量风险

| 风险 | 建议 |
|------|------|
| 评测集过拟合 | 保留未参与调优的测试集 |
| 评测不稳定 | 多次采样取均值，设置容差 |
| 评测成本高 | 用小模型做 Judge，抽样评测 |
| 只测 Happy Path | 覆盖异常、边界、恶意输入 |
| 无版本记录 | 评测集与 Prompt 一起版本化 |

## 9.2 成本与稳定性

- 评测集规模控制在可承受范围，优先高价值用例；
- 使用缓存与并发控制，降低重复调用成本；
- 对不稳定用例标记 `flaky`，单独跟踪而非直接忽略。

## 9.3 版本与合规

- 评测集、Prompt、模型版本三者绑定记录；
- 评测数据若含用户隐私，必须脱敏并限制访问；
- 结果报告留档，便于回溯"哪个版本开始退化"。

## 9.4 组织与流程

- 评测集由业务与工程共同维护；
- 每次上线前必须附评测报告；
- 指标退化需给出原因与整改计划，而非直接放宽阈值。

# 10. 小结

## 10.1 核心要点

- 单元测试 Mock 模型，集成测试验真实链路，评测集防退化。
- LLM-as-Judge 是自动化评测的主力，但要防偏差。
- RAG 看检索与忠实度，Agent 看结果与轨迹。

## 10.2 落地建议

- 评测必须接入 CI，否则优化只是"感觉更好"。
- 评测集、Prompt、模型版本三者绑定，指标退化可回溯。
- 保留一份未参与调优的测试集，防止过拟合。
