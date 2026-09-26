# 1. 为什么需要分布式

Agent 运行时的负载天然具有"长耗时、突发、不可预测"的特点：一次任务可能调用数十次模型与工具，单次耗时从几百毫秒到几分钟不等。单机进程很快会撞到天花板。

## 1.1 单机的边界

```
单机：并发受限、无法水平扩展、故障即中断
分布式：多实例并行、弹性伸缩、故障转移
```

单机部署的主要瓶颈：

| 瓶颈 | 表现 | 后果 |
|------|------|------|
| 线程/连接数 | 模型调用阻塞线程 | 并发上不去 |
| 内存 | 会话与状态驻留进程 | OOM |
| CPU | 序列化、工具本地计算 | 延迟抖动 |
| 单点 | 进程崩溃 | 任务全丢 |
| 无法扩容 | 只能垂直升级 | 成本陡增 |

## 1.2 分布式的收益与代价

### 1.2.1 收益

- **水平扩展**：Worker 无状态化后可按队列积压线性扩容。
- **故障转移**：单实例崩溃不影响整体，任务可被其他 Worker 接手。
- **资源隔离**：重任务与轻任务可分池，避免相互拖垮。
- **就近部署**：按区域部署，降低模型调用网络延迟。

### 1.2.2 代价

代价：**任务分发、状态一致性、幂等、去重**变得复杂。

| 新增复杂度 | 说明 |
|------------|------|
| 任务分发 | 需要队列与路由策略 |
| 状态一致性 | 多写入者并发覆盖 |
| 幂等 | 消息至少一次投递导致重复 |
| 去重 | 需要跨实例的共享去重存储 |
| 可观测 | 需要分布式追踪串联 |

## 1.3 何时不该上分布式

分布式不是默认选项。以下场景优先单机 + 多线程：

- 任务量小（QPS < 10），单机余量充足。
- 状态强一致且无法容忍最终一致（如强事务型 Agent）。
- 团队尚无分布式运维能力，引入队列反而增加事故面。

# 2. 整体架构

## 2.1 组件分层

```
                 ┌──────────────┐
  请求 ─────────▶│  API 网关     │
                 └──────┬───────┘
                        ▼
                 ┌──────────────┐
                 │ 任务分发器    │  写入队列
                 └──────┬───────┘
                        ▼
                 ┌──────────────┐
                 │ 消息队列      │  Kafka / RocketMQ / RabbitMQ
                 └──┬───┬───┬───┘
                    ▼   ▼   ▼
                 ┌────┐┌────┐┌────┐
                 │Worker││Worker││Worker│  消费任务
                 └──┬─┘└──┬─┘└──┬─┘
                    └─────┼─────┘
                          ▼
                   ┌────────────┐
                   │ 共享状态存储 │  Redis / DB
                   └────────────┘
```

各层职责：

| 层 | 职责 | 典型组件 |
|----|------|----------|
| 网关 | 鉴权、限流、路由 | Spring Cloud Gateway |
| 分发器 | 生成 taskId、写队列、写初始状态 | 应用服务 |
| 队列 | 缓冲、削峰、解耦 | Kafka / RocketMQ |
| Worker | 消费任务、执行 Agent | 应用服务 |
| 状态存储 | 保存任务状态、去重标记 | Redis + PostgreSQL |

## 2.2 数据流

```
submit → 状态 CREATED → 入队
        ↓
Worker 消费 → 状态 RUNNING → 执行
        ↓
成功 → 状态 COMPLETED
失败 → 状态 FAILED → 重试/死信
```

## 2.3 关键设计决策

- **状态外置**：Worker 必须无状态，所有状态写共享存储，才能随意扩缩容。
- **消息只承载指令**：大对象放对象存储，消息里只放引用，避免队列膨胀。
- **分区键即任务 ID**：保证同一任务串行、有序。
- **幂等贯穿全程**：从消费到副作用都要可重入。

# 3. 任务分发

## 3.1 分发器接口

```java
public interface TaskDispatcher {
    String submit(AgentTask task);           // 返回 taskId
    AgentState query(String taskId);
    void cancel(String taskId);
}
```

接口设计要点：

