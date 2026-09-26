# 1. 什么是舱壁模式

舱壁模式（bulkhead pattern）通过资源分区实现故障隔离。结论先行：**把资源划分为互不干扰的池，让单一依赖的故障只耗尽自己那一份资源，而不波及整个系统**。

## 1.1 起源

来源于船舶设计：**船体被分成多个水密舱**，一个舱进水不会导致整船沉没。

```
船体横截面:
┌────┬────┬────┬────┐
│ 舱1│ 舱2│ 舱3│ 舱4│
└────┴────┴────┴────┘
  进水 → 只淹没该舱，其余仍提供浮力
```

## 1.2 软件中的舱壁

软件中的舱壁隔离：

```
一个依赖故障 → 只耗尽分配给它的资源 → 其他功能不受影响
```

实现方式有两种：线程池隔离（每个依赖独占线程池）与信号量隔离（每个依赖独占并发许可）。

## 1.3 与限流/熔断的区别

| 模式 | 解决的问题 | 作用对象 |
|------|------------|----------|
| 限流 | 请求速率过大 | 请求入口 |
| 熔断 | 依赖持续故障 | 单个依赖 |
| 舱壁 | 资源被单一依赖耗尽 | 资源池 |
| 降级 | 服务不可用时的替代 | 调用结果 |

舱壁关注「资源」，熔断关注「调用」，限流关注「速率」，三者互补。

## 1.4 边界与反例

- **反例：舱壁数量过多**。每个依赖一个线程池会导致线程数爆炸，上下文切换开销剧增。
- **边界：舱壁不减少总资源**。它只做分配，不增加容量；分区太细会让每份资源都不足。
- **边界：共享下游仍会互相影响**。两个舱壁若调用同一个下游，下游过载时两者都会失败。

# 2. 为什么 Agent 需要隔离

结论先行：**Agent 同时调用模型、检索、工具、MCP 等多种依赖，若共用线程池，任一慢依赖都会拖垮全部功能**。

## 2.1 共享资源的故障传播

```
Agent 调用多种依赖：模型、向量库、工具、MCP Server
若共用同一个线程池：
  某个慢工具占满线程 → 模型调用也排队 → 整个服务不可用
```

## 2.2 无隔离 vs 有隔离

```
无隔离：
┌─────────────────────────────────┐
│  共享线程池（20 线程）           │
│  工具A ×20（全部卡住）           │  ← 模型调用无线程可用
└─────────────────────────────────┘

有隔离：
┌──────────┐ ┌──────────┐ ┌──────────┐
│ 模型池 ×10│ │ 工具A ×5 │ │ 工具B ×5 │
└──────────┘ └──────────┘ └──────────┘
              ↑ 工具A 卡死，不影响模型与工具B
```

## 2.3 边界与反例

- **反例：只隔离线程不隔离连接**。HTTP 客户端连接池若共享，线程隔离也无效。
- **边界：隔离不能防依赖本身故障**。它只限制影响范围，依赖故障仍需熔断与降级。

# 3. 隔离的维度

结论先行：**按「依赖类型、工具、租户、用户等级、任务类型」五个维度划分舱壁，粒度以「故障域一致」为原则**。

## 3.1 五维度表

| 维度 | 说明 |
|------|------|
| 依赖类型 | 模型、检索、工具分别隔离 |
| 工具 | 每个工具独立线程池 |
| 租户 | 大租户独立资源 |
| 用户等级 | VIP 与普通用户隔离 |
| 任务类型 | 同步请求与异步任务隔离 |

## 3.2 边界与反例

- **反例：按每个用户建舱壁**。用户数量巨大，舱壁数量爆炸，应改为「等级分组」。
- **边界：维度交叉**。同时按工具与租户隔离会产生 N×M 个池，需限定组合。

# 4. 线程池隔离

结论先行：**线程池隔离为每个依赖分配独立线程与有界队列，支持超时中断，是隔离最彻底的方式，但开销最高**。

