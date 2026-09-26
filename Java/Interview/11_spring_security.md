# 1. 知识点
- **认证（Authentication）**：验证用户身份的过程，确定"你是谁"。常用方式：用户名密码、短信验证码、生物识别、JWT Token。
- **授权（Authorization）**：确定用户权限的过程，决定"你能做什么"。常用模型：RBAC（基于角色的访问控制）、ABAC（基于属性）、ACL（访问控制列表）。
- **认证流程**：`UsernamePasswordAuthenticationFilter` 拦截登录请求 → 封装 `UsernamePasswordAuthenticationToken` → 调用 `AuthenticationManager` → `ProviderManager` 遍历 `AuthenticationProvider` → `DaoAuthenticationProvider` 调用 `UserDetailsService.loadUserByUsername()` → 密码比对 `PasswordEncoder.matches()` → 认证成功返回已认证的 `Authentication` → 存入 `SecurityContextHolder`。
- **SecurityContext**：存储当前用户的 `Authentication` 对象。`SecurityContextHolder` 默认使用 `MODE_THREADLOCAL`（每个线程独立），可选 `MODE_INHERITABLETHREADLOCAL`（子线程可继承）、`MODE_GLOBAL`（全局共享）。
- **`SecurityContextHolder.getContext().getAuthentication()`**：获取当前登录用户的认证信息（包含 Principal、Credentials、Authorities）。
- **SecurityFilterChain**：Spring Security 核心，组织一系列 Security Filter 形成过滤器链。每个 `SecurityFilterChain` 匹配特定请求路径。
- **DelegatingFilterProxy**：Servlet Container 和 Spring 容器之间的桥梁 Filter，将请求委托给 Spring 管理的 `FilterChainProxy` Bean。
- **FilterChainProxy**：`SecurityFilterChain` 的入口，根据请求匹配对应的 `SecurityFilterChain` 并执行其中的 Security Filters。
- **内置 Filter 列表（按大致执行顺序）**：`ChannelProcessingFilter` → `SecurityContextPersistenceFilter` → `ConcurrentSessionFilter` → `UsernamePasswordAuthenticationFilter` → `DefaultLoginPageGeneratingFilter` → `BasicAuthenticationFilter` → `RequestCacheAwareFilter` → `SecurityContextHolderAwareRequestFilter` → `RememberMeAuthenticationFilter` → `AnonymousAuthenticationFilter` → `ExceptionTranslationFilter` → `FilterSecurityInterceptor`。
- **PasswordEncoder**：密码加密接口。`BCryptPasswordEncoder` 是目前推荐实现，每次 salt 不同（BCrypt 内置 salt），加密后字符串包含算法版本、强度和 salt 信息。
- **DelegatingPasswordEncoder**：代理密码编码器，支持多格式密码共存。通过密码前缀标识算法（如 `{bcrypt}$2a$10$...`、`{noop}plaintext`），用于密码算法平滑升级。
- **JWT（JSON Web Token）**：三段式结构：`Header.Payload.Signature`，Base64Url 编码。Header 声明算法（HS256/RS256）；Payload 包含声明（Claims），如用户 ID、过期时间；Signature 防篡改。
- **JWT 优劣势**：优势——无状态、不依赖服务端存储、适合分布式系统、跨域友好；劣势——无法主动失效（除非用黑名单）、payload 仅 Base64 非加密（敏感信息需服务端加密)、Token 体积较大。
- **双 Token 模式**：AccessToken（短期，5-15 分钟）+ RefreshToken（长期，7-30 天）。AccessToken 过期后用 RefreshToken 换取新的 AccessToken。RefreshToken 安全级别更高（存储于数据库，可主动撤销）。
- **OAuth2 授权模式**：授权码模式（Authorization Code，最安全，有后端即可用）、客户端凭证（Client Credentials，服务间调用）、设备码（Device Code，TV/打印机）、简化模式（Implicit，SPA，已废弃）、密码模式（Password，已废弃，不安全）。
- **授权码模式流程**：用户点击授权 → 跳转到授权服务器的授权页面 → 用户同意 → 授权服务器颁发授权码（Authorization Code）→ 客户端用授权码换取 AccessToken（和可选的 RefreshToken）。
- **Spring Authorization Server**：Spring 官方 OAuth2.1 授权服务器实现。替代已废弃的 Spring Security OAuth2，提供 `/oauth2/authorize`（授权端点）、`/oauth2/token`（令牌端点）、`/oauth2/introspect`（令牌内省）、OIDC 支持。
- **Resource Server**：资源服务器，保护 API 资源。通过解析请求中的 Bearer Token（JWT/opaque）验证请求合法性。配置 `jwk-set-uri` 或 `issuer-uri` 即可集成。
- **CSRF（跨站请求伪造）**：攻击者诱导用户点击恶意链接，利用用户已登录的 Cookie 发送伪造请求。Spring Security 默认通过同步令牌模式（Synchronizer Token Pattern）防护。
- **`csrf().disable()`**：前后端分离架构且使用 JWT/Bearer Token（非 Cookie）时可关闭 CSRF，因为请求通过 Authorization 头传 Token，不依赖浏览器 Cookie。
- **CORS（跨域）**：Spring Security 通过 `.cors()` 方法启用 `CorsConfigurationSource`，与 Spring MVC 的 `@CrossOrigin` 互补。需注意 `allowCredentials=true` 时不能设置 `allowedOrigins("*")`。
- **方法级安全**：`@PreAuthorize("hasRole('ADMIN')")`（方法执行前权限校验，SpEL 表达式）、`@PostAuthorize("returnObject.userId == principal.id")`（方法执行后校验返回值）、`@Secured("ROLE_ADMIN")`（Spring 原生，不支持 SpEL）、`@RolesAllowed`（JSR-250 标准）。
- **RBAC（基于角色的访问控制）**：用户 → 角色 → 权限（资源+操作）的三层模型。用户拥有角色，角色拥有权限，通过角色关联实现灵活授权。
- **动态权限**：权限数据存储在数据库，运行时从数据库加载 URL-角色映射，通过自定义 `FilterSecurityInterceptor` 或 `AccessDecisionVoter` 实现动态判断。
- **Voter 机制**：Spring Security 访问决策投票器，`AffirmativeBased`（一票通过）、`ConsensusBased`（多数票）、`UnanimousBased`（全票通过）。`RoleVoter` 和 `AuthenticatedVoter` 是常用实现。
- **Remember Me**：记住我功能，基于 Token 实现持久化登录。`PersistentTokenBasedRememberMeServices` 将 Token 存储数据库，安全性更高。
- **Session 管理**：`sessionCreationPolicy(SessionCreationPolicy.STATELESS)` 设置无状态（JWT 场景常用）；`maximumSessions(1)` 控制并发会话数防止一个账号多地登录；集群通过 Spring Session 实现 Session 共享。
- **自定义认证**：实现 `AuthenticationProvider` 接口，覆盖 `authenticate()` 和 `supports()` 方法。支持短信验证码登录、第三方 OAuth 登录等场景。
- **Spring Security OAuth2（已废弃）**：`@EnableAuthorizationServer`、`@EnableResourceServer`、`@EnableOAuth2Client` 三大注解，旧项目中仍可遇到。Spring Security 5+ 推荐使用 Spring Authorization Server。
- **`@EnableWebSecurity`**：启用 Spring Security 的 Web 安全支持，通常与 `@Configuration` 配合使用（Spring Boot 自动配置下无需手动添加）。
- **SecurityContext 传递**：`@Async` 方法默认不继承 SecurityContext，需配置 `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)` 或使用 `DelegatingSecurityContextAsyncTaskExecutor`。
- **`WebSecurityConfigurerAdapter`**（已废弃）：Spring Security 5.7 前用于自定义安全配置，现已推荐使用基于 Bean 的配置方式（返回 `SecurityFilterChain` Bean）。

