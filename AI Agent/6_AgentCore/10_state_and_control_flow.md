# 1. 为什么状态重要

结论先行：Agent 是**多步、有状态**的，状态必须外置持久化，否则无法恢复、扩展、调试与审计。

## 1.1 无状态与有状态的差异

### 1.1.1 没有状态管理的四个后果

没有状态管理，就无法：

- 中断后恢复（用户离开、服务重启）；
- 多实例部署（请求可能落到不同节点）；
- 调试（无法回放执行过程）；
- 审计（无法追溯每一步）。

```
无状态 Agent：所有状态在内存 → 重启丢失、无法扩展
有状态 Agent：状态外置持久化 → 可恢复、可扩展、可审计
```

### 1.1.2 对比表

| 维度 | 无状态 | 有状态（外置） |
|------|--------|----------------|
| 崩溃恢复 | 丢失全部进度 | 从检查点恢复 |
| 水平扩展 | 无法路由 | 可路由到任意实例 |
| 调试 | 无法回放 | 可重放执行轨迹 |
| 审计 | 无记录 | 每步可追溯 |
| 复杂度 | 低 | 需存储与序列化 |

## 1.2 状态外置的原理

### 1.2.1 外置架构

```
┌──────────────┐      ┌──────────────────┐
│  Agent 实例  │◀────▶│  状态存储         │
│  （无状态）  │      │  Redis / DB / OSS │
└──────────────┘      └──────────────────┘
```

实例本身不持有会话状态，每次处理都从存储加载、处理、再保存，从而可以随时被替换或扩容。

### 1.2.2 边界与坑

- 序列化成本：状态越大，每次读写越慢，需控制粒度。
- 版本兼容：状态结构变更后，旧检查点可能无法反序列化，需带 schema 版本。
- 存储成为单点：状态存储故障会同时影响所有实例，需要高可用。

# 2. Agent 的状态组成

结论先行：Agent 状态由任务标识、目标、消息历史、计划、当前步、中间结果、状态枚举与时间戳构成，其中消息与计划是核心。

## 2.1 状态模型

### 2.1.1 完整 Java 示例

```java
public record AgentState(
        String taskId,                       // 任务标识
        String goal,                         // 原始目标
        List<Message> messages,              // 消息历史
        List<SubTask> plan,                  // 当前计划
        int currentStep,                     // 当前步
        Map<String, Object> scratchpad,      // 中间结果
        AgentStatus status,                  // 状态枚举
        Instant updatedAt
) {}
```

### 2.1.2 字段说明

| 字段 | 作用 |
|------|------|
| `messages` | 对话与工具调用历史 |
| `plan` | 计划（可被重规划覆盖） |
| `scratchpad` | 中间计算结果 |
| `status` | 控制流依据 |

## 2.2 状态的生命周期

### 2.2.1 阶段

| 阶段 | 状态变化 |
|------|----------|
| 创建 | 写入 goal、空 messages |
| 规划 | 写入 plan |
| 执行 | 追加 messages、更新 currentStep |
| 挂起 | status 变为 WAITING_HUMAN |
| 结束 | status 变为 COMPLETED/FAILED |

### 2.2.2 不可变更新

状态对象建议设计为不可变（Immutable），每次更新产生新对象，便于比较、回放与并发安全。

```java
public AgentState withStatus(AgentStatus next) {
    return new AgentState(taskId, goal, messages, plan,
            currentStep, scratchpad, next, Instant.now());
}
```

## 2.3 边界与坑

- 状态包含不可序列化对象：如数据库连接、线程句柄，无法持久化。
- 消息历史无限增长：需定期压缩或摘要。
- 可变共享：多线程直接修改同一状态对象会引发数据竞争。

# 3. 状态机

结论先行：用显式状态机控制流转，可以让规则可测试、非法流转可拒绝、与工作流引擎可集成。

## 3.1 为什么用显式状态机

### 3.1.1 好处

