# 1. 为什么需要结构化输出

## 1.1 自然语言的不可消费性

**结论先行**：自然语言输出无法直接被程序消费，业务系统需要的是**可反序列化的对象**。把"人类可读"当作"机器可读"，是所有解析事故的根源。

```
自然语言："张三今年 28 岁，是后端工程师。"
        ↓  不可靠（解析困难、格式漂移）
结构化：  { "name": "张三", "age": 28, "job": "后端工程师" }
```

## 1.2 结构化输出的价值

### 1.2.1 Agent 可靠性

**机制**：Agent 的每一步——工具选择、参数填充、决策结果——都需要被程序读取。只有结构化，才能把模型输出接进确定性的业务逻辑。

| 场景 | 自然语言的问题 | 结构化后的收益 |
|------|----------------|----------------|
| 工具调用 | 参数解析失败 | 参数类型安全 |
| 分类 | 标签大小写/同义漂移 | 枚举收敛 |
| 抽取 | 字段缺失/错位 | 可校验、可重试 |
| 路由 | 判定不可解析 | 稳定分支 |

### 1.2.2 类型安全

结构化输出把"字符串约定"升级为"编译期契约"：字段名、类型、可空性都在 Java 类型上表达，重构与测试都更可靠。

## 1.3 整体机制

**结论先行**：Spring AI 通过 `BeanOutputConverter` 把模型输出映射为 Java 对象，底层生成 JSON Schema 注入提示，再解析响应。

```
Java 类型 ──► BeanOutputConverter ──► JSON Schema
                                        │
                              追加到提示文本
                                        │
                                    模型输出
                                        │
                              convert() 解析为对象
                                        │
                                    Java 对象
```

# 2. 基础用法：entity()

## 2.1 定义实体

### 2.1.1 record

**结论先行**：优先用 `record` 定义输出类型——不可变、字段语义清晰，且序列化框架对其支持良好。

```java
public record ActorFilms(String actor, List<String> movies) {}
```

### 2.1.2 字段命名

**注意**：字段名会成为 JSON Schema 的键名。若要与外部契约对齐（如 `snake_case`），需用序列化注解显式映射，否则模型会按 Java 字段名输出。

## 2.2 直接获取对象

```java
ActorFilms result = chatClient.prompt()
        .user("列出成龙参演的三部电影")
        .call()
        .entity(ActorFilms.class);

System.out.println(result.actor());   // 成龙
System.out.println(result.movies());  // [...]
```

## 2.3 内部三步机制

Spring AI 会自动：

1. 由 `ActorFilms` 生成 JSON Schema；
2. 把 Schema 追加到提示，要求模型按格式输出；
3. 解析模型返回的 JSON 并反序列化为对象。

## 2.4 边界与坑

| 现象 | 原因 | 建议 |
|------|------|------|
| 解析异常 | 输出含额外文字 | 用原生 Schema 或清洗 |
| 字段为 null | 模型未提供 | 设默认值 + 重试 |
| 类型不匹配 | 数字/字符串混用 | 宽松反序列化 |

# 3. 集合与泛型

## 3.1 ParameterizedTypeReference

**结论先行**：泛型集合因类型擦除无法用 `Class` 表达，必须用 `ParameterizedTypeReference` 保留类型信息。

```java
List<ActorFilms> list = chatClient.prompt()
        .user("列出三位动作演员及其代表作")
        .call()
        .entity(new ParameterizedTypeReference<List<ActorFilms>>() {});
```

## 3.2 嵌套泛型

**机制**：`ParameterizedTypeReference` 通过匿名子类捕获泛型参数，因此可以表达 `Map<String, List<ActorFilms>>` 这类复杂结构。

```java
Map<String, List<ActorFilms>> byGenre = chatClient.prompt()
        .user("按类型分组列出演员及代表作")
        .call()
        .entity(new ParameterizedTypeReference<Map<String, List<ActorFilms>>>() {});
```

## 3.3 坑

**边界**：泛型层级越深，Schema 越复杂，模型出错概率越高。生产上应优先"扁平化 + 多次调用"，而非一次生成深嵌套结构。

# 4. 手动使用 BeanOutputConverter

## 4.1 生成格式说明

**结论先行**：需要更细粒度控制时（例如自己组装 `Prompt`），可直接使用 `BeanOutputConverter`。

```java
BeanOutputConverter<ActorFilms> converter =
        new BeanOutputConverter<>(ActorFilms.class);

String format = converter.getFormat();   // 生成格式说明（含 JSON Schema）
```

`converter.getFormat()` 会给出类似：

```text
Your response should be in JSON format.
Do not include any explanations...
The JSON object must follow this schema:
{ ... JSON Schema ... }
```

## 4.2 解析

```java
Prompt prompt = new Prompt("""
        列出成龙参演的三部电影。
        %s
        """.formatted(format));

ChatResponse response = chatModel.call(prompt);
ActorFilms films = converter.convert(
        response.getResult().getOutput().getText());
```

## 4.3 与 ChatClient 的关系

