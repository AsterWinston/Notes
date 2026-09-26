# 1. 优化原则

本章结论先行：RAG（Retrieval-Augmented Generation，检索增强生成）优化有一条铁律——**没有评测，就没有优化**。先定位瓶颈是检索还是生成，再按"收益/成本"排序，从最便宜的手段做起。

## 1.1 评测先行

结论：没有评测集，任何调参都是"盲调"，很可能改好 A 却改坏 B。

### 1.1.1 为什么必须先建评测

优化本质是"在多个方案间做选择"，而选择需要可比较的度量。缺少评测会带来三个问题：

- 无法判断某次改动是**真提升**还是随机波动；
- 无法发现回归（regression），旧问题被新改动破坏；
- 团队对"变好了"缺乏共识，决策靠感觉。

### 1.1.2 最小评测集构成

一个可用的最小评测集应包含：

| 要素 | 说明 | 示例 |
|------|------|------|
| 问题 | 真实用户问法 | "差旅住宿费上限多少" |
| 标准答案 | 参考答案 | "普通城市 500 元，上限 800 元" |
| 应命中文档 | 正确来源 | `travel_policy.pdf` 第 3 页 |
| 类型标签 | 便于分类分析 | 事实型 / 多跳 / 无答案 |

问题数量建议从 50 条起步，覆盖"简单事实、多跳、无答案、口语化"四类。无答案问题专门用于检验拒答与幻觉。

## 1.2 收益/成本排序

### 1.2.1 先定位瓶颈

```
答案不在上下文 → 检索问题（召回失败）
答案在上下文但答错 → 生成问题（提示/模型）
```

只有先归因，才能避免"检索有病、生成吃药"。

### 1.2.2 排序原则

| 原则 | 说明 |
|------|------|
| 先便宜后昂贵 | 分块调整几乎零成本 |
| 先高杠杆后微调 | 混合检索收益远大于换模型 |
| 先可解释后黑盒 | 提示与分块比微调更可控 |
| 每次只改一处 | 便于归因 |

# 2. 端到端优化清单

结论：优化项遍布解析、分块、检索、生成各阶段，应按收益/成本比排序执行。

## 2.1 全链路清单

### 2.1.1 清单表

| 阶段 | 优化项 | 成本 | 收益 |
|------|--------|:----:|:----:|
| 解析 | 换更好的解析器 | 低 | 中高 |
| 分块 | 调块大小/重叠/结构 | 低 | 高 |
| 分块 | 标题拼接、父子分块 | 低 | 高 |
| 元数据 | 补全、加过滤 | 低 | 中 |
| Embedding | 换更强模型 | 中 | 中高 |
| 检索 | 提高 topK | 低 | 中 |
| 检索 | 加 BM25 混合 | 中 | 高 |
| 检索 | 加 Rerank | 中 | 高 |
| 查询 | 查询改写/HyDE | 中 | 中高 |
| 上下文 | 上下文检索增强 | 高 | 高 |
| 生成 | 优化提示、要求引用 | 低 | 中高 |
| 生成 | 换更强模型 | 高 | 中 |

### 2.1.2 优先级解读

按"成本低 + 收益高"优先，最该先做的是：**分块优化、混合检索、重排、提示约束**。这四项加起来往往能解决大部分问题，且都不需要训练。

## 2.2 杠杆点

### 2.2.1 分块与检索是最大杠杆

分块决定了"检索单元"的语义质量，检索决定了"生成依据"的质量。二者位于流水线前端，错误会逐级放大：

```
分块差 → 召回差 → 上下文差 → 答案差
```

因此前端优化通常比后端（换模型、微调）性价比更高。

# 3. 第一步：定位瓶颈

结论：任何优化都应从诊断开始，用日志回答"检索到了什么、答案在不在里面、模型答了什么"。

## 3.1 诊断方法

### 3.1.1 三步诊断

1. 打印检索结果，看是否包含正确文档；
2. 判断标准答案是否出现在上下文中；
3. 观察模型最终回答，与上下文比对。

### 3.1.2 代码示例

