# 1. 上下文工程 vs 提示工程

**结论先行**：提示工程（Prompt Engineering）决定"怎么说"，上下文工程（Context Engineering）决定"给什么"；
当信息给错时，再精致的措辞也无济于事。

## 1.1 一句话区分

### 1.1.1 定义对照

```
提示工程：怎么说（措辞、格式、示例）
上下文工程：给什么（信息取舍、优先级、组织）
```

> **上下文工程比措辞更重要**：给错信息，再好的措辞也没用。

### 1.1.2 关注点差异

| 维度 | 提示工程 | 上下文工程 |
|------|----------|------------|
| 核心问题 | 怎么表达指令 | 放哪些信息进去 |
| 主要对象 | 措辞、格式、示例 | 检索、记忆、历史、工具定义 |
| 变化频率 | 版本级（低频） | 每轮请求（高频） |
| 失效表现 | 输出格式漂移 | 答非所问、信息缺失 |
| 优化手段 | 改写、加示例、加 CoT | 取舍、排序、压缩、预算 |

## 1.2 为什么上下文更关键

**结论先行**：模型只能基于"实际进入上下文窗口的 token"推理，
上下文里没有的信息，任何措辞技巧都无法凭空补出来。

### 1.2.1 信息错位的代价

| 现象 | 后果 | 措辞能否补救 |
|------|------|:------------:|
| 缺少关键事实 | 幻觉或拒答 | 否 |
| 放入无关噪声 | 注意力被稀释 | 否 |
| 版本过时 | 输出错误结论 | 否 |
| 措辞模糊 | 格式不稳定 | 是 |

### 1.2.2 一个反例

```java
// ❌ 措辞再精致，资料里没有答案也是徒劳
String prompt = """
        请极其严谨、逐步推理、务必准确回答：
        根据以下资料回答用户问题。

        【资料】%s

        【问题】%s
        """.formatted(context, question);
// 若 context 为空，模型只能编造。
```

**边界与坑**：很多团队把"效果差"归因于提示词写得不好，实际根因是上下文缺失或噪声过多。

**生产建议**：排查效果问题时，先看"进入模型的内容"，再改措辞。

## 1.3 两者的关系

**结论先行**：二者互补——上下文提供"原料"，提示提供"加工指令"。

- 没有好上下文：模型无米下锅，只能幻觉。
- 没有好提示：模型有料但不会用，输出跑偏。
- 正确顺序：先保证上下文正确、相关、精简，再优化指令与格式。

## 1.4 与 Agent 的关系

**结论先行**：Agent 每一轮工具调用都要重新组装上下文，上下文工程直接决定其可靠性与成本。

| 环节 | 上下文工程的作用 |
|------|------------------|
| 规划 | 提供任务目标与约束 |
| 工具调用 | 提供工具定义与调用历史 |
| 观察 | 回填工具结果并去噪 |
| 记忆 | 注入相关长期记忆 |
| 收敛 | 控制历史长度避免超限 |

**生产建议**：把上下文组装逻辑当作核心模块单独测试与监控，而非散落在业务代码里。

# 2. 上下文的组成

**结论先行**：一次请求的上下文由七部分组成，它们稳定性不同、优先级不同，必须分层管理。

## 2.1 七层结构

### 2.1.1 结构图

```
┌───────────────────────────────────┐
│  上下文窗口（有限预算）             │
├───────────────────────────────────┤
│ ① System 规则（最高优先）          │
│ ② 工具定义                         │
│ ③ 长期记忆（用户画像）              │
│ ④ 检索知识（RAG）                  │
│ ⑤ 对话历史                         │
│ ⑥ 当前问题                         │
│ ⑦ 输出预留                         │
└───────────────────────────────────┘
```

### 2.1.2 各层职责

| 层 | 内容 | 稳定性 | 优先级 |
|----|------|--------|:------:|
| ① System 规则 | 角色、边界、格式约束 | 高 | 最高 |
| ② 工具定义 | 函数签名与说明 | 高 | 高 |
| ③ 长期记忆 | 用户画像、偏好 | 中 | 中 |
| ④ 检索知识 | RAG 片段 | 中 | 高 |
| ⑤ 对话历史 | 近期多轮消息 | 低 | 中 |
| ⑥ 当前问题 | 本轮用户输入 | 最低 | 最高 |
| ⑦ 输出预留 | 模型生成空间 | — | 必须 |

## 2.2 生命周期与稳定性

