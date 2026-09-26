# 1. Redis 作为 Session 存储的优势

| 优势 | 说明 |
|------|------|
| **高性能** | 纯内存操作，读写延迟通常在 1ms 以下 |
| **高可用** | 支持主从复制、哨兵模式、集群模式 |
| **数据持久化** | 支持 RDB 快照和 AOF 日志，防止数据丢失 |
| **丰富的结构** | HASH 结构天然适合 Session 属性存储 |
| **TTL 支持** | Key 级别 TTL 自动过期，完美匹配 Session 生命周期 |
| **Pub/Sub** | 内置发布订阅，用于集群事件传播 |

---

# 2. Redis 中 Session 的数据结构

## 2.1 核心 Key 结构

```
命名空间: spring:session  (可通过 spring.session.redis.namespace 配置)

Key 1: "spring:session:sessions:{sessionId}"
  类型: HASH
  作用: 存储 Session 的元数据和属性

Key 2: "spring:session:sessions:expires:{sessionId}"
  类型: STRING
  作用: 通过 TTL 控制 Session 过期（标记用）

Key 3: "spring:session:index:{indexName}:{indexValue}"
  类型: SET
  作用: 索引（如按用户名查找 Session）
```

## 2.2 HASH 数据结构详解

**Key**: `spring:session:sessions:abc123-def456`

| Field | Value | 说明 |
|-------|-------|------|
| `creationTime` | `1700000000000` | 创建时间（epoch 毫秒） |
| `lastAccessedTime` | `1700000100000` | 最后访问时间 |
| `maxInactiveInterval` | `1800` | 最大非活动间隔（秒） |
| `sessionAttr:{name}` | 序列化后的属性值 | Session 属性 |

**示例**：

```bash
redis-cli HGETALL "spring:session:sessions:abc123-def456"

# 输出：
# creationTime → "1700000000000"
# lastAccessedTime → "1700000100000"
# maxInactiveInterval → "1800"
# sessionAttr:SPRING_SECURITY_CONTEXT → "...序列化数据..."
# sessionAttr:username → "...序列化数据..."
```

## 2.3 过期 Key 结构

**Key**: `spring:session:sessions:expires:abc123-def456`
**Value**: "" (空字符串)
**TTL**: `maxInactiveInterval` 秒

**设计原理**：

Redis 的 HASH 结构**不支持**对单个 Field 设置 TTL，无法直接通过 HASH Key 的 TTL 实现 Session 过期。因此 Spring Session 采用**专用过期 Key** 的方式：

```
┌──────────────────────────────────────────────────┐
│ sessions:{id} (HASH)                             │
│   - creationTime: xxx                            │
│   - lastAccessedTime: xxx                        │
│   - ...                                          │
│ 无 TTL（或比 expires 长）                         │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ sessions:expires:{id} (STRING)                   │
│ Value: ""                                        │
│ TTL: maxInactiveInterval 秒                      │
│ 过期时触发 Redis 键空间通知                       │
└──────────────────────────────────────────────────┘
```

**过期处理流程**：

```
expires key 过期
  │
  ├── Redis 自动删除 "sessions:expires:{id}"
  │
  ├── 可选：触发 __keyevent@*__:expired 通知
  │     └── Spring Session 监听到通知
  │          ├── 删除 sessions:{id} (HASH)
  │          ├── 清理索引数据
  │          └── 发布 SessionExpiredEvent
  │
  └── 无通知时：需额外处理 sessions:{id} 的清理
```

## 2.4 索引 Key 结构

**Key**: `spring:session:index:PRINCIPAL_NAME_INDEX_NAME:zhangsan`
**类型**: SET
**Members**: `["abc123", "def456"]` (该用户的所有 Session ID)

```bash
# 查看用户的所有 Session
redis-cli SMEMBERS "spring:session:index:PRINCIPAL_NAME_INDEX_NAME:zhangsan"
```

---

# 3. RedisSessionRepository 源码分析

## 3.1 类结构

```java
package org.springframework.session.data.redis;

public class RedisSessionRepository
        implements FindByIndexNameSessionRepository<RedisSession> {

    private final RedisOperations<String, Object> redisOperations;
    private final RedisSessionExpirationPolicy expirationPolicy;
    private String namespace = "spring:session";
    private Duration defaultMaxInactiveInterval = Duration.ofMinutes(30);
    private FlushMode flushMode = FlushMode.ON_SAVE;
    private SaveMode saveMode = SaveMode.ON_SET_ATTRIBUTE;
    // ...
}
```

