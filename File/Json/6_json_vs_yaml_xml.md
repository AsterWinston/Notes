JSON、YAML、XML 都能表达结构化数据，但设计目标和适用场景不同。本篇从可读性、注释、数据类型和工具生态四个维度对比三种格式，并给出 API 传输、配置文件和文档场景下的选型建议。

# 1. 三种格式的定位

| 格式 | 设计初衷 | 主流用途 |
|------|----------|----------|
| JSON | 轻量的数据交换格式 | Web API、日志、消息 |
| YAML | 人类可读的配置语言 | K8s、CI/CD、应用配置 |
| XML | 可扩展的标记语言 | 文档、SOAP、复杂 schema |

一句话记忆：

- JSON 适合**机器之间传数据**。
- YAML 适合**人写配置**。
- XML 适合**描述文档和复杂结构**。

# 2. 语法对比

## 2.1 可读性

同一份用户数据：

```json
{
  "user": {
    "name": "alice",
    "age": 18,
    "tags": ["admin", "user"]
  }
}
```

```yaml
user:
  name: alice
  age: 18
  tags:
    - admin
    - user
```

```xml
<user>
  <name>alice</name>
  <age>18</age>
  <tags>
    <tag>admin</tag>
    <tag>user</tag>
  </tags>
</user>
```

对比：

| 格式 | 优点 | 缺点 |
|------|------|------|
| JSON | 结构明确、解析简单 | 括号和引号多，不能注释 |
| YAML | 最简洁、支持注释 | 缩进敏感，隐式类型有坑 |
| XML | 表达能力强、命名空间 | 冗长，解析成本高 |

## 2.2 注释与扩展

| 格式 | 注释 | 扩展能力 |
|------|------|----------|
| JSON | 不支持 | 依靠字段扩展 |
| YAML | 支持 `#` | 锚点、别名、标签 |
| XML | 支持 `<!-- -->` | 命名空间、DTD、XSD、属性 |

YAML 锚点示例：

```yaml
defaults: &defaults
  timeout: 3s
  retries: 2

serviceA:
  <<: *defaults
  url: https://a.example.com

serviceB:
  <<: *defaults
  url: https://b.example.com
```

XML 命名空间示例：

```xml
<order xmlns="https://example.com/order"
       xmlns:user="https://example.com/user">
  <user:id>42</user:id>
</order>
```

## 2.3 数据类型

| 类型 | JSON | YAML | XML |
|------|------|------|-----|
| 字符串 | 双引号 | 可裸写 | 文本节点 |
| 数字 | 一种 number | 整数/浮点/科学计数 | 文本，需 schema 解释 |
| 布尔 | `true`/`false` | `true`/`false`，但 `yes/no/on/off` 有版本差异 | 文本，需 schema |
| null | `null` | `null`/`~` | 空元素或 `xsi:nil` |
| 日期 | 无原生类型 | 可被解析为日期 | 需 schema |
| 注释 | 不支持 | 支持 | 支持 |

⚠ YAML 的隐式类型是常见坑。例如在某些 YAML 1.1 解析器中，`NO` 会被解析为布尔 `false`，`1.0` 可能变成数字。配置中涉及版本号、国家代码时建议加引号。

# 3. 使用场景对比

## 3.1 API 传输

JSON 是 Web API 的主流格式，原因：

- 浏览器原生支持。
- 解析库成熟。
- 体积比 XML 小。
- 与 JavaScript 数据结构接近。

```http
Content-Type: application/json

{"id":42,"name":"alice"}
```

## 3.2 配置文件

YAML 在配置领域流行，因为：

- 层级直观，适合表达嵌套配置。
- 支持注释，方便说明。
- K8s、GitHub Actions、Docker Compose 等生态广泛使用。

但要注意：

- 缩进必须一致，Tab 和空格混用会报错。
- 字符串不加引号时可能被隐式转换。
- 深层嵌套仍然难读。

## 3.3 文档与消息

XML 的优势：

- 命名空间适合多标准混合。
- XSD 可以表达复杂约束。
- XSLT 可以转换文档。
- SOAP、Office 文档、SVG 等场景仍大量使用。

如果只是传简单数据，XML 通常过于冗长。

# 4. 转换与互操作

三种格式可以互转，但会丢失信息：

| 信息 | 转换后 |
|------|--------|
| 注释 | 可能丢失 |
| 命名空间 | JSON/YAML 无原生对应 |
| 属性 vs 子元素 | 需要约定映射规则 |
| 日期类型 | 可能变成字符串 |
| 锚点和别名 | 展开后丢失 |
| 重复键 | 行为不一致 |

转换工具：

```bash
# YAML -> JSON
yq -o=json config.yaml

# JSON -> YAML
yq -P config.json

# XML -> JSON（需要约定映射规则）
xq . data.xml
```

# 5. 选型建议

| 场景 | 推荐 | 理由 |
|------|------|------|
| Web API | JSON | 浏览器和语言支持最好 |
| 应用配置 | YAML | 可读、可注释、生态成熟 |
| K8s/CI | YAML | 平台标准 |
| 文档/SOAP | XML | 命名空间和 schema 能力 |
| 日志 | JSON | 易解析、易检索 |
| 简单键值配置 | Properties/TOML | 比 YAML 更少歧义 |

选型原则：

- 优先跟随生态标准，不要为了个人偏好逆生态。
- 需要注释和可读性时选 YAML，但要接受缩进和隐式类型风险。
- 需要严格结构和命名空间时选 XML。
- 机器传输优先 JSON。

# 6. 小结

| 格式 | 优势 | 主要坑 |
|------|------|--------|
| JSON | 简单、通用、解析快 | 无注释、大数精度 |
| YAML | 可读、可注释、配置友好 | 缩进敏感、隐式类型 |
| XML | 命名空间、schema、文档能力强 | 冗长、解析复杂 |

下一篇总结 JSON 的接口设计实践，讲命名、嵌套、分页、兼容性和错误结构。
