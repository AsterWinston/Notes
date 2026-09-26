
# 1. 概述

Spring Security OAuth2 Resource Server 是服务端验证 OAuth 2.0 Bearer Token（通常是 JWT）的模块，用于保护 REST API 资源。它是微服务架构下实现**无状态认证**的核心组件。

```
┌──────────────────────────────────────────────────────────────┐
│                  Resource Server 架构                         │
│                                                               │
│  客户端请求                                   资源服务器       │
│  Authorization: Bearer eyJhbGci...          ┌───────────┐    │
│ ───────────────────────────────────────────►│ JWT 验签  │    │
│                                             │ ↓         │    │
│                                             │ 提取Claims│    │
│                                             │ ↓         │    │
│                                             │ 构建Auth  │    │
│                                             │ ↓         │    │
│                                             │ 权限校验  │    │
│                                             │ ↓         │    │
│                                             │ 处理业务  │    │
│                                             └───────────┘    │
│                                                               │
│  JWT 验签公钥来源：                                           │
│  ├─ jwk-set-uri: https://auth-server/.well-known/jwks.json   │
│  ├─ issuer-uri: https://auth-server (自动发现 jwks)           │
│  └─ 本地公钥文件 / 配置                                        │
└──────────────────────────────────────────────────────────────┘
```

# 2. 快速入门

## 2.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

## 2.2 最小配置

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # 方式1：通过 issuer-uri 自动发现 JWKS
          issuer-uri: https://auth.example.com

          # 方式2：直接指定 JWKS 端点
          # jwk-set-uri: https://auth.example.com/.well-known/jwks.json

          # 方式3：使用本地公钥（HMAC 对称密钥也类似）
          # public-key-location: classpath:public-key.pem
```

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            );
        return http.build();
    }
}
```

# 3. JWT 验证机制深度解析

## 3.1 过滤器链

```java
/**
 * Resource Server 的认证流程：
 *
 * 1. BearerTokenAuthenticationFilter 拦截请求
 *    从 Authorization 头提取 Bearer Token
 *
 * 2. BearerTokenAuthenticationFilter → AuthenticationManager
 *    → ProviderManager → JwtAuthenticationProvider
 *
 * 3. JwtAuthenticationProvider.authenticate()
 *    ├─ JwtDecoder.decode(token)
 *    │    ├─ 验证签名（使用 JWKS 中的公钥）
 *    │    ├─ 验证 iss（如果配置了 issuer-uri）
 *    │    └─ 验证 exp/nbf
 *    └─ 创建 JwtAuthenticationToken
 *         ├─ principal = Jwt（包含所有 claims）
 *         └─ authorities = 从 Jwt 中提取
 *
 * 4. SecurityContextHolder 设置认证信息
 *
 * 5. 后续 FilterSecurityInterceptor 执行授权决策
 */
```

## 3.2 NimbusJwtDecoder 配置

```java
@Bean
public JwtDecoder jwtDecoder() {
    // 方式1：使用 issuer-uri 自动发现（推荐）
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withIssuerLocation("https://auth.example.com")
        .build();

    // 方式2：使用 JWKS 端点
    // NimbusJwtDecoder decoder = NimbusJwtDecoder
    //     .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
    //     .build();

    // 方式3：使用本地公钥
    // RSAPublicKey publicKey = ...;
    // NimbusJwtDecoder decoder = NimbusJwtDecoder
    //     .withPublicKey(publicKey)
    //     .build();

    // 自定义验证规则
    decoder.setJwtValidator(
        JwtValidators.createDefaultWithIssuer("https://auth.example.com")
    );

    // 自定义 Claim 验证
    OAuth2TokenValidator<Jwt> audienceValidator = new JwtClaimValidator<List<String>>(
        "aud", aud -> aud.contains("my-api-gateway")
    );

    decoder.setJwtValidator(
        new DelegatingOAuth2TokenValidator<>(
            JwtValidators.createDefaultWithIssuer("https://auth.example.com"),
            audienceValidator
        )
    );

    return decoder;
}
```

## 3.3 自定义 JwtAuthenticationConverter

