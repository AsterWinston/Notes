# 1. 为什么必须持久化

**结论先行**：Agent 是多步、长时的，状态放在内存会带来致命问题；状态外置（Externalize State）是 Agent 生产化的第一道门槛。

## 1.1 内存状态的问题

### 1.1.1 六类致命问题

| 问题 | 后果 |
|------|------|
| 服务重启 | 任务丢失，用户需重来 |
| 多实例部署 | 请求落到不同实例，状态不一致 |
| 长任务 | 无法跨请求保持进度 |
| 崩溃恢复 | 无法从断点继续 |
| 审计 | 无法回溯执行过程 |
| 人工审批 | 挂起后无法恢复 |

### 1.1.2 典型故障场景

```
用户提问 → 实例A 内存中创建任务 → 实例A 重启
                                        │
用户追问 → 负载均衡到实例B → 找不到任务 → 从头再来
```

> **状态外置是 Agent 生产化的第一道门槛。**

## 1.2 状态外置的价值

### 1.2.1 可恢复

任务状态落库后，进程崩溃、发布重启都不影响任务续跑。恢复能力直接决定长任务的可用性。

### 1.2.2 可扩展

状态外置后，任何实例都能处理任何任务，支持水平扩容与滚动发布。

| 能力 | 内存状态 | 外置状态 |
|------|----------|----------|
| 崩溃恢复 | 不支持 | 支持 |
| 水平扩容 | 受限（粘性会话） | 无限制 |
| 人工审批 | 难 | 易 |
| 审计回溯 | 无 | 完整 |

## 1.3 持久化的边界

持久化不是万能的：它引入序列化、并发、版本兼容等新问题。本节后续章节会逐一讨论这些代价。

# 2. 状态组成

**结论先行**：状态模型要覆盖"目标、进度、中间结果、幂等信息、并发版本"五类内容，缺一不可。

## 2.1 状态模型

### 2.1.1 AgentState 记录

```java
public record AgentState(
        String taskId,                        // 任务 ID
        String tenantId,                      // 租户
        String userId,                        // 用户
        String goal,                          // 原始目标
        List<Message> messages,               // 消息历史
        List<SubTask> plan,                   // 当前计划
        int currentStep,                      // 当前步
        Map<String, Object> scratchpad,       // 中间结果
        Set<String> executedToolSignatures,   // 已执行工具（幂等）
        AgentStatus status,                   // 状态
        int version,                          // 乐观锁版本
        Instant createdAt,
        Instant updatedAt
) {}
```

### 2.1.2 字段说明

| 字段 | 作用 | 是否必存 |
|------|------|----------|
| taskId | 主键、幂等入口 | 是 |
| tenantId/userId | 隔离与鉴权 | 是 |
| goal | 恢复时重建上下文 | 是 |
| messages | 对话历史 | 是 |
| plan/currentStep | 断点位置 | 是 |
| scratchpad | 中间结果 | 是 |
| executedToolSignatures | 恢复幂等 | 是 |
| version | 并发控制 | 是 |

## 2.2 字段设计要点

### 2.2.1 必存字段

恢复所需的最小集合：目标、进度、已执行记录。缺少任何一个都无法安全续跑。

### 2.2.2 幂等相关字段

`executedToolSignatures` 与工具结果必须成对持久化，否则恢复时无法判断某工具是否已执行。

# 3. 检查点

**结论先行**：检查点（Checkpoint）是"每执行一步就保存一次"的状态快照，它是崩溃恢复的基础。

## 3.1 每步保存

### 3.1.1 基本流程

```java
public AgentState step(String taskId) {
    AgentState state = stateStore.load(taskId);

    AgentState next = doStep(state);       // 执行一步

    stateStore.save(taskId, next);         // 保存检查点
    return next;
}
```

### 3.1.2 检查点时机

```
Step 1 → 保存 → Step 2 → 保存 → Step 3 → 保存 → ...
                          ↑
                    崩溃后从这里恢复
```

保存应在"副作用完成之后、下一步开始之前"，保证恢复时不会丢失已产生的副作用。

## 3.2 检查点粒度

### 3.2.1 粒度选择

| 粒度 | 优点 | 缺点 |
|------|------|------|
| 每步保存 | 恢复精确 | 写放大 |
| 每 N 步保存 | 写少 | 恢复需重放 |
| 仅关键节点 | 最省 | 恢复最粗糙 |

### 3.2.2 写放大问题

每步全量写 JSON 会导致大状态写放大，可只写增量（delta）或对大字段单独存储。

# 4. 存储接口设计

**结论先行**：用接口抽象存储，屏蔽底层差异；关系库适合需要事务与审计的场景，Redis 适合高频短任务。

## 4.1 接口

### 4.1.1 AgentStateStore

```java
public interface AgentStateStore {
    AgentState load(String taskId);
    void save(String taskId, AgentState state);
    void delete(String taskId);
    List<AgentState> findByUser(String userId, AgentStatus status);
}
```

