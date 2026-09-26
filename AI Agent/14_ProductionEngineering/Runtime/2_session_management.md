# 1. 会话的本质与组成

**结论先行**：会话（Session）是 Agent 与用户之间一次持续交互的容器，它把"身份、历史、状态、元数据"打包在一起，是运行时最基础的数据结构。没有会话，多轮对话就退化成彼此独立的单次请求。

## 1.1 会话的构成要素

会话不是一张表，而是一组要素的聚合。下图给出典型组成：

```
会话（Session）
├── 会话标识：sessionId
├── 归属：tenantId + userId
├── 消息历史：List<Message>
├── 摘要记忆：长期压缩结果
├── 上下文变量：用户画像、偏好
├── 状态：活跃 / 空闲 / 关闭
└── 元数据：创建时间、最后活跃时间、设备
```

### 1.1.1 会话标识

会话标识（sessionId）是会话的主键，也是跨请求、跨实例、跨存储定位会话的唯一依据。它必须满足三点：全局唯一、不可预测、可校验归属。唯一性靠 UUID 或业务组合保证；不可预测性防止攻击者枚举他人会话；可校验归属则用于访问控制。

### 1.1.2 归属与安全上下文

归属（ownership）通常由 `tenantId + userId` 两段组成。租户维度用于数据隔离（多租户 SaaS 场景），用户维度用于水平越权防护。任何加载会话的操作，都必须把当前请求的安全上下文（SecurityContext）与存储中的归属做比对。

### 1.1.3 消息历史与记忆

消息历史（Message History）是会话的正文，记录用户消息（UserMessage）、AI 消息（AiMessage）、工具消息（ToolMessage）等。当历史过长时需要压缩为摘要（Summary），摘要与历史共同构成"记忆"，而真正喂给模型的只是记忆的一个切片，即"上下文"。

用 Java 记录类（record）表达会话，天然不可变、易序列化：

```java
public record Session(
        String id,
        String tenantId,
        String userId,
        List<Message> messages,
        String summary,
        Map<String, Object> variables,
        SessionState state,
        SessionMeta meta
) {
    public Session addMessage(Message message) {
        List<Message> next = new ArrayList<>(messages);
        next.add(message);
        return new Session(id, tenantId, userId, next, summary,
                variables, state, meta.touch());
    }
}
```

不可变对象配合"替换式"更新，可以避免并发下的意外共享修改；每次更新返回新实例，便于做快照与审计。

## 1.2 会话与记忆、上下文的区别

| 概念 | 说明 | 生命周期 |
|------|------|----------|
| 会话 | 一次交互的生命周期（可能包含多轮） | 分钟到天 |
| 记忆 | 消息历史 + 摘要 + 长期偏好 | 可跨会话 |
| 上下文 | 本次请求实际带给模型的内容 | 单次请求 |

```
会话 = 容器
记忆 = 容器内的历史
上下文 = 从记忆中挑选后喂给模型的部分
```

**边界与坑**：把三者混为一谈是常见错误。例如把"上下文窗口"当成"会话存储"，会导致裁剪后历史丢失；把"记忆"当成"会话"，会导致跨会话的长期偏好无处安放。

**生产建议**：明确三层的存储与生命周期边界，会话层可回收，记忆层可持久，上下文层每次重建。

# 2. 会话 ID 设计

**结论先行**：会话 ID 的设计要在"可查询"与"防枚举"之间取平衡。随机 UUID 最安全但不可查询；业务组合可查询但需防越权；哈希方案取两者之中。

## 2.1 三种生成方案

```java
// 方案一：随机 UUID，简单且不可预测
String sessionId = UUID.randomUUID().toString();

// 方案二：租户:用户:会话 三段式，可查询可审计
String sessionId = tenantId + ":" + userId + ":" + UUID.randomUUID();

// 方案三：哈希，稳定可复现（同一设备同一天可复用）
String sessionId = sha256(tenantId + userId + deviceId + date);
```

| 方案 | 优点 | 缺点 |
|------|------|------|
| 随机 UUID | 无碰撞、不可预测 | 无法从 ID 反推归属 |
| 业务组合 | 可查询、可审计 | 需防止越权枚举 |
| 哈希 | 稳定、可复现 | 碰撞风险（极低） |

