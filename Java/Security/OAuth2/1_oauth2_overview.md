# 1. OAuth2 定义与核心定位

OAuth2（Open Authorization 2.0）是一个**授权框架协议**，由 IETF OAuth 工作组制定（RFC 6749），核心目标是允许**第三方应用在不知道用户密码的前提下，获得用户对受保护资源的有限访问权限**。

## 1.1 OAuth2 解决的核心问题

在 OAuth 出现之前，第三方应用要访问用户数据，常见做法是**密码共享**：

1. 用户把 Google 账号密码告诉某笔记应用
2. 笔记应用用该密码登录 Google 拉取联系人
3. 问题：笔记应用拥有完整权限，用户无法撤销、无法控制范围，密码泄露后果严重

OAuth2 的解决思路：**不共享密码，而是颁发一个有时间限制、可撤销、限定范围的 "令牌（Token）"**。

```
传统密码共享模式：
  用户 ──(密码)──> 第三方应用 ──(密码)──> 资源服务器
  问题：第三方持有完整凭证，不可控

OAuth2 令牌授权模式：
  用户 ──(授权确认)──> 授权服务器 ──(Token)──> 第三方应用 ──(Token)──> 资源服务器
  优势：Token 可限范围、有时效、可撤销、不可获取密码
```

## 1.2 OAuth2 vs OAuth1.0

| 对比维度 | OAuth 1.0 | OAuth 2.0 |
|----------|-----------|-----------|
| 签名要求 | 每次请求需计算签名（HMAC-SHA1） | 依赖 HTTPS，不要求请求签名 |
| 客户端类型 | 仅支持 Web 应用 | 支持 Web、移动端、SPA、IoT |
| Token 类型 | 单一 Access Token | Access Token + Refresh Token |
| 授权流程 | 单一 3 步流程 | 多种授权模式（4 种标准 + 扩展） |
| 角色分离 | 角色划分不清晰 | 明确 4 种角色 |
| 实现复杂度 | 签名逻辑复杂，客户端实现成本高 | 依赖 TLS，实现成本低 |
| API 风格 | 非 RESTful | RESTful 友好 |

OAuth2 **不是 OAuth1 的升级版**，而是全新设计，两者不兼容。OAuth2 的主要牺牲是安全性部分转移给了 TLS，但换来了极大的简洁性和灵活性。

## 1.3 OAuth2 不是什么

- **不是认证协议**：OAuth2 解决的是"你能做什么"，不是"你是谁"。OpenID Connect（OIDC）是在 OAuth2 基础上构建的身份认证层
- **不是单点登录（SSO）**：虽然 OAuth2 常被用于 SSO，但 SSO 是更高层概念
- **不是细粒度权限控制**：OAuth2 的 Scope 是粗粒度的资源范围界定

> OAuth2 规范的权威文档：
> - RFC 6749 — OAuth 2.0 授权框架
> - RFC 6750 — Bearer Token 用法
> - RFC 6819 — OAuth 2.0 威胁模型与安全考量

---

# 2. 核心角色

OAuth2 定义了四个核心角色，理解各角色的职责是掌握 OAuth2 的基础。

## 2.1 资源拥有者（Resource Owner）

资源拥有者是**能够授予对受保护资源访问权限的实体**。在大多数场景下，资源拥有者就是**最终用户**。

```
角色职责：
- 拥有受保护资源（如 Google 联系人、GitHub 仓库）
- 授权第三方应用访问其资源
- 可以随时撤销授权
```

> 特殊场景：机器对机器通信（M2M）中，资源拥有者可能是一个系统或服务，而非人类用户。此时使用 Client Credentials 模式，不需要用户交互。

## 2.2 资源服务器（Resource Server）

资源服务器**托管受保护资源**，能够接受并使用 Access Token 来响应受保护的资源请求。

```
核心职责：
1. 存储和管理受保护资源（用户数据、文件、API 等）
2. 验证请求中携带的 Access Token（有效性、过期、Scope）
3. 根据 Token 的 Scope 决定是否返回资源
4. 可独立部署，与授权服务器分离
```

