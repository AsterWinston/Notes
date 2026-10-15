# 1. 授权模式总览

OAuth2 的核心是通过不同的**授权模式（Authorization Grant Types）**来适配不同的客户端类型和安全场景。RFC 6749 定义了 4 种标准模式，后续 RFC 补充了额外的授权模式。

## 1.1 模式对比一览

| 授权模式 | grant_type | 用户参与 | Token 发放方式 | 当前推荐度 | 主要风险 |
|----------|-----------|----------|---------------|------------|----------|
| Authorization Code | authorization_code | 是（浏览器重定向） | 后端 code 换 token | 强烈推荐 | redirect_uri 劫持（用 PKCE 防御） |
| PKCE-enhanced Auth Code | authorization_code + code_verifier | 是 | 后端 code+verifier 换 token | 最强推荐 | 基本无（现代标准） |
| Implicit | implicit | 是 | 重定向 URL Fragment 直接返回 | 已废弃 | Token 泄露、无 Refresh Token |
| Resource Owner Password | password | 是（客户端拿密码） | 直接 POST 换 token | 已废弃 | 密码暴露、不支持 MFA |
| Client Credentials | client_credentials | 否 | 直接 POST 换 token | 推荐（M2M） | secret 泄露 |
| Device Code | urn:ietf:params:oauth:grant-type:device_code | 是（另一设备确认） | 轮询获取 | 推荐（TV/IoT） | 用户码暴力破解（需速率限制） |
| JWT Bearer | urn:ietf:params:oauth:grant-type:jwt-bearer | 否 | JWT 断言 | 推荐（企业集成） | JWT 签名验证 |

---

# 2. Authorization Code（授权码模式）— 详细分析

## 2.1 完整协议流程

Authorization Code 模式是整个 OAuth2 的基石，理解其流程等于掌握了 OAuth2 的核心。

```
      用户                  客户端               授权服务器          资源服务器
       │                     │                     │                  │
       │  1. 点击"GitHub登录"  │                     │                  │
       │────────────────────>│                     │                  │
       │                     │  2. GET /authorize?  │                  │
       │                     │     response_type=code│                 │
       │                     │     &client_id=xxx    │                  │
       │                     │     &redirect_uri=xxx │                  │
       │                     │     &scope=read       │                  │
       │                     │     &state=random123  │                  │
       │                     │     &code_challenge=xxx│                 │
       │                     │────────────────────>│                  │
       │                     │                     │ 3. 验证客户端身份 │
       │                     │                     │    展示登录页面   │
       │                     │  4. 302 登录页面       │                  │
       │<──────────────────────────────────────────│                  │
       │                     │                     │                  │
       │  5. 输入用户名/密码     │                     │                  │
       │──────────────────────────────────────────>│                  │
       │                     │                     │ 6. 验证用户凭证   │
       │                     │                     │    展示授权页面   │
       │  7. 确认授权 Scope    │                     │                  │
       │<──────────────────────────────────────────│                  │
       │                     │                     │                  │
       │  8. 点击"允许"        │                     │                  │
       │──────────────────────────────────────────>│                  │
       │                     │                     │ 9. 生成 auth code │
       │                     │  10. 302 redirect    │                  │
       │                     │      ?code=abc123    │                  │
       │                     │      &state=random123│                  │
       │                     │<────────────────────│                  │
       │                     │                     │                  │
       │                     │  11. POST /token     │                  │
       │                     │      grant_type=     │                  │
       │                     │        authorization_code              │
       │                     │      &code=abc123    │                  │
       │                     │      &redirect_uri=xxx                  │
       │                     │      &client_id=xxx  │                  │
       │                     │      &client_secret=xxx                 │
       │                     │      &code_verifier=xxx                 │
       │                     │────────────────────>│                  │
       │                     │                     │ 12. 验证 code     │
       │                     │                     │     + client      │
       │                     │                     │     + verifier    │
       │                     │  13. JSON Response   │                  │
       │                     │      {access_token,  │                  │
       │                     │       refresh_token, │                  │
       │                     │       expires_in,    │                  │
       │                     │       token_type}    │                  │
       │                     │<────────────────────│                  │
       │                     │                     │                  │
       │                     │  14. GET /api/user   │                  │
       │                     │      Authorization:  │                  │
       │                     │      Bearer xxx      │                  │
       │                     │─────────────────────────────────────>│
       │                     │                     │    15. 验证 Token │
       │                     │  16. User Data       │                  │
       │                     │<─────────────────────────────────────│
```

