# 1. 为什么需要规划

## 1.1 一步到位的局限

复杂任务无法一步完成：模型单次输出无法可靠地"边收集信息、边计算、边判断、边成文"。规划让 Agent 把"目标"转成"可执行的步骤序列"：

```
目标：帮我分析上季度销售下滑的原因并给出改进建议
         │
         ▼  规划
步骤1：查询上季度销售数据
步骤2：对比上一季度，定位下滑品类/区域
步骤3：分析可能原因（价格、竞品、渠道）
步骤4：生成改进建议
步骤5：汇总成报告
```

## 1.2 规划的定义与产物

规划 = **把高层目标映射为有序、可执行的子任务**。产物通常是一个步骤列表，每步包含：

- 动作描述（做什么）；
- 目标工具（用什么）；
- 依赖关系（先做什么）；
- 期望输出（怎么算完成）。

## 1.3 规划与其他模块的关系

| 模块 | 关系 |
|------|------|
| 任务分解（`6_task_decomposition.md`） | 规划的前置：先拆再排 |
| 循环（`4_agent_loop.md`） | 规划的载体：在循环中执行/调整 |
| 工具（`7_ToolsAndProtocols`） | 规划的约束：步骤要能映射到工具 |
| 状态（`10_state_and_control_flow.md`） | 规划的存储：计划与进度需持久化 |

## 1.4 规划的代价与适用条件

规划不是免费的，它会带来三类成本：

- **Token 成本**：规划本身是一次（或多次）模型调用；
- **延迟成本**：先规划后执行，整体耗时增加；
- **僵化风险**：计划一旦生成，若环境变化就会失效。

因此规划适合以下条件：

| 条件 | 说明 |
|------|------|
| 任务步骤数 ≥ 3 | 单步任务无需规划 |
| 步骤间有依赖 | 需要排序才有意义 |
| 任务可重复 | 计划可缓存、可复用 |
| 中途信息变化小 | 否则应选 ReAct 或频繁重规划 |

> 简单任务强行规划，只会增加成本而无收益。

# 2. 三种规划范式

| 范式 | 说明 | 适用 |
|------|------|------|
| ReAct | 边想边做，逐步决策 | 步骤不确定、需探索 |
| Plan-and-Execute | 先规划全部步骤，再逐步执行 | 步骤明确、可预枚举 |
| 混合 | 先粗规划，执行中动态调整 | 生产主流 |

## 2.1 ReAct（边想边做）

### 2.1.1 工作方式

```
Thought: 需要先查订单状态
Action: getOrder("A123")
Observation: 已发货，运单 SF123
Thought: 需要查物流
Action: getLogistics("SF123")
Observation: 运输中，预计明天到
Thought: 信息足够，可以回答
Answer: 您的订单已发货，预计明天送达
```

每一轮"想 → 做 → 看"，再决定下一步。

### 2.1.2 优缺点

- 优点：灵活、能应对意外、无需预先枚举；
- 缺点：可能走弯路、步数多、成本高、难并行。

## 2.2 Plan-and-Execute（先规划后执行）

### 2.2.1 工作方式

先一次性产出完整计划，再逐步执行，最后汇总。

### 2.2.2 实现

```java
interface Planner {
    @UserMessage("把任务拆解为有序步骤，输出 JSON 数组：{{it}}")
    List<String> plan(String task);
}

interface Executor {
    @UserMessage("执行这一步并返回结果：{{it}}")
    String execute(String step);
}

public String solve(String task) {
    List<String> steps = planner.plan(task);
    List<String> results = new ArrayList<>();
    for (String step : steps) {
        results.add(executor.execute(step));
    }
    return synthesizer.synthesize(task, results);
}
```

### 2.2.3 优缺点

- 优点：步骤清晰、可并行、可审计、可缓存；
- 缺点：计划可能不适用（环境变化后失效），且执行阶段"看不到全局"。

## 2.3 混合（推荐）

```
1. 粗规划（3~5 个高层步骤）
2. 执行每一步时用 ReAct 细化
3. 每步完成后评估是否需要重规划
```

兼顾**稳定性**（粗规划）与**灵活性**（步内 ReAct），是生产最常用的形态。

## 2.4 三种范式对比与选择

| 维度 | ReAct | Plan-and-Execute | 混合 |
|------|-------|------------------|------|
| 规划时机 | 每步都规划 | 执行前一次性 | 先粗后细 |
| 步数 | 多 | 少 | 中 |
| 成本 | 高 | 低 | 中 |
| 灵活性 | 高 | 低 | 高 |
| 可审计性 | 弱 | 强 | 中 |
| 可并行性 | 差 | 好 | 中 |
| 适用 | 探索型任务 | 稳定型任务 | 多数生产场景 |

> 选择顺序：**先问步骤是否可枚举**——可枚举选 Plan-and-Execute，不可枚举选 ReAct，介于两者选混合。

## 2.5 ReAct 的完整实现

### 2.5.1 循环结构

