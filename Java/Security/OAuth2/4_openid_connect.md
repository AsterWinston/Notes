# 1. OpenID Connect 概述与定位

OpenID Connect 1.0（OIDC）是构建在 OAuth 2.0 之上的**身份认证层**，由 OpenID Foundation 维护。它解决了一个根本性问题：**在 OAuth2 的授权流中，同时完成用户身份认证**。

## 1.1 OAuth2 为什么不能做认证？

```
OAuth2 的 Access Token 回答了：
  "这个客户端有权访问这个用户的数据"

但不能回答：
  "这个用户是谁？"
  "这个用户现在确实在操作吗？"
  "这个用户是如何认证的？"

原因：
1. Access Token 不透明 — 客户端不知道里面有什么
2. 不同授权服务器返回的用户标识格式不同 — 没有标准化
3. 没有标准化用户信息获取方式 — 每个实现不同
4. Access Token 的受众是资源服务器 — 不是客户端

OIDC 通过 ID Token 解决：
  - 标准化的身份令牌（JWT 格式）
  - 包含标准化的用户标识声明（sub, name, email 等）
  - ID Token 的受众是客户端本身
  - 统一了用户信息的获取方式（/userinfo 端点）
```

## 1.2 OIDC 与 OAuth2 的关系

```
┌─────────────────────────────────────────────────┐
│                  OpenID Connect                   │
│  ┌───────────────────────────────────────────┐  │
│  │              OAuth 2.0                     │  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │          HTTP + TLS                 │  │  │
│  │  └─────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────┘  │
│                                                  │
│  + ID Token (JWT 格式的用户身份令牌)               │
│  + /userinfo 端点 (标准化用户信息接口)              │
│  + 标准化的 Claims (用户属性声明)                   │
│  + 会话管理 (Session Management)                  │
│  + 注销机制 (RP-Initiated Logout)                 │
│  + 动态注册 (Dynamic Client Registration)         │
└─────────────────────────────────────────────────┘
```

| 维度 | OAuth 2.0 | OpenID Connect |
|------|-----------|----------------|
| 协议类型 | 授权框架 | 身份认证协议 |
| 核心产物 | Access Token | ID Token + Access Token |
| Access Token 受众 | 资源服务器 | 资源服务器 |
| ID Token 受众 | 无 | 客户端 |
| 用户信息获取 | 不标准化 | `/userinfo` 端点（标准化） |
| 会话管理 | 不涉及 | Session Management 规范 |
| 发现机制 | RFC 8414 | `/.well-known/openid-configuration` |

---

# 2. ID Token 深度解析

ID Token 是 OIDC 的核心产物，是一个 JWT 格式的安全令牌，包含用户身份认证的声明。

## 2.1 ID Token 的结构

```
ID Token = Header.Payload.Signature

Header:
{
  "alg": "RS256",
  "kid": "1e9gdk7",
  "typ": "JWT"
}

Payload:
{
  "iss": "https://auth.example.com",          // 颁发者（Issuer）
  "sub": "248289761001",                       // 用户唯一标识（Subject）
  "aud": "s6BhdRkqt3",                         // 受众 — 客户端 ID（Audience）
  "exp": 1650000000,                           // 过期时间
  "iat": 1649996400,                           // 颁发时间
  "auth_time": 1649996400,                     // 用户认证时间
  "nonce": "n-0S6_WzA2Mj",                     // 防重放
  "at_hash": "77QmUPtjPfzWtF2AnpK9Rg",         // Access Token 哈希
  "c_hash": "LDktKdoQak3Pk0cnX70Clg",          // Code 哈希（Auth Code 模式）
  "acr": "urn:mace:incommon:iap:silver",       // 认证上下文类引用
  "amr": ["pwd", "mfa"],                       // 认证方法引用
  "azp": "s6BhdRkqt3"                          // 授权方（Authorized Party）
}
```

## 2.2 ID Token 的关键声明（Claims）

### 2.2.1 必需声明（Required Claims）

| Claim | 说明 | 验证要求 |
|-------|------|----------|
| iss | 颁发者标识符（URL） | 必须等于配置的 Issuer |
| sub | 用户唯一标识（≤ 255 ASCII） | 客户端内部唯一标识用户的 key |
| aud | 受众（Audience） | 必须包含 client_id |
| exp | 过期时间（Unix Timestamp） | 必须验证未过期 |
| iat | 颁发时间（Unix Timestamp） | 允许一定时钟偏差 |

