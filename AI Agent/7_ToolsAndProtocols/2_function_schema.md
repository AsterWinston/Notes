# 1. Schema 的作用

## 1.1 核心结论

模型通过 **JSON Schema**（JSON 模式）理解工具的：

- 参数名与类型
- 参数含义
- 哪些必填、哪些可选

一句话：Schema 是"机器可读的参数说明书"，模型据此生成符合格式的参数再调用。

## 1.2 模型如何使用 Schema

### 1.2.1 决策链路

```
模型 → 读取 Schema → 生成符合格式的参数 → 调用
```

在决策阶段，模型同时看到工具的 `name`、`description` 与 `parameters`，据此判断"是否调用、调用哪个、传什么参数"。

### 1.2.2 Schema 与描述的分工

| 位置 | 主要作用 |
|------|----------|
| 工具级 description | 说明工具用途与边界 |
| 参数级 description | 说明单个参数的格式/范围/示例 |
| type / enum / required | 约束参数的取值结构 |

## 1.3 边界：Schema 不是校验器

结论先行：Schema 只是给模型的**提示**，不是运行时校验。模型可能生成不符合 Schema 的参数（尤其是弱模型），服务端必须再校验一次。

# 2. 标准格式

## 2.1 结论与总览

结论先行：主流函数调用格式都遵循 OpenAI 的函数定义结构：`name` + `description` + `parameters`（JSON Schema 对象）。

## 2.2 完整 JSON 示例

```json
{
  "name": "queryOrders",
  "description": "按日期范围查询订单",
  "parameters": {
    "type": "object",
    "properties": {
      "startDate": {
        "type": "string",
        "description": "开始日期，格式 yyyy-MM-dd"
      },
      "endDate": {
        "type": "string",
        "description": "结束日期，格式 yyyy-MM-dd"
      },
      "status": {
        "type": "string",
        "enum": ["PAID", "CANCELLED", "REFUNDED"],
        "description": "订单状态，可选"
      }
    },
    "required": ["startDate", "endDate"]
  }
}
```

### 2.2.1 必填与可选

`required` 数组决定哪些参数必填；未列入的参数默认可选。可选参数应在描述里说明"缺省行为"，否则模型不知道省略后的语义。

### 2.2.2 常用约束关键词

| 关键词 | 作用 | 示例 |
|--------|------|------|
| `enum` | 限定取值集合 | `["PAID","CANCELLED"]` |
| `minimum` / `maximum` | 数值范围 | `1` 到 `100` |
| `minLength` / `maxLength` | 字符串长度 | `1` 到 `64` |
| `pattern` | 正则约束 | `^[A-Z][0-9]{3}$` |
| `default` | 缺省值 | `"PAID"` |

## 2.3 字段说明

### 2.3.1 name 与 description

- `name`：函数名，必须与代码里的可调用对象一致，通常要求只含字母、数字、下划线。
- `description`：工具的用途与边界，是模型选择工具的主要依据。

### 2.3.2 parameters

- `type`：顶层固定为 `object`。
- `properties`：每个参数的名称与约束。
- `required`：必填参数名数组；不在其中即为可选。

## 2.4 常见格式错误

| 错误 | 后果 | 修正 |
|------|------|------|
| `required` 写了不存在的字段 | 校验失败 | 与 properties 对齐 |
| 缺少 `type: object` | 部分模型拒绝 | 补上顶层类型 |
| description 为空 | 模型乱填 | 每个参数都写描述 |
| enum 与描述不一致 | 模型困惑 | 保持一致 |

# 3. 参数类型

## 3.1 类型总览表

| JSON 类型 | 说明 | 示例 |
|-----------|------|------|
| `string` | 字符串 | 日期、ID |
| `integer` | 整数 | 数量、页码 |
| `number` | 小数 | 金额 |
| `boolean` | 布尔 | 是否 |
| `array` | 数组 | 标签列表 |
| `object` | 对象 | 嵌套结构 |
| `enum` | 枚举 | 状态 |

## 3.2 枚举显著提升准确性

```java
// 枚举显著提升准确性
public enum OrderStatus { PAID, CANCELLED, REFUNDED, SHIPPED }

@Tool("查询订单")
List<Order> queryOrders(
        @P("状态") OrderStatus status,     // 模型只能从中选
        @P("页码，从 1 开始") int page);
```

枚举把开放取值收敛为有限集合，能大幅降低模型生成非法值的概率。

## 3.3 类型反例

- 金额用 `string`：模型可能传"一百元"，无法解析。
- 布尔用 `string`：模型可能传"yes"/"是"/"1"，歧义大。
- 所有参数都用 `string`：看似简单，实则把解析负担推给了服务端。

## 3.4 Java 类型与 JSON 类型的映射

结论先行：框架按 Java 类型推断 JSON 类型，映射关系需牢记，否则会出现"以为是整数、实际是字符串"的偏差。

