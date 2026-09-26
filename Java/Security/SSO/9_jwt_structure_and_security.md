
# 1. 概述

**JWT（JSON Web Token）** 是一种紧凑、URL 安全的令牌格式，用于在各方之间传输声明（Claims）。JWT 是现代 SSO 和无状态认证的核心技术，也是 OIDC 中 ID Token 的标准格式。

```
JWT = Header.Payload.Signature

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9       ← Base64URL(Header)
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IuW8oOS4iSIsImlhdCI6MTUxNjIzOTAyMn0  ← Base64URL(Payload)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c    ← Base64URL(Signature)
```

# 2. JWT 结构详解

## 2.1 Header（头部）

```json
{
    "alg": "RS256",        // 签名算法（必须）
    "typ": "JWT",          // 令牌类型（推荐）
    "kid": "key-2024-01",  // 密钥标识（用于密钥轮换）
    "cty": "JWT"           // 内容类型（嵌套 JWT 时使用）
}
```

## 2.2 Payload（载荷）

JWT 的 Claims 分为三类：

```json
{
    // ========== 注册声明（Registered Claims） ==========
    "iss": "https://auth.example.com",    // Issuer - 签发者
    "sub": "user-12345",                  // Subject - 主题（用户唯一标识）
    "aud": ["app-a", "app-b"],            // Audience - 受众
    "exp": 1712345678,                    // Expiration Time - 过期时间
    "nbf": 1712341978,                    // Not Before - 生效时间
    "iat": 1712341978,                    // Issued At - 签发时间
    "jti": "unique-token-id-001",         // JWT ID - 唯一标识（防重放）

    // ========== 公开声明（Public Claims） ==========
    "name": "张三",
    "email": "zhangsan@example.com",
    "roles": ["ROLE_USER", "ROLE_ADMIN"],

    // ========== 私有声明（Private Claims） ==========
    "department": "技术部",
    "employee_id": "EMP-001",
    "tenant_id": "tenant-abc"
}
```

## 2.3 Signature（签名）

```
签名 = HMAC-SHA256(Secret, Header.Base64URL + "." + Payload.Base64URL)
或
签名 = RSA-SHA256(PrivateKey, Header.Base64URL + "." + Payload.Base64URL)

签名的两个作用：
1. 完整性：Payload 被修改 → 签名不匹配 → 拒绝
2. 认证来源：只有持有密钥的签发者才能生成有效签名
```

# 3. JWT 签名算法

## 3.1 HMAC 对称签名

```java
// HMAC 使用共享密钥（对称加密）
// 签发和验证使用同一个 secret

public class HmacJwtExample {

    private static final SecretKey SECRET_KEY =
        Keys.hmacShaKeyFor("my-very-long-secret-key-at-least-256-bits".getBytes());

    // 签发 JWT
    public String createToken(String username) {
        return Jwts.builder()
            .subject(username)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 3600_000))
            .signWith(SECRET_KEY)  // HS256
            .compact();
    }

    // 验证 JWT
    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(SECRET_KEY)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}

// HMAC 风险：
// - 密钥必须在签发方和验证方之间共享
// - 任何持有密钥的方都可以签发任意 Token
// - 密钥泄露 → 所有 Token 可被伪造
// - 不适合多服务（密钥共享风险大）
```

## 3.2 RSA 非对称签名

```java
// RSA 使用公私钥对（非对称加密）
// 签发方持有私钥 → 生成签名
// 验证方持有公钥 → 验证签名（公钥可公开）

public class RsaJwtExample {

    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;

    // 签发 JWT（使用私钥）
    public String createToken(String username) {
        return Jwts.builder()
            .subject(username)
            .issuer("https://auth.example.com")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 3600_000))
            .header().keyId("key-2024-01").and()  // kid 用于密钥轮换
            .signWith(privateKey)                  // RS256
            .compact();
    }

    // 验证 JWT（使用公钥）
    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(publicKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}

// RSA 优势：
// - 公钥可安全分发（JWKS 端点）
// - 任何服务都可以用公钥验证（无需共享密钥）
// - 私钥泄露 → 仅影响新 Token
// - 适合微服务 / 多服务架构
```

## 3.3 算法对比

