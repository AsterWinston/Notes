# 1. 常见问题与排查指南

## 1.1 授权流程问题

### Q1: redirect_uri 不匹配错误

```
错误信息：error=redirect_uri_mismatch

原因：
1. 授权请求中的 redirect_uri 与注册的不一致
2. 注册了多个 redirect_uri 但请求中没有指定
3. 使用了 HTTP 而注册的是 HTTPS
4. 路径末尾多了/少了斜杠
5. 端口号不一致（开发:8080 vs 生产:443）

排查步骤：
□ 打印请求中的 redirect_uri
□ 打印数据库中注册的 redirect_uri
□ 确认 full URI 严格一致（包括 protocol + host + port + path）
□ 确认使用 RFC 3986 编码
```

### Q2: invalid_grant 错误（Token 端点）

```
错误信息：error=invalid_grant

常见原因：
□ Authorization Code 已过期（超过 60 秒）
□ Authorization Code 已被使用过（重放）
□ Authorization Code 与 client_id 不匹配
□ redirect_uri 与授权请求中的不一致
□ PKCE code_verifier 不正确
□ Refresh Token 已过期或被撤销
□ grant_type 参数不对

排查：查看授权服务器日志中的具体验证失败点
```

### Q3: CORS 错误（SPA 直接请求 Token 端点）

```
错误：Access to fetch at 'https://auth.example.com/token' blocked by CORS policy

原因：浏览器阻止了跨域 POST 请求

解决方案：
1. 授权服务器配置 CORS：
   - Access-Control-Allow-Origin: <你的 SPA 域名>
   - Access-Control-Allow-Methods: POST
   - Access-Control-Allow-Headers: Content-Type, Authorization

2. 或使用 BFF 模式（推荐）：
   - SPA → BFF → Token Endpoint
   - 不存在跨域问题
```

## 1.4 Token 相关问题

### Q4: Access Token 总是过期，用户需要频繁登录

```
根本原因：没有正确使用 Refresh Token 自动续期

解决方案：
1. 客户端实现 Token 自动刷新：
   - 请求前检查 Token 是否即将过期
   - 提前 60 秒调用 Refresh Token 换取新 Token
   - 使用定时器或拦截器自动处理

2. 使用 OAuth2AuthorizedClientManager（Spring Security）：
   - 自动管理 Token 刷新
   
3. BFF 模式：
   - BFF 端透明处理刷新
   - 前端使用长期 Session Cookie
```

### Q5: JWT 签名验证失败

```
错误：JWT signature verification failed

可能原因与排查：

1. 公钥不匹配
   □ 检查 JWKS 端点返回的公钥是否是最新的
   □ 密钥是否已轮换（旧 Token 用旧公钥，新 Token 用新公钥）
   □ JWKS 缓存是否过旧（未及时更新）

2. 算法不匹配
   □ JWT Header 中的 alg 与实际使用的算法一致？
   □ 授权服务器用 RS256，资源服务器配置的是 HS256？
   □ 是否被降级攻击（alg: none）？

3. Token 被篡改
   □ 验证 Token 的完整性（三部分是否完整）
   □ HTTPS 中间人代理修改了 Token？

排查代码：
```

```java
// 调试 JWT 验证
public void debugJwtValidation(String token) {
    try {
        // 1. 解析 JWT（不验证）
        SignedJWT jwt = SignedJWT.parse(token);
        
        System.out.println("Header: " + jwt.getHeader());
        System.out.println("Algorithm: " + jwt.getHeader().getAlgorithm());
        System.out.println("Key ID: " + jwt.getHeader().getKeyID());
        
        // 2. 检查 Claims
        JWTClaimsSet claims = jwt.getJWTClaimsSet();
        System.out.println("Issuer: " + claims.getIssuer());
        System.out.println("Audience: " + claims.getAudience());
        System.out.println("Expiration: " + claims.getExpirationTime());
        System.out.println("Issue Time: " + claims.getIssueTime());
        
        // 3. 从 JWKS 获取公钥
        JWKSet jwkSet = JWKSet.load(new URL(jwksUri));
        JWK jwk = jwkSet.getKeyByKeyId(jwt.getHeader().getKeyID());
        System.out.println("JWK found: " + (jwk != null));
        System.out.println("JWK algorithm: " + jwk.getAlgorithm());
        
    } catch (Exception e) {
        System.err.println("Debug failed: " + e.getMessage());
    }
}
```

