# 1. 为什么需要 ChatClient

## 1.1 直接使用 ChatModel 的痛点

**结论先行**：`ChatModel` 是 Spring AI 的底层能力入口，只负责"发一次请求、拿一次响应"，所有工程化的横切逻辑都要调用方自己承担。业务代码一旦直接依赖它，很快会被样板代码淹没。

### 1.1.1 手工装配的样板代码

每次调用都要手动组装 `Prompt`、`Message` 列表、`ChatOptions`：

```java
// 直接用 ChatModel：每次都要重复这套装配
SystemMessage system = new SystemMessage("你是一个专业助手。");
UserMessage user = new UserMessage(question);
ChatOptions options = OpenAiChatOptions.builder()
        .temperature(0.3)
        .maxTokens(512)
        .build();
Prompt prompt = new Prompt(List.of(system, user), options);
ChatResponse response = chatModel.call(prompt);
String text = response.getResult().getOutput().getText();
```

问题在于：这段代码里真正"业务特有"的只有 `question`，其余全是重复的框架噪声。

### 1.1.2 横切关注点散落

当需求扩展到日志、记忆、RAG、工具调用、重试时，直接调用 `ChatModel` 会导致横切逻辑散落在每个业务方法里：

| 关注点 | 直接调用 ChatModel 的后果 |
|--------|---------------------------|
| 记忆 | 每处都要手动把历史消息拼进 `Prompt` |
| 日志 | 每处都要手动打印请求与响应 |
| RAG | 每处都要手动检索并拼接上下文 |
| 工具 | 每处都要手动注册并处理工具回调 |
| 参数默认值 | 每处都要重复设置 `temperature` 等 |

## 1.2 ChatClient 的定位与能力边界

### 1.2.1 Fluent API 门面

`ChatClient` 提供了一套 **Fluent API 门面（facade）**，把上述一切串成一条可读的链：

```
ChatClient.prompt()
    ├── .system(...) / .user(...) / .messages(...)   组装输入
    ├── .options(...)                                调用参数
    ├── .advisors(...)                               拦截器
    ├── .tools(...)                                  工具
    └── .call() / .stream()                          执行
            ├── .content()                           取文本
            ├── .chatResponse()                      取完整响应
            └── .entity(Class)                       取结构化对象
```

### 1.2.2 与 ChatModel 的关系

需要明确边界：`ChatClient` **不替代** `ChatModel`，而是包在它外面的一层。

| 维度 | `ChatModel` | `ChatClient` |
|------|-------------|--------------|
| 抽象层级 | 低（单次请求） | 高（调用编排） |
| 默认设定 | 无 | 有（构建期注入） |
| 记忆/日志/工具 | 手动 | Advisor 自动 |
| 返回形态 | `ChatResponse` | 文本/对象/流 |
| 适用场景 | 自定义协议、底层扩展 | 日常业务调用 |

## 1.3 调用主链路总览

**机制**：`prompt()` 返回一个可变的请求构建器，链式方法逐层累积输入、参数与 Advisor；`call()` 或 `stream()` 触发终端操作，把累积结果固化为 `Prompt` 交给 `ChatModel`。

```
业务调用 ──► ChatClient 链式构建 ──► Advisor 链 ──► ChatModel ──► 模型 API
                    ▲                                     │
                    └────────── 解析/转换结果 ◄───────────┘
```

**生产建议**：把 `ChatClient` 当作"面向业务的调用契约"，把 `ChatModel` 当作"面向厂商的适配层"，两者不要混用在同一处业务代码里。

# 2. 创建 ChatClient

## 2.1 注入 Builder（推荐）

**结论先行**：优先注入 `ChatClient.Builder`，因为它允许在构建时设定默认 system、默认 Advisor、默认工具，实现"全局基线 + 局部覆盖"。

### 2.1.1 构建期默认设定

```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder
                .defaultSystem("你是一个专业、简洁的中文助手。")
                .build();
    }
}
```

### 2.1.2 原型作用域说明

**注意**：`ChatClient.Builder` 由自动配置提供，作用域为原型（prototype）——每次注入都是新实例，因此可以安全地在其上叠加各自不同的默认设定，互不污染。

```java
@Service
public class AnotherService {
    public AnotherService(ChatClient.Builder builder) {
        // 这里再叠加默认设定，不会影响 ChatService
        ChatClient client = builder.defaultSystem("你是数据分析助手。").build();
    }
}
```

## 2.2 使用自动配置的 ChatClient Bean

Spring AI 也会提供一个默认的 `ChatClient` Bean，可直接注入：