**结论先行**：越稳定的内容越应靠前，越易变的内容越应靠后，以兼顾缓存与注意力。

```
稳定 ──────────────────────────────► 易变
System 规则 → 工具定义 → 知识库 → 历史 → 当前问题
```

## 2.3 Token 预算分配

### 2.3.1 预算示例

**结论先行**：必须为输出预留 token，否则模型会因空间不足而截断。

| 部分 | 建议占比 | 说明 |
|------|:--------:|------|
| System + 工具 | 10% | 固定开销 |
| 检索知识 | 40% | 按相关性截断 |
| 历史 | 25% | 只保留近期 |
| 当前问题 | 5% | 一般较短 |
| 输出预留 | 20% | 防止截断 |

```java
public int contextBudget(int modelLimit, int maxOutputTokens) {
    return modelLimit - maxOutputTokens - SAFETY_MARGIN;
}
```

**边界与坑**：不同模型 tokenizer（分词器）不同，估算偏差可达 10%~20%，预算需留安全边际。

**生产建议**：用实际 tokenizer 估算而非字符数，并持续监控超限率。

# 3. 信息取舍原则

**结论先行**：取舍的目标是"在有限预算内最大化与当前问题的信息增益"。

## 3.1 五大原则

| 原则 | 说明 |
|------|------|
| 相关优先 | 只放与当前问题相关的 |
| 精简 | 去掉冗余与噪声 |
| 分层 | 稳定内容在前，易变内容在后 |
| 预算 | 按 Token 预算分配 |
| 去重 | 相似内容合并 |

## 3.2 相关性与去重

**结论先行**：检索回来的片段常有大量重叠，去重能显著提升信息密度。

### 3.2.1 相似度去重

```java
public List<Document> dedupe(List<Document> docs, double threshold) {
    List<Document> kept = new ArrayList<>();
    for (Document doc : docs) {
        boolean dup = kept.stream()
                .anyMatch(k -> cosine(k.embedding(), doc.embedding()) > threshold);
        if (!dup) kept.add(doc);
    }
    return kept;
}
```

### 3.2.2 去重与排序配合

```
检索 TopN → 去重 → 按相关性排序 → 按预算截断
```

## 3.3 精简与分层

**结论先行**：精简针对"内容本身"，分层针对"内容顺序"。

- 精简：删客套话、删重复句、删与问题无关的段落。
- 分层：把稳定前缀集中在前，把每轮变化的部分集中在后。

## 3.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 过度精简 | 删掉了关键约束 | 约束单独建白名单 |
| 去重误删 | 语义不同但向量接近 | 阈值调高 + 人工抽查 |
| 只按相似度 | 忽略时效性 | 相似度 × 时效加权 |

**生产建议**：把"相关、精简、分层、预算、去重"固化为组装管线的固定阶段。

# 4. 优先级排序

**结论先行**：预算不足时按优先级从低到高裁剪，保证"必须保留"的部分永不丢失。

## 4.1 优先级分层

```
必须保留：
① System 规则
② 当前问题
③ 高相关检索片段

可裁剪：
④ 近期对话（保留最近）
⑤ 长期记忆（按需）
⑥ 低相关检索片段
```

## 4.2 组装算法

### 4.2.1 Java 实现

```java
public List<Message> assemble(ContextParts parts, int budget) {
    List<Message> result = new ArrayList<>();
    int used = 0;

    // ① 必留
    result.add(SystemMessage.from(parts.system()));
    used += estimate(parts.system());

    // ② 高相关检索
    for (Document doc : parts.retrievalDocs()) {
        int t = estimate(doc.getText());
        if (used + t > budget * 0.6) break;
        result.add(SystemMessage.from(doc.getText()));
        used += t;
    }

    // ③ 近期对话
    List<Message> recent = parts.history().subList(
            Math.max(0, parts.history().size() - 10), parts.history().size());
    result.addAll(recent);
    used += recent.stream().mapToInt(m -> estimate(m.text())).sum();

    // ④ 当前问题
    result.add(UserMessage.from(parts.question()));

    return result;
}
```

### 4.2.2 算法要点

| 要点 | 说明 |
|------|------|
| 先放必留项 | System 与当前问题优先 |
| 检索限量 | 用预算比例而非固定条数 |
| 历史留最近 | 时间近的上下文更相关 |
| 问题最后 | 贴近生成位置，注意力更强 |

## 4.3 边界与坑