## 3.2 createSession()

```java
// 简化版源码
@Override
public RedisSession createSession() {
    RedisSession session = new RedisSession();
    session.setMaxInactiveInterval(defaultMaxInactiveInterval);
    return session;
}
```

## 3.3 save() — 核心保存逻辑

```java
// 简化版源码
@Override
public void save(RedisSession session) {
    if (!session.isNew() && !session.isChanged()) {
        return; // 没有变更，不保存
    }

    String key = namespace + ":sessions:" + session.getId();
    Map<String, Object> map = new HashMap<>();

    // 1. 保存元数据
    map.put("creationTime", session.getCreationTime().toEpochMilli());
    map.put("lastAccessedTime", session.getLastAccessedTime().toEpochMilli());
    map.put("maxInactiveInterval", session.getMaxInactiveInterval().getSeconds());

    // 2. 保存属性（delta 变更）
    // saveMode = ON_SET_ATTRIBUTE: 只保存变更的属性
    // saveMode = ALWAYS: 保存所有属性
    for (Entry<String, Object> attr : session.getChangedAttributes().entrySet()) {
        map.put("sessionAttr:" + attr.getKey(), attr.getValue());
    }

    // 3. 持久化到 Redis（HASH）
    redisOperations.opsForHash().putAll(key, map);

    // 4. 设置过期 Key
    String expireKey = namespace + ":sessions:expires:" + session.getId();
    redisOperations.opsForValue().set(expireKey, "");
    redisOperations.expire(expireKey, session.getMaxInactiveInterval());

    // 5. 更新索引
    updateIndex(session);

    // 6. 清除变更标记
    session.clearChangeFlags();
}
```

## 3.4 findById()

```java
// 简化版源码
@Override
public RedisSession findById(String id) {
    String key = namespace + ":sessions:" + id;
    Map<Object, Object> map = redisOperations.opsForHash().entries(key);

    if (map.isEmpty()) {
        return null; // Session 不存在
    }

    // 从 Redis HASH 反序列化构建 RedisSession
    RedisSession session = new RedisSession(
        id,
        map,
        defaultMaxInactiveInterval
    );

    if (session.isExpired()) {
        deleteById(id);
        return null;
    }

    return session;
}
```

## 3.5 deleteById()

```java
// 简化版源码
@Override
public void deleteById(String id) {
    String key = namespace + ":sessions:" + id;
    String expireKey = namespace + ":sessions:expires:" + id;

    // 1. 删除 HASH
    redisOperations.delete(key);

    // 2. 删除过期 Key
    redisOperations.delete(expireKey);

    // 3. 清理索引
    // 从所有索引集合中移除此 Session ID
    cleanupIndex(id);
}
```

---

# 4. Redis 序列化配置

## 4.1 默认序列化器

Spring Session 默认使用 **JDK 序列化** (`JdkSerializationRedisSerializer`)：

```java
// 默认行为
// 序列化后的数据不可读（二进制），类型安全但跨语言支持差
```

## 4.2 自定义为 JSON 序列化

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;

@Configuration
public class SessionRedisConfig {

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        // 使用 Jackson JSON 序列化
        GenericJackson2JsonRedisSerializer serializer =
            new GenericJackson2JsonRedisSerializer();

        // 可选：自定义 ObjectMapper
        // ObjectMapper mapper = new ObjectMapper();
        // mapper.registerModule(new JavaTimeModule());
        // mapper.activateDefaultTyping(
        //     BasicPolymorphicTypeValidator.builder().build(),
        //     ObjectMapper.DefaultTyping.NON_FINAL
        // );
        // serializer = new GenericJackson2JsonRedisSerializer(mapper);

