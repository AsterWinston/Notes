# 1. 概述与简介

## 1.1 什么是 Redisson

Redisson 是一个基于 Java 的 Redis 智能客户端，提供了众多丰富的分布式服务和数据结构，远不止是一个 Redis 客户端。它构建在 Redis 核心之上，将 Redis 从键值存储升级为具有内存数据网格（IMDG）能力的全功能分布式系统。

```
┌─────────────────────────────────────────────────────────────────┐
│                         Redisson 生态                             │
├─────────────────────────────────────────────────────────────────┤
│  分布式对象  │ 分布式锁  │ 分布式集合  │ 分布式服务  │ Spring 集成 │
│  RMap       │ RLock    │ RSet       │ RExecutor   │ Cache      │
│  RSet       │ RedLock  │ RList      │ RScheduler  │ Session    │
│  RList      │ RReadWriteLock │ RQueue │ RTopic     │ 注解       │
│  RAtomicLong│ RFairLock│ RBloomFilter│ RRateLimiter│ 模版       │
│  RBucket    │ RMultiLock│ RHyperLogLog│ RRemoteService│          │
└─────────────────────────────────────────────────────────────────┘
```

## 1.2 架构概述

Redisson 采用分层的架构设计：

```
┌──────────────────────────────────────────────────────┐
│  用户应用（业务代码）                                    │
├──────────────────────────────────────────────────────┤
│  RedissonClient 对外 API                              │
│  ┌────────────────────────────────────────────────┐   │
│  │  分布式对象/锁/集合/服务 抽象层面                 │   │
│  ├────────────────────────────────────────────────┤   │
│  │  编解码器 (Codec)                              │   │
│  │  Jackson JSON / Avro / Smile / CBOR / Kryo    │   │
│  ├────────────────────────────────────────────────┤   │
│  │  连接管理器 (ConnectionManager)                │   │
│  │  Netty 4 (全异步非阻塞)                        │   │
│  ├────────────────────────────────────────────────┤   │
│  │  Redis 协议层 (RESP2 / RESP3)                  │   │
│  └────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────┤
│  Redis 服务器（单机/哨兵/集群/主从/复制）                │
└──────────────────────────────────────────────────────┘
```

## 1.3 Redisson vs Jedis vs Lettuce

| 对比项 | Jedis | Lettuce | Redisson |
|--------|-------|---------|----------|
| 设计模式 | 同步阻塞 | 异步响应式 | 分布式数据网格 |
| 线程安全 | 否（连接池） | 是（单连接多线程） | 是（基于 Netty） |
| 连接方式 | BIO (阻塞 IO) | NIO (基于 Netty) | NIO (基于 Netty) |
| API 风格 | 底层命令式 | 底层命令式 + Reactive | 高级分布式对象抽象 |
| 分布式锁 | 自己实现 | 自己实现 | 内置全套方案 |
| 学习曲线 | 低 | 中 | 中高（但使用更方便） |
| 扩展性 | 低 | 中 | 高（内置集群支持） |
| 推荐场景 | 简单 Redis 操作 | 高性能异步场景 | 分布式组件（锁、集合、服务） |

**总结**：
- **Jedis**：轻量，适合简单场景或已有大量 Jedis 代码的遗留项目。
- **Lettuce**：异步驱动，Spring Boot 2.x 默认 Redis 客户端，适合对吞吐有高要求的场景。
- **Redisson**：功能最丰富，适合需要分布式锁、分布式对象等高级特性的微服务架构项目。

# 2. 核心功能与 API

## 2.1 配置与 RedissonClient

### 2.1.1 单机模式

```java
// 编程式配置
Config config = new Config();
config.useSingleServer()
    .setAddress("redis://127.0.0.1:6379")
    .setPassword("password")
    .setDatabase(0)
    .setConnectionPoolSize(64)
    .setConnectionMinimumIdleSize(24)
    .setTimeout(3000)
    .setRetryAttempts(3);

RedissonClient redisson = Redisson.create(config);
```

### 2.1.2 集群模式

```java
Config config = new Config();
config.useClusterServers()
    .addNodeAddress(
        "redis://127.0.0.1:7001",
        "redis://127.0.0.1:7002",
        "redis://127.0.0.1:7003"
    )
    .setScanInterval(2000)     // 扫描间隔（毫秒）
    .setPassword("password")
    .setMasterConnectionPoolSize(64)
    .setSlaveConnectionPoolSize(64);

RedissonClient redisson = Redisson.create(config);
```

### 2.1.3 哨兵模式

```java
Config config = new Config();
config.useSentinelServers()
    .addSentinelAddress(
        "redis://127.0.0.1:26379",
        "redis://127.0.0.1:26380"
    )
    .setMasterName("mymaster")
    .setDatabase(0)
    .setPassword("password");
```

### 2.1.4 主从模式

```java
Config config = new Config();
config.useMasterSlaveServers()
    .setMasterAddress("redis://127.0.0.1:6379")
    .addSlaveAddress("redis://127.0.0.1:6380", "redis://127.0.0.1:6381")
    .setPassword("password");
```

### 2.1.5 复制模式（Replicated）

```java
Config config = new Config();
config.useReplicatedServers()
    .addNodeAddress("redis://127.0.0.1:6379", "redis://127.0.0.1:6380")
    .setScanInterval(2000);
```

### 2.1.6 YAML 配置文件

```yaml
# redisson-config.yml
singleServerConfig:
  address: "redis://127.0.0.1:6379"
  password: null
  database: 0
  connectionPoolSize: 64
  connectionMinimumIdleSize: 24
  timeout: 3000
  retryAttempts: 3
codec: !<org.redisson.codec.Kryo5Codec> {}
```

```java
RedissonClient redisson = Redisson.create(
    Config.fromYAML(new File("redisson-config.yml"))
);
```

## 2.2 分布式对象

### 2.2.1 RBucket（通用对象桶）

