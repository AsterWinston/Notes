# 1. 为什么需要限流

限流（rate limiting）是对请求速率进行约束的机制，它是可靠性体系的第一道闸门。结论先行：**限流不是为了拒绝用户，而是为了在过载时保住系统，把有限资源分配给可服务的请求**。

## 1.1 不限流的后果

如果不做任何限流，系统在流量峰值下会连锁崩溃。典型后果如下：

```
不限流的后果：
- 模型配额被打满（其他业务不可用）
- 成本一夜暴涨
- 下游工具被压垮
- 恶意用户刷爆系统
```

### 1.1.1 模型配额被打满

大模型 API 通常按账号或项目分配配额（quota），例如每分钟请求数（RPM，requests per minute）与每分钟 Token 数（TPM，tokens per minute）。某个业务或某个用户突发流量会耗尽共享配额，导致其他业务直接收到 `429 Too Many Requests`。

### 1.1.2 成本一夜暴涨

LLM 调用按 Token 计费。若没有配额约束，一次爬虫式刷取或一次失控的 Agent 循环（agent loop）就可能产生数万元账单。成本失控在财务上等同于故障。

### 1.1.3 下游工具被压垮

Agent 会调用搜索、数据库、内部 API 等工具。这些下游服务的容量通常远小于模型网关，一旦被 Agent 的并发调用打满，会引发跨服务级联故障。

## 1.2 保护的三个对象

限流保护三个对象：**模型配额、下游工具、系统自身**。

| 保护对象 | 风险 | 限流手段 |
|----------|------|----------|
| 模型配额 | 配额耗尽、成本失控 | RPM / TPM 限制 |
| 下游工具 | 被压垮、级联故障 | 工具级 QPS 限制 |
| 系统自身 | 线程耗尽、内存溢出 | 全局并发与队列限制 |

## 1.3 边界与反例

- **反例：只做全局限流**。全局限流无法区分正常业务与恶意刷取，也无法保护单个下游。必须做多维限流。
- **边界：限流不解决容量不足**。若系统常态容量就低于正常流量，限流只是把「慢」变成「快失败」，仍需扩容。
- **边界：限流不能替代鉴权与风控**。恶意请求应被风控拦截，而非仅靠限流。

# 2. 限流的维度

限流必须落在具体的维度上，单一维度无法覆盖所有风险。结论先行：**按「谁在用、用多少、用哪个资源」三个问题选择维度，并组合成多层限流**。

## 2.1 五个维度

| 维度 | 说明 | 例子 |
|------|------|------|
| 用户 | 每用户 QPS/日配额 | 100 次/天 |
| 租户 | 每租户总量 | 10 万 token/天 |
| 模型 | 保护模型配额 | 100 RPM |
| 工具 | 保护下游 | 50 QPS |
| 全局 | 保护系统 | 1000 QPS |

各维度的优先级不同：全局限流保护系统自身，应最先判定；用户与租户限流保护公平性；模型与工具限流保护外部依赖。

## 2.2 策略模型

### 2.2.1 策略定义

限流策略用不可变对象（immutable object）描述，便于配置化与版本管理。

```java
public record RateLimitPolicy(
        int requestsPerMinute,          // 每分钟请求数上限
        long tokensPerDay,              // 每日 Token 上限
        BigDecimal costPerDay           // 每日成本上限（元）
) {
    public static RateLimitPolicy ofDefault() {
        return new RateLimitPolicy(100, 1_000_000L,
                new BigDecimal("50.00"));
    }
}
```

## 2.3 边界与反例

- **反例：维度缺失**。只按用户限流，忽略租户总量，会导致单租户注册大量账号绕过限制。
- **边界：维度越多开销越大**。每个维度都需要一次计数操作，分布式下每个维度都是一次 Redis 往返，需权衡。
- **反例：所有维度同一阈值**。用户级与全局级阈值量纲不同，不能简单复用。

# 3. 四种限流算法

限流算法决定如何计数与判定。结论先行：**令牌桶（token bucket）是最通用的选择，兼顾平均速率与突发；固定窗口实现最简单但有临界问题；滑动窗口最平滑但内存开销大**。

## 3.1 固定窗口

### 3.1.1 机制

