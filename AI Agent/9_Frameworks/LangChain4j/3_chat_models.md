# 1. 模型抽象总览

LangChain4j 把"对话"抽象成统一接口，屏蔽厂商差异。理解模型抽象是使用框架的第一步。

## 1.1 接口家族

```
                    ┌─────────────────────┐
                    │     ChatModel       │  同步
                    │  StreamingChatModel │  流式
                    └──────────┬──────────┘
                               │
   ┌───────────┬───────────┬───┴────┬───────────┬──────────┐
   ▼           ▼           ▼        ▼           ▼          ▼
OpenAI    Anthropic     Gemini   Ollama      Qwen     DeepSeek ...
```

LangChain4j 的模型接口只关心"输入消息、输出消息"，**换厂商只需换实现类**。

### 1.1.1 同步与流式的关系

`StreamingChatModel` 不是 `ChatModel` 的子接口，而是并列接口。一个厂商实现类通常同时提供两者（如 `OpenAiChatModel` 与 `OpenAiStreamingChatModel`），但需分别构建。

### 1.1.2 其他模型接口

| 接口 | 用途 |
|------|------|
| `ChatModel` | 对话 |
| `StreamingChatModel` | 流式对话 |
| `EmbeddingModel` | 文本向量化 |
| `ImageModel` | 图像生成 |
| `ScoringModel` | 相关性打分（重排） |
| `ModerationModel` | 内容审核 |

## 1.2 换厂商的代价

| 切换内容 | 是否需要改代码 | 说明 |
|----------|---------------|------|
| 构建类 | 需要 | `OpenAiChatModel` → `AnthropicChatModel` |
| API Key / baseUrl | 需要 | 配置项变化 |
| 业务调用代码 | 不需要 | 依赖 `ChatModel` 接口 |
| 消息构造 | 不需要 | `ChatMessage` 体系统一 |
| 工具定义 | 不需要 | `@Tool` 与厂商无关 |

## 1.3 能力矩阵

| 能力 | ChatModel | StreamingChatModel | 说明 |
|------|-----------|--------------------|------|
| 单轮对话 | 是 | 是 | |
| 多轮对话 | 是 | 是 | 需自行传历史 |
| 工具调用 | 是 | 是 | 厂商需支持 |
| 多模态 | 视模型 | 视模型 | 文本模型不支持 |
| 结构化输出 | 是 | 部分 | 流式结构化较难 |
| Token 用量 | 多数 | 多数 | 需容忍 null |

# 2. 依赖引入

## 2.1 BOM 管理

用 `langchain4j-bom` 统一管理版本，避免子模块版本漂移：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-bom</artifactId>
            <version>1.20.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2.1.1 为什么需要 BOM

LangChain4j 有大量子模块，若各自写版本，升级时极易出现 `langchain4j-core` 与适配模块版本不一致，导致 `NoSuchMethodError`。

### 2.1.2 Gradle 写法

```groovy
dependencies {
    implementation platform("dev.langchain4j:langchain4j-bom:1.20.0")
    implementation "dev.langchain4j:langchain4j"
    implementation "dev.langchain4j:langchain4j-open-ai"
}
```

## 2.2 单模块引入

```xml
<dependencies>
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j</artifactId>
    </dependency>
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-open-ai</artifactId>
    </dependency>
</dependencies>
```

## 2.3 版本对齐

| 做法 | 效果 |
|------|------|
| 统一用 BOM | 推荐，避免冲突 |
| 每个依赖写版本 | 易漂移，不推荐 |
| Gradle platform | `platform("dev.langchain4j:langchain4j-bom:1.20.0")` |

### 2.3.1 冲突排查

用 `mvn dependency:tree` 检查 `dev.langchain4j` 下是否存在多个版本号，若有则强制对齐。

# 3. OpenAI 及兼容模型

## 3.1 基本构建

```java
ChatModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .temperature(0.3)
        .timeout(Duration.ofSeconds(60))
        .maxRetries(2)
        .logRequests(true)
        .logResponses(true)
        .build();
```

### 3.1.1 构建器要点

- 未设 `apiKey` 会直接抛异常，建议从环境变量读取；
- `timeout` 默认值可能偏长，生产应显式设置；
- `logRequests` 会打印完整提示，注意隐私。

