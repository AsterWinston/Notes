# 1. JOSE 规范族概述

JOSE（JSON Object Signing and Encryption）是一组关于 JSON 格式安全传输的 RFC 规范族，JWT 只是其中一个组成部分。

## 1.1 规范关系

```
┌────────────────────────────────────────────────────┐
│                      JWT (RFC 7519)                │
│              JSON Web Token 结构定义                │
├──────────────────┬─────────────────────────────────┤
│                  │                                 │
│   JWS (RFC 7515) │    JWE (RFC 7516)              │
│   JSON Web       │    JSON Web Encryption          │
│   Signature      │    加密 → 机密性                │
│   签名 → 完整性  │                                 │
│                  │                                 │
├──────────────────┴─────────────────────────────────┤
│          JWA (RFC 7518) - 算法注册表                │
│  定义 HS256, RS256, ES256, RSA-OAEP, A256GCM 等    │
├────────────────────────────────────────────────────┤
│          JWK (RFC 7517) - 密钥表示                  │
│  以 JSON 格式表示公钥/私钥/对称密钥                 │
└────────────────────────────────────────────────────┘
```

---

# 2. JWK（JSON Web Key）— RFC 7517

## 2.1 什么是 JWK

JWK 是一种以 JSON 格式表示加密密钥的标准。它将密钥的各个参数编码为 JSON 对象，便于在系统间传输和存储。

## 2.2 JWK 的通用成员

| 成员 | 必需 | 说明 |
|------|------|------|
| `kty` | 是 | Key Type：密钥类型标识 |
| `use` | 否 | Public Key Use：`sig`（签名）或 `enc`（加密） |
| `key_ops` | 否 | Key Operations：`sign`, `verify`, `encrypt`, `decrypt` 等 |
| `alg` | 否 | Algorithm：此密钥适用的算法 |
| `kid` | 否 | Key ID：密钥唯一标识符 |
| `x5u` | 否 | X.509 URL |
| `x5c` | 否 | X.509 Certificate Chain |
| `x5t` | 否 | X.509 Certificate SHA-1 Thumbprint |
| `x5t#S256` | 否 | X.509 Certificate SHA-256 Thumbprint |

## 2.3 各类型 JWK 示例

### 2.3.1 对称密钥（oct — Octet Sequence）

```json
{
  "kty": "oct",
  "k": "AyM1SysPpbyDfgZld3umj1qzKObwVMkoqQ-EstJQLr_T-1qS0gZH75aKtMN3Yj0iPS4hcgUuTwjAzZr1Z9CAow",
  "kid": "hmac-key-001",
  "alg": "HS256",
  "use": "sig"
}
```

- `k`：Base64URL 编码的密钥值

### 2.3.2 RSA 公钥

```json
{
  "kty": "RSA",
  "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknjhMstn64tZ_2W-5JsGY4Hc5n9yBXArwl93lqt7_RN5w6Cf0h4QyQ5v-65YGjQR0_FDW2QvzqY368QQMicAtaSqzs8KJZgnYb9c7d0zgdAZHzu6qMQvRL5hajrn1n91CbOpbISD08qNLyrdkt-bFTWhAI4vMQFh6WeZu0fM4lFd2NcRwr3XPksINHaQ-G_xBniIqbw0Ls1jF44-csFCur-kEgU8awapJzKnqDKgw",
  "e": "AQAB",
  "alg": "RS256",
  "kid": "rsa-key-001",
  "use": "sig"
}
```

| 参数 | 说明 |
|------|------|
| `n` | RSA 模数（modulus），Base64URL 编码 |
| `e` | RSA 公共指数（exponent），Base64URL 编码 |
| `d` | RSA 私钥指数（仅私钥） |
| `p`, `q`, `dp`, `dq`, `qi` | RSA 私钥 CRT 参数（仅私钥） |

### 2.3.3 EC 公钥（P-256）

```json
{
  "kty": "EC",
  "crv": "P-256",
  "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
  "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0",
  "kid": "ec-key-001",
  "alg": "ES256"
}
```

| 参数 | 说明 |
|------|------|
| `crv` | 曲线名称：P-256, P-384, P-521 |
| `x` | X 坐标，Base64URL 编码 |
| `y` | Y 坐标，Base64URL 编码 |
| `d` | 私钥（仅私钥） |

### 2.3.4 OKP 公钥（Ed25519）

```json
{
  "kty": "OKP",
  "crv": "Ed25519",
  "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
  "kid": "ed25519-key-001",
  "alg": "EdDSA"
}
```

