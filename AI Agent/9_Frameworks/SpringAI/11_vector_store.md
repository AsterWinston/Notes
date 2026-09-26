# 1. VectorStore 抽象

**结论先行**：`VectorStore`（向量存储）是 Spring AI 对各家向量数据库的统一抽象。它屏蔽了底层实现差异，让业务代码只面向文本与元数据编程；换库通常只改依赖与配置。

## 1.1 接口与职责

### 1.1.1 核心方法

```java
public interface VectorStore extends DocumentWriter {
    void add(List<Document> documents);
    void delete(List<String> ids);
    void delete(Filter.Expression filterExpression);
    List<Document> similaritySearch(SearchRequest request);
    List<Document> similaritySearch(String query);
}
```

- `add`：写入文档（内部自动向量化）。
- `delete`：按 ID 或过滤条件删除。
- `similaritySearch`：相似度检索，返回命中文档。

### 1.1.2 Document 与 SearchRequest

| 概念 | 说明 |
|------|------|
| `Document` | 文本 + 元数据（+ 向量） |
| `SearchRequest` | 查询向量/文本、topK、相似度阈值、过滤表达式 |
| `EmbeddingModel` | 由 `VectorStore` 内部调用，把文本转向量 |

> Spring AI 的 `VectorStore` 会自动调用 `EmbeddingModel` 完成"文本 → 向量"的转换，你只需操作文本。

## 1.2 自动向量化机制

### 1.2.1 文本到向量的转换

```
add(text)                       similaritySearch(query)
   │                                     │
   ▼                                     ▼
EmbeddingModel.embed(text)     EmbeddingModel.embed(query)
   │                                     │
   ▼                                     ▼
 vector + metadata               query vector
   │                                     │
   ▼                                     ▼
 写入向量库  ◀──── 相似度计算（余弦/内积/欧氏）────▶  Top-K
```

关键点：**写入与查询必须使用同一个 Embedding 模型**，否则向量空间不一致，检索结果无意义。

### 1.2.2 关键概念表

| 术语 | 中文 | 说明 |
|------|------|------|
| Embedding | 嵌入/向量化 | 把文本映射为高维稠密向量 |
| Dimension | 维度 | 向量长度，如 1536、1024 |
| Distance | 距离 | 余弦距离、内积、欧氏距离 |
| Top-K | 前 K 个 | 返回相似度最高的 K 条 |

# 2. 支持的向量库

**结论先行**：没有"最好"的向量库，只有"最合适"的。选型应围绕数据规模、过滤需求、现有技术栈与运维成本四点决策。

## 2.1 主流向量库对比

| 向量库 | 特点 | 适用 |
|--------|------|------|
| PGVector | 基于 PostgreSQL，事务一致、运维简单 | 中小规模、已有 PG |
| Redis | 内存级性能、可共用缓存 | 高并发、低延迟 |
| Milvus | 专业向量库、水平扩展 | 大规模、十亿级 |
| Qdrant | 过滤能力强、Rust 实现 | 复杂过滤场景 |
| Elasticsearch | 向量 + 全文混合检索 | 已有 ES 生态 |
| Chroma | 轻量、易上手 | 原型验证 |
| Weaviate / Pinecone / OpenSearch | 各有侧重 | 按需 |

## 2.2 选型建议

- **原型/中小规模**：Chroma 或 PGVector，零额外运维。
- **已有 PostgreSQL**：优先 PGVector，复用事务与备份体系。
- **大规模（亿级以上）**：Milvus 或专用云服务。
- **强过滤 + 高并发**：Qdrant、Redis。
- **需要混合检索**：Elasticsearch / OpenSearch。

## 2.3 混合检索能力

混合检索（hybrid search）指同时使用**向量相似度**与**关键词匹配**：

| 检索方式 | 优势 | 劣势 |
|----------|------|------|
| 向量检索 | 语义泛化好 | 专有名词、数字易漏 |
| 关键词检索 | 精确匹配强 | 无法理解同义 |
| 混合检索 | 兼顾两者 | 需融合排序 |