# 2. 面试题

## 2.1 Spring Security 的认证流程是怎样的？
1. 用户提交用户名和密码，`UsernamePasswordAuthenticationFilter` 拦截 `/login` 的 POST 请求。
2. 将用户名和密码封装为 `UsernamePasswordAuthenticationToken`（此时 `authenticated=false`，表示未认证）。
3. 调用 `AuthenticationManager.authenticate(token)`。
4. `ProviderManager`（AuthenticationManager 实现）遍历所有的 `AuthenticationProvider`，找到支持该类型的 Provider。
5. `DaoAuthenticationProvider` 调用 `UserDetailsService.loadUserByUsername(username)` 获取 `UserDetails`。
6. 通过 `PasswordEncoder.matches(rawPassword, encodedPassword)` 验证密码。
7. 认证成功后，返回新的已认证的 `Authentication`（`authenticated=true`），包含 `UserDetails` 和权限信息 `GrantedAuthority`。
8. `SecurityContextHolder.getContext().setAuthentication(authentication)` 将认证对象存入当前线程的安全上下文。
9. 后续的请求通过 `SecurityContextPersistenceFilter` 从 Session 恢复 SecurityContext（如有状态），或每次请求重新验证 Token（JWT 无状态）。

## 2.2 认证（Authentication）和授权（Authorization）有什么区别？
| 维度 | 认证 Authentication | 授权 Authorization |
|------|-------------------|-------------------|
| 含义 | 验证用户身份 | 验证用户权限 |
| 问题 | "你是谁?" | "你能做什么?" |
| 时机 | 先于授权 | 认证通过后 |
| 典型实现 | 用户名+密码、JWT、OAuth2 | RBAC 角色、ACL、URL 权限 |
| 核心组件 | AuthenticationManager、UserDetailsService | AccessDecisionManager、@PreAuthorize |

