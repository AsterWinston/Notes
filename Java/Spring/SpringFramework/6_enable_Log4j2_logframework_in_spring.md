# 1. 引入Log4j2日志框架
```
<!--log4j2的依赖-->
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-core</artifactId>
  <version>2.19.0</version>
</dependency>
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-slf4j2-impl</artifactId>
  <version>2.19.0</version>
</dependency>

```
# 2. 提供log4j2.xml配置文件
```
在类根路径下提供 log4j2.xml 配置文件
配置文件的文件名固定为：log4j2.xml，且文件必须放到类根路径下
配置文件的文件名是固定的，位置也是固定的，必须在类的根目录及其子目录下。

<?xml version="1.0" encoding="UTF-8"?>

<configuration>
    
    <loggers>
        <!--
            level指定日志级别，从低到高的优先级：
                ALL < TRACE < DEBUG < INFO < WARN < ERROR < FATAL < OFF
            达到相应的级别才会输出日志信息
            级别越低输出的日志信息越多
        -->
				<!-- 日志输出级别为 DEBUG，该级别之下的日志不会输出 -->
        <root level="DEBUG">
            <appender-ref ref="spring6log"/>
        </root>
    </loggers>
    
    <appenders>
        <!--输出日志信息到控制台-->
        <console name="spring6log" target="SYSTEM_OUT">
            <!--控制日志输出的格式-->
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss SSS} [%t] %-3level %logger{1024} - %msg%n"/>
        </console>
    </appenders>

</configuration>

```
# 3. 使用日志框架
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


@org.junit.Test
public void testLog() {
    // 获取日志记录器对象（记录日志需要一个日志记录器）
    // 获取Test类的日志记录器对象，只要是Test类中的代码执行记录日志的话，就输出相关的日志信息。
    Logger logger = LoggerFactory.getLogger("cw.spring.study.test.Test");
    
    // 记录日志，记录不同的级别的日志
    // 输出日志时，根据配置文件中设置的输出日志级别进行输出
    logger.info("我是一条消息");
    logger.debug("我是一条调试信息");
    logger.error("我是一条错误信息");
    logger.trace("我是一条查找追踪信息");
}
```

# 4. 源码解析：Spring 的日志体系

## 4.1. SLF4J + 桥接模式

```
Spring 本身不实现日志功能，而是通过 SLF4J（Simple Logging Facade for Java）来桥接：

┌─────────────────────────────────────────┐
│         Spring 框架代码                   │
│  使用 org.apache.commons.logging.Log    │
│  （JCL — Jakarta Commons Logging）       │
└────────────────┬────────────────────────┘
                │
                ▼  Spring 5+ 后内部委派
┌─────────────────────────────────────────┐
│         SLF4J（门面）                     │
│  org.slf4j.Logger + LoggerFactory       │
└────────────────┬────────────────────────┘
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Log4j2   │ │ Logback  │ │ JUL      │
│ (实现)   │ │ (实现)   │ │ (实现)   │
└──────────┘ └──────────┘ └──────────┘

这就是经典的门面模式（Facade Pattern）+ 桥接模式：
- SLF4J 是门面，提供统一的日志 API
- 各个日志框架是实现，通过桥接器（log4j-slf4j2-impl）适配到 SLF4J
```

## 4.2. Spring 内部日志的实现：Spring-JCL 桥

```
Spring 5+ 内部将 JCL 调用桥接到 SLF4J：

// Spring 的 spring-jcl 模块中：
// 传统的 JCL 调用 log.info("xxx") 内部被重定向到 SLF4J

// org.apache.commons.logging.LogAdapter 的核心逻辑：
final class LogAdapter {
    public static Log createLog(String name) {
        // 按优先级尝试找到可用的日志实现：
        // 1. SLF4J（通过 log4j-slf4j2-impl 桥接到 Log4j2）
        // 2. Log4j 2.x API
        // 3. java.util.logging（JUL）
        for (LogApi api : LogApi.values()) {
            Log log = api.createLog(name);
            if (log != null) return log;
        }
        // 4. 兜底：no-op（不输出任何日志）
        return new NoOpLog(name);
    }
}

// 所以在 Spring 源码中看到的日志调用：
// protected final Log logger = LogFactory.getLog(getClass());
// logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
// 最终都通过 SLF4J → Log4j2 输出
```

## 4.3. 日志级别映射

```
SLF4J 的日志级别与 Log4j2 日志级别的对应关系：

SLF4J Level    │ Log4j2 Level  │ 用途
───────────────┼───────────────┼──────────
trace          │ TRACE         │ 最详细的追踪信息
debug          │ DEBUG         │ 调试信息
info           │ INFO          │ 普通信息（Spring Bean 创建日志在此级别）
warn           │ WARN          │ 警告
error          │ ERROR         │ 错误

log4j2.xml 中的日志级别配置：
<Root level="info">  ← 只输出 info 及以上级别的日志
  <AppenderRef ref="console"/>
</Root>

可以通过包名精确控制日志级别：
<Logger name="org.springframework.beans" level="debug"/>  
  ← 只对 Spring Bean 相关日志输出 debug 级别
<Logger name="org.springframework.jdbc" level="trace"/>   
  ← JDBC 操作输出最详细的 trace 日志（包含 SQL 参数）
```

## 4.4. 设计思想总结

1. 门面模式（Facade Pattern）：
   SLF4J 作为日志门面，让 Spring 不需要绑定具体的日志实现。
   用户可以根据需要切换 Log4j2、Logback 等，而无需改动业务代码。

2. 桥接模式（Bridge Pattern）：
   log4j-slf4j2-impl 是 Log4j2 到 SLF4J 的桥接器，
   它将 SLF4J 的 API 调用转发给 Log4j2 的实现。

3. 内部日日至外部日志的演进：
   Spring 早期使用 JCL（Jakarta Commons Logging），
   Spring 5 后通过 spring-jcl 模块内部桥接到 SLF4J。
   这体现了"最小依赖"原则和"可替换性"设计。
