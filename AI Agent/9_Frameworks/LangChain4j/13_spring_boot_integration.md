# 1. 集成方式总览

结论先行：**原型期用 Starter 提效，生产期在关键装配点用手动 Bean 兜底**。LangChain4j 与 Spring Boot 的集成有两条路径：社区 Spring Boot Starter（自动配置 + 声明式 `@AiService`）与纯手动 `@Bean` 装配。本文以社区 Starter 为主，同时给出手动配置方式。

## 1.1 两种集成方式

| 方式 | 依赖 | 特点 |
|------|------|------|
| 社区 Spring Boot Starter | `langchain4j-{model}-spring-boot-starter` | 自动配置 + `@AiService` 声明式 |
| 纯手动配置 | `langchain4j` + `@Bean` | 完全可控，无额外抽象 |

### 1.1.1 社区 Starter 方式

Starter 读取 `langchain4j.*` 配置，装配 `ChatModel`（对话模型）、`StreamingChatModel`（流式对话模型）、`EmbeddingModel`（嵌入模型）等 Bean，并扫描 `@AiService` 接口注册为 Spring Bean，业务侧直接注入即可。

### 1.1.2 纯手动配置方式

不引入 Starter，直接构建模型并交给 `AiServices` 装配，适合需要精细控制监听器、超时、重试与多实例的场景。代价是配置样板代码更多，需自行管理生命周期。

## 1.2 选择建议

- 只需一个模型、一个助手：用 Starter，代码量最小；
- 需要多模型路由、自定义 `ChatModelListener`：手动装配为主；
- 已有大量 `@Bean` 基础设施：手动装配避免与自动配置打架。

## 1.3 边界与坑

- Starter 是**社区项目**，版本节奏与 `langchain4j` 核心不一定同步；
- 自动配置默认 `@ConditionalOnMissingBean`，自定义同名 Bean 会使其退让；
- `@AiService` 的装配错误往往在**启动时**才暴露，需测试覆盖。

## 1.4 生产建议

- 用 BOM 锁死版本，避免 Starter 与核心错配；
- 业务接口与模型 Bean 分层，业务只依赖 `AiService`。

# 2. 依赖与版本管理

结论先行：**统一用 BOM（Bill of Materials，依赖版本清单）管理版本**，是避免 `NoSuchMethodError`、`NoClassDefFoundError` 的第一道防线。

## 2.1 BOM 统一版本

通过 `dependencyManagement` 导入 `langchain4j-bom`，子依赖无需再写版本号。

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

若某 Starter 未纳入 BOM，可单独指定版本，但必须与核心版本对齐。

## 2.2 Starter 依赖清单

| 依赖 | 作用 |
|------|------|
| `langchain4j-spring-boot-starter` | 核心自动配置 + `@AiService` |
| `langchain4j-open-ai-spring-boot-starter` | OpenAI 协议模型自动配置 |
| `langchain4j-ollama-spring-boot-starter` | Ollama 本地模型 |
| `langchain4j-community-*-spring-boot-starter` | 社区模型（如 DashScope） |

## 2.3 版本冲突排查

用 `mvn dependency:tree -Dincludes=dev.langchain4j` 查看依赖树；若同一 `langchain4j-core` 出现多个版本，用 `<exclusions>` 剔除传递依赖，再由 BOM 统一。

## 2.4 边界与坑

- Starter 版本号常带 `-beta` 后缀，与核心 `1.x` 主版本不同；
- 不同厂商 Starter 可能各自携带不同版本的 `langchain4j-core`；
- 升级核心版本时，务必同步升级全部 Starter。

## 2.5 生产建议

- CI 中加入 `dependency:tree` 检查，禁止多版本共存；
- 升级走"核心 + 全部 Starter"原子提交，便于回滚；
- 用锁文件（如 Gradle lockfile）固化依赖。

# 3. 模型自动配置

结论先行：**Starter 会把 YAML 配置翻译成模型 Bean**，你只需注入使用。

## 3.1 YAML 配置项

### 3.1.1 对话模型

```yaml
langchain4j:
  open-ai:
    chat-model:
      api-key: ${OPENAI_API_KEY}
      model-name: gpt-4o-mini
      temperature: 0.3
      timeout: 60s
      log-requests: false
      log-responses: false
```

### 3.1.2 流式与嵌入模型

`streaming-chat-model` 生成流式 Bean，`embedding-model` 生成嵌入 Bean，二者共用同一 `api-key`，配置方式与对话模型一致，只需把 `chat-model` 换成对应节点名并填写 `model-name`。

## 3.2 自动装配的 Bean

| 配置前缀 | 产生的 Bean 类型 |
|----------|------------------|
| `chat-model` | `ChatModel` |
| `streaming-chat-model` | `StreamingChatModel` |
| `embedding-model` | `EmbeddingModel` |
| `image-model` | `ImageModel` |

