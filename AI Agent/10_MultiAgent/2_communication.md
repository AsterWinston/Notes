# 1. 通信方式总览

**结论先行**：多智能体系统的复杂度，大半来自"Agent 之间怎么说话"。通信方式决定了系统的耦合度、可观测性与失败模式。四种基本方式——直接消息、共享状态、消息队列、事件总线——各有适用边界，实践中常组合使用。

**原理**：Agent 之间需要交换三样东西：意图（要做什么）、数据（产物与上下文）、控制权（下一步谁做）。不同通信方式，本质上是这三样东西的载体不同。

| 方式 | 载体 | 耦合度 | 同步性 | 适用场景 |
|------|------|--------|--------|----------|
| 直接消息 | 方法调用 / RPC | 高 | 同步为主 | 简单委托、强顺序 |
| 共享状态 | 共享内存 / 黑板 | 中 | 异步 | 协作完善同一产物 |
| 消息队列 | 中间件（Kafka/MQ） | 低 | 异步 | 解耦、削峰、跨进程 |
| 事件总线 | 主题 / 订阅 | 低 | 异步 | 一对多广播 |

## 1.1 四种基本通信方式

**结论先行**：先理解每种方式的"数据放在哪、谁来触发、失败怎么办"，再谈选型。

### 1.1.1 直接消息

**原理**：A 直接调用 B 的处理方法，把消息作为参数传入，返回值即为 B 的响应。控制流清晰、调试容易，但 A 必须知道 B 的存在与地址。

```java
public record AgentMessage(
        String from,
        String to,
        String content,
        Map<String, Object> metadata,
        Instant timestamp
) {}

public class DirectMessaging {

    private final AgentRegistry registry;

    public DirectMessaging(AgentRegistry registry) {
        this.registry = registry;
    }

    public String send(String from, String to, String content) {
        Agent target = registry.get(to);
        if (target == null) {
            throw new IllegalArgumentException("未知 Agent: " + to);
        }
        return target.handle(new AgentMessage(from, to, content, Map.of(), Instant.now()));
    }
}
```

**边界与坑**：直接消息是同步阻塞的。若 B 内部又调用 A，极易形成调用环；跨进程时"直接调用"实际退化为 RPC（远程过程调用），必须补上超时与重试。

**生产建议**：进程内委托用直接消息即可；一旦需要超时、重试、熔断，就应改用异步消息或队列。

### 1.1.2 共享状态（黑板）

**原理**：所有 Agent 读写同一块"黑板"，各自只关心自己需要的字段，产物逐层叠加。

```
所有 Agent 读写同一块"黑板"：

  ┌─────────────────────────────┐
  │          Blackboard          │
  │  question: ...               │
  │  retrievedDocs: [...]        │
  │  analysis: ...               │
  │  draft: ...                  │
  └─────────────────────────────┘
      ▲        ▲         ▲
      │        │         │
   检索Agent  分析Agent  写作Agent
```

**边界与坑**：黑板是隐式耦合——Agent 不直接依赖彼此，但依赖"字段约定"。字段一改，全体受影响。

**生产建议**：把黑板字段定义成不可变记录（record）或明确的 Schema，并做版本管理。

### 1.1.3 消息队列

**原理**：发送方把消息投递到队列，接收方异步消费。生产与消费解耦，天然支持削峰与重试。

```java
public record TaskMessage(
        String messageId,
        String taskId,
        String payload,
        int attempt,
        Instant createdAt
) {}

// 生产者：投递任务
public void dispatch(TaskMessage message) {
    rabbitTemplate.convertAndSend("agent.tasks", message);
}

// 消费者：幂等处理
@RabbitListener(queues = "agent.tasks")
public void onMessage(TaskMessage message) {
    if (dedupRepo.exists(message.messageId())) {
        return;                     // 幂等：重复消息直接丢弃
    }
    worker.handle(message.payload());
    dedupRepo.save(message.messageId());
}
```

