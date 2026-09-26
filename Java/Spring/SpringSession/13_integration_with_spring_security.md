# 1. 集成概述

Spring Session 与 Spring Security 的集成是实现**分布式安全架构**的关键。Spring Security 负责认证授权，Spring Session 负责会话存储和管理，两者结合可以构建完整的分布式安全体系。

```
Spring Security                         Spring Session
┌─────────────────────┐              ┌──────────────────┐
│ 认证 (Authentication) │              │ Session 外部存储   │
│ 授权 (Authorization)  │◄── 协同 ──► │ Session 索引      │
│ 并发会话控制          │              │ Session 事件      │
│ CSRF 防护             │              │ Session 生命周期  │
│ Session Fixation 防护 │              │ 分布式 Session    │
└─────────────────────┘              └──────────────────┘
```

---

# 2. 自动集成机制

## 2.1 自动配置

当同时引入 `spring-security` 和 `spring-session-*` 依赖时，Spring Boot 自动配置会：

1. **SecurityContextPersistenceFilter** → 不再将 `SecurityContext` 持久化到本地 Session，而是委托给 Spring Session
2. **SessionRegistry** → 自动使用 `SpringSessionBackedSessionRegistry`
3. **并发会话控制** → 自动支持跨实例的并发会话管理
4. **Session Fixation** → 使用 `SessionRepository` 的 `changeSessionId()` 实现

## 2.2 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

无需额外配置，Spring Security 自动使用 Spring Session 来管理 `SecurityContext`。

---

# 3. SecurityContext 与 Session 的关系

## 3.1 存储流程

```
用户认证成功
  │
  ├── UsernamePasswordAuthenticationFilter
  │   └── 创建 Authentication 对象
  │
  ├── SecurityContextHolder.setContext()
  │   └── 设置到当前线程
  │
  └── SecurityContextPersistenceFilter (响应提交时)
      └── 从 SecurityContextHolder 获取 SecurityContext
      └── session.setAttribute("SPRING_SECURITY_CONTEXT", context)
      └── SessionRepositoryFilter 将 Session 持久化到 Redis/DB
```

## 3.2 读取流程

```
用户请求到达
  │
  ├── SessionRepositoryFilter
  │   └── 从 Redis/DB 加载 Session
  │
  ├── SecurityContextPersistenceFilter
  │   └── 从 Session 读取 SPRING_SECURITY_CONTEXT
  │   └── 设置到 SecurityContextHolder
  │
  └── 后续 Filter 和 Controller 可使用 SecurityContextHolder
```

## 3.3 代码验证

```java
@RestController
public class SecurityContextController {

    @GetMapping("/show-session-security")
    public Map<String, Object> showSessionSecurity(HttpSession session) {
        Map<String, Object> result = new HashMap<>();

        // 从 Session 获取 Spring Security 上下文
        SecurityContext context = (SecurityContext) session
            .getAttribute("SPRING_SECURITY_CONTEXT");

        if (context != null) {
            Authentication auth = context.getAuthentication();
            result.put("authenticated", auth.isAuthenticated());
            result.put("username", auth.getName());
            result.put("authorities", auth.getAuthorities());
        }

        // 从 SecurityContextHolder 获取（两种方式等效）
        Authentication currentAuth =
            SecurityContextHolder.getContext().getAuthentication();
        result.put("currentUser", currentAuth != null ?
            currentAuth.getName() : null);

        result.put("sessionId", session.getId());
        return result;
    }
}
```

---

# 4. SessionManagement 配置

