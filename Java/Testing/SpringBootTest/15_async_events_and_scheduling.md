# 1. 异步测试

## 1.1 `@Async` 的测试思路

异步方法通常在线程池中执行，测试需考虑：

- 线程切换导致的时序问题
- 事务上下文无法传播

## 1.2 使用同步执行器

在测试中可覆盖 `TaskExecutor`：

```java
@TestConfiguration
static class SyncExecutorConfig {
    @Bean
    public TaskExecutor taskExecutor() {
        return new SyncTaskExecutor();
    }
}
```

此方式让 `@Async` 方法在当前线程同步执行，便于断言。

## 1.3 等待异步完成

```java
CountDownLatch latch = new CountDownLatch(1);
asyncService.run(() -> latch.countDown());
assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
```

---

# 2. 定时任务测试

## 2.1 核心策略

- 将调度逻辑与业务逻辑拆分
- 直接测试业务方法，减少对定时触发的依赖

## 2.2 覆盖调度器

```java
@TestConfiguration
static class SchedulerConfig {
    @Bean
    public TaskScheduler taskScheduler() {
        return new ConcurrentTaskScheduler(Executors.newSingleThreadScheduledExecutor());
    }
}
```

---

# 3. 事件驱动测试

## 3.1 事件监听器

```java
@Component
class OrderCreatedListener {
    @EventListener
    public void on(OrderCreatedEvent event) {}
}
```

## 3.2 记录与断言事件

Spring Boot 提供 `@RecordApplicationEvents`：

```java
@SpringBootTest
@RecordApplicationEvents
class EventTest {

    @Autowired
    private ApplicationEvents events;

    @Test
    void shouldPublishEvent() {
        // 调用业务逻辑
        assertThat(events.stream(OrderCreatedEvent.class)).hasSize(1);
    }
}
```

---

# 4. 常见问题

- 异步测试过度等待 → 引入 `SyncTaskExecutor`
- 定时任务触发难 → 将调度与业务解耦
- 事件数量不一致 → 检查事务提交时机
