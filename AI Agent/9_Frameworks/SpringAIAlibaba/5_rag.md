# 1. RAG 总览

## 1.1 复用 Spring AI 的 RAG 能力

结论先行：Spring AI Alibaba 不自造 RAG 轮子，而是直接复用 Spring AI 的检索增强生成（RAG，Retrieval-Augmented Generation）组件，只在中文与国内生态上做增强。

```
Spring AI Alibaba 的 RAG 构建在 Spring AI 之上：
  DocumentReader / Splitter / VectorStore / Advisor
```

机制：框架把"读取 → 切分 → 向量化 → 存储 → 检索 → 拼上下文 → 生成"这条流水线拆成可替换组件，业务只需组合。

```java
@Bean
ChatClient ragChatClient(ChatModel model, VectorStore vectorStore) {
    return ChatClient.builder(model)
            .defaultSystem("""
                    你是企业知识库助手。
                    只依据检索到的资料回答，资料不足时说明。
                    """)
            .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                    .searchRequest(SearchRequest.builder()
                            .topK(5)
                            .similarityThreshold(0.7)
                            .build())
                    .build())
            .build();
}
```

## 1.2 RAG 流水线拆解

```
文档 ─▶ Reader ─▶ Splitter ─▶ Embedding ─▶ VectorStore
                                               │
用户问题 ─▶ Embedding ─▶ 相似度检索 ───────────┘
                              │
                              ▼
                    拼装上下文 ─▶ LLM ─▶ 带引用的答案
```

| 阶段 | 组件 | 关键参数 |
|------|------|----------|
| 读取 | DocumentReader | 解析格式 |
| 切分 | TextSplitter | chunkSize/overlap |
| 向量化 | EmbeddingModel | 维度 |
| 存储 | VectorStore | 索引类型 |
| 检索 | SearchRequest | topK/threshold |
| 生成 | ChatClient | 提示词模板 |

## 1.3 与纯 Prompt 的对比

| 维度 | 纯 Prompt | RAG |
|------|-----------|-----|
| 知识来源 | 模型参数 | 外部知识库 |
| 时效性 | 训练截止 | 实时可更新 |
| 可溯源 | 无 | 有引用 |
| 成本 | 低 | 检索 + 生成 |

> RAG 的本质是"用检索把外部知识塞进上下文"，绕开微调的高成本。

## 1.4 何时不该用 RAG

- 知识极少且稳定：直接写进系统提示更简单；
- 需要改变模型"说话风格"：微调更合适；
- 要求强推理而非查资料：RAG 帮不上；
- 检索质量无保障：上 RAG 只会放大错误。

# 2. 检索链路组件

## 2.1 DocumentReader（读取）

结论先行：Reader 负责把异构文档（PDF、Word、HTML、Markdown）解析成统一的 `Document`。

```java
// 伪代码：把目录下的文档读入
List<Document> docs = new DirectoryReader(resourceDir).read();
```

- 边界：扫描件需 OCR，纯 Reader 拿不到文字；
- 生产建议：解析失败要单独落盘记录，避免静默丢文档。

### 2.1.1 常见格式与陷阱

| 格式 | 陷阱 |
|------|------|
| PDF | 双栏、表格错乱 |
| Word | 批注/修订混入 |
| HTML | 导航噪声 |
| Markdown | 层级丢失 |

- 生产建议：先清洗（去页眉页脚、去导航），再切分。

## 2.2 TextSplitter（切分）

### 2.2.1 切分策略

机制：切分要兼顾"语义完整"与"检索粒度"。

| 策略 | 特点 | 适用 |
|------|------|------|
| 固定长度 | 简单 | 无结构文本 |
| 按段落 | 语义好 | 说明文档 |
| 递归字符 | 平衡 | 通用 |
| 按标题 | 层级清晰 | 手册/规范 |

- chunk 太大：噪声多、Token 贵；
- chunk 太小：语义断裂、召回差；
- 需设置 overlap（重叠）防止句子被切断。

### 2.2.2 中文切分

- 中文没有空格，按英文空格切分会失效；
- 优先按标点（。！？；）与段落切；
- 需要时叠加中文分词（如按词边界）提升检索质量。

```java
// 伪代码：中文友好的递归切分
TextSplitter splitter = new RecursiveCharacterTextSplitter(
        800, 100, List.of("\n\n", "。", "！", "？", "\n", " "));
```

## 2.3 VectorStore（向量库）

```java
@Bean
VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return new SimpleVectorStore(embeddingModel);
}
```

- 开发期可用内存实现，生产期换持久化向量库；
- 写入时携带元数据（tenantId、来源、时间），供过滤与溯源。

### 2.3.1 元数据设计

| 字段 | 用途 |
|------|------|
| tenantId | 租户隔离 |
| source | 答案溯源 |
| updatedAt | 时效过滤 |
| docType | 分类过滤 |

- 元数据字段应"可过滤、低基数"，避免高基数拖慢检索；
- 生产建议：把元数据 schema 固化，禁止随意新增字段。

## 2.4 Advisor（问答增强）

