# 1. 什么是 LangGraph4j

**结论先行**：**LangGraph4j** = LangGraph（Python）的 Java 实现，用**图**编排多 Agent 与复杂流程。它把"多 Agent 协作"从散落的 if-else 提升为可组合、可持久化、可中断的显式图。

**原理**：多 Agent 流程本质上是一个状态机。LangGraph4j 把状态机显式化：节点是处理步骤，边是流转规则，状态是共享数据。显式化的好处是——流程可画、可测、可断点续跑。

```
节点（Node）+ 边（Edge）+ 状态（State） = 图
```

## 1.1 定位

**结论先行**：LangGraph4j 不是 Agent 框架，而是**编排框架**。Agent 逻辑仍由你（或 Spring AI）实现，LangGraph4j 负责"谁在什么时候被调用"。

| 对比项 | 纯手写编排 | LangGraph4j |
|--------|------------|-------------|
| 流程表达 | if-else / switch | 显式图 |
| 状态管理 | 手工传递 | State + Schema |
| 中断恢复 | 需自行实现 | 检查点原生支持 |
| 可视化 | 无 | 图结构可导出 |
| 循环控制 | 手工计数 | 条件边 + 递归上限 |

## 1.2 与 LangGraph 的关系

**原理**：LangGraph4j 对齐 Python 版 LangGraph 的核心抽象（StateGraph、Node、Edge、Checkpoint），但并非逐 API 移植，命名与泛型设计带有 Java 风格。

**边界与坑**：不要把 Python LangGraph 的代码逐行翻译——两者的 `compile`、`invoke` 签名与状态合并语义存在差异，应以 Java 文档为准。

## 1.3 核心概念

| 概念 | 说明 |
|------|------|
| `StateGraph` | 状态图，承载节点与边 |
| `Node` | 处理步骤 |
| `Edge` | 流转（普通/条件） |
| `State` | 共享状态 |
| `Checkpoint` | 状态检查点 |
| `CompiledGraph` | 编译后的可执行图 |

**生产建议**：把图当作"代码"管理——版本化、评审、写测试，而不是当成配置随手改。

# 2. 核心概念详解

**结论先行**：理解 LangGraph4j 的关键是搞清"状态如何流动、边如何决定流向、节点返回什么"。

## 2.1 StateGraph

**原理**：`StateGraph` 是可变构建器（builder），`addNode`/`addEdge` 声明拓扑，`compile()` 冻结为不可变的 `CompiledGraph`。构建期与执行期分离，避免运行中改图。

```java
StateGraph<AgentState> graph = new StateGraph<>(AgentState.SCHEMA, AgentState::new)
        .addNode("retrieve", this::retrieveNode)
        .addNode("analyze", this::analyzeNode);
```

**边界与坑**：忘记 `compile()` 直接执行会编译期报错；`compile()` 之后不能再 `addNode`。

## 2.2 Node 与 Edge

**结论先行**：节点负责"做事并产出状态更新"，边负责"决定下一步去哪"。

### 2.2.1 普通边

**原理**：`addEdge(from, to)` 表示无条件流转。`StateGraph.START` 与 `StateGraph.END` 是保留节点。

```java
graph.addEdge(StateGraph.START, "retrieve")
     .addEdge("retrieve", "analyze");
```

### 2.2.2 条件边

**原理**：`addConditionalEdges(from, router, mapping)` 中，`router` 读取状态返回一个键，`mapping` 把键映射到目标节点。

```java
graph.addConditionalEdges("write", this::shouldRevise,
        Map.of("revise", "review", "done", StateGraph.END));
```

**边界与坑**：`router` 返回的键若不在 `mapping` 中，运行时会报错；务必保证路由函数穷尽所有分支。

## 2.3 State 与 Schema

**原理**：Schema 定义状态的字段及其"合并策略"（reducer）。默认策略是覆盖，列表字段通常需要追加策略。

```java
public static final Schema<AgentState> SCHEMA = Schema.builder(AgentState.class)
        .field("question")
        .field("documents", (oldVal, newVal) -> append(oldVal, newVal))
        .field("analysis")
        .field("answer")
        .field("retryCount")
        .build();
```

**边界与坑**：并发写入同一字段时，合并策略决定结果；若列表不加 append，后写会覆盖前写，导致并行结果丢失。

