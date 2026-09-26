# 1. 数据源类型

RAG（检索增强生成，Retrieval-Augmented Generation）的知识来源远不止文件。加载（Loading）阶段的第一件事，就是把所有可能的知识载体统一抽象成"文档 + 元数据"，后面的解析、分块、向量化才有共同输入。数据源选错，后面所有环节都是在错误的原料上做加工。

## 1.1 六类数据源全景

| 类型 | 例子 | 加载方式 | 典型难点 |
|------|------|----------|----------|
| 文件 | PDF/Word/Excel/PPT/Markdown | 文档解析器 | 格式繁杂、版面复杂 |
| 网页 | 官网、Wiki、帮助中心 | 爬取 + HTML 解析 | 反爬、正文提取 |
| 数据库 | 文章、工单、商品 | JDBC 查询 | 字段映射、增量位点 |
| API | 飞书、Confluence、Notion | 开放接口 | 鉴权、分页、限流 |
| 对象存储 | S3/OSS/MinIO | 下载后解析 | 大文件、网络抖动 |
| 消息 | IM 记录、邮件 | 导出后解析 | 会话切分、时间线 |

结论先行：**优先选"结构化、有稳定 ID、有更新时间"的数据源**。数据库与 API 天然带主键和 `updated_at`，增量与去重都好做；纯文件与网页则必须自己补元数据。

## 1.2 数据源选型与特征对比

### 1.2.1 结构化 vs 非结构化

结构化数据（数据库、API）在加载阶段就能拿到字段级语义，元数据几乎免费；非结构化数据（PDF、图片）必须先解析再抽取元数据。选型时先问一句：**这份知识有没有结构化副本？** 有就优先用副本，把 PDF 当兜底。

```
结构化数据 ──► 直接字段映射 ──► 元数据齐全
非结构化数据 ─► 解析 ─► 抽取 ─► 元数据可能缺失
```

### 1.2.2 拉取模式：全量、增量、事件

| 模式 | 触发 | 适用 | 代价 |
|------|------|------|------|
| 全量 | 定时重建 | 小规模、强一致 | 成本高 |
| 增量 | 定时扫描更新位点 | 中大规模、常规 | 需指纹 |
| 事件 | Webhook/CDC | 实时性要求高 | 需幂等 |

### 1.2.3 多源混合的合并策略

同一份知识可能同时存在于数据库和文件里。做法是**指定一个权威源（source of truth）**，其他源只做补充，避免同一内容出现两个版本导致检索结果自相矛盾。

## 1.3 边界与反例

- 反例一：把整个数据库表无脑导出成一个大文本文件，丢失了行级结构，检索时无法定位到具体记录。
- 反例二：网页抓取时把导航、广告一起入库，噪声块污染向量空间。
- 边界：二进制附件（如 PDF 扫描件）在没有 OCR 能力时，加载阶段应显式标记为"待 OCR"，而不是静默丢弃。

## 1.4 数据源接入的验收标准

接入一个新数据源，不能只看"能不能读出来"，还要看是否满足下游治理需求。

| 验收项 | 通过标准 |
|--------|----------|
| 稳定 ID | 每条记录有全局唯一、跨次加载不变的 ID |
| 更新时间 | 能取到 `updated_at` 或等价位点 |
| 权限标签 | 能标注租户、部门或 ACL |
| 正文完整 | 抽样解析无截断、无乱码 |
| 删除可见 | 能识别"已删除"状态 |

结论先行：**不满足"稳定 ID + 更新时间 + 权限标签"的数据源，应先在接入层补齐，再进 RAG 流水线**，否则后期治理成本极高。

# 2. Spring AI 的加载方式

Spring AI 用 `DocumentReader` 统一加载入口，返回值是 `List<Document>`。理解接口比记住类名更重要。

## 2.1 核心抽象：DocumentReader 与 Resource

### 2.1.1 DocumentReader 接口

```java
@FunctionalInterface
public interface DocumentReader extends Supplier<List<Document>> {
    List<Document> get();
}
```