- **预算硬编码**：不同模型窗口不同，应参数化。
- **裁剪无日志**：被裁掉什么必须可观测，否则难排查。
- **顺序错乱**：System 若排在历史之后，指令遵从度会下降。

**生产建议**：裁剪策略要打点记录"裁掉了哪类内容、多少 token"。

# 5. 检索上下文的组织

**结论先行**：检索片段必须以可引用、可溯源、边界清晰的方式注入，否则模型会混用或误引。

## 5.1 组织要素

| 要素 | 作用 |
|------|------|
| 编号 | 便于引用 |
| 来源 | 可溯源 |
| 分隔 | 明确边界 |

## 5.2 Java 实现

```java
public String buildRetrievalContext(List<Document> docs) {
    StringBuilder sb = new StringBuilder();
    sb.append("【参考资料】\n");

    for (int i = 0; i < docs.size(); i++) {
        Document doc = docs.get(i);
        sb.append("[%d] %s\n".formatted(i + 1, doc.getText()));
        sb.append("（来源：%s）\n\n".formatted(doc.getMetadata().get("source")));
    }
    return sb.toString();
}
```

## 5.3 排序与截断

**结论先行**：模型对"开头和结尾"更敏感，最相关的片段应放在两端。

```
最相关 ─┐
        ├─ 放首尾（注意力强）
次相关 ─┘
```

| 策略 | 说明 |
|------|------|
| 相关性降序 | 简单直接 |
| 首尾优先 | 强相关放两端 |
| 预算截断 | 超出即停止追加 |

## 5.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 无编号 | 模型无法引用 | 统一编号 |
| 无来源 | 无法审计 | 带 metadata |
| 片段粘连 | 边界不清 | 显式分隔符 |

**生产建议**：要求模型在答案中标注引用编号，便于自动核验与溯源。

# 6. 历史压缩

**结论先行**：历史不能无限增长，需在"保留细节"与"节省预算"间取得平衡。

## 6.1 压缩策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 滑动窗口 | 只保留最近 N 轮 | 短会话 |
| 摘要压缩 | 早期对话变摘要 | 长会话 |
| 混合 | 摘要 + 近期原文 | 通用 |
| 关键事实抽取 | 只留实体与结论 | 信息密集 |

## 6.2 Java 实现

```java
public List<Message> compressHistory(List<Message> history, int keepRecent) {
    if (history.size() <= keepRecent) return history;

    List<Message> old = history.subList(0, history.size() - keepRecent);
    List<Message> recent = history.subList(history.size() - keepRecent, history.size());

    // 早期内容压缩为摘要
    String summary = summarizer.summarize(old);

    List<Message> result = new ArrayList<>();
    result.add(SystemMessage.from("之前对话摘要：\n" + summary));
    result.addAll(recent);
    return result;
}
```

> 详见 `8_Memory/8_memory_compression.md`。

## 6.3 边界与坑

- **摘要丢失细节**：摘要会遗漏具体数值、编号，重要事实应单独保留。
- **摘要误差累积**：多轮摘要层层压缩会失真，需定期从原文重建。
- **摘要本身占位**：摘要过长时反而更费 token。

**生产建议**：对摘要质量做抽样评估，关键事实用结构化字段单独存储。

# 7. 稳定前置与 Prompt 缓存

**结论先行**：把稳定内容放前面、易变内容放后面，可命中厂商侧 Prompt 缓存，显著降本降延迟。

## 7.1 原理

```
把稳定内容放前面，易变内容放后面：
- 命中厂商侧 Prompt 缓存 → 降本
- 提高缓存命中率
```

**机制**：缓存以"前缀完全一致"为命中条件，任何前缀变化都会导致后续全部失效。

## 7.2 布局示例

### 7.2.1 Java 实现

```java
String prompt = """
        %s        ← System 规则（稳定）
        %s        ← 工具定义（稳定）
        %s        ← 知识库（半稳定）
        %s        ← 对话历史（较易变）
        %s        ← 当前问题（每次变）
        """.formatted(system, tools, context, history, question);
```

### 7.2.2 布局对照

```
✅ 稳定 → 易变：[System][Tools][KB][History][Question]
❌ 易变 → 稳定：[Question][History][KB][Tools][System]
```

## 7.3 提高命中率

| 做法 | 效果 |
|------|------|
| 固定前缀措辞 | 避免细微改动 |
| 时间戳后置 | 不破坏前缀 |
| 工具定义排序稳定 | 顺序变即失效 |
| 动态内容统一后置 | 保护前缀 |

