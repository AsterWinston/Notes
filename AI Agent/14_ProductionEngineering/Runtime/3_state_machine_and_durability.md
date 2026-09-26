# 1. 为什么用状态机

**结论先行**：Agent 的流转如果只用 if-else，很快会失控。显式状态机（Explicit State Machine）把"当前处于什么状态、能到哪里去"变成数据，而不是散落在代码里的分支。

## 1.1 显式状态机的价值

显式状态机带来：

- 流转规则可见、可测试；
- 非法流转被拒绝；
- 与工作流引擎、持久化天然契合；
- 便于监控每个状态的分布与耗时。

把这四点拆开看：

| 价值 | 具体收益 |
|------|----------|
| 可见 | 一张流转表即可读懂全部路径 |
| 可测试 | 状态 × 事件 的矩阵可穷举 |
| 可拒绝 | 非法跃迁抛出异常而非静默执行 |
| 可监控 | 按状态打点，定位卡点 |

## 1.2 状态机 vs 隐式流程

```
隐式（if-else）：
if (done) return;
else if (waiting) { ... }
else if (paused) { ... }
// 分支随需求增长而爆炸，难以回答"当前到底在哪一步"

显式（状态机）：
state = store.load(taskId);
assertTransition(state, target);
state = apply(state, target);
```

**边界与坑**：状态机不是银弹。对于只有两三个状态、没有持久化需求的短流程，引入完整状态机会增加复杂度；此时普通的枚举 + switch 就足够。

**生产建议**：当出现"等待外部事件（工具、人工）""需要恢复""需要审计"三者之一时，就应引入显式状态机。

用一个反面例子说明隐式流程为何危险：当需求追加"暂停后恢复"时，if-else 需要新增分支并保证它与所有已有分支的组合正确，而状态机只需在流转表中加一条边。

```java
// 隐式：新增"暂停"后，每个分支都要考虑 paused 情况
if (paused) { ... }
else if (executing && !paused) { ... }
// 组合爆炸，且难以证明覆盖完整

// 显式：只改一处数据
PAUSED -> Set.of(EXECUTING, CANCELLED)
```

# 2. 状态定义

**结论先行**：状态应覆盖 Agent 的全部生命周期，且每个状态都有明确的进入条件与退出条件。状态名要能自解释。

## 2.1 状态枚举设计

```java
public enum AgentStatus {
    CREATED,          // 已创建
    PLANNING,         // 规划中
    EXECUTING,        // 执行中
    WAITING_TOOL,     // 等待工具返回
    WAITING_HUMAN,    // 等待人工审批
    PAUSED,           // 人工暂停
    COMPLETED,        // 已完成
    FAILED,           // 已失败
    CANCELLED         // 已取消
}
```

## 2.2 状态与业务语义

| 状态 | 含义 | 终态 | 可恢复 |
|------|------|------|--------|
| CREATED | 已创建未开始 | 否 | 是 |
| PLANNING | 正在规划 | 否 | 是 |
| EXECUTING | 正在执行 | 否 | 是 |
| WAITING_TOOL | 等待工具返回 | 否 | 是 |
| WAITING_HUMAN | 等待人工审批 | 否 | 是 |
| PAUSED | 人工暂停 | 否 | 是 |
| COMPLETED | 成功结束 | 是 | 否 |
| FAILED | 失败结束 | 是 | 否 |
| CANCELLED | 用户取消 | 是 | 否 |

### 2.2.1 终态与可恢复

终态（terminal state）一旦进入不可再流转；非终态在崩溃后应能被恢复扫描发现。区分二者是恢复逻辑的前提。

### 2.2.2 等待态的意义

`WAITING_TOOL` 与 `WAITING_HUMAN` 都是"挂起等待外部输入"的状态，它们的区别在于触发方：工具是系统内部，人工是外部审批。分开定义便于设置不同的超时与告警策略。

