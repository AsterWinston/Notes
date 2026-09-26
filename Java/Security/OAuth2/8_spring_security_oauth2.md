# 1. Spring Security OAuth2 概述

Spring Security 提供了对 OAuth2 的一等支持，覆盖了客户端、资源服务器和授权服务器三种角色。Spring Security 6.x + Spring Boot 3.x 采用全新的 OAuth2 实现。

## 1.1 Spring Security OAuth2 演进

```
Spring Security OAuth2 演进历程：

┌──────────────────────────────────────────────────────────┐
│ Spring Security OAuth 2.x（已废弃）                       │
│ - 独立的 spring-security-oauth2 模块                      │
│ - 注解驱动：@EnableAuthorizationServer, @EnableResourceServer│
│ - 内部管理 Token 存储                                     │
│ - 2022 年正式退役（EOL）                                  │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│ Spring Security 5.x / Spring Boot 2.x                     │
│ - 新模块：spring-security-oauth2-client                   │
│ - 新模块：spring-security-oauth2-resource-server          │
│ - 新模块：spring-security-oauth2-jose                    │
│ - 不再提供授权服务器实现                                  │
│ - 推荐使用第三方授权服务器（Keycloak, Auth0, Okta）       │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│ Spring Security 6.x / Spring Boot 3.x（当前版本）         │
│ - 全面采用新 API（Lambda DSL）                            │
│ - 更好的 OIDC 支持                                       │
│ - Spring Authorization Server 作为官方授权服务器           │
└──────────────────────────────────────────────────────────┘
```

> **重要变化**：Spring 不再在 `spring-security` 自身中提供授权服务器实现。需要实现授权服务器时，使用独立的 **Spring Authorization Server** 项目。

---

# 2. OAuth2 Client（客户端）集成

## 2.1 基础配置

### Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### application.yml 配置

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - user:email
              - read:user
            client-name: GitHub
            
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - openid
              - profile
              - email
              
        provider:
          github:
            authorization-uri: https://github.com/login/oauth/authorize
            token-uri: https://github.com/login/oauth/access_token
            user-info-uri: https://api.github.com/user
            user-name-attribute: login
            
          google:
            # 使用 OIDC 发现，无需手动配置端点
            issuer-uri: https://accounts.google.com
            user-name-attribute: sub
```

### SecurityConfig 基础配置

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
            );
        
        return http.build();
    }
}
```

> 通过 `.oauth2Login()` 配置了 OAuth2 登录后，Spring Security 自动处理：
> - 重定向到授权服务器（`/oauth2/authorization/{registrationId}`）
> - 接收回调（`/login/oauth2/code/{registrationId}`）
> - 交换 Token
> - 加载用户信息
> - 创建 `OAuth2AuthenticationToken`

## 2.2 自定义 OAuth2UserService

```java
/**
 * 自定义 OAuth2 用户服务
 * 在 Spring Security 标准流程中插入自定义逻辑：
 * - 获取更多用户属性
 * - 查找或创建本地用户
 * - 分配本地权限
 */
@Component
public class CustomOAuth2UserService 
        extends DefaultOAuth2UserService {
    
    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    
    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) {
        // 1. 调用默认实现（获取标准用户信息）
        OAuth2User oAuth2User = super.loadUser(userRequest);
        
        // 2. 获取注册信息
        String registrationId = userRequest.getClientRegistration()
            .getRegistrationId();  // "github" / "google"
        String userNameAttributeName = userRequest.getClientRegistration()
            .getProviderDetails().getUserInfoEndpoint()
            .getUserNameAttributeName();  // "login" / "sub"
        
        // 3. 提取用户属性
        Map<String, Object> attributes = oAuth2User.getAttributes();
        String externalId = String.valueOf(attributes.get(userNameAttributeName));
        String email = extractEmail(registrationId, attributes);
        String name = extractName(registrationId, attributes);
        String avatarUrl = extractAvatar(registrationId, attributes);
        
        // 4. 查找或创建本地用户
        UserEntity user = userRepository.findByExternalIdAndProvider(externalId, registrationId)
            .orElseGet(() -> createNewUser(externalId, registrationId, email, name, avatarUrl));
        
        // 5. 获取用户权限
        Set<GrantedAuthority> authorities = loadAuthorities(user);
        
        // 6. 返回自定义的 OAuth2User 实现
        return new CustomOAuth2User(
            authorities, attributes, userNameAttributeName,
            user.getId(), user.getEmail(), user.getName()
        );
    }
    
    private String extractEmail(String registrationId, Map<String, Object> attributes) {
        return switch (registrationId) {
            case "github" -> {
                // GitHub 的 email 可能在主属性中，也可能需要通过 email API 获取
                // 这里通过 OAuth2UserRequest 的 Access Token 调用 email API
                yield (String) attributes.get("email");
            }
            case "google" -> (String) attributes.get("email");
            default -> null;
        };
    }
}
```

