# 1. 整体分层

LangChain4j 的核心概念可以用一张分层图概括。理解这张图，就理解了各组件之间的依赖方向与职责边界。

## 1.1 五层结构

```
┌───────────────────────────────────────────────┐
│  声明式层   AiServices（接口 + 注解）            │
├───────────────────────────────────────────────┤
│  编排层     ChatMemory / ContentRetriever       │
│             RetrievalAugmentor / Agentic         │
├───────────────────────────────────────────────┤
│  模型层     ChatModel / StreamingChatModel       │
│             EmbeddingModel / ImageModel ...      │
├───────────────────────────────────────────────┤
│  数据层     EmbeddingStore / Document / Segment  │
├───────────────────────────────────────────────┤
│  适配层     各厂商模型 / 各向量库 / 文档解析器    │
└───────────────────────────────────────────────┘
```

| 层次 | 职责 | 典型接口 |
|------|------|----------|
| 声明式层 | 把 Java 接口翻译成 LLM 调用 | `AiServices` |
| 编排层 | 组合记忆、检索、工具 | `ChatMemory`、`ContentRetriever` |
| 模型层 | 统一"消息进、消息出" | `ChatModel`、`EmbeddingModel` |
| 数据层 | 向量与文档的存储抽象 | `EmbeddingStore`、`Document` |
| 适配层 | 协议翻译（HTTP/JSON） | 各厂商实现类 |

## 1.2 数据流与依赖方向

依赖方向自上而下，**下层不感知上层**：

```
业务代码 ──▶ AiServices ──▶ ChatModel ──▶ 厂商 HTTP API
                 │
                 ├──▶ ChatMemory
                 ├──▶ ContentRetriever ──▶ EmbeddingStore
                 └──▶ @Tool 方法
```

这个约束带来一个重要好处：模型、向量库都能被替换为任意实现，业务代码零改动。

## 1.3 双轨 API 的边界

| 能力 | 低层 API 是否自动 | 高层 `AiServices` 是否自动 |
|------|------------------|---------------------------|
| 提示组装 | 否 | 是 |
| 记忆读写 | 否 | 是（配 `ChatMemory`） |
| 工具循环 | 否 | 是（配 `tools`） |
| RAG 检索 | 否 | 是（配 `contentRetriever`） |
| 结构化输出 | 部分（`chat(..., Class)`） | 是（方法返回类型） |
| 重试与观测 | 是 | 是 |

# 2. ChatModel：对话模型抽象

## 2.1 接口定义

结论先行：`ChatModel` 是 LangChain4j 的最小契约，输入是消息、输出是响应。

```java
public interface ChatModel {
    String chat(String userMessage);
    ChatResponse chat(ChatMessage... messages);
    ChatResponse chat(List<ChatMessage> messages);
    ChatResponse chat(ChatRequest chatRequest);
}
```

| 接口 | 说明 |
|------|------|
| `ChatModel` | 同步对话（1.0 前名为 `ChatLanguageModel`） |
| `StreamingChatModel` | 流式对话，回调或 `TokenStream` |

## 2.2 请求与响应对象

### 2.2.1 ChatRequest

`ChatRequest` 是完整请求载体，除消息外还携带模型参数：

```java
ChatRequest request = ChatRequest.builder()
        .messages(UserMessage.from("你好"))
        .temperature(0.3)
        .maxTokens(512)
        .build();
```

### 2.2.2 ChatResponse 与元数据

`ChatResponse` 除了 `AiMessage`，还包含 Token 用量等元数据，是成本核算的依据：

```java
ChatResponse resp = model.chat(request);
String text = resp.aiMessage().text();
TokenUsage usage = resp.tokenUsage();
// usage.inputTokenCount() / outputTokenCount()
```

## 2.3 同步 vs 流式

```java
ChatModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .build();

String text = model.chat("你好");
```

流式版本把同一个请求拆成多个增量回调，见第 3 篇 `3_chat_models.md`。

## 2.4 边界与坑

- `chat(String)` 是便捷方法，等价于单条 `UserMessage`，不适合多轮；
- 元数据（Token 用量）**不一定**每个厂商都返回，代码需容忍 null；
- 低层 API **不保存**任何历史，多轮需自行拼装消息列表。

