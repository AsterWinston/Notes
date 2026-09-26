# 1. 项目介绍

本目录是一套 **以 Java 生态为主线** 的 AI Agent 学习笔记，覆盖「AI 基础 → 大模型原理 → Prompt / LLM 应用 → RAG → Agent → 主流框架 → 多智能体 → 生产化与企业级」的完整路径。

目标是：**从零学起，理论讲透，代码可跑，最终能落地并企业化**。

## 1.1 这是什么

这是一套按主题分层的长期笔记工程，而不是一本线性教材。它把 AI Agent 从底层原理到企业落地的全部环节拆成 17 个一级大类，每个大类再拆成若干主题笔记，便于按需检索与逐步深入。

## 1.2 目标读者

| 读者类型 | 建议起点 |
|----------|----------|
| Java 后端工程师转 AI 方向 | 从 `1_Foundations` 开始顺读 |
| 已有 AI 基础想补工程能力 | 从 `4_LLMEngineering` 起，重点看 5 到 16 |
| 面试冲刺 | 直接看 `17_Interview` 并按索引回查 |
| 架构与平台同学 | 重点看 `13_Observability` 到 `16_Applications` |

## 1.3 设计原则

- 原理讲透：不满足于「会用」，要能解释「为什么」。
- 代码可跑：落地代码统一用 Java，避免只停留在伪代码。
- 面向生产：每篇力求「原理 → 代码 → 生产注意事项」三段式。
- 索引优先：用 README 与交叉引用构建可检索的知识网络。

## 1.4 如何使用本仓库

- 先看第 2 章的路线图确定阶段，再看第 3 章定位具体目录。
- 每个一级大类都有 `README.md`，进入正文前先读模块 README。
- 遇到跨领域问题，用第 4 章的企业级关注点索引横向跳转。

## 1.5 学习节奏建议

| 节奏 | 每天投入 | 适合人群 |
|------|:--------:|----------|
| 稳扎稳打 | 1 到 2 小时 | 在职学习，周期约 3 个月 |
| 集中攻坚 | 4 小时以上 | 转岗冲刺，周期约 1 个月 |
| 查漏补缺 | 按需 | 已有基础，只补短板 |

无论哪种节奏，都建议「读一段、写一段代码」，避免只看不练。

---

# 2. 学习路线图

## 2.1 阶段划分

| 阶段 | 目标 | 涉及模块 |
|------|------|----------|
| 第一阶段：打地基 | 理解神经网络、Transformer、NLP 与 Token 机制 | `1_Foundations`、`2_LLM` |
| 第二阶段：会用模型 | 掌握提示工程、模型 API、结构化输出、函数调用 | `3_PromptEngineering`、`4_LLMEngineering` |
| 第三阶段：喂知识 | 搭建可用的 RAG 系统 | `5_RAG` |
| 第四阶段：造 Agent | 理解 Agent 原理并实现单智能体 | `6_AgentCore`、`7_ToolsAndProtocols`、`8_Memory` |
| 第五阶段：用框架 | 基于 Spring AI / LangChain4j 等框架工程化开发 | `9_Frameworks` |
| 第六阶段：上多智能体 | 多 Agent 协作与编排 | `10_MultiAgent` |
| 第七阶段：调优与评测 | 模型定制、评测、可观测性 | `11_ModelCustomization`、`12_Evaluation`、`13_Observability` |
| 第八阶段：企业化 | 生产可靠性、安全、部署、实战项目 | `14_ProductionEngineering`、`15_Security`、`16_Applications` |
| 冲刺：面试 | 高频考点与系统设计 | `17_Interview` |

## 2.2 推荐阅读顺序

```
1_Foundations → 2_LLM → 3_PromptEngineering → 4_LLMEngineering
      → 5_RAG → 6_AgentCore → 7_ToolsAndProtocols → 8_Memory
      → 9_Frameworks（Spring AI 与 LangChain4j 为主线）
      → 10_MultiAgent → 11_ModelCustomization
      → 12_Evaluation / 13_Observability
      → 14_ProductionEngineering / 15_Security
      → 16_Applications → 17_Interview
```

