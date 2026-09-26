# 1. 概述与简介

## 1.1 什么是 Logback

Logback 是由 Log4j 创始人 Ceki Gülcü 设计的 Java 日志框架，是 Log4j 1.x 的替代品。它被设计为 SLF4J 的原生实现，在性能、可靠性和功能上都超越了 Log4j 1.x。Logback 目前是 Spring Boot 默认的日志框架。

## 1.2 Logback 的发展历程

- **2006年**：Ceki Gülcü 离开 Apache 后开始开发 Logback
- **2009年**：Logback 1.0 第一个候选版本发布
- **2011年**：Logback 1.0.0 正式发布
- **2014年**：Spring Boot 1.0 选择 Logback 作为默认日志框架
- **至今**：持续维护更新，最新稳定版为 1.4.x（与 SLF4J 2.0 兼容）

## 1.3 Logback 的优势

- **原生 SLF4J 实现**：完全兼容 SLF4J API，零性能损耗
- **自动重载配置**：支持配置文件的自动热重载
- **条件化配置**：支持 if-then-else 条件处理
- **更快的执行速度**：比 Log4j 1.x 快 10 倍以上
- **更小的内存占用**：对象创建和序列化方面更优化
- **丰富的 Appender**：提供多种输出目标
- **强大的 Filter**：支持正则、级别、求值等多种过滤器

# 2. 核心概念与架构

## 2.1 三层架构

Logback 分为三个模块：

### 2.1.1 logback-core

核心模块，为其他两个模块提供基础：

- Appender 接口和基类
- Layout/Encoder 基础
- 过滤器机制
- 状态管理（StatusManager）
- 配置解析基础

### 2.1.2 logback-classic

经典模块，是 SLF4J 的原生实现：

- 实现了 `org.slf4j.Logger` 接口
- 提供了 `ch.qos.logback.classic.Logger`
- 完整的配置支持（logback.xml）
- MDC 实现
- TurboFilter（全局过滤器，在任何过滤器之前执行）

### 2.1.3 logback-access

访问模块，与 Servlet 容器集成：

- 提供 HTTP 访问日志功能
- 可集成到 Tomcat、Jetty 等 Servlet 容器
- 支持输出 HTTP 请求/响应信息

## 2.2 核心组件关系

```
+------------------+      +------------------+
|    Logger        |      |    Appender      |
|  (日志记录器)     |----->|  (日志输出目标)    |
+------------------+      +--------+---------+
       |                           |
       | 级别继承                    | 包含
       v                           v
+------------------+      +------------------+
| Logger Hierarchy |      |    Layout/Encoder|
| (父子层级关系)    |      |  (日志格式化器)    |
+------------------+      +------------------+
```

## 2.3 Logger 的层级关系

Logback 中的 Logger 遵循层级命名规则，使用点号分隔：

```java
// Logger 层级示例
Logger root = LoggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);  // 根 Logger
Logger com = LoggerFactory.getLogger("com");
Logger comExample = LoggerFactory.getLogger("com.example");
Logger comExampleService = LoggerFactory.getLogger("com.example.Service");
```

层级继承特性：
- 子 Logger 继承父 Logger 的级别（如果未显式设置）
- 子 Logger 默认继承父 Logger 的 Appender（可通过 `additivity` 控制）
- 根 Logger 是所有 Logger 的祖先

## 2.4 日志级别继承规则

```
根 Logger 级别: INFO
    ├── com (未设级别) -> 继承 INFO
    │   ├── com.example (未设级别) -> 继承 INFO
    │   │   └── com.example.Service (未设级别) -> 继承 INFO
    │   └── com.other (级别: WARN) -> 显式设置 WARN
    └── org (未设级别) -> 继承 INFO
        └── org.springframework (级别: DEBUG) -> 显式设置 DEBUG
```

## 2.5 输出流程

```
Logger.log()
    |
    v
Level 检查 (有效级别 vs 消息级别)
    |
    v (通过)
TurboFilter 链 (全局过滤器)
    |
    v (通过)
Appender 选择
    |
    v
Filter 链 (Appender 级别过滤器)
    |
    v (通过)
Layout/Encoder 格式化
    |
    v
输出到目标 (控制台/文件/网络等)
```

# 3. 配置详解

## 3.1 logback.xml 基础结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds" debug="false">

    <!-- 属性定义 -->
    <property name="LOG_PATH" value="./logs" />
    <property name="LOG_PATTERN" value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n" />

    <!-- 控制台 Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 根 Logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>

