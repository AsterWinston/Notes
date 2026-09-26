# 1. 为什么单独评测工具调用

工具调用（Tool Calling）是 Agent 的**关键能力**，也是**高频失败点**。**结论先行：只评测最终答案会掩盖过程错误，工具调用必须作为独立评测对象单独度量。**

## 1.1 工具调用的失败模式

### 1.1.1 六类典型失败

```
- 该调用工具却没调用（漏调，Miss）
- 不该调用却调用了（误调，Spurious）
- 调用了错误的工具（错选，Wrong Selection）
- 参数错误（参数，Param Error）
- 重复调用（冗余，Redundant）
- 该停止却继续（不收敛，No Convergence）
```

### 1.1.2 失败模式与指标映射

每种失败模式都需要一个可计算的指标来度量，否则"评测"只是一句口号：

| 失败模式 | 对应指标 | 度量方式 |
|----------|----------|----------|
| 漏调 | 召回率（Recall） | 期望动作被调用的比例 |
| 误调 | 精确率（Precision） | 实际动作在期望集合中的比例 |
| 错选 | 工具选择准确率 | 集合是否完全相等 |
| 参数错误 | 参数准确率 | 参数键值是否匹配 |
| 冗余 | 冗余率 | 重复调用次数 / 总调用次数 |
| 不收敛 | 超步率 | 达到 maxSteps 的比例 |

## 1.2 为什么通用评测不够

### 1.2.1 结果正确可能掩盖过程错误

一个 Agent 可能"答对了"，但过程里误调了 `deleteOrder`、把 `refundOrder` 当作查询使用。这类**过程性错误**在只比对最终答案的评测中完全不可见，却可能在生产环境造成真实损失。

### 1.2.2 Trace 是可观测性的前提

要评测工具调用，Agent 必须暴露**调用轨迹（Trace）**，即每一步的工具名、参数、状态、耗时。评测代码依赖 `runWithTrace` 这类接口拿到逐步信息：

```java
public interface Assistant {
    AgentTrace runWithTrace(String goal);
}

public record Step(String toolName, String arguments,
                   String status, long latencyMs) { }

public record AgentTrace(List<Step> steps, AgentStatus finalState) { }
```

# 2. 评测维度与指标体系

**结论先行：工具调用评测有六个维度——选择、参数、时机、冗余、恢复、多轮；六者共同构成一张完整的过程质量画像。**

## 2.1 六大维度

### 2.1.1 维度定义

| 维度 | 指标 | 说明 |
|------|------|------|
| 工具选择准确率 | Selection Accuracy | 选对工具的比例 |
| 参数准确率 | Param Accuracy | 参数正确的比例 |
| 调用时机 | Timing | 该调时调、该停时停 |
| 冗余率 | Redundancy | 重复调用比例 |
| 错误恢复 | Recovery | 失败后能否纠正 |
| 多轮准确性 | Multi-step Accuracy | 多步调用的整体正确性 |

### 2.1.2 维度之间的关系

六个维度并非彼此独立。选择准确率高，但参数准确率低，说明模型"知道用什么"却"不会填参数"；时机与冗余则共同描述"调用节奏"是否合理。评测报告应同时呈现，避免单指标误导。

## 2.2 指标的计算口径

### 2.2.1 集合比较与序列比较

| 口径 | 适用场景 | 特点 |
|------|----------|------|
| 集合比较 | 不关心顺序 | 宽松，易高分 |
| 序列比较 | 关心调用顺序 | 严格，贴近真实 |
| 子序列匹配 | 允许插入额外步骤 | 折中 |

### 2.2.2 参数匹配的粒度

参数匹配分全等、子集、模糊三种粒度，生产中常用**子集 + 归一化**。

# 3. 工具选择准确率

**结论先行：工具选择准确率衡量"选得对不对"，最直接的口径是期望工具集合与实际工具集合是否相等。**

## 3.1 定义与公式

设第 i 个用例的期望工具集合为 $E_i$，实际调用集合为 $A_i$，则：

```
SelectionAccuracy = (1/N) * Σ 1[ set(A_i) == set(E_i) ]
```

若使用精确率与召回率，则分别定义为：

```
Precision = |A ∩ E| / |A|
Recall    = |A ∩ E| / |E|
F1        = 2 * P * R / (P + R)
```

## 3.2 Java 实现

### 3.2.1 基础集合比较

```java
public double toolSelectionAccuracy(List<EvalCase> cases) {
    int correct = 0, total = 0;

    for (EvalCase c : cases) {
        List<String> expectedTools = c.expectedActions();
        List<String> actualTools = agent.runWithTrace(c.goal())
                .steps().stream().map(Step::toolName).toList();

        // 集合比较（不关心顺序）
        if (new HashSet<>(actualTools).equals(new HashSet<>(expectedTools))) {
            correct++;
        }
        total++;
    }
    return correct / (double) total;
}
```