## 4.1 机制

每个依赖拥有独立的 `ThreadPoolExecutor`。调用被提交到对应线程池，主线程通过 `Future.get(timeout)` 等待，超时则取消任务并中断工作线程。

### 4.1.1 调用流程

```
请求线程 ──提交──▶ 依赖A线程池 ──▶ 依赖A
   │                    (独立队列)
   └──提交──▶ 依赖B线程池 ──▶ 依赖B
```

工作流程分四步：提交任务、工作线程执行、主线程限时等待、超时取消。核心线程数等于最大线程数时，超出核心数的任务直接进入队列；队列满则触发拒绝策略。

### 4.1.2 超时中断

```java
Future<String> future = modelExecutor.submit(() -> model.chat(prompt));
try {
    return future.get(3, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true);   // 中断工作线程
    throw new ToolTimeoutException("模型调用超时");
}
```

`cancel(true)` 会向工作线程发送中断信号，但能否真正停止取决于调用是否响应中断。阻塞式 IO 若不响应中断，线程仍会占用到调用结束。

## 4.2 配置

### 4.2.1 线程池参数

```java
@Configuration
public class BulkheadConfig {

    // 模型调用线程池
    @Bean("modelExecutor")
    public ExecutorService modelExecutor() {
        return new ThreadPoolExecutor(
                10, 10, 0, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(100),
                new ThreadPoolExecutor.AbortPolicy());   // 满了直接拒绝
    }

    // 检索线程池
    @Bean("retrievalExecutor")
    public ExecutorService retrievalExecutor() {
        return new ThreadPoolExecutor(
                8, 8, 0, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(200),
                new ThreadPoolExecutor.AbortPolicy());
    }

    // 工具线程池（每个工具独立或按组）
    @Bean("toolExecutor")
    public ExecutorService toolExecutor() {
        return new ThreadPoolExecutor(
                16, 16, 0, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(200),
                new ThreadPoolExecutor.AbortPolicy());
    }
}
```

> 关键：**队列要有界**，无界队列会让请求无限堆积，最终 OOM。

### 4.2.2 参数含义

| 参数 | 含义 | 建议 |
|------|------|------|
| corePoolSize | 核心线程数 | 按下游容量 |
| maximumPoolSize | 最大线程数 | 与核心一致防抖动 |
| keepAliveTime | 空闲回收时间 | 0 表示不回收 |
| queue | 等待队列 | 必须有界 |
| handler | 拒绝策略 | AbortPolicy |

## 4.3 拒绝策略

| 策略 | 行为 | 适用 |
|------|------|------|
| AbortPolicy | 抛 `RejectedExecutionException` | 快速失败 |
| CallerRunsPolicy | 调用者线程执行 | 反压 |
| DiscardPolicy | 静默丢弃 | 可丢任务 |
| DiscardOldestPolicy | 丢最旧任务 | 只保最新 |

生产上推荐 `AbortPolicy` 配合降级，或 `CallerRunsPolicy` 实现自然反压，但后者会阻塞调用线程，需评估。注意 `new LinkedBlockingQueue<>()` 无参构造即无界，是常见事故来源；线程数应基于下游能承受的并发而非 CPU 核数。

# 5. 信号量隔离

结论先行：**信号量隔离只限制并发数、不创建额外线程，开销低但无法超时中断，适合纯限并发场景**。

## 5.1 机制

信号量（semaphore）隔离用 `Semaphore` 控制并发许可，调用仍在当前线程执行，不涉及线程切换。

```java
// 限制并发数，不额外创建线程
Semaphore modelSemaphore = new Semaphore(10);

public String callModel(String prompt) {
    if (!modelSemaphore.tryAcquire(500, TimeUnit.MILLISECONDS)) {
        throw new BulkheadFullException("模型调用并发已达上限");
    }
    try {
        return model.chat(prompt);
    } finally {
        modelSemaphore.release();
    }
}
```

