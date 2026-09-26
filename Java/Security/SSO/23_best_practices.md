
# 1. 概述

本章汇总 SSO 从设计、开发、部署到运维的全生命周期最佳实践，覆盖协议选择、Token 管理、安全策略、性能优化等各个方面，作为决策速查手册。

# 2. 协议与架构设计

## 2.1 协议选型原则

| 场景 | 推荐协议 | 原因 |
|------|---------|------|
| 企业内部 Java Web 系统 | CAS | 成熟、SLO 原生支持、配置简单 |
| 微服务 + 多端（Web/iOS/Android） | OAuth2 + OIDC + JWT | 无状态、多端统一、云原生 |
| 跨组织联邦认证 | SAML 2.0 | 企业标准、签名完善 |
| B2C 互联网产品 | OIDC | 轻量、支持移动端 |
| 新旧系统混合 | 认证网关 + 多协议 | 统一入口、协议转换 |

## 2.2 架构设计原则

```
① 认证与授权分离
   ├─ 认证中心：集中验证身份 + 签发 Token
   └─ 业务系统：独立判断权限

② 无状态优先
   ├─ Token（JWT）→ 无状态，天然水平扩展
   └─ Session → 需要 Redis 共享

③ 纵深防御
   网络 → 传输 → 认证 → 令牌 → 应用 → 审计
   层层防护，不依赖单一防线

④ 故障降级
   ├─ 认证中心不可用 → 本地缓存
   └─ Redis 不可用 → 回退到本地 Session

⑤ 最小权限
   OAuth2 Scope → 精确控制应用能访问的用户数据范围
   Token 中放最少权限信息
```

# 3. Token 管理实践

## 3.1 推荐 Token 策略

```
Access Token:
├── 格式：JWT（RS256 签名）
├── 有效期：5-15 分钟
├── 内容：sub + roles（精简，< 1KB）
└── 传递：Authorization: Bearer xxx

Refresh Token:
├── 格式：Opaque（随机字符串）
├── 有效期：7-30 天（滑动）
├── 存储：Redis（支持即时吊销）
└── 安全：Rotation + Reuse Detection

ID Token:
├── 格式：JWT
├── 有效期：5-10 分钟
└── 用途：仅身份传递，不用于 API 调用
```

## 3.2 Token 安全规则

| 规则 | 违反后果 |
|------|---------|
| JWT 不要存密码/密钥 | 明文泄露 |
| Token Hash 要验证 | Token 被篡改 |
| jti 要防重放 | Token 可被重放 |
| Token 要短 TTL | 泄露影响范围大 |
| Refresh Token 要 Rotation | Token 泄露无法发现 |
| 密钥要定期轮换 | 长期密钥泄露影响大 |

# 4. 安全实践

## 4.1 Cookie 配置清单

```java
// SSO Cookie 安全最佳配置
Cookie cookie = new Cookie("SSO_TOKEN", token);
cookie.setHttpOnly(true);     // ✅ 必须：防 XSS 窃取
cookie.setSecure(true);       // ✅ 必须：仅 HTTPS
cookie.setPath("/");          // ✅ 推荐：对全路径可见
cookie.setDomain(".example.com"); // ✅ 同域共享
cookie.setMaxAge(7200);       // ✅ 2小时过期
cookie.setAttribute("SameSite", "Lax"); // ✅ SSO 必须用 Lax
response.addCookie(cookie);

// CSP 头（防 XSS）
response.setHeader("Content-Security-Policy",
    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");

// HSTS 头（强制 HTTPS）
response.setHeader("Strict-Transport-Security",
    "max-age=31536000; includeSubDomains; preload");
```

## 4.2 OAuth2 安全清单

```
✅ 使用授权码模式（Authorization Code + PKCE）
✅ 严格校验 redirect_uri（精确匹配 + 白名单）
✅ 每次请求生成新的 state + nonce
✅ 客户端认证用 private_key_jwt（而非 client_secret）
✅ SPA 使用 BFF 模式（不直接获取 Token）
✅ 强制 HTTPS（不接收 HTTP 请求）
✅ 启用 PKCE（即使是有后端的应用）
❌ 不使用隐式模式（Implicit Grant）
❌ 不使用密码模式（Resource Owner Password）
❌ 不使用通配符 redirect_uri
```

# 5. 开发实践

## 5.1 前后端分离的 SSO

```
推荐架构：BFF（Backend For Frontend）模式

Browser (SPA)
    │
    │ HttpOnly Cookie (SSO_SESSION)
    ▼
BFF (Node.js / Spring Boot)
    │
    │ Bearer Token
    ▼
API Gateway
    │
    ▼
Microservices

为什么 BFF：
1. Token 永不在浏览器端存储（防 XSS）
2. BFF 持有 Refresh Token，可静默刷新
3. 前端无需处理 OAuth2 流程
4. 前端 Cookie 由 BFF 统一管理
```

