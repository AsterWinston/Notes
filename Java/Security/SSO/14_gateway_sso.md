
# 1. 概述

在微服务架构中，网关是流量的统一入口，也是实现 SSO 认证的最佳位置。通过在网关层统一处理认证，后端微服务无需各自集成认证逻辑，实现**认证与业务分离**。

```
┌──────────────────────────────────────────────────────────────────┐
│                   网关统一认证架构                                │
│                                                                   │
│  客户端                 网关                     后端服务         │
│  ┌──────┐         ┌──────────────┐          ┌──────────┐        │
│  │Browser│  HTTPS │ Spring Cloud │  HTTP    │ Service A│        │
│  │      │────────►│   Gateway    │─────────►│          │        │
│  │  JS  │         │              │          └──────────┘        │
│  └──────┘         │ ┌──────────┐ │          ┌──────────┐        │
│                   │ │GlobalFilter│          │ Service B│        │
│  ┌──────┐         │ │ 认证/授权 │ │─────────►│          │        │
│  │Mobile│────────►│ └──────────┘ │          └──────────┘        │
│  │  App │         │              │                               │
│  └──────┘         │ ┌──────────┐ │                               │
│                   │ │RateLimiter│ │                               │
│                   │ └──────────┘ │                               │
│                   └──────────────┘                               │
│                                                                   │
│           认证中心 (Auth Server)                                  │
│           ┌──────────────────┐                                   │
│           │ OAuth2 / OIDC    │                                   │
│           │ CAS Server       │                                   │
│           └──────────────────┘                                   │
└──────────────────────────────────────────────────────────────────┘
```

# 2. Spring Cloud Gateway 认证方案

## 2.1 整体流程

```
① 用户访问 /api/user
② Gateway GlobalFilter 检查：是否携带有效 Token？
   ├─ 是 → 验证通过 → 转发到 Service A
   └─ 否 → 返回 401 / 重定向到登录页

③ 用户登录 → 获取 Token
④ 后续请求携带 Token → Gateway 验证 → 转发
```

## 2.2 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

## 2.3 基础配置

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=1

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

# 3. GlobalFilter 实现统一认证

## 3.1 JWT 认证 GlobalFilter

```java
/**
 * 全局 JWT 认证过滤器
 * 优先级设置为最高，确保最先执行
 */
@Component
@Order(-1)
public class JwtAuthenticationGlobalFilter implements GlobalFilter {

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";

    // 白名单路径（无需认证）
    private static final List<String> WHITE_LIST = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/public/",
        "/actuator/health"
    );

    @Autowired
    private JwtService jwtService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();

        // 1. 白名单放行
        if (isWhiteListed(path)) {
            return chain.filter(exchange);
        }

        // 2. 提取 Token
        String token = extractToken(request);
        if (token == null) {
            return unauthorized(exchange, "缺少认证令牌");
        }

        // 3. 验证 Token
        try {
            Claims claims = jwtService.parseToken(token);

            // 4. 将用户信息传递到下游服务（添加 Header）
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Name", claims.get("username", String.class))
                .header("X-User-Roles", claims.get("roles", String.class))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JwtException e) {
            return unauthorized(exchange, "令牌无效或已过期");
        }
    }

    private String extractToken(ServerHttpRequest request) {
        String authHeader = request.getHeaders().getFirst(AUTHORIZATION_HEADER);
        if (authHeader != null && authHeader.startsWith(BEARER_PREFIX)) {
            return authHeader.substring(BEARER_PREFIX.length());
        }

        // 也从 Cookie 中尝试提取（SSO 场景）
        List<HttpCookie> cookies = request.getCookies().get("SSO_TOKEN");
        if (cookies != null && !cookies.isEmpty()) {
            return cookies.get(0).getValue();
        }

        return null;
    }

    private boolean isWhiteListed(String path) {
        return WHITE_LIST.stream().anyMatch(path::startsWith);
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        byte[] body = String.format(
            "{\"code\":401,\"message\":\"%s\"}", message).getBytes();

        DataBuffer buffer = response.bufferFactory().wrap(body);
        return response.writeWith(Mono.just(buffer));
    }
}
```

## 3.2 OAuth2 登录网关模式

```java
/**
 * 网关作为 OAuth2 Client，负责完整的 OAuth2 流程
 * 后端服务完全无感知认证
 */
@Configuration
@EnableWebFluxSecurity
public class GatewayOAuth2Config {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(
            ServerHttpSecurity http) {

        http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/login", "/oauth2/**", "/public/**").permitAll()
                .anyExchange().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .authenticationSuccessHandler((webFilterExchange, auth) -> {
                    // OAuth2 登录成功后，将 Token 传递到前端
                    // 并通过网关转发
                    ServerHttpResponse response = webFilterExchange.getExchange()
                        .getResponse();
                    response.setStatusCode(HttpStatus.FOUND);
                    response.getHeaders().setLocation(URI.create("/home"));
                    return Mono.empty();
                })
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            )
            .csrf(ServerHttpSecurity.CsrfSpec::disable);

        return http.build();
    }
}
```

