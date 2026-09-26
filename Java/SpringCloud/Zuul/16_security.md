
# 1. 安全体系概览

Zuul 作为 API 网关，是安全体系的第一道防线。典型安全职责：

```
客户端 → Zuul 网关安全层
              │
              ├── 身份认证（Authentication）
              ├── 权限校验（Authorization）
              ├── CORS 跨域处理
              ├── IP 黑白名单
              ├── 请求参数校验
              ├── SQL/XSS 注入防护
              └── 限流保护
              │
              ▼
         后端微服务（可信任调用）
```

# 2. CORS 跨域配置

## 2.1 通过 Zuul 配置 CORS

```yaml
zuul:
  routes:
    web-app:
      path: /web/**
      serviceId: web-service
      sensitive-headers:
```

Java 配置方式：

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("http://localhost:3000");
        config.addAllowedOrigin("https://app.example.com");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
}
```

## 2.2 CORS Preflight 处理

OPTIONS 预检请求需要正确响应，否则后续真实请求会被浏览器拦截：

```java
@Component
public class CorsPreflightFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 0; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return "OPTIONS".equalsIgnoreCase(ctx.getRequest().getMethod());
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletResponse response = ctx.getResponse();

        // 直接返回 CORS 头，不转发到后端
        response.setHeader("Access-Control-Allow-Origin",
            ctx.getRequest().getHeader("Origin"));
        response.setHeader("Access-Control-Allow-Methods",
            "GET,POST,PUT,DELETE,OPTIONS");
        response.setHeader("Access-Control-Allow-Headers",
            "Content-Type,Authorization,X-Requested-With");
        response.setHeader("Access-Control-Allow-Credentials", "true");
        response.setHeader("Access-Control-Max-Age", "3600");

        // 终止过滤器链，直接返回 200
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(200);

        return null;
    }
}
```

## 2.3 注意点

Zuul 与 Spring MVC 的 CORS 处理可能冲突。如果同时配置了 `CorsFilter`（Spring）和 Zuul 自定义 CORS Filter，响应头可能重复。建议：

- 仅在 Zuul 层统一处理 CORS（推荐）
- 或关闭 Zuul 的 CORS 处理，完全交由 Spring MVC

# 3. JWT 认证过滤器

## 3.1 认证流程设计

```
客户端                            Zuul 网关                         后端服务
   │                                │                                │
   ├─ POST /auth/login ────────────→│                                │
   │                                ├─ 用户名/密码校验               │
   │                                ├─ 生成 JWT Token               │
   │←─ 返回 JWT Token ─────────────┤                                │
   │                                │                                │
   │                                │                                │
   ├─ GET /user/profile ───────────→│                                │
   │  Header: Authorization         ├─ 解析 JWT Token               │
   │         Bearer xxx.yyy.zzz     ├─ 校验 Token 有效性            │
   │                                ├─ 注入 X-User-Id 等 Header     │
   │                                ├───────────────────────────────→│
   │                                │                                ├─ 获取用户信息
   │                                │←───────────────────────────────┤
   │←─ 返回用户信息 ────────────────┤                                │
```

## 3.2 JWT 认证过滤器实现

```java
@Component
public class JwtAuthFilter extends ZuulFilter {

    private static final String TOKEN_PREFIX = "Bearer ";
    private static final String HEADER_AUTH = "Authorization";

    // 无需认证的白名单路径
    private static final Set<String> AUTH_WHITELIST = Set.of(
        "/auth/login",
        "/auth/register",
        "/auth/refresh",
        "/actuator/health"
    );

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 0; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String uri = ctx.getRequest().getRequestURI();

        // OPTIONS 预检请求放行（浏览器 CORS）
        if ("OPTIONS".equalsIgnoreCase(ctx.getRequest().getMethod())) {
            return false;
        }

        // 白名单路径不校验
        return AUTH_WHITELIST.stream().noneMatch(uri::startsWith);
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        String authHeader = request.getHeader(HEADER_AUTH);
        if (authHeader == null || !authHeader.startsWith(TOKEN_PREFIX)) {
            rejectRequest(ctx, 401, "未提供认证令牌");
            return null;
        }