资源服务器**不负责发 Token**，Token 校验可以通过以下方式：
- **本地校验**（JWT 自包含 Token）：通过公钥验证签名
- **远程校验**（不透明 Token）：调用授权服务器的 Introspection 端点
- **共享存储**（如 Redis）：通过共享 Token 存储查询

## 2.3 客户端（Client）

客户端是**代表资源拥有者请求受保护资源的应用**。

```
客户端分类（按安全能力）：
┌──────────────────────────────────────────────────────┐
│ 公开客户端                              机密客户端   │
│ (Public Client)                        (Confidential)│
├──────────────────────────────────────────────────────┤
│ SPA / 单页应用                          后端 Web 应用 │
│ 移动 App (Native)                       服务器到服务器 │
│ 桌面应用                                (M2M)         │
│ ┌─ 无法安全保存密钥                                 │
│ └─ 使用 PKCE 增强安全                                │
│                                         ┌─ 能安全保存密钥  │
│                                         └─ 使用 client_secret│
└──────────────────────────────────────────────────────┘
```

**客户端注册**：客户端需要在授权服务器上注册，获取：
- `client_id`：公开标识符
- `client_secret`：仅机密客户端使用
- `redirect_uri`：回调地址白名单
- `scope`：可请求的权限范围

## 2.4 授权服务器（Authorization Server）

授权服务器是 OAuth2 的**核心中枢**，负责认证资源拥有者、获取授权、颁发 Token。

```
授权服务器的关键职责：
1. 认证资源拥有者（通常是用户登录）
2. 展示授权页面，获取用户同意
3. 验证客户端身份（client_id + client_secret）
4. 颁发 Access Token 和 Refresh Token
5. 提供 Token 校验端点（Introspection）
6. 提供 Token 吊销端点（Revocation）
7. 管理客户端注册信息
8. 记录审计日志
```

授权端点的 URL 在 RFC 8414（OAuth 2.0 Authorization Server Metadata）中标准化，客户端可通过发现端点自动获取：

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/oauth2/authorize",
  "token_endpoint": "https://auth.example.com/oauth2/token",
  "jwks_uri": "https://auth.example.com/oauth2/jwks",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code", "token"],
  "grant_types_supported": ["authorization_code", "client_credentials"]
}
```

---

# 3. OAuth2 协议概览

## 3.1 抽象协议流程

OAuth2 的抽象协议流程如下（RFC 6749 Section 1.2）：

```
   +--------+                               +---------------+
   |        |--(A)- Authorization Request ->|   Resource    |
   |        |                               |     Owner     |
   |        |<-(B)- Authorization Grant ---|               |
   |        |                               +---------------+
   |        |
   |        |                               +---------------+
   |        |--(C)- Authorization Grant -->| Authorization |
   | Client |                               |     Server    |
   |        |<-(D)----- Access Token ------|               |
   |        |                               +---------------+
   |        |
   |        |                               +---------------+
   |        |--(E)----- Access Token ----->|    Resource   |
   |        |                               |     Server    |
   |        |<-(F)--- Protected Resource --|               |
   +--------+                               +---------------+
```

步骤说明：
- **(A)** 客户端向资源拥有者请求授权
- **(B)** 资源拥有者返回授权许可（Authorization Grant）
- **(C)** 客户端用授权许可向授权服务器请求 Access Token
- **(D)** 授权服务器认证客户端并验证授权许可，颁发 Access Token
- **(E)** 客户端用 Access Token 向资源服务器请求受保护资源
- **(F)** 资源服务器验证 Token 后返回受保护资源

## 3.2 核心数据结构

### 3.2.1 Access Token 响应（RFC 6749 Section 5.1）

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "read write"
}
```

### 3.2.2 Scope（权限范围）

Scope 是 OAuth2 的**粗粒度权限控制机制**，是一个由空格分隔的字符串列表，表示客户端请求的访问范围：

```
scope=read write profile
```

> Scope 的语义由授权服务器定义，OAuth2 规范本身不规定 Scope 的具体含义。常见实践：
> - `read` / `write`：基本读写权限
> - `openid` / `profile` / `email`：OIDC 标准 Scope
> - `user:read` / `admin:write`：分层级命名

---

# 4. 四种核心授权模式

OAuth2 定义了四种标准授权模式，每种适用于不同的场景。这里给出概览，详细分析见第 2 节。

