# 1. Spring AI 与 LangChain4j 的核心区别？

## 1.1 标准答案

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 出身 | Spring 官方 | 社区 |
| 特色 | 统一抽象 + 自动配置 | 声明式 AiServices |
| 集成 | 原生 Spring | Starter |
| 抽象 | 偏底层统一 | 高阶组件多 |

**结论**：Spring 体系选 Spring AI；快速原型/组件定制选 LangChain4j。

## 1.2 深入原理

### 1.2.1 设计哲学差异

Spring AI 追求跨厂商的统一抽象（`ChatModel` + `ChatClient`）；LangChain4j 贴近 LangChain 生态，提供大量高阶组件。

### 1.2.2 与 Spring 的融合度

Spring AI 用自动配置把模型、向量库、工具装配为 Bean；LangChain4j 以 Starter 接入，配置粒度更细。

## 1.3 代码示例

```java
String a1 = chatClient.prompt().user(q).call().content(); // Spring AI
String a2 = assistant.chat(q);                            // LangChain4j
```

## 1.4 常见追问

**Q：两者能混用吗？**
A：可以，但会增加依赖与概念负担，通常只选一条主线。

## 1.5 生产视角

选型看**团队栈与治理体系**：与现有 Spring 监控、配置中心契合的一方，落地成本更低。

# 2. Spring AI 的核心抽象？

## 2.1 标准答案

| 抽象 | 作用 |
|------|------|
| `ChatModel` | 对话模型 |
| `ChatClient` | 应用门面（Fluent API） |
| `Prompt` / `Message` | 输入 |
| `Advisor` | 拦截器链 |
| `ChatMemory` | 记忆 |
| `VectorStore` | 向量存储 |
| `ToolCallback` | 工具 |
| `@Tool` | 工具注解 |

## 2.2 深入原理

### 2.2.1 分层结构

底层是 `ChatModel`（厂商适配），上层是 `ChatClient`（应用门面），业务只依赖门面以屏蔽厂商差异。

### 2.2.2 Message 角色

`System` 定行为、`User` 传输入、`Assistant` 存回复、`Tool` 回填结果，四类角色构成对话。

## 2.3 代码示例

```java
ChatClient client = ChatClient.builder(chatModel)
    .defaultSystem("你是严谨的技术助手")
    .defaultAdvisors(new SimpleLoggerAdvisor())
    .build();
```

## 2.4 常见追问

**Q：ChatClient 与 ChatModel 区别？**
A：前者是门面，提供流式、记忆、Advisor；后者只负责一次模型调用。

## 2.5 生产视角

业务层应**只注入 `ChatClient`**，避免直接依赖具体模型类，方便统一加护栏与观测。

# 3. Advisor 是什么？

## 3.1 标准答案

**答**：类似 Servlet Filter / AOP 的拦截器链。

```
用户请求 → [Advisor1] → [Advisor2] → ChatModel → 反向返回
```

内置：`MessageChatMemoryAdvisor`（记忆）、`QuestionAnswerAdvisor`（RAG）、`SimpleLoggerAdvisor`（日志）、`SafeGuardAdvisor`（安全）。

**顺序（order）很重要**：安全靠前，统计靠后。

## 3.2 深入原理

### 3.2.1 环绕式增强

Advisor 可在调用前改写 Prompt（如注入检索结果），也可在调用后处理响应（如日志、脱敏）。

### 3.2.2 顺序语义

order 小的先进入、后返回；安全校验应最早执行，统计应最后执行以观测真实结果。

## 3.3 代码示例

```java
ChatClient.builder(model)
    .defaultAdvisors(
        new SafeGuardAdvisor(blockedWords, 0),   // 最先
        QuestionAnswerAdvisor.builder(store).build(),
        new SimpleLoggerAdvisor())               // 最后
    .build();
```

## 3.4 常见追问

**Q：Advisor 与 AOP 切面有何不同？**
A：Advisor 面向 LLM 调用链，能读写 Prompt/响应语义；AOP 更通用但不懂提示语义。

## 3.5 生产视角

把**安全、审计、成本统计**统一收敛到 Advisor，避免散落在业务代码里造成漏防。

# 4. LangChain4j 的 AiServices 是什么？

## 4.1 标准答案

**答**：用接口 + 注解描述 LLM 交互，框架在运行时生成实现。

```java
interface Assistant {
    @SystemMessage("你是助手")
    String chat(@UserMessage String message);
}
```

## 4.2 深入原理

### 4.2.1 声明式编程

开发者只声明"要什么"，框架负责拼装 Prompt、解析响应、管理记忆与工具调用。

### 4.2.2 动态代理