# 3. ChatMessage：消息体系

## 3.1 消息类型

```
ChatMessage
├── SystemMessage            系统指令
├── UserMessage              用户输入（支持多模态）
├── AiMessage                模型回复（可含 toolExecutionRequests）
├── ToolExecutionResultMessage 工具执行结果
└── CustomMessage            自定义（厂商特有角色）
```

| 类型 | 角色 | 关键字段 |
|------|------|----------|
| `SystemMessage` | system | `text` |
| `UserMessage` | user | `contents`（多模态） |
| `AiMessage` | assistant | `text`、`toolExecutionRequests` |
| `ToolExecutionResultMessage` | tool | `id`、`toolName`、`text` |
| `CustomMessage` | 厂商自定义 | `attributes` |

## 3.2 多模态消息

### 3.2.1 内容块类型

`UserMessage` 可包含多个内容块，实现图文混合输入：

```java
UserMessage message = UserMessage.from(
        TextContent.from("这张图说明了什么？"),
        ImageContent.from("https://example.com/chart.png")
);
```

## 3.3 工具调用消息

工具调用是"模型请求 → 应用执行 → 回填结果"的三段式：

```java
// 模型请求调用工具
AiMessage ai = response.aiMessage();
List<ToolExecutionRequest> requests = ai.toolExecutionRequests();

// 回填工具结果
ToolExecutionResultMessage result =
        ToolExecutionResultMessage.from(requests.get(0), "25℃");
model.chat(List.of(userMessage, ai, result));
```

## 3.4 消息顺序与约束

一次对话的消息列表有严格顺序要求：

| 规则 | 说明 |
|------|------|
| 系统消息在前 | `SystemMessage` 通常放首位 |
| 一问一答交替 | `UserMessage` 后应跟 `AiMessage` |
| 工具成对出现 | 每个 `toolExecutionRequests` 都要有对应结果 |
| ID 必须匹配 | `ToolExecutionResultMessage` 的 `id` 对应请求 ID |

## 3.5 边界与坑

- 漏回填工具结果会导致多数厂商直接报错；
- 多个工具请求时，结果消息数量必须与请求数量一致；
- 部分模型不接受中途插入的 `SystemMessage`。

# 4. Prompt 与 PromptTemplate

## 4.1 Prompt

`Prompt` 是"最终发给模型的内容"，可由静态文本或模板生成：

```java
// 静态提示
Prompt prompt = Prompt.from("你好");
```

## 4.2 PromptTemplate

### 4.2.1 占位符

```java
// 模板
PromptTemplate template = PromptTemplate.from("用{{style}}的风格解释{{topic}}");
Map<String, Object> vars = Map.of("style", "简洁", "topic", "向量数据库");
Prompt p = template.apply(vars);
```

> `PromptTemplate` 支持 `{{var}}` 占位符；声明式写法见 `@SystemMessage` / `@UserMessage`。

### 4.2.2 转义与注入

用户输入若包含 `{{...}}`，可能被误当成占位符。生产环境应：

- 模板变量由代码注入，避免直接把用户输入拼进模板文本；
- 对用户内容做转义或使用参数化 API。

## 4.3 边界与坑

- 缺失变量会抛异常或留下空占位符，需校验；
- 模板与声明式注解二选一即可，混用易混乱；
- 提示注入（Prompt Injection）是安全重点，见生产建议。

# 5. AiServices：声明式核心（重点）

`AiServices` 是 LangChain4j 最具辨识度的能力：**用接口描述 LLM 交互，框架生成实现**。

## 5.1 最小示例

### 5.1.1 接口即契约

```java
interface Assistant {
    @SystemMessage("你是一个专业的中文助手。")
    String chat(@UserMessage String userMessage);
}

Assistant assistant = AiServices.create(Assistant.class, model);
String answer = assistant.chat("介绍一下 LangChain4j");
```

## 5.2 自动处理的能力

它自动处理：

- 提示组装（`@SystemMessage` / `@UserMessage` / `@V`）
- 结构化输出映射
- 工具调用循环
- 记忆读写（`@MemoryId`）
- RAG 检索增强

