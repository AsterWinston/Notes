手写 DOM 代码把 XML 转成对象，字段一多就变成体力活。JAXB（Java Architecture for XML Binding）用注解描述映射关系，让序列化和反序列化自动完成。本篇讲注解、命名空间映射、Jakarta 迁移，以及什么时候该换别的方案。

# 1. 对象绑定是什么

## 1.1 从 XML 到对象

对象绑定（binding）在 XML 结构和 Java 对象之间建立映射：

```text
book.xml                      Book 对象
<book id="b1">          →     id = "b1"
  <title>XML 入门</title>  →  title = "XML 入门"
  <price>59.00</price>    →   price = 59.00
```

- 反序列化（unmarshal）：XML → Java 对象。
- 序列化（marshal）：Java 对象 → XML。

## 1.2 JAXB 的组成

| 组件 | 作用 |
|------|------|
| 注解 | 描述类、字段、属性与 XML 的对应关系 |
| `JAXBContext` | 管理映射元数据，线程安全，可复用 |
| `Unmarshaller` | 把 XML 读成对象 |
| `Marshaller` | 把对象写成 XML |
| `XmlAdapter` | 自定义类型转换 |
| `package-info.java` | 包级命名空间配置 |

★ **关键认知**：`JAXBContext` 的创建成本较高，应该按类或包缓存复用；`Marshaller` 和 `Unmarshaller` 不是线程安全的，每次使用新建即可。

# 2. 常用注解

## 2.1 根元素与类型

```java
import jakarta.xml.bind.annotation.*;

@XmlRootElement(name = "book", namespace = "https://example.com/library")
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(propOrder = {"title", "price"})
public class Book {
    @XmlAttribute(name = "id", required = true)
    private String id;

    @XmlElement(name = "title", required = true)
    private String title;

    @XmlElement(name = "price")
    private java.math.BigDecimal price;
}
```

| 注解 | 作用 |
|------|------|
| `@XmlRootElement` | 声明可作为根元素的类 |
| `@XmlType` | 控制元素顺序、名称、命名空间 |
| `@XmlAccessorType` | 指定按字段还是按属性映射 |
| `@XmlElement` | 映射子元素 |
| `@XmlAttribute` | 映射属性 |
| `@XmlTransient` | 排除字段，不参与映射 |

`@XmlAccessorType(XmlAccessType.FIELD)` 表示只看字段，getter/setter 不参与映射。默认是 `PUBLIC_MEMBER`，会同时看 public 字段和属性，容易和字段重复映射。

## 2.2 元素与属性

```java
@XmlAccessorType(XmlAccessType.FIELD)
public class Book {
    @XmlAttribute
    private String id;              // <book id="...">

    @XmlElement(name = "book-title")
    private String title;           // <book-title>...</book-title>

    @XmlElement(nillable = true)
    private String note;            // 允许 xsi:nil="true"

    @XmlTransient
    private String cacheKey;        // 不写入 XML
}
```

⚠ `@XmlElement(required = true)` 主要影响 schema 生成和校验，不会让 Marshaller 自动补值。字段为 `null` 时，元素通常直接不输出，是否合法取决于 schema。

## 2.3 集合

集合需要区分"包装元素"和"重复元素"：

```java
@XmlElementWrapper(name = "tags")
@XmlElement(name = "tag")
private List<String> tags = new java.util.ArrayList<>();
```

输出：

```xml
<tags>
  <tag>xml</tag>
  <tag>java</tag>
</tags>
```

如果不需要外层 `<tags>`，去掉 `@XmlElementWrapper`，每个元素直接重复。

★ **关键认知**：JAXB 映射集合时，**字段必须初始化**。如果 `List` 为 `null`，反序列化空集合和序列化空列表的行为容易不一致，业务代码也可能直接空指针。统一写成 `new ArrayList<>()` 最稳。

## 2.4 适配器

JAXB 不直接支持 `LocalDate`、`Instant` 等类型，需要适配器：

