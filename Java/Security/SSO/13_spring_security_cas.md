
# 1. 概述

Spring Security 提供 `spring-security-cas` 模块，使 Java Web 应用可以无缝集成 CAS Server 作为 SSO 客户端。它提供了 CAS 专用的认证过滤器、Ticket 验证器和用户信息映射。

```
┌──────────────────────────────────────────────────────────────────┐
│              Spring Security CAS 集成架构                         │
│                                                                   │
│  浏览器                    Spring Security (CAS Client)           │
│  ┌──────┐              ┌──────────────────────────────┐          │
│  │ TGC  │─────────────►│ CasAuthenticationFilter      │          │
│  └──────┘              │ ├─ 检测 ST 参数              │          │
│      │                 │ └─ 创建 CasAuthenticationToken│          │
│      │                 │                              │          │
│      │                 │ CasAuthenticationProvider    │          │
│      │                 │ ├─ TicketValidator 验证 ST   │          │
│      │                 │ └─ 创建认证Authentication    │          │
│      │                 │                              │          │
│      │                 │ ServiceProperties            │          │
│      │                 │ ├─ casServerUrlPrefix        │          │
│      │                 │ └─ service (回调地址)        │          │
│      │                 │                              │          │
│      │                 │ SingleSignOutFilter          │          │
│      │                 │ └─ 接收 CAS 的登出通知       │          │
│      │                 └──────────────────────────────┘          │
│      │                                   │                        │
│      │                    CAS 验证请求    │                        │
│      └───────────────────────────────────┼─────────────────────► │
│                                          │           CAS Server   │
│                                          │◄────────────────────── │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 依赖与基础配置

## 2.1 Gradle/Maven 依赖

```groovy
// build.gradle
dependencies {
    implementation 'org.springframework.security:spring-security-cas'
    implementation 'org.jasig.cas.client:cas-client-core:3.6.4'
}
```

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-cas</artifactId>
</dependency>
<dependency>
    <groupId>org.jasig.cas.client</groupId>
    <artifactId>cas-client-core</artifactId>
    <version>3.6.4</version>
</dependency>
```

## 2.2 基础 Java Config

```java
@Configuration
@EnableWebSecurity
public class CasSecurityConfig {

    @Value("${cas.server-url-prefix}")
    private String casServerUrlPrefix;

    @Value("${cas.server-login-url}")
    private String casServerLoginUrl;

    @Value("${cas.client-host-url}")
    private String clientHostUrl;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
            CasAuthenticationFilter casAuthenticationFilter) throws Exception {

        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )

            // 添加 CAS 认证过滤器
            .addFilter(casAuthenticationFilter)
            .addFilterBefore(casLogoutFilter(), LogoutFilter.class)
            .addFilterBefore(singleSignOutFilter(), CasAuthenticationFilter.class)

            // CAS 认证入口（未登录跳转 CAS）
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(casAuthenticationEntryPoint())
            )

            // 登出配置
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl(casServerUrlPrefix + "/logout")
            )

            // Session 管理
            .sessionManagement(session -> session
                .sessionFixation().migrateSession()
                .maximumSessions(1)
                .sessionRegistry(sessionRegistry())
            );

        return http.build();
    }

    // ========== CAS 核心 Bean 定义 ==========

    /**
     * CAS Service 配置
     */
    @Bean
    public ServiceProperties serviceProperties() {
        ServiceProperties serviceProperties = new ServiceProperties();
        serviceProperties.setService(clientHostUrl + "/login/cas");
        serviceProperties.setSendRenew(false);  // 不使用 renew 参数
        serviceProperties.setAuthenticateAllArtifacts(true);
        return serviceProperties;
    }

    /**
     * CAS 认证入口：未登录 → 重定向到 CAS 登录页
     */
    @Bean
    public CasAuthenticationEntryPoint casAuthenticationEntryPoint() {
        CasAuthenticationEntryPoint entryPoint = new CasAuthenticationEntryPoint();
        entryPoint.setLoginUrl(casServerLoginUrl);
        entryPoint.setServiceProperties(serviceProperties());
        return entryPoint;
    }

    /**
     * CAS Ticket 验证器
     */
    @Bean
    public Cas30ServiceTicketValidator casTicketValidator() {
        return new Cas30ServiceTicketValidator(casServerUrlPrefix);
        // 注意：默认使用 /p3/serviceValidate，与 CAS 3.0 协议兼容
    }

    /**
     * CAS 认证提供者
     */
    @Bean
    public CasAuthenticationProvider casAuthenticationProvider() {
        CasAuthenticationProvider provider = new CasAuthenticationProvider();
        provider.setAuthenticationUserDetailsService(casUserDetailsService());
        provider.setServiceProperties(serviceProperties());
        provider.setTicketValidator(casTicketValidator());
        provider.setKey("cas_auth_provider_key");
        return provider;
    }

    /**
     * CAS 认证过滤器
     */
    @Bean
    public CasAuthenticationFilter casAuthenticationFilter() throws Exception {
        CasAuthenticationFilter filter = new CasAuthenticationFilter();
        filter.setServiceProperties(serviceProperties());
        filter.setAuthenticationManager(authenticationManager());
        filter.setFilterProcessesUrl("/login/cas");  // 回调处理 URL
        filter.setAuthenticationSuccessHandler(casAuthenticationSuccessHandler());
        filter.setAuthenticationFailureHandler(casAuthenticationFailureHandler());
        return filter;
    }

    /**
     * AuthenticationManager
     */
    @Bean
    public AuthenticationManager authenticationManager() {
        return new ProviderManager(casAuthenticationProvider());
    }
}
```