# 4. 网关转发用户信息

## 4.1 下游服务接收用户信息

```java
/**
 * 下游服务中通过 Header 接收网关传递的用户信息
 */
@Configuration
public class DownstreamServiceConfig {

    @Bean
    public FilterRegistrationBean<UserContextFilter> userContextFilter() {
        FilterRegistrationBean<UserContextFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new UserContextFilter());
        registration.addUrlPatterns("/*");
        registration.setOrder(1);
        return registration;
    }
}

public class UserContextFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {

        // 从网关转发的 Header 中提取用户信息
        String userId = request.getHeader("X-User-Id");
        String userName = request.getHeader("X-User-Name");
        String roles = request.getHeader("X-User-Roles");

        if (userId != null) {
            // 构建用户上下文
            UserContext context = new UserContext();
            context.setUserId(Long.parseLong(userId));
            context.setUsername(userName);
            context.setRoles(Arrays.asList(roles.split(",")));

            // 存入 ThreadLocal
            UserContextHolder.set(context);
        }

        try {
            chain.doFilter(request, response);
        } finally {
            UserContextHolder.clear();
        }
    }
}

/**
 * 下游服务中的用户上下文（ThreadLocal）
 */
public class UserContextHolder {
    private static final ThreadLocal<UserContext> CONTEXT = new ThreadLocal<>();

    public static void set(UserContext context) { CONTEXT.set(context); }
    public static UserContext get() { return CONTEXT.get(); }
    public static void clear() { CONTEXT.remove(); }
}

@Data
public class UserContext {
    private Long userId;
    private String username;
    private List<String> roles;

    public boolean hasRole(String role) {
        return roles != null && roles.contains(role);
    }
}
```

## 4.2 自定义网关过滤器：权限校验

```java
/**
 * 网关层面的权限校验
 * 在网关统一校验权限，拒绝无权限的请求，不再转发到下游
 */
@Component
public class AuthorizationGatewayFilter implements GlobalFilter, Ordered {

    // 路径-角色映射（简化示例，生产环境建议从配置中心加载）
    private static final Map<String, String> PATH_ROLE_MAP = Map.of(
        "/api/admin/", "ROLE_ADMIN",
        "/api/order/delete", "ROLE_ADMIN",
        "/api/user/", "ROLE_USER"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // 查找该路径需要的角色
        String requiredRole = PATH_ROLE_MAP.entrySet().stream()
            .filter(entry -> path.startsWith(entry.getKey()))
            .map(Map.Entry::getValue)
            .findFirst()
            .orElse(null);

        if (requiredRole == null) {
            // 无特殊权限要求，放行
            return chain.filter(exchange);
        }

        // 从请求头中获取用户角色
        String userRoles = exchange.getRequest().getHeaders()
            .getFirst("X-User-Roles");

        if (userRoles == null || !userRoles.contains(requiredRole)) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;  // 在 JWT 认证之后执行
    }
}
```

# 5. 网关 SSO 完整方案

## 5.1 网关 CAS 客户端模式

```java
/**
 * 网关作为 CAS Client 的代理
 * 适用于企业内部所有后端服务统一使用 CAS 的场景
 */
@Component
public class CasGatewayFilter implements GlobalFilter, Ordered {

    @Value("${cas.server-url-prefix}")
    private String casServerUrlPrefix;

    @Value("${cas.gateway-host}")
    private String gatewayHost;

    @Autowired
    private RestTemplate restTemplate;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();

        // 1. 从 Cookie 中提取 SSO 令牌（TGC）
        List<HttpCookie> cookies = request.getCookies().get("TGC");
        String tgc = (cookies != null && !cookies.isEmpty())
            ? cookies.get(0).getValue() : null;

        // 2. 从 URL 中检测 ST 参数
        String ticket = request.getQueryParams().getFirst("ticket");

        if (ticket != null) {
            // CAS 回调：验证 ST 并创建 Session
            return handleCasCallback(exchange, chain, ticket);
        }

        if (tgc != null) {
            // 用户已登录：传播用户信息到下游
            return propagateUserInfo(exchange, chain, tgc);
        }

        // 未登录：重定向到 CAS 登录页
        return redirectToCas(exchange, request);
    }

    private Mono<Void> handleCasCallback(ServerWebExchange exchange,
            GatewayFilterChain chain, String ticket) {
        // 调用 CAS Server 验证 ST
        String validateUrl = casServerUrlPrefix + "/p3/serviceValidate"
            + "?service=" + gatewayHost + exchange.getRequest().getURI().getPath()
            + "&ticket=" + ticket;

        String response = restTemplate.getForObject(validateUrl, String.class);

        // 解析用户信息 → 存入 Session/Redis → 创建局部会话
        CasUserInfo userInfo = parseCasResponse(response);

        // 将用户信息添加到请求头转发
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", userInfo.getUserId())
            .header("X-User-Name", userInfo.getUsername())
            .build();

        // 302 去掉 ticket 参数
        exchange.getResponse().setStatusCode(HttpStatus.FOUND);
        exchange.getResponse().getHeaders().setLocation(
            URI.create(exchange.getRequest().getURI().getPath()));

        return exchange.getResponse().setComplete();
    }

    private Mono<Void> redirectToCas(ServerWebExchange exchange,
            ServerHttpRequest request) {
        String serviceUrl = gatewayHost + request.getURI().getPath();
        String casLoginUrl = casServerUrlPrefix + "/login?service="
            + URLEncoder.encode(serviceUrl, StandardCharsets.UTF_8);

        exchange.getResponse().setStatusCode(HttpStatus.FOUND);
        exchange.getResponse().getHeaders().setLocation(URI.create(casLoginUrl));
        return exchange.getResponse().setComplete();
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```

