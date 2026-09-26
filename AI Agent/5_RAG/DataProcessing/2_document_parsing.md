# 1. 解析的目标

解析（Parsing）是 RAG 数据处理流水线的第二站，任务是把各种格式的文档转成**干净、结构化、可切分的纯文本**。解析质量直接决定检索质量的上限。

## 1.1 解析在 RAG 流水线中的位置

```
加载 Loading ──► 解析 Parsing ──► 清洗 Cleaning
                                   │
                                   ▼
              分块 Chunking ──► 向量化 Embedding ──► 存储 Store
```

加载负责"找到原料"，解析负责"把原料变成可读文本"，两者是独立的失败点：加载失败是文件找不到，解析失败是文本读不出。

## 1.2 解析输出应包含什么

合格的解析输出不只是纯文本，还应带上结构信息，供后续分块使用。

```
PDF/Word/Excel/PPT/HTML/Markdown
        │  Parser
        ▼
   纯文本 + 结构信息（标题、表格、页码）
```

| 结构信息 | 用途 |
|----------|------|
| 标题层级 | 按结构分块、上下文注入 |
| 页码 | 引用定位、溯源 |
| 表格边界 | 表格整体成块 |
| 段落边界 | 递归分块的切分点 |

## 1.3 边界与反例

- 反例一：把解析等同于"读出所有字符"，丢掉标题层级，后续只能做固定长度分块，效果差。
- 反例二：解析器抛出异常后被 `catch` 吞掉，文档静默丢失，检索时才发现缺内容。
- 边界：并非所有格式都能完美解析，扫描件、加密 PDF、损坏文件要有明确的降级路径。

> 解析质量直接决定 RAG 上限——**垃圾进，垃圾出**（Garbage In, Garbage Out）。

# 2. 各格式的解析难点

## 2.1 格式难点总览

| 格式 | 难点 |
|------|------|
| PDF（文本型） | 分栏、页眉页脚、阅读顺序 |
| PDF（扫描型） | 无文本层，需 OCR |
| Word | 表格、批注、样式 |
| Excel | 多 Sheet、合并单元格、公式 |
| PPT | 文本框顺序、图表 |
| HTML | 导航/广告噪声、正文提取 |
| Markdown | 相对简单，保留标题层级 |

## 2.2 PDF 的深层难点

### 2.2.1 阅读顺序与分栏

PDF 内部是按"绘制指令"存储的，并没有逻辑阅读顺序。双栏排版若按坐标从左到右读，会把两栏内容交错在一起，语义完全错乱。专业解析器会用版面分析（layout analysis）判断栏边界。

```
双栏 PDF 的坐标流：
[左栏行1] [右栏行1] [左栏行2] [右栏行2] ...
错误：左1 右1 左2 右2      正确：左1 左2 右1 右2
```

### 2.2.2 页眉页脚与脚注

每页重复的页眉、页脚、页码会进入正文，形成大量重复噪声。需要在解析后按"跨页重复"规则识别并剔除。

## 2.3 Office 与 HTML 的难点

- Word：批注、修订痕迹若被读入，会污染正文；表格跨页会拆散。
- Excel：公式单元格若只读缓存值会丢计算逻辑；合并单元格在转文本时易错位。
- PPT：文本框无固定顺序，需按位置（上→下、左→右）排序；图表需单独处理。
- HTML：正文与导航/广告混在一起，需正文提取算法。

## 2.4 边界与坑

- 坑一：用正则处理 HTML/PDF，遇到嵌套结构必然出错，应使用真正的解析库。
- 坑二：把页眉页脚当正文向量化，导致每块都含相同噪声，相似度计算失真。
- 边界：加密 PDF 需先解密，权限受限的文档应显式拒绝而非产出乱码。

## 2.5 格式探测与解析器路由

生产环境不能靠文件扩展名判断格式——扩展名可能被改错，也可能缺失。应使用内容探测（magic number）决定路由。

