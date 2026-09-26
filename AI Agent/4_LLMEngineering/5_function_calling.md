# 1. 原理

函数调用（Function Calling），又称工具调用（Tool Calling），让大语言模型（LLM, Large Language Model）在生成回答前先"申请"执行外部函数，从而获得实时数据或副作用能力。

## 1.1 四步闭环

结论先行：函数调用本质是一个"请求—执行—回填—再生成"的闭环，模型本身不执行任何代码，执行永远在客户端。

### 1.1.1 请求携带工具定义

客户端把可用工具的名称、描述与参数 Schema（模式）随消息一起发送。

### 1.1.2 模型返回 tool_calls

模型判断需要调用工具时，不直接回答，而是返回结构化的 `tool_calls` 数组。

### 1.1.3 客户端执行与回填

客户端解析参数、执行真实函数，把结果以 `tool` 角色消息回填。

### 1.1.4 模型生成最终回答

模型基于工具结果生成自然语言回答，闭环结束。

```
① 请求中带上工具定义（Schema）
② 模型判断需要调用工具 → 返回 tool_calls
③ 客户端执行工具 → 回填结果
④ 模型基于结果生成回答
```

## 1.2 与结构化输出的关系

函数调用可视为结构化输出的一种特例：`tool_calls` 的 `arguments` 就是受工具 Schema 约束的 JSON。区别在于它多了一个"执行"环节。

## 1.3 本文范围

> 详细机制见 `7_ToolsAndProtocols/3_tool_calling_mechanics.md`，本文聚焦 LLM 工程视角。

# 2. 请求格式

结论先行：请求的核心是 `tools` 数组，其中 `parameters` 用 JSON Schema 描述参数，描述质量直接决定模型调用准确率。

## 2.1 消息与工具定义

```json
{
  "model": "gpt-4o-mini",
  "messages": [{"role": "user", "content": "北京天气如何？"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "getWeather",
      "description": "查询城市天气",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {"type": "string", "description": "城市名"}
        },
        "required": ["city"]
      }
    }
  }]
}
```

## 2.2 参数 Schema 细节

### 2.2.1 必填与默认

`required` 声明必填字段；可选字段可在描述中说明默认行为，例如"不传则默认当天"。

### 2.2.2 枚举与嵌套

```json
{
  "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
  "location": {
    "type": "object",
    "properties": {
      "lat": {"type": "number"},
      "lon": {"type": "number"}
    }
  }
}
```

枚举能显著减少模型自由发挥；嵌套对象适合复合参数。

## 2.3 边界与坑

| 坑 | 说明 |
|----|------|
| 描述含糊 | 模型不知道何时调用 |
| 参数过泛 | `type: string` 无法约束取值 |
| 工具过多 | 模型选择困难、Token 暴涨 |
| 名称冲突 | 不同工具同名导致歧义 |

# 3. 响应格式

结论先行：响应里出现 `tool_calls` 且 `finish_reason` 为 `tool_calls`，表示模型在等待客户端执行。

