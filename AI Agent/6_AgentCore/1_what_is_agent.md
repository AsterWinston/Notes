# 1. 定义

**Agent（智能体）= 以大模型为大脑，能自主感知、规划、调用工具、执行多步任务并达成目标的系统。**

它和"聊天机器人"的本质区别：**聊天机器人只回答问题，Agent 会为了目标采取行动。**

```
聊天机器人：用户问 → 模型答 → 结束
Agent：     用户给目标 → 模型规划 → 调用工具 → 观察结果
                      → 调整计划 → ... → 达成目标
```

## 1.1 一句话定义

Agent = **LLM（决策）+ 工具（行动）+ 记忆（状态）+ 规划（编排）**，在一个带终止条件的循环里自主推进任务。

- 它接收的是**目标**，而非固定问题；
- 它产出的是**行动序列 + 最终结果**，而非单段文本；
- 它的路径在**运行时**由模型决定，而非编码时固定。

## 1.2 与聊天机器人的本质区别

### 1.2.1 对照示例

| 场景 | 聊天机器人 | Agent |
|------|-----------|-------|
| "帮我查下 A123 订单" | 回答"请去官网查询" | 调用 `getOrder("A123")` 并返回结果 |
| "帮我退款" | 回答退款政策 | 校验资格 → 调用退款 → 反馈结果 |
| "分析上季度销售" | 给出分析框架 | 取数 → 计算 → 归因 → 成文 |

### 1.2.2 判定标准

一个系统是不是 Agent，看三点：

1. 是否有**目标驱动**（而非一问一答）；
2. 是否能**调用工具改变外部状态**；
3. 是否**多步自主推进**（路径非预先写死）。

## 1.3 Agent 的最小闭环

```
        目标
         │
         ▼
   ┌───────────┐
   │  决策/规划  │◀──────────┐
   └─────┬─────┘            │
         ▼                  │
   ┌───────────┐      ┌───────────┐
   │   行动     │─────▶│   观察     │
   │（工具调用） │      │（结果回填）│
   └───────────┘      └───────────┘
         │
         ▼
      达成目标？
```

这个闭环就是 Agent 的内核，后续章节（`4_agent_loop.md`）会详细展开。

## 1.4 常见误解澄清

### 1.4.1 误解一：Agent 就是加了插件（Plugin）的聊天机器人

聊天机器人加插件仍可能只是"一问一答 + 单次工具调用"。判定关键是**是否在循环中自主推进**：

- 单次调用工具后直接返回 → 仍属"增强型问答"；
- 调用工具后**根据结果决定下一步**，直到满足终止条件 → Agent。

### 1.4.2 误解二：Agent 必须由多 Agent（Multi-Agent，多智能体）协作

多 Agent 只是一种**编排形态**，不是 Agent 的定义。单 Agent + 多工具已能覆盖绝大多数生产场景；多 Agent 会引入通信开销、状态一致性与调试复杂度，应在确有分工必要时才用（见 `10_MultiAgent`）。

### 1.4.3 误解三：自主性越高越好

自主性越高，路径越不可预测、成本越难控制、审计越困难。生产系统往往**主动限制自主性**：

- 限定可用工具集合；
- 限定最大步数与预算；
- 高风险动作强制人工确认。

## 1.5 Agent 的自主性光谱

```
固定脚本 ── 单次 LLM ── 工作流+LLM节点 ── 受限 Agent ── 自主 Agent
   │           │             │                │              │
 无智能      无行动       部分自主          受控自主       高度自主
 高可控      高可控       较可控            平衡           难治理
```

> 选择落在光谱的哪一点，取决于**任务不确定性**与**失败代价**的权衡，而非"越右越好"。

# 2. Agent 的四大构成要素

```
              ┌──────────────────────┐
              │   LLM（大脑）         │
              │   推理 / 决策 / 生成   │
              └──────────┬───────────┘
                         │
      ┌──────────────────┼──────────────────┐
      ▼                  ▼                  ▼
┌──────────┐      ┌────────────┐     ┌────────────┐
│ 规划      │      │ 工具        │     │ 记忆        │
│ Planning │      │ Tools      │     │ Memory     │
└──────────┘      └────────────┘     └────────────┘
  任务分解          与外部交互          短期/长期
  步骤编排          读写世界            经验积累
```

| 要素 | 作用 | 缺失后果 |
|------|------|----------|
| LLM | 推理与决策 | 无法理解任务 |
| 工具 | 执行动作、获取信息 | 只能空谈，无法落地 |
| 记忆 | 保持上下文、积累经验 | 无法多轮、无法学习 |
| 规划 | 分解与编排任务 | 复杂任务无从下手 |

## 2.1 LLM（大脑）

LLM 负责"想"：理解目标、判断下一步、生成工具参数、汇总结果。它的能力决定了 Agent 的**上限**。

