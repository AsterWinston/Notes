# 1. 自定义 Session ID 生成策略

## 1.1 默认策略

Spring Session 默认使用 `UUID.randomUUID().toString()` 生成 Session ID：

```java
// 默认实现
public String generateId() {
    return UUID.randomUUID().toString();
}
// 示例: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
```

## 1.2 自定义生成器

```java
@Component
public class CustomSessionIdGenerator {

    public String generate() {
        // 方式一：带时间戳的 ID（可排序，便于排查）
        return System.currentTimeMillis() + "-"
            + UUID.randomUUID().toString().substring(0, 8);
        // 示例: "1700000000000-f47ac10b"

        // 方式二：纯字母数字（URL 安全）
        // return Base64.getUrlEncoder().withoutPadding()
        //     .encodeToString(UUID.randomUUID().toString().getBytes());
    }
}
```

---

# 2. 自定义 Cookie 序列化

## 2.1 自定义 CookieSerializer

```java
@Configuration
public class CustomCookieConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();

        // 自定义 Cookie 名称
        serializer.setCookieName("AUTH_SESSION");

        // 自定义 Cookie 路径
        serializer.setCookiePath("/");

        // 域名正则匹配（自动适配子域名）
        serializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$");

        // 自定义 SameSite
        serializer.setSameSite("Strict");

        // 使用 Base64 编码 Session ID（URL 安全）
        // serializer.setUseBase64Encoding(true);

        return serializer;
    }

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver(
            CookieSerializer cookieSerializer) {
        CookieHttpSessionIdResolver resolver =
            new CookieHttpSessionIdResolver();
        resolver.setCookieSerializer(cookieSerializer);
        return resolver;
    }
}
```

## 2.2 Base64 编码 Session ID

默认 Session ID 是 UUID 格式（含 `-`），启用 Base64 编码后更紧凑：

```java
// 默认: "f47ac10b-58cc-4372-a567-0e02b2c3d479" (36 字符)
// Base64: "9HrBC1jMQ3KlZw4CssPUeQ" (22 字符)
// 可节省 Cookie 空间
```

---

# 3. 自定义 Header 策略

## 3.1 多 Header 支持

```java
@Configuration
public class MultiHeaderSessionConfig {

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        // 自定义：同时支持 X-Auth-Token 和 Authorization
        return new HttpSessionIdResolver() {

            @Override
            public List<String> resolveSessionIds(HttpServletRequest request) {
                List<String> ids = new ArrayList<>();

                // 优先从 X-Auth-Token Header 读取
                String token = request.getHeader("X-Auth-Token");
                if (token != null) ids.add(token);

                // 也支持 Bearer Token 格式
                String auth = request.getHeader("Authorization");
                if (auth != null && auth.startsWith("Bearer ")) {
                    ids.add(auth.substring(7));
                }

                return ids;
            }

            @Override
            public void setSessionId(HttpServletRequest request,
                    HttpServletResponse response, String sessionId) {
                response.setHeader("X-Auth-Token", sessionId);
            }

            @Override
            public void expireSession(HttpServletRequest request,
                    HttpServletResponse response) {
                response.setHeader("X-Auth-Token", "");
            }
        };
    }
}
```

## 3.2 混合策略（Cookie + Header 双通道）

```java
@Configuration
public class HybridSessionConfig {

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        CookieHttpSessionIdResolver cookieResolver =
            new CookieHttpSessionIdResolver();
        HeaderHttpSessionIdResolver headerResolver =
            new HeaderHttpSessionIdResolver("X-Auth-Token");

        return new HttpSessionIdResolver() {

            @Override
            public List<String> resolveSessionIds(HttpServletRequest request) {
                List<String> ids = new ArrayList<>();
                ids.addAll(cookieResolver.resolveSessionIds(request));
                ids.addAll(headerResolver.resolveSessionIds(request));
                return ids;
            }

            @Override
            public void setSessionId(HttpServletRequest request,
                    HttpServletResponse response, String sessionId) {
                // 同域请求通过 Cookie，跨域请求通过 Header
                cookieResolver.setSessionId(request, response, sessionId);
                if (isAjaxRequest(request)) {
                    headerResolver.setSessionId(request, response, sessionId);
                }
            }

            @Override
            public void expireSession(HttpServletRequest request,
                    HttpServletResponse response) {
                cookieResolver.expireSession(request, response);
                headerResolver.expireSession(request, response);
            }

            private boolean isAjaxRequest(HttpServletRequest request) {
                return "XMLHttpRequest".equals(
                    request.getHeader("X-Requested-With"));
            }
        };
    }
}
```

