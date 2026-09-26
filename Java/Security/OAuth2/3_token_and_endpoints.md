# 1. Token 类型与设计原理

OAuth2 中涉及多种 Token，每种 Token 有独立的职责、生命周期和安全模型。

## 1.1 Token 类型总览

| Token 类型 | 作用 | 生命周期 | 持有者 | 可否自包含 |
|------------|------|----------|--------|------------|
| Access Token | 访问受保护资源 | 短期（分钟 ~ 小时） | 客户端 | 是（JWT）/ 否（Opaque） |
| Refresh Token | 获取新的 Access Token | 长期（天 ~ 月） | 仅授权服务器和客户端 | 通常不透明 |
| Authorization Code | 换取 Access Token 的凭证 | 极短（≤ 60 秒） | 客户端后端 | 否 |
| ID Token（OIDC） | 认证用户身份 | 与 Access Token 类似 | 客户端 | 是（JWT） |
| Device Code | 设备流程中的临时代码 | 15-30 分钟 | 设备 | 否 |

## 1.2 为什么需要多种 Token？

```
设计哲学：职责分离

┌──────────────────────────────────────────────────┐
│                                                  │
│  Access Token（短期，频繁使用）                    │
│  ├─ 每天都随 API 请求发送到资源服务器              │
│  ├─ 泄露风险最高，所以有效期最短                    │
│  └─ 不包含过多信息（减少暴露面）                    │
│                                                  │
│  Refresh Token（长期，极少使用）                    │
│  ├─ 仅与授权服务器通信时使用                        │
│  ├─ 泄露风险较低，有效期更长                        │
│  └─ 可被撤销而无需立即影响 Access Token             │
│                                                  │
│  安全效果：                                       │
│  - Access Token 泄露 → 影响时间有限                │
│  - Refresh Token 泄露 → 可在下次使用时检测并撤销    │
│  - 两者同时泄露的风险远低于单一长期 Token             │
│                                                  │
└──────────────────────────────────────────────────┘
```

## 1.3 Token 格式对比：Opaque vs JWT

```
Opaque Token（不透明 Token）：
  示例：2YotnFZFEjr1zCsicMWpAA
  特点：
  - 随机字符串，无任何自包含信息
  - 资源服务器需调用 Introspection 端点验证
  - Token 可即时撤销（删除存储即可）
  - 无法离线验证（需网络调用）
  
  ┌──────────┐         ┌──────────────┐         ┌──────────┐
  │ 客户端    │──Token──>│ 资源服务器    │──Token──>│ 授权服务器 │
  │          │<─Resource─│             │<─Valid?──│ (Introspect)│
  └──────────┘         └──────────────┘         └──────────┘

JWT Token（自包含 Token）：
  示例：eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNry...
  特点：
  - 基于 JSON + Base64URL + 签名
  - Token 自身包含声明（claims）
  - 资源服务器可本地验证签名（无需网络调用）
  - 撤销需依赖过期时间或维护黑名单
  
  ┌──────────┐         ┌──────────────┐
  │ 客户端    │──JWT──>│ 资源服务器    │
  │          │<─Resource─│ (本地验签)   │
  └──────────┘         └──────────────┘
```

| 维度 | Opaque Token | JWT Token |
|------|-------------|-----------|
| 验证性能 | 每次需远程调用 | 本地验证，极快 |
| 即时撤销 | 支持（删除即可） | 不支持（需黑名单或等过期） |
| Token 大小 | 小（~30 bytes） | 大（~1KB+） |
| 信息自包含 | 否 | 是 |
| 隐私 | 好（无信息泄露） | 差（claims 可被解码读取） |
| 适合场景 | 内部系统，需强撤销 | 分布式高并发系统 |

---

# 2. Access Token 详细设计

## 2.1 Access Token 的生命周期

```
┌─────────────────────────────────────────┐
│          Access Token 生命周期            │
├─────────────────────────────────────────┤
│                                         │
│  创建 → 活跃 → 即将过期 → 过期            │
│   │      │        │         │          │
│   │      │        │         └─ 资源服务器拒绝  │
│   │      │        └─ 客户端用 Refresh Token 续期│
│   │      └─ 客户端正常使用                    │
│   └─ 授权服务器颁发                         │
│                                         │
│  可被主动撤销（Revocation Endpoint）        │
│  可因用户修改密码而失效                      │
│  可因用户撤销授权而失效                      │
│                                         │
└─────────────────────────────────────────┘
```

## 2.2 Access Token 有效期设计

