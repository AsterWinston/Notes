
# 1. 概述

单点登出（SLO）是指用户在一个系统登出后，所有已登录的系统同时失效登录态。SLO 是 SSO 的配套能力，也是实现中最容易被忽视和最难完美实现的部分。

```
┌──────────────────────────────────────────────────────────────────┐
│                    SLO 场景要求                                  │
│                                                                   │
│  用户登出系统 A                                                    │
│       │                                                           │
│       ├── 系统 A 的局部 Session 被销毁 ✓                           │
│       ├── 系统 B 的局部 Session 也应被销毁 ✓                       │
│       ├── 系统 C 的局部 Session 也应被销毁 ✓                       │
│       └── 认证中心的全局 Session 被销毁 ✓                          │
│                                                                   │
│  下次用户再访问系统 B → 显示为未登录状态 ✓                         │
└──────────────────────────────────────────────────────────────────┘
```

# 2. SLO 实现方案

## 2.1 方案对比

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **CAS SLO** | CAS Server 通知所有 Client | 原生支持，成熟稳定 | 仅 CAS 协议 |
| **SAML SLO** | IdP 发送 LogoutRequest 到各 SP | 标准协议 | 配置复杂 |
| **OIDC RP-Initiated Logout** | RP 主动请求 IdP 登出 | 标准化 | 需 IdP 支持 |
| **消息广播** | 认证中心通过 MQ 通知各系统 | 解耦，可靠 | 需基础设施 |
| **Token 黑名单** | JWT jti 加入黑名单 | 实现简单 | 有延迟（Token 过期前） |
| **短 TTL + 不实现 SLO** | Token 足够短，无需登出 | 最简单 | 用户体验可能差 |

## 2.2 方案选择建议

```
场景 1: CAS 协议 → 使用 CAS SLO（原生支持，最完善）
场景 2: SAML 协议 → 使用 SAML SLO
场景 3: OIDC 协议 → 使用 RP-Initiated Logout（OIDC 标准）
场景 4: 自建 JWT 系统 → Token 黑名单 + MQ 广播 + 短 TTL
场景 5: 可容忍延迟 → 短 TTL Access Token（无需专门 SLO）
```

# 3. CAS SLO 实现

## 3.1 CAS SLO 流程

```
SP-Initiated SLO（用户在 SP 点击登出）:

SP A              CAS Server              SP B              SP C
 │                     │                    │                │
 │ ① 销毁局部 Session   │                    │                │
 │ ② 302→CAS /logout   │                    │                │
 │────────────────────►│                    │                │
 │                     │ ③ 销毁 TGT         │                │
 │                     │ ④ 清除 TGC Cookie  │                │
 │                     │                    │                │
 │                     │ ⑤ 异步通知各 SP     │                │
 │                     │    POST SLO Request│                │
 │                     │───────────────────►│                │
 │                     │───────────────────────────────────►│
 │                     │                    │                │
 │                     │                    │ ⑥ 销毁局部     │
 │                     │                    │    Session     │
 │                     │                    │ ⑦ 返回 200 OK  │
 │                     │◄───────────────────│                │
 │                     │◄───────────────────────────────────│
 │                     │                    │                │
 │                     │ ⑧ 展示登出成功页（或跳回 SP A）     │
 │◄────────────────────│                    │                │
```

## 3.2 CAS Client SLO 实现

```java
/**
 * CAS Client 端 SLO 配置（Spring Security）
 */
@Configuration
@EnableWebSecurity
public class CasSloConfig {

    @Bean
    public SingleSignOutFilter singleSignOutFilter() {
        SingleSignOutFilter filter = new SingleSignOutFilter();
        return filter;
    }

    @Bean
    public LogoutFilter casLogoutFilter() {
        // 本地登出后重定向到 CAS Server 的 SLO
        String logoutRedirectUrl = casServerUrlPrefix + "/logout";

        LogoutFilter logoutFilter = new LogoutFilter(
            logoutRedirectUrl,
            new SecurityContextLogoutHandler(),
            new CookieClearingLogoutHandler("JSESSIONID", "MYAPP_SESSION")
        );
        logoutFilter.setFilterProcessesUrl("/logout");
        return logoutFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
            SingleSignOutFilter ssoFilter,
            LogoutFilter logoutFilter) throws Exception {
        http
            .addFilterBefore(ssoFilter, CasAuthenticationFilter.class)
            .addFilterBefore(logoutFilter, LogoutFilter.class)
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl(casServerUrlPrefix + "/logout")
            );
        return http.build();
    }
}
```

## 3.3 CAS Server SLO 配置

```json
// CAS Server 端 Service 注册
{
  "@class": "org.apereo.cas.services.RegexRegisteredService",
  "serviceId": "^https://myapp\\.example\\.com/.*",
  "name": "My Application",
  "id": 1001,
  "logoutType": "BACK_CHANNEL",
  "logoutUrl": "https://myapp.example.com/logout",
  "attributeReleasePolicy": {
    "@class": "org.apereo.cas.services.ReturnAllAttributeReleasePolicy"
  }
}
```

