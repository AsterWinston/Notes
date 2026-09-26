# 1. 架构总览

Agent 架构回答三个根本问题：**谁来决定下一步（控制流归属）、状态存放在哪里（状态归属）、组件之间如何分工（组件边界）**。同一个业务目标，用不同架构实现，能力上限、成本、可调试性差别巨大。

## 1.1 什么是 Agent 架构

结论先行：架构不是"用了哪个框架"，而是**控制流的组织方式**。它决定了：

- 决策由模型还是代码做出；
- 路径是运行时动态生成还是预先固定；
- 状态是集中还是分散、是否可恢复；
- 组件之间是函数调用还是消息传递。

框架（LangChain4j、Spring AI Alibaba、LangGraph4j）只是这些组织方式的工程化封装，理解架构才不会被框架绑架。

## 1.2 五种架构速览

| 架构 | 核心思想 | 适用 |
|------|----------|------|
| 单体 Agent | 一个 LLM 完成全部决策 | 简单任务 |
| 规划器-执行器 | 规划与执行分离 | 中长任务 |
| 分层 Agent | 上层决策、下层执行 | 复杂系统 |
| 图编排 | 节点 + 边显式控制流 | 需要确定性 |
| 多 Agent | 多个专职 Agent 协作 | 复杂分工 |

五者并非互斥：生产系统常是"图编排的骨架 + 单体的节点 + 规划器-执行器作为子图"的混合形态。

## 1.3 架构演进的主线

### 1.3.1 控制流的三种归属

```
控制流归属
├── 代码（Workflow）      路径预先写死，模型只在节点内做局部判断
├── 模型（Agent）         每一步由模型决定，路径运行时生成
└── 混合（Hybrid）        骨架由代码固定，节点内部交给模型
```

> **越往模型侧，灵活性越高，可预测性越低。** 架构选型本质是在这条谱系上选点。

### 1.3.2 复杂度与能力的权衡

| 维度 | 单体 | 规划器-执行器 | 分层 | 图编排 | 多 Agent |
|------|:----:|:------------:|:----:|:------:|:--------:|
| 能力上限 | 低 | 中 | 高 | 高 | 最高 |
| 实现复杂度 | 低 | 中 | 高 | 中高 | 最高 |
| 可预测性 | 低 | 中 | 中 | 高 | 低 |
| 可调试性 | 易 | 中 | 难 | 中 | 难 |
| 成本可控性 | 中 | 中 | 低 | 高 | 低 |

> 复杂度是**成本**，不是**能力**。能不用就不用。

# 2. 单体 Agent

## 2.1 结构

结论先行：单体 Agent 把**规划、决策、执行、记忆**全部压进一次模型调用循环。

```
目标 → [LLM + 工具 + 记忆] → 结果
```

模型在每一步自行决定：调用哪个工具、传什么参数、何时给出最终答案。

## 2.2 完整实现

### 2.2.1 接口定义

```java
interface Assistant {
    @SystemMessage("""
            你是一个订单助手。
            规则：只依据工具返回的信息回答；信息不足时明确说明。
            """)
    String chat(@UserMessage String message);
}
```

### 2.2.2 构建与调用

```java
Assistant agent = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(allTools)
        .chatMemory(memory)
        .build();

String answer = agent.chat("订单 A123 能退款吗？");
```

内部循环等价于：模型输出工具调用 → 框架执行 → 结果回填 → 再次调用模型，直到模型不再请求工具。

## 2.3 边界与坑

| 坑 | 表现 | 原因 |
|----|------|------|
| 工具选择困难 | 频繁误选相邻工具 | 工具数过多，描述重叠 |
| 长任务漂移 | 多步后忘记原始目标 | 上下文被工具结果淹没 |
| 无中间可见性 | 只有最终答案 | 决策过程未单独记录 |
| 成本不可控 | 循环步数无上限 | 缺少 maxSteps |

**反例**：给单体 Agent 挂 40 个工具，指望它自动选对——正确做法是先按意图裁剪工具集，再交给单体。

## 2.4 生产建议

- 工具数控制在 **< 10**，步骤 **< 5**；
- 必设 `maxSteps`、单次 Token 上限与超时；
- 把目标写进系统提示的**固定位置**，每轮重申；
- 记录每一步的工具调用签名，便于事后归因。

> 适合工具少（<10）、步骤短（<5）的任务。

