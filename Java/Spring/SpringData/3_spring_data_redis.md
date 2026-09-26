# 1.Spring Data Redis 概述

## 1.1 什么是 Spring Data Redis

Spring Data Redis 是 Spring Data 家族中用于访问 Redis 的模块，它提供了与 Redis 交互的便捷方式：

- 封装了底层 Redis 客户端（Jedis/Lettuce），屏蔽驱动差异
- 提供 `RedisTemplate` 和 `StringRedisTemplate` 作为操作入口
- 支持 Redis 全部数据结构：String、List、Set、ZSet、Hash、HyperLogLog、Geo、Stream 等
- 提供了 Spring Cache 抽象的 Redis 实现
- 支持 Redis Sentinel 和 Redis Cluster 集群模式
- 提供 Reactive（响应式）编程支持
- 提供 Repository 编程模型（Redis Repositories）

## 1.2 Redis 驱动选择：Jedis vs Lettuce

| 特性 | Jedis | Lettuce |
|------|-------|---------|
| 连接方式 | 连接池（每个实例独立连接） | 基于 Netty，单连接共享 |
| 线程安全 | 非线程安全（需连接池） | 原生线程安全 |
| 同步/异步 | 仅同步 | 同步 + 异步 + Reactive |
| 连接数 | 每线程一个连接 | 少量连接即可 |
| Spring Boot 2.x+ 默认 | 否 | 是（默认驱动） |
| 社区活跃度 | 较老牌 | 更现代，Spring 官方推荐 |

Spring Boot 2.x 之后默认使用 Lettuce 作为 Redis 客户端。如需使用 Jedis：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

## 1.3 Spring Data Redis 整体架构

```
                                                     Spring Cache 抽象
┌────────────────────────────────────────┐     ┌─────────────────────┐
│         RedisRepository               │     │ @Cacheable          │
│   (类似 JPA Repository 编程模型)        │     │ @CacheEvict         │
└────────────────┬───────────────────────┘     │ @CachePut           │
                 │                             └─────────┬───────────┘
                 │                                       │
┌────────────────▼───────────────────────────────────────▼───────────┐
│                    RedisTemplate / StringRedisTemplate             │
│                    (核心操作入口，封装序列化和连接管理)                  │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                    RedisConnectionFactory                          │
│               (LettuceConnectionFactory / JedisConnectionFactory)   │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                    Redis Client (Lettuce / Jedis)                   │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                          Redis Server                               │
└────────────────────────────────────────────────────────────────────┘
```

# 2.快速入门

## 2.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- 连接池依赖（Lettuce 使用 commons-pool2） -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

## 2.2 基础配置

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: 123456         # 生产环境不要明文配置
      database: 0              # 使用的数据库编号
      timeout: 3000ms          # 连接超时时间
      username: default        # Redis 6.0+ 支持 ACL
      ssl:
        enabled: false

      # Lettuce 连接池配置
      lettuce:
        pool:
          max-active: 16       # 最大活跃连接数
          max-idle: 8          # 最大空闲连接数
          min-idle: 4          # 最小空闲连接数
          max-wait: 2000ms     # 获取连接最大等待时间
          time-between-eviction-runs: 30s

        # Lettuce 集群拓扑刷新（Cluster 模式重要！）
        cluster:
          refresh:
            adaptive: true     # 自适应刷新
            period: 30s        # 定时刷新周期

      # 如果使用 Jedis：
      # jedis:
      #   pool:
      #     max-active: 16
      #     max-idle: 8
      #     min-idle: 4
```

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // 设置序列化器
        Jackson2JsonRedisSerializer<Object> jacksonSerializer =
            new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL);
        jacksonSerializer.setObjectMapper(om);

        // Key 使用 String 序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value 使用 JSON 序列化
        template.setValueSerializer(jacksonSerializer);
        template.setHashValueSerializer(jacksonSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

## 2.3 第一个 Redis 操作

```java
@Service
public class RedisDemoService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;

    public RedisDemoService(RedisTemplate<String, Object> redisTemplate,
                             StringRedisTemplate stringRedisTemplate) {
        this.redisTemplate = redisTemplate;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    // 使用 StringRedisTemplate（Key/Value 都是 String）
    public void demo1() {
        ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
        ops.set("hello", "world");
        String value = ops.get("hello");
        System.out.println(value);  // world
    }

    // 使用 RedisTemplate 存储对象
    public void demo2() {
        User user = new User(1L, "张三", "zhangsan@example.com");
        redisTemplate.opsForValue().set("user:1", user);

        User cached = (User) redisTemplate.opsForValue().get("user:1");
        System.out.println(cached.getName());  // 张三
    }
}
```

# 3.RedisTemplate 详解

## 3.1 RedisTemplate vs StringRedisTemplate

| 特性 | RedisTemplate<K, V> | StringRedisTemplate |
|------|-------------------|--------------------|
| 泛型 | 通用 K/V 泛型 | 固定 String/String |
| 序列化器 | 默认 JDK 序列化（不推荐） | String 序列化 |
| 使用场景 | 对象存储（需配置 JSON 序列化） | 纯字符串操作 |
| 可读性 | 默认存到 Redis 不可读（二进制） | 直观可读 |

**为什么要配置序列化器**：

```java
// 默认 JDK 序列化存在的问题：
// 1. Redis 中看到的是乱码：\xAC\xED\x00\x05t\x00\x04user
// 2. 不同 JVM 版本可能不兼容
// 3. 无法被其他语言程序读取
// 4. 占用空间大

