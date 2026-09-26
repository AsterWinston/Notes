# 1. 观测总览

## 1.1 观测能力清单

结论先行：AI 应用的不确定性远高于普通服务，没有可观测性就无法定位"是模型问题、检索问题还是编排问题"。

```
Spring AI Alibaba 提供：
├── Trace（全链路）
├── 指标（Token、延迟、成本）
├── 日志（结构化）
└── 与阿里云 ARMS 集成
```

| 能力 | 回答的问题 |
|------|------------|
| Trace | 这一步慢在哪、错在哪 |
| 指标 | 整体健康度与趋势 |
| 日志 | 具体发生了什么 |
| 成本 | 钱花在哪 |

## 1.2 三大支柱（Trace/Metrics/Logs）

| 支柱 | 粒度 | 用途 | 代价 |
|------|------|------|------|
| Trace | 单请求 | 定位瓶颈 | 存储/采样 |
| Metrics | 聚合 | 监控告警 | 基数 |
| Logs | 事件 | 复盘取证 | 体积/脱敏 |

机制：三者互补——指标发现异常，Trace 定位环节，日志还原细节。

## 1.3 与 Spring AI 的关系

```
Spring AI 的 Observation（Micrometer）
        +
Spring AI Alibaba 的增强
  ├── Graph 节点级 Trace
  ├── 阿里云可观测集成
  └── 国内平台适配
```

- Spring AI 用 Micrometer Observation 统一埋点；
- Spring AI Alibaba 在其上补充 Graph 节点级 Span 与云平台对接；
- 边界：增强部分依赖框架实现，升级时需回归验证。

## 1.4 观测成熟度模型

| 阶段 | 特征 | 目标 |
|------|------|------|
| L0 | 无观测 | 先补日志 |
| L1 | 有指标 | 能告警 |
| L2 | 有 Trace | 能定位 |
| L3 | 成本可见 | 能控费 |
| L4 | 闭环优化 | 能自愈 |

- 结论先行：多数团队卡在 L1→L2，缺的是 Trace 而非指标；
- 生产建议：按阶段补齐，不要一次性追求 L4。

# 2. 启用观测

## 2.1 依赖引入

结论先行：观测能力由 Spring Boot Actuator + Micrometer Tracing 提供，需显式引入依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

## 2.2 配置项

### 2.2.1 Observation 配置

```yaml
management:
  observations:
    annotations:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  tracing:
    sampling:
      probability: 0.1
```

### 2.2.2 Tracing 采样

| 采样值 | 含义 | 适用 |
|:------:|------|------|
| 0.0 | 全不采 | 关闭 |
| 0.1 | 10% | 生产常规 |
| 1.0 | 全采 | 排障/低流量 |

- 采样越低，开销越小，但漏掉的长尾越多；
- 建议"低比例 + 错误全采"组合。

## 2.3 验证与边界

- 访问 `/actuator/metrics`，确认出现 `ai.*` 指标；
- 触发一次调用，确认 Trace 出现在后端；
- 边界：`annotations.enabled` 为 false 时 `@Observed` 不生效；
- 生产建议：把观测配置纳入配置中心，便于动态调整采样。

## 2.4 观测开销与取舍

机制：埋点本身有成本——CPU、内存、网络上报与存储。

| 开销项 | 影响因素 |
|--------|----------|
| 采样率 | 越高越贵 |
| Span 数量 | 节点越多越贵 |
| 属性大小 | Prompt 越长越贵 |
| 上报频率 | 越频繁越贵 |

- 边界：过度埋点会拖慢主链路，需做压测；
- 生产建议：观测配置分级，生产环境默认低开销。

# 3. 全链路 Trace

## 3.1 Trace 模型

结论先行：一次用户请求对应一个 Trace，内部每个处理步骤是一个 Span。

```
Trace（一次请求）
 ├── Span: HTTP 接入
 ├── Span: Graph 执行
 │    ├── Span: classify 节点
 │    └── Span: generate 节点
 │         └── Span: LLM 调用
 └── Span: 响应
```

