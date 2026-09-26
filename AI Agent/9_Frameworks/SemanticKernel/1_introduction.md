# 1. 定位

Semantic Kernel（简称 SK，语义内核）是微软推出的 AI 应用开发框架，支持 .NET、Python、Java 多语言。它的核心命题是：**如何把大语言模型（LLM，Large Language Model）的非确定性能力与传统代码的确定性能力安全地编排在一起**。

## 1.1 什么是 Semantic Kernel

### 1.1.1 定义与目标

Semantic Kernel 是一个面向生产应用的 SDK，而非简单的模型调用封装。它提供了统一的服务抽象、函数抽象、记忆抽象与规划能力，使开发者可以用一致的方式组合模型与代码。

```
核心目标：把 LLM 能力与传统代码（函数）编排在一起
```

它解决三类问题：统一不同模型供应商的接入差异；把业务函数暴露给模型调用；在模型与代码之间做可靠的流程编排。与只做 API 封装的客户端库相比，SK 的差异在于它提供了**编排所需的完整抽象**。

### 1.1.2 多语言支持

SK 官方支持三种语言运行时，各自独立演进。

| 语言 | 成熟度 | 主要用户 | 备注 |
|------|--------|----------|------|
| .NET / C# | 最高 | 微软生态 | 功能最完整 |
| Python | 高 | 数据与 AI 团队 | 生态丰富 |
| Java | 相对有限 | 企业 Java 团队 | 能力覆盖较少 |

三者的抽象概念一致（Kernel、Plugin、Function、Memory），但 API 细节与功能覆盖存在差异，跨语言迁移时不能假设行为完全一致。

## 1.2 设计哲学

### 1.2.1 编排优先

SK 认为 AI 应用的难点不在单次调用，而在**多步骤编排**：模型需要调用工具、需要记忆、需要根据中间结果决定下一步。因此它把 Kernel 设计为中央容器，把一切能力注册为可被编排的函数。

```
用户请求
   │
   ▼
Kernel（容器）
   ├── 选择服务（模型）
   ├── 调用函数（原生/语义）
   ├── 读写记忆
   └── 执行过滤器
```

### 1.2.2 与提示工程的关系

SK 不替代提示工程，而是把提示词工程化为可管理的资产：语义函数把 Prompt 模板与参数绑定，使提示词可版本化、可测试、可复用，而非散落在代码字符串里。

### 1.2.3 与 RAG 的关系

检索增强生成（RAG，Retrieval-Augmented Generation）在 SK 中体现为记忆与函数编排的组合：检索作为原生函数或记忆插件，把召回内容注入 Prompt，再由语义函数生成答案。

## 1.3 依赖与快速上手

### 1.3.1 Maven 依赖

Java 侧通过 Maven 引入 API 与具体连接器，版本必须锁定。

```yaml
#伪代码：Maven 坐标示意
dependencies:
  - com.microsoft.semantickernel:semantickernel-api:1.x.y
  - com.microsoft.semantickernel:semantickernel-connectors-ai-openai:1.x.y
```

### 1.3.2 最小可运行示例

一个最小示例只需构建 Kernel、注册服务与插件，然后调用函数。

```java
ChatCompletionService chatService = OpenAIChatCompletion.builder()
        .withModelId("gpt-4o-mini")
        .withOpenAIAsyncClient(client)
        .build();

Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .build();
```

## 1.4 生产建议

把 SK 视为编排层而非模型层。模型接入细节应封装在服务抽象之后，避免业务代码直接依赖某家 SDK，从而保留替换空间。

## 1.5 术语表

| 术语 | 英文 | 含义 |
|------|------|------|
| 语义内核 | Semantic Kernel | 框架名 |
| 内核 | Kernel | 中央容器 |
| 插件 | Plugin | 函数集合 |
| 原生函数 | Native Function | 确定性代码 |
| 语义函数 | Semantic Function | Prompt 模板 |
| 规划器 | Planner | 自动编排 |
| 记忆 | Memory | 短期/长期上下文 |
| 连接器 | Connector | 模型/存储适配 |
| 函数调用 | Function Calling | 模型触发函数 |
| 过滤器 | Filter | 环绕拦截器 |

# 2. 核心特色

SK 的竞争力来自一套相互配合的核心概念，理解这些概念是使用框架的前提。

## 2.1 六大特色

### 2.1.1 特色总览

| 特色 | 说明 |
|------|------|
| Kernel | 中央容器，管理服务与插件 |
| Plugin | 函数集合（原生 + 语义） |
| Function | 原生函数 / 语义函数 |
| Planner | 自动规划与编排 |
| Memory | 记忆与向量存储 |
| 多语言 | .NET / Python / Java |