## 5.3 builder 配置项

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .chatMemory(memory)
        .tools(new BookingTools())
        .contentRetriever(retriever)
        .build();
```

| 配置项 | 作用 |
|--------|------|
| `chatModel` / `streamingChatModel` | 指定底层模型 |
| `chatMemory` / `chatMemoryProvider` | 记忆 |
| `tools` | 工具集合 |
| `contentRetriever` | 检索器 |
| `retrievalAugmentor` | 完整 RAG 编排 |
| `chatModelListener` | 观测 |

## 5.4 返回值与流式

`AiServices` 的返回类型决定行为，详见第 9 节。

## 5.5 边界与坑

- 接口方法不能是 `default` / `static` 以外被代理忽略的形式；
- 动态代理基于 JDK Proxy，接口而非类；
- 每个 `@MemoryId` 对应独立记忆，注意内存增长。

> 详见 `5_ai_services.md`。

# 6. ChatMemory：记忆

## 6.1 接口

```java
public interface ChatMemory {
    void add(ChatMessage message);
    List<ChatMessage> messages();
    void clear();
}
```

## 6.2 实现

```java
ChatMemory memory = MessageWindowChatMemory.withMaxMessages(20);
```

| 实现 | 说明 |
|------|------|
| `MessageWindowChatMemory` | 保留最近 N 条 |
| `TokenWindowChatMemory` | 按 Token 预算保留 |
| `PersistentChatMemoryStore` | 自定义持久化（DB/Redis） |

## 6.3 持久化

`ChatMemory` 本身是内存视图，持久化由 `ChatMemoryStore` 承担：

```java
ChatMemory memory = MessageWindowChatMemory.builder()
        .id("user-42")
        .maxMessages(20)
        .chatMemoryStore(redisStore)
        .build();
```

## 6.4 边界与坑

- 窗口裁剪会**丢弃**早期消息，重要约束应放系统提示；
- 多实例部署时内存态记忆不共享，必须用外部存储；
- Token 预算计算依赖分词器，不同模型结果不同。

> 详见 `7_chat_memory.md` 与 `8_Memory`。

# 7. RAG 组件

## 7.1 五件套

### 7.1.1 两条链路

```
Document ─DocumentSplitter─▶ TextSegment ─EmbeddingModel─▶ Embedding
                                                              │
                                                              ▼
                                                      EmbeddingStore
                                                              │
 查询 ─EmbeddingModel─▶ Embedding ─ContentRetriever─▶ List<Content>
                                                              │
                                                              ▼
                                                   RetrievalAugmentor
                                                              │
                                                              ▼
                                                        ChatModel
```

| 概念 | 说明 |
|------|------|
| `Document` | 原始文档（文本 + 元数据） |
| `TextSegment` | 切分后的片段 |
| `EmbeddingModel` | 文本 → 向量 |
| `EmbeddingStore` | 向量存取 |
| `ContentRetriever` | 检索器（向量、全文、混合） |
| `RetrievalAugmentor` | 检索增强编排 |

## 7.2 摄取链路

```
加载文档 → 解析 → 切分 → 向量化 → 写入 EmbeddingStore
```

```java
List<Document> docs = FileSystemDocumentLoader.loadDocuments("docs/");
List<TextSegment> segments = DocumentSplitters.recursive(500, 50).split(docs);
EmbeddingStoreIngestor.ingest(docs, store);   // 一步到位
```

## 7.3 检索链路

```java
ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
        .embeddingStore(store)
        .embeddingModel(embeddingModel)
        .maxResults(5)
        .minScore(0.6)
        .build();
```

## 7.4 边界与坑

- 切分粒度直接决定召回质量，需按文档结构调参；
- 向量库与 Embedding 模型必须**维度一致**；
- `minScore` 阈值因模型而异，需实测；
- 检索到的内容要防注入，勿当系统指令。

> 详见 `8_rag.md`、`9_embeddings_vector_store.md`。

# 8. Tools：工具

## 8.1 定义

### 8.1.1 注解说明

用 `@Tool` 注解普通 Java 方法即可暴露为模型可调用的工具：

```java
class Calculator {
    @Tool("计算两个数之和")
    double add(@P("第一个数") double a, @P("第二个数") double b) {
        return a + b;
    }
}
```

## 8.2 工具调用循环

```
模型输出 toolExecutionRequests
        │
        ▼