---

# 4. 自定义 SessionRepository 实现

## 4.1 何时需要自定义

| 场景 | 说明 |
|------|------|
| **自定义存储** | 使用 Spring Session 不支持的后端（如 OSS、Elasticsearch） |
| **加密存储** | Session 数据需要加密后存储 |
| **多级缓存** | 本地缓存 + 远程存储的二级缓存策略 |
| **自定义过期策略** | 实现特殊的分级过期、滑动窗口等 |
| **数据脱敏** | 存储前对敏感字段进行脱敏处理 |

## 4.2 实现步骤

```java
// Step 1: 创建自定义 Session 类
public class CustomSession implements Session {

    private String id;
    private Instant creationTime;
    private Instant lastAccessedTime;
    private Duration maxInactiveInterval;
    private Map<String, Object> attrs = new ConcurrentHashMap<>();

    // 构造函数
    public CustomSession() {
        this.id = UUID.randomUUID().toString();
        this.creationTime = Instant.now();
        this.lastAccessedTime = Instant.now();
    }

    // 实现 Session 接口的所有方法
    @Override
    public String getId() { return id; }

    @Override
    public String changeSessionId() {
        String newId = UUID.randomUUID().toString();
        this.id = newId;
        return newId;
    }

    @Override
    public <T> T getAttribute(String name) {
        return (T) attrs.get(name);
    }

    @Override
    public Set<String> getAttributeNames() {
        return attrs.keySet();
    }

    @Override
    public void setAttribute(String name, Object value) {
        attrs.put(name, value);
    }

    @Override
    public void removeAttribute(String name) {
        attrs.remove(name);
    }

    @Override
    public Instant getCreationTime() { return creationTime; }

    @Override
    public Instant getLastAccessedTime() { return lastAccessedTime; }

    @Override
    public void setLastAccessedTime(Instant lastAccessedTime) {
        this.lastAccessedTime = lastAccessedTime;
    }

    @Override
    public Duration getMaxInactiveInterval() { return maxInactiveInterval; }

    @Override
    public void setMaxInactiveInterval(Duration interval) {
        this.maxInactiveInterval = interval;
    }

    @Override
    public boolean isExpired() {
        return lastAccessedTime.plus(maxInactiveInterval).isBefore(Instant.now());
    }
}

// Step 2: 实现自定义 SessionRepository
public class CustomSessionRepository
        implements FindByIndexNameSessionRepository<CustomSession> {

    private final YourStorageClient storageClient;

    @Override
    public CustomSession createSession() {
        CustomSession session = new CustomSession();
        session.setMaxInactiveInterval(Duration.ofMinutes(30));
        return session;
    }

    @Override
    public void save(CustomSession session) {
        storageClient.put(session.getId(), serialize(session),
            session.getMaxInactiveInterval());
        // 如果有索引功能，更新索引
        updateIndex(session);
    }

    @Override
    public CustomSession findById(String id) {
        byte[] data = storageClient.get(id);
        if (data == null) return null;

        CustomSession session = deserialize(data);
        if (session.isExpired()) {
            deleteById(id);
            return null;
        }
        return session;
    }

    @Override
    public void deleteById(String id) {
        storageClient.delete(id);
        cleanupIndex(id);
    }

    @Override
    public Map<String, CustomSession> findByIndexNameAndIndexValue(
            String indexName, String indexValue) {
        Set<String> ids = storageClient.getIndex(indexName, indexValue);
        Map<String, CustomSession> result = new HashMap<>();
        for (String id : ids) {
            CustomSession session = findById(id);
            if (session != null) {
                result.put(id, session);
            }
        }
        return result;
    }

    // 序列化/反序列化
    private byte[] serialize(CustomSession session) { /* ... */ }
    private CustomSession deserialize(byte[] data) { /* ... */ }
    private void updateIndex(CustomSession session) { /* ... */ }
    private void cleanupIndex(String id) { /* ... */ }
}

// Step 3: 注册为 Bean
@Configuration
public class CustomSessionConfig {

    @Bean
    public CustomSessionRepository sessionRepository(
            YourStorageClient storageClient) {
        return new CustomSessionRepository(storageClient);
    }
}
```

