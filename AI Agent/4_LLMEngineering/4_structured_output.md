# 1. 为什么需要结构化输出

自然语言输出对人类友好，对程序不友好。结构化输出（Structured Output）指让大语言模型（LLM, Large Language Model）以固定、可解析的格式返回结果，通常是 JSON（JavaScript Object Notation），从而能被强类型语言直接反序列化。

## 1.1 自然语言与结构化数据的鸿沟

结论先行：自然语言是"给人看的"，结构化数据是"给程序用的"，二者之间缺少稳定契约。

### 1.1.1 解析不可靠的根因

```
自然语言："张三 28 岁，后端工程师" → 难以可靠解析
结构化：  {"name":"张三","age":28,"job":"后端工程师"} → 可直接消费
```

- 同义表达多：`28 岁`、`年龄 28`、`二十八岁` 混用。
- 分隔符不固定：逗号、顿号、换行随机出现。
- 数字形态多样：中文数字、带单位、带区间。
- 缺少字段名，正则（Regular Expression）极易漏匹配或误匹配。

### 1.1.2 Agent 链路中的下游依赖

在 Agent（智能体）链路中，一个节点的输出常常是下一个节点的输入：

```
LLM 输出 → 反序列化 → 业务对象 → 工具调用 / 数据库 / 决策分支
```

一旦反序列化失败，整条链路中断。因此结构化输出是 **Agent 可靠性的前提**。

## 1.2 结构化输出的收益与代价

### 1.2.1 收益

| 维度 | 收益 |
|------|------|
| 可消费 | 直接映射为 Java 对象 |
| 可校验 | 字段类型、范围、格式可校验 |
| 可观测 | 支持字段级日志与埋点 |
| 可测试 | 稳定契约便于单元测试 |

### 1.2.2 代价

- 约束越强，模型可发挥空间越小，可能影响语义质量。
- 依赖模型与框架支持，存在厂商差异。
- 复杂 Schema（模式）会额外消耗 Token（词元）。

## 1.3 本文范围

本文聚焦 LLM 工程视角：三种实现方式、Java 落地、校验、容错与生产治理。协议层细节见 `7_ToolsAndProtocols` 相关笔记。

# 2. 三种实现方式总览

结论先行：可靠性从弱到强依次为提示约束、JSON 模式、JSON Schema，工程上优先选后者。

## 2.1 能力光谱

### 2.1.1 提示约束（Prompt Constraint）

只靠提示词要求模型输出 JSON，无任何解码层保证，可靠性最低。

### 2.1.2 JSON 模式（JSON Mode）

厂商在解码层保证输出是合法 JSON，但不保证字段结构符合预期。

### 2.1.3 JSON Schema（结构化输出）

厂商在解码层按 Schema 约束，字段、类型、必填项都被强制满足，可靠性最高。

## 2.2 横向对比

| 方式 | 可靠性 | 语法保证 | 结构保证 | 说明 |
|------|:------:|:--------:|:--------:|------|
| 提示约束 | 低 | 否 | 否 | 靠提示要求 JSON |
| JSON 模式 | 中 | 是 | 否 | 保证输出是合法 JSON |
| JSON Schema | 高 | 是 | 是 | 强约束结构（推荐） |

## 2.3 选型决策

```
需要严格字段契约？ ── 是 ──> 模型支持 Schema？ ── 是 ──> JSON Schema
        │                              └── 否 ──> JSON 模式 + 校验
        └── 否 ──> 提示约束 + 清理 + 校验
```

经验值：在信息抽取任务上，三种方式端到端成功率大致为提示约束 70%~85%、JSON 模式 90%~97%、JSON Schema 99% 以上。

# 3. 提示约束（最弱）

结论先行：提示约束适合原型验证，不适合生产关键链路。

## 3.1 原理与示例

```java
String prompt = """
        提取信息，只输出 JSON，不要任何其他文字：
        {"name":"","age":0,"job":""}

        文本：张三 28 岁，后端工程师
        """;
```

模型会模仿示例模板（few-shot，少样本）输出，但仅此而已。

## 3.2 失效模式

### 3.2.1 额外解释文字

模型可能在 JSON 前后添加"好的，以下是提取结果："等自然语言。

### 3.2.2 代码块包裹