### 2.2.2 重要可选声明

| Claim | 说明 | 安全用途 |
|-------|------|----------|
| auth_time | 用户完成认证的时间 | 配合 `max_age` 参数限制会话时长 |
| nonce | 客户端生成的随机值 | 防重放攻击 |
| at_hash | Access Token 左半截 SHA-256 的 Base64 | 绑定 ID Token 和 Access Token |
| c_hash | Authorization Code 的 SHA-256 | 绑定 ID Token 和 Code（hybrid 模式） |
| acr | 认证强度标识 | 根据业务要求验证认证强度 |
| amr | 认证方法列表 | 记录用户用什么方式登录 |

### 2.2.3 nonce 防重放机制

```
攻击场景（无 nonce）：
1. 攻击者在公开网络截获了 ID Token
2. 攻击者重放该 ID Token 给客户端
3. 客户端可能将攻击者的身份绑定到自己的会话

nonce 防御：
1. 客户端生成随机 nonce（如 "n-0S6_WzA2Mj"）
2. 授权请求中携带 nonce
3. 授权服务器将 nonce 放入 ID Token
4. 客户端验证 ID Token 中的 nonce 与请求时的 nonce 一致
5. 不匹配 → 拒绝 Token
```

### 2.2.4 at_hash 的 Token 绑定

```
at_hash 计算方式：
  at_hash = Base64UrlEncode(SHA-256(AccessToken)[0..15])

验证步骤：
1. 客户端收到 ID Token 和 Access Token
2. 计算 Access Token 的 at_hash
3. 与 ID Token 中的 at_hash 比较
4. 不匹配 → Token 可能被篡改/替换

安全效果：
- 防止攻击者在中间替换 Access Token
- 确保 ID Token 和 Access Token 来自同一授权服务器
- 防止跨协议攻击（Cross-Protocol Attack）
```

---

# 3. OIDC 的标准 Scope

OIDC 扩展了 OAuth2 的 scope 机制，定义了几个标准 Scope，每个对应一组用户属性声明。

## 3.1 Scope 与 Claims 的映射关系

| Scope | 包含的 Claims | 说明 |
|-------|-------------|------|
| openid | sub, iss, aud, exp, iat | 必须请求（告知授权服务器这是 OIDC 请求） |
| profile | name, family_name, given_name, middle_name, nickname, preferred_username, profile, picture, website, gender, birthdate, zoneinfo, locale, updated_at | 用户基本信息 |
| email | email, email_verified | 邮箱地址及验证状态 |
| address | address (JSON Object) | 地址信息 |
| phone | phone_number, phone_number_verified | 电话号码 |

```
示例请求：
scope=openid profile email

返回 ID Token Claims：
{
  "sub": "248289761001",
  "name": "Jane Doe",
  "given_name": "Jane",
  "family_name": "Doe",
  "preferred_username": "jane.doe",
  "email": "janedoe@example.com",
  "email_verified": true,
  "picture": "https://example.com/jane_doe.jpg"
}
```

> `openid` scope 是 OIDC 与纯 OAuth2 的分界线：
> - 请求中包含 `openid` scope → 授权服务器知道客户端期望 OIDC 行为（返回 ID Token）
> - 不包含 `openid` scope → 纯 OAuth2 流程（只返回 Access Token）

## 3.2 自定义 Scope 和 Claims

授权服务器可以定义自己的 Scope 和 Claims：

```
自定义 Scope：
  scope=openid department role

自定义 Claims（需要与授权服务器协商）：
  {
    "department": "engineering",
    "role": "senior-developer"
  }
```

---

# 4. OIDC 的三种认证流程

OIDC 基于 OAuth2 的 Authorization Code 和 Implicit 模式，定义了三种标准认证流程。

## 4.1 Authorization Code Flow（授权码流程）

**这是 OIDC 唯一推荐的认证流程。**

```
OIDC Authorization Code Flow：

1. 客户端 → /authorize
   response_type=code
   &scope=openid profile email
   &nonce=n-0S6_WzA2Mj

2. 授权服务器 → 302 /callback
   ?code=SplxlOBeZQQYbYS6WxSbIA

3. 客户端 → POST /token
   grant_type=authorization_code
   &code=SplxlOBeZQQYbYS6WxSbIA

4. 授权服务器 → JSON
   {
     "access_token": "eyJhbGci...",
     "token_type": "Bearer",
     "id_token": "eyJhbGciOiJSUzI1NiIs..."  ← ID Token 在这里返回
   }
```

