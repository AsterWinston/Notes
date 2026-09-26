# 1. Spring AI 是什么

Spring AI 是 Spring 官方推出的 AI 应用开发框架（AI Application Framework），目标是把「接入大模型、构建 AI 应用」这件事，变成和用 Spring 写 Web 应用一样自然。

## 1.1 定义与核心价值

结论先行：Spring AI 的价值不在于"能调用大模型"——任何 HTTP 客户端都能做到；它的价值在于提供一套**可移植的抽象**（Portable Abstraction），让 AI 能力像数据库访问一样，成为 Spring 应用中的标准组件。

机制拆解：

1. 定义稳定的抽象接口，如 `ChatModel`、`EmbeddingModel`、`VectorStore`。
2. 各厂商提供适配实现（Adapter），如 OpenAI、Ollama、Anthropic、DashScope。
3. Spring Boot 自动配置（Auto-Configuration）按 classpath 与配置项装配 Bean。
4. `@ConditionalOnMissingBean` 允许用户用自定义 Bean 覆盖框架默认实现。

一句话总结：

> **Spring AI = Spring 风格的 LLM 应用开发框架，核心价值是"可移植的抽象 + 无缝融入 Spring 生态"。**

## 1.2 架构总览

```text
Spring 生态（Boot / DI / AOP / Actuator / Security / Batch）
        │
        ▼
Spring AI（统一抽象层）
        │
        ├── ChatModel        对话模型
        ├── EmbeddingModel   向量模型
        ├── ImageModel       图像模型
        ├── AudioModel       语音模型
        ├── VectorStore      向量存储
        ├── ToolCallback     工具调用
        ├── ChatMemory       对话记忆
        └── Advisor          拦截器链
        │
        ▼
底层模型/服务：OpenAI / Anthropic / Ollama / DashScope / Azure / Bedrock ...
```

上图自上而下是"复用生态 → 统一抽象 → 适配厂商"三层关系：越往上越贴近业务，越往下越贴近具体服务。中间层就是 Spring AI 的立足点。

### 1.2.1 分层职责

| 层次 | 代表类型 | 职责 |
|------|----------|------|
| 应用层 | `ChatClient` | 面向业务的流式门面（Facade） |
| 编排层 | `Advisor` / `ChatMemory` | 横切逻辑：记忆、RAG、日志 |
| 模型层 | `ChatModel` / `EmbeddingModel` | 稳定的统一抽象 |
| 协议层 | `OpenAiChatModel` 等 | 厂商协议适配 |

### 1.2.2 数据流示意

```text
用户输入 ──▶ ChatClient ──▶ Advisor 链 ──▶ ChatModel ──▶ HTTP ──▶ 模型服务
                                                    ◀── 响应 ◀──
```

## 1.3 与直接使用厂商 SDK 的对比

### 1.3.1 代码形态差异

结论先行：直接使用厂商 SDK，业务代码与厂商强耦合；使用 Spring AI，业务代码只依赖抽象接口。

```java
// 厂商 SDK：与 OpenAI 强耦合，换模型需重写
OpenAiApi api = new OpenAiApi(System.getenv("OPENAI_API_KEY"));
ChatCompletionResult result = api.createChatCompletion(
        new ChatCompletionRequest("gpt-4o-mini", List.of(
                new ChatMessage("user", "你好"))));
String text = result.getChoices().get(0).getMessage().getContent();
```

```java
// Spring AI：面向抽象编程，配置切换模型
ChatModel chatModel = ...;      // 可以是 OpenAI、Ollama、Anthropic...
String answer = chatModel.call(new Prompt("你好"))
        .getResult().getOutput().getText();
```

### 1.3.2 可移植性的边界

边界与坑：抽象只覆盖**公共子集**，厂商特有能力（如某些厂商的缓存命中字段、扩展参数）需要借助具体 `*ChatOptions` 或原生 API 才能访问。

| 能力 | 是否可移植 | 说明 |
|------|-----------|------|
| 基础对话 | 是 | 所有 `ChatModel` 实现都支持 |
| 流式输出 | 是 | `StreamingChatModel` 统一返回 `Flux` |
| 工具调用 | 是（1.0+） | 统一为 `ToolCallback` 抽象 |
| 结构化输出 | 是 | `BeanOutputConverter` 统一处理 |
| 厂商特有参数 | 否 | 需使用具体 `*ChatOptions` |
| 私有鉴权/签名 | 否 | 需自定义实现或经网关 |