- `submit` 必须同步返回 `taskId`，让调用方立即拿到句柄。
- `query` 读状态存储而非队列，保证幂等可查。
- `cancel` 需同时写"取消标记"，Worker 在执行中轮询该标记。

## 3.2 基于 MQ 的实现

```java
@Service
@RequiredArgsConstructor
public class MqTaskDispatcher implements TaskDispatcher {

    private final KafkaTemplate<String, AgentTask> kafka;
    private final AgentStateStore stateStore;

    @Override
    public String submit(AgentTask task) {
        String taskId = UUID.randomUUID().toString();

        stateStore.save(taskId, AgentState.created(taskId, task));

        // 用 taskId 作为 key，保证同一任务路由到同一分区
        kafka.send("agent-tasks", taskId,
                task.withTaskId(taskId));

        return taskId;
    }
}
```

> **用 taskId 作为分区键**，可保证同一任务的消息有序。

### 3.2.1 分区键选择

| 分区键 | 有序性 | 负载均衡 | 适用 |
|--------|--------|----------|------|
| taskId | 单任务内有序 | 好 | 大多数场景 |
| sessionId | 单会话内有序 | 中（长会话倾斜） | 会话强序 |
| 随机 | 无序 | 最好 | 完全独立任务 |

### 3.2.2 任务元数据

```java
public record AgentTask(
        String taskId,
        String agentId,
        String agentVersion,
        String tenantId,
        String inputRef,          // 大输入放对象存储
        Map<String, String> params,
        int attempt,
        Instant submittedAt
) {
    public AgentTask withTaskId(String id) {
        return new AgentTask(id, agentId, agentVersion, tenantId,
                inputRef, params, attempt, submittedAt);
    }
}
```

## 3.3 分发策略对比

| 策略 | 说明 | 缺点 |
|------|------|------|
| 直接投递 | 网关直接写队列 | 缺乏状态记录 |
| 状态先行 | 先写状态再入队 | 两次写，需补偿 |
| 事务消息 | 本地事务 + 消息 | 实现复杂 |
| Outbox 表 | 本地表 + 投递器 | 有投递延迟 |

# 4. 任务消费

## 4.1 消费者实现

```java
@Component
@RequiredArgsConstructor
public class AgentTaskConsumer {

    private final AgentRunner runner;
    private final AgentStateStore stateStore;

    @KafkaListener(topics = "agent-tasks", concurrency = "4")
    public void onMessage(ConsumerRecord<String, AgentTask> record) {
        AgentTask task = record.value();

        // 幂等：检查是否已处理
        AgentState state = stateStore.load(task.taskId());
        if (state.status() == AgentStatus.COMPLETED) {
            log.info("任务已完成，跳过: {}", task.taskId());
            return;
        }

        try {
            runner.run(task);
        } catch (Exception e) {
            log.error("任务执行失败: {}", task.taskId(), e);
            // 不抛异常，避免无限重投；由状态机决定是否重试
        }
    }
}
```

## 4.2 幂等前置检查

消费端第一件事永远是查状态，而不是直接执行。检查逻辑：

```
load(taskId)
  ├─ COMPLETED → 直接 ACK，跳过
  ├─ RUNNING   → 可能重复投递，检查心跳
  ├─ FAILED    → 判断 attempt < maxRetry
  └─ 不存在    → 非法消息，投死信
```

## 4.3 消费位移与提交策略

| 提交策略 | 语义 | 风险 |
|----------|------|------|
| 自动提交 | 至少一次 | 重复处理 |
| 手动同步提交 | 至少一次，可控 | 吞吐略降 |
| 事务提交 | 精确一次（Kafka 内） | 复杂，仅限 Kafka |

> 与外部副作用（发邮件、下单）联动时，Kafka 事务无法覆盖，仍需业务幂等。

# 5. 幂等与去重

## 5.1 为什么需要幂等

分布式环境下，消息可能重复投递。重复来源包括：

- 生产者重试；
- 消费者 rebalance 后重放；
- 故障转移重新入队；
- 人为补数。

## 5.2 Redis SETNX 去重

