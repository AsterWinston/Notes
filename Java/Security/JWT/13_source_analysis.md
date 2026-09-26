# 1. 源码分析概述

通过分析主流 JWT 库的源码实现，深入理解 JWT 的底层机制，帮助开发者在遇到问题时能够快速定位根源，以及在选型时做出更明智的决策。

本章重点分析 **JJWT 0.12.x** 和 **Nimbus JOSE+JWT** 两个最常用库的核心实现。

---

# 2. JJWT 0.12.x 源码分析

## 2.1 核心类架构

```
Jwts (入口)
├── JwtBuilder (构建器)
│   ├── DefaultJwtBuilder (签发实现)
│   └── JwtParserBuilder → JwtParser (解析器构建)
├── Jws (已签名 JWT 结果)
├── Claims (负载)
├── SecureDigestAlgorithm (签名算法抽象)
└── KeyAlgorithm → MacAlgorithm / SignatureAlgorithm
```

## 2.2 JwtBuilder 构建流程

### 2.2.1 DefaultJwtBuilder 核心逻辑

```java
// jjwt-impl/src/main/java/io/jsonwebtoken/impl/DefaultJwtBuilder.java
public class DefaultJwtBuilder implements JwtBuilder {
    
    private Header header;
    private Claims claims;
    private SecureDigestAlgorithm<?, ?> algorithm;
    private Key key;
    
    @Override
    public String compact() {
        // 1. 确保 Header 和 Payload 已准备好
        ensureHeader();     // 如果未显式设置，创建默认 Header
        ensurePayload();    // 确保有 Payload（Claims 或 Content）
        
        // 2. 将 Payload 序列化为 JSON 字符串
        String payloadJson = serializePayload();
        
        // 3. 构建签名输入：base64(header) + "." + base64(payload)
        String base64UrlHeader = base64UrlEncode(toJson(header));
        String base64UrlPayload = base64UrlEncode(payloadJson);
        String signingInput = base64UrlHeader + "." + base64UrlPayload;
        
        // 4. 计算签名
        byte[] signatureBytes = sign(signingInput);
        String base64UrlSignature = base64UrlEncode(signatureBytes);
        
        // 5. 拼接最终 JWT
        return base64UrlHeader + "." + base64UrlPayload + "." + base64UrlSignature;
    }
    
    // 签名计算
    private byte[] sign(String signingInput) {
        byte[] inputBytes = signingInput.getBytes(StandardCharsets.US_ASCII);
        
        if (algorithm instanceof MacAlgorithm macAlg) {
            // HMAC 算法
            SecretKey secretKey = (SecretKey) key;
            Mac mac = Mac.getInstance(macAlg.getJcaName());
            mac.init(secretKey);
            return mac.doFinal(inputBytes);
        } else if (algorithm instanceof SignatureAlgorithm sigAlg) {
            // 非对称算法（RSA, EC, EdDSA）
            PrivateKey privateKey = (PrivateKey) key;
            Signature sig = Signature.getInstance(sigAlg.getJcaName());
            sig.initSign(privateKey);
            sig.update(inputBytes);
            return sig.sign();
        }
        throw new JwtException("Unsupported algorithm: " + algorithm);
    }
}
```

### 2.2.2 密钥与算法的绑定机制

JJWT 0.12.x 通过密钥类型推断算法，实现了**算法绑定防御**：

```java
// Jwts.SIG.HS256 是一个 KeyAlgorithm 实现
public class HmacShaAlgorithm implements MacAlgorithm {
    
    // 生成密钥
    @Override
    public SecretKeyBuilder key() {
        return new DefaultSecretKeyBuilder(this);
    }
    
    // 从字节构建密钥
    @Override
    public SecretKey keyFrom(byte[] bytes) {
        // 验证密钥长度
        if (bytes.length * 8 < getKeyBitLength()) {
            throw new WeakKeyException(
                "The specified key byte array is " + bytes.length * 8 +
                " bits which is not secure enough for " + getId() +
                ". The JWT JWA Specification requires keys to be >= " +
                getKeyBitLength() + " bits long.");
        }
        return new SecretKeySpec(bytes, getJcaName());
    }
}

// 验证时自动限制算法
DefaultJwtParser.verifyWith(SecretKey key) {
    // 根据密钥类型推断算法，拒绝不匹配的 alg 声明
    // 例如：传入 HMAC 密钥时，只接受 HS256/HS384/HS512
}
```