生产建议：把"厂商特有"的部分收敛到配置层或网关层，避免污染业务代码。

## 1.4 在 Spring 生态中的位置

结论先行：Spring AI 不是孤立框架，而是 Spring 全家桶中"AI 能力"这一格。

| Spring 项目 | 对应能力 |
|-------------|----------|
| Spring Data | 数据访问抽象 |
| Spring Integration | 消息集成抽象 |
| Spring Batch | 批处理 |
| Spring AI | 大模型与 AI 能力抽象 |

### 1.4.1 复用生态的具体方式

```java
// 借助 Spring Retry 实现调用重试（示意）
@Retryable(retryFor = ResourceAccessException.class, maxAttempts = 3)
public String ask(String question) {
    return chatClient.prompt().user(question).call().content();
}
```

### 1.4.2 与可观测性的结合

| Spring 能力 | 在 AI 场景的用途 |
|-------------|------------------|
| Micrometer | 记录 Token 用量、调用耗时 |
| Observation | 生成 Span，串联调用链 |
| Actuator | 暴露健康检查与指标端点 |

生产建议：优先复用 Actuator、Security、Retry 等既有能力，而不是在 AI 层重复实现。

# 2. 它解决什么问题

## 2.1 痛点清单

结论先行：Spring AI 主要解决"接口不统一"和"与业务系统割裂"这两类问题。

| 痛点 | Spring AI 的方案 |
|------|------------------|
| 每家模型 SDK 接口都不一样 | 统一的 `ChatModel` / `EmbeddingModel` 抽象 |
| 换模型要改大量代码 | 面向接口编程，配置切换 |
| 提示、记忆、RAG、工具各写一套 | 提供 Prompt / ChatMemory / VectorStore / ToolCallback 标准组件 |
| 与业务系统（事务、安全、监控）割裂 | 原生融入 Spring Boot 自动配置与生态 |
| 可观测性缺失 | 内置 Observation / Micrometer 指标 |

机制拆解：Spring AI 的做法是把"每次模型调用"抽象成一次可拦截、可观测、可替换的方法调用，从而把散落的胶水代码收敛到框架内部。

```text
传统写法：业务代码 ──▶ HTTP ──▶ 模型（重试/日志/指标各自为政）
Spring AI：业务代码 ──▶ ChatClient ──▶ Advisor ──▶ ChatModel（统一治理）
```

## 2.2 抽象带来的收益与代价

### 2.2.1 收益

- 一次编码，多模型运行：开发用 Ollama，生产用云端模型。
- 组件复用：记忆、RAG、工具是标准件，不必重复造轮子。
- 治理能力：超时、重试、限流、指标可借助 Spring 生态统一处理。

### 2.2.2 代价

- 学习成本：需要理解 `ChatModel` / `ChatClient` / `Advisor` 等概念。
- 抽象泄漏：厂商新特性通常先于抽象层支持，存在时间差。
- 版本演进快：0.x 到 1.0 曾有大量破坏性变更，升级需评估。

### 2.2.3 量化权衡

| 维度 | 自研封装 | Spring AI |
|------|----------|-----------|
| 初期开发速度 | 快 | 更快（组件现成） |
| 长期维护 | 高 | 低（官方维护） |
| 灵活性 | 完全可控 | 受抽象约束 |
| 团队学习 | 无 | 有曲线 |

## 2.3 不解决的问题

边界与坑：Spring AI 不负责"模型质量"。以下问题仍需自行处理：

| 问题 | 归属 |
|------|------|
| 模型幻觉（Hallucination） | 提示工程 + RAG + 评测 |
| 数据安全与合规 | 业务与安全团队 |
| Token 成本控制 | 业务策略 + 缓存 + 路由 |
| 向量库运维 | 基础设施团队 |

生产建议：把 Spring AI 当作"连接层"，而非"智能来源"。

# 3. 设计理念

## 3.1 可移植性（Portability）

Spring AI 借鉴了 Spring Data、Spring Integration 的设计思路：**定义稳定的抽象接口，由各厂商适配实现**。

```java
// 面向抽象编程，而不是面向 OpenAI 编程
ChatModel chatModel = ...;      // 可以是 OpenAI、Ollama、Anthropic...
String answer = chatModel.call(new Prompt("你好")).getResult().getOutput().getText();
```

