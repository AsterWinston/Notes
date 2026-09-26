# 1. 模型分类

Embedding 模型按部署形态分为云端 API 与开源本地两大类，选择哪一类决定了成本结构、数据边界与运维复杂度。

## 1.1 云端 API 与开源本地

结论先行：云端 API 省运维、按量付费、数据出境；开源本地可控、零边际成本、需自建推理。

### 1.1.1 云端 API

代表：OpenAI text-embedding-3 系列、Cohere Embed v3、Voyage AI，以及国内的通用大模型厂商 Embedding 接口。

```text
应用 -> HTTPS -> 厂商 Embedding API -> 返回向量
```

优点：开箱即用、持续升级、无需 GPU。
缺点：按量计费、有网络延迟、数据需出网、存在限流。

### 1.1.2 开源本地

代表：BGE（智源）、E5（微软）、GTE（阿里）、M3E / text2vec（中文）、Jina Embeddings。

```text
应用 -> 本地推理(CPU/GPU) -> 返回向量
```

优点：数据不出内网、零边际成本、可微调。
缺点：需要资源与运维、效果可能略逊于顶级云端模型。

## 1.2 分类全景图

```text
Embedding 模型
├── 云端 API
│   ├── OpenAI text-embedding-3-small / large
│   ├── Cohere Embed v3
│   ├── Voyage AI
│   └── 国内：通义、智谱、百川等
└── 开源本地
    ├── BGE（智源）
    ├── E5（微软）
    ├── GTE（阿里）
    ├── M3E / text2vec（中文）
    └── Jina Embeddings
```

## 1.3 边界与选择倾向

| 场景 | 倾向 |
|------|------|
| 快速验证、小规模 | 云端 API |
| 数据敏感、合规要求高 | 开源本地 |
| 大规模、成本敏感 | 本地独立服务 |
| 需要领域微调 | 开源本地 |

# 2. 主流模型对比

结论先行：没有"最好"的模型，只有最适合当前语言、领域与成本约束的模型；选型必须结合自有数据评测。

## 2.1 云端模型

| 模型 | 维度 | 语言 | 特点 |
|------|------|------|------|
| `text-embedding-3-small` | 1536 | 多语言 | 性价比高 |
| `text-embedding-3-large` | 3072 | 多语言 | 效果更好、更贵 |
| Cohere embed-v3 | 1024 | 多语言 | 商业、压缩支持 |

## 2.2 开源模型

| 模型 | 维度 | 语言 | 特点 |
|------|------|------|------|
| BGE-large-zh-v1.5 | 1024 | 中文 | 中文检索强，可本地 |
| BGE-m3 | 1024 | 多语言 | 支持稠密 + 稀疏 + 多向量 |
| E5-large-v2 | 1024 | 多语言 | 需加 `query:`/`passage:` |
| GTE-large-zh | 1024 | 中文 | 阿里开源，中文友好 |
| Jina-embeddings-v3 | 1024 | 多语言 | 长文本、多任务 |

## 2.3 综合对比表

| 模型 | 维度 | 语言 | 部署 | 前缀 | 适用 |
|------|------|------|------|------|------|
| text-embedding-3-small | 1536 | 多语言 | 云端 | 无 | 通用 |
| text-embedding-3-large | 3072 | 多语言 | 云端 | 无 | 高质量 |
| BGE-large-zh-v1.5 | 1024 | 中文 | 本地 | 查询加指令 | 中文检索 |
| BGE-m3 | 1024 | 多语言 | 本地 | 视配置 | 混合检索 |
| E5-large-v2 | 1024 | 多语言 | 本地 | query/passage | 多语言 |
| GTE-large-zh | 1024 | 中文 | 本地 | 无 | 中文检索 |

## 2.4 边界与反例

- 榜单高分模型未必适合你的领域（法律、医疗、代码差异大）；
- 多语言模型在单语任务上未必优于专用单语模型；
- 维度的"更强"不必然带来业务指标提升。

# 3. 选型维度

结论先行：选型应从语言、效果、维度、部署、成本、上下文长度、领域适配七个维度综合权衡，并以自有数据评测收口。

