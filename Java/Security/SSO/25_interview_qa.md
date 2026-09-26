
# 1. 概述

本章汇总 SSO 领域的高频面试题，涵盖基础概念、协议原理、JWT、Spring Security、分布式 Session、安全攻防等方面，每题提供考点分析和参考答案要点。

# 2. 基础概念

## 2.1 Q1: 什么是 SSO？它解决了什么问题？

**考点**：对 SSO 核心价值的理解

**参考要点**：
- SSO（Single Sign-On）是一种认证机制，用户一次登录即可访问多个相互信任的系统
- 解决的核心问题：多次登录、密码疲劳、管理成本高、安全风险分散
- 核心三要素：认证中心、全局会话、票据/令牌传递
- 典型场景：企业内部门户、微服务架构、第三方登录

## 2.2 Q2: 认证（Authentication）和授权（Authorization）的区别？

**考点**：基础概念的清晰区分

**参考要点**：
- 认证（AuthN）：验证"你是谁"（用户名密码、OTP、生物识别）
- 授权（AuthZ）：判断"你能做什么"（RBAC、ACL、OAuth2 Scope）
- 依赖关系：先认证后授权
- 在 SSO 中：认证中心负责认证，业务系统各自负责授权

## 2.3 Q3: Cookie、Session、Token 三者的关系和区别？

**考点**：对状态管理机制的理解

**参考要点**：
- Cookie：客户端存储机制，浏览器自动携带，受 Domain/Path/SameSite 限制
- Session：服务端存储机制，通过 SessionID（Cookie 传递）关联
- Token：自包含凭证（如 JWT），通常放在 HTTP Header，不依赖 Cookie
- 在 SSO 中：TGC 是 Cookie，TGT 是服务端 Session，ST/JWT 是 Token

# 3. 协议原理

## 3.1 Q4: CAS 协议的核心工作流程？

**考点**：CAS 协议理解

**参考要点**：
1. 用户访问 App → 302 到 CAS（带 service 参数）
2. CAS 检测 TGC Cookie → 无则展示登录页 / 有则直接签发 ST
3. 用户登录 → CAS 生成 TGT + TGC Cookie + ST
4. 302 回 App（带 ticket=ST）
5. App 后台验证 ST → 返回用户信息
6. App 创建局部 Session

关键票据：TGC（Cookie）、TGT（服务端）、ST（一次性）
安全机制：ST 一次性使用 + 短生命周期

## 3.2 Q5: OAuth 2.0 授权码模式为什么比隐式模式更安全？

**考点**：OAuth2 安全设计理解

**参考要点**：
1. 授权码模式：code 在前端传递（不安全信道），换取 Token 在后端（安全信道）
2. 隐式模式：Token 直接在 URL Fragment 返回，暴露在前端
3. 三个安全关键：
   - code 是一次性的、短生命周期的
   - client_secret 从不暴露给前端
   - PKCE 即使 code 被截获也无法使用
4. OAuth 2.1 已移除隐式模式

## 3.3 Q6: OIDC 与 OAuth 2.0 的关系？

**考点**：对两者边界的清晰理解

**参考要点**：
- OAuth 2.0 是**授权**框架（Authorization）
- OIDC 是 OAuth 2.0 之上的**认证**层（Authentication）
- OIDC 增加了：
  - ID Token（JWT 格式的身份凭证）
  - UserInfo 端点（标准化的用户信息接口）
  - Discovery 机制（自动发现 IdP 配置）
  - Session Management
- 一句话：OIDC = OAuth 2.0 + 身份认证

## 3.4 Q7: SAML 2.0 vs CAS vs OIDC，选型考虑？

**考点**：协议选型的工程判断力

**参考要点**：
- SAML 2.0：企业级联邦身份、跨组织、XML 签名完善 → 传统企业/B2B
- CAS：Java 企业内部系统、SLO 支持好、配置简单 → 教育/企业内部
- OIDC：现代 Web+移动端、JSON/JWT、云原生 → 新系统首选
- 混合场景：Keycloak 等多协议网关适配

# 4. JWT 相关

## 4.1 Q8: JWT 的结构和签名验证过程？

**考点**：JWT 底层理解

