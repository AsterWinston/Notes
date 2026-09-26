# 1. 什么是长期记忆

长期记忆（Long-term Memory）是 **跨会话持久化的记忆**，让 Agent 在多次交互之间记住用户与经验。

## 1.1 定义与直观示例

**结论先行**：短期记忆解决"这一次对话说了什么"，长期记忆解决"这个用户是谁、我们以前经历过什么"。

### 1.1.1 跨会话记忆的本质

长期记忆的核心特征有三个：持久化（persistence）、用户级隔离（per-user isolation）、按相关性召回（relevance-based recall）。它不依赖当前上下文窗口，而是把信息写进外部存储，在需要时再取回。

```
会话1：用户说"我是后端工程师，偏好简洁回答"
会话2：Agent 直接以简洁风格回答，无需再问
```

如果只有短期记忆，第二句话就无从谈起；长期记忆让 Agent 具备"连续性人格"。

### 1.1.2 一个最小示例

```java
public String chat(UserContext user, String question) {
    String memory = memoryService.buildMemoryContext(user, question);
    String answer = chatModel.call(memory + "\n用户：" + question);
    memoryService.record(user, question, answer);
    return answer;
}
```

这里 `buildMemoryContext` 读长期记忆，`record` 在会话结束后写长期记忆，形成闭环。

## 1.2 长期记忆的分类

**结论先行**：长期记忆不是单一仓库，按内容性质至少分为用户画像、事实、经验、反馈四类，处理方式各异。

### 1.2.1 四类记忆

| 类型 | 内容 | 例子 |
|------|------|------|
| 用户画像 | 身份、偏好、习惯 | "后端工程师，偏好简洁" |
| 事实记忆 | 用户告诉过的事实 | "公司用 Java 21" |
| 经验记忆 | 历史交互的结论 | "上次这个问题用 X 方案解决" |
| 反馈记忆 | 用户的纠正 | "不要用表格回答" |

### 1.2.2 分类对比表

| 类型 | 稳定性 | 是否可推理 | 推荐存储 |
|------|--------|------------|----------|
| 用户画像 | 高 | 否 | 结构化 KV |
| 事实记忆 | 中 | 可 | 结构化 + 向量 |
| 经验记忆 | 中 | 可 | 向量 + 时间序 |
| 反馈记忆 | 高 | 否 | 结构化（优先级高） |

## 1.3 与其他记忆的关系

**结论先行**：长期记忆是记忆体系的一层，需要与短期记忆、情景/语义/程序性记忆协同，而不是孤立存在。

### 1.3.1 记忆层次

```
┌──────────────────────────────────────────┐
│  长期记忆（跨会话，用户级）              │
│   ├── 用户画像 / 事实 / 经验 / 反馈      │
│   └── 由情景归纳为语义，再提炼为程序性   │
├──────────────────────────────────────────┤
│  短期记忆（单会话，窗口）                │
└──────────────────────────────────────────┘
```

### 1.3.2 边界

- 长期记忆 **不等于** 知识库（RAG）：知识库是公共的、只读的，长期记忆是用户私有的、可写的。
- 长期记忆 **不等于** 日志：日志用于审计与回放，记忆用于影响未来回答。
- 不要把整段对话原样塞进长期记忆，那会迅速膨胀且难以召回。

# 2. 与短期记忆的区别

## 2.1 维度对比

**结论先行**：短期靠窗口、全量带入；长期靠存储、按相关性召回。

| 维度 | 短期记忆 | 长期记忆 |
|------|----------|----------|
| 范围 | 单次会话 | 跨会话 |
| 存储 | 会话存储 | 用户级存储 |
| 容量 | 小（窗口） | 大 |
| 召回 | 全量/窗口 | 按相关性 |
| 更新 | 追加 | 提取 + 合并 |
| 生命周期 | 会话结束即失效 | 持久，直到遗忘 |

## 2.2 存储位置与生命周期

