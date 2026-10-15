# 1. 概述与简介

## 1.1 什么是 Caffeine

Caffeine 是一个基于 Java 8+ 的高性能本地缓存库，由 Ben Manes 开发，深受 Google Guava Cache 的设计启发。在 Guava Cache 的基础上，Caffeine 引入了全新的 Window TinyLFU（W-TinyLFU）淘汰算法，在吞吐量和命中率上全面超越了 Guava Cache，目前已成为 Spring Boot 5.x 默认的本地缓存实现。

```
├─ Caffeine
│   ├─ 本地缓存（进程内缓存）
│   ├─ 高性能（接近 ConcurrentHashMap 的读写性能）
│   └─ 支持多种淘汰策略（大小、时间、引用）
```

## 1.2 设计目标与核心优势

- **高吞吐量**：读性能接近 ConcurrentHashMap，远高于 Guava Cache（8~10x）。
- **高命中率**：W-TinyLFU 淘汰算法在多种负载类型下命中率优于 LRU、LFU 和 FIFO。
- **低延迟**：平均读写延迟极低，适用于延迟敏感型应用。
- **零依赖**：仅依赖 JDK 标准库，无第三方依赖。
- **线程安全**：基于 ConcurrentHashMap 实现，完全线程安全。
- **自动加载**：支持 CacheLoader 自动加载和 Callable 手动加载。

## 1.3 与 Guava Cache 的关系

| 对比项 | Guava Cache | Caffeine |
|--------|-------------|----------|
| 淘汰算法 | LRU + 容量限制 | W-TinyLFU（自适应） |
| 异步缓存 | 不支持 | AsyncCache 支持 CompletableFuture |
| 引用淘汰 | 部分支持 | 完整支持 weakKeys/softValues |
| 统计信息 | 基本 | 详细（命中率、平均加载时间、淘汰计数等） |
| 性能 | 基准 | 8~10x 提升 |
| API 风格 | 相似 | 几乎一致（Caffeine 设计上兼容 Guava API） |

# 2. 核心功能与 API

## 2.1 Caffeine 构建器（Caffeine Builder）

Caffeine 采用 Builder 模式配置缓存实例。

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)                     // 最大条目数
    .expireAfterWrite(10, TimeUnit.MINUTES)  // 写入后过期
    .expireAfterAccess(5, TimeUnit.MINUTES)  // 访问后过期
    .refreshAfterWrite(1, TimeUnit.MINUTES)  // 写入后自动刷新
    .weakKeys()                              // 弱引用键
    .softValues()                            // 软引用值
    .recordStats()                           // 开启统计
    .build();
```

### 2.1.1 核心配置参数

**maximumSize(long size)**：基于缓存条目数量的最大容量。当缓存大小超过此值时，触发 W-TinyLFU 淘汰。

**maximumWeight(long weight, Weigher)**：基于权重的容量限制。

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumWeight(10_000_000)  // 最大权重和
    .weigher((key, value) -> {
        User user = (User) value;
        return user.getProfile().length();  // 按数据大小加权
    })
    .build();
```

**expireAfterWrite / expireAfterAccess**：基于时间的过期策略。expireAfterWrite 是绝对过期，expireAfterAccess 是空闲过期。

**refreshAfterWrite**：自动刷新，在过期之前异步重新加载，避免请求阻塞。

```java
// 每 1 分钟刷新一次，保留旧值直到新值加载完成
Caffeine.newBuilder()
    .refreshAfterWrite(1, TimeUnit.MINUTES)
    .build(new CacheLoader<String, Data>() {
        @Override
        public Data load(String key) {
            return loadFromDatabase(key);
        }

        @Override
        public CompletableFuture<Data> asyncReload(String key, Data oldValue) {
            return CompletableFuture.supplyAsync(() -> reloadFromDatabase(key));
        }
    });
```

**weakKeys / weakValues / softValues**：基于引用的淘汰。

| 配置 | 说明 | GC 回收条件 |
|------|------|-------------|
| weakKeys | 键使用 WeakReference | 键不再被外部强引用 |
| weakValues | 值使用 WeakReference | 值不再被外部强引用 |
| softValues | 值使用 SoftReference | JVM 内存不足时回收 |

> 注意：weakKeys 基于引用相等性（==）而非 equals() 比较，因此引用类型键须谨慎使用。

**recordStats()**：开启统计信息收集。

## 2.2 缓存类型

### 2.2.1 Cache（基础缓存）

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .build();

// 手动写入
cache.put("key1", user1);

