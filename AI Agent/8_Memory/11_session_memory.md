# 1. 会话级记忆的职责

会话级记忆（session memory）保存"一次会话内"的完整状态，
是模型理解当前对话、维持多轮连贯性的基础。

## 1.1 会话级记忆的组成

结论先行：会话级记忆不是单一的消息列表，
而是"消息历史 + 会话摘要 + 会话变量 + 会话元数据"四件套。

### 1.1.1 组成结构

```
会话级记忆 = 会话内的完整状态
├── 消息历史
├── 会话摘要
├── 会话变量（如当前订单号）
└── 会话元数据（创建时间、最后活跃）
```

### 1.1.2 组成对比

| 组成 | 内容 | 生命周期 | 用途 |
|------|------|----------|------|
| 消息历史 | 逐轮 user/assistant 消息 | 会话结束归档 | 上下文连续性 |
| 会话摘要 | 历史压缩摘要 | 随会话更新 | 长会话降本 |
| 会话变量 | 结构化槽位/状态 | 会话结束丢弃 | 多轮表单 |
| 会话元数据 | 创建/活跃时间、状态 | 随会话 | 生命周期管理 |

## 1.2 与长期记忆的区别

结论先行：会话级随会话结束而归档或过期，
长期记忆（long-term memory）跨会话持久存在。

| 维度 | 会话级记忆 | 长期记忆 |
|------|------------|----------|
| 作用域 | 单个 session | 跨 session / 用户 |
| 生命周期 | 结束即归档/过期 | 长期（可手动删除） |
| 存储 | 关系库/Redis | 向量库 + 关系库 |
| 容量 | 有限（消息数） | 大（海量条目） |
| 读取 | 全量近期 | 召回 TopK |

> 与"长期记忆"的区别：会话级随会话结束而归档或过期。

## 1.3 会话状态模型

结论先行：把会话看成一台状态机，所有读写都围绕"当前状态"展开。

```
客户端 ──▶ 会话服务 ──▶ [会话存储]
                │            │
                ├─ 消息存储 ──┤
                ├─ 摘要服务 ──┤
                └─ 变量存储 ──┘
```

# 2. 存储设计

## 2.1 表结构

结论先行：会话与消息分两张表，会话表保存状态与变量，
消息表保存逐轮明细。

### 2.1.1 建表语句

```sql
CREATE TABLE session (
    session_id    VARCHAR(128) PRIMARY KEY,
    tenant_id     VARCHAR(64)  NOT NULL,
    user_id       VARCHAR(64)  NOT NULL,
    status        VARCHAR(16)  NOT NULL,   -- ACTIVE / IDLE / CLOSED
    summary       TEXT,
    variables     JSONB,
    message_count INT          NOT NULL DEFAULT 0,
    total_tokens  INT          NOT NULL DEFAULT 0,
    created_at    TIMESTAMP    NOT NULL,
    last_active_at TIMESTAMP   NOT NULL
);

CREATE TABLE session_message (
    id          BIGSERIAL PRIMARY KEY,
    session_id  VARCHAR(128) NOT NULL,
    role        VARCHAR(32)  NOT NULL,
    content     TEXT         NOT NULL,
    metadata    JSONB,
    created_at  TIMESTAMP    NOT NULL
);

CREATE INDEX idx_session_message ON session_message (session_id, created_at);
CREATE INDEX idx_session_user ON session (tenant_id, user_id, status);
```

字段设计要点：

### 2.1.2 字段设计要点

| 字段 | 作用 | 注意 |
|------|------|------|
| status | 生命周期状态 | 用枚举字符串，可读 |
| variables | 结构化会话变量 | JSONB 便于扩展 |
| message_count | 消息计数 | 用于监控与压缩触发 |
| total_tokens | Token 累计 | 成本核算 |
| last_active_at | 最后活跃 | 超时判断依据 |

## 2.2 索引与查询

结论先行：会话查询有两类高频路径——"按会话取消息"与"按用户找会话"，
分别需要复合索引。

| 查询 | 索引 | 说明 |
|------|------|------|
| 取会话消息 | (session_id, created_at) | 时间有序 |
| 找用户会话 | (tenant_id, user_id, status) | 多端列举 |
| 找超时会话 | (status, last_active_at) | 生命周期扫描 |

## 2.3 存储选型

