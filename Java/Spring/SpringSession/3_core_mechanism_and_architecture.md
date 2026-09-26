# 1. 整体架构

Spring Session 的整体架构遵循**分层设计**原则，从上到下依次为：

```
┌──────────────────────────────────────────────────────────┐
│                     使用方                                 │
│  HttpSession API  │  Session API  │  WebSession (WebFlux) │
├──────────────────────────────────────────────────────────┤
│                     适配层                                 │
│  SessionRepositoryFilter  │  WebSessionManagerFilter      │
│  (Servlet)                 │  (WebFlux)                    │
├──────────────────────────────────────────────────────────┤
│                     核心层                                 │
│  SessionRepository  │  SessionIdResolver                  │
│  Session 接口       │  SessionEvent                       │
├──────────────────────────────────────────────────────────┤
│                     存储层                                 │
│  Redis  │  JDBC  │  Hazelcast  │  MongoDB  │  HashMap    │
└──────────────────────────────────────────────────────────┘
```

---

# 2. SessionRepositoryFilter — 核心入口

## 2.1 设计动机

`SessionRepositoryFilter` 是整个 Spring Session 的**核心入口 Filter**，它的核心作用是**替换原生的 HttpSession**。

```
请求到达 Tomcat
  │
  ▼
SessionRepositoryFilter.doFilter()
  │
  ├── 1. 包装 HttpServletRequest  → SessionRepositoryRequestWrapper
  │     └── 重写 getSession() 方法
  │
  ├── 2. 包装 HttpServletResponse → SessionRepositoryResponseWrapper
  │     └── 重写 encodeRedirectURL()、onResponseCommitted() 等
  │
  ├── 3. 调用 filterChain.doFilter(wrappedReq, wrappedRes)
  │     └── 后续所有组件拿到的都是包装后的 HttpSession
  │
  └── 4. 响应提交时
        ├── 持久化 Session 到存储
        └── 写入 Session ID Cookie（新创建的）
```

## 2.2 请求包装类 — SessionRepositoryRequestWrapper

```java
// 简化版源码结构
private final class SessionRepositoryRequestWrapper
        extends HttpServletRequestWrapper {

    private HttpSessionWrapper requestedSession;
    private boolean requestedSessionCached = false;
    private String requestedSessionId;

    @Override
    public HttpSession getSession(boolean create) {
        if (requestedSessionCached) {
            return requestedSession;
        }

        // 1. 解析请求中的 Session ID
        List<String> sessionIds = httpSessionIdResolver.resolveSessionIds(this);

        // 2. 尝试从 Repository 加载已存在的 Session
        for (String sessionId : sessionIds) {
            S session = sessionRepository.findById(sessionId);
            if (session != null && !session.isExpired()) {
                this.requestedSession = new HttpSessionWrapper(session, getServletContext());
                break;
            }
        }

        // 3. 如果不存在且 create=true，创建新 Session
        if (this.requestedSession == null && create) {
            S session = sessionRepository.createSession();
            this.requestedSession = new HttpSessionWrapper(session, getServletContext());
        }

        this.requestedSessionCached = true;
        return this.requestedSession;
    }

    @Override
    public String changeSessionId() {
        // 防止 Session Fixation，替换 Session ID
        String newId = this.requestedSession.getSession().changeSessionId();
        return newId;
    }

    // ...
}
```

**关键设计**：

| 设计点 | 说明 |
|--------|------|
| **延迟加载** | Session 仅在首次 `getSession()` 时从存储加载 |
| **缓存机制** | 在同一请求中多次调用 `getSession()` 返回同一实例 |
| **Session Fixation 防护** | `changeSessionId()` 逻辑与存储交互 |
| **透明性** | 业务代码调用 `request.getSession()` 无感知 |

## 2.3 响应包装类 — SessionRepositoryResponseWrapper

```java
private final class SessionRepositoryResponseWrapper
        extends OnCommittedResponseWrapper {

    @Override
    protected void onResponseCommitted() {
        // 响应提交时的核心逻辑
        HttpSessionWrapper session = getRequestedSession();

        if (session != null) {
            if (session.isNew()) {
                // 新 Session：写入 Cookie
                httpSessionIdResolver.setSessionId(request, this,
                    session.getId());
            }
            if (session.isInvalidated()) {
                // 已失效：删除 Cookie
                httpSessionIdResolver.expireSession(request, this);
            }
            // 持久化 Session 到存储
            sessionRepository.save(session.getSession());
        }
    }
}
```