框架反射执行 @Tool 方法
        │
        ▼
把结果作为 ToolExecutionResultMessage 回填
        │
        ▼
再次请求模型，直到无工具请求
```

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new Calculator())
        .build();
```

## 8.3 边界与坑

- 工具描述（`@Tool` 文本）质量决定模型是否会用对；
- 参数越简单越好，复杂对象增加模型出错概率；
- 工具执行需做权限校验与超时，勿信任模型参数；
- 有副作用的工具要考虑幂等与确认。

> 详见 `6_tools.md`。

# 9. 返回值类型

## 9.1 类型清单

### 9.1.1 类型选择建议

| 返回类型 | 说明 |
|----------|------|
| `String` | 纯文本 |
| `AiMessage` | 完整消息（含工具请求） |
| `ChatResponse` | 响应 + 元数据（Token 用量） |
| `TokenStream` | 流式 token |
| POJO / `record` / 枚举 / `List<T>` | 结构化输出 |
| `Result<T>` | 成功/失败封装，不抛异常 |

## 9.2 结构化输出

```java
interface Service {
    Result<Sentiment> analyze(String text);   // 结构化 + 不抛异常
    TokenStream stream(String prompt);          // 流式
}
```

模型返回的 JSON 会映射为 `Sentiment`，失败则封装进 `Result`。

## 9.3 Result 封装

`Result<T>` 适合不希望异常中断流程的场景：

```java
Result<Sentiment> r = service.analyze(text);
if (r.isSuccess()) {
    Sentiment s = r.content();
} else {
    String cause = r.error().getMessage();
}
```

# 10. ChatModelListener：监听器

## 10.1 生命周期回调

### 10.1.1 回调时机

监听器在请求、响应、错误三个时机被调用，是观测与审计的统一入口。

## 10.2 示例

```java
ChatModelListener listener = new ChatModelListener() {
    @Override
    public void onRequest(ChatModelRequestContext ctx) {
        log.info("请求: {}", ctx.chatRequest().messages());
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        log.info("响应: {}", ctx.chatResponse().aiMessage().text());
        log.info("Token: {}", ctx.chatResponse().tokenUsage());
    }

    @Override
    public void onError(ChatModelErrorContext ctx) {
        log.error("错误", ctx.error());
    }
};

ChatModel model = OpenAiChatModel.builder()
        .apiKey(key)
        .modelName("gpt-4o-mini")
        .listeners(List.of(listener))
        .build();
```

## 10.3 生产用途

| 用途 | 说明 |
|------|------|
| 日志 | 记录请求与响应 |
| 成本 | 累计 Token 用量 |
| 审计 | 留存敏感操作证据 |
| 链路追踪 | 注入 traceId |

> 可观测性的基础，详见 `12_observability.md`。

# 11. 核心概念关系图

## 11.1 关系图

```
AiServices ──uses──▶ ChatModel ──produces──▶ ChatResponse
    │                    │                        │
    │                    └──uses──▶ ChatRequest    └──contains──▶ AiMessage
    ├──uses──▶ ChatMemory ──provides──▶ List<ChatMessage>
    ├──uses──▶ ContentRetriever ──provides──▶ List<Content>
    ├──uses──▶ @Tool methods ──produce──▶ ToolExecutionResult
    └──uses──▶ ChatModelListener（观测/审计）
```

## 11.2 依赖规则

- `AiServices` 依赖模型、记忆、检索器、工具，是唯一"总编排者"；
- `ChatModel` 只依赖消息体系，不依赖任何上层组件；
- `EmbeddingStore` 只被 `ContentRetriever` 使用，与对话无关。

# 12. 小结

- **低层 `ChatModel` 可控，高层 `AiServices` 高效**，双轨并用。
- `ChatMessage` 体系与 Spring AI 高度相似，理解其一即可迁移。
- `AiServices` 是 LangChain4j 的核心竞争力：接口即 Agent。
- RAG 五件套：`Document → TextSegment → Embedding → EmbeddingStore → ContentRetriever`。
- 记忆、工具、检索都通过"配置注入"的方式挂到 `AiServices` 上，保持业务接口干净。
