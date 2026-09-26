
# 1. 概述

`RequestInterceptor` 是 Feign 提供的**请求前置拦截**接口，在 `Encoder` 执行前、HTTP 请求发送前被调用，用于对 `RequestTemplate` 进行**统一修改**。

```java
public interface RequestInterceptor {
    void apply(RequestTemplate template);
}
```

# 2. 典型使用场景

| 场景 | 示例 |
|------|------|
| **认证 Token 传递** | 从当前上下文获取 Token，写入 `Authorization` 头 |
| **分布式链路追踪** | 写入 TraceId / SpanId（如 `X-B3-TraceId`） |
| **版本标识** | 写入 API 版本号（`X-Api-Version`） |
| **灰度标记** | 写入灰度染色标记（`X-Gray`） |
| **通用请求头** | 写入 User-Agent、来源服务名等 |
| **请求签名** | 对请求参数签名，写入签名头 |

# 3. 基础示例：Token 传递

```java
@Component
public class AuthRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String token = getCurrentToken();
        if (StringUtils.hasText(token)) {
            template.header("Authorization", "Bearer " + token);
        }
    }

    private String getCurrentToken() {
        // 从 SecurityContext / ThreadLocal / 请求属性中获取
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken) {
            return ((JwtAuthenticationToken) auth).getToken().getTokenValue();
        }
        return null;
    }
}
```

# 4. 分布式链路追踪示例

```java
@Component
public class TraceRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // Sleuth / Micrometer Tracing 自动注入
        // 这里演示手动添加 TraceId 到 Header
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            template.header("X-B3-TraceId", currentSpan.context().traceId());
            template.header("X-B3-SpanId", currentSpan.context().spanId());
        }
    }
}
```

# 5. 上下文传递（ThreadLocal）

```java
public class UserContext {
    private static final ThreadLocal<Map<String, String>> CONTEXT = new ThreadLocal<>();

    public static void set(String key, String value) {
        CONTEXT.get().put(key, value);
    }

    public static String get(String key) {
        Map<String, String> map = CONTEXT.get();
        return map != null ? map.get(key) : null;
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

```java
@Component
public class ContextRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String userId = UserContext.get("userId");
        if (userId != null) {
            template.header("X-User-Id", userId);
        }
        String tenantId = UserContext.get("tenantId");
        if (tenantId != null) {
            template.header("X-Tenant-Id", tenantId);
        }
    }
}
```

在拦截器/过滤器中设置上下文：

```java
@Component
public class UserContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        try {
            UserContext.set("userId", request.getHeader("X-User-Id"));
            UserContext.set("tenantId", request.getHeader("X-Tenant-Id"));
            chain.doFilter(req, res);
        } finally {
            UserContext.clear();
        }
    }
}
```

# 6. 配置注册 Interceptor

## 6.1 方式 1：@Component 全局生效

```java
@Component
public class AuthRequestInterceptor implements RequestInterceptor {
    // 被 @ComponentScan 扫描到的所有 FeignClient 都会使用此拦截器
}
```

## 6.2 方式 2：针对特定 FeignClient

```java
@FeignClient(name = "user-service",
             configuration = UserFeignConfig.class)
public interface UserClient { }
```

```java
public class UserFeignConfig {
    @Bean
    public RequestInterceptor userAuthInterceptor() {
        return template -> template.header("X-Service-Key", "user-key-123");
    }
}
```

## 6.3 方式 3：YAML 全局配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            request-interceptors:
              - com.example.feign.AuthRequestInterceptor
              - com.example.feign.TraceRequestInterceptor
```

## 6.4 方式 4：Feign.Builder 手动注入

```java
@Bean
public UserClient userClient() {
    return Feign.builder()
            .requestInterceptor(new AuthRequestInterceptor())
            .target(UserClient.class, "http://localhost:8080");
}
```

# 7. 多个拦截器的执行顺序

```
RequestTemplate 创建
    │
    ├── RequestInterceptor A  (先注册的先执行)
    ├── RequestInterceptor B
    ├── RequestInterceptor C
    │
    ▼
Encoder.encode()  (请求体在此之后编码)
    │
    ▼
Client.execute()  → HTTP
```

**控制顺序**：
- `@Component` + `@Order` 控制全局拦截器顺序
- 特定 `@FeignClient(configuration=...)` 中注册的拦截器在全局之后执行

```java
@Component
@Order(1)
public class TraceInterceptor implements RequestInterceptor { }

@Component
@Order(2)
public class AuthInterceptor implements RequestInterceptor { }
```

# 8. 源码流程

```
SynchronousMethodHandler.invoke(args)
  │
  ├── 构建 RequestTemplate
  │     template = buildTemplateFromArgs.create(argv)
  │
  ├── 遍历所有 RequestInterceptor
  │     for (RequestInterceptor interceptor : requestInterceptors) {
  │         interceptor.apply(requestTemplate);
  │     }
  │
  ├── encoder.encode(body, bodyType, requestTemplate)
  │
  └── client.execute(request, options)
```

关键源码位置（`SynchronousMethodHandler`）：

```java
RequestTemplate template = buildTemplateFromArgs.create(argv);
Retryer retryer = this.retryer.clone();
while (true) {
    try {
        return executeAndDecode(template, options);
    } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
    }
}
```

其中 `executeAndDecode` 会在内部调用 `targetRequest(template)`，此时遍历 Interceptor：

```java
Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
        interceptor.apply(template);
    }
    return target.apply(template);
}
```

# 9. 注意事项

1. **不要阻塞 I/O** — Interceptor.apply() 在请求主路径上执行，避免网络调用
2. **注意线程安全** — Interceptor 是单例，不要持有线程不安全的可变状态
3. **FeignClient 级别隔离** — 如果某个 Interceptor 只用于特定 Client，通过 `configuration` 注入，不要用 `@Component`
4. **RequestTemplate 修改 Header 而非直接修改 Request** — 此时 HTTP 请求尚未构建，改 `template.header()` 即可
5. **配合 `@Headers` 使用** — 静态 Header 用 `@Headers`，动态 Header 用 `RequestInterceptor`