### 2.2.1 短期记忆

短期记忆通常放在 Redis 或进程内，键为 `sessionId`，值为消息列表，TTL 设为会话闲置时长。

```java
public interface ShortTermMemory {
    List<Message> recent(String sessionId, int n);
    void append(String sessionId, Message message);
    void expire(String sessionId, Duration ttl);
}
```

### 2.2.2 长期记忆

长期记忆放在关系库（结构化）+ 向量库（语义），键为 `tenantId + userId`，无固定 TTL，靠遗忘机制管理。

```java
public interface LongTermMemory {
    void save(UserContext user, MemoryItem item);
    List<MemoryItem> recall(UserContext user, String question, int budget);
    void forget(String userId, LongTermMemory.ForgetPolicy policy);
}
```

## 2.3 边界与反例

| 反例 | 问题 | 修正 |
|------|------|------|
| 把偏好存进 Redis 会话键 | 换会话就丢 | 存用户级存储 |
| 把整段历史存进向量库 | 噪声大、成本高 | 先提取再存 |
| 长期记忆无 TTL | 无限膨胀 | 加遗忘机制 |
| 用长期记忆做实时状态 | 延迟高 | 实时状态放短期 |

# 3. 长期记忆的流程

## 3.1 写入流程

**结论先行**：写入发生在"会话结束或关键事件"时，经提取、去重合并、存储三步。

### 3.1.1 触发时机

- 会话结束（用户离开、超时）
- 关键事件（用户明确纠正、表达偏好）
- 定时任务（批量归纳历史）

### 3.1.2 提取

调用 LLM 从对话中抽取值得记住的信息，并给出重要性评分，见第 4 章。

### 3.1.3 合并存储

与已有记忆比对，去重、合并、冲突处理，再落库，见第 5、6 章。

## 3.2 读取流程

### 3.2.1 召回

新会话开始时，按用户召回相关记忆（结构化偏好 + 语义相关事实）。

### 3.2.2 注入

将召回结果格式化后注入系统提示，受 Token 预算约束。

## 3.3 全链路

```
写入：
  会话结束 / 关键事件
        │
        ▼
  提取值得记住的信息（LLM 提取）
        │
        ▼
  去重、合并、冲突处理
        │
        ▼
  存储（结构化 + 向量）

读取：
  新会话开始
        │
        ▼
  按用户召回相关记忆
        │
        ▼
  注入上下文（受 Token 预算）
```

## 3.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 同步写阻塞响应 | 提取调用 LLM 慢 | 异步队列写 |
| 重复触发 | 会话结束多次回调 | 幂等键 |
| 提取质量差 | 把寒暄也记住 | 明确正反例提示词 |
| 写入失败丢失 | 无重试 | 落盘 + 重试 |

# 4. 记忆提取

## 4.1 提取原则

**结论先行**：只记"稳定、可复用、用户相关"的信息，宁缺毋滥。

| 应记住 | 不应记住 |
|--------|----------|
| 身份、偏好、习惯 | 寒暄、客套 |
| 明确告知的事实 | 临时上下文 |
| 纠正与反馈 | 一次性口令 |

## 4.2 提取器接口

### 4.2.1 定义

```java
public record ExtractedMemory(
        MemoryType type,
        String content,
        double importance,
        Map<String, Object> metadata
) {}

interface MemoryExtractor {
    @UserMessage("""
            从下面的对话中提取值得长期记住的信息。

            值得记住：
            - 用户的身份、偏好、习惯
            - 用户明确告知的事实
            - 用户的纠正与反馈

            不值得记住：
            - 寒暄、无关内容
            - 临时信息

            对话：
            {{conversation}}

            输出 JSON 数组：
            [{"type":"PREFERENCE","content":"","importance":0.8}]
            """)
    List<ExtractedMemory> extract(String conversation);
}
```

### 4.2.2 结构化输出

