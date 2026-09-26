# 1. 索引流水线

**结论先行**：索引流水线把文件变成可检索的向量块，步骤为"解析 → 补元数据 → 切分 → 向量化 → 入库"，每步幂等，失败可重跑。

## 1.1 解析

### 1.1.1 解析器选择

| 文档类型 | 解析器 | 说明 |
|----------|--------|------|
| 纯文本/Markdown | Tika | 快速 |
| 简单 PDF | Tika | 直接抽取 |
| 复杂版面 PDF | Docling | 保留结构 |
| Word | Tika | 支持 .docx |
| 网页 | Tika | 去标签 |

### 1.1.2 解析实现

```java
Document document = FileSystemDocumentLoader.loadDocument(file, parser);
```

解析失败时降级为纯文本并标记待人工处理。

### 1.1.3 解析质量控制

- 校验正文字符数是否异常偏少。
- 检测乱码比例，超阈值告警。
- 记录解析器版本，便于回溯。

## 1.2 元数据补全

### 1.2.1 字段补全

```java
document.metadata().put("source", file.getFileName().toString());
document.metadata().put("tenantId", user.tenantId());
document.metadata().put("department", user.department());
document.metadata().put("updatedAt", Instant.now().toString());
```

元数据在切分前补全，确保每个块都携带权限与来源信息。

### 1.2.2 元数据校验

权限相关字段缺失时拒绝入库，避免产生"无主"块。

## 1.3 切分

### 1.3.1 递归切分

```java
DocumentSplitter splitter = DocumentSplitters.recursive(800, 100);
List<TextSegment> segments = splitter.split(document);
```

### 1.3.2 标题拼接

切分后把标题拼到块首，弥补上下文丢失，提升检索命中。

```java
String prefix = "《" + title + "》" + section + "：";
```

### 1.3.3 切分参数调优

| 参数 | 偏小 | 偏大 |
|------|------|------|
| chunkSize | 精度高、上下文碎 | 上下文全、噪声多 |
| overlap | 断句风险 | 冗余增加 |

## 1.4 向量化与入库

### 1.4.1 批量嵌入

```java
List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
```

批量嵌入减少调用次数，注意按服务上限分批。

### 1.4.2 事务写入

向量与元数据需同批写入，避免出现"有向量无元数据"的半成品。

### 1.4.3 失败重试

嵌入失败按指数退避重试，多次失败进入死信队列。

## 1.5 完整代码

```java
@Service
@RequiredArgsConstructor
public class IndexingService {

    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;
    private final DocumentParser parser;

    public int index(Path file, UserContext user) throws IOException {
        // ① 解析
        Document document = FileSystemDocumentLoader.loadDocument(file, parser);

        // ② 补元数据
        document.metadata().put("source", file.getFileName().toString());
        document.metadata().put("tenantId", user.tenantId());
        document.metadata().put("department", user.department());
        document.metadata().put("updatedAt", Instant.now().toString());

        // ③ 切分（递归 + 标题拼接）
        DocumentSplitter splitter = DocumentSplitters.recursive(800, 100);
        List<TextSegment> segments = splitter.split(document);

        // ④ 向量化 + 入库
        List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
        vectorStore.add(embeddings, segments);

        return segments.size();
    }
}
```

# 2. 混合检索

**结论先行**：混合检索同时跑稠密与稀疏两路召回，用 RRF 融合，再用重排模型精排，兼顾语义与关键词。

## 2.1 稠密检索

### 2.1.1 实现

```java
List<Document> dense = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query(question)
                .topK(20)
                .filterExpression(filter)
                .build());
```

### 2.1.2 过滤表达式

过滤表达式由服务端构造，包含租户、部门与 ACL 条件。

## 2.2 稀疏检索

### 2.2.1 BM25

稀疏检索基于 BM25，擅长精确匹配专有名词与编号。

```java
List<Document> sparse = fullTextSearch.search(question, filter, 20);
```

| 检索 | 擅长 | 短板 |
|------|------|------|
| 稠密 | 语义 | 专名 |
| 稀疏 | 关键词 | 同义 |

### 2.2.2 分词

中文需先分词（如 jieba/IK），分词质量直接影响 BM25 效果。

## 2.3 RRF 融合

### 2.3.1 实现

```text
score(d) = Σ 1 / (k + rank_i(d))     k 常取 60
```

RRF 只用排名不用分数，避免两路分数量纲不一致的问题。

### 2.3.2 去重

同一块在两路出现时按 id 合并，分数累加。

## 2.4 重排

### 2.4.1 重排模型

重排对候选做交叉编码，精度高于向量召回，但成本高，仅用于融合后的少量候选。

```java
return rerankModel.rerank(question, fused).stream()
        .limit(topN)
        .toList();
```

### 2.4.2 批处理

