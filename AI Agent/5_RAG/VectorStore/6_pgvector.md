# 1. 定位与适用场景

## 1.1 结论先行

PGVector 是 PostgreSQL（开源关系型数据库）的向量扩展，核心价值是**无需新增组件、事务一致、SQL 生态完整**。它把向量当作一种原生列类型，因此向量与业务数据可以同库、同事务、同备份，这是专用向量库难以做到的。一句话概括：**千万级以内、要强一致、已有 PG 的场景首选**。

## 1.2 优势拆解

| 优势 | 说明 |
|------|------|
| 零新增组件 | 复用现有 PostgreSQL |
| 事务一致 | 向量与业务数据同库同事务 |
| SQL 过滤 | 完整的 WHERE 条件、JOIN、子查询 |
| 运维成熟 | 备份、监控、扩容都有现成方案 |
| 混合查询 | 向量 + 标量条件 + JOIN |
| 生态兼容 | ORM、迁移工具、BI 均可直接使用 |

## 1.3 劣势与边界

| 劣势 | 说明 |
|------|------|
| 规模上限 | 千万级较合适，亿级吃力 |
| 检索性能 | 不如专用向量库 |
| 索引内存 | HNSW 索引常驻内存，占用可观 |
| 高并发 | 大规模 KNN 会消耗大量 CPU/内存 |
| 扩展性 | 水平扩展依赖分库分表或分区 |

## 1.4 与其他向量库对比

| 维度 | PGVector | Redis | Elasticsearch | Milvus |
|------|----------|-------|---------------|--------|
| 事务 | 强 | 弱 | 弱 | 弱 |
| SQL 生态 | 完整 | 无 | 中 | 弱 |
| 向量性能 | 中 | 高 | 中 | 很高 |
| 规模 | 中大 | 中小 | 中大 | 超大 |
| 运维 | 简单 | 简单 | 复杂 | 中等 |
| 混合检索 | 强 | 强 | 很强 | 中 |

> 选型经验：**"向量只是业务数据的一个字段"**时，PGVector 的工程收益最大；纯检索服务且规模巨大时，专用向量库更合适。

# 2. 核心概念与安装

## 2.1 关键概念

| 概念 | 说明 |
|------|------|
| `vector` | 向量列类型，需指定维度 |
| 距离操作符 | `<->`、`<#>`、`<=>` |
| opclass | 索引操作符类，需与距离一致 |
| HNSW | 图索引，查询快、内存高 |
| IVFFlat | 倒排索引，内存低、需训练 |
| `ef_search` | HNSW 查询宽度 |

## 2.2 安装扩展

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

> 需 PostgreSQL 版本与扩展版本匹配；部分托管数据库需在控制台启用扩展。

## 2.3 部署方式

Docker 方式最快：

```bash
docker run -d --name pgvector \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

| 部署方式 | 说明 |
|----------|------|
| Docker | 快速本地验证 |
| 自建 | 需编译安装扩展 |
| 托管（RDS/云） | 多数已内置 pgvector，控制台开启 |

> 生产建议使用托管 PG 或成熟的容器化方案，避免扩展编译带来的升级困难。

# 3. 建表与索引

## 3.1 建表

```sql
CREATE TABLE knowledge (
    id          BIGSERIAL PRIMARY KEY,
    content     TEXT        NOT NULL,
    embedding   vector(1024) NOT NULL,
    tenant_id   VARCHAR(64) NOT NULL,
    source      VARCHAR(512),
    updated_at  TIMESTAMP   NOT NULL DEFAULT now()
);

-- HNSW 索引（余弦）
CREATE INDEX idx_knowledge_embedding ON knowledge
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 过滤字段索引
CREATE INDEX idx_knowledge_tenant ON knowledge (tenant_id);
```

> 维度在 `vector(N)` 中固定，建表后修改维度需要改列类型并重建索引，务必先确定模型。

## 3.2 距离操作符

| 距离操作符 | 含义 | 索引 opclass |
|------------|------|--------------|
| `<->` | L2 欧氏距离 | `vector_l2_ops` |
| `<#>` | 负内积 | `vector_ip_ops` |
| `<=>` | 余弦距离 | `vector_cosine_ops` |

