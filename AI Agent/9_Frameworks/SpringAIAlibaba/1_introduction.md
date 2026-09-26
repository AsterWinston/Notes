# 1. 定位

结论先行：**Spring AI Alibaba 是阿里基于 Spring AI 的增强实现，面向国内生态**，用 Spring 的编程模型接入通义、百炼等模型，并补齐图编排与可观测能力。

## 1.1 什么是 Spring AI Alibaba

它不是一个全新的框架，而是站在 Spring AI（官方抽象层）之上的扩展层，复用 Spring AI 的核心抽象，再叠加国内模型适配与企业级能力。理解这一定位，就能理解它为什么能"平滑引入"已有工程。

### 1.1.1 官方定位

```
Spring AI（官方抽象）
      +
Spring AI Alibaba（国内增强）
  ├── 国内模型深度适配（通义、百炼）
  ├── Graph 多智能体编排
  ├── 可观测性增强
  └── 企业级实践
```

### 1.1.2 增强点概览

| 增强点 | 说明 |
|--------|------|
| 模型适配 | 通义千问、百炼平台原生支持 |
| 图编排 | Graph 多智能体工作流 |
| 可观测 | 与阿里云可观测平台集成 |
| 工程实践 | 面向国内部署与合规场景 |

## 1.2 设计目标

- 让 Spring 开发者用**熟悉的注解与配置**接入大模型，降低学习成本；
- 把国内模型平台的差异（鉴权方式、参数命名、流式协议）封装在 Starter 中；
- 提供多 Agent 编排能力，支撑需要多步骤与条件分支的复杂业务；
- 复用 Spring 生态的依赖注入、配置管理与可观测设施。

这些目标决定了它的使用方式：**配置驱动 + 面向接口**，业务代码尽量不感知具体模型厂商。

## 1.3 边界与坑

- 它是 Spring AI 的**扩展而非替代**，核心概念（`ChatClient`、`Advisor`、`ChatMemory`）仍来自 Spring AI；
- 版本迭代较快，部分 API 可能标注为预览（preview），稳定性需评估；
- 能力重心在阿里云生态，跨云或多云场景需谨慎评估。

## 1.4 生产建议

- 先掌握 Spring AI 基础，再使用增强能力；
- 锁定版本，避免预览 API 突然变更；
- 模型层保留抽象，避免深度绑定单一平台。

# 2. 与 Spring AI 的关系

结论先行：**基础复用、增强叠加、互补而非替代**。已有 Spring AI 项目可平滑引入。

## 2.1 关系矩阵

| 关系 | 说明 |
|------|------|
| 基础 | 构建在 Spring AI 之上 |
| 兼容 | 复用 Spring AI 抽象 |
| 增强 | 补充国内模型、Graph、观测 |
| 互补 | 不是替代，是扩展 |

## 2.2 依赖继承与兼容

### 2.2.1 版本对齐

Spring AI Alibaba 的版本与 Spring AI、Spring Boot 存在对应关系，引入时需保证三者兼容，否则会出现 `NoSuchMethodError` 或自动配置失效。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2.2.2 平滑引入

在既有 Spring AI 工程中加入 Alibaba Starter，原有 `ChatClient` 调用保持不变，新能力按需启用，迁移成本低。建议先在一个独立模块验证，再逐步替换。

## 2.3 边界与坑

- Spring AI 升级时，Alibaba 版本可能滞后，需等待适配；
- 两者同时提供同类型 Bean 时，注意自动配置优先级与 `@ConditionalOnMissingBean`；
- 混用官方 Starter 与 Alibaba Starter 需核对依赖树。

## 2.4 生产建议

- 用 BOM 统一 Spring AI 与 Alibaba 版本；
- CI 检查依赖冲突；
- 关键能力做冒烟测试覆盖。

# 3. 核心能力

结论先行：**模型接入、Graph 编排、RAG、可观测、工具生态**构成五大能力面。

## 3.1 能力总览

| 能力 | 说明 |
|------|------|
| 模型接入 | 通义千问、百炼平台 |
| Graph 编排 | 多 Agent 工作流 |
| RAG | 国内场景优化 |
| 可观测 | 与 ARMS/可观测平台集成 |
| 工具 | 国内生态工具 |

## 3.2 模型接入

### 3.2.1 通义千问

通过 DashScope（灵积）协议接入 `qwen` 系列模型，支持对话、流式与嵌入，配置集中在 `spring.ai.dashscope` 前缀下。

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
          temperature: 0.7
      embedding:
        options:
          model: text-embedding-v3
