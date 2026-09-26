# 1. 模块定位

本模块是**整套笔记的重心**。前面所有原理，最终都要落到框架上才能高效开发。这里以 **Spring AI** 与 **LangChain4j** 为双主线深入讲解，其余框架作为补充与对照。

每个框架都遵循统一的三段式讲解：**理论基础 → 落地实战 → 企业化实践**。

## 1.1 本模块解决什么问题

- 把前序模块的原理（提示、工具、记忆、RAG、Agent 循环）映射到具体框架 API。
- 建立"模型层抽象"，使业务代码不绑定任何单一框架。
- 提供可执行的选型依据，避免"只看功能表"式决策。
- 给出企业级落地模式：多模型路由、可靠性、可观测与多租户。

### 1.1.1 为什么框架是"加速器"而非"地基"

框架解决的是"写得快"，原理解决的是"做得对"。原理不清时，框架会把错误设计固化，且一旦更换框架，业务代码需整体返工。因此本模块始终先讲机制，再讲 API。

### 1.1.2 三段式讲解法

| 阶段 | 目标 | 典型产出 |
|------|------|----------|
| 理论基础 | 理解框架的抽象与机制 | 抽象映射图、调用链 |
| 落地实战 | 能跑通一个真实场景 | 可运行的最小示例 |
| 企业化实践 | 能安全可靠地生产化 | 路由、容错、可观测方案 |

## 1.2 在整个知识体系中的位置

```
PromptEngineering / LLMEngineering / RAG / AgentCore / Tools / Memory
                              │
                              ▼
                     Frameworks（本模块）
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   MultiAgent            Evaluation           ProductionEngineering
```

### 1.2.1 与相邻模块的边界

| 模块 | 关注点 | 与本模块的边界 |
|------|--------|----------------|
| `6_AgentCore` | Agent 原理与循环 | 本模块讲框架如何封装循环 |
| `7_ToolsAndProtocols` | 工具与 MCP 协议 | 本模块讲框架如何接入工具 |
| `8_Memory` | 记忆原理 | 本模块讲框架的 ChatMemory |
| `14_ProductionEngineering` | 生产化工程 | 本模块只覆盖框架侧实践 |

## 1.3 双主线与补充框架

- 主线一 Spring AI：与 Spring 生态无缝、抽象统一、企业友好。
- 主线二 LangChain4j：组件丰富、声明式 AI Services、上手快。
- 补充框架：Spring AI Alibaba、Semantic Kernel、Embabel、Google ADK、Quarkus LangChain4j，按场景与生态选用。

# 2. 学习目标

## 2.1 知识目标

- 深入掌握 Spring AI 与 LangChain4j 的核心抽象与源码级机制。
- 理解各框架的设计哲学与适用场景，能做出选型决策。
- 理解 MCP、RAG、记忆在框架中的装配方式与差异。

## 2.2 能力目标

- 能用框架完成 Chat、Prompt、结构化输出、工具调用、记忆、RAG、MCP 集成。
- 能把框架用法落地到企业级架构中。
- 能用自有接口隔离框架 API，保持模型层可替换。
- 能对同一场景在两套主线框架间完成迁移。

## 2.3 前置知识

| 前置 | 说明 |
|------|------|
| Spring Boot | 依赖注入、自动配置、Starter |
| Java 17+ | 记录类型、密封类、虚拟线程 |
| 前序模块 | 提示、工具、记忆、RAG、Agent 循环 |
| 构建工具 | Maven / Gradle 依赖管理 |

# 3. 文件清单与大纲

## 3.1 根目录

| 文件 | 大纲 |
|------|------|
| `0_framework_comparison.md` | Java AI 框架全景对比、选型决策树、生态趋势 |

## 3.2 SpringAI/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | Spring AI 定位、设计理念、版本与生态 |
| `2_environment_setup.md` | 依赖引入、API Key 配置、多模型接入 |
| `3_core_concepts.md` | ChatModel / ChatClient / Advisor / VectorStore 等核心抽象 |
| `4_chat_client.md` | ChatClient 用法：调用、参数、多模型切换 |
| `5_prompt_engineering.md` | PromptTemplate、System/User 消息、模板渲染 |
| `6_structured_output.md` | BeanOutputConverter、实体映射与校验 |
| `7_function_calling_tools.md` | `@Tool` 注解、工具注册、执行闭环 |
| `8_chat_memory.md` | ChatMemory 抽象、JDBC/Redis 实现、会话隔离 |
| `9_advisors.md` | Advisor 机制、日志/记忆/RAG Advisor、自定义 Advisor |
| `10_rag.md` | Spring AI RAG 全链路：ETL、检索、增强 |
| `11_vector_store.md` | VectorStore 抽象与各向量库适配 |
| `12_etl_pipeline.md` | DocumentReader / Splitter / Writer ETL 流水线 |
| `13_mcp_integration.md` | Spring AI 的 MCP Server / Client 集成 |
| `14_multimodality.md` | 图像、音频等多模态能力接入 |
| `15_observability.md` | 与 Micrometer / Actuator 集成的可观测性 |
| `16_evaluation_testing.md` | Evaluator、测试支持与集成测试 |
| `17_enterprise_patterns.md` | 企业级：多模型路由、配置治理、可靠性、权限 |
| `18_best_practices.md` | Spring AI 最佳实践与常见坑 |

