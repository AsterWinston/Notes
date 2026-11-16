XML 允许多套词汇表混在同一份文档里，代价是"同名不同义"的冲突：订单里的 `id` 和用户里的 `id` 都叫 `id`，解析器凭什么区分？命名空间就是答案。它看起来只是几个 `xmlns` 声明，实际决定了元素和属性在解析后的真实身份。

# 1. 为什么需要命名空间

## 1.1 同名冲突

假设一份订单文档同时包含订单信息和用户信息：

```xml
<order>
  <id>o-1001</id>
  <user>
    <id>u-42</id>
    <name>张三</name>
  </user>
</order>
```

两个 `id` 含义完全不同。如果另一套系统也定义了自己的 `id`，或者需要把订单、用户、支付等多套标准拼在一起，光靠元素名无法判断某个 `id` 属于哪套规范。

命名空间（Namespace）给元素和属性加一层**全局限定名**，把"名字"变成"命名空间 URI + 本地名"。

## 1.2 三个核心概念

| 概念 | 说明 | 例子 |
|------|------|------|
| 命名空间 URI | 标识一套词汇表的字符串 | `https://example.com/order` |
| 前缀 | URI 在文档里的局部别名 | `o`、`u` |
| 扩展名 | 解析后的真实名字 | `{https://example.com/order}id` |

★ **关键认知**：命名空间 URI 只是**标识符**，不是下载地址。解析器不会去访问 `https://example.com/order`，也不要求这个地址真实存在。它的作用是和别人约定"这套元素归这个字符串管"。

# 2. 声明与使用

## 2.1 xmlns 与前缀

用 `xmlns:前缀="URI"` 声明一个前缀，再把这个前缀加到元素或属性名前：

```xml
<o:order xmlns:o="https://example.com/order"
         xmlns:u="https://example.com/user">
  <o:id>o-1001</o:id>
  <u:user>
    <u:id>u-42</u:id>
    <u:name>张三</u:name>
  </u:user>
</o:order>
```

这里的 `o:id` 和 `u:id` 展开后分别是：

```text
{https://example.com/order}id
{https://example.com/user}id
```

它们不是同一个元素，即使本地名都叫 `id`。

## 2.2 默认命名空间

如果一套词汇表占据文档主体，可以省略前缀，用默认命名空间：

```xml
<order xmlns="https://example.com/order">
  <id>o-1001</id>
  <amount>59.00</amount>
</order>
```

规则：没有前缀的元素自动属于默认命名空间；默认命名空间**不会**自动应用到没有前缀的属性上。

★ **关键认知**：默认命名空间只影响**元素**，不影响**属性**。这是 XML 命名空间里最容易记反的一条规则。即使根元素写了 `xmlns="..."`，它的 `id="42"` 属性仍然不属于任何命名空间。

## 2.3 作用域

命名空间声明的作用范围是**声明它的元素及其所有后代**：

```xml
<root xmlns:a="https://example.com/a">
  <a:item>属于 a</a:item>
  <child xmlns:a="https://example.com/b">
    <a:item>属于 b，前缀被重新定义</a:item>
  </child>
  <a:item>又回到 a</a:item>
</root>
```

解析后的归属：

```text
root
├── a:item          → {https://example.com/a}item
├── child
│   └── a:item      → {https://example.com/b}item   ← 就近声明生效
└── a:item          → {https://example.com/a}item   ← 离开 child 后恢复
```

前缀不是全局的，它像局部变量一样遵循词法作用域。同名前缀在不同子树里可以指向不同 URI。

## 2.4 多套命名空间共存

实际文档常常同时使用业务命名空间和标准命名空间：

```xml
<o:order xmlns:o="https://example.com/order"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://example.com/order order.xsd">
  <o:id>o-1001</o:id>
  <o:amount currency="CNY">59.00</o:amount>
</o:order>
```

`xsi` 是 XML Schema 实例命名空间的约定前缀，`schemaLocation` 告诉校验器去哪里找 schema。这里出现了两个 URI，各管各的，互不干扰。

# 3. 名字解析规则

## 3.1 URI 只是字符串标识

命名空间 URI 通常写成 URL 形式，但这只是历史习惯。比较两个名字是否属于同一命名空间时，看的是 URI 字符串是否相等，而不是网络资源是否可访问。

⚠ 不要依赖命名空间 URI 的"可访问性"做业务判断，也不要在运行时去请求它。URI 末尾多一个斜杠、大小写不同，都会被当成不同的命名空间。

## 3.2 前缀可以变，名字不能变

下面两段 XML 表达的是同一组扩展名：

```xml
<o:order xmlns:o="https://example.com/order">
  <o:id>o-1001</o:id>
</o:order>
```

```xml
<order xmlns="https://example.com/order">
  <id>o-1001</id>
</order>
```

第一段用前缀 `o`，第二段用默认命名空间，展开后都是 `{https://example.com/order}order` 和 `{https://example.com/order}id`。

★ **关键认知**：**前缀本身不是名字的一部分**。`o:id`、`ord:id`、`x:id` 只要绑定到同一个 URI，解析后就是同一个元素。写 XPath 或做对象映射时，应该匹配 URI 和本地名，而不是死记前缀。

## 3.3 属性的命名空间

属性的命名空间规则和元素不同：

| 写法 | 是否属于命名空间 | 说明 |
|------|------------------|------|
| `id="42"` | 否 | 无前缀属性永远没有命名空间 |
| `o:id="42"` | 是 | 展开为 `{URI}id` |
| `xml:lang="zh"` | 是 | `xml` 前缀由规范预定义 |
| `xmlns:o="..."` | 特殊 | 命名空间声明，不算普通属性 |

