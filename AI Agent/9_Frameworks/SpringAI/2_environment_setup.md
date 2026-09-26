# 1. 版本要求

结论先行：Spring AI 的版本必须与 Spring Boot、JDK 三者对齐，否则最常见的症状就是"启动即报错"或"Bean 找不到"。

## 1.1 版本矩阵

| 组件 | 要求 |
|------|------|
| JDK | 17+（推荐 21） |
| Spring Boot | 3.x（Spring AI 1.0.x） / 4.x（Spring AI 2.0.x） |
| 构建工具 | Maven 3.8+ 或 Gradle 8+ |
| Spring AI | 1.0.x GA（本笔记主线） |

### 1.1.1 推荐组合

| Spring AI | Spring Boot | JDK | 说明 |
|-----------|-------------|-----|------|
| 1.0.x | 3.3+ / 3.4+ | 17 / 21 | 当前生产主力 |
| 1.1.x | 3.4+ | 17 / 21 | 增量增强 |
| 2.0.x | 4.0+ | 17 / 21 | 需大版本升级 |

### 1.1.2 不推荐组合

边界与坑：Spring AI 2.0 与 Spring Boot 3.x 组合会在启动阶段直接失败，因为自动配置类依赖 Boot 4 的 API。

## 1.2 版本兼容性说明

### 1.2.1 Spring Boot 与 Spring AI 的对应

```text
Spring Boot 3.x ──▶ Spring AI 1.0.x / 1.1.x
Spring Boot 4.x ──▶ Spring AI 2.0.x
```

Spring AI 的 BOM 会传递 Spring Framework 版本约束，因此不要手动指定 `spring-core` 等底层依赖版本。

### 1.2.2 JDK 选择

生产建议：优先 JDK 21（LTS），虚拟线程（Virtual Threads）可显著提升高并发下的阻塞式模型调用吞吐。

## 1.3 校验环境

```bash
java -version
mvn -v
```

```text
java version "21.0.x" ...
Apache Maven 3.9.x ...
```

结论先行：先确认 JDK 与构建工具版本，再动代码，可省去大量无效排查。

# 2. 引入依赖

## 2.1 使用 BOM 统一版本（推荐）

结论先行：用 BOM（Bill of Materials，物料清单）统一管理 Spring AI 相关依赖版本，避免逐个指定导致冲突。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> 注意：1.0 GA 的正式版已发布到 Maven Central，**不再需要** 0.x 时代的 `spring-milestones` 仓库。若使用里程碑版本才需要额外配置仓库。

### 2.1.1 为什么需要 BOM

```text
无 BOM：每个依赖手写版本 ──▶ 版本漂移风险高
有 BOM：只写 groupId/artifactId ──▶ 版本由 BOM 统一裁决
```

### 2.1.2 父 POM 的继承方式

若项目已继承 `spring-boot-starter-parent`，则只需追加 `spring-ai-bom`，两者互不冲突。

## 2.2 引入 OpenAI starter

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- 对话模型 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>

    <!-- 可选：向量库（后面 RAG 会用到） -->
    <!--
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
    </dependency>
    -->
</dependencies>
```

### 2.2.1 starter 命名规律

| 前缀 | 用途 | 示例 |
|------|------|------|
| `spring-ai-starter-model-*` | 模型接入 | `-openai`、`-ollama` |
| `spring-ai-starter-vector-store-*` | 向量库 | `-pgvector` |
| `spring-ai-starter-mcp-*` | MCP 集成 | `-client` |

### 2.2.2 依赖树排查

```bash
mvn dependency:tree -Dincludes=org.springframework.ai
```

边界与坑：若同时引入多个 `*-model-*` starter，可能得到多个 `ChatModel` Bean，导致注入歧义。

## 2.3 Gradle 写法

```groovy
dependencies {
    implementation platform("org.springframework.ai:spring-ai-bom:1.0.0")
    implementation "org.springframework.ai:spring-ai-starter-model-openai"
    implementation "org.springframework.boot:spring-boot-starter-web"
}
```

生产建议：Gradle 项目同样使用 `platform(...)` 引入 BOM，保证与 Maven 行为一致。

# 3. 配置模型接入

## 3.1 OpenAI（或兼容网关）

结论先行：配置的核心是"密钥 + base-url + 模型名"三要素。

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com     # 可指向自建网关 / 兼容服务
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
      embedding:
        options:
          model: text-embedding-3-small
```

