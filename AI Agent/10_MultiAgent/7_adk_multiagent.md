# 1. 委托机制

**ADK**（Agent Development Kit，智能体开发套件）是 Google 推出的 Agent 框架，其 Java 版把多 Agent 协作抽象为两种机制：**委托（Delegation）**与**工作流 Agent（Workflow Agent）**。委托用于"根 Agent 把任务转交给子 Agent"，工作流 Agent 用于"用顺序/并行/循环把多个 Agent 固定组合起来"。

```
Root Agent 根据 description 决定委托给哪个子 Agent
```

## 1.1 什么是委托

结论先行：委托是一种"由 LLM 在运行时决定转交对象"的动态路由。根 Agent 看到子 Agent 的名称与描述后，把当前请求连同上下文交给最合适的子 Agent 处理。

委托的三个要素：

| 要素 | 说明 |
|------|------|
| 子 Agent 列表 | 通过 `subAgents` 挂载 |
| 描述 | 子 Agent 的 `description`，供根 Agent 判断 |
| 转交动作 | 运行时由模型决定是否转交、转交给谁 |

与工作流 Agent 的区别：

| 维度 | 委托 | 工作流 Agent |
|------|------|-------------|
| 谁决定顺序 | LLM | 开发者 |
| 可预测性 | 中 | 高 |
| 适用 | 意图分发 | 固定流程 |

## 1.2 定义子 Agent

```java
LlmAgent orderAgent = LlmAgent.builder()
        .name("order-agent")
        .description("处理订单查询、取消、催单")
        .model("gemini-2.0-flash")
        .tools(new OrderTools())
        .build();

LlmAgent refundAgent = LlmAgent.builder()
        .name("refund-agent")
        .description("处理退款申请与查询")
        .model("gemini-2.0-flash")
        .tools(new RefundTools())
        .build();
```

字段职责：

| 字段 | 作用 | 建议 |
|------|------|------|
| name | 唯一标识 | 用短横线小写 |
| description | 委托判断依据 | 写清边界与输入输出 |
| model | 使用的模型 | 简单任务可用小模型 |
| tools | 可调用的工具 | 最小权限 |
| instruction | 系统指令 | 明确角色与输出格式 |

## 1.3 根 Agent 与委托触发

```java
LlmAgent rootAgent = LlmAgent.builder()
        .name("root")
        .description("客服总管，负责分发")
        .model("gemini-2.0-flash")
        .instruction("根据用户问题，转交给合适的子 Agent。")
        .subAgents(orderAgent, refundAgent)
        .build();
```

### 1.3.1 description 的作用

`description` 不是给人看的注释，而是**给模型看的路由依据**。它决定了委托的准确率。

好的描述：

```
处理订单查询、取消、催单。输入：订单号或用户 ID。不处理退款。
```

差的描述：

```
订单相关。
```

对比：

| 描述质量 | 模型行为 | 结果 |
|---------|---------|------|
| 具体、有边界 | 明确命中 | 委托准确 |
| 模糊、重叠 | 反复犹豫或误选 | 委托错误 |

### 1.3.2 委托的运行时行为

```
用户请求
   │
   ▼
┌──────────┐   读取 subAgents 的 name/description
│   root   │────────────────────────────────┐
└────┬─────┘                                │
     │ 决定委托                              │
     ▼                                       │
┌──────────────┐   执行并返回结果            │
│ order-agent  │────────────────────────────┘
└──────────────┘
     │
     ▼
  最终回答
```

> 边界：委托是"模型自主决策"，不是确定性的 if-else。若业务上必须精确路由，应先做规则预筛或直接用工作流 Agent。

# 2. 工作流 Agent

结论先行：工作流 Agent 是"确定性的多 Agent 组合器"，它把多个子 Agent 按固定顺序、并行或循环方式执行，不依赖模型决定流程。

## 2.1 顺序 Agent

