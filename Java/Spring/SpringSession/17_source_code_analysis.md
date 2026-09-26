# 1. 源码学习路线

```
建议阅读顺序：
1. Session 接口           → 理解核心抽象
2. MapSession            → 理解基本实现
3. SessionRepository     → 理解存储抽象
4. RedisSessionRepository → 理解 Redis 实现（最常用）
5. SessionRepositoryFilter → 理解 Filter 入口
6. FindByIndexNameSessionRepository → 理解索引机制
7. SpringSessionBackedSessionRegistry → 理解 Security 集成
```

---

# 2. Session 接口源码

## 2.1 核心接口定义

```java
// 位置：org.springframework.session.Session
public interface Session {

    String getId();
    String changeSessionId();
    <T> T getAttribute(String attributeName);
    Set<String> getAttributeNames();
    void setAttribute(String attributeName, Object attributeValue);
    void removeAttribute(String attributeName);
    Instant getCreationTime();
    Instant getLastAccessedTime();
    void setLastAccessedTime(Instant lastAccessedTime);
    Duration getMaxInactiveInterval();
    void setMaxInactiveInterval(Duration maxInactiveInterval);
    boolean isExpired();
}
```

**设计分析**：

- 接口与 Servlet API 完全解耦，不依赖 `jakarta.servlet.http.HttpSession`
- 使用 `java.time.Instant` / `java.time.Duration` 代替 `long`，类型安全
- `changeSessionId()` 方法在接口中定义，实现 Session Fixation 防护
- 属性操作使用通用 Object 而非特定类型，保持灵活性

---

# 3. MapSession 源码

## 3.1 核心实现

```java
// 位置：org.springframework.session.MapSession
public final class MapSession implements Session, Serializable {

    private String id;
    private final String originalSessionId;
    private final Map<String, Object> sessionAttrs;
    private Instant creationTime;
    private Instant lastAccessedTime;
    private Duration maxInactiveInterval;

    public MapSession() {
        this(UUID.randomUUID().toString());
    }

    public MapSession(String id) {
        this.id = id;
        this.originalSessionId = id;
        this.sessionAttrs = new ConcurrentHashMap<>();
        this.creationTime = Instant.now();
        this.lastAccessedTime = Instant.now();
        this.maxInactiveInterval = Duration.ofMinutes(30);
    }

    @Override
    public String changeSessionId() {
        String changedId = UUID.randomUUID().toString();
        this.id = changedId;
        return changedId;
    }

    @Override
    public boolean isExpired() {
        return isExpired(Instant.now());
    }

    boolean isExpired(Instant now) {
        if (maxInactiveInterval.isNegative()) {
            return false;  // 永不过期
        }
        return now.minus(maxInactiveInterval).compareTo(lastAccessedTime) >= 0;
    }
}
```

**关键设计点**：

| 设计点 | 分析 |
|--------|------|
| `ConcurrentHashMap` | 线程安全的属性存储，支持并发访问 |
| `originalSessionId` | 保留原始 ID，用于审计和追踪 |
| `isExpired()` | 到期时间为 `lastAccessedTime + maxInactiveInterval` |
| `Serializable` | 支持序列化到外部存储 |
| `final` | 不可继承，避免被子类破坏不变性 |

---

# 4. SessionRepository 接口源码

```java
// 位置：org.springframework.session.SessionRepository
public interface SessionRepository<S extends Session> {

    S createSession();
    void save(S session);
    S findById(String id);
    void deleteById(String id);
}
```

**设计模式分析**：

这是一个典型的**仓储 (Repository) 模式**，将存储操作抽象为统一接口：

```
SessionRepository            — 持久化抽象层
  ├── MapSessionRepository   — 基于 HashMap（测试用）
  ├── RedisSessionRepository — 基于 Redis（最常用）
  ├── JdbcSessionRepository  — 基于 JDBC
  └── HazelcastSessionRepository — 基于 Hazelcast
```

**为什么使用泛型 `<S extends Session>`？**

因为不同存储后端可能有不同的 Session 实现（如 `RedisSession` 有额外的变更追踪），泛型让调用者在编译时就知道具体的 Session 类型。

---

# 5. RedisSessionRepository 源码

## 5.1 类结构

