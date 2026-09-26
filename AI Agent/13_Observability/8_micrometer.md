# 1. 为什么用 Micrometer

**结论先行**：Micrometer 是 JVM 生态的指标门面（Facade），应用只写一次埋点代码，就能把指标导出到 Prometheus、Datadog、CloudWatch 等多种后端。它同时是 Spring Boot Actuator 与 Spring AI Observation 的底层实现。

## 1.1 门面模式的价值

```
应用代码 → Micrometer API → Prometheus / Datadog / CloudWatch ...
```

换后端不改代码，与 Spring AI 的 Observation 天然集成。若直接依赖某个后端的 SDK，切换后端时需要重写全部埋点，且容易把业务代码与厂商耦合。

| 方式 | 切换成本 | 耦合度 | 生态 |
|------|:--------:|:------:|------|
| 直连后端 SDK | 高 | 高 | 单一 |
| Micrometer 门面 | 低 | 低 | 多后端 |

## 1.2 与 Observation 的关系

Micrometer 有两层：`MeterRegistry`（指标）与 `ObservationRegistry`（观测）。Observation 是更上层的抽象，一次观测同时产生指标、Trace 与日志，Micrometer 指标是其中一种输出。

```
Observation（观测）
 ├── Metrics   → MeterRegistry
 ├── Tracing   → Tracer（OTel/Brave）
 └── Logging   → Handler
```

## 1.3 指标类型总览

| 类型 | 语义 | 典型用途 |
|------|------|----------|
| Counter | 单调递增 | 请求数、错误数 |
| Gauge | 瞬时值 | 队列长度、连接数 |
| Timer | 耗时分布 | 延迟 |
| DistributionSummary | 任意分布 | Token 数、载荷大小 |

选错类型会导致查询困难，例如用 Gauge 统计请求数会丢失累计语义。

# 2. 依赖与配置

## 2.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Actuator 提供端点与自动配置，`micrometer-registry-prometheus` 提供 Prometheus 格式的导出器。若用 OTLP 后端，则替换为 `micrometer-registry-otlp`。

## 2.2 配置

### 2.2.1 端点暴露

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ai-app
  observations:
    annotations:
      enabled: true
```

### 2.2.2 公共标签

`management.metrics.tags` 下的标签会附加到所有指标，常用于标识应用名、环境、区域。注意公共标签也会计入基数，不要放高基数值。

### 2.2.3 直方图与百分位

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        ai.chat.duration: true
      slo:
        ai.chat.duration: 100ms,500ms,1s
```

百分位直方图（Percentile Histogram）让 Prometheus 侧可计算任意分位，代价是每个序列多出若干桶（bucket）。

# 3. 核心 API

## 3.1 MeterRegistry

`MeterRegistry` 是注册表的入口，负责创建与持有所有指标。注入它即可在任意组件埋点。指标按名字与标签唯一确定，重复创建会返回同一实例。

## 3.2 Counter

```java
Counter.builder("ai.chat.requests")
        .tag("model", model)
        .tag("status", success ? "success" : "error")
        .register(registry)
        .increment();
```

计数器只增不减，适合请求数、错误数、Token 总量。

## 3.3 Timer

```java
registry.timer("ai.chat.duration", "model", model)
        .record(durationMs, TimeUnit.MILLISECONDS);
```

Timer 同时记录次数、总耗时与分布，可算出 QPS 与平均延迟。

## 3.4 Gauge 与 DistributionSummary

```java
// Gauge：必须持有被观测对象的强引用，否则会被 GC
Queue<?> queue = taskQueue;
Gauge.builder("agent.queue.size", queue, Queue::size)
        .register(registry);

// DistributionSummary：记录任意数值分布
DistributionSummary.builder("agent.tokens.per.task")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(registry)
        .record(tokens);
```

## 3.5 完整示例