机制：`QuestionAnswerAdvisor` 在每次调用前自动检索并把结果拼进提示词，业务代码无需手写检索。

- 优点：与 `ChatClient` 无缝集成，横切关注点内聚；
- 边界：Advisor 是"单次调用"级别，多步编排需用 Graph；
- 生产建议：把 `topK` 与 `similarityThreshold` 做成可配置项。

# 3. 国内场景增强

## 3.1 增强清单

结论先行：Spring AI Alibaba 针对中文与国内云生态，在四个点上做了增强。

| 增强 | 说明 |
|------|------|
| 中文 Embedding | 通义 text-embedding-v3 |
| 中文分词 | 适配中文检索 |
| 国内向量库 | 阿里云向量检索服务 |
| 百炼知识库 | 托管知识库能力 |

## 3.2 中文 Embedding

### 3.2.1 配置

```yaml
spring:
  ai:
    dashscope:
      embedding:
        options:
          model: text-embedding-v3
```

> 中文场景用中文优化的 Embedding 模型效果更好。

### 3.2.2 维度与成本

| 维度档 | 表达力 | 存储 | 适用 |
|:------:|:------:|:----:|------|
| 低 | 弱 | 省 | 粗召回 |
| 高 | 强 | 贵 | 高精度 |

- 更换 Embedding 模型必须全量重建索引；
- 计费按输入 Token，缓存重复文本可省钱。

## 3.3 国内向量库

- 阿里云向量检索服务：托管、免运维、与百炼同生态；
- 边界：数据在云端，需评估合规；
- 生产建议：本地开发用内存库，生产切托管库，接口保持一致。

### 3.3.1 向量库选型对比

| 类型 | 运维 | 成本 | 适用 |
|------|------|------|------|
| 内存 | 无 | 低 | 开发/测试 |
| 自建 | 高 | 中 | 强合规 |
| 云托管 | 低 | 随量 | 生产 |

- 边界：不同库的相似度算法与距离度量不同，切换后要重评效果；
- 生产建议：用统一 `VectorStore` 抽象隔离实现，便于迁移。

# 4. 混合检索

## 4.1 稠密 vs 稀疏

结论先行：稠密（向量）擅长语义，稀疏（关键词）擅长精确匹配，二者互补。

| 类型 | 原理 | 强项 | 弱项 |
|------|------|------|------|
| 稠密 | 向量相似度 | 语义泛化 | 专名/数字 |
| 稀疏 | BM25 词频 | 精确命中 | 同义改写 |

机制：混合检索同时跑两路，再融合排序，兼顾语义与精确。

## 4.2 RRF 融合

### 4.2.1 融合算法

RRF（Reciprocal Rank Fusion，倒数排名融合）按"排名的倒数"加权求和：

```
score(d) = Σ  1 / (k + rank_i(d))
          i
```

- k 是平滑常数（常取 60），防止头部排名过于主导；
- 只依赖排名，不依赖各路分数量纲，天然可融合异构检索。

### 4.2.2 代码实现

```java
// 稠密 + 稀疏（中文 BM25）
public List<Document> hybridSearch(String question) {
    List<Document> dense = vectorStore.similaritySearch(
            SearchRequest.builder().query(question).topK(20).build());

    List<Document> sparse = bm25Search(question, 20);

    return rrfFuse(dense, sparse);
}
```

```java
// 伪代码：RRF 融合
List<Document> rrfFuse(List<Document> a, List<Document> b) {
    Map<String, Double> score = new HashMap<>();
    accumulate(score, a);
    accumulate(score, b);
    return sortByScoreDesc(score);
}
```

## 4.3 重排序（Rerank）

- 粗排（召回）取回 20~50 条，精排（Rerank）再打分取前 5；
- Rerank 模型比向量检索更贵，但显著提升相关性；
- 边界：Rerank 增加延迟，需评估端到端 SLA。

## 4.4 边界与坑

- 边界：混合检索的两路 topK 若悬殊，RRF 可能被单路主导；
- 边界：BM25 分词器与建索引分词器必须一致，否则匹配异常；
- 生产建议：融合权重与 Rerank 开关做成可调参数，便于 A/B。

# 5. 百炼知识库

## 5.1 托管能力

```
百炼平台提供托管的知识库能力：
  ├── 文档上传与解析
  ├── 自动切分与向量化
  ├── 检索 API
  └── 与应用编排集成

优势：免运维
劣势：数据在云端，需评估合规
```

## 5.2 接入方式

| 方式 | 说明 |
|------|------|
| 控制台 | 手工上传文档 |
| OpenAPI | 程序化上传与检索 |
| 应用编排 | 与提示词、工具串联 |

## 5.3 优势与劣势

- 优势：免运维、开箱即用、与百炼应用打通；
- 劣势：切分与检索策略不可细调、数据出境合规、成本随量增长；
- 生产建议：核心知识库自建以保证可控，边缘场景用托管。

### 5.3.1 与自建知识库对比

| 维度 | 百炼托管 | 自建 |
|------|----------|------|
| 运维 | 免 | 高 |
| 可控性 | 低 | 高 |
| 合规 | 云端 | 可内网 |
| 集成 | 与应用编排打通 | 需自研 |