把时间切成固定长度的窗口，每个窗口内独立计数，窗口结束计数归零。

```
每分钟 100 次：00:00-00:59 计数，到 00:60 归零
```

- 优点：实现简单，只需一个计数器。
- 缺点：临界问题。

### 3.1.2 临界问题

固定窗口在窗口边界会产生「双倍突发」：第 1 个窗口末尾与第 2 个窗口开头各放行 100 次，则跨窗口的 1 秒内实际放行 200 次。

```
时间轴:  |---- 窗口1 (100) ----|---- 窗口2 (100) ----|
请求:                   └─ 99s~100s 内 100 次 ─┘
                                  +
                        └─ 100s~101s 内 100 次 ─┘
                        = 1 秒内 200 次（超限一倍）
```

## 3.2 滑动窗口

### 3.2.1 机制

统计「最近 60 秒」的请求数，窗口随时间滑动，而非固定切割。

```
统计"最近 60 秒"的请求数
```

- 优点：平滑，无临界问题。
- 缺点：需记录时间戳，内存开销。常见实现有精确时间戳、滑动日志分桶与加权双窗口三种，分桶法在精度与内存间折中。

## 3.3 漏桶

```
请求 → 桶（固定容量）→ 恒定速率流出
```

- 优点：输出速率恒定，适合平滑流量。
- 缺点：无法应对突发，桶满即拒绝。

漏桶（leaky bucket）本质是一个固定容量队列加恒定消费速率，输出速率严格恒定。它适合「下游只接受恒定速率」的场景，例如写数据库。

## 3.4 令牌桶

### 3.4.1 机制

令牌桶以固定速率生成令牌放入桶中，桶有容量上限。请求到来时取走令牌，取不到则拒绝或等待。桶中积累的令牌允许短时突发。

```
令牌以固定速率放入桶（容量 N）
请求取令牌，取不到则拒绝/等待
桶中有积累的令牌 → 允许突发
```

| 参数 | 含义 | 效果 |
|------|------|------|
| 容量 capacity | 桶内最多令牌数 | 决定最大突发量 |
| 补充速率 refillRate | 每秒生成令牌数 | 决定平均速率 |

### 3.4.2 Java 实现

```java
public class TokenBucket {
    private final double capacity;
    private final double refillRate;      // 每秒补充
    private double tokens;
    private long lastRefill;

    public TokenBucket(double capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefill = System.nanoTime();
    }

    public synchronized boolean tryAcquire(int permits) {
        refill();
        if (tokens >= permits) {
            tokens -= permits;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefill) / 1e9;
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefill = now;
    }
}
```

> 注意：`synchronized` 仅适用于单机。分布式需用 Redis + Lua 保证原子性。

## 3.5 算法对比

| 算法 | 突发支持 | 平滑度 | 内存 | 实现复杂度 | 典型场景 |
|------|:--------:|:------:|:----:|:----------:|----------|
| 固定窗口 | 差 | 差 | 极低 | 低 | 简单接口防刷 |
| 滑动窗口 | 中 | 好 | 高 | 中 | 精确配额 |
| 漏桶 | 无 | 最好 | 低 | 中 | 恒定速率下游 |
| 令牌桶 | 好 | 好 | 低 | 中 | 通用限流 |

## 3.6 边界与反例

- **反例：用固定窗口做计费配额**。临界问题会让用户在边界处双倍消耗，账目与承诺不符。
- **边界：令牌桶容量过大等于无限流**。若容量设为一小时的量，突发可瞬间打满下游。
- **边界：`System.nanoTime()` 溢出**。差值用 `long` 计算，通常不会溢出，但跨 JVM 时间不同步在分布式下无意义。

# 4. Java 实现

结论先行：**单机用 Guava RateLimiter 或 Bucket4j，需要分布式则用 Bucket4j + Redis 或自研 Redis Lua，Resilience4j 适合与熔断/重试统一治理**。

## 4.1 Guava RateLimiter（单机）

Guava 的 `RateLimiter` 基于令牌桶，单机、线程安全、API 简洁。它只支持「预热型」与「突发型」两种模式。