# 3. 规划器-执行器（Planner-Executor）

## 3.1 结构

结论先行：把"想"和"做"拆成两个角色——规划器产出步骤序列，执行器逐步执行，最后由汇总器整合。

```
        ┌──────────┐
目标 ──▶│  Planner │──▶ 步骤序列
        └──────────┘
              │
              ▼
        ┌──────────┐
        │ Executor │──▶ 逐步执行（可含 ReAct）
        └──────────┘
              │
              ▼
        ┌──────────┐
        │Synthesizer│──▶ 汇总结果
        └──────────┘
```

## 3.2 完整实现

### 3.2.1 数据结构

```java
record SubTask(int index, String action, String tool, List<Integer> dependsOn) {}

record TaskResult(int index, boolean ok, String output, String error) {}
```

用 record 承载计划与结果，便于序列化落库、便于审计。

### 3.2.2 编排代码

```java
public String solve(String task) {
    List<SubTask> plan = planner.plan(task);

    List<TaskResult> results = new ArrayList<>();
    for (SubTask sub : plan) {
        results.add(executor.execute(sub));       // 执行器内部可用 ReAct
    }
    return synthesizer.synthesize(task, results);
}
```

若步骤间无依赖，可并行执行：

```java
public List<TaskResult> runParallel(List<SubTask> plan) throws Exception {
    try (var pool = Executors.newFixedThreadPool(4)) {
        List<Future<TaskResult>> futures = new ArrayList<>();
        for (SubTask sub : plan) {
            futures.add(pool.submit(() -> executor.execute(sub)));
        }
        List<TaskResult> results = new ArrayList<>();
        for (Future<TaskResult> f : futures) {
            results.add(f.get());   // 伪代码：真实场景需处理超时与异常
        }
        return results;
    }
}
```

> 并行只对**无依赖且无副作用冲突**的步骤安全；有写操作时须做冲突检测。

## 3.3 重规划

执行器发现"计划假设失效"时，应把**失败原因 + 已完成结果**回传给规划器：

```java
if (isFailure(result) && replanBudget-- > 0) {
    plan = planner.replan(task, results, result.error());
    // 重置游标，基于已完成结果继续
}
```

## 3.4 边界与坑

| 坑 | 说明 |
|----|------|
| 计划失效 | 环境变化后原计划不可执行 |
| 重规划震荡 | 反复"规划-失败-再规划" |
| 执行盲区 | 执行阶段看不到全局，局部最优 |
| 计划过长 | 10+ 步的计划极易过时 |

## 3.5 生产建议

- 计划步数限制在 3~8 步，结构化输出（JSON）；
- 重规划上限 1~3 次，且必须携带失败原因；
- 每个子任务独立超时与重试；
- 计划与结果持久化，支持中断续跑。

# 4. 分层 Agent

## 4.1 结构

结论先行：**上层定策略、下层做执行**，每层只对相邻层负责。

```
┌─────────────────────────────┐
│  决策层（Strategy Agent）    │  定目标、定策略
└──────────────┬──────────────┘
               │ 下发子任务
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐┌────────┐┌────────┐
│执行Agent││执行Agent││执行Agent│  各自带工具
└────────┘└────────┘└────────┘
```

## 4.2 实现

```java
interface StrategyAgent {
    List<Assignment> decompose(String goal, WorldState state);
}

interface WorkerAgent {
    WorkerResult handle(Assignment task);
}

public String run(String goal) {
    List<Assignment> tasks = strategy.decompose(goal, stateStore.load(goalId));
    List<WorkerResult> done = new ArrayList<>();
    for (Assignment a : tasks) {
        done.add(workerFor(a.domain()).handle(a));   // 按领域路由到不同 worker
    }
    return strategy.merge(goal, done);
}
```

## 4.3 通信与状态同步

| 方式 | 说明 | 风险 |
|------|------|------|
| 直接调用 | 上层同步调用下层 | 耦合强、阻塞 |
| 消息队列 | 异步解耦 | 时序与幂等复杂 |
| 黑板（Blackboard） | 共享状态，各自读写 | 并发写冲突 |

> 状态外置（见 `10_state_and_control_flow.md`）是分层的必要条件。

## 4.4 边界与坑

- **层间协议不清晰**：上下层对字段含义理解不一致；
- **状态双写**：多层各自维护副本，产生不一致；
- **层层转发**：层级过多导致信息衰减与延迟。

