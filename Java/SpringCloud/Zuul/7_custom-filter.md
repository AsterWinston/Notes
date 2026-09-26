
# 1. 开发规范

自定义 Zuul 过滤器只需继承 `ZuulFilter` 抽象类并实现四个核心方法：

```java
public class MyCustomFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "pre";  // pre / route / post / error
    }

    @Override
    public int filterOrder() {
        return 0;  // 数字越小优先级越高
    }

    @Override
    public boolean shouldFilter() {
        return true;  // 运行时判断是否执行
    }

    @Override
    public Object run() throws ZuulException {
        // 过滤器核心业务逻辑
        return null;
    }
}
```

# 2. Pre 自定义过滤器示例

## 2.1 请求日志过滤器

```java
@Component
public class RequestLogFilter extends ZuulFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLogFilter.class);

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        log.info("=== Zuul Pre Filter ===");
        log.info("Request Method: {}", request.getMethod());
        log.info("Request URL: {}", request.getRequestURL());
        log.info("Request URI: {}", request.getRequestURI());
        log.info("Remote Address: {}", request.getRemoteAddr());
        log.info("Headers: {}", Collections.list(request.getHeaderNames())
            .stream().collect(Collectors.toMap(h -> h, request::getHeader)));

        // 注入 TraceId，用于全链路追踪
        String traceId = UUID.randomUUID().toString().replace("-", "");
        ctx.addZuulRequestHeader("X-Trace-Id", traceId);

        // 记录请求开始时间，供 Post 过滤器计算耗时
        ctx.set("startTime", System.currentTimeMillis());

        return null;
    }
}
```

## 2.2 认证过滤器（Token 校验）

```java
@Component
public class AuthFilter extends ZuulFilter {

    private static final Set<String> WHITE_LIST = new HashSet<>(Arrays.asList(
        "/api/auth/login",
        "/api/auth/register",
        "/actuator/health"
    ));

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;  // 最高优先级
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String requestUri = ctx.getRequest().getRequestURI();
        // 白名单路径不校验
        return WHITE_LIST.stream().noneMatch(requestUri::startsWith);
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        String token = request.getHeader("Authorization");
        if (token == null || token.isEmpty()) {
            // 终止请求处理，直接返回错误
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("{\"code\":401,\"message\":\"未授权\"}");
            ctx.getResponse().setContentType("application/json;charset=UTF-8");
            return null;
        }

        try {
            // 解析并校验 Token
            Claims claims = JwtUtils.parseToken(token.replace("Bearer ", ""));
            // 将用户信息传递给下游服务
            ctx.addZuulRequestHeader("X-User-Id", claims.get("userId", String.class));
            ctx.addZuulRequestHeader("X-User-Name", claims.get("username", String.class));
        } catch (Exception e) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("{\"code\":401,\"message\":\"Token无效或已过期\"}");
            ctx.getResponse().setContentType("application/json;charset=UTF-8");
        }
        return null;
    }
}
```

# 3. Route 自定义过滤器示例

## 3.1 自定义灰度路由过滤器

```java
@Component
public class GrayscaleRoutingFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "route";
    }

    @Override
    public int filterOrder() {
        // 必须在 RibbonRoutingFilter (10) 之前执行
        return 5;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        // 通过请求头判断是否为灰度用户
        String grayTag = request.getHeader("X-Gray-Tag");
        return "true".equals(grayTag);
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 方案一：修改 serviceId，追加灰度版本号
        String originalServiceId = (String) ctx.get("serviceId");
        ctx.set("serviceId", originalServiceId + "-gray");

        // 方案二：直接在 RequestContext 中添加灰度标记
        ctx.addZuulRequestHeader("X-Canary", "true");

        return null;
    }
}
```

# 4. Post 自定义过滤器示例

## 4.1 响应日志与耗时统计

```java
@Component
public class ResponseLogFilter extends ZuulFilter {

    private static final Logger log = LoggerFactory.getLogger(ResponseLogFilter.class);

    @Override
    public String filterType() {
        return "post";
    }

    @Override
    public int filterOrder() {
        // SendResponseFilter (1000) 之前执行，以便访问响应数据
        return 900;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        HttpServletResponse response = ctx.getResponse();

        Long startTime = (Long) ctx.get("startTime");
        long duration = (startTime != null)
            ? System.currentTimeMillis() - startTime
            : -1;

        log.info("=== Zuul Post Filter ===");
        log.info("Request URI: {}", request.getRequestURI());
        log.info("Response Status: {}", ctx.getResponseStatusCode());
        log.info("Duration: {} ms", duration);

        // 统一添加响应头
        ctx.addZuulResponseHeader("X-Application-Name", "zuul-gateway");
        ctx.addZuulResponseHeader("X-Response-Time", String.valueOf(duration));

        return null;
    }
}
```

## 4.2 跨域响应头设置过滤器

```java
@Component
public class CorsFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "post";
    }

    @Override
    public int filterOrder() {
        return 0;  // 尽早执行
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletResponse response = ctx.getResponse();

        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
        response.setHeader("Access-Control-Allow-Headers",
            "Content-Type,Authorization,X-Requested-With");
        response.setHeader("Access-Control-Max-Age", "3600");

        return null;
    }
}
```

# 5. Error 自定义过滤器示例

## 5.1 统一错误响应格式

