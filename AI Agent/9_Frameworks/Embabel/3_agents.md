# 1. Agent 的组成

结论先行：在 Embabel 中，Agent 不是"一个会聊天的对象"，而是**一组目标、动作与条件的集合**。它复用 GOAP（Goal-Oriented Action Planning，目标导向动作规划）规划器，把"做什么"交给目标、把"能做什么"交给动作、把"何时能做"交给条件。

本节拆解 Agent 的三要素及其与规划器的关系。

## 1.1 三要素

```
Embabel Agent = 目标 + 动作 + 条件
  ├── 目标：要达成什么
  ├── 动作：能做什么
  └── 条件：动作何时可执行
```

### 1.1.1 目标

目标是 Agent 要达成的终态，例如 `planReady = true`。Agent 可以有多个目标，规划器按优先级或调用方指定选择。

| 目标来源 | 说明 |
|----------|------|
| 调用方指定 | 运行时传入 `Goal` |
| Agent 默认 | 注解声明默认目标 |
| 上层规划 | 由父 Agent 作为子目标下派 |

```java
Goal goal = Goal.builder()
        .condition("planReady", true)
        .priority(1)
        .build();
```

### 1.1.2 动作

动作是 Agent 能力的原子单位，用前置条件与效果声明，方法体实现真实逻辑。动作可以调用服务、LLM（Large Language Model，大语言模型）或外部工具。

### 1.1.3 条件

条件是动作可执行的前提。规划器把条件当作"门禁"，只有满足条件的动作才会被纳入路径。

## 1.2 与 GOAP 的关系

### 1.2.1 复用规划器

Agent 本身不实现搜索算法，它把动作与目标注册到规划器，由规划器搜索路径。因此第 2 篇（GOAP 规划）的结论在这里全部适用。

### 1.2.2 Agent 是动作的容器

```
Agent
  ├── Action A (Pre, Effect)
  ├── Action B (Pre, Effect)
  └── Action C (Pre, Effect)
        │
        ▼
     Planner 搜索
        │
        ▼
    动作序列 → Executor 执行
```

## 1.3 生命周期

| 阶段 | 动作 |
|------|------|
| 注册 | 扫描 `@Agent` / `@Action` |
| 规划 | 根据目标搜索动作序列 |
| 执行 | 依次执行动作 |
| 收尾 | 产出结果、清理上下文 |

# 2. 定义 Agent

结论先行：用 `@Agent` 标注类，用 `@Action` 标注方法，框架自动注册动作。声明与实现必须一致。

## 2.1 @Agent 注解

### 2.1.1 注解参数

| 参数 | 类型 | 含义 |
|------|------|------|
| `description` | String | Agent 用途描述，供文档与路由 |
| `name` | String | 可选，Agent 名称 |

```yaml
embabel:
  agent:
    scan-packages:
      - com.example.agents
    default-goal: planReady
```

### 2.1.2 描述与命名

描述应说明"这个 Agent 解决什么问题"，便于多 Agent 场景下由上层选择合适 Agent。

## 2.2 完整示例

```java
@Agent(description = "旅行规划助手")
public class TravelAgent {

    @Action(
            name = "查询天气",
            preconditions = {},
            effects = {"hasWeather"})
    public void checkWeather(AgentContext ctx, String city) {
        ctx.setState("weather", weatherService.get(city));
        ctx.setState("hasWeather", true);
    }

    @Action(
            name = "生成行程",
            preconditions = {"hasWeather"},
            effects = {"planReady"})
    public void generatePlan(AgentContext ctx) {
        String plan = llm.generate("""
                基于天气 %s 生成旅行行程
                """.formatted(ctx.getState("weather")));
        ctx.setState("plan", plan);
        ctx.setState("planReady", true);
    }
}
```

### 2.2.1 动作集合

上面的 Agent 提供两个动作：`查询天气`（无条件）与 `生成行程`（需要 `hasWeather`）。规划器据此可推出 `[查询天气, 生成行程]`。

### 2.2.2 状态流转

| 步骤 | 状态变化 |
|------|----------|
| 初始 | `{}` |
| 查询天气 | `{hasWeather: true, weather: ...}` |
| 生成行程 | `{planReady: true, plan: ...}` |

## 2.3 边界与反例

反例：把所有逻辑塞进一个"万能动作"，前置条件为空、效果为 `done`。这样规划器无搜索空间，退化为直接调用，失去 GOAP 的意义。

生产建议：动作粒度以"可被前置条件区分"为准，既不过粗也不过细。

# 3. 条件

结论先行：条件分为前置条件与效果。前置条件决定"何时能执行"，效果决定"执行后状态如何变化"，二者共同构成规划器的推演依据。

