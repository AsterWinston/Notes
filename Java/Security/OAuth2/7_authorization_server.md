# 1. 授权服务器概述

授权服务器（Authorization Server，AS）是 OAuth2 架构中最复杂的组件，负责整个授权流程的编排、客户端管理、用户认证、Token 颁发与吊销。

## 1.1 授权服务器的核心功能

```
授权服务器的能力矩阵：

┌──────────────────────────────────────────────────────┐
│                   授权服务器                          │
├──────────────────────────────────────────────────────┤
│ 核心功能：                                           │
│  □ 客户端注册与管理（Client Registration）            │
│  □ 用户认证与授权页面（Consent Management）           │
│  □ 授权码生成与验证（Authorization Code）             │
│  □ Token 颁发（Access Token / Refresh Token / ID Token）│
│  □ Token 验证（Introspection）                       │
│  □ Token 吊销（Revocation）                          │
│  □ 密钥管理（JWKS / Signing Keys）                   │
│  □ 审计日志（Audit Trail）                           │
│                                                      │
│ 可选功能：                                           │
│  □ OIDC 支持（ID Token、UserInfo、Logout）            │
│  □ 自适应认证（Adaptive Auth / Step-up Auth）         │
│  □ 多租户（Multi-Tenancy）                           │
│  □ Token Exchange（RFC 8693）                        │
│  □ CIBA（Client-Initiated Backchannel Authentication）│
└──────────────────────────────────────────────────────┘
```

---

# 2. 授权服务器架构设计

## 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      授权服务器                              │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐ │
│  │ 授权端点  │  │ Token   │  │ Introspect│  │ Revoke     │ │
│  │/authorize│  │ /token  │  │ 端点      │  │ 端点       │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘ │
│       │              │             │               │        │
│  ┌────┴──────────────┴─────────────┴───────────────┴──────┐ │
│  │                   核心服务层                            │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │ │
│  │  │ 客户端   │ │ 用户认证 │ │ Token    │ │ 授权管理  │ │ │
│  │  │ 管理     │ │ 服务     │ │ 服务     │ │ (Consent) │ │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   数据存储层                            │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │ │
│  │  │ 客户端   │ │ Token    │ │ 授权记录  │ │ 用户     │ │ │
│  │  │ 配置 DB  │ │ 存储 DB  │ │ DB       │ │ DB      │ │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 2.2 数据库设计

### 2.2.1 客户端配置表

```sql
CREATE TABLE oauth2_clients (
    id VARCHAR(100) PRIMARY KEY,          -- client_id
    client_secret VARCHAR(255) NOT NULL,  -- BCrypt 加密
    client_name VARCHAR(200) NOT NULL,
    client_type ENUM('confidential', 'public') NOT NULL,
    
    -- 安全配置
    redirect_uris TEXT NOT NULL,          -- JSON 数组格式
    grant_types VARCHAR(500) NOT NULL,    -- 空格分隔
    scopes VARCHAR(500),                  -- 授权范围
    token_endpoint_auth_method VARCHAR(50) DEFAULT 'client_secret_basic',
    jwks_uri VARCHAR(500),
    
    -- 过期配置
    access_token_ttl INT DEFAULT 3600,    -- Access Token 有效期（秒）
    refresh_token_ttl INT DEFAULT 2592000, -- Refresh Token 有效期（30 天）
    
    -- 状态
    enabled BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 2.2.2 授权码表

```sql
CREATE TABLE oauth2_authorization_codes (
    code VARCHAR(255) PRIMARY KEY,        -- 授权码（建议 hash 存储）
    client_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(100) NOT NULL,
    redirect_uri VARCHAR(500) NOT NULL,
    scopes VARCHAR(500),
    code_challenge VARCHAR(255),
    code_challenge_method VARCHAR(10),
    nonce VARCHAR(255),
    state VARCHAR(255),
    
    expires_at TIMESTAMP NOT NULL,
    used BOOLEAN DEFAULT FALSE,           -- 防重放
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_client_user (client_id, user_id),
    INDEX idx_expires (expires_at)
);
```

### 2.2.3 Token 存储表

```sql
CREATE TABLE oauth2_tokens (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    token_id VARCHAR(100) UNIQUE NOT NULL,  -- JWT jti 或 Token hash
    token_type ENUM('access_token', 'refresh_token', 'id_token') NOT NULL,
    client_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(100) NOT NULL,
    scopes VARCHAR(500),
    claims JSON,                           -- Token 中包含的 claims
    
    status ENUM('active', 'revoked', 'expired') DEFAULT 'active',
    
    expires_at TIMESTAMP NOT NULL,
    issued_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP NULL,
    
    -- Refresh Token 家族追踪（Rotation 支持）
    refresh_token_family VARCHAR(100),
    
    INDEX idx_user_client (user_id, client_id),
    INDEX idx_status_expires (status, expires_at),
    INDEX idx_token_id (token_id)
);
```

### 2.2.4 用户授权记录表

```sql
CREATE TABLE oauth2_consents (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(100) NOT NULL,
    granted_scopes VARCHAR(500) NOT NULL,
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    
    UNIQUE KEY uk_user_client (user_id, client_id)
);
```

---

# 3. 授权端点核心实现

## 3.1 /authorize 端点实现逻辑

```java
/**
 * 授权端点的完整请求处理流程
 */
