# 1. 为什么需要记忆

结论先行：大模型本身是无状态的，多轮对话的本质是"每一轮都把历史重新发一遍"。

## 1.1 无状态带来的问题

大模型是**无状态**的：每次调用都从零开始。要实现多轮对话，必须把历史消息重新带给模型。

```
无记忆：
  用户：我叫张三
  模型：你好张三
  用户：我叫什么？      ← 模型不知道，因为上一次请求已结束
  模型：抱歉，我不知道你的名字

有记忆：
  用户：我叫张三
  模型：你好张三
  用户：我叫什么？
  模型：你叫张三        ← 历史被重新注入
```

## 1.2 记忆的两种含义

### 1.2.1 会话记忆 vs 长期记忆

| 类型 | 存储对象 | 生命周期 | 本文件 |
|------|----------|----------|--------|
| 会话记忆 | 原始消息列表 | 会话内 | 是 |
| 长期记忆 | 用户画像/偏好 | 跨会话 | 否（见 `8_Memory`） |

### 1.2.2 为什么不用模型微调

微调无法承载"临时会话状态"，且成本高、不可实时更新；注入历史是最轻量的方案。

## 1.3 记忆的代价

结论先行：记忆不是免费的，它用 Token、延迟与隐私风险换取多轮连贯性。

### 1.3.1 三类成本

| 成本 | 表现 | 缓解 |
|------|------|------|
| Token | 每轮重发全部历史 | 窗口 + 摘要 |
| 延迟 | 输入越长首字越慢 | 限制窗口 |
| 隐私 | 历史可能含 PII | 脱敏 + TTL |

### 1.3.2 一个反例

把用户全部历史无上限地塞进每轮请求，短期"效果很好"，但第 50 轮时输入可能已达数万 Token，成本和延迟双双失控，且早期无关内容会稀释当前意图。

# 2. ChatMemory 抽象

结论先行：`ChatMemory` 只做一件事——按 `conversationId` 存取消息列表。

## 2.1 接口定义

```java
public interface ChatMemory {
    void add(String conversationId, List<Message> messages);
    List<Message> get(String conversationId);
    void clear(String conversationId);
}
```

## 2.2 关键概念

### 2.2.1 conversationId

- **conversationId**：会话隔离键，通常由「用户ID + 会话ID」组成。
- 记忆本质是"**按会话存取消息列表**"，与向量长期记忆（`8_Memory`）不同。

### 2.2.2 消息类型

| 消息 | 说明 |
|------|------|
| `UserMessage` | 用户输入 |
| `AssistantMessage` | 模型输出 |
| `SystemMessage` | 系统提示 |
| `ToolResponseMessage` | 工具结果 |

## 2.3 读写时序

结论先行：Advisor 在每次调用前 `get` 历史、调用后 `add` 本轮消息。

### 2.3.1 一次带记忆的调用

```
prompt() 触发
   │
   ▼
Advisor: memory.get(conversationId)   → 历史消息
   │
   ▼
历史 + 本轮输入 一起发给模型
   │
   ▼
Advisor: memory.add(conversationId, [user, assistant])
```

### 2.3.2 写入时机

写入发生在响应成功之后；若调用失败，本轮消息通常不会入库，避免污染历史。

# 3. 内置实现

结论先行：开发用内存，生产必须持久化。

## 3.1 实现对比

| 实现 | 存储 | 适用 |
|------|------|------|
| `InMemoryChatMemory` | JVM 内存 | 开发/单机 |
| `JdbcChatMemoryRepository` | 关系型数据库 | 生产、持久化 |
| `RedisChatMemoryRepository` | Redis | 生产、高性能、可过期 |
| `MongoChatMemoryRepository` | MongoDB | 文档型存储 |
| `CassandraChatMemoryRepository` | Cassandra | 大规模分布式 |

## 3.2 存储抽象的分层

### 3.2.1 Repository 与 ChatMemory 的关系

`ChatMemory` 是策略层（窗口、摘要），`ChatMemoryRepository` 是存储层；两者通过 builder 组合，可以独立替换。

### 3.2.2 内存实现的局限

> 生产环境**不要用 `InMemoryChatMemory`**：重启即丢失，且无法水平扩展。

## 3.3 自定义存储实现

结论先行：内置存储不满足时，实现 `ChatMemoryRepository` 接口即可接入任意后端。

### 3.3.1 接口契约

```java
public interface ChatMemoryRepository {
    List<Message> findByConversationId(String conversationId);
    void saveAll(String conversationId, List<Message> messages);
    void deleteByConversationId(String conversationId);
}
```

### 3.3.2 选型建议

| 后端 | 场景 |
|------|------|
| 对象存储 | 冷会话归档 |
| 图数据库 | 需要消息关系 |
| 自研分片 | 超大规模 |

