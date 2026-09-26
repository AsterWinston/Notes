# 1. 文档处理全流程

结论先行：LangChain4j 把非结构化文件变成可检索向量的过程，是一条"解析 → 切分 → 向量化 → 入库"的流水线。解析（parse）与切分（split）是两个独立抽象，任何一环都可以按格式或场景替换，而不影响其余环节。

## 1.1 端到端数据流

```
文件（PDF/Word/Excel/PPT/HTML/Markdown/TXT）
        │  DocumentParser
        ▼
    Document（文本 + 元数据）
        │  DocumentSplitter
        ▼
    List<TextSegment>
        │  EmbeddingModel
        ▼
   EmbeddingStore
```

上图中每个箭头都是一次接口调用。`Document` 是不可变的文本与元数据容器；`TextSegment` 是 `Document` 被切分后的片段，同样携带元数据副本；`EmbeddingStore` 只负责存储向量与片段，不关心内容从哪来。

## 1.2 为什么解析与切分要分离

### 1.2.1 抽象边界

`DocumentParser` 的职责是"从字节流得到纯文本 + 元数据"，它只面对文件格式的差异；`DocumentSplitter` 的职责是"从一段长文本得到若干可嵌入片段"，它只面对文本长度的差异。两者关注点正交，因此可以自由组合：同一个切分器既能切 PDF 也能切 Word。

```java
DocumentParser parser = new ApachePdfBoxDocumentParser();
DocumentSplitter splitter = DocumentSplitters.recursive(800, 100);

Document document = FileSystemDocumentLoader.loadDocument(path, parser);
List<TextSegment> segments = splitter.split(document);
```

### 1.2.2 替换成本

| 变更场景 | 只需替换 | 不需要动 |
|----------|----------|----------|
| 新增一种文件格式 | `DocumentParser` | 切分、嵌入、存储 |
| 调整块大小 | `DocumentSplitter` | 解析、嵌入、存储 |
| 更换嵌入模型 | `EmbeddingModel` | 解析、切分 |
| 更换向量库 | `EmbeddingStore` | 解析、切分、嵌入 |

正因为边界清晰，RAG 系统的实验迭代（换切分参数、换模型）才不需要重写整条链路。

## 1.3 与 RAG 其他环节的关系

文档处理是 RAG 的"离线侧"。它产出的向量质量直接决定在线检索的上限：解析丢失的表格内容，检索阶段无法凭空恢复；切分不当造成的语义割裂，重排序（rerank）也难以弥补。因此工程上通常把 60% 以上的调优精力放在数据加工，而非 Prompt。

> 分块策略的深入讨论见 `5_RAG/DataProcessing/3_chunking_strategies.md`。

# 2. DocumentParser

结论先行：`DocumentParser` 是一个只有单个方法的函数式接口，不同实现之间的差异只体现在"支持格式"与"版面还原质量"两点上。

## 2.1 接口契约

### 2.1.1 parse 方法

```java
public interface DocumentParser {
    Document parse(InputStream inputStream);
}
```

方法接收 `InputStream` 而非 `Path`，意味着解析器不关心数据来源：本地文件、HTTP 响应流、对象存储流都可以。调用方负责流的生命周期，解析完成后框架会关闭流。

### 2.1.2 Document 结构

```java
Document document = parser.parse(inputStream);

String text = document.text();                 // 抽取出的纯文本
Metadata metadata = document.metadata();       // 可写元数据
Document copy = document.toBuilder()
        .metadata(metadata)
        .build();
```

`Document` 一旦构造完成，`text()` 不可变，但 `metadata()` 返回的 `Metadata` 是可变映射，可在解析后继续补充字段。

## 2.2 内置解析器总览

| 解析器 | 依赖 | 支持格式 |
|--------|------|----------|
| `TextDocumentParser` | 核心 | 纯文本 |
| `ApachePdfBoxDocumentParser` | `langchain4j-document-parser-apache-pdfbox` | PDF |
| `ApachePoiDocumentParser` | `langchain4j-document-parser-apache-poi` | Word/Excel/PPT |
| `ApacheTikaDocumentParser` | `langchain4j-document-parser-apache-tika` | 几乎全部格式 |
| `MarkdownDocumentParser` | `langchain4j-document-parser-markdown` | Markdown |
| `DoclingDocumentParser` | `langchain4j-document-parser-docling` | 高质量 PDF/版面（1.x） |

