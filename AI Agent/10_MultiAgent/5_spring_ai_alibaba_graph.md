# 1. 概述

Spring AI Alibaba（阿里开源的 Spring AI 增强实现）提供 **Graph** 编排能力，用于构建多 Agent（智能体）与复杂工作流。它把多 Agent 协作建模为一张有向图：节点承载 Agent 或工具，边决定流转，状态在节点间传递。本文聚焦"多 Agent 编排"这一用途，图的通用机制请参见 `9_Frameworks/SpringAIAlibaba/4_graph.md`。

```
StateGraph = 节点 + 边 + 状态
```

## 1.1 定位与适用场景

结论先行：当任务需要多个职责不同的 Agent 协作、且协作流程可以被显式描述时，用 Graph 编排优于"让一个 Agent 自己想办法"。

适用场景：

- 流程稳定、步骤可枚举（如"检索 → 分析 → 写作 → 汇总"）。
- 需要并行分支再合并（Map-Reduce 式）。
- 需要人工介入、检查点、断点续跑。
- 需要精确可观测与可审计的每一步。

不适用场景：

- 流程高度动态、无法预先枚举（更适合 GOAP/规划式，见 `6_embabel.md`）。
- 单个 Agent 加工具即可完成，无需多角色。
- 对延迟极敏感且无法接受主管多次往返调用。

## 1.2 StateGraph 三要素

| 要素 | 英文 | 职责 | 常见实现 |
|------|------|------|----------|
| 节点 | Node | 执行一步逻辑，读写状态 | Agent、工具、纯函数 |
| 边 | Edge | 决定下一步流向 | 普通边、条件边 |
| 状态 | State | 跨节点共享的数据载体 | 可变 POJO / 不可变 record |

三者的关系可以概括为：

```
状态 ──► 节点读取 ──► 节点返回增量 ──► 合并回状态 ──► 边选择下一节点
  ▲                                                          │
  └──────────────────── 循环直到 END ────────────────────────┘
```

## 1.3 与其他编排方式对比

| 维度 | Graph 编排 | 纯提示词编排 | GOAP 规划 | 代码硬编码 |
|------|-----------|-------------|-----------|-----------|
| 流程可见性 | 高（图即文档） | 低 | 中 | 高 |
| 动态适应 | 中（条件边） | 高 | 高 | 低 |
| 可调试 | 高 | 低 | 中 | 高 |
| 人工介入 | 原生支持 | 难 | 难 | 需自研 |
| 学习成本 | 中 | 低 | 高 | 低 |

> 结论：稳定流程优先 Graph；动态目标优先规划式；简单场景不要过度设计。

# 2. 多 Agent 结构

多 Agent 编排最常见的是 **Supervisor（主管）模式**：一个主管 Agent 决定每一步交给哪个专业 Agent，专业 Agent 完成后回到主管，直到主管判定完成。

## 2.1 Supervisor 拓扑

```
              ┌──────────────┐
              │  Supervisor  │
              └──────┬───────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ 检索Agent│  │ 分析Agent│  │ 写作Agent│
   └────┬────┘  └────┬────┘  └────┬────┘
        └────────────┼────────────┘
                     ▼
                ┌─────────┐
                │  汇总    │
                └─────────┘
```

该拓扑的特点是"星型收敛"：所有专业 Agent 只与主管交互，专业 Agent 之间不直接通信，降低了耦合。

## 2.2 消息与状态流转

一次典型执行的状态演化：

| 轮次 | 当前节点 | 写入状态字段 | 说明 |
|------|---------|-------------|------|
| 0 | supervisor | nextAgent, step | 决定先检索 |
| 1 | retriever | documents | 写入检索结果 |
| 2 | supervisor | nextAgent, step | 决定分析 |
| 3 | analyzer | analysis | 写入分析结论 |
| 4 | supervisor | nextAgent, step | 决定写作 |
| 5 | writer | draft | 写入草稿 |
| 6 | supervisor | nextAgent=END | 判定完成 |
| 7 | synthesizer | finalAnswer | 汇总输出 |

## 2.3 角色划分原则

