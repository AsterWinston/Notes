限流和熔断是网关最容易被配错的两块：配松了没效果，配紧了误杀，配错了还可能把故障放大。本篇先把算法讲清，再落到 APISIX 的插件，最后讲重试、超时、降级和压测之间的关系。

# 1. 为什么需要限流与熔断

## 1.1 雪崩链路

```text
某个上游变慢
  ▼
网关连接和 worker 被占满
  ▼
重试把流量放大 2-3 倍
  ▼
上游彻底过载，大量 502/504
  ▼
网关继续重试，正常上游也被拖垮
  ▼
整个站点不可用
```

限流解决"进来太多"，熔断解决"下游已经坏了，别再打"。两者都不是为了挡住用户，而是让系统在过载时**有序退化**。

## 1.2 在网关层做的好处

- 所有服务共享一套规则，不用每个服务各写一遍。
- 可以在路由、Service、Consumer 多个维度计数。
- 不侵入业务代码，调整规则不需要发版。
- 能结合 IP、Header、Consumer 做精细化配额。

★ **关键认知**：网关限流是**第一道闸门**，不是唯一闸门。上游仍需要并发控制、线程池隔离和数据库连接池保护。网关只能挡住入口流量，挡不住内部服务之间的调用洪峰。

# 2. 限流算法

| 算法 | 突发流量 | 输出速率 | APISIX 对应 |
|------|----------|----------|-------------|
| 固定窗口 | 边界处可突刺 | 不恒定 | `limit-count` |
| 滑动窗口 | 平滑 | 较恒定 | 需自实现 |
| 漏桶 | 可排队削峰 | 恒定 | `limit-req` |
| 令牌桶 | 允许一定突发 | 平均恒定 | 常见于自实现 |
| 并发限制 | 不涉及速率 | 由处理速度决定 | `limit-conn` |

**固定窗口**把时间切成一分钟一分钟，每窗独立计数。实现简单，但 59 秒打满 100 次、61 秒再打满 100 次，2 秒内实际通过 200 次。

**漏桶**把请求放进桶里，以固定速率放行，桶满则拒绝：

```text
请求 ──► ┌─────────────┐
         │  桶（burst） │
         │  ● ● ● ●    │
         └──────┬──────┘
                │ 固定速率 rate
                ▼
              上游
```

**令牌桶**以固定速率生成令牌，请求拿到令牌才能通过，桶里可以积累令牌，所以允许一定突发。

★ **关键认知**：`limit-req` 的 `rate` + `burst` 是漏桶模型，不是令牌桶。它允许在 `burst` 范围内排队或突发放行，但**长期平均速率**始终被 `rate` 限制住。

# 3. limit-req

## 3.1 参数与示例

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `rate` | 必填 | 允许速率，单位 req/s |
| `burst` | 必填 | 允许的突发量，桶容量 |
| `key` | `remote_addr` | 计数维度 |
| `key_type` | `var` | `var`、`var_combination`、`constant` |
| `rejected_code` | `503` | 拒绝状态码 |
| `nodelay` | `false` | 突发请求是否立即放行 |
| `policy` | `local` | `local`、`redis`、`redis-cluster` |
| `allow_degradation` | `false` | Redis 不可用时是否放行 |

```json
{
  "limit-req": {
    "rate": 10,
    "burst": 20,
    "key": "remote_addr",
    "key_type": "var",
    "rejected_code": 429,
    "rejected_msg": "too many requests",
    "nodelay": false
  }
}
```

含义：平均每秒放行 10 个请求，最多允许 20 个请求在桶里排队。

## 3.2 nodelay 与排队

| `nodelay` | 行为 | 延迟 | 适用 |
|-----------|------|------|------|
| `false` | 超出 rate 的请求排队等待 | 增加 | 能接受延迟、希望削峰 |
| `true` | 突发立即放行，超过 burst 才拒绝 | 不增加 | 延迟敏感、能承受短时突发 |

⚠ **关键认知**：`nodelay: false` 会让请求在网关里等待，占用连接和 worker 时间。`burst` 设得太大、`rate` 太小，等待会非常长，客户端可能先超时。`burst` 要按"可接受的额外延迟 × rate"估算。

# 4. limit-count

## 4.1 参数

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `count` | 必填 | 窗口内允许的请求数 |
| `time_window` | 必填 | 窗口长度，秒 |
| `key` | `remote_addr` | 计数维度 |
| `policy` | `local` | `local`、`redis`、`redis-cluster` |
| `rejected_code` | `503` | 拒绝状态码 |
| `show_limit_quota_header` | `true` | 返回剩余配额头 |
| `group` | 空 | 共享计数的组名 |
| `allow_degradation` | `false` | Redis 不可用时是否放行 |