// 解决：使用 JSON 序列化器（Jackson/Gson/Fastjson）
```

## 3.2 序列化器对比

| 序列化器 | 说明 | Redis 可读性 | 跨语言 |
|---------|------|-------------|--------|
| `JdkSerializationRedisSerializer` | 默认，JDK 序列化 | 否 | 否 |
| `StringRedisSerializer` | 字符串序列化 | 是 | 是 |
| `Jackson2JsonRedisSerializer` | JSON 序列化，需指定类型 | 是 | 是 |
| `GenericJackson2JsonRedisSerializer` | 通用 JSON 序列化，带 @class | 是 | 是 |
| `FastJsonRedisSerializer` | 阿里 FastJson 序列化 | 是 | 是 |

**Jackson2JsonRedisSerializer vs GenericJackson2JsonRedisSerializer**：

```java
// Jackson2JsonRedisSerializer：
// - 需指定目标类型：new Jackson2JsonRedisSerializer(User.class)
// - 不能存储不同类型的对象用同一个 template
// - JSON 中不包含 @class 信息，更轻量
// - 适合同一 application 内使用

// GenericJackson2JsonRedisSerializer：
// - 自动在 JSON 中添加 @class 字段
// - 可以存储任意类型，反序列化时自动识别
// - JSON 中包含类信息，占用空间更大
// - 存在反序列化安全风险，需配置白名单
```

## 3.3 各种序列化配置

```java
@Configuration
public class RedisSerializerConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // === 方式1：通用 JSON 序列化（推荐） ===
        GenericJackson2JsonRedisSerializer jsonSerializer =
            new GenericJackson2JsonRedisSerializer();
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        template.setKeySerializer(stringSerializer);
        template.setValueSerializer(jsonSerializer);
        template.setHashKeySerializer(stringSerializer);
        template.setHashValueSerializer(jsonSerializer);

        // === 方式2：使用 FastJson ===
        // FastJsonRedisSerializer<Object> fastJsonSerializer =
        //     new FastJsonRedisSerializer<>(Object.class);
        // template.setValueSerializer(fastJsonSerializer);

        // === 方式3：使用 ProtoStuff（高性能二进制序列化）===
        // template.setValueSerializer(new ProtoStuffSerializer());

        template.afterPropertiesSet();
        return template;
    }
}
```

# 4.数据类型操作详解

## 4.1 String（字符串）操作

```java
// 获取 String 操作接口
ValueOperations<String, Object> valueOps = redisTemplate.opsForValue();

// SET：设置值
valueOps.set("key1", "hello");
valueOps.set("key2", "world", Duration.ofMinutes(10));  // 带过期时间

// SETNX：仅当 key 不存在时设置（分布式锁的基础）
Boolean success = valueOps.setIfAbsent("lock:order:1001", "thread-1",
    Duration.ofSeconds(30));

// GET：获取值
Object value = valueOps.get("key1");

// MGET：批量获取
List<Object> values = valueOps.multiGet(Arrays.asList("key1", "key2"));

// MSET：批量设置
Map<String, Object> map = new HashMap<>();
map.put("key3", "v3");
map.put("key4", "v4");
valueOps.multiSet(map);

// INCR / DECR：原子增减（计数器场景）
Long count = valueOps.increment("article:view:1001");      // +1
Long count2 = valueOps.increment("article:view:1001", 10); // +10
Long count3 = valueOps.decrement("inventory:sku:999");     // -1

// GETSET：设置新值并返回旧值
Object oldValue = valueOps.getAndSet("key1", "newValue");

// APPEND：追加字符串
Integer newLen = valueOps.append("key1", "-suffix");

// STRLEN：字符串长度
Long len = valueOps.size("key1");

// GETRANGE：获取子字符串
String sub = valueOps.get("key1", 0, 4);

// SETEX 等价于 set + Duration
valueOps.set("token:abc123", "active", Duration.ofHours(2));

// GETBIT / SETBIT：位图操作
Boolean bit = valueOps.getBit("user:sign:2024-01", 15);
valueOps.setBit("user:sign:2024-01", 15, true);
```

## 4.2 List（列表）操作

```java
ListOperations<String, Object> listOps = redisTemplate.opsForList();

// LPUSH / RPUSH：从左/右入队
Long len = listOps.leftPush("queue:tasks", "task1");
Long len2 = listOps.rightPush("queue:tasks", "task2");

// LPUSH 批量
List<String> tasks = Arrays.asList("task3", "task4", "task5");
listOps.leftPushAll("queue:tasks", tasks);

// LPOP / RPOP：从左/右出队
Object task = listOps.leftPop("queue:tasks");     // 阻塞版本见 BLPop
Object task2 = listOps.rightPop("queue:tasks");

// LRANGE：按范围获取
List<Object> range = listOps.range("queue:tasks", 0, -1);  // 获取全部

// LLEN：列表长度
Long size = listOps.size("queue:tasks");

// LINDEX：按索引获取
Object element = listOps.index("queue:tasks", 0);

// LREM：按值删除
Long removed = listOps.remove("queue:tasks", 1, "task3");

// LSET：按索引设置值
listOps.set("queue:tasks", 0, "updatedTask");

// LTRIM：修剪列表（保留指定范围，其余删除）
listOps.trim("queue:tasks", 0, 99);  // 保留前 100 条
```

**List 与消息队列场景**：

```java
// 生产者-消费者模型
// 生产者
redisTemplate.opsForList().leftPush("order:queue", orderJson);
// 消费者
while (true) {
    // BLPOP：阻塞式左弹出（Lettuce 支持）
    Object order = redisTemplate.opsForList().rightPop("order:queue", 5, TimeUnit.SECONDS);
    if (order != null) {
        processOrder(order);
    }
}
```

## 4.3 Set（集合）操作

```java
SetOperations<String, Object> setOps = redisTemplate.opsForSet();

// SADD：添加元素
Long added = setOps.add("tags:article:1001", "Java", "Spring", "Redis");

// SREM：删除元素
Long removed = setOps.remove("tags:article:1001", "Redis");

// SMEMBERS：获取所有成员
Set<Object> members = setOps.members("tags:article:1001");