</configuration>
```

## 3.2 配置文件查找顺序

Logback 按照以下顺序查找配置文件：

1. `logback.groovy`（如果 Groovy 在类路径上）
2. `logback-test.xml`（测试环境配置）
3. `logback.xml`（生产环境配置）
4. 如果都未找到，使用 `BasicConfigurator`（输出到控制台，级别为 DEBUG）

## 3.3 Appender 详解

### 3.3.1 ConsoleAppender

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
    </encoder>
    <!-- target: System.out 或 System.err -->
    <target>System.out</target>
</appender>
```

`%highlight`、`%cyan` 等是 Logback 的着色转换字，只在控制台输出中生效。

### 3.3.2 FileAppender

```xml
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>${LOG_PATH}/app.log</file>
    <append>true</append>
    <encoder>
        <pattern>${LOG_PATTERN}</pattern>
    </encoder>
</appender>
```

### 3.3.3 RollingFileAppender

最常用的文件滚动策略，支持基于时间、文件大小或两者的组合：

```xml
<appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_PATH}/app.log</file>

    <!-- 滚动策略 -->
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!-- 文件名模式：按天滚动 -->
        <fileNamePattern>${LOG_PATH}/app.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
        <!-- 最大保留 30 天 -->
        <maxHistory>30</maxHistory>
        <!-- 总日志文件大小上限 -->
        <totalSizeCap>10GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>${LOG_PATTERN}</pattern>
    </encoder>
</appender>
```

基于大小和时间的混合策略：

```xml
<appender name="SIZE_ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_PATH}/app.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${LOG_PATH}/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <!-- 每个文件最大 100MB -->
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>10GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>${LOG_PATTERN}</pattern>
    </encoder>
</appender>
```

`%i` 是滚动索引，当单个文件达到 `maxFileSize` 时递增。

### 3.3.4 SocketAppender & SMTPAppender

```xml
<!-- 网络发送 -->
<appender name="SOCKET" class="ch.qos.logback.classic.net.SocketAppender">
    <remoteHost>logserver.example.com</remoteHost>
    <port>4560</port>
    <reconnectionDelay>30000</reconnectionDelay>
</appender>

<!-- 邮件发送（需包含 javax.mail 依赖） -->
<appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <smtpHost>smtp.example.com</smtpHost>
    <to>admin@example.com</to>
    <from>noreply@example.com</from>
    <subject>[ERROR] %logger{20} - %m</subject>
    <layout class="ch.qos.logback.classic.html.HTMLLayout" />
    <evaluator class="ch.qos.logback.classic.boolex.OnMarkerEvaluator">
        <marker>FATAL_AUDIT</marker>
    </evaluator>
</appender>
```

### 3.3.5 SiftingAppender

根据运行时值动态创建子 Appender。典型场景：为每个用户或每个线程创建独立的日志文件：

```xml
<appender name="SIFT" class="ch.qos.logback.classic.sift.SiftingAppender">
    <!-- MDC 中的值作为区分键 -->
    <discriminator>
        <key>userId</key>
        <defaultValue>unknown</defaultValue>
    </discriminator>
    <sift>
        <!-- 每个 userId 会创建一个独立的 FileAppender -->
        <appender name="FILE-${userId}" class="ch.qos.logback.core.FileAppender">
            <file>${LOG_PATH}/user_${userId}.log</file>
            <append>true</append>
            <layout class="ch.qos.logback.classic.PatternLayout">
                <pattern>%d{HH:mm:ss.SSS} [%thread] - %msg%n</pattern>
            </layout>
        </appender>
    </sift>
</appender>
```

**注意**：`SiftingAppender` 会为每个不同的 MDC 值创建新的 Appender 实例，需要关注内存管理。可通过 `timeout` 属性控制空闲 Appender 的回收。

## 3.4 Encoder 与 Layout

### 3.4.1 PatternLayout

Logback 的 PatternLayout 支持丰富的转换字：

