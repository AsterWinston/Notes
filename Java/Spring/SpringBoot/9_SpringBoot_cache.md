# 1. Spring Cache抽象

Spring Cache是Spring框架提供的一套缓存抽象，通过注解的方式简化缓存操作，无需关心底层缓存实现。

## 1.1 启用缓存
在启动类上添加 `@EnableCaching` 注解：

```java
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 1.2 核心注解

| 注解 | 说明 |
|------|------|
| @Cacheable | 触发缓存填充（先查缓存，缓存有则返回；缓存没有则执行方法，结果存入缓存） |
| @CacheEvict | 触发缓存删除（删除指定的缓存） |
| @CachePut | 触发缓存更新（总是执行方法，并将结果更新到缓存） |
| @Caching | 组合多个缓存操作 |
| @CacheConfig | 类级别共享缓存配置 |

## 1.3 @Cacheable

```java
@Service
public class UserService {

    // value：缓存名称  key：缓存键（SpEL表达式）
    @Cacheable(value = "user", key = "#id")
    public User getById(Long id) {
        System.out.println("=== 查询数据库，id=" + id + " ===");
        // 模拟数据库查询
        return userMapper.selectById(id);
    }

    // condition：满足条件才缓存
    @Cacheable(value = "user", key = "#id", condition = "#id > 0")
    public User getByIdConditional(Long id) {
        return userMapper.selectById(id);
    }

    // unless：满足条件则不缓存（与condition相反）
    @Cacheable(value = "user", key = "#id", unless = "#result == null")
    public User getByIdUnless(Long id) {
        return userMapper.selectById(id);
    }
}
```

执行流程：
1. 第一次调用 `getById(1L)`：执行方法体，打印"查询数据库"，结果存入缓存
2. 第二次调用 `getById(1L)`：不执行方法体，直接从缓存返回
3. 第三次调用 `getById(2L)`：缓存中没有key为2的数据，执行方法体

## 1.4 @CacheEvict

```java
@Service
public class UserService {

    // 删除单条缓存
    @CacheEvict(value = "user", key = "#id")
    public void deleteById(Long id) {
        userMapper.deleteById(id);
    }

    // allEntries = true：删除user下的所有缓存
    @CacheEvict(value = "user", allEntries = true)
    public void deleteAll() {
        userMapper.deleteAll();
    }

    // beforeInvocation = true：在方法执行前清除缓存（默认是执行后）
    @CacheEvict(value = "user", key = "#id", beforeInvocation = true)
    public void deleteByIdBefore(Long id) {
        userMapper.deleteById(id);
        // 如果这里出了异常，缓存不会被清除(beforeInvocation=true时则相反)
    }
}
```

## 1.5 @CachePut

```java
@Service
public class UserService {

    // 总是执行方法，并将结果更新到缓存
    @CachePut(value = "user", key = "#user.id")
    public User update(User user) {
        userMapper.update(user);
        return user; // 返回值会更新到缓存中
    }
}
```

@Cacheable 和 @CachePut 的区别：
- @Cacheable：缓存中有就直接返回，**不执行**方法
- @CachePut：**总是执行**方法，并用返回值更新缓存

## 1.6 @Caching 组合操作

```java
@Caching(
    cacheable = {
        @Cacheable(value = "user", key = "#id"),
        @Cacheable(value = "user", key = "#username")
    },
    put = {
        @CachePut(value = "user", key = "#result.email")
    }
)
public User getById(Long id, String username) {
    return userMapper.selectById(id);
}
```

## 1.7 @CacheConfig 类级别配置

```java
@Service
@CacheConfig(cacheNames = "user")  // 类中所有缓存默认使用 "user" 这个缓存名
public class UserService {

    @Cacheable(key = "#id")  // 无需重复写 value = "user"
    public User getById(Long id) {
        return userMapper.selectById(id);
    }

