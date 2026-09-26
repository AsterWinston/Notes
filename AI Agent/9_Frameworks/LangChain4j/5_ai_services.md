# 1. 什么是 AI Services

**结论先行**：`AiServices`（AI 服务）让开发者只定义 Java 接口并加注解，框架自动生成实现，把提示、记忆、工具、RAG、结构化输出统一封装，业务代码从此只依赖接口。

## 1.1 核心价值

`AiServices` 是 LangChain4j 最具特色的能力：**定义一个 Java 接口，框架自动生成实现**，把提示、记忆、工具、RAG、结构化输出全部封装起来。

```java
interface Assistant {
    @SystemMessage("你是一个专业的中文助手。")
    String chat(@UserMessage String userMessage);
}

// 框架生成代理实现
Assistant assistant = AiServices.create(Assistant.class, model);
String answer = assistant.chat("介绍一下 LangChain4j");
```

### 1.1.1 声明式编排

提示、记忆、工具、检索这些"编排逻辑"全部由注解和 Builder 声明，而不是散落在业务方法里。业务侧只看到 `assistant.chat(...)` 这样的方法调用。

### 1.1.2 与低层 API 对比

对比低层写法：

```java
// 低层：需要自己组装消息、解析响应
ChatResponse response = model.chat(List.of(
        SystemMessage.from("你是一个专业的中文助手。"),
        UserMessage.from("介绍一下 LangChain4j")));
String answer = response.aiMessage().text();
```

| 维度 | 低层 API | AI Services |
|------|----------|-------------|
| 消息组装 | 手动 | 注解自动 |
| 结构化输出 | 手动解析 | 类型映射 |
| 记忆 | 手动传历史 | `@MemoryId` |
| 工具循环 | 手动实现 | 自动 |
| 业务侵入 | 高 | 低（只依赖接口） |

> AI Services 的价值：**把"提示工程 + 工程编排"声明化，业务代码只依赖接口。**

## 1.2 工作流程

### 1.2.1 代理生成机制

```
interface Assistant
        │  AiServices.builder(...).build()
        ▼
动态代理 / 字节码生成
        │  方法调用被拦截
        ▼
读取注解 → 组装消息 → 调用模型 → 解析结果
        │
        ▼
返回 String / POJO / TokenStream / Result
```

### 1.2.2 调用时序

```
业务 ──chat("...")──► 代理 ──► 记忆注入 ──► 检索注入
                          │
                          ▼
                       模型调用 ──► 工具循环（若有）
                          │
                          ▼
                       解析返回值 ──► 业务
```

## 1.3 边界与坑

- 接口方法一旦被调用方依赖，就成为契约，改动需评审；
- 代理在运行时生成，注解写错往往到首次调用才暴露，建议写冒烟测试；
- 大量小接口会增加 Bean 数量，需在"复用"与"数量"间平衡。

# 2. 创建方式

**结论先行**：简单场景用 `AiServices.create`，生产一律用 `AiServices.builder`，因为记忆、工具、RAG 等能力只能通过 Builder 配置。

## 2.1 最简创建

```java
Assistant assistant = AiServices.create(Assistant.class, model);
```

`create` 等价于"只挂一个模型"，适合探针与最简单的问答。

## 2.2 Builder 方式

### 2.2.1 常用配置项

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
| `chatModel` | 指定对话模型 |
| `streamingChatModel` | 指定流式模型 |
| `chatMemory` / `chatMemoryProvider` | 记忆 |
| `tools` | 工具集 |
| `contentRetriever` / `retrievalAugmentor` | RAG |
| `moderationModel` | 内容审核 |

### 2.2.2 多模型

可在同一个 Builder 里同时指定同步与流式模型，让接口按返回类型自动选择：

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .streamingChatModel(streamingModel)
        .build();