```
有效期设计原则：

1. 尽可能短（在不影响用户体验的前提下）
   - Web 应用：15-60 分钟
   - API 服务：5-15 分钟
   - 高敏感操作：1-5 分钟

2. 配合 Refresh Token 实现无缝续期
   - 用户无需感知 Token 过期
   - 后台静默刷新

3. 考虑资源服务器验证开销
   - JWT Token：有效期可以略长（本地验证成本低）
   - Opaque Token：有效期应更短（每次需远程验证）

4. 不同资源使用不同有效期
   - 只读操作：30 分钟
   - 写入操作：15 分钟
   - 删除/管理操作：5 分钟
```

## 2.3 Access Token 的传输方式

### Bearer Token（RFC 6750）

```http
# 方式一：Authorization Header（推荐）
GET /resource HTTP/1.1
Authorization: Bearer mF_9.B5f-4.1JqM

# 方式二：Form-Encoded Body Parameter（不推荐）
POST /resource HTTP/1.1
Content-Type: application/x-www-form-urlencoded

access_token=mF_9.B5f-4.1JqM

# 方式三：URI Query Parameter（不推荐，OAuth 2.1 中已移除）
GET /resource?access_token=mF_9.B5f-4.1JqM
```

| 传输方式 | 安全性 | 推荐度 | 原因 |
|----------|--------|--------|------|
| Authorization Header | 最高 | 强烈推荐 | 不会出现在 URL 中，不被服务器日志记录 |
| Form Body | 中等 | 仅限 POST | 仅在 Authorization Header 不可用时使用 |
| Query String | 最低 | 禁止 | 记录在服务器日志、浏览器历史、Referer Header |

### DPoP (Demonstration of Proof-of-Possession)

DPoP（RFC 9449）是 Bearer Token 的安全增强，防止 Token 被窃取后直接使用：

```http
# DPoP 要求客户端对每个请求签名，证明 Token 持有者确实是原始申请者

POST /resource HTTP/1.1
Authorization: DPoP eyJhbGciOiJSUzI1NiIs...
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIs...
```

```
传统 Bearer Token 的问题：
  - 任何人持有 Token 就能访问资源
  - Token 被窃取 = 完全失控

DPoP 的防护：
  - 客户端持有私钥，每个请求生成一个 JWT 作为 DPoP 证明
  - DPoP JWT 中嵌入了 Token 的哈希（绑定 Token 到证明）
  - 即使 Token 泄露，没有私钥也无法使用
```

---

# 3. Refresh Token 详细设计

## 3.1 Refresh Token 的作用与机制

Refresh Token 是 OAuth2 中实现**长期访问能力**而**不牺牲安全性**的核心设计。

```
为什么需要 Refresh Token？

方案 A：长期 Access Token（24h）
  问题：一旦泄露，攻击者有 24 小时完整窗口

方案 B：短期 Access Token（15min） + 要求用户重新登录
  问题：用户体验极差，每 15 分钟就要重新授权

方案 C：短期 Access Token（15min） + Refresh Token（30 天）
  优势：
  - Access Token 泄露影响有限（15 分钟）
  - Refresh Token 只与授权服务器通信，泄露风险低
  - 两者同时泄露的概率极小
```

## 3.2 Refresh Token 轮换（Rotation）

Refresh Token 轮换是 OAuth 2.1 的强制要求：

```
传统模式（危险）：
  客户端 ──> /token (refresh_token=RT1)
  授权服务器 ──> {access_token: AT2, refresh_token: RT1}  ← 不换！
  问题：如果 RT1 被盗，攻击者和合法用户都能刷新，无法检测

轮换模式（安全）：
  客户端 ──> /token (refresh_token=RT1)
  授权服务器 ──> {access_token: AT2, refresh_token: RT2}  ← 替换！
  
  场景一：正常用户先刷新
    用户：RT1 → 获得 AT2 + RT2，RT1 失效
    攻击者：RT1 → 请求失败（已失效）→ 可能触发安全告警
  
  场景二：攻击者先刷新
    攻击者：RT1 → 获得 AT2 + RT2
    用户：RT1 → 请求失败 → 触发重新授权流程
    结果：攻击者暂时获取访问权限，但用户会被通知
```

```
实现要点：

1. 轮换触发条件
   - 每次使用 Refresh Token 都必须生成新的 Refresh Token
   - 旧的 Refresh Token 立即失效
   
2. 重放检测
   - 如果检测到已失效的 Refresh Token 被使用
   - 必须撤销该用户/客户端的所有 Token
   - 触发安全告警，要求用户重新授权

3. Refresh Token 家族（Family）概念
   - 所有由同一个初始 Refresh Token 派生的 Token 属于同一家族
   - 家族中任意 Token 被判定为泄露，整个家族失效
```