| 维度 | `entity()` | `BeanOutputConverter` |
|------|-----------|------------------------|
| 抽象层级 | 高 | 低 |
| 是否自动追加 Schema | 是 | 否（手动） |
| 适用 | 日常调用 | 自定义 Prompt、批处理 |

**机制**：`.entity()` 内部正是使用 `BeanOutputConverter` 完成 Schema 生成与解析，两者是同一机制的不同封装层级。

# 5. 同时获取对象与元数据

## 5.1 responseEntity

**结论先行**：既需要对象、又需要 Token 等元数据时，用 `.responseEntity(Class)`。

```java
ResponseEntity<ChatResponse, ActorFilms> re = chatClient.prompt()
        .user("列出成龙的三部电影")
        .call()
        .responseEntity(ActorFilms.class);

ActorFilms entity = re.entity();
Usage usage = re.response().getMetadata().getUsage();
```

## 5.2 用途

| 需求 | 方法 |
|------|------|
| 只要对象 | `entity()` |
| 只要元数据 | `chatResponse()` |
| 两者都要 | `responseEntity()` |

## 5.3 坑

**注意**：`ResponseEntity` 的两部分命名易混淆——`entity()` 是业务对象，`response()` 才是底层 `ChatResponse`。记错会拿到错误的类型。

# 6. 复杂类型

## 6.1 嵌套对象

**结论先行**：嵌套对象可以表达，但要控制深度。

```java
public record Order(
        String orderId,
        Customer customer,
        List<Item> items,
        BigDecimal total
) {
    public record Customer(String name, String phone) {}
    public record Item(String sku, int quantity) {}
}
```

使用方式与普通类型一致：

```java
Order order = chatClient.prompt()
        .user("解析订单：订单号 A1001，客户王五 13800000000，"
                + "商品 SKU-1 两件、SKU-2 一件，合计 199.00 元")
        .call()
        .entity(Order.class);
```

**边界与坑**：嵌套越深，模型遗漏字段的概率越高。经验上层级不超过三层；超过时应拆成多次调用，先取主对象再按 ID 补齐子对象。

## 6.2 枚举

```java
public enum Sentiment { POSITIVE, NEUTRAL, NEGATIVE }

public record Analysis(String text, Sentiment sentiment, double confidence) {}
```

> 枚举能显著提升分类类任务的稳定性。

**机制**：枚举会被转换为 JSON Schema 的 `enum` 约束，模型只能从有限集合中选择，天然抑制"同义漂移"。

```java
Analysis analysis = chatClient.prompt()
        .user("判断情感：这家餐厅味道不错，就是有点贵。")
        .call()
        .entity(Analysis.class);
```

**边界与坑**：若模型返回 `"positive"`（小写）而枚举为 `POSITIVE`，反序列化可能失败。可在枚举上用 Jackson 的 `@JsonCreator` 做大小写归一，或统一约定输出大写。

## 6.3 带校验

### 6.3.1 Bean Validation

结合 Jakarta Bean Validation：

```java
public record UserProfile(
        @NotBlank String name,
        @Min(0) @Max(150) int age,
        @Email String email
) {}
```

```java
UserProfile profile = chatClient.prompt()
        .user("提取：我叫李四，30 岁，邮箱 lisi@example.com")
        .call()
        .entity(UserProfile.class);

validator.validate(profile);   // 失败则重试或兜底
```

### 6.3.2 校验失败处理

**生产建议**：校验失败不要静默接受，应触发一次带"纠错提示"的重试，仍失败则走兜底。

```java
Set<ConstraintViolation<UserProfile>> errors = validator.validate(profile);
if (!errors.isEmpty()) {
    log.warn("结构化输出校验失败: {}", errors);
    // 触发重试或返回兜底
}
```

# 7. 使用 JSON Schema 约束（原生结构化输出）

## 7.1 原生 Structured Outputs

**结论先行**：部分模型（OpenAI 等）支持原生 Structured Outputs，可用 JSON Schema 在模型侧强约束，比"提示里塞 Schema"更可靠。

```java
// 概念示例：通过 ChatOptions 指定 responseFormat 与 schema
OpenAiChatOptions options = OpenAiChatOptions.builder()
        .model("gpt-4o-mini")
        .responseFormat(new ResponseFormat(ResponseFormat.Type.JSON_SCHEMA, schema))
        .build();
```

其中 `schema` 是一段标准 JSON Schema 字符串（示意）：

```text
{
  "type": "object",
  "properties": {
    "actor":  { "type": "string" },
    "movies": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["actor", "movies"],
  "additionalProperties": false
}
```

**注意**：`additionalProperties: false` 与 `required` 是原生结构化输出的关键约束，它们能杜绝"多字段、缺字段"两类最常见的问题。

## 7.2 Spring AI 支持方式

**机制**：原生模式下，Schema 由模型 API 强制遵守，而非仅作为提示文本。这消除了"格式漂移"这一主要失败来源。