LangChain4j 会把返回的 JSON 反序列化为 `List<ExtractedMemory>`；若模型输出不稳，可加 `@V` 变量与重试。

## 4.3 Spring AI 实现

```java
@Component
public class SpringAiExtractor {

    private final ChatClient chatClient;

    public SpringAiExtractor(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public List<ExtractedMemory> extract(String conversation) {
        return chatClient.prompt()
                .user(u -> u.text("""
                        提取值得长期记住的信息，输出 JSON 数组。
                        对话：
                        {conversation}
                        """).param("conversation", conversation))
                .call()
                .entity(new ParameterizedTypeReference<List<ExtractedMemory>>() {});
    }
}
```

## 4.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 模型返回非 JSON | 解析失败 | 结构化输出 + 重试 |
| importance 乱给 | 全 0.9 | 校准 + 阈值 |
| 提取过多 | 每轮都提 | 关键事件才提 |
| 多语言混杂 | 编码问题 | 统一 UTF-8 |

## 4.5 生产建议

- 提取与主对话解耦，走异步队列。
- 对提取结果做长度与敏感词校验。
- 保存原始对话引用（traceId）便于审计。

# 5. 存储设计

## 5.1 结构化存储

### 5.1.1 表设计

```sql
CREATE TABLE user_memory (
    id          BIGSERIAL PRIMARY KEY,
    user_id     VARCHAR(64)  NOT NULL,
    tenant_id   VARCHAR(64)  NOT NULL,
    type        VARCHAR(32)  NOT NULL,   -- PREFERENCE / FACT / FEEDBACK
    key         VARCHAR(128),            -- 如 "answer_style"
    content     TEXT         NOT NULL,
    importance  REAL         NOT NULL DEFAULT 0.5,
    created_at  TIMESTAMP    NOT NULL,
    updated_at  TIMESTAMP    NOT NULL,
    expires_at  TIMESTAMP
);

CREATE INDEX idx_user_memory ON user_memory (tenant_id, user_id, type);
```

### 5.1.2 索引与约束

- 复合索引 `(tenant_id, user_id, type)` 覆盖绝大多数查询。
- `key` 唯一约束（可选）：`UNIQUE (tenant_id, user_id, key)` 用于偏好类覆盖写。

## 5.2 向量存储

### 5.2.1 向量化

```java
// 记忆向量化，便于语义召回
TextSegment segment = TextSegment.from(
        memory.content(),
        Metadata.from(Map.of(
                "userId", userId,
                "tenantId", tenantId,
                "type", memory.type().name(),
                "importance", memory.importance())));

memoryStore.add(embeddingModel.embed(segment).content(), segment);
```

### 5.2.2 元数据

| 元数据键 | 用途 | 是否过滤 |
|----------|------|----------|
| userId | 用户隔离 | 必须 |
| tenantId | 租户隔离 | 必须 |
| type | 类型筛选 | 可选 |
| importance | 加权排序 | 可选 |

## 5.3 双写一致性

**结论先行**：结构化与向量库是两套存储，必须保证同一逻辑记忆的双写一致。

```java
@Transactional
public void saveWithOutbox(UserContext user, MemoryItem item) {
    store.save(item);              // ① 结构化
    outbox.save(new OutboxEvent(item.id()));   // ② 事务内写 outbox
}
// ③ 异步消费 outbox → 写向量库，失败重试
```

## 5.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 双写不一致 | 一边成功一边失败 | Outbox / 重试 |
| 无租户字段 | 跨租户泄露 | 强制 tenant_id |
| 向量维度变更 | 迁移困难 | 记录模型版本 |
| 大文本入库 | 召回噪声 | 摘要后入库 |

# 6. 去重与合并

## 6.1 相似度判定

**结论先行**：新增前先查相似记忆，命中则更新，未命中才新增。