选择原则：能用专用解析器就不要用万能解析器，专用解析器对特定格式的还原通常更准确；只有格式未知或种类繁杂时才用 Tika 兜底。

## 2.3 PDF

### 2.3.1 依赖与用法

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-document-parser-apache-pdfbox</artifactId>
</dependency>
```

```java
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/handbook.pdf"),
        new ApachePdfBoxDocumentParser());
```

### 2.3.2 局限

PDFBox 按字符流抽取文本，遇到多栏排版、表格、页眉页脚时会打乱阅读顺序；扫描件没有文本层，抽取结果为空。前者需要 Docling 等基于版面的解析器，后者需要 OCR。

## 2.4 Office（Word/Excel/PPT）

```java
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/policy.docx"),
        new ApachePoiDocumentParser());
```

POI 解析 `.docx/.xlsx/.pptx`（OOXML）效果稳定；对老式 `.doc/.xls/.ppt` 二进制格式支持较弱，建议先转换为 OOXML 或交给 Tika。

## 2.5 Tika（万能兜底）

```java
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/unknown.xyz"),
        new ApacheTikaDocumentParser());
```

> Tika 兼容性最好，但对复杂版面（多栏、表格）的还原能力一般。

Tika 内部集成了大量格式探测器，适合"格式未知、以文本可用为目标"的批量导入场景。

## 2.6 Markdown

```java
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/readme.md"),
        new MarkdownDocumentParser());
```

Markdown 解析会保留标题层级信息，配合 `DocumentByParagraphSplitter` 可实现按标题切分，特别适合技术文档与知识库。

## 2.7 Docling

Docling 是 1.x 引入的高质量解析方案，擅长版面分析、表格结构还原与阅读顺序恢复，代价是模型下载与推理资源开销。适合对表格、多栏 PDF 有强需求的场景。

```java
Document document = FileSystemDocumentLoader.loadDocument(
        Path.of("D:/docs/report.pdf"),
        new DoclingDocumentParser());
```

## 2.8 从多种来源加载

### 2.8.1 目录递归

```java
List<Document> docs = FileSystemDocumentLoader.loadDocuments(
        Path.of("D:/knowledge-base"), parser);
```

递归加载目录下所有文件，每个文件产出一个 `Document`。

### 2.8.2 glob 过滤

```java
List<Document> pdfs = FileSystemDocumentLoader.loadDocuments(
        Path.of("D:/docs"), PathMatcher.of("*.pdf"), parser);
```

`PathMatcher` 支持 glob 语法，可在批量导入时只挑选目标格式，避免解析无关文件。

## 2.9 自定义解析器

当内置解析器无法满足需求（例如私有二进制格式、需要保留坐标信息）时，实现接口即可：

```java
public class CsvDocumentParser implements DocumentParser {

    @Override
    public Document parse(InputStream in) {
        try (BufferedReader reader =
                     new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8))) {
            String text = reader.lines().collect(Collectors.joining("\n"));
            return Document.from(text);
        } catch (IOException e) {
            throw new RuntimeException("CSV 解析失败", e);
        }
    }
}
```

## 2.10 解析器选择决策树

```
格式已知？
├─ 否 ──────────────────▶ ApacheTikaDocumentParser
└─ 是
   ├─ 纯文本/Markdown ───▶ Text / MarkdownDocumentParser
   ├─ Office 系列 ───────▶ ApachePoiDocumentParser
   └─ PDF
      ├─ 简单文本型 ─────▶ ApachePdfBoxDocumentParser
      └─ 表格/多栏 ──────▶ DoclingDocumentParser