输出常被 ```` ```json ```` 围栏包裹，直接反序列化会失败。

### 3.2.3 格式漂移

字段名改写（`name` → `userName`）、类型漂移（数字变字符串）时有发生。

## 3.3 生产建议

- 必须配合响应清理与校验使用。
- 仅用于探索期或对失败不敏感的场景。
- 关键链路一律升级为 Schema。

# 4. JSON 模式

结论先行：JSON 模式解决"是不是合法 JSON"，不解决"字段对不对"。

## 4.1 原理

厂商在采样阶段屏蔽非法 JSON 的 token 组合，从而保证输出可被标准 JSON 解析器解析。

其本质是解码期的约束采样（Constrained Sampling）：维护一个状态机，根据已生成前缀过滤候选 token，只有能构成合法 JSON 的 token 才被允许。注意它只关心语法，不关心语义，因此 `{"name":1}` 这类"语法合法但类型不对"的输出依然会出现。

## 4.2 Spring AI 实现

```java
ChatOptions options = ChatOptions.builder()
        .responseFormat(ResponseFormat.JSON)     // 保证合法 JSON
        .build();

String json = chatClient.prompt()
        .user("提取：张三 28 岁，后端工程师")
        .options(options)
        .call()
        .content();
```

## 4.3 边界与坑

### 4.3.1 字段不符合预期

合法 JSON 不等于正确结构，例如 `{"n":"张三","a":"28"}` 也是合法 JSON。

### 4.3.2 截断与空对象

达到 `maxTokens`（最大输出词元）时可能截断，或直接返回 `{}` 空对象。

## 4.4 生产建议

- 始终叠加 Bean Validation（Java 校验规范）。
- 提高 `maxTokens` 防止截断。
- 解析失败进入重试或降级流程。

# 5. JSON Schema（最强）

结论先行：JSON Schema 是生产环境的默认选择，语法与结构双重保证。

## 5.1 原理

### 5.1.1 受约束解码

模型解码时，每一步只在满足 Schema 的候选 token 中选择，从数学上保证输出合规。相比 JSON 模式，它把状态机扩展到了字段名、字段类型、必填项与枚举取值，因此能同时保证语法与结构。

### 5.1.2 Schema 子集限制

多数厂商只支持 JSON Schema 子集：`type`、`properties`、`required`、`enum`、`additionalProperties` 等；`oneOf`、`$ref` 等支持程度不一。

## 5.2 完整 Java 示例

### 5.2.1 手写 Schema

```java
Map<String, Object> schema = Map.of(
        "type", "object",
        "properties", Map.of(
                "name", Map.of("type", "string"),
                "age", Map.of("type", "integer"),
                "job", Map.of("type", "string")),
        "required", List.of("name", "age", "job"),
        "additionalProperties", false);

OpenAiChatOptions options = OpenAiChatOptions.builder()
        .responseFormat(new ResponseFormat(ResponseFormat.Type.JSON_SCHEMA, schema))
        .build();
```

### 5.2.2 由 record 生成 Schema

```java
// 伪代码：框架可由 Java 类型反射生成 Schema
public record Person(String name, int age, String job) {}

OpenAiChatOptions options = OpenAiChatOptions.builder()
        .responseFormat(ResponseFormat.Type.JSON_SCHEMA)
        .outputSchema(Person.class)     // 具体 API 以框架版本为准
        .build();
```

## 5.3 厂商支持差异

| 厂商 | 支持程度 | 备注 |
|------|:--------:|------|
| OpenAI | 高 | 原生 Structured Outputs |
| Anthropic | 中 | 依赖工具调用间接实现 |
| 开源模型 | 视推理框架 | 需 grammar / 约束解码 |

## 5.4 边界与坑

- Schema 过于复杂会被降级或忽略。
- 部分模型首次请求需预热 Schema，产生额外延迟。
- 字段名大小写、下划线风格需与 Java 映射保持一致。

## 5.5 生产建议

- 用 `additionalProperties:false` 收敛输出。
- 把 Schema 纳入版本管理，变更走评审。
- 保留 JSON 模式作为不支持 Schema 时的回退。

# 6. 映射为 Java 对象

结论先行：优先用框架的自动映射，复杂场景再手写转换。

## 6.1 Spring AI 的 entity()

### 6.1.1 基本用法

```java
public record Person(String name, int age, String job) {}

Person person = chatClient.prompt()
        .user("提取：张三 28 岁，后端工程师")
        .call()
        .entity(Person.class);