```java
SequentialAgent pipeline = SequentialAgent.builder()
        .name("research-pipeline")
        .subAgents(searchAgent, analyzeAgent, writeAgent)
        .build();
```

特点：

- 子 Agent 按声明顺序依次执行。
- 前一个的输出通过会话状态传给后一个。
- 任一子 Agent 失败，通常中止后续执行（可按需配置）。

适用：流水线式任务，如"检索 → 分析 → 写作"。

## 2.2 并行 Agent

```java
ParallelAgent parallel = ParallelAgent.builder()
        .name("parallel-research")
        .subAgents(agentA, agentB, agentC)
        .build();
```

特点：

- 子 Agent 并发执行，缩短总延迟。
- 各分支应写入**不同的状态键**，避免覆盖。
- 需等待所有分支完成后才继续。

| 场景 | 是否适合并行 |
|------|-------------|
| 多源检索 | 适合 |
| 互不依赖的分析 | 适合 |
| 有先后依赖的步骤 | 不适合 |
| 共享配额的下游 | 需限流 |

## 2.3 循环 Agent

```java
LoopAgent loop = LoopAgent.builder()
        .name("refine-loop")
        .subAgents(generateAgent, reviewAgent)
        .maxIterations(3)
        .build();
```

### 2.3.1 maxIterations 与退出

循环的终止条件有两类：

| 终止方式 | 说明 | 是否可靠 |
|---------|------|---------|
| 达到 maxIterations | 硬上限 | 可靠 |
| 子 Agent 发出退出信号 | 提前结束 | 依赖实现 |
| 质量达标 | 由评审判断 | 需显式信号 |

```java
// 伪代码：评审 Agent 判定达标后请求退出循环
if (reviewResult.passed()) {
    events.actions.escalate();   // 请求结束当前循环
}
```

> 边界：`maxIterations` 是必需的兜底。只依赖"质量达标"退出，遇到模型判断不稳时会无限循环。

### 2.3.2 循环中的状态

循环体每轮都会读写状态，需注意：

- 累积型字段（如修改历史）会随轮次增长，应设置上限。
- 每轮应覆盖"当前版本"，而不是不断追加。
- 轮次号写入状态，便于日志与断点。

# 3. 组合使用

结论先行：委托与工作流 Agent 可以任意嵌套，用工作流保证骨架稳定，用委托提供局部灵活性。

## 3.1 嵌套工作流

```java
// 复杂流程：并行研究 → 顺序写作 → 循环优化
ParallelAgent research = ParallelAgent.builder()
        .subAgents(webSearchAgent, docSearchAgent)
        .build();

SequentialAgent write = SequentialAgent.builder()
        .subAgents(outlineAgent, draftAgent)
        .build();

LoopAgent refine = LoopAgent.builder()
        .subAgents(reviewAgent, reviseAgent)
        .maxIterations(2)
        .build();

SequentialAgent workflow = SequentialAgent.builder()
        .name("full-workflow")
        .subAgents(research, write, refine)
        .build();
```

结构示意：

```
full-workflow (Sequential)
├── research (Parallel)
│   ├── webSearchAgent
│   └── docSearchAgent
├── write (Sequential)
│   ├── outlineAgent
│   └── draftAgent
└── refine (Loop, maxIterations=2)
    ├── reviewAgent
    └── reviseAgent
```

## 3.2 混合委托与工作流

在固定骨架中嵌入一个"主管 Agent"，让它在其职责范围内动态选择子 Agent：

```java
LlmAgent triageAgent = LlmAgent.builder()
        .name("triage")
        .description("根据问题类型选择处理分支")
        .subAgents(faqAgent, orderAgent, refundAgent)
        .build();

SequentialAgent service = SequentialAgent.builder()
        .name("customer-service")
        .subAgents(triageAgent, followUpAgent)
        .build();
```

好处：外层顺序可控，内层分发灵活。

## 3.3 组合的边界

