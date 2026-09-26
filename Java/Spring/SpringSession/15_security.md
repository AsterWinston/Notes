# 1. Session 安全概述

Session 是 Web 应用安全的核心环节。安全的 Session 管理需要覆盖以下几个方面：

| 安全维度 | 说明 |
|----------|------|
| **Session ID 安全** | 防止猜测、窃取、固定（Fixation） |
| **Cookie 安全** | HttpOnly、Secure、SameSite 等属性 |
| **传输安全** | 全程 HTTPS 加密 |
| **存储安全** | 存储中的数据加密、脱敏 |
| **并发控制** | 同一账号多设备登录控制 |
| **Session 吊销** | 及时失效，防止被滥用 |

---

# 2. Cookie 安全属性

## 2.1 推荐配置

```yaml
server:
  servlet:
    session:
      cookie:
        http-only: true       # 禁止 JavaScript 访问（防 XSS）
        secure: true          # 仅通过 HTTPS 传输
        same-site: Lax        # 防止跨站请求携带
```

## 2.2 HttpOnly

```java
// 设置 HttpOnly 后，以下 JS 代码无法获取 Cookie
// document.cookie  → 看不到 SESSION Cookie

// 正确配置：
@Bean
public CookieSerializer cookieSerializer() {
    DefaultCookieSerializer serializer = new DefaultCookieSerializer();
    serializer.setUseHttpOnlyCookie(true);    // 生产环境必须 true
    return serializer;
}
```

## 2.3 Secure

```yaml
# 开发环境（HTTP）需要关闭
server.servlet.session.cookie.secure: false

# 生产环境必须开启
# 建议使用 Spring Profile 区分环境
spring.profiles.active: prod
```

## 2.4 SameSite

| 值 | 效果 | 安全等级 |
|----|------|----------|
| **Strict** | 完全阻止跨站携带 Cookie | 最高（可能影响体验） |
| **Lax** | 允许部分跨站请求（GET 导航）携带 | 高（推荐） |
| **None** | 允许跨站携带（必须配合 Secure） | 低（仅用于第三方场景） |

---

# 3. Session ID 安全

## 3.1 防止 Session ID 猜测

```java
// 默认使用 UUID（128 位随机），足够安全
// 不要使用自增 ID 或基于用户信息的可预测 ID

// 如果需要额外安全层，可以添加 HMAC 签名：
public String generateSecureId() {
    String uuid = UUID.randomUUID().toString();
    String hmac = HmacUtils.hmacSha256Hex(secretKey, uuid);
    return uuid + "." + hmac;
}
```

## 3.2 Session Fixation 防护

Spring Security + Spring Session 默认已启用：

```java
// 登录成功后自动执行：
// session.changeSessionId()（Spring Session 实现）
// 旧 Session ID 失效，生成新 ID
// 攻击者即使知道旧 ID 也无法利用

// 自定义配置
http
    .sessionManagement(session -> session
        .sessionFixation().migrateSession() // 或 newSession()
    );
```

## 3.3 Session ID 泄露处理

```java
// 如果怀疑 Session ID 已泄露，强制重新生成：
@PostMapping("/security/regenerate-session")
public String regenerateSession(HttpServletRequest request) {
    HttpSession session = request.getSession();
    // 手动更换 Session ID
    String oldId = session.getId();
    String newId = request.changeSessionId(); // Spring Session 支持

    logger.warn("Session ID 已重新生成: {} → {}", oldId, newId);
    return "Session ID 已更新";
}
```

---

# 4. 传输安全

## 4.1 HTTPS 强制

```java
@Configuration
public class HttpsConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .requiresChannel(channel -> channel
                .anyRequest().requiresSecure()  // 强制 HTTPS
            );
        return http.build();
    }
}
```

## 4.2 HSTS 响应头

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**效果**：浏览器在指定时间内自动将 HTTP 请求升级为 HTTPS。

---

# 5. 存储安全

## 5.1 Redis 连接加密

```yaml
spring:
  data:
    redis:
      ssl:
        enabled: true
      # 客户端证书（如需双向认证）
      # ssl.key-store: classpath:redis-client.p12
      # ssl.key-store-password: password
```

