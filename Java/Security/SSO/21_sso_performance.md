
# 1. 概述

SSO 认证中心是所有用户认证请求的汇聚点，其性能直接影响全局用户体验。一个高并发场景下的认证请求可能在数毫秒内完成，也可能成为整个系统的瓶颈。本章系统讲解 SSO 性能优化的策略。

# 2. 性能指标

## 2.1 关键指标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| **登录响应时间** | < 500ms (P99) | 用户输入密码到完成认证 |
| **Token 验签时间** | < 5ms (P99) | 单次 JWT 验签 |
| **ST 验证时间** | < 50ms (P99) | CAS ST 验证 |
| **SSO 跳转耗时** | < 2s (全程) | 从访问应用到完成认证 |
| **并发处理能力** | > 10,000 TPS | 认证中心吞吐量 |
| **可用性** | 99.99% | 年度可用率 |

# 3. 认证中心性能优化

## 3.1 用户数据缓存

```java
/**
 * 多级缓存策略
 * L1: Caffeine 本地缓存（超快，但有内存限制）
 * L2: Redis 分布式缓存（较快，可共享）
 * L3: 数据库（慢，但数据最全）
 */
@Service
public class UserCacheService {

    // L1: 本地缓存
    private final Cache<String, UserDetails> localCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .recordStats()
        .build();

    // L2: Redis 缓存
    @Autowired
    private RedisTemplate<String, UserDetails> redisTemplate;

    @Autowired
    private UserRepository userRepository;

    public UserDetails loadUser(String username) {
        // L1 查询
        UserDetails user = localCache.getIfPresent(username);
        if (user != null) {
            return user;
        }

        // L2 查询
        user = redisTemplate.opsForValue().get("user:" + username);
        if (user != null) {
            localCache.put(username, user);
            return user;
        }

        // L3 查询（数据库）
        user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        // 回写缓存
        redisTemplate.opsForValue().set("user:" + username, user,
            Duration.ofMinutes(30));
        localCache.put(username, user);

        return user;
    }

    /**
     * 缓存失效（密码修改/用户禁用时调用）
     */
    public void invalidateCache(String username) {
        localCache.invalidate(username);
        redisTemplate.delete("user:" + username);
        // 通过 MQ 广播让其他节点也失效本地缓存
        cacheInvalidationPublisher.publish(username);
    }
}
```

## 3.2 密码验证优化

```java
/**
 * BCrypt 验证是 CPU 密集型操作
 * 优化策略：
 * 1. 降低 BCrypt 轮数（根据安全需求平衡）
 * 2. 使用连接池复用
 * 3. 异步记录登录事件
 */
@Configuration
public class PasswordEncoderConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        // strength=10 意味着 2^10 = 1024 轮哈希
        // 验证耗时约 50-100ms，需根据性能和安全需求调整
        // 强度越低越快，但安全性越低
        return new BCryptPasswordEncoder(10);
        // 高性能场景可考虑 Argon2id（硬件加速更好）
    }
}
```

## 3.3 Token 生成优化

```java
/**
 * JWT 签名是 CPU 密集型操作
 * 优化：预计算 + 连接池
 */
@Service
public class OptimizedJwtService {

    // 使用 JWS Signer 连接池（Nimbus 库）
    private final JWSSigner signer;

    /**
     * 批量预签名（适用于需要批量生成 Token 的场景）
     */
    public List<String> batchCreateTokens(List<UserDetails> users) {
        return users.parallelStream()  // 并行处理
            .map(user -> Jwts.builder()
                .subject(user.getUsername())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + 900_000))
                .signWith(privateKey)
                .compact())
            .collect(Collectors.toList());
    }
}
```

# 4. JWT 验签优化

## 4.1 公钥缓存

```java
/**
 * JWKS 公钥缓存
 * 避免每次验签都请求 JWKS 端点
 */
@Configuration
public class JwtDecoderConfig {

    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder
            .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")

            // 缓存 JWKS 10 分钟
            .cache(Duration.ofMinutes(10))

            // 设置连接池
            .restOperations(createRestTemplate())

            .build();
    }

    private RestOperations createRestTemplate() {
        SimpleClientHttpRequestFactory factory =
            new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(3000);
        factory.setReadTimeout(5000);
        return new RestTemplate(factory);
    }
}
```

## 4.2 本地公钥热加载

```java
/**
 * 本地公钥文件 + 定期热加载
 * 比每次请求 JWKS 更快
 */
@Service
public class LocalKeyManager {

    private volatile RSAPublicKey publicKey;
    private final String keyFilePath = "/etc/secrets/jwt-public.pem";

    @PostConstruct
    public void init() {
        loadKey();
    }

    @Scheduled(fixedRate = 300_000) // 每5分钟检查
    public void reloadKey() {
        loadKey();
    }

    private void loadKey() {
        try {
            String pemContent = Files.readString(Path.of(keyFilePath));
            this.publicKey = PemUtils.parsePublicKey(pemContent);
        } catch (Exception e) {
            log.error("加载公钥失败", e);
        }
    }

    public RSAPublicKey getPublicKey() {
        return publicKey;
    }
}
```

