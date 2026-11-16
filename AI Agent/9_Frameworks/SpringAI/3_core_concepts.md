# 1. 整体分层

结论先行：Spring AI 的抽象可以分成四层，理解这张图就掌握了框架的骨架。

## 1.1 四层模型

```text
┌──────────────────────────────────────────────────────────┐
│  应用层      ChatClient（流式门面 / Fluent API）           │
├──────────────────────────────────────────────────────────┤
│  编排层      Advisor（拦截器链） / ChatMemory / RAG        │
├──────────────────────────────────────────────────────────┤
│  模型层      ChatModel / EmbeddingModel / ImageModel ...   │
├──────────────────────────────────────────────────────────┤
│  协议层      各厂商适配（OpenAI / Ollama / Anthropic ...）  │
└──────────────────────────────────────────────────────────┘
```

- **模型层**是核心抽象，**应用层**是日常使用入口，**编排层**负责横切能力。

## 1.2 各层职责

### 1.2.1 应用层与编排层

| 层 | 代表类型 | 关注点 |
|----|----------|--------|
| 应用层 | `ChatClient` | 易用性、链式调用 |
| 编排层 | `Advisor` / `ChatMemory` | 横切逻辑、上下文管理 |

### 1.2.2 模型层与协议层

| 层 | 代表类型 | 关注点 |
|----|----------|--------|
| 模型层 | `ChatModel` / `EmbeddingModel` | 稳定性、可移植 |
| 协议层 | `OpenAiChatModel` | 协议细节、鉴权 |

## 1.3 分层的意义

生产建议：出问题时按层定位——业务逻辑问题看应用层，上下文问题看编排层，调用失败看协议层。

# 2. ChatModel：对话模型的统一抽象

结论先行：`ChatModel` 是框架的"心脏"，所有对话能力最终都落到它的 `call` 方法。

## 2.1 接口定义

```java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {
    ChatResponse call(Prompt prompt);
    default Flux<ChatResponse> stream(Prompt prompt) { ... }
}
```

## 2.2 核心概念

| 概念 | 说明 |
|------|------|
| `Prompt` | 一次请求的输入，包含一组 `Message` 与 `ChatOptions` |
| `ChatResponse` | 一次响应，包含一个或多个 `Generation` |
| `Generation` | 单个候选输出，含 `AssistantMessage` 与元数据 |
| `ChatModel` | 同步调用 |
| `StreamingChatModel` | 流式调用，返回 `Flux<ChatResponse>` |

### 2.2.1 Prompt 的组成

```text
Prompt
 ├── List<Message>   指令与输入
 └── ChatOptions     模型与采样参数
```

### 2.2.2 ChatResponse 的读取

```java
ChatResponse response = chatModel.call(new Prompt("用一句话解释什么是向量"));
String text = response.getResult().getOutput().getText();
```

## 2.3 同步与流式

```java
// 同步
String answer = chatModel.call(new Prompt("你好"))
        .getResult().getOutput().getText();

// 流式
Flux<ChatResponse> flux = chatModel.stream(new Prompt("写一首诗"));
flux.map(r -> r.getResult().getOutput().getText()).subscribe(System.out::print);
```

## 2.4 边界与坑

边界与坑：`call` 是阻塞调用，在高并发场景会占用线程；应配合超时与线程池，或改用流式/虚拟线程。

生产建议：给每次调用设置读取超时，避免慢响应拖垮线程池。

# 3. Message：消息体系

结论先行：`Message` 是提示工程的最小单元，角色划分决定了模型的"视角"。

## 3.1 消息类型

```text
Message
├── SystemMessage      系统指令（角色/规则）
├── UserMessage        用户输入（可带多模态 Media）
├── AssistantMessage   模型回复（可含 ToolCall）
└── ToolResponseMessage 工具执行结果
```

| 类型 | 角色 | 用途 |
|------|------|------|
| `SystemMessage` | system | 设定人设与规则 |
| `UserMessage` | user | 用户输入 |
| `AssistantMessage` | assistant | 模型输出 |
| `ToolResponseMessage` | tool | 工具返回值 |

## 3.2 构造与使用

```java
Prompt prompt = new Prompt(List.of(
        new SystemMessage("你是一个严谨的 Java 架构师。"),
        new UserMessage("什么是依赖注入？")
));
```

