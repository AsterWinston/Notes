# 1. JWT 基础

JWT（JSON Web Token，RFC 7519）是一种紧凑、URL 安全的声明表示方式，用于在各方之间传输信息。虽然 JWT 不是 OAuth2 的必需组件，但实践中绝大多数 OAuth2/OIDC 实现都使用 JWT 作为 Token 格式。

## 1.1 JWT 的结构

```
JWT = Header.Payload.Signature（三部分，Base64URL 编码，用 . 连接）

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0
├────────── Header ──────────┤├────────── Payload ──────────┤

.dozjgNryP4J8jK2djkf...
├───── Signature ──────┤
```

## 1.2 JWT Header

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "signing-key-2024-01"
}
```

| 字段 | 说明 | 是否必需 |
|------|------|----------|
| alg | 签名算法（RS256, ES256, HS256 等） | 是 |
| typ | Token 类型，通常为 "JWT" | 推荐 |
| kid | 密钥 ID（用于选择验证公钥） | 推荐 |
| cty | 内容类型（嵌套 JWT 时使用） | 可选 |
| jku | JWK Set URL | 可选 |
| x5u | X.509 证书 URL | 可选 |
| x5t | X.509 证书指纹 | 可选 |

## 1.3 JWT Payload（Claims）

JWT 的 Payload 由 Claims（声明）组成。标准的 Registered Claims：

| Claim | 全称 | 说明 | 类型 |
|-------|------|------|------|
| iss | Issuer | 颁发者标识（URL） | String |
| sub | Subject | 主题（用户/实体标识） | String |
| aud | Audience | 受众（接收方标识） | String/String[] |
| exp | Expiration Time | 过期时间（Unix timestamp） | NumericDate |
| nbf | Not Before | 生效时间 | NumericDate |
| iat | Issued At | 颁发时间 | NumericDate |
| jti | JWT ID | 唯一标识符（防重放） | String |

---

# 2. JWT 签名算法深入分析

## 2.1 算法对比

| 算法 | 类型 | 密钥 | 签名长度 | 性能 | 使用场景 |
|------|------|------|----------|------|----------|
| HS256 | HMAC + SHA-256 | 对称密钥 | 32 bytes | 极快 | 内部服务间（密钥需安全分发） |
| HS384 | HMAC + SHA-384 | 对称密钥 | 48 bytes | 快 | 内部服务间 |
| HS512 | HMAC + SHA-512 | 对称密钥 | 64 bytes | 快 | 内部服务间 |
| RS256 | RSA + SHA-256 | 公私钥对 | 256 bytes | 慢（签名）/ 快（验证） | OAuth2/OIDC 推荐 |
| RS384 | RSA + SHA-384 | 公私钥对 | 384 bytes | 慢 | 高安全性需求 |
| RS512 | RSA + SHA-512 | 公私钥对 | 512 bytes | 慢 | 高安全性需求 |
| ES256 | ECDSA + P-256 + SHA-256 | 公私钥对 | 64 bytes | 快 | 移动端 / IoT |
| ES384 | ECDSA + P-384 + SHA-384 | 公私钥对 | 96 bytes | 快 | 移动端 |
| ES512 | ECDSA + P-521 + SHA-512 | 公私钥对 | 132 bytes | 快 | 移动端 |
| PS256 | RSA-PSS + SHA-256 | 公私钥对 | 256 bytes | 慢 | 现代推荐（安全） |
| none | 无签名 | 无 | 0 | - | 仅用于测试（禁止生产使用） |

## 2.2 算法选择指南

```
OAuth2 / OIDC 场景算法推荐：

首选：RS256 (RSA + SHA256)
  - 广泛支持（所有 JWT 库）
  - 公私钥分离（只有授权服务器持有私钥）
  - 资源服务器只需公钥验证

移动/IoT 优化：ES256 (ECDSA + P-256)
  - 签名体积小（节省带宽）
  - 签名验证速度快
  - 移动设备友好

企业高性能：EdDSA (Ed25519)（JWT 库逐步支持）
  - 最快且最安全
  - 128-bit 安全强度