```java
RBucket<User> bucket = redisson.getBucket("user:1001");

// 设置值
bucket.set(new User(1001, "Alice"));

// 获取值
User user = bucket.get();

// 原子比较并交换
bucket.compareAndSet(expected, newValue);

// 获取并设置
User old = bucket.getAndSet(newValue);

// 设置过期时间
bucket.set(newValue, 10, TimeUnit.MINUTES);

// 如果不存在则设置
bucket.trySet(newValue);
bucket.trySet(newValue, 10, TimeUnit.MINUTES);
```

### 2.2.2 RMap（分布式 Map）

```java
RMap<String, Object> map = redisson.getMap("user:1001:profile");

// 基本操作
map.put("name", "Alice");
map.put("age", 30);
map.put("email", "alice@example.com");

// 批量
map.putAll(Map.of("city", "Beijing", "phone", "13800138000"));

// 获取
String name = (String) map.get("name");
Map<String, Object> all = map.readAllMap();

// 原子操作
map.fastPut("name", "Bob");            // 不返回旧值，更快
map.fastRemove("phone");               // 不返回旧值，更快

// 并发安全操作
map.putIfAbsent("name", "Charlie");
map.replace("name", "Alice", "Bob");   // CAS 更新
map.remove("name", "Alice");           // CAS 删除

// 监听
map.addListener(new EntryCreatedListener() { ... });
map.addListener(new EntryUpdatedListener() { ... });
map.addListener(new EntryRemovedListener() { ... });
```

**RMap 与本地 HashMap 的重要区别**：
- 每次操作都涉及网络 IO。
- 建议使用 `fastPut` / `fastRemove` 优于 `put` / `remove` 以减少返回值传输。
- 大 Map 遍历使用 `readAllMap()` / `readAllEntrySet()` 一次性读取，避免逐条遍历。

### 2.2.3 RSet（分布式 Set）

```java
RSet<String> set = redisson.getSet("online_users");

set.add("user1");
set.add("user2");
set.add("user3");

boolean exists = set.contains("user1");
Set<String> all = set.readAll();

// 批量操作
set.addAll(Arrays.asList("user4", "user5"));

// 交集/并集/差集
RSet<String> set1 = redisson.getSet("set1");
RSet<String> set2 = redisson.getSet("set2");
set1.intersection("set2");      // 交集
set1.union("set2");             // 并集
set1.diff("set2");              // 差集
```

### 2.2.4 RList（分布式 List）

```java
RList<String> list = redisson.getList("my_queue");

list.add("item1");
list.add("item2");
list.add(0, "item0");         // 头部插入

String first = list.get(0);
String removed = list.remove(0);
```

### 2.2.5 RQueue / RDeque（分布式队列）

```java
RQueue<String> queue = redisson.getQueue("task_queue");
queue.add("task1");
queue.add("task2");

String task = queue.poll();     // 非阻塞出队（没有则返回 null）
String task = queue.poll(10, TimeUnit.SECONDS);  // 等待最多 10 秒

// 阻塞队列
RBlockingQueue<String> blockingQueue = redisson.getBlockingQueue("blocking_queue");
blockingQueue.put("item");
String item = blockingQueue.take();  // 阻塞直到有元素
String item = blockingQueue.poll(5, TimeUnit.SECONDS);  // 超时等待
```

### 2.2.6 RAtomicLong（分布式原子变量）

```java
RAtomicLong counter = redisson.getAtomicLong("visitor_count");

counter.set(0);
long value = counter.incrementAndGet();     // +1 并返回
long value = counter.addAndGet(10);          // +10 并返回
long value = counter.getAndDecrement();      // 返回后 -1
boolean success = counter.compareAndSet(5, 10);
```

### 2.2.7 RBloomFilter（布隆过滤器）

布隆过滤器用于判断一个元素是否"可能存在于集合中"或"一定不存在"。

```java
RBloomFilter<String> bloomFilter = redisson.getBloomFilter("user_bloom");

// 初始化：预期容量 100000，误差率 1%
bloomFilter.tryInit(100000L, 0.01);

// 添加元素
bloomFilter.add("user1");
bloomFilter.add("user2");

// 判断是否存在
boolean exists = bloomFilter.contains("user1");   // true
boolean exists = bloomFilter.contains("unknown"); // false（大概率）
```

**应用场景**：防止缓存穿透——在访问 DB 前先检查 Bloom Filter，不存在则直接返回。

### 2.2.8 RHyperLogLog（基数估计）

```java
RHyperLogLog<String> log = redisson.getHyperLogLog("uv:2024-01-01");

log.add("user1");
log.add("user2");
log.add("user1");  // 重复，不计入

long count = log.count();  // ≈ 2
```

**特点**：标准误差约 0.81%，内存占用固定 12KB，适合 UV 统计等大规模基数估算场景。

### 2.2.9 RRateLimiter（分布式限流器）

```java
RRateLimiter rateLimiter = redisson.getRateLimiter("api_rate_limiter");

// 每秒最多 10 个请求
rateLimiter.trySetRate(RateType.OVERALL, 10, 1, RateIntervalUnit.SECONDS);

// 尝试获取许可
boolean acquired = rateLimiter.tryAcquire();
boolean acquired = rateLimiter.tryAcquire(5);          // 尝试获取 5 个许可
boolean acquired = rateLimiter.tryAcquire(1, 100, TimeUnit.MILLISECONDS); // 超时等待

// 阻塞获取
rateLimiter.acquire();      // 阻塞直到获取到许可
```

**RateType**：
- `RateType.OVERALL`：所有客户端共享速率（全局限流）。
- `RateType.PER_CLIENT`：每个客户端独立速率。

### 2.2.10 RBitSet（分布式位图）

```java
RBitSet bitSet = redisson.getBitSet("user_checkin:2024-01");

bitSet.set(1, true);   // 用户 ID=1 签到
bitSet.set(2, true);   // 用户 ID=2 签到

boolean checked = bitSet.get(1);  // 用户 1 是否签到

// 位运算
bitSet.or("another_bit_set");
```

