
# 1. 概述

Spring Security OAuth2 Client 提供完整的 OAuth 2.0 / OIDC 客户端支持，自动处理授权码流程、Token 管理和用户信息获取。是 Spring Boot 应用中实现"第三方登录"和 SSO 客户端的主要方式。

# 2. 快速入门

## 2.1 依赖与基础配置

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          # ===== OIDC 方式（Google） =====
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,profile,email

          # ===== 通用 OAuth2 方式（GitHub） =====
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user,user:email

          # ===== 自定义 SSO 服务器 =====
          my-sso:
            client-id: my-app
            client-secret: my-secret
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: openid,profile,email
            client-name: My SSO

        provider:
          # 自定义 SSO 服务器的 OIDC 配置
          my-sso:
            issuer-uri: https://sso.example.com
            # 或手动指定（非 OIDC 提供方）：
            # authorization-uri: https://sso.example.com/oauth2/authorize
            # token-uri: https://sso.example.com/oauth2/token
            # user-info-uri: https://sso.example.com/oauth2/userinfo
            # user-name-attribute: sub
```

## 2.2 最简单的配置

```java
@Configuration
@EnableWebSecurity
public class OAuth2ClientConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")  // 自定义登录页
                .defaultSuccessUrl("/home", true)
            );
        return http.build();
    }
}
```

```html
<!-- templates/login.html -->
<!DOCTYPE html>
<html>
<head><title>登录</title></head>
<body>
    <h2>选择登录方式</h2>

    <!-- Spring Security 自动生成的 OAuth2 登录链接格式：
         /oauth2/authorization/{registrationId} -->
    <a href="/oauth2/authorization/google">使用 Google 登录</a>
    <a href="/oauth2/authorization/github">使用 GitHub 登录</a>
    <a href="/oauth2/authorization/my-sso">使用 SSO 登录</a>
</body>
</html>
```

# 3. OAuth2 Client 核心机制

## 3.1 自动处理的流程

```
Spring Security OAuth2 Client 自动处理以下步骤：

┌───────────────────────────────────────────────────────────────┐
│ ① 用户点击登录链接                                             │
│    GET /oauth2/authorization/google                           │
│    → OAuth2AuthorizationRequestRedirectFilter 拦截             │
│    → 构建 Authorization Request                               │
│    → 302 重定向到 Google 授权页                                │
│                                                               │
│ ② 用户在授权服务器上完成认证授权                                │
│    → 302 重定向回 redirect_uri?code=xxx&state=yyy            │
│                                                               │
│ ③ 回调处理                                                     │
│    → OAuth2LoginAuthenticationFilter 拦截                     │
│    → OAuth2LoginAuthenticationProvider 处理：                  │
│       ├─ 用 code 换取 Access Token + ID Token                 │
│       ├─ 用 Access Token 获取 UserInfo                        │
│       └─ 创建 OAuth2User (Authentication)                     │
│                                                               │
│ ④ 建立本地会话                                                 │
│    → SecurityContextHolder 存入认证信息                        │
│    → HttpSession 存储 SecurityContext                          │
│    → Redirect 到原始请求页面                                   │
└───────────────────────────────────────────────────────────────┘
```

## 3.2 核心类体系

```
OAuth2AuthorizationRequestRedirectFilter     ← 发起授权请求
OAuth2LoginAuthenticationFilter             ← 处理回调（code → token）
OAuth2LoginAuthenticationProvider           ← 执行认证逻辑
    └─ OidcAuthorizationCodeAuthenticationProvider  ← OIDC 特化
OAuth2AuthorizedClientService               ← 管理授权后的 Client
    ├─ InMemoryOAuth2AuthorizedClientService       ← 内存存储
    └─ JdbcOAuth2AuthorizedClientService           ← 数据库存储
OAuth2UserService                           ← 获取用户信息
    ├─ DefaultOAuth2UserService                    ← 默认实现
    └─ OidcUserService                             ← OIDC 实现
