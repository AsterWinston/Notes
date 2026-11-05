缓存不是"加个 Redis 就快了"。真正上线后，你很快会遇到这些问题：为什么缓存和数据库不一致？为什么一个不存在的 ID 把数据库打挂了？为什么缓存集体失效的瞬间服务崩了？本篇讲的是**缓存的使用模式**，回答的是"什么时机读写缓存、失效了怎么办、如何防止缓存把数据库拖垮"。

# 1. 缓存的三种读写模式

## 1.1 Cache Aside（旁路缓存）

业务代码自己管理缓存和数据库，最常用：

```text
读：先读缓存 → 命中返回
             → 未命中读数据库 → 写回缓存 → 返回

写：先更新数据库 → 再删除缓存（不是更新缓存）
```

```go
func GetUser(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    // 1. 先查缓存
    data, err := rdb.Get(ctx, key).Result()
    if err == nil {
        var u User
        json.Unmarshal([]byte(data), &u)
        return &u, nil // 命中直接返回
    }
    if err != redis.Nil {
        return nil, err // 真正的 Redis 错误，交给上层决策
    }

    // 2. 未命中，查数据库
    u, err := db.QueryUser(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. 回写缓存，设置过期时间兜底
    b, _ := json.Marshal(u)
    rdb.Set(ctx, key, b, 10*time.Minute)
    return u, nil
}

func UpdateUser(ctx context.Context, u *User) error {
    if err := db.UpdateUser(ctx, u); err != nil {
        return err
    }
    // 先更新库，再删缓存
    return rdb.Del(ctx, fmt.Sprintf("user:%d", u.ID)).Err()
}
```

★ **关键认知**：写操作要**删缓存**而不是**更新缓存**。原因有二：更新缓存需要把数据库的最新值完整算一遍，成本高；并发写时，两个更新到达缓存的顺序可能和数据库相反，导致缓存里是旧值。删除是"下次读时再加载"，天然避免了顺序问题。

⚠ **关键认知**：先更新数据库、再删缓存，在极端并发下仍可能不一致：读请求在缓存失效后读到旧库值，写请求随后删了缓存，读请求又把旧值写回。彻底解决要靠**延迟双删**（写完等几百毫秒再删一次）或**订阅数据库 binlog** 异步失效。

## 1.2 Read Through / Write Through

把缓存当成"主要数据源"，业务只和缓存交互，缓存层自己负责同步数据库：

```text
Read Through ：缓存未命中 → 缓存组件自己加载数据库 → 写回缓存
Write Through：写请求 → 同时写缓存和数据库（同步）
```

这与 Cache Aside 的区别是**谁来管加载和回写**。Cache Aside 是业务代码管，Read Through 是缓存层管。Go 生态里通常用 `cache` 库或自研封装实现，Redis 本身不提供这个能力。

| 模式 | 谁管理缓存 | 一致性 | 复杂度 |
|------|-----------|--------|--------|
| Cache Aside | 业务代码 | 最终一致 | 低，最常用 |
| Read Through | 缓存组件 | 最终一致 | 中，需封装 |
| Write Through | 缓存组件 | 较强 | 中，写放大 |
| Write Behind | 缓存组件 | 弱，可能丢数据 | 高，异步刷盘 |

★ **关键认知**：Write Behind（写回）先把数据写缓存，异步批量刷数据库，写性能最好，但缓存宕机会丢数据。金融、订单类场景不要用；统计、日志类可以接受。

# 2. 三大缓存故障

## 2.1 缓存穿透：查一个不存在的数据

```text
请求 id = -1（数据库里根本没有）
   │
   ▼
缓存未命中 ──→ 数据库也没有 ──→ 不写缓存
   │
   ▼
下一个同样的请求…… 又打数据库
```

恶意攻击者用大量不存在的 ID 请求，缓存永远命中不了，压力全打到数据库。

**方案一：缓存空值**

```go
u, err := db.QueryUser(ctx, id)
if errors.Is(err, sql.ErrNoRows) {
    // 数据库确认不存在，缓存一个短 TTL 的空标记
    rdb.Set(ctx, key, "NULL", 2*time.Minute)
    return nil, ErrNotFound
}
```