| 存储 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 关系库 | 事务、易查询 | 高并发写压力 | 中小规模 |
| Redis | 低延迟 | 需持久化策略 | 活跃会话 |
| Redis + 关系库 | 冷热分层 | 复杂度高 | 生产推荐 |

生产建议：活跃会话放 Redis，关闭时落关系库，形成"热缓存 + 冷归档"的分层。

# 3. 会话变量

## 3.1 变量模型

结论先行：会话变量保存会话内需要记住的结构化信息，
如当前订单号、槽位、最近意图。

会话内需要记住的结构化信息：

```java
public record SessionVariables(
        String currentOrderId,
        String currentTopic,
        Map<String, Object> slots,      // 槽位填充（多轮表单）
        String lastIntent
) {}
```

| 变量 | 说明 | 示例 |
|------|------|------|
| currentOrderId | 当前处理对象 | "ORD-2024-001" |
| currentTopic | 当前话题 | "退款流程" |
| slots | 多轮表单槽位 | {amount: 100} |
| lastIntent | 最近识别意图 | "QUERY_REFUND" |

## 3.2 读写实现

结论先行：变量更新采用"读—改—写"，合并而非覆盖，
避免丢失其他变量。

### 3.2.1 合并写入

```java
public void updateVariables(String sessionId, Map<String, Object> updates) {
    Session session = store.load(sessionId);
    Map<String, Object> vars = new HashMap<>(session.variables());
    vars.putAll(updates);
    store.updateVariables(sessionId, vars);
}
```

### 3.2.2 并发安全

```java
public boolean updateVariablesCas(String sessionId,
        Map<String, Object> updates, long expectedVersion) {
    Session s = store.load(sessionId);
    if (s.version() != expectedVersion) return false;   // 版本冲突，重试
    Map<String, Object> vars = new HashMap<>(s.variables());
    vars.putAll(updates);
    return store.updateVariablesIfVersion(sessionId, vars, expectedVersion);
}
```

## 3.3 边界与坑

- 变量值可能被模型"幻觉"填入，需校验后落库。
- JSONB 深合并只发生在顶层；变量过期需单独策略（会话未关但变量失效）。

> 槽位填充（slot filling）是多轮表单的关键：用户分多轮提供信息，逐轮填充。

# 4. 槽位填充

## 4.1 槽位填充机制

结论先行：槽位填充把"用户分多轮提供的信息"累积到结构化槽位，
填满后触发执行，未填满则追问。

```
第1轮 "我要退款"        → intent=退款, slots={}
第2轮 "订单 ORD-001"    → slots={orderId: ORD-001}
第3轮 "金额 100"        → slots={orderId, amount:100}
填满 → 执行退款
```

## 4.2 完整实现

### 4.2.1 会话处理主流程

```java
public class SlotFillingSession {

    public String handle(String sessionId, String userInput) {
        Session session = store.load(sessionId);
        Map<String, Object> slots = session.variables().get("slots");

        // 抽取本轮信息填充槽位
        slots.putAll(extractSlots(userInput));

        // 检查是否填满
        List<String> missing = requiredSlots().stream()
                .filter(s -> !slots.containsKey(s))
                .toList();

        if (!missing.isEmpty()) {
            store.save(sessionId, slots);
            return "还需要提供：" + String.join("、", missing);
        }

        // 填满 → 执行
        return execute(slots);
    }
}
```

### 4.2.2 槽位抽取

LangChain4j 风格的槽位抽取接口：

```java
interface SlotExtractor {

    @UserMessage("""
            从用户输入中抽取订单相关槽位，未提及的字段留空。
            用户输入：{{input}}
            输出 JSON：{"orderId": "", "amount": "", "reason": ""}
            """)
    SlotResult extract(String input);
}
```

## 4.3 边界与坑

| 风险 | 后果 | 对策 |
|------|------|------|
| 抽取错误 | 执行错误操作 | 执行前二次确认 |
| 槽位覆盖 | 用户改口丢失 | 显式覆盖 + 确认 |
| 永不填满 | 会话卡死 | 超时放弃/转人工 |
| 类型不符 | 执行失败 | 抽取后做类型校验 |

生产建议：涉及金额、订单等关键槽位，
执行前必须回显确认（"确认为订单 ORD-001 退款 100 元吗？"）。

# 5. 会话生命周期

## 5.1 状态机

结论先行：会话在 ACTIVE / IDLE / CLOSED 间流转，
超时驱动状态迁移，关闭时归档。

