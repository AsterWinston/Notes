
# 1. 异常处理体系概览

Zuul 的异常处理涉及多个层面：

```
                    ┌──────────────────────┐
                    │   应用层异常           │
                    │   Filter.run() 抛出    │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   FilterProcessor     │
                    │   捕获 → ctx.setThrowable│
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   ZuulServlet         │
                    │   跳转 error() 方法    │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   ErrorFilter 链      │
                    │   SendErrorFilter 兜底│
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   Spring Boot         │
                    │   ErrorController     │
                    └──────────────────────┘
```

# 2. ZuulException 体系

`ZuulException` 是 Zuul 中的核心异常类：

```java
public class ZuulException extends Exception {
    public int nStatusCode;      // HTTP 状态码
    public String errorCause;    // 错误原因描述

    public ZuulException(Throwable throwable, int nStatusCode, String errorCause) {
        super(errorCause, throwable);
        this.nStatusCode = nStatusCode;
        this.errorCause = errorCause;
    }

    public ZuulException(String errorCause, int nStatusCode, String msg) {
        super(msg);
        this.nStatusCode = nStatusCode;
        this.errorCause = errorCause;
    }
}
```

## 2.1 异常分类

| 异常类型 | 典型原因 | HTTP 状态码 |
|----------|---------|-------------|
| `ZuulException` | 路由未找到、过滤器执行失败 | 500/404 |
| `SocketTimeoutException` | 后端响应超时 | 504 |
| `ConnectException` | 后端服务不可达 | 502 |
| `HystrixRuntimeException` | 熔断器打开 | 503 |
| `HystrixTimeoutException` | Hystrix 超时 | 504 |
| `ClientException` | 客户端请求错误 | 400 |
| `NestedServletException` | Servlet 层异常 | 500 |

# 3. SendErrorFilter 源码分析

`SendErrorFilter` 是 Zuul 的错误处理兜底过滤器（order=0）：

```java
public class SendErrorFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getThrowable() != null
            && !ctx.getBoolean("sendErrorFilter.ran", false);
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        Throwable throwable = ctx.getThrowable();
        HttpServletRequest request = ctx.getRequest();
        HttpServletResponse response = ctx.getResponse();

        // 1. 提取异常对应的 HTTP 状态码
        int statusCode = extractStatusCode(throwable);

        // 2. 设置 request attributes（符合 Servlet 错误规范）
        request.setAttribute("javax.servlet.error.status_code", statusCode);
        request.setAttribute("javax.servlet.error.exception", throwable);
        request.setAttribute("javax.servlet.error.message", throwable.getMessage());
        request.setAttribute("javax.servlet.error.request_uri",
            request.getRequestURI());

        // 3. 标记已执行，防止重复（关键设计！）
        ctx.set("sendErrorFilter.ran", true);

        // 4. 获取 Spring Boot 配置的 ErrorPage 路径
        //    默认 /error，最终由 BasicErrorController 处理
        String errorPath = extractErrorPath();

        // 5. 内部转发到错误处理路径
        try {
            RequestDispatcher dispatcher = request.getRequestDispatcher(errorPath);
            if (dispatcher != null) {
                dispatcher.forward(request, response);
            }
        } catch (Exception e) {
            // 转发失败则直接发送错误状态码
            response.sendError(statusCode);
        }

        return null;
    }

    private int extractStatusCode(Throwable throwable) {
        if (throwable instanceof ZuulException) {
            return ((ZuulException) throwable).nStatusCode;
        }
        if (throwable instanceof SocketTimeoutException) {
            return 504;
        }
        if (throwable instanceof ConnectException) {
            return 502;
        }
        return 500;
    }
}
```

## 3.1 设计亮点：防止重复执行

```java
ctx.set("sendErrorFilter.ran", true);
```

这是 `SendErrorFilter` 的关键设计。在 `shouldFilter()` 中检查：

```java
return ctx.getThrowable() != null
    && !ctx.getBoolean("sendErrorFilter.ran", false);
```

防止在执行 Error 过滤器的过程中再次触发新的 Error 过滤器调用，从而避免死循环。

# 4. Spring Boot ErrorController 集成

当 `SendErrorFilter` 将请求转发到 `/error` 后，由 Spring Boot 的 `BasicErrorController` 接管：