框架基于接口生成动态代理，在方法调用时把参数映射为消息并触发模型调用。

## 4.3 代码示例

```java
Assistant assistant = AiServices.builder(Assistant.class)
    .chatModel(model)
    .tools(new Tools())
    .chatMemory(memory)
    .contentRetriever(retriever)
    .build();
```

## 4.4 常见追问

**Q：返回类型能是 record 吗？**
A：可以，框架会按类型做结构化解析。

## 4.5 生产视角

接口即契约：AiServices 接口变更等价于对外契约变更，需纳入版本管理与评审。

# 5. 如何在框架中实现 RAG？

## 5.1 标准答案

**Spring AI**：

```java
ChatClient.builder(model)
    .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
        .searchRequest(SearchRequest.builder().topK(5).build())
        .build())
    .build();
```

**LangChain4j**：`contentRetriever(EmbeddingStoreContentRetriever...)`。

## 5.2 深入原理

### 5.2.1 RAG 的两段式

检索（Retriever）负责召回相关片段，增强（Advisor/ContentRetriever）负责把片段拼入 Prompt。

### 5.2.2 检索质量决定上限

再强的模型也无法弥补错误召回，切分策略、TopK、重排是调优重点。

## 5.3 代码示例

```java
List<Document> docs = vectorStore.similaritySearch(query, 5);
String context = docs.stream().map(Document::getText)
    .collect(Collectors.joining("\n---\n"));
String prompt = "仅依据以下资料回答：\n" + context + "\n问题：" + query;
```

## 5.4 常见追问

**Q：TopK 越大越好吗？**
A：不是，过大引入噪声并挤占上下文，通常 3~10 并配合重排。

## 5.5 生产视角

RAG 必须可观测：记录召回文档、相似度分数与是否命中，便于定位是检索问题还是生成问题。

# 6. MCP 在框架中怎么用？

## 6.1 标准答案

**Spring AI**：`spring-ai-starter-mcp-server-*` / `-client`

```yaml
spring.ai.mcp.client.stdio.connections.filesystem:
  command: npx
  args: ["-y", "@modelcontextprotocol/server-filesystem", "/data"]
```

**LangChain4j**：`langchain4j-mcp`。**关键**：按权限裁剪 MCP 工具，不能全量暴露。

## 6.2 深入原理

### 6.2.1 MCP 的角色

MCP（Model Context Protocol，模型上下文协议）统一工具与资源接入方式，使同一工具可被不同模型/框架复用。

### 6.2.2 传输方式

stdio 适合本地进程，HTTP/SSE 适合远程服务；远程需额外做鉴权与限流。

## 6.3 代码示例

```java
// 伪代码：只挂载被授权的 MCP 工具
McpToolProvider provider = McpToolProvider.builder()
    .mcpClients(clients)
    .filter((client, tool) -> permissions.allows(tool.name()))
    .build();
```

## 6.4 常见追问

**Q：MCP 与普通 Function Calling 区别？**
A：MCP 是协议层标准化，解决工具跨平台复用；Function Calling 是模型能力。

## 6.5 生产视角

MCP 服务需纳入**统一鉴权与审计**，每次工具调用记录调用方、参数与结果，防止数据外泄。

# 7. 如何实现工具调用？

## 7.1 标准答案

**Spring AI**：

```java
@Tool(description = "查询天气")
String getWeather(@ToolParam(description = "城市") String city) {...}

chatClient.prompt().user(q).tools(new WeatherTools()).call().content();
```

**LangChain4j**：`@Tool("查询天气") String getWeather(@P("城市") String city)`。

## 7.2 深入原理

### 7.2.1 调用流程

① 请求携带工具 Schema；② 模型返回 tool_calls；③ 框架执行本地方法；④ 回填结果；⑤ 模型继续。

### 7.2.2 描述即接口

模型靠工具描述选择工具，描述不清会导致误用，因此描述质量直接影响正确率。

## 7.3 代码示例

```java
class WeatherTools {
    @Tool(description = "查询指定城市的当前天气，返回温度与天气")
    Weather getWeather(@ToolParam(description = "城市名，如北京") String city) {
        return service.query(city);
    }
}
```

## 7.4 常见追问

**Q：工具太多怎么办？**
A：按场景分组、按权限裁剪，单次挂载 ≤ 10~15 个。

## 7.5 生产视角

所有工具必须**参数校验 + 鉴权 + 幂等**；写操作带幂等键，避免模型重试导致重复下单。

# 8. 如何做结构化输出？

## 8.1 标准答案

**Spring AI**：`.call().entity(Person.class)`

