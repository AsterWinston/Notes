# 1. 异步测试

异步是测试确定性的头号敌人：主线程发起调用后立即返回，工作在线程池里"稍后"完成——测试若直接断言结果，就是在和调度器赛跑。策略只有两条正路：**要么把异步同步化（让它在当前线程执行），要么保留异步但用受控等待（Awaitility/latch）**。`Thread.sleep` 不属于任何一条正路。

## 1.1 `@Async` 的测试思路

`@Async` 方法被 Spring 代理后提交给 `TaskExecutor` 线程池执行。测试需考虑：

- **线程切换导致的时序问题**：断言时任务可能还没跑；
- **事务上下文无法传播**：`@Async` + `@Transactional` 组合里，异步线程开自己的事务，看不到发起方未提交的数据（与第 13 篇的结论一致）；
- **异常被吞**：void 返回的 `@Async` 方法抛异常只进 `AsyncUncaughtExceptionHandler`，测试里不显式检查就永远发现不了。

两种策略的取舍：

| 策略 | 验证范围 | 速度 | 风险 |
|------|----------|------|------|
| 同步化 Executor | 业务逻辑本身 | 快、确定 | **测不到真实线程池行为**（队列满、拒绝策略、并发竞争） |
| 保留异步 + Awaitility | 逻辑 + 真实并发路径 | 慢一点 | 超时阈值设置不当会 flaky |

**行业实践**是双轨：单元/切片测试用同步化 Executor 验证逻辑，另留少量集成测试用真实线程池 + Awaitility 验证"异步这件事本身"（任务确实被提交、并发下无共享状态问题）。

## 1.2 使用同步执行器

在测试中覆盖 `TaskExecutor`，让 `@Async` 在调用线程内执行：

```java
@TestConfiguration
static class SyncExecutorConfig {

    @Bean
    @Primary                                    // 顶掉生产线程池
    public TaskExecutor applicationTaskExecutor() {
        return new SyncTaskExecutor();          // execute() 就是直接 run()
    }
}

@SpringBootTest
@Import(SyncExecutorConfig.class)
class AsyncSyncedTest {

    @Autowired
    private ReportService reportService;        // 内部 @Async 方法

    @Test
    void should_complete_inline() {
        reportService.generateAsync("2026-01"); // 同步执行完才返回
        assertThat(reportRepository.count()).isEqualTo(1);  // 可直接断言，无需等待
    }
}
```

此方式让 `@Async` 方法在当前线程同步执行，便于断言。注意两个细节：

- Boot 3.x 的自动配置 Bean 名叫 `applicationTaskExecutor`，覆盖时同名 + `@Primary` 最稳；
- 同步化后**返回 `CompletableFuture` 的方法在调用线程内就完成了 future**，`future.get()` 永不阻塞——这正好方便断言（见 1.4）。

## 1.3 等待异步完成

保留真实异步时，用受控等待。三种手段按推荐度排序：

```java
// ① Awaitility（推荐）：轮询条件直到成立，语义清晰、超时可控
await().atMost(3, TimeUnit.SECONDS)
       .pollInterval(50, TimeUnit.MILLISECONDS)
       .untilAsserted(() ->
           assertThat(reportRepository.findByMonth("2026-01")).isPresent());

// ② CountDownLatch：被测代码能接收回调时的精确同步
CountDownLatch latch = new CountDownLatch(1);
asyncService.run(() -> latch.countDown());
assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();   // 超时返回 false 而非卡死

// ③ CompletableFuture 直接 join/get（返回 future 的 API 天然可等）
CompletableFuture<Result> future = asyncService.compute(input);
assertThat(future.get(2, TimeUnit.SECONDS).value()).isEqualTo(42);
```

