# 1. GOAP 驱动的多 Agent

Embabel 是一个面向 JVM 的 Agent 框架，其最大特点是采用 **GOAP**（Goal-Oriented Action Planning，面向目标的动作规划）来组织多 Agent 协作。与"由主管显式分配任务"的传统模式不同，Embabel 让开发者只声明"每个 Agent 能做什么、需要什么前置条件、会产生什么效果"，由规划器在运行时自动搜索出一条从当前状态到目标的动作序列。

```
传统多 Agent：主管显式分配任务
Embabel：     用 GOAP 从目标自动规划，选择合适的 Agent/动作
```

## 1.1 什么是 GOAP

结论先行：GOAP 把"怎么做"从代码里抽出来，交给规划器，代码只描述"能做什么"。

GOAP 的核心概念：

| 概念 | 英文 | 含义 |
|------|------|------|
| 目标 | Goal | 期望达成的状态断言 |
| 动作 | Action | 可执行的操作 |
| 前置条件 | Preconditions | 动作执行前必须成立的状态 |
| 效果 | Effects | 动作执行后成立的状态 |
| 世界状态 | World State | 当前所有已知断言 |
| 规划器 | Planner | 搜索动作序列的算法 |

规划问题可以形式化为：给定初始状态 `S0` 与目标 `G`，寻找动作序列 `[a1, a2, ..., an]`，使得依次执行后 `G ⊆ Sn`。

## 1.2 Embabel 的定位

- 语言与生态：JVM（Java/Kotlin），与 Spring 生态亲和。
- 抽象层级：以"领域动作"为一等公民，而非以"图节点"为一等公民。
- 核心能力：类型化输入输出、规划式编排、混合 LLM 与确定性代码。
- 目标用户：希望复用 Spring 依赖注入、同时获得动态编排能力的团队。

## 1.3 与主流框架的差异

| 框架 | 编排范式 | 路径决定时机 | 路径决定者 |
|------|---------|-------------|-----------|
| LangGraph4j | 图 | 编码时 | 开发者 |
| Spring AI Alibaba Graph | 图 | 编码时（条件边） | 开发者 |
| AutoGen/CrewAI | 角色对话 | 运行时 | LLM 主管 |
| Embabel | GOAP 规划 | 运行时 | 规划器 |

> 结论：Embabel 的独特价值在于"路径由规划器搜索"，而非由开发者画死或由 LLM 临场决定。

# 2. 核心思路

结论先行：Embabel 把多 Agent 协作看作一个规划问题——目标是终态，Agent 的动作是算子，规划器负责串联。

## 2.1 从目标到路径

```
目标：完成一份市场分析报告
  ↓ GOAP 规划
需要：数据 + 分析 + 撰写
  ↓ 映射到 Agent
数据 Agent → 分析 Agent → 撰写 Agent
```

规划过程可展开为：

```
目标断言：reportReady = true
        ▲
        │ 效果：reportReady
   [撰写报告] 前置：analyzed = true
        ▲
        │ 效果：analyzed
   [分析数据] 前置：hasData = true
        ▲
        │ 效果：hasData
   [收集数据] 前置：（无）
```

规划器从目标反向展开，直到所有前置条件都能被初始状态满足。

## 2.2 状态与效果

- 状态是布尔断言集合，例如 `hasData`、`analyzed`、`reportReady`。
- 效果通常写成"新增断言"（add list），必要时也可写"删除断言"（delete list）。
- 复杂值（如数据正文）不进入规划空间，只作为动作的输出被携带。

> 边界：把大对象放进规划状态会导致状态空间爆炸，规划状态应只保留"布尔/枚举"这类轻量断言。

## 2.3 规划器与执行器分工

| 组件 | 职责 | 是否调用 LLM |
|------|------|-------------|
| 规划器 | 搜索动作序列 | 否（纯算法） |
| 动作 | 执行具体步骤 | 可能 |
| 执行器 | 按序执行并维护状态 | 否 |
| 守卫 | 校验前置条件 | 否 |

职责分离的好处：规划是确定性的、可测试的；LLM 只出现在真正需要生成的动作内部。

## 2.4 规划算法选择