## 2.2 安全设计要点

### 2.2.1 防枚举

ID 必须包含足够的熵（entropy）。纯自增 ID 或 `user:1` 这类可猜测格式，攻击者可以遍历所有会话。推荐在业务组合中追加 UUID 或随机后缀。

### 2.2.2 长度与编码

ID 会被写入 Redis key、日志、URL。建议控制在 128 字符以内，并使用 URL 安全字符集（Base64URL 或十六进制）。避免在 ID 中出现分隔符与业务数据冲突。

```java
public final class SessionIdGenerator {

    public static String business(String tenantId, String userId) {
        String raw = "%s:%s:%s".formatted(
                tenantId, userId, UUID.randomUUID());
        if (raw.length() > 128) {
            throw new IllegalArgumentException("sessionId 过长");
        }
        return raw;
    }
}
```

**边界与坑**：哈希方案在同一秒内、同一设备重复请求会得到相同 ID，若用于"每次登录一个新会话"会导致会话复用。业务组合方案如果直接暴露在 URL，容易被日志采集泄露。

**生产建议**：对外使用随机 UUID，内部映射表记录归属；或使用业务组合但强制在每次加载时做归属校验。

# 3. 会话隔离与鉴权

**结论先行**：会话隔离是安全底线，其中水平越权（Horizontal Privilege Escalation，同一角色用户访问他人资源）是最常见的漏洞。所有会话读取路径都必须经过归属校验。

## 3.1 水平越权防护

```java
public Session loadSession(String sessionId, UserContext user) {
    Session session = sessionStore.get(sessionId);

    if (session == null) {
        throw new NotFoundException("会话不存在");
    }
    // 归属校验，防止水平越权
    if (!session.tenantId().equals(user.tenantId())
            || !session.userId().equals(user.userId())) {
        throw new AccessDeniedException("无权访问该会话");
    }
    return session;
}
```

关键点：先判空，再判归属。空会话应返回 404 而非 403，避免通过错误码差异探测会话是否存在。

## 3.2 垂直越权与租户隔离

### 3.2.1 角色校验

管理员可能需要查看他人会话用于审计，此时需要垂直维度（角色）校验，而不能简单放开归属判断。

### 3.2.2 租户边界

多租户系统中，`tenantId` 必须来自服务端签发的凭证（JWT / Session），而不是客户端请求参数。否则攻击者伪造 `tenantId` 即可跨租户访问。

| 越权类型 | 表现 | 防护 |
|----------|------|------|
| 水平越权 | 用户 A 读用户 B 会话 | 归属校验 |
| 垂直越权 | 普通用户读管理接口 | 角色校验 |
| 跨租户 | 租户 X 读租户 Y 数据 | tenantId 来自凭证 |

**边界与坑**：仅在列表接口做过滤、详情接口忘记校验，是最典型的遗漏；把 `tenantId` 作为查询参数传入是更危险的模式。

**生产建议**：把归属校验封装为统一拦截器或仓储层强制条件，避免每处手动书写导致遗漏。

```java
@Component
@RequiredArgsConstructor
public class SessionOwnershipGuard {

    private final SessionStore store;

    public Session require(String sessionId, UserContext user) {
        Session session = store.get(sessionId);
        if (session == null) {
            throw new NotFoundException("会话不存在");
        }
        if (!session.tenantId().equals(user.tenantId())) {
            // 跨租户一律拒绝，即使同用户
            throw new AccessDeniedException("租户不匹配");
        }
        if (!session.userId().equals(user.userId())
                && !user.hasRole("ADMIN")) {
            throw new AccessDeniedException("无权访问该会话");
        }
        return session;
    }
}
```

把校验点收敛到一个 `require` 方法，所有上层服务调用它即可，杜绝"某处忘了校验"。

# 4. 会话存储

**结论先行**：活跃会话读多写多、要求低延迟，用 Redis；审计与冷数据要求持久，用关系库。生产上最常见的是"热数据 Redis + 冷数据 DB"的混合方案。

## 4.1 存储选型

| 存储 | 适用 | TTL |
|------|------|-----|
| Redis | 活跃会话 | 是 |
| 关系库 | 需持久与审计 | 手动清理 |
| 混合 | 热数据 Redis + 冷数据 DB | 分层 |

