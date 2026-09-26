# 1. 什么是短期记忆

**结论先行**：短期记忆（Short-term Memory）是当前会话（Session）内的消息历史，它让对话保持连续，是多轮交互的基础。

## 1.1 定义与作用

### 1.1.1 定义

短期记忆 = **当前会话内的消息历史**，让对话保持连续。它的生命周期与一次会话绑定，会话结束或超时后即失效。

### 1.1.2 连续性示例

```text
用户：我叫张三
AI：你好张三
用户：我叫什么？
AI：你叫张三          ← 依赖短期记忆
```

## 1.2 短期记忆与其他记忆的关系

| 记忆类型 | 与短期记忆的关系 |
|----------|------------------|
| 上下文 | 短期记忆是上下文的主要来源 |
| 长期记忆 | 跨会话持久化，短期记忆的补充 |
| 情景记忆 | 从短期记忆中提取重要事件形成 |

**边界与坑**：短期记忆不等于"全部历史"。它受窗口限制，超出部分必须靠摘要或长期记忆承接。

## 1.3 边界与生产建议

- 短期记忆应带 TTL，避免僵尸会话长期占用存储。
- 会话与用户、租户强绑定，防止串话。
- 明确"窗口内"与"摘要外"的边界，避免信息真空。

# 2. 存储结构

**结论先行**：短期记忆的数据模型需要包含会话标识、归属信息、消息列表、摘要、时间与 Token 统计。

## 2.1 数据模型

```java
public record ConversationMemory(
        String sessionId,
        String userId,
        String tenantId,
        List<Message> messages,
        String summary,               // 早期对话摘要
        Instant createdAt,
        Instant lastActiveAt,
        int totalTokens
) {}
```

## 2.2 字段说明

| 字段 | 作用 | 注意 |
|------|------|------|
| sessionId | 会话唯一标识 | 全局唯一或复合键 |
| userId | 归属用户 | 用于隔离校验 |
| tenantId | 租户标识 | 多租户必备 |
| messages | 消息序列 | 有序、含工具消息 |
| summary | 早期摘要 | 可为空 |
| totalTokens | Token 统计 | 用于预算 |
| createdAt | 创建时间 | 审计 |
| lastActiveAt | 最近活跃 | 用于清理 |

## 2.3 边界与生产建议

- 字段应可扩展，预留 metadata。
- totalTokens 随写入增量更新，避免每次重算。
- 摘要与消息列表要保证一致性，避免重复。

# 3. 窗口策略

**结论先行**：窗口策略决定保留多少历史，主流有消息窗口（按条数）与 Token 窗口（按 Token）两种，后者更精确。

## 3.1 消息窗口

```java
// Spring AI
ChatMemory memory = MessageWindowChatMemory.builder()
        .maxMessages(20)
        .build();

// LangChain4j
ChatMemory memory = MessageWindowChatMemory.withMaxMessages(20);
```

## 3.2 Token 窗口

```java
// 按 Token 预算保留，更精确
ChatMemory memory = TokenWindowChatMemory.builder()
        .maxTokens(4000, tokenCountEstimator)
        .build();
```

## 3.3 策略对比

| 策略 | 优点 | 缺点 |
|------|------|------|
| 消息窗口 | 简单可预测 | 长消息可能撑爆 |
| Token 窗口 | 精确控制 | 需 Token 估算器 |

## 3.4 摘要压缩

当早期消息被窗口挤出时，用摘要承接，避免信息丢失。

```java
public String summarize(List<Message> evicted) {
    String text = evicted.stream()
            .map(m -> m.role() + ": " + m.text())
            .collect(Collectors.joining("\n"));
    return chatClient.prompt()
            .system("请将以下对话压缩为不超过 200 字的要点，保留事实与偏好。")
            .user(text)
            .call()
            .content();
}
```

## 3.5 边界与生产建议

- 消息窗口应按"轮"而非"条"，避免工具对断裂。
- Token 窗口需统一估算器口径。
- 两种策略可叠加：先按条数限制，再按 Token 兜底。

# 4. 消息类型

**结论先行**：短期记忆必须保留完整的消息序列，尤其是工具调用（Tool Call）相关的请求与结果消息，缺一不可。

## 4.1 消息序列

```text
SystemMessage（通常不重复存储，由配置提供）
UserMessage
AiMessage（含工具调用请求）
ToolExecutionResultMessage
```

> **工具调用相关消息必须保留**，否则模型无法理解"为什么之前调用了工具"。

## 4.2 工具调用消息

一次工具调用的完整序列如下：

```text
AiMessage(toolCalls=[getWeather(city="北京")])
   │
   ▼
ToolExecutionResultMessage(id=..., result="晴 25℃")
   │
   ▼
AiMessage("北京今天晴，25℃")
```

