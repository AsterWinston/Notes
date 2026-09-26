# 1. 全景对比

Java 生态的 AI 框架已从"能用"走向"分工明确"。选型前先建立全局认知，避免被单一框架的营销话术绑架。

## 1.1 七大框架一览

| 框架 | 出身 | 核心特色 | 与 Spring 集成 | 上手难度 |
|------|------|----------|:--------------:|:--------:|
| Spring AI | Spring 官方 | 统一抽象 + 自动配置 | 原生 | 中 |
| LangChain4j | 社区 | 声明式 AiServices、组件丰富 | Starter | 低 |
| Spring AI Alibaba | 阿里 | 国内模型 + Graph 编排 | 原生 | 中 |
| Semantic Kernel | 微软 | Kernel/Plugin/Planner | 可集成 | 中 |
| Embabel | 社区 | GOAP 规划 | Spring 原生 | 中高 |
| Google ADK (Java) | Google | Agent 定义 + 多 Agent | 可集成 | 中 |
| Quarkus LangChain4j | Quarkus | 云原生 + 原生镜像 | Quarkus | 中 |

## 1.2 出身与治理模式

### 1.2.1 官方与社区之分

框架的"出身"决定了它的稳定性预期、更新节奏与长期维护风险。

| 类型 | 代表 | 优势 | 风险 |
|------|------|------|------|
| 厂商官方 | Spring AI、Semantic Kernel、ADK | 长期维护、企业信任 | 路线受厂商战略影响 |
| 社区主导 | LangChain4j、Embabel | 迭代快、组件丰富 | 稳定性依赖社区活跃度 |
| 云厂商 | Spring AI Alibaba、Quarkus LangChain4j | 与平台深度整合 | 存在平台绑定倾向 |

### 1.2.2 治理模式对选型的影响

- 强合规行业优先选官方框架，审计与长期支持更可控。
- 快速验证阶段可先用社区框架，避免过度设计。
- 国内模型为主的团队，需评估云厂商框架的适配深度。

## 1.3 上手难度与成熟度

| 框架 | 学习曲线 | 文档质量 | 生态成熟度 |
|------|:--------:|:--------:|:----------:|
| Spring AI | 中 | 高 | 高 |
| LangChain4j | 低 | 高 | 高 |
| Spring AI Alibaba | 中 | 中 | 中高 |
| Semantic Kernel | 中 | 中 | 中高 |
| Embabel | 中高 | 中 | 中 |
| Google ADK (Java) | 中 | 中 | 中 |
| Quarkus LangChain4j | 中 | 中 | 中 |

# 2. 核心抽象对比

结论先行：主流框架的核心抽象高度相似，差异主要落在"门面 API"与"拦截机制"上。

## 2.1 抽象映射表

| 能力 | Spring AI | LangChain4j | Semantic Kernel |
|------|-----------|-------------|-----------------|
| 模型 | `ChatModel` | `ChatModel` | `Kernel` |
| 对话门面 | `ChatClient` | `AiServices` | `ChatCompletionService` |
| 提示 | `PromptTemplate` | `@SystemMessage` | `PromptTemplate` |
| 工具 | `@Tool` | `@Tool` | `KernelFunction` |
| 记忆 | `ChatMemory` | `ChatMemory` | `ChatHistory` |
| 向量 | `VectorStore` | `EmbeddingStore` | `Memory` |
| RAG | `QuestionAnswerAdvisor` | `ContentRetriever` | `Memory` + 插件 |
| 拦截 | `Advisor` | 无（用 Service 包装） | Filter |

## 2.2 抽象差异解析

### 2.2.1 门面差异：命令式与声明式

Spring AI 的 `ChatClient` 是流式构建的命令式门面；LangChain4j 的 `AiServices` 是接口代理式的声明式门面。前者灵活、后者简洁。

```java
// Spring AI 风格：命令式链式调用
String answer = chatClient.prompt()
        .system("你是订单助手")
        .user("查询订单 A123")
        .call()
        .content();
```

```java
// LangChain4j 风格：声明式接口
interface OrderAssistant {
    @SystemMessage("你是订单助手")
    String chat(@UserMessage String question);
}
```

### 2.2.2 拦截机制差异

Spring AI 的 `Advisor` 可插入调用链，适合做日志、记忆、RAG 增强；LangChain4j 无统一拦截层，通常通过包装 Service 或监听器实现横切逻辑。

