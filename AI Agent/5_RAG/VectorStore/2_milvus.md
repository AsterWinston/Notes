# 1. 定位与架构

Milvus 是**专用分布式向量数据库**，面向十亿级向量规模。结论先行：当单机数据库（PGVector、Redis）在容量或并发上触顶时，Milvus 通过存算分离与水平分片，把向量检索做成可独立扩缩容的基础设施。

## 1.1 定位

| 定位项 | 说明 |
|--------|------|
| 目标规模 | 千万到十亿级向量 |
| 核心优势 | 分布式、存算分离、多索引支持 |
| 典型场景 | 大规模知识库、推荐、图像检索 |
| 部署形态 | 单机 / 集群 / 托管 |
| 开发语言 | Go + C++ |

Milvus 不是"更快的 PGVector"，而是为向量而生的独立系统。它牺牲了部署简单性，换取了规模与吞吐的上限。

## 1.2 架构分层

```
┌──────────────────────────────────────────┐
│              接入层（SDK / REST）          │
├──────────────────────────────────────────┤
│  协调层   RootCoord / QueryCoord / DataCoord│
├──────────────────────────────────────────┤
│  执行层   QueryNode（查询）/ DataNode（数据）│
├──────────────────────────────────────────┤
│  存储层   对象存储（MinIO/S3）+ 消息队列（Pulsar/Kafka）│
│           + 元数据存储（etcd）              │
└──────────────────────────────────────────┘
```

四层职责：

| 层 | 组件 | 职责 |
|----|------|------|
| 接入层 | SDK / REST / gRPC | 对外接口 |
| 协调层 | RootCoord / QueryCoord / DataCoord | 元数据、调度、负载均衡 |
| 执行层 | QueryNode / DataNode / IndexNode | 查询、写入、建索引 |
| 存储层 | 对象存储 / 消息队列 / etcd | 持久化、日志、元数据 |

## 1.3 部署模式

| 部署模式 | 说明 | 适用 |
|----------|------|------|
| Milvus Lite | 本地轻量（开发） | 笔记本、单元测试 |
| Standalone | 单机（小规模） | 百万级、开发测试 |
| Cluster | 分布式（生产） | 千万到亿级 |
| Zilliz Cloud | 托管服务 | 无运维团队 |

选型建议：开发用 Lite 或 Standalone，生产规模上亿用 Cluster，团队无运维能力考虑 Zilliz Cloud。

# 2. 核心概念

结论先行：Milvus 的模型可归纳为"集合 -> 分区 -> 实体 -> 字段"，理解这四层，API 就不再陌生。

## 2.1 概念总览

| 概念 | 说明 |
|------|------|
| Collection | 集合（类似表） |
| Partition | 分区（加速过滤） |
| Entity | 一条记录 |
| Field | 字段（向量 + 标量） |
| Index | 向量索引 |
| Metric Type | COSINE / IP / L2 |
| Segment | 存储与索引的最小单位 |
| Load | 把集合加载进内存以支持查询 |

## 2.2 概念之间的关系

```
Collection（集合）
   ├── Partition tenant_a（分区）
   │      └── Entity（实体：id + embedding + content + tenant_id）
   ├── Partition tenant_b
   └── Schema
          ├── Field: id        (VarChar, 主键)
          ├── Field: embedding (FloatVector, 1024 维)
          └── Field: content   (VarChar)
```

关键点：分区是集合内的逻辑子集，检索时指定分区可缩小搜索范围；Segment 是系统内部的物理分片，对使用者透明。

# 3. 依赖引入与环境准备

结论先行：Java 侧优先使用官方 SDK（`milvus-sdk-java`）或框架封装（LangChain4j / Spring AI）；开发期可用 Milvus Lite 免部署。

## 3.1 Maven 依赖

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-milvus</artifactId>
</dependency>
```

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-milvus</artifactId>
</dependency>
```

若直接使用官方 SDK：

```xml
<dependency>
    <groupId>io.milvus</groupId>
    <artifactId>milvus-sdk-java</artifactId>
    <version>2.4.x</version>
</dependency>
```

## 3.2 连接配置

```java
MilvusServiceClient milvusClient = new MilvusServiceClient(
        ConnectParam.newBuilder()
                .withHost("localhost")
                .withPort(19530)
                .build());
```

生产建议：连接串、用户名密码从配置中心注入，勿硬编码；为客户端设置超时与重试，避免网络抖动导致请求悬挂。

# 4. 定义集合（Schema）

结论先行：Schema 一旦创建就难以大改，维度、主键、过滤字段必须提前想清楚。维度不一致是最高频的线上事故。