# 3. 定义状态

**结论先行**：状态设计是 LangGraph4j 的地基。只放"跨节点需要共享的数据"，其余放在节点内部。

**原理**：状态会被检查点序列化，因此必须可序列化；同时它也是各节点之间唯一的通信通道。

## 3.1 状态设计

```java
public class AgentState {
    private String question;
    private List<Document> documents;
    private String analysis;
    private String answer;
    private int retryCount;
}
```

| 字段 | 类型 | 说明 | 合并策略 |
|------|------|------|----------|
| question | String | 用户问题 | 覆盖 |
| documents | List | 检索文档 | 追加 |
| analysis | String | 分析结论 | 覆盖 |
| answer | String | 最终答案 | 覆盖 |
| retryCount | int | 重试次数 | 覆盖（递增） |

**边界与坑**：状态字段过多会导致每次检查点都写入大对象，持久化成本与延迟上升。

**生产建议**：大对象（如原始文档）只存 ID 或引用，正文放外部存储。

## 3.2 状态合并策略

**结论先行**：默认覆盖；列表用追加；计数用求和；需要自定义时写 reducer 函数。

```java
private static List<Document> append(List<Document> oldVal, List<Document> newVal) {
    if (oldVal == null) {
        return newVal == null ? List.of() : List.copyOf(newVal);
    }
    if (newVal == null) {
        return oldVal;
    }
    List<Document> merged = new ArrayList<>(oldVal);
    merged.addAll(newVal);
    return List.copyOf(merged);
}
```

**边界与坑**：append 策略在"节点重跑"时可能重复追加同一批文档（至少一次语义），需要按 ID 去重。

**生产建议**：reducer 保持纯函数、无副作用、幂等，便于重放。

# 4. 构建图

**结论先行**：构建图分三步——声明节点、连接边、编译。建议先画草图再写代码。

**原理**：`addNode` 注册名字与处理函数，`addEdge`/`addConditionalEdges` 声明流转，`compile` 校验图合法性（如可达性、终点存在）并生成可执行图。

## 4.1 基本构建

```java
StateGraph<AgentState> graph = new StateGraph<>(AgentState.SCHEMA, AgentState::new)
        .addNode("retrieve", this::retrieveNode)
        .addNode("analyze", this::analyzeNode)
        .addNode("write", this::writeNode)
        .addNode("review", this::reviewNode)

        .addEdge(StateGraph.START, "retrieve")
        .addEdge("retrieve", "analyze")
        .addEdge("analyze", "write")
        .addConditionalEdges("write", this::shouldRevise,
                Map.of("revise", "review", "done", StateGraph.END))
        .addEdge("review", "write")

        .compile();
```

## 4.2 编译与配置

```java
// 带检查点的编译
CompiledGraph<AgentState> compiled = graph.compile(
        CheckpointConfig.builder()
                .checkpointSaver(new PostgresSaver(dataSource))
                .build());
```

**边界与坑**：`review → write` 构成回边，如果没有 `shouldRevise` 的 `retryCount` 上限，图会无限循环。

**生产建议**：任何回边都必须配一个"确定性终止条件"，不能只依赖模型判断。

# 5. 节点实现

**结论先行**：节点是纯函数式的一步：读状态、做事、返回"状态增量"。不要在图里写业务分支，分支交给边。

**原理**：节点签名通常接收状态并返回 `Map<String, Object>`（要更新的字段）。返回空 Map 表示不修改状态。

## 5.1 检索节点

```java
private Map<String, Object> retrieveNode(AgentState state) {
    List<Document> docs = retriever.retrieve(state.getQuestion());
    return Map.of("documents", docs);
}
```

## 5.2 分析节点

```java
private Map<String, Object> analyzeNode(AgentState state) {
    String analysis = analyzer.analyze(state.getQuestion(), state.getDocuments());
    return Map.of("analysis", analysis);
}
```

## 5.3 节点约定

| 约定 | 说明 |
|------|------|
| 单一职责 | 一个节点做一件事 |
| 只返回增量 | 不返回完整状态 |
| 幂等 | 重跑结果一致 |
| 可观测 | 记录入参、出参、耗时 |
| 异常可控 | 抛出可识别的领域异常 |