### 3.2.2 序列比较与精确率

若关心调用顺序，改用序列比较；若关心单工具表现，用逐工具精确率/召回率替代集合全等。

```java
public double sequenceAccuracy(EvalCase c, Assistant agent) {
    List<String> actual = agent.runWithTrace(c.goal()).steps()
            .stream().map(Step::toolName).toList();
    return actual.equals(c.expectedActions()) ? 1.0 : 0.0;
}
```

## 3.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 顺序被忽略 | 集合比较不区分先后 | 关键流程用序列比较 |
| 多调即失败 | 多调一个无害工具也算错 | 引入"允许集合" |
| 用例不平衡 | 简单工具样本过多 | 按工具分层统计 |
| 别名问题 | 同一工具不同命名 | 统一工具注册表 |

> 生产建议：选择准确率单独看意义有限，应与参数准确率、时机一起看。

# 4. 参数准确率

**结论先行：工具选对不代表用对，参数准确率衡量"参数填得对不对"，是工具调用质量的第二道关卡。**

## 4.1 定义与公式

对每个期望调用（工具 + 参数），判断轨迹中是否存在一次调用，其工具名相同且期望参数均被满足：

```
ParamAccuracy = (1/M) * Σ 1[ ∃ step: name(step)=tool_k ∧ params(step) ⊇ params_k ]
```

其中 M 为所有用例期望调用的总数。

## 4.2 Java 实现

### 4.2.1 基础实现

```java
public double paramAccuracy(List<EvalCase> cases) {
    int correct = 0, total = 0;

    for (EvalCase c : cases) {
        List<Step> steps = agent.runWithTrace(c.goal()).steps();

        for (ExpectedCall expected : c.expectedCalls()) {
            total++;
            boolean match = steps.stream().anyMatch(s ->
                    s.toolName().equals(expected.tool())
                    && paramsMatch(s.arguments(), expected.params()));
            if (match) correct++;
        }
    }
    return correct / (double) total;
}

private boolean paramsMatch(String actualJson, Map<String, Object> expected) {
    Map<String, Object> actual = parseJson(actualJson);
    return expected.entrySet().stream()
            .allMatch(e -> Objects.equals(actual.get(e.getKey()), e.getValue()));
}
```

### 4.2.2 递归与归一化匹配

真实参数常为嵌套结构，且存在数值/大小写差异，需要递归与归一化：

```java
private boolean deepMatch(Object actual, Object expected) {
    if (expected instanceof Map<?, ?> em && actual instanceof Map<?, ?> am) {
        return em.entrySet().stream().allMatch(e ->
                am.containsKey(e.getKey())
                        && deepMatch(am.get(e.getKey()), e.getValue()));
    }
    if (expected instanceof Number en && actual instanceof Number an) {
        // 数值容差：避免 1 与 1.0 被判不等
        return Math.abs(en.doubleValue() - an.doubleValue()) < 1e-6;
    }
    if (expected instanceof String es && actual instanceof String as) {
        return es.trim().equalsIgnoreCase(as.trim());
    }
    return Objects.equals(actual, expected);
}
```

## 4.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 严格全等 | 多填字段判错 | 用子集匹配 |
| 类型不一致 | `1` 与 `"1"` | 归一化类型 |
| 大小写/空格 | `"A123 "` vs `"A123"` | trim + 忽略大小写 |
| 动态参数 | 时间戳、UUID 无法预知 | 用正则或占位符 |
| 参数顺序 | JSON 键序不同 | 解析为 Map 后比较 |

> 生产建议：为动态参数引入**模式占位符**（如 `"orderId": "<ANY>"`），否则用例无法复用。

# 5. 调用时机评测

**结论先行：调用时机衡量"该调时调、该停时停"，核心是三类偏差——漏调、误调、不收敛。**

## 5.1 三类时机偏差

```
漏调  ：期望动作缺失
误调  ：出现禁止动作
不收敛：步数达到上限仍未结束
```

## 5.2 Java 实现

### 5.2.1 TimingMetrics

```java
public record TimingMetrics(int missedCalls,
                            int unnecessaryCalls,
                            int lateStop) { }

public TimingMetrics evaluateTiming(List<EvalCase> cases) {
    int missedCalls = 0, unnecessaryCalls = 0, lateStop = 0;

    for (EvalCase c : cases) {
        List<Step> steps = agent.runWithTrace(c.goal()).steps();

        // 漏调：期望调用但未调用
        for (String expected : c.expectedActions()) {
            if (steps.stream().noneMatch(s -> s.toolName().equals(expected))) {
                missedCalls++;
            }
        }

        // 误调：不应调用但调用了
        for (Step step : steps) {
            if (c.forbiddenActions().contains(step.toolName())) {
                unnecessaryCalls++;
            }
        }

        // 不收敛：达到最大步数仍未结束
        if (steps.size() >= c.maxSteps()) {
            lateStop++;
        }
    }
    return new TimingMetrics(missedCalls, unnecessaryCalls, lateStop);
}
```

