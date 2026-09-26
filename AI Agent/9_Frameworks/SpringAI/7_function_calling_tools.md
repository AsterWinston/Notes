# 1. 原理回顾

结论先行：工具调用（Tool Calling，也称 Function Calling）把「模型决策」与「代码执行」彻底解耦——模型只输出结构化的调用意图，真正的副作用发生在你的 Java 进程里。

## 1.1 核心结论

工具调用让模型突破"只会说话"的限制，去执行真实操作：

```
用户提问
   │
   ▼
模型判断：需要调用工具 getWeather(city="北京")
   │
   ▼
Spring AI 执行 Java 方法 getWeather("北京")
   │
   ▼
把结果作为 ToolResponseMessage 回填给模型
   │
   ▼
模型结合结果生成最终自然语言回答
```

三条必须刻进骨子里的边界：

- 模型**不执行**任何代码，它只生成"调用哪个工具、传什么参数"；
- Spring AI 只负责解析意图、路由到 Java 方法、回填结果，**不做业务判断**；
- 最终答案仍由模型生成，因此工具结果的**结构与措辞**会直接影响回答质量。

## 1.2 一次调用的完整时序

### 1.2.1 消息角色与数据流

一次工具调用在消息层面会产生 4 类角色：

| 角色 | 说明 | 谁产生 |
|------|------|--------|
| `UserMessage` | 用户原始问题 | 你的代码 |
| `AssistantMessage` | 含 `toolCalls` 的助手消息 | 模型 |
| `ToolResponseMessage` | 工具执行结果 | Spring AI |
| `AssistantMessage` | 最终自然语言答案 | 模型 |

### 1.2.2 时序拆解

```
[1] prompt() 组装 UserMessage
[2] 附带 tools 的 schema 一起发给模型
[3] 模型返回 AssistantMessage(toolCalls=[...])
[4] Spring AI 逐个解析 toolCall -> 反射调用 Java 方法
[5] 把返回值包装成 ToolResponseMessage
[6] 再次调用模型（带完整历史）
[7] 模型产出最终 AssistantMessage
```

> 第 6 步意味着：**一次工具调用至少产生两次模型请求**，Token 成本随之翻倍。

## 1.3 与普通对话的差异

### 1.3.1 对比表

| 维度 | 普通对话 | 工具调用 |
|------|----------|----------|
| 模型输出 | 纯文本 | 结构化 toolCall |
| 执行方 | 无 | 你的 Java 代码 |
| 请求次数 | 1 | ≥2 |
| 失败面 | 模型幻觉 | 幻觉 + 执行异常 |
| 幂等要求 | 无 | 写操作必须幂等 |

### 1.3.2 何时不该用工具

- 问题纯属常识或纯生成任务，无外部数据依赖；
- 工具结果不稳定且模型无法纠错，反而放大错误；
- 实时性要求极高、无法容忍额外一次模型往返。

# 2. 定义工具：@Tool

结论先行：`@Tool` 是声明工具最简洁的方式，工具名默认取方法名，描述取注解的 `description`。

## 2.1 基础写法

### 2.1.1 最小可用示例

```java
@Component
public class WeatherTools {

    @Tool(description = "查询指定城市的当前天气")
    String getWeather(@ToolParam(description = "城市名称，如：北京") String city) {
        // 真实场景应调用天气 API
        return city + "：晴，气温 25℃，微风";
    }
}
```

### 2.1.2 注解属性说明

| 属性 | 位置 | 说明 |
|------|------|------|
| `name` | `@Tool` | 工具名，默认方法名 |
| `description` | `@Tool` | 工具用途，**决定调用准确率** |
| `returnDirect` | `@Tool` | 为 true 时结果直接返回用户，不再经过模型 |
| `description` | `@ToolParam` | 参数说明，写入 JSON Schema |
| `required` | `@ToolParam` | 是否必填，默认 true |

## 2.2 返回值与参数结构

### 2.2.1 结构化返回值

```java
public record WeatherInfo(String city, String condition, int temperature) {}

@Tool(description = "查询城市天气，返回结构化天气信息")
WeatherInfo getWeatherDetail(@ToolParam(description = "城市名称") String city) {
    return new WeatherInfo(city, "晴", 25);
}
```

### 2.2.2 参数约束

```java
@Tool(description = "按日期范围查询订单")
List<Order> queryOrders(
        @ToolParam(description = "开始日期，格式 yyyy-MM-dd") String startDate,
        @ToolParam(description = "结束日期，格式 yyyy-MM-dd") String endDate,
        @ToolParam(description = "订单状态，可选：PAID/CANCELLED", required = false) String status) {
    ...
}
```

