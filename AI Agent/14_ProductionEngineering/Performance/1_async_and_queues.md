# 1. 为什么需要异步

**结论先行**：LLM（Large Language Model，大语言模型）调用慢且不可预测（1~30 s），同步处理会占满线程、拖慢响应、在突发流量下直接压垮服务。异步化的核心收益是释放线程、削峰填谷、提升吞吐。

## 1.1 同步处理的困境

### 1.1.1 线程被长时间占用

同步请求在模型返回前一直占着 Web 线程，线程池很快耗尽，其他请求只能排队。

```
- 占满 Web 线程 → 无法处理其他请求
- 用户长时间等待 → 体验差
- 突发流量直接压垮服务
```

### 1.1.2 突发流量放大问题

传统接口耗时短，排队时间可忽略；LLM 请求耗时长，少量突发就会让队列迅速积压。

| 场景 | 传统接口 | LLM 接口 |
|------|----------|----------|
| 单请求耗时 | 100 ms | 5 s |
| 200 线程容量 | ~2000 QPS | ~40 QPS |
| 突发 500 请求 | 轻微排队 | 严重积压 |

### 1.1.3 资源利用不均衡

生成阶段主要在等下游，线程处于等待而非计算，属于纯浪费。

## 1.2 异步化的三种收益

| 收益 | 机制 |
|------|------|
| 释放线程 | 等待时线程可服务其他请求 |
| 削峰填谷 | 队列缓冲突发，Worker 匀速消费 |
| 提升吞吐 | 同样的资源支撑更多并发 |

> 异步不是银弹：它把复杂度从「等待」转移到「状态管理与一致性」。

# 2. 三种处理模式

**结论先行**：按用户等待方式，LLM 服务可分为同步、流式、异步任务三种模式，选择取决于任务时长与交互需求。

## 2.1 模式对比

| 模式 | 适用 | 用户等待 |
|------|------|----------|
| 同步 | 快速响应（< 3 s） | 全程等待 |
| 流式 | 对话类 | 边生成边看 |
| 异步任务 | 长任务（> 30 s） | 提交后查询 |

## 2.2 代码示例

### 2.2.1 同步模式

```java
// 同步：简单但占用线程
@PostMapping("/chat")
public String chat(@RequestBody String q) {
    return assistant.chat(q);
}
```

### 2.2.2 流式模式

```java
// 流式：SSE，首字节快
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String q) {
    return chatClient.prompt().user(q).stream().content();
}
```

### 2.2.3 异步任务模式

```java
// 异步：立即返回 taskId
@PostMapping("/task")
public TaskRef submit(@RequestBody TaskRequest req) {
    String taskId = dispatcher.submit(req);
    return new TaskRef(taskId, "PENDING");
}
```

## 2.3 选择建议

| 任务时长 | 推荐模式 |
|----------|----------|
| < 3 s | 同步 |
| 3~30 s | 流式 |
| > 30 s | 异步任务 |

> 同一系统可组合使用：对话走流式，批处理走异步任务。

# 3. 异步任务模式

**结论先行**：异步任务把「执行」与「请求」解耦，客户端提交后立即拿到 taskId，通过轮询或通知获取结果。

## 3.1 生命周期

### 3.1.1 四步流程

```
① 客户端提交任务 → 立即返回 taskId
② 服务端异步执行（队列 + Worker）
③ 客户端轮询或接收通知
④ 获取结果
```

### 3.1.2 状态机

| 状态 | 含义 | 终态 |
|------|------|:----:|
| PENDING | 已入队未开始 | 否 |
| EXECUTING | 执行中 | 否 |
| SUCCEEDED | 成功 | 是 |
| FAILED | 失败 | 是 |
| CANCELLED | 已取消 | 是 |

## 3.2 代码实现

### 3.2.1 控制器

```java
@RestController
@RequiredArgsConstructor
public class AsyncTaskController {

    private final TaskDispatcher dispatcher;
    private final AgentStateStore stateStore;

    @PostMapping("/tasks")
    public TaskRef submit(@RequestBody TaskRequest req) {
        String taskId = dispatcher.submit(req.toAgentTask());
        return new TaskRef(taskId, "PENDING");
    }

    @GetMapping("/tasks/{taskId}")
    public TaskStatus query(@PathVariable String taskId) {
        AgentState state = stateStore.load(taskId);
        return new TaskStatus(state.taskId(), state.status().name(),
                state.result());
    }
}
```

### 3.2.2 结果通知

轮询简单但浪费；长任务建议用 Webhook（回调）或 SSE 通知，二者可并存。

## 3.3 边界与坑

- taskId 必须全局唯一且不可预测，避免越权查询。
- 结果要有 TTL，否则存储无限膨胀。
- 客户端可能重复提交，需幂等键去重。