## 4.5 生产建议

- 层间用**显式 DTO**定义契约，版本化；
- 状态单一来源（Single Source of Truth），其余层只读；
- 限制层数（建议 ≤ 3 层）。

> 是 `10_MultiAgent` 中 Supervisor 模式的基础。

# 5. 图编排（Graph）

## 5.1 结构

结论先行：用**节点（Node）+ 边（Edge）+ 共享状态（State）**显式描述控制流，兼顾确定性与灵活性。

用节点和边显式描述控制流，兼顾确定性与灵活性：

```
        ┌─────────┐
        │  START  │
        └────┬────┘
             ▼
        ┌─────────┐
        │ 意图识别 │
        └────┬────┘
      ┌──────┼──────┐
      ▼      ▼      ▼
   ┌────┐ ┌────┐ ┌────┐
   │检索│ │工具│ │Agent│
   └──┬─┘ └──┬─┘ └──┬─┘
      └──────┼──────┘
             ▼
        ┌─────────┐
        │  汇总    │
        └────┬────┘
             ▼
        ┌─────────┐
        │   END   │
        └─────────┘
```

## 5.2 状态与节点

图编排的核心是**共享状态对象**：每个节点读取状态、返回对状态的增量修改。

```java
record GraphState(String query, String intent, String context,
                  String answer, int steps) {}

interface Node {
    GraphState apply(GraphState state);
}

interface ConditionalEdge {
    String route(GraphState state);   // 返回下一个节点名
}
```

## 5.3 实现（伪代码）

```java
public GraphState run(GraphState init) {
    GraphState state = init;
    String node = "START";
    while (!"END".equals(node) && state.steps() < MAX_STEPS) {
        state = nodes.get(node).apply(state);
        node = router.route(state);      // 依据状态选择下一节点
    }
    return state;
}
```

代表实现：

| 实现 | 说明 |
|------|------|
| LangGraph4j | LangGraph 的 Java 版 |
| Spring AI Alibaba Graph | 节点、边、状态流转 |
| LangChain4j Agentic | 声明式顺序/并行/循环 |

> 详见 `10_MultiAgent/4_langgraph4j.md`。

## 5.4 边界与坑

- **图复杂度爆炸**：节点与边增多后，图本身成为难以理解的"意大利面"；
- **状态字段膨胀**：所有节点往同一个状态里塞字段；
- **条件边过深**：嵌套路由导致路径不可枚举。

## 5.5 生产建议

- 单图节点数控制在 **< 20**，按子域拆分子图；
- 状态字段分组、命名带前缀，避免污染；
- 每个节点独立超时，失败可回退到上一稳定节点；
- 图结构纳入版本管理，变更需回归评测。

# 6. 多 Agent 协作

## 6.1 协作拓扑

结论先行：多 Agent 的价值来自**专业化与并行**，代价是通信与协调成本。

| 拓扑 | 说明 | 适用 |
|------|------|------|
| Supervisor | 一个主管分派、汇总 | 任务可清晰分工 |
| 流水线 | A 产出交给 B | 阶段明确的流程 |
| 群聊/辩论 | 多 Agent 讨论收敛 | 需要多视角 |
| 黑板 | 共享状态协作 | 弱耦合探索 |

## 6.2 实现

```java
record Message(String from, String to, String content, int round) {}

public String supervise(String goal) {
    List<Message> transcript = new ArrayList<>();
    for (int round = 0; round < MAX_ROUNDS; round++) {
        Message decision = supervisor.decide(goal, transcript);
        if (decision.to().equals("FINISH")) return decision.content();

        Message reply = agents.get(decision.to()).handle(decision);
        transcript.add(decision);
        transcript.add(reply);
    }
    return "未在限定轮次内收敛";
}
```

## 6.3 边界与坑

- **死循环对话**：两个 Agent 互相等待/推诿；
- **成本倍增**：每个 Agent 都消耗 Token，轮次越多越贵；
- **一致性缺失**：多个 Agent 对同一事实给出不同版本；
- **责任不清**：失败后无法定位是哪个 Agent 的错。

## 6.4 生产建议

- 设总轮次上限与总成本上限；
- 主管负责终止判定，禁止 Agent 之间无限直接对话；
- 所有消息进 Trace，按 `from/to/round` 可回放；
- 能用单 Agent + 工具解决的，不引入多 Agent。

