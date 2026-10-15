# 1. OAuth2 / JWT 集成

## 1.1 OAuth2 Client & Resource Server

**OAuth2 授权码流程（Authorization Code Grant）：**

```
用户          前端         后端(Client)    授权服务器     资源服务器
 │             │              │               │              │
 │──点击登录──→│              │               │              │
 │             │──GET /auth──→│               │              │
 │             │              │──重定向到授权页→│              │
 │←─显示登录页─│──────────────│               │              │
 │──输入凭证───→│              │               │              │
 │             │              │               │验证身份      │
 │             │              │←─授权码+重定向─│              │
 │             │              │──用授权码换Token→│            │
 │             │              │←──访问Token+刷新Token─│      │
 │             │─登录成功─────│               │              │
 │             │              │──请求资源──────────────→│     │
 │             │              │←──资源数据──────────────│     │
```

**引入依赖：**

```xml
<!-- OAuth2 Client -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<!-- OAuth2 Resource Server（验证 JWT） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

## 1.2 JWT 验证机制

**JWT 结构：**

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.abc123...
└────── Header ──────┘ └── Payload ──┘ └ Signature ┘
```

| 部分 | 内容 | 说明 |
|------|------|------|
| Header | `{"alg":"HS256","typ":"JWT"}` | 算法信息 |
| Payload | `{"sub":"admin","exp":1234567890}` | 实际载荷 |
| Signature | HMAC-SHA256(base64Url(header)+"."+base64Url(payload), secret) | 签名 |

**JWT 验证流程：**

```
请求 → Authorization: Bearer eyJ...
    → 提取 Token → 验证签名 → 检查过期 → 解析载荷
    → 构建 Authentication → 存入 SecurityContextHolder
```

**自定义 JWT 验证注解：**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
}

// AOP 切面验证
@Aspect
@Component
public class PermissionAspect {

    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint,
            RequirePermission requirePermission) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        boolean hasPermission = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals(requirePermission.value()));

        if (!hasPermission) {
            throw new AccessDeniedException("缺少权限: " + requirePermission.value());
        }
        return joinPoint.proceed();
    }
}
```

## 1.3 Spring Security 与 JWT 集成示例

**完整 JWT 项目结构：**

```
src/main/java/com/example/
├── config/
│   ├── SecurityConfig.java          // 安全配置
│   └── CorsConfig.java              // CORS配置
├── security/
│   ├── JwtTokenProvider.java        // Token生成/验证
│   ├── JwtAuthenticationFilter.java // JWT过滤器
│   ├── CustomUserDetailsService.java// 用户加载
│   └── UserPrincipal.java          // 自定义 UserDetails
├── controller/
│   ├── AuthController.java          // 登录/注册
│   └── UserController.java          // 用户API
├── entity/
│   ├── User.java
│   └── Role.java
└── repository/
    └── UserRepository.java
```

**application.yml JWT 配置：**

```yaml
jwt:
  secret: your-256-bit-secret-key-minimum-32-characters-long
  expiration: 3600000  # 1小时

spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server.example.com
          jwk-set-uri: https://auth-server.example.com/.well-known/jwks.json
