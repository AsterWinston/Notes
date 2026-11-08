# 1. 日志与输出捕获

日志在测试里有两个身份：**排错时的第一现场**（上下文加载失败、Bean 冲突、SQL 打印都靠日志还原）与**被断言的对象本身**（"这个操作必须留下审计日志"是业务契约的一部分）。前者靠调级别，后者靠捕获扩展——两件事的工具和纪律完全不同。

## 1.1 OutputCaptureExtension

Spring Boot 提供输出捕获扩展，把测试期间写到 `System.out` / `System.err` 的内容收进 `CapturedOutput`，用于断言日志或控制台输出：

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.system.CapturedOutput;
import org.springframework.boot.test.system.OutputCaptureExtension;

@ExtendWith(OutputCaptureExtension.class)     // 来自 spring-boot-test，JUnit 5 扩展
class LogTest {

    @Test
    void logShouldContain(CapturedOutput output) {
        service.processOrder(order);                       // 被测代码打日志

        assertThat(output).contains("order accepted")      // CapturedOutput 自带流式断言
                          .contains("id=1001");
        assertThat(output.getOut()).contains("started");   // 也可取原始字符串
        assertThat(output.getAll()).containsIgnoringCase("WARN");  // out + err 合并视图
    }
}
```

工作原理与限制：

- 扩展在测试方法前用 `System.setOut/setErr` 换成代理流，方法后还原；`CapturedOutput` 参数注入的是**本方法执行至今**的累积内容；
- **默认日志框架会写到控制台**（Logback ConsoleAppender），所以业务日志能被捕获——但如果 `logback-test.xml` 把日志只写文件，`OutputCaptureExtension` 就抓不到；
- 捕获是**线程不安全**的：并行测试（JUnit parallel）下另一个线程的输出可能混进来；异步日志 appender（`AsyncAppender`）的输出可能在断言之后才落——两种情况都会 flaky；
- 它捕获不了 SLF4J 的 `Marker`、级别、Logger 名等结构化信息，只是纯文本。要断言"以 WARN 级别、在哪个 Logger 上打了什么"，用 Logback 的 `ListAppender` 挂到目标 Logger 上更精确：

```java
class AuditLogTest {

    private ListAppender<ILoggingEvent> appender;

    @BeforeEach
    void attachAppender() {
        Logger logger = (Logger) LoggerFactory.getLogger(AuditService.class);
        appender = new ListAppender<>();                       // Logback 内存 appender
        appender.start();
        logger.addAppender(appender);                          // 只挂目标类，不影响全局
    }

    @Test
    void should_log_warn_when_retry() {
        auditService.retry(failedTask);

        assertThat(appender.list)
                .anySatisfy(event -> {
                    assertThat(event.getLevel()).isEqualTo(Level.WARN);      // 级别可断言
                    assertThat(event.getFormattedMessage()).contains("retry");
                });
    }
}
```

> **设计争议**：断言日志算不算过度测试？日志措辞一改测试就红，看起来脆弱。共识是**分级对待**：普通调试日志不断言；**审计日志、合规日志、告警日志属于业务契约**，必须断言（内容关键字 + 级别），这是监管行业的硬性要求。断言时选稳定的业务标识（订单号、错误码）而不是整句话，能大幅降低脆性。

## 1.2 日志配置覆盖

在 `src/test/resources` 中配置 `logback-test.xml`（Logback 对它的加载优先于 `logback.xml`）或 `log4j2-test.xml`：

```xml
<!-- src/test/resources/logback-test.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="org.springframework" level="INFO"/>   <!-- 框架日志压到 INFO -->
    <logger name="org.hibernate.SQL" level="DEBUG"/>    <!-- 打开 SQL 打印 -->
    <logger name="com.example" level="DEBUG"/>          <!-- 业务日志放开 -->

    <root level="WARN">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

轻量方案是直接在测试配置里用属性（不必写 XML）：

```yaml
# src/test/resources/application-test.yml
logging:
  level:
    root: WARN                        # 全局压低，测试输出干净
    com.example: DEBUG                # 业务代码放开
    org.hibernate.SQL: DEBUG          # 看生成的 SQL
    org.hibernate.orm.jdbc.bind: TRACE  # 看绑定参数（Boot 3.x 新 Logger 名）
```

Boot 3.x 的一个变化值得记住：Hibernate 6 把"打印绑定参数"从 `org.hibernate.type.descriptor.sql` 挪到了 `org.hibernate.orm.jdbc.bind`，老项目升级后"看不到 SQL 参数"多半是 Logger 名过期。

---

# 2. 诊断与排错工具

测试排错的核心思路：**让框架自己说出它为什么这么做**。Spring Test/Boot 在关键路径上埋了 DEBUG 日志，打开就有答案。