重排按批调用，控制单批候选数与超时。

## 2.5 完整代码

```java
@Service
@RequiredArgsConstructor
public class HybridRetriever {

    private final VectorStore vectorStore;
    private final FullTextSearch fullTextSearch;
    private final RerankModel rerankModel;

    public List<Document> retrieve(String question, UserContext user, int topN) {
        String filter = """
                tenantId == '%s' && (department == '%s' || acl == 'PUBLIC')
                """.formatted(user.tenantId(), user.department());

        // ① 稠密检索
        List<Document> dense = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(20)
                        .filterExpression(filter)
                        .build());

        // ② 稀疏检索
        List<Document> sparse = fullTextSearch.search(question, filter, 20);

        // ③ RRF 融合
        List<Document> fused = rrfFuse(dense, sparse);

        // ④ 重排
        return rerankModel.rerank(question, fused).stream()
                .limit(topN)
                .toList();
    }

    private List<Document> rrfFuse(List<Document> dense, List<Document> sparse) {
        Map<String, Double> scores = new HashMap<>();
        Map<String, Document> docMap = new HashMap<>();

        for (int i = 0; i < dense.size(); i++) {
            String id = idOf(dense.get(i));
            scores.merge(id, 1.0 / (60 + i + 1), Double::sum);
            docMap.put(id, dense.get(i));
        }
        for (int i = 0; i < sparse.size(); i++) {
            String id = idOf(sparse.get(i));
            scores.merge(id, 1.0 / (60 + i + 1), Double::sum);
            docMap.putIfAbsent(id, sparse.get(i));
        }
        return scores.entrySet().stream()
                .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                .map(e -> docMap.get(e.getKey()))
                .toList();
    }
}
```

# 3. 问答服务

**结论先行**：问答服务先检索、后组装上下文、再强约束生成，最后做引用校验；无依据时明确拒答。

## 3.1 检索与拒答

### 3.1.1 拒答条件

- 候选为空：直接拒答。
- 最高分低于阈值：提示资料不足。
- 引用校验失败：返回兜底话术。

```java
if (docs.isEmpty()) {
    return QaResult.refusal("未找到相关信息");
}
```

### 3.1.2 阈值设定

阈值需在评测集上标定，过高导致拒答多，过低导致噪声。

## 3.2 上下文组装

### 3.2.1 编号与来源

```java
private String buildContext(List<Document> docs) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < docs.size(); i++) {
        Document doc = docs.get(i);
        sb.append("[%d] %s\n（来源：%s / %s）\n\n".formatted(
                i + 1, doc.getText(),
                doc.getMetadata().get("source"),
                doc.getMetadata().get("section")));
    }
    return sb.toString();
}
```

### 3.2.2 上下文长度

按模型窗口裁剪，优先保留高分块，避免超限截断关键信息。

## 3.3 生成与约束

### 3.3.1 提示词

```java
String answer = chatClient.prompt()
        .system("""
                只依据【参考资料】回答，严格遵守：
                1. 不得使用外部知识；
                2. 每条结论标注来源编号 [n]；
                3. 资料不足时回答"资料中未提及"；
                4. 不得编造引用。
                """)
        .user("【参考资料开始】\n%s\n【参考资料结束】\n问题：%s"
                .formatted(context, question))
        .call().content();
```

### 3.3.2 温度设置

生成温度取低值（如 0.1），降低随机性与幻觉。

## 3.4 引用校验

### 3.4.1 校验实现

校验答案中的引用编号是否都在给定资料范围内，防止编造引用。

```java
if (!citationsValid(answer, docs)) {
    log.warn("引用校验失败: {}", question);
    answer = "抱歉，未能找到可靠依据回答该问题。";
}
```

### 3.4.2 校验算法

用正则提取 [n]，检查 n 是否落在 [1, docs.size()] 区间。

## 3.5 完整代码

```java
@Service
@RequiredArgsConstructor
public class KnowledgeQaService {

    private final HybridRetriever retriever;
    private final ChatClient chatClient;

    public QaResult ask(String question, UserContext user) {
        // ① 检索
        List<Document> docs = retriever.retrieve(question, user, 5);

        if (docs.isEmpty()) {
            return QaResult.refusal("未找到相关信息");
        }
        // ② 组装上下文（带编号与来源）
        String context = buildContext(docs);

        // ③ 生成（强约束）
        String answer = chatClient.prompt()
                .system("""
                        只依据【参考资料】回答，严格遵守：
                        1. 不得使用外部知识；
                        2. 每条结论标注来源编号 [n]；
                        3. 资料不足时回答"资料中未提及"；
                        4. 不得编造引用。
                        """)
                .user("【参考资料开始】\n%s\n【参考资料结束】\n问题：%s"
                        .formatted(context, question))
                .call().content();

        // ④ 引用校验
        if (!citationsValid(answer, docs)) {
            log.warn("引用校验失败: {}", question);
            answer = "抱歉，未能找到可靠依据回答该问题。";
        }
        return QaResult.ok(answer, toSources(docs));
    }

    private String buildContext(List<Document> docs) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < docs.size(); i++) {
            Document doc = docs.get(i);
            sb.append("[%d] %s\n（来源：%s / %s）\n\n".formatted(
                    i + 1, doc.getText(),
                    doc.getMetadata().get("source"),
                    doc.getMetadata().get("section")));
        }
        return sb.toString();
    }
}
```

