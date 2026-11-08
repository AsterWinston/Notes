
# 1. 概述

SkyWalking 日志集成的核心主张是：**日志不应该只是一堆带时间戳的文本，而应该天然挂在 Trace 上**。只要每条日志携带 TraceId，"从一条慢 Trace 直接跳到它执行期间打印的日志"就成了可观测性里最常用的一次跳转。整体机制分两条路线，用途完全不同：

```text
路线 A（关联查询）：Agent 注入 TraceId → 日志落本地/ELK → 用 TraceId 反查
  Logger.info("处理订单")
    │
    ├── %tid / MDC["traceId"]  ← SkyWalking toolkit 自动注入
    ▼
  2024-01-15 10:30:00.123 [TID:abc123def456] INFO 处理订单

路线 B（上报存储）：gRPC Appender → OAP → 日志进 SkyWalking 存储 → UI 内直接查
  GRPCLogClientAppender ──gRPC 11800──→ Log Receiver → LAL 分析 → ES/BanyanDB
```

两条路线**不互斥且建议共存**：路线 A 零成本、不改变现有 ELK 架构，解决"有 Trace 找不到日志"；路线 B 把日志纳入 SkyWalking 统一管理，解决"日志量不大但想要开箱即用的 trace-log 联动"。9.x 起日志上报走独立的 `LogReportService` gRPC 流，与 Trace 通道分离，日志洪峰不会挤掉 trace 上报。

# 2. Logback 集成

## 2.1 依赖

要输出 `%tid` 占位符，需要引入 toolkit（注意：**它只用于打 TraceId，不需要也不影响 agent 的自动埋点**；但 gRPC 上报模式必须引 toolkit，因为 appender 类在 toolkit 里）：

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>${skywalking.version}</version> <!-- 与 agent 大版本对齐，如 9.0.0 -->
</dependency>
```

## 2.2 logback-spring.xml

```xml
<configuration>
    <!-- 控制台输出（带 TraceId） -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] %-5level %logger{36} - %msg%n</pattern>
            </layout>
        </encoder>
    </appender>

    <!-- gRPC 上报到 OAP（路线 B） -->
    <appender name="GRPC" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] %-5level %logger{36} - %msg%n</pattern>
            </layout>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="GRPC"/>
    </root>
</configuration>
```

`GRPCLogClientAppender` 内部把日志封装成 gRPC Log 数据后**复用 agent 的通道**上报——前提是 JVM 里挂了 skywalking-agent；没挂 agent 时该 appender 静默不发送。这一点是常见误解：toolkit 不是独立上报器，它依赖 agent 提供的传输与上下文能力。

## 2.3 Pattern 占位符

| 占位符 | 含义 | 无 Trace 时输出 |
|--------|------|----------------|
| `%tid` | TraceId（全局唯一 ID） | `TID: N/A` |
| `%sw_ctx` | SkyWalking 上下文（sw8 头，跨线程/异步场景用） | `SW_CTX: N/A` |

> **行业实践**：JSON 日志（ELK 采集场景）建议把 traceId 放 MDC 而不是 pattern 文本：logback 用 `TraceIdMDCPatternLogbackLayout`，pattern 写 `%X{tid}`，JSON encoder 会把 traceId 输出成独立字段，Filebeat/Logstash 直接解析为 `trace_id`，与 ES 里的 trace 文档做 join 查询，比字符串匹配可靠得多。

# 3. Log4j2 集成

Log4j2 用插件式 Converter 而非 Layout，配置思路一致——把 `%tid` 换成 log4j2 的 `%traceId`，gRPC 上报换 `SkyWalkingAppender`：

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-log4j-2.x</artifactId>
</dependency>
```

```xml
<!-- log4j2.xml：注册 converter 后在 PatternLayout 使用 %traceId -->
<Console name="Console" target="SYSTEM_OUT">
    <PatternLayout charset="UTF-8"
        pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%traceId] %-5level %logger{36} - %msg%n"/>
</Console>
<!-- gRPC 上报：type 为 SkyWalking -->
<SkyWalking name="GRPCLog">
    <PatternLayout pattern="%d{ISO8601} [%traceId] %-5level %msg%n"/>
</SkyWalking>
```

若不想在应用引 toolkit，agent 侧也提供 **toolkit activation 类桥接插件**（在 `agent/optional-plugins` 与 `bootstrap-plugins` 目录下，具体 jar 名随版本略有变化），由 agent 的 classloader 桥接 `%tid` 输出，应用侧零依赖——两种做法选一种，混用会导致重复初始化。