| 风险 | 说明 | 建议 |
|------|------|------|
| 层级过深 | 难以调试 | 一般不超过 3 层 |
| 状态键冲突 | 并行分支互相覆盖 | 命名空间隔离 |
| 错误传播 | 一个分支失败拖垮整体 | 设置分支级容错 |
| 成本叠加 | 嵌套导致调用数翻倍 | 评估每层必要性 |

> 边界：组合能力越强，越容易写出"看不懂的树"。务必为每个工作流写明职责与输入输出契约。

# 4. 状态共享

结论先行：ADK 的 Agent 之间不直接传参，而是通过**会话状态（Session State）**共享数据；状态键即隐式接口，需要像 API 一样管理。

## 4.1 会话状态

```java
// Agent 间通过会话状态共享
session.state().put("researchData", data);

// 子 Agent 读取
String data = (String) session.state().get("researchData");
```

状态是跨 Agent 的"共享内存"，读写时机：

| 时机 | 操作 |
|------|------|
| 工具返回后 | 写入结果 |
| Agent 生成后 | 写入输出 |
| 委托前 | 写入交接信息 |
| 最终输出前 | 读取汇总 |

## 4.2 状态命名与隔离

建议使用前缀隔离不同分支：

```java
session.state().put("research.web.result", webData);
session.state().put("research.doc.result", docData);
```

| 命名方式 | 示例 | 评价 |
|---------|------|------|
| 扁平名 | result | 易冲突 |
| 前缀隔离 | research.web.result | 推荐 |
| 嵌套对象 | research -> {web, doc} | 类型安全但复杂 |

## 4.3 状态生命周期

- 会话级：随会话存在，跨多轮对话保留。
- 调用级：单次请求内有效，请求结束清理。
- 临时级：仅在某个工作流内有效。

生产建议：

- 明确每个键的生命周期，避免"上一轮的数据污染本轮"。
- 大对象外置存储，状态里放引用。
- 敏感数据不写入会话状态，或在会话结束时清理。

# 5. 委托链 Trace

结论先行：委托是动态的，没有 Trace 就无法回答"为什么交给了这个 Agent"。至少记录委托决策链与工具调用。

## 5.1 Trace 结构

```
Trace: 用户请求
├── Span: root（决策委托）
├── Span: order-agent
│   ├── Span: 工具调用 queryOrder
│   └── Span: 生成回答
└── Span: root（汇总）
```

## 5.2 关键指标

| 指标 | 含义 | 异常信号 |
|------|------|---------|
| 委托深度 | 链路层数 | 接近深度上限 |
| 委托次数 | 单请求转交次数 | 频繁来回转交 |
| 子 Agent 耗时 | 各分支 P95 | 某分支偏慢 |
| 工具失败率 | 工具调用异常 | 外部依赖不稳 |
| 循环轮次 | Loop 实际轮数 | 常达 maxIterations |

## 5.3 调试委托问题

常见问题与排查：

| 现象 | 可能原因 | 排查手段 |
|------|---------|---------|
| 委托到错误 Agent | description 重叠 | 查看决策日志 |
| 反复来回委托 | 描述无边界 | 限制委托深度 |
| 不委托直接回答 | instruction 未强调 | 明确转交规则 |
| 子 Agent 无输出 | 状态键不匹配 | 检查状态读写 |

> 边界：不要只看最终回答。委托错误往往在中间步骤，最终答案可能"看起来还行"但过程错误。

# 6. 回调与生命周期

结论先行：ADK 提供回调（Callback）钩子，让开发者在 Agent/模型/工具执行前后插入逻辑，用于日志、校验、脱敏与短路。

## 6.1 回调钩子

| 钩子 | 触发时机 | 典型用途 |
|------|---------|---------|
| beforeAgent | Agent 执行前 | 注入上下文、鉴权 |
| afterAgent | Agent 执行后 | 记录输出、脱敏 |
| beforeModel | 模型调用前 | 修改提示、预算检查 |
| afterModel | 模型调用后 | 记录用量、内容过滤 |
| beforeTool | 工具调用前 | 参数校验、权限 |
| afterTool | 工具调用后 | 结果缓存、错误包装 |

