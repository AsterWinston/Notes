
# 1. 限流算法

## 1.1 令牌桶算法

Spring Cloud Gateway 内置的 Redis 限流基于令牌桶算法（Token Bucket）。

```
            ┌─────────────────────┐
            │   令牌桶（容量 N）    │
            │                     │
   请求 →   │  🪙 🪙 🪙 🪙 🪙    │  → 有令牌 → 通过
            │  🪙 🪙 🪙           │
            └─────────────────────┘
                    ↑
              按速率补充令牌
               （r 个/秒）
```

**算法要点**：
- 桶容量（burstCapacity）：桶最多能存放的令牌数，控制瞬发流量
- 补充速率（replenishRate）：每秒向桶中放入的令牌数，控制平均速率
- 每次请求消耗 1 个或多个令牌（requestedTokens）

## 1.2 漏桶算法

漏桶算法的流出速率固定，适合平滑流量：

```
    请求 → ║ ║ ║ ║ ║ ║  → 恒定速率流出
          ────────────
```

**令牌桶 vs 漏桶**：

| 特性 | 令牌桶 | 漏桶 |
|------|--------|------|
| 突发流量 | 允许短时突发 | 严格限制 |
| 平均速率 | 由补充速率控制 | 由流出速率控制 |
| 实现复杂度 | 简单 | 简单 |
| 适用场景 | 允许突发流量 | 严格平滑 |

# 2. RequestRateLimiter 过滤器源码解析

## 2.1 核心接口

```java
// 限流器接口
public interface RateLimiter<C> {
    Mono<Response> isAllowed(String routeId, String id);
    Map<String, C> getConfig();
    Class<C> getConfigClass();
}

// 限流响应
public class Response {
    private final boolean allowed;     // 是否允许通过
    private final long tokensRemaining; // 剩余令牌数

    public Response(boolean allowed, long tokensRemaining) {
        this.allowed = allowed;
        this.tokensRemaining = tokensRemaining;
    }

    public static Response denied() {
        return new Response(false, -1);
    }
}

// 键解析器——限流维度
public interface KeyResolver {
    Mono<String> resolve(ServerWebExchange exchange);
}
```

## 2.2 RequestRateLimiterGatewayFilterFactory

```java
public class RequestRateLimiterGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RequestRateLimiterGatewayFilterFactory.Config> {

    public static final String GATEWAY_REQUEST_RATE_LIMITER_ATTR = "gatewayRequestRateLimiter";

    @Override
    public GatewayFilter apply(Config config) {
        KeyResolver resolver = getResolver(config.getKeyResolver());
        RateLimiter<Object> limiter = getLimiter(config.getRateLimiter());

        return (exchange, chain) -> resolver.resolve(exchange)
            .flatMap(key -> {
                // 从路由获取 routeId
                Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
                String routeId = route != null ? route.getId() : "unknown";

                // 调用限流器判断
                return limiter.isAllowed(routeId, key)
                    .flatMap(response -> {
                        // 记录限流结果到 exchange 属性
                        exchange.getAttributes().put(
                            GATEWAY_REQUEST_RATE_LIMITER_ATTR, response);

                        // 是否允许通过
                        if (response.isAllowed()) {
                            return chain.filter(exchange);
                        }

                        // 被限流：返回 429 Too Many Requests
                        exchange.getResponse().setStatusCode(
                            config.getStatusCode());
                        return exchange.getResponse().setComplete();
                    });
            });
    }
}
```

## 2.3 KeyResolver——限流维度

**内置实现**：

```java
// Principal 维度
public class PrincipalNameKeyResolver implements KeyResolver {
    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return exchange.getPrincipal()
            .map(Principal::getName)
            .switchIfEmpty(Mono.empty());
    }
}
```

**自定义 KeyResolver**：

```java
// IP 维度限流
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> {
        String ip = exchange.getRequest().getRemoteAddress()
            .getAddress().getHostAddress();
        return Mono.just(ip);
    };
}

// 用户 ID 维度限流
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        String userId = exchange.getRequest().getHeaders()
            .getFirst("X-User-Id");
        if (userId == null) {
            return Mono.just("anonymous");
        }
        return Mono.just(userId);
    };
}

// 请求路径 + 维度组合
@Bean
public KeyResolver pathAndIpKeyResolver() {
    return exchange -> {
        String path = exchange.getRequest().getPath().value();
        String ip = exchange.getRequest().getRemoteAddress()
            .getAddress().getHostAddress();
        return Mono.just(ip + ":" + path);
    };
}

// API Key 维度限流
@Bean
public KeyResolver apiKeyResolver() {
    return exchange -> {
        String apiKey = exchange.getRequest().getHeaders()
            .getFirst("X-API-Key");
        return Mono.justOrEmpty(apiKey)
            .defaultIfEmpty("unknown");
    };
}
```

# 3. Redis 限流实现

## 3.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

## 3.2 基于 Redis 的令牌桶实现

