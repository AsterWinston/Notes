# 1. 知识点
- **SLF4J(Simple Logging Facade for Java)**：Java 日志门面，只提供 API 接口不实现日志功能，需配合具体日志实现框架
- **门面模式(Facade)**：SLF4J 是典型的门面模式，统一编码接口，底层可切换不同日志实现
- **logback**：SLF4J 同门作者(Ceki Gulcu)，Spring Boot 默认内置日志实现，是 log4j 1.x 的继承者
- **log4j2**：Apache 的全新日志实现，异步性能最好，架构大改版（不同于 log4j 1.x）
- **log4j**：老的 Apache 日志框架，已被 log4j2 和 logback 取代
- **运行绑定**：SLF4J 在运行时 Classpath 加载具体实现（logback-classic、slf4j-log4j12 等）
- **静态绑定(Static Binding)**：SLF4J 通过 org.slf4j.impl.StaticLoggerBinder 绑定具体实现，编译后固定
- **桥接jar(Bridge)**：将旧框架的日志调用重定向到 SLF4J，如 jcl-over-slf4j 将 commons-logging 桥接到 SLF4J
- **jcl-over-slf4j**：桥接 commons-logging 到 SLF4J，log4j-over-slf4j 桥接 log4j 1.x 到 SLF4J
- **logback-classic**：logback 的核心模块，实现 SLF4J 接口，包含 logback-core 基础功能
- **logback-spring.xml**：Spring Boot 推荐的 logback 配置文件名（优先 logback-spring.xml > logback.xml），支持 Spring 扩展特性
- **appender**：日志输出目的地，如 ConsoleAppender(控制台)、FileAppender(文件)、RollingFileAppender(滚动文件)
- **log format模式**：PatternLayout 定义日志输出格式，常用占位符：%d{日期}、%t{线程}、%p{级别}、%c{类名}、%logger{logger名}、%m{消息}、%n{换行}
- **%d**：日期占位符，如 %d{yyyy-MM-dd HH:mm:ss.SSS}
- **%t**：线程名占位符
- **%p/%level**：日志级别占位符
- **%c/%logger**：Logger 名称占位符，通常为类的全限定名
- **%m/%msg**：日志消息占位符
- **%n**：换行符
- **maxHistory**：滚动日志保留的最大历史文件数或天数
- **Filter LevelFilter**：logback 的过滤机制，按级别过滤日志，可设置 onMatch/onMismatch 行为
- **springProfile**：logback-spring.xml 专有特性，根据 Spring 激活的 profile 区分不同环境的日志配置
- **异步日志(AsyncAppender)**：logback 的异步日志组件，将日志写入队列，后台线程异步处理
- **discardingThreshold**：AsyncAppender 队列满时丢弃策略的阈值，队列使用率超过该百分比后丢弃 TRACE/DEBUG/INFO 级别日志
- **neverBlock**：AsyncAppender 的配置，true 时队列满也不阻塞 Event 线程，默认 false
- **LogstashLogbackEncoder**：输出 JSON 格式日志的 Encoder，供 ELK/Logstash 直接收集处理
- **log4j2-spring.xml**：Spring Boot 使用 log4j2 时推荐的配置文件名
- **Disruptor**：高性能无锁环形队列，log4j2 AsyncLogger 基于 Disruptor 实现异步日志，吞吐量极高
- **异步日志性能对比**：log4j2 AsyncLogger > logback AsyncAppender > logback 同步
- **日志级别(ALL < TRACE < DEBUG < INFO < WARN < ERROR < FATAL < OFF)**：设置 Logger 级别后，低于该级别的日志被过滤
- **TRACE**：最详细的调试级别，跟踪程序执行路径
- **DEBUG**：调试信息，记录变量值、方法调用等
- **INFO**：一般业务信息，如服务启动、重要业务节点，默认生产环境推荐级别
- **WARN**：警告信息，存在潜在问题但不影响程序运行
- **ERROR**：错误信息，运行异常或功能失败，需要关注处理
- **FATAL**：致命错误，JVM 崩溃或系统级别致命故障
- **正确选择日志级别**：生产用 INFO，开发用 DEBUG，敏感调试临时开 TRACE
- **MDC(Mapped Diagnostic Context)**：映射诊断上下文，底层基于 ThreadLocal，存储当前线程的自定义上下文信息
- **ThreadLocal存储上下文**：MDC 内部使用 InheritableThreadLocal，在线程池场景需注意子线程不会自动继承
- **userId/requestId/traceId**：MDC 最常见的存储内容，便于日志检索串联
- **Filter注入MDC**：编写 Servlet Filter/Spring Interceptor，在请求入口将 traceId/userId 存入 MDC，请求结束时清除
- **%X{traceId}**：logback 中 MDC 的输出占位符，%X{key} 输出指定 key 的值
- **PatternLayout**：定义日志输出格式的类，将 LoggingEvent 转换为字符串输出
- **日期/线程/level/logger/message/newline**：日志格式的几大核心组成部分
- **TimeBasedRollingPolicy**：按日期/时间滚动的策略，如按天创建新日志文件
- **SizeBasedTriggeringPolicy**：按文件大小触发滚动，达到设定大小后新建日志文件
- **SizeAndTimeBasedRollingPolicy**：复合滚动策略，按时间滚动且每个时间段按大小拆分
- **日志最佳实践**：不在循环中使用日志、使用占位符 {} 避免字符串拼接、异常日志包含栈信息、生产环境禁止 DEBUG 级别、异步日志提升性能
- **占位符 {}**：SLF4J 提供的占位符语法 `logger.info("user {} login", userId)`，相比字符串拼接 `"user "+userId+" login"`，DEBUG 级别关闭时可避免拼接开销
- **占位符多参数**：`logger.debug("a={}, b={}", a, b)`，支持多个参数；最后参数为 Throwable 时自动打印异常栈
- **日志收集规范**：统一 JSON 格式输出、包含 traceId/sessionId、便于 ELK 解析检索
- **SkyWalking 日志关联**：SkyWalking 自带的 logback/log4j2 布局可将 Trace ID（%tid）自动输出到日志中，实现从日志跳转到 Trace
- **%tid**：SkyWalking 日志集成后自动输出的 trace id 占位符
- **敏感日志脱敏**：在日志输出前对手机号中间4位替换为 ****、身份证号掩码、密码/Token完全屏蔽处理