```

## 2.3 边界与坑

- 只配 `chatModel` 却声明 `TokenStream` 返回值会失败，需同时配 `streamingChatModel`；
- `chatMemory` 是全局共享单例，多用户场景必须用 `chatMemoryProvider`；
- Builder 不校验工具/检索器的线程安全，需自行保证。

# 3. 参数与返回值

**结论先行**：单参数默认作为用户消息；多参数必须用 `@V` 显式命名；返回值支持文本、消息、流式、POJO、`Result` 等多种类型。

## 3.1 参数解析

### 3.1.1 单参数

```java
interface Assistant {
    String chat(String userMessage);   // 默认作为 UserMessage
}
```

单参数若不加注解，框架默认把它当作 `UserMessage` 内容。

### 3.1.2 多参数 @V

```java
interface Assistant {
    @UserMessage("从{{text}}中抽取{{field}}")
    String extract(@V("text") String text, @V("field") String field);
}
```

> 多参数时若不加 `@V`，LangChain4j 会报错——这是防止"参数丢失"的显式约束。

## 3.2 返回值类型

### 3.2.1 文本与消息

| 返回类型 | 说明 |
|----------|------|
| `String` | 文本 |
| `AiMessage` | 完整消息（含工具请求） |
| `ChatResponse` | 响应 + Token 元数据 |
| `TokenStream` | 流式 |
| POJO / `record` / 枚举 / `List<T>` | 结构化输出 |
| `Result<T>` | 成功/失败封装（不抛异常） |

## 3.3 边界与坑

- `String` 返回值会丢失 Token 元数据，需要计量时返回 `ChatResponse`；
- `AiMessage` 可能包含工具请求而非最终文本，消费前需判断；
- 泛型 `List<T>` 在部分厂商上解析稳定性不如单对象，建议小批量。

# 4. 结构化输出

**结论先行**：把返回值声明为 `record`/枚举/集合，框架会自动生成 JSON Schema 并解析响应，是比"提示里要求 JSON"更可靠的做法。

## 4.1 record 映射

```java
record Person(String name, int age, String occupation) {}

interface Extractor {
    @UserMessage("从文本中抽取人物信息：{{it}}")
    Person extract(String text);
}

Person p = extractor.extract("我叫张三，28 岁，是后端工程师");
```

## 4.2 集合与枚举

```java
interface Extractor {
    List<Person> extractAll(String text);
}
```

```java
enum Sentiment { POSITIVE, NEUTRAL, NEGATIVE }

interface Analyzer {
    Sentiment analyze(String text);
}
```

## 4.3 嵌套对象

```java
record Order(String orderId, Customer customer, List<Item> items) {
    record Customer(String name, String phone) {}
    record Item(String sku, int quantity) {}
}
```

嵌套结构同样支持，但层级越深解析越易失败，生产优先扁平化。

## 4.4 校验与重试

```java
record UserProfile(
        @NotBlank String name,
        @Min(0) @Max(150) int age) {}
```

```java
UserProfile profile = extractor.extract(text);
validator.validate(profile);   // 解析后校验，失败重试或兜底
```

## 4.5 边界与坑

| 风险 | 说明 | 对策 |
|------|------|------|
| 多余文本 | 模型夹带解释 | 用支持严格模式的厂商 |
| 枚举越界 | 输出未定义值 | 默认值 + 校验 |
| 嵌套过深 | 泛型/多态支持有限 | 扁平化 |
| 解析失败 | 直接抛异常 | `Result<T>` + 重试 |

# 5. 不抛异常的 Result

**结论先行**：`Result<T>` 把"成功内容"与"失败原因"包在一个返回值里，适合不希望异常打断主流程、需要统一错误处理的场景。

## 5.1 基本用法

```java
interface Assistant {
    Result<String> chat(String message);
}

Result<String> result = assistant.chat("你好");
if (result.isSuccess()) {
    System.out.println(result.content());
} else {
    System.out.println("失败: " + result.error());
}
```

> `Result<T>` 适合**不希望异常打断流程**的场景，也便于统一错误处理。

## 5.2 适用场景与边界

- `Result` 只封装"调用与解析"的错误，业务校验失败仍需自行判断；
- 失败后是否重试由调用方决定，框架不会自动重试；
- 不要把 `Result` 当作吞掉所有异常的手段，严重故障仍应上抛。

# 6. 记忆集成

**结论先行**：用 `@MemoryId` 标注会话隔离键，配合 `chatMemoryProvider` 为每个会话提供独立记忆，是多用户场景的正确姿势。

## 6.1 @MemoryId

```java
interface Assistant {
    String chat(@MemoryId String userId, @UserMessage String message);
}
```

## 6.2 chatMemoryProvider

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .chatMemoryProvider(userId -> MessageWindowChatMemory.builder()
                .id(userId)
                .maxMessages(20)
                .build())
        .build();
```

