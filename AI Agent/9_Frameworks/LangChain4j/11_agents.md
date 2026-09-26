# 1. Agent 的本质

结论先行：在 LangChain4j 中，Agent = **LLM + 工具 + 记忆 + 循环控制**。它不是一种全新的模型，而是一套让模型能够"多步自主决策"的编排模式。最基础的形式就是"带工具的 AI Service"。

## 1.1 定义：LLM + 工具 + 记忆 + 循环

| 组成 | 作用 | 缺失后果 |
|------|------|----------|
| LLM | 决策与生成 | 无法推理 |
| 工具（Tool） | 与外部世界交互 | 只能空谈，无法行动 |
| 记忆（Memory） | 保留对话与中间结果 | 多轮即失忆 |
| 循环控制 | 多步执行与终止 | 单步即返回或死循环 |

四个要素缺一不可。工具让模型能"做事"，记忆让模型能"记住做过什么"，循环让模型能"一步步做下去"，而终止条件让循环能"停下来"。

## 1.2 最简 Agent：带工具的 AI Service

```java
interface Assistant {
    String chat(String message);
}

Assistant agent = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new OrderTools(), new WeatherTools())
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
        .build();

String result = agent.chat("帮我查订单 A123 的物流，如果超时就催单");
```

框架内部执行：

```
思考 → 调用工具 → 观察结果 → 再思考 → ... → 输出最终答案
```

上面这段代码里，`AiServices` 自动完成了"消息累积、工具调用、结果回填、再次调用模型"的整个循环，开发者只写了一个接口方法。

## 1.3 与普通工具调用的区别

> 与 `6_Tools` 的区别：Agent 强调**自主决策与多步执行**，工具只是其能力之一。

普通工具调用（tool calling）是"一次模型请求 → 一次工具执行 → 一次回答"，轮数固定；Agent 是"模型自己决定还要不要继续"，轮数动态。因此 Agent 的核心风险也来自这种动态性：可能停不下来，也可能过早停下。

## 1.4 何时该用 Agent

Agent 不是默认选择。单步就能完成的任务，用普通 AI Service 更便宜、更稳定。

| 任务特征 | 推荐方案 |
|----------|----------|
| 一问一答、无需外部数据 | 纯 LLM / AI Service |
| 需要一次外部查询 | 单轮工具调用 |
| 需要多步、结果决定下一步 | Agent |
| 步骤固定且可预先列出 | Plan-and-Execute |
| 高风险、需人工确认 | HITL Agent |

判断标准很简单：**如果任务无法预先写出确定步骤，且下一步依赖上一步的结果，才需要 Agent。**

# 2. 经典 Agent 循环

结论先行：所有 Agent 框架（无论 LangChain4j、LangGraph 还是自研）共享同一个内核——一个带终止条件的"决策-执行"循环。

## 2.1 循环四阶段

```
┌──────────────────────────────────────┐
│  1. 接收目标                          │
│  2. LLM 决策：调用工具 / 直接回答      │
│  3. 若调用工具 → 执行 → 结果回填       │
│  4. 回到 2，直到：                     │
│     - 生成最终答案                     │
│     - 达到最大轮数                     │
│     - 触发终止条件                     │
└──────────────────────────────────────┘
```

终止条件必须显式设置，否则可能死循环。

### 2.1.1 决策阶段

模型读取当前全部消息，判断是"调用工具"还是"直接回答"。这个判断依赖模型的函数调用（function calling）能力。

### 2.1.2 执行阶段

若模型请求调用工具，框架执行对应 Java 方法，并把返回值封装成工具结果消息追加到历史中，随后进入下一轮。

## 2.2 终止条件

终止条件至少有四类，生产系统应同时配置：

### 2.2.1 硬终止

轮数上限、总超时、预算上限。这类条件由框架或外层强制，不依赖模型配合，是防死循环的底线。

### 2.2.2 软终止

模型不再请求工具、或输出约定的结束标记。这类条件依赖模型"自觉"，必须与硬终止配合使用。

