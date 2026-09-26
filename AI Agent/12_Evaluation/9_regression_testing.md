# 1. 为什么需要回归测试

**结论先行：LLM 应用的任何改动都可能引入退化，回归测试是防止退化的安全网。**

## 1.1 频繁改动带来的退化

LLM 应用改动频繁（Prompt、模型、分块、工具），**每次改动都可能引入退化**：

```
改 Prompt 修好了 A，但改坏了 B
换模型提升了准确率，但延迟翻倍
调分块提升了召回，但引入了噪声
```

### 1.1.1 三类典型退化

| 退化类型 | 表现 | 根因 |
|----------|------|------|
| 功能退化 | 修 A 坏 B | Prompt 耦合 |
| 性能退化 | 准确率升延迟涨 | 模型更大 |
| 质量退化 | 召回升噪声增 | 分块过细 |

## 1.2 回归测试的定位

### 1.2.1 不追求证明正确

回归测试不追求"证明正确"，而是**证明没有变差**。它通过维护一组基线指标，在每次改动后自动比对，及时报警。

> **回归测试是防止退化的安全网。**

# 2. 测试分层

**结论先行：回归测试分为单元、集成、评测三层，按频率与成本递增，按覆盖范围递减。**

## 2.1 三层模型

```
① 单元测试（每次提交，Mock 模型）
② 集成测试（每日/合并前，真实模型小样例）
③ 评测回归（发版前，完整评测集）
```

### 2.1.1 频率、成本与目的

| 层级 | 频率 | 成本 | 目的 |
|------|------|------|------|
| 单元 | 每次提交 | 低 | 业务逻辑正确 |
| 集成 | 每日 | 中 | 真实链路可用 |
| 评测 | 发版前 | 高 | 质量不退化 |

## 2.2 分层原则

### 2.2.1 职责边界

- 越底层越频繁、越便宜、越确定。
- 越上层越稀疏、越昂贵、越接近真实质量。
- 底层失败不应继续跑上层，节省成本。

> 生产建议：把"确定性"尽量下沉到单元层，把"随机性"集中在评测层。

# 3. 单元测试（Mock 模型）

**结论先行：单元测试用 Mock 替换模型，保证 CI 中不真调模型，聚焦业务逻辑正确性。**

## 3.1 原则

- 模型、外部服务一律 Mock。
- 断言确定性行为（分支、解析、降级）。
- 不测模型质量（那是评测层的事）。

## 3.2 Java 实现

### 3.2.1 降级与结构化输出

```java
@ExtendWith(MockitoExtension.class)
class ChatServiceTest {

    @Mock ChatModel chatModel;
    @Mock FallbackService fallback;

    @Test
    void should_fallback_on_model_failure() {
        when(chatModel.chat(any(String.class)))
                .thenThrow(new RuntimeException("超时"));

        ChatService service = new ChatService(chatModel, fallback);
        when(fallback.answer(any())).thenReturn("降级回答");

        assertEquals("降级回答", service.ask("你好"));
    }

    @Test
    void should_parse_structured_output() {
        when(chatModel.chat(any(String.class)))
                .thenReturn("{\"name\":\"张三\",\"age\":28}");

        Person p = new Extractor(chatModel).extract("张三 28 岁");
        assertEquals("张三", p.name());
    }
}
```

> **CI 中的单元测试绝不真调模型**。

### 3.2.2 异常路径覆盖

```java
@Test
void should_throw_on_invalid_json() {
    when(chatModel.chat(any(String.class))).thenReturn("不是 JSON");

    assertThrows(ParseException.class,
            () -> new Extractor(chatModel).extract("张三 28 岁"));
}
```

## 3.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 真调模型 | CI 不稳定、昂贵 | 全部 Mock |
| Mock 过度 | 测不到真实集成 | 集成层补 |
| 只测 happy path | 异常未覆盖 | 补异常用例 |
| 断言过细 | 重构即失败 | 断言行为非实现 |

