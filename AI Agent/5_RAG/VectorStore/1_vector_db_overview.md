# 1. 向量库的职责与定位

向量库（Vector Database）是 RAG（检索增强生成，Retrieval-Augmented Generation）的存储与检索基础设施。它把文本、图片等非结构化内容经嵌入模型（Embedding Model）转成稠密向量后持久化，并在查询时按相似度返回 Top-K 结果。结论先行：没有向量库，RAG 只能做关键词匹配，无法做语义检索，也无法把"权限、租户、时间"等业务约束与语义检索合并执行。

## 1.1 写入与查询两条主链路

向量库的所有能力都可以归结为两条链路：

```
写入：向量 + 元数据 + 原文  ->  持久化存储
查询：查询向量 + 过滤条件   ->  Top-K 相似结果
```

写入链路负责把内容切块（Chunk）、向量化、连同元数据（Metadata）一并落盘；查询链路负责把用户问题向量化，附加过滤条件，再在候选集合中做近似最近邻（ANN，Approximate Nearest Neighbor）检索。两条链路的关键差别在于：写入是"离线、批量、可重试"的，查询是"在线、低延迟、高并发"的。这决定了索引结构与内存策略通常向查询倾斜。

## 1.2 核心能力拆解

| 能力 | 说明 | 缺失后的后果 |
|------|------|--------------|
| 相似度检索 | ANN（近似最近邻），支持 COSINE/IP/L2 | 只能关键词命中，召回差 |
| 元数据过滤 | 按条件筛选（权限、租户、时间） | 越权返回、结果不可控 |
| 增删改 | 支持增量维护、软删除、更新 | 只能全量重建索引 |
| 持久化 | 数据落盘、崩溃可恢复 | 重启即丢，无法上线 |
| 水平扩展 | 分片/副本支撑亿级向量 | 单机内存天花板 |
| 一致性 | 写入后可见性、事务语义 | 检索到旧数据或漏数据 |

其中"过滤"常被低估。生产环境中，检索几乎从不裸查，总要带 `tenant_id`、`acl`、`status` 之类的条件；能否把过滤下推到向量检索内部，直接决定结果是否可靠（详见第 7 节）。

## 1.3 在 RAG 链路中的位置

```
问题 -> 嵌入模型 -> 查询向量 ┐
                              ├-> 向量库(过滤+ANN) -> Top-K -> 重排 -> LLM
文档 -> 切块 -> 嵌入模型 -> 写入 ┘
```

向量库处在"检索"环节的正中心：上游是嵌入模型，下游是重排器（Reranker）与大模型。它的输出质量（召回率、延迟）会沿着链路放大——召回漏了，重排和 LLM 都救不回来。

## 1.4 常见误解

| 误解 | 事实 |
|------|------|
| 向量库等于搜索引擎 | 搜索引擎偏关键词，向量库偏语义，二者可互补 |
| 向量越大越好 | 维度高则内存与延迟上升，收益递减 |
| 存进去就能查 | Milvus 等需 load，部分库需 flush 才可见 |
| 元数据随便塞 | 可过滤字段需建索引，否则拖慢检索 |
| 一次选型定终身 | 借助统一抽象，可按规模演进换库 |

# 2. 选型维度

选型没有"最好"，只有"最合适"。结论先行：先用"规模 + 过滤"筛掉一半候选，再用"现有技术栈 + 运维成本"做最终决策。

## 2.1 规模与成本维度

| 维度 | 说明 | 经验阈值 |
|------|------|----------|
| 数据规模 | 万级 / 百万级 / 亿级 | 千万以下可单机，亿级必须分布式 |
| 维度 | 常见 768 / 1024 / 1536 / 3072 | 维度翻倍，内存近似翻倍 |
| 延迟要求 | 毫秒级 / 百毫秒级 | 在线问答通常 P99 < 100ms |
| 成本 | 自建 vs 托管 | 托管按量计费，规模大后自建更省 |
| 内存预算 | HNSW 常驻内存 | 1 亿 × 1024 维 × 4B ≈ 400GB 原始向量 |

一个容易被忽略的换算：单条 1024 维 float32 向量约 4KB。1000 万条即约 40GB 纯向量内存，再叠加 HNSW 图结构（通常再放大 1.2~2 倍），单机就非常吃力。

