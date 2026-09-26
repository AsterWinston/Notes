
# 1. 概述

**OpenID Connect 1.0**（OIDC）是建立在 OAuth 2.0 之上的**身份认证层**。OAuth 2.0 解决的是"授权"问题（允许应用访问你的资源），OIDC 解决的是"认证"问题（你是谁）。OIDC 通过引入 **ID Token**（JWT 格式）来承载用户身份信息。

```
┌──────────────────────────────────────────────────────┐
│              OAuth 2.0  ←→  OIDC                     │
│                                                       │
│  OAuth 2.0 是一个授权框架：                           │
│  "我允许这个应用访问我的 Google Drive"                 │
│                                                       │
│  OIDC 是在 OAuth 2.0 上添加了认证能力：               │
│  "我是 zhangsan@example.com，这是我的身份证明"         │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ OAuth 2.0│  │  OIDC    │  │ 需要二者         │   │
│  │  授权    │  │  认证    │  │ 认证 + 授权      │   │
│  └──────────┘  └──────────┘  └──────────────────┘   │
│       │             │                │               │
│       ▼             ▼                ▼               │
│   Access Token    ID Token     Access + ID Token     │
│   (授权凭证)      (身份凭证)   (授权 + 身份)         │
└──────────────────────────────────────────────────────┘
```

# 2. ID Token 详解

## 2.1 ID Token 结构

```
ID Token = JWT（JSON Web Token）
= Header.Payload.Signature

Header:
{
  "alg": "RS256",        ← 签名算法
  "kid": "key-id-123",   ← 密钥标识（用于密钥轮换）
  "typ": "JWT"
}

Payload:
{
  "iss": "https://auth.example.com",     ← 签发者（IdP 的 URL）
  "sub": "248289761001",                 ← 用户唯一标识（Subject）
  "aud": "s6BhdRkqt3",                   ← 受众（Client ID）
  "exp": 1712345678,                     ← 过期时间
  "iat": 1712341978,                     ← 签发时间
  "auth_time": 1712341978,              ← 用户认证时间
  "nonce": "n-0S6_WzA2Mj",              ← 防重放随机数
  "amr": ["pwd", "mfa"],                ← 认证方法引用
  "azp": "s6BhdRkqt3",                  ← 授权方（授权给谁）
  "name": "张三",
  "given_name": "三",
  "family_name": "张",
  "middle_name": "",
  "nickname": "小张",
  "preferred_username": "zhangsan",
  "profile": "https://example.com/zhangsan",
  "picture": "https://example.com/zhangsan/avatar.jpg",
  "website": "https://zhangsan.me",
  "email": "zhangsan@example.com",
  "email_verified": true,
  "gender": "male",
  "birthdate": "1990-01-01",
  "zoneinfo": "Asia/Shanghai",
  "locale": "zh-CN",
  "phone_number": "+86 13800138000",
  "phone_number_verified": false,
  "address": {
    "street_address": "朝阳区望京SOHO",
    "locality": "北京",
    "region": "北京",
    "postal_code": "100000",
    "country": "中国"
  },
  "updated_at": 1712341978
}

Signature:
签名字符串 = Base64URL(Header) + "." + Base64URL(Payload)
签名 = RSASHA256(签名字符串, IdP私钥)
```

## 2.2 ID Token 标准 Claim

| Claim | 类型 | 说明 |
|-------|------|------|
| **iss** | string | Issuer，IdP 的唯一标识 URL |
| **sub** | string | Subject，用户在 IdP 中的唯一标识 |
| **aud** | string/array | Audience，Token 的合法接收者（Client ID） |
| **exp** | number | Expiration，过期时间（Unix 时间戳） |
| **iat** | number | Issued At，签发时间 |
| **auth_time** | number | 用户最近一次认证的时间 |
| **nonce** | string | 客户端发送的随机数，防重放 |
| **amr** | array | Authentication Methods Reference，认证方式 |
| **azp** | string | Authorized Party，实际被授权的客户端 |

## 2.3 ID Token 验证

