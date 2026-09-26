# 1. 什么是 Planner

**结论先行**：**Planner（规划器）** 是让模型自动规划"用哪些函数、按什么顺序"完成任务的组件。它把"流程决策权"从开发者手中交给大语言模型（LLM，Large Language Model）。

```
用户目标 → Planner 生成计划 → 执行计划
```

## 1.1 一句话定义与边界

Planner 的本质是"**把自然语言目标翻译成可执行的函数调用序列**"。它不负责执行，只负责产出计划；执行由 Kernel（内核）完成。

需要区分三个容易混淆的概念：

| 概念 | 职责 | 是否调用模型 |
|------|------|:------------:|
| Planner | 生成计划（函数序列或步骤） | 是 |
| Kernel | 注册函数、路由调用 | 否 |
| Function | 具体能力单元 | 否 |

边界：Planner 只处理"**做什么、按什么顺序**"，不处理"**怎么做**"。后者是函数内部实现。

## 1.2 机制拆解

### 1.2.1 计划的三个组成部分

一个计划通常包含：

1. **步骤列表**：若干有序步骤。
2. **函数绑定**：每步绑定到具体函数名与参数。
3. **数据流**：上一步输出如何成为下一步输入。

### 1.2.2 运行时序

```
┌─────────┐   goal    ┌──────────┐  prompt   ┌───────┐
│  Caller │──────────▶│ Planner  │──────────▶│  LLM  │
└─────────┘           └──────────┘◀──────────└───────┘
                            │  计划（函数序列）
                            ▼
                      ┌──────────┐
                      │  Kernel  │ 逐个执行
                      └──────────┘
                            │
                            ▼
                       最终结果
```

关键点：LLM 看到的是**函数清单（名称 + 描述 + 参数 schema）**，而不是函数实现。因此函数描述的质量直接决定计划质量。

## 1.3 最小可用示例

```java
// 注册规划器：把插件挂到 Kernel 上，让模型"看得见"可用函数
Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(weatherPlugin)
        .withPlugin(travelPlugin)
        .build();

// 自动规划（Planner 函数在 Java 版中 API 可能变动，此处为示意）
KernelFunction planner = ...;  // 规划器函数
FunctionResult result = kernel.invokeAsync(planner, Map.of(
        "goal", "帮我规划去东京的行程，先查天气")).block();
```

## 1.4 边界与坑

- 模型生成的计划可能引用不存在的函数。
- 参数类型可能不匹配（字符串 vs 数字）。
- 计划可能缺少终止条件。

## 1.5 生产建议

- 为每个函数写清晰的 `@Description`（描述）。
- 限制可用函数集合，不要暴露全部插件。
- 记录完整计划文本，便于回放。

# 2. 规划的价值

**结论先行**：Planner 的价值在于**灵活性**，代价是**可控性、可调试性与成本**。

## 2.1 手工编排 vs Planner

```
手工编排：开发者写死流程
Planner： 模型自动决定流程
```

| 维度 | 手工编排 | Planner |
|------|:--------:|:-------:|
| 灵活性 | 低 | 高 |
| 可控性 | 高 | 低 |
| 调试 | 易 | 难 |
| 成本 | 低 | 高（多次调用） |

## 2.2 价值来源分析

### 2.2.1 决策外包

开发者无法穷举所有分支时，把分支决策交给模型。例如"用户可能问天气、可能问机票、可能两者都要"。

### 2.2.2 动态组合

当函数数量增长到几十个、组合空间爆炸时，手写 `if-else` 不可维护，Planner 用语义搜索替代穷举。

### 2.2.3 冷启动

需求尚未稳定时，Planner 可以快速跑通端到端，后期再逐步固化为显式编排。

## 2.3 何时不值得用 Planner

| 场景 | 原因 |
|------|------|
| 流程固定 | 直接编排更省成本 |
| 强合规要求 | 路径必须可审计 |
| 低延迟要求 | 多次 LLM 调用太慢 |
| 函数少于 5 个 | 穷举即可 |

## 2.4 生产建议

先用手工编排跑通，确认稳定后再评估是否需要 Planner 提升灵活性。**不要一开始就上 Planner**。

