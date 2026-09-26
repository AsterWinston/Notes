# 1. OpenTelemetry 概述

## 1.1 是什么

### 1.1.1 定位与架构

结论先行：OpenTelemetry（简称 OTel，开放遥测）是可观测性的开放标准，统一 Trace、Metrics、Logs 的采集与导出。

它提供与厂商无关的 API（应用埋点接口）、SDK（实现）与协议（OTLP），让应用只埋一次点，就能把数据送到任意后端。

```
应用（OTel SDK）
   |
   v
OTel Collector（接收、处理、导出）
   |
   +--> Jaeger / Tempo（Trace）
   +--> Prometheus（Metrics）
   +--> Loki / ELK（Logs）
```

机制拆解：应用通过 OTLP（OpenTelemetry Protocol，开放遥测协议）把数据推给 Collector；Collector 用流水线（pipeline）做批处理、属性增删，再导出到不同后端。优势：**厂商中立，换后端不用改代码**。

## 1.2 为什么需要

### 1.2.1 厂商中立与统一标准

结论先行：不用为每个后端写一套埋点，避免被单一厂商锁定；统一的数据模型与语义约定使现成看板、告警、采样策略可复用。

## 1.3 边界

结论先行：OTel 是"采集与传输"标准，不负责存储与展示，后端仍需自行部署。

# 2. 语义约定

## 2.1 GenAI 语义约定

### 2.1.1 属性清单

结论先行：OTel 为 LLM 定义了标准属性名（GenAI 语义约定），遵循它才能复用现成工具。

| 属性 | 说明 |
|------|------|
| `gen_ai.system` | 模型厂商（openai/anthropic） |
| `gen_ai.request.model` | 请求模型 |
| `gen_ai.response.model` | 响应模型 |
| `gen_ai.usage.input_tokens` | 输入 Token |
| `gen_ai.usage.output_tokens` | 输出 Token |
| `gen_ai.request.temperature` | 温度 |
| `gen_ai.request.max_tokens` | 最大 Token |
| `gen_ai.response.finish_reasons` | 结束原因 |
| `gen_ai.operation.name` | 操作类型（chat/embeddings） |

### 2.1.2 代码

```java
span.setAttribute("gen_ai.system", "openai");
span.setAttribute("gen_ai.request.model", "gpt-4o-mini");
span.setAttribute("gen_ai.usage.input_tokens", 850);
span.setAttribute("gen_ai.usage.output_tokens", 120);
```

> 遵循语义约定，才能复用现成的看板与告警规则。

## 2.2 命名与版本

### 2.2.1 稳定性与演进

结论先行：语义约定分稳定、实验等级别，生产应优先使用稳定属性；属性名可能随规范升级变化，需锁定 SDK 版本并关注变更日志。

## 2.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 自造属性名 | 无法复用看板 | 遵循约定 |
| 属性名拼写错 | 数据对不上 | 常量集中管理 |
| 升级 SDK 不读日志 | 属性名失效 | 关注变更 |

生产建议：把 GenAI 属性名封装成常量类，避免散落各处拼写错误。

# 3. Java 接入

## 3.1 依赖

### 3.1.1 Maven 依赖

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

### 3.1.2 BOM 版本管理

结论先行：用 BOM（Bill of Materials，物料清单）统一管理 OTel 各模块版本，避免版本错配。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-bom</artifactId>
            <version>1.40.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 3.2 配置

### 3.2.1 应用配置与环境变量

结论先行：OTel 支持 YAML 与环境变量两种配置，容器化部署优先用环境变量注入 endpoint。

```yaml
otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  traces:
    sampler: parentbased_traceidratio
    sampler-arg: "0.1"
  resource:
    attributes:
      service.name: ai-app
      deployment.environment: prod
```

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: ai-app
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://otel-collector:4317
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"
```

## 3.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 只加 api 不加 sdk | 无数据导出 | 补齐 sdk/exporter |
| 端口混用 | 导出失败 | gRPC 4317 / HTTP 4318 |
| 未设 service.name | 后端不可辨识 | 必设 |

生产建议：优先用环境变量注入 endpoint，便于多环境部署。

# 4. 自动埋点

## 4.1 覆盖范围

### 4.1.1 组件清单与 Java Agent

结论先行：OTel 提供大量自动埋点，通过 Java Agent（字节码增强）可零代码接入。

| 组件 | 埋点内容 |
|------|----------|
| Spring Web | HTTP 请求 |
| WebClient / RestTemplate | 出站 HTTP |
| JDBC | 数据库 |
| Kafka | 消息 |
| Redis | 缓存 |

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=ai-app \
     -jar app.jar
```