- 单一职责：一个 Agent 只做一类事，便于提示词与工具聚焦。
- 描述可判别：主管靠 `description` 选择 Agent，描述必须互斥且具体。
- 数量克制：Agent 越多，主管决策空间越大、越易选错，建议 3–6 个。
- 能力可回退：关键 Agent 失败时应有降级路径，避免整图卡死。

# 3. 状态定义与设计

结论先行：状态是图编排的"契约"，字段设计直接决定并发安全、内存占用与可恢复性。字段宜少、宜明确、宜可合并。

## 3.1 状态字段设计

原始状态定义：

```java
public class MultiAgentState {
    private String question;
    private String nextAgent;
    private List<Document> documents;
    private String analysis;
    private String draft;
    private String finalAnswer;
    private int step;
}
```

使用 record 的不可变版本（JDK 17+）：

```java
public record MultiAgentState(
        String question,
        String nextAgent,
        List<Document> documents,
        String analysis,
        String draft,
        String finalAnswer,
        int step) {

    public static MultiAgentState initial(String question) {
        return new MultiAgentState(question, null, List.of(), null, null, null, 0);
    }
}
```

字段职责：

| 字段 | 类型 | 写入者 | 说明 |
|------|------|--------|------|
| question | String | 初始 | 用户原始问题 |
| nextAgent | String | supervisor | 路由信号 |
| documents | List | retriever | 检索结果 |
| analysis | String | analyzer | 分析结论 |
| draft | String | writer | 初稿 |
| finalAnswer | String | synthesizer | 最终输出 |
| step | int | supervisor | 步数计数 |

## 3.2 状态合并策略

节点通常返回"增量"，框架将其合并回状态。合并策略分两类。

### 3.2.1 覆盖式合并

新值直接替换旧值，适合 `analysis`、`draft` 这类"最新即有效"的字段。

```java
Map<String, Object> update = Map.of("analysis", newAnalysis);
```

### 3.2.2 追加式合并

新值追加到集合，适合 `documents`、`history` 这类需要累积的字段。

```java
// 伪代码：声明 reducer 为追加语义
stateSchema.addReducer("documents", (oldList, newList) -> {
    List<Document> merged = new ArrayList<>(oldList);
    merged.addAll(newList);
    return merged;
});
```

## 3.3 状态膨胀与治理

- 只放必要字段：不要把整段对话、全部工具返回塞进状态。
- 大对象外置：文档正文可存对象存储，状态里只放 ID 与摘要。
- 定期裁剪：历史超过 N 条时压缩为摘要。
- 版本兼容：状态结构变更时，旧检查点需要迁移或失效处理。

> 边界：状态字段一旦被检查点持久化，就形成了事实上的"数据模型"，随意改名会破坏可恢复性。

# 4. 构建 Supervisor 图

结论先行：图构建分三步——注册节点、连接边（含条件边）、编译。编译后得到可执行图。

## 4.1 节点与边注册

```java
StateGraph graph = new StateGraph<>(MultiAgentState.class)
        .addNode("supervisor", this::supervisorNode)
        .addNode("retriever", this::retrieverNode)
        .addNode("analyzer", this::analyzerNode)
        .addNode("writer", this::writerNode)
        .addNode("synthesizer", this::synthesizerNode)

        .addEdge(StateGraph.START, "supervisor")
        .addConditionalEdges("supervisor", this::route,
                Map.of("retriever", "retriever",
                       "analyzer", "analyzer",
                       "writer", "writer",
                       "FINISH", "synthesizer"))
        .addEdge("retriever", "supervisor")
        .addEdge("analyzer", "supervisor")
        .addEdge("writer", "supervisor")
        .addEdge("synthesizer", StateGraph.END)

        .compile();
```

要点：

- `START` 是虚拟入口，`END` 是虚拟出口。
- 专业 Agent 完成后统一回到 `supervisor`，形成"决策-执行-再决策"循环。
- 条件边的目标必须覆盖所有可能返回值，否则运行时找不到下一节点。

## 4.2 条件路由

条件边是一个函数：读状态，返回下一个节点名。

```java
private String route(MultiAgentState state) {
    if (state.getStep() > MAX_STEPS) {
        return "FINISH";
    }
    String next = state.getNextAgent();
    return (next == null || "FINISH".equals(next)) ? "FINISH" : next;
}
```

路由函数应当是**纯函数**：只依赖入参状态，不产生副作用，便于测试与重放。