| 算法 | 特点 | 适用 |
|------|------|------|
| 前向搜索 | 从初始状态出发 | 动作少、目标明确 |
| 反向搜索 | 从目标反向展开 | 目标集中、分支多 |
| A* | 带启发式的最优搜索 | 需要最优路径 |
| 贪心 | 每次选当前最优 | 实时性要求高 |
| 分层规划 | 先粗后细 | 状态空间极大 |

选择原则：

- 动作数 < 20 且目标单一 → 前向搜索足够。
- 有明显启发式（如距离目标步数）→ A* 更省搜索。
- 实时场景 → 贪心 + 预置模板，宁可次优也不要卡顿。
- 状态空间大 → 分层，先规划"阶段"再规划"动作"。

> 边界：无论选哪种算法，都要设置搜索上限。规划器本身也可能陷入组合爆炸，不能假设它一定能返回结果。

# 3. 定义 Agent 为动作

结论先行：在 Embabel 中，一个 Agent 是一组相关动作的容器；每个动作通过注解声明前置条件与效果，规划器据此把它纳入路径。

## 3.1 @Agent 与 @Action 注解

```java
@Agent(description = "市场分析系统")
public class MarketAnalysisAgent {

    @Action(name = "收集数据", preconditions = {}, effects = {"hasData"})
    public void collectData(AgentContext ctx, String topic) {
        String data = dataService.collect(topic);
        ctx.setState("data", data);
        ctx.setState("hasData", true);
    }

    @Action(name = "分析数据", preconditions = {"hasData"}, effects = {"analyzed"})
    public void analyze(AgentContext ctx) {
        String analysis = llm.chat("分析：" + ctx.getState("data"));
        ctx.setState("analysis", analysis);
        ctx.setState("analyzed", true);
    }

    @Action(name = "撰写报告", preconditions = {"analyzed"}, effects = {"reportReady"})
    public void write(AgentContext ctx) {
        String report = llm.chat("撰写报告：" + ctx.getState("analysis"));
        ctx.setState("report", report);
        ctx.setState("reportReady", true);
    }
}
```

注解语义：

| 注解 | 作用 | 关键属性 |
|------|------|---------|
| `@Agent` | 声明 Agent 容器 | description |
| `@Action` | 声明可规划动作 | name、preconditions、effects |

## 3.2 前置条件与效果

前置条件与效果构成动作的"契约"：

- 前置条件不满足 → 该动作当前不可用。
- 效果声明错误 → 规划器会给出错误路径。
- 效果应"真实反映"动作做了什么，不能少写（否则后续动作前置不满足）也不能多写（否则规划出无效路径）。

```java
@Action(
        name = "校验数据",
        preconditions = {"hasData"},
        effects = {"dataValidated"})
public void validate(AgentContext ctx) {
    boolean ok = validator.check((String) ctx.getState("data"));
    ctx.setState("dataValidated", ok);
}
```

## 3.3 上下文读写

`AgentContext` 是动作之间传递数据的载体。

### 3.3.1 状态存取

```java
ctx.setState("data", data);                    // 写入
String data = (String) ctx.getState("data");   // 读取
```

建议：

- 键名集中定义为常量，避免拼写漂移。
- 值类型保持稳定，读取处显式转换。
- 只把"下游需要"的数据写入上下文，中间态可丢弃。

### 3.3.2 类型化输入输出

更安全的做法是用强类型参数与返回，而不是字符串键：

```java
@Action(name = "分析数据", preconditions = {"hasData"})
public Analysis analyze(DataBundle data) {
    String text = llm.chat("分析：" + data.content());
    return new Analysis(text, confidence(text));
}
```

类型化设计让编译器帮你发现"前置数据缺失"，比字符串键更不易出错。

> 边界：类型化输入输出与规划断言是两套机制——类型负责数据流，断言负责路径搜索，二者需要保持一致。

# 4. 目标驱动执行

结论先行：使用者只声明目标与初始输入，规划器与执行器负责把路径跑完并返回结果。

## 4.1 定义目标

```java
Goal goal = Goal.of("reportReady", true);
```

目标可以是单个断言，也可以是多个断言的合取：

```java
Goal goal = Goal.allOf(
        Goal.of("reportReady", true),
        Goal.of("dataValidated", true));
```

