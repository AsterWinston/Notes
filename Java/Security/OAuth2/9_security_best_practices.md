# 1. OAuth2 安全威胁模型

OAuth2 的安全性不是协议本身保证的，而是由**实现质量**和**部署配置**共同决定的。RFC 6819 列出了 OAuth2 的 50+ 种威胁，本章基于最新 OAuth 2.0 Security Best Current Practice 和 OAuth 2.1 草案，总结关键安全实践。

## 1.1 威胁分类

```
OAuth2 威胁全景图：

┌──────────────────────────────────────────────────────────────────┐
│                          OAuth2 威胁                              │
├────────────────┬──────────────────┬──────────────┬──────────────┤
│ 客户端攻击      │ Token 攻击        │ 端点攻击      │ 协议攻击      │
├────────────────┼──────────────────┼──────────────┼──────────────┤
│ 客户端凭证泄露  │ Access Token 泄露 │ CSRF         │ 混合攻击      │
│ 授权码截获     │ Refresh Token 重用│ 开放重定向    │ 跨协议攻击    │
│ 恶意客户端     │ Token 注入        │ 参数篡改      │ 降级攻击      │
│ 注册劫持      │ Token 侧信道      │ 重放攻击      │ 中间人攻击    │
│ 客户端伪冒     │ Bearer Token 滥用 │ 暴力破解      │ SSRF         │
└────────────────┴──────────────────┴──────────────┴──────────────┘
```

---

# 2. 强制安全配置

## 2.1 Authorize Code + PKCE（所有用户参与的场景）

```
必须执行的检查项：

1. response_type 必须是 "code"（禁止 "token"）
2. 必须使用 PKCE（code_challenge_method=S256）
3. code_challenge_method 只接受 "S256"（不接受 "plain"）
4. redirect_uri 严格匹配（完整字符串对比，不是前缀匹配）
5. state 参数必须存在且验证
```

### 2.1.1 授权服务器端强制 PKCE 验证

```java
// 在 /token 端点验证 PKCE
public void validatePkce(String storedCodeChallenge, String storedCodeChallengeMethod, 
                          String receivedCodeVerifier) {
    if (storedCodeChallenge == null) {
        throw new InvalidGrantException("PKCE is required");
    }
    
    if (!"S256".equals(storedCodeChallengeMethod)) {
        throw new InvalidGrantException("Only S256 code_challenge_method is supported");
    }
    
    String expectedChallenge = sha256Base64Url(receivedCodeVerifier);
    
    // 使用恒定时间比较（防时序攻击）
    if (!MessageDigest.isEqual(
            expectedChallenge.getBytes(StandardCharsets.US_ASCII),
            storedCodeChallenge.getBytes(StandardCharsets.US_ASCII))) {
        throw new InvalidGrantException("PKCE verification failed");
    }
}
```

## 2.2 Redirect URI 严格验证

```java
/**
 * 合法 redirect_uri 示例和非法示例
 */
public class RedirectUriValidator {
    
    // 合法的 redirect_uri 规则：
    // 1. 必须使用 HTTPS（localhost 除外）
    // 2. 完整匹配（含路径）
    // 3. 不包含 Fragment（#）
    // 4. 必须是绝对 URI
    
    private static final Set<String> ALLOWED_SCHEMES = Set.of("https", "http");
    private static final Set<String> LOCALHOST_HOSTS = Set.of("localhost", "127.0.0.1", "[::1]");
    
    public boolean isValid(String registeredUri, String requestedUri) {
        if (requestedUri == null || registeredUri == null) {
            return false;
        }
        
        URI registered = URI.create(registeredUri);
        URI requested = URI.create(requestedUri);
        
        // 1. Scheme 检查
        if (!ALLOWED_SCHEMES.contains(requested.getScheme())) {
            return false;
        }
        
        // 2. 非 localhost 必须 HTTPS
        if (!LOCALHOST_HOSTS.contains(requested.getHost()) 
            && !"https".equals(requested.getScheme())) {
            return false;
        }
        
        // 3. 不允许 Fragment
        if (requested.getFragment() != null) {
            return false;
        }
        
        // 4. Host 严格匹配
        if (!registered.getHost().equals(requested.getHost())) {
            return false;
        }
        
        // 5. Port 匹配（或默认端口）
        int regPort = registered.getPort() == -1 ? defaultPort(registered.getScheme()) : registered.getPort();
        int reqPort = requested.getPort() == -1 ? defaultPort(requested.getScheme()) : requested.getPort();
        if (regPort != reqPort) {
            return false;
        }
        
        // 6. Path 精确匹配
        if (!registered.getPath().equals(requested.getPath())) {
            return false;
        }
        
        return true;
    }
}
```