### 3.2.1 相似度换算

`<=>` 返回的是**余弦距离**（0 表示完全相同），展示相似度需做 `1 - distance`。内积操作符 `<#>` 返回负内积（越小越相似），相似度需取负。

```sql
SELECT 1 - (embedding <=> $1::vector) AS cosine_similarity FROM knowledge;
```

## 3.3 索引类型

### 3.3.1 HNSW

HNSW（Hierarchical Navigable Small World）是图索引，查询快、召回高，但索引常驻内存。

```sql
CREATE INDEX ON knowledge
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);
```

### 3.3.2 IVFFlat

IVFFlat（Inverted File with Flat compression）是倒排索引，内存占用低，但需先有数据再建索引（训练聚类中心），召回略低。

```sql
CREATE INDEX ON knowledge
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

| 索引类型 | 适用 | 内存 | 召回 |
|----------|------|------|------|
| HNSW | 查询快、内存占用高（推荐） | 高 | 高 |
| IVFFlat | 内存低、需训练、召回略低 | 低 | 中 |

## 3.4 索引参数

| 参数 | 说明 | 影响 |
|------|------|------|
| `m` | 每节点连接数 | 越大越准、内存越多 |
| `ef_construction` | 建索引宽度 | 越大索引越好、越慢 |
| `lists` | IVFFlat 聚类数 | 越大越准、内存越多 |

> 经验值：`m` 取 16~64，`ef_construction` 取 64~200，`lists` 取 `rows/1000`（百万行约 1000）。

# 4. 依赖与配置

## 4.1 Maven 依赖

LangChain4j 风格：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-pgvector</artifactId>
</dependency>
```

Spring AI 风格：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
```

## 4.2 连接配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ai
    username: postgres
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
```

> 向量检索是 CPU/内存密集型操作，连接池不宜过大，避免把数据库 CPU 打满。

# 5. 数据写入

## 5.1 写入机制

写入时向量以字符串形式（如 `[0.1,0.2,...]`）传入并显式转型为 `vector`。若使用 JDBC 的 `PreparedStatement`，需用 `?::vector` 或 `CAST(? AS vector)` 转型。

## 5.2 单条写入

```sql
INSERT INTO knowledge (content, embedding, tenant_id, source)
VALUES (?, ?::vector, ?, ?);
```

```java
jdbcTemplate.update(
        "INSERT INTO knowledge (content, embedding, tenant_id, source) VALUES (?, ?::vector, ?, ?)",
        content,
        toVectorString(embedding),      // "[0.1,0.2,...]"
        tenantId,
        source);
```

## 5.3 批量写入

```java
jdbcTemplate.batchUpdate(
        "INSERT INTO knowledge (content, embedding, tenant_id, source) VALUES (?, ?::vector, ?, ?)",
        new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Doc d = docs.get(i);
                ps.setString(1, d.content());
                ps.setString(2, toVectorString(d.embedding()));
                ps.setString(3, d.tenantId());
                ps.setString(4, d.source());
            }

            @Override
            public int getBatchSize() {
                return docs.size();
            }
        });
```

> 大批量写入前可临时 `SET synchronous_commit = off` 提升吞吐，但要接受少量丢失风险。

## 5.4 向量字符串格式

```java
public static String toVectorString(float[] vector) {
    StringBuilder sb = new StringBuilder("[");
    for (int i = 0; i < vector.length; i++) {
        if (i > 0) sb.append(",");
        sb.append(vector[i]);
    }
    return sb.append("]").toString();
}
```

> 不要用默认的 `Arrays.toString`，它包含空格且不满足向量字面量格式。

# 6. 检索

## 6.1 检索机制

PGVector 的检索就是一条普通 SQL：`ORDER BY embedding <=> $1 LIMIT k`。当查询包含 `ORDER BY <distance> LIMIT` 且存在匹配 opclass 的索引时，规划器会走 ANN（Approximate Nearest Neighbor，近似最近邻）索引扫描。