## 2.3 贯穿性主题

> 说明：`12_Evaluation`、`13_Observability`、`14_ProductionEngineering`、`15_Security` 属于贯穿性主题，可在学习框架阶段就穿插阅读。

- 不必等到最后才看评测与观测，越早建立反馈闭环越好。
- 安全与可靠性同理，属于每一步都要带的习惯，而不是收尾补丁。

## 2.4 学习方式

- 主线精读：按推荐顺序通读，建立完整心智模型。
- 支线穿插：评测、观测、安全按需提前阅读。
- 项目驱动：以 `16_Applications` 的项目为目标反推需要补的模块。

---

# 3. 目录索引

## 3.1 全目录总表

| 目录 | 主题 | 核心产出 |
|------|------|----------|
| `1_Foundations` | AI 基础（深度学习 / Transformer / NLP） | 理解模型为何能"理解"语言 |
| `2_LLM` | 大模型原理（训练 / 推理 / 模型谱系） | 理解大模型的能力边界与成本来源 |
| `3_PromptEngineering` | 提示工程 | 写出稳定、可复用的 Prompt |
| `4_LLMEngineering` | LLM 应用工程 | 用 Java 调用模型并处理工程细节 |
| `5_RAG` | 检索增强生成 | 搭建企业级知识库问答 |
| `6_AgentCore` | Agent 核心原理 | 从零实现一个 ReAct Agent |
| `7_ToolsAndProtocols` | 工具调用与协议 | 工具设计、MCP、A2A |
| `8_Memory` | 记忆系统 | 短期/长期记忆与会话上下文 |
| `9_Frameworks` | 主流框架（Java 生态） | Spring AI / LangChain4j 等深度实战 |
| `10_MultiAgent` | 多智能体 | 协作、编排与失败模式 |
| `11_ModelCustomization` | 模型定制与微调 | LoRA、SFT、本地部署 |
| `12_Evaluation` | 评测 | 用数据衡量 Agent 好不好 |
| `13_Observability` | 可观测性 | Trace / Metrics / 成本监控 |
| `14_ProductionEngineering` | 生产化与企业级工程 | 状态持久化、可靠性、性能、部署 |
| `15_Security` | 安全与对齐 | 注入防御、权限控制、MCP 隔离 |
| `16_Applications` | 实战项目 | 可写进简历的完整项目 |
| `17_Interview` | 面试 | 高频考点与系统设计 |

## 3.2 按主题归类

### 3.2.1 基础层

`1_Foundations`、`2_LLM`、`3_PromptEngineering`、`4_LLMEngineering`，解决「模型为什么能用」与「怎么调用模型」。

### 3.2.2 应用层

`5_RAG`、`6_AgentCore`、`7_ToolsAndProtocols`、`8_Memory`、`9_Frameworks`、`10_MultiAgent`，解决「怎么造出一个 Agent」。

### 3.2.3 工程层

`11_ModelCustomization` 到 `17_Interview`，解决「怎么让它稳定、可控、能上线、能讲清」。

## 3.3 目录依赖关系

```
  1_Foundations → 2_LLM → 3_PromptEngineering → 4_LLMEngineering
        │
        ▼
     5_RAG ──► 6_AgentCore ──► 7_ToolsAndProtocols ──► 8_Memory
        │
        ▼
     9_Frameworks ──► 10_MultiAgent ──► 11_ModelCustomization
        │
        ▼
  12_Evaluation / 13_Observability / 14_ProductionEngineering
        │
        ▼
  15_Security / 16_Applications ──► 17_Interview
```

该图只是主干关系，实际阅读时评测与观测可随时穿插。

---

# 4. 企业级关注点索引

## 4.1 关注点总表

生产中真正让 Agent 从"能演示"到"能上线"的关键点，统一索引如下：

