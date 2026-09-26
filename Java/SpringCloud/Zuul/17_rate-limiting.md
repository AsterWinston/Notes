
# 1. 限流概述

在网关层实施限流，是保护后端微服务免受突发流量冲击的重要手段。Zuul 本身不内置限流功能，但可以通过以下方式实现：

| 方案 | 实现方式 | 适用场景 |
|------|---------|---------|
| 自定义 ZuulFilter | Pre 过滤器中实现令牌桶/漏桶 | 轻量级、简单场景 |
| Guava RateLimiter | 基于 Guava 的令牌桶 | 单机限流 |
| Bucket4j | 令牌桶算法库，支持分布式 | 分布式限流 |
| Sentinel | 阿里开源，QPS/并发限流 | 生产级多维度限流 |
| Redis + Lua | 分布式计数器/滑动窗口 | 集群限流 |

# 2. 基于 Guava RateLimiter 的单机限流

```java
@Component
public class RateLimitFilter extends ZuulFilter {

    // 为每个服务创建独立的限流器
    private final Map<String, RateLimiter> rateLimiters = new ConcurrentHashMap<>();

    // 服务级别 QPS 配置
    private static final Map<String, Double> SERVICE_QPS = Map.of(
        "user-service", 100.0,    // 用户服务 100 QPS
        "order-service", 50.0,    // 订单服务 50 QPS
        "default", 200.0          // 默认 200 QPS
    );

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 10;  // 鉴权之后、路由之前
    }

    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        String serviceId = (String) ctx.get("serviceId");

        if (serviceId == null) {
            return null;  // 未匹配到服务路由，放行
        }

        // 获取或创建限流器
        RateLimiter rateLimiter = rateLimiters.computeIfAbsent(
            serviceId,
            k -> RateLimiter.create(SERVICE_QPS.getOrDefault(k, SERVICE_QPS.get("default")))
        );

        // 尝试获取令牌
        if (!rateLimiter.tryAcquire()) {
            // 限流——返回 429 Too Many Requests
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(429);
            ctx.setResponseBody(
                "{\"code\":429,\"message\":\"请求过于频繁，请稍后重试\"}");
            ctx.getResponse().setContentType("application/json;charset=UTF-8");

            // 添加 Retry-After 头，告知客户端多久后可重试
            ctx.getResponse().setHeader("Retry-After", "1");
        }

        return null;
    }
}
```

## 2.1 Guava RateLimiter 的平滑突发 vs 预热

```java
// 平滑突发限流（SmoothBursty）
RateLimiter smoothBursty = RateLimiter.create(100.0);  // 100 QPS
// 允许一定程度的突发流量（最高可突增至存储的所有令牌）

// 预热模式（SmoothWarmingUp）
RateLimiter warmingUp = RateLimiter.create(100.0, 3, TimeUnit.SECONDS);
// 启动后逐步增加速率，3 秒后达到 100 QPS
// 适用于冷启动场景，避免服务启动后瞬间被打崩
```

# 3. 令牌桶算法自实现

```java
public class TokenBucket {
    private final long capacity;         // 桶容量
    private final long refillRate;       // 令牌生成速率（令牌/秒）
    private long tokens;                 // 当前令牌数
    private long lastRefillTimestamp;    // 上次填充时间

    public TokenBucket(long capacity, long refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;  // 初始填满
        this.lastRefillTimestamp = System.currentTimeMillis();
    }

    public synchronized boolean tryAcquire(int permits) {
        refill();  // 先补充令牌
        if (tokens >= permits) {
            tokens -= permits;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastRefillTimestamp;
        long newTokens = elapsed * refillRate / 1000;
        if (newTokens > 0) {
            tokens = Math.min(capacity, tokens + newTokens);
            lastRefillTimestamp = now;
        }
    }
}
```

# 4. 基于 Redis 的分布式限流

## 4.1 固定窗口算法