# 4. 增量更新

**结论先行**：以内容哈希判断变更，未变则跳过，变更则先删旧块再重建，最后更新哈希记录。

## 4.1 变更检测

### 4.1.1 内容哈希

```java
String currentHash = sha256(Files.readAllBytes(file));
String storedHash = jdbc.queryForObject(
        "SELECT content_hash FROM document_meta WHERE source = ?",
        String.class, source);
```

### 4.1.2 哈希表

内容哈希记录在 document_meta 表，作为变更判断的唯一依据。

## 4.2 增删改

### 4.2.1 实现

```java
if (currentHash.equals(storedHash)) {
    return;                        // 未变更
}
// 删除旧块 + 重新索引
vectorStore.delete(FilterExpressionBuilder.b("source").eq(source).build());
indexingService.index(file, user);

jdbc.update("""
        INSERT INTO document_meta (source, content_hash, updated_at)
        VALUES (?, ?, now())
        ON CONFLICT (source) DO UPDATE
        SET content_hash = EXCLUDED.content_hash, updated_at = now()
        """, source, currentHash);
```

### 4.2.2 并发控制

同一 source 的更新加锁串行，避免并发残留旧块。

## 4.3 全量重建

### 4.3.1 别名切换

```text
新索引构建 → 别名切换 → 清理旧索引
```

切换原子，查询无抖动。

# 5. 难点与解决

**结论先行**：实现难点集中在解析、检索、权限与引用四类，均有明确解决手段。

## 5.1 解析难点

### 5.1.1 复杂 PDF

| 难点 | 解决 |
|------|------|
| 双栏排版 | Docling 解析器 |
| 表格丢失 | 转 Markdown/自然语言 |
| 扫描件 | OCR 兜底 |

### 5.1.2 质量校验

解析后做抽样人工校验，错误率超标则更换解析器。

## 5.2 检索难点

### 5.2.1 召回与精度

| 难点 | 解决 |
|------|------|
| 检索不准 | 混合检索 + 重排 |
| 长尾问题 | 查询改写 |
| 噪声块 | 分数阈值过滤 |

## 5.3 权限与引用难点

### 5.3.1 越权与引用

| 难点 | 解决 |
|------|------|
| 越权 | 元数据强制过滤 |
| 引用错误 | 引用校验 |
| 全量重建抖动 | 别名切换 |

# 6. 部署与运维

**结论先行**：部署需关注配置外部化、资源隔离与监控告警，保证服务稳定可运维。

## 6.1 部署配置

### 6.1.1 配置项

| 配置 | 说明 | 建议 |
|------|------|------|
| chunkSize | 块大小 | 800 |
| chunkOverlap | 重叠 | 100 |
| denseTopK | 稠密候选 | 20 |
| rerankTopN | 重排后 | 5 |
| minScore | 阈值 | 调优确定 |

```java
@ConfigurationProperties(prefix = "kb")
public record KbProperties(
        int chunkSize, int chunkOverlap,
        int denseTopK, int sparseTopK,
        int rerankTopN, double minScore) {
}
```

### 6.1.2 环境隔离

开发、测试、生产使用独立向量库与模型配额，避免相互影响。

## 6.2 监控与调优

### 6.2.1 指标

| 指标 | 说明 |
|------|------|
| 检索命中率 | 召回质量 |
| 引用校验失败率 | 幻觉信号 |
| P95 延迟 | 性能 |
| 拒答率 | 覆盖度 |

### 6.2.2 告警

延迟、失败率、拒答率超阈值时告警，便于快速定位。

调优顺序：先检索、再重排、后提示词。

# 7. 小结

**结论先行**：实现阶段完成了索引、检索、问答、增量更新与运维，形成可运行闭环。

## 7.1 关键结论

- 索引流水线：解析 → 补元数据 → 切分 → 向量化 → 入库。
- 混合检索：稠密 + 稀疏 + RRF + 重排。
- 问答用强约束提示 + 引用校验。
- 增量更新用内容哈希判断变更。
- 权限过滤在检索时强制注入。

## 7.2 下一步

接入评测集做回归，按检索、重排、提示词顺序持续调优。