@GetMapping("/oauth2/authorize")
public String authorize(
        @RequestParam("response_type") String responseType,
        @RequestParam("client_id") String clientId,
        @RequestParam(value = "redirect_uri", required = false) String redirectUri,
        @RequestParam(value = "scope", required = false) String scope,
        @RequestParam(value = "state", required = false) String state,
        @RequestParam(value = "code_challenge", required = false) String codeChallenge,
        @RequestParam(value = "code_challenge_method", required = false) String codeChallengeMethod,
        @RequestParam(value = "nonce", required = false) String nonce,
        @RequestParam(value = "prompt", required = false) String prompt,
        @RequestParam(value = "max_age", required = false) Long maxAge,
        HttpSession session) {
    
    // 第一步：参数验证
    ValidationResult validation = validator.validate(new AuthorizationRequest()
        .responseType(responseType)
        .clientId(clientId)
        .redirectUri(redirectUri)
        .scope(scope)
    );
    
    if (!validation.isValid()) {
        return buildErrorRedirect(redirectUri, validation.getError(), state);
    }
    
    // 第二步：加载客户端配置
    Client client = clientService.loadByClientId(clientId);
    if (client == null) {
        return buildErrorRedirect(redirectUri, "unauthorized_client", state);
    }
    
    // 第三步：验证 redirect_uri（严格匹配）
    if (redirectUri != null) {
        if (!client.getRedirectUris().contains(redirectUri)) {
            return buildErrorRedirect(null, "invalid_request", state);
        }
    } else if (client.getRedirectUris().size() == 1) {
        redirectUri = client.getRedirectUris().iterator().next();
    } else {
        return buildErrorRedirect(null, "invalid_request", state);
    }
    
    // 第四步：处理 prompt 参数
    if ("none".equals(prompt)) {
        // 静默模式：用户必须已登录，否则返回错误
        if (session.getAttribute("user") == null) {
            return buildErrorRedirect(redirectUri, "login_required", state);
        }
    }
    
    // 第五步：检查用户登录状态
    User user = (User) session.getAttribute("user");
    if (user == null) {
        // 未登录 → 重定向到登录页面
        session.setAttribute("pending_authorization", buildPendingAuthorization());
        return "redirect:/login";
    }
    
    // 第六步：检查是否需要展示授权页面
    if (!"none".equals(prompt) && !"consent".equals(prompt)) {
        Consent consent = consentService.findByUserAndClient(user.getId(), clientId);
        // 如果之前授权过且 scope 未超出，可以跳过授权页面
        if (consent != null && consent.getGrantedScopes().containsAll(requestedScopes)) {
            return issueAuthorizationCode(user, client, redirectUri, scope, state,
                codeChallenge, codeChallengeMethod, nonce);
        }
    }
    
    // 第七步：展示授权页面
    session.setAttribute("pending_auth", new PendingAuthorization(
        clientId, redirectUri, scope, state, codeChallenge, codeChallengeMethod, nonce));
    
    return "oauth2_consent";
}
```

## 3.2 授权码生成

```java
/**
 * 生成安全的 Authorization Code
 */
public class AuthorizationCodeService {
    
    private final SecureRandom secureRandom = new SecureRandom();
    
    /**
     * 生成授权码
     * 
     * 安全要求：
     * 1. 高熵（≥ 128 bit 随机性）
     * 2. 短期有效（≤ 60 秒）
     * 3. 一次性使用
     * 4. 绑定 client_id、redirect_uri、code_challenge
     */
    public String generateAuthorizationCode(AuthorizationCodeContext context) {
        // 生成高熵随机码
        byte[] bytes = new byte[32];  // 256 bit
        secureRandom.nextBytes(bytes);
        String code = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
        
        // 保存授权码信息
        AuthorizationCode authCode = AuthorizationCode.builder()
            .code(hashCode(code))  // 数据库存储 hash，不存原文
            .clientId(context.getClientId())
            .userId(context.getUserId())
            .redirectUri(context.getRedirectUri())
            .scopes(context.getScopes())
            .codeChallenge(context.getCodeChallenge())
            .codeChallengeMethod(context.getCodeChallengeMethod())
            .nonce(context.getNonce())
            .state(context.getState())
            .expiresAt(Instant.now().plusSeconds(60))  // 60 秒有效期
            .build();
        
        codeRepository.save(authCode);
        
        return code;  // 返回原文（通过 URL 传递）
    }
    
