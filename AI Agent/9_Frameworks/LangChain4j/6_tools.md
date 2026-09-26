# 1. 原理回顾

**结论先行**：工具调用（Tool Calling）的本质是"模型提出调用意图，框架反射执行 Java 方法，再把结果回填给模型"；LangChain4j 用 `@Tool` 注解把普通 Java 方法暴露给模型，并自动完成调用循环。

## 1.1 工具调用循环

```
用户提问
   │
   ▼
模型决定：调用 getWeather(city="北京")
   │
   ▼
LangChain4j 反射执行 Java 方法
   │
   ▼
把结果作为 ToolExecutionResultMessage 回填
   │
   ▼
模型结合结果生成最终回答
```

### 1.1.1 完整时序

```
业务 ──chat()──► AiService ──► 模型
                                 │ 返回 tool_calls
                                 ▼
                            执行 @Tool 方法
                                 │
                                 ▼
                    结果回填 → 再调用模型（可多轮）
                                 │
                                 ▼
                            最终文本回答
```

## 1.2 消息模型

### 1.2.1 ToolExecutionResultMessage

工具执行结果以 `ToolExecutionResultMessage` 的形式追加进消息列表，与 `AiMessage` 中的工具调用请求通过 ID 对应。模型据此判断"上一次请求是否已被满足"。

## 1.3 边界与坑

- 工具调用不是"模型直接执行代码"，而是模型输出结构化意图、由框架执行；
- 模型可能提出不存在的工具或错误参数，框架需容错；
- 结果回填的格式直接影响模型能否自我纠错，务必结构化。

# 2. 定义工具

**结论先行**：用 `@Tool` 标注方法、用 `@P` 描述参数，方法描述与参数描述的质量直接决定模型调用准确率。

## 2.1 @Tool 基础

```java
@Component
public class WeatherTools {

    @Tool("查询指定城市的当前天气")
    String getWeather(@P("城市名称，如：北京") String city) {
        return city + "：晴，25℃，微风";
    }
}
```

| 注解 | 作用 |
|------|------|
| `@Tool` | 标记方法为可调用工具，值即工具描述 |
| `@P` | 描述参数含义 |

> 描述质量直接决定模型调用准确率，**务必写清"何时用、参数是什么、返回什么"**。

### 2.1.1 描述要素

| 要素 | 示例 |
|------|------|
| 用途 | "查询指定城市的当前天气" |
| 触发场景 | "当用户询问实时天气时使用" |
| 参数格式 | "城市名称，如：北京" |
| 返回值 | "返回天气状况与温度" |
| 限制 | "仅支持中国城市" |

### 2.1.2 命名规范

工具方法名建议用"动词 + 名词"（如 `getWeather`、`createOrder`），并与 `@Tool` 描述保持一致，便于模型建立"意图到方法"的映射。

## 2.2 复杂返回类型

```java
record WeatherInfo(String city, String condition, int temperature) {}

@Tool("查询城市天气，返回结构化信息")
WeatherInfo getWeatherDetail(@P("城市名称") String city) {
    return new WeatherInfo(city, "晴", 25);
}
```

返回对象会被序列化后回填给模型，结构化结果比长文本更利于模型理解。

## 2.3 可选参数

```java
@Tool("按日期范围查询订单")
List<Order> queryOrders(
        @P("开始日期 yyyy-MM-dd") String startDate,
        @P("结束日期 yyyy-MM-dd") String endDate,
        @P(value = "订单状态 PAID/CANCELLED，可省略", required = false) String status) {
    ...
}
```

## 2.4 边界与坑

- 工具方法不应依赖 ThreadLocal（回调可能在其他线程），需显式传参；
- 参数类型要能被模型正确生成，优先用 `String`/`int`/枚举，避免复杂自定义类型入参；
- 一个类里的工具会全部暴露，注意不要把内部方法也标成 `@Tool`。

# 3. 注册工具

**结论先行**：在 `AiServices` 里用 `tools(...)` 挂载最省事；需要精细控制时走低层 API；生产必须按用户权限动态裁剪工具集。

## 3.1 在 AiServices 中

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new WeatherTools(), new OrderTools())
        .build();