## 2.5 成本模型

### 2.5.1 调用次数

| 方案 | 模型调用次数 | 说明 |
|------|:------------:|------|
| 手工编排 | 1 次（仅生成答案） | 参数由代码填 |
| 顺序规划 | 1 + N 次 | 规划 1 次 + 每步执行 |
| 步骤规划 | 1 + 2N 次 | 规划 + 每步再决策 |

# 3. 规划模式

**结论先行**：常见规划模式有三类——**顺序规划、步骤规划、手写规划**，可控性依次升高，灵活性依次下降。

## 3.1 三种模式总览

| 模式 | 说明 | 输出形态 | 可控性 |
|------|------|----------|:------:|
| 顺序规划 | 生成函数调用序列 | 函数名 + 参数 | 低 |
| 步骤规划 | 生成自然语言步骤，逐步执行 | 文本步骤 | 中 |
| 手写规划 | 预定义流程 + 模型填充参数 | 固定流程 | 高 |

## 3.2 顺序规划

### 3.2.1 机制

模型一次性输出若干函数调用，Kernel 顺序执行。适合函数间数据流简单、无分支的场景。

### 3.2.2 示例

```java
// 示意：模型输出的计划（JSON）
String plan = """
    [
      {"function": "weather.getWeather", "args": {"city": "Tokyo"}},
      {"function": "travel.buildItinerary", "args": {"city": "Tokyo"}}
    ]
    """;
```

## 3.3 步骤规划

### 3.3.1 机制

模型先输出自然语言步骤，每一步执行前再决定具体函数。适合步骤间依赖复杂、需要边执行边调整的场景。

### 3.3.2 示例

```
步骤 1：查询东京未来三天天气
步骤 2：根据天气决定室内/室外景点
步骤 3：生成行程
```

## 3.4 手写规划

### 3.4.1 机制

开发者预定义流程图（Graph，图），模型只负责**填充参数**或**选择分支**。可控性最高。

### 3.4.2 示例

```java
// 流程固定，模型只填参数
String city = model.extract(userInput, "city");
String weather = weatherFunction.invokeAsync(city).block();
String itinerary = itineraryFunction.invokeAsync(city, weather).block();
```

## 3.5 边界与坑

- 顺序规划：一步失败后整条链断裂，缺少补偿。
- 步骤规划：步骤描述歧义导致重复调用。
- 手写规划：退化为普通编排，Planner 收益有限。

## 3.6 生产建议

优先顺序规划（可解析、可缓存）；步骤规划仅用于探索性任务；手写规划作为稳定路径兜底。

## 3.7 模式选择

```
任务分支数是否可穷举？
 ├─ 是 → 手写规划
 └─ 否 → 步骤间是否有强数据依赖？
          ├─ 是 → 步骤规划
          └─ 否 → 顺序规划
```

# 4. 使用示例

**结论先行**：使用 Planner 的完整链路是——**构建 Kernel → 注册插件 → 挂载规划器 → 传入目标 → 执行计划 → 收集结果**。

## 4.1 环境准备与依赖

```xml
<!-- Maven 依赖（示意，版本以官方为准） -->
<dependency>
    <groupId>com.microsoft.semantickernel</groupId>
    <artifactId>semantickernel-api</artifactId>
    <version>1.4.0</version>
</dependency>
<dependency>
    <groupId>com.microsoft.semantickernel</groupId>
    <artifactId>semantickernel-connectors-ai-openai</artifactId>
    <version>1.4.0</version>
</dependency>
```

```yaml
# application.yml：模型配置
sk:
  openai:
    endpoint: https://api.openai.com/v1
    api-key: ${OPENAI_API_KEY}
    model: gpt-4o
```

## 4.2 注册插件与规划器

### 4.2.1 Kernel 构建

```java
Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(weatherPlugin)
        .withPlugin(travelPlugin)
        .build();
```

### 4.2.2 插件注册

