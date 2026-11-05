本板块整理开发中最常见的结构化数据与文档格式：JSON、YAML、XML、Markdown。目标不是罗列语法，而是让初学者能看懂、写对、校验并在 Java、Go、前端等实际项目里正确选型。

# 1. 模块定位

## 1.1 本模块解决什么问题

本模块围绕五类问题展开：

- **语法**：怎么写才是合法的。
- **解析**：不同语言如何读取和生成。
- **校验**：如何保证结构和取值正确。
- **安全**：不可信输入可能带来哪些风险。
- **选型**：什么场景该用哪种格式。

## 1.2 与其他板块的关系

| 板块 | 关联 |
|------|------|
| Java | 接口序列化、配置文件、XML 解析 |
| Go | `encoding/json`、YAML 配置、结构体标签 |
| Frontend | JSON 请求响应、前端配置、Markdown 文档 |
| AI Agent | 提示词与工具调用的 JSON 结构 |

# 2. 学习目标

## 2.1 知识目标

- 掌握 JSON、YAML、XML、Markdown 的基本语法和数据类型。
- 理解解析、序列化、校验和查询的基本流程。
- 了解常见安全风险，例如精度丢失、原型污染、XXE。
- 能根据场景选择合适的格式。

## 2.2 能力目标

- 能独立读写和校验一份格式文件。
- 能定位格式错误并解释原因。
- 能在 Java、Go、前端中调用对应的解析库。
- 能设计稳定的接口数据结构。

# 3. 文件清单与大纲

## 3.1 Json

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_syntax.md](Json/1_overview_and_syntax.md) | 概览与语法 | 六种值类型、合法/非法示例、常见语法错误 |
| [2_parsing_and_serialization.md](Json/2_parsing_and_serialization.md) | 解析与序列化 | JS/Python/Java/Go 四种语言的读写与精度问题 |
| [3_schema_and_validation.md](Json/3_schema_and_validation.md) | Schema 与校验 | JSON Schema 关键字、Java 校验、契约演进 |
| [4_jsonpath_and_query.md](Json/4_jsonpath_and_query.md) | JSONPath | 路径、通配符、过滤表达式与跨语言差异 |
| [5_security_and_pitfalls.md](Json/5_security_and_pitfalls.md) | 安全与坑 | 深度嵌套、精度丢失、原型污染、反序列化 |
| [6_json_vs_yaml_xml.md](Json/6_json_vs_yaml_xml.md) | 格式对比 | JSON/YAML/XML 的定位、语法和选型 |
| [7_best_practices_and_summary.md](Json/7_best_practices_and_summary.md) | 设计实践 | 命名、分页、兼容性、错误结构和性能 |

## 3.2 Yaml

规划中，建议篇目：语法与缩进、类型与坑、锚点与合并、多文档与 schema、K8s/CI 示例、与 JSON 对比。

## 3.3 XML

规划中，建议篇目：语法与命名空间、DTD/XSD、DOM/SAX/StAX、XPath/XSLT、JAXB、XXE 安全。

## 3.4 Markdown

规划中，建议篇目：基础语法、GFM 扩展、表格代码图片、数学与图表、文档站工具链、规范与 lint。

# 4. 推荐阅读路径

```text
初学者          后端开发者              文档编写者
  │                 │                       │
  ▼                 ▼                       ▼
Json 1 → 2       Json 1 → 2              Markdown
  │                 │                       │
  ▼                 ▼                       ▼
Json 3 → 5       Json 3 → 7              XML/YAML
  │                 │
  ▼                 ▼
Json 6           格式对比与选型
```

- **快速入门**：JSON 1 → 2 → 6，先能读写和选型。
- **接口开发**：JSON 1 → 2 → 3 → 7，重点掌握契约和设计。
- **安全排查**：JSON 5 → XML 安全篇，理解不可信输入的风险。

# 5. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 本节最核心的机制或结论 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均标注语言，示例尽量可直接运行 |
| 命令清单 | 可照抄执行，注意替换路径和域名 |

# 6. 小结

| 主题 | 完成标准 |
|------|----------|
| 语法 | 能写出合法格式并解释错误 |
| 解析 | 能在至少两种语言中读写 |
| 校验 | 能用 schema 或校验器约束结构 |
| 安全 | 能识别常见输入风险并防御 |
| 选型 | 能根据场景说明为什么选该格式 |

建议先完成 JSON 模块，再按实际项目需要学习 YAML、XML 和 Markdown。