## 2.1 提升上下文日志

```properties
# 上下文加载/缓存决策过程
logging.level.org.springframework.test.context=DEBUG
# 会打印：缓存命中/未命中、MergedContextConfiguration 内容、为什么新建容器
logging.level.org.springframework.boot.test=DEBUG
```

`org.springframework.test.context.cache` 的 DEBUG 日志会输出每个测试类使用的缓存 Key 与命中情况——排查"为什么起了 8 个容器"（第 17 篇主题）时这是唯一权威依据。

## 2.2 观察 Bean 加载

```properties
logging.level.org.springframework.beans=DEBUG          # Bean 注册与依赖解析细节
logging.level.org.springframework.context.annotation=DEBUG  # 条件评估、配置类解析
```

配合两个更重型的工具：

- **`-Ddebug`（或 `debug=true` 属性）**：输出自动配置条件评估报告（ConditionEvaluationReport）——每个 AutoConfiguration 为什么匹配/为什么没匹配、缺哪个类、哪个属性。"为什么这个 Bean 没加载"的标准答案在这里；
- **`/actuator/conditions` 端点**：同样内容的 JSON 版，`RANDOM_PORT` 集成测试里可直接 HTTP 拉取。

```text
 排错日志组合速查
┌────────────────────────────────┬──────────────────────────────────┐
│ 症状                            │ 打开的日志                        │
├────────────────────────────────┼──────────────────────────────────┤
│ 上下文加载失败 / Bean 缺失        │ org.springframework.beans=DEBUG │
│                                 │ + -Ddebug 条件报告                │
│ 容器数量异常多 / 套件慢           │ org.springframework.test.context│
│                                 │ .cache=DEBUG                     │
│ 属性值不符合预期                 │ /actuator/env 或 Environment 遍历 │
│ SQL 没执行 / 参数不对             │ org.hibernate.SQL=DEBUG          │
│                                 │ org.hibernate.orm.jdbc.bind=TRACE│
│ 事务行为异常                     │ org.springframework.orm=DEBUG    │
│                                 │ org.springframework.transaction  │
└────────────────────────────────┴──────────────────────────────────┘
```

---

# 3. Actuator 在测试中的使用

Actuator 端点在测试里既是**被测对象**（健康检查、指标端点本身要测）也是**观测工具**（通过端点反查容器内部状态）。

## 3.1 启用端点

```properties
# 测试配置：暴露需要的端点（生产暴露清单另行收敛）
management.endpoints.web.exposure.include=health,info,metrics,env,conditions
```

`MOCK` 环境的 `@SpringBootTest` + `@AutoConfigureMockMvc` 或 `@WebMvcTest` 里，Actuator 端点走 MockMvc；`RANDOM_PORT` 下走真实 HTTP。端点的自动配置在切片测试里**默认不加载**，需要 `@ImportAutoConfiguration` 或全量测试。

## 3.2 通过 MockMvc 验证

```java
@SpringBootTest
@AutoConfigureMockMvc
class HealthEndpointTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void health_should_report_up() throws Exception {
        mockMvc.perform(get("/actuator/health"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("UP"))
            .andExpect(jsonPath("$.components.db.status").value("UP"));  // 细到组件级
    }
}
```

健康检查的测试价值在 CI 语境被低估了：**自定义 `HealthIndicator` 写错（异常没捕获、依赖没判空）会让生产环境的探活直接翻车**，而它在测试里一行就能验。对 `DOWN` 路径同样要测：

```java
@Test
void health_should_report_down_when_queue_unreachable() throws Exception {
    given(queueHealthIndicator.health()).willReturn(Health.down().build());

    mockMvc.perform(get("/actuator/health"))
        .andExpect(status().isServiceUnavailable())     // DOWN → 503（Web 暴露默认行为）
        .andExpect(jsonPath("$.status").value("DOWN"));
}
```

---

# 4. 观测与指标

## 4.1 `@AutoConfigureObservability`

Boot 3.0 起，**测试默认关闭 Micrometer 的指标导出**（`ObservationAutoConfiguration` 仍在，但 exporter 被测试属性压制），避免测试往真实监控系统打点。需要在测试里验证指标时，用 Boot 3.0+ 的注解显式打开：

```java
@SpringBootTest
@AutoConfigureObservability      // 恢复完整的观测栈：metrics + tracing
class MetricsExportTest { }
```

它等价于打开一组测试默认关掉的属性（`management.defaults.metrics.export.enabled`、tracing 采样等）。只想要指标不想要 tracing 时，可用 `@AutoConfigureObservability(tracing = false)`。

## 4.2 验证指标被记录

验证指标不需要真的 Prometheus——直接问容器里的 `MeterRegistry`：