## 2.4 JWK Set（JWKS）

JWK Set 是包含多个 JWK 的 JSON 数组，用于密钥分发：

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "rsa-key-2025-01",
      "use": "sig",
      "alg": "RS256",
      "n": "...",
      "e": "AQAB"
    },
    {
      "kty": "EC",
      "kid": "ec-key-2025-04",
      "use": "sig",
      "alg": "ES256",
      "crv": "P-256",
      "x": "...",
      "y": "..."
    },
    {
      "kty": "OKP",
      "kid": "ed25519-key-2025-07",
      "use": "sig",
      "alg": "EdDSA",
      "crv": "Ed25519",
      "x": "..."
    }
  ]
}
```

## 2.5 JWK 在 Java 中的处理

### 2.5.1 Nimbus 库

```java
import com.nimbusds.jose.jwk.*;
import com.nimbusds.jose.jwk.gen.*;

// 生成 RSA JWK
RSAKey rsaJwk = new RSAKeyGenerator(2048)
    .keyID("rsa-key-001")
    .algorithm(JWSAlgorithm.RS256)
    .keyUse(KeyUse.SIGNATURE)
    .generate();

// 导出公钥 JWK
RSAKey publicJwk = rsaJwk.toPublicJWK();
String jwkJson = publicJwk.toJSONString();

// 从 JSON 解析
RSAKey parsedJwk = RSAKey.parse(jwkJson);

// 转换为 Java 标准 Key 对象
RSAPublicKey publicKey = parsedJwk.toRSAPublicKey();
RSAPrivateKey privateKey = parsedJwk.toRSAPrivateKey();

// 生成 EC JWK
ECKey ecJwk = new ECKeyGenerator(Curve.P_256)
    .keyID("ec-key-001")
    .algorithm(JWSAlgorithm.ES256)
    .generate();

// 生成 HMAC JWK
OctetSequenceKey octJwk = new OctetSequenceKeyGenerator(256)
    .keyID("hmac-key-001")
    .algorithm(JWSAlgorithm.HS256)
    .generate();
```

### 2.5.2 构建 JWK Set 端点

```java
@RestController
public class JwkSetController {
    
    private final JWKSet jwkSet;
    
    public JwkSetController(List<RSAKey> signingKeys) {
        this.jwkSet = new JWKSet(signingKeys);
    }
    
    @GetMapping("/.well-known/jwks.json")
    public String getJwkSet() {
        return jwkSet.toString();  // 自动序列化为 JSON
    }
}
```

---

# 3. JWS（JSON Web Signature）— RFC 7515

## 3.1 JWS 概述

JWS 定义了如何对 JSON 数据进行数字签名，确保数据的**完整性**和**不可抵赖性**。

## 3.2 JWS 的两种序列化形式

### 3.2.1 紧凑序列化（JWS Compact Serialization）

JWT 最常用的形式，三个 Base64URL 编码段以 `.` 连接：

```
BASE64URL(UTF8(JWS Protected Header)) .
BASE64URL(JWS Payload) .
BASE64URL(JWS Signature)
```

### 3.2.2 JWS 签名计算过程

```
JWS Signing Input = ASCII(BASE64URL(UTF8(JWS Protected Header)) || '.' || BASE64URL(JWS Payload))