```java
@Service
@RequiredArgsConstructor
public class LlmMetricsService {

    private final MeterRegistry registry;

    public void recordChat(String model, long durationMs, Usage usage,
                           boolean success) {
        // 计数
        registry.counter("ai.chat.requests",
                "model", model,
                "status", success ? "success" : "error").increment();

        // 耗时
        registry.timer("ai.chat.duration", "model", model)
                .record(durationMs, TimeUnit.MILLISECONDS);

        // Token
        if (usage != null) {
            registry.summary("ai.tokens.input", "model", model)
                    .record(usage.getPromptTokens());
            registry.summary("ai.tokens.output", "model", model)
                    .record(usage.getCompletionTokens());
        }
    }
}
```

# 4. Spring AI 的 Observation

Spring AI 自动为模型调用创建 Observation：

```
observation.name: spring.ai.chat.client
keyValues:
  spring.ai.chat.client.operation = chat
  spring.ai.chat.client.request.model = gpt-4o-mini
```

## 4.1 Observation 生命周期

一次 Observation 经历 `start` → 业务执行 → `stop`，在 stop 时把耗时与关键值写入指标与 Trace。异常会被记录为事件并标记为错误。

## 4.2 自动生成的指标

### 4.2.1 指标命名

| 指标 | 含义 |
|------|------|
| `spring.ai.chat.client` | 调用次数与耗时 |
| `gen_ai.client.token.usage` | Token 使用量 |

具体名称随 Spring AI 版本变化，接入前应先在 `/actuator/metrics` 确认实际名称。

### 4.2.2 关键值

关键值（KeyValue）分为低基数（Low Cardinality，可进指标标签）与高基数（High Cardinality，仅进 Trace）。模型名、操作名属于低基数；提示内容、用户 ID 属于高基数。

## 4.3 启用与配置

启用后自动生成指标与 Trace：

```yaml
management:
  observations:
    annotations:
      enabled: true
```

还需保证存在 `ObservationRegistry` 与至少一个指标导出器，否则观测数据无处落地。

# 5. 自定义 Observation

## 5.1 创建与低基数键

```java
@Component
public class RagObservation {

    private final ObservationRegistry registry;

    public RagObservation(ObservationRegistry registry) {
        this.registry = registry;
    }

    public List<Document> retrieveWithObservation(String query) {
        return Observation.createNotStarted("rag.retrieval", registry)
                .lowCardinalityKeyValue("tenant", TenantContext.current())
                .observe(() -> {
                    List<Document> docs = retriever.retrieve(query);
                    return docs;
                });
    }
}
```

`observe` 会自动处理开始、结束与异常，是推荐写法。

## 5.2 覆盖 RAG 与工具

### 5.2.1 RAG 检索

在检索外层加 Observation，记录命中数与耗时，便于区分"检索慢"还是"生成慢"。

### 5.2.2 工具调用

```java
// 伪代码：为工具调用加观测
return Observation.createNotStarted("agent.tool", registry)
        .lowCardinalityKeyValue("tool", toolName)
        .observe(() -> tool.execute(args));
```

工具名是低基数（数量有限），但参数与结果不要作为标签。

## 5.3 异常与事件

```java
Observation observation = Observation.start("rag.retrieval", registry);
try (Observation.Scope scope = observation.openScope()) {
    // 业务
} catch (Exception e) {
    observation.error(e);
    throw e;
} finally {
    observation.stop();
}
```

手动管理时务必在 `finally` 中 stop，否则会泄漏并产生错误的耗时。

# 6. 自定义健康检查

## 6.1 HealthIndicator

```java
@Component
public class LlmHealthIndicator implements HealthIndicator {

    private final ChatModel model;

    @Override
    public Health health() {
        try {
            // 轻量检查：只验证配置与连通性，不真调模型
            if (model == null) {
                return Health.down().withDetail("reason", "模型未配置").build();
            }
            return Health.up()
                    .withDetail("model", modelName())
                    .build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

> 健康检查**不要调用昂贵的模型接口**，否则探针会拖垮系统。

## 6.2 探针分组

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        readiness:
          include: readinessState,llm
```

存活探针（Liveness）判断进程是否要重启；就绪探针（Readiness）判断是否能接流量。模型不可用通常应影响就绪而非存活。

## 6.3 边界与坑