所有加载器都实现这个接口，因此可以互相替换、组合。`Document` 由正文 `text` 与 `metadata` 两部分组成。

### 2.1.2 Resource 抽象

Spring 的 `Resource` 把 classpath、文件系统、URL、输入流统一起来，加载器通常接收 `Resource`，因此同一套代码既能读本地文件也能读网络资源。

```java
Resource r1 = new ClassPathResource("/docs/handbook.docx");
Resource r2 = new FileSystemResource("D:/docs/handbook.docx");
Resource r3 = new UrlResource("https://example.com/handbook.pdf");
```

## 2.2 单文件加载

```java
Document document = new TikaDocumentReader(
        new ClassPathResource("/docs/handbook.docx")).get().get(0);
```

`get()` 返回列表，是因为一个文件可能被切成多段（如 PDF 按页）。取第 0 个前应先判断非空。

## 2.3 目录批量加载

```java
@Bean
public List<Document> loadDocuments() throws IOException {
    Resource[] resources = new PathMatchingResourcePatternResolver()
            .getResources("file:D:/knowledge/**/*.pdf");

    List<Document> documents = new ArrayList<>();
    for (Resource resource : resources) {
        documents.addAll(new PagePdfDocumentReader(resource).get());
    }
    return documents;
}
```

结论先行：批量加载要**限制匹配范围与单文件大小**，否则一个误放的超大文件会拖垮整个加载任务。

## 2.4 数据库加载

```java
public class DbDocumentReader implements DocumentReader {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public List<Document> get() {
        return jdbcTemplate.query(
                "SELECT id, title, content, updated_at, dept FROM articles WHERE status = 'PUBLISHED'",
                (rs, i) -> new Document(
                        rs.getString("content"),
                        Map.of(
                                "id", rs.getString("id"),
                                "title", rs.getString("title"),
                                "dept", rs.getString("dept"),
                                "updatedAt", rs.getTimestamp("updated_at").toString())));
    }
}
```

数据库加载的关键是**把主键与更新时间写进元数据**，这两项是增量与去重的基础。

## 2.5 边界与坑

- 坑一：`Map.of` 不允许 null 值，数据库字段为 null 时直接抛异常。应先用 `Optional` 或自定义 Map 构造。
- 坑二：一次性 `query` 全表到内存，百万行会 OOM。大表要分页游标读取。
- 边界：跨库/跨表关联的知识，建议在 SQL 里先 join 成扁平结构，而不是加载后再关联。

## 2.6 自定义加载器与组合

当内置 Reader 不满足需求（如从消息队列、图数据库加载）时，实现 `DocumentReader` 即可接入统一流水线。

```java
public class KafkaDocumentReader implements DocumentReader {

    private final Consumer<String, String> consumer;

    public KafkaDocumentReader(Consumer<String, String> consumer) {
        this.consumer = consumer;
    }

    @Override
    public List<Document> get() {
        List<Document> result = new ArrayList<>();
        ConsumerRecords<String, String> records =
                consumer.poll(Duration.ofSeconds(1));
        for (ConsumerRecord<String, String> rec : records) {
            result.add(new Document(rec.value(), Map.of(
                    "source", "kafka:" + rec.topic(),
                    "offset", rec.offset(),
                    "partition", rec.partition())));
        }
        return result;
    }
}
```

多个 Reader 可用组合模式聚合，把"多源加载"变成一个 Reader。

```java
public class CompositeDocumentReader implements DocumentReader {

    private final List<DocumentReader> readers;

    public CompositeDocumentReader(List<DocumentReader> readers) {
        this.readers = readers;
    }

    @Override
    public List<Document> get() {
        return readers.stream()
                .flatMap(r -> r.get().stream())
                .toList();
    }
}
```

边界：组合 Reader 要注意部分失败——某个子 Reader 抛异常不应导致整体失败，应逐个捕获并记录。

# 3. LangChain4j 的加载方式