**参考要点**：
- 结构：`Header.Payload.Signature`（Base64URL 编码）
- Header：`{"alg":"RS256","typ":"JWT"}`
- Payload：Claims（iss/sub/aud/exp/iat 等）
- Signature：`RSA-SHA256(base64(header) + "." + base64(payload), privateKey)`
- 验证：用公钥重新计算签名 → 对比是否一致

## 4.2 Q9: JWT 如何实现登出（Token 吊销）？

**考点**：JWT 的局限性及解决方案

**参考要点**：
- JWT 天然无法"撤销"（自包含，无状态）
- 解决方案：
  1. **黑名单**：jti + Redis，TTL = Token 剩余有效期（有延迟）
  2. **短 TTL**：5-15 分钟，配合 Refresh Token（用户可接受）
  3. **Refresh Token Rotation** + Reuse Detection
  4. **MQ 广播登出** + 各系统清理 Session
- 面试加分：承认 JWT 的吊销是弱项，对比 Session 的即时吊销

## 4.3 Q10: JWT 签名算法 HS256 和 RS256 的区别？如何选择？

**考点**：对称 vs 非对称加密的工程理解

**参考要点**：
- HS256（HMAC-SHA256）：对称签名，单密钥
  - 优点：速度快
  - 缺点：密钥必须共享，任何持有者都能签发
  - 适用：单服务 / 简单场景
- RS256（RSA-SHA256）：非对称签名，公私钥对
  - 优点：公钥可安全分发，私钥保密
  - 缺点：签名体较大，验签稍慢
  - 适用：微服务架构 / IdP（推荐）
- **alg=none 攻击**：必须固定算法，拒绝 none

## 4.4 Q11: JWT 和 Session 的对比？什么时候用哪个？

**考点**：两种模式的取舍

**参考要点**：

| 维度 | JWT | Session |
|------|-----|---------|
| 扩展性 | 天然水平扩展 | 需 Redis 共享 |
| 吊销 | 需黑名单（延迟） | 直接删除（即时） |
| 性能 | 验签 CPU 消耗 | Redis 网络 I/O |
| 大小 | 较大（1-5KB） | Cookie 仅 SessionID |
| 跨域 | HTTP Header 天然支持 | Cookie 受限 |
| 适用 | 微服务、API 网关、多端 | 传统 Web、单体应用 |

# 5. Spring Security

## 5.1 Q12: Spring Security 的过滤器链是怎样的？

**考点**：对安全框架架构的理解

**参考要点**：
- 核心：`FilterChainProxy` → `SecurityFilterChain` 列表
- 关键过滤器（按顺序）：
  1. `SecurityContextPersistenceFilter`：Session 反序列化
  2. `CsrfFilter`：CSRF 防护
  3. `LogoutFilter`：登出
  4. 认证过滤器（UsernamePassword/OAuth2/CAS/BearerToken）
  5. `ExceptionTranslationFilter`：异常转换
  6. `FilterSecurityInterceptor`：最终授权
- 多重过滤器链：`/api/**` 用 JWT，`/web/**` 用 Session

## 5.2 Q13: 如何在 Spring Security 中同时支持 JWT（API）和 Session（Web 页面）？

**考点**：混合认证模式

**参考要点**：
- 使用多个 `SecurityFilterChain`，配置不同 `securityMatcher`
- API 链：`securityMatcher("/api/**")` + `oauth2ResourceServer().jwt()` + `STATELESS`
- Web 链：`securityMatcher("/web/**")` + `formLogin()` 或 `oauth2Login()` + `IF_REQUIRED`
- `@Order` 控制匹配优先级

## 5.3 Q14: Spring Security OAuth2 Client 自动做了哪些事？

**考点**：对框架内部机制的理解

**参考要点**：
- `OAuth2AuthorizationRequestRedirectFilter`：构建授权 URL + 302 重定向
- `OAuth2LoginAuthenticationFilter`：拦截回调请求
- `OAuth2LoginAuthenticationProvider`：用 code 换 token + 获取 userinfo
- 自动创建 `OAuth2AuthenticationToken` 并存入 `SecurityContextHolder`
- 自动管理 `OAuth2AuthorizedClient`（Token 缓存和刷新）