## 2.2 步骤拆解与分析

### 2.2.1 第 1-4 步：发起授权请求

```http
GET /authorize?response_type=code
              &client_id=s6BhdRkqt3
              &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback
              &scope=openid%20profile%20email
              &state=xcoiv98wsefkj
              &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
              &code_challenge_method=S256
Host: auth-server.example.com
```

参数说明：

| 参数 | 必填 | 说明 |
|------|------|------|
| response_type | 是 | 授权码模式固定为 `code` |
| client_id | 是 | 客户端注册时获得的标识 |
| redirect_uri | 条件 | 客户端注册过多个 redirect_uri 时必填 |
| scope | 可选 | 请求的权限范围（空格分隔） |
| state | 推荐 | 防 CSRF 的随机字符串 |
| code_challenge | PKCE 必填 | BASE64URL(SHA256(code_verifier)) |
| code_challenge_method | PKCE 必填 | 固定为 `S256` |

> **重要**：所有参数必须放在 URL Query String 中（不能放在 POST Body），因为这是浏览器 302 重定向。

### 2.2.2 第 5-8 步：用户认证并授权

```
授权服务器内部处理流程：

1. 解析请求参数，验证 client_id 和 redirect_uri
2. 检查用户当前是否已登录
   - 已登录：直接跳到授权页面
   - 未登录：展示登录页面
3. 展示授权页面（Consent Page）：
   ┌──────────────────────────────────────┐
   │  应用 "My App" 请求以下权限：          │
   │  ✓ 读取你的公开信息                    │
   │  ✓ 读取你的邮箱地址                    │
   │  □ 修改你的仓库（未请求）              │
   │                                      │
   │   [允许]        [拒绝]               │
   └──────────────────────────────────────┘
4. 用户点击"允许"或"拒绝"
5. 如果拒绝：重定向回 redirect_uri，错误参数 error=access_denied
```

### 2.2.3 第 9-10 步：颁发授权码并重定向

```
授权服务器生成 Authorization Code：

code 的特征：
- 短期有效（推荐 ≤ 60 秒，最多 10 分钟）
- 一次性使用（用后即失效，防重放）
- 不可猜测（高熵随机字符串，≥ 128 bit）
- 绑定 client_id（只有申请的客户端能使用）
- 绑定 redirect_uri（只有申请时指定的 URI 能使用）
- 绑定 code_challenge（如果使用 PKCE）

302 Redirect：
Location: https://client.example.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xcoiv98wsefkj
```

### 2.2.4 第 11-13 步：用 Code 换取 Token（Token Endpoint）

**这是 Authorization Code 模式最关键的步骤**——Front Channel 和 Back Channel 的切换点。

```http
POST /token HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

> **client_secret 放在哪？**
> - 推荐：HTTP Basic Auth Header (`Authorization: Basic base64(client_id:client_secret)`)
> - 不推荐：POST Body (`client_id=xxx&client_secret=xxx`) — 安全性较低
> - 禁止：URL Query String — 会被记录在服务器日志中

**授权服务器的验证步骤**（必须严格按顺序）：

```
1. 验证 client_id 是否存在、client_secret 是否正确
2. 验证 grant_type 是否为 authorization_code
3. 验证 authorization code 是否存在、是否已过期
4. 验证 code 是否已被使用过（防重放）
5. 验证 redirect_uri 与授权请求中是否完全一致
6. 验证 code_verifier（如果使用了 PKCE）：
   BASE64URL(SHA256(code_verifier)) == code_challenge?
