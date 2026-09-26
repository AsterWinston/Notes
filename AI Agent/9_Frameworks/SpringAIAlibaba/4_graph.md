# 1. 为什么需要 Graph

## 1.1 ChatClient 的边界

`ChatClient` 擅长**单次调用**：给它提示，它返回答案。但当流程变成"多步骤 + 条件分支 + 并行 + 循环 + 状态传递"时，用 `ChatClient` 手写 if/else 会迅速失控：

- 控制流散落在业务代码里，难以阅读与维护；
- 状态靠局部变量传递，跨步骤易丢失；
- 无法可视化、难以做可观测与中断恢复。

## 1.2 Graph 解决的问题

Graph（图编排）把流程**显式建模**为"节点 + 边 + 状态"：

```
Graph = 节点（Node）+ 边（Edge）+ 状态（State）
```

- 节点：一个处理步骤（LLM 调用、工具调用、逻辑判断）；
- 边：节点间的流转（顺序、条件、并行）；
- 状态：在节点间传递的共享数据。

> 用图描述流程，控制流一目了然，且天然支持可视化、追踪与恢复。

## 1.3 与其他编排方式对比

| 方式 | 结构 | 适用 |
|------|------|------|
| 手写 if/else | 隐式 | 极简单流程 |
| 链式（Chain） | 线性 | 固定顺序 |
| Graph | 有向图 | 多分支、并行、循环 |
| 工作流引擎 | 有向图 + 持久化 | 长流程、需恢复 |

Spring AI Alibaba 的 Graph 介于"链式"与"工作流引擎"之间，轻量且够用。

# 2. 核心概念

## 2.1 节点（Node）

节点是图中的一个处理单元，通常是一个方法，接收状态、返回状态更新。

### 2.1.1 节点签名

```java
private Map<String, Object> classifyNode(AgentState state) {
    String intent = intentClassifier.classify(state.getQuestion());
    return Map.of("intent", intent);
}
```

签名约定：**入参是状态，出参是状态增量（`Map`）**，不返回完整状态。

### 2.1.2 四类节点

| 类型 | 职责 | 示例 |
|------|------|------|
| LLM 节点 | 生成/判断 | 意图分类、答案生成 |
| 工具节点 | 调用外部工具 | 查订单、检索 |
| 逻辑节点 | 纯代码加工 | 路由、格式化 |
| 子图节点 | 复用子图 | 多 Agent 协作 |

## 2.2 边（Edge）

边定义节点之间的流转规则：

| 边类型 | 作用 |
|--------|------|
| 顺序边 | A → B 固定流转 |
| 条件边 | 按状态选择下一个节点 |
| 起始/结束边 | START / END |

### 2.2.1 条件边的分支映射

条件边 = **路由函数 + 分支映射表**：

```java
.addConditionalEdges("classify", this::routeByIntent, Map.of(
        "NEED_RAG", "retrieve",
        "NEED_TOOL", "tool",
        "DIRECT", "generate"))
```

路由函数返回分支名，映射表把分支名翻译为目标节点。

## 2.3 状态（State）

状态是节点间共享的数据载体，通常是一个 POJO：

```java
public class AgentState {
    private String question;
    private String intent;
    private List<Document> documents;
    private String toolResult;
    private String answer;
    private int retryCount;
    // getters / setters
}
```

## 2.4 三者的关系

```
状态：在节点间传递的共享数据
节点：读写状态的函数
边  ：决定"下一个节点是谁"
```

```
        START
          │
          ▼
      ┌────────┐
      │ 意图识别 │
      └───┬────┘
    ┌─────┼─────┐
    ▼     ▼     ▼
  检索   工具  直接回答
    └─────┼─────┘
          ▼
      ┌────────┐
      │  汇总   │
      └───┬────┘
          ▼
         END
```

# 3. 定义状态

## 3.1 状态类设计

```java
public class AgentState {
    private String question;
    private String intent;
    private List<Document> documents;
    private String toolResult;
    private String answer;
    private int retryCount;

    // getters / setters
}
```

## 3.2 状态字段的取舍

状态是**节点间的契约**，字段应"够用且精简"：

| 字段类型 | 建议 |
|----------|------|
| 输入（question） | 必放 |
| 中间结果（intent、documents） | 放，供下游使用 |
| 控制变量（retryCount） | 放，驱动循环 |
| 大对象（原始文件、全量历史） | 不放，改为引用/摘要 |

## 3.3 状态的可变性

- 节点返回的是**状态更新**（增量），而非整个新状态；
- 未返回的字段保持原值；
- 避免在节点里随意修改共享对象，尽量"返回更新"。

### 3.3.1 增量更新的语义

```
状态 s  = {question:"...", intent:null}
节点返回 {intent:"POLICY"}
状态 s' = {question:"...", intent:"POLICY"}   // 其余字段不变
```

### 3.3.2 并发写冲突

并行节点若返回**同一字段**，最终值不确定。约定：并行分支只写各自独立字段，由汇聚节点合并。

# 4. 构建图

## 4.1 创建 StateGraph

```java
StateGraph graph = new StateGraph<>(AgentState.class);
```

泛型参数是状态类，框架据此管理状态的读写。

## 4.2 添加节点

```java
graph.addNode("classify", this::classifyNode)
     .addNode("retrieve", this::retrieveNode)
     .addNode("tool", this::toolNode)
     .addNode("generate", this::generateNode);
```

### 4.2.1 节点命名

节点名是**日志与 Trace 的标识**，应使用动词短语（`classify`、`retrieve`），避免 `node1`、`step2` 这类无语义命名。