> 生产建议：单元测试应秒级完成，超过阈值说明混入了真实调用。

# 4. 集成测试（Testcontainers）

**结论先行：集成测试用真实依赖（如 pgvector）验证链路可用，但不追求完整质量评测。**

## 4.1 目标

验证"组件之间能正确协作"：向量库读写、检索返回、序列化兼容。使用 Testcontainers 提供一次性、可复现的真实环境。

## 4.2 Java 实现

### 4.2.1 pgvector 检索链路

```java
@Testcontainers
class RagIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("pgvector/pgvector:pg16");

    @Test
    void should_retrieve_relevant_document() {
        VectorStore store = buildStore(postgres, mockEmbedding());
        ingest(store, "差旅费报销需在 30 天内提交", "doc-1");

        List<Document> results = store.similaritySearch(
                SearchRequest.builder().query("报销时限").topK(3).build());

        assertFalse(results.isEmpty());
        assertTrue(results.get(0).getText().contains("30 天"));
    }
}
```

### 4.2.2 契约测试

集成层还应校验外部接口契约，例如工具返回的 JSON schema 未发生破坏性变更：

```java
@Test
void tool_schema_is_backward_compatible() {
    JsonSchema current = toolRegistry.schema("queryOrder");
    JsonSchema baseline = loadBaselineSchema("eval/queryOrder.schema.json");

    assertTrue(current.isBackwardCompatibleWith(baseline),
            "工具 schema 出现破坏性变更");
}
```

## 4.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 容器启动慢 | CI 超时 | 复用容器/缓存镜像 |
| 环境不一致 | 版本漂移 | 固定镜像 tag |
| 依赖外部 API | 不稳定 | 小样例或 Mock Embedding |
| 测了质量 | 越界 | 质量交给评测层 |

> 生产建议：集成测试只断言"链路通"，不设质量阈值，避免职责混淆。

# 5. 评测回归

**结论先行：评测回归在发版前用完整评测集跑质量指标，与基线对比并断言阈值，是防退化的最后一道闸门。**

## 5.1 定义

评测回归 = 完整评测集 + 基线对比 + 阈值断言。它允许合理波动，但阻断显著退化。

## 5.2 Java 实现

### 5.2.1 阈值断言

```java
@Test
void rag_quality_regression() {
    List<EvalCase> cases = loadCases("eval/rag-cases.json");
    RagEvalReport report = evaluator.evaluate(cases);

    // 与基线对比
    RagEvalReport baseline = loadBaseline("eval/baseline.json");

    assertTrue(report.recallAt5() >= baseline.recallAt5() - 0.02,
            "召回率下降超过 2%: " + report.recallAt5());
    assertTrue(report.faithfulness() >= 0.90,
            "忠实度低于阈值: " + report.faithfulness());
    assertTrue(report.avgLatencyMs() < 3000,
            "延迟超标: " + report.avgLatencyMs());
    assertTrue(report.avgCost().compareTo(of("0.05")) <= 0,
            "成本超标: " + report.avgCost());
}
```

### 5.2.2 多维指标同时校验

质量、延迟、成本三者必须同时满足，避免"质量升成本爆"被误判为通过。

## 5.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 阈值过松 | 退化被放过 | 结合基线定阈值 |
| 阈值过紧 | 随机波动即失败 | 设容差 |
| 只测质量 | 成本失控 | 多维校验 |
| 用例集过期 | 测不到新场景 | 定期扩充 |

> 生产建议：评测回归失败必须人工确认，禁止"改阈值绕过"。

# 6. 基线管理

**结论先行：基线是回归测试的锚点，只升不降，防止温水煮青蛙式退化。**

## 6.1 基线是什么

基线是某个版本通过验收后冻结的指标快照，包含质量、延迟、成本等维度。