| 转换字 | 说明 | 示例 |
|--------|------|------|
| `%d{pattern}` | 日期 | `%d{yyyy-MM-dd HH:mm:ss.SSS}` |
| `%thread` | 线程名 | `http-nio-8080-exec-1` |
| `%-5level` | 日志级别（左对齐） | `INFO`、`DEBUG` |
| `%logger{length}` | Logger 名 | `%logger{36}` |
| `%msg` | 日志消息 | 用户自定义消息 |
| `%n` | 换行 | 平台无关换行 |
| `%M` | 方法名 | `createUser` |
| `%L` | 行号 | `42` |
| `%class` | 全类名 | `com.example.UserService` |
| `%method` | 方法名 | `createUser` |
| `%X{key}` | MDC 值 | `%X{requestId}` |
| `%marker` | Marker | `AUDIT` |
| `%replace(p){r,t}` | 正则替换 | `%replace(%msg){'password=\w+','password=***'}` |
| `%highlight` | ANSI 着色 | `%highlight(%-5level)` |
| `%cyan`、`%green`、`%red`、`%yellow` | 颜色 | `%cyan(%logger{36})` |

### 3.4.2 Encoder vs Layout

Logback 0.9.19 之前使用 `Layout` 进行格式化，之后推荐使用 `Encoder`：

```xml
<!-- 老式 Layout -->
<layout class="ch.qos.logback.classic.PatternLayout">
    <pattern>%msg%n</pattern>
</layout>

<!-- 推荐的 Encoder 方式（更高效） -->
<encoder>
    <pattern>%msg%n</pattern>
    <charset>UTF-8</charset>
    <!-- 立即刷新 -->
    <immediateFlush>true</immediateFlush>
</encoder>
```

`Encoder` 相比 `Layout` 的优势：
- 可以在输出前对字节数组进行操作
- 支持字符编码设置
- 在 Appender 生命周期中更高效

## 3.5 过滤器（Filter）

### 3.5.1 LevelFilter

```xml
<appender name="ERROR_FILE" class="ch.qos.logback.core.FileAppender">
    <file>${LOG_PATH}/error.log</file>
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>ERROR</level>
        <onMatch>ACCEPT</onMatch>     <!-- ERROR 级别日志接受 -->
        <onMismatch>DENY</onMismatch>  <!-- 其他级别拒绝 -->
    </filter>
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

### 3.5.2 ThresholdFilter

```xml
<appender name="WARN_FILE" class="ch.qos.logback.core.FileAppender">
    <file>${LOG_PATH}/warn_and_above.log</file>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>WARN</level>  <!-- 只输出 WARN 及以上级别 -->
    </filter>
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

### 3.5.3 EvaluatorFilter

基于评估表达式的复杂过滤（需要 Janino 库）：

```xml
<dependency>
    <groupId>org.codehaus.janino</groupId>
    <artifactId>janino</artifactId>
    <version>3.1.9</version>
</dependency>
```

```xml
<appender name="SPECIAL" class="ch.qos.logback.core.FileAppender">
    <file>${LOG_PATH}/special.log</file>
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">
        <evaluator class="ch.qos.logback.classic.boolex.JaninoEventEvaluator">
            <!-- 表达式：匹配包含 "TRANSACTION" 且级别为 INFO 及以上的日志 -->
            <expression>
                return message.contains("TRANSACTION") && level >= DEBUG_INT;    <!-- 实际应为 INFO_INT 等常量 -->
            </expression>
        </evaluator>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
    </filter>
    <encoder>
        <pattern>%msg%n</pattern>
    </encoder>
</appender>
```

### 3.5.4 TurboFilter

全局过滤器，在 Logger 级别决定是否处理日志事件（性能最高）：

```java
package com.example.filter;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.turbo.TurboFilter;
import ch.qos.logback.core.spi.FilterReply;
import org.slf4j.Marker;

public class PerformanceTurboFilter extends TurboFilter {
    @Override
    public FilterReply decide(Marker marker, Logger logger,
                              Level level, String format, Object[] params, Throwable t) {
        // 根据 Logger 名称和级别做全局决策
        if (logger.getName().startsWith("com.example.performance")) {
            return FilterReply.ACCEPT;
        }
        // 对特定 Marker 的处理
        if (MarkerFactory.getMarker("AUDIT").equals(marker)) {
            return FilterReply.ACCEPT;
        }
        return FilterReply.NEUTRAL;  // 继续后续处理
    }
}
```

配置：

```xml
<configuration>
    <turboFilter class="com.example.filter.PerformanceTurboFilter" />
    <!-- ... -->
</configuration>
```

## 3.6 参数化配置