### Q6: Refresh Token 轮换导致的问题

```
问题：开启 Refresh Token 轮换后，某些请求失败

场景一：并发刷新
  线程 A 使用 RT1 → 获得 AT2 + RT2（RT1 失效）
  线程 B 同时使用 RT1 → 失败（已被撤销）
  
解决：
  - 使用分布式锁保护 Refresh Token 刷新操作
  - Token 刷新串行化

场景二：网络问题导致 RT 丢失
  客户端用 RT1 换取 RT2，但响应丢失
  客户端仍持有 RT1 → 重试 → RT1 已被撤销
  → 触发重放检测 → 整个家族失效 → 必须重新授权

解决：
  - 客户端应妥善保存新的 Refresh Token
  - 可能触发重新授权（让用户重新登录）是预期行为
```

---

# 2. 面试高频问题

## 2.1 概念与基础

### Q: OAuth2 是什么？解决什么问题？

```
OAuth2 是一个授权框架协议，允许第三方应用在不知道用户密码的前提下，
获得用户对受保护资源的有限访问权限。

核心解决的问题：
- 密码反模式（第三方应用不应该知道用户密码）
- 授权可控制（限范围、限时间、可撤销）
- 权限分离（认证 vs 授权）
```

### Q: OAuth2 的四种角色？

```
1. Resource Owner（资源拥有者）— 通常是用户
2. Resource Server（资源服务器）— 托管受保护资源
3. Client（客户端）— 请求访问资源的应用
4. Authorization Server（授权服务器）— 颁发 Token
```

### Q: 四种授权模式及使用场景？

```
1. Authorization Code（授权码模式）
   场景：有后端的 Web 应用（机密客户端）
   特点：最安全、推荐；Token 不经过浏览器

2. Client Credentials（客户端凭证模式）
   场景：M2M、微服务间通信
   特点：无用户参与

3. Implicit（隐式模式）
   场景：已不推荐（被 Authentication Code + PKCE 替代）

4. Resource Owner Password（密码模式）
   场景：已不推荐（违反 OAuth2 设计理念）
   替代：Authentication Code + PKCE
```

### Q: OAuth2 和 OIDC 的区别？

```
OAuth2：授权协议 — "这个第三方应用可以访问你的什么资源"
OIDC： 认证协议 — "这个用户是谁"

OAuth2：产出 Access Token（访问资源用）
OIDC：  产出 ID Token（证明身份用）+ Access Token

OAuth2：Access Token 的受众是资源服务器
OIDC：  ID Token 的受众是客户端

OIDC 是建立在 OAuth2 之上的身份认证层
```

### Q: PKCE 是什么？为什么需要它？

```
PKCE（Proof Key for Code Exchange）是对 Authorization Code 模式的安全增强。

工作原理：
1. 客户端生成 code_verifier（随机字符串）
2. 授权请求中发送 code_challenge = SHA256(code_verifier)
3. Token 请求中发送原始 code_verifier
4. 授权服务器验证 code_challenge == SHA256(code_verifier)

为什么需要：
- Authorization Code 在浏览器 URL 中传输（可能被截获）
- 对于公开客户端（SPA、移动 App），client_secret 不可信
- PKCE 提供了额外的、只有原始客户端才知道的秘密
- OAuth 2.1 中 PKCE 是所有 Auth Code 流程的强制要求
```

## 2.2 安全与架构

### Q: JWT Token 如何保证安全？

```
1. 签名验证：使用非对称算法（RS256/ES256），资源服务器验证签名
2. 短有效期：Access Token 15-30 分钟
3. HTTPS 传输：防止中间人窃取
4. 算法白名单：拒绝 "none" 和对称算法
5. 验证所有 Claims：iss, aud, exp, iat, nbf
6. 不在 JWT 中放敏感信息
7. 配合 Refresh Token 轮换
```