# 5. 数据库优化

## 5.1 索引设计

```sql
-- 用户表索引
CREATE INDEX idx_user_username ON users(username);
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_user_status ON users(enabled, account_locked);

-- Token 表（如果使用数据存储 Token）
CREATE INDEX idx_token_user_expires ON tokens(user_id, expires_at);
CREATE INDEX idx_token_jti ON tokens(jti);

-- 审计日志表（分区表）
CREATE TABLE audit_log (
    id BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    username VARCHAR(100),
    ip_address VARCHAR(45),
    event_time DATETIME(3),
    PRIMARY KEY (id, event_time)
) PARTITION BY RANGE (TO_DAYS(event_time)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01'))
    -- ...
);
```

## 5.2 读写分离

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource router = new RoutingDataSource();
        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put("master", masterDataSource());
        dataSources.put("slave", slaveDataSource());
        router.setTargetDataSources(dataSources);
        router.setDefaultTargetDataSource(masterDataSource());
        return router;
    }

    // 读操作 → slave, 写操作 → master
    @Transactional(readOnly = true)
    public UserDetails loadUser(String username) {
        // 自动路由到 slave
        return userRepository.findByUsername(username);
    }

    @Transactional
    public void updateLoginTime(String username) {
        // 自动路由到 master
        userRepository.updateLastLoginTime(username);
    }
}
```

# 6. 认证请求异步化

## 6.1 非关键操作异步

```java
/**
 * 认证成功后的非关键操作异步执行
 * 减少用户感知的延迟
 */
@Service
public class LoginAsyncService {

    @Autowired
    private TaskExecutor taskExecutor;

    public void onLoginSuccess(String username, HttpServletRequest request) {
        // 同步操作（必须在响应前完成）
        SecurityContextHolder.getContext().setAuthentication(auth);

        // 异步操作（不阻塞响应）
        taskExecutor.execute(() -> {
            // 记录登录日志
            auditService.logLogin(username, request.getRemoteAddr(),
                request.getHeader("User-Agent"));

            // 更新最后登录时间
            userRepository.updateLastLoginTime(username, Instant.now());

            // 发送登录通知
            if (isNewDevice(request)) {
                notificationService.sendNewDeviceAlert(username);
            }
        });
    }
}
```

# 7. 连接池调优

```yaml
# 数据库连接池（HikariCP）
spring:
  datasource:
    hikari:
      maximum-pool-size: 50        # 最大连接数
      minimum-idle: 10              # 最小空闲连接
      connection-timeout: 3000      # 获取连接超时
      idle-timeout: 600000          # 空闲超时（10分钟）
      max-lifetime: 1800000         # 连接最大生命周期（30分钟）

# Redis 连接池（Lettuce）
spring:
  redis:
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 3000ms
      shutdown-timeout: 200ms

# Tomcat 线程池
server:
  tomcat:
    threads:
      max: 200           # 最大工作线程
      min-spare: 20      # 最小空闲线程
    accept-count: 100    # 队列满后等待队列长度
    max-connections: 10000  # 最大连接数
```

# 8. 监控与告警

```java
/**
 * 性能监控埋点
 */
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final MeterRegistry meterRegistry;
    private final Timer loginTimer;

    public AuthController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.loginTimer = Timer.builder("sso.login.duration")
            .description("Login request duration")
            .publishPercentiles(0.5, 0.90, 0.95, 0.99)
            .register(meterRegistry);
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            // 认证逻辑
            Authentication auth = authManager.authenticate(...);

            // 记录成功
            Counter.builder("sso.login.success")
                .register(meterRegistry).increment();

            return ResponseEntity.ok(result);
        } catch (AuthenticationException e) {
            // 记录失败
            Counter.builder("sso.login.failure")
                .tag("reason", e.getClass().getSimpleName())
                .register(meterRegistry).increment();

            throw e;
        } finally {
            sample.stop(loginTimer);
        }
    }
}
```

# 9. 性能优化清单

| 优化点 | 预期收益 | 实现难度 |
|--------|---------|---------|
| **多级缓存（L1+L2）** | RT 降低 80%+ | 中 |
| **公钥缓存（JWT 验签）** | RT 降低 50%+ | 低 |
| **数据库索引优化** | RT 降低 30%+ | 低 |
| **连接池调优** | 并发提升 2-3x | 低 |
| **异步化非关键操作** | 用户感知延迟降低 | 中 |
| **数据库读写分离** | 读性能翻倍 | 中 |
| **BCrypt 强度调整** | RT 降低 50%+ | 低 |
| **JWT Claims 精简** | 网络传输减少 | 低 |
| **Nginx 缓存静态资源** | 前端加载加速 | 低 |
| **CDN 加速** | 全球用户访问加速 | 中 |
| **水平扩展** | 线性扩容 | 中 |