// 获取，如果不存在返回 null
User user = cache.getIfPresent("key1");

// 获取，如果不存在通过 Callable 加载
User user = cache.get("key1", key -> loadFromDatabase(key));

// 批量获取
Map<String, User> result = cache.getAllPresent(Arrays.asList("key1", "key2"));

// 删除
cache.invalidate("key1");
cache.invalidateAll(Arrays.asList("key1", "key2"));
cache.invalidateAll();
```

### 2.2.2 LoadingCache（自动加载缓存）

```java
LoadingCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .build(new CacheLoader<String, User>() {
        @Override
        public User load(String key) throws Exception {
            return loadFromDatabase(key);
        }

        @Override
        public Map<String, User> loadAll(Iterable<? extends String> keys) {
            // 批量加载优化
            return loadAllFromDatabase(keys);
        }
    });

// 自动加载
User user = cache.get("key1");

// 批量自动加载
Map<String, User> users = cache.getAll(Arrays.asList("key1", "key2", "key3"));
```

### 2.2.3 AsyncCache（异步缓存）

```java
AsyncCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .buildAsync();

// 存入 CompletableFuture
CompletableFuture<User> future = cache.get("key1", key -> loadFromDatabase(key));

// 同步获取
User user = future.get(1, TimeUnit.SECONDS);

// 批量异步获取
CompletableFuture<Map<String, User>> futures = cache.getAllPresent(keys);

// 转换为同步视图
Cache<String, User> syncCache = cache.synchronous();
```

```java
// AsyncLoadingCache
AsyncLoadingCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .buildAsync(new CacheLoader<String, User>() {
        @Override
        public User load(String key) {
            return loadFromDatabase(key);
        }
    });

CompletableFuture<User> future = cache.get("key1");
```

## 2.3 移除监听器（Removal Listener）

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .removalListener((String key, User value, RemovalCause cause) -> {
        switch (cause) {
            case EXPLICIT:      // 手动 invalidate
                System.out.println("removed explicitly");
                break;
            case REPLACED:      // 被新值替换
                System.out.println("replaced");
                break;
            case COLLECTED:     // GC 回收（weakKeys/softValues）
                System.out.println("collected by GC");
                break;
            case EXPIRED:       // 过期
                System.out.println("expired");
                break;
            case SIZE:          // 容量淘汰
                System.out.println("evicted due to size");
                break;
        }
    })
    .build();
```

# 3. 高级特性

## 3.1 过期策略的高级定制

### 3.1.1 Expiry（自定义过期时间）

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfter(new Expiry<String, User>() {
        @Override
        public long expireAfterCreate(String key, User value, long currentTime) {
            // 不同 key 设置不同过期时间（纳秒）
            if (value.isVip()) {
                return TimeUnit.MINUTES.toNanos(30);
            }
            return TimeUnit.MINUTES.toNanos(10);
        }

        @Override
        public long expireAfterUpdate(String key, User value,
                                       long currentTime, long currentDuration) {
            // 更新时重置过期时间
            return currentDuration;
        }

        @Override
        public long expireAfterRead(String key, User value,
                                     long currentTime, long currentDuration) {
            // 读取时延长过期时间
            return Math.max(currentDuration, TimeUnit.MINUTES.toNanos(5));
        }
    })
    .build();
```

### 3.1.2 可变过期策略

通过 Expiry 接口可以实现：
- 每条缓存记录独立的 TTL
- 冷热数据差异化过期
- 写后变更过期策略

## 3.2 CacheWriter（缓存写入器）

CacheWriter 允许在缓存读写时同步到外部存储（如数据库、Redis），实现缓存旁路模式。

```java
LoadingCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .writer(new CacheWriter<String, User>() {
        @Override
        public void write(String key, User value) {
            // 同步写入数据库
            database.save(key, value);
        }

        @Override
        public void delete(String key, User value, RemovalCause cause) {
            // 同步删除数据库
            database.delete(key);
        }
    })
    .build(new CacheLoader<String, User>() {
        @Override
        public User load(String key) {
            return database.load(key);
        }
    });
```

## 3.3 Ticker（时钟自定义）

可自定义时间源，便于单元测试。

```java
// 测试中使用 Virtual Clock
Ticker testTicker = new Ticker() {
    private final AtomicLong nanos = new AtomicLong();

    @Override
    public long read() {
        return nanos.get();
    }

    public void advance(long duration, TimeUnit unit) {
        nanos.addAndGet(unit.toNanos(duration));
    }
};

Cache<String, User> cache = Caffeine.newBuilder()
    .ticker(testTicker::read)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();