| 类型 | 触发 | 说明 |
|------|------|------|
| 自然终止 | 模型不再请求工具 | 正常完成 |
| 轮数上限 | 达到 maxIterations | 防死循环 |
| 超时 | 总耗时超过阈值 | 防长尾 |
| 业务终止 | 命中特定条件 | 如预算耗尽 |

## 2.3 循环的变体

- **ReAct**：思考（Reason）与行动（Act）交替，最经典。
- **Plan-and-Execute**：先规划全部步骤，再逐步执行，轮数更可控。
- **Reflexion**：执行后自我批判并重试，提升复杂任务成功率。
- **多 Agent 协作**：多个角色各司其职，通过消息协作完成任务。

# 3. Agentic 模块（1.x）

结论先行：LangChain4j 1.x 引入 `langchain4j-agentic` 模块，把 Agent 从"AI Service 的一种用法"升级为一等抽象，并提供声明式定义与组合编排能力。

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-agentic</artifactId>
</dependency>
```

## 3.1 声明式 Agent

```java
interface WeatherAgent {
    @UserMessage("查询 {{city}} 的天气")
    String weather(@V("city") String city);
}

WeatherAgent agent = AgenticServices.agentBuilder(WeatherAgent.class)
        .chatModel(model)
        .tools(new WeatherTools())
        .build();
```

用接口 + 注解声明 Agent 的输入与职责，`AgenticServices.agentBuilder` 负责把模型、工具、记忆组装起来。相比 `AiServices`，Agentic 模块更强调 Agent 作为"可组合单元"的定位。

### 3.1.1 接口即契约

接口方法名与注解共同定义了 Agent 的职责边界。`@UserMessage` 是模板，`@V` 绑定参数，模型据此理解要做什么。

### 3.1.2 构建过程

`agentBuilder` 返回一个 fluent builder，可链式配置模型、工具、记忆与输出解析器，最后 `build()` 生成代理对象。

### 3.1.3 与 AiServices 的关系

`AgenticServices.agentBuilder` 在底层同样基于 `AiServices` 的机制，只是额外暴露了编排与组合能力，二者并不冲突。

## 3.2 组合 Agent

```java
interface TravelPlanner {
    @UserMessage("""
            为 {{city}} 制定旅行计划：
            1. 先查询天气
            2. 再推荐室内/室外活动
            """)
    String plan(@V("city") String city);
}
```

组合 Agent 的 Prompt 中显式描述步骤，把一个复杂目标拆成若干子目标，再借助工具或其他 Agent 完成。

## 3.3 编排模式

> Agentic 模块还提供顺序、并行、循环、条件等编排模式，详见官方文档与 `10_MultiAgent`。

| 模式 | 语义 | 典型场景 |
|------|------|----------|
| 顺序（sequence） | 依次执行 | 规划 → 执行 → 汇总 |
| 并行（parallel） | 同时执行 | 多源检索 |
| 循环（loop） | 重复直到满足条件 | 迭代精修 |
| 条件（conditional） | 按判断分支 | 路由不同 Agent |

# 4. 手写 ReAct Agent（理解原理）

结论先行：框架封装了循环，但理解原理是排查问题的前提。下面这段约 40 行的代码揭示了所有 Agent 框架的共同内核：**消息累积 + 工具调用 + 循环 + 终止条件**。

## 4.1 完整实现

```java
public class ReActAgent {

    private final ChatModel model;
    private final List<Object> tools;
    private final int maxIterations;

