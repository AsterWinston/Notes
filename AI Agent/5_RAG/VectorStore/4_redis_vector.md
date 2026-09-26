# 1. 定位与适用场景

## 1.1 结论先行

Redis Stack（Redis 官方发行版）通过 RediSearch（Redis 的搜索模块）提供向量检索能力。它的核心价值不是"最强的向量库"，而是**与缓存共用一套基础设施**：如果你的系统已经跑着 Redis，那么引入向量检索几乎不需要新增组件、不需要新增运维面。一句话概括：**低延迟、易落地、TTL 友好，但受内存成本约束**。

## 1.2 优势拆解

| 优势 | 说明 |
|------|------|
| 内存级性能 | 亚毫秒~毫秒级延迟，数据常驻内存 |
| 复用现有 Redis | 无需新增组件与运维栈 |
| 支持 TTL | 数据可自动过期，天然适配会话类数据 |
| 混合检索 | 向量 + 全文 + 标签 + 数值可同查询组合 |
| 简单运维 | 团队通常已熟悉 Redis 运维 |
| 原子操作 | 借助 Redis 事务/Lua 可做简单一致性控制 |

## 1.3 劣势与边界

| 劣势 | 说明 |
|------|------|
| 内存成本高 | 每 1024 维 float32 向量约 4KB，规模上升成本显著 |
| 规模有限 | 不如 Milvus/Qdrant 等专用向量库 |
| 持久化风险 | 内存库的持久化语义弱于磁盘库，需评估丢数据窗口 |
| 索引能力有限 | 复杂聚合、多字段排序不如 ES/PG |
| 分片语义弱 | Redis Cluster 下跨槽查询与索引分布需额外设计 |

## 1.4 与其他向量库对比

| 维度 | Redis | Elasticsearch | PGVector | Milvus/Qdrant |
|------|-------|---------------|----------|---------------|
| 延迟 | 极低 | 低 | 中 | 低 |
| 成本 | 高（内存） | 中 | 低 | 中 |
| 规模 | 中小 | 中大 | 中大 | 超大 |
| 事务 | 弱 | 弱 | 强 | 弱 |
| 混合检索 | 强 | 很强 | 强 | 中 |
| TTL | 原生 | 无 | 需自建 | 无 |
| 运维 | 简单 | 复杂 | 简单 | 中等 |

> 选型经验：**热数据、会话记忆、低延迟**优先 Redis；**已有 ES/PG 且要强过滤/事务**优先对应库；**亿级规模**优先专用向量库。

# 2. 核心概念与数据模型

## 2.1 关键概念

| 概念 | 说明 |
|------|------|
| Index（索引） | 定义在 Hash 或 JSON 之上的搜索索引 |
| Hash/JSON | 向量与元数据的存储载体 |
| Vector Field（向量字段） | 存储向量的字段，类型为 FLOAT32/FLOAT64/BFLOAT16 |
| KNN | K 最近邻查询 |
| FT.SEARCH | 检索命令（全称 Full-Text Search） |
| FT.CREATE | 建索引命令 |
| Dialect（方言） | 查询语法版本，KNN 需 dialect 2 及以上 |

## 2.2 存储载体 Hash vs JSON

Redis 支持两种底层结构承载向量：**Hash（哈希）** 与 **JSON（RedisJSON）**。二者索引方式不同。

### 2.2.1 Hash 模式

Hash 每个字段是扁平键值，向量以二进制字符串存放。优点是内存紧凑、写入简单；缺点是嵌套结构需要手动拍平。

### 2.2.2 JSON 模式

JSON 允许嵌套路径（如 `$.embedding`），元数据组织更自然，但内存开销略高。索引时字段路径用 JSONPath 表达。

| 维度 | Hash | JSON |
|------|------|------|
| 内存 | 更省 | 略高 |
| 嵌套 | 不支持 | 支持 |
| 索引路径 | 字段名 | JSONPath |
| 适用 | 结构简单 | 元数据复杂 |

## 2.3 向量字段类型

| 类型 | 说明 | 内存 |
|------|------|------|
| FLOAT32 | 默认，精度高 | 4 字节/维 |
| FLOAT64 | 双精度 | 8 字节/维 |
| BFLOAT16 | 半精度，可省一半 | 2 字节/维 |

> 大多数 embedding 模型输出 float32，直接使用 FLOAT32 即可；内存吃紧时再考虑 BFLOAT16 量化。