| 方式 | 约束位置 | 可靠性 |
|------|----------|--------|
| 提示塞 Schema | 模型自觉 | 中 |
| 原生 JSON Schema | 模型 API 强制 | 高 |
| 工具调用参数 | 模型 API 强制 | 高 |

## 7.3 兼容性与版本

**边界与坑**：原生结构化输出需模型支持，且不同厂商能力不一。Spring AI 2.0 对结构化输出有进一步重构，使用前请核对版本文档。不支持时需回退到提示约束。

# 8. 解析失败的容错

## 8.1 重试

**结论先行**：模型输出可能因截断、额外文字、格式漂移导致解析失败，生产必须兜底。

```java
public <T> T safeEntity(String question, Class<T> type, T fallback) {
    for (int i = 0; i < 3; i++) {
        try {
            return chatClient.prompt().user(question).call().entity(type);
        } catch (Exception e) {
            log.warn("结构化输出解析失败，第 {} 次重试", i + 1, e);
        }
    }
    return fallback;
}
```

封装为可复用组件：

```java
@Component
public class StructuredInvoker {

    private final ChatClient chatClient;
    private static final Logger log = LoggerFactory.getLogger(StructuredInvoker.class);

    public StructuredInvoker(ChatClient.Builder builder) {
        this.chatClient = builder
                .defaultOptions(ChatOptions.builder().temperature(0.1).build())
                .build();
    }

    public <T> Optional<T> invoke(String question, Class<T> type) {
        for (int i = 0; i < 3; i++) {
            try {
                return Optional.ofNullable(
                        chatClient.prompt().user(question).call().entity(type));
            } catch (Exception e) {
                log.warn("结构化输出解析失败，第 {} 次重试", i + 1, e);
            }
        }
        return Optional.empty();
    }
}
```

## 8.2 清洗输出

**机制**：常见脏数据是 Markdown 代码块包裹。解析前先剥离围栏：

```java
String cleaned = raw.trim();
if (cleaned.startsWith("```")) {
    cleaned = cleaned.replaceFirst("^```(json)?", "")
                     .replaceFirst("```$", "")
                     .trim();
}
```

## 8.3 兜底策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 重试 | 同一提示再试 | 偶发失败 |
| 修复提示重试 | 附上错误信息 | 结构性失败 |
| 换模型 | 升级到更强模型 | 能力不足 |
| 返回默认值 | 保证主流程不崩 | 非关键字段 |

## 8.4 配套手段

| 手段 | 说明 |
|------|------|
| 提高 `maxTokens` | 防止 JSON 被截断 |
| 降低 `temperature` | 减少格式漂移 |
| 显式要求"只输出 JSON" | 减少解释性文字 |
| 使用原生 JSON Schema | 模型侧强约束 |
| 解析失败重试 | 最终兜底 |
| 记录失败样本 | 用于优化提示与评测 |

# 9. 常见坑

## 9.1 解析类

| 现象 | 原因 | 解决 |
|------|------|------|
| `JsonParseException` | 模型输出了额外文字/代码块 | 清理 ```` ```json ```` 包裹，或用原生 Schema |
| 字段缺失 | 模型"偷懒" | 字段设默认值 + 重试 |

## 9.2 类型类

| 现象 | 原因 | 解决 |
|------|------|------|
| 数字变字符串 | 类型不匹配 | 使用宽松反序列化或重试 |
| 枚举值越界 | 模型造词 | 用原生 Schema 限定 enum |

## 9.3 长度与编码

| 现象 | 原因 | 解决 |
|------|------|------|
| 中文乱码 | 编码问题 | 统一 UTF-8 |
| 大对象被截断 | 超出 maxTokens | 分次生成或提高上限 |

# 10. 生产建议

## 10.1 契约管理

**结论先行**：把"结构定义"当作**契约**，变更需评估对下游的影响。

- 结构定义要**稳定、扁平、可校验**，避免过深层级。
- 字段新增宜向后兼容；重命名与删除属破坏性变更，需版本化。
- 为关键结构维护示例与 Schema 快照。

## 10.2 观测与评测

- 记录解析失败率、重试率、校验失败率。
- 保留失败样本，用于优化提示与回归测试。
- 结构化输出是 Agent 可靠性的前提：工具参数、决策结果都应结构化。

## 10.3 性能

| 关注点 | 建议 |
|--------|------|
| 延迟 | 结构越复杂越慢，优先扁平 |
| 成本 | Schema 会占用输入 Token |
| 稳定性 | 原生 Schema > 提示约束 |
| 兜底 | 永远准备 fallback，不要假设模型 100% 合规 |

# 11. 小结

## 11.1 核心要点

- `.entity(Class)` 是从"文本"到"对象"的最短路径。
- `BeanOutputConverter` 负责 Schema 生成与解析，是底层机制。
- 枚举 + 校验 + 重试 = 生产级结构化输出三件套。

## 11.2 落地检查表

- 能用原生 JSON Schema 就别靠提示硬约束。
- 结构即契约，变更要有版本与回归。
- 永远准备兜底，别假设模型 100% 合规。
