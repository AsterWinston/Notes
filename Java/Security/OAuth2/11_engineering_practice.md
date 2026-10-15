# 1. 性能优化

## 1.1 Token 验证性能优化

### 1.1.1 资源服务器端 Token 验证优化

```java
/**
 * 高性能 JWT 验证器
 * 整合了多个优化策略
 */
@Component
public class HighPerformanceJwtValidator {
    
    // 1. 提前过期检查（不解码完整 JWT）
    public boolean isExpiredQuickCheck(String token) {
        String[] parts = token.split("\\.");
        if (parts.length != 3) return true;
        
        try {
            byte[] payload = Base64.getUrlDecoder().decode(parts[1]);
            // 使用轻量 JSON 解析（避免完整反序列化）
            int expIndex = findClaimInJson(payload, "exp");
            if (expIndex < 0) return false;
            
            long exp = parseLongValue(payload, expIndex);
            return System.currentTimeMillis() / 1000 > exp;
        } catch (Exception e) {
            return true; // 解析失败 = 无效 Token
        }
    }
    
    // 2. 缓存验证结果（短期）
    private final Cache<String, JWTClaimsSet> claimCache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .maximumSize(20000)
        .recordStats()
        .build();
    
    public JWTClaimsSet validateWithCache(String token) {
        return claimCache.get(token, t -> {
            if (isExpiredQuickCheck(t)) {
                throw new TokenExpiredException("Token expired (quick check)");
            }
            return doFullValidation(t);
        });
    }
    
    // 3. 公钥预热和定期刷新
    private final LoadingCache<String, RSAPublicKey> keyCache = Caffeine.newBuilder()
        .expireAfterWrite(1, TimeUnit.HOURS)
        .refreshAfterWrite(15, TimeUnit.MINUTES)
        .maximumSize(50)
        .build(this::loadPublicKey);
}
```

### 1.1.2 授权服务器端 Token 生成优化

```java
/**
 * 高性能 Token 生成
 */
@Service
public class OptimizedTokenService {
    
    // 1. 复用 JWSSigner（线程安全，避免每次创建）
    private final JWSSigner signer;
    private final String keyId;
    
    @PostConstruct
    public void init() {
        // 初始化时创建 signer，整个生命周期复用
        this.signer = new RSASSASigner(privateKey);
        this.keyId = UUID.randomUUID().toString();
    }
    
    // 2. 预构建 JWSHeader（避免每次重复创建）
    private final JWSHeader jwtHeader = new JWSHeader.Builder(JWSAlgorithm.RS256)
        .keyID(keyId)
        .type(JOSEObjectType.JWT)
        .build();
    
    // 3. 批量生成 Token（减少对象分配）
    public List<String> generateTokens(List<TokenContext> contexts) {
        return contexts.parallelStream()
            .map(this::generateSingleToken)
            .collect(Collectors.toList());
    }
    
    private String generateSingleToken(TokenContext context) {
        JWTClaimsSet claims = new JWTClaimsSet.Builder()
            .issuer(issuer)
            .subject(context.getUserId())
            .issueTime(new Date())
            .expirationTime(Date.from(Instant.now().plusSeconds(ttl)))
            .jwtID(context.getTokenId())
            .claim("scope", context.getScopes())
            .build();
        
        SignedJWT jwt = new SignedJWT(jwtHeader, claims);
        jwt.sign(signer);
        return jwt.serialize();
    }
}
```

## 1.2 网络开销优化

### 1.2.1 Introspection 调用优化

```java
@Service
public class OptimizedIntrospectionService {
    
    private final RestTemplate restTemplate;
    private final Cache<String, Boolean> activeCache;
    private final Cache<String, List<String>> scopeCache;
    
    /**
     * 分层缓存策略
     */
    public IntrospectionResult introspect(String token) {
        String tokenHash = sha256Hash(token);
        
        // 第一层：内存活跃 Token 缓存（30 秒）
        Boolean isActive = activeCache.getIfPresent(tokenHash);
        if (isActive != null && isActive) {
            // Token 确认为活跃，使用缓存的 Scope
            List<String> scopes = scopeCache.getIfPresent(tokenHash);
            if (scopes != null) {
                return new IntrospectionResult(true, scopes);
            }
        }
        
        // 第二层：远程 Introspection 调用
        IntrospectionResponse response = callIntrospectionEndpoint(token);
        
        if (response.isActive()) {
            // 活跃 Token → 缓存（短期）
            activeCache.put(tokenHash, true, 30, TimeUnit.SECONDS);
            scopeCache.put(tokenHash, response.getScopes(), 30, TimeUnit.SECONDS);
        }
        // 不活跃 Token → 不缓存（负数缓存危险）
        
        return response;
    }
    
    /**
     * 降级与熔断
     */
    @CircuitBreaker(name = "introspection", fallbackMethod = "fallbackIntrospect")
    public IntrospectionResponse callIntrospectionEndpoint(String token) {
        // ... 实际 Introspection 调用
    }
    
    public IntrospectionResponse fallbackIntrospect(String token, Exception e) {
        // 降级：使用 JWT 本地验证
        LOG.warn("Introspection failed, falling back to local JWT validation");
        return localJwtValidation(token);
    }
}
```