```java
// 位置：org.springframework.session.data.redis.RedisSessionRepository
public class RedisSessionRepository
        implements FindByIndexNameSessionRepository<RedisSession> {

    private final RedisOperations<String, Object> redisOperations;
    private final RedisSessionExpirationPolicy expirationPolicy;
    private String namespace = "spring:session";
    private Duration defaultMaxInactiveInterval = Duration.ofMinutes(30);
    private FlushMode flushMode = FlushMode.ON_SAVE;
    private SaveMode saveMode = SaveMode.ON_SET_ATTRIBUTE;
    private final ApplicationEventPublisher eventPublisher;

    // Redis 中的 Key 构建
    private String getSessionKey(String sessionId) {
        return this.namespace + ":sessions:" + sessionId;
    }

    private String getExpiredKey(String sessionId) {
        return this.namespace + ":sessions:expires:" + sessionId;
    }

    private String getPrincipalIndexKey(String principalName) {
        return this.namespace + ":index:"
            + FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME
            + ":" + principalName;
    }
}
```

## 5.2 save() 方法深度解析

```java
@Override
public void save(RedisSession session) {
    // 1. 检查是否需要保存
    if (!session.isNew() && !session.isChanged()) {
        return;  // 节省不必要的 Redis 写入
    }

    String sessionKey = getSessionKey(session.getId());

    // 2. 构建 HASH 数据
    Map<String, Object> delta = new HashMap<>();

    if (session.isNew()) {
        delta.put("creationTime", session.getCreationTime().toEpochMilli());
        delta.put("maxInactiveInterval",
            session.getMaxInactiveInterval().getSeconds());
        delta.put("lastAccessedTime",
            session.getLastAccessedTime().toEpochMilli());
    }

    // 3. 仅保存变更的属性（增量更新）
    if (saveMode == SaveMode.ON_SET_ATTRIBUTE) {
        delta.putAll(session.getChangedAttributes());
    } else if (saveMode == SaveMode.ALWAYS) {
        // 全量保存所有属性
        for (String name : session.getAttributeNames()) {
            delta.put("sessionAttr:" + name, session.getAttribute(name));
        }
    }

    // 4. 更新 lastAccessedTime
    delta.put("lastAccessedTime", System.currentTimeMillis());

    // 5. 执行 Redis 命令
    redisOperations.opsForHash().putAll(sessionKey, delta);

    // 6. 设置/更新过期时间
    redisOperations.expire(sessionKey,
        session.getMaxInactiveInterval().plus(Duration.ofMinutes(5)));

    // 7. 管理过期 Key（用于 TTL 和事件通知）
    String expiredKey = getExpiredKey(session.getId());
    redisOperations.opsForValue().set(expiredKey, "");
    redisOperations.expire(expiredKey, session.getMaxInactiveInterval());

    // 8. 更新用户索引
    updatePrincipalIndex(session);

    // 9. 清除变更标记
    session.clearChangeFlags();
}
```

**关键设计分析**：

| 设计 | 目的 |
|------|------|
| **Delta 更新** | 仅保存变更的属性，减少网络传输和 Redis 命令 |
| **HASH 存储** | 每个属性一个 Field，支持增量写入 |
| **两个 Key 管理** | Session HASH 和 Expires STRING 分离 |
| **HASH TTL 大于 Expires TTL** | 防止 HASH 先过期导致数据丢失 |

## 5.3 findById() 源码

```java
@Override
public RedisSession findById(String id) {
    String sessionKey = getSessionKey(id);

    // 1. 获取所有 HASH 字段
    Map<Object, Object> map = redisOperations.opsForHash().entries(sessionKey);
    if (map.isEmpty()) {
        return null;
    }

    // 2. 构建 RedisSession 对象
    RedisSession session = new RedisSession(
        id,
        map,
        defaultMaxInactiveInterval
    );

    // 3. 检查是否过期
    if (session.isExpired()) {
        deleteById(id);
        return null;
    }

    return session;
}
```

## 5.4 deleteById() 源码

```java
@Override
public void deleteById(String id) {
    // 1. 先获取 Session（需要知道 principalName 以清理索引）
    RedisSession session = findById(id);

    // 2. 删除 HASH Key
    redisOperations.delete(getSessionKey(id));

    // 3. 删除过期 Key
    redisOperations.delete(getExpiredKey(id));

    // 4. 清理用户索引
    if (session != null) {
        String principalName = session.getAttribute(
            FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME
        );
        if (principalName != null) {
            redisOperations.opsForSet()
                .remove(getPrincipalIndexKey(principalName), id);
        }
    }
}
```