## 3.1 关键维度

| 维度 | 说明 |
|------|------|
| 语言 | 中文优先 BGE/GTE/M3E |
| 效果 | 看 MTEB / C-MTEB 榜单（结合自身领域验证） |
| 维度 | 影响存储与检索成本 |
| 部署 | 云端 API vs 本地推理 |
| 成本 | 按量计费 vs 一次性部署 |
| 上下文长度 | 能编码多长的文本 |
| 领域适配 | 通用 vs 领域微调 |

### 3.1.1 上下文长度的影响

```text
模型最大长度 < chunk 长度 -> 截断 -> 信息丢失
```

因此 chunk 大小必须与模型上下文长度匹配。

### 3.1.2 多语言需求

若语料含中英混合，优先选择多语言对齐良好的模型，避免跨语言检索失效。

## 3.2 评测方法

### 3.2.1 榜单参考

MTEB（Massive Text Embedding Benchmark）与 C-MTEB（中文版）提供通用排名，可作初筛。

### 3.2.2 自有数据评测

用业务真实查询与标注相关文档，计算 Recall@K、MRR、NDCG，作为最终决策依据。

```java
// 伪代码：Recall@K
double recall = hit / relevantTotal;
```

## 3.3 决策流程

```text
语言筛选 -> 榜单初筛 -> 候选 2~3 个 -> 自有数据评测 -> 成本评估 -> 定稿
```

## 3.4 生产建议

> **榜单只能参考**，最终必须用**自己的数据**做检索评测。

# 4. 云端 API 使用

结论先行：云端 API 接入简单，关键是把模型名、维度与密钥集中配置，避免硬编码。

## 4.1 Spring AI

```java
EmbeddingModel model = OpenAiEmbeddingModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .options(OpenAiEmbeddingOptions.builder()
                .model("text-embedding-3-small")
                .dimensions(1024)          // 可选：降维
                .build())
        .build();
```

### 4.1.1 批量调用

```java
List<Embedding> embeddings = model.embedAll(segments).content();
```

### 4.1.2 配置外部化

把 `model`、`dimensions`、`apiKey` 放入配置中心，避免散落在代码中。

## 4.2 LangChain4j

```java
EmbeddingModel model = OpenAiEmbeddingModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("text-embedding-3-small")
        .build();
```

### 4.2.1 向量获取

```java
Embedding embedding = model.embed("如何报销差旅费").content();
float[] vector = embedding.vector();
```

### 4.2.2 超时与重试

```java
// 伪代码：配置超时与指数退避重试
client.withTimeout(Duration.ofSeconds(30));
client.withMaxRetries(3);
```

## 4.3 边界与坑

- API 限流需做退避重试，避免雪崩；
- 不同厂商向量维度与语义不可混用；
- 密钥必须走环境变量或密钥管理，禁止硬编码。

## 4.4 生产建议

对 API 调用做统一封装，集中处理鉴权、重试、超时与埋点。

## 4.5 完整集成示例

结论先行：把 Embedding 封装为单一 Bean，注入向量库与检索服务，避免多处重复构造。

```java
@Configuration
public class EmbeddingConfig {

    @Bean
    public EmbeddingModel embeddingModel(
            @Value("${embedding.api-key}") String apiKey,
            @Value("${embedding.model}") String modelName,
            @Value("${embedding.dimensions}") Integer dimensions) {

        return OpenAiEmbeddingModel.builder()
                .apiKey(apiKey)
                .options(OpenAiEmbeddingOptions.builder()
                        .model(modelName)
                        .dimensions(dimensions)
                        .build())
                .build();
    }
}
```

```java
@Service
public class RetrievalService {

    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;

    public RetrievalService(EmbeddingModel embeddingModel, VectorStore vectorStore) {
        this.embeddingModel = embeddingModel;
        this.vectorStore = vectorStore;
    }

    public List<Document> search(String question, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(topK)
                        .build());
    }
}
```

### 4.5.1 注意事项

- `dimensions` 必须与向量库建库时一致；
- 密钥从配置中心注入，禁止写入代码；
- 检索入口统一，便于埋点与限流。