# 3. 环境准备与依赖

## 3.1 部署 Redis Stack

向量检索需要 RediSearch 模块，**普通 Redis 镜像不含该模块**，必须使用 `redis/redis-stack` 镜像或自行加载模块。

```bash
docker run -d --name redis-stack \
  -p 6379:6379 -p 8001:8001 \
  redis/redis-stack:latest
```

> 8001 端口是 RedisInsight（可视化工具），可在其中查看索引与执行查询。

## 3.2 Maven 依赖

LangChain4j 风格：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-redis</artifactId>
</dependency>
```

Spring AI 风格：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-redis</artifactId>
</dependency>
```

底层客户端通常使用 Jedis（Redis 的 Java 客户端）：

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

## 3.3 连接配置

```java
public JedisPool jedisPool() {
    JedisPoolConfig config = new JedisPoolConfig();
    config.setMaxTotal(64);
    config.setMaxIdle(16);
    config.setTestOnBorrow(true);
    return new JedisPool(config, "localhost", 6379, 2000);
}
```

> 生产建议开启连接池并设置合理的 `maxTotal`，向量检索本身很快，瓶颈常在连接争用。

# 4. 索引定义

## 4.1 索引机制

RediSearch 的索引是**独立于数据的元结构**：先声明索引（前缀、字段、算法），之后写入匹配前缀的键会被自动纳入索引。索引不存数据，只存指向数据的倒排/图结构。删除数据后索引项也会移除，但**索引定义需显式删除**（`FT.DROPINDEX`）。

## 4.2 创建索引完整示例

```java
public void createIndex(JedisPool pool) {
    try (Jedis jedis = pool.getResource()) {
        Schema schema = new Schema()
                .addTextField("content", 1.0)
                .addVectorField("embedding",
                        Schema.VectorField.VectorAlgo.HNSW,
                        Map.of(
                                "TYPE", "FLOAT32",
                                "DIM", 1024,
                                "DISTANCE_METRIC", "COSINE",
                                "M", 16,
                                "EF_CONSTRUCTION", 200))
                .addTagField("tenantId")
                .addTagField("source");

        IndexDefinition definition = new IndexDefinition()
                .setPrefixes(new String[]{"doc:"});

        jedis.ftCreate("idx:knowledge",
                IndexOptions.defaultOptions().setDefinition(definition),
                schema);
    }
}
```

### 4.2.1 HNSW 参数

| 参数 | 说明 | 影响 |
|------|------|------|
| `M` | 每节点连接数 | 越大越准、内存越多 |
| `EF_CONSTRUCTION` | 建索引时的候选宽度 | 越大索引质量越高、越慢 |
| `DIM` | 向量维度 | 必须与模型一致 |
| `DISTANCE_METRIC` | 距离度量 | COSINE/L2/IP |

### 4.2.2 FLAT 参数

FLAT 为暴力检索（Brute Force），无近似，召回 100%，但耗时随数据量线性增长。

```java
.addVectorField("embedding",
        Schema.VectorField.VectorAlgo.FLAT,
        Map.of(
                "TYPE", "FLOAT32",
                "DIM", 1024,
                "DISTANCE_METRIC", "COSINE"))
```

> 小数据量（万级以内）用 FLAT 反而更快且更准；数据量上来后换 HNSW。

## 4.3 索引管理与查看

```bash
FT.INFO idx:knowledge          # 查看索引状态与文档数
FT.DROPINDEX idx:knowledge     # 删除索引（DD 保留数据）
```

> 使用 `FT.DROPINDEX ... DD` 会连同数据一起删除，慎用。

# 5. 数据写入

## 5.1 写入机制

写入即 `HSET` 一条 Hash，向量字段必须序列化为**二进制字节数组**（FLOAT32 小端序）。字段类型与索引声明必须一致，否则该文档不会被正确索引。

## 5.2 Hash 写入示例

```java
public void add(Jedis jedis, String id, float[] vector,
                String content, String tenantId, String source) {
    Map<String, Object> fields = new HashMap<>();
    fields.put("content", content);
    fields.put("embedding", floatArrayToBytes(vector));
    fields.put("tenantId", tenantId);
    fields.put("source", source);

    jedis.hset("doc:" + id, toStringMap(fields));
    jedis.expire("doc:" + id, 30 * 24 * 3600);   // TTL 30 天
}
```

