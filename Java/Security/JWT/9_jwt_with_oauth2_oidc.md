# 1. JWT 在 OAuth2 与 OpenID Connect 中的角色

JWT 是 OAuth2 和 OIDC 生态中的核心令牌格式。理解三者的关系是掌握现代身份认证体系的关键。

## 1.1 三者关系概览

```
┌──────────────────────────────────────────────────────┐
│                    OpenID Connect                      │
│               (OAuth2 之上的身份认证层)                 │
│  ┌────────────────────────────────────────────────┐  │
│  │              OAuth 2.0 (授权框架)                │  │
│  │  ┌──────────────────────────────────────────┐  │  │
│  │  │        JWT (令牌格式)                      │  │  │
│  │  └──────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

- **OAuth 2.0**：授权框架，定义如何让第三方应用获取有限的资源访问权限
- **OpenID Connect (OIDC)**：OAuth2 之上的身份认证层
- **JWT**：令牌编码格式，OAuth2 的 Access Token 和 OIDC 的 ID Token 均可使用 JWT

---

# 2. OAuth 2.0 中的 JWT

## 2.1 OAuth 2.0 角色

| 角色 | 说明 | 示例 |
|------|------|------|
| Resource Owner（资源所有者） | 用户本人 | 终端用户 |
| Client（客户端） | 请求访问资源的应用 | Web App、Mobile App |
| Authorization Server（授权服务器） | 签发令牌的服务器 | Auth0、Keycloak、自建 |
| Resource Server（资源服务器） | 托管受保护资源的服务器 | API 服务 |

## 2.2 OAuth 2.0 中的 JWT 作为 Access Token

传统 OAuth2 的 Access Token 是不透明字符串（Bearer Token），但 RFC 9068 定义了 JWT Profile for OAuth2 Access Token：

### 2.2.1 JWT Access Token 的结构

```json
// JWT Header
{
  "alg": "RS256",
  "typ": "at+jwt",   // RFC 9068 规定的类型标识
  "kid": "2025-key-1"
}

// JWT Payload
{
  "iss": "https://auth.example.com",
  "sub": "user123",
  "aud": ["https://api.example.com", "https://service-b.example.com"],
  "client_id": "web-app-client",
  "iat": 1712345600,
  "exp": 1712346500,
  "jti": "unique-token-id",
  "scope": "openid profile email api:read api:write",
  "roles": ["USER"]
}
```

## 2.3 OAuth 2.0 授权流程

### 2.3.1 授权码流程（Authorization Code Flow）

```
┌──────────┐     ┌───────────────┐     ┌────────────────────┐     ┌───────────────┐
│  User    │     │    Client     │     │ Authorization Server│     │Resource Server│
│ (Browser)│     │  (Web App)    │     │      (Auth0)         │     │   (API)       │
└────┬─────┘     └──────┬────────┘     └─────────┬──────────┘     └───────┬───────┘
     │                   │                        │                        │
     │ 1. 访问应用       │                        │                        │
     │──────────────────>│                        │                        │
     │                   │ 2. 重定向到授权页面    │                        │
     │<──────────────────│───────────────────────>│                        │
     │                   │                        │                        │
     │ 3. 输入凭证+授权   │                        │                        │
     │───────────────────────────────────────────>│                        │
     │                   │                        │                        │
     │                   │ 4. 授权码 (code)        │                        │
     │                   │<───────────────────────│                        │
     │                   │                        │                        │
     │                   │ 5. POST /token          │                        │
     │                   │   (code + client_secret)│                        │
     │                   │───────────────────────>│                        │
     │                   │                        │                        │
     │                   │ 6. {access_token,       │                        │
     │                   │    id_token,            │                        │
     │                   │    refresh_token}       │                        │
     │                   │<───────────────────────│                        │
     │                   │                        │                        │
     │                   │ 7. API 调用 (access_token)                       │
     │                   │─────────────────────────────────────────────────>│
     │                   │ 8. 200 OK                                       │
     │                   │<─────────────────────────────────────────────────│
```

### 2.3.2 客户端凭据流程（Client Credentials Flow）

用于服务间通信，不涉及用户：

```java
// 客户端获取 Access Token
HttpHeaders headers = new HttpHeaders();
headers.setBasicAuth(clientId, clientSecret);
headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

String body = "grant_type=client_credentials&scope=api:read";

HttpEntity<String> request = new HttpEntity<>(body, headers);
ResponseEntity<Map> response = restTemplate.postForEntity(
    "https://auth.example.com/oauth/token", request, Map.class
);

