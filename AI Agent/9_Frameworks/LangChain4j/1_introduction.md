# 1. LangChain4j 是什么

LangChain4j 是面向 Java 的 **LLM（Large Language Model，大语言模型）应用开发框架**，目标是把"接入大模型、构建 LLM 应用"的能力，用 Java 开发者熟悉的方式提供出来。

## 1.1 一句话定义

> **LangChain4j = Java 生态里组件最全、上手最快的 LLM 应用框架，核心特色是"声明式 AI Services"。**

它同时提供两套编程模型，开发者可以按场景自由切换：

| 编程模型 | 代表 API | 特点 | 适合场景 |
|----------|----------|------|----------|
| 低层命令式 | `ChatModel` / `EmbeddingModel` | 完全可控、贴近协议 | 精细控制、协议调试 |
| 高层声明式 | `AiServices` | 接口即 Agent、自动编排 | 业务开发、快速交付 |

### 1.1.1 它解决什么问题

在没有框架时，构建一个带记忆和知识库的问答服务，需要手写：消息拼装、历史裁剪、向量检索、工具循环、JSON 解析、重试与超时。LangChain4j 把这些重复劳动封装成可组合组件。

### 1.1.2 它不解决什么问题

- 不提供模型本身，仍需调用云端或本地模型；
- 不提供向量数据库，只提供统一访问接口；
- 不做工作流可视化编排，复杂图编排需另寻方案。

## 1.2 架构全景

### 1.2.1 分层示意

```
┌──────────────────────────────────────────────┐
│              LangChain4j 应用层                │
│   AiServices（声明式）  /  低层 API（命令式）    │
├──────────────────────────────────────────────┤
│  能力组件：Prompt / Memory / RAG / Tools / Agent │
├──────────────────────────────────────────────┤
│  模型抽象：ChatModel / EmbeddingModel / ...     │
├──────────────────────────────────────────────┤
│  适配层：OpenAI / Anthropic / Gemini / Ollama   │
│          / Qwen / DeepSeek / Azure / Bedrock ...│
└──────────────────────────────────────────────┘
```

自上而下是"业务语义逐渐剥离、协议细节逐渐增加"的过程：

- 应用层只关心接口方法与注解；
- 组件层把 Memory、RAG、Tools 组装成一次完整对话；
- 模型层统一"消息进、消息出"的契约；
- 适配层负责把统一契约翻译成各家 HTTP/JSON 协议。

### 1.2.2 一次请求的完整链路

以"带记忆 + 带检索 + 带工具"的一次调用为例：

```
用户调用 assistant.chat("帮我订明天的会议室")
        │
        ▼
[AiServices 代理] 读取接口上的注解，组装系统提示
        │
        ▼
[ChatMemory] 取出历史消息，与本次输入合并
        │
        ▼
[ContentRetriever] 用输入做向量检索，拼入参考资料
        │
        ▼
[ChatModel] 发送 ChatRequest 到厂商 HTTP 接口
        │
        ▼
模型返回 toolExecutionRequests（要调工具）
        │
        ▼
[AiServices] 反射执行 @Tool 方法，回填结果再发一次
        │
        ▼
模型返回最终文本 → 映射为方法返回类型
        │
        ▼
[ChatMemory] 把本轮消息写回记忆
```

这条链路是理解 LangChain4j 的关键：**框架的价值就在于把上面这些步骤自动化**。

### 1.2.3 最小可运行示例

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
</dependency>
```

```java
interface Assistant {
    String chat(String userMessage);
}