| 维度 | Advisor | 监听器/包装 |
|------|---------|-------------|
| 侵入性 | 低（声明式注册） | 中（需自行封装） |
| 顺序控制 | 显式 order | 手动编排 |
| 典型用途 | 记忆、RAG、日志 | 日志、指标 |

## 2.3 Java 示例对照

### 2.3.1 工具定义对照

```java
// Spring AI
public class OrderTools {
    @Tool("查询订单状态")
    public String queryOrderStatus(String orderId) {
        return orderService.status(orderId);
    }
}
```

```java
// LangChain4j
public class OrderTools {
    @Tool("查询订单状态")
    public String queryOrderStatus(String orderId) {
        return orderService.status(orderId);
    }
}
```

> 注解名相同、语义相同，迁移时主要是注册方式与包名不同。

### 2.3.2 记忆配置对照

```java
// Spring AI 伪代码：注册记忆 Advisor
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
```

```java
// LangChain4j 伪代码：通过 AiServices 装配记忆
OrderAssistant assistant = AiServices.builder(OrderAssistant.class)
        .chatLanguageModel(model)
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
        .build();
```

# 3. 选型决策树

结论先行：先看团队技术栈，再看治理诉求，最后看模型来源与编排复杂度。

## 3.1 决策树

```
团队用 Spring Boot？
  ├── 是
  │    ├── 追求官方支持与统一抽象 → Spring AI
  │    ├── 追求声明式与组件丰富 → LangChain4j
  │    └── 国内模型 + 图编排 → Spring AI Alibaba
  └── 否
       ├── 用 Quarkus → Quarkus LangChain4j
       ├── 用 .NET 生态 → Semantic Kernel
       └── 需要 GOAP 规划 → Embabel
```

## 3.2 决策维度

### 3.2.1 五大维度

| 维度 | 说明 | 权重建议 |
|------|------|:--------:|
| 技术栈 | 现有框架与团队熟悉度 | 高 |
| 模型来源 | 国内外模型、私有化需求 | 高 |
| 编排复杂度 | 单 Agent 还是多 Agent/图 | 中高 |
| 治理诉求 | 可观测、权限、审计 | 中 |
| 交付速度 | 原型还是长期产品 | 中 |

### 3.2.2 权重取舍

- 长期产品：技术栈与治理优先，交付速度次之。
- 短期原型：交付速度优先，但需预留迁移接口。

## 3.3 边界与反例

- 反例一：只因"某框架功能全"就选它，忽视团队完全不熟，结果维护成本高企。
- 反例二：为省事把所有能力绑定在单一框架，模型层未抽象，换模型时全量返工。
- 反例三：盲目追求最新框架，文档与社区案例不足，踩坑无人可问。
- 边界：决策树只做初筛，最终必须用 PoC（概念验证）验证关键路径。

## 3.4 选型 PoC 检查清单

| 检查项 | 通过标准 |
|--------|----------|
| 模型接入 | 目标模型可稳定调用 |
| 工具调用 | 能完成一次真实工具闭环 |
| 结构化输出 | 实体映射一次成功 |
| 记忆 | 多轮上下文可保持 |
| RAG | 能召回并引用文档 |
| 可观测 | 有可用的追踪与指标 |
| 迁移预留 | 核心路径未直接依赖框架 API |

# 4. 场景推荐

## 4.1 场景推荐表

| 场景 | 推荐 |
|------|------|
| 企业级 Spring 应用 | Spring AI |
| 快速原型 | LangChain4j |
| 国内模型为主 | Spring AI Alibaba / LangChain4j |
| 云原生 / Serverless | Quarkus LangChain4j |
| 多 Agent 编排 | Spring AI Alibaba Graph / LangGraph4j |
| 复杂规划 | Embabel |
| 微软生态 | Semantic Kernel |

## 4.2 场景解析

### 4.2.1 企业级与国内模型

Spring AI 的优势在于与 Spring 生态无缝、抽象统一、企业友好；国内模型为主的团队则需要评估 Spring AI Alibaba 的适配深度与社区活跃度。

### 4.2.2 原型与云原生

LangChain4j 组件丰富、上手快，适合快速原型；Quarkus LangChain4j 在启动时间与内存占用上占优，适合 Serverless。

### 4.2.3 多 Agent 与复杂规划