JWS Signature = Sign(JWS Signing Input, Private Key, Algorithm)
```

详细过程：

```java
// 伪代码：JWS 签名
public String sign(Payload payload, PrivateKey key, Algorithm alg) {
    // 1. 序列化 Protected Header
    String headerJson = "{\"alg\":\"RS256\"}";
    String headerB64 = BASE64URL(headerJson);
    
    // 2. 序列化 Payload（即使 Payload 本身是 Base64URL 编码的 JWT Claims）
    String payloadB64 = payload.toBase64URL();
    
    // 3. 构建签名输入
    String signingInput = headerB64 + "." + payloadB64;
    
    // 4. 计算签名
    Signature signer = Signature.getInstance("SHA256withRSA");
    signer.initSign(key);
    signer.update(signingInput.getBytes(StandardCharsets.US_ASCII));
    byte[] signature = signer.sign();
    
    // 5. Base64URL 编码签名
    String signatureB64 = BASE64URL(signature);
    
    // 6. 拼接最终结果
    return headerB64 + "." + payloadB64 + "." + signatureB64;
}
```

### 3.2.3 JSON 序列化（JWS JSON Serialization）

支持多个签名（多方签署）：

**通用 JSON 序列化：**

```json
{
  "payload": "eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ",
  "signatures": [
    {
      "protected": "eyJhbGciOiJSUzI1NiJ9",
      "header": {"kid": "2010-12-29"},
      "signature": "cC4hiUPoj9Eetdgtv3hF80EGrhuB__dzERat0XF9g2VtQgr9PJbu3XOiZj5RZmh7AAuHIm4Bh-0Qc_lF5YKt_O8W2Fp5jujGbds9uJdbF9CUAr7t1dnZcAcQjbKBYNX4BAynRFdiuB--f_nZLgrnbyTyRRjzOszByjQvCO3UnpP0wQ"
    },
    {
      "protected": "eyJhbGciOiJFUzI1NiJ9",
      "header": {"kid": "e9bc097a-ce51-4036-9562-d2ade882db0d"},
      "signature": "DtEhU3ljbEg8L38VWAfUAqOyKAM6-Xx-F4GawxaepmXFCgfTjDxw5djxLa8ISlSApmWQxfKTUJqPP3-Kg6NU1Q"
    }
  ]
}
```

**扁平 JSON 序列化（单个签名）：**

```json
{
  "payload": "eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ",
  "protected": "eyJhbGciOiJFUzI1NiJ9",
  "header": {"kid": "e9bc097a-ce51-4036-9562-d2ade882db0d"},
  "signature": "DtEhU3ljbEg8L38VWAfUAqOyKAM6-Xx-F4GawxaepmXFCgfTjDxw5djxLa8ISlSApmWQxfKTUJqPP3-Kg6NU1Q"
}
```

## 3.3 JWS Header 参数

### 3.3.1 与签名相关的参数

| 参数 | 说明 |
|------|------|
| `alg` | **必需**。签名算法（如 RS256, ES256, HS256） |
| `jku` | JWK Set URL |
| `jwk` | 嵌入的 JWK 公钥 |
| `kid` | 密钥标识符 |
| `x5u` | X.509 URL |
| `x5c` | X.509 证书链 |
| `x5t` | X.509 证书 SHA-1 指纹 |
| `x5t#S256` | X.509 证书 SHA-256 指纹 |
| `crit` | 关键 Header 参数列表 |

### 3.3.2 crit 参数

```json
{
  "alg": "ES256",
  "crit": ["exp", "http://example.com/extension"],
  "exp": 1712345600,
  "http://example.com/extension": {
    "custom": "data"
  }
}
```

如果 `crit` 列表中有未被接收方理解的参数，接收方**必须拒绝**该 JWS。

---

# 4. JWE（JSON Web Encryption）— RFC 7516

## 4.1 JWE 概述

JWE 提供数据的**机密性**保护。与 JWS 不同，JWE 的 Payload 被完全加密，传输过程中不可见。

JWE 紧凑序列化有 5 个部分：

```
BASE64URL(JWE Protected Header) .
BASE64URL(JWE Encrypted Key) .
BASE64URL(JWE Initialization Vector) .
BASE64URL(JWE Ciphertext) .
BASE64URL(JWE Authentication Tag)
```

## 4.2 JWE 加密过程

```
步骤1：确定加密算法和密钥加密算法
  alg: RSA-OAEP（非对称加密 Content Encryption Key）
  enc: A256GCM（对称加密 Payload）

步骤2：生成随机 CEK（Content Encryption Key）
  生成 256 位随机密钥

步骤3：加密 CEK
  encryptedKey = RSA-OAEP-Encrypt(CEK, recipient's public key)

步骤4：生成随机 IV（Initialization Vector）
  生成 96 位随机 IV

步骤5：加密 Payload
  ciphertext + authTag = AES-256-GCM-Encrypt(
      plaintext=payload,
      key=CEK,
      iv=IV,
      aad=base64(protectedHeader)  // 附加认证数据
  )

步骤6：拼接 5 部分
  header.encryptedKey.IV.ciphertext.authTag
```

### 4.2.1 为什么使用两层加密

```
Payload 使用对称算法加密（AES-GCM） → 高效加密大数据
对称密钥使用非对称算法加密（RSA-OAEP） → 安全传递密钥给接收方
```

## 4.3 JWE 解密过程