选型维度：读写延迟、持久性要求、查询复杂度、成本。会话通常只需按 ID 点查，因此键值存储足够；但审计与统计需要关系库。

## 4.2 混合存储实现

```java
public class HybridSessionStore {

    private final StringRedisTemplate redis;      // 热数据
    private final SessionRepository repository;   // 冷数据

    public Session get(String sessionId) {
        // 先查 Redis
        String json = redis.opsForValue().get(key(sessionId));
        if (json != null) {
            return deserialize(json);
        }

        // 回源数据库
        Session session = repository.findById(sessionId).orElse(null);
        if (session != null) {
            redis.opsForValue().set(key(sessionId), serialize(session),
                    Duration.ofHours(2));          // 回填
        }
        return session;
    }

    public void save(Session session) {
        redis.opsForValue().set(key(session.id()), serialize(session),
                Duration.ofHours(2));
        repository.saveAsync(session);             // 异步落库
    }
}
```

### 4.2.1 缓存一致性

先写 Redis 再异步落库，存在"Redis 写成功、DB 写失败"的窗口。解决方式：使用可靠消息队列投递落库任务，或定期对账（reconciliation）修复。

### 4.2.2 缓存穿透与雪崩

不存在的会话 ID 会击穿到 DB。可用空值缓存（短 TTL）或布隆过滤器（Bloom Filter）拦截。大量会话同时过期会造成缓存雪崩，需给 TTL 加随机抖动。

```java
Duration ttl = Duration.ofHours(2)
        .plusSeconds(ThreadLocalRandom.current().nextInt(600));
```

## 4.3 序列化与版本兼容

会话结构会随版本演进（新增字段、调整枚举）。JSON 反序列化遇到未知字段应忽略，遇到缺失字段应使用默认值。推荐在存储中带上 `schemaVersion`。

```java
public record SessionEnvelope(int schemaVersion, String payload) {}
```

**边界与坑**：直接把 Java 对象用 JDK 原生序列化写入 Redis，会导致版本升级后无法反序列化；`saveAsync` 未处理异常会造成静默丢数据。

**生产建议**：统一使用 JSON（Jackson）序列化并显式版本号；落库失败必须有告警与重试。

仓储层接口建议只暴露必要操作，避免上层直接拼 SQL：

```java
public interface SessionRepository {

    Optional<Session> findById(String sessionId);

    void saveAsync(Session session);

    List<Session> findIdleBefore(Instant threshold);

    void archive(String sessionId, String summary);
}
```

| 操作 | 目标存储 | 一致性要求 |
|------|----------|------------|
| get | Redis 优先 | 最终一致 |
| save | Redis 同步 + DB 异步 | 最终一致 |
| archive | DB | 强一致 |
| delete | Redis + DB | 强一致 |

# 5. 会话生命周期

**结论先行**：会话从创建到归档是一个有限状态机，核心是"何时释放内存、何时归档、何时删除"。TTL 与定时任务共同承担回收职责。

## 5.1 状态与流转

```
创建 ──▶ 活跃 ──▶ 空闲（超时）──▶ 归档/删除
          │
          └──▶ 主动关闭
```

| 状态 | 说明 | 处理 |
|------|------|------|
| 活跃 | 近期有交互 | 正常服务 |
| 空闲 | 超过 N 分钟无交互 | 释放内存，保留存储 |
| 过期 | 超过 TTL | 归档或删除 |
| 关闭 | 用户主动结束 | 生成摘要后归档 |

## 5.2 空闲回收与归档

```java
@Scheduled(fixedDelay = 300_000)
public void closeIdleSessions() {
    List<Session> idle = sessionStore.findIdleBefore(
            Instant.now().minus(Duration.ofMinutes(30)));

    for (Session session : idle) {
        String summary = summarizer.summarize(session.messages());
        sessionStore.archive(session.id(), summary);
        sessionStore.delete(session.id());
    }
}
```

### 5.2.1 归档内容

归档不等于删除全部，通常保留：会话摘要、关键实体、最终状态、统计指标。原始消息可另存冷存储以控制成本。

### 5.2.2 关闭时的摘要