```java
// 伪代码：在模型调用前做预算检查
beforeModelCallback = (context, request) -> {
    if (context.usage().exceedsBudget()) {
        return shortCircuit("预算已用尽");
    }
    return null;   // 继续正常流程
};
```

## 6.2 使用场景

- 统一日志与 Trace 埋点，避免散落在各 Agent。
- 敏感信息脱敏（手机号、身份证、密钥）。
- 工具参数校验与权限检查。
- 成本预算与限流。
- 输出格式的最终校验与修复。

## 6.3 边界

> 边界：回调是"全局切面"，一旦抛异常可能影响所有 Agent。回调内应做好异常隔离，不能让日志失败拖垮主流程。

生产建议：回调保持轻量、幂等、无阻塞；耗时操作（如远程审计）异步化。

# 7. 会话、记忆与多轮

结论先行：ADK 的 Agent 活在"会话"里。会话承载历史与状态，记忆（Memory）承载跨会话的知识，多轮对话则要求委托决策考虑上下文。

## 7.1 会话与服务

一个典型 ADK 应用的组件：

| 组件 | 职责 | 是否持久化 |
|------|------|-----------|
| Agent | 决策与执行 | 否 |
| Session | 单次会话的历史与状态 | 可配置 |
| SessionService | 会话的增删查 | 是 |
| Memory | 跨会话长期记忆 | 是 |
| Runner | 驱动 Agent 与事件循环 | 否 |

```java
// 伪代码：创建 Runner 并驱动一次会话
Runner runner = Runner.builder()
        .agent(rootAgent)
        .sessionService(sessionService)
        .memoryService(memoryService)
        .build();

runner.runAsync(userId, sessionId, userMessage)
        .forEach(event -> log.info("事件：{}", event));
```

## 7.2 记忆与上下文

- 会话历史：当前会话的完整对话，供模型理解上下文。
- 长期记忆：跨会话的用户偏好、事实，需检索后注入。
- 工作状态：Agent 间共享的键值，见第 4 节。

上下文膨胀的处理：

| 策略 | 说明 |
|------|------|
| 截断 | 只保留最近 N 轮 |
| 摘要 | 把旧历史压缩为摘要 |
| 检索 | 按相关性召回历史片段 |
| 结构化 | 只保留关键事实 |

> 边界：把整个会话历史塞进每次模型调用，会导致成本与延迟线性上升，并稀释关键信息。

## 7.3 多轮中的委托

多轮对话里，委托决策要利用历史：

- 上一轮委托给谁、结果如何，应写入会话状态。
- 用户代词（"它""那个订单"）需要借助历史消解。
- 避免"每轮都从零开始判断"，可缓存上轮意图。

生产建议：为会话设置空闲超时与最大轮数，避免会话无限增长。

# 8. 完整示例：客服多 Agent

结论先行：把委托与工作流组合起来，即可实现一个可用的客服系统——根 Agent 负责分发，专业 Agent 负责处理，顺序工作流负责收尾。

## 8.1 目标与结构

```
customer-service (Sequential)
├── triage (LlmAgent, 委托)
│   ├── faq-agent      常见问题
│   ├── order-agent    订单查询/取消/催单
│   └── refund-agent   退款申请/查询
└── followup-agent     满意度回访与工单归档
```

## 8.2 Agent 定义

```java
LlmAgent faqAgent = LlmAgent.builder()
        .name("faq-agent")
        .description("回答产品与政策类常见问题，不处理订单与退款")
        .model("gemini-2.0-flash")
        .tools(new FaqTools())
        .build();

LlmAgent orderAgent = LlmAgent.builder()
        .name("order-agent")
        .description("处理订单查询、取消、催单。输入订单号或用户 ID")
        .model("gemini-2.0-flash")
        .tools(new OrderTools())
        .build();

LlmAgent refundAgent = LlmAgent.builder()
        .name("refund-agent")
        .description("处理退款申请与退款进度查询")
        .model("gemini-2.0-flash")
        .tools(new RefundTools())
        .build();

LlmAgent triage = LlmAgent.builder()
        .name("triage")
        .description("客服分诊，根据问题转交合适子 Agent")
        .instruction("先判断问题类型，再转交；无法判断时先向用户澄清。")
        .subAgents(faqAgent, orderAgent, refundAgent)
        .build();
```

