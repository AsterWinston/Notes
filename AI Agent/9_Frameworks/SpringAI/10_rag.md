# 1. RAG 在 Spring AI 中的定位

**结论先行**：RAG（Retrieval-Augmented Generation，检索增强生成）把"外部知识"注入大模型，让模型回答时"有据可依"。Spring AI 把 RAG 拆成**离线索引**与**在线查询**两条链路，分别由 ETL Pipeline（抽取-转换-加载流水线）与 `QuestionAnswerAdvisor`（问答顾问）承载。

## 1.1 RAG 的核心思想

### 1.1.1 检索增强的动机

大语言模型（LLM，Large Language Model）的知识止于训练数据截止时间，且无法访问企业私有数据。RAG 的思路是：**不把知识塞进模型参数，而是在提问时把相关资料检索出来，拼进提示词（prompt）**。

这样做的好处：

- 知识可实时更新，无需重新训练；
- 可溯源，回答能标注引用来源；
- 成本远低于微调（fine-tuning）；
- 权限可控，检索阶段就能做隔离。

### 1.1.2 与微调的区别

| 维度 | RAG | 微调（Fine-tuning） |
|------|-----|---------------------|
| 知识更新 | 改文档即可 | 需重新训练 |
| 成本 | 低（检索+推理） | 高（标注+训练） |
| 可溯源 | 强 | 弱 |
| 适合场景 | 事实问答、私有知识 | 风格迁移、能力对齐 |
| 数据隐私 | 文档留在本地 | 数据进入训练集 |

> 实践结论：**知识类问题优先用 RAG，行为/风格类问题才考虑微调**，两者也可叠加。

## 1.2 Spring AI 的 RAG 拆分

### 1.2.1 离线索引阶段

离线阶段负责把原始文档转成可检索的向量：

```
离线（索引阶段）                      在线（查询阶段）
─────────────────                    ─────────────────
文档 → Reader → Transformer          用户问题
     → EmbeddingModel                     │
     → VectorStore                        ▼
                                    QuestionAnswerAdvisor
                                          │  检索 Top-K
                                          ▼
                                    拼装上下文 → ChatModel → 回答
```

- **离线**：`ETL Pipeline`（见 `12_etl_pipeline.md`）。
- **在线**：`QuestionAnswerAdvisor` + `VectorStore`（本文件）。

### 1.2.2 在线查询阶段

在线阶段每次提问都会：

1. 把用户问题交给 `EmbeddingModel`（嵌入模型）转成查询向量；
2. 用查询向量在 `VectorStore`（向量存储）中做相似度检索，取 Top-K；
3. 把命中片段拼成上下文，与问题一起送给 `ChatModel`（对话模型）；
4. 模型基于上下文生成回答。

## 1.3 关键组件总览

| 组件 | 职责 | 所属阶段 |
|------|------|----------|
| `DocumentReader` | 读取文档为 `Document` | 离线 |
| `DocumentTransformer` | 分块、清洗、增强元数据 | 离线 |
| `VectorStore` | 存储向量并支持相似度检索 | 离线+在线 |
| `EmbeddingModel` | 文本转向量 | 离线+在线 |
| `QuestionAnswerAdvisor` | 自动检索并注入上下文 | 在线 |
| `ChatModel` / `ChatClient` | 生成最终回答 | 在线 |

# 2. 最小可用 RAG

**结论先行**：最小可用的 RAG 只需三样东西——一个向量库依赖、一个 `QuestionAnswerAdvisor`、一个带系统提示的 `ChatClient`。先跑通再优化。

## 2.1 依赖与配置

### 2.1.1 Maven 依赖

```xml
<!-- 向量库（以 pgvector 为例） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>

<!-- RAG Advisor -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
```

### 2.1.2 应用配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ai
    username: postgres
    password: ${DB_PASSWORD}
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
        initialize-schema: true
```

## 2.2 写入知识

### 2.2.1 Document 构造

`Document`（文档对象）= 文本 + 元数据（metadata）。元数据是可过滤、可引用的关键字段。

```java
Document doc = new Document(
        "员工需在费用发生后 30 天内提交报销申请。",
        Map.of("source", "employee-handbook.pdf", "department", "HR"));