只有配置了对应节点，才会创建该 Bean。

## 3.3 多模型（自定义 Bean）

当需要快慢双模型时，自定义 `@Bean` 并用 `@Qualifier`（限定符）命名区分。

### 3.3.1 @Qualifier 区分

```java
@Configuration
public class ModelConfig {

    @Bean
    @Qualifier("fastModel")
    ChatModel fastModel(@Value("${app.ai.fast.api-key}") String apiKey) {
        return OpenAiChatModel.builder()
                .apiKey(apiKey)
                .modelName("gpt-4o-mini")
                .build();
    }

    @Bean
    @Qualifier("strongModel")
    ChatModel strongModel(@Value("${app.ai.strong.api-key}") String apiKey) {
        return OpenAiChatModel.builder()
                .apiKey(apiKey)
                .modelName("gpt-4o")
                .build();
    }
}
```

### 3.3.2 与自动配置共存

自定义 Bean 默认名称为方法名（`fastModel`、`strongModel`），与自动配置的 `openAiChatModel` 不冲突，可共存。若要完全接管，可在配置中禁用自动配置节点。

## 3.4 边界与坑

- 多个 `ChatModel` 注入时**必须**用 `@Qualifier`，否则 `NoUniqueBeanDefinitionException`；
- `timeout` 字段名在不同版本可能不同（`timeout` vs `request-timeout`），需对照文档；
- `log-requests` 会打印完整提示，**生产环境可能泄漏敏感数据**。

## 3.5 生产建议

- 快慢模型分别配置独立密钥与配额，便于成本核算；
- 默认关闭 `log-requests` / `log-responses`，仅在排障时开启；
- 为每个模型显式配置 `timeout` 与 `maxRetries`。

# 4. 声明式 AI Service（@AiService）

结论先行：`@AiService` 用一个接口 + 注解，把模型、工具、记忆、RAG **一次装配完成**，是 Starter 的核心卖点。

## 4.1 注解属性

| 属性 | 说明 |
|------|------|
| `chatModel` | 使用的 `ChatModel` Bean 名 |
| `streamingChatModel` | 流式模型 Bean 名 |
| `tools` | 工具 Bean 名数组 |
| `chatMemory` / `chatMemoryProvider` | 记忆 Bean 名 |
| `contentRetriever` | 检索器 Bean 名 |
| `retrievalAugmentor` | 检索增强器 Bean 名 |

## 4.2 完整示例

```java
@AiService(
        chatModel = "fastModel",
        tools = {"weatherTools", "orderTools"},
        chatMemoryProvider = "chatMemoryProvider",
        contentRetriever = "contentRetriever"
)
public interface Assistant {

    @SystemMessage("你是一个专业的中文助手。")
    String chat(@MemoryId String sessionId, @UserMessage String message);
}
```

## 4.3 装配规则

只写 `chatModel` 时仅装配对话模型；同时写 `chatModel` 与 `streamingChatModel` 才支持同步与流式两种调用；引用不存在的 Bean 名会在启动期报错；接口会自动注册为 Spring Bean，可直接注入。

## 4.4 边界与坑

- `tools` / `contentRetriever` 引用的 Bean 必须存在，否则启动失败；
- 注解属性是**字符串 Bean 名**，IDE 无法重构重命名；
- 同名 `@AiService` 接口多实例时，需用 `@Qualifier` 注入。

## 4.5 生产建议

- 用常量类管理 Bean 名，减少字符串魔法值；
- 为 `@AiService` 接口写集成测试，覆盖启动装配；
- 复杂装配改用手动 `@Bean`，避免注解膨胀。

# 5. 手动装配 AI Service

结论先行：**不用 `@AiService` 时，`AiServices.builder` 提供等价且更可控的装配**。

## 5.1 AiServices.builder

```java
@Configuration
public class AiServiceConfig {

    @Bean
    Assistant assistant(ChatModel model,
                        ChatMemoryProvider memoryProvider,
                        WeatherTools weatherTools,
                        ContentRetriever retriever) {
        return AiServices.builder(Assistant.class)
                .chatModel(model)
                .chatMemoryProvider(memoryProvider)
                .tools(weatherTools)
                .contentRetriever(retriever)
                .build();
    }
}
```

## 5.2 记忆 Provider

`ChatMemoryProvider`（记忆提供者）按 `memoryId` 惰性创建记忆实例，配合持久化存储可跨重启保留上下文。

```java
@Bean
ChatMemoryProvider chatMemoryProvider(ChatMemoryStore store) {
    return memoryId -> MessageWindowChatMemory.builder()
            .id(memoryId)
            .maxMessages(20)
            .chatMemoryStore(store)
            .build();
}
```

## 5.3 两种方式对比