```java
public class IdempotentConsumer {

    private final RedisTemplate<String, String> redis;

    public boolean tryProcess(String messageId, Runnable action) {
        Boolean first = redis.opsForValue()
                .setIfAbsent("consumed:" + messageId, "1",
                        Duration.ofHours(24));

        if (!Boolean.TRUE.equals(first)) {
            return false;                      // 已处理
        }
        action.run();
        return true;
    }
}
```

上述写法存在一个漏洞：`action.run()` 失败时，去重键已写入，重试会被误判为"已处理"。更严谨的写法是"处理成功后再标记"，或用三态键：

```java
public boolean tryProcess(String messageId, Runnable action) {
    String key = "consumed:" + messageId;

    // 先抢占
    Boolean acquired = redis.opsForValue()
            .setIfAbsent(key, "PROCESSING", Duration.ofHours(24));
    if (!Boolean.TRUE.equals(acquired)) {
        String v = redis.opsForValue().get(key);
        return "DONE".equals(v);            // 已完成才算重复
    }

    try {
        action.run();
        redis.opsForValue().set(key, "DONE", Duration.ofHours(24));
        return true;
    } catch (Exception e) {
        redis.delete(key);                  // 失败释放，允许重试
        throw e;
    }
}
```

## 5.3 去重方案对比

| 去重方案 | 说明 |
|----------|------|
| Redis SETNX | 简单高效，需 TTL |
| 数据库唯一约束 | 可靠，有性能开销 |
| 状态机检查 | 结合业务状态判断 |

| 方案 | 一致性 | 性能 | 持久性 |
|------|--------|------|--------|
| Redis SETNX | 最终 | 高 | 弱（需 AOF/RDB） |
| DB 唯一约束 | 强 | 中 | 强 |
| 状态机检查 | 强（业务） | 中 | 强 |
| 布隆过滤器 | 概率 | 极高 | 弱 |

# 6. 状态一致性

## 6.1 并发写冲突

```
Worker A 和 Worker B 同时处理同一任务 → 状态互相覆盖
```

冲突的根因是"同一任务被多个 Worker 并行处理"。要根治，需从"分发独占"和"写入保护"两端下手。

## 6.2 分区键方案

分区键保证同一任务只会被消费者组内一个消费者处理，从源头避免并行。

```
taskId 哈希 → 固定分区 → 组内唯一消费者 → 天然串行
```

## 6.3 乐观锁与分布式锁

解决：

```java
// 方案一：分区键保证同一任务只被一个消费者处理
kafka.send(topic, taskId, task);   // taskId 为 key

// 方案二：乐观锁
int updated = jdbc.update("""
        UPDATE agent_state SET state_json = ?, version = version + 1
        WHERE task_id = ? AND version = ?
        """, json, taskId, version);
if (updated == 0) throw new OptimisticLockException();

// 方案三：分布式锁
redisson.getLock("task:" + taskId).tryLock(...);
```

### 6.3.1 乐观锁

- 适合冲突概率低的场景。
- 冲突时抛异常并重读重试。
- 需注意 ABA 问题（用单调递增 version 而非时间戳）。

### 6.3.2 分布式锁

- 适合冲突概率高、临界区长的场景。
- 必须设置租约时间，避免死锁。
- 需考虑锁续期（watchdog）与 Redis 主从切换丢锁。

```java
RLock lock = redisson.getLock("task:" + taskId);
boolean locked = lock.tryLock(3, 30, TimeUnit.SECONDS);
if (!locked) {
    throw new IllegalStateException("获取锁失败: " + taskId);
}
try {
    doProcess(taskId);
} finally {
    lock.unlock();
}
```

## 6.4 一致性方案对比

| 方案 | 冲突容忍 | 实现成本 | 适用 |
|------|----------|----------|------|
| 分区键 | 低冲突 | 低 | 默认首选 |
| 乐观锁 | 低冲突 | 中 | 状态写频繁 |
| 分布式锁 | 高冲突 | 高 | 临界区保护 |
| 单写入者 | 无冲突 | 中 | 状态机串行化 |

# 7. 弹性伸缩

## 7.1 HPA 配置

```yaml
# K8s HPA：按队列积压或 CPU 扩缩容
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-worker
spec:
  scaleTargetRef:
    kind: Deployment
    name: agent-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumergroup_lag
        target:
          type: AverageValue
          averageValue: "100"
```