# 4. OIDC RP-Initiated Logout

## 4.1 流程

```
用户点击登出
     │
     ▼
GET /logout?
    id_token_hint=eyJhbGciOi...  ← 当前用户的 ID Token
    &post_logout_redirect_uri=   ← 登出后跳转地址
      https://myapp.com/logged_out
    &state=xxx                   ← 防 CSRF

IdP 收到后:
1. 验证 ID Token（签名、过期时间）
2. 销毁 IdP Session
3. 可选：通知其他 RP 登出（Back-Channel Logout）
4. 302 重定向到 post_logout_redirect_uri
```

## 4.2 Spring Security 实现

```java
/**
 * OIDC 登出配置
 */
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
    public OidcClientInitiatedLogoutSuccessHandler oidcLogoutSuccessHandler(
            ClientRegistrationRepository clientRegistrationRepository) {

        OidcClientInitiatedLogoutSuccessHandler handler =
            new OidcClientInitiatedLogoutSuccessHandler(
                clientRegistrationRepository);
        handler.setPostLogoutRedirectUri(
            "{baseUrl}/logged-out");  // 登出后跳转 URL
        return handler;
    }
}
```

# 5. 消息广播 SLO（通用方案）

## 5.1 架构

```
认证中心
  │
  │ 用户登出事件
  ▼
消息队列（RabbitMQ / Kafka）
  │
  ├────► 系统 A: 消费登出事件 → 销毁局部 Session
  ├────► 系统 B: 消费登出事件 → 销毁局部 Session
  └────► 系统 C: 消费登出事件 → 销毁局部 Session
```

## 5.2 实现

```java
// ========== 登出事件 ==========
@Data
@Builder
public class LogoutEvent implements Serializable {
    private String userId;
    private String sessionId;
    private List<String> tokenIds;  // 要吊销的 Token jti 列表
    private Instant timestamp;
    private String reason;  // "user_logout" | "admin_kick" | "password_changed"
}

// ========== 发送方（认证中心） ==========
@Service
public class LogoutEventPublisher {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    private static final String LOGOUT_EXCHANGE = "sso.logout";
    private static final String LOGOUT_ROUTING_KEY = "logout.broadcast";

    /**
     * 发布登出事件
     */
    public void publishLogout(String userId, List<String> tokenIds, String reason) {
        LogoutEvent event = LogoutEvent.builder()
            .userId(userId)
            .tokenIds(tokenIds)
            .timestamp(Instant.now())
            .reason(reason)
            .build();

        rabbitTemplate.convertAndSend(LOGOUT_EXCHANGE, LOGOUT_ROUTING_KEY, event);
        log.info("已发布登出事件: userId={}, tokens={}", userId, tokenIds.size());
    }
}

// ========== 接收方（各应用系统） ==========
@Component
@RabbitListener(queues = "sso.logout.queue")
public class LogoutEventConsumer {

    @Autowired
    private TokenBlacklistService blacklistService;

    @Autowired
    private SessionRepository<?> sessionRepository;

    @RabbitHandler
    public void handleLogoutEvent(LogoutEvent event) {
        log.info("收到登出事件: userId={}, reason={}",
            event.getUserId(), event.getReason());

        // 1. 将 Token 加入黑名单
        if (event.getTokenIds() != null) {
            for (String jti : event.getTokenIds()) {
                blacklistService.blacklist(jti,
                    Date.from(event.getTimestamp().plus(Duration.ofMinutes(15))));
            }
        }

        // 2. 销毁该用户的所有本地 Session
        if (event.getUserId() != null) {
            Map<String, ? extends Session> sessions = sessionRepository
                .findByPrincipalName(event.getUserId());
            for (String sessionId : sessions.keySet()) {
                sessionRepository.deleteById(sessionId);
            }
        }
    }
}
```

# 6. Token 黑名单 SLO

## 6.1 原理

```
JWT 无法主动失效 → 引入黑名单机制

登出时：
1. 提取当前 Access Token 的 jti
2. 将 jti 加入 Redis 黑名单
3. 设置 TTL = Token 剩余有效期

每次请求时：
1. 验证 JWT 签名
2. 检查 jti 是否在黑名单中
3. 在黑名单中 → 拒绝请求（401）

局限性：
- 登出后 Token 在 TTL 内仍可访问（取决于黑名单查询）
- 无法"即时"登出（但通常 TTL 短到可接受）
- 需要 Redis 查询（增加延迟）
```

## 6.2 批量登出