```java
public void save(UserContext user, ExtractedMemory memory) {
    // ① 查找相似记忆
    List<MemoryItem> similar = findBySimilarity(
            user.userId(), memory.content(), 0.9);

    if (!similar.isEmpty()) {
        // ② 存在相似 → 更新（以新为准）
        MemoryItem existing = similar.get(0);
        existing.updateContent(memory.content());
        existing.setUpdatedAt(Instant.now());
        store.update(existing);
    } else {
        // ③ 不存在 → 新增
        store.save(toItem(user, memory));
    }
}
```

## 6.2 合并策略

### 6.2.1 冲突表

| 冲突类型 | 处理 |
|----------|------|
| 完全重复 | 忽略 |
| 相似但更新 | 覆盖旧值 |
| 矛盾 | 以时间新的为准 |
| 互补 | 合并 |

### 6.2.2 互补合并示例

```java
public MemoryItem merge(MemoryItem oldItem, ExtractedMemory newItem) {
    if (isContradictory(oldItem, newItem)) {
        return oldItem.overwriteWith(newItem);   // 新 > 旧
    }
    String merged = oldItem.content() + "；" + newItem.content();
    return oldItem.withContent(merged)
            .withImportance(Math.max(oldItem.importance(), newItem.importance()));
}
```

## 6.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 阈值过低 | 误合并不同事实 | 提高阈值 |
| 阈值过高 | 重复存储 | 结合 key 精确匹配 |
| 矛盾未处理 | 新旧并存 | 定义优先级 |
| 合并丢信息 | 直接覆盖 | 保留历史版本 |

# 7. 召回策略

## 7.1 三类召回

**结论先行**：偏好类走结构化全量，事实类走向量相关，经验类走时间+相似。

| 来源 | 方式 | 特点 |
|------|------|------|
| 偏好 | 结构化全量 | 小、必带 |
| 事实 | 向量相似 | 按需 |
| 经验 | 向量 + 时间 | 参考历史 |

## 7.2 混合排序

### 7.2.1 评分公式

```
score = similarity * w1 + importance * w2 + recency * w3
其中 w1 + w2 + w3 = 1
```

### 7.2.2 代码

```java
public List<MemoryItem> recall(UserContext user, String question, int budget) {
    // ① 结构化记忆（偏好类，全量小）
    List<MemoryItem> preferences = store.findByUserAndType(
            user.userId(), MemoryType.PREFERENCE);

    // ② 语义召回（事实类，按相关性）
    List<MemoryItem> relevant = vectorRecall(
            user.userId(), question, budget);

    // ③ 合并去重，按重要性排序
    return merge(preferences, relevant).stream()
            .sorted(Comparator.comparingDouble(MemoryItem::importance).reversed())
            .limit(budget)
            .toList();
}
```

## 7.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 只按相似度 | 召回无关 | 加权重要性/时效 |
| 不带偏好 | 风格不一致 | 偏好强制注入 |
| 召回过多 | 超预算 | budget 限制 |
| 不按用户过滤 | 越权 | 强制 userId |

## 7.4 生产建议

- 召回结果缓存（同问题短时间复用）。
- 记录召回命中率，用于调阈值。
- 对偏好类做"总是注入"，对事实类做"按需注入"。

# 8. 注入上下文

## 8.1 注入格式

**结论先行**：记忆要作为"背景知识"注入，而非命令，避免生硬复述。

```java
public String buildMemoryContext(UserContext user, String question) {
    List<MemoryItem> memories = recall(user, question, 10);

    if (memories.isEmpty()) return "";

    String memoryText = memories.stream()
            .map(m -> "- [%s] %s".formatted(m.type(), m.content()))
            .collect(Collectors.joining("\n"));

    return """
            【关于用户的已知信息】
            %s

            请结合这些信息回答，但不要生硬地复述。
            """.formatted(memoryText);
}
```

## 8.2 Token 预算

| 记忆类型 | 预算占比 | 说明 |
|----------|----------|------|
| 偏好 | 20% | 短、必带 |
| 事实 | 50% | 按相关性 |
| 经验 | 30% | 参考 |