> 角色划分是提示工程的基础，详见 `3_PromptEngineering/2_prompt_structure.md`。

### 3.2.1 多模态消息

```java
UserMessage msg = UserMessage.builder()
        .text("这张图里有什么？")
        .media(new Media(MimeTypeUtils.IMAGE_PNG, resource))
        .build();
```

### 3.2.2 工具消息

```java
AssistantMessage call = AssistantMessage.builder()
        .toolCalls(List.of(new ToolCall("getWeather", Map.of("city", "上海"))))
        .build();
```

## 3.3 边界与坑

边界与坑：并非所有模型都支持 `system` 角色；部分模型会把 `system` 折叠进首条 `user` 消息，效果可能不同。

生产建议：对多模型场景，先用最小用例验证角色支持度，再决定提示结构。

# 4. ChatOptions：调用参数

结论先行：`ChatOptions` 决定"这次调用怎么跑"，分为可移植字段与厂商特有字段。

## 4.1 可移植参数

```java
OpenAiChatOptions options = OpenAiChatOptions.builder()
        .model("gpt-4o-mini")
        .temperature(0.3)
        .maxTokens(1024)
        .build();

ChatResponse resp = chatModel.call(new Prompt("你好", options));
```

- 参数包括：模型、温度、最大 Token、top-p、停止词等。
- 不同厂商有各自的 `*ChatOptions`，也可用可移植的 `ChatOptions` 基础字段。

| 参数 | 作用 | 典型取值 |
|------|------|----------|
| `model` | 指定模型 | `gpt-4o-mini` |
| `temperature` | 随机性 | 0.0–1.0 |
| `maxTokens` | 输出上限 | 512–4096 |
| `topP` | 采样范围 | 0.8–1.0 |
| `stopSequences` | 停止词 | 自定义 |

## 4.2 厂商特有参数
```java
AnthropicChatOptions options = AnthropicChatOptions.builder()
        .model("claude-3-5-sonnet")
        .temperature(0.2)
        .build();
```

边界与坑：厂商特有参数不可移植，换模型后可能被忽略或报错。

生产建议：把稳定参数（模型、超时）放配置，把易变参数（temperature）放调用级，调用级 options 优先级最高。

# 5. ChatClient：面向应用的门面

结论先行：`ChatClient` 是**日常开发最常用的入口**，提供 Fluent API，把 Prompt 组装、Advisor、记忆、工具、结构化输出统一起来。

## 5.1 为什么需要门面

```text
直接用 ChatModel：手动组装 Prompt、手动挂 Advisor、手动取结果
用 ChatClient：链式声明，框架负责组装
```

## 5.2 Fluent API

```java
String answer = chatClient.prompt()
        .system("你是法律顾问")
        .user("劳动合同到期不续签有赔偿吗？")
        .call()
        .content();
```

### 5.2.1 基础调用

| 方法 | 作用 |
|------|------|
| `.system(...)` | 设置系统提示 |
| `.user(...)` | 设置用户输入 |
| `.call()` | 同步调用 |
| `.stream()` | 流式调用 |
| `.content()` | 直接取文本 |

### 5.2.2 默认值

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultSystem("你是企业助手")
        .defaultAdvisors(new SimpleLoggerAdvisor())
        .build();
```

> `ChatClient` 是"门面（Facade）"，底层仍是 `ChatModel`。详见 `4_chat_client.md`。

## 5.3 边界与坑

边界与坑：`ChatClient` 线程安全，但 `defaultSystem` 等默认值在构建时固化，运行期无法按请求动态改变；动态部分应放调用级。

生产建议：按业务域构建多个 `ChatClient`（客服、摘要、代码助手），而不是一个巨型客户端。

# 6. Advisor：拦截器链

结论先行：`Advisor` 类似 Servlet Filter / AOP，在请求前后插入横切逻辑。

## 6.1 拦截器模型

```text
用户请求 → [Advisor1] → [Advisor2] → ChatModel → [Advisor2] → [Advisor1] → 响应
```

## 6.2 内置 Advisor

常见内置 Advisor：

| Advisor | 作用 |
|---------|------|
| `MessageChatMemoryAdvisor` | 注入历史消息 |
| `QuestionAnswerAdvisor` | RAG 检索增强 |
| `SimpleLoggerAdvisor` | 打印请求/响应日志 |
| `SafeGuardAdvisor` | 敏感内容拦截 |

> 详见 `9_advisors.md`。

## 6.3 自定义 Advisor

### 6.3.1 实现方式

```java
public class TimingAdvisor implements CallAdvisor {

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        long start = System.nanoTime();
        ChatClientResponse response = chain.nextCall(request);
        // 记录耗时
        return response;
    }
}
```

### 6.3.2 顺序控制

```java
ChatClient.builder(model)
        .defaultAdvisors(
                new SimpleLoggerAdvisor(),
                new MessageChatMemoryAdvisor(memory))
        .build();