public class Demo {
    public static void main(String[] args) {
        ChatModel model = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o-mini")
                .build();

        Assistant assistant = AiServices.create(Assistant.class, model);
        System.out.println(assistant.chat("用一句话解释什么是向量"));
    }
}
```

## 1.3 设计目标与取舍

LangChain4j 的定位不是"万能编排引擎"，而是"企业 Java 里够用的 LLM 工具箱"，因此有明显取舍：

| 设计目标 | 具体表现 | 代价 |
|----------|----------|------|
| 上手快 | 零配置即可跑通最小示例 | 抽象不追求极致灵活 |
| 类型安全 | 接口 + 注解 + 泛型 | 动态场景略啰嗦 |
| 模块化 | 按需引入依赖 | 需自行管理版本 |
| 贴近 Java 惯例 | Builder、SPI、Optional | 与 Python 版不一一对应 |

## 1.4 谁适合使用

| 角色 | 使用动机 |
|------|----------|
| Java 后端工程师 | 在既有服务中嵌入 LLM 能力 |
| 架构师 | 需要可替换的模型与向量库抽象 |
| Quarkus 开发者 | 需要原生镜像友好方案 |
| 从 Python 转来的开发者 | 复用已有的 RAG/Tool 概念 |

# 2. 与 LangChain（Python）的关系

## 2.1 定位差异

| 维度 | 说明 |
|------|------|
| 是否移植 | **不是** LangChain 的 Java 移植版，是独立设计 |
| 设计取向 | 更强调类型安全、编译期校验、Java 惯例 |
| 灵感来源 | 借鉴了 LangChain 的组件思想（Memory、Retriever、Chain） |
| 定位差异 | LangChain 生态庞大但抽象偏重；LangChain4j 更克制、更贴近企业 Java |

> 不要用"Java 版 LangChain"来理解它，把它当作"为 Java 设计的 LLM 工具箱"更准确。

## 2.2 借鉴与创新

### 2.2.1 借鉴的组件思想

LangChain4j 沿用了 LangChain 中已经被验证的抽象名词：

- `ChatMemory`：对话历史管理；
- `ContentRetriever` / `EmbeddingStore`：RAG 检索链路；
- `Tool`：函数调用（Function Calling）封装；
- `Chain` 概念被弱化，改为 `AiServices` 的隐式编排。

这些名字降低了从 Python 迁移的认知成本。

### 2.2.2 独立设计的部分

- **接口代理**：`AiServices.create(Interface.class, model)` 通过 JDK 动态代理生成实现，Python 版没有等价物；
- **编译期类型映射**：结构化输出直接映射到 `record` / POJO，而非运行期字典；
- **SPI 可插拔**：模型、向量库、文档解析器都通过独立 artifact 提供。

### 2.2.3 概念对照表

| LangChain（Python） | LangChain4j | 备注 |
|---------------------|-------------|------|
| `LLM` / `ChatOpenAI` | `ChatModel` | 统一接口 |
| `ConversationBufferMemory` | `MessageWindowChatMemory` | 窗口裁剪 |
| `VectorStore` | `EmbeddingStore` | 命名不同 |
| `Retriever` | `ContentRetriever` | 支持多种策略 |
| `Agent` / `Tool` | `@Tool` + `AiServices` | 无需手写 Agent |
| `Chain` | `AiServices` 组合 | 隐式编排 |

## 2.3 迁移心态建议

从 LangChain 转过来时，常见误区是"到处找 Chain"。实际上在 LangChain4j 里：

```java
// 不需要手写 Chain，接口本身就是编排单元
interface Assistant {
    String chat(String message);
}
```

需要显式编排时，优先用 `AiServices.builder()` 组合，而不是拼装低层调用。

# 3. 设计理念

## 3.1 双轨设计：低层 API + 高层声明式

### 3.1.1 低层 API

```java
// 低层：完全可控，适合精细场景
ChatModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .build();
String answer = model.chat("用一句话解释什么是向量");
```

低层 API 的输入输出是 `ChatRequest` / `ChatResponse`，中间不掺任何魔法，便于调试协议与排查线上问题。

### 3.1.2 高层声明式

```java
// 高层：声明式，接口即 Agent
interface Assistant {
    String chat(String message);
}
Assistant assistant = AiServices.create(Assistant.class, model);
String answer2 = assistant.chat("用一句话解释什么是向量");
```

高层 API 把提示组装、结构化输出、工具循环、记忆读写、RAG 全部自动完成。

### 3.1.3 何时用哪一轨

| 需求 | 推荐轨 |
|------|--------|
| 快速验证模型连通性 | 低层 |
| 业务功能开发 | 高层 |
| 需要精确控制每一条消息 | 低层 |
| 需要多轮记忆 + 工具 + RAG | 高层 |
| 排查厂商协议差异 | 低层 |

## 3.2 类型安全

### 3.2.1 编译期校验

用 Java 接口 + 注解描述 LLM 交互，方法签名、参数类型、返回类型都能被编译器检查。拼错注解或参数数量不匹配，在编译期就会暴露。

### 3.2.2 结构化输出映射

```java
record Person(String name, int age) {}

