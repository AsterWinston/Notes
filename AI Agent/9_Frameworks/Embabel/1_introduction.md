# 1. 定位

**结论先行**：**Embabel** 是面向 JVM（Java 虚拟机）的 Agent（智能体）框架，由 Spring 生态知名开发者 Rod Johnson 创建，核心特色是**基于 GOAP（Goal-Oriented Action Planning，目标导向动作规划）的目标导向规划**。

```
Embabel = Spring + GOAP 规划 + Agent
```

## 1.1 背景与作者

### 1.1.1 Rod Johnson

Rod Johnson 是 Spring Framework 的创始人。Embabel 延续了他"**用约定与容器简化复杂编排**"的设计哲学：把 Spring 的依赖注入（DI）与面向切面（AOP）思路，迁移到 Agent 的动作规划上。

### 1.1.2 设计动机

传统 Agent 框架多用"提示 + 工具调用循环"，路径由模型每步临时决定，难以保证可达性。Embabel 的动机是：

1. **目标可声明**：开发者声明"要达成什么"，而非"怎么走"。
2. **路径可搜索**：由规划器在动作空间里搜索可行路径。
3. **类型安全**：用 JVM 类型系统约束动作的输入输出。

## 1.2 核心公式

| 组成 | 作用 | 对应技术 |
|------|------|----------|
| Spring | 容器、依赖注入、生命周期 | Spring Framework |
| GOAP | 目标导向动作规划 | AI 规划算法 |
| Agent | 目标 + 动作 + 规划器 | Embabel 运行时 |

## 1.3 边界与坑

- 框架较新，API 可能变动。
- 规划质量依赖动作定义（前置条件/效果）的完备性。
- 不是"零配置"框架，需要理解 GOAP 才能用好。

## 1.4 生产建议

- 先做概念验证（PoC），再评估是否进入生产。
- 与 Spring AI 组合使用，而非二选一。
- 给规划器设置步数上限。

## 1.5 架构分层

```
┌────────────────────────────────────┐
│  应用层：目标声明（Goal）           │
├────────────────────────────────────┤
│  规划层：GOAP Planner（搜索路径）   │
├────────────────────────────────────┤
│  动作层：Action（领域方法）         │
├────────────────────────────────────┤
│  模型层：LLM（可选，用于生成/决策） │
├────────────────────────────────────┤
│  基础设施：Spring 容器 / 外部服务   │
└────────────────────────────────────┘
```

| 层 | 职责 | 是否必需 |
|----|------|:--------:|
| 应用层 | 声明目标 | 是 |
| 规划层 | 搜索路径 | 是 |
| 动作层 | 执行动作 | 是 |
| 模型层 | 语义生成 | 否 |
| 基础设施 | 依赖注入 | 是 |

关键洞察：**Embabel 的规划不强制依赖 LLM**。GOAP 是符号规划，LLM 只用于动作内部的语义处理。

# 2. GOAP 是什么

**结论先行**：**GOAP（Goal-Oriented Action Planning）** = 从目标出发，自动搜索达成目标的动作序列。

```
目标：完成旅行规划
  ↓ 反向搜索
前置条件：知道天气、知道预算
  ↓
动作：查天气、查预算、生成行程
```

## 2.1 机制拆解

### 2.1.1 状态与目标

GOAP 把世界建模为**状态集合**（一组键值对），目标是**期望状态**，动作是**状态的转换函数**。

| 概念 | 形式化 | 例子 |
|------|--------|------|
| 状态 State | `{key: value}` | `{weather: unknown}` |
| 目标 Goal | 期望状态 | `{itinerary: done}` |
| 动作 Action | 前置条件 → 效果 | 查天气 |

### 2.1.2 前向与反向搜索

```
反向搜索（Embabel 常用）：
  目标 itinerary=done
    ← 需要 precond: weather=known
    ← 动作 getWeather 可产出 weather=known
    ← 需要 precond: city=known
    ← 动作 resolveCity 可产出 city=known

前向搜索：
  初始状态 → 不断应用可用动作 → 直到满足目标
```

### 2.1.3 代价与启发