| Java 类型 | JSON 类型 | 备注 |
|-----------|-----------|------|
| `String` | `string` | 最通用，但语义弱 |
| `int` / `Integer` | `integer` | 页码、数量 |
| `long` / `Long` | `integer` | 注意前端精度 |
| `double` / `BigDecimal` | `number` | 金额建议用字符串或 `number` |
| `boolean` | `boolean` | 避免用字符串代替 |
| `List<T>` | `array` | 元素类型要明确 |
| `enum` | `string` + `enum` | 首选 |
| `record` / POJO | `object` | 嵌套结构 |

# 4. 在 Spring AI 中定义

## 4.1 结论与依赖

结论先行：Spring AI 通过 `@Tool` 与 `@ToolParam` 注解在方法上声明工具，框架用反射自动生成 JSON Schema。

## 4.2 完整示例

```java
@Component
public class OrderTools {

    private final OrderService orderService;

    public OrderTools(OrderService orderService) {
        this.orderService = orderService;
    }

    @Tool("按日期范围查询订单")
    public List<Order> queryOrders(
            @ToolParam(description = "开始日期，格式 yyyy-MM-dd")
            String startDate,

            @ToolParam(description = "结束日期，格式 yyyy-MM-dd")
            String endDate,

            @ToolParam(description = "订单状态，可选",
                    required = false)
            String status) {
        return orderService.query(startDate, endDate, status);
    }
}
```

## 4.3 Spring AI 的边界与坑

- `@Tool` 方法所在对象必须是 Spring Bean，否则不会被扫描到。
- 复杂返回类型会被序列化，注意控制字段数量。
- 注解 `description` 是描述的唯一来源，不要留空。
- 版本差异：不同 Spring AI 版本的注解包名与属性名可能变化，升级时需回归。

# 5. 在 LangChain4j 中定义

## 5.1 结论与依赖

结论先行：LangChain4j 使用 `@Tool` 与 `@P` 注解，风格与 Spring AI 接近，但注解类来自 `dev.langchain4j.agent.tool` 包。

## 5.2 完整示例

```java
public class OrderTools {

    private final OrderService orderService;

    public OrderTools(OrderService orderService) {
        this.orderService = orderService;
    }

    @Tool("按日期范围查询订单")
    public List<Order> queryOrders(
            @P("开始日期，格式 yyyy-MM-dd") String startDate,
            @P("结束日期，格式 yyyy-MM-dd") String endDate,
            @P(value = "订单状态，可选", required = false) String status) {
        return orderService.query(startDate, endDate, status);
    }
}
```

## 5.3 两框架对比

### 5.3.1 注解与 API 差异

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 工具注解 | `@Tool` | `@Tool` |
| 参数注解 | `@ToolParam` | `@P` |
| 注册方式 | Bean 扫描 / ToolCallback | `AiServices` / 工具列表 |
| 生态 | Spring 全家桶 | 更贴近 LLM 抽象 |

### 5.3.2 选型建议

- 已在使用 Spring Boot，优先 Spring AI。
- 需要更细粒度控制消息与工具执行，LangChain4j 更直接。
- 两者都支持手动构造 Schema，便于迁移与对照。

# 6. 复杂参数

## 6.1 嵌套对象

```java
public record Address(
        String province,
        String city,
        String detail) {}

@Tool("创建收货地址")
Address createAddress(@P("地址信息") Address address);
```

嵌套对象在强模型上表现良好，在弱模型上容易生成结构错误的 JSON。

## 6.2 数组

```java
@Tool("批量查询订单")
List<Order> batchQuery(@P("订单号列表，最多 20 个") List<String> orderIds);
```

数组要写清元素类型与数量上限，避免模型一次传上千个元素。

## 6.3 约束表达

```java
@Tool("分页查询")
List<Order> query(
        @P("页码，从 1 开始") int page,
        @P("每页条数，1~100") int size);
```

> 把约束写进描述（"1~100"），并在服务端校验。

## 6.4 复杂参数的反例与替代

结论先行：能用标量表达的就不要用嵌套对象，能用枚举的就不要用自由文本。

| 反例 | 问题 | 替代方案 |
|------|------|----------|
| `Map<String,Object> extra` | 结构不可预测 | 拆成具名字段 |
| `List<List<String>>` | 双层数组易错 | 扁平化为一维 |
| `Object data` | 无类型信息 | 具体 record |
| 递归结构 | 模型难以构造 | 限制深度或改树为路径 |

```java
// 反例：结构不可预测
@Tool("更新订单")
void update(@P("任意附加信息") Map<String, Object> extra);

// 正例：具名字段 + 枚举
@Tool("更新订单备注")
void updateNote(
        @P("订单号") String orderId,
        @P("备注内容，不超过 200 字") String note);
```

# 7. 描述的最佳实践

## 7.1 结论

结论先行：参数描述必须包含**格式、示例、范围、单位**四要素中的相关项，越具体模型越不容易填错。

## 7.2 示例