```

## 3.2 在低层 API 中

```java
ChatRequest request = ChatRequest.builder()
        .messages(UserMessage.from("北京天气如何？"))
        .toolSpecifications(ToolSpecifications.toolSpecificationsFrom(new WeatherTools()))
        .build();

ChatResponse response = model.chat(request);
```

低层方式需自行处理工具调用循环，灵活性高但成本也高。

## 3.3 动态工具

```java
List<Object> tools = permissionService.allowedTools(currentUser());
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(tools)          // 只挂载有权限的工具
        .build();
```

> **不要依赖"模型不会调用"来保证安全**，必须按权限裁剪工具集。详见 `15_Security/8_tool_permission_control.md`。

### 3.3.1 权限模型

| 粒度 | 做法 |
|------|------|
| 按角色 | 角色到工具白名单 |
| 按租户 | 租户维度隔离工具集 |
| 按会话 | 临时授权，会话结束即失效 |

## 3.4 边界与坑

| 风险 | 说明 | 对策 |
|------|------|------|
| 全量挂载 | 越权风险 | 按权限裁剪 |
| 工具过多 | 模型选择困难 | 按场景分组 |
| 重复注册 | 同名工具冲突 | 唯一命名 |

# 4. 工具执行结果

**结论先行**：工具不要抛异常，而要返回结构化的成功/失败信息，让模型能读取错误并自我纠错。

## 4.1 结构化结果

```java
record ToolResult(boolean success, String data, String error) {}

@Tool("查询库存")
ToolResult queryStock(@P("商品编码") String sku) {
    try {
        return new ToolResult(true, String.valueOf(stockService.get(sku)), null);
    } catch (Exception e) {
        return new ToolResult(false, null, "查询失败：" + e.getMessage());
    }
}
```

原则：**工具不要抛异常，返回结构化成功/失败信息**，让模型能自我纠错。

### 4.1.1 字段设计

`data` 应保持精简且可直接被模型引用；复杂对象优先返回关键字段而非整行记录，以降低 Token 占用。

## 4.2 失败语义

| 字段 | 含义 | 示例 |
|------|------|------|
| `success` | 是否成功 | `true` / `false` |
| `data` | 成功时的数据 | `"25"` |
| `error` | 失败时的原因 | `"查询超时"` |

## 4.3 边界与坑

- 结果过长会挤占上下文，需精简；
- 错误信息要"可操作"，例如"库存不足，剩 3 件"而非堆栈；
- 不要把敏感内部信息（SQL、内网地址）写进 `error`。

# 5. 工具的错误处理策略

**结论先行**：错误处理分三种策略——回传错误给模型、抛出异常中断、返回兜底结果；默认策略是把错误回传给模型。

## 5.1 回传错误

LangChain4j 提供可配置的错误处理：

```java
// 工具执行异常时，把错误信息回传给模型（默认）
.toolExecutionErrorHandler((error, ctx) ->
        ToolExecutionResultMessage.from(ctx.toolExecutionRequest(),
                "工具执行失败：" + error.getMessage()))