## 4.3 编译与执行

```java
// 编译
CompiledGraph compiled = graph.compile();

// 同步执行
MultiAgentState result = compiled.invoke(MultiAgentState.initial("问题"));

// 带线程 ID 执行（便于检查点与恢复）
MultiAgentState result2 = compiled.invoke(
        MultiAgentState.initial("问题"),
        RunnableConfig.builder().threadId("session-1").build());
```

> 边界：`invoke` 是阻塞调用，Web 场景应放在受控线程池或使用流式接口，避免占满容器线程。

# 5. Supervisor 节点实现

结论先行：主管节点是全图的"大脑"，其输出质量决定整图成败。实现上分三部分：构造提示词、调用模型、解析结构化结果。

## 5.1 决策提示词设计

主管提示词必须包含三块信息：可用 Agent 列表、当前任务、已完成历史。

### 5.1.1 可用 Agent 描述

```java
private String describeAgents() {
    return """
            - retriever：检索企业知识库，返回相关文档
            - analyzer：基于文档做数据分析并给出洞察
            - writer：基于分析结论撰写结构化报告
            """;
}
```

描述要"可判别"：写清输入、输出与边界，避免两个 Agent 职责重叠。

### 5.1.2 历史压缩

历史越长，主管越容易迷失且成本越高。压缩策略：

| 策略 | 说明 | 适用 |
|------|------|------|
| 截断 | 只保留最近 N 轮 | 简单、成本低 |
| 摘要 | 用模型把旧历史压缩成摘要 | 长任务 |
| 结构化 | 只保留"谁做了什么 + 结论" | 推荐 |
| 向量检索 | 按相关性召回历史片段 | 超长历史 |

```java
private String buildHistory(MultiAgentState state) {
    return """
            已检索文档数：%d
            分析结论：%s
            草稿长度：%d 字
            """.formatted(
            state.getDocuments().size(),
            abbreviate(state.getAnalysis(), 500),
            length(state.getDraft()));
}
```

## 5.2 结构化输出解析

主管必须输出机器可解析的结构。优先使用结构化输出能力，退化时用 JSON + 校验。

```java
public record SupervisorDecision(
        String action,
        String instruction,
        String reason) {}

private Map<String, Object> supervisorNode(MultiAgentState state) {
    // 主管决策下一步交给谁
    String next = supervisorAgent.decide(state);
    return Map.of("nextAgent", next, "step", state.getStep() + 1);
}

private String route(MultiAgentState state) {
    if (state.getStep() > MAX_STEPS) {
        return "FINISH";
    }
    return state.getNextAgent();
}
```

伪代码：带校验的解析。

```java
SupervisorDecision decision = parse(rawOutput);
if (!VALID_ACTIONS.contains(decision.action())) {
    // 非法动作：回退到规则兜底或直接 FINISH
    return "FINISH";
}
```

## 5.3 步数上限与终止

终止条件应多重设置，任一满足即结束：

- 主管显式输出 `FINISH`。
- `step` 超过 `MAX_STEPS`。
- 连续两轮无状态变化（死循环探测）。
- 总耗时或总 Token 超预算。

```java
private static final int MAX_STEPS = 8;
```

> 边界：只靠"主管自觉结束"不可靠，LLM 可能陷入"再检索一次更好"的循环。硬上限是必须的。

# 6. 各 Agent 节点实现

结论先行：专业 Agent 节点应尽量"薄"——只负责调用能力并返回状态增量，不负责路由决策。

## 6.1 检索 Agent

```java
private Map<String, Object> retrieverNode(MultiAgentState state) {
    List<Document> docs = retriever.retrieve(state.getQuestion());
    return Map.of("documents", docs);
}
```

边界与坑：

- 空结果要显式处理，返回空列表而非 `null`，否则下游 NPE。
- 检索失败应记录错误到状态，供主管决策是否重试。

## 6.2 分析 Agent

```java
private Map<String, Object> analyzerNode(MultiAgentState state) {
    String analysis = analyzer.analyze(state.getQuestion(), state.getDocuments());
    return Map.of("analysis", analysis);
}
```

要点：

- 输入文档可能超长，先做截断/重排，避免超出上下文窗口。
- 要求结构化输出（结论 + 依据 + 置信度），便于下游写作与审核。

