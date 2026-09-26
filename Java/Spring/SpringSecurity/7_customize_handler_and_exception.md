# 1. 自定义失败处理

## 1.1 AuthenticationFailureHandler

**登录失败时触发**，可自定义返回格式（JSON 而非重定向）：

```java
@Component
public class CustomAuthenticationFailureHandler 
        implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException exception)
            throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");

        String message;
        if (exception instanceof UsernameNotFoundException) {
            message = "用户不存在";
        } else if (exception instanceof BadCredentialsException) {
            message = "用户名或密码错误";
        } else if (exception instanceof LockedException) {
            message = "账户已被锁定";
        } else if (exception instanceof DisabledException) {
            message = "账户已被禁用";
        } else if (exception instanceof AccountExpiredException) {
            message = "账户已过期";
        } else if (exception instanceof CredentialsExpiredException) {
            message = "密码已过期";
        } else {
            message = "登录失败: " + exception.getMessage();
        }

        Map<String, Object> result = Map.of(
            "code", 401,
            "message", message,
            "data", (Object) null
        );

        response.getWriter().write(new ObjectMapper().writeValueAsString(result));
    }
}
```

## 1.2 AccessDeniedHandler

**权限不足时触发**（已登录但无访问权限），返回 403：

```java
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) 
            throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType("application/json;charset=UTF-8");

        Map<String, Object> result = Map.of(
            "code", 403,
            "message", "无访问权限",
            "data", (Object) null
        );

        response.getWriter().write(new ObjectMapper().writeValueAsString(result));
    }
}
```

## 1.3 AuthenticationEntryPoint

**未登录访问受保护资源时触发**，返回 401：

```java
@Component
public class CustomAuthenticationEntryPoint 
        implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) 
            throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");

        Map<String, Object> result = Map.of(
            "code", 401,
            "message", "请先登录",
            "data", (Object) null
        );

        response.getWriter().write(new ObjectMapper().writeValueAsString(result));
    }
}
```

**三种异常的处理链：**

```
未登录 → AuthenticationEntryPoint → 返回 401
登录失败 → AuthenticationFailureHandler → 返回 401
无权限 → AccessDeniedHandler → 返回 403
```

## 1.4 异常处理流程

```java
http.exceptionHandling(exceptions -> exceptions
    .authenticationEntryPoint(customAuthenticationEntryPoint)   // 401
    .accessDeniedHandler(customAccessDeniedHandler)              // 403
);
```

---

# 2. 认证成功处理器

## 2.1 AuthenticationSuccessHandler

```java
@Component
public class CustomAuthenticationSuccessHandler 
        implements AuthenticationSuccessHandler {

    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException {
        // 生成 JWT
        String token = jwtTokenProvider.generateToken(authentication);

        // 获取用户信息
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("code", 200);
        result.put("message", "登录成功");
        result.put("data", Map.of(
            "token", token,
            "tokenType", "Bearer",
            "username", userDetails.getUsername(),
            "authorities", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList()
        ));

        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(new ObjectMapper().writeValueAsString(result));
    }
}
```

## 2.2 登录成功后跳转或返回数据

**表单登录场景（重定向）：**

```java
@Component
public class RedirectSuccessHandler 
        extends SavedRequestAwareAuthenticationSuccessHandler {

    public RedirectSuccessHandler() {
        setDefaultTargetUrl("/index");      // 默认跳转
        setAlwaysUseDefaultTargetUrl(false); // 优先跳转登录前想访问的URL
    }
}
```

**前后端分离场景（返回 JSON）：**

```java
http.formLogin(form -> form
    .successHandler(customAuthenticationSuccessHandler) // 返回 JSON
    .failureHandler(customAuthenticationFailureHandler)
);
```

---

# 3. 认证失败处理器

## 3.1 AuthenticationFailureHandler

```java
@Component
public class LoginFailureHandler implements AuthenticationFailureHandler {

    private static final int MAX_RETRY_COUNT = 5;

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException exception)
            throws IOException {

        String username = request.getParameter("username");

        // 可选：记录登录失败次数，超过阈值锁定账户
        // redisTemplate.opsForValue().increment("login:fail:" + username);

        response.setStatus(401);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":401,\"msg\":\"登录失败\"}");
    }
}
```

## 3.2 登录失败逻辑

常见处理：
- 记录失败次数 → 达到上限锁定账号
- 记录登录 IP 和时间 → 安全审计
- 返回具体失败原因（注意不要泄露用户是否存在）

---

# 4. 登出成功处理器

## 4.1 LogoutSuccessHandler

```java
@Component
public class CustomLogoutSuccessHandler implements LogoutSuccessHandler {

    @Override
    public void onLogoutSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException {

        // 可选：将 JWT Token 加入黑名单
        // String token = resolveToken(request);
        // tokenBlacklist.add(token);

        response.setContentType("application/json;charset=UTF-8");

        Map<String, Object> result = Map.of(
            "code", 200,
            "message", "已成功退出登录"
        );

        response.getWriter().write(new ObjectMapper().writeValueAsString(result));
    }
}
```

## 4.2 自定义登出返回数据

**在 SecurityConfig 中注册所有自定义 Handler：**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private CustomAuthenticationSuccessHandler successHandler;
    @Autowired
    private CustomAuthenticationFailureHandler failureHandler;
    @Autowired
    private CustomLogoutSuccessHandler logoutSuccessHandler;
    @Autowired
    private CustomAuthenticationEntryPoint entryPoint;
    @Autowired
    private CustomAccessDeniedHandler accessDeniedHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .formLogin(form -> form
                .loginProcessingUrl("/api/auth/login")
                .successHandler(successHandler)
                .failureHandler(failureHandler)
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/api/auth/logout")
                .logoutSuccessHandler(logoutSuccessHandler)
            )
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(entryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            );

        return http.build();
    }
}
```

**完整异常处理流程图：**

```
HTTP 请求到达
    │
    ├─ 未登录 → ExceptionTranslationFilter
    │              └─ AuthenticationEntryPoint → 返回 401
    │
    ├─ 登录请求 POST /login
    │    ├─ 成功 → AuthenticationSuccessHandler → 返回 200 + JWT
    │    └─ 失败 → AuthenticationFailureHandler → 返回 401
    │
    ├─ 已登录但权限不足 → AuthorizationFilter
    │    └─ AccessDeniedHandler → 返回 403
    │
    └─ 登出请求 /logout
         └─ LogoutSuccessHandler → 返回 200
```
