# 1. JWT 知识体系总览

## 1.1 核心知识地图

```
JWT 知识体系
├── 基础概念
│   ├── JWT 是什么
│   ├── JWT 的组成部分（Header + Payload + Signature）
│   ├── Base64URL 编码
│   └── 无状态认证理念
│
├── JOSE 规范族
│   ├── JWT (RFC 7519) — 结构定义
│   ├── JWS (RFC 7515) — 签名
│   ├── JWE (RFC 7516) — 加密
│   ├── JWK (RFC 7517) — 密钥表示
│   └── JWA (RFC 7518) — 算法
│
├── 签名算法
│   ├── 对称：HS256, HS384, HS512
│   ├── 非对称-RSA：RS256, RS384, RS512, PS256
│   ├── 非对称-EC：ES256, ES384, ES512
│   └── 非对称-EdDSA：EdDSA (Ed25519)
│
├── Claims 设计
│   ├── 注册声明：iss, sub, aud, exp, nbf, iat, jti
│   ├── 公共声明
│   └── 私有声明
│
├── 认证流程
│   ├── 双令牌模式（Access Token + Refresh Token）
│   ├── Refresh Token 轮换
│   └── Token 吊销机制
│
├── 安全攻防
│   ├── alg=none 攻击
│   ├── 密钥混淆攻击
│   ├── 重放攻击
│   ├── 弱密钥暴力破解
│   └── XSS/CSRF 结合攻击
│
├── Java 库
│   ├── JJWT (Okta)
│   ├── Nimbus JOSE+JWT
│   ├── jose4j
│   └── Auth0 JWT
│
├── 集成
│   ├── Spring Security
│   ├── OAuth2 Resource Server
│   └── Spring Authorization Server
│
└── 工程实践
    ├── 密钥管理
    ├── 性能优化
    ├── 监控告警
    └── 最佳实践清单
```

---

# 2. 面试高频问题

## 2.1 JWT 基础

### Q1：JWT 的组成部分是什么？每个部分的作用？

**答案**：JWT 由三部分组成，以 `.` 分隔：

1. **Header（头部）**：Base64URL 编码的 JSON 对象，包含令牌类型 (`typ`) 和签名算法 (`alg`)
2. **Payload（负载）**：Base64URL 编码的 JSON 对象，包含 Claims（声明），如 `sub`、`exp`、`iss` 等
3. **Signature（签名）**：对 Header 和 Payload 的签名，保证数据完整性

### Q2：Base64 和 Base64URL 的区别？为什么 JWT 使用 Base64URL？

**答案**：
- Base64 使用 `+`、`/`、`=`（填充）
- Base64URL 使用 `-`、`_`、省略 `=`
- JWT 经常在 URL 参数或 HTTP Header 中传递，Base64URL 避免了 `+` 被解码为空格、`/` 被解释为路径分隔符

### Q3：JWT 是加密的吗？

**答案**：默认情况下，JWT 只做 Base64URL **编码**和**签名**，不做**加密**。任何人都可以解码查看内容。如需加密保护，应使用 JWE 规范。

### Q4：JWT vs Session 的区别？各自适用什么场景？

**答案**：

| JWT | Session |
|-----|---------|
| 无状态，客户端持有 | 有状态，服务端存储 |
| 水平扩展方便 | 扩展需 Session 共享 |
| 天然支持跨域 | 受同源策略限制 |
| 吊销复杂（需黑名单） | 吊销简单 |
| 适合分布式/微服务/移动端 | 适合传统单体应用 |

---

## 2.2 JWT 安全

### Q5：什么是 alg=none 攻击？如何防范？

**答案**：攻击者将 JWT Header 中的 `alg` 改为 `none` 并删除签名部分。如果服务端未正确限制算法，可能接受未签名的恶意 JWT。

**防范**：
1. 不同算法使用独立 Parser 实例
2. 明确算法白名单，拒绝 `none`
3. JJWT 0.12+ 通过密钥类型自动限制算法