| 项 | 说明 |
|----|------|
| 代价 Cost | 每个动作的成本，规划器优先低成本路径 |
| 启发 Heuristic | 估算到目标的距离，加速搜索 |
| 剪枝 | 排除无效或超限路径 |

## 2.2 一个例子

```
目标：itinerary=done

可用动作：
  A1 resolveCity:  {}                → {city: known}
  A2 getWeather:   {city: known}     → {weather: known}
  A3 getBudget:    {}                → {budget: known}
  A4 buildPlan:    {weather: known, budget: known}
                                     → {itinerary: done}

规划结果：A1 → A2 → A3 → A4
```

> 与传统流程编排不同，GOAP **由目标驱动**，自动寻找可行路径。

## 2.3 边界与坑

- 动作空间大时搜索可能爆炸。
- 前置条件定义不全，规划器会产出不可行路径。
- 状态是"声明式近似"，与真实世界可能不一致。

## 2.4 生产建议

- 动作数量控制在几十个以内。
- 为动作标注代价，引导规划器走低成本路径。
- 规划结果执行前做校验。

## 2.5 与其他规划方法对比

| 方法 | 决策者 | 是否需 LLM | 可解释性 | 典型用途 |
|------|--------|:----------:|:--------:|----------|
| GOAP | 规划器搜索 | 否 | 高 | 目标导向任务 |
| ReAct | LLM 逐步推理 | 是 | 中 | 通用 Agent |
| Plan-and-Execute | LLM 生成计划 | 是 | 中 | 复杂任务 |
| 规则引擎 | 开发者 | 否 | 高 | 固定规则 |

结论：**GOAP 的可解释性与可预测性优于纯 LLM 规划，但灵活性受限于动作定义**。

# 3. 核心概念

**结论先行**：Embabel 的世界由五个概念构成——**Goal、Action、Precondition、Effect、Planner**。

## 3.1 概念总览

| 概念 | 说明 |
|------|------|
| Goal | 目标（要达到的状态） |
| Action | 动作（改变状态） |
| Precondition | 前置条件（动作可执行的前提） |
| Effect | 效果（动作执行后的状态变化） |
| Planner | 规划器（搜索动作序列） |

## 3.2 Goal 与状态

### 3.2.1 定义

Goal 是一个期望达成的状态。Embabel 用注解标注"哪个动作达成目标"。

### 3.2.2 示例

```java
// 伪代码：目标声明
@Agent(description = "旅行规划智能体")
public class TravelAgent {

    @AchievesGoal(description = "生成完整的旅行行程")
    @Action
    public Itinerary buildItinerary(Weather weather, Budget budget) {
        return itineraryService.compose(weather, budget);
    }
}
```

## 3.3 Action 与 Effect

### 3.3.1 定义

Action 是可执行单元，方法返回值即"效果"（新增到状态中的对象）。

### 3.3.2 示例

```java
// 伪代码：动作定义
@Action
public Weather getWeather(City city) {
    return weatherClient.query(city.getName());
}

@Action
public City resolveCity(String cityName) {
    return cityService.find(cityName);
}
```

## 3.4 Precondition 与 Planner

### 3.4.1 前置条件

方法的**参数类型**天然构成前置条件：要调用 `getWeather(City)`，状态中必须已有 `City`。

```
City 不存在 → getWeather 不可执行
City 存在   → getWeather 可执行，产出 Weather
```

### 3.4.2 规划器

规划器扫描所有动作，从目标反向推导，找到一条"参数可满足"的调用链。

| 规划器行为 | 说明 |
|------------|------|
| 类型匹配 | 参数类型必须已存在于状态 |
| 反向链 | 从目标动作回溯依赖 |
| 路径选择 | 多路径时按代价/规则选择 |

## 3.5 边界与坑

- 类型相同但语义不同的对象会混淆前置条件。
- 可选参数、集合参数会让规划复杂化。
- 循环依赖（A 依赖 B、B 依赖 A）导致规划失败。

## 3.6 生产建议

- 用专门的领域类型，避免都用 `String`。
- 保持动作职责单一，便于规划器组合。
- 对规划结果打日志，便于排查。

# 4. 与传统编排的对比

**结论先行**：传统编排是**流程驱动**，GOAP 是**目标驱动**；前者可预测性高，后者灵活性高。