## 4.2 运行与结果

```java
AgentContext ctx = AgentContext.builder()
        .withInput("topic", "AI 市场")
        .build();

AgentResult result = runner.run(marketAnalysisAgent, goal, ctx);
```

> 规划器自动搜索"收集 → 分析 → 撰写"的路径。

结果对象通常包含：

| 字段 | 含义 |
|------|------|
| success | 是否达成目标 |
| context | 最终上下文 |
| plan | 实际执行的路径 |
| cost | 累计代价（若启用） |
| errors | 执行中的错误 |

## 4.3 执行轨迹

一次成功执行的轨迹可以记录为：

```
Trace: goal=reportReady
├── Plan: [收集数据, 分析数据, 撰写报告]
├── Step 1: 收集数据  → 效果 hasData
├── Step 2: 分析数据  → 效果 analyzed
└── Step 3: 撰写报告  → 效果 reportReady
```

生产建议：把"规划出的路径"与"实际执行的路径"都记入日志，二者不一致往往意味着效果声明有误或运行时有干预。

# 5. 动态适应

结论先行：当某动作失败或环境变化时，Embabel 可以基于当前状态重新规划，寻找替代路径——这是 GOAP 相对固定编排的核心优势。

## 5.1 失败后的替代路径

```
若"收集数据"失败：
  GOAP 自动寻找替代路径
  （如：换数据源、跳过、降级）
```

替代路径示例：

| 原路径 | 失败点 | 替代路径 |
|--------|--------|---------|
| 数据源 A → 分析 → 撰写 | A 不可用 | 数据源 B → 分析 → 撰写 |
| 精确分析 → 撰写 | 数据不足 | 粗略估计 → 撰写（降级） |
| 检索 → 分析 → 撰写 | 检索为空 | 直接撰写（附免责声明） |

```java
// 伪代码：执行器在动作失败后触发重规划
if (!actionSucceeded) {
    State current = ctx.snapshot();
    Plan newPlan = planner.plan(current, goal);
    execute(newPlan);
}
```

## 5.2 成本与效用函数

规划器可在多条可行路径中选择"代价最小"的一条。

```java
// 伪代码：为动作声明成本
@Action(name = "调用付费接口", cost = 10)
public void callPaidApi(AgentContext ctx) { /* ... */ }

@Action(name = "使用缓存", cost = 1)
public void useCache(AgentContext ctx) { /* ... */ }
```

| 维度 | 说明 | 权重示例 |
|------|------|---------|
| 金钱成本 | API/Token 费用 | 高 |
| 时间成本 | 预期延迟 | 中 |
| 质量 | 预期输出质量 | 高 |
| 风险 | 失败概率 | 中 |

## 5.3 约束与守卫

> 边界：纯效用最优可能选出"技术上可行但业务上不可接受"的路径，必须加硬约束。

```java
@Action(
        name = "撰写报告",
        preconditions = {"analyzed"},
        effects = {"reportReady"})
@Guard("complianceChecker.allows(#ctx)")
public void write(AgentContext ctx) { /* ... */ }
```

约束类型：

- 合规约束：不得输出未审核内容。
- 权限约束：无权限的动作不可规划。
- 预算约束：累计成本不得超过上限。
- 步数约束：路径长度不得超过 N。

生产建议：把硬约束做成"前置条件的一部分"，让不可行路径在规划阶段就被排除，而不是执行时才失败。

# 6. 与图编排的对比

结论先行：图编排适合"流程已知"，GOAP 适合"目标已知但路径可灵活变化"。

## 6.1 路径来源

| 维度 | Graph 编排 | GOAP |
|------|-----------|------|
| 路径 | 开发者定义 | 规划器搜索 |
| 适应变化 | 需改图 | 自动 |
| 可预测性 | 高 | 中 |
| 调试 | 易 | 较难 |
| 适用 | 稳定流程 | 动态目标 |

## 6.2 可预测性与调试

- 图编排：路径静态，任何一次执行都能对着图核对。
- GOAP：路径动态，同样输入可能因规划器实现或成本变化走出不同路径。

调试 GOAP 的关键抓手：