### Q6：什么是密钥混淆攻击（Algorithm Confusion Attack）？

**答案**：服务端使用 RSA 公钥验证 JWT，但攻击者获取 RSA 公钥后，将其作为 HMAC 的对称密钥，用 HS256 算法伪造 JWT。如果服务端根据 `alg` 字段动态选择密钥而不做算法类型检查，就会用 RSA 公钥验证 HS256 签名，从而通过验证。

**防范**：不同算法使用独立验证器，不根据 Header 中的 `alg` 动态选择验证方式。

### Q7：如何安全地存储 JWT？

**答案**：

| 位置 | 安全性 | 说明 |
|------|--------|------|
| localStorage | 低 | 易受 XSS 攻击 |
| HttpOnly Cookie | 高 | JS 不可访问，天然防 XSS |
| 内存变量 | 最高 | 页面刷新后丢失，需配合 Refresh Token |

推荐方案：Access Token 存储在内存，Refresh Token 存储在 HttpOnly Cookie。

### Q8：JWT 过期后如何处理？

**答案**：使用双令牌模式（Access Token + Refresh Token）：
1. Access Token（短有效期 5-30min）过期后，客户端使用 Refresh Token 请求新 Access Token
2. Refresh Token 也应有过期时间，过期后需重新登录
3. 实现 Refresh Token 轮换，防止 Token 泄漏

---

## 2.3 算法与密钥

### Q9：HS256、RS256、ES256 的区别和选择？

**答案**：

| 算法 | 类型 | 密钥 | 签名大小 | 速度 | 建议场景 |
|------|------|------|----------|------|----------|
| HS256 | 对称 | 共享密钥 ≥256位 | 43 字符 | 极快 | 内部服务间 |
| RS256 | 非对称 | RSA ≥2048位 | 342 字符 | 中等 | 互联互通 |
| ES256 | 非对称 | ECDSA P-256 | 86 字符 | 快 | 移动端/新项目 |

- HS256：同一服务签发和验证
- RS256/ES256：多方验证，只需分发公钥

### Q10：为什么 RS256 的签名比 HS256 大很多？

**答案**：RS256 签名的本质是 RSA 运算结果，对于 2048 位密钥，签名固定为 256 字节（2048 位）。经过 Base64URL 编码后约 342 字符。而 HS256 的 HMAC 输出是固定的 32 字节（256 位），编码后约 43 字符。

### Q11：什么是 JWK？和 JWT 的关系？

**答案**：JWK（JSON Web Key）是以 JSON 格式表示密钥的标准（RFC 7517）。JWT 使用 JWK 来分发公钥（通过 JWK Set 端点），接收方根据 JWT Header 中的 `kid` 从 JWK Set 中查找对应的公钥进行验证。

---

## 2.4 认证流程

### Q12：为什么要用双令牌（Access Token + Refresh Token）？

**答案**：
1. **安全与体验平衡**：Access Token 短有效期（15min），即使泄露影响有限；Refresh Token 长有效期（7天），用户无需频繁登录
2. **权限实时性**：Access Token 短有效期保证权限变更能快速生效
3. **吊销能力**：Refresh Token 服务端存储，可主动吊销

### Q13：Refresh Token 为什么需要轮换？

**答案**：
1. **检测泄漏**：如果攻击者使用窃取的 Refresh Token，合法用户持有的旧 Refresh Token 就会失效，从而暴露泄漏
2. **限制影响**：每个 Refresh Token 只能使用一次
3. **符合安全合规**：OAuth2 安全最佳实践（RFC 6819）建议实现 Refresh Token 轮换

### Q14：如何实现 JWT 的吊销（登出）？

**答案**：
1. **黑名单（最常用）**：将 JWT 的 `jti` 加入 Redis 黑名单，TTL 与 JWT 剩余有效期一致
2. **版本号递增**：用户密码修改时递增版本号，所有旧 Token 失效
3. **短有效期**：Access Token 15 分钟过期，自然失效
4. **客户端清除**：清除存储的 JWT，但无法阻止 Token 被盗用