### Q: Token 存在服务端还是客户端？

```
按 Token 类型：
- JWT Access Token：存储在客户端（HTTP Header），服务端无状态
- Opaque Access Token：元数据存储在服务端（Redis/DB）
- Refresh Token：存储在服务端，客户端持有引用
- Session/State：存储在服务端（结合 Cookie 的 Session ID）

按客户端类型：
- 机密 Web 后端：Token 存储在后端（数据库加密）
- SPA（推荐 BFF）：Token 存在 BFF，浏览器用 Session Cookie
- 移动 App：Token 存储在 Keychain/KeyStore
```

### Q: 用户修改密码后，如何让所有已颁发的 Token 失效？

```
方案对比：

1. Token 版本号（推荐）
   JWT 中包含 token_version
   用户修改密码 → token_version++
   验证时检查 JWT version == DB version
   优点：即时生效，开销低
   
2. 缩短 Access Token 有效期
   15 分钟有效期 → 最多 15 分钟延迟
   优点：实现简单
   缺点：不是即时撤销
   
3. 黑名单
   维护已撤销的 Token ID 列表
   验证时检查 JWT.jti 是否在黑名单中
   优点：即时
   缺点：需要额外存储和网络调用
   
4. 密钥轮换
   更换签名密钥 → 所有旧 Token 立即失效
   优点：最彻底
   缺点：影响所有用户（不精细）
```

### Q: 如何防止 CSRF 攻击？

```
OAuth2 中的 CSRF 攻击：
攻击者诱导受害者使用攻击者的 authorization code 登录

防御机制 — state 参数：
1. 客户端生成随机 state 值
2. 存储在用户会话中（session / cookie）
3. 授权请求中携带 state
4. 授权服务器在回调中原样返回 state
5. 客户端验证 received_state == stored_state

不匹配 → 拒绝请求，不交换 Token
```

### Q: 如何防止 XSS 窃取 Token？

```
问题：如果 Token 存储在 localStorage 中，XSS 可以轻易窃取

防御策略：
1. 首选 BFF 模式（Token 不出浏览器）
   - 使用 HttpOnly + Secure + SameSite Cookie
   - JS 无法读取 HttpOnly Cookie

2. 降级方案（SPA 无 BFF）
   - Token 仅存内存（不持久化）
   - 页面刷新后通过静默刷新获取新 Token
   - Service Worker 拦截请求注入 Token

3. 通用防护
   - CSP（Content-Security-Policy）限制脚本来源
   - 输入输出编码
   - 第三方脚本审查
```

## 2.3 Spring Security 相关

### Q: Spring Security 中 OAuth2 的核心组件有哪些？

```
1. ClientRegistration — 客户端配置（client-id, client-secret, 端点等）
2. ClientRegistrationRepository — 客户端配置仓库
3. OAuth2AuthorizedClientService — 已授权客户端管理（存储 Token）
4. OAuth2AuthorizedClientManager — Token 自动管理（获取、刷新）
5. OAuth2UserService — 从 UserInfo 端点加载用户信息
6. JwtDecoder — JWT 解码与验证
7. JwtAuthenticationConverter — JWT Claims 转 Authentication
```

### Q: @EnableWebSecurity vs @EnableMethodSecurity？

```
@EnableWebSecurity：启用 Web 安全（URL 级别的访问控制）
  - 配置 SecurityFilterChain
  - URL Pattern 匹配授权

@EnableMethodSecurity：启用方法级安全
  - @PreAuthorize / @PostAuthorize
  - @Secured / @RolesAllowed
  - SpEL 表达式支持

两者可以同时使用，各自作用于不同层级。
```

### Q: 如何实现方法级权限控制？