# 3. CAS 用户信息加载

## 3.1 从 CAS Assertion 加载用户

```java
/**
 * CAS 用户信息服务
 * 从 CAS Server 返回的 Assertion 中提取用户信息
 */
@Component
public class CasUserDetailsService
        implements AuthenticationUserDetailsService<CasAssertionAuthenticationToken> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserDetails(CasAssertionAuthenticationToken token) {
        // 1. 获取 CAS Assertion
        Assertion assertion = token.getAssertion();

        // 2. 获取用户标识（Principal）
        String username = assertion.getPrincipal().getName();

        // 3. 获取用户属性（CAS 3.0+ 提供）
        Map<String, Object> attributes = assertion.getPrincipal().getAttributes();
        String email = (String) attributes.getOrDefault("email", username);
        String displayName = (String) attributes.getOrDefault("displayName", username);

        // 4. 在本地数据库中查找或创建用户
        User user = userRepository.findByUsername(username)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setUsername(username);
                newUser.setEmail(email);
                newUser.setDisplayName(displayName);
                newUser.setRole("ROLE_USER");
                return userRepository.save(newUser);
            });

        // 5. 同步 CAS 属性（如果用户在 CAS 端更新了信息）
        if (!email.equals(user.getEmail())) {
            user.setEmail(email);
            userRepository.save(user);
        }

        // 6. 返回 Spring Security UserDetails
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password("")  // CAS 用户本地无需密码
            .authorities(user.getRole().split(","))
            .disabled(!user.isEnabled())
            .build();
    }
}
```

## 3.2 从 LDAP 或其他源获取属性

```java
/**
 * 高级 CAS 配置：CAS + LDAP 组合
 * CAS Server 负责认证，通过 CAS 返回用户信息
 * 本地应用也可以通过 LDAP 补充更多属性
 */
@Component
public class AdvancedCasUserDetailsService
        implements AuthenticationUserDetailsService<CasAssertionAuthenticationToken> {

    @Override
    public UserDetails loadUserDetails(CasAssertionAuthenticationToken token) {
        String username = token.getAssertion().getPrincipal().getName();

        // 方式1：使用 CAS 返回的属性（推荐）
        Map<String, Object> attrs = token.getAssertion().getPrincipal().getAttributes();

        // 方式2：如果需要更多属性，CAS Server 6.x 支持
        // Attribute Release Policy 配置哪些属性返回给特定 Service

        // 方式3：本地再从 LDAP/DB 查询补充属性
        // List<String> roles = ldapService.getUserRoles(username);
        // String department = ldapService.getDepartment(username);

        List<GrantedAuthority> authorities = new ArrayList<>();
        // 从 CAS attributes 提取角色
        if (attrs.containsKey("roles")) {
            @SuppressWarnings("unchecked")
            List<String> roles = (List<String>) attrs.get("roles");
            roles.forEach(role ->
                authorities.add(new SimpleGrantedAuthority("ROLE_" + role)));
        }

        return new User(username, "", authorities);
    }
}
```