LangChain4j 的加载由 `DocumentLoader` 与 `DocumentParser` 两部分组成：加载器负责"找到文件"，解析器负责"读出文本"。这种拆分比 Spring AI 更显式。

## 3.1 DocumentLoader 抽象

`FileSystemDocumentLoader` 提供静态方法，核心参数是路径、路径匹配器与解析器。

## 3.2 单文件与目录

```java
// 单文件
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/handbook.pdf"),
        new ApachePdfBoxDocumentParser());

// 目录（递归）
List<Document> docs = FileSystemDocumentLoader.loadDocuments(
        Path.of("D:/knowledge"), parser);

// 按模式匹配
List<Document> pdfs = FileSystemDocumentLoader.loadDocuments(
        Path.of("D:/docs"), PathMatcher.of("*.pdf"), parser);
```

## 3.3 模式匹配与元数据

`PathMatcher.of("*.pdf")` 只匹配当前目录，若要递归需配合 glob（如 `**/*.pdf`）。LangChain4j 默认会为文档补上 `absolute_directory_path` 与 `file_name` 元数据，但业务字段（租户、部门）仍需自行注入。

```java
for (Document doc : pdfs) {
    doc.metadata().put("tenantId", "acme");
    doc.metadata().put("department", "HR");
}
```

## 3.4 两大框架对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 加载入口 | `DocumentReader` | `DocumentLoader` + `DocumentParser` |
| 返回类型 | `List<Document>` | `Document` / `List<Document>` |
| 解析器可插拔 | 每个 Reader 绑定 | 加载与解析分离 |
| 与生态整合 | Spring 依赖注入好 | 独立、轻量 |

结论先行：**已是 Spring 技术栈就选 Spring AI，想细粒度控制解析就选 LangChain4j。**

# 4. 网页与 API 加载

## 4.1 静态网页抓取

```java
// 思路：抓取 HTML → 转纯文本 → 补元数据
public Document loadWebPage(String url) throws IOException {
    String html = Jsoup.connect(url).get().html();
    String text = Jsoup.parse(html).text();

    return Document.from(text, Metadata.from(Map.of(
            "source", url,
            "type", "web")));
}
```

`Jsoup.parse(html).text()` 会去掉标签，但导航、页脚也会被保留，需配合正文提取（如按 `<article>` 或 Readability 算法）。

## 4.2 动态渲染页面

### 4.2.1 无头浏览器

SPA（单页应用）的正文由 JavaScript 渲染，`Jsoup` 只能拿到空壳。此时用 Playwright、Selenium 等无头浏览器先渲染再取 DOM。

### 4.2.2 API 优先策略

多数站点内部有 XHR（XMLHttpRequest）接口，直接调接口拿 JSON 比渲染页面更稳、更快。**能调 API 就不要爬页面。**

## 4.3 抓取合规与边界

- 遵守 `robots.txt` 与站点服务条款。
- 控制频率，加退避（backoff），避免被封。
- 记录抓取时间，网页内容会变，需重抓。
- 边界：登录态页面抓取涉及凭证管理，务必最小权限。

> 网页抓取要注意：反爬、频率限制、robots.txt、内容时效。

# 5. 元数据设计（关键）

**元数据必须在加载阶段就设计好**，事后补需要重建索引——这是加载阶段最容易返工的地方。

## 5.1 为什么必须前置设计

元数据决定三件事：检索时的过滤条件、回答时的引用来源、治理时的更新与删除。索引一旦建成，向量库只存向量与元数据快照，事后补字段等于全量重跑。因此元数据清单应在写第一行加载代码前确定。

## 5.2 元数据字段清单

### 5.2.1 溯源类

| 字段 | 用途 |
|------|------|
| `source` | 引用来源 |
| `title` / `section` | 定位与展示 |
| `docType` | 分类过滤 |

### 5.2.2 权限类

| 字段 | 用途 |
|------|------|
| `tenantId` | 多租户隔离 |
| `department` / `acl` | 权限过滤 |

### 5.2.3 时效与版本类

| 字段 | 用途 |
|------|------|
| `updatedAt` | 时效管理、排序 |
| `version` | 版本治理 |

