# 1. 性能评估体系

## 1.1 关键性能指标

| 指标 | 描述 | 关注点 |
|------|------|--------|
| **读延迟** | 从存储获取 Session 的时间 | P50 < 5ms, P99 < 20ms |
| **写延迟** | 保存 Session 到存储的时间 | P50 < 5ms, P99 < 20ms |
| **并发能力** | 每秒可处理的 Session 操作数 | 应达到万级 QPS |
| **内存占用** | 每个 Session 平均大小 | 建议 < 10KB |
| **存储容量** | 存储的总数据量 | 需要监控和规划 |

## 1.2 性能基准

```java
@Component
public class SessionPerformanceBenchmark {

    @Autowired
    private SessionRepository<?> repository;

    public void benchmark() {
        long start = System.nanoTime();

        // 创建 Session
        Session session = repository.createSession();
        session.setAttribute("test", "value");
        repository.save(session);

        // 读取 Session
        Session loaded = repository.findById(session.getId());

        // 删除 Session
        repository.deleteById(session.getId());

        long end = System.nanoTime();
        long elapsedMs = (end - start) / 1_000_000;

        System.out.printf("Session 全生命周期耗时: %d ms%n", elapsedMs);
    }
}
```

---

# 2. 序列化优化

## 2.1 选择最快序列化器

| 序列化方式 | 序列化速度 | 反序列化速度 | 数据大小 |
|------------|------------|--------------|----------|
| JDK | 基准 | 基准 | 基准 |
| Jackson JSON | 1.2x | 1.5x | 0.7x |
| Kryo | 3x-5x | 3x-5x | 0.4x |
| Protobuf | 4x-6x | 4x-6x | 0.3x |

## 2.2 注册 Kryo 类（提升性能）

```java
@Component
public class KryoSerializerConfig {

    private final ThreadLocal<Kryo> kryoThreadLocal = ThreadLocal.withInitial(() -> {
        Kryo kryo = new Kryo();
        // 注册已知类，避免写入完整类名
        kryo.register(HashMap.class);
        kryo.register(ArrayList.class);
        kryo.register(UserInfo.class);
        kryo.register(User.class);  // Spring Security User
        kryo.register(SimpleGrantedAuthority.class);
        // 注册得越多，性能越好
        return kryo;
    });

    // ...
}
```

## 2.3 压缩存储

```java
@Component
public class CompressedRedisSerializer implements RedisSerializer<Object> {

    private final RedisSerializer<Object> delegate = new KryoRedisSerializer();

    @Override
    public byte[] serialize(Object object) {
        byte[] raw = delegate.serialize(object);
        // GZIP 压缩
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (GZIPOutputStream gzip = new GZIPOutputStream(bos)) {
            gzip.write(raw);
        }
        return bos.toByteArray();
    }

    @Override
    public Object deserialize(byte[] bytes) {
        ByteArrayInputStream bis = new ByteArrayInputStream(bytes);
        byte[] decompressed;
        try (GZIPInputStream gzip = new GZIPInputStream(bis)) {
            decompressed = gzip.readAllBytes();
        }
        return delegate.deserialize(decompressed);
    }
}
```

**压缩注意**：小对象压缩效果不明显，甚至可能增大体积（GZIP 头开销）。建议仅对大对象（> 1KB）启用压缩。

---

# 3. 连接池优化

## 3.1 Lettuce 共享连接

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          enabled: true
          max-active: 20        # 最大活跃连接
          max-idle: 10          # 最大空闲连接
          min-idle: 5           # 最小空闲连接
          max-wait: 2000ms      # 获取连接超时
          time-between-eviction-runs: 30s # 空闲连接回收周期
```

## 3.2 Lettuce 共享原生连接

```java
@Configuration
public class RedisPoolConfig {

    @Bean
    public LettuceClientConfigurationBuilderCustomizer customizer() {
        return builder -> builder
            .clientOptions(ClientOptions.builder()
                .autoReconnect(true)                        // 自动重连
                .disconnectedBehavior(DisconnectedBehavior.REJECT_COMMANDS) // 断开时拒绝命令
                .socketOptions(SocketOptions.builder()
                    .connectTimeout(Duration.ofSeconds(2))  // 连接超时
                    .keepAlive(true)                        // TCP KeepAlive
                    .build())
                .build())
            .commandTimeout(Duration.ofSeconds(3));          // 命令超时
    }
}
```

---

# 4. 存储层优化

## 4.1 Redis 优化

```bash
# 禁用不必要的持久化（如果 Session 丢失可以接受重新登录）
# redis.conf
save ""

# 如果必须持久化，使用 AOF（数据更安全）
appendonly yes
appendfsync everysec

# 内存管理
maxmemory 2gb
maxmemory-policy volatile-lru  # 仅驱逐有过期时间的 Key

