# 1. 继承 Spring AI 的抽象

结论先行：Spring AI Alibaba 不重复造轮子，它在 Spring AI 的抽象之上做“适配、编排、可观测”三层增强，因此掌握 Spring AI 的核心抽象是理解本框架的前提。

## 1.1 复用与增强的分层

### 1.1.1 Spring AI 提供的抽象

Spring AI 定义了与模型供应商无关的编程接口，主要抽象如下：

| 抽象 | 职责 | 典型实现 |
|------|------|----------|
| ChatModel | 对话补全（Chat Completion） | DashScopeChatModel |
| ChatClient | 面向开发者的门面（Facade） | 由 ChatModel 构建 |
| Prompt / Message | 提示词与消息载体 | SystemMessage、UserMessage |
| Advisor | 调用链上的横切拦截 | 日志、RAG、脱敏 |
| EmbeddingModel | 文本向量化 | DashScopeEmbeddingModel |
| VectorStore | 向量存储与相似检索 | SimpleVectorStore |
| ToolCallback | 工具/函数调用回调 | FunctionToolCallback |

这些抽象的共同特征是“面向接口编程”：业务只依赖接口，具体供应商实现由自动装配注入，从而做到可替换、可测试。

```java
@Service
public class AssistantService {

    private final ChatClient chatClient;

    public AssistantService(ChatClient.Builder builder) {
        this.chatClient = builder
                .defaultSystem("你是专业的中文助手。")
                .build();
    }

    public String ask(String question) {
        return chatClient.prompt().user(question).call().content();
    }
}
```

### 1.1.2 Spring AI Alibaba 的增强点

```
Spring AI 抽象（复用）
├── ChatModel / ChatClient
├── Prompt / Message
├── Advisor
├── VectorStore / EmbeddingModel
└── ToolCallback

Spring AI Alibaba 增强
├── DashScope 模型适配
├── Graph 编排
├── 可观测增强
└── 国内生态工具
```

- DashScope 模型适配：通义千问（Qwen）、text-embedding 系列开箱即用。
- Graph 编排：以“节点 + 边 + 状态”描述多步 Agent 流程。
- 可观测增强：与阿里云 ARMS（应用实时监控服务）打通。
- 国内生态工具：搜索、地图、RAG 等本土化工具接入。

## 1.2 依赖与自动装配

### 1.2.1 Maven 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0-M6</version>
</dependency>
```

> 版本号随 Spring AI 主版本演进，落地时以官方 BOM 为准，避免手工对齐版本。

### 1.2.2 自动装配

引入 starter 后，Spring Boot 自动装配（Auto-Configuration）读取 `spring.ai.dashscope.*` 配置，自动注册 `ChatModel`、`EmbeddingModel` 等 Bean，业务侧只需注入即可。

```java
@Configuration
public class AiConfig {

    @Bean
    ChatClient chatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel).build();
    }
}
```

若需要覆盖自动装配的默认 Bean，只需自行声明同名 Bean，Spring Boot 的 `@ConditionalOnMissingBean` 会让自定义配置优先。

## 1.3 边界与坑

| 坑 | 说明 | 应对 |
|----|------|------|
| 抽象强绑定 | 里程碑版本间有破坏性变更 | 升级前回归测试 |
| 绕过门面 | 直接用底层 HTTP 丢失 Advisor | 统一走 ChatClient |
| 网络与端点 | 国内访问需注意代理 | 显式配置端点 |

# 2. 模型适配

结论先行：模型适配把“供应商差异”收敛到统一接口，理想状态下切换模型时业务代码零改动。

## 2.1 DashScope ChatModel

### 2.1.1 编程式构建

```java
// 通义千问
ChatModel model = DashScopeChatModel.builder()
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-plus")
        .build();
```

### 2.1.2 通过 ChatClient 使用

```java
ChatClient client = ChatClient.builder(model)
        .defaultSystem("你是专业的中文助手。")
        .build();

String answer = client.prompt()
        .user("用一句话解释什么是 RAG。")
        .call()
        .content();
```

需要边生成边返回时，改用流式（Stream）：

```java
Flux<String> stream = client.prompt()
        .user("写一段关于向量检索的介绍。")
        .stream()
        .content();

