# 1. RAG 概览与组件

## 1.1 结论先行

**Quarkus LangChain4j 的 RAG（Retrieval-Augmented Generation，检索增强生成）完全复用 LangChain4j 的组件模型，Quarkus 扩展只负责把向量库、Embedding 模型、检索器装配为 CDI Bean，并通过注解接入 AI Service。**

## 1.2 组件链路

### 1.2.1 链路图

```
问题 ──► QueryTransformer（可选改写）
   │
   ▼
EmbeddingModel ──► EmbeddingStore（向量检索）
   │
   ▼
ContentRetriever（过滤/排序/取 Top-K）
   │
   ▼
ContentInjector ──► 拼装进 Prompt ──► LLM ──► 回答
```

### 1.2.2 组件表

| 组件 | 职责 | 是否必需 |
|------|------|----------|
| EmbeddingModel | 文本转向量 | 是 |
| EmbeddingStore | 向量存取 | 是 |
| ContentRetriever | 检索与过滤 | 是 |
| QueryTransformer | 查询改写 | 否 |
| ContentInjector | 上下文注入 | 是 |
| RetrievalAugmentor | 串联以上 | 是 |

### 1.2.3 离线与在线

| 阶段 | 操作 | 触发 |
|------|------|------|
| 离线 | 解析、切分、向量化、入库 | 导入任务 |
| 在线 | 检索、注入、生成 | 用户请求 |

## 1.3 生产建议

- 离线导入与在线检索解耦，导入走异步任务。
- 把检索质量与生成质量分开度量，便于定位问题。

# 2. 向量库配置

## 2.1 pgvector 配置

### 2.1.1 properties

```properties
# 使用 pgvector
quarkus.langchain4j.pgvector.datasource.url=jdbc:postgresql://localhost:5432/ai
quarkus.langchain4j.pgvector.datasource.username=postgres
quarkus.langchain4j.pgvector.datasource.password=${DB_PASSWORD}
quarkus.langchain4j.pgvector.table=embeddings
quarkus.langchain4j.pgvector.dimension=1536
```

### 2.1.2 表结构要点

| 列 | 说明 |
|----|------|
| embedding | 向量列，维度须与模型一致 |
| text | 原文片段 |
| metadata | JSON，用于过滤 |

维度必须与 Embedding 模型输出一致，否则写入失败。

## 2.2 其他向量库

### 2.2.1 对比表

| 向量库 | 扩展 | 适用 |
|--------|------|------|
| pgvector | quarkus-langchain4j-pgvector | 已有 Postgres |
| Redis | quarkus-langchain4j-redis | 低延迟 |
| 内存 | 内置 | 测试 |
| Milvus / Qdrant | 对应扩展 | 大规模 |

### 2.2.2 边界与坑

- 更换 Embedding 模型必须重建全部向量，维度或语义空间会变化。
- 内存向量库在重启后丢失，禁止用于生产。

## 2.3 生产建议

- 向量库独立部署或使用独立 schema，避免与业务库争抢资源。
- 为向量列建立合适索引（如 HNSW），并定期评估召回率。

## 2.4 索引与性能

### 2.4.1 索引类型

| 索引 | 特点 | 适用 |
|------|------|------|
| 精确（Flat） | 全量比对，准 | 小规模 |
| HNSW | 近似，快 | 大规模 |
| IVF | 分桶近似 | 超大规模 |

### 2.4.2 边界与坑

- 近似索引以召回率换速度，需评估 Recall@K。
- 写入量大时索引维护成本高，应批量导入。

## 2.5 生产建议

- 导入与查询分离到不同连接池，避免互相拖累。
- 定期重建索引，控制碎片。

# 3. 声明式 RAG

## 3.1 最小配置

### 3.1.1 代码

```java
@RegisterAiService(
        retrievalAugmentor = DefaultRetrievalAugmentor.class)
public interface KnowledgeAssistant {

    @SystemMessage("""
            只依据检索到的资料回答，资料不足时说明。
            """)
    String ask(String question);
}
```

### 3.1.2 DefaultRetrievalAugmentor

`DefaultRetrievalAugmentor` 是 LangChain4j 提供的默认实现，自动装配检索器与注入器。指定它即启用 RAG 流程。

```
ask(question)
  → DefaultRetrievalAugmentor
      → ContentRetriever.retrieve(question)
      → ContentInjector.inject(contents, userMessage)
  → ChatLanguageModel
```

## 3.2 检索参数

