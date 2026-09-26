# 1. 为什么要用多 Agent？

## 1.1 标准答案

**答**：多 Agent（Multi-Agent，多智能体）是把复杂任务拆解给多个各司其职的 Agent，通过协作完成单 Agent 难以胜任的工作。

**收益**：分工专业化、上下文隔离、并行执行、可独立演进、权限隔离。

**代价**：复杂度、成本、延迟、调试难度、失败模式全面上升。

**结论**：先用工作流或单 Agent，确认不足再上多 Agent。

## 1.2 深入原理

### 1.2.1 单 Agent 的三个瓶颈

- **上下文瓶颈**：工具、知识、历史塞进同一窗口，注意力被稀释；
- **能力瓶颈**：一个提示难兼顾严谨的代码审查与发散的创意写作；
- **权限瓶颈**：持有全部工具时，越权风险无法收敛。

### 1.2.2 收益从何而来

多 Agent 的本质是**用组织分工弥补单点智能不足**：以"专家 + 协调机制"替代"全能模型"。收益来自专业化提示、独立上下文与并行化。

## 1.3 代码示例

```java
boolean shouldUseMultiAgent(Task task) {
    return task.subTasks() > 2
        && task.requiresDistinctSkills()
        && task.budgetTokens() > 50_000;
}
```

## 1.4 常见追问

**Q：多 Agent 一定比单 Agent 强吗？**
A：不一定；若单提示可解，只会增加成本与误差累积。

**Q：什么时候收益最明显？**
A：任务可分解、子任务技能差异大、允许并行、对延迟不敏感时。

## 1.5 生产视角

上线前必须回答两个问题：**基线是什么**、**增量是否值回成本**。没有单 Agent 基线的多 Agent 系统无法证明价值，属于不可运维的系统。

# 2. 常见编排模式？

## 2.1 标准答案

| 模式 | 决策者 | 特点 |
|------|--------|------|
| Supervisor | 主管 Agent | 中心化、可控 |
| Swarm | Agent 间自主 | 去中心、灵活 |
| 流水线 | 预定义顺序 | 稳定、难应变 |
| 并行 | 预定义 | 快、需聚合 |
| 分层 | 多层主管 | 适配复杂组织 |
| 辩论 | 规则 | 提质量、成本高 |

**Supervisor 最常用**，流程清晰可控。

## 2.2 深入原理

### 2.2.1 中心化与去中心化

**中心化**（Supervisor、分层）由协调者决定下一步，链路清晰、易追踪；**去中心化**（Swarm）由 Agent 自主交接，灵活但易失控。

### 2.2.2 拓扑决定可观测性

星型拓扑的协作 Trace 天然成树，便于回放；网状拓扑的调用关系是图，回溯成本高。

## 2.3 代码示例

```java
enum Pattern { SUPERVISOR, SWARM, PIPELINE, PARALLEL, HIERARCHY, DEBATE }

Orchestrator pick(Pattern p) {
    return switch (p) {
        case SUPERVISOR -> new SupervisorOrchestrator();
        case SWARM -> new SwarmOrchestrator();
        default -> throw new UnsupportedOperationException(p.name());
    };
}
```

## 2.4 常见追问

**Q：为什么生产偏爱 Supervisor？**
A：决策集中、终止明确、Trace 成树，运维与评测都更容易。

**Q：Swarm 的价值在哪？**
A：适合探索性、边界模糊的任务，但需要更强的护栏。

## 2.5 生产视角

模式选择应服从**可观测性与终止性**：优先能画出清晰流程图、能明确定义终止条件的模式，避免"看起来智能、实则不可控"的设计。

# 3. Supervisor 模式如何实现？

## 3.1 标准答案

每轮：**决策 → 委托 → 收集 → 再决策**，直到主管判定完成。

```java
for (step in maxSteps) {
    decision = supervisor.decide(agents, task, history);
    if (decision.isFinish()) return synthesize(task, history);
    result = agents.get(decision.action()).handle(decision.instruction());
    history.add(result);
}
```

## 3.2 深入原理

### 3.2.1 主管只做三件事

