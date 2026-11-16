# 1. Java JWT 库全景

Java 生态中有多个 JWT 库，它们在功能完整性、API 设计、性能、安全性和社区活跃度方面各有特点。选择合适的库是 JWT 正确实施的第一步。

## 1.1 主流库速览

| 库 | 组织 | 最新版本（参考） | GitHub Stars | License |
|----|------|-----------------|-------------|---------|
| **Nimbus JOSE+JWT** | Connect2id | 9.x | 1.5k+ | Apache 2.0 |
| **JJWT** | Okta | 0.12.x | 10k+ | Apache 2.0 |
| **jose4j** | Brian Campbell | 0.9.x | 600+ | Apache 2.0 |
| **Auth0 JWT** | Auth0 | 4.x | 5k+ | MIT |
| **Vert.x JWT** | Eclipse | 4.x | 集成在 Vert.x 中 | Apache 2.0 |

---

# 2. Nimbus JOSE+JWT — 功能最完整的库

## 2.1 概述

Nimbus JOSE+JWT 是 Java 生态中 JOSE（Javascript Object Signing and Encryption）规范实现最完整的库。它被 Spring Security 5+ 的 OAuth2 模块作为底层依赖使用。

## 2.2 核心能力矩阵

| 能力 | 支持状态 |
|------|----------|
| JWS（签名） | 完全支持 |
| JWE（加密） | 完全支持 |
| JWK（密钥表示） | 完全支持 |
| JWA（算法） | 所有标准算法 |
| JWT Claims | 完全支持 |
| OAuth 2.0 Token Introspection | 支持 |
| OpenID Connect | 原生支持 |
| 嵌套 JWT | 支持 |
| JSON 序列化 | 支持 |

## 2.3 基本使用

### 2.3.1 Maven 依赖

```xml
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
    <version>9.37.3</version>
</dependency>
```

### 2.3.2 签发 HS256 JWT

```java
import com.nimbusds.jose.*;
import com.nimbusds.jose.crypto.*;
import com.nimbusds.jwt.*;

// 1. 准备密钥（至少 256 位）
SecureRandom random = new SecureRandom();
byte[] sharedSecret = new byte[32];
random.nextBytes(sharedSecret);

// 2. 创建 JWS Header
JWSHeader header = new JWSHeader.Builder(JWSAlgorithm.HS256)
    .type(JOSEObjectType.JWT)
    .keyID("key-2025-01")       // 可选：密钥 ID
    .build();

// 3. 构建 Claims
JWTClaimsSet claimsSet = new JWTClaimsSet.Builder()
    .issuer("https://auth.example.com")
    .subject("user123")
    .audience("api.example.com")
    .expirationTime(new Date(System.currentTimeMillis() + 3600 * 1000))
    .issueTime(new Date())
    .notBeforeTime(new Date())
    .jwtID(UUID.randomUUID().toString())
    .claim("roles", List.of("ROLE_USER", "ROLE_ADMIN"))
    .claim("email", "user@example.com")
    .build();

// 4. 创建 SignedJWT
SignedJWT signedJWT = new SignedJWT(header, claimsSet);

// 5. 签名
JWSSigner signer = new MACSigner(sharedSecret);
signedJWT.sign(signer);

// 6. 序列化
String jwt = signedJWT.serialize();
// eyJraWQiOiJrZXktMjAyNS0wMSIsImFsZyI6IkhTMjU2IiwidHlwIjoiSldUIn0...
```

### 2.3.3 验证 JWT

```java
// 1. 解析 JWT
SignedJWT parsedJWT = SignedJWT.parse(jwt);

// 2. 创建验证器（绑定密钥，自动限制算法）
JWSVerifier verifier = new MACVerifier(sharedSecret);

// 3. 验证签名
if (!parsedJWT.verify(verifier)) {
    throw new JOSEException("Signature verification failed");
}

// 4. 提取并验证 Claims
JWTClaimsSet claims = parsedJWT.getJWTClaimsSet();

// 5. 验证过期
Date expiration = claims.getExpirationTime();
if (expiration != null && expiration.before(new Date())) {
    throw new JOSEException("Token expired");
}

// 6. 提取自定义 Claims
String email = claims.getStringClaim("email");
List<String> roles = claims.getStringListClaim("roles");
String subject = claims.getSubject();
```

### 2.3.4 签发 RS256 JWT

