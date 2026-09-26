# 1. EmbeddingModel 抽象

结论先行：`EmbeddingModel` 把文本映射为定长向量，是 RAG 的入口。它决定"语义空间"，一旦选定，向量库的维度、相似度计算与检索效果都被绑定。

## 1.1 接口

### 1.1.1 方法

```java
public interface EmbeddingModel {

    Response<Embedding> embed(String text);

    Response<Embedding> embed(TextSegment textSegment);

    Response<List<Embedding>> embedAll(List<TextSegment> textSegments);
}
```

三个方法覆盖"单条文本、带元数据的片段、批量片段"三种输入。`Response` 包裹结果并附带用量信息。

### 1.1.2 核心概念

| 概念 | 说明 |
|------|------|
| `Embedding` | 向量（`float[]`，含维度） |
| `TextSegment` | 文本片段（文本 + 元数据） |
| `Response<T>` | 结果 + Token 用量 + 完成原因 |

### 1.1.3 维度与向量表示

`Embedding.vector()` 返回 `float[]`，长度即维度。维度是向量库的硬约束：写入与查询必须同维度，检索才成立。

```
"什么是向量检索" ─▶ EmbeddingModel ─▶ [0.12, -0.03, 0.88, ...]
                                        └──── 长度 = 维度 ────┘
```

不同模型的向量空间不可混用：即便维度相同，语义坐标也完全不同。

## 1.2 构建与调用

### 1.2.1 云端模型

```java
EmbeddingModel model = OpenAiEmbeddingModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("text-embedding-3-small")
        .build();

Embedding embedding = model.embed("什么是向量检索").content();
float[] vector = embedding.vector();
```

### 1.2.2 本地模型

本地 ONNX 模型无需联网，数据不出域，适合合规场景：

```java
EmbeddingModel localModel = new OnnxEmbeddingModel(
        Path.of("models/bge-small-zh.onnx"),
        Path.of("models/tokenizer.json"));
```

边界与坑：本地模型的 tokenizer 必须与训练时一致，否则向量质量显著下降。

## 1.3 批处理与并发

```java
Response<List<Embedding>> response = model.embedAll(segments);
List<Embedding> embeddings = response.content();
```

生产建议：优先 `embedAll` 批量调用，减少网络往返；对超大集合自行分批（如每批 100 条），并控制并发，避免触发服务端限流。

### 1.3.1 分批与重试

```java
public List<Embedding> embedInBatches(List<TextSegment> segments, int batch) {
    List<Embedding> all = new ArrayList<>();
    for (int i = 0; i < segments.size(); i += batch) {
        List<TextSegment> slice = segments.subList(i, Math.min(i + batch, segments.size()));
        all.addAll(model.embedAll(slice).content());
    }
    return all;
}
```

边界与坑：失败重试需保证幂等，且不要把整批重试变成"部分成功 + 部分重复"。

# 2. 常用 Embedding 模型

结论先行：中文场景优先 BGE/GTE，多语言场景选 E5/Cohere，成本敏感或数据敏感场景用本地 ONNX。选型是"一次性决策"，变更需重建全部向量。

## 2.1 模型清单

| 模型 | 维度 | 特点 |
|------|------|------|
| `text-embedding-3-small` | 1536 | 性价比高 |
| `text-embedding-3-large` | 3072 | 效果更好，成本更高 |
| BGE 系列 | 512/768/1024 | 中文友好，可本地部署 |
| E5 系列 | 768/1024 | 多语言，检索强 |
| GTE 系列 | 768/1024 | 中文与多语言 |
| Cohere Embed | 1024+ | 商业，多语言 |

## 2.2 选型维度

### 2.2.1 语言与维度

| 维度 | 说明 |
|------|------|
| 语言 | 中文优先选 BGE/GTE |
| 维度 | 越高越准，存储与检索成本越大 |
| 领域 | 通用 vs 领域微调 |

