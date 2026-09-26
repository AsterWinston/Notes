
# 1. 概述

Token 是现代 SSO 的核心凭证。好的 Token 设计需要平衡安全性、性能和开发体验。本章从设计原则出发，系统讲解 Access Token、Refresh Token 和各种 Token 模式的设计与生命周期管理。

# 2. Token 类型

## 2.1 Token 分类

| 类型 | 生命周期 | 用途 | 格式 | 存储位置 |
|------|---------|------|------|---------|
| **Access Token** | 5-30 分钟 | 访问受保护资源 | JWT / Opaque | 客户端/Header |
| **Refresh Token** | 7-90 天 | 获取新 Access Token | Opaque / JWT | 服务端(Redis) + 客户端 |
| **ID Token** | 5-10 分钟 | 身份信息传递 | JWT | 客户端 |
| **Authorization Code** | 1-10 分钟 | OAuth2 授权码换 Token | Opaque | URL 参数（临时） |
| **Device Code** | 5-15 分钟 | 设备码模式 | Opaque | 设备 |
| **API Key** | 长期 | M2M 认证 | Opaque | 服务端 |

## 2.2 Token 生命周期状态机

```
                    ┌──────────┐
                    │  创建     │ ← 登录 / 授权
                    └────┬─────┘
                         │
                         ▼
                    ┌──────────┐
              ┌─────│  活跃     │─────┐
              │     └────┬─────┘     │
              │          │           │
              │     ┌────▼─────┐     │
              │     │  即将过期  │     │
              │     └────┬─────┘     │
              │          │           │
         Refresh Token   │     用户登出
              │          │           │
              ▼          ▼           ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │新Token │ │  过期   │ │  吊销   │
         └────────┘ └────────┘ └────────┘
                          │           │
                          └─────┬─────┘
                                ▼
                          ┌──────────┐
                          │  失效     │ ← 不可再用
                          └──────────┘
```

# 3. Access Token 设计

## 3.1 JWT Access Token 最佳实践

```java
/**
 * Access Token Payload 设计
 */
public class AccessTokenClaims {
    // 标准 Claims（必须）
    private String iss;      // "https://auth.example.com"
    private String sub;      // "user-12345"
    private String aud;      // ["service-a", "service-b"]
    private Date exp;        // now + 15 minutes
    private Date iat;        // now
    private String jti;      // UUID（防重放）

    // 轻量权限（可选，建议控制在 5 个以内）
    private List<String> roles;   // ["ROLE_USER"]
    private List<String> scopes;  // ["read:order", "write:order"]
    private String tenantId;      // 多租户标识
    private String tokenType;     // "access"

    // ⚠️ 不要放的内容：
    // - 密码、密钥等敏感信息
    // - 大量用户属性（用 UserInfo 端点获取）
    // - 动态数据（Token 签发后不会更新）
    // - 过长的权限列表（控制在 1KB 以内）
}
```

## 3.2 Opaque Token（不透明 Token）

```java
/**
 * Opaque Token：Token 本身是随机字符串，不包含任何信息
 * 验证时需要调用 Introspection 端点
 */
@Service
public class OpaqueTokenService {

    @Autowired
    private RedisTemplate<String, TokenInfo> redisTemplate;

    /**
     * 创建 Opaque Access Token
     */
    public String createAccessToken(UserDetails user) {
        String tokenId = UUID.randomUUID().toString()
            .replace("-", "");  // 32 位随机字符串

        TokenInfo info = TokenInfo.builder()
            .username(user.getUsername())
            .authorities(user.getAuthorities())
            .issuedAt(Instant.now())
            .expiresAt(Instant.now().plusSeconds(900))  // 15 分钟
            .tokenType("access")
            .build();

        // 存储 Token 信息到 Redis
        redisTemplate.opsForValue().set(
            "token:" + tokenId,
            info,
            Duration.ofMinutes(15));

        return tokenId;
    }

    /**
     * 验证 Opaque Token（Introspection）
     */
    public TokenInfo validateToken(String tokenId) {
        TokenInfo info = redisTemplate.opsForValue()
            .get("token:" + tokenId);

        if (info == null) {
            throw new InvalidTokenException("Token 无效或已过期");
        }

        if (info.getExpiresAt().isBefore(Instant.now())) {
            redisTemplate.delete("token:" + tokenId);
            throw new InvalidTokenException("Token 已过期");
        }

        return info;
    }
}

// Opaque Token vs JWT 对比：
// JWT:
//   ✅ 零网络 I/O（本地验签）
//   ✅ 适合微服务（公钥分发给所有服务）
//   ❌ Token 体积大（1-5KB）
//   ❌ 吊销困难（需黑名单）
// Opaque:
//   ✅ Token 体积小（< 100 字节）
//   ✅ 即时吊销
//   ❌ 每次需查存储（Redis/DB）
//   ❌ 存储中心是性能瓶颈
```

# 4. Refresh Token 设计

## 4.1 Refresh Token 模式

