
# 1. 概述

在传统单体应用中，Session 存储在应用服务器的内存中。当应用横向扩展为多实例时，同一个用户的请求可能路由到不同实例，导致 Session 丢失。分布式 Session 解决的是多实例场景下会话一致性的问题。

```
┌──────────────────────────────────────────────────────────────────┐
│                    分布式 Session 问题                            │
│                                                                   │
│   用户请求 1 ──→ Nginx ──→ App 实例 1                             │
│       │                        │ 创建 Session (sessionid=ABC)     │
│       │                        │ 存用户信息                       │
│                                                                    │
│   用户请求 2 ──→ Nginx ──→ App 实例 2                             │
│                                │ 查找 Session (sessionid=ABC)     │
│                                │ ❌ 找不到！用户变为未登录         │
│                                                                    │
│   解决方案：                                                       │
│   ├─ 方案1: 粘性路由（ip_hash）                                   │
│   ├─ 方案2: Session 复制                                          │
│   ├─ 方案3: 集中存储（Redis/DB） ← 最推荐                         │
│   └─ 方案4: 无状态（JWT）                                         │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 方案对比

## 2.1 各方案优劣

| 方案 | 原理 | 优点 | 缺点 | 推荐度 |
|------|------|------|------|--------|
| **粘性路由** | Nginx ip_hash 固定请求到同一节点 | 零改造成本 | 节点宕机 Session 丢失 | ⭐⭐ |
| **Session 复制** | 多播/广播 Session 到所有节点 | 节点对等，任意访问 | 网络开销大，O(N²) 复杂度 | ⭐ |
| **集中存储(Redis)** | 所有 Session 存 Redis | 无状态节点，水平扩展 | Redis 成为单点（需集群） | ⭐⭐⭐⭐⭐ |
| **JWT 无状态** | 用户信息编码在 Token | 零服务端存储 | Token 吊销难，体积大 | ⭐⭐⭐⭐ |

# 3. Spring Session + Redis

## 3.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  session:
    store-type: redis              # 使用 Redis 存储
    timeout: 30m                   # Session 超时（30 分钟）
    redis:
      namespace: "myapp:session"   # Redis key 前缀
      flush-mode: on_save          # 保存时机：on_save | immediate
      save-mode: on_set_attribute  # 变更追踪：on_set_attribute | always

  redis:
    host: redis.example.com
    port: 6379
    password: ${REDIS_PASSWORD}
    timeout: 3000ms
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5

  # 启用 Spring Session
  # 添加 @EnableRedisHttpSession 或通过此配置自动启用
```

## 3.2 Java Config

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800) // 30 分钟
public class SessionConfig {

    /**
     * Session 事件监听
     */
    @Bean
    public ApplicationListener<SessionCreatedEvent> sessionCreatedListener() {
        return event -> {
            String sessionId = event.getSession().getId();
            log.info("Session 创建: {}", sessionId);
        };
    }

    @Bean
    public ApplicationListener<SessionDestroyedEvent> sessionDestroyedListener() {
        return event -> {
            String sessionId = event.getSession().getId();
            log.info("Session 销毁: {}", sessionId);
        };
    }

    @Bean
    public ApplicationListener<SessionExpiredEvent> sessionExpiredListener() {
        return event -> {
            String sessionId = event.getSession().getId();
            log.info("Session 过期: {}", sessionId);
        };
    }

    /**
     * Cookie 序列化器配置
     */
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("MYAPP_SESSION");
        serializer.setCookiePath("/");
        serializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$");
        serializer.setUseHttpOnlyCookie(true);
        serializer.setUseSecureCookie(true);
        serializer.setSameSite("Lax");     // SSO 推荐 Lax
        return serializer;
    }

    /**
     * Redis 序列化器（推荐 Jackson）
     */
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        // 使用 Jackson 序列化，支持复杂对象
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

## 3.3 Redis 中的 Session 数据结构

```
# Spring Session 在 Redis 中的 Key 结构

# 1. Session 数据（Hash）
Key:   myapp:session:sessions:session-uuid-xxx
Type:  Hash
TTL:   30 minutes
Fields:
  creationTime: 1712345678000
  lastAccessedTime: 1712349278000
  maxInactiveInterval: 1800
  sessionAttr:SPRING_SECURITY_CONTEXT: <序列化的 SecurityContext>
  sessionAttr:USER_INFO: <序列化的用户信息>

# 2. 过期索引（Set）
Key:   myapp:session:sessions:expires:session-uuid-xxx
Type:  String
Value: ""
TTL:   30 minutes

# 3. 按用户名索引（Set）- 用于并发控制
Key:   myapp:session:index:org.springframework.session.FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME:zhangsan
Type:  Set
Members: session-uuid-1, session-uuid-2
```