## 5.2 代码

```java
public class SemaphoreBulkhead {

    private final Semaphore semaphore;
    private final Duration maxWait;

    public SemaphoreBulkhead(int permits, Duration maxWait) {
        this.semaphore = new Semaphore(permits);
        this.maxWait = maxWait;
    }

    public <T> T execute(Supplier<T> action) {
        boolean acquired = false;
        try {
            acquired = semaphore.tryAcquire(maxWait.toMillis(),
                    TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new BulkheadFullException("并发已满");
            }
            return action.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BulkheadFullException("等待被中断");
        } finally {
            if (acquired) {
                semaphore.release();
            }
        }
    }
}
```

## 5.3 线程池 vs 信号量

| 隔离方式 | 开销 | 超时控制 | 适用 |
|----------|:----:|:--------:|------|
| 线程池 | 高 | 支持 | 需要超时中断 |
| 信号量 | 低 | 不支持 | 纯限并发 |

> 信号量隔离的「超时」只是获取许可的超时，一旦获得许可，调用本身无法被中断，仍需依赖底层超时机制。

### 5.3.1 选择建议

| 场景 | 推荐 | 原因 |
|------|------|------|
| 阻塞 IO、需硬超时 | 线程池 | 可中断 |
| 异步非阻塞 | 信号量 | 无线程切换开销 |
| 高并发轻量调用 | 信号量 | 内存占用小 |
| 隔离要求严格 | 线程池 | 线程级隔离 |

## 5.4 边界与反例

- **反例：信号量忘记释放**。异常路径未在 `finally` 释放会导致许可泄漏，最终永久阻塞。
- **边界：信号量不隔离线程**。若调用是阻塞 IO，占用的仍是调用者线程。
- **边界：许可数与下游容量不匹配**。许可数应等于下游能同时处理的请求数。

# 6. Resilience4j Bulkhead

结论先行：**Resilience4j 同时提供信号量舱壁与线程池舱壁，可与熔断、重试统一装饰**。

## 6.1 信号量舱壁

```java
// 信号量舱壁
Bulkhead bulkhead = Bulkhead.of("model", BulkheadConfig.custom()
        .maxConcurrentCalls(10)
        .maxWaitDuration(Duration.ofMillis(500))
        .build());
```

## 6.2 线程池舱壁

### 6.2.1 配置参数

| 参数 | 含义 | 建议 |
|------|------|------|
| maxThreadPoolSize | 最大线程数 | 按下游容量 |
| coreThreadPoolSize | 核心线程数 | 与最大一致 |
| queueCapacity | 队列容量 | 有界，100 左右 |
| keepAliveDuration | 空闲回收 | 20ms 起 |
| writableStackTraceEnabled | 是否记录栈 | 生产建议关闭 |

```java
// 线程池舱壁
ThreadPoolBulkhead threadPoolBulkhead = ThreadPoolBulkhead.of("tools",
        ThreadPoolBulkheadConfig.custom()
                .maxThreadPoolSize(16)
                .coreThreadPoolSize(16)
                .queueCapacity(100)
                .build());

public CompletableFuture<String> callTool(String args) {
    return ThreadPoolBulkhead.decorateSupplier(threadPoolBulkhead,
            () -> tool.execute(args)).get();
}
```

线程池舱壁返回 `CompletableFuture`，天然支持异步与超时。

## 6.3 组合使用

```java
Supplier<String> decorated = Decorators.ofSupplier(() -> model.chat(prompt))
        .withCircuitBreaker(circuitBreaker)
        .withBulkhead(bulkhead)
        .withRateLimiter(rateLimiter)
        .withRetry(retry)
        .decorate();
```

## 6.4 边界与反例

- **反例：同时用信号量与线程池舱壁**。双重限制会让实际并发远低于预期，通常二选一。
- **边界：线程池舱壁队列满**。默认抛 `BulkheadFullException`，需捕获并降级。