# 6. 分布式与高可用

## 6.1 Q15: 分布式环境下如何共享 Session？

**考点**：分布式 Session 解决方案

**参考要点**：
- 方案1：粘性路由（ip_hash）- 简单但不可靠
- 方案2：Session 复制 - 网络开销大
- 方案3：**Redis 集中存储**（Spring Session） - **推荐**
  - 优点：无状态节点，水平扩展，支持即时失效
  - 实现：`spring-session-data-redis` + Jackson 序列化
- 方案4：JWT 无状态 - 无需 Session

## 6.2 Q16: 认证中心如何做高可用？

**考点**：高可用架构设计

**参考要点**：
- 应用层：多实例部署（>=3），无状态设计
- 负载均衡：Nginx/HAProxy + 健康检查 + 自动摘除
- 存储层：Redis Sentinel/Cluster + 数据库主从
- 配置：统一配置中心，动态下发改
- 降级：应用层缓存 + 降级开关
- 多活：同城双活 + 异地灾备

## 6.3 Q17: 如何实现单点登出（SLO）？

**考点**：SLO 实现能力

**参考要点**：
1. CAS SLO：原生支持，Server 通知所有 Client
2. SAML SLO：IdP 发送 LogoutRequest
3. OIDC RP-Initiated Logout + Back-Channel Logout
4. MQ 广播：认证中心发布登出事件，各系统消费
5. Token 黑名单：JWT jti 加入黑名单
6. 短 TTL + 不实现 SLO（最简方案）

# 7. 安全攻防

## 7.1 Q18: SSO 中常见的 CSRF 攻击如何防御？

**考点**：CSRF 攻击与防御

**参考要点**：
- OAuth2 登录 CSRF：攻击者用自己 code 诱导受害者访问 callback
  - 防御：state 参数（随机 + Session 校验）
- 登出 CSRF：攻击者诱导用户点击登出链接
  - 防御：登出用 POST + CSRF Token / SameSite Cookie
- 通用防御：CSRF Token、SameSite=Lax、Referer 校验

## 7.2 Q19: XSS 如何窃取 Token？如何防御？

**考点**：XSS 攻击在 SSO 场景的防护

**参考要点**：
- 攻击：注入脚本 → `document.cookie` / `localStorage.getItem("token")` / 监听 postMessage
- 防御：
  - Cookie：HttpOnly（JS 不可读）+ Secure（仅 HTTPS）
  - Token：不存 localStorage，使用 BFF 模式
  - 响应头：CSP（限制脚本来源）
  - 输入：对所有用户输入做 HTML 转义

## 7.3 Q20: 什么是 Session Fixation 攻击？如何防御？

**考点**：Session 安全

**参考要点**：
- 攻击：攻击者获取 SessionID → 诱导受害者用此 SessionID 登录 → 攻击者劫持会话
- 防御：**认证成功后更换 SessionID**
  - Spring Security 默认启用：`sessionFixation().migrateSession()`
  - 更好的：`sessionFixation().newSession()`（全新 Session）

## 7.4 Q21: Refresh Token Rotation 和 Reuse Detection 的原理？

**考点**：高级 Token 安全

**参考要点**：
- Rotation：每次使用 Refresh Token 后，旧 Token 立即失效，返回新 Token
- Reuse Detection：如果旧的 Refresh Token 被重复使用 → 说明 Token 泄露
  - 响应：撤销整个 Token 家族
- 优势：即使 Refresh Token 泄露，正常用户下次刷新时也能检测到异常

# 8. 场景设计

## 8.1 Q22: 设计一个支持 10 万并发的 SSO 系统？

**考点**：系统设计能力

**参考要点**：
1. 架构：认证中心（OIDC）+ 网关 + 微服务
2. Token：JWT（RS256）+ 短 TTL（5min）+ Refresh Token
3. 缓存：JWT 公钥本地缓存 + 用户信息多级缓存（Caffeine → Redis → DB）
4. 存储：Redis Cluster（Ticket/Session/黑名单）+ MySQL 主从（用户数据）
5. 扩展：无状态认证中心水平扩展 + Nginx 负载均衡
6. 性能：BCrypt 强度调整、数据库索引、连接池优化
7. 安全：HTTPS + HSTS + CSP + MFA + 速率限制
8. 监控：Prometheus + Grafana + 告警规则