```xml
<configuration scan="true" scanPeriod="30 seconds">

    <!-- 属性定义 -->
    <property name="APP_NAME" value="myapp" />
    <property name="LOG_HOME" value="./logs" />
    <property name="LOG_PATTERN" value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n" />

    <!-- 使用系统属性默认值 -->
    <property name="LOG_LEVEL" value="${LOG_LEVEL:-INFO}" />

    <!-- 条件化配置（需要 Janino） -->
    <if condition='property("ENV").equals("prod")'>
        <then>
            <root level="INFO">
                <appender-ref ref="FILE" />
                <appender-ref ref="ASYNC" />
            </root>
        </then>
        <else>
            <root level="DEBUG">
                <appender-ref ref="CONSOLE" />
            </root>
        </else>
    </if>

    <!-- Logger 配置 -->
    <logger name="com.example" level="DEBUG" additivity="false">
        <appender-ref ref="FILE" />
    </logger>

    <!-- 根 Logger -->
    <root level="${LOG_LEVEL}">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

# 4. 高级特性

## 4.1 MDC 支持

Logback 原生支持 SLF4J 的 MDC，并在配置中提供丰富的访问方式：

```xml
<appender name="MDC_FILE" class="ch.qos.logback.core.FileAppender">
    <file>logs/mdc.log</file>
    <encoder>
        <pattern>
            %d{HH:mm:ss.SSS}
            |%X{requestId:-N/A}          <!-- MDC 中的 requestId，缺省为 N/A -->
            |%X{userId:-anonymous}        <!-- MDC 中的 userId，缺省为 anonymous -->
            |%thread
            |%-5level
            |%logger{36}
            |%msg%n
        </pattern>
    </encoder>
</appender>
```

MDC 的高级用法 - 在异步线程传递 MDC：

```java
import ch.qos.logback.classic.util.LogbackMDCAdapter;

// 自定义 Runnable 包装器
public class MDCRunnable implements Runnable {
    private final Runnable original;
    private final Map<String, String> mdcContext;

    public MDCRunnable(Runnable original) {
        this.original = original;
        this.mdcContext = MDC.getCopyOfContextMap();
    }

    @Override
    public void run() {
        if (mdcContext != null) {
            MDC.setContextMap(mdcContext);
        }
        try {
            original.run();
        } finally {
            MDC.clear();
        }
    }
}
```

## 4.2 配置自动重载

通过 `scan="true"` 和 `scanPeriod` 启用自动重载：

```xml
<!-- 每 60 秒扫描配置文件变化 -->
<configuration scan="true" scanPeriod="60 seconds">
```

注意：
- 默认 `scanPeriod` 为 1 分钟
- 单位可以是 `milliseconds`、`seconds`、`minutes`、`hours`
- 修改配置文件后 Logback 会自动应用新配置，**不丢失任何日志事件**
- 自动重载通过 `ReconfigureOnChangeTask` 实现

## 4.3 条件化配置

```xml
<configuration>
    <!-- 条件化配置，需要 Janino 运行时依赖 -->
    <if condition='property("HOSTNAME").contains("prod")'>
        <then>
            <property name="LOG_LEVEL" value="INFO" />
        </then>
        <else>
            <property name="LOG_LEVEL" value="DEBUG" />
        </else>
    </if>

    <!-- 多条件判断 -->
    <if condition='property("ENV").equals("prod") || property("ENV").equals("staging")'>
        <then>
            <appender-ref ref="ROLLING_FILE" />
        </then>
        <elseif condition='property("ENV").equals("dev")'/>
            <then>
                <appender-ref ref="CONSOLE" />
            </then>
        </elseif>
        <else>
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="FILE" />
        </else>
    </if>
</configuration>
```

可用变量：
- `property(key)` - 配置属性或系统属性
- `p(key)` - 同 `property(key)`
- `isDefined(key)` - 属性是否已定义
- `isNull(key)` - 属性是否为 null

## 4.4 Logger 加法性（Additivity）

```xml
<configuration>
    <logger name="com.example" level="DEBUG" additivity="false">
        <appender-ref ref="FILE" />
    </logger>
    <!--
        additivity="false" 表示 com.example 的日志不会传递给父 Logger（root）
        不设置时默认为 true，日志会在当前 Logger 和父 Logger 都输出
    -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

# 5. 底层原理

## 5.1 LoggerContext

`LoggerContext` 是 Logback 的核心容器，负责：

- 维护 Logger 层级树
- 管理 Appender 实例
- 持有配置状态
- 生命周期管理