## 6.3 写作 Agent

```java
private Map<String, Object> writerNode(MultiAgentState state) {
    String draft = writer.write(state.getQuestion(), state.getAnalysis());
    return Map.of("draft", draft);
}
```

要点：

- 写作只依赖分析结论，避免把原始文档再次塞入造成干扰。
- 输出长度可控，必要时分段生成再拼接。

## 6.4 汇总 Agent

```java
private Map<String, Object> synthesizerNode(MultiAgentState state) {
    String answer = synthesizer.combine(
            state.getQuestion(), state.getDraft(), state.getDocuments());
    return Map.of("finalAnswer", answer);
}
```

> 边界：汇总节点是"最后一公里"，应加事实一致性检查，避免把中间错误原样输出。

# 7. 并行子任务

结论先行：当多个子任务互不依赖时，扇出并行再扇入合并，可显著降低总延迟；但并发写同一状态字段会引入冲突。

## 7.1 扇出与扇入

```java
// 多个无依赖 Agent 并行
.addEdge("supervisor", "retriever")
.addEdge("supervisor", "analyzer")       // 并行
.addEdge("retriever", "merge")
.addEdge("analyzer", "merge")
```

```
             ┌──────────────┐
             │  supervisor  │
             └──────┬───────┘
              ┌─────┴─────┐
              ▼           ▼
        ┌─────────┐  ┌─────────┐
        │retriever│  │analyzer │   （并行）
        └────┬────┘  └────┬────┘
             └─────┬─────┘
                   ▼
              ┌─────────┐
              │  merge  │
              └─────────┘
```

## 7.2 并发合并冲突

| 场景 | 冲突 | 解决 |
|------|------|------|
| 两分支写同一标量 | 后写覆盖先写 | 改为各写独立字段 |
| 两分支写同一列表 | 顺序不确定 | 用追加 reducer |
| 合并依赖顺序 | 结果不稳定 | 合并节点做排序 |

```java
// 伪代码：追加 reducer 保证并发写入不丢失
stateSchema.addReducer("documents", appendReducer());
```

## 7.3 并行度与限流

- 并行分支数应受线程池与下游 QPS 限制。
- 对 LLM 调用加信号量，避免瞬时打爆配额。
- 单个分支超时要有兜底，不能拖垮整个 merge。

> 边界：并行不是越多越好。分支数超过 CPU/配额后，收益递减且故障面扩大。

# 8. 流式与可观测

结论先行：多 Agent 系统若无可观测性，故障几乎无法定位。至少要有"每步事件 + 全链路 Trace + 关键指标"。

## 8.1 流式事件

```java
// 流式获取每步
graph.stream(initialState).subscribe(event ->
        log.info("节点 {} 完成，耗时 {}ms",
                event.node(), event.durationMs()));
```

流式接口的价值：

- 前端可实时展示"正在检索/正在分析"。
- 长任务可提前返回中间结果。
- 出错时可定位到具体节点。

## 8.2 Trace 与指标

```
Trace: 用户请求
├── Span: supervisor（第 1 轮）
├── Span: retriever
├── Span: supervisor（第 2 轮）
├── Span: analyzer
├── Span: supervisor（第 3 轮）
├── Span: writer
└── Span: synthesizer
```

关键指标：

| 指标 | 含义 | 告警阈值建议 |
|------|------|-------------|
| step 数 | 每请求轮次 | 接近 MAX_STEPS |
| 节点耗时 | 各节点 P95 | 超 SLA |
| 失败率 | 节点异常比例 | > 1% |
| Token 消耗 | 每请求成本 | 超预算 |
| 终止原因 | FINISH/超步/超时 | 超步占比高需排查 |

## 8.3 日志与审计

- 每次主管决策记录 `action/instruction/reason`。
- 专业 Agent 记录输入摘要与输出摘要，避免记录敏感全文。
- 用户可见输出与内部推理分离，审计只留必要信息。

> 边界：日志中若包含用户隐私或密钥，会带来合规风险；务必脱敏。

# 9. 人工介入与检查点

结论先行：当流程涉及高风险动作或不可逆操作时，图必须支持"中断-确认-恢复"，而不是一路跑到底。

## 9.1 中断与恢复