```java
// 1. 加载 RSA 密钥
KeyStore keyStore = KeyStore.getInstance("JKS");
keyStore.load(new FileInputStream("keystore.jks"), "password".toCharArray());
RSAPrivateKey privateKey = (RSAPrivateKey) keyStore.getKey("alias", "password".toCharArray());

// 2. 创建 Header
JWSHeader header = new JWSHeader.Builder(JWSAlgorithm.RS256)
    .type(JOSEObjectType.JWT)
    .build();

// 3. 签名
SignedJWT signedJWT = new SignedJWT(header, claimsSet);
JWSSigner signer = new RSASSASigner(privateKey);
signedJWT.sign(signer);

String jwt = signedJWT.serialize();
```

### 2.3.5 使用 JWK 验证

```java
// 1. 从 JWK 解析公钥
String jwkJson = """
{
  "kty": "RSA",
  "e": "AQAB",
  "use": "sig",
  "kid": "key-2025-01",
  "alg": "RS256",
  "n": "oahUIoWw0K0usKNuOR6H4wkf4oBUXHTxRvgb..."
}
""";

RSAKey rsaKey = RSAKey.parse(jwkJson);

// 2. 根据 kid 选择 JWK
SignedJWT parsedJWT = SignedJWT.parse(jwt);
String kid = parsedJWT.getHeader().getKeyID();

// 3. 加载对应的公钥并验证
RSAPublicKey publicKey = rsaKey.toRSAPublicKey();
JWSVerifier verifier = new RSASSAVerifier(publicKey);

if (!parsedJWT.verify(verifier)) {
    throw new JOSEException("Invalid signature");
}
```

### 2.3.6 JWE 加密

```java
// 创建 JWE Header
JWEHeader header = new JWEHeader.Builder(JWEAlgorithm.RSA_OAEP_256, EncryptionMethod.A256GCM)
    .contentType("JWT")
    .build();

// 创建 JWE Object
JWEObject jweObject = new JWEObject(header, new Payload(signedJWT));

// 加密
RSAEncrypter encrypter = new RSAEncrypter(publicKey);
jweObject.encrypt(encrypter);

String encryptedToken = jweObject.serialize();
// 五个部分：header.encryptedKey.iv.ciphertext.tag

// 解密
JWEObject parsedJWE = JWEObject.parse(encryptedToken);
RSADecrypter decrypter = new RSADecrypter(privateKey);
parsedJWE.decrypt(decrypter);

// 获取内层 JWT
SignedJWT decryptedJWT = parsedJWE.getPayload().toSignedJWT();
```

## 2.4 Nimbus 高级特性

### 2.4.1 JWT 断言验证

```java
// 使用 JWTAssertionValidator 进行复杂声明验证
DefaultJWTClaimsVerifier<?> verifier = new DefaultJWTClaimsVerifier<>(
    new JWTClaimsSet.Builder()
        .issuer("https://auth.example.com")
        .build(),
    Set.of("sub", "iat", "exp", "jti")  // 必需的声明
);

verifier.verify(claims, null);  // 不匹配时抛出 BadJWTException
```

### 2.4.2 JWK Set 远程加载

```java
// 从 OAuth2 授权服务器加载 JWK Set
URL jwkSetURL = new URL("https://auth.example.com/.well-known/jwks.json");
JWKSource<SecurityContext> keySource = new RemoteJWKSet<>(jwkSetURL);

// 验证时自动选择正确的 JWK
JWTClaimsSet claims = SignedJWT.parse(jwt).verify(
    // 根据 kid 自动选择 JWK
    new DefaultJWTClaimsVerifier<>()
);
```

---

# 3. JJWT — 最流行的 Java JWT 库

## 3.1 概述

JJWT 由 Okta 维护，以流式 Builder/Fluent API 著称，代码可读性强，社区活跃度高。是 Spring Boot 项目中最常用的 JWT 库之一。

## 3.2 基本使用

### 3.2.1 Maven 依赖

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

> 注意：0.12.x 版本包名从 `io.jsonwebtoken` 保持不变，但 API 有较大调整，内置算法常量变更。

### 3.2.2 签发 HS256 JWT（0.12.x API）

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;

// 生成安全密钥（HS256 ≥ 256 位）
SecretKey key = Jwts.SIG.HS256.key().build();