**边界与坑**：异步意味着"结果不再即时可得"，需要回调或轮询；消息可能重复投递，消费端必须幂等。

**生产建议**：为每条消息生成唯一 messageId 并做去重表；设置死信队列（DLQ）承接反复失败的消息。

### 1.1.4 事件总线

**原理**：发布者把事件广播到主题，多个订阅者各自响应。适合"一个事件触发多方反应"。

```java
public record AgentEvent(String type, String source, Map<String, Object> payload) {}

public class EventBus {

    private final Map<String, List<Consumer<AgentEvent>>> subscribers = new ConcurrentHashMap<>();

    public void subscribe(String type, Consumer<AgentEvent> handler) {
        subscribers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>()).add(handler);
    }

    public void publish(AgentEvent event) {
        subscribers.getOrDefault(event.type(), List.of())
                .forEach(handler -> handler.accept(event));
    }
}
```

**边界与坑**：广播会放大成本——N 个订阅者就是 N 次调用；事件顺序在多订阅者场景下不保证。

**生产建议**：限制订阅者数量，给每个订阅者设独立超时与熔断；事件只承载"发生了什么"，不承载"要做什么"。

## 1.2 选型对比

**结论先行**：没有最好，只有匹配。用"耦合度需求 + 同步性需求 + 扇出规模"三个维度决策。

| 维度 | 直接消息 | 共享状态 | 消息队列 | 事件总线 |
|------|----------|----------|----------|----------|
| 耦合度 | 高 | 中 | 低 | 低 |
| 同步性 | 同步 | 异步 | 异步 | 异步 |
| 扇出 | 1:1 | N:N | 1:1 | 1:N |
| 调试难度 | 低 | 中 | 高 | 高 |
| 持久化 | 无 | 视实现 | 有 | 视实现 |
| 典型框架 | Spring Bean | LangGraph State | Kafka/RabbitMQ | Spring Events |

**边界与坑**：不要为了"看起来解耦"而过早引入队列。单机、低频、强顺序的场景，直接消息反而更可靠。

# 2. 直接消息

**结论先行**：直接消息是多智能体通信的最小单元。把它设计对（结构化、可追溯、有超时），上层编排才稳。

**原理**：一条消息 = 谁发的 + 发给谁 + 说了什么 + 怎么关联 + 何时发的。这五个要素缺一不可。

## 2.1 消息模型

**结论先行**：消息必须是结构化对象，而不是裸字符串。

### 2.1.1 AgentMessage 记录

**原理**：用 Java record 定义不可变消息，天然线程安全、便于序列化与日志。

```java
public record AgentMessage(
        String id,
        String from,
        String to,
        MessageType type,
        String content,
        Map<String, Object> artifacts,
        String correlationId,
        Instant timestamp
) {

    public static AgentMessage request(String from, String to, String content) {
        return new AgentMessage(
                UUID.randomUUID().toString(), from, to,
                MessageType.REQUEST, content, Map.of(),
                UUID.randomUUID().toString(), Instant.now());
    }

    public AgentMessage reply(String content) {
        return new AgentMessage(
                UUID.randomUUID().toString(), to, from,
                MessageType.RESPONSE, content, Map.of(),
                correlationId, Instant.now());
    }
}
```

**边界与坑**：`artifacts` 用 `Map<String, Object>` 灵活但无类型约束，容易在跨版本时"字段悄悄变空"。

**生产建议**：对稳定产物定义专用 record；`Map` 只用于实验性字段。

### 2.1.2 消息类型枚举

```java
public enum MessageType {
    REQUEST,      // 请求：期望得到响应
    RESPONSE,     // 响应：对某个请求的答复
    NOTIFY,       // 通知：无需响应
    HANDOFF       // 交接：把控制权转移给下一个 Agent
}
```

**边界与坑**：把"交接"混用为"通知"，会导致接收方不知道该不该继续推进，流程静默中断。

**生产建议**：每种类型配一个处理分支，未识别类型直接拒绝而非忽略。

## 2.2 同步与异步发送

