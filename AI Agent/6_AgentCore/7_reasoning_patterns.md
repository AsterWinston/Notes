# 1. 范式总览

## 1.1 什么是推理范式

推理范式（Reasoning Pattern）是 Agent 组织"思考"的固定套路。它决定了模型在一次任务中**如何分解问题、如何利用中间结果、如何收敛到答案**。

选择合适范式，往往比换更强的模型更能提升效果与降低成本。

从工程视角看，范式回答三个问题：

- **思考的粒度**：一次生成一整段推理，还是拆成多步交替；
- **信息来源**：只依赖模型内部知识，还是允许调用工具获取外部信息；
- **收敛方式**：一次成稿，还是反复采样、评估、回溯后收敛。

## 1.2 六种范式一览

| 范式 | 核心思想 | 提出场景 |
|------|----------|----------|
| CoT | 分步推理 | 提升推理准确性 |
| ReAct | 推理 + 行动交替 | Agent 基础范式 |
| Reflexion | 反思失败并改进 | 自我提升 |
| ToT | 树状搜索多条思路 | 复杂决策 |
| LATS | 树搜索 + 蒙特卡洛 | 复杂规划 |
| Self-Consistency | 多路径投票 | 提升稳定性 |

## 1.3 选择原则：成本递增

```
CoT（最便宜）
  → ReAct（可行动）
    → Reflexion / Self-Consistency（可纠错/稳定）
      → ToT / LATS（最强，最贵）
```

> **先上最便宜的范式，不够再升级**，不要一上来就用 ToT/LATS。

## 1.4 范式的共同结构

尽管六种范式形态各异，它们都共享同一套骨架：

```
┌──────────────────────────────────────────┐
│  生成（Generate）                          │
│      │                                    │
│      ▼                                    │
│  评估（Evaluate）                          │
│      │                                    │
│      ├── 满足 → 输出                       │
│      └── 不满足 → 反馈（Feedback）→ 回到生成 │
└──────────────────────────────────────────┘
```

差异只在三个维度：**是否允许行动**、**候选是否分支**、**反馈从哪来**。

| 范式 | 生成方式 | 是否行动 | 反馈来源 |
|------|----------|:--------:|----------|
| CoT | 单链 | 否 | 无（一次性） |
| ReAct | 单链交替 | 是 | 工具观测 |
| Reflexion | 单链重试 | 是 | 评估器 |
| Self-Consistency | 多路采样 | 否 | 投票 |
| ToT | 树状分支 | 否 | 评估器剪枝 |
| LATS | 树状分支 | 是 | 环境 + 评估器 |

# 2. CoT（思维链）

## 2.1 原理

```
问题 → 一步步推理 → 结论
```

让模型把中间推理显式写出来，而不是直接给答案。显式推理能显著提升数学、逻辑、多跳问答的准确率。

### 2.1.1 为什么有效

- **分解计算**：把一步到位的复杂映射，拆成多个简单映射，降低单步错误率；
- **利用上下文**：中间结果进入上下文，成为后续步骤的"工作记忆"；
- **可解释**：推理链可被人工检查，出错时能定位到具体一步。

## 2.2 实现

```java
@SystemMessage("""
        解决问题时先分步推理，再给结论。格式：
        推理：
        1. ...
        2. ...
        结论：...
        """)
```

### 2.2.1 Zero-shot 与 Few-shot CoT

| 方式 | 做法 | 适用 |
|------|------|------|
| Zero-shot CoT | 仅加一句"请一步步思考" | 通用、成本低 |
| Few-shot CoT | 给 1~3 个带推理的示例 | 领域固定、格式要求高 |

```java
@SystemMessage("""
        按示例的推理风格作答。
        示例：
        问：小明有 3 个苹果，又买了 5 个，吃了 2 个，还剩几个？
        推理：3 + 5 = 8，8 - 2 = 6。
        结论：6 个。
        """)
```

## 2.3 适用与局限

### 2.3.1 适用

- 数学计算、逻辑推理；
- 多跳问答；
- 需要可解释性的场景。

