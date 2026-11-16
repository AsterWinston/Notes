解析 XML 有两种完全不同的思路：把整份文档读成一棵树，或者边读边触发事件。前者是 DOM，后者包括 SAX 和 StAX。选错模型，轻则代码别扭，重则大文件直接把内存打满。本篇把三种模型的原理、Java 写法和选型讲清楚。

# 1. 解析的本质

## 1.1 从字节到结构

解析器做的事情可以拆成三步：

```text
字节流 ── 解码 ──→ 字符流 ── 词法/语法分析 ──→ 事件或节点树
```

- 解码：按声明的编码把字节转成字符。
- 词法分析：识别开始标签、属性、文本、结束标签。
- 语法分析：检查标签是否配对、结构是否良构。
- 输出：DOM 生成树，SAX/StAX 生成事件。

## 1.2 树模型与流模型

| 模型 | 代表 | 输出 | 内存 | 访问方式 |
|------|------|------|------|----------|
| 树模型 | DOM | 完整节点树 | 与文档大小成正比 | 随机访问、可修改 |
| 推式流 | SAX | 回调事件 | 常量级 | 只读、顺序 |
| 拉式流 | StAX | 可拉取事件 | 常量级 | 只读或写入、顺序 |

★ **关键认知**：DOM 的代价不是"解析慢"，而是**整棵树常驻内存**。一个 100 MB 的 XML 解析成 DOM 后，占用可能是原文件的数倍，因为每个节点、属性、文本都要变成对象。

# 2. DOM

## 2.1 树模型与内存布局

DOM 把文档解析成 `Document` 树：

```text
Document
└── Element(library)
    ├── Element(book)
    │   ├── Element(title) ── Text("XML 入门")
    │   └── Attr(id="b1")
    └── Element(book)
        └── Element(title) ── Text("XPath 实战")
```

树建好之后，可以按父节点、子节点、兄弟节点随意移动，也可以增删改节点，最后再序列化回去。

## 2.2 Java 读取示例

```java
import javax.xml.parsers.DocumentBuilderFactory;
import org.w3c.dom.Element;

var factory = DocumentBuilderFactory.newInstance();
factory.setNamespaceAware(true); // 需要命名空间时必须开启

var builder = factory.newDocumentBuilder();
var doc = builder.parse(new java.io.File("books.xml"));

var books = doc.getElementsByTagName("book");
for (int i = 0; i < books.getLength(); i++) {
    var book = (Element) books.item(i);
    var title = book.getElementsByTagName("title").item(0).getTextContent();
    System.out.println(book.getAttribute("id") + " / " + title);
}
```

`getElementsByTagName` 会返回**所有后代**中匹配的元素，不只是直接子元素。需要精确层级时，应该从父节点逐层取子节点，或改用 XPath。

## 2.3 DOM 的优缺点

| 优点 | 缺点 |
|------|------|
| 随机访问方便 | 内存占用高 |
| 可以修改后写回 | 解析大文件容易 OOM |
| API 直观，适合小文档 | 节点对象多，遍历成本高 |
| 支持命名空间、XPath | 空白文本节点需要额外处理 |

★ **关键认知**：DOM 适合**小文档和需要修改的场景**，例如配置文件、模板、少量报文。不要用 DOM 解析几百 MB 的日志或数据导出文件。

# 3. SAX

## 3.1 推模式

SAX 是事件驱动模型，解析器主动把事件推给 `Handler`：

```text
解析器读到一个开始标签
  → 调用 startElement()
解析器读到一段文本
  → 调用 characters()
解析器读到一个结束标签
  → 调用 endElement()
```

开发者只负责在回调里处理数据，不能主动让解析器"跳到下一个元素"。

## 3.2 Java 示例

```java
import javax.xml.parsers.SAXParserFactory;
import org.xml.sax.Attributes;
import org.xml.sax.helpers.DefaultHandler;

var factory = SAXParserFactory.newInstance();
factory.setNamespaceAware(true);
var parser = factory.newSAXParser();

var handler = new DefaultHandler() {
    private StringBuilder buffer = new StringBuilder();
    private boolean inTitle = false;

    @Override
    public void startElement(String uri, String localName,
                             String qName, Attributes attrs) {
        if ("title".equals(localName)) {
            inTitle = true;
            buffer.setLength(0); // 每个 title 重新开始收集
        }
    }

    @Override
    public void characters(char[] ch, int start, int length) {
        if (inTitle) {
            buffer.append(ch, start, length);
        }
    }

    @Override
    public void endElement(String uri, String localName, String qName) {
        if ("title".equals(localName)) {
            System.out.println(buffer.toString().trim());
            inTitle = false;
        }
    }
};

parser.parse(new java.io.File("books.xml"), handler);
```

## 3.3 characters 分片坑

`characters()` 不保证一次读完整个文本节点。解析器可能因为缓冲区边界，把一段文本拆成多次回调：

```text
<text>这是一段很长的文本</text>
```

可能触发：

```text
characters("这是一段")
characters("很长的文本")
```

⚠ 不要每次 `characters()` 都直接输出或覆盖变量，必须用 `StringBuilder` 累积，等 `endElement()` 再统一处理。

★ **关键认知**：SAX 的核心不是"回调 API"，而是**用状态机把流式事件还原成业务对象**。当前在哪个元素里、正在收集哪个字段，都要靠自己的布尔变量或栈来维护。

# 4. StAX

## 4.1 拉模式

StAX（Streaming API for XML）把控制权交给调用方：程序主动调用 `next()` 拉取下一个事件。

```text
DOM：解析器建好整棵树 → 你来访问
SAX：解析器推事件 → 你被动回调
StAX：你拉事件 → 解析器按需前进
```

