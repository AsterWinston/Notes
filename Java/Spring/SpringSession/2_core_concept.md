# 1. Session 与 HttpSession

## 1.1 Session 的概念

**Session（会话）** 是 Web 应用中服务器端维护的用户状态容器。由于 HTTP 协议是无状态的，服务器需要一种机制来识别同一个用户的多次请求，Session 就是这种机制的载体。

```
客户端                          服务端
  │                               │
  │  ① 首次请求 (无Cookie)         │
  │ ──────────────────────────►   │
  │                               │ 创建 Session
  │  ② 响应 Set-Cookie: JSESSIONID=ABC123  │
  │ ◄──────────────────────────   │
  │                               │
  │  ③ 后续请求 Cookie: JSESSIONID=ABC123  │
  │ ──────────────────────────►   │
  │                               │ 通过 ABC123 查找 Session
  │  ④ 响应                       │
  │ ◄──────────────────────────   │
```

## 1.2 Servlet HttpSession 接口

```java
public interface HttpSession {
    long getCreationTime();          // 创建时间
    String getId();                  // Session ID
    long getLastAccessedTime();      // 最后访问时间
    ServletContext getServletContext();
    void setMaxInactiveInterval(int interval); // 最大非活动时间（秒）
    int getMaxInactiveInterval();
    Object getAttribute(String name);
    Enumeration<String> getAttributeNames();
    void setAttribute(String name, Object value);
    void removeAttribute(String name);
    void invalidate();               // 立即失效
    boolean isNew();                 // 是否新创建的
}
```

---

# 2. Spring Session 核心接口

## 2.1 Session 接口

Spring Session 定义了自己的 `Session` 接口，是对 `HttpSession` 的抽象扩展：

```java
package org.springframework.session;

public interface Session {
    // 获取唯一的 Session ID
    String getId();

    // 获取此次 Session 的变更 ID（用于处理并发修改）
    String changeSessionId();

    // 获取/设置属性
    <T> T getAttribute(String attributeName);
    Set<String> getAttributeNames();
    void setAttribute(String attributeName, Object attributeValue);
    void removeAttribute(String attributeName);

    // 获取创建时间（Instant）
    Instant getCreationTime();

    // 获取最后访问时间
    Instant getLastAccessedTime();

    // 获取/设置最大非活动间隔
    Duration getMaxInactiveInterval();
    void setMaxInactiveInterval(Duration maxInactiveInterval);

    // 是否过期
    boolean isExpired();
}
```

**关键设计要点**：

| 设计点 | 说明 |
|--------|------|
| **使用 Instant** | 时间使用 `java.time.Instant`，比 `long` 更清晰、类型安全 |
| **使用 Duration** | 间隔使用 `java.time.Duration`，避免秒/毫秒混淆 |
| **changeSessionId()** | 用于防止 Session Fixation 攻击，每个 Session 对象有一个变更记录 |
| **不依赖 Servlet API** | 接口与 Servlet API 完全解耦，可在非 Servlet 环境（如 WebFlux）使用 |

## 2.2 SessionRepository 接口

`SessionRepository` 是 Spring Session 最核心的接口，定义了 Session 的存储操作：

```java
package org.springframework.session;

public interface SessionRepository<S extends Session> {

    // 创建一个新的 Session（带默认属性）
    S createSession();

    // 保存 Session 到存储
    void save(S session);

    // 根据 ID 查找 Session
    S findById(String id);

    // 根据 ID 删除 Session
    void deleteById(String id);
}
```

**设计意图**：

```
SessionRepository<S extends Session>
  ├── MapSessionRepository        (内存实现，适合测试)
  ├── RedisSessionRepository      (Redis 实现)
  ├── JdbcSessionRepository       (JDBC 实现)
  └── HazelcastSessionRepository  (Hazelcast 实现)
```

这是一个典型的**仓储模式 (Repository Pattern)**，将 Session 的存储逻辑与业务逻辑解耦。更换后端存储只需要替换 `SessionRepository` 的实现即可。

## 2.3 FindByIndexNameSessionRepository 接口

按**索引名称**查找 Session 的扩展接口，常用于"查找某用户的所有活跃 Session"：

```java
package org.springframework.session;

public interface FindByIndexNameSessionRepository<S extends Session>
        extends SessionRepository<S> {

    // Spring Security 用于索引用户名的常量
    String PRINCIPAL_NAME_INDEX_NAME = "PRINCIPAL_NAME_INDEX_NAME";

    // 根据索引名和索引值查找所有 Session
    Map<String, ? extends S> findByIndexNameAndIndexValue(
            String indexName, String indexValue);
}
```

**使用场景**：

```java
// 查找用户 "zhangsan" 的所有活跃 Session
Map<String, ? extends Session> sessions = repository
    .findByIndexNameAndIndexValue(
        FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
        "zhangsan"
    );

// 强制所有 Session 下线
sessions.values().forEach(s -> repository.deleteById(s.getId()));
```

**实现原理**：

- 在 `save()` 时，提取 Session 中的 `PRINCIPAL_NAME_INDEX_NAME` 属性值
- 将其作为索引写入存储（如 Redis 的 `SET` 数据结构）
- 查找时通过索引快速定位对应的 Session ID

## 2.4 SessionIdResolver 接口

`HttpSessionIdResolver` 接口负责从 HTTP 请求/响应中解析和写入 Session ID：

```java
package org.springframework.session.web.http;

public interface HttpSessionIdResolver {
    // 从请求中解析 Session IDs（可能返回多个）
    List<String> resolveSessionIds(HttpServletRequest request);

    // 将 Session IDs 写入响应
    void setSessionId(HttpServletRequest request, HttpServletResponse response,
                      String sessionId);

    // 使响应中的 Session ID 过期
    void expireSession(HttpServletRequest request, HttpServletResponse response);
}
```