在 Spring Security 中，认证确认 `Authentication.isAuthenticated()==true`，授权则根据 `Authentication.getAuthorities()` 判断是否有权访问资源。

## 2.3 JWT 令牌的结构是什么？有哪些优势和劣势？
JWT 由三部分组成（用 `.` 分隔）：
```
Header.Payload.Signature
```

- **Header**（头部）：声明令牌类型和签名算法，如 `{"alg":"HS256","typ":"JWT"}`，Base64Url 编码。
- **Payload**（负载）：包含 Claims（声明），如 `iss`（签发者）、`sub`（主题/用户ID）、`exp`（过期时间）、`iat`（签发时间），Base64Url 编码。
- **Signature**（签名）：`HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)` 防篡改。

**优势**：
- 无状态，服务端无需存储 Session，适合分布式/微服务和水平扩展。
- JSON 格式通用，跨语言支持好。
- 可在 Payload 中包含非敏感用户信息减少数据库查询。
**劣势**：
- 无法主动失效（除非引入黑名单/Redis 缓存）。
- Payload 仅 Base64 编码，非加密，不应存放敏感信息。
- 每次请求都需携带 Token，请求体积增加。
- Token 泄露后有效期内的风险无法避免（只能缩短有效期）。

## 2.4 如何实现 AccessToken + RefreshToken 双 Token 机制？
**机制设计**：
- AccessToken 短期有效（5-15 分钟），直接访问 API，请求负载小。
- RefreshToken 长期有效（7-30 天），仅用于刷新 AccessToken，存储在数据库 Redis 中可撤销。