## 4.1 字段设计原则

| 原则 | 说明 |
|------|------|
| 主键 | 用业务唯一 ID，避免重复写入产生脏数据 |
| 向量字段 | 维度必须与嵌入模型一致，全链路统一 |
| 过滤字段 | 需要过滤的字段单独建列并建标量索引 |
| 文本字段 | 设置足够 `maxLength`，但不宜过大 |
| 分区键 | 高频过滤维度可作为分区键 |

## 4.2 完整 Schema 示例

```java
public class MilvusConfig {

    public static final String COLLECTION = "knowledge_base";
    public static final String VECTOR_FIELD = "embedding";
    public static final int DIMENSION = 1024;

    public static CreateCollectionParam schema() {
        return CreateCollectionParam.newBuilder()
                .withCollectionName(COLLECTION)
                .withShardsNum(2)
                .addField(FieldType.newBuilder()
                        .withName("id")
                        .withDataType(DataType.VarChar)
                        .withMaxLength(64)
                        .withPrimaryKey(true)
                        .build())
                .addField(FieldType.newBuilder()
                        .withName(VECTOR_FIELD)
                        .withDataType(DataType.FloatVector)
                        .withDimension(DIMENSION)
                        .build())
                .addField(FieldType.newBuilder()
                        .withName("content")
                        .withDataType(DataType.VarChar)
                        .withMaxLength(8192)
                        .build())
                .addField(FieldType.newBuilder()
                        .withName("tenant_id")
                        .withDataType(DataType.VarChar)
                        .withMaxLength(64)
                        .build())
                .addField(FieldType.newBuilder()
                        .withName("source")
                        .withDataType(DataType.VarChar)
                        .withMaxLength(512)
                        .build())
                .build();
    }
}
```

## 4.3 动态字段与分区键

```java
// 开启动态字段：未在 Schema 中声明的字段可写入 $meta
CreateCollectionParam.newBuilder()
        .withCollectionName(COLLECTION)
        .enableDynamicField(true)
        .build();

// 以 tenant_id 作为分区键（自动分区，无需手动建分区）
CreateCollectionParam.newBuilder()
        .withCollectionName(COLLECTION)
        .addField(FieldType.newBuilder()
                .withName("tenant_id")
                .withDataType(DataType.VarChar)
                .withMaxLength(64)
                .withIsPartitionKey(true)
                .build())
        .build();
```

动态字段灵活但有代价：过滤性能不如显式字段，且类型不安全。生产环境建议对高频过滤字段显式声明。

## 4.4 创建与删除集合

```java
// 创建集合
milvusClient.createCollection(MilvusConfig.schema());

// 判断是否存在
boolean exists = milvusClient.hasCollection(HasCollectionParam.newBuilder()
        .withCollectionName(MilvusConfig.COLLECTION)
        .build()).getData();

// 删除集合（不可恢复，生产慎用）
milvusClient.dropCollection(DropCollectionParam.newBuilder()
        .withCollectionName(MilvusConfig.COLLECTION)
        .build());
```

删除集合会连同数据与索引一并移除，且通常不可回滚。生产环境应先用 `releaseCollection` 下线，确认无流量后再删除，并保留对象存储快照。

# 5. 创建索引

结论先行：向量索引决定检索性能，标量索引决定过滤性能，二者缺一不可；建完索引必须 `loadCollection` 才能查询。

## 5.1 向量索引

```java
// HNSW 索引
IndexParam indexParam = IndexParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withFieldName(VECTOR_FIELD)
        .withIndexType(IndexType.HNSW)
        .withMetricType(MetricType.COSINE)
        .withExtraParam("{\"M\":16,\"efConstruction\":200}")
        .build();

milvusClient.createIndex(indexParam);
```

## 5.2 标量索引

```java
// 标量字段索引（加速过滤）
milvusClient.createIndex(IndexParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withFieldName("tenant_id")
        .withIndexType(IndexType.INVERTED)
        .build());
```

| 标量索引类型 | 适用 |
|--------------|------|
| INVERTED | 等值、集合包含、部分范围 |
| STL_SORT | 数值范围过滤 |
| Trie | 前缀匹配 |
| BITMAP | 低基数字段 |

### 5.2.1 何时必须建标量索引

只要过滤表达式里出现某字段，就应为该字段建索引，否则 Milvus 可能退化为暴力过滤。多租户的 `tenant_id`、时序的 `date` 是最典型的必建字段。

### 5.2.2 标量索引的代价

标量索引会占用额外内存与写入开销：写入时需同步更新索引，字段越多、索引越复杂，写入越慢。因此只为真正需要过滤的字段建索引，避免"全字段建索引"。