## 2.3 访问 Token 以调用 API

```java
/**
 * 使用 OAuth2 Access Token 调用第三方 API
 */
@RestController
public class GitHubController {
    
    /**
     * 通过注入 OAuth2AuthorizedClientService 获取 Access Token
     */
    private final OAuth2AuthorizedClientService authorizedClientService;
    
    @GetMapping("/api/github/repos")
    public List<Repository> getRepos(
            @AuthenticationPrincipal OAuth2User user) {
        
        // 获取当前用户的 GitHub 授权客户端
        OAuth2AuthenticationToken authToken = 
            (OAuth2AuthenticationToken) SecurityContextHolder
                .getContext().getAuthentication();
        
        OAuth2AuthorizedClient authorizedClient = authorizedClientService
            .loadAuthorizedClient(
                authToken.getAuthorizedClientRegistrationId(),
                authToken.getName()
            );
        
        if (authorizedClient == null) {
            throw new UnauthorizedException("Not authorized with GitHub");
        }
        
        String accessToken = authorizedClient.getAccessToken().getTokenValue();
        
        // 使用 Access Token 调用 GitHub API
        return githubApiService.listRepos(accessToken);
    }
    
    /**
     * 自动刷新过期的 Access Token
     */
    private String getValidAccessToken(String registrationId, String principalName) {
        OAuth2AuthorizedClient authorizedClient = authorizedClientService
            .loadAuthorizedClient(registrationId, principalName);
        
        OAuth2AccessToken accessToken = authorizedClient.getAccessToken();
        
        // 检查是否即将过期（提前 60 秒）
        if (accessToken.getExpiresAt() != null &&
            accessToken.getExpiresAt().isBefore(
                Instant.now().plusSeconds(60))) {
            
            // 使用 RefreshTokenGrantRequest 刷新
            OAuth2RefreshToken refreshToken = authorizedClient.getRefreshToken();
            if (refreshToken != null) {
                // Spring Security 自动处理刷新后的 Token 持久化
                // 需要通过 OAuth2AuthorizedClientManager 管理
            }
        }
        
        return accessToken.getTokenValue();
    }
}
```

## 2.4 OAuth2AuthorizedClientManager（更灵活的 Token 管理）

```java
@Configuration
public class OAuth2ClientConfig {
    
    /**
     * 使用 OAuth2AuthorizedClientManager 实现自动 Token 刷新
     * 这是 SPA BFF 或微服务间调用的推荐方式
     */
    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientService authorizedClientService) {
        
        // 用于管理 Access Token 的获取和刷新
        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()     // 授权码模式
                .refreshToken()          // 自动刷新
                .clientCredentials()     // 客户端凭证模式
                .build();
        
        DefaultOAuth2AuthorizedClientManager authorizedClientManager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientService);
        
        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);
        
        // 可选：设置上下文属性映射器
        authorizedClientManager.setContextAttributesMapper(
            context -> {
                Map<String, Object> attributes = new HashMap<>();
                // 从当前请求或会话中获取额外信息
                return attributes;
            }
        );
        
        return authorizedClientManager;
    }
}
```

---

# 3. OAuth2 Resource Server（资源服务器）集成

## 3.1 基础配置

### Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### application.yml 配置

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # 方案一：OIDC Issuer URI（自动发现 JWKS 端点）
          issuer-uri: https://auth.example.com
          
          # 方案二：直接指定 JWKS URI（不依赖 OIDC 发现）
          # jwk-set-uri: https://auth.example.com/oauth2/jwks
          
          # 方案三：本地公钥（仅开发/测试）
          # public-key-location: classpath:public-key.pem