```

### 2.2.2 批量写入

```java
@RestController
public class IngestController {

    private final VectorStore vectorStore;

    public IngestController(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    @PostMapping("/ingest")
    public String ingest(@RequestBody List<String> texts) {
        List<Document> docs = texts.stream()
                .map(t -> new Document(t, Map.of("source", "manual")))
                .toList();
        vectorStore.add(docs);
        return "已写入 " + docs.size() + " 条";
    }
}
```

## 2.3 检索问答

### 2.3.1 QuestionAnswerAdvisor 配置

```java
@Configuration
public class RagConfig {

    @Bean
    ChatClient ragChatClient(ChatModel chatModel, VectorStore vectorStore) {
        return ChatClient.builder(chatModel)
                .defaultSystem("""
                        你是企业知识库助手。只依据检索到的资料回答；
                        资料未提及时，明确说明"资料中未找到相关信息"。
                        """)
                .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                        .searchRequest(SearchRequest.builder()
                                .topK(5)
                                .similarityThreshold(0.6)
                                .build())
                        .build())
                .build();
    }
}
```

### 2.3.2 调用与结果

```java
String answer = ragChatClient.prompt()
        .user("报销流程是怎样的？")
        .call()
        .content();
```

# 3. 手动 RAG（可控性更高）

**结论先行**：Advisor 把检索封装成"黑盒"，一旦需要重排、过滤、多路融合或记录命中，就必须手动控制检索过程。手动 RAG 是生产系统的常态。

## 3.1 为什么需要手动控制

Advisor 适合快速原型，但生产系统常需要：

- 对检索结果做**重排（rerank）、过滤、去重**；
- 记录检索命中情况用于**评测**；
- 实现**多路检索融合**（向量 + 关键词）；
- 按 Token 预算动态裁剪上下文。

## 3.2 完整实现

### 3.2.1 检索与拼装

```java
public String askWithManualRag(String question) {
    // 1. 检索
    List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                    .query(question)
                    .topK(5)
                    .similarityThreshold(0.6)
                    .build());

    // 2. 拼装上下文
    String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n---\n"));

    // 3. 构造提示
    String prompt = """
            请仅依据下面的参考资料回答问题。若资料不足，请说明。
            参考资料：
            %s
            问题：%s
            """.formatted(context, question);

    return chatClient.prompt().user(prompt).call().content();
}
```

### 3.2.2 多路检索融合

```java
public List<Document> hybridSearch(String query) {
    // 向量召回
    List<Document> vectorHits = vectorStore.similaritySearch(
            SearchRequest.builder().query(query).topK(10).build());

    // 关键词召回（示意：由搜索引擎或 BM25 提供）
    List<Document> keywordHits = keywordSearcher.search(query, 10);

    // 融合去重（RRF，Reciprocal Rank Fusion，倒数排名融合）
    return rrfMerge(vectorHits, keywordHits, 5);
}
```

## 3.3 Advisor 与手动方式对比

| 维度 | QuestionAnswerAdvisor | 手动 RAG |
|------|-----------------------|----------|
| 接入成本 | 极低 | 中 |
| 可定制性 | 低 | 高 |
| 可评测性 | 弱（需额外埋点） | 强 |
| 适合阶段 | 原型、简单问答 | 生产、复杂流程 |

# 4. 带元数据过滤的检索

**结论先行**：元数据过滤（metadata filter）是"检索范围"的开关，也是多租户与权限隔离的技术基础。它与向量相似度共同决定最终召回。

## 4.1 过滤表达式语法

```java
SearchRequest request = SearchRequest.builder()
        .query("年假规定")
        .topK(5)
        .filterExpression("department == 'HR' && year >= 2024")
        .build();

List<Document> docs = vectorStore.similaritySearch(request);
```

常用运算符：

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `==` | 等于 | `tenantId == 'acme'` |
| `!=` | 不等于 | `status != 'archived'` |
| `>` `>=` `<` `<=` | 比较 | `year >= 2024` |
| `&&` | 逻辑与 | `a == 'x' && b > 1` |
| `\|\|` | 逻辑或 | `dept == 'HR' \|\| dept == 'FIN'` |
| `in` | 集合包含 | `dept in ['HR','FIN']` |

## 4.2 复杂过滤示例

```java
String expr = "tenantId == 'acme' && year >= 2024 && dept in ['HR', 'FIN']";