```java
// 伪代码：格式探测后路由到解析器
public Document parse(Resource resource) {
    String mime = tika.detect(resource);          // 基于内容的 MIME
    return switch (mime) {
        case "application/pdf" -> pdfParser.parse(resource);
        case "application/vnd.openxmlformats-officedocument"
             + ".wordprocessingml.document" -> wordParser.parse(resource);
        case "text/markdown" -> markdownParser.parse(resource);
        default -> tikaParser.parse(resource);    // 兜底
    };
}
```

| 探测依据 | 可靠性 | 说明 |
|----------|--------|------|
| 扩展名 | 低 | 可被篡改 |
| MIME 声明 | 中 | 客户端可能谎报 |
| Magic number | 高 | 读文件头字节 |

结论先行：**路由优先用 magic number，扩展名只作辅助**，并始终保留 Tika 兜底分支。

# 3. Spring AI 解析器

## 3.1 解析器清单

| 解析器 | 依赖 | 支持 |
|--------|------|------|
| `TextReader` | 核心 | 纯文本 |
| `JsonReader` | 核心 | JSON（JSON Pointer） |
| `PagePdfDocumentReader` | `spring-ai-pdf-document-reader` | PDF 按页 |
| `ParagraphPdfDocumentReader` | 同上 | PDF 按段落（需目录） |
| `MarkdownDocumentReader` | `spring-ai-markdown-document-reader` | Markdown |
| `TikaDocumentReader` | `spring-ai-tika-document-reader` | 万能 |

## 3.2 Tika 万能解析

### 3.2.1 Tika 原理

Apache Tika 通过"类型检测 + 对应解析器"自动选择解析逻辑，能处理数百种格式，是兜底方案的默认选择。它的优势是广覆盖，劣势是深结构（如表格、分栏）支持有限。

### 3.2.2 使用示例

```java
// Tika：一劳永逸
TikaDocumentReader reader = new TikaDocumentReader(resource);
List<Document> docs = reader.get();
```

## 3.3 PDF 专用解析器

`PagePdfDocumentReader` 按页产出文档，天然带页码元数据；`ParagraphPdfDocumentReader` 依赖 PDF 目录（bookmark）按段落切分，适合有目录的手册，但无目录的 PDF 会退化。

```java
PagePdfDocumentReader reader = new PagePdfDocumentReader(
        new ClassPathResource("/docs/handbook.pdf"),
        PdfDocumentReaderConfig.builder()
                .withPagesPerDocument(1)
                .build());
```

## 3.4 边界与坑

- 坑一：Tika 对扫描件返回空字符串而不报错，必须配合质量校验。
- 坑二：`ParagraphPdfDocumentReader` 在没有书签的 PDF 上可能只产出一个巨大文档。
- 边界：同一份文档用不同解析器结果差异很大，选型要拿真实文档对比。

## 3.5 解析器组合与降级链

生产解析应设计降级链：优先用质量最高的解析器，失败或质量不达标时逐级降级到兜底方案。

```
Docling（复杂版面）
   │ 失败/结果过短
   ▼
PDFBox / POI（格式专用）
   │ 失败/结果为空
   ▼
Tika（万能兜底）
   │ 结果为空且疑似扫描
   ▼
OCR（Tesseract / 云 OCR）
```

```java
// 伪代码：降级链
public String parseWithFallback(Resource r) {
    for (DocumentParser parser : List.of(docling, pdfBox, tika)) {
        try {
            String text = parser.parse(r);
            if (text != null && text.length() > 50) {
                return text;
            }
        } catch (Exception e) {
            log.warn("解析器 {} 失败: {}", parser.getClass().getSimpleName(), e.getMessage());
        }
    }
    return ocrIfScanned(r);   // 最后一层
}
```

边界：降级链会放大耗时，应对每级设置超时，避免一个坏文件拖垮整批任务。

# 4. LangChain4j 解析器

## 4.1 解析器清单

| 解析器 | 依赖 |
|--------|------|
| `TextDocumentParser` | 核心 |
| `ApachePdfBoxDocumentParser` | `langchain4j-document-parser-apache-pdfbox` |
| `ApachePoiDocumentParser` | `langchain4j-document-parser-apache-poi` |
| `ApacheTikaDocumentParser` | `langchain4j-document-parser-apache-tika` |
| `MarkdownDocumentParser` | `langchain4j-document-parser-markdown` |
| `DoclingDocumentParser` | `langchain4j-document-parser-docling` |