```java
// LoggerContext 可以通过以下方式获取
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
```

## 5.2 日志事件处理流程

```java
// Logger.log() 方法的核心逻辑（简化）
public void log(Level level, String msg, Object[] args, Throwable t) {
    // 1. 检查日志级别
    if (!isEnabledFor(level)) return;

    // 2. 创建日志事件
    LoggingEvent event = new LoggingEvent(
        fqcn,                       // 调用者类名
        this,                       // Logger 实例
        level,                      // 日志级别
        msg,                        // 消息
        t,                          // 异常
        args                        // 参数
    );

    // 3. 调用 Appender
    callAppenders(event);
}

// callAppenders 会遍历 Logger 的 Appender 链
public void callAppenders(ILoggingEvent event) {
    for (Appender<ILoggingEvent> appender : appenderList) {
        appender.doAppend(event);
    }
    // 如果 additivity 为 true，还调用父 Logger 的 Appender
    if (additive && parent != null) {
        parent.callAppenders(event);
    }
}
```

## 5.3 Appender 职责链

每个 Appender 在输出前会经过过滤器链：

```
appender.doAppend(event)
    |
    v
TurboFilter 决策（LoggerContext 级别）
    |--- ACCEPT -> 继续
    |--- DENY -> 丢弃
    |--- NEUTRAL -> 继续检查
    |
    v
Appender 级别的 Filter 链
    |--- ACCEPT -> 继续
    |--- DENY -> 丢弃
    |--- NEUTRAL -> 使用 Appender 默认行为
    |
    v
Encoder/Layout 格式化
    |
    v
输出到目标
```

## 5.4 Logback 状态报告

配置问题排查：

```xml
<configuration debug="true">
    <!-- debug="true" 会输出 Logback 内部状态信息 -->
</configuration>
```

或在代码中查看：

```java
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
StatusPrinter.print(lc);  // 打印所有状态消息
```

## 5.5 Logback 性能优化原理

- **日志事件延迟创建**：只在需要输出时才创建 `LoggingEvent` 对象
- **StringBuilder 复用**：`MessageFormatter` 使用 ThreadLocal 的 StringBuilder
- **Appender 异步化**：通过 `AsyncAppender` 将 I/O 操作移到后台线程
- **无锁设计**：大部分操作在设计上避免使用同步锁

# 6. 性能优化

## 6.1 AsyncAppender

异步追加日志的核心组件，使用阻塞队列（BlockingQueue）将日志事件交给独立线程处理：

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 引用实际输出的 Appender -->
    <appender-ref ref="ROLLING_FILE" />

    <!-- 队列大小（默认 256） -->
    <queueSize>1024</queueSize>

    <!-- 丢弃策略：队列满时丢弃 TRACE/DEBUG/INFO 级别日志 -->
    <discardingThreshold>0</discardingThreshold>

    <!-- 是否阻塞调用线程（队列满时） -->
    <neverBlock>false</neverBlock>

    <!-- 当队列满时调用者线程是否直接输出 -->
    <!-- 如果 neverBlock=false，调用者会等待 -->
    <!-- 如果 neverBlock=true，超出队列的日志直接被丢弃 -->
</appender>
```

### 6.1.1 AsyncAppender 工作原理

```
生产者线程 (应用)                   消费者线程 (后台)
                +-------------------+
  Logger.log()  |  BlockingQueue    |   Worker Thread
  ------------> |  [event1][event2] | ---------------> Appender.doAppend()
                |  [event3][event4] |
                +-------------------+
```

### 6.1.2 丢弃策略

当队列容量达到 `discardingThreshold`（默认是队列大小的 20%）时：
- TRACE、DEBUG、INFO 级别的日志被直接丢弃
- WARN、ERROR 级别的日志仍会被入队
- 这可以防止应用因日志压力过大而崩溃

```xml
<!-- 不丢弃任何级别（不建议在生产中使用） -->
<discardingThreshold>0</discardingThreshold>

<!-- 当队列剩余 < 10% 时开始丢弃（保守策略） -->
<discardingThreshold>90</discardingThreshold>
```

### 6.1.3 性能测试配置建议

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="ROLLING_FILE" />
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <neverBlock>false</neverBlock>
    <includeCallerData>false</includeCallerData>
    <maxFlushTime>10000</maxFlushTime>
</appender>
```

**重要参数说明**：
- `includeCallerData="false"`：不包含调用者信息（行号、方法名等），提升性能
- `maxFlushTime`：应用关闭时等待队列处理的最大时间（毫秒）

