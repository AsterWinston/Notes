# 1. 完整流程

## 1.1 核心结论

工具调用本质是一个**"决策 - 执行 - 回填 - 再决策"**的循环，直到模型不再请求工具、给出最终答案为止。

## 1.2 七步流程拆解

### 1.2.1 请求与决策

```
① 请求：消息 + 工具 Schema 发给模型
        │
        ▼
② 决策：模型返回 tool_calls（工具名 + 参数）
        │
        ▼
③ 解析：框架解析 tool_calls
```

### 1.2.2 执行与回填

```
④ 执行：调用真实 Java 方法
        │
        ▼
⑤ 回填：结果作为 ToolMessage 追加到消息
```

### 1.2.3 再决策与循环

```
⑥ 再决策：模型基于结果决定下一步
        │
        ▼
⑦ 循环：直到模型给出最终答案
```

## 1.3 流程的边界

- 每一轮都要把历史消息完整带回模型，否则上下文丢失。
- 循环必须有上限，防止模型陷入"调用 - 失败 - 再调用"的死循环。
- 每步都要可观测：记录工具名、参数、耗时、结果大小。

# 2. 消息序列

## 2.1 结论

结论先行：工具调用的正确性依赖**完整的消息序列**。每一步的 Assistant 与 Tool 消息都必须保留，模型才能理解上下文。

## 2.2 完整示例

```
[System]    你是助手，可以使用工具
[User]      查一下订单 A123 的物流
[Assistant] tool_calls: [{name: "getOrder", args: {"orderId":"A123"}}]
[Tool]      {"orderId":"A123","status":"SHIPPED","waybill":"SF123"}
[Assistant] tool_calls: [{name: "queryLogistics", args: {"waybill":"SF123"}}]
[Tool]      {"waybill":"SF123","status":"运输中","updatedAt":"..."}
[Assistant] 您的订单已发货，运单 SF123，当前运输中...
```

> 每一步的 Assistant 与 Tool 消息都必须保留，模型才能理解上下文。

## 2.3 消息角色表

### 2.3.1 System、User 与 Assistant

| 角色 | 作用 |
|------|------|
| System | 设定身份与可用工具说明 |
| User | 用户输入 |
| Assistant | 模型输出，可能含 tool_calls |

### 2.3.2 Tool 消息

| 字段 | 作用 |
|------|------|
| tool_call_id | 关联是哪次调用的结果 |
| name | 工具名（部分协议需要） |
| content | 结构化结果（通常为 JSON 字符串） |

## 2.4 消息序列的坑

- 丢弃中间的 Assistant/Tool 消息会导致模型"失忆"，重复调用同一工具。
- Tool 消息必须紧跟在触发它的 Assistant 消息之后，顺序不能乱。
- 并行调用时，多个 Tool 消息要一次性补齐，不能只补一个。

## 2.5 消息序列的三条不变式

结论先行：只要满足以下三条，多数"模型失忆"问题都能避免。

| 不变式 | 说明 | 违反后果 |
|--------|------|----------|
| 完整 | 所有历史消息都带回 | 重复调用 |
| 有序 | Tool 紧跟其 Assistant | 关联失败 |
| 配对 | 每个 tool_call 都有结果 | 部分模型报错 |

```java
// 维护消息序列的推荐做法：只追加、不重建
messages.add(aiMessage);
for (ToolExecutionRequest req : aiMessage.toolExecutionRequests()) {
    messages.add(ToolExecutionResultMessage.from(req, executeTool(req)));
}
// 下一轮直接把整个 messages 再发给模型
```

# 3. 底层实现（Spring AI）

## 3.1 结论

结论先行：手动实现循环可以看清机制；生产环境通常使用框架内置的 `ToolCallingManager` 或 ChatClient 的自动工具执行。

## 3.2 完整示例

```java
public String chatWithTools(String question) {
    List<Message> messages = new ArrayList<>();
    messages.add(SystemMessage.from("你是助手，可以使用工具。"));
    messages.add(UserMessage.from(question));

    for (int step = 0; step < MAX_STEPS; step++) {
        ChatResponse response = chatModel.call(ChatRequest.builder()
                .messages(messages)
                .toolCallbacks(toolCallbacks)
                .build());

        AiMessage ai = response.getResult().getOutput();
        messages.add(ai);

        // 无工具调用 → 结束
        if (!ai.hasToolCalls()) {
            return ai.getText();
        }

        // 执行工具
        for (ToolCall call : ai.getToolCalls()) {
            String result = executeTool(call);
            messages.add(new ToolResponseMessage(
                    List.of(new ToolResponse(call.id(), call.name(), result))));
        }
    }
    return "达到最大步数";
}
```

## 3.3 边界与坑

- `MAX_STEPS` 是必须的护栏，建议 5~10。
- 若框架已内置工具执行，手动循环会与之冲突，二选一。
- 不同版本的 `ToolResponseMessage` 构造方式可能不同，以实际 API 为准。

# 4. 底层实现（LangChain4j）

## 4.1 结论