### 3.1.1 可移植性如何实现

```text
业务代码 ──依赖──▶ ChatModel（接口）
                        ▲
        ┌───────────────┼───────────────┐
   OpenAiChatModel  OllamaChatModel  AnthropicChatModel
```

### 3.1.2 切换模型的配置成本

```yaml
# 只需切换依赖与配置，业务代码零改动
spring:
  ai:
    ollama:
      chat:
        options:
          model: qwen2.5:7b
```

### 3.1.3 抽象泄漏的应对

边界与坑：当必须使用厂商特有能力时，建议在 Service 层用 `instanceof` 隔离，而不是让上层感知具体类型。

```java
if (chatModel instanceof OpenAiChatModel openAi) {
    // 仅在此处触碰厂商特有 API
}
```

## 3.2 约定优于配置（Convention over Configuration）

引入 starter + 配置 `api-key`，Spring Boot 自动装配 `ChatModel`、`ChatClient.Builder` 等 Bean，直接注入使用。

```java
@RestController
class DemoController {
    private final ChatClient chatClient;

    DemoController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
}
```

### 3.2.1 默认约定一览

| 约定 | 默认值/前缀 |
|------|-------------|
| 属性前缀 | `spring.ai.<provider>.*` |
| 开关属性 | `spring.ai.<provider>.chat.enabled` |
| 默认模型 | 由各厂商 starter 决定 |
| 自动装配类 | `<Provider>AutoConfiguration` |

### 3.2.2 覆盖默认 Bean

边界与坑：自动配置依赖 classpath 与属性前缀约定，一旦自定义 Bean，框架会退让，需自行保证线程安全与生命周期。

```java
@Bean
ChatModel chatModel(ChatModel delegate) {
    return new MyResilientChatModel(delegate);
}
```

## 3.3 组件化与可组合

Prompt、Advisor、Memory、VectorStore、Tool 都是独立组件，可自由组合，符合 Spring 的"组合优于继承"风格。

```text
ChatClient
   ├── defaultSystem(...)      提示
   ├── defaultAdvisors(...)    拦截器
   ├── defaultTools(...)       工具
   └── defaultFunctions(...)   （旧名，1.0 起统一为 tools）
```

### 3.3.1 组合示例

```java
ChatClient client = builder
        .defaultSystem("你是企业知识助手")
        .defaultAdvisors(new SimpleLoggerAdvisor())
        .defaultTools(new WeatherTools())
        .build();
```

### 3.3.2 组件的生命周期

| 组件 | 作用域 | 建议 |
|------|--------|------|
| `ChatModel` | 单例 | 无状态，可共享 |
| `ChatClient` | 单例 | 线程安全，可共享 |
| `ChatMemory` | 会话级 | 按 `conversationId` 隔离 |
| `Advisor` | 单例/会话级 | 无状态用单例 |

生产建议：无状态组件用单例，会话相关状态显式放到 `ChatMemory`，避免在 Bean 中保存用户数据。

## 3.4 面向接口与覆盖机制

生产建议：优先通过"自定义 Bean"扩展，而非 fork 框架源码。覆盖点包括 `ChatModel`、`ChatMemory`、`VectorStore`、`Advisor`。

| 覆盖点 | 典型用途 |
|--------|----------|
| `ChatModel` | 加熔断、限流、缓存 |
| `ChatMemory` | 接 Redis / 数据库 |
| `VectorStore` | 接自研向量库 |
| `Advisor` | 统一日志、审计、脱敏 |

# 4. 核心能力地图

## 4.1 能力总览表

| 能力 | 核心类型 | 对应笔记 |
|------|----------|----------|
| 对话调用 | `ChatModel` / `ChatClient` | 本目录 3、4 |
| 提示模板 | `Prompt` / `PromptTemplate` | 5 |
| 结构化输出 | `BeanOutputConverter` | 6 |
| 工具调用 | `@Tool` / `ToolCallback` | 7 |
| 对话记忆 | `ChatMemory` | 8 |
| 拦截增强 | `Advisor` | 9 |
| RAG | `VectorStore` + `QuestionAnswerAdvisor` | 10、11、12 |
| MCP | `spring-ai-mcp` | 13 |
| 多模态 | `Media` / `ImageModel` / `AudioModel` | 14 |
| 可观测性 | Observation + Micrometer | 15 |
| 评测 | `Evaluator` | 16 |