SearchRequest.builder()
        .query("差旅报销标准")
        .topK(20)
        .filterExpression(expr)
        .build();
```

## 4.3 边界与坑

- **过滤发生在检索时还是检索后**：部分向量库支持"预过滤"，部分只能"后过滤"。后过滤可能导致 Top-K 被大量过滤掉后**召回不足**，需要放大 `topK`。
- **字段类型**：数值字段写成字符串比较会失效或报错。
- **注入风险**：不要把用户输入直接拼进表达式字符串。

> 元数据过滤是**多租户/权限隔离**的关键：只检索当前用户有权访问的文档。

# 5. 多租户与权限（重要）

**结论先行**：生产 RAG 必须做租户隔离，否则会出现"检索到别的公司的文档"这类严重事故。隔离既要在写入时打标，也要在检索时强制过滤。

## 5.1 隔离方案对比

| 方案 | 隔离强度 | 运维成本 | 适用 |
|------|----------|----------|------|
| 同集合 + 元数据过滤 | 中 | 低 | 大多数场景 |
| 每租户独立集合/表 | 高 | 高 | 大客户、合规要求高 |
| 独立实例 | 最高 | 最高 | 强隔离/专有云 |

## 5.2 写入与检索实现

### 5.2.1 写入打标

```java
// 写入时打上租户与权限标签
Document doc = new Document(text, Map.of(
        "tenantId", tenantId,
        "department", dept,
        "acl", String.join(",", allowedRoles)
));
vectorStore.add(List.of(doc));
```

### 5.2.2 检索强制过滤

```java
SearchRequest.builder()
        .query(q)
        .filterExpression("tenantId == '" + tenantId + "'")
        .build();
```

> 更安全的做法是用 `FilterExpressionBuilder` 构造，避免字符串拼接。

## 5.3 越权测试

必须为隔离写专门的测试用例：

```java
@Test
void tenantCannotSeeOthersDocuments() {
    ingest("acme-secret", Map.of("tenantId", "acme"));
    List<Document> hits = searchAsTenant("beta", "secret");
    assertTrue(hits.isEmpty(), "beta 不应检索到 acme 的文档");
}
```

# 6. 查询增强（Query Transformation）

**结论先行**：用户问题往往口语化、信息不全，直接检索命中率低。查询增强在检索前对问题做改写、扩展或分解，显著提升召回覆盖率。

## 6.1 查询改写

把口语化问题改写成检索友好表述：

```
原问题：那个报销要多久来着？
改写后：员工费用报销的提交时限是多少天？
```

## 6.2 多查询扩展

```java
// 多查询扩展：先用 LLM 生成多个子查询，再分别检索合并
List<String> subQueries = queryExpander.expand(question);
Set<Document> merged = new LinkedHashSet<>();
for (String q : subQueries) {
    merged.addAll(vectorStore.similaritySearch(
            SearchRequest.builder().query(q).topK(3).build()));
}
```

## 6.3 HyDE 与查询分解

- **HyDE**（Hypothetical Document Embeddings，假设文档嵌入）：先让模型生成一段"假想答案"，再拿这段答案去检索。因为答案与文档的语义空间更接近，命中率更高。
- **查询分解**（Query Decomposition）：把"比较 A 和 B 的报销标准"拆成"A 的报销标准""B 的报销标准"两个子查询分别检索。

## 6.4 技术对比表

| 技术 | 作用 | 成本 |
|------|------|------|
| 查询改写 | 把口语化问题改成检索友好表述 | 1 次 LLM 调用 |
| 多查询扩展 | 多角度召回，提升覆盖率 | N 次检索 + 1 次 LLM |
| HyDE | 先生成假设答案再检索 | 1 次 LLM + 1 次检索 |
| 查询分解 | 复杂问题拆成多个子问题 | N 次检索 + 1 次 LLM |

> 详见 `5_RAG/Retrieval/5_query_transformation.md`。

# 7. 重排（Rerank）

**结论先行**：向量召回是"粗排"，速度快但精度有限；交叉编码模型（cross-encoder）精排能显著提升 Top 结果的相关性。"召回多、精排少"是标准两阶段架构。

## 7.1 召回-精排两阶段

```
查询 ──▶ 向量召回 Top-20（快，粗）
             │
             ▼
        交叉编码器重排（慢，准）
             │
             ▼
        取 Top-5 送入 LLM