| 手段 | 作用 |
|------|------|
| 记录规划结果 | 知道"打算怎么做" |
| 记录实际执行 | 知道"实际怎么做" |
| 记录失败与重规划 | 知道"为何偏离" |
| 可视化状态空间 | 知道"为何选这条" |

## 6.3 选型建议

- 流程稳定、需要严格审计 → 图编排。
- 目标明确、环境多变、允许自动降级 → GOAP。
- 团队熟悉 Spring、想少写胶水代码 → Embabel 有吸引力。
- 对框架成熟度要求极高、需要大规模生产验证 → 谨慎评估后再用。

# 7. 完整示例：市场分析报告

结论先行：一个可运行的 Embabel 应用包含三部分——领域数据模型、动作集合、以及"目标 + 初始输入"的运行调用。下面把它们串起来。

## 7.1 领域模型设计

```java
public record DataBundle(String topic, String content, Instant fetchedAt) {}

public record Analysis(String text, double confidence) {}

public record Report(String title, String body, List<String> citations) {}
```

领域模型的作用：

- 让动作的输入输出有明确类型，便于测试与重构。
- 把"数据搬运"从字符串键中解放出来。
- 便于序列化后写入检查点或审计日志。

| 模型 | 生产者 | 消费者 |
|------|--------|--------|
| DataBundle | 收集数据动作 | 分析数据动作 |
| Analysis | 分析数据动作 | 撰写报告动作 |
| Report | 撰写报告动作 | 汇总/输出 |

## 7.2 动作实现细节

### 7.2.1 数据采集动作

```java
@Action(
        name = "收集数据",
        preconditions = {},
        effects = {"hasData"},
        cost = 3)
public DataBundle collect(AgentContext ctx, String topic) {
    String content = dataService.collect(topic);
    if (content == null || content.isBlank()) {
        throw new ActionFailedException("数据源返回为空");
    }
    ctx.setState("hasData", true);
    return new DataBundle(topic, content, Instant.now());
}
```

要点：

- 动作内部可以抛异常表示失败，执行器据此触发重规划。
- 成功时才写入效果断言，失败时保持状态不变。
- 成本声明让规划器在等价路径间择优。

### 7.2.2 分析与写作动作

```java
@Action(
        name = "分析数据",
        preconditions = {"hasData"},
        effects = {"analyzed"},
        cost = 5)
public Analysis analyze(DataBundle data) {
    String prompt = "请分析以下数据并给出结论与置信度：\n" + data.content();
    String text = llm.chat(prompt);
    return new Analysis(text, estimateConfidence(text));
}

@Action(
        name = "撰写报告",
        preconditions = {"analyzed"},
        effects = {"reportReady"},
        cost = 5)
public Report write(Analysis analysis) {
    String body = llm.chat("基于以下结论撰写市场分析报告：\n" + analysis.text());
    return new Report("AI 市场分析", body, analysis.citations());
}
```

> 边界：LLM 动作应设置超时与重试上限，并把"生成失败"与"内容不佳"区分开——前者可重试，后者应交给评审动作。

## 7.3 组装、运行与观察

```java
Goal goal = Goal.of("reportReady", true);

AgentContext ctx = AgentContext.builder()
        .withInput("topic", "AI 市场")
        .withMaxSteps(10)
        .withCostBudget(50)
        .build();

AgentResult result = runner.run(
        new MarketAnalysisAgent(dataService, llm),
        goal,
        ctx);

if (result.success()) {
    Report report = result.get("report", Report.class);
    log.info("报告生成成功，路径={}", result.plan());
} else {
    log.warn("未达成目标，错误={}", result.errors());
}
```

观察要点：

| 观察项 | 说明 | 异常信号 |
|--------|------|---------|
| plan 长度 | 规划路径步数 | 异常长说明效果声明有误 |
| cost | 累计成本 | 接近预算上限 |
| 重规划次数 | 触发频率 | 频繁说明依赖不稳 |
| 失败动作 | 具体动作名 | 集中于某外部依赖 |

# 8. 局限与适用边界

结论先行：GOAP 不是万能钥匙。它在流程高度确定、需要严格审计或状态空间过大的场景下反而拖后腿。

## 8.1 不适合的场景

