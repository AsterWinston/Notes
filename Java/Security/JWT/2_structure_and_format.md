# 1. JWT 的基本结构

JWT 由三部分组成，使用点号（`.`）分隔：

```
header.payload.signature
```

每部分先经过 JSON 序列化，再进行 Base64URL 编码。最终形式如下：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

## 1.1 JWT 的 ABNF 定义

根据 RFC 7519，JWT 的正式语法定义如下：

```
JWT = [header] "." [payload] "." [signature]

header    = BASE64URL(UTF8(JSON(Header)))
payload   = BASE64URL(UTF8(JSON(Payload)))
signature = BASE64URL(JWS Signature 或 JWE 密文)
```

---

# 2. Header（头部）

Header 是一个 JSON 对象，描述令牌的元信息，主要包含两个字段：

## 2.1 Header 的必需字段

### 2.1.1 `alg`（Algorithm）— 算法

指定签名或加密使用的算法。该字段是**必需**的，用于告知接收方如何验证令牌。

```json
{
  "alg": "HS256"
}
```

### 2.1.2 `typ`（Type）— 类型

通常设置为 `JWT`，表示这是一个 JWT 令牌。当嵌套签名/加密时，使用 `JOSE`。

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

## 2.2 Header 的可选字段

| 字段 | 全称 | 说明 |
|------|------|------|
| `cty` | Content Type | 嵌套令牌时指明内容类型，通常为 `JWT` |
| `kid` | Key ID | 用于标识使用哪个密钥进行签名，在多密钥环境下非常重要 |
| `jku` | JWK Set URL | 指向 JWK Set 的 URL |
| `jwk` | JSON Web Key | 直接在 Header 中嵌入公钥（不推荐） |
| `x5u` | X.509 URL | X.509 证书 URL |
| `x5t` | X.509 Certificate SHA-1 Thumbprint | X.509 证书 SHA-1 指纹 |
| `x5t#S256` | X.509 Certificate SHA-256 Thumbprint | X.509 证书 SHA-256 指纹 |
| `crit` | Critical | 声明哪些 Header 参数是必须被理解处理的 |

## 2.3 Header 示例

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "2025-01-key-001",
  "x5t#S256": "abc123def456..."
}
```

对应的 Base64URL 编码过程：

```
原始 JSON → Base64URL 编码 → "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjIwMjUtMDEta2V5LTAwMSJ9"
```

---

# 3. Payload（负载）

Payload 是一个 JSON 对象，包含 Claims（声明）——关于实体（通常是用户）及其附加数据的陈述。

详见 [第 4 章：声明与负载](4_claims_and_payload.md)，此处仅概述结构：

```json
{
  "iss": "https://auth.example.com",
  "sub": "user123",
  "aud": "api.example.com",
  "exp": 1712345678,
  "iat": 1712345600,
  "nbf": 1712345600,
  "jti": "unique-token-id-001",
  "name": "John Doe",
  "email": "john@example.com",
  "roles": ["ROLE_USER", "ROLE_ADMIN"]
}
```

## 3.1 Payload 的 Base64URL 编码

注意：Payload 是**编码**而非**加密**。任何人拿到 JWT 都可以解码查看内容，因此：

- **绝对不要**在 Payload 中存放密码、密钥等敏感信息
- 如需保密性，应使用 JWE 对 JWT 进行加密

---

# 4. Signature（签名）

Signature 用于验证消息在传输过程中未被篡改，对于使用非对称算法的 JWT，还能验证发送者身份。

## 4.1 签名的生成过程

对于 JWS（签名令牌），签名的生成公式为：

```
签名 = HMAC-SHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secret
)
```

或对于非对称算法：

```
签名 = RSA-SHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    privateKey
)
```

### 4.1.1 签名过程详解

```
步骤1：构建签名字符串
  signingInput = BASE64URL(UTF8(Header)) + "." + BASE64URL(UTF8(Payload))

步骤2：使用密钥和算法计算签名
  signature = SIGN(algorithm, signingInput, key)

步骤3：对签名进行 Base64URL 编码
  encodedSignature = BASE64URL(signature)

步骤4：拼接完整 JWT
  jwt = signingInput + "." + encodedSignature
```

## 4.2 验证过程

```
接收方拿到 JWT：
1. 按 "." 拆分为 header、payload、signature
2. 用 header 中的 alg 参数确定算法
3. 计算 signingInput = header + "." + payload
4. 使用密钥验证签名：VERIFY(alg, signingInput, signature, key)
5. 验证 payload 中的 exp、nbf、iss、aud 等声明
```

## 4.3 签名是必需的

- **JWT 可以没有 Signature 吗？** 根据 RFC 7519，JWT 不强制要求签名。理论上存在未签名的 JWT（alg: none），但实际应用中**绝不应接受**未签名或无签名验证的 JWT
- 2015 年出现了著名的 `alg=none` 攻击：攻击者将 Header 中的 `alg` 字段改为 `none` 并删除签名部分，某些实现会错误地认为这是无签名的 JWT 而通过验证

---

# 5. Base64URL 编码

## 5.1 为什么使用 Base64URL

JWT 使用 Base64URL 编码（而非标准 Base64）的原因：

| 特性 | Base64 | Base64URL |
|------|--------|-----------|
| 第 62 个字符 | `+` | `-` |
| 第 63 个字符 | `/` | `_` |
| 填充字符 | `=`（可选） | 通常省略 `=` |

**为什么？** JWT 经常作为 URL 查询参数或在 HTTP Header 中传递：
- `+` 在 URL 中会被解码为空格
- `/` 在 URL 中表示路径分隔符
- `=` 在 URL 中表示键值分隔符

## 5.2 Java 中的 Base64URL 编码

Java 8+ 提供了内置的 Base64URL 支持：

```java
import java.util.Base64;