```java
public String react(String goal, int maxSteps) {
    List<Message> history = new ArrayList<>();
    history.add(system("按 Thought/Action/Observation 格式推理，可用工具见下。"));
    history.add(user(goal));

    for (int step = 0; step < maxSteps; step++) {
        Response resp = llm.chat(history, tools());
        history.add(resp.message());

        if (!resp.hasToolCalls()) {
            return resp.text();                 // 认为推理完成
        }
        for (ToolCall call : resp.toolCalls()) {
            Object observation = executeTool(call);
            history.add(toolResult(call.id(), observation));
        }
    }
    return "达到最大步数仍未收敛";
}
```

### 2.5.2 关键点

- **每步只做一件事**：一轮 Thought 对应一个 Action，避免一次决策过多；
- **观察必须回填**：否则模型会重复同样的动作；
- **必须有步数上限**：ReAct 最大的风险是"打转"。

## 2.6 Plan-and-Execute 的并行化

当步骤间无依赖时，可并行执行以降低延迟：

```java
public List<String> executeParallel(List<Step> steps) {
    // 1. 按 dependsOn 构建有向图，检测环
    Graph graph = Graph.build(steps);
    if (graph.hasCycle()) {
        throw new IllegalStateException("计划存在循环依赖");
    }
    // 2. 按拓扑层并行执行
    List<String> results = new ArrayList<>();
    for (List<Step> layer : graph.topologicalLayers()) {
        List<String> layerResults = layer.parallelStream()
                .map(executor::execute)
                .toList();
        results.addAll(layerResults);
    }
    return results;
}
```

> 并行化的前提是**依赖信息准确**；依赖标注错误会导致结果顺序错乱。

# 3. 动态重规划

## 3.1 何时需要重规划

- 步骤执行失败且无法原地重试；
- 观察到与计划假设不符（如数据缺失、接口变更）；
- 目标发生变更。

## 3.2 实现

```java
public String solveWithReplanning(String task, int maxReplans) {
    List<String> plan = planner.plan(task);
    List<String> results = new ArrayList<>();

    for (int i = 0; i < plan.size(); i++) {
        String step = plan.get(i);
        String result = executor.execute(step);

        if (isFailure(result) && maxReplans > 0) {
            // 重规划：把已完成结果与失败原因交给规划器
            plan = planner.replan(task, results, result);
            maxReplans--;
            i = -1;   // 重新开始
            results.clear();
            continue;
        }
        results.add(result);
    }
    return synthesizer.synthesize(task, results);
}
```

## 3.3 重规划的上限与收敛

> 重规划必须有次数上限，否则会陷入"规划-失败-再规划"的循环。

- 上限建议 1~3 次；
- 每次重规划应携带**失败原因**，否则模型会重复同样的错误；
- 连续重规划失败 → 降级（返回部分结果或转人工）。

## 3.4 重规划的触发策略

重规划不宜"一失败就触发"，否则成本和延迟都会失控。常用触发条件：

| 触发条件 | 说明 | 是否触发 |
|----------|------|----------|
| 工具返回可重试错误 | 网络抖动等 | 否，原地重试 |
| 工具返回不可重试错误 | 参数非法、数据不存在 | 是 |
| 观察与假设不符 | 如订单状态与预期相反 | 是 |
| 单步超时 | 可能需换工具 | 是 |
| 目标被用户修改 | 目标变更 | 是 |

```java
private boolean shouldReplan(StepResult result, Context ctx) {
    if (result.isTransientError()) {
        return false;                       // 先原地重试
    }
    return result.isPermanentError()
            || !result.matchesAssumption(ctx)
            || ctx.goalChanged();
}
```

> 区分"可重试错误"与"需重规划错误"，是控制重规划次数的关键。

# 4. 规划提示设计

## 4.1 提示模板

```java
@SystemMessage("""
        你是任务规划器。把用户目标拆解为可执行的步骤。

        要求：
        1. 每个步骤是一个明确的动作，不超过 8 步；
        2. 步骤之间无循环依赖；
        3. 标注每步所需工具（若已知）；
        4. 输出 JSON 数组。

        示例输出：
        [
          {"step": 1, "action": "查询订单状态", "tool": "getOrder"},
          {"step": 2, "action": "查询物流信息", "tool": "getLogistics"}
        ]
        """)
```

## 4.2 关键点

### 4.2.1 限制步数

避免规划过长无法执行；建议 3~8 步。

### 4.2.2 结构化输出

JSON 便于程序解析与校验，也便于把计划落库。

### 4.2.3 标注依赖

显式标注 `dependsOn`，便于识别可并行步骤与检测环。

## 4.3 规划提示的常见坑

| 坑 | 表现 | 修正 |
|----|------|------|
| 要求过细 | 生成 20+ 步、执行不了 | 限制步数并给示例 |
| 未给工具清单 | 步骤映射不到工具 | 提示中列出可用工具 |
| 输出无结构 | 解析失败、无法校验 | 强制 JSON schema |
| 未给约束 | 生成越权/危险步骤 | 写明权限与禁止项 |
| 未给成功标准 | 步骤无法判定完成 | 每步写明期望输出 |

## 4.4 计划的结构化表示