# 7. 按工具隔离

结论先行：**每个工具或每组工具独立舱壁，避免某个慢工具耗尽全部并发；并发额度按工具特性差异化配置**。

## 7.1 注册表实现

```java
public class ToolBulkheadRegistry {

    private final Map<String, Semaphore> semaphores = new ConcurrentHashMap<>();

    public ToolResult execute(ToolCall call, int maxConcurrent) {
        Semaphore semaphore = semaphores.computeIfAbsent(
                call.name(), n -> new Semaphore(maxConcurrent));

        if (!semaphore.tryAcquire(200, TimeUnit.MILLISECONDS)) {
            return ToolResult.fail("该工具并发已达上限，请稍后重试");
        }
        try {
            return doExecute(call);
        } finally {
            semaphore.release();
        }
    }
}
```

## 7.2 并发建议

| 工具类型 | 并发建议 |
|----------|----------|
| 快速查询 | 高（20~50） |
| 外部 API | 中（10~20） |
| 重计算 | 低（2~5） |
| 写操作 | 低（5~10） |

并发额度应基于工具的下游容量与自身资源消耗，而非统一设定。注意工具名动态出现时 `computeIfAbsent` 会持续创建信号量，需清理不再使用的条目。

# 8. 租户隔离

结论先行：**多租户场景下应为租户分配独立舱壁，防止大租户挤占小租户资源，实现公平性与噪声隔离**。

## 8.1 问题

大租户可能挤占小租户资源：

```
无租户隔离:
┌───────────────────────────┐
│  共享并发 ×20             │
│  租户A ×20（占满）        │  ← 租户B、C 全部被拒
└───────────────────────────┘
```

## 8.2 实现

```java
public class TenantBulkhead {

    private final Map<String, Semaphore> tenantSemaphores = new ConcurrentHashMap<>();
    private static final int DEFAULT_PERMITS = 10;

    public boolean tryAcquire(String tenantId) {
        Semaphore semaphore = tenantSemaphores.computeIfAbsent(
                tenantId, t -> new Semaphore(tenantPermits(t)));
        return semaphore.tryAcquire();
    }

    private int tenantPermits(String tenantId) {
        // 大租户分配更多配额
        return tenantQuotaService.permits(tenantId);
    }
}
```

### 8.2.1 配额分配

租户配额可按套餐等级分配，等级越高许可越多：

| 套餐 | 并发许可 | 说明 |
|------|:--------:|------|
| 免费 | 2 | 保底可用 |
| 标准 | 10 | 默认 |
| 企业 | 50 | 独立池 |
| 内部 | 不限 | 不隔离 |

配额变更应支持热更新，避免重启应用。租户隔离有硬隔离、软隔离与层级隔离三种策略，硬隔离资源利用率低，软隔离隔离弱，层级隔离（大租户独立、小租户共享）较折中。

## 8.3 边界与反例

- **反例：每租户一个线程池**。租户数量大时线程爆炸，应用信号量或共享池 + 配额。
- **边界：租户数无上限**。`ConcurrentHashMap` 会随租户增长，需设置容量上限或使用 LRU 淘汰。
- **边界：小租户饿死**。共享池下小租户可能长期抢不到许可，需预留保底额度。

# 9. 隔离与降级配合

结论先行：**舱壁满时应触发降级而非简单报错，让用户在资源紧张时仍能获得降级服务**。

## 9.1 舱壁满的处理

```
舱壁满
  │
  ├─ 降级到备用服务
  ├─ 返回缓存结果
  ├─ 返回排队提示
  └─ 明确拒绝（带 Retry-After）
```

### 9.1.1 降级策略选择

| 场景 | 策略 | 说明 |
|------|------|------|
| 可缓存查询 | 返回缓存 | 标注时效 |
| 有备用服务 | 降级调用 | 独立舱壁 |
| 延迟不敏感 | 排队提示 | 引导稍后 |
| 写操作 | 明确拒绝 | 不可降级 |