    public String run(String goal) {
        List<ChatMessage> messages = new ArrayList<>();
        messages.add(SystemMessage.from("""
                你可以使用工具完成任务。
                每一步先说明思考，再决定是否调用工具。
                任务完成后输出"最终答案："。
                """));
        messages.add(UserMessage.from(goal));

        for (int i = 0; i < maxIterations; i++) {
            ChatResponse response = model.chat(ChatRequest.builder()
                    .messages(messages)
                    .toolSpecifications(ToolSpecifications
                            .toolSpecificationsFrom(tools.toArray()))
                    .build());

            AiMessage ai = response.aiMessage();
            messages.add(ai);

            // 无工具调用 → 任务结束
            if (!ai.hasToolExecutionRequests()) {
                return ai.text();
            }

            // 执行工具
            for (ToolExecutionRequest req : ai.toolExecutionRequests()) {
                String result = execute(req);
                messages.add(ToolExecutionResultMessage.from(req, result));
            }
        }
        return "达到最大迭代次数，未能完成任务";
    }
}
```

## 4.2 代码逐段拆解

| 片段 | 作用 |
|------|------|
| `messages` 列表 | 累积全部历史，是"记忆"的载体 |
| `toolSpecificationsFrom` | 把 Java 方法暴露为模型可调用的工具 |
| `hasToolExecutionRequests()` | 判断是否继续循环的天然信号 |
| `ToolExecutionResultMessage` | 把工具结果回填给模型 |
| `maxIterations` | 兜底终止条件 |

### 4.2.1 消息累积

每一轮都把模型的 `AiMessage` 与工具的 `ToolExecutionResultMessage` 追加到同一个列表，模型因此能看到"我调用了什么、得到了什么"。列表只增不减，这也是上下文膨胀的根源。

### 4.2.2 工具结果回填

`ToolExecutionResultMessage.from(req, result)` 必须携带原始请求，模型据此把结果与先前的调用对应起来。若工具抛异常，应把异常信息作为结果文本回填，让模型自行决定重试或换策略，而不是直接中断循环。

## 4.3 与框架的对应关系

`AiServices` 内部的 `DefaultAiServices` 做的正是这件事，只是额外处理了记忆窗口、工具异常、流式输出、内容审核等细节。理解手写版后，再读框架源码或排查"为什么工具没被调用"就轻松得多。

## 4.4 工具执行器的实现

手写版中最容易忽略的是 `execute(req)`：它要按名称找到 Java 方法、反序列化参数、调用并序列化结果。

```java
private String execute(ToolExecutionRequest req) {
    try {
        Object tool = findToolByName(req.name());
        Method method = resolveMethod(tool, req.name());
        Object[] args = objectMapper.readValue(
                req.arguments(), method.getParameterTypes());
        Object result = method.invoke(tool, args);
        return objectMapper.writeValueAsString(result);
    } catch (Exception e) {
        return "工具执行失败: " + e.getMessage();
    }
}
```

框架的 `DefaultToolExecutor` 封装了这一过程，包括参数类型转换与错误处理。自研时务必对参数做校验，防止模型传入越界值。

# 5. 规划与任务分解

结论先行：对于步骤明确的任务，先规划再执行（Plan-and-Execute）比边想边做更稳定、更省钱；对高度不确定的任务，则需要动态重规划。

## 5.1 Plan-and-Execute

```java
interface Planner {
    @UserMessage("""
            把下面的任务拆解为有序步骤，每步一个动作：
            任务：{{it}}
            输出 JSON 数组：["步骤1", "步骤2", ...]
            """)
    List<String> decompose(String task);
}

interface Executor {
    @UserMessage("执行这一步并返回结果：{{it}}")
    String execute(String step);
}