## 2.3 分布式锁

### 2.3.1 RLock（可重入锁）

```java
RLock lock = redisson.getLock("order:lock:1001");

// 加锁（阻塞等待）
lock.lock();

// 加锁（超时等待）
boolean locked = lock.tryLock(10, TimeUnit.SECONDS);
if (locked) {
    try {
        // 业务逻辑
    } finally {
        lock.unlock();
    }
}

// 指定锁持有时间（自动解锁，避免死锁）
lock.lock(30, TimeUnit.SECONDS);  // 30 秒后自动解锁

// 公平锁：等待最久的线程优先获取
RLock fairLock = redisson.getFairLock("fair_order_lock");
fairLock.lock();
```

**可重入性**：同一线程可以多次加锁，需配套相同次数 unlock。

### 2.3.2 RedLock（红锁）

RedLock 是 Redis 官方推荐的分布式锁算法，要求在 N/2+1 个 Redis 节点上成功获取锁才算加锁成功。

```java
// 需要多个独立的 Redis 节点（非集群模式，而是多个主节点）
Config config1 = new Config();
config1.useSingleServer().setAddress("redis://node1:6379");
RedissonClient redisson1 = Redisson.create(config1);

Config config2 = new Config();
config2.useSingleServer().setAddress("redis://node2:6379");
RedissonClient redisson2 = Redisson.create(config2);

Config config3 = new Config();
config3.useSingleServer().setAddress("redis://node3:6379");
RedissonClient redisson3 = Redisson.create(config3);

RLock lock1 = redisson1.getLock("redlock:resource");
RLock lock2 = redisson2.getLock("redlock:resource");
RLock lock3 = redisson3.getLock("redlock:resource");

RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);

try {
    boolean locked = redLock.tryLock(10, 30, TimeUnit.SECONDS);
    if (locked) {
        // 执行业务逻辑
    }
} finally {
    redLock.unlock();
}
```

**RedLock 前提条件**：
- 节点相互独立（不同物理机器）。
- 需要至少 N/2 + 1 个节点加锁成功（N=3 时，需要 2 个节点）。
- 获取锁的总时长必须小于锁的 TTL。

### 2.3.3 RReadWriteLock（读写锁）

```java
RReadWriteLock rwLock = redisson.getReadWriteLock("config_lock");
RLock readLock = rwLock.readLock();
RLock writeLock = rwLock.writeLock();

// 读锁：可共享（多个线程可同时获取）
readLock.lock();
try {
    String config = redisson.getBucket("config").get();
    // 只读操作
} finally {
    readLock.unlock();
}

// 写锁：排他（读/写都阻塞）
writeLock.lock();
try {
    redisson.getBucket("config").set(newConfig);
    // 写操作
} finally {
    writeLock.unlock();
}
```

### 2.3.4 RMultiLock（联锁）

同时对多个锁加锁，全部成功才算成功。

```java
RLock lock1 = redisson.getLock("lock1");
RLock lock2 = redisson.getLock("lock2");
RLock lock3 = redisson.getLock("lock3");

RMultiLock multiLock = new RedissonMultiLock(lock1, lock2, lock3);
multiLock.lock();
try {
    // 所有锁都获取后才能执行
} finally {
    multiLock.unlock();
}
```

## 2.4 分布式集合

### 2.4.1 RMapCache（带过期功能的 Map）

```java
RMapCache<String, Object> mapCache = redisson.getMapCache("session_cache");

mapCache.put("session1", sessionObj, 30, TimeUnit.MINUTES);
mapCache.put("session2", sessionObj, 1, TimeUnit.HOURS);

// 不同字段可设置不同过期时间（Map 的本地缓存不允许）
mapCache.put("temp", tempObj, 10, TimeUnit.SECONDS, 5, TimeUnit.SECONDS);
// 参数：key, value, ttl, ttlUnit, maxIdleTime, maxIdleUnit
```

### 2.4.2 RSetCache（带过期功能的 Set）

```java
RSetCache<String> setCache = redisson.getSetCache("temp_set");
setCache.add("item1", 5, TimeUnit.MINUTES);
```

### 2.4.3 RScoredSortedSet（带分数的有序集合）

```java
RScoredSortedSet<String> set = redisson.getScoredSortedSet("leaderboard");

set.add(100.0, "player1");
set.add(200.0, "player2");
set.add(150.0, "player3");

// 按分数升序
Collection<String> players = set.valueRange(0, -1);

// 获取排名
Integer rank = set.rank("player1");
Double score = set.getScore("player1");
```

## 2.5 分布式服务

### 2.5.1 RRemoteService（远程服务）

基于 Redis 的分布式 RPC 调用。

```java
// 服务端：注册服务
RRemoteService remoteService = redisson.getRemoteService();
remoteService.register(MyService.class, new MyServiceImpl());

// 客户端：获取服务代理
RRemoteService remoteService = redisson.getRemoteService();
MyService service = remoteService.get(MyService.class);
String result = service.doSomething("param");
```

### 2.5.2 RExecutorService（分布式执行器）

```java
RExecutorService executor = redisson.getExecutorService("my_executor");

// 提交任务
Future<String> future = executor.submit(new MyCallableTask());

// 定时任务
ScheduledFuture<?> scheduledFuture = executor.schedule(
    new MyTask(), 10, TimeUnit.SECONDS
);
```

### 2.5.3 RScheduledService（分布式调度）

```java
RScheduledExecutorService executor = redisson.getExecutorService("scheduler");

// cron 表达式调度
executor.scheduleWithCron("0 0 12 * * ?", new DailyReportTask());
```

## 2.6 分布式发布订阅

### 2.6.1 RTopic（主题 Pub/Sub）

```java
// 订阅者
RTopic topic = redisson.getTopic("notifications");
topic.addListener(String.class, (channel, msg) -> {
    System.out.println("Received: " + msg + " from " + channel);
});

// 发布者
RTopic topic = redisson.getTopic("notifications");
topic.publish("Hello, subscribers!");

// 获取发布数量
long subscribers = topic.publish("message");
```

