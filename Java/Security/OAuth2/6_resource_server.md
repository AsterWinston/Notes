# 1. 资源服务器概述

资源服务器（Resource Server）托管受保护资源，负责验证请求中的 Access Token 并决定是否返回资源。在微服务架构中，资源服务器通常就是各个业务服务。

## 1.1 资源服务器的核心职责

```
资源服务器的三个核心职责：

1. Token 验证 — 确认 Access Token 有效且未过期
2. 权限判定 — Token 的 Scope 是否覆盖当前请求的资源
3. 资源返回 — 根据验证结果返回数据或拒绝访问
```

## 1.2 资源服务器在系统架构中的位置

```
                          API Gateway
                    ┌──────────────────────┐
                    │  Token 初步验证       │
                    │  路由分发             │
                    └──────┬───────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
    ┌─────────▼──┐  ┌──────▼─────┐  ┌──▼──────────┐
    │ Resource   │  │ Resource   │  │ Resource     │
    │ Server A   │  │ Server B   │  │ Server C     │
    │ (用户服务)  │  │ (订单服务)  │  │ (商品服务)    │
    │            │  │            │  │              │
    │ Token 验证 │  │ Token 验证 │  │ Token 验证    │
    │ Scope 检查 │  │ Scope 检查  │  │ Scope 检查    │
    └────────────┘  └────────────┘  └──────────────┘
```

---

# 2. Token 验证策略

## 2.1 三种验证策略对比

| 策略 | 实现方式 | 延迟 | 可用性 | 撤销支持 | 适用场景 |
|------|----------|------|--------|----------|----------|
| 本地验证 | JWT 签名验证 | 极低 | 高（无外部依赖） | 弱（依赖过期） | 高并发、低延迟系统 |
| 远程验证 | Introspection 端点 | 较高 | 依赖授权服务器 | 强（即时生效） | 需要即时撤销的场景 |
| 混合策略 | 本地优先 + 定期刷新黑名单 | 低 | 高 | 较强 | 大规模分布式系统 |

## 2.2 本地验证（JWT 自包含 Token）

### 验证流程

```java
public class JwtTokenValidator {
    
    private final Map<String, PublicKey> publicKeyCache = new ConcurrentHashMap<>();
    
    /**
     * 验证 JWT Access Token 并提取认证信息
     */
    public Authentication validateJwtToken(String jwtToken) {
        try {
            // 1. 解析 JWT（不验证签名，先获取 header 信息）
            SignedJWT signedJWT = SignedJWT.parse(jwtToken);
            JWSHeader header = signedJWT.getHeader();
            
            // 2. 获取签名密钥
            String keyId = header.getKeyID();
            PublicKey publicKey = getPublicKey(keyId);
            
            // 3. 验证签名
            JWSVerifier verifier = new RSASSAVerifier((RSAPublicKey) publicKey);
            if (!signedJWT.verify(verifier)) {
                throw new InvalidTokenException("JWT signature verification failed");
            }
            
            // 4. 验证 Claims
            JWTClaimsSet claims = signedJWT.getJWTClaimsSet();
            validateClaims(claims);
            
            // 5. 构建认证对象
            return buildAuthentication(claims);
            
        } catch (ParseException | JOSEException e) {
            throw new InvalidTokenException("JWT parsing failed", e);
        }
    }
    
    /**
     * 验证 JWT Claims
     */
    private void validateClaims(JWTClaimsSet claims) {
        // 验证过期时间
        Date expiration = claims.getExpirationTime();
        if (expiration == null || expiration.before(new Date())) {
            throw new TokenExpiredException("Token has expired");
        }
        
        // 验证颁发时间（允许 5 分钟时钟偏差）
        Date issueTime = claims.getIssueTime();
        if (issueTime != null && issueTime.after(new Date(System.currentTimeMillis() + 300000))) {
            throw new InvalidTokenException("Token issued in the future");
        }
        
        // 验证 Not Before
        Date notBefore = claims.getNotBeforeTime();
        if (notBefore != null && notBefore.after(new Date())) {
            throw new InvalidTokenException("Token not yet valid");
        }
        
        // 验证 Issuer
        String issuer = claims.getIssuer();
        if (issuer == null || !expectedIssuers.contains(issuer)) {
            throw new InvalidTokenException("Invalid issuer: " + issuer);
        }
        
        // 验证 Audience
        List<String> audience = claims.getAudience();
        if (audience == null || !audience.contains(expectedAudience)) {
            throw new InvalidTokenException("Invalid audience");
        }
    }
    
    /**
     * 从 JWKS 端点获取并缓存公钥
     */
    private PublicKey getPublicKey(String keyId) {
        return publicKeyCache.computeIfAbsent(keyId, kid -> {
            try {
                JWKSet jwkSet = JWKSet.load(new URL(jwksUri));
                JWK jwk = jwkSet.getKeyByKeyId(kid);
                if (jwk == null) {
                    throw new InvalidTokenException("Key not found: " + kid);
                }
                return jwk.toRSAKey().toPublicKey();
            } catch (Exception e) {
                throw new KeyResolutionException("Failed to load JWKS", e);
            }
        });
    }
}
```