```java
public class IdTokenValidator {

    /**
     * 完整的 ID Token 验证流程（RFC 7519 + OpenID Connect Core 1.0）
     */
    public Claims validateIdToken(String idTokenString, String clientId,
                                   String issuer, JWKSet jwkSet) {

        // 1. 解析 ID Token（不做验证，先获取 header 中的 kid）
        JWT parsedJwt = JWTParser.parse(idTokenString);

        // 2. 根据 kid 获取对应的公钥
        String kid = parsedJwt.getHeader().getKeyID();
        JWK jwk = jwkSet.getKeyByKeyId(kid);
        if (jwk == null) {
            throw new JWTValidationException("Unknown kid: " + kid);
        }

        // 3. 使用公钥验证签名
        SignedJWT signedJWT = (SignedJWT) parsedJwt;
        JWSVerifier verifier = new RSASSAVerifier((RSAKey) jwk);
        if (!signedJWT.verify(verifier)) {
            throw new JWTValidationException("Invalid signature");
        }

        // 4. 验证 Claims
        Claims claims = signedJWT.getJWTClaimsSet();

        // 4.1 验证 iss（签发者必须匹配）
        if (!issuer.equals(claims.getIssuer())) {
            throw new JWTValidationException("Invalid issuer: " + claims.getIssuer());
        }

        // 4.2 验证 aud（受众包含当前 Client ID）
        List<String> audience = claims.getAudience();
        if (!audience.contains(clientId)) {
            throw new JWTValidationException("Invalid audience");
        }

        // 4.3 验证 exp（未过期）
        Date expirationTime = claims.getExpirationTime();
        if (expirationTime != null && new Date().after(expirationTime)) {
            throw new JWTValidationException("Token expired");
        }

        // 4.4 验证 iat（允许一定时钟偏差，通常 5 分钟）
        Date issuedAt = claims.getIssueTime();
        long clockSkew = 5 * 60 * 1000; // 5 分钟
        if (issuedAt != null && issuedAt.getTime() > System.currentTimeMillis() + clockSkew) {
            throw new JWTValidationException("Token issued in the future");
        }

        // 4.5 验证 auth_time（如果存在，确保在合理时间范围内）
        Date authTime = claims.getDateClaim("auth_time");
        if (authTime != null) {
            long maxAuthAge = 30 * 60 * 1000; // 30 分钟
            if (System.currentTimeMillis() - authTime.getTime() > maxAuthAge) {
                throw new JWTValidationException("Authentication too old");
            }
        }

        // 4.6 验证 nonce（如果认证请求中发送了 nonce）
        // String expectedNonce = getNonceFromSession();
        // if (expectedNonce != null && !expectedNonce.equals(claims.getStringClaim("nonce"))) {
        //     throw new JWTValidationException("Nonce mismatch");
        // }

        return claims;
    }
}
```

# 3. OIDC 端点体系

## 3.1 核心端点

| 端点 | 用途 | URL 示例 |
|------|------|---------|
| **Authorization** | 发起认证请求 | `GET /authorize` |
| **Token** | 用授权码换取 Token | `POST /token` |
| **UserInfo** | 获取用户详细信息 | `GET /userinfo` |
| **JWKS** | 获取签名公钥 | `GET /.well-known/jwks.json` |
| **Discovery** | 获取所有端点配置 | `GET /.well-known/openid-configuration` |
| **EndSession** | 登出（RP-Initiated Logout） | `GET /logout?id_token_hint=...` |

## 3.2 Discovery（发现服务）

```json
// GET https://auth.example.com/.well-known/openid-configuration
{
    "issuer": "https://auth.example.com",
    "authorization_endpoint": "https://auth.example.com/authorize",
    "token_endpoint": "https://auth.example.com/token",
    "userinfo_endpoint": "https://auth.example.com/userinfo",
    "jwks_uri": "https://auth.example.com/oauth2/jwks",
    "registration_endpoint": "https://auth.example.com/register",
    "end_session_endpoint": "https://auth.example.com/logout",
    "scopes_supported": [
        "openid", "profile", "email", "address", "phone", "offline_access"
    ],
    "response_types_supported": [
        "code", "id_token", "token id_token", "code id_token", "code token",
        "code token id_token"
    ],
    "grant_types_supported": [
        "authorization_code", "implicit", "refresh_token",
        "client_credentials", "password"
    ],
    "subject_types_supported": ["public", "pairwise"],
    "id_token_signing_alg_values_supported": ["RS256", "RS384", "RS512", "ES256"],
    "token_endpoint_auth_methods_supported": [
        "client_secret_basic", "client_secret_post",
        "client_secret_jwt", "private_key_jwt"
    ],
    "claims_supported": [
        "sub", "iss", "aud", "exp", "iat", "auth_time",
        "name", "email", "email_verified", "phone_number"
    ],
    "code_challenge_methods_supported": ["S256"],
    "request_parameter_supported": true,
    "request_uri_parameter_supported": false
}
```