状态机的好处：

- 流转规则显式，易于测试；
- 非法流转可被拒绝；
- 便于与工作流引擎集成。

### 3.1.2 状态枚举

```java
public enum AgentStatus {
    CREATED, PLANNING, EXECUTING,
    WAITING_HUMAN, COMPLETED, FAILED, CANCELLED
}
```

## 3.2 状态流转图与规则

### 3.2.1 流转图

用显式状态机控制 Agent 流转：

```
        ┌────────┐
        │ CREATED│
        └───┬────┘
            ▼
       ┌─────────┐  需要审批   ┌──────────────┐
       │ PLANNING│────────────▶│ WAITING_HUMAN│
       └────┬────┘             └──────┬───────┘
            ▼                         │ 批准
       ┌─────────┐◀───────────────────┘
       │EXECUTING│
       └────┬────┘
       ┌────┴─────┐
       ▼          ▼
  ┌─────────┐ ┌────────┐
  │COMPLETED│ │ FAILED │
  └─────────┘ └────────┘
```

### 3.2.2 合法流转实现

```java
private static final Map<AgentStatus, Set<AgentStatus>> ALLOWED = Map.of(
    AgentStatus.CREATED,       Set.of(AgentStatus.PLANNING, AgentStatus.CANCELLED),
    AgentStatus.PLANNING,      Set.of(AgentStatus.EXECUTING, AgentStatus.WAITING_HUMAN),
    AgentStatus.EXECUTING,     Set.of(AgentStatus.WAITING_HUMAN, AgentStatus.COMPLETED,
                                      AgentStatus.FAILED, AgentStatus.CANCELLED),
    AgentStatus.WAITING_HUMAN, Set.of(AgentStatus.EXECUTING, AgentStatus.CANCELLED)
);

public AgentState transition(AgentState s, AgentStatus next) {
    if (!ALLOWED.getOrDefault(s.status(), Set.of()).contains(next)) {
        throw new IllegalStateException("非法流转：" + s.status() + " → " + next);
    }
    return s.withStatus(next);
}
```

## 3.3 边界与坑

- 状态枚举过粗：EXECUTING 内部若还细分，应显式建模而非塞进字段。
- 缺少终态保护：COMPLETED/FAILED 之后不应再流转。
- 状态机与存储脱节：应在保存前完成流转校验，避免写入非法状态。

# 4. 状态持久化

结论先行：每步保存检查点是崩溃恢复的基础；存储选型取决于任务时长、事务与体积需求。

## 4.1 每步检查点

### 4.1.1 原理

在每一步执行后立即保存完整状态快照，崩溃后从最近一次快照继续。

### 4.1.2 Java 示例

```java
public void executeStep(String taskId) {
    AgentState state = stateStore.load(taskId);

    AgentState next = step(state);        // 执行一步

    stateStore.save(taskId, next);        // 保存检查点
}
```

> **每步保存**，崩溃后可从最近检查点恢复。

## 4.2 存储选型

### 4.2.1 对比表

| 存储 | 适用 |
|------|------|
| Redis | 高频、短生命周期任务 |
| 关系型数据库 | 需要事务与审计 |
| 对象存储 | 大状态（如长消息历史） |
| 工作流引擎 | 长流程、可视化 |

### 4.2.2 接口抽象

```java
public interface AgentStateStore {
    AgentState load(String taskId);
    void save(String taskId, AgentState state);
    void delete(String taskId);
}
```

抽象存储接口后，可在开发环境用内存实现，生产环境切换为 Redis 或数据库。

## 4.3 检查点粒度

### 4.3.1 粒度权衡

| 粒度 | 优点 | 缺点 |
|------|------|------|
| 每步 | 恢复损失小 | 写入频繁、成本高 |
| 每 N 步 | 平衡 | 可能重复执行 N 步 |
| 仅关键节点 | 成本低 | 恢复损失大 |

### 4.3.2 边界与坑