**流程**：
1. 用户登录成功，返回 AccessToken（15分钟）和 RefreshToken（7天，存放在 HTTP-Only Cookie 或安全存储）。
2. 客户端每次请求携带 AccessToken。
3. 当 AccessToken 过期，客户端用 RefreshToken 调用 `/refresh` 接口。
4. 服务端验证 RefreshToken（是否在数据库中、是否被撤销、是否过期）。
5. 验证通过后，生成新的 AccessToken（和可选的新的 RefreshToken，Refresh Token Rotation 机制增强安全）。
6. 如果 RefreshToken 泄露，用户/管理员可在数据库中撤销，使下次刷新失败。

**Refresh Token Rotation**：每次刷新 AccessToken 时也更新 RefreshToken，旧的 RefreshToken 立即失效（防止被盗用的 RefreshToken 被持续使用）。

## 2.5 OAuth2 的授权码模式（Authorization Code）是如何工作的？
这是最安全、最常用的 OAuth2 授权模式：

1. **发起授权**：客户端将用户重定向到授权服务器：`GET /authorize?response_type=code&client_id=xxx&redirect_uri=xxx&scope=xxx&state=xxx`。
2. **用户授权**：授权服务器展示授权页面，用户输入凭据并同意授权。
3. **回调返回授权码**：授权服务器重定向到 `redirect_uri?code=AUTHORIZATION_CODE&state=xxx`。
4. **换取 Token**：客户端后端用授权码向授权服务器的 `/token` 端点发起 POST 请求：
```
POST /token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=xxx
```
5. **返回 Token**：授权服务器验证后返回 `{access_token, token_type, expires_in, refresh_token}`。

**为什么安全**：授权码通过前端（浏览器）传递但短暂使用，Token 通过后端 HTTPS 直接交换（不经过浏览器），且需要 `client_secret` 验证。

## 2.6 CSRF 是什么？为什么前后端分离 + JWT 可以关闭 CSRF？
**CSRF（Cross-Site Request Forgery）**：攻击者诱导用户点击恶意链接/访问恶意页面，利用用户浏览器中已保存的 Cookie（自动携带到请求）发送伪造请求。

**场景**：用户登录了 `bank.com`，Cookie 中有 SessionID；同时访问了恶意网站 `evil.com`；evil.com 中的脚本向 `bank.com/transfer?to=hacker&amount=10000` 发送请求，浏览器会自动携带 SessionID Cookie。

**为什么 JWT 可以关闭 CSRF**：
- JWT 通常通过 `Authorization: Bearer <token>` 头传递，不是通过 Cookie。
- 浏览器的 Cookie 自动携带机制不适用于自定义 Header。
- 第三方网站无法设置 `Authorization` 头（跨域请求需 CORS 预检）。
- 配置 `csrf().disable()` 的前提是确认完全使用 Token（非 Cookie）鉴权。

## 2.7 Spring Security 中有哪些核心 Filter？它们的作用是什么？
| Filter | 作用 |
|--------|------|
| `SecurityContextPersistenceFilter` | 请求开始时从 Session 恢复 SecurityContext，结束时清除（防止内存泄露） |
| `UsernamePasswordAuthenticationFilter` | 处理表单登录（默认 `/login` POST），执行用户名密码认证 |
| `BasicAuthenticationFilter` | 处理 HTTP Basic 认证（请求头 `Authorization: Basic base64`） |
| `ExceptionTranslationFilter` | 将 `AuthenticationException`（认证异常）和 `AccessDeniedException`（授权异常）转为 HTTP 响应 |
| `FilterSecurityInterceptor` | 方法安全授权的决策点，调用 `AccessDecisionManager` 判断是否有权限 |
| `RememberMeAuthenticationFilter` | 如果用户未登录但有 RememberMe Token，自动完成认证 |
| `AnonymousAuthenticationFilter` | 为未认证的请求生成匿名用户（`anonymousUser`），统一后续访问决策 |
| `CsrfFilter` | CSRF 防护，验证表单中的 `_csrf` Token |
| `CorsFilter` | CORS 跨域处理 |
| `LogoutFilter` | 处理退出登录 |