```java
@Component
public class CustomErrorFilter extends ZuulFilter {

    private static final Logger log = LoggerFactory.getLogger(CustomErrorFilter.class);

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        // 在 SendErrorFilter (0) 之前执行
        return -1;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getThrowable() != null
            && !ctx.getBoolean("sendErrorFilter.ran", false);
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        Throwable throwable = ctx.getThrowable();

        log.error("Zuul 请求处理异常: {}", throwable.getMessage(), throwable);

        // 构造统一错误响应体
        Map<String, Object> errorBody = new LinkedHashMap<>();
        errorBody.put("code", extractStatusCode(throwable));
        errorBody.put("message", extractMessage(throwable));
        errorBody.put("timestamp", System.currentTimeMillis());

        HttpServletResponse response = ctx.getResponse();
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus((int) errorBody.get("code"));

        try {
            String json = new ObjectMapper().writeValueAsString(errorBody);
            response.getOutputStream().write(json.getBytes(StandardCharsets.UTF_8));
        } catch (Exception e) {
            log.error("Error writing error response", e);
        }

        // 标记已处理，防止 SendErrorFilter 重复处理
        ctx.set("sendErrorFilter.ran", true);

        return null;
    }

    private int extractStatusCode(Throwable throwable) {
        if (throwable instanceof ZuulException) {
            return ((ZuulException) throwable).nStatusCode;
        }
        if (throwable.getCause() instanceof SocketTimeoutException) {
            return 504;  // Gateway Timeout
        }
        return 500;
    }

    private String extractMessage(Throwable throwable) {
        if (throwable.getCause() instanceof ConnectException) {
            return "后端服务连接失败";
        }
        if (throwable.getCause() instanceof SocketTimeoutException) {
            return "后端服务响应超时";
        }
        return throwable.getMessage();
    }
}
```

# 6. 过滤器注册方式

## 6.1 Spring Bean 注册（推荐）

最常见的注册方式：使用 `@Component` 注解将自定义过滤器声明为 Spring Bean。Spring 的 `ZuulServerAutoConfiguration` 会自动扫描并注册所有 `ZuulFilter` 类型的 Bean。

```java
@Component  // ← 自动注册到 FilterRegistry
public class MyFilter extends ZuulFilter { ... }
```

注意：如果配置了多个同类过滤器实例，过滤器名（`filterName()`）必须不同，否则注册时会冲突。

## 6.2 手动注册到 FilterRegistry

```java
@PostConstruct
public void registerFilters() {
    FilterRegistry registry = FilterRegistry.instance();
    registry.put("myCustomFilter", new MyFilter());
}
```

## 6.3 Groovy 动态注册（动态过滤器）

Zuul 支持使用 Groovy 脚本动态加载过滤器，运行时修改后自动热更新：

```groovy
// MyDynamicFilter.groovy
class MyDynamicFilter extends ZuulFilter {
    @Override
    String filterType() { return "pre" }
    @Override
    int filterOrder() { return 15 }
    @Override
    boolean shouldFilter() { return true }
    @Override
    Object run() {
        RequestContext ctx = RequestContext.getCurrentContext()
        println "Dynamic Groovy Filter: ${ctx.request.requestURI}"
        return null
    }
}
```

配置启用 Groovy 动态过滤器：

```yaml
zuul:
  filter:
    root: classpath:groovy/filters    # Groovy 文件目录
    interval: 30                       # 检测变更间隔（秒）
```

FilterLoader 会定期检查目录中 Groovy 文件的修改时间，如有变更则重新编译并替换旧过滤器。

# 7. 过滤器开发的注意事项

## 7.1 filterOrder 冲突

内置过滤器的 order 值固定，自定义过滤器如果 order 值与内置过滤器冲突，行为不确定：
- 同 order 值的过滤器执行顺序由 `FilterRegistry` 的 `HashMap` 迭代顺序决定，不可靠
- 建议自定义过滤器的 order 留有间隙，如 `10, 20, 30...`

## 7.2 shouldFilter 性能

`shouldFilter()` 在每次请求时都会被调用，应避免在其中执行重量级操作（如数据库查询、远程调用等）：

```java
// 不推荐
@Override
public boolean shouldFilter() {
    return redisTemplate.hasKey("rate-limit:" + getCurrentUser());
}

// 推荐：将判断逻辑放到 run() 中，shouldFilter 仅做轻量判断
@Override
public boolean shouldFilter() {
    return RequestContext.getCurrentContext().sendZuulResponse();
}
```

## 7.3 使用 setSendZuulResponse(false) 终止处理链

当 Pre 过滤器需要直接返回响应时（如鉴权失败），必须调用：

```java
ctx.setSendZuulResponse(false);  // 关键：阻止后续过滤器执行
```

这会使得 `RibbonRoutingFilter` 和 `SimpleHostRoutingFilter` 的 `shouldFilter()` 返回 false，从而不会向后端发起请求。

## 7.4 异常处理

- 在 `run()` 中抛出的异常会终止当前类型的过滤器链，跳转到 error 过滤器
- 不要在 `run()` 中吞掉异常，应该让它自然传播，由 error 过滤器统一处理
- 如需自定义错误响应，编写自定义 Error 过滤器，order 设为 -1 确保在 `SendErrorFilter` 之前执行

## 7.5 线程安全

`RequestContext` 基于 `ThreadLocal`，天然线程安全。但自定义过滤器中的**成员变量**如果存储了请求级数据，需要在 `run()` 或 `shouldFilter()` 中保证线程安全。推荐的实践是自定义过滤器**不使用成员变量存储请求相关数据**，全部通过 `RequestContext` 传递。

```java
// 不推荐：成员变量存储请求级数据（线程不安全）
public class BadFilter extends ZuulFilter {
    private String currentUser;  // 多个请求会共享此变量
    // ...
}

// 推荐：所有数据通过 RequestContext 传递
public class GoodFilter extends ZuulFilter {
    // 无成员变量（除了注入的依赖）
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.set("currentUser", extractUser(ctx.getRequest()));
        return null;
    }
}
```