| 关注点 | 落点 |
|--------|------|
| Agent 状态持久化 | `14_ProductionEngineering/Runtime/1_agent_state_persistence`、`6_AgentCore/10_state_and_control_flow` |
| 会话管理 | `14_ProductionEngineering/Runtime/2_session_management`、`8_Memory/3_short_term_memory` |
| Token 管理 | `14_ProductionEngineering/Performance/3_token_budget_management`、`4_LLMEngineering/6_token_and_cost_management` |
| 上下文压缩 | `8_Memory/8_memory_compression`、`3_PromptEngineering/10_context_engineering` |
| Tool 超时 | `14_ProductionEngineering/Reliability/1_tool_timeout` |
| Tool 重试 | `14_ProductionEngineering/Reliability/2_tool_retry` |
| 幂等 | `14_ProductionEngineering/Reliability/3_idempotency` |
| 限流 | `14_ProductionEngineering/Reliability/4_rate_limiting` |
| 熔断 | `14_ProductionEngineering/Reliability/5_circuit_breaker` |
| Agent 观测 | `13_Observability` |
| Trace | `13_Observability/2_tracing`、`13_Observability/6_opentelemetry` |
| Evaluation | `12_Evaluation` |
| Prompt 管理 | `3_PromptEngineering/13_prompt_management` |
| 权限控制 | `15_Security/10_access_control_and_authz` |
| MCP Tool 权限隔离 | `7_ToolsAndProtocols/MCP/6_security`、`15_Security/9_mcp_permission_isolation` |

## 4.2 如何用好这张表

- 设计阶段：对照表格逐项确认，缺失的关注点要在架构里预留位置。
- 实现阶段：每个落点都当成一个 checklist 项，做完打勾。
- 复盘阶段：线上事故往往对应表中被忽略的一行，按表反查最快。

## 4.3 关注点与阶段映射

| 学习阶段 | 应重点关注的条目 |
|----------|------------------|
| 造 Agent | 状态持久化、会话管理、上下文压缩 |
| 用框架 | Prompt 管理、Token 管理、Agent 观测 |
| 上多智能体 | Trace、Evaluation |
| 企业化 | Tool 超时/重试、幂等、限流、熔断 |
| 安全 | 权限控制、MCP Tool 权限隔离 |

---

# 5. 技术栈约定

## 5.1 语言与框架

- **主语言**：Java（JDK 17+ / 21）。
- **主线框架**：Spring AI、LangChain4j。
- **补充框架**：Spring AI Alibaba、Semantic Kernel (Java)、Embabel、Google ADK (Java)、Quarkus LangChain4j。
- **工程基础**：Spring Boot、WebFlux / SSE、Resilience4j、Micrometer、OpenTelemetry。

## 5.2 数据与模型

- **向量与存储**：Milvus、Qdrant、Redis Vector、Elasticsearch、pgvector。
- **模型接入**：OpenAI 兼容协议、Ollama / vLLM 本地部署。
- **Python 说明**：AI 论文与部分工具链以 Python 为主，阅读时以"能看懂原理"为准，落地代码统一用 Java。

## 5.3 版本与兼容

| 项 | 约定 |
|----|------|
| JDK | 17 起步，推荐 21 |
| 构建 | Maven / Gradle 均可 |
| 协议 | 优先 OpenAI 兼容协议，便于切换供应商 |
| 本地部署 | Ollama 用于开发，vLLM 用于性能验证 |

## 5.4 环境准备

- 本地安装 JDK 17 或 21，并准备一个 Maven 或 Gradle 工程。
- 开发期用 Ollama 拉取小模型，降低调试成本。
- 生产验证再切到 vLLM 或云端 OpenAI 兼容接口。

---

# 6. 笔记写作规范

## 6.1 命名与编号

- 文件名采用「数字前缀 + 英文 snake_case」，如 `1_introduction.md`、`3_kv_cache.md`。
- 正文采用 `# 1.` / `## 1.1` / `### 1.1.1` 三级编号。

## 6.2 内容组织