```java
// Spring Boot BasicErrorController
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {

    // HTML 错误页面
    @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
    public ModelAndView errorHtml(HttpServletRequest request,
                                   HttpServletResponse response) {
        HttpStatus status = getStatus(request);
        Map<String, Object> model = getErrorAttributes(request,
            isIncludeStackTrace(request, MediaType.TEXT_HTML));
        response.setStatus(status.value());
        ModelAndView modelAndView = resolveErrorView(request, response, status, model);
        return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
    }

    // JSON 错误响应
    @RequestMapping
    public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
        HttpStatus status = getStatus(request);
        Map<String, Object> body = getErrorAttributes(request,
            isIncludeStackTrace(request, MediaType.ALL));
        return new ResponseEntity<>(body, status);
    }
}
```

默认情况下，错误响应包含：

```json
{
    "timestamp": "2024-01-15T10:30:00.000+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "message": "...",
    "path": "/user/profile"
}
```

# 5. 自定义异常响应

## 5.1 自定义 ErrorAttributes

```java
@Component
public class CustomErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(
            WebRequest webRequest, ErrorAttributeOptions options) {

        Map<String, Object> errorAttributes =
            super.getErrorAttributes(webRequest, options);

        // 自定义错误响应结构
        Map<String, Object> customError = new LinkedHashMap<>();
        customError.put("code", errorAttributes.get("status"));
        customError.put("message", errorAttributes.get("error"));

        // 获取 RequestContext 中的自定义错误信息
        RequestContext ctx = RequestContext.getCurrentContext();
        if (ctx != null && ctx.getThrowable() != null) {
            Throwable cause = ctx.getThrowable();
            if (cause instanceof ConnectException) {
                customError.put("message", "后端服务连接失败");
            } else if (cause instanceof SocketTimeoutException) {
                customError.put("message", "后端服务响应超时");
            }
        }

        customError.put("path", errorAttributes.get("path"));
        customError.put("timestamp", errorAttributes.get("timestamp"));

        return customError;
    }
}
```

## 5.2 禁用白标签错误页面

```yaml
server:
  error:
    whitelabel:
      enabled: false  # 禁用 Spring Boot 默认白标签错误页
```

## 5.3 自定义 ErrorController

```java
@RestController
public class CustomErrorController implements ErrorController {

    @RequestMapping("/error")
    public ResponseEntity<Map<String, Object>> handleError(
            HttpServletRequest request) {

        HttpStatus status = getStatus(request);
        Map<String, Object> body = new LinkedHashMap<>();

        body.put("code", status.value());
        body.put("message", getMessage(status));
        body.put("timestamp", System.currentTimeMillis());
        body.put("path", request.getAttribute(
            "javax.servlet.error.request_uri"));

        // 从 RequestContext 获取详细异常信息
        RequestContext ctx = RequestContext.getCurrentContext();
        if (ctx != null && ctx.getThrowable() != null) {
            Throwable throwable = ctx.getThrowable();
            if (throwable instanceof HystrixRuntimeException) {
                body.put("detail", "服务熔断中，请稍后重试");
            }
        }

        return new ResponseEntity<>(body, status);
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer statusCode = (Integer) request.getAttribute(
            "javax.servlet.error.status_code");
        if (statusCode != null) {
            return HttpStatus.valueOf(statusCode);
        }
        return HttpStatus.INTERNAL_SERVER_ERROR;
    }

    private String getMessage(HttpStatus status) {
        switch (status) {
            case NOT_FOUND: return "请求的资源不存在";
            case SERVICE_UNAVAILABLE: return "服务暂不可用";
            case GATEWAY_TIMEOUT: return "服务响应超时";
            default: return "服务内部错误";
        }
    }
}
```

# 6. 自定义 ErrorFilter

自定义 Error 过滤器可以在 `SendErrorFilter` 之前执行，完成日志记录、告警、实体化异常等操作：

## 6.1 异常告警过滤器