| 授权模式 | RFC 章节 | 适用场景 | 用户交互 | 安全关键点 |
|----------|----------|----------|----------|------------|
| Authorization Code | Section 4.1 | Web 应用、有后端服务 | 需要 | PKCE、state 参数、redirect_uri 校验 |
| Implicit（已不推荐） | Section 4.2 | 历史遗留 SPA | 需要 | Access Token 暴露在 URL Fragment |
| Resource Owner Password | Section 4.3 | 高度信任的第一方应用 | 需要（但客户端收集密码） | 不应再使用 |
| Client Credentials | Section 4.4 | M2M、后台服务 | 不需要 | client_secret 安全性 |

```
使用场景决策树：

有用户在操作吗？
├── 是 ──> 第三方应用？
│          ├── 是 ──> 有后端服务？──> Authorization Code + PKCE（推荐）
│          │                           Authorization Code（无 PKCE 时仅限机密客户端）
│          └── 否 ──> 第一方原生应用？──> Authorization Code + PKCE
│                   └── 高度信任的自家应用（不推荐）──> Resource Owner Password（已不推荐）
│
└── 否 ──> 后台服务间通信？──> Client Credentials
```

## 4.1 Authorization Code（授权码模式）

这是 OAuth2 的**核心授权模式**，也是唯一推荐在用户参与的 Web 应用中使用的标准模式。

```
流程概览（不含 PKCE）：
1. 用户访问客户端，点击"用 GitHub 登录"
2. 客户端重定向到授权服务器 /authorize
3. 用户登录授权服务器、确认授权
4. 授权服务器重定向回客户端，附上 authorization code
5. 客户端后端用 code + client_secret 换 Access Token
6. 客户端用 Access Token 访问资源

关键优势：
- Access Token 不经过浏览器（后端换 Token）
- authorization code 仅一次有效、短期有效
- 可配合 Refresh Token 获得长期访问能力
```

## 4.2 Implicit（隐式模式）

**OAuth 2.0 Security Best Current Practice 已明确不推荐使用**，但理解其原理有助于了解历史演进。

```
历史原因：早期的 JS 应用没有跨域请求能力，无法直接访问 Token 端点，
所以设计为从 URL Fragment 直接返回 Access Token。

安全隐患：
- Access Token 暴露在 URL Fragment 中
- 浏览器历史记录可能泄露 Token
- 无法使用 Refresh Token（安全限制）
- 无法验证 Access Token 的真实接收者
```

替代方案：**Authorization Code + PKCE** 已经解决了 SPA 的无后端问题。

## 4.3 Resource Owner Password Credentials

**OAuth 2.0 Security Best Current Practice 已不推荐**，仅限高度信任的第一方应用和遗留系统迁移。

```
工作方式：客户端直接收集用户名/密码，发送到 Token 端点换取 Access Token。

POST /token
  grant_type=password
  &username=user
  &password=pass
  &client_id=xxx

被淘汰的原因：
- 违反 OAuth2 的基本理念（不应接触用户密码）
- 无法区分"用户在与谁交互"
- 不支持 MFA（多因素认证）
- 无法做到最小权限原则
```

## 4.4 Client Credentials

用于**服务器到服务器**（Server-to-Server）的通信，无需用户参与。

```
典型场景：
- 后台定时任务调用 API
- 微服务间安全通信
- CI/CD 系统访问资源
- 物联网设备上报数据

POST /token
  grant_type=client_credentials
  &client_id=xxx
  &client_secret=xxx
  &scope=read

特点：
- 没有 Refresh Token（客户端可直接重新请求）
- Token 通常不与特定用户关联
- 适合自动化的、非交互式的访问
```

---

# 5. OAuth2 安全机制

## 5.1 PKCE（Proof Key for Code Exchange）

PKCE（RFC 7636）是 Authorization Code 模式的安全增强，**在所有使用授权码模式的情况下都应启用**。

