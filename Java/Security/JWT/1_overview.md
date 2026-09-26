# 1. 什么是 JWT

JWT（JSON Web Token）是一种基于 JSON 的开放标准（RFC 7519），用于在各方之间**安全地传递声明（Claims）**。它是一种紧凑、URL 安全、自包含的令牌格式，广泛应用于分布式系统中的身份认证与信息交换。

> 版本说明：本文档基于 JWT RFC 7519、JWS RFC 7515、JWE RFC 7516、JWK RFC 7517、JWA RFC 7518。

## 1.1 为什么需要 JWT

在分布式系统、微服务架构和前后端分离场景下，传统的基于 Session 的认证方式面临以下问题：

| 问题 | Session 方案 | JWT 方案 |
|------|-------------|---------|
| 水平扩展 | Session 需要共享存储（Redis/DB） | JWT 无状态，任何服务节点可独立验证 |
| CORS 跨域 | Cookie 受同源策略限制 | JWT 通过 HTTP Header 传输，天然跨域 |
| 移动端/非浏览器 | Cookie 机制不适用 | JWT 可直接存储在客户端，App 友好 |
| 服务间调用 | 难以传递认证上下文 | JWT 可在服务间直接传递，无需中转认证 |
| CSRF 防护 | 需要额外处理 | JWT 不依赖 Cookie，天然免疫 CSRF |

## 1.2 JWT 的核心特点

- **紧凑（Compact）**：JWT 体积小，适合在 URL、POST 参数或 HTTP Header 中传输
- **自包含（Self-contained）**：JWT 负载中包含了所有必要的用户信息，减少数据库查询
- **可验证（Verifiable）**：通过数字签名保证令牌的完整性和真实性
- **无状态（Stateless）**：服务端无需维护会话状态，天然支持水平扩展

## 1.3 JWT 的历史与发展

| 时间 | 里程碑 |
|------|--------|
| 2010 | Simple Web Token (SWT) 和 JSON 序列化概念出现 |
| 2011 | JWT 草案首次提出，目标是用 JSON 替代 XML 安全令牌 |
| 2015.5 | JWT (RFC 7519) + JWS (RFC 7515) + JWE (RFC 7516) + JWK (RFC 7517) + JWA (RFC 7518) 正式发布 |
| 2016+ | 成为 OAuth2/OIDC 的核心令牌格式，被广泛采用 |
| 2024+ | JWT 稳定版本，关键安全修正（如 `none` 算法禁用建议）已广泛实施 |

## 1.4 JWT 规范的组成部分

JWT 不是单一规范，而是一组 RFC 规范族：

| 规范 | RFC | 内容 |
|------|-----|------|
| **JWT** | RFC 7519 | 定义 JWT 的基本结构、Claims 类型 |
| **JWS** | RFC 7515 | JSON Web Signature：签名机制，保证完整性 |
| **JWE** | RFC 7516 | JSON Web Encryption：加密机制，保证机密性 |
| **JWK** | RFC 7517 | JSON Web Key：密钥的 JSON 表示格式 |
| **JWA** | RFC 7518 | JSON Web Algorithms：定义支持的算法集合 |
| **JWT Profile for OAuth2** | RFC 9068 | OAuth 2.0 访问令牌的 JWT 规范 |

## 1.5 JWT 与其他令牌格式对比

### 1.5.1 JWT vs SAML (Security Assertion Markup Language)

| 维度 | JWT | SAML |
|------|-----|------|
| 格式 | JSON | XML |
| 体积 | 小（JSON 紧凑） | 大（XML 冗余） |
| 解析 | 简单（原生 JSON） | 复杂（需 XML 库） |
| 适用场景 | REST API、移动端、SPA | 企业 SSO、传统 Web |
| 签名 | 灵活（多种算法） | 固定（XML Signature） |

### 1.5.2 JWT vs Opaque Token（不透明令牌）

| 维度 | JWT | Opaque Token |
|------|-----|--------------|
| 内容 | 自包含用户信息 | 随机字符串（如 UUID） |
| 验证方式 | 本地签名验证 | 需访问授权服务（Introspection） |
| 性能 | 低延迟（无需网络调用） | 需要额外网络请求 |
| 吊销 | 困难（需黑名单） | 容易（删除服务器端记录） |
| 体积 | 较大（含信息） | 小 |