### 2.3.2 局限

- 推理链变长 → Token 增加；
- 推理错误会被放大；
- 无法获取外部信息（无行动能力）。

### 2.3.3 变体

- **CoT-SC**：CoT + Self-Consistency，多次采样取多数；
- **Zero-shot CoT**：靠一句触发语引出推理；
- **Plan-and-Solve**：先让模型出计划，再按计划逐步求解。

> CoT 是其他范式的基础；对推理类任务提升明显。

# 3. ReAct（推理 + 行动）

## 3.1 原理

```
Thought: 我需要先查订单
Action:  getOrder("A123")
Observation: 已发货，运单 SF123
Thought: 需要查物流
Action:  getLogistics("SF123")
Observation: 运输中，明天到
Answer:  您的订单明天送达
```

"想 → 做 → 看"交替进行，让模型在行动中获取新信息并调整。

## 3.2 输出格式

```java
@SystemMessage("""
        你可以使用工具。每次输出格式：
        Thought: 你的思考
        Action: 工具名
        Action Input: JSON 参数

        得到 Observation 后继续，直到可以输出：
        Answer: 最终答案
        """)
```

## 3.3 与 CoT 的区别

| 维度 | CoT | ReAct |
|------|-----|-------|
| 是否有行动 | 无 | 有 |
| 是否获取外部信息 | 否 | 是 |
| 适用 | 纯推理 | 需外部信息/动作 |

> ReAct 是所有 Agent 框架的基础范式，详见 `4_agent_loop.md`。

## 3.4 ReAct 循环的工程实现

ReAct 的执行循环本质是一个"生成 → 解析 → 执行 → 回填"的状态机：

```java
public String run(String task, int maxSteps) {
    List<Message> history = new ArrayList<>(List.of(new UserMessage(task)));

    for (int step = 0; step < maxSteps; step++) {
        String output = model.chat(history);          // 生成
        Action action = parser.parse(output);          // 解析

        if (action.isFinalAnswer()) {
            return action.answer();                    // 收敛
        }
        String observation = tools.execute(action);    // 执行
        history.add(new AssistantMessage(output));
        history.add(new ToolMessage(observation));     // 回填
    }
    throw new MaxStepsExceededException();
}
```

> 边界：必须设 `maxSteps`。模型可能陷入"反复调用同一工具"的循环，需要检测重复 Action 并强制收敛。

# 4. Reflexion（反思）

## 4.1 原理

```
尝试 → 失败 → 反思原因 → 改进 → 重试
```

把"失败教训"作为反馈注入下一次尝试，让 Agent 越试越准。

## 4.2 实现

```java
public String solveWithReflexion(String task, int maxAttempts) {
    String attempt = agent.chat(task);
    for (int i = 0; i < maxAttempts; i++) {
        String evaluation = evaluator.evaluate(task, attempt);
        if (evaluation.isSuccess()) return attempt;

        String reflection = reflector.reflect(task, attempt, evaluation.feedback());
        attempt = agent.chat(task + "\n\n上次失败原因：\n" + reflection);
    }
    return attempt;
}
```

### 4.2.1 评估器设计

Reflexion 的效果高度依赖评估器质量：

| 评估器类型 | 判定依据 | 优点 | 缺点 |
|------------|----------|------|------|
| 规则评估 | 测试/断言 | 客观、便宜 | 覆盖有限 |
| LLM 评估 | 打分/评审 | 通用 | 有噪声、成本高 |
| 环境评估 | 工具返回状态 | 真实 | 依赖可执行环境 |

> 没有可靠评估器时，Reflexion 会"反思错误的方向"，越改越差。

## 4.3 适用

- 一次做不对、但可迭代改进的任务（写作、代码）；
- 有明确评估标准的场景（测试、校验）。

## 4.4 与重试的区别

| 维度 | 朴素重试 | Reflexion |
|------|----------|-----------|
| 是否携带教训 | 否 | 是 |
| 是否改变策略 | 否 | 是 |
| 成本 | 低 | 高 |
| 效果 | 只解决随机波动 | 可纠正系统性错误 |