# 4. 单点登出（SLO）集成

## 4.1 SingleSignOutFilter 配置

```java
/**
 * CAS Single Sign-Out Filter
 * 接收 CAS Server 发送的登出请求，销毁本地 Session
 */
@Bean
public SingleSignOutFilter singleSignOutFilter() {
    SingleSignOutFilter filter = new SingleSignOutFilter();
    // 如果 CAS Client 是 3.6+，也可以使用更简单的方式
    return filter;
}

@Bean
public SecurityContextLogoutHandler securityContextLogoutHandler() {
    return new SecurityContextLogoutHandler();
}

/**
 * CAS 登出过滤器
 * 本地登出 + 全局登出（重定向到 CAS /logout）
 */
@Bean
public LogoutFilter casLogoutFilter() {
    String logoutRedirectUrl = casServerUrlPrefix
        + "/logout?service=" + URLEncoder.encode(clientHostUrl, StandardCharsets.UTF_8);

    LogoutFilter logoutFilter = new LogoutFilter(
        logoutRedirectUrl,
        securityContextLogoutHandler(),
        new CookieClearingLogoutHandler("JSESSIONID")
    );
    logoutFilter.setFilterProcessesUrl("/logout/cas");
    return logoutFilter;
}
```

## 4.2 CAS Server 端 SLO 配置

```json
// CAS Server 端的 Service 注册配置
// services/MyApp.json
{
  "@class": "org.apereo.cas.services.RegexRegisteredService",
  "serviceId": "^https://myapp\\.example\\.com/.*",
  "name": "My Application",
  "id": 1001,
  "logoutType": "BACK_CHANNEL",
  "logoutUrl": "https://myapp.example.com/logout/cas",
  "attributeReleasePolicy": {
    "@class": "org.apereo.cas.services.ReturnAllAttributeReleasePolicy"
  }
}
```

# 5. 自定义认证成功/失败处理

## 5.1 认证成功处理

```java
/**
 * CAS 认证成功处理器
 */
@Component
public class CasAuthenticationSuccessHandler
        extends SavedRequestAwareAuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException, ServletException {

        // 获取 CAS Assertion
        CasAssertionAuthenticationToken casToken =
            (CasAssertionAuthenticationToken) authentication;
        Assertion assertion = casToken.getAssertion();

        // 记录登录日志
        String username = authentication.getName();
        log.info("CAS SSO 登录成功: {}", username);

        // 如果是首次登录，可能需要在 Session 中设置额外信息
        Map<String, Object> attributes = assertion.getPrincipal().getAttributes();
        request.getSession().setAttribute("USER_ATTRS", attributes);

        // 判断是否 AJAX 请求
        if (isAjaxRequest(request)) {
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write(
                "{\"code\":0,\"message\":\"登录成功\",\"redirectUrl\":\"/home\"}");
            return;
        }

        // 传统 Web 应用：重定向到原始请求 URL
        super.onAuthenticationSuccess(request, response, authentication);
    }

    private boolean isAjaxRequest(HttpServletRequest request) {
        return "XMLHttpRequest".equals(request.getHeader("X-Requested-With"));
    }
}
```

## 5.2 认证失败处理

```java
/**
 * CAS 认证失败处理
 */
@Component
public class CasAuthenticationFailureHandler
        implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException exception)
            throws IOException {

        response.setContentType("application/json;charset=UTF-8");

        String message;
        if (exception instanceof BadCredentialsException) {
            message = "CAS 认证失败：票据无效";
        } else if (exception instanceof AuthenticationServiceException) {
            message = "CAS 服务异常，请稍后重试";
        } else {
            message = "SSO 认证失败: " + exception.getMessage();
        }

        log.error("CAS 认证失败: {}", message, exception);
        response.getWriter().write(
            String.format("{\"code\":1,\"message\":\"%s\"}", message));
    }
}
```

# 6. 高级特性

## 6.1 CAS Proxy 集成