```

# 4. 自定义 OAuth2User

## 4.1 OAuth2User 与 UserDetails 整合

```java
/**
 * 自定义 OAuth2UserService：处理第三方登录后的用户信息
 */
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) {
        // 1. 获取默认的 OAuth2UserService 加载用户信息
        OAuth2UserService<OAuth2UserRequest, OAuth2User> delegate =
            new DefaultOAuth2UserService();
        OAuth2User oauth2User = delegate.loadUser(userRequest);

        // 2. 获取 Provider 信息
        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttributeName = userRequest.getClientRegistration()
            .getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();

        // 3. 提取用户属性
        Map<String, Object> attributes = oauth2User.getAttributes();
        String email = (String) attributes.get("email");
        String name = (String) attributes.get("name");

        // 4. 在本地数据库中查找或创建用户
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setEmail(email);
                newUser.setName(name);
                newUser.setProvider(registrationId);
                newUser.setProviderId(oauth2User.getName()); // sub
                newUser.setRole("ROLE_USER");
                return userRepository.save(newUser);
            });

        // 5. 返回自定义的 OAuth2User
        return new CustomOAuth2User(user, attributes);
    }
}

/**
 * 自定义 OAuth2User：实现 OAuth2User 和 UserDetails 双接口
 */
public class CustomOAuth2User implements OAuth2User, UserDetails {

    private final User user;
    private final Map<String, Object> attributes;

    public CustomOAuth2User(User user, Map<String, Object> attributes) {
        this.user = user;
        this.attributes = attributes;
    }

    @Override
    public String getName() {
        return user.getEmail();
    }

    @Override
    public Map<String, Object> getAttributes() {
        return attributes;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority(user.getRole()));
    }

    @Override
    public String getPassword() {
        // OAuth2 用户没有本地密码
        return null;
    }

    @Override
    public String getUsername() {
        return user.getEmail();
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return true; }

    public User getUser() { return user; }
}

// 在 SecurityConfig 中配置
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private CustomOAuth2UserService customOAuth2UserService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)
                )
            );
        return http.build();
    }
}
```

# 5. OAuth2AuthorizedClient 管理

## 5.1 服务端到服务端调用

```java
/**
 * 场景：当前用户已通过 OAuth2 登录，后端需要代表用户调用第三方 API
 * 使用 OAuth2AuthorizedClientManager 自动管理 Token
 */

@Configuration
public class WebClientConfig {

    /**
     * 配置 OAuth2AuthorizedClientManager
     * 自动处理 Token 获取、缓存、刷新
     */
    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()          // 授权码模式
                .refreshToken()               // 支持 Refresh Token
                .clientCredentials()          // 客户端凭证模式
                .build();

        DefaultOAuth2AuthorizedClientManager authorizedClientManager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

        return authorizedClientManager;
    }

    /**
     * WebClient 自动携带 Bearer Token
     */
    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        // ServletOAuth2AuthorizedClientExchangeFilterFunction 自动
        // 为每个请求添加 Authorization: Bearer xxx 头
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
        oauth2.setDefaultOAuth2AuthorizedClient(true);
        oauth2.setDefaultClientRegistrationId("my-sso");

        return WebClient.builder()
            .filter(oauth2)
            .build();
    }
}

// 使用 WebClient 自动携带 Token
@Service
public class ApiService {

    @Autowired
    private WebClient webClient;