## 2.3 JwtParser 解析流程

### 2.3.1 DefaultJwtParser 核心逻辑

```java
// jjwt-impl/src/main/java/io/jsonwebtoken/impl/DefaultJwtParser.java
public class DefaultJwtParser implements JwtParser {
    
    private KeyLocator<?> keyLocator;
    private final ClaimsValidator claimsValidator = new DefaultClaimsValidator();
    
    @Override
    public Jws<Claims> parseSignedClaims(String jwt) {
        // 1. 格式验证
        assertJwtFormat(jwt);           // 检查是否有 2 个 "."
        
        // 2. 拆分 Header、Payload、Signature
        String[] parts = jwt.split("\\.", 3);
        String base64Header = parts[0];
        String base64Payload = parts[1];
        String base64Signature = parts[2];
        
        // 3. 解析 Header
        Header header = parseHeader(base64Header);
        
        // 4. 根据 Header 找到验证密钥
        Key key = keyLocator.locate(header);  // 支持 kid 查找
        
        // 5. 验证签名
        String signingInput = base64Header + "." + base64Payload;
        boolean valid = verifySignature(algorithm, key, signingInput, base64Signature);
        
        if (!valid) {
            throw new SignatureException("JWT signature does not match");
        }
        
        // 6. 解析 Payload
        Claims claims = parseClaims(base64Payload);
        
        // 7. 验证 Claims（exp, nbf, iss, aud）
        claimsValidator.validate(claims);
        
        // 8. 返回结果
        return new DefaultJws<>(header, claims, base64Signature);
    }
    
    // 签名验证实现
    private boolean verifySignature(Algorithm alg, Key key, 
                                     String signingInput, String signatureB64) {
        byte[] signature = base64Decode(signatureB64);
        byte[] input = signingInput.getBytes(StandardCharsets.US_ASCII);
        
        if (alg instanceof MacAlgorithm) {
            Mac mac = Mac.getInstance(alg.getJcaName());
            mac.init((SecretKey) key);
            byte[] computed = mac.doFinal(input);
            return MessageDigest.isEqual(computed, signature); // 恒定时间比较
        } else {
            Signature sig = Signature.getInstance(alg.getJcaName());
            sig.initVerify((PublicKey) key);
            sig.update(input);
            return sig.verify(signature);
        }
    }
}
```

### 2.3.2 恒定时间比较（防止时序攻击）

```java
// JJWT 内部使用 MessageDigest.isEqual() 进行恒定时间比较
// 防止通过签名验证的耗时差异推断签名值

// JDK MessageDigest.isEqual 源码（简化）：
public static boolean isEqual(byte[] digesta, byte[] digestb) {
    if (digesta == digestb) return true;
    if (digesta == null || digestb == null) return false;
    
    int lenA = digesta.length;
    int lenB = digestb.length;
    
    if (lenB == 0) return lenA == 0;
    
    // 恒定时间比较：遍历所有字节，即使已发现不匹配也继续
    int result = 0;
    result |= lenA - lenB;  // 长度差异
    for (int i = 0; i < lenA; i++) {
        // 如果 lenA > lenB，循环访问 digestb 的取模索引
        // 但取模运算会引入分支，这里用位运算优化
        int indexB = ((i - lenB) >>> 31) * i;
        result |= digesta[i] ^ digestb[indexB];
    }
    return result == 0;
}
```

---

# 3. Nimbus JOSE+JWT 源码分析

## 3.1 核心类架构

```
JWT
├── SignedJWT (JWS 实现)
├── EncryptedJWT (JWE 实现)
├── PlainJWT (无签名)
│
JWSObject
├── sign(JWSSigner) → 签名
├── verify(JWSVerifier) → 验证
│
JWEObject
├── encrypt(JWEEncrypter) → 加密
├── decrypt(JWEDecrypter) → 解密
│
JWSSigner / JWSVerifier
├── MACSigner / MACVerifier           (HMAC)
├── RSASSASigner / RSASSAVerifier     (RSA)
├── ECDSASigner / ECDSAVerifier       (EC)
├── Ed25519Signer / Ed25519Verifier   (EdDSA)
│
JWTClaimsSet (Claims 容器)
```

