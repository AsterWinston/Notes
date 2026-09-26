# 1. SK 的记忆抽象

记忆是 LLM 应用保持上下文与知识的基础能力。Semantic Kernel 把记忆抽象为两层：面向当前会话的短期记忆（对话历史）与面向跨会话知识的长期记忆（向量存储）。理解这两层的边界，是设计可靠对话系统与 RAG 系统的前提。

## 1.1 记忆的两层结构

### 1.1.1 短期与长期

```
记忆 = 存储 + 检索
  ├── 短期：对话历史（ChatHistory）
  └── 长期：向量记忆（VectorStore）
```

短期记忆随会话结束而失效，解决"记住刚才说了什么"；长期记忆跨会话持久化，解决"记住业务知识"。二者组合才能支撑完整的多轮问答。

| 类型 | 生命周期 | 检索方式 | 典型实现 |
|------|----------|----------|----------|
| 短期 | 会话内 | 顺序读取 | ChatHistory |
| 长期 | 持久 | 相似度检索 | VectorStore |

### 1.1.2 抽象类型

| 类型 | 说明 |
|------|------|
| `ChatHistory` | 对话消息历史 |
| `VectorStore` | 向量存储抽象 |
| `MemoryStore` | 记忆存储抽象 |

`VectorStore` 屏蔽了具体向量库差异，使应用可以在内存库与生产库之间切换而不改业务代码。

## 1.2 记忆在 Kernel 中的位置

### 1.2.1 与 Kernel 的关系

记忆既可独立使用，也可注册进 Kernel 供函数与模型访问。注册后，记忆成为 Kernel 的一项能力，可被过滤器审计、被编排层统一管理。

```
Kernel
  ├── AI 服务
  ├── 插件
  ├── 记忆 ──┬── ChatHistory（短期）
  │          └── VectorStore（长期）
  └── 过滤器
```

### 1.2.2 使用方式

短期记忆通常随请求创建并传入调用；长期记忆作为单例组件注入，供检索使用。

```java
ChatHistory history = new ChatHistory();
history.addUserMessage("你好");
```

## 1.3 生产建议

不要把短期记忆与长期记忆混为一谈：短期记忆放 Redis 并设 TTL，长期记忆放向量库并做容量治理，两者的扩容与备份策略完全不同。

## 1.4 记忆的读写路径

### 1.4.1 读路径

读路径在每次生成前发生：加载会话历史、检索长期知识、组装成 Prompt。

```
请求 ─▶ 加载 ChatHistory ─▶ 检索 VectorStore ─▶ 组装 Prompt ─▶ 模型
```

### 1.4.2 写路径

写路径在生成后发生：追加用户与助手消息、必要时写入长期记忆。

```java
// 伪代码：读改写闭环
ChatHistory history = sessionStore.load(sessionId);
history.addUserMessage(question);
String answer = chatService.getChatMessageContentAsync(history, settings, kernel)
        .block().getContent();
history.addAssistantMessage(answer);
sessionStore.save(sessionId, history);
```

| 阶段 | 目标 | 失败处理 |
|------|------|----------|
| 读 | 恢复上下文 | 降级为空历史 |
| 检索 | 补充事实 | 空召回兜底 |
| 写 | 持久化 | 重试或落盘 |

# 2. 对话历史

对话历史是最直接的记忆形式，由有序的消息列表构成。

## 2.1 ChatHistory 结构

### 2.1.1 消息角色

对话历史中的每条消息都有角色，角色决定模型如何理解它。

| 角色 | 含义 | 典型内容 |
|------|------|----------|
| system | 系统指令 | 人设、约束 |
| user | 用户输入 | 问题 |
| assistant | 模型回复 | 答案 |
| tool | 工具结果 | 函数返回值 |

### 2.1.2 使用示例

```java
ChatHistory history = new ChatHistory();
history.addSystemMessage("你是专业助手。");
history.addUserMessage("你好");
history.addAssistantMessage("你好！有什么可以帮你？");
history.addUserMessage("介绍一下 SK");

ChatMessageContent response = chatService.getChatMessageContentAsync(
        history, settings, kernel).block();
```

