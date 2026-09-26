
# 1. 异常处理架构

## 1.1 异常处理链路

```
请求 → 过滤器链 → 路由转发 → 后端服务
    │                              │
    └── 异常 ──→ ErrorWebExceptionHandler
                       │
                       ├── 记录日志
                       ├── 解析异常类型
                       ├── 确定 HTTP 状态码
                       └── 返回错误响应
```

Spring Cloud Gateway 中的异常处理分为三层：

| 层级 | 处理器 | 职责 |
|------|--------|------|
| WebFlux | `DefaultErrorWebExceptionHandler` | 全局异常处理 |
| 网关 | `ExceptionHandlingWebHandler` | 封装 `DefaultErrorWebExceptionHandler` |
| 过滤器 | 自定义 `onErrorResume` | 按过滤器粒度的异常处理 |

## 1.2 DefaultErrorWebExceptionHandler 源码

```java
public class DefaultErrorWebExceptionHandler
        extends AbstractErrorWebExceptionHandler {

    public DefaultErrorWebExceptionHandler(
            ErrorAttributes errorAttributes,
            WebProperties.Resources resources,
            ApplicationContext applicationContext) {
        super(errorAttributes, resources, applicationContext);
    }

    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(
            ErrorAttributes errorAttributes) {
        // 定义错误路由：处理所有错误
        return RouterFunctions.route(
            RequestPredicates.all(), this::renderErrorResponse);
    }

    private Mono<ServerResponse> renderErrorResponse(
            ServerRequest request) {
        // 获取错误属性
        Map<String, Object> errorProperties =
            getErrorAttributes(request, ErrorAttributeOptions.defaults());

        // 确定 HTTP 状态码
        int httpStatus = getHttpStatus(errorProperties);

        // 返回 JSON 错误响应
        return ServerResponse.status(httpStatus)
            .contentType(MediaType.APPLICATION_JSON)
            .body(BodyInserters.fromValue(errorProperties));
    }

    protected int getHttpStatus(Map<String, Object> errorAttributes) {
        // 从错误属性中提取状态码
        return (int) errorAttributes.get("status");
    }
}
```

# 2. 默认错误响应

## 2.1 默认格式

当网关发生错误时，默认返回如下 JSON：

```json
{
    "timestamp": "2024-01-15T10:30:00.000+00:00",
    "path": "/api/users",
    "status": 404,
    "error": "Not Found",
    "message": "No matching route",
    "requestId": "abc123"
}
```

## 2.2 常见的异常状态码

| 场景 | 状态码 | 说明 |
|------|--------|------|
| 无匹配路由 | 404 | `NotFoundException` |
| 后端服务不可用 | 503 | `ConnectException` |
| 后端超时 | 504 | `TimeoutException` |
| 请求被限流 | 429 | 自定义返回 |
| 请求被熔断 | 503 | 降级处理 |
| 认证失败 | 401 | `ResponseStatusException` |
| 鉴权失败 | 403 | `ResponseStatusException` |
| 请求过大 | 413 | `RequestSizeException` |

# 3. 自定义错误响应

## 3.1 自定义 ErrorAttributes

```java
@Component
public class CustomErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(
            ServerRequest request,
            ErrorAttributeOptions options) {
        Map<String, Object> attributes = super.getErrorAttributes(request, options);

        // 自定义错误响应结构
        Map<String, Object> result = new HashMap<>();
        result.put("code", attributes.get("status"));
        result.put("message", attributes.get("error"));
        result.put("detail", attributes.get("message"));
        result.put("path", attributes.get("path"));
        result.put("timestamp", attributes.get("timestamp"));

        // 去掉默认字段
        result.remove("status");
        result.remove("error");

        // 添加请求 ID 方便排查
        result.put("traceId", request.headers()
            .firstHeader("X-Trace-Id"));

        // 不在 500 错误中暴露详细异常信息
        int status = (int) attributes.get("status");
        if (status >= 500) {
            result.put("detail", "Internal server error");
        }

        return result;
    }
}
```

## 3.2 自定义 ErrorWebExceptionHandler