## 3.2 SignedJWT 签名流程

```java
// com/nimbusds/jwt/SignedJWT.java
public class SignedJWT extends JWT implements SignedJWTClaimsSet {
    
    private final JWSHeader header;
    private final Base64URL signature;
    private State state;
    
    // 状态机
    private enum State {
        UNSIGNED,   // 初始状态
        SIGNED,     // 已签名
        VERIFIED    // 已验证
    }
    
    // 签名
    public synchronized void sign(JWSSigner signer) throws JOSEException {
        // 1. 状态检查
        if (state == State.SIGNED || state == State.VERIFIED) {
            throw new IllegalStateException("Already signed");
        }
        
        // 2. 构建签名输入
        // signingInput = base64(protectedHeader) + "." + base64(payload)
        String signingInput = composeSigningInput();
        
        // 3. 委托给具体 Signer 计算签名
        // JWSSigner 是接口，有不同算法实现
        byte[] signatureBytes = signer.sign(
            header, 
            signingInput.getBytes(StandardCharsets.UTF_8)
        );
        
        // 4. Base64URL 编码签名
        this.signature = Base64URL.encode(signatureBytes);
        this.state = State.SIGNED;
    }
    
    // 验证
    public synchronized boolean verify(JWSVerifier verifier) throws JOSEException {
        // 1. 状态检查
        if (state == State.UNSIGNED) {
            throw new IllegalStateException("Not yet signed");
        }
        
        // 2. 构建签名输入
        String signingInput = composeSigningInput();
        
        // 3. 委托给具体 Verifier 验证
        boolean verified = verifier.verify(
            header,
            signingInput.getBytes(StandardCharsets.UTF_8),
            signature.decode()  // Base64URL 解码
        );
        
        if (verified) {
            state = State.VERIFIED;
        }
        
        return verified;
    }
    
    // 关键：签名字符串只包含 header + "." + payload（不含签名部分）
    private String composeSigningInput() {
        return getHeader().toBase64URL().toString() + "." +
               getPayload().toBase64URL().toString();
    }
}
```

## 3.3 MACSigner 实现（HS256）

```java
// com/nimbusds/jose/crypto/MACSigner.java
public class MACSigner extends MACProvider implements JWSSigner {
    
    public MACSigner(byte[] sharedSecret) throws KeyLengthException {
        super(sharedSecret, SUPPORTED_ALGORITHMS); // 支持 HS256, HS384, HS512
    }
    
    @Override
    public Base64URL sign(JWSHeader header, byte[] signingInput) 
            throws JOSEException {
        
        // 1. 根据 header 中的 alg 确定 MAC 算法
        JWSAlgorithm alg = header.getAlgorithm();
        int minKeyLength = getMinKeyLength(alg);
        
        // 2. 验证密钥长度
        if (getSecret().length < minKeyLength) {
            throw new KeyLengthException("The secret length must be at least " 
                + minKeyLength + " bits");
        }
        
        // 3. 获取 JDK MAC 实例
        String jcaAlg = alg.toJCAAlgorithm();  // "HmacSHA256"
        Mac mac = Mac.getInstance(jcaAlg, getJCAContext().getProvider());
        
        // 4. 初始化并计算 HMAC
        mac.init(getSecretKey());
        mac.update(signingInput);
        
        // 5. 返回签名
        return Base64URL.encode(mac.doFinal());
    }
}
```

## 3.4 JWEObject 加密流程