7. 所有验证通过，颁发 Access Token 和 Refresh Token
8. 将 code 标记为已使用或删除
```

成功响应：

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8",
  "scope": "openid profile email",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."  // OIDC 场景
}
```

错误响应：

```json
{
  "error": "invalid_grant",
  "error_description": "The authorization code has expired"
}
```

错误码定义（RFC 6749 Section 5.2）：

| error | 说明 |
|-------|------|
| invalid_request | 请求缺少必要参数 |
| invalid_client | 客户端认证失败（client_id 或 client_secret 错误） |
| invalid_grant | 授权码无效、过期或已被使用 |
| unauthorized_client | 客户端没有权限使用此授权模式 |
| unsupported_grant_type | 不支持的 grant_type |
| invalid_scope | 请求的 scope 超出允许范围 |

## 2.3 Authorization Code 的安全特性分析

### 2.3.1 为什么 Authorization Code 比 Implicit 更安全？

```
安全核心：Access Token 不经过浏览器（User-Agent）传递

Implicit 模式的问题：
  浏览器 ──(Access Token 在 URL Fragment)──> 客户端前端
  风险点：浏览器历史、Referrer Header、JS 脚本可读取

Authorization Code 模式的保障：
  浏览器 ──(Authorization Code)──> 客户端后端 ──(code → 后端换 Token)──> 授权服务器
  保障点：
  1. code 泄露后无法单独使用（需要 client_secret 或 code_verifier）
  2. Access Token 只在后端传输，不经过浏览器
  3. code 只能使用一次且短期有效
```

### 2.3.2 授权码被拦截的风险与防御

**攻击场景**：恶意应用注册了相同的自定义 URL Scheme，截获回调 URL：

```
Android 示例：
用户安装了恶意 App "Evil App"，也注册了 myapp://callback 这个 scheme。
授权服务器重定向 myapp://callback?code=real_code 时，系统弹出选择框。
用户选择了 Evil App → Evil App 拿到了 code → Evil App 用自己的 client_secret 换 Token

无 PKCE 时：Evil App 可以成功获取 Token
有 PKCE 时：Evil App 不知道 code_verifier → 无法完成 Token 交换
```

---

# 3. PKCE (Proof Key for Code Exchange) 深入解析

## 3.1 PKCE 的必要性

PKCE（RFC 7636，发音 "pixy"）最初是为移动 App 和 Native 应用设计的，但现在已成为**所有使用 Authorization Code 模式的通用安全要求**。

**为什么 Native App 和 SPA 尤其需要 PKCE？**

```
场景对比：

Web 应用 (机密客户端)：
  - 有后端服务器，能安全保存 client_secret
  - code 被截获后，攻击者不知道 client_secret，无法换 Token

Native App (公开客户端)：
  - 没有后端服务器，client_secret 写在 App 中 = 公开信息
  - code 被截获后，攻击者可以提取 client_secret，直接换 Token
  - 需要 PKCE 提供额外的、只有发起方才知道的秘密
```

## 3.2 PKCE 完整参数与算法

### 3.2.1 code_verifier

```
定义：高熵随机字符串，由客户端生成
字符集：ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~
长度：43-128 字符（RFC 7636 要求）

Java 生成示例：
SecureRandom sr = new SecureRandom();
byte[] code = new byte[32];
sr.nextBytes(code);
String code_verifier = Base64.getUrlEncoder().withoutPadding().encodeToString(code);
// 结果示例：dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

### 3.2.2 code_challenge

```
两种计算方式：

1. S256（推荐，必须实现）：
   code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))

2. plain（不推荐，仅限不支持 SHA256 的环境）：
   code_challenge = code_verifier

Java 计算示例：
MessageDigest md = MessageDigest.getInstance("SHA-256");
byte[] digest = md.digest(code_verifier.getBytes(StandardCharsets.US_ASCII));
String code_challenge = Base64.getUrlEncoder().withoutPadding().encodeToString(digest);
```

### 3.2.3 参数传递流程

```
请求 /authorize：
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256

请求 /token：
  &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk

授权服务器验证：
  BASE64URL(SHA256(ASCII(received_code_verifier))) == stored_code_challenge ?
