# 1. 为什么超时是第一要务

结论先行：没有超时（Timeout）的外部调用会让执行线程被无限期占用，最终耗尽线程池，导致整个服务不可用。超时是可靠性里最基础、成本最低的一道防线。

```
没有超时 = 线程被无限占用 = 线程池耗尽 = 整个服务不可用
```

Agent 一次请求可能调用多个工具（Tool），任一工具卡住都会拖垮整个链路。

> 超时是可靠性的第一道防线，比重试（Retry）、熔断（Circuit Breaker）更基础。

## 1.1 无超时的连锁反应

### 1.1.1 线程耗尽的机制

结论先行：一个没有超时的阻塞调用，会永久占住一个线程；当并发请求数超过线程池容量，新请求只能排队，随后排队也满，服务整体不可用。

机制拆解：

1. 请求进入，占用线程 T1，向外部服务发起阻塞调用。
2. 外部服务变慢或假死，调用既不返回也不报错。
3. T1 被无限期占用，无法回到线程池。
4. 后续请求陆续占满剩余线程。
5. 线程池队列填满，新请求被拒绝（Rejected）。
6. 服务对外表现为整体超时，而非某个工具失败。

```
时间轴（线程池大小 = 3）

t0  T1: 调外部服务 ──────────────── 永不返回
t1  T2: 调外部服务 ──────────────── 永不返回
t2  T3: 调外部服务 ──────────────── 永不返回
t3  请求 4/5/6 ... 排队 → 队列满 → 拒绝
```

### 1.1.2 雪崩与级联失败

结论先行：单点慢调用会沿着调用链向上扩散，形成级联失败（Cascading Failure）。

- 上游服务的线程同样被占满，故障向上游传导。
- 健康检查（Health Check）超时，实例被摘除，剩余实例压力更大。
- 重试流量叠加，被拖慢的服务彻底崩溃。

> 超时把"无限等待"变成"有限等待"，是阻断雪崩的第一道闸门。

## 1.2 超时与重试、熔断的关系

结论先行：三者职责不同、必须配合。超时负责"及时止损"，重试负责"处理抖动"，熔断负责"处理持续故障"。

| 机制 | 解决的问题 | 触发条件 | 失效后果 |
|------|-----------|---------|---------|
| 超时 | 单次调用无限等待 | 超过时间阈值 | 线程耗尽 |
| 重试 | 临时抖动 | 可重试错误 | 放大故障 |
| 熔断 | 持续故障 | 失败率超阈值 | 资源空转 |
| 限流 | 过载 | 并发/速率超限 | 雪崩 |

顺序上通常为：熔断在最外层，超时贴近实际调用。

## 1.3 超时的成本权衡

结论先行：超时值太小会误杀正常慢请求，太大则失去保护意义，需要按接口分布设定并持续校准。

| 超时值 | 风险 |
|--------|------|
| 过小 | 正常请求被误判失败，重试放大流量 |
| 过大 | 慢调用长时间占线程，保护失效 |
| 一刀切 | 快慢接口混用同一阈值，必有一方受损 |

生产建议：以 P99.9 耗时为基础，乘以 1.5~2 倍作为初始值，再结合容量压测校准。

# 2. 超时的分层

结论先行：一次请求的总预算必须按层拆分，且下层超时严格小于上层，否则上层先超时、下层工作白做。

```
用户请求（总预算 30s）
├── Agent 循环（25s）
│   ├── 模型调用（10s）
│   ├── 工具 A（5s）
│   ├── 工具 B（3s）
│   └── 检索（2s）
└── 其他开销（5s）
```

总预算 ≥ 各层之和，且下层必须小于上层，否则上层超时先触发，下层白做。

## 2.1 超时预算的层次结构

### 2.1.1 层次结构示意

结论先行：从用户请求到最底层网络连接，形成一棵倒置的超时树，越往下阈值越小。