### 3.2.1 参数表

| 参数 | 含义 | 建议 |
|------|------|------|
| maxResults | 返回片段数 | 3~8 |
| minScore | 最低相似度 | 0.6~0.8 |
| dynamicFilter | 动态过滤 | 多租户必设 |

### 3.2.2 边界与坑

- `maxResults` 过大导致上下文超限与噪声增加。
- `minScore` 过高会漏召回，过低会引入无关内容。

## 3.3 生产建议

- 参数以评测集调优，而非凭感觉设定。
- 记录每次检索命中的片段 id，便于溯源与反馈。

## 3.4 提示词设计

### 3.4.1 约束模板

```
只依据检索到的资料回答。
若资料不足，回答"资料中未提及"。
引用时标注来源编号。
```

### 3.4.2 引用来源

在注入上下文时为每个片段编号，要求模型回答时引用，便于人工核验。

### 3.4.3 边界与坑

- 无约束时模型会混入自身知识，产生幻觉。
- 引用格式需与注入格式一致，否则无法解析。

# 4. 自定义检索器

## 4.1 ContentRetriever

### 4.1.1 代码

```java
@ApplicationScoped
public class RetrieverConfig {

    @Produces
    @Singleton
    public ContentRetriever contentRetriever(
            EmbeddingStore<TextSegment> store,
            EmbeddingModel model) {
        return EmbeddingStoreContentRetriever.builder()
                .embeddingStore(store)
                .embeddingModel(model)
                .maxResults(5)
                .minScore(0.7)
                .build();
    }
}
```

### 4.1.2 builder 参数

| 方法 | 作用 |
|------|------|
| embeddingStore | 指定向量库 |
| embeddingModel | 指定向量化模型 |
| maxResults | Top-K |
| minScore | 阈值 |
| dynamicFilter | 动态元数据过滤 |

## 4.2 高级检索

### 4.2.1 查询转换

对口语化问题先改写为检索友好查询，可显著提升召回。

```
原始问题 ──► QueryTransformer ──► 多个检索查询
```

### 4.2.2 重排序

检索后用 Cross-Encoder 或 LLM 对候选重排，提升精度但增加延迟。

### 4.2.3 边界与坑

- 多路查询会增加向量库压力与成本。
- 重排序引入额外模型调用，需评估端到端延迟。

## 4.3 生产建议

- 先做基线（纯向量检索），再逐项加转换与重排。
- 用 A/B 评测量化每项增强的收益。

# 5. 文档导入

## 5.1 导入流程

### 5.1.1 链路

```
文件 ──► DocumentParser ──► Document
   ──► DocumentSplitter ──► List<TextSegment>
   ──► EmbeddingModel.embedAll ──► List<Embedding>
   ──► EmbeddingStore.addAll
```

### 5.1.2 代码

```java
@ApplicationScoped
public class IngestService {

    @Inject
    EmbeddingStore<TextSegment> store;

    @Inject
    EmbeddingModel embeddingModel;

    public int ingest(Path file, String tenantId) {
        DocumentParser parser = new ApachePdfBoxDocumentParser();
        Document document = FileSystemDocumentLoader.loadDocument(file, parser);

        document.metadata().put("tenantId", tenantId);
        document.metadata().put("source", file.getFileName().toString());

        DocumentSplitter splitter = DocumentSplitters.recursive(800, 100);
        List<TextSegment> segments = splitter.split(document);

        List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
        store.addAll(embeddings, segments);

        return segments.size();
    }
}
```

## 5.2 切分策略

### 5.2.1 参数表

| 参数 | 含义 | 建议 |
|------|------|------|
| maxSegmentSize | 片段上限（字符/Token） | 500~1000 |
| maxOverlapSize | 重叠 | 10%~20% |
| 分隔符 | 递归切分依据 | 段落/句 |

### 5.2.2 边界与坑

- 切分过碎丢失上下文，过大引入噪声并浪费预算。
- 表格与代码需专门处理，按字符切分会破坏结构。

## 5.3 元数据

### 5.3.1 字段设计

| 字段 | 用途 |
|------|------|
| tenantId | 多租户隔离 |
| source | 溯源 |
| version | 版本控制 |
| createdAt | 时效管理 |

### 5.3.2 生产建议

- 元数据是过滤与治理的基础，导入时一次性写全。
- 支持按 source 删除，便于文档更新时清理旧向量。

## 5.4 增量与更新

### 5.4.1 按 source 重建