```java
/**
 * CAS Proxy：后端代表用户请求其他 CAS 保护的服务
 */
@Configuration
public class CasProxyConfig {

    @Bean
    public Cas20ProxyTicketValidator cas20ProxyTicketValidator() {
        Cas20ProxyTicketValidator validator =
            new Cas20ProxyTicketValidator(casServerUrlPrefix);
        // 启用代理回调
        validator.setAcceptAnyProxy(true);
        // validator.setProxyCallbackUrl(clientHostUrl + "/proxy/callback");
        // validator.setProxyGrantingTicketStorage(proxyGrantingTicketStorage());
        return validator;
    }

    /**
     * 获取 Proxy Ticket，代表用户访问其他服务
     */
    public String getProxyTicket(String targetService) {
        // 从当前认证中获取 Assertion
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        CasAssertionAuthenticationToken casAuth =
            (CasAssertionAuthenticationToken) auth;

        // 获取 PGT（Proxy Granting Ticket）
        String pgtId = casAuth.getAssertion().getPrincipal().getProxyGrantingTicketId();

        // 用 PGT 换取 PT（Proxy Ticket）
        return retrieveProxyTicket(pgtId, targetService);
    }

    private String retrieveProxyTicket(String pgtId, String targetService) {
        try {
            return new URL(casServerUrlPrefix + "/proxy"
                + "?pgt=" + pgtId
                + "&targetService=" + URLEncoder.encode(targetService, "UTF-8"))
                .openStream()
                .toString();
        } catch (Exception e) {
            throw new RuntimeException("Failed to get proxy ticket", e);
        }
    }
}
```

## 6.2 多 CAS Server 支持

```java
/**
 * 场景：不同子域使用不同的 CAS Server
 */
@Configuration
public class MultiCasConfig {

    /**
     * 主 CAS Server（内部员工）
     */
    @Bean
    @Primary
    public CasAuthenticationProvider internalCasProvider() {
        CasAuthenticationProvider provider = new CasAuthenticationProvider();
        provider.setTicketValidator(
            new Cas30ServiceTicketValidator("https://cas.internal.example.com"));
        // ...
        return provider;
    }

    /**
     * 外部 CAS Server（合作伙伴）
     */
    @Bean
    public CasAuthenticationProvider externalCasProvider() {
        CasAuthenticationProvider provider = new CasAuthenticationProvider();
        provider.setTicketValidator(
            new Cas30ServiceTicketValidator("https://cas.partner.com"));
        // ...
        return provider;
    }
}
```

# 7. 完整配置文件示例

```yaml
# application.yml
app:
  cas:
    server-url-prefix: https://sso.example.com/cas
    server-login-url: https://sso.example.com/cas/login
    client-host-url: https://myapp.example.com

server:
  servlet:
    session:
      cookie:
        name: MYAPP_SESSION
        http-only: true
        secure: true
        same-site: lax
```

```properties
# cas.properties（CAS Client 配置）
cas.server-url-prefix=https://sso.example.com/cas
cas.server-login-url=https://sso.example.com/cas/login
cas.client-host-url=https://myapp.example.com
cas.validation-type=CAS3
```

# 8. 故障排查

| 问题 | 可能原因 | 解决 |
|------|---------|------|
| 无限重定向 | `service` URL 与 CAS Server 中注册的不一致 | 核对 Service Registration 配置 |
| "ticket not recognized" | ST 已过期或被使用多次 | 确保 CAS Client IP 与 CAS Server 时间同步 |
| CAS 返回 403 | Service 未在白名单中 | 在 CAS Server 中注册 Service |
| 单点登出不生效 | logoutUrl 不可达 / 配置错误 | 检查 SLO 配置和服务间网络连通性 |
| Session 过期太快 | Session 超时设置不当 | `server.servlet.session.timeout=30m` |
| 获取不到用户属性 | CAS 未配置属性释放策略 | CAS Server 配置 Attribute Release Policy |

# 9. 总结

| 组件 | 用途 |
|------|------|
| `ServiceProperties` | 配置 CAS Service（本应用回调地址） |
| `CasAuthenticationEntryPoint` | 未登录时 302 重定向到 CAS 登录页 |
| `CasAuthenticationFilter` | 处理 CAS 回调（拦截带 ticket 的请求） |
| `CasAuthenticationProvider` | 调用 CAS Server 验证 ST，创建认证 |
| `Cas30ServiceTicketValidator` | 具体的 ST 验证实现（CAS 3.0 协议） |
| `CasUserDetailsService` | 从 CAS Assertion 中提取用户信息 |
| `SingleSignOutFilter` | 接收 CAS Server 的 SLO 通知 |
| `CasProxyTicketValidator` | 支持 CAS Proxy 模式 |