## 8.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 注入过长 | 挤占对话 | 预算裁剪 |
| 与系统提示冲突 | 指令打架 | 明确优先级 |
| 生硬复述 | 体验差 | 提示"自然融入" |
| 注入他人记忆 | 越权 | 隔离校验 |

# 9. 遗忘机制

## 9.1 TTL 过期

```java
// ① TTL 过期
@Scheduled(cron = "0 0 2 * * *")
public void expireMemories() {
    store.deleteExpired(Instant.now());
}
```

## 9.2 重要性衰减

```java
// ② 重要性衰减
public void decay() {
    store.findAll().forEach(m -> {
        double daysSinceAccess = daysBetween(m.lastAccessedAt(), Instant.now());
        double decayed = m.importance() * Math.pow(0.95, daysSinceAccess);
        store.updateImportance(m.id(), decayed);
    });
}
```

## 9.3 容量淘汰

```java
// ③ 容量淘汰
public void evictIfNeeded(String userId) {
    long count = store.countByUser(userId);
    if (count > MAX_MEMORIES_PER_USER) {
        store.deleteLeastImportant(userId, count - MAX_MEMORIES_PER_USER);
    }
}
```

## 9.4 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 衰减过快 | 有用记忆丢失 | 调半衰期 |
| 淘汰误删 | 删掉偏好 | 偏好豁免 |
| 定时任务堆积 | 全表扫描慢 | 分批 + 索引 |
| 无访问追踪 | 无法衰减 | 记录 lastAccessed |

# 10. 用户可见与可控

## 10.1 管理接口

**结论先行**：用户必须能查看与删除自己的记忆，这是功能也是合规底线。

```java
// 提供记忆管理接口，让用户查看与删除
@RestController
@RequestMapping("/api/memories")
public class MemoryController {

    @GetMapping
    public List<MemoryItem> list(UserContext user) {
        return store.findByUser(user.userId());
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id, UserContext user) {
        store.deleteOwned(id, user.userId());      // 校验归属
    }

    @DeleteMapping
    public void clearAll(UserContext user) {
        store.deleteByUser(user.userId());
    }
}
```

## 10.2 合规要求

> 合规要求：用户有权查看与删除自己的记忆。

- 提供导出（可携带权）。
- 删除需覆盖结构化与向量两处。
- 记录删除审计日志。

## 10.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 只删一处 | 向量库残留 | 双删 + 重试 |
| 未校验归属 | 越权删他人 | deleteOwned |
| 无审计 | 无法追溯 | 审计日志 |

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 什么都记 | LLM 提取 + 重要性评分 |
| 记忆冲突 | 定义优先级（新 > 旧） |
| 无去重 | 相似度去重 |
| 无遗忘 | TTL + 衰减 + 容量淘汰 |
| 用户不可控 | 提供查看/删除 |
| 隐私泄露 | 脱敏 + 隔离 |
| 召回过多 | Token 预算限制 |

## 11.2 监控指标

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| 写入量/用户 | 记忆膨胀 | 超过上限 |
| 召回命中率 | 检索质量 | 过低 |
| 提取失败率 | 模型稳定性 | 突增 |
| 平均注入 Token | 成本 | 超预算 |

## 11.3 架构建议

- 写路径异步化，读路径缓存化。
- 结构化与向量库双写用 Outbox 保证最终一致。
- 按租户分片，避免单表过大。

# 12. 小结

- 长期记忆跨会话，让 Agent 记住用户与经验。
- 流程：提取 → 去重合并 → 存储 → 召回 → 注入。
- 用 LLM 提取值得记住的信息，并评重要性。
- 必须有遗忘机制：TTL、衰减、容量淘汰。
- 用户有权查看与删除自己的记忆。
- 存储上结构化与向量并存，双写一致性用 Outbox。
- 召回要按用户/租户隔离，并结合相似度、重要性、时效。
