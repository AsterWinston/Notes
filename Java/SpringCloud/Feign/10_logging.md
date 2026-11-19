# 1. 概述

## 1.1 两套日志的分工入口

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

四级是**递进包含**关系（FULL ⊃ HEADERS ⊃ BASIC），配置时只需选档，没有组合。日志行的前缀 `[UserClient#getUser]` 来自 `methodTag(configKey)`——configKey 即 `5_contract.md` §3.2 的"方法身份证"，所以按方法名 grep 日志、按接口聚合错误，天然与熔断器/监控的命名对齐。Logger 实例是**每个客户端各一个**（构建代理时注入，级别取该客户端上下文的 `Logger.Level` Bean），这保证了多客户端可以把日志档位配成不同值互不干扰。

# 2. 日志级别对照

## 2.1 NONE

无任何输出。

## 2.2 BASIC

```text
[UserClient#getUser] ---> GET http://user-service/api/users/1 HTTP/1.1
[UserClient#getUser] <--- HTTP/1.1 200 (45ms)
```

## 2.3 HEADERS

```text
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

```text
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

一个必须先看到的风险：示例的 HEADERS 档输出里就有 `Authorization: Bearer xxx` 原文——**令牌进日志从 HEADERS 级就开始，不用等到 FULL**。合规场景下要么给这类头做脱敏（§4.1），要么把日志档停在 BASIC 并接受"头信息靠抓包排查"。响应侧还有一行隐藏信息量：`--->/END` 之间的耗时与 `<---` 括号里的 `(45ms)` 一起看，可以把"慢在排队/慢在传输/慢在下游"分开归因。

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

两个开关缺一不可，是"配了 FULL 却没日志"的头号原因。注意 `logging.level` 处的名字是**客户端接口的包/类名**：Spring Cloud 构建代理时注入的默认 Logger 是 `Slf4jLogger(targetType)`，SLF4J logger 名就是接口全限定名（§5 详述），按 `com.example.feign` 整包开 DEBUG 即可覆盖包内所有客户端。

## 3.2 代码配置

```java
@Bean
public Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL;
}
```

`Logger.Level` 是组件 Bean，作用域规则同 `4_configuration.md`：写在按服务的配置类里只影响该客户端；被主容器扫到则全局 FULL（叠加 §2.3 的令牌泄露风险与 §6 的 rebuffer 内存开销，这个污染尤其危险）。

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

在 `log()` 层做拦截的好处是**一处兜住全部四级输出**（logRequest/logAndRebufferResponse 内部最终都走它）；但注意按行前缀匹配的脆弱性——FULL 级的请求体是一行整 JSON，身份证、手机号这类**体内敏感字段**要另在 Encoder/Decoder 层处理或限制 FULL 只在开发档启用。注册为 `Logger` 组件 Bean 即可按客户端替换（同 §3.2 作用域规则）。

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

独立 appender 的实用配套：`<rollingPolicy>` 必配（FULL 级单请求 4-6 行、报文全量，不设滚动策略的 feign.log 是磁盘事故）；`additivity="false"` 防止双写主日志；若还要保留"主日志里只留 BASIC"的能力，可以自定义 Logger 按级别分派两个底层 logger——比再造 appender 更省事。

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

## 5.1 两个构造器的差别正是 §3.1 配置名的由来

无参构造把 SLF4J logger 名固定成 `feign.Logger`——那是**纯 OpenFeign**（`Feign.builder()` 直接构建）时的归属；而 Spring Cloud 的 `FeignClientFactoryBean` 在没有现成 `Logger` Bean 时注入 `new Slf4jLogger(targetType)`，logger 名变成**客户端接口类**。于是同一个类有两种"该给谁开 DEBUG"的答案：Spring Cloud 工程按接口包配（§3.1），裸 Feign 按 `feign.Logger` 配。`isDebugEnabled` 守卫也值得留意：级别没开时连字符串 format 都不发生，这就是"BASIC 档常驻的生产开销几乎可忽略、但 FULL 依然贵"的原因——开销差异不在打不打日志，而在 §6 的 body 处理。

# 6. 流程源码

```text
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

这条流程解释了三件事：其一，`(45ms)` 只统计 `client.execute`，**不含** Encoder/Decoder 时间，"Feign 日志 RT 比 APM 低"是口径差异不是 bug；其二，`logRequest` 在 execute 之前，所以网络失败时日志里只有 `--->` 半边没有 `<---`（§8 排障口诀）；其三，FULL 的代价是把响应体**完整搬进堆内存**再重建 Response——大响应等于一次额外的全量缓冲，这就是文件下载/大列表接口必须避开 FULL 的底层原因（rebuffer 与 `6_encoder-decoder.md` §5 的"流只能读一次"是同一件事的两面）。

# 7. 生产环境日志建议

| 环境 | 推荐级别 | 原因 |
|------|----------|------|
| **开发** | FULL | 方便排查请求/响应细节 |
| **测试/预发** | HEADERS | 关注头信息，避免敏感数据泄露 |
| **生产** | BASIC 或 NONE | 记录请求/响应时间即可，避免性能开销和敏感数据泄露 |

补一个把两级开关用起来的操作模式：`logger-level` 在代理构建时定格，改它要重启；`logging.level` 可以走 actuator `/loggers` **运行时热调**。所以"生产常备 BASIC、故障期间对单接口临时 FULL"的正确姿势不是改 yml 重启，而是提前把 `logger-level` 配到 FULL、`logging.level` 停在 INFO——平时零输出零开销，排障时一条 POST 把特定接口包切到 DEBUG，FULL 立即生效且影响面精确。

# 8. 常见问题

**Q：配置了 logger-level: FULL 但没有日志输出？**

检查 `logging.level` 是否设为 `DEBUG`（Feign Logger 输出 DEBUG 级别）。

```yaml
logging:
  level:
    com.example.feign: DEBUG
```

若仍无输出，按 §5.1 核对 logger 名是否挂错（接口包 vs `feign.Logger`），再确认没有第二个同名 `Logger` Bean 把默认实现顶掉（脱敏 Logger 忘了调 super 输出、或 JUL/SLF4J 桥接缺失也在候选清单里）。

**Q：FULL 日志对性能有影响吗？**

有影响。响应体需要全部读入内存才能打日志，大响应（如文件下载）会有明显性能开销。生产环境慎用。

**Q：日志里只有 `--->` 没有 `<---`，是什么故障？**

说明失败发生在 `client.execute` 内部（超时、连接拒绝、DNS），响应根本没回来——对应的异常就是 `RetryableException`（status=-1，见 `8_error-handling.md` §8 图外路径与 `9_retry.md` §5）。成片的半边箭头 + 特定目标服务，基本可以定位到那台下游的网络或容量问题，比看异常栈更快。

**Q：想只记录失败调用的完整报文，成功调用记 BASIC？**

Feign 的 Level 是无条件按档输出，做不到"按结果分级"；组合方案是 `logger-level: NONE` + 在 `ErrorDecoder`（§3）里对失败响应打结构化错误日志，成功路径交给 APM/Micrometer 指标（`21_logging_monitoring.md`）。