```java
@RestController
public class DemoController {
    private final ChatClient chatClient;
    public DemoController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
}
```

> 实践建议：默认 Bean 适合快速验证；一旦需要多种默认设定，就改用注入 `Builder` 自行构建。

## 2.3 手动构建（多模型场景）

当应用中存在多个 `ChatModel` 时，用 `ChatClient.builder(model)` 显式绑定：

```java
ChatClient openAiClient = ChatClient.builder(openAiChatModel)
        .defaultSystem("你是 OpenAI 助手")
        .build();

ChatClient ollamaClient = ChatClient.builder(ollamaChatModel)
        .defaultSystem("你是本地助手")
        .build();
```

## 2.4 依赖与配置

### 2.4.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

版本建议通过 `spring-ai-bom` 统一管理（具体版本以官方文档为准）。

### 2.4.2 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.3
          max-tokens: 1024
```

**生产建议**：`api-key` 永远走环境变量或配置中心，禁止硬编码进仓库。

# 3. 基本调用

## 3.1 单轮对话

**结论先行**：最简单的调用就是 `.user(...).call().content()`，适用于无状态、单轮的场景。

```java
String answer = chatClient.prompt()
        .user("用三句话介绍 Spring AI")
        .call()
        .content();
```

## 3.2 指定 system

**机制**：`.system(...)` 会构造一条 `SystemMessage`，优先级高于 `defaultSystem`，用于本次覆盖全局设定。

```java
String answer = chatClient.prompt()
        .system("你是一个只会用 JSON 回答的接口。")
        .user("返回用户的姓名和年龄：我叫张三，今年 28。")
        .call()
        .content();
```

## 3.3 多消息对话

当需要精确控制角色序列（例如注入 few-shot 示例）时，用 `.messages(...)`：

```java
String answer = chatClient.prompt()
        .messages(
                new SystemMessage("你是一个数学老师。"),
                new UserMessage("什么是质数？"),
                new AssistantMessage("质数是只能被 1 和自身整除的大于 1 的自然数。"),
                new UserMessage("那 1 是质数吗？")
        )
        .call()
        .content();
```

**边界与坑**：

| 现象 | 原因 | 建议 |
|------|------|------|
| 消息顺序被模型忽略 | 部分模型对长对话中间内容不敏感 | 关键规则放 system 或末尾 |
| `.system()` 与 `.messages()` 混用冲突 | 两条 system 消息叠加 | 二选一，避免重复设定角色 |
| 历史消息过长 | 未做裁剪 | 交给记忆 Advisor 管理 |

## 3.4 返回值形式总览

| 方法 | 返回 | 用途 |
|------|------|------|
| `.content()` | `String` | 只要文本 |
| `.chatResponse()` | `ChatResponse` | 需要元数据（Token、模型等） |
| `.entity(Class)` | 泛型对象 | 结构化输出（见 6） |
| `.entity(ParameterizedTypeReference)` | 泛型集合 | 列表/复杂泛型 |
| `.responseEntity(Class)` | `ResponseEntity<ChatResponse, T>` | 既要对象又要元数据 |

## 3.5 异常与超时

**边界**：模型调用是网络 I/O，可能抛出连接超时、限流、鉴权失败等异常。业务侧不要裸抛，应统一包装：

```java
try {
    return chatClient.prompt().user(q).call().content();
} catch (Exception e) {
    log.error("模型调用失败", e);
    throw new AiInvokeException("助手暂时不可用，请稍后重试", e);
}
```

# 4. 获取不同形式的返回值

## 4.1 content() 与 chatResponse()

**结论先行**：只需要文本用 `content()`；需要元数据（Token、模型、结束原因）用 `chatResponse()`。

```java
ChatResponse resp = chatClient.prompt()
        .user("你好")
        .call()
        .chatResponse();

String text = resp.getResult().getOutput().getText();
Usage usage = resp.getMetadata().getUsage();   // Token 统计
System.out.println("输入 Token: " + usage.getPromptTokens());
System.out.println("输出 Token: " + usage.getCompletionTokens());
```

### 4.1.1 Usage 元数据

`Usage` 是成本与限流的数据来源，典型字段如下：

| 字段 | 含义 | 用途 |
|------|------|------|
| `getPromptTokens()` | 输入 Token | 上下文成本 |
| `getCompletionTokens()` | 输出 Token | 生成成本 |
| `getTotalTokens()` | 合计 | 计费与限流 |

### 4.1.2 结果结构

`ChatResponse` 是一棵结果树：`getResult().getOutput().getText()` 取模型文本，`getMetadata().getUsage()` 取 Token 统计，`getMetadata().getModel()` 取实际使用的模型。理解这棵结构有助于正确取值，避免类型混淆。

## 4.2 entity() 与 responseEntity()

**机制**：`.entity(T.class)` 在内部使用 `BeanOutputConverter`，自动生成 JSON Schema 并解析响应为对象。`.responseEntity(T.class)` 则在返回对象的同时保留 `ChatResponse` 元数据。详见 `6_structured_output.md`。

```java
ResponseEntity<ChatResponse, ActorFilms> re = chatClient.prompt()
        .user("列出成龙的三部电影")
        .call()
        .responseEntity(ActorFilms.class);