### 3.1.2 复用与单例

`ChatModel` 实现通常线程安全，可作为单例 Bean 复用，无需每次请求新建。

## 3.2 指向兼容服务

绝大多数国产模型都提供 OpenAI 兼容接口，这是接入成本最低的路径：

```java
ChatModel model = OpenAiChatModel.builder()
        .baseUrl("https://dashscope.aliyuncs.com/compatible-mode/v1")
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-plus")
        .build();
```

> 只要协议兼容，就可以复用 `OpenAiChatModel`，无需引入厂商专用模块。

### 3.2.1 兼容度差异

| 特性 | 兼容情况 |
|------|----------|
| 基础对话 | 普遍兼容 |
| 流式 | 普遍兼容 |
| 工具调用 | 多数兼容 |
| 结构化输出（严格模式） | 部分兼容 |
| 多模态 | 部分兼容 |

## 3.3 常用参数

| 参数 | 作用 | 建议 |
|------|------|------|
| `modelName` | 模型名 | 按任务分层选择 |
| `temperature` | 随机性 | 事实类 0~0.3，创作类 0.7~1.0 |
| `topP` | 核采样 | 与 temperature 二选一 |
| `maxTokens` | 输出上限 | 控制成本与截断 |
| `timeout` | 超时 | 生产必设 |
| `maxRetries` | 重试 | 幂等场景可开 |
| `logRequests/Responses` | 日志 | 仅开发环境开启 |

## 3.4 边界与坑

- `temperature` 与 `topP` 同时调大易产生不稳定输出；
- `maxRetries` 对非幂等的工具调用可能重复执行；
- `logRequests` 会打印完整提示，可能泄露隐私，切勿在生产开启；
- 兼容服务的参数支持度不一，某些字段可能被忽略。

# 4. 流式模型

流式让首字延迟（TTFT）显著降低，是对话产品的标配。

## 4.1 回调式

```java
StreamingChatModel streamingModel = OpenAiStreamingChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .build();

streamingModel.chat("讲一个笑话", new StreamingChatResponseHandler() {
    @Override
    public void onPartialResponse(String partialResponse) {
        System.out.print(partialResponse);      // 打字机效果
    }

    @Override
    public void onCompleteResponse(ChatResponse completeResponse) {
        System.out.println("\n完成");
    }

    @Override
    public void onError(Throwable error) {
        error.printStackTrace();
    }
});
```

### 4.1.1 回调线程模型

回调可能在 IO 线程执行，**不要**在其中做重活；需要写数据库或推送时，转交业务线程池。

## 4.2 TokenStream（更易用）

`TokenStream` 把回调包装成链式 API，并支持 `start()` 触发与 `onRetrieved` 等钩子：

```java
TokenStream stream = streamingModel.chat("讲一个笑话");
stream.onPartialResponse(System.out::print)
      .onCompleteResponse(r -> System.out.println("\n完成"))
      .onError(Throwable::printStackTrace)
      .start();
```

### 4.2.1 start 的时机

`TokenStream` 是惰性的，必须调用 `start()` 才会真正发起请求；忘记调用会"静默无输出"。

## 4.3 阻塞式流式（SSE 场景）

在 Web 层把流式回调桥接为响应式流，实现 SSE 推送：

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String q) {
    return Flux.create(sink -> {
        streamingModel.chat(q, new StreamingChatResponseHandler() {
            public void onPartialResponse(String t) { sink.next(t); }
            public void onCompleteResponse(ChatResponse r) { sink.complete(); }
            public void onError(Throwable e) { sink.error(e); }
        });
    });
}
```

## 4.4 边界与坑

| 问题 | 说明 | 对策 |
|------|------|------|
| 网关缓冲 | 反向代理攒批导致无打字机效果 | 关闭响应缓冲 |
| 错误在流中 | 首包后失败只能通过 `onError` | 前端需处理半截输出 |
| 线程模型 | 回调可能在其他线程 | 注意 ThreadLocal 丢失 |
| 取消 | 用户关闭连接需中断 | 持有句柄并主动取消 |

# 5. 本地模型：Ollama

## 5.1 安装与拉取

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-ollama</artifactId>
</dependency>
```