```

## 3.3 PKCE 的攻防矩阵

| 攻击类型 | 无 PKCE | 有 PKCE（S256） | 有 PKCE（plain） |
|----------|---------|-----------------|------------------|
| 授权码截获 (code interception) | 成功 | 阻止 | 阻止 |
| 恶意 App 注册同 scheme | 成功 | 阻止 | 阻止 |
| 授权码重放 (code replay) | 成功 | 阻止 | 阻止 |
| code_challenge 被窃取（plain 模式） | N/A | N/A | 可能成功 |

> **安全建议**：始终使用 S256，永远不要用 plain 模式。

---

# 4. Client Credentials（客户端凭证模式）深入解析

## 4.1 适用场景

Client Credentials 是唯一一个不涉及用户的授权模式，专为**服务器到服务器**通信设计。

```
典型应用场景：

1. 微服务间 API 调用
   Service A → [OAuth2] → Service B

2. 后台任务 / 批处理
   Cron Job → [OAuth2] → REST API

3. CI/CD 流水线
   Jenkins/GitHub Actions → [OAuth2] → Cloud API

4. 物联网设备上报
   IoT Device → [OAuth2] → Data Platform

5. 数据同步服务
   Sync Engine → [OAuth2] → Data Source
```

## 4.2 协议流程

```http
POST /token HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=client_credentials
&scope=read write
```

> Client Credentials 模式使用 HTTP Basic Auth 传递客户端凭证（`client_id:client_secret` 的 Base64 编码）。

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read write"
}
```

> 注意：Client Credentials 模式**不返回 Refresh Token**，因为客户端可以在 Token 过期后直接用凭证重新申请，Refresh Token 在这种情况下没有额外的安全价值。

## 4.3 与 Authorization Code 的关键区别

| 维度 | Authorization Code | Client Credentials |
|------|-------------------|-------------------|
| 用户上下文 | 与特定用户关联 | 仅关联到客户端本身 |
| Token 中包含谁 | sub = user_id | sub = client_id（或 service account） |
| 用户操作 | 需要用户授权 | 完全不需要 |
| Refresh Token | 有 | 无（直接重新请求） |
| 适用场景 | 用户委托授权 | 服务自主调用 |

## 4.4 Client Credentials 的安全考量

```
安全要点：

1. client_secret 保护
   - 使用安全的密钥管理服务（Vault、KMS、环境变量）
   - 定期轮换（Rotation）client_secret
   - 绝不在代码中硬编码

2. 最小权限原则
   - 每个微服务使用独立的 client_id
   - scope 严格最小化，不交叉
   - 为不同环境（dev/staging/prod）分配不同 scope

3. Token 有效期
   - 保持短有效期（5-15 分钟）
   - 服务应实现 Token 缓存机制（避免每个请求都换新 Token）

4. 审计与监控
   - 记录所有 Token 申请请求
   - 监控异常的请求频率
   - 对 client_id 设置速率限制
```

## 4.5 M2M 授权升级：JWT Bearer Assertion

企业场景中，Client Credentials 可以升级为 JWT Bearer Assertion 模式（RFC 7523）：

```http
POST /token HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=eyJhbGciOiJSUzI1NiIs...（客户端自己签名的 JWT）
&scope=read
```

JWT Assertion 的优势：
- 无需传输 client_secret（改用公私钥签名验证）
- 可在 JWT 中包含更多声明（目标 API、请求时间等）
- 支持委托链（一个服务代表另一个服务请求）

---

# 5. Resource Owner Password Credentials（已不推荐）

## 5.1 为什么被淘汰

```
核心问题：违背 OAuth2 的根本设计理念

OAuth2 的初衷：用户不把密码给第三方
Password 模式：用户把密码给第三方

具体危害：
1. 客户端持有用户明文密码
   - 客户端被攻破 = 用户密码泄露
   - 客户端不可信 = 无法证明其是否安全存储密码

2. 阻断了认证升级的可能
   - 无法使用多因素认证（MFA/2FA）
   - 无法使用 WebAuthn / Passkey
   - 无法使用生物识别认证
   - 无法使用单点登录（SSO）

3. 无法区分客户端
   - 攻击者截获密码后可以从任意客户端登录
   - 无法针对特定客户端撤销权限

4. 合规性问题
   - 违反 GDPR 最小数据收集原则
   - 违反 PSD2 强客户认证（SCA）要求
```