### 公钥缓存机制

```java
// 使用 Caffeine 缓存 JWKS 公钥，减少网络请求

LoadingCache<String, JWKSet> jwksCache = Caffeine.newBuilder()
    .expireAfterWrite(15, TimeUnit.MINUTES)  // JWKS 更新不频繁
    .refreshAfterWrite(10, TimeUnit.MINUTES) // 异步刷新
    .build(keyId -> JWKSet.load(new URL(jwksUri)));

// 缓存未命中时的降级策略：
// 1. 重试获取
// 2. 检查是否有旧版本缓存可用
// 3. 全部失败 → 拒绝请求（安全优先）
```

## 2.3 远程验证（Opaque Token / Introspection）

```java
public class IntrospectionTokenValidator {
    
    private final RestTemplate restTemplate;
    private final Cache<String, IntrospectionResult> cache;
    
    /**
     * 通过 Introspection 端点验证 Token
     */
    public IntrospectionResult validateOpaqueToken(String token) {
        // 1. 先检查缓存
        IntrospectionResult cached = cache.getIfPresent(token);
        if (cached != null && cached.isActive()) {
            return cached;
        }
        
        // 2. 调用 Introspection 端点
        HttpHeaders headers = new HttpHeaders();
        headers.setBasicAuth(resourceServerId, resourceServerSecret);
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        
        String body = "token=" + URLEncoder.encode(token, StandardCharsets.UTF_8)
                    + "&token_type_hint=access_token";
        
        HttpEntity<String> request = new HttpEntity<>(body, headers);
        
        try {
            ResponseEntity<IntrospectionResponse> response = restTemplate.postForEntity(
                introspectionEndpoint, request, IntrospectionResponse.class);
            
            IntrospectionResult result = new IntrospectionResult(response.getBody());
            
            // 3. 缓存结果
            if (result.isActive()) {
                // 活跃 Token 短期缓存（如 60 秒）
                cache.put(token, result, 60, TimeUnit.SECONDS);
            }
            // 不活跃 Token 不缓存（下次请求可能已重新激活）
            
            return result;
            
        } catch (RestClientException e) {
            // 4. 降级策略
            // 如果缓存中有旧的活跃结果，可以考虑在短时间内使用
            if (cached != null && cached.isGracePeriod()) {
                LOG.warn("Introspection endpoint unavailable, using cached result for token");
                return cached;
            }
            throw new IntrospectionException("Introspection failed", e);
        }
    }
}
```

### Introspection 缓存策略

```
缓存策略精要：

1. 活跃 Token → 短期缓存（30-60 秒）
   原因：减少 Introspection 调用，但保持撤销时效性

2. 不活跃 Token → 不缓存
   原因：负数缓存会导致 Token 有效但被错误拒绝

3. 缓存 Key 设计：
   使用 token 的哈希值而非 token 本身
   key = SHA256(token) 的前 16 字节
   这样即使缓存被泄漏也不会暴露原始 Token

4. 分布式缓存一致性问题：
   - Token 在服务器 A 被撤销
   - 服务器 B 的缓存中该 Token 仍为活跃
   - 解决：撤销操作时广播缓存失效消息（Redis Pub/Sub 等）
```

## 2.4 混合策略（本地验证 + 黑名单）

```
混合验证流程：

1. 先本地验证 JWT 签名（快速过滤无效 Token）
2. 检查 Token ID（jti）是否在黑名单中
3. 都通过 → 接受请求

黑名单维护：
- Redis Key: revoked_tokens:{jti}
- TTL: 对齐该 Token 的原始过期时间
- 撤销操作时写入黑名单

性能数据对比：
┌────────────────┬──────────┬──────────┬──────────┐
│ 策略           │ 延迟     │ 可靠性   │ 即时撤销  │
├────────────────┼──────────┼──────────┼──────────┤
│ 纯 JWT 本地    │ < 1ms    │ 高       │ 否       │
│ 纯 Introspect  │ 5-50ms   │ 中       │ 是       │
│ JWT + 黑名单   │ 1-5ms    │ 高       │ 接近即时  │
└────────────────┴──────────┴──────────┴──────────┘
```