**边界与坑**：节点内做重试会绕过图的检查点，重跑时产生副作用叠加。

**生产建议**：把重试交给图（条件边回环），而不是节点内部 while 循环。

# 6. 条件边

**结论先行**：条件边是图"智能"的来源，也是循环控制的唯一入口。路由函数必须确定、可测、穷尽。

## 6.1 路由函数

```java
private String shouldRevise(AgentState state) {
    if (state.getRetryCount() < 2 && !isGood(state.getAnswer())) {
        return "revise";
    }
    return "done";
}
```

**边界与坑**：路由函数调用 LLM 会引入不确定性，且难以测试。优先用规则，LLM 只做兜底。

## 6.2 循环控制

**结论先行**：循环 = 条件边 + 回边。必须用计数器封顶。

```java
private Map<String, Object> reviewNode(AgentState state) {
    return Map.of("retryCount", state.getRetryCount() + 1);
}

private String shouldRevise(AgentState state) {
    if (state.getRetryCount() >= 3) {
        return "done";                 // 硬上限，防无限循环
    }
    return isGood(state.getAnswer()) ? "done" : "revise";
}
```

**生产建议**：把"最大循环次数"作为图级常量集中管理，并在编译期校验每条回边都受其约束。

# 7. 检查点（持久化）

**结论先行**：检查点让图执行可中断、可恢复、可回放。生产环境应默认开启。

**原理**：每个超级步（super-step）结束后，把状态快照写入 `CheckpointSaver`。恢复时按 `threadId` 载入最近快照，从断点继续。

## 7.1 机制

```
threadId=session-1
  step0: {question}                 ← checkpoint
  step1: {question, documents}      ← checkpoint
  step2: {question, documents, analysis}  ← checkpoint
  step3: {..., answer}
                    ↑
              崩溃后从此恢复
```

| 概念 | 说明 |
|------|------|
| threadId | 一次会话/执行链的唯一标识 |
| checkpoint | 某个超级步后的状态快照 |
| saver | 存储后端（内存/Postgres/Redis） |

## 7.2 实现

```java
// 编译时启用检查点
CompiledGraph<AgentState> compiled = graph.compile(
        CheckpointConfig.builder()
                .checkpointSaver(new PostgresSaver(dataSource))
                .build());

// 执行（自动保存检查点）
AgentState result = compiled.invoke(initialState,
        RunnableConfig.builder().threadId("session-1").build());

// 恢复
AgentState resumed = compiled.invoke(null,
        RunnableConfig.builder().threadId("session-1").build());
```

## 7.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 状态不可序列化 | 检查点写入失败 | 状态字段保持可序列化 |
| threadId 冲突 | 会话串数据 | threadId 全局唯一 |
| 存储膨胀 | 检查点无限增长 | 定期清理历史检查点 |
| 非幂等副作用 | 恢复后重复执行 | 副作用节点做幂等 |
| 版本不兼容 | 旧检查点无法反序列化 | 状态字段版本化 |

**生产建议**：生产用 Postgres/Redis 而非内存 saver；为检查点设 TTL（生存时间）与归档策略。

# 8. 人工介入

**结论先行**：高风险节点应支持"中断—人工确认—继续"。这是 LangGraph4j 相对手写编排最大的工程优势。

**原理**：`interruptBefore`/`interruptAfter` 在指定节点前后暂停执行并保存检查点；人工确认后以同一 `threadId` 继续。

## 8.1 中断机制

```
执行 ──▶ [retrieve] ──▶ [analyze] ──║ 中断 ║──▶ 人工确认 ──▶ [write] ──▶ END
                                     ↑
                              状态已持久化
```

## 8.2 实现

```java
// 在关键节点中断，等待人工确认
CompiledGraph<AgentState> compiled = graph.compile(
        CompileConfig.builder()
                .interruptBefore("write")          // 在 write 前中断
                .build());

// 执行到中断点
AgentState state = compiled.invoke(input, config);

// 人工确认后继续（传入人工修改后的状态）
AgentState result = compiled.invoke(state, config);
```

**边界与坑**：中断期间状态停留在存储中，可能被并发修改；需要乐观锁或版本号避免"人工改一半被覆盖"。

**生产建议**：把"哪些节点可中断"作为显式策略，并对人工修改做审计留痕。

