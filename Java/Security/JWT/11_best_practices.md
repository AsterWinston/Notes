# 1. JWT 最佳实践总览

JWT 正确实施需要贯穿**设计、开发、部署、运维**全生命周期的实践指导。本章汇总了业界公认的最佳实践，涵盖安全、性能、架构等维度。

---

# 2. 密钥管理最佳实践

## 2.1 密钥生成

```java
// 安全生成 HMAC 密钥（HS256）
SecretKey key = Jwts.SIG.HS256.key().build();
// 等价于：使用 SecureRandom 生成 256 位随机字节

// 安全生成 RSA 密钥
KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
keyGen.initialize(2048); // 或 3072
KeyPair keyPair = keyGen.generateKeyPair();

// 安全生成 EC 密钥
KeyPairGenerator ecGen = KeyPairGenerator.getInstance("EC");
ecGen.initialize(new ECGenParameterSpec("secp256r1"));
KeyPair ecPair = ecGen.generateKeyPair();

// 安全生成 EdDSA（Ed25519）密钥（Java 15+）
KeyPairGenerator edGen = KeyPairGenerator.getInstance("Ed25519");
KeyPair edPair = edGen.generateKeyPair();
```

## 2.2 黄金法则

```
1. 不同环境使用不同密钥（开发 ≠ 测试 ≠ 生产）
2. 不同用途使用不同密钥（签名 ≠ 加密）
3. 不同算法使用不同密钥（HMAC ≠ RSA）
4. 定期轮换（≤ 90 天）
5. 密钥绝不硬编码在代码中
6. 密钥绝不提交到版本控制系统
7. 密钥存储于安全位置（Vault、KMS、HSM）
8. 密钥访问有审计日志
```

## 2.3 密钥轮换方案

```java
public class KeyRotationManager {
    
    // 当前活跃的密钥
    private volatile SigningKey currentKey;
    
    // 最近过期的密钥（仍用于验证旧 Token）
    private final List<SigningKey> recentKeys;
    
    // 定时任务：轮换密钥
    @Scheduled(cron = "0 0 2 ? * 1") // 每周一凌晨 2 点
    public void rotateKeys() {
        // 1. 生成新密钥
        SigningKey newKey = generateNewKey("key-rotation-" + now());
        
        // 2. 将当前密钥移入历史列表
        recentKeys.add(currentKey);
        
        // 3. 更新当前密钥
        currentKey = newKey;
        
        // 4. 清理超过 TTL 的历史密钥
        recentKeys.removeIf(key -> 
            key.createdAt().plus(7, ChronoUnit.DAYS).isBefore(Instant.now()));
        
        log.info("Key rotated. Active key: {}", currentKey.kid());
    }
    
    // 验证时：先查当前密钥，再查历史密钥
    public Key resolveVerificationKey(String kid) {
        if (currentKey.kid().equals(kid)) {
            return currentKey.verificationKey();
        }
        
        return recentKeys.stream()
            .filter(k -> k.kid().equals(kid))
            .findFirst()
            .orElseThrow(() -> new SecurityException("Unknown key: " + kid))
            .verificationKey();
    }
}

record SigningKey(
    String kid,
    Instant createdAt,
    PrivateKey signingKey,
    PublicKey verificationKey
) {}
```

---

# 3. 令牌生命周期最佳实践

## 3.1 有效期设置

```java
// 推荐的 Token 生命周期配置
public class TokenLifetimeConfig {
    
    // Access Token：短有效期
    public static final long ACCESS_TOKEN_MS = Duration.ofMinutes(15).toMillis();
    
    // Refresh Token：用于获取新 Access Token
    public static final long REFRESH_TOKEN_MS = Duration.ofDays(7).toMillis();
    
    // ID Token：一次性身份断言
    public static final long ID_TOKEN_MS = Duration.ofMinutes(10).toMillis();
    
    // 操作令牌（支付确认等高风险操作）
    public static final long OPERATION_TOKEN_MS = Duration.ofMinutes(5).toMillis();
    
    // 邮件验证链接
    public static final long VERIFICATION_TOKEN_MS = Duration.ofHours(24).toMillis();
}
```

## 3.2 令牌过期处理策略

```
Access Token 过期：
  → 客户端发起 Refresh Token 请求
  → 服务端验证 Refresh Token
  → 服务端签发新 Access Token + 新 Refresh Token（轮换）
  → 客户端使用新 Token

Refresh Token 过期/吊销：
  → 客户端引导用户重新登录

特殊场景：
  → 密码修改：吊销所有 Refresh Token（递增版本号）
  → 账号删除：吊销所有 Token
  → 异常行为检测：临时吊销 + 通知
```

