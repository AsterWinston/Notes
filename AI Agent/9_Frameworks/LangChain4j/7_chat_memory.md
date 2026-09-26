# 1. 为什么需要记忆

结论先行：大语言模型（LLM，Large Language Model）本身是无状态的，每一次调用都是一次独立的函数调用，模型不会"记得"上一次说了什么。因此多轮对话的连续性必须由框架层显式维护，LangChain4j 用 `ChatMemory` 抽象统一解决这个问题。

## 1.1 大模型的无状态本质

### 1.1.1 无状态带来的问题

模型只对"本次请求里携带的 token"负责。请求里没有历史，模型就只能凭空回答。

```
无记忆：用户说"我叫张三"，下一轮"我叫什么？" → 模型不知道
有记忆：历史消息自动带入 → 模型记得"你叫张三"
```

这意味着：所谓"记忆"并不是模型的能力，而是"把历史消息重新塞回请求"的工程手段。

### 1.1.2 历史重放机制

每一轮对话，框架把历史消息与新消息拼成一条完整的消息列表，再发给模型：

```
用户第 1 轮: "我叫张三"
        │
        ▼
  [UserMessage] ──▶ ChatModel ──▶ [AiMessage] "你好，张三"
        │                              │
        └──────── ChatMemory ◀─────────┘
                     │
用户第 2 轮: "我叫什么？"
        │
        ▼
  ChatMemory.messages() = [UserMessage("我叫张三"),
                           AiMessage("你好，张三")]
        │
        ▼
  ChatModel(history + 新问题) ──▶ "你叫张三"
```

关键点：**记忆是客户端行为**，不是服务端会话。谁维护这段历史，谁就控制了上下文长度、成本与隐私。

## 1.2 记忆要解决的三个问题

### 1.2.1 连续性

没有记忆，多轮对话退化为单轮问答。代词消解、追问、上下文指代（"它""那个"）全部失效。记忆是连续性的唯一来源。

### 1.2.2 成本与窗口

历史无限增长会撞上模型的上下文窗口（Context Window），也会线性推高 token 费用。记忆必须带裁剪策略：要么按条数，要么按 token 预算。

### 1.2.3 个性化与合规

记忆里可能包含用户姓名、订单号、隐私信息。它既是"个性化"的数据源，也是"合规"的风险点，需要持久化、脱敏、审计与过期。

> 记忆与 RAG（Retrieval-Augmented Generation，检索增强生成）是两套机制：前者管对话历史，后者管外部知识。两者常同时使用。

# 2. ChatMemory 抽象

结论先行：LangChain4j 把"记忆"拆成两个正交抽象——`ChatMemory` 管策略（保留哪些、怎么裁剪），`ChatMemoryStore` 管存储（放在内存、数据库还是 Redis）。

## 2.1 ChatMemory 接口

### 2.1.1 接口方法详解

```java
public interface ChatMemory {

    Object id();                          // 记忆 ID（会话标识）

    void add(ChatMessage message);        // 追加一条消息

    List<ChatMessage> messages();         // 取出当前窗口内的消息

    void clear();                         // 清空该会话记忆
}
```

- `id()`：会话唯一标识，通常对应 `userId` 或 `userId:sessionId`。
- `add(...)`：写入并触发裁剪；裁剪策略由实现类决定。
- `messages()`：返回的是"裁剪后"的视图，而不是全量历史。
- `clear()`：语义是"删除该会话的记忆"，不只是清空内存。

### 2.1.2 id 与会话标识

`id` 决定了记忆的隔离边界。相同 `id` 共享记忆，不同 `id` 相互隔离。生产中不要用可变值（如时间戳）当 `id`，否则无法命中历史。

## 2.2 ChatMemoryStore 接口

### 2.2.1 三个方法

```java
public interface ChatMemoryStore {

    List<ChatMessage> getMessages(Object memoryId);

    void updateMessages(Object memoryId, List<ChatMessage> messages);

    void deleteMessages(Object memoryId);
}
```

注意 `updateMessages` 是"整体覆盖"语义，而不是增量追加。实现时应先删旧再写新，或用事务保证原子性。

### 2.2.2 策略与存储分离

```
   ChatMemory（策略层）
   ├─ MessageWindowChatMemory   保留最近 N 条
   ├─ TokenWindowChatMemory     按 token 预算保留
   │
   │  读写时委托
   ▼
   ChatMemoryStore（存储层）
   ├─ 默认：内存（重启即丢）
   ├─ 自定义：JDBC / Redis / MongoDB
   └─ 社区：官方与第三方持久化实现
```

> 记住一句话：`ChatMemory` 负责"窗口/裁剪策略"，`ChatMemoryStore` 负责"存储位置"。

# 3. 内置实现