```java
@Component
@Order(-1)  // 优先级高于默认处理器
public class CustomErrorWebExceptionHandler
        extends AbstractErrorWebExceptionHandler {

    private final ObjectMapper objectMapper;

    public CustomErrorWebExceptionHandler(
            ErrorAttributes errorAttributes,
            WebProperties webProperties,
            ApplicationContext applicationContext,
            ObjectMapper objectMapper) {
        super(errorAttributes, webProperties.getResources(),
            applicationContext);
        this.objectMapper = objectMapper;
    }

    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(
            ErrorAttributes errorAttributes) {
        return RouterFunctions.route(
            RequestPredicates.all(), this::customErrorResponse);
    }

    private Mono<ServerResponse> customErrorResponse(ServerRequest request) {
        Map<String, Object> errorAttributes = getErrorAttributes(
            request, ErrorAttributeOptions.defaults());

        int status = (int) errorAttributes.getOrDefault("status", 500);
        String message = (String) errorAttributes.getOrDefault("error", "Unknown error");

        // 构建统一的错误响应
        ApiErrorResponse response = new ApiErrorResponse(
            status,
            getErrorMessage(status, message),
            request.path(),
            request.headers().firstHeader("X-Trace-Id")
        );

        return ServerResponse.status(status)
            .contentType(MediaType.APPLICATION_JSON)
            .body(BodyInserters.fromValue(response));
    }

    private String getErrorMessage(int status, String defaultMessage) {
        if (status == 404) return "The requested resource was not found";
        if (status == 429) return "Too many requests, please try again later";
        if (status == 503) return "Service temporarily unavailable";
        if (status >= 500) return "Internal server error";
        return defaultMessage;
    }

    record ApiErrorResponse(
        int code,
        String message,
        String path,
        String traceId
    ) {}
}
```

# 4. 过滤器级别的异常处理

## 4.1 统一异常处理过滤器

```java
@Component
@Order(-2)
public class UnifiedExceptionHandlingFilter implements GlobalFilter {

    private static final Logger log = LoggerFactory.getLogger(
        UnifiedExceptionHandlingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange)
            .onErrorResume(throwable -> handleException(exchange, throwable));
    }

    private Mono<Void> handleException(
            ServerWebExchange exchange, Throwable throwable) {
        log.error("Request failed: {} {}", exchange.getRequest().getMethod(),
            exchange.getRequest().getPath(), throwable);

        // 根据异常类型确定状态码和响应
        if (throwable instanceof NotFoundException) {
            return errorResponse(exchange, HttpStatus.NOT_FOUND,
                "Route not found: " + exchange.getRequest().getPath());
        } else if (throwable instanceof ResponseStatusException statusEx) {
            return errorResponse(exchange,
                HttpStatus.valueOf(statusEx.getStatusCode().value()),
                statusEx.getReason());
        } else if (throwable instanceof TimeoutException) {
            return errorResponse(exchange, HttpStatus.GATEWAY_TIMEOUT,
                "Backend service timeout");
        } else if (throwable instanceof ConnectException) {
            return errorResponse(exchange, HttpStatus.SERVICE_UNAVAILABLE,
                "Cannot connect to backend service");
        } else if (throwable instanceof CircuitBreakerStatusCodeException) {
            return errorResponse(exchange, HttpStatus.SERVICE_UNAVAILABLE,
                "Circuit breaker is open");
        } else {
            return errorResponse(exchange,
                HttpStatus.INTERNAL_SERVER_ERROR, "Internal server error");
        }
    }

    private Mono<Void> errorResponse(
            ServerWebExchange exchange,
            HttpStatus status, String message) {
        if (exchange.getResponse().isCommitted()) {
            return Mono.error(new ResponseStatusException(status, message));
        }

        exchange.getResponse().setStatusCode(status);
        exchange.getResponse().getHeaders()
            .setContentType(MediaType.APPLICATION_JSON);

        try {
            Map<String, Object> body = new HashMap<>();
            body.put("code", status.value());
            body.put("message", message);
            body.put("path", exchange.getRequest().getPath().value());
            body.put("timestamp", Instant.now().toString());

            byte[] bytes = new ObjectMapper().writeValueAsBytes(body);
            DataBuffer buffer = exchange.getResponse()
                .bufferFactory().wrap(bytes);
            return exchange.getResponse()
                .writeWith(Mono.just(buffer));
        } catch (JsonProcessingException e) {
            return exchange.getResponse().setComplete();
        }
    }
}
```

## 4.2 忽略某些异常