## 2.2 过滤与一致性维度

| 维度 | 说明 | 影响 |
|------|------|------|
| 过滤能力 | 是否支持预过滤、嵌套字段、逻辑组合 | 多租户、权限场景生死线 |
| 过滤复杂度 | 单条件 / 多条件 AND/OR / 集合包含 | 决定表达式表达力 |
| 一致性 | 是否需要事务、写后立即可见 | 影响"刚写入就查"的场景 |
| 删除语义 | 硬删 / 软删 / 过期 TTL | 影响合规与成本 |

需要事务（如"写向量与写业务表要么都成功"）时，纯向量库往往不给力，需借助外部事务或选择支持事务的方案（如 PGVector 借助 PostgreSQL 事务）。

## 2.3 Java 生态与运维维度

### 2.3.1 客户端成熟度

Java（JDK 17/21）项目重点看：是否有官方 SDK、是否被 Spring AI / LangChain4j 一等公民支持、社区活跃度。官方 SDK 通常意味着 API 稳定、功能覆盖全；仅有社区客户端则可能长期滞后于服务端版本。

### 2.3.2 运维成本

"能否复用现有基础设施"经常比性能更重要。已有 PostgreSQL 的团队用 PGVector，几乎零新增运维；已有 Elasticsearch 的团队用 ES 向量检索，可同时做关键词混合检索；反之，引入 Milvus 集群意味着要维护 etcd、对象存储、消息队列三套组件。

### 2.3.3 打分矩阵

| 维度 | 权重建议 | 说明 |
|------|----------|------|
| 功能匹配 | 30% | 过滤、混合检索、多向量 |
| 规模适配 | 25% | 能否覆盖 3 年数据量 |
| 运维成本 | 20% | 团队能否长期维护 |
| 生态成熟 | 15% | SDK、文档、社区 |
| 成本 | 10% | 硬件 / 托管费用 |

## 2.4 成本估算示例

以 1000 万条、1024 维、float32 向量为例：

| 项目 | 估算 | 说明 |
|------|------|------|
| 原始向量 | 约 40GB | 1000 万 × 4KB |
| HNSW 图结构 | 约 48~80GB | 原始向量的 1.2~2 倍 |
| 原文与元数据 | 数十 GB | 取决于文本长度 |
| 内存需求 | 64~128GB | 需留出查询余量 |

若启用标量量化（Int8），内存可降至约 1/4；启用乘积量化（PQ）可再压缩，但召回会下降。选型时先算清这笔账，再决定是否需要分布式。

# 3. 主流向量库对比

结论先行：中小规模优先复用现有数据库，亿级规模选 Milvus，复杂过滤选 Qdrant，原型验证用 Chroma，不想运维用 Pinecone。

## 3.1 对比总表

| 向量库 | 基础 | 规模 | 过滤 | Java 支持 | 适用 |
|--------|------|------|------|-----------|------|
| PGVector | PostgreSQL | 中 | 强（SQL） | JDBC | 已有 PG、中小规模 |
| Redis | Redis Stack | 中 | 中 | Jedis/Lettuce | 高并发、复用缓存 |
| Milvus | 专用 | 超大 | 强 | 官方 SDK | 亿级规模 |
| Qdrant | 专用（Rust） | 大 | 很强 | 官方 SDK | 复杂过滤 |
| Elasticsearch | 搜索引擎 | 大 | 强 | 官方客户端 | 已有 ES、混合检索 |
| Chroma | 专用 | 小 | 弱 | 有社区客户端 | 原型 |
| Pinecone | 托管 | 大 | 中 | 官方 SDK | 免运维 |
| Weaviate | 专用 | 大 | 强 | 官方 SDK | 混合检索、多模态 |

## 3.2 各库画像与适用场景