```

## 3.4 调度方式

Caffeine 的过期间歇性清理依赖于读写操作驱动（惰性删除），而非后台线程定期扫描。

- **惰性删除**：读/写操作时检测并淘汰过期条目
- **定时兜底**：ScheduledExecutorService 定期清理（可自定义）

```java
// 自定义调度器
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

Cache<String, User> cache = Caffeine.newBuilder()
    .scheduler(Scheduler.forScheduledExecutorService(scheduler))
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();
```

## 3.5 统计信息

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .recordStats()
    .build();

// 获取统计
CacheStats stats = cache.stats();

stats.hitCount();            // 命中次数
stats.missCount();           // 未命中次数
stats.hitRate();             // 命中率
stats.missRate();            // 未命中率
stats.loadCount();           // 加载次数
stats.loadSuccessCount();    // 成功加载次数
stats.loadFailureCount();    // 加载失败次数
stats.loadFailureRate();     // 加载失败率
stats.totalLoadTime();       // 总加载时间（纳秒）
stats.averageLoadPenalty();  // 平均加载耗时
stats.evictionCount();       // 淘汰次数
stats.evictionWeight();      // 淘汰总权重
stats.requestCount();        // 总请求次数
```

```java
// 重置统计
stats.snapshot();  // 获取当前快照，不影响统计继续累积
```

# 4. 底层原理与设计

## 4.1 W-TinyLFU 淘汰算法

W-TinyLFU（Window Tiny Least Frequently Used）是 Caffeine 的核心竞争力，由三个组件构成：

```
┌─────────────────────────────────────────────────────────┐
│                    W-TinyLFU 架构                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   Admission Window (1%)       Main Space (99%)           │
│   ┌────────────────────┐     ┌────────────────────────┐  │
│   │  LRU (准入窗口)     │ ──► │  LRU (试用区)          │  │
│   │  新数据先进入此处    │     │                        │  │
│   └────────────────────┘     │  ┌──────────────────┐  │  │
│        │ evict               │  │  LRU 保护区       │  │  │
│        ▼                     │  │  频繁访问的数据    │  │  │
│   Frequency Sketch           │  └──────────────────┘  │  │
│   (频率估算过滤器)  ◄────────│  │                        │  │
│                              │  └───────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 4.1.1 Admission Window（准入窗口）

- **大小**：占整个缓存大小的 1%（默认配置）。
- **策略**：纯 LRU（最近最少使用）。
- **目的**：避免新数据因访问次数少而被立即淘汰。新数据先进入该窗口，即使早期访问频率低也能保留一段时间。

### 4.1.2 Main Space（主缓存空间）

- **大小**：占整个缓存大小的 99%。
- **内部结构**：使用 Segmented LRU（SLRU）分两段。

| 分段 | 比例 | 说明 |
|------|------|------|
| Probation（试用区） | 约 20% | 刚进入主空间的候选者 |
| Protected（保护区） | 约 80% | 被频繁访问的数据 |

- **晋升规则**：Probation 区的数据被再次访问时，晋升到 Protected 区。
- **降级规则**：Protected 区满时，最久未访问的数据降级回 Probation 区。

### 4.1.3 Frequency Sketch（频率草图）

- **算法**：4-bit CountMin Sketch（计数最小草图）。
- **作用**：以极低的存储开销估算元素的访问频率。
- **特点**：
  - 概率性数据结构，允许一定程度的哈希冲突。
  - 每个计数器仅用 4 位，溢出后取最大值 15。
  - 时间复杂度 O(1)，空间效率极高。
  - 支持频率衰减，适应访问模式的变化（使用 reset 机制，定期将计数器值减半）。

**CountMin Sketch 原理**：

```
插入元素 key：对 key 计算多个哈希函数 h1(key), h2(key), h3(key), h4(key)
分别映射到 4 个数组的对应位置，将计数 +1。

查询频率：取 min(arr1[h1(key)], arr2[h2(key)], arr3[h3(key)], arr4[h4(key)])
```

### 4.1.4 淘汰决策流程

```
1. 新数据进入 Admission Window（LRU）
2. Window 满 → 窗口尾部的数据（最久未访问）被驱逐
3. 驱逐候选者与 Main Space（Probation 区）的尾部数据竞争
4. 通过 Frequency Sketch 比较两者的访问频率：
   - 如果候选者频率更高 → 淘汰 Main Space 中的数据，候选者进入 Probation
   - 否则 → 丢弃候选者