**结论先行**：默认同步；当扇出或延迟不可接受时改异步。

### 2.2.1 同步请求-响应

```java
public AgentMessage call(AgentMessage request, Duration timeout) {
    Agent target = registry.get(request.to());
    CompletableFuture<AgentMessage> future =
            CompletableFuture.supplyAsync(() -> target.handle(request));
    try {
        return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        future.cancel(true);
        throw new AgentTimeoutException(request.to(), timeout);
    } catch (Exception e) {
        throw new AgentInvocationException(request.to(), e);
    }
}
```

### 2.2.2 异步发送

```java
public CompletableFuture<AgentMessage> sendAsync(AgentMessage request) {
    return CompletableFuture
            .supplyAsync(() -> registry.get(request.to()).handle(request), executor)
            .orTimeout(5, TimeUnit.SECONDS)
            .exceptionally(ex -> AgentMessage.error(request, ex.getMessage()));
}
```

**边界与坑**：异步链路上异常不会自动冒泡，必须显式 `exceptionally`/`handle`，否则失败会被静默吞掉。

## 2.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 无超时 | 线程被拖死 | 每次调用设超时 |
| 无幂等 | 重试导致重复副作用 | 引入 messageId 去重 |
| 无关联 | 响应找不到请求 | 带 correlationId |
| 循环调用 | A→B→A 栈溢出 | 记录调用链并检测环 |
| 大消息体 | 内存/带宽爆炸 | 限制大小，大对象走引用 |

# 3. 共享状态（黑板模式）

**结论先行**：当多个 Agent 需要"逐步完善同一产物"时，黑板模式比点对点消息更自然。

**原理**：黑板是一块所有 Agent 都能读写的共享数据结构。每个 Agent 只读取自己关心的字段、写回自己的产物，最终由某个 Agent 汇总。

## 3.1 黑板机制

```java
public class SharedState {

    private final Map<String, Object> state = new ConcurrentHashMap<>();

    public synchronized void put(String key, Object value) {
        state.put(key, value);
        notifyAll();                  // 通知等待的 Agent
    }

    public Object get(String key) {
        return state.get(key);
    }
}
```

**边界与坑**：`put` 与 `get` 单独加锁，但"读-改-写"序列不是原子的，两个 Agent 可能互相覆盖。

**生产建议**：需要复合操作时用 `compute`/`merge`，或对整个序列加同一把锁。

## 3.2 并发控制

**结论先行**：黑板的核心难点不是读写，而是"谁在等谁、何时唤醒"。

### 3.2.1 悲观锁

```java
public class LockedBlackboard {

    private final ReentrantLock lock = new ReentrantLock();
    private final Map<String, Object> state = new HashMap<>();

    public void update(String key, UnaryOperator<Object> fn) {
        lock.lock();
        try {
            state.put(key, fn.apply(state.get(key)));
        } finally {
            lock.unlock();
        }
    }
}
```

### 3.2.2 条件等待

```java
public Object await(String key, Duration timeout) throws InterruptedException {
    long deadline = System.nanoTime() + timeout.toNanos();
    synchronized (this) {
        while (!state.containsKey(key)) {
            long remain = deadline - System.nanoTime();
            if (remain <= 0) {
                throw new AgentTimeoutException("等待字段超时: " + key);
            }
            TimeUnit.NANOSECONDS.timedWait(this, remain);
        }
        return state.get(key);
    }
}
```

**边界与坑**：`wait` 必须放在 `while` 循环里（防虚假唤醒）；用 `notifyAll` 而非 `notify`，否则可能唤醒错线程。

## 3.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 丢失更新 | 后写覆盖先写 | 用 compute/merge |
| 死锁 | 多把锁交叉等待 | 统一加锁顺序 |
| 状态膨胀 | 黑板字段无限增长 | 定义 Schema，及时清理 |
| 隐式耦合 | 改字段全员受影响 | 字段版本化 |
| 难以追踪 | 不知谁改了字段 | 每次写入记录来源 |