## 6.2 immediateFlush 优化

```xml
<!-- 对于文件 Appender，关闭立即刷新以提升 I/O 性能 -->
<encoder>
    <pattern>...</pattern>
    <immediateFlush>false</immediateFlush>
</encoder>
```

- `immediateFlush="true"`（默认）：每次写入后立即刷新，数据不丢失但性能差
- `immediateFlush="false"`：利用缓冲区批量写入，性能提升明显但可能会丢失少量数据

## 6.3 异步 Appender 配置最佳实践

```xml
<configuration>
    <!-- 同步控制台 Appender（低延迟，开发环境） -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 同步滚动文件 Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <immediateFlush>false</immediateFlush>
        </encoder>
    </appender>

    <!-- 异步包装 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE" />
        <queueSize>2048</queueSize>
        <neverBlock>false</neverBlock>
    </appender>

    <!-- 生产环境：控制台同步 + 文件异步 -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ASYNC_FILE" />
    </root>
</configuration>
```

## 6.4 性能对比数据（参考）

| 场景 | 吞吐量（日志/秒） | 延迟影响 |
|------|------------------|---------|
| 同步 ConsoleAppender | ~50,000 | 高延迟 |
| 同步 FileAppender | ~200,000 | 中延迟 |
| 异步 FileAppender | ~5,000,000+ | 几乎无延迟 |
| 同步 + immediateFlush=true | ~100,000 | I/O 瓶颈 |
| 同步 + immediateFlush=false | ~500,000 | 改进明显 |

# 7. 与 Spring Boot 集成

## 7.1 Spring Boot 默认日志配置

Spring Boot 自动配置 Logback，默认输出到控制台，日志级别为 INFO：

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/myapp.log
    max-size: 10MB
    max-history: 7
  logback:
    rolling:
      max-history: 7
      max-file-size: 10MB
      total-size-cap: 1GB
  charset:
    console: UTF-8
    file: UTF-8
```

## 7.2 logback-spring.xml

Spring Boot 推荐使用 `logback-spring.xml` 替代 `logback.xml`，因为前者可以使用 Spring Boot 的扩展特性：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- Spring Boot 定义的属性 -->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"
                    defaultValue="unknown" />
    <springProperty name="LOG_PATH" source="logging.file.path" defaultValue="./logs" />
    <springProperty name="LOG_LEVEL" source="logging.level.root" defaultValue="INFO" />

    <!-- Spring Boot 的控制台 Appender（已预定义，可直接引用） -->
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />

    <!-- Spring Boot 的文件 Appender（已预定义） -->
    <include resource="org/springframework/boot/logging/logback/file-appender.xml" />

    <!-- 预定义的默认配置 -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />

    <appender name="ROLLING_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 基于 Spring Profile 的条件配置 -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="ROLLING_FILE" />
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="ROLLING_FILE" />
        </root>
    </springProfile>

    <springProfile name="!dev &amp; !prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>
</configuration>
```

## 7.3 Spring Boot 预定义配置参考

Spring Boot 在 classpath 上提供了以下预定义配置片段：

```
org/springframework/boot/logging/logback/
    base.xml              # 基础配置（包含 console + file 的默认值）
    defaults.xml          # 默认属性（日志模式、字符集等）
    console-appender.xml  # 控制台 Appender
    file-appender.xml     # 文件 Appender
```

## 7.4 编程方式修改日志配置

```java
import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.LoggerContext;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class LoggingConfigManager {

    public void setLogLevel(String packageName, String level) {
        LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
        Logger logger = lc.getLogger(packageName);
        logger.setLevel(Level.toLevel(level));
    }

    public void enableDebugForTransaction() {
        // 动态开启事务日志
        setLogLevel("org.springframework.transaction", "DEBUG");
        setLogLevel("org.hibernate.SQL", "DEBUG");
    }
}
```

## 7.5 自定义 Appender 示例

