# 1. 多 Agent 概述

当单 Agent 的指令、工具与职责过载时，应拆分为多个各司其职的 Agent 协作。ADK 原生支持子 Agent（subAgent）与工作流 Agent。

## 1.1 结论先行

- 多 Agent 的价值在于"职责分离"：每个 Agent 只关心自己的指令、工具与输出。
- ADK 提供两类协作：LlmAgent 的 `subAgents` 委托（模型决策），以及 Sequential/Parallel/Loop 工作流（确定性编排）。
- 拆分不是越多越好：每个 Agent 都增加一次模型调用与一次上下文组装，成本与延迟随之上升。

## 1.2 机制拆解

### 1.2.1 拓扑结构

```
                 +-------------+
                 |  Root Agent |
                 +------+------+
                        |
        +---------------+---------------+
        |               |               |
   +---------+    +---------+    +---------+
   | order   |    | refund  |    | search  |
   +---------+    +---------+    +---------+
```

### 1.2.2 两种协作范式

| 范式 | 决策者 | 特点 | 适用 |
|------|--------|------|------|
| 委托（subAgents） | 模型 | 动态路由 | 意图分类、专家分诊 |
| 工作流 Agent | 开发者 | 确定性执行 | 固定流程、流水线 |

### 1.2.3 何时该拆分

| 信号 | 说明 |
|------|------|
| 指令过长 | 规则互相干扰 |
| 工具过多 | 误调用率高 |
| 职责混杂 | 一个 Agent 干多域 |
| 需要并行 | 分支可独立 |

- 反之，如果拆分后仍需大量共享上下文，说明边界没划对。

## 1.3 代码示例

```java
LlmAgent root = LlmAgent.builder()
        .name("root")
        .model("gemini-2.0-flash")
        .instruction("根据用户问题转交给合适的子 Agent。")
        .subAgents(orderAgent, refundAgent)
        .build();
```

## 1.4 边界与坑

- 无限制的委托会形成"传话游戏"，信息逐层失真。
- 每个 Agent 的指令都会进入上下文，总上下文可能超限。
- 过度拆分导致链路长、延迟高、排障难。

## 1.5 生产建议

- 按业务域拆分，而非按函数拆分。
- 保持 Agent 树扁平，深度一般不超过 2~3 层。
- 为委托链建立 Trace，记录"谁委托给谁、为什么"。

# 2. 子 Agent 与委托

委托（delegation）是 LlmAgent 最常用的多 Agent 模式：父 Agent 依据子 Agent 的 `description` 决定转交。

## 2.1 结论先行

- 通过 `.subAgents(...)` 注册子 Agent。
- 父 Agent 把子 Agent 的名称与描述作为"可转交选项"告知模型。
- 委托本质是一次模型决策，因此 `description` 是路由准确率的关键。

## 2.2 机制拆解

### 2.2.1 description 路由

| 字段 | 作用 | 写法建议 |
|------|------|----------|
| `name` | 唯一标识 | `域-职责` |
| `description` | 路由依据 | 一句话说明"处理什么" |
| `instruction` | 子 Agent 行为 | 独立完整 |

### 2.2.2 委托流程

```
用户问题
   |
   v
Root Agent 判断意图
   |
   +-- 订单相关 --> orderAgent
   +-- 退款相关 --> refundAgent
   +-- 其他 ------> 自己回答
```

### 2.2.3 上下文交接

| 内容 | 是否传递 | 说明 |
|------|----------|------|
| 用户原始问题 | 是 | 子 Agent 需知道任务 |
| 历史对话 | 视实现 | 需实测确认 |
| State | 是 | 共享键值 |
| 工具 schema | 否 | 各 Agent 独立 |

- 若框架不自动传递历史，应把关键事实写入 State 再委托。

## 2.3 代码示例

```java
LlmAgent orderAgent = LlmAgent.builder()
        .name("order-agent")
        .description("处理订单相关问题")
        .model("gemini-2.0-flash")
        .instruction("你负责订单查询与修改。")
        .tools(new OrderTools())
        .build();

LlmAgent refundAgent = LlmAgent.builder()
        .name("refund-agent")
        .description("处理退款相关问题")
        .model("gemini-2.0-flash")
        .instruction("你负责退款申请与进度查询。")
        .tools(new RefundTools())
        .build();

LlmAgent rootAgent = LlmAgent.builder()
        .name("root")
        .description("客服总管")
        .model("gemini-2.0-flash")
        .instruction("根据用户问题转交给合适的子 Agent。")
        .subAgents(orderAgent, refundAgent)
        .build();
```