```
ACTIVE ──超时──▶ IDLE ──超时──▶ CLOSED
  ▲                │
  └──新消息────────┘
```

| 状态 | 含义 | 迁移条件 |
|------|------|----------|
| ACTIVE | 活跃中 | 5 分钟无活动 → IDLE |
| IDLE | 空闲 | 新消息 → ACTIVE；30 分钟 → CLOSED |
| CLOSED | 已关闭 | 终态（可恢复为新会话） |

## 5.2 生命周期管理实现

### 5.2.1 定时扫描

```java
@Scheduled(fixedDelay = 60_000)
public void manageLifecycle() {
    Instant now = Instant.now();

    // ACTIVE → IDLE（5 分钟无活动）
    store.findActiveBefore(now.minus(Duration.ofMinutes(5)))
            .forEach(s -> store.updateStatus(s.id(), IDLE));

    // IDLE → CLOSED（30 分钟无活动）
    store.findIdleBefore(now.minus(Duration.ofMinutes(30)))
            .forEach(this::closeSession);
}

private void closeSession(Session session) {
    // 关闭前生成摘要并归档
    String summary = summarizer.summarize(session.messages());
    archiveService.archive(session, summary);
    store.updateStatus(session.id(), CLOSED);
}
```

## 5.3 边界与坑

- 定时任务在集群下会重复执行，需分布式锁或分片；状态迁移要幂等。
- 关闭前必须归档；超时阈值按业务区分（客服会话可短，长任务会话需长）。

# 6. 会话摘要

## 6.1 摘要的作用

结论先行：会话越长，全量消息越超预算，
摘要把历史压缩为"背景"，近期消息保留原文。

| 内容 | 处理 | 原因 |
|------|------|------|
| 历史（早期） | 摘要 | 降 Token |
| 会话变量 | 结构化 | 保证精确 |
| 近期消息 | 原文 | 保证连贯 |

## 6.2 上下文组装实现

### 6.2.1 组装实现

```java
public String buildSessionContext(String sessionId) {
    Session session = store.load(sessionId);
    StringBuilder sb = new StringBuilder();

    // ① 历史摘要（如果会话很长）
    if (session.summary() != null) {
        sb.append("【会话摘要】\n").append(session.summary()).append("\n\n");
    }

    // ② 会话变量
    if (!session.variables().isEmpty()) {
        sb.append("【当前上下文】\n")
          .append(formatVariables(session.variables())).append("\n\n");
    }

    // ③ 近期消息
    List<Message> recent = messageStore.recent(sessionId, 10);
    sb.append("【近期对话】\n").append(formatMessages(recent));

    return sb.toString();
}
```

## 6.3 摘要策略

结论先行：摘要应增量更新而非每次全量重算，否则成本与延迟不可控。

```
每 N 轮 或 每 M token：
    newSummary = summarize(oldSummary + 新增消息)
    覆盖保存 summary
```

边界与坑：多次增量摘要会累积失真（"摘要的摘要"），
应周期性回看原始消息做一次全量摘要修正。

# 7. 会话恢复

## 7.1 恢复流程

结论先行：恢复会话前必须校验归属，再决定是继续还是新建。

```
请求恢复 ──▶ 加载会话 ──▶ 归属校验 ──▶ 已关闭？
                                    ├─ 是 → 询问是否继续
                                    └─ 否 → 组装上下文返回
```

## 7.2 实现

### 7.2.1 恢复实现

```java
public String resume(String sessionId, UserContext user) {
    Session session = store.load(sessionId);

    // ① 归属校验
    if (!session.belongsTo(user)) {
        throw new AccessDeniedException("无权访问该会话");
    }

    // ② 若已关闭，询问是否继续
    if (session.status() == SessionStatus.CLOSED) {
        return "该会话已结束。是否需要基于上次内容继续？";
    }

    // ③ 恢复上下文
    String context = buildSessionContext(sessionId);
    return "已恢复会话。\n" + context;
}
```

## 7.3 边界与坑

| 风险 | 后果 | 对策 |
|------|------|------|
| 未校验归属 | 越权访问 | 强制 belongsTo |
| 恢复已删除会话 | 空指针 | 存在性检查 |
| 摘要丢失 | 上下文断层 | 关闭前归档 |

# 8. 多端会话

## 8.1 两种方案

结论先行：多端场景下要么"每端独立会话"，要么"多端共享会话"，
前者简单、后者一致但复杂。