## 2.2 特色详解

### 2.2.1 Kernel 与 Plugin

Kernel 是依赖注入容器与执行入口，Plugin 是被注册进 Kernel 的函数集合。二者关系类似 Spring 的 `ApplicationContext` 与 `@Component`：Kernel 负责装配与解析，Plugin 提供能力。

### 2.2.2 Planner 与 Memory

Planner 根据用户目标自动选择并串联函数，适合流程不完全确定的场景；Memory 提供短期对话与长期向量检索能力，使模型具备上下文。二者都依赖 Kernel 才能工作。

### 2.2.3 Function 与 Filter

Function 是执行单元，Filter 是环绕执行的拦截器。Filter 使日志、鉴权、限流等横切逻辑可以统一实现，而不必侵入每个函数。

## 2.3 概念关系图

```
          ┌───────────────────────┐
          │        Kernel         │
          │  ┌─────────────────┐  │
          │  │ AI 服务（模型）  │  │
          │  ├─────────────────┤  │
          │  │ Plugin A        │  │
          │  │  ├─ Native Fn   │  │
          │  │  └─ Semantic Fn │  │
          │  ├─────────────────┤  │
          │  │ Memory          │  │
          │  ├─────────────────┤  │
          │  │ Filter          │  │
          │  └─────────────────┘  │
          └───────────────────────┘
```

## 2.4 生产建议

Planner 自动编排虽然灵活，但可预测性差、成本不可控，生产环境建议**优先使用显式编排**（固定流程 + 函数调用），仅在流程确实无法预先枚举时才启用 Planner。

# 3. 两种函数

SK 的函数分两类，理解区别是正确使用框架的关键：原生函数是确定性代码，语义函数由 LLM 执行、结果非确定。

## 3.1 原生函数

### 3.1.1 定义与示例

原生函数（Native Function）是普通 Java 方法，通过注解暴露给 Kernel。

```java
// 原生函数
@DefineKernelFunction(name = "getWeather", description = "查询天气")
public String getWeather(String city) {
    return city + "：晴，25℃";
}
```

它的执行不消耗 token，结果可预测，适合数据库查询、计算、外部 API 调用等确定性逻辑。

### 3.1.2 参数描述

参数通过注解描述，描述文本会进入函数 schema 供模型理解，因此必须准确。

```java
@DefineKernelFunction(name = "getWeather", description = "查询指定城市的天气")
public String getWeather(
        @KernelFunctionParameter(name = "city", description = "城市名，如北京")
        String city) {
    return city + "：晴，25℃";
}
```

## 3.2 语义函数

### 3.2.1 定义与示例

语义函数（Semantic Function）是 Prompt 模板加参数，由 LLM 执行，结果非确定。

```java
// 语义函数（用 Prompt 模板）
@DefineKernelFunction(name = "summarize", description = "总结文本")
public String summarize(String text) {
    return "请总结：" + text;   // 实际由 LLM 执行
}
```

同样的输入可能产生不同输出，因此语义函数不适合承担需要严格一致性的逻辑。

### 3.2.2 结构化输出

若语义函数需要结构化结果，应在 Prompt 中约束格式并在调用后解析校验，解析失败可重试。

```java
// 伪代码：约束 JSON 输出并解析
String prompt = "以 JSON 输出，字段为 title、points(数组)：\n" + text;
// 解析失败 -> 重新调用或降级到纯文本
```

## 3.3 对比与选择

### 3.3.1 对比表

| 维度 | 原生函数 | 语义函数 |
|------|----------|----------|
| 执行者 | JVM 代码 | LLM |
| 确定性 | 确定 | 非确定 |
| 成本 | 无 token | 消耗 token |
| 延迟 | 毫秒级 | 秒级 |
| 适合 | 查询、计算、副作用 | 生成、总结、改写 |

### 3.3.2 边界与反例

常见反例是让语义函数做金额计算或状态变更，这会因非确定性引入难以复现的 bug。正确做法是用原生函数做确定性部分，语义函数只负责理解与生成，二者通过编排组合。

## 3.4 函数调用流程

模型决定调用哪个函数，Kernel 负责解析参数并执行，再把结果回填给模型。

```
用户问题
   │
   ▼
LLM 判断需要函数 ─▶ 生成函数名 + 参数
   │
   ▼
Kernel 定位函数 ─▶ 执行原生/语义函数
   │
   ▼
结果回填 LLM ─▶ 生成最终回答
```