### 2.2.2 部署与成本

| 维度 | 说明 |
|------|------|
| 部署 | 云端 API vs 本地 ONNX |
| 成本 | 按 Token 计费 vs 一次性部署 |
| 延迟 | 网络往返 vs 本地推理 |
| 合规 | 数据是否允许出域 |

> 详见 `5_RAG/Embedding/2_embedding_models.md`。

## 2.3 归一化与相似度

多数 Embedding 模型输出需做 L2 归一化，再用余弦相似度。若模型已归一化，则内积等价于余弦。维度与归一化方式必须与索引配置一致。

| 相似度 | 公式直觉 | 适用 |
|--------|----------|------|
| 余弦 | 方向夹角 | 文本语义（最常用） |
| 内积 | 方向 + 模长 | 已归一化向量 |
| 欧氏 | 空间距离 | 部分图像场景 |

## 2.4 模型对比实验

选型不能只看榜单，要在自己的数据上做小规模对比：

| 步骤 | 做法 |
|------|------|
| 取样 | 从真实语料抽 100-200 条问题 + 应命中片段 |
| 跑分 | 对候选模型分别建索引，计算 Recall@5 |
| 看延迟 | 测 P95 单条耗时与批量吞吐 |
| 定模型 | 综合质量、成本、合规决策 |

生产建议：把该实验固化为脚本，模型升级前重跑，用同一黄金集对比新旧模型。

# 3. EmbeddingStore 抽象

结论先行：`EmbeddingStore` 是向量与原文的持久化抽象，提供写入、删除、检索三类操作。它屏蔽了底层向量库差异。

## 3.1 接口

```java
public interface EmbeddingStore<Embedded> {

    String add(Embedding embedding);

    void add(String id, Embedding embedding);

    String add(Embedding embedding, Embedded embedded);

    List<String> addAll(List<Embedding> embeddings);

    void addAll(List<Embedding> embeddings, List<Embedded> embedded);

    void remove(String id);

    void removeAll(Collection<String> ids);

    EmbeddingSearchResult<Embedded> search(EmbeddingSearchRequest request);
}
```

## 3.2 写入方法

- `add(embedding)`：只存向量，返回自动生成的 ID；
- `add(id, embedding)`：指定 ID，便于幂等覆盖；
- `add(embedding, embedded)`：同时存向量与原文（`Embedded` 常为 `TextSegment`）；
- `addAll(...)`：批量，生产首选。

## 3.3 检索请求与结果

`EmbeddingSearchRequest` 承载查询向量、`maxResults`、`minScore` 与 `filter`；返回 `EmbeddingSearchResult`，其 `matches()` 为按分数降序的 `EmbeddingMatch` 列表，每项含 `score`、`embeddingId` 与 `embedded`。

## 3.4 与 EmbeddingModel 的配合

`EmbeddingStore` 只存与检索向量，不负责把文本转成向量。二者由检索器或调用方串联：

```
文本 ─EmbeddingModel─▶ 向量 ─EmbeddingStore.add─▶ 存储
问题 ─EmbeddingModel─▶ 向量 ─EmbeddingStore.search─▶ 命中
```

边界与坑：写入与查询必须使用**同一个** `EmbeddingModel` 实例配置（模型名、维度、归一化一致），否则分数失真。

# 4. 常用向量库

结论先行：原型用内存，生产按"规模 + 过滤能力 + 运维成本"三要素选型。已有 PostgreSQL 优先 PGVector，大规模选 Milvus，强过滤选 Qdrant。

## 4.1 清单对比

| 向量库 | 依赖 | 适用 |
|--------|------|------|
| `InMemoryEmbeddingStore` | 核心 | 原型、测试 |
| PGVector | `langchain4j-pgvector` | 已有 PostgreSQL |
| Milvus | `langchain4j-milvus` | 大规模 |
| Qdrant | `langchain4j-qdrant` | 强过滤 |
| Redis | `langchain4j-redis` | 高并发 |
| Elasticsearch | `langchain4j-elasticsearch` | 混合检索 |
| Chroma | `langchain4j-chroma` | 轻量原型 |
| Pinecone / Weaviate | 对应模块 | 托管服务 |