- 选型要点：推理能力、工具调用能力、上下文长度、成本与延迟；
- 详见 `2_LLM/Models/3_model_selection.md`。

## 2.2 工具（手脚）

工具负责"做"：查询数据、调用 API、执行代码、写文件。没有工具，Agent 只能"纸上谈兵"。

- 工具设计原则：描述清晰、参数明确、结果结构化；
- 详见 `7_ToolsAndProtocols/1_tool_design.md`。

## 2.3 记忆（经验）

记忆负责"记"：短期保存会话上下文，长期沉淀事实与经验。

- 短期记忆：最近若干轮消息；
- 长期记忆：向量库、知识图谱、用户画像；
- 详见 `8_Memory/1_memory_overview.md`。

## 2.4 规划（指挥）

规划负责"编排"：把复杂目标拆成可执行的步骤序列，并动态调整。

- 三种范式：ReAct、Plan-and-Execute、混合；
- 详见 `5_planning.md`。

## 2.5 四要素缺一不可

### 2.5.1 缺失后果对照

| 缺失 | 表现 |
|------|------|
| 无 LLM | 退化为固定脚本 |
| 无工具 | 只能生成文本，无法产生副作用 |
| 无记忆 | 每轮失忆，无法多轮/学习 |
| 无规划 | 复杂任务原地打转 |

### 2.5.2 四要素与失败模式

```
模型弱        → 决策错误、工具参数幻觉
工具差        → 调用失败、结果不可用
上下文脏/缺   → 目标漂移、答非所问
控制流缺失    → 死循环、成本失控
```

> 生产事故中，**大部分失败不是模型不够强，而是工具、上下文与控制流没设计好**。

## 2.6 四要素的依赖关系

四要素不是并列拼装，而是存在依赖：

```
        LLM（决策中枢）
         │    │     │
   ┌─────┘    │     └──────┐
   ▼          ▼            ▼
 规划       工具         记忆
   │          │            │
   └──────► 循环 ◄─────────┘
             （控制流）
```

- **规划**依赖 LLM 的推理，也依赖**工具清单**（步骤要可映射到工具）；
- **工具**的调用参数由 LLM 生成，结果回填到**记忆**；
- **记忆**为下一轮 LLM 决策提供上下文；
- 四者由**循环**串起来，缺少循环则退化为单次调用。

## 2.7 四要素协作的完整示例

### 2.7.1 场景描述

用户目标："帮我把 A123 订单的收货地址改成上海。"

### 2.7.2 代码

```java
public String handleGoal(String goal) {
    List<Message> memory = new ArrayList<>();          // 记忆
    memory.add(system("你是订单助手，可使用工具。"));

    Plan plan = planner.plan(goal);                     // 规划
    memory.add(assistant("计划：" + plan.steps()));

    for (int step = 0; step < MAX_STEPS; step++) {
        Response resp = llm.chat(memory, tools());      // LLM 决策
        memory.add(resp.message());

        if (!resp.hasToolCalls()) {
            return resp.text();
        }
        for (ToolCall call : resp.toolCalls()) {
            Object result = executeTool(call);          // 工具执行
            memory.add(toolResult(call.id(), result));  // 回填记忆
        }
    }
    return "超出最大步数";
}
```

### 2.7.3 四要素映射

| 代码元素 | 对应要素 |
|----------|----------|
| `llm.chat(...)` | LLM（决策） |
| `planner.plan(...)` | 规划（编排） |
| `executeTool(...)` | 工具（行动） |
| `memory` | 记忆（状态） |
| `for + MAX_STEPS` | 控制流（循环与终止） |

# 3. Agent 与"提示工程"的关系

## 3.1 控制粒度不同

```
提示工程 ──▶ 控制单次输出（怎么写提示）
Agent    ──▶ 控制多步行为（怎么完成任务）
```

| 维度 | 提示工程 | Agent |
|------|----------|-------|
| 作用范围 | 单次调用 | 多步循环 |
| 核心对象 | 提示文本 | 状态 + 工具 + 控制流 |
| 失败表现 | 答得不好 | 走错路、失控 |

## 3.2 能力叠加关系

Agent 是**提示工程 + 工具调用 + 循环控制**的组合，是更高阶的形态：

```
提示工程（基础）
   + 工具调用（扩展行动能力）
   + 循环控制（扩展多步能力）
   = Agent
```

## 3.3 提示工程做不到的事

提示工程能改善**单次输出**，但无法解决以下问题：

- **无法产生副作用**：再好的提示也不能真的下单、发邮件；
- **无法跨轮保持状态**：提示本身不携带执行进度；
- **无法自主决定"做多少步"**：步数由代码写死或由模型在一次输出内模拟；
- **无法在失败后重试**：重试逻辑属于控制流，不属于提示。