## 2.4 边界与坑

- 子 Agent 描述含糊会导致路由错误或反复委托。
- 子 Agent 之间无法互相看见，需要共享信息时应走 State。
- 委托后上下文如何交接取决于框架实现，务必实测确认历史是否传递。

## 2.5 生产建议

- 用评测集度量路由准确率，覆盖近义意图。
- 为每个子 Agent 独立设工具与指令，保持自治。
- 委托决策记入 Trace，便于分析误路由。

# 3. 顺序 Agent（Sequential）

SequentialAgent 按固定顺序依次执行子 Agent，适合"研究 → 撰写 → 审核"这类流水线。

## 3.1 结论先行

- 顺序 Agent 是确定性编排：子 Agent 一个接一个执行。
- 适合阶段清晰、前后依赖的流程。
- 前序输出通常经 State 传给后续 Agent。

## 3.2 机制拆解

```
input --> [research] --> [write] --> [review] --> output
             |              |            |
             +-- state -----+-- state ---+
```

| 特性 | 说明 |
|------|------|
| 执行顺序 | 注册顺序 |
| 数据传递 | State / outputKey |
| 失败策略 | 中断或跳过（视配置） |

### 3.2.1 数据传递

| 方式 | 优点 | 缺点 |
|------|------|------|
| outputKey → State | 结构化、可靠 | 需约定 key |
| 事件内容 | 无需配置 | 依赖模型复述 |
| 共享制品 Artifact | 适合大对象 | 需引用管理 |

## 3.3 代码示例

```java
SequentialAgent pipeline = SequentialAgent.builder()
        .name("pipeline")
        .subAgents(researchAgent, writeAgent, reviewAgent)
        .build();

Runner runner = Runner.builder()
        .agent(pipeline)
        .sessionService(sessionService)
        .build();
```

```java
// 用 outputKey 串联
LlmAgent researchAgent = LlmAgent.builder()
        .name("research")
        .model("gemini-2.0-flash")
        .instruction("调研主题并输出要点。")
        .outputKey("research_notes")
        .build();

LlmAgent writeAgent = LlmAgent.builder()
        .name("writer")
        .model("gemini-2.0-flash")
        .instruction("依据 state 中的 research_notes 撰写文章。")
        .outputKey("draft")
        .build();
```

## 3.4 边界与坑

- 任一阶段失败若不处理会中断整条流水线。
- 中间结果若只走自然语言，后续 Agent 解析成本高，应走 State。
- 顺序执行延迟是各阶段之和，长流程需评估超时。

## 3.5 生产建议

- 每阶段定义明确的输入/输出契约（State key）。
- 为长流程设置阶段级超时与重试。
- 阶段产物落库，支持断点续跑。

# 4. 并行 Agent（Parallel）

ParallelAgent 并发执行多个子 Agent，适合可独立并行、最后汇总的任务。

## 4.1 结论先行

- 并行 Agent 同时启动多个子 Agent，缩短总耗时。
- 各分支应相互独立，避免写同一 State 键造成竞争。
- 结果通常需要汇聚（merge）后交给后续节点。

## 4.2 机制拆解

```
            +--> [agentA] --+
input ------+               +--> merge --> output
            +--> [agentB] --+
```

| 特性 | 说明 |
|------|------|
| 执行方式 | 并发 |
| 耗时 | 约等于最慢分支 |
| 结果汇聚 | 各分支结果合并 |
| 风险 | 竞争、部分失败 |

### 4.2.1 汇聚策略

| 策略 | 说明 |
|------|------|
| 全量收集 | 等所有分支完成 |
| 最快返回 | 取首个成功结果 |
| 多数表决 | 多分支投票 |
| 结构化合并 | 各分支字段拼接 |

- 汇聚通常由一个下游 LlmAgent 或自定义逻辑完成。

## 4.3 代码示例

```java
ParallelAgent parallel = ParallelAgent.builder()
        .name("parallel")
        .subAgents(agentA, agentB)
        .build();
```