1. 选择下一个 Agent 与指令；2. 判断是否完成；3. 汇总最终答案。主管**不应亲自干活**，否则职责扩散。

### 3.2.2 循环的收敛条件

必须同时设置：最大步数、Token 预算、超时。任一到上限即强制进入汇总，避免无限循环。

## 3.3 代码示例

```java
record Decision(String action, String instruction, boolean finish) {}

Decision decide(List<Agent> agents, String task, List<Result> history) {
    String prompt = buildSupervisorPrompt(agents, task, history);
    return model.call(prompt).entity(Decision.class);
}
```

## 3.4 常见追问

**Q：主管如何知道该结束？**
A：在提示中给出完成判定标准，并强制输出 `finish=true/false` 的结构化字段。

**Q：主管用大模型还是小模型？**
A：主管负责决策，通常用较强模型；执行 Agent 可按需降级。

## 3.5 生产视角

把主管的决策也纳入 Trace，记录"选了谁、给了什么指令、为什么"。主管决策是排查问题的第一现场。

# 4. 多 Agent 的通信方式？

## 4.1 标准答案

| 方式 | 说明 | 适用 |
|------|------|------|
| 直接消息 | 点对点委托 | 简单调用 |
| 共享状态（黑板） | 协作完善同一产物 | 共同创作 |
| 消息队列 | 异步解耦 | 高并发 |
| 事件总线 | 广播订阅 | 松耦合 |

**传递摘要而非全量历史**，避免上下文爆炸。

## 4.2 深入原理

### 4.2.1 上下文隔离是双刃剑

隔离避免了污染，却导致信息不对称。解决方式是把必要上下文**显式结构化**地随消息传递。

### 4.2.2 黑板模式

多个 Agent 读写同一份共享产物（如文档草稿），通过版本与锁保证一致性。适合"多人改一份稿"的场景。

## 4.3 代码示例

```java
record Message(String from, String to, String summary, Map<String, Object> payload) {}

// 只传摘要与结构化字段，不传完整对话
Message handoff(Agent from, Agent to, Result result) {
    return new Message(from.id(), to.id(), result.summary(), result.keyFacts());
}
```

## 4.4 常见追问

**Q：为什么不直接传全部历史？**
A：全量历史会迅速撑爆上下文窗口，且引入无关噪声。

**Q：共享状态如何防冲突？**
A：加乐观锁或串行化写入，并记录版本号。

## 4.5 生产视角

通信协议要**版本化**：消息 schema 变更时前后兼容，否则升级期会大面积失败。消息体做大小上限保护。

# 5. 多 Agent 的失败模式？

## 5.1 标准答案

| 失败 | 对策 |
|------|------|
| 死循环 | 步数上限 + 循环检测 |
| 责任扩散 | 明确职责边界 |
| 通信爆炸 | 星型拓扑（非网状） |
| 目标漂移 | 每轮重申目标 |
| 上下文丢失 | 传结构化上下文 |
| 成本失控 | 限制 Agent 数与轮数 |

## 5.2 深入原理

### 5.2.1 责任扩散（Diffusion of Responsibility）

当多个 Agent 都能"负责"时，实际无人负责。对策是把每个子任务的**验收标准**绑定到唯一 Agent。

### 5.2.2 误差累积

每轮交接都可能引入偏差，多轮叠加后结果偏离。因此**减少不必要轮次**本身即是一种质量保障。

## 5.3 代码示例

```java
class LoopGuard {
    private final Set<String> signatures = new HashSet<>();
    private int steps = 0;

    boolean shouldStop(String signature) {
        return ++steps > MAX_STEPS || !signatures.add(signature);
    }
}
```

## 5.4 常见追问

**Q：死循环和正常重试如何区分？**
A：用调用签名（Agent + 工具 + 参数）判重；签名重复即视为循环。

**Q：如何防目标漂移？**
A：每轮在提示中重申原始目标与验收标准。

## 5.5 生产视角

失败模式应**逐个映射到监控指标**：循环率、平均轮数、单位任务成本、交接失败率。没有指标兜底的失败模式等于没有防护。

# 6. 什么是涌现行为？

## 6.1 标准答案