向量转字节的工具方法（小端序）：

```java
public static byte[] floatArrayToBytes(float[] vector) {
    ByteBuffer buffer = ByteBuffer.allocate(vector.length * 4)
            .order(ByteOrder.LITTLE_ENDIAN);
    for (float v : vector) {
        buffer.putFloat(v);
    }
    return buffer.array();
}
```

## 5.3 JSON 写入示例

```java
public void addJson(JedisPool pool, String id, float[] vector, String content) {
    try (Jedis jedis = pool.getResource()) {
        Map<String, Object> doc = new HashMap<>();
        doc.put("content", content);
        doc.put("embedding", floatArrayToBytes(vector));
        jedis.jsonSet("doc:" + id, doc);
    }
}
```

> JSON 模式下索引字段路径需写成 JSONPath（如 `$.embedding`），创建索引时用 `addVectorField` 的对应重载。

## 5.4 批量写入

批量写入用 Pipeline（管道）减少往返：

```java
public void addBatch(JedisPool pool, List<Doc> docs) {
    try (Jedis jedis = pool.getResource()) {
        Pipeline pipeline = jedis.pipelined();
        for (Doc d : docs) {
            pipeline.hset("doc:" + d.id(), toStringMap(toFields(d)));
        }
        pipeline.sync();
    }
}
```

> 大 key 是隐患：单条 Hash 不要塞入过多字段，必要时拆分。

# 6. 检索

## 6.1 KNN 检索机制

RediSearch 的 KNN 语法形如 `FILTER =>[KNN k @field $param AS score]`。前面的部分是可选的过滤条件，`=>` 之后是向量检索。`score` 是距离（COSINE 下为余弦距离，越小越相似）。

## 6.2 基础 KNN 示例

```java
public List<Map<String, String>> search(Jedis jedis, float[] queryVector,
                                        String tenantId, int topK) {
    String query = "@tenantId:{%s}=>[KNN %d @embedding $vec AS score]"
            .formatted(tenantId, topK);

    return jedis.ftSearch("idx:knowledge", query,
            FTSearchParams.searchParams()
                    .addParam("vec", floatArrayToBytes(queryVector))
                    .dialect(2)
                    .returnFields("content", "source", "score")
                    .limit(0, topK));
}
```

## 6.3 混合查询

### 6.3.1 向量 + 标签

```text
@tenantId:{acme}=>[KNN 10 @embedding $vec AS score]
```

### 6.3.2 向量 + 全文

```text
(@content:(报销 流程))=>[KNN 10 @embedding $vec AS score]
```

### 6.3.3 向量 + 数值范围

```text
(@tenantId:{acme} @content:(报销 流程) @updatedAt:[1750000000 +inf])
=>[KNN 10 @embedding $vec AS score]
```

> Redis 支持在同一条查询中组合**向量 + 文本 + 标签 + 数值**过滤，这是它的亮点。

## 6.4 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 忘记 dialect 2 | KNN 语法需 dialect ≥ 2 | 显式设置 |
| 过滤字段未建 tag | 过滤不走索引，退化为全扫 | 加 `addTagField` |
| 距离与相似度混淆 | `score` 是距离 | 展示时做 `1 - score` 换算 |
| topK 过大 | 返回与排序开销上升 | 限制 topK |
| 维度不匹配 | 写入即失败或不入索引 | 与索引 DIM 对齐 |

# 7. 框架集成

## 7.1 Spring AI 集成

```java
@Bean
VectorStore vectorStore(EmbeddingModel embeddingModel, JedisPool pool) {
    return RedisVectorStore.builder(pool, embeddingModel)
            .indexName("idx:knowledge")
            .prefix("doc:")
            .metadataFields(
                    RedisVectorStore.MetadataField.tag("tenantId"),
                    RedisVectorStore.MetadataField.tag("source"))
            .initializeSchema(true)
            .build();
}
```

`initializeSchema(true)` 会在启动时自动建索引，省去手动 `FT.CREATE`。

## 7.2 Spring AI 使用

```java
@Service
public class KnowledgeService {

    private final VectorStore vectorStore;

    public KnowledgeService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public List<Document> search(String question, String tenantId) {
        SearchRequest request = SearchRequest.builder()
                .query(question)
                .topK(10)
                .filterExpression("tenantId == '" + tenantId + "'")
                .build();
        return vectorStore.similaritySearch(request);
    }
}
```