---

# 3. Scope 权限验证

## 3.1 标准 Scope 验证

```java
/**
 * 注解驱动的 Scope 权限验证
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiredScope {
    String[] value();
    Logical logical() default Logical.AND;
}

// 使用示例
@RestController
public class OrderController {
    
    @RequiredScope({"order:read"})
    @GetMapping("/api/orders")
    public List<Order> listOrders() {
        // ...
    }
    
    @RequiredScope({"order:write", "order:admin"}, logical = Logical.OR)
    @PostMapping("/api/orders")
    public Order createOrder() {
        // ...
    }
}

// AOP 切面实现
@Aspect
@Component
public class ScopeValidationAspect {
    
    @Around("@annotation(requiredScope)")
    public Object validateScope(ProceedingJoinPoint joinPoint, RequiredScope requiredScope) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String grantedScopes = auth.getAttributes().get("scope");
        Set<String> grantedSet = Arrays.stream(grantedScopes.split(" "))
            .collect(Collectors.toSet());
        
        List<String> required = Arrays.asList(requiredScope.value());
        boolean hasAccess = requiredScope.logical() == Logical.AND
            ? grantedSet.containsAll(required)
            : required.stream().anyMatch(grantedSet::contains);
        
        if (!hasAccess) {
            throw new InsufficientScopeException(
                "Required scopes: " + required + ", Granted: " + grantedScopes);
        }
        
        return joinPoint.proceed();
    }
}
```

## 3.2 细粒度权限设计（OAuth2 Scope + 应用层 RBAC）

```
OAuth2 Scope 负责粗粒度（资源分类），应用层 RBAC 负责细粒度（操作/数据权限）。

三层权限模型：

第一层：OAuth2 Scope（服务级）
  scope=order:read  →  允许访问订单服务

第二层：应用角色（功能级）
  role=ORDER_VIEWER  →  允许查看订单
  role=ORDER_EDITOR  →  允许编辑订单

第三层：数据权限（记录级）
  用户只能查看自己部门的订单
  经理可以查看下属的订单

实现：
// Token Claims 中包含权限信息
{
  "sub": "user-123",
  "scope": "order:read",
  "authorities": ["ORDER_VIEWER", "ORDER_EDITOR"],
  "department": "engineering"
}
```

---

# 4. 资源服务器与 API Gateway 的协作

## 4.1 Gateway 统一验证 vs 各服务自验证

| 维度 | Gateway 统一验证 | 服务自验证 |
|------|-----------------|-----------|
| 延迟 | Token 验证一次 | 每个服务都验证一次 |
| 一致性 | 集中配置，易保证 | 各服务可能配置不一致 |
| 攻击面 | Gateway 是单点 | 每层都能防御 |
| 故障隔离 | Gateway 故障影响所有服务 | 单服务故障不影响其他 |
| Scope 校验 | 只能在 Gateway 做粗粒度 | 可在业务层做细粒度 |

## 4.2 推荐方案：分层验证

```
┌──────────────────────────────────────────────────────┐
│  API Gateway（第一层验证）                             │
│  ├─ Token 格式/签名验证（非法 Token 在此拦截）          │
│  ├─ 基本有效期检查                                    │
│  ├─ 粗粒度 Scope 验证（app:access）                   │
│  └─ 将验证后的 Claims 放入 Header 传给下游             │
└──────────────────────────────────────────────────────┘
                         │
                         │ X-User-Id: 123
                         │ X-User-Scope: order:read
                         │ X-User-Authorities: ORDER_VIEWER
                         ▼
┌──────────────────────────────────────────────────────┐
│  Resource Server（第二层验证）                         │
│  ├─ 信任 Gateway 传递的 Claims（内部通信）              │
│  ├─ 细粒度 Scope 验证（order:write）                  │
│  ├─ 业务权限验证（该用户能否访问此订单）                 │
│  └─ 速率限制、审计日志                                │
└──────────────────────────────────────────────────────┘
```