### 5.2.2 逐步时机匹配

若关心"第几步该调用什么"，可将期望动作带上步序号：

```java
public boolean timingOrderMatches(List<Step> actual,
                                  List<ExpectedAction> expected) {
    int cursor = 0;
    for (Step step : actual) {
        if (cursor < expected.size()
                && expected.get(cursor).tool().equals(step.toolName())) {
            cursor++;
        }
    }
    return cursor == expected.size();
}
```

## 5.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 把重试算误调 | 失败重试是合理的 | 区分状态与工具 |
| 并行调用 | 同一轮多个工具 | 按轮次而非步数计 |
| 上限过小 | 正常任务被误判不收敛 | 按任务类型设上限 |
| 禁止动作过宽 | 合理工具被禁 | 用例评审 |

> 生产建议：漏调与误调的代价不同，**禁止动作（如删除、退款）应视为硬性失败**，权重高于漏调。

# 6. 错误恢复评测

**结论先行：真实工具会失败，Agent 能否在失败后自我纠正，是区别于"一次性问答"的核心能力。**

## 6.1 失败注入机制

错误恢复评测的关键是**可控地制造失败**。常用手段：Mock 工具、故障注入代理、超时开关。

| 注入方式 | 说明 | 适用 |
|----------|------|------|
| Mock 首次失败 | 第一次调用抛错 | 重试恢复 |
| 间歇失败 | 按概率失败 | 鲁棒性 |
| 超时注入 | 强制延迟 | 超时处理 |
| 参数校验失败 | 返回错误码 | 参数自纠 |

## 6.2 Java 实现

### 6.2.1 Mock 工具失败

```java
public void evaluateErrorRecovery(Assistant agent) {
    // 场景：第一次调用参数错误，Agent 应自我纠正
    mockTool.failFirstWith("参数格式错误");

    AgentTrace trace = agent.runWithTrace("查询订单 A123");

    // 期望：失败后调整参数重试成功
    long failures = trace.steps().stream()
            .filter(s -> "FAILED".equals(s.status())).count();
    boolean recovered = trace.finalState() == AgentStatus.COMPLETED;

    assertTrue(failures > 0, "应出现失败");
    assertTrue(recovered, "应能从失败中恢复");
}
```

### 6.2.2 恢复率指标

```java
public double recoveryRate(List<EvalCase> cases) {
    int recovered = 0, failing = 0;
    for (EvalCase c : cases) {
        injectFailure(c.failureMode());
        AgentTrace trace = agent.runWithTrace(c.goal());
        boolean hadFailure = trace.steps().stream()
                .anyMatch(s -> "FAILED".equals(s.status()));
        if (!hadFailure) continue;
        failing++;
        if (trace.finalState() == AgentStatus.COMPLETED) recovered++;
    }
    return failing == 0 ? 1.0 : recovered / (double) failing;
}
```

## 6.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 无失败样本 | 全是理想路径 | 专门注入失败 |
| 无限重试 | 死循环 | 设重试上限 |
| 掩盖错误 | 重试成功但逻辑错 | 校验最终结果 |
| 失败信息不足 | 模型无法纠正 | 提供结构化错误 |

> 生产建议：错误恢复率应作为发版门禁指标之一，低于阈值直接阻断。

# 7. 多轮调用准确性

**结论先行：多步任务的整体正确性，不能简单等于每步独立正确，需要按序列/覆盖度整体评估。**

## 7.1 序列匹配口径

多轮准确性有两层含义：**每一步都在期望集合内**（无多余），且**期望集合被完整覆盖**（无遗漏）。

## 7.2 Java 实现

### 7.2.1 覆盖式匹配

```java
public double multiStepAccuracy(List<EvalCase> cases) {
    int correct = 0;

    for (EvalCase c : cases) {
        AgentTrace trace = agent.runWithTrace(c.goal());

        // 检查每一步是否在期望序列中
        boolean allStepsExpected = trace.steps().stream()
                .allMatch(s -> c.expectedActions().contains(s.toolName()));

        // 检查是否覆盖了所有期望步骤
        Set<String> actual = trace.steps().stream()
                .map(Step::toolName).collect(Collectors.toSet());
        boolean allExpected = actual.containsAll(c.expectedActions());

        if (allStepsExpected && allExpected) correct++;
    }
    return correct / (double) cases.size();
}
```

### 7.2.2 失败归因