```

**这种设计的核心思想**：新数据需要先积累足够的频率证据才能进入主缓存空间，避免突发性流量污染缓存。（区别于 LRU 的"扫描抵抗"能力弱的问题。）

## 4.2 为什么 Caffeine 比 Guava Cache 快

| 优化点 | 说明 |
|--------|------|
| 缓存行填充（@Contended） | 减少伪共享（False Sharing）对多核 CPU 的影响 |
| Mild 加锁策略 | 大量读操作使用乐观读（类似于 ConcurrentHashMap 的读无锁） |
| W-TinyLFU vs LRU | LRU 在循环扫描场景下命中率急剧下降，W-TinyLFU 则能保持高命中率 |
| 懒加载+惰性淘汰 | 避免定时线程空转和锁竞争 |
| 内存布局优化 | 使用对象池（ObjectPool）减少 GC 压力 |
| 异步加载 | 充分利用多核 CPU |

### 4.2.1 伪共享优化

Caffeine 使用 `@jdk.internal.vm.annotation.Contended` 或 `sun.misc.Contended` 注解，确保频繁修改的计数器和状态变量位于独立的缓存行中，避免多核 CPU 缓存一致性协议（MESI）导致的性能下降。

```
┌──── 64 字节 ────┬──── 64 字节 ────┬──── 64 字节 ────┐
│   Core 1 数据    │  padding        │   Core 2 数据    │
│   (修改频繁)      │  (填充物)       │   (修改频繁)      │
└────────────────┴────────────────┴────────────────┘
```

## 4.3 ConcurrentHashMap 为基础存储

Caffeine 内部存储基于 ConcurrentHashMap，充分利用其分段锁（或 JDK 8+ 的桶锁+CAS）机制实现高效的并发安全。

```java
// 简化示意：Caffeine 内部存储结构
ConcurrentHashMap<K, Node<K, V>> data;

class Node<K, V> {
    K key;
    V value;
    volatile long accessTime;      // 访问时间
    volatile long writeTime;       // 写入时间
    volatile int accessCount;      // 访问次数
    Node<K, V> prevInQueue;       // 双链表指针（用于 LRU 队列）
    Node<K, V> nextInQueue;
}
```

## 4.4 过期机制

Caffeine 的过期检测没有使用后台定时线程（默认），而是在每个缓存操作时抽样检查。

- **写操作**：检查是否过期，过期则移除。
- **读操作**：检查是否过期，过期则移除。
- **定期兜底**：如果配置了 scheduler，会有一个定时任务周期打扫。
- **惰性删除**：客户端即使有过期数据，也会在下一次访问时被清理，不会影响一致性。

# 5. 性能优化

## 5.1 配置建议

### 5.1.1 容量设置

```java
// 根据实际数据量和内存预算设置
// 使用 JProfiler / YourKit 等工具分析内存占用
Cache<String, LargeObject> cache = Caffeine.newBuilder()
    .maximumSize(1_000)  // 估算：每条记录 ~10KB，总内存 ~10MB
    .build();
```

### 5.1.2 权重缓存

当缓存对象大小不一时，按权重淘汰比按数量淘汰更精确。

```java
Cache<String, byte[]> cache = Caffeine.newBuilder()
    .maximumWeight(100 * 1024 * 1024)  // 100MB
    .weigher((String key, byte[] value) -> value.length)
    .build();
```

### 5.1.3 初始容量

```java
// 预估容量，减少 ConcurrentHashMap 扩容开销
Cache<String, User> cache = Caffeine.newBuilder()
    .initialCapacity(10_000)  // 初始容量
    .maximumSize(100_000)
    .build();
```

## 5.2 异步加载最佳实践

```java
AsyncLoadingCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .buildAsync(new CacheLoader<String, User>() {
        @Override
        public User load(String key) {
            return slowDatabaseQuery(key);  // 200ms
        }

        @Override
        public CompletableFuture<User> asyncLoad(String key, Executor executor) {
            return CompletableFuture.supplyAsync(() -> load(key), executor);
        }
    });

// 并发请求相同 key 只触发一次加载
CompletableFuture<User> f1 = cache.get("key1");
CompletableFuture<User> f2 = cache.get("key1");  // 复用同一个 Future
```

## 5.3 避免缓存污染

使用 refreshAfterWrite 代替 expireAfterWrite 可避免缓存雪崩（大量 key 同时过期导致的数据库压力）。

```java
// 推荐组合：短刷新 + 长过期
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)   // 兜底过期
    .refreshAfterWrite(1, TimeUnit.MINUTES)   // 频繁刷新
    .build(new CacheLoader<String, User>() {
        @Override
        public User load(String key) {
            return loadFromDatabase(key);
        }
    });