```

### 5.1.1 默认行为

默认的错误处理器会把异常消息回传，模型看到后通常会重试或改用其他参数；若业务不允许重试，应替换为抛出异常的策略。

## 5.2 抛异常与兜底

| 策略 | 说明 | 适用 |
|------|------|------|
| 回传错误给模型 | 模型自行纠错 | 参数错误、临时失败 |
| 抛出异常 | 中断调用 | 权限不足、严重故障 |
| 返回兜底结果 | 降级 | 外部依赖不可用 |

## 5.3 版本差异

> 1.20 异步路径下默认行为不同：工具**执行**错误会导致调用失败，而参数**解析**错误会回传给模型。使用前请核对版本文档。

## 5.4 边界与坑

- 回传错误可能被模型"幻觉式"忽略，关键错误仍应硬中断；
- 异常信息可能泄露内部实现，需脱敏；
- 错误处理策略要与重试策略协同，避免"回传 + 自动重试"造成重复执行。

# 6. 多轮工具调用

**结论先行**：复杂任务常需要多轮"调用—回填—再调用"，框架自动完成；但必须限制最大轮数，防止无限循环。

## 6.1 调用链

```java
String answer = assistant.chat("查一下订单 A123 的物流状态");
// 内部：queryOrder("A123") → 得到运单号 → queryLogistics(运单号) → 汇总
```

LangChain4j 自动完成多轮"调用-回填-再调用"，直到模型给出最终答案。

### 6.1.1 依赖传递

当第二个工具需要第一个工具的输出时，模型会把上一步结果作为参数传入；因此工具返回值必须包含下游需要的标识（如运单号）。

## 6.2 最大轮数

```
轮次 1：queryOrder("A123")      → 运单号
轮次 2：queryLogistics(运单号)  → 物流状态
轮次 3：模型汇总 → 最终回答
```

> 注意设置**最大调用轮数**，防止无限循环。

## 6.3 边界与坑

- 轮数越多，Token 成本与延迟越高；
- 工具间存在隐式依赖时，模型可能选错顺序，需在描述里说明前置条件；
- 达到最大轮数后要有兜底回复，不能直接抛错给用户。

# 7. 工具设计原则

**结论先行**：好的工具是"单一职责、命名清晰、描述完整、结果结构化、不抛异常、幂等、有超时、有鉴权"。

## 7.1 接口设计

| 原则 | 说明 |
|------|------|
| 单一职责 | 一个工具只做一件事 |
| 命名清晰 | 动词开头，语义明确 |
| 描述完整 | 说清用途、参数、返回值、限制 |

### 7.1.1 粒度选择

粒度以"模型一次决策能完成"为准：过粗则参数难以填准，过细则轮数激增。通常一个业务动作对应一个工具。

## 7.2 结果设计

| 原则 | 说明 |
|------|------|
| 结果结构化 | 便于模型理解与后续处理 |
| 不抛异常 | 用结构化错误 |
| 幂等 | 写操作必须幂等 |
| 超时 | 内部调用外部依赖必须设超时 |
| 鉴权 | 工具内部校验权限 |

## 7.3 边界与坑

- "万能工具"（一个方法做很多事）会让模型难以正确填参；
- 工具粒度太细会显著增加调用轮数与成本；
- 描述里不要写"调用前需确认"之类无法被模型执行的话。

# 8. 生产级工具示例

**结论先行**：生产级工具四要素是**鉴权、超时、幂等、结构化错误**，下面一个示例全部覆盖。

## 8.1 鉴权

```java
String userId = securityContext.currentUserId();

if (!orderService.belongsTo(userId, orderId)) {
    return ToolResult.fail("无权查询该订单");     // 鉴权
}
```

## 8.2 超时与幂等

```java
Order order = orderService.getWithTimeout(
        orderId, Duration.ofSeconds(3));          // 超时
```

```java
return ToolResult.ok(
        orderService.createIdempotent(idempotencyKey, sku, quantity));  // 幂等
```

## 8.3 完整示例

```java
@Component
@RequiredArgsConstructor
public class OrderTools {

    private final OrderService orderService;
    private final SecurityContext securityContext;

    @Tool("查询订单详情，只能查询当前用户自己的订单")
    public ToolResult getOrder(@P("订单号，如 A123") String orderId) {
        String userId = securityContext.currentUserId();

        if (!orderService.belongsTo(userId, orderId)) {
            return ToolResult.fail("无权查询该订单");     // 鉴权
        }
        try {
            Order order = orderService.getWithTimeout(
                    orderId, Duration.ofSeconds(3));      // 超时
            return ToolResult.ok(order);
        } catch (TimeoutException e) {
            return ToolResult.fail("查询超时，请稍后重试"); // 结构化错误
        } catch (Exception e) {
            return ToolResult.fail("查询失败：" + e.getMessage());
        }
    }

    @Tool("创建订单，同一幂等键只会创建一次")
    public ToolResult createOrder(
            @P("幂等键，唯一标识本次请求") String idempotencyKey,
            @P("商品编码") String sku,
            @P("数量") int quantity) {
        return ToolResult.ok(
                orderService.createIdempotent(idempotencyKey, sku, quantity));
    }
}
```

> 这一个示例覆盖了工具生产化的全部要点：**鉴权、超时、结构化错误、幂等**。

### 8.3.1 要点对照

| 要点 | 在示例中的体现 |
|------|----------------|
| 鉴权 | `belongsTo` 校验归属 |
| 超时 | `getWithTimeout` 3 秒 |
| 幂等 | `createIdempotent` |
| 结构化错误 | `ToolResult.fail` |

## 8.4 边界与坑

- `@RequiredArgsConstructor` 注入的依赖必须线程安全；
- 幂等键要由调用方生成并保证唯一，不能由模型编造；
- 鉴权失败的信息不要暴露"订单是否存在"，避免信息泄露。

# 9. 工具与可观测

**结论先行**：通过监听器（Listener）记录工具名、参数、耗时与结果状态，是工具可观测的基础。

## 9.1 监听器

```java
ChatModel model = OpenAiChatModel.builder()
        .apiKey(key)
        .modelName("gpt-4o-mini")
        .listeners(List.of(new ToolCallLoggingListener()))
        .build();