## 5.2 网关 Token Relay（令牌中继）

```java
/**
 * TokenRelay：网关将当前用户的 Token 转发到下游服务
 * 下游服务可用该 Token 继续调用其他服务
 */
@Configuration
public class TokenRelayConfig {

    @Bean
    public TokenRelayGatewayFilterFactory tokenRelayFilter() {
        // Spring Cloud Gateway 内置的令牌中继过滤器
        // 自动将网关获得的 Token 转发到下游
        return new TokenRelayGatewayFilterFactory();
    }
}

// 路由配置中使用 TokenRelay
// spring:
//   cloud:
//     gateway:
//       routes:
//         - id: user-service
//           uri: lb://user-service
//           predicates:
//             - Path=/api/user/**
//           filters:
//             - TokenRelay=   ← 自动转发 Access Token
//             - StripPrefix=1
```

# 6. 网关安全增强

## 6.1 IP 白名单

```java
/**
 * IP 白名单过滤器：配合 SSO 实现深度防护
 */
@Component
public class IpWhitelistGatewayFilter implements GlobalFilter, Ordered {

    // 内网 IP 段（直接放行，不需要 SSO）
    private static final List<String> INTERNAL_IPS = List.of(
        "10.", "172.16.", "172.17.", "172.18.", "172.19.",
        "172.20.", "172.21.", "172.22.", "172.23.", "172.24.",
        "172.25.", "172.26.", "172.27.", "172.28.", "172.29.",
        "172.30.", "172.31.", "192.168."
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String remoteIp = exchange.getRequest().getRemoteAddress()
            .getAddress().getHostAddress();

        // 内网 IP 直接放行
        if (INTERNAL_IPS.stream().anyMatch(remoteIp::startsWith)) {
            return chain.filter(exchange);
        }

        // 外网 IP 必须认证
        // ... (继续 JWT/Cookie 验证)
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

## 6.2 限流保护

```java
/**
 * 网关认证接口限流
 * 防止暴力破解和 DDoS
 */
@Configuration
public class RateLimitConfig {

    @Bean
    public KeyResolver authKeyResolver() {
        // 对 /login 接口按 IP 限流
        return exchange -> {
            String path = exchange.getRequest().getURI().getPath();
            if (path.contains("/login")) {
                String ip = exchange.getRequest().getRemoteAddress()
                    .getAddress().getHostAddress();
                return Mono.just(ip);
            }
            return Mono.just("global");
        };
    }
}

// spring:
//   cloud:
//     gateway:
//       routes:
//         - id: auth-service
//           uri: lb://auth-service
//           predicates:
//             - Path=/api/auth/**
//           filters:
//             - name: RequestRateLimiter
//               args:
//                 redis-rate-limiter:
//                   replenishRate: 5        # 每秒 5 个令牌
//                   burstCapacity: 10        # 最大突发 10 个
```

# 7. 总结

| 模式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **网关 GlobalFilter** | 微服务统一认证 | 后端服务零侵入 | 网关是认证单点 |
| **网关 OAuth2 Client** | 第三方登录 SSO | 完整 OAuth2 流程在网关 | 配置较复杂 |
| **网关 CAS Proxy** | 企业内部系统 | 后端服务无感知 | 网关承担 ST 验证 |
| **Token Relay** | 服务间调用链 | Token 自动传播 | 需各服务支持 JWT |
| **IP 白名单** | 内网服务互通 | 简单高效 | 仅适合内网 |