```java
package com.example.logging;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.AppenderBase;
import org.springframework.web.client.RestTemplate;

public class LogServiceAppender extends AppenderBase<ILoggingEvent> {

    private String serviceUrl;
    private RestTemplate restTemplate;

    @Override
    public void start() {
        if (serviceUrl == null) {
            addError("serviceUrl is not set");
            return;
        }
        this.restTemplate = new RestTemplate();
        super.start();
    }

    @Override
    protected void append(ILoggingEvent event) {
        // 将 ERROR 级别日志发送到远程日志收集服务
        if (event.getLevel().isGreaterOrEqual(ch.qos.logback.classic.Level.ERROR)) {
            try {
                LogEntry entry = new LogEntry(
                    event.getTimeStamp(),
                    event.getLevel().toString(),
                    event.getLoggerName(),
                    event.getFormattedMessage()
                );
                restTemplate.postForEntity(serviceUrl, entry, String.class);
            } catch (Exception e) {
                addError("Failed to send log to remote service", e);
            }
        }
    }

    public void setServiceUrl(String serviceUrl) {
        this.serviceUrl = serviceUrl;
    }
}
```

```xml
<appender name="LOG_SERVICE" class="com.example.logging.LogServiceAppender">
    <serviceUrl>http://log-collector.example.com/api/logs</serviceUrl>
</appender>
```

## 7.6 Spring Boot 中排除 Logback 改用 Log4j2

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

# 8. 最佳实践与常见问题

## 8.1 最佳实践

### 8.1.1 配置分离策略

```
resources/
├── logback-spring.xml          # 通用配置（引入 profile 特定文件）
├── logback-spring-dev.xml      # 开发环境
├── logback-spring-test.xml     # 测试环境
├── logback-spring-prod.xml     # 生产环境
```

### 8.1.2 日志文件管理规范

```xml
<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
    <!-- 按天 + 按大小滚动 -->
    <fileNamePattern>${LOG_HOME}/myapp.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
    <maxFileSize>100MB</maxFileSize>     <!-- 单文件最大 100MB -->
    <maxHistory>30</maxHistory>          <!-- 保留 30 天 -->
    <totalSizeCap>10GB</totalSizeCap>    <!-- 总日志大小上限 -->
</rollingPolicy>
```

### 8.1.3 生产环境配置模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds">

    <property name="LOG_HOME" value="./logs" />
    <property name="APP_NAME" value="${spring.application.name:-app}" />

    <!-- 控制台：仅输出 WARN+ -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>WARN</level>
        </filter>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 全量日志（异步） -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{requestId}] - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 错误日志单独存放 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/error.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{requestId}] - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 异步包装 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE" />
        <queueSize>2048</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>false</neverBlock>
    </appender>

    <!-- Logger 配置 -->
    <logger name="com.example" level="INFO" />
    <logger name="org.springframework" level="WARN" />
    <logger name="org.hibernate" level="WARN" />

    <root level="INFO">
        <appender-ref ref="ASYNC_FILE" />
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ERROR_FILE" />
    </root>
</configuration>
```

### 8.1.4 日志消息安全

```java
// 脱敏处理
public class LogSanitizer {
    private static final Pattern SENSITIVE_PATTERN =
        Pattern.compile("(password|creditCard|ssn)=([^&\s]+)");

    public static String sanitize(String message) {
        return SENSITIVE_PATTERN.matcher(message)
            .replaceAll("$1=***");
    }
}
```

### 8.1.5 调试日志事件丢失

使用 native 注解查看 Logback 内部状态：

```xml
<configuration debug="true">
    <!-- 输出 Logback 内部状态到控制台 -->
</configuration>
```

或在代码中检查：

```java
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
StatusManager sm = lc.getStatusManager();
for (Status status : sm.getCopyOfStatusList()) {
    System.err.println(status);
}
```

## 8.2 常见问题

### 8.2.1 logback.xml 没有被加载

**原因**：配置文件不在 classpath 根目录，或文件名不正确。

**解决**：确保 `logback.xml` 或 `logback-spring.xml` 在 `src/main/resources/` 目录下。

### 8.2.2 AsyncAppender 日志丢失

**原因**：
- 队列满时日志被丢弃
- 应用关闭时队列中的日志未被处理

**解决**：
```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE" />
    <queueSize>4096</queueSize>            <!-- 增大队列 -->
    <discardingThreshold>0</discardingThreshold>  <!-- 不丢弃 -->
    <neverBlock>false</neverBlock>          <!-- 阻塞等待 -->
    <maxFlushTime>30000</maxFlushTime>      <!-- 30秒等待队列清空 -->
</appender>
```

### 8.2.3 日志格式错误

```xml
<!-- 错误的：转换字未识别 -->
<pattern>%date %thread %msg%n</pattern>