多 Agent 编排推荐图式编排框架；需要目标-行动自动规划时，Embabel 的 GOAP 思路更贴合。

# 5. 能力矩阵

结论先行：基础能力各家趋同，差异集中在多 Agent、图编排与 MCP 支持深度。

## 5.1 能力矩阵表

| 能力 | Spring AI | LangChain4j | Alibaba | SK | ADK |
|------|:---------:|:-----------:|:-------:|:--:|:---:|
| 多模型接入 | 支持 | 支持 | 支持 | 支持 | 支持 |
| 结构化输出 | 支持 | 支持 | 支持 | 支持 | 支持 |
| 工具调用 | 支持 | 支持 | 支持 | 支持 | 支持 |
| 记忆 | 支持 | 支持 | 支持 | 支持 | 支持 |
| RAG | 支持 | 支持 | 支持 | 部分 | 支持 |
| MCP | 支持 | 支持 | 支持 | 部分 | 支持 |
| 多 Agent | 部分 | 支持 | 支持 | 支持 | 支持 |
| 可观测 | 支持 | 支持 | 支持 | 支持 | 支持 |
| 图编排 | 部分 | 部分 | 支持 | 部分 | 部分 |

## 5.2 能力差异解析

### 5.2.1 多 Agent 与图编排

Spring AI 与 LangChain4j 的多 Agent 能力通常借助外部编排库补足；Alibaba 的 Graph 与 LangGraph4j 提供原生图编排。

### 5.2.2 RAG 与 MCP

RAG 各家基本齐备，Semantic Kernel 的 Java 侧相对偏弱；MCP 已成为事实标准，主流框架均已支持或正在补齐。

## 5.3 图例与判读

| 标记 | 含义 |
|------|------|
| 支持 | 官方内置或官方插件 |
| 部分 | 需第三方库或能力较弱 |
| 弱 | 尚不成熟，谨慎用于生产 |

> 能力矩阵会随版本快速变化，选型时应以当前版本实测为准。

## 5.4 能力评测方法

结论先行：能力矩阵是"纸面结论"，真实差异要靠 PoC 量化。

### 5.4.1 评测维度

| 维度 | 度量方式 |
|------|----------|
| 工具调用准确率 | 期望工具被调用的比例 |
| RAG 召回质量 | 命中率与排序质量 |
| 流式首字延迟 | TTFT（首 Token 时间） |
| 内存与启动 | 冷启动时间、常驻内存 |
| 迁移成本 | 核心路径改动行数 |

### 5.4.2 评测原则

- 用同一业务场景、同一模型、同一数据集横向对比。
- 记录版本号，框架 API 变动频繁，结论需标注时效。
- 关注长尾失败，而非平均指标。

# 6. 生态与趋势

## 6.1 趋势表

| 趋势 | 说明 |
|------|------|
| MCP 成为标配 | 主流框架都支持 |
| Agent 化 | 从 Chat 到 Agent |
| 多 Agent 编排 | 图/声明式编排 |
| 可观测性 | OTel 集成 |
| 国内生态 | Alibaba 系崛起 |

## 6.2 趋势解读

### 6.2.1 MCP 与工具生态

MCP 统一了"模型如何接入外部能力"，工具生态从各家私有协议收敛为公共协议，框架的工具接入成本大幅下降。

### 6.2.2 Agent 化与可观测

框架重心正从"聊天"转向"Agent"，随之而来的是对轨迹、成本、失败率的可观测需求，OTel 集成成为标配。

## 6.3 对选型的影响

- 优先选 MCP 支持完善的框架，工具生态可直接复用。
- 优先选可观测体系完整的框架，降低上线风险。
- 关注国内生态，国内模型适配往往决定实际可用性。

## 6.4 版本与风险

### 6.4.1 演进节奏

AI 框架仍处高速演进期，API 破坏性变更并不罕见，选型时需评估升级成本。

| 风险 | 表现 | 缓解手段 |
|------|------|----------|
| API 频繁变更 | 升级即返工 | 隔离层 + 版本锁定 |
| 文档滞后 | 示例不可用 | 以源码与测试为准 |
| 生态分裂 | 插件互不兼容 | 优先 MCP 等标准 |
| 社区衰退 | 无人维护 | 关注提交与 issue 活跃度 |

### 6.4.2 缓解原则