## 4.2 使用示例

```java
Document document = FileSystemDocumentLoader.loadDocument(
        path, new ApachePdfBoxDocumentParser());
```

解析器与加载器解耦，同一加载器可换不同解析器，便于做 A/B 对比。

## 4.3 与 Spring AI 对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 解析器与加载 | 绑定 | 分离 |
| Tika 支持 | `TikaDocumentReader` | `ApacheTikaDocumentParser` |
| 专业版面解析 | 依赖第三方 | `DoclingDocumentParser` 内置 |
| 结构信息保留 | 视解析器 | 视解析器 |

结论先行：**需要精细控制解析行为时，LangChain4j 的解析器分离设计更灵活。**

# 5. 表格处理

表格是 RAG 的难点——转成纯文本后行列关系丢失，检索到"500"却不知道它属于"住宿费"。

## 5.1 为什么表格难

表格的语义存在于"行标签 × 列标签 → 单元格"的三元关系里。纯文本线性化会丢失这层关系。解决思路是把关系显式写进文本。

## 5.2 转 Markdown

```java
// 保留表格结构
| 项目 | 标准 | 上限 |
|------|------|------|
| 住宿 | 500  | 800  |
| 餐饮 | 100  | 200  |
```

Markdown 保留了结构，但向量模型对表格文本的语义捕捉一般，检索命中率不稳定。

## 5.3 转自然语言

```java
// 把每行转成一句自然语言，便于语义检索
"住宿费标准为 500 元，上限 800 元。"
"餐饮费标准为 100 元，上限 200 元。"
```

> 对表格类知识，**转自然语言**往往比保留 Markdown 表更利于向量检索。

## 5.4 转结构化 JSON

若表格要参与计算或精确查询，可转成 JSON 并存元数据，检索时用过滤条件而非语义匹配。

```java
// 伪代码：表格 → 结构化记录
record Expense(String item, int standard, int limit) {}
List<Expense> rows = List.of(
        new Expense("住宿", 500, 800),
        new Expense("餐饮", 100, 200));
```

## 5.5 边界与坑

- 坑一：跨页表格被拆成两个表，表头丢失，需做表头继承。
- 坑二：合并单元格转换后错位，行列对不齐，需先展开合并。
- 边界：超宽表格转自然语言后句子过长，应拆成"每行一句"而非"整表一句"。

## 5.6 表格提取完整示例

以 Apache POI 读取 Word 表格为例，把表头与每行拼成自然语言句子。

```java
public List<String> extractTableSentences(XWPFTable table) {
    List<XWPFTableRow> rows = table.getRows();
    if (rows.isEmpty()) {
        return List.of();
    }

    List<String> headers = rows.get(0).getTableCells().stream()
            .map(XWPFTableCell::getText)
            .toList();

    List<String> sentences = new ArrayList<>();
    for (int i = 1; i < rows.size(); i++) {
        List<String> cells = rows.get(i).getTableCells().stream()
                .map(XWPFTableCell::getText)
                .toList();
        StringBuilder sb = new StringBuilder();
        for (int j = 0; j < cells.size() && j < headers.size(); j++) {
            if (!cells.get(j).isBlank()) {
                sb.append(headers.get(j)).append("为")
                  .append(cells.get(j)).append("；");
            }
        }
        if (!sb.isEmpty()) {
            sentences.add(sb.toString());
        }
    }
    return sentences;
}
```

输出形如："项目为住宿；标准为 500 元；上限为 800 元；"

边界：合并单元格在 POI 中会重复出现相同文本，需去重后再拼接，否则句子冗余。

# 6. OCR（扫描件）

## 6.1 何时需要 OCR

当 PDF 没有文本层（扫描生成），任何文本解析器都只能返回空。判断方法是检查解析结果长度是否接近 0。

```java
// 伪代码：判断是否需要 OCR
if (parsedText.isBlank() && pageCount > 0) {
    parsedText = ocrPdf(path);
}
```

## 6.2 OCR 流水线

