分布式系统里，故障不是"会不会发生"，而是"什么时候发生"。下游超时、实例宕机、流量突增、消息重复，任何一个都能让小问题滚成雪崩。稳定性治理的本质不是消灭故障，而是**把故障限制在局部，并让系统可预期地退化**。本篇按超时、重试、熔断、限流、降级、幂等、最终一致性的顺序，把每道防线讲清楚。

# 1. 故障是常态

## 1.1 雪崩是怎么发生的

```text
请求暴涨 → order 协程全部阻塞在 inventory RPC → 队列积压
        → 上游超时重试放大流量 → inventory 被打垮
        → 故障沿调用链扩散到所有依赖方
```

关键点：**阻塞会沿着调用链反向传播**。一个下游变慢，上游的连接池、协程、队列会被占满，最终整条链路不可用。

## 1.2 治理手段全景

| 手段 | 解决的问题 | 作用位置 |
|------|-----------|----------|
| 超时 | 不无限等待 | 每次调用 |
| 重试 | 偶发失败 | 幂等调用 |
| 熔断 | 下游持续故障 | 客户端 |
| 降级 | 非核心功能不可用 | 业务层 |
| 幂等 | 重复请求 | 写操作 |

★ **关键认知**：这些手段**必须组合使用**，单独任何一个都不够。只加重试会在下游故障时放大流量；只加限流会在下游变慢时仍然阻塞；只加熔断会在流量突增时直接拒绝。正确顺序是：先限流控制入口，再超时控制等待，再熔断隔离故障，最后用降级保住核心。

# 2. 超时

## 2.1 逐层递减

```go
func (s *OrderService) Create(ctx context.Context, req *orderv1.CreateRequest) (*orderv1.CreateReply, error) {
	deadline, ok := ctx.Deadline()
	if !ok {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, time.Second)
		defer cancel()
		deadline, _ = ctx.Deadline()
	}
	remaining := time.Until(deadline)
	if remaining < 100*time.Millisecond {
		return nil, v1.ErrorDeadlineExceeded("BUDGET_EXHAUSTED", "剩余时间不足")
	}
	childCtx, cancel := context.WithTimeout(ctx, remaining/2)
	defer cancel()
	return s.doCreate(childCtx, req) // 下游只拿到一半预算
}
```

## 2.2 超时之后发生了什么

★ **关键认知**：超时只代表**调用方不再等待**，不代表**下游停止执行**。如果不把取消信号传下去，下游可能继续跑完整个业务，造成"用户看到失败，库存却被扣了"。所以必须把 `ctx` 一路传到数据库驱动、HTTP 客户端和 gRPC 调用，让它们响应 `ctx.Done()`。

⚠ **关键认知**：超时值不能拍脑袋。它应该来自**容量与体验的权衡**：P99 耗时是多少、用户能接受多久、重试几次。设得太短会误杀正常请求，设得太长会拖垮上游。上线前用压测数据校准。

# 3. 重试

## 3.1 只重试可重试的错误

| 错误类型 | 是否重试 | 原因 |
|----------|----------|------|
| 连接失败、`UNAVAILABLE` | 是 | 可能瞬时网络抖动 |
| `RESOURCE_EXHAUSTED` | 是（带退避） | 限流，稍后可能恢复 |
| `INVALID_ARGUMENT` | 否 | 参数错误，重试无用 |

## 3.2 退避与抖动

```go
func Retry(max int, base time.Duration) grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply any,
		cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		var err error
		for i := 0; i < max; i++ {
			err = invoker(ctx, method, req, reply, cc, opts...)
			if err == nil || !retryable(err) {
				return err
			}
			delay := base * time.Duration(1<<i)  // 指数退避
			jitter := time.Duration(rand.Int63n(int64(delay / 2)))
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-time.After(delay + jitter): // 抖动打散重试
			}
		}
		return err
	}
}
```

`retryable` 只对 `Unavailable`、`ResourceExhausted`、`Aborted` 返回 true，参数错误和 `NotFound` 不重试。

★ **关键认知**：重试必须带**退避和抖动**。固定间隔重试会让所有客户端在同一时刻再次打向故障实例，形成"重试风暴"。抖动的作用是把重试时间打散，让下游有机会恢复。

## 3.3 重试放大

三层调用如果每层都重试 3 次，一次用户请求最坏会变成 27 次数据库查询：网关 → order（3 次）→ inventory（3 次）→ db（3 次）。

⚠ **关键认知**：重试次数要**全局考虑**，不是每层都设 3 次。常见做法是只在最外层或关键一跳重试，内层快速失败；同时给重试设总预算，超过预算直接返回，不再尝试。