### 3.1.1 Span 常用属性

| 属性 | 说明 |
|------|------|
| model | 使用的模型 |
| input.tokens | 输入 Token |
| output.tokens | 输出 Token |
| node | 图节点名 |
| tenant | 租户（低基数） |

- 属性应"可检索、低基数"，避免把整段 Prompt 塞进去。

## 3.2 Graph 节点级 Trace

### 3.2.1 自动 Span

机制：Graph 执行时为每个节点自动创建 Span，无需手写埋点。

```
Trace: 用户请求
├── Span: classify 节点
├── Span: retrieve 节点
│   └── Span: 向量检索
├── Span: generate 节点
│   └── Span: LLM 调用
└── Span: 完成
```

```java
// Graph 执行时自动为每个节点创建 Span
graph.stream(state).subscribe(event ->
        log.info("节点 {} 耗时 {}ms",
                event.node(), event.durationMs()));
```

### 3.2.2 手动 Span

当框架未覆盖某段逻辑时，用 `Observation` 手动埋点。

```java
Observation.createNotStarted("custom.retrieve", registry)
        .observe(() -> retriever.retrieve(question));
```

- 手动 Span 命名用点分层级（`custom.retrieve`）便于聚合；
- 边界：手动埋点过多会稀释信噪比，只埋关键路径。

### 3.2.3 异步与流式上下文传播

机制：Trace 上下文存放在 ThreadLocal，跨线程/流式时必须显式传递，否则 Span 断裂。

```java
// 伪代码：把上下文带到异步任务
try (Scope scope = currentSpan.makeCurrent()) {
    return Mono.fromCallable(() -> callModel(prompt))
            .subscribeOn(Schedulers.boundedElastic());
}
```

- 边界：`subscribeOn` 切换线程后若未传播上下文，子 Span 会挂到根上；
- 生产建议：对关键异步链路写专项测试，断言父子关系正确。

## 3.3 边界与坑

- 边界：异步/流式执行若未传递上下文，Span 会断裂；
- 边界：Span 属性写入敏感数据会泄露；
- 生产建议：给节点 Span 统一加 `tenant`、`model` 等低基数标签。

# 4. 关键指标

## 4.1 指标清单

结论先行：AI 应用的指标要同时覆盖"资源、模型、业务"三层。

| 指标 | 说明 |
|------|------|
| `ai.tokens.input` | 输入 Token |
| `ai.tokens.output` | 输出 Token |
| `ai.cost` | 累计成本 |
| `ai.chat.duration` | 调用耗时 |
| `graph.node.duration` | 节点耗时 |
| `graph.executions` | 图执行次数 |

### 4.1.1 指标命名规范

- 前缀统一（`ai.` / `graph.`），便于聚合；
- 单位后缀（`.duration`、`.count`、`.bytes`）；
- 边界：命名一旦上线不宜频繁改名，否则历史曲线断裂。

## 4.2 指标采集

### 4.2.1 计数器

```java
registry.counter("ai.cost", "model", model, "tenant", tenant)
        .increment(cost);
```

### 4.2.2 计时器

```java
registry.timer("graph.node.duration", "node", nodeName)
        .record(duration);
```

- 计数器适合累计量（Token、成本）；
- 计时器适合耗时分布，可算 P50/P95/P99。

### 4.2.3 直方图与分位数

机制：计时器底层是直方图，分位数由桶（bucket）估算，桶配置不当会让 P99 失真。

| 桶配置 | 影响 |
|--------|------|
| 过粗 | 分位数不准 |
| 过细 | 存储膨胀 |
| 默认 | 通用但不贴合 |

- 生产建议：按业务延迟分布自定义桶，并定期校准。

## 4.3 高基数问题

| 标签 | 基数 | 建议 |
|------|------|------|
| model | 低 | 可加 |
| node | 低 | 可加 |
| tenant | 中 | 评估上限 |
| userId | 高 | 禁止 |
| query | 极高 | 禁止 |

