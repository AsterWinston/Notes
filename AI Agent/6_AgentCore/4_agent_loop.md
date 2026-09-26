# 1. 核心循环

Agent 的心脏是一个"思考—行动—观察"的闭环：模型不再一次性给出答案，而是**反复决策**，每一步都根据上一步的结果决定下一步，直到达成目标或触发终止条件。

与普通 LLM 调用相比，Agent 循环有三个本质差异：

- **多步**：一次任务可能包含若干次模型调用与工具调用；
- **有状态**：历史消息、工具结果在循环中不断累积并回灌给模型；
- **可终止**：必须有明确的停止条件，否则会无限循环、烧钱甚至失控。

本节先建立整体直觉，再逐层拆解：四个阶段 → 代码实现 → 终止条件 → 常见问题 → 效率优化 → 可观测。

## 1.1 循环的本质

一句话：**把"一次生成"变成"带反馈的迭代"**。

```
观察当前状态 → 模型决策下一步 → 执行动作 → 观察结果 → 再决策 …
                        ↑                              │
                        └──────────── 反馈 ────────────┘
```

普通调用是"输入 → 输出"的单次映射；循环则是把上一次的输出（尤其是工具结果）作为下一次的输入，形成闭环。

## 1.2 与"工作流"的区别

| 维度 | 工作流（Workflow） | Agent 循环 |
|------|--------------------|------------|
| 路径 | 预先固定 | 运行时动态决定 |
| 步数 | 固定 | 不定（受上限约束） |
| 决策者 | 代码 | 模型 |
| 可预测性 | 高 | 低 |
| 调试难度 | 低 | 高 |
| 适用场景 | 稳定、确定的流程 | 开放、需探索的任务 |

> **能用工作流解决的，就不要用 Agent**。循环的自由度带来灵活性，也带来不可预测性与成本。

## 1.3 一个最小直觉示例

以"查询订单并判断是否可退款"为例，循环会这样展开：

```
Step 1  模型：需要订单信息 → 调用 getOrder("A123")
Step 2  工具：返回 {status: "已发货", amount: 199}
Step 3  模型：需要退款政策 → 调用 getPolicy("已发货")
Step 4  工具：返回 "已发货订单 7 天内可退"
Step 5  模型：信息足够 → 直接输出结论（无工具调用，循环结束）
```

可以看到：模型并非一次答完，而是"走一步、看一步"。

# 2. 四个阶段

循环的骨架可拆为四个阶段，业界常称为 **Perceive–Plan–Act–Observe**。

```
        ┌─────────────────────────────────────┐
        │                                     │
        ▼                                     │
   ┌─────────┐    ┌─────────┐    ┌─────────┐  │
   │  感知    │───▶│  规划    │───▶│  行动    │──┤
   │ Perceive│    │  Plan   │    │  Act    │  │
   └─────────┘    └─────────┘    └────┬────┘  │
        ▲                             │       │
        │        ┌─────────┐          │       │
        └────────│  观察    │◀─────────┘       │
                 │ Observe │                  │
                 └────┬────┘                  │
                      │                       │
                      ▼                       │
              达成目标？─否─────────────────────┘
                      │
                     是
                      ▼
                   输出结果
```

| 阶段 | 内容 | 实现方式 |
|------|------|----------|
| 感知 Perceive | 接收目标与当前状态 | 组装消息（系统提示 + 历史 + 工具结果） |
| 规划 Plan | 决定下一步做什么 | LLM 推理，产出工具调用或最终答案 |
| 行动 Act | 调用工具 / 生成答案 | 工具执行 / 直接输出 |
| 观察 Observe | 获取行动结果 | 工具返回值回填为消息 |

## 2.1 感知 Perceive

感知阶段的职责是**把"世界状态"翻译成模型能读懂的消息序列**。它要回答三个问题：

1. 目标是什么？（用户原始诉求）
2. 已经做了什么？（历史决策与工具结果）
3. 现在处于什么状态？（可用工具、剩余预算、是否接近上限）

### 2.1.1 消息的组成

一条典型的感知消息列表包含：

```
[System]  角色、规则、可用工具说明、终止约束
[User]    原始目标
[Assistant] 第 1 步的工具调用请求
[Tool]    第 1 步的工具结果
[Assistant] 第 2 步的工具调用请求
[Tool]    第 2 步的工具结果
...
```

### 2.1.2 感知阶段的常见错误

- 把工具结果**原样全量**塞回消息，导致 Token 爆炸；
- 忘记把**目标**放进消息，多步后模型跑偏；
- 工具结果里的**指令性文本**未做隔离，引入提示注入风险。

## 2.2 规划 Plan

规划阶段是**唯一由模型主导**的环节：模型读取当前消息，决定"下一步是调用工具还是直接回答"。