- `@MemoryId` 指定会话隔离键；
- `chatMemoryProvider` 按 ID 提供独立的记忆实例。

## 6.3 边界与坑

- 使用 `chatMemory`（单例）会让所有用户共享历史，属严重缺陷；
- `@MemoryId` 的值来自业务，必须校验归属，防止越权读取他人历史；
- 记忆窗口过大会增加 Token 成本，需结合摘要或裁剪。

> 详见 `7_chat_memory.md`。

# 7. 工具集成

**结论先行**：把带 `@Tool` 的对象交给 Builder，框架自动完成"模型决定调用 → 执行 Java 方法 → 回填结果 → 生成回答"的循环。

## 7.1 注册工具

```java
interface Assistant {
    String chat(String message);
}

Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new WeatherTools(), new BookingTools())
        .build();

String answer = assistant.chat("北京今天天气如何？");
```

## 7.2 调用循环

```
用户提问 ──► 模型 ──► 工具调用请求
                         │
                         ▼
                    执行 Java 方法
                         │
                         ▼
                结果回填为 ToolExecutionResultMessage
                         │
                         ▼
                    模型 ──► 最终回答
```

> 框架自动完成"模型决定调用 → 执行 Java 方法 → 回填 → 生成回答"的循环。详见 `6_tools.md`。

## 7.3 边界与坑

- 工具集必须按权限裁剪，不能依赖"模型不会调用"；
- 工具内部要设超时，避免拖垮整个调用；
- 多轮工具调用要限制最大轮数，防止无限循环。

# 8. RAG 集成

**结论先行**：配置 `contentRetriever` 即可让 AI Service 在回答前自动检索并注入上下文；复杂链路用 `RetrievalAugmentor` 组合查询改写与内容聚合。

## 8.1 contentRetriever

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .contentRetriever(retriever)
        .build();

String answer = assistant.chat("报销流程是怎样的？");
```

`contentRetriever` 会在调用前自动检索并注入上下文。

## 8.2 RetrievalAugmentor

多检索器场景用 `RetrievalAugmentor`：

```java
RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
        .contentRetriever(retriever)
        .queryTransformer(queryTransformer)
        .contentAggregator(contentAggregator)
        .build();
```

| 组件 | 作用 |
|------|------|
| `ContentRetriever` | 召回内容 |
| `QueryTransformer` | 改写/扩展查询 |
| `ContentAggregator` | 去重、重排、截断 |

## 8.3 边界与坑

- 检索结果过长会挤占上下文，需限制条数与长度；
- 检索失败时应降级为"无资料"回答，而非直接报错；
- 必须提示模型"只依据资料回答"，否则仍可能编造。

> 详见 `8_rag.md`。

# 9. 流式 AI Service

**结论先行**：把返回值声明为 `TokenStream`，即可获得打字机式流式输出，适合对话类前端。

## 9.1 TokenStream

```java
interface Assistant {
    TokenStream chat(String message);
}

TokenStream stream = assistant.chat("讲个笑话");
stream.onPartialResponse(System.out::print)
      .onCompleteResponse(r -> System.out.println())
      .onError(Throwable::printStackTrace)
      .start();
```

### 9.1.1 回调钩子

| 钩子 | 触发时机 |
|------|----------|
| `onPartialResponse` | 每收到一段增量文本 |
| `onCompleteResponse` | 流正常结束 |
| `onError` | 出错 |
| `onRetrieved` | RAG 检索完成后 |

## 9.2 边界与坑

- `TokenStream` 是惰性的，必须 `start()` 才真正发起请求；
- 首包后出错只能通过 `onError` 处理，前端需支持半截输出；
- 流式下结构化输出与工具循环的支持有限，需核对版本。

# 10. 多模态 AI Service

**结论先行**：当方法参数是 `UserMessage` 时，可以携带文本与图片等多模态内容，实现图文问答。

## 10.1 图文输入

```java
interface VisionAssistant {
    String describe(@UserMessage UserMessage message);
}