// 签发 JWT
String jwt = Jwts.builder()
    .issuer("https://auth.example.com")
    .subject("user123")
    .audience().add("api.example.com").and()
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 3600000))
    .id(UUID.randomUUID().toString())
    .claim("roles", List.of("ROLE_USER", "ROLE_ADMIN"))
    .claim("email", "user@example.com")
    .signWith(key)                  // 算法由 key 类型自动推断
    .compact();
```

### 3.2.3 验证 JWT（0.12.x API）

```java
// 解析 JWT
Jws<Claims> jws = Jwts.parser()
    .verifyWith(key)
    .requireIssuer("https://auth.example.com")
    .requireAudience("api.example.com")
    .clockSkewSeconds(30)
    .build()
    .parseSignedClaims(jwt);

Claims claims = jws.getPayload();

// 提取 Claims
String subject = claims.getSubject();
String email = claims.get("email", String.class);
List<String> roles = claims.get("roles", List.class);
```

### 3.2.4 签发 RS256 JWT

```java
import java.security.*;

// 加载密钥对
KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
keyGen.initialize(2048);
KeyPair keyPair = keyGen.generateKeyPair();

// 签发
String jwt = Jwts.builder()
    .subject("user123")
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 3600000))
    .signWith(keyPair.getPrivate())  // 使用私钥签名
    .compact();

// 验证
Jws<Claims> jws = Jwts.parser()
    .verifyWith((PublicKey) keyPair.getPublic())  // 使用公钥验证
    .build()
    .parseSignedClaims(jwt);
```

### 3.2.5 0.11.x API（旧版本兼容说明）

```java
// 0.11.x API（与 0.12.x 不同的签名方式）
String jwt = Jwts.builder()
    .setSubject("user123")
    .setExpiration(new Date(System.currentTimeMillis() + 3600000))
    .signWith(key, SignatureAlgorithm.HS256)  // 显式指定算法
    .compact();

// 解析
Jws<Claims> jws = Jwts.parserBuilder()
    .setSigningKey(key)
    .build()
    .parseClaimsJws(jwt);
```

### 3.2.6 JJWT 多密钥支持

```java
// 使用 SigningKeyResolver 支持多个密钥
SigningKeyResolver keyResolver = new SigningKeyResolverAdapter() {
    @Override
    public Key resolveSigningKey(JwsHeader header, Claims claims) {
        String kid = header.getKeyId();
        return keyRepository.getKeyById(kid);
    }
};

Jws<Claims> jws = Jwts.parserBuilder()
    .setSigningKeyResolver(keyResolver)
    .build()
    .parseClaimsJws(jwt);
```

---

# 4. jose4j — 企业级 JOSE 库

## 4.1 概述

jose4j 由 Brian Campbell（Ping Identity 工程师）开发，功能全面且经过大量生产环境验证。

### 4.1.1 Maven 依赖

```xml
<dependency>
    <groupId>org.bitbucket.b_c</groupId>
    <artifactId>jose4j</artifactId>
    <version>0.9.4</version>
</dependency>
```

### 4.1.2 基本使用

```java
import org.jose4j.jws.*;
import org.jose4j.jwt.*;
import org.jose4j.keys.*;
import org.jose4j.lang.*;

// 签发
JsonWebSignature jws = new JsonWebSignature();
jws.setAlgorithmHeaderValue(AlgorithmIdentifiers.HMAC_SHA256);
jws.setKey(new HmacKey(sharedSecret));
jws.setPayload("{" +
    "\"sub\":\"user123\"," +
    "\"exp\":" + (System.currentTimeMillis() / 1000 + 3600) +
    "}");
jws.setHeader("typ", "JWT");

String jwt = jws.getCompactSerialization();

// 验证
JsonWebSignature verifierJws = new JsonWebSignature();
verifierJws.setCompactSerialization(jwt);
verifierJws.setKey(new HmacKey(sharedSecret));

if (!verifierJws.verifySignature()) {
    throw new JoseException("Invalid signature");
}

JwtClaims claims = JwtClaims.parse(verifierJws.getPayload());
```

---

# 5. Auth0 JWT — 轻量快速

## 5.1 概述

简洁的 API，适合简单场景。不支持 JWE/JWK。

### 5.1.1 Maven 依赖

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>4.4.0</version>
</dependency>
```

### 5.1.2 基本使用