## 2.8 `@PreAuthorize` 和 `@PostAuthorize` 的区别？怎么使用？
- **`@PreAuthorize`**：在方法执行**之前**进行权限校验，SpEL 表达式可访问方法的参数值。失败抛 `AccessDeniedException`。
- **`@PostAuthorize`**：在方法执行**之后**进行权限校验，SpEL 表达式可访问返回值。适用于需要根据方法结果判断的场景（如用户只能查看自己的数据）。

```java
// 只有 ADMIN 角色才能访问
@PreAuthorize("hasRole('ADMIN')")
public List<User> allUsers() { ... }

// 只有用户名匹配的才能访问（参数级权限）
@PreAuthorize("#username == authentication.principal.username")
public User getUser(String username) { ... }

// 只能删除自己的帖子
@PreAuthorize("@postService.getById(#postId).creatorId == authentication.principal.id")
public void deletePost(Long postId) { ... }

// 只能返回当前用户自己的订单
@PostAuthorize("returnObject.userId == authentication.principal.id")
public Order getOrder(Long orderId) { ... }
```

需要在配置类上添加 `@EnableGlobalMethodSecurity(prePostEnabled = true)`。

## 2.9 Spring Security 如何集成 JWT 实现无状态认证？
1. **自定义 JWT 认证过滤器**，继承 `OncePerRequestFilter`：
```java
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
            HttpServletResponse response, FilterChain chain) {
        String token = extractToken(request);
        if (token != null && jwtUtil.validate(token)) {
            String userId = jwtUtil.getUserId(token);
            // 从缓存/DB 加载用户信息
            UserDetails userDetails = userDetailsService.loadUserByUsername(userId);
            UsernamePasswordAuthenticationToken authToken = 
                new UsernamePasswordAuthenticationToken(userDetails, null, 
                    userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }
        chain.doFilter(request, response);
    }
}
```
2. **配置 SecurityFilterChain**：
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) {
    http.csrf().disable()
        .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .authorizeRequests()
        .antMatchers("/auth/**").permitAll()
        .anyRequest().authenticated()
        .and()
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

## 2.10 如何自定义短信验证码认证？
1. **自定义认证 Token**：`SmsAuthenticationToken` 继承 `AbstractAuthenticationToken`。
2. **自定义 AuthenticationProvider**：实现 `AuthenticationProvider.authenticate()`，根据手机号和验证码验证。
3. **自定义 Filter**：`SmsAuthFilter` 继承 `AbstractAuthenticationProcessingFilter`，从请求中提取手机号和验证码。
4. **配置**：在 `SecurityFilterChain` 中通过 `.addFilterBefore()` 或 `.addFilterAt()` 注册自定义 Filter。
5. **Provider 注册**：在 `AuthenticationManagerBuilder` 中注册自定义的 `AuthenticationProvider`。

## 2.11 Spring Security 的 RBAC 权限模型如何设计？
RBAC（Role-Based Access Control）权限模型三层结构：

```
用户(User) ──多对多──> 角色(Role) ──多对多──> 权限(Permission)
```

**数据库设计**：
- `user` 表：用户信息。
- `role` 表：角色（ADMIN、EDITOR、VIEWER）。
- `permission` 表：权限（user:read、user:write、order:delete）。
- `user_role` 表：用户-角色关联。
- `role_permission` 表：角色-权限关联。

**权限字符串规范**：`资源:操作` 模式，如 `sys:user:add`、`order:export`。

**实现**：
- `UserDetails.getAuthorities()` 返回权限集合，每个权限作为一个 `GrantedAuthority`。
- `@PreAuthorize("hasAuthority('sys:user:delete')")` 判断具体权限。
- `@PreAuthorize("hasRole('ADMIN')")` 判断角色（自动添加 `ROLE_` 前缀）。

## 2.12 PasswordEncoder 中 BCrypt 的工作原理是什么？
BCrypt 是一种基于 Blowfish 加密算法的密码哈希函数，特点：

1. **自带 Salt**：每次哈希都生成随机 salt，避免彩虹表攻击，因此同一密码两次哈希结果不同。
2. **可调节强度**：通过 `log rounds` 参数（如 `BCryptPasswordEncoder(12)`），复杂度随轮数指数级增长，11 基本够用，越高越慢。
3. **哈希格式**：`$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy`，`$2a` 表示算法版本，`$10` 表示强度轮数，后面 22 位是 salt，余下是哈希结果。
4. **验证不需要 salt 参数**：因为 salt 已包含在哈希字符串中，`matches(rawPassword, encodedPassword)` 会自动从 encodedPassword 提取 salt 做比对。

## 2.13 Spring Security 中如何处理跨域请求？
```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http.cors().configurationSource(corsConfigurationSource());
        return http.build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(Collections.singletonList("*"));  // 建议用 pattern
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(Arrays.asList("*"));
        config.setAllowCredentials(true);  // 允许携带 Cookie
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

**注意**：`allowCredentials(true)` 情况下不能使用 `allowedOrigins("*")`（通配符），应使用 `allowedOriginPatterns("*")` 或指定具体的 `allowedOrigins`。

## 2.14 Spring Security 中的 Session 管理策略有哪些？无状态认证如何配置？
| 策略 | 说明 |
|------|------|
| `SessionCreationPolicy.ALWAYS` | 总是创建 HttpSession |
| `SessionCreationPolicy.IF_REQUIRED` | 需要时才创建（默认） |
| `SessionCreationPolicy.NEVER` | Spring Security 不创建，但若已存在则使用 |
| `SessionCreationPolicy.STATELESS` | 完全不使用 Session，每个请求独立认证 |

**JWT 无状态配置**：
```java
http.sessionManagement()
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

**Session 并发控制**：
```java
http.sessionManagement()
    .maximumSessions(1)                            // 同一账号最多 1 个会话
    .maxSessionsPreventsLogin(false)               // false: 后登录踢掉前登录; true: 禁止后登录
    .expiredSessionStrategy(event -> { ... });     // 会话过期处理
```

## 2.15 OAuth2 Resource Server 如何验证 JWT Token？
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://auth-server:9000
          jwk-set-uri: http://auth-server:9000/oauth2/jwks
```

```java
@Configuration
public class ResourceServerConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http.oauth2ResourceServer(resourceServer -> 
            resourceServer.jwt(jwt -> 
                jwt.jwtAuthenticationConverter(jwtAuthConverter())
            )
        );
        return http.build();
    }
}
```

Resource Server 使用 `JwtAuthenticationProvider` 验证 JWT 的签名（通过 JWK Set）、有效期、issuer 等。验证通过后从 JWT 中提取声明（Claims）映射为 `GrantedAuthority`。

## 2.16 Spring Security 中如何实现 Remember Me 功能？
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) {
    http.rememberMe()
        .key("unique-and-secret-key")             // 加密 RememberMe Token 的密钥
        .tokenValiditySeconds(1209600)             // Token 有效期（两周 = 14天）
        .rememberMeParameter("remember-me")        // 前端 CheckBox 的 name
        .rememberMeCookieName("remember-me");      // Cookie 名称
    return http.build();
}
```