拉模式的好处是可以随时跳过不关心的子树，代码也更像普通的循环。

## 4.2 Java 读取示例

```java
import javax.xml.stream.XMLInputFactory;
import javax.xml.stream.XMLStreamConstants;

var factory = XMLInputFactory.newInstance();
var reader = factory.createXMLStreamReader(
        new java.io.FileInputStream("books.xml"));

while (reader.hasNext()) {
    int event = reader.next();
    if (event == XMLStreamConstants.START_ELEMENT
            && "title".equals(reader.getLocalName())) {
        System.out.println(reader.getElementText()); // 直接读到结束标签
    }
}
reader.close();
```

`getElementText()` 会把当前元素内的文本一次性读完，适合简单字段；如果元素内部还有子元素，它会抛异常。

## 4.3 Java 写入示例

StAX 也能写：

```java
import javax.xml.stream.XMLOutputFactory;

var factory = XMLOutputFactory.newInstance();
var writer = factory.createXMLStreamWriter(
        new java.io.FileOutputStream("out.xml"), "UTF-8");

writer.writeStartDocument("UTF-8", "1.0");
writer.writeStartElement("books");
writer.writeStartElement("book");
writer.writeAttribute("id", "b1");
writer.writeStartElement("title");
writer.writeCharacters("XML 入门");
writer.writeEndElement();
writer.writeEndElement();
writer.writeEndElement();
writer.writeEndDocument();
writer.close();
```

写的时候必须严格成对调用 `writeStartElement` 和 `writeEndElement`，否则生成的文档不是良构的。

# 5. 三种模型的对比与选型

| 维度 | DOM | SAX | StAX |
|------|-----|-----|------|
| 内存 | 高，随文档增长 | 低 | 低 |
| 访问方式 | 随机 | 顺序、只读 | 顺序、可读可写 |
| 控制方式 | 主动 | 被动回调 | 主动拉取 |
| 修改文档 | 支持 | 不支持 | 写新文档可以 |
| 代码复杂度 | 低 | 中，需要状态机 | 中 |
| 适合场景 | 小文档、配置、模板 | 超大文件、只读统计 | 大文件、需要跳过或写入 |

选型口诀：

- 文件小、要改、要随机查：DOM。
- 文件大、只需扫一遍：SAX。
- 文件大、想自己控制节奏或边读边写：StAX。

★ **关键认知**：不要用"性能"三个字笼统比较三者。DOM 在小文档上往往更快，因为 API 调用简单；SAX 和 StAX 的优势是**内存不会随文件线性增长**。真正的判断依据是文档大小和是否需要随机访问。

# 6. 解析器配置

无论用哪种模型，生产环境都要显式配置解析器。以 DOM 为例：

```java
var factory = DocumentBuilderFactory.newInstance();
factory.setNamespaceAware(true);          // 启用命名空间
factory.setXIncludeAware(false);          // 禁用 XInclude 外部包含
factory.setExpandEntityReferences(false); // 不展开实体引用
factory.setFeature(
        "http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature(
        "http://xml.org/sax/features/external-general-entities", false);
factory.setFeature(
        "http://xml.org/sax/features/external-parameter-entities", false);
```

常见配置项：

| 配置 | 作用 | 建议 |
|------|------|------|
| `setNamespaceAware(true)` | 识别命名空间 | 需要时开启 |
| `disallow-doctype-decl` | 禁止 DTD | 不可信输入首选 |
| `external-general-entities` | 外部通用实体 | 设为 `false` |
| `external-parameter-entities` | 外部参数实体 | 设为 `false` |
| `setExpandEntityReferences(false)` | 保留实体引用 | 配合安全策略 |
| `setXIncludeAware(false)` | 禁用 XInclude | 默认关闭 |
| `FEATURE_SECURE_PROCESSING` | 限制资源消耗 | 处理不可信输入时开启 |

⚠ 不同解析器实现的 feature URI 可能不同。JDK 自带的 Xerces 支持上面这些，但换实现时要确认是否识别，否则 `setFeature` 会抛 `SAXNotRecognizedException`。

# 7. 常见坑

| 坑 | 表现 | 处理 |
|----|------|------|
| 未开命名空间 | `getElementsByTagNameNS` 返回空 | `setNamespaceAware(true)` |
| 文本分片 | 字段只剩最后一段 | 用 `StringBuilder` 累积 |
| 空白文本节点 | 第一个子节点不是业务元素 | 过滤空白节点 |
| 忘记关闭流 | 文件句柄泄漏 | `try-with-resources` 或 `close()` |
| DOM 解析大文件 | `OutOfMemoryError` | 换 SAX/StAX 或分片处理 |
| 默认实体展开 | 解析变慢或有安全风险 | 关闭 DTD 和外部实体 |
| 重复解析 | 同一文件反复 `parse` | 缓存解析结果或复用 `Schema` |

# 8. 动手练习

1. 用 DOM 读取一份包含多本书的 XML，输出每本书的 `id` 和标题。
2. 用 SAX 统计文件中 `book` 元素的数量，注意只在 `startElement` 计数。
3. 用 StAX 读取同一文件，遇到不需要的 `description` 子树时直接跳过，比较代码量和内存占用。

# 9. 小结

| 模型 | 一句话 |
|------|--------|
| DOM | 整棵树进内存，随机访问方便 |
| SAX | 事件推送，内存恒定，需要状态机 |
| StAX | 主动拉取，流式且可控 |
| 配置 | 命名空间、DTD、外部实体都要显式设置 |
| 选型 | 看文档大小和是否需要修改 |

下一篇进入 XPath 与 XSLT，讲如何用路径表达式从树中精确取数，以及如何用样式表把 XML 转成 HTML 或其他结构。