## 2.4 依赖坐标速查

不同向量库对应不同的 starter，换库时替换依赖坐标即可：

| 向量库 | Starter 坐标后缀 |
|--------|------------------|
| PGVector | `spring-ai-starter-vector-store-pgvector` |
| Redis | `spring-ai-starter-vector-store-redis` |
| Milvus | `spring-ai-starter-vector-store-milvus` |
| Qdrant | `spring-ai-starter-vector-store-qdrant` |
| Elasticsearch | `spring-ai-starter-vector-store-elasticsearch` |
| Chroma | `spring-ai-starter-vector-store-chroma` |

> 坐标前缀统一为 `org.springframework.ai`。换库后只需调整配置项，业务代码保持不变，这正是抽象层的价值。

# 3. 使用示例（PGVector）

**结论先行**：PGVector 把向量当作 PostgreSQL 的一列，因此可复用现有数据库的事务、备份与权限体系，是最低门槛的生产方案。

## 3.1 依赖与配置

### 3.1.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### 3.1.2 配置项详解

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

| 配置项 | 含义 | 注意 |
|--------|------|------|
| `index-type` | 索引类型 | HNSW / IVFFlat / NONE |
| `distance-type` | 距离度量 | 需与模型匹配 |
| `dimensions` | 向量维度 | 必须与模型一致 |
| `initialize-schema` | 自动建表 | 生产建议关闭，用迁移脚本 |

## 3.2 写入与检索

### 3.2.1 Service 封装

```java
@Service
public class KnowledgeService {

    private final VectorStore vectorStore;

    public KnowledgeService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void save(String text, Map<String, Object> metadata) {
        vectorStore.add(List.of(new Document(text, metadata)));
    }

    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(query)
                        .topK(topK)
                        .similarityThreshold(0.6)
                        .build());
    }
}
```

### 3.2.2 批量写入

```java
// 批量写入优于逐条写入，减少网络往返与 Embedding 调用次数
public void saveBatch(List<Document> docs) {
    int batch = 100;
    for (int i = 0; i < docs.size(); i += batch) {
        List<Document> slice = docs.subList(i, Math.min(i + batch, docs.size()));
        vectorStore.add(slice);
    }
}
```

# 4. Document 与元数据设计

**结论先行**：元数据（metadata）是过滤、引用与治理的基础。它的字段设计必须在**建库前**确定，事后补充意味着全量重建索引。

## 4.1 Document 结构

```java
Document doc = new Document(
        "第三章 报销流程：员工需在费用发生后 30 天内提交申请……",
        Map.of(
                "source", "employee-handbook.pdf",
                "chapter", "第三章",
                "page", 12,
                "department", "HR",
                "tenantId", "acme",
                "updatedAt", "2026-01-15"
        )
);
```

## 4.2 元数据用途

- **过滤**：按租户、部门、时间、版本过滤；
- **引用**：回答时展示来源；
- **治理**：版本、失效、审计。

## 4.3 元数据设计原则

| 原则 | 说明 |
|------|------|
| 必填字段固定 | 如 `tenantId`、`source`、`updatedAt` |
| 类型一致 | 同一字段在所有文档中类型相同 |
| 可枚举优先 | 部门、状态用枚举值而非自由文本 |
| 预留版本 | `docVersion` 便于增量更新 |
| 控制基数 | 高基数（如 UUID）字段不宜做过滤 |

> 元数据字段应提前规划，**后补元数据成本极高**（需重建索引）。

# 5. SearchRequest 参数

**结论先行**：`SearchRequest` 决定"检索什么、检索多少、过滤哪些"。其中 `topK` 与 `similarityThreshold` 是一对需要联调的参数。

## 5.1 参数总览

| 参数 | 作用 | 调优建议 |
|------|------|----------|
| `query` | 查询文本 | — |
| `topK` | 返回条数 | 召回多（10~20）再重排 |
| `similarityThreshold` | 相似度下限 | 过低引入噪声，过高漏召回 |
| `filterExpression` | 元数据过滤 | 权限隔离必用 |

## 5.2 参数调优

