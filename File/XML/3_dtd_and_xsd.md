良构只能保证 XML"语法没写错"，却保证不了"业务字段对不对"。元素顺序、出现次数、是否必填、价格能不能是负数，这些都需要 schema 来约束。本篇讲 DTD 和 XSD 两代约束语言，重点放在 XSD 的元素、属性、类型与校验流程上。

# 1. 为什么需要 schema

## 1.1 从良构到合法

下面这份 XML 完全良构，但 `price` 是 `abc`，业务上显然不合法：

```xml
<library>
  <book id="b1"><title>XML 入门</title><price>abc</price></book>
</library>
```

schema 负责表达"元素顺序、出现次数、数据类型、取值范围"这些语法层之外的规则：

```text
XML 文档 ── 解析器 ──→ 良构检查（语法）
                │
                └── schema 校验 ──→ 合法检查（结构 + 类型 + 约束）
```

## 1.2 DTD 与 XSD 的定位

| 维度 | DTD | XSD |
|------|-----|-----|
| 出身 | SGML 时代 | XML Schema 标准 |
| 语法 | 非 XML 语法 | XML 语法 |
| 数据类型 | 几乎没有 | 字符串、整数、日期、小数等 |
| 命名空间 | 不支持 | 原生支持 |
| 约束能力 | 顺序、次数、枚举 | 类型、范围、模式、断言 |
| 工具支持 | 老系统、部分解析器 | 主流校验器、代码生成 |

★ **关键认知**：DTD 约束的是"**结构**"，XSD 约束的是"**结构 + 类型**"。DTD 能说你必须有一个 `price` 元素，却很难说它必须是两位小数的非负数；XSD 可以。

# 2. DTD

## 2.1 内部 DTD 与外部 DTD