---

# 3. Token 安全最佳实践

## 3.1 Access Token 安全

| 实践 | 说明 | 严重程度 |
|------|------|----------|
| 短有效期 | 15-30 分钟，不超过 1 小时 | 关键 |
| 仅 Bearer Header | 禁止 URL 参数和 POST Body 传输 | 关键 |
| 使用 JWT 格式 | 资源服务器可本地验证，减少网络调用 | 推荐 |
| 包含 jti 声明 | 支持撤销检测和审计 | 推荐 |
| 限制 audience | aud 指定目标资源服务器 | 推荐 |
| 避免敏感信息 | 不在 JWT 中包含密码、完整身份证等 | 重要 |
| HTTPS 传输 | 所有 Token 传输使用 TLS 1.2+ | 关键 |

## 3.2 Refresh Token 安全

| 实践 | 说明 | 严重程度 |
|------|------|----------|
| 使用后轮换 | 每次刷新都颁发新 Refresh Token，旧 Token 立即失效 | 关键 |
| 重放检测 | 检测到已失效的 Refresh Token 被使用 → 撤销整个家族 | 关键 |
| 发送方限制 | 绑定到使用它的客户端 | 推荐 |
| HTTPS 传输 | 仅通过 HTTPS POST 传输 | 关键 |
| 不透明格式 | 不使用 JWT（避免信息泄露和额外安全隐患） | 推荐 |
| 独立于 Access Token | 有效期较长（天~月），但比 Access Token 更难获取 | - |

## 3.3 Sender-Constrained Token（DPoP / mTLS）

```
Bearer Token 的固有问题：持有 Token 的任何人就可以使用它
解决方案：Sender-Constrained Token（绑定发送方）

方式一：DPoP（RFC 9449）
  客户端持有一个密钥对
  每个 API 请求携带一个 DPoP 证明（JWT）
  DPoP JWT 包含：
  - 请求的 HTTP Method 和 URL（防跨请求重放）
  - Access Token 的哈希（绑定 Token）
  DPoP JWT 由客户端私钥签名 → 服务器用公钥验证

方式二：mTLS（双向 TLS）
  Access Token 绑定到客户端 TLS 证书
  攻击者即使盗取 Token，没有对应的 TLS 证书也无法使用
  通过 Token 中的 cnf 声明实现绑定：
  {
    "cnf": {
      "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg2"
    }
  }
  
当前状态：DPoP 仍在推广中，mTLS 部署复杂度高
```

---

# 4. 客户端安全实践

## 4.1 不同客户端的存储规范

```
                    Token 存储安全指南

机密客户端 (Web 后端)：
┌──────────────────────────────────────────────────────┐
│ Access Token:  内存 + 数据库（AES-256-GCM 加密）      │
│ Refresh Token: 数据库（AES-256-GCM 加密）             │
│ 会话:          HttpOnly Secure SameSite Cookie        │
│ 密钥:          环境变量 / Vault / KMS                  │
└──────────────────────────────────────────────────────┘

公开客户端 (SPA)：
┌──────────────────────────────────────────────────────┐
│ 推荐方案:  BFF 模式（Token 不出浏览器）                │
│ 降级方案:  内存存储 + 静默刷新                         │
│ 存储:      Service Worker 拦截注入（不用 localStorage）│
│ 密钥:      无（不存储 client_secret）                  │
└──────────────────────────────────────────────────────┘

公开客户端 (移动 App)：
┌──────────────────────────────────────────────────────┐
│ Access Token:  EncryptedSharedPreferences (Android)   │
│                Keychain (iOS)                         │
│ Refresh Token: Android Keystore / iOS Keychain        │
│ 签发:         AppAuth 库                              │
│ 密钥:         不存储 client_secret（使用 PKCE）        │
└──────────────────────────────────────────────────────┘
```