**执行流程**：

```
请求处理完成，准备写入响应
  │
  ▼
onResponseCommitted()
  │
  ├── Session 是新的？
  │     └── Yes → 写入 Session ID Cookie
  │
  ├── Session 被 invalidate 了？
  │     └── Yes → 清除 Cookie，删除存储中的 Session
  │
  └── 持久化 Session 变更到存储
        └── sessionRepository.save(session)
```

## 2.4 过滤器顺序

Spring Session 的 Filter 注册顺序非常重要，通常需要**排在 Spring Security 过滤器链之后**：

```java
// Spring Boot 自动配置的默认顺序
// 可以通过 spring.session.servlet.filter-order 调整
spring:
  session:
    servlet:
      filter-order: -2147483598   # 默认值 AbstractRequestDispatcherInitializer.DEFAULT_FILTER_ORDER
```

---

# 3. SessionRepository — 存储抽象层

## 3.1 核心类图

```
                    «interface»
                    SessionRepository<S>
                    ├── createSession(): S
                    ├── save(S session)
                    ├── findById(String): S
                    └── deleteById(String)

                            △
                            │ implements
        ┌───────────────────┼───────────────────────┐
        │                   │                       │
MapSessionRepository  RedisSessionRepository  JdbcSessionRepository
(内存，测试用)       (Redis 生产用)          (JDBC 生产用)

┌──────────────────────────────┐
│ «interface»                  │
│ FindByIndexNameSessionRepository │
│ └── findByIndexNameAndIndexValue()  │
└──────────────────────────────┘
```

## 3.2 存储操作流程

### 3.2.1 创建 Session

```
createSession()
  │
  ├── 1. 生成 UUID 作为 Session ID
  │     └── UUID.randomUUID().toString()
  │
  ├── 2. 初始化 Session 属性
  │     ├── creationTime = Instant.now()
  │     ├── lastAccessedTime = Instant.now()
  │     ├── maxInactiveInterval = Duration.ofMinutes(30)
  │     └── sessionAttrs = new HashMap<>()
  │
  └── 3. 返回 Session 对象（此时尚未持久化）
        └── 在 onResponseCommitted() 时才 save()
```

### 3.2.2 保存 Session

```
save(session)
  │
  ├── Redis 实现:
  │   ├── 将 Session 属性序列化为 Redis HASH
  │   ├── 设置 TTL = maxInactiveInterval
  │   ├── 更新索引（如 PRINCIPAL_NAME_INDEX_NAME）
  │   └── 发布 Session 变更事件（Pub/Sub）
  │
  ├── JDBC 实现:
  │   ├── INSERT INTO SPRING_SESSION ... ON DUPLICATE KEY UPDATE
  │   └── 更新 SPRING_SESSION_ATTRIBUTES 表
  │
  └── Hazelcast 实现:
      ├── 写入分布式 Map
      └── 设置 TTL
```

### 3.2.3 查找 Session

```
findById(sessionId)
  │
  ├── 从存储中查询
  │   ├── Redis: HGETALL spring:session:sessions:{id}
  │   ├── JDBC: SELECT * FROM SPRING_SESSION WHERE SESSION_ID = ?
  │   └── Hazelcast: IMap.get(sessionId)
  │
  ├── 检查是否过期
  │   ├── isExpired() → 超过 maxInactiveInterval 未活动
  │   └── 若过期 → 删除并返回 null
  │
  └── 返回 Session 对象
```

---

# 4. 请求-响应完整生命周期

## 4.1 首次请求（创建 Session）

```
1. 浏览器 → GET /login → 服务器
2. SessionRepositoryFilter
   ├── resolveSessionIds(request) → [空]（无 Cookie）
   └── getSession() → 不创建（因为是 GET 且未调用 getSession(true)）
3. POST /login → 服务器
4. SessionRepositoryFilter
   ├── resolveSessionIds(request) → [空]
   └── getSession(true) → 创建新 Session + changeSessionId()（防 Fixation）
5. 认证成功
6. onResponseCommitted()
   ├── 持久化 Session 到 Redis/DB
   └── Set-Cookie: SESSION=abc123
```

## 4.2 后续请求（使用已有 Session）