```java
// Nimbus 库 JWE 解密
public SignedJWT decrypt(String encryptedToken, RSAPrivateKey privateKey) 
        throws Exception {
    
    // 1. 解析 JWE
    JWEObject jweObject = JWEObject.parse(encryptedToken);
    
    // 2. 解密
    RSADecrypter decrypter = new RSADecrypter(privateKey);
    jweObject.decrypt(decrypter);
    
    // 3. 获取内层 Payload
    Payload payload = jweObject.getPayload();
    
    // 4. 如果 Payload 是 JWS
    return payload.toSignedJWT();
}
```

## 4.4 JWE 算法组合

| 密钥加密算法 (alg) | 内容加密算法 (enc) | 说明 |
|-------------------|-------------------|------|
| `RSA-OAEP` | `A128GCM` | RSA 加密 CEK，128 位 AES-GCM 加密内容 |
| `RSA-OAEP-256` | `A256GCM` | RSA 加密 CEK，256 位 AES-GCM 加密内容 |
| `ECDH-ES` | `A128GCM` | ECDH 密钥协商 + 128 位 AES-GCM |
| `ECDH-ES+A128KW` | `A256GCM` | ECDH 密钥协商 + AES 密钥包装 + 256 位 AES-GCM |
| `A128KW` | `A128GCM` | AES 密钥包装（对称） |
| `PBES2-HS256+A128KW` | `A256GCM` | 基于密码的加密 |
| `dir` | `A256GCM` | 直接使用 CEK 作为对称密钥（不包装） |

## 4.5 JWE Header 参数

| 参数 | 说明 |
|------|------|
| `alg` | 密钥加密算法 |
| `enc` | 内容加密算法 |
| `zip` | 压缩算法（`DEF`） |
| `jku`, `jwk`, `kid` | 密钥引用（与 JWS 相同） |
| `apu` | 协议方 U 信息（用于 ECDH-ES） |
| `apv` | 协议方 V 信息（用于 ECDH-ES） |
| `iv` | Initialization Vector |
| `tag` | Authentication Tag |
| `epk` | Ephemeral Public Key（用于 ECDH-ES） |

---

# 5. JWA（JSON Web Algorithms）— RFC 7518

## 5.1 JWA 算法分类

JWA 定义了 JOSE 中可用的所有算法集合：

### 5.1.1 签名算法（用于 JWS）

| 算法 | 描述 | 密钥要求 |
|------|------|----------|
| `HS256` | HMAC using SHA-256 | 对称密钥 ≥ 256 位 |
| `HS384` | HMAC using SHA-384 | 对称密钥 ≥ 384 位 |
| `HS512` | HMAC using SHA-512 | 对称密钥 ≥ 512 位 |
| `RS256` | RSASSA-PKCS1-v1_5 using SHA-256 | RSA 密钥 ≥ 2048 位 |
| `RS384` | RSASSA-PKCS1-v1_5 using SHA-384 | RSA 密钥 |
| `RS512` | RSASSA-PKCS1-v1_5 using SHA-512 | RSA 密钥 |
| `ES256` | ECDSA using P-256 and SHA-256 | EC P-256 密钥 |
| `ES384` | ECDSA using P-384 and SHA-384 | EC P-384 密钥 |
| `ES512` | ECDSA using P-521 and SHA-512 | EC P-521 密钥 |
| `PS256` | RSASSA-PSS using SHA-256 and MGF1 | RSA 密钥 |
| `PS384` | RSASSA-PSS using SHA-384 and MGF1 | RSA 密钥 |
| `PS512` | RSASSA-PSS using SHA-512 and MGF1 | RSA 密钥 |
| `EdDSA` | EdDSA signature algorithm | Ed25519/Ed448 密钥 |
| `none` | No digital signature | **应禁用** |

### 5.1.2 密钥加密算法（用于 JWE）

| 算法 | 描述 |
|------|------|
| `RSA1_5` | RSAES-PKCS1-v1_5 |
| `RSA-OAEP` | RSAES OAEP + SHA-1 MGF1 |
| `RSA-OAEP-256` | RSAES OAEP + SHA-256 MGF1 |
| `ECDH-ES` | ECDH Ephemeral-Static |
| `ECDH-ES+A128KW` | ECDH + AES 128-bit Key Wrapping |
| `ECDH-ES+A256KW` | ECDH + AES 256-bit Key Wrapping |
| `A128KW` | AES Key Wrap 128-bit |
| `A256KW` | AES Key Wrap 256-bit |
| `PBES2-*` | Password-Based Encryption |

### 5.1.3 内容加密算法（用于 JWE）

| 算法 | 密钥长度 |
|------|----------|
| `A128CBC-HS256` | 256 位 |
| `A256CBC-HS512` | 512 位 |
| `A128GCM` | 128 位 |
| `A192GCM` | 192 位 |
| `A256GCM` | 256 位 |

