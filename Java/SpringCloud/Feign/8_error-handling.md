
# 1. 概述

`ErrorDecoder` 是 Feign 处理**非 2xx HTTP 响应**的核心组件。当服务端返回 4xx/5xx 状态码时，Feign 不会直接返回业务对象，而是调用 `ErrorDecoder` 将 HTTP 响应**解码为异常**抛出。

```java
public interface ErrorDecoder {
    Exception decode(String methodKey, Response response);
}
```

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

# 5. FeignException 家族

```
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

## 6.2 FeignException 关键方法

```java
FeignException e = ...;
e.status();           // HTTP 状态码，-1 表示连接失败
e.getMessage();       // 异常消息
e.contentUTF8();      // 响应体（UTF-8）
e.request();          // 关联的 Feign Request 对象
e.responseHeaders();  // 响应头 Map
```

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

# 8. 异常处理链路总结

```
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

# 9. 最佳实践

1. **解析服务端业务异常** — 通过自定义 ErrorDecoder 将服务端的错误响应转为业务异常
2. **保留默认兜底** — 解析失败时回退到 `ErrorDecoder.Default`
3. **连接失败的 status = -1** — 需要在异常处理中单独处理
4. **RetryableException 语义** — 如果某个错误是可以重试的（如 503），让 ErrorDecoder 抛出 `RetryableException`
5. **注意响应体读取** — `response.body()` 只能读取一次，ErrorDecoder 中读取后就不会再被 Decoder 读取
