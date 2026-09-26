# 1. 指标基础与类型

## 1.1 为什么需要指标

### 1.1.1 指标与日志的分工

结论先行：指标用低成本的聚合数值回答"整体是否健康"，日志用高成本的事件明细回答"到底发生了什么"。

指标（Metrics）在客户端预聚合，存储为时间序列，查询快、成本低；日志保留原始事件，维度高但昂贵。告警、看板、容量规划应基于指标，而非日志。

```
业务请求 --+--> 指标（计数/耗时，聚合） --> 告警/看板
           |
           +--> 日志（明细，采样）      --> 排障证据
```

### 1.1.2 LLM 指标的特殊性

结论先行：LLM 指标除了常规的请求、错误、延迟，还必须把 Token 与成本当作一等指标。

普通 Web 服务的成本与请求数近似线性；LLM 服务一次请求的 Token 消耗可能相差百倍，且直接映射为真金白银。若没有 Token/成本指标，成本失控时无法及时发现。

机制拆解：LLM 调用返回的 `Usage`（用量）对象提供输入/输出 Token，是成本指标的原始数据来源。

## 1.2 四类指标

### 1.2.1 类型定义

| 类型 | 说明 | 例子 |
|------|------|------|
| Counter | 只增不减 | 请求总数、错误数 |
| Gauge | 可增可减 | 活跃连接数、队列长度 |
| Timer | 耗时分布 | 模型调用耗时 |
| DistributionSummary | 数值分布 | Token 数、成本 |

结论先行：Counter 用于累计事件，Gauge 用于瞬时状态，Timer 与 Summary 用于分布统计。

### 1.2.2 Java 代码

```java
// Counter
registry.counter("ai.requests", "model", model).increment();

// Gauge
registry.gauge("ai.active.sessions", sessionRegistry, r -> r.size());

// Timer
Timer.Sample sample = Timer.start(registry);
// ... 调用
sample.stop(registry.timer("ai.chat.duration", "model", model));

// DistributionSummary
registry.summary("ai.tokens.input", "model", model).record(tokens);
```

机制拆解：`Timer.Sample` 记录起始时间，`stop()` 时把耗时写入对应 Timer；`DistributionSummary` 记录任意数值分布，适合 Token 数。

## 1.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 用 Counter 记录可减量 | 语义错误 | 改用 Gauge |
| Timer 记录非耗时 | 单位混乱 | 用 Summary |
| 每次新建 Meter | 开销大 | 复用缓存 |
| 忘记 stop | 丢失耗时 | try/finally |

生产建议：Meter 由 `MeterRegistry` 缓存管理，无需手动缓存，但标签组合必须稳定。

# 2. LLM 应用核心指标

## 2.1 指标清单

### 2.1.1 请求与错误

| 指标 | 类型 | 说明 |
|------|------|------|
| `ai.requests` | Counter | 请求总数 |
| `ai.requests.errors` | Counter | 失败数 |
| `ai.requests.inflight` | Gauge | 在途请求数 |

### 2.1.2 性能与用量

| 指标 | 类型 | 说明 |
|------|------|------|
| `ai.chat.duration` | Timer | 模型调用耗时 |
| `ai.tokens.input` | Summary | 输入 Token |
| `ai.tokens.output` | Summary | 输出 Token |
| `ai.cost` | Counter | 累计成本 |

### 2.1.3 质量与可靠性

| 指标 | 类型 | 说明 |
|------|------|------|
| `ai.tool.calls` | Counter | 工具调用数 |
| `ai.tool.errors` | Counter | 工具错误数 |
| `ai.retrieval.hits` | Summary | 检索命中数 |
| `ai.agent.steps` | Summary | Agent 步数 |
| `ai.guardrail.triggered` | Counter | 护栏触发 |
| `ai.degradation` | Counter | 降级次数 |

结论先行：以上十二个指标构成 LLM 应用的最小可用指标集。

## 2.2 指标命名规范

### 2.2.1 命名约定

结论先行：统一用"域.对象.动作"命名，如 `ai.chat.duration`，避免同义异名。

| 好 | 坏 |
|----|----|
| `ai.chat.duration` | `chatTime` |
| `ai.tokens.input` | `input_tokens` |
| `ai.requests.errors` | `errorCount` |

### 2.2.2 单位

结论先行：单位写进名字或元数据，避免歧义。

| 指标 | 单位 |
|------|------|
| duration | 秒（Prometheus 约定）或毫秒 |
| tokens | 个 |
| cost | 美元/分 |

## 2.3 边界

结论先行：指标粒度不是越细越好，需在"可定位"与"低基数"之间权衡。

生产建议：先定义排查场景，再反推需要的指标与标签，避免堆砌无用指标。

# 3. 标签设计

## 3.1 基数问题

### 3.1.1 什么是基数

结论先行：基数（Cardinality）是标签取值组合的数量，直接决定时序库的内存占用。

`ai.requests{model,status,tenant}` 若有 5 个模型、3 种状态、100 个租户，则基数为 1500 条时间序列。

### 3.1.2 高基数危害