        return serializer;
    }
}
```

**JSON 序列化的优缺点**：

| 优点 | 缺点 |
|------|------|
| 可读性好，方便调试 | 性能略低于 JDK 序列化 |
| 跨语言支持 | 需要处理类型安全（Jackson default typing） |
| 无需 `Serializable` 接口 | 复杂对象可能反序列化失败 |

## 4.3 序列化性能对比

| 序列化方式 | 大小（1000字节对象） | 速度 | 可读性 | 跨语言 |
|------------|---------------------|------|--------|--------|
| JDK | ~1200 bytes | 中 | 不可读 | 不支持 |
| Jackson JSON | ~800 bytes | 中 | 可读 | 支持 |
| Protobuf | ~300 bytes | 快 | 不可读 | 支持 |
| Kryo | ~400 bytes | 快 | 不可读 | 不支持 |

---

# 5. Redis 连接配置

## 5.1 单机模式

```yaml
spring:
  data:
    redis:
      host: 192.168.1.100
      port: 6379
      password: your-password
      database: 2
      timeout: 5000ms
      client-type: lettuce       # lettuce 或 jedis
      lettuce:
        pool:
          max-active: 16         # 最大连接数
          max-idle: 8            # 最大空闲连接
          min-idle: 4            # 最小空闲连接
          max-wait: 2000ms       # 获取连接最大等待时间
```

## 5.2 哨兵模式

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - 192.168.1.101:26379
          - 192.168.1.102:26379
          - 192.168.1.103:26379
      password: your-password
```

## 5.3 集群模式

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - 192.168.1.101:6379
          - 192.168.1.102:6379
          - 192.168.1.103:6379
          - 192.168.1.104:6379
          - 192.168.1.105:6379
          - 192.168.1.106:6379
        max-redirects: 3
```

## 5.4 SSL/TLS 加密连接

```yaml
spring:
  data:
    redis:
      ssl:
        enabled: true
```

---

# 6. 高级特性

## 6.1 自定义 RedisOperations

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;

@Configuration
public class CustomRedisConfig {

    @Bean
    public RedisTemplate<String, Object> sessionRedisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // Key 使用字符串序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value 使用 JSON 序列化
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());

        template.afterPropertiesSet();
        return template;
    }
}
```

## 6.2 Redis 键空间通知

启用 Redis 键空间通知以感知 Session 过期事件：

```bash
# Redis 配置
notify-keyspace-events Egx
```

```yaml
# application.yml
spring:
  session:
    redis:
      configure-action: notify_keyspace_events  # Spring Boot 3.x
```

注意：键空间通知是**不可靠的**（Redis 不保证通知一定能送达），不能完全依赖它来触发 Session 清理逻辑。

## 6.3 定时清理

即使有键空间通知，也建议配置定时清理任务：

```java
@Component
public class SessionCleanupTask {

    private static final Logger log =
        LoggerFactory.getLogger(SessionCleanupTask.class);

    @Autowired
    private RedisSessionRepository sessionRepository;

    @Scheduled(fixedRate = 300000) // 每 5 分钟
    public void cleanExpiredSessions() {
        // 遍历过期 Session 并清理
        // 实际实现依赖于存储后端的能力
        log.debug("Session cleanup task executed");
    }
}
```

---

# 7. Redis 运维要点

## 7.1 监控指标

```bash
# Session 相关 Key 数量
redis-cli DBSIZE

# 内存使用
redis-cli INFO memory

# 查看所有 Session key
redis-cli --scan --pattern "spring:session:*"

# 统计各类型 Key 数量
redis-cli KEYS "spring:session:sessions:*" | wc -l      # Session 数量
redis-cli KEYS "spring:session:expirations:*" | wc -l   # 过期 Key 数量
redis-cli KEYS "spring:session:index:*" | wc -l         # 索引数量
```

## 7.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Redis 内存飙升 | Session 数量过多 | 缩短超时时间，限制并发会话数 |
| 存在未过期但无用的 Session | TTL 设置不合理 | 检查 expires key 是否存在 |
| 跨实例 Session 不一致 | Pub/Sub 问题 | 检查网络，升级 Redis 版本 |
| 序列化失败 | 对象不兼容 | 使用 JSON 序列化，统一类型定义 |

## 7.3 Redis 集群注意事项

- Session 的 HASH key 和 expires key **必须在同一 slot**（通过 Hash Tag 保证）
- 推荐将 Session 数据放在独立的 Redis 实例中，与业务缓存隔离
- 集群模式下 Pub/Sub 会广播到所有节点，注意网络开销