---

# 6. RedisSession 内部实现

```java
final class RedisSession implements Session {

    private String id;
    private String originalSessionId;
    private final Map<String, Object> sessionAttrs = new HashMap<>();
    private final Map<String, Object> changedAttrs = new HashMap<>();   // 变更的 key
    private final Set<String> deletedAttrs = new HashSet<>();           // 删除的 key
    private boolean isNew;
    private boolean changed;

    @Override
    public void setAttribute(String attributeName, Object attributeValue) {
        sessionAttrs.put(attributeName, attributeValue);
        changedAttrs.put("sessionAttr:" + attributeName, attributeValue);
        deletedAttrs.remove(attributeName);
        this.changed = true;
    }

    @Override
    public void removeAttribute(String attributeName) {
        sessionAttrs.remove(attributeName);
        changedAttrs.remove("sessionAttr:" + attributeName);
        deletedAttrs.add(attributeName);
        this.changed = true;
    }

    // 获取变更的属性（用于增量保存）
    Map<String, Object> getChangedAttributes() {
        return changedAttrs;
    }

    void clearChangeFlags() {
        changedAttrs.clear();
        deletedAttrs.clear();
        changed = false;
    }
}
```

---

# 7. SessionRepositoryFilter 源码

## 7.1 核心注册逻辑

```java
// 位置：org.springframework.session.web.http.SessionRepositoryFilter
@Order(SessionRepositoryFilter.DEFAULT_ORDER)
public class SessionRepositoryFilter<S extends Session>
        extends OncePerRequestFilter {

    public static final int DEFAULT_ORDER = Integer.MIN_VALUE + 50;

    private final SessionRepository<S> sessionRepository;
    private final HttpSessionIdResolver httpSessionIdResolver;
    private final ApplicationEventPublisher eventPublisher;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        // 1. 包装请求
        SessionRepositoryRequestWrapper wrappedRequest =
            new SessionRepositoryRequestWrapper(request, response);

        // 2. 包装响应
        SessionRepositoryResponseWrapper wrappedResponse =
            new SessionRepositoryResponseWrapper(request, response);

        try {
            // 3. 继续执行过滤器链
            chain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            // 4. 请求结束后提交 Session
            wrappedRequest.commitSession();
        }
    }

    // 内部类：请求包装器
    private final class SessionRepositoryRequestWrapper
            extends HttpServletRequestWrapper {

        private HttpSessionWrapper requestedSession;
        private boolean requestedSessionCached = false;

        @Override
        public HttpSession getSession(boolean create) {
            if (requestedSessionCached) {
                return requestedSession;
            }
            // 从存储加载或创建 Session
            // ...
            requestedSessionCached = true;
            return requestedSession;
        }
    }

    // 内部类：响应包装器
    private final class SessionRepositoryResponseWrapper
            extends OnCommittedResponseWrapper {

        @Override
        protected void onResponseCommitted() {
            HttpSessionWrapper session = getRequestedSession();
            if (session != null) {
                if (session.isNew()) {
                    httpSessionIdResolver.setSessionId(
                        request, this, session.getId());
                }
                sessionRepository.save(session.getSession());
            }
        }
    }
}
```

**关键设计分析**：

| 设计 | 目的 |
|------|------|
| `OncePerRequestFilter` | 保证每个请求只过滤一次 |
| `DEFAULT_ORDER` | `Integer.MIN_VALUE + 50`，在 Spring Security 之前执行 |
| `SessionRepositoryRequestWrapper` | 包装 `HttpServletRequest`，重写 `getSession()` |
| `SessionRepositoryResponseWrapper` | 继承 `OnCommittedResponseWrapper`，在响应提交时持久化 |
| `commitSession()` | 在 finally 块中提交，确保即使异常也保存 Session |

---

# 8. FindByIndexNameSessionRepository 索引机制

```java
// Redis 实现中的索引更新逻辑
private void updatePrincipalIndex(RedisSession session) {
    String principalName = session.getAttribute(
        FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME
    );

    String oldPrincipalName = session.getOriginalPrincipalName();

    if (principalName != null && !principalName.equals(oldPrincipalName)) {
        // 添加新索引
        String indexKey = getPrincipalIndexKey(principalName);
        redisOperations.opsForSet().add(indexKey, session.getId());
    }

    if (oldPrincipalName != null && !oldPrincipalName.equals(principalName)) {
        // 删除旧索引
        String oldIndexKey = getPrincipalIndexKey(oldPrincipalName);
        redisOperations.opsForSet().remove(oldIndexKey, session.getId());
    }
}
```