## 5.2 Spring Security 配置模板

```java
@Configuration
@EnableWebSecurity
public class SsoSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. 认证（三选一或共存）
            // .oauth2Login(...)   // OAuth2/OIDC 登录
            // .addFilter(casFilter) // CAS 登录
            // .oauth2ResourceServer(oauth2 -> oauth2.jwt(...))  // JWT 认证

            // 2. 授权
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // 3. Session 管理
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .sessionFixation().migrateSession()
                .maximumSessions(1)
            )

            // 4. 登出
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID", "SSO_SESSION")
            )

            // 5. CSRF（无状态 API 可关闭）
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")
            );

        return http.build();
    }
}
```

# 6. 运维实践

## 6.1 监控指标

```
必须监控的 SSO 指标：

认证中心：
├── 登录 QPS / 成功率 / 失败率
├── 登录响应时间（P50/P95/P99）
├── Token 签发速率
├── Token 验证速率
├── Redis 连接数 / 命中率
├── JVM 内存 / GC
└── 活跃 Session 数

业务系统：
├── Token 验证延迟
├── Token 过期率（需提前告警）
├── 401 错误率
└── 降级模式触发次数

告警规则：
├── 登录失败率 > 5% → 告警
├── P99 登录时间 > 2s → 告警
├── Redis 不可用 → 紧急告警
├── 证书即将过期（30天前） → 告警
└── Token 签发异常增长 → 告警
```

## 6.2 日志规范

```java
// SSO 日志记录要点
// ✅ 需要记录：
log.info("用户登录成功: username={}, ip={}", username, ip);
log.warn("登录失败: username={}, ip={}, reason={}", username, ip, reason);
log.info("Token 刷新: username={}", username);
log.info("用户登出: username={}", username);
log.warn("Token 检测到重复使用（可能泄露）: username={}", username);

// ❌ 绝对不能记录：
log.info("密码: {}", password);               // 明文密码
log.info("Token: {}", accessToken);            // 完整 Token
log.info("密钥: {}", secretKey);               // 密钥
```

## 6.3 证书管理

```
SSL/TLS 证书：
- 使用 Let's Encrypt 自动续签（公网）
- 使用内部 CA（内网）
- 证书到期前 30 天发送告警
- 切换证书时确保新旧共存窗口

JWT 签名密钥：
- 使用 RS256/ES256（非对称）
- 定期轮换（30 天）
- 轮换时保留旧公钥用于验签（7天窗口）
- 私钥存储在 HSM / KMS / Vault
```

# 7. 测试实践

## 7.1 安全测试清单

```
□ OWASP Top 10 渗透测试
□ Token 重放测试
□ CSRF 测试（state 参数验证）
□ redirect_uri 劫持测试
□ Session 固定测试
□ 暴力破解防护测试
□ 速率限制测试
□ SQL 注入测试
□ XSS 测试
□ 密钥泄露应急演练
□ 认证中心故障切换演练
```

## 7.2 性能测试

```java
// JMH 基准测试示例：JWT 验签性能
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
public class JwtVerifyBenchmark {

    @Benchmark
    public Jwt verifyRsa256() {
        // 测试 RS256 JWT 验签吞吐量
        return jwtDecoder.decode(TEST_RSA_TOKEN);
    }

    @Benchmark
    public Jwt verifyEs256() {
        // 测试 ES256 JWT 验签吞吐量
        return jwtDecoder.decode(TEST_ES_TOKEN);
    }
}

// 压力测试场景：
// - 模拟 10000 并发用户同时登录
// - 模拟 50000 RPS Token 验证
// - 模拟 Redis 延迟 50ms 下的表现
// - 模拟认证中心单节点宕机后的切换
```

# 8. 决策速查

| 决策点 | 常见做法 ✅ | 避免 ❌ |
|--------|-----------|--------|
| Token 格式 | JWT（RS256） | 自创格式 / 裸 Base64 |
| Token 有效期 | 15 分钟（Access） / 7 天（Refresh） | 永久有效 / 超过 24h |
| 签名算法 | RS256 / ES256 | HS256（多服务） / alg=none |
| 密钥存储 | HSM / KMS / Vault | 代码中 / 配置文件明码 |
| 前端 Token 存储 | BFF HttpOnly Cookie | localStorage |
| 跨域方案 | OAuth2/OIDC/SAML 标准 | 自定义跨域 Cookie 方案 |
| 登出方案 | CAS SLO / OIDC Logout / MQ 广播 | 仅清除局部 Session |
| Session 共享 | Redis 集中存储 | 粘性路由（无高可用） |
| 限流 | Redis + Sentinel（Lua 脚本原子） | 内存计数（单机） |
| 日志 | 记录事件，不记录密钥/密码/Token全文 | 记录完整 Token |