**答**：多个 Agent 协作产生的、单个 Agent 不具备的整体行为。

- 正向：互补、纠错、深度、覆盖；
- 负向：从众、极化、退化、幻觉传播。

**必须对比单 Agent 基线**才能判断价值。

## 6.2 深入原理

### 6.2.1 正向涌现

多个视角相互校验（如"生成者 + 批评者"），可显著提升事实准确率与覆盖度。

### 6.2.2 负向涌现

Agent 倾向于附和上游结论（从众），或把小错误放大传播（幻觉级联）。辩论模式下还可能立场极化。

## 6.3 代码示例

```java
// 用"独立生成 + 交叉校验"抑制从众
List<Answer> answers = agents.parallelStream()
    .map(a -> a.solve(task))     // 并行独立作答，避免相互影响
    .toList();
Answer merged = reconcile(answers); // 显式处理分歧
```

## 6.4 常见追问

**Q：如何判断涌现是正向的？**
A：与单 Agent 基线在评测集上做 A/B，看质量指标是否显著提升。

**Q：如何抑制负向涌现？**
A：让 Agent 先独立作答再汇总，避免顺序依赖造成从众。

## 6.5 生产视角

涌现行为**不可直接预测**，只能通过离线评测 + 线上灰度观察。任何"多 Agent 一定更好"的假设都必须由数据证伪或证实。

# 7. 如何控制多 Agent 成本？

## 7.1 标准答案

1. 限制 Agent 数（≤ 5）；
2. 限制轮数（≤ 10）；
3. 模型分层（简单 Agent 用小模型）；
4. 缓存中间结果；
5. 规则预筛（明显意图直接路由）；
6. 按任务设预算。

## 7.2 深入原理

### 7.2.1 成本是乘法而非加法

轮数 × Agent 数 × 每轮上下文 = 总 Token，任一项翻倍都会让成本指数上升。**优先压轮数**。

### 7.2.2 模型分层

协调与决策用强模型，格式转换、抽取等确定性高的步骤用小模型，可省 60% 以上成本。

## 7.3 代码示例

```java
class Budget {
    private long tokensUsed = 0;
    private final long limit;

    Budget(long limit) { this.limit = limit; }

    boolean tryConsume(long delta) {
        if (tokensUsed + delta > limit) return false;
        tokensUsed += delta;
        return true;
    }
}
```

## 7.4 常见追问

**Q：缓存什么最有效？**
A：子任务结果、检索结果、系统提示前缀（可命中前缀缓存）。

**Q：预算耗尽怎么办？**
A：降级为单 Agent 汇总，或直接返回当前最佳结果并标注"未完成"。

## 7.5 生产视角

为每个任务类型设**独立预算**并监控 P99 成本。成本异常通常先于质量异常暴露系统问题。

# 8. 如何保证多 Agent 可观测？

## 8.1 标准答案

**答**：完整协作 Trace，记录谁、何时、输入、输出、耗时、成本。

```
Trace: 用户任务
├── Span: Supervisor 决策 1
├── Span: Agent A（输入、输出、耗时、Token）
├── Span: Supervisor 决策 2
├── Span: Agent B
└── Span: 汇总
```

## 8.2 深入原理

### 8.2.1 Span 的父子关系

每个 Agent 调用是一个子 Span，主管决策是父 Span，最终汇总为根 Span，形成可回放的树。

### 8.2.2 关键字段

除耗时与 Token 外，必须记录**决策理由、工具调用、失败原因**，否则无法定位问题。

## 8.3 代码示例

```java
try (Scope span = tracer.span("agent." + agent.id())) {
    span.tag("input.summary", input.summary());
    Result r = agent.handle(input);
    span.tag("output.summary", r.summary());
    span.tag("tokens", String.valueOf(r.tokens()));
    return r;
}
```

## 8.4 常见追问

**Q：Trace 与日志有何区别？**
A：Trace 关注**调用链与耗时**，日志关注**事件明细**，两者互补。

**Q：采样比例如何定？**
A：错误全采、成功按比例采样，兼顾成本与可排查性。

## 8.5 生产视角

Trace 必须能回答："这次任务为什么慢/贵/错"。做不到这三点，说明可观测性不达标。

