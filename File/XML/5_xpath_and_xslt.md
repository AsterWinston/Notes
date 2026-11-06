DOM 给了你一棵树，但真正要从树里精确取数，还得靠 XPath。XPath 是 XML 的查询语言，XSLT 则用它做模板匹配和转换。本篇从路径语法讲到 Java 调用，最后给出性能和安全上的注意点。

# 1. XPath 是什么

## 1.1 把 XML 看成节点树

XPath 把文档抽象成节点树：

```text
Document
└── Element(library)
    ├── Element(book)
    │   ├── Attr(id="b1")          ← 属性节点
    │   ├── Element(title) ── Text("XML 入门")
    │   └── Comment("备注")
    └── Element(book)
```

XPath 表达式的结果可以是节点集、字符串、数字或布尔值。Java 里常用 `NodeList` 接收节点集结果。

## 1.2 一个最小例子

```xpath
/library/book/title
```

含义：从根节点出发，找 `library` 子元素，再找它的 `book` 子元素，最后取每个 `book` 的 `title` 子元素。

★ **关键认知**：XPath 的 `/` 是**层级分隔符**，不是文件路径分隔符。它以文档根为起点，按节点关系导航，和操作系统路径没有关系。

# 2. 路径语法

## 2.1 绝对路径与相对路径

| 写法 | 含义 |
|------|------|
| `/library/book` | 从文档根开始的绝对路径 |
| `book/title` | 从当前节点开始的相对路径 |
| `//title` | 文档中任意位置的 `title` |
| `.` / `..` | 当前节点 / 父节点 |
| `@id` | 当前节点的 `id` 属性 |
| `text()` | 当前节点的文本子节点 |

⚠ `//title` 看起来方便，实际会扫描大量节点。文档大时优先写具体路径，例如 `/library/book/title`。

## 2.2 轴

轴决定"往哪个方向找"：

| 轴 | 含义 |
|----|------|
| `child` | 子节点，默认轴 |
| `descendant` | 所有后代 |
| `descendant-or-self` | 自己及所有后代，`//` 的完整形式 |
| `parent` / `ancestor` | 父节点 / 所有祖先 |
| `attribute` / `self` | 属性 / 自己 |

完整写法用 `::`：`/library/book/child::title`、`/library/book/attribute::id`。`child` 可省略，`attribute` 可缩写为 `@`。

## 2.3 节点测试

| 测试 | 匹配 |
|------|------|
| `*` / `@*` | 任意元素 / 任意属性 |
| `node()` | 任意类型节点 |
| `text()` | 文本节点 |
| `comment()` | 注释节点 |
| `processing-instruction()` | 处理指令 |

## 2.4 谓词

谓词写在方括号里，用来过滤节点：

```xpath
/library/book[1]
/library/book[last()]
/library/book[@id='b1']
/library/book[price > 50]
/library/book[position() <= 3]
/library/book[title]
/library/book[contains(title,'XML')]
```

上面依次表示：第一本、最后一本、按 id 过滤、价格过滤、前三条、存在 title 子元素、标题包含 XML。

谓词可以串联：`/library/book[price > 50][1]/title` 表示先过滤再取第一个。

★ **关键认知**：XPath 的位置下标从 **1** 开始，不是 0。`book[1]` 是第一本，`book[0]` 不会匹配任何节点。

## 2.5 运算符

| 运算符 | 用途 | 示例 |
|--------|------|------|
| `+ - * div mod` | 算术 | `price * 1.1` |
| `=` `!=` | 相等比较 | `@id = 'b1'` |
| `<` `>` `<=` `>=` | 大小比较 | `price > 50` |
| `and` `or` | 逻辑运算 | `price > 50 and price < 100` |

⚠ XML 里写 XPath 时，`<` 需要写成 `&lt;`，否则会破坏 XML 结构。把 XPath 放在属性值里时尤其容易踩这个坑。

# 3. 常用函数

## 3.1 字符串函数

| 函数 | 作用 |
|------|------|
| `contains(s, sub)` / `starts-with(s, prefix)` | 包含 / 前缀判断 |
| `normalize-space(s)` | 去掉首尾空白并压缩中间空白 |

```xpath
/library/book[starts-with(title,'XML')]/title
/library/book[normalize-space(author)='张三']
```

## 3.2 数值与聚合函数

| 函数 | 作用 |
|------|------|
| `count(nodes)` | 节点数量 |
| `sum(nodes)` | 数值求和 |
| `position()` / `last()` | 当前位置 / 节点集最后位置 |

## 3.3 布尔与节点函数

| 函数 | 作用 |
|------|------|
| `not(x)` / `true()` / `false()` | 布尔运算与常量 |
| `name()` | 节点限定名 |
| `local-name()` | 本地名 |

★ **关键认知**：XPath 1.0 的比较规则比较隐晦。节点集和字符串比较时，会把节点集的字符串值拿来比；数字和字符串比较时又会做类型转换。写复杂条件时，最好显式用 `number()`、`string()` 转换，别依赖隐式规则。

# 4. Java 中使用 XPath

## 4.1 基本查询

```java
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.xpath.XPathConstants;
import javax.xml.xpath.XPathFactory;

var factory = DocumentBuilderFactory.newInstance();
factory.setNamespaceAware(true);
var doc = factory.newDocumentBuilder().parse(new java.io.File("books.xml"));

var xpath = XPathFactory.newInstance().newXPath();

// 编译一次，重复使用，避免每次解析表达式
var expr = xpath.compile("/library/book[@id='b1']/title");
var nodes = (org.w3c.dom.NodeList) expr.evaluate(doc, XPathConstants.NODESET);
for (int i = 0; i < nodes.getLength(); i++) {
    System.out.println(nodes.item(i).getTextContent());
}
```