---

# 2. 高可用架构设计

## 2.1 授权服务器高可用

```
                    高可用授权服务器架构

                         Load Balancer
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
        │  AS-1      │  │  AS-2      │  │  AS-3      │
        │ (授权服务器) │  │ (授权服务器) │  │ (授权服务器) │
        └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
                    ┌─────────▼─────────┐
                    │                   │
              ┌─────▼─────┐     ┌──────▼──────┐
              │  Redis    │     │    MySQL    │
              │ (Session  │     │ (持久化存储)  │
              │  Token)   │     │             │
              └───────────┘     └─────────────┘

关键设计点：
1. 授权服务器本身无状态（配合 Redis 存储会话）
2. 所有 Token 元数据存 Redis（支持即时撤销）
3. 授权码存 Redis（短期，自动过期）
4. 客户端配置在数据库 + 本地缓存
5. 签名密钥统一由 KMS 管理（所有 AS 实例共享）
```

### 2.1.1 一致性考量

```
OAuth2 状态存储的一致性要求：

高一致性要求（必须强一致）：
  - Authorization Code 的一次性使用
  - Refresh Token 的轮换和重放检测

最终一致性可接受：
  - 客户端信息（注册后短期不一致影响小）
  - 用户授权记录（Consent 决策可宽限期）

实现策略：
  - 授权码和 Refresh Token 用 Redis Lua Script 保证原子性
  - 客户端信息用数据库 + Caffeine 本地缓存 + 消息通知刷新
```

```java
// 使用 Redis Lua Script 保证授权码消费的原子性
@Component
public class RedisAuthorizationCodeService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    private static final String CONSUME_CODE_SCRIPT = """
        local codeKey = KEYS[1]
        local code = redis.call('GET', codeKey)
        if code == false then
            return nil  -- 授权码不存在
        end
        redis.call('DEL', codeKey)  -- 消费：先删除
        return code
        """;
    
    /**
     * 原子性地消费授权码（获取并删除）
     * 防止并发场景下的重放
     */
    public AuthorizationCode consumeCode(String codeHash) {
        String result = redisTemplate.execute(
            new DefaultRedisScript<>(CONSUME_CODE_SCRIPT, String.class),
            Collections.singletonList("auth_code:" + codeHash)
        );
        
        if (result == null) {
            throw new InvalidGrantException("Authorization code not found or already used");
        }
        
        return deserialize(result);
    }
}
```

## 2.2 大规模 Token 吊销

```
场景：用户修改密码或账户被盗，需要立即吊销所有 Token

方案一：维护 Token 版本号
  用户表：token_version INT
  每次修改密码：token_version++
  JWT 中包含：token_version
  验证时：检查 JWT version == DB version
  → 修改密码后，所有旧 JWT 立即失效

方案二：维护颁发时间阈值
  用户表：tokens_valid_after TIMESTAMP
  每次吊销时更新为当前时间
  验证时：检查 JWT.iat >= user.tokens_valid_after

方案三：Redis 黑名单批量操作
  用户注销时：将用户 ID 加入 set
  Key: revoked_users  Value: Set<user_id>
  TTL: max(所有该用户 Token 的剩余有效期)
  验证时：先检查用户是否在 revoked_users 中
```

---

# 3. 多租户实现

## 3.1 多租户架构

