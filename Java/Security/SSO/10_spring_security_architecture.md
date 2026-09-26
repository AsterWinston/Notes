
# 1. 概述

Spring Security 是 Java 生态下最强大的安全框架，提供认证（Authentication）、授权（Authorization）和攻击防护（CSRF/XSS/Session Fixation 等）能力。理解其架构是掌握 SSO 集成的关键前提。

```
┌──────────────────────────────────────────────────────────────────┐
│                Spring Security 核心架构                           │
│                                                                   │
│  请求 → FilterChainProxy → SecurityFilterChain                   │
│                                  │                                │
│               ┌──────────────────┼──────────────────┐            │
│               ▼                  ▼                  ▼            │
│         SecurityContext     Authentication      Access          │
│         PersistenceFilter   Filters            Decision          │
│               │                  │              Manager           │
│               ▼                  ▼                  │            │
│         SecurityContext    Authentication          ▼             │
│         Holder             Manager           FilterSecurity      │
│               │                  │              Interceptor       │
│               ▼                  ▼                               │
│         ThreadLocal<       Authentication                       │
│         SecurityContext>   Provider(s)                          │
│                             │       │                           │
│                             ▼       ▼                           │
│                     UserDetailsService                          │
│                     PasswordEncoder                             │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 核心组件

## 2.1 SecurityContextHolder

```java
// SecurityContextHolder 是 Spring Security 的核心存储
// 使用 ThreadLocal 存储当前请求的安全上下文

// 在任何地方获取当前认证用户：
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

if (authentication != null && authentication.isAuthenticated()) {
    String username = authentication.getName();
    Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
    Object principal = authentication.getPrincipal();      // UserDetails 实例
    Object credentials = authentication.getCredentials();  // 通常是密码（认证后清除）
    Object details = authentication.getDetails();          // WebAuthenticationDetails
}

// ThreadLocal 存储模式支持三种策略：
// MODE_THREADLOCAL（默认）：每个线程独立
// MODE_INHERITABLETHREADLOCAL：子线程继承父线程
// MODE_GLOBAL：全局共享（不推荐）
```

## 2.2 Authentication

```java
// Authentication 接口 —— 认证的核心抽象
public interface Authentication extends Principal, Serializable {

    // 用户的权限集合
    Collection<? extends GrantedAuthority> getAuthorities();

    // 凭证（通常是密码，认证通过后会被清除）
    Object getCredentials();

    // 认证的附加信息（IP、证书序列号等）
    Object getDetails();

    // 用户主体（认证前是用户名 String，认证后是 UserDetails）
    Object getPrincipal();

    // 是否已认证
    boolean isAuthenticated();

    // 设置认证状态
    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}

// 认证过程的状态变化：
// 1. 未认证状态：
//    principal = "zhangsan"（用户名）
//    credentials = "password123"（密码）
//    authenticated = false
//    authorities = []
//
// 2. 已认证状态：
//    principal = UserDetails 实例
//    credentials = null（安全起见，清除了密码）
//    authenticated = true
//    authorities = [ROLE_USER, ROLE_ADMIN]
```

## 2.3 AuthenticationManager

```java
// AuthenticationManager 是认证的入口
// 通常只有一个实现：ProviderManager

public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;
}

// ProviderManager 管理多个 AuthenticationProvider
public class ProviderManager implements AuthenticationManager {

    private List<AuthenticationProvider> providers;

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        for (AuthenticationProvider provider : providers) {
            // 遍历 providers，找到支持该认证类型的提供商
            if (!provider.supports(authentication.getClass())) {
                continue;
            }

            try {
                // 尝试认证
                Authentication result = provider.authenticate(authentication);
                if (result != null) {
                    // 复制 details
                    copyDetails(authentication, result);
                    return result;
                }
            } catch (AccountStatusException | InternalAuthenticationServiceException e) {
                throw e;  // 账户状态异常（锁定、过期等），直接抛出
            } catch (AuthenticationException e) {
                // 记录最后一个异常，继续尝试下一个 provider
                lastException = e;
            }
        }

        // 所有 provider 都尝试失败
        throw lastException != null ? lastException
            : new ProviderNotFoundException("No provider supports authentication");
    }
}
```

## 2.4 AuthenticationProvider

```java
// 各种认证方式的提供者：