## 3.1 前置条件（Precondition）

### 3.1.1 语义

前置条件是动作执行的前提，必须是确定状态才能执行；规划器据此搜索路径。

```
Precondition（前置条件）：动作执行的前提
  - 必须是确定状态才能执行
  - 规划器据此搜索路径
```

### 3.1.2 常见写法

| 写法 | 含义 |
|------|------|
| `{}` | 无条件，任意时刻可执行 |
| `{"hasWeather"}` | 单条件 |
| `{"hasWeather", "hasBudget"}` | 多条件，全部满足 |

## 3.2 效果（Effect）

### 3.2.1 声明式效果

效果是声明式的：规划器不理解动作内部逻辑，只信任你声明的状态变化。

```
Effect（效果）：动作执行后的状态变化
  - 声明式，规划器据此推演
```

### 3.2.2 效果一致性

效果必须与实际 `setState` 完全一致。若声明了 `planReady` 却没写入，规划器会规划出错误序列。

| 情况 | 后果 |
|------|------|
| 声明且写入 | 正常 |
| 声明未写入 | 后续动作读到空值 |
| 写入未声明 | 规划器认为状态未变 |

## 3.3 边界与反例

反例：效果里写"所有字段都变"，导致规划器认为一次动作即可满足任意目标，路径失真。正确做法是精确声明本动作真正改变的字段。

# 4. 动作类型

结论先行：动作按确定性分为原生动作、LLM 动作与工具动作。选择哪种取决于该步骤是"确定的"还是"需要判断的"。

## 4.1 原生动作

纯 Java 逻辑，确定性、可测试，适合计算、校验、格式化。

```java
@Action(name = "校验预算", preconditions = {"hasBudgetInput"}, effects = {"budgetValid"})
public void validateBudget(AgentContext ctx, BigDecimal budget) {
    if (budget.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("budget must be positive");
    }
    ctx.setState("budgetValid", true);
}
```

## 4.2 LLM 动作

调用大模型，非确定性，适合理解、判断、生成。需处理超时、重试与输出校验。

```java
@Action(name = "提炼要点", preconditions = {"retrieved"}, effects = {"summarized"})
public void summarize(AgentContext ctx) {
    String text = (String) ctx.getState("context");
    ctx.setState("summary", llm.generate("用三点总结：" + text));
    ctx.setState("summarized", true);
}
```

## 4.3 工具动作

调用外部 API 或数据库，确定性与外部系统一致，需处理失败与幂等。

```java
@Action(name = "检索知识", preconditions = {}, effects = {"retrieved"})
public void retrieve(AgentContext ctx, String question) {
    List<Document> docs = vectorStore.similaritySearch(question, 5);
    ctx.setState("context", join(docs));
    ctx.setState("retrieved", true);
}
```

## 4.4 类型选择与对比

| 类型 | 说明 | 确定性 | 典型用途 |
|------|------|--------|----------|
| 原生动作 | Java 代码 | 确定 | 计算、校验 |
| LLM 动作 | 调用模型 | 非确定 | 分析、生成 |
| 工具动作 | 调用外部 API | 依赖外部 | 检索、集成 |

```
原生动作 ── 确定逻辑
LLM 动作 ── 判断/生成
工具动作 ── 外部世界
```

# 5. 与 LLM 结合

结论先行：LLM 用于"需要判断"的步骤，原生代码用于"确定"的步骤。把 LLM 包在动作里，使其结果也进入状态，从而可被后续动作与规划器感知。

## 5.1 何时用 LLM

| 步骤性质 | 选择 |
|----------|------|
| 规则明确、可枚举 | 原生代码 |
| 需要语义理解 | LLM |
| 需要生成文本 | LLM |
| 需要严格一致 | 原生代码 |

## 5.2 代码示例

### 5.2.1 Spring AI ChatClient

```java
@Action(name = "分析需求", preconditions = {}, effects = {"analyzed"})
public void analyze(AgentContext ctx, String request) {
    String analysis = chatClient.prompt()
            .user("分析这个需求的要点：" + request)
            .call().content();

    ctx.setState("analysis", analysis);
    ctx.setState("analyzed", true);
}
```

### 5.2.2 结构化输出

直接返回自由文本不利于后续动作消费。推荐让模型输出结构化结果：

```java
record Analysis(String topic, List<String> points) {}

@Action(name = "分析需求", preconditions = {}, effects = {"analyzed"})
public void analyzeStructured(AgentContext ctx, String request) {
    Analysis a = chatClient.prompt()
            .user("以 JSON 分析需求要点：" + request)
            .call()
            .entity(Analysis.class);

    ctx.setState("analysis", a);
    ctx.setState("analyzed", true);
}
```