检查点（Checkpoint）把每一步状态持久化，使执行可暂停、可恢复。

```java
// 编译时启用检查点
CompiledGraph compiled = graph.compile(
        CheckpointConfig.builder()
                .checkpointSaver(new PostgresSaver(dataSource))
                .build());

// 在关键节点前中断，等待人工确认
CompiledGraph guarded = graph.compile(
        CompileConfig.builder()
                .interruptBefore("synthesizer")
                .build());

// 恢复执行
MultiAgentState resumed = guarded.invoke(null,
        RunnableConfig.builder().threadId("session-1").build());
```

| 能力 | 说明 | 依赖 |
|------|------|------|
| 暂停 | 指定节点前中断 | interruptBefore |
| 恢复 | 用同一 threadId 继续 | 检查点存储 |
| 回放 | 从任意检查点重跑 | 状态不可变 |
| 审计 | 每步状态留痕 | 持久化策略 |

## 9.2 审批节点设计

审批节点本质是一个"等待外部输入"的节点：

```java
// 伪代码：审批节点阻塞等待人工决策
private Map<String, Object> approvalNode(MultiAgentState state) {
    ApprovalResult result = approvalGateway.request(
            state.getDraft(), Duration.ofHours(2));
    if (result.approved()) {
        return Map.of("approved", true);
    }
    return Map.of("approved", false, "nextAgent", "writer");
}
```

要点：

- 审批要有超时，超时按拒绝处理，避免永久挂起。
- 审批界面只展示必要信息，避免泄露内部推理。
- 审批结果写回状态，供后续条件边判断。

## 9.3 幂等与副作用

> 边界：检查点恢复会**重放**节点，带副作用的节点（写库、发邮件、下单）必须幂等。

```java
// 幂等键：同一逻辑操作重复执行只生效一次
String idempotencyKey = "report:" + state.getThreadId() + ":" + state.getStep();
if (!idempotencyStore.firstTime(idempotencyKey)) {
    return Map.of(); // 已执行，跳过
}
```

生产建议：

- 把"生成"与"提交"分离，提交动作放在人工审批之后。
- 副作用节点单独建节点，便于定位与重试。
- 记录幂等键与执行结果，便于对账。

# 10. 生产注意事项

结论先行：多 Agent 系统的生产风险集中在"停不下来、决策错、成本高、看不见、不安全"五类。

## 10.1 稳定性风险

| 风险 | 建议 |
|------|------|
| 循环无终止 | 步数上限 |
| 主管决策错误 | 主管用强模型 |
| 状态膨胀 | 只放必要字段 |
| 无 Trace | 记录每步 |
| 成本高 | 限制 Agent 数 |

补充：

- 节点异常要有 try/catch，写入错误状态而非直接抛出。
- 外部依赖（检索、数据库）加超时与熔断。
- 部署多实例时，检查点存储需共享（如 Postgres/Redis）。

## 10.2 成本与性能

- 主管每轮一次模型调用，轮次越多成本越高；用规则预筛明显意图。
- 专业 Agent 可用小模型，主管用强模型。
- 相似请求缓存主管决策与检索结果。
- 并行化无依赖分支压缩延迟。

## 10.3 安全与权限

- 每个 Agent 按最小权限授予工具。
- 工具入参做校验与转义，防注入。
- 输出做敏感信息过滤与合规检查。
- 高风险动作（写库、发邮件）走人工审批节点。

# 11. 小结

## 11.1 要点回顾

- Spring AI Alibaba Graph 用图编排多 Agent。
- Supervisor 模式：主管决策 + 条件路由 + 回到主管。
- 支持并行、流式、可观测。
- 必须有步数上限。
- 主管 Agent 建议用强模型。
- 状态是核心契约，字段宜少、合并策略要明确。

## 11.2 落地清单

- [ ] 状态字段最小化，定义好合并 reducer。
- [ ] 主管提示词包含 Agent 描述、任务、压缩历史。
- [ ] 条件路由覆盖所有返回值，且为纯函数。
- [ ] 设置 MAX_STEPS、超时、Token 预算三重终止。
- [ ] 接入流式事件与全链路 Trace。
- [ ] 并行分支做限流与合并冲突处理。
- [ ] 每个 Agent 最小权限，高风险动作人工审批。