# 9. Swarm 与 Supervisor 的区别？

## 9.1 标准答案

| 维度 | Supervisor | Swarm |
|------|-----------|-------|
| 决策者 | 主管 | 各 Agent |
| 中心化 | 是 | 否 |
| 可追踪 | 易 | 难 |
| 循环风险 | 低 | 高 |

**生产推荐 Supervisor**。

## 9.2 深入原理

### 9.2.1 控制权的差异

Supervisor 把控制权集中在协调者，Swarm 把控制权下放给每个 Agent（通过交接函数决定下一位）。

### 9.2.2 失控风险

Swarm 中 A 交给 B、B 又交回 A 是常见死循环；Supervisor 由主管统一判断，天然抑制该问题。

## 9.3 代码示例

```java
// Swarm：Agent 自己决定下一位（伪代码）
record Handoff(String nextAgent, String instruction) {}

Handoff afterWork(Result r) {
    return r.needsReview()
        ? new Handoff("reviewer", r.summary())
        : new Handoff("__end__", r.summary());
}
```

## 9.4 常见追问

**Q：Swarm 适合什么场景？**
A：流程高度动态、无法预定义角色流转、且能承受一定失控风险的探索任务。

**Q：能否混合？**
A：可以，用 Supervisor 做外层护栏，内层小范围用 Swarm。

## 9.5 生产视角

若选 Swarm，必须额外加**最大交接次数**与**交接去重**，否则线上极易出现费用与时间失控。

# 10. 多 Agent 的状态如何管理？

## 10.1 标准答案

- 共享状态外置（持久化）；
- 并发写加锁；
- 交接传摘要；
- 状态字段精简；
- 支持中断恢复。

## 10.2 深入原理

### 10.2.1 状态外置的意义

把状态放在 Redis/数据库，Agent 本身无状态，才能横向扩展与崩溃恢复。

### 10.2.2 幂等与恢复

恢复时重放必须幂等：写操作带幂等键，已完成的步骤直接跳过。

## 10.3 代码示例

```java
record AgentState(
    String taskId,
    String goal,
    List<String> completedSteps,
    Map<String, Object> facts,
    Set<String> executedSignatures) {}
```

## 10.4 常见追问

**Q：状态里该存什么？**
A：目标、已完成步骤签名、关键事实、当前阶段；不存完整对话。

**Q：并发写冲突怎么办？**
A：加乐观锁（版本号 CAS）或把写操作串行化。

## 10.5 生产视角

状态结构要**可演进**：新增字段需向后兼容，历史任务恢复时不能因 schema 变化而失败。

# 11. 如何评测多 Agent？

## 11.1 标准答案

```
① 结果质量（对比单 Agent 基线）
② 过程指标（步数、Agent 调用分布）
③ 效率（耗时、成本）
④ 稳定性（成功率、死循环率）
⑤ 安全（越权、注入）
```

## 11.2 深入原理

### 11.2.1 过程指标为何重要

多 Agent 的质量可能达标但过程低效（绕远路）。过程指标能揭示"是否做了无用协作"。

### 11.2.2 稳定性指标

成功率、超时率、循环率共同刻画系统韧性，是能否上线的硬门槛。

## 11.3 代码示例

```java
record EvalReport(
    double qualityVsBaseline,
    double avgSteps,
    long p99LatencyMs,
    double successRate,
    double loopRate) {}
```

## 11.4 常见追问

**Q：如何构造基线？**
A：用同一任务集跑单 Agent，得到质量/成本/延迟三组对照数据。

**Q：循环率怎么算？**
A：触发循环护栏的任务数 ÷ 总任务数。

## 11.5 生产视角

评测集要覆盖**正常、边界、对抗**三类样本，并接入 CI；每次提示或拓扑变更都自动回归。

# 12. 什么时候不该用多 Agent？

## 12.1 标准答案

- 任务简单（单 Agent 够）；
- 强顺序依赖；
- 延迟敏感；
- 提升不明显（成本不值）；
- 负向涌现明显。

## 12.2 深入原理

### 12.2.1 强顺序依赖