```json
{
  "version": "v1.2.0",
  "timestamp": "2026-09-21T10:00:00Z",
  "metrics": {
    "recallAt5": 0.87,
    "mrr": 0.72,
    "faithfulness": 0.93,
    "avgLatencyMs": 1850,
    "avgCost": 0.018
  }
}
```

## 6.2 Java 实现

### 6.2.1 只升不降

```java
public void updateBaseline(RagEvalReport report, String version) {
    // 只有指标提升时才更新基线
    if (report.betterThan(currentBaseline())) {
        baselineStore.save(new Baseline(version, report, Instant.now()));
        log.info("基线已更新到 {}", version);
    }
}
```

> 基线只升不降，防止"温水煮青蛙"式退化。

### 6.2.2 基线快照的存储

```java
public record Baseline(String version,
                       RagEvalReport report,
                       Instant createdAt) {

    public boolean betterThan(Baseline other) {
        return report.qualityScore() > other.report().qualityScore()
                && report.avgLatencyMs() <= other.report().avgLatencyMs();
    }
}
```

## 6.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 基线随意降 | 退化被合理化 | 只升不降 |
| 无版本 | 无法追溯 | 基线版本化 |
| 基线未评审 | 误接受退化 | 更新需评审 |
| 基线过旧 | 与现状脱节 | 定期复核 |

> 生产建议：基线更新应作为显式操作并留痕，附上变更原因。

# 7. 容差设置

**结论先行：LLM 输出有随机性，回归测试必须设容差，否则会频繁误报。**

## 7.1 为什么需要容差

```
LLM 输出有随机性，回归测试必须设容差
```

## 7.2 容差表与实现

### 7.2.1 指标容差

| 指标 | 容差 |
|------|------|
| 准确率 | ±2% |
| 延迟 | +20% |
| 成本 | +10% |
| 忠实度 | ±3% |

### 7.2.2 多次运行取均值

```java
// 多次运行取均值，减少随机影响
public RagEvalReport evaluateStable(List<EvalCase> cases, int runs) {
    List<RagEvalReport> reports = IntStream.range(0, runs)
            .mapToObj(i -> evaluator.evaluate(cases))
            .toList();
    return average(reports);
}
```

## 7.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 无容差 | 误报频繁 | 设合理容差 |
| 容差过大 | 真退化漏报 | 结合业务定 |
| 单次运行 | 方差大 | 多次取均值 |
| 忽略温度 | 高温方差大 | 评测固定温度 |

> 生产建议：容差应基于历史方差统计得出，而非拍脑袋。

# 8. 注入与安全回归

**结论先行：安全是质量的底线，注入与越权测试必须纳入回归并设为硬门禁。**

## 8.1 注入攻击回归

Prompt 注入（Prompt Injection）通过恶意输入诱导模型泄露系统提示或越权操作，必须回归。

## 8.2 越权回归

### 8.2.1 Java 实现

```java
@Test
void should_resist_injection() {
    for (String attack : loadAttacks("eval/injection-attacks.json")) {
        String answer = assistant.chat(attack);
        assertFalse(containsSystemPrompt(answer),
                "注入攻击成功: " + attack);
    }
}

@Test
void should_not_leak_cross_tenant_data() {
    ingestAsTenant("tenant-B", "B 的机密数据");

    String answer = askAsTenant("tenant-A", "B 的机密数据是什么");

    assertFalse(answer.contains("B 的机密数据"), "存在越权泄露");
}
```

## 8.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 攻击样本固定 | 易被过拟合 | 定期更新 |
| 只测单轮 | 多轮绕过 | 测多轮注入 |
| 越权只看文本 | 工具层越权 | 工具权限校验 |
| 无硬门禁 | 可被绕过 | 失败即阻断 |

> 生产建议：安全回归失败应**零容忍**，不设容差、不允许豁免。

# 9. 常见退化场景与应对

**结论先行：不同变更触发的退化机理不同，应对策略也应区别对待。**