## 5.3 加载与释放

```java
// 加载到内存
milvusClient.loadCollection(LoadCollectionParam.newBuilder()
        .withCollectionName(COLLECTION)
        .build());
```

不 `load` 就查询会直接报错；不再使用时 `releaseCollection` 释放内存。加载是"用内存换延迟"的显式操作，这是 Milvus 与 Qdrant 等库的重要差异。

# 6. 写入

结论先行：批量写入、主键幂等、控制并发，是写入链路的三个要点。

## 6.1 单条与批量写入

```java
// 单条写入
JsonObject row = new JsonObject();
row.addProperty("id", UUID.randomUUID().toString());
row.add("embedding", gson.toJsonTree(vector));
row.addProperty("content", text);
milvusClient.insert(InsertParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withRows(List.of(row))
        .build());

// 批量写入：一次 500~1000 条为宜
milvusClient.insert(InsertParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withRows(rows)
        .build());
```

## 6.2 完整示例

```java
List<JsonObject> rows = segments.stream().map(seg -> {
    JsonObject row = new JsonObject();
    row.addProperty("id", UUID.randomUUID().toString());
    row.add("embedding", gson.toJsonTree(embeddingModel.embed(seg.text())
            .content().vector()));
    row.addProperty("content", seg.text());
    row.addProperty("tenant_id", tenantId);
    row.addProperty("source", seg.metadata("source"));
    return row;
}).toList();

milvusClient.insert(InsertParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withRows(rows)
        .build());
```

## 6.3 写入一致性

| 操作 | 语义 |
|------|------|
| insert | 追加，主键重复会报错或覆盖（取决于配置） |
| upsert | 存在则更新，不存在则插入 |
| delete | 按主键或表达式删除 |
| flush | 强制落盘，代价高，勿频繁调用 |

写入后立即可查（一致性级别可调）。若发现"刚写入查不到"，先检查是否查询了未加载的分区，或一致性级别设置过高。

# 7. 检索（带过滤）

结论先行：检索 = 向量相似度 + 过滤表达式 + 检索参数；过滤字段必须有标量索引，否则会退化为全量扫描。

## 7.1 基础检索

```java
SearchParam searchParam = SearchParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withMetricType(MetricType.COSINE)
        .withTopK(10)
        .withVectors(List.of(queryVector))
        .withVectorFieldName(VECTOR_FIELD)
        .withExpr("tenant_id == 'acme' && source like 'handbook%'")
        .withParams("{\"ef\":128}")
        .build();

SearchResults results = milvusClient.search(searchParam);
```

## 7.2 过滤表达式

| 表达式 | 含义 |
|--------|------|
| `tenant_id == 'acme'` | 等值匹配 |
| `source like 'handbook%'` | 前缀匹配 |
| `score > 0.8` | 数值比较 |
| `id in [1, 2, 3]` | 集合包含 |
| `a && b` / `a \|\| b` | 逻辑与 / 或 |
| `not (status == 'DELETED')` | 逻辑非 |

## 7.3 检索参数

| 参数 | 说明 | 建议 |
|------|------|------|
| `topK` | 返回条数 | 召回阶段可放大到 50 |
| `ef` | HNSW 查询宽度 | 64~128 |
| `offset` | 分页偏移 | 大偏移性能差 |
| `nprobe` | IVF 探测桶数 | 与召回权衡 |

# 8. 分区（Partition）

结论先行：分区是把过滤"物理化"的手段，能显著加速按租户/日期的检索，但分区数量不宜过多。

## 8.1 分区与分区键

按租户/日期分区，可显著加速过滤检索：

```java
milvusClient.createPartition(CreatePartitionParam.newBuilder()
        .withCollectionName(COLLECTION)
        .withPartitionName("tenant_acme")
        .build());

// 检索时指定分区
SearchParam.newBuilder()
        .withPartitionNames(List.of("tenant_acme"))
        ...
```

## 8.2 分区键选择

| 分区键选择 | 适用 |
|------------|------|
| tenant_id | 多租户 |
| date | 时序数据 |
| category | 分类检索 |

注意：分区数过万会带来元数据开销；若租户数量极多，优先用"分区键（Partition Key）"自动分区，而非手动建分区。分区与过滤表达式可叠加使用，但分区本身不替代标量索引。

# 9. HNSW 参数调优

结论先行：调参的本质是在"召回率、延迟、内存"三者之间取舍，先满足召回，再压延迟，最后省内存。

## 9.1 参数说明