## 3.3 令牌提前过期（Proactive Expiration）

```java
// 配合业务规则动态缩短有效期
public class DynamicExpiration {
    
    public long calculateExpiration(User user, HttpServletRequest request) {
        long baseExpiration = 15 * 60 * 1000; // 基准 15 分钟
        
        // 1. 高风险操作缩短有效期
        if (isHighRiskEndpoint(request)) {
            return 5 * 60 * 1000; // 5 分钟
        }
        
        // 2. 新设备缩短有效期
        if (isNewDevice(request)) {
            return 10 * 60 * 1000; // 10 分钟
        }
        
        // 3. 管理员延长有效期（可选）
        if (user.hasRole("ADMIN") && request.isSecure()) {
            return 30 * 60 * 1000; // 30 分钟
        }
        
        return baseExpiration;
    }
}
```

---

# 4. Claims 设计最佳实践

## 4.1 Payload 精简原则

```json
// 好的实践：精简
{
  "sub": "usr_abc123",
  "iss": "https://auth.example.com",
  "iat": 1712345600,
  "exp": 1712346500,
  "scope": "api:read api:write",
  "tenant_id": "t-001"
}

// 不好的实践：臃肿
{
  "sub": "usr_abc123",
  "name": "Johnathan William Doe",
  "email": "john@example.com",
  "department": "Engineering - Backend Infrastructure Team",
  "manager": "usr_xyz789",
  "office_location": "Building A-12, Floor 3, Desk 42",
  "all_permissions": ["perm:1", "perm:2", ..., "perm:500"],
  "preferences": { "theme": "dark", "lang": "en", "timezone": "UTC" }
}
```

## 4.2 Claims 命名规范

```json
{
  // 标准 Claims：使用 RFC 定义的名称
  "sub": "...", "iss": "...", "aud": "...",
  
  // 公共 Claims：使用 IANA 注册名称
  "email": "...", "name": "...",
  
  // 私有 Claims：
  // 方案 A：短名称（推荐，已被协议接受）
  "tenant_id": "t-001",
  "roles": ["ADMIN"],
  "permissions": ["read", "write"],
  
  // 方案 B：URI 命名空间（严格防冲突）
  "https://yourcompany.com/claims/department": "Engineering",
  
  // 避免：
  // - 使用下划线前缀（_claim）
  // - 与标准 Claims 名称冲突
}
```

## 4.3 敏感数据保护

```java
// 敏感数据不应放在 JWT Claims 中。
// 如果需要传递敏感信息，优先使用 JWE 加密。

// 错误做法：
claims.put("credit_card", "4111-1111-1111-1111");  // 绝对禁止！
claims.put("password", "xxx");                      // 绝对禁止！
claims.put("ssn", "123-45-6789");                   // 绝对禁止！

// 正确做法：
// 1. JWT 中只放标识，敏感数据在服务端查询
// 2. 或使用 JWE 加密整个 JWT

JWTClaimsSet claims = new JWTClaimsSet.Builder()
    .subject("user123")
    .claim("sensitive_data_id", "ref_abc123")  // 服务端查询引用
    .build();
```

---

# 5. 安全配置最佳实践

## 5.1 服务端安全清单

```yaml
# Spring Security 配置清单
jwt:
  security:
    # 1. 算法白名单
    allowed-algorithms:
      - "RS256"
      - "ES256"
    # 明确拒绝 none、未知算法
    
    # 2. 密钥配置
    require-key-id: true          # 要求 JWT Header 包含 kid
    key-rotation-days: 90
    
    # 3. 验证配置
    validate-issuer: true
    validate-audience: true
    clock-skew-seconds: 30        # 时钟偏差容忍度
    max-token-age-minutes: 60     # 最大令牌年龄（即使未过期）
    
    # 4. 安全 Headers
    hsts-max-age: 31536000
    content-security-policy: "default-src 'self'"
    
    # 5. 日志
    mask-sensitive-headers: true  # 日志脱敏
    log-token-validation-failures: true  # 记录验证失败
```

## 5.2 客户端存储配置