避免：
  - HS256：需要共享密钥，不适合 OAuth2 场景
  - none：禁止生产使用
  - RS256 < 2048-bit 密钥长度：不够安全
```

## 2.3 "none" 算法攻击与防御

```
经典攻击场景（CVE-2015-2951）：

攻击者构造 JWT：
Header: {"alg": "none", "typ": "JWT"}
Payload: {"sub": "admin", "role": "admin"}
Signature: ""（空）

如果服务器端的 JWT 库不做算法白名单校验，
直接调用 alg=none 的验证器，验证 "通过"，
→ 攻击者获得管理员权限

防御代码示例：
```

```java
public class SafeJwtVerifier {
    
    private static final Set<String> ALLOWED_ALGORITHMS = Set.of(
        "RS256", "RS384", "RS512",
        "ES256", "ES384", "ES512",
        "PS256", "PS384", "PS512"
    );
    
    public Claims verify(String token) {
        SignedJWT jwt = SignedJWT.parse(token);
        JWSAlgorithm algorithm = jwt.getHeader().getAlgorithm();
        
        // 1. 算法白名单验证（关键！）
        if (!ALLOWED_ALGORITHMS.contains(algorithm.getName())) {
            throw new SecurityException("Unsupported algorithm: " + algorithm);
        }
        
        // 2. 拒绝对称算法（当使用非对称密钥时）
        if (algorithm == JWSAlgorithm.HS256 
            || algorithm == JWSAlgorithm.HS384 
            || algorithm == JWSAlgorithm.HS512) {
            throw new SecurityException("Symmetric algorithms not allowed");
        }
        
        // 3. 获取正确算法的公钥
        JWSVerifier verifier = getVerifierByAlgorithm(algorithm, jwt.getHeader().getKeyID());
        
        // 4. 验证签名
        if (!jwt.verify(verifier)) {
            throw new SecurityException("JWT signature verification failed");
        }
        
        return jwt.getJWTClaimsSet();
    }
}
```

---

# 3. JWT Claims 深入

## 3.1 Public Claims（公共声明）

JWT 规范注册了以下 IANA Claims（公开可用）：

```
https://www.iana.org/assignments/jwt/jwt.xhtml

部分常见 Public Claims：
  name, given_name, family_name, middle_name, nickname
  preferred_username, profile, picture, website
  email, email_verified
  gender, birthdate, zoneinfo, locale
  phone_number, phone_number_verified
  address（JSON 对象）
  updated_at
```

## 3.2 Private Claims（私有声明）

Private Claims 是自定义声明，需要双方协商以避免冲突：

```
命名规范：
- 使用命名空间：https://mycompany.com/claims/department
- 或简短但有意义的前缀：dept_code、org_unit
- 避免与 IANA 注册的公共声明冲突
- 文档化所有 Private Claims 的含义

示例：
{
  "sub": "user-123",
  "name": "Alice",
  "https://mycompany.com/claims/department": "engineering",
  "https://mycompany.com/claims/cost-center": "CC-456"
}
```

## 3.3 JWT Claims 的大小优化

```
JWT 大小优化策略：

1. 使用短键名（协商后的场景）
   sub 而不是 subject
   jti 而不是 jwt_id
   自定义：dpt 而不是 https://mycompany.com/claims/department

2. 避免冗余数据
   ID Token：仅用户身份（sub, name, email）
   UserInfo 端点：完整资料
   不要把所有用户信息都塞进 JWT

3. 注意序列化开销
   每个 Claim 的 key 都会增加 JWT 体积
   如果 JWT 放在 HTTP Header，建议控制在 8KB 以内

4. 压缩（可选）
   JWE 支持压缩（Deflate），但增加了处理复杂度
   一般不推荐在 OAuth2 场景中使用 JWE 压缩
```

---

# 4. JWT 生命周期

## 4.1 时间相关 Claims 的含义

```
时间线示意：

  iat                nbf               exp
  │                   │                  │
  ├───────────────────┼──────────────────┤
  │  JWT 已创建        │  JWT 开始有效     │  JWT 失效
  │                   │                  │
  ←─ 不可用 ──────────→←── 可用 ──────────→←─ 不可用 ─