输出通常有两类：

- **工具调用请求**（Tool Call）：`{name, arguments}`；
- **最终答案**：自然语言文本（或结构化对象）。

### 2.2.1 规划质量的三个影响因素

| 因素 | 说明 |
|------|------|
| 提示质量 | 是否清晰说明目标、工具用法、终止条件 |
| 工具设计 | 工具描述是否准确、参数是否明确 |
| 上下文 | 历史是否精简且关键信息完整 |

> 规划是"提示工程 + 工具设计"的综合体现，详见 `3_PromptEngineering/6_react.md` 与 `7_ToolsAndProtocols/1_tool_design.md`。

## 2.3 行动 Act

行动阶段把模型的决策落地为**真实的副作用**：查询数据库、调用外部 API、执行代码等。

关键原则：

- **可重试**：读操作可安全重试，写操作需幂等；
- **可超时**：任何工具调用都要有超时上限；
- **可观测**：记录入参、出参、耗时、状态码。

### 2.3.1 并行行动

当多个工具之间没有依赖时，应并行执行以降低总延迟：

```java
List<CompletableFuture<String>> futures = requests.stream()
        .map(req -> CompletableFuture.supplyAsync(() -> execute(req), executor))
        .toList();
```

## 2.4 观察 Observe

观察阶段把工具结果**标准化后回填**到消息中。它决定了下一轮感知看到的"世界"。

好的观察应当：

- **精简**：只回填必要字段（如从 50 个字段里取 3 个）；
- **结构化**：JSON/表格优先，便于模型解析；
- **带错误语义**：失败要明确区分"空结果"与"调用失败"。

```java
private String observe(ToolResult raw) {
    if (raw.failed()) {
        return "{\"error\":\"" + raw.code() + "\",\"message\":\"" + raw.message() + "\"}";
    }
    return objectMapper.writeValueAsString(raw.pick("status", "amount", "deadline"));
}
```

## 2.5 四阶段与状态流转

四个阶段并非严格串行：观察结果会回到感知，形成闭环。整个循环可以看作一个**状态机**：

```
        ┌────────────────────────────────────┐
        │            AgentState              │
        │  goal / history / scratchpad /     │
        │  step / tokens / done              │
        └────────────────────────────────────┘
                 ▲                 │
      写回结果    │                 │ 读取决策
                 │                 ▼
        ┌────────────────┐   ┌────────────────┐
        │   Observe      │   │    Plan        │
        └────────────────┘   └────────────────┘
```

# 3. 循环的代码实现

理解四阶段后，我们把它落到代码。先给最小实现，再给基于 Spring AI 的实现。

## 3.1 最小实现

```java
public String run(String goal, List<Tool> tools, int maxSteps) {
    List<ChatMessage> messages = new ArrayList<>();
    messages.add(SystemMessage.from(SYSTEM_PROMPT));
    messages.add(UserMessage.from(goal));

    for (int step = 0; step < maxSteps; step++) {
        // 1. 规划（LLM 决策）
        ChatResponse response = model.chat(ChatRequest.builder()
                .messages(messages)
                .toolSpecifications(specs(tools))
                .build());

        AiMessage ai = response.aiMessage();
        messages.add(ai);

        // 2. 终止判断：无工具调用 = 给出最终答案
        if (!ai.hasToolExecutionRequests()) {
            return ai.text();
        }

        // 3. 行动 + 观察
        for (ToolExecutionRequest req : ai.toolExecutionRequests()) {
            String result = execute(req);                    // 行动
            messages.add(ToolExecutionResultMessage.from(req, result));  // 观察
        }
    }
    return "达到最大步数，未能完成";
}
```

这段代码包含了循环的全部要素：**消息组装、模型决策、终止判断、工具执行、结果回填**。

## 3.2 基于 Spring AI 的实现

Spring AI 提供了更高层的封装，工具通过 `@Tool` 声明，由框架负责调用与回填：

```java
@Service
public class OrderAgent {

    private final ChatClient chatClient;

    public OrderAgent(ChatClient.Builder builder, OrderTools tools) {
        this.chatClient = builder
                .defaultSystem("""
                        你是订单助手。规则：
                        1. 需要订单信息时调用 getOrder；
                        2. 需要政策时调用 getPolicy；
                        3. 信息足够后直接回答，不要重复调用；
                        4. 最多 6 步，超限请给出当前结论。
                        """)
                .defaultTools(tools)
                .build();
    }

    public String ask(String question) {
        return chatClient.prompt()
                .user(question)
                .call()
                .content();
    }
}
```

> Spring AI 内部已实现"规划—行动—观察"的循环，开发者只需声明工具与提示；但**终止条件仍需自行兜底**（见第 4 节）。

## 3.3 关键设计点

### 3.3.1 消息如何组织