**LangChain4j**：接口方法返回 POJO/record/枚举。

**优先用类型映射**，而非提示约束。

## 8.2 深入原理

### 8.2.1 类型映射的优势

框架会生成 JSON Schema 并做解析校验，比"请在提示里输出 JSON"更稳定。

### 8.2.2 校验与兜底

解析失败应重试或降级，不能直接抛异常中断业务。

## 8.3 代码示例

```java
record Person(String name, int age, List<String> skills) {}

Person p = chatClient.prompt().user(text).call().entity(Person.class);
```

## 8.4 常见追问

**Q：模型输出字段缺失怎么办？**
A：用可选类型或默认值，并对关键字段做二次校验。

## 8.5 生产视角

结构化输出是**与下游系统的契约**，需纳入 schema 版本管理与回归测试。

# 9. 如何实现流式？

## 9.1 标准答案

**Spring AI**：`.stream().content()` → `Flux<String>`

**LangChain4j**：`TokenStream` 或 `onPartialResponse`。

**SSE 是标准方案**，网关必须关闭缓冲。

## 9.2 深入原理

### 9.2.1 流式的价值

首字延迟（TTFT）决定体感速度，流式让用户先看到内容，显著改善交互体验。

### 9.2.2 网关陷阱

Nginx 等网关默认缓冲响应，会导致流式失效，需关闭 `proxy_buffering`。

## 9.3 代码示例

```java
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
Flux<String> chat(@RequestParam String q) {
    return chatClient.prompt().user(q).stream().content();
}
```

## 9.4 常见追问

**Q：流中出错怎么办？**
A：以 SSE 事件发送错误并正常关闭连接，客户端据此提示重试。

## 9.5 生产视角

流式链路上任何一环（网关、负载均衡、客户端）缓冲都会破坏体验，需端到端验证。

# 10. 多模型如何接入？

## 10.1 标准答案

**答**：用 OpenAI 兼容协议。

```java
OpenAiChatModel.builder()
    .baseUrl("https://dashscope.aliyuncs.com/compatible-mode/v1")
    .modelName("qwen-plus")
    .build();
```

**用逻辑模型名屏蔽真实模型名**，换模型只改配置。

## 10.2 深入原理

### 10.2.1 兼容协议的意义

多数国产模型提供 OpenAI 兼容接口，一套代码可对接多家，降低迁移成本。

### 10.2.2 逻辑模型名

业务使用 `chat-default` 之类逻辑名，映射到真实模型由配置中心维护，实现零代码切换。

## 10.3 代码示例

```java
String logical = "chat-default";
ChatModel model = OpenAiChatModel.builder()
    .baseUrl(config.get("models." + logical + ".url"))
    .modelName(config.get("models." + logical + ".name"))
    .build();
```

## 10.4 常见追问

**Q：如何灰度切换模型？**
A：按流量比例或租户维度路由到不同真实模型。

## 10.5 生产视角

多模型接入要统一**限流、重试、降级**策略，并记录真实模型名以便成本归因。

# 11. Spring AI Alibaba 的定位？

## 11.1 标准答案

**答**：Spring AI 的国内增强实现。

增强：通义/百炼模型适配、**Graph 编排**、可观测、国内生态。

**Graph = 节点 + 边 + 状态**，用于多 Agent 编排。

## 11.2 深入原理

### 11.2.1 定位

在 Spring AI 抽象之上补齐国内模型适配与图编排能力，使多 Agent 编排可用统一 Spring 风格表达。

### 11.2.2 Graph 三要素

节点封装动作、边定义流转、状态承载数据，支持条件边与循环。

## 11.3 代码示例

```java
// 伪代码：构建带条件边的图
StateGraph graph = new StateGraph(schema)
    .addNode("planner", plannerNode)
    .addNode("executor", executorNode)
    .addConditionalEdges("planner", route)
    .addEdge("executor", "planner");
```

## 11.4 常见追问

**Q：是否绑定阿里云？**
A：对通义/百炼适配最好，但抽象层仍兼容其他模型。

## 11.5 生产视角

若已用阿里云且技术栈为 Spring，可显著降低编排与模型接入的集成成本。

# 12. 框架选型考虑哪些因素？

## 12.1 标准答案

1. 团队技术栈（Spring/Quarkus/其他）；
2. 模型来源（国内/国外）；
3. 治理诉求（可观测、安全、成本）；
4. 生态成熟度；
5. 迁移成本。

## 12.2 深入原理

### 12.2.1 技术栈优先

框架与既有栈越贴合，团队学习与运维成本越低，长期收益越大。

### 12.2.2 治理能力