# 禁用慢命令日志记录阈值
slowlog-log-slower-than 5000  # 5ms（Session 操作应该是微秒级）
```

## 4.2 JDBC 优化

```sql
-- 优化索引使用
EXPLAIN SELECT * FROM SPRING_SESSION WHERE SESSION_ID = 'xxx';

-- 定期优化表（MySQL InnoDB）
OPTIMIZE TABLE SPRING_SESSION;
OPTIMIZE TABLE SPRING_SESSION_ATTRIBUTES;

-- 分区表（按 EXPIRY_TIME 分区，加速清理）
ALTER TABLE SPRING_SESSION
PARTITION BY RANGE (EXPIRY_TIME) (
    PARTITION p0 VALUES LESS THAN (1700000000000),
    PARTITION p1 VALUES LESS THAN (1700086400000),
    -- ...
);
```

## 4.3 数据淘汰与清理

```java
@Component
public class SessionEvictionService {

    // 定期清理长期不活跃的 Session
    @Scheduled(fixedRate = 300000) // 每 5 分钟
    public void evictInactiveSessions() {
        // 对于 Redis：依赖 TTL 自动过期
        // 对于 JDBC：执行 DELETE 语句
    }
}
```

---

# 5. 缓存策略

## 5.1 本地缓存 + 远程存储（二级缓存）

```java
@Component
public class TwoLevelSessionCache {

    // L1: 本地缓存（Caffeine）
    private final Cache<String, Session> localCache = Caffeine.newBuilder()
        .maximumSize(10000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .build();

    // L2: 远程存储（Redis）
    @Autowired
    private SessionRepository<?> remoteRepository;

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

    public void save(Session session) {
        remoteRepository.save(session);
        localCache.put(session.getId(), session);
    }

    public void deleteById(String id) {
        remoteRepository.deleteById(id);
        localCache.invalidate(id);
    }
}
```

**二级缓存注意事项**：

| 注意点 | 说明 |
|--------|------|
| **一致性** | 本地缓存可能滞后于远程存储，需要设置合理的过期时间 |
| **内存占用** | 本地缓存占用 JVM 堆内存，需要控制大小 |
| **适用范围** | 适合读多写少的场景，不适合频繁修改 Session 的场景 |
| **过期同步** | 远程 Session 过期后，本地缓存可能仍保留旧数据 |

## 5.2 无需二级缓存的场景

大多数场景下，**Redis 本身已经足够快**（1ms 以内），不需要二级缓存：

```
Redis P50 延迟 ≈ 0.5ms
本地缓存 P50 延迟 ≈ 0.001ms

收益：0.499ms
代价：缓存一致性维护成本

结论：通常不值得
```

---

# 6. 请求级优化

## 6.1 按需加载 Session

```java
// 不要在所有请求中都加载 Session
// 只有在业务需要时才调用 request.getSession()

// 好：仅在需要时获取
@GetMapping("/public/status")
public String status() {
    return "OK";  // 不需要 Session
}

// 差：不必要的 Session 创建
@GetMapping("/public/status")
public String status(HttpSession session) {
    return "OK";  // 不必要地创建/加载了 Session
}
```

## 6.2 减少 Session 写入次数

```java
// 差：每个方法都单独写入
session.setAttribute("attr1", value1);
session.setAttribute("attr2", value2);
session.setAttribute("attr3", value3);

// 好：批量设置后统一保存（FlushMode.ON_SAVE 下适用）
// 请求结束时自动保存，减少网络 I/O
```

## 6.3 Session 属性大小控制

```java
// 每个 Session 属性建议 < 10KB

// 差：存储大对象
session.setAttribute("userFullProfile", fetchFullProfile());  // 可能 100KB

// 好：存储标识
session.setAttribute("userId", userId);
// 需要 profile 时通过 userId 查询数据库或缓存
```

---

# 7. 监控与告警

## 7.1 Micrometer 指标

```java
@Component
public class SessionMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter sessionCreateCounter;
    private final Timer sessionLoadTimer;

    public SessionMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.sessionCreateCounter = Counter.builder("session.created")
            .description("Session 创建数量")
            .register(meterRegistry);

        this.sessionLoadTimer = Timer.builder("session.load.time")
            .description("Session 加载耗时")
            .register(meterRegistry);
    }

    // 在业务代码中记录
    public void recordSessionCreated() {
        sessionCreateCounter.increment();
    }

    public void recordSessionLoad(long durationMs) {
        sessionLoadTimer.record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

## 7.2 关键告警规则

| 指标 | 告警条件 | 说明 |
|------|----------|------|
| Session 创建速率 | 突增 > 3x | 可能是攻击或异常流量 |
| Session 查找延迟 | P99 > 100ms | Redis/JDBC 性能问题 |
| Redis 连接等待时间 | > 500ms | 连接池不足或 Redis 负载高 |
| 过期 Session 堆积 | > 正常值的 2x | TTL 或清理任务异常 |
| 活跃 Session 总数 | > 预期上限的 80% | Redis 内存可能不足 |
