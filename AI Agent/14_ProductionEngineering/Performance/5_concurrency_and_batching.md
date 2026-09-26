# 1. 并发控制的目标

**结论先行**：并发控制的目标是"既要充分利用资源，又不能压垮下游"，即在吞吐与稳定之间找到可运营的平衡点。

```
既要充分利用资源，又不能压垮下游
```

## 1.1 吞吐与稳定的平衡

### 1.1.1 并发过低

| 问题 | 后果 |
|------|------|
| 并发过低 | 资源闲置、吞吐低 |

### 1.1.2 并发过高

| 问题 | 后果 |
|------|------|
| 并发过高 | 下游崩溃、限流、成本失控 |
| 无隔离 | 慢操作拖垮全局 |

## 1.2 隔离原则

**结论先行**：不同下游依赖必须使用独立线程池/信号量，避免相互拖累。

```
模型调用池    ← 慢、易限流
向量库池      ← 中等
工具调用池    ← 外部依赖，最不可控
```

- 慢依赖独立池，防止占满公共线程。
- 关键路径与离线任务分离。
- 池大小按各自下游容量分别设定。

## 1.3 目标量化

| 目标 | 指标 |
|------|------|
| 吞吐 | QPS / 每分钟完成任务数 |
| 稳定 | 错误率、429 比例 |
| 延迟 | P50 / P95 / P99 |
| 资源 | 线程池活跃度、队列长度 |

> 生产建议：并发上限应以"下游 429 率 < 阈值"为约束反推，而非拍脑袋设定。

---

# 2. 并发模型

**结论先行**：不同并发模型适用于不同场景，LLM 应用多为 I/O 密集，线程池 + 有界队列是主流选择。

## 2.1 模型对比

| 模型 | 说明 | 适用 |
|------|------|------|
| 同步阻塞 | 一请求一线程 | 低并发 |
| 线程池 | 复用线程 | 中高并发 |
| 异步非阻塞 | 事件循环 | 高并发 I/O |
| 响应式 | 背压流控 | 流式、高并发 |

## 2.2 线程池实现

### 2.2.1 核心参数

```java
// 线程池：控制并发
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        16, 16,                          // 核心/最大线程
        0, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<>(200),  // 有界队列
        new ThreadPoolExecutor.AbortPolicy());
```

| 参数 | 含义 | 建议 |
|------|------|------|
| corePoolSize | 常驻线程数 | 按稳态并发 |
| maximumPoolSize | 最大线程数 | 按峰值并发 |
| 队列 | 缓冲 | 必须有界 |
| 拒绝策略 | 过载处理 | 快速失败 |

### 2.2.2 拒绝策略

```java
// 过载时快速失败，而不是无限排队
new ThreadPoolExecutor.AbortPolicy();
// 或用调用者运行，形成天然背压
new ThreadPoolExecutor.CallerRunsPolicy();
```

> **队列必须有界**，否则请求堆积导致 OOM。

## 2.3 异步与响应式

- `CompletableFuture` 适合组合少量异步任务。
- 响应式（如 Reactor）适合流式与声明式并发控制。
- 异步代码要避免阻塞调用污染事件循环。

> 生产建议：I/O 密集用有界线程池；高并发流式场景用响应式 + 背压。

---

# 3. 并发度确定

**结论先行**：并发度由"下游承载能力、本地资源限制、成本预算"三者取最小值决定。

## 3.1 反推公式

```
并发度 = min(下游承载能力, 本地资源限制, 成本预算)
```

```java
// 按 RPM 反推并发
// 若模型 500 RPM，平均耗时 2s，则并发 ≈ 500/60 * 2 ≈ 16
int concurrency = (int) (rpm / 60.0 * avgLatencySeconds);
```

## 3.2 按下游容量

| 下游 | 建议并发 |
|------|----------|
| 模型 API | 按 RPM/TPM 反推 |
| 向量库 | 按 QPS 容量 |
| 外部工具 API | 按对方限流 |
| 数据库 | 按连接池大小 |

## 3.3 动态调整

- 根据 429 比例动态降并发（自适应限流）。
- 根据队列长度动态扩并发（在安全上限内）。
- 下游恢复后逐步回升，避免抖动。

```java
// 伪代码：自适应并发——429 多则降，队列积压则升
public int adjust(int current, double rate429, int queueSize) {
    if (rate429 > 0.05) {
        return Math.max(1, (int) (current * 0.8)); // 过载，收缩
    }
    if (queueSize > 50 && current < MAX_CONCURRENCY) {
        return Math.min(MAX_CONCURRENCY, current + 2); // 空闲，扩张
    }
    return current;
}
```