String accessToken = (String) response.getBody().get("access_token");
// Access Token 是 JWT，可直接解码验证，无需调用 Introspection 端点
```

## 2.4 JWT vs Opaque Token 在 OAuth2 中的对比

| 维度 | JWT Access Token | Opaque Access Token |
|------|-----------------|---------------------|
| 验证方式 | 本地签名验证 | 调用 Introspection 端点 |
| 网络开销 | 无（本地验证） | 每次需 API 调用 |
| 实时性 | 差（吊销需黑名单） | 好（每次查询最新状态） |
| 体积 | 大（数百字节） | 小（数十字节） |
| 自包含 | 是 | 否 |

## 2.5 OAuth 2.0 Token Introspection（RFC 7662）

即使使用 JWT 作为 Access Token，某些场景仍可能需要 Introspection：

```java
// Resource Server 端：验证 Access Token 并获取最新状态
@PostMapping("/introspect")
public Map<String, Object> introspect(@RequestParam("token") String token) {
    // 1. 本地验证 JWT 签名
    Claims claims = jwtParser.parseClaimsJws(token).getBody();
    
    // 2. 检查是否被吊销（黑名单）
    boolean active = !blacklistService.isBlacklisted(claims.getId());
    
    return Map.of(
        "active", active,
        "sub", claims.getSubject(),
        "scope", claims.get("scope", String.class),
        "client_id", claims.get("client_id", String.class),
        "exp", claims.getExpiration().getTime() / 1000,
        "iat", claims.getIssuedAt().getTime() / 1000
    );
}
```

---

# 3. OpenID Connect (OIDC) 中的 JWT

## 3.1 OIDC 核心概念

OIDC 在 OAuth2 之上增加了 **ID Token** 和 **UserInfo Endpoint**：

```
OAuth2: Access Token → 授权（你能访问什么资源）
OIDC:   ID Token     → 认证（你是谁）
```

## 3.2 ID Token

### 3.2.1 ID Token 的结构

ID Token 始终是 JWT 格式：

```json
// Header
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "2025-key-1"
}

// Payload
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",    // 用户唯一 ID
  "aud": "your-client-id.apps.googleusercontent.com",  // 必须是此客户端的 ID
  "iat": 1712345600,
  "exp": 1712346200,                // ID Token 通常短有效期（≤10分钟）
  "nonce": "random-string-xyz",     // 防重放
  "azp": "your-client-id",          // 授权方
  "auth_time": 1712345600,          // 最近认证时间
  "at_hash": "xxx",                 // Access Token 哈希
  "email": "user@gmail.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://example.com/photo.jpg"
}
```

### 3.2.2 ID Token 的用途

- **一次性身份断言**：证明用户在某个时间点通过了认证
- **客户端获取用户信息**：包含用户的基本身份信息
- **不用于 API 授权**：ID Token 不应作为 API 调用的凭证

### 3.2.3 ID Token 验证清单

```
□ 验证签名（使用 JWK Set 中的公钥）
□ 验证 iss（是否匹配预期的 OIDC Provider）
□ 验证 aud（是否包含当前 Client ID）
□ 验证 exp（是否未过期）
□ 验证 iat（是否在合理范围内）
□ 验证 nonce（如果请求中使用了 nonce）
□ 验证 azp（如果有多个 aud）
□ 验证 auth_time（如果需要强制重新认证）
```

## 3.3 OIDC 的 JWT 相关端点

| 端点 | URL | 返回的 JWT |
|------|-----|-----------|
| Authorization | `/authorize` | 返回 authorization code |
| Token | `/token` | Access Token (可能为 JWT) + ID Token (JWT) |
| UserInfo | `/userinfo` | JWT 或 JSON |
| JWK Set | `/.well-known/jwks.json` | JWK Set（公钥集合） |
| Discovery | `/.well-known/openid-configuration` | OIDC 元数据 |

---

# 4. JWT Profile for OAuth 2.0 Access Tokens（RFC 9068）

## 4.1 核心要求

RFC 9068 定义了 OAuth2 Access Token 使用 JWT 格式的规范：

### 4.1.1 类型声明

JWT Access Token 的 `typ` Header 必须设置为 `at+jwt`：

```json
{
  "alg": "RS256",
  "typ": "at+jwt",
  "kid": "2025-key-1"
}
```

### 4.1.2 必需的 Claims

| Claim | 要求 | 说明 |
|-------|------|------|
| `iss` | 必需 | Authorization Server 的标识 |
| `sub` | 必需 | 资源所有者标识（客户端凭据模式可省略） |
| `aud` | 必需 | Resource Server 标识（可数组） |
| `exp` | 必需 | 过期时间 |
| `iat` | 必需 | 签发时间 |
| `jti` | 必需 | 唯一标识 |
| `client_id` | 必需 | 客户端的 OAuth2 Client ID |
| `scope` | 条件 | 当有 scope 时必须包含 |

```json
{
  "iss": "https://auth.example.com",
  "sub": "user123",
  "aud": "https://api.example.com",
  "client_id": "s6BhdRkqt3",
  "iat": 1712345600,
  "exp": 1712346500,
  "jti": "unique-token-id",
  "scope": "read write"
}
```

---

# 5. 使用 Spring Security 实现 OAuth2 Resource Server

## 5.1 完整配置示例

### 5.1.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### 5.1.2 application.yml

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:9000
          jwk-set-uri: http://localhost:9000/oauth2/jwks
          authorities-claim-name: roles
          authority-prefix: ROLE_
```