## 4.2 PGVector 示例

### 4.2.1 依赖

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-pgvector</artifactId>
</dependency>
```

### 4.2.2 构建

```java
EmbeddingStore<TextSegment> store = PgVectorEmbeddingStore.builder()
        .host("localhost")
        .port(5432)
        .database("ai")
        .user("postgres")
        .password(System.getenv("DB_PASSWORD"))
        .table("embeddings")
        .dimension(1536)
        .createTable(true)
        .build();
```

边界与坑：`dimension` 必须与 Embedding 模型输出维度完全一致，否则写入报错或检索无结果。

## 4.3 InMemory（开发/测试）

```java
EmbeddingStore<TextSegment> store = new InMemoryEmbeddingStore<>();
```

> 生产**不要**用 `InMemoryEmbeddingStore`：重启丢失、无法扩展。

## 4.4 选型建议

| 规模 | 过滤需求 | 推荐 |
|------|----------|------|
| 万级 | 低 | InMemory / Chroma |
| 十万级 | 中 | PGVector / Redis |
| 百万级 | 中高 | Qdrant / Milvus |
| 千万级 | 高 | Milvus / 托管服务 |

## 4.5 连接与资源管理

- 向量库客户端应复用（连接池），不要每次检索新建连接；
- 大批量写入单独走离线任务，与在线检索隔离资源；
- 为客户端设置超时与重试，避免向量库抖动拖垮整个请求链。

## 4.6 托管与自建

| 维度 | 托管服务 | 自建 |
|------|----------|------|
| 运维 | 免运维 | 需运维集群 |
| 成本 | 按量/订阅 | 固定资源 |
| 数据 | 可能出域 | 完全自控 |
| 扩展 | 弹性 | 需自行扩缩容 |
| 定制 | 受限 | 可深度定制 |

生产建议：数据合规优先自建；追求上线速度与弹性可选托管。混合架构（热数据托管、冷数据自建）也可行，但会增加复杂度。

# 5. 写入与检索

结论先行：写入时把"文本 + 元数据"打包成 `TextSegment` 并向量化入库；检索时用查询向量加过滤条件，取 Top-K 并读取原文。

## 5.1 写入

### 5.1.1 TextSegment 与元数据

```java
TextSegment segment = TextSegment.from(
        "报销需在费用发生后 30 天内提交",
        Metadata.from(Map.of("source", "handbook.pdf", "tenantId", "acme")));

Embedding embedding = embeddingModel.embed(segment).content();
store.add(embedding, segment);
```

### 5.1.2 批量写入

```java
// 批量写入
List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
store.addAll(embeddings, segments);
```

生产建议：批量写入时对输入去重，避免重复片段稀释检索；为每批设置大小上限并记录失败重试。

## 5.2 检索

### 5.2.1 构建请求

```java
Embedding queryEmbedding = embeddingModel.embed("报销时限").content();

EmbeddingSearchRequest request = EmbeddingSearchRequest.builder()
        .queryEmbedding(queryEmbedding)
        .maxResults(5)
        .minScore(0.7)
        .filter(metadataKey("tenantId").isEqualTo("acme"))
        .build();
```

### 5.2.2 处理结果

```java
EmbeddingSearchResult<TextSegment> result = store.search(request);