**安全建议**：
- 使用 `PersistentTokenBasedRememberMeServices`（将 Token 持久化到数据库 `persistent_logins` 表），比默认的简单签名方式更安全。
- 不要使用固定的 Key（启动时随机生成），泄露后所有 RememberMe Token 可被伪造。

## 2.17 如何实现动态权限（URL 权限从数据库加载）？
1. **启动时加载权限**：从数据库查出所有 URL 和角色/权限的映射关系。
2. **自定义 FilterSecurityInterceptor 的 SecurityMetadataSource**：
```java
@Component
public class DynamicSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {
    @Override
    public Collection<ConfigAttribute> getAttributes(Object object) {
        String requestUrl = ((FilterInvocation) object).getRequestUrl();
        // 从缓存的权限映射中查询该 URL 所需的角色
        List<String> roles = permissionCache.getRolesByUrl(requestUrl);
        return SecurityConfig.createList(roles.toArray(new String[0]));
    }
}
```
3. **自定义 AccessDecisionManager**：判断当前用户是否拥有所需角色，支持通配符匹配（如 `/admin/**`）。
4. 权限规则变更后刷新缓存，无需重启应用。

## 2.18 Spring Security 如何处理认证失败和授权失败的异常？
在 `SecurityFilterChain` 中通过 `http.exceptionHandling()` 配置：
```java
http.exceptionHandling()
    .authenticationEntryPoint((request, response, authException) -> {
        // 认证失败（未登录或 Token 无效）→ 401
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(401);
        response.getWriter().write("{\"code\":401,\"message\":\"未登录或 Token 已过期\"}");
    })
    .accessDeniedHandler((request, response, accessDeniedException) -> {
        // 授权失败（有登录但权限不足）→ 403
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(403);
        response.getWriter().write("{\"code\":403,\"message\":\"没有操作权限\"}");
    });
```