| 场景 | 为何不适合 | 替代方案 |
|------|-----------|---------|
| 线性固定流程 | 规划开销无收益 | 图编排/顺序代码 |
| 严格审计要求 | 动态路径难复现 | 图编排 + 检查点 |
| 动作数极多 | 状态空间爆炸 | 分层规划/分组激活 |
| 实时低延迟 | 规划耗时叠加 | 预编译路径缓存 |
| 需要人工逐步确认 | 路径不固定 | 图编排中断节点 |

## 8.2 与图编排混合

实践中常见"外层用图、内层用 GOAP"的混合模式：

```
┌─────────────────────────────────────────┐
│  StateGraph（稳定骨架）                  │
│  ① 鉴权 → ② 意图识别 → ③ GOAP 子任务   │
│                              │          │
│                              ▼          │
│              ┌────────────────────────┐ │
│              │ GOAP 规划（动态部分）   │ │
│              │ 收集→分析→撰写（可替换）│ │
│              └────────────────────────┘ │
│  ④ 审核 → ⑤ 输出                         │
└─────────────────────────────────────────┘
```

好处：骨架可控可审计，动态部分保持灵活。

## 8.3 迁移与演进

- 先用接口隔离 Agent 能力，再决定由谁编排。
- 从"固定路径"起步，稳定后再逐步放开规划自由度。
- 记录历史路径，若某路径长期高频，可将其固化为模板以降低规划开销。
- 保持动作的粒度适中：过粗难复用，过细规划空间大。

# 9. 生产注意事项

结论先行：GOAP 的主要生产风险是"规划空间爆炸、行为不可预测、缺少终止保证、调试困难、框架较新"。

## 9.1 规划空间治理

| 风险 | 建议 |
|------|------|
| 规划空间爆炸 | 限制动作数 |
| 不可预测 | 加约束与验证 |
| 无终止 | 步数上限 |
| 调试难 | 记录规划与执行 |
| 框架较新 | 评估成熟度 |

补充措施：

- 对动作按领域分组，只在需要时激活相关组。
- 用守卫剪枝，让不可行动作尽早出局。
- 给规划器设置搜索上限（时间/节点数），超限则回退到预置路径。

## 9.2 可观测与审计

- 记录每次规划的目标、输入状态、输出路径与代价。
- 记录每次重规划的原因（哪个前置条件失效）。
- 高风险动作保留人工审批与完整审计日志。
- 对 LLM 生成内容做事实校验与合规过滤。

## 9.3 成熟度与依赖

- 评估框架版本稳定性、社区活跃度与长期维护承诺。
- 核心业务逻辑尽量与框架解耦（用接口隔离），降低替换成本。
- 关键路径准备"降级到确定性代码"的兜底方案。
- 压测规划器在最大动作集下的延迟与内存表现。

## 9.4 回退策略

当规划失败或超预算时，系统不能直接报错，应有序回退：

| 层级 | 回退动作 | 用户感知 |
|------|---------|---------|
| 一级 | 缩短规划范围重试 | 略慢 |
| 二级 | 使用预置固定路径 | 结果可能略差 |
| 三级 | 降级为单 Agent 直答 | 功能受限但可用 |
| 四级 | 明确告知失败并转人工 | 明确失败原因 |

生产建议：回退路径必须经过与主路径同等的测试，否则"兜底"本身会成为新的故障点。

# 10. 小结

## 10.1 要点回顾

- Embabel 用 GOAP 从目标自动规划多 Agent 协作。
- Agent 的动作带前置条件与效果。
- 环境变化时自动重规划。
- 比图编排灵活，但可预测性较低。
- 需限制动作数与步数。
- 硬约束应作为前置条件在规划阶段排除。

## 10.2 落地清单

- [ ] 动作的 effects 与 preconditions 声明准确、可验证。
- [ ] 规划状态只放轻量断言，大对象走上下文。
- [ ] 为动作声明成本，明确成本函数。
- [ ] 加合规/权限/预算/步数四类硬约束。
- [ ] 同时记录"规划路径"与"执行路径"。
- [ ] 准备搜索超限时的确定性降级路径。
- [ ] 评估框架成熟度并做接口隔离。