    public String callApi() {
        return webClient.get()
            .uri("https://api.example.com/data")
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}
```

## 5.2 数据库存储 OAuth2AuthorizedClient

```java
// 默认使用内存存储，重启后 Token 丢失
// 生产环境应使用数据库持久化

// 表结构（Spring Security 自动创建）：
CREATE TABLE oauth2_authorized_client (
    client_registration_id VARCHAR(100) NOT NULL,
    principal_name VARCHAR(200) NOT NULL,
    access_token_type VARCHAR(100) NOT NULL,
    access_token_value BLOB NOT NULL,
    access_token_issued_at TIMESTAMP NOT NULL,
    access_token_expires_at TIMESTAMP NOT NULL,
    access_token_scopes VARCHAR(1000) DEFAULT NULL,
    refresh_token_value BLOB DEFAULT NULL,
    refresh_token_issued_at TIMESTAMP DEFAULT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (client_registration_id, principal_name)
);

// application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          my-sso:
            client-id: xxx
            client-secret: yyy
            # ...
```

# 6. 自定义认证流程

## 6.1 自定义认证成功/失败处理

```java
@Component
public class OAuth2AuthenticationSuccessHandler
        extends SavedRequestAwareAuthenticationSuccessHandler {

    @Autowired
    private JwtService jwtService;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException, ServletException {

        CustomOAuth2User oauth2User = (CustomOAuth2User) authentication.getPrincipal();

        // 生成自己的 JWT
        String jwt = jwtService.createAccessToken(oauth2User);
        String refreshToken = jwtService.createRefreshToken(oauth2User);

        // 对于前后端分离，返回 JWT 而非创建 Session
        response.setContentType("application/json");
        response.getWriter().write(String.format(
            "{\"accessToken\":\"%s\",\"refreshToken\":\"%s\"}", jwt, refreshToken));
    }
}
```

## 6.2 多 Provider 下的不同处理

```java
@Configuration
@EnableWebSecurity
public class MultiOAuth2Config {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .successHandler((request, response, authentication) -> {
                    OAuth2User oauth2User = (OAuth2User) authentication.getPrincipal();
                    String provider = resolveProvider(oauth2User);

                    // 不同 Provider 不同处理
                    switch (provider) {
                        case "google":
                            response.sendRedirect("/google-dashboard");
                            break;
                        case "github":
                            response.sendRedirect("/github-dashboard");
                            break;
                        case "my-sso":
                            // 企业 SSO → 检查是否需要补充注册信息
                            response.sendRedirect("/sso-home");
                            break;
                        default:
                            response.sendRedirect("/home");
                    }
                })
            );
        return http.build();
    }
}
```

# 7. 常见问题处理

## 7.1 注册与登录合一

```java
@Service
public class CustomOAuth2UserService
        implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) {
        OAuth2User oauth2User = new DefaultOAuth2UserService()
            .loadUser(userRequest);

        String email = oauth2User.getAttribute("email");
        String registrationId = userRequest.getClientRegistration()
            .getRegistrationId();

        User user = userRepository.findByEmail(email)
            .orElseGet(() -> createNewUser(oauth2User, registrationId));

        // 如果用户是首次登录，需要额外处理
        if (user.isNewlyCreated()) {
            // 可以抛出一个自定义异常让用户补充信息
            // 也可以在 Session 中标记，引导到补充信息页面
        }

        return new CustomOAuth2User(user, oauth2User.getAttributes());
    }
}
```

## 7.2 关联已有账号

```java
/**
 * 场景：用户已有本地账号，想用 Google 登录关联
 */
@Service
public class AccountLinkingService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private UserConnectionRepository connectionRepository;

    /**
     * 关联 OAuth2 账号到现有本地账号
     */
    public void linkAccount(User currentUser, OAuth2User oauth2User, String providerId) {
        // 检查是否已被其他用户关联
        Optional<UserConnection> existing = connectionRepository
            .findByProviderIdAndProviderUserId(providerId, oauth2User.getName());

        if (existing.isPresent() && !existing.get().getUserId().equals(currentUser.getId())) {
            throw new IllegalStateException("该账号已关联其他用户");
        }

        // 创建关联记录
        UserConnection connection = new UserConnection();
        connection.setUserId(currentUser.getId());
        connection.setProviderId(providerId);
        connection.setProviderUserId(oauth2User.getName());
        connection.setDisplayName(oauth2User.getAttribute("name"));
        connection.setEmail(oauth2User.getAttribute("email"));
        connectionRepository.save(connection);
    }
}
```

## 7.3 Session 过期后刷新 Token

```java
/**
 * 当 Session 中 OAuth2 的 Token 过期时自动刷新
 */
@Bean
public OAuth2AuthorizedClientProvider authorizedClientProvider(
        OAuth2AuthorizedClientRepository authorizedClientRepository) {

    return OAuth2AuthorizedClientProviderBuilder.builder()
        .authorizationCode()
        .refreshToken(refreshToken -> refreshToken
            .clockSkew(Duration.ofSeconds(30))
            .clock(new SystemClock())
        )
        .build();
}
```

# 8. 总结

| 配置项 | 说明 |
|--------|------|
| `registration.{id}.client-id` | 客户端 ID |
| `registration.{id}.client-secret` | 客户端密钥 |
| `registration.{id}.scope` | 请求权限范围 |
| `provider.{id}.issuer-uri` | OIDC Discovery 地址 |
| `provider.{id}.user-name-attribute` | 用户名对应的属性名 |
| 登录入口 | `/oauth2/authorization/{registrationId}` |
| 回调地址 | `/login/oauth2/code/{registrationId}` |
| 自定义用户 | 实现 `OAuth2UserService` 并注入 |
| Token 管理 | 使用 `OAuth2AuthorizedClientManager` |
| 持久化 | JDBC 存储 + `oauth2_authorized_client` 表 |
