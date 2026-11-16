JSON 本身只有语法约束，没有字段类型和业务规则约束。Schema 用来回答"这份 JSON 是否结构正确、字段是否必填、取值范围是否合法"，是前后端契约和配置校验的重要工具。

# 1. 为什么需要 Schema

## 1.1 弱类型的代价

下面这份 JSON 语法完全合法：

```json
{
  "id": "42",
  "email": 123,
  "age": -1
}
```

但它可能有三个问题：

- `id` 应该是数字，却传了字符串。
- `email` 应该是字符串，却传了数字。
- `age` 应该是正整数，却是负数。

仅靠 `JSON.parse` 只能发现语法错误，无法发现结构和业务错误。

## 1.2 JSON Schema 的定位

JSON Schema 是一份**描述 JSON 结构的 JSON**：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "id": {"type": "integer"},
    "name": {"type": "string"}
  },
  "required": ["id", "name"]
}
```

它可以用在：

- 接口请求/响应校验。
- 配置文件校验。
- 数据管道输入校验。
- 生成文档和代码。

★ **关键认知**：JSON Schema 校验的是**结构和约束**，不是业务规则的全部。例如"用户名不能重复"需要查数据库，不适合放进 schema。

# 2. JSON Schema 基础

## 2.1 type 与 properties

```json
{
  "type": "object",
  "properties": {
    "id": {"type": "integer"},
    "name": {"type": "string"},
    "active": {"type": "boolean"},
    "profile": {
      "type": "object",
      "properties": {
        "age": {"type": "integer", "minimum": 0}
      }
    }
  }
}
```

常见 `type`：

| 类型 | 说明 |
|------|------|
| `object` | 对象 |
| `array` | 数组 |
| `string` | 字符串 |
| `number` | 任意数字 |
| `integer` | 整数 |
| `boolean` | 布尔 |
| `null` | 空值 |

## 2.2 required

`required` 声明必填字段：

```json
{
  "type": "object",
  "properties": {
    "name": {"type": "string"},
    "email": {"type": ["string", "null"]}
  },
  "required": ["name"]
}
```

区别：

- 字段在 `required` 中：必须出现。
- 类型包含 `null`：可以出现，值可以是 `null`。
- 不在 `required` 中：可以缺失。

⚠ JSON Schema 的 `default` 只是文档性说明，**不会自动把默认值写入数据**。需要默认值时要自己补。

## 2.3 数组与嵌套

```json
{
  "type": "object",
  "properties": {
    "tags": {
      "type": "array",
      "items": {"type": "string"},
      "minItems": 1,
      "maxItems": 10,
      "uniqueItems": true
    }
  }
}
```

常用数组关键字：

| 关键字 | 作用 |
|--------|------|
| `items` | 每个元素的 schema |
| `minItems` / `maxItems` | 元素数量限制 |
| `uniqueItems` | 是否要求元素唯一 |
| `contains` | 是否至少包含一个满足条件的元素 |

# 3. 校验实践

## 3.1 Java 校验

使用 `networknt/json-schema-validator`：

```java
JsonSchemaFactory factory = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V202012);
JsonSchema schema = factory.getSchema("""
    {
      "type": "object",
      "properties": {
        "name": {"type": "string", "minLength": 2},
        "age": {"type": "integer", "minimum": 1}
      },
      "required": ["name", "age"]
    }
    """);

JsonNode input = new ObjectMapper().readTree("""
    {"name": "a", "age": 0}
    """);

Set<ValidationMessage> errors = schema.validate(input);
for (ValidationMessage error : errors) {
    System.out.println(error.getInstanceLocation() + " " + error.getMessage());
}
```

输出可以定位到具体字段，例如 `/name` 和 `/age`。对外返回前应转换成项目统一的错误结构，不要直接暴露校验器内部文本。

## 3.2 前后端契约

同一份 schema 可以用于：

```text
前端表单校验
    │
    ▼
接口请求校验
    │
    ▼
服务端响应校验
    │
    ▼
测试用例数据校验
```

实践建议：

- 把 schema 作为契约文件纳入版本控制。
- 请求和响应分别维护 schema。
- 响应也校验，避免服务端返回结构漂移。
- 代码生成工具可以从 schema 生成类型，但生成代码仍需人工审查。

# 4. 版本兼容与演进

| 变更 | 是否兼容 | 说明 |
|------|----------|------|
| 新增可选字段 | 通常兼容 | 旧客户端应忽略未知字段 |
| 新增必填字段 | 不兼容 | 旧客户端不会提供 |
| 删除字段 | 可能不兼容 | 依赖该字段的客户端会失败 |
| 扩大枚举 | 通常兼容 | 旧客户端可能不认识新值 |
| 缩小枚举 | 不兼容 | 旧值可能被拒绝 |
| 放宽约束 | 通常兼容 | 例如把 `minLength` 调小 |
| 收紧约束 | 不兼容 | 旧数据可能不合法 |

★ **关键认知**：兼容性要从"旧客户端遇到新服务端"和"新客户端遇到旧服务端"两个方向看，只考虑一边容易出问题。

# 5. 常见错误

| 错误 | 后果 | 建议 |
|------|------|------|
| schema 过严 | 新增字段导致旧客户端被拒 | 对未知字段保持宽容 |
| 只校验请求不校验响应 | 服务端 bug 难发现 | 响应也校验 |
| 把 `default` 当自动填充 | 数据仍然缺失 | 显式补默认值 |
| 用 schema 做数据库唯一性校验 | 无法表达 | 放到业务层 |
| 错误信息直接返回 | 暴露内部结构 | 转换为统一错误 |
| schema 与代码不同步 | 契约和实现漂移 | 纳入 CI 或代码生成 |

# 6. 小结

| 关键字 | 作用 |
|--------|------|
| `type` | 限定值类型 |
| `properties` | 定义对象字段 |
| `required` | 必填字段 |
| `items` | 数组元素 schema |
| `enum` | 枚举值 |
| `pattern` | 正则约束 |
| `minimum` / `maximum` | 数值范围 |
| `minLength` / `maxLength` | 字符串长度 |

下一篇进入 JSONPath，讲如何在复杂 JSON 中按路径和条件查询数据。