```java
@Component
@Order(-3)
public class SelectiveExceptionFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange)
            .onErrorResume(throwable -> {
                // 404 和 401 直接传递，不处理
                if (throwable instanceof NotFoundException
                        || throwable instanceof ResponseStatusException statusEx
                            && statusEx.getStatusCode() == HttpStatus.UNAUTHORIZED) {
                    return Mono.error(throwable);  // 传播给下一级处理器
                }
                // 其他异常统一处理
                return handleException(exchange, throwable);
            });
    }

    private Mono<Void> handleException(
            ServerWebExchange exchange, Throwable throwable) {
        // 异常处理逻辑
        return Mono.empty();
    }
}
```

# 5. 响应的安全处理

## 5.1 避免信息泄露

```java
@Component
public class SafeErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(
            ServerRequest request, ErrorAttributeOptions options) {
        Map<String, Object> attrs = super.getErrorAttributes(request, options);

        Throwable error = getError(request);

        // 生产环境隐藏敏感信息
        if (isProduction()) {
            attrs.remove("trace");       // 移除堆栈信息
            if ((int) attrs.get("status") >= 500) {
                attrs.put("message", "Internal server error");
            }
        }

        return attrs;
    }

    private boolean isProduction() {
        String[] activeProfiles = SpringContextHolder
            .getActiveProfiles();
        return List.of(activeProfiles).contains("prod");
    }
}
```

# 6. 错误场景模拟

## 6.1 路由匹配失败

```java
@Service
public class RouteNotFoundSimulator {

    @EventListener(ApplicationReadyEvent.class)
    public void simulateRouteNotFound() {
        // 当未匹配任何路由时，抛出 NotFoundException
        // RoutePredicateHandlerMapping.lookupRoute() 中：
        // .switchIfEmpty(Mono.error(() -> new NotFoundException("No matching route")))
    }
}
```

返回：

```json
{
    "code": 404,
    "message": "Route not found",
    "path": "/nonexistent/path"
}
```

## 6.2 后端超时

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        response-timeout: 5s
```

后端 5 秒未响应返回：

```json
{
    "code": 504,
    "message": "Backend service timeout",
    "path": "/api/slow-service"
}
```

# 7. 最佳实践

## 7.1 统一错误码规范

```java
public enum ErrorCode {
    // 4xx 客户端错误
    BAD_REQUEST(400, "Bad request"),
    UNAUTHORIZED(401, "Authentication required"),
    FORBIDDEN(403, "Access denied"),
    NOT_FOUND(404, "Resource not found"),
    TOO_MANY_REQUESTS(429, "Rate limit exceeded"),

    // 5xx 服务端错误
    INTERNAL_ERROR(500, "Internal server error"),
    BAD_GATEWAY(502, "Bad gateway"),
    SERVICE_UNAVAILABLE(503, "Service unavailable"),
    GATEWAY_TIMEOUT(504, "Gateway timeout"),
    CIRCUIT_BREAKER_OPEN(503, "Circuit breaker is open");

    private final int code;
    private final String defaultMessage;
}
```

## 7.2 错误响应格式

```java
public class ApiResponse<T> {

    private int code;
    private String message;
    private T data;
    private String path;
    private String traceId;
    private long timestamp;

    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.code = 0;
        response.message = "success";
        response.data = data;
        response.timestamp = System.currentTimeMillis();
        return response;
    }

    public static ApiResponse<Void> error(int code, String message,
            String path, String traceId) {
        ApiResponse<Void> response = new ApiResponse<>();
        response.code = code;
        response.message = message;
        response.path = path;
        response.traceId = traceId;
        response.timestamp = System.currentTimeMillis();
        return response;
    }
}
```

## 7.3 生产环境配置

```yaml
server:
  error:
    include-stacktrace: never              # 不输出堆栈信息
    include-message: never                  # 不输出异常详情
    include-exception: false                # 不输出异常类型

spring:
  cloud:
    gateway:
      # 自定义错误处理
      default-filters:
        - name: RemoveResponseHeader
          args:
            name: Server                        # 隐藏服务器信息
```

## 7.4 注意事项

1. **异常分类**：区分客户端异常（4xx）和服务端异常（5xx）
2. **安全隐私**：不在错误响应中暴露堆栈、内部 IP、SQL 等敏感信息
3. **统一格式**：所有错误响应使用统一的 JSON 格式
4. **可追溯性**：错误响应中包含 traceId 便于日志关联
5. **幂等性**：错误响应不应该改变客户端状态
6. **响应已提交**：注意 `response.isCommitted()` 检查，避免写入已提交的响应