```json
{
  "limit-count": {
    "count": 1000,
    "time_window": 3600,
    "key": "consumer_name",
    "key_type": "var",
    "policy": "redis",
    "redis_host": "10.0.0.20",
    "redis_port": 6379,
    "redis_password": "$ENV://REDIS_PASSWORD",
    "redis_timeout": 1000,
    "rejected_code": 429,
    "show_limit_quota_header": true
  }
}
```

## 4.2 local 与 redis

| 策略 | 计数范围 | 优点 | 缺点 |
|------|----------|------|------|
| `local` | 单个 APISIX 实例 | 无外部依赖、快 | 多实例时总配额 = 配置值 × 实例数 |
| `redis` | 所有实例共享 | 配额精确 | 多一次网络往返，Redis 成为依赖 |
| `redis-cluster` | Redis 集群 | 容量大、高可用 | 配置更复杂 |

★ **关键认知**：3 台 APISIX 配 `policy: local`、`count: 1000`，真实总配额是 3000，不是 1000。需要精确全局限额必须用 Redis；如果只是"每台机器各自限流"，local 反而更合适，不会把 Redis 变成单点。

`group` 让多条路由共享同一个计数器：同一个 `group` + 同一个 key 的请求，无论在哪个路由上都消耗同一份配额。

⚠ **关键认知**：`limit-count` 是固定窗口，边界可能出现两倍突刺。对"绝对不允许超过"的场景，要么自实现滑动窗口，要么把窗口切小并接受更频繁的拒绝。

# 5. 并发限流与熔断

## 5.1 limit-conn

`limit-conn` 限制**同时进行中的连接数**，不是速率：

```json
{
  "limit-conn": {
    "conn": 100,
    "burst": 20,
    "default_conn_delay": 0.1,
    "key": "remote_addr",
    "rejected_code": 503
  }
}
```

它适合保护"处理慢、连接占用久"的上游，比如文件导出、长轮询。`limit-req` 管"每秒进来多少"，`limit-conn` 管"同时有多少个没处理完"。

## 5.2 api-breaker 状态机

```text
                连续失败 N 次
   ┌────────┐ ───────────────► ┌────────┐
   │ CLOSED │                  │  OPEN  │
   └────────┘ ◄─────────────── └────────┘
       ▲        连续成功 M 次        │
       │                            │ 超过 max_breaker_sec
       │                            ▼
       │                      ┌───────────┐
       └──────────────────────│ HALF_OPEN │
              探测成功          └───────────┘
```

- **CLOSED**：正常转发，统计失败次数。
- **OPEN**：直接返回 `break_response_code`，不再请求上游。
- **HALF_OPEN**：放少量请求探测，成功达到阈值回 CLOSED，失败则重新 OPEN。

```json
{
  "api-breaker": {
    "break_response_code": 503,
    "break_response_body": "{\"code\":503,\"message\":\"service unavailable\"}",
    "max_breaker_sec": 60,
    "unhealthy": { "http_statuses": [500, 502, 503, 504], "failures": 3 },
    "healthy": { "http_statuses": [200], "successes": 2 }
  }
}
```

| 字段 | 说明 |
|------|------|
| `break_response_code` | 熔断期间返回的状态码，必填 |
| `unhealthy.failures` | 连续失败多少次进入 OPEN |
| `healthy.successes` | 连续成功多少次回到 CLOSED |
| `max_breaker_sec` | OPEN 最长持续秒数，默认 300 |

★ **关键认知**：`api-breaker` 的计数是**单实例**的。3 台 APISIX、阈值 3 次，最坏要累计 9 次失败才会全部熔断。阈值要按实例数摊薄，或接受"每台独立判断"。

⚠ **关键认知**：`break_response_code` 不要设成 502。502 通常被理解为"网关到上游失败"，而熔断是主动拒绝，用 503 更准确，也便于和真实故障区分。

# 6. 重试与超时

## 6.1 重试放大

```text
1 个客户端请求
  ▼ 失败
重试 2 次 ──► 实际打到上游 3 次
  ▼ 上游已过载
3 个请求都失败 ──► 更多重试
```

重试必须满足三个条件才有意义：

1. 请求是幂等的。
2. 失败是瞬时的（连接失败、上游刚重启）。
3. 总预算足够，不会把客户端拖到超时。

## 6.2 超时预算

```text
客户端超时 10s
  └── 网关 retry_timeout 6s
        └── 单次 read 2s
              └── 上游处理 1.5s
```

如果 `retry_timeout` 大于客户端超时，重试还没结束客户端就已经断开，白白浪费上游资源。