```
模式 1：长期 Refresh Token（简单，安全性低）
  Refresh Token = 随机UUID → 存 Redis，7-30天有效期
  每次用到都不过期，只返回新 Access Token

模式 2：Refresh Token Rotation（推荐，安全性高）
  每次用 Refresh Token 刷新 → 
    ├─ 旧 Refresh Token 立即失效
    ├─ 返回新的 Access Token
    └─ 返回新的 Refresh Token
  
  如检测到旧 Refresh Token 被重复使用 → 
    └─ 整个 Token 家族全部吊销（Reuse Detection）

模式 3：Sender-Constrained Refresh Token（最安全）
  Refresh Token 绑定到特定客户端证书/DPoP Key
  即使 Refresh Token 泄露，其他人也无法使用
```

## 4.2 Refresh Token Rotation 实现

```java
@Service
public class RefreshTokenRotationService {

    @Autowired
    private RedisTemplate<String, RefreshTokenEntry> redisTemplate;

    /**
     * Refresh Token 条目
     */
    @Data
    @Builder
    static class RefreshTokenEntry {
        private String username;
        private String family;       // Token 家族 ID（用于泄露检测）
        private String previousToken; // 前一个 Refresh Token
        private Instant expiredAt;
        private boolean used;        // 是否已被使用
    }

    /**
     * 创建 Refresh Token
     */
    public String createRefreshToken(String username) {
        String tokenValue = generateSecureToken();
        String family = UUID.randomUUID().toString();

        RefreshTokenEntry entry = RefreshTokenEntry.builder()
            .username(username)
            .family(family)
            .previousToken(null)
            .expiredAt(Instant.now().plus(Duration.ofDays(7)))
            .used(false)
            .build();

        redisTemplate.opsForValue().set(
            "refresh:" + tokenValue,
            entry,
            Duration.ofDays(7));

        return tokenValue;
    }

    /**
     * 使用 Refresh Token 换取新 Token（Rotation）
     */
    public TokenRotationResult rotate(String oldRefreshToken) {
        RefreshTokenEntry oldEntry = redisTemplate.opsForValue()
            .get("refresh:" + oldRefreshToken);

        if (oldEntry == null) {
            throw new InvalidTokenException("Refresh Token 无效");
        }

        // 【泄露检测】如果该 Token 已被使用过 → 说明 Token 被盗
        if (oldEntry.isUsed()) {
            // 撤销整个 Token 家族
            revokeTokenFamily(oldEntry.getFamily());
            throw new SecurityException(
                "检测到 Refresh Token 被重复使用，已撤销所有 Token");
        }

        // 检查是否过期
        if (oldEntry.getExpiredAt().isBefore(Instant.now())) {
            throw new InvalidTokenException("Refresh Token 已过期");
        }

        // 标记旧 Token 为已使用
        oldEntry.setUsed(true);
        redisTemplate.opsForValue().set(
            "refresh:" + oldRefreshToken, oldEntry,
            Duration.between(Instant.now(), oldEntry.getExpiredAt()));

        // 创建新 Token（Rotation）
        String newRefreshToken = generateSecureToken();
        RefreshTokenEntry newEntry = RefreshTokenEntry.builder()
            .username(oldEntry.getUsername())
            .family(oldEntry.getFamily())
            .previousToken(oldRefreshToken)
            .expiredAt(oldEntry.getExpiredAt())  // 保持原过期时间
            .used(false)
            .build();

        redisTemplate.opsForValue().set(
            "refresh:" + newRefreshToken,
            newEntry,
            Duration.between(Instant.now(), oldEntry.getExpiredAt()));

        return new TokenRotationResult(
            newRefreshToken,
            oldEntry.getUsername()
        );
    }

    /**
     * 撤销整个 Token 家族
     */
    private void revokeTokenFamily(String family) {
        // 实际中需要扫描所有 Refresh Token，删除属于该 family 的
        // 简化实现：
        Set<String> keys = redisTemplate.keys("refresh:*");
        for (String key : keys) {
            RefreshTokenEntry entry = redisTemplate.opsForValue().get(key);
            if (entry != null && family.equals(entry.getFamily())) {
                redisTemplate.delete(key);
            }
        }
    }

    private String generateSecureToken() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding()
            .encodeToString(bytes);
    }

    @Data
    @AllArgsConstructor
    static class TokenRotationResult {
        private String refreshToken;
        private String username;
    }
}
```

# 5. Token 黑名单管理

## 5.1 Redis 实现

```java
@Service
public class TokenBlacklistService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final String BLACKLIST_PREFIX = "token_blacklist:";

    /**
     * 将 Access Token 加入黑名单
     */
    public void blacklist(String jti, Date expiresAt) {
        long ttl = expiresAt.getTime() - System.currentTimeMillis();
        if (ttl > 0) {
            redisTemplate.opsForValue().set(
                BLACKLIST_PREFIX + jti,
                "revoked",
                Duration.ofMillis(ttl));
        }
        // Token 已过期，无需加入黑名单
    }

    /**
     * 检查 Token 是否在黑名单中
     */
    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey(BLACKLIST_PREFIX + jti));
    }

    /**
     * 使用 Bloom Filter 优化（减少不必要的 Redis 查询）
     */
    @Bean
    public BloomFilter<String> tokenBloomFilter() {
        BloomFilter<String> filter = BloomFilter.create(
            Funnels.stringFunnel(StandardCharsets.UTF_8),
            100_000,    // 预计元素数量
            0.01        // 误判率 1%
        );
        return filter;
    }

    public boolean mightBeBlacklisted(String jti) {
        // Bloom Filter 说"不在"→ 一定不在
        // Bloom Filter 说"可能在"→ 需要查 Redis 确认
        return tokenBloomFilter().mightContain(jti);
    }
}
```

