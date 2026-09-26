
# 1. 概述

## 1.1 什么是 JAXP

JAXP（Java API for XML Processing）是 Java 处理 XML 的**标准 API**，提供统一的 XML 解析、转换接口，屏蔽具体解析器差异。

```
┌──────────────────────────────┐
│      JAXP API               │
│  SAX / DOM / StAX / XSLT    │
├──────────────────────────────┤
│  Provider 接口（SPI）        │
├──────────────────────────────┤
│  JDK 默认实现（JAXP 1.6）    │
│  Xerces / Xalan / ...        │
└──────────────────────────────┘
```

- JAXP 默认实现包含在 JDK 中（Xerces 的部分能力）。
- 从 JDK 11 起，Xalan 转换器部分独立化，推荐通过 Maven 显式引入（见 15.5）。

## 1.2 三种解析方式

| 方式 | 模型 | 特点 | 适用 |
|------|------|------|------|
| **DOM** | 树模型 | 整个文档载入内存，随机访问 | 文档小、需频繁修改 |
| **SAX** | 事件模型 | 流式解析，内存占用低 | 大文档、只读 |
| **StAX** | 拉式事件 | 流式解析，方向可控 | 大文档、可读写 |

# 2. DOM 解析

## 2.1 特点

- 将整个 XML 解析成内存中的 **Document 对象树**。
- 可随机访问、修改、保存。
- 缺点：文档大时内存占用高。

## 2.2 读取示例

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setNamespaceAware(true);
DocumentBuilder builder = factory.newDocumentBuilder();

Document doc = builder.parse(new File("users.xml"));

// 根节点
Element root = doc.getDocumentElement();

// 遍历
NodeList nodes = doc.getElementsByTagName("user");
for (int i = 0; i < nodes.getLength(); i++) {
    Element user = (Element) nodes.item(i);
    String name = user.getElementsByTagName("name").item(0).getTextContent();
    int age = Integer.parseInt(user.getAttribute("age"));
    System.out.println(name + " / " + age);
}
```

## 2.3 创建/写入示例

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
Document doc = builder.newDocument();

Element root = doc.createElement("users");
doc.appendChild(root);

Element user = doc.createElement("user");
user.setAttribute("id", "1");
Element name = doc.createElement("name");
name.setTextContent("张三");
user.appendChild(name);
root.appendChild(user);

TransformerFactory tf = TransformerFactory.newInstance();
Transformer transformer = tf.newTransformer();
transformer.setOutputProperty(OutputKeys.INDENT, "yes");
transformer.transform(new DOMSource(doc), new StreamResult(new File("out.xml")));
```

## 2.4 DOM 核心接口

| 接口 | 说明 |
|------|------|
| `Document` | 文档对象（整棵树） |
| `Element` | 元素节点 |
| `Node` | 通用节点（元素/文本/属性/注释等） |
| `NodeList` | 节点列表 |
| `Attr` | 属性节点 |
| `Text` | 文本节点 |

# 3. SAX 解析

## 3.1 特点

- **事件驱动**：边读边触发事件（`startElement`、`characters`、`endElement`），不需要一次载入内存。
- 内存占用低、速度快，适合超大 XML。
- 只读，无法修改。

## 3.2 示例

```java
SAXParserFactory factory = SAXParserFactory.newInstance();
SAXParser parser = factory.newSAXParser();

DefaultHandler handler = new DefaultHandler() {
    private StringBuilder sb = new StringBuilder();

    @Override
    public void startElement(String uri, String localName, String qName,
                             Attributes attributes) {
        if (qName.equals("name")) {
            sb.setLength(0);
        }
    }

    @Override
    public void characters(char[] ch, int start, int length) {
        sb.append(ch, start, length);
    }

    @Override
    public void endElement(String uri, String localName, String qName) {
        if (qName.equals("name")) {
            System.out.println("name = " + sb);
        }
    }
};

parser.parse(new File("users.xml"), handler);
```

## 3.3 DefaultHandler 主要回调