## 6.2 带过滤检索

```sql
SELECT id, content, source,
       1 - (embedding <=> ?::vector) AS similarity
FROM knowledge
WHERE tenant_id = ?
  AND updated_at >= ?
ORDER BY embedding <=> ?::vector
LIMIT 10;
```

```java
List<Document> docs = jdbcTemplate.query(
        """
        SELECT content, source, 1 - (embedding <=> ?::vector) AS score
        FROM knowledge
        WHERE tenant_id = ?
        ORDER BY embedding <=> ?::vector
        LIMIT ?
        """,
        rowMapper,
        toVectorString(queryVector), tenantId,
        toVectorString(queryVector), topK);
```

> PGVector 天然支持**预过滤**（WHERE 在索引扫描时生效），多租户场景可靠。

## 6.3 预过滤与后过滤

| 方式 | 机制 | 风险 |
|------|------|------|
| 预过滤 | 先用 WHERE 缩小集合再 ANN | 过滤太严可能走全表扫 |
| 后过滤 | 先 ANN 再过滤 | 可能过滤到不足 k 条 |

### 6.3.1 规划器选择

PGVector 会根据过滤选择性估算选择策略。若过滤后剩余行数很少，规划器可能放弃索引改为顺序扫描——这时需要评估是否值得。

## 6.4 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 未建索引 | 全表扫描，慢 | 数据量上来前建 HNSW |
| 操作符与 opclass 不匹配 | 索引用不上 | 对齐 `<=>` 与 `vector_cosine_ops` |
| 维度不匹配 | 写入报错 | 建表前确定维度 |
| 过滤选择性差 | 规划器放弃索引 | 加过滤字段索引、评估 |
| 返回不足 k 条 | 过滤掉太多 | 提高候选或放宽过滤 |

# 7. 框架集成

## 7.1 Spring AI 集成

```java
@Bean
VectorStore vectorStore(JdbcTemplate jdbcTemplate, EmbeddingModel model) {
    return PgVectorStore.builder(jdbcTemplate, model)
            .dimensions(1024)
            .distanceType(PgVectorDistanceType.COSINE_DISTANCE)
            .indexType(PgVectorIndexType.HNSW)
            .initializeSchema(true)
            .schemaName("public")
            .vectorTableName("vector_store")
            .maxDocumentBatchSize(10000)
            .build();
}
```

| 配置项 | 说明 |
|--------|------|
| `dimensions` | 向量维度 |
| `distanceType` | 距离类型，需与索引一致 |
| `indexType` | HNSW / IVFFlat |
| `initializeSchema` | 自动建表与索引 |

## 7.2 LangChain4j 集成

```java
EmbeddingStore<TextSegment> store = PgVectorEmbeddingStore.builder()
        .host("localhost")
        .port(5432)
        .database("ai")
        .user("postgres")
        .password(System.getenv("DB_PASSWORD"))
        .table("embeddings")
        .dimension(1024)
        .createTable(true)
        .build();
```

## 7.3 使用示例

```java
@Service
public class KnowledgeService {

    private final VectorStore vectorStore;

    public KnowledgeService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public List<Document> search(String question, String tenantId) {
        return vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(10)
                        .filterExpression("tenant_id == '" + tenantId + "'")
                        .build());
    }
}
```

> 框架自动建的索引参数可能不是最优，生产环境建议手动建索引后再关闭 `initializeSchema`。

# 8. 索引调优

## 8.1 参数

| 参数 | 说明 | 影响 |
|------|------|------|
| `m` | 每节点连接数 | 越大越准、内存越多 |
| `ef_construction` | 建索引宽度 | 越大索引越好、越慢 |
| `hnsw.ef_search` | 查询宽度 | 越大越准、越慢 |

## 8.2 ef_search

`ef_search` 是**查询期**参数，可动态调整，是精度/延迟的主要旋钮。

```sql
SET hnsw.ef_search = 100;   -- 会话级
```

```sql
SET LOCAL hnsw.ef_search = 200;   -- 事务级
```