# 2. 面试题

## 2.1 Java 有哪些常用的日志框架？它们的关系是什么？
- **SLF4J**：日志门面，只定义接口没有实现，统一 API 入口
- **Logback**：SLF4J 的具体实现，Spring Boot 默认日志框架
- **Log4j2**：Apache 的全新日志实现，异步性能最好
- **Log4j 1.x**：Apache 老旧日志框架，已停止维护
- **JUL(java.util.logging)**：JDK 内置日志，功能简陋基本不用
- **JCL(Jakarta Commons Logging)**：古老的日志门面，Spring 曾使用，基本被 SLF4J 替代

**关系**：应用代码 → SLF4J API → 具体实现(Logback/Log4j2) → 日志输出(Appender → 控制台/文件/ELK)
桥接：旧框架(commons-logging) → jcl-over-slf4j → SLF4J → Logback

## 2.2 SLF4J 的绑定原理是什么？
SLF4J 在编译时并不依赖具体实现，而是在运行时通过 Classpath 搜索：
1. SLF4J API 中定义的 `LoggerFactory` 会通过 `StaticLoggerBinder` 获取 `ILoggerFactory`
2. 每个日志实现框架提供一个自己的 `StaticLoggerBinder` 类（如 logback-classic.jar、slf4j-log4j12.jar）
3. 如果 Classpath 中有多个实现，SLF4J 会警告并随机选一个
4. 如果没有任何实现，SLF4J 会降级为 NOPLogger（不输出日志）

因此切换日志框架只需替换 Maven 依赖，无需修改代码。

## 2.3 什么是日志桥接(Bridge)？常见的桥接有哪些？
日志桥接(Bridge)是将旧日志框架的 API 调用重定向到 SLF4J，由 SLF4J 最终委托给统一的日志实现。

常见桥接：
| 桥接 Jar | 作用 |
|---------|------|
| jcl-over-slf4j | 将 commons-logging API 调用转为 SLF4J（Spring 内部 jcl） |
| log4j-over-slf4j | 将 log4j 1.x API 调用转为 SLF4J |
| jul-to-slf4j | 将 java.util.logging 转为 SLF4J |

**注意**：桥接 jar（如 log4j-over-slf4j）和实现 jar（如 log4j 1.x jar）不能同时存在于 Classpath，否则会死循环！

## 2.4 logback 和 log4j2 有什么区别？如何选型？
| 维度 | Logback | Log4j2 |
|------|---------|--------|
| 作者 | Ceki Gulcu（同 SLF4J） | Apache |
| Spring Boot 默认 | 是 | 否，需显式排除 logback |
| 配置文件 | logback-spring.xml | log4j2-spring.xml |
| 同步性能 | 较好 | 更好 |
| 异步性能 | AsyncAppender（Queue） | AsyncLogger（Disruptor 无锁队列） |
| 刷新配置 | 自动重载配置 | 自动重载配置 |
| 插件/扩展性 | 有限 | 丰富的 Plugin 体系 |
| Java 版本 | 兼容性好 | 需要 Java 8+ |