public String solve(String task) {
    List<String> steps = planner.decompose(task);
    List<String> results = new ArrayList<>();
    for (String step : steps) {
        results.add(executor.execute(step));
    }
    return summarizer.summarize(results);
}
```

规划器只调用一次模型产出步骤列表，执行器逐步调用，最后汇总。这种结构让 Token 消耗可预估，也便于对每一步单独加重试与超时。

### 5.1.1 规划阶段

规划器把模糊目标转成有序、可执行的步骤列表。输出结构（这里是 JSON 数组）越明确，后续解析越稳定。

### 5.1.2 执行阶段

执行器逐条执行步骤，每步独立调用，可分别设置重试、超时与工具集。步骤之间通过结果传递上下文。

## 5.2 动态重规划

> Plan-and-Execute 模式适合步骤明确的任务；复杂任务需要动态重规划。详见 `6_AgentCore/5_planning.md`。

当某一步执行失败或结果与预期不符时，应把失败信息回传给规划器，触发重新规划，而非机械地继续执行后续步骤。

## 5.3 适用边界

| 任务特征 | 推荐模式 |
|----------|----------|
| 步骤固定、依赖清晰 | Plan-and-Execute |
| 步骤随结果变化 | ReAct / 动态重规划 |
| 需要高质量最终产物 | Reflexion |

# 6. 反思与自我批判

结论先行：Reflexion（反思）通过"生成 → 批判 → 改进"的闭环，显著提升复杂任务的成功率，代价是额外的模型调用成本。

## 6.1 Reflexion 模式

```java
interface Critic {
    @UserMessage("""
            评估下面的回答质量，指出问题并给出改进建议：
            问题：{{question}}
            回答：{{answer}}
            """)
    String critique(@V("question") String q, @V("answer") String a);
}

public String refinedAnswer(String question) {
    String answer = agent.chat(question);
    String critique = critic.critique(question, answer);
    return agent.chat("根据反馈改进回答：\n反馈：" + critique);
}
```

> Reflexion 模式能显著提升复杂任务成功率。详见 `6_AgentCore/8_reflection_and_self_critique.md`。

## 6.2 反思的成本与边界

反思本质是"用更多 Token 换质量"。对简单问答是浪费，对代码生成、长文写作、复杂推理才有收益。生产上应按任务类型开关，并限制反思轮数（通常 1~2 轮即边际收益递减）。

### 6.2.1 收益递减

第一轮反思通常带来最大提升，第二轮次之，第三轮后往往只增加成本。因此应设固定上限，而非"反思到满意为止"。

### 6.2.2 开关策略

| 任务类型 | 是否反思 |
|----------|----------|
| 简单问答 | 否 |
| 代码生成 | 是（1 轮） |
| 长文写作 | 是（1~2 轮） |
| 高风险决策 | 是 + 人工复核 |

# 7. 人工介入（HITL）

结论先行：HITL（Human-in-the-Loop，人在回路）是高风险 Agent 的安全阀。凡涉及资金、删除、对外发送等不可逆动作，必须设置人工审批点。

## 7.1 审批点设计

```java
public String executeWithApproval(String task) {
    String plan = planner.decompose(task).toString();

    if (requiresApproval(plan)) {
        // 挂起，等待人工确认（通过数据库/消息队列）
        approvalService.requestApproval(taskId, plan);
        return "已提交审批，请等待确认";
    }
    return executor.execute(plan);
}
```

### 7.1.1 风险分级

不是所有动作都需要审批。按可逆性与影响面分级，只对高风险动作设卡：

| 等级 | 示例 | 处理 |
|------|------|------|
| 低 | 查询、计算 | 直接执行 |
| 中 | 写草稿、发内部通知 | 记录 + 事后审计 |
| 高 | 转账、删除、对外发送 | 人工审批 |

### 7.1.2 审批触发

`requiresApproval(plan)` 应基于结构化信息判断（动作类型、金额、目标系统），而非让模型自述"这是高风险"。判断逻辑必须是确定性的代码。

## 7.2 挂起与恢复

> 涉及资金、删除、对外发送等高风险动作时，必须有人工审批点。详见 `6_AgentCore/11_human_in_the_loop.md`。

审批流程通常需要把 Agent 状态持久化（消息列表、已执行步骤），审批通过后再从断点恢复，而不是重新执行整个任务。

## 7.3 审批与审计

审批记录本身也是审计资料：谁在何时批准了什么计划、依据是什么。生产上应把审批流与 Agent 轨迹（trace）关联，形成"决策 → 审批 → 执行"的完整证据链。

# 8. Agent 的失败模式

结论先行：Agent 的失败不是偶发异常，而是可枚举的模式。识别模式并针对性防御，比事后调试更有效。

## 8.1 失败模式总览

| 模式 | 表现 | 对策 |
|------|------|------|
| 无限循环 | 反复调用同一工具 | 最大轮数 + 终止条件 |
| 目标漂移 | 偏离原始任务 | 每轮重申目标 |
| 工具误用 | 调错工具/参数 | 优化描述 + 参数校验 |
| 幻觉工具 | 调用不存在的工具 | 工具集固定 + 白名单 |
| 过早终止 | 未完成就作答 | 明确完成标准 |
| 上下文爆炸 | 消息越积越多 | 记忆窗口 + 压缩 |

> 详见 `6_AgentCore/13_failure_modes.md`。

## 8.2 典型反例

### 8.2.1 无限循环

模型把工具返回的空结果当作"没查到"，于是换个措辞重复查询，永远得不到答案。对策是给同一工具的连续调用计数，超过阈值强制终止。

### 8.2.2 目标漂移

用户问"退款进度"，Agent 查到物流后顺手回答了物流详情，忘记退款。对策是在系统提示中固定目标，并在每轮重申。

### 8.2.3 幻觉工具

模型臆造一个 `sendEmail` 工具并"调用"，实际无此工具。对策是工具集固定，模型请求未知工具时直接返回错误而非静默忽略。

## 8.3 防御清单

1. 必设最大迭代次数。
2. 每次工具调用记录名称、参数、结果。
3. 工具参数做类型与范围校验。
4. 明确"完成标准"并写入系统提示。
5. 记忆设窗口，超长时压缩或摘要。

## 8.4 上下文爆炸的治理

Agent 每轮都追加消息，Token 随轮数线性增长。治理手段有三层：

```
消息列表持续增长
   │
   ├─ 窗口裁剪：只保留最近 N 条（简单，但会丢早期信息）
   ├─ 摘要压缩：把早期消息总结成一段（保信息，但需额外调用）
   └─ 结构化记忆：把关键事实抽出存外部（最灵活，实现最重）