> **注意**：Awaitility 默认超时 10 秒、轮询 100ms，多数场景不用改。`untilAsserted` 与 `until` 的区别：前者包住断言（断言失败视为"还没好"继续轮询），后者包住布尔条件。等待期间**数据库可见性问题**依旧存在——若测试方法带 `@Transactional`，异步线程看不到测试事务未提交的数据，等待会白等到超时（第 13 篇第 6 节的坑在异步场景加倍常见）。

## 1.4 CompletableFuture 断言

`CompletableFuture` 是异步 API 的最佳返回类型——可组合、可等待、可断言：

```java
@Test
void should_complete_with_result() throws Exception {
    CompletableFuture<Price> future = pricingService.quoteAsync(item);

    Price price = future.get(2, TimeUnit.SECONDS);        // 显式超时，不裸 get()
    assertThat(price.amount()).isPositive();
}

@Test
void should_complete_exceptionally_when_upstream_down() {
    given(upstreamClient.fetch(any())).willThrow(new ConnectException());

    CompletableFuture<Price> future = pricingService.quoteAsync(item);

    assertThatThrownBy(() -> future.get(1, TimeUnit.SECONDS))
            .isInstanceOf(ExecutionException.class)
            .hasCauseInstanceOf(ConnectException.class);  // 异常链要拆到真实原因
    // 或者：assertThat(future.isCompletedExceptionally()).isTrue();
}
```

裸 `future.get()`（无超时）在测试里是禁用项——上游 bug 会让测试永久挂起而不是失败。JUnit 5 的 `assertTimeoutPreemptively` 可以做外层保险，但直接给 `get(timeout, unit)` 更简单。

---

# 2. 定时任务测试

## 2.1 核心策略

`@Scheduled` 测试的第一原则：**将调度逻辑与业务逻辑拆分**。

```java
@Component
class InvoiceRetryJob {

    @Scheduled(cron = "0 0/10 * * * *")        // 调度声明：薄薄一层
    public void trigger() {
        retryOverdueInvoices();                 // 业务逻辑：独立公开方法
    }

    public int retryOverdueInvoices() {         // 测试直接调它，不碰调度器
        return invoiceRepository.findOverdue().size();
    }
}

@SpringBootTest
class InvoiceRetryJobTest {

    @Autowired
    private InvoiceRetryJob job;

    @Test
    void should_retry_overdue() {
        seedOverdueInvoice();
        assertThat(job.retryOverdueInvoices()).isEqualTo(1);   // 手动触发业务方法
    }
}
```

直接测试业务方法，减少对定时触发的依赖——**"cron 表达式写对了"本身几乎不值得测**（它是声明，不是逻辑）；值得测的是"触发时执行的业务"。真正需要验证调度接线（表达式是否被注册、fixedDelay 是否生效）时，才动调度器。

## 2.2 覆盖调度器

测试环境的默认动作是**关掉调度**，防止后台任务在断言期间突然执行造成 flaky：

```yaml
# application-test.yml
app:
  scheduler:
    enabled: false          # 配合 @ConditionalOnProperty 包住 @EnableScheduling 的配置类
```

需要保留调度但想控制线程模型时，覆盖 `TaskScheduler`：

```java
@TestConfiguration
static class SchedulerConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        // 单线程调度器：任务串行执行，行为可预测
        return new ConcurrentTaskScheduler(Executors.newSingleThreadScheduledExecutor());
    }
}
```

验证"任务真的被调度了"的集成手段：把 cron 设为每秒触发（`@Scheduled(cron = "* * * * * *")` 的测试专用配置或 `fixedRate = 100`），配合 Awaitility 等计数器增长——这类测试慢且贴近 flaky，只留给调度接线本身的冒烟用例。

## 2.3 Clock 抽象与时间测试

一切"依赖当前时间"的逻辑（超时判断、账期计算、重试间隔）都应通过注入 `Clock` 而非 `Instant.now()`/`LocalDate.now()` 获取时间——这是时间可测性的唯一正解：