| 算法 | 类型 | 密钥大小 | 签名大小 | 性能 | 使用场景 |
|------|------|---------|---------|------|---------|
| **HS256** | HMAC + SHA-256 | >= 256 bit | 32 bytes | 快 | 单服务 / 简单场景 |
| **HS384** | HMAC + SHA-384 | >= 384 bit | 48 bytes | 中 | |
| **HS512** | HMAC + SHA-512 | >= 512 bit | 64 bytes | 慢 | |
| **RS256** | RSA + SHA-256 | 2048 bit | 256 bytes | 慢（验签快） | **推荐**：微服务 / IdP |
| **RS384** | RSA + SHA-384 | 3072 bit | 384 bytes | 较慢 | |
| **RS512** | RSA + SHA-512 | 4096 bit | 512 bytes | 很慢 | |
| **ES256** | ECDSA + P-256 | 256 bit | 64 bytes | 快 | 移动端 / IoT |
| **ES384** | ECDSA + P-384 | 384 bit | 96 bytes | 快 | |
| **ES512** | ECDSA + P-521 | 521 bit | 132 bytes | 快 | |
| **EdDSA** | Ed25519 | 256 bit | 64 bytes | 最快 | 现代应用推荐 |
| **PS256** | RSA-PSS | 2048 bit | 256 bytes | 中 | 更高安全要求 |

**推荐策略：RS256（通用）或 ES256（移动/IoT），内部服务可用 EdDSA。**

# 4. JWE（加密）

## 4.1 JWE vs JWS

```
JWS（JSON Web Signature）= 签名 → 防篡改，但是 Payload 明文可见
JWE（JSON Web Encryption）= 加密 → 防窥探，Payload 加密

JWE 结构：
Header.EncryptedKey.IV.Ciphertext.AuthenticationTag

当 JWT 包含敏感信息时（如身份证号、银行卡号），应使用 JWE 加密。
```

## 4.2 JWE 加密示例

```java
public class JweExample {

    private final RSAKey rsaKey;

    /**
     * 使用 RSA 公钥加密 JWT（生成 JWE）
     */
    public String encryptJwt(String claims) throws Exception {
        // 创建 JWE Header
        JWEHeader header = new JWEHeader.Builder(JWEAlgorithm.RSA_OAEP_256, EncryptionMethod.A256GCM)
            .contentType("JWT")
            .build();

        // 创建 JWE 对象
        JWEObject jweObject = new JWEObject(header, new Payload(claims));

        // 使用公钥加密
        RSAEncrypter encrypter = new RSAEncrypter(rsaKey.toRSAPublicKey());
        jweObject.encrypt(encrypter);

        return jweObject.serialize();
    }

    /**
     * 使用 RSA 私钥解密 JWE
     */
    public String decryptJwt(String encryptedJwt) throws Exception {
        JWEObject jweObject = JWEObject.parse(encryptedJwt);

        RSADecrypter decrypter = new RSADecrypter(rsaKey);
        jweObject.decrypt(decrypter);

        return jweObject.getPayload().toString();
    }
}
```

## 4.3 嵌套 JWT（签名 + 加密）

```
步骤：
1. 先对 Claims 签名 → 生成 JWS
2. 再用 JWE 加密整个 JWS → 生成嵌套 JWT

好处：同时实现防篡改（签名）和防窥探（加密）
```

# 5. JWK（JSON Web Key）

## 5.1 JWK 格式

```json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "kid": "2024-01-15-signing-key",
            "alg": "RS256",
            "n": "0vx7agoebGcQSuuPiLJXZpt...（Base64URL 编码的模数）",
            "e": "AQAB",
            "x5c": ["MIIDXTCCAkWgAwIBAg..."],
            "x5t": "abc123..."
        },
        {
            "kty": "EC",
            "use": "enc",
            "kid": "2024-01-15-encryption-key",
            "alg": "ECDH-ES+A256KW",
            "crv": "P-256",
            "x": "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
            "y": "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM"
        }
    ]
}
```

## 5.2 JWK 密钥轮换

```java
@Component
public class JwkKeyRotationManager {

    private final Map<String, RSAKey> activeKeys = new ConcurrentHashMap<>();

    /**
     * 定期轮换密钥（如每 30 天）
     */
    @Scheduled(cron = "0 0 2 1 * ?")  // 每月1号凌晨2点
    public void rotateKeys() {
        // 1. 生成新密钥对
        RSAKey newKey = generateNewRsaKey();
        String newKid = "signing-key-" + Instant.now();

        // 2. 将现有活跃密钥变为"退役"密钥
        for (String kid : activeKeys.keySet()) {
            retiredKeys.put(kid, activeKeys.get(kid));
        }

        // 3. 新密钥设为活跃
        activeKeys.clear();
        activeKeys.put(newKid, newKey);

        // 4. 更新 JWKS 端点
        updateJwksEndpoint();
    }

    /**
     * 验证 Token 时，根据 kid 选择对应密钥
     */
    public RSAKey getKeyByKid(String kid) {
        RSAKey key = activeKeys.get(kid);
        if (key != null) return key;

        key = retiredKeys.get(kid);
        if (key != null && !isExpired(key)) return key;

        throw new InvalidKeyException("Unknown kid: " + kid);
    }

    // JWKS 端点返回活跃 + 退役（未过期）的密钥
    public JWKSet getJwkSet() {
        List<JWK> keys = new ArrayList<>();
        keys.addAll(activeKeys.values());
        keys.addAll(getValidRetiredKeys());
        return new JWKSet(keys);
    }
}
```