## 4.1 基本配置

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .sessionManagement(session -> session
                // 会话创建策略
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)

                // 限制最大并发会话数
                .maximumSessions(1)

                // 超出最大会话数时是否阻止新登录
                .maxSessionsPreventsLogin(false)
                // true: 阻止新登录，提示错误
                // false: 踢掉最旧的会话，允许新登录

                // 会话过期跳转地址
                .expiredUrl("/login?expired")

                // 会话过期策略
                .expiredSessionStrategy(event -> {
                    event.getResponse().setContentType("application/json");
                    event.getResponse().setCharacterEncoding("UTF-8");
                    event.getResponse().getWriter().write(
                        "{\"code\":401,\"message\":\"会话已过期\"}"
                    );
                })
            );

        return http.build();
    }
}
```

## 4.2 SessionCreationPolicy 策略

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| **ALWAYS** | 始终创建 Session | 传统 Web 应用（安全审计） |
| **IF_REQUIRED** | 需要时才创建（默认） | 大多数应用 |
| **NEVER** | 不创建，但可使用已有 Session | REST API + 可能复用 Cookie |
| **STATELESS** | 完全不使用 Session | 纯 JWT/Token 认证 |

```java
// STATELESS 模式下的配置
http
    .sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
    )
    .csrf(AbstractHttpConfigurer::disable); // 无 Session 时需禁用 CSRF
```

---

# 5. SessionRegistry 集成

## 5.1 自动配置

Spring Session 自动提供 `SpringSessionBackedSessionRegistry`，它是 `SessionRegistry` 的实现：

```java
@Configuration
public class SessionRegistryConfig {

    @Bean
    public SessionRegistry sessionRegistry(
            FindByIndexNameSessionRepository<?> sessionRepository) {
        return new SpringSessionBackedSessionRegistry<>(sessionRepository);
    }
}
```

## 5.2 使用 SessionRegistry

```java
@RestController
@RequestMapping("/admin/sessions")
public class SessionManagementController {

    @Autowired
    private SessionRegistry sessionRegistry;

    @Autowired
    private FindByIndexNameSessionRepository<?> sessionRepository;

    // 获取所有在线用户
    @GetMapping("/online-users")
    public List<String> getOnlineUsers() {
        return sessionRegistry.getAllPrincipals().stream()
            .map(Object::toString)
            .collect(Collectors.toList());
    }

    // 获取某用户的所有会话
    @GetMapping("/user-sessions")
    public List<String> getUserSessions(@RequestParam String username) {
        return sessionRegistry.getAllSessions(username, false).stream()
            .map(s -> s.getSessionId())
            .collect(Collectors.toList());
    }

    // 强制过期某用户的会话
    @PostMapping("/expire")
    public String expireSession(@RequestParam String sessionId) {
        SessionInformation info = sessionRegistry
            .getSessionInformation(sessionId);
        if (info != null) {
            info.expireNow();
            return "Session " + sessionId + " 已过期";
        }
        return "Session 不存在";
    }

    // 强制过期某用户的所有会话
    @PostMapping("/expire-user")
    public String expireUserSessions(@RequestParam String username) {
        Map<String, ? extends Session> sessions = sessionRepository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );
        sessions.keySet().forEach(sessionRepository::deleteById);
        return "用户 " + username + " 的所有会话已强制下线";
    }
}
```

---

# 6. 并发会话控制详解

## 6.1 运作机制

```
用户 "zhangsan" 登录请求到达
  │
  ├── 认证成功
  │
  ├── ConcurrentSessionControlAuthenticationStrategy
  │   └── 检查 "zhangsan" 当前有多少活跃 Session
  │
  ├── 已到达 maximumSessions 限制？
  │   │
  │   ├── Yes AND maxSessionsPreventsLogin=true
  │   │   └── 抛出 AuthenticationException → 登录失败
  │   │
  │   └── Yes AND maxSessionsPreventsLogin=false
  │       └── 标记旧会话为 expired
  │       └── 发布 SessionInformationExpiredEvent
  │       └── 允许新登录
  │
  └── 注册新会话到 SessionRegistry
```

## 6.2 前后端分离下的并发控制

```java
@Configuration
public class RestConcurrencyConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .sessionManagement(session -> session
                .maximumSessions(1)
                .maxSessionsPreventsLogin(true) // 阻止新登录
                .expiredSessionStrategy(event -> {
                    // REST API 方式返回 JSON 错误
                    HttpServletResponse response = event.getResponse();
                    response.setStatus(401);
                    response.setContentType("application/json;charset=UTF-8");
                    response.getWriter().write("""
                        {
                            "code": 401,
                            "message": "您的账号已在其他设备登录，当前会话已失效"
                        }
                        """);
                })
            );
        return http.build();
    }
}
```

## 6.3 跨服务并发控制

```
┌─────────────┐     ┌─────────────┐
│  认证服务 A   │     │  认证服务 B   │
│  (8080)     │     │  (8081)     │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └───────┬───────────┘
               │
      ┌────────▼────────┐
      │  Redis (共享的   │
      │  SessionRegistry)│
      └─────────────────┘

