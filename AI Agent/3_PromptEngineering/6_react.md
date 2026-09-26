# 1. 什么是 ReAct

## 1.1 定义与直觉

### 1.1.1 一句话定义

**ReAct = Reasoning（推理）+ Acting（行动）**，让模型交替进行"思考"与"行动"。

结论先行：ReAct 是让语言模型具备"边想边做"能力的提示范式，也是所有 Agent 框架的基础。

### 1.1.2 Reasoning 与 Acting 的分工

| 能力 | 作用 | 产出 |
|------|------|------|
| Reasoning（推理） | 决定下一步做什么 | Thought |
| Acting（行动） | 调用工具获取信息 | Action + Observation |

> 推理指导行动，行动结果反馈推理，形成闭环。

## 1.2 一个完整轨迹

### 1.2.1 轨迹示例

```
Thought:  我需要先查订单状态
Action:   getOrder("A123")
Observation: 已发货，运单 SF123
Thought:  需要查物流
Action:   queryLogistics("SF123")
Observation: 运输中，明天送达
Thought:  信息足够
Answer:   您的订单已发货，预计明天送达
```

### 1.2.2 轨迹解读

- `Thought` 是模型内部规划，不产生副作用。
- `Action` 是对外部的调用，需解析并执行。
- `Observation` 是真实环境返回，不能被模型编造。

## 1.3 为什么需要 ReAct

### 1.3.1 纯推理的局限

```
纯推理（CoT）：只能基于已有信息推理，无法获取新信息
→ 遇到"实时数据"（库存、天气、订单）就无能为力
```

### 1.3.2 纯行动的局限

```
纯行动：盲目调用工具，缺乏规划
→ 不知道何时调用、调用哪个、调用几次
```

结论：ReAct 将两者结合——推理指导行动，行动结果反馈推理。

| 范式 | 能获取新信息 | 有规划 | 结果 |
|------|:------------:|:------:|------|
| 纯推理 | 否 | 是 | 遇到实时数据失效 |
| 纯行动 | 是 | 否 | 盲目调用、浪费 |
| ReAct | 是 | 是 | 闭环、可控 |

# 2. 提示格式

## 2.1 文本 ReAct 格式

### 2.1.1 格式定义

```
每次输出以下格式之一：

1. 需要调用工具时：
   Thought: 你的思考
   Action: 工具名
   Action Input: 工具参数（JSON）

2. 可以回答时：
   Thought: 信息已足够
   Answer: 最终答案
```

### 2.1.2 约束规则

| 规则 | 目的 |
|------|------|
| 每次只输出一个 Action | 便于逐步执行 |
| 不要编造 Observation | 防止幻觉 |
| 最多执行 8 步 | 防止死循环 |

## 2.2 Java 实现（系统提示）

```java
@SystemMessage("""
        你可以使用工具来完成任务。

        每次输出以下格式之一：

        1. 需要调用工具时：
        Thought: 你的思考
        Action: 工具名
        Action Input: 工具参数（JSON）

        2. 可以回答时：
        Thought: 信息已足够
        Answer: 最终答案

        规则：
        - 每次只输出一个 Action
        - 不要编造 Observation
        - 最多执行 8 步
        """)
```

可用工具清单应显式注入提示，防止模型幻觉工具：

```java
@SystemMessage("""
        可用工具：
        - getOrder(orderId): 查询订单状态
        - queryLogistics(trackingNo): 查询物流

        只能使用以上工具。
        """)
```

## 2.3 格式边界

| 边界 | 问题 | 对策 |
|------|------|------|
| 参数含特殊字符 | JSON 解析失败 | 强制 JSON 转义 |
| 多 Action | 执行顺序混乱 | 强调"一次一个" |
| 混入解释文本 | 解析歧义 | 规定严格前缀 |

生产建议：文本 ReAct 主要用于理解原理；生产优先使用原生函数调用。

# 3. 循环实现

## 3.1 循环骨架

### 3.1.1 五个阶段

```
① 生成 Thought + Action
② 检查是否给出最终答案
③ 解析 Action
④ 执行工具
⑤ 追加 Observation 到 scratchpad
```

### 3.1.2 Java 实现

```java
public String reactLoop(String question, List<Tool> tools, int maxSteps) {
    StringBuilder scratchpad = new StringBuilder();

    for (int step = 0; step < maxSteps; step++) {
        // ① 生成 Thought + Action
        String output = model.chat(buildPrompt(question, scratchpad.toString()));

        // ② 检查是否给出最终答案
        if (output.contains("Answer:")) {
            return extractAnswer(output);
        }

        // ③ 解析 Action
        String toolName = extractTool(output);
        String args = extractArgs(output);

        // ④ 执行工具
        String observation = executeTool(toolName, args);

        // ⑤ 追加到 scratchpad
        scratchpad.append(output).append("\n");
        scratchpad.append("Observation: ").append(observation).append("\n");
    }
    return "达到最大步数";
}
```

