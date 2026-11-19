# 1. 概述

## 1.1 ErrorDecoder 的坐标

`ErrorDecoder` 是 Feign 处理**非 2xx HTTP 响应**的核心组件。当服务端返回 4xx/5xx 状态码时，Feign 不会直接返回业务对象，而是调用 `ErrorDecoder` 将 HTTP 响应**解码为异常**抛出。

```java
public interface ErrorDecoder {
    Exception decode(String methodKey, Response response);
}
```

接口的返回值类型是 `Exception` 而不是"抛异常"，这是 Feign 刻意保留的自由度：ErrorDecoder 既能把错误翻译成异常，也能（配合 `decode404` 之外的手段）返回 `RetryableException` 把决定权交给重试体系（§9 与 `9_retry.md` 的协同点）。同时注意方法签名里**没有目标类型信息**——它只能决定"抛出什么"，无法决定"返回什么"，想让 404 变成本地对象必须走 `decode404` 而不是 ErrorDecoder。

# 2. ErrorDecoder.Default 源码

```java
public static class Default implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        FeignException exception = errorStatus(methodKey, response);
        // 提取响应体作为异常消息
        String body = response.body() != null
                ? Util.toString(response.body().asReader(UTF_8))
                : "";
        return new FeignException(
            response.status(),
            exception.getMessage() + "\n" + body,
            response.request(),
            response.headers(),
            body.getBytes(UTF_8)
        );
    }

    // 根据 HTTP 状态码映射 FeignException 子类
    private FeignException errorStatus(String methodKey, Response response) {
        int status = response.status();
        if (status >= 400 && status < 500) {
            return new FeignException.FeignClientException(status, message, ...);
        }
        if (status >= 500) {
            return new FeignException.FeignServerException(status, message, ...);
        }
        return new FeignException(status, message, ...);
    }
}
```

**关键点**：
- 400-499 → `FeignException.FeignClientException`
- 500-599 → `FeignException.FeignServerException`
- 将响应 body 作为异常消息写入

4xx/5xx 两级抽象的用意是让上层的通用处理不必枚举具体状态码：熔断统计里"哪些该算失败"可以按 Client/Server 异常一刀切（4xx 多为调用方参数/权限问题，不该拖垮熔断；5xx 是被调方故障，应当计入，`14_circuit-breaker.md` 的 `record-exceptions` 口径同理）。另外"body 拼进 message"在排障时非常友好，但 FULL 报文可能连带敏感数据进日志，高安全场景要自定义 ErrorDecoder 收敛 message。

# 3. 自定义 ErrorDecoder

## 3.1 示例：解析服务端业务异常

```java
public class BusinessErrorDecoder implements ErrorDecoder {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private final ErrorDecoder defaultDecoder = new ErrorDecoder.Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            String body = Util.toString(response.body().asReader(StandardCharsets.UTF_8));
            ErrorResponse errorResponse = objectMapper.readValue(body, ErrorResponse.class);
            return new BusinessException(
                    errorResponse.getCode(),
                    errorResponse.getMessage(),
                    response.status()
            );
        } catch (Exception e) {
            // 解析失败时回退到默认处理
            return defaultDecoder.decode(methodKey, response);
        }
    }
}
```

统一错误响应 DTO：

```java
public class ErrorResponse {
    private int code;
    private String message;
    private String traceId;
    // getter/setter...
}
```

自定义业务异常：

```java
public class BusinessException extends RuntimeException {
    private final int code;
    private final int httpStatus;

    public BusinessException(int code, String message, int httpStatus) {
        super(message);
        this.code = code;
        this.httpStatus = httpStatus;
    }
}
```

示例里的回退写法值得展开成三条纪律：**body 为 null 要短路**（HEAD 类响应、部分网关的 5xx 空 body，`asReader` 直接 NPE 后掉进回退分支会重复消费已读流，正确做法是判空后再读）；**traceId 要留在异常里**——它是把调用方异常与提供方日志对起来的唯一线索；**回退对象要新建而不是复用已读空的 response**（body 流只能读一次，回退前把 body 字符串缓存、用 `response.toBuilder().body(bytes)` 重建，Default 才能再拿到内容）。

