# 1. Starter 引入

## 1.1 spring-boot-starter-security

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    <!-- 无需指定版本，spring-boot-starter-parent 管理 -->
</dependency>
```

**传递依赖（关键组件）：**

| 依赖 | 作用 |
|------|------|
| `spring-security-core` | 认证授权核心 |
| `spring-security-config` | Java Config 支持 |
| `spring-security-web` | Web 安全过滤器链 |
| `spring-security-oauth2-core` | OAuth2 核心 |
| `spring-security-crypto` | 密码编码器 |

## 1.2 自动配置机制

`spring-boot-starter-security` 通过 `@EnableAutoConfiguration` 自动注册以下 Bean：

```
SecurityAutoConfiguration
├── springSecurityFilterChain (DelegatingFilterProxy)
│   └── 默认 SecurityFilterChain
│       ├── CSRF 已启用
│       ├── 所有端点需要认证
│       ├── 表单登录
│       ├── HTTP Basic
│       └── 登出
├── PasswordEncoder (DelegatingPasswordEncoder)
├── AuthenticationManager
├── AuthenticationEventPublisher
└── 默认 UserDetailsService (InMemoryUserDetailsManager)
```

**自定义 Bean 会取代默认行为：**

```java
// 定义一个 SecurityFilterChain Bean → 默认的会被替换
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .build();
}
```

> 如果希望**自定义配置与默认配置共存**，需在 Bean 上添加 `@Order` 注解来定义多个 `SecurityFilterChain`。

---

# 2. 自动登录页面

## 2.1 默认登录页特点

Spring Security 通过 `DefaultLoginPageGeneratingFilter` 生成默认登录页：

- 纯 HTML，无任何外部依赖
- 支持表单登录和 OAuth2 登录按钮
- 支持显示错误信息（`?error` 参数）
- 支持登出成功信息（`?logout` 参数）

## 2.2 自定义登录页示例

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.formLogin(form -> form
        .loginPage("/login.html")        // 自定义登录页（需自己提供）
        .loginProcessingUrl("/doLogin")  // 表单提交地址
        .usernameParameter("username")   // 用户名参数名（默认 username）
        .passwordParameter("password")   // 密码参数名（默认 password）
        .defaultSuccessUrl("/index", true) // 登录成功默认跳转
        .failureUrl("/login.html?error")   // 失败跳转
        .permitAll()
    );
    return http.build();
}
```

**自定义登录页注意事项：**

```html
<!-- 必须使用 POST 方法提交 -->
<form action="/doLogin" method="post">
    <!-- 默认参数名为 username 和 password -->
    <input type="text" name="username">
    <input type="password" name="password">
    <button type="submit">登录</button>

    <!-- 如果未禁用 CSRF，必须包含这行 -->
    <input type="hidden" th:name="${_csrf.parameterName}" 
                         th:value="${_csrf.token}">
</form>
```

---

# 3. 配置属性

## 3.1 application.yml 配置安全策略

```yaml
spring:
  security:
    user:
      name: admin                    # 默认管理员用户名
      password: $2a$10$...           # 加密后的密码
      roles:                         # 默认角色
        - ADMIN
        - USER

  # Spring Security 6.x 官方配置属性前缀
  # spring.security.user.* 主要用于开发测试
```

## 3.2 常用配置属性

```yaml
spring:
  security:
    # 用户名密码（仅开发环境使用）
    user:
      name: admin
      password: admin123

    # OAuth2 Client 配置
    oauth2:
      client:
        registration:
          github:
            client-id: your-client-id
            client-secret: your-client-secret
            scope: read:user,user:email
          google:
            client-id: your-client-id
            client-secret: your-client-secret
        provider:
          github:
            authorization-uri: https://github.com/login/oauth/authorize
            token-uri: https://github.com/login/oauth/access_token
            user-info-uri: https://api.github.com/user

      # OAuth2 Resource Server 配置
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

**自定义属性注入：**

```yaml
# 自定义安全配置
app:
  security:
    jwt:
      secret: ${JWT_SECRET:default-secret-key}
      expiration: 86400000  # 24小时
      refresh-expiration: 604800000  # 7天
    login:
      max-retry-count: 5
      lock-duration: 30  # 分钟