# 4. 窗口记忆：MessageWindowChatMemory

结论先行：窗口记忆是最简单有效的策略——只保留最近 N 条。

## 4.1 基本用法

最常用的策略是"只保留最近 N 条消息"：

```java
ChatMemory memory = MessageWindowChatMemory.builder()
        .chatMemoryRepository(new InMemoryChatMemoryRepository())
        .maxMessages(20)                 // 最多保留 20 条
        .build();
```

## 4.2 截断行为

```
消息序列（旧 → 新）：
  M1 M2 M3 M4 M5 M6 M7 M8 M9 M10
                 │
        maxMessages=5 时
                 ▼
              M6 M7 M8 M9 M10   ← 只带这些给模型
```

### 4.2.1 截断边界

> 注意：窗口会**截断早期信息**（如用户最初说的名字），需要配合摘要记忆（见 `8_Memory/8_memory_compression.md`）。

### 4.2.2 系统消息的处理

系统消息通常不应占用窗口，或需单独保留，避免被截断导致人设丢失。

## 4.3 摘要窗口与混合策略

结论先行：窗口只能"丢弃"，无法"记住"，长对话需要摘要补位。

### 4.3.1 摘要触发条件

- 消息数超过阈值；
- 或估算 Token 超过预算；
- 触发时把最旧的一批消息交给 LLM 压缩成一段摘要。

### 4.3.2 摘要的存储位置

摘要可作为一条 `SystemMessage` 或专门的摘要消息，排在窗口消息之前，保证模型先看到背景。

# 5. 在 ChatClient 中启用记忆

结论先行：记忆通过 `MessageChatMemoryAdvisor` 注入，会话由调用时的 `CONVERSATION_ID` 参数决定。

## 5.1 通过 Advisor 注入

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
        .maxMessages(20)
        .build();

ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
```

## 5.2 调用时指定会话

```java
@GetMapping("/chat")
public String chat(@RequestParam String sessionId,
                   @RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(
                    ChatMemory.CONVERSATION_ID, sessionId))
            .call()
            .content();
}
```

`ChatMemory.CONVERSATION_ID` 是约定的参数名，Advisor 据此读写对应会话。

### 5.2.1 MessageChatMemoryAdvisor vs PromptChatMemoryAdvisor

| Advisor | 注入方式 | 特点 |
|---------|----------|------|
| `MessageChatMemoryAdvisor` | 作为独立消息插入 | 结构清晰，推荐 |
| `PromptChatMemoryAdvisor` | 拼进系统提示 | 老式兼容用法 |

### 5.2.2 完整示例

```java
@RestController
@RequestMapping("/chat")
public class MemoryChatController {

    private final ChatClient chatClient;

    public MemoryChatController(ChatModel chatModel, ChatMemory memory) {
        this.chatClient = ChatClient.builder(chatModel)
                .defaultSystem("你是一个友好的中文助手。")
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(memory).build())
                .build();
    }

    @PostMapping("/{sessionId}")
    public String chat(@PathVariable String sessionId,
                       @RequestBody String message) {
        return chatClient.prompt()
                .user(message)
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
                .call()
                .content();
    }
}
```

## 5.3 会话初始化与清理

结论先行：会话是有生命周期的资源，需要创建、列举与删除。

### 5.3.1 清空会话

```java
chatMemory.clear(conversationId);
```

### 5.3.2 生命周期

| 操作 | 触发 | 说明 |
|------|------|------|
| 创建 | 首次消息 | 惰性创建 |
| 过期 | TTL 到期 | Redis 自动 |
| 归档 | 用户主动 | 转冷存储 |

# 6. 持久化记忆：JDBC

结论先行：需要强一致、可审计时选 JDBC。

## 6.1 依赖与建表

### 6.1.1 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### 6.1.2 建表

```sql
CREATE TABLE IF NOT EXISTS spring_ai_chat_memory (
    conversation_id VARCHAR(255) NOT NULL,
    content         TEXT         NOT NULL,
    type            VARCHAR(50)  NOT NULL,
    timestamp       TIMESTAMP    NOT NULL,
    PRIMARY KEY (conversation_id, timestamp)
);
```

## 6.2 配置与装配

### 6.2.1 配置

```yaml
spring:
  ai:
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always
```

### 6.2.2 Bean 装配

```java
@Bean
ChatMemory chatMemory(JdbcChatMemoryRepository repository) {
    return MessageWindowChatMemory.builder()
            .chatMemoryRepository(repository)
            .maxMessages(30)
            .build();
}
```

# 7. 持久化记忆：Redis

结论先行：高并发、需要 TTL 的场景选 Redis。

## 7.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-redis</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    chat:
      memory:
        repository:
          redis:
            host: localhost
            port: 6379
            time-to-live: 7d          # 会话自动过期
```