stream.subscribe(System.out::print);
```

## 2.2 其他模型适配

### 2.2.1 多模型共存

同一应用可注册多个 `ChatClient`，按场景选用不同模型：

```java
@Bean
ChatClient fastClient(ChatModel model) {
    return ChatClient.builder(model)
            .defaultOptions(ChatOptions.builder().model("qwen-turbo").build())
            .build();
}
```

### 2.2.2 Embedding 与 Rerank

- EmbeddingModel 把文本转成向量，供 VectorStore 检索。
- 检索后可用重排（Rerank）模型对候选精排，提升精度。

| 能力 | 模型示例 | 用途 |
|------|----------|------|
| 对话 | qwen-plus / qwen-turbo | 生成 |
| 向量 | text-embedding-v3 | 召回 |
| 重排 | gte-rerank | 精排 |

三者常组合成 RAG 流水线：先用 Embedding 召回候选，再用 Rerank 精排，最后交给对话模型生成。

```
用户问题 ──▶ Embedding ──▶ 向量召回 Top-K
                              │
                              ▼
                          Rerank 精排 ──▶ 对话模型 ──▶ 答案
```

## 2.3 边界与坑

- 不同模型的分词器（Tokenizer）不同，Token 估算不可一概而论。
- 上下文窗口有限，超长输入需截断或分片。
- 流式（Stream）与非流式返回类型不同，注意 API 差异。

# 3. 配置驱动

结论先行：框架遵循 Spring Boot“约定优于配置”，绝大多数参数通过 `application.yml` 声明，便于多环境治理。

## 3.1 核心配置项

### 3.1.1 chat 配置

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
          temperature: 0.7
```

### 3.1.2 embedding 配置

```yaml
spring:
  ai:
    dashscope:
      embedding:
        options:
          model: text-embedding-v3
```

## 3.2 多环境与覆盖

### 3.2.1 Profile 切换

`application-dev.yml`、`application-prod.yml` 分别声明模型与配额，通过 `--spring.profiles.active` 切换。

### 3.2.2 运行时覆盖

```java
ChatOptions options = ChatOptions.builder()
        .temperature(0.2)
        .build();

String answer = client.prompt()
        .options(options)
        .user("请严格按 JSON 输出。")
        .call()
        .content();
```

## 3.3 边界与坑

| 坑 | 说明 | 应对 |
|----|------|------|
| 密钥硬编码 | 泄漏风险 | 环境变量/配置中心 |
| 参数不生效 | 优先级理解错 | 运行时 > 默认 |
| 环境串用 | 测试打到生产 | 隔离 Profile 与 Key |

# 4. Graph 编排（核心增强）

结论先行：Graph 是本框架相对 Spring AI 最核心的增强，用有向图显式描述多步流程，把“控制流”从代码中抽离出来。

## 4.1 Graph 三要素

### 4.1.1 节点

节点（Node）是一个处理步骤：LLM 调用、工具调用或纯逻辑，签名通常为 `state -> state`。

### 4.1.2 边

边（Edge）描述节点间流转，分三类：

| 边类型 | 作用 | 示例 |
|--------|------|------|
| 顺序边 | A 之后必到 B | addEdge("a", "b") |
| 条件边 | 按状态选分支 | addConditionalEdges |
| 并行边 | 同时触发多节点 | fan-out |

### 4.1.3 状态

状态（State）是节点间传递的数据载体，通常是一个 POJO 或 record。

```
Graph = 节点 + 边 + 状态

节点：一个处理步骤（LLM 调用、工具调用、逻辑）
边：节点间的流转（顺序、条件、并行）
状态：在节点间传递的数据
```

## 4.2 构建流程

### 4.2.1 定义状态

```java
public class AgentState {
    private String question;
    private String intent;
    private List<Document> documents;
    private String answer;
    // getters/setters
}
```

### 4.2.2 定义节点

```java
NodeAction classifyNode = state -> {
    state.setIntent(classifier.classify(state.getQuestion()));
    return state;
};
```

### 4.2.3 编译与执行