结论先行：LangChain4j 用 `ToolSpecifications` 描述工具、用 `ToolExecutionResultMessage` 回填结果，机制与 Spring AI 一致。

## 4.2 完整示例

```java
public String chatWithTools(String question) {
    List<ChatMessage> messages = new ArrayList<>();
    messages.add(SystemMessage.from("你是助手，可以使用工具。"));
    messages.add(UserMessage.from(question));

    for (int step = 0; step < MAX_STEPS; step++) {
        ChatResponse response = model.chat(ChatRequest.builder()
                .messages(messages)
                .toolSpecifications(ToolSpecifications
                        .toolSpecificationsFrom(tools.toArray()))
                .build());

        AiMessage ai = response.aiMessage();
        messages.add(ai);

        if (!ai.hasToolExecutionRequests()) {
            return ai.text();
        }

        for (ToolExecutionRequest req : ai.toolExecutionRequests()) {
            String result = executeTool(req);
            messages.add(ToolExecutionResultMessage.from(req, result));
        }
    }
    return "达到最大步数";
}
```

## 4.3 两框架对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 工具描述 | ToolCallback / 注解 | ToolSpecification |
| 调用结果消息 | ToolResponseMessage | ToolExecutionResultMessage |
| 自动执行 | ChatClient 内置 | AiServices 内置 |
| 循环控制 | 手动 / 内置 | 手动 / 内置 |

## 4.4 手动构造工具规格

结论先行：当工具是动态生成（例如来自数据库配置）时，注解无法覆盖，需要手动构造 `ToolSpecification`。

```java
ToolSpecification spec = ToolSpecification.builder()
        .name("queryOrders")
        .description("按日期范围查询订单")
        .parameters(JsonObjectSchema.builder()
                .addStringProperty("startDate", "开始日期 yyyy-MM-dd")
                .addStringProperty("endDate", "结束日期 yyyy-MM-dd")
                .required("startDate", "endDate")
                .build())
        .build();
```

> 上述 `JsonObjectSchema` 的 API 为示意，不同版本类名与构建方法可能不同。

# 5. 并行工具调用

## 5.1 结论

结论先行：部分模型一次返回多个 tool_calls；当这些调用之间**无依赖**时，可以并行执行以降低总耗时。

## 5.2 完整示例

### 5.2.1 并行执行

```java
// 模型返回：
// tool_calls: [
//   {name: "getWeather", args: {"city":"北京"}},
//   {name: "getWeather", args: {"city":"上海"}}
// ]

// 并行执行（无依赖时）
List<CompletableFuture<ToolResult>> futures = calls.stream()
        .map(call -> CompletableFuture.supplyAsync(
                () -> executeTool(call), executor))
        .toList();

List<ToolResult> results = futures.stream()
        .map(CompletableFuture::join)
        .toList();
```

### 5.2.2 有依赖时串行

若第二个调用依赖第一个的结果（例如先查订单号、再查物流），必须串行，由模型在下一轮发起第二次调用。

## 5.3 边界与坑

> 并行能显著降低多工具调用的总耗时，但要注意下游承载能力。

- 并行度要有上限，避免打爆下游限流。
- 单个工具超时不应拖垮整批，使用 `orTimeout` 与降级结果。
- 结果回填顺序要与 `tool_calls` 顺序对应（或严格按 id 关联）。

### 5.3.1 并行度与超时控制

```java
// 限定并行度，避免打爆下游
ExecutorService executor = new ThreadPoolExecutor(
        2, 8, 60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(64),
        new ThreadPoolExecutor.CallerRunsPolicy());

CompletableFuture<ToolResult> f = CompletableFuture
        .supplyAsync(() -> executeTool(call), executor)
        .orTimeout(3, TimeUnit.SECONDS)
        .exceptionally(e -> ToolResult.fail("工具超时"));
```

# 6. 工具 ID 关联

## 6.1 结论

结论先行：**必须用 id 关联请求与响应**，否则多工具调用时会错位。

## 6.2 示例

```
模型返回 tool_call 带 id：
  {id: "call_abc", name: "getOrder", args: {...}}

回填时用同一 id：
  ToolMessage(tool_call_id: "call_abc", content: "...")
```

## 6.3 边界与坑

- 不要用"调用顺序"来关联，并行或乱序时会错位。
- 缺失 id 的旧协议（如部分本地模型）要用名称 + 参数哈希兜底。
- 回填时务必回传模型原始生成的 id 字符串。

# 7. 多轮工具调用的终止

## 7.1 结论

结论先行：必须有终止条件，否则会出现无限循环、重复调用与资源耗尽。最基本的是**步数上限**与**签名去重**。

## 7.2 LoopGuard 示例

### 7.2.1 步数上限

```java
public class LoopGuard {

    private static final int MAX_STEPS = 10;
    private final Set<String> executed = new HashSet<>();

    public boolean shouldStop(int step, ToolCall call) {
        // ① 步数上限
        if (step >= MAX_STEPS) return true;

        // ② 重复调用
        String sig = call.name() + ":" + call.args();
        if (!executed.add(sig)) {
            log.warn("重复调用: {}", sig);
            return true;
        }
        return false;
    }
}
```