## 3.5 生产建议

为语义函数设置输出校验（如结构化解析 + 重试），不要假设 LLM 一定返回合法格式；为原生函数设置超时与幂等，因为模型可能在多轮中被重复触发。语义函数与函数调用均可能失败，需区分可重试与不可重试错误：超时与限流可退避重试，参数错误与鉴权失败则不应重试。

# 4. Kernel

Kernel 是 SK 的中枢，承担服务注册、插件管理与调用入口三项职责。

## 4.1 Kernel 的构成

### 4.1.1 结构

```
Kernel = 容器
  ├── AI 服务（模型）
  ├── 插件（函数集合）
  ├── 记忆
  └── 过滤器
```

### 4.1.2 构建代码

Kernel 通过 Builder 构建，注册 AI 服务与插件。

```java
Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(weatherPlugin)
        .withPlugin(summarizePlugin)
        .build();
```

## 4.2 服务注册与解析

### 4.2.1 注册 AI 服务

AI 服务按接口类型注册，同一接口可注册多个实现并通过名称选择，便于多模型并存。

```java
Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, primaryService, "primary")
        .withAIService(ChatCompletionService.class, backupService, "backup")
        .build();

ChatCompletionService service = kernel.getService(ChatCompletionService.class);
```

## 4.3 插件注册方式

### 4.3.1 从对象创建

通过工厂把带注解的对象转换为插件，并指定插件名。

```java
KernelPlugin plugin = KernelPluginFactory.createFromObject(
        new WeatherPlugin(), "weather");

Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(plugin)
        .build();
```

## 4.4 过滤器

### 4.4.1 函数调用过滤器

过滤器类似 Spring AI 的 Advisor，可环绕函数执行做日志与鉴权。

```java
public class LoggingFilter implements FunctionInvocationFilter {

    @Override
    public FunctionResult invoke(FunctionInvocationContext context) {
        log.info("调用函数: {}", context.getFunction().getName());
        FunctionResult result = context.next();
        log.info("结果: {}", result.getResult());
        return result;
    }
}
```

## 4.5 生产建议

Kernel 建议以单例方式创建并在应用启动时完成注册，避免每次请求重建容器；服务实现应支持热替换，以便在不重启的情况下切换模型。若插件携带请求级上下文（如租户信息），则应为每个请求派生独立 Kernel 或通过参数传递上下文，避免全局可变状态。

# 5. 适用场景

选择框架应先看场景与团队技术栈，而非追逐概念。

## 5.1 场景匹配

### 5.1.1 适用表

| 场景 | 说明 |
|------|------|
| .NET / C# 生态 | 首选 |
| 需要多语言统一 | SK 支持多语言 |
| 已有微软技术栈 | 集成好 |
| 需要 Planner | 内置 |

## 5.2 不适用场景

### 5.2.1 反例

若团队是纯 Java 且无微软依赖，SK 的 Java 版功能有限、生态偏微软，此时 Spring AI 或 LangChain4j 通常更合适。若应用只是简单的一次性模型调用，引入完整框架反而增加复杂度。

## 5.3 与团队匹配

### 5.3.1 匹配维度

| 团队特征 | 推荐 |
|----------|------|
| 已有 .NET 服务 | SK |
| Spring 全家桶 | Spring AI |
| 追求轻量灵活 | LangChain4j |
| 需要跨语言共享逻辑 | SK |

## 5.4 生产建议

选型时做一次技术验证：用真实业务场景分别实现原型，比较功能覆盖、文档质量、社区活跃度与长期维护成本，再决定。

# 6. 与 Spring AI / LangChain4j 对比

三大 Java 友好框架各有侧重，理解差异有助于快速选型。

## 6.1 对比表

### 6.1.1 多维对比

| 维度 | SK | Spring AI | LangChain4j |
|------|----|-----------| ------------|
| 出身 | 微软 | Spring | 社区 |
| 主语言 | .NET/Python/Java | Java | Java |
| 核心抽象 | Kernel/Plugin | ChatClient | AiServices |
| Spring 集成 | 可集成 | 原生 | Starter |
| 生态 | 微软 | Spring | Java LLM |

> Java 团队若非微软生态，通常选 Spring AI 或 LangChain4j。

## 6.2 选型建议

### 6.2.1 决策路径

```
是否已有微软/.NET 技术栈？
  ├── 是 ─▶ Semantic Kernel
  └── 否
        └── 是否重度使用 Spring 生态？
              ├── 是 ─▶ Spring AI
              └── 否 ─▶ LangChain4j
```