## 5.3 元数据写入代码

```java
Document doc = new Document(content, Map.of(
        "source", "employee-handbook.pdf",   // 来源（用于引用）
        "title", "员工手册",                   // 标题
        "section", "第三章 报销",              // 章节（用于定位）
        "tenantId", "acme",                   // 租户（隔离）
        "department", "HR",                    // 部门（权限）
        "updatedAt", "2026-01-15",             // 时效
        "version", "v2",                       // 版本
        "docType", "policy"                    // 类型
));
```

| 元数据 | 用途 |
|--------|------|
| `source` | 引用来源 |
| `title` / `section` | 定位与展示 |
| `tenantId` | 多租户隔离 |
| `department` / `acl` | 权限过滤 |
| `updatedAt` | 时效管理、排序 |
| `version` | 版本治理 |
| `docType` | 分类过滤 |

## 5.4 边界与坑

- 坑一：把权限过滤留到检索后再做，等于先把越权内容召回，存在泄漏风险。**权限字段必须参与向量检索的过滤（pre-filter）。**
- 坑二：元数据值用中文枚举（"人力资源" vs "HR"）不统一，过滤会漏。建议定义枚举常量。
- 边界：元数据不是越多越好，字段过多会增大索引与过滤开销，只留真正参与过滤/展示的字段。

# 6. 增量加载

## 6.1 指纹模型

```java
public record DocFingerprint(String id, String contentHash, Instant updatedAt) {}
```

指纹 = 稳定 ID + 内容哈希 + 更新时间。内容哈希用 SHA-256，避免用 `updatedAt` 单独判断（有些系统更新了时间却没改内容）。

## 6.2 增量流程

```java
public void incrementalLoad(List<Source> sources) {
    for (Source source : sources) {
        DocFingerprint current = fingerprint(source);
        DocFingerprint stored = fingerprintStore.get(source.id());

        if (stored == null) {
            ingest(source);                       // 新增
        } else if (!current.contentHash().equals(stored.contentHash())) {
            reindex(source);                      // 变更
        }
        // 相同则跳过
    }
}
```

> 用内容哈希判断变更，避免重复向量化（省成本）。

## 6.3 触发方式：定时与事件

| 方式 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| 定时扫描 | `@Scheduled` | 简单、可控 | 有延迟 |
| 事件驱动 | Webhook/MQ | 实时 | 需幂等 |
| CDC | 数据库日志 | 准实时、无侵入 | 运维复杂 |

## 6.4 删除与失效处理

增量不仅处理"新增/变更"，还要处理**删除**。数据库软删（`status='DELETED'`）时，向量库对应文档也要删除，否则检索会命中已删内容。

```java
public void handleDeleted(String id) {
    vectorStore.delete(List.of(id));
    fingerprintStore.remove(id);
}
```

## 6.5 边界与坑

- 坑一：哈希基于正文，但正文里含页眉页码，每次解析结果略有差异，导致"假变更"。应基于清洗后的稳定文本计算。
- 坑二：并发增量任务可能同时写同一文档，需要幂等键（id + version）。
- 边界：内容没变但元数据（如部门）变了，也要更新索引中的元数据。

## 6.6 幂等与并发控制

增量任务可能被重复触发（重试、多实例调度），必须保证幂等：同一 `id` 同一 `version` 重复写入不产生副作用。

```java
// 伪代码：基于幂等键的写入
String idempotentKey = docId + ":" + version;
if (idempotentStore.exists(idempotentKey)) {
    return; // 已处理，直接跳过
}
vectorStore.upsert(document);
idempotentStore.mark(idempotentKey);
```

并发场景下用分布式锁（如基于 Redis 的锁）保证同一文档不被两个 worker 同时处理。

```java
// 伪代码：分布式锁
try (Lock lock = lockManager.acquire("doc:" + docId)) {
    incrementalLoadOne(docId);
}
```

边界：幂等存储本身也可能成为瓶颈，注意设置过期时间与清理策略。

