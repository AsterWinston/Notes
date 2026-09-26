# 1. 会话事件体系

Spring Session 使用 Spring 的 `ApplicationEvent` 机制发布会话生命周期事件。通过监听这些事件，可以实现审计日志、实时通知、Session 预热等扩展功能。

## 1.1 事件类层次结构

```
ApplicationEvent
  └── SessionEvent
        ├── SessionCreatedEvent       (Session 创建)
        ├── SessionDestroyedEvent     (Session 销毁)
        │     └── SessionExpiredEvent (Session 过期 - 销毁的子类)
        └── SessionDeletedEvent       (Session 被删除)
```

## 1.2 事件说明

| 事件 | 触发时机 | 携带信息 |
|------|----------|----------|
| **SessionCreatedEvent** | 新 Session 创建时（`createSession()`） | Session ID |
| **SessionDestroyedEvent** | Session 销毁时（`invalidate()` 或过期） | Session ID |
| **SessionExpiredEvent** | Session 因超时而过期 | Session ID |
| **SessionDeletedEvent** | Session 被从存储中删除（`deleteById()`） | Session ID |

**注意**：
- `SessionExpiredEvent` 是 `SessionDestroyedEvent` 的子类
- `SessionDestroyedEvent` 和 `SessionDeletedEvent` 的区别：
  - `Destroyed` = Session 主动失效（`invalidate()` 或过期）
  - `Deleted` = Session 从存储中被删除（触发 `deleteById()`）

---

# 2. 事件监听器

## 2.1 基本监听器

```java
package com.example.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.session.events.*;
import org.springframework.stereotype.Component;

@Component
public class SessionEventListener {

    private static final Logger log =
        LoggerFactory.getLogger(SessionEventListener.class);

    @EventListener
    public void onSessionCreated(SessionCreatedEvent event) {
        log.info("Session 创建: sessionId={}",
            event.getSessionId());
        // 可以记录审计日志、发送通知等
    }

    @EventListener
    public void onSessionDestroyed(SessionDestroyedEvent event) {
        log.info("Session 销毁: sessionId={}",
            event.getSessionId());
        // 清理用户相关的临时资源
    }

    @EventListener
    public void onSessionExpired(SessionExpiredEvent event) {
        log.info("Session 过期: sessionId={}",
            event.getSessionId());
        // 发送"会话已过期"通知
    }

    @EventListener
    public void onSessionDeleted(SessionDeletedEvent event) {
        log.info("Session 从存储删除: sessionId={}",
            event.getSessionId());
    }
}
```

## 2.2 带 Session 属性的事件监听

当需要访问 Session 属性时：

```java
@Component
public class AdvancedSessionListener {

    @Autowired
    private FindByIndexNameSessionRepository<? extends Session> repository;

    @EventListener
    public void onSessionDestroyed(SessionDestroyedEvent event) {
        String sessionId = event.getSessionId();

        // 从存储中查找 Session（可能已被删除，需要判空）
        Session session = repository.findById(sessionId);
        if (session != null) {
            String username = session.getAttribute("username");
            log.info("用户 {} 的 Session ({}) 已销毁", username, sessionId);

            // 清理用户在线状态
            userOnlineCache.remove(username);
        }
    }
}
```

## 2.3 使用 @TransactionalEventListener

对于需要在事务提交后才执行的逻辑：

```java
@Component
public class TransactionalSessionListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onSessionCreated(SessionCreatedEvent event) {
        // 在事务提交后执行，如发送消息队列
        messageQueue.send(new SessionCreatedMessage(event.getSessionId()));
    }
}
```

---

# 3. 跨实例事件同步

## 3.1 问题背景

在分布式部署中，每个应用实例只能监听到自己 JVM 内的事件。当一个实例上的 Session 过期时，**其他实例无法感知**。

```
实例 A                          实例 B
  │                               │
  │ SessionExpiredEvent           │ (不知道这个事件)
  │                               │
  ▼                               ▼
└────────── Redis Pub/Sub ────────┘
          (事件传播通道)
```

## 3.2 Redis 事件传播实现

Spring Session Redis 通过 Redis Pub/Sub 实现跨实例事件传播：

```java
// 内部实现（简化）
public class RedisSessionExpirationPolicy {

    // 发布端：Session 事件发生时发送消息
    public void onExpirationUpdated(String key, Duration ttl) {
        redisTemplate.convertAndSend(
            SESSION_EXPIRED_CHANNEL,
            key.getBytes()
        );
    }

    // 订阅端：监听 Redis Channel
    @PostConstruct
    public void init() {
        redisMessageListenerContainer.addMessageListener(
            (message, pattern) -> {
                String sessionKey = new String(message.getBody());
                // 从 key 解析 sessionId
                String sessionId = extractSessionId(sessionKey);
                // 发布本地事件
                applicationEventPublisher.publishEvent(
                    new SessionExpiredEvent(sessionId));
            },
            new PatternTopic("__keyevent@*:expired")
        );
    }
}
```

**Redis 配置要求**：

```bash
# 必须启用 Redis 键空间通知
# redis.conf:
notify-keyspace-events Egx

# 或在 Spring Boot 中配置:
spring.session.redis.configure-action: notify_keyspace_events
```

## 3.3 事件订阅的局限性

| 限制 | 说明 |
|------|------|
| **不可靠** | Redis 不保证通知送达（Pub/Sub 是 fire-and-forget） |
| **延迟** | 通知有一定延迟，不能用于实时性要求极高的场景 |
| **网络依赖** | 网络故障时通知丢失 |
| **集群模式** | 集群模式下会广播到所有节点，可能产生重复 |

---

# 4. Session 过期处理机制