> 生产建议：并发度做成可热更新的配置项，并记录每次调整的原因与效果。

---

# 4. Agent 内的并行

**结论先行**：Agent 内部彼此独立的操作应并行执行，以缩短总延迟。

## 4.1 并行工具调用

```java
public List<ToolResult> executeParallel(List<ToolCall> calls) {
    List<CompletableFuture<ToolResult>> futures = calls.stream()
            .map(call -> CompletableFuture.supplyAsync(
                    () -> toolExecutor.execute(call), toolPool))
            .toList();

    return futures.stream().map(CompletableFuture::join).toList();
}
```

## 4.2 并行检索

```java
CompletableFuture<List<Document>> dense = CompletableFuture.supplyAsync(
        () -> denseSearch(q), retrievalPool);
CompletableFuture<List<Document>> sparse = CompletableFuture.supplyAsync(
        () -> sparseSearch(q), retrievalPool);

List<Document> fused = rrfFuse(dense.join(), sparse.join());
```

```
          ┌── 稠密检索 ──┐
问题 ─────┤              ├── RRF 融合 ── 重排
          └── 稀疏检索 ──┘
```

## 4.3 并行子任务

```java
// Planner-Executor：无依赖的子任务并行执行
List<SubTask> independent = plan.stream()
        .filter(t -> t.dependsOn().isEmpty())
        .toList();

Map<String, TaskResult> results = independent.parallelStream()
        .collect(Collectors.toMap(
                SubTask::id,
                executor::execute));
```

> 生产建议：并行前先做依赖分析，无依赖才并行；有依赖的按 DAG 分层调度。

---

# 5. 批处理

**结论先行**：批处理把多个独立请求合并为一次调用，能减少系统提示重复与网络往返。

## 5.1 批量向量化

```java
// 一次请求向量化多段文本
List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
```

## 5.2 批量 LLM 调用

### 5.2.1 合并调用

```java
// 合并多个独立问题为一次调用
public List<String> batchClassify(List<String> items, int batchSize) {
    List<String> results = new ArrayList<>();

    for (int i = 0; i < items.size(); i += batchSize) {
        List<String> batch = items.subList(i,
                Math.min(i + batchSize, items.size()));

        String prompt = """
                请分别判断以下 %d 条的情感，按顺序输出 POSITIVE/NEGATIVE：
                %s
                """.formatted(batch.size(), formatNumbered(batch));

        results.addAll(parseLines(model.chat(prompt)));
    }
    return results;
}
```

### 5.2.2 分片

| 批大小 | 风险 |
|:------:|------|
| 过小 | 摊薄效果差 |
| 过大 | 输出被截断、单点失败影响大 |
| 适中（如 10~50） | 平衡 |

## 5.3 Batch API

| 批处理收益 | 说明 |
|------------|------|
| 减少系统提示重复 | 一次调用共享 |
| 减少网络往返 | 延迟降低 |
| 部分厂商批量折扣 | 成本降低 |

部分厂商提供异步 Batch API，价格更低（如 50%）：

```java
// 适合非实时任务：离线索引、批量分析
BatchJob job = batchApi.submit(requests);
// 稍后轮询结果
BatchResult result = batchApi.poll(job.id());
```

> 生产建议：实时路径不适合批处理，离线路径优先用 Batch API。

---

# 6. 背压

**结论先行**：背压（backpressure）让消费者控制生产速率，防止生产端压垮消费端。

## 6.1 背压策略

| 背压策略 | 说明 |
|----------|------|
| 有界队列 + 拒绝 | 简单直接 |
| 信号量控制 | 限制并发 |
| 响应式 flatMap(concurrency) | 声明式并发控制 |
| 动态限流 | 按下游负载调整 |

## 6.2 响应式实现

```java
// 响应式背压：消费者控制生产速率
Flux.fromStream(questions.stream())
        .flatMap(q -> Mono.fromCallable(() -> answer(q))
                .subscribeOn(Schedulers.boundedElastic()),
                4)                      // 并发 4，天然背压
        .subscribe(System.out::println);
```

```java
// 信号量：限制同时进行的调用数
Semaphore permits = new Semaphore(16);
permits.acquire();
try {
    return callModel(...);
} finally {
    permits.release();
}
```

## 6.3 边界

- 背压只在"下游能反馈速率"时有效。
- 阻塞式 API 无法背压，只能靠有界队列 + 拒绝。
- 拒绝要有可观测指标，不能静默丢弃。