## 3.4 手动操作 Session（API）

```java
@RestController
public class SessionController {

    @Autowired
    private FindByIndexNameSessionRepository<? extends Session>
        sessionRepository;

    /**
     * 获取当前用户的 Session
     */
    @GetMapping("/session/info")
    public Map<String, Object> sessionInfo(HttpSession session) {
        Map<String, Object> info = new HashMap<>();
        info.put("id", session.getId());
        info.put("creationTime", new Date(session.getCreationTime()));
        info.put("lastAccessedTime", new Date(session.getLastAccessedTime()));
        info.put("maxInactiveInterval", session.getMaxInactiveInterval());

        // 遍历所有属性
        Map<String, Object> attrs = new HashMap<>();
        Enumeration<String> names = session.getAttributeNames();
        while (names.hasMoreElements()) {
            String name = names.nextElement();
            attrs.put(name, session.getAttribute(name));
        }
        info.put("attributes", attrs);

        return info;
    }

    /**
     * 查询某个用户的所有活跃 Session
     */
    @GetMapping("/admin/sessions/{username}")
    public List<Map<String, Object>> getUserSessions(@PathVariable String username) {
        Map<String, ? extends Session> sessions = sessionRepository
            .findByPrincipalName(username);

        return sessions.values().stream()
            .map(s -> {
                Map<String, Object> info = new HashMap<>();
                info.put("id", s.getId());
                info.put("creationTime", s.getCreationTime());
                info.put("lastAccessedTime", s.getLastAccessedTime());
                return info;
            })
            .collect(Collectors.toList());
    }

    /**
     * 强制踢出用户（管理员操作）
     */
    @DeleteMapping("/admin/sessions/{sessionId}")
    public String kickSession(@PathVariable String sessionId) {
        sessionRepository.deleteById(sessionId);
        return "Session " + sessionId + " 已被强制终止";
    }

    /**
     * 修改 Session 超时时间
     */
    @PutMapping("/session/timeout/{minutes}")
    public String setTimeout(@PathVariable int minutes, HttpSession session) {
        session.setMaxInactiveInterval(minutes * 60);
        return "超时时间已设置为 " + minutes + " 分钟";
    }
}
```

# 4. SSO 中的分布式 Session

## 4.1 SSO 会话层级

```
┌──────────────────────────────────────────────────────────────┐
│                    SSO 三层会话                                │
│                                                               │
│  1. 全局会话（CAS TGT / OAuth2 Login Session）                │
│     ├─ 存储在认证中心的 Redis 中                              │
│     └─ 浏览器携带 TGC Cookie / SSO Session Cookie             │
│                                                               │
│  2. 网关会话（Gateway Session）                               │
│     ├─ 网关作为集中认证点，存储用户 Session                    │
│     └─ 浏览器与网关之间维护                                    │
│                                                               │
│  3. 应用本地会话（App Session）                               │
│     ├─ 通过 Spring Session + Redis 共享                      │
│     └─ 各应用实例之间共享，用户无感知切换                      │
└──────────────────────────────────────────────────────────────┘
```

## 4.2 多应用共享 Session 配置

```java
/**
 * 多个应用共享同一个 Redis 作为 Session 存储
 * 通过 namespace 和 cookie 配置实现隔离或共享
 */

// ===== 方案 A：Session 不共享，各自独立 =====
// 每个应用使用不同的 namespace 和 cookie name
// app-A:
spring.session.redis.namespace=appa:session
server.servlet.session.cookie.name=APP_A_SESSION

// app-B:
spring.session.redis.namespace=appb:session
server.servlet.session.cookie.name=APP_B_SESSION

// ===== 方案 B：Session 完全共享 =====
// 所有应用使用相同的配置
// 所有应用:
spring.session.redis.namespace=shared:session
server.servlet.session.cookie.name=SHARED_SESSION
server.servlet.session.cookie.domain=.example.com  // 同域共享
server.servlet.session.cookie.path=/

// ⚠️ 注意：
// - Session 共享意味着用户在一个应用登录，所有应用都能看到
// - Session 中的 SecurityContext 是 Spring Security 的，跨应用不一定兼容
// - 通常不建议完全共享 Session，而是通过 SSO Token 统一认证

// ===== 方案 C：SSO Token + 各自本地 Session（推荐） =====
// 每个应用独立 Session，通过共享的 SSO Cookie/Token 实现认证
```