## 8.2 Q23: 如何设计一个支持多租户的 SSO 系统？

**考点**：多租户架构

**参考要点**：
1. 租户隔离：tenant_id 在 JWT 中，资源服务做租户校验
2. 多 IdP：每个租户可有自己的 IdP（SAML/OIDC）
3. 数据隔离：Ticket/Session Key 加租户前缀
4. 域名策略：`tenant1.sso.com` 或统一域名 + 租户选择页
5. 权限：租户级角色 + 系统级角色的双层 RBAC

# 9. 其他高频题

## 9.1 Q24: SameSite Cookie 属性在 SSO 中的重要性？

**参考要点**：
- `SameSite=Strict`：302 重定向不携带 Cookie → SSO 完全失效 ❌
- `SameSite=Lax`：GET 顶级导航携带 Cookie → SSO 正常工作 ✅
- `SameSite=None + Secure`：所有请求携带 → SSO 正常工作 ✅（但 CSRF 风险）
- SSO 最佳选择：**Lax**

## 9.2 Q25: HTTPS 在 SSO 中为什么是必须的？

**参考要点**：
- Token/Cookie 明文传输 = 裸奔
- 中间人攻击：拦截 Token → 冒用身份
- SSL Stripping：降级到 HTTP → 窃取
- HSTS：强制浏览器使用 HTTPS
- 结论：**生产环境 SSO 如无 HTTPS = 完全不设防**

## 9.3 Q26: 如何实现"记住我"功能与 SSO 的结合？

**参考要点**：
- 传统：RememberMe Cookie（持久化 Token）+ PersistentTokenRepository
- SSO 中：延长 TGC/Refresh Token 的 TTL
- CAS：`tgt.maxTimeToLiveInSeconds` 设为更长时间
- OAuth2/OIDC：Refresh Token 设为长期（30 天+）
- 安全：加 HttpOnly + Secure + 设备绑定

## 9.4 Q27: Token 过期前端如何无感刷新？

**参考要点**：
- 方案1：前端 Axios 拦截器 401 → 自动发 refresh 请求 → 重试
- 方案2：BFF 模式 → Token 在后端刷新，前端无感知
- 方案3：定时器提前 N 秒静默刷新
- 方案4：iframe 静默刷新（OIDC prompt=none）
- 注意：避免并发刷新（加锁/防抖）

## 9.5 Q28: OAuth 2.1 相比 OAuth 2.0 有哪些变化？

**参考要点**：
1. 授权码模式 + PKCE 成为唯一推荐的流程
2. 移除隐式模式（Implicit Grant）
3. 移除密码模式（Resource Owner Password Credentials）
4. redirect_uri 必须精确匹配
5. Refresh Token 必须支持发送者约束或一次性使用
6. 新增 DPoP 支持

## 9.6 Q29: 如何防止 SSO Token 在日志中泄露？

**参考要点**：
- 日志脱敏：对 Authorization Header 做 mask 处理
- 只记录 Token 前 8 个字符（用于调试排重）
- 不记录完整 Token 字符串
- 接入日志系统前做过滤
- JSON 日志排印用 mask pattern

## 9.7 Q30: SSO 认证中心挂了怎么办？

**参考要点**：
- 事前：高可用部署（集群 + 多活）
- 事中：降级策略
  - 业务系统使用本地 Token 缓存（已登录用户继续可用）
  - 新用户拒绝登录（友好提示）
  - 关键系统开启离线模式
- 事后：快速恢复 + 复盘

# 10. 面试准备建议

| 层级 | 准备方向 |
|------|---------|
| **初级** | SSO 概念、CAS/OAuth2/OIDC 协议流程、JWT 结构、Cookie/Session/Toke 区别 |
| **中级** | Spring Security 集成、Token 管理、SLO 实现、分布式 Session、常见安全攻击 |
| **高级** | 架构设计（高并发/高可用/多租户）、Token 泄露检测、密钥管理、多活部署 |
| **专家** | 源码级理解（CAS/OAuth2 协议实现）、性能调优、零信任架构、FIDO2/WebAuthn |