| 坑 | 后果 | 规避 |
|----|------|------|
| 调真实模型 | 探针变慢、费用上升 | 只查配置 |
| 无超时 | 探针挂起 | 加超时 |
| 缓存健康结果过久 | 状态滞后 | 合理缓存窗口 |

# 7. 常用端点

## 7.1 端点清单

| 端点 | 用途 |
|------|------|
| `/actuator/health` | 健康检查 |
| `/actuator/health/liveness` | 存活探针 |
| `/actuator/health/readiness` | 就绪探针 |
| `/actuator/metrics` | 指标列表 |
| `/actuator/prometheus` | Prometheus 格式 |
| `/actuator/info` | 应用信息 |

## 7.2 安全暴露

```yaml
management:
  server:
    port: 9090
  endpoints:
    web:
      base-path: /internal
```

把管理端点放在独立端口与内网，避免通过公网暴露 `/actuator`。敏感端点（如 `env`、`heapdump`）绝不对外开放。

# 8. 自定义业务指标

## 8.1 计数器

用于累计成功与失败的任务数，是计算成功率的基础。

## 8.2 分布摘要

用于刻画每任务 Token 的分布，关注 P95/P99 而非均值。

## 8.3 完整示例

```java
@Component
public class BusinessMetrics {

    private final Counter tasksCompleted;
    private final Counter tasksFailed;
    private final DistributionSummary tokensPerTask;

    public BusinessMetrics(MeterRegistry registry) {
        this.tasksCompleted = Counter.builder("agent.tasks.completed")
                .description("成功完成的任务数")
                .register(registry);

        this.tasksFailed = Counter.builder("agent.tasks.failed")
                .description("失败的任务数")
                .register(registry);

        this.tokensPerTask = DistributionSummary.builder("agent.tokens.per.task")
                .description("每个任务消耗的 Token")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);
    }

    public void onTaskCompleted(long tokens) {
        tasksCompleted.increment();
        tokensPerTask.record(tokens);
    }

    public void onTaskFailed() {
        tasksFailed.increment();
    }
}
```

在构造器中创建并持有指标，避免每次调用都 builder，减少开销。

# 9. Prometheus 抓取

## 9.1 抓取配置

```yaml
scrape_configs:
  - job_name: 'ai-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['ai-app:8080']
```

## 9.2 与告警联动

PromQL 可直接消费 Micrometer 指标，例如错误率：

```yaml
- alert: HighLlmErrorRate
  expr: |
    sum(rate(ai_chat_requests_total{status="error"}[5m]))
      / sum(rate(ai_chat_requests_total[5m])) > 0.05
  for: 10m
```

# 10. 生产注意事项

## 10.1 基数控制

| 标签 | 基数 | 是否可用 |
|------|:----:|:--------:|
| model | 低 | 可用 |
| status | 低 | 可用 |
| tenant | 中 | 谨慎 |
| userId | 高 | 禁用 |
| traceId | 极高 | 禁用 |

高基数标签会让时间序列数量爆炸，最终拖垮 Prometheus。

## 10.2 性能开销

- 复用已创建的 Counter/Timer，避免热路径反复 builder。
- 百分位直方图会增加内存与存储，按需开启。
- 每新增一个标签维度，序列数成倍增长。

## 10.3 告警

| 风险 | 建议 |
|------|------|
| 健康检查调模型 | 只查配置/连通性 |
| 高基数标签 | 用低基数 |
| 指标过多 | 聚焦核心 |
| 端点暴露过多 | 只暴露必要端点 |
| 无 Prometheus | 接入抓取 |
| 无告警 | 配合 AlertManager |

# 11. 小结

- Micrometer 是指标门面，一次埋点多后端。
- 指标类型要选对：Counter、Gauge、Timer、DistributionSummary。
- Spring AI 的 Observation 自动生成指标与 Trace。
- 自定义 Observation 可覆盖 RAG、工具等环节。
- 健康检查要轻量，不调模型。
- 控制标签基数是稳定运行的前提。
- 配合 Prometheus + AlertManager 实现监控告警。