```

### 3.2.2 百炼平台

百炼（Bailian）是阿里云大模型服务平台，提供应用编排、知识库与模型托管，Spring AI Alibaba 可直接对接其应用与检索能力，适合"平台已沉淀知识库"的企业。

### 3.2.3 多模型与参数

同一应用可注入多个 `ChatClient`，按用途选择模型与参数（温度、最大输出长度），实现快慢分层。

```java
@Bean
ChatClient fastClient(ChatClient.Builder builder) {
    return builder
            .defaultOptions(DashScopeChatOptions.builder()
                    .withModel("qwen-turbo")
                    .withTemperature(0.3)
                    .build())
            .build();
}
```

## 3.3 Graph 编排

Graph 用"节点 + 边 + 状态"描述多步骤、条件分支、并行与循环流程，适合多 Agent 协作。

```java
// 伪代码：图编排示意
StateGraph graph = new StateGraph();
graph.addNode("classify", this::classifyNode);
graph.addNode("retrieve", this::retrieveNode);
graph.addNode("generate", this::generateNode);
graph.addConditionalEdges("classify", this::routeByIntent,
        Map.of("NEED_RAG", "retrieve", "DIRECT", "generate"));
graph.addEdge("retrieve", "generate");
graph.addEdge("generate", StateGraph.END);
```

细节见 `4_graph.md`。

## 3.4 RAG 与工具

RAG（Retrieval-Augmented Generation，检索增强生成）侧针对国内向量库与文档格式优化；工具侧可把 Spring Bean 暴露为可调用工具，供模型在对话中按需触发。

| 能力 | 说明 |
|------|------|
| 文档处理 | 读取、切分、嵌入 |
| 向量检索 | 对接国内向量库 |
| 工具调用 | Spring Bean 暴露为工具 |

## 3.5 可观测

与阿里云 ARMS、链路追踪集成，可采集模型调用次数、延迟、Token 用量与错误率。

| 指标 | 含义 |
|------|------|
| 调用次数 | 按模型/应用 |
| 延迟 | P50/P95/P99 |
| Token 用量 | 输入/输出 |
| 错误率 | 超时、限流、5xx |

## 3.6 边界与坑

- Graph 能力与 Spring AI 原生能力边界需厘清，避免重复编排；
- 部分工具/向量库适配仍依赖社区贡献，成熟度不一；
- 可观测能力在非阿里云环境可能受限。

## 3.7 生产建议

- 先打通单模型调用，再引入 Graph；
- 把工具与向量库做成可替换 Bean；
- 指标接入统一监控，避免厂商锁定。

# 4. 依赖引入与工程结构

结论先行：**一个 Starter 即可起步**，版本以官方最新为准。

## 4.1 Maven 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0.2</version>
</dependency>
```

若需要 Graph 等独立能力，可再引入对应子模块，版本与主 Starter 保持一致。

## 4.2 工程结构

```
src/main/java
  └── com.example
      ├── config      配置类（模型、向量库）
      ├── controller  接入层
      ├── service     业务编排
      └── Application 启动类
src/main/resources
  └── application.yml 模型与平台配置
```

分层原则与 LangChain4j 一致：接入层只做协议转换，业务编排下沉到 Service。

## 4.3 版本管理

| 组件 | 说明 |
|------|------|
| Spring Boot | 3.x，要求 JDK 17+ |
| Spring AI | 与 Alibaba 版本对齐 |
| spring-ai-alibaba-starter | 核心增强包 |

## 4.4 边界与坑

- Starter 版本号形如 `1.0.0.x`，与 Spring AI 版本并非一一对应；
- JDK 版本不满足会导致启动失败；
- 快照（SNAPSHOT）版本不建议用于生产。

## 4.5 生产建议

- 用 `dependencyManagement` 固化版本；
- 固定 JDK 与 Spring Boot 大版本；
- 升级前阅读官方迁移说明。

# 5. 快速上手

结论先行：**注入 `ChatClient.Builder`，一次 `.call().content()` 即可完成对话**。

## 5.1 最小示例

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt().user(message).call().content();
    }
}
```

## 5.2 配置说明

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
```

`api-key` 从环境变量注入，`model` 指定使用的通义模型。

## 5.3 流式与多轮

### 5.3.1 流式输出

把 `.call()` 换成 `.stream()`，返回 `Flux<String>`，配合 SSE 即可实现打字机效果。

```java
@GetMapping(value = "/chat/stream",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String message) {
    return chatClient.prompt().user(message).stream().content();
}
```

### 5.3.2 对话记忆

通过 `ChatMemory` 或 `MessageChatMemoryAdvisor`（消息记忆顾问）把历史消息拼入提示，实现多轮对话。

```java
ChatClient client = builder
        .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
        .build();
```

### 5.3.3 工具调用

把普通 Spring Bean 的方法注册为工具后，模型可在对话中自动触发调用，再把结果并入回答。

```java
@Bean
ChatClient toolClient(ChatClient.Builder builder, OrderTools tools) {
    return builder.defaultTools(tools).build();
}
```

