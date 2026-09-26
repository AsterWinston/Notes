# 1. Agent 为什么难调试

Agent（智能体）是由大模型驱动的多步决策系统。它的调用链比普通微服务更长、更不确定，传统调试手段（断点、单步、日志）在 Agent 场景下几乎全部失效，因此必须换一套思路。

## 1.1 五大难点拆解

| 难点 | 说明 | 直接后果 |
|------|------|----------|
| 非确定 | 同样输入不同输出 | 无法稳定复现 |
| 链路长 | 多轮模型 / 工具 / 检索 | 定位成本高 |
| 黑盒 | 看不到模型"想什么" | 只能靠输入输出推断 |
| 环境敏感 | 提示微小变化影响结果 | 改一处坏多处 |
| 无法断点 | 不能单步调试模型 | 传统 IDE 调试失效 |

### 1.1.1 非确定性与黑盒

模型本质是一个概率采样器（sampling）：同样的 Prompt，在 `temperature > 0` 时每次输出都可能不同。这不是 Bug，而是设计特性。它带来两个问题：

- 无法用"输入 A 必然得到输出 B"来写断言；
- 失败往往是间歇性的，测十次过一次，难以稳定观察。

黑盒意味着我们无法查看模型的内部权重或注意力分布，只能通过"它看到了什么"和"它输出了什么"来间接推断。因此**完整记录模型输入**成为调试的第一要务。

### 1.1.2 长链路与工具编排

一次 Agent 请求可能包含：查询改写 → 向量检索 → 模型决策 → 工具调用 → 再检索 → 再决策 → 生成答案。任意一环出错，最终表现都是"答案不对"。若不拆分环节，就无法判断错误发生在哪一步。

### 1.1.3 环境敏感与不可断点

- Prompt 里多一个换行、少一个标点，都可能改变输出；
- 外部工具（搜索、数据库、API）返回内容随时间变化；
- 模型版本、系统提示、检索分块策略任一改动都会影响结果。

同时，模型调用发生在远端，无法像本地代码那样设断点。只能把"断点"转化为**在关键节点打印结构化快照**。

## 1.2 与普通服务调试的对比

| 维度 | 普通微服务 | Agent 系统 |
|------|------------|------------|
| 执行路径 | 基本确定 | 每次可能不同 |
| 复现方式 | 重放请求即可 | 需重放完整上下文与参数 |
| 错误定位 | 堆栈 + 行号 | 轨迹 + 步骤快照 |
| 断点 | IDE 断点 | 日志 / Trace 埋点 |
| 根因 | 代码缺陷 | 提示、检索、模型、工具、控制流 |
| 回归防护 | 单元测试断言 | 评测集 + 相似度 / LLM 评判 |

## 1.3 可观测性是调试的前提

> **可观测性是调试的前提**：没有完整 Trace，Agent 无法调试。

可观测性（observability）为调试提供三样东西：

1. **可见**：每一步的输入输出、耗时、Token 都可查；
2. **可回放**：能用 `traceId` 还原当时的完整执行；
3. **可归因**：能区分失败发生在检索、生成还是工具环节。

缺少任何一样，调试都会退化为"猜"。因此埋点必须在系统设计阶段完成，而不是出问题后再补。

# 2. 调试方法论

调试 Agent 不能靠随机试错，而应遵循一套可重复的流程。

## 2.1 五步法

```
① 复现：能否稳定复现？
② 定位：失败在检索、生成还是工具？
③ 归因：属于哪类失败模式？
④ 修复：改提示/工具/控制流？
⑤ 验证：加评测用例防回归
```

这五步是**顺序依赖**的：不能复现就无法定位，不能定位就无法归因，归因错误则修复无效。

## 2.2 每一步的输入、动作与产物

| 步骤 | 输入 | 动作 | 产物 |
|------|------|------|------|
| 复现 | traceId / 原始请求 | 回放完整轨迹 | 可稳定触发的用例 |
| 定位 | 轨迹各步快照 | 判断失败环节 | 失败环节结论 |
| 归因 | 失败环节 + 现象 | 匹配失败模式 | 根因假设 |
| 修复 | 根因假设 | 单变量改动 | 修复版本 |
| 验证 | 修复版本 + 评测集 | 回归 + 对比 | 通过 / 回滚结论 |

## 2.3 方法论常见误区

| 误区 | 表现 | 正确做法 |
|------|------|----------|
| 跳过复现 | 直接凭感觉改提示 | 先稳定复现 |
| 定位粗放 | 只知"答错了" | 拆分检索 / 生成 / 工具 |
| 多变量同改 | 一次改提示又换模型 | 一次只改一个变量 |
| 只改不测 | 修完就上线 | 加入评测集回归 |
| 忽略环境 | 未记录模型版本 | 记录模型与参数 |