### 4.1.2 存储选型

| 存储 | 适用 | 说明 |
|------|------|------|
| Redis | 高频、短任务 | 快，需配置持久化 |
| PostgreSQL | 需事务与审计 | 可靠，可查询 |
| MongoDB | 状态文档较大 | 灵活 schema |
| 对象存储 | 超大状态 | 便宜，延迟高 |

## 4.2 关系库实现

### 4.2.1 JdbcAgentStateStore

```java
@Repository
@RequiredArgsConstructor
public class JdbcAgentStateStore implements AgentStateStore {

    private final JdbcTemplate jdbc;
    private final ObjectMapper objectMapper;

    @Override
    public AgentState load(String taskId) {
        String json = jdbc.queryForObject(
                "SELECT state_json FROM agent_state WHERE task_id = ?",
                String.class, taskId);
        return deserialize(json);
    }

    @Override
    public void save(String taskId, AgentState state) {
        String json = serialize(state);
        jdbc.update("""
                INSERT INTO agent_state (task_id, state_json, status, updated_at)
                VALUES (?, ?, ?, now())
                ON CONFLICT (task_id) DO UPDATE
                SET state_json = EXCLUDED.state_json,
                    status = EXCLUDED.status,
                    updated_at = now()
                """, taskId, json, state.status().name());
    }
}
```

### 4.2.2 建表

```sql
CREATE TABLE agent_state (
    task_id     VARCHAR(64) PRIMARY KEY,
    state_json  JSONB       NOT NULL,
    status      VARCHAR(32) NOT NULL,
    updated_at  TIMESTAMP   NOT NULL,
    created_at  TIMESTAMP   NOT NULL DEFAULT now()
);

CREATE INDEX idx_agent_state_status ON agent_state (status, updated_at);
```

## 4.3 Redis 实现

```java
public class RedisAgentStateStore implements AgentStateStore {

    private final StringRedisTemplate redis;
    private final Duration ttl = Duration.ofHours(24);

    @Override
    public void save(String taskId, AgentState state) {
        redis.opsForValue().set("agent:state:" + taskId,
                serialize(state), ttl);
    }
}
```

Redis 需开启 AOF 或 RDB，否则宕机仍会丢状态。

# 5. 崩溃恢复

**结论先行**：恢复的本质是"从最后一致检查点继续"，前提是幂等与状态完整性校验。

## 5.1 恢复流程

### 5.1.1 resume

```java
public String resume(String taskId) {
    AgentState state = stateStore.load(taskId);

    while (state.status() == AgentStatus.EXECUTING) {
        state = step(state);
    }
    return finalResult(state);
}
```

### 5.1.2 完整性校验

恢复前应校验状态完整性：计划是否完整、当前步是否越界、已执行记录与消息是否一致。校验失败应标记为损坏而非硬跑。

## 5.2 卡住任务扫描

### 5.2.1 定时扫描

```java
// 启动时扫描"卡住"的任务
@Scheduled(fixedDelay = 60_000)
public void recoverStuckTasks() {
    List<AgentState> stuck = stateStore.findByStatusAndUpdatedBefore(
            AgentStatus.EXECUTING, Instant.now().minus(Duration.ofMinutes(5)));

    for (AgentState state : stuck) {
        log.warn("恢复卡住的任务: {}", state.taskId());
        agentRunner.resumeAsync(state.taskId());
    }
}
```

### 5.2.2 超时判定

长时间未更新的 EXECUTING 任务应视为超时，标记失败或重试。判定阈值要大于单步最大耗时。

# 6. 幂等

**结论先行**：没有幂等，恢复就会重复产生副作用（重复下单、重复扣款）。

## 6.1 已执行签名

### 6.1.1 签名机制

用"工具名 + 参数"生成签名，执行前查签名表，命中则直接返回历史结果。

```java
public String executeTool(AgentState state, ToolCall call) {
    String signature = call.name() + ":" + call.arguments();

    // 已执行过 → 返回缓存结果，不重复执行
    if (state.executedToolSignatures().contains(signature)) {
        return state.toolResults().get(signature);
    }

    String result = doExecute(call);

    // 记录已执行
    state.executedToolSignatures().add(signature);
    state.toolResults().put(signature, result);

    return result;
}
```

### 6.1.2 实现要点

签名与结果必须原子落库，避免"执行了但没记录"导致恢复时重复执行。

## 6.2 幂等边界

### 6.2.1 副作用工具

对写操作（下单、扣款）必须强幂等，最好在业务侧也带幂等键，双保险。

### 6.2.2 签名冲突

参数序列化必须稳定（字段顺序固定），否则同一调用会生成不同签名，导致幂等失效。

> **没有幂等，恢复就会重复产生副作用**（重复下单、重复扣款）。详见 `Reliability/3_idempotency.md`。