```

生产建议：顺序影响结果——记忆注入应在日志之前，安全拦截应尽量靠前。

# 7. ToolCallback：工具调用

结论先行：工具调用让模型从"只会说"变成"能做事"。

## 7.1 注解声明

Spring AI 用 `@Tool` 注解把普通 Java 方法暴露为模型可调用的工具：

```java
class WeatherTools {
    @Tool(description = "查询指定城市的天气")
    String getWeather(@ToolParam(description = "城市名") String city) {
        return city + "：晴，25℃";
    }
}
```

## 7.2 调用流程

```text
模型 → 决定调用工具 → 返回 ToolCall → Spring AI 执行 Java 方法 → ToolResponse → 模型继续
```

### 7.2.1 注册工具

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultTools(new WeatherTools())
        .build();
```

### 7.2.2 多轮工具调用

边界与坑：一次请求可能触发多次工具调用，需设置最大轮次，避免无限循环。

```java
// 伪代码：限制工具调用轮次
options.setMaxToolCallRounds(5);
```

> 详见 `7_function_calling_tools.md`。

## 7.3 边界与坑

生产建议：工具必须有参数校验、超时与权限检查，切勿把内部高危接口直接暴露给模型。

# 8. ChatMemory：对话记忆

结论先行：模型本身无状态，多轮对话的"记忆"需要应用侧显式维护。

## 8.1 接口定义

```java
public interface ChatMemory {
    void add(String conversationId, List<Message> messages);
    List<Message> get(String conversationId);
    void clear(String conversationId);
}
```

- `conversationId` 用于隔离不同用户/会话。
- 实现：`InMemoryChatMemory`、`JdbcChatMemoryRepository`、`RedisChatMemoryRepository` 等。

## 8.2 实现类型

| 实现 | 存储 | 适用 |
|------|------|------|
| `InMemoryChatMemory` | 内存 | 开发/单实例 |
| `JdbcChatMemoryRepository` | 关系库 | 需持久化 |
| `RedisChatMemoryRepository` | Redis | 分布式 |

## 8.3 与 Advisor 的结合

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(memory).build())
        .build();

client.prompt()
        .user("接着说")
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42"))
        .call()
        .content();
```

> 详见 `8_chat_memory.md` 与 `8_Memory`。

边界与坑：记忆无限增长会挤爆上下文窗口，必须做截断或摘要。

生产建议：按 `conversationId` 设置 TTL，并限制保留的消息条数。

# 9. EmbeddingModel 与 VectorStore

结论先行：RAG 的两块基石——把文本变向量，再按相似度检索。

## 9.1 EmbeddingModel

```text
文本 ──EmbeddingModel──▶ 向量 ──VectorStore──▶ 存储/检索
```

| 概念 | 说明 |
|------|------|
| `EmbeddingModel` | 把文本转成向量 |
| `VectorStore` | 向量的存取与相似度检索 |
| `Document` | 文本 + 元数据 + 向量的载体 |
| `SearchRequest` | 检索参数（Top-K、相似度阈值、过滤条件） |

```java
float[] vector = embeddingModel.embed("Spring AI 支持多模型接入");
```

## 9.2 VectorStore

### 9.2.1 写入

```java
Document doc = new Document("Spring AI 支持多模型接入",
        Map.of("source", "handbook.pdf"));
vectorStore.add(List.of(doc));
```

### 9.2.2 检索

```java
List<Document> hits = vectorStore.similaritySearch(
        SearchRequest.builder().query("如何接入模型").topK(5).build());
