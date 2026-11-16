解析和序列化是 JSON 最日常的操作，但不同语言在数字精度、字段映射、未知字段处理上的行为并不一致。本篇用 JavaScript、Python、Java、Go 四种语言的示例建立统一认知，并补充流式处理和性能注意事项。

# 1. 解析与序列化

## 1.1 从文本到对象

解析（parse）是把 JSON 文本转换成内存中的对象或数据结构：

```text
JSON 文本
  │
  ▼
词法分析：识别字符串、数字、标点
  │
  ▼
语法分析：构建对象、数组和值
  │
  ▼
内存对象：Map、struct、dict、Object
```

解析失败时，错误通常包含位置信息，例如行号、列号或字符偏移。排查时先看报错位置附近的字符，常见原因是逗号、引号或括号不匹配。

## 1.2 序列化与反序列化

两个方向：

| 术语 | 方向 | 说明 |
|------|------|------|
| 序列化（serialize） | 对象 → JSON 文本 | 用于发送、存储 |
| 反序列化（deserialize） | JSON 文本 → 对象 | 用于接收、读取 |

字段映射是重点：不同语言和库对命名风格、未知字段、空值、日期的默认行为不同。

# 2. 各语言示例

## 2.1 JavaScript

```javascript
const text = '{"id":42,"name":"alice","tags":["go","java"]}';

try {
    const user = JSON.parse(text);
    console.log(user.name);
} catch (err) {
    console.error("JSON 解析失败", err.message);
}

const json = JSON.stringify(
    { id: 42, name: "alice" },
    null, // replacer，可过滤字段
    2     // 缩进空格数
);
```

`reviver` 可以在解析时转换值：

```javascript
const user = JSON.parse(text, (key, value) => {
    if (key === "createdAt") {
        return new Date(value); // 把字符串转换成 Date
    }
    return value;
});
```

⚠ `JSON.stringify` 会忽略 `undefined`、函数和 Symbol 属性；循环引用会抛错。

## 2.2 Python

```python
import json

text = '{"id":42,"name":"alice"}'

try:
    user = json.loads(text)
    print(user["name"])
except json.JSONDecodeError as err:
    print("解析失败", err.msg, err.pos)

json_text = json.dumps(
    {"id": 42, "name": "张三"},
    ensure_ascii=False,  # 保留中文，不转成 \uXXXX
    indent=2
)
```

`object_hook` 可以在解析时转换对象：

```python
def convert(d):
    if "created_at" in d:
        d["created_at"] = datetime.fromisoformat(d["created_at"])
    return d

user = json.loads(text, object_hook=convert)
```

⚠ Python 的 `json` 默认把整数解析为 `int`，超出范围的大整数也能处理；但很多其他语言会丢精度。

## 2.3 Java

使用 Jackson：

```java
ObjectMapper mapper = new ObjectMapper()
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

String text = "{\"id\":42,\"name\":\"alice\"}";

User user = mapper.readValue(text, User.class);
String json = mapper.writeValueAsString(user);
```

```java
public class User {
    private long id;
    private String name;

    @JsonIgnoreProperties(ignoreUnknown = true)
    // getter/setter 省略
}
```

常用配置：

| 配置 | 作用 |
|------|------|
| `FAIL_ON_UNKNOWN_PROPERTIES` | 遇到未知字段是否报错 |
| `JavaTimeModule` | 支持 `LocalDate`、`Instant` |
| `WRITE_DATES_AS_TIMESTAMPS` | 日期是否输出为时间戳 |
| `WRITE_NULLS` / `NON_NULL` | 是否输出 null 字段 |

⚠ 日期格式要统一约定。默认时间戳和 ISO-8601 字符串不同，前后端不一致时会出现解析失败或时区偏差。

## 2.4 Go

```go
type User struct {
    ID   int64    `json:"id"`
    Name string   `json:"name"`
    Tags []string `json:"tags,omitempty"`
}

var user User
if err := json.Unmarshal([]byte(text), &user); err != nil {
    log.Fatal(err)
}

data, err := json.Marshal(user)
if err != nil {
    log.Fatal(err)
}
```

`omitempty` 会在字段为零值时省略输出。注意：

- 对数字字段，`omitempty` 会把 `0` 也省略。
- `int64` 大整数在 `interface{}` 中可能被解析为 `float64`，导致精度丢失。
- 需要保留大整数精度时使用 `json.Decoder.UseNumber`：

```go
decoder := json.NewDecoder(strings.NewReader(text))
decoder.UseNumber()

var value map[string]any
if err := decoder.Decode(&value); err != nil {
    log.Fatal(err)
}
```

# 3. 流式处理

大 JSON 不能一次性读入内存，否则可能 OOM。流式解析逐个读取 token 或事件：

```text
读取字节流
  │
  ▼
逐个解析 token
  │
  ▼
处理当前对象，释放已处理数据
```

| 语言 | 流式方式 |
|------|----------|
| Java | Jackson `JsonParser`、Gson `JsonReader` |
| Go | `json.Decoder` |
| Python | `ijson` 等第三方库 |
| JavaScript | 没有标准流式解析器，可用第三方库或按行处理 NDJSON |

NDJSON（每行一个 JSON 对象）比单个超大 JSON 数组更适合流式处理。

# 4. 性能与内存

- 复用 `ObjectMapper`、`JsonSerializer` 等重量级对象，不要每次请求都创建。
- 避免重复序列化同一对象。
- 大响应只输出必要字段，减少体积。
- 对高频接口考虑缓存序列化结果。
- 不要在日志里序列化超大对象，既慢又可能泄露数据。

# 5. 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| 大整数精度丢失 | 被解析为双精度浮点 | 用字符串或 `json.Number` |
| 字段名不匹配 | 命名风格不一致 | 使用 `@JsonProperty`、`json` 标签 |
| 未知字段报错 | 默认严格模式 | 根据契约决定是否忽略 |
| 日期格式不一致 | 时间戳 vs ISO 字符串 | 统一格式和时区 |
| 空值处理不一致 | `null`、缺失、空字符串混用 | 明确约定 |
| 循环引用 | 对象互相引用 | 使用 ID 引用或自定义序列化 |

# 6. 动手练习

用 JavaScript 和 Python 处理同一份数据：

```json
{
  "id": 9007199254740993,
  "name": "alice",
  "createdAt": "2026-01-01T10:00:00Z",
  "tags": ["go", "java"]
}
```

要求：

1. 分别解析并打印 `id`，观察 JavaScript 是否发生精度变化。
2. 把 `createdAt` 转成日期对象。
3. 序列化回 JSON，比较字段顺序、中文编码和日期格式。

# 7. 小结

| 语言 | 解析 | 序列化 | 注意点 |
|------|------|--------|--------|
| JavaScript | `JSON.parse` | `JSON.stringify` | 大整数精度、循环引用 |
| Python | `json.loads` | `json.dumps` | `ensure_ascii`、`object_hook` |
| Java | Jackson `readValue` | `writeValueAsString` | 未知字段、日期格式 |
| Go | `json.Unmarshal` | `json.Marshal` | 标签、`omitempty`、大整数 |

下一篇进入 JSON Schema，讲如何校验结构、必填字段、类型和取值范围。