```bash
ollama pull qwen2.5:7b
```

## 5.2 构建

```java
ChatModel model = OllamaChatModel.builder()
        .baseUrl("http://localhost:11434")
        .modelName("qwen2.5:7b")
        .temperature(0.7)
        .build();
```

### 5.2.1 流式与嵌入

Ollama 同样提供 `OllamaStreamingChatModel` 与 `OllamaEmbeddingModel`，可完全本地化 RAG 链路。

## 5.3 适用与代价

> 本地模型适合：数据不出内网、成本敏感、离线场景。代价是能力与并发不如云端。

| 维度 | 本地 Ollama | 云端 API |
|------|-------------|----------|
| 数据隐私 | 高（不出内网） | 依赖厂商承诺 |
| 成本 | 硬件一次性投入 | 按 Token 计费 |
| 能力 | 受显存与模型规模限制 | 顶级模型可选 |
| 并发 | 受 GPU 数量限制 | 弹性 |
| 运维 | 需自管部署 | 厂商托管 |

## 5.4 边界与坑

- 首次调用会加载模型，延迟较高；
- 上下文长度受显存限制，超长会报错；
- 模型名与本地已拉取版本必须一致，否则 404。

# 6. 其他厂商模型

## 6.1 厂商对照表

| 模型 | 依赖 | 构建类 |
|------|------|--------|
| Anthropic Claude | `langchain4j-anthropic` | `AnthropicChatModel` |
| Google Gemini | `langchain4j-google-ai-gemini` | `GoogleAiGeminiChatModel` |
| Azure OpenAI | `langchain4j-azure-open-ai` | `AzureOpenAiChatModel` |
| AWS Bedrock | `langchain4j-bedrock` | `BedrockChatModel` |
| 通义千问 | `langchain4j-community-*` 或 OpenAI 兼容 | `QwenChatModel` / `OpenAiChatModel` |
| DeepSeek | OpenAI 兼容 | `OpenAiChatModel` |

## 6.2 构建示例

```java
ChatModel claude = AnthropicChatModel.builder()
        .apiKey(System.getenv("ANTHROPIC_API_KEY"))
        .modelName("claude-sonnet-4-5")
        .build();

ChatModel gemini = GoogleAiGeminiChatModel.builder()
        .apiKey(System.getenv("GEMINI_API_KEY"))
        .modelName("gemini-2.5-flash")
        .build();
```

### 6.2.1 云厂商的额外参数

Azure 需要 `endpoint` 与 `deploymentName`；Bedrock 需要 `region` 与凭证链，通常走 SDK 默认凭证提供者。

## 6.3 边界与坑

- 各厂商对 `SystemMessage` 的处理不同，有的单独字段、有的拼进首条；
- 工具调用的 JSON Schema 支持度有差异；
- 部分厂商需要额外传入 `projectId` / `region` / `deploymentName`。

# 7. 多模态输入

## 7.1 图片输入

```java
UserMessage message = UserMessage.from(
        TextContent.from("这张图表说明了什么趋势？"),
        ImageContent.from("https://example.com/chart.png")
);
ChatResponse response = model.chat(message);
```

## 7.2 本地图片

```java
// 本地图片
ImageContent.from(new File("D:/chart.png").toPath(), "image/png")
```

| 输入方式 | 写法 | 说明 |
|----------|------|------|
| URL | `ImageContent.from(url)` | 模型需能访问 |
| 本地文件 | `ImageContent.from(path, mime)` | 会转 Base64 上传 |
| Base64 | `ImageContent.from(base64, mime)` | 避免重复读盘 |

### 7.2.1 音频与视频

部分模型支持 `AudioContent` / `VideoContent`，但 LangChain4j 的跨厂商支持程度不一，使用前需确认具体实现。

## 7.3 边界与坑

> 多模态支持取决于具体模型；文本模型传图片会报错或忽略。

- 大图会显著增加 Token 与延迟，建议压缩；
- 图片 URL 若在私网，模型无法访问，需上传 Base64；
- 不同厂商对图片数量与分辨率有限制。

# 8. 结构化输出（模型层）

## 8.1 直接映射