结论先行：LangChain4j 内置两种窗口策略。消息窗口简单可预测，token 窗口精确但需要估算器。多数场景从消息窗口起步，成本敏感或长消息场景用 token 窗口。

## 3.1 MessageWindowChatMemory

### 3.1.1 窗口裁剪机制

它维护一个"最近 N 条消息"的滑动窗口。超出时从最旧的消息开始丢弃。有一个重要细节：它不会把 `SystemMessage` 丢弃，且会尽量保持"用户消息—AI 回复"的配对完整，避免出现孤立的消息。

### 3.1.2 用法

```java
// 消息窗口：保留最近 20 条消息
ChatMemory memory = MessageWindowChatMemory.withMaxMessages(20);

// 或使用 builder，便于挂载 ChatMemoryStore
ChatMemory memory = MessageWindowChatMemory.builder()
        .id("user-1")
        .maxMessages(20)
        .chatMemoryStore(store)
        .build();
```

## 3.2 TokenWindowChatMemory

### 3.2.1 Token 估算

按 token 预算保留消息，需要 `TokenCountEstimator`。常见实现是 `OpenAiTokenCountEstimator`，它按指定模型名估算 token 数。

```java
TokenCountEstimator estimator =
        new OpenAiTokenCountEstimator("gpt-4o-mini");
```

注意：估算器是按模型 tokenizer 近似计算的，与实际计费可能略有偏差，预算要留余量。

### 3.2.2 用法

```java
// Token 窗口：总量不超过 2000 token
ChatMemory memory = TokenWindowChatMemory.withMaxTokens(
        2000,
        new OpenAiTokenCountEstimator("gpt-4o-mini"));
```

## 3.3 选型对比

| 实现 | 策略 | 优点 | 缺点 |
|------|------|------|------|
| `MessageWindowChatMemory` | 保留最近 N 条消息 | 简单、可预测、无需估算器 | 长消息可能撑爆上下文 |
| `TokenWindowChatMemory` | 按 Token 预算保留 | 精确控制成本与窗口 | 需要 Token 估算器，估算有误差 |

边界与坑：

- 消息窗口下，一条超长工具返回可能一条就顶满窗口。
- token 窗口下，估算器与真实模型不匹配时，可能出现"估算没超、实际超限"。

生产建议：窗口大小按"最坏单条长度 × 2"反推，并对单条消息设置最大长度截断。

# 4. 在 AiServices 中使用

结论先行：`AiServices` 把 `ChatMemory` 自动接入调用链。单会话用 `chatMemory(...)`，多会话用 `@MemoryId` + `chatMemoryProvider(...)`。

## 4.1 单会话

### 4.1.1 绑定 chatMemory

```java
interface Assistant {
    String chat(String message);
}

Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
        .build();

assistant.chat("我叫张三");
assistant.chat("我叫什么？");   // → 你叫张三
```

### 4.1.2 生命周期

单会话模式下，`Assistant` 实例与一份记忆绑定。若把该实例作为单例共享给所有用户，就会发生"串话"。因此单会话仅适合 CLI、测试或"一人一实例"的场景。

## 4.2 多会话 @MemoryId

### 4.2.1 @MemoryId 语义

`@MemoryId` 标注的参数不参与提示内容，只用于定位记忆。它告诉框架"这次调用属于哪个会话"。

```java
interface Assistant {
    String chat(@MemoryId String userId, @UserMessage String message);
}
```

### 4.2.2 chatMemoryProvider

`chatMemoryProvider` 接收 `memoryId`，按需创建或取出对应的 `ChatMemory`：

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder()
                .id(memoryId)
                .maxMessages(20)
                .build())
        .build();

assistant.chat("user-1", "我叫张三");
assistant.chat("user-2", "我叫什么？");   // → 不知道（会话隔离）
assistant.chat("user-1", "我叫什么？");   // → 你叫张三
```

> `@MemoryId` 是会话隔离的关键，通常是 `userId` 或 `userId:sessionId`。

边界与坑：

- `memoryId` 必须来自可信上下文，绝不能直接用客户端传入的任意字符串，否则可越权读取他人记忆。
- 同一 `memoryId` 并发请求时，provider 可能创建多个记忆实例，导致写入竞争（见第 9 节）。

## 4.3 与 @SystemMessage 的关系

`@SystemMessage` 定义系统提示，每轮都会重新发送，但它**不重复存入记忆**。因此记忆里通常只有 `UserMessage`、`AiMessage` 与工具消息。若需要"动态系统提示"，应通过 `@V` 变量或动态参数注入，而不是塞进记忆。

## 4.4 流式输出与记忆

使用流式（Streaming）响应时，最终回复通过 `onCompleteResponse` 回调或 `TokenStream` 获取。记忆的写入必须发生在"完整回复"之后，而不是每个 token 都写。

```java
TokenStream stream = assistant.chat("讲个笑话");
stream.onPartialResponse(System.out::print)
      .onCompleteResponse(response -> {
          // 此时框架已将完整回复写入 ChatMemory
          System.out.println();
      })
      .onError(Throwable::printStackTrace)
      .start();