> **密钥必须来自环境变量或配置中心**，禁止硬编码进代码或提交到 Git。

### 3.1.1 常用参数说明

| 参数 | 含义 | 建议 |
|------|------|------|
| `temperature` | 随机性 | 事实问答取 0–0.3 |
| `max-tokens` | 最大输出长度 | 按成本预算设定 |
| `top-p` | 采样范围 | 一般与 temperature 二选一调 |
| `timeout` | 超时 | 生产必须显式配置 |

### 3.1.2 通过网关统一接入

```yaml
spring:
  ai:
    openai:
      api-key: ${GATEWAY_KEY}
      base-url: https://ai-gateway.internal/v1
```

生产建议：把多厂商差异收敛到网关，应用侧只认 OpenAI 兼容协议。

## 3.2 使用国产模型（OpenAI 兼容协议）

绝大多数国产模型提供 OpenAI 兼容接口，只需改 `base-url` 与 `model`：

```yaml
spring:
  ai:
    openai:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode
      chat:
        options:
          model: qwen-plus
```

### 3.2.1 常见兼容端点

| 厂商 | base-url 片段 | 示例模型 |
|------|---------------|----------|
| 阿里云百炼 | `dashscope.aliyuncs.com/compatible-mode` | `qwen-plus` |
| 深度求索 | `api.deepseek.com` | `deepseek-chat` |
| 智谱 | `open.bigmodel.cn/api/paas/v4` | `glm-4` |

### 3.2.2 兼容性边界

边界与坑：兼容协议通常只覆盖 chat/completions，工具调用、结构化输出、嵌入等能力未必完全一致，需逐项验证。

## 3.3 本地模型（Ollama）

结论先行：本地模型适合开发调试与数据不出内网的场景，但吞吐与效果通常弱于云端。

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: qwen2.5:7b
```

```bash
# 先拉取模型
ollama pull qwen2.5:7b
```

### 3.3.1 本地部署建议

| 模型规模 | 显存建议 | 场景 |
|----------|----------|------|
| 7B | 8GB+ | 开发调试 |
| 14B | 16GB+ | 轻量生产 |
| 32B+ | 24GB+ | 高质量离线 |

### 3.3.2 开发与生产切换

```yaml
# application-dev.yaml
spring:
  ai:
    ollama:
      chat:
        options:
          model: qwen2.5:7b
```

生产建议：用 Spring Profile 切换模型供应商，业务代码保持零改动。

## 3.4 多模型共存

边界与坑：同一应用同时接入多个厂商时，多个 `ChatModel` Bean 会让按类型注入失败。

```java
@Configuration
class MultiModelConfig {

    @Bean
    ChatClient openAiClient(@Qualifier("openAiChatModel") ChatModel model) {
        return ChatClient.builder(model).build();
    }
}
```

生产建议：用 `@Qualifier` 或自定义 `@Bean` 显式区分，避免依赖注入歧义。

# 4. 第一个程序

## 4.1 启动类

```java
@SpringBootApplication
public class AiDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(AiDemoApplication.class, args);
    }
}
```

## 4.2 注入 ChatClient 并调用

### 4.2.1 同步调用

```java
@RestController
@RequestMapping("/ai")
public class ChatController {

    private final ChatClient chatClient;

    // ChatClient.Builder 由 Spring AI 自动配置提供
    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder
                .defaultSystem("你是一个简洁、专业的中文助手。")
                .build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
                .user(message)
                .call()
                .content();
    }
}
```

### 4.2.2 流式调用

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .stream()
            .content();
}
```

生产建议：面向浏览器优先用 SSE（Server-Sent Events，服务器发送事件），并在网关侧关闭响应缓冲。

## 4.3 测试

```bash
curl "http://localhost:8080/ai/chat?message=你好，介绍一下你自己"
```

```bash
curl -N "http://localhost:8080/ai/stream?message=讲个笑话"
```

### 4.3.1 单元测试