## 3.3 JWKS（JSON Web Key Set）

```json
// GET https://auth.example.com/.well-known/jwks.json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "kid": "2024-01-signing-key",
            "alg": "RS256",
            "n": "0vx7agoebGcQSuuPiLJXZpt...",
            "e": "AQAB",
            "x5c": ["MIIDXTCCAkWgAwIBAg..."]
        },
        {
            "kty": "EC",
            "use": "sig",
            "kid": "2024-01-ec-key",
            "alg": "ES256",
            "crv": "P-256",
            "x": "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
            "y": "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM"
        }
    ]
}
```

# 4. OIDC 认证流程

## 4.1 授权码模式（推荐）

这是 OIDC 中最常用、最安全的流程，实际上就是 OAuth 2.0 授权码模式 + ID Token。

```
GET /authorize?
    response_type=code            ← 授权码模式
    &client_id=my-app
    &redirect_uri=https://app.com/callback
    &scope=openid%20profile%20email  ← openid 必须包含
    &state=随机字符串
    &nonce=随机字符串                 ← OIDC 特有，防重放
    &code_challenge=PKCE
    &code_challenge_method=S256
    &max_age=3600                    ← 最大认证年龄（秒）
    &prompt=login                    ← 交互模式（见下文）

回调：
/callback?code=xxx&state=yyy

Token 请求：
POST /token
    grant_type=authorization_code
    &code=xxx
    &code_verifier=PKCE_verifier
    &client_id=my-app
    &client_secret=xxx

Token 响应：
{
    "access_token": "eyJhbGci...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "8x7d93...",
    "id_token": "eyJhbGci...",       ← OIDC 特有
    "scope": "openid profile email"
}
```

## 4.2 prompt 参数

```java
// prompt 控制用户交互行为

// prompt=none: 静默认证（用户不可见）
// 如果用户已登录 IdP → 直接返回 code
// 如果用户未登录 → 返回错误（不展示登录页）
// 用途：iframe 静默刷新、后台检测登录状态
GET /authorize?prompt=none&...

// prompt=login: 强制重新登录
// 即使用户已有 IdP Session，也展示登录页
// 用途：敏感操作前的重新认证
GET /authorize?prompt=login&...

// prompt=consent: 强制重新授权
// 强制展示授权确认页
GET /authorize?prompt=consent&...

// prompt=select_account: 选择账号
// 展示账号选择页面（如 Google 的多账号切换）
GET /authorize?prompt=select_account&...
```

## 4.3 max_age 参数

```java
// max_age: 指定允许的最大认证年龄（秒）
// 如果用户上次认证时间超过 max_age，IdP 必须要求重新认证

GET /authorize?max_age=3600&...
// → 如果用户 1 小时前认证的 → 需重新登录
// → 如果用户 30 分钟前认证的 → 直接返回 code

// auth_time Claim（在 ID Token 中）:
// 记录用户真实的认证时间，SP/RP 也可以自行验证
```

# 5. UserInfo 端点

## 5.1 请求与响应

```java
// 使用 Access Token 获取用户详细信息
GET /userinfo
Authorization: Bearer eyJhbGci...

// 响应（JSON）：
{
    "sub": "248289761001",
    "name": "张三",
    "given_name": "三",
    "family_name": "张",
    "email": "zhangsan@example.com",
    "email_verified": true,
    "picture": "https://example.com/avatar.jpg"
}

// 也可以获取 JWT 格式的 UserInfo（加密）
GET /userinfo
Accept: application/jwt

// 响应（JWT）：
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIyNDgyODk3NjEwMDEi...
```

## 5.2 ID Token vs UserInfo