> LLM 用于"需要判断"的步骤，原生代码用于"确定"的步骤。

## 5.3 非确定性的处理

| 风险 | 处理 |
|------|------|
| 输出格式错 | 结构化输出 + 校验 |
| 超时 | 超时 + 重试 |
| 幻觉 | 检索增强 + 约束 |
| 成本高 | 缓存 + 降级模型 |

## 5.4 边界与反例

反例：让 LLM 决定"是否调用某个动作"。LLM 的决策不可靠，应由规划器基于前置条件决定，LLM 只负责动作内部的内容生成。

# 6. Agent 执行

结论先行：执行分三步——定义目标、构造上下文、调用执行入口。执行器自动完成"规划 + 执行"。

## 6.1 目标与上下文

### 6.1.1 定义目标

```java
// 定义目标
Goal goal = Goal.of("planReady", true);
```

### 6.1.2 构造上下文

```java
// 创建 Agent 上下文
AgentContext ctx = AgentContext.builder()
        .withInput("city", "东京")
        .build();
```

## 6.2 执行入口

### 6.2.1 agentRunner.run

```java
// 执行（自动规划 + 执行）
AgentResult result = agentRunner.run(travelAgent, goal, ctx);
```

```yaml
embabel:
  runner:
    max-steps: 20            # 单次执行最大步数
    action-timeout-ms: 5000  # 单动作超时
    trace-enabled: true      # 打开执行轨迹
```

### 6.2.2 结果处理

| 结果 | 处理 |
|------|------|
| 成功 | 取状态中的产出 |
| 规划失败 | 检查动作/条件是否覆盖目标 |
| 执行失败 | 重试或重规划 |

## 6.3 端到端示例

```java
public String planTrip(String city) {
    TravelAgent agent = new TravelAgent(weatherService, llm);
    Goal goal = Goal.of("planReady", true);
    AgentContext ctx = AgentContext.builder()
            .withInput("city", city)
            .build();

    AgentResult result = agentRunner.run(agent, goal, ctx);
    if (!result.success()) {
        throw new IllegalStateException("planning failed: " + result.reason());
    }
    return (String) result.state().get("plan");
}
```

## 6.4 生产建议

- 每次执行生成唯一 traceId，贯穿规划与执行。
- 限制单次执行的步数与耗时。
- 结果对象不可变，避免外部篡改状态。

# 7. 生产注意事项

结论先行：Agent 的风险集中在"动作设计"与"LLM 非确定性"两点。前者靠分组与测试，后者靠重试与降级。

## 7.1 动作管理

### 7.1.1 动作分组

按领域把动作拆到不同 Agent，避免单个 Agent 动作过多导致搜索空间膨胀。

### 7.1.2 数量限制

| 动作数 | 影响 |
|--------|------|
| 少量 | 搜索快、易调试 |
| 过多 | 组合爆炸、路径难预测 |

## 7.2 前置条件测试

对每个动作的前置条件与效果编写单元测试，确保声明与实现一致。

```java
@Test
void generatePlanRequiresWeather() {
    Action a = Action.of(TravelAgent.class, "generatePlan");
    assertTrue(a.preconditions().contains("hasWeather"));
}
```

## 7.3 LLM 失败处理

| 失败 | 策略 |
|------|------|
| 超时 | 重试 + 备用模型 |
| 格式错 | 重新提示 + 校验 |
| 不可用 | 降级为规则逻辑 |

## 7.4 可观测与状态管理

记录每步动作、耗时与状态变化；状态只放必要字段，避免膨胀；敏感字段脱敏后入日志。

```java
log.info("traceId={} agent={} action={} state={}",
        traceId, agentName, action.name(), ctx.snapshot());
```

## 7.5 安全

动作级鉴权、输入校验、提示注入（Prompt Injection）防御缺一不可。

| 风险 | 建议 |
|------|------|
| 动作过多 | 分组、限制数量 |
| 前置条件错误 | 单元测试 |
| LLM 动作失败 | 重试 + 降级 |
| 无 Trace | 记录每步 |
| 状态膨胀 | 只放必要字段 |

# 8. 小结

## 8.1 核心要点

- Agent = 目标 + 动作 + 条件。
- 动作用前置条件与效果声明，声明必须与实现一致。
- 区分原生动作、LLM 动作与工具动作。
- LLM 处理判断，代码处理确定性逻辑。

## 8.2 落地清单

- 测试前置条件、限制动作数、记录 Trace。
- 执行分三步：定义目标、构造上下文、调用入口。
- 对 LLM 动作做重试、降级与结构化输出校验。