> 详见 `8_reflection_and_self_critique.md`。

# 5. ToT（思维树）

## 5.1 原理

```
               问题
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
    思路A     思路B     思路C
      │         │
   ┌──┴──┐   ┌──┴──┐
   ▼     ▼   ▼     ▼
  A1    A2  B1    B2
      │
      ▼
   评估选择最优路径
```

把推理展开成树，逐层生成候选、评估剪枝，最终选最优路径。

## 5.2 实现

```java
public String treeOfThoughts(String problem, int depth, int breadth) {
    List<String> thoughts = generateThoughts(problem, breadth);

    for (int d = 1; d < depth; d++) {
        List<String> next = new ArrayList<>();
        for (String thought : thoughts) {
            next.addAll(generateThoughts(problem + "\n当前思路：" + thought, breadth));
        }
        thoughts = evaluateAndSelect(problem, next, breadth);   // 剪枝
    }
    return best(thoughts);
}
```

## 5.3 成本分析

- 调用次数 ≈ `breadth^depth`，**指数增长**；
- 必须剪枝（评估函数），否则不可用；
- 适合"决策空间大但单次试错便宜"的场景。

优点：探索多路径；缺点：成本高。

## 5.4 搜索策略对比

ToT 的节点扩展顺序决定了实际成本与效果：

| 策略 | 说明 | 适用 |
|------|------|------|
| BFS | 逐层扩展全部节点 | 深度浅、分支少 |
| DFS | 深入单条路径 | 有明确目标方向 |
| 最佳优先 | 每次扩展评分最高的节点 | 有可靠评估器 |

> 边界：ToT 依赖可比较的评估分数。若评估器不可靠，剪枝会"剪掉正确路径"，反而不如单链 CoT。

# 6. Self-Consistency（自洽性）

## 6.1 原理

```
同一问题采样多次 → 取多数答案
```

用较高温度多次采样，通过"投票"降低单次随机性带来的错误。

## 6.2 实现

```java
public String selfConsistent(String question, int samples) {
    List<String> answers = IntStream.range(0, samples)
            .mapToObj(i -> model.chat(question, ChatOptions.builder()
                    .temperature(0.7)      // 提高多样性
                    .build()))
            .toList();

    return mostCommon(answers);         // 多数表决
}
```

## 6.3 适用与局限

- 适用：有唯一正确答案的推理题、分类题；
- 局限：开放生成任务无"多数"可言；成本随采样数线性增长。

## 6.4 采样数与温度

| 参数 | 影响 | 建议 |
|------|------|------|
| 采样数 N | 越大越稳，成本线性上升 | 5~10 通常足够 |
| 温度 | 太低则答案雷同，失去投票意义 | 0.5~1.0 |
| 归一化 | 答案需可比较 | 先抽取最终答案再投票 |

> 边界：投票前要**抽取并规范化答案**（如统一数字、去空格）。否则"6 个"与"六个"会被当成两个不同答案。

# 7. LATS（语言 Agent 树搜索）

## 7.1 原理

LATS = ToT + 蒙特卡洛树搜索 + 环境反馈，用于复杂规划：

```
选择 → 扩展 → 模拟 → 回溯
```

## 7.2 四阶段

| 阶段 | 动作 |
|------|------|
| 选择 | 按价值选择最有希望的节点 |
| 扩展 | 生成候选动作（工具调用） |
| 模拟 | 用 LLM 评估 + 环境反馈打分 |
| 回溯 | 把结果回传更新节点价值 |

- 节点 = 一个状态（含历史动作）
- 边 = 一个动作（工具调用）
- 价值 = LLM 评估 + 环境反馈

## 7.3 适用

> 成本高，适合高价值、可承受多轮试错的场景。

## 7.4 与 ToT 的区别

| 维度 | ToT | LATS |
|------|-----|------|
| 是否有行动 | 否 | 是 |
| 搜索算法 | 树搜索 | 蒙特卡洛树搜索 |
| 反馈来源 | 评估器 | 环境 + 评估器 |
| 是否回溯更新 | 弱 | 强（价值回传） |
| 成本 | 高 | 很高 |