| 维度 | `@AiService` | 手动 `AiServices` |
|------|--------------|-------------------|
| 声明位置 | 接口注解 | `@Bean` 方法 |
| 动态性 | 编译期固定 | 运行期可编程 |
| 多实例 | 较难 | 容易 |
| 可测性 | 依赖容器 | 可直接 `new` |
| 可读性 | 简洁 | 显式 |

## 5.4 边界与坑

- `AiServices.builder` 每次 `build()` 都生成代理，**不要在高频路径反复构建**；
- `tools` 传对象或类，混用会引发重载歧义；
- 记忆 Provider 若返回共享实例，会导致会话串扰。

## 5.5 生产建议

- 把 `Assistant` Bean 设计为单例，缓存代理；
- 记忆按 `tenantId:userId` 拼接 `memoryId`，避免串扰；
- 手动装配时显式挂载 `ChatModelListener`。

# 6. Web 层集成

结论先行：**Controller 只负责协议转换，业务编排下沉到 Service**，模型调用不直接暴露给 Web 层。

## 6.1 同步接口

```java
@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
public class ChatController {

    private final Assistant assistant;

    @PostMapping
    public String chat(@RequestHeader("X-Session-Id") String sessionId,
                       @RequestBody String message) {
        return assistant.chat(sessionId, message);
    }
}
```

## 6.2 流式接口（SSE）

### 6.2.1 StreamingChatModel 方式

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestHeader("X-Session-Id") String sessionId,
                           @RequestParam String message) {
    return Flux.create(sink -> {
        streamingModel.chat(message, new StreamingChatResponseHandler() {
            public void onPartialResponse(String token) { sink.next(token); }
            public void onCompleteResponse(ChatResponse r) { sink.complete(); }
            public void onError(Throwable t) { sink.error(t); }
        });
    });
}
```

### 6.2.2 取消与背压

客户端断开时，`Flux` 的取消信号需传递到模型请求，否则后台仍持续消耗 Token。可借助 `sink.onCancel(...)` 或 `TokenStream` 的 `cancel()`。

## 6.3 边界与坑

- 同步接口在模型慢时占用 Web 线程，需配置足够线程池或改用流式；
- SSE 需要禁用响应缓冲（如 Nginx `proxy_buffering off`）；
- `@RequestBody String` 直接当消息，缺少长度与注入校验。

## 6.4 生产建议

- 流式接口设置整体超时与心跳；
- 对请求做长度限制、敏感词与注入检测；
- 统一异常处理，把模型错误映射为可读的 HTTP 状态。

# 7. 持久化记忆

结论先行：**默认记忆存内存，重启即丢**；生产必须用 `ChatMemoryStore`（记忆存储）落到 JDBC 或 Redis。

## 7.1 为什么持久化

| 问题 | 影响 |
|------|------|
| 内存记忆 | 重启/多实例丢失 |
| 无窗口限制 | 上下文无限增长、成本飙升 |
| 无隔离 | 会话互相污染 |

## 7.2 JDBC 记忆存储

```java
@Configuration
public class MemoryConfig {

    @Bean
    ChatMemoryStore chatMemoryStore(JdbcTemplate jdbcTemplate) {
        return new JdbcChatMemoryStore(jdbcTemplate);
    }

    @Bean
    ChatMemoryProvider chatMemoryProvider(ChatMemoryStore store) {
        return memoryId -> MessageWindowChatMemory.builder()
                .id(memoryId)
                .maxMessages(20)
                .chatMemoryStore(store)
                .build();
    }
}
```

## 7.3 Redis 记忆存储

Redis 适合多实例共享与低延迟读取，实现 `ChatMemoryStore` 后注入 `MessageWindowChatMemory` 即可，需设置 TTL 防止无限堆积。

## 7.4 边界与坑

- `MessageWindowChatMemory` 按**消息条数**裁剪，不是按 Token，长消息仍可能超限；
- 持久化存储的序列化格式变更可能不兼容旧数据；
- 并发写入同一 `memoryId` 需加锁，避免丢更新。

## 7.5 生产建议

- 记忆加 TTL 与容量上限；
- 定期归档冷会话；
- 敏感内容在落库前脱敏。

# 8. 向量库与 RAG 装配

结论先行：**RAG（Retrieval-Augmented Generation，检索增强生成）装配 = EmbeddingStore + EmbeddingModel + ContentRetriever**，三者都应是可替换的 Bean。

## 8.1 EmbeddingStore

```java
@Bean
EmbeddingStore<TextSegment> embeddingStore() {
    return PgVectorEmbeddingStore.builder()
            .host("localhost")
            .port(5432)
            .database("ai")
            .user("postgres")
            .password(System.getenv("DB_PASSWORD"))
            .table("embeddings")
            .dimension(1536)
            .createTable(true)
            .build();
}
```

## 8.2 ContentRetriever

```java
@Bean
ContentRetriever contentRetriever(EmbeddingStore<TextSegment> store,
                                  EmbeddingModel embeddingModel) {
    return EmbeddingStoreContentRetriever.builder()
            .embeddingStore(store)
            .embeddingModel(embeddingModel)
            .maxResults(5)
            .minScore(0.7)
            .build();
}
```

## 8.3 边界与坑

- `dimension` 必须与嵌入模型输出维度一致，否则写入失败；
- `minScore` 过低会引入噪声，过高会漏召回；
- `createTable(true)` 仅适合开发，生产应走迁移脚本。

## 8.4 生产建议

- 向量表与业务库隔离，独立扩缩容；
- 索引元数据带 `tenantId`，配合 `dynamicFilter` 做隔离；
- 记录检索命中率与延迟指标。

# 9. 可观测性集成

结论先行：**通过 `ChatModelListener` 观测每次模型交互**，是成本、性能与审计的统一入口。

## 9.1 ChatModelListener

```java
public class LoggingListener implements ChatModelListener {