| 对比 | ID Token | UserInfo |
|------|---------|----------|
| **获取方式** | Token 端点直接返回 | 额外请求 UserInfo 端点 |
| **格式** | 必须是 JWT | JSON（或 JWT） |
| **签名** | 必须签名 | 可选签名/加密 |
| **内容** | 关键身份信息（sub, iss, aud） | 详细属性（地址、部门等） |
| **使用场景** | 快速获取基本身份 | 获取详细用户属性 |
| **生存期** | 短（通常 5-10 分钟） | 通过 Access Token 随时获取 |

# 6. 登出（RP-Initiated Logout）

```
OIDC Session Management + RP-Initiated Logout:

1. RP-Initiated Logout（主动登出）：
GET /logout?
    id_token_hint=eyJ...        ← ID Token（证明登出的是哪个用户）
    &post_logout_redirect_uri=  ← 登出后跳转到哪里
      https://app.com/goodbye
    &state=xxx                  ← 防 CSRF

IdP 收到后：
- 销毁 IdP Session
- 302 跳转到 post_logout_redirect_uri
```

```java
// Spring Security 中配置 OIDC Logout
@Configuration
@EnableWebSecurity
public class OidcLogoutConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .logout(logout -> logout
                .logoutSuccessHandler(oidcLogoutSuccessHandler())
            )
            .oauth2Login(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public OidcClientInitiatedLogoutSuccessHandler oidcLogoutSuccessHandler() {
        OidcClientInitiatedLogoutSuccessHandler handler =
            new OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository);
        handler.setPostLogoutRedirectUri("https://myapp.com/");
        return handler;
    }
}
```

# 7. Subject 类型：public vs pairwise

```
public sub（默认）:
- 所有 RP 看到的 sub 相同
- 例如：所有应用都看到 sub=248289761001
- 优点：方便跨应用关联用户
- 缺点：隐私问题（可跨应用追踪用户）

pairwise sub:
- 每个 RP 看到不同的 sub
- 例如：App A 看到 sub=abc123，App B 看到 sub=xyz789
- 优点：保护用户隐私，防止跨应用追踪
- 缺点：无法跨应用关联用户

// sub 计算公式（pairwise）：
pairwise_sub = SHA256(sector_identifier + local_sub + salt)
```

# 8. OIDC 安全最佳实践

| 安全措施 | 说明 |
|---------|------|
| **验签 ID Token** | 必须验证 JWT 签名，使用 jwks_uri 中的公钥 |
| **验证 nonce** | Authorization Request 中带 nonce，验证 ID Token 中的 nonce |
| **验证 aud** | 确保 aud 包含自己的 client_id |
| **验证 iss** | 确保 iss 匹配已知的 IdP |
| **验证 exp/iat** | 不接受过期或签发时间超前的 Token |
| **HTTPS 必须** | 所有通信必须使用 HTTPS |
| **PKCE 强制** | 授权码模式必须使用 PKCE |
| **短 TTL** | ID Token 有效期尽量短（5-10 分钟） |
| **密钥轮换** | 定期更换签名密钥，使用 kid 区分 |
| **Sub 选择** | 对外部应用使用 pairwise sub，保护隐私 |
| **max_age** | 敏感操作设置较短 max_age |
| **Token 泄露** | Access Token 不暴露给前端（BFF 模式） |

# 9. OIDC 与纯 OAuth 2.0 的区别

| 维度 | OAuth 2.0 | OIDC |
|------|----------|------|
| **核心目的** | 授权（Authorization） | 认证（Authentication） |
| **身份凭证** | 无标准（需自定义） | ID Token（JWT） 标准化 |
| **用户信息端点** | 无标准 | UserInfo Endpoint 标准化 |
| **发现机制** | 无标准 | Discovery 标准化 |
| **Session 管理** | 无 | Session Management 标准 |
| **登出** | 无 | RP-Initiated Logout |
| **scope** | 自定义 | openid/profie/email... 标准化 |
| **关系** | OAuth 2.0 是基础 | OIDC 是 OAuth 2.0 之上的认证层 |

一句话总结：**OIDC = OAuth 2.0 + ID Token + UserInfo + Discovery + Session Management**