当整体失败时，用编辑距离（Levenshtein Distance）定位"多了哪步、少了哪步"，把误差归因到具体步骤，便于针对性修复。

## 7.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 顺序敏感 | 合理换序被判错 | 关键流程才严格 |
| 等价路径 | 多种解法都正确 | 标注可接受集合 |
| 长任务误差累积 | 步数越多越易错 | 分阶段统计 |
| 中间步无校验 | 只看最终 | 逐步归因 |

> 生产建议：对长任务按**阶段**（如"查询 → 计算 → 提交"）分段统计，快速定位退化环节。

# 8. 评测用例设计

**结论先行：用例是评测的燃料，字段设计决定评测能发现什么。**

## 8.1 用例结构

```json
{
  "id": "tool-001",
  "goal": "查询订单 A123 的物流并催单",
  "expectedActions": ["queryOrder", "queryLogistics", "urgeOrder"],
  "forbiddenActions": ["deleteOrder", "refundOrder"],
  "expectedCalls": [
    {"tool": "queryOrder", "params": {"orderId": "A123"}},
    {"tool": "urgeOrder", "params": {"waybill": "SF123"}}
  ],
  "maxSteps": 6
}
```

| 字段 | 用途 |
|------|------|
| `expectedActions` | 期望调用的工具 |
| `forbiddenActions` | 禁止调用的工具 |
| `expectedCalls` | 期望的调用与参数 |
| `maxSteps` | 步数上限 |

## 8.2 用例分布与难度

### 8.2.1 正例、反例与边界

| 类型 | 说明 | 示例 |
|------|------|------|
| 正例 | 正常应调用 | 查物流 |
| 反例 | 应拒绝调用 | 要求删除他人订单 |
| 边界 | 易混淆工具 | 查物流 vs 查订单 |
| 对抗 | 诱导误调 | "顺便帮我退款" |

### 8.2.2 难度分层

用例应包含简单（单工具）、中等（2-3 步）、困难（多步 + 干扰）三档，避免整体分数被简单用例抬高。

## 8.3 标注规范

- 每个用例至少一名标注者，关键用例双人复核。
- 期望参数使用占位符处理动态值。
- 禁止动作需有业务依据，不可随意扩大。

# 9. 工具描述质量评测

**结论先行：工具描述（Description）直接影响模型选工具的准确率，可通过对照实验量化其贡献。**

## 9.1 描述与准确率的关系

工具描述是模型"认识工具"的唯一入口。模糊描述导致错选，缺少边界说明导致误调。**描述质量可以像超参数一样被调优。**

## 9.2 Java 实现

### 9.2.1 对照实验

```java
// 换不同的工具描述，看调用准确率变化
public void evaluateToolDescriptions(List<EvalCase> cases) {
    List<String> descriptions = List.of(
            "查询订单",                          // 模糊
            "查询订单详情，包括状态、金额",        // 清晰
            "查询订单详情。适用：用户问订单状态时。不适用：查询物流"  // 明确边界
    );

    for (String desc : descriptions) {
        toolRegistry.updateDescription("queryOrder", desc);
        double accuracy = toolSelectionAccuracy(cases);
        log.info("描述「{}」准确率: {}", desc, accuracy);
    }
}
```

> 单次实验受模型随机性影响，应多次运行取均值与标准差。这是优化工具描述的有效方法。

## 9.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 只测一次 | 随机性掩盖差异 | 多次取均值 |
| 描述过长 | 挤占上下文 | 精炼 + 边界 |
| 描述与实现不符 | 误导模型 | 描述即契约 |
| 只改描述不回归 | 修好 A 坏 B | 全量用例回归 |

> 生产建议：把工具描述纳入版本管理与回归测试，任何描述改动都跑一遍选择准确率。

# 10. 生产注意事项

**结论先行：工具调用评测落地的关键，是把过程指标接入 CI 并设为门禁。**

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 只测最终结果 | 单独测工具调用 |
| 无期望动作标注 | 用例标注 |
| 不测参数 | 校验参数正确性 |
| 不测时机 | 测漏调与误调 |
| 不测恢复 | 模拟失败场景 |
| 不测禁止动作 | 标注 forbiddenActions |

## 10.2 落地建议

- 建立 Trace 可观测性，是一切过程评测的前提。
- 把选择、参数、时机、恢复四项纳入发版门禁。
- 对禁止动作设置硬失败，零容忍。

# 11. 小结

## 11.1 核心结论

- 工具调用是 Agent 的关键能力与高频失败点。
- 维度：选择、参数、时机、冗余、恢复、多轮。
- 用例要标注期望动作、禁止动作、期望参数。
- 错误恢复能力必须专门测试。
- 通过对比实验优化工具描述。

## 11.2 落地清单

- 过程指标应接入 CI，禁止动作零容忍。
- 工具描述改动触发全量回归。