## 8.3 运行与观察

```java
SequentialAgent service = SequentialAgent.builder()
        .name("customer-service")
        .subAgents(triage, followupAgent)
        .build();
```

观察重点：

| 观察项 | 期望 | 异常处理 |
|--------|------|---------|
| 委托命中率 | 高 | 优化 description |
| 澄清次数 | 合理 | 过高说明描述模糊 |
| 工单归档率 | 100% | 检查 followup 状态键 |
| 高风险动作 | 有审批 | 补充确认流程 |

# 9. 与其他框架对比

结论先行：ADK 的优势在于"官方工具链 + 工作流 Agent 原生支持"，劣势在于生态与部署绑定。

## 9.1 编排范式对比

| 框架 | 委托 | 工作流 | 状态共享 | 语言 |
|------|------|--------|---------|------|
| ADK | 原生 subAgents | Sequential/Parallel/Loop | Session State | Java/Python |
| LangGraph4j | 需自建 | 图 | State | Java |
| Spring AI Alibaba Graph | 需自建 | 图 | State | Java |
| Embabel | 规划器决定 | 无固定工作流 | AgentContext | JVM |

## 9.2 选型建议

- 需要开箱即用的顺序/并行/循环组合 → ADK。
- 需要精细控制每步流转、检查点、人工介入 → 图编排。
- 目标动态、路径需要自动搜索 → Embabel。
- 已在 Spring 生态 → Spring AI Alibaba Graph 亲和度更高。

> 结论：没有绝对最优，按"流程确定性"与"生态约束"两个维度选择。

# 10. 生产注意事项

结论先行：ADK 多 Agent 的生产风险集中在"无限委托、循环不终止、成本失控、缺乏可观测、权限过大"。

## 10.1 稳定性

| 风险 | 建议 |
|------|------|
| 无限委托 | 深度限制 |
| 循环无终止 | maxIterations |
| 成本高 | 限制 Agent 数 |
| 无 Trace | 记录委托链 |
| 权限 | Agent 级授权 |

补充：

- 子 Agent 异常要有兜底，避免整个工作流崩溃。
- 外部工具加超时与熔断。
- 并行分支限流，防止打爆下游。

## 10.2 成本与性能

- 简单子 Agent 用小模型，主管/根 Agent 用较强模型。
- 并行无依赖分支以降低延迟。
- 缓存工具结果与常见问答。
- 限制单请求的模型调用总数。

## 10.3 安全与权限

- 每个 Agent 按最小权限授予工具。
- 高风险工具（退款、删单）加二次确认或人工审批。
- 工具入参校验，防注入与越权。
- 输出做敏感信息过滤。

# 11. 小结

## 11.1 要点回顾

- ADK 用 `subAgents` 实现委托。
- 提供 Sequential/Parallel/Loop 工作流 Agent。
- 可组合出复杂流程。
- Agent 间通过会话状态共享数据。
- 需限制委托深度与循环次数。
- description 是委托准确率的关键。

## 11.2 落地清单

- [ ] 每个子 Agent 的 description 具体且有边界。
- [ ] 设置委托深度上限与 Loop 的 maxIterations。
- [ ] 并行分支使用前缀隔离的状态键。
- [ ] 明确状态键的生命周期，及时清理。
- [ ] 用回调统一 Trace、脱敏与预算检查。
- [ ] 每个 Agent 最小权限，高风险动作人工审批。
- [ ] 组合层级不超过 3 层，写清输入输出契约。