---

## 2.5 实践与优化

### Q15：JWT 适合存储大量权限信息吗？

**答案**：不适合。理由：
1. JWT 体积会过大，影响每次 HTTP 请求的网络开销
2. 权限变更后 JWT 仍是旧的
3. HTTP Header 有 8KB 左右的限制

建议：JWT 中只存放角色标识，详细权限通过 API 按需查询。

### Q16：JWT 验证时如何保证性能？

**答案**：
1. 预构建 `JwtParser` 实例（线程安全），重复使用
2. 使用 HMAC 签名的内部服务通信
3. 缓存 JWK Set 公钥（TTL 15 分钟）
4. 在 API 网关集中验证，后端服务信任网关
5. 选择合适的签名算法（ES256 验证速度快于 RS256）

### Q17：多个微服务如何共享 JWT 验证？

**答案**：
1. **JWK Set 端点**：授权服务提供 `/.well-known/jwks.json` 端点，各服务从此获取公钥
2. **配置中心**：通过 Spring Cloud Config 等分发公钥
3. **API 网关模式**：网关统一验证，后端服务信任网关传递的用户信息

---

## 2.6 深入原理

### Q18：JWT 如何保证无状态？

**答案**：每个 JWT 是自包含的，包含了用户身份信息和过期时间。服务端只需持有公钥即可验证 JWT，不需要查询数据库或共享 Session 存储。任何持有公钥的服务节点都可以独立验证 JWT。

### Q19：JWT 的签名输入（Signing Input）是什么？

**答案**：签名输入 = `BASE64URL(Header) + "." + BASE64URL(Payload)`

注意：签名部分本身不包含在签名输入中。验证时重新计算签名并比对。

### Q20：为什么 JWT 的签名使用恒定时间比较？

**答案**：防止时序攻击（Timing Attack）。攻击者可以通过测量签名验证的响应时间来逐字节推断正确的签名值。恒定时间比较确保无论签名是否匹配，验证时间保持一致。

---

# 3. 高频陷阱总结

| 陷阱 | 解释 |
|------|------|
| Payload 放密码 | Base64URL 非加密，任何人都可以解码 |
| 使用 alg=none | 历史上最严重的安全漏洞之一 |
| 密钥硬编码 | 代码泄露 = 密钥泄露 |
| 不验证 aud | 令牌可能在不同服务间滥用 |
| 不验证 iss | 令牌可能来自恶意签发者 |
| Access Token 有效期过长 | 泄漏后影响时间长 |
| 不实现 Refresh Token 轮换 | 无法检测令牌泄漏 |
| 在 localStorage 存储 JWT | XSS 攻击可直接读取 |
| 日志中打印完整 JWT | 日志泄露导致令牌泄露 |
| 所有服务共享 HMAC 密钥 | 任一服务泄露则全局受影响 |

---

# 4. 参考资料

## 4.1 RFC 规范

| RFC | 标题 |
|-----|------|
| RFC 7519 | JSON Web Token (JWT) |
| RFC 7515 | JSON Web Signature (JWS) |
| RFC 7516 | JSON Web Encryption (JWE) |
| RFC 7517 | JSON Web Key (JWK) |
| RFC 7518 | JSON Web Algorithms (JWA) |
| RFC 9068 | JWT Profile for OAuth 2.0 Access Tokens |
| RFC 9449 | OAuth 2.0 DPoP |

## 4.2 官方资源

- JWT 官方：https://jwt.io
- JJWT GitHub：https://github.com/jwtk/jjwt
- Nimbus JOSE+JWT：https://connect2id.com/products/nimbus-jose-jwt
- Spring Security：https://spring.io/projects/spring-security
- OAuth 2.0：https://oauth.net/2/

## 4.3 安全指南

- OWASP JWT Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html
- IANA JWT Claims Registry：https://www.iana.org/assignments/jwt/jwt.xhtml
- Auth0 JWT Handbook：https://auth0.com/resources/ebooks/jwt-handbook