```java
RateLimiter limiter = RateLimiter.create(10.0);   // 10 QPS

public String call(String args) {
    if (!limiter.tryAcquire(100, TimeUnit.MILLISECONDS)) {
        throw new TooManyRequestsException("请求过于频繁");
    }
    return tool.execute(args);
}
```

- `create(10.0)`：稳定速率 10 QPS。
- `tryAcquire(timeout)`：在超时内尝试获取，失败返回 `false`。
- 局限：不支持分布式，不支持按 key 动态限流。

## 4.2 Bucket4j（支持分布式）

### 4.2.1 单机

```java
Bucket bucket = Bucket.builder()
        .addLimit(Bandwidth.classic(10, Refill.greedy(10, Duration.ofSeconds(1))))
        .build();

if (!bucket.tryConsume(1)) {
    throw new TooManyRequestsException();
}
```

`Bandwidth.classic` 表示容量 10、每秒补充 10 的经典带宽；`Refill.greedy` 表示均匀补充令牌。

### 4.2.2 分布式

```java
// 分布式（基于 Redis）
RedisBasedProxyManager<String> proxyManager = new RedisBasedProxyManager<>(
        redisClient, expirationAfterWrite);

Bucket bucket = proxyManager.builder()
        .build("user:" + userId,
                () -> BucketConfiguration.builder()
                        .addLimit(Bandwidth.classic(100, Refill.greedy(100,
                                Duration.ofMinutes(1))))
                        .build());

if (!bucket.tryConsume(1)) {
    throw new TooManyRequestsException("每分钟最多 100 次");
}
```

Bucket4j 通过 Redis 执行 Lua 脚本实现原子令牌操作，`expirationAfterWrite` 控制 key 的过期时间，避免僵尸 key 累积。

## 4.3 Resilience4j RateLimiter

Resilience4j 的限流器采用「周期刷新」语义，适合与熔断、重试、舱壁统一装饰。

```java
RateLimiter limiter = RateLimiter.of("model", RateLimiterConfig.custom()
        .limitForPeriod(100)                              // 每周期 100 次
        .limitRefreshPeriod(Duration.ofMinutes(1))        // 周期 1 分钟
        .timeoutDuration(Duration.ZERO)                   // 不等待，立即拒绝
        .build());

Supplier<String> restricted = RateLimiter.decorateSupplier(limiter,
        () -> model.chat(prompt));
```

`timeoutDuration(Duration.ZERO)` 表示不等待，取不到许可立即抛 `RequestNotPermitted`；设为非零值则会在超时内等待。

## 4.4 边界与反例

- **反例：Guava 用在集群**。每个实例独立计数，N 个实例等于把阈值放大 N 倍。
- **边界：Resilience4j RateLimiter 不按 key**。它绑定单个实例，多用户需为每个 key 建独立限流器，注意内存。
- **边界：Redis 故障**。分布式限流依赖 Redis，需决定故障时「放行」还是「拒绝」，通常选择快速失败并降级。

# 5. Agent 场景的限流点

结论先行：**Agent 请求链路长，应在用户、租户、模型、工具四个位置分别限流，形成纵深防御**。

## 5.1 分层限流链路

```
用户请求
   │
   ▼
① 用户级限流（每日次数/Token）
   │
   ▼
② 租户级限流（配额）
   │
   ▼
③ 模型调用限流（RPM/TPM）
   │
   ▼
④ 工具调用限流（保护下游）
```

## 5.2 代码实现

```java
public class AgentRateLimiter {

    public void checkBeforeRequest(UserContext user, String question) {
        // 用户级
        if (!userLimiter.tryAcquire(user.userId())) {
            throw new TooManyRequestsException("请求过于频繁");
        }
        // 租户级配额
        if (!quotaService.hasRemaining(user.tenantId(), estimateCost(question))) {
            throw new QuotaExceededException("租户配额已用尽");
        }
    }

    public void checkBeforeTool(String toolName, UserContext user) {
        if (!toolLimiter.tryAcquire(toolName)) {
            throw new ToolRateLimitedException("工具调用过于频繁: " + toolName);
        }
    }
}
```

## 5.3 边界与反例

