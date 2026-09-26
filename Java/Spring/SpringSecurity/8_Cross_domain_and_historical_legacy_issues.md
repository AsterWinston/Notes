# 1. 跨域（CORS）

## 1.1 CORS 基本概念

浏览器的**同源策略**限制跨域请求。CORS (Cross-Origin Resource Sharing) 允许服务端声明哪些源可以访问。

```
前端: http://localhost:3000
后端: http://localhost:8080
→ 协议、域名、端口任一不同即为跨域
```

**CORS 相关 HTTP 响应头：**

| 响应头 | 说明 |
|--------|------|
| `Access-Control-Allow-Origin` | 允许的源 |
| `Access-Control-Allow-Methods` | 允许的 HTTP 方法 |
| `Access-Control-Allow-Headers` | 允许的请求头 |
| `Access-Control-Allow-Credentials` | 是否允许携带 Cookie |
| `Access-Control-Max-Age` | 预检请求缓存时间(秒) |
| `Access-Control-Expose-Headers` | 前端可读取的响应头 |

**请求分类：**
- **简单请求**：GET/POST/HEAD + 标准头 → 直接发送
- **预检请求**：PUT/DELETE 或自定义头 → 先发 OPTIONS 请求确认

## 1.2 Spring Boot 配置 CORS

### 方式一：WebMvcConfigurer 全局配置

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")           // 对哪些路径生效
            .allowedOrigins("http://localhost:3000") // 允许的源
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")                     // 允许的请求头
            .allowCredentials(true)                  // 允许携带 Cookie
            .maxAge(3600);                           // 预检缓存
    }
}
```

> **注意：** `WebMvcConfigurer` 方式配置的 CORS 在 Spring Security 过滤器链中**可能失效**，因为请求走到 CorsFilter 前就被 Security 拦截了。

### 方式二：Spring Security 中配置（推荐）

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors
                .configurationSource(corsConfigurationSource())  // 自定义 CORS 配置
            )
            // ... 其他配置
        ;
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of(
            "http://localhost:3000",
            "http://localhost:5173",    // Vite
            "https://www.example.com"
        ));
        configuration.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### 方式三：`@CrossOrigin` 注解（单个 Controller）

```java
@RestController
@CrossOrigin(origins = "http://localhost:3000")
public class UserController {

    @GetMapping("/api/users")
    @CrossOrigin(origins = {"http://localhost:3000", "http://localhost:5173"})
    public List<User> getUsers() { ... }
}
```

## 1.3 前后端分离注意事项

1. **`allowCredentials(true)` 时 `allowedOrigins` 不能为 `*`**，必须指定具体域名
2. **PUT、DELETE 等非简单方法**会先发 OPTIONS 预检，确保 OPTIONS 请求不被拦截
3. **Nginx 反向代理**优先于 CORS：如果在同域下反代，不需要处理 CORS

---

# 2. 遗留小问题

## 2.1 Session Fixation 防护

Spring Security 默认已启用，登录成功后会自动更换 Session ID。

```java
http.sessionManagement(session -> session
    .sessionFixation().changeSessionId()  // 默认行为，无需显式配置
);
```

**验证：** 在 Chrome DevTools → Application → Cookies 中查看，登录前后 `JSESSIONID` 值会改变。

## 2.2 并发登录控制

```java
http.sessionManagement(session -> session
    .maximumSessions(1)               // 同一账号最多1个会话
    .maxSessionsPreventsLogin(false)  // false: 踢旧; true: 阻止新
    .expiredUrl("/login?expired")     // 过期后跳转
);

// 自定义踢出处理
http.sessionManagement(session -> session
    .maximumSessions(1)
    .expiredSessionStrategy(new CustomExpiredSessionStrategy())
);
```

**实现原理：** Spring Security 内部维护 `SessionRegistry`，所有登录的 Session 都注册其中。新登录时检查已有会话数，超过限制则标记旧 Session 为过期。

```java
@Bean
public SessionRegistry sessionRegistry() {
    return new SessionRegistryImpl();
}

@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    // 必须注册这个 Bean，否则 SessionRegistry 无法感知 Session 销毁
    return new HttpSessionEventPublisher();
}
```

## 2.3 CSRF 与 REST API 冲突

**场景：** 前后端分离架构中，JWT 认证是无状态的，不存在 Session 劫持风险，因此 CSRF 防护无意义。

**结论：REST API + JWT → 禁用 CSRF**

```java
http.csrf(AbstractHttpConfigurer::disable);
```

**但如果 API 仍用 Session + Cookie：**

```java
// 使用 CookieCsrfTokenRepository（适合前后端分离+Session的场景）
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

## 2.4 Filter 顺序问题

当有**自定义 Filter** 时，需注意它在过滤器链中的位置：

```java
http.addFilterBefore(myFilter, UsernamePasswordAuthenticationFilter.class); // 之前
http.addFilterAfter(myFilter, UsernamePasswordAuthenticationFilter.class);  // 之后
http.addFilterAt(myFilter, UsernamePasswordAuthenticationFilter.class);     // 替换位置
```

**常见自定义 Filter 位置建议：**

| 自定义 Filter | 建议位置 |
|---------------|----------|
| JWT Token 校验 | Before `UsernamePasswordAuthenticationFilter` |
| 验证码校验 | Before `UsernamePasswordAuthenticationFilter` |
| 操作日志记录 | After `SecurityContextHolderFilter` |
| 请求限流 | Before `SecurityContextHolderFilter` |

```java
// JWT 认证过滤器示例
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .addFilterBefore(jwtAuthenticationFilter, 
            UsernamePasswordAuthenticationFilter.class)
        .addFilterBefore(smsCodeValidationFilter, 
            UsernamePasswordAuthenticationFilter.class)
        // ...
    ;
    return http.build();
}
```

## 2.5 安全响应头

Spring Security 默认添加以下安全响应头（`HeaderWriterFilter`）：

| 响应头 | 默认值 | 作用 |
|--------|--------|------|
| `X-Frame-Options` | `DENY` | 防止 Clickjacking（点击劫持） |
| `X-Content-Type-Options` | `nosniff` | 防止 MIME 类型嗅探 |
| `X-XSS-Protection` | `0` | 6.x 中默认关闭（浏览器已废弃） |
| `Strict-Transport-Security` | `max-age=31536000 ; includeSubDomains` | HSTS |
| `Content-Security-Policy` | 默认不设 | CSP（需自定义） |
| `Cache-Control` | 保护页面不被缓存 | |
| `Referrer-Policy` | 默认不设 | |

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())           // X-Frame-Options
    .contentSecurityPolicy(csp -> csp              // Content-Security-Policy
        .policyDirectives("default-src 'self'"))
    .httpStrictTransportSecurity(hsts -> hsts      // HSTS
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000))
);
```