# 5. Session 安全与性能

## 5.1 安全配置

```java
@Configuration
@EnableWebSecurity
public class SessionSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                // 会话策略
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                // 会话固定防护
                .sessionFixation().migrateSession()
                // 并发控制
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)  // new login kicks old
                .expiredUrl("/login?expired")
                .sessionRegistry(sessionRegistry())
            );
        return http.build();
    }

    @Bean
    public SpringSessionBackedSessionRegistry<? extends Session>
            sessionRegistry() {
        return new SpringSessionBackedSessionRegistry<>(
            sessionRepository, new ConcurrentHashMap<>());
    }
}
```

## 5.2 性能优化

```yaml
# Redis Session 性能优化配置
spring:
  session:
    redis:
      flush-mode: on_save      # on_save: 仅在保存时写入（推荐）
                                # immediate: 每次变更立即写入
      save-mode: on_set_attribute  # 属性变更时标记脏数据
                                    # always: 每次请求都保存
                                    
  redis:
    lettuce:
      pool:
        max-active: 20         # 连接池大小
        max-idle: 10
        min-idle: 5
        max-wait: 3000ms
      cluster:
        refresh:
          adaptive: true       # 自适应拓扑刷新
          period: 30s
```

```java
/**
 * Session 序列化性能优化
 * GenericJackson2JsonRedisSerializer 适合复杂对象但序列化体积大
 * 可自定义序列化器只序列化必要字段
 */
@Bean
public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.registerModule(new JavaTimeModule());
    mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    mapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
    // 忽略不必要的属性
    mapper.addMixIn(Authentication.class, AuthenticationMixin.class);
    mapper.addMixIn(UserDetails.class, UserDetailsMixin.class);
    return new GenericJackson2JsonRedisSerializer(mapper);
}

@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY, getterVisibility = JsonAutoDetect.Visibility.NONE)
abstract class AuthenticationMixin {}

@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY)
abstract class UserDetailsMixin {}
```

# 6. Redis 高可用

## 6.1 Redis Sentinel

```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - sentinel1:26379
        - sentinel2:26379
        - sentinel3:26379
    password: ${REDIS_PASSWORD}
```

## 6.2 Redis Cluster

```yaml
spring:
  redis:
    cluster:
      nodes:
        - redis-cluster-1:6379
        - redis-cluster-2:6379
        - redis-cluster-3:6379
        - redis-cluster-4:6379
        - redis-cluster-5:6379
        - redis-cluster-6:6379
      max-redirects: 3
```

## 6.3 故障降级

```java
/**
 * Redis 不可用时的降级策略
 * 回退到本地 Session（容忍短暂不一致）
 */
@Configuration
public class SessionFallbackConfig {

    @Bean
    public SessionRepository<?> sessionRepository(
            RedisIndexedSessionRepository redisRepo) {

        return new SessionRepository<MapSession>() {
            private final Map<String, MapSession> localFallback = new ConcurrentHashMap<>();

            @Override
            public MapSession createSession() {
                try {
                    // 优先使用 Redis
                    return redisRepo.createSession();
                } catch (Exception e) {
                    // Redis 不可用，回退到本地
                    MapSession session = new MapSession();
                    localFallback.put(session.getId(), session);
                    log.warn("Redis 不可用，使用本地 Session 降级");
                    return session;
                }
            }

            @Override
            public void save(MapSession session) {
                try {
                    redisRepo.save(session);
                } catch (Exception e) {
                    localFallback.put(session.getId(), session);
                }
            }

            @Override
            public MapSession findById(String id) {
                try {
                    return redisRepo.findById(id);
                } catch (Exception e) {
                    return localFallback.get(id);
                }
            }

            @Override
            public void deleteById(String id) {
                try {
                    redisRepo.deleteById(id);
                } catch (Exception e) {
                    localFallback.remove(id);
                }
            }
        };
    }
}
```

# 7. 总结

| 关键点 | 说明 |
|--------|------|
| **选型** | Spring Session + Redis 是 Java 生态最优解 |
| **序列化** | 使用 Jackson JSON 序列化，避免 JDK 序列化 |
| **安全** | Cookie 设置 HttpOnly + Secure + SameSite=Lax |
| **高可用** | Redis Sentinel / Cluster 保障 |
| **性能** | flush-mode=on_save，避免每次请求都写 Redis |
| **降级** | Redis 不可用时回退到本地 Session |
| **SSO 整合** | 各应用独立 Session + SSO Token 共享认证状态 |