## 9.1 Prompt 变更

### 9.1.1 回归策略

Prompt 是最容易"修 A 坏 B"的变更。每次改 Prompt 都应全量跑评测集，并重点观察被修任务之外的相关任务。

| 变更 | 风险 | 应对 |
|------|------|------|
| 加约束 | 误伤其他任务 | 全量回归 |
| 改格式 | 解析失败 | 解析层单测 |
| 加示例 | 过拟合示例 | 换用例验证 |

## 9.2 模型升级

### 9.2.1 灰度与回滚

模型升级往往同时改变质量、延迟与风格。应灰度发布，保留快速回滚能力。

```java
public class ModelRollout {
    private final Model canary;
    private final Model stable;

    public Model select(double ratio) {
        return ThreadLocalRandom.current().nextDouble() < ratio
                ? canary : stable;
    }
}
```

## 9.3 数据与分块变更

### 9.3.1 召回监控

分块策略、Embedding 模型变更会直接影响召回。应监控 Recall@k 与噪声比例，二者需同时看。

## 9.4 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 只测改动点 | 关联退化漏检 | 全量回归 |
| 无回滚 | 故障无法恢复 | 保留旧版本 |
| 监控单一 | 召回升噪声涨 | 多指标联看 |

> 生产建议：把"变更类型 → 必跑测试集"固化成流程，减少人为遗漏。

# 10. CI 集成

**结论先行：把三层测试接入 CI，按事件触发，形成自动化质量门禁。**

## 10.1 分层触发

| 层级 | 触发事件 | 是否阻断 |
|------|----------|----------|
| 单元 | 每次 push | 阻断 |
| 集成 | 每日 / 合并前 | 阻断 |
| 评测 | PR / 发版前 | 阻断 |

## 10.2 YAML 示例

### 10.2.1 GitHub Actions

```yaml
- name: Unit tests
  run: ./mvnw -q test

- name: Integration tests
  run: ./mvnw -q verify -Pintegration

- name: Eval regression
  if: github.event_name == 'pull_request'
  env:
    OPENAI_API_KEY: ${{ secrets.EVAL_KEY }}
  run: ./mvnw -q exec:java -Dexec.mainClass=EvalRunner
```

### 10.2.2 门禁设置

评测任务应设为必需检查（required check），未通过不得合并；同时设置超时，避免单次抖动长时间占用。

## 10.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 密钥泄露 | 日志打印 Key | 用 Secrets |
| 评测太慢 | PR 卡住 | 抽样/并行 |
| 波动误报 | 频繁失败 | 容差 + 重试 |
| 无缓存 | 重复构建 | 缓存依赖 |

> 生产建议：评测任务应设置超时与重试，避免单次抖动阻断发布。

# 11. 生产注意事项

**结论先行：回归测试落地的关键，是维护基线、设置容差、覆盖安全、接入 CI 且禁止随意降基线。**

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 无回归测试 | 必建 |
| 无基线 | 维护基线 |
| 无容差 | 设合理容差 |
| 单次运行 | 多次取均值 |
| 不测安全 | 注入与越权必测 |
| 不接 CI | 接入门禁 |
| 基线随意降 | 只升不降 |

## 11.2 落地建议

- 三层测试职责清晰：单元测逻辑、集成测链路、评测测质量。
- 基线只升不降，更新需评审留痕。
- 容差基于历史方差统计，避免误报与漏报。
- 安全回归零容忍，硬门禁。
- 评测任务设置超时、重试与抽样策略。

# 12. 小结

## 12.1 核心结论

- 回归测试是防止退化的安全网。
- 三层：单元（Mock）、集成（真实链路）、评测（完整集）。
- 维护基线，只升不降。
- 设合理容差，多次运行取均值。

## 12.2 落地清单

- 注入与越权测试必须纳入回归。
- 接入 CI，形成自动化质量门禁。