```java
@Component
public class RedisRateLimitFilter extends ZuulFilter {

    private final StringRedisTemplate redisTemplate;

    // Lua 脚本：固定窗口限流
    private static final String FIXED_WINDOW_SCRIPT =
        "local key = KEYS[1]\n" +
        "local limit = tonumber(ARGV[1])\n" +
        "local window = tonumber(ARGV[2])\n" +
        "local current = redis.call('INCR', key)\n" +
        "if current == 1 then\n" +
        "    redis.call('EXPIRE', key, window)\n" +
        "end\n" +
        "if current > limit then\n" +
        "    return 0\n" +
        "end\n" +
        "return 1";

    private final RedisScript<Long> script =
        new DefaultRedisScript<>(FIXED_WINDOW_SCRIPT, Long.class);

    @Override
    public String filterType() { return "pre"; }
    @Override
    public int filterOrder() { return 10; }
    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        String serviceId = (String) ctx.get("serviceId");
        if (serviceId == null) return null;

        String clientIp = getClientIp(ctx.getRequest());
        String key = String.format("rate_limit:%s:%s", serviceId, clientIp);

        Long result = redisTemplate.execute(script,
            List.of(key),
            String.valueOf(100),  // limit: 100 requests
            String.valueOf(1)     // window: 1 second
        );

        if (result == 0L) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(429);
            ctx.setResponseBody("{\"code\":429,\"message\":\"限流\"}");
            ctx.getResponse().setContentType("application/json;charset=UTF-8");
        }

        return null;
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff != null) return xff.split(",")[0].trim();
        return request.getRemoteAddr();
    }
}
```

## 4.2 滑动窗口算法（更平滑）

```lua
-- 滑动窗口限流 Lua 脚本
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])  -- 窗口大小（秒）
local now = tonumber(ARGV[3])     -- 当前时间戳（毫秒）

-- 移除窗口外的记录
local windowStart = now - window * 1000
redis.call('ZREMRANGEBYSCORE', key, 0, windowStart)

-- 统计窗口内的请求数
local count = redis.call('ZCARD', key)

if count >= limit then
    return 0
end

-- 添加当前请求记录
redis.call('ZADD', key, now, now .. '_' .. math.random())
redis.call('EXPIRE', key, window)

return limit - count  -- 返回剩余配额
```

# 5. 基于 Sentinel 的限流

Sentinel 是阿里开源的流量保卫组件，提供比 Hystrix 更完善的限流降级能力。

## 5.1 集成 Sentinel

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.8.6</version>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.8.6</version>
</dependency>
```

## 5.2 Sentinel 限流过滤器

```java
@Component
public class SentinelRateLimitFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }
    @Override
    public int filterOrder() { return 10; }
    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        String serviceId = (String) ctx.get("serviceId");
        if (serviceId == null) return null;

        String resourceName = "zuul_" + serviceId;
        Entry entry = null;

        try {
            entry = SphU.entry(resourceName);
            // 限流通过，继续执行
            return null;
        } catch (BlockException e) {
            // 被限流或降级
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(429);
            ctx.setResponseBody(
                "{\"code\":429,\"message\":\"请求频率过高，已被限流\"}");
            ctx.getResponse().setContentType("application/json;charset=UTF-8");
            return null;
        } finally {
            if (entry != null) {
                entry.exit();
            }
        }
    }
}
```

## 5.3 Sentinel 控制台配置

```bash
# 启动 Sentinel 控制台
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 \
     -jar sentinel-dashboard.jar
```

```yaml
# 应用配置：连接控制台
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
        port: 8719  # 与控制台通信的端口
```

在 Sentinel 控制台中可以直接配置：
- **QPS 限流**：对某个资源限制每秒请求数
- **线程数限流**：对某个资源限制并发线程数
- **预热模式**：冷启动逐步增加流量
- **匀速排队**：请求匀速通过（漏桶算法）

# 6. 多维度的限流策略

```java
@Component
public class MultiDimensionalRateLimitFilter extends ZuulFilter {