## 2.3 方法签名约束

### 2.3.1 可见性与静态性

- 方法必须可被框架调用，建议声明为 `public`；
- 支持实例方法与静态方法；
- 参数类型需能被 JSON Schema 描述（基本类型、`String`、`record`、`List`、`Map`）。

### 2.3.2 线程安全

工具实例可能被并发调用（尤其并行工具调用时），**无状态或线程安全**是硬要求：

```java
@Component
public class SafeTools {

    private final AtomicInteger counter = new AtomicInteger();

    @Tool(description = "生成自增序号")
    int next() {
        return counter.incrementAndGet();
    }
}
```

# 3. 注册工具

结论先行：工具必须"注册"到某次 `prompt()` 或 `ChatClient` 默认配置，模型才可能看到它。

## 3.1 注册方式总览

| 方式 | 作用域 | 适用 |
|------|--------|------|
| `.tools(...)` | 单次调用 | 场景化工具 |
| `defaultTools(...)` | 全局默认 | 通用工具 |
| `.toolCallbacks(...)` | 单次调用 | 底层 `ToolCallback` |
| `defaultToolCallbacks(...)` | 全局默认 | 底层 `ToolCallback` |
| `.toolNames(...)` | 单次调用 | 按名称（版本差异大） |

### 3.1.1 单次调用注册

```java
String answer = chatClient.prompt()
        .user("北京今天天气怎么样？")
        .tools(new WeatherTools())
        .call()
        .content();
```

### 3.1.2 默认注册（全局）

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultTools(new WeatherTools(), new OrderTools())
        .build();
```

### 3.1.3 按名称注册与版本差异

```java
.tools("getWeather", "queryOrders")   // 需配合工具 Bean 的解析机制
```

> 不同版本对按名称解析的支持有差异（2.0 移除了 `toolNames()` / `SpringBeanToolCallbackResolver`，改为显式 `ToolCallback` Bean）。生产代码建议**显式传入工具实例或 `ToolCallback`**，避免依赖隐式解析。

## 3.2 注册范围与优先级

### 3.2.1 覆盖规则

单次调用的 `.tools(...)` 与全局 `defaultTools(...)` 是**合并**关系，而非覆盖；如需精确控制，用 `ToolCallback` 显式管理。

### 3.2.2 工具去重

同名工具重复注册可能导致 Schema 冲突，务必保证工具名全局唯一。

# 4. ToolCallback：底层抽象

结论先行：`@Tool` 只是语法糖，底层统一抽象为 `ToolCallback`，理解它才能做动态注册与细粒度控制。

## 4.1 ToolCallback 家族

### 4.1.1 类型对比

| 类型 | 说明 |
|------|------|
| `FunctionToolCallback` | 由 `Function`/方法引用构建 |
| `MethodToolCallback` | 由 `@Tool` 方法构建（注解方式的底层） |
| `ToolCallbackProvider` | 批量提供工具 |

### 4.1.2 手工构建 ToolCallback

```java
ToolCallback callback = FunctionToolCallback.builder("getWeather", weatherService::getWeather)
        .description("查询指定城市的天气")
        .inputType(String.class)
        .build();

String answer = chatClient.prompt()
        .user("上海天气如何？")
        .toolCallbacks(callback)
        .call()
        .content();
```

## 4.2 ToolCallbackProvider 批量提供

### 4.2.1 实现示例

```java
@Bean
ToolCallbackProvider weatherTools(WeatherTools tools) {
    return MethodToolCallbackProvider.builder()
            .toolObjects(tools)
            .build();
}
```

### 4.2.2 与 @Tool 的等价关系

`MethodToolCallbackProvider` 会把对象中所有 `@Tool` 方法扫描成 `MethodToolCallback`，效果等同于直接 `defaultTools(...)`，但提供了以 Bean 形式集中管理工具的能力。

# 5. 工具返回结果的处理

结论先行：工具永远不要抛异常给模型，而是返回结构化的成功/失败信息，让模型能自行纠错。

## 5.1 结构化返回原则

### 5.1.1 成功/失败统一结构

```java
public record ToolResult(boolean success, String data, String error) {