| 库 | 一句话画像 | 何时优先选 |
|----|------------|------------|
| PGVector | 让 PostgreSQL 长出向量能力 | 已有 PG、数据 < 千万、要事务 |
| Redis | 把向量放进内存缓存 | 已有 Redis、要极低延迟、数据可重建 |
| Milvus | 为向量而生的分布式数据库 | 亿级、要独立扩缩容 |
| Qdrant | 过滤表达力最强的专用库 | 复杂权限、多租户、嵌套过滤 |
| Elasticsearch | 搜索栈顺带做向量 | 已有 ES、要 BM25 + 向量混合 |
| Chroma | 上手最快的原型库 | Demo、教学、本地实验 |
| Pinecone | 完全托管的向量服务 | 团队无运维、要快速上线 |
| Weaviate | 多模态 + 混合检索 | 图文混合、内置向量化模块 |

## 3.3 性能量级参考

以下为数量级直觉，非基准测试结论，实际以压测为准：

| 库 | 千万级延迟量级 | 扩展方式 |
|----|----------------|----------|
| PGVector | 十毫秒~百毫秒 | 垂直 + 读副本 |
| Redis | 亚毫秒~毫秒 | 集群分片 |
| Milvus | 毫秒~十毫秒 | 水平分片 |
| Qdrant | 毫秒~十毫秒 | 分片 + 副本 |
| Elasticsearch | 十毫秒~百毫秒 | 分片 + 副本 |

# 4. 选型决策树

结论先行：决策顺序应为"现有技术栈优先 -> 规模兜底 -> 过滤兜底 -> 托管兜底"。

## 4.1 决策流程

```
已有 PostgreSQL？
   ├── 是 且规模 < 千万 -> PGVector（最省事）
   └── 否
        ├── 已有 Elasticsearch -> ES 向量检索
        ├── 已有 Redis 且需高并发 -> Redis Vector
        ├── 规模上亿 -> Milvus
        ├── 过滤复杂 -> Qdrant
        └── 不想运维 -> Pinecone / 托管服务
```

## 4.2 典型场景映射表

| 场景 | 数据规模 | 推荐 | 理由 |
|------|----------|------|------|
| 内部知识库 | 百万 | PGVector | 复用 PG，运维零增量 |
| SaaS 多租户 | 千万 | Qdrant | 预过滤可靠，隔离清晰 |
| 电商图文搜索 | 亿级 | Milvus | 分布式扩展，多向量 |
| 实时推荐 | 千万 | Redis | 内存延迟极低 |
| 日志 + 语义检索 | 亿级 | Elasticsearch | 已有 ES，混合检索 |
| 快速 Demo | 万级 | Chroma | 单机启动，零配置 |
| 无运维团队 | 任意 | Pinecone | 全托管 |

## 4.3 反例：不推荐的选型

| 反例 | 问题 |
|------|------|
| 千万级用 Chroma | 单机内存瓶颈，无高可用 |
| 多租户用纯后过滤库 | 结果不足、越权风险 |
| 无 ES 却为混合检索上 ES | 引入整套运维负担 |
| 亿级用 PGVector | 索引构建与查询都会成为瓶颈 |
| 数据可重建却上分布式 | 过度设计，成本浪费 |

# 5. 关键概念对照

不同向量库对同一事物的命名不同，跨库迁移时最容易踩坑。结论先行：先建立"容器 -> 记录 -> 距离"三层映射，再看差异。

## 5.1 容器、记录与距离对照

| 概念 | PGVector | Milvus | Qdrant | ES | Redis |
|------|----------|--------|--------|----|-------|
| 顶层容器 | table | collection | collection | index | index |
| 分区 | 表分区 | partition | shard | shard | — |
| 记录 | row | entity | point | document | hash |
| 距离类型 | `<=>` 等 | COSINE/IP/L2 | Cosine/Dot/Euclid | cosine/l2 | COSINE/L2/IP |
| 元数据 | 列 | 标量字段 | payload | fields | hash fields |
| 主键 | 主键列 | primary key | point id | `_id` | key |

## 5.2 概念陷阱

- 距离类型不可混用：写入时归一化的向量与 Cosine 配套；若改用 L2，排序会变。
- Milvus 的 collection 需显式 `loadCollection` 才能查询；Qdrant 的 collection 创建后即可查。
- ES 的 `dense_vector` 字段需在 mapping 中声明维度与相似度，创建后难改。
- PGVector 的索引是"表上的索引"，建索引语句与普通 SQL 一致，容易忘记。

