# 1. 定位与适用场景

## 1.1 结论先行

Elasticsearch（ES，分布式搜索引擎）从 7.x 起支持 `dense_vector`（稠密向量）字段，8.x 起原生支持 KNN（K-Nearest Neighbors，K 最近邻）检索。它的核心价值是**向量 + 全文混合检索**：BM25（Best Matching 25，经典全文相关性算法）与向量检索在同一引擎内天然融合，已有 ES 的团队可以零新增组件落地 RAG（Retrieval-Augmented Generation，检索增强生成）。

## 1.2 优势拆解

| 优势 | 说明 |
|------|------|
| 混合检索 | 全文（BM25）+ 向量天然融合，RRF 融合开箱可用 |
| 生态成熟 | 已有 ES 的团队零新增组件 |
| 过滤强 | 结构化过滤、聚合分析、排序、高亮 |
| 可扩展 | 分片、副本、集群水平扩展 |
| 分析能力 | 聚合、排序、高亮、聚合桶 |
| 向量量化 | int8/bbq 量化降低内存占用 |

## 1.3 劣势与边界

| 劣势 | 说明 |
|------|------|
| 向量检索性能 | 不如 Milvus/Qdrant 等专用向量库 |
| 内存占用 | 向量字段吃堆外（off-heap）内存 |
| 运维复杂 | 集群调优、分片规划门槛高 |
| 版本差异 | 7.x 与 8.x 的 KNN API 差异较大 |
| 实时性 | 近实时（NRT），写入到可检索有刷新延迟 |

## 1.4 与其他向量库对比

| 维度 | Elasticsearch | Redis | PGVector | Milvus |
|------|---------------|-------|----------|--------|
| 全文检索 | 很强 | 强 | 中（tsvector） | 弱 |
| 混合检索 | 很强 | 强 | 强 | 中 |
| 向量性能 | 中 | 高 | 中 | 很高 |
| 规模 | 中大 | 中小 | 中大 | 超大 |
| 运维 | 复杂 | 简单 | 简单 | 中等 |
| 过滤/聚合 | 很强 | 中 | 强 | 中 |

> 选型经验：**已有 ES、且检索场景同时需要关键词与语义**时，ES 是性价比最高的选择。

# 2. 核心概念与索引映射

## 2.1 关键概念

| 概念 | 说明 |
|------|------|
| Index（索引） | 一类文档的集合，含 mapping 定义 |
| Mapping（映射） | 字段类型与索引方式定义 |
| `dense_vector` | 稠密向量字段类型 |
| KNN | 近似最近邻检索 |
| `num_candidates` | 每个分片的候选集大小 |
| Retriever | 8.x 引入的检索器抽象，支持 RRF |
| BM25 | 默认全文相关性打分算法 |

## 2.2 索引映射定义

```json
PUT /knowledge
{
  "mappings": {
    "properties": {
      "content":    { "type": "text", "analyzer": "ik_max_word" },
      "embedding":  {
        "type": "dense_vector",
        "dims": 1024,
        "index": true,
        "similarity": "cosine"
      },
      "tenantId":   { "type": "keyword" },
      "source":     { "type": "keyword" },
      "updatedAt":  { "type": "date" }
    }
  }
}
```

| 字段 | 说明 |
|------|------|
| `dense_vector` | 向量字段 |
| `dims` | 维度（必须与模型一致） |
| `index: true` | 启用 KNN 检索 |
| `similarity` | cosine / dot_product / l2_norm |

> 中文全文检索需安装 IK 分词器（中文分词插件），并配置 `analyzer: ik_max_word`。

## 2.3 相似度与距离度量

| `similarity` | 含义 | 适用 |
|--------------|------|------|
| `cosine` | 余弦相似度 | 归一化向量，通用推荐 |
| `dot_product` | 内积 | 模型输出未归一化 |
| `l2_norm` | 欧氏距离 | 空间距离敏感场景 |

### 2.3.1 归一化的影响

若 embedding 已归一化（模长为 1），cosine 与 dot_product 等价。多数现代 embedding 模型输出已归一化，选 cosine 最省心。

# 3. 环境准备与依赖

## 3.1 部署与分词器

本地开发可用单节点模式快速启动：

```bash
docker run -d --name es \
  -p 9200:9200 -p 9300:9300 \
  -e discovery.type=single-node \
  -e xpack.security.enabled=false \
  docker.elastic.co/elasticsearch/elasticsearch:8.13.0
```

IK 分词器需手动安装到 `plugins` 目录后重启。

## 3.2 Maven 依赖

LangChain4j 风格：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-elasticsearch</artifactId>
</dependency>
```

Spring AI 风格：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-elasticsearch</artifactId>
</dependency>
```

## 3.3 客户端配置

```java
@Bean
public ElasticsearchClient elasticsearchClient() {
    RestClient restClient = RestClient.builder(
            new HttpHost("localhost", 9200, "http")).build();

    ElasticsearchTransport transport = new RestClientTransport(
            restClient, new JacksonJsonpMapper());

    return new ElasticsearchClient(transport);
}
```