### 6.2.1 渲染

```java
// 思路：PDF → 图片 → OCR → 文本
public String ocrPdf(Path pdf) {
    List<BufferedImage> pages = renderToImages(pdf);
    StringBuilder sb = new StringBuilder();
    for (BufferedImage page : pages) {
        sb.append(ocrEngine.recognize(page)).append("\n");
    }
    return sb.toString();
}
```

渲染 DPI 一般取 200~300，过低识别率差，过高耗时与内存激增。

### 6.2.2 识别与后处理

OCR 结果含错别字与断行，需后处理：合并被错误换行的句子、修正常见 OCR 混淆字（如"0/O""1/l"）。

## 6.3 方案对比

| 方案 | 说明 |
|------|------|
| Tesseract | 开源，中文需训练数据 |
| PaddleOCR | 中文效果好 |
| 云 OCR | 准确率高，按量计费 |
| 多模态模型 | 直接让视觉模型读图 |

## 6.4 边界与坑

- 坑一：对已是文本型的 PDF 也跑 OCR，浪费算力且可能引入识别错误。必须先检测。
- 坑二：手写体、印章、复杂表格 OCR 错误率高，需人工复核或换方案。
- 边界：OCR 应异步批处理，不阻塞主流水线。

# 7. 高质量解析：Docling 与版面分析

对复杂版面（多栏、表格、图文混排），专业解析器效果明显更好。

## 7.1 版面分析原理

版面分析模型先检测页面元素（标题、正文、表格、图片、页眉），再按人类阅读顺序重排，最后输出结构化文档（如 Markdown/JSON）。这解决了纯文本解析器无法处理的阅读顺序问题。

## 7.2 Docling 使用

```java
DoclingDocumentParser parser = DoclingDocumentParser.builder()
        .build();

Document document = FileSystemDocumentLoader.loadDocument(path, parser);
```

Docling 会把表格识别为结构化对象，可导出 Markdown 表或 JSON，是复杂 PDF 的优选。

## 7.3 多模态模型直读

视觉语言模型（VLM，Vision Language Model）可直接"看"页面图片并输出结构化文本，对图文混排、图表效果好，但成本高、有幻觉风险，适合少量高价值文档。

## 7.4 选型建议

| 文档类型 | 建议解析器 |
|----------|-----------|
| 纯文本/简单 PDF | Tika / PDFBox |
| 多栏/表格 PDF | Docling |
| 扫描件 | OCR 流水线 |
| 图文混排 | Docling 或 VLM |

> 版面复杂时，解析器质量差异极大，值得投入选型。

## 7.5 解析器选型实测方法

解析器选型不能只看宣传，要用真实文档做对比评测。

| 步骤 | 做法 |
|------|------|
| 取样 | 从业务文档中随机抽 30~50 份 |
| 跑批 | 每个候选解析器各跑一遍 |
| 打分 | 人工按"可读性/结构保留/表格正确率"打分 |
| 决策 | 综合质量与成本选型 |

| 解析器 | 可读性 | 结构保留 | 表格正确率 |
|--------|--------|----------|------------|
| Tika | 3/5 | 1/5 | 1/5 |
| PDFBox | 3/5 | 2/5 | 2/5 |
| Docling | 5/5 | 5/5 | 4/5 |

边界：评测样本要覆盖最差的一批文档（扫描、多栏、复杂表格），否则结论偏乐观。

# 8. 解析后的清洗

## 8.1 清洗目标

清洗只做"确定无信息量"的删除：页码、页脚、控制字符、重复空白。目标是降低噪声，而非改写内容。

## 8.2 清洗代码

```java
public String clean(String text) {
    return text
            .replaceAll("(?m)^第\\s*\\d+\\s*页.*$", "")      // 页码
            .replaceAll("(?m)^版权所有.*$", "")              // 页脚
            .replaceAll("[\\u0000-\\u001F]", "")             // 控制字符
            .replaceAll("\\s+", " ")                          // 多余空白
            .replaceAll("([，。！？])\\1+", "$1")             // 重复标点
            .trim();
}
```

## 8.3 清洗的边界（不要过度清洗）