## 4.2 client_secret 管理

```
client_secret 全生命周期管理：

1. 生成：高熵随机字符串（≥ 128 bit）
   生成工具：openssl rand -base64 32

2. 传输：第一次注册时通过安全通道（TLS）传输
   后续获取通过管理界面

3. 存储：
   - 授权服务器端：BCrypt/Argon2 哈希存储
   - 客户端：密钥管理服务（Vault, KMS, 环境变量）
   - 绝不：代码硬编码、Git 仓库、日志

4. 轮换：
   - 支持重叠期（新旧密钥同时有效 24h）
   - 轮换后旧密钥标记为待过期
   - 紧急轮换（泄露时）立即生效

5. 销毁：
   - 不再需要的密钥立即销毁
   - 定期清理过期密钥的哈希
```

---

# 5. 授权服务器安全实践

## 5.1 关键安全配置

```yaml
# 授权服务器安全配置清单

⏤ Authorization Code:
  - 有效期 ≤ 60 秒
  - 一次性使用（使用后立即失效）
  - 数据库存储 hash 值
  - 绑定 client_id, redirect_uri, code_challenge
  - 重放检测后撤销所有用户 Token

⏤ Access Token:
  - 有效期 15-30 分钟
  - JWT 格式（RS256/ES256 签名）
  - 包含 jti 声明
  - audience 精确指定

⏤ Refresh Token:
  - 每次使用时自动轮换
  - 家族追踪（父子关系链）
  - 重放检测（整个家族立即失效）

⏤ 客户端认证:
  - client_secret 使用 BCrypt/Argon2 哈希存储
  - private_key_jwt > client_secret_basic > client_secret_post > none
  - 限制 client_secret 传输方式为 HTTP Basic Auth

⏤ 传输安全:
  - 所有端点强制 HTTPS（HSTS）
  - 禁止 Token 通过 URL Query String 传递
  - CORS 白名单（仅信任的 origin）
```

## 5.2 限流与暴力破解防护

```java
/**
 * Token Endpoint 限流防护
 * 防止暴力破解 client_secret / authorization code
 */
@Component
public class TokenEndpointRateLimiter {
    
    // 每个 IP 每分钟最多 30 次 Token 请求
    private final RateLimiter ipLimiter = RateLimiter.create(30.0 / 60.0);
    
    // 每个 client_id 每分钟最多 60 次 Token 请求
    private final LoadingCache<String, RateLimiter> clientLimiters = Caffeine.newBuilder()
        .expireAfterAccess(10, TimeUnit.MINUTES)
        .build(key -> RateLimiter.create(60.0 / 60.0));
    
    // 同一个 authorization code 最多尝试 3 次（防暴力猜测）
    private final LoadingCache<String, AtomicInteger> codeAttempts = Caffeine.newBuilder()
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build(key -> new AtomicInteger(0));
    
    public void checkRateLimit(String clientId, String ipAddress, String code) {
        // IP 级别限流
        if (!ipLimiter.tryAcquire()) {
            throw new RateLimitExceededException("Too many requests from IP: " + ipAddress);
        }
        
        // 客户端级别限流
        RateLimiter clientLimiter = clientLimiters.get(clientId);
        if (!clientLimiter.tryAcquire()) {
            throw new RateLimitExceededException("Too many requests for client: " + clientId);
        }
        
        // 授权码尝试次数限制（防暴力猜测）
        if (code != null) {
            int attempts = codeAttempts.get(code).incrementAndGet();
            if (attempts > 3) {
                // 撤销关联的授权码
                authorizationCodeService.revokeByCode(code);
                throw new SecurityAlertException("Possible code brute-force detected for code");
            }
        }
    }
}
```

---

# 6. 安全审计与监控

## 6.1 关键审计日志