用户主动关闭时生成摘要，可作为长期记忆写入用户画像，供下次会话加载。

**边界与坑**：定时任务在多实例部署时会重复执行，需加分布式锁或让任务幂等；归档失败却删除了活跃数据会造成不可逆丢失，应先归档成功再删除。

**生产建议**：归档与删除分两步，删除前二次确认；对归档失败的任务落死信队列人工处理。

# 6. 并发控制

**结论先行**：同一会话的并发请求会导致消息乱序、覆盖写入。会话级串行化是必需的，跨会话则可并行。

## 6.1 会话级串行化

```java
public String chat(String sessionId, String message) {
    RLock lock = redisson.getLock("session:lock:" + sessionId);

    if (!lock.tryLock(3, 30, TimeUnit.SECONDS)) {
        throw new TooManyRequestsException("会话正忙，请稍后重试");
    }
    try {
        Session session = sessionStore.get(sessionId);
        session.addMessage(UserMessage.from(message));

        String reply = chatClient.prompt()
                .messages(session.messages())
                .call().content();

        session.addMessage(AiMessage.from(reply));
        sessionStore.save(session);
        return reply;
    } finally {
        lock.unlock();
    }
}
```

## 6.2 分布式锁与本地锁

### 6.2.1 锁粒度

锁粒度必须是"会话级"，不能用"用户级"（会阻塞多设备并发）或"全局级"（吞吐骤降）。锁 key 用 sessionId 即可。

### 6.2.2 锁的超时与续期

`tryLock` 的第一个参数是等待时间，第二个是持有时间。持有时间必须大于单次 LLM 调用耗时，否则锁提前释放导致并发进入。Redisson 的看门狗（watchdog）可在未显式指定租期时自动续期。

| 方案 | 适用 | 风险 |
|------|------|------|
| 本地锁（synchronized/ReentrantLock） | 单实例 | 多实例失效 |
| Redis 分布式锁 | 多实例 | 锁超时、脑裂 |
| 队列串行化 | 强顺序 | 复杂度高 |

**边界与坑**：锁超时后业务仍在执行，另一请求进入造成乱序；`unlock` 在非持有线程调用会抛异常。用队列（每会话一个单线程消费者）可彻底避免超时问题。

**生产建议**：优先用"会话级单线程队列"代替显式锁；若用锁，务必设置合理租期并捕获解锁异常。

基于队列的串行化示意：

```java
// 每个会话一个单线程执行器，天然串行
private final Map<String, ExecutorService> lanes = new ConcurrentHashMap<>();

public CompletableFuture<String> chat(String sessionId, String message) {
    ExecutorService lane = lanes.computeIfAbsent(sessionId,
            id -> Executors.newSingleThreadExecutor());
    return CompletableFuture.supplyAsync(() -> doChat(sessionId, message), lane);
}
```

该方案的代价是每会话一个线程，会话数极大时需改用"分区队列 + 固定线程池"（按 sessionId 哈希分区，同一分区串行）。

# 7. 多端同步

**结论先行**：同一用户多设备时，默认采用"每设备独立会话"，避免冲突；仅当产品明确要求多端一致时才共享会话。

## 7.1 同步策略

| 策略 | 说明 |
|------|------|
| 独立会话 | 每设备独立 sessionId（推荐） |
| 共享会话 | 多端看到同一会话，需处理冲突 |
| 主从 | 一端为主，其他只读 |

## 7.2 冲突处理

### 7.2.1 乐观锁

共享会话下，用版本号（version）做乐观锁，提交时校验版本，冲突则重试或提示。

```java
public void update(String sessionId, long expectedVersion, Session next) {
    boolean ok = repository.compareAndSet(sessionId, expectedVersion, next);
    if (!ok) {
        throw new OptimisticLockException("会话已被其他设备修改");
    }
}
```

### 7.2.2 事件广播

多端实时同步可用 WebSocket 广播会话增量事件，客户端本地重放。

**边界与坑**：共享会话在两端同时发送消息时，消息插入顺序不确定；乐观锁在高并发下重试风暴。

**生产建议**：默认独立会话，把"跨端延续"做成显式的"继续上次会话"入口，而非隐式共享。

# 8. 会话与记忆的关系