## 3.3 Refresh Token 的存储与传输

```
存储要求（客户端）：
  机密客户端：
  - 加密存储在数据库（AES-256-GCM）
  - 关联到用户会话
  
  公开客户端（SPA）：
  - 推荐：通过 BFF（Backend For Frontend）模式存储
  - 替代：Service Worker + IndexedDB
  - 不推荐：localStorage、sessionStorage、cookie（非 HttpOnly）

传输要求：
  - 仅通过 HTTPS POST 到 Token Endpoint
  - 禁止通过 URL 参数传递
  - Refresh Token 本身不应包含用户信息（防信息泄露）
```

---

# 4. 核心端点详细设计

## 4.1 端点全景图

```
                    OAuth2 核心端点
                    
┌─────────────────────────────────────────────────────────────┐
│                      授权服务器                              │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │ /authorize      │  │ /token           │                  │
│  │ (授权端点)       │  │ (令牌端点)        │                  │
│  │ Authorization   │  │ Token            │                  │
│  │ Endpoint        │  │ Endpoint         │                  │
│  └────────┬────────┘  └────────┬────────┘                  │
│           │                    │                            │
│  ┌────────┴────────┐  ┌───────┴─────────┐                 │
│  │ /introspect     │  │ /revoke         │                  │
│  │ (令牌内省端点)   │  │ (令牌吊销端点)   │                  │
│  │ Introspection   │  │ Revocation      │                  │
│  │ Endpoint        │  │ Endpoint        │                  │
│  └─────────────────┘  └─────────────────┘                 │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │ /.well-known/oauth-authorization-server│                 │
│  │ (元数据发现端点) — RFC 8414            │                  │
│  └──────────────────────────────────────┘                  │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │ /userinfo                            │                  │
│  │ (用户信息端点) — OIDC                 │                  │
│  └──────────────────────────────────────┘                  │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │ /jwks                                │                  │
│  │ (JWK Set 端点) — OIDC / JWT          │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## 4.2 授权端点 (/authorize) — RFC 6749 Section 3.1

授权端点是 OAuth2 流程中**唯一在浏览器中执行的步骤**（Front Channel）。

### 请求参数完整列表

| 参数 | 必填 | 格式 | 说明 |
|------|------|------|------|
| response_type | 是 | `code` 或 `token` | 授权类型（推荐 `code`） |
| client_id | 是 | string | 客户端标识 |
| redirect_uri | 条件 | URI | 回调地址，多值注册时必填 |
| scope | 可选 | 空格分隔的字符串 | 权限范围 |
| state | 推荐 | 随机字符串 | 防 CSRF |
| code_challenge | PKCE | BASE64URL | PKCE challenge |
| code_challenge_method | PKCE | `S256` | PKCE 方法 |
| prompt | 可选 | `none`/`login`/`consent`/`select_account` | 控制用户交互行为 |
| max_age | 可选 | 秒数（integer） | 用户上次认证后的最大时间 |
| login_hint | 可选 | email/username | 提示登录账号 |
| acr_values | 可选 | string | 认证上下文类引用 |
| nonce | OIDC | 随机字符串 | ID Token 防重放 |
| display | 可选 | `page`/`popup`/`touch`/`wap` | 授权页面展示方式 |
| ui_locales | 可选 | BCP47 语言标签 | 授权页面语言 |
| id_token_hint | 可选 | JWT | 已登录用户的 ID Token 提示 |

### prompt 参数详解

| prompt 值 | 行为 |
|-----------|------|
| 无（不传） | 授权服务器自行判断（已登录可跳过登录页） |
| none | 必须已登录且无交互，否则返回错误（适合静默刷新） |
| login | 强制重新认证（即使用户已登录） |
| consent | 强制展示授权页面（即使之前已授权） |
| select_account | 展示账号选择页面（多账号场景） |

> `prompt=none` 与 `id_token_hint` 的组合是**静默刷新**的标准做法。

### 授权端点响应

```
成功响应（用户同意授权）：

HTTP/1.1 302 Found
Location: https://client.example.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xcoiv98wsefkj

错误响应（用户拒绝或出错）：

HTTP/1.1 302 Found
Location: https://client.example.com/callback?error=access_denied&state=xcoiv98wsefkj
```

| error 参数 | 说明 |
|------------|------|
| invalid_request | 请求缺少参数、参数无效 |
| unauthorized_client | 客户端无权限使用此方法 |
| access_denied | 用户或授权服务器拒绝请求 |
| unsupported_response_type | 不支持的 response_type |
| invalid_scope | scope 无效或超出范围 |
| server_error | 服务器内部错误 |
| temporarily_unavailable | 服务器临时不可用 |

## 4.3 令牌端点 (/token) — RFC 6749 Section 3.2

Token Endpoint 是**后端通信**（Back Channel），所有请求必须通过 HTTPS POST 发送。

### 客户端认证（Client Authentication）

授权服务器必须先认证客户端身份，再颁发 Token。

```http
# 方式一：HTTP Basic Auth（推荐）
POST /token HTTP/1.1
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