- 高基数标签会让时序库膨胀甚至拖垮；
- 生产建议：把高基信息放进 Trace/日志，而非指标标签。

# 5. 日志

## 5.1 结构化日志

- 用 JSON 结构化输出，便于检索与聚合；
- 关键字段：traceId、node、duration、status；
- 边界：多行堆栈需合并为单事件，避免切分。

## 5.2 节点日志

```java
@Slf4j
public class GraphLoggingListener {

    public void onNodeComplete(String node, AgentState state, long durationMs) {
        log.info("graph.node node={} duration={}ms state={}",
                node, durationMs, summarize(state));
    }
}
```

> 日志必须脱敏，尤其含 Prompt 与用户数据时。

### 5.2.1 日志与 Trace 关联

机制：在日志中带上 `traceId`，即可从一条日志跳转到完整调用链。

```
log.info("graph.node traceId={} node={} duration={}ms",
        currentTraceId(), node, durationMs);
```

- 边界：日志框架与 Trace 上下文需正确桥接，否则 traceId 为空；
- 生产建议：把 traceId 作为日志的固定字段，便于全文检索。

## 5.3 脱敏

| 数据 | 处理 |
|------|------|
| 手机号/身份证 | 掩码 |
| 密钥/Token | 不落日志 |
| 用户问题 | 按合规决定 |
| 模型原始输出 | 截断 |

- 脱敏应在日志框架层统一做，避免遗漏；
- 生产建议：对脱敏规则做单元测试。

## 5.4 边界与生产建议

- 边界：日志量随调用量线性增长，需设置保留期与采样；
- 边界：`summarize(state)` 若序列化整个状态，体积与泄露风险都大；
- 生产建议：只记录状态摘要与长度，不记录原文。

# 6. 成本监控

## 6.1 成本模型

结论先行：成本 = 输入 Token 单价 × 输入量 + 输出 Token 单价 × 输出量，按模型档位不同。

```
成本 = Σ (inTokens × priceIn + outTokens × priceOut)
```

| 档位 | 输入单价 | 输出单价 |
|------|:--------:|:--------:|
| turbo | 低 | 低 |
| plus | 中 | 中 |
| max | 高 | 高 |

### 6.1.1 定价配置化

- 单价会变，必须做成配置而非硬编码；
- 不同模型、不同区域单价不同，需按维度建表；
- 边界：缓存与批量折扣会扭曲简单乘法，成本模型要能扩展。

## 6.2 成本追踪节点

```java
public class CostTrackingNode implements GraphNode {

    @Override
    public Map<String, Object> apply(AgentState state) {
        Usage usage = lastUsage(state);
        BigDecimal cost = pricing.cost(usage);

        costStore.record(state.getTenantId(), cost);
        return Map.of();
    }
}
```

### 6.2.1 用量（Usage）提取

机制：`Usage` 来自模型响应，含输入/输出 Token；流式场景需累加增量。

```java
// 伪代码：从响应或流中提取用量
Usage usage = response.getMetadata().getUsage();
long in = usage.getPromptTokens();
long out = usage.getCompletionTokens();
```

- 边界：部分兼容接口不返回 Usage，需降级为估算；
- 生产建议：把"实测 Token"与"估算 Token"分开统计，避免对账偏差。

## 6.3 按租户统计

- 成本必须按租户/用户维度归集，才能计费与限流；
- 边界：流式响应下 Usage 可能最后才返回，需在流结束时结算；
- 生产建议：成本数据落库，支持对账与趋势分析。

## 6.4 边界与坑

- 边界：缓存命中的请求不应重复计费，需区分；
- 边界：失败重试会重复消耗 Token，成本监控要能识别；
- 生产建议：设置日/月成本预算与告警阈值。

# 7. 阿里云 ARMS 集成

## 7.1 接入配置

```yaml
arms:
  license-key: ${ARMS_LICENSE_KEY}
  app-name: ai-app
```

## 7.2 平台能力