## 4.1 Redis 过期通知流程

```
1. Session expires key TTL 到期
       │
2. Redis 自动删除 key
       │
3. Redis 发布 __keyevent@*__:expired 通知
       │
       ├── 通知到达实例 A
       │     └── onMessage() → 发布 SessionExpiredEvent
       │
       └── 通知到达实例 B
             └── onMessage() → 发布 SessionExpiredEvent
       │
4. 各实例处理 SessionExpiredEvent
   ├── 清理本地的本地缓存
   └── 执行自定义监听器逻辑
```

## 4.2 JDBC 定时清理

JDBC 模式下没有 Redis 的过期通知机制，依赖定时任务：

```java
@Scheduled(cron = "0 * * * * *") // 每分钟执行一次
public void cleanUpExpiredSessions() {
    // 删除过期的 Session 记录
    jdbcOperations.update(
        "DELETE FROM SPRING_SESSION WHERE EXPIRY_TIME < ?",
        System.currentTimeMillis()
    );
}
```

---

# 5. Session 属性变更检测

## 5.1 变更追踪

Spring Session 内部维护了变更追踪机制：

```java
// RedisSession 内部实现（简化）
public class RedisSession implements Session {

    // 变更标记
    private boolean isNew;
    private boolean changed;
    private Set<String> changedKeys = new HashSet<>();
    private Set<String> deletedKeys = new HashSet<>();

    @Override
    public void setAttribute(String name, Object value) {
        sessionAttrs.put(name, value);
        changedKeys.add(name);
        deletedKeys.remove(name);
        changed = true;
    }

    @Override
    public void removeAttribute(String name) {
        sessionAttrs.remove(name);
        changedKeys.remove(name);
        deletedKeys.add(name);
        changed = true;
    }

    // 获取变更的属性（用于增量持久化）
    public Map<String, Object> getChangedAttributes() {
        Map<String, Object> delta = new HashMap<>();
        for (String key : changedKeys) {
            delta.put(key, sessionAttrs.get(key));
        }
        return delta;
    }
}
```

## 5.2 SaveMode 对变更检测的影响

```java
// ON_SET_ATTRIBUTE: 标记在 setAttribute() 时设置
// session.setAttribute("key", value); → changedKeys.add("key")

// ON_GET_ATTRIBUTE: 还需要跟踪获取属性的变更
// User user = session.getAttribute("user"); user.setName("new");
// → 反序列化时创建代理对象，跟踪内部变更

// ALWAYS: 不做变更检测，每次都全量保存
```

---

# 6. 实用场景

## 6.1 在线人数统计

```java
@Component
public class OnlineUserCounter {

    private final AtomicLong onlineCount = new AtomicLong(0);
    private final Set<String> activeSessionIds =
        ConcurrentHashMap.newKeySet();

    @EventListener
    public void onSessionCreated(SessionCreatedEvent event) {
        activeSessionIds.add(event.getSessionId());
        onlineCount.incrementAndGet();
    }

    @EventListener
    public void onSessionDestroyed(SessionDestroyedEvent event) {
        if (activeSessionIds.remove(event.getSessionId())) {
            onlineCount.decrementAndGet();
        }
    }

    public long getOnlineCount() {
        return onlineCount.get();
    }
}
```

## 6.2 强制下线通知

```java
@Component
public class ForceLogoutManager {

    private final SimpMessagingTemplate messagingTemplate;
    private final FindByIndexNameSessionRepository<?> repository;

    public void forceLogout(String username) {
        Map<String, ? extends Session> sessions = repository
            .findByIndexNameAndIndexValue(
                FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME,
                username
            );

        // 删除所有 Session
        sessions.keySet().forEach(repository::deleteById);

        // 通过 WebSocket 通知用户
        messagingTemplate.convertAndSendToUser(
            username, "/queue/logout", "您的账号已在其他设备登录");
    }
}
```

## 6.3 Session 审计日志

```java
@Component
public class SessionAuditLogger {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @EventListener
    public void onSessionCreated(SessionCreatedEvent event) {
        AuditLog log = AuditLog.builder()
            .action("SESSION_CREATED")
            .sessionId(event.getSessionId())
            .timestamp(Instant.now())
            .build();
        persist(log);
    }

    @EventListener
    public void onSessionDestroyed(SessionDestroyedEvent event) {
        AuditLog log = AuditLog.builder()
            .action("SESSION_DESTROYED")
            .sessionId(event.getSessionId())
            .timestamp(Instant.now())
            .build();
        persist(log);
    }

    private void persist(AuditLog log) {
        // 写入数据库或文件
    }
}
```

---

# 7. 自定义事件发布

## 7.1 扩展定义

```java
// 自定义事件
public class SessionAttributeChangedEvent extends ApplicationEvent {

    private final String sessionId;
    private final String attributeName;
    private final Object oldValue;
    private final Object newValue;

    public SessionAttributeChangedEvent(Object source, String sessionId,
            String attributeName, Object oldValue, Object newValue) {
        super(source);
        this.sessionId = sessionId;
        this.attributeName = attributeName;
        this.oldValue = oldValue;
        this.newValue = newValue;
    }

    // getters
}
```

## 7.2 触发自定义事件

```java
@Component
public class SessionAttributeChangeNotifier {

    private final ApplicationEventPublisher eventPublisher;

    // 在 Session 属性变更时发布自定义事件
    public void notifyAttributeChange(String sessionId, String attrName,
                                       Object oldValue, Object newValue) {
        eventPublisher.publishEvent(
            new SessionAttributeChangedEvent(
                this, sessionId, attrName, oldValue, newValue)
        );
    }
}
```