    @Override
    public void onRequest(ChatModelRequestContext ctx) {
        log.info("request id={}", ctx.chatRequest().messages().size());
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        log.info("tokens={}", ctx.chatResponse().tokenUsage());
    }
}
```

## 9.2 指标与追踪

把监听器接入 `MeterRegistry`，即可按模型/租户输出下列指标。

| 指标 | 含义 |
|------|------|
| 调用次数 | 按模型/租户 |
| 延迟 | P50/P95/P99 |
| Token 用量 | 输入/输出 |
| 错误率 | 超时、限流、5xx |

自动配置的模型**未必**自动挂载监听器；稳妥做法是在自定义 `ChatModel` Bean 中显式 `.listeners(listener)`。

## 9.3 边界与生产建议

- 监听器异常可能中断模型调用，需内部捕获；
- 记录完整提示会带来隐私与存储成本，应异步、限长、脱敏；
- 指标基数过高（如按 `sessionId` 打标签）会拖垮监控系统，按模型/租户聚合。

# 10. 与事务、安全的协作

结论先行：**LLM（Large Language Model，大语言模型）调用必须移出数据库事务**，并纳入 Spring Security 的权限上下文。

## 10.1 事务边界

```java
@Service
@RequiredArgsConstructor
public class OrderAssistantService {

    private final Assistant assistant;

    public String ask(String sessionId, String question) {
        String answer = assistant.chat(sessionId, question);
        auditService.record(sessionId, question, answer);
        return answer;
    }
}
```

不要在事务内调用 LLM：模型调用耗时长，会长时间占用数据库连接。

## 10.2 异步与线程池

模型调用耗时长，可放到独立线程池配合 `@Async`，但 `ThreadLocal`（如 SecurityContext、MDC）不自动传递，需显式传播。

## 10.3 Spring Security

| 关注点 | 做法 |
|--------|------|
| 鉴权 | 校验用户可访问的工具与数据 |
| 租户 | 从认证信息取 `tenantId` 注入检索过滤 |
| 审计 | 记录调用者、模型、耗时、Token |

## 10.4 边界与坑

- `@Transactional` 包裹 LLM 调用会长时间占用连接，拖垮连接池；
- 异步线程中 `SecurityContextHolder` 默认不可见；
- 审计写入失败不应影响主流程，建议异步或补偿。

## 10.5 生产建议

- 事务只包裹短小的数据库操作，LLM 调用放事务外；
- 审计与指标异步化；
- 权限在工具调用层二次校验，防止提示注入绕过。

# 11. 生产注意事项汇总

结论先行：**上线的门槛是可靠性、安全、成本、可观测四条线同时达标**。

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| Starter 版本与核心不一致 | 用 BOM 统一版本 |
| 自动配置覆盖自定义 Bean | 理解 `@ConditionalOnMissingBean` 语义 |
| 事务内调用 LLM | 移出事务 |
| 密钥进配置文件 | 环境变量/配置中心 |
| 无超时/重试 | 在模型构建器上显式配置 |
| 多模型 Bean 冲突 | 用 `@Qualifier` 区分 |
| 记忆不持久 | 用 `ChatMemoryStore` |
| 日志泄漏提示 | 默认关闭请求/响应日志 |

## 11.2 上线检查表

- BOM 统一、无多版本共存；
- 所有模型配置超时与重试；
- LLM 调用在事务外，记忆持久化并带 TTL；
- 监听器已挂载且脱敏，密钥来自环境变量或配置中心。

# 12. 小结

- 社区 Starter 提供自动配置与 `@AiService` 声明式能力。
- `@AiService` 把模型、工具、记忆、RAG 一次装配完成。
- 复杂场景用 `@Bean` 手动装配，更可控。
- LLM 调用务必移出数据库事务。
- 版本、密钥、记忆持久化、可观测是生产四要素。