for (EmbeddingMatch<TextSegment> match : result.matches()) {
    System.out.printf("score=%.3f text=%s%n",
            match.score(), match.embedded().text());
}
```

### 5.2.3 分数语义

`score` 的含义由向量库与相似度函数决定，不同库之间不可直接比较。`minScore` 阈值必须结合所选库与模型实测确定。

| 库/模型 | score 含义 | 越大越好 |
|---------|------------|----------|
| 余弦相似度 | 夹角余弦 | 是 |
| 内积 | 点积 | 是（已归一化） |
| 欧氏距离 | 距离 | 否（需转换） |

## 5.3 用 ContentRetriever 简化

在 RAG 中通常不直接调用 `store.search`，而是交给检索器：

```java
ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
        .embeddingStore(store)
        .embeddingModel(embeddingModel)
        .maxResults(5)
        .minScore(0.7)
        .build();
```

# 6. 元数据过滤

结论先行：元数据是权限、时效与版本治理的基础。过滤在"检索时"生效，但字段必须在"写入时"打好。

## 6.1 过滤表达式

```java
// 简单条件
metadataKey("department").isEqualTo("HR")

// 组合条件
metadataKey("tenantId").isEqualTo(tenant)
        .and(metadataKey("year").isGreaterThanOrEqualTo(2024))
```

## 6.2 常见场景

| 场景 | 过滤字段 |
|------|----------|
| 多租户 | `tenantId` |
| 权限 | `department` / `acl` |
| 时效 | `updatedAt` |
| 版本 | `version` |

## 6.3 边界与坑

- 过滤字段未建索引时，大数据量下会退化为全表扫描；
- 不同向量库支持的过滤操作符不同，跨库迁移需重新验证；
- 组合条件要注意空值语义（`isEqualTo(null)` 的行为因库而异）。

> 元数据必须在**写入时**打好，事后补需要重建索引。

## 6.4 过滤下推

过滤应尽量"下推"到向量库执行，而不是取回后在应用层过滤：

```
好：search(filter=tenantId=acme, topK=5)   ← 库内先过滤再算向量
差：search(topK=1000) → 应用层 filter       ← 取多了、慢、可能漏
```

边界与坑：部分向量库在"过滤 + 近似检索"组合下召回率会下降，需实测；必要时改用"先过滤后检索"的两段式。

# 7. 索引与性能

结论先行：向量检索的性能由"索引类型 + 维度 + 过滤选择性"共同决定。先建索引，再谈调参。

## 7.1 关注点

| 关注点 | 手段 |
|--------|------|
| 查询延迟 | HNSW 参数调优、减少维度 |
| 写入吞吐 | 批量 `addAll`、异步、分批 |
| 存储成本 | 量化（PQ/SQ）、降维 |
| Embedding 成本 | 缓存向量、去重后再向量化、本地模型 |

## 7.2 索引类型

### 7.2.1 HNSW

基于图的近似最近邻，查询快、召回高，代价是内存占用较大。常用参数 `M`（连接数）与 `efConstruction`/`efSearch`（搜索宽度）。`M` 越大召回越好、内存越高。

### 7.2.2 IVF 与 PQ

IVF（倒排文件）先聚类再在最近簇内搜索，适合大规模；PQ（乘积量化）压缩向量，显著降低存储但损失精度。二者常组合为 IVF-PQ。

| 索引 | 查询速度 | 召回 | 内存 |
|------|----------|------|------|
| 暴力搜索 | 慢 | 100% | 高 |
| HNSW | 快 | 高 | 高 |
| IVF-PQ | 快 | 中 | 低 |

## 7.3 批量与缓存

- 写入用批量接口，单条写入在高吞吐下不可行；
- 对"问题 → 向量"做缓存，相同问题不重复调用 Embedding；
- 对"问题 → 检索结果"做短 TTL 缓存，缓解热点。

## 7.4 参数调优

以 HNSW 为例，调参的基本方向：

| 参数 | 调大 | 调小 |
|------|------|------|
| `M` | 召回高、内存高 | 内存低、召回降 |
| `efConstruction` | 建索引慢、质量高 | 建索引快、质量降 |
| `efSearch` | 查询慢、召回高 | 查询快、召回降 |

生产建议：先用默认值上线，再基于评测集在"召回—延迟"曲线上取工作点。

# 8. 多租户隔离

结论先行：多租户要么用元数据过滤（简单、共享索引），要么用独立表/集合（隔离强、运维重）。无论哪种，都要写越权测试。

## 8.1 方案对比

| 方案 | 隔离强度 | 成本 | 适用 |
|------|----------|------|------|
| 同表 + 元数据过滤 | 中 | 低 | 租户多 |
| 每租户独立表/集合 | 高 | 中 | 租户少、合规高 |
| 独立实例 | 最高 | 高 | 大客户 |

## 8.2 实现

```java
// 方案一：同表 + 元数据过滤（简单）
.filter(metadataKey("tenantId").isEqualTo(tenant))