```

### SecurityConfig 配置

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .requestMatchers("/api/**").hasAuthority("SCOPE_read")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // 无状态
            )
            .csrf(AbstractHttpConfigurer::disable);  // 无状态 API 可禁用 CSRF
        
        return http.build();
    }
    
    /**
     * 自定义 JWT 认证转换器
     * 将 JWT Claims 转换为 Spring Security Authentication 对象
     */
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("authorities");  // JWT 中的权限字段
        grantedAuthoritiesConverter.setAuthorityPrefix("");  // 不添加前缀
        
        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        
        return jwtConverter;
    }
}
```

### 在 Controller 中获取认证信息

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping
    public List<Order> listOrders(
            @AuthenticationPrincipal JwtAuthenticationToken authToken) {
        
        // 方式一：从 Authentication 获取 JWT
        Jwt jwt = authToken.getToken();
        String userId = jwt.getSubject();          // sub
        String clientId = jwt.getClaimAsString("client_id");
        String scopes = jwt.getClaimAsString("scope");
        
        // 方式二：使用 @AuthenticationPrincipal Jwt 直接注入
        // 用 @AuthenticationPrincipal Jwt jwt 参数
        
        return orderService.findByUserId(userId);
    }
    
    @GetMapping("/{id}")
    public Order getOrder(
            @PathVariable Long id,
            @AuthenticationPrincipal Jwt jwt) {
        
        String userId = jwt.getSubject();
        return orderService.findByIdAndUserId(id, userId);
    }
}
```

## 3.2 深入 JWT 定制

### 自定义 JWT 验证器

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/oauth2/jwks")
        .jwsAlgorithm(RS256)  // 指定接受的签名算法
        .build();
    
    // 自定义验证规则
    OAuth2TokenValidator<Jwt> audienceValidator = new JwtClaimValidator<List<String>>(
        "aud", aud -> aud.contains("resource-server-1"));
    
    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(
        "https://auth.example.com");
    
    // 组合多个验证器
    OAuth2TokenValidator<Jwt> combinedValidator = 
        new DelegatingOAuth2TokenValidator<>(withIssuer, audienceValidator);
    
    jwtDecoder.setJwtValidator(combinedValidator);
    
    return jwtDecoder;
}
```

### 方法级权限控制

```java
@EnableMethodSecurity
@Configuration
public class MethodSecurityConfig {
    // 启用 @PreAuthorize 注解支持
}

@RestController
@RequestMapping("/api/admin")
public class AdminController {
    
    // 基于 JWT Claim 的权限控制
    @PreAuthorize("hasAuthority('ROLE_ADMIN')")
    @GetMapping("/users")
    public List<User> listUsers() {
        return userService.findAll();
    }
    
    // 基于 Scope 的权限控制
    @PreAuthorize("hasAuthority('SCOPE_user:write')")
    @PostMapping("/users")
    public User createUser(@RequestBody User user) {
        return userService.create(user);
    }
    
    // 基于 SpEL 表达式的复杂权限
    @PreAuthorize("@accessEvaluator.canAccessOrder(#id, authentication)")
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);
    }
}
```

## 3.3 Opaque Token 验证

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/oauth2/introspect
          client-id: resource-server-1
          client-secret: ${RESOURCE_SERVER_SECRET}
```

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authorize -> authorize
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .opaqueToken(opaque -> opaque
                .introspectionUri("https://auth.example.com/oauth2/introspect")
                .introspectionClientCredentials("resource-server-1", secret)
            )
        );
    
    return http.build();
}
```

---

# 4. Spring Authorization Server（授权服务器）

## 4.1 概述

Spring Authorization Server 是 Spring 官方的授权服务器实现，基于 Spring Security 6 构建。

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
    <version>1.3.0</version>
</dependency>
```

## 4.2 基础配置

```java
@Configuration
@EnableWebSecurity
public class AuthorizationServerConfig {
    