企业级场景中，可观测、限流、审计、密钥管理往往比"组件多"更重要。

## 12.3 代码示例

```java
record SelectionScore(
    int stackFit, int modelFit, int governance,
    int ecosystem, int migrationCost) {}
```

## 12.4 常见追问

**Q：能否两个框架同时用？**
A：可以但成本高，建议单一主线，避免抽象泄漏。

## 12.5 生产视角

选型应产出**可替换性验证**：核心业务仅依赖抽象接口，证明未来换框架代价可控。

# 13. 如何保持模型层可替换？

## 13.1 标准答案

```java
private final ChatModel model;          // 依赖抽象
private final OpenAiChatModel model;    // 依赖实现（应避免）
```

用配置化模型名 + 兼容协议 + 抽象接口。

## 13.2 深入原理

### 13.2.1 依赖倒置

高层业务依赖抽象（`ChatModel`），底层实现由配置注入，可在不改代码的情况下替换厂商。

### 13.2.2 兼容协议兜底

即使抽象不完全一致，OpenAI 兼容协议也能让大部分模型共用一套适配。

## 13.3 代码示例

```java
@Service
public class AssistantService {
    private final ChatClient client;
    public AssistantService(ChatClient client) { this.client = client; }
    public String ask(String q) { return client.prompt().user(q).call().content(); }
}
```

## 13.4 常见追问

**Q：厂商独有能力怎么办？**
A：通过适配器封装，能力缺失时降级，避免泄漏到业务层。

## 13.5 生产视角

把"换模型只改配置"作为**架构验收标准**，并用测试守护，防止抽象被逐步侵蚀。

# 14. 框架中如何实现可观测？

## 14.1 标准答案

**Spring AI**：Observation（Micrometer）+ Advisor。

**LangChain4j**：`ChatModelListener`。

**关键**：Token、成本、延迟、工具错误率。

## 14.2 深入原理

### 14.2.1 观测三件套

指标（Metrics）看趋势、追踪（Tracing）看链路、日志（Logging）看明细。

### 14.2.2 成本即指标

把 Token 与成本纳入指标，才能在预算超支前预警。

## 14.3 代码示例

```java
ChatModelListener listener = new ChatModelListener() {
    public void onResponse(ChatModelResponseContext ctx) {
        meterRegistry.counter("llm.tokens",
            "model", ctx.modelName()).increment(ctx.tokenUsage().total());
    }
};
```

## 14.4 常见追问

**Q：必须记录提示内容吗？**
A：生产建议脱敏后采样记录，兼顾排查与隐私。

## 14.5 生产视角

可观测的目标是回答三问：**慢在哪、贵在哪、错在哪**；三者缺一即不达标。

# 15. Advisor 与 Service 包装的区别？

## 15.1 标准答案

| 维度 | Advisor | Service 包装 |
|------|---------|--------------|
| 粒度 | 单次调用 | 业务逻辑 |
| 侵入 | 低 | 中 |
| 框架 | Spring AI 原生 | 通用 |

## 15.2 深入原理

### 15.2.1 关注点不同

Advisor 处理**模型调用前后**的横切逻辑（记忆、RAG、安全）；Service 包装处理**业务流程**（编排、事务、缓存）。

### 15.2.2 选择原则

横切且与模型调用强相关用 Advisor；涉及业务语义与事务边界用 Service 包装。

## 15.3 代码示例

```java
new MessageChatMemoryAdvisor(chatMemory); // Advisor：每次调用生效

public String askWithAudit(String q, String userId) { // Service：业务级编排
    audit.log(userId, q);
    return client.prompt().user(q).call().content();
}
```

## 15.4 常见追问

**Q：Advisor 能做业务事务吗？**
A：不建议，事务边界属于 Service 层。

## 15.5 生产视角

用**分层原则**决定落点：模型横切能力进 Advisor，业务能力进 Service，保持代码可读与可测。

# 16. 小结

## 16.1 核心结论

- Spring AI 与 LangChain4j 是 Java 两大主线；
- 核心抽象相似：模型、提示、工具、记忆、检索；
- Advisor/Listener 是横切能力的实现方式；
- 保持模型层抽象，便于切换；
- 先懂原理，框架只是加速器。

## 16.2 速记清单

| 主题 | 一句话 |
|------|--------|
| 选型 | 贴合现有技术栈与治理体系 |
| 抽象 | 业务只依赖 ChatClient/ChatModel |
| 横切 | 安全/记忆/日志用 Advisor |
| 工具 | 描述清晰、按权限裁剪 |
| 切换 | 逻辑模型名 + 兼容协议 |
| 观测 | Token、成本、延迟、错误率 |