### 2.6.2 RPatternTopic（模式订阅）

```java
RPatternTopic patternTopic = redisson.getPatternTopic("notifications:*");
patternTopic.addListener(String.class, (pattern, channel, msg) -> {
    System.out.println("Pattern: " + pattern + ", Channel: " + channel + ", Msg: " + msg);
});
```

# 3. 高级特性

## 3.1 看门狗机制（Watchdog）

Redisson 的分布式锁默认启用看门狗（Watchdog）机制，自动续期。

**工作原理**：

```
1. 调用 lock() 加锁（不含 leaseTime 参数）
2. Redis 中创建 key，默认过期时间 30 秒
3. Redisson 启动一个后台定时任务（每 10 秒执行一次）
4. 如果业务线程还在运行（锁未释放），续期 30 秒
5. 业务完成 → unlock() 释放锁并取消看门狗
```

```java
// 启用看门狗（不指定 leaseTime）
RLock lock = redisson.getLock("watchdog_lock");
lock.lock();  // 看门狗自动续期，默认每 10 秒续 30 秒

// 自定义看门狗超时时间
Config config = new Config();
config.setLockWatchdogTimeout(20_000);  // 20 秒（默认 30 秒）
RedissonClient redisson = Redisson.create(config);

// 禁止看门狗（指定 leaseTime）
lock.lock(10, TimeUnit.SECONDS);  // 10 秒后自动释放，不会续期
```

**看门狗源码简化流程**：

```java
// Redisson 内部定时逻辑示意
private void renewExpiration() {
    Timeout task = executorService.schedule(() -> {
        // 检查锁是否还被当前线程持有
        if (isHeldByCurrentThread()) {
            // 通过 Lua 脚本续期
            redis.eval(
                "if redis.call('hexists', KEYS[1], ARGV[1]) == 1 then " +
                "  redis.call('pexpire', KEYS[1], ARGV[2]); " +
                "  return 1; " +
                "end; " +
                "return 0;",
                keys, values
            );
            // 递归调度自己
            renewExpiration();
        }
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
}
```

## 3.2 Lua 脚本

Redisson 内部大量使用 Lua 脚本保证原子性。

```java
// 自定义 Lua 脚本
String luaScript = """
    local current = redis.call('get', KEYS[1])
    if current then
        return tonumber(current) >= tonumber(ARGV[1])
    end
    return false
""";

// 执行脚本
RScript script = redisson.getScript();
boolean result = script.eval(
    RScript.Mode.READ_ONLY,
    luaScript,
    RScript.ReturnType.BOOLEAN,
    Arrays.asList("threshold"),
    100
);
```

**原子锁 Lua 示例**：

```lua
-- 加锁
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
return redis.call('pttl', KEYS[1]);
```

## 3.3 编解码器（Codec）

Redisson 支持多种序列化方式：

| Codec | 说明 | 适用场景 |
|-------|------|---------|
| JsonJacksonCodec | 默认，JSON 格式 | 通用，可读性好 |
| Kryo5Codec | Kryo 序列化 | 性能优先，跨语言需自定义 |
| AvroCodec | Apache Avro | 大数据生态 |
| SmileCodec | Jackson Smile 二进制 JSON | 兼容 JSON 且更小 |
| CBORCodec | CBOR 二进制 JSON | IOT 场景 |
| StringCodec | 纯字符串 | 仅处理 String 类型 |
| MarshallingCodec | JDK 序列化 | 兼容遗留系统 |

```java
// 全局配置
Config config = new Config();
config.setCodec(new Kryo5Codec());

// 每个实例独立配置
config.useSingleServer().setAddress("redis://127.0.0.1:6379");

// 自定义序列化注册
Kryo5Codec codec = new Kryo5Codec(
    MyClass.class, AnotherClass.class
);
```

## 3.4 事务支持

```java
RTransaction transaction = redisson.createTransaction(
    TransactionOptions.defaults()
);

try {
    RBucket<String> bucket = transaction.getBucket("tx_key");
    bucket.set("value1");

    RMap<String, String> map = transaction.getMap("tx_map");
    map.put("k1", "v1");

    transaction.commit();
} catch (Exception e) {
    transaction.rollback();
}
```

## 3.5 管道（Batch）

```java
RBatch batch = redisson.createBatch(BatchOptions.defaults());

// 批量提交命令到 batch
batch.getBucket("key1").setAsync("value1");
batch.getBucket("key2").setAsync("value2");
batch.getMap("map1").putAsync("k", "v");

// 原子执行
BatchResult<?> result = batch.execute();
```

# 4. 底层原理与设计

## 4.1 连接管理

Redisson 基于 Netty 框架构建异步客户端。连接管理器（ConnectionManager）维护到 Redis 服务器的连接池。

```
┌────────────┐     ┌───────────────────┐     ┌────────────┐
│  业务线程   │────►│   CommandAsync    │────►│   Channel  │
│            │     │   Service         │     │   (Netty)  │
└────────────┘     └───────────────────┘     └────────────┘
                         │                        │
                         ▼                        ▼
                   ┌───────────────────┐     ┌────────────┐
                   │  连接池管理        │     │   Redis    │
                   │  (连接健康检测)    │     │   Server   │
                   └───────────────────┘     └────────────┘
```

## 4.2 分布式锁原理

### 4.2.1 加锁流程（简化）

```
lock.lock()
    │
    ├─ 1. 尝试加锁（Lua 脚本）
    │      ┌─ KEY[1] = "mylock"                 (锁名称)
    │      └─ ARGV[1] = "UUID:ThreadId"         (锁持有者标识)
    │
    ├─ 2. 加锁成功 → 设置看门狗续期定时器
    │
    ├─ 3. 加锁失败 → 订阅锁释放消息（Redis Pub/Sub）
    │      └─ 阻塞等待，收到消息后重新尝试
    │
    └─ 4. 达到最大等待时间 → 抛出异常
```

