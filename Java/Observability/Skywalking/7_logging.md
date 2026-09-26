
# 1. 概述

SkyWalking 通过将 TraceId 注入日志 MDC，实现日志与 Trace 的一站式关联查询。

```
Logger.info("处理订单")
  │
  ├── MDC.put("traceId", "xxx")  ← SkyWalking 自动注入
  │
  ▼
日志输出:
  2024-01-15 10:30:00.123 [TID:xxx] INFO  处理订单
                                      │
                                      └── TraceId 已注入
```

# 2. Logback 集成

## 2.1 依赖

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>${skywalking.version}</version>
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

    <!-- gRPC 上报到 OAP -->
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

## 2.3 Pattern 占位符

| 占位符 | 含义 |
|--------|------|
| `%tid` | TraceId（全局唯一 ID） |
| `%sw_ctx` | SkyWalking 上下文（完整 sw8） |

# 3. Log4j2 集成

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-log4j-2.x</artifactId>
</dependency>
```

```xml
<PatternLayout>
    <Pattern>%d [%traceId] %-5level %logger{36} - %msg%n</Pattern>
</PatternLayout>
```

# 4. TraceId 注入原理

```java
// SkyWalking 内部在 Span 创建时自动注入 MDC
public class TraceIdMDCPatternConverter extends LogEventPatternConverter {

    @Override
    public void format(LogEvent event, StringBuilder builder) {
        // 从 ThreadLocal 获取当前 TraceId
        String traceId = ContextManager.getGlobalTraceId();
        if (traceId != null && !traceId.equals(AbstractTracerContext.IGNORE_TRACE)) {
            builder.append(traceId);
        } else {
            builder.append("N/A");
        }
    }
}
```

# 5. 日志与 Trace 联动查询

在 SkyWalking UI 中：
1. 查看某个 Trace → 点击 "Related Logs"
2. 跳转到日志页面 → 自动按 TraceId 过滤
3. 查看该 Trace 过程中的所有日志

```
Trace:
  Segment (ServiceA)
    ├── Span: /api/user (100ms)
    │     ├── Log: "query user id=1"         ← 应用日志
    │     └── Log: "user found"              ← 应用日志
    └── Span: JDBC/select (20ms)
          └── Log: "SQL executed in 20ms"    ← 应用日志
```

# 6. LAL — 日志分析语言

```yaml
# config/lal/default.yaml
rules:
  - name: error-log
    dsl: |
      filter {
        text {
          abortOnFailure false
          regexp "(?<timestamp>\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}\\.\\d{3})" 
        }
        extractor {
          metrics {
            timestamp "$timestamp" 
            labels level: parsed.level, service: parsed.service
            name "log_error_count"
            value 1
          }
        }
        sink {}
      }
```

# 7. 最佳实践

1. **生产环境推荐 gRPC Appender**：日志直接上报 OAP，无需中间存储
2. **Pattern 始终包含 %tid**：确保每条日志可追溯到 Trace
3. **TID:N/A 处理**：非 HTTP 请求（如定时任务）时 TraceId 为 N/A
4. **日志级别**：生产环境 INFO 级别，避免 DEBUG 日志淹没追踪信息
