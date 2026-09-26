
# 1. Header 处理概览

Zuul 的 Header 处理分为三个阶段：

```
请求到达 → Pre 过滤器（修改请求头）
               │
               ▼
         Route 过滤器（传递请求头到后端，接收响应头）
               │
               ▼
         Post 过滤器（修改响应头，输出到客户端）
```

# 2. 敏感 Header（sensitiveHeaders）

## 2.1 默认敏感 Header

Spring Cloud Zuul 默认将以下 Header 标记为敏感，**不会**传递给下游服务：

```java
// ZuulProperties
private Set<String> sensitiveHeaders = new LinkedHashSet<>(
    Arrays.asList("Cookie", "Set-Cookie", "Authorization")
);
```

这意味着：
- 客户端的 `Cookie` 不会传递给后端服务
- 客户端的 `Authorization` 不会传递给后端服务
- 后端服务响应中的 `Set-Cookie` 不会传递给客户端

## 2.2 配置敏感 Header

```yaml
# 全局配置
zuul:
  sensitive-headers:
    - Cookie
    - Set-Cookie
    - Authorization
    - X-Internal-Token

# 按路由配置（覆盖全局）
zuul:
  routes:
    user-service:
      path: /user/**
      serviceId: user-service
      sensitive-headers:  # 空列表 = 传递所有 Header
        # 不传递任何敏感 Header 配置，即传递所有

    # 等同于
    user-service:
      path: /user/**
      serviceId: user-service
      sensitiveHeaders: []  # 传递所有 Header
```

## 2.3 Cookie 传递问题

Cookie 是最常需要传递的身份信息。如果后端需要验证 Session，必须放行 Cookie：

```yaml
zuul:
  routes:
    web-service:
      path: /web/**
      serviceId: web-service
      sensitive-headers:  # 清空敏感 Header 列表，允许传递 Cookie
```

或者保留 Cookie 的其他默认敏感 Header：

```yaml
zuul:
  sensitive-headers:
    - Authorization  # 仅过滤 Authorization
    # 移除 Cookie 和 Set-Cookie 的过滤

  routes:
    web-service:
      path: /web/**
      serviceId: web-service
```

## 2.4 敏感 Header 处理源码

在 `SimpleHostRoutingFilter` 和 `RibbonRoutingFilter` 中：

```java
// 过滤敏感 Header
private void addRequestHeaders(HttpServletRequest request,
                                HttpRequest httpRequest) {
    Enumeration<String> headerNames = request.getHeaderNames();
    while (headerNames.hasMoreElements()) {
        String headerName = headerNames.nextElement();
        // 跳过敏感 Header
        if (isSensitiveHeader(headerName)) {
            continue;
        }
        // 跳过 zuul 内部 Header
        if (headerName.toLowerCase().startsWith("x-zuul-")) {
            continue;
        }
        httpRequest.getHeaders().add(headerName,
            request.getHeader(headerName));
    }
}

private boolean isSensitiveHeader(String headerName) {
    Set<String> sensitiveHeaders = getSensitiveHeaders();
    return sensitiveHeaders.contains(headerName);
}
```

# 3. 忽略 Header（ignoredHeaders）

除了敏感 Header，Zuul 还支持完全忽略某些 Header 的传递：

```yaml
zuul:
  # 这些 Header 无论来自客户端还是后端，都不会被传递
  ignored-headers:
    - X-Frame-Options
    - X-Content-Type-Options
    - X-XSS-Protection
```

与 `sensitiveHeaders` 的区别：
- `sensitiveHeaders`：**选择性不传递**（可以按路由覆盖）
- `ignoredHeaders`：**全局强制不传递**（不可覆盖）

## 3.1 ignoredHeaders 处理源码

```java
// PreDecorationFilter
private boolean isIgnoredHeader(String headerName) {
    headerName = headerName.toLowerCase();
    for (String ignoredHeader : zuulProperties.getIgnoredHeaders()) {
        if (headerName.equals(ignoredHeader.toLowerCase())) {
            return true;
        }
    }
    return false;
}
```

# 4. 添加和修改 Header

## 4.1 添加请求头（zuulRequestHeaders）

在 Pre 过滤器中通过 `RequestContext` 添加自定义请求头：

```java
@Component
public class AddTraceIdFilter extends ZuulFilter {
    @Override
    public String filterType() { return "pre"; }
    @Override
    public int filterOrder() { return 10; }
    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 方式一：添加自定义 Header
        ctx.addZuulRequestHeader("X-Trace-Id", UUID.randomUUID().toString());
        ctx.addZuulRequestHeader("X-Request-Id", ctx.getRequest().getHeader("X-Request-Id"));
        ctx.addZuulRequestHeader("X-Forwarded-For", ctx.getRequest().getRemoteAddr());

        // 方式二：设置（覆盖同名 Header）
        ctx.set("zuulRequestHeaders", headersMap);

        return null;
    }
}
```

Route 过滤器中合并这些自定义 Header：

```java
// RibbonRoutingFilter / SimpleHostRoutingFilter
private void addRequestHeaders(HttpServletRequest request, HttpRequest httpRequest) {
    // 1. 添加原始请求头（过滤敏感 Header）
    // 2. 添加 zuulRequestHeaders（Pre 过滤器添加的自定义头）
    RequestContext ctx = RequestContext.getCurrentContext();
    Map<String, String> zuulRequestHeaders = ctx.getZuulRequestHeaders();
    for (Map.Entry<String, String> entry : zuulRequestHeaders.entrySet()) {
        httpRequest.getHeaders().add(entry.getKey(), entry.getValue());
    }
}
```

## 4.2 添加响应头（zuulResponseHeaders）

在 Post 过滤器中添加自定义响应头：