`evaluate` 的返回类型必须和 `XPathConstants` 匹配：

| 返回类型 | 常量 | Java 类型 |
|----------|------|-----------|
| 节点集 | `NODESET` | `NodeList` |
| 字符串 | `STRING` | `String` |
| 数字 | `NUMBER` | `Double` |

## 4.2 命名空间

XPath 1.0 不会自动继承文档的默认命名空间，必须注册前缀：

```java
import javax.xml.namespace.NamespaceContext;
import javax.xml.XMLConstants;

xpath.setNamespaceContext(new NamespaceContext() {
    public String getNamespaceURI(String prefix) {
        return "lib".equals(prefix) ? "https://example.com/library"
                                    : XMLConstants.NULL_NS_URI;
    }
    public String getPrefix(String uri) { return null; }
    public java.util.Iterator<String> getPrefixes(String uri) {
        return java.util.List.of("lib").iterator();
    }
});

var nodes = (org.w3c.dom.NodeList) xpath.evaluate(
        "/lib:library/lib:book/lib:title", doc, XPathConstants.NODESET);
```

⚠ 即使文档用的是默认命名空间，XPath 里也不能直接写 `/library/book`。要么注册前缀，要么用 `local-name()` 绕开命名空间，但后者会牺牲精确性。

# 5. XSLT

## 5.1 转换模型

XSLT 用样式表描述"把输入树变成输出树"的规则：

```text
XML 输入 ── XSLT 处理器 ──→ HTML / XML / 文本输出
                ▲
                │
            XSLT 样式表
```

样式表本身也是 XML，核心是模板匹配：处理器遍历输入树，遇到匹配的节点就应用对应模板。

## 5.2 一个完整样式表

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="html" encoding="UTF-8" indent="yes"/>
  <xsl:template match="/">
    <html><body>
      <h1>图书列表</h1>
      <ul>
        <xsl:for-each select="library/book">
          <li>
            <xsl:value-of select="title"/>
            <xsl:text> - </xsl:text>
            <xsl:value-of select="price"/>
          </li>
        </xsl:for-each>
      </ul>
    </body></html>
  </xsl:template>
</xsl:stylesheet>
```

常用指令：

| 指令 | 作用 |
|------|------|
| `xsl:template match` | 定义匹配规则 |
| `xsl:apply-templates` | 对子节点应用模板 |
| `xsl:for-each` | 遍历节点集 |
| `xsl:value-of` | 输出节点值 |

## 5.3 Java 调用

```java
var factory = javax.xml.transform.TransformerFactory.newInstance();
// 编译样式表，重复转换时复用，避免每次解析 XSLT
var templates = factory.newTemplates(
        new javax.xml.transform.stream.StreamSource(new java.io.File("books.xsl")));

templates.newTransformer().transform(
        new javax.xml.transform.stream.StreamSource(new java.io.File("books.xml")),
        new javax.xml.transform.stream.StreamResult(new java.io.File("books.html")));
```

★ **关键认知**：`Templates` 是编译后的样式表，可以复用；`Transformer` 不是线程安全的。高并发场景下，应该缓存 `Templates`，每次转换新建 `Transformer`。

# 6. 性能与安全

## 6.1 性能注意

- 编译 XPath 表达式，不要在循环里反复 `evaluate` 字符串。
- 避免 `//` 全文档扫描，尽量写具体路径。
- 避免在谓词里调用昂贵函数，尤其是对每个节点都执行的函数。
- DOM + XPath 需要整棵树在内存；超大文档考虑 StAX 或流式 XSLT。

## 6.2 安全注意

XSLT 和 XPath 都可能触发外部资源加载：

```java
var factory = javax.xml.transform.TransformerFactory.newInstance();
factory.setAttribute(javax.xml.XMLConstants.ACCESS_EXTERNAL_DTD, "");
factory.setAttribute(javax.xml.XMLConstants.ACCESS_EXTERNAL_STYLESHEET, "");
```

⚠ 不要对用户提交的 XSLT 样式表直接执行。XSLT 可以调用扩展函数、读取外部文档，攻击面比普通 XML 解析更大。确实需要动态样式表时，应使用白名单和沙箱。

## 6.3 常见错误

| 错误 | 表现 | 修复 |
|------|------|------|
| 未注册命名空间 | 查询结果为空 | 设置 `NamespaceContext` |
| 下标从 0 开始 | 取不到第一个节点 | XPath 下标从 1 开始 |
| `evaluate` 类型不匹配 | `ClassCastException` | 返回类型与常量一致 |
| 模板未匹配 | 输出为空 | 检查 `match` 路径 |

# 7. 动手练习

1. 用 XPath 取出 `price` 大于 50 的所有 `title`，再按价格降序输出。
2. 给带默认命名空间的文档注册 `NamespaceContext`，用 `/lib:library/lib:book` 查询成功。
3. 写一个 XSLT，把图书列表转成 HTML 表格，并用 Java 的 `Templates` 复用样式表执行两次转换。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 路径 | `/` 绝对，`//` 任意后代，`.` 当前，`..` 父 |
| 轴 | `child` 默认，`@` 是 `attribute` 缩写 |
| 谓词 | 下标从 1 开始，可串联过滤 |
| Java | `XPathFactory` + `compile` + `evaluate` |
| 命名空间 | XPath 1.0 必须显式注册前缀 |
| XSLT | 模板匹配，`Templates` 可复用 |
| 安全 | 限制外部 DTD 和样式表访问 |

下一篇进入 JAXB 与对象绑定，讲如何把 XML 直接映射成 Java 对象，以及 `javax.xml.bind` 迁移到 `jakarta.xml.bind` 时要注意什么。