## 7.2 扩容指标选择

| 指标 | 优点 | 缺点 |
|------|------|------|
| CPU | 通用 | Agent 多为 IO 等待，不敏感 |
| 队列积压 lag | 直接反映负载 | 需 External Metrics |
| 请求并发数 | 直观 | 不代表真实耗时 |
| 端到端延迟 | 面向 SLO | 反馈滞后 |

> Agent Worker 建议以 **队列积压** 为主指标，CPU 为辅。

## 7.3 缩容与冷启动

- **缩容要慢**：长任务正在执行，立即缩容会触发 rebalance 与重放。
- **设置 stabilizationWindow**：避免抖动。
- **优雅停机**：收到 SIGTERM 后停止拉取、等待在途任务、提交位移。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 120
```

# 8. 故障转移

## 8.1 故障场景

| 场景 | 表现 | 影响 |
|------|------|------|
| Worker 崩溃 | 进程退出 | 在途任务丢失 |
| Worker 卡死 | 无响应 | 分区被占用 |
| 网络分区 | 消费者失联 | rebalance |
| 队列节点故障 | 分区不可用 | 局部不可用 |

## 8.2 故障转移流程

```
Worker 崩溃
   │
   ▼
Kafka 消费者组 rebalance
   │
   ▼
未提交的 offset 重新投递
   │
   ▼
其他 Worker 接手（依赖幂等）
```

## 8.3 幂等与故障转移的关系

> **故障转移的正确性依赖幂等**：重复处理不能产生重复副作用。

要点：

- 状态写入要幂等（UPSERT 而非 INSERT）。
- 副作用要带幂等键（如外部 API 的 `Idempotency-Key`）。

# 9. 长任务与心跳

## 9.1 心跳机制

长任务需要心跳防止被判定为死任务：

```java
@Scheduled(fixedDelay = 10_000)
public void heartbeat() {
    for (String taskId : runningTasks) {
        stateStore.updateHeartbeat(taskId, Instant.now());
    }
}
```

## 9.2 死任务检测与重入队

```java
// 扫描无心跳的任务
@Scheduled(fixedDelay = 60_000)
public void detectDeadTasks() {
    List<AgentState> dead = stateStore.findByHeartbeatBefore(
            Instant.now().minus(Duration.ofMinutes(2)));

    for (AgentState state : dead) {
        log.warn("任务心跳丢失，重新入队: {}", state.taskId());
        dispatcher.submit(state.toTask());
    }
}
```

> 重入队前务必校验 `attempt` 上限，否则死任务会被无限重投。

## 9.3 心跳参数调优

| 参数 | 建议值 | 说明 |
|------|--------|------|
| 心跳间隔 | 10s | 远小于超时阈值 |
| 判定超时 | 3× 心跳间隔 | 容忍网络抖动 |
| 最大重试 | 3–5 次 | 超过则进死信 |

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 重复投递 | 幂等 + 去重表 |
| 状态覆盖 | 分区键 / 乐观锁 |
| 无心跳 | 长任务加心跳 |
| 无限重投 | 最大重试次数 + 死信队列 |
| 队列积压 | HPA 自动扩容 |
| 无监控 | 监控积压、消费延迟 |

## 10.2 监控指标

| 指标 | 含义 | 告警阈值示例 |
|------|------|--------------|
| consumergroup_lag | 队列积压 | > 1000 持续 5min |
| task_duration_p99 | 任务耗时 | > 2× 基线 |
| retry_rate | 重试率 | > 10% |
| dead_letter_count | 死信数 | > 0 |
| worker_heartbeat_miss | 心跳丢失 | > 0 |

# 11. 小结

## 11.1 核心要点

- 分布式执行解决扩展与可用性，但引入一致性复杂度。
- 用 taskId 作分区键保证任务有序与独占。
- 幂等是故障转移正确性的前提。
- 长任务需心跳，防止被误判为死任务。

## 11.2 落地建议

- 用队列积压驱动自动扩缩容。
- 状态必须外置，Worker 保持无状态才能自由伸缩。
- 高价值副作用需 DB 唯一约束与业务状态机双保险。