```java
// com/nimbusds/jose/JWEObject.java
public class JWEObject extends JOSEObject {
    
    private final JWEHeader header;
    private Base64URL encryptedKey;
    private Base64URL iv;
    private Base64URL cipherText;
    private Base64URL authTag;
    private State state;
    
    // 加密过程
    public synchronized void encrypt(JWEEncrypter encrypter) throws JOSEException {
        // 1. 生成随机 CEK (Content Encryption Key)
        SecretKey cek = ContentEncryptionKeyGenerator.generate(
            header.getEncryptionMethod()  // 如 A256GCM → 256 位密钥
        );
        
        // 2. 使用密钥加密算法加密 CEK
        // 如 RSA-OAEP 使用接收方公钥加密 CEK
        EncryptedKey encryptedKeyObj = encrypter.encrypt(
            header, cek, getPayload().toBytes());
        this.encryptedKey = Base64URL.encode(encryptedKeyObj.getValue());
        
        // 3. 生成随机 IV
        byte[] ivBytes = generateIV(header.getEncryptionMethod());
        this.iv = Base64URL.encode(ivBytes);
        
        // 4. 使用内容加密算法加密 Payload
        // 如 AES-256-GCM
        AuthenticatedCipherText act = encryptPayload(
            header, cek, ivBytes, getPayload().toBytes());
        
        this.cipherText = Base64URL.encode(act.getCipherText());
        this.authTag = Base64URL.encode(act.getAuthenticationTag());
        
        this.state = State.ENCRYPTED;
    }
}
```

---

# 4. 算法协商与密钥解析机制

## 4.1 Spring Security OAuth2 的 JWT 解析链

```java
// Spring Security 6.x 中的 JWT 处理流程
// NimbusJwtDecoder → JwtDecoder → 返回 Jwt

public final class NimbusJwtDecoder implements JwtDecoder {
    
    private final JWKSource<SecurityContext> jwkSource;
    private final JWSAlgorithm expectedJwsAlgorithm;
    private final OAuth2TokenValidator<Jwt> validator;
    
    @Override
    public Jwt decode(String token) throws JwtException {
        // 1. 解析 JWT
        JWT jwt = JWTParser.parse(token);
        
        if (jwt instanceof SignedJWT signedJwt) {
            // 2. 从 Header 提取算法
            JWSAlgorithm algorithm = signedJwt.getHeader().getAlgorithm();
            
            // 3. 获取验证密钥
            JWK jwk = selectJwk(signedJwt.getHeader()); // 基于 kid
            
            // 4. 创建对应算法的验证器
            JWSVerifier verifier = createVerifier(algorithm, jwk);
            
            // 5. 验证签名
            if (!signedJwt.verify(verifier)) {
                throw new BadJwtException("Invalid signature");
            }
            
            // 6. 提取 Claims 并构建 Jwt 对象
            JwtClaimsSet claimsSet = signedJwt.getJWTClaimsSet();
            Map<String, Object> headers = convertHeaders(signedJwt.getHeader());
            Map<String, Object> claims = convertClaims(claimsSet);
            
            Jwt decodedJwt = new Jwt(token, 
                claimsSet.getIssueTime().toInstant(),
                claimsSet.getExpirationTime().toInstant(),
                headers, claims);
            
            // 7. 自定义验证
            OAuth2TokenValidatorResult validationResult = 
                this.validator.validate(decodedJwt);
            if (validationResult.hasErrors()) {
                throw new JwtValidationException("JWT validation failed", 
                    validationResult.getErrors());
            }
            
            return decodedJwt;
        }
        
        throw new BadJwtException("Unsupported JWT type");
    }
}
```

---

# 5. 防时序攻击的深入分析

## 5.1 时序攻击原理

```
攻击者通过测量服务器处理不同签名值的响应时间来推断正确的签名：

正常签名（前几个字节正确）：服务器在签名验证阶段需要更多时间
错误签名（完全不匹配）：服务器快速返回错误

通过统计大量请求的响应时间差异，攻击者可以逐字节破解签名
```

## 5.2 JJWT 的防护实现

```java
// jjwt-impl 中的 DefaultJwtParser
private boolean isByteArrayEqual(byte[] a, byte[] b) {
    // 使用 MessageDigest.isEqual 进行恒定时间比较
    return MessageDigest.isEqual(a, b);
}

// 关键点：
// 1. 先做长度比较（但使用位运算隐藏长度信息）
// 2. 逐字节 XOR 并累积结果
// 3. 不使用早期返回（不因不匹配提前退出循环）
// 4. 无论是否匹配，循环次数相同
```

## 5.3 自实现恒定时间比较