```javascript
// 推荐：Access Token 内存存储
let accessToken = null;  // 存储在闭包中，不暴露给全局

// Refresh Token 使用 HttpOnly Cookie（由服务端设置）
// 前端不直接操作 Refresh Token

// 获取 Token 的函数
async function getValidAccessToken() {
    if (accessToken && !isExpired(accessToken)) {
        return accessToken;
    }
    
    // 尝试刷新
    const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        credentials: 'include'  // 自动携带 Refresh Token Cookie
    });
    
    if (!response.ok) {
        // 刷新失败 → 跳转登录
        window.location.href = '/login';
        throw new Error('Authentication required');
    }
    
    const data = await response.json();
    accessToken = data.access_token;
    return accessToken;
}
```

---

# 6. 令牌吊销机制

## 6.1 吊销方案选择

| 方案 | 实时性 | 复杂度 | 性能影响 | 适用场景 |
|------|--------|--------|----------|----------|
| 黑名单（Redis） | 实时 | 中 | 每次需查询 | 需要即时吊销 |
| 版本号递增 | 近似实时 | 低 | 无额外查询 | 批量吊销 |
| 短有效期 | 延迟 | 低 | 无 | 大多数场景 |
| 令牌绑定 | 中 | 高 | 低 | 高安全场景 |

## 6.2 黑名单实现

```java
@Service
public class TokenBlacklistService {
    
    private final RedisTemplate<String, String> redis;
    
    public void addToBlacklist(String jti, Instant expiresAt) {
        long ttl = Duration.between(Instant.now(), expiresAt).toSeconds();
        if (ttl <= 0) return;  // 已过期的 Token 不需要加入黑名单
        
        redis.opsForValue().set(
            "blacklist:jti:" + jti,
            "revoked",
            Duration.ofSeconds(ttl)
        );
    }
    
    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(redis.hasKey("blacklist:jti:" + jti));
    }
    
    // 批量吊销（按用户）
    public void revokeAllForUser(String userId) {
        // 递增该用户的 Token 版本号
        redis.opsForValue().increment("token_version:user:" + userId);
    }
    
    // 检查用户版本号
    public boolean isTokenVersionValid(String userId, long tokenVersion) {
        String key = "token_version:user:" + userId;
        String value = redis.opsForValue().get(key);
        if (value == null) return true;  // 新用户，无版本限制
        return Long.parseLong(value) == tokenVersion;
    }
}
```

---

# 7. 错误处理与日志管理

## 7.1 安全地记录 JWT 相关错误

```java
@Component
public class SecureJwtLogger {
    
    private static final Logger log = LoggerFactory.getLogger(SecureJwtLogger.class);
    
    // 安全：只记录令牌 ID 和失败类型，不打印完整令牌
    public void logValidationFailure(String token, Exception e) {
        try {
            // 仅解码不验证，提取 jti 用于追踪
            String[] parts = token.split("\\.");
            if (parts.length >= 2) {
                String payload = new String(Base64.getUrlDecoder().decode(parts[1]));
                String jti = JsonPath.read(payload, "$.jti");
                log.warn("JWT validation failed - jti: {}, reason: {}", jti, e.getMessage());
            } else {
                log.warn("JWT validation failed - invalid format");
            }
        } catch (Exception ex) {
            log.warn("JWT validation failed - unable to parse token");
        }
        
        // 绝不要：log.warn("Token: {}", token);  ← 会将 JWT 写入日志！
    }
}
```

## 7.2 统一错误响应

```java
@RestControllerAdvice
public class JwtExceptionHandler {
    
    @ExceptionHandler(ExpiredJwtException.class)
    public ResponseEntity<Map<String, Object>> handleExpiredToken(ExpiredJwtException e) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(Map.of(
                "error", "token_expired",
                "message", "The access token has expired",
                "expired_at", e.getClaims().getExpiration().toString()
            ));
    }
    
    @ExceptionHandler(JwtException.class)
    public ResponseEntity<Map<String, Object>> handleInvalidToken(JwtException e) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(Map.of(
                "error", "invalid_token",
                "message", "The token is invalid or malformed"
            ));
    }
    
    // 注意：不在错误响应中暴露过多的内部信息
    // 错误信息应是通用的，不暴露具体原因（如签名错误 vs 密钥错误）
}
```

---

# 8. 架构最佳实践

## 8.1 API 网关模式