## 4.3 添加边与条件边

```java
graph.addEdge(StateGraph.START, "classify")
     .addConditionalEdges("classify", this::routeByIntent, Map.of(
             "NEED_RAG", "retrieve",
             "NEED_TOOL", "tool",
             "DIRECT", "generate"))
     .addEdge("retrieve", "generate")
     .addEdge("tool", "generate")
     .addEdge("generate", StateGraph.END);
```

## 4.4 编译与校验

```java
CompiledGraph compiled = graph.compile();
```

`compile()` 会校验：起始节点是否存在、所有节点是否可达、条件边的目标是否合法。**尽早编译**能在启动期暴露图结构错误。

# 5. 节点实现

## 5.1 意图识别节点

```java
private Map<String, Object> classifyNode(AgentState state) {
    String intent = intentClassifier.classify(state.getQuestion());
    return Map.of("intent", intent);
}
```

## 5.2 检索节点

```java
private Map<String, Object> retrieveNode(AgentState state) {
    List<Document> docs = retriever.retrieve(state.getQuestion());
    return Map.of("documents", docs);
}
```

## 5.3 工具节点

```java
private Map<String, Object> toolNode(AgentState state) {
    String result = toolService.execute(state.getQuestion());
    return Map.of("toolResult", result);
}
```

## 5.4 生成节点

```java
private Map<String, Object> generateNode(AgentState state) {
    String answer = chatClient.prompt()
            .user(buildPrompt(state))
            .call().content();
    return Map.of("answer", answer);
}
```

## 5.5 节点返回值约定

| 返回 | 含义 |
|------|------|
| `Map.of("k", v)` | 更新状态字段 k |
| 空 Map | 不更新任何字段 |
| 抛异常 | 中断执行，可配合重试/降级 |

> 节点应保持**单一职责**：一个节点只做一件事，便于复用与测试。

# 6. 条件路由

## 6.1 路由函数

```java
private String routeByIntent(AgentState state) {
    return switch (state.getIntent()) {
        case "POLICY" -> "NEED_RAG";
        case "ACTION" -> "NEED_TOOL";
        default -> "DIRECT";
    };
}
```

### 6.1.1 路由即纯函数

路由函数只读状态、返回分支名，**不产生副作用**。这样它可独立单测，也便于框架做静态分析。

## 6.2 路由与状态

路由函数是**纯函数**：只读状态、返回分支名，不产生副作用。这样路由逻辑可单测。

## 6.3 默认分支

条件边务必覆盖所有可能取值，提供 `default` 分支，否则运行时可能找不到下一节点。

# 7. 并行节点

## 7.1 并行结构

多个无依赖节点可并行执行：

```java
graph.addEdge("start", "retrieveDocs")
     .addEdge("start", "queryData")
     .addEdge("retrieveDocs", "merge")
     .addEdge("queryData", "merge");
```

## 7.2 汇聚（join）

并行分支在 `merge` 节点汇聚；该节点需能处理"部分分支先到"的情况。

## 7.3 注意事项

- 并行节点**不得互相写同一状态字段**，否则产生竞争；
- 并行分支应各自写入独立字段，由汇聚节点合并；
- 并行能降延迟，但会增加并发与限流压力。

# 8. 循环与重试

## 8.1 条件回边

生成后检查质量，不达标则回到生成节点：

```java
graph.addConditionalEdges("generate", state ->
        state.getRetryCount() < 3 && !isGood(state.getAnswer())
                ? "refine"
                : "end")
     .addEdge("refine", "generate");
```

## 8.2 重试上限

循环**必须有上限**（retryCount / maxSteps），否则会无限循环烧钱。

### 8.2.1 重试必须配合幂等

若回边会重新执行有副作用的节点（如下单），必须保证**幂等**，否则会重复产生副作用。读操作可安全重试，写操作需幂等键去重。

## 8.3 质量校验

`isGood` 可用规则（长度、关键词）或模型打分（LLM-as-Judge）实现。

# 9. 执行与流式

## 9.1 同步执行

```java
AgentState result = compiled.invoke(initialState);
```

## 9.2 流式执行

```java
compiled.stream(initialState).subscribe(event ->
        log.info("节点 {} 完成", event.node()));
```

流式可获取每一步事件，适合做进度反馈与 Trace。

### 9.2.1 流式事件字段

每个事件至少包含：节点名、状态增量、耗时、是否结束。据此可拼出完整的执行轨迹。

## 9.3 与 SSE 集成

```java
@GetMapping(value = "/agent/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream(@RequestParam String q) {
    return compiled.stream(new AgentState(q))
            .map(e -> ServerSentEvent.builder(e.node()).build());
}
```

> 流式响应的完整实现见 `14_ProductionEngineering/Architecture/2_webflux_sse.md`。

# 10. 生产注意事项

| 风险 | 建议 |
|------|------|
| 图过于复杂 | 模块化，拆子图 |
| 无终止条件 | 限制循环次数 |
| 状态膨胀 | 只放必要字段 |
| 无 Trace | 记录每步事件 |
| 节点失败 | 重试 + 降级 |
| 并行写冲突 | 分支写独立字段 |
| 路由漏分支 | 提供 default 分支 |

# 11. 小结

- Graph 用"节点 + 边 + 状态"编排多步流程，控制流显式化。
- 支持条件路由、并行、循环，适合复杂 Agent 流程与多 Agent 协作。
- 状态是节点间契约，宜精简、以增量更新方式传递。
- 循环必须设上限，并行分支避免写同一字段。
- 用流式事件做进度反馈与可观测；图要模块化、可测试。
