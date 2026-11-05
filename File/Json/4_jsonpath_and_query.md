当 JSON 嵌套层级变深、需要按条件取字段时，逐层遍历会变得冗长。JSONPath 提供类似 XPath 的查询语法，本篇从根节点、数组下标讲到过滤表达式，并给出 Java、Python、JavaScript 中的使用示例。

# 1. 为什么需要查询语法

面对这样的数据：

```json
{
  "store": {
    "books": [
      {"title": "Go 入门", "price": 59, "stock": 3},
      {"title": "Java 进阶", "price": 89, "stock": 0},
      {"title": "JSON 实战", "price": 39, "stock": 8}
    ]
  }
}
```

要取"所有有库存的书名"，手写遍历需要多层循环和判空：

```javascript
const titles = [];
for (const book of data.store.books) {
    if (book.stock > 0) {
        titles.push(book.title);
    }
}
```

JSONPath 可以写成：

```text
$.store.books[?(@.stock > 0)].title
```

★ **关键认知**：JSONPath 适合"从已知结构中取数"，不适合做复杂业务计算。它只负责查询，不负责连接、聚合、事务和类型约束。

# 2. JSONPath 基础

## 2.1 根节点与子节点

| 表达式 | 含义 |
|--------|------|
| `$` | 根节点 |
| `$.store` | 根下的 `store` 字段 |
| `$.store.books` | 继续访问 `books` |
| `$['store']` | 方括号写法，适合字段名含特殊字符 |

```text
$.store.books[0].title
```

## 2.2 数组与通配符

| 表达式 | 含义 |
|--------|------|
| `$.books[0]` | 第一个元素 |
| `$.books[-1]` | 最后一个元素（部分实现支持） |
| `$.books[*]` | 所有元素 |
| `$..title` | 递归查找所有 `title` |
| `$.store.*` | `store` 下所有字段 |

```text
$.store.books[*].title
```

返回结果通常是数组，即使只有一个匹配项。不同库对"单值还是数组"的处理不一致，使用前要确认。

## 2.3 过滤表达式

```text
$.store.books[?(@.price < 50)].title
$.store.books[?(@.stock > 0 && @.price < 100)].title
```

- `?()` 表示过滤。
- `@` 表示当前元素。
- 支持比较运算和逻辑运算，但不同库的语法支持程度不同。

⚠ JSONPath 没有统一的正式标准，Jayway、jsonpath-ng、JavaScript 的 jsonpath 库在过滤器语法、负索引、递归下降等细节上可能不同。跨语言使用时要用测试用例锁定行为。

# 3. 各语言使用

## 3.1 Java

使用 Jayway JsonPath：

```java
String json = """
    {
      "store": {
        "books": [
          {"title": "Go 入门", "price": 59, "stock": 3},
          {"title": "Java 进阶", "price": 89, "stock": 0}
        ]
      }
    }
    """;

DocumentContext context = JsonPath.parse(json);

List<String> titles = context.read(
    "$.store.books[?(@.stock > 0)].title"
);

System.out.println(titles);
```

如果确定只取一个值，可以用 `read` 指定类型：

```java
String firstTitle = context.read("$.store.books[0].title");
```

## 3.2 Python

使用 `jsonpath-ng`：

```python
from jsonpath_ng import parse
import json

data = json.loads(text)
expr = parse("$.store.books[?(@.stock > 0)].title")

titles = [match.value for match in expr.find(data)]
print(titles)
```

`jsonpath-ng` 的过滤语法与 Jayway 不完全一致，复杂表达式要先写单元测试验证。

## 3.3 JavaScript

使用 `jsonpath` 库：

```javascript
import { JSONPath } from "jsonpath-plus";

const titles = JSONPath({
    path: "$.store.books[?(@.stock > 0)].title",
    json: data
});
```

如果只是简单取值，原生 JavaScript 的可选链和数组方法更直观：

```javascript
const titles = data.store.books
    .filter(book => book.stock > 0)
    .map(book => book.title);
```

★ **关键认知**：不是所有查询都值得引入 JSONPath。简单结构用语言原生能力更易读；只有在路径复杂、需要配置化查询时，JSONPath 的优势才明显。

# 4. 与 SQL、GraphQL 的边界

| 能力 | JSONPath | SQL | GraphQL |
|------|----------|-----|---------|
| 查询 JSON 字段 | 是 | 部分数据库支持 | 是 |
| 连接多表 | 否 | 是 | 是 |
| 聚合 | 有限 | 是 | 有限 |
| 类型约束 | 否 | 是 | 是 |
| 写入 | 否 | 是 | Mutation |
| 适合场景 | 取数、断言 | 结构化查询 | API 按需取字段 |

JSONPath 不能替代数据库查询。它是"在已有 JSON 上找数据"的工具，不是数据存储和计算引擎。

# 5. 常见错误

| 错误 | 后果 | 建议 |
|------|------|------|
| 路径写错 | 返回空结果 | 先用简单路径逐级验证 |
| 假设返回单值 | 拿到数组或 null | 明确库的返回类型 |
| 过滤器语法不兼容 | 换语言后结果不同 | 用测试锁定行为 |
| 对缺失字段做运算 | 空指针或异常 | 先判空再计算 |
| 递归下降范围过大 | 性能差、结果过多 | 尽量使用明确路径 |
| 在循环里重复解析 | 性能低 | 解析一次，复用表达式 |

# 6. 小结

| 语法 | 作用 |
|------|------|
| `$` | 根节点 |
| `.` / `[]` | 访问子节点 |
| `[*]` | 数组所有元素 |
| `..` | 递归查找 |
| `[?()]` | 条件过滤 |
| `@` | 当前元素 |

JSONPath 适合取数和断言，不适合复杂聚合和业务计算。下一篇进入 JSON 安全，讲清深度嵌套、精度丢失、原型污染和反序列化风险。