    private final Map<String, RateLimiter> perUserLimiters = new ConcurrentHashMap<>();
    private final Map<String, RateLimiter> perIpLimiters = new ConcurrentHashMap<>();
    private final Map<String, RateLimiter> perApiLimiters = new ConcurrentHashMap<>();

    @Override
    public String filterType() { return "pre"; }
    @Override
    public int filterOrder() { return 10; }
    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // 维度一：API 级别限流（所有用户共享）
        String apiKey = "api:" + request.getRequestURI();
        RateLimiter apiLimiter = perApiLimiters.computeIfAbsent(
            apiKey, k -> RateLimiter.create(500.0));
        if (!apiLimiter.tryAcquire()) {
            return reject(ctx, "API 访问过于频繁");
        }

        // 维度二：IP 级别限流（每个 IP 独立）
        String ip = getClientIp(request);
        String ipKey = "ip:" + ip;
        RateLimiter ipLimiter = perIpLimiters.computeIfAbsent(
            ipKey, k -> RateLimiter.create(50.0));
        if (!ipLimiter.tryAcquire()) {
            return reject(ctx, "IP 访问过于频繁");
        }

        // 维度三：用户级别限流
        String userId = request.getHeader("X-User-Id");
        if (userId != null) {
            String userKey = "user:" + userId;
            RateLimiter userLimiter = perUserLimiters.computeIfAbsent(
                userKey, k -> RateLimiter.create(20.0));
            if (!userLimiter.tryAcquire()) {
                return reject(ctx, "用户访问过于频繁");
            }
        }

        return null;
    }

    private Object reject(RequestContext ctx, String message) {
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(429);
        ctx.setResponseBody(String.format(
            "{\"code\":429,\"message\":\"%s\"}", message));
        ctx.getResponse().setContentType("application/json;charset=UTF-8");
        return null;
    }

    private String getClientIp(HttpServletRequest request) { ... }
}
```

# 7. 限流配置化

```yaml
# 自定义限流配置
rate-limit:
  global:
    enabled: true
    default-qps: 200

  routes:
    user-service:
      qps: 100
      burst: 20        # 允许的突发量
      warmup-seconds: 5
    order-service:
      qps: 50
    payment-service:
      qps: 30
      ip-limit: 10     # 每 IP 限制

  per-ip:
    enabled: true
    default-qps: 30

  per-user:
    enabled: true
    default-qps: 20
```

配合配置加载：

```java
@Component
@ConfigurationProperties(prefix = "rate-limit")
public class RateLimitConfig {
    private boolean enabled = true;
    private double defaultQps = 200;
    private Map<String, RouteLimit> routes = new HashMap<>();
    private PerLimit perIp = new PerLimit();
    private PerLimit perUser = new PerLimit();

    public static class RouteLimit {
        private double qps;
        private double burst;
        private int warmupSeconds;
        private double ipLimit;
        // getters/setters
    }

    public static class PerLimit {
        private boolean enabled;
        private double defaultQps;
        // getters/setters
    }
}
```

# 8. 限流响应最佳实践

```java
private void reject(RequestContext ctx, int status, String message) {
    ctx.setSendZuulResponse(false);
    ctx.setResponseStatusCode(status);

    // 统一错误响应体
    Map<String, Object> errorBody = new LinkedHashMap<>();
    errorBody.put("code", status);
    errorBody.put("message", message);
    errorBody.put("timestamp", System.currentTimeMillis());

    // 告知客户端何时可以重试（秒）
    HttpServletResponse response = ctx.getResponse();
    response.setHeader("Retry-After", "5");
    response.setHeader("X-RateLimit-Retry-After", "5");
    response.setContentType("application/json;charset=UTF-8");

    try {
        String json = new ObjectMapper().writeValueAsString(errorBody);
        response.getOutputStream().write(
            json.getBytes(StandardCharsets.UTF_8));
    } catch (Exception e) {
        // 降级：直接设置状态码
    }
}
```