- 系统提示放**稳定规则**，用户消息放**目标**，工具结果放**Tool 角色**；
- 用户输入永远放 User 角色，防止提示注入（详见 `3_PromptEngineering/2_prompt_structure.md`）。

### 3.3.2 工具规范如何传入

- 工具描述要写清"何时用、参数含义、返回格式"；
- 工具过多时按意图动态挂载，避免干扰规划。

### 3.3.3 如何判断终止

- **正常终止**：模型不再请求工具；
- **强制终止**：步数 / Token / 时间 / 安全上限触发。

# 4. 终止条件（关键）

Agent 必须有明确的终止条件，否则会无限循环烧钱。终止分为**正常终止**与**强制终止**两类。

## 4.1 正常终止

模型给出最终答案（本轮无任何工具调用请求）即为正常终止。

## 4.2 强制终止

强制终止是"安全网"，通常需要多重叠加：

| 终止条件 | 说明 |
|----------|------|
| 达到最大步数 | 硬性上限，防止死循环 |
| 达到 Token/成本上限 | 成本保护 |
| 达到时间上限 | 超时保护 |
| 命中禁止行为 | 安全拦截 |
| 需要人工介入 | 高风险动作挂起 |

## 4.3 LoopGuard 实现

```java
public class LoopGuard {
    private final int maxSteps;
    private final long maxTokens;
    private final Duration maxDuration;
    private final Instant start = Instant.now();

    public boolean shouldStop(int steps, long tokens) {
        return steps >= maxSteps
                || tokens >= maxTokens
                || Duration.between(start, Instant.now()).compareTo(maxDuration) > 0;
    }
}
```

### 4.3.1 步数上限怎么定

- 简单问答：3~5 步；
- 多工具任务：8~15 步；
- 长流程：按任务预算设定，并配合成本上限。

### 4.3.2 Token/成本上限

按"单次会话预算"设定，例如单会话 ≤ 50k Token；接近阈值时**降级为直接回答**而非硬失败。

### 4.3.3 时间上限

面向交互式场景（如客服），端到端延迟通常要求 < 10s，需同时设置单工具超时与整体超时。

### 4.3.4 安全拦截

命中禁止行为（如尝试调用高危写操作、访问越权数据）时**立即终止**并转人工。

## 4.4 终止设计的常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 只设步数上限 | 单步极贵时仍会超支 | 步数 + 成本双上限 |
| 无时间上限 | 工具挂起导致请求堆积 | 单工具超时 + 整体超时 |
| 超限直接抛异常 | 用户体验差 | 降级返回当前结论 |
| 无安全拦截 | 越权/高危操作被执行 | 终止 + 转人工 |

# 5. 循环的常见问题

## 5.1 重复调用同一工具

### 5.1.1 现象与根因

```
步1: getOrder("A123") → "订单不存在"
步2: getOrder("A123") → "订单不存在"   ← 重复
步3: getOrder("A123") → "订单不存在"
```

根因：模型未意识到"同样的调用不会得到新结果"，倾向于反复尝试同一动作。

### 5.1.2 检测：调用签名去重

把"工具名 + 参数"作为签名，记录已执行集合：

```java
private final Set<String> calledSignatures = new HashSet<>();

private String execute(ToolExecutionRequest req) {
    String signature = req.name() + ":" + req.arguments();
    if (!calledSignatures.add(signature)) {
        return "该调用已执行过，请勿重复调用，尝试其他方法或直接回答。";
    }
    return doExecute(req);
}
```

### 5.1.3 修复：提示约束 + 结果缓存 + 强制终止

1. **提示约束**：系统提示声明"不要重复调用同一工具相同参数"；
2. **结果缓存**：命中签名直接返回缓存结果并附提示；
3. **强制终止**：同一签名重复 ≥ N 次即终止。

### 5.1.4 完整实现

```java
public class DedupingToolExecutor {
    private final Map<String, String> cache = new HashMap<>();
    private final Map<String, Integer> hits = new HashMap<>();
    private static final int MAX_HITS = 3;

    public String execute(ToolExecutionRequest req) {
        String sig = req.name() + ":" + req.arguments();
        if (cache.containsKey(sig)) {
            int n = hits.merge(sig, 1, Integer::sum);
            if (n >= MAX_HITS) {
                throw new LoopAbortException("检测到重复调用：" + sig);
            }
            return "[缓存] " + cache.get(sig)
                    + "（该结果已返回过，请换一种方式或直接作答）";
        }
        String result = doExecute(req);
        cache.put(sig, result);
        return result;
    }
}
```

### 5.1.5 边界与反例

- 若工具本身**有副作用**（如下单），缓存会导致漏执行——写操作不应缓存；
- 若参数含**时间戳/随机数**，签名永远不同，去重失效，需归一化参数。