```

---

# 2. 自定义安全策略

## 2.1 自定义 AuthenticationProvider

```java
@Component
public class WeChatAuthenticationProvider implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication auth) 
            throws AuthenticationException {
        String wxCode = auth.getCredentials().toString();

        // 1. 调用微信API获取 openId
        WxSession wxSession = wxService.code2Session(wxCode);
        String openId = wxSession.getOpenId();

        // 2. 查询或创建用户
        UserEntity user = userService.findOrCreateByOpenId(openId);

        // 3. 构建认证对象
        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getCode()))
            .collect(Collectors.toList());

        return new WeChatAuthenticationToken(user, null, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return WeChatAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

## 2.2 自定义 UserDetailsService

> 详见 [5_authentication_implement.md §3.3.1](5_authentication_implement.md#331-userdetailsservice-实现类)。

## 2.3 自定义 Filter 与 SecurityFilterChain

```java
// 请求日志过滤器
public class RequestLoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {
        log.info("{} {} from {}", request.getMethod(),
                 request.getRequestURI(), request.getRemoteAddr());
        chain.doFilter(request, response);
    }
}

// 注册
http.addFilterBefore(new RequestLoggingFilter(),
    SecurityContextHolderFilter.class);
```

---

# 3. 其他权限校验方法

## 3.1 方法级注解权限控制

```java
// @PreAuthorize - 方法执行前校验（最常用）
@PreAuthorize("hasRole('ADMIN') or hasAuthority('report:manage')")
public Report generateReport() { ... }

// @PostAuthorize - 方法执行后校验（可检查返回值）
@PostAuthorize("returnObject.status != 'DRAFT'")
public Document getDocument(Long id) { ... }

// @PreFilter - 过滤输入集合
@PreFilter("filterObject.owner == authentication.name")
public void batchUpdate(List<Document> docs) { ... }

// @PostFilter - 过滤输出集合
@PostFilter("filterObject.isPublic() or hasRole('ADMIN')")
public List<Document> getAllDocuments() { ... }

// @Secured - 角色校验
@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
public void sensitiveOperation() { ... }
```

**自定义权限校验函数：**

```java
@Component("authz")
public class CustomAuthz {

    public boolean checkProjectOwner(Authentication auth, Long projectId) {
        return projectService.isOwner(auth.getName(), projectId);
    }
}

// 使用
@PreAuthorize("@authz.checkProjectOwner(authentication, #projectId)")
public Project getProject(Long projectId) { ... }
```

## 3.2 URL 级权限控制

```java
// 请求方法 + 路径 组合控制
http.authorizeHttpRequests(auth -> auth
    // GET 允许所有已认证用户
    .requestMatchers(HttpMethod.GET, "/api/users/**")
        .authenticated()
    // POST 需要 user:create
    .requestMatchers(HttpMethod.POST, "/api/users/**")
        .hasAuthority("user:create")
    // PUT 需要 user:update
    .requestMatchers(HttpMethod.PUT, "/api/users/**")
        .hasAuthority("user:update")
    // DELETE 需要 user:delete
    .requestMatchers(HttpMethod.DELETE, "/api/users/**")
        .hasAuthority("user:delete")
);
```

## 3.3 自定义 Filter/Interceptor 权限验证

```java
@Component
public class ApiRateLimitFilter extends OncePerRequestFilter {

    private final Cache<String, AtomicInteger> counter = 
        Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(10000)
            .build();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {
        String key = request.getRemoteAddr() + ":" + request.getRequestURI();
        AtomicInteger count = counter.get(key, k -> new AtomicInteger(0));

        if (count.incrementAndGet() > 100) {
            response.setStatus(429);
            response.getWriter().write("{\"code\":429,\"msg\":\"请求过多\"}");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

---

# 4. 基于配置的权限控制

## 4.1 HttpSecurity 配置 `.authorizeHttpRequests()`

> Spring Security 6.x 中 `authorizeRequests()` 已废弃，使用 `authorizeHttpRequests()`。

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/admin/**").hasRole("ADMIN")
        .requestMatchers("/api/**").authenticated()
        .requestMatchers("/public/**").permitAll()
        .anyRequest().denyAll()  // 默认拒绝
    );
    return http.build();
}
```

## 4.2 Role 与 Authority 区别

| 特性 | Role | Authority |
|------|------|-----------|
| 前缀 | 自动加 `ROLE_` | 原样使用 |
| API 校验 | `.hasRole("ADMIN")` | `.hasAuthority("ROLE_ADMIN")` |
| 注解校验 | `@PreAuthorize("hasRole('ADMIN')")` | `@PreAuthorize("hasAuthority('user:delete')")` |
| 粒度 | 粗粒度 | 细粒度 |
| 使用建议 | 角色分类 | 操作权限 |

```
Role: ROLE_ADMIN, ROLE_USER
Authority: user:read, user:write, order:delete, report:export
```

## 4.3 配置链顺序问题

**Spring Security 匹配规则是"先匹配生效"**，更具体的规则应放在前面：

```java
http.authorizeHttpRequests(auth -> auth
    // ✅ 正确：具体先于通用
    .requestMatchers("/api/users/admin/**").hasRole("ADMIN")   // 先具体
    .requestMatchers("/api/users/**").authenticated()          // 后通用

    // ❌ 错误：通用先于具体——/api/users/admin/** 永远走不到
    .requestMatchers("/api/users/**").authenticated()
    .requestMatchers("/api/users/admin/**").hasRole("ADMIN")  // 被上一行覆盖
);
```

---

# 5. CSRF 专题

## 5.1 CSRF 原理与防护

```
CSRF攻击的必要条件：
1. 用户已登录目标网站（有 Session Cookie）
2. 用户访问了恶意网站
3. 恶意网站向目标网站发送请求（浏览器自动带上 Cookie）

防护手段：
→ 服务端生成一个随机 Token
→ 每次状态变更请求（POST/PUT/DELETE）必须携带 Token
→ 恶意网站无法读取该 Token（同源策略）
```

## 5.2 Token 配置与前端传递

**服务端配置：**

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

**前端获取和传递：**

```javascript
// 从Cookie读取CSRF Token（服务端自动写入XSRF-TOKEN Cookie）
const csrfToken = document.cookie
    .split('; ')
    .find(row => row.startsWith('XSRF-TOKEN='))
    ?.split('=')[1];

// AJAX请求携带
fetch('/api/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': csrfToken  // 请求头携带
    },
    body: JSON.stringify({ name: 'test' })
});
```

## 5.3 REST API 项目 CSRF 处理方式

**无状态 JWT API → 直接禁用 CSRF：**

```java
http.csrf(AbstractHttpConfigurer::disable);
```

**原因：** JWT 认证的 API 不使用 Cookie 维持会话，没有 CSRF 攻击面（CSRF 依赖浏览器自动发送 Cookie）。

**例外情况：** 如果 API 仍使用 Cookie（如 `JSESSIONID`），则需保留 CSRF 保护。