# 4. 熔断

## 4.1 状态机

```text
Closed --失败率超阈值--> Open
  ^                        |
  |-- 试探成功 -- Half-Open <-- 冷却时间到
```

Closed 正常放行并统计失败率；Open 直接拒绝、快速失败；Half-Open 放少量请求试探，成功恢复、失败继续熔断。

## 4.2 实现

```go
var cb = gobreaker.NewCircuitBreaker(gobreaker.Settings{
	Name:        "inventory",
	MaxRequests: 5,                // Half-Open 允许的试探数
	Timeout:     30 * time.Second, // Open 持续多久
	ReadyToTrip: func(c gobreaker.Counts) bool { return c.ConsecutiveFailures >= 5 },
})

func (c *inventoryClient) Deduct(ctx context.Context, req *inventory.DeductRequest) (*inventory.DeductReply, error) {
	result, err := cb.Execute(func() (any, error) { return c.client.Deduct(ctx, req) })
	if err != nil {
		return nil, err // 熔断时返回 gobreaker.ErrOpenState
	}
	return result.(*inventory.DeductReply), nil
}
```

★ **关键认知**：熔断的目的是**保护调用方**，不是惩罚下游。当下游已经不可用，继续等待只会耗尽上游资源；快速失败能把资源释放出来，让上游有时间降级或返回缓存。

⚠ **关键认知**：熔断阈值要按接口分别设置。查询接口失败率 50% 可以熔断，支付接口可能 5% 就必须熔断。用一个全局阈值，要么误伤，要么失灵。

# 5. 限流

## 5.1 算法对比

| 算法 | 特点 | 缺点 |
|------|------|------|
| 固定窗口 | 简单，按时间片计数 | 临界点双倍流量 |
| 滑动窗口 | 平滑，精度高 | 实现稍复杂 |
| 令牌桶 | 允许突发，长期限速 | 需要维护桶状态 |

```go
limiter := rate.NewLimiter(rate.Limit(100), 200) // 100 QPS，允许突发 200

func (s *OrderService) Create(ctx context.Context, req *orderv1.CreateRequest) (*orderv1.CreateReply, error) {
	if !limiter.Allow() {
		return nil, v1.ErrorResourceExhausted("RATE_LIMITED", "请求过于频繁")
	}
	return s.doCreate(ctx, req)
}
```

Kratos 直接用 `ratelimit.Server()` 中间件即可接入 BBR 自适应限流。

## 5.2 本地与分布式

常见方案有四类：单机令牌桶快，但集群总量等于单机乘实例数；Redis 固定窗口全局精确，但每次请求多一次往返；Redis + Lua 原子，但脚本要维护；网关限流统一入口，但只保护入口。

Redis 固定窗口的核心是一条 Lua 脚本：`INCR` 计数，首次计数时 `PEXPIRE` 设置窗口，返回当前值；Go 侧调用 `Eval` 并判断是否超过阈值。Lua 保证"计数 + 过期"原子执行，避免并发下漏设过期时间。

★ **关键认知**：限流要分层。网关做**粗粒度**的全局限流，服务做**细粒度**的接口级限流，数据库连接池做**资源级**限制。只在最外层限流，内层仍可能被绕过；只在最内层限流，入口已经被打满。

# 6. 降级

## 6.1 降级层级

| 级别 | 做法 | 示例 |
|------|------|------|
| 功能降级 | 关闭非核心功能 | 关闭推荐、评论 |
| 读降级 | 返回缓存或默认值 | 商品详情读本地缓存 |
| 写降级 | 转异步或排队 | 下单先落消息队列 |

## 6.2 开关设计

开关用 `atomic.Bool` 保存，读取无锁；业务代码在数据库失败且开关打开时改读缓存。开关要能通过配置中心热更新，并且**默认值必须是安全值**：配置中心挂了，系统应该保持"核心可用、非核心关闭"，而不是全部打开。

⚠ **关键认知**：降级不是"随便返回一个空值"。返回空列表会让用户以为"没有数据"，返回错误又会让用户以为"系统坏了"。降级结果要有明确语义，例如"当前人多，推荐暂不可用"，并记录降级日志与指标。

# 7. 幂等

## 7.1 为什么必须幂等

网络超时后客户端重试、消息队列至少一次投递、用户重复点击，都会产生重复请求。**写接口不幂等，就会重复扣款、重复下单、重复发货**。

## 7.2 幂等键

```text
客户端生成 idempotency-key → 服务端以 key 做唯一索引落库
  ├── 插入成功 → 执行业务
  └── 插入冲突 → 查询已有结果返回
```