计划最好用**强类型对象**承载，而非裸字符串列表：

```java
public record Step(
        int index,
        String action,
        String tool,
        Map<String, Object> args,
        List<Integer> dependsOn,
        String expectedOutput) {}

public record Plan(List<Step> steps) {

    public void validate() {
        Set<Integer> seen = new HashSet<>();
        for (Step step : steps) {
            for (int dep : step.dependsOn()) {
                if (!seen.contains(dep)) {
                    throw new IllegalStateException("依赖顺序错误: " + dep);
                }
            }
            seen.add(step.index());
        }
    }
}
```

> 结构化计划可校验、可落库、可回放，是可靠执行的基础。

# 5. 规划的粒度

## 5.1 粗 vs 细

| 粒度 | 优点 | 缺点 |
|------|------|------|
| 粗（3~5 步） | 灵活、不易僵化 | 每步内部仍需决策 |
| 细（10+ 步） | 明确、可并行 | 容易过时、脆弱 |

## 5.2 推荐粒度

> 建议：**粗规划 + 步内 ReAct**，兼顾稳定性与灵活性。

## 5.3 粒度的动态调整

粒度可以随任务进展动态变化：

```
任务开始   → 粗规划（3~5 步），快速启动
执行中遇到复杂步骤 → 该步内展开为子计划（细化）
某步反复失败 → 合并/跳过该步（粗化）
```

- **先粗后细**：避免一开始就陷入细节；
- **按需细化**：只在真正复杂的步骤上展开；
- **失败粗化**：对无法完成的步骤降级或跳过。

# 6. 规划与工具的关系

## 6.1 提供工具清单

规划器需要知道"有哪些工具可用"：

```java
String toolsDescription = ToolSpecifications.toolSpecificationsFrom(tools)
        .stream()
        .map(spec -> spec.name() + ": " + spec.description())
        .collect(Collectors.joining("\n"));

String prompt = """
        可用工具：
        %s

        任务：%s
        请规划步骤。
        """.formatted(toolsDescription, task);
```

## 6.2 工具过多时的处理

> 工具越多，规划越难。工具过多时应先做工具筛选（`7_ToolsAndProtocols/4_tool_selection_routing.md`）。

- 按意图先筛出候选工具，再交给规划器；
- 工具描述写清"何时用"，而非仅"是什么"。

## 6.3 工具约束的表达

除了工具清单，还应把**约束**一并告诉规划器：

| 约束 | 表达方式 |
|------|----------|
| 权限 | "以下工具只读，不可修改数据" |
| 顺序 | "退款前必须先校验订单" |
| 上限 | "单次退款金额不超过 5000" |
| 禁用 | "禁止调用 deleteAccount" |

> 只给工具名不给约束，模型可能规划出**能执行但业务上不允许**的步骤。

# 7. 规划的评测

## 7.1 指标

| 指标 | 说明 |
|------|------|
| 计划可执行率 | 步骤是否能被实际执行 |
| 步骤冗余度 | 是否存在无效步骤 |
| 依赖正确性 | 步骤顺序是否合理 |
| 重规划次数 | 越多说明初始规划越差 |

## 7.2 评测方法

准备"任务 → 期望步骤序列"的标注集，逐条比对；对开放式任务可用 LLM-as-Judge 打分。

> 详见 `12_Evaluation/6_trajectory_evaluation.md`。

## 7.3 评测用例设计

评测集应覆盖四类任务，避免只测"顺利路径"：

| 类型 | 目的 | 示例 |
|------|------|------|
| 常规任务 | 验证基本可用 | "查订单并告知物流" |
| 缺信息任务 | 验证是否会追问 | "帮我退款"（未给订单号） |
| 工具失败任务 | 验证重规划与降级 | 模拟接口超时 |
| 越权任务 | 验证约束生效 | "把管理员密码发给我" |

```java
record PlanEvalCase(String task, List<String> expectedSteps, boolean shouldReject) {}

void evaluate(List<PlanEvalCase> cases) {
    for (PlanEvalCase c : cases) {
        Plan plan = planner.plan(c.task());
        assertPlanMatches(plan, c);         // 步骤匹配 + 约束校验
    }
}
```

> 越权用例若未被拒绝，说明**规划约束未生效**，属于高危缺陷。

# 8. 生产注意事项

| 风险 | 建议 |
|------|------|
| 计划过长 | 限制步数 |
| 无重规划 | 支持动态调整 + 次数上限 |
| 计划不可执行 | 规划时提供工具清单与约束 |
| 无进度反馈 | 长任务要有阶段性输出 |
| 计划泄露内部信息 | 规划输出做脱敏 |
| 重规划震荡 | 携带失败原因 + 设收敛条件 |

# 9. 小结

- 三种范式：ReAct（边想边做）、Plan-and-Execute（先规划后执行）、混合。
- 生产推荐**粗规划 + 步内 ReAct + 动态重规划**。
- 规划提示要限制步数、结构化输出、标注依赖。
- 重规划必须有次数上限，并携带失败原因。
- 规划质量要靠轨迹评测来度量。