## 2.2 历史管理

### 2.2.1 窗口与截断

历史不能无限增长，否则 token 成本与延迟都会失控。常用策略是保留最近 N 轮并对早期内容做摘要。

```java
// 伪代码：滑动窗口
int maxTurns = 10;
while (history.getMessages().size() > maxTurns * 2 + 1) {
    history.remove(1);   // 保留 system 消息
}
```

| 策略 | 优点 | 缺点 |
|------|------|------|
| 滑动窗口 | 简单、成本低 | 丢失早期信息 |
| 摘要压缩 | 保留要点 | 额外调用成本 |
| 全量 | 信息完整 | 成本高、易超限 |

### 2.2.2 持久化

会话历史应外置到 Redis 或数据库，并设置 TTL，避免进程重启丢失上下文。

```yaml
#伪代码：会话存储配置
session:
  store: redis
  ttl: 2h
```

## 2.3 生产建议

为会话设置唯一 ID（如 租户:用户:会话），并在多实例部署下确保同一会话路由到共享存储，避免记忆串号或丢失。

## 2.4 多轮对话示例

### 2.4.1 完整回合

```java
public String chat(String sessionId, String userInput) {
    ChatHistory history = sessionStore.load(sessionId);
    history.addUserMessage(userInput);

    ChatMessageContent reply = chatService.getChatMessageContentAsync(
            history, settings, kernel).block();

    history.addAssistantMessage(reply.getContent());
    sessionStore.save(sessionId, history);
    return reply.getContent();
}
```

## 2.5 消息裁剪

### 2.5.1 裁剪策略

```java
// 伪代码：保留 system 与最近 N 条
List<ChatMessageContent> messages = history.getMessages();
if (messages.size() > maxMessages) {
    List<ChatMessageContent> recent =
            messages.subList(messages.size() - maxMessages, messages.size());
    history = new ChatHistory(recent);
}
```

| 参数 | 含义 | 建议值 |
|------|------|--------|
| maxMessages | 保留消息数 | 20 |
| maxTokens | 上下文上限 | 按模型 |
| 摘要阈值 | 触发摘要的长度 | 80% 上限 |

# 3. 向量存储

向量存储承载长期记忆与 RAG 检索，是 SK 记忆体系的重量级组件。

## 3.1 记录定义

### 3.1.1 注解说明

向量记录通过注解标记主键、数据字段与向量字段。

| 注解 | 作用 |
|------|------|
| `@VectorStoreRecordKey` | 主键 |
| `@VectorStoreRecordData` | 可过滤/展示字段 |
| `@VectorStoreRecordVector` | 向量字段与维度 |

### 3.1.2 完整示例

```java
// 定义记录
public record KnowledgeRecord(
        @VectorStoreRecordKey String id,
        @VectorStoreRecordData String content,
        @VectorStoreRecordVector(dimensions = 1536) List<Float> embedding
) {}
```

维度必须与嵌入模型一致，一旦建库便难以更改，需在建库前确定。

## 3.2 集合与读写

### 3.2.1 创建集合

```java
// 存储
VectorStore vectorStore = new InMemoryVectorStore(options);
vectorStore.createCollectionAsync("knowledge", schema).block();
```

### 3.2.2 写入与检索

```java
vectorStore.upsertAsync("knowledge", List.of(record)).block();

// 检索
List<VectorSearchResult> results = vectorStore.searchAsync(
        "knowledge",
        new VectorSearchOptions().setTop(5).setVector(embedding),
        null).block();
```

## 3.3 检索参数

### 3.3.1 topK 与相似度

| 参数 | 作用 | 建议 |
|------|------|------|
| topK | 召回数量 | 3 到 5 |
| 相似度阈值 | 过滤低质结果 | 按测试集标定 |
| 过滤条件 | 元数据过滤 | 强制租户隔离 |

topK 过大既增加成本又引入噪声，过小则可能漏召回，需按业务标定。

## 3.4 嵌入模型与维度

### 3.4.1 维度选择

嵌入模型的维度决定向量字段大小与检索精度。更换嵌入模型必须重建整个集合，因为不同模型的向量空间不可比。