| 能力 | 说明 |
|------|------|
| 应用监控 | 调用链、慢请求 |
| 大模型监控 | Token、成本、质量 |
| 告警 | 阈值告警 |

- 应用监控提供分布式调用链与慢请求分析；
- 大模型监控补充 Token、成本与质量指标；
- 边界：云平台能力与自建 Prometheus 可能重叠，需统一口径。

## 7.3 边界与坑

- 边界：License Key 属敏感信息，必须外置；
- 边界：云平台 Agent 会增加少量开销；
- 生产建议：先小流量验证上报正确，再全量开启。

## 7.4 自建可观测栈对比

| 维度 | 阿里云 ARMS | 自建（Prometheus + Jaeger） |
|------|-------------|------------------------------|
| 运维 | 免 | 高 |
| 集成 | 与云产品打通 | 需自接 |
| 成本 | 按量 | 固定 + 人力 |
| 可控 | 低 | 高 |

- 边界：自建栈的存储与保留策略需自行设计；
- 生产建议：指标用 Prometheus，Trace 用云平台，二者口径对齐。

# 8. 告警与 SLO

## 8.1 告警规则

结论先行：告警要针对"用户可感知的问题"，而非所有指标。

| 规则 | 阈值示例 |
|------|----------|
| 错误率 | > 5% 持续 5 分钟 |
| P95 延迟 | > 3s |
| 成本 | 日预算 80% |
| 配额 | 使用率 > 90% |

### 8.1.1 告警分级

| 级别 | 触发 | 响应 |
|------|------|------|
| P0 | 全量不可用 | 立即 |
| P1 | 部分失败 | 15 分钟 |
| P2 | 成本异常 | 当日 |
| P3 | 趋势劣化 | 排期 |

- 边界：分级不清会导致所有告警都变 P0，最终被忽视；
- 生产建议：每级告警绑定明确的接收人与升级路径。

## 8.2 SLO 设计

| SLI | SLO |
|-----|-----|
| 可用性 | 99.9% |
| 延迟 | P95 < 3s |
| 成功率 | > 99% |

- SLO 是目标，告警是手段，二者要联动；
- 边界：模型侧波动会导致 SLO 抖动，需与供应商状态联动。

## 8.3 边界与坑

- 边界：告警风暴会淹没真正的问题，需做聚合与抑制；
- 边界：阈值拍脑袋会误报，应基于历史分布设定；
- 生产建议：每个告警都应有明确的处置手册。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 无 Trace | 无法定位 |
| 日志泄露 | 脱敏 |
| 指标高基数 | 低基数标签 |
| 全量采样 | 比例 + 错误全采 |
| 无成本监控 | 按租户统计 |
| 无告警 | 关键指标告警 |

## 9.2 采样策略

- 常规请求低比例采样，降低成本；
- 错误请求与慢请求 100% 采样；
- 边界：采样后指标与 Trace 口径可能不一致，需在文档中说明。

## 9.3 数据安全

- Trace 与日志中的 Prompt、用户数据必须脱敏；
- 观测数据出境需评估合规；
- 生产建议：观测数据设置保留期，到期自动清理。

## 9.4 上线检查清单

| 检查项 | 通过标准 |
|--------|----------|
| Actuator | 指标端点可用 |
| Trace | 端到端链路完整 |
| 采样 | 比例 + 错误全采 |
| 脱敏 | 日志无敏感信息 |
| 成本 | 按租户可统计 |
| 告警 | 关键指标已配置 |

- 边界：清单需随架构演进更新；
- 生产建议：把观测就绪作为发布门禁的一部分。

# 10. 小结

## 10.1 核心要点

- Spring AI Alibaba 复用 Spring AI 的 Observation。
- 增强：Graph 节点级 Trace、阿里云集成。
- 关键指标：Token、成本、节点耗时。
- 可与 ARMS 集成实现应用监控。

## 10.2 落地清单

- 日志脱敏、指标低基数、按租户统计成本。
- 采样用"比例 + 错误全采"，并配置关键告警。