// SISMEMBER：判断是否成员
Boolean isMember = setOps.isMember("tags:article:1001", "Java");

// SCARD：获取成员数量
Long count = setOps.size("tags:article:1001");

// SINTER：交集（共同关注）
Set<Object> intersection = setOps.intersect("user:follows:1", "user:follows:2");

// SUNION：并集（所有关注）
Set<Object> union = setOps.union("user:follows:1", "user:follows:2");

// SDIFF：差集（我关注但对方不关注的）
Set<Object> diff = setOps.difference("user:follows:1", "user:follows:2");

// SINTERSTORE / SUNIONSTORE / SDIFFSTORE：将结果存入新 key
Long resultLen = setOps.intersectAndStore(
    "user:follows:1", "user:follows:2", "temp:common:follows");

// SRANDMEMBER：随机获取成员（抽奖场景）
Object random = setOps.randomMember("lottery:pool");
List<Object> winners = setOps.randomMembers("lottery:pool", 3);

// SPOP：随机弹出（不放回抽奖）
Object winner = setOps.pop("lottery:pool");

// SMOVE：移动元素到另一个集合
Boolean moved = setOps.move("set1", "value1", "set2");
```

## 4.4 ZSet（有序集合）操作

```java
ZSetOperations<String, Object> zSetOps = redisTemplate.opsForZSet();

// ZADD：添加元素及分数
zSetOps.add("leaderboard:score", "player1", 100);
zSetOps.add("leaderboard:score", "player2", 200);
zSetOps.add("leaderboard:score", "player3", 150);

// ZINCRBY：增加分数
Double newScore = zSetOps.incrementScore("leaderboard:score", "player1", 50);

// ZRANK：按分数从低到高排名（从 0 开始）
Long rank = zSetOps.rank("leaderboard:score", "player2");  // 最高是 2

// ZREVRANK：从高到低排名
Long reverseRank = zSetOps.reverseRank("leaderboard:score", "player2");  // 最高是 0

// ZRANGE：按排名范围获取（从低到高）
Set<Object> range = zSetOps.range("leaderboard:score", 0, 9);          // Top 10

// ZREVRANGE：按排名范围获取（从高到低）
Set<Object> reverseRange = zSetOps.reverseRange("leaderboard:score", 0, 9);

// ZRANGEBYSCORE：按分数范围获取
Set<Object> scoreRange = zSetOps.rangeByScore("leaderboard:score", 100, 200);

// ZCARD：元素数量
Long card = zSetOps.size("leaderboard:score");

// ZCOUNT：指定分数范围内的元素数量
Long count = zSetOps.count("leaderboard:score", 100, 200);

// ZSCORE：获取元素分数
Double score = zSetOps.score("leaderboard:score", "player1");

// ZREM：删除元素
Long removedCount = zSetOps.remove("leaderboard:score", "player3");

// ZREMRANGEBYRANK：按排名范围删除
zSetOps.removeRange("leaderboard:score", 0, 99);      // 删除排名 0-99

// ZREMRANGEBYSCORE：按分数范围删除
zSetOps.removeRangeByScore("leaderboard:score", 0, 60);

// ZUNIONSTORE：合并多个 ZSet（并集）
zSetOps.unionAndStore("key1", "key2", "dest:key");
```

**ZSet 典型应用场景**：

```java
// 排行榜服务
@Service
public class LeaderboardService {

    private final StringRedisTemplate redis;

    public LeaderboardService(StringRedisTemplate redis) {
        this.redis = redis;
    }

    // 更新分数
    public void updateScore(String playerId, double score) {
        redis.opsForZSet().add("leaderboard", playerId, score);
    }

    // 获取 Top N
    public Set<ZSetOperations.TypedTuple<String>> getTopN(int n) {
        return redis.opsForZSet()
            .reverseRangeWithScores("leaderboard", 0, n - 1);
    }

    // 获取玩家排名
    public Long getRank(String playerId) {
        Long rank = redis.opsForZSet().reverseRank("leaderboard", playerId);
        return rank == null ? -1 : rank + 1;  // 1-based ranking
    }
}
```

## 4.5 Hash（哈希）操作

```java
HashOperations<String, Object, Object> hashOps = redisTemplate.opsForHash();

// HSET：设置字段值
hashOps.put("user:1", "name", "张三");
hashOps.put("user:1", "email", "zhangsan@example.com");
hashOps.put("user:1", "age", "25");

// HMSET：批量设置
Map<String, Object> userMap = new HashMap<>();
userMap.put("name", "李四");
userMap.put("email", "lisi@example.com");
hashOps.putAll("user:2", userMap);

// HGET：获取字段值
Object name = hashOps.get("user:1", "name");

// HMGET：批量获取
List<Object> fields = hashOps.multiGet("user:1", Arrays.asList("name", "email"));

// HGETALL：获取所有字段
Map<Object, Object> allFields = hashOps.entries("user:1");

// HKEYS：获取所有字段名
Set<Object> keys = hashOps.keys("user:1");

// HVALS：获取所有字段值
List<Object> values = hashOps.values("user:1");

// HEXISTS：判断字段是否存在
Boolean exists = hashOps.hasKey("user:1", "email");

// HDEL：删除字段
Long deleted = hashOps.delete("user:1", "age");

// HINCRBY：整数增量（计数器）
Long newCount = hashOps.increment("article:1001", "views", 1);

// HINCRBYFLOAT：浮点增量
Double newBalance = hashOps.increment("user:1", "balance", 9.99);

// HLEN：字段数量
Long fieldCount = hashOps.size("user:1");

// HSCAN：渐进式遍历（大数据量场景）
Cursor<Map.Entry<Object, Object>> cursor = hashOps.scan("big:hash",
    ScanOptions.scanOptions().match("*").count(100).build());