**索引数据结构选择**：

```
为什么使用 Redis SET（集合）存储索引？

1. 一个用户名可能对应多个 Session（多设备登录）
2. SET 天然去重
3. 支持 SMEMBERS 快速获取所有 Session ID
4. 支持 SREM 删除单个 Session ID
5. 支持 SCARD 统计会话数量
```

---

# 9. SpringSessionBackedSessionRegistry 源码

```java
// 位置：org.springframework.session.security.SpringSessionBackedSessionRegistry
public class SpringSessionBackedSessionRegistry<S extends Session>
        implements SessionRegistry, ApplicationListener<SessionDestroyedEvent> {

    private final FindByIndexNameSessionRepository<S> sessionRepository;

    @Override
    public List<SessionInformation> getAllSessions(
            Object principal, boolean includeExpiredSessions) {

        // 1. 通过索引查找用户的所有 Session
        Map<String, ? extends S> sessions = sessionRepository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                principal.toString()
            );

        // 2. 转换为 SessionInformation 列表
        return sessions.entrySet().stream()
            .map(entry -> new SessionInformation(
                principal,
                entry.getValue().getId(),
                Date.from(entry.getValue().getLastAccessedTime())
            ))
            .collect(Collectors.toList());
    }

    @Override
    public void onApplicationEvent(SessionDestroyedEvent event) {
        // 当 Session 被销毁时，触发 Spring Security 的过期事件
        // 移除相关的 SessionInformation
    }
}
```

---

# 10. 配置注解源码

## 10.1 @EnableRedisHttpSession

```java
// 位置：org.springframework.session.data.redis.config.annotation.web.http
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(RedisHttpSessionConfiguration.class)
@Configuration
public @interface EnableRedisHttpSession {

    int maxInactiveIntervalInSeconds() default
        MapSession.DEFAULT_MAX_INACTIVE_INTERVAL_SECONDS;

    String redisNamespace() default
        RedisSessionRepository.DEFAULT_NAMESPACE;

    FlushMode flushMode() default FlushMode.ON_SAVE;

    SaveMode saveMode() default SaveMode.ON_SET_ATTRIBUTE;

    String cleanupCron() default RedisHttpSessionConfiguration.DEFAULT_CLEANUP_CRON;
}
```

## 10.2 自动配置导入逻辑

```java
// RedisHttpSessionConfiguration（简化）
@Configuration
public class RedisHttpSessionConfiguration
        extends SpringHttpSessionConfiguration {

    @Bean
    public RedisSessionRepository sessionRepository(
            RedisOperations<String, Object> redis) {
        RedisSessionRepository repository =
            new RedisSessionRepository(redis);
        repository.setDefaultMaxInactiveInterval(/* ... */);
        repository.setRedisNamespace(/* ... */);
        repository.setFlushMode(/* ... */);
        repository.setSaveMode(/* ... */);
        return repository;
    }

    @Bean
    public SessionRepositoryFilter<?> springSessionRepositoryFilter(
            SessionRepository<?> repository) {
        SessionRepositoryFilter<?> filter =
            new SessionRepositoryFilter<>(repository);
        filter.setHttpSessionIdResolver(httpSessionIdResolver());
        return filter;
    }
}
```

---

# 11. 设计模式总结

| 设计模式 | 在 Spring Session 中的应用 |
|----------|---------------------------|
| **仓储模式 (Repository)** | `SessionRepository` 接口抽象存储操作 |
| **策略模式 (Strategy)** | `HttpSessionIdResolver`（Cookie vs Header） |
| **装饰器模式 (Decorator)** | `SessionRepositoryRequestWrapper` 包装原生 HttpServletRequest |
| **观察者模式 (Observer)** | `ApplicationEvent` + `@EventListener` 事件驱动 |
| **模板方法模式** | `OncePerRequestFilter.doFilterInternal()` |
| **工厂模式** | `SessionRepository.createSession()` |
| **适配器模式** | `SpringSessionBackedSessionRegistry` 适配 SessionRegistry |