| 回调方法 | 触发时机 |
|----------|----------|
| `startDocument()` | 文档开始 |
| `endDocument()` | 文档结束 |
| `startElement()` | 元素开始标签 |
| `characters()` | 文本内容 |
| `endElement()` | 元素结束标签 |
| `error()` / `fatalError()` | 解析错误 |

# 4. StAX 解析

## 4.1 特点

- **拉式（Pull）解析**：程序主动调用 `next()` 获取事件，控制权在开发者。
- 比 SAX 更灵活（可随时跳过），性能好。
- 可读写（XMLStreamWriter）。

## 4.2 读取示例

```java
XMLInputFactory factory = XMLInputFactory.newInstance();
XMLStreamReader reader =
        factory.createXMLStreamReader(new FileInputStream("users.xml"));

while (reader.hasNext()) {
    int event = reader.next();
    if (event == XMLStreamConstants.START_ELEMENT) {
        if ("user".equals(reader.getLocalName())) {
            String id = reader.getAttributeValue(null, "id");
            System.out.println("user id = " + id);
        }
    } else if (event == XMLStreamConstants.CHARACTERS) {
        String text = reader.getText().trim();
        if (!text.isEmpty()) {
            System.out.println("text = " + text);
        }
    }
}
reader.close();
```

## 4.3 写入示例

```java
XMLOutputFactory factory = XMLOutputFactory.newInstance();
XMLStreamWriter writer = factory.createXMLStreamWriter(
        new FileOutputStream("out.xml"));

writer.writeStartDocument("UTF-8", "1.0");
writer.writeStartElement("users");
writer.writeStartElement("user");
writer.writeAttribute("id", "1");
writer.writeStartElement("name");
writer.writeCharacters("张三");
writer.writeEndElement();
writer.writeEndElement();
writer.writeEndElement();
writer.writeEndDocument();
writer.close();
```

# 5. XSLT 转换

用 XSLT 样式表将 XML 转换为其他格式（XML/HTML/文本）：

```java
TransformerFactory tf = TransformerFactory.newInstance();
Source xslt = new StreamSource(new File("users.xsl"));
Transformer transformer = tf.newTransformer(xslt);

Source xml = new StreamSource(new File("users.xml"));
transformer.transform(xml, new StreamResult(new File("users.html")));
```

```xml
<!-- users.xsl -->
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
    <xsl:template match="/">
        <html>
            <body>
                <xsl:for-each select="users/user">
                    <p><xsl:value-of select="name"/></p>
                </xsl:for-each>
            </body>
        </html>
    </xsl:template>
</xsl:stylesheet>
```

## 5.1 JDK 9+ 注意

- JDK 9 起 Xalan 转换器不再默认绑定，使用 XSLT 需要显式引入依赖：

```xml
<dependency>
    <groupId>net.sf.saxon</groupId>
    <artifactId>Saxon-HE</artifactId>
    <version>12.4</version>
</dependency>
```

# 6. XXE 安全防护（重要）

解析不受信任的 XML 时，必须防御 XXE（XML 外部实体注入）：

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

// 禁用外部实体（核心配置）
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);

factory.setXIncludeAware(false);
factory.setExpandEntityReferences(false);
```

> 详细安全防护见 `Security/` 目录及 `Interview/60_web_security.md`。

# 7. 三种解析方式选型

| 场景 | 推荐 |
|------|------|
| 小文档、需要修改/随机访问 | DOM |
| 超大文档、只读流式 | SAX |
| 需要控制解析流程、可读写 | StAX |
| 需要格式转换 | XSLT |
| 需要对象映射 | JAXB（见 `16_jaxb.md`） |

# 8. 与 JSON 处理的对比

| 对比 | XML | JSON |
|------|-----|------|
| 解析方式 | DOM/SAX/StAX/JAXB | JSON-P/JSON-B |
| 数据体积 | 冗余大 | 紧凑 |
| 跨语言 | 强（XSD 校验） | 弱 |
| 现代生态 | 企业存量多 | Web/微服务主流 |