- 边界：托管侧切分策略不可见，效果调优空间有限；
- 生产建议：先用托管验证价值，再决定是否迁移自建。

# 6. 完整 RAG 示例

## 6.1 依赖与配置

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      embedding:
        options:
          model: text-embedding-v3
```

## 6.2 服务实现

### 6.2.1 检索（带租户过滤）

```java
@Service
@RequiredArgsConstructor
public class KnowledgeService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;

    public String ask(String question, UserContext user) {
        // ① 检索（带租户过滤）
        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(5)
                        .filterExpression("tenantId == '" + user.tenantId() + "'")
                        .build());
```

### 6.2.2 上下文组装

```java
        // ② 组装上下文
        String context = docs.stream()
                .map(Document::getText)
                .collect(Collectors.joining("\n---\n"));
```

### 6.2.3 生成与引用

```java
        // ③ 生成
        return chatClient.prompt()
                .user("""
                        依据资料回答，资料不足时说明。
                        资料：
                        %s
                        问题：%s
                        """.formatted(context, question))
                .call().content();
    }
}
```

- 边界：拼接过长会超上下文，需做 Token 预算与截断；
- 生产建议：把命中的来源 ID 一并返回，支持答案溯源。

## 6.3 边界与坑

- 边界：`filterExpression` 用字符串拼接有注入风险，租户值应来自可信上下文；
- 边界：检索为空时不能让模型"编"，必须显式兜底；
- 生产建议：记录每次检索的 query、命中 ID、耗时，便于排查。

# 7. 与 Graph 结合

## 7.1 RAG 作为节点

结论先行：当 RAG 需要条件分支、循环或多步协作时，把它拆成 Graph 节点比 Advisor 更可控。

```java
// RAG 作为 Graph 的一个节点
StateGraph graph = new StateGraph(AgentState.class)
        .addNode("retrieve", this::retrieveNode)
        .addNode("rerank", this::rerankNode)
        .addNode("generate", this::generateNode)
        .addEdge(START, "retrieve")
        .addEdge("retrieve", "rerank")
        .addEdge("rerank", "generate")
        .addEdge("generate", END)
        .compile();
```

## 7.2 检索-重排-生成流水线

```
START ─▶ retrieve ─▶ rerank ─▶ generate ─▶ END
           │            │           │
        向量+BM25     精排取前5    带引用生成
```

- 每个节点职责单一，便于单独替换与测试；
- 可在 rerank 后加"质量校验"条件边，不合格回退 retrieve。

## 7.3 边界与坑

- 边界：Graph 的节点共享状态，检索结果作为字段传递要控制体积；
- 边界：循环重检索必须有次数上限，防止烧钱；
- 生产建议：节点级打点，定位是"检索差"还是"生成差"。

# 8. 评测与优化

## 8.1 黄金集

- 收集真实问题 + 标准答案 + 应命中文档；
- 覆盖简单、多跳、无答案三类场景；
- 边界：黄金集需定期扩充，防止过拟合。

### 8.1.1 黄金集规模与来源

| 来源 | 说明 |
|------|------|
| 线上日志 | 真实分布 |
| 客服工单 | 高频问题 |
| 专家构造 | 边界用例 |

- 规模建议：起步 100~300 条，覆盖主要意图；
- 边界：黄金集标注本身有主观性，需多人交叉校验。

## 8.2 关键指标

| 指标 | 含义 |
|------|------|
| 召回率 | 应命中的是否召回 |
| 精确率 | 召回的是否相关 |
| 答案正确率 | 最终答案对错 |
| 忠实度 | 是否忠于资料 |
| 延迟 | 端到端耗时 |

## 8.3 迭代方法

1. 先修检索（召回不行，生成再好也没用）；
2. 再修提示词与上下文格式；
3. 最后考虑 Rerank 与模型升级；
4. 每次改动跑黄金集，避免拍脑袋。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 中文效果差 | 用中文优化模型 |
| 租户越权 | 元数据强制过滤 |
| 上下文超限 | Token 预算 |
| 无评测 | 建黄金集 |
| 数据合规 | 评估云端知识库 |

## 9.2 安全与合规

- 过滤条件必须在服务端强制注入，禁止由前端传入；
- 敏感知识库评估是否允许上云；
- 检索日志脱敏，避免泄露用户问题与文档内容。

## 9.3 Token 预算

```
总预算 = 系统提示 + 检索上下文 + 历史 + 输出预留
检索上下文占比建议 ≤ 60%
```

- 超预算时优先裁剪低分文档；
- 生产建议：对每次请求记录 Token 消耗，设置租户配额。

# 10. 小结

## 10.1 核心要点

- Spring AI Alibaba 复用 Spring AI 的 RAG 组件。
- 增强点：中文 Embedding、国内向量库、百炼知识库。
- 中文场景用中文优化的 Embedding。
- RAG 可作为 Graph 的节点。

## 10.2 落地清单

- 租户过滤与 Token 预算是必备。
- 建立黄金集，先修检索再修生成。