**方案二：布隆过滤器**

在缓存前加一层布隆过滤器，判断"这个 key 可能存在 / 一定不存在"：

```go
// 伪代码：启动时把数据库中所有合法 ID 灌入布隆过滤器
bf := bloom.NewWithEstimates(1_000_000, 0.01) // 100 万元素，1% 误判率
for _, id := range allUserIDs {
    bf.Add([]byte(strconv.FormatInt(id, 10)))
}

func GetUserWithBloom(ctx context.Context, id int64) (*User, error) {
    if !bf.Test([]byte(strconv.FormatInt(id, 10))) {
        return nil, ErrNotFound // 一定不存在，直接返回
    }
    return GetUser(ctx, id) // 可能存在，走正常缓存流程
}
```

| 方案 | 优点 | 缺点 |
|------|------|------|
| 缓存空值 | 实现简单 | 大量随机 key 会占用内存 |
| 布隆过滤器 | 内存极小 | 有误判，不支持删除 |
| 参数校验 | 从源头拦截 | 只能防明显非法值 |

★ **关键认知**：布隆过滤器只会**误判存在**（假阳性），不会**漏判存在**（假阴性）。所以它能安全地拦截"一定不存在"的请求，放行"可能存在"的请求。误判率由位数组大小和哈希函数个数决定。

## 2.2 缓存击穿：热点 key 突然失效

某个热点 key（比如首页配置）过期瞬间，大量并发请求同时未命中，一起冲向数据库。

**方案一：互斥锁，只放一个请求回源**

```go
func GetWithMutex(ctx context.Context, key string, load func() (string, error)) (string, error) {
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return val, nil
    }
    if err != redis.Nil {
        return "", err
    }

    // 只有抢到锁的请求去查数据库
    lockKey := "lock:" + key
    ok, err := rdb.SetNX(ctx, lockKey, "1", 5*time.Second).Result()
    if err != nil {
        return "", err
    }
    if !ok {
        // 没抢到锁，短暂等待后重试读缓存
        time.Sleep(50 * time.Millisecond)
        return rdb.Get(ctx, key).Result()
    }
    defer rdb.Del(ctx, lockKey) // 简化写法，生产要用 Lua 校验唯一值

    val, err = load() // 回源
    if err != nil {
        return "", err
    }
    rdb.Set(ctx, key, val, 10*time.Minute)
    return val, nil
}
```

**方案二：singleflight，进程内合并请求**

```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func GetWithSingleflight(ctx context.Context, key string) (string, error) {
    val, err, _ := g.Do(key, func() (interface{}, error) {
        // 同一个 key 的并发调用只会执行一次
        return loadFromDBAndCache(ctx, key)
    })
    if err != nil {
        return "", err
    }
    return val.(string), nil
}
```

★ **关键认知**：`singleflight` 只解决**单实例内**的重复回源；多实例部署时，每个实例仍会各回源一次。要全局只回源一次，必须用 Redis 分布式锁，或者接受"每实例一次"的少量回源。

## 2.3 缓存雪崩：大量 key 同时失效

两类原因：**同一时刻批量写入且 TTL 相同**，或 **Redis 整体宕机**。

```go
// 错误示范：所有缓存都设 10 分钟，同一批写入会同时过期
rdb.Set(ctx, key, val, 10*time.Minute)

// 正确做法：TTL 加随机抖动
ttl := 10*time.Minute + time.Duration(rand.Intn(120))*time.Second
rdb.Set(ctx, key, val, ttl)
```

**应对 Redis 整体不可用**：

- 本地缓存（如 `ristretto`、`freecache`）做二级缓存，Redis 挂了还能顶一阵。
- 熔断降级：连续失败后直接返回兜底数据，不要继续压数据库。
- 限流：在数据库前加并发/速率限制。
- 高可用部署：主从 + Sentinel 或 Cluster，减少单点故障时间。

★ **关键认知**：雪崩的本质是"**失效时间集中 + 回源能力不足**"。抖动 TTL 解决前者，限流、熔断、本地缓存解决后者。只加抖动不做降级，Redis 真挂了照样崩。

# 3. 过期策略

## 3.1 Redis 的两种删除方式