// 方案二：每租户独立表/集合（隔离强，运维重）
PgVectorEmbeddingStore.builder().table("embeddings_" + tenant).build();
```

> 无论哪种，都要写"越权检索"测试用例。

边界与坑：方案二若租户数量大，会带来大量小表与连接管理负担；方案一若过滤遗漏，会直接泄漏数据。推荐默认方案一 + 强制过滤中间件。

## 8.3 租户路由

独立表/实例方案需要一个"租户 → 存储"的路由层：

```java
public class TenantStoreRouter {

    private final Map<String, EmbeddingStore<TextSegment>> stores;

    public EmbeddingStore<TextSegment> forTenant(String tenantId) {
        return stores.computeIfAbsent(tenantId, this::buildStore);
    }
}
```

生产建议：路由表加缓存并设上限，防止租户爆炸导致连接耗尽；租户下线时同步释放资源。

# 9. 生产注意事项

结论先行：向量库的风险集中在维度/模型变更、过滤注入、无索引、无备份与全量重建抖动。变更要"可回滚、可灰度"。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 维度不匹配 | 换模型需整体重建向量库 |
| 过滤注入 | 用类型安全的过滤 API，勿拼接字符串 |
| 无索引 | 大数据量前先建索引 |
| 无备份 | 向量数据同样需要备份 |
| 模型漂移 | Embedding 升级会导致语义空间变化，需重建 |
| 全量重建抖动 | 用别名/新表切换 |

## 9.2 维度与模型变更

换 Embedding 模型意味着整个语义空间改变，旧向量与新查询向量不可比。必须：新建索引 → 全量重算 → 切别名 → 验证 → 下线旧索引。

## 9.3 备份与重建

- 定期备份向量表（或可从原始文档重建，视成本而定）；
- 重建时用新表 + 别名原子切换，避免线上检索中断；
- 记录每次重建的模型版本与时间戳，便于回溯。

## 9.4 监控指标

| 指标 | 说明 |
|------|------|
| 检索 P95 延迟 | 用户体验 |
| 写入吞吐 | 索引任务进度 |
| 平均命中数 | 过滤是否过严 |
| 空结果率 | 召回是否失效 |
| 向量库内存/磁盘 | 容量规划 |
| Embedding 调用量 | 成本监控 |

生产建议：为空结果率与延迟设告警阈值，并区分"过滤导致"与"召回导致"的空结果。

# 10. 小结

## 10.1 核心要点

- `EmbeddingModel` + `EmbeddingStore` 是 RAG 的存储底座。
- 向量库选择看规模、过滤能力与运维成本。
- 元数据是过滤、权限与治理的基础，写入时就要打好。
- Embedding 模型与维度是"一次性决策"，变更代价高。

## 10.2 实践清单

| 检查项 | 目标 |
|--------|------|
| 维度与模型一致 | 写入不报错 |
| 已建向量索引 | 查询不退化 |
| 元数据齐全 | 可过滤、可溯源 |
| 有越权测试 | 不泄漏数据 |
| 有备份/重建方案 | 可恢复、可切换 |
| 有向量缓存 | 降本提速 |