```java
@SpringBootTest
class OrderMetricsTest {

    @Autowired
    private MeterRegistry meterRegistry;      // 测试上下文里是 SimpleMeterRegistry 或复合注册表

    @Autowired
    private OrderService orderService;

    @Test
    void should_record_order_created_counter() {
        orderService.create(cart);            // 业务代码里 meterRegistry.counter(...).increment()

        Counter counter = meterRegistry.find("orders.created")   // find：查不到返回 null 不抛
                .tag("channel", "web")
                .counter();
        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(1.0);

        // Timer 型指标：断言次数与耗时分布
        Timer timer = meterRegistry.find("orders.checkout").timer();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isPositive();
    }
}
```

要点与坑：

- **`find(...)` 与 `get(...)` 的区别**：`get` 查不到直接抛 `MeterNotFoundException`（适合"必须存在"的断言），`find` 返回 null（适合先判空再断言值）；
- **注册表状态跨测试累积**：共享上下文时 `MeterRegistry` 也共享，计数器不会因方法结束归零——断言用**增量**（记录调用前后差值）或在测试配置里换独立的 `SimpleMeterRegistry`；
- 指标名与 tag 拼错不会报错，只是永远查不到——断言 `isNotNull()` 是防拼错的第一道闸；
- **注意指标采集异步化带来的时序问题**：走 `Observation` API 的指标在 observation stop 时记录，如果业务在异步线程里开 observation，断言前需要等它完成（第 15 篇的 Awaitility 用得上）。

## 4.3 测试 trace 传播

微服务链路追踪（Micrometer Tracing + Brave/OTel）在测试里的验证目标是：**跨线程/跨服务调用时 traceId 是否正确传播**。测试环境的做法：

```yaml
# application-test.yml：全采样 + 不导出到真实 collector
management:
  tracing:
    sampling:
      probability: 1.0        # 测试里 100% 采样，保证每个请求都有 span
```

```java
@SpringBootTest
@AutoConfigureObservability
class TracePropagationTest {

    @Autowired
    private Tracer tracer;                       // Micrometer Tracer 门面

    @Test
    void should_keep_trace_across_async_call() {
        Span span = tracer.nextSpan().name("test-op").start();
        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            asyncService.process(task);            // 内部跨线程
            await().untilAsserted(() ->
                assertThat(capturedTraceIds()).contains(span.context().traceId()));
        } finally {
            span.end();
        }
    }
}
```

Boot 3.x 的异步传播依赖 `TaskDecorator`（`ContextPropagatingTaskDecorator`）把 trace 上下文带过线程池边界——**这正是"生产配置对了、测试同步化 Executor 反而测不到"的典型**：验证 trace 传播的测试必须用真实线程池，不能用第 15 篇的 `SyncTaskExecutor`。

---

# 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 日志缺失 | 日志级别过高 / root 是 WARN | 调低对应包的 `logging.level`，确认 logger 名没写错 |
| 输出断言不稳定 | 异步 appender 或并行测试串流 | 强制同步 appender、类级 `@Isolated`，或改 `ListAppender` 精确挂载 |
| SQL 参数看不到 | Hibernate 6 换了 Logger 名 | `org.hibernate.orm.jdbc.bind=TRACE` |
| 指标断言为 null | 名字/tag 拼错，或测试默认关导出 | `find().isNotNull()` 排查；必要时 `@AutoConfigureObservability` |
| 计数值比预期大 | 共享上下文注册表累积 | 断言增量或用独立 `SimpleMeterRegistry` |
| 日志把 CI 输出刷爆 | root=DEBUG 全开 | 只在排错时临时开；常态 root=WARN + 按包放开 |
| 审计日志断言过不了但功能正常 | 只捕获了 out，日志走了 err 或文件 | 用 `output.getAll()`，或检查 appender 指向 |

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 输出捕获 | `OutputCaptureExtension` 抓纯文本；要级别/Logger 名用 Logback `ListAppender` |
| 断言边界 | 只断言审计/合规/告警类日志，选业务标识不选整句措辞 |
| 日志配置 | `logback-test.xml` 或测试 profile 的 `logging.level`；Hibernate 6 参数 Logger 是 `org.hibernate.orm.jdbc.bind` |
| 排错日志 | `test.context.cache=DEBUG` 查容器数量；`-Ddebug` 查自动配置评估 |
| Actuator | 健康端点 UP/DOWN 两路都测；切片里端点默认不加载 |
| 指标 | 测试默认关导出，`@AutoConfigureObservability` 打开；断言直接问 `MeterRegistry`，注意共享累积 |
| trace | 传播测试必须用真实线程池，采样率 1.0 |

下一篇进入性能优化：上下文缓存最大化、分层执行、并行配置与启动耗时分析。