## 3.4 从提示工程到 Agent 的演进

```
阶段1：单次提示            —— 输入 → 输出
阶段2：提示 + 结构化输出    —— 输出可被程序消费
阶段3：提示 + 工具调用      —— 输出可触发动作
阶段4：提示 + 工具 + 循环   —— Agent
```

> 每一阶段都在前一阶段之上叠加能力，而非替换。

# 4. 最小 Agent 示例

## 4.1 伪代码

```java
public String run(String goal) {
    List<Message> history = new ArrayList<>();
    history.add(system("你可以使用工具完成任务。"));
    history.add(user(goal));

    for (int step = 0; step < MAX_STEPS; step++) {
        Response response = llm.chat(history, availableTools());
        history.add(response.message());

        if (!response.hasToolCalls()) {
            return response.text();          // 目标达成
        }
        for (ToolCall call : response.toolCalls()) {
            history.add(executeTool(call));  // 执行并回填
        }
    }
    return "超出最大步数";
}
```

## 4.2 逐行解读

### 4.2.1 消息累积

`history` 承载了"世界状态"：系统提示、用户目标、模型决策、工具结果。**Agent 的"记忆"在最简形态下就是这条消息列表**。

### 4.2.2 工具调用

`response.hasToolCalls()` 为真时，模型是在"请求行动"。执行后把结果作为新消息回填，供下一轮决策。

### 4.2.3 终止判断

- 无工具调用 → 正常结束；
- 达到 `MAX_STEPS` → 强制结束（必须有）。

## 4.3 用 Spring AI 实现

```java
ChatClient client = ChatClient.builder(model)
        .defaultSystem("你可以使用工具完成任务。")
        .defaultTools(orderTools, policyTools)
        .build();

String answer = client.prompt().user(goal).call().content();
```

框架把循环、工具调用与结果回填都封装好了，开发者只需声明工具与提示。

> 无论框架多复杂，内核都是 4.1 那段逻辑。详见 `4_agent_loop.md`。

## 4.4 完整可运行示例（Java 21）

### 4.4.1 场景与工具

以"查询订单并计算退款金额"为例，提供两个工具：

```java
public class OrderTools {

    @Tool("根据订单号查询订单详情")
    public Order getOrder(@P("订单号") String orderId) {
        return orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("订单不存在"));
    }

    @Tool("计算指定订单的退款金额（含运费规则）")
    public BigDecimal calcRefund(@P("订单号") String orderId) {
        Order order = getOrder(orderId);
        return refundPolicy.calculate(order);
    }
}
```

### 4.4.2 主循环

```java
public class MinimalAgent {

    private static final int MAX_STEPS = 8;

    private final ChatModel model;
    private final List<Object> tools;

    public MinimalAgent(ChatModel model, List<Object> tools) {
        this.model = model;
        this.tools = tools;
    }

    public String run(String goal) {
        List<Message> history = new ArrayList<>();
        history.add(SystemMessage.from("你可以使用工具完成任务。"));
        history.add(UserMessage.from(goal));

        for (int step = 0; step < MAX_STEPS; step++) {
            ChatResponse response = model.call(new Prompt(history, tools));

            AiMessage ai = response.aiMessage();
            history.add(ai);

            if (!ai.hasToolExecutionRequests()) {
                return ai.text();
            }
            for (ToolExecutionRequest req : ai.toolExecutionRequests()) {
                String result = execute(req);
                history.add(ToolExecutionResultMessage.from(req, result));
            }
        }
        return "超出最大步数，未完成目标";
    }

    private String execute(ToolExecutionRequest req) {
        return toolExecutor.execute(req, tools);
    }
}
```

### 4.4.3 运行与观察

```java
MinimalAgent agent = new MinimalAgent(model, List.of(new OrderTools()));
String answer = agent.run("查一下 A123 订单能退多少钱？");
System.out.println(answer);
```

执行时可在日志中观察到：`getOrder("A123")` → `calcRefund("A123")` → 生成自然语言答案。这正是 4.1 伪代码的真实形态。

## 4.5 常见实现错误与反例

| 错误 | 后果 | 修正 |
|------|------|------|
| 没有 `MAX_STEPS` | 死循环、成本失控 | 加最大步数 + 超时 |
| 工具结果不回填 | 模型重复调用同一工具 | 每步都追加工具结果消息 |
| 工具异常直接抛出 | 循环中断 | 捕获并作为错误结果回填 |
| 把大对象塞进上下文 | Token 爆炸 | 只回填必要字段 |
| 无系统提示约束 | 目标漂移 | 明确角色与终止条件 |

# 5. Agent 的能力边界

## 5.1 擅长

- 多步信息收集与整合（研究、调研）；
- 需要调用外部系统的任务（查询、下单、发邮件）；
- 流程性任务（按规则执行、异常处理）；
- 需要一定判断的重复工作。