// 编码
String original = "{\"alg\":\"HS256\",\"typ\":\"JWT\"}";
String encoded = Base64.getUrlEncoder()
    .withoutPadding()
    .encodeToString(original.getBytes(StandardCharsets.UTF_8));

// 解码
byte[] decoded = Base64.getUrlDecoder().decode(encoded);
String result = new String(decoded, StandardCharsets.UTF_8);
```

## 5.3 手动解码 JWT

```bash
# 使用命令行解码 JWT 的 Header 和 Payload
jwt="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.xxx"

# 解码 Header
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" | base64 -d | python3 -m json.tool

# 解码 Payload
echo "eyJzdWIiOiIxMjM0NTY3ODkwIn0=" | base64 -d | python3 -m json.tool
```

---

# 6. JWT 的序列化形式

## 6.1 压缩序列化（Compact Serialization）

最常见的形式：三个 Base64URL 编码段以 `.` 连接。

```
header.payload.signature
```

用于 URL、Authorization Header 等需要紧凑表示的场景。

## 6.2 JSON 序列化（JSON Serialization）

JWS 和 JWE 支持 JSON 序列化，用于多签名或多接收者场景。

### 6.2.1 通用 JSON 序列化

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

### 6.2.2 扁平 JSON 序列化（Flattened JSON Serialization）

```json
{
  "payload": "eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ",
  "protected": "eyJhbGciOiJFUzI1NiJ9",
  "header": {"kid": "e9bc097a-ce51-4036-9562-d2ade882db0d"},
  "signature": "DtEhU3ljbEg8L38VWAfUAqOyKAM6-Xx-F4GawxaepmXFCgfTjDxw5djxLa8ISlSApmWQxfKTUJqPP3-Kg6NU1Q"
}
```

---

# 7. JWT 变体：嵌套令牌

## 7.1 签名 + 加密（Nested JWT）

先对 JWT 签名生成 JWS，再将整个 JWS 作为 Payload 进行加密生成 JWE：

```
原始 JWT → 签名（JWS） → 加密（JWE）
```

常用于需要同时保证完整性和机密性的场景：

```java
// Nimbus JOSE+JWT 示例
// 1. 签名
SignedJWT signedJWT = new SignedJWT(header, claimsSet);
signedJWT.sign(signer);

// 2. 将整个 SignedJWT 作为 Payload 加密
JWEObject jweObject = new JWEObject(
    new JWEHeader(JWEAlgorithm.RSA_OAEP_256, EncryptionMethod.A256GCM),
    new Payload(signedJWT)
);
jweObject.encrypt(encrypter);
String nestedToken = jweObject.serialize();
```

嵌套 JWT 的 Header 中 `cty` 字段会标明内层类型：

```json
{
  "alg": "RSA-OAEP",
  "enc": "A256GCM",
  "cty": "JWT"
}
```

## 7.2 多签名 JWT

同一个 Payload 使用多个密钥或多个算法签名，用于：
- 多方签署同一份声明
- 算法迁移期间同时支持新旧算法

---

# 8. JWT 的大小考量

## 8.1 大小影响因素

| 因素 | 影响 |
|------|------|
| Payload 数据量 | 每增加一个字段，增加约 10-50 字节 |
| 算法类型 | RSA 签名约 256-512 字节，HMAC 签名约 43 字节 |
| 编码开销 | Base64URL 编码会膨胀约 33% |
| 嵌套 | 签名+加密嵌套会使大小翻倍 |

## 8.2 大小限制建议

- HTTP Header 通常限制在 8KB 左右，JWT 应尽量控制在 **2KB 以内**
- Cookie 限制 4KB，避免在 Cookie 中存储 JWT
- 如需传递大量数据，可将详细信息存储在服务端，JWT 中只保留关键标识

---

# 9. 深入理解：JWT 的编解码完整过程

## 9.1 编码（签发）

```
步骤1：构建 Header JSON
{
  "alg": "HS256",
  "typ": "JWT"
}

步骤2：构建 Payload JSON
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}

步骤3：Base64URL 编码 Header
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9

步骤4：Base64URL 编码 Payload
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ

步骤5：构建签名字符串
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ

步骤6：使用密钥计算 HMAC-SHA256
HMAC-SHA256(签名字符串, "your-256-bit-secret") → 32 字节二进制

步骤7：Base64URL 编码签名 → 43 字符字符串

步骤8：拼接完整 JWT
头.体.签名
```

## 9.2 解码（验证）

```
步骤1：按 "." 拆分 JWT → [header, payload, signature]

步骤2：解码 Header JSON
{"alg":"HS256","typ":"JWT"}

步骤3：解码 Payload JSON
{"sub":"1234567890","name":"John Doe","iat":1516239022}

步骤4：根据 alg 选择验证算法 → HMAC-SHA256

步骤5：重新计算签名
HMAC-SHA256(header.payload, secret)

步骤6：比较计算签名与接收到的签名（恒定时间比较防止时序攻击）

步骤7：验证 Claims（exp、nbf、iss、aud 等）

步骤8：确定令牌有效，提取用户信息
```