## 5.4 边界与坑

- `ChatClient` 是无状态的，多轮需显式挂载记忆；
- 直接返回模型原文，缺少敏感信息与注入防护；
- 同步调用在模型慢时占用 Web 线程。

## 5.5 生产建议

- 流式接口设置超时与取消；
- 请求做长度限制与内容审核；
- 记忆持久化到 Redis 或数据库。

# 6. 与 LangChain4j 的对比

结论先行：**Spring 技术栈 + 国内模型选 Spring AI Alibaba，通用与多框架场景选 LangChain4j**。

## 6.1 对比表

| 维度 | Spring AI Alibaba | LangChain4j |
|------|-------------------|-------------|
| 基础 | Spring AI | 独立 |
| 国内模型 | 原生深度适配 | 需自行配置 |
| 图编排 | Graph 内置 | LangGraph4j |
| Spring 集成 | 原生 | Starter |
| 适用 | Spring + 国内模型 | 通用 |

## 6.2 选型建议

- 已有 Spring Boot 体系、主要用通义/百炼：优先 Spring AI Alibaba；
- 需要跨语言、跨框架或大量第三方集成：优先 LangChain4j；
- 两者可共存于同一工程，但应统一模型抽象层，避免重复建设。

## 6.3 边界与坑

- 两者概念相似但 API 不同，团队需统一约定；
- 同时引入可能导致 Bean 命名冲突；
- 迁移成本主要来自提示与工具代码，而非模型调用本身。

## 6.4 共存实践

若同一工程需要两者，建议在适配层收口：定义统一的 `LlmPort` 接口，由两套实现分别委托给 `ChatClient` 与 LangChain4j 的 `ChatModel`，业务只依赖 `LlmPort`。

```java
public interface LlmPort {
    String chat(String message);
}
```

这样切换或替换框架时，业务代码零改动。

# 7. 适用场景

结论先行：**国内企业 Spring 应用、需要图编排与可观测的场景最契合**。

## 7.1 场景清单

| 场景 | 说明 |
|------|------|
| 国内企业 Spring 应用 | 首选 |
| 使用通义/百炼 | 深度适配 |
| 需要图编排 | Graph 内置 |
| 需要可观测 | 阿里云集成 |

## 7.2 反例（不适合的场景）

- 主要使用海外模型且不涉国内生态：直接用 Spring AI 更轻；
- 追求极简依赖的 Serverless 函数：Starter 体积可能偏大；
- 非 JVM 技术栈：应选择对应语言生态的框架。

## 7.3 生产建议

- 用场景驱动选型，而非框架热度；
- 对关键场景做 PoC 后再规模化；
- 保留退出路径，避免锁定。

# 8. 学习路径

结论先行：**先基础、再模型、后编排、最后工程化**。

## 8.1 四步路径

```
① 掌握 Spring AI 基础
② 接入通义/百炼模型
③ 学习 Graph 编排
④ 企业级实践（可观测、RAG）
```

## 8.2 各阶段产出

| 阶段 | 产出 |
|------|------|
| ① 基础 | 理解 `ChatClient` / `Advisor` / `ChatMemory` |
| ② 模型 | 可运行的对话与流式接口 |
| ③ 编排 | 多节点工作流与条件分支 |
| ④ 工程化 | 可观测、RAG、多租户 |

## 8.3 生产建议

- 每阶段配套小练习与评测样例；
- 编排阶段先画流程图再写代码；
- 工程化阶段补齐超时、重试与限流。

# 9. 生产注意事项

结论先行：**版本、兼容、绑定、文档**是四个主要风险点。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 版本迭代快 | 锁定版本 |
| 与 Spring AI 版本兼容 | 对齐依赖 |
| 绑定阿里云 | 保持模型层抽象 |
| 文档以中文为主 | 关注官方更新 |
| 密钥管理 | 环境变量/配置中心 |
| 无超时重试 | 显式配置并做降级 |

## 9.2 上线检查表

- Spring AI 与 Alibaba 版本已对齐；
- 密钥来自环境变量或配置中心；
- 模型调用配置超时与重试；
- 关键链路有指标与追踪；
- 提示与输出有安全校验；
- 多轮记忆已持久化并带 TTL。

## 9.3 安全与成本

| 维度 | 措施 |
|------|------|
| 输入安全 | 长度限制、注入检测、敏感词 |
| 输出安全 | 内容审核、脱敏 |
| 成本控制 | 模型分层、`maxTokens`、配额 |
| 可观测 | Token 用量与调用次数统计 |

# 10. 小结

- Spring AI Alibaba 是 Spring AI 的国内增强实现。
- 核心能力：国内模型、Graph 编排、可观测。
- 已有 Spring AI 项目可平滑引入。
- 适合国内企业 Spring 应用。
- 与 LangChain4j 互补，不是替代。