    /**
     * 验证授权码并消费（一次性使用）
     */
    public AuthorizationCode consumeAuthorizationCode(
            String code, String clientId, String redirectUri, String codeVerifier) {
        
        String codeHash = hashCode(code);
        AuthorizationCode authCode = codeRepository.findByCodeHash(codeHash)
            .orElseThrow(() -> new InvalidGrantException("Invalid authorization code"));
        
        // 验证过期
        if (authCode.getExpiresAt().isBefore(Instant.now())) {
            codeRepository.delete(authCode);
            throw new InvalidGrantException("Authorization code expired");
        }
        
        // 验证已使用（防重放）
        if (authCode.isUsed()) {
            // 检测到重放！可能是攻击 → 吊销该用户所有 Token + 告警
            revokeAllUserTokens(authCode.getUserId());
            throw new InvalidGrantException("Authorization code already used");
        }
        
        // 验证 client_id
        if (!authCode.getClientId().equals(clientId)) {
            throw new InvalidGrantException("Client mismatch");
        }
        
        // 验证 redirect_uri（严格匹配）
        if (!authCode.getRedirectUri().equals(redirectUri)) {
            throw new InvalidGrantException("Redirect URI mismatch");
        }
        
        // 验证 PKCE
        if (authCode.getCodeChallenge() != null) {
            verifyPkce(authCode.getCodeChallenge(), authCode.getCodeChallengeMethod(), codeVerifier);
        }
        
        // 标记为已使用
        codeRepository.markAsUsed(authCode.getId());
        
        return authCode;
    }
    
    private void verifyPkce(String expectedChallenge, String method, String codeVerifier) {
        String actualChallenge;
        if ("S256".equals(method)) {
            actualChallenge = sha256Base64Url(codeVerifier);
        } else if ("plain".equals(method)) {
            actualChallenge = codeVerifier;
        } else {
            throw new InvalidGrantException("Unsupported code challenge method");
        }
        
        if (!MessageDigest.isEqual(
                actualChallenge.getBytes(StandardCharsets.US_ASCII),
                expectedChallenge.getBytes(StandardCharsets.US_ASCII))) {
            throw new InvalidGrantException("PKCE verification failed");
        }
    }
}
```

---

# 4. Token 颁发服务

## 4.1 Access Token 生成（JWT 格式）

```java
/**
 * JWT Access Token 生成服务
 */
@Service
public class JwtTokenService {
    
    private final JWSSigner signer;
    private final String issuer;
    private final int accessTokenTtl;
    
    /**
     * 生成 JWT Access Token
     */
    public TokenResponse generateAccessToken(TokenContext context) {
        Instant now = Instant.now();
        Instant expiresAt = now.plusSeconds(context.getAccessTokenTtl());
        
        // 构建 JWT Claims
        JWTClaimsSet.Builder claimsBuilder = new JWTClaimsSet.Builder()
            .issuer(issuer)
            .subject(context.getUserId())       // sub = 用户 ID
            .audience(context.getAudience())     // aud = 资源服务器标识
            .issueTime(Date.from(now))
            .expirationTime(Date.from(expiresAt))
            .notBeforeTime(Date.from(now))
            .jwtID(UUID.randomUUID().toString()) // jti = 唯一 Token ID
            .claim("client_id", context.getClientId())
            .claim("scope", context.getScopes());
        
        // 如果配置了，加入额外的声明
        if (context.getAuthorities() != null) {
            claimsBuilder.claim("authorities", context.getAuthorities());
        }
        
        JWTClaimsSet claims = claimsBuilder.build();
        
        // 构建 JWS（签名的 JWT）
        SignedJWT signedJWT = new SignedJWT(
            new JWSHeader.Builder(JWSAlgorithm.RS256)
                .keyID(getCurrentKeyId())
                .type(JOSEObjectType.JWT)
                .build(),
            claims
        );
        
        // 签名
        signedJWT.sign(signer);
        
        String accessToken = signedJWT.serialize();
        
        // 存储 Token 元数据（用于吊销和审计）
        tokenRepository.save(TokenRecord.builder()
            .tokenId(claims.getJWTID())
            .tokenType("access_token")
            .clientId(context.getClientId())
            .userId(context.getUserId())
            .scopes(context.getScopes())
            .expiresAt(expiresAt)
            .status("active")
            .build()
        );
        
        return TokenResponse.builder()
            .accessToken(accessToken)
            .tokenType("Bearer")
            .expiresIn(context.getAccessTokenTtl())
            .scope(context.getScopes())
            .build();
    }
}
```

## 4.2 Refresh Token 生成与管理

```java
/**
 * Refresh Token 生成与轮换服务
 */