### 5.1.3 SecurityConfig

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(AbstractHttpConfigurer::disable);
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthorities = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthorities.setAuthorityPrefix("ROLE_");
        grantedAuthorities.setAuthoritiesClaimName("roles");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthorities);
        return converter;
    }
}
```

---

# 6. 自建 Authorization Server（使用 Spring Authorization Server）

## 6.1 Spring Authorization Server 1.x 概述

Spring Authorization Server 是 Spring 团队提供的 OAuth2 + OIDC 认证服务器实现。

### 6.1.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

### 6.1.2 基本配置

```java
@Configuration
public class AuthorizationServerConfig {
    
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public SecurityFilterChain authServerSecurityFilterChain(HttpSecurity http) 
            throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        
        http
            .getConfigurer(OAuth2AuthorizationServerConfigurer.class)
                .oidc(Customizer.withDefaults());
        
        http
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/login"))
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        
        return http.build();
    }
    
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("web-client")
            .clientSecret("{noop}secret")  // 生产环境使用 BCrypt
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .redirectUri("http://127.0.0.1:8080/login/oauth2/code/web-client")
            .scope("openid")
            .scope("profile")
            .scope("email")
            .clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(true)
                .build())
            .build();
        
        return new InMemoryRegisteredClientRepository(client);
    }
    
    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        // 生成 RSA 密钥对
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        
        RSAKey rsaKey = new RSAKey.Builder(publicKey)
            .privateKey(privateKey)
            .keyID(UUID.randomUUID().toString())
            .build();
        
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }
    
    private static KeyPair generateRsaKey() {
        try {
            KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
            generator.initialize(2048);
            return generator.generateKeyPair();
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

### 6.1.3 自定义 JWT Claims

```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> jwtCustomizer() {
    return context -> {
        JwtClaimsSet.Builder claims = context.getClaims();
        
        // 添加自定义 Claims
        Authentication principal = context.getPrincipal();
        if (principal.getPrincipal() instanceof UserDetails user) {
            claims.claim("email", getEmail(user));
            claims.claim("roles", user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList());
        }
    };
}
```

---

# 7. OIDC Discovery 机制

## 7.1 Discovery Endpoint

每个 OIDC Provider 必须提供 Discovery 端点：

```
GET /.well-known/openid-configuration
```

返回示例：

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/oauth2/authorize",
  "token_endpoint": "https://auth.example.com/oauth2/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/oauth2/jwks",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic"]
}
```

## 7.2 使用 Spring Security 自动发现

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # Spring Security 会自动追加 /.well-known/openid-configuration
          # 并从中提取 jwks_uri
```

---

# 8. JWT 与 OAuth2 的安全最佳实践

## 8.1 Access Token 安全

```
□ Access Token 使用 RSA/ECDSA 非对称签名（RS256/ES256）
□ Access Token 有效期短（5-30 分钟）
□ 配合 Refresh Token 使用
□ typ 设置为 "at+jwt"（RFC 9068 合规）
□ aud 声明必须限定为特定 Resource Server
□ 使用 scope 声明做细粒度权限控制
□ Resource Server 本地验证签名，不每次都调用 Introspection
```

## 8.2 ID Token 安全

```
□ ID Token 不使用作用 API 调用
□ ID Token 验证 aud 必须包含 Client ID
□ 在授权请求中使用 nonce（防重放）
□ auth_time 用于强制重新认证（max_age 参数）
□ 验证 at_hash 和 c_hash（防止跨令牌攻击）
□ ID Token 不在前端日志中打印
```

## 8.3 混合攻击防范

**跨 JWT 混淆攻击**：攻击者将 ID Token 用作 Access Token

```java
// Resource Server 应区分两种 Token
public class TokenTypeValidator {
    
    private static final String ACCESS_TOKEN_TYPE = "at+jwt";
    private static final String ID_TOKEN_TYPE = "JWT";
    
    public void validateAccessToken(String token) {
        DecodedJWT jwt = JWT.decode(token);
        String typ = jwt.getHeaderClaim("typ").asString();
        
        // Access Token 必须有正确的类型
        if (!ACCESS_TOKEN_TYPE.equals(typ)) {
            throw new InvalidTokenException(
                "Invalid token type for access token: " + typ);
        }
    }
}
```