```java
/**
 * 批量登出：管理员踢出某用户 / 密码修改后撤销所有 Token
 */
@Service
public class BatchLogoutService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 撤销某用户的所有 Token
     */
    public void revokeAllTokens(String username) {
        // 1. 找到该用户的所有 Refresh Token
        Set<String> refreshTokens = findUserRefreshTokens(username);

        // 2. 删除所有 Refresh Token
        for (String token : refreshTokens) {
            redisTemplate.delete("refresh:" + token);
        }

        // 3. 发布登出广播（MQ），让各系统清理 Session
        logoutEventPublisher.publishLogout(
            username, null, "admin_revoke_all");

        log.info("已撤销用户 {} 的所有 Token ({} 个)", username, refreshTokens.size());
    }

    /**
     * 密码修改后撤销所有 Token（除了当前）
     */
    public void revokeOtherTokens(String username, String currentJti) {
        // 同上，但保留当前的 Token
        revokeAllTokens(username);
        // 移除当前 jti 的黑名单（如果有）
        redisTemplate.delete("token_blacklist:" + currentJti);
    }

    private Set<String> findUserRefreshTokens(String username) {
        // 实际中需要维护 username → refresh tokens 的索引
        Set<String> tokens = redisTemplate.opsForSet()
            .members("user_refresh_tokens:" + username);
        return tokens != null ? tokens : Collections.emptySet();
    }
}
```

# 7. Back-Channel Logout

## 7.1 协议规范

```
OIDC Back-Channel Logout 规范：

IdP 在用户登出时，向后端直接 POST 登出通知到 RP：

POST https://rp.example.com/logout/backchannel
Content-Type: application/x-www-form-urlencoded

logout_token=eyJhbGciOi...[JWT]

logout_token 的 Payload：
{
    "iss": "https://idp.example.com",
    "sub": "user-12345",
    "aud": "rp-client-id",
    "iat": 1712345678,
    "jti": "unique-logout-event-id",
    "events": {
        "http://schemas.openid.net/event/backchannel-logout": {}
    }
}
```

## 7.2 Spring Security 实现

```java
/**
 * Back-Channel Logout 端点
 */
@RestController
@RequestMapping("/logout")
public class BackChannelLogoutController {

    @Autowired
    private SessionRepository<?> sessionRepository;

    @PostMapping("/backchannel")
    public ResponseEntity<Void> handleBackchannelLogout(
            @RequestParam("logout_token") String logoutToken) {

        // 1. 验证 logout_token JWT
        Claims claims = jwtService.parseToken(logoutToken);

        // 2. 提取用户标识
        String sub = claims.getSubject();

        // 3. 销毁该用户的所有 Session
        Map<String, ? extends Session> sessions = sessionRepository
            .findByPrincipalName(sub);
        for (String sessionId : sessions.keySet()) {
            sessionRepository.deleteById(sessionId);
        }

        log.info("Back-Channel Logout: 用户 {} 的 {} 个 Session 已销毁",
            sub, sessions.size());

        return ResponseEntity.ok().build();
    }
}
```

# 8. SLO 失败处理

## 8.1 失败重试

```java
@Component
public class SloRetryHandler {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final int MAX_RETRIES = 3;
    private static final Duration RETRY_INTERVAL = Duration.ofSeconds(30);

    /**
     * SLO 通知失败时，将任务存入 Redis 延迟队列
     */
    public void scheduleRetry(String serviceUrl, LogoutEvent event, int attempt) {
        if (attempt > MAX_RETRIES) {
            log.error("SLO 通知失败达到最大重试次数: {} → {}", event, serviceUrl);
            // 发送告警
            alertService.sendSloFailureAlert(serviceUrl, event);
            return;
        }

        RetryTask task = new RetryTask(serviceUrl, event, attempt);
        redisTemplate.opsForZSet().add(
            "slo_retry_queue",
            task.toJson(),
            System.currentTimeMillis() + RETRY_INTERVAL.toMillis()
        );
    }

    /**
     * 定时处理重试任务
     */
    @Scheduled(fixedDelay = 10000) // 每 10 秒
    public void processRetries() {
        Set<String> tasks = redisTemplate.opsForZSet()
            .rangeByScore("slo_retry_queue", 0,
                System.currentTimeMillis());

        for (String taskJson : tasks) {
            RetryTask task = RetryTask.fromJson(taskJson);
            try {
                sendSloNotification(task.getServiceUrl(), task.getEvent());
                // 成功，移除重试任务
                redisTemplate.opsForZSet()
                    .remove("slo_retry_queue", taskJson);
            } catch (Exception e) {
                // 失败，增加重试次数
                scheduleRetry(task.getServiceUrl(), task.getEvent(),
                    task.getAttempt() + 1);
            }
        }
    }
}
```

# 9. 总结

| 方案 | 适用协议 | 实时性 | 复杂度 | 推荐度 |
|------|---------|--------|--------|--------|
| **CAS SLO** | CAS | 高 | 低 | ⭐⭐⭐⭐⭐ |
| **SAML SLO** | SAML 2.0 | 高 | 高 | ⭐⭐⭐⭐ |
| **OIDC RP-Initiated** | OIDC | 中 | 中 | ⭐⭐⭐⭐ |
| **Back-Channel Logout** | OIDC | 高 | 中 | ⭐⭐⭐⭐ |
| **MQ 广播** | JWT / 自建 | 高 | 中 | ⭐⭐⭐⭐ |
| **Token 黑名单** | JWT | 中（有延迟） | 低 | ⭐⭐⭐ |
| **短 TTL（不实现 SLO）** | JWT | 低 | 最低 | ⭐⭐ |