**边界与坑**：状态过多会让流转表爆炸；状态过少（如把"等待工具"和"等待人工"合并为 WAITING）会丢失告警与超时的区分能力。

**生产建议**：状态数量控制在 10 个以内；用"状态 + 子原因（reason）"扩展语义，而不是新增状态。

状态对象本身建议用不可变记录类，携带版本与时间戳：

```java
public record AgentState(
        String taskId,
        AgentStatus status,
        int step,
        String checkpoint,
        String reason,
        long version,
        Instant updatedAt
) {
    public AgentState withStatus(AgentStatus next) {
        return new AgentState(taskId, next, step, checkpoint,
                reason, version + 1, Instant.now());
    }

    public boolean isTerminal() {
        return status == AgentStatus.COMPLETED
                || status == AgentStatus.FAILED
                || status == AgentStatus.CANCELLED;
    }
}
```

# 3. 合法流转

**结论先行**：把所有合法流转显式列成表，任何不在表中的跃迁都视为非法并抛出异常。这张表是状态机的单一事实来源。

## 3.1 流转表

```
CREATED ──▶ PLANNING ──▶ EXECUTING ──▶ COMPLETED
                            │  ▲
                            │  └── WAITING_TOOL
                            ├──▶ WAITING_HUMAN ──▶ EXECUTING
                            ├──▶ PAUSED ──▶ EXECUTING
                            └──▶ FAILED

任意状态 ──▶ CANCELLED（用户取消）
```

```java
public final class AgentTransitions {

    private static final Map<AgentStatus, Set<AgentStatus>> ALLOWED = Map.of(
            AgentStatus.CREATED,        Set.of(AgentStatus.PLANNING,
                                               AgentStatus.CANCELLED),
            AgentStatus.PLANNING,       Set.of(AgentStatus.EXECUTING,
                                               AgentStatus.FAILED),
            AgentStatus.EXECUTING,      Set.of(AgentStatus.WAITING_TOOL,
                                               AgentStatus.WAITING_HUMAN,
                                               AgentStatus.PAUSED,
                                               AgentStatus.COMPLETED,
                                               AgentStatus.FAILED),
            AgentStatus.WAITING_TOOL,   Set.of(AgentStatus.EXECUTING,
                                               AgentStatus.FAILED),
            AgentStatus.WAITING_HUMAN,  Set.of(AgentStatus.EXECUTING,
                                               AgentStatus.CANCELLED),
            AgentStatus.PAUSED,         Set.of(AgentStatus.EXECUTING,
                                               AgentStatus.CANCELLED)
    );

    public static void assertTransition(AgentStatus from, AgentStatus to) {
        Set<AgentStatus> allowed = ALLOWED.getOrDefault(from, Set.of());
        if (!allowed.contains(to)) {
            throw new IllegalStateException(
                    "非法状态流转: %s → %s".formatted(from, to));
        }
    }
}
```

## 3.2 非法流转拒绝

### 3.2.1 幂等流转

若 `from == to`，通常视为幂等操作直接返回，而不是报错。但 `COMPLETED → COMPLETED` 这类终态重复提交应被识别为重复请求。

```java
public static void assertTransition(AgentStatus from, AgentStatus to) {
    if (from == to && !from.isTerminal()) {
        return;                 // 幂等，允许
    }
    // ... 查表校验
}
```

### 3.2.2 取消的优先级

"任意状态可取消"是一条特殊规则。实现时需注意：终态不可再取消，取消本身也要幂等。

| 源状态 | 目标 | 是否合法 |
|--------|------|----------|
| CREATED | PLANNING | 是 |
| EXECUTING | COMPLETED | 是 |
| COMPLETED | EXECUTING | 否 |
| FAILED | CANCELLED | 否 |
| EXECUTING | CANCELLED | 是 |

**边界与坑**：用 `Map.of` 定义流转表时，未列出的源状态返回空集合，任何目标都非法——这正是我们想要的"默认拒绝"。但要注意 `Map.of` 不可变，运行时无法动态扩展。