```java
public void diagnose(String question, String expected) {
    // 1. 检索到了什么
    List<Document> docs = retriever.retrieve(question);
    log.info("检索结果：{}", docs.stream().map(Document::getText).toList());

    // 2. 答案是否在检索结果里
    boolean answerInContext = docs.stream()
            .anyMatch(d -> containsAnswer(d.getText(), expected));
    log.info("答案是否在上下文中：{}", answerInContext);

    // 3. 模型回答
    String answer = generate(question, docs);
    log.info("生成答案：{}", answer);
}
```

## 3.2 归因判断

### 3.2.1 判断表

| 现象 | 归因 | 优化方向 |
|------|------|----------|
| 答案不在上下文 | 召回失败 | 检索优化 |
| 答案在上下文但答错 | 生成问题 | 提示/模型 |
| 答案在上下文但答不全 | 上下文不足 | 提高 topK/迭代 |
| 答案对但引用错 | 组织问题 | 编号与校验 |

### 3.2.2 常见误判

| 误判 | 真相 |
|------|------|
| "模型太笨" | 往往检索没给对资料 |
| "要换大模型" | 可能只需改分块 |
| "topK 不够大" | 可能是噪声淹没 |

# 4. 检索优化

结论：检索问题分两类——**召回不足**（漏）与**噪声过多**（误），对策方向相反，必须先分清。

## 4.1 召回不足

### 4.1.1 原因与对策表

| 原因 | 对策 |
|------|------|
| 块太大/太小 | 调整分块 |
| 语义鸿沟 | 查询改写、HyDE |
| 专有名词 | 加 BM25 混合 |
| topK 太小 | 提高到 20~50 |
| 阈值太高 | 降低阈值 |
| Embedding 弱 | 换模型 |

### 4.1.2 代码：混合检索

稠密（dense）检索擅长语义，稀疏（sparse）检索擅长关键词，两者互补：

```java
public List<Document> hybridSearch(String query, int topK) {
    List<Document> dense = vectorStore.similaritySearch(
            SearchRequest.builder().query(query).topK(topK).build());
    List<Document> sparse = bm25Retriever.search(query, topK);
    return reciprocalRankFusion(dense, sparse, topK);
}
```

融合常用 RRF（Reciprocal Rank Fusion，倒数排名融合），无需归一化分数：

```
score(d) = Σ 1 / (k + rank_i(d))     k 常取 60
```

## 4.2 噪声过多

### 4.2.1 原因与对策表

| 原因 | 对策 |
|------|------|
| 阈值太低 | 提高阈值 |
| 缺重排 | 加 Rerank |
| 块太大 | 缩小块 |
| 元数据未过滤 | 加过滤条件 |

### 4.2.2 代码：Rerank

先大量召回，再用交叉编码器（Cross-Encoder）精排，是"高召回 + 高精度"的标准组合：

```java
public List<Document> retrieveAndRerank(String query) {
    List<Document> candidates = hybridSearch(query, 50);   // 粗排大召回
    return rerankModel.rerank(query, candidates, 5);       // 精排小输出
}
```

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 盲目加 topK | 噪声增加反而变差 | 配合重排 |
| 混合权重固定 | 不同查询需求不同 | 用评测调权重 |
| 重排太慢 | 交叉编码器昂贵 | 控制候选数 |
| 忽略元数据过滤 | 跨租户串数据 | 检索前强制过滤 |

# 5. 生成优化

结论：生成问题的根源常是"提示约束不足"，而非模型能力不足；先用提示，再考虑换模型。

## 5.1 常见生成问题

### 5.1.1 问题表

| 问题 | 对策 |
|------|------|
| 幻觉 | 提示要求"只依据资料"，无资料则拒答 |
| 不引用来源 | 要求标注来源，校验引用 |
| 答非所问 | 明确提示任务与输出格式 |
| 啰嗦 | 限制输出长度与格式 |
| 遗漏信息 | 提供更完整的上下文 |

### 5.1.2 强约束提示