**选型建议**：如果追求极致异步性能和大吞吐量日志选 log4j2；如果是 Spring Boot 标准项目用 logback 即可满足大部分需求。

## 2.5 logback 中如何配置异步日志？
```xml
<!-- logback-spring.xml -->
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志，设为 0 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 队列大小 -->
    <queueSize>1024</queueSize>
    <!-- 队列满时不阻塞主线程（推荐 false 保证日志不丢失） -->
    <neverBlock>false</neverBlock>
    <!-- 关联真实 Appender -->
    <appender-ref ref="FILE"/>
</appender>

<root level="INFO">
    <appender-ref ref="ASYNC"/>
</root>
```
关键配置：
- `discardingThreshold`：队列使用率超过此百分比时丢弃低级别日志（0 表示不丢弃）
- `queueSize`：阻塞队列大小（幂次方 1024, 2048），太小日志丢失太大内存爆
- `neverBlock`：队列满时是否阻塞主线程，false 保日志完整性

## 2.6 什么是 MDC？如何使用 MDC 存储 TraceId？
MDC（Mapped Diagnostic Context）是 SLF4J 提供的功能，底层基于 ThreadLocal，用于存储当前线程的上下文信息（userId、traceId、requestId、sessionId）。

**使用步骤**：
1. 在请求入口（Filter/Interceptor）中设置 MDC：
```java
import org.slf4j.MDC;

@WebFilter("/*")
public class TraceIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        try {
            String traceId = generateTraceId(); // 从请求头获取或生成
            MDC.put("traceId", traceId);
            MDC.put("userId", getUserId());
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();  // 必须清除，防止内存泄漏和线程池复用问题
        }
    }
}
```
2. logback 配置中使用 `%X{traceId}` 输出：
```xml
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] [%thread] %-5level %logger{36} - %msg%n</pattern>
```
输出效果：`2024-01-01 12:00:00.000 [abc123def456] [http-nio-8080-exec-1] INFO  com.example.Controller - user login`

## 2.7 如何实现日志输出为 JSON 格式（供 ELK 收集）？
使用 Logstash Logback Encoder：
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```
```xml
<!-- logback-spring.xml -->
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <customFields>{"application":"my-service","env":"prod"}</customFields>
    </encoder>
</appender>
```
输出格式：
```json
{"@timestamp":"2024-01-01T12:00:00.000Z","message":"user login","level":"INFO","logger_name":"com.example.Controller","thread_name":"http-nio-8080-exec-1","traceId":"abc123","application":"my-service","env":"prod"}
```
然后 Filebeat 直接采集 JSON 日志 → Logstash → Elasticsearch → Kibana，无需解析。

## 2.8 日志滚动策略有哪些？如何配置？
**RollingPolicy 类型**：
- **TimeBasedRollingPolicy**：按时间滚动（小时/天/月）
- **SizeBasedTriggeringPolicy**：按文件大小滚动（如超过 100MB）
- **SizeAndTimeBasedRollingPolicy**：时间和大小复合策略

**logback 配置示例**：
```xml
<appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <!-- 滚动文件名格式 -->
        <fileNamePattern>logs/app-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <!-- 每个文件最大 100MB -->
        <maxFileSize>100MB</maxFileSize>
        <!-- 保留最近 30 天的日志 -->
        <maxHistory>30</maxHistory>
        <!-- 日志总大小上限（优先级高于 maxHistory） -->
        <totalSizeCap>10GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>%d %-5level [%thread] %logger{36}: %msg%n</pattern>
    </encoder>