`AuthenticationEntryPoint` 处理 `AuthenticationException`（认证异常），`AccessDeniedHandler` 处理 `AccessDeniedException`（授权异常）。

## 2.19 Spring Authorization Server 和旧的 Spring Security OAuth2 有什么区别？
| 维度 | Spring Security OAuth2（已废弃） | Spring Authorization Server |
|------|--------------------------------|---------------------------|
| 状态 | 已废弃，只维护到 2.5.x | 官方推荐 |
| 实现方式 | `@EnableAuthorizationServer` 注解一键开启 | 基于 SecurityFilterChain 显式配置 |
| OAuth2 版本 | OAuth 2.0 | OAuth 2.1 草案规范 |
| 令牌格式 | 支持 JWT 和 opaque | 支持 JWT 和 opaque |
| OpenID Connect | 需额外集成 | 内置 OIDC 1.0 支持 |
| 配置方式 | SecurityProperties 和配置器 | 链式 API 配置 |

迁移建议：新项目使用 Spring Authorization Server，旧项目有计划地迁移。

## 2.20 Spring Security 中 SecurityContext 如何在多线程/异步中传递？
默认 `SecurityContextHolder` 使用 `MODE_THREADLOCAL`，只能在同一线程内访问，`@Async` 异步方法拿不到。

**解决方案**：
1. **全局设置**：
```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```
2. **自定义 TaskExecutor 包装**：
```java
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setTaskDecorator(new DelegatingSecurityContextTaskDecorator());
    executor.initialize();
    return executor;
}
```
3. **显式传递**：
```java
SecurityContext context = SecurityContextHolder.getContext();
executor.submit(() -> {
    SecurityContextHolder.setContext(context);
    // 业务逻辑
});
```

推荐使用 `DelegatingSecurityContextAsyncTaskExecutor` 或 `DelegatingSecurityContextTaskDecorator` 包装线程池。

## 2.21 AuthenticationManager 和 AuthenticationProvider 链的工作原理？
**认证流程的底层核心组件**：

1. **AuthenticationManager**：认证管理器接口，默认实现为 `ProviderManager`。`ProviderManager` 维护一个 `List<AuthenticationProvider>` 列表。