```

选型口诀：**专用优先、Tika 兜底、版面复杂上 Docling、扫描件先 OCR**。

# 3. 元数据

结论先行：元数据是解析阶段就要补全的"一等公民"，后续的权限过滤、引用展示、增量更新全部依赖它，入库后再补会非常被动。

## 3.1 元数据的作用

元数据为每个片段提供上下文，使检索可以带上过滤条件（如只搜某个租户、某个部门），并让回答能标注引用来源。

```java
Document document = FileSystemDocumentLoader.loadDocument(path, parser);
document.metadata().put("source", path.getFileName().toString());
document.metadata().put("tenantId", tenantId);
document.metadata().put("updatedAt", LocalDate.now().toString());
```

## 3.2 常用字段

| 元数据 | 用途 |
|--------|------|
| `source` | 引用来源 |
| `tenantId` | 多租户隔离 |
| `department` | 权限过滤 |
| `updatedAt` | 时效管理 |
| `version` | 版本治理 |

> 解析阶段就要补全元数据，后续过滤与引用都依赖它。

## 3.3 元数据校验

生产环境应在入库前做必填校验，缺字段直接拒绝而非静默写入：

```java
private void validate(Metadata metadata) {
    for (String key : List.of("source", "tenantId", "updatedAt")) {
        if (metadata.getString(key) == null) {
            throw new IllegalArgumentException("缺少必填元数据: " + key);
        }
    }
}
```

## 3.4 元数据在检索中的应用

元数据最终服务于检索时的过滤条件。以下示例只检索当前租户、且指定部门的片段：

```java
EmbeddingSearchRequest request = EmbeddingSearchRequest.builder()
        .queryEmbedding(embeddingModel.embed(question).content())
        .filter(and(
                metadataKey("tenantId").isEqualTo(currentTenant()),
                metadataKey("department").isEqualTo(userDept())))
        .maxResults(5)
        .minScore(0.7)
        .build();
```

| 过滤维度 | 元数据键 | 效果 |
|----------|----------|------|
| 多租户 | `tenantId` | 数据物理隔离 |
| 权限 | `department` | 只搜可见内容 |
| 时效 | `updatedAt` | 排除过期文档 |
| 来源 | `source` | 限定文档范围 |

过滤把"检索召回"与"权限控制"合二为一，比检索后再过滤更高效也更安全。

# 4. 文档切分

结论先行：切分（chunking）是 RAG 效果的最大变量。没有万能参数，必须用评测集驱动调优。

## 4.1 DocumentSplitter 接口

```java
public interface DocumentSplitter {
    List<TextSegment> split(Document document);
}
```

输入一个 `Document`，输出若干 `TextSegment`。每个片段会继承原文档元数据，可再补充 `index` 等片段级字段。

## 4.2 递归切分参数

```java
// 递归切分（推荐默认）
DocumentSplitter splitter = DocumentSplitters.recursive(
        800,      // 每段最大字符数
        100);     // 重叠字符数

List<TextSegment> segments = splitter.split(document);
```

递归切分先按段落分隔，段落仍过长则按句子，再按词，尽量在语义边界处断开。

### 4.2.1 切分参数说明

| 参数 | 含义 | 建议 |
|------|------|------|
| maxSegmentSize | 段大小 | 500~1000 字符 |
| maxOverlapSize | 重叠 | 10%~20% |
| separator | 分隔符 | 默认按段落→句子→词递归 |

### 4.2.2 重叠的意义

重叠（overlap）让相邻片段共享一段文本，避免答案恰好落在切分边界时两边都检索不到。重叠过大会导致存储与 Token 浪费，过小则起不到保护作用，10%~20% 是常见起点。

## 4.3 按文档结构切分

```java
// Markdown 按标题层级
DocumentByParagraphSplitter splitter =
        new DocumentByParagraphSplitter(1000, 100);

DocumentByLineSplitter lineSplitter =
        new DocumentByLineSplitter(1000, 100);

DocumentBySentenceSplitter sentenceSplitter =
        new DocumentBySentenceSplitter(500, 50);