## 3.3 LangChain4j/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | LangChain4j 定位、与 LangChain 的关系、模块划分 |
| `2_core_concepts.md` | ChatLanguageModel / AiServices / Memory / Retriever 等抽象 |
| `3_chat_models.md` | 各厂商模型接入、流式、多模态、参数配置 |
| `4_prompt_templates.md` | 提示模板、变量、消息模板 |
| `5_ai_services.md` | 声明式 AI Services、接口即 Agent |
| `6_tools.md` | `@Tool`、工具注册与执行、工具错误处理 |
| `7_chat_memory.md` | ChatMemory、消息窗口、持久化实现 |
| `8_rag.md` | RAG 组件：加载、切分、检索、增强 |
| `9_embeddings_vector_store.md` | Embedding 模型与向量库集成 |
| `10_document_parsers.md` | 文档解析器（PDF/Office/HTML）实战 |
| `11_agents.md` | LangChain4j 的 Agent 能力与工作流 |
| `12_observability.md` | 监听器、日志、与可观测平台集成 |
| `13_spring_boot_integration.md` | Spring Boot Starter 集成与配置 |
| `14_enterprise_patterns.md` | 企业级落地模式、可靠性、多租户 |

## 3.4 SpringAIAlibaba/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | 定位、与 Spring AI 的关系、国内生态适配 |
| `2_core_concepts.md` | 核心抽象与扩展点 |
| `3_models.md` | 通义等国内模型接入与配置 |
| `4_graph.md` | Graph 编排：节点、边、状态流转 |
| `5_rag.md` | 面向国内场景的 RAG 能力 |
| `6_observability.md` | 可观测性与调优 |
| `7_enterprise.md` | 企业级实践与生产案例 |

## 3.5 SemanticKernel/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | Semantic Kernel 设计理念、多语言支持 |
| `2_kernel_and_plugins.md` | Kernel、Plugin、Function 抽象 |
| `3_memory.md` | 记忆与向量存储集成 |
| `4_planners.md` | Planner 规划器与自动编排 |
| `5_agents.md` | Agent Framework 与协作 |
| `6_enterprise.md` | 企业集成与落地 |

## 3.6 Embabel/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | Embabel 定位、基于 GOAP 的 Agent 框架理念 |
| `2_goap_planning.md` | 目标-行动规划机制与领域建模 |
| `3_agents.md` | Agent 定义、行动与条件 |
| `4_enterprise.md` | Spring 生态集成与企业实践 |

## 3.7 GoogleADKJava/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | Google ADK（Java）定位与概念 |
| `2_agents.md` | Agent 定义、指令、模型配置 |
| `3_tools.md` | 工具接入与内置工具 |
| `4_multiagent.md` | 多 Agent 编排与协作 |
| `5_deployment.md` | 部署与运行时 |

## 3.8 QuarkusLangChain4j/

| 文件 | 大纲 |
|------|------|
| `1_introduction.md` | Quarkus + LangChain4j 组合、云原生优势 |
| `2_basics.md` | 快速上手、原生镜像与启动优化 |
| `3_rag.md` | Quarkus 下的 RAG 实现 |
| `4_enterprise.md` | 企业级与 Serverless 场景 |

## 3.9 模块间关系

```
0_framework_comparison（全局认知）
        │
        ├── SpringAI/（主线一）
        ├── LangChain4j/（主线二）
        └── 补充框架（Alibaba / SK / Embabel / ADK / Quarkus）
```

# 4. 学习顺序与依赖

## 4.1 推荐学习路径

```
0_framework_comparison（先建立全局认知）
      ↓
SpringAI（主线一，1→18）
      ↓
LangChain4j（主线二，1→14）
      ↓
补充框架（按需：SpringAIAlibaba / SemanticKernel / Embabel / ADK / Quarkus）
```

## 4.2 依赖关系

### 4.2.1 上游依赖

- `3_PromptEngineering`、`4_LLMEngineering`、`5_RAG`。
- `6_AgentCore`、`7_ToolsAndProtocols`、`8_Memory`。

### 4.2.2 下游影响

- `10_MultiAgent`、`12_Evaluation`、`13_Observability`。
- `14_ProductionEngineering`、`16_Applications`。

## 4.3 阶段划分与里程碑

| 阶段 | 内容 | 里程碑 |
|------|------|--------|
| 阶段一 | 全局对比 | 能给出选型理由 |
| 阶段二 | Spring AI 1→18 | 能完成端到端 RAG + 工具 |
| 阶段三 | LangChain4j 1→14 | 能用 AiServices 实现 Agent |
| 阶段四 | 补充框架 | 能按场景选并快速上手 |

# 5. 关键知识点速览

## 5.1 核心结论

- Spring AI 的优势是"与 Spring 生态无缝、抽象统一、企业友好"。
- LangChain4j 的优势是"组件丰富、声明式 AI Services、上手快"。
- 框架核心抽象高度相似：模型、提示、工具、记忆、检索器。
- 框架只是加速器，原理不清时框架会成为黑盒。
- 选型看团队技术栈与治理诉求，而非单纯功能对比。

## 5.2 术语表

| 术语 | 含义 |
|------|------|
| ChatClient | Spring AI 的命令式对话门面 |
| AiServices | LangChain4j 的声明式接口门面 |
| Advisor | Spring AI 的调用链拦截组件 |
| ChatMemory | 对话记忆抽象 |
| VectorStore | 向量存储抽象 |
| ContentRetriever | LangChain4j 的检索抽象 |
| MCP | 模型上下文协议，工具接入标准 |
| Kernel | Semantic Kernel 的核心容器 |

## 5.3 自检清单

- [ ] 能说清 Spring AI 与 LangChain4j 的抽象对应关系。
- [ ] 能为一个场景给出选型理由并说明取舍。
- [ ] 能用两条主线各实现一次工具调用与记忆。
- [ ] 能用自有接口隔离框架 API。
- [ ] 能判断何时需要组合多个框架。
