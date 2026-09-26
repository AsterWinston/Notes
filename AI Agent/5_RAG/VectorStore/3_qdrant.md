# 1. 定位与特点

Qdrant 是用 **Rust** 编写的向量数据库，以**过滤能力强**和**部署简单**著称。结论先行：当检索需要携带复杂业务条件（多租户、权限、时间范围、标签组合）时，Qdrant 的过滤表达力与预过滤语义使其成为首选。

## 1.1 定位

| 定位项 | 说明 |
|--------|------|
| 目标规模 | 百万到亿级向量 |
| 核心优势 | 强过滤、预过滤、部署简单 |
| 典型场景 | 多租户 SaaS、权限检索、推荐 |
| 部署形态 | 单二进制 / Docker / 集群 |
| 开发语言 | Rust |

Qdrant 的差异化不在"更大"，而在"更准"——它把过滤当作一等公民，而不是检索之后的补丁。

## 1.2 核心特点

| 特点 | 说明 |
|------|------|
| 强过滤 | 支持复杂条件、嵌套字段过滤 |
| 预过滤 | 过滤在检索前执行，结果可靠 |
| 部署简单 | 单二进制 / Docker |
| 混合检索 | 支持稠密 + 稀疏向量 |
| 量化 | 标量量化、乘积量化降内存 |
| 多向量 | 一条记录可有多个向量 |

## 1.3 与同类对比

| 维度 | Qdrant | Milvus | PGVector |
|------|--------|--------|----------|
| 过滤表达力 | 很强 | 强 | 强（SQL） |
| 部署复杂度 | 低 | 高 | 低 |
| 规模上限 | 大 | 超大 | 中 |
| 预过滤 | 原生 | 支持 | 依赖查询计划 |
| 内存优化 | 量化 + memmap | 量化 + DiskANN | 依赖 PG |

# 2. 核心概念

结论先行：Qdrant 的模型是"集合 -> 点 -> 向量 + payload"，payload 即元数据，也是过滤的作用对象。

## 2.1 概念总览

| 概念 | 说明 |
|------|------|
| Collection | 集合 |
| Point | 一条记录（id + vector + payload） |
| Payload | 元数据（可过滤） |
| Distance | Cosine / Dot / Euclid |
| Filter | 过滤条件 |
| Payload Index | 元数据字段索引（加速过滤） |
| Named Vector | 命名向量（一条记录多个向量） |

## 2.2 数据模型

```
Collection: knowledge_base
   └── Point(id=uuid)
          ├── vector: [0.12, -0.03, ...]   (1024 维)
          └── payload:
                 ├── content   = "..."
                 ├── tenantId  = "acme"
                 ├── source    = "handbook"
                 └── updatedAt = "2026-05-01"
```

payload 是 JSON 结构，支持嵌套对象；过滤针对 payload 字段，字段类型包括 keyword、integer、float、bool、datetime 等。

## 2.3 与 Milvus 概念对照

| Qdrant | Milvus | 说明 |
|--------|--------|------|
| Collection | Collection | 顶层容器 |
| Point | Entity | 一条记录 |
| Payload | Scalar Field | 元数据 |
| Named Vector | Vector Field | 向量字段 |
| Filter | Expr | 过滤条件 |
| Payload Index | Scalar Index | 过滤加速 |
| Distance | Metric Type | 距离类型 |

# 3. 依赖引入与连接

结论先行：Qdrant 提供 gRPC 与 REST 两种接口，Java 侧优先用 gRPC 客户端，性能更好。

## 3.1 Maven 依赖

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-qdrant</artifactId>
</dependency>
```

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-qdrant</artifactId>
</dependency>
```

官方客户端：

```xml
<dependency>
    <groupId>io.qdrant</groupId>
    <artifactId>client</artifactId>
    <version>1.9.x</version>
</dependency>
```

## 3.2 连接方式

```java
// gRPC（推荐，默认端口 6334）
QdrantClient client = new QdrantClient(
        QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

// REST（默认端口 6333），调试方便
QdrantClient restClient = new QdrantClient(
        QdrantClientConfig.setHost("localhost").setPort(6333));
```

生产建议：使用 API Key 认证，开启 TLS；连接池与超时按 QPS 配置，避免连接泄漏。

# 4. 连接与建集合

结论先行：创建集合必须指定向量维度与距离类型，二者与嵌入模型绑定；多向量集合需显式声明命名向量。

## 4.1 创建集合

```java
QdrantClient client = new QdrantClient(
        QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.createCollectionAsync("knowledge_base",
        Collections.VectorParams.newBuilder()
                .setSize(1024)
                .setDistance(Collections.Distance.Cosine)
                .build()).get();
```

## 4.2 集合参数