# 方式二：POST Body（不推荐）
POST /token HTTP/1.1

client_id=s6BhdRkqt3
&client_secret=7Fjfp0ZBr1KtDRbnfVdmIw

# 方式三：JWT Assertion（私钥签名，高级场景）
POST /token HTTP/1.1

client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIs...
```

### grant_type 参数汇总

| grant_type | 使用场景 | 相关 RFC |
|-----------|----------|----------|
| authorization_code | Auth Code 模式 | RFC 6749 |
| client_credentials | M2M | RFC 6749 |
| password | Resource Owner Password（不推荐） | RFC 6749 |
| refresh_token | 刷新 Access Token | RFC 6749 |
| urn:ietf:params:oauth:grant-type:device_code | 设备模式 | RFC 8628 |
| urn:ietf:params:oauth:grant-type:jwt-bearer | JWT 断言 | RFC 7523 |
| urn:ietf:params:oauth:grant-type:token-exchange | Token 交换 | RFC 8693 |
| urn:openid:params:grant-type:ciba | CIBA（后端发起认证） | OpenID CIBA |

### Refresh Token 续期请求

```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&scope=read
```

> Refresh Token 续期时的 scope 必须 ≤ 原始 scope。
> 如果 Refresh Token 轮换启用，响应中会返回新的 Refresh Token。

### 令牌响应中的 JSON 字段

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8",
  "scope": "read write",
  "id_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| access_token | 是 | string | 访问令牌 |
| token_type | 是 | string | 通常为 `Bearer`（大小写不敏感） |
| expires_in | 推荐 | integer | 有效期秒数 |
| refresh_token | 条件 | string | 仅授权码模式等需要长期访问的场景 |
| scope | 条件 | string | 实际授予的 scope（可能小于请求的） |
| id_token | OIDC | string | OIDC 身份令牌 |

## 4.4 令牌内省端点 (/introspect) — RFC 7662

资源服务器调用此端点验证 Opaque Token 的有效性：

```http
POST /introspect HTTP/1.1
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

token=mF_9.B5f-4.1JqM
&token_type_hint=access_token
```

成功响应（Token 有效）：

```json
{
  "active": true,
  "scope": "read write",
  "client_id": "s6BhdRkqt3",
  "username": "alice@example.com",
  "token_type": "Bearer",
  "exp": 1650000000,
  "iat": 1649996400,
  "nbf": 1649996400,
  "sub": "user-123",
  "aud": ["api1", "api2"],
  "iss": "https://auth.example.com",
  "jti": "unique-token-id-xyz"
}
```

Token 无效/过期/被撤销：

```json
{
  "active": false
}
```

> **性能优化**：在高并发场景下，不应每个请求都做 Introspection。可以添加短期缓存（如 60 秒），在 Active 为 True 时缓存，为 False 时不缓存。

## 4.5 令牌吊销端点 (/revoke) — RFC 7009

```http
POST /revoke HTTP/1.1
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

token=45ghiukldjahdnhzdauz
&token_type_hint=refresh_token
```

> 吊销端点返回 200 即表示成功（无论 Token 是否存在）。不存在的 Token 也应返回 200，防止 Token 枚举攻击。

## 4.6 元数据发现端点 (/.well-known/oauth-authorization-server) — RFC 8414

客户端通过此端点自动发现授权服务器的配置，免去手动配置端点 URL。

```http
GET /.well-known/oauth-authorization-server
```

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/oauth2/authorize",
  "token_endpoint": "https://auth.example.com/oauth2/token",
  "jwks_uri": "https://auth.example.com/oauth2/jwks",
  "registration_endpoint": "https://auth.example.com/oauth2/register",
  "scopes_supported": ["openid", "profile", "email", "address"],
  "response_types_supported": ["code", "token"],
  "response_modes_supported": ["query", "fragment", "form_post"],
  "grant_types_supported": ["authorization_code", "client_credentials", "refresh_token"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post", "private_key_jwt"],
  "code_challenge_methods_supported": ["S256"],
  "revocation_endpoint": "https://auth.example.com/oauth2/revoke",
  "introspection_endpoint": "https://auth.example.com/oauth2/introspect",
  "service_documentation": "https://auth.example.com/docs"
}
```