# 5. 本地模型

结论先行：本地模型数据不出内网、零边际成本，但需要权衡推理资源与运维复杂度；大规模场景推荐独立服务而非进程内推理。

## 5.1 ONNX 推理

### 5.1.1 使用示例

```java
EmbeddingModel model = new OnnxEmbeddingModel(
        Path.of("models/bge-small-zh.onnx"),
        Path.of("models/tokenizer.json"));
```

> 说明：`OnnxEmbeddingModel` 为示意 API，不同库的构造签名不同，以实际依赖为准。

### 5.1.2 优缺点

优点：无网络依赖、数据不出内网、零边际成本。
缺点：需 GPU/CPU 资源、效果可能略逊。

## 5.2 独立服务

### 5.2.1 架构

```text
应用 -> HTTP -> Embedding 服务（GPU）
              （TEI / vLLM / Infinity / 自建）
```

### 5.2.2 方案对比

| 方案 | 说明 |
|------|------|
| HuggingFace TEI | 高性能推理服务 |
| vLLM | 支持 embedding 模型 |
| Infinity | 专为 embedding 优化 |

## 5.3 边界与坑

- 进程内 ONNX 难以水平扩展，多实例会重复占用显存；
- 独立服务需处理限流、多副本与健康检查；
- CPU 推理吞吐低，需评估延迟是否达标。

## 5.4 生产建议

> 生产大规模场景，**独立部署 Embedding 服务**比进程内 ONNX 更易扩展。

# 6. 前缀与查询/文档区分

结论先行：许多模型对输入格式有严格要求，查询与文档需区别处理；不加前缀会显著降低检索效果。

## 6.1 前缀要求

### 6.1.1 BGE

```java
// BGE（中文）：查询需加指令前缀，文档不加
String query = "为这个句子生成表示以用于检索相关文章：" + question;
String doc = document;
```

### 6.1.2 E5

```java
// E5：query/passage 前缀
String query = "query: " + question;
String doc = "passage: " + document;
```

## 6.2 非对称封装

```java
public class AsymmetricEmbedding {

    private final EmbeddingModel model;

    public Embedding embedQuery(String query) {
        return model.embed("query: " + query).content();
    }

    public Embedding embedDocument(String doc) {
        return model.embed("passage: " + doc).content();
    }
}
```

## 6.3 边界与坑

- 前缀属于模型契约，换模型必须同步更换；
- 文档误加查询前缀会降低效果；
- 部分模型（OpenAI v3、GTE）不需要前缀，乱加反而有害。

## 6.4 生产建议

> **不加前缀会显著降低检索效果**，这是常见坑。

把前缀策略配置化，并在启动时对已知模型做校验。

## 6.5 端到端示例

结论先行：查询与文档走两条不同路径，但最终写入同一个向量空间。

```java
public class RagIndexer {

    private final AsymmetricEmbedding embedding;

    public RagIndexer(AsymmetricEmbedding embedding) {
        this.embedding = embedding;
    }

    public void index(List<String> docs, VectorStore store) {
        for (String doc : docs) {
            store.add(List.of(embedding.embedDocument(doc)));
        }
    }

    public List<Embedding> query(String question) {
        return List.of(embedding.embedQuery(question));
    }
}
```

### 6.5.1 一致性校验

在启动时对同一段文本分别走查询与文档路径，确认前缀策略与模型配置生效。

# 7. 维度选择与降维

结论先行：维度决定存储与检索成本，可借助 Matryoshka 降维在几乎不损失效果的前提下压缩向量。

## 7.1 维度决策

| 维度区间 | 建议 |
|----------|------|
| 384~512 | 边缘、低延迟 |
| 768~1024 | 通用默认 |
| 1536~3072 | 高质量、资源充足 |

## 7.2 Matryoshka 降维

OpenAI v3 支持 Matryoshka（套娃）降维：

```java
OpenAiEmbeddingOptions.builder()
        .model("text-embedding-3-large")
        .dimensions(1024)          // 从 3072 降到 1024
        .build();
```

好处：存储与检索成本降低，效果损失有限。

### 7.2.1 降维后的注意点