## 5.2 不擅长

- 强确定性、强一致性的核心交易（用工作流更稳）；
- 需要精确数学计算（应交给工具/代码）；
- 长链条高风险操作（易累积错误）；
- 无明确成功标准的开放性任务。

## 5.3 边界判断清单

| 问题 | 若是 | 若否 |
|------|------|------|
| 步骤是否稳定可枚举？ | 工作流 | 继续问 |
| 失败代价是否很高？ | 工作流/人工 | 继续问 |
| 是否需要多步判断？ | Agent | 单次 LLM |
| 是否有明确成功标准？ | Agent | 人工介入 |

> **不要用 Agent 解决"工作流就能解决"的问题**，这是最常见的误用。

## 5.4 能力边界的量化评估

判断某任务是否适合 Agent，可用四个维度打分（1~5 分）：

| 维度 | 低分（适合工作流） | 高分（适合 Agent） |
|------|-------------------|-------------------|
| 路径不确定性 | 步骤固定 | 步骤随输入变化 |
| 失败代价 | 高（不可回滚） | 低（可重试） |
| 输入开放性 | 结构化 | 自然语言/多模态 |
| 判断复杂度 | 规则可枚举 | 需常识/推理 |

> 四项合计低 → 工作流；合计高 → Agent；居中的 → 混合模式。

## 5.5 边界之外的替代方案

当任务落在 Agent"不擅长"区间时，常用替代：

- **精确计算** → 交给代码/计算器工具，而非让模型心算；
- **强一致交易** → 工作流 + 数据库事务；
- **高风险长链条** → 拆成多个受限 Agent 节点，节点间加人工审批；
- **无成功标准的开放任务** → 先与用户澄清目标，再决定形态。

# 6. Agent 的能力来源分解

## 6.1 公式

```
任务成功率 ≈ 模型能力 × 工具质量 × 上下文质量 × 控制流设计
```

注意是**乘法**：任一环节接近 0，整体成功率都会崩塌。

## 6.2 四个乘数

| 乘数 | 说明 | 优化方向 |
|------|------|----------|
| 模型能力 | 推理与工具调用 | 选对模型（`2_LLM/Models/3_model_selection.md`） |
| 工具质量 | 描述、参数、结果 | 见 `7_ToolsAndProtocols` |
| 上下文质量 | 给对信息、去掉噪声 | 见 `8_Memory` |
| 控制流 | 终止、重试、状态 | 见 `10_state_and_control_flow.md` |

## 6.3 生产事故归因

```
工具失败        → 检查描述/参数/错误处理
答非所问        → 检查上下文与提示
死循环/超支     → 检查终止条件
参数幻觉        → 检查工具 schema 与校验
```

> 大部分失败都能在"工具、上下文、控制流"三层定位，而非换模型。

## 6.4 一个归因实例

现象：Agent 频繁调用 `searchOrder` 却拿不到结果。

```
排查顺序：
1. 工具层  → 参数是否合法？描述是否让模型误解？
2. 上下文  → 订单号是否在历史中被截断？
3. 控制流  → 失败后是否无限重试同一调用？
4. 模型层  → 以上都正常，才考虑换模型
```

> 结论：多数问题在前三层即可定位，换模型往往是最后手段。

# 7. 典型 Agent 形态

## 7.1 五种形态

| 形态 | 说明 | 例子 |
|------|------|------|
| 工具型 Agent | 主要靠工具完成动作 | 订票、下单 |
| 检索型 Agent | 主要靠检索获取知识 | 研究助手、知识问答 |
| 编码型 Agent | 读写代码、执行测试 | 编程助手 |
| 流程型 Agent | 按规则推进多步流程 | 审批、工单 |
| 协作型 Agent | 多 Agent 分工 | 见 `10_MultiAgent` |

## 7.2 形态与架构选型

- 工具型/流程型：偏工作流骨架 + Agent 节点；
- 检索型：RAG + 轻量循环；
- 编码型：强工具（代码执行/文件） + 长循环 + 沙箱；
- 协作型：Supervisor / Swarm 等编排（见 `10_MultiAgent`）。

## 7.3 形态选择的决策流程

```
任务主要靠什么？
  ├── 外部动作 → 工具型
  ├── 知识检索 → 检索型
  ├── 代码读写 → 编码型
  ├── 规则流程 → 流程型
  └── 多角色分工 → 协作型
```

# 8. 小结

- Agent = LLM + 工具 + 记忆 + 规划，能自主完成多步任务。
- 与聊天机器人的本质区别是"会为目标采取行动"。
- 最小 Agent 内核就是"消息累积 + 工具调用 + 循环 + 终止条件"。
- 成功率由模型、工具、上下文、控制流共同决定（乘法关系）。
- 能用工作流解决的，不要用 Agent。