```java
import jakarta.xml.bind.annotation.adapters.XmlAdapter;
import java.time.LocalDate;

public class LocalDateAdapter extends XmlAdapter<String, LocalDate> {
    public LocalDate unmarshal(String value) { return LocalDate.parse(value); }
    public String marshal(LocalDate value) { return value.toString(); }
}
```

```java
@XmlElement(name = "published")
@XmlJavaTypeAdapter(LocalDateAdapter.class)
private LocalDate published;
```

适配器适合格式固定的类型。需要多格式解析时，把解析逻辑放进适配器，不要在 getter 里做隐式转换。

# 3. 序列化与反序列化

## 3.1 反序列化

```java
import jakarta.xml.bind.JAXBContext;

var context = JAXBContext.newInstance(Book.class);
var unmarshaller = context.createUnmarshaller();

var book = (Book) unmarshaller.unmarshal(new java.io.File("book.xml"));
System.out.println(book.getTitle());
```

反序列化默认不校验 schema。需要严格校验时，先创建 `Schema`，再设置给 `Unmarshaller`：

```java
var schemaFactory = javax.xml.validation.SchemaFactory.newInstance(
        javax.xml.XMLConstants.W3C_XML_SCHEMA_NS_URI);
var schema = schemaFactory.newSchema(new java.io.File("book.xsd"));
unmarshaller.setSchema(schema);
```

## 3.2 序列化

```java
var marshaller = context.createMarshaller();
marshaller.setProperty(jakarta.xml.bind.Marshaller.JAXB_FORMATTED_OUTPUT, true);
marshaller.setProperty(jakarta.xml.bind.Marshaller.JAXB_ENCODING, "UTF-8");

marshaller.marshal(book, new java.io.File("out.xml"));
```

也可以写到字符串：

```java
var writer = new java.io.StringWriter();
marshaller.marshal(book, writer);
System.out.println(writer.toString());
```

# 4. 命名空间映射

## 4.1 package-info.java

包级配置可以把命名空间规则集中管理：

```java
@jakarta.xml.bind.annotation.XmlSchema(
        namespace = "https://example.com/library",
        elementFormDefault = jakarta.xml.bind.annotation.XmlNsForm.QUALIFIED,
        xmlns = @jakarta.xml.bind.annotation.XmlNs(
                prefix = "lib", namespaceURI = "https://example.com/library"))
package com.example.library;

import jakarta.xml.bind.annotation.*;
```

这个文件必须放在对应包下，文件名固定为 `package-info.java`。配置后，包内类的 `@XmlRootElement` 不写命名空间也会使用包级默认值。

## 4.2 字段级命名空间

单个元素需要不同命名空间时，直接在注解上指定：

```java
@XmlElement(name = "id", namespace = "https://example.com/user")
private String userId;
```

属性也可以带命名空间，但要注意属性默认是 `unqualified`：

```java
@XmlAttribute(name = "id", namespace = "https://example.com/user")
private String userId;
```

## 4.3 qualified 与 unqualified

| 配置 | 效果 |
|------|------|
| `elementFormDefault = QUALIFIED` | 局部元素进入目标命名空间 |
| `elementFormDefault = UNQUALIFIED` | 局部元素不带命名空间，默认值 |
| `attributeFormDefault = QUALIFIED` | 局部属性进入目标命名空间 |
| `attributeFormDefault = UNQUALIFIED` | 局部属性无命名空间，默认值 |

★ **关键认知**：JAXB 的命名空间映射错误，通常不是注解写错，而是**包级默认值和字段级覆盖混用**。排查时先看 `package-info.java`，再看字段上的 `namespace`，最后看 schema 里的 `elementFormDefault` 是否一致。

# 5. Jakarta 迁移

## 5.1 为什么迁移

Java 9 开始弃用 Java EE 模块，JDK 11 正式移除了 `javax.xml.bind`。新项目必须显式引入 JAXB 依赖；Jakarta EE 9 又把包名从 `javax.*` 改成了 `jakarta.*`。

| 时代 | 包名 | 典型版本 |
|------|------|----------|
| JAXB 2.x | `javax.xml.bind` | 2.3 |
| Jakarta EE 9+ | `jakarta.xml.bind` | 3.0、4.0 |

