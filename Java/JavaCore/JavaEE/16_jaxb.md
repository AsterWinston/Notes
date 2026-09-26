
# 1. 概述

## 1.1 什么是 JAXB

JAXB（Java Architecture for XML Binding）是 JavaEE 的 **XML 绑定规范**，实现 **Java 对象 ↔ XML 文档** 的双向映射：

```
Java 对象 ──marshal──► XML 文档
Java 对象 ◄──unmarshal── XML 文档
```

- 通过注解声明映射规则，无需手写 DOM/SAX 解析。
- RI 由 JDK 内置（`com.sun.xml.bind`），Jakarta EE 9 后包名改为 `jakarta.xml.bind`。

## 1.2 JDK 版本注意

- JDK 8：内置 JAXB（`javax.xml.bind`）。
- **JDK 11+：JAXB 从 JDK 移除**，需显式引入依赖：

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

# 2. 核心注解

| 注解 | 作用 |
|------|------|
| `@XmlRootElement` | 类映射为 XML 根元素 |
| `@XmlElement(name=...)` | 字段/属性映射为子元素 |
| `@XmlAttribute` | 字段映射为属性 |
| `@XmlType` | 类型与元素顺序 |
| `@XmlAccessorType` | 字段/属性的访问策略（FIELD/PROPERTY/PUBLIC_MEMBER） |
| `@XmlValue` | 简单文本内容 |
| `@XmlTransient` | 忽略该字段 |
| `@XmlElementWrapper` | 集合包装元素 |
| `@XmlJavaTypeAdapter` | 自定义类型适配器 |
| `@XmlSeeAlso` | 声明子类 |

# 3. 映射示例

```java
@XmlRootElement(name = "user")
@XmlAccessorType(XmlAccessType.FIELD)
public class User {

    @XmlAttribute(name = "id")
    private Long id;

    @XmlElement(name = "name")
    private String name;

    @XmlElement(name = "age")
    private int age;

    @XmlElementWrapper(name = "roles")
    @XmlElement(name = "role")
    private List<String> roles = new ArrayList<>();

    @XmlTransient
    private String internalSecret;

    @XmlJavaTypeAdapter(LocalDateAdapter.class)
    @XmlElement(name = "birthday")
    private LocalDate birthday;

    // 构造器、getter/setter...
}
```

```java
public class LocalDateAdapter extends XmlAdapter<String, LocalDate> {
    @Override
    public LocalDate unmarshal(String v) {
        return LocalDate.parse(v);
    }
    @Override
    public String marshal(LocalDate v) {
        return v.toString();
    }
}
```

# 4. 编解码操作

## 4.1 Marshal（对象 → XML）

```java
JAXBContext context = JAXBContext.newInstance(User.class);
Marshaller marshaller = context.createMarshaller();
marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);   // 格式化
marshaller.setProperty(Marshaller.JAXB_ENCODING, "UTF-8");

User user = new User();
user.setId(1L);
user.setName("张三");
user.setAge(25);
user.setRoles(List.of("admin", "user"));

marshaller.marshal(user, System.out);
marshaller.marshal(user, new File("user.xml"));
```

输出：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<user id="1">
    <name>张三</name>
    <age>25</age>
    <roles>
        <role>admin</role>
        <role>user</role>
    </roles>
</user>
```

## 4.2 Unmarshal（XML → 对象）

```java
JAXBContext context = JAXBContext.newInstance(User.class);
Unmarshaller unmarshaller = context.createUnmarshaller();

User user = (User) unmarshaller.unmarshal(new File("user.xml"));
System.out.println(user.getName());
```

## 4.3 字符串/流形式

```java
// 字符串输入
String xml = "<user id=\"1\"><name>张三</name></user>";
User user = (User) unmarshaller.unmarshal(new StringReader(xml));

// 字符串输出
StringWriter sw = new StringWriter();
marshaller.marshal(user, sw);
String xmlStr = sw.toString();
```

# 5. 继承与多态

```java
@XmlSeeAlso({Admin.class, Customer.class})
@XmlRootElement
public abstract class BaseUser {
    @XmlAttribute
    private Long id;
}

@XmlRootElement(name = "admin")
public class Admin extends BaseUser {
    @XmlElement
    private String permission;
}
```

- 用 `@XmlSeeAlso` 声明子类，反序列化时可根据元素名识别具体子类。
- 也可用 `@XmlType(name="...")` + 工厂方法实现多态绑定。

# 6. 常用配置

## 6.1 Marshaller 常用属性

| 属性 | 作用 |
|------|------|
| `JAXB_FORMATTED_OUTPUT` | 格式化输出（缩进） |
| `JAXB_ENCODING` | 编码（默认 UTF-8） |
| `JAXB_FRAGMENT` | 是否省略 XML 声明头 |
| `JAXB_SCHEMA_LOCATION` | 输出 schemaLocation |

## 6.2 空值处理

- 默认：`null` 字段不输出元素。
- 可通过 `@XmlElement(nillable = true)` 输出 `<name xsi:nil="true"/>`。

# 7. 自定义 XmlAdapter 场景

| 场景 | 适配器用法 |
|------|-----------|
| 时间类型映射 | `LocalDate`/`LocalDateTime` ↔ String |
| 加密字段 | 编解码时加解密 |
| 集合转逗号字符串 | List ↔ String |
| 枚举自定义值 | 枚举 ↔ 特定字符串 |

# 8. JAXB vs 其他方案

| 对比 | JAXB | XStream | Jackson XML |
|------|------|---------|-------------|
| 标准 | JavaEE 标准 | 第三方 | 第三方 |
| 性能 | 中 | 中 | 高 |
| 注解映射 | 丰富 | 中 | 中 |
| 现代项目 | 存量系统 | 少 | Spring 生态常用 |

# 9. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| `ClassNotFoundException: JAXBContext` | JDK 11+ 需引入 `jaxb-runtime` |
| 字段未序列化 | 检查 `@XmlAccessorType` 策略（默认 PROPERTY，需要 getter） |
| 反序列化多态丢失 | 用 `@XmlSeeAlso` 声明子类 |
| 循环引用死循环 | 用 `@XmlTransient` 或 `@XmlIDREF` 处理对象引用 |
| 中文乱码 | 设置 `JAXB_ENCODING` 为 UTF-8 |