**生产建议**：把流转表集中在一处，并为其编写单元测试，穷举所有 (from, to) 组合断言合法性。

# 4. 状态机实现

**结论先行**：状态机实现的核心是"加载 → 校验 → 变更 → 持久化"四步，且每一步都应是原子操作。无状态（stateless）实现便于水平扩展。

## 4.1 无状态状态机

```java
@Service
@RequiredArgsConstructor
public class AgentStateMachine {

    private final AgentStateStore store;

    public AgentState transition(String taskId, AgentStatus target) {
        AgentState state = store.load(taskId);
        AgentTransitions.assertTransition(state.status(), target);

        AgentState next = state.withStatus(target)
                .withUpdatedAt(Instant.now());

        store.save(taskId, next);
        return next;
    }

    public AgentState executeStep(String taskId) {
        AgentState state = store.load(taskId);

        return switch (state.status()) {
            case PLANNING    -> transition(taskId, AgentStatus.EXECUTING);
            case EXECUTING   -> doExecute(taskId, state);
            case WAITING_HUMAN, PAUSED -> state;   // 挂起，不推进
            default          -> state;
        };
    }
}
```

## 4.2 执行步骤

### 4.2.1 状态对象的不可变性

`withStatus` 返回新实例而非修改原对象，保证持久化与内存中状态的一致性，也便于做检查点快照。

### 4.2.2 乐观锁

多实例并发推进同一任务时，`save` 需带版本号做乐观锁，防止丢失更新。

```java
public void save(String taskId, AgentState next) {
    boolean ok = repository.compareAndSet(
            taskId, next.version() - 1, next);
    if (!ok) {
        throw new OptimisticLockException("状态已被其他实例修改");
    }
}
```

### 4.2.3 事务边界

"校验 + 保存"必须在同一事务内完成，否则校验通过后状态被他人改写，保存时就会覆盖。

**边界与坑**：`executeStep` 中 `doExecute` 可能耗时很长，若持有数据库事务会导致锁等待；应把长执行移出事务，仅在状态变更时开事务。

**生产建议**：状态变更走短事务，业务执行走长事务或后台任务；用乐观锁而非悲观锁，减少锁竞争。

# 5. 崩溃恢复

**结论先行**：进程崩溃后，处于非终态的任务会永久卡住。恢复机制要解决两件事：发现卡住的任务、安全地重新推进它。

## 5.1 卡住任务扫描

```
正常：EXECUTING → 保存检查点 → 继续
崩溃：进程死亡，最后状态为 EXECUTING

恢复：
1. 扫描超时的 EXECUTING 任务
2. 校验状态完整性
3. 从检查点重放（幂等）
```

```java
@Scheduled(fixedDelay = 60_000)
public void recover() {
    Instant threshold = Instant.now().minus(Duration.ofMinutes(5));
    List<AgentState> stuck = store.findByStatusUpdatedBefore(
            AgentStatus.EXECUTING, threshold);

    for (AgentState state : stuck) {
        try {
            validate(state);                       // 校验完整性
            resumeWithIdempotency(state);          // 幂等重放
        } catch (Exception e) {
            log.error("恢复失败: {}", state.taskId(), e);
            transition(state.taskId(), AgentStatus.FAILED);
        }
    }
}
```

## 5.2 幂等重放

### 5.2.1 校验完整性

`validate` 检查状态对象是否自洽：必填字段存在、引用资源存在、检查点未损坏。校验失败的任务不能盲目重放，应转人工。

### 5.2.2 重放与重复副作用

重放从最后一个检查点继续。若检查点之后已产生副作用（如已调用工具），必须靠幂等键避免重复执行。

```java
private void resumeWithIdempotency(AgentState state) {
    String key = state.taskId() + ":" + state.step();
    if (processedStore.contains(key)) {
        log.info("该步已执行，跳过重放: {}", key);
        return;
    }
    doExecute(state.taskId(), state);
}
```