# 3. 复现

复现（reproduction）是整个调试流程的地基。复现的本质是"把当时的环境原样重演一遍"。

## 3.1 回放机制与数据结构

### 3.1.1 Trace 与 Span 的数据模型

回放依赖一份完整的执行记录。最小模型如下：

```java
public record StepSnapshot(
        String name,          // 步骤名：retrieve / plan / tool / generate
        String type,          // 步骤类型
        String input,         // 该步完整输入（含 Prompt）
        String output,        // 该步输出
        long durationMs,      // 耗时
        int promptTokens,     // 输入 Token
        int completionTokens  // 输出 Token
) {}

public record ExecutionReplay(
        String traceId,
        String modelName,          // 模型名
        double temperature,        // 采样温度
        String promptVersion,      // Prompt 版本号
        List<Document> retrievalDocs,
        List<StepSnapshot> steps,
        String finalAnswer,
        String expectedAnswer
) {}
```

### 3.1.2 回放实现

```java
// 用 traceId 完整回放
public void replay(String traceId) {
    ExecutionReplay replay = traceService.replay(traceId);

    System.out.println("模型: " + replay.modelName()
            + " 温度: " + replay.temperature()
            + " Prompt 版本: " + replay.promptVersion());

    replay.steps().forEach(step -> {
        System.out.println("=== " + step.name() + " ===");
        System.out.println("输入: " + step.input());
        System.out.println("输出: " + step.output());
        System.out.println("耗时: " + step.durationMs() + "ms");
        System.out.println("Token: "
                + step.promptTokens() + "+"
                + step.completionTokens());
    });
}
```

### 3.1.3 回放的两种模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| 只读回放 | 仅还原展示轨迹，不重新调用模型 | 定位问题 |
| 重跑回放 | 用当时上下文重新执行 | 验证修复 |

重跑回放时要注意：外部工具（如实时搜索）的结果可能已变化，需要**冻结工具返回**（record & replay）才能保证可比性。

## 3.2 复现四要素

| 要素 | 说明 |
|------|------|
| 完整输入 | 用户问题 + 检索结果 + 历史 |
| 模型参数 | 模型名、temperature、maxTokens |
| Prompt 版本 | 当时使用的版本 |
| 工具结果 | 每个工具返回了什么 |

> 缺任何一项都无法准确复现。

## 3.3 无法复现时的处理

- 若间歇性失败：固定 `temperature=0` 并重复运行多次，统计失败率；
- 若外部依赖漂移：冻结工具返回，改为回放数据；
- 若连 traceId 都没有：只能先补齐埋点，再等待复现——这正是"可观测性优先"的原因。

# 4. 定位：检索还是生成

定位（localization）的目标是把"答案不对"缩小到具体环节。最常见的二分法是：**检索没找到** 还是 **找到但生成错**。

## 4.1 归因判定逻辑

```
                    开始
                     │
                     ▼
         答案是否在检索结果中？
            ┌────────┴────────┐
           否                 是
            │                  │
            ▼                  ▼
     检索失败             生成是否正确？
   （分块/Embedding/          ┌───┴───┐
     查询改写）              否        是
                             │         │
                             ▼         ▼
                        生成失败     正常
                      （提示/模型）
```

## 4.2 完整诊断实现

```java
public DiagnosisResult diagnose(String traceId) {
    ExecutionReplay replay = traceService.replay(traceId);

    List<Document> retrieved = replay.retrievalDocs();
    String answer = replay.finalAnswer();
    String expected = replay.expectedAnswer();

    boolean answerInContext = retrieved.stream()
            .anyMatch(d -> containsAnswer(d.getText(), expected));

    if (!answerInContext) {
        return DiagnosisResult.retrievalFailure(
                "答案不在检索结果中，检查分块/Embedding/查询改写");
    }
    if (!isCorrect(answer, expected)) {
        return DiagnosisResult.generationFailure(
                "答案在上下文中但生成错误，检查提示/模型");
    }
    return DiagnosisResult.ok();
}

// 辅助：判断文档是否包含答案
private boolean containsAnswer(String text, String expected) {
    if (expected == null || expected.isBlank()) {
        return false;
    }
    return text.contains(expected);
}

// 辅助：判断生成答案是否正确（可换 LLM 评判）
private boolean isCorrect(String answer, String expected) {
    return answer != null && answer.contains(expected);
}
```

## 4.3 边界与反例