```

### 6.1.2 自定义转换器

```java
Person person = chatClient.prompt()
        .user("提取：张三 28 岁，后端工程师")
        .call()
        .entity(Person.class, spec -> spec
                .converter(new CustomJacksonConverter()));
```

## 6.2 LangChain4j 的 AiServices

```java
interface Extractor {
    Person extract(String text);
}

Person person = extractor.extract("张三 28 岁，后端工程师");
```

AiServices（AI 服务）会把返回类型自动转为结构化描述并注入提示。

```java
// 若需要自定义 JSON 映射，可在 builder 上替换 ObjectMapper
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(new WeatherTools())
        .build();
```

LangChain4j 会在调用前把 `Person` 的字段结构写入系统提示，并在返回后自动反序列化，失败则抛出异常供上层兜底。

## 6.3 手写 Jackson 映射

```java
ObjectMapper mapper = new ObjectMapper()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

Person person = mapper.readValue(cleanJson(raw), Person.class);
```

`FAIL_ON_UNKNOWN_PROPERTIES=false` 让模型多返回的字段被忽略而非报错；若字段缺失，则需配合 `@JsonInclude` 或默认值处理。

## 6.4 生产建议

- 关闭未知字段报错，提升兼容性。
- 明确时间、金额格式（ISO-8601、`BigDecimal`）。
- 映射失败统一走容错流程。

# 7. 复杂结构建模

结论先行：用 record（记录类）表达嵌套与列表，用 enum（枚举）收敛取值。

## 7.1 嵌套对象与列表

```java
public record Order(
        String orderId,
        Customer customer,
        List<Item> items,
        BigDecimal total,
        OrderStatus status
) {
    public record Customer(String name, String phone) {}
    public record Item(String sku, int quantity, BigDecimal price) {}
    public enum OrderStatus { PENDING, PAID, SHIPPED }
}
```

## 7.2 枚举与类型化字段

| 类型 | 说明 |
|------|------|
| 嵌套对象 | 用 record 嵌套 |
| 列表 | `List<T>` |
| 枚举 | 显著提升稳定性 |
| 数字/布尔 | 用具体类型而非字符串 |

## 7.3 递归与可选字段

```java
public record Node(String label, List<Node> children) {}      // 递归结构
public record Profile(String name, Optional<String> email) {} // 可选字段
```

递归与可选字段在部分厂商 Schema 中支持有限，需实测。

处理建议：递归结构可限制最大深度并在提示中说明；可选字段在 Schema 中用非必填表达，Java 侧用 `Optional` 或可空引用承接。

## 7.4 生产建议

- 枚举值只增不改，兼容旧数据。
- 可选字段用 `Optional` 或显式默认值。
- 深层嵌套控制在 3 层以内。

# 8. 校验与约束

结论先行：模型输出"格式对"不代表"内容对"，必须叠加业务校验。

## 8.1 Bean Validation

```java
public record UserProfile(
        @NotBlank String name,
        @Min(0) @Max(150) int age,
        @Email String email) {}

UserProfile profile = extractor.extract(text);

Set<ConstraintViolation<UserProfile>> violations =
        validator.validate(profile);