ActorFilms entity = re.entity();
Usage usage = re.response().getMetadata().getUsage();
```

## 4.3 对比表

| 方法 | 是否含元数据 | 是否结构化 | 典型场景 |
|------|--------------|------------|----------|
| `content()` | 否 | 否 | 简单问答 |
| `chatResponse()` | 是 | 否 | 成本监控、审计 |
| `entity()` | 否 | 是 | 抽取、分类 |
| `responseEntity()` | 是 | 是 | 既要结构又要计费 |

# 5. 调用参数

## 5.1 单次覆盖

**结论先行**：`.options(...)` 只影响本次调用，是最小粒度的参数控制。

```java
String answer = chatClient.prompt()
        .user("写一首关于春天的诗")
        .options(OpenAiChatOptions.builder()
                .temperature(1.2)
                .maxTokens(256)
                .build())
        .call()
        .content();
```

## 5.2 默认参数

**机制**：`defaultOptions` 在构建期设定，作为所有调用的基线。

```java
ChatClient client = builder
        .defaultOptions(OpenAiChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.3)
                .build())
        .build();
```

## 5.3 参数优先级

参数优先级从高到低：

```
单次 .options()  >  defaultOptions  >  application.yml 配置
```

**边界与坑**：单次 `.options()` 是**整体替换**还是**逐字段覆盖**，取决于具体实现与版本。稳妥做法是：需要覆盖时显式给出所有关键字段，不要依赖"只改一个字段、其余继承"。

## 5.4 常用参数速查

| 参数 | 作用 | 建议 |
|------|------|------|
| `model` | 指定模型 | 按任务分层选择 |
| `temperature` | 随机性 | 事实类 0~0.3；创作类 0.7~1.0 |
| `maxTokens` | 输出上限 | 控制成本与截断风险 |
| `topP` | 核采样 | 一般与 temperature 二选一 |
| `stopSequences` | 停止词 | 精确控制输出边界 |

# 6. 流式响应

## 6.1 stream().content()

**结论先行**：需要"打字机"体验时用 `.stream()`，它返回 `Flux`，适合 WebFlux 端点。

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .stream()
            .content();          // Flux<String>
}
```

## 6.2 带元数据的流式处理

**机制**：`.stream().chatResponse()` 返回 `Flux<ChatResponse>`，每个元素是一次增量（delta）。

```java
Flux<ChatResponse> flux = chatClient.prompt()
        .user(message)
        .stream()
        .chatResponse();

flux.subscribe(resp -> {
    String delta = resp.getResult().getOutput().getText();
    System.out.print(delta);
});
```

**边界与坑**：`subscribe()` 是异步的，直接用于 Web 端点会丢失背压与错误传播。生产环境应直接返回 `Flux` 交给框架处理。

## 6.3 SSE 与网关

### 6.3.1 WebFlux 端点

流式端点必须返回响应式类型，且 `produces` 指定 `text/event-stream`：

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> chat(@RequestParam String q) {
    return chatClient.prompt().user(q).stream().content()
            .map(chunk -> ServerSentEvent.<String>builder(chunk).build());
}
```

### 6.3.2 缓冲问题

**生产建议**：流式适合聊天界面；**注意在网关/代理层关闭缓冲**（如 Nginx `proxy_buffering off`），否则看不到"打字机"效果。生产细节见 `14_ProductionEngineering/Architecture/2_webflux_sse.md`。

| 现象 | 原因 | 解决 |
|------|------|------|
| 响应一次性到达 | 代理缓冲 | 关闭 `proxy_buffering` |
| 首字节延迟高 | 上游未 flush | 配置 flush 间隔 |
| 中途断流 | 超时 | 设置合理的读超时 |

# 7. 默认设定（构建期）

## 7.1 defaultSystem 与 defaultAdvisors

**结论先行**：把"所有请求都该有"的东西放到默认设定，把"本次特有"的放到 `.prompt()` 链上。下面给出把 system、Advisor、工具、参数一次性注入的完整示例。

## 7.2 defaultTools 与 defaultOptions

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultSystem("你是企业知识库助手，只依据提供的资料回答。")
        .defaultAdvisors(
                new SimpleLoggerAdvisor(),
                MessageChatMemoryAdvisor.builder(chatMemory).build()
        )
        .defaultTools(new OrderTools(), new UserTools())
        .defaultOptions(ChatOptions.builder().temperature(0.2).build())
        .build();
```