| 嵌入模型 | 维度 | 特点 |
|----------|------|------|
| text-embedding-3-small | 1536 | 成本低 |
| text-embedding-3-large | 3072 | 精度高 |
| 通义 text-embedding-v3 | 1024 | 中文友好 |

### 3.4.2 重建流程

```
下线写入 ─▶ 全量重嵌入 ─▶ 新建集合 ─▶ 校验召回 ─▶ 切换读取
```

## 3.5 元数据过滤

### 3.5.1 过滤示例

检索时强制附加租户与可见性条件，实现数据隔离。

```java
// 伪代码：带过滤的检索
VectorSearchOptions options = new VectorSearchOptions()
        .setTop(5)
        .setVector(queryEmb)
        .setFilter("tenantId == 't1' and visibility == 'public'");
```

| 字段 | 用途 | 是否必填 |
|------|------|----------|
| tenantId | 租户隔离 | 是 |
| visibility | 可见范围 | 视业务 |
| source | 来源溯源 | 建议 |

## 3.6 生产建议

过滤字段需在建库时声明为可过滤，否则运行时可能报错或退化为后过滤，导致召回数量不足。写入与检索都应设置超时；批量写入时控制批次大小，避免一次提交过多记录导致超时或内存压力。

# 4. 支持的向量库

SK 通过 `VectorStore` 抽象对接多种向量库，应用可平滑切换。

## 4.1 向量库总览

### 4.1.1 对比表

| 存储 | 说明 |
|------|------|
| InMemory | 开发测试 |
| Azure AI Search | Azure 生态 |
| Qdrant | 开源 |
| Redis | 高并发 |
| PostgreSQL | pgvector |

## 4.2 选型建议

### 4.2.1 决策路径

```
数据规模与并发？
  ├── 开发/测试 ─▶ InMemory
  ├── 中小规模 ─▶ PostgreSQL(pgvector)
  ├── 高并发低延迟 ─▶ Redis / Qdrant
  └── Azure 生态 ─▶ Azure AI Search
```

## 4.3 生产建议

生产环境务必使用具备持久化与备份能力的向量库，InMemory 仅用于本地开发；切换向量库时应通过抽象层而非直接改业务代码。

## 4.4 连接与配置

### 4.4.1 配置示例

连接信息放配置中心或环境变量，密钥不落盘。

```yaml
#伪代码：向量库连接配置
vectorstore:
  type: qdrant
  host: ${QDRANT_HOST}
  port: 6333
  api-key: ${QDRANT_API_KEY}
  collection: knowledge
  dimensions: 1024
```

# 5. 记忆与 RAG

RAG（Retrieval-Augmented Generation，检索增强生成）是长期记忆最典型的应用：先检索相关资料，再据此生成答案。

## 5.1 RAG 流程

### 5.1.1 三步流程

```
问题 ─▶ ① 嵌入 + 检索 ─▶ ② 组装上下文 ─▶ ③ 生成答案
```

### 5.1.2 完整示例

```java
// 用向量记忆实现 RAG
public String ragAnswer(String question) {
    // ① 检索
    List<Float> queryEmb = embeddingService.embed(question);
    List<VectorSearchResult> results = vectorStore.searchAsync(
            "knowledge", new VectorSearchOptions().setVector(queryEmb).setTop(5),
            null).block();

    // ② 组装上下文
    String context = results.stream()
            .map(r -> r.getRecord().get("content").toString())
            .collect(Collectors.joining("\n---\n"));

    // ③ 生成
    ChatHistory history = new ChatHistory();
    history.addUserMessage("依据资料回答：\n" + context + "\n问题：" + question);
    return chatService.getChatMessageContentAsync(history, settings, kernel)
            .block().getContent();
}
```

## 5.2 记忆增强

### 5.2.1 短期与长期结合

完整方案应同时注入短期对话与长期检索结果，使模型既知道"刚才聊了什么"，又知道"业务事实是什么"。

| 来源 | 注入位置 | 作用 |
|------|----------|------|
| 短期历史 | 消息列表 | 维持对话连贯 |
| 长期检索 | system/user 消息 | 提供事实依据 |