while (cursor.hasNext()) {
    Map.Entry<Object, Object> entry = cursor.next();
    // 处理 entry
}
```

**Hash vs String 存储对象的选择**：

| 方式 | 优点 | 缺点 | 场景 |
|------|------|------|------|
| Hash 存对象 | 可单独操作某个字段 | 字段不能设置独立 TTL | 需要部分更新 |
| String JSON | 可整体设 TTL | 更新需序列化整个对象 | 完整存取 |

## 4.6 Geo（地理位置）操作

```java
GeoOperations<String, Object> geoOps = redisTemplate.opsForGeo();

// GEOADD：添加地理位置
Point beijing = new Point(116.38, 39.90);
Point shanghai = new Point(121.47, 31.23);
geoOps.add("cities", beijing, "北京");
geoOps.add("cities", shanghai, "上海");
geoOps.add("cities", new Point(113.23, 23.16), "广州");

// GEOPOS：获取位置坐标
List<Point> positions = geoOps.position("cities", "北京", "上海");

// GEODIST：计算两点距离
Distance distance = geoOps.distance("cities", "北京", "上海",
    RedisGeoCommands.DistanceUnit.KILOMETERS);

// GEORADIUS（SDK版本可能已调整为GEOSEARCH）：以某点为中心查询周边
GeoResults<GeoLocation<Object>> results = geoOps.radius("cities",
    new Circle(new Point(116.38, 39.90), new Distance(600, DistanceUnit.KILOMETERS)),
    RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
        .includeCoordinates()
        .includeDistance()
        .sortAscending()
        .limit(10));

for (GeoResult<GeoLocation<Object>> result : results) {
    GeoLocation<Object> content = result.getContent();
    System.out.println(content.getName() + " 距离: " +
        result.getDistance().getValue() + "km");
}

// GEODIST 批量计算（附近的商家）
List<Point> myLocation = Collections.singletonList(new Point(121.47, 31.23));
List<Point> storeLocations = getStoreCoords();
// 可使用 GeoHash 编码进行范围查询
```

## 4.7 HyperLogLog（基数统计）

```java
HyperLogLogOperations<String, Object> hllOps = redisTemplate.opsForHyperLogLog();

// PFADD：添加元素
hllOps.add("uv:page:home", "user1", "user2", "user3");
hllOps.add("uv:page:home", "user1");  // 重复添加不影响计数

// PFCOUNT：估算独立元素数量（标准误差 0.81%）
Long uv = hllOps.size("uv:page:home");

// PFMERGE：合并多个 HLL
Long mergedSize = hllOps.union("uv:total",
    "uv:page:home", "uv:page:detail", "uv:page:cart");

// UV 统计示例
@Service
public class UVStatisticsService {

    private final StringRedisTemplate redis;

    public void recordPV(String userId) {
        redis.opsForValue().increment("pv:" + today());  // PV 精确计数
        redis.opsForHyperLogLog().add("uv:" + today(), userId);  // UV 估算
    }

    public long getUV(String date) {
        return redis.opsForHyperLogLog().size("uv:" + date);
    }

    private String today() {
        return LocalDate.now().toString();
    }
}
```

## 4.8 Bitmap（位图）操作

```java
// SETBIT：设置位
redisTemplate.opsForValue().setBit("sign:user:1001:2024-01", 0, true);  // 1月1日
redisTemplate.opsForValue().setBit("sign:user:1001:2024-01", 15, true); // 1月16日

// GETBIT：获取位
Boolean signed = redisTemplate.opsForValue().getBit("sign:user:1001:2024-01", 0);

// BITCOUNT：统计为 1 的位数（签到天数）
Long signedDays = redisTemplate.execute(
    (RedisCallback<Long>) connection ->
        connection.bitCount("sign:user:1001:2024-01".getBytes()));

// BITOP：位操作（用于统计连续签到）
// BITOP AND / OR / XOR / NOT
```

## 4.9 Stream（消息流）操作 — Redis 5.0+

```java
StreamOperations<String, Object, Object> streamOps = redisTemplate.opsForStream();

// XADD：添加消息
Map<String, Object> messageBody = new HashMap<>();
messageBody.put("orderId", "202401-001");
messageBody.put("amount", 9999);
RecordId messageId = streamOps.add("stream:orders", messageBody);

// XREAD：读取消息
List<MapRecord<String, Object, Object>> messages = streamOps.read(
    Consumer.from("group1", "consumer1"),
    StreamReadOptions.empty().count(10).block(Duration.ofSeconds(2)),
    StreamOffset.create("stream:orders", ReadOffset.lastConsumed()));

// XGROUP：创建消费组
streamOps.createGroup("stream:orders", ReadOffset.from("0"), "group1");
```

# 5.高级操作

## 5.1 Pipeline（管道操作）

减少网络往返次数，批量执行命令：

```java
// 不使用 Pipeline：99 次网络往返
List<Object> results = redisTemplate.executePipelined(
    (RedisCallback<Object>) connection -> {
        StringRedisConnection stringRedisConn =
            (StringRedisConnection) connection;
        for (int i = 0; i < 100; i++) {
            stringRedisConn.set("key:" + i, "value:" + i);
        }
        return null;  // 必须返回 null
    });

// 数据量较小的批量操作，可直接用 multiSet/multiGet 代替
// Pipeline 适用于不同命令类型的批量执行
```

**使用场景**：
- 批量初始化缓存数据
- 批量计数器更新
- 批量写入性能要求较高的场景

## 5.2 Redis 事务（MULTI/EXEC）

```java
// Redis 事务特点：
// - 命令按顺序执行，执行期间不会被其他客户端命令打断
// - 不支持回滚（Redis 的事务是弱事务）
// - 语法错误：所有命令不执行
// - 运行时错误（如对 String key 执行 LPOP）：错误的命令报错，其他命令继续执行