## 5.2 依赖

使用 Jakarta 命名空间时：

```xml
<dependency>
  <groupId>jakarta.xml.bind</groupId>
  <artifactId>jakarta.xml.bind-api</artifactId>
  <version>4.0.2</version>
</dependency>
<dependency>
  <groupId>org.glassfish.jaxb</groupId>
  <artifactId>jaxb-runtime</artifactId>
  <version>4.0.5</version>
</dependency>
```

仍在使用 `javax` 的老项目，对应依赖是 `javax.xml.bind:jaxb-api:2.3.1` 和 `org.glassfish.jaxb:jaxb-runtime:2.3.9`。

⚠ API 和 runtime 的实现版本必须匹配。只引 API 会在运行时找不到 `JAXBContextImpl` 之类的实现类。

## 5.3 迁移清单

1. 全局替换 `javax.xml.bind` 为 `jakarta.xml.bind`。
2. 替换 `javax.xml.bind.annotation` 为 `jakarta.xml.bind.annotation`。
3. 更新 `package-info.java` 的 import 和注解。
4. 检查 XML 绑定插件（如 `jaxb2-maven-plugin`）是否支持 Jakarta。
5. 确认 Spring Boot 版本：Spring Boot 3 使用 Jakarta 命名空间。
6. 跑一遍反序列化和序列化回归，重点看命名空间和日期格式。

# 6. 替代方案

| 方案 | 优点 | 缺点 |
|------|------|------|
| JAXB | 标准、注解直观、schema 支持好 | Jakarta 迁移成本、运行时依赖 |
| Jackson XML | JSON/XML 统一 API、生态活跃 | 注解与 JAXB 不同，复杂 schema 支持弱 |
| XStream | 配置少、老项目常见 | 安全历史问题多，需严格白名单 |
| Simple XML | 轻量、注解简单 | 社区和功能相对小 |
| DOM/StAX 手写 | 无额外依赖、完全可控 | 代码量大、易出错 |

如果项目已经大量使用 Jackson，优先考虑 `jackson-dataformat-xml`；如果是标准企业接口、需要 schema 和 JAXB 兼容，继续用 JAXB 更合适。

# 7. 常见坑

| 坑 | 表现 | 处理 |
|----|------|------|
| 没有无参构造器 | 反序列化报错 | 保留默认构造器 |
| 字段和 getter 重复映射 | 元素重复或启动报错 | 统一 `@XmlAccessorType` |
| 集合为 null | 空集合处理不一致 | 初始化为空列表 |
| 命名空间不一致 | 反序列化得到空对象 | 对齐包级和字段级配置 |
| 日期格式不匹配 | 解析异常 | 使用 `XmlAdapter` |
| 共享 Marshaller | 并发下数据错乱 | 每次新建或加锁 |
| 未防 XXE | 解析不可信 XML 有风险 | 限制 DTD 和外部实体 |

# 8. 动手练习

1. 为 `Book` 增加 `tags` 集合和 `published` 日期字段，分别用 `@XmlElementWrapper` 和 `XmlAdapter` 映射。
2. 用 `package-info.java` 配置命名空间，序列化后检查根元素和子元素是否都带正确命名空间。
3. 把示例中的 `jakarta.*` 改成 `javax.*`，补齐依赖，观察哪些导入和运行时报错最先出现。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 核心注解 | `@XmlRootElement`、`@XmlElement`、`@XmlAttribute` |
| 集合 | 初始化列表，区分包装元素和重复元素 |
| 适配器 | 自定义类型用 `XmlAdapter` |
| 命名空间 | 包级 `@XmlSchema` + 字段级覆盖 |
| 迁移 | `javax` → `jakarta`，API 与 runtime 版本匹配 |
| 替代 | Jackson XML、XStream、手写 StAX |

下一篇进入 XML 安全，讲 XXE 的攻击原理、外部实体如何读取文件和发起 SSRF，以及 DOM、SAX、StAX 各自该怎么加固。