@Service
public class RefreshTokenService {
    
    private final SecureRandom secureRandom = new SecureRandom();
    
    /**
     * 生成 Refresh Token
     * 格式：不透明字符串（高熵随机，不可预测）
     */
    public String generateRefreshToken(TokenContext context) {
        byte[] bytes = new byte[64];  // 512 bit 熵
        secureRandom.nextBytes(bytes);
        String refreshToken = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
        String refreshTokenHash = sha256(refreshToken);
        
        Instant expiresAt = Instant.now().plusSeconds(context.getRefreshTokenTtl());
        
        // 存储 Refresh Token 元数据
        tokenRepository.save(RefreshTokenRecord.builder()
            .tokenHash(refreshTokenHash)
            .tokenType("refresh_token")
            .clientId(context.getClientId())
            .userId(context.getUserId())
            .scopes(context.getScopes())
            .family(context.getRefreshTokenFamily())  // 家族标识（用于旋转检测）
            .expiresAt(expiresAt)
            .status("active")
            .build()
        );
        
        return refreshToken;  // 返回原文（只传输一次）
    }
    
    /**
     * Refresh Token 轮换
     * 
     * 安全要求（OAuth 2.1）：
     * 1. 每次使用后旧 Token 立即失效
     * 2. 颁发新的 Refresh Token
     * 3. 检测到旧 Token 被重用 → 整个家族失效
     */
    public TokenResponse rotateRefreshToken(String refreshToken) {
        String tokenHash = sha256(refreshToken);
        RefreshTokenRecord oldToken = tokenRepository.findByTokenHash(tokenHash)
            .orElseThrow(() -> new InvalidGrantException("Invalid refresh token"));
        
        // 检测重放：如果 Token 已过期或被撤销，但有人还在用
        if (!"active".equals(oldToken.getStatus())) {
            // Token 已经被使用过 → 这是重放攻击！
            // 撤销整个 Refresh Token 家族
            revokeTokenFamily(oldToken.getFamily());
            // 发送安全告警
            securityAlertService.reportReplayAttack(
                oldToken.getClientId(), oldToken.getUserId(), oldToken.getFamily());
            throw new InvalidGrantException("Refresh token reuse detected");
        }
        
        // 撤销旧 Refresh Token
        tokenRepository.revokeByTokenHash(tokenHash);
        
        // 生成新的 Token 对
        TokenContext newContext = TokenContext.from(oldToken);
        newContext.setRefreshTokenFamily(oldToken.getFamily());  // 保持家族不变
        
        String newAccessToken = accessTokenService.generate(newContext);
        String newRefreshToken = generateRefreshToken(newContext);
        
        return TokenResponse.builder()
            .accessToken(newAccessToken)
            .refreshToken(newRefreshToken)
            .tokenType("Bearer")
            .expiresIn(accessTokenTtl)
            .build();
    }
    
    /**
     * 撤销整个 Refresh Token 家族
     */
    private void revokeTokenFamily(String family) {
        tokenRepository.revokeByFamily(family);
        // 也撤销该家族颁发的所有 Access Token
        tokenRepository.revokeAccessTokensByFamily(family);
    }
}
```

---

# 5. 密钥管理

## 5.1 JWKS (JSON Web Key Set) 规范

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "signing-key-2024-01",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZ...",
      "e": "AQAB"
    },
    {
      "kty": "EC",
      "kid": "signing-key-2024-02",
      "use": "sig",
      "alg": "ES256",
      "crv": "P-256",
      "x": "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
      "y": "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM"
    }
  ]
}
```

## 5.2 密钥轮换（Key Rotation）