```

边界与坑：若流式过程中报错中断，可能只写入半截 `AiMessage`。生产建议在 `onError` 中回滚该轮记忆，或标记该轮为失败不参与后续上下文。

# 5. 持久化记忆

结论先行：默认 `ChatMemory` 存在内存里，进程重启即丢。生产必须把记忆落到外部存储，并配套 TTL（Time To Live，存活时间）与清理策略。

## 5.1 自定义 ChatMemoryStore（JDBC）

### 5.1.1 表结构

一种常见的表设计：一行存一条消息，或一行存整个会话的 JSON。前者便于查询与增量，后者实现简单。

```sql
CREATE TABLE chat_memory (
    conversation_id VARCHAR(128) NOT NULL,
    seq             INT          NOT NULL,
    message_json    TEXT         NOT NULL,
    updated_at      TIMESTAMP    NOT NULL,
    PRIMARY KEY (conversation_id, seq)
);
```

### 5.1.2 实现

```java
@Component
@RequiredArgsConstructor
public class JdbcChatMemoryStore implements ChatMemoryStore {

    private final ChatMemoryRepository repository;

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        return repository.findByConversationId(memoryId.toString());
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        repository.saveAll(memoryId.toString(), messages);
    }

    @Override
    public void deleteMessages(Object memoryId) {
        repository.deleteByConversationId(memoryId.toString());
    }
}
```

挂载到记忆：

```java
ChatMemoryStore store = new JdbcChatMemoryStore(repository);

ChatMemory memory = MessageWindowChatMemory.builder()
        .id("user-1")
        .maxMessages(20)
        .chatMemoryStore(store)
        .build();
```

## 5.2 Redis 实现（高并发 + TTL）

### 5.2.1 key 设计与 TTL

key 建议带命名空间前缀，如 `chat:memory:{id}`，并设置过期时间，避免冷会话永久占用内存。

### 5.2.2 实现

```java
@Component
@RequiredArgsConstructor
public class RedisChatMemoryStore implements ChatMemoryStore {

    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;
    private final Duration ttl = Duration.ofDays(7);

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        String json = redis.opsForValue().get(key(memoryId));
        return json == null ? List.of() : deserialize(json);
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        redis.opsForValue().set(key(memoryId), serialize(messages), ttl);
    }

    @Override
    public void deleteMessages(Object memoryId) {
        redis.delete(key(memoryId));
    }

    private String key(Object id) {
        return "chat:memory:" + id;
    }
}
```

> 也可使用社区提供的持久化实现（JDBC、Redis、MongoDB 等）。

## 5.3 序列化问题

`ChatMessage` 是接口，序列化时需保留具体类型（`UserMessage`/`AiMessage`/`ToolExecutionResultMessage`）与工具调用请求。直接反序列化成 `Map` 会丢失类型信息，导致再次发送时模型无法识别。推荐用官方提供的序列化工具或带类型字段的 DTO。

## 5.4 一致性

`updateMessages` 是整体覆盖。高并发下"读—改—写"会丢更新。解决方式：

- 会话级锁（本地或分布式），同一 `memoryId` 串行化；
- 或按 `memoryId` 做一致性哈希路由到固定节点；
- 或用带版本号的乐观锁，冲突时重试。

# 6. 记忆与消息类型

结论先行：记忆里保存的是完整的消息序列，而不只是"用户说过的话"。工具调用相关消息必须保留，否则模型无法理解历史行为。

## 6.1 消息序列组成

```
SystemMessage（一般由 @SystemMessage 提供，不重复存入）
UserMessage
AiMessage（含 toolExecutionRequests）
ToolExecutionResultMessage
```

## 6.2 工具调用消息

当模型决定调用工具时，会产生 `AiMessage`（携带 `toolExecutionRequests`），随后产生 `ToolExecutionResultMessage`。这两类消息必须成对保留。

```
UserMessage "北京天气？"
      │
      ▼
AiMessage(toolExecutionRequests=[getWeather("北京")])
      │
      ▼
ToolExecutionResultMessage("晴，25℃")
      │
      ▼