### 4.2.2 解锁流程

```
unlock()
    │
    ├─ 1. Lua 脚本检查持有者
    │      ┌─ 持有者是当前线程 → 释放锁
    │      └─ 持有者不是当前线程 → 不做操作
    │
    ├─ 2. 释放锁后 → 向锁释放频道发送消息（唤醒等待者）
    │
    └─ 3. 取消看门狗定时器
```

### 4.2.3 锁的数据结构

Redisson 的锁使用 Redis Hash 存储：

```
key: "mylock"
field: "UUID:ThreadId" (持有者标识)
value: 重入计数

示例：
mylock = {
    "a3f2c1d0:42": 1   // UUID:a3f2c1d0 的线程 42 持有锁，重入 1 次
}
```

## 4.3 RedLock 算法详解

Redisson 的 RedLock 遵循 Martin Kleppmann 和 Redis 作者 antirez 讨论的 RedLock 算法：

```
客户端获取当前时间 T1
对于 N 个 Redis 节点（N=3 或 5，通常 5）：
    for node in nodes:
        success = tryLock(node, resource, lockValue, ttl)
        记录超时

计算获取锁耗费的时间 elapsed = 当前时间 - T1
条件：
    1. 成功加锁节点数 >= N/2 + 1
    2. elapsed < lockTTL
满足两个条件 → 加锁成功
否则 → 对所有节点释放锁
```

## 4.4 发布订阅机制

Redisson 利用 Redis 的 Pub/Sub 实现分布式锁的等待通知：

```
加锁失败时：
1. 订阅频道 "__keyevent@db__:2:__:mylock"
2. 阻塞等待信号量 Semaphore
3. 收到消息 → 重新尝试加锁

优点：
- 避免空转轮询
- 消息实时性高
- 释放锁时唤醒等待者，减少延迟
```

## 4.5 线程模型

Redisson 基于 Netty EventLoop 实现：

- **Boss EventLoop**：处理连接建立事件。
- **Worker EventLoop**：处理读写 IO 事件。
- 默认 Worker 数为 CPU 核数 * 2。
- 业务线程将命令封装为 `Future` 后提交到 EventLoop，通过回调/CompletableFuture 获取结果。

# 5. 性能优化

## 5.1 连接池配置

```java
Config config = new Config();
config.useSingleServer()
    .setConnectionPoolSize(64)              // 最大连接数（默认 64）
    .setConnectionMinimumIdleSize(24)       // 最小空闲连接（默认 24）
    .setConnectionAcquisitionTimeout(5000)  // 获取连接超时（毫秒）
    .setIdleConnectionTimeout(10000)        // 空闲连接超时
    .setConnectTimeout(10000)               // 连接超时
    .setRetryAttempts(3)                    // 重试次数
    .setRetryInterval(1500);                // 重试间隔（毫秒）
```

**最佳实践**：
- 连接池大小 = 业务并发线程数 × 每个事务的操作次数。
- 不要过度设置（过大增加上下文切换）。
- 建议压测确定最优值。

## 5.2 批量操作优化

```java
// ❌ 不推荐：逐条操作（产生 N 次网络 IO）
for (String key : keys) {
    redisson.getBucket(key).set(value);
}

// ✅ 推荐：使用 RBatch（一次网络 IO）
RBatch batch = redisson.createBatch(BatchOptions.defaults());
batch.getBucket("key1").setAsync("value1");
batch.getBucket("key2").setAsync("value2");
batch.getBucket("key3").setAsync("value3");
BatchResult<?> result = batch.execute();
```

## 5.3 序列化优化

```java
// 性能：Kryo > Jackson-Smile > JSON > JDK 序列化
// 推荐：Kryo5Codec（体积小、速度快）

Config config = new Config();
config.setCodec(new Kryo5Codec());

// 如果不需要跨语言，Kryo 是最优选择
// 需要跨语言则使用 JSON
```

**序列化性能对比**：

| 编解码器 | 序列化速度 | 反序列化速度 | 体积 | 跨语言 |
|---------|-----------|------------|------|--------|
| Kryo5Codec | 最快 | 最快 | 最小 | 需配置 |
| JsonJacksonCodec | 中等 | 中等 | 较大 | 是 |
| SmileCodec | 中上 | 中上 | 中等 | 是 |
| JDK | 慢 | 慢 | 最大 | 否 |

## 5.4 大集合操作优化

```java
// ❌ 不推荐：大数据量的逐条操作
RMap<String, Data> map = redisson.getMap("big_map");
for (String key : thousandKeys) {
    map.get(key);  // N 次请求
}

// ✅ 推荐：批量读取
Map<String, Data> result = map.getAll(Set.copyOf(thousandKeys));

// ✅ 使用分布式迭代器（适合超大集合）
RSet<String> bigSet = redisson.getSet("huge_set");
for (String element : bigSet) {
    // 内部自动分批读取
}
```

## 5.5 线程池配置

```java
Config config = new Config();
config.setExecutor(
    new ThreadPoolExecutor(
        4, 8, 60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(5000),
        new ThreadFactoryBuilder().setNameFormat("redisson-exec-%d").build()
    )
);

config.setNettyThreads(8);  // Netty 工作线程数（默认 CPU*2）
```

# 6. 与 Spring Boot 集成

## 6.1 redisson-spring-boot-starter

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.24.0</version>
</dependency>
```

### 6.1.1 application.yml 配置

```yaml
spring:
  redis:
    redisson:
      config: classpath:redisson.yml    # 指定配置文件
```

或者在 application.yml 中直接配置 Spring Redis 属性：

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    database: 0
    timeout: 3000ms
    redisson:
      # Redisson 会自动读取以上配置并初始化
```