## 7.3 构建期与运行期划分

| 内容 | 放哪里 | 理由 |
|------|--------|------|
| 角色设定、安全边界 | `defaultSystem` | 全局一致 |
| 日志、记忆 Advisor | `defaultAdvisors` | 横切复用 |
| 常用工具 | `defaultTools` | 避免重复注册 |
| 默认参数 | `defaultOptions` | 统一基线 |
| 本次问题、临时工具 | `.prompt()` 链 | 局部差异 |

**边界与坑**：默认设定过多会降低可读性，且难以针对单次调用关闭。对"偶发才需要"的能力（如某工具），宁可放到链上显式开启。

# 8. 多模型路由示例

## 8.1 静态路由

**结论先行**：最简单的路由是构建两个 `ChatClient`，按业务条件选择。

```java
@Service
public class ModelRouter {

    private final ChatClient fast;    // 便宜、快
    private final ChatClient strong;  // 贵、强

    public ModelRouter(ChatModel cheapModel, ChatModel strongModel) {
        this.fast = ChatClient.builder(cheapModel).build();
        this.strong = ChatClient.builder(strongModel).build();
    }

    public String ask(String question, boolean complex) {
        return (complex ? strong : fast).prompt()
                .user(question)
                .call()
                .content();
    }
}
```

## 8.2 动态路由策略

**机制**：路由依据可以是问题长度、关键词、历史失败次数，或先小模型试答再决定是否升级。

```java
public String ask(String question) {
    boolean complex = question.length() > 200
            || question.contains("分析")
            || question.contains("推理");
    return (complex ? strong : fast).prompt().user(question).call().content();
}
```

| 策略 | 判据 | 适用 |
|------|------|------|
| 长度阈值 | 字符数 | 快速实现 |
| 关键词 | 领域词表 | 可控 |
| 分类器 | 小模型判定 | 更准但更贵 |
| 升级重试 | 低置信度时换强模型 | 质量优先 |

## 8.3 边界与坑

**反例**：不要仅凭 `temperature` 高低就断言"强/弱模型"，不同厂商模型的参数语义与默认值差异很大。路由决策应基于评测数据，而非直觉。

> 更完整的路由策略见 `4_LLMEngineering/9_model_routing.md`。

# 9. 生产注意事项

## 9.1 超时与重试

| 风险 | 建议 |
|------|------|
| 超时不可控 | 配置 HTTP 客户端连接/读取超时 |
| 无重试 | 对幂等的只读调用加重试与退避 |
| 重试放大 | 设置最大重试次数与熔断 |

**注意**：具体超时配置键随版本变化，不确定时以 `RestClient`/`WebClient` 层的超时配置为准。

## 9.2 成本与限流

**结论先行**：Token 是成本与限流的核心指标，必须采集并告警。

| 风险 | 建议 |
|------|------|
| Token 失控 | 限制 `maxTokens`、监控 Usage |
| 并发超限 | 客户端限流、队列化 |
| 成本突增 | 按用户/租户维度计量 |

## 9.3 可观测性

**机制**：通过 Advisor 统一记录请求、响应、耗时与 Token，避免每个业务方法各自埋点。

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(new SimpleLoggerAdvisor())
        .build();
```

## 9.4 安全

| 风险 | 建议 |
|------|------|
| 密钥硬编码 | 走环境变量/配置中心 |
| 异常裸抛 | 统一异常处理，返回友好错误 |
| 阻塞 Web 线程 | 高并发场景用 `stream()` + WebFlux |
| 敏感信息外泄 | 日志脱敏，勿记录完整用户隐私 |

# 10. 小结

## 10.1 核心要点

- `ChatClient` 是日常入口，`.prompt()...call()` 是主链路。
- 默认设定放"公共基线"，链式调用放"本次差异"。
- 记得采集 `Usage`，它是成本与限流的数据来源。
- 流式 + WebFlux 是高并发对话系统的标准形态。

## 10.2 落地清单

- 多模型路由是成本与质量的杠杆，但决策要基于评测。
- 超时、重试、限流、脱敏是上线前的必查项。