```java
@SpringBootTest
class ChatServiceTest {

    @Autowired
    ChatClient chatClient;

    @Test
    void shouldReturnNonEmpty() {
        String answer = chatClient.prompt().user("你好").call().content();
        assertThat(answer).isNotBlank();
    }
}
```

边界与坑：真实调用会产生费用且不稳定，CI 中应使用 Mock 或禁用该类测试。

## 4.4 配置化改造

```yaml
app:
  ai:
    system-prompt: 你是一个简洁、专业的中文助手。
```

```java
@ConfigurationProperties(prefix = "app.ai")
public record AiProperties(String systemPrompt) {
}
```

生产建议：把系统提示词（System Prompt）外置为配置，便于按环境与业务线调整，无需重新打包。

# 5. 项目结构建议

## 5.1 目录结构

```text
src/main/java/com/example/ai
├── AiDemoApplication.java
├── config
│   └── AiConfig.java          # 自定义 ChatClient、VectorStore 等 Bean
├── controller
│   └── ChatController.java    # 对外接口
├── service
│   └── ChatService.java       # 业务逻辑，隔离框架细节
└── advisor
    └── LoggingAdvisor.java    # 自定义拦截器（后续章节）
```

> 生产项目中，**不要让 Controller 直接依赖 ChatClient**，应通过 Service 层封装，方便替换框架与编写测试。

## 5.2 分层职责

| 层 | 职责 | 依赖 |
|----|------|------|
| Controller | 参数校验、协议转换 | Service |
| Service | 业务编排、提示组装 | ChatClient |
| Config | Bean 装配、参数绑定 | Spring AI |

## 5.3 配置类示例

```java
@Configuration
public class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, AiProperties props) {
        return builder
                .defaultSystem(props.systemPrompt())
                .build();
    }
}
```

生产建议：配置类集中管理所有 Spring AI 相关 Bean，便于审阅与替换。

# 6. 常见启动问题

## 6.1 启动期问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `No qualifying bean of type ChatModel` | 未引入对应 starter 或未配置 api-key | 检查依赖与 `spring.ai.*.api-key` |
| artifact 找不到 | 用了 0.x 的旧坐标 | 改用 `spring-ai-starter-model-*` |
| 启动报 Spring Boot 版本冲突 | Spring AI 2.0 需 Boot 4.x | 对齐版本或降级 Spring AI |

## 6.2 运行期问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `401 Unauthorized` | 密钥错误/过期 | 校验环境变量与网关配置 |
| `429 Too Many Requests` | 触发限流 | 加退避重试与限流 |
| `Connection timed out` | 网络/代理问题 | 配置 `base-url` 或代理 |
| 响应截断 | `max-tokens` 过小 | 调大输出上限 |

## 6.3 排查方法

```text
1. 启动日志搜索 "Spring AI" / "AutoConfiguration"
2. 确认 Bean 是否存在：/actuator/beans
3. 打开 DEBUG 日志看请求体（注意脱敏）
```

生产建议：把常见错误映射为业务可读提示，避免把 500 直接抛给前端。

# 7. 生产化准备

## 7.1 密钥与配置管理

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
```

```text
优先级：环境变量 > 配置中心 > 本地配置
```

边界与坑：日志框架可能打印配置属性，务必确认密钥不会被输出。

## 7.2 超时与重试

| 参数 | 建议值 | 说明 |
|------|--------|------|
| 连接超时 | 5s | 网络抖动容忍 |
| 读取超时 | 60s | 长回答需放宽 |
| 重试次数 | 2–3 | 仅重试幂等调用 |

## 7.3 可观测性

```java
@Bean
ObservationRegistryCustomizer observationCustomizer() {
    return registry -> registry.observationConfig()
            .observationHandler(new DefaultMeterObservationHandler(meterRegistry));
}
```

生产建议：至少记录调用次数、耗时、Token 用量三个指标，并按模型与租户打标签。

# 8. 小结

## 8.1 核心要点

- 三步走：**引依赖 → 配 api-key → 注入 ChatClient**。
- 统一用 BOM 管理版本，避免依赖冲突。
- 密钥外置、Service 层隔离框架，是生产化的第一课。

## 8.2 下一步

- 多模型共存需显式区分 Bean，避免注入歧义，再进入核心概念学习。