### 6.1.2 纯 Redisson 配置（不完全依赖 Spring Data Redis）

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://127.0.0.1:6379")
            .setConnectionPoolSize(32)
            .setConnectionMinimumIdleSize(8);
        return Redisson.create(config);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        return template;
    }
}
```

## 6.2 Spring Cache 集成

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedissonClient redissonClient) {
        Map<String, CacheConfig> config = new HashMap<>();

        // 配置 users 缓存：TTL 30 分钟，最大空闲 10 分钟
        config.put("users", new CacheConfig(
            TimeUnit.MINUTES.toMillis(30),   // TTL
            TimeUnit.MINUTES.toMillis(10)    // 最大空闲时间
        ));

        // 配置 sessions 缓存：TTL 1 小时
        config.put("sessions", new CacheConfig(
            TimeUnit.HOURS.toMillis(1),
            TimeUnit.MINUTES.toMillis(30)
        ));

        return new RedissonSpringCacheManager(redissonClient, config);
    }
}
```

### 6.2.1 @Cacheable 使用

```java
@Service
public class UserService {

    @Cacheable(
        value = "users",
        key = "#userId",
        unless = "#result == null"
    )
    public User getUser(Long userId) {
        return userMapper.selectById(userId);
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        userMapper.updateById(user);
        return user;
    }

    @CacheEvict(value = "users", key = "#userId")
    public void deleteUser(Long userId) {
        userMapper.deleteById(userId);
    }
}
```

## 6.3 Spring Session 集成

Redisson 可以作为 Spring Session 的存储后端，替代默认的内存 Session。

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-session</artifactId>
    <version>3.24.0</version>
</dependency>
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    // 使用 Redis 存储 Session，支持分布式部署
}
```

## 6.4 Spring Boot Actuator 健康检查

```java
@Configuration
public class RedissonHealthIndicator implements HealthIndicator {

    @Autowired
    private RedissonClient redissonClient;