**生产建议**：黑板更适合"单进程、字段固定、Agent 数量有限"的场景；跨进程共享状态应退回消息队列或外部存储（Redis/DB）。

# 4. 消息格式与协议

**结论先行**：消息格式是 Agent 之间的"接口契约"。契约不稳，上层编排再怎么优化也白搭。

## 4.1 结构化消息

**原理**：结构化消息包含信封（envelope）与载荷（payload）两部分。信封描述路由与元数据，载荷承载业务内容。

```java
public record Envelope(
        String id,
        String from,
        String to,
        MessageType type,
        String correlationId,
        Map<String, Object> headers
) {}

public record Payload(String task, List<String> constraints, Map<String, Object> data) {}

public record ProtocolMessage(Envelope envelope, Payload payload) {}
```

**边界与坑**：把路由信息混进 payload，会导致中间件无法做基于信封的分发。

## 4.2 消息类型与语义

| 类型 | 语义 | 是否期望响应 | 典型用途 |
|------|------|--------------|----------|
| REQUEST | 请求执行 | 是 | 委派任务 |
| RESPONSE | 答复请求 | 否 | 返回结果 |
| NOTIFY | 单向通知 | 否 | 状态广播 |
| HANDOFF | 转移控制权 | 否（由下家继续） | Swarm 交接 |

**边界与坑**：语义混淆是"流程静默中断"的头号原因——发送方以为对方会回复，对方以为只是通知。

## 4.3 协议设计原则

**结论先行**：六条原则——结构化、可追溯、幂等、有界、超时、审计。

| 原则 | 说明 | 落地手段 |
|------|------|----------|
| 结构化 | 消息是结构化对象 | record + JSON Schema |
| 可追溯 | 能关联请求与响应 | correlationId |
| 幂等 | 重复消息可安全处理 | messageId 去重表 |
| 有界 | 限制消息大小 | 大小上限 + 大对象引用 |
| 超时 | 通信有超时 | Duration + 熔断 |
| 审计 | 记录通信 | 结构化日志 |

**边界与坑**：六条原则里最容易漏的是"有界"。一个 Agent 把整篇文档塞进消息，下游直接 OOM（内存溢出）。

**生产建议**：在网关层统一校验消息大小与必填字段，不合规直接拒绝。

# 5. 上下文传递

**结论先行**：多 Agent 传递的不是"全部历史"，而是"完成任务所必需的最小信息"。

**原理**：上下文越长，成本越高、注意力越稀释。传递策略本质上是在"信息完整"与"成本可控"之间取平衡。

## 5.1 三种传递策略

```
多 Agent 传递什么？
① 完整对话历史（昂贵）
② 摘要 + 关键信息（推荐）
③ 仅任务描述 + 必要数据
```

| 策略 | 成本 | 信息完整度 | 适用 |
|------|------|------------|------|
| 全量历史 | 高 | 高 | 短链路、强依赖 |
| 摘要 + 关键信息 | 中 | 中高 | 多数场景 |
| 任务 + 必要数据 | 低 | 低 | 弱耦合、独立子任务 |

**边界与坑**：摘要是有损压缩。若关键约束在摘要中被抹掉，下游 Agent 会"合理地做错事"。

## 5.2 HandoffContext 设计

```java
public record HandoffContext(
        String task,
        String summary,               // 前序工作的摘要
        Map<String, Object> data,     // 关键数据
        List<String> constraints      // 约束
) {

    public static HandoffContext of(String task, String summary) {
        return new HandoffContext(task, summary, Map.of(), List.of());
    }
}
```

**边界与坑**：`constraints` 若为空，下游可能违反安全或格式要求。约束必须显式传递，不能指望"对方应该知道"。

**生产建议**：把约束固化为"不可省略字段"，缺失时拒绝执行。

## 5.3 摘要压缩

**结论先行**：摘要应保留"目标、结论、未决问题、约束"，丢弃"寒暄、重复、中间推理"。