```java
public class WeatherPlugin {

    @KernelFunction
    @Description("查询指定城市的当前天气")
    public String getWeather(
            @KernelFunctionParameter(name = "city", description = "城市名")
            String city) {
        return weatherClient.query(city);
    }
}

KernelPlugin plugin = KernelPluginFactory.createFromObject(
        new WeatherPlugin(), "weather");
kernel = kernel.toBuilder().withPlugin(plugin).build();
```

## 4.3 执行计划

```java
// 自动规划
KernelFunction planner = ...;  // 规划器函数
FunctionResult result = kernel.invokeAsync(planner, Map.of(
        "goal", "帮我规划去东京的行程，先查天气")).block();
System.out.println(result.getResult());
```

## 4.4 完整可运行示例

```java
public class PlannerDemo {

    public static void main(String[] args) {
        ChatCompletionService chat = OpenAIChatCompletion.builder()
                .withOpenAIAsyncClient(client)
                .withModelId("gpt-4o")
                .build();

        Kernel kernel = Kernel.builder()
                .withAIService(ChatCompletionService.class, chat)
                .withPlugin(KernelPluginFactory.createFromObject(
                        new WeatherPlugin(), "weather"))
                .withPlugin(KernelPluginFactory.createFromObject(
                        new TravelPlugin(), "travel"))
                .build();

        // 规划 + 执行（API 示意）
        String answer = kernel.invokePromptAsync(
                "目标：{{$goal}}。可用函数：weather.getWeather、"
                + "travel.buildItinerary。请输出调用序列并执行。",
                KernelArguments.builder()
                        .withVariable("goal", "规划东京行程")
                        .build())
                .block()
                .getResult();

        System.out.println(answer);
    }
}
```

## 4.5 边界与坑

- Java 版 Planner 支持不如 .NET 完整，部分规划器缺失。
- `invokePromptAsync` 不自动执行函数，需开启自动函数调用。
- 参数注入需与函数签名严格对齐。

## 4.6 生产建议

- 用 `OpenTelemetry`（开放遥测）包裹规划与执行两阶段。
- 对计划做 schema 校验后再执行。
- 缓存常见目标的计划。

# 5. 规划的限制

**结论先行**：Planner 的四大问题是——**计划不可执行、调用错误函数、成本高、难调试**。

## 5.1 四类典型失效

```
Planner 的问题：
- 生成的计划可能不可执行
- 可能调用错误的函数
- 成本高（多次 LLM 调用）
- 难以调试
```

| 失效类型 | 现象 | 根因 |
|----------|------|------|
| 不可执行 | 引用了不存在的函数 | 函数清单过期 |
| 调用错误 | 语义相近函数选错 | 描述模糊 |
| 成本高 | 多次往返模型 | 无计划缓存 |
| 难调试 | 无法复现路径 | 无完整日志 |

## 5.2 失效根因分析

### 5.2.1 函数签名与语义鸿沟

函数名 `getWeather` 与用户说的"看看冷不冷"之间存在语义鸿沟。模型依赖描述做匹配，描述差则匹配错。

### 5.2.2 状态不可见

Planner 看不到运行时状态（用户权限、数据可用性），因此计划可能在执行阶段才失败。

## 5.3 缓解手段

| 手段 | 说明 |
|------|------|
| 函数白名单 | 只暴露必要函数 |
| 描述规范 | 统一模板：用途 + 输入 + 输出 |
| 计划校验 | 执行前校验函数与参数 |
| 步数上限 | 防止无限规划 |
| 回退编排 | 规划失败走固定流程 |

## 5.4 生产建议

> 现代实践**更倾向显式编排**（如 Graph），而非全自动 Planner。

把 Planner 限制在"**低风险、探索性、非关键路径**"的场景。

## 5.5 反例：一次失败的计划

```
目标：帮我订明天去东京的机票

模型生成计划：
  1. flight.search(city=Tokyo, date=tomorrow)
  2. flight.book(...)         ← 缺少用户身份与支付授权
  3. hotel.book(...)          ← 用户并未要求订酒店

问题：
  - 越权：第 2 步涉及支付
  - 越界：第 3 步超出目标范围
```

修复：把 `book` 类函数从白名单移除，只暴露 `search`；要求计划必须通过校验器。

## 5.6 计划校验器