# 7. 去重

## 7.1 内容级去重

```java
// 内容级去重
Set<String> seen = new HashSet<>();
List<Document> unique = documents.stream()
        .filter(d -> seen.add(sha256(d.getText())))
        .toList();
```

去重能显著降低向量化成本与检索噪声。

## 7.2 语义级去重

同一内容的不同措辞（如两份相似的政策）无法被哈希识别。可用向量相似度做近邻去重：新文档与已有文档余弦相似度超过阈值（如 0.95）则判为重复。

```java
// 伪代码：语义去重
float[] emb = embeddingModel.embed(text);
List<Match> near = vectorStore.similaritySearch(
        SearchRequest.query(text).withTopK(1));
if (!near.isEmpty() && near.get(0).score() > 0.95) {
    return; // 视为重复，跳过
}
```

## 7.3 边界与坑

- 坑一：`sha256` 对空白敏感，先规范化（trim、统一换行）再哈希。
- 坑二：短文本（如标题）哈希去重会误删，阈值要按长度区分。
- 边界：去重应在"块级"与"文档级"分别做，块级去重能消除重叠造成的冗余。

# 8. 加载阶段的质量检查

## 8.1 校验清单

| 检查项 | 目的 |
|--------|------|
| 非空 | 防止解析失败 |
| 长度下限 | 过滤碎片 |
| 必填元数据 | 保证可过滤/可引用 |
| 编码 | 防乱码 |

## 8.2 校验代码

```java
public void validate(Document doc) {
    Assert.hasText(doc.getText(), "内容为空");
    Assert.isTrue(doc.getText().length() > 50, "内容过短，可能解析失败");
    Assert.notNull(doc.getMetadata().get("source"), "缺少来源");
    Assert.notNull(doc.getMetadata().get("tenantId"), "缺少租户");
}
```

> 解析失败（内容为空、乱码、过短）是最常见的隐性质量问题，必须校验。

## 8.3 告警与失败处理

校验失败不应静默跳过。做法：记录失败文档 ID、失败原因，汇总指标（失败率），超过阈值触发告警。失败文档进入"隔离区"，人工复核后重试，而不是直接丢弃。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 元数据缺失 | 加载时强制校验 |
| 重复入库 | 内容哈希去重 |
| 全量重复向量化 | 增量 + 指纹判断 |
| 解析失败静默 | 校验 + 告警 |
| 数据源权限 | 加载时带权限标签 |
| 大文件 OOM | 流式、分批、限大小 |
| 无更新机制 | 定时/事件驱动增量 |

## 9.2 大文件与流式

超大 PDF/日志不能一次读入内存。做法：按页/按行流式读取，边读边解析边写向量库；同时设置单文件大小上限（如 50MB），超限走异步队列。

```java
// 伪代码：流式加载
try (Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8)) {
    lines.forEach(this::ingestLine);
}
```

## 9.3 可观测性

记录每次加载的指标：文档数、成功数、失败数、总耗时、平均文档大小、去重命中数。这些指标是定位"检索效果变差"的第一手线索。

## 9.4 成本控制

向量化按 Token 计费，加载阶段的每一个浪费都会放大到全库。

| 浪费来源 | 控制手段 |
|----------|----------|
| 重复向量化 | 内容指纹增量 |
| 重复内容 | 文档级/块级去重 |
| 噪声内容 | 解析清洗 + 质量校验 |
| 无效元数据 | 只保留参与过滤的字段 |

结论先行：**在加载阶段省下的每一份无效 Token，都会在向量化账单上被放大成百上千倍。**

# 10. 小结

- 数据源多样：文件、网页、数据库、API、对象存储、消息。
- 元数据必须在加载阶段设计并写入，且权限字段要参与检索过滤。
- 用内容指纹实现增量加载，避免重复向量化，并处理删除。
- 内容级去重降低噪声与成本，语义级去重补足近似重复。
- 加载后必须做质量校验（空、短、缺元数据）并告警。
- 大文件流式处理，加载全程可观测。
