
# 1. 概述

Feign 提供了一套日志接口，可以记录 HTTP 请求/响应的详细信息。

```java
public abstract class Logger {

    public enum Level {
        NONE,    // 不记录日志
        BASIC,   // 仅记录请求方法和 URL、响应状态码、执行时间
        HEADERS, // 记录 BASIC + 请求/响应头
        FULL     // 记录 HEADERS + 请求/响应体
    }

    // 子类需实现的抽象方法
    protected abstract void log(String configKey, String format, Object... args);

    // 记录 HTTP 请求
    protected void logRequest(String configKey, Level logLevel, Request request) { ... }
    // 记录 HTTP 响应
    protected Response logAndRebufferResponse(String configKey, Level logLevel,
            Response response, long elapsedTime) { ... }
}
```

# 2. 日志级别对照

## 2.1 NONE

无任何输出。

## 2.2 BASIC

```
[UserClient#getUser] ---> GET http://user-service/api/users/1 HTTP/1.1
[UserClient#getUser] <--- HTTP/1.1 200 (45ms)
```

## 2.3 HEADERS

```
[UserClient#getUser] ---> GET http://user-service/api/users/1 HTTP/1.1
[UserClient#getUser] Authorization: Bearer xxx
[UserClient#getUser] Accept: application/json
[UserClient#getUser] ---> END HTTP (0-byte body)
[UserClient#getUser] <--- HTTP/1.1 200 (45ms)
[UserClient#getUser] Content-Type: application/json
[UserClient#getUser] Content-Length: 128
[UserClient#getUser] <--- END HTTP (128-byte body)
```

## 2.4 FULL

```
[UserClient#getUser] ---> GET http://user-service/api/users/1 HTTP/1.1
[UserClient#getUser] Authorization: Bearer xxx
[UserClient#getUser] Accept: application/json
[UserClient#getUser] ---> END HTTP (0-byte body)
[UserClient#getUser] <--- HTTP/1.1 200 (45ms)
[UserClient#getUser] Content-Type: application/json
[UserClient#getUser] Content-Length: 97
[UserClient#getUser]
[UserClient#getUser] {"id":1,"name":"John","age":30,"email":"john@example.com"}
[UserClient#getUser] <--- END HTTP (97-byte body)
```

# 3. 配置日志级别

## 3.1 YAML 配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            logger-level: BASIC
          user-service:
            logger-level: FULL

logging:
  level:
    com.example.feign.UserClient: DEBUG   # 必须配 DEBUG
```

**重要**：
- `logger-level` 控制 Feign 发什么日志
- `logging.level` 控制 SLF4J 接收哪些级别的日志
- Feign Logger 输出 `DEBUG` 级别 → `logging.level` 必须设为 `DEBUG` 或更低

## 3.2 代码配置

```java
@Bean
public Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL;
}
```

# 4. 自定义 Logger

## 4.1 示例：脱敏日志

```java
public class MaskedLogger extends feign.Logger {

    private static final Logger log = LoggerFactory.getLogger(MaskedLogger.class);

    @Override
    protected void log(String configKey, String format, Object... args) {
        // 对敏感 header 值脱敏
        Object[] maskedArgs = maskSensitiveHeaders(args);
        log.debug(String.format(methodTag(configKey) + format, maskedArgs));
    }

    private Object[] maskSensitiveHeaders(Object[] args) {
        Object[] result = new Object[args.length];
        for (int i = 0; i < args.length; i++) {
            String s = String.valueOf(args[i]);
            // 脱敏 Authorization / X-Secret
            if (s.startsWith("Authorization: ") || s.startsWith("X-Secret: ")) {
                result[i] = s.replaceAll("(Authorization|X-Secret): (.+)",
                                         "$1: ****");
            } else {
                result[i] = args[i];
            }
        }
        return result;
    }
}
```

## 4.2 示例：将 Feign 日志重定向到独立文件

```java
public class FeignFileLogger extends feign.Logger {

    private final Logger fileLogger = LoggerFactory.getLogger("feign-file");

    @Override
    protected void log(String configKey, String format, Object... args) {
        fileLogger.debug(String.format(methodTag(configKey) + format, args));
    }
}
```

**logback.xml**：

```xml
<appender name="FEIGN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/feign.log</file>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss} %msg%n</pattern>
    </encoder>
</appender>

<logger name="feign-file" level="DEBUG" additivity="false">
    <appender-ref ref="FEIGN_FILE"/>
</logger>
```

# 5. Slf4jLogger 源码解析

Feign 默认使用 `Slf4jLogger`（当 classpath 存在 SLF4J 时）：

```java
public class Slf4jLogger extends Logger {

    private final Logger logger;

    public Slf4jLogger() {
        this(LoggerFactory.getLogger(feign.Logger.class));
    }

    public Slf4jLogger(Logger logger) {
        this.logger = logger;
    }

    @Override
    protected void log(String configKey, String format, Object... args) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format(methodTag(configKey) + format, args));
        }
    }
}
```

# 6. 流程源码

```
SynchronousMethodHandler.executeAndDecode()
  │
  ├── targetRequest(template)   → 构建 Request
  │
  ├── logger.logRequest(configKey, level, request)  ← 记录请求
  │
  ├── long start = System.nanoTime()
  │
  ├── client.execute(request, options)
  │
  ├── long elapsed = System.nanoTime() - start
  │
  ├── response = logger.logAndRebufferResponse(configKey, level, response, elapsed)
  │                                           ← 记录响应（并 rebuffer body）
  │
  └── decoder.decode(response, returnType)
```

**注意**：FULL 级别读响应体后需要 rebuffer，因为 `Response.body()` 是 `InputStream`，只能读一次。

```java
protected Response logAndRebufferResponse(String configKey, Level logLevel,
        Response response, long elapsedTime) throws IOException {
    // 记录响应行和响应头
    log(configKey, "<--- HTTP/1.1 %s (%sms)", response.status(), elapsedTime);
    // ...

    if (logLevel == Level.FULL && response.body() != null) {
        // 读取 body 用于日志
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        log(configKey, "%s", decodedBody);
        // 重建 body，使后续 Decoder 可以再次读取
        return response.toBuilder()
                .body(bodyData)
                .build();
    }
    return response;
}
```

# 7. 生产环境日志建议

| 环境 | 推荐级别 | 原因 |
|------|----------|------|
| **开发** | FULL | 方便排查请求/响应细节 |
| **测试/预发** | HEADERS | 关注头信息，避免敏感数据泄露 |
| **生产** | BASIC 或 NONE | 记录请求/响应时间即可，避免性能开销和敏感数据泄露 |

# 8. 常见问题

**Q：配置了 logger-level: FULL 但没有日志输出？**

检查 `logging.level` 是否设为 `DEBUG`（Feign Logger 输出 DEBUG 级别）。

```yaml
logging:
  level:
    com.example.feign: DEBUG
```

**Q：FULL 日志对性能有影响吗？**

有影响。响应体需要全部读入内存才能打日志，大响应（如文件下载）会有明显性能开销。生产环境慎用。