- 保存与执行非原子：保存失败但执行成功会造成不一致。
- 大状态全量写：应增量或分片，否则 I/O 成为瓶颈。
- 缺少 TTL：已完成任务的状态长期占用存储。

# 5. 中断与恢复

结论先行：恢复的关键是"状态幂等可重放"——已完成的工具调用结果必须被记录，恢复时跳过。

## 5.1 恢复流程

### 5.1.1 Java 示例

```java
public String resume(String taskId) {
    AgentState state = stateStore.load(taskId);

    // 从上次中断处继续
    while (state.status() == AgentStatus.EXECUTING) {
        state = step(state);
        stateStore.save(taskId, state);
    }
    return finalResult(state);
}
```

### 5.1.2 恢复要点

恢复要点：

- 状态必须**幂等可重放**（工具调用结果已记录，不重复执行）；
- 记录"已完成的工具调用"，恢复时跳过；
- 恢复前校验状态一致性。

## 5.2 一致性校验

### 5.2.1 校验项

| 校验项 | 目的 |
|--------|------|
| 状态版本 | 防止旧结构被误读 |
| 步骤序号 | 确认从哪里继续 |
| 工具结果完整性 | 避免半完成步骤 |
| 超时与租约 | 防止僵尸任务 |

### 5.2.2 边界与坑

- 恢复时重放已执行步骤：可能产生重复副作用。
- 状态损坏静默继续：应在校验失败时转入人工处理。
- 并发恢复同一任务：需要租约或锁保护。

# 6. 幂等与去重

结论先行：恢复与重试必须能识别"已执行过"，否则写操作会重复产生副作用。

## 6.1 副作用的来源

- 崩溃恢复重放未提交的步骤；
- 用户重复提交；
- 消息队列至少一次投递。

## 6.2 幂等实现

### 6.2.1 内存去重

```java
private final Set<String> executedCalls = ConcurrentHashMap.newKeySet();

public String executeTool(ToolCall call) {
    String key = call.name() + ":" + call.arguments();

    if (!executedCalls.add(key)) {
        return cachedResult(key);          // 已执行，返回缓存
    }
    String result = doExecute(call);
    cacheResult(key, result);
    return result;
}
```

### 6.2.2 持久化幂等键

内存去重在多实例与重启后失效，生产上应使用持久化幂等键：

```java
public Result createOrder(String idemKey, String sku, int qty) {
    Optional<Result> existing = idemStore.find(idemKey);
    if (existing.isPresent()) return existing.get();

    Result r = orderService.create(sku, qty);
    idemStore.save(idemKey, r);            // 与业务写同一事务
    return r;
}
```

> 恢复/重试时，**写操作必须能识别"已执行过"**，否则会重复产生副作用。详见 `14_ProductionEngineering/Reliability/3_idempotency.md`。

## 6.3 边界与坑

- 幂等键生成不当：若每次重试都生成新键，去重失效。
- 幂等记录与业务写非同一事务：可能出现"业务成功但记录丢失"。
- 键冲突：不同请求用了相同键，会返回错误结果。

# 7. 分布式执行

结论先行：多实例部署要求服务无状态、状态外置，并通过任务路由或分布式锁保证同一任务串行执行。

## 7.1 部署架构

### 7.1.1 架构图

多实例部署时：

```
                  ┌─────────────┐
   请求 ────────▶ │  负载均衡    │
                  └──┬───┬───┬──┘
                     ▼   ▼   ▼
                  ┌────┐┌────┐┌────┐
                  │实例1││实例2││实例3│
                  └──┬─┘└──┬─┘└──┬─┘
                     └─────┼─────┘
                           ▼
                    ┌────────────┐
                    │ 共享状态存储 │  Redis / DB
                    └────────────┘
```

### 7.1.2 关键点

关键：

- 服务无状态，状态外置；
- 同一任务路由到同一实例（或加分布式锁）；
- 任务分发用消息队列，保证至少一次 + 幂等消费。