```java
/**
 * 从 JWT Claims 中提取权限（Roles/Authorities）
 */
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
        new JwtGrantedAuthoritiesConverter();
    grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
    grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
    // 如果 scope 格式: "SCOPE_read"
    // grantedAuthoritiesConverter.setAuthorityPrefix("SCOPE_");

    JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
    jwtConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
    return jwtConverter;
}

/**
 * 更复杂的权限提取逻辑
 */
@Component
public class CustomJwtGrantedAuthoritiesConverter
        implements Converter<Jwt, Collection<GrantedAuthority>> {

    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        List<GrantedAuthority> authorities = new ArrayList<>();

        // 1. 从 roles Claim 提取角色
        List<String> roles = jwt.getClaimAsStringList("roles");
        if (roles != null) {
            roles.forEach(role ->
                authorities.add(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase())));
        }

        // 2. 从 scope Claim 提取权限
        String scope = jwt.getClaimAsString("scope");
        if (scope != null) {
            Arrays.stream(scope.split(" "))
                .map(s -> new SimpleGrantedAuthority("SCOPE_" + s))
                .forEach(authorities::add);
        }

        // 3. 从 permissions Claim 提取细粒度权限
        List<String> permissions = jwt.getClaimAsStringList("permissions");
        if (permissions != null) {
            permissions.forEach(perm ->
                authorities.add(new SimpleGrantedAuthority(perm)));
        }

        return authorities;
    }
}
```

# 4. 多租户 JWT 验证

```java
/**
 * 场景：不同租户的 JWT 有不同的签发者
 * 需要根据请求动态选择 JwtDecoder
 */
@Component
public class MultiTenantJwtDecoder implements JwtDecoder {

    private final Map<String, JwtDecoder> decoders = new ConcurrentHashMap<>();

    /**
     * 注册租户的 JwtDecoder
     */
    public void registerTenant(String tenantId, String issuerUri) {
        JwtDecoder decoder = NimbusJwtDecoder
            .withIssuerLocation(issuerUri)
            .build();
        decoders.put(tenantId, decoder);
    }

    @Override
    public Jwt decode(String token) throws JwtException {
        // 先解析 JWT 获取 iss（不验证签名）
        try {
            JWT parsed = JWTParser.parse(token);
            String issuer = parsed.getJWTClaimsSet().getIssuer();

            // 根据 issuer 选择对应的 decoder
            for (Map.Entry<String, JwtDecoder> entry : decoders.entrySet()) {
                try {
                    return entry.getValue().decode(token);
                } catch (JwtException e) {
                    // 尝试下一个 decoder
                }
            }

            throw new JwtException("No matching decoder found for issuer: " + issuer);
        } catch (Exception e) {
            throw new JwtException("Failed to decode JWT: " + e.getMessage(), e);
        }
    }
}
```

# 5. Opaque Token 模式

## 5.1 何时使用

```
JWT 模式：Token 自包含信息，本体验证（签名验证）
Opaque Token 模式：Token 是不透明字符串，需调用授权服务器验证

使用 Opaque Token 的场景：
- 授权服务器不签发 JWT（老系统）
- 需要实时检查 Token 是否被吊销
- Token 信息量大，不适合放在 JWT 中
```

## 5.2 Opaque Token 配置

```xml
<!-- 额外依赖 -->
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>oauth2-oidc-sdk</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/oauth2/introspect
          client-id: my-resource-server
          client-secret: my-secret
```

```java
@Configuration
@EnableWebSecurity
public class OpaqueTokenConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .opaqueToken(opaque -> opaque
                    .introspectionUri("https://auth.example.com/oauth2/introspect")
                    .introspectionClientCredentials("resource-server", "secret")
                )
            );
        return http.build();
    }
}
```

## 5.3 Token Introspection 响应

```json
// POST /oauth2/introspect
// Authorization: Basic base64(resource-server:secret)
// Content-Type: application/x-www-form-urlencoded
// token=xxx

// 有效 Token 响应：
{
    "active": true,
    "sub": "user-12345",
    "scope": "read write",
    "client_id": "my-app",
    "username": "zhangsan",
    "exp": 1712345678,
    "iat": 1712341978,
    "authorities": ["ROLE_USER", "ROLE_ADMIN"]
}

// 无效 Token 响应：
{
    "active": false
}
```

# 6. 自定义异常处理

## 6.1 认证失败处理

```java
/**
 * 处理 401 Unauthorized（无 Token / Token 无效）
 */
@Component
public class CustomAuthenticationEntryPoint
        implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) {

        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        String message;
        if (authException instanceof InsufficientAuthenticationException) {
            message = "缺少认证令牌";
        } else if (authException instanceof JwtException) {
            message = "JWT 令牌无效或已过期";
        } else {
            message = "认证失败";
        }

        response.getWriter().write(String.format(
            "{\"code\":401,\"message\":\"%s\"}", message));
    }
}

/**
 * 处理 403 Forbidden（有 Token 但权限不足）
 */
@Component
public class CustomAccessDeniedHandler
        implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) {

        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.getWriter().write(
            "{\"code\":403,\"message\":\"权限不足\"}");
    }
}

// 配置：
http
    .exceptionHandling(exceptions -> exceptions
        .authenticationEntryPoint(customAuthenticationEntryPoint)
        .accessDeniedHandler(customAccessDeniedHandler)
    );
```