```java
import com.auth0.jwt.*;
import com.auth0.jwt.algorithms.Algorithm;

// 签发
Algorithm algorithm = Algorithm.HMAC256("secret-must-be-256-bits-or-longer");
String token = JWT.create()
    .withIssuer("auth.example.com")
    .withSubject("user123")
    .withClaim("email", "user@example.com")
    .withExpiresAt(new Date(System.currentTimeMillis() + 3600000))
    .withJWTId(UUID.randomUUID().toString())
    .sign(algorithm);

// 验证
DecodedJWT jwt = JWT.require(algorithm)
    .withIssuer("auth.example.com")
    .build()
    .verify(token);

String subject = jwt.getSubject();
String email = jwt.getClaim("email").asString();
```

---

# 6. 库对比与选型指南

## 6.1 功能对比矩阵

| 功能 | Nimbus | JJWT | jose4j | Auth0 |
|------|--------|------|--------|-------|
| JWS（签名） | ✅ | ✅ | ✅ | ✅ |
| JWE（加密） | ✅ | ✅ (0.12+) | ✅ | ❌ |
| JWK | ✅ | ✅ (0.12+) | ✅ | ❌ |
| JWA 全部算法 | ✅ | ✅ | ✅ | ✅ |
| JWK Set 远程加载 | ✅ | ❌ | ✅ | ❌ |
| OAuth2/OIDC | ✅ | ❌ | ✅ | ❌ |
| Fluent API | ⚠️ | ✅ | ❌ | ✅ |
| 强制 Claims 验证 | ⚠️ | ✅ | ⚠️ | ✅ |
| 自动算法检测防御 | ✅ | ✅ (key 绑定) | ⚠️ | ⚠️ |
| Spring Security 集成 | ✅ (内置) | 需手动 | 需手动 | 需手动 |
| JSON 序列化 | ✅ | ❌ | ✅ | ❌ |
| 压缩序列化 | ✅ | ✅ | ✅ | ✅ |

## 6.2 选型建议

```java
// OAuth2 / OpenID Connect 项目
推荐：Nimbus JOSE+JWT（Spring Security OAuth2 底层已集成）

// Spring Boot Web 应用（常规 JWT 认证）
推荐：JJWT（API 优雅、社区活跃、文档丰富）

// 需要 JWE 加密 + 非 OAuth2 场景
推荐：Nimbus JOSE+JWT

// 简单 API 签名验证
推荐：JJWT 或 Auth0 JWT

// 金融 / 政府等合规场景
推荐：jose4j 或 Nimbus（功能最完整、最稳定）

// 微服务内部门户
推荐：JJWT（简单高效）
```

## 6.3 性能对比（参考数据）

| 操作（HS256） | Nimbus | JJWT | Auth0 |
|---------------|--------|------|-------|
| 签发 100k 次 | ~800ms | ~600ms | ~500ms |
| 验证 100k 次 | ~700ms | ~500ms | ~450ms |
| JAR 大小 | ~500KB | ~700KB | ~70KB |

> 实际性能取决于 JVM 版本、硬件和具体使用方式。选择库时功能完整性和安全性应优先于微小性能差异。

---

# 7. 安全使用指南

## 7.1 公共安全要求

无论选择哪个库，必须遵守：

```java
// 1. 绝不接受 alg=none
// 2. 绝不硬编码密钥
// 3. 使用足够强度的密钥
// 4. 验证所有标准 Claims（iss, aud, exp, nbf）
// 5. 使用不同密钥实例做到算法隔离

// JJWT 示例：算法隔离
public class AlgorithmIsolatedParser {
    private final JwtParser hs256Parser;
    private final JwtParser rs256Parser;
    
    public AlgorithmIsolatedParser(SecretKey hsKey, PublicKey rsKey) {
        this.hs256Parser = Jwts.parser().verifyWith(hsKey).build();
        this.rs256Parser = Jwts.parser().verifyWith(rsKey).build();
    }
    
    public Claims parse(String token) {
        String alg = extractAlgorithm(token);
        return switch (alg) {
            case "HS256" -> hs256Parser.parseSignedClaims(token).getPayload();
            case "RS256" -> rs256Parser.parseSignedClaims(token).getPayload();
            default -> throw new JwtException("Unsupported algorithm: " + alg);
        };
    }
}
```

## 7.2 库版本管理

```xml
<!-- 建议使用 Maven Enforcer 或 Dependabot 保持库版本最新 -->

<!-- 已知安全漏洞版本（仅供参考，具体以 CVE 为准）：
  Nimbus < 9.x：不安全的默认配置
  JJWT < 0.12.0：API 混淆，算法混淆攻击风险
  Auth0 < 3.18：弱密钥检测不充分
-->
```