- **反例：只在入口限流**。入口限流无法约束 Agent 内部的循环调用（如规划-执行循环），必须在模型与工具处再加限。
- **边界：估算成本与实际不符**。`estimateCost` 基于问题长度，但多轮对话实际消耗可能远超预估，需在响应后校正配额。
- **边界：工具限流粒度**。按工具名限流时，同名工具的不同实例共享配额，可能误伤。

# 6. Token 限流（LLM 特有）

结论先行：**LLM 限流必须同时限制 RPM 与 TPM，因为请求数与 Token 数两个维度会分别被配额约束**。

## 6.1 RPM 与 TPM

LLM 通常按 **RPM（请求/分钟）** 与 **TPM（Token/分钟）** 双重限流：

| 维度 | 含义 | 被谁约束 |
|------|------|----------|
| RPM | 每分钟请求数 | 模型配额 |
| TPM | 每分钟 Token 数 | 模型配额、成本 |

一次长上下文请求可能只占 1 次 RPM，却消耗数万 Token。只限 RPM 会导致 TPM 被击穿。

## 6.2 实现

### 6.2.1 Token 估算

请求前无法得知准确输出 Token，需估算：

> 估算 Token 可用字符数近似：中文约 1 token/字，英文约 1 token/4 字符。

更精确的做法是使用模型对应的分词器（tokenizer），但开销较大，通常用近似即可。

### 6.2.2 双限实现

```java
public class TokenRateLimiter {

    private final Map<String, RateLimiter> rpmLimiters = new ConcurrentHashMap<>();
    private final Map<String, TokenBucket> tpmBuckets = new ConcurrentHashMap<>();

    public boolean tryAcquire(String model, int estimatedTokens) {
        RateLimiter rpm = rpmLimiters.get(model);
        TokenBucket tpm = tpmBuckets.get(model);

        if (!rpm.tryAcquire()) return false;
        if (!tpm.tryAcquire(estimatedTokens)) {
            return false;
        }
        return true;
    }
}
```

> 缺陷：RPM 已扣减但 TPM 失败时不会归还 RPM。生产实现需在 TPM 失败时回补 RPM，或先判定 TPM 再判定 RPM。

## 6.3 边界与反例

- **反例：只限 TPM**。大量小请求虽 Token 少，但会占满并发连接与 RPM，仍需 RPM 限制。
- **边界：估算偏差**。输出 Token 无法预知，应按「输入 + 最大输出」保守预留，或按历史分位数预留。
- **边界：流式响应**。流式（streaming）场景下 Token 逐步产生，需在流结束后按实际用量结算。

# 7. 限流后的处理

结论先行：**限流命中后不应只会返回 429，应根据业务选择拒绝、排队、降级或缓存，最大化可用性**。

## 7.1 四种策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 立即拒绝 | 返回 429 | 保护系统 |
| 排队等待 | 短暂等待重试 | 延迟不敏感 |
| 降级模型 | 换便宜模型 | 保可用性 |
| 返回缓存 | 有缓存则返回 | 幂等查询 |

## 7.2 降级实现

```java
public String chatWithFallback(String prompt) {
    if (!limiter.tryAcquire()) {
        // 降级到小模型
        return fallbackModel.chat(prompt);
    }
    return primaryModel.chat(prompt);
}
```

降级链可进一步扩展：主模型 → 备用模型 → 缓存 → 静态兜底。

## 7.3 边界与反例

- **反例：所有场景都排队**。对实时对话排队会显著增加尾延迟，可能比直接拒绝体验更差。
- **边界：降级模型质量差异**。降级后输出质量下降，需在响应中标注，避免用户误解。
- **边界：缓存穿透**。限流时返回缓存需校验缓存时效与权限，避免越权返回他人数据。

# 8. 限流的用户体验

结论先行：**限流必须给用户明确、可操作的反馈：标准状态码、`Retry-After`、以及可读的错误信息**。

## 8.1 HTTP 语义

限流应返回 `429 Too Many Requests`，并携带 `Retry-After` 头告知客户端何时重试。这是 RFC 7231 定义的语义，客户端与网关均可识别，可辅以 `X-RateLimit-Limit` 与 `X-RateLimit-Remaining` 暴露额度。

## 8.2 全局异常处理