## 4.2 手动埋点

### 4.2.1 注解与 API 方式

结论先行：Spring Boot Starter 提供注解式埋点，代码最简；注解无法覆盖的场景用 Tracer API 手动创建 Span。

```java
// 手动创建 Span
@WithSpan("llm.chat")
public String chat(@SpanAttribute("llm.model") String model,
                   String prompt) {
    return modelClient.call(model, prompt);
}
```

## 4.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| Agent 与 SDK 混用冲突 | 数据重复 | 二选一 |
| 注解不生效 | 方法非代理调用 | 检查代理 |
| 自动埋点遗漏 LLM 调用 | 无 GenAI 属性 | 手动补充 |

生产建议：先用 Agent 拿到基础链路，再对 LLM 调用手动补充 GenAI 语义属性。

# 5. 自定义 Span

## 5.1 Span 生命周期

### 5.1.1 创建/激活/结束

结论先行：Span 必须先创建、激活（makeCurrent），业务结束后务必 end；属性（Attribute）是键值对，事件（Event）是时间点标记。

```
spanBuilder --> startSpan --> makeCurrent --> 业务 --> end
```

## 5.2 完整实现

### 5.2.1 Tracer 与 Meter

结论先行：Tracer 用于创建 Span，Meter 用于创建指标，均从 OpenTelemetry 实例获取。

### 5.2.2 异常记录

结论先行：异常要同时 setStatus(ERROR) 与 recordException，才能被后端正确识别。

```java
@Component
public class OTelLlmTracer {

    private final Tracer tracer;
    private final Meter meter;

    public OTelLlmTracer(OpenTelemetry otel) {
        this.tracer = otel.getTracer("ai-app");
        this.meter = otel.getMeter("ai-app");
    }

    public <T> T traceChat(String model, Supplier<T> action) {
        Span span = tracer.spanBuilder("chat " + model)
                .setAttribute("gen_ai.operation.name", "chat")
                .setAttribute("gen_ai.system", "openai")
                .setAttribute("gen_ai.request.model", model)
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            T result = action.get();
            span.setStatus(StatusCode.OK);
            return result;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }

    public void recordTokens(long input, long output) {
        meter.counterBuilder("gen_ai.client.token.usage")
                .setUnit("{token}")
                .build()
                .add(input + output,
                     Attributes.of(
                             AttributeKey.stringKey("gen_ai.token.type"), "total"));
    }
}
```

## 5.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 忘记 end | Span 泄漏 | try/finally |
| 未 makeCurrent | 子 Span 断链 | 用 Scope |
| 属性含敏感数据 | 泄漏 | 脱敏 / Collector 删除 |
| 属性过多 | 后端臃肿 | 只留必要 |

生产建议：把 Span 生命周期封装成模板方法或装饰器，避免遗漏 end。

# 6. 上下文传播

## 6.1 传播机制

### 6.1.1 W3C traceparent

结论先行：OTel 默认用 W3C Trace Context 标准的 `traceparent` 头在服务间传播上下文；入站自动提取、出站自动注入，跨服务链路才能连成一条。

```
服务 A --traceparent--> 服务 B --traceparent--> 服务 C
```

## 6.2 异步与线程

### 6.2.1 Context 传递

结论先行：Context 也是 ThreadLocal 存储，异步任务需显式包裹；可对线程池做包装让任务自动携带当前 Context。

```java
public CompletableFuture<String> asyncCall(String input) {
    Context context = Context.current();

    return CompletableFuture.supplyAsync(
            context.wrap(() -> doCall(input)), executor);
}
```

## 6.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 异步未包裹 | 链路断裂 | context.wrap |
| 跨进程头被网关丢弃 | 断链 | 网关透传 |
| 自定义协议未注入 | 断链 | 手动注入 |

生产建议：在网关与消息中间件上确认 traceparent 透传，否则链路会断在边界。

# 7. 与 Spring Boot 集成

## 7.1 Micrometer Tracing

### 7.1.1 桥接与依赖

结论先行：Spring Boot 3 内置 Micrometer Tracing，可桥接到 OTel，复用 Spring 生态。

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

## 7.2 配置

### 7.2.1 采样与 OTLP 导出

结论先行：通过 `management.tracing.sampling.probability` 控制采样比例，并配置 OTLP 导出端点。

```yaml
management:
  tracing:
    sampling:
      probability: 0.1
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
  metrics:
    export:
      otlp:
        enabled: true
```