```java
// 生产代码：时间从注入的 Clock 来
@Service
class InvoiceService {

    private final Clock clock;                       // 构造注入

    InvoiceService(Clock clock) { this.clock = clock; }

    public boolean isOverdue(Invoice invoice) {
        return Instant.now(clock).isAfter(invoice.dueDate());   // 可被假时钟控制
    }
}

// 主配置提供系统时钟
@Configuration
class TimeConfig {
    @Bean
    Clock systemClock() { return Clock.systemDefaultZone(); }
}

// 测试配置提供固定时钟
@TestConfiguration
static class FixedClockConfig {
    @Bean
    @Primary
    Clock fixedClock() {
        return Clock.fixed(Instant.parse("2026-03-01T00:00:00Z"), ZoneOffset.UTC);
    }
}

@SpringBootTest
@Import(FixedClockConfig.class)
class InvoiceOverdueTest {

    @Test
    void should_be_overdue_after_due_date() {
        Invoice invoice = new Invoice(Instant.parse("2026-02-01T00:00:00Z"));
        assertThat(invoiceService.isOverdue(invoice)).isTrue();   // "现在"被固定在 3 月 1 日
    }
}
```

需要时间**推进**（测"7 天后过期提醒"）时，固定时钟不够，用可变时钟：

```java
class MutableClock extends Clock {                    // 测试专用时钟：手动拨表
    private Instant now;
    MutableClock(Instant start) { this.now = start; }
    void advance(Duration d) { now = now.plus(d); }
    @Override public ZoneId getZone() { return ZoneOffset.UTC; }
    @Override public Clock withZone(ZoneId zone) { return this; }
    @Override public Instant instant() { return now; }
}
```

> **行业实践**：Spring Framework 6.2+ 自带 `org.springframework.test.util.MutableClock`（`MutableClock.systemUTC()` 起步，`advance(Duration)` 拨表），不必再手写。团队规范里应写明：**生产代码禁止静态调用 `LocalDate.now()`，一律 `LocalDate.now(clock)`**——静态检查（ArchUnit/ErrorProne）能守住这条线。

---

# 3. 事件驱动测试

## 3.1 事件监听器

Spring 的 `ApplicationEvent` 机制默认**同步**执行（发布者线程直接调监听器），标了 `@Async` 或 `@TransactionalEventListener` 才有异步/事务语义：

```java
@Component
class OrderCreatedListener {

    @EventListener                              // 同步：发布即执行，在发布者线程
    public void on(OrderCreatedEvent event) { }
}

@Component
class NotificationListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderCreatedEvent event) { } // 事务提交后才执行；无事务默认不执行！
}
```

`@TransactionalEventListener` 的默认行为是测试盲区高发区：**发布者没有活动事务时，监听器默认被跳过**（`fallbackExecution = false`）。测试里发布事件后断言"监听器执行了"却失败，多半是这个原因——要么发布方包在事务里，要么监听器声明 `fallbackExecution = true`。

## 3.2 记录与断言事件

验证"业务动作发布了正确的事件"，Spring Framework 5.3.3+ 提供 `@RecordApplicationEvents` + `ApplicationEvents`，这是**事件发布侧测试的标准工具**：

```java
import org.springframework.test.context.event.ApplicationEvents;
import org.springframework.test.context.event.RecordApplicationEvents;

@SpringBootTest
@RecordApplicationEvents                      // 为每个测试方法记录发布的事件
class OrderEventTest {

    @Autowired
    private ApplicationEvents events;         // 每方法自动重置，注入的是当前方法的记录器

    @Autowired
    private OrderService orderService;

    @Test
    void should_publish_event_when_order_created() {
        orderService.create(cart);            // 调用业务逻辑

        assertThat(events.stream(OrderCreatedEvent.class))
                .hasSize(1)
                .first()
                .satisfies(e -> assertThat(e.orderId()).isEqualTo(1001L));
    }

    @Test
    void should_not_publish_when_validation_fails() {
        assertThatThrownBy(() -> orderService.create(emptyCart))
                .isInstanceOf(IllegalArgumentException.class);
        assertThat(events.stream(OrderCreatedEvent.class)).isEmpty();  // 负例同样重要
    }
}
```