## 3.2 针对特定 HTTP 状态码

```java
public class StatusAwareErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        switch (response.status()) {
            case 401:
                return new UnauthorizedException("认证失败，请重新登录");
            case 403:
                return new ForbiddenException("暂无访问权限");
            case 404:
                return new NotFoundException("请求的资源不存在: " + methodKey);
            case 429:
                return new RateLimitException("请求频率过高，请稍后再试");
            case 503:
                return new ServiceUnavailableException("服务暂时不可用");
            default:
                return new ErrorDecoder.Default().decode(methodKey, response);
        }
    }
}
```

按状态码分类的下一步通常是给异常定义**处置策略**：哪些直接上抛给全局处理器转译成用户提示（401/403/404）、哪些应交给重试（429/503，转 `RetryableException` 见 `9_retry.md` §6）、哪些应计入熔断统计。三个异常族（业务码版 §3.1、状态码版、重试信号版）不要各自为战地叠加注册——ErrorDecoder 每个客户端只有一个实例，所有策略要在一个实现的入口里按"先状态码、再 body 可解析性、最后兜底 Default"的顺序短路。

# 4. 注册 ErrorDecoder

## 4.1 全局注册

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            error-decoder: com.example.feign.BusinessErrorDecoder
```

## 4.2 针对特定 FeignClient

```java
@FeignClient(name = "user-service", configuration = UserFeignConfig.class)
public interface UserClient { }
```

```java
public class UserFeignConfig {
    @Bean
    public ErrorDecoder errorDecoder() {
        return new BusinessErrorDecoder();
    }
}
```

## 4.3 通过 Feign.Builder 注册

```java
Feign.builder()
    .errorDecoder(new BusinessErrorDecoder())
    .target(UserClient.class, "http://localhost:8080");
```

两种 Spring 入口的差异回到组件覆盖的通用规则：yml 只支持无参构造类（不能注入 ObjectMapper 定制），@Bean 方式按 `4_configuration.md` §6 的优先级压过 yml。**同一个客户端两处都配会静默选择 @Bean**，排查"配置了却没生效"时先确认另一处是否重复注册。

# 5. FeignException 家族

```text
RuntimeException
  └── FeignException
        ├── FeignClientException    (4xx)
        │     ├── NotFound          (404)
        │     ├── Unauthorized      (401)
        │     ├── Forbidden         (403)
        │     ├── MethodNotAllowed  (405)
        │     └── ...
        ├── FeignServerException    (5xx)
        │     ├── InternalServerError (500)
        │     ├── ServiceUnavailable (503)
        │     └── ...
        ├── RetryableException      (可重试异常，Retryer 使用)
        └── FeignException          (其他)
```

捕获时的两个层级选择：面向"所有远程调用失败"用 `FeignException`（含网络故障的 status=-1 成员）；面向"HTTP 语义错误"用两个子类。**RetryableException 也在这个家族里**，但它对业务是透明的——默认 Retryer（`NEVER_RETRY`）下它会原样上抛，业务层一般不需要单独 catch，除非你显式实现了重试（见 §9 第 4 条的语义）。异常携带的 `errorBody()`（字节数组）比 getMessage() 更适合程序化处理错误体，message 里的拼接体适合人读不适合再解析 JSON——这是 §3.1 自定义解析与默认拼接分工的原因。

# 6. 处理 FeignException

## 6.1 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(FeignException.class)
    public ResponseEntity<ErrorResponse> handleFeignException(FeignException e) {
        int status = e.status();
        if (status == -1) {
            // 连接失败（网络不可达、超时等）
            return ResponseEntity.status(503)
                    .body(new ErrorResponse(503, "上游服务不可达"));
        }
        return ResponseEntity.status(status)
                .body(new ErrorResponse(status,
                        "调用上游服务失败: " + e.getMessage()));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        return ResponseEntity.status(e.getHttpStatus())
                .body(new ErrorResponse(e.getCode(), e.getMessage()));
    }
}
```