### 1.5.3 JWT vs Session / Cookie

| 维度 | JWT | Session / Cookie |
|------|-----|-------------------|
| 状态 | 无状态（客户端持有） | 有状态（服务端持有） |
| 存储位置 | 客户端（LocalStorage/内存） | 服务端 + 客户端 Cookie |
| 扩展性 | 高（无需共享存储） | 需 Redis 等共享存储 |
| 安全性 | 数字签名（防篡改） | Cookie 签名 + HttpOnly |
| CSRF | 天然免疫 | 需防护 |

---

# 2. JWT 的典型应用场景

## 2.1 身份认证（Authentication）

最常见的场景：用户登录后，服务端签发 JWT，客户端携带 JWT 访问受保护资源。

```
客户端                        服务端
  │                            │
  │  POST /login (user/pass)   │
  │───────────────────────────>│
  │                            │ 验证凭证
  │  { access_token: "eyJ..."  │ 签发 JWT
  │<───────────────────────────│
  │                            │
  │  GET /api/users            │
  │  Authorization: Bearer eyJ.│
  │───────────────────────────>│
  │                            │ 验证 JWT 签名 + 过期
  │  200 OK (users data)      │
  │<───────────────────────────│
```

## 2.2 信息交换（Information Exchange）

JWT 可用于在服务之间安全传递信息。由于有签名保护，接收方可以验证发送方身份和数据的完整性。

- 微服务间传递用户上下文
- 分布式事务中的凭证传递
- 第三方 API 调用的身份证明

## 2.3 单点登录（SSO）

JWT 是实现跨域 SSO 的理想选择：

1. 用户在 SSO 认证中心登录，获得 JWT
2. 访问不同子域系统时，携带同一 JWT
3. 各系统独立验证 JWT，无需回到认证中心

## 2.4 API 授权与访问控制

JWT 的 Claims 中可携带权限信息（如 `scope`、`roles`），API 网关或服务可基于 Claims 做细粒度访问控制。

---

# 3. JWT 的局限性

## 3.1 无法主动吊销

JWT 签发后在其过期时间前一直有效。要吊销某个令牌，必须引入额外的机制：
- 黑名单（Token Blacklist）
- 令牌版本控制（Token Version）
- 短有效期 + Refresh Token

## 3.2 体积较大

JWT 的自包含特性意味着它比简单的 Session ID 大得多。每次请求都在 HTTP Header 中携带，增加了网络开销。

## 3.3 不适合存储敏感信息

JWT 的 Payload 部分仅做 Base64URL 编码（非加密），任何人都可以解码查看。敏感信息应避免放入 JWT，或使用 JWE 加密。

## 3.4 实现复杂度

正确实现 JWT 认证涉及：
- 密钥管理与轮换
- Token 刷新机制
- 安全存储（客户端）
- 防重放攻击
- 算法选择与密钥强度

---

# 4. JWT 生态与工具

## 4.1 Java 主流 JWT 库

| 库名称 | 组织 | 特点 |
|--------|------|------|
| **Nimbus JOSE+JWT** | Connect2id | 功能最完整，支持 JWE/JWK/JWA，OAuth2 生态首选 |
| **JJWT** | Okta | API 流式优雅，文档丰富，社区活跃 |
| **Auth0 JWT** | Auth0 | 轻量级，适合简单场景 |
| **jose4j** | Bitbucket | 功能全面，企业级使用 |
| **Vert.x JWT** | Eclipse | Vert.x 生态集成 |

## 4.2 在线调试工具

- **jwt.io**：JWT 官方调试器，支持 Decode/Verify
- **JWT Builder**：在线创建测试令牌
- **JWK Set Endpoint**：标准化的公钥分发端点

## 4.3 相关标准与协议

- **OAuth 2.0**（RFC 6749）：授权框架，使用 JWT 作为 Access Token
- **OpenID Connect**（OIDC）：身份认证层，JWT 作为 ID Token
- **JWT Profile for OAuth 2.0**（RFC 9068）：OAuth 2.0 访问令牌的 JWT 规范
- **JWT Secured Authorization Request (JAR)**（RFC 9101）：使用 JWT 保护 OAuth2 授权请求
- **DPoP**（RFC 9449）：Demonstration of Proof-of-Possession，绑定 JWT 到客户端密钥