## 7.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 只桥接不导出 | 无数据 | 配 OTLP |
| 采样率 1.0 | 成本高 | 按需下调 |
| 指标与链路端不一致 | 数据分叉 | 统一 endpoint |

生产建议：trace 与 metrics 走同一个 Collector，便于统一处理。

# 8. Collector 配置

## 8.1 架构

### 8.1.1 Receiver/Processor/Exporter

结论先行：Collector 由接收器、处理器、导出器三段组成，通过流水线串联；每种信号（traces/metrics/logs）各有一条流水线，可独立配置。

## 8.2 配置示例

### 8.2.1 完整配置

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  batch: {}
  attributes:
    actions:
      - key: user.email
        action: delete          # 删除敏感属性

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

### 8.2.2 属性删除

结论先行：敏感属性在 Collector 统一删除，比在每个应用里脱敏更可靠。

| 动作 | 用途 |
|------|------|
| delete | 删除敏感属性 |
| hash | 哈希化标识 |
| truncate | 截断长值 |

## 8.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 未配 batch | 请求过多 | 必开批处理 |
| 敏感属性未删 | 泄漏 | attributes 处理器 |
| 单点 Collector | 丢数据 | 集群部署 |
| 无内存限制 | OOM | memory_limiter |

生产建议：流水线顺序固定为 memory_limiter、batch、attributes，防止过载。

# 9. 采样策略

## 9.1 采样类型

### 9.1.1 头部与尾部

结论先行：头部采样（Head Sampling）在链路起点决策，成本低但无法按结果决定；尾部采样（Tail Sampling）在链路结束后按错误/耗时决定，需在 Collector 完成。

| 类型 | 决策时机 | 优点 | 缺点 |
|------|----------|------|------|
| 头部 | 起点 | 简单、省资源 | 无法保留错误 |
| 尾部 | 结束 | 可保错误/慢链路 | 需缓冲全部数据 |

## 9.2 自定义采样

### 9.2.1 Sampler 与关键路径必采

结论先行：对 LLM 调用、工具调用、错误路径强制采样，其余按比例。

```java
// 头部采样
Sampler.parentBased(Sampler.traceIdRatioBased(0.1));

// 自定义：错误与慢请求必采
public class LlmSampler implements Sampler {

    @Override
    public SamplingResult shouldSample(Context parentContext, String traceId,
                                       String name, SpanKind kind,
                                       Attributes attributes, List<LinkData> links) {
        // LLM 相关 Span 提高采样率
        if (name.startsWith("chat ") || name.startsWith("tool.")) {
            return SamplingResult.recordAndSample();
        }
        return SamplingResult.drop();
    }
}
```

## 9.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 全量采样 | 成本爆炸 | 比例 + 关键必采 |
| 采样不一致 | 链路残缺 | parentBased |
| 尾部采样缓冲过大 | Collector OOM | 限内存 + 调参 |

生产建议：先头部比例采样兜底，再用 Collector 尾部采样保留错误与慢链路。

# 10. 生产注意事项

## 10.1 风险清单

### 10.1.1 语义与安全

| 风险 | 建议 |
|------|------|
| 未用语义约定 | 遵循 GenAI 约定 |
| 敏感属性上报 | Collector 中删除 |
| 属性拼写错误 | 常量集中管理 |

### 10.1.2 可靠性

| 风险 | 建议 |
|------|------|
| 全量采样 | 比例 + 关键必采 |
| 上下文丢失 | 显式传递 |
| Collector 单点 | 集群部署 |
| 无资源属性 | 设置 service.name 等 |

## 10.2 落地建议

### 10.2.1 部署与成本

结论先行：Collector 无状态，可水平扩展，用负载均衡前置避免单点；采样率与保留期是成本的两个主要旋钮，按业务价值分级。

## 10.3 检查表

| 检查项 | 通过标准 |
|--------|----------|
| 语义约定 | GenAI 属性 |
| 采样 | 比例 + 必采 |
| 敏感属性 | Collector 删除 |
| 资源属性 | service.name |
| Collector | 集群 + 限流 |

# 11. 小结

## 11.1 核心要点

- OTel 是可观测性的开放标准，厂商中立。
- 遵循 GenAI 语义约定，可复用现成看板。
- Spring Boot 通过 Micrometer Tracing 桥接。
- Collector 可做属性删除等敏感信息处理。
- 采样用"比例 + 关键路径必采"。

## 11.2 落地检查清单

| 项 | 状态 |
|----|------|
| BOM 版本锁定 | 已完成 |
| 语义属性常量 | 已封装 |
| Collector 流水线 | 已配置 |
| 采样策略 | 已上线 |
| 敏感属性删除 | 已生效 |