> 对于 OIDC 场景，使用 `/.well-known/openid-configuration`（是 oauth-authorization-server 的子集 + OIDC 扩展字段）。

---

# 5. OAuth2 参数安全性深度分析

## 5.1 redirect_uri 安全

### 严格匹配 vs 模糊匹配

```
授权服务器必须实现严格字符串匹配：

问题案例：
  注册 redirect_uri: https://app.example.com/callback
  攻击者请求:       https://app.example.com/callback.evil.com
  模糊匹配结果:     "前缀匹配" → 攻击成功
  严格匹配结果:     "不匹配" → 攻击失败

  注册 redirect_uri: https://app.example.com/callback
  攻击者请求:       https://app.example.com/callback/../other
  模糊匹配结果:     "允许子路径" → 攻击成功
  严格匹配结果:     "不匹配" → 攻击失败
```

### redirect_uri 的开放重定向攻击

```
# 攻击链条：
1. 攻击者构造：
   https://auth.example.com/authorize?client_id=xxx&redirect_uri=https://client.example.com/callback?next=https://evil.com

2. 授权服务器重定向：
   https://client.example.com/callback?next=https://evil.com&code=real_code

3. client.example.com 处理不严，302 到 next 参数：
   https://evil.com?code=real_code

4. 攻击者获取 authorization code

防御：
- 授权服务器只验证 redirect_uri 的 scheme+host+path（不含 query）
- 客户端不信任回调 URL 中的额外参数做跳转
- 客户端实现稳定的回调端点，不基于参数做任意重定向
```

## 5.2 state 参数的 CSRF 防护机制

```
攻击场景（缺少 state 参数）：

1. 攻击者用自己的 GitHub 账号完成 OAuth2 流程
   获得 authorization code: code_attacker

2. 攻击者在公共论坛发送精心构造的链接：
   https://client.example.com/callback?code=code_attacker

3. 受害者点击（或浏览器预加载 / 通过 <img> 标签触发）

4. 客户端后端接收到受害者的 session + 攻击者的 code
   用 code_attacker 换取攻击者的 Access Token
   将攻击者的 GitHub 账号绑定到受害者的客户端账号

5. 攻击者通过绑定的 GitHub 账号登录受害者的客户端账号

state 防御：

客户端生成：state = session["oauth_state"] = random_string
请求参数：  state=random_string
回调验证：  request.state == session["oauth_state"]
          不等 → 拒绝请求
```

---

# 6. Token 存储最佳实践

## 6.1 各平台 Token 存储策略

| 平台 | Access Token 存储 | Refresh Token 存储 | 说明 |
|------|------------------|-------------------|------|
| 后端 Web | 内存 + 数据库加密 | 数据库（AES-256-GCM 加密） | 不应在客户端可见 |
| SPA | 内存（JS 变量） | BFF（Backend For Frontend） | 绝不用 localStorage |
| iOS | Keychain | Keychain | 系统级安全存储 |
| Android | EncryptedSharedPreferences | Android Keystore | 硬件级加密 |
| CLI | 文件系统（权限 600） | 文件系统（权限 600） | 或系统 keyring |

## 6.2 SPA Token 存储深度分析

```
为什么 SPA 不能用 localStorage？

问题一：XSS 攻击
  - localStorage 可被任何 JS 代码读取
  - 一个第三方 npm 包的恶意代码就能窃取所有 Token
  - 无法通过 CSP 或任何纯前端手段完全防御

问题二：无自动过期机制
  - localStorage 数据持久化
  - 关闭浏览器不会清除
  - 公共电脑上的 Token 会遗留

问题三：同源策略不够
  - 同一域名下的所有 JS 都能读取 localStorage
  - 广告脚本、分析脚本、CDN 引入的第三方代码都能读取

推荐方案（SPA）：

方案 A：BFF（Backend For Frontend）模式
  ┌──────┐  Session Cookie  ┌─────────┐  Token    ┌─────────┐
  │ SPA  │<────────────────>│  BFF    │<────────>│ Auth    │
  │      │  HttpOnly,Secure │  Server │          │ Server  │
  └──────┘                  └─────────┘          └─────────┘
  
  优势：
  - Access Token 不进入浏览器
  - 使用 HttpOnly Secure SameSite Cookie 维护会话
  - BFF 代为管理 Token 刷新
  - 符合机密客户端安全模型

方案 B：Token 仅存内存 + Service Worker
  - Token 存在 JS 闭包变量中
  - 刷新页面后需重新获取（静默刷新）
  - Service Worker 可拦截请求并注入 Token（更安全）
```