## 5.3 迁移映射示例

```java
// PGVector: 距离运算符
// SELECT id, content FROM kb ORDER BY embedding <=> ? LIMIT 10

// Milvus: 通过 MetricType 表达
SearchParam.newBuilder().withMetricType(MetricType.COSINE).build();

// Qdrant: 通过 Distance 表达
Collections.VectorParams.newBuilder().setDistance(Collections.Distance.Cosine);
```

同一批向量迁移到新库后，务必用固定查询集比对 Top-K 结果是否一致，避免因距离语义差异导致召回漂移。

# 6. ANN 索引类型

结论先行：Flat 精确但慢，HNSW 是通用默认，IVF/PQ 省内存但需权衡召回，DiskANN 面向超大规模。

## 6.1 索引总览

| 索引 | 说明 | 特点 |
|------|------|------|
| Flat | 暴力搜索 | 精确但慢，O(n) |
| IVF | 倒排文件 | 快，需训练，召回与 nprobe 相关 |
| HNSW | 分层图 | 高召回高性能，内存占用大 |
| PQ | 乘积量化 | 压缩，精度损失 |
| DiskANN | 磁盘索引 | 超大规模、低内存 |

```
HNSW 分层结构（简化）：

Layer 2:        A ---------- B
Layer 1:   A --- C --- D --- B
Layer 0: A-C-E-F-D-G-H-B-I-J  (全量节点)
查询从顶层粗定位，逐层下降到 Layer 0 精搜
```

## 6.2 索引选择建议

### 6.2.1 小数据量（< 10 万）

直接用 Flat，精确、无训练、无参数，性能足够。

### 6.2.2 中等数据量（10 万 ~ 千万）

首选 HNSW，参数 `M`、`efConstruction`、`ef` 按召回与延迟调优。

### 6.2.3 大数据量（> 千万 / 内存受限）

考虑 IVF + PQ 组合压缩，或 DiskANN 把索引放磁盘。

## 6.3 索引参数速查

| 参数 | 所属 | 调大效果 |
|------|------|----------|
| `M` | HNSW | 召回更高、内存更大 |
| `efConstruction` | HNSW | 索引更优、构建更慢 |
| `ef` / `efSearch` | HNSW | 召回更高、查询更慢 |
| `nlist` | IVF | 桶更细、训练更久 |
| `nprobe` | IVF | 召回更高、查询更慢 |

## 6.4 索引构建与重建策略

| 策略 | 做法 | 适用 |
|------|------|------|
| 写入即建 | 数据到达即触发建索引 | 数据量小、实时性高 |
| 批量建 | 攒批后统一构建 | 大规模初始导入 |
| 后台重建 | 新索引构建完成后原子切换 | 换索引类型/参数 |
| 灰度切换 | 新旧索引并行，逐步切流 | 高风险变更 |

重建索引通常需要 2 倍资源（新旧并存），规划容量时要预留。若库支持"构建后切换"，优先选择，避免停机。

> 详见 `7_index_algorithms.md`。

# 7. 预过滤 vs 后过滤

结论先行：多租户与权限场景必须使用预过滤（Pre-filter），否则结果数量不可控甚至越权。

## 7.1 机制差异

```
预过滤（Pre-filter）：先按条件筛，再在子集里做向量检索
后过滤（Post-filter）：先向量检索，再按条件筛

后过滤的问题：Top-K 里可能大部分被过滤掉，结果不足
```

假设 Top-K=10，某租户只占全库 0.1%。后过滤时，10 条候选里期望只有 0.01 条属于该租户，结果几乎必然为空。要拿满 10 条，就得把 Top-K 放大到 10000，代价是延迟暴涨。

## 7.2 多租户陷阱与验证

| 策略 | 实现 | 风险 |
|------|------|------|
| 后过滤 | 检索后代码过滤 | 结果不足、越权泄漏 |
| 预过滤 | 库内过滤下推 | 需标量索引支撑 |
| 物理隔离 | 每租户独立集合 | 集合数量爆炸 |
| 分区隔离 | 按租户分区 | 分区数上限 |

验证方法：构造一个只占全库极小比例的租户，检索 Top-10，看是否稳定返回 10 条。若返回 0~2 条，即为后过滤。