```

## 5.4 Executor 自定义

为异步操作指定自定义线程池，避免使用 ForkJoinPool.commonPool()。

```java
Executor executor = new ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadFactoryBuilder().setNameFormat("caffeine-%d").build()
);

AsyncCache<String, User> cache = Caffeine.newBuilder()
    .executor(executor)
    .buildAsync();
```

# 6. 与 Spring Boot 集成

## 6.1 自动配置

Spring Boot 2.x+ 默认将 Caffeine 列为本地缓存的推荐实现。只需添加依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<!-- Caffeine 被 Spring Boot 自动引入 -->
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        // 默认配置
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats());
        // 允许动态创建缓存（@Cacheable 中如果没定义会自动创建）
        cacheManager.setCacheNames(Arrays.asList("users", "orders"));
        return cacheManager;
    }
}

// 或者使用配置文件
```

## 6.2 application.yml 配置

```yaml
spring:
  cache:
    type: caffeine
    cache-names: users,orders,products
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=10m,recordStats
```

spec 支持的参数：

| 参数 | 缩写 | 示例 |
|------|------|------|
| maximumSize | max | maximumSize=10000 |
| maximumWeight | 无 | maximumWeight=100000 |
| expireAfterWrite | expireAfterWrite | expireAfterWrite=10m |
| expireAfterAccess | expireAfterAccess | expireAfterAccess=5m |
| refreshAfterWrite | refreshAfterWrite | refreshAfterWrite=1m |
| weakKeys | weakKeys | weakKeys=true |
| softValues | softValues | softValues=true |
| recordStats | recordStats | recordStats=true |

时间单位：ns（纳秒）、us（微秒）、ms（毫秒）、s（秒）、m（分钟）、h（小时）、d（天）

## 6.3 @Cacheable 用法

```java
@Service
public class UserService {

    @Cacheable(
        value = "users",          // 缓存名称 → Caffeine 实例
        key = "#userId",          // SpEL 表达式指定 key
        condition = "#userId > 0",// 条件缓存
        unless = "#result == null"// 不缓存的条件
    )
    public User getUserById(Long userId) {
        return userMapper.selectById(userId);  // 查询数据库
    }

    // 多个 key 组合
    @Cacheable(value = "users", key = "#name + '_' + #age")
    public User getUserByNameAndAge(String name, int age) {
        return userMapper.selectByNameAndAge(name, age);
    }

    // 复杂 SpEL
    @Cacheable(value = "users", key = "#user.id", condition = "#user.role == 'VIP'")
    public User saveUser(User user) {
        userMapper.insert(user);
        return user;
    }
}
```

## 6.4 @CacheEvict 用法

```java
@CacheEvict(value = "users", key = "#id")  // 删除指定缓存
public void deleteUser(Long id) {
    userMapper.deleteById(id);
}

@CacheEvict(value = "users", allEntries = true)  // 清空所有 users 缓存
public void clearUserCache() {
    // 业务逻辑
}

@CacheEvict(value = "users", key = "#id", beforeInvocation = true)  // 方法执行前清除
public void updateUser(Long id, User user) {
    userMapper.updateById(user);
}
```

## 6.5 @CachePut 用法

```java
@CachePut(value = "users", key = "#user.id")  // 方法执行后更新缓存，不阻塞方法执行
public User updateUser(User user) {
    userMapper.updateById(user);
    return user;
}
```

## 6.6 @Caching 组合注解

```java
@Caching(
    cacheable = @Cacheable(value = "users", key = "#id"),
    evict = {
        @CacheEvict(value = "userList", allEntries = true),
        @CacheEvict(value = "userStats", key = "#id")
    }
)
public User getUserWithEvict(Long id) {
    return userMapper.selectById(id);
}
```

## 6.7 自定义缓存配置（每个缓存不同策略）

