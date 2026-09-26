# 1. 模块定位

RAG（Retrieval-Augmented Generation，检索增强生成）是企业落地 AI 最普遍、最刚需的形态：让大模型基于**私有知识**回答问题，而不必微调。本模块覆盖从文档处理、向量化、检索、重排到评测的完整链路。

## 1.1 是什么

RAG 在生成前先"查资料"，把检索到的相关片段作为上下文喂给大模型。它把"知识"从模型参数里解耦出来，放到可随时更新的外部知识库中：

```
用户问题 ─► 检索外部知识 ─► 拼装上下文 ─► 大模型生成 ─► 带依据的回答
```

### 1.1.1 离线与在线两阶段

RAG 的工程实现分成两段：离线做索引，在线做问答。

```
离线索引：采集 ─► 解析 ─► 分块 ─► 向量化 ─► 入库
在线服务：提问 ─► 改写 ─► 检索 ─► 重排 ─► 生成
```

- 离线阶段决定"能被检索到什么"，是召回上限。
- 在线阶段决定"能不能用好"，是精度与忠实度。

### 1.1.2 与纯生成的区别

| 维度 | 纯生成 | RAG |
|------|--------|-----|
| 知识来源 | 模型参数 | 外部知识库 |
| 更新成本 | 需重训 | 更新索引 |
| 可溯源性 | 弱 | 强 |
| 幻觉风险 | 高 | 较低（受约束） |

## 1.2 解决什么问题

| 问题 | RAG 的应对 |
|------|-----------|
| 模型不知道企业内部知识 | 外挂私有知识库 |
| 知识更新快、微调成本高 | 更新索引即可，无需重训 |
| 回答不可溯源 | 返回引用来源 |
| 幻觉严重 | 用检索内容约束生成 |

### 1.2.1 典型场景

- 企业知识库问答：制度、手册、工单。
- 智能客服：产品文档 + 会话上下文。
- 文档助手：长合同、研究报告的问答与摘要。
- 代码库问答：仓库文档与源码检索。

## 1.3 与微调的边界

- 需要**注入事实知识、频繁更新** → 优先 RAG。
- 需要**改变说话风格、固定输出格式** → 优先微调。
- 两者不互斥：常见组合是"微调调风格 + RAG 供事实"。

### 1.3.1 选择依据

| 需求 | 首选 |
|------|------|
| 注入可更新的私有事实 | RAG |
| 固定输出风格与格式 | 微调 |
| 降低幻觉、要求可溯源 | RAG |
| 压缩延迟、省上下文 | 微调 |
| 同时满足 | 微调 + RAG |

# 2. 学习目标

## 2.1 知识目标

- 理解 RAG 全链路与各环节的失败点。
- 掌握向量表示、相似度度量与索引算法原理。
- 理解混合检索、重排、查询改写的收益与代价。
- 理解评测指标的含义与适用边界。

## 2.2 能力目标

- 用 Java 完成文档解析、切分、向量化、入库与检索。
- 掌握混合检索、重排、查询改写等提效手段。
- 能搭建并评测一个企业级知识库问答系统。
- 能定位线上失败并做归因与回流。

## 2.3 前置要求

| 前置 | 说明 |
|------|------|
| Java 基础 | JDK 17/21，熟悉集合与流式 API |
| Spring 生态 | Spring Boot、依赖注入 |
| LLM 基础 | 见 `2_LLM` 模块 |
| LLM 工程 | 见 `4_LLMEngineering` 模块 |

# 3. 文件清单与大纲

本模块共 7 个子目录，按"基础 → 数据处理 → 表示 → 存储 → 检索 → 进阶 → 评测"组织。

## 3.1 Basics/

| 文件 | 大纲 |
|------|------|
| `1_rag_overview.md` | RAG 是什么、解决什么问题、与微调的边界 |
| `2_rag_pipeline.md` | 离线索引 + 在线检索生成两阶段流水线 |
| `3_use_cases.md` | 知识库问答、客服、文档助手等典型场景 |

## 3.2 DataProcessing/

| 文件 | 大纲 |
|------|------|
| `1_document_loading.md` | 数据源接入：文件、数据库、网页、API |
| `2_document_parsing.md` | PDF/Word/HTML/Markdown 解析、表格与版式处理 |
| `3_chunking_strategies.md` | 固定长度、递归、语义、父子分块与重叠策略 |
| `4_metadata_and_filtering.md` | 元数据设计、权限过滤、结构化过滤 |

## 3.3 Embedding/

| 文件 | 大纲 |
|------|------|
| `1_embedding_basics.md` | 向量表示、语义相似度、维度与归一化 |
| `2_embedding_models.md` | BGE / E5 / GTE / OpenAI Embedding 等模型选型 |
| `3_similarity_metrics.md` | 余弦、点积、欧氏距离的适用与陷阱 |
| `4_embedding_finetuning.md` | 领域适配、对比学习微调、负样本构造 |

## 3.4 VectorStore/