```java
// 各分支写不同 key，避免竞争
LlmAgent agentA = LlmAgent.builder()
        .name("risk-check")
        .model("gemini-2.0-flash")
        .instruction("评估风险，输出到 risk_result。")
        .outputKey("risk_result")
        .build();

LlmAgent agentB = LlmAgent.builder()
        .name("credit-check")
        .model("gemini-2.0-flash")
        .instruction("评估授信，输出到 credit_result。")
        .outputKey("credit_result")
        .build();
```

## 4.4 边界与坑

- 并发分支写同一 State 键会互相覆盖，必须使用独立键。
- 部分分支失败时，汇聚逻辑要能容错或整体失败。
- 并发会瞬时放大下游依赖与模型配额压力。

## 4.5 生产建议

- 明确各分支的输出键，汇聚节点显式读取。
- 为并发分支设统一超时，超时按失败处理。
- 监控并发度与限流，避免打爆下游。

# 5. 循环 Agent（Loop）

LoopAgent 反复执行子 Agent，直到满足终止条件或达到最大轮次，适合"生成 → 校验"的迭代。

## 5.1 结论先行

- 循环 Agent 用于迭代改进：生成、评估、再生成。
- 必须有明确终止条件：`maxIterations` 或子 Agent 发出完成信号。
- 无终止保护的循环是成本失控的头号来源。

## 5.2 机制拆解

### 5.2.1 终止条件

| 条件 | 说明 |
|------|------|
| `maxIterations` | 硬性轮次上限 |
| 完成信号 | 子 Agent 标记完成 |
| 校验通过 | 评估 Agent 判定达标 |
| 错误 | 异常中断 |

### 5.2.2 执行流程

```
      +-------------------+
      v                   |
[generate] --> [check] --通过?-- 否 --> 回到 generate
                   |
                  是
                   v
                 output
```

### 5.2.3 迭代预算

| 预算 | 建议 |
|------|------|
| 最大轮次 | 2~5 轮 |
| 单轮超时 | 与流程匹配 |
| 累计 token | 设上限告警 |
| 改进阈值 | 无提升即停 |

## 5.3 代码示例

```java
LoopAgent loop = LoopAgent.builder()
        .name("loop")
        .subAgents(generateAgent, checkAgent)
        .maxIterations(3)
        .build();
```

```java
LlmAgent generateAgent = LlmAgent.builder()
        .name("generator")
        .model("gemini-2.0-flash")
        .instruction("根据反馈改进答案，输出到 draft。")
        .outputKey("draft")
        .build();

LlmAgent checkAgent = LlmAgent.builder()
        .name("checker")
        .model("gemini-2.0-flash")
        .instruction("校验 draft，若不达标说明问题；达标则输出 PASS。")
        .outputKey("check_result")
        .build();
```

## 5.4 边界与坑

- 校验标准模糊会导致"永远不通过"或"轻易通过"。
- 每轮都累积上下文，轮次多时 token 成本陡增。
- 循环内的副作用（写库、发消息）可能被重复执行，需幂等。

## 5.5 生产建议

- 始终设置 `maxIterations` 与单轮超时。
- 让校验 Agent 输出结构化结论，便于程序判定。
- 循环内副作用操作加幂等键。

# 6. 工作流组合与自定义 Agent

真实系统往往是多种模式的嵌套组合，必要时还可实现自定义 Agent。

## 6.1 结论先行

- 工作流 Agent 可互相嵌套：顺序里放循环、并行里放顺序。
- 组合时最关心"数据如何流动"与"失败如何传播"。
- 内置模式不够用时，可实现自定义 Agent 扩展。

## 6.2 机制拆解

### 6.2.1 组合模式

| 模式 | 说明 | 例子 |
|------|------|------|
| Sequential | 顺序执行 | 研究→写作→审核 |
| Parallel | 并行执行 | 风控∥授信 |
| Loop | 循环执行 | 生成↔校验 |
| LlmAgent | 模型决策 | 意图分诊 |

### 6.2.2 组合示例

```
Sequential
  |-- Parallel (risk ∥ credit)
  |-- LlmAgent (决策)
  +-- Loop (draft <-> review)
```

- 嵌套越深，可观测与调试越难，需配套 Trace。

### 6.2.3 自定义 Agent

```java
// 伪代码：实现自定义控制流
public class RouterAgent extends BaseAgent {
    @Override
    protected Flowable<Event> runAsync(InvocationContext ctx) {
        String intent = classify(ctx.userMessage());
        Agent target = switch (intent) {
            case "order" -> orderAgent;
            case "refund" -> refundAgent;
            default -> fallbackAgent;
        };
        return target.runAsync(ctx);
    }
}
```

