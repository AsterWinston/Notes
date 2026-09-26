# 1. ETL 三阶段

**结论先行**：ETL（Extract-Transform-Load，抽取-转换-加载）是"把文档变成可检索知识"的流水线。Spring AI 用 `DocumentReader`、`DocumentTransformer`、`DocumentWriter` 三个接口把这条链路标准化，开发者只需按需拼装。

## 1.1 ETL 概念与定位

### 1.1.1 什么是 ETL

```
DocumentReader ──▶ DocumentTransformer ──▶ DocumentWriter
   （Extract）         （Transform）          （Load）
   读取文档            切分/清洗/增强         写入 VectorStore
```

在 RAG（Retrieval-Augmented Generation，检索增强生成）系统中，ETL 对应**离线索引阶段**：它决定了知识以什么粒度、带什么元数据进入向量库。

### 1.1.2 为什么需要 ETL

原始文档（PDF、Word、HTML）无法直接检索，必须先：

1. 抽取为纯文本；
2. 切分成适合向量化的片段；
3. 补充元数据（来源、章节、权限）；
4. 转成向量并写入向量库。

**ETL 的质量直接决定 RAG 召回的上限**——分块不当，再好的模型也救不回来。

## 1.2 三阶段流水线

### 1.2.1 Extract 抽取

```
PDF/Word/HTML/JSON/Markdown
        │  Reader
        ▼
   List<Document>
```

### 1.2.2 Transform 转换

```
   List<Document>
        │  Transformer（TokenTextSplitter / KeywordMetadataEnricher ...）
        ▼
   List<Document>（带元数据）
```

### 1.2.3 Load 加载

```
   List<Document>（带元数据）
        │  Writer（VectorStore）
        ▼
     向量库
```

## 1.3 组件接口

| 接口 | 方法 | 职责 |
|------|------|------|
| `DocumentReader` | `get()` | 读取为 `List<Document>` |
| `DocumentTransformer` | `apply(List<Document>)` | 转换文档列表 |
| `DocumentWriter` | `write(List<Document>)` | 写入目标存储 |

# 2. DocumentReader

**结论先行**：Reader 的选择取决于文档格式。格式单一就用专用 Reader，格式混杂就用 Tika（Apache Tika，万能文本抽取库）兜底。

## 2.1 常用 Reader

### 2.1.1 文本与结构化 Reader

| Reader | 用途 |
|--------|------|
| `TextReader` | 纯文本 |
| `JsonReader` | JSON（可指定 JSON Pointer 提取字段） |
| `MarkdownDocumentReader` | Markdown（按标题层级） |
| `HtmlDocumentReader` | HTML |

### 2.1.2 PDF Reader

| Reader | 用途 |
|--------|------|
| `PagePdfDocumentReader` | 分页 PDF（按页） |
| `ParagraphPdfDocumentReader` | PDF 按段落（需目录） |

> PDF 是最难的格式：扫描件需要 OCR（Optical Character Recognition，光学字符识别），双栏排版可能错序。

## 2.2 Tika：一劳永逸的格式兼容

### 2.2.1 依赖与用法

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
```

```java
TikaDocumentReader reader = new TikaDocumentReader(
        new ClassPathResource("/docs/employee-handbook.docx"));
List<Document> docs = reader.get();
```

`TikaDocumentReader` 支持 PDF、Word、Excel、PPT 等，是"万能格式"方案。

### 2.2.2 Tika 的代价

- 抽取结果是**扁平文本**，会丢失标题层级与表格结构；
- 依赖较重，启动稍慢；
- 复杂排版可能抽取质量不佳。

> 取舍：**原型期用 Tika 快速覆盖格式，生产期对核心格式改用专用 Reader**。

## 2.3 JSON Reader

```java
JsonDocumentReader reader = new JsonDocumentReader(
        new ClassPathResource("/docs/faq.json"),
        JsonMetadataGenerator.builder().build(),
        "/faqs");   // JSON Pointer
```

JSON Pointer（JSON 指针）用于定位要抽取的数组或对象节点。

## 2.4 自定义 Reader

### 2.4.1 数据库 Reader

```java
public class DbDocumentReader implements DocumentReader {

    private final JdbcTemplate jdbcTemplate;