> 生产环境务必启用安全（TLS + 认证），示例为方便关闭了 `xpack.security`。

# 4. 数据写入

## 4.1 写入机制

ES 的写入是**近实时**的：文档先入内存缓冲与 translog，默认 1 秒后 refresh 才可被检索。向量字段以数组形式传入，维度必须与 mapping 一致。

## 4.2 单条写入示例

```java
public void index(String id, String content, float[] vector,
                  String tenantId, String source) throws IOException {
    Map<String, Object> doc = Map.of(
            "content", content,
            "embedding", toList(vector),
            "tenantId", tenantId,
            "source", source,
            "updatedAt", Instant.now().toString());

    IndexRequest request = new IndexRequest("knowledge")
            .id(id)
            .source(doc, XContentType.JSON);

    client.index(request, RequestOptions.DEFAULT);
}
```

## 4.3 批量写入（Bulk）

```java
public void bulkIndex(List<Doc> docs) throws IOException {
    BulkRequest.Builder br = new BulkRequest.Builder();
    for (Doc d : docs) {
        br.operations(op -> op.index(idx -> idx
                .index("knowledge")
                .id(d.id())
                .document(toMap(d))));
    }
    BulkResponse response = client.bulk(br.build());
    if (response.errors()) {
        // 逐条检查失败原因
        response.items().stream()
                .filter(i -> i.error() != null)
                .forEach(i -> log.warn("bulk error: {}", i.error().reason()));
    }
}
```

## 4.4 刷新策略

```java
// 大批量导入时临时关闭刷新，导入完再恢复
client.indices().putSettings(s -> s
        .index("knowledge")
        .settings(v -> v.refreshInterval(t -> t.time("-1"))));
// ... bulk ...
client.indices().refresh(r -> r.index("knowledge"));
```

> 批量导入时关闭 refresh 可显著提升吞吐，但会延迟可见性，导入结束后必须手动 refresh。

# 5. KNN 检索

## 5.1 检索机制

ES 8.x 的 KNN 采用 HNSW（Hierarchical Navigable Small World，分层可导航小世界图）近似检索。`k` 是返回条数，`num_candidates` 是每个分片的候选集，候选越多越准但越慢。

## 5.2 基础 KNN 示例

```java
public List<SearchHit> knnSearch(float[] queryVector, String tenantId, int k)
        throws IOException {

    Query filterQuery = QueryBuilders.termQuery("tenantId", tenantId);

    KnnSearchBuilder knn = new KnnSearchBuilder()
            .field("embedding")
            .queryVector(toList(queryVector))
            .k(k)
            .numCandidates(k * 10)          // 候选数，越大越准
            .filter(filterQuery);           // 预过滤

    SearchSourceBuilder source = new SearchSourceBuilder().knn(knn);

    SearchResponse response = client.search(
            new SearchRequest("knowledge").source(source),
            RequestOptions.DEFAULT);

    return Arrays.asList(response.getHits().getHits());
}
```

| 参数 | 说明 |
|------|------|
| `k` | 返回条数 |
| `numCandidates` | 候选集大小（影响精度与性能） |
| `filter` | 预过滤条件 |

### 5.2.1 num_candidates 取值经验

| 数据量 | 建议 |
|--------|------|
| 小（< 10 万） | `num_candidates` = k 或 2k |
| 中（10 万 ~ 100 万） | `num_candidates` = 10k ~ 20k |
| 大（> 100 万） | 至少 100，必要时 500+ |

## 5.3 边界与坑

| 坑 | 说明 | 规避 |
|----|------|------|
| 预过滤后候选不足 | filter 太严导致 k 结果不满 | 提高 num_candidates |
| 维度不匹配 | 写入报错 | mapping 与模型对齐 |
| 未 refresh | 刚写入查不到 | 手动 refresh 或等待 |
| 堆外内存溢出 | 向量占 off-heap | 监控并调大 off-heap |
| 分片过多 | 每分片独立候选，结果合并开销大 | 合理规划分片 |

# 6. 混合检索（全文 + 向量）

## 6.1 RRF 融合

RRF（Reciprocal Rank Fusion，倒数排名融合）是混合检索的标准融合方式，避免不同检索器分数不可比的问题。

```java
// 全文检索
SearchSourceBuilder bm25 = new SearchSourceBuilder()
        .query(QueryBuilders.matchQuery("content", question))
        .size(20);

// 向量检索
SearchSourceBuilder knn = new SearchSourceBuilder()
        .knn(new KnnSearchBuilder()
                .field("embedding")
                .queryVector(toList(queryVector))
                .k(20)
                .numCandidates(200));

// 用 RRF 组合
RrfRetriever rrf = new RrfRetriever(
        List.of(new StandardRetriever(bm25), new StandardRetriever(knn)),
        RrfRetriever.RANK_CONSTANT);

SearchResponse response = client.search(new SearchRequest("knowledge")
        .source(new SearchSourceBuilder().retriever(rrf)), RequestOptions.DEFAULT);
```

## 6.2 手动 RRF 实现