| 场景 | 二分法结论 | 是否可靠 |
|------|------------|----------|
| 答案需多文档拼接 | 单文档不含答案 → 误判检索失败 | 不可靠，需放宽判定 |
| 生成答案同义改写 | 字符串不匹配 → 误判生成失败 | 不可靠，需语义判定 |
| 检索含噪声 | 含答案但模型忽略 | 实为生成 / 提示问题 |
| 工具链间接获答 | 答案既不在检索也不在生成 | 二分法覆盖不到 |

因此纯字符串判定只能作为**快速初筛**，生产环境应引入语义相似度或 LLM 评判（LLM-as-judge）来提高准确率。

# 5. 常见问题速查

## 5.1 症状对照表

| 症状 | 可能原因 | 排查 |
|------|----------|------|
| 答不出 | 检索未命中 | 看检索结果 |
| 答错 | 提示不清 / 模型弱 | 看模型输入 |
| 编造 | 提示未约束 | 强化"只依据资料" |
| 工具没调用 | 描述不清 | 看工具描述与决策 |
| 工具调用错 | 工具重叠 | 看工具集与参数 |
| 死循环 | 无终止条件 | 看工具调用序列 |
| 很慢 | 某步耗时长 | 看 Span 耗时 |
| 成本高 | 上下文膨胀 | 看 Token 分布 |

## 5.2 高频根因细分

### 5.2.1 检索类问题

- 分块过大 / 过小，语义被切断；
- Embedding（向量化）模型与查询语言不匹配；
- 查询未改写，口语化表达召回差；
- Top-K 太小，正确文档被截断。

### 5.2.2 生成类问题

- 系统提示未约束"仅依据资料"；
- 上下文过长导致"中间遗忘"（lost in the middle）；
- 模型能力不足，指令遵循差。

### 5.2.3 工具类问题

- 工具描述模糊，模型不知道何时调用；
- 多个工具职责重叠，模型选错；
- 参数 Schema 不清晰，模型填错；
- 缺少终止条件，反复调用同一工具。

## 5.3 快速排查流程

```
现象
 │
 ├─ 没有答案？ ──▶ 看检索结果 → 命中率低？改检索
 │
 ├─ 答案错？ ────▶ 看模型输入 → 上下文有答案？改提示
 │
 ├─ 乱调工具？ ──▶ 看工具描述 → 描述清晰？看决策日志
 │
 ├─ 很慢？ ──────▶ 看各步耗时 → 定位慢 Span
 │
 └─ 很贵？ ──────▶ 看 Token 分布 → 压缩上下文
```

# 6. 调试技巧

以下技巧用于在受控条件下观察 Agent 行为。核心原则：**一次只改一个变量**。

## 6.1 打印模型输入

```java
// 临时：把完整提示打到 DEBUG 日志
if (log.isDebugEnabled()) {
    log.debug("模型输入:\n{}", prompt);
}
```

注意生产环境必须对敏感信息脱敏，且不要长期开启全量 Prompt 日志（会显著增加存储成本）。

## 6.2 固定随机性

```java
// 调试时用 temperature=0，减少随机
ChatOptions.builder().temperature(0.0).build();
```

`temperature=0` 并非绝对确定（受浮点与并发影响），但已足以稳定大多数场景。若仍不稳定，可进一步固定 `seed`。

## 6.3 单步执行

```java
// 把 Agent 循环拆开，逐步手动执行
AgentState state = initialState(goal);
state = plannerStep(state);       // 只看规划
System.out.println(state.plan());

state = executeStep(state, 0);    // 只执行第一步
System.out.println(state.toolResults());
```

单步执行把不可断点的循环变成可观察的序列，是定位控制流问题的最有效手段。

## 6.4 对比实验

```java
// 控制变量：只改一个因素
String baseline = chatWithConfig(configA);
String variant  = chatWithConfig(configB);
compare(baseline, variant);
```

对比实验要保证除目标变量外，其余配置（模型、温度、检索结果）完全一致。

## 6.5 补充技巧

### 6.5.1 二分定位

当失败出现在多步链路时，可在中间步骤注入"人工正确结果"：若注入后成功，说明问题在后半段；反之在前半段。逐次二分即可快速收敛。

### 6.5.2 最小复现

把失败用例裁剪到最小：缩短上下文、只保留一个工具、固定单轮对话。最小复现能排除干扰因素，也便于加入评测集。

# 7. 失败案例管理

失败案例（failure case）是调试的产出物，也是评测集的来源。管理得当，每次修复都会提升系统整体质量。

## 7.1 失败案例数据模型