```

> 详见 `11_vector_store.md` 与 `5_RAG`。

## 9.3 边界与坑

边界与坑：写入与查询必须使用**同一个嵌入模型**，否则向量空间不一致，检索结果会失真。

# 10. ETL：文档处理流水线

结论先行：RAG 的效果很大程度取决于文档切分质量。

## 10.1 三段式

```text
DocumentReader ──▶ DocumentTransformer ──▶ DocumentWriter
  （读取）            （切分/清洗）           （写入 VectorStore）
```

| 阶段 | 接口 | 示例 |
|------|------|------|
| 读取 | `DocumentReader` | PDF、Markdown、JSON |
| 转换 | `DocumentTransformer` | 切分、去重、脱敏 |
| 写入 | `DocumentWriter` | 写入 `VectorStore` |

## 10.2 代码示例

```java
List<Document> docs = reader.read();
List<Document> chunks = splitter.apply(docs);
vectorStore.add(chunks);
```

## 10.3 边界与坑

边界与坑：切分过大导致检索噪声，过小导致语义割裂；需按文档类型调参。

> 详见 `12_etl_pipeline.md`。

生产建议：对表格、代码块采用结构化切分，避免按固定字符数硬切。

# 11. 自动配置机制

结论先行：Spring Boot 的自动配置是 Spring AI"开箱即用"的关键。

## 11.1 装配流程

```text
引入 spring-ai-starter-model-openai
        │
        ▼
OpenAiAutoConfiguration
        │  读取 spring.ai.openai.*
        ▼
创建 OpenAiChatModel / OpenAiEmbeddingModel Bean
        │
        ▼
ChatClientAutoConfiguration 创建 ChatClient.Builder
```

## 11.2 条件装配

条件装配要点：

| 条件 | 含义 |
|------|------|
| `@ConditionalOnClass` | classpath 存在对应类才装配 |
| `@ConditionalOnProperty` | 配置项满足才装配（如 `spring.ai.openai.chat.enabled=true`） |
| `@ConditionalOnMissingBean` | 用户自定义 Bean 优先，框架兜底 |

> 这意味着：**你可以随时用自定义 Bean 覆盖框架默认实现**，这是 Spring AI 可扩展性的基础。

## 11.3 覆盖机制

```java
@Bean
ChatModel chatModel(ChatModel delegate) {
    return new MetricsChatModel(delegate);
}
```

生产建议：覆盖时保持委托语义，只增加横切能力，不要改变核心行为。

# 12. 核心概念关系图

## 12.1 关系总览

```text
ChatClient ──uses──▶ ChatModel ──uses──▶ ChatOptions
    │                    │
    │                    └──returns──▶ ChatResponse ──contains──▶ Generation
    │                                                              │
    ├──uses──▶ Advisor ──modifies──▶ Prompt / ChatResponse         │
    ├──uses──▶ ChatMemory ──provides──▶ List<Message>              │
    ├──uses──▶ ToolCallback ──produces──▶ ToolResponse ────────────┘
    └──uses──▶ VectorStore ──retrieves──▶ List<Document>
```

## 12.2 数据流

```text
用户输入
   │
   ▼
ChatClient.prompt() ──▶ Prompt（含 Message + Options）
   │
   ▼
Advisor 链（记忆注入 / RAG 检索 / 日志）
   │
   ▼
ChatModel.call() ──▶ ChatResponse ──▶ 文本 / 工具调用
```

## 12.3 常见组合

| 场景 | 组合 |
|------|------|
| 多轮客服 | ChatClient + ChatMemory + Advisor |
| 知识问答 | ChatClient + VectorStore + QuestionAnswerAdvisor |
| 智能体 | ChatClient + ToolCallback + ChatMemory + MCP |

# 13. 小结

## 13.1 核心要点

- **ChatModel 是心脏，ChatClient 是门面，Advisor 是血管，Memory/Tool/RAG 是器官。**
- 所有组件都可通过自定义 Bean 覆盖，扩展性强。

## 13.2 下一步

- 学习顺序：ChatModel → ChatClient → Prompt → 结构化输出 → Tool → Memory → Advisor → RAG → MCP。