2. **AuthenticationProvider**：具体的认证逻辑实现。ProviderManager 按注册顺序遍历所有 AuthenticationProvider：
   - 调用 `provider.supports(authenticationClass)` 判断是否支持当前认证类型（如 `UsernamePasswordAuthenticationToken.class`、`SmsCodeAuthenticationToken.class`）。
   - 如果支持，调用 `provider.authenticate(authentication)` 执行认证：
     - `DaoAuthenticationProvider`：最常见，通过 `UserDetailsService.loadUserByUsername()` 加载用户信息，再用 `PasswordEncoder.matches()` 比较密码。
   - 如果认证成功（返回带权限信息的 Authentication）→ 停止遍历，将认证结果存入 SecurityContext。
   - 如果认证失败（抛出 AuthenticationException）→ 继续下一个 Provider 尝试。
   - 遍历完所有 Provider 且全部失败 → 抛出最后的认证异常。

3. **自定义认证链路**：实现自定义 `AuthenticationProvider`，注入到 ProviderManager 即可添加新认证方式（如短信验证码认证、指纹认证）。

```java
@Component
public class SmsCodeAuthenticationProvider implements AuthenticationProvider {
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String phone = (String) authentication.getPrincipal();
        // 验证短信验证码
        // 加载或创建用户
        return new SmsCodeAuthenticationToken(user, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return SmsCodeAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

## 2.22 如何实现多种登录方式共存（密码登录 + 短信登录 + OAuth2 社交登录）？
在 `SecurityFilterChain` 中配置多个认证入口：

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        // 表单登录（密码登录）
        .formLogin(login -> login
            .loginProcessingUrl("/api/login/password")
            .successHandler(jsonAuthSuccessHandler)
            .failureHandler(jsonAuthFailureHandler))
        // OAuth2 社交登录（微信/GitHub）
        .oauth2Login(oauth -> oauth
            .loginProcessingUrl("/api/login/oauth2/code/*")
            .successHandler(oauthSuccessHandler))
        // 自定义短信登录（需自定义 Filter + Provider）
        .addFilterBefore(smsCodeAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
        .oauth2ResourceServer(resource -> resource.jwt())  // JWT 无状态认证
        .build();
}
```

**三种认证方式同时工作的关键**：
1. 每种登录方式对应各自的**认证 Filter**（`UsernamePasswordAuthenticationFilter`、`OAuth2LoginAuthenticationFilter`、自定义 `SmsCodeFilter`）。
2. 每个 Filter 创建不同类型的 Authentication Token → 匹配各自的 AuthenticationProvider。
3. 所有 Filter 在 SecurityFilterChain 中按顺序执行，一旦某个 Filter 认证成功（设置 SecurityContext），后续 Filter 直接放行。
4. 多个 Filter 之间互不干扰，由 Spring Security 框架自动管理认证状态。

## 2.23 AuthorizationManager 是什么？和旧的 AccessDecisionManager 有什么区别？
从 **Spring Security 5.8** 开始，`AuthorizationManager<T>` 替代了 `AccessDecisionManager` 和 `AccessDecisionVoter`，成为授权决策的新式 API。

**对比**：
| 维度 | AccessDecisionManager（旧） | AuthorizationManager（新） |
|------|---------------------------|--------------------------|
| 决策模型 | Manager + Voter 多投票制（赞成/反对/弃权） | 单一决策方法 `check()`，返回 `AuthorizationDecision`（grant/deny/abstain） |
| 接口 | 复杂，需要组合 Manager 和 Voter | 简洁，一个 `check()` 方法 |
| 函数式 | 不支持 | 支持 Lambda 表达式 |
| 注解授权 | 内部使用 Voter 机制 | 内部使用 AuthorizationManager |

**新 API 示例**：
```java
// 旧方式
.access("hasRole('ADMIN')")

// 新方式（AuthorizationManager Lambda）
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").access((authentication, requestContext) -> {
        boolean granted = requestContext.getRequest().isUserInRole("ADMIN");
        return new AuthorizationDecision(granted);
    })
    .anyRequest().authenticated()
);
```

迁移过程是渐进的：5.8+ 开始引入，6.0+ 推荐使用。`@PreAuthorize` 注解底下也开始使用 AuthorizationManager。这是 Spring Security 架构现代化的一部分，方便扩展和测试。