## 4.1 对比表

| 维度 | 传统编排（Graph） | GOAP |
|------|-------------------|------|
| 驱动 | 流程驱动 | 目标驱动 |
| 路径 | 开发者定义 | 规划器搜索 |
| 灵活性 | 中 | 高 |
| 可预测性 | 高 | 中 |
| 适用 | 稳定流程 | 目标导向任务 |

## 4.2 何时用 GOAP

### 4.2.1 适合

- 目标明确但路径不固定。
- 动作可自由组合。
- 环境动态变化，需要重新规划。

### 4.2.2 不适合

- 流程强合规、必须固定。
- 延迟敏感（规划有额外开销）。
- 动作极少（两个动作，直接调用即可）。

## 4.3 决策示意

```
路径是否固定？
 ├─ 是 → 传统编排
 └─ 否 → 动作是否类型清晰、可组合？
          ├─ 是 → GOAP
          └─ 否 → 显式编排 + 模型决策
```

## 4.4 边界与坑

- GOAP 的可预测性"中"，意味着仍需人工校验。
- 规划开销在简单任务上是纯浪费。

## 4.5 生产建议

- 关键路径用编排固定，长尾用 GOAP。
- 为规划设置超时与步数上限。

# 5. 适用场景

**结论先行**：Embabel 适合**目标导向、动态环境、复杂决策**且**使用 Spring 生态**的任务。

## 5.1 场景清单

| 场景 | 说明 |
|------|------|
| 目标导向任务 | 给定目标，自动规划 |
| 动态环境 | 条件变化时重新规划 |
| 复杂决策 | 多个可行路径 |
| Spring 生态 | 原生集成 |

## 5.2 典型案例

### 5.2.1 目标导向

用户说"帮我安排一次三天两晚的出差"，系统自动规划"查航班 → 查酒店 → 排日程"。

### 5.2.2 动态环境

执行中航班取消，规划器重新搜索替代路径（改签 + 换酒店）。

### 5.2.3 复杂决策

多约束（预算、时间、偏好）下的方案选择，规划器枚举可行路径并按代价排序。

## 5.3 反例

```
反例：用 Embabel 做"固定三步的数据清洗管道"
问题：路径完全固定，GOAP 规划纯属浪费
修复：直接写顺序代码
```

## 5.4 边界与坑

- 目标描述模糊会导致规划结果不稳定。
- 动态重规划需要幂等动作支持。

## 5.5 生产建议

- 目标与约束显式化、结构化。
- 动作设计为幂等，支持重试与重规划。

## 5.6 场景评分

| 场景 | 目标明确度 | 路径灵活性 | 是否适合 |
|------|:----------:|:----------:|:--------:|
| 出差安排 | 高 | 高 | 是 |
| 固定报表 | 高 | 低 | 否 |
| 故障排查 | 中 | 高 | 是 |
| 合规审批 | 高 | 低 | 否 |
| 多约束选品 | 高 | 中 | 是 |

判断口诀：**目标明确 + 路径多变 = 适合 GOAP**。

# 6. 与其他框架对比

**结论先行**：Embabel 以 GOAP 规划为差异化卖点，Spring AI 与 LangChain4j 更通用、更成熟。

## 6.1 对比表

| 维度 | Embabel | Spring AI | LangChain4j |
|------|---------|-----------|-------------|
| 核心 | GOAP 规划 | 统一抽象 | 声明式服务 |
| Spring | 原生 | 原生 | Starter |
| 成熟度 | 较新 | 成熟 | 成熟 |
| 适用 | 目标导向 | 通用 | 通用 |

## 6.2 组合使用

### 6.2.1 与 Spring AI

用 Spring AI 提供模型访问与向量库，用 Embabel 负责规划层。

```java
// 伪代码：Spring AI 提供 ChatClient，Embabel 动作内部调用
@Action
public Summary summarize(Document doc) {
    return chatClient.prompt()
            .user("总结：" + doc.text())
            .call()
            .entity(Summary.class);
}
```

### 6.2.2 与 LangChain4j

用 LangChain4j 的声明式接口（`@AiService`）封装模型调用，作为 Embabel 的动作实现。