## 5.3 生产建议

检索结果必须携带来源标识并在回答中引用，便于用户核对；对空召回要有兜底话术，避免模型凭空编造。

## 5.4 检索质量优化

### 5.4.1 优化手段

| 手段 | 作用 | 成本 |
|------|------|------|
| 查询改写 | 提升召回 | 一次 LLM 调用 |
| 混合检索 | 兼顾关键词与语义 | 中 |
| 重排 | 提升排序质量 | 中 |
| 去重 | 降低冗余 | 低 |

### 5.4.2 混合检索示意

```java
// 伪代码：向量 + 关键词合并后重排
List<Document> dense = vectorStore.similaritySearch(denseRequest);
List<Document> sparse = keywordSearch(query);
List<Document> merged = reranker.rerank(query, concat(dense, sparse));
```

## 5.5 引用与溯源

### 5.5.1 引用格式

在 Prompt 中要求模型标注引用编号，并在回答中回填来源链接，可显著降低幻觉的误导性。

```
依据资料回答，并用 [1][2] 标注引用：
[1] ... 内容 ...
[2] ... 内容 ...
问题：...
```

## 5.6 生产建议

建立离线评测集，对召回率、准确率与引用正确率做回归测试，避免调参后质量悄然下降。

# 6. 生产注意事项

## 6.1 风险清单

### 6.1.1 风险与建议

| 风险 | 建议 |
|------|------|
| 记忆无限增长 | 窗口 + TTL |
| 越权检索 | 元数据过滤 |
| 无持久化 | 用外部存储 |
| 维度不匹配 | 建库前确定 |

## 6.2 容量与隔离

### 6.2.1 控制手段

| 维度 | 手段 |
|------|------|
| 容量 | 窗口、TTL、归档 |
| 隔离 | 租户过滤、独立集合 |
| 质量 | 去重、阈值、重排 |
| 成本 | topK 控制、缓存 |

## 6.3 生产建议

为向量库建立监控：写入延迟、检索延迟、召回数量分布、空召回率。这些指标能提前暴露嵌入模型变更或数据质量下降。

## 6.4 监控指标

### 6.4.1 关键指标

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| 检索 P99 | 延迟 | 超阈值告警 |
| 空召回率 | 无结果比例 | 上升告警 |
| 平均 topK 命中 | 召回数量 | 异常波动 |
| 写入失败率 | 数据完整性 | 大于 0 |

## 6.5 备份与重建

### 6.5.1 数据保护

向量库需定期快照；嵌入模型或切分策略变更时按重建流程操作，并保留旧集合以便回滚。

```yaml
#伪代码：快照策略
snapshot:
  schedule: "0 2 * * *"
  retention: 7d
```

## 6.6 生产建议

把记忆容量与检索质量纳入容量规划：长期记忆增长会持续推高存储与检索成本，需定期归档冷数据。

# 7. 小结

## 7.1 要点回顾

- SK 的记忆分短期（ChatHistory）与长期（VectorStore）。
- VectorStore 提供统一抽象，支持多种向量库。
- 用向量记忆实现 RAG。
- 生产注意持久化、隔离与容量控制。

## 7.2 演进建议

后续可引入重排模型提升召回质量、加入记忆去重与冲突消解，并把短期记忆自动摘要进长期记忆，形成更接近人类记忆的层次化结构。

## 7.3 术语表

| 术语 | 英文 | 含义 |
|------|------|------|
| 对话历史 | ChatHistory | 短期记忆 |
| 向量存储 | VectorStore | 长期记忆 |
| 嵌入 | Embedding | 文本转向量 |
| 召回 | Retrieval | 检索候选 |
| 重排 | Rerank | 二次排序 |
| 切片 | Chunking | 文档切分 |
| 相似度 | Similarity | 向量距离度量 |
| 空召回 | Empty Retrieval | 无命中结果 |

## 7.4 落地检查

落地前确认：会话是否持久化并设 TTL、向量库是否具备备份、检索是否强制租户过滤、维度是否与嵌入模型一致、是否具备空召回兜底与引用溯源。