## 7.3 过滤下推的实现差异

| 库 | 过滤下推 | 备注 |
|----|----------|------|
| PGVector | 是（SQL 与索引结合） | 依赖查询计划器 |
| Milvus | 是（表达式 + 标量索引） | 过滤字段建议建 INVERTED |
| Qdrant | 是（filter 原生支持） | 过滤即一等公民 |
| Redis | 部分 | 复杂过滤能力有限 |

> 多租户场景务必选择支持**预过滤**的向量库。

# 8. Spring AI 统一抽象

结论先行：Spring AI 的 `VectorStore` 接口屏蔽了底层差异，换库只需改依赖与配置，业务代码不变。

## 8.1 VectorStore 接口

```java
public interface VectorStore {
    void add(List<Document> documents);
    void delete(List<String> ids);
    List<Document> similaritySearch(SearchRequest request);
}
```

`Document` 携带内容与元数据，`SearchRequest` 携带查询文本、Top-K、相似度阈值与过滤表达式。

```java
SearchRequest request = SearchRequest.builder()
        .query("如何申请年假")
        .topK(5)
        .similarityThreshold(0.7)
        .filterExpression("tenant_id == 'acme'")
        .build();

List<Document> docs = vectorStore.similaritySearch(request);
```

## 8.2 多库切换示例

### 8.2.1 PGVector

```java
@Bean
VectorStore vectorStore(JdbcTemplate jdbc, EmbeddingModel model) {
    return PgVectorStore.builder(jdbc, model)
            .dimensions(1536)
            .distanceType(PgVectorDistanceType.COSINE_DISTANCE)
            .indexType(PgVectorIndexType.HNSW)
            .build();
}
```

### 8.2.2 Milvus 与 Qdrant

```java
@Bean
VectorStore vectorStore(EmbeddingModel model) {
    return MilvusVectorStore.builder(milvusClient, model)
            .collectionName("knowledge_base")
            .initializeSchema(true)
            .build();
}

@Bean
VectorStore qdrantVectorStore(EmbeddingModel model) {
    return QdrantVectorStore.builder(qdrantClient, model)
            .collectionName("knowledge_base")
            .initializeSchema(true)
            .build();
}
```

换库只改配置/依赖，注入点 `VectorStore` 保持不变。

## 8.3 自定义 VectorStore 实现要点

若需接入 Spring AI 未覆盖的库，实现 `VectorStore` 时至少注意：

```java
public class MyVectorStore implements VectorStore {

    @Override
    public void add(List<Document> documents) {
        // 1. 调用 EmbeddingModel 批量向量化
        // 2. 组装底层 SDK 的写入参数并批量 upsert
        // 3. 明确写入可见性（是否需要 flush）
    }

    @Override
    public List<Document> similaritySearch(SearchRequest request) {
        // 1. 向量化 request.getQuery()
        // 2. 将 filterExpression 翻译为底层过滤语法
        // 3. 执行 ANN 检索并按 score 排序返回
        throw new UnsupportedOperationException("伪代码");
    }
}
```

## 8.4 批量写入与并发

```java
// 批量向量化，减少往返
List<Document> batch = docs.subList(0, Math.min(64, docs.size()));
vectorStore.add(batch);

// 注意：不同库的批量上限不同，过大可能触发请求体超限
```

写入时优先批量（batch）而非逐条，可显著降低网络与模型调用开销。并发写入要控制线程数，避免压垮向量库的写入通道；若库要求"建索引前先 flush"，需在批次间显式触发。

# 9. LangChain4j 统一抽象

结论先行：LangChain4j 用 `EmbeddingStore` 表达同一件事，粒度更细，`Embedding` 与载体分离。

## 9.1 EmbeddingStore 接口

```java
public interface EmbeddingStore<Embedded> {
    String add(Embedding embedding, Embedded embedded);
    void addAll(List<Embedding> embeddings, List<Embedded> embedded);
    EmbeddingSearchResult<Embedded> search(EmbeddingSearchRequest request);
}
```