> 生产建议：查询接口返回进度百分比，改善长任务体验。

# 4. 消息队列

**结论先行**：消息队列是异步任务的骨架，负责削峰、解耦与可靠投递，选型取决于吞吐、延迟与运维成本。

## 4.1 队列选型

| 队列 | 特点 | 适用 |
|------|------|------|
| Kafka | 高吞吐、可回溯 | 大规模任务流 |
| RocketMQ | 事务消息、延迟消息 | 国内企业常用 |
| RabbitMQ | 灵活路由、低延迟 | 中等规模 |
| Redis Stream | 轻量 | 小规模 |
| SQS / 云队列 | 免运维 | 云原生 |

## 4.2 队列的作用

### 4.2.1 削峰填谷

```
削峰填谷：
  突发 1000 请求 → 队列缓冲 → Worker 按能力消费
```

### 4.2.2 解耦

```
解耦：
  提交方与执行方解耦，各自独立扩缩容
```

### 4.2.3 可靠性

```
可靠性：
  消息持久化，Worker 崩溃后重新投递
```

## 4.3 投递语义

| 语义 | 说明 | 代价 |
|------|------|------|
| 至多一次 | 可能丢消息 | 最低 |
| 至少一次 | 可能重复 | 需幂等 |
| 恰好一次 | 不丢不重 | 最难，通常靠幂等模拟 |

> 生产建议：默认采用「至少一次 + 幂等消费」，这是工程上最务实的组合。

# 5. 削峰填谷

**结论先行**：削峰填谷让生产者快速入队、消费者按能力匀速消费，从而把瞬时峰值摊平为可持续负载。

## 5.1 提交与消费

### 5.1.1 提交侧

```java
// 提交侧：快速入队
public String submit(AgentTask task) {
    String taskId = UUID.randomUUID().toString();
    stateStore.save(taskId, AgentState.created(taskId, task));

    kafka.send("agent-tasks", taskId, task.withTaskId(taskId));
    return taskId;
}
```

### 5.1.2 消费侧

```java
// 消费侧：按能力消费
@KafkaListener(topics = "agent-tasks", concurrency = "8")
public void consume(ConsumerRecord<String, AgentTask> record) {
    runner.run(record.value());
}
```

## 5.2 效果示意

```
请求速率        消费速率
  ▲               ▲
  │    ╱╲         │ ────────  恒定消费
  │   ╱  ╲        │
  │  ╱    ╲       │
  └────────────▶  └────────────▶
    队列吸收峰值
```

## 5.3 边界与坑

- 队列不能无限缓冲，突发超出容量仍需拒绝。
- 消费并发要匹配下游配额，盲目加并发会触发限流。
- 消息堆积会推高延迟，需监控积压量与消费滞后。

> 生产建议：为队列设置「积压告警」与「消费延迟告警」双指标。

# 6. 背压（Backpressure）

**结论先行**：当消费跟不上生产时，系统必须有背压机制主动拒绝或降速，否则内存与延迟会同时失控。

## 6.1 背压策略

| 策略 | 说明 |
|------|------|
| 队列满拒绝 | 返回 429 |
| 限流 | 控制提交速率 |
| 降级 | 简化处理 |
| 动态扩容 | HPA 扩容 Worker |

## 6.2 代码实现

### 6.2.1 队列满拒绝

```java
public String submit(AgentTask task) {
    if (queueDepth() > MAX_QUEUE_DEPTH) {
        throw new TooManyRequestsException("系统繁忙，请稍后重试");
    }
    // ...
}
```

### 6.2.2 响应式背压

```java
// Flux 天然支持背压：下游请求多少，上游产出多少
Flux<String> stream = chatClient.prompt()
        .user(q)
        .stream()
        .content()
        .onBackpressureBuffer(256, BufferOverflowStrategy.DROP_OLDEST);
```

## 6.3 边界与坑

- 无界队列是内存泄漏的温床，迟早 OOM。
- 拒绝要快速失败并返回明确状态码，而非静默丢弃。
- 背压阈值需压测确定，拍脑袋设置要么过松要么误伤。

> 生产建议：背压触发时记录日志与指标，作为容量预警信号。

# 7. WebFlux 响应式

**结论先行**：WebFlux 用少量事件循环线程支撑大量并发连接，天然适配流式场景。

## 7.1 响应式模型

### 7.1.1 事件循环

Netty 事件循环线程数通常等于 CPU 核数，通过非阻塞 I/O 处理成千上万连接。

### 7.1.2 与同步模型对比

| 维度 | 同步（MVC） | 响应式（WebFlux） |
|------|-------------|-------------------|
| 线程模型 | 一请求一线程 | 事件循环 |
| 并发上限 | 受线程数限制 | 受连接与内存限制 |
| 流式支持 | 需特殊处理 | 原生 Flux |
| 编程难度 | 低 | 高 |