```java
// 反例：描述模糊
@ToolParam(description = "日期") String date;

// 正例：描述具体
@ToolParam(description = "开始日期，格式 yyyy-MM-dd，如 2026-01-01") String startDate;

// 正例：带示例
@ToolParam(description = "订单号，格式为字母+数字，如 A123、B456") String orderId;

// 正例：说明取值范围
@ToolParam(description = "数量，1~100 之间的整数") int quantity;
```

## 7.3 描述要素表

| 要素 | 说明 |
|------|------|
| 格式 | yyyy-MM-dd |
| 示例 | 如 A123 |
| 范围 | 1~100 |
| 单位 | 元、天 |

# 8. Schema 生成机制

## 8.1 反射生成

结论先行：框架在启动或首次注册时，通过反射读取方法签名与注解，生成 JSON Schema，无需手写。

```java
// 框架自动生成 Schema（反射）
ToolCallback callback = FunctionToolCallback.builder("queryOrders", orderService::query)
        .description("按日期范围查询订单")
        .inputType(QueryRequest.class)
        .build();
```

> 上述 `FunctionToolCallback.builder(...)` 的 API 在不同版本可能略有差异，此处为示意。

## 8.2 打印与调试

```java
// 查看生成的 Schema（调试用）
System.out.println(callback.getToolDefinition().inputSchema());
```

> 调试时打印 Schema，能发现"参数描述不清"等问题。

## 8.3 生成机制的边界

- 反射对泛型擦除敏感：`List<T>` 可能丢失元素类型，建议用具体类型。
- 默认值不会自动进入 Schema，需要在描述里说明缺省行为。
- 校验注解（如 `@Min`）不一定会被转换成 Schema 约束，需确认框架行为。

# 9. Schema 与模型能力

## 9.1 模型支持差异

| 模型 | Schema 支持 |
|------|-------------|
| OpenAI | 完整 JSON Schema |
| Anthropic | 完整 |
| Gemini | 完整 |
| 部分开源模型 | 支持有限，需简化 Schema |

## 9.2 简化策略

> 对 Schema 支持弱的模型，应**简化参数结构**（减少嵌套、用扁平参数）。

- 把嵌套对象拍平为多个标量参数。
- 用 `string` + enum 替代复杂对象。
- 减少 `anyOf` / `oneOf` 等联合类型。

### 9.2.1 扁平化前后对比

```java
// 扁平化前：嵌套对象
@Tool("创建地址")
void create(@P("地址") Address address);

// 扁平化后：标量参数
@Tool("创建地址")
void create(
        @P("省份") String province,
        @P("城市") String city,
        @P("详细地址") String detail);
```

## 9.3 兼容性矩阵

| 特性 | 强支持模型 | 弱支持模型 | 建议 |
|------|-----------|-----------|------|
| 深层嵌套 | 可用 | 易出错 | 扁平化 |
| 数组 of object | 可用 | 易出错 | 改为 JSON 字符串 |
| enum | 支持 | 多数支持 | 保留 |
| 联合类型 | 支持 | 支持差 | 避免 |

## 9.4 能力探测方法

结论先行：上线前用一个"最小 Schema"做冒烟测试，观察模型是否正确回传结构化参数。

- 用两个参数（一个 `string`、一个 `enum`）构造测试工具。
- 观察是否出现参数缺失、类型错误、JSON 截断。
- 逐步增加嵌套深度，记录模型开始出错的阈值。
- 把探测结果固化为该模型的"Schema 复杂度预算"。

# 10. 生产注意事项

## 10.1 风险表

| 风险 | 建议 |
|------|------|
| 参数过多 | ≤ 5 个 |
| 类型模糊 | 用具体类型与枚举 |
| 描述过简 | 写清格式/示例/范围 |
| 深层嵌套 | 尽量扁平 |
| 无服务端校验 | Schema 只是提示，必须服务端校验 |
| 不支持复杂 Schema | 按模型简化 |

## 10.2 服务端校验

结论先行：无论 Schema 多完善，服务端都要做一次强校验，包括类型、范围、权限与业务规则。

```java
public List<Order> query(String startDate, String endDate, String status) {
    LocalDate from = parseOrThrow(startDate, "startDate");
    LocalDate to = parseOrThrow(endDate, "endDate");
    if (from.isAfter(to)) {
        throw new IllegalArgumentException("开始日期不能晚于结束日期");
    }
    if (status != null && !VALID_STATUS.contains(status)) {
        throw new IllegalArgumentException("非法状态：" + status);
    }
    return orderRepository.query(from, to, status);
}
```

# 11. 小结

## 11.1 核心结论

- Schema 是模型理解工具参数的依据。
- 用具体类型、枚举、清晰描述。
- 描述要含格式、示例、范围。
- 调试时打印生成的 Schema。
- Schema 只是提示，服务端必须再校验。

## 11.2 落地清单

- 为每个参数写描述，包含四要素。
- 优先使用枚举收敛取值。
- 按模型能力决定是否扁平化。
- 上线前打印 Schema 做一次人工审查。