    @Tool(description = "查询库存")
    ToolResult queryStock(@ToolParam(description = "商品编码") String sku) {
        try {
            int stock = stockService.get(sku);
            return new ToolResult(true, String.valueOf(stock), null);
        } catch (Exception e) {
            return new ToolResult(false, null, "查询失败：" + e.getMessage());
        }
    }
}
```

### 5.1.2 错误信息写法

错误信息应包含：**发生了什么、可能原因、可执行的下一步**。避免堆栈或内部实现细节泄露。

## 5.2 返回值序列化

### 5.2.1 JSON 序列化与体积

工具返回值会被序列化后写入 `ToolResponseMessage`，体积过大将挤占上下文窗口，甚至挤掉关键历史。

### 5.2.2 返回内容截断

```java
private String truncate(String s, int max) {
    return s.length() <= max ? s : s.substring(0, max) + "...(truncated)";
}
```

## 5.3 大结果集与分页

### 5.3.1 分页协议

结论先行：一次返回全部结果会撑爆上下文，工具应支持分页并让模型按需取页。

```java
public record Page<T>(List<T> items, int page, int size, long total) {}

@Tool(description = "分页查询订单，默认每页 20 条")
Page<Order> listOrders(
        @ToolParam(description = "页码，从 1 开始") int page,
        @ToolParam(description = "每页条数，最大 100", required = false) Integer size) {
    return orderService.page(page, size == null ? 20 : Math.min(size, 100));
}
```

# 6. 多工具与工具选择

结论先行：工具越多，提示越长、选择越难，误用概率越高；单次挂载建议不超过 10~15 个。

## 6.1 工具过多的代价

- 提示膨胀（每个工具的描述都占用 Token）；
- 模型选择困难、误用概率上升；
- Schema 冲突与参数串味。

## 6.2 选择与路由策略

### 6.2.1 策略对比

| 策略 | 说明 |
|------|------|
| 按场景分组 | 订单场景只挂订单相关工具 |
| 工具检索 | 先用一次调用筛出相关工具，再挂载 |
| 描述精炼 | 描述说清"何时用、何时不用" |
| 数量控制 | 单次挂载建议不超过 10~15 个 |

### 6.2.2 工具检索

```
用户问题
   │
   ▼
[元工具] 在工具库中检索 top-k 相关工具
   │
   ▼
动态挂载命中工具 -> 正常工具调用
```

### 6.2.3 描述精炼

描述应回答三个问题：**做什么、何时用、何时不用**。反例："查询订单"；正例："查询某用户的订单列表，仅当用户提供了用户 ID 时使用；不要用于查询物流"。

> 详见 `7_ToolsAndProtocols/4_tool_selection_routing.md`。

# 7. 工具执行与安全

结论先行：工具是"模型能触发的副作用入口"，必须默认按不可信输入对待。

## 7.1 副作用工具必须幂等

### 7.1.1 幂等键模式

```java
@Tool(description = "创建工单")
CreateResult createTicket(
        @ToolParam(description = "幂等键，唯一标识本次请求") String idempotencyKey,
        @ToolParam(description = "问题描述") String description) {
    return ticketService.createIdempotent(idempotencyKey, description);
}
```

### 7.1.2 重试与去重

> 工具可能被重试/重复调用，**写操作必须幂等**。详见 `14_ProductionEngineering/Reliability/3_idempotency.md`。

## 7.2 超时与资源限制

```java
@Tool(description = "调用外部 API")
String callExternalApi(...) {
    return withTimeout(Duration.ofSeconds(5),
            () -> externalClient.call(...));
}
```

## 7.3 权限校验

### 7.3.1 工具级鉴权

```java
@Tool(description = "删除用户数据")
void deleteUserData(@ToolParam(description = "用户ID") String userId) {
    SecurityUtils.requireRole("ADMIN");        // 工具级鉴权
    if (!SecurityUtils.canAccess(userId)) {
        throw new AccessDeniedException("无权操作该用户");
    }
    userService.delete(userId);
}
```

### 7.3.2 参数注入防护

模型生成的参数同样可能被提示注入（Prompt Injection）操纵，所有参数都要做白名单校验与转义。

> 详见 `15_Security/8_tool_permission_control.md`。

## 7.4 输入校验与脱敏

结论先行：模型给的参数是不可信输入，必须像对待外部请求一样校验。

### 7.4.1 白名单校验

```java
@Tool(description = "按状态查询订单")
List<Order> queryByStatus(
        @ToolParam(description = "状态，可选 PAID/CANCELLED") String status) {
    if (!Set.of("PAID", "CANCELLED").contains(status)) {
        return List.of();   // 或返回结构化错误
    }
    return orderService.queryByStatus(status);
}
```

### 7.4.2 防注入

- 不把参数直接拼进 SQL / Shell / URL；
- 对文件路径做规范化与目录白名单；
- 记录参数原文以便审计，但日志中要脱敏。

# 8. 多轮工具调用

结论先行：Spring AI 自动完成"调用-回填-再调用"循环，但必须设置最大轮数防死循环。

## 8.1 自动循环机制

### 8.1.1 循环流程

```java
String answer = chatClient.prompt()
        .user("帮我查一下订单 A123 的物流状态")
        .tools(new OrderTools(), new LogisticsTools())
        .call()
        .content();