```
用户请求总预算 30s
└── Agent 循环 25s
    ├── 模型调用 10s
    │   └── 连接建立 2s
    ├── 工具 A 5s
    │   ├── 连接 1s
    │   └── 读取 4s
    └── 检索 2s
```

### 2.1.2 层级倒挂的后果

结论先行：当下层超时大于上层时，上层会先触发超时并放弃结果，下层的等待完全浪费，还占用资源。

| 配置 | 结果 |
|------|------|
| 上层 3s，下层 5s | 上层 3s 先超时，下层继续跑 2s 才被丢弃 |
| 上层 10s，下层 3s | 合理，下层先失败并可重试 |
| 上层等于下层 | 边界竞态，结果不确定 |

## 2.2 各层超时建议值

结论先行：不同类型调用的建议区间不同，需按业务容忍度与下游能力设定。

| 层级 | 建议超时 |
|------|----------|
| 用户请求总预算 | 30~60s |
| 单次模型调用 | 10~30s（流式另算） |
| 单个工具调用 | 3~10s |
| 检索 | 1~3s |
| 连接建立 | 1~3s |

## 2.3 流式与长连接的分层差异

结论先行：流式（Streaming）响应不能套用固定总超时，应使用空闲超时（Idle Timeout）配合整体上限。

| 场景 | 主用超时 | 辅助超时 |
|------|---------|---------|
| 普通请求 | 总超时 | 连接超时 |
| 流式响应 | 空闲超时 | 首字节超时 + 总上限 |
| WebSocket | 心跳超时 | 空闲超时 |

# 3. 超时的三种类型

结论先行：一次网络调用至少涉及连接超时、读取超时与总超时三类，只设其一会留下保护缺口。

| 类型 | 含义 | 设置 |
|------|------|------|
| 连接超时 | 建立连接的时间 | 1~3s |
| 读取超时 | 等待响应的时间 | 按接口 |
| 总超时 | 整个调用的时间 | 连接 + 读取 + 重试 |

## 3.1 连接超时

结论先行：连接超时（Connect Timeout）限制 TCP 握手（含 TLS 握手）的时间，防止目标不可达时长时间挂起。

- 覆盖 DNS 解析、TCP 三次握手、TLS 握手。
- 建议 1~3s；跨地域可适当放宽。
- 连接超时通常可重试（属于可重试错误）。

## 3.2 读取超时

结论先行：读取超时（Read Timeout）限制"连接建立后等待数据"的时间，是防慢响应最关键的一层。

- 指两次数据到达之间的最大间隔，而非总耗时。
- 长任务接口需要单独放大或改用异步回调。
- 读取超时同样通常可重试。

## 3.3 总超时

结论先行：总超时（Total Timeout）限制从发起到结束的整体时间，包含连接、读取与内部重试，是最终兜底。

- 必须大于等于"连接 + 读取"之和。
- 若内部含重试，需把重试时间计入预算。
- 总超时触发时，底层调用应被真正取消。

## 3.4 三者在 Java HttpClient 中的落地

结论先行：`java.net.http.HttpClient` 用 `connectTimeout` 管连接、用 `HttpRequest.timeout` 管总超时，读取超时则通过总超时与响应处理间接约束。

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.time.Duration;

// 连接超时：整个 client 级别
HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(2))       // 连接超时
        .build();

// 总超时：单个请求级别
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/tool"))
        .timeout(Duration.ofSeconds(5))              // 请求总超时
        .GET()
        .build();