# 9. 流式执行

**结论先行**：流式让前端能实时展示"当前进行到哪个节点"，显著改善长链路任务的体验。

**原理**：`stream` 按超级步产出事件，每个事件携带节点名与可选的状态增量。

## 9.1 事件流

| 事件类型 | 含义 |
|----------|------|
| node | 某节点执行完成 |
| state | 状态发生更新 |
| token | 模型逐 Token 输出（视实现） |

## 9.2 实现

```java
compiled.stream(input, config)
        .forEach(event -> {
            log.info("节点 {} 完成", event.node());
            if (event.hasState()) {
                // 处理状态更新
                AgentState snapshot = event.state();
                progressReporter.report(snapshot);
            }
        });
```

**边界与坑**：流式事件是"至少一次"投递，前端需按节点名去重；大状态随事件全量下发会拖慢流。

**生产建议**：只向下游推送"状态摘要"而非完整状态，敏感字段（如原始文档）不下发。

# 10. 多 Agent 示例

**结论先行**：用一张图表达 Supervisor 模式：`supervisor` 节点决策，专家节点执行，执行后回到 `supervisor`。

## 10.1 Supervisor 图

```
            ┌────────────┐
     START ─▶│ supervisor │◀────────┐
            └─────┬──────┘         │
        ┌─────────┼─────────┐      │
        ▼         ▼         ▼      │
   [researcher][writer]   [END]    │
        │         │                │
        └─────────┴────────────────┘
```

## 10.2 完整代码

```java
StateGraph<AgentState> graph = new StateGraph<>(AgentState.SCHEMA, AgentState::new)
        .addNode("supervisor", this::supervisorNode)
        .addNode("researcher", this::researcherNode)
        .addNode("writer", this::writerNode)
        .addConditionalEdges("supervisor", this::routeFromSupervisor,
                Map.of("researcher", "researcher",
                       "writer", "writer",
                       "done", StateGraph.END))
        .addEdge("researcher", "supervisor")
        .addEdge("writer", "supervisor")
        .compile();

private Map<String, Object> supervisorNode(AgentState state) {
    String next = supervisor.decide(state.getQuestion(), state.getHistory());
    return Map.of("next", next, "steps", state.getSteps() + 1);
}

private String routeFromSupervisor(AgentState state) {
    if (state.getSteps() >= MAX_STEPS) {
        return "done";                     // 硬上限
    }
    return state.getNext();
}
```

**边界与坑**：`supervisor` 是唯一汇聚点，若专家节点忘记连回 `supervisor`，流程会提前结束。

**生产建议**：把"所有专家节点都必须回连 supervisor"写成图构建期的断言。

# 11. 生产注意事项

**结论先行**：LangGraph4j 的图越复杂，越需要"模块化 + 终止条件 + 持久化 + 可观测"四件套。

## 11.1 风险与对策

| 风险 | 建议 |
|------|------|
| 图过于复杂 | 模块化，拆子图 |
| 无限循环 | 步数上限 |
| 状态膨胀 | 只放必要字段 |
| 无检查点 | 启用持久化 |
| 无 Trace | 记录每步 |
| 状态不可序列化 | 字段可序列化 + 版本化 |
| 副作用重复 | 节点幂等 |

## 11.2 上线检查清单

- 每条回边都有确定性终止条件。
- 状态字段均可序列化且已版本化。
- 已配置持久化 CheckpointSaver。
- 每个节点有独立 Trace 与耗时指标。
- 高风险节点配置人工中断。
- 图按业务域拆分为子图。
- 恢复执行经幂等性验证。

**边界与坑**：最危险的组合是"回边 + 非幂等副作用 + 无检查点"，一次崩溃就可能重复扣款、重复发信。

**生产建议**：把"副作用节点"单独标记，强制其幂等，并优先配置中断与人工确认。

# 12. 小结

- LangGraph4j 用图编排多 Agent 与复杂流程。
- 核心：StateGraph + Node + Edge + State。
- 支持条件边、检查点、人工介入、流式。
- 状态合并策略决定并发写入的正确性。
- 检查点让执行可中断可恢复。
- 任何回边都要有确定性终止条件。
- 图要模块化、状态要可序列化、副作用要幂等。