## 5.2 目标漂移

### 5.2.1 现象与根因

多步之后模型忘记原始目标，转而去解决"顺手发现的问题"。根因是长上下文中早期目标被稀释。

### 5.2.2 对策：目标重申 + 结构化状态

```java
messages.add(SystemMessage.from("原始目标：" + goal + "。始终围绕该目标行动。"));
```

更进一步，把目标与进度放入结构化状态，每轮显式回灌：

```java
record AgentState(String goal, List<String> done, String next) {}
```

## 5.3 上下文爆炸

### 5.3.1 现象与根因

消息越积越多，Token 超限、成本上升、模型注意力被稀释。根因是历史线性增长且工具结果冗长。

### 5.3.2 对策：窗口 + 压缩 + 摘要

- **滑动窗口**：只保留最近 N 轮；
- **结果精简**：只回填必要字段；
- **摘要压缩**：超阈值后把早期历史压缩为摘要。

> 详见 `8_Memory/8_memory_compression.md`。

## 5.4 工具死循环

两个工具互相触发（A 的结果让模型调 B，B 的结果又让模型调 A），且始终不收敛。

对策：**调用图去环**——记录"工具 A→B→A"的循环模式并强制终止。

## 5.5 幻觉工具调用

模型调用了不存在的工具或参数不合规范。

对策：

- 工具规范校验，非法调用直接返回结构化错误而非抛异常；
- 在系统提示中给出工具清单与示例；
- 对未知工具返回"工具不存在，可用工具有…"。

# 6. 循环效率优化

| 优化 | 说明 |
|------|------|
| 并行工具调用 | 无依赖的工具并发执行 |
| 工具结果精简 | 只回填必要字段，避免 Token 浪费 |
| 缓存 | 相同调用直接复用结果 |
| 早停 | 已能回答时不再调用工具 |
| 减少轮数 | 优化提示，让模型一次规划多步 |

## 6.1 并行工具调用

```java
List<CompletableFuture<String>> futures = requests.stream()
        .map(req -> CompletableFuture.supplyAsync(() -> execute(req), executor))
        .toList();
```

## 6.2 结果精简

从工具返回中只取下游需要的字段，可将 Token 降低 50% 以上。

## 6.3 缓存

- 读操作按签名缓存；
- 语义缓存：相似问题复用答案（详见 `4_LLMEngineering/7_caching.md`）。

## 6.4 早停与合并规划

在提示中要求"若已能回答则直接回答"，并鼓励"一次规划多步"，减少往返。

## 6.5 优化效果（经验值）

| 优化 | 典型收益 |
|------|----------|
| 并行调用 | 延迟 −30%~60% |
| 结果精简 | Token −40%~60% |
| 缓存命中 | 成本 −20%~50% |
| 早停 | 步数 −1~3 |

> 具体收益取决于任务形态，务必用自有数据测量。

# 7. 循环的可观测

没有每轮记录的 Trace，Agent 几乎无法调试。

## 7.1 每轮应记录什么

```
Trace: 用户目标
├── Step 1: LLM 决策 → 调用 getOrder("A123")
│    └── Tool: getOrder, 耗时 120ms, 结果 200
├── Step 2: LLM 决策 → 调用 getLogistics("SF123")
│    └── Tool: getLogistics, 耗时 340ms, 结果 200
└── Step 3: LLM 决策 → 输出最终答案
```

## 7.2 Trace 结构

每步一个 Span，包含：决策类型、工具名、入参摘要、出参摘要、耗时、Token、状态。

## 7.3 关键指标

| 指标 | 含义 |
|------|------|
| 平均步数 | 反映任务复杂度与提示效率 |
| 工具失败率 | 反映工具稳定性 |
| 重复调用率 | 反映规划质量 |
| 单任务 Token/成本 | 成本控制基线 |

> 详见 `13_Observability/3_agent_trace_and_spans.md`。

# 8. 生产注意事项

| 风险 | 建议 |
|------|------|
| 无最大步数 | 必设硬上限 |
| 重复调用 | 签名去重 + 提示约束 |
| 上下文爆炸 | 记忆窗口 + 压缩 |
| 无 Trace | 每轮记录决策与工具调用 |
| 成本失控 | Token 预算 + 告警 |
| 工具慢 | 并行 + 超时 |
| 无安全拦截 | 高危动作终止 + 转人工 |

# 9. 小结

- Agent 循环 = 感知 → 规划 → 行动 → 观察，本质是"带反馈的迭代"。
- **终止条件是循环的生命线**：正常结束 + 多重硬性上限（步数/成本/时间/安全）。
- 重复调用、目标漂移、上下文爆炸是三大常见问题，各有成熟对策。
- 并行、缓存、精简结果、早停是主要优化手段。
- 每轮都要可观测，否则无法调试。