List<Object> txResults = redisTemplate.execute(new SessionCallback<List<Object>>() {
    @Override
    public List<Object> execute(RedisOperations operations) {
        operations.multi();  // 开启事务

        operations.opsForValue().set("tx:key1", "value1");
        operations.opsForValue().increment("tx:counter");
        operations.opsForList().leftPush("tx:list", "item1");

        return operations.exec();  // 执行事务，返回每条命令的结果
    }
});

// 配合 WATCH 实现乐观锁（check-and-set）
redisTemplate.execute(new SessionCallback<Object>() {
    @Override
    public Object execute(RedisOperations operations) {
        operations.watch("tx:balance");  // 监视 key

        // 读取当前值
        String balance = (String) operations.opsForValue().get("tx:balance");
        int newBalance = Integer.parseInt(balance) - 100;
        if (newBalance < 0) {
            operations.unwatch();
            return false;
        }

        operations.multi();
        operations.opsForValue().set("tx:balance", String.valueOf(newBalance));
        List<Object> result = operations.exec();  // 如果 "tx:balance" 被其他客户端修改，exec 返回 null

        return result != null && !result.isEmpty();
    }
});
```

## 5.3 发布/订阅（Pub/Sub）

```java
// === 消息发布 ===
@Service
public class MessagePublisher {

    private final RedisTemplate<String, Object> redisTemplate;

    public void publish(String channel, Object message) {
        redisTemplate.convertAndSend(channel, message);
    }
}

// === 消息订阅 ===
@Component
public class OrderMessageListener implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String body = new String(message.getBody());
        System.out.println("收到消息 [Channel=" + channel + "]: " + body);
        // 处理业务逻辑
    }
}

// === 订阅配置 ===
@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory factory,
            OrderMessageListener listener) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);

        // 订阅频道
        container.addMessageListener(listener,
            new ChannelTopic("order:created"));
        // 按模式订阅（通配符）
        container.addMessageListener(listener,
            new PatternTopic("order:*"));

        return container;
    }
}
```

**Pub/Sub 的局限性**：
- 消息不持久化，订阅者离线期间的消息会丢失
- 不支持消息确认和重试
- 生产环境消息队列应优先使用 RabbitMQ / Kafka / RocketMQ

## 5.4 Lua 脚本

```java
// Lua 脚本在 Redis 中原子执行，适合复杂操作
@Configuration
public class RedisLuaConfig {

    // 加载脚本（预编译，减少网络传输）
    @Bean
    public DefaultRedisScript<Long> releaseLockScript() {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setScriptText("""
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """);
        script.setResultType(Long.class);
        return script;
    }

    @Bean
    public DefaultRedisScript<Boolean> rateLimitScript() {
        DefaultRedisScript<Boolean> script = new DefaultRedisScript<>();
        script.setScriptText("""
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])

            local current = redis.call('INCR', key)
            if current == 1 then
                redis.call('EXPIRE', key, window)
            end
            if current > limit then
                return false
            end
            return true
            """);
        script.setResultType(Boolean.class);
        return script;
    }
}

// 执行 Lua 脚本
@Service
public class LuaScriptService {

    private final StringRedisTemplate redis;
    private final DefaultRedisScript<Long> releaseLockScript;
    private final DefaultRedisScript<Boolean> rateLimitScript;

    // 释放锁
    public boolean releaseLock(String lockKey, String lockValue) {
        Long result = redis.execute(releaseLockScript,
            Collections.singletonList(lockKey), lockValue);
        return result != null && result == 1;
    }

    // 限流
    public boolean isAllowed(String userId, int limit, int windowSec) {
        String key = "rate:limit:" + userId;
        Boolean result = redis.execute(rateLimitScript,
            Collections.singletonList(key),
            String.valueOf(limit), String.valueOf(windowSec));
        return Boolean.TRUE.equals(result);
    }
}
```

# 6.Redis Repositories

## 6.1 Redis Repository 概念

Spring Data Redis 提供了类似 JPA 的 Repository 编程模型，将 Hash 结构映射为 Java 对象：

```java
// 实体定义
@RedisHash("users")              // Redis key 前缀：users:<id>
public class User {

    @Id                            // 标识符，对应 key：users:<id>
    private String id;

    @Indexed                       // 创建二级索引（Set）
    private String name;

    @Indexed
    private String email;

    private Integer age;

    @TimeToLive(unit = TimeUnit.SECONDS)  // TTL
    private Long expiration;

    // getters and setters ...
}

// Repository 接口
public interface UserRepository extends CrudRepository<User, String> {
    List<User> findByName(String name);     // @Indexed 字段支持查询
    List<User> findByEmail(String email);
    List<User> findByAgeGreaterThan(int age);
}

// Service 使用
@Service
public class UserRedisService {

    private final UserRepository userRepository;

    public User save(User user) {
        return userRepository.save(user);
    }

    public Optional<User> findById(String id) {
        return userRepository.findById(id);
    }

    public List<User> findByName(String name) {
        return userRepository.findByName(name);
    }
}
```

**Redis Repository 底层存储结构**：

```
保存 User{id="u1", name="张三", email="zh@ex.com"} 时，Redis 中实际存储：

1. users:u1  (Hash)
   HGETALL users:u1 → {_class, id, name, email, age}

2. users:name:张三 (Set)  ← 二级索引
   SMEMBERS users:name:张三 → {"u1"}

3. users:email:zh@ex.com (Set)  ← 二级索引
   SMEMBERS users:email:zh@ex.com → {"u1"}

4. users (Set)  ← 保存了所有 ID
   SMEMBERS users → {"u1"}
```

## 6.2 查询方法支持

```java
public interface UserRepository extends CrudRepository<User, String> {

    // 精确匹配
    List<User> findByName(String name);