结论先行：把 userId、traceId 作为标签会让时序数据库内存爆炸。

反例：100 万用户 × 每次请求一个 traceId，等价于无限条时间序列，Prometheus 会 OOM 或拒绝写入。

## 3.2 标签选择

### 3.2.1 可用标签

```java
// 低基数标签
registry.counter("ai.requests",
        "model", "gpt-4o-mini",
        "status", "success",
        "tenant", "acme").increment();
```

### 3.2.2 禁用标签

| 可用标签 | 禁用标签 |
|----------|----------|
| model | userId |
| status | sessionId |
| tenant | traceId |
| endpoint | requestId |
| tool.name | 具体参数值 |

> 高基数标签会让时序数据库内存爆炸。

## 3.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 高基数标签 | 内存爆炸 | 用低基数维度 |
| 标签取值不稳定 | 序列碎片化 | 枚举化 |
| 标签过多 | 查询慢 | 只留必要维度 |
| 动态标签名 | 无法聚合 | 固定 schema |

生产建议：为每个指标的标签集建立评审清单，上线前评估基数上界。

# 4. 用 Advisor 采集指标

## 4.1 采集架构

### 4.1.1 埋点位置

结论先行：指标埋点应放在 Advisor 中，与日志埋点分层，避免侵入业务代码。

### 4.1.2 Advisor 机制

结论先行：Advisor 环绕模型调用，天然能拿到耗时、用量与异常。

```
[日志 Advisor] --> [指标 Advisor] --> [模型调用]
      order=0          order=200
```

## 4.2 完整实现

### 4.2.1 成功路径

结论先行：成功时记录耗时、输入/输出 Token 与成本。

### 4.2.2 失败路径

结论先行：失败时记录带 `status=error` 的耗时与错误计数。

### 4.2.3 成本计算

结论先行：按模型单价与 Token 数计算成本并累加。

```java
public class MetricsAdvisor implements CallAdvisor {

    private final MeterRegistry registry;

    public MetricsAdvisor(MeterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        Timer.Sample sample = Timer.start(registry);
        String model = extractModel(request);

        try {
            ChatClientResponse response = chain.nextCall(request);

            sample.stop(registry.timer("ai.chat.duration",
                    "model", model, "status", "success"));

            Usage usage = response.chatResponse().getMetadata().getUsage();
            if (usage != null) {
                registry.summary("ai.tokens.input", "model", model)
                        .record(usage.getPromptTokens());
                registry.summary("ai.tokens.output", "model", model)
                        .record(usage.getCompletionTokens());

                registry.counter("ai.cost", "model", model)
                        .increment(calculateCost(model, usage));
            }
            return response;

        } catch (Exception e) {
            sample.stop(registry.timer("ai.chat.duration",
                    "model", model, "status", "error"));
            registry.counter("ai.requests.errors", "model", model).increment();
            throw e;
        }
    }

    private double calculateCost(String model, Usage usage) {
        double inPrice = pricePerKToken(model, true);
        double outPrice = pricePerKToken(model, false);
        return usage.getPromptTokens() / 1000.0 * inPrice
                + usage.getCompletionTokens() / 1000.0 * outPrice;
    }

    @Override
    public String getName() { return "MetricsAdvisor"; }

    @Override
    public int getOrder() { return 200; }
}
```

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| Usage 为空 | 流式或异常时缺失 | 判空 |
| 单价过期 | 成本算错 | 配置化 + 版本 |
| 标签含模型版本 | 基数上升 | 只留模型名 |
| 采样后计费不准 | 指标是真实全量 | 指标不采样 |

生产建议：成本单价放在配置中心，变更时记录版本；指标计数不采样，采样只针对日志。

# 5. 关键指标定义

## 5.1 成功率

### 5.1.1 定义

结论先行：成功率 = 成功请求数 / 总请求数，分母必须包含超时与降级。

### 5.1.2 实现

```java
// 成功率 = 成功请求 / 总请求
registry.counter("ai.requests", "status", "success").increment();
registry.counter("ai.requests", "status", "error").increment();
```

## 5.2 延迟分位

### 5.2.1 平均值陷阱

结论先行：平均值会掩盖长尾，必须看分位数。

```
不要只看平均值！
平均值 1s 可能掩盖：95% 是 200ms，5% 是 16s
```

### 5.2.2 分位选择

| 指标 | 用途 |
|------|------|
| P50 | 典型体验 |
| P95 | 多数用户上限 |
| P99 | 长尾（告警用） |

### 5.2.3 直方图配置

结论先行：分位数需要直方图桶（bucket），桶边界要覆盖 LLM 的长耗时区间。

```java
Timer.builder("ai.chat.duration")
     .publishPercentiles(0.5, 0.95, 0.99)
     .sla(Duration.ofSeconds(1), Duration.ofSeconds(5),
          Duration.ofSeconds(10), Duration.ofSeconds(30))
     .register(registry);
```

## 5.3 Token 效率

### 5.3.1 输出输入比

```java
// 输出/输入比，异常时说明提示有问题
double ratio = outputTokens / (double) inputTokens;
```