优势：
- ID Token 通过后端交换，不经过浏览器
- 可配合 PKCE
- 支持 Refresh Token

## 4.2 Hybrid Flow（混合流程）

Hybrid Flow 是 Authorization Code Flow 的变体，允许部分 Token 从前端返回：

```
response_type=code id_token
response_type=code token
response_type=code id_token token

用途：客户端可以提前获取 ID Token（验证身份），同时需要通过后端获取 Access Token
安全考量：ID Token 在 URL Fragment 中返回，需要防范泄露风险
```

> Hybrid Flow 在现代实践中已很少使用，通常 Authorization Code Flow 即可满足需求。

## 4.3 Implicit Flow（隐式流程）

**OIDC 也已不推荐 Implicit Flow**，应使用 Authorization Code Flow + PKCE 替代。

---

# 5. UserInfo 端点

UserInfo 端点（`/userinfo`）是 OIDC 标准化的用户信息 API，客户端使用 Access Token 访问。

## 5.1 请求方式

```http
# Bearer Token（推荐）
GET /userinfo HTTP/1.1
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

# Form Body（不推荐）
POST /userinfo HTTP/1.1
Content-Type: application/x-www-form-urlencoded

access_token=eyJhbGciOiJSUzI1NiIs...
```

## 5.2 响应格式

UserInfo 端点支持两种响应格式，由客户端在注册时指定：

### JSON 格式（默认）

```json
{
  "sub": "248289761001",
  "name": "Jane Doe",
  "given_name": "Jane",
  "family_name": "Doe",
  "preferred_username": "jane.doe",
  "email": "janedoe@example.com",
  "email_verified": true,
  "picture": "https://example.com/jane_doe.jpg",
  "updated_at": 1649996400
}
```

### JWT 格式（签名/加密的 UserInfo 响应）

```
Header: Content-Type: application/jwt

响应体：完整的 JWT，与 ID Token 具有相同的签名验证要求
优势：客户端可以直接验证 UserInfo 的完整性和来源
```

## 5.3 ID Token vs UserInfo 端点的使用策略

| 数据 | 获取方式 | 适用场景 |
|------|----------|----------|
| sub（唯一标识） | ID Token（已验证） | 用户身份标识（应始终使用 sub） |
| 个人资料 | ID Token 或 UserInfo | 基本信息（name, picture）在 ID Token 中即可 |
| 敏感信息 | UserInfo 端点 | 地址、电话等信息通过 UserInfo 按需获取 |
| 自定义属性 | UserInfo 端点 | 企业自定义属性通常在 UserInfo 中返回 |

```
最佳实践：
1. sub 是唯一稳定的用户标识 — 始终用 sub 做关联
2. email 可能会变化 — 不要用 email 做主键
3. ID Token 的内容用于"识别"用户 — on-the-fly 验证
4. UserInfo 的内容用于"丰富"用户资料 — 按需获取并缓存
```

---

# 6. OIDC 会话管理与注销

## 6.1 RP-Initiated Logout（客户端发起注销）

RP（Relying Party，即客户端）发起注销，同时登出应用和授权服务器。

```http
GET /end_session?  
    id_token_hint=eyJhbGciOiJSUzI1NiIs...
    &post_logout_redirect_uri=https://client.example.com/logged-out
    &state=logout_state_123
```

```
注销流程：

1. 用户在客户端点击"退出登录"
2. 客户端清除本地会话
3. 客户端重定向用户到 /end_session 端点
4. 携带 id_token_hint（告知授权服务器要注销哪个用户）
5. 授权服务器清除用户会话
6. 重定向回 post_logout_redirect_uri
```

> **安全要点**：
> - `id_token_hint` 必须包含有效的 ID Token，授权服务器验证后才知道要注销谁
> - `post_logout_redirect_uri` 必须在客户端注册时预配置，防止开放重定向
> - 授权服务器应同时通知其他已登录的应用（通过 Back-Channel Logout）

## 6.2 Back-Channel Logout

授权服务器主动通知所有应用：某用户已注销。

```http
POST /backchannel_logout HTTP/1.1
Content-Type: application/x-www-form-urlencoded

logout_token=eyJhbGciOiJSUzI1NiIs...
```

Logout Token 是一个特殊的 JWT：