```java
@Component
public class AddResponseHeadersFilter extends ZuulFilter {
    @Override
    public String filterType() { return "post"; }
    @Override
    public int filterOrder() { return 900; }
    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 添加自定义响应头
        ctx.addZuulResponseHeader("X-Application-Name", "zuul-gateway");
        ctx.addZuulResponseHeader("X-Request-Duration",
            String.valueOf(System.currentTimeMillis() - (Long) ctx.get("startTime")));
        ctx.addZuulResponseHeader("X-Content-Type-Options", "nosniff");

        // 安全响应头
        ctx.addZuulResponseHeader("X-Frame-Options", "DENY");
        ctx.addZuulResponseHeader("X-XSS-Protection", "1; mode=block");

        return null;
    }
}
```

# 5. Host Header 处理

## 5.1 addHostHeader

```yaml
zuul:
  add-host-header: true  # 将原始请求的 Host 头转发到后端
```

源码实现（`PreDecorationFilter`）：

```java
if (zuulProperties.isAddHostHeader()) {
    ctx.addZuulRequestHeader("Host", ctx.getRequest().getHeader("Host"));
}
```

默认值为 `false`，因为 Host 头应该指向后端服务的实际地址。开启后可能导致一些依赖 Host 头的服务端路由出错。

## 5.2 addProxyHeaders

```yaml
zuul:
  add-proxy-headers: true
```

开启后自动添加标准代理头：

```
X-Forwarded-Host:  原始请求的 Host
X-Forwarded-Port:  原始请求的端口
X-Forwarded-Proto: 原始请求的协议（http/https）
X-Forwarded-Prefix: 路由前缀
X-Forwarded-For:   客户端 IP（链式追加）
```

## 5.3 setHost

```yaml
zuul:
  routes:
    user-service:
      path: /user/**
      serviceId: user-service
  host:
    socket-timeout-millis: 10000
    connect-timeout-millis: 2000
    max-total-connections: 200
    max-per-route-connections: 20
    time-to-live: 600000
    time-unit: MILLISECONDS
```

# 6. 请求头过滤实战

## 6.1 过滤内部调用 Header

```java
@Component
public class InternalHeaderFilter extends ZuulFilter {

    // 内部服务间使用的 Header，不可暴露给外部客户端
    private static final Set<String> INTERNAL_HEADERS = Set.of(
        "X-Internal-Token",
        "X-Service-Trace",
        "X-Debug-Info",
        "X-Correlation-Id"
    );

    @Override
    public String filterType() { return "post"; }

    @Override
    public int filterOrder() { return 950; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletResponse response = ctx.getResponse();

        // 从后端返回的响应中移除内部 Header
        List<Pair<String, String>> zuulResponseHeaders =
            ctx.getZuulResponseHeaders();
        if (zuulResponseHeaders != null) {
            zuulResponseHeaders.removeIf(header ->
                INTERNAL_HEADERS.contains(header.first().toLowerCase()));
        }

        return null;
    }
}
```

## 6.2 注入认证信息

```java
@Component
public class AuthHeaderFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 0; }

    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        String token = request.getHeader("Authorization");
        if (token != null && token.startsWith("Bearer ")) {
            // 解析 Token 并注入用户信息
            UserInfo user = tokenService.parseToken(token);
            ctx.addZuulRequestHeader("X-User-Id", user.getId());
            ctx.addZuulRequestHeader("X-User-Name", user.getUsername());
            ctx.addZuulRequestHeader("X-User-Role", String.join(",", user.getRoles()));

            // 将原始 Token 包装为新 Token 或移除敏感信息后传递
            ctx.addZuulRequestHeader("Authorization", "Bearer " + token);
        }

        return null;
    }
}
```

# 7. Header 大小写敏感性

HTTP Header 名称在规范中是大小写不敏感的（case-insensitive）。Zuul 中的 Header 处理需要特别注意：

```java
// 问题场景：敏感 Header 配置
sensitiveHeaders: [Authorization]

// 实际请求的 Header 可能为:
// authorization, AUTHORIZATION, Authorization, Authorization

// Zuul 的处理：默认不做大小写转换，可能导致配置不生效
```

解决方案：在自定义过滤器中做大小写标准化：

```java
private boolean isSensitive(String headerName) {
    return sensitiveHeaders.stream()
        .anyMatch(s -> s.equalsIgnoreCase(headerName));
}
```

# 8. Content-Type 处理

Zuul 默认会透传 `Content-Type` 到后端，但有些场景需要特殊处理：

```yaml
zuul:
  # 设置代理请求的默认 Content-Type
  set-content-length: true
```

Post 过滤器中处理响应 Content-Type：

```java
// SendResponseFilter
if (servletResponse.getContentType() == null
        && context.getResponseContentType() != null) {
    servletResponse.setContentType(context.getResponseContentType());
}
```

# 9. Header 操作的工具方法

```java
public class HeaderUtils {

    // 复制请求头（排除指定的 Header）
    public static void copyHeaders(HttpServletRequest source,
                                    HttpRequest target,
                                    Set<String> excludeHeaders) {
        Enumeration<String> headerNames = source.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String name = headerNames.nextElement();
            if (excludeHeaders.stream()
                    .noneMatch(e -> e.equalsIgnoreCase(name))) {
                target.getHeaders().add(name, source.getHeader(name));
            }
        }
    }

    // 安全地设置 CORS 响应头（不覆盖已有的）
    public static void setCorsHeader(HttpServletResponse response,
                                      String name, String value) {
        if (!response.containsHeader(name)) {
            response.setHeader(name, value);
        }
    }

    // 解析 X-Forwarded-For 获取原始客户端 IP
    public static String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```