## 5.2 替代方案

| 原场景 | 替代方案 |
|--------|----------|
| 第一方原生 App | Authorization Code + PKCE + AppAuth 库 |
| 内部管理后台 | Authorization Code + OIDC |
| 自动化脚本 | Client Credentials / JWT Bearer |
| 数据库/API 凭证管理 | API Key / Service Account |

---

# 6. Implicit（隐式模式）— 已不推荐

## 6.1 历史角色与淘汰原因

```
历史背景：
2012 年 OAuth2 发布时，浏览器的跨域请求能力有限。
SPA 无法用 JS 直接 POST 到 Token Endpoint（CORS 未普及）。
因此设计了 Implicit 模式，Token 通过 URL Fragment 直接返回到前端。

淘汰原因（OAuth 2.0 Security BCP）：
1. Access Token 暴露在浏览器 URL Fragment
   - 浏览器历史记录保存 Token
   - Referer Header 可能泄露 Token
   - 恶意 JS 可读取 window.location.hash

2. 无法返回 Refresh Token（安全考虑）
   - Implicit 模式下不允许 Refresh Token
   - 用户必须频繁重新授权

3. 缺乏客户端认证
   - client_secret 无法在 SPA 中使用
   - 攻击者可以冒充客户端

4. CORS 已广泛支持
   - 现代浏览器完全支持跨域 POST
   - 不需要 Implicit 模式的 hack
```

## 6.2 现代替代方案

```
SPA 的 OAuth2 最佳实践：

方案：Authorization Code + PKCE（无 client_secret）

步骤：
1. SPA 发起 /authorize（含 code_challenge）
2. 用户授权后，回到 SPA 的回调 URL（code 在 URL 参数中）
3. SPA 提取 code，通过 JS 用 fetc / axios 直接 POST 到 /token
   - 携带 code 和 code_verifier
   - 不携带 client_secret（SPA 是公开客户端）
4. 获得 Access Token 和 Refresh Token
5. Token 存储在内存中（不用 localStorage）

关键保障：
- code 被截获后无法单独用（需要 code_verifier）
- Access Token 通过 HTTPS POST 响应返回，不经过 URL
- 支持 Refresh Token
```

---

# 7. Device Code（设备授权模式）

## 7.1 适用场景

设备授权模式（RFC 8628）专为**输入受限或没有浏览器的设备**设计：

- 智能电视（TV）、游戏主机
- 打印机、智能音箱
- IoT 设备（无显示屏或仅低分辨率显示屏）
- CLI 工具（如 `gh auth login`）

## 7.2 协议流程

```
           设备/CLI                  授权服务器                  用户的手机/PC
              │                         │                          │
              │  1. POST /device_authorization                     │
              │──────────────────────>│                         │
              │  2. 返回                │                         │
              │     device_code        │                         │
              │     user_code          │                         │
              │     verification_uri   │                         │
              │<──────────────────────│                         │
              │                         │                         │
              │  3. 展示 user_code 和 URL 给用户                    │
              │  "请访问 https://auth.example.com/device           │
              │   输入验证码: ABCD-1234"                          │
              │──────────────────────────────────────────────────>│
              │                         │                         │
              │  4. 轮询 POST /token     │  5. 用户在手机浏览器上访问  │
              │     grant_type=device_code│    输入 user_code         │
              │     &device_code=xxx    │    确认授权              │
              │  Throttled polling ───>│<───────────────────────│
              │                         │                         │
              │  6. authorization_pending│                         │
              │<──────────────────────│                         │
              │                         │                         │
              │  7. 继续轮询...          │                         │
              │                         │                         │
              │  8. 返回 Access Token    │                         │
              │<──────────────────────│                         │
```