若子任务必须严格串行且无法并行，多 Agent 只增加交接开销，不如单 Agent 直连。

### 12.2.2 延迟敏感

每次交接都增加一次模型往返，交互式场景下多 Agent 首字延迟难以接受。

## 12.3 代码示例

```java
boolean shouldStaySingle(Task task) {
    return task.steps() <= 2
        || task.strictlySequential()
        || task.latencyBudgetMs() < 3_000;
}
```

## 12.4 常见追问

**Q：延迟敏感但又需要多能力怎么办？**
A：用一个模型 + 多工具（工具即能力），而非多 Agent。

**Q：如何发现"负向涌现明显"？**
A：对比基线的质量指标下降即为信号。

## 12.5 生产视角

**默认单 Agent，多 Agent 需要"举证"**：只有在评测中证明质量或覆盖度显著提升，才允许引入。

# 13. 图编排与多 Agent 的关系？

## 13.1 标准答案

**答**：图编排是**实现多 Agent 的主流方式**。

```
节点 = Agent 或步骤
边   = 流转规则
状态 = 共享状态
```

代表：LangGraph4j、Spring AI Alibaba Graph。

## 13.2 深入原理

### 13.2.1 图带来的确定性

相比让模型自由决定流转，图把拓扑固定下来，只在节点内部保留自主性，兼顾灵活与可控。

### 13.2.2 条件边与循环

条件边实现分支，回边实现重试/反思循环；配合最大迭代数防止死循环。

## 13.3 代码示例

```java
StateGraph graph = new StateGraph(schema)
    .addNode("supervisor", supervisorNode)
    .addNode("worker", workerNode)
    .addConditionalEdges("supervisor", routeFn)
    .addEdge("worker", "supervisor")
    .setEntryPoint("supervisor");
```

## 13.4 常见追问

**Q：图编排与工作流引擎有何区别？**
A：图编排的节点内可调用 LLM 自主决策，工作流引擎节点通常是确定性代码。

**Q：状态怎么流转？**
A：节点读取共享状态、返回状态更新，由框架按 schema 合并。

## 13.5 生产视角

用图编排时，务必为**每个节点**加超时与重试，并为循环边设最大迭代数，否则单点异常会拖垮整图。

# 14. 多 Agent 的设计原则？

## 14.1 标准答案

1. 从简到繁；
2. 职责清晰；
3. 星型拓扑；
4. 状态外置；
5. 全程可观测；
6. 终止明确；
7. 成本可控；
8. 评测驱动。

## 14.2 深入原理

### 14.2.1 从简到繁

先单 Agent、再 Supervisor、最后才考虑复杂拓扑。每步演进都需评测数据支撑。

### 14.2.2 终止明确

任何协作都必须有确定的终止条件：目标达成、预算耗尽或步数上限。

## 14.3 代码示例

```java
record DesignChecklist(
    boolean minimalTopology,
    boolean clearOwnership,
    boolean stateExternalized,
    boolean observable,
    boolean boundedTermination) {}
```

## 14.4 常见追问

**Q：职责清晰到什么程度？**
A：每个 Agent 有唯一验收标准，且不与其他 Agent 重叠。

**Q：为什么强调星型？**
A：星型避免 Agent 两两通信导致的组合爆炸与死锁。

## 14.5 生产视角

把这 8 条固化为**设计评审清单**，未满足项需给出豁免理由，防止复杂度悄悄失控。

# 15. 小结

## 15.1 核心结论

- 多 Agent 收益与代价并存，按需使用；
- Supervisor 最常用，Swarm 慎用；
- 传摘要、限数量、防循环、全 Trace；
- 必须对比单 Agent 基线；
- 星型拓扑优于网状。

## 15.2 速记清单

| 主题 | 一句话 |
|------|--------|
| 选型 | 默认单 Agent，多 Agent 需举证 |
| 拓扑 | 星型优先，慎用网状 |
| 通信 | 传摘要与结构化字段 |
| 状态 | 外置、幂等、可恢复 |
| 护栏 | 步数 + 预算 + 循环检测 |
| 评测 | 结果 + 过程 + 效率 + 稳定 + 安全 |