---

# 6. 完整示例：签名 + 加密嵌套 JWT

## 6.1 Nimbus JOSE+JWT 完整实现

```java
public class NestedJwtExample {
    
    public static String createNestedJwt() throws Exception {
        // ===== 1. 准备密钥 =====
        // 签名密钥（发送方私钥）
        KeyPair signKeyPair = loadOrGenerateRSAKeyPair();
        PrivateKey signPrivateKey = signKeyPair.getPrivate();
        
        // 加密密钥（接收方公钥）
        RSAPublicKey encPublicKey = (RSAPublicKey) loadRecipientPublicKey();
        
        // ===== 2. 创建内层 JWS（签名） =====
        JWSHeader jwsHeader = new JWSHeader.Builder(JWSAlgorithm.RS256)
            .type(JOSEObjectType.JWT)
            .keyID("sender-key-001")
            .build();
        
        JWTClaimsSet claimsSet = new JWTClaimsSet.Builder()
            .issuer("https://sender.example.com")
            .subject("user123")
            .audience("https://receiver.example.com")
            .issueTime(new Date())
            .expirationTime(new Date(System.currentTimeMillis() + 3600_000))
            .jwtID(UUID.randomUUID().toString())
            .claim("sensitive_data", "This is confidential")
            .build();
        
        SignedJWT signedJWT = new SignedJWT(jwsHeader, claimsSet);
        signedJWT.sign(new RSASSASigner(signPrivateKey));
        
        // ===== 3. 创建外层 JWE（加密） =====
        JWEHeader jweHeader = new JWEHeader.Builder(
                JWEAlgorithm.RSA_OAEP_256,
                EncryptionMethod.A256GCM)
            .contentType("JWT")       // 指示内层是 JWT
            .keyID("recipient-key-001")
            .build();
        
        JWEObject jweObject = new JWEObject(jweHeader, new Payload(signedJWT));
        jweObject.encrypt(new RSAEncrypter(encPublicKey));
        
        // ===== 4. 序列化 =====
        return jweObject.serialize();
        // 5 部分：header.encryptedKey.iv.ciphertext.tag
        // 每部分都是 Base64URL 编码
    }
    
    public static Claims decryptAndVerify(String nestedToken) throws Exception {
        // ===== 1. 解析外层 JWE =====
        JWEObject jweObject = JWEObject.parse(nestedToken);
        
        // ===== 2. 解密 =====
        RSAPrivateKey encPrivateKey = (RSAPrivateKey) loadRecipientPrivateKey();
        jweObject.decrypt(new RSADecrypter(encPrivateKey));
        
        // ===== 3. 提取内层 JWS =====
        SignedJWT signedJWT = jweObject.getPayload().toSignedJWT();
        
        // ===== 4. 验证签名 =====
        RSAPublicKey signPublicKey = (RSAPublicKey) loadSenderPublicKey();
        if (!signedJWT.verify(new RSASSAVerifier(signPublicKey))) {
            throw new SecurityException("Signature verification failed");
        }
        
        // ===== 5. 提取 Claims =====
        JWTClaimsSet claims = signedJWT.getJWTClaimsSet();
        
        // 验证标准 Claims
        if (claims.getExpirationTime().before(new Date())) {
            throw new SecurityException("Token expired");
        }
        
        return claims;
    }
}
```

---

# 7. 格式与场景选择指南

## 7.1 使用场景决策

```
需要数据完整性（防篡改）？
    ├── 是 → 使用 JWS（JWT 默认形式）
    └── 否 → 继续判断
    
需要数据机密性（防止内容被看到）？
    ├── 是 → 使用 JWE
    │         └── 同时需要完整性？
    │              ├── 是 → 嵌套 JWT（JWS → JWE）
    │              └── 否 → 仅 JWE
    └── 否 → 仅 JWS（最常见）

多密钥、多算法、多接收方？
    └── 是 → JWK Set + kid 分发
```

## 7.2 推荐配置

```
Web API 认证（常规）:
  JWS + RS256/ES256 + JWK Set 端点

敏感数据传输（医疗、金融）:
  嵌套 JWT（JWS + JWE）:
    内层签名 → RS256
    外层加密 → RSA-OAEP-256 + A256GCM

内部微服务通信:
  JWS + HS256（同域内）

移动端 API:
  JWS + ES256（密钥小，速度快）
```