    // 大于/小于
    List<User> findByAgeGreaterThan(Integer age);
    List<User> findByAgeBetween(Integer min, Integer max);

    // Like 查询（模糊匹配）
    List<User> findByNameStartingWith(String prefix);
    List<User> findByNameEndingWith(String suffix);
    List<User> findByNameContaining(String keyword);

    // AND / OR 组合
    List<User> findByNameAndAge(String name, Integer age);
    List<User> findByNameOrAge(String name, Integer age);

    // 排序
    List<User> findByAgeOrderByNameDesc(Integer age);

    // 计数
    long countByName(String name);

    // 删除
    void deleteByName(String name);
}
```

## 6.3 Redis Repository 适用场景与限制

**适用场景**：
- 缓存用户/配置等以 Hash 存储的简单实体
- 需要快速 CRUD 且数据相对稳定的场景
- 不需要复杂关联查询

**限制**：
- 不支持关联查询（无 JOIN）
- @Indexed 字段查询依赖 Set 辅助索引，内存占用较大
- 不适合大数据量场景（索引集膨胀）
- 不支持分页（Spring Data Redis 3.x 之前）

# 7.Spring Cache 集成

## 7.1 启用 Redis 缓存

```java
@Configuration
@EnableCaching  // 启用 Spring Cache
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))           // 默认过期时间
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .prefixCacheNameWith("cache:")               // 缓存 key 前缀
            .disableCachingNullValues();                 // 不缓存 null

        // 为不同缓存名指定不同 TTL
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        cacheConfigs.put("users", config.entryTtl(Duration.ofHours(1)));
        cacheConfigs.put("products", config.entryTtl(Duration.ofMinutes(30)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigs)
            .transactionAware()
            .build();
    }
}
```

## 7.2 缓存注解使用

```java
@Service
public class ProductService {

    // @Cacheable：先从缓存取，缓存无则执行方法并缓存结果
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    // 批量查询（条件缓存）
    @Cacheable(value = "products", key = "#root.methodName + '_' + #category",
               condition = "#category != null && #category.length() > 0")
    public List<Product> getByCategory(String category) {
        return productRepository.findByCategory(category);
    }

    // @CachePut：总是执行方法并更新缓存
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // @CacheEvict：删除缓存
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    // 清除多个缓存
    @CacheEvict(value = {"products", "categories"}, allEntries = true)
    public void clearAllCache() { }

    // @Caching：组合多个缓存操作
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#product.id"),
        @CacheEvict(value = "products", key = "'byCategory_' + #product.category")
    })
    public void deleteProductComplex(Product product) { }
}
```

**缓存注解详解**：

| 注解 | 说明 | 核心属性 |
|------|------|---------|
| `@Cacheable` | 查询缓存 | value, key, condition, unless, cacheManager |
| `@CachePut` | 更新缓存 | value, key, condition, unless |
| `@CacheEvict` | 删除缓存 | value, key, allEntries, beforeInvocation |
| `@Caching` | 组合操作 | cacheable, put, evict |
| `@CacheConfig` | 类级别配置 | cacheNames, keyGenerator |

**SpEL 表达式常用写法**：

```java
// key 的 SpEL 表达式
@Cacheable(key = "#id")                     // 方法参数
@Cacheable(key = "#user.id")                // 参数对象属性
@Cacheable(key = "#p0")                     // 第 1 个参数
@Cacheable(key = "#a0")                     // 第 1 个参数（别名）
@Cacheable(key = "#root.methodName")         // 方法名
@Cacheable(key = "#root.targetClass")        // 目标类名
@Cacheable(key = "T(java.lang.String).valueOf(#id)")  // 静态方法
```

# 8.分布式锁

## 8.1 基于 SETNX 的简单实现

```java
@Component
public class RedisDistributedLock {

    private final StringRedisTemplate redis;