```
OAuth2 多租户场景：
  一个授权服务器实例服务多个组织（Tenant）
  每个 Tenant 有独立的用户、客户端、权限

设计模式：

方案 A：Issuer 隔离
  Tenant 1: https://auth.example.com/tenant-a
  Tenant 2: https://auth.example.com/tenant-b
  优点：完全隔离
  缺点：运维复杂、资源浪费

方案 B：OIDC Discovery 中的 acr_values
  acr_values=tenant:tenant-a
  同一 Issuer，通过 claim 区分
  优点：简洁
  缺点：隔离不彻底

方案 C：Issuer 字段区分
  iss: https://auth.example.com/tenant-a
  通过路径区分，所有 Token 包含明确 Issuer
  资源服务器根据 iss 实施不同策略
```

```java
/**
 * 多租户 JWT 验证
 */
@Component
public class MultiTenantJwtValidator {
    
    private final Map<String, TenantConfig> tenantConfigs;
    private final Map<String, JWKSet> tenantJwks;
    
    public Authentication validate(String token, String expectedTenant) {
        SignedJWT jwt = SignedJWT.parse(token);
        String iss = jwt.getJWTClaimsSet().getIssuer();
        
        // 从 Issuer 中提取租户信息
        String tenantId = extractTenantFromIssuer(iss);
        
        // 验证租户
        if (!tenantId.equals(expectedTenant)) {
            throw new InvalidTenantException("Cross-tenant access denied");
        }
        
        // 使用对应租户的 JWKS 验证
        TenantConfig config = tenantConfigs.get(tenantId);
        JWKSet jwkSet = tenantJwks.get(tenantId);
        
        return validateWithTenantConfig(jwt, config, jwkSet);
    }
}
```

---

# 4. 与微服务架构集成

## 4.1 服务间 Token 传递

```
微服务调用链中的 Token 传递：

┌─────────┐     ┌────────────┐
│ 用户     │────>│ Service A  │ (收 Access Token)
└─────────┘     └─────┬──────┘
                      │ 需要调用 Service B
                      │ 使用用户的 Token
                      ▼
               ┌────────────┐
               │ Service B  │ (验证同一个 Access Token)
               └─────┬──────┘
                      │ 需要调用 Service C
                      │ 
                      ▼
               ┌────────────┐
               │ Service C  │
               └────────────┘
```

```java
/**
 * 服务间 Token 传递过滤器
 */
@Component
public class TokenRelayFilter implements ClientHttpRequestInterceptor {
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                         ClientHttpRequestExecution execution) {
        // 从当前 SecurityContext 获取 Token
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            // 将 Token 传递给下游服务
            request.getHeaders().setBearerAuth(jwtAuth.getToken().getTokenValue());
        }
        
        return execution.execute(request, body);
    }
}

// 配置 RestTemplate
@Bean
public RestTemplate restTemplate() {
    RestTemplate restTemplate = new RestTemplate();
    restTemplate.getInterceptors().add(new TokenRelayFilter());
    return restTemplate;
}
```

## 4.2 Token Exchange（RFC 8693）

```
场景：Service A 需要代表用户调用 Service B，但不想传递用户的原始 Token

方案：Token Exchange

POST /token
  grant_type=urn:ietf:params:oauth:grant-type:token-exchange
  &subject_token=<用户的 Access Token>
  &subject_token_type=urn:ietf:params:oauth:token-type:access_token
  &scope=downstream:read
  &audience=ServiceB
  &requested_token_type=urn:ietf:params:oauth:token-type:access_token

授权服务器：
1. 验证 Service A 的身份
2. 验证用户的 Access Token
3. 验证 Service A 有权限代表用户
4. 颁发新的 Access Token（受众=ServiceB，范围缩小）
5. 新 Token 中记录：
   {
     "sub": "user-123",       // 最终用户
     "act": {                 // 行动者（Actor）
       "sub": "service-a"
     }
   }
```

---

# 5. CI/CD 与配置管理

## 5.1 环境管理

```yaml
# application-dev.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-dev.example.com

# application-staging.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-staging.example.com

# application-prod.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

## 5.2 Secrets 管理

```
OAuth2 配置中的 Secrets：

必须通过 Secret Manager 管理的：
  - client_secret
  - JWT 签名私钥
  - 数据库密码
  - Introspection 凭证
  - 加密密钥（Token 存储加密）

推荐工具：
  - HashiCorp Vault
  - AWS Secrets Manager
  - Azure Key Vault
  - Kubernetes Secrets (配合 Sealed Secrets 或 External Secrets)

Spring Boot 集成 Vault：
  spring.cloud.vault.authentication=TOKEN
  spring.cloud.vault.token=${VAULT_TOKEN}
  spring.config.import=vault://secret/oauth2
```