| 参数 | 说明 | 建议 |
|------|------|------|
| `size` | 向量维度 | 与嵌入模型一致 |
| `distance` | 距离类型 | 归一化向量用 Cosine |
| `hnsw_config` | HNSW 参数 | `m`、`ef_construct` |
| `optimizers_config` | 优化器 | 控制索引构建时机 |
| `quantization_config` | 量化 | 内存优化 |
| `on_disk` | 是否落盘 | 大规模省内存 |

```java
// 带 HNSW 参数的集合
Collections.VectorParams.newBuilder()
        .setSize(1024)
        .setDistance(Collections.Distance.Cosine)
        .setHnswConfigDiff(HnswConfigDiff.newBuilder()
                .setM(16)
                .setEfConstruct(200)
                .build())
        .build();
```

## 4.3 多向量与稀疏向量

```java
// 命名向量：一条记录可有 dense 与 sparse 两个向量
Map<String, Collections.VectorParams> vectors = Map.of(
        "dense", Collections.VectorParams.newBuilder()
                .setSize(1024)
                .setDistance(Collections.Distance.Cosine)
                .build());

Map<String, Collections.SparseVectorParams> sparseVectors = Map.of(
        "sparse", Collections.SparseVectorParams.newBuilder().build());
```

命名向量是混合检索的基础，写入与查询都要指定向量名。

## 4.4 集合别名与迁移

```java
// 创建别名指向当前集合
client.createAliasAsync("kb_current", "knowledge_base_v1").get();

// 迁移：新建 v2，回填数据后原子切换别名
client.createCollectionAsync("knowledge_base_v2", vectorParams).get();
client.updateAliasesAsync(UpdateAliases.newBuilder()
        .addActions(AliasOperations.newBuilder()
                .setDeleteAlias(DeleteAlias.newBuilder().setAliasName("kb_current"))
                .build())
        .addActions(AliasOperations.newBuilder()
                .setCreateAlias(CreateAlias.newBuilder()
                        .setAliasName("kb_current")
                        .setCollectionName("knowledge_base_v2"))
                .build())
        .build()).get();
```

别名让"换索引参数/换嵌入模型"可以做到原子切换，业务代码始终读写别名，无需停机。

# 5. 写入

结论先行：Qdrant 用 `upsert` 表达"写入或更新"，点 ID 幂等；payload 可增量覆盖。

## 5.1 Point 结构

```java
PointStruct point = PointStruct.newBuilder()
        .setId(PointId.newBuilder().setUuid(id).build())
        .setVectors(Vectors.newBuilder()
                .setVector(Vector.newBuilder()
                        .addAllData(toList(embedding))
                        .build())
                .build())
        .putAllPayload(payload)
        .build();
```

## 5.2 批量 upsert

```java
List<PointStruct> points = segments.stream().map(seg -> {
    Map<String, Value> payload = Map.of(
            "content", Value.newBuilder().setStringValue(seg.text()).build(),
            "tenantId", Value.newBuilder().setStringValue(tenantId).build(),
            "source", Value.newBuilder().setStringValue(source).build(),
            "updatedAt", Value.newBuilder().setStringValue(updatedAt).build());

    return PointStruct.newBuilder()
            .setId(id(seg))
            .setVectors(Vectors.newBuilder()
                    .setVector(Vector.newBuilder()
                            .addAllData(toList(embedding))
                            .build())
                    .build())
            .putAllPayload(payload)
            .build();
}).toList();

client.upsertAsync("knowledge_base",
        PointsList.newBuilder().addAllPoints(points).build()).get();
```

## 5.3 更新与删除

| 操作 | API | 说明 |
|------|-----|------|
| 全量覆盖 | `upsert` | 相同 ID 覆盖 |
| 局部更新 | `setPayload` | 只改部分 payload 字段 |
| 按 ID 删除 | `delete` | 指定 point ids |
| 按条件删除 | `delete` + Filter | 按 payload 条件批量删 |

按条件删除是 Qdrant 的强项，可用于"下架某来源的全部文档"这类运营操作。

## 5.4 写入建议

| 建议 | 说明 |
|------|------|
| 批量 upsert | 每批 100~500 条，减少往返 |
| 等待确认 | 用 `wait=true` 保证写后可查 |
| ID 幂等 | 用业务唯一 ID，避免重复点 |
| 控制并发 | 并发过高会拖慢索引构建 |

写入后若立即检索，务必开启等待确认（`wait`），否则可能因索引尚未构建而查不到刚写入的点。

# 6. 检索（带复杂过滤）

结论先行：Qdrant 的过滤在检索前执行（预过滤），因此即使过滤后子集很小，也能稳定返回 Top-K。

## 6.1 基础检索