- 用自有接口包住框架，升级时只改适配层。
- 锁版本 + 定期评估，避免被强制升级。

# 7. 迁移成本

## 7.1 迁移面

```
好消息：核心抽象高度相似
  ChatModel / Prompt / Tool / Memory / Retriever

迁移主要改：
  - 依赖与配置
  - 门面 API（ChatClient vs AiServices）
  - Advisor vs 自定义包装
```

> 理解原理后，框架间迁移成本不高。

## 7.2 迁移示例

### 7.2.1 Spring AI 到 LangChain4j

```java
// Spring AI：命令式调用
String r1 = chatClient.prompt().user(q).call().content();

// LangChain4j：声明式接口
interface Assistant {
    String chat(@UserMessage String q);
}
Assistant a = AiServices.create(Assistant.class, model);
String r2 = a.chat(q);
```

### 7.2.2 LangChain4j 到 Spring AI

```java
// LangChain4j：工具在接口上声明
interface Tools {
    @Tool("查询天气")
    String weather(String city);
}

// Spring AI：工具在 Bean 上声明并注册为回调
@Bean
ToolCallback weatherTool() {
    return ToolCallbacks.from(new WeatherTools())[0];
}
```

## 7.3 边界与坑

- 提示模板语法不同：变量占位符与消息角色组织方式有差异。
- 记忆实现不同：窗口策略与持久化接口不通用。
- 拦截机制不同：Advisor 逻辑需改写为监听器或包装。
- 边界：迁移成本低的前提是"业务代码依赖自身抽象，而非框架 API"。

# 8. 组合使用

结论先行：框架不是互斥的，可按能力组合，用各自最强项。

## 8.1 组合模式

```
Spring AI（主）+ LangGraph4j（多 Agent 编排）
LangChain4j（主）+ Spring AI 的 VectorStore
Spring AI + MCP（工具生态）
```

## 8.2 组合示例

### 8.2.1 主框架加编排库

以 Spring AI 作为模型与工具层，以 LangGraph4j 负责多 Agent 的图式编排，二者通过共享的模型接口衔接。

```java
// 伪代码：图节点内部调用 Spring AI 的 ChatClient
Node planner = state -> {
    String plan = chatClient.prompt().user(state.question()).call().content();
    return state.withPlan(plan);
};
```

### 8.2.2 复用向量存储

LangChain4j 为主时，可直接接入 Spring AI 的 `VectorStore` 实现，避免重复维护向量库适配。

## 8.3 边界与坑

- 版本冲突：多框架共存时注意传递依赖的版本对齐。
- 抽象泄漏：组合处应通过自有接口隔离，避免两套 API 互相渗透。
- 可观测割裂：不同框架的追踪需统一到同一 OTel 后端。
- 边界：组合的复杂度应低于它带来的收益，否则不如单框架。

# 9. 建议

## 9.1 分阶段建议

| 阶段 | 建议 |
|------|------|
| 学习 | 先学原理，再学框架 |
| 选型 | 按团队技术栈与治理诉求 |
| 落地 | 主框架 + 按需补充 |
| 演进 | 保持模型层抽象，便于切换 |

## 9.2 反模式

### 9.2.1 常见反模式

| 反模式 | 后果 |
|--------|------|
| 框架 API 渗透业务 | 迁移成本剧增 |
| 多框架无边界堆叠 | 依赖地狱、调试困难 |
| 只看功能表选型 | 上线后治理缺位 |
| 跳过 PoC | 关键路径踩坑 |

### 9.2.2 正确做法

- 用自有接口封装模型、提示、工具与记忆四类抽象。
- 选型前跑通"一个真实场景"的端到端 PoC。
- 保留模型层可替换能力，框架只是加速器。

# 10. 小结

## 10.1 核心要点

- Spring AI 与 LangChain4j 是 Java 两大主线。
- 选型看团队技术栈、治理诉求与模型来源。
- 核心抽象相似，迁移成本不高。
- 框架可组合，不是互斥。
- 先懂原理，框架只是加速器。

## 10.2 一句话选型

| 若你…… | 选…… |
|---------|------|
| 深度使用 Spring 且重治理 | Spring AI |
| 要快、要组件丰富 | LangChain4j |
| 国内模型与图编排 | Spring AI Alibaba |
| 云原生 Serverless | Quarkus LangChain4j |
| 目标驱动自动规划 | Embabel |