## 4.3 边界与生产建议

- 工具消息与请求消息必须成对存储与裁剪。
- SystemMessage 通常不写入历史，由配置统一注入。
- 序列化要保留消息类型信息，反序列化才能还原。

# 5. 存储实现

**结论先行**：生产环境用 Redis 或数据库存储短期记忆；Redis 适合高频读写与 TTL，数据库适合审计与复杂查询。

## 5.1 Redis（推荐）

```java
@Component
@RequiredArgsConstructor
public class RedisConversationMemory {

    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;
    private final Duration ttl = Duration.ofDays(7);

    public void append(String sessionId, Message message) {
        String key = key(sessionId);
        String json = serialize(message);

        redis.opsForList().rightPush(key, json);
        redis.expire(key, ttl);

        // 限制长度
        Long size = redis.opsForList().size(key);
        if (size != null && size > MAX_MESSAGES) {
            redis.opsForList().trim(key, size - MAX_MESSAGES, -1);
        }
    }

    public List<Message> getMessages(String sessionId) {
        List<String> jsons = redis.opsForList().range(key(sessionId), 0, -1);
        return jsons == null ? List.of()
                : jsons.stream().map(this::deserialize).toList();
    }

    private String key(String sessionId) {
        return "memory:conversation:" + sessionId;
    }
}
```

## 5.2 数据库

```sql
CREATE TABLE conversation_message (
    id          BIGSERIAL PRIMARY KEY,
    session_id  VARCHAR(128) NOT NULL,
    tenant_id   VARCHAR(64)  NOT NULL,
    role        VARCHAR(32)  NOT NULL,
    content     TEXT         NOT NULL,
    metadata    JSONB,
    created_at  TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE INDEX idx_conv_session ON conversation_message (session_id, created_at);
```

## 5.3 选型对比

| 维度 | Redis | 数据库 |
|------|-------|--------|
| 读写性能 | 高 | 中 |
| TTL | 原生支持 | 需定时任务 |
| 持久化 | 弱 | 强 |
| 查询能力 | 弱 | 强 |
| 成本 | 内存贵 | 磁盘便宜 |

## 5.4 边界与生产建议

- Redis 只存近期热数据，冷数据落库归档。
- 使用 pipeline 批量写入，降低往返延迟。
- 键名带租户前缀，便于分片与清理。

# 6. 上下文组装

**结论先行**：组装时按"摘要 + 近期消息 + 当前问题"的顺序拼接，并受 Token 预算约束。

## 6.1 组装流程

```text
加载会话记忆
   │
   ├──▶ 摘要（早期对话）
   ├──▶ 近期消息（窗口）
   └──▶ 当前问题
         │
         ▼
   最终消息列表（受预算）
```

## 6.2 Java 实现

```java
public List<Message> buildContext(String sessionId, String question) {
    ConversationMemory memory = store.load(sessionId);

    List<Message> messages = new ArrayList<>();

    // ① 摘要（早期对话）
    if (memory.summary() != null) {
        messages.add(SystemMessage.from(
                "之前对话摘要：\n" + memory.summary()));
    }

    // ② 近期消息（窗口）
    List<Message> recent = memory.messages();
    if (recent.size() > WINDOW_SIZE) {
        recent = recent.subList(recent.size() - WINDOW_SIZE, recent.size());
    }
    messages.addAll(recent);

    // ③ 当前问题
    messages.add(UserMessage.from(question));

    return messages;
}
```

## 6.3 摘要与窗口去重

摘要已包含的早期内容不应再出现在窗口中，否则会重复占用预算。

```java
public List<Message> dedup(List<Message> recent, String summary) {
    if (summary == null || summary.isBlank()) {
        return recent;
    }
    return recent.stream()
            .filter(m -> !summary.contains(truncate(m.text(), 20)))
            .toList();
}
```

## 6.4 边界与生产建议

- 摘要与窗口内容可能重复，需去重。
- 组装后复核 Token，超限则裁剪。
- 保留消息 ID，便于追踪来源。

# 7. 记忆写入时机

**结论先行**：读在请求前，写在响应后；工具调用结果也要在工具执行后写入，保证序列完整。

## 7.1 三个时机

| 时机 | 说明 |
|------|------|
| 请求前 | 读记忆组装上下文 |
| 响应后 | 写用户消息与模型回复 |
| 工具调用后 | 写工具结果 |

## 7.2 完整对话实现

```java
public String chat(String sessionId, String question) {
    // ① 读
    List<Message> context = buildContext(sessionId, question);

    // ② 调用
    String answer = chatClient.prompt().messages(context).call().content();

    // ③ 写（用户消息 + 模型回复）
    memory.append(sessionId, UserMessage.from(question));
    memory.append(sessionId, AiMessage.from(answer));

    return answer;
}
```