```java
@SystemMessage("""
        你是企业知识库助手。严格遵守：
        1. 只依据【参考资料】回答，不得使用外部知识；
        2. 每条结论必须标注来源编号，如 [1]；
        3. 资料中没有的信息，回答"资料中未提及"；
        4. 不得编造引用或来源。

        【参考资料】
        {{context}}
        """)
```

## 5.2 引用与校验

### 5.2.1 机制拆解

要求模型输出结构化引用（如 JSON），再用程序校验引用编号是否越界，可显著减少"编造来源"。

### 5.2.2 代码示例

```java
record Answer(String text, List<Integer> citations) {}

public Answer askWithCitation(String question, List<Document> docs) {
    Answer answer = citationModel.answer(question, buildContext(docs));
    boolean valid = answer.citations().stream()
            .allMatch(i -> i >= 1 && i <= docs.size());
    return valid ? answer
                 : new Answer(answer.text(), List.of());   // 丢弃非法引用
}
```

## 5.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 提示过长 | 挤压资料空间 | 精简规则 |
| 强制 JSON | 影响自然度 | 仅结构化场景用 |
| 拒答过度 | 有资料也拒答 | 提示"有依据才答" |
| 引用编号漂移 | 上下文重排后错位 | 生成前固定编号 |

# 6. 上下文组织优化

结论：同样的检索结果，不同的组织方式会显著影响生成质量；组织是"零成本"的优化。

## 6.1 组织策略

### 6.1.1 策略表

| 优化 | 说明 |
|------|------|
| 去重 | 相似片段合并 |
| 排序 | 最相关放前面（防位置偏差） |
| 裁剪 | 按 Token 预算截断 |
| 编号 | 片段编号便于引用 |
| 精简 | 去掉无关元数据 |

### 6.1.2 代码：按 Token 预算裁剪

```java
public String buildContext(List<Document> docs, int maxTokens) {
    StringBuilder sb = new StringBuilder();
    int tokens = 0;

    for (int i = 0; i < docs.size(); i++) {
        Document doc = docs.get(i);
        String block = "[%d] %s\n".formatted(i + 1, doc.getText());

        int blockTokens = estimateTokens(block);
        if (tokens + blockTokens > maxTokens) break;

        sb.append(block);
        tokens += blockTokens;
    }
    return sb.toString();
}
```

## 6.2 位置偏差

### 6.2.1 机制与对策

大模型对上下文**开头与结尾**更敏感（"lost in the middle"）。对策是：把最相关的片段放在最前或最后，避免关键信息落在中段。

```
[片段1 最相关] ... [片段N 次相关] ... [片段2 次相关]
     ↑ 开头强                  ↑ 结尾强   ↑ 中段弱
```

## 6.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 截断丢关键 | 按顺序截断可能丢尾部关键片段 | 先重排再截断 |
| 去重误删 | 相似但语义不同被合并 | 用高阈值 |
| 编号混乱 | 拼接后编号不连续 | 生成前重编号 |

# 7. 性能优化

结论：性能优化的目标是降低 P95 延迟；并行、缓存、流式是三板斧。

## 7.1 延迟优化

### 7.1.1 手段表

| 手段 | 说明 |
|------|------|
| 并行检索 | 稠密与稀疏并行 |
| 批量 | 向量化批量、重排批量 |
| 索引调优 | HNSW 参数 |
| 流式 | 生成阶段流式返回 |
| 预计算 | 热点查询预生成 |

### 7.1.2 代码：并行检索

```java
public List<Document> parallelRetrieve(String query) throws Exception {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        Future<List<Document>> dense =
                executor.submit(() -> vectorStore.search(query, 20));
        Future<List<Document>> sparse =
                executor.submit(() -> bm25Retriever.search(query, 20));

        return merge(dense.get(), sparse.get());   // JDK 21 虚拟线程
    }
}
```

在 JDK 17 上可用固定线程池替代虚拟线程；核心是让两条检索链路并发而非串行。

## 7.2 缓存策略

### 7.2.1 缓存层次