### 7.2.1 请求 Device Code

```http
POST /device_authorization HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded

client_id=s6BhdRkqt3
&scope=read write
```

响应：

```json
{
  "device_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
  "user_code": "ABCD-1234",
  "verification_uri": "https://auth.example.com/device",
  "verification_uri_complete": "https://auth.example.com/device?user_code=ABCD-1234",
  "expires_in": 1800,
  "interval": 5
}
```

### 7.2.2 轮询 Token

```http
POST /token HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:device_code
&device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
&client_id=s6BhdRkqt3
```

可能的错误响应：

| error | 说明 |
|-------|------|
| authorization_pending | 用户尚未授权，继续等待 |
| slow_down | 轮询过快，需要增加间隔 |
| expired_token | device_code 已过期，需重新开始 |
| access_denied | 用户拒绝授权 |

## 7.3 安全考量

```
1. 速率限制
   - 轮询间隔应逐渐增加（exponential backoff）
   - 防止暴力猜测 user_code

2. user_code 安全
   - user_code 应有足够熵值
   - 短小而精悍（20-bit+ 的熵）
   - 过期时间有限（推荐 10-15 分钟）

3. device_code 绑定
   - device_code 绑定到 client_id
   - 单次使用
```

---

# 8. Token Exchange（令牌交换）

RFC 8693 定义了 Token Exchange 机制，允许一个 Token 换取另一个 Token：

```
场景：
- 委托授权（Delegation）— 服务 A 用其 Token 换取代表用户的 Token 调用服务 B
- 降权限（Downgrade Scope）— 减少 Token 的权限范围
- 跨域（Cross-domain）— 一个域颁发的 Token 换取另一个域颁发的 Token

POST /token
  grant_type=urn:ietf:params:oauth:grant-type:token-exchange
  &subject_token=old_token
  &subject_token_type=urn:ietf:params:oauth:token-type:access_token
  &scope=read
  &audience=api2.example.com
```

---

# 9. 授权模式选择决策树（完整版）

```
                          ┌─────────────┐
                          │  开始选择    │
                          └──────┬──────┘
                                 │
                    ┌────────────┴────────────┐
                    │ 是否需要用户参与授权？      │
                    └────────┬───────┬────────┘
                             │       │
                          是 │       │ 否
                             │       │
              ┌──────────────┘       └──────────────────┐
              │                                          │
   ┌──────────┴──────────┐                    ┌──────────┴──────────┐
   │ 客户端通过浏览器交互？ │                    │ 微服务/M2M 通信？    │
   └──────────┬──────────┘                    └──────────┬──────────┘
              │                                          │
    ┌─────┴─────┐                              ┌───────┴────────┐
    │           │                              │                │
   是          否                             是               否
    │           │                              │                │
    │    ┌──────┴──────┐                ┌──────┴──────┐  ┌──────┴──────┐
    │    │ 设备/CLI？   │                │ Client      │  │ 企业内部    │
    │    └──────┬──────┘                │ Credentials │  │ 服务？      │
    │           │                       └─────────────┘  └──────┬──────┘
    │     ┌─────┴─────┐                                        │
    │     │           │                                  ┌─────┴─────┐
    │    是          否                                  │           │
    │     │           │                                 是          否
    │  Device      Native                               │           │
    │  Code        App                            JWT Bearer   自定义？
    │                           Authorization Code + PKCE
    │
    │     ┌────────────┐
    │     │ 客户端类型？ │
    │     └──────┬─────┘
    │            │
    │     ┌──────┴──────┐
    │     │             │
    │  有后端        无后端
    │  (机密客户端)   (公开客户端：SPA)
    │     │             │
    │  Auth Code     Auth Code
    │  (可选+PKCE)    + PKCE
    │                 (必须)
```

## 9.1 总结原则

1. **有用户参与 → 总是用 Authorization Code + PKCE**
2. **纯 M2M → Client Credentials**
3. **输入受限设备 → Device Code**
4. **永远不要用 Implicit 和 Resource Owner Password**