| 恢复场景 | 处理 |
|----------|------|
| 检查点完整 | 从检查点重放 |
| 检查点损坏 | 标记失败，转人工 |
| 副作用已完成 | 幂等跳过 |
| 超时未完成 | 重试并记录次数 |

**边界与坑**：多实例同时扫描会重复恢复同一任务，需分布式锁或"恢复中"标记；超时阈值过小会误判慢任务为崩溃任务。

**生产建议**：恢复扫描加分布式锁；对同一任务的恢复次数设上限，超过则转失败并告警。

# 6. 持久化的三种粒度

**结论先行**：持久化粒度决定了"崩溃后损失多少进度"与"写入成本"。粒度越细越安全但越贵，需按业务权衡。

## 6.1 三种粒度对比

| 粒度 | 说明 | 适用 |
|------|------|------|
| 每步保存 | 每步一个检查点 | 默认，最安全 |
| 关键点保存 | 只在重要节点保存 | 状态大、步骤多 |
| 事件溯源 | 记录所有事件，状态由事件重建 | 需完整审计 |

| 粒度 | 恢复精度 | 写入量 | 实现复杂度 |
|------|----------|--------|------------|
| 每步 | 一步 | 高 | 低 |
| 关键点 | 一段 | 中 | 低 |
| 事件溯源 | 一步 | 高 | 高 |

## 6.2 事件溯源实现

```java
public sealed interface AgentEvent
        permits StepStarted, ToolInvoked, ToolReturned,
                StepCompleted, TaskFailed {}

public record ToolInvoked(String taskId, String tool,
                          String args, Instant at)
        implements AgentEvent {}

// 状态由事件重放得到
public AgentState rebuild(List<AgentEvent> events) {
    AgentState state = AgentState.initial();
    for (AgentEvent event : events) {
        state = apply(state, event);
    }
    return state;
}
```

优点：完整审计、可回放；缺点：实现复杂、存储量大。

### 6.2.1 快照 + 截断

事件无限增长会导致重放变慢。定期把状态快照（snapshot）落库，并截断快照之前的事件。

```
events:  e1 e2 e3 ... e100  [snapshot@100]  e101 e102 ...
rebuild: 从 snapshot@100 开始，仅重放 e101、e102
```

### 6.2.2 事件版本兼容

事件结构会演进，`apply` 需能处理旧版本事件，通常通过事件版本号与升级函数实现。

**边界与坑**：`sealed interface` 要求所有实现显式声明，新增事件类型需修改 permits 列表，容易遗漏；`apply` 若抛异常，整个重建失败。

**生产建议**：事件溯源用于审计要求高的场景；普通场景优先"每步保存 + 幂等"。

事件存储接口与快照配合：

```java
public interface EventStore {

    void append(String taskId, AgentEvent event);

    List<AgentEvent> loadSince(String taskId, long fromSeq);

    void saveSnapshot(String taskId, AgentState snapshot, long seq);
}
```

# 7. Exactly-Once 思路

**结论先行**：Agent 的"恰好一次"（Exactly-Once）通常做不到，实际是"至少一次 + 幂等"（At-Least-Once + Idempotent）。

## 7.1 至少一次加幂等

```
至少一次投递 + 幂等消费 = 效果上等价于恰好一次
```

```java
public void process(TaskMessage message) {
    String idempotencyKey = message.taskId() + ":" + message.step();

    if (processedStore.contains(idempotencyKey)) {
        log.info("已处理，跳过: {}", idempotencyKey);
        return;
    }

    doProcess(message);
    processedStore.add(idempotencyKey);
}
```

## 7.2 幂等键设计

### 7.2.1 键的构成

幂等键必须能唯一标识"一次业务操作"，常见构成是 `taskId:step` 或 `taskId:toolCallId`。键不能包含时间戳等每次都变的部分，否则失去幂等意义。