```java
SearchRequest.builder()
        .query("差旅报销标准")
        .topK(20)
        .similarityThreshold(0.5)
        .filterExpression("tenantId == 'acme' && department == 'HR'")
        .build();
```

调优思路：

1. 先设 `topK=20`、`threshold=0.0`，观察命中分布；
2. 根据分数分布确定合理阈值；
3. 加入过滤条件后重新评估召回数量；
4. 用重排把 20 条精排成 5 条。

## 5.3 过滤表达式

```java
// 多条件组合
String expr = "tenantId == 'acme' && year >= 2024 && dept in ['HR','FIN']";
```

> 过滤字段必须已存在于元数据中，否则匹配为空。

# 6. 索引算法与调优

**结论先行**：索引算法决定"检索速度"与"召回精度"的平衡。小数据用暴力检索，大数据必须用近似最近邻（ANN，Approximate Nearest Neighbor）索引。

## 6.1 算法对比

| 算法 | 原理 | 特点 |
|------|------|------|
| Flat（暴力） | 逐一比较 | 精确但慢，适合小数据 |
| IVF | 聚类分桶 | 快，召回略降，需训练 |
| HNSW | 分层可导航小世界图 | 高召回高性能，内存占用高 |
| PQ | 乘积量化 | 压缩存储，精度损失 |

### 6.1.1 Flat 与 IVF

- **Flat**：无索引，查询时全量比较，复杂度 O(N)。数据量小于几万时完全够用。
- **IVF**（Inverted File Index，倒排文件索引）：先用 K-Means 把向量聚成若干桶，查询时只搜最近的若干个桶。`nlist`（桶数）与 `nprobe`（搜索桶数）是关键参数。

### 6.1.2 HNSW 与 PQ

- **HNSW**（Hierarchical Navigable Small World，分层可导航小世界图）：构建多层图，查询从上层快速逼近。召回与延迟表现最佳，代价是内存。
- **PQ**（Product Quantization，乘积量化）：把向量分段量化，大幅压缩存储，但损失精度。常与 IVF 组合为 IVF-PQ。

## 6.2 HNSW 参数

| 参数 | 含义 | 影响 |
|------|------|------|
| `m` | 每节点连接数 | 越大越准，内存越多 |
| `efConstruction` | 建索引搜索宽度 | 越大索引质量越好，建得越慢 |
| `efSearch` | 查询搜索宽度 | 越大越准，查询越慢 |

## 6.3 调优实战

```
调优顺序：
1. 先确定距离度量与维度（不可随意改）
2. 调 m（16~48）：影响内存与精度
3. 调 efConstruction（100~500）：影响建索引时间
4. 调 efSearch（64~256）：影响查询延迟与召回
5. 用评测集验证 Recall@K，找到延迟与精度的平衡点
```

> 详见 `5_RAG/VectorStore/7_index_algorithms.md`。

# 7. 删除与更新

**结论先行**：向量库通常不支持原地更新，"更新"等价于"删除旧数据 + 写入新数据"。生产环境应通过别名（alias）切换实现无中断重建。

## 7.1 删除 API

```java
// 按 ID 删除
vectorStore.delete(List.of("doc-1", "doc-2"));

// 按条件删除（如文档更新时清旧版本）
vectorStore.delete(FilterExpressionBuilder
        .b("source").eq("employee-handbook.pdf").build());
```

## 7.2 更新策略

### 7.2.1 删除后重写

```java
public void reindex(Resource resource, String source) {
    vectorStore.delete(FilterExpressionBuilder.b("source").eq(source).build());
    // 重新走 ETL 写入
    ingest(resource, source);
}
```

### 7.2.2 别名切换

```
建新索引 index_v2 ──▶ 灌数据 ──▶ 校验 ──▶ 别名 alias 指向 index_v2
                                              │
                                     旧索引 index_v1 保留一段时间后删除
```

## 7.3 边界与坑

- 删除是**异步/最终一致**的，删除后立即检索可能仍命中旧数据；
- 按条件删除在大数据量下可能很慢，注意分批；
- 别名切换需要底层向量库支持，不是所有实现都有。