| 文件 | 大纲 |
|------|------|
| `1_vector_db_overview.md` | 向量库能力对比、选型维度、Java 客户端概览 |
| `2_milvus.md` | Milvus 架构、集合/分区、Java SDK 实战 |
| `3_qdrant.md` | Qdrant 特性、过滤检索、Java SDK 实战 |
| `4_redis_vector.md` | Redis Stack 向量检索、与缓存共用 |
| `5_elasticsearch.md` | ES 向量字段、混合检索、Java 客户端 |
| `6_pgvector.md` | PostgreSQL + pgvector、事务一致性优势 |
| `7_index_algorithms.md` | HNSW / IVF / PQ 原理、参数调优与召回权衡 |

## 3.5 Retrieval/

| 文件 | 大纲 |
|------|------|
| `1_dense_retrieval.md` | 稠密向量检索、Top-K、相似度阈值 |
| `2_sparse_retrieval_bm25.md` | 稀疏检索、BM25 原理、倒排索引 |
| `3_hybrid_search.md` | 混合检索、RRF 融合、权重调优 |
| `4_reranking.md` | 交叉编码重排、Rerank 模型、召回-精排两阶段 |
| `5_query_transformation.md` | HyDE、多查询扩展、查询分解与改写 |
| `6_contextual_retrieval.md` | 上下文增强检索、Anthropic Contextual Retrieval 思路 |

## 3.6 Advanced/

| 文件 | 大纲 |
|------|------|
| `1_parent_document_retriever.md` | 父子文档检索、小块检索大块喂给模型 |
| `2_graph_rag.md` | 知识图谱 + RAG、实体关系抽取与图检索 |
| `3_agentic_rag.md` | 让 Agent 自主决定检索策略与多轮检索 |
| `4_multimodal_rag.md` | 图文混合检索、多模态 Embedding |
| `5_rag_optimization.md` | 端到端优化清单、常见瓶颈与调优顺序 |

## 3.7 Evaluation/

| 文件 | 大纲 |
|------|------|
| `1_rag_evaluation.md` | 忠实度、答案相关性、上下文召回/精确度（RAGAS 思想） |
| `2_failure_analysis.md` | 定位失败在检索还是生成、错误归因方法 |

# 4. 学习顺序与依赖

## 4.1 推荐路径

```
Basics → DataProcessing → Embedding → VectorStore → Retrieval → Advanced → Evaluation
```

- 先建立全局观（Basics），再打通数据侧（DataProcessing）。
- 理解向量与索引（Embedding、VectorStore）后进入检索（Retrieval）。
- 最后学习进阶方案与评测，形成闭环。

### 4.1.1 依赖前置

| 目录 | 前置 |
|------|------|
| DataProcessing | 无 |
| Embedding | DataProcessing |
| VectorStore | Embedding |
| Retrieval | VectorStore |
| Advanced | Retrieval |
| Evaluation | Retrieval |

### 4.1.2 学习节奏

| 节奏 | 建议 |
|------|------|
| 快速入门 | Basics + DataProcessing + Retrieval 前半 |
| 系统学习 | 按 4.1 顺序全量 |
| 专项突破 | 直接进入 VectorStore 或 Retrieval |

## 4.2 上下游依赖

- 上游依赖：`2_LLM`、`4_LLMEngineering`。
- 下游影响：`6_AgentCore`、`8_Memory`（向量记忆）、`16_Applications/KnowledgeBase`。

```
2_LLM ─┐
       ├─► 5_RAG ─┬─► 6_AgentCore
4_LLMEng ┘         ├─► 8_Memory
                   └─► 16_Applications/KnowledgeBase
```

## 4.3 阶段目标

| 阶段 | 覆盖目录 | 阶段目标 |
|------|----------|----------|
| 入门 | Basics、DataProcessing | 跑通最小 RAG |
| 核心 | Embedding、VectorStore、Retrieval | 召回与精度达标 |
| 进阶 | Advanced | 应对复杂场景 |
| 闭环 | Evaluation | 可持续迭代 |

# 5. 关键知识点速览

## 5.1 核心结论

- RAG 效果 = 检索质量 × 生成质量，检索是瓶颈。
- 分块策略决定召回上限，重排决定精度上限。
- 混合检索 + 重排是当前性价比最高的组合。
- 没有评测的 RAG 优化都是盲调。

### 5.1.1 一句话记忆

> 先保召回，再提精度，最后压幻觉；每一步都要有数字支撑。

## 5.2 优化优先级

| 顺序 | 动作 | 收益 |
|:----:|------|------|
| 1 | 修数据与分块 | 提升召回上限 |
| 2 | 混合检索 | 补专有名词短板 |
| 3 | 加重排 | 提升精度 |
| 4 | 查询改写 | 弥合语义鸿沟 |
| 5 | 提示与上下文治理 | 提升忠实度 |

## 5.3 常见误区

- 一上来就换大模型，却不修检索。
- 只调 topK，不看分块质量。
- 只看个例感受，不做评测回归。
- 忽略无答案样本，拒答能力无人测。

## 5.4 术语表

| 术语 | 含义 |
|------|------|
| RAG | 检索增强生成 |
| Embedding | 文本的稠密向量表示 |
| Chunk | 分块后的最小检索单元 |
| Top-K | 检索返回的候选数量 |
| Rerank | 对候选做精细重排 |
| RRF | 倒数排名融合，混合检索常用 |
| HyDE | 先假设答案再检索的改写方法 |
| RAGAS | 一套 RAG 评测指标思想 |
| Faithfulness | 忠实度，回答是否有依据 |