## 6.3 代码风格对比

### 6.3.1 调用风格

三者调用模型的方式接近，差异主要在抽象命名与配置方式。

```java
// SK：Kernel 取服务后调用
ChatCompletionService service = kernel.getService(ChatCompletionService.class);

// Spring AI：ChatClient 链式调用
String answer = chatClient.prompt().user("你好").call().content();
```

## 6.4 生产建议

框架可组合使用，但应避免同时引入多套抽象造成维护负担。若必须集成，建议以一套为主、另一套通过适配层隔离。迁移成本方面，SK 与 Spring AI 的概念基本可映射，双向迁移均需适配层，不迁移则只需锁定版本。

# 7. Java 侧现状

Java 开发者使用 SK 前，必须了解其 Java 版与 .NET/Python 版的差距。

## 7.1 现状说明

### 7.1.1 功能差异

```
SK 的 Java 版本相对 .NET/Python 版本功能较少
Java 团队主流仍选 Spring AI / LangChain4j
```

Planner、部分连接器与高级记忆能力在 Java 版中可能尚未完全对齐，使用时需查阅对应版本文档确认。

## 7.2 社区与版本

### 7.2.1 版本策略

SK 迭代较快，API 可能发生不兼容变更。生产项目应锁定版本并在升级前阅读变更日志。

```yaml
#伪代码：锁定依赖版本
dependencies:
  com.microsoft.semantickernel:semantickernel-api:1.x.y
```

## 7.3 Java 版能力矩阵

### 7.3.1 能力对照

| 能力 | .NET/Python | Java |
|------|-------------|------|
| 原生函数 | 完整 | 支持 |
| 语义函数 | 完整 | 支持 |
| Planner | 完整 | 部分 |
| 向量记忆 | 完整 | 部分 |
| 连接器数量 | 多 | 少 |

## 7.4 生产建议

为 Java 版缺失的能力准备降级方案（自行实现或改用其他框架的对应模块），不要把关键路径完全绑定在尚未稳定的特性上。

# 8. 学习路径

建议按依赖顺序自底向上学习，先掌握函数与容器，再进入规划与记忆。

## 8.1 学习步骤

### 8.1.1 五步路径

```
① 理解 Kernel / Plugin / Function
② 编写原生函数与语义函数
③ 使用 Planner 自动编排
④ 集成记忆与向量存储
⑤ 企业集成
```

## 8.2 实践建议

### 8.2.1 练习项目

可从一个天气助手开始：注册天气插件，让模型通过函数调用获取天气并生成自然语言回答，再逐步加入记忆与检索。

## 8.3 常见误区

### 8.3.1 误区清单

| 误区 | 后果 | 纠正 |
|------|------|------|
| 把语义函数当确定性函数 | 结果不稳定 | 用原生函数 |
| 过度依赖 Planner | 成本与延迟不可控 | 显式编排 |
| 忽略函数描述 | 模型选错函数 | 精写描述 |
| 不锁版本 | 升级破坏 | 锁定版本 |

## 8.4 生产建议

学习阶段用官方示例即可，但进入生产前需补充可观测、超时、限流与密钥管理等工程能力，这些通常不在入门教程覆盖范围内。学习资源以官方仓库示例、API 文档与变更日志为主，变更日志是升级评估的必备参考。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| Java 版功能有限 | 评估后再选 |
| 生态偏微软 | 结合团队技术栈 |
| 版本迭代 | 锁定版本 |
| 与 Spring 集成 | 用适配层 |

## 9.2 选型检查清单

### 9.2.1 检查项

选型前确认：所需能力在 Java 版是否可用、是否有活跃维护、许可证是否满足、与现有依赖是否冲突、能否通过适配层隔离。

### 9.2.2 迁移预案

若未来需要替换框架，适配层应把 SK 的 Kernel 与 Function 抽象限制在少数类中，业务代码只依赖自定义接口。

## 9.3 生产建议

把版本锁定、适配层、功能验证三项作为引入 SK 的前置条件，可显著降低长期维护风险。

# 10. 小结

## 10.1 要点回顾

- Semantic Kernel 是微软的多语言 AI 框架。
- 核心：Kernel（容器）+ Plugin（函数集合）。
- 区分原生函数与语义函数。
- Java 版功能相对有限，Java 团队多选 Spring AI/LangChain4j。
- 适合微软技术栈团队。

## 10.2 演进建议

若团队已投入 SK，可先从原生函数与显式编排入手，逐步引入记忆与受控的规划能力，同时用适配层保留未来切换框架的余地。