```java
@Configuration
@EnableCaching
public class CacheConfig {

    public static final String CACHE_USERS = "users";
    public static final String CACHE_ORDERS = "orders";
    public static final String CACHE_PRODUCTS = "products";
    public static final String CACHE_CONFIG = "config";

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();

        // 配置不同缓存的个性化策略
        List<CaffeineCache> caches = new ArrayList<>();

        caches.add(buildCache(CACHE_USERS, 10_000, 30, TimeUnit.MINUTES));
        caches.add(buildCache(CACHE_ORDERS, 5_000, 60, TimeUnit.MINUTES));
        caches.add(buildCache(CACHE_PRODUCTS, 20_000, 10, TimeUnit.MINUTES));
        caches.add(buildCache(CACHE_CONFIG, 1_000, 24, TimeUnit.HOURS));

        // 自定义 SimpleCacheManager 或在 CaffeineCacheManager 基础上改造
        // Spring 官方 CaffeineCacheManager 不支持为每个缓存单独配置
        // 可通过自定义 CacheManager 实现：

        SimpleCacheManager simpleCacheManager = new SimpleCacheManager();
        simpleCacheManager.setCaches(caches);
        return simpleCacheManager;
    }

    private CaffeineCache buildCache(String name, int maxSize,
                                      int duration, TimeUnit unit) {
        return new CaffeineCache(name, Caffeine.newBuilder()
            .maximumSize(maxSize)
            .expireAfterWrite(duration, unit)
            .recordStats()
            .build());
    }
}
```

或者使用 Spring Cache 的 Composite 方式 + Redis 二级缓存：

```java
@Bean
public CacheManager cacheManager() {
    // 本地缓存（一级）
    CaffeineCacheManager local = new CaffeineCacheManager();
    local.setCaffeine(Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.MINUTES));

    // Redis 缓存（二级）
    RedisCacheManager redis = RedisCacheManager.builder(redisConnectionFactory).build();

    return new CompositeCacheManager(local, redis);
}
```

## 6.8 异步缓存 + Spring

```java
@Async
@Cacheable(value = "users", key = "#id")
public CompletableFuture<User> getUserAsync(Long id) {
    return CompletableFuture.completedFuture(userMapper.selectById(id));
}
```

# 7. 最佳实践与常见问题

## 7.1 缓存穿透

**问题**：缓存和数据中都不存在该 key，每次请求都直达数据库。

**解决方案**：

```java
// 1. 缓存空值（占位符）
@Cacheable(value = "users", unless = "#result == null")
public User getUser(Long id) {
    User user = userMapper.selectById(id);
    if (user == null) {
        // 缓存空对象，TTL 设短一些
        return new User();  // 空对象占位
    }
    return user;
}

// 2. Bloom Filter 预处理（参见 Redisson 方案）
```

## 7.2 缓存雪崩

**问题**：大量缓存同时过期，请求全部打到数据库。

**解决方案**：

```java
// 1. 使用 refreshAfterWrite 分摊过期时间
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .refreshAfterWrite(1, TimeUnit.MINUTES)
    .build();

// 2. 过期时间加随机偏移
// Caffeine 不支持直接设定随机过期，可在应用层面实现：
Cache<String, User> cache = Caffeine.newBuilder()
    .expireAfter(new Expiry<String, User>() {
        @Override
        public long expireAfterCreate(String key, User value, long currentTime) {
            long baseTtl = TimeUnit.MINUTES.toNanos(10);
            long randomOffset = ThreadLocalRandom.current().nextLong(TimeUnit.MINUTES.toNanos(5));
            return baseTtl + randomOffset;
        }
        // ...
    })
    .build();
```

## 7.3 缓存击穿

**问题**：热点 key 在过期瞬间被大量并发请求穿透。

**解决方案**：

```java
// Caffeine 的 get 方法自带锁机制，同一 key 同时只会有一个请求加载
// LoadingCache 天然防止缓存击穿
User user = loadingCache.get("hotKey");

// 也可以显式使用 synchronized 或分布式锁
```

## 7.4 热点缓存

**问题**：某些 key 被频繁访问，需要特殊保护。

**解决方案**：Caffeine 的 W-TinyLFU 自动将热点数据晋升到 Protected 区，无需手动干预。但可以配合分层缓存策略——Caffeine 作为 L1 本地缓存，Redis 作为 L2 分布式缓存。

## 7.5 缓存一致性

### 7.5.1 更新策略

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| Cache-Aside | 读→Cache Miss→读 DB→写 Cache | 通用 |
| Write-Through | 写 Cache→写 DB（通过 CacheWriter） | 强一致性需求 |
| Write-Behind | 写 Cache→异步写 DB | 写性能优先 |

### 7.5.2 Cache-Aside 模式下的更新顺序

```java
// 正确的更新操作：
// 1. 更新 DB
// 2. 删除缓存（而非更新缓存）
// 原因：避免并发写导致缓存与数据库数据不一致

@CacheEvict(value = "users", key = "#user.id")
public User updateUser(User user) {
    userMapper.updateById(user);
    return user;
}
```