辅助方法示意：

```java
private String buildPrompt(String question, String scratchpad) {
    return """
            问题：%s

            已完成步骤：
            %s

            请输出下一步 Thought 与 Action，或直接给出 Answer。
            """.formatted(question, scratchpad);
}

private String executeTool(String toolName, String args) {
    return toolRegistry.get(toolName).invoke(args);
}
```

## 3.2 解析 Action

### 3.2.1 文本解析

```java
private String extractTool(String output) {
    // 匹配 "Action: xxx"
    return output.lines()
            .filter(l -> l.startsWith("Action:"))
            .map(l -> l.substring("Action:".length()).trim())
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("缺少 Action"));
}
```

### 3.2.2 容错

| 失败 | 处理 |
|------|------|
| 缺少 Action | 提示模型重试 |
| 参数非 JSON | 尝试修复或报错 |
| 未知工具 | 返回错误 Observation |

## 3.3 终止条件

| 条件 | 说明 |
|------|------|
| 输出 Answer | 正常结束 |
| 达到最大步数 | 强制结束 |
| 工具连续报错 | 提前中止 |
| 重复调用 | 去重后结束 |

> 终止条件是 ReAct 稳定性的关键，必须显式设计。

# 4. 原生函数调用版 ReAct

## 4.1 原理

现代框架用**原生函数调用（Native Function Calling）** 替代文本解析，更可靠：模型直接返回结构化的工具调用请求。

```
文本 ReAct：模型输出文本 → 应用解析文本
函数调用：  模型输出结构化 tool_calls → 应用直接执行
```

## 4.2 Java 实现（Spring AI）

### 4.2.1 完整循环

```java
public String reactWithTools(String question, List<ToolCallback> tools) {
    List<Message> messages = new ArrayList<>();
    messages.add(SystemMessage.from("你可以使用工具完成任务。"));
    messages.add(UserMessage.from(question));

    for (int step = 0; step < MAX_STEPS; step++) {
        ChatResponse response = chatClient.prompt()
                .messages(messages)
                .toolCallbacks(tools)
                .call()
                .chatResponse();

        AiMessage ai = response.getResult().getOutput();
        messages.add(ai);

        if (!ai.hasToolCalls()) {
            return ai.getText();                  // 最终答案
        }
        for (ToolCall call : ai.getToolCalls()) {
            String result = executeTool(call);
            messages.add(new ToolResponseMessage(List.of(
                    new ToolResponse(call.id(), call.name(), result))));
        }
    }
    return "达到最大步数";
}
```

### 4.2.2 工具定义

```java
public class OrderTools {

    @Tool(description = "根据订单号查询订单状态")
    public String getOrder(String orderId) {
        return orderService.query(orderId);
    }
}
```

```java
public class LogisticsTools {

    @Tool(description = "根据运单号查询物流轨迹")
    public String queryLogistics(String trackingNo) {
        return logisticsService.track(trackingNo);
    }
}
```

> 工具描述（description）会作为提示的一部分，必须清晰、无歧义。

## 4.3 两种方式对比

| 方式 | 优点 | 缺点 |
|------|------|------|
| 文本解析 | 通用、可解释 | 解析易错 |
| 原生函数调用 | 可靠、结构化 | 依赖模型支持 |

> 生产优先用**原生函数调用**，文本 ReAct 用于理解原理。

# 5. ReAct 的关键要素

## 5.1 五要素

| 要素 | 说明 |
|------|------|
| Thought | 显式推理，指导行动 |
| Action | 选择工具与参数 |
| Observation | 工具返回结果 |
| 循环 | 直到信息足够 |
| 终止 | 给出 Answer 或达到上限 |

## 5.2 要素间关系

```
Thought ──决定──> Action ──执行──> Observation
   ^                                    │
   └──────────反馈────────────<────────┘
```

## 5.3 缺失要素的后果

| 缺失 | 后果 |
|------|------|
| Thought | 行动缺乏规划 |
| Action | 无法获取新信息 |
| Observation | 无法闭环，易幻觉 |
| 循环 | 只能单步 |
| 终止 | 可能无限执行 |

# 6. ReAct 的失败模式

## 6.1 常见失败