```java
public class ConstantTimeUtils {
    
    public static boolean equals(byte[] a, byte[] b) {
        if (a == null || b == null) return false;
        
        int result = a.length ^ b.length; // 长度不同则记入 result
        
        int minLen = Math.min(a.length, b.length);
        for (int i = 0; i < minLen; i++) {
            result |= a[i] ^ b[i]; // 逐字节 XOR，累积差异
        }
        
        // 即使提前发现差异，也继续比较所有字节
        return result == 0;
    }
    
    // 方法2：使用 JDK 标准方法（推荐）
    public static boolean secureEquals(byte[] a, byte[] b) {
        return MessageDigest.isEqual(a, b);
    }
}
```

---

# 6. JWT 解析器的线程安全分析

## 6.1 JJWT 线程安全设计

```java
// JwtParser 接口声明：
// "A JwtParser instance is immutable and thread-safe."
// 因为 Parser 是无状态的不可变对象，所有状态在构建时确定

// DefaultJwtParserBuilder.build() 返回的 Parser 是只读的
// Parser 不再依赖外部可变状态

// 正确用法：
@Bean
public JwtParser jwtParser(Key key) {
    return Jwts.parser()
        .verifyWith(key)
        .build();  // 构建后不可变，线程安全
}

// KeyLocator 等回调接口由用户保证线程安全
```

## 6.2 Nimbus 线程安全设计

```java
// SignedJWT 本身不是线程安全的（有状态机）
// 但解析和验证过程中的临时对象是局部的

// JWKSource 接口：
public interface JWKSource<C extends SecurityContext> {
    List<JWK> get(JWKSelector jwkSelector, C context) throws KeySourceException;
}
// 实现需保证线程安全

// RemoteJWKSet 内部使用缓存 + 读写锁：
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
```

---

# 7. 错误处理机制

## 7.1 JJWT 异常层次

```
JwtException
├── SecurityException
│   └── SignatureException
├── MalformedJwtException
├── UnsupportedJwtException
├── ExpiredJwtException
│   └── PrematureJwtException
├── ClaimJwtException
│   ├── IncorrectClaimException
│   └── MissingClaimException
├── InvalidClaimException
├── JwkException
└── WeakKeyException
```

## 7.2 Nimbus 异常层次

```
JOSEException
├── KeyLengthException
├── KeyException
├── BadJOSEException
│   └── BadJWTException
│       └── BadJWSException
└── JWEException
```

---

# 8. 扩展点分析

## 8.1 JJWT 扩展点

```java
// 1. 自定义 Claims 验证器
public class CustomClaimsValidator implements ClaimsValidator {
    @Override
    public Claims validate(Claims claims) {
        // 自定义验证逻辑
        if (!"custom-value".equals(claims.get("custom-claim", String.class))) {
            throw new InvalidClaimException("Custom claim validation failed");
        }
        return claims;
    }
}

// 注册
JwtParser parser = Jwts.parser()
    .verifyWith(key)
    .claimsValidator(new CustomClaimsValidator())
    .build();

// 2. 自定义 KeyLocator
JwtParser parser = Jwts.parser()
    .keyLocator(header -> {
        String kid = (String) header.get("kid");
        return keyService.getKey(kid);
    })
    .build();

// 3. 自定义 CompressionCodec
JwtBuilder builder = Jwts.builder()
    .compressWith(new GzipCompressionCodec())
    .subject("test")
    .signWith(key);
```

## 8.2 Nimbus 扩展点

```java
// 1. 自定义 JWSSigner / JWSVerifier
public class CustomSigner implements JWSSigner {
    @Override
    public Base64URL sign(JWSHeader header, byte[] signingInput) {
        // 自定义签名实现
    }
}

// 2. 自定义 JWTClaimsSetVerifier
JWTClaimsSetVerifier<SecurityContext> verifier = 
    new DefaultJWTClaimsVerifier<>(expectedClaims, requiredClaims);

// 3. 自定义 JWKSource（如从数据库加载）
public class DatabaseJwkSource implements JWKSource<SecurityContext> {
    @Override
    public List<JWK> get(JWKSelector selector, SecurityContext context) {
        // 从数据库查询 JWK
    }
}
```