| 参数 | 作用 | 调大影响 |
|------|------|----------|
| `M` | 每节点连接数 | 更准、内存更多 |
| `efConstruction` | 建索引宽度 | 索引更好、建得更慢 |
| `ef`（检索） | 查询宽度 | 更准、更慢 |

建议起点：`M=16, efConstruction=200, ef=64~128`，再按召回率与延迟调整。

## 9.2 调优方法

1. 固定评测集与查询集，先测基线召回率。
2. 逐步调大 `ef`，直到召回达标，记录对应延迟。
3. 若内存吃紧，再考虑降 `M` 或启用量化。
4. 每次只改一个参数，避免无法归因。

## 9.3 量化与内存

| 量化方式 | 内存 | 召回影响 |
|----------|------|----------|
| 无 | 100% | 基准 |
| SQ8 | ~25% | 略降 |
| PQ | ~10% | 明显下降 |
| DiskANN | 磁盘为主 | 受磁盘 IO 影响 |

# 10. Spring AI 与 LangChain4j 集成

结论先行：框架封装屏蔽了 SDK 细节，业务代码只依赖统一接口；但过滤表达式语法仍因库而异。

## 10.1 Spring AI

```java
@Bean
VectorStore vectorStore(MilvusServiceClient client, EmbeddingModel model) {
    return MilvusVectorStore.builder(client, model)
            .collectionName("knowledge_base")
            .initializeSchema(true)
            .build();
}
```

```java
List<Document> docs = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query("如何申请年假")
                .topK(5)
                .filterExpression("tenant_id == 'acme'")
                .build());
```

## 10.2 LangChain4j

```java
EmbeddingStore<TextSegment> store = MilvusEmbeddingStore.builder()
        .host("localhost")
        .port(19530)
        .collectionName("knowledge_base")
        .dimension(1024)
        .metricType(MilvusMetricType.COSINE)
        .build();

store.addAll(embeddings, segments);
```

| 集成方式 | 优点 | 注意 |
|----------|------|------|
| Spring AI | 与 Spring 生态无缝 | 过滤语法需适配 |
| LangChain4j | 可脱离 Spring | 需显式向量化 |
| 官方 SDK | 功能最全 | 需自行封装 |

## 10.3 端到端流程

```
1. 创建集合（Schema：id + embedding + content + tenant_id）
2. 创建向量索引（HNSW, COSINE）
3. 为 tenant_id 创建 INVERTED 标量索引
4. loadCollection 加载到内存
5. 批量 insert 文档向量
6. search（带 tenant_id 过滤）返回 Top-K
7. 交给重排器与 LLM
```

上述流程每一步都不可省略；其中第 3、4 步最常被遗漏，分别导致过滤慢与查询失败。

# 11. 生产注意事项

结论先行：Milvus 的运维复杂度高于单机库，集群、备份、容量、版本四项必须提前规划。

## 11.1 运维风险

| 风险 | 建议 |
|------|------|
| 未 load 就查询 | 查询前 `loadCollection` |
| 标量字段无索引 | 为过滤字段建 INVERTED 索引 |
| 单机撑不住 | 用 Cluster 模式 |
| 无备份 | 使用对象存储快照 |
| 版本不兼容 | 锁定 SDK 与 server 版本 |
| 维度不匹配 | 建集合前确定维度 |

## 11.2 容量与扩容

| 维度 | 规划要点 |
|------|----------|
| 内存 | QueryNode 需容纳热数据与索引 |
| 磁盘 | 对象存储按原始数据 + 索引估算 |
| 分片 | ShardsNum 影响写入并行度 |
| 副本 | 副本数决定查询吞吐与可用性 |
| 扩容 | 增加 QueryNode / DataNode |

扩容时注意：加节点不会自动重新平衡所有数据，需关注负载是否均匀。生产环境建议启用监控（Prometheus + Grafana）观察各节点的内存与 QPS。

### 11.2.1 扩容策略

优先横向增加 QueryNode 提升查询吞吐，DataNode 提升写入吞吐；存储层通过对象存储天然可扩展。

### 11.2.2 负载均衡

扩容后关注各节点负载是否均匀，必要时触发数据再平衡；副本数增加可同时提升吞吐与可用性。

# 12. 小结

- Milvus 适合十亿级规模，架构分协调/执行/存储三层。
- 集合 = 表，分区 = 加速过滤的利器。
- HNSW 是主流索引，参数需按召回率与延迟调优。
- 过滤字段必须建标量索引，否则拖慢检索。
- 生产用 Cluster + 对象存储 + 消息队列。
- Schema 的维度与主键一旦确定就难以更改，务必提前设计。