```
PKCE 工作流程：

客户端                                            授权服务器
  │                                                  │
  │  1. 生成 code_verifier（随机字符串 43-128 字符）     │
  │  2. 计算 code_challenge = SHA256(code_verifier)   │
  │                                                  │
  │  3. /authorize?code_challenge=xxx                 │
  │     &code_challenge_method=S256                ──>│
  │                                                  │  4. 存储 code_challenge
  │  5. 返回 authorization code                   <──│
  │                                                  │
  │  6. /token?code=xxx&code_verifier=xxx          ──>│
  │                                                  │  7. 验证 code_challenge == SHA256(code_verifier)
  │  8. 返回 Access Token                         <──│
```

PKCE 解决的安全问题：

| 攻击场景 | 无 PKCE | 有 PKCE |
|----------|---------|---------|
| 授权码拦截攻击 | 攻击者截获 code 即可换 Token | 需要持有 code_verifier 才能换 Token |
| 恶意 App 注册同 scheme | 可能截获回调 URL | 无法完成 Token 交换 |
| 授权码重放攻击 | 可能成功 | 无法重放 |

## 5.2 State 参数（防 CSRF）

```
攻击场景：
1. 攻击者用自己的账号获取一个 authorization code
2. 攻击者诱导受害者点击链接，将该 code 绑定到受害者的客户端会话
3. 受害者客户端用攻击者的 code 换 Token，将攻击者的账号绑定到受害者身份
4. 攻击者通过已绑定的自己的账号访问受害者的资源

防御：state 参数
- 客户端生成随机 state，存储在 session 中
- 授权请求携带 state
- 授权服务器在回调中原样返回 state
- 客户端验证 state 是否匹配，不匹配则拒绝
```

## 5.3 Redirect URI 校验

授权服务器必须严格校验 `redirect_uri`：

```
安全要求：
1. 必须完整匹配（不能是部分匹配或正则模糊匹配）
2. 不允许开放重定向（Open Redirect）
3. 生产环境只允许 HTTPS（localhost 除外）
4. 不允许使用自定义 scheme 除非经过严格注册

攻击示例（开放重定向漏洞）：
客户端请求：redirect_uri=https://legit-app.com/callback
攻击者修改：redirect_uri=https://legit-app.com/callback@evil.com
如果授权服务器使用宽松匹配，authorization code 会发往 evil.com
```

## 5.4 Token 安全传输与存储

| Token 类型 | 传输要求 | 存储要求（Web） | 存储要求（移动端） |
|------------|----------|-----------------|-------------------|
| Access Token | 仅 HTTPS，Bearer 头 | 内存（不建议 localStorage） | Keychain（iOS）/ KeyStore（Android） |
| Refresh Token | 仅 HTTPS | HttpOnly Secure Cookie / 后端存储 | Keychain / KeyStore |
| Authorization Code | 仅 HTTPS（参数在 URL） | 一次性使用后立即丢弃 | 不适用 |

---

# 6. OAuth2 与相关协议的关系

## 6.1 OAuth2 + OIDC

```
    ┌──────────────────────────────┐
    │       OpenID Connect          │
    │  ┌────────────────────────┐  │
    │  │      OAuth 2.0          │  │
    │  │  ┌──────────────────┐  │  │
    │  │  │    HTTP + TLS     │  │  │
    │  │  └──────────────────┘  │  │
    │  └────────────────────────┘  │
    └──────────────────────────────┘
```

- **OAuth2**：授权协议，"这个第三方应用能访问你的哪些资源"
- **OIDC**：在 OAuth2 之上构建的身份认证层，"这个用户是谁"
- **JWT**：Token 的编码格式，常被 OAuth2 和 OIDC 使用（但不是必须）

## 6.2 OAuth2 + JWT

OAuth2 规范本身没有规定 Access Token 的格式。实践中主要两种：

| 类型 | 特点 | 校验方式 |
|------|------|----------|
| 不透明 Token（Opaque） | 随机字符串，无自包含信息 | 调用 Introspection 端点 |
| JWT Token（自包含） | JSON Web Token，内嵌声明 | 本地验证签名即可 |

JWT Access Token 的优势：
- 资源服务器可离线验证，减少网络往返
- 自包含用户信息和权限，无需查数据库
- 适合分布式高并发场景

JWT Access Token 的劣势：
- 无法即时撤销（需配合黑名单或短有效期）
- Token 体积较大（尤其包含较多 claims 时）
- 撤销依赖过期时间，时效性不如不透明 Token