要点：

- `ApplicationEvents` 记录的是**当前测试方法**内发布的所有事件（方法结束自动清空），线程安全的流式 API；
- 它同样适用于**纯 Jupiter 测试**（配合 `@ExtendWith(SpringExtension.class)` 发布 `ApplicationEventPublisher` 的场景较少，主要用于容器测试）；
- 事件的**消费侧**（监听器逻辑）应该单独直测：`new OrderCreatedListener(deps).on(event)`——监听器也是普通类，别为了测它启动容器。

```text
 事件测试的两侧分工
┌─────────────────────────────┐      ┌─────────────────────────────┐
│ 发布侧：@RecordApplicationEvents │      │ 消费侧：直接 new 监听器调用     │
│ 断言 events.stream(...)       │      │ 断言监听器产生的副作用           │
│ （业务代码发布了什么）           │      │ （监听器拿到事件做了什么）        │
└─────────────────────────────┘      └─────────────────────────────┘
        两侧独立测试，不做"发布 → 监听"的容器内全链路串联（除非冒烟）
```

> **设计争议**：`@RecordApplicationEvents` 测发布侧、直测监听器测消费侧，两者都绿**并不保证事件真的能送达**（监听器没注册、类型不匹配、`@TransactionalEventListener` 无事务跳过等问题在两侧测试里都发现不了）。所以每个事件类型至少要有一条**全链路冒烟**：真实发布 → 断言监听器副作用发生。

---

# 4. 常见问题

| 症状 | 根因 | 解法 |
|------|------|------|
| 异步测试过度等待、偶发超时 | 真实线程池 + sleep 轮询 | 引入 `SyncTaskExecutor`（逻辑测试）或 Awaitility（并发测试） |
| 断言时数据"看不到" | 测试事务未提交，异步线程另一连接 | 放弃测试事务或用同步 Executor（第 13 篇第 6 节） |
| 定时任务在测试中途突然执行 | 调度未关闭 | 测试 profile 关 `@EnableScheduling`，业务方法手动触发 |
| 时间相关断言月底/年底必挂 | 生产代码 `LocalDate.now()` | 注入 `Clock`，测试用 fixed/mutable Clock |
| `@TransactionalEventListener` 没触发 | 发布者无事务，默认跳过 | 发布包进事务或 `fallbackExecution = true` |
| 事件数量不一致 | 同步监听器异常中断发布链 / 记录器跨方法污染 | 检查监听器异常处理；确认 `@RecordApplicationEvents` 每方法重置 |
| future.get() 挂死测试 | 裸 get 无超时 | 一律 `get(timeout, unit)` |

# 5. 小结

| 主题 | 关键结论 |
|------|---------|
| `@Async` | 逻辑测试同步化（`SyncTaskExecutor`），并发测试保留线程池 + Awaitility |
| 等待手段 | Awaitility > latch > `future.get(timeout)`；禁用 sleep 与裸 get |
| 定时任务 | 调度与业务拆分，业务方法直测；测试 profile 关闭调度器 |
| 时间 | 生产代码注入 `Clock`；测试用 `Clock.fixed` / Spring 6.1+ `MutableClock` |
| 事件发布侧 | `@RecordApplicationEvents` + `ApplicationEvents` 流式断言 |
| 事件消费侧 | 监听器直接 new 出来单测；每个事件留一条全链路冒烟 |
| AFTER_COMMIT | 无事务默认不执行；测试要么带事务提交路径，要么 fallbackExecution |

下一篇进入可观测性与日志：测试中控制日志级别、用 `OutputCaptureExtension` 断言输出、验证 Micrometer 指标。