# 4. TraceId 注入原理

**TraceId 注入的本质是"从当前线程的 ContextSnapshot 里取 global trace id"**。Logback/Log4j2 的 converter 在格式化每条日志时调用 `ContextManager.getGlobalTraceId()`：

```java
// SkyWalking toolkit 内部（logback %tid 的转换器简化示意）
public class TraceIdPatternConverter extends ClassicConverter {
    @Override
    public String convert(ILoggingEvent event) {
        // 底层调 agent 的 ContextManager，从当前线程上下文取全局 TraceId
        String traceId = TraceContext.traceId();
        if (traceId == null || traceId.isEmpty()) {
            return "TID: N/A";   // 无上下文（非请求线程/异步线程未传播）
        }
        return "TID: " + traceId;
    }
}
```

由此推出三个行为特征：**其一**，TraceId 与采样无关——未采样的 Trace 依然有 ID、依然会打进日志（agent 传播了上下文，只是不上报 segment）；**其二**，线程池场景默认拿不到正确 TraceId，因为上下文存在 ThreadLocal，需要 `@TraceCrossThread`/`SkyWalkingRunnable` 或 agent 对线程池插件的传播支持，否则异步任务日志显示 `N/A`；**其三**，定时任务、MQ 消费等非入站线程若没被插件创建 Entry Span，日志同样是 `N/A`。

> **注意**：`%tid` 输出的是 SkyWalking 自己的 global traceId（`32位hex.时间戳.序列`格式），与 Zipkin/W3C traceparent 的 ID 不是一回事；如果混用多套追踪协议，日志里最好同时打印两种 ID，或在 OTLP 接入模式下统一取 W3C 格式。

# 5. gRPC 上报链路（LTS 通道）

路线 B 的完整链路：`Appender →（异步队列）→ agent 的 GRPCChannel → OAP LogReceiver → LAL 分析 → Storage`。三个环节都有可调参数，日志量大时逐一评估：

```text
Appender(非阻塞投递)          OAP 侧
  │ 内存 ring buffer，满了直接   │ 11800 gRPC LogReportService
  │ 丢弃(默认策略)而非阻塞业务    ▼
  └──────── gRPC ────────→  LogReceiver → LAL 规则 → ES/BanyanDB
                                    │
                                    └── traceId 相同的 log 与 segment 建立关联索引
```

关键语义：**上报的每条 log 自带 service/instance/traceId 三元组**，OAP 收到后按 traceId 建索引，这就是 UI 里"Trace → Logs"秒开的原理。appender 是异步的、且**宁可丢日志不阻塞业务线程**——队列满时的丢弃只发生在上报通道，本地文件日志不受影响。

# 6. LTS vs ELK 存储选型

SkyWalking 自身的日志存储（LTS 能力）与外接 ELK 是两条路线的落地形态，选型要诚实评估体量：

| 维度 | SkyWalking 内置（LAL + ES/BanyanDB） | 外接 ELK/Loki |
|------|-------------------------------------|---------------|
| 适用量级 | 中小规模（日均 < 数百 GB） | 大规模（TB 级） |
| trace 联动 | UI 原生一键跳转 | 需 Kibana/Loki 按 trace_id 过滤 |
| 全文检索 | 弱（按 traceId/endpoint 维度查） | 强（ES 全文/DSL） |
| 运维成本 | 复用 OAP 栈，几乎零增量 | 独立集群、独立成本 |
| 日志分析 | LAL 抽取指标/错误率 | Logstash/vector transform |

**推荐倾向**：日志总量不大、团队以排障为主要诉求 → 直接 gRPC 上报进 SkyWalking，省去一套 Kafka+Logstash 管线；日志本来就有 ELK/Loki 底座 → 只走路线 A（MDC 带 traceId），在 Kibana 里按 `trace_id` 过滤，SkyWalking UI 配置 **Log 查询跳转（外部链接模板）** 把用户送到 Kibana 对应查询页——两边都能跳转比纠结"日志存哪"更重要。

> **行业实践**：常见折中是"**ERROR 级上报 SkyWalking、全量日志走 ELK**"：用 threshold filter 只把 WARN/ERROR 送 gRPC 通道，错误日志享受原生联动，海量 INFO 不占 SkyWalking 存储。

# 7. LAL — 日志分析语言

LAL（Log Analysis Language）是 OAP 侧对上报日志做"解析—抽取—转指标/事件"的 DSL，类似日志版 OAL。典型用法：抽取错误日志生成 `error_count` 指标喂给告警：