## 7.4 边界与坑

- **插入时间戳**：把当前时间放开头会让每次前缀都不同，缓存全失效。
- **动态工具集**：按用户裁剪工具会改变前缀，需权衡。
- **缓存有 TTL**：厂商缓存有生存期，低频请求命中率低。

**生产建议**：监控缓存命中率与省下的 token 成本，作为上下文布局的验收指标。

# 8. 动态上下文

**结论先行**：不是每个问题都需要检索与记忆，按问题复杂度动态装配可同时降本提质。

## 8.1 动态规划思路

| 判断 | 结果 |
|------|------|
| 需要外部知识？ | 决定是否检索 |
| 涉及用户偏好？ | 决定是否注入长期记忆 |
| 简单改写类？ | 减少检索与历史 |

## 8.2 Java 实现

```java
public ContextPlan planContext(String question, UserContext user) {
    boolean needRetrieval = needsRetrieval(question);
    boolean needMemory = needsMemory(question);

    return ContextPlan.builder()
            .includeSystem(true)
            .includeTools(true)
            .includeRetrieval(needRetrieval)
            .retrievalTopK(needRetrieval ? 5 : 0)
            .includeLongTermMemory(needMemory)
            .historyWindow(10)
            .build();
}
```

## 8.3 边界与坑

- **判断本身要成本**：用小模型或规则做路由，避免大模型二次开销。
- **漏判**：把需要检索的问题判为不需要，直接导致答非所问。
- **过度动态**：分支过多难以测试，应控制组合数量。

**生产建议**：对路由判断做离线评估，统计"漏判率"并纳入回归。

# 9. 上下文质量检查

**结论先行**：组装完成后必须校验，超限、缺关键部分、空检索都应在调用前拦截。

## 9.1 校验项

| 校验 | 目的 |
|------|------|
| Token 预算 | 防止超限报错 |
| System 存在 | 防止无约束 |
| 检索非空 | 防止无据可依 |
| 角色正确 | 防止内容错位 |

## 9.2 Java 实现

```java
public void validateContext(Context context) {
    // ① Token 预算
    int total = context.totalTokens();
    if (total > MAX_CONTEXT_TOKENS) {
        throw new ContextOverflowException("上下文超限: " + total);
    }
    // ② 关键部分存在
    if (context.system() == null || context.system().isBlank()) {
        throw new IllegalStateException("缺少 System 提示");
    }
    // ③ 检索结果非空（若需要检索）
    if (context.requiresRetrieval() && context.retrievalDocs().isEmpty()) {
        log.warn("需要检索但未命中任何文档");
    }
}
```

## 9.3 可观测性

| 指标 | 说明 |
|------|------|
| 上下文 token 分布 | 各层占比 |
| 超限率 | 触发裁剪或报错比例 |
| 空检索率 | 需要检索但未命中的比例 |
| 缓存命中率 | Prompt 缓存节省情况 |

**生产建议**：把上下文快照（脱敏后）采样留存，便于复现问题。

# 10. 生产注意事项

**结论先行**：上下文工程的生产重点是"预算可控、噪声可控、稳定前置、全程可观测"。

## 10.1 风险与对策

| 风险 | 建议 |
|------|------|
| 上下文超限 | 预算 + 裁剪 |
| 噪声过多 | 只放相关的 |
| 关键信息被裁 | 优先级裁剪 |
| 稳定内容后置 | 前置提高缓存命中 |
| 检索为空仍继续 | 检查并提示 |
| 无监控 | 监控 Token 与命中 |

## 10.2 监控指标

```java
public record ContextMetric(
        int totalTokens,
        int retrievalCount,
        boolean retrievalHit,
        boolean cacheHit,
        boolean truncated,
        long latencyMs) {
}
```

## 10.3 落地清单

- 参数化窗口与输出预留，不硬编码。
- 组装管线分阶段：取舍 → 排序 → 分层 → 校验。
- 裁剪与空检索必须打点告警。
- 稳定前缀布局纳入评审，防止被无意破坏。

# 11. 小结

## 11.1 要点回顾

- 上下文工程决定"给模型看什么"，比措辞更重要。
- 上下文有七部分组成，按优先级分配预算。
- 取舍原则：相关、精简、分层、预算、去重。

## 11.2 落地清单

- 稳定内容前置以提升 Prompt 缓存命中。
- 历史压缩要防止细节丢失与误差累积。
- 上下文质量要检查（超限、缺失、空检索）。