```java
Filter filter = Filter.newBuilder()
        .addMust(matchKeyword("tenantId", tenantId))
        .addMust(matchKeyword("source", source))
        .addMust(rangeGte("updatedAt", "2026-01-01"))
        .build();

List<ScoredPoint> results = client.searchAsync(
        SearchPoints.newBuilder()
                .setCollectionName("knowledge_base")
                .addAllVector(toList(queryVector))
                .setLimit(10)
                .setFilter(filter)
                .setWithPayload(WithPayloadSelector.newBuilder()
                        .setEnable(true).build())
                .build()).get();
```

## 6.2 过滤条件类型

| 条件 | 构造 |
|------|------|
| 精确匹配 | `matchKeyword("field", value)` |
| 数值范围 | `rangeGte("price", 100)` |
| 集合包含 | `matchAny("tag", List.of("a","b"))` |
| 存在性 | `isNull` / `isNotNull` |
| 逻辑组合 | `must` / `should` / `mustNot` |

### 6.2.1 标量条件

等值、范围、集合包含等单字段条件，是过滤的基础构件。

### 6.2.2 组合与嵌套

多个条件通过 must / should / mustNot 组合；嵌套对象用点号路径访问，如 `meta.department`。

## 6.3 逻辑组合

```java
// 复杂条件：租户匹配 且 (部门=HR 或 部门=Finance) 且 未过期
Filter filter = Filter.newBuilder()
        .addMust(matchKeyword("tenantId", tenantId))
        .addMust(Filter.newBuilder()
                .addShould(matchKeyword("department", "HR"))
                .addShould(matchKeyword("department", "Finance"))
                .build())
        .addMustNot(matchKeyword("status", "EXPIRED"))
        .build();
```

> Qdrant 的过滤表达力是它的核心优势。

## 6.4 过滤性能与 payload 索引

```java
// 为高频过滤字段建 payload 索引
client.createPayloadIndexAsync("knowledge_base",
        CreateFieldIndex.newBuilder()
                .setFieldName("tenantId")
                .setFieldType(FieldType.FieldTypeKeyword)
                .setFieldSchema(KeywordIndexParams.newBuilder().build())
                .build()).get();

client.createPayloadIndexAsync("knowledge_base",
        CreateFieldIndex.newBuilder()
                .setFieldName("updatedAt")
                .setFieldType(FieldType.FieldTypeDatetime)
                .build()).get();
```

| 字段类型 | 索引类型 | 适用条件 |
|----------|----------|----------|
| keyword | keyword | 等值、集合包含 |
| integer/float | integer/float | 数值范围 |
| datetime | datetime | 时间范围 |
| bool | bool | 开关条件 |

未建索引的字段过滤会退化为逐点扫描，数据量大时延迟会急剧上升。上线前应确认所有高频过滤字段均已建索引。

# 7. 量化（降低内存）

结论先行：量化用少量精度换取数倍内存下降，适合大规模、对召回容忍度稍高的场景。

## 7.1 量化类型

| 量化类型 | 内存 | 精度 |
|----------|------|------|
| 无 | 100% | 最高 |
| 标量（Int8） | ~25% | 略降 |
| 乘积（PQ） | ~10% | 明显降 |

## 7.2 配置示例

```java
client.updateCollectionAsync("knowledge_base",
        UpdateCollection.newBuilder()
                .setOptimizersConfig(OptimizersConfigDiff.newBuilder()
                        .setMemmapThreshold(10000)
                        .build())
                .setQuantizationConfig(QuantizationConfig.newBuilder()
                        .setScalar(ScalarQuantization.newBuilder()
                                .setType(ScalarType.Int8)
                                .build())
                        .build())
                .build()).get();
```

## 7.3 memmap 与阈值

`memmapThreshold` 控制向量何时从内存映射到磁盘：低于阈值的向量常驻内存，高于阈值的按需读取。阈值调小可省内存但增加磁盘 IO；调大则内存换性能。生产环境建议结合内存水位压测后确定。

## 7.4 量化选择建议

| 场景 | 建议 |
|------|------|
| 内存充足、要最高精度 | 不量化 |
| 内存紧张、可接受微降 | 标量量化（Int8） |
| 超大规模、可容忍降召回 | 乘积量化（PQ） |
| 冷热分层 | 热数据内存 + memmap |

启用量化后必须用评测集重新测召回率，确认下降在可接受范围内。

# 8. 混合检索（稠密 + 稀疏）

结论先行：稠密向量擅长语义，稀疏向量擅长关键词；二者融合可同时提升召回与准确率。

## 8.1 稠密与稀疏