无论用户被路由到哪个实例，
SessionRegistry 都能正确统计和限制并发会话数。
```

---

# 7. 自定义认证成功/失败处理

## 7.1 认证成功处理

```java
@Component
public class CustomAuthenticationSuccessHandler
        implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication auth)
            throws IOException {

        HttpSession session = request.getSession();

        // 记录登录信息到 Session
        session.setAttribute("LOGIN_TIME", Instant.now());
        session.setAttribute("LOGIN_IP", request.getRemoteAddr());
        session.setAttribute("USER_AGENT", request.getHeader("User-Agent"));

        // REST API 返回 JSON
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(String.format(
            "{\"code\":200,\"message\":\"登录成功\",\"sessionId\":\"%s\"}",
            session.getId()
        ));
    }
}
```

## 7.2 认证失败处理

```java
@Component
public class CustomAuthenticationFailureHandler
        implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException ex)
            throws IOException {

        response.setStatus(401);
        response.setContentType("application/json;charset=UTF-8");

        String message = switch (ex) {
            case BadCredentialsException e -> "用户名或密码错误";
            case SessionAuthenticationException e -> "会话认证失败";
            case LockedException e -> "账户已被锁定";
            case DisabledException e -> "账户已被禁用";
            default -> "认证失败";
        };

        response.getWriter().write(
            "{\"code\":401,\"message\":\"" + message + "\"}"
        );
    }
}
```

---

# 8. RememberMe 与 Spring Session

## 8.1 配置 RememberMe

```java
@Configuration
@EnableWebSecurity
public class RememberMeConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .rememberMe(remember -> remember
                .key("uniqueAndSecret")
                .tokenValiditySeconds(604800) // 7 天
                .rememberMeParameter("remember-me")
            );
        return http.build();
    }
}
```

## 8.2 RememberMe 与 Session 的关系

```
用户勾选"记住我"登录
  │
  ├── RememberMeAuthenticationFilter
  │   └── 从 Cookie 解析 RememberMe Token
  │   └── 自动认证用户
  │
  └── 创建新的 HttpSession
      └── 将 Authentication 存入 Session
      └── Session 持久化到 Redis

下次请求：
  ├── RememberMe Token 仍有效
  ├── Spring Session 加载已有 Session（包含 Authentication）
  └── 用户无需重新登录
```

**注意**：`RememberMe` 使用的是独立的 Cookie（`remember-me`），与 Session Cookie（`SESSION`）不同，两者独立工作。

---

# 9. Session Fixation 防护

## 9.1 攻击原理

```
攻击者诱导用户使用攻击者已知的 Session ID，
用户登录后，攻击者也可以使用同一 Session ID 冒充用户。
```

## 9.2 防护机制

Spring Security 默认通过 `SessionFixationProtectionStrategy` 防御：

```java
// 默认行为：登录成功后更换 Session ID
// 在 Spring Session 环境下：
// session.changeSessionId() → sessionRepository.changeSessionId()
// 会创建新的 Session 数据并删除旧的

@Configuration
public class SessionFixationConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .sessionManagement(session -> session
                .sessionFixation()
                // changeSessionId: 登录后更换 Session ID（默认）
                // newSession: 创建全新 Session
                // migrateSession: 迁移 Session 数据到新 ID
                // none: 不处理（不推荐）
                .migrateSession()
            );
        return http.build();
    }
}
```

## 9.3 策略对比

| 策略 | 行为 | 安全性 | 用户体验 |
|------|------|--------|----------|
| **changeSessionId** | 更换 ID，保留数据 | 高 | 无感知（默认） |
| **newSession** | 创建新 Session，复制数据 | 最高 | 无感知 |
| **migrateSession** | 迁移数据到新 ID | 中 | 无感知 |
| **none** | 不做处理 | 低 | 无差异 |