```java
@Component
public class AlarmErrorFilter extends ZuulFilter {

    private static final Logger log = LoggerFactory.getLogger(AlarmErrorFilter.class);

    @Override
    public String filterType() { return "error"; }

    @Override
    public int filterOrder() {
        return -2;  // 在 SendErrorFilter (0) 之前执行
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getThrowable() != null;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        Throwable throwable = ctx.getThrowable();
        HttpServletRequest request = ctx.getRequest();

        // 记录详细错误日志
        log.error("Zuul 异常 - URI: {}, Method: {}, Error: {}",
            request.getRequestURI(),
            request.getMethod(),
            throwable.getMessage(),
            throwable);

        // 连接超时告警
        if (throwable.getCause() instanceof SocketTimeoutException) {
            alarmService.send(getServiceId(ctx) + " 服务响应超时");
        }

        // 连接拒绝告警
        if (throwable.getCause() instanceof ConnectException) {
            alarmService.send(getServiceId(ctx) + " 服务不可达");
        }

        return null;
    }

    private String getServiceId(RequestContext ctx) {
        String serviceId = (String) ctx.get("serviceId");
        return serviceId != null ? serviceId : "unknown";
    }
}
```

## 6.2 设置 sendErrorFilter.ran 避免默认错误处理

如果自定义 ErrorFilter 已经输出了错误响应，应阻止 `SendErrorFilter` 再次处理：

```java
@Override
public Object run() throws ZuulException {
    // ... 自定义错误响应处理逻辑 ...
    HttpServletResponse response = ctx.getResponse();
    response.setStatus(500);
    response.setContentType("application/json;charset=UTF-8");
    response.getWriter().write("{\"code\":500,\"message\":\"自定义错误\"}");

    // 阻止 SendErrorFilter 默认处理
    ctx.set("sendErrorFilter.ran", true);

    return null;
}
```

# 7. 预路由异常处理

Pre 阶段异常的特殊处理路径：

```
ZuulServlet.service()
    │
    ├── preRoute() → 抛出异常
    │     │
    │     ├── error(e)     ← 执行 Error 过滤器
    │     ├── postRoute()  ← 执行 Post 过滤器（清理工作）
    │     └── return        ← 不执行 route()
    │
    └── 如果 Authentication 失败等：
          ctx.setSendZuulResponse(false);
          ctx.setResponseBody("认证失败");
          // 不会进入异常处理链，Post 过滤器正常执行
```

**注意**：通过 `ctx.setSendZuulResponse(false)` 终止的请求**不会**进入 error 阶段，Post 过滤器仍然会正常执行。需要通过异常终止的请求才会进入 error 阶段。

# 8. 常见生产场景处理

## 8.1 连接超时

```java
// 场景：后端服务响应慢，Hystrix 超时触发
// 表现：HystrixTimeoutException

@Bean
public FallbackProvider timeoutFallback() {
    return new FallbackProvider() {
        @Override
        public String getRoute() { return "*"; }

        @Override
        public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
            if (cause instanceof HystrixTimeoutException) {
                return jsonResponse(504, "请求超时");
            }
            return jsonResponse(500, "服务异常");
        }
    };
}
```

## 8.2 服务不可达

```java
// 场景：后端服务未启动或网络不通
// 表现：ConnectException: Connection refused

// 通过 Ribbon 重试机制 + Hystrix Fallback 兜底
```

配置建议：

```yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000  # 超时不需太长

ribbon:
  ConnectTimeout: 2000   # 连接超时快速失败
  MaxAutoRetriesNextServer: 2  # 快速切换实例
```

## 8.3 熔断器打开

```java
// 场景：后端服务大面积故障，熔断器跳闸
// 表现：HystrixRuntimeException (CIRCUIT_OPEN)

// 应对策略：
// 1. FallbackProvider 提供降级响应
// 2. 监控告警，人工介入
// 3. 增大熔断休眠窗口，给后端恢复时间
```

# 9. 异常处理最佳实践

1. **统一错误响应格式**：通过自定义 `ErrorAttributes` 或 `ErrorController` 统一所有错误响应的 JSON 结构
2. **敏感信息过滤**：生产环境不暴露堆栈信息（`server.error.include-stacktrace=never`）
3. **层次化错误处理**：业务异常 → ErrorFilter → Spring Boot ErrorController → 客户端
4. **告警与监控**：ErrorFilter 中集成告警，确保关键异常可及时发现
5. **区分客户端错误和服务端错误**：4xx（输入校验失败、鉴权失败）vs 5xx（服务端错误、超时）