```yaml
# config/lal/default.yaml
rules:
  - name: service-log
    layer: General
    dsl: |
      filter {
        # 1) 解析：用 regexp/grok/json 把原文结构化
        extractor {
          json {}
          # 或 grok { pattern => "%{TIMESTAMP:timestamp} \[%{DATA:tid}\]" }
        }
        # 2) 转指标：level=ERROR 的日志计数
        metrics {
          meter: log_error_count
          labels level: parsed.level, service: tags.service
          value 1
        }
        # 3) 落库：原样存为 log 记录（可按 filter 丢弃 INFO）
        sink {
          es {
            column timestamp: parsed.timestamp
          }
        }
      }
```

LAL 的价值在"**日志驱动的指标**"：错误率、异常关键字计数不需要业务埋点，日志本身就是数据源。规则文件修改后 OAP 支持热加载（`watch` 机制），比 OAL 改动重启友好。

# 8. 日志查询与 Trace 跳转

UI 侧的联动路径与 API 侧查询条件：

```text
Trace 页面                          Logs 页面
  点开一条 Trace                      按 service + endpoint + traceId 过滤
  ├── Span 时间轴上直接内联显示 log      ├── 时间戳对齐到 Span 起止区间
  └── "Logs" 标签页 → 自动带 traceId ──→ └── 每条 log 可再跳回所属 Trace
```

GraphQL 查询日志（`queryLogs`）必带三个条件之一才有好性能：**traceId（最精确）、serviceId+endpointId、时间窗**。裸查全文在 SkyWalking 存储模型里是不支持的低效操作——这是它和 Kibana 的本质差异，也再次印证第 6 节的选型边界。

# 9. 动态日志级别

排障时最痛苦的不是"没有日志"，而是"线上只有 INFO，想看 DEBUG 又不敢重启"。SkyWalking 9.x 的**动态配置（Dynamic Config）服务**支持不重启调整上报级别：在 `application.yml` 启用 `configuration` 组件（zk/etcd/consul/k8s configmap），下发 key：

```yaml
# ZooKeeper / K8s ConfigMap 中下发（key: agent动态日志）
# 支持的值：INFO / DEBUG / WARN / ERROR
logging.level=DEBUG
```

对业务日志级别的动态调整则依赖 logback 自身的 `scan=true` 或 actuator `/loggers` 端点；SkyWalking 动态配置管的是 **agent 自身与上报通道**的行为。两者配合的实战流程：actuator 把业务 logger 调到 DEBUG + SkyWalking 把采样调到全量，排完障双双调回——全程无重启。

> **注意**：动态 DEBUG 是"线上放火"操作，务必配合 `logback` 的 rate-limiting filter 或缩短窗口（如用定时任务 10 分钟后自动调回），否则一次流量尖峰就能把磁盘打满。

# 10. 最佳实践

1. **生产环境推荐"ERROR 上云"折中**：全量走 ELK/Loki，WARN/ERROR 经 gRPC 上报 SkyWalking，兼得联动与成本
2. **Pattern 必须包含 %tid**：无 TraceId 的日志在微服务排障里等于噪音
3. **TID: N/A 的正确处理**：定时任务/线程池场景显式创建 Entry Span（`@Trace` 注解或 `@TraceCrossThread`），让日志重新挂上链路
4. **日志级别**：生产 INFO 起步，DEBUG 只走动态开关短窗口开启
5. **两端互跳**：SkyWalking UI 配置外部 log 查询地址（Kibana Discover 带 traceId 参数模板），别逼人在两个系统间手抄 ID

# 11. 小结

| 主题 | 要点 |
|------|------|
| 两条路线 | 关联查询（%tid/MDC + ELK）与上报存储（gRPC Appender → OAP）共存互补 |
| 注入原理 | converter 取 `ContextManager.getGlobalTraceId()`，与采样无关，线程池需传播支持 |
| 上报链路 | 异步、宁丢不阻塞，log 自带 service/instance/traceId 三元组 |
| 存储选型 | 小体量 SkyWalking 内置，大体量 ELK/Loki + traceId 过滤 + UI 互跳 |
| LAL | 日志抽取指标（错误计数）与结构化落库，规则热加载 |
| 动态级别 | 动态配置管 agent 行为，logback scan/actuator 管业务 logger |

日志打通后，下一篇《8. 持续性能剖析》继续向下钻取到方法级火焰图。