// 1. DaoAuthenticationProvider（用户名密码认证）
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    private PasswordEncoder passwordEncoder;
    private UserDetailsService userDetailsService;

    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) {

        String presentedPassword = authentication.getCredentials().toString();
        if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            throw new BadCredentialsException("密码错误");
        }
    }
}

// 2. JwtAuthenticationProvider（JWT 令牌认证）
// 3. LdapAuthenticationProvider（LDAP 认证）
// 4. RememberMeAuthenticationProvider（记住我认证）
// 5. PreAuthenticatedAuthenticationProvider（预认证，如 SSO 代理）
// 6. CasAuthenticationProvider（CAS 认证）
// 7. OAuth2LoginAuthenticationProvider（OAuth2 登录认证）
// 8. OidcAuthorizationCodeAuthenticationProvider（OIDC 认证）
```

## 2.5 UserDetailsService

```java
// UserDetailsService —— 加载用户数据的核心接口
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}

// 典型实现：
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "用户不存在: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())          // BCrypt 加密后的密码
            .disabled(!user.isEnabled())            // 是否禁用
            .accountExpired(!user.isAccountNonExpired())  // 是否过期
            .accountLocked(!user.isAccountNonLocked())    // 是否锁定
            .credentialsExpired(!user.isCredentialsNonExpired()) // 凭证是否过期
            .authorities(user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .toList())
            .build();
    }
}
```

# 3. SecurityFilterChain

## 3.1 过滤器链结构

```
HTTP 请求
    │
    ▼
┌──────────────────────────────┐
│  ChannelProcessingFilter     │ ← HTTP → HTTPS 重定向
├──────────────────────────────┤
│  WebAsyncManagerIntegration  │ ← 异步请求支持
├──────────────────────────────┤
│  SecurityContextPersistence  │ ← 从 Session 恢复 SecurityContext
├──────────────────────────────┤
│  HeaderWriterFilter         │ ← 添加安全响应头
├──────────────────────────────┤
│  CorsFilter                 │ ← 跨域处理
├──────────────────────────────┤
│  CsrfFilter                 │ ← CSRF 防护
├──────────────────────────────┤
│  LogoutFilter               │ ← 处理登出
├──────────────────────────────┤
│  UsernamePasswordAuthFilter │ ← 表单登录认证
│  OAuth2LoginAuthFilter      │ ← OAuth2 登录
│  CasAuthenticationFilter    │ ← CAS 认证
│  BearerTokenAuthFilter      │ ← Bearer Token 认证
│  ... (各种认证过滤器)        │
├──────────────────────────────┤
│  RequestCacheAwareFilter    │ ← 缓存原始请求（认证后恢复）
├──────────────────────────────┤
│  SecurityContextHolderAware  │ ← 包装请求（servlet API）
├──────────────────────────────┤
│  RememberMeAuthentication   │ ← 记住我
├──────────────────────────────┤
│  AnonymousAuthentication    │ ← 匿名用户
├──────────────────────────────┤
│  ExceptionTranslationFilter │ ← 异常处理（403/302等）
├──────────────────────────────┤
│  FilterSecurityInterceptor  │ ← 授权决策（最后一道防线！）
└──────────────────────────────┘
    │
    ▼
业务 Controller
```

## 3.2 FilterChainProxy 与多重过滤器链

```java
// Spring Security 支持多个 SecurityFilterChain
// 每个 Chain 有特定的匹配规则

@Configuration
@EnableWebSecurity
public class MultiChainConfig {