```java
@RestControllerAdvice
public class RateLimitExceptionHandler {

    @ExceptionHandler(TooManyRequestsException.class)
    public ResponseEntity<ErrorResponse> handle(TooManyRequestsException e) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .header("Retry-After", "60")
                .body(new ErrorResponse("RATE_LIMITED",
                        "请求过于频繁，请稍后重试"));
    }
}
```

> 返回 `Retry-After` 让客户端知道何时重试。客户端应遵循指数退避并尊重该值，避免二次冲击。

## 8.3 边界与反例

- **反例：返回 500**。把限流错误当成系统错误会误导客户端重试并触发告警。
- **边界：Retry-After 过长**。若建议重试时间远超客户端超时，用户会直接放弃，需配合降级。
- **边界：错误信息泄露内部阈值**。精确暴露阈值会被攻击者用于探测边界。

# 9. 分布式限流

结论先行：**多实例部署时单机限流会放大阈值，必须使用共享存储实现全局限流，Redis + Lua 是最常见方案**。

## 9.1 为什么需要

单机限流器的计数存在每个 JVM 内存中。若部署 10 个实例、每实例限 100 QPS，则集群实际可承受 1000 QPS，与预期严重不符。

## 9.2 Redis 方案

### 9.2.1 固定窗口 Lua

```lua
-- KEYS[1] = 限流 key, ARGV[1] = 上限, ARGV[2] = 窗口秒数
local current = redis.call('INCR', KEYS[1])
if current == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
  return 0
end
return 1
```

利用 `INCR` 的原子性与首次自增时设置过期时间，实现简单可靠的固定窗口。缺点是仍有临界问题。令牌桶的实现类似：在 Redis 中存储 `tokens` 与上次时间戳，每次请求按经过时间补充令牌，实现分布式令牌桶。

## 9.3 边界与反例

- **反例：多条命令代替 Lua**。`GET` 后再 `SET` 存在竞态，会超发。
- **边界：热点 key**。单用户高频请求会让某个 Redis key 成为热点，可加本地二级限流。
- **边界：Redis 集群分片**。同一 key 必须落在同一分片，否则计数不共享。

# 10. 监控与容量

结论先行：**限流阈值应由容量与成本测算得出，并通过监控触发率持续校准，而非拍脑袋设定**。

## 10.1 指标

| 指标 | 说明 |
|------|------|
| 限流触发率 | 被拒绝请求 / 总请求 |
| 各维度触发数 | 用户/租户/模型/工具分别统计 |
| 拒绝原因分布 | 区分 QPS、配额、Token |
| 队列等待时长 | 排队策略下的等待时间 |

## 10.2 阈值测算与告警

阈值取三方面约束的最小值：下游容量、模型配额、`成本预算 / 单次成本`。告警条件包括：触发率持续超过 5%、某维度触发率骤升、长期零触发。

## 10.3 边界与反例

- **反例：只看总量不看维度**。总量正常但单用户触发率极高，说明存在滥用。
- **边界：动态阈值**。自动调参虽好，但需防止震荡，通常加冷却期。

# 11. 生产注意事项

## 11.1 常见风险表

| 风险 | 建议 |
|------|------|
| 单机限流 | 分布式场景用 Redis 版 |
| 无维度划分 | 用户/租户/模型/工具分别限流 |
| 阈值拍脑袋 | 按容量与成本测算 |
| 无监控 | 监控限流触发率 |
| 拒绝无提示 | 返回 429 + Retry-After |
| 忽略 Token | RPM + TPM 双限 |
| 无降级 | 命中限流时提供降级路径 |

## 11.2 配置清单

```yaml
rate-limit:
  global: { qps: 1000 }
  model: { rpm: 100, tpm: 100000 }
  user: { per-day: 100, per-minute: 10 }
  tool: { default-qps: 50 }
  redis: { enabled: true, key-prefix: "rl:", ttl: 120s }
```

# 12. 小结

- 限流保护模型配额、下游工具与系统自身。
- 维度：用户、租户、模型、工具、全局。
- 令牌桶最常用，支持突发；分布式用 Bucket4j + Redis。
- LLM 需 RPM + TPM 双限。
- 限流后要有降级与友好的用户提示。
- 阈值由容量与成本测算，并用监控持续校准。