if (!violations.isEmpty()) {
    // 重试或兜底
}
```

## 8.2 语义校验

```java
public void validateSemantics(Order order) {
    BigDecimal sum = order.items().stream()
            .map(i -> i.price().multiply(BigDecimal.valueOf(i.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    if (sum.compareTo(order.total()) != 0) {
        throw new IllegalStateException("明细合计与总额不一致");
    }
}
```

## 8.3 校验失败的闭环

结论先行：校验不是为了丢弃结果，而是为了驱动一次有针对性的重试。

```
校验失败 → 拼接错误信息 → 追加到提示重试 → 仍失败 → 降级/人工
```

## 8.4 生产建议

- 校验规则与 Schema 保持一致，避免双重口径。
- 错误信息要回传给模型用于自我修正。
- 统计各类校验失败率，驱动提示与 Schema 优化。

# 9. 解析失败的容错

结论先行：任何结构化输出都必须有兜底，否则一次失败即整链失败。

## 9.1 重试策略

### 9.1.1 简单重试

```java
public <T> T safeEntity(String prompt, Class<T> type, T fallback) {
    for (int i = 0; i < MAX_RETRIES; i++) {
        try {
            return chatClient.prompt()
                    .user(prompt)
                    .call()
                    .entity(type);
        } catch (Exception e) {
            log.warn("结构化输出解析失败，第 {} 次重试", i + 1, e);
        }
    }
    return fallback;
}
```

### 9.1.2 带修复的重试

```java
String repairPrompt = """
        上一次输出不符合要求，错误：%s
        请只输出合法 JSON：%s
        """.formatted(error, originalPrompt);
```

## 9.2 降级与兜底

| 场景 | 兜底 |
|------|------|
| 解析失败 | 返回默认对象 |
| 字段缺失 | 填充默认值 |
| 语义非法 | 走人工审核队列 |

## 9.3 失败样本采集

```java
failureStore.record(new FailureSample(prompt, raw, error, Instant.now()));
```

## 9.4 生产建议

- 重试次数有限且带退避（backoff）。
- 兜底对象要能区分"默认值"与"真实值"。
- 失败样本定期回归，形成提示与 Schema 的迭代闭环。

# 10. 响应清理

结论先行：清理是最后一道防线，不能替代 Schema。

## 10.1 常见污染形态

| 形态 | 示例 |
|------|------|
| 代码块包裹 | 三反引号 json 包裹 |
| 前后解释 | `好的，结果如下：{...}` |
| 尾随逗号 | `{"a":1,}` |
| 单引号 | `{'a':1}` |
| 注释 | `{"a":1 /* 备注 */}` |
| 中文标点 | `｛"a"：1｝` |

其中尾随逗号、单引号、注释在 JSON5 中合法，在标准 JSON 中非法；清理时可用宽松解析器（如 Jackson 的 `JsonReadFeature`）容错。

## 10.2 清理实现

```java
public String cleanJson(String raw) {
    String cleaned = raw.trim();

    // 移除 Markdown 代码块
    if (cleaned.startsWith("```")) {
        cleaned = cleaned.replaceFirst("^```(json)?\\s*", "")
                .replaceFirst("\\s*```$", "");
    }
    // 移除前后解释文字（取第一个 { 到最后一个 }）
    int start = cleaned.indexOf('{');
    int end = cleaned.lastIndexOf('}');
    if (start >= 0 && end > start) {
        cleaned = cleaned.substring(start, end + 1);
    }
    return cleaned;
}
```

## 10.3 边界与坑

- 取第一个 `{` 到最后一个 `}` 在数组根节点（`[...]`）时会失效。
- 字符串内部含 `}` 时可能误截。
- 清理后仍需标准解析器校验，不可跳过。

## 10.4 生产建议

- 清理逻辑要覆盖根为数组的情况。
- 记录清理前后的差异，监控污染率。
- 污染率上升说明提示或模型需要调整。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 靠提示约束格式 | 用 Schema |
| 字段缺失 | 默认值 + 校验 |
| JSON 被截断 | 提高 maxTokens |
| 类型不符 | 用具体类型 |
| 解析失败无兜底 | 重试 + fallback |
| 无失败记录 | 记录失败样本 |

## 11.2 可观测性

```java
meterRegistry.counter("structured.parse.fail",
        "model", model, "type", type.getSimpleName()).increment();
```

建议暴露的核心指标：

| 指标 | 含义 |
|------|------|
| `structured.parse.success` | 解析成功计数 |
| `structured.parse.fail` | 解析失败计数 |
| `structured.validation.fail` | 校验失败计数 |
| `structured.retry.count` | 重试次数分布 |

## 11.3 灰度与回归

- 提示与 Schema 变更先在灰度租户验证。
- 维护结构化输出回归集，量化成功率。
- 成功率下降自动回滚。

# 12. 小结

## 12.1 要点回顾

- 结构化输出是 Agent 可靠性的前提。
- 三种方式：提示约束（弱）、JSON 模式（中）、Schema（强）。
- 优先用原生 JSON Schema，其次框架的 `entity()`。
- 枚举、类型化字段能显著提升稳定性。
- 必须有解析失败的兜底。

## 12.2 落地清单

- 关键链路使用 JSON Schema。
- 叠加 Bean Validation 与语义校验。
- 重试 + fallback + 失败样本采集。
- 清理逻辑覆盖数组根节点。
- 监控解析成功率并纳入告警。
- Schema 与提示纳入版本管理与灰度。
- 兜底策略可区分默认值与真实值。