```java
store.removeAll(metadataKey("source").isEqualTo(fileName));
ingestService.ingest(path, tenantId);
```

### 5.4.2 边界与坑

- 先删后写存在短暂空窗，需评估一致性要求。
- 并发导入同一文档可能产生重复向量。

## 5.5 生产建议

- 导入任务幂等，重复执行不产生重复数据。
- 为导入建立批次记录，支持回滚。

# 6. 多租户过滤

## 6.1 动态过滤

### 6.1.1 代码

```java
EmbeddingStoreContentRetriever.builder()
        .embeddingStore(store)
        .embeddingModel(model)
        .dynamicFilter(query -> metadataKey("tenantId")
                .isEqualTo(TenantContext.current()))
        .build();
```

### 6.1.2 机制

每次检索时根据当前请求上下文生成过滤条件，向量库在检索阶段即隔离数据。

```
请求 ──► TenantContext.current() = "t-42"
   ──► filter: tenantId == "t-42"
   ──► 仅返回该租户片段
```

## 6.2 安全边界

### 6.2.1 越权风险

| 风险 | 后果 |
|------|------|
| 未设过滤 | 跨租户数据泄露 |
| 过滤值来源不可信 | 伪造租户越权 |
| 缓存未按租户分区 | 串数据 |

### 6.2.2 生产建议

- 租户标识只能来自服务端认证上下文，绝不能来自请求参数。
- 对检索结果做二次校验，形成纵深防御。

# 7. 原生镜像的 RAG

## 7.1 兼容性挑战

### 7.1.1 问题清单

```
注意：原生镜像中：
- 文档解析器可能依赖反射 → 需配置
- ONNX Embedding 模型可能不兼容 → 用远程 Embedding
- 向量库客户端需检查兼容性
```

### 7.1.2 规避配置

```properties
# 原生镜像中排除有问题的解析器
quarkus.langchain4j.document-parser.tika.enabled=false
```

| 组件 | 原生风险 | 规避 |
|------|----------|------|
| Tika 解析器 | 反射/资源 | 关闭或注册 |
| ONNX 本地 Embedding | 本地库 | 用远程 API |
| 向量库客户端 | gRPC/原生库 | 选 HTTP 客户端 |

## 7.2 生产建议

- 在 CI 中对每个解析器与向量库做原生冒烟测试。
- 优先选择纯 Java 或 HTTP 协议的组件，降低原生风险。

## 7.3 替代方案

### 7.3.1 远程服务化

把 Embedding 与解析放到独立 JVM 服务，主应用保持原生镜像精简。

```
原生应用 ──HTTP──► Embedding 服务（JVM）
        ──HTTP──► 向量库（pgvector）
```

### 7.3.2 边界与坑

- 远程化引入网络延迟与新的故障点。
- 需为远程 Embedding 服务设置超时与降级。

## 7.4 生产建议

- 若原生兼容成本过高，退回 JVM 模式仍是合理选择。
- 把兼容性验证结果沉淀为团队文档。

# 8. 生产注意事项

## 8.1 风险清单

| 风险 | 建议 |
|------|------|
| 原生镜像兼容性 | 逐个测试依赖 |
| 租户越权 | 元数据过滤 |
| 无评测 | 建黄金集 |
| 上下文超限 | Token 预算 |
| 向量陈旧 | 重建与清理 |
| 检索无溯源 | 记录片段 id |

## 8.2 落地清单

- [ ] 向量维度与模型一致
- [ ] 导入写全元数据
- [ ] 动态过滤接入认证上下文
- [ ] 建立黄金评测集
- [ ] 设置 Token 预算
- [ ] 原生镜像冒烟测试

## 8.3 边界与坑

- 上下文超限时模型会静默截断，回答质量下降但无报错。
- 文档更新后旧向量未删除，会导致新旧内容同时命中。

## 8.4 生产建议

- 把 RAG 质量纳入持续评测，而非上线即完。
- 对检索与生成分别设 SLO。

# 9. 小结

## 9.1 要点回顾

- Quarkus LangChain4j 复用 LangChain4j 的 RAG 组件。
- 用 `@RegisterAiService` 声明 RAG。
- 向量库通过配置接入。
- 多租户用动态过滤。
- 原生镜像需注意依赖兼容性。
- 导入要写全元数据，评测与溯源是生产必备。

## 9.2 落地行动

- 建立黄金评测集，量化召回与生成质量。
- 为检索与生成分别设置 SLO 与 Token 预算。