| 层次 | 缓存内容 | 失效策略 |
|------|----------|----------|
| L1 | 查询向量 | 按查询文本哈希 |
| L2 | 检索结果 | 按查询 + 过滤条件 |
| L3 | 生成结果 | 按问题 + 上下文哈希 |
| L4 | 文档向量 | 按文档内容哈希 |

### 7.2.2 代码示例

```java
public String cachedAnswer(String question) {
    String key = "qa:" + sha256(question);
    String hit = cache.get(key);
    if (hit != null) return hit;

    String answer = ragAnswer(question);
    cache.put(key, answer, Duration.ofHours(6));
    return answer;
}
```

## 7.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 缓存污染 | 权限不同共用缓存 | 缓存键含用户/租户 |
| 缓存过期 | 知识更新后仍命中旧答案 | 短 TTL + 版本号 |
| 并行失控 | 线程池打满 | 限流与超时 |

# 8. 成本优化

结论：成本主要来自向量化、检索与生成三处；模型分层与缓存是最有效的手段。

## 8.1 成本构成与手段

### 8.1.1 成本表

| 环节 | 成本来源 |
|------|----------|
| 入库 | 解析、Embedding |
| 检索 | 向量库算力、Rerank 调用 |
| 生成 | 输入/输出 Token |
| 存储 | 向量与原文存储 |

### 8.1.2 手段表

| 手段 | 说明 |
|------|------|
| 内容去重 | 减少向量化量 |
| 向量缓存 | 相同文本不重复向量化 |
| 模型分层 | 简单任务小模型 |
| 减少候选 | 控制 topK 与重排数量 |
| 结果缓存 | 热点问题缓存 |
| 本地模型 | Embedding/Rerank 本地化 |

## 8.2 模型分层

### 8.2.1 机制与代码

把任务按难度路由到不同规模的模型：分类、改写用便宜模型，最终生成用强模型。

```java
public String generate(String question, String context) {
    boolean hard = question.length() > 200 || context.length() > 4000;
    ChatModel model = hard ? strongModel : cheapModel;
    return model.generate(question, context);
}
```

# 9. 优化顺序与陷阱

结论：优化要按"先评测、再分块、后检索、再生成、最后微调"的顺序推进，并警惕常见陷阱。

## 9.1 推荐顺序

### 9.1.1 顺序清单

```
1. 建评测集（必须第一步）
2. 调分块（收益最大、成本最低）
3. 加混合检索（稠密 + BM25）
4. 加 Rerank
5. 查询改写 / HyDE
6. 上下文增强（标题/父子/Contextual）
7. 换 Embedding / 生成模型
8. 微调（最后手段）
```

### 9.1.2 理由

越靠前的手段成本越低、收益越高、越可解释。微调成本最高且需要标注数据，通常在前七步都做尽后仍不达标才考虑。

## 9.2 常见陷阱

### 9.2.1 陷阱表

| 陷阱 | 说明 |
|------|------|
| 没评测就调参 | 可能改好 A 改坏 B |
| 只调模型 | 忽略分块与检索 |
| 盲目加 topK | 噪声增加，反而变差 |
| 无脑上 GraphRAG | 成本高且未必需要 |
| 忽略延迟 | 效果提升但体验下降 |
| 过拟合测试集 | 需保留未参与调优的测试集 |

### 9.2.2 反例

某团队把 topK 从 5 提到 50，离线正确率略升，但线上 P95 延迟翻倍且幻觉增多——原因是噪声片段挤占了有效上下文。正确做法是"大召回 + 强重排"，而非单纯加 topK。

# 10. 小结

## 10.1 核心结论

- 优化前必须建评测集，先定位瓶颈。
- 收益/成本最高的是**分块优化**与**混合检索 + 重排**。
- 上下文组织与提示约束能显著减少幻觉。
- 性能与成本优化靠并行、缓存、模型分层。
- 微调是最后手段。

## 10.2 落地清单

- 用 50 条以上、覆盖四类问题的评测集起步。
- 每次只改一处，改完即评，防止回归。
- 检索先用"大召回 + 强重排"，不要盲目加 topK。
- 上下文先重排再裁剪，注意"lost in the middle"。
- 缓存键必须包含用户与租户，避免越权命中。