## 7.2 代码示例

```java
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chat(@RequestParam String q) {
    return chatClient.prompt()
            .user(q)
            .stream()
            .content();
}
```

## 7.3 边界与坑

- 响应式链路中任何阻塞调用（JDBC、同步 HTTP）都会拖垮事件循环。
- 调试与异常栈更复杂，团队需培训。
- CPU 密集任务应切到有界弹性线程池。

> 详见 `Architecture/2_webflux_sse.md`。

# 8. 并行化

**结论先行**：Agent 内部相互独立的操作应并行执行，把串行耗时压缩为最长分支的耗时。

## 8.1 并行场景

| 场景 | 并行对象 |
|------|----------|
| 多工具调用 | 无依赖的工具 |
| 混合检索 | 稠密 + 稀疏 |
| 多路召回 | 多数据源 |
| 多模型投票 | 多个模型 |

## 8.2 代码实现

### 8.2.1 并行工具调用

```java
// 并行执行无依赖的工具
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(
        () -> toolA.execute(), executor);
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(
        () -> toolB.execute(), executor);

CompletableFuture.allOf(f1, f2).join();
```

### 8.2.2 并行检索

```java
// 并行检索（稠密 + 稀疏）
CompletableFuture<List<Document>> dense = CompletableFuture.supplyAsync(
        () -> vectorSearch(q), executor);
CompletableFuture<List<Document>> sparse = CompletableFuture.supplyAsync(
        () -> bm25Search(q), executor);
```

## 8.3 边界与坑

- 必须使用独立的有界线程池，避免与 Web 线程池互相拖累。
- 并行分支要设置统一超时，任一超时即取消其余。
- 下游配额是共享的，并行会放大对配额的压力。

> 生产建议：对并行分支做全链路超时与熔断，防止一个慢分支拖垮整个请求。

# 9. 超时与取消

**结论先行**：异步任务必须支持超时与取消，否则失败任务会长期占用资源。

## 9.1 取消机制

### 9.1.1 协作式取消

取消不是强杀，而是设置标志，由 Worker 在检查点主动退出。

```java
public void cancel(String taskId) {
    AgentState state = stateStore.load(taskId);

    if (state.status() == AgentStatus.EXECUTING) {
        // 设置取消标志，Worker 在检查点处退出
        stateStore.setCancelled(taskId);
        log.info("任务已标记取消: {}", taskId);
    }
}
```

### 9.1.2 Worker 检查点

```java
// Worker 循环中检查
while (state.status() == AgentStatus.EXECUTING) {
    if (stateStore.isCancelled(taskId)) {
        transition(taskId, AgentStatus.CANCELLED);
        break;
    }
    state = step(state);
}
```

## 9.2 超时控制

| 层级 | 超时对象 | 建议值 |
|------|----------|--------|
| 单次模型调用 | LLM 请求 | 30~60 s |
| 单个工具调用 | 下游 API | 5~10 s |
| 整个任务 | 端到端 | 任务上限 |
| 连接 | TCP / 网关 | 5 s |

```java
// 用 JDK 21 虚拟线程 + 结构化并发设置任务级超时（伪代码）
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> toolA.execute());
    scope.fork(() -> toolB.execute());
    scope.joinUntil(Instant.now().plusSeconds(10));
}
```

## 9.3 边界与坑

- 取消是协作式的，阻塞在 I/O 上的步骤不会立即响应。
- 取消后要清理已占用的资源与已产生的副作用。
- 超时时间要小于上游网关超时，避免「上游已断、下游仍在跑」。

> 生产建议：为每类操作配置独立超时，并统计超时率作为健康指标。

# 10. 生产注意事项

**结论先行**：异步系统的风险集中在队列、背压、取消与监控四方面。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 队列无界 | 设置有界队列 |
| 无背压 | 队列满时拒绝 |
| 任务无法取消 | 支持取消标志 |
| 无进度反馈 | 提供查询接口 |
| 消息丢失 | 持久化 + ack |
| 无监控 | 监控积压与消费延迟 |
| 线程池不隔离 | 异步与同步任务分离 |
| 重复消费 | 幂等键去重 |

## 10.2 落地检查表

- 队列是否有界并配置背压？
- 消费是否幂等？
- 任务是否支持取消与超时？
- 是否监控积压、消费延迟与取消率？
- 线程池是否按用途隔离？

> 生产建议：把异步链路纳入全链路压测，验证积压恢复能力。

# 11. 小结

- LLM 调用慢，异步化释放线程、削峰填谷。
- 三种模式：同步、流式、异步任务。
- 队列用于削峰、解耦、可靠性。
- 必须有背压与取消机制。
- Agent 内部独立操作要并行化。
- 至少一次投递 + 幂等消费是最务实的组合。