> Gateway 到 Resource Server 的内部通信注意事项：
> - 使用 mTLS 确保通信安全
> - 通过固定 Header（如 `X-Auth-User-Id`）传递用户信息
> - Resource Server 应验证请求确实来自 Gateway（IP 白名单或内部 Token）

---

# 5. 资源服务器性能优化

## 5.1 Token 验证性能优化

```java
// 1. 本地缓存解析后的 JWT Claims
@Component
public class TokenClaimCache {
    
    private final Cache<String, JWTClaimsSet> cache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .maximumSize(10000)
        .build();
    
    public JWTClaimsSet getClaims(String token) {
        return cache.get(token, k -> {
            // 解析并验证 JWT
            SignedJWT jwt = SignedJWT.parse(token);
            if (!jwt.verify(verifier)) {
                throw new InvalidTokenException("Invalid signature");
            }
            return jwt.getJWTClaimsSet();
        });
    }
}

// 2. 提前验证过期时间（快速失败，避免解析完整 JWT）
public boolean isExpiredQuickCheck(String token) {
    // 解析 payload（Base64 解码，不验证签名）
    String[] parts = token.split("\\.");
    if (parts.length != 3) return true;
    
    byte[] payload = Base64.getUrlDecoder().decode(parts[1]);
    JsonNode node = objectMapper.readTree(payload);
    long exp = node.get("exp").asLong();
    
    return System.currentTimeMillis() / 1000 > exp;
}

// 3. JWKS 预热
@PostConstruct
public void warmUpJwks() {
    // 应用启动时预先加载 JWKS
    jwksCache.get("default");
}
```

## 5.2 高并发场景设计

```
优化策略：

1. 无状态设计
   - Token 验证不依赖 Session / 数据库
   - 资源服务器可水平扩展

2. 异步 Token 验证
   - 对于 JWT，验证是 CPU 密集型（RSA 签名验证）
   - 使用 Reactive 模型（WebFlux, Project Reactor）提升吞吐

3. 熔断机制
   - Introspection 端点不可用时，使用 JWT 本地验证降级
   - 黑名单服务不可用时，日志告警但继续服务（平衡安全与可用性）

4. 批量验证（少见场景）
   - 如果需要一次验证多个 Token，使用批量 Introspection
```

---

# 6. 错误响应规范

## 6.1 标准错误格式

```json
{
  "error": "invalid_token",
  "error_description": "The access token expired",
  "error_uri": "https://api.example.com/docs/errors#invalid_token"
}
```

## 6.2 资源服务器错误码

| HTTP 状态码 | WWW-Authenticate Header | 说明 |
|-------------|------------------------|------|
| 401 | Bearer error="invalid_token", error_description="..." | Token 无效/过期 |
| 401 | Bearer error="missing_token" | 没有提供 Token |
| 403 | Bearer error="insufficient_scope", scope="order:write" | Scope 不足 |
| 400 | (无) | Token 格式错误 |

```http
# 401 - Token 无效
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="example",
                  error="invalid_token",
                  error_description="The access token expired"

# 403 - Scope 不足
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer realm="example",
                  error="insufficient_scope",
                  scope="order:write"
```

---

# 7. 安全注意事项

```
资源服务器安全清单：

□ Token 传输强制 HTTPS（生产环境）
□ 验证 Issuer（iss）— 防止跨 Issuer Token 滥用
□ 验证 Audience（aud）— 确保 Token 是颁发给本服务的
□ 验证签名算法 — 拒绝 "none" 算法（CVE-2015-2951）
□ JWKS 端点使用 HTTPS 且验证证书
□ Token 验证失败时返回 401（不是 500）
□ 不在日志中记录完整 Token
□ 不在错误响应中泄露 Token 内容
□ 合理设置 Token 有效期（5-15 分钟）
□ Gateway 到 Resource Server 的内部通信需认证
□ 定期轮换签名密钥（Key Rotation）
```

### 签名算法验证（防止 "none" 算法攻击）

```java
// 危险：接受 JWT Header 中声明的算法
String algorithm = header.getAlgorithm();  // 攻击者设置为 "none"
// 后面按 "none" 验证 → Token 永远通过

// 安全：只接受允许的算法列表
private static final Set<String> ALLOWED_ALGORITHMS = Set.of("RS256", "RS384", "RS512", "ES256");

if (!ALLOWED_ALGORITHMS.contains(header.getAlgorithm().getName())) {
    throw new InvalidTokenException("Unsupported algorithm: " + header.getAlgorithm());
}
```