AiMessage("北京今天晴，25℃")
```

## 6.3 常见误区

| 误区 | 后果 |
|------|------|
| 只存 UserMessage | 模型不知道自己之前答过什么 |
| 丢弃工具消息 | 再次请求时工具调用状态不一致，可能报错 |
| 把 SystemMessage 存进记忆 | 重复注入、提示膨胀 |
| 存了但顺序错乱 | 模型对上下文理解错位 |

> 工具调用相关的消息也必须保留，否则模型无法理解"为什么之前调用了工具"。

# 7. 上下文压缩

结论先行：窗口记忆会截断早期信息，长对话必然丢上下文。压缩是把"被截断的历史"转成"一段摘要"重新注入，用较少 token 保留关键信息。

## 7.1 摘要压缩

### 7.1.1 思路

超过阈值时，把早期消息交给 LLM 生成摘要，摘要以 `SystemMessage` 形式放在最前，后面拼接最近若干条原始消息。

### 7.1.2 实现

```java
// 思路：超过阈值时，把早期消息交给 LLM 摘要
public List<ChatMessage> compress(List<ChatMessage> messages, int keepRecent) {
    if (messages.size() <= keepRecent + 2) return messages;

    List<ChatMessage> old = messages.subList(0, messages.size() - keepRecent);
    List<ChatMessage> recent = messages.subList(messages.size() - keepRecent, messages.size());

    String summary = summarizer.summarize(old);   // 调用 LLM 生成摘要

    List<ChatMessage> result = new ArrayList<>();
    result.add(SystemMessage.from("以下是之前对话的摘要：\n" + summary));
    result.addAll(recent);
    return result;
}
```

## 7.2 其他压缩策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 摘要压缩 | LLM 生成历史摘要 | 长对话、开放域 |
| 关键信息抽取 | 抽取实体/事实存为结构化 | 客服、订单 |
| 滑动窗口 | 只留最近 N 条 | 短会话 |
| 分层记忆 | 近期原文 + 远期摘要 | 复杂助手 |

## 7.3 压缩的边界

- 摘要是有损的，可能丢掉后续追问依赖的细节；
- 摘要本身消耗一次 LLM 调用，注意成本与延迟；
- 摘要应缓存，避免每轮重算；
- 摘要与"最近原文"之间要避免信息重复。

> 详见 `8_Memory/8_memory_compression.md`。

# 8. 记忆与 RAG 的分工

结论先行：记忆解决"对话连续性"，RAG 解决"外部知识"。两者数据源、时效、检索方式都不同，生产中通常同时使用。

## 8.1 对比

| 维度 | ChatMemory | RAG |
|------|------------|-----|
| 内容 | 本次对话历史 | 外部知识库 |
| 时效 | 会话内 | 长期 |
| 目的 | 连续性 | 知识补充 |
| 检索 | 顺序/窗口 | 语义相似度 |
| 存储 | 会话存储 | 向量库 |

## 8.2 组合模式

```
用户问题
   │
   ├─▶ ChatMemory ── 历史消息 ──┐
   │                             │
   └─▶ RAG ── 检索到的知识 ─────┤
                                 ▼
                       组装 Prompt → ChatModel
```

典型提示结构：`SystemMessage(角色)` + `检索资料` + `历史对话` + `当前问题`。注意两者都吃 token，需要统一做预算分配。

> 两者常同时使用：RAG 提供知识，Memory 提供对话连续性。

# 9. 生产注意事项

结论先行：记忆是"有状态"的组件，风险集中在数据丢失、增长失控、越权与并发。生产必须逐项设防。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 内存实现丢数据 | 生产必须持久化 |
| 记忆无限增长 | 窗口 + TTL + 摘要 |
| 会话越权 | 校验 memoryId 归属 |
| 上下文超限 | Token 窗口 + 裁剪 |
| 敏感信息落库 | 脱敏/加密 + 访问审计 |
| 并发写乱序 | 会话级锁或按会话路由 |

## 9.2 监控指标

| 指标 | 说明 |
|------|------|
| 平均消息条数 | 观察记忆膨胀趋势 |
| 平均 token 数 | 估算单轮成本 |
| 摘要触发率 | 判断压缩是否过频 |
| 存储命中率 | 评估缓存与 TTL 设置 |
| 越权拦截数 | 安全审计 |

生产建议：

- 给每条消息设最大长度，超长先截断再入库；
- 记忆写入与业务事务解耦，异步落库时保证顺序；
- 为 `ChatMemoryStore` 做降级：存储不可用时退回内存，避免整个对话不可用。

# 10. 小结

## 10.1 核心要点

- `ChatMemory` 管策略，`ChatMemoryStore` 管存储，二者分离。
- `@MemoryId` + `chatMemoryProvider` 实现多会话隔离。
- 生产必须持久化（JDBC/Redis）+ TTL + 压缩。
- 记忆存的是完整消息序列，包含工具调用消息。
- 记忆与 RAG 分工明确，常组合使用。

## 10.2 实践清单

| 检查项 | 目标 |
|--------|------|
| 记忆是否持久化 | 重启不丢 |
| 是否设置窗口上限 | 不撞上下文 |
| 是否有 TTL | 冷数据可回收 |
| memoryId 是否校验归属 | 不越权 |
| 是否保留工具消息 | 工具链完整 |
| 是否配置压缩 | 长对话可用 |
| 是否有监控 | 可观测 |