```java
// 集合中定义多个命名向量
Collections.VectorParams dense = ...;
Collections.SparseVectorParams sparse = ...;

// 查询时分别检索，再用 RRF 融合
List<ScoredPoint> denseResults = client.searchAsync(denseQuery).get();
List<ScoredPoint> sparseResults = client.searchAsync(sparseQuery).get();
List<ScoredPoint> fused = rrfFuse(denseResults, sparseResults);
```

## 8.2 RRF 融合

RRF（Reciprocal Rank Fusion，倒数排名融合）按排名而非分数融合：

| 项 | 说明 |
|----|------|
| 输入 | 多路检索结果及各自排名 |
| 公式 | `score = Σ 1 / (k + rank)` |
| 优点 | 无需校准各路分数量纲 |
| 参数 | `k` 常取 60 |

RRF 的好处是各路的相似度分数量纲不同也无妨，只依赖排名，工程上稳健。

## 8.3 混合检索完整示例

```java
// 稠密查询：语义相似
SearchPoints denseQuery = SearchPoints.newBuilder()
        .setCollectionName("knowledge_base")
        .setVectorName("dense")
        .addAllVector(toList(denseEmbedding))
        .setLimit(30)
        .setFilter(filter)
        .build();

// 稀疏查询：关键词权重
SearchPoints sparseQuery = SearchPoints.newBuilder()
        .setCollectionName("knowledge_base")
        .setVectorName("sparse")
        .setSparseIndices(SparseIndices.newBuilder()
                .addAllData(sparseIndices).build())
        .setSparseValues(SparseValues.newBuilder()
                .addAllData(sparseValues).build())
        .setLimit(30)
        .setFilter(filter)
        .build();

List<ScoredPoint> fused = rrfFuse(
        client.searchAsync(denseQuery).get(),
        client.searchAsync(sparseQuery).get());
```

两路检索应使用同一过滤条件，避免融合后出现不该返回的结果。

# 9. Spring AI 与 LangChain4j 集成

结论先行：框架封装让 Qdrant 的接入成本极低，但复杂过滤仍需落到原生 Filter 或表达式语法。

## 9.1 Spring AI

```java
@Bean
VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return QdrantVectorStore.builder(embeddingModel)
            .host("localhost")
            .port(6334)
            .collectionName("knowledge_base")
            .initializeSchema(true)
            .build();
}
```

## 9.2 LangChain4j

```java
EmbeddingStore<TextSegment> store = QdrantEmbeddingStore.builder()
        .host("localhost")
        .port(6334)
        .collectionName("knowledge_base")
        .build();

store.addAll(embeddings, segments);
```

| 集成方式 | 优点 | 注意 |
|----------|------|------|
| Spring AI | 自动建 Schema、与 Spring 无缝 | 过滤语法需适配 |
| LangChain4j | 轻量、可脱离 Spring | 需显式向量化 |
| 官方客户端 | 过滤能力最全 | 需自行封装 |

# 10. 生产注意事项

结论先行：Qdrant 部署虽简单，但过滤性能依赖 payload 索引，内存与备份仍需认真规划。

## 10.1 过滤与索引

| 风险 | 建议 |
|------|------|
| payload 未建索引 | 常用过滤字段建索引 |
| 过滤条件注入 | 用类型安全 API，勿拼字符串 |
| 索引类型不符 | keyword 与 integer 分别建索引 |
| 嵌套字段过深 | 扁平化 payload，减少嵌套 |

### 10.1.1 payload 索引类型

keyword 字段用 keyword 索引，数值用 integer/float 索引，时间用 datetime 索引，类型必须匹配。

### 10.1.2 常见误区

把本应数值比较的字段存成字符串，会导致范围过滤失效；过滤字段与索引字段类型不一致时，索引不会被命中。

## 10.2 运维与备份

| 风险 | 建议 |
|------|------|
| 内存不足 | 开启量化 + memmap |
| 单点 | 集群模式 |
| 快照缺失 | 开启快照备份 |
| 版本不兼容 | 锁定客户端与服务端版本 |
| 无监控 | 监控延迟、内存、QPS |

## 10.3 上线检查清单

| 检查项 | 通过标准 |
|--------|----------|
| 维度一致 | 集合 size 与模型一致 |
| payload 索引 | 所有过滤字段已建索引 |
| 预过滤验证 | 小比例租户稳定返回 Top-K |
| 快照恢复 | 已完成恢复演练 |
| 量化压测 | 量化后召回率达标 |
| 版本锁定 | 客户端与服务端版本固定 |

# 11. 小结

- Qdrant 以**强过滤**与**简单部署**见长。
- 过滤在检索前执行，多租户场景结果可靠。
- 支持量化，可大幅降低内存。
- 支持混合检索与多向量。
- 复杂权限过滤场景优先考虑 Qdrant。
- payload 索引是过滤性能的前提，上线前务必补齐。