| 失败 | 表现 | 对策 |
|------|------|------|
| 重复调用 | 反复调同一工具 | 签名去重 |
| 幻觉工具 | 调用不存在的工具 | 工具白名单 |
| 不调用工具 | 直接编答案 | 提示强调先查证 |
| 死循环 | 无法收敛 | 最大步数 |
| 忽略结果 | 不基于 Observation | 提示强调 |

## 6.2 根因分析

### 6.2.1 重复调用

模型未意识到"已获得该信息"，或对结果不满意而重试。

### 6.2.2 幻觉工具

提示中未明确列出可用工具，模型按常识"编造"工具名。

## 6.3 防护措施

```
签名去重：以 (工具名 + 参数) 为键，命中则跳过并提示"已获取"
工具白名单：执行前校验工具名是否在注册表内
步数上限：硬性截断
结果裁剪：Observation 过长时截断，避免污染上下文
```

```java
private final Set<String> seen = new HashSet<>();

private boolean duplicate(String toolName, String args) {
    return !seen.add(toolName + "|" + args);   // 返回 false 表示已见过
}
```

# 7. 与其他范式对比

## 7.1 范式总览

| 范式 | 说明 |
|------|------|
| CoT | 只推理，不行动 |
| ReAct | 推理 + 行动 |
| Reflexion | ReAct + 反思 |
| Plan-and-Execute | 先规划全部，再执行 |

## 7.2 对比表

| 范式 | 是否行动 | 是否反思 | 是否全局规划 |
|------|:--------:|:--------:|:------------:|
| CoT | 否 | 否 | 否 |
| ReAct | 是 | 否 | 否（边走边定） |
| Reflexion | 是 | 是 | 否 |
| Plan-and-Execute | 是 | 否 | 是 |

## 7.3 选型建议

结论先行：ReAct 是最基础的"能行动"范式，其他范式多在其上增强。

```
需要行动？ 否 → CoT
         是 → 需要从失败中学习？ 是 → Reflexion
                              否 → 任务步骤固定？ 是 → Plan-and-Execute
                                                 否 → ReAct
```

| 场景 | 推荐范式 |
|------|----------|
| 单轮问答 | 无 |
| 需要外部信息 | ReAct |
| 需要从错误中学习 | Reflexion |
| 步骤多且可预知 | Plan-and-Execute |

# 8. 在框架中

## 8.1 Spring AI

```java
// Spring AI
String answer = chatClient.prompt()
        .user(question)
        .tools(new OrderTools(), new LogisticsTools())
        .call()
        .content();
// 框架自动完成 ReAct 循环
```

```java
@Configuration
public class ChatConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("你可以使用工具完成任务，最多 8 步。")
                .defaultOptions(ChatOptions.builder()
                        .temperature(0.2)
                        .build())
                .build();
    }
}
```

## 8.2 LangChain4j

```java
// LangChain4j
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new OrderTools(), new LogisticsTools())
        .build();
```

## 8.3 框架边界

| 边界 | 说明 |
|------|------|
| 循环次数 | 框架默认上限可能不够，需显式配置 |
| 工具异常 | 异常如何反馈给模型需确认 |
| 上下文膨胀 | 长轨迹需裁剪 |
| 可观测性 | 需自行接入 Trace |

# 9. 生产注意事项

## 9.1 安全

### 9.1.1 工具鉴权

| 风险 | 建议 |
|------|------|
| 工具无鉴权 | 工具内校验 |
| 越权调用 | 按用户身份限制工具 |
| 副作用工具 | 写操作需二次确认 |

### 9.1.2 提示注入

```
外部数据（如订单备注）可能包含恶意指令，
需将其视为"数据"而非"指令"，并做转义与隔离。
```

## 9.2 可观测性

### 9.2.1 Trace

| 风险 | 建议 |
|------|------|
| 无 Trace | 记录每步 |
| 结果过大 | 精简 |

```java
// 每步记录：step、thought、action、observation、耗时
log.info("step={} action={} args={} costMs={}",
        step, toolName, args, costMs);
```

### 9.2.2 指标

| 指标 | 含义 |
|------|------|
| 步数分布 | 是否接近上限 |
| 工具调用成功率 | 稳定性 |
| 重复调用率 | 提示质量 |
| 平均延迟 | 体验 |

## 9.3 稳定性

| 风险 | 建议 |
|------|------|
| 无步数上限 | 必设 |
| 重复调用 | 去重 |
| 工具过多 | 按场景裁剪 |

# 10. 小结

- ReAct = 推理 + 行动，是 Agent 的基础范式。
- 格式：Thought → Action → Observation 循环。
- 生产用原生函数调用替代文本解析。
- 关键要素：推理、行动、观察、终止。
- 必须设步数上限并防止重复调用。
- 工具需鉴权，轨迹需 Trace，外部数据需防注入。