### 7.2.2 去重存储

`processedStore` 可用 Redis（带 TTL）或数据库唯一索引。Redis 快但可能丢失，数据库可靠但慢。

| 存储 | 优点 | 缺点 |
|------|------|------|
| Redis SET | 快、支持 TTL | 重启可能丢 |
| DB 唯一索引 | 可靠 | 有写入开销 |

### 7.2.3 副作用外部化

对于"调用支付"这类不可幂等的外部副作用，应使用对方提供的幂等接口（如幂等 token），把幂等责任下沉。

**边界与坑**：`doProcess` 成功但 `processedStore.add` 失败，会导致下次重复执行；因此去重记录必须与业务结果在同一事务或先写记录。

**生产建议**：先写幂等键（状态=处理中）再执行业务，成功后更新为完成；外部调用尽量使用对方幂等接口。

# 8. 状态监控

**结论先行**：状态机的可观测性体现在"每个状态有多少任务、停留多久、失败比例"。这些指标是发现卡点与容量瓶颈的关键。

## 8.1 指标

```java
// 各状态的任务数
gauge("agent.tasks", "status", AgentStatus.EXECUTING.name())
        .set(store.countByStatus(AgentStatus.EXECUTING));

// 各状态停留时长
timer("agent.state.duration", "status", status.name()).record(duration);
```

核心指标：

- 各状态任务数（分布）；
- 各状态停留时长（卡点）；
- 失败率（质量）；
- 恢复次数（稳定性）。

## 8.2 告警

告警：

| 指标 | 阈值 |
|------|------|
| WAITING_HUMAN 数量 | 突增告警 |
| FAILED 比例 | > 5% 告警 |
| EXECUTING 超时 | > 10 分钟告警 |
| 状态停留时长 | P99 超阈值 |

### 8.2.1 状态分布快照

定期把所有状态的计数打点，可绘制状态流转的"热力图"，快速发现异常堆积。

### 8.2.2 卡点定位

某状态停留时长 P99 上升，说明该状态的处理逻辑变慢，需下钻到具体步骤。

**边界与坑**：只监控终态失败率，会漏掉"大量任务卡在 WAITING_HUMAN"这类问题；标签基数过大会拖垮监控系统。

**生产建议**：监控非终态的堆积量，而非只盯失败率；标签只用 status、taskType 等低基数字段。

一个可直接使用的状态统计查询：

```java
public Map<AgentStatus, Long> distribution() {
    return repository.findAll().stream()
            .collect(Collectors.groupingBy(
                    AgentState::status, Collectors.counting()));
}
```

对 `distribution` 结果逐状态打点，即可在监控面板上看到堆积变化趋势。

# 9. 生产注意事项

## 9.1 风险与对策

| 风险 | 建议 |
|------|------|
| 非法流转 | 显式校验，拒绝非法 |
| 无检查点 | 每步保存 |
| 恢复重复副作用 | 幂等 + 已处理记录 |
| 状态不一致 | 恢复前校验 |
| 事件无限增长 | 定期快照 + 截断事件 |
| 无监控 | 监控状态分布与耗时 |

## 9.2 补充建议

- 状态对象序列化需版本兼容，升级时旧数据要能读出；
- 恢复逻辑本身要有单元测试，用故障注入验证；
- 对长期停留在等待态的任务设置超时自动处理（超时取消或提醒）。

# 10. 小结

## 10.1 核心要点

- 显式状态机让流转可控、可测、可监控。
- 非法流转必须被拒绝。
- 崩溃恢复 = 扫描卡住任务 + 校验 + 幂等重放。
- "恰好一次"通过"至少一次 + 幂等"实现。
- 事件溯源适合需要完整审计的场景。

## 10.2 落地建议

- 持久化粒度是安全性与成本的权衡，优先"每步保存"；
- 监控要看状态分布与停留时长，而不只是失败率。