降维改变了向量分布，阈值必须重新校准；评测必须基于降维后的向量进行。

## 7.3 边界与坑

- 并非所有模型支持任意降维，需按官方说明；
- 降维后不同维度间不可混用；
- 降维省下的是存储与检索，模型推理成本不变。

## 7.4 生产建议

若采用降维，应在元数据中记录实际维度，防止后续误用。

# 8. 模型切换的代价

结论先行：更换 Embedding 模型等于让所有历史向量失效，必须全量重新向量化并重建索引，代价极高。

## 8.1 为什么代价高

```text
换 Embedding 模型 = 所有向量失效 = 全量重新向量化 + 重建索引
```

不同模型的向量空间彼此独立，旧向量与新查询向量不可比，因此必须整体重算。

## 8.2 平滑切换方案

### 8.2.1 新索引 + 别名

```text
建新索引 -> 全量重算 -> 评测通过 -> 别名切换到新索引 -> 删除旧索引
```

用索引别名（alias）对外提供稳定名称，切换时只改别名，避免服务中断。

### 8.2.2 双写与灰度

```text
阶段一：双写新旧向量
阶段二：灰度切查询
阶段三：观察指标 -> 全量切换
```

## 8.3 边界与坑

- 切换期间需同时承担新旧两套成本；
- 回滚必须可行，旧索引在确认前不能删除；
- 切换后必须重跑检索评测。

## 8.4 生产建议

因此：

- 建库前充分选型；
- 预留"模型版本"元数据；
- 切换时用**新索引 + 别名切换**，避免服务中断；
- 切换后必须跑检索评测。

# 9. 生产注意事项

结论先行：把模型视为基础设施，用配置、元数据与监控固化其契约，才能安全演进。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 只看榜单 | 用自己的数据评测 |
| 漏加前缀 | 按模型要求加 query/passage |
| 本地模型无 GPU | 评估吞吐，必要时独立服务 |
| 频繁换模型 | 代价极高，提前规划 |
| 无版本记录 | 元数据记录模型版本 |
| 单点 Embedding 服务 | 多副本 + 限流 |

## 9.2 上线检查表

```text
[ ] 模型选型已用自有数据评测
[ ] 前缀策略已配置并校验
[ ] 维度与降维已确定
[ ] 模型版本已写入元数据
[ ] 切换方案与回滚已演练
```

## 9.3 监控与告警

| 指标 | 含义 | 异常信号 |
|------|------|----------|
| 调用成功率 | 稳定性 | 下降需排查 |
| P99 延迟 | 性能 | 上升影响体验 |
| 限流次数 | 容量 | 频繁需扩容 |
| 检索命中率 | 质量 | 下降需重评 |

## 9.4 成本与容量规划

结论先行：容量规划应同时考虑向量化吞吐、存储增长与索引重建窗口。

### 9.4.1 规划要点

| 项 | 估算方式 |
|----|----------|
| 存储 | 向量数 x 维度 x 4 字节 x 冗余 |
| 向量化吞吐 | 峰值文档增速 / 单副本吞吐 |
| 重建窗口 | 全量向量化耗时 |
| 成本 | 单位 Token 单价 x 总量 |

### 9.4.2 增长预留

按未来 12~24 个月的数据增长预留存储与显存，避免频繁扩容导致索引重建。

# 10. 小结

结论先行：模型选型是 RAG 的基石决策，应结合语言、领域、成本与部署约束，用自有数据评测收口，并规划好切换路径。

## 10.1 核心要点

- 模型分云端 API 与开源本地两类。
- 中文优先 BGE/GTE，多语言看 E5/Jina。
- 榜单只作参考，必须用自有数据评测。
- 注意模型的前缀与查询 / 文档区分要求。
- 换模型代价极高，需提前规划与别名切换。

## 10.2 决策清单

| 决策 | 建议 |
|------|------|
| 部署 | 敏感数据选本地 |
| 模型 | 中文优先 BGE/GTE |
| 维度 | 768~1024 起步 |
| 前缀 | 按模型要求 |
| 切换 | 新索引 + 别名 |
| 验证 | 自有评测集 |