```

```java
@Component
@ConfigurationProperties(prefix = "app.security")
@Data
public class SecurityProperties {
    private JwtProperties jwt = new JwtProperties();
    private LoginProperties login = new LoginProperties();

    @Data
    public static class JwtProperties {
        private String secret;
        private long expiration;
        private long refreshExpiration;
    }

    @Data
    public static class LoginProperties {
        private int maxRetryCount;
        private int lockDuration;
    }
}
```

---

# 4. 与 Spring MVC 集成

## 4.1 控制器中使用 SecurityContext

```java
@RestController
@RequestMapping("/api/user")
public class UserController {

    // 方式一：注入 Authentication
    @GetMapping("/profile")
    public UserProfile getProfile(Authentication authentication) {
        String username = authentication.getName();
        return userService.getProfileByUsername(username);
    }

    // 方式二：注入 Principal
    @GetMapping("/info")
    public Map<String, Object> getInfo(Principal principal) {
        return Map.of("username", principal.getName());
    }

    // 方式三：注入 @AuthenticationPrincipal（推荐）
    @GetMapping("/detail")
    public UserDetail getDetail(
            @AuthenticationPrincipal UserDetails userDetails) {
        return userService.getDetail(userDetails.getUsername());
    }

    // 方式四：自定义 Principal 注解
    @GetMapping("/me")
    public UserVO getCurrentUser(
            @AuthenticationPrincipal CustomUserPrincipal principal) {
        return userService.getCurrentUser(principal.getId());
    }

    // 方式五：手动获取
    @GetMapping("/roles")
    public List<String> getRoles() {
        Authentication auth = SecurityContextHolder.getContext()
            .getAuthentication();
        return auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());
    }
}
```

## 4.2 方法级注解与 URL 过滤结合

**最佳实践：URL 级做粗粒度控制，方法级做细粒度控制：**

```java
// SecurityConfig —— URL 级（第一道防线）
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")   // 必须管理员
    .requestMatchers("/api/**").authenticated()       // 必须登录
    .requestMatchers("/public/**").permitAll()
);
```

```java
// Service —— 方法级（第二道防线）
@Service
public class UserService {

    @PreAuthorize("hasAuthority('user:read')")
    public List<User> findAllUsers() { ... }

    @PreAuthorize("hasAuthority('user:create')")
    public User createUser(User user) { ... }

    @PreAuthorize("hasAuthority('user:delete') or #id == authentication.principal.id")
    public void deleteUser(Long id) { ... }
}
```

## 4.3 在视图中使用安全标签

**Thymeleaf 集成：**

```xml
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>
```

```html
<!-- 显示当前用户名 -->
<div sec:authentication="name"></div>

<!-- 仅登录用户可见 -->
<div sec:authorize="isAuthenticated()">
    <a href="/logout">退出</a>
</div>

<!-- 仅管理员可见 -->
<div sec:authorize="hasRole('ADMIN')">
    <a href="/admin">管理后台</a>
</div>

<!-- 仅特定权限可见 -->
<button sec:authorize="hasAuthority('user:delete')">
    删除用户
</button>
```

## 4.4 多个 SecurityFilterChain 共存

当不同的 URL 路径需要不同的安全策略时，可定义多个 `SecurityFilterChain`，用 `@Order` 指定优先级：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // API 安全策略（优先级高）
    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")  // 仅匹配 /api/**
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
        return http.build();
    }

    // 管理后台策略
    @Bean
    @Order(2)
    public SecurityFilterChain adminFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/admin/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().hasRole("ADMIN"));
        return http.build();
    }

    // 默认策略（兜底）
    @Bean
    @Order(3)
    public SecurityFilterChain defaultFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

**`securityMatcher()` vs `requestMatchers()`：**

| 方法 | 作用 | 层级 |
|------|------|------|
| `.securityMatcher()` | 限定此 `SecurityFilterChain` 作用于哪些 URL | FilterChain 级别 |
| `.authorizeHttpRequests().requestMatchers()` | 在同一个 FilterChain 内细分权限 | 请求级别 |