    private static final String LOCK_PREFIX = "lock:";
    private static final String UNLOCK_LUA = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """;

    /**
     * 获取锁
     */
    public boolean tryLock(String key, String value, long expireSeconds) {
        Boolean success = redis.opsForValue().setIfAbsent(
            LOCK_PREFIX + key, value, Duration.ofSeconds(expireSeconds));
        return Boolean.TRUE.equals(success);
    }

    /**
     * 释放锁（Lua 脚本保证原子性）
     */
    public boolean unlock(String key, String value) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(UNLOCK_LUA, Long.class);
        Long result = redis.execute(script, Collections.singletonList(LOCK_PREFIX + key), value);
        return Long.valueOf(1L).equals(result);
    }
}

// 使用示例
@Service
public class OrderService {

    private final RedisDistributedLock lock;

    public void createOrder(OrderDTO dto) {
        String lockKey = "order:" + dto.getProductId();
        String lockValue = UUID.randomUUID().toString();

        try {
            // 尝试获取锁
            if (!lock.tryLock(lockKey, lockValue, 30)) {
                throw new RuntimeException("操作过于频繁，请稍后再试");
            }

            // 执行业务逻辑
            doCreateOrder(dto);

        } finally {
            lock.unlock(lockKey, lockValue);
        }
    }
}
```

## 8.2 使用 Redisson（推荐）

Redisson 提供更成熟的分布式锁解决方案：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.27.0</version>
</dependency>
```

```yaml
spring:
  redis:
    # Redisson 自动读取 spring.redis 配置
    redisson:
      config: |
        singleServerConfig:
          address: "redis://localhost:6379"
          password: 123456
```

```java
@Service
public class RedissonLockService {

    private final RedissonClient redisson;

    public void executeWithLock(String lockKey) {
        RLock lock = redisson.getLock(lockKey);

        try {
            // tryLock(waitTime, leaseTime, unit)
            // waitTime: 获取锁的最大等待时间
            // leaseTime: 锁的持有时间（自动续期）
            if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
                // 执行业务逻辑
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    // 自动续期（看门狗机制）
    // 不指定 leaseTime 时，Redisson 自动每 30s/3=10s 续期一次
    public void executeWithWatchdog(String lockKey) {
        RLock lock = redisson.getLock(lockKey);
        lock.lock();  // 无 leaseTime，看门狗自动续期
        try {
            // 长时间业务逻辑
        } finally {
            lock.unlock();
        }
    }
}
```

**Redisson 分布式锁对比原生实现**：

| 特性 | SETNX 实现 | Redisson |
|------|-----------|---------|
| 获取锁 | SETNX + EXPIRE | Lua 脚本原子操作 |
| 锁续期 | 需自行实现 | 看门狗自动续期 |
| 可重入 | 需自行实现 | 内置支持 |
| 公平锁 | 不支持 | 支持 |
| 读写锁 | 不支持 | 支持 |
| 联锁/红锁 | 需自行实现 | 内置支持 |

# 9.Redis 集群支持

## 9.1 Sentinel（哨兵）模式

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - 192.168.1.10:26379
          - 192.168.1.11:26379
          - 192.168.1.12:26379
      password: 123456
      lettuce:
        pool:
          max-active: 16
```

哨兵模式原理：
```
客户端先连接 Sentinel → 获取当前 Master 地址 → 直接连接 Master
                                                  ↓
                                          Master 故障 → Sentinel 选举新 Master
                                                           ↓
                                          客户端收到切换通知 → 连接新 Master
```

## 9.2 Cluster（集群）模式

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - 192.168.1.10:6379
          - 192.168.1.11:6379
          - 192.168.1.12:6379
          - 192.168.1.13:6379
          - 192.168.1.14:6379
          - 192.168.1.15:6379
        max-redirects: 3          # 最大重定向次数
      password: 123456
      lettuce:
        cluster:
          refresh:
            adaptive: true        # 自适应集群拓扑刷新
            period: 30s
```

**Cluster 模式注意事项**：
- 跨 slot 的 mget/mset 操作会报错（需 key 在同一 slot）
- Lua 脚本涉及的所有 key 必须在同一节点（用 hash tag `{key}` 保证）
- 事务（MULTI/EXEC）不能跨节点

```java
// Hash Tag 用法：所有 {order:1001} 相关的 key 都会落到同一 slot
redis.opsForValue().set("{order:1001}:status", "paid");
redis.opsForValue().set("{order:1001}:amount", "9999");
// 这两个 key 可以使用 MULTI/EXEC 事务操作
```

# 10.性能优化

## 10.1 连接池配置优化

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 50        # 最大活跃连接（根据 QPS 调整）
          max-idle: 30          # 最大空闲连接
          min-idle: 10          # 最小空闲（预热连接）
          max-wait: 3000        # 获取连接最大等待时间（ms）
```

## 10.2 序列化优化

| 序列化方式 | 存储大小 | 序列化速度 | 跨语言 | 推荐度 |
|-----------|---------|-----------|--------|--------|
| JDK 序列化 | 大 | 慢 | 否 | 不推荐 |
| JSON (Jackson) | 中 | 中 | 是 | 推荐（通用） |
| ProtoStuff | 小 | 快 | 否 | 推荐（高性能） |
| Kryo | 小 | 快 | 否 | 推荐（高性能） |
| MessagePack | 小 | 快 | 是 | 推荐（跨语言） |

## 10.3 避免的危险操作

```java
// 危险1：KEYS *（生产禁用！）
// redisTemplate.keys("*")  ← 阻塞 Redis

// 正确做法：使用 SCAN 渐进遍历
Cursor<String> cursor = redisTemplate.scan(
    ScanOptions.scanOptions().match("user:*").count(100).build());
while (cursor.hasNext()) {
    String key = cursor.next();
    // 处理 key
}

// 危险2：大 Value（建议小于 10KB，最大不超过 1MB）
// 超过 10KB 的 value 考虑压缩或拆分

// 危险3：大 Hash / 大 Set / 大 ZSet
// 单个集合超过 5000 个元素，考虑拆分或使用 HSCAN/SSCAN/ZSCAN

// 危险4：永不过期的 key
// 所有 key 设置合理的 TTL，避免内存无限增长
```

## 10.4 BigKey 排查与处理

```java
// 排查 BigKey
redis-cli --bigkeys

// 查看某个 key 的内存占用
redis-cli MEMORY USAGE user:1

// Service 中使用
public void checkKeySize(String key) {
    Long size = redisTemplate.execute(
        (RedisCallback<Long>) conn ->
            conn.memoryUsage(key.getBytes()));
    if (size != null && size > 10240) {  // 超过 10KB
        log.warn("BigKey found: {}, size: {} bytes", key, size);
    }
}
```

# 11.常见问题与面试题

## 11.1 缓存穿透、击穿、雪崩

| 问题 | 描述 | 解决方案 |
|------|------|---------|
| **缓存穿透** | 查询不存在的数据，请求直达 DB | 布隆过滤器、缓存空值（短 TTL） |
| **缓存击穿** | 热点数据过期，大量请求涌入 DB | 互斥锁（SETNX）、永不过期 + 异步更新 |
| **缓存雪崩** | 大量缓存同时过期 | TTL 加随机值、多级缓存、限流降级 |

```java
// 缓存穿透 - 缓存空值
@Cacheable(value = "products", key = "#id",
           unless = "#result == null")
public Product getProduct(Long id) {
    Product product = productRepository.findById(id).orElse(null);
    // 将 null 缓存短时间
    return product;
}

// 缓存击穿 - 互斥锁
public Product getProductWithLock(Long id) {
    String cacheKey = "product:" + id;
    Product product = (Product) redisTemplate.opsForValue().get(cacheKey);

    if (product == null) {
        String lockKey = "lock:product:" + id;
        if (tryLock(lockKey, "1", 10)) {
            try {
                // 双重检查
                product = (Product) redisTemplate.opsForValue().get(cacheKey);
                if (product == null) {
                    product = productRepository.findById(id).orElse(null);
                    // 缓存时间 + 随机值避免雪崩
                    int randomMinutes = new Random().nextInt(10);
                    redisTemplate.opsForValue().set(cacheKey, product,
                        Duration.ofMinutes(30 + randomMinutes));
                }
            } finally {
                unlock(lockKey, "1");
            }
        }
    }
    return product;
}
```

## 11.2 Redis 内存淘汰策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `noeviction` | 不淘汰，写满报错 | 不推荐 |
| `allkeys-lru` | 所有 key 中 LRU 淘汰 | **缓存推荐** |
| `allkeys-lfu` | 所有 key 中 LFU 淘汰 | 访问频率差异大 |
| `volatile-lru` | 过期 key 中 LRU 淘汰 | 混合使用场景 |
| `volatile-lfu` | 过期 key 中 LFU 淘汰 | — |
| `volatile-ttl` | 即将过期的 key 优先淘汰 | — |
| `volatile-random` | 过期 key 中随机淘汰 | — |
| `allkeys-random` | 所有 key 中随机淘汰 | — |

```yaml
# Redis 配置
maxmemory 2gb
maxmemory-policy allkeys-lru
```

## 11.3 面试高频考点

**1. Redis 单线程为什么快？**
- 纯内存操作，无磁盘 IO 瓶颈
- 单线程避免了多线程的上下文切换和锁竞争
- 基于 epoll/kqueue 的 IO 多路复用

**2. Redis 与 Memcached 的区别？**
- Redis 支持更丰富的数据结构（List、Set、ZSet、Hash 等）
- Redis 支持持久化（RDB 和 AOF）
- Redis 支持集群、主从、哨兵模式
- Redis 单线程但性能不输多线程 Memcached

**3. Lettuce 相比 Jedis 的优势？**
- 基于 Netty，原生支持异步和 Reactive
- 连接是线程安全的，单连接可多线程复用
- Spring Boot 2.x+ 默认推荐使用

**4. 如何保证缓存与数据库一致性？**
- 先更新 DB，再删除缓存（Cache Aside 模式）
- 延迟双删策略
- 订阅 binlog + 消息队列异步更新缓存（最终一致性）

# 12.Redis 配置详解

## 12.1 Spring Boot 全量配置参考

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      username:
      password:
      database: 0
      timeout: 2000ms
      connect-timeout: 2000ms
      client-name: ${spring.application.name}
      ssl:
        enabled: false

      lettuce:
        shutdown-timeout: 100ms
        pool:
          enabled: true
          max-active: 16
          max-idle: 8
          min-idle: 4
          max-wait: 2000ms
          time-between-eviction-runs: 30s
        cluster:
          refresh:
            adaptive: true
            period: 30s

      # Sentinel 模式
      # sentinel:
      #   master: mymaster
      #   nodes:
      #     - 192.168.1.10:26379
      #     - 192.168.1.11:26379

      # Cluster 模式
      # cluster:
      #   nodes:
      #     - 192.168.1.10:6379
      #     - 192.168.1.11:6379
      #   max-redirects: 3
```

## 12.2 编程方式配置

```java
@Configuration
public class RedisCustomConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration standaloneConfig =
            new RedisStandaloneConfiguration();
        standaloneConfig.setHostName("localhost");
        standaloneConfig.setPort(6379);
        standaloneConfig.setPassword(RedisPassword.of("123456"));
        standaloneConfig.setDatabase(0);

        LettuceClientConfiguration clientConfig =
            LettuceClientConfiguration.builder()
                .commandTimeout(Duration.ofSeconds(2))
                .shutdownTimeout(Duration.ofMillis(100))
                .useSsl()
                .build();

        return new LettuceConnectionFactory(standaloneConfig, clientConfig);
    }
}
```

# 13.响应式 Redis（Reactive）

```java
// 依赖
// <dependency>
//     <groupId>org.springframework.boot</groupId>
//     <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
// </dependency>

@Service
public class ReactiveRedisService {

    private final ReactiveRedisTemplate<String, String> reactiveRedis;

    public ReactiveRedisService(ReactiveRedisTemplate<String, String> reactiveRedis) {
        this.reactiveRedis = reactiveRedis;
    }

    public Mono<String> getValue(String key) {
        return reactiveRedis.opsForValue().get(key);
    }

    public Mono<Boolean> setValue(String key, String value, Duration ttl) {
        return reactiveRedis.opsForValue().set(key, value, ttl);
    }

    public Flux<String> getCacheKeys(String pattern) {
        return reactiveRedis.keys(pattern)
            .flatMap(key -> reactiveRedis.opsForValue().get(key));
    }
}
```

# 14.最佳实践总结

```
1. 序列化器选择
   - 对象缓存：GenericJackson2JsonRedisSerializer（或 FastJson）
   - 纯字符串：StringRedisTemplate
   - 高性能场景：ProtoStuff / Kryo

2. Key 命名规范
   - 统一命名：业务:模块:标识:属性
   - 示例：order:2024:10086:status、user:1:info

3. TTL 策略
   - 所有 Key 必须设置过期时间（除非是持久数据）
   - 热点数据永不过期，通过异步更新保证一致性
   - 过期时间添加随机抖动，避免雪崩

4. 连接池
   - 监控连接池使用情况
   - 合理设置 max-active 和 max-idle

5. 禁止操作
   - 禁用 KEYS *：使用 SCAN 替代
   - 禁用 FLUSHALL/FLUSHDB
   - 避免大 Value（>1MB）

6. 监控指标
   - 命中率、连接数、内存使用、慢查询
   - Spring Boot Actuator + Micrometer 集成
```