> 生产建议：`ef_search` 至少 ≥ 返回条数 k；一般取 40~200，越大越准但越慢。

## 8.3 索引类型选择

| 索引类型 | 适用 |
|----------|------|
| HNSW | 查询快、内存占用高（推荐） |
| IVFFlat | 内存低、需训练、召回略低 |

> IVFFlat 建索引前表里必须有数据（用于聚类），且建完索引后新增数据不会自动纳入新簇，需定期重建。

# 9. 事务一致性

## 9.1 机制

PGVector 是 PG 的扩展，向量列与其他列在同一张表、同一个事务中。这是它相对所有专用向量库的**独有优势**：业务数据与向量可以原子提交。

## 9.2 示例

```java
@Transactional
public void saveArticleWithEmbedding(Article article) {
    articleRepository.save(article);              // 业务数据
    vectorStore.add(List.of(new Document(...)));  // 向量数据
    // 同一事务，一起提交或一起回滚
}
```

## 9.3 注意事项

| 注意 | 说明 |
|------|------|
| 事务别太长 | 长事务阻塞 vacuum，影响索引 |
| 批量大小 | 大事务会撑大 WAL |
| 一致性边界 | 同一库内才有事务；跨库不行 |
| 索引更新 | 事务提交后索引才对外可见 |

> 需要"业务数据与向量严格一致"时，PGVector 是最佳选择。

# 10. 规模与性能

## 10.1 规模建议

| 规模 | 建议 |
|------|------|
| < 100 万 | PGVector 完全够用 |
| 100 万 ~ 1000 万 | PGVector + 分区 + HNSW |
| > 1000 万 | 考虑 Milvus / Qdrant |

## 10.2 优化手段

- 按 `tenant_id` 分区表；
- 只 SELECT 必要字段（避免回表）；
- 合理设置 `ef_search`；
- 读写分离（只读副本承载查询）；
- 建索引时调大 `maintenance_work_mem`。

```sql
SET maintenance_work_mem = '2GB';   -- 建索引会话
```

## 10.3 分区

```sql
CREATE TABLE knowledge_2026 PARTITION OF knowledge
FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

> 分区的好处是单分区索引更小、更容易命中内存；坏处是跨分区查询要合并结果。按租户或时间分区是常见做法。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 维度不匹配 | 建表前确定 |
| 未建索引 | 数据量上来前建 HNSW |
| 分区缺失 | 大表按租户/时间分区 |
| 内存不足 | `maintenance_work_mem` 调大以建索引 |
| 全表扫描 | 确认 `ORDER BY <=>` 走了索引 |
| 备份 | 纳入常规 PG 备份 |

## 11.2 监控

| 指标 | 含义 | 关注点 |
|------|------|--------|
| 查询延迟 | ANN 耗时 | `ef_search` 过大 |
| 索引大小 | HNSW 内存 | 接近 shared_buffers 上限 |
| `pg_stat_user_indexes` | 索引使用 | 是否真的走了索引 |
| 连接数 | 并发 | 连接池与 CPU 平衡 |

## 11.3 备份与迁移

```bash
pg_dump -Fc -d ai -f ai.dump        # 逻辑备份
pg_restore -d ai_new ai.dump        # 恢复
```

> 恢复后需确认扩展已安装（`CREATE EXTENSION vector`），并重建 HNSW 索引（索引不随逻辑备份导出，会随恢复重建）。

# 12. 小结

## 12.1 关键结论

- PGVector 的优势是**零新增组件 + 事务一致 + SQL 生态**。
- 支持预过滤，多租户可靠。
- HNSW 是推荐索引，`m`、`ef_construction`、`ef_search` 影响精度与性能。
- 千万级以内非常合适，更大规模考虑专用向量库。
- 需要"业务与向量强一致"时首选。

## 12.2 选型建议

- 已有 PG、数据量千万级以内：优先 PGVector。
- 需要海量全文 + 向量：考虑 Elasticsearch。
- 需要极低延迟热数据：考虑 Redis。
- 规模超亿或要独立扩缩容：考虑 Milvus/Qdrant。