示例透传上游状态码的做法要慎用：上游的 500 直接透传会把你自己的调用方也拖进"服务端故障"告警，而多数场景应降级为**语义明确的本层错误**（把上游故障包装成本服务的 502/503 + 内部错误码），告警口径和对外契约才不混。另外注意处理器优先级：自定义 `BusinessException` 走专属方法，说明你的 ErrorDecoder 已把 HTTP 错误"翻译"成了业务语言——若 Advice 里还留着一堆按 `FeignException.status()` 的分支，两套语言并存，建议统一在 ErrorDecoder 完成翻译。

## 6.2 FeignException 关键方法

```java
FeignException e = ...;
e.status();           // HTTP 状态码，-1 表示连接失败
e.getMessage();       // 异常消息
e.contentUTF8();      // 响应体（UTF-8）
e.request();          // 关联的 Feign Request 对象
e.responseHeaders();  // 响应头 Map
```

`status() == -1` 是纯网络层失败（连接超时、读超时、DNS、连接被重置），此时没有响应也就没有 header/body；日志里它和 HTTP 错误码的处置价值完全不同：-1 高发指向网络/容量，5xx 高发指向提供方，4xx 突发多半是你发布改动破坏了契约。

# 7. decode404 配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:
            decode404: true
```

当 `decode404 = true` 时，404 响应**不会**触发 `ErrorDecoder`，而是走 `Decoder` 的正常流程，可以返回 `null` 或空对象。

```java
// decode404 = true 时，404 响应会走到 Decoder.decode()
// 可以用 Optional 接收
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    Optional<User> getUser(@PathVariable Long id);
}
```

`Optional` 返回类型与 `decode404` 是配套设计：解码结果为空时返回 `Optional.empty()`，"记录不存在"从异常路径变成显式业务分支。两个边界要认清：**空 body 的 404** 对多数返回类型会解码成 `null`（基本类型签名直接 NPE，方法返回 `User` 时调用侧判空是必须的）；`dismiss404`（§速查表见 `4_configuration.md`）更进一步跳过解码直接 null。开启前必须与提供方确认"404 是否只表示资源不存在"——网关路由缺失、路径拼错同样返回 404，`decode404` 会把这类真故障全部吞成"查无此人"。

# 8. 异常处理链路总结

```text
HTTP Response
  │
  ├── 2xx → Decoder.decode() → 返回业务对象
  │
  └── 非 2xx
        │
        ├── decode404=true & 404 → Decoder.decode() → 返回 null/空
        │
        └── 其他 → ErrorDecoder.decode() → 抛出异常
              │
              ├── FeignClientException (4xx)
              ├── FeignServerException (5xx)
              ├── RetryableException (触发 Retryer)
              └── 自定义 BusinessException
                    │
                    ▼
              @RestControllerAdvice 统一处理
```

图外还有一条不进 ErrorDecoder 的路径值得画上：**IOException 类网络失败**由 `SynchronousMethodHandler` 直接包成 `RetryableException`（status=-1），ErrorDecoder 对它没有介入机会——所以"把所有异常都翻译成 BusinessException"的自定义 ErrorDecoder 并不能覆盖超时/断连场景，Advice 里对 `FeignException`（-1）的分支处理永远必要。

# 9. 最佳实践

1. **解析服务端业务异常** — 通过自定义 ErrorDecoder 将服务端的错误响应转为业务异常
2. **保留默认兜底** — 解析失败时回退到 `ErrorDecoder.Default`
3. **连接失败的 status = -1** — 需要在异常处理中单独处理
4. **RetryableException 语义** — 如果某个错误是可以重试的（如 503），让 ErrorDecoder 抛出 `RetryableException`
5. **注意响应体读取** — `response.body()` 只能读取一次，ErrorDecoder 中读取后就不会再被 Decoder 读取
6. **翻译层只做一次** — HTTP→业务异常的映射集中在 ErrorDecoder，Advice 不再按 status 重复分支；对外响应码由本服务语义决定，不盲目透传上游状态码；
7. **给熔断留口径** — 自定义异常继承 `RuntimeException` 会被 Resilience4j 默认计入失败率，"参数校验类"的 4xx 翻译建议用 `ignorableExceptions`/`record-exceptions` 白名单排除，避免下游把上游的正常业务拒绝当成故障熔断（详见 `14_circuit-breaker.md`）。