## 7.2 选型对比

| 维度 | JDBC | Redis |
|------|------|-------|
| 一致性 | 强 | 最终 |
| 过期 | 需自建 | 原生 TTL |
| 审计 | 易 | 难 |
| 性能 | 中 | 高 |

> Redis 适合高并发、需要 TTL 的场景；JDBC 适合需要强一致与审计的场景。

# 8. 会话管理要点

结论先行：会话管理 = 隔离 + 安全 + 并发。

## 8.1 conversationId 设计

```java
String conversationId = userId + ":" + sessionId;
```

| 方案 | 说明 |
|------|------|
| 只用 sessionId | 简单，但跨设备不通用 |
| userId + sessionId | 可隔离、可列举用户所有会话（推荐） |
| tenantId + userId + sessionId | 多租户场景 |

## 8.2 会话隔离与安全

### 8.2.1 水平越权

- **必须校验 conversationId 归属**，防止用户 A 读取用户 B 的会话（水平越权）。
- 多租户下要按租户隔离存储与密钥。

### 8.2.2 敏感信息

会话历史可能包含 PII，需要脱敏或加密存储，并控制访问审计。

## 8.3 并发控制

同一会话并发写入可能乱序。可用：

- 会话级分布式锁；
- 或按会话路由到同一处理线程；
- 或写入时带时间戳 + 乐观锁。

> 详见 `14_ProductionEngineering/Runtime/2_session_management.md`。

## 8.4 会话迁移与共享

结论先行：跨端、跨设备是常见需求，迁移前必须先做归属校验。

### 8.4.1 跨端续聊

同一 `userId` 下不同 `sessionId` 是不同会话；如需跨端续聊，可把 `sessionId` 提升为用户级。

### 8.4.2 共享会话

多人共享同一会话时，需记录每条消息的发言人，避免模型混淆主体。

# 9. 记忆的进阶策略

结论先行：窗口只是起点，长对话需要摘要 + 向量 + 长期记忆组合。

## 9.1 策略对比

| 策略 | 说明 |
|------|------|
| 窗口记忆 | 保留最近 N 条（本文件） |
| 摘要记忆 | 用 LLM 压缩早期对话（`8_Memory/8_memory_compression.md`） |
| 向量记忆 | 把历史向量化，按相关性召回（`8_Memory/6_vector_memory.md`） |
| 长期记忆 | 跨会话持久化用户画像与偏好（`8_Memory/4_long_term_memory.md`） |

## 9.2 组合模式

### 9.2.1 窗口 + 摘要

```
[摘要消息] 早期对话已被压缩为一段摘要
[窗口消息] 最近 N 条原始消息
[本轮消息] 用户最新输入
```

### 9.2.2 窗口 + 向量召回

窗口保证近期连贯，向量召回保证远期相关，二者互补。

## 9.3 记忆召回质量

结论先行：记忆是否"有用"，取决于召回内容与当前问题的相关性，而非数量。

### 9.3.1 评估维度

| 维度 | 问题 |
|------|------|
| 相关性 | 召回内容是否服务当前问题 |
| 时效 | 是否混入过时信息 |
| 一致性 | 是否与最新事实冲突 |

### 9.3.2 冲突处理

新旧信息冲突时，应显式标注时间或让新信息优先，避免模型采信过期记忆。

# 10. 生产注意事项

结论先行：记忆系统的三条生命线——隔离、TTL、Token 预算。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 记忆无限增长 | 设 `maxMessages` + TTL |
| 上下文超限 | 结合 Token 预算裁剪（`14/Performance/3`） |
| 敏感信息落库 | 脱敏、加密、访问审计 |
| 会话越权 | 校验 conversationId 归属 |
| 重启丢记忆 | 用持久化存储 |
| 成本失控 | 历史越长，每轮 Token 越多，需监控 |

## 10.2 容量估算

### 10.2.1 Token 增长

每轮请求 Token ≈ 系统提示 + 窗口内所有消息 + 本轮输入。窗口越大，边际成本越高。

### 10.2.2 存储估算

单条消息按平均 200 字节计，日活 1 万、人均 20 轮，则日增约 40MB，需按 TTL 清理。

# 11. 小结

## 11.1 要点回顾

- 记忆 = 按 `conversationId` 存取消息列表。
- 生产必须持久化（JDBC/Redis），别用内存实现。
- 窗口记忆是最简策略，长对话需配合摘要与向量记忆。
- 会话隔离、TTL、Token 预算是记忆系统的三条生命线。

## 11.2 延伸阅读

- 记忆压缩：`8_Memory/8_memory_compression.md`
- 向量记忆：`8_Memory/6_vector_memory.md`
- 会话管理：`14_ProductionEngineering/Runtime/2_session_management.md`