    @Bean
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }

    @Bean
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/web/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED));
        return http.build();
    }

    // FilterChainProxy 按顺序匹配，第一个匹配的 Chain 处理请求
    // /api/** → apiFilterChain（无状态 JWT）
    // /web/** → webFilterChain（表单登录 + Session）
}
```

# 4. 认证流程完整剖析

## 4.1 表单登录完整链路

```java
/**
 * 表单登录的完整调用链路：
 *
 * 1. UsernamePasswordAuthenticationFilter.attemptAuthentication(request, response)
 *    从请求中提取 username 和 password
 *    创建 UsernamePasswordAuthenticationToken（未认证）
 *
 * 2. AuthenticationManager.authenticate(token)
 *    → ProviderManager.authenticate(token)
 *
 * 3. DaoAuthenticationProvider.authenticate(authentication)
 *    ├── 3.1 retrieveUser(username) → UserDetailsService.loadUserByUsername()
 *    │       从数据库/缓存/LDAP 加载用户信息
 *    ├── 3.2 preAuthenticationChecks(userDetails)
 *    │       检查账户是否锁定/过期/禁用
 *    ├── 3.3 additionalAuthenticationChecks(userDetails, authentication)
 *    │       PasswordEncoder.matches(输入的密码, 存储的密码)
 *    └── 3.4 createSuccessAuthentication(userDetails, authentication)
 *            创建已认证的 Authentication 对象（credentials=null）
 *
 * 4. 认证成功回调 → AbstractAuthenticationProcessingFilter.successfulAuthentication()
 *    ├── SecurityContextHolder.getContext().setAuthentication(authResult)
 *    ├── 调用 RememberMeServices（如需要）
 *    ├── 发布 AuthenticationSuccessEvent
 *    └── 调用 AuthenticationSuccessHandler
 *        ├── SavedRequestAwareAuthenticationSuccessHandler
 *        │   → 重定向到认证前访问的 URL
 *        └── 或自定义 Handler（返回 JSON、重定向等）
 *
 * 5. Session 管理
 *    ├── SessionFixationProtectionStrategy → 更换 SessionID
 *    └── 存储 SecurityContext 到 HttpSession
 *
 * 后续请求：
 * 6. SecurityContextPersistenceFilter 从 Session 恢复 SecurityContext
 *    → 反序列化 Authentication 对象
 *    → 设置到 SecurityContextHolder
 */

// ========== 自定义认证成功 Handler ==========
@Component
public class CustomAuthenticationSuccessHandler
        implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException {

        // 判断是否 AJAX 请求
        if ("XMLHttpRequest".equals(request.getHeader("X-Requested-With"))) {
            response.setContentType("application/json;charset=UTF-8");
            UserDetails user = (UserDetails) authentication.getPrincipal();
            String json = String.format(
                "{\"code\":0,\"msg\":\"登录成功\",\"data\":{\"username\":\"%s\"}}",
                user.getUsername());
            response.getWriter().write(json);
        } else {
            // 传统 Web 应用，重定向
            response.sendRedirect("/index");
        }
    }
}
```

## 4.2 认证失败处理

```java
@Component
public class CustomAuthenticationFailureHandler
        implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException exception)
            throws IOException {

        response.setContentType("application/json;charset=UTF-8");

        String errorMsg;
        if (exception instanceof BadCredentialsException) {
            errorMsg = "用户名或密码错误";
        } else if (exception instanceof LockedException) {
            errorMsg = "账户已被锁定";
        } else if (exception instanceof DisabledException) {
            errorMsg = "账户已被禁用";
        } else if (exception instanceof AccountExpiredException) {
            errorMsg = "账户已过期";
        } else if (exception instanceof CredentialsExpiredException) {
            errorMsg = "密码已过期";
        } else if (exception instanceof SessionAuthenticationException) {
            errorMsg = "会话并发限制，已在其他设备登录";
        } else {
            errorMsg = "认证失败: " + exception.getMessage();
        }

        response.getWriter().write(
            String.format("{\"code\":1,\"msg\":\"%s\"}", errorMsg));
    }
}
```

# 5. 授权模型

## 5.1 基于角色的访问控制（RBAC）

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // 启用方法级别安全
public class AuthorizationConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // URL 级别授权
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").hasAuthority("READ")
                .requestMatchers(HttpMethod.POST, "/api/products/**").hasAuthority("WRITE")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}

// 方法级别授权
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public List<User> getAllUsers() {
        // 只有 ADMIN 可以调用
    }

    @PreAuthorize("hasRole('USER') and #username == authentication.name")
    public User getUser(String username) {
        // 用户只能查看自己的信息
    }

    @PostAuthorize("returnObject.owner == authentication.name")
    public Document getDocument(Long id) {
        // 只有文档所有者可以获取
    }

    @PreFilter("filterObject.owner == authentication.name")
    public List<Document> filterDocuments(List<Document> docs) {
        // 过滤掉不属于当前用户的数据
    }

    @PostFilter("filterObject.visible == true")
    public List<Document> getAllDocuments() {
        // 返回结果中过滤掉不可见的文档
    }
}
```