## 5.2 Redis 密码认证

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}  # 使用环境变量或配置中心
      # 不要硬编码密码
```

## 5.3 Session 数据加密

```java
@Component
public class EncryptionService {

    private final SecretKey aesKey;

    public byte[] encrypt(Object data) {
        // AES-256-GCM 加密
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, aesKey);
        return cipher.doFinal(serialize(data));
    }

    public <T> T decrypt(byte[] encrypted) {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.DECRYPT_MODE, aesKey);
        return (T) deserialize(cipher.doFinal(encrypted));
    }
}
```

## 5.4 敏感数据脱敏

```java
// 不要在 Session 中存储明文密码、完整身份证号等

// 正确做法：
session.setAttribute("userId", userId);          // 存标识
session.setAttribute("userName", maskedName);    // 脱敏后的数据

// 错误做法：
session.setAttribute("password", rawPassword);   // 绝不能存密码
session.setAttribute("idCard", "110101...");     // 敏感数据不宜存 Session
```

---

# 6. 并发登录防护

## 6.1 账号共享检测

```java
@Component
public class AccountSharingDetector {

    private static final int MAX_IPS_PER_ACCOUNT = 3;

    public boolean isAccountSharing(String username, String newIp,
            FindByIndexNameSessionRepository<?> repository) {
        Map<String, ? extends Session> sessions = repository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );

        Set<String> uniqueIps = new HashSet<>();
        for (Session session : sessions.values()) {
            String ip = session.getAttribute("LOGIN_IP");
            if (ip != null) uniqueIps.add(ip);
        }

        // 从多个 IP 登录 → 可能账号共享
        if (uniqueIps.size() >= MAX_IPS_PER_ACCOUNT && !uniqueIps.contains(newIp)) {
            logger.warn("账号 {} 已从 {} 个不同 IP 登录，可能是账号共享",
                username, uniqueIps.size());
            return true;
        }
        return false;
    }
}
```

## 6.2 异常登录检测

```java
@Component
public class AbnormalLoginDetector {

    // 短时间内在不同地理位置登录 → 可能账号被盗
    public boolean isAbnormal(String username, GeoLocation currentLocation) {
        // 检查已有的 Session 中的地理位置
        // 如果差距过大且在短时间内 → 异常
        return false; // 示例
    }
}
```

---

# 7. Session 吊销与退出

## 7.1 安全退出（Logout）

```java
@Configuration
public class LogoutConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)     // 使 Session 失效
                .clearAuthentication(true)       // 清除认证信息
                .deleteCookies("SESSION", "JSESSIONID", "remember-me")
                .addLogoutHandler((request, response, auth) -> {
                    // 自定义清理逻辑
                    String sessionId = request.getSession().getId();
                    logger.info("用户注销: sessionId={}", sessionId);
                })
            );
        return http.build();
    }
}
```

## 7.2 远程Session吊销

```java
@RestController
@RequestMapping("/admin")
public class RemoteSessionRevocationController {

    @Autowired
    private FindByIndexNameSessionRepository<?> repository;

    // 管理员吊销用户所有会话
    @PostMapping("/revoke-sessions/{username}")
    public String revokeSessions(@PathVariable String username) {
        Map<String, ? extends Session> sessions = repository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );

        int count = sessions.size();
        sessions.keySet().forEach(repository::deleteById);

        return String.format("已吊销用户 %s 的 %d 个会话", username, count);
    }
}
```

---

# 8. 日志安全

## 8.1 避免记录敏感信息

```java
// 错误：记录完整 Session 内容
logger.info("Session: {}", session); // 可能包含密码、Token

// 正确：仅记录 Session ID
logger.info("Session 创建: {}", session.getId());

// Session 属性日志脱敏
void logSessionAttribute(Session session, String key) {
    Object value = session.getAttribute(key);
    if (isSensitive(key)) {
        logger.debug("Session attr {}: ***", key);
    } else {
        logger.debug("Session attr {}: {}", key, value);
    }
}
```