## 7.2 任务路由与锁

### 7.2.1 会话路由

```java
public int shardOf(String taskId) {
    return Math.floorMod(taskId.hashCode(), instanceCount);
}
```

### 7.2.2 分布式锁

```java
public String processWithLock(String taskId) {
    String lockKey = "agent:lock:" + taskId;
    if (!lock.tryLock(lockKey, Duration.ofSeconds(30))) {
        throw new IllegalStateException("任务正在处理中");
    }
    try {
        return resume(taskId);
    } finally {
        lock.unlock(lockKey);
    }
}
```

## 7.3 边界与坑

- 锁超时短于任务时长：会导致两个实例同时执行，需要续租。
- 队列重复投递：必须有幂等消费，否则任务被重复推进。
- 状态存储分区：跨分区的状态读取会带来一致性问题。

# 8. 控制流的常见模式

结论先行：控制流可分为线性、条件分支、循环、并行、人工闸门五种模式，选择取决于依赖关系与风险。

## 8.1 五种模式

### 8.1.1 对比表

| 模式 | 说明 |
|------|------|
| 线性 | 顺序执行计划步骤 |
| 条件分支 | 按模型/规则判断走哪条路 |
| 循环 | 重复直到满足条件（需上限） |
| 并行 | 无依赖子任务并发 |
| 人工闸门 | 关键节点等待审批 |

### 8.1.2 线性与条件分支

```java
// 线性
for (SubTask t : plan) {
    execute(t);
}

// 条件分支
if (needsRefund(state)) {
    plan = replanForRefund(state);
} else {
    plan = replanForExchange(state);
}
```

### 8.1.3 循环与并行

```java
// 循环（必须有上限）
int round = 0;
while (!satisfied(state) && round++ < MAX_ROUNDS) {
    state = step(state);
}

// 并行（无依赖子任务）
List<CompletableFuture<String>> futures = subtasks.stream()
        .map(t -> CompletableFuture.supplyAsync(() -> execute(t), pool))
        .toList();
```

## 8.2 人工闸门

### 8.2.1 与 HITL 的关系

人工闸门是控制流中等待审批的节点，是人在回路（Human-in-the-Loop）在控制流上的体现，详见 `11_human_in_the_loop.md`。

### 8.2.2 边界与坑

- 循环无上限：模型可能反复重规划，需硬性步数上限。
- 并行子任务写冲突：并发写同一状态需合并策略或串行化。
- 闸门位置不当：闸门过多会失去自动化收益。

# 9. 生产注意事项

结论先行：上线前核对"状态外置、每步检查点、幂等恢复、并发控制、状态治理、显式状态机"六件事。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 状态只在内存 | 必须外置持久化 |
| 无检查点 | 每步保存 |
| 恢复重复执行 | 幂等 + 已执行记录 |
| 并发写冲突 | 分布式锁 / 单会话串行 |
| 状态无限增长 | 压缩 + TTL |
| 无状态机 | 显式状态枚举，拒绝非法流转 |

## 9.2 部署检查表

- 状态存储具备高可用与备份。
- 检查点带 schema 版本，兼容结构变更。
- 幂等键与业务写在同一事务。
- 分布式锁带续租机制。
- 已完成任务状态设置 TTL 自动清理。

# 10. 小结

## 10.1 核心要点

- Agent 是有状态的，状态必须外置持久化才能恢复与扩展。
- 用显式状态机管理控制流。
- 每步检查点是崩溃恢复的基础。
- 恢复与重试必须幂等，避免重复副作用。
- 分布式部署依赖共享状态存储 + 任务路由。

## 10.2 落地建议

1. 先把状态模型设计成不可变、可序列化。
2. 存储接口做抽象，开发用内存、生产用 Redis/DB。
3. 检查点保存与业务执行尽量靠近原子操作。
4. 为所有写操作设计幂等键。