</appender>
```

## 2.9 日志开发有哪些最佳实践？
1. **不在循环体中打日志**：循环体中的日志会导致 I/O 暴增，严重影响性能
2. **使用参数化占位符 {}**：`logger.debug("user:{} login", userId)` 而非字符串拼接 `"user:" + userId + " login"`。当 DEBUG 级别关闭时，占位符方法无开销，字符串拼接会有
3. **打印完整异常信息**：`logger.error("处理订单异常", exception)`，不要只打 `e.getMessage()` 而丢失栈信息
4. **生产用 INFO 级别**：DEBUG/TRACE 日志量巨大，严重拖低性能并冲硬盘
5. **敏感信息脱敏**：手机号、身份证、信用卡号中间四位打印 ****
6. **使用异步日志**：特别是高并发应用，同步日志成为性能瓶颈
7. **合适的 log level**：正常业务流程用 INFO，异常用 ERROR，需尽快改进用 WARN
8. **日志包含上下文**：记录关键业务字段（userId/orderId/traceId），方便定位问题
9. **关键节点日志**：接口入参出参、调用外部服务、任务开始/结束等
10. **finalize 中清理 MDC**：请求结束时 `MDC.clear()`

## 2.10 为什么日志占位符 `{}` 比字符串拼接好？
```java
// 字符串拼接 — 坏实践
logger.debug("Processing request: " + request + " with params: " + params);
// 即使 DEBUG 级别关闭，字符串拼接依然执行，创建无用 String 对象消耗 CPU 和内存

// 占位符 — 好实践
logger.debug("Processing request: {} with params: {}", request, params);
// DEBUG 关闭时跳过字符串格式化和分配，几乎零开销
```
背后的原理是 SLF4J 延迟执行：先检查日志级别是否符合条件（`logger.isDebugEnabled()`），符合条件的才格式化消息。

## 2.11 如何实现日志的敏感信息脱敏？
方案一：覆写 logback MessageConverter（级别：格式化链路）
```java
public class SensitiveConverter extends MessageConverter {
    @Override
    public String convert(ILoggingEvent event) {
        String message = super.convert(event);
        // 手机号中间四位脱敏
        message = message.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
        // 身份证中间数字脱敏
        message = message.replaceAll("(\\d{4})\\d{10}(\\d{4})", "$1****$2");
        return message;
    }
}
```

方案二：自定义注解 + AOP 处理（适用于特定方法的入参和返回值）

方案三：JSON 序列化时使用自定义 Serializer（Jackson @JsonSerializer），在对象序列前滤掉敏感字段

方案四：使用 Fastjson 的 filter 或 Jackson 的 PropertyFilter 在序列化中剔除敏感字段

推荐方案一全局覆盖日志输出。

## 2.12 日志收集的规范架构是什么样的？
```
业务应用(JSON Log + TraceId + MDC)
    ↓ (写入日志文件)
Filebeat (采集)
    ↓
Kafka (削峰缓冲，防止日志风暴打爆下游)
    ↓
Logstash/Fluentd (解析清洗，过滤不需要的日志，提高字段)
    ↓
Elasticsearch (存储日志数据，提供搜索聚合)
    ↓
Kibana (可视化展示和日志搜索，告警)
```

规范要求：
- 所有应用统一 JSON 日志格式（时间戳、级别、消息、traceId/spanId、服务名、环境）方便 ES 索引和 Kibana 看板
- 日志规范记录 traceId，唯一跨服务共享，实现端到端日志串联
- 与 APM（如 SkyWalking）集成，实现 Trace → Log 一键跳转
- 日志统计做长期指标聚合，一个月后归档到冷存储

## 2.13 如何将 SkyWalking 的 TraceId 输出到日志中？
1. 添加依赖（Logback 集成）：
```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>${skywalking.version}</version>
</dependency>
```
2. 在 logback-spring.xml 中使用 TraceIdPatternLogbackLayout：
```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
        <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
            <pattern>%d %-5level [%tid] [%thread] %logger{36} - %msg%n</pattern>
        </layout>
    </encoder>
</appender>
```
或使用 SkyWalking 的 gRPC Reporter 日志组件（`apm-toolkit-logging-common`），直接将日志发到 SkyWalking OAP 后端。

## 2.14 异步日志的原理和注意事项？
**原理**：主线程不在 I/O 阻塞中将日志写入 appender（磁盘/网络），而是将日志 Event 放入队列（ArrayBlockingQueue/Disruptor RingBuffer），异步线程消费队列执行真正的磁盘写入。

**注意事项**：
- **队列大小**：合理设置（512-8192），太小频繁满队列溢出，太大内存压力
- **丢日志风险**：队列满时按照 Drop 策略丢弃日志（logback 的 AsyncAppender 默认丢弃 TRACE/DEBUG/INFO），生产环境建议 discardingThreshold=0(不丢弃)
- **JVM 退出**：应用正常关闭时，logback 不会立即终止而是等待队列清空。断电/kill -9 会丢队列中未消费完的空闲日志
- **顺序性问题**：同一队列同步输出保证 Ordered，但不同 AsyncAppender 之间的日志顺序可能混乱
- **MDC 传递**：Async 场景 logback 会自动将 MDC 上下文传给工作线程，不需额外处理