```java
// 伪代码：执行前校验计划
public void validate(List<Step> plan) {
    for (Step s : plan) {
        if (!allowedFunctions.contains(s.function())) {
            throw new PlanRejectedException("函数不在白名单：" + s.function());
        }
        if (plan.size() > MAX_STEPS) {
            throw new PlanRejectedException("步数超限");
        }
    }
}
```

# 6. 规划 vs 显式编排

**结论先行**：Planner 由模型决策、可预测性低；显式编排由开发者决策、可预测性高。

## 6.1 对比表

| 维度 | Planner | 显式编排（Graph） |
|------|---------|-------------------|
| 决策者 | 模型 | 开发者 |
| 可预测性 | 低 | 高 |
| 调试 | 难 | 易 |
| 成本 | 高 | 低 |
| 适用 | 探索性任务 | 稳定流程 |

## 6.2 混合架构：编排为主 + 规划为辅

### 6.2.1 结构

```
┌──────────────────────────────┐
│  显式编排（主流程，Graph）    │
│  ┌────────┐   ┌────────────┐ │
│  │ 固定步 │──▶│ Planner 子图│ │
│  └────────┘   └────────────┘ │
└──────────────────────────────┘
```

### 6.2.2 示例

```java
// 主流程固定，只在"行程细化"这一步交给 Planner
String weather = weatherFn.invokeAsync(city).block();     // 固定
String plan = plannerFn.invokeAsync(goal).block();        // 规划
String itinerary = itineraryFn.invokeAsync(plan).block(); // 固定
```

## 6.3 选型决策树

```
流程是否稳定？
 ├─ 是 → 显式编排
 └─ 否 → 是否有强合规/低延迟要求？
          ├─ 是 → 显式编排 + 人工确认
          └─ 否 → 可考虑 Planner
```

## 6.4 生产建议

> 生产推荐：**显式编排为主，Planner 为辅**。

用编排保证关键路径，用 Planner 处理长尾分支。

## 6.5 混合架构的失败模式

| 失败模式 | 现象 | 对策 |
|----------|------|------|
| 规划子图逃逸 | Planner 调用了主流程函数 | 子图隔离函数白名单 |
| 状态不一致 | 子图改了主流程状态 | 明确输入输出契约 |
| 超时传导 | 子图超时拖垮主流程 | 子图独立超时 |

# 7. 生产注意事项

**结论先行**：Planner 上生产的关键是**限权、限步、限成本、留痕**。

## 7.1 风险清单

| 风险 | 建议 |
|------|------|
| 计划不可执行 | 提供函数清单与约束 |
| 无限循环 | 步数上限 |
| 成本高 | 限制使用场景 |
| 难调试 | 记录完整规划过程 |
| 无鉴权 | 函数级权限 |

## 7.2 治理配置

### 7.2.1 步数与预算

```java
// 伪代码：规划器执行约束
PlannerOptions options = PlannerOptions.builder()
        .maxSteps(6)                 // 最多 6 步
        .maxTokens(4000)             // 预算上限
        .timeout(Duration.ofSeconds(30))
        .build();
```

### 7.2.2 权限与审计

```java
// 伪代码：函数级权限校验
public FunctionResult invoke(FunctionInvocationContext ctx) {
    String fn = ctx.getFunction().getName();
    if (!authz.canInvoke(currentUser(), fn)) {
        throw new AccessDeniedException("无权限调用：" + fn);
    }
    audit.log(currentUser(), fn, ctx.getArguments());
    return ctx.next();
}
```

## 7.3 可观测

```java
// 伪代码：记录完整规划过程
logger.info("plan.raw={}", planText);
meterRegistry.counter("planner.invocations").increment();
meterRegistry.timer("planner.duration").record(elapsed);
```

## 7.4 生产建议

- 计划文本必须入库，支持回放。
- 给每个函数设置超时与重试。
- 对高风险函数加二次确认。

# 8. 小结

- Planner 让模型自动规划函数调用序列。
- 灵活但可控性低、成本高、难调试。
- 现代实践更倾向显式编排（Graph）。
- 生产用"显式编排为主 + Planner 为辅"。
- 使用 Planner 需限制步数与成本。