```sql
CREATE TABLE idempotency (
  idem_key   VARCHAR(64) PRIMARY KEY,
  biz_type   VARCHAR(32) NOT NULL,
  status     TINYINT     NOT NULL DEFAULT 0, -- 0 处理中 1 成功 2 失败
  result     TEXT,
  created_at DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

```go
func (s *OrderService) Create(ctx context.Context, req *orderv1.CreateRequest) (*orderv1.CreateReply, error) {
	key := req.GetIdempotencyKey()
	inserted, err := s.repo.InsertIdempotency(ctx, key, "order.create")
	if err != nil {
		return nil, err
	}
	if !inserted { // 已存在：返回旧结果或处理中
		record, _ := s.repo.GetIdempotency(ctx, key)
		if record.Status == 0 {
			return nil, v1.ErrorAborted("PROCESSING", "请求处理中")
		}
		return &orderv1.CreateReply{OrderId: record.OrderID}, nil
	}
	order, err := s.doCreate(ctx, req)
	if err != nil {
		_ = s.repo.MarkIdempotencyFailed(ctx, key)
		return nil, err
	}
	_ = s.repo.MarkIdempotencySuccess(ctx, key, order.ID)
	return &orderv1.CreateReply{OrderId: order.ID}, nil
}
```

## 7.3 状态机幂等

状态机幂等靠条件更新：`UPDATE orders SET status = 'PAID' WHERE id = ? AND status = 'PENDING'`。影响行数为 0 说明已经被处理过，直接返回成功即可。这比"先查再改"更安全，因为它是**原子**的。

★ **关键认知**：幂等的本质是**让重复请求产生与一次请求相同的结果**，不是"阻止重复请求"。用唯一索引、状态机条件更新、去重表都可以实现；关键是选一个在并发下仍然成立的机制。

# 8. 最终一致性

## 8.1 本地消息表（Outbox）

跨服务写操作不能用一个本地事务覆盖，常用"本地事务 + 事件"：

```sql
CREATE TABLE outbox (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  aggregate_id BIGINT      NOT NULL,
  event_type   VARCHAR(64) NOT NULL,
  payload      JSON        NOT NULL,
  status       TINYINT     NOT NULL DEFAULT 0, -- 0 待发送 1 已发送
  created_at   DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

```text
① 本地事务：写 orders + 写 outbox（同一数据库，原子）
② Relay 轮询 outbox 发 Kafka，成功后标记 status = 1
③ 下游按 event_id 幂等消费
```

★ **关键认知**：Outbox 解决的是"**业务写入与消息发送不能同时成功**"的问题。先写库再发消息，可能写库成功、发消息失败；先发消息再写库，可能消息发了、写库回滚。把消息写进同一个本地事务，再由后台可靠投递，才能保证最终一致。

## 8.2 补偿与对账

| 机制 | 作用 | 实现 |
|------|------|------|
| Saga | 长事务拆成多个本地事务，失败时反向补偿 | 编排或协同 |
| 对账 | 定时比对两侧数据 | 日终/准实时 |
| 重试队列 | 失败消息延后重投 | 带最大次数与死信 |

⚠ **关键认知**：最终一致性不是"不管了，等它自己一致"。必须有**补偿、重试、对账、告警**四件套，否则不一致会静默存在，直到用户投诉才被发现。

# 9. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 下游故障拖垮上游 | 没有超时/熔断 | 超时 + 熔断 + 快速失败 |
| 重试把下游打死 | 固定间隔、无上限 | 指数退避 + 抖动 + 预算 |
| 重复下单 | 写接口无幂等 | 幂等键 + 唯一索引 |
| 超时后数据不一致 | 取消信号没传下去 | ctx 贯穿全链路 |

# 10. 练习

1. 给一个 RPC 调用加上超时、重试、熔断三层保护，用故障注入工具模拟下游 50% 失败，观察调用方成功率。
2. 用 Redis + Lua 实现一个全局限流器，要求 100 QPS、允许 200 的突发。
3. 设计下单接口的幂等方案：写出表结构、请求流程，以及"处理中"状态下客户端应该怎么处理。

# 11. 小结

| 主题 | 要点 |
|------|------|
| 超时 | 逐层递减，取消信号传到底 |
| 重试 | 只重试可重试错误，退避 + 抖动 |
| 熔断 | Closed/Open/Half-Open，按接口设阈值 |
| 降级 | 明确语义，默认安全 |
| 幂等 | 唯一索引、状态机、去重表 |

下一篇进入部署，讲配置管理、健康检查、优雅关闭、Docker 多阶段构建、Kubernetes 探针与滚动发布，让服务能安全地上线。