```xml
<order xmlns="https://example.com/order"
       xmlns:o="https://example.com/order">
  <id>o-1001</id>
  <item o:id="i-1" id="i-2"/>
</order>
```

这里根元素的默认命名空间是订单命名空间，所以 `id` 元素属于该命名空间；但 `item` 上的 `o:id` 属于命名空间，`id` 不属于任何命名空间。两个属性名字看起来一样，实际身份不同。

## 3.4 默认命名空间下的属性陷阱

```xml
<order xmlns="https://example.com/order">
  <amount currency="CNY">59.00</amount>
</order>
```

- `order`、`amount` 属于 `https://example.com/order`。
- `currency` **不属于**任何命名空间。

如果 schema 要求 `currency` 必须带命名空间，就必须写成 `o:currency="CNY"` 并声明 `o` 前缀。很多"schema 校验失败但看起来没问题"的案例都出在这里。

# 4. 常见冲突与坑

## 4.1 同一 URI 的多个前缀

```xml
<root xmlns:a="https://example.com/x"
      xmlns:b="https://example.com/x">
  <a:item/>
  <b:item/>
</root>
```

`a:item` 和 `b:item` 是同一个扩展名。前缀不同只是书写方式不同，不构成两个元素。

## 4.2 同一前缀的多次重定义

前缀重定义不是冲突，而是作用域覆盖。解析器会按最近的声明解析，离开该子树后恢复外层声明。阅读复杂文档时，不要只看根元素上的 `xmlns`，要沿着祖先链找最近的定义。

## 4.3 取消默认命名空间

默认命名空间可以用 `xmlns=""` 取消：

```xml
<order xmlns="https://example.com/order">
  <id>o-1001</id>
  <legacy xmlns="">
    <id>plain-id</id>
  </legacy>
</order>
```

`legacy` 和它内部的 `id` 都不属于任何命名空间。这是混合新旧格式时的常用技巧。

## 4.4 前缀未声明

```xml
<o:order>
  <o:id>o-1001</o:id>
</o:order>
```

这段 XML 不是良构的，因为 `o` 前缀没有绑定 URI。解析器会报 `Namespace prefix o is not declared` 之类的错误。前缀不是随便起的标签装饰，必须先声明。

## 4.5 把命名空间当成普通属性

在 DOM 里，`xmlns` 声明通常不作为普通属性暴露，而是通过命名空间相关 API 访问。用 `getAttribute("xmlns")` 取默认命名空间可能取不到，应该用 `lookupNamespaceURI` 或解析器的命名空间方法。

★ **关键认知**：命名空间声明是**元数据**，不是业务属性。不要把 `xmlns` 当成普通字段去读写，更不要把它序列化成业务数据。

# 5. Java 中处理命名空间

DOM 默认不开启命名空间支持，必须显式打开：

```java
var factory = javax.xml.parsers.DocumentBuilderFactory.newInstance();
factory.setNamespaceAware(true); // 不开则 getElementsByTagNameNS 无法工作
var doc = factory.newDocumentBuilder().parse(new java.io.File("order.xml"));

// 按命名空间 URI 和本地名查找，不依赖前缀
var nodes = doc.getElementsByTagNameNS("https://example.com/order", "id");
for (int i = 0; i < nodes.getLength(); i++) {
    System.out.println(nodes.item(i).getTextContent());
}

// 取带命名空间的属性
var item = (org.w3c.dom.Element) doc.getElementsByTagNameNS(
        "https://example.com/order", "item").item(0);
System.out.println(item.getAttributeNS("https://example.com/order", "id"));
```

StAX 读取时，`getNamespaceURI()` 返回当前元素的 URI，`getLocalName()` 返回本地名：

```java
var reader = javax.xml.stream.XMLInputFactory.newInstance()
        .createXMLStreamReader(new java.io.FileInputStream("order.xml"));
while (reader.hasNext()) {
    if (reader.next() == javax.xml.stream.XMLStreamConstants.START_ELEMENT
            && "https://example.com/order".equals(reader.getNamespaceURI())
            && "id".equals(reader.getLocalName())) {
        System.out.println(reader.getElementText());
    }
}
```

⚠ 用 XPath 查询带默认命名空间的文档时，必须注册命名空间上下文。XPath 1.0 里没有前缀的名字默认匹配"无命名空间"，不会自动继承文档的默认命名空间。

# 6. 动手练习

1. 写一份同时包含订单和用户命名空间的 XML，再用 DOM 的 `getElementsByTagNameNS` 分别取出两个 `id`。
2. 把下面这段 XML 中每个元素的扩展名写出来，注意属性：

```xml
<root xmlns="https://example.com/a" xmlns:b="https://example.com/b">
  <item b:id="1" id="2"/>
  <b:item/>
</root>
```

3. 解释为什么 `xmlns="https://example.com/a"` 不能让 `id="2"` 属性进入该命名空间，并给出两种修复方式。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 命名空间 URI | 标识一套词汇表，不要求可访问 |
| 前缀 | URI 的局部别名，可变，不是名字的一部分 |
| 默认命名空间 | 只作用于无前缀元素，不作用于属性 |
| 作用域 | 声明元素及其后代，可由内层重定义 |
| 扩展名 | `{URI}local`，比较时看 URI 和本地名 |
| 属性 | 无前缀属性永远无命名空间 |
| 取消默认 | 用 `xmlns=""` 回到无命名空间 |

下一篇进入 DTD 与 XSD，讲清楚如何把"元素顺序、出现次数、数据类型"这些约束写成机器可校验的规则，以及 DTD 为什么逐渐被 XSD 取代。