时钟偏差（Clock Skew）处理：
  - 服务器时钟可能存在偏差（通常 ± 5 分钟）
  - 验证时允许适度偏差
  - 使用 NTP 同步所有服务器时钟
```

## 4.2 JWT 的撤销问题

```
JWT 的根本矛盾：自包含 vs 可撤销

JWT 设计要求自包含（无需查询数据库）
但撤销需要外部状态（黑名单/白名单）

撤销策略：

1. 短有效期
   Access Token 15 分钟 → 最长 15 分钟后自动失效
   适用于大多数场景

2. 维护黑名单（JWT ID）
   Redis key: revoked:{jti}  TTL: 到 Token 原始过期时间
   验证时先检查黑名单
   需要额外的存储和网络调用

3. 维护版本号（Claim version）
   JWT 中包含用户 token_version
   用户修改密码后 token_version++
   验证时检查 Token 中的 version 与数据库中的 version 是否一致

4. 签名密钥轮换
   紧急情况：轮换签名密钥，所有旧 Token 立即失效
   需要等待客户端刷新 Token

最佳实践：
  方案一 + 方案三 组合：
  - Access Token 15 分钟有效期
  - 用户修改密码/注销时 token_version++
```

---

# 5. JWE（JSON Web Encryption）

JWE 提供 JWT 的加密层，将 Claims 加密而非仅签名。

```
当需要使用 JWE：
- Token 中包含敏感信息（身份证号、银行账号）
- 传输通道不完全可信（即使有 TLS 也需要纵深防御）
- 监管要求加密传输

JWE 的代价：
- 更大的 Token 体积（加密比签名大得多）
- 更高的 CPU 开销
- 更复杂的密钥管理

OAuth2 场景建议：
- 绝大多数场景：HTTPS + JWS（签名）足够
- 敏感场景：使用 JWE 加密 ID Token
- 尽量避免在 Access Token 中放敏感信息
```

---

# 6. JWT 与 Session 对比

| 维度 | JWT | Session |
|------|-----|---------|
| 存储位置 | 客户端（HTTP Header） | 服务器（内存/Redis/DB） |
| 扩展性 | 水平扩展友好（无状态） | 需要共享 Session 存储 |
| 撤销能力 | 弱（依赖过期或黑名单） | 强（删除即可） |
| 体积开销 | Token 本身 1-8 KB | Cookie 仅 Session ID（~30 bytes） |
| 信息携带 | 自包含 Claims | 服务器维护 |
| 跨域 | 天然支持 | 需要 CORS + 共享存储 |
| 移动端 | 友好 | 需要 Cookie 支持 |

```
选择建议：
- 微服务 + API Gateway → JWT
- 传统 MVC Web 应用 → Session
- 移动 App → JWT
- 需要即时撤销权限 → Session（或 JWT + 黑名单）
```

---

# 7. JWT Java 实践库

```java
// Nimbus JOSE + JWT（最常用的 Java JWT 库）
// Spring Security 默认使用 Nimbus

// 生成 RSA 签名的 JWT
RSAKey rsaJWK = new RSAKeyGenerator(2048)
    .keyID("my-key-id")
    .generate();

// Header
JWSHeader header = new JWSHeader.Builder(JWSAlgorithm.RS256)
    .keyID(rsaJWK.getKeyID())
    .type(JOSEObjectType.JWT)
    .build();

// Payload
JWTClaimsSet claims = new JWTClaimsSet.Builder()
    .issuer("https://auth.example.com")
    .subject("user-123")
    .audience("resource-server-1")
    .expirationTime(Date.from(Instant.now().plusSeconds(900)))
    .issueTime(new Date())
    .jwtID(UUID.randomUUID().toString())
    .claim("scope", "read write")
    .build();

// 签名
SignedJWT signedJWT = new SignedJWT(header, claims);
signedJWT.sign(new RSASSASigner(rsaJWK));
String token = signedJWT.serialize();
```

```xml
<!-- Maven 依赖 -->
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
    <version>9.37.3</version>
</dependency>
```