> 向量库通常**不支持原地更新**，更新 = 删除 + 重新写入。批量重建时建议用别名（alias）切换，避免查询中断。

# 8. 多租户隔离

**结论先行**：多租户隔离的核心是"任何一次检索都必须带租户条件"。方案选择取决于隔离强度与运维成本的权衡。

## 8.1 方案对比

| 方案 | 隔离强度 | 运维成本 | 适用 |
|------|----------|----------|------|
| 同集合 + 元数据过滤 | 中 | 低 | 多数场景 |
| 每租户独立集合/表 | 高 | 中 | 大客户 |
| 独立实例 | 最高 | 最高 | 强合规 |

## 8.2 实现示例

```java
// 方案一：同一集合 + 元数据过滤（简单，需严格测试过滤条件）
filterExpression("tenantId == '" + tenantId + "'")

// 方案二：每租户独立集合/表（隔离强，运维成本高）
// 通过多数据源或动态表名路由
```

## 8.3 越权测试

```java
@Test
void crossTenantSearchIsBlocked() {
    ingest("acme-confidential", Map.of("tenantId", "acme"));
    List<Document> hits = searchAsTenant("beta", "confidential");
    assertTrue(hits.isEmpty());
}
```

> 生产建议：**默认按元数据过滤，超大租户可独立集合**。无论哪种，都要写"越权检索"测试用例。

# 9. 性能与成本

**结论先行**：向量库的性能瓶颈通常出现在**写入吞吐**与**查询延迟**两端，成本瓶颈则在**存储**与**Embedding 调用**。分别用批量、缓存、量化与降维来治理。

## 9.1 性能优化

| 关注点 | 手段 |
|--------|------|
| 写入吞吐 | 批量写入、异步、分批提交 |
| 查询延迟 | 缓存热点查询、HNSW 调参、减少维度 |
| 存储成本 | 量化（PQ/SQ）、降低维度、定期清理 |
| Embedding 成本 | 缓存向量、批量调用、本地模型 |

## 9.2 成本控制

- **向量缓存**：相同文本不重复调用 Embedding；
- **本地模型**：对延迟不敏感、数据敏感的场景用本地 Embedding；
- **量化压缩**：PQ/SQ 可把存储降低数倍；
- **生命周期**：过期文档定期清理，避免无限增长。

## 9.3 容量规划

| 维度 | 估算 |
|------|------|
| 存储 | 向量数 × 维度 × 4 字节 × 压缩比 |
| 内存 | HNSW 图结构常驻内存，约为原始向量数倍 |
| QPS | 由索引类型、`efSearch`、硬件共同决定 |

# 10. 生产注意事项

**结论先行**：向量库的"一次性决策"（维度、模型、索引）改动代价最高，必须在建库前想清楚；其余问题都能通过运维手段缓解。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 维度不匹配 | 建库前确定 Embedding 模型维度，换模型需重建 |
| 过滤条件注入 | 使用参数化/安全构造，勿字符串拼接用户输入 |
| 索引未建 | 大数据量前先建索引，否则查询极慢 |
| 无备份 | 向量数据也要备份与灾备 |
| 版本漂移 | Embedding 模型版本升级会导致语义空间变化，需整体重建 |

## 10.2 运维建议

- 监控查询 P99 延迟与召回率；
- 定期做数据一致性校验；
- 升级 Embedding 模型时走"双写 + 灰度切换"。

# 11. 小结

## 11.1 核心要点

- `VectorStore` 统一了各家向量库，换库只改配置。
- 元数据设计是过滤、引用与治理的基础。
- `topK` 召回 + 重排精排是标准组合。
- 维度、模型、索引是"一次性决策"，改动代价高，需提前规划。

## 11.2 实践清单

1. 建库前确定 Embedding 模型与维度。
2. 元数据字段提前规划，必填字段入库前校验。
3. 大数据量必须建 ANN 索引并做参数调优。
4. 多租户必须带租户过滤并写越权测试。
5. 更新用"删除 + 重写"，重建用别名切换。