## 6.3 OAuth2 + SAML

| 对比维度 | SAML 2.0 | OAuth2 + OIDC |
|----------|----------|---------------|
| 协议年代 | 2005 | 2012+ |
| 数据格式 | XML | JSON / JWT |
| 设计理念 | 企业 SSO | Web / 移动 / API 授权 |
| 移动端支持 | 差 | 原生支持 |
| 复杂度 | 高（XML 签名/加密） | 相对低 |
| 典型场景 | 企业内部系统 | 互联网应用 |

---

# 7. OAuth2 生态与扩展标准

| RFC 编号 | 名称 | 说明 |
|----------|------|------|
| RFC 6749 | OAuth 2.0 Authorization Framework | 核心框架 |
| RFC 6750 | Bearer Token Usage | Token 使用方式 |
| RFC 7636 | PKCE | 授权码拦截防护 |
| RFC 7662 | Token Introspection | Token 校验端点 |
| RFC 7009 | Token Revocation | Token 吊销端点 |
| RFC 7519 | JSON Web Token (JWT) | JWT 格式定义 |
| RFC 8414 | Authorization Server Metadata | 授权服务器元数据发现 |
| RFC 8628 | Device Authorization Grant | 设备授权模式（TV/IoT） |
| RFC 8707 | Resource Indicators | 多资源服务器标识 |
| RFC 9068 | JWT Access Token Profile | JWT 格式的 Access Token 规范 |
| RFC 9126 | Pushed Authorization Requests (PAR) | 推送授权请求 |
| RFC 9207 | OAuth 2.0 Authorization Server Issuer Identification | 颁发者标识 |
| RFC 9449 | DPoP (Demonstration of Proof-of-Possession) | Token 持有证明 |

---

# 8. OAuth2 常见误区

## 8.1 "OAuth2 = 单点登录"

OAuth2 是**授权**协议，不是认证协议。单点登录（SSO）可以通过 OAuth2 + OIDC 实现，但 OAuth2 本身不处理身份认证。

## 8.2 "JWT = OAuth2"

JWT 是一种 Token 格式，OAuth2 是一种授权框架。OAuth2 可以使用 JWT 作为 Token 格式，也可以使用不透明 Token，反之 JWT 也可以用于非 OAuth2 场景。

## 8.3 "使用 OAuth2 就安全了"

OAuth2 的安全性高度依赖**实现质量**。RFC 6749 本身存在一些安全考量不足的地方（如 Implicit 模式），需要配合最新的安全最佳实践（OAuth 2.0 Security Best Current Practice）才能做到安全。

## 8.4 "OAuth2 支持细粒度权限"

OAuth2 的 Scope 是**粗粒度**的资源访问范围控制，不适合做字段级别或记录级别的细粒度权限控制。细粒度权限应在应用层（RBAC/ABAC）实现。

---

# 9. 版本与规范状态

| 年份 | 事件 |
|------|------|
| 2007 | OAuth 1.0 首次发布 |
| 2010 | OAuth 1.0a（修复会话固定攻击）发布 |
| 2012 | OAuth 2.0（RFC 6749）发布 |
| 2014 | OpenID Connect 1.0 正式发布 |
| 2015 | PKCE (RFC 7636) 发布 |
| 2018 | OAuth 2.0 Security Best Current Practice（草稿） |
| 2021 | OAuth 2.1 草案发布（整合最佳实践，简化协议） |
| 2024 | OAuth 2.1 仍在草案阶段，持续推进中 |

## 9.1 OAuth 2.1 的主要变化

OAuth 2.1 不是一个新协议，而是对 OAuth 2.0 的整合和简化：

1. **移除 Implicit 模式**：彻底删除
2. **强制 PKCE**：所有使用 Authorization Code 的流程
3. **移除 Resource Owner Password 模式**：不再作为标准授权模式
4. **强制使用 exact string matching 校验 redirect_uri**
5. **Refresh Token 安全增强**：发送方限制、轮换机制
6. **不再使用 Bearer Token 通过 URL 参数传递**

> 本文档基于 OAuth 2.0 (RFC 6749) + 最新安全最佳实践编写，内容覆盖 OAuth 2.1 草案中的推荐做法。