# 7. 架构选择决策

## 7.1 决策树

```
工具数量 < 10 且步骤短？
    ├── 是 → 单体 Agent
    └── 否 → 步骤能否预先规划？
              ├── 能 → 规划器-执行器
              └── 否 → 需要确定性控制流？
                        ├── 需要 → 图编排
                        └── 不需要 → 多 Agent 协作
```

## 7.2 决策补充维度

| 维度 | 倾向单 Agent | 倾向多 Agent / 图 |
|------|--------------|------------------|
| 任务可分解性 | 低 | 高 |
| 领域数量 | 单一 | 多个异构领域 |
| 确定性要求 | 低 | 高 |
| 成本敏感度 | 高 | 中 |
| 团队调试能力 | 弱 | 强 |

## 7.3 常见误判

- 把"任务复杂"直接等同于"需要多 Agent"——复杂不等于可分解；
- 把"想要可观测"寄托于图编排——可观测靠 Trace，不靠架构；
- 一开始就上多 Agent，导致调试成本淹没收益。

# 8. 组件化设计

## 8.1 组件划分

无论哪种架构，都应按组件拆分：

```
AgentService
├── Planner          规划
├── Executor         执行
├── ToolRegistry     工具注册与权限
├── MemoryManager    记忆
├── Guardrails       安全
└── Observer         Trace/指标
```

## 8.2 接口隔离

```java
public interface Planner {
    List<SubTask> plan(String task);
    List<SubTask> replan(String task, List<TaskResult> done, String reason);
}

public interface ToolRegistry {
    List<Object> toolsFor(String scene, User user);   // 按场景与权限裁剪
}
```

每个组件只依赖接口，便于**替换实现**（如把 Planner 从 LLM 换成规则引擎）与**独立测试**。

## 8.3 依赖方向

```
AgentService → Planner / Executor / ToolRegistry / MemoryManager
     │                     │
     └──────────────► Observer（旁路，不反向依赖）
```

- 组件之间禁止循环依赖；
- 横切关注点（Trace、审计、限流）用装饰器/拦截器注入；
- 状态通过显式参数传递，避免隐式全局变量。

> 组件化让每个部分可独立测试、替换与演进。

# 9. 架构与可靠性

## 9.1 风险与对策

| 架构 | 主要风险 | 对策 |
|------|----------|------|
| 单体 | 漂移、工具误选 | 限制工具数、明确提示 |
| 规划器-执行器 | 计划失效 | 动态重规划 |
| 分层 | 通信/状态不一致 | 显式协议 + 状态外置 |
| 图编排 | 图复杂度爆炸 | 限制节点数、模块化 |
| 多 Agent | 死循环、成本高 | 步数/成本上限 |

## 9.2 通用可靠性手段

| 手段 | 作用 |
|------|------|
| 多重上限 | 步数、Token、时间、成本四道闸 |
| 幂等键 | 写操作可安全重试 |
| 状态外置 | 崩溃可恢复、可水平扩展 |
| 全链路 Trace | 失败可定位、可回放 |
| 降级路径 | 稳定工作流兜底 |

## 9.3 边界说明

架构只能降低风险，不能消除风险。任何架构都必须在**上限 + 可观测 + 兜底**三者齐备时才可上生产。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 架构过度设计 | 从单体起步，按需演进 |
| 组件耦合 | 接口隔离，独立测试 |
| 无状态外置 | 状态统一管理 |
| 无统一 Trace | 全链路追踪 |
| 工具全量挂载 | 按场景/权限裁剪 |

## 10.2 落地顺序

1. 先跑通单体 + 少量工具；
2. 指标不足时再引入规划器-执行器；
3. 需要确定性时再上图层；
4. 只有当领域确实异构时才拆多 Agent。

# 11. 小结

## 11.1 核心要点

- 五种架构：单体、规划器-执行器、分层、图编排、多 Agent。
- **从单体起步，按复杂度演进**，避免过度设计。
- 图编排适合需要确定性控制流的场景。
- 无论哪种架构，都要组件化 + 状态外置 + 全链路可观测。

## 11.2 关联章节

- 状态与控制流：`10_state_and_control_flow.md`。
- 人机协同与审批：`11_human_in_the_loop.md`。
- 失败模式与防御：`13_failure_modes.md`。