## 5.2 ACL（访问控制列表）

```java
// Spring Security ACL —— 对领域对象的细粒度权限控制
// 场景：文档 A 对用户 zhangsan 可读可写，对 lisi 只读

// 数据库表结构：
// acl_sid: 安全标识（用户/角色）
// acl_class: 受保护的类（如 com.example.Document）
// acl_object_identity: 受保护的对象实例
// acl_entry: 权限条目（针对特定 SID 的特定对象的权限掩码）

@Service
public class DocumentService {

    @Autowired
    private MutableAclService aclService;

    /**
     * 为文档添加访问权限
     */
    public void grantPermission(Document document, String username,
                                 Permission permission) {
        // 创建或获取 ObjectIdentity
        ObjectIdentity oid = new ObjectIdentityImpl(Document.class, document.getId());

        // 创建新的 ACL 或获取现有 ACL
        MutableAcl acl;
        try {
            acl = (MutableAcl) aclService.readAclById(oid);
        } catch (NotFoundException e) {
            acl = aclService.createAcl(oid);
        }

        // 添加权限条目
        Sid sid = new PrincipalSid(username);
        acl.insertAce(acl.getEntries().size(), permission, sid, true);

        aclService.updateAcl(acl);
    }
}
```

# 6. Spring Security 配置模式

## 6.1 Lambda DSL（推荐，Spring Security 6+）

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. 认证配置
            .formLogin(form -> form
                .loginPage("/login")
                .loginProcessingUrl("/login")
                .successHandler(customSuccessHandler())
                .failureHandler(customFailureHandler())
                .permitAll()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                )
            )

            // 2. 授权配置
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // 3. Session 管理
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
            )

            // 4. 登出配置
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
                .deleteCookies("JSESSIONID", "remember-me")
            )

            // 5. CSRF 防护
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")  // 无状态 API 可关闭
            )

            // 6. 记住我
            .rememberMe(remember -> remember
                .key("unique-and-secret")
                .tokenValiditySeconds(86400)
            );

        return http.build();
    }
}
```

## 6.2 自定义配置最佳实践

```java
// application.yml 中的安全配置
spring:
  security:
    user:
      name: admin
      password: $2a$10$...
    oauth2:
      client:
        registration:
          google:
            client-id: xxx
            client-secret: yyy
            scope: openid,profile,email
          github:
            client-id: xxx
            client-secret: yyy
        provider:
          google:
            issuer-uri: https://accounts.google.com
```

# 7. Spring Security 在 SSO 中的角色

```
SSO 场景中 Spring Security 承担的角色：

┌───────────────────────────────────────────────────────────┐
│                    认证层                                  │
│                                                           │
│  作为 CAS Client:   负责 CAS ST 验证 + 创建本地 Session   │
│  作为 OAuth2 Client: 负责发起授权 + 换取 Token            │
│  作为 OIDC RP:       负责 OIDC 认证 + ID Token 验证       │
│  作为 SAML SP:       负责 SAML 断言验证 + Create Session  │
│                                                           │
│                    授权层                                  │
│                                                           │
│  RBAC:  根据角色控制 URL 和方法的访问                     │
│  Scope: 根据 OAuth2 Scope 控制 API 访问                   │
│  ACL:   细粒度领域对象权限                                │
│                                                           │
│                    防护层                                  │
│                                                           │
│  CSRF:  防护跨站请求伪造                                  │
│  XSS:   通过 HttpOnly Cookie 和 Content-Security-Policy   │
│  Session: 会话固定/并发控制/超时管理                      │
└───────────────────────────────────────────────────────────┘
```

# 8. 总结

| 组件 | 核心职责 |
|------|---------|
| SecurityContextHolder | 存储当前请求的认证信息（ThreadLocal） |
| Authentication | 用户身份和权限的抽象 |
| AuthenticationManager | 认证的顶层入口 |
| AuthenticationProvider | 具体的认证逻辑（密码/JWT/CAS/OAuth2） |
| UserDetailsService | 加载用户详细信息 |
| PasswordEncoder | 密码加密和验证 |
| SecurityFilterChain | 过滤器链，处理认证和授权 |
| FilterSecurityInterceptor | 最终的授权决策点 |