DTD 写在 `DOCTYPE` 声明里，可以直接内嵌：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE library [
  <!ELEMENT library (book+)>
  <!ELEMENT book (title, author, price)>
  <!ELEMENT title (#PCDATA)>
  <!ELEMENT author (#PCDATA)>
  <!ELEMENT price (#PCDATA)>
  <!ATTLIST book id ID #REQUIRED>
]>
<library><book id="b1"><title>XML 入门</title><author>张三</author><price>59.00</price></book></library>
```

也可以把规则放到外部文件：`<!DOCTYPE library SYSTEM "library.dtd">`。外部 DTD 便于复用，但解析时会多一次文件读取，也增加了外部实体风险。

## 2.2 元素声明

语法是 `<!ELEMENT 名字 内容模型>`，内容模型决定子元素怎么组合：

| 写法 | 含义 |
|------|------|
| `(#PCDATA)` | 只包含文本 |
| `EMPTY` | 空元素，不能有内容 |
| `(a, b)` | 先 `a` 后 `b`，顺序固定 |
| `(a | b)` | `a` 或 `b` 二选一 |
| `(a?)` / `(a*)` / `(a+)` | 零或一个 / 零或多个 / 一个或多个 |

★ **关键认知**：DTD 的 `,` 表示**顺序**，`|` 表示**选择**，`? * +` 表示**次数**。这三组符号能表达大部分结构，但表达不了"价格必须是小数"这类值约束。

## 2.3 属性声明

语法是 `<!ATTLIST 元素名 属性名 类型 默认值>`：

```dtd
<!ATTLIST book
  id ID #REQUIRED
  lang CDATA #IMPLIED
  version CDATA #FIXED "1.0"
  status (draft|published|archived) "draft">
```

| 部分 | 取值 | 含义 |
|------|------|------|
| 类型 | `CDATA` | 普通文本 |
| 类型 | `ID` | 文档内唯一标识 |
| 类型 | `IDREF`/`IDREFS` | 引用其他 ID |
| 类型 | `(a|b|c)` | 枚举 |
| 默认值 | `#REQUIRED` | 必须出现 |
| 默认值 | `#IMPLIED` | 可选 |
| 默认值 | `"v"` | 缺省值 |

⚠ `ID` 类型的值必须符合 XML 名字规则，不能以数字开头；`CDATA` 不会做任何类型检查。不要指望 DTD 校验日期或数字范围。

## 2.4 DTD 的局限

- 不支持命名空间。
- 缺少数据类型和范围约束。
- 实体机制带来 XXE 风险。

新项目基本不再用 DTD。它仍存在于历史系统和部分标准文档中，读得懂即可，不建议作为新契约的首选。不可信输入上的 DTD 处理详见第 7 篇。

# 3. XSD

## 3.1 schema 根与命名空间

XSD 文件本身也是 XML：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns:lib="https://example.com/library"
           targetNamespace="https://example.com/library"
           elementFormDefault="qualified">
  <xs:element name="library">
    <xs:complexType><xs:sequence>
      <xs:element name="book" maxOccurs="unbounded">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="title" type="xs:string"/>
            <xs:element name="author" type="xs:string" minOccurs="0"/>
            <xs:element name="price" type="xs:decimal"/>
          </xs:sequence>
          <xs:attribute name="id" type="xs:ID" use="required"/>
        </xs:complexType>
      </xs:element>
    </xs:sequence></xs:complexType>
  </xs:element>
</xs:schema>
```

对应的实例文档：

```xml
<library xmlns="https://example.com/library"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://example.com/library library.xsd">
  <book id="b1">
    <title>XML 入门</title><author>张三</author><price>59.00</price>
  </book>
</library>
```

| 属性 | 作用 |
|------|------|
| `xmlns:xs` | 引入 XSD 语言自身的命名空间 |
| `targetNamespace` | 本 schema 定义的元素所属命名空间 |
| `elementFormDefault="qualified"` | 局部元素默认带目标命名空间 |
| `attributeFormDefault` | 局部属性是否带目标命名空间，默认 `unqualified` |

⚠ `targetNamespace`、实例文档的 `xmlns`、`xsi:schemaLocation` 里的 URI 必须一致。三者写错一个，校验器就会报"找不到元素声明"。

## 3.2 元素与简单类型

| 类型 | 用途 | 示例 |
|------|------|------|
| `xs:string` | 字符串 | `hello` |
| `xs:integer` | 整数 | `42` |
| `xs:decimal` | 精确小数 | `59.00` |
| `xs:dateTime` | 日期时间 | `2026-11-06T10:30:00` |

```xml
<xs:element name="price" type="xs:decimal"/>
<xs:element name="published" type="xs:boolean"/>
<xs:element name="createdAt" type="xs:dateTime"/>
```

## 3.3 约束 facets

简单类型通过 `restriction` 加约束：

```xml
<xs:simpleType name="Price">
  <xs:restriction base="xs:decimal">
    <xs:minInclusive value="0"/>
    <xs:maxInclusive value="9999.99"/>
    <xs:fractionDigits value="2"/>
  </xs:restriction>
</xs:simpleType>

<xs:simpleType name="Status">
  <xs:restriction base="xs:string">
    <xs:enumeration value="NEW"/>
    <xs:enumeration value="PAID"/>
    <xs:enumeration value="CLOSED"/>
  </xs:restriction>
</xs:simpleType>
```

常用 facet：`minInclusive`、`maxInclusive`、`minLength`、`maxLength`、`pattern`、`enumeration`、`fractionDigits`、`totalDigits`。金额用 `xs:decimal`，不要用 `xs:double`，否则会出现精度误差。

## 3.4 复杂类型与 sequence

复杂类型描述"有子元素或属性"的元素：

```xml
<xs:complexType name="Book">
  <xs:sequence>
    <xs:element name="title" type="xs:string"/>
    <xs:element name="author" type="xs:string" minOccurs="0"/>
    <xs:element name="price" type="lib:Price"/>
    <xs:element name="tag" type="xs:string" minOccurs="0" maxOccurs="unbounded"/>
  </xs:sequence>
  <xs:attribute name="id" type="xs:ID" use="required"/>
</xs:complexType>
```

组合器：`xs:sequence` 按顺序出现，`xs:choice` 只能选一个，`xs:all` 每个最多一次且顺序不限。`minOccurs` 和 `maxOccurs` 控制出现次数，默认都是 `1`，`maxOccurs="unbounded"` 表示不限次数。

★ **关键认知**：XSD 默认按 `sequence` 严格匹配顺序。实例文档把 `author` 写在 `title` 前面，即使字段都齐全也会校验失败。顺序不重要时要用 `xs:all`，不要指望校验器宽容。

## 3.5 属性声明

```xml
<xs:attribute name="id" type="xs:ID" use="required"/>
<xs:attribute name="currency" type="xs:string" default="CNY"/>
<xs:attribute name="version" type="xs:string" fixed="1.0"/>
```

`use` 可以是 `required`、`optional`、`prohibited`。注意属性默认是 `unqualified`，即使 `elementFormDefault="qualified"`，局部属性也不自动进入目标命名空间，需要时显式写 `form="qualified"`。

复杂类型可以通过 `type` 复用，也可以用 `xs:extension` 继承已有类型并追加元素。继承适合表达"所有实体都有审计字段"这类公共结构，但层级过深会让契约难读。

# 4. schema 校验

## 4.1 Java 校验

```java
var factory = javax.xml.validation.SchemaFactory.newInstance(
        javax.xml.XMLConstants.W3C_XML_SCHEMA_NS_URI);
var schema = factory.newSchema(new java.io.File("library.xsd"));
var validator = schema.newValidator();

// 校验失败会抛 SAXException，错误信息包含行号和约束原因
validator.validate(new javax.xml.transform.stream.StreamSource(
        new java.io.File("library.xml")));
System.out.println("校验通过");
```

`Schema` 对象可以复用；`Validator` 不是线程安全的，不要跨线程共享。

## 4.2 命令行校验

```bash
# 用 XSD 校验
xmllint --noout --schema library.xsd library.xml

# 用 DTD 校验
xmllint --noout --dtdvalid library.dtd library.xml
```

输出会指出行号和违反的约束。排查时先确认是"元素顺序不对"还是"类型不匹配"，再回 schema 找对应声明。

# 5. 版本演进

DTD 是 SGML 时代的遗产，XSD 1.0 在 2001 年成为 W3C 推荐标准，补上了数据类型、命名空间和更精确的约束。XSD 1.1 又增加了 `xs:assert`、`xs:alternative`、`xs:openContent` 等能力。

| 能力 | XSD 1.0 | XSD 1.1 |
|------|---------|---------|
| 断言 | 不支持 | `xs:assert` |
| 条件类型 | 不支持 | `xs:alternative` |
| 工具支持 | 广泛 | 较少，需确认校验器 |

★ **关键认知**：XSD 1.1 表达力更强，但**工具链支持才是选型关键**。很多企业校验器和代码生成器仍停留在 1.0，写了 1.1 特性可能无法校验或生成代码。

契约演进策略：

- 只加可选元素，老客户端忽略新字段即可。
- 不要随意改类型，`xs:int` 改 `xs:string` 会让老数据失效。
- 不要插入必填元素，老文档会立刻校验失败。
- 大版本用命名空间隔离，例如 `https://example.com/order/v2`。

# 6. 常见错误

| 错误 | 表现 | 修复 |
|------|------|------|
| 命名空间不一致 | 找不到元素声明 | 对齐 `targetNamespace`、实例 `xmlns`、`schemaLocation` |
| 顺序写反 | 报 unexpected element | 调整顺序或改用 `xs:all` |
| 属性未加 `form` | 属性被当成无命名空间 | 需要时写 `form="qualified"` |
| 用 `xs:double` 存金额 | 出现精度误差 | 金额用 `xs:decimal` |

# 7. 动手练习

1. 为 `library/book` 写一份 XSD，要求 `title` 必填、`author` 可选、`price` 为两位小数且不小于 0。
2. 故意把实例文档中的 `author` 移到 `title` 前面，用 `xmllint` 观察报错，再把 `sequence` 改为 `all` 验证结果。
3. 给 `status` 增加枚举约束，分别用合法值和非法值测试校验器输出。

# 8. 小结

| 主题 | 要点 |
|------|------|
| DTD | 结构约束，语法非 XML，不支持命名空间 |
| XSD | XML 语法，支持类型、命名空间和精细约束 |
| 元素声明 | 名字、类型、出现次数 |
| 属性声明 | 类型、`use`、默认值、`form` |
| 复杂类型 | `sequence`、`choice`、`all` 组合子元素 |
| 校验 | `SchemaFactory` 或 `xmllint --schema` |
| 演进 | 加可选字段优先，大改动用命名空间隔离 |

下一篇进入解析器，讲 DOM、SAX、StAX 三种模型的内存布局、事件流程和 Java 用法，以及为什么大文件不能一把塞进 DOM。