- 反例：把所有换行都替换成空格，段落边界丢失，递归分块失效。
- 反例：删除所有数字或英文，导致关键信息（金额、型号）丢失。
- 边界：清洗规则应可配置、可回滚，并在清洗前后保留长度对比指标。

## 8.4 清洗规则的可配置实现

把清洗规则抽成配置，避免硬编码后难以调整。

```java
public record CleanRule(String name, Pattern pattern, String replacement) {

    public String apply(String text) {
        return pattern.matcher(text).replaceAll(replacement);
    }
}

public class TextCleaner {

    private final List<CleanRule> rules;

    public TextCleaner(List<CleanRule> rules) {
        this.rules = rules;
    }

    public String clean(String text) {
        String result = text;
        for (CleanRule rule : rules) {
            int before = result.length();
            result = rule.apply(result);
            log.debug("规则 {} 删除 {} 字符", rule.name(), before - result.length());
        }
        return result.trim();
    }
}
```

| 规则 | 正则 | 说明 |
|------|------|------|
| 页码 | `(?m)^第\s*\d+\s*页.*$` | 逐行匹配 |
| 页脚 | `(?m)^版权所有.*$` | 逐行匹配 |
| 控制字符 | `[\u0000-\u001F]` | 不可见字符 |

边界：正则逐条作用于全文，规则过多会变慢，可合并为一次遍历。

# 9. 解析质量检查

## 9.1 质量指标

| 指标 | 含义 | 异常信号 |
|------|------|----------|
| 文本长度 | 解析字符数 | 接近 0 |
| 中文占比 | 中文/总字符 | 明显偏低 |
| 可打印字符比 | 有效字符占比 | 出现乱码 |
| 重复率 | 行级重复比例 | 页眉页脚未清 |

## 9.2 校验代码

```java
public void validateParsed(Document doc) {
    String text = doc.getText();
    if (text.isBlank()) throw new ParseException("解析结果为空，可能是扫描件");
    if (text.length() < 50) throw new ParseException("解析结果过短");
    double cnRatio = countChinese(text) / (double) text.length();
    if (cnRatio < 0.3) log.warn("中文占比异常，可能乱码: {}", cnRatio);
}
```

## 9.3 失败处理

解析失败要分类：空结果（疑似扫描件，转 OCR）、乱码（编码问题，重解析）、过短（版面异常，人工复核）。分类后走不同降级路径，而不是一律丢弃。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 扫描件无文本 | 检测 + OCR 兜底 |
| 表格关系丢失 | 转 Markdown 或自然语言 |
| 页眉页脚噪声 | 清洗规则 |
| 乱码 | 统一 UTF-8 + 编码检测 |
| 解析静默失败 | 质量校验 + 告警 |
| 复杂版面错乱 | 用专业解析器（Docling） |
| 解析耗时 | 异步、批处理 |

## 10.2 性能与批处理

解析是 CPU 密集型操作，PDF 尤其慢。做法：解析任务进消息队列，多 worker 并行，设置超时与重试上限；对同一文件缓存解析结果（按内容哈希），避免重复解析。

## 10.3 可观测性

记录每种解析器的成功率、平均耗时、失败原因分布。当某类文档失败率突增时，能快速定位是解析器退化还是数据源变化。

## 10.4 成本与限流

OCR 与专业版面解析成本较高，需按价值分级处理。

| 文档价值 | 处理策略 |
|----------|----------|
| 高（合同、政策） | Docling / 云 OCR，人工抽检 |
| 中（手册、FAQ） | 格式专用解析器 |
| 低（日志、草稿） | Tika 快速解析 |

结论先行：**不要让所有文档都走最贵的解析路径**，按业务价值分流，并对云 OCR 设置并发与配额上限。

# 11. 小结

- 解析质量决定 RAG 上限，不能轻视。
- Tika 万能兜底，Docling 适合复杂版面，扫描件必须 OCR。
- 表格建议转自然语言描述，或转结构化 JSON 精确查询。
- 解析后要清洗（只删噪声，不过度改写）+ 质量校验 + 失败分类告警。
- 解析耗时，需异步批处理并缓存结果。