## 7.3 LangChain4j 集成

```java
EmbeddingStore<TextSegment> store = RedisEmbeddingStore.builder()
        .host("localhost")
        .port(6379)
        .indexName("idx:knowledge")
        .dimension(1024)
        .metadataKeys(Set.of("tenantId", "source"))
        .build();
```

> 框架封装会隐藏索引细节；出现"检索不到"时，先回到 `FT.INFO` 确认索引是否真正建成、字段是否匹配。

# 8. TTL 与生命周期

## 8.1 TTL 机制

Redis 的 TTL（Time To Live，存活时间）是独有优势：键到期自动删除，索引项随之移除。这让向量库可以承载**短生命周期数据**，无需外部清理任务。

## 8.2 TTL 使用示例

```java
// 会话记忆类数据：7 天过期
jedis.expire(key, 7 * 24 * 3600);

// 刷新式：每次访问续期
jedis.expire(key, 30 * 60);   // 30 分钟滑动过期
```

## 8.3 场景建议

| 场景 | TTL 建议 |
|------|----------|
| 对话记忆 | 7~30 天 |
| 临时文档 | 按业务 |
| 长期知识 | 不设 TTL 或用其他库 |
| 会话缓存 | 分钟级滑动过期 |

> 注意：TTL 到期删除是异步的，`FT.INFO` 的文档数可能有短暂延迟。

# 9. 内存优化

## 9.1 内存估算

```
1024 维 float32 = 4KB / 向量
100 万向量 ≈ 4GB（不含索引与元数据）
HNSW 图结构额外约 30%~50% 开销
```

> 实际规划时按"向量 + 索引 + 元数据 + 副本"叠加，通常要在纯向量体积上乘 1.5~2 倍余量。

## 9.2 量化与压缩

| 手段 | 说明 | 代价 |
|------|------|------|
| BFLOAT16/INT8 量化 | 降低单向量占用 | 精度下降 |
| 缩短向量维度 | 换小维度模型 | 语义表达力下降 |
| 减少冗余字段 | 只存必要元数据 | 功能受限 |

## 9.3 分片与冷热分离

| 手段 | 说明 |
|------|------|
| Redis Cluster | 水平扩展，注意跨槽查询 |
| 冷热分离 | 热数据在 Redis，冷数据在专用库 |
| 按租户分索引 | 大租户独立索引，便于清理 |

> 冷热分离的典型做法：写入时同时写 Redis（热）与专用库（冷），检索优先 Redis，miss 再查冷库。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 内存成本失控 | 评估规模，必要时换专用库 |
| 未建索引 | 检索前确保 `FT.CREATE` 已执行 |
| 过滤字段未建 tag | 加 `addTagField` |
| 大 key | 拆分、控制单条大小 |
| 持久化配置 | 开启 AOF/RDB，评估丢数据风险 |
| 单实例瓶颈 | Redis Cluster |

## 10.2 监控指标

| 指标 | 含义 | 关注点 |
|------|------|--------|
| `used_memory` | 已用内存 | 接近上限需扩容 |
| `evicted_keys` | 被淘汰键数 | 非 0 说明内存不足 |
| `keyspace_hits/misses` | 命中率 | 低命中需排查索引 |
| `latency` | 命令延迟 | 上升排查大 key/慢查询 |

## 10.3 高可用与持久化

以下是 `redis.conf` 的关键项：

```bash
appendonly yes
appendfsync everysec
maxmemory-policy noeviction   # 向量库不建议自动淘汰
```

> 向量库通常**不应使用 `allkeys-lru`** 之类的淘汰策略，否则会悄悄丢掉检索数据；建议 `noeviction` 并在内存告警时扩容。

# 11. 小结

## 11.1 关键结论

- Redis 向量检索的优势是**复用基础设施 + 低延迟 + TTL**。
- 支持向量、全文、标签、数值的混合查询，是它相对其他向量库的亮点。
- 内存成本是主要约束，适合中小规模与热数据。
- TTL 让它天然适合会话记忆等短生命周期数据。

## 11.2 选型建议

- 已用 Redis 且数据量在千万级以内：优先 Redis。
- 需要严格事务或复杂 SQL：改用 PGVector。
- 需要海量全文 + 向量混合：改用 Elasticsearch。
- 规模很大或要独立扩缩容：考虑"Redis 热 + 专用库冷"的分层架构。