    @Bean 
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) 
            throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        
        http
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/login"))
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            );
        
        return http.build();
    }
    
    @Bean 
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) 
            throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());
        
        return http.build();
    }
    
    /**
     * 注册客户端
     */
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient registeredClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("messaging-client")
            .clientSecret("{bcrypt}$2a$10$...")  // BCrypt 编码的密钥
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_POST)
            
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            
            .redirectUri("https://client.example.com/callback")
            .redirectUri("https://client.example.com/authorized")
            
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope(OidcScopes.EMAIL)
            .scope("read")
            .scope("write")
            
            .clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(true)
                .requireProofKey(true)  // PKCE
                .build()
            )
            
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofDays(30))
                .reuseRefreshTokens(false)  // Refresh Token 轮换
                .authorizationCodeTimeToLive(Duration.ofMinutes(5))
                .build()
            )
            .build();
        
        return new InMemoryRegisteredClientRepository(registeredClient);
    }
    
    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRsa();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return (jwkSelector, securityContext) -> jwkSelector.select(jwkSet);
    }
    
    private static RSAKey generateRsa() {
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        return new RSAKey.Builder(publicKey)
            .privateKey(privateKey)
            .keyID(UUID.randomUUID().toString())
            .build();
    }
    
    private static KeyPair generateRsaKey() {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
        keyPairGenerator.initialize(2048);
        return keyPairGenerator.generateKeyPair();
    }
    
    @Bean
    public OAuth2AuthorizationService authorizationService() {
        // 使用 JdbcOAuth2AuthorizationService 持久化到数据库
        return new InMemoryOAuth2AuthorizationService();
    }
    
    @Bean
    public OAuth2AuthorizationConsentService authorizationConsentService() {
        return new InMemoryOAuth2AuthorizationConsentService();
    }
}
```

> 生产环境应使用 `JdbcRegisteredClientRepository`、`JdbcOAuth2AuthorizationService` 和 `JdbcOAuth2AuthorizationConsentService` 替代内存实现。

---

# 5. Spring Cloud Gateway 作为 API 网关集成 OAuth2

## 5.1 配置 Gateway

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/users/**
          filters:
            - TokenRelay=  # 将 OAuth2 Token 传递给下游服务
            
        - id: order-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/orders/**
          filters:
            - TokenRelay=
```

```java
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {
    
    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/public/**").permitAll()
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            )
            .oauth2Login(Customizer.withDefaults());  // 如果需要用户登录
        
        return http.build();
    }
}
```

---

# 6. 常见集成方案

## 6.1 Spring Boot + OAuth2 + Keycloak

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: ${KEYCLOAK_CLIENT_ID}
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            scope: openid, profile, email
        provider:
          keycloak:
            issuer-uri: http://localhost:8080/realms/my-realm
            user-name-attribute: preferred_username
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/my-realm
```

## 6.2 Spring Boot + OAuth2 + Auth0

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          auth0:
            client-id: ${AUTH0_CLIENT_ID}
            client-secret: ${AUTH0_CLIENT_SECRET}
            scope:
              - openid
              - profile
              - email
        provider:
          auth0:
            issuer-uri: https://YOUR_DOMAIN.auth0.com/
      resourceserver:
        jwt:
          issuer-uri: https://YOUR_DOMAIN.auth0.com/
```

---

# 7. Spring Security OAuth2 最佳实践

```
开发实践清单：

□ 在 yml 中使用 ${ENVVAR} 而非硬编码密钥
□ client-secret 使用环境变量或配置中心（Spring Cloud Config / Vault）
□ 生产环境使用数据库存储 Token 和客户端配置（而非 InMemory）
□ 启用 Refresh Token 轮换（reuseRefreshTokens=false）
□ 强制 PKCE（requireProofKey=true）
□ 使用 BCrypt/Argon2 哈希存储 client_secret
□ JWT 签名使用 RS256 或 ES256（而非 HS256）
□ 资源服务器使用无状态 Session（SessionCreationPolicy.STATELESS）
□ 区分 auth 和 API 的 SecurityFilterChain（order 控制）
□ 配置 Token 有效期（Access Token 15 分钟，Refresh Token 30 天）
□ 实现自定义的 redirect_uri 验证（严格匹配）
□ 监控和日志：记录 Token 颁发、刷新、吊销事件
□ 定期轮换签名密钥
□ SPA 场景使用 BFF 模式，不直接将 Token 暴露给浏览器
```