## 3.1 tool_calls 结构

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "tool_calls": [{
        "id": "call_abc",
        "type": "function",
        "function": {
          "name": "getWeather",
          "arguments": "{\"city\":\"北京\"}"
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

注意 `arguments` 是**字符串**，内部才是 JSON，需要二次解析。

## 3.2 finish_reason 语义

### 3.2.1 tool_calls

模型请求调用工具，需执行后继续对话。

### 3.2.2 stop

模型正常结束，直接输出文本。

### 3.2.3 length

达到 `maxTokens`（最大输出词元）被截断，`arguments` 可能不完整。

## 3.3 边界与坑

- `arguments` 可能为空串或非法 JSON。
- `length` 截断时不可强行解析，应重试并提高上限。
- 部分厂商字段名不同（如 `function_call` 旧格式）。

# 4. 回填结果

结论先行：回填必须携带 `tool_call_id`，它是请求与响应的唯一关联键。

## 4.1 tool 角色消息

```json
{
  "messages": [
    {"role": "user", "content": "北京天气如何？"},
    {"role": "assistant", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "call_abc", "content": "北京：晴，25℃"}
  ]
}
```

## 4.2 tool_call_id 关联

### 4.2.1 单工具

只有一个调用时，id 错配通常也能工作，但不要依赖这一行为。

### 4.2.2 多工具错位

> **必须用 `tool_call_id` 关联**，否则多工具调用会错位。

并行调用时若 id 与结果错位，模型会基于错误数据推理，产生难以察觉的幻觉。

## 4.3 边界与坑

- 每个 `tool_call` 都必须有且仅有一条对应回填。
- 回填数量不足，部分厂商直接报 400 错误。
- `content` 建议为字符串；结构化结果需先序列化。

# 5. Spring AI 实现

结论先行：Spring AI 用 `@Tool` 注解声明工具，方法签名即参数 Schema。

## 5.1 声明工具

```java
@Component
public class WeatherTools {

    @Tool(description = "查询指定城市的天气")
    String getWeather(@ToolParam(description = "城市名称") String city) {
        return city + "：晴，25℃";
    }
}
```

## 5.2 注册与调用

### 5.2.1 方法级注册

```java
String answer = chatClient.prompt()
        .user("北京天气如何？")
        .tools(new WeatherTools())
        .call()
        .content();
```

### 5.2.2 全局默认工具

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, WeatherTools tools) {
    return builder
            .defaultTools(tools)     // 所有请求默认可用
            .build();
}
```

## 5.3 生产建议

- 工具方法保持幂等、无副作用或显式声明副作用。
- 返回值尽量精简，避免超大 JSON 回填。
- 用 `@ToolParam(required=false)` 表达可选参数。

# 6. LangChain4j 实现

结论先行：LangChain4j 用 `@Tool` 与 `@P` 声明，通过 `AiServices` 装配到接口代理。

## 6.1 声明工具

```java
public class WeatherTools {

    @Tool("查询指定城市的天气")
    String getWeather(@P("城市名称") String city) {
        return city + "：晴，25℃";
    }
}
```

## 6.2 AiServices 装配

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new WeatherTools())
        .build();
```

## 6.3 与 Spring AI 对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 声明注解 | `@Tool` / `@ToolParam` | `@Tool` / `@P` |
| 装配方式 | ChatClient | AiServices |
| 依赖注入 | 原生 Spring | 手动或集成 |
| 适用 | Spring 应用 | 通用 Java |

## 6.4 生产建议

- 工具类无状态，便于并发。
- 需要鉴权时在工具内读取上下文（如租户、用户）。
- 统一异常返回格式，便于模型修正。

# 7. 并行工具调用

结论先行：多个无依赖的工具调用应并行执行，可显著降低端到端延迟。

## 7.1 触发条件

模型在一次响应中返回多个 `tool_calls` 时，即产生并行调用机会。

```json
{"tool_calls": [
  {"id": "call_1", "function": {"name": "getWeather", "arguments": "{\"city\":\"北京\"}"}},
  {"id": "call_2", "function": {"name": "getWeather", "arguments": "{\"city\":\"上海\"}"}}
]}
```

## 7.2 并行执行实现

### 7.2.1 CompletableFuture

```java
// 并行执行
List<CompletableFuture<ToolResult>> futures = calls.stream()
        .map(c -> CompletableFuture.supplyAsync(() -> execute(c), executor))
        .toList();
```

### 7.2.2 超时与聚合

```java
List<ToolResult> results = futures.stream()
        .map(f -> f.orTimeout(3, TimeUnit.SECONDS)
                .exceptionally(e -> ToolResult.error(e.getMessage())))
        .map(CompletableFuture::join)
        .toList();
```

## 7.3 边界与坑

- 有依赖关系的调用不可并行（如先查 ID 再查详情）。
- 线程池需隔离，避免与业务线程池互相拖垮。
- 任一工具超时应回传错误而非抛出，让模型决定下一步。

## 7.4 生产建议

- 为每个工具设置独立超时。
- 限制单次并行的最大数量，防雪崩。
- 记录每个工具的耗时分布。

# 8. 强制工具调用

结论先行：`tool_choice` 控制模型是否必须调用工具，是流程编排的关键开关。

## 8.1 tool_choice 取值

```java
// 强制模型必须调用某个工具
ChatOptions options = ChatOptions.builder()
        .toolChoice("getWeather")          // 或 "required" / "auto" / "none"
        .build();
```

| 取值 | 含义 |
|------|------|
| `auto` | 模型自主决定 |
| `required` | 必须调用工具 |
| `none` | 禁止调用 |
| 具体工具名 | 必须调用该工具 |

## 8.2 使用场景

- 抽取任务：`required` 保证一定走工具。
- 纯聊天：`none` 避免误调用。
- 固定流程：指定工具名做确定性编排。

## 8.3 边界与坑

- 指定工具名时，若参数不足，模型仍可能编造参数。
- `required` 与不支持该取值的厂商不兼容，需降级。
- 强制调用会牺牲模型的自主性，慎用于探索型任务。

# 9. 多轮工具调用

结论先行：多轮循环必须设置最大步数，否则模型可能陷入反复调用。

## 9.1 循环控制

```
循环：请求 → 是否有 tool_calls？
  是 → 执行并回填 → 继续循环
  否 → 返回文本，结束
达到最大步数 → 强制结束
```

## 9.2 完整实现

### 9.2.1 消息累积

```java
public String chatWithTools(String question) {
    List<Message> messages = new ArrayList<>();
    messages.add(UserMessage.from(question));

    for (int step = 0; step < MAX_STEPS; step++) {
        ChatResponse response = chatClient.prompt()
                .messages(messages)
                .tools(tools)
                .call()
                .chatResponse();

        AiMessage ai = response.getResult().getOutput();
        messages.add(ai);

        if (!ai.hasToolCalls()) {
            return ai.getText();
        }

        for (ToolCall call : ai.getToolCalls()) {
            messages.add(new ToolResponseMessage(List.of(
                    new ToolResponse(call.id(), call.name(),
                            executeTool(call)))));
        }
    }
    return "达到最大步数";
}
```

### 9.2.2 终止条件

- 模型不再返回工具调用。
- 达到最大步数。
- 累计 Token 或耗时超过预算。

## 9.3 边界与坑

- 消息列表随轮次增长，Token 消耗快速上升。
- 重复调用同一工具往往是提示不清或工具返回值无效。
- 必须保留 assistant 的 `tool_calls` 消息，否则回填无法关联。

## 9.4 生产建议

- 最大步数建议 5~10。
- 对重复调用做去重或短路。
- 每轮记录工具名与参数，便于排查。

# 10. 参数错误的处理

结论先行：参数错误不应抛出异常中断，而应回传结构化错误让模型自我修正。

## 10.1 错误分类

| 错误 | 原因 |
|------|------|
| JSON 解析失败 | `arguments` 非法 |
| 类型不符 | 数字传成字符串 |
| 业务非法 | 城市不存在 |
| 鉴权失败 | 无权限 |

## 10.2 回传结构化错误

```java
// 模型生成的参数可能格式错误
public String executeTool(ToolCall call) {
    try {
        Map<String, Object> args = parseArgs(call.arguments());
        return doExecute(call.name(), args);
    } catch (JsonParseException e) {
        // 参数解析失败 → 回传错误让模型修正
        return "参数格式错误，请检查 JSON 格式：" + e.getMessage();
    } catch (IllegalArgumentException e) {
        return "参数不合法：" + e.getMessage();
    }
}
```

## 10.3 校验与重试

```java
// 伪代码：校验参数后再执行，失败则返回可读原因
if (!"北京".equals(city) && !knownCities.contains(city)) {
    return "未知城市：" + city + "，请使用标准城市名";
}
```

## 10.4 生产建议

- 错误信息要具体、可操作，模型才能修正。
- 限制同一工具的错误重试次数。
- 统计高频参数错误，反推 Schema 描述问题。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 无限循环 | 最大步数 |
| 参数错误 | 回传结构化错误 |
| 结果过大 | 精简 + 截断 |
| 工具过多 | 按场景裁剪 |
| 无鉴权 | 工具内校验 |
| 无超时 | 每个工具设超时 |

## 11.2 可观测性

```java
meterRegistry.timer("tool.latency", "tool", name).record(duration);
meterRegistry.counter("tool.error", "tool", name, "type", errorType).increment();
```

## 11.3 安全与鉴权

- 工具是权限边界，必须在工具内做鉴权与参数白名单。
- 防止提示注入（Prompt Injection）诱导调用危险工具。
- 写操作工具建议引入人工确认或幂等键。

# 12. 小结

## 12.1 要点回顾

- 函数调用让模型能执行真实操作。
- 请求带 Schema，响应返回 tool_calls，客户端执行后回填。
- 用 `tool_call_id` 关联请求与响应。
- 无依赖的工具调用可并行。
- 参数错误要回传，让模型自我修正。

## 12.2 落地清单

- 工具描述清晰、参数用枚举收敛。
- 多轮循环设最大步数。
- 每个工具设超时并回传结构化错误。
- 工具内做鉴权与参数白名单。
- 采集工具耗时、错误率与调用次数。