```
密钥轮换策略：

1. 时间线：
   ┌─────────────┬──────────────┬──────────────┬──────────────┐
   │ key-2024-Q1 │  key-2024-Q2 │  key-2024-Q3 │  key-2024-Q4 │
   │─────────────│──────────────│──────────────│──────────────│
   │   活跃签名   │   活跃签名    │    验证用     │    验证用     │
   │   (sign)    │   (sign)     │   (verify)   │   (verify)   │
   └─────────────┴──────────────┴──────────────┴──────────────┘
                               ↑ 当前时间

2. JWKS 响应同时包含：
   - 当前签名密钥（kid=sig-2024-Q2）
   - 上一季度密钥（kid=sig-2024-Q1）— 验证未过期 Token
   - 下一季度密钥（kid=sig-2024-Q3）— 预发布，资源服务器可提前缓存

3. 轮换步骤：
   a. 生成新密钥对 → 在 JWKS 中发布（仅验证，不签名）
   b. 等待资源服务器拉取新 JWKS（25min 缓存期）
   c. 切换到新密钥签名
   d. 24h 后从 JWKS 中移除旧密钥
```

```java
/**
 * 密钥管理服务
 */
@Service
public class KeyManagementService {
    
    private final Map<String, SigningKey> signingKeys = new ConcurrentHashMap<>();
    
    /**
     * JWKS 端点响应
     */
    public JWKSet getJwks() {
        List<JWK> keys = new ArrayList<>();
        
        // 当前签名密钥
        signingKeys.forEach((kid, signingKey) -> {
            if (signingKey.isActive()) {
                keys.add(signingKey.toPublicJWK());
            }
        });
        
        // 最近退役的密钥（仍用于验证）
        recentRetiredKeys.forEach((kid, key) -> {
            keys.add(key.toPublicJWK());
        });
        
        // 预发布的未来密钥
        upcomingKeys.forEach((kid, key) -> {
            keys.add(key.toPublicJWK());
        });
        
        return new JWKSet(keys);
    }
}
```

---

# 6. 授权管理（Consent）

## 6.1 授权页面设计

```
授权页面（Consent Page）设计要点：

1. 清晰展示请求的权限
   ┌──────────────────────────────────────────┐
   │  "My Calendar App" 请求访问你的日历：       │
   │  ─────────────────────────────────────── │
   │  ✓ 查看你的日历事件                        │
   │  ✓ 查看你的邮箱地址                        │
   │  ✗ 修改你的日历（未请求）                   │
   │  ✗ 删除事件（未请求）                       │
   │                                          │
   │  以你的身份登录: alice@example.com          │
   │                                          │
   │  [拒绝]                    [允许]          │
   └──────────────────────────────────────────┘

2. 记录授权决策
   - 用户点击"允许" → 保存授权记录 + 颁发授权码
   - 用户点击"拒绝" → 返回 error=access_denied
   
3. 授权有效期
   - 可设置 "记住此决定" 或定期重新确认
   - 授权记录存储在 oauth2_consents 表

4. 用户可随时在授权服务器的设置页面撤销已授权的应用
```

---

# 7. 安全实现要点

## 7.1 授权码安全

```
授权码安全要求清单：

□ 使用高熵随机值（≥ 128 bit）
□ 短期有效（≤ 60 秒）
□ 一次性使用（用后立即失效）
□ 数据库存储 hash 值，不存原文
□ 绑定 client_id、redirect_uri、code_challenge
□ 启用 PKCE（S256 模式）
□ 检测重放 → 撤销所有 Token + 安全告警
□ 定期清理过期的授权码
```

## 7.2 客户端认证安全

```
client_secret 存储：
□ 使用 BCrypt / Argon2 哈希存储（不是明文）
□ 支持 client_secret 轮换
□ 提供 client_secret 到期机制

private_key_jwt 认证：
□ 验证 JWT 签名
□ 验证 JWT 有效期（短有效期，如 60 秒）
□ 验证 JWT 中包含正确的 iss（client_id）和 aud（Token Endpoint URL）
□ 使用 jti 防重放（缓存已使用的 jti）
```

## 7.3 安全日志与监控

```
必须记录的安全事件：

授权端点：
  - /authorize 请求（包含 client_id, scope, redirect_uri）
  - 用户登录/登出
  - 用户同意/拒绝授权
  - 授权码颁发/使用/重放检测

Token 端点：
  - Token 请求（grant_type, client_id）
  - 认证失败（无效 client_id / client_secret）
  - Token 颁发（Token ID，非完整 Token）
  - Refresh Token 轮换/重放检测

吊销端点：
  - Token 吊销请求（谁吊销了什么 Token）

异常检测指标：
  - 短时间内大量 Token 请求（暴力攻击）
  - 同一用户从不同地理位置登录
  - 授权码重放尝试
  - Refresh Token 重用尝试
```