## 4.2 各能力的依赖关系

```text
ChatModel ──▶ ChatClient ──▶ Advisor
                  │              ├── ChatMemory
                  │              ├── QuestionAnswerAdvisor ──▶ VectorStore
                  │              └── SafeGuardAdvisor
                  └──▶ ToolCallback
```

结论先行：RAG 与工具调用都建立在对话能力之上，因此必须先吃透 `ChatModel` / `ChatClient`。

```java
// 依赖关系在代码中的体现：ChatClient 包装 ChatModel 并挂载 Advisor
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .defaultTools(new WeatherTools())
        .build();
```

## 4.3 学习路径映射

生产建议：按"调用 → 增强 → 检索 → 智能体"四阶段推进，每个阶段都补上可观测性与评测。

| 阶段 | 组件 | 里程碑 |
|------|------|--------|
| 调用 | ChatModel / ChatClient | 单轮对话 |
| 增强 | Prompt / 结构化输出 / Memory | 多轮 + 结构化 |
| 检索 | Embedding / VectorStore / RAG | 知识库问答 |
| 智能体 | Tool / Advisor / MCP | 自主任务 |

# 5. 版本演进

## 5.1 版本时间线

| 版本 | 时间 | 关键变化 |
|------|------|----------|
| 0.x（里程碑） | 2023–2024 | 快速迭代，包名与 API 频繁变动 |
| 1.0.0 GA | 2025-05 | 首个正式版，API 稳定，artifact 命名规范化 |
| 1.0.x / 1.1.x | 2025–2026 | 持续修复与增强，MCP 集成完善 |
| 2.0.0 GA | 2026-06 | 要求 Spring Boot 4.x，工具调用重构（`ToolCallingAdvisor`）、MCP SDK 2.0 |

> 本笔记以 **1.0.x GA 的 API 为主线**（企业落地最广、文档最全），涉及 2.0 的差异会单独标注。

## 5.2 1.0 之后的重要命名变化

### 5.2.1 坐标变化

1.0 GA 起，starter 的 artifact 命名统一为 `spring-ai-starter-*`：

| 旧（0.x 里程碑） | 新（1.0+） |
|------------------|-----------|
| `spring-ai-openai-spring-boot-starter` | `spring-ai-starter-model-openai` |
| `spring-ai-ollama-spring-boot-starter` | `spring-ai-starter-model-ollama` |
| `spring-ai-pgvector-store-spring-boot-starter` | `spring-ai-starter-vector-store-pgvector` |

### 5.2.2 迁移注意事项

边界与坑：坐标变化之外，部分类名与包路径也调整过，升级时不要只改 `pom.xml`。

| 变化项 | 0.x | 1.0+ |
|--------|-----|------|
| 函数调用 | `FunctionCallback` | `ToolCallback` |
| 函数注册 | `.functions(...)` | `.tools(...)` |
| 记忆 Advisor | `PromptChatMemoryAdvisor` | `MessageChatMemoryAdvisor` |

### 5.2.3 依赖管理建议

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

## 5.3 2.0 的变化

结论先行：2.0 是一次大版本跃迁，核心是跟随 Spring Boot 4.x 与 MCP 生态。

- 工具调用重构为 `ToolCallingAdvisor`，调用链更可控。
- MCP 采用 SDK 2.0。
- 要求 Spring Boot 4.x / JDK 17+（推荐 21）。

生产建议：存量系统不要盲目追新，先评估 Spring Boot 大版本升级成本。

# 6. Spring AI 与 LangChain4j 的关系

## 6.1 定位对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 出身 | Spring 官方 | 社区（后并入 Quarkus 生态） |
| 设计风格 | Spring 式抽象 + 自动配置 | 组件丰富、声明式 AI Services |
| 生态融合 | 与 Spring Boot/Security/Actuator 无缝 | 与 Quarkus/Spring Boot 均可集成 |
| 抽象层次 | 偏底层、统一 | 偏高阶、开箱即用 |
| 适用场景 | 已有 Spring 体系的企业 | 快速原型、组件定制 |

## 6.2 选型建议

结论先行：已有 Spring 技术栈选 Spring AI，追求组件丰富度与声明式体验可评估 LangChain4j。