UserMessage msg = UserMessage.from(
        TextContent.from("这张图有什么问题？"),
        ImageContent.from("https://example.com/diagram.png"));

String answer = visionAssistant.describe(msg);
```

## 10.2 边界与坑

- 多模态支持取决于具体模型，文本模型传图片会报错或忽略；
- 图片会显著增加 Token 与延迟，建议先压缩；
- 私网图片模型无法访问，需转 Base64 上传。

# 11. 非阻塞与响应式

**结论先行**：1.20 起支持异步返回类型，调用不再占用线程；该能力标记为 `@Experimental`，使用前需核对版本文档。

## 11.1 异步返回类型

1.20 起支持异步返回类型，调用不再占用线程：

```java
interface Assistant {
    CompletableFuture<String> chat(String message);   // 单次异步
    Flow.Publisher<String> stream(String message);    // 响应式流
}
```

## 11.2 边界与坑

> 该能力标记为 `@Experimental`；同步与 `TokenStream` API 不受影响。异步路径下多工具调用会并发执行，行为细节见官方文档。

- 异步路径下工具会并发执行，工具本身必须线程安全；
- 异常以 `CompletableFuture` 的完成态传递，需显式处理；
- `@Experimental` 的 API 可能在版本间变化，生产需锁版本。

# 12. 组合与复用

**结论先行**：用多个小型 AI Service 组合出复杂业务，比一个"万能提示"更可靠、更易测试；路由 + 专职助手是典型模式。

## 12.1 路由模式

```java
interface Router {
    @UserMessage("""
            判断用户意图类别，只输出其中一个：
            ORDER / REFUND / TECH / OTHER
            用户：{{it}}
            """)
    Intent classify(String message);
}
```

## 12.2 分层组合

```java
@Service
public class IntentService {

    private final Router router;
    private final OrderAssistant orderAssistant;
    private final RefundAssistant refundAssistant;

    public String handle(String message) {
        return switch (router.classify(message)) {
            case ORDER -> orderAssistant.chat(message);
            case REFUND -> refundAssistant.chat(message);
            default -> "暂不支持该请求";
        };
    }
}
```

> 用多个小型 AI Service 组合出复杂业务，比一个"万能提示"更可靠、更易测试。

## 12.3 边界与坑

- 路由分类本身也有成本，规则可覆盖时优先规则；
- 每个专职助手应各配独立的提示与工具集，避免相互污染；
- 组合层要处理"分类失败"的兜底路径。

# 13. 生产注意事项

**结论先行**：AI Service 的风险集中在"契约、解析、隔离、权限"四类，逐项设防即可。

## 13.1 稳定性

| 风险 | 建议 |
|------|------|
| 接口定义漂移 | 接口即契约，变更需评审 |
| 多参数漏标 @V | 显式声明，写单测 |
| 结构化解析失败 | `Result<T>` + 重试 + 兜底 |
| 流式无输出 | 记得调用 `start()` |

## 13.2 安全与合规

| 风险 | 建议 |
|------|------|
| 记忆未隔离 | `@MemoryId` + 校验归属 |
| 工具无鉴权 | 工具内部做权限校验 |
| 提示硬编码 | 模板外置版本化 |
| 敏感数据入日志 | 关闭请求日志 |

## 13.3 可观测

- 监听每次调用的 Token 用量，按接口维度归集成本；
- 记录路由分类结果，便于分析误分流；
- 采样结构化解析失败案例，用于优化提示。

# 14. 小结

- `AiServices` = 接口 + 注解，框架自动实现提示、记忆、工具、RAG。
- 多参数必须用 `@V`，返回值支持文本/对象/流式/`Result`。
- 小接口组合优于大而全的接口。
- 接口就是契约，要像设计 API 一样设计它。
- 记忆用 `@MemoryId` 隔离，工具按权限裁剪，是生产底线。