## 7.6 常用监控

```java
// Spring Boot Actuator + Caffeine Stats
@RestController
public class CacheMonitorController {

    @Autowired
    private CacheManager cacheManager;

    @GetMapping("/cache/stats")
    public Map<String, Object> getCacheStats() {
        Map<String, Object> result = new HashMap<>();

        for (String name : cacheManager.getCacheNames()) {
            CaffeineCache cache = (CaffeineCache) cacheManager.getCache(name);
            CacheStats stats = cache.getNativeCache().stats();
            Map<String, Object> statMap = new HashMap<>();
            statMap.put("hitRate", stats.hitRate());
            statMap.put("missRate", stats.missRate());
            statMap.put("averageLoadPenalty", stats.averageLoadPenalty() / 1_000_000); // ms
            statMap.put("evictionCount", stats.evictionCount());
            statMap.put("requestCount", stats.requestCount());
            result.put(name, statMap);
        }

        return result;
    }
}
```

## 7.7 常见误区

### 7.7.1 误区 1：混淆 expireAfterWrite 和 refreshAfterWrite

- `expireAfterWrite`：过期后彻底移除，下次访问同步阻塞加载。
- `refreshAfterWrite`：过期后异步加载新值，加载期间返回旧值（如果旧值已过期则阻塞加载）。

### 7.7.2 误区 2：弱引用键的使用

```java
// 错误：weakKeys 后使用 String 类型的 key
// weakKeys 意味着使用 == 比较而非 equals()
// String 字面量可能 == 相等，但 new String() 对象则不一定
cache = Caffeine.newBuilder().weakKeys().build();
cache.put(new String("key1"), value1);
cache.getIfPresent(new String("key1"));  // → null，因为不是同一个引用

// 正确：weakKeys 适合使用业务对象作为 key 且确保引用相同的场景
// 或者使用 Integer/Long 等享元模式的对象
```

### 7.7.3 误区 3：在大对象场景下设置过大的 maximumSize

```java
// 错误：不考虑内存
cache = Caffeine.newBuilder().maximumSize(100_000_000).build();  // 1 亿条→内存溢出

// 正确：基于内存预算估算
// 预算 1GB，每条对象平均 1KB → 约 100 万条
```

# 8. 与其他技术对比

## 8.1 Caffeine vs Guava Cache

| 对比项 | Guava Cache | Caffeine |
|--------|-------------|----------|
| 淘汰算法 | LRU（访问顺序队列） | W-TinyLFU |
| 并发性能 | ~10M ops/s | ~50-60M ops/s |
| 避免扫描污染 | 弱 | 强 |
| 异步缓存 | ❌ | ✅ |
| 自定义过期策略 | ❌ | ✅ Expiry |
| CacheWriter | ❌ | ✅ |
| 内存 footprint | 较大 | 更小 |
| API 兼容性 | — | 与 Guava 接口兼容 |

## 8.2 Caffeine vs Ehcache

| 对比项 | Ehcache | Caffeine |
|--------|---------|----------|
| 定位 | 全功能缓存（支持磁盘持久化、集群） | 纯内存本地缓存 |
| 依赖性 | 大（XML 配置、Tx 支持） | 极轻量（零依赖） |
| 性能 | 中 | 极高 |
| 分布式 | 支持（Terracotta） | 不支持 |
| 堆外存储 | 支持 | 不支持 |
| 适用场景 | 企业级、需要持久化的场景 | 高性能本地缓存 |
| Spring Boot 支持 | 通过 JCache | 原生无缝对接 |

## 8.3 Caffeine vs Redis（本地缓存视角）

| 对比项 | Caffeine | Redis |
|--------|----------|-------|
| 吞吐量 | 微秒级（内存读取） | 毫秒级（网络 IO） |
| 数据类型 | 键值对 | 丰富（String/Hash/Set/SortedSet/List） |
| 持久化 | 不支持（进程内内存） | RDB/AOF |
| 分布式 | 单机 | 原生分布式 |
| 容量 | 受限于 JVM 堆内存 | 受限于服务器内存 |
| 适用场景 | 热点数据本地缓存（L1） | 分布式共享缓存（L2） |

最佳组合：Caffeine（L1 本地缓存）+ Redis（L2 分布式缓存）

```java
// 两级缓存示意
public class TwoLevelCache<K, V> {

    private final Cache<K, V> localCache;   // Caffeine
    private final RedisTemplate<K, V> redis; // Redis

    public V get(K key) {
        V value = localCache.getIfPresent(key);
        if (value != null) return value;

        value = redis.opsForValue().get(key);
        if (value != null) {
            localCache.put(key, value);
            return value;
        }

        value = loadFromDB(key);
        localCache.put(key, value);
        redis.opsForValue().set(key, value);
        return value;
    }
}
```