```text
已有 Spring Boot 微服务？
   ├── 是 ──▶ Spring AI（复用 DI / Actuator / Security）
   └── 否 ──▶ 看团队偏好：Quarkus → LangChain4j
                          纯 Java 库 → 两者皆可
```

### 6.2.1 决策权重

| 权重项 | Spring AI 更优 | LangChain4j 更优 |
|--------|---------------|------------------|
| 与 Spring 集成 | 是 | 一般 |
| 组件数量 | 一般 | 是 |
| 声明式接口 | 一般 | 是 |
| 官方支持 | 是 | 社区为主 |

## 6.3 是否可以混用

边界与坑：二者可在同一 JVM 内共存，但会引入两套 HTTP 客户端、重试与序列化依赖，增加冲突风险。

> 两者并非二选一：本笔记把二者都作为主线，学完原理后按团队技术栈选型即可。对比详见 `9_Frameworks/0_framework_comparison.md`。

生产建议：一个项目内只选一套作为主抽象，避免概念与依赖双重负担。

# 7. 适用场景

## 7.1 典型场景

- **企业知识库问答**：RAG + VectorStore。
- **智能客服 / 对话系统**：ChatClient + ChatMemory + 工具调用。
- **Agent 应用**：工具 + 记忆 + Advisor + MCP。
- **AI 能力中台**：用统一抽象封装多模型，供业务系统调用。
- **已有 Spring 微服务**：以最小改动引入 AI 能力。

```java
// 最小改动：在既有 Service 中注入 ChatClient
@Service
public class SummaryService {
    private final ChatClient chatClient;

    public SummaryService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String summarize(String text) {
        return chatClient.prompt()
                .user("请用三句话总结：" + text)
                .call()
                .content();
    }
}
```

## 7.2 不适合的场景

边界与坑：以下情况用 Spring AI 可能"杀鸡用牛刀"。

| 场景 | 原因 | 替代 |
|------|------|------|
| 只调一次模型、无状态 | 引入框架收益低 | 直接 HTTP / 厂商 SDK |
| 需要极致低延迟推理 | 抽象层有轻微开销 | 直接调用推理服务 |
| 非 JVM 技术栈 | 无法复用 | 对应语言 SDK |

判断标准：如果 AI 调用只占系统的一小部分，且已有成熟的 HTTP 封装，就不必强行引入框架。

## 7.3 生产落地检查清单

```text
[ ] 密钥外置（环境变量 / 配置中心）
[ ] 超时与重试策略已配置
[ ] 限流与成本监控已接入
[ ] 日志脱敏（不打印完整 Prompt）
[ ] 模型降级与熔断方案
[ ] 评测集与回归流程
```

# 8. 学习建议

## 8.1 推荐路径

1. 先跑通一个最小 Chat 示例（见 `2_environment_setup.md`）。
2. 吃透核心抽象（`3_core_concepts.md`），不要一上来就上 RAG。
3. 按「Chat → Prompt → 结构化输出 → 工具 → 记忆 → RAG → MCP」顺序推进。
4. 每学一个组件，都思考它的**生产化风险**（超时、重试、成本、权限）。

## 8.2 每阶段验证方式

| 阶段 | 验证方式 |
|------|----------|
| Chat | 能流式输出中文回答 |
| Prompt | 模板参数化生效 |
| 结构化输出 | 返回对象可反序列化 |
| 工具 | 模型能正确选择工具并回填结果 |
| 记忆 | 多轮对话上下文连续 |
| RAG | 检索命中相关文档且答案有引用 |

## 8.3 常见误区

边界与坑：

- 误区一：以为抽象能抹平所有厂商差异。实际上差异仍然存在，只是被收拢。
- 误区二：一上来就搭 RAG。应先验证基础对话与提示质量。
- 误区三：忽略成本。每次调用都是真金白银，必须可观测。

生产建议：先做"能跑通"，再做"可观测"，最后做"可治理"。

# 9. 小结

## 9.1 核心要点

- Spring AI 是 Spring 风格的 LLM 应用开发框架，核心是**可移植抽象 + 生态融合**。
- 它解决接口不统一与生态割裂问题，但不解决模型质量与数据合规。
- 以 1.0.x GA 为主线，注意 `spring-ai-starter-*` 新命名与 2.0 的 Boot 4.x 要求。

## 9.2 下一步

- 按学习路径推进：ChatModel → ChatClient → Prompt → 结构化输出 → Tool → Memory → Advisor → RAG → MCP。