## 6.2 JWT 验证失败的详细处理

```java
/**
 * 自定义 BearerTokenAuthenticationEntryPoint
 * 区分不同类型的 JWT 错误
 */
public class DetailedBearerTokenAuthenticationEntryPoint
        extends BearerTokenAuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) {

        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        String errorCode;
        String errorMessage;

        Throwable cause = authException.getCause();
        if (cause instanceof BadJwtException) {
            errorCode = "invalid_token";
            errorMessage = "JWT 格式无效或签名验证失败";
        } else if (cause instanceof JwtValidationException) {
            JwtValidationException jve = (JwtValidationException) cause;
            if (jve.getErrors().stream().anyMatch(
                    e -> e instanceof InvalidTokenException
                      && "expired".equals(e.getMessage()))) {
                errorCode = "token_expired";
                errorMessage = "JWT 已过期";
            } else {
                errorCode = "token_invalid";
                errorMessage = "JWT 验证失败";
            }
        } else if (authException instanceof InsufficientAuthenticationException) {
            errorCode = "missing_token";
            errorMessage = "缺少 Bearer Token";
        } else {
            errorCode = "unauthorized";
            errorMessage = "认证失败";
        }

        response.getWriter().write(String.format(
            "{\"error\":\"%s\",\"error_description\":\"%s\"}",
            errorCode, errorMessage));
    }
}
```

# 7. JWT 与 Session 共存

```java
/**
 * 同时支持 JWT Bearer Token（API）和 Session（Web 页面）
 * 使用多重 SecurityFilterChain
 */
@Configuration
@EnableWebSecurity
public class MixedAuthConfig {

    /**
     * API 路径 → JWT 无状态认证
     */
    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(AbstractHttpConfigurer::disable);
        return http.build();
    }

    /**
     * Web 页面 → Session 认证（可配合 SSO）
     */
    @Bean
    @Order(2)
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/web/**", "/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/web/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            // 或 .oauth2Login(...) 或 .cas(...)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED));
        return http.build();
    }
}
```

# 8. 性能优化

## 8.1 JWT 验签缓存

```java
/**
 * NimbusJwtDecoder 默认每次验签都获取 JWKS
 * 高并发下成为瓶颈，应启用缓存
 */
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .cache(Duration.ofMinutes(10))          // JWKS 缓存 10 分钟
        .jwtProcessorCustomizer(processor ->
            processor.setJWSTypeVerifier(
                new DefaultJOSEObjectTypeVerifier<>(new JOSEObjectType("JWT"))
            )
        )
        .build();

    return decoder;
}
```

## 8.2 预解析 JWT 获取 kid

```java
/**
 * 支持多密钥轮换：根据 JWT header 中的 kid 预加载特定公钥
 */
@Bean
public JwtDecoder jwtDecoder() {
    return token -> {
        // 从 Token Header 解析 kid
        JWT jwt = JWTParser.parse(token);
        String kid = (String) jwt.getHeader().get("kid");

        // 根据 kid 选择对应公钥
        RSAPublicKey publicKey = keyCache.get(kid);
        if (publicKey == null) {
            // 从 JWKS 重新加载
            publicKey = loadPublicKey(kid);
            keyCache.put(kid, publicKey);
        }

        // 使用特定公钥验证
        NimbusJwtDecoder decoder = NimbusJwtDecoder
            .withPublicKey(publicKey).build();
        return decoder.decode(token);
    };
}
```

# 9. 总结

| 配置项 | 说明 |
|--------|------|
| `issuer-uri` | OIDC 签发者 URL，自动发现 JWKS |
| `jwk-set-uri` | 直接指定 JWKS 端点 |
| `public-key-location` | 本地公钥路径 |
| `jwtAuthenticationConverter` | 自定义 Claims → Authorities 映射 |
| `authenticationEntryPoint` | 处理 401（无/无效 Token） |
| `accessDeniedHandler` | 处理 403（权限不足） |
| `NimbusJwtDecoder` | 核心 JWT 解码器，支持 Issuer/JWKS/公钥 |
| Opaque Token | 调用 `/introspect` 端点实时验证 |