## 9.2 代码

```java
public String handle(String prompt) {
    try {
        return modelBulkhead.execute(() -> model.chat(prompt));
    } catch (BulkheadFullException e) {
        // 并发满 → 降级
        return fallbackService.answer(prompt);
    }
}
```

## 9.3 边界与反例

- **反例：降级路径也占用同一舱壁**。降级调用若再次进入同一舱壁，会递归耗尽许可。
- **边界：降级服务容量**。降级服务同样有容量上限，需独立舱壁与限流。
- **边界：降级放大延迟**。先等舱壁超时再降级会增加总延迟，可考虑快速失败后立即降级。

# 10. 监控与容量

结论先行：**舱壁需监控活跃线程数、队列长度、拒绝数，并据此动态调整线程与队列容量**。

## 10.1 指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| 活跃线程数 | 当前执行中的任务 | 持续接近上限 |
| 队列长度 | 等待任务数 | 持续增长 |
| 拒绝数 | 被拒绝的任务 | 出现即关注 |
| 平均等待时长 | 队列等待时间 | 明显上升 |

```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) modelExecutor;
metrics.gauge("bulkhead.active", executor.getActiveCount());
metrics.gauge("bulkhead.queue", executor.getQueue().size());
```

### 10.1.1 指标采集

建议以固定周期采集并上报，而非在每次调用时采集，避免高频指标开销：

```java
@Scheduled(fixedRate = 5000)
public void reportBulkheadMetrics() {
    ThreadPoolExecutor ex = (ThreadPoolExecutor) modelExecutor;
    metrics.gauge("bulkhead.model.active", ex.getActiveCount());
    metrics.gauge("bulkhead.model.queue", ex.getQueue().size());
    metrics.gauge("bulkhead.model.completed", ex.getCompletedTaskCount());
}
```

活跃线程数反映并发压力，队列长度反映积压程度，两者需同时观察。

## 10.2 线程池调优

线程数估算可参考：

```
线程数 = 下游目标并发（或）
        CPU 核数 × 目标利用率 × (1 + 等待时间 / 计算时间)
```

IO 密集型任务等待时间长，可放大线程数；但必须受下游容量约束。

## 10.3 告警

- 队列长度持续为满：下游处理能力不足或流量突增。
- 拒绝数突增：舱壁过小或依赖变慢。
- 活跃线程长期满载：需扩容或优化下游。注意线程未满但队列堆积说明消费速率不足，且线程池指标是瞬时值，需用滑动平均观察趋势。

# 11. 生产注意事项

## 11.1 常见风险表

| 风险 | 建议 |
|------|------|
| 无界队列 | 必须设置有界队列 |
| 线程池过大 | 按下游承载能力设置 |
| 隔离粒度太细 | 按组隔离，避免线程爆炸 |
| 无监控 | 监控队列长度、拒绝数 |
| 无降级 | 舱壁满时要有兜底 |
| 线程池不隔离 | 关键路径必须独立 |
| 信号量泄漏 | 必须在 finally 释放 |
| 连接池共享 | 与线程池同步隔离 |

## 11.2 配置清单

```yaml
bulkhead:
  model: { type: threadpool, core-size: 10, max-size: 10, queue-capacity: 100 }
  retrieval: { type: threadpool, core-size: 8, max-size: 8, queue-capacity: 200 }
  tools: { type: semaphore, default-permits: 20, max-wait: 200ms }
  tenant: { type: semaphore, default-permits: 10, max-tenants: 10000 }
```

# 12. 小结

- 舱壁隔离防止单个依赖拖垮整个系统。
- 线程池隔离支持超时中断，信号量隔离更轻量。
- 队列必须有界，否则请求堆积导致 OOM。
- 按依赖、工具、租户等维度隔离。
- 舱壁满时要配合降级。
- 隔离需连同连接池、内存一起考虑，并持续监控队列与拒绝数。