## 8.4 Caffeine vs Cache2k

| 对比项 | Cache2k | Caffeine |
|--------|---------|----------|
| API 风格 | 独有 API | 兼容 Guava API |
| 淘汰算法 | LRU + 自适应 | W-TinyLFU |
| 堆外存储 | 支持 | 不支持 |
| 吞吐量 | 相近 | 相近或略优 |
| 社区生态 | 较小 | 大（Spring 默认） |

# 9. 面试要点

## 9.1 高频面试题

### 9.1.1 Q1：Caffeine 为什么比 Guava Cache 快？

**核心要点**：
1. W-TinyLFU 替代 LRU，有更高的命中率，减少回源次数。
2. 内部使用类似 ConcurrentHashMap 的锁优化 + 缓存行填充，减少 CPU 伪共享。
3. 惰性淘汰机制避免后台线程的竞争开销。
4. 异步加载支持 CompletableFuture，更好地利用多核。

### 9.1.2 Q2：W-TinyLFU 算法原理是什么？

1. **准入窗口**：1% 的 LRU 窗口，新数据先进来积累频率。
2. **主空间**：99% 的 SLRU 分 Probation 和 Protected 区。
3. **频率草图**：CountMin Sketch 以极低内存开销估算访问频率。
4. **淘汰决策**：窗口驱逐 + 主空间尾部竞争，频率高者存活。

### 9.1.3 Q3：Caffeine 过期机制是如何实现的？

1. 默认没有后台打扫线程（可配置 Scheduler）。
2. 读写操作时惰性检测并删除过期数据。
3. 使用 O(1) 时间复杂度的哈希访问 + 双链表维护时间顺序。
4. 过期时间存储在 Node 节点的 writeTime / accessTime 字段。

### 9.1.4 Q4：Caffeine 如何防止缓存击穿？

1. Cache.get(key, Function) 内部对同一 key 的并发加载做了同互斥（类似 Map.computeIfAbsent）。
2. LoadingCache 的 get 方法会同步阻塞直到值加载完成。
3. AsyncCache 返回 CompletableFuture，重复请求复用同一个 Future。

### 9.1.5 Q5：Caffeine 支持的淘汰策略有哪些？

1. **大小淘汰**：maximumSize / maximumWeight
2. **时间淘汰**：expireAfterWrite / expireAfterAccess / 自定义 Expiry
3. **引用淘汰**：weakKeys / weakValues / softValues
4. **手动淘汰**：invalidate / invalidateAll

### 9.1.6 Q6：Caffeine 的 recordStats 有哪些指标？

- hitRate / missRate
- loadCount / loadSuccessCount / loadFailureCount
- totalLoadTime / averageLoadPenalty
- evictionCount / evictionWeight
- requestCount

### 9.1.7 Q7：Caffeine 的 refreshAfterWrite 和 expireAfterWrite 的区别？

- refreshAfterWrite：到期后异步刷新，返回旧值（不会阻塞），但若数据已被 expireAfterWrite 淘汰则阻塞加载。
- expireAfterWrite：到期后彻底删除，所有请求同步等待加载。
- 推荐组合使用：expireAfterWrite 更长作为兜底，refreshAfterWrite 更短作为热刷新。

### 9.1.8 Q8：如何在 Spring Boot 中配置 Caffeine 缓存？

```
// Maven:
spring-boot-starter-cache
// yml:
spring.cache.type=caffeine
spring.cache.caffeine.spec=maximumSize=10000,expireAfterWrite=10m
// @EnableCaching + @Cacheable
```

### 9.1.9 Q9：Caffeine 的软引用（softValues）使用场景？

适用于堆内存较大且愿意以性能换内存的场景：
- 允许 GC 回收不常用的缓存值以释放内存。
- 适合缓存结果较大的对象（如大图片、大文件内容）。
- 缺点是与垃圾回收器的交互增加了延迟。

### 9.1.10 Q10：Caffeine 和 Redis 如何配合使用？

两级缓存架构：
- L1：Caffeine 本地缓存（微秒级，存储热点数据）。
- L2：Redis 分布式缓存（毫秒级，存储全量缓存数据）。
- 更新策略：写 DB → 删除 L2 → 删除 L1。
- 一致性：通过消息队列广播失效通知保持 L1 小范围一致。