```java
public class RedisRateLimiter implements RateLimiter<RedisRateLimiter.Config> {

    private final RedisTemplate<String, String> redisTemplate;
    private final RedisScript<List<Long>> script;

    @Override
    @SuppressWarnings("unchecked")
    public Mono<Response> isAllowed(String routeId, String id) {
        Config config = getConfig().get(routeId);
        if (config == null) {
            return Mono.just(new Response(true, -1));
        }

        int replenishRate = config.getReplenishRate();
        int burstCapacity = config.getBurstCapacity();
        int requestedTokens = config.getRequestedTokens();

        try {
            // 调用 Lua 脚本执行令牌桶逻辑
            List<Long> result = redisTemplate.execute(this.script,
                List.of(this.getKeys(routeId, id)),           // keys
                replenishRate, burstCapacity, requestedTokens,  // args
                1);  // 当前时间（秒）

            boolean allowed = result.get(0) == 1L;
            long tokensLeft = result.get(1);
            return Mono.just(new Response(allowed, tokensLeft));
        } catch (Exception e) {
            // Redis 异常时默认放行（fail-open）
            log.error("Redis rate limiter error", e);
            return Mono.just(new Response(true, -1));
        }
    }
}
```

## 3.3 Lua 脚本核心逻辑

```lua
-- 令牌桶 Lua 脚本（简化版）
-- KEYS[1] = token bucket key
-- KEYS[2] = timestamp key
-- ARGV[1] = replenishRate
-- ARGV[2] = burstCapacity
-- ARGV[3] = requestedTokens
-- ARGV[4] = current timestamp in seconds

local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]

local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

-- 获取当前令牌数和最后时间戳
local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
    last_tokens = capacity  -- 初始化为满桶
end

local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
    last_refreshed = 0
end

-- 计算时间差和应补充的令牌数
local delta = math.max(0, now - last_refreshed)
local filled_tokens = math.min(capacity, last_tokens + (delta * rate))

-- 判断是否允许通过
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
if allowed then
    new_tokens = filled_tokens - requested
end

-- 更新 Redis
redis.call("setex", tokens_key, 2, new_tokens)
redis.call("setex", timestamp_key, 2, now)

-- 返回结果
return { allowed and 1 or 0, new_tokens }
```

## 3.4 配置

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password:
    timeout: 5000ms
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5

  cloud:
    gateway:
      redis-rate-limiter:
        # 全局默认限流配置
        config:
          default:
            replenishRate: 100     # 每秒填充令牌数
            burstCapacity: 200     # 桶容量——可应对的突发请求数
            requestedTokens: 1     # 每次请求消耗的令牌数
          # 按路由 ID 单独配置
          user_service:
            replenishRate: 50
            burstCapacity: 100
          order_service:
            replenishRate: 200
            burstCapacity: 400

        # 是否在响应头中返回限流信息
        include-headers: true
        # 自定义响应头名称
        replenish-rate-header: X-RateLimit-Replenish-Rate
        burst-capacity-header: X-RateLimit-Burst-Capacity
        remaining-header: X-RateLimit-Remaining
        requested-tokens-header: X-RateLimit-Requested-Tokens
```

## 3.5 按路由的限流配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@ipKeyResolver}"
                redis-rate-limiter:
                  replenishRate: 50
                  burstCapacity: 100
```

# 4. 自定义 RateLimiter

## 4.1 基于 Guava RateLimiter 的本地限流

```java
@Component
public class GuavaRateLimiter implements RateLimiter<GuavaRateLimiter.Config> {

    private final Map<String, com.google.common.util.concurrent.RateLimiter> limiters
        = new ConcurrentHashMap<>();

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        Config config = getConfig().get(routeId);
        if (config == null) return Mono.just(new Response(true, -1));

        // 每个 routeId:id 组合对应一个限流器
        String key = routeId + ":" + id;
        com.google.common.util.concurrent.RateLimiter limiter = limiters.computeIfAbsent(
            key, k -> com.google.common.util.concurrent.RateLimiter.create(
                config.getPermitsPerSecond()));

        boolean allowed = limiter.tryAcquire();
        return Mono.just(new Response(allowed, (long) limiter.getRate()));
    }

    @Override
    public Map<String, Config> getConfig() {
        return Collections.singletonMap("default", new Config());
    }

    @Override
    public Class<Config> getConfigClass() {
        return Config.class;
    }

    @Override
    public Config newConfig() {
        return new Config();
    }

    public static class Config {
        private int permitsPerSecond = 100;

        public int getPermitsPerSecond() { return permitsPerSecond; }
        public void setPermitsPerSecond(int permitsPerSecond) {
            this.permitsPerSecond = permitsPerSecond;
        }
    }
}
```

## 4.2 基于滑动窗口的内存限流