```java
public record FailureCase(
        String id,
        String traceId,
        String userQuery,
        String expected,
        String actual,
        boolean reproducible,   // 是否可稳定复现
        FailureType type        // 归因类型
) {
    public FailureCase withTrace(ExecutionReplay replay) {
        return this; // 附加完整轨迹后返回
    }

    public EvalCase toEvalCase() {
        return new EvalCase(userQuery, expected);
    }
}
```

## 7.2 记录与回流实现

```java
public class FailureCaseManager {

    public void record(FailureCase c) {
        // ① 保存完整上下文
        failureStore.save(c.withTrace(
                traceService.fullTrace(c.traceId())));

        // ② 归因分类
        FailureType type = classifier.classify(c);
        failureStore.tag(c.id(), type);

        // ③ 回流评测集
        if (c.isReproducible()) {
            evalSet.add(c.toEvalCase());
        }
    }
}
```

## 7.3 失败模式分类

| 失败模式 | 典型现象 | 修复方向 |
|----------|----------|----------|
| 检索失败 | 上下文无答案 | 分块 / Embedding / 查询改写 |
| 生成失败 | 有答案却答错 | 提示 / 模型 |
| 工具选择失败 | 未调用或调错工具 | 工具描述 / 拆分 |
| 参数失败 | 工具参数错误 | Schema / 示例 |
| 控制流失败 | 死循环 / 提前终止 | 终止条件 / 步数上限 |
| 幻觉 | 编造不存在内容 | 约束提示 / 引用校验 |

# 8. 调试工具清单

## 8.1 工具矩阵

| 工具 | 用途 | 类型 |
|------|------|------|
| Langfuse / LangSmith | 查看完整 Trace | LLM 专用 |
| Jaeger / Tempo | 分布式追踪 | 通用 |
| 日志平台（ELK） | 检索日志 | 通用 |
| 评测平台 | 对比效果 | LLM 专用 |
| 本地复现脚本 | 单步调试 | 自建 |

## 8.2 自建调试端点

在没有现成平台时，可先自建一个最小的调试接口：

```java
@RestController
@RequestMapping("/debug")
public class DebugController {

    private final TraceService traceService;

    public DebugController(TraceService traceService) {
        this.traceService = traceService;
    }

    @GetMapping("/replay/{traceId}")
    public ExecutionReplay replay(@PathVariable String traceId) {
        return traceService.replay(traceId);
    }
}
```

对应配置（仅在内网环境开启）：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
debug:
  replay:
    enabled: true        # 生产环境务必关闭或加鉴权
    max-steps: 200       # 防止超大轨迹拖垮服务
```

# 9. 调试检查清单

## 9.1 排查清单

- [ ] 能否用 traceId 完整回放？
- [ ] 模型看到了什么（完整提示）？
- [ ] 检索命中了哪些文档、分数多少？
- [ ] 模型决策了什么（工具/答案）？
- [ ] 工具返回了什么？
- [ ] 各步耗时多少？
- [ ] Token 消耗分布如何？
- [ ] 用了哪个 Prompt 版本？

## 9.2 上线前与回归检查

- [ ] 失败用例是否已回流评测集？
- [ ] 修复是否只改了一个变量？
- [ ] 回归评测是否通过？
- [ ] 敏感信息是否已脱敏？
- [ ] 调试端点是否已关闭或加鉴权？

# 10. 生产注意事项

## 10.1 风险与建议

| 风险 | 建议 |
|------|------|
| 无 Trace | 无法复现，必须埋点 |
| 记录不完整 | 保留完整上下文 |
| 无 Prompt 版本 | 记录版本号 |
| 失败不回流 | 进评测集 |
| 只改不测 | 加回归用例 |
| 多变量同改 | 一次只改一个 |

## 10.2 调试能力的反模式

| 反模式 | 危害 |
|--------|------|
| 全量记录 Prompt 明文 | 泄露敏感数据、存储爆炸 |
| 只在出错时记日志 | 正常轨迹缺失，无法对比 |
| traceId 不贯穿全链路 | 跨服务无法拼接 |
| 采样率过高 / 过低 | 要么丢样本，要么成本失控 |
| 调试开关长期开启 | 性能与安全风险 |

# 11. 小结

- 可观测性是 Agent 调试的前提。
- 方法论：复现 → 定位（检索/生成）→ 归因 → 修复 → 验证。
- 复现需要完整输入、参数、Prompt 版本、工具结果。
- 定位靠"答案是否在上下文"做快速二分，但需语义判定兜底。
- 失败案例必须回流为评测用例。
- 调试时一次只改一个变量。