```json
{
  "iss": "https://auth.example.com",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1650000000,
  "jti": "unique-logout-event-id",
  "events": {
    "http://schemas.openid.net/event/backchannel-logout": {}
  }
}
```

## 6.3 Session Management（iframe 方式，已不推荐）

```
旧方式：OP iframe + postMessage
- 在客户端嵌入一个隐藏的 OP iframe
- 通过 postMessage 检测用户在 OP 的会话状态
- 问题：第三方 Cookie 被浏览器逐步禁用，此方案已失效

新方式（推荐）：Back-Channel Logout + 定期 Token 刷新
```

---

# 7. OIDC 的发现与注册机制

## 7.1 OpenID Provider 发现

```
OIDC 的发现基于 Issuer URL：

已知：https://accounts.google.com

发现端点：
https://accounts.google.com/.well-known/openid-configuration

响应：
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "end_session_endpoint": "https://accounts.google.com/logout",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["iss", "sub", "aud", "exp", "iat", "name", "email"]
}
```

## 7.2 动态客户端注册（Dynamic Client Registration）

```
POST /register HTTP/1.1
Content-Type: application/json

{
  "application_type": "web",
  "redirect_uris": ["https://client.example.com/callback"],
  "client_name": "My Application",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "client_secret_basic",
  "scope": "openid profile email"
}

响应：
{
  "client_id": "s6BhdRkqt3",
  "client_secret": "cf136dc3c1fc9ca...",
  "client_id_issued_at": 1649996400,
  "client_secret_expires_at": 0  // 0 表示永不过期
}
```

---

# 8. ID Token 验证清单

客户端在接收到 ID Token 后，必须按以下顺序进行验证：

```
□ 1. 验证 iss（Issuer）—— 必须等于你配置的 Issuer URL
□ 2. 验证 aud（Audience）—— 必须包含你的 client_id
□ 3. 验证 exp（Expiration）—— 当前时间 < exp
□ 4. 验证 iat（Issued At）—— 允许合理的时钟偏差（如 5 分钟）
□ 5. 验证 nonce（如果请求时携带了 nonce）—— 与请求时的 nonce 一致
□ 6. 验证 at_hash（如果 ID Token 与 Access Token 同时接收）
□ 7. 验证签名（使用 /jwks 端点的公钥）
□ 8. 验证 azp（Authorized Party，如果存在）—— 与 client_id 一致
□ 9. 验证 auth_time（如果业务需要限制认证时效）
□ 10. 验证 acr（如果请求了特定的认证强度）
```

### JWT 签名验证（Java 示例）

```java
// 1. 从 JWKS 端点获取公钥
String jwksUri = "https://auth.example.com/oauth2/jwks";
JwkSet jwkSet = JwkSet.load(new URL(jwksUri));

// 2. 解析 ID Token 的 Header，获取 kid
Jwt parsedJwt = JWTParser.parse(idTokenString);
String keyId = parsedJwt.getHeader().getKeyId();

// 3. 通过 kid 找到对应的公钥
Jwk jwk = jwkSet.getKeyByKeyId(keyId);
PublicKey publicKey = jwk.toRSAKey().toPublicKey();

// 4. 验证签名
SignedJWT signedJWT = SignedJWT.parse(idTokenString);
JWSVerifier verifier = new RSASSAVerifier((RSAPublicKey) publicKey);
boolean isValid = signedJWT.verify(verifier);

// 5. 读取 Claims
JWTClaimsSet claims = signedJWT.getJWTClaimsSet();
String sub = claims.getSubject();
String issuer = claims.getIssuer();
```

---

# 9. OIDC 安全特性总结

| 安全机制 | 作用 | 实现方式 |
|----------|------|----------|
| nonce | 防重放攻击 | 客户端生成，包含在请求和 ID Token 中 |
| at_hash | Token 绑定 | 绑定 ID Token 和 Access Token |
| c_hash | Code 绑定 | 绑定 ID Token 和 Authorization Code |
| PKCE | 防 Code 截获 | 与 OAuth2 相同 |
| state | 防 CSRF | 与 OAuth2 相同 |
| iss 验证 | 防伪造 Issuer | 严格匹配 Issuer URL |
| aud 验证 | 防跨客户端滥用 | aud 必须包含当前 client_id |
| exp / iat | 时效性控制 | Token 有效期验证 |
| auth_time | 会话时效控制 | 配合 max_age 参数 |
| acr / amr | 认证强度保证 | 确认认证方式和强度 |