# 6. JWT 在 Java 中的实现

## 6.1 使用 jjwt 库

```java
// 依赖：io.jsonwebtoken:jjwt-api:0.12.x
//       io.jsonwebtoken:jjwt-impl:0.12.x
//       io.jsonwebtoken:jjwt-jackson:0.12.x

public class JwtService {

    private final SecretKey key = Jwts.SIG.HS256.key().build();

    /**
     * 创建 Access Token
     */
    public String createAccessToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .claim("type", "access")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000)) // 15分钟
            .id(UUID.randomUUID().toString())  // jti 防重放
            .signWith(key)
            .compact();
    }

    /**
     * 创建 Refresh Token
     */
    public String createRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("type", "refresh")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 7 * 24 * 3600_000)) // 7天
            .id(UUID.randomUUID().toString())
            .signWith(key)
            .compact();
    }

    /**
     * 解析并验证 Token
     */
    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    /**
     * 验证 Token 是否有效
     */
    public boolean isTokenValid(String token, UserDetails userDetails) {
        try {
            Claims claims = parseToken(token);
            String username = claims.getSubject();
            return username.equals(userDetails.getUsername())
                && !claims.getExpiration().before(new Date());
        } catch (JwtException e) {
            return false;
        }
    }
}
```

## 6.2 Spring Security JWT Filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtService jwtService;

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        // 1. 从请求头提取 Token
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String jwt = authHeader.substring(7);

        // 2. 解析用户名
        String username;
        try {
            username = jwtService.parseToken(jwt).getSubject();
        } catch (JwtException e) {
            filterChain.doFilter(request, response);
            return;
        }

        // 3. 检查是否已认证
        if (username != null
                && SecurityContextHolder.getContext().getAuthentication() == null) {

            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            // 4. 验证 Token
            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

# 7. JWT 安全最佳实践

## 7.1 安全措施清单

| 措施 | 说明 | 风险等级 |
|------|------|---------|
| **HTTPS 必须** | Token 通过 HTTP 明文传输 = 裸奔 | 🔴 致命 |
| **短有效期** | Access Token 5-15 分钟，Refresh Token 1-7 天 | 🟠 高 |
| **非对称签名** | 用 RS256/ES256 替代 HS256（多服务场景） | 🟠 高 |
| **验证所有 Claim** | iss/aud/exp/nbf 必须全部验证 | 🟠 高 |
| **不存敏感信息** | JWT Payload 是 Base64URL（可解码），不存密码/信用卡号 | 🟠 高 |
| **使用 JWE 加密** | 如果确实需要存敏感字段，用 JWE 加密 | 🟡 中 |
| **jti 防重放** | 每个 Token 带唯一 jti，服务端维护已使用的 jti 列表 | 🟡 中 |
| **密钥轮换** | 定期更换签名密钥，使用 kid 区分 | 🟡 中 |
| **Token 黑名单** | 登出/密码修改时，将 Token jti 加入黑名单 | 🟡 中 |
| **限制 Payload 大小** | 避免 Token 过大（影响 HTTP Header 性能） | 🟢 低 |
| **不存 localStorage** | SPA 中不存 localStorage（XSS 可读） | 🟠 高 |

## 7.2 常见漏洞与防御

```java
// ========== 漏洞 1：alg=none 攻击 ==========
// 攻击者修改 Header 中的 alg 为 "none"，Payload 不变，移除签名
// 如果验证方不检查 alg=none → 任何 Payload 都通过

// 防御：
Jwts.parser()
    .verifyWith(key)
    .build()
    .parseSignedClaims(token);  // parseSignedClaims 会拒绝 unsigned token

// ========== 漏洞 2：HMAC/RSA 混淆攻击 ==========
// 如果验证方同时支持 HS256 和 RS256：
// 攻击者用 RS256 公钥（公开）作为 HMAC 密钥
// → 生成 RS256 签名的 Token，但声称是 HS256
// → 验证方用公钥（当作 HMAC 密钥）验证 → 通过！

// 防御：固定使用的算法
Jwts.parser()
    .verifyWith(publicKey)      // 指定公钥（仅支持 RSA）
    .build()
    .parseSignedClaims(token);

// ========== 漏洞 3：kid 注入 ==========
// 如果 kid 来自用户输入或 URL 参数，攻击者可能注入恶意 kid
// → 服务端加载攻击者控制的密钥

// 防御：
// 1. kid 只能是服务端已知的密钥标识符
// 2. 不要从用户输入中提取 kid
// 3. 使用白名单验证 kid

// ========== 漏洞 4：过期 Token 未拒绝 ==========
// 必须验证 exp Claim，不允许过期 Token
Jwts.parser()
    .requireExpiration(new Date())  // 要求有 exp 且未过期
    .verifyWith(key)
    .build()
    .parseSignedClaims(token);
```

# 8. JWT vs Session

| 维度 | Session（传统） | JWT（无状态） |
|------|----------------|--------------|
| **状态存储** | 服务端存储 | 客户端存储（Token中） |
| **扩展性** | 需要 Redis 等共享存储 | 天然水平扩展 |
| **吊销** | 直接删除 Session（即时） | 需黑名单（有延迟） |
| **大小** | Cookie 仅存 SessionID（< 1KB） | Token 通常 1-5KB |
| **服务器负载** | 每次请求查 Session | 每次请求验签（CPU） |
| **跨域** | Cookie 受限 | HTTP Header，天然跨域 |
| **CSRF** | 需防护 | Bearer Token 天然防 CSRF |
| **XSS** | HttpOnly Cookie 防护 | localStorage 易被 XSS 窃取 |
| **移动端** | Cookie 支持弱 | 天然支持 |

# 9. JWT 最佳使用模式

```java
// ========== 推荐模式：双 Token 机制 ==========

// Access Token（短期，5-15分钟）
// + Refresh Token（长期，7-30天）

// 完整流程：
public class JwtAuthFlow {

    // 1. 登录 → 返回双 Token
    public LoginResponse login(String username, String password) {
        UserDetails user = authenticate(username, password);

        String accessToken = jwtService.createAccessToken(user);    // 15 分钟
        String refreshToken = jwtService.createRefreshToken(user);  // 7 天

        // Refresh Token 存 Redis（用于吊销和刷新）
        redisTemplate.opsForValue().set(
            "refresh:" + refreshToken,
            user.getUsername(),
            Duration.ofDays(7)
        );

        return new LoginResponse(accessToken, refreshToken);
    }

    // 2. Access Token 过期 → 用 Refresh Token 刷新
    public LoginResponse refresh(String refreshToken) {
        // 验证 Refresh Token
        Claims claims = jwtService.parseToken(refreshToken);
        String username = claims.getSubject();

        // 检查 Refresh Token 是否在 Redis 中（未被吊销）
        String storedUsername = (String) redisTemplate.opsForValue()
            .get("refresh:" + refreshToken);
        if (storedUsername == null || !storedUsername.equals(username)) {
            throw new SecurityException("Refresh token revoked");
        }

        // 吊销旧 Refresh Token
        redisTemplate.delete("refresh:" + refreshToken);

        // 发新 Token
        UserDetails user = userDetailsService.loadUserByUsername(username);
        return login(username, null);  // 返回新的双 Token
    }

    // 3. 登出 → 吊销 Refresh Token + Access Token 加入黑名单
    public void logout(String accessToken, String refreshToken) {
        // 吊销 Refresh Token
        redisTemplate.delete("refresh:" + refreshToken);

        // Access Token 加入黑名单（jti → 过期时间）
        Claims claims = jwtService.parseToken(accessToken);
        String jti = claims.getId();
        Date expiration = claims.getExpiration();
        long ttl = expiration.getTime() - System.currentTimeMillis();

        if (ttl > 0) {
            redisTemplate.opsForValue().set(
                "jti_blacklist:" + jti, "revoked", Duration.ofMillis(ttl));
        }
    }
}
```

# 10. 总结

| 要点 | 说明 |
|------|------|
| **使用 RS256/ES256** | 非对称签名，适合多服务和微服务 |
| **短有效期** | Access Token 5-15 分钟 |
| **双 Token 模式** | Access Token（短期） + Refresh Token（长期） |
| **Refresh Token 存 Redis** | 支持吊销和泄露检测 |
| **HTTPS 强制** | 所有 Token 传输必须 HTTPS |
| **不存敏感数据** | JWT Payload 不是加密的，只是 Base64URL 编码 |
| **验证所有 Claim** | iss/aud/exp/nbf 一个不漏 |
| **密钥轮换** | 定期更换密钥，使用 kid 区分 |
| **防算法攻击** | 固定签名算法，不接受 alg=none |