    public DbDocumentReader(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public List<Document> get() {
        return jdbcTemplate.query(
                "SELECT id, title, content, updated_at FROM articles",
                (rs, rowNum) -> new Document(
                        rs.getString("content"),
                        Map.of(
                                "id", rs.getString("id"),
                                "title", rs.getString("title"),
                                "updatedAt", rs.getTimestamp("updated_at").toString()
                        )));
    }
}
```

### 2.4.2 Reader 选型对比

| 场景 | 推荐 Reader | 理由 |
|------|-------------|------|
| 单一 Markdown 知识库 | `MarkdownDocumentReader` | 保留标题层级 |
| 多格式混合目录 | `TikaDocumentReader` | 一劳永逸 |
| 结构化 FAQ | `JsonReader` | 可提取字段 |
| 业务数据库 | 自定义 Reader | 灵活 |
| 扫描版 PDF | OCR + `TextReader` | 需先转文字 |

# 3. DocumentTransformer

**结论先行**：分块（chunking）是 RAG 效果的最大变量。没有万能参数，必须结合文档结构并评测驱动。

## 3.1 TokenTextSplitter（最常用）

### 3.1.1 参数详解

| 参数 | 含义 |
|------|------|
| `chunkSize` | 分块大小（Token） |
| `minChunkSizeChars` | 小于该字符数不单独成块 |
| `keepSeparator` | 是否保留分隔符 |

### 3.1.2 使用示例

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
        .withChunkSize(800)        // 每块目标 Token 数
        .withMinChunkSizeChars(350)
        .withMinChunkLengthToEmbed(5)
        .withMaxNumChunks(10000)
        .withKeepSeparator(true)
        .build();

List<Document> chunks = splitter.apply(docs);
```

## 3.2 分块策略选择

### 3.2.1 策略对比

| 策略 | 说明 | 适用 |
|------|------|------|
| 固定长度 | 按 Token/字符切 | 通用兜底 |
| 递归切分 | 按 段落→句子→词 逐级 | 结构化文本（推荐） |
| 语义切分 | 按语义相似度断点 | 高质量要求 |
| 按结构切分 | 按标题/章节 | Markdown/HTML |
| 父子分块 | 小块检索、大块喂模型 | 长文档问答 |

### 3.2.2 分块参数经验

```
经验起点：
- chunkSize：300~800 Token
- overlap（重叠）：10%~20% chunkSize，避免答案被切断
- 保留标题：把所属章节标题拼进每个块，增强上下文
- 表格/代码：尽量整块保留，避免从中间切断
```

> 分块是 RAG 效果的最大变量之一，详见 `5_RAG/DataProcessing/3_chunking_strategies.md`。

## 3.3 元数据增强

### 3.3.1 关键词增强

```java
// 自动生成关键词元数据
KeywordMetadataEnricher enricher = new KeywordMetadataEnricher(chatModel, 5);
List<Document> enriched = enricher.apply(chunks);
```

### 3.3.2 摘要增强

```java
// 自动生成文档摘要
SummaryMetadataEnricher summaryEnricher = new SummaryMetadataEnricher(
        chatModel, List.of(SummaryType.CURRENT, SummaryType.PREVIOUS));
```

> 元数据增强会调用 LLM，产生额外成本，适合高价值知识库。

## 3.4 Transformer 组合与顺序

```
raw docs
   │ TokenTextSplitter（先分块）
   ▼
chunks
   │ KeywordMetadataEnricher（再增强元数据）
   ▼
enriched chunks
   │
   ▼
Writer
```

顺序原则：**先切分、后增强**。若先增强再切分，摘要会被切碎且浪费 LLM 调用。

# 4. DocumentWriter

**结论先行**：`VectorStore` 本身就是 `DocumentWriter`，绝大多数场景无需自定义 Writer。写入时注意批量与幂等。

## 4.1 VectorStore 即 Writer

```java
DocumentWriter writer = vectorStore;
writer.write(chunks);
```

## 4.2 自定义 Writer

```java
public class MultiSinkWriter implements DocumentWriter {

    private final List<DocumentWriter> sinks;

    public MultiSinkWriter(List<DocumentWriter> sinks) {
        this.sinks = sinks;
    }

    @Override
    public void write(List<Document> documents) {
        // 同时写入向量库与全文检索库
        sinks.forEach(s -> s.write(documents));
    }
}
```

## 4.3 写入批处理

| 关注点 | 做法 |
|--------|------|
| 吞吐 | 每批 100~500 条 |
| 幂等 | 用内容哈希作为 ID 或去重键 |
| 失败重试 | 记录失败批次，可重放 |

# 5. 完整 ETL 示例

**结论先行**：一个可用的 ETL 服务应封装"抽取-补元数据-分块-写入"四步，并支持多格式入口与错误处理。

## 5.1 端到端服务

```java
@Service
public class KnowledgeIngestService {

    private final VectorStore vectorStore;

    public KnowledgeIngestService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public int ingest(Resource resource, String sourceName) {
        // 1. Extract
        DocumentReader reader = new TikaDocumentReader(resource);
        List<Document> raw = reader.get();

        // 2. 补元数据
        raw.forEach(d -> d.getMetadata().put("source", sourceName));

        // 3. Transform：分块
        TokenTextSplitter splitter = TokenTextSplitter.builder()
                .withChunkSize(800)
                .build();
        List<Document> chunks = splitter.apply(raw);

        // 4. Load
        vectorStore.add(chunks);
        return chunks.size();
    }
}
```

## 5.2 多格式入口

```java
public DocumentReader readerFor(String filename) {
    if (filename.endsWith(".md")) {
        return new MarkdownDocumentReader(...);
    }
    if (filename.endsWith(".json")) {
        return new JsonDocumentReader(...);
    }
    return new TikaDocumentReader(...);
}
```

## 5.3 错误处理与重试

| 故障 | 处理 |
|------|------|
| 文件损坏 | 跳过并记录，不中断整批 |
| Embedding 超时 | 指数退避重试 |
| 部分写入失败 | 记录批次，支持重放 |
| 超大文件 | 限制大小，超限拒绝 |

# 6. 增量与更新策略

**结论先行**：生产索引必须支持增量更新。核心是"能识别哪些文档变了"，并保证更新过程不中断查询。

## 6.1 更新模式

```
新增文档 → 直接 add
修改文档 → 先按 source 删除旧块，再 add 新块
删除文档 → 按 source 删除
```

```java
public void reindex(Resource resource, String source) {
    vectorStore.delete(FilterExpressionBuilder.b("source").eq(source).build());
    ingest(resource, source);
}
```

| 策略 | 说明 |
|------|------|
| 全量重建 | 简单但耗时、期间影响服务 |
| 增量更新 | 只处理变更文档，需记录文档指纹 |
| 别名切换 | 建新索引后原子切换，避免服务中断 |

## 6.2 文档指纹与去重

```java
// 用内容哈希做去重键与变更检测
String fingerprint = sha256(fileBytes);
if (fingerprint.equals(storedFingerprint(source))) {
    return 0; // 未变化，跳过
}
```

| 去重粒度 | 说明 |
|----------|------|
| 文件级 | 文件未变则整篇跳过 |
| 分块级 | 块内容哈希，避免重复块入库 |

## 6.3 别名切换

```
1. 建新索引 knowledge_v2
2. ETL 灌入全部数据
3. 抽样校验（数量、召回）
4. 别名 knowledge 指向 v2
5. 观察后删除 v1
```

# 7. 生产注意事项

**结论先行**：ETL 的生产问题集中在"资源、质量、成本"三类。资源靠限流与分批，质量靠校验，成本靠去重与缓存。

## 7.1 风险清单

| 风险 | 建议 |
|------|------|
| 大文件 OOM | 流式解析、分批处理、限制单文件大小 |
| 解析乱码 | 统一 UTF-8，扫描件需 OCR |
| 重复写入 | 用内容哈希去重 |
| 切分不当 | 结合文档结构调整，保留标题上下文 |
| 元数据缺失 | 入库前校验必填元数据 |
| 全量重建抖动 | 用别名切换 + 分批写入 |
| Embedding 成本 | 批量、缓存、去重后再向量化 |

## 7.2 质量校验

```java
// 入库前校验必填元数据
void validate(List<Document> docs) {
    for (Document d : docs) {
        if (!d.getMetadata().containsKey("tenantId")) {
            throw new IllegalStateException("缺少 tenantId：" + d.getId());
        }
    }
}
```

- 校验分块数量是否异常（如单文档被切成上千块）；
- 抽样检查分块内容是否语义完整；
- 统计空块、超短块比例。

## 7.3 成本优化

| 手段 | 效果 |
|------|------|
| 内容去重 | 减少重复 Embedding 调用 |
| 向量缓存 | 相同文本复用向量 |
| 本地模型 | 降低 API 费用 |
| 增量处理 | 只处理变更文档 |

# 8. 小结

## 8.1 核心要点

- ETL = Reader → Transformer → Writer，Spring AI 已提供全套组件。
- 分块策略是 RAG 效果的关键变量，没有万能参数，需评测驱动。
- 元数据在 Extract 阶段就要打好，事后补成本高。
- 增量更新 + 别名切换是生产级索引维护的标准做法。

## 8.2 实践清单

1. 按格式选择 Reader，核心格式用专用 Reader，其余用 Tika 兜底。
2. 先切分、后增强元数据，避免浪费 LLM 调用。
3. 入库前校验必填元数据，异常分块及时告警。
4. 用内容哈希去重与变更检测，支持增量更新。
5. 重建索引用别名切换，避免查询中断。