interface Extractor {
    Person extract(String text);
}
```

模型返回的 JSON 会被反序列化为 `Person` 实例，调用方拿到的就是强类型对象，无需手工解析字符串。

### 3.2.3 与运行期反射的边界

类型安全只覆盖方法签名，**模型输出的内容质量无法编译期保证**。因此仍需在业务层校验字段合法性。

## 3.3 模块化

### 3.3.1 按需引入

按需引入依赖：核心、OpenAI、Ollama、向量库、文档解析器、Spring Boot Starter 等互不耦合。不需要 RAG 就不引入向量库，不需要 Spring 就不引入 Starter。

### 3.3.2 依赖体积控制

| 场景 | 建议引入 |
|------|----------|
| 纯对话 | `langchain4j-open-ai` |
| 本地模型 | `langchain4j-ollama` |
| RAG | 上述 + `langchain4j-{store}` + 解析器 |
| Spring 应用 | `langchain4j-spring-boot-starter` |

## 3.4 约定优于配置

大量 `withDefault*` / `builder()` 提供合理默认值，降低上手成本。例如 `OpenAiChatModel` 在只填 `apiKey` 时也能工作，其余参数走默认。

# 4. 模块结构

## 4.1 核心与聚合模块

| 模块 | 作用 |
|------|------|
| `langchain4j-core` | 核心抽象（`ChatModel`、`EmbeddingModel`、`EmbeddingStore`、`ChatMemory`…） |
| `langchain4j` | 聚合模块，含 `AiServices`、RAG 组件 |

## 4.2 模型适配模块

| 模块 | 作用 |
|------|------|
| `langchain4j-open-ai` | OpenAI / OpenAI 兼容接口 |
| `langchain4j-ollama` | 本地 Ollama |
| `langchain4j-anthropic` / `-google-ai-gemini` / `-qwen` / `-bedrock` … | 各厂商适配 |

### 4.2.1 OpenAI 兼容的意义

只要厂商协议与 OpenAI 兼容，就可以复用 `langchain4j-open-ai`，这是覆盖国产模型最省事的方式。

### 4.2.2 厂商专用模块的价值

专用模块能暴露兼容协议没有的能力，例如厂商特有的缓存、思考模式、系统提示字段。

## 4.3 存储与解析模块

| 模块 | 作用 |
|------|------|
| `langchain4j-{store}` | 各向量库（pgvector、milvus、qdrant、redis、chroma…） |
| `langchain4j-document-parser-*` | PDF / Office / HTML 解析 |

## 4.4 集成与扩展模块

| 模块 | 作用 |
|------|------|
| `langchain4j-spring-boot-starter` | Spring Boot 集成 |
| `langchain4j-mcp` | MCP 协议支持 |
| `langchain4j-agentic` | Agent 与多 Agent 编排（1.x 新增） |

> 模块之间通过接口解耦：只要实现了 `EmbeddingStore`，业务代码不感知背后是 pgvector 还是 Milvus。

# 5. 版本演进

## 5.1 主要里程碑

| 版本 | 时间 | 关键变化 |
|------|------|----------|
| 0.x | 2023–2025 | 快速迭代，API 频繁调整 |
| 1.0.0 GA | 2025-05 | 首个正式版，API 稳定，`ChatModel` 命名统一 |
| 1.1x – 1.19.x | 2025–2026 | MCP 完善、Agentic 模块、向量库增强 |
| 1.20.0 | 2026-09 | 非阻塞/响应式支持、Jackson 3 可选、MCP 2026-07 协议 |

> 本笔记以 **1.x GA API** 为主线，涉及版本差异会单独标注。

## 5.2 1.0 GA 的破坏性变化

从 0.x 升到 1.0 时最需要注意的是重命名：

| 旧名（0.x） | 新名（1.x） |
|-------------|-------------|
| `ChatLanguageModel` | `ChatModel` |
| `StreamingChatLanguageModel` | `StreamingChatModel` |
| `EmbeddingModel`（部分旧签名） | 统一为 `embed` 系列方法 |

迁移时优先全量搜索旧接口名，再逐一替换。

## 5.3 版本升级注意事项

- 通过 `langchain4j-bom` 统一版本，避免子模块版本漂移；
- 关注 release notes 中的 `breaking change` 标记；
- 升级前用监听器抓取真实请求，回归验证行为未变。

# 6. 与 Spring AI 的对比

## 6.1 对比表

| 维度 | LangChain4j | Spring AI |
|------|-------------|-----------|
| 出身 | 社区，后与 Quarkus 生态紧密 | Spring 官方 |
| 核心特色 | 声明式 `AiServices`、组件丰富 | 统一抽象 + Spring 自动配置 |
| 上手速度 | 快，几乎零配置即可用 | 需要理解 Spring 抽象 |
| 与 Spring 集成 | 提供 Starter，可用但非原生 | 原生无缝 |
| 抽象层次 | 高阶组件多，开箱即用 | 偏底层统一，组合自由 |
| 社区规模 | Java LLM 领域最大 | 增长快，背靠 Spring |

## 6.2 选型决策

> 结论：**快速原型/组件定制选 LangChain4j；深度 Spring 体系选 Spring AI。** 详见 `9_Frameworks/0_framework_comparison.md`。

补充判断维度：

- 团队已有大量 Spring 基础设施 → Spring AI 更顺；
- 需要丰富向量库/解析器适配 → LangChain4j 更全；
- 使用 Quarkus / 原生镜像 → LangChain4j 更友好。

## 6.3 混用与迁移

两者都是普通 Java 库，理论上可在同一项目共存（分别持有不同 `ChatModel` 实例）。但混用会增加依赖体积与认知负担，建议二选一，通过适配层隔离。

### 6.3.1 迁移检查清单

- 依赖坐标与版本是否已对齐；
- 是否仍在使用旧接口名（如 `ChatLanguageModel`）；
- 记忆、检索等组件是否有等价实现；
- 提示模板与工具定义是否需要调整；
- 观测与超时策略是否已补齐。

# 7. 适用场景

## 7.1 推荐场景

- 快速搭建 Chatbot、RAG 知识库；
- 用声明式接口把 LLM 能力嵌入既有 Java 服务；
- 需要丰富组件（多种向量库、文档解析器、MCP）的场景；
- Quarkus / 云原生应用（原生镜像友好）。

## 7.2 谨慎场景

| 场景 | 原因 |
|------|------|
| 需要极致自定义编排图 | 更适合 LangGraph 类方案 |
| 纯 Python 数据科学生态 | 直接选 Python 版更省事 |
| 已深度绑定 Spring 全家桶 | Spring AI 集成成本更低 |

## 7.3 反例：不适合直接套用的需求

- 高频低延迟的确定性规则引擎：无需 LLM；
- 强一致性事务：LLM 输出不确定，不能作为唯一裁决；
- 完全离线且无 GPU：本地模型体验可能不达标。

# 8. 学习建议

## 8.1 学习路径

1. 先跑通 `ChatModel` 与 `AiServices` 两个最小示例。
2. 理解"低层可控 / 高层声明式"的双轨设计。
3. 按 `Prompt → AI Services → Tools → Memory → RAG → Agent` 顺序推进。
4. 每学一个组件都对照 Spring AI 的等价实现，建立迁移能力。

## 8.2 常见误区

- 把 `AiServices` 当黑盒，出问题不会用监听器排查；
- 在低层 API 里手写记忆与工具循环，重复造轮子；
- 忽略超时与重试，线上偶发线程挂死；
- 把 API Key 写进代码或提交到仓库。

## 8.3 推荐练习

| 练习 | 目标组件 |
|------|----------|
| 命令行问答 | `ChatModel` |
| 带人设的多轮助手 | `AiServices` + `ChatMemory` |
| 天气查询助手 | `@Tool` |
| 本地文档问答 | RAG 五件套 |

# 9. 常见问题（FAQ）

## 9.1 与 LangChain4j 相关的常见疑问

| 问题 | 简答 |
|------|------|
| 必须用 Spring 吗 | 不必须，纯 Java 也能用 |
| 支持国产模型吗 | 支持，OpenAI 兼容或专用模块 |
| 能离线运行吗 | 可以，配合 Ollama 等本地模型 |
| 有中文文档吗 | 官方文档为英文，社区有中文整理 |
| 支持 GraalVM 吗 | Quarkus 场景下较成熟 |

## 9.2 遇到问题时的排查顺序

1. 打开 `logRequests` / `logResponses` 看真实报文；
2. 检查 API Key、baseUrl、modelName 是否正确；
3. 用监听器确认请求是否发出、响应是否返回；
4. 对比厂商文档确认参数与角色支持情况。

# 10. 小结

- LangChain4j 是**为 Java 独立设计**的 LLM 应用框架，而非 LangChain 移植版。
- 双轨设计是其灵魂：低层 `ChatModel` 求可控，高层 `AiServices` 求效率。
- 模块化 + 类型安全 + 约定优于配置，让它在企业 Java 中落地成本低。
- 与 Spring AI 是互补关系，按团队技术栈与组件需求选择。
- 学习路径应遵循"先跑通、再拆解、后组合"的顺序。