```java
public String summarize(List<AgentMessage> history) {
    String joined = history.stream()
            .map(m -> m.from() + " -> " + m.content())
            .collect(Collectors.joining("\n"));
    return summarizer.summarize("""
            请压缩以下多 Agent 对话，保留：
            1) 原始目标 2) 已达成的结论 3) 未决问题 4) 硬性约束
            丢弃：寒暄、重复表述、中间推理过程。
            ---
            %s
            """.formatted(joined));
}
```

**边界与坑**：用同一模型既做任务又做摘要，可能把"未决问题"误判为"已解决"。

**生产建议**：摘要用更便宜的小模型，并对关键字段做规则校验（如约束条数不能为零）。

# 6. 通信的可观测

**结论先行**：没有跨 Agent 的 Trace（链路追踪），多智能体系统几乎无法调试。

**原理**：一次用户请求会横跨多个 Agent 与多次消息传递。可观测的目标是"把这条链路还原成一棵树"。

## 6.1 Trace 结构

```
Trace: 复杂任务
├── Span: Supervisor 决策
├── Span: -> Agent A（消息）
│   └── Span: Agent A 执行
├── Span: Agent A -> Agent B（交接）
│   └── Span: Agent B 执行
└── Span: 汇总
```

| 概念 | 含义 |
|------|------|
| Trace | 一次端到端请求 |
| Span | 一个执行单元（节点/消息/调用） |
| Parent | 上一级 Span，构成树 |

**边界与坑**：只记录 LLM 调用、不记录消息传递，会丢失"谁把任务转给了谁"的关键信息。

## 6.2 correlationId 关联

```java
public AgentMessage handle(AgentMessage message, Span parent) {
    Span span = tracer.spanBuilder("agent." + message.to())
            .setParent(Context.current().with(parent))
            .setAttribute("correlationId", message.correlationId())
            .startSpan();
    try (Scope scope = span.makeCurrent()) {
        return doHandle(message);
    } finally {
        span.end();
    }
}
```

**边界与坑**：异步线程池会丢失 ThreadLocal 上下文，需显式传递 Context。

## 6.3 关键指标

| 指标 | 含义 | 告警阈值示例 |
|------|------|--------------|
| agent.calls | 单请求 Agent 调用数 | > 20 |
| msg.bytes | 消息大小 | > 256 KB |
| handoff.depth | 交接深度 | > 10 |
| cycle.count | 检测到的环 | > 0 |
| timeout.rate | 超时比例 | > 1% |

**生产建议**：把"交接深度"和"环检测"作为硬指标，超过即熔断，避免无限协作。

# 7. 生产注意事项

**结论先行**：通信问题的治理，靠"限制 + 追踪 + 兜底"三件套。

## 7.1 风险与对策

| 风险 | 建议 |
|------|------|
| 消息无限增长 | 限制大小与数量 |
| 无超时 | 通信设超时 |
| 循环通信 | 检测循环 |
| 上下文爆炸 | 传摘要 |
| 无 Trace | correlationId 关联 |
| 状态不一致 | 共享状态加锁 |
| 重复副作用 | 幂等去重 |

## 7.2 上线检查清单

- 每条消息都有唯一 id 与 correlationId。
- 所有跨 Agent 调用都有超时。
- 消费端对消息幂等。
- 消息大小有上限。
- 交接深度与环检测有硬上限。
- 跨 Agent 链路有统一 Trace。
- 黑板字段有 Schema 与版本。

**边界与坑**：清单最容易在"赶进度"时被跳过，而通信问题往往在生产压力下才暴露。

# 8. 小结

- 通信方式：直接消息、共享状态、消息队列、事件总线。
- 选择依据：耦合度、同步性、扇出规模。
- 黑板模式适合"多个 Agent 逐步完善同一产物"。
- 消息要结构化、可追溯、幂等、有界、超时、审计。
- **传摘要而非全量历史**，避免上下文爆炸。
- 跨 Agent 通信必须可追踪，交接深度与环要有硬上限。