```

生产常用"窗口裁剪 + 摘要"组合：近期消息原样保留，远期消息压缩为摘要。

# 9. 生产注意事项

结论先行：生产 Agent 的难点不是"能不能跑通"，而是终止条件、鉴权、超时与可观测这四件事。

## 9.1 安全与权限

| 风险 | 建议 |
|------|------|
| 工具无鉴权 | 工具内部校验权限 |
| 高风险动作 | 人工审批 |
| 越权访问数据 | 工具按当前用户上下文过滤 |

工具是 Agent 与外部系统的唯一通道，权限校验必须落在工具内部，不能依赖模型"自觉"。

## 9.2 稳定性

| 风险 | 建议 |
|------|------|
| 无轮数上限 | 必设最大迭代次数 |
| 无超时 | 每轮调用与工具都设超时 |
| 工具异常 | 捕获后作为观察结果回填，而非抛出中断循环 |

## 9.3 成本与可观测

| 风险 | 建议 |
|------|------|
| 无 Trace | 记录每轮决策与工具调用 |
| 成本失控 | 监控 Token、限制轮数 |
| 无法复盘 | 持久化完整消息轨迹 |

## 9.4 部署形态

Agent 通常是有状态的长任务，部署时要注意：

| 形态 | 特点 | 适用 |
|------|------|------|
| 同步请求 | 简单，但长任务易超时 | 短任务 |
| 异步任务 | 提交后轮询/回调 | 中等任务 |
| 工作流引擎 | 状态可持久化、可恢复 | 长任务 + HITL |
| 事件驱动 | 与消息队列集成 | 高并发 |

涉及 HITL 的 Agent 必须把状态外置（数据库/工作流引擎），否则进程重启即丢失待审批任务。

# 10. 小结

- Agent = LLM + 工具 + 记忆 + 循环。
- 最简 Agent 就是"带工具的 AI Service"。
- `langchain4j-agentic` 提供声明式 Agent 与编排。
- 手写 ReAct 循环能帮你理解所有框架的内核。
- 生产 Agent 的难点是**终止条件、鉴权、超时与可观测**。