```java
@Component
public class SlidingWindowRateLimiter implements RateLimiter<SlidingWindowRateLimiter.Config> {

    // 窗口计数器：key → 时间戳列表
    private final Map<String, Deque<Long>> windows = new ConcurrentHashMap<>();

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        Config config = getConfig().get(routeId);
        if (config == null) return Mono.just(new Response(true, -1));

        String key = routeId + ":" + id;
        long now = System.currentTimeMillis();
        long windowStart = now - config.getWindowSizeMs();

        Deque<Long> timestamps = windows.computeIfAbsent(key, k -> new ConcurrentLinkedDeque<>());

        synchronized (timestamps) {
            // 移除窗口外的记录
            while (!timestamps.isEmpty() && timestamps.peekFirst() < windowStart) {
                timestamps.pollFirst();
            }

            // 检查是否超过阈值
            if (timestamps.size() >= config.getMaxRequests()) {
                return Mono.just(Response.denied());
            }

            timestamps.addLast(now);
            return Mono.just(new Response(true,
                config.getMaxRequests() - timestamps.size()));
        }
    }

    public static class Config {
        private int maxRequests = 100;     // 窗口内最大请求数
        private long windowSizeMs = 1000;  // 窗口大小（毫秒）

        // getters & setters
    }
}
```

# 5. 限流结合分布式缓存

## 5.1 Redisson 分布式限流

```java
@Component
public class RedissonRateLimiter implements RateLimiter<RedisRateLimiter.Config> {

    private final RedissonClient redisson;
    private final Config defaultConfig = new Config();

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        String key = "rate_limiter:" + routeId + ":" + id;
        RRateLimiter rateLimiter = redisson.getRateLimiter(key);

        // 初始化限流器配置
        rateLimiter.trySetRate(RateType.OVERALL,
            defaultConfig.getReplenishRate(),
            defaultConfig.getBurstCapacity(),
            RateIntervalUnit.SECONDS);

        boolean allowed = rateLimiter.tryAcquire();
        return Mono.just(new Response(allowed,
            rateLimiter.availablePermits()));
    }
}
```

# 6. 限流排错与监控

## 6.1 日志排查

```yaml
logging:
  level:
    org.springframework.cloud.gateway.filter.ratelimit: DEBUG
    org.springframework.data.redis.core: DEBUG
```

## 6.2 响应头监控

启用 `include-headers: true` 后，响应头会包含限流信息：

```
X-RateLimit-Replenish-Rate: 100
X-RateLimit-Burst-Capacity: 200
X-RateLimit-Remaining: 45
X-RateLimit-Requested-Tokens: 1
```

# 7. 最佳实践

## 7.1 限流维度选择

| 维度 | KeyResolver | 适用场景 |
|------|------------|----------|
| 客户端 IP | `ipKeyResolver` | 全局限流、防爬虫 |
| 用户 ID | `userKeyResolver` | 用户级别限流 |
| API Key | `apiKeyResolver` | 第三方 API 调用 |
| 路径 + IP | `pathAndIpKeyResolver` | 按接口限流 |

## 7.2 参数计算

```yaml
# 假设接口平均 QPS 为 1000，允许短时突发到 2000
replenishRate: 1000      # 每秒产生 1000 令牌
burstCapacity: 2000      # 桶容量 2000（允许 2 秒的突发量）
requestedTokens: 1       # 每次请求消耗 1 令牌
```

## 7.3 多层级限流

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api_service
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            # 全局 IP 限流（粗粒度）
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@ipKeyResolver}"
                redis-rate-limiter:
                  replenishRate: 1000
                  burstCapacity: 2000
            # 用户级别限流（细粒度）
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@userKeyResolver}"
                redis-rate-limiter:
                  replenishRate: 100
                  burstCapacity: 200
```

## 7.4 限流响应

```java
// 自定义限流响应的过滤器
@Component
public class RateLimitResponseFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange).then(Mono.defer(() -> {
            // 检查是否被限流
            Response rateLimitResponse = exchange.getAttribute(
                RequestRateLimiterGatewayFilterFactory.GATEWAY_REQUEST_RATE_LIMITER_ATTR);
            if (rateLimitResponse != null && !rateLimitResponse.isAllowed()) {
                exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                exchange.getResponse().getHeaders()
                    .setContentType(MediaType.APPLICATION_JSON);
                byte[] body = "{\"code\":429,\"message\":\"Too many requests\"}".getBytes();
                return exchange.getResponse()
                    .writeWith(Mono.just(exchange.getResponse().bufferFactory().wrap(body)));
            }
            return Mono.empty();
        }));
    }

    @Override
    public int getOrder() {
        return NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER - 1;
    }
}
```

## 7.5 注意事项

1. **Redis 故障容错**：Redis 限流器默认 fail-open（Redis 异常时放行），可通过 `RedisRateLimiter` 的 `setFailOpen` 方法控制
2. **令牌桶参数**：`burstCapacity` 建议大于 `replenishRate`，否则失去突发处理能力
3. **限流粒度**：太粗（全局限流）无法精细控制，太细（每个路由每个用户）消耗 Redis 资源
4. **分布式一致性**：基于 Redis 的限流在分布式环境下是一致的
5. **Lua 脚本原子性**：限流操作是原子性的，不会出现并发问题