        String token = authHeader.substring(TOKEN_PREFIX.length());
        try {
            // 解析 JWT Token
            Claims claims = Jwts.parser()
                .setSigningKey(SECRET_KEY)
                .parseClaimsJws(token)
                .getBody();

            // 将用户信息注入请求头（通过 RequestContext）
            ctx.addZuulRequestHeader("X-User-Id",
                claims.get("userId", String.class));
            ctx.addZuulRequestHeader("X-User-Name",
                claims.getSubject());
            ctx.addZuulRequestHeader("X-User-Roles",
                claims.get("roles", String.class));

        } catch (ExpiredJwtException e) {
            rejectRequest(ctx, 401, "令牌已过期");
        } catch (SignatureException e) {
            rejectRequest(ctx, 401, "令牌签名无效");
        } catch (Exception e) {
            rejectRequest(ctx, 401, "令牌解析失败");
        }

        return null;
    }

    private void rejectRequest(RequestContext ctx, int status, String message) {
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(status);
        ctx.setResponseBody(
            String.format("{\"code\":%d,\"message\":\"%s\"}", status, message));
        ctx.getResponse().setContentType("application/json;charset=UTF-8");
    }
}
```

# 4. OAuth2 / Spring Security 集成

## 4.1 集成 Spring Security OAuth2

```java
@Configuration
@EnableOAuth2Sso  // 启用 OAuth2 SSO
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
            .antMatchers("/login", "/actuator/health").permitAll()
            .anyRequest().authenticated()
            .and()
            .csrf().disable();
    }
}
```

配置 `application.yml`：

```yaml
security:
  oauth2:
    client:
      client-id: zuul-gateway
      client-secret: secret
      access-token-uri: http://auth-server:8080/oauth/token
      user-authorization-uri: http://auth-server:8080/oauth/authorize
      scope: read,write
    resource:
      user-info-uri: http://auth-server:8080/user/me
```

然后在 Zuul 过滤器中传递 OAuth2 Token：

```java
@Component
public class OAuth2TokenRelayFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 1; }

    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 从当前 SecurityContext 获取 Token
        Authentication auth = SecurityContextHolder.getContext()
            .getAuthentication();
        if (auth != null && auth.getDetails() instanceof OAuth2AuthenticationDetails) {
            OAuth2AuthenticationDetails details =
                (OAuth2AuthenticationDetails) auth.getDetails();
            // 将 Token 传递到下游服务
            ctx.addZuulRequestHeader("Authorization",
                "Bearer " + details.getTokenValue());
        }

        return null;
    }
}
```

# 5. IP 黑白名单

```java
@Component
public class IPAccessControlFilter extends ZuulFilter {

    // IP 白名单（仅允许这些 IP 访问）
    private static final Set<String> WHITE_LIST = Set.of(
        "192.168.1.0/24",
        "10.0.0.0/8"
    );