| 策略 | 机制 | 特点 |
|------|------|------|
| 惰性删除 | 访问 key 时才检查是否过期 | 不占 CPU，但过期 key 可能长期占内存 |
| 定期删除 | 周期性随机抽样检查并删除 | 折中，抽样而非全扫 |

Redis 把两者结合使用，所以 key 过期后**不会立刻从内存消失**，而是等被访问或被抽样到。

⚠ **关键认知**：`TTL` 返回 -1 表示**键存在但没设过期**，-2 表示**键不存在**。很多"判断键是否过期"的 bug 就出在没区分这两种情况。

## 3.2 过期策略的选择

| 数据 | 建议 TTL | 理由 |
|------|----------|------|
| 热点配置 | 5-30 分钟 + 抖动 | 允许短时不一致 |
| 用户资料 | 10-60 分钟 + 抖动 | 变更不频繁 |
| 会话 token | 与登录有效期一致 | 安全相关 |
| 验证码 | 5 分钟 | 业务要求 |
| 排行榜 | 按周期（日榜 24h） | 可重建 |
| 空值缓存 | 1-5 分钟 | 防穿透但不能长期占用 |

# 4. 一致性的现实取舍

缓存和数据库是两套系统，**无法做到强一致**，只能做"最终一致 + 短暂不一致窗口可接受"。

```text
            写请求
              │
              ▼
      ┌───────────────┐
      │   更新数据库   │
      └───────┬───────┘
              ▼
      ┌───────────────┐
      │   删除缓存     │  ← 让下次读重新加载
      └───────┬───────┘
              ▼
        读请求回源最新数据
```

更严格的做法是**订阅 binlog**（Canal、Debezium），数据库变更后异步删缓存。这样缓存失效由数据变更驱动，不依赖业务代码是否记得删。

★ **关键认知**：不要追求"缓存和数据库实时一致"。先问业务能接受多长的不一致窗口：商品详情能接受 1 分钟，库存扣减不能。**需要强一致的字段压根不要缓存**，直接读库。

# 5. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 缓存和库不一致 | 先删缓存后更库、并发回写旧值 | 改"先更库后删缓存"+ 延迟双删 |
| 内存持续增长 | `SET` 覆盖丢了 TTL、空值缓存太多 | 检查 TTL、给空值短过期 |
| 数据库被不存在 ID 打挂 | 缓存穿透 | 空值缓存 + 布隆过滤器 |
| 热点失效瞬间 DB 飙高 | 缓存击穿 | 互斥锁 / singleflight |
| 大量 key 同时过期 | TTL 未加抖动 | 随机化 TTL |
| 缓存里出现旧 JSON 字段 | 结构体新增字段但缓存未失效 | 版本化 key 前缀或强刷 |

⚠ **关键认知**：修改缓存里 JSON 的结构时，旧缓存不会自动适配新结构。要么在 key 里加版本号（`user:v2:1001`），要么发布时批量删除旧 key，否则会出现"反序列化后字段为零值"的隐蔽 bug。

# 6. 动手练习

1. 实现一个带空值缓存的 `GetUser`，数据库返回 `sql.ErrNoRows` 时写 `"NULL"`，TTL 2 分钟。
2. 用 `singleflight` 包装一个回源函数，起 100 个 goroutine 同时请求同一个 key，统计实际回源次数。
3. 给缓存 TTL 加 0-120 秒随机抖动，对比固定 TTL 的过期分布。

# 7. 小结

| 概念 | 要点 |
|------|------|
| Cache Aside | 读穿缓存、写更库后删缓存 |
| Read/Write Through | 缓存层负责同步，业务不感知 |
| 缓存穿透 | 不存在的数据 → 空值缓存 / 布隆过滤器 |
| 缓存击穿 | 热点 key 失效 → 互斥锁 / singleflight |
| 缓存雪崩 | 批量失效或宕机 → TTL 抖动 + 降级 |
| 过期策略 | 惰性 + 定期删除，TTL 要随机化 |
| 一致性 | 只求最终一致，强一致字段不要缓存 |

下一篇进入分布式锁，讲清楚 `SET NX EX`、唯一值、Lua 释放、续期、Redlock 争议这些"听起来简单、写对很难"的细节。