```

## 7.2 实现示例

```java
List<Document> candidates = vectorStore.similaritySearch(
        SearchRequest.builder().query(q).topK(20).build());

List<Document> reranked = rerankModel.rerank(q, candidates)
        .stream().limit(5).toList();
```

## 7.3 边界与坑

- 重排模型通常**有最大输入长度**，过长片段会被截断；
- 重排增加延迟，需与召回数量权衡；
- 重排模型多为外部 API，注意网络与配额。

> 召回多、精排少，是"召回-精排"两阶段的标准做法。详见 `5_RAG/Retrieval/4_reranking.md`。

# 8. RAG 质量诊断

**结论先行**：RAG 出问题先分类——是"检索失败"还是"生成失败"。二者优化手段完全不同，盲目调提示词常常无效。

## 8.1 失败分类

```
用户问 → 检索 → 生成
          │       │
          │       └─ 生成失败：上下文有答案但模型答错
          └─ 检索失败：答案没被召回（分块/向量/查询问题）
```

## 8.2 排查流程

排查顺序：

1. 把命中的文档打印出来，看**答案是否在召回结果里**；
2. 若不在 → 优化分块、Embedding、查询改写、提高 topK；
3. 若在但答错 → 优化提示、换更强模型、加"引用原文"要求。

## 8.3 评测指标

| 指标 | 含义 | 关注阶段 |
|------|------|----------|
| 召回率（Recall@K） | 正确文档是否进入 Top-K | 检索 |
| 精确率（Precision@K） | Top-K 中相关文档占比 | 检索 |
| 忠实度（Faithfulness） | 回答是否忠于上下文 | 生成 |
| 答案相关性 | 回答是否切题 | 生成 |

> 详见 `5_RAG/Evaluation`。

# 9. 生产注意事项

**结论先行**：RAG 上生产的关键不是"能不能跑"，而是"上下文预算、延迟、权限、时效、评测"五个维度的持续治理。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 上下文超限 | 控制 topK 与片段长度，按 Token 预算裁剪 |
| 检索延迟 | 向量库选型、索引参数、加缓存 |
| 幻觉引用 | 要求模型标注来源，无法引用则拒答 |
| 权限越界 | 元数据强制过滤 + 租户隔离 |
| 知识过期 | 增量更新、文档版本与失效策略 |
| 无评测 | 建黄金集，度量召回率与忠实度 |

## 9.2 上下文预算控制

```java
// 按 Token 预算裁剪上下文（伪代码）
int budget = 3000;
StringBuilder ctx = new StringBuilder();
for (Document d : docs) {
    int cost = tokenCounter.count(d.getText());
    if (ctx.length() > 0 && used + cost > budget) break;
    ctx.append(d.getText()).append("\n---\n");
    used += cost;
}
```

## 9.3 可观测性

- 记录每次请求的**检索命中 ID 与分数**；
- 记录**上下文 Token 数**与**首字延迟**；
- 抽样做人工标注，持续更新评测集。

# 10. 小结

## 10.1 核心要点

- Spring AI 用 `QuestionAnswerAdvisor` 一行接入 RAG，但生产常需手动控制检索。
- 元数据过滤是权限隔离的关键。
- 检索质量决定 RAG 上限，重排决定精度。
- 先诊断"检索失败"还是"生成失败"，再对症优化。

## 10.2 实践清单

1. 先跑通最小可用 RAG，再逐步加过滤、重排、查询增强。
2. 元数据在写入阶段就规划好，事后补成本极高。
3. 任何多租户系统都要写越权检索测试。
4. 上线前建评测集，用数据驱动参数调整。