## 6.3 边界与坑

- 多框架叠加会增加依赖与学习成本。
- 版本兼容需验证。

## 6.4 生产建议

- 只用 Embabel 的规划层，模型访问交给成熟框架。
- 保持动作实现与规划声明解耦。

## 6.5 选型矩阵

| 需求 | 推荐 |
|------|------|
| 通用 LLM 应用 | Spring AI |
| 声明式接口、快速开发 | LangChain4j |
| 目标导向、路径动态 | Embabel |
| 已有 Spring 体系 | Embabel + Spring AI |

# 7. 学习路径

**结论先行**：建议按"**概念 → 定义 → 规划 → 集成 → 企业实践**"五步推进。

## 7.1 五步路径

```
① 理解 GOAP 概念
② 定义 Goal / Action / Precondition
③ 使用 Planner 自动规划
④ 集成到 Spring 应用
⑤ 企业实践
```

| 阶段 | 目标 | 产出 |
|------|------|------|
| ① | 理解概念 | 能画出状态-动作图 |
| ② | 定义领域动作 | 一组带注解的方法 |
| ③ | 跑通规划 | 最小可运行示例 |
| ④ | Spring 集成 | 可注入的 Agent Bean |
| ⑤ | 企业实践 | 可观测、可治理的 Agent |

## 7.2 最小示例

### 7.2.1 Goal / Action 定义

```java
// 伪代码：Embabel 风格的 Agent
@Agent(description = "旅行助手")
public class TravelAgent {

    @Action
    public City resolveCity(String name) {
        return cityService.find(name);
    }

    @Action
    public Weather getWeather(City city) {
        return weatherClient.query(city);
    }

    @AchievesGoal(description = "生成行程")
    @Action
    public Itinerary plan(City city, Weather weather) {
        return itineraryService.compose(city, weather);
    }
}
```

### 7.2.2 运行

```java
// 伪代码：交给规划器执行
AgentProcess process = agentPlatform
        .createAgentProcessFrom(TravelAgent.class);
Object result = process.run("帮我规划东京行程");
```

## 7.3 边界与坑

- 注解名与 API 以官方文档为准，框架仍在演进。
- 规划失败时需回退到人工流程。

## 7.4 生产建议

- 从单一目标、少量动作起步。
- 逐步增加动作与约束。

# 8. 生产注意事项

**结论先行**：Embabel 上生产的关键是**评估成熟度、约束规划、限制步数、保留回退**。

## 8.1 风险清单

| 风险 | 建议 |
|------|------|
| 框架较新 | 评估成熟度 |
| 规划不可预测 | 加约束与验证 |
| 无终止 | 步数上限 |
| 生态小 | 结合 Spring AI 使用 |

## 8.2 规划约束

### 8.2.1 步数上限

```java
// 伪代码：规划约束
PlannerConfig config = PlannerConfig.builder()
        .maxSteps(10)
        .timeout(Duration.ofSeconds(20))
        .build();
```

### 8.2.2 结果校验

```java
// 伪代码：执行前校验
if (!validator.isReachable(plan)) {
    throw new PlanRejectedException("规划路径不可达");
}
```

## 8.3 可观测

| 指标 | 说明 |
|------|------|
| 规划耗时 | 性能 |
| 规划步数 | 复杂度 |
| 规划成功率 | 稳定性 |
| 回退次数 | 可用性 |

## 8.4 生产建议

- 规划与执行分两阶段，各自埋点。
- 保留固定编排作为回退路径。
- 关注框架版本迭代，及时跟进。

## 8.5 上线检查清单

| 项 | 检查点 |
|----|--------|
| 规划 | 步数上限、超时、校验器 |
| 动作 | 幂等、超时、异常处理 |
| 观测 | 规划耗时、步数、成功率 |
| 回退 | 固定编排兜底、可远程关闭 |
| 版本 | 锁定版本、回归测试 |

# 9. 小结

- Embabel 是 JVM 的 Agent 框架，核心是 GOAP 规划。
- GOAP 从目标出发，自动搜索动作序列。
- 与 Spring 原生集成。
- 适合目标导向、动态环境的任务。
- 框架较新，需评估成熟度。