**结论先行**：会话是容器，记忆是容器内的历史，上下文是从记忆中挑选喂给模型的部分。三者层次不同，不可混淆。

## 8.1 概念区分

| 概念 | 说明 |
|------|------|
| 会话 | 一次交互的生命周期（可能包含多轮） |
| 记忆 | 消息历史 + 摘要 + 长期偏好 |
| 上下文 | 本次请求实际带给模型的内容 |

```
会话 = 容器
记忆 = 容器内的历史
上下文 = 从记忆中挑选后喂给模型的部分
```

## 8.2 上下文组装

### 8.2.1 裁剪策略

当消息总量超过上下文窗口时，按"系统提示 + 摘要 + 最近 N 轮 + 当前问题"的顺序组装。摘要承担远距离记忆，最近 N 轮保证连贯。

```java
public List<Message> buildContext(Session session, int maxTokens) {
    List<Message> ctx = new ArrayList<>();
    ctx.add(systemPrompt(session.profile()));
    ctx.add(summaryMessage(session.summary()));
    ctx.addAll(recentMessages(session.messages(), 10));
    return trimToTokenBudget(ctx, maxTokens);
}
```

### 8.2.2 记忆写回

会话关闭时，把摘要与关键事实写回用户级记忆，实现跨会话的长期记忆。

**边界与坑**：把整个历史塞进上下文会超窗；只保留摘要会丢失细节；摘要生成有信息损耗，需保留原始消息以便回溯。

**生产建议**：上下文组装与记忆写回解耦，前者每次请求执行，后者在会话归档时执行。

# 9. 会话统计与治理

**结论先行**：没有度量就没有治理。活跃会话数、消息数、时长、Token 消耗是四个核心指标。

## 9.1 指标定义

```java
public record SessionMetrics(
        long activeSessions,
        long messagesPerSession,
        Duration avgSessionDuration,
        long tokensPerSession
) {}
```

监控项包括活跃会话数、平均消息数、会话时长与每会话 Token，分别对应容量规划、记忆策略、TTL 设置与成本控制。

## 9.2 监控与告警

| 指标 | 用途 | 告警阈值 |
|------|------|----------|
| 活跃会话数 | 容量规划 | 突增 50% |
| 平均消息数 | 记忆压缩策略 | 超预期 |
| 平均会话时长 | TTL 调优 | 异常增长 |
| 每会话 Token | 成本控制 | 超预算 |

### 9.2.1 埋点位置

在会话创建、消息追加、会话关闭三个位置埋点，即可推导出全部指标。

### 9.2.2 异常检测

会话数骤降可能意味着登录故障；Token 均值骤升可能意味着提示词被注入。

**边界与坑**：只统计平均值会掩盖长尾；高基数标签（如按 sessionId 打点）会撑爆时序库。

**生产建议**：使用聚合指标而非明细指标；对成本类指标做按租户维度的分位数统计。

# 10. 生产注意事项

## 10.1 风险与对策

| 风险 | 建议 |
|------|------|
| 越权访问 | 强制校验归属 |
| 会话无限增长 | TTL + 归档 |
| 并发乱序 | 会话级锁 |
| 内存泄漏 | 空闲会话及时释放 |
| 敏感信息落库 | 脱敏/加密 |
| 多实例不一致 | 共享存储 |
| 无监控 | 监控活跃数与异常 |

## 10.2 补充建议

- 敏感信息（身份证、手机号）在写入会话前脱敏，或整体加密存储；
- 会话 ID 不得出现在对外 URL 查询串中，防止被日志与浏览器历史泄露；
- 灰度发布时注意序列化格式兼容，新旧实例可能同时读写同一会话。

# 11. 小结

## 11.1 核心要点

- 会话 = 标识 + 归属 + 历史 + 状态 + 元数据。
- 会话 ID 设计要兼顾可查询与防枚举。
- **归属校验是防水平越权的关键**。
- 热数据 Redis + 冷数据 DB 的混合方案最实用。
- 会话级锁保证消息顺序，空闲会话及时归档。
- 会话、记忆、上下文分属不同层次，需分别治理。

## 11.2 落地建议

- 先用指标驱动 TTL、容量与成本的持续调优；
- 会话生命周期尽早用状态机固化，避免散落的分支判断。