```java
@EnableMethodSecurity  // 启用方法级安全

// 基于角色
@PreAuthorize("hasRole('ADMIN')")

// 基于权限
@PreAuthorize("hasAuthority('user:delete')")

// 基于 Scope
@PreAuthorize("hasAuthority('SCOPE_user:write')")

// 基于复杂表达式
@PreAuthorize("@securityService.canAccessOrder(#orderId, authentication)")

// 过滤返回值
@PostAuthorize("returnObject.owner == authentication.principal.username")
```

## 2.4 场景设计题

### Q: 设计一个支持第三方登录的系统

```
架构：
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  SPA/BFF │────>│  应用服务器    │────>│  授权服务器    │
│          │     │              │     │  (自建/Keycloak)│
└──────────┘     └──────┬───────┘     └──────┬───────┘
                        │                     │
                        │              ┌──────┴────────┐
                        │              │  第三方 IdP     │
                        │              │ GitHub/Google  │
                        │              └───────────────┘
                        │
                 ┌──────┴────────┐
                 │    数据库      │
                 │ users 表      │
                 │ external_ids  │
                 └───────────────┘

核心设计：
1. users 表 + external_ids 表（一对多）
   - provider + external_id 组成唯一键
   - 支持多种登录方式绑定到同一用户

2. 登录流程：
   a. 用户点击 "GitHub 登录"
   b. 完成 OAuth2 流程，获取 ID Token / UserInfo
   c. 用 provider + external_id 查找 external_ids 表
   d. 找到 → 关联已有用户
   e. 未找到 → 创建新用户（或引导绑定已有账号）

3. 账号绑定：
   - 已登录用户可以绑定多个第三方账号
   - 每个 provider 只能绑定一次
```

### Q: 微服务架构中如何传递用户身份？

```
方案：

1. Token 转发（Token Relay）
   每个服务将接收到的 Access Token 原样转发给下游
   下游服务独立验证 Token
   优点：简单
   缺点：Token 在多个服务间传递增加风险

2. Gateway 验证 + 内部 Header
   Gateway 验证 Token 后将用户信息放入 Header
   X-User-Id: 123
   X-User-Roles: USER,ADMIN
   下游服务信任这些 Header（需确保只从 Gateway 接收）
   优点：性能好（只验证一次）
   缺点：需要保证 Gateway 到服务的网络安全

3. Token Exchange
   服务 A 用自己的 Service Token + 用户 Token
   换取一个新的、范围缩小的 Token 调用服务 B
   优点：最小权限
   缺点：实现复杂
```

---

# 3. 常见配置错误与解决

| 错误 | 原因 | 解决 |
|------|------|------|
| invalid_client | client_id 或 client_secret 错误 | 检查配置，确认密钥未被截断 |
| unsupported_grant_type | grant_type 参数不对 | 确认拼写：authorization_code（有下划线） |
| insufficient_scope | scope 不足 | 请求时加大 scope，或检查 Token 实际 scope |
| login_required（prompt=none） | 用户未登录 | 先让用户登录，或用其他 prompt 值 |
| 401 vs 403 | 401 = 未认证（Token 无效），403 = 未授权（Token 有效但权限不足） | 按情况处理 |

---

# 4. 快速调试检查清单

```
OAuth2 问题快速排查：

□ 1. 确认 Token 是否存在
   - 查看浏览器 DevTools → Network → Request Headers
   - 确认 Authorization: Bearer xxx 存在

□ 2. 确认 Token 格式
   - 如果是 JWT：jwt.io 粘贴解码
   - 检查 exp（是否过期）、iss、aud

□ 3. 确认 Token 有效期
   - exp - 当前时间（秒） > 0

□ 4. 确认 Scope
   - JWT Claims 中的 scope 字段是否包含所需权限

□ 5. 确认 endpoints 配置
   - 授权服务器 metadata 是否正确
   - issuer-uri 是否可访问

□ 6. 确认日志
   - 授权服务器日志（code 是否生成、Token 是否颁发）
   - 资源服务器日志（Token 验证失败原因）
   - 客户端日志（请求参数是否正确）

□ 7. 测试 Token
   - curl -H "Authorization: Bearer xxx" https://api.example.com/resource
   - 是否返回预期数据
```