```java
// 方案一：每端独立会话（推荐）
String sessionId = tenantId + ":" + userId + ":"
        + deviceId + ":" + UUID.randomUUID();

// 方案二：共享会话（多端同步）
// 需处理并发与冲突
```

## 8.2 对比与选型

| 方案 | 优点 | 缺点 |
|------|------|------|
| 独立会话 | 简单、无冲突 | 上下文不共享 |
| 共享会话 | 上下文统一 | 需同步与冲突处理 |

选型建议：

- 客服/工单类：独立会话，避免多端互相干扰。
- 个人助手类：共享会话，用户在手机与电脑间无缝切换。
- 共享会话需版本号 + 冲突合并，且必须保证跨端事件有序。

## 8.3 共享会话的冲突合并

结论先行：共享会话下，两端可能同时追加消息，
用"追加式消息 + 版本号"避免覆盖，用时间戳/序号保证顺序。

| 冲突类型 | 表现 | 处理 |
|----------|------|------|
| 同时追加 | 消息乱序 | 服务端统一分配序号 |
| 同时改变量 | 一方被覆盖 | 版本号 CAS |
| 一端关闭 | 另一端仍在写 | 状态校验 + 重开会话 |

生产建议：共享会话以"服务端为唯一真相源"，
客户端只做乐观更新，冲突时以服务端顺序为准。

# 9. 会话并发

## 9.1 并发问题

结论先行：同一会话的并发请求会导致"读—改—写"丢失更新，
必须串行化。

```
请求A 读 messages=[m1]       请求B 读 messages=[m1]
请求A 写 [m1,a]              请求B 写 [m1,b]
结果：[m1,b]  ← A 的写入丢失
```

## 9.2 加锁实现

### 9.2.1 分布式锁实现

```java
public String chat(String sessionId, String message) {
    RLock lock = redisson.getLock("session:lock:" + sessionId);

    if (!lock.tryLock(3, 30, TimeUnit.SECONDS)) {
        throw new TooManyRequestsException("会话正忙，请稍后重试");
    }
    try {
        // 读 → 调用 → 写，保证顺序
        return doChat(sessionId, message);
    } finally {
        lock.unlock();
    }
}
```

## 9.3 边界与坑

- 锁粒度按 sessionId，不能按 user（会过度串行）。
- 必须设置租约（leaseTime）防止死锁；等待超时应返回可重试错误。
- 跨进程/跨节点必须用分布式锁，本地锁无效。

# 10. 会话监控

## 10.1 关键指标

结论先行：会话指标用于容量规划、记忆策略调优与成本核算。

```java
registry.gauge("session.active", store, s -> s.countByStatus(ACTIVE));
registry.summary("session.messages").record(messageCount);
registry.summary("session.duration.minutes").record(durationMinutes);
registry.summary("session.tokens").record(totalTokens);
```

| 指标 | 用途 |
|------|------|
| 活跃会话数 | 容量规划 |
| 平均消息数 | 记忆策略 |
| 会话时长 | TTL 设置 |
| 每会话 Token | 成本 |

## 10.2 监控实现

结论先行：指标要按租户/业务维度打标签才能定位问题；上报
`session.messages`、`session.tokens`、`session.closed` 等即可。

边界与坑：高基数标签（如 sessionId）会撑爆监控系统，标签应限制在租户、业务线等低基数维度。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 会话无限增长 | TTL + 归档 |
| 越权访问 | 归属校验 |
| 并发乱序 | 会话级锁 |
| 关闭未归档 | 关闭前摘要归档 |
| 变量丢失 | 持久化 variables |
| 无监控 | 监控活跃数与大小 |

## 11.2 生产建议

结论先行：会话服务是"有状态"的，扩展时要把状态外置到共享存储。

- 无状态化：应用节点不保存会话，状态全在 Redis/DB。
- 幂等写入：消息带客户端生成的 messageId，防重复。
- 优雅关闭：节点下线前完成进行中会话的写入。
- 数据合规：会话归档需支持按用户删除（被遗忘权）。

# 12. 小结

## 12.1 核心要点

- 会话级记忆包含消息、摘要、变量、元数据。
- 会话变量用于槽位填充等多轮场景。
- 生命周期：ACTIVE → IDLE → CLOSED，超时归档。

## 12.2 工程要点

- 会话并发用锁保证顺序，访问要校验归属。
- 会话指标用于容量规划与成本分析。