---

# 5. 自定义 Session 过期策略

## 5.1 分层过期策略

```java
public class TieredSessionRepository implements SessionRepository<Session> {

    // 核心思想：短期 Session（5分钟）用本地缓存，长期 Session（30分钟）用 Redis
    private final Cache<String, Session> localCache; // Caffeine
    private final SessionRepository<Session> remoteRepository; // Redis

    @Override
    public Session findById(String id) {
        // 先查本地缓存
        Session session = localCache.getIfPresent(id);
        if (session != null && !session.isExpired()) {
            return session;
        }

        // 再查远程存储
        session = remoteRepository.findById(id);
        if (session != null) {
            localCache.put(id, session);
        }
        return session;
    }

    @Override
    public void save(Session session) {
        remoteRepository.save(session);
        localCache.put(session.getId(), session);
    }

    @Override
    public void deleteById(String id) {
        remoteRepository.deleteById(id);
        localCache.invalidate(id);
    }
}
```

## 5.2 按用户类型设置不同过期时间

```java
@Component
public class DynamicSessionTimeoutManager {

    @Autowired
    private SessionRepository<?> sessionRepository;

    public void applyTimeout(String sessionId, String userType) {
        Session session = sessionRepository.findById(sessionId);
        if (session == null) return;

        switch (userType) {
            case "VIP":
                session.setMaxInactiveInterval(Duration.ofHours(2));
                break;
            case "NORMAL":
                session.setMaxInactiveInterval(Duration.ofMinutes(30));
                break;
            case "GUEST":
                session.setMaxInactiveInterval(Duration.ofMinutes(10));
                break;
        }
        sessionRepository.save(session);
    }
}
```

## 5.3 滑动窗口保持活跃

```java
@Component
public class KeepAliveInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            // 每次请求都重置最后访问时间
            // Spring Session 默认就会更新 lastAccessedTime
        }
        return true;
    }
}
```

---

# 6. Session 数据加密

## 6.1 透明加密包装器

```java
@Component
public class EncryptedSessionRepository
        implements FindByIndexNameSessionRepository<Session> {

    private final FindByIndexNameSessionRepository<?> delegate;
    private final EncryptionService encryptionService;

    @Override
    public Session createSession() {
        return new EncryptedSessionWrapper(
            delegate.createSession(),
            encryptionService
        );
    }

    // EncryptedSessionWrapper 在 setAttribute/getAttribute 时自动加解密
}

public class EncryptedSessionWrapper implements Session {

    private final Session delegate;
    private final EncryptionService encryptionService;

    @Override
    public <T> T getAttribute(String name) {
        byte[] encrypted = (byte[]) delegate.getAttribute(name);
        return (T) encryptionService.decrypt(encrypted);
    }

    @Override
    public void setAttribute(String name, Object value) {
        byte[] encrypted = encryptionService.encrypt(serialize(value));
        delegate.setAttribute(name, encrypted);
    }

    // 其余方法委托给 delegate
}
```

---

# 7. 自定义并发会话控制

```java
@Service
public class CustomSessionRegistry {

    @Autowired
    private FindByIndexNameSessionRepository<?> repository;

    /**
     * 踢出用户所有旧会话，保留最新的一个
     */
    public void keepOnlyLatestSession(String username, String newSessionId) {
        Map<String, ? extends Session> sessions = repository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );

        // 删除除最新外的所有 Session
        sessions.keySet().stream()
            .filter(id -> !id.equals(newSessionId))
            .forEach(repository::deleteById);
    }

    /**
     * 查询用户所有活跃会话数
     */
    public int countActiveSessions(String username) {
        Map<String, ? extends Session> sessions = repository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );
        return sessions.size();
    }

    /**
     * 强制注销所有会话
     */
    public void invalidateAllSessions(String username) {
        Map<String, ? extends Session> sessions = repository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );
        sessions.keySet().forEach(repository::deleteById);
    }
}
```