    @Override
    public Health health() {
        try {
            RKeys keys = redissonClient.getKeys();
            long count = keys.count();
            return Health.up()
                .withDetail("totalKeys", count)
                .withDetail("nodes", redissonClient.getConfig().getNodeAddresses())
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

# 7. 最佳实践与常见问题

## 7.1 分布式锁最佳实践

### 7.1.1 加锁范式

```java
// ✅ 标准写法
RLock lock = redisson.getLock("myLock");
try {
    // 等待最多 10 秒，自动释放 30 秒
    boolean locked = lock.tryLock(10, 30, TimeUnit.SECONDS);
    if (!locked) {
        throw new RuntimeException("获取锁失败");
    }
    // 业务逻辑
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} finally {
    // 必须释放锁
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 7.1.2 避免死锁

```java
// ❌ 错误：可能抛出异常导致未释放锁
lock.lock();
doSomething(); // 如果抛出异常，锁永远不会释放
lock.unlock();

// ✅ 正确：无论是否异常都释放
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();
}

// ✅ 更安全：指定 leaseTime 防止业务超时未释放
lock.lock(30, TimeUnit.SECONDS);
try {
    doSomething(); // 如果执行超过 30 秒，锁自动释放
} finally {
    lock.unlock();
}
```

### 7.1.3 锁粒度控制

```java
// ❌ 粗粒度：锁定整个订单服务
RLock lock = redisson.getLock("order");

// ✅ 细粒度：按订单 ID 加锁
RLock lock = redisson.getLock("order:" + orderId);

// ✅ 更细粒度：按单个商品库存加锁
RLock lock = redisson.getLock("stock:" + productId + ":" + skuId);
```

### 7.1.4 锁超时与看门狗决策

```java
// 场景 1：明确知道业务耗时 →指定 leaseTime，避免看门狗额外消耗
lock.lock(100, TimeUnit.MILLISECONDS);

// 场景 2：业务耗时不确定 →使用看门狗自动续期
lock.lock(); // 默认看门狗 30 秒，每 10 秒续期

// 场景 3：短任务 + 高并发 →显式超时快速失败
boolean locked = lock.tryLock(0, 30, TimeUnit.SECONDS);
// tryLock(0, ...) 立即返回，不排队
```

## 7.2 RBloomFilter 防止缓存穿透

```java
@Component
public class BloomFilterService {

    private final RBloomFilter<String> bloomFilter;

    public BloomFilterService(RedissonClient redisson) {
        this.bloomFilter = redisson.getBloomFilter("user_id_bloom");
        this.bloomFilter.tryInit(1_000_000L, 0.001); // 100 万预期，0.1% 误差
    }

    public boolean mightExist(String userId) {
        return bloomFilter.contains(userId);
    }

    public void add(String userId) {
        bloomFilter.add(userId);
    }
}

// 使用
@Service
public class UserService {
    @Autowired private BloomFilterService bloomFilter;
    @Autowired private UserMapper userMapper;

    public User getUser(Long id) {
        // 第一步：检查 Bloom Filter
        if (!bloomFilter.mightExist(String.valueOf(id))) {
            return null; // 一定不存在，直接返回
        }
        // 第二步：查缓存或 DB
        return userMapper.selectById(id);
    }
}
```

## 7.3 分布式限流实践

```java
@Component
public class RateLimiterService {

    private final RedissonClient redisson;

    public RateLimiterService(RedissonClient redisson) {
        this.redisson = redisson;
    }

    /**
     * 接口限流：每秒最多 N 次
     */
    public boolean tryAcquire(String resource, int permitsPerSecond) {
        RRateLimiter limiter = redisson.getRateLimiter("ratelimit:" + resource);
        limiter.trySetRate(RateType.OVERALL, permitsPerSecond, 1, RateIntervalUnit.SECONDS);
        return limiter.tryAcquire();
    }

    /**
     * 用户级别限流：每用户每秒最多 N 次
     */
    public boolean tryAcquirePerUser(String userId, int permitsPerSecond) {
        RRateLimiter limiter = redisson.getRateLimiter("ratelimit:user:" + userId);
        limiter.trySetRate(RateType.PER_CLIENT, permitsPerSecond, 1, RateIntervalUnit.SECONDS);
        return limiter.tryAcquire();
    }
}

// 使用
@RestController
public class ApiController {

    @Autowired
    private RateLimiterService rateLimiter;

    @GetMapping("/api/data")
    public ResponseEntity<?> getData() {
        if (!rateLimiter.tryAcquire("data-api", 100)) {
            return ResponseEntity.status(429).body("Too Many Requests");
        }
        return ResponseEntity.ok(loadData());
    }
}
```

## 7.4 信号量与 CountDownLatch

### 7.4.1 RSemaphore（分布式信号量）

```java
RSemaphore semaphore = redisson.getSemaphore("task_concurrency");
semaphore.trySetPermits(5); // 最多 5 个并发任务

// 获取许可
semaphore.acquire();  // 阻塞
boolean acquired = semaphore.tryAcquire(1, TimeUnit.SECONDS);

// 释放许可
semaphore.release();
```

### 7.4.2 RCountDownLatch（分布式倒计时门闩）

```java
// 主节点
RCountDownLatch latch = redisson.getCountDownLatch("job_latch");
latch.trySetCount(3);

// 工作节点 1 — 完成后
RCountDownLatch latch = redisson.getCountDownLatch("job_latch");
latch.countDown();

// 主节点等待所有工作完成
RCountDownLatch latch = redisson.getCountDownLatch("job_latch");
latch.await();  // 阻塞直到 count 为 0
latch.await(10, TimeUnit.SECONDS);  // 超时等待
```

## 7.5 常见问题与解决方案

### 7.5.1 Q1：Redisson 连接超时

```yaml
# 优化配置
spring:
  redis:
    timeout: 5000ms
    lettuce:
      pool:
        max-active: 32
        max-idle: 8
        min-idle: 4
```

```java
// 设置重试
Config config = new Config();
config.useSingleServer()
    .setTimeout(5000)
    .setRetryAttempts(3)
    .setRetryInterval(1500);
```

### 7.5.2 Q2：Redis 与 Redisson 版本兼容

| Redis 版本 | Redisson 版本 |
|-----------|--------------|
| Redis 6.x | 3.17+ |
| Redis 7.x | 3.21+ |
| Redis 7.2 | 3.24+ |

### 7.5.3 Q3：看门狗失效场景

- 业务线程执行时间超过 leaseTime（当指定了 leaseTime 时，看门狗不会启动）。
- 业务线程被阻塞（如无限等待、死锁），看门狗正常续期，但需要时间超长。
- 业务线程抛出异常后未释放锁，看门狗持续续期直到锁过期。

### 7.5.4 Q4：集群环境下的锁跨节点问题

- 主从模式下，加锁后主节点宕机，从节点未同步锁信息 → 锁丢失风险。
- 解决方案：使用 RedLock（跨多个独立节点），或使用 Redis 7.x 的 WAIT 命令。

### 7.5.5 Q5：大 Value 引起的性能问题

```java
// ❌ 不推荐：大 Value（>10KB）直接存储
redisson.getBucket("big_data").set(hugeData);

// ✅ 推荐：大 Value 压缩或拆分
// 方案 1：压缩
byte[] compressed = compress(hugeData);
redisson.getBucket("big_data").set(compressed);

// 方案 2：拆分存储（使用 RMap 分段）
RMap<String, byte[]> map = redisson.getMap("big_data_segments");
for (int i = 0; i < segments; i++) {
    map.put("segment_" + i, segmentData[i]);
}
```

### 7.5.6 Q6：Watchdog 导致的并发问题

```java
// 现象：业务执行时间 > leaseTime 时锁已被自动释放，但业务不知道
// 解决方案：在业务代码中检查锁状态

// 方案 1：不使用看门狗，显式设置足够长的 leaseTime
lock.lock(30, TimeUnit.SECONDS);

// 方案 2：业务循环检查锁是否还被持有
while (System.currentTimeMillis() < expireTime) {
    if (!lock.isHeldByCurrentThread()) {
        throw new RuntimeException("Lock lost, retry...");
    }
    // 执行业务片断
}
```

# 8. 与其他技术对比

## 8.1 Redisson vs Jedis

| 对比项 | Jedis | Redisson |
|--------|-------|----------|
| 驱动模型 | 同步阻塞（BIO） | 异步（Netty NIO） |
| 线程安全 | 否（需连接池） | 是（单实例多线程） |
| API 层次 | 底层命令封装 | 高阶分布式对象 |
| 分布式锁 | 需自行实现 | 内置全套方案 |
| 代码量 | 多（需自己组合命令） | 少（直接调用 API） |
| 学习成本 | 低 | 中 |
| 并发性能 | 中 | 高 |
| 版本迭代 | 较慢 | 活跃 |

```java
// Jedis：需要自行管理连接和 Lua 脚本
Jedis jedis = jedisPool.getResource();
try {
    String script = "if redis.call('setnx', KEYS[1], ARGV[1]) == 1 then ... end";
    jedis.eval(script, ...);
} finally {
    jedis.close();
}

// Redisson：一行搞定
RLock lock = redisson.getLock("mylock");
lock.lock();
```

## 8.2 Redisson vs Lettuce

| 对比项 | Lettuce | Redisson |
|--------|---------|----------|
| 设计理念 | 异步响应式客户端 | 分布式数据网格 |
| 响应式支持 | 原生（Reactive/RxJava） | 有限 |
| 高级抽象 | 无 | 丰富（锁、集合、服务） |
| 灵活性 | 高（底层操作全开放） | 中（封装度更高） |
| Spring 默认 | Spring Data Redis 默认 | 非默认 |
| 分布式锁 | 需自行实现 | 内置 |
| 集群支持 | 优秀 | 优秀 |

## 8.3 Redisson vs RedisTemplate

RedisTemplate 是 Spring Data Redis 的核心类，底层可以使用 Jedis 或 Lettuce 驱动。

| 对比项 | RedisTemplate | Redisson |
|--------|--------------|----------|
| 层次 | 模板方法封装 | 高级对象封装 |
| 操作方式 | `opsForValue()`, `opsForHash()` | `getBucket()`, `getMap()` |
| 事务 | @Transactional + Redis 事务 | RTransaction |
| 分布式锁 | 需自行实现 | 内置全套锁 |
| 序列化 | 需手动配置 | 自动（多种 Codec） |
| 代码可读性 | 较低 | 较高 |

```java
// RedisTemplate
redisTemplate.opsForValue().set("key", "value");
redisTemplate.opsForHash().put("map", "field", "value");

// Redisson
redisson.getBucket("key").set("value");
redisson.getMap("map").fastPut("field", "value");
```

## 8.4 Redisson vs 其他 Java 分布式组件

| 组件 | 注册中心需求 | CAP | 存储 |
|------|------------|-----|------|
| Redisson | Redis | CP（锁）/ AP（缓存） | Redis |
| ZooKeeper | ZK 集群 | CP | ZK 树 |
| etcd | etcd 集群 | CP | Raft |
| Hazelcast | 无（P2P） | AP | 内存网格 |
| Apache Ignite | 无（P2P） | AP/CP | 内存+磁盘 |

**选择建议**：
- 已有 Redis → Redisson（最小的架构改动）
- 需要强一致性分布式锁 → ZooKeeper 或 etcd
- 需要分布式计算 → Hazelcast / Ignite

# 9. 面试要点

## 9.1 高频面试题

### 9.1.1 Q1：Redisson 分布式锁的实现原理是什么？

1. **加锁（Lua 脚本）**：使用 Lua 脚本保证原子性。检查 KEYS[1] 是否存在：
   - 不存在 → 创建 Hash，field=UUID:threadId，value=1，设置过期时间。
   - 存在且 field 匹配 → value+1（重入）。
   - 存在且 field 不匹配 → 返回 PTTL（剩余过期时间）。
2. **看门狗（Watchdog）**：加锁成功后启动定时任务（每 10 秒），检查锁是否还在持有，是则续期 30 秒。
3. **阻塞等待**：如果加锁失败，订阅 Redis 频道等待锁释放消息，收到消息后重新尝试加锁。
4. **解锁（Lua 脚本）**：检查持有者，递减重入计数，为 0 时删除 key，向频道发送释放消息。

### 9.1.2 Q2：什么是看门狗机制？为什么要用它？

看门狗是一个后台定时任务，在锁持有期间自动续期过期时间，防止业务逻辑执行时间超过锁的 TTL 导致锁被提前释放。

默认配置：
- 锁过期时间 30 秒
- 看门狗每 10 秒续期一次（延长到 30 秒）

如果指定了 leaseTime（锁持有时间），看门狗不会启动。

### 9.1.3 Q3：Redisson 的锁是可重入的吗？如何实现的？

是的，RLock 是可重入锁。底层使用 Redis Hash 数据结构：
- field = UUID + ThreadId（标识持有者）
- value = 重入计数（每次 lock +1，每次 unlock -1）

重入锁确保了同一线程可以多次获取同一把锁而不会死锁。

### 9.1.4 Q4：什么是 RedLock？什么情况下需要使用？

RedLock 是跨多个独立 Redis 节点的分布式锁算法，解决主从模式下锁丢失的问题。

使用场景：
- 对锁的可靠性要求极高（金融交易、支付等）。
- 部署了多个独立的 Redis 节点（非集群模式）。

缺点：
- 需要额外的 Redis 资源。
- 性能开销较大。
- 有争议（Martin Kleppmann 认为 RedLock 仍有安全问题）。

### 9.1.5 Q5：Redisson 如何保证加锁操作的原子性？

通过 Lua 脚本执行所有 Redis 命令。Lua 脚本在 Redis 中是原子执行的，期间不会插入其他命令。

核心加锁脚本：
```lua
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
return redis.call('pttl', KEYS[1]);
```

### 9.1.6 Q6：Redisson 如何实现可重入锁的？

记录持有者的重入次数（Hash 的 value 字段），lock() 时加 1，unlock() 时减 1，直到减到 0 才真正释放锁。

```java
// 内部 Hash 结构
// key: "mylock"
// field: "UUID:threadId"
// value: 2  (重入了 2 次)
```

### 9.1.7 Q7：Redisson 的 RLock 和 Java 的 ReentrantLock 有什么区别？

| 对比项 | ReentrantLock | RLock |
|--------|--------------|-------|
| 范围 | 单 JVM 内 | 跨 JVM（分布式） |
| 实现 | CAS + AQS 队列 | Redis Lua 脚本 |
| 可重入 | ✅ | ✅ |
| 公平性 | ✅ 可选 | ✅ FairLock |
| 看门狗续期 | 不适用 | ✅ |
| 等待队列 | AQS CLH 队列 | Redis Pub/Sub |
| 性能 | 纳秒级 | 毫秒级（网络 IO） |

### 9.1.8 Q8：Redisson 的分布式集合（RMap）和 RedisTemplate 的 opsForHash 有何区别？

RMap 提供了类似 Java Map 的高级接口，包括：
- 原子操作（putIfAbsent、replace、remove 等）
- 过期支持（RMapCache）
- 监听器（EntryListener）
- 批量读写优化（fastPut、fastRemove）

RedisTemplate 的 opsForHash 则更加底层，需要手动组合命令。

### 9.1.9 Q9：如何处理 Redisson 锁的超时问题？

1. 明确业务耗时 → `lock(leaseTime, unit)` 指定锁超时，禁止看门狗。
2. 耗时不确定 → `lock()` 启用看门狗自动续期。
3. 防止死锁 → 永远在 finally 中 unlock()。
4. 获取锁失败 → `tryLock(waitTime, leaseTime, unit)` 避免无限等待。

### 9.1.10 Q10：Redisson 的线程安全如何保证？

- 基于 Netty 的异步非阻塞模型，所有网络 IO 在 EventLoop 中处理。
- 命令管道（CommandAsyncService）保证请求的顺序性。
- 内部使用大量的原子类和 CAS 操作。
- 连接池使用线程安全的连接复用机制。