```
必须记录的审计事件：

1. 授权事件
   - 用户登录（成功/失败）
   - 用户授权申请（client_id, scope, time）
   - 用户同意/拒绝授权
   - 用户撤销授权

2. Token 事件
   - Token 颁发（access_token_id, client_id, user_id, scope）
   - Token 刷新（new_token_id, old_token_id）
   - Token 吊销（token_id, reason）
   - Introspection 请求（token_id, client_id）

3. 安全事件
   - 认证失败（client_id, reason, IP）
   - PKCE 验证失败
   - Redirect URI 不匹配
   - State 不匹配
   - 授权码重放检测
   - Refresh Token 重用检测
   - 速率限制触发

日志格式（结构化）：
{
  "event": "token.issued",
  "timestamp": "2024-01-15T10:30:00Z",
  "actor": {"client_id": "xxx", "user_id": "yyy"},
  "target": {"token_id": "jti-abc123", "type": "access_token"},
  "context": {"grant_type": "authorization_code", "scopes": "read write"},
  "outcome": "success",
  "source_ip": "192.168.1.100"
}
```

## 6.2 安全告警规则

```
实时告警条件：

危险级别 - 立即响应：
  □ Refresh Token 重放检测
  □ 同一用户短时间内（< 1min）大量 Token 请求
  □ client_secret 暴力猜测（连续认证失败）
  □ 授权码批量尝试

警告级别 - 值班关注：
  □ 用户从不常见的地理位置登录
  □ 用户短时间内授权多个新应用
  □ Introspection 端点高频请求（异常流量）
  □ 速率限制频繁触发

通知级别 - 定期审查：
  □ 长期不活跃的 Refresh Token 突然使用
  □ 新的应用注册
  □ 密钥轮换操作
```

---

# 7. OAuth2 降级攻击防护

## 7.1 算法降级攻击

```
攻击场景：
1. 授权服务器支持 RS256 和 HS256 签名算法
2. client_secret 是已知值（公开客户端）
3. 攻击者将 alg 改为 HS256，用 client_secret 作为 HMAC 密钥签名
4. 资源服务器使用 client_secret 验证签名 → 通过

防御：
1. 资源服务器只接受非对称算法（RS256, ES256 等）
2. 明确配置允许的算法列表
3. 拒绝 alg: "none"
```

```java
// 资源服务器端算法白名单
private static final Set<String> ALLOWED_ALGORITHMS = Set.of(
    "RS256", "RS384", "RS512",
    "ES256", "ES384", "ES512",
    "PS256", "PS384", "PS512"
);

// 拒绝对称算法和 "none"
private static final Set<String> BLOCKED_ALGORITHMS = Set.of(
    "none", "HS256", "HS384", "HS512"
);
```

## 7.2 Grant Type 降级攻击

```
防御：
- 每个客户端明确配置允许的 grant_types
- 不允许客户端在 Token 请求中使用未注册的 grant_type
- 拒绝 Resource Owner Password 和 Implicit grant type
```

---

# 8. 混合模式（Hybrid Flow）安全

```
混合模式的风险：
response_type=code id_token

问题：
- ID Token 在 URL Fragment 中返回 → 可能被窃取
- 攻击者可以在 code 交换前获取 ID Token → 冒充用户

防御：
- 验证 c_hash（绑定 ID Token 和 code）
- 验证 ID Token 的 nonce
- 如果不需要提前获取 ID Token，使用纯 code 模式
```

---

# 9. 生产环境部署检查清单

```
OAuth2 生产环境安全检查清单

部署前必检：
□ 所有通信使用 TLS 1.2+
□ HSTS 已配置（max-age ≥ 1 year）
□ 证书有效且由受信任的 CA 签发
□ 授权服务器和资源服务器间的内部通信使用 mTLS/内网

配置安全检查：
□ client_secret 使用 BCrypt/Argon2 存储
□ PKCE 强制启用（S256 模式）
□ Refresh Token 轮换已启用
□ redirect_uri 严格匹配验证
□ state 参数验证已实现
□ Token 有效期已合理配置
□ 仅限于推荐的 grant_type
□ CORS 策略已限制到白名单
□ 速率限制已配置

密钥管理检查：
□ 签名密钥安全生成和存储
□ 密钥轮换流程已建立
□ JWKS 端点仅暴露公钥
□ 私钥存储在 HSM/KMS 中

监控和运维检查：
□ 完整的审计日志
□ 安全事件告警已配置
□ Token 吊销流程已就绪
□ 应急响应流程已文档化
□ 定期安全测试计划

数据保护检查：
□ Token 在数据库中加密存储
□ 备份数据加密
□ 日志中不记录完整 Token
□ PII 数据处理合规（GDPR 等）
```