## 5.2 黑名单清理与监控

```java
@Component
public class BlacklistCleanupTask {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 定时任务：清理过期的黑名单条目
     * （Redis 的 TTL 会自动清理，但也可再手动检查）
     */
    @Scheduled(cron = "0 0 */1 * * ?")  // 每小时
    public void cleanupExpiredEntries() {
        Set<String> keys = redisTemplate.keys("token_blacklist:*");
        if (keys != null) {
            for (String key : keys) {
                Long ttl = redisTemplate.getExpire(key, TimeUnit.SECONDS);
                if (ttl != null && ttl <= 0) {
                    redisTemplate.delete(key);
                }
            }
        }
    }

    /**
     * 监控黑名单大小
     */
    @Scheduled(fixedRate = 60000)  // 每分钟
    public void monitorBlacklistSize() {
        Long size = redisTemplate.countExistingKeys(
            Collections.singletonList("token_blacklist:*"));
        if (size > 10000) {
            log.warn("Token 黑名单超过 10000 条: {}", size);
        }
    }
}
```

# 6. Token 安全存储

## 6.1 服务端存储策略

| 存储方式 | 用途 | 推荐度 |
|---------|------|--------|
| **Redis** | Refresh Token、黑名单、Opaque Token | ⭐⭐⭐⭐⭐ |
| **数据库** | 长期 Token 记录（审计） | ⭐⭐⭐ |
| **内存** | 签名密钥缓存、公钥缓存 | ⭐⭐⭐⭐ |
| **密钥管理服务** | 签名私钥、加密密钥 | ⭐⭐⭐⭐⭐ |

## 6.2 客户端存储策略

| 客户端类型 | Access Token | Refresh Token |
|-----------|-------------|---------------|
| **Web 应用（BFF）** | BFF Session（HttpOnly Cookie） | BFF Session |
| **SPA** | BFF 代理获取，前端无感知 | BFF 管理 |
| **移动 App (iOS)** | Keychain | Keychain |
| **移动 App (Android)** | Android Keystore (EncryptedSharedPreferences) | Android Keystore |
| **后端服务** | 环境变量 / Vault / ConfigMap | 环境变量 / Vault |

# 7. Token 过期策略

## 7.1 时间窗口设计

```java
/**
 * 推荐的 Token 过期时间配置
 */
public class TokenExpirationConfig {

    // Access Token: 15 分钟
    public static final Duration ACCESS_TOKEN_TTL = Duration.ofMinutes(15);

    // Refresh Token: 7 天
    public static final Duration REFRESH_TOKEN_TTL = Duration.ofDays(7);

    // ID Token: 5 分钟（做身份验证用，不是访问凭证）
    public static final Duration ID_TOKEN_TTL = Duration.ofMinutes(5);

    // Authorization Code: 5 分钟（一次性使用）
    public static final Duration AUTH_CODE_TTL = Duration.ofMinutes(5);

    // 设备码: 10 分钟（用户有时间输入）
    public static final Duration DEVICE_CODE_TTL = Duration.ofMinutes(10);

    // 签名密钥轮换周期: 30 天
    public static final Duration KEY_ROTATION_PERIOD = Duration.ofDays(30);

    // 验签允许的时钟偏差: 30 秒
    public static final Duration CLOCK_SKEW = Duration.ofSeconds(30);
}
```

## 7.2 滑动过期 vs 绝对过期

```java
// 滑动过期：每次使用延长有效期（Redis TTL 天然支持）
redisTemplate.expire("refresh:" + token, Duration.ofDays(7));

// 绝对过期：固定时间点过期（JWT exp 字段）
// 优点：简单、可预测
// 缺点：活跃用户也可能突然过期

// 混合策略（推荐）：
// - Access Token: 绝对过期（JWT exp），短周期
// - Refresh Token: 滑动过期（Redis TTL），不在 JWT 中
// - 整体会话: 绝对过期（最长 24h），之后必须重新登录
```

# 8. 总结

| 设计点 | 推荐方案 |
|--------|---------|
| Access Token 格式 | JWT（RS256 签名） |
| Access Token 有效期 | 15 分钟 |
| Refresh Token 格式 | Opaque（随机字符串） |
| Refresh Token 有效期 | 7 天（滑动） |
| Token 吊销 | jti 黑名单（Redis）+ Refresh Token Rotation |
| 泄露检测 | Refresh Token Reuse Detection |
| 密钥管理 | 非对称密钥（RS256/ES256），定期轮换 |
| 存储 | Redis（热数据） + DB（审计记录） |