- 对比用表格，流程与架构用 ASCII 图，代码用带语言标注的围栏代码块。
- 每篇力求"原理 → 代码 → 生产注意事项"三段式。
- 每个一级目录都配 `README.md`，内含定位、文件清单与学习顺序。

## 6.3 图片与代码

- 图片存放于同级 `images/` 目录，引用格式：`<img src="images/01.png" width="100%"/>`。
- 代码块务必标注语言，保证渲染与检索体验一致。

## 6.4 常见写作问题

- 标题不编号，导致无法交叉引用。
- 代码块缺少语言标注，渲染与检索体验差。
- 只写结论不写推理，复习时无法还原思路。

---

# 7. 规模统计

## 7.1 填充进度

| 模块 | 骨架 | 正文 |
|------|:----:|:----:|
| 1_Foundations | 完成 | 完成 |
| 2_LLM | 完成 | 完成 |
| 3_PromptEngineering | 完成 | 完成 |
| 4_LLMEngineering | 完成 | 完成 |
| 5_RAG | 完成 | 完成 |
| 6_AgentCore | 完成 | 完成 |
| 7_ToolsAndProtocols | 完成 | 完成 |
| 8_Memory | 完成 | 完成 |
| 9_Frameworks/SpringAI | 完成 | 完成 |
| 9_Frameworks/LangChain4j | 完成 | 完成 |
| 9_Frameworks/其他 | 完成 | 完成 |
| 10_MultiAgent | 完成 | 完成 |
| 11_ModelCustomization | 完成 | 完成 |
| 12_Evaluation | 完成 | 完成 |
| 13_Observability | 完成 | 完成 |
| 14_ProductionEngineering | 完成 | 完成 |
| 15_Security | 完成 | 完成 |
| 16_Applications | 完成 | 完成 |
| 17_Interview | 完成 | 完成 |

## 7.2 规模统计

| 项 | 数量 |
|----|------|
| 一级大类 | 17 |
| 模块 README | 18 |
| 主题笔记 | 329 |
| 合计 md 文件 | 347 |

## 7.3 统计口径说明

- 模块 README 指一级目录及 `9_Frameworks` 子目录下的 `README.md`。
- 主题笔记指除 README 外的全部正文文件。
- 合计为上述两类之和，统计以当前仓库实际文件为准。

## 7.4 阅读进度建议

| 进度 | 标志 |
|------|------|
| 入门 | 读完基础层四个模块 |
| 进阶 | 能独立实现单 Agent |
| 熟练 | 能基于框架做工程化开发 |
| 企业级 | 能设计并上线一个完整应用 |

---

# 8. 常见问题

## 8.1 需要什么基础

有 Java 后端基础即可起步；深度学习部分从零讲起，不要求数学推导功底。

## 8.2 要不要学 Python

要能看懂，不必强求熟练。论文与开源工具多为 Python，但本笔记落地代码统一用 Java。

## 8.3 学完能做什么

能独立设计并实现一个可演示、可评测、可上线的企业级 AI Agent 应用，并能在面试中讲清原理与取舍。

## 8.4 反馈与纠错

- 发现错误或表述不清，欢迎提交 issue 或直接修改。
- 新增内容请遵循第 6 章的写作规范。
- 修改后请同步更新第 7 章的规模统计。

---

# 9. 状态说明

**全部内容已完成。**

- 一级大类均包含 `README.md`，内含该模块的定位、文件清单大纲、学习顺序与知识点速览。
- 全部 347 篇笔记（含 18 篇模块 README）已完成**正文填充**，无待填占位。

## 9.1 完成情况

所有模块的骨架与正文均已填充完毕，可直接按路线图阅读，无需等待补充。

## 9.2 后续维护

- 新增主题时，同步更新本 README 的目录索引与规模统计。
- 统计数字（17、18、329、347）在文件增删后需一并核对。

## 9.3 版本记录

| 版本 | 说明 |
|------|------|
| v1.0 | 完成全部模块骨架与正文填充 |
| 后续 | 按反馈持续修订与扩充 |