```

监听器可记录：工具名、参数、耗时、结果状态，用于 Trace 与评测。

### 9.1.1 可记录字段

工具名、请求 ID、入参、开始与结束时间、结果状态、错误类型；这些字段足以支撑 Trace 与成功率统计。

## 9.2 指标与追踪

| 指标 | 用途 |
|------|------|
| 调用次数 | 评估工具热度 |
| 成功率 | 发现不稳定工具 |
| 耗时分布 | 定位慢依赖 |
| 参数分布 | 优化描述 |

## 9.3 边界与坑

- 监听器不要做重活，避免拖慢主链路；
- 日志中的参数可能含个人信息，需脱敏；
- 异步路径下监听器可能并发执行，注意线程安全。

# 10. 常见坑

**结论先行**：工具相关故障可归为"调用不准、参数出错、结果被忽略、无限循环、重复写、越权"六类，多数可通过优化描述与权限裁剪解决。

## 10.1 调用准确率

| 现象 | 原因 | 解决 |
|------|------|------|
| 模型不调用工具 | 描述不清 | 优化 `@Tool` 描述，写明适用场景 |
| 参数类型错误 | `@P` 描述模糊 | 明确格式与取值范围 |
| 工具结果被忽略 | 返回过长/不结构化 | 返回精简结构化结果 |

### 10.1.1 描述优化技巧

在描述里加入"当用户询问……时使用"的场景句，以及参数示例值；必要时给出 1~2 个调用示例，能明显提升命中率。

## 10.2 安全与稳定性

| 现象 | 原因 | 解决 |
|------|------|------|
| 无限循环 | 缺少终止条件 | 限制轮数、优化提示 |
| 写操作重复 | 重试 + 非幂等 | 幂等键 + 去重 |
| 越权调用 | 工具集未按权限裁剪 | 按用户授权动态挂载 |

## 10.3 边界与坑

- 不同模型对工具描述的理解差异较大，换模型后需回归测试；
- 工具数量超过一定规模后，模型选择准确率会下降，需分场景裁剪；
- 工具返回的时区、单位要显式标注，避免模型误读。

# 11. 生产注意事项

**结论先行**：工具的生产化风险集中在"权限、幂等、超时、可观测"四类，逐项设防即可。

## 11.1 稳定性

| 风险 | 建议 |
|------|------|
| 外部依赖慢 | 必设超时 + 熔断 |
| 无限循环 | 限制最大调用轮数 |
| 重复写 | 幂等键 + 唯一约束 |
| 工具过多 | 按场景分组裁剪 |

### 11.1.1 熔断降级

外部依赖连续失败时应触发熔断，工具直接返回兜底结果，避免模型反复调用同一个不可用工具。

## 11.2 安全与合规

| 风险 | 建议 |
|------|------|
| 越权 | 工具内部二次鉴权 |
| 注入 | 工具参数做白名单校验 |
| 数据泄露 | 错误信息与日志脱敏 |
| 危险操作 | 高风险工具走人工确认 |

## 11.3 可观测

- 记录每次工具调用的入参、耗时、结果状态；
- 对失败率高的工具设告警；
- 采样失败案例，用于优化工具描述与提示。

# 12. 小结

- `@Tool` + `@P` 把 Java 方法变成模型可调用的能力。
- 描述质量决定调用准确率，结果结构化决定纠错能力。
- 生产级工具四要素：**鉴权、超时、幂等、结构化错误**。
- 工具集要按权限动态裁剪，而非全量挂载。
- 多轮调用要限制轮数，并用监听器补齐可观测。