```
                       ┌─────────────────────┐
                       │    API Gateway       │
                       │  (JWT 验证 + 路由)    │
                       │                     │
客户端 ── JWT ────────>│  - 验证签名          │
                       │  - 验证 Claims       │
                       │  - 防重放            │
                       │  - 限流 + 监控       │
                       └──────┬──────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ 服务 A   │   │ 服务 B   │   │ 服务 C   │
        │ (用户)   │   │ (订单)   │   │ (支付)   │
        └──────────┘   └──────────┘   └──────────┘
```

网关集中验证 JWT，后端服务信任网关传递的认证上下文：

```java
// Gateway 验证 JWT 后，通过 Header 传递用户信息
request.addHeader("X-User-Id", claims.getSubject());
request.addHeader("X-User-Roles", String.join(",", roles));
request.addHeader("X-Tenant-Id", claims.get("tenant_id", String.class));

// 后端服务直接读取 Header（信任网关）
String userId = request.getHeader("X-User-Id");
```

## 8.2 BFF（Backend For Frontend）模式

```
浏览器 ←─ Session Cookie ─→ BFF ←─ JWT ─→ 微服务
                              ↑
                    持有 JWT，浏览器接触不到 JWT
```

最适合需要高安全性的 Web 应用（金融、医疗等）。

## 8.3 零信任架构下的 JWT

```
每一次请求都独立验证 JWT：
  - 不依赖于"已认证"状态
  - 不依赖于 IP 白名单
  - 每次验证所有 Claims（exp、nbf、aud、iss）
  - 敏感操作额外要求（MFA、设备指纹、地理位置）
```

---

# 9. 监控与可观测性

## 9.1 关键指标

```java
@Component
public class JwtMetrics {
    
    private final MeterRegistry registry;
    
    public void recordTokenIssued(String issuer) {
        registry.counter("jwt.issued.total", "issuer", issuer).increment();
    }
    
    public void recordTokenValidated(boolean success, String algorithm) {
        registry.counter("jwt.validated.total",
            "result", success ? "success" : "failure",
            "alg", algorithm
        ).increment();
    }
    
    public void recordTokenNearExpiry(long secondsRemaining) {
        registry.summary("jwt.near_expiry.seconds").record(secondsRemaining);
    }
    
    public void recordBlacklistHit(String reason) {
        registry.counter("jwt.blacklist.hit", "reason", reason).increment();
    }
}
```

## 9.2 告警规则

```
告警触发条件：
1. JWT 验证失败率 > 5%（可能遭受攻击）
2. 使用过期密钥的 JWT 数量 > 阈值（密钥轮换后旧密钥仍在大量使用）
3. 单个 IP JWT 验证失败次数 > 阈值（暴力破解）
4. Refresh Token 重用事件（Token 被盗用信号）
5. 黑名单命中率异常（可能正在清理后重放）
```

---

# 10. 开发流程最佳实践

## 10.1 测试建议

```java
// JWT 相关测试应覆盖的场景：

// 单元测试
@Test void shouldGenerateValidToken() {}
@Test void shouldRejectExpiredToken() {}
@Test void shouldRejectTokenWithWrongAudience() {}
@Test void shouldRejectTamperedToken() {}
@Test void shouldRejectTokenWithNoneAlgorithm() {}
@Test void shouldHandleClockSkew() {}
@Test void shouldRejectReplayedToken() {}

// 集成测试
@Test void shouldRefreshExpiredToken() {}
@Test void shouldRejectRevokedRefreshToken() {}
@Test void shouldPropagateTokenAcrossServices() {}

// 安全测试
@Test void shouldResistAlgorithmConfusionAttack() {}
@Test void shouldNotLogFullToken() {}
@Test void shouldRateLimitFailedAttempts() {}
```

---

# 11. 常见错误与纠正

| 错误 | 正确做法 |
|------|----------|
| 硬编码密钥 | 从环境变量或密钥管理服务读取 |
| 使用 alg=none | 明确算法白名单 |
| 不验证 aud | 始终验证 aud 包含当前服务 |
| Payload 放敏感数据 | 只放非敏感标识，或用 JWE |
| Access Token 长期有效 | 15 分钟以内 + Refresh Token |
| 不实现 Refresh Token 轮换 | 每次刷新时更换 Refresh Token |
| 在 localStorage 存储 Access Token | 存储于内存或 HttpOnly Cookie |
| 在 URL 中传递 JWT | 使用 Authorization Header |
| 日志中打印完整 JWT | 仅记录 jti 或脱敏 |
| 不同服务共享 HMAC 密钥 | 内部服务间使用受限，外部使用非对称 |