// 内部：queryOrder("A123") → 得到运单号 → queryLogistics(运单号) → 汇总回答
```

### 8.1.2 并行工具调用

当模型一次返回多个 `toolCall` 时，框架可并行执行以降低延迟；此时工具必须线程安全。

## 8.2 终止条件与上限

### 8.2.1 最大轮数

> 注意设置**最大调用轮数**，防止无限循环。Spring AI 2.0 支持可配置的工具调用上限（`ToolCallingAdvisor`）。

### 8.2.2 循环检测

可记录最近若干次调用的 (工具名, 参数) 指纹，出现重复即中断并回退到兜底回答。

# 9. 工具调用日志与可观测

结论先行：工具调用是 Agent 可观测性的重点，调用次数、耗时、成功率、参数正确性都应纳入 Trace。

## 9.1 日志接入

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(new SimpleLoggerAdvisor())
        .defaultTools(new WeatherTools())
        .build();
```

## 9.2 指标与 Trace

### 9.2.1 关键指标

| 指标 | 含义 |
|------|------|
| `tool.calls` | 调用次数 |
| `tool.latency` | 执行耗时 |
| `tool.errors` | 失败次数 |
| `tool.rounds` | 每请求轮数 |

### 9.2.2 Span 设计

每次工具执行建议单独起一个 Span，父 Span 为模型调用，便于定位"是模型选错还是工具执行慢"。

> 详见 `13_Observability/3_agent_trace_and_spans.md`。

# 10. 常见坑与排查

结论先行：绝大多数"工具不生效"都源于描述不清或 Schema 不明确。

## 10.1 现象-原因-解决表

| 现象 | 原因 | 解决 |
|------|------|------|
| 模型不调用工具 | 描述不清 / 工具与问题不相关 | 优化 `description`，写清适用场景 |
| 参数类型错误 | Schema 不明确 | 用具体类型 + `@ToolParam` 说明 |
| 工具结果被忽略 | 返回内容过长/不结构化 | 返回精简结构化结果 |
| 无限循环调用 | 缺少终止条件 | 设置最大轮数、改进提示 |
| 写操作重复执行 | 重试 + 非幂等 | 幂等键 + 去重 |

## 10.2 排查清单

- 打印实际发送给模型的工具 Schema；
- 确认工具名唯一且描述可读；
- 检查工具是否真的被注册（`defaultTools` 是否生效）；
- 打开 `SimpleLoggerAdvisor` 观察完整往返。

# 11. 生产最佳实践

结论先行：工具设计要像设计 API 一样严肃——命名、描述、错误、权限、幂等一个都不能少。

## 11.1 设计清单

| 维度 | 检查项 |
|------|--------|
| 命名 | 动词开头、语义唯一 |
| 描述 | 何时用/何时不用 |
| 返回 | 结构化、精简、可纠错 |
| 安全 | 鉴权、参数校验、超时 |
| 幂等 | 写操作带幂等键 |

## 11.2 配置模板

```yaml
spring:
  ai:
    tools:
      # 部分版本提供的全局开关，具体键名以所用版本为准
      observation:
        enabled: true
```

# 12. 小结

## 12.1 要点回顾

- `@Tool` 把 Java 方法变成模型可调用的能力。
- 工具**描述质量**决定调用准确率。
- 工具返回值要**结构化**，错误也要结构化。
- 写操作必须**幂等 + 鉴权 + 超时**。
- 工具调用是 Agent 从"问答"走向"行动"的关键一步。

## 12.2 延伸阅读

- 工具选择与路由：`7_ToolsAndProtocols/4_tool_selection_routing.md`
- 工具权限控制：`15_Security/8_tool_permission_control.md`
- 幂等设计：`14_ProductionEngineering/Reliability/3_idempotency.md`
- Agent Trace：`13_Observability/3_agent_trace_and_spans.md`