## 7.3 工具结果写入

工具执行结果必须在执行后立即写入，与请求消息成对保存。

```java
public String chatWithTools(String sessionId, String question) {
    List<Message> context = buildContext(sessionId, question);
    AiMessage ai = chatClient.prompt().messages(context).call().chatResponse()
            .getResult().getOutput();

    memory.append(sessionId, UserMessage.from(question));
    memory.append(sessionId, ai);

    // 若模型请求调用工具，执行并写入结果
    if (ai.hasToolCalls()) {
        for (ToolCall call : ai.getToolCalls()) {
            String result = toolExecutor.execute(call);
            memory.append(sessionId,
                    ToolExecutionResultMessage.from(call, result));
        }
    }
    return ai.getText();
}
```

## 7.4 边界与生产建议

- 写入应幂等，避免重试导致重复消息。
- 写入失败要有补偿，避免历史缺失。
- 工具结果写入必须与工具请求对应。

# 8. 并发控制

**结论先行**：同一会话并发写入会导致消息乱序，需用会话级锁串行化。

## 8.1 并发问题

```text
并发写入导致乱序：
  线程A：写 UserMessage(Q1)
  线程B：写 UserMessage(Q2)
  实际顺序：Q1 → Q2 → A1 → A2   ← 问答错位
```

## 8.2 分布式锁实现

```java
public String chatWithLock(String sessionId, String question) {
    RLock lock = redisson.getLock("memory:lock:" + sessionId);

    if (!lock.tryLock(3, 30, TimeUnit.SECONDS)) {
        throw new TooManyRequestsException("会话正忙");
    }
    try {
        return doChat(sessionId, question);
    } finally {
        lock.unlock();
    }
}
```

## 8.3 队列串行化

当并发量高时，可用消息队列按会话分区串行消费，替代分布式锁。

```text
请求 ──▶ Kafka(topic=chat, key=sessionId)
                     │
                     ▼
        同一 sessionId 落到同一分区
                     │
                     ▼
            单消费者串行处理
```

## 8.4 边界与生产建议

- 锁粒度到会话，避免全局锁。
- 设置锁超时，防止死锁。
- 高并发场景可改为消息队列串行消费。

# 9. 会话隔离

**结论先行**：会话数据必须校验归属，防止水平越权（Horizontal Privilege Escalation）导致跨用户读取。

## 9.1 越权风险

| 风险 | 后果 |
|------|------|
| 不校验归属 | 读取他人会话 |
| 可枚举 sessionId | 批量越权 |
| 无租户隔离 | 跨租户泄漏 |

## 9.2 归属校验实现

```java
public ConversationMemory load(String sessionId, UserContext user) {
    ConversationMemory memory = store.load(sessionId);

    // 归属校验，防水平越权
    if (!memory.tenantId().equals(user.tenantId())
            || !memory.userId().equals(user.userId())) {
        throw new AccessDeniedException("无权访问该会话");
    }
    return memory;
}
```

## 9.3 边界与生产建议

- sessionId 使用不可预测的随机值（UUID）。
- 校验在服务层统一拦截，不依赖前端。
- 记录越权访问尝试，触发安全告警。

# 10. 生产注意事项

**结论先行**：短期记忆的风险集中在存储、长度、并发、越权、工具消息、脱敏与监控七个方面。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 内存存储 | 生产用 Redis/DB |
| 无长度限制 | 窗口 + TTL |
| 并发乱序 | 会话级锁 |
| 越权访问 | 归属校验 |
| 工具消息丢失 | 完整保留序列 |
| 无脱敏 | 入库前脱敏 |
| 无监控 | 监控会话大小与数量 |

## 10.2 监控指标

| 指标 | 含义 | 告警建议 |
|------|------|----------|
| session.messages.count | 单会话消息数 | 异常增长告警 |
| session.active.count | 活跃会话数 | 突增排查 |
| memory.write.latency | 写入延迟 | P99 超阈值 |
| session.token.usage | 会话 Token | 接近预算告警 |

## 10.3 实施清单

- 选定存储并配置 TTL。
- 实现窗口策略与摘要。
- 加入会话级锁与归属校验。
- 入库前脱敏，接入监控。
- 定期清理僵尸会话。

# 11. 小结

## 11.1 核心结论

- 短期记忆是会话内的消息历史。
- 用窗口策略控制大小，Redis 存储 + TTL 最实用。
- 必须保留工具调用相关消息。
- 会话并发用锁保证顺序，访问要校验归属。

## 11.2 实践要点

- 写入时机：响应后写用户消息与模型回复。
- 摘要承接早期内容，避免窗口外的信息真空。
- 监控会话大小、数量与 Token 使用。