### 5.3.2 异常识别

结论先行：比例骤降可能表示模型空转，骤升可能表示提示注入或失控生成。

## 5.4 成本指标

### 5.4.1 成本模型

结论先行：成本 = 输入 Token × 输入单价 + 输出 Token × 输出单价。

### 5.4.2 归因

| 维度 | 用途 |
|------|------|
| 租户 | 计费/预算 |
| 模型 | 选型优化 |
| 接口 | 定位大户 |

生产建议：按租户设置日/月预算指标，达到阈值触发告警。

# 6. 告警规则

## 6.1 告警设计原则

### 6.1.1 症状与原因

结论先行：优先对"用户可感知的症状"告警，而非底层原因，减少噪声。

### 6.1.2 分级

结论先行：按影响面分级，P1 立即处理，P2 工作时间内处理。

## 6.2 规则示例

### 6.2.1 错误率

结论先行：错误率超阈值并持续一段时间才告警，避免抖动误报。

### 6.2.2 延迟

结论先行：用 P99 而非平均延迟做告警。

### 6.2.3 成本

结论先行：成本类告警用"占预算比例"，而非绝对金额。

| 指标 | 阈值 | 级别 |
|------|------|------|
| 错误率 | > 5% | P1 |
| P99 延迟 | > 10s | P2 |
| 单任务 Token | > 100k | P2 |
| 租户日成本 | > 80% 预算 | P2 |
| 工具错误率 | > 10% | P2 |
| 熔断触发 | 任意 | P2 |
| 队列积压 | > 1000 | P2 |

```yaml
groups:
  - name: ai-app
    rules:
      - alert: HighErrorRate
        expr: rate(ai_requests_errors_total[5m]) / rate(ai_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "AI 服务错误率超过 5%"

      - alert: HighP99Latency
        expr: histogram_quantile(0.99, rate(ai_chat_duration_seconds_bucket[5m])) > 10
        for: 10m
        labels:
          severity: warning
```

## 6.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 无 `for` 持续期 | 抖动误报 | 设置持续时间 |
| 阈值太敏感 | 告警疲劳 | 调优阈值 |
| 无分级 | 优先级混乱 | 按影响面分级 |
| 告警无 Runbook | 不知如何处置 | 附处理手册 |

生产建议：每条告警都要有对应的处置手册（Runbook），否则告警形同虚设。

# 7. 看板设计

## 7.1 分层看板

### 7.1.1 总览层

结论先行：总览层只放 4 到 6 个核心指标，供值班快速判断。

### 7.1.2 明细层

结论先行：明细层按模型、工具、检索、成本、可靠性分块，供专项排查。

## 7.2 布局

### 7.2.1 ASCII 布局

```
+--------------------------------------------+
| 总览：QPS / 错误率 / P99 / 成本             |
+--------------------------------------------+
| 模型：各模型调用量、延迟、Token             |
+--------------------------------------------+
| 工具：调用量、成功率、耗时                  |
+--------------------------------------------+
| 检索：命中数、相似度分布                    |
+--------------------------------------------+
| 成本：按租户/模型/接口                      |
+--------------------------------------------+
| 可靠性：重试、熔断、降级、限流              |
+--------------------------------------------+
```

### 7.2.2 使用规范

结论先行：看板要能回答具体问题，每块图表都应有明确标题与单位。

## 7.3 生产建议

结论先行：为不同角色准备不同看板——值班看总览，研发看明细，财务看成本。

# 8. 生产注意事项

## 8.1 风险清单

### 8.1.1 基数与性能

| 风险 | 建议 |
|------|------|
| 高基数标签 | 用低基数维度 |
| 指标过多 | 聚焦核心 |
| 采集开销大 | 异步 + 批量 |

### 8.1.2 告警质量

| 风险 | 建议 |
|------|------|
| 无告警 | 关键指标都设阈值 |
| 只看平均 | 关注 P95/P99 |
| 无成本指标 | Token/成本一等 |

## 8.2 落地建议

### 8.2.1 指标评审

结论先行：新增指标需评审名称、类型、标签基数与用途。

### 8.2.2 容量规划

结论先行：用历史 Token 与 QPS 指标做容量与预算预测。

## 8.3 检查表

| 检查项 | 通过标准 |
|--------|----------|
| 指标集 | 覆盖十二项 |
| 标签 | 低基数 |
| 分位 | P95/P99 |
| 告警 | 有阈值与 Runbook |
| 看板 | 分层 |

# 9. 小结

## 9.1 核心要点

- 四类指标：Counter、Gauge、Timer、Summary。
- 核心指标：请求、错误、延迟、Token、成本、工具、检索、步数。
- 标签必须低基数，避免时序库爆炸。
- 延迟关注 P95/P99，不只看平均。
- 关键指标必须配置告警。

## 9.2 落地检查清单

| 项 | 状态 |
|----|------|
| 指标命名规范 | 已定义 |
| 基数评审 | 已执行 |
| 成本单价配置 | 已接入 |
| 告警 Runbook | 已编写 |
| 分层看板 | 已上线 |