### 7.2.2 签名去重

签名由"工具名 + 参数"构成。同一签名重复出现，通常意味着模型没有从上次结果中获取新信息，应停止并返回当前最好结果。

## 7.3 其他终止条件

| 条件 | 说明 |
|------|------|
| 步数上限 | 最基础的护栏 |
| 签名去重 | 防止重复调用 |
| 总 Token 预算 | 防止上下文爆炸 |
| 总耗时上限 | 面向用户超时 |
| 显式结束标记 | 模型声明任务完成 |

# 8. 工具结果回填格式

## 8.1 结论

结论先行：结果要**精简且结构化**，错误也要结构化，便于模型理解与自我调整。

## 8.2 示例

```java
// 结果要精简且结构化
public String formatResult(ToolResult result) {
    if (!result.success()) {
        // 错误也要结构化，便于模型理解
        return objectMapper.writeValueAsString(Map.of(
                "success", false,
                "error", result.error()));
    }
    return objectMapper.writeValueAsString(result.data());
}
```

> **不要回填整个对象的 toString()**，会浪费大量 Token。

## 8.3 边界与坑

- 大结果要截断并标注"已截断"，提示模型可缩小范围重查。
- 二进制内容（图片、文件）应转为 URL 或摘要。
- 字段名用稳定的英文 key，避免模型解析歧义。

## 8.4 大结果的截断与摘要

结论先行：单个工具结果建议控制在数千 Token 以内；超出时截断并显式标注，提示模型可缩小范围重查。

| 结果规模 | 处理方式 |
|----------|----------|
| 小于 1 KB | 原样回填 |
| 1 KB ~ 8 KB | 原样或精简字段 |
| 大于 8 KB | 截断 + 提示缩小范围 |
| 二进制内容 | 转 URL 或摘要 |

```java
private String truncate(String json, int maxChars) {
    if (json.length() <= maxChars) {
        return json;
    }
    return json.substring(0, maxChars)
            + "...(结果已截断，可缩小查询范围后重试)";
}
```

# 9. 错误处理策略

## 9.1 结论

结论先行：**返回结构化错误，而非抛异常**，让模型能自我调整。

## 9.2 示例

```java
public String executeTool(ToolCall call) {
    try {
        return formatResult(doExecute(call));
    } catch (ToolTimeoutException e) {
        return formatError("工具超时，请稍后重试或换用其他方式");
    } catch (AccessDeniedException e) {
        return formatError("无权限执行该操作");
    } catch (IllegalArgumentException e) {
        return formatError("参数错误：" + e.getMessage());
    } catch (Exception e) {
        return formatError("工具执行失败：" + e.getMessage());
    }
}
```

## 9.3 错误分类表

| 错误类型 | 是否可重试 | 回填建议 |
|----------|-----------|----------|
| 参数错误 | 否，需改参数 | 指出哪个参数非法 |
| 超时 | 是 | 建议重试或换工具 |
| 无权限 | 否 | 明确拒绝，不暴露细节 |
| 下游 5xx | 视情况 | 建议稍后重试 |
| 业务规则拒绝 | 否 | 说明业务原因 |

## 9.4 重试策略

结论先行：只对"可重试错误"重试，且必须配合幂等键，否则会放大副作用。

| 错误 | 重试 | 退避策略 |
|------|------|----------|
| 超时 | 是 | 指数退避 |
| 限流 429 | 是 | 按 Retry-After |
| 参数错误 | 否 | 不重试，让模型改参 |
| 业务拒绝 | 否 | 直接返回原因 |
| 下游 5xx | 视情况 | 有限次 + 退避 |

# 10. 生产注意事项

## 10.1 风险表

| 风险 | 建议 |
|------|------|
| 无步数上限 | 必设最大步数 |
| 重复调用 | 签名去重 |
| 结果过大 | 精简 + 截断 |
| 异常裸抛 | 结构化错误 |
| id 未关联 | 用 tool_call_id 关联 |
| 串行执行慢 | 无依赖时并行 |
| 无超时 | 每个工具设超时 |

## 10.2 可观测性

结论先行：把每轮的工具名、参数摘要、耗时、结果大小、是否重试都记录为结构化日志，是排查 Agent 问题的前提。

```java
log.info("tool_call step={} name={} args={} costMs={} resultBytes={}",
        step, call.name(), abbreviate(call.arguments()),
        cost, resultBytes);
```

# 11. 小结

## 11.1 核心结论

- 工具调用是"决策-执行-回填"的循环。
- 消息序列必须完整保留。
- 用 tool_call_id 关联请求与响应。
- 无依赖的工具调用可并行。
- 错误要结构化返回，并设步数与超时上限。

## 11.2 落地清单

- 设定最大步数、总耗时、Token 预算三道护栏。
- 每轮记录结构化日志。
- 并行调用设并行度上限与单工具超时。
- 错误分类处理，明确哪些可重试。