    // IP 黑名单（禁止这些 IP 访问）
    private static final Set<String> BLACK_LIST = Set.of(
        "1.2.3.4",
        "5.6.7.8"
    );

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return -1; }  // 最高优先级

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        // 内部请求（如健康检查）放行
        return !ctx.getRequest().getRequestURI().startsWith("/actuator");
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        String clientIp = getClientIp(ctx.getRequest());

        // 黑名单检查
        if (BLACK_LIST.contains(clientIp)) {
            rejectRequest(ctx, 403, "IP 已被禁止访问");
            return null;
        }

        // 白名单检查
        boolean allowed = WHITE_LIST.stream()
            .anyMatch(prefix -> isInRange(clientIp, prefix));

        if (!allowed) {
            rejectRequest(ctx, 403, "IP 不在白名单中");
            return null;
        }

        return null;
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff != null && !xff.isEmpty()) {
            return xff.split(",")[0].trim();
        }
        String xri = request.getHeader("X-Real-IP");
        if (xri != null && !xri.isEmpty()) {
            return xri.trim();
        }
        return request.getRemoteAddr();
    }

    private boolean isInRange(String ip, String cidr) {
        // CIDR 匹配逻辑（如 192.168.1.0/24）
        String[] parts = cidr.split("/");
        int maskLength = Integer.parseInt(parts[1]);
        return ipToLong(ip) >> (32 - maskLength)
            == ipToLong(parts[0]) >> (32 - maskLength);
    }

    private long ipToLong(String ip) {
        String[] octets = ip.split("\\.");
        return (Long.parseLong(octets[0]) << 24)
            | (Long.parseLong(octets[1]) << 16)
            | (Long.parseLong(octets[2]) << 8)
            | Long.parseLong(octets[3]);
    }

    private void rejectRequest(RequestContext ctx, int status, String message) {
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(status);
        ctx.setResponseBody(
            String.format("{\"code\":%d,\"message\":\"%s\"}", status, message));
        ctx.getResponse().setContentType("application/json;charset=UTF-8");
    }
}
```

# 6. 请求参数校验

```java
@Component
public class RequestValidationFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 5; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String method = ctx.getRequest().getMethod();
        return "POST".equalsIgnoreCase(method)
            || "PUT".equalsIgnoreCase(method);
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // Content-Type 校验
        String contentType = request.getContentType();
        if (contentType != null && !contentType.contains("application/json")) {
            rejectRequest(ctx, 415, "不支持的 Content-Type");
            return null;
        }

        // 请求体大小校验
        if (request.getContentLength() > 10 * 1024 * 1024) { // 10MB
            rejectRequest(ctx, 413, "请求体大小超过限制");
            return null;
        }

        return null;
    }

    private void rejectRequest(RequestContext ctx, int status, String message) {
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(status);
        ctx.setResponseBody(
            String.format("{\"code\":%d,\"message\":\"%s\"}", status, message));
        ctx.getResponse().setContentType("application/json;charset=UTF-8");
    }
}
```

# 7. SQL 注入与 XSS 防护

```java
@Component
public class SecurityFilter extends ZuulFilter {

    // 常见的 SQL 注入和 XSS 关键字
    private static final Pattern SQL_INJECTION =
        Pattern.compile(
            "(\\bselect\\b|\\binsert\\b|\\bdelete\\b|\\bupdate\\b|\\bdrop\\b" +
            "|\\bunion\\b|\\bexec\\b|\\bscript\\b|\\balert\\b|<script>)",
            Pattern.CASE_INSENSITIVE);

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 3; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // 检查 URL 参数
        Map<String, String[]> params = request.getParameterMap();
        for (Map.Entry<String, String[]> entry : params.entrySet()) {
            for (String value : entry.getValue()) {
                if (SQL_INJECTION.matcher(value).find()) {
                    rejectRequest(ctx, 400, "检测到可能的攻击参数");
                    return null;
                }
            }
        }

        // 检查 URI 路径
        String uri = request.getRequestURI();
        if (SQL_INJECTION.matcher(uri).find()) {
            rejectRequest(ctx, 400, "检测到可能的攻击路径");
            return null;
        }

        return null;
    }

    private void rejectRequest(RequestContext ctx, int status, String message) {
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(status);
        ctx.setResponseBody(
            String.format("{\"code\":%d,\"message\":\"%s\"}", status, message));
        ctx.getResponse().setContentType("application/json;charset=UTF-8");
    }
}
```

# 8. 安全配置综合建议

```yaml
zuul:
  routes:
    user-service:
      path: /user/**
      serviceId: user-service

  # 安全相关全局配置
  sensitive-headers:
    - Cookie
    - Set-Cookie
    - Authorization

  add-host-header: false
  add-proxy-headers: true  # X-Forwarded-* 头帮助后端识别真实客户端

# Spring Boot 安全开关
server:
  error:
    include-stacktrace: never  # 生产环境不暴露堆栈

management:
  endpoints:
    web:
      exposure:
        include: health,info  # 仅暴露必要的端点
  endpoint:
    health:
      show-details: when-authorized

# HTTPS 配置（生产环境推荐）
server:
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    key-alias: zuul
```