```java
public List<String> rrfFuse(List<List<String>> rankedLists, int k) {
    Map<String, Double> scores = new HashMap<>();

    for (List<String> list : rankedLists) {
        for (int rank = 0; rank < list.size(); rank++) {
            String id = list.get(rank);
            scores.merge(id, 1.0 / (k + rank + 1), Double::sum);
        }
    }
    return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .map(Map.Entry::getKey)
            .toList();
}
```

### 6.2.1 RRF 公式

```text
score(d) = Σ_r  1 / (k + rank_r(d))
```

其中 `k` 是平滑常数（ES 默认 60），`rank_r(d)` 是文档 d 在检索器 r 中的排名（从 1 开始）。

## 6.3 线性加权 vs RRF

| 方式 | 说明 | 缺点 |
|------|------|------|
| 线性加权 | `α·bm25 + (1-α)·cosine` | 分数尺度不可比，需归一化 |
| RRF | 只用排名，不需归一化 | 丢失分数幅度信息 |

> 经验：跨检索器融合优先 RRF；同尺度分数（如都是余弦）才考虑加权。

# 7. 框架集成

## 7.1 Spring AI 集成

```java
@Bean
VectorStore vectorStore(ElasticsearchClient client, EmbeddingModel model) {
    return ElasticsearchVectorStore.builder(client, model)
            .indexName("knowledge")
            .initializeSchema(true)
            .build();
}
```

## 7.2 Spring AI 使用

```java
@Service
public class KnowledgeService {

    private final VectorStore vectorStore;

    public KnowledgeService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public List<Document> search(String question) {
        return vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(10)
                        .similarityThreshold(0.7)
                        .build());
    }
}
```

## 7.3 LangChain4j 集成

```java
EmbeddingStore<TextSegment> store = ElasticsearchEmbeddingStore.builder()
        .serverUrl("http://localhost:9200")
        .indexName("knowledge")
        .dimension(1024)
        .build();
```

> 框架默认的 `initializeSchema(true)` 会按默认维度建索引，务必确认与模型维度一致。

# 8. 性能调优

## 8.1 调优清单

| 手段 | 说明 |
|------|------|
| 控制 `numCandidates` | 精度与性能的平衡 |
| 使用预过滤 | 缩小搜索范围 |
| 向量量化 | `int8_hnsw` / `bbq_hnsw` 降低内存 |
| 分片规划 | 按数据量合理设置分片数 |
| 堆外内存 | 向量占用堆外，注意 off-heap 配置 |
| 副本 | 提升查询吞吐 |

## 8.2 向量量化配置

```json
"embedding": {
  "type": "dense_vector",
  "dims": 1024,
  "index_options": {
    "type": "int8_hnsw"
  }
}
```

| 量化类型 | 说明 | 内存 |
|----------|------|------|
| `hnsw` | 默认 float32 | 100% |
| `int8_hnsw` | 8 位整数量化 | ~25% |
| `bbq_hnsw` | 二值量化 | 更省，精度更低 |

### 8.2.1 量化代价

量化以召回率换取内存：int8 通常损失很小，bbq 损失较大，需按场景评估。

## 8.3 分片规划

| 数据量 | 分片建议 |
|--------|----------|
| < 100 万 | 1~3 个主分片 |
| 100 万 ~ 1000 万 | 3~10 个主分片 |
| > 1000 万 | 按节点数规划，单分片 30~50GB |

> 分片过多会放大每分片候选集合并的开销；分片过少则无法利用集群并行。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 维度不匹配 | 建索引前确定 |
| 分片过多/过少 | 按数据量与节点数规划 |
| 堆内存不足 | 关注堆外内存与 GC |
| 混合检索分数不可比 | 用 RRF 而非加权 |
| 中文分词 | 安装 IK 并配置 analyzer |
| 集群调优 | 参考官方 sizing 指南 |

## 9.2 监控指标

| 指标 | 含义 | 关注点 |
|------|------|--------|
| `jvm.mem.heap_used` | 堆使用 | 避免频繁 GC |
| off-heap 用量 | 向量内存 | 接近上限需扩容 |
| `search.latency` | 查询延迟 | KNN 延迟上升排查候选数 |
| `refresh.time` | 刷新耗时 | 过高排查写入压力 |

## 9.3 版本与兼容

| 版本 | KNN 支持 |
|------|----------|
| 7.x | `_knn_search`（实验性） |
| 8.x | 原生 `knn` 查询 + Retriever/RRF |

> 升级前确认客户端与集群版本匹配，7.x 与 8.x 的 KNN API 不兼容。

# 10. 小结

## 10.1 关键结论

- ES 的优势是**向量 + 全文混合检索**与成熟生态。
- `dense_vector` + KNN 支持预过滤，`num_candidates` 是精度/性能旋钮。
- 混合检索用 RRF 融合，避免分数不可比。
- 向量字段吃堆外内存，需关注容量规划。
- 已有 ES 的团队可零新增组件落地 RAG。

## 10.2 选型建议

- 检索同时需要关键词与语义：优先 ES。
- 只做纯向量、规模大：考虑专用向量库。
- 需要强事务：考虑 PGVector。
- 需要极低延迟热数据：考虑 Redis。