**默认实现**：

| 实现类 | 方式 | 适用场景 |
|--------|------|----------|
| **CookieHttpSessionIdResolver** | Cookie 中的 `SESSION` | 传统 Web 应用（默认） |
| **HeaderHttpSessionIdResolver** | 请求头 `X-Auth-Token` | 前后端分离、REST API |

**Cookie 方式**：

```
请求：Cookie: SESSION=abc123-def456-ghi789
响应：Set-Cookie: SESSION=abc123-def456-ghi789; Path=/; HttpOnly
```

**Header 方式**：

```
请求：X-Auth-Token: abc123-def456-ghi789
响应：X-Auth-Token: abc123-def456-ghi789
```

---

# 3. Session 数据模型

## 3.1 MapSession

`MapSession` 是 `Session` 接口的基础实现，内部使用 `ConcurrentHashMap` 存储属性：

```java
public final class MapSession implements Session, Serializable {

    private String id;                           // Session ID
    private final String originalSessionId;      // 原始 Session ID
    private final Map<String, Object> sessionAttrs = new ConcurrentHashMap<>();
    private Instant creationTime;                // 创建时间
    private Instant lastAccessedTime;           // 最后访问时间
    private Duration maxInactiveInterval;        // 最大非活动间隔

    // ...
}
```

**特点**：
- 线程安全（使用 `ConcurrentHashMap`）
- 支持序列化
- 适合单机测试或小规模使用

## 3.2 RedisSession

`RedisSession` 是 Redis 后端的 Session 实现，具有以下特点：

```
RedisSession 数据结构：
  ┌────────────────────────────────────────────┐
  │ Key: "spring:session:sessions:{sessionId}" │
  │ Type: HASH                                 │
  │ Fields:                                    │
  │   creationTime: 1700000000000              │
  │   lastAccessedTime: 1700000010000          │
  │   maxInactiveInterval: 1800                │
  │   sessionAttr:{attrName}: {value}          │
  │                                            │
  │ Key: "spring:session:sessions:expires:{sessionId}" │
  │ Type: STRING (TTL 控制)                    │
  │                                            │
  │ Key: "spring:session:index:{indexName}:{indexValue}" │
  │ Type: SET (索引)                           │
  │ Members: {sessionId1}, {sessionId2}        │
  └────────────────────────────────────────────┘
```

---

# 4. Session 配置模型

## 4.1 Cookie 配置

```java
// application.yml
server:
  servlet:
    session:
      cookie:
        name: MYSESSION            # Cookie 名称（默认 SESSION）
        domain: example.com        # Cookie 作用域
        path: /                    # Cookie 路径
        http-only: true           # 禁止 JavaScript 访问
        secure: true              # 仅 HTTPS 传输
        same-site: Lax            # SameSite 策略
        max-age: 30m              # Cookie 有效期
```

## 4.2 Session 配置

```java
// application.yml
spring:
  session:
    timeout: 30m                  # Session 超时时间
    redis:
      namespace: spring:session   # Redis key 命名空间
      flush-mode: on_save        # Redis 刷新模式 (on_save / immediate)
      save-mode: on_set_attribute # 保存模式 (always / on_set_attribute / on_get_attribute)
    jdbc:
      initialize-schema: embedded # 初始化表结构 (always / embedded / never)
      table-name: SPRING_SESSION  # 表名
      cleanup-cron: 0 * * * * *   # 定时清理过期 Session (cron)
```

---

# 5. Session 属性与命名空间

## 5.1 属性存储

Session 中的属性必须是**可序列化的**：

```java
// 存储简单类型
session.setAttribute("userId", 12345L);
session.setAttribute("username", "zhangsan");

// 存储复杂对象（必须实现 Serializable）
session.setAttribute("userInfo", new UserInfo("zhangsan", "zhangsan@example.com"));

// 通过 Spring Session 方式
Session session = sessionRepository.findById(sessionId);
session.setAttribute("loginTime", Instant.now());
sessionRepository.save(session);
```

## 5.2 命名空间约定

为避免属性名冲突，建议使用命名空间前缀：

```java
// 推荐：使用前缀避免冲突
session.setAttribute("app:user:id", userId);
session.setAttribute("app:user:name", username);
session.setAttribute("app:cart:items", cartItems);
session.setAttribute("app:locale", locale);
```

## 5.3 索引属性

索引属性用于快速查找 Session，需要在 Session 中设置时带上 `FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME`：

```java
// Spring Security 自动设置
session.setAttribute("SPRING_SECURITY_CONTEXT", securityContext);
// 同时设置索引
session.setAttribute("PRINCIPAL_NAME_INDEX_NAME", principalName);
```

---

# 6. Session 刷新策略

## 6.1 FlushMode（刷新模式）

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **ON_SAVE** | 仅在显式调用 `save()` 时持久化到存储（默认） | 大多数场景，减少 Redis 写入次数 |
| **IMMEDIATE** | 每次 `setAttribute()` 立即持久化 | 对一致性要求极高的场景 |

## 6.2 SaveMode（保存模式）

| 模式 | 说明 |
|------|------|
| **ON_SET_ATTRIBUTE** | 仅在 `setAttribute()` 时标记脏数据并保存（默认） |
| **ON_GET_ATTRIBUTE** | `getAttribute()` 时也检查变更（如获取可变对象并修改） |
| **ALWAYS** | 每次都保存全部 Session 数据 |

```java
// 配置示例
@Configuration
@EnableRedisHttpSession(flushMode = FlushMode.IMMEDIATE,
                        saveMode = SaveMode.ON_SET_ATTRIBUTE)
public class SessionConfig {
    // ...
}
```