```java
EmbeddingStore<TextSegment> store = MilvusEmbeddingStore.builder()
        .host("localhost").port(19530)
        .collectionName("knowledge_base")
        .dimension(1024)
        .build();

store.addAll(embeddings, segments);
EmbeddingSearchResult<TextSegment> result = store.search(
        EmbeddingSearchRequest.builder()
                .queryEmbedding(queryEmbedding)
                .maxResults(5)
                .minScore(0.7)
                .build());
```

## 9.2 与 Spring AI 的差异

| 对比项 | Spring AI | LangChain4j |
|--------|-----------|-------------|
| 核心接口 | `VectorStore` | `EmbeddingStore<Embedded>` |
| 是否自带向量化 | 接口内含 `EmbeddingModel` | 需显式传入 `Embedding` |
| 过滤方式 | `filterExpression` 字符串 | 库特定 API / metadata filter |
| 与框架耦合 | 深度整合 Spring 上下文 | 独立，可脱离 Spring |
| 返回类型 | `Document` | `EmbeddingMatch<Embedded>` |

## 9.3 过滤与元数据

LangChain4j 通过 `Metadata` 携带过滤字段，各库适配器将其翻译为原生过滤语法。注意 `minScore` 的语义是"最小相似度"，不同库对 Cosine/Dot 的打分范围不同，跨库时需重新校准阈值。

## 9.4 检索参数与阈值

| 参数 | 含义 | 调优建议 |
|------|------|----------|
| `maxResults` | 返回条数 | 先大后小，交给重排精筛 |
| `minScore` | 最小相似度 | 用评测集确定，勿拍脑袋 |
| `dynamicFilter` | 动态过滤 | 多租户必用 |

推荐流程：召回阶段 `maxResults=50`、`minScore` 放宽，再用重排器（Reranker）精选 Top-5 交给 LLM。召回宁可多、不可漏。

# 10. 生产注意事项

结论先行：向量库不是"存了就行"，容量、索引、备份、版本每一项都可能引发线上事故。

## 10.1 容量与索引规划

| 风险 | 建议 |
|------|------|
| 规模估算不足 | 按 3 年数据量规划 |
| 索引未建 | 大数据量前建 HNSW |
| 维度不匹配 | 建库前确定维度并全链路统一 |
| 无预过滤 | 多租户场景必选 |
| 内存不足 | 启用量化 / memmap / DiskANN |

## 10.2 可用性、备份与版本

| 风险 | 建议 |
|------|------|
| 单点 | 集群部署、多副本 |
| 无备份 | 向量数据同样要备份、定期演练恢复 |
| 版本不兼容 | 锁定客户端与服务端版本 |
| 写入未刷新 | 明确写后可见性，必要时手动 flush |
| 无监控 | 监控召回率、P99 延迟、内存水位 |

## 10.3 可观测性指标

| 指标 | 含义 | 告警阈值建议 |
|------|------|--------------|
| 召回率 | 评测集命中的比例 | 低于基线 5% 告警 |
| P99 延迟 | 查询尾延迟 | 超过 SLA 告警 |
| QPS | 每秒查询数 | 接近容量上限告警 |
| 内存水位 | HNSW 常驻内存占比 | 超过 80% 告警 |
| 写入积压 | 未落盘/未建索引量 | 持续增长告警 |

## 10.4 上线检查清单

| 检查项 | 通过标准 |
|--------|----------|
| 维度一致性 | 嵌入模型与库维度一致 |
| 过滤字段索引 | 所有常用过滤字段已建索引 |
| 预过滤验证 | 小比例租户可稳定返回 Top-K |
| 备份恢复 | 已完成一次恢复演练 |
| 版本锁定 | 客户端与服务端版本固定 |
| 容量评估 | 已按 3 年数据量估算内存 |
| 监控告警 | 召回率、延迟、内存均有监控 |

# 11. 小结

- 向量库负责存储与 ANN 检索，是 RAG 的基础设施。
- 选型看规模、过滤能力、运维成本与现有技术栈。
- 已有 PG 且规模不大，PGVector 最省事。
- 亿级规模选 Milvus，复杂过滤选 Qdrant。
- 多租户务必选支持预过滤的库。
- Spring AI 与 LangChain4j 都提供了统一抽象，换库成本可控。
- 容量、索引、备份、版本与可观测性是上线前的必查项。