<!-- 正确的 -->
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
```

### 8.2.4 内存泄漏风险

**MDC 未清理**：
```java
// 错误：未清理 MDC
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    MDC.put("requestId", UUID.randomUUID().toString());
    chain.doFilter(req, res);
    // MDC.clear() 未调用！
}

// 正确：finally 中清理
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    try {
        MDC.put("requestId", UUID.randomUUID().toString());
        chain.doFilter(req, res);
    } finally {
        MDC.clear();
    }
}
```

**SiftingAppender 实例过多**：
- 每个不同的 MDC 值会创建新的 Appender 实例
- 使用 `timeout` 属性回收空闲实例

```xml
<appender name="SIFT" class="ch.qos.logback.classic.sift.SiftingAppender">
    <timeout>30 minutes</timeout>  <!-- 30 分钟无访问则回收 -->
    <maxAppenderCount>1000</maxAppenderCount>  <!-- 最大实例数 -->
</appender>
```

### 8.2.5 Logback 与 Log4j 2 的兼容性问题

- 不能同时包含 `logback-classic.jar` 和 `log4j-slf4j-impl.jar`
- 使用 Log4j 2 时需要排除 Logback

# 9. 面试要点

## 9.1 基础概念

**Q: Logback 的架构是什么？**
A: Logback 分为三个模块：logback-core（核心基础）、logback-classic（SLF4J 原生实现）、logback-access（Servlet 容器集成）。核心组件包括 Logger（日志记录器）、Appender（输出目标）、Layout/Encoder（格式化器）。

**Q: Logback 的 Logger 继承规则是什么？**
A: Logger 基于点号名称形成层级树。子 Logger 继承父 Logger 的日志级别（如果未显式设置）。Appender 默认从父 Logger 继承（additivity=true），设置 additivity=false 可以切断继承。

**Q: Logback 配置文件如何查找？**
A: Logback 按顺序查找：logback.groovy → logback-test.xml → logback.xml。都未找到则使用 BasicConfigurator 输出到控制台（DEBUG 级别）。

## 9.2 高级原理

**Q: AsyncAppender 的工作原理和配置要点？**
A: AsyncAppender 使用 BlockingQueue 缓冲日志事件，由独立的后台线程处理 I/O 操作。关键参数：queueSize（队列大小，默认256）、discardingThreshold（丢弃阈值）、neverBlock（是否阻塞）。WARN/ERROR 级别的日志不会被丢弃。

**Q: Logback 如何实现配置自动重载？**
A: 通过 `scan="true"` 属性开启。Logback 会启动一个新线程定期（默认每分钟）检查配置文件的时间戳。如果发现文件变化，会解析新配置并替换 LoggerContext 中的配置，不丢失任何日志事件。

**Q: Logback 的 MDC 实现原理是什么？**
A: Logback 使用 LogbackMDCAdapter 实现 MDC，内部基于 ThreadLocal 存储 Map。MDC 可以在 PatternLayout 中通过 `%X{key}` 引用。在异步场景中需手动传递 MDC 上下文。

**Q: 什么是 TurboFilter？与普通 Filter 的区别？**
A: TurboFilter 是全局过滤器，挂载在 LoggerContext 级别，在所有日志处理之前执行。它不需要创建日志事件对象即可做出决策，性能更优。普通 Filter 挂载在 Appender 上，需要先创建日志事件。

## 9.3 实践问题

**Q: Spring Boot 中如何配置 Logback？**
A: 可使用 application.yml/logging.* 属性简单配置，或使用 logback-spring.xml 详细配置。logback-spring.xml 支持 Spring Boot 的扩展特性如 `<springProfile>` 和 `<springProperty>`。

**Q: 如何解决 Logback 性能瓶颈？**
A: 主要手段：使用 AsyncAppender 将 I/O 异步化、关闭 immediateFlush、合理配置队列大小、减少调用者数据采集（includeCallerData=false）、使用大小和时间的混合滚动策略。

**Q: Logback 的 FileAppender 和 RollingFileAppender 的区别？**
A: FileAppender 输出到单一文件，不支持滚动。RollingFileAppender 支持根据时间、大小、或两者组合进行文件滚动，是生产环境推荐的方案。

**Q: SiftingAppender 的适用场景？**
A: 典型场景：多租户系统为每个用户创建独立的日志文件，或按交易 ID 拆分日志。基于 MDC 中的值动态创建子 Appender 实例，需要注意内存管理和空闲回收。