> 简单记：LATS 是"能动手的 ToT"，并且带价值回传，因此更适合需要与环境交互的长程规划。

# 8. 范式选择指南

## 8.1 按任务特征

| 任务特征 | 推荐范式 |
|----------|----------|
| 简单问答 | 直接回答 / CoT |
| 需要外部信息或动作 | ReAct |
| 一次做不对，可迭代改进 | Reflexion |
| 决策空间大、需要比较 | ToT |
| 有唯一答案、要求稳定 | Self-Consistency |
| 复杂长程规划 | LATS / Plan-and-Execute |

## 8.2 决策树

```
是否需要外部信息/动作？
  ├── 否 → 需要多步推理？
  │          ├── 是 → CoT
  │          └── 否 → 直接回答
  └── 是 → ReAct
            └── 一次做不对？
                  ├── 是 → + Reflexion
                  └── 决策空间大 → + ToT / LATS
```

## 8.3 边界与反例

- **反例一**：用 ToT 做单步事实问答。分支毫无意义，纯粹浪费成本。
- **反例二**：用 Self-Consistency 做创意写作。没有"正确答案"，投票会抹平多样性。
- **反例三**：没有可靠评估器却上 Reflexion。会放大错误而非纠正。
- **反例四**：任务有副作用（下单、转账）却用 LATS 反复试错。可能产生不可逆的真实操作。

> 选择范式前先问三个问题：**有外部信息吗？有评估器吗？有副作用吗？**

# 9. 组合使用

## 9.1 常见组合

```
ReAct（执行） + Reflexion（失败后反思） + Self-Consistency（关键决策投票）
```

## 9.2 实现

```java
public String robustSolve(String task) {
    for (int attempt = 0; attempt < 3; attempt++) {
        String result = reactAgent.run(task);        // ReAct 执行
        if (evaluator.isAcceptable(task, result)) {
            return result;
        }
        task = task + "\n\n改进要求：" + reflector.reflect(task, result);  // Reflexion
    }
    return "未能完成任务";
}
```

## 9.3 组合的代价

| 组合 | 成本放大 | 收益 |
|------|:--------:|------|
| ReAct + Reflexion | 尝试次数倍 | 纠错 |
| ReAct + Self-Consistency | 采样数倍 | 关键步骤稳定 |
| Reflexion + ToT | 指数 × 尝试 | 复杂决策纠错 |
| 三者全上 | 极高 | 仅高价值任务 |

> 边界：组合是**乘法**关系。每加一层，成本翻倍甚至指数上升。只在高价值、低频任务上组合多层。

# 10. 成本与收益

## 10.1 对比

| 范式 | 相对成本 | 相对收益 |
|------|:--------:|:--------:|
| CoT | 1.2x | 中 |
| ReAct | 2~5x | 高（可行动） |
| Reflexion | 3~8x | 高（可纠错） |
| Self-Consistency | Nx | 中 |
| ToT | 指数级 | 高（复杂决策） |
| LATS | 很高 | 高（复杂规划） |

## 10.2 升级策略

> **先上最便宜的范式，不够再升级**，不要一上来就用 ToT/LATS。

- 从 CoT/ReAct 起步；
- 效果不足再加 Reflexion/Self-Consistency；
- 仅对高价值任务启用 ToT/LATS。

# 11. 小结

## 11.1 关键要点

- CoT 是基础，ReAct 是 Agent 的通用范式。
- Reflexion 解决"一次做不对"，Self-Consistency 提升稳定性。
- ToT/LATS 能力强但成本高，按需使用。
- 生产中常组合使用，并按"成本递增"逐级升级。

## 11.2 延伸阅读

- `4_agent_loop.md`：ReAct 循环与 Agent 主循环。
- `8_reflection_and_self_critique.md`：反思与自我批判。
- `6_task_decomposition.md`：与 Plan-and-Execute 的关系。