- 自定义 Agent 适合内置模式无法表达的控制流（如规则+模型混合路由）。

## 6.3 代码示例

```java
ParallelAgent checks = ParallelAgent.builder()
        .name("checks")
        .subAgents(riskAgent, creditAgent)
        .build();

LoopAgent refine = LoopAgent.builder()
        .name("refine")
        .subAgents(draftAgent, reviewAgent)
        .maxIterations(3)
        .build();

SequentialAgent flow = SequentialAgent.builder()
        .name("approval-flow")
        .subAgents(checks, decisionAgent, refine)
        .build();
```

## 6.4 边界与坑

- 深层嵌套使事件流冗长，前端渲染与排障成本上升。
- 不同子树的 State key 命名冲突会导致数据串扰。
- 失败传播规则需事先定义，否则错误被静默吞掉。

## 6.5 生产建议

- 为组合流程画数据流图，标注 State key 与失败策略。
- 统一 key 命名规范（如 `<阶段>.<字段>`）。
- 端到端 Trace 覆盖整个嵌套树。

# 7. 生产注意事项

多 Agent 系统的生产风险集中在成本、失控与可观测性。

## 7.1 结论先行

- 委托深度、循环次数、并发度都必须设上限。
- 每个 Agent 都是成本与延迟的乘数，拆分需有收益。
- 委托链必须可观测，否则无法定位路由错误。

## 7.2 风险与对策

| 风险 | 影响 | 建议 |
|------|------|------|
| 无限委托 | 成本失控 | 深度限制 |
| 循环无终止 | 死循环 | maxIterations |
| Agent 过多 | 成本高 | 限制数量 |
| 无 Trace | 难排障 | 记录委托链 |
| 越权 | 安全 | Agent 级授权 |

## 7.3 成本控制

| 手段 | 说明 |
|------|------|
| 减少 Agent 数 | 合并同域 |
| 小模型分诊 | 路由用快模型 |
| 结果缓存 | 相同输入复用 |
| 上下文裁剪 | 只传必要字段 |

## 7.4 可观测

```java
runner.runStreaming(session, input)
        .doOnNext(e -> trace.record(e))
        .subscribe();
```

| 记录项 | 说明 |
|--------|------|
| 委托链 | 谁转给谁 |
| 路由原因 | 命中的 description |
| 各 Agent 耗时 | 定位瓶颈 |
| 各 Agent token | 成本归因 |

## 7.5 授权

- 每个 Agent 只注册其职责所需的工具。
- 高风险子 Agent 加审批或人工确认。
- 记录跨 Agent 的数据访问，满足合规。

## 7.6 灰度与回滚

| 阶段 | 动作 | 观察 |
|------|------|------|
| 影子 | 只记录委托决策 | 与线上差异 |
| 小流量 | 放量 5% | 路由准确率 |
| 扩量 | 逐步 50% | 成本、延迟 |
| 全量 | 100% | 稳定性 |

- 委托拓扑、子 Agent 指令任一变更都应灰度，并保留上一版拓扑以便回滚。

# 8. 小结

## 8.1 核心要点回顾

- ADK 用 `subAgents` 实现多 Agent，父 Agent 按 `description` 委托。
- 提供 Sequential/Parallel/Loop 工作流 Agent，可嵌套组合。
- 顺序靠 State 传值，并行需独立 key，循环必须设终止条件。
- 必须限制委托深度与循环次数，控制成本。
- 委托链需 Trace 记录，便于排障与优化。
- 组合优于堆砌：优先用工作流 Agent 表达确定性流程。
- 拆分收益必须大于其带来的成本与延迟。
- 组合模式应优先于自定义控制流，降低维护成本。
- 多 Agent 的失败传播与回滚策略需提前定义。

## 8.2 生产检查清单

- [ ] Agent 按业务域拆分且命名规范。
- [ ] 每个子 Agent 有清晰 description 与独立工具。
- [ ] 委托深度与 Agent 数量有上限。
- [ ] 循环设置 maxIterations 与超时。
- [ ] 并行分支使用独立 State key。
- [ ] 委托链与各 Agent 成本可观测。
- [ ] 失败传播与回滚策略已定义。