```java
StateGraph graph = new StateGraph(AgentState.class)
        .addNode("classify", classifyNode)
        .addNode("retrieve", retrieveNode)
        .addNode("generate", generateNode)
        .addNode("fallback", fallbackNode)
        .addEdge(START, "classify")
        .addConditionalEdges("classify", intentRouter,
                Map.of("need_rag", "retrieve",
                       "direct", "generate"))
        .addEdge("retrieve", "generate")
        .addEdge("generate", END)
        .compile();
```

编译后即可用初始状态驱动执行：

```java
AgentState initial = new AgentState();
initial.setQuestion("退货运费谁承担？");

AgentState result = graph.invoke(initial);
System.out.println(result.getAnswer());
```

## 4.3 边界与坑

- 图应无环或显式声明循环上限，否则可能死循环。
- 节点尽量幂等，便于失败重试。
- 复杂图应可视化，便于排查与评审。

# 5. 状态传递

结论先行：状态是 Graph 的“血脉”，设计好状态结构与合并策略，编排才能清晰可控。

## 5.1 状态模型

### 5.1.1 状态类设计

用 POJO 或 record 承载状态；如需在多分支间合并，则声明合并（Reducer）函数。

### 5.1.2 状态合并策略

| 策略 | 语义 | 适用 |
|------|------|------|
| 覆盖 | 后写覆盖前写 | 单值字段 |
| 追加 | 列表累加 | 消息、文档 |
| 归并 | 自定义合并 | 计数、去重 |

以“消息追加”为例，合并函数把新消息并入已有列表：

```java
public static AgentState mergeMessages(AgentState a, AgentState b) {
    List<Message> merged = new ArrayList<>(a.getMessages());
    merged.addAll(b.getMessages());
    a.setMessages(merged);
    return a;
}
```

## 5.2 并发与隔离

### 5.2.1 线程安全

并行节点会并发写状态，需保证状态对象或合并函数线程安全。

### 5.2.2 请求级隔离

每次请求创建独立状态实例，禁止把状态声明为单例 Bean。

## 5.3 边界与坑

- 状态过大导致序列化与拷贝开销高。
- 可变对象在节点间共享易产生副作用。
- 忘记合并策略会丢失并行分支结果。

# 6. 条件路由

结论先行：条件路由（Conditional Routing）让流程按运行时状态动态选路，是 Agent“自主决策”的落点。

## 6.1 路由函数

### 6.1.1 基于意图

```java
public String routeByIntent(AgentState state) {
    return switch (state.getIntent()) {
        case "POLICY" -> "retrieve";
        case "CHITCHAT" -> "direct";
        default -> "fallback";
    };
}
```

### 6.1.2 基于置信度

```java
public String routeByScore(AgentState state) {
    return state.getTopScore() >= 0.75 ? "generate" : "fallback";
}
```

## 6.2 多分支与默认分支

### 6.2.1 映射表

条件边用 `Map<分支名, 目标节点>` 声明，路由函数返回分支名。

```java
graph.addConditionalEdges(
        "classify",
        state -> routeByIntent(state),
        Map.of(
                "retrieve", "retrieve",
                "direct", "generate",
                "fallback", "fallback"));
```

### 6.2.2 兜底

必须提供默认分支，防止路由函数返回未知值时抛错。

## 6.3 边界与坑

| 坑 | 说明 |
|----|------|
| 无默认分支 | 未知返回值导致运行失败 |
| 路由副作用 | 路由函数应只读状态 |
| 条件过多 | 图复杂度爆炸，难维护 |

# 7. 与 Spring AI Advisor 的关系

结论先行：Advisor 与 Graph 不在同一层次——Advisor 管“单次调用的横切”，Graph 管“多步流程的编排”，二者互补而非替代。

## 7.1 对比

### 7.1.1 维度对比表

| 维度 | Advisor | Graph |
|------|---------|-------|
| 粒度 | 单次调用 | 多步流程 |
| 结构 | 链式 | 图 |
| 状态 | 无状态 | 有状态 |
| 适用 | 横切关注点 | 复杂编排 |

### 7.1.2 组合使用

在 Graph 的每个节点内部仍可使用带 Advisor 的 ChatClient，实现“编排 + 横切”叠加。