```

边界与坑：

- `HttpRequest.timeout` 是总超时，超时后抛出 `HttpTimeoutException`。
- 该客户端没有独立的读取超时，读超时靠总超时覆盖。
- 若需精细读取超时，改用 OkHttp / Apache HttpClient 的 `readTimeout`。

生产建议：连接超时设为 1~3s，总超时按接口设定；两者都设，避免只设其一。

# 4. Java 实现超时的方式

结论先行：Java 侧有四种常见做法——`CompletableFuture`、Resilience4j `TimeLimiter`、Spring `@TimeLimiter`、JDK 21 结构化并发，按场景选择。

| 方案 | 依赖 | 适用 |
|------|------|------|
| CompletableFuture | JDK 内置 | 通用、轻量 |
| Resilience4j TimeLimiter | Resilience4j | 需统一治理 |
| Spring @TimeLimiter | WebFlux | 响应式链路 |
| 结构化并发 | JDK 21+ | 虚拟线程场景 |

## 4.1 CompletableFuture

### 4.1.1 基础用法

结论先行：`CompletableFuture.get(timeout)` 是最简单的超时手段，但只让等待方超时，不中断后台任务。

```java
import java.time.Duration;
import java.util.concurrent.*;
import java.util.function.Supplier;

public <T> T withTimeout(Duration timeout, Supplier<T> action) {
    try {
        return CompletableFuture.supplyAsync(action)
                .get(timeout.toMillis(), TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        throw new ToolTimeoutException("工具执行超时: " + timeout);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

### 4.1.2 orTimeout 与 completeOnTimeout

结论先行：JDK 9+ 提供 `orTimeout`（超时异常）与 `completeOnTimeout`（超时兜底值），写法更声明式。

```java
// 超时抛 TimeoutException（包装为 CompletionException）
CompletableFuture<String> f1 = CompletableFuture
        .supplyAsync(() -> tool.execute(args))
        .orTimeout(5, TimeUnit.SECONDS);

// 超时返回默认值，不抛异常
CompletableFuture<String> f2 = CompletableFuture
        .supplyAsync(() -> tool.execute(args))
        .completeOnTimeout("工具超时，请稍后重试", 5, TimeUnit.SECONDS);
```

边界与坑：这两个方法都基于 `CompletableFuture` 的延迟调度，仍不会中断底层阻塞调用。

## 4.2 Resilience4j TimeLimiter

结论先行：`TimeLimiter` 与线程池、熔断、重试可组合，适合需要统一治理策略的生产系统。

```java
import io.github.resilience4j.timelimiter.TimeLimiter;
import java.time.Duration;
import java.util.concurrent.*;

TimeLimiter timeLimiter = TimeLimiter.of(Duration.ofSeconds(5));

public String callTool(String args) {
    Callable<String> callable = () -> tool.execute(args);
    try {
        return timeLimiter.executeFutureSupplier(
                () -> CompletableFuture.supplyAsync(callable));
    } catch (TimeoutException e) {
        return "工具超时，请稍后重试";
    }
}
```

## 4.3 Spring @TimeLimiter（WebFlux）

结论先行：在响应式链路中，用注解声明超时，方法必须返回 `CompletableFuture` 或 `Mono`。

```java
import org.springframework.resilience.annotation.TimeLimiter;
import java.util.concurrent.CompletableFuture;

@TimeLimiter(name = "toolA")
public CompletableFuture<String> callToolA(String args) {
    return CompletableFuture.supplyAsync(() -> toolA.execute(args));
}
```

# 5. 工具级超时封装

结论先行：把超时逻辑封装到统一的工具执行器（Tool Executor）中，避免每个工具各写一套，便于统一监控与治理。

## 5.1 封装设计

结论先行：执行器需要同时持有线程池、超时策略、结果封装与指标埋点四要素。

| 要素 | 职责 |
|------|------|
| 线程池 | 隔离工具调用，防止拖垮主线程 |
| 超时策略 | 每工具可配不同阈值 |
| 结果封装 | 统一成功/失败结构 |
| 指标埋点 | 耗时、超时计数 |

## 5.2 完整实现

结论先行：下面是一个可直接落地的最小实现，包含超时、异常归类与耗时统计。

```java
import java.time.Duration;
import java.util.concurrent.*;

@Component
public class ToolExecutor {

    private final ExecutorService pool = Executors.newFixedThreadPool(20);

    public ToolResult execute(ToolCall call, Duration timeout) {
        long start = System.currentTimeMillis();

        try {
            String result = CompletableFuture
                    .supplyAsync(() -> doExecute(call), pool)
                    .get(timeout.toMillis(), TimeUnit.MILLISECONDS);

            return ToolResult.ok(result, System.currentTimeMillis() - start);

        } catch (TimeoutException e) {
            return ToolResult.fail(
                    "工具执行超时（%ds）".formatted(timeout.toSeconds()));
        } catch (Exception e) {
            return ToolResult.fail("工具执行失败: " + e.getMessage());
        }
    }

    private String doExecute(ToolCall call) {
        return switch (call.name()) {
            case "weather" -> weatherTool.query(call.arguments());
            case "order"   -> orderTool.query(call.arguments());
            default        -> throw new IllegalArgumentException(
                    "未知工具: " + call.name());
        };
    }
}
```

## 5.3 中断与取消语义

结论先行：`CompletableFuture.get(timeout)` 只让等待方超时，后台任务可能仍在跑，对外部调用必须配合底层 HTTP 超时确保真正中断。

```
调用方 ──get(5s)──▶ 等待 ──5s 到──▶ 抛 TimeoutException
                                    │
后台线程 ──仍可能继续执行─────────────┘   ← 资源未释放
```

| 层面 | 是否中断 | 说明 |
|------|---------|------|
| `get(timeout)` | 否 | 仅等待方放弃 |
| `future.cancel(true)` | 尽力 | 依赖任务响应中断 |
| 底层 HTTP 超时 | 是 | 真正关闭连接 |

# 6. 超时后的处理

结论先行：超时不是终点，而是一个需要被"翻译"的信号，应根据工具性质选择返回错误、重试、降级或中断。

## 6.1 四种处理策略

结论先行：不同工具对超时的容忍度不同，策略需按"是否只读、是否有兜底"选择。

| 处理 | 说明 | 适用 |
|------|------|------|
| 返回错误给模型 | 让模型换策略 | 只读工具 |
| 重试 | 幂等 + 临时抖动 | 只读工具 |
| 降级 | 返回缓存/默认值 | 有兜底 |
| 中断任务 | 无法继续 | 关键路径 |

## 6.2 结构化错误返回

结论先行：把超时转换成模型能理解的结构化错误，而不是抛异常中断整个 Agent。

```java
@Tool("查询订单")
ToolResult queryOrder(@P("订单号") String orderId) {
    try {
        return ToolResult.ok(orderService.getWithTimeout(
                orderId, Duration.ofSeconds(3)));
    } catch (ToolTimeoutException e) {
        return ToolResult.fail("查询超时，请稍后重试或换用其他方式");
    }
}
```

> 超时返回结构化错误，让模型能自我调整，而不是抛异常中断整个 Agent。

## 6.3 与重试、幂等的衔接

结论先行：超时错误是否重试，取决于工具是否幂等；写操作重试必须携带同一幂等键（详见 `2_tool_retry.md`、`3_idempotency.md`）。

| 工具类型 | 超时后动作 |
|---------|-----------|
| 只读 | 可重试 2~3 次 |
| 幂等写 | 可重试，复用幂等键 |
| 非幂等写 | 不可盲目重试 |

# 7. 流式调用的超时

结论先行：流式响应不能简单用总超时，需用空闲超时，只要持续有 token 返回就不算超时。

## 7.1 空闲超时的必要性

结论先行：一次大模型流式输出可能持续数十秒，用固定总超时会误杀正常响应。

```
固定总超时：  [=====输出中=====|超时]  ← 误杀
空闲超时：    [=token=token=token=...]  ← 正常
                    ↑ 每次间隔 < 10s
```

## 7.2 实现

结论先行：响应式链路用 `timeout(Duration)` 限定两次 token 之间的最大间隔。

```java
import reactor.core.publisher.Flux;
import java.time.Duration;

Flux<String> stream = chatClient.prompt()
        .user(q)
        .stream()
        .content()
        .timeout(Duration.ofSeconds(10));       // 两次 token 之间的最大间隔
```

# 8. 超时预算传递

结论先行：把剩余时间预算（Deadline）沿调用链向下传递，每层只使用剩余额度，避免"前面用光、后面没得用"。

## 8.1 预算对象设计

结论先行：预算对象记录绝对截止时间，并对外暴露"剩余时间"与"按比例分配"两个能力。

```java
import java.time.Duration;

public class TimeoutBudget {
    private final long deadline;

    public TimeoutBudget(Duration total) {
        this.deadline = System.currentTimeMillis() + total.toMillis();
    }

    public Duration remaining() {
        long left = deadline - System.currentTimeMillis();
        if (left <= 0) throw new ToolTimeoutException("预算已耗尽");
        return Duration.ofMillis(left);
    }

    public Duration allocate(double ratio) {
        return Duration.ofMillis((long) (remaining().toMillis() * ratio));
    }
}
```

> 把剩余预算按比例分配给后续步骤，避免"前面用光了，后面没得用"。

## 8.2 分配策略

结论先行：为关键步骤预留比例，非关键步骤用剩余额度，防止前序步骤吃满预算。

| 步骤 | 分配比例 | 说明 |
|------|---------|------|
| 检索 | 10% | 快，失败可降级 |
| 工具 | 40% | 核心，需保障 |
| 模型 | 50% | 耗时最大 |

# 9. 监控

结论先行：没有监控的超时配置无法校准，必须采集耗时分布与超时计数并设置告警。

## 9.1 指标设计

结论先行：至少采集三类指标：调用耗时、超时计数、超时率。

```java
import io.micrometer.core.instrument.MeterRegistry;

registry.timer("tool.duration", "tool", name).record(duration);
registry.counter("tool.timeouts", "tool", name).increment();
```

| 指标 | 类型 | 用途 |
|------|------|------|
| `tool.duration` | Timer | 耗时分布、P99 |
| `tool.timeouts` | Counter | 超时次数 |
| `tool.timeout.rate` | Gauge | 超时率 |

## 9.2 告警规则

结论先行：告警应基于超时率与耗时分位数，而非绝对次数。

| 指标 | 告警条件 |
|------|---------|
| `tool.timeouts` | 突增告警 |
| `tool.duration` P99 | 超阈值告警 |
| 超时率 | > 1% 告警 |

生产建议：把超时率纳入 SLO，超时率持续超阈时自动触发容量评估。

# 10. 生产注意事项

结论先行：超时问题的根因往往不是"没设超时"，而是"设了但设错"，需要按清单逐项检查。

## 10.1 常见风险清单

| 风险 | 建议 |
|------|------|
| 无超时 | 所有外部调用必设 |
| 超时层级倒挂 | 下层 < 上层 |
| 只设总超时 | 连接/读取/总都设 |
| 超时未中断 | 配合底层客户端超时 |
| 超时抛异常 | 返回结构化错误 |
| 无监控 | 监控超时率与耗时 |

## 10.2 上线检查清单

结论先行：上线前逐项确认，可覆盖绝大多数超时事故。

- 每个外部调用是否都有超时。
- 连接、读取、总超时是否都设置。
- 超时层级是否自下而上递增。
- 超时后是否真正中断底层调用。
- 超时是否返回结构化错误而非抛异常。
- 是否采集耗时与超时指标。
- 是否配置超时率告警。

# 11. 小结

## 11.1 要点回顾

- 超时是可靠性第一道防线，比重试熔断更基础。
- 超时必须分层，下层小于上层。
- 区分连接超时、读取超时、总超时。
- 超时后返回结构化错误，让模型能调整。

## 11.2 落地行动

- 用超时预算避免"前面用光、后面没得用"。
- 用监控持续校准超时阈值，形成闭环。