    @CacheEvict(key = "#id")
    public void deleteById(Long id) {
        userMapper.deleteById(id);
    }
}
```

## 1.8 默认缓存实现

Spring Boot默认使用 `ConcurrentMapCacheManager` 作为缓存管理器，数据存储在JVM内存中，适合单机开发和简单场景。

```properties
# 可以设置缓存名称（可选，@Cacheable 的 value 会自动创建）
spring.cache.cache-names=user,product,order
```

默认实现的局限：
- 数据存在JVM内存中，应用重启缓存丢失
- 不支持分布式环境
- 没有过期策略

# 2. 整合Redis实现缓存

生产环境中通常使用Redis作为缓存中间件，支持分布式、持久化和丰富的过期策略。

## 2.1 引入依赖
```xml
<!-- Redis依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- 连接池依赖（Spring Boot 3.x 默认使用 Lettuce + commons-pool2） -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

## 2.2 Redis连接配置
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password:
      database: 0
      timeout: 5000ms
      lettuce:
        pool:
          max-active: 8      # 最大连接数
          max-idle: 8        # 最大空闲连接
          min-idle: 2        # 最小空闲连接
          max-wait: 1000ms   # 获取连接最大等待时间
```

或 properties 格式：
```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=
spring.data.redis.database=0
spring.data.redis.timeout=5000ms
spring.data.redis.lettuce.pool.max-active=8
spring.data.redis.lettuce.pool.min-idle=2
```

## 2.3 配置Redis缓存管理器
```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                // 设置缓存过期时间为30分钟
                .entryTtl(Duration.ofMinutes(30))
                // 设置key序列化方式（String）
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                // 设置value序列化方式（JSON）
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                // 缓存空值，防止缓存穿透
                .disableCachingNullValues()
                // 设置缓存key前缀
                .prefixCacheNameWith("cache:");

        // 可以对不同的缓存名称设置不同的过期时间
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put("user", config.entryTtl(Duration.ofMinutes(10)));
        configMap.put("product", config.entryTtl(Duration.ofHours(1)));
        configMap.put("order", config.entryTtl(Duration.ofMinutes(30)));

        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .withInitialCacheConfigurations(configMap)
                .build();
    }
}
```

配置Redis缓存管理器后，`@Cacheable`、`@CacheEvict`、`@CachePut` 等注解会自动使用Redis作为缓存后端，无需修改业务代码。

## 2.4 Redis序列化配置说明

常用的序列化方案：

| 序列化器 | 说明 | 优缺点 |
|---------|------|--------|
| StringRedisSerializer | 字符串序列化 | Key首选，简单易读 |
| GenericJackson2JsonRedisSerializer | Jackson JSON序列化 | 带类型信息，序列化后体积较大 |
| Jackson2JsonRedisSerializer | Jackson JSON序列化 | 需指定类型，不带类型信息，体积小 |
| JdkSerializationRedisSerializer | JDK序列化（默认） | 要求类实现Serializable，可读性差 |

推荐配置：Key用String序列化，Value用JSON序列化。

# 3. RedisTemplate 操作Redis

除了使用注解方式，也可以直接使用 `RedisTemplate` 编程式操作Redis。

## 3.1 自定义RedisTemplate

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // 使用Jackson序列化
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL);

        Jackson2JsonRedisSerializer<Object> jacksonSerializer =
                new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        // Key使用String序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value使用Jackson序列化
        template.setValueSerializer(jacksonSerializer);
        template.setHashValueSerializer(jacksonSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

## 3.2 String类型操作

```java
@Component
public class RedisService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // opsForValue 操作字符串
    public void stringOps() {
        // 设置值
        redisTemplate.opsForValue().set("key1", "hello");
        // 设置值并指定过期时间
        redisTemplate.opsForValue().set("key2", "world", 30, TimeUnit.SECONDS);
        // 获取值
        String value = (String) redisTemplate.opsForValue().get("key1");
        // 删除
        redisTemplate.delete("key1");
        // 判断key是否存在
        Boolean exists = redisTemplate.hasKey("key2");
        // 设置过期时间
        redisTemplate.expire("key2", 60, TimeUnit.SECONDS);
    }
}
```

## 3.3 Hash类型操作

```java
public void hashOps() {
    // 设置单个字段
    redisTemplate.opsForHash().put("user:1", "name", "张三");
    redisTemplate.opsForHash().put("user:1", "age", "25");
    // 批量设置
    Map<String, Object> map = new HashMap<>();
    map.put("email", "zhangsan@example.com");
    map.put("phone", "13800138000");
    redisTemplate.opsForHash().putAll("user:1", map);
    // 获取单个字段
    String name = (String) redisTemplate.opsForHash().get("user:1", "name");
    // 获取所有字段
    Map<Object, Object> entries = redisTemplate.opsForHash().entries("user:1");
    // 删除字段
    redisTemplate.opsForHash().delete("user:1", "phone");
}
```

## 3.4 List类型操作

```java
public void listOps() {
    // 从右边添加
    redisTemplate.opsForList().rightPush("queue", "task1");
    redisTemplate.opsForList().rightPush("queue", "task2");
    // 从左边取出
    String task = (String) redisTemplate.opsForList().leftPop("queue");
    // 获取列表所有元素
    List<Object> list = redisTemplate.opsForList().range("queue", 0, -1);
    // 获取列表长度
    Long size = redisTemplate.opsForList().size("queue");
}
```

## 3.5 Set类型操作

```java
public void setOps() {
    redisTemplate.opsForSet().add("tags", "java", "spring", "redis");
    // 判断是否存在
    Boolean isMember = redisTemplate.opsForSet().isMember("tags", "java");
    // 获取所有成员
    Set<Object> members = redisTemplate.opsForSet().members("tags");
    // 获取集合大小
    Long size = redisTemplate.opsForSet().size("tags");
}
```

## 3.6 ZSet类型操作

```java
public void zsetOps() {
    // 添加元素并指定分数
    redisTemplate.opsForZSet().add("ranking", "player1", 95.5);
    redisTemplate.opsForZSet().add("ranking", "player2", 88.0);
    redisTemplate.opsForZSet().add("ranking", "player3", 92.0);
    // 按分数从低到高排序获取
    Set<Object> ranking = redisTemplate.opsForZSet().range("ranking", 0, -1);
    // 按分数从高到低排序获取
    Set<Object> rankingDesc = redisTemplate.opsForZSet().reverseRange("ranking", 0, -1);
    // 获取元素的排名
    Long rank = redisTemplate.opsForZSet().rank("ranking", "player1");
}
```

# 4. 缓存常见问题与解决方案

## 4.1 缓存穿透

**问题**：查询一个不存在的数据，缓存没有命中，请求直接打到数据库。

**解决方案**：
- 缓存空值：`RedisCacheConfiguration.defaultCacheConfig().disableCachingNullValues()` 去掉即可缓存空值
- 布隆过滤器：在缓存前加一层布隆过滤器判断key是否存在

```java
@Cacheable(value = "user", key = "#id", unless = "#result == null")
public User getById(Long id) {
    return userMapper.selectById(id);
}
// unless = "#result == null" 表示结果为null时不缓存
// 如果想要缓存空值防止穿透，去掉 unless 即可
```

## 4.2 缓存击穿

**问题**：一个热点key在过期瞬间，大量请求同时打到数据库。

**解决方案**：
- 互斥锁：只让一个请求查询数据库并回写缓存，其他请求等待
- 逻辑过期：设置永不过期的"物理缓存" + 逻辑过期字段

```java
@Cacheable(value = "user", key = "#id")
public User getById(Long id) {
    // 实际项目中可使用分布式锁（如Redisson）解决击穿
    return userMapper.selectById(id);
}
```

## 4.3 缓存雪崩

**问题**：大量key在同一时间过期，导致大量请求打到数据库。

**解决方案**：
- 随机过期时间：在设置过期时间时加上随机值
- 限流降级：使用Sentinel/Hystrix对数据库访问进行限流

```java
// 给不同的缓存设置不同的TTL，错开过期时间
configMap.put("user", config.entryTtl(Duration.ofMinutes(10)));
configMap.put("product", config.entryTtl(Duration.ofMinutes(10 + new Random().nextInt(5))));
configMap.put("order", config.entryTtl(Duration.ofMinutes(10 + new Random().nextInt(10))));
```