```java
NodeAction generateNode = state -> {
    String answer = chatClient.prompt()
            .user(state.getQuestion())
            .call()
            .content();
    state.setAnswer(answer);
    return state;
};
```

## 7.2 Advisor 在 Graph 中的位置

### 7.2.1 节点内 Advisor

节点内构建 ChatClient 时挂载日志、RAG、脱敏 Advisor。

### 7.2.2 全局 Advisor

通过 `ChatClient.builder().defaultAdvisors(...)` 统一挂载，对所有调用生效。

## 7.3 边界与坑

- Advisor 无跨调用状态，做不了多步记忆。
- Graph 不替代 Advisor 的横切能力，日志等仍需 Advisor。
- 不要混淆“链顺序”与“图拓扑”两种语义。

# 8. 可观测增强

结论先行：可观测（Observability）由 Trace（链路追踪）、Metrics（指标）、Logging（日志）三支柱构成，框架默认埋点，降低接入成本。

## 8.1 可观测三要素

### 8.1.1 Trace

全链路追踪，串联一次请求经过的所有节点与模型调用。

### 8.1.2 指标

采集 Token 数、延迟、调用次数与成本，用于配额与告警。

### 8.1.3 日志

输出结构化日志，与 Trace 通过 traceId 关联。

| 能力 | 说明 |
|------|------|
| Trace | 全链路追踪 |
| 指标 | Token、延迟、成本 |
| 日志 | 结构化日志 |

## 8.2 与 ARMS 集成

### 8.2.1 配置

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
```

> 生产环境按流量调整采样率，但审计类事件不得采样。

### 8.2.2 自定义观测

```java
Observation.createNotStarted("agent.node", observationRegistry)
        .lowCardinalityKeyValue("node", "retrieve")
        .observe(() -> retrieveNode.apply(state));
```

## 8.3 边界与坑

- 采样率过高导致存储成本上升，过低导致排障困难。
- 高基数标签（如 userId）会拖垮指标系统。
- 观测数据同样可能含 PII，需脱敏。

# 9. 核心概念关系

结论先行：所有概念围绕“一次对话”与“一次流程”两条主线组织，理解调用链即可正确选型。

## 9.1 概念关系图

### 9.1.1 全景图

```
ChatClient ──▶ ChatModel ──▶ DashScope（通义）
    │
    ├── Advisor（横切）
    ├── VectorStore（RAG）
    └── Graph（多步编排）
            └── 节点内可用 ChatClient
```

### 9.1.2 调用链

一次 Graph 执行 = 多个节点顺序/条件执行；每个节点内部又是一次带 Advisor 的 ChatClient 调用。

## 9.2 选型指引

### 9.2.1 何时用 Advisor

需要横切能力（日志、脱敏、RAG、重试）时，用 Advisor。

### 9.2.2 何时用 Graph

需要多步、分支、循环、状态流转时，用 Graph。

## 9.3 边界与坑

- 用 Graph 做单次调用是过度设计。
- 用 Advisor 模拟多步流程会失去状态与可视化。

# 10. 小结

## 10.1 核心结论

- Spring AI Alibaba 复用 Spring AI 抽象，增强国内适配。
- 核心增强是 Graph 编排，Graph = 节点 + 边 + 状态。
- Advisor 处理横切，Graph 处理编排，二者互补。
- 状态是 Graph 的血脉，需设计合并策略与隔离。
- 可观测与阿里云 ARMS 集成。

## 10.2 常见误区

- 绕过 ChatClient 直接用底层 HTTP，丢失 Advisor 与可观测。
- 把状态声明为单例 Bean，导致请求间串数据。
- 条件路由无默认分支，未知返回值直接抛错。
- 用 Graph 编排单次调用，徒增复杂度。

## 10.3 生产检查清单

| 项 | 检查点 |
|----|--------|
| 模型 | 端点、密钥、配额已配置 |
| 配置 | Profile 隔离，无硬编码密钥 |
| 编排 | 无环或循环有上限，有兜底分支 |
| 状态 | 请求级隔离，合并策略明确 |
| 观测 | Trace/指标/日志已接入并脱敏 |