```

结构感知切分尊重文档原本的语义单元，通常比固定长度效果更好，前提是文档格式规范。

## 4.4 切分策略对比

| 策略 | 优点 | 缺点 |
|------|------|------|
| 固定长度 | 简单可预测 | 可能截断语义 |
| 递归 | 尽量保持语义完整 | 段长不均 |
| 按结构 | 尊重文档结构 | 依赖格式规范 |
| 语义切分 | 语义最完整 | 计算成本高 |

> 分块是 RAG 效果的最大变量，详见 `5_RAG/DataProcessing/3_chunking_strategies.md`。

## 4.5 切分质量评测

切分效果无法凭直觉判断，应构建"问题 → 期望命中片段"的评测集，比较不同参数下的召回率。只有可量化的对比，才能选出适配业务语料的参数。

## 4.6 切分与 Token 预算

字符数不等于 Token 数。中文约 1 字符 ≈ 1 Token，英文约 4 字符 ≈ 1 Token，代码与符号密度更高。以字符数配置块大小时，应留出安全余量，确保"块 + Prompt + 输出"不超过模型上下文上限：

```
可用上下文 = 模型上限 - 系统提示 - 输出预留
块上限     = 可用上下文 / 召回块数
```

若模型上下文为 8192，系统提示与输出各预留 1000，召回 5 块，则单块不宜超过约 1200 Token。

## 4.7 切分常见反例

| 反例 | 后果 | 修正 |
|------|------|------|
| 按固定 200 字硬切 | 句子被腰斩 | 改用递归切分 |
| 重叠为 0 | 边界答案检索不到 | 加 10%~20% 重叠 |
| 块过大（>2000 字） | 噪声稀释语义 | 缩小块并增加召回数 |
| 块过小（<100 字） | 上下文不足 | 合并相邻小段 |

# 5. 完整 ETL 示例

结论先行：一个可复用的知识入库服务应把"选解析器、补元数据、切分、向量化入库"四步封装在一个方法内，并对异常与批量场景做统一处理。

## 5.1 服务实现

```java
@Service
@RequiredArgsConstructor
public class KnowledgeIngestService {

    private final EmbeddingStore<TextSegment> store;
    private final EmbeddingModel embeddingModel;

    public int ingest(Path file, String tenantId) {
        // 1. 选择解析器
        DocumentParser parser = resolveParser(file);

        // 2. 加载 + 补元数据
        Document document = FileSystemDocumentLoader.loadDocument(file, parser);
        document.metadata().put("source", file.getFileName().toString());
        document.metadata().put("tenantId", tenantId);

        // 3. 切分
        DocumentSplitter splitter = DocumentSplitters.recursive(800, 100);
        List<TextSegment> segments = splitter.split(document);

        // 4. 向量化 + 入库
        List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
        store.addAll(embeddings, segments);

        return segments.size();
    }