# 7. 并发控制

**结论先行**：同一任务可能被多个请求或恢复线程并发更新，必须用乐观锁或分布式锁串行化。

## 7.1 乐观锁

### 7.1.1 版本号

每条状态带 version，更新时校验版本，冲突则失败重试。

### 7.1.2 实现

同一任务并发更新会覆盖：

```java
@Override
public void save(String taskId, AgentState state) {
    int updated = jdbc.update("""
            UPDATE agent_state
            SET state_json = ?, version = version + 1
            WHERE task_id = ? AND version = ?
            """, serialize(state), taskId, state.version());

    if (updated == 0) {
        throw new OptimisticLockException("状态已被其他请求更新: " + taskId);
    }
}
```

## 7.2 分布式锁

### 7.2.1 Redisson

或使用会话级分布式锁：

```java
public <T> T withTaskLock(String taskId, Supplier<T> action) {
    RLock lock = redisson.getLock("agent:task:" + taskId);
    if (!lock.tryLock(5, 30, TimeUnit.SECONDS)) {
        throw new IllegalStateException("任务正在处理中");
    }
    try {
        return action.get();
    } finally {
        lock.unlock();
    }
}
```

### 7.2.2 锁的边界

- 锁必须设租约（lease），避免持锁进程崩溃导致死锁；
- 锁粒度按 taskId，不要用全局锁；
- 加锁后仍应做乐观锁校验，防御锁失效。

# 8. 状态大小控制

**结论先行**：消息历史会无限增长，必须压缩，否则序列化、传输、存储都会成为瓶颈。

## 8.1 压缩手段

### 8.1.1 四种手段

| 手段 | 说明 |
|------|------|
| 记忆窗口 | 只保留最近 N 条 |
| 摘要压缩 | 早期消息转摘要 |
| 工具结果精简 | 只存必要字段 |
| 分表存储 | 大字段单独存 |

### 8.1.2 实现

```java
// 序列化前压缩
AgentState compacted = state.withMessages(
        memoryCompressor.compress(state.messages(), 20));
```

## 8.2 序列化与版本

### 8.2.1 schema 版本化

状态 JSON 应带 schemaVersion，便于升级时做迁移与兼容。

```json
{
  "schemaVersion": 2,
  "taskId": "t-123",
  "status": "EXECUTING"
}
```

### 8.2.2 向后兼容

新增字段应可空，删除字段需先停止读取；反序列化遇到未知字段应忽略而非报错。

# 9. 状态生命周期

**结论先行**：状态不能只增不减，必须定义状态机与保留策略，控制存储成本与合规风险。

## 9.1 状态机

### 9.1.1 状态流转

```
CREATED → EXECUTING → COMPLETED
              │  └──▶ FAILED
              ├──▶ WAITING_HUMAN → EXECUTING
              └──▶ CANCELLED
```

### 9.1.2 人工审批挂起

WAITING_HUMAN 状态让任务可长期挂起，审批后从 EXECUTING 继续，这是状态外置最重要的收益之一。

## 9.2 保留策略

### 9.2.1 分级保留

```
保留策略：
- COMPLETED：保留 7~30 天后归档
- FAILED：保留更久（用于分析）
- CANCELLED：短期保留
```

### 9.2.2 归档清理

```sql
-- 定期清理
DELETE FROM agent_state
WHERE status = 'COMPLETED' AND updated_at < now() - INTERVAL '30 days';
```

清理前应先归档到冷存储，满足审计与合规要求。

# 10. 生产注意事项

**结论先行**：状态持久化的风险集中在并发、幂等、体积、兼容四点，需配套监控与检查清单。

## 10.1 监控与告警

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| agent.state.save.latency | 保存耗时 | P99 > 500ms |
| agent.state.size | 单任务状态大小 | > 1MB |
| agent.task.stuck | 卡住任务数 | > 0 |
| agent.resume.fail | 恢复失败率 | > 1% |

## 10.2 常见反例与检查清单

反例：状态只存内存，重启后任务全丢；恢复时未查幂等，重复下单；并发更新未加锁，后写覆盖前写；状态无限增长导致 OOM。

| 风险 | 建议 |
|------|------|
| 状态在内存 | 必须外置 |
| 无检查点 | 每步保存 |
| 恢复重复执行 | 幂等 + 已执行记录 |
| 并发覆盖 | 乐观锁 / 分布式锁 |
| 状态无限增长 | 压缩 + TTL |
| 序列化不兼容 | schema 版本化 + 向后兼容 |
| 无监控 | 监控任务卡住、失败率 |

# 11. 小结

- 状态外置是 Agent 生产化的第一道门槛。
- 每步保存检查点，崩溃后可恢复。
- 恢复的前提是**幂等**：已执行工具不重复执行。
- 并发更新用乐观锁或分布式锁。
- 状态需压缩与生命周期管理。