```json
{
  "retries": 1,
  "retry_timeout": 3,
  "timeout": { "connect": 1, "send": 2, "read": 2 },
  "plugins": {
    "api-breaker": {
      "break_response_code": 503,
      "unhealthy": { "http_statuses": [500, 502, 503, 504], "failures": 3 },
      "healthy": { "http_statuses": [200], "successes": 2 }
    }
  }
}
```

★ **关键认知**：限流、熔断、重试要一起设计。只加限流不加熔断，上游故障时流量仍会持续打进去；只加重试不加限流，故障时流量会被放大。三者是配套的。

# 7. 降级与压测

## 7.1 降级手段

```json
{
  "fault-injection": {
    "abort": {
      "http_status": 503,
      "body": "{\"code\":503,\"message\":\"degraded\"}",
      "headers": { "X-Degraded": "true" }
    }
  }
}
```

`delay: { "duration": 2 }` 用于演练超时和熔断。`mock` 直接返回固定响应，适合上游未就绪但契约已确定的场景；`redirect` 把请求转到维护页。

⚠ **关键认知**：`mock` 和 `redirect` 会让请求不再到达上游，适合短期降级。长期返回假数据会掩盖真实故障，监控也看不到上游已经挂了。

## 7.2 压测方法

| 工具 | 特点 | 示例 |
|------|------|------|
| `wrk` | 轻量、高并发 | `wrk -t4 -c200 -d60s URL` |
| `wrk2` | 固定速率、精确延迟 | `wrk2 -t4 -c200 -R5000 -d60s URL` |
| `hey` | 简单易用 | `hey -z 60s -c 200 URL` |
| `k6` | 脚本化场景、断言 | 多接口链路压测 |

```bash
# 200 并发压 60 秒，观察 429 比例和 P99
wrk -t4 -c200 -d60s http://127.0.0.1:9080/api/orders/1
```

压测至少记录：RPS、P50/P95/P99、错误率、网关 CPU 和内存、连接数、上游 CPU、Redis 延迟（如果用了分布式限流）。

★ **关键认知**：网关的容量上限往往不是 CPU，而是**连接数、文件描述符和共享内存**。同时观察 `worker_connections` 使用率、`TIME_WAIT` 数量和共享内存是否打满，比只看 QPS 更有意义。

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 配了 `limit-req` 完全不生效 | 插件没挂到路由，或 key 取值相同 | 读回路由，打印 key |
| 限流总量是配置值的 N 倍 | 多实例用了 `local` 策略 | 改用 Redis 策略 |
| 请求大量超时但上游很快 | `nodelay: false` 导致排队 | 调小 `burst` 或设 `nodelay: true` |
| 429 比例异常高 | `rate` 单位理解错（是每秒） | 按秒换算 |
| Redis 挂了后全部 503 | `allow_degradation` 为 false | 按业务决定是否放行 |
| 熔断一直不触发 | 状态码不在 `unhealthy.http_statuses` | 补上实际返回码 |
| 熔断后长时间不恢复 | `max_breaker_sec` 太大或探测仍失败 | 调小时间，检查上游 |
| 上游故障时流量反而更大 | 重试次数过多 | 降 `retries`，加熔断 |

# 9. 练习

1. 配 `limit-req`：`rate: 5`、`burst: 10`，用 `wrk` 以 20 并发压 30 秒，统计 429 比例，再改 `nodelay` 对比延迟。
2. 用 Redis 策略配 `limit-count`，起两个 APISIX 实例指向同一个 Redis，验证总配额是否仍是配置值。
3. 配 `api-breaker`，用 `fault-injection` 持续注入 500，观察 CLOSED 到 OPEN 的时间，以及 `max_breaker_sec` 到期后的恢复行为。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 固定窗口 | 实现简单，边界可突刺，对应 `limit-count` |
| 漏桶 | 恒定输出，可排队，对应 `limit-req` |
| 并发限流 | 管"同时处理中"的数量，对应 `limit-conn` |
| 分布式限流 | `local` 是单实例，跨实例要用 Redis |
| 熔断 | CLOSED → OPEN → HALF_OPEN，按实例独立计数 |
| 重试 | 幂等才重试，会放大流量，要和熔断配合 |
| 降级 | `fault-injection`、`mock`、`redirect` 各有边界 |
| 压测 | 关注 P99、错误率、连接数和共享内存，不只看 QPS |

下一篇进入可观测性，讲清访问日志格式、Prometheus 指标、OpenTelemetry 链路追踪、Admin API 与 Control API 的排错用法，以及 Dashboard 和告警体系。