> 生产建议：入口处即施加背压，越早拒绝成本越低。

---

# 7. 并发与幂等

**结论先行**：并发执行 + 副作用 = 必须幂等，否则重试会放大错误。

```
并发执行 + 副作用 = 需要幂等
```

## 7.1 幂等键

```java
// 幂等键由业务唯一标识生成
String key = order.id() + ":" + operation;
paymentService.chargeIdempotent(key, order.amount());
```

## 7.2 并行副作用

```java
// 并行执行有副作用的操作时，必须保证幂等
public void parallelCharge(List<Order> orders) {
    orders.parallelStream().forEach(order ->
            paymentService.chargeIdempotent(
                    order.idempotencyKey(), order.amount()));
}
```

## 7.3 去重

| 手段 | 说明 |
|------|------|
| 幂等键 | 服务端去重 |
| 去重表 | 记录已处理 ID |
| 唯一约束 | 数据库兜底 |

> 并行 + 重试 + 非幂等 = 重复副作用的高发场景。

---

# 8. 并发与顺序

**结论先行**：有些操作天然顺序敏感，必须串行，不能为了吞吐牺牲正确性。

## 8.1 必须串行的操作

有些操作**必须串行**：

| 操作 | 原因 |
|------|------|
| 同一会话的消息 | 顺序敏感 |
| 同一订单的状态变更 | 状态机顺序 |
| 累加计数 | 竞态 |

## 8.2 会话级锁

```java
// 会话级串行，会话间并行
public String chat(String sessionId, String message) {
    RLock lock = redisson.getLock("session:" + sessionId);
    lock.lock();
    try {
        return doChat(sessionId, message);
    } finally {
        lock.unlock();
    }
}
```

## 8.3 锁的坑

- 锁粒度太粗会退化为全局串行。
- 忘记设置超时会导致死锁。
- 分布式锁要考虑锁续期与释放原子性。

> 生产建议：锁的粒度按"业务唯一键"（会话 ID、订单 ID）设计，越小越好。

---

# 9. 监控

**结论先行**：并发系统的健康度必须通过队列、活跃线程、拒绝数等指标持续观测。

## 9.1 指标采集

```java
registry.gauge("executor.active", executor, ThreadPoolExecutor::getActiveCount);
registry.gauge("executor.queue", executor, e -> e.getQueue().size());
registry.counter("executor.rejected", executor, e -> e.getRejectedExecutionHandler());
```

## 9.2 告警规则

| 指标 | 告警 |
|------|------|
| 活跃线程数 | 持续满需扩容 |
| 队列长度 | 持续增长告警 |
| 拒绝数 | > 0 告警 |
| 任务耗时 P99 | 超阈值告警 |

## 9.3 容量规划

| 步骤 | 做法 |
|------|------|
| 估算 | 用 P99 耗时与目标 QPS 反推并发 |
| 留量 | 预留 20%~30% 余量应对突发 |
| 验证 | 定期压测验证并发上限 |
| 回归 | 每次扩缩容后复核 429 与 P99 |

- 用 P99 耗时与目标 QPS 反推所需并发。
- 预留 20%~30% 余量应对突发。
- 定期压测验证并发上限。

> 生产建议：队列长度与拒绝率是"过载最早期"的信号，优先告警。

---

# 10. 生产注意事项

**结论先行**：并发问题的根因多为"无界、无隔离、无幂等、无监控"，需逐条治理。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 无界队列 | 必须有界 |
| 并发过高 | 按下游容量设定 |
| 共享线程池 | 按依赖隔离 |
| 并行副作用非幂等 | 幂等 + 去重 |
| 该串行的并行了 | 会话/状态操作串行 |
| 无监控 | 监控队列与拒绝 |

## 10.2 检查表

- [ ] 所有线程池队列有界且有拒绝策略
- [ ] 不同下游依赖线程池隔离
- [ ] 并发上限可配置、可热更新
- [ ] 有副作用的并行操作幂等
- [ ] 会话/状态操作串行
- [ ] 队列、拒绝、P99 已接入告警

> 生产建议：把并发上限、队列长度、超时时间作为一组参数统一治理。

---

# 11. 小结

- 并发度 = min(下游承载、本地资源、成本预算)。
- Agent 内独立操作应并行，但有副作用的必须幂等。
- 批处理能减少系统提示重复与网络往返。
- 背压机制防止生产压垮消费。
- 会话等顺序敏感操作必须串行。
- 无界队列、无隔离、无监控是三大高频事故根因。