```
1. 浏览器 → GET /home (Cookie: SESSION=abc123) → 服务器
2. SessionRepositoryFilter
   ├── resolveSessionIds(request) → ["abc123"]
   ├── findById("abc123") → Redis/DB 查询
   └── 创建 HttpSessionWrapper 包装查询到的 Session
3. Controller 处理
4. onResponseCommitted()
   └── 保存 Session（更新 lastAccessedTime）
```

## 4.3 Session 过期处理

```
┌─────────────────────────────────────────────┐
│ Redis: 通过 Key 的 TTL 自动过期              │
│ 过期后 Redis 自动删除 key                    │
│ 可通过键空间通知 (Keyspace Notification) 感知 │
│                                             │
│ JDBC: 通过定时任务清理                        │
│ @Scheduled(cron = "0 * * * * *")            │
│ DELETE FROM SPRING_SESSION WHERE             │
│   EXPIRY_TIME < ?                            │
└─────────────────────────────────────────────┘
```

---

# 5. 集群事件传播机制

## 5.1 为什么需要事件传播

在分布式部署中，同一个用户可能被路由到不同实例。当某个实例上的 Session 变更（创建/删除/过期）时，其他实例需要感知：

```
实例 A                              实例 B
  │                                   │
  │  用户登录，创建 Session            │
  │  UserA → session:abc123           │
  │                                   │
  ├── 发布 Session 创建事件 ──────────►│
  │     (Redis Pub/Sub)               │ 收到事件
  │                                   │ 更新本地 Session 缓存
  │                                   │
  │  用户访问实例 B                    │
  │                                   ├── 正确的 Session 状态
```

## 5.2 Redis 事件传播

```java
// Spring Session 通过 Redis Pub/Sub 实现事件传播
// 每个实例启动时订阅指定 Channel

// 发布端
@EventListener
public void onSessionCreated(SessionCreatedEvent event) {
    redisTemplate.convertAndSend(
        "spring:session:event:created",
        event.getSessionId()
    );
}

// 订阅端
// RedisMessageListenerContainer 监听指定 Channel
```

**事件类型与 Channel 映射**：

| 事件 | Channel |
|------|---------|
| `SessionCreatedEvent` | `spring:session:event:{namespace}:created` |
| `SessionDeletedEvent` | `spring:session:event:{namespace}:deleted` |
| `SessionExpiredEvent` | `spring:session:event:{namespace}:expired` |

## 5.3 Hazelcast 事件传播

Hazelcast 通过其原生的分布式 Topic 机制实现事件传播，无需额外配置 Pub/Sub。

---

# 6. 与 Spring Security 的协作机制

## 6.1 SecurityContext 持久化

```
认证成功
  │
  ├── Spring Security 创建 Authentication
  │   └── UsernamePasswordAuthenticationToken
  │
  ├── SecurityContextPersistenceFilter
  │   └── 将 SecurityContext 设置到 SecurityContextHolder
  │
  └── onResponseCommitted()
      ├── SecurityContextPersistenceFilter 将 SecurityContext
      │   存入 HttpSession:
      │   session.setAttribute("SPRING_SECURITY_CONTEXT", context)
      │
      └── SessionRepositoryFilter 将 Session 持久化到 Redis/DB
```

## 6.2 并发会话控制

```java
// Spring Security 的 SessionRegistry 依赖 Spring Session
@Bean
public SessionRegistry sessionRegistry(
        FindByIndexNameSessionRepository<?> sessionRepository) {
    return new SpringSessionBackedSessionRegistry<>(sessionRepository);
}

// 使用示例
http.sessionManagement(session -> session
    .maximumSessions(1)          // 同一用户最多 1 个会话
    .maxSessionsPreventsLogin(false) // 超限时踢掉最旧的会话
    .sessionRegistry(sessionRegistry())
);
```

## 6.3 跨服务 Session 共享

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  认证服务  │     │  订单服务  │     │  商品服务  │
│ (8080)   │     │ (8081)   │     │ (8082)   │
└─────┬─────┘     └─────┬─────┘     └─────┬─────┘
      │                 │                 │
      └────────┬────────┴────────┬────────┘
               │                 │
          ┌────▼─────────────────▼────┐
          │     Redis (共享Session)    │
          └───────────────────────────┘
```

所有服务共享同一个 Redis，认证服务写入的 Session，其他服务可直接读取。