    private DocumentParser resolveParser(Path file) {
        String name = file.getFileName().toString().toLowerCase();
        if (name.endsWith(".pdf")) return new ApachePdfBoxDocumentParser();
        if (name.endsWith(".docx")) return new ApachePoiDocumentParser();
        if (name.endsWith(".md")) return new MarkdownDocumentParser();
        return new ApacheTikaDocumentParser();
    }
}
```

## 5.2 解析器路由

`resolveParser` 把"扩展名 → 解析器"的映射集中在一处，新增格式时只改这里。生产上建议把它做成策略表而非 if 链，便于配置化与单元测试。

## 5.3 批量导入与并发

批量导入时注意嵌入模型的批大小限制，并按批调用 `embedAll`，避免一次性提交过多片段导致内存峰值：

```java
public int ingestBatch(List<Path> files, String tenantId) {
    int total = 0;
    for (Path file : files) {
        try {
            total += ingest(file, tenantId);
        } catch (Exception e) {
            log.error("入库失败 file={}", file, e);
        }
    }
    return total;
}
```

## 5.4 幂等与事务

ETL 应设计为幂等：同一文件重复导入，结果一致。常用手段是"先按 `source` 删除，再写入"，并给每次导入打上批次号：

| 手段 | 说明 |
|------|------|
| 内容哈希 | 未变更则跳过 |
| 按来源删除 | 保证不残留旧片段 |
| 批次号 | 便于回滚与审计 |

注意向量库通常不支持跨集合事务，无法做到"原子替换"。因此生产更依赖"新集合构建 + 别名切换"（见 6.2）而非单次事务。

# 6. 增量更新

结论先行：文档更新时不能只追加新片段，否则旧片段会与新片段同时被检索到，产生矛盾答案。必须先失效旧版本。

## 6.1 删除后重写

```java
public void reindex(Path file, String tenantId) {
    String source = file.getFileName().toString();

    // 删除旧版本
    store.removeAll(store.search(EmbeddingSearchRequest.builder()
                    .queryEmbedding(embeddingModel.embed("").content())
                    .filter(metadataKey("source").isEqualTo(source))
                    .maxResults(10000)
                    .build())
            .matches().stream()
            .map(m -> m.embeddingId())
            .toList());

    // 重新写入
    ingest(file, tenantId);
}
```

这里用一个"空查询"配合元数据过滤来定位旧片段。注意空向量在部分向量库中不合法，实际实现应改用按元数据删除的专用 API（如 `removeAll(Filter)`），上述写法仅示意流程。

## 6.2 别名切换

> 生产更推荐"新表构建 + 别名切换"，避免删除期间查询不到数据。

做法是：把新版本写入一个新集合（collection/index），全部完成后把查询别名指向新集合，再删除旧集合。整个过程对在线检索无感知。

## 6.3 内容哈希去重

重复文件会导致重复片段，降低检索质量。入库前计算内容哈希，已存在则跳过：

```java
String hash = sha256(Files.readAllBytes(file));
if (store.existsByHash(hash)) {
    return 0;
}
```

## 6.4 删除策略

删除分为硬删除与软删除。硬删除直接从向量库移除；软删除只标记 `deletedAt`，检索时通过过滤器排除。软删除便于审计与误删恢复，代价是存储持续增长，需要定期清理。

| 策略 | 优点 | 缺点 |
|------|------|------|
| 硬删除 | 存储干净 | 不可恢复 |
| 软删除 | 可审计、可恢复 | 需定期回收 |
| 别名切换 | 在线无感 | 实现复杂 |

# 7. 生产注意事项

结论先行：文档处理的风险集中在资源（大文件、并发）、质量（扫描件、版面）与治理（去重、元数据）三类，需要分别设防。

## 7.1 资源与性能

| 风险 | 建议 |
|------|------|
| 大文件 OOM | 流式解析、限制单文件大小、分批 |
| 批量导入抖动 | 分批 + 限流 + 异步队列 |
| 解析耗时 | 解析与嵌入分离为独立任务 |

## 7.2 质量风险

| 风险 | 建议 |
|------|------|
| 扫描件无文本 | 需 OCR（Tika + OCR 或 Docling） |
| 表格/多栏错乱 | 用 Docling 等高质量解析器 |
| 乱码 | 统一 UTF-8 |
| 重复入库 | 内容哈希去重 |

## 7.3 运维治理

| 风险 | 建议 |
|------|------|
| 元数据缺失 | 入库前校验必填字段 |
| 全量重建抖动 | 别名切换 |
| 版本混乱 | 记录 `version` 与 `updatedAt` |

## 7.4 监控指标

文档处理虽是离线任务，也需要可观测：

| 指标 | 含义 |
|------|------|
| `ingest.files` | 处理文件数 |
| `ingest.segments` | 产出片段数 |
| `ingest.failures` | 解析失败数 |
| `ingest.duration` | 单文件耗时 |
| `ingest.empty` | 抽取文本为空的文件数 |

其中 `ingest.empty` 尤其重要：它往往意味着扫描件未做 OCR，是检索质量下降的早期信号。

# 8. 小结

- 解析与切分分离，按格式选 `DocumentParser`。
- Tika 万能兜底，Docling 适合高质量 PDF。
- 元数据在解析阶段就要补全。
- 切分策略需评测驱动，没有万能参数。
- 增量更新 + 别名切换是生产级索引维护方式。