低层 API 支持直接指定目标类型：

```java
record Person(String name, int age) {}

Person person = model.chat(
        "提取：我叫张三，28 岁",
        Person.class);        // 直接映射为对象
```

## 8.2 JSON Schema 约束

框架会把 Java 类型转换为 JSON Schema 传给模型，要求模型按结构输出，再反序列化：

```
Person(record)
   │ 反射提取字段
   ▼
JSON Schema { name: string, age: integer }
   │ 作为提示/约束下发
   ▼
模型输出 JSON → 反序列化为 Person
```

### 8.2.1 支持严格模式的厂商

部分厂商支持 JSON Schema 严格模式，能显著提升解析成功率；不支持时框架会退化为"提示模型输出 JSON"。

## 8.3 边界与坑

- 模型可能输出多余文本，需依赖框架容错或使用支持严格模式的厂商；
- 嵌套泛型与多态类型支持有限，优先扁平结构；
- 解析失败会抛异常，建议配合 `Result<T>` 使用。

> 更完整的结构化输出见 `5_ai_services.md`（`AiServices` 方式更灵活）。

# 9. 参数化与多模型切换

## 9.1 分层路由

按任务复杂度选择不同档位模型，是降本的核心手段：

```java
public enum ModelTier { FAST, STANDARD, STRONG }

@Component
public class ModelRegistry {

    private final Map<ModelTier, ChatModel> models = new EnumMap<>(ModelTier.class);

    public ModelRegistry(OpenAiChatModel fast,
                         OpenAiChatModel standard,
                         OpenAiChatModel strong) {
        models.put(ModelTier.FAST, fast);
        models.put(ModelTier.STANDARD, standard);
        models.put(ModelTier.STRONG, strong);
    }

    public ChatModel get(ModelTier tier) {
        return models.get(tier);
    }
}
```

> 模型分层路由是降本的核心手段，详见 `4_LLMEngineering/9_model_routing.md`。

### 9.1.1 路由决策方式

| 方式 | 说明 |
|------|------|
| 规则路由 | 按任务类型硬编码 |
| 长度路由 | 短输入走小模型 |
| 分类路由 | 先用小模型判断难度 |
| 级联路由 | 小模型失败再升级 |

## 9.2 配置驱动

| 档位 | 典型模型 | 用途 |
|------|----------|------|
| FAST | 小参数模型 | 分类、改写、意图识别 |
| STANDARD | 中档模型 | 常规问答 |
| STRONG | 旗舰模型 | 复杂推理、代码生成 |

### 9.2.1 外部化配置

把档位与模型名写入配置文件或配置中心，切换模型无需重新编译。

## 9.3 边界与坑

- 路由决策本身也有成本，简单规则优先；
- 不同模型提示格式可能需微调；
- 降级到弱模型时要有质量兜底。

# 10. 生产注意事项

## 10.1 稳定性

| 风险 | 建议 |
|------|------|
| 未设超时 | 必设 `timeout`，避免线程挂死 |
| 无重试 | 只读调用可配 `maxRetries`，写操作需幂等 |
| 单点模型 | 配置备用模型，支持降级 |
| 流式被缓冲 | 网关/代理需关闭响应缓冲 |

## 10.2 安全与合规

| 风险 | 建议 |
|------|------|
| 日志泄露 | `logRequests/Responses` 仅开发开启 |
| 密钥硬编码 | 环境变量 / 配置中心 |
| 提示注入 | 用户内容与指令隔离，输出做校验 |
| 数据出境 | 按合规要求选择模型地域 |

## 10.3 成本与可观测

| 维度 | 做法 |
|------|------|
| Token 统计 | 监听器累计 `TokenUsage` |
| 调用追踪 | 注入 traceId 到日志 |
| 成本归因 | 按业务/用户打标签 |
| 质量监控 | 采样评估输出 |

# 11. 小结

- 模型接口统一，换厂商只改实现类与配置。
- OpenAI 兼容协议是接入国产模型的最快路径。
- 流式用 `StreamingChatModel` / `TokenStream`。
- 生产必设超时、重试、降级，并隔离密钥与日志。
- 多模型分层路由是控制成本与质量的关键工程手段。
