Redis 之所以能一个中间件干这么多活，靠的就是它内置的八种数据结构。选错结构，代码会又慢又绕；选对了，一个命令就能解决别人写几十行的事。本篇把最常用的类型逐个过一遍，每个都给命令、Go 示例和适用场景，最后给一张选型表。

# 1. 键的设计基础

## 1.1 键的命名规范

Redis 是扁平的键空间，没有表、没有 schema，所有结构都靠**键名约定**来区分：

```text
业务:实体:标识:字段

user:1001:profile      用户资料
order:20261105:count   某天的订单计数
rank:article:daily     文章日榜
lock:order:1001        订单相关分布式锁
```

★ **关键认知**：冒号分隔不是 Redis 的语法，而是社区约定。它的价值在于：客户端工具能自动按前缀分组、`SCAN` 时能按业务前缀遍历、排查时一眼看出归属。**键名要能自解释，不要用 `u1`、`k2` 这种短名**。

⚠ **关键认知**：键名长度会占用内存。一个 100 字节的键名存 1000 万个键就是 1GB。过于冗长同样浪费，控制在 30 字节以内比较合理。

## 1.2 先看全景

| 类型 | 底层结构 | 一句话定位 |
|------|----------|-----------|
| String | SDS | 单个值、计数器、位图 |
| Hash | listpack / hashtable | 对象字段，按字段读写 |
| List | quicklist | 有序可重复，双端队列 |
| Set | intset / hashtable | 无序不重复 |
| ZSet | listpack / skiplist | 带分值的有序不重复集合 |
| HyperLogLog | 稀疏/稠密编码 | 基数估算，极小内存 |
| Geo | 基于 ZSet | 地理位置与距离查询 |
| Stream | radix tree | 持久化消息队列 |

# 2. String

## 2.1 常用命令

```bash
SET key value              # 设置
GET key                    # 读取
SETEX key 60 value         # 设置并带过期时间
SETNX key value            # 不存在才设置（分布式锁基础）
INCR key                   # 原子加 1
INCRBY key 10              # 原子加 10
MSET k1 v1 k2 v2           # 批量写
MGET k1 k2                 # 批量读
```

## 2.2 Go 示例

```go
ctx := context.Background()

// 普通读写
err := rdb.Set(ctx, "user:1001:name", "alice", 0).Err() // 0 表示不过期
name, err := rdb.Get(ctx, "user:1001:name").Result()

// 带过期时间的写入，适合缓存
rdb.Set(ctx, "user:1001:profile", jsonStr, 10*time.Minute)

// 计数器：秒杀库存、接口限流
n, err := rdb.Incr(ctx, "api:limit:1001").Result()

// 批量操作，减少网络往返
rdb.MSet(ctx, "a", 1, "b", 2)
vals, err := rdb.MGet(ctx, "a", "b").Result() // []interface{}{"1", "2"}
```

★ **关键认知**：`INCR` 是**原子**的，多个服务实例同时自增不会丢更新。这是 Redis 做计数器和限流的基础。但 `GET` + `SET` 组合**不是**原子的，中间可能被其他客户端插入修改。

## 2.3 过期时间

```go
rdb.Set(ctx, "k", "v", 30*time.Second) // 写入时设过期
rdb.Expire(ctx, "k", time.Minute)       // 单独设置
ttl, err := rdb.TTL(ctx, "k").Result()  // 剩余时间
rdb.Persist(ctx, "k")                   // 去掉过期，变成永久键
```

⚠ **关键认知**：对已存在的键执行 `SET` 会**清掉原来的过期时间**（除非用 `KEEPTTL` 选项）。这是缓存永不过期、内存缓慢上涨的常见原因之一。

# 3. Hash

## 3.1 常用命令

```bash
HSET user:1001 name alice age 30   # 设置多个字段
HGET user:1001 name                # 读一个字段
HMGET user:1001 name age           # 读多个字段
HGETALL user:1001                  # 读全部字段
HDEL user:1001 age                 # 删除字段
HINCRBY user:1001 age 1            # 字段自增
HLEN user:1001                     # 字段数量
```

## 3.2 Go 示例

```go
// 批量写字段
err := rdb.HSet(ctx, "user:1001", map[string]interface{}{
    "name":  "alice",
    "age":   30,
    "city":  "shanghai",
}).Err()

// 读单个字段
age, err := rdb.HGet(ctx, "user:1001", "age").Int()

// 读全部字段，返回 map[string]string
all, err := rdb.HGetAll(ctx, "user:1001").Result()

// 只更新一个字段，不影响其他字段
rdb.HSet(ctx, "user:1001", "age", 31)

// 字段级自增
rdb.HIncrBy(ctx, "user:1001", "login_count", 1)
```

★ **关键认知**：Hash 的最大价值是**字段级读写**。用 String 存 JSON 时，改一个字段要"读全量 → 反序列化 → 改 → 序列化 → 写全量"；用 Hash 一个 `HSET` 就够了。对象字段多、频繁局部更新时优先选 Hash。

⚠ **关键认知**：`HGETALL` 在大 Hash 上会阻塞 Redis。字段数上千时改用 `HSCAN` 渐进遍历，避免单条命令耗时过长。

## 3.3 String 存 JSON 还是 Hash

| 维度 | String + JSON | Hash |
|------|---------------|------|
| 整体读写 | 一次网络往返 | 一次网络往返 |
| 单字段更新 | 全量重写 | 只改该字段 |
| 内存开销 | 略小（无字段名重复） | 字段名有额外开销 |
| 嵌套结构 | 天然支持 | 不支持嵌套 |
| 部分读取 | 需反序列化 | `HMGET` 直接取 |

选择建议：结构嵌套深、整体读写为主 → String；字段多、经常只改一两个字段 → Hash。

# 4. List

## 4.1 常用命令

```bash
LPUSH queue task1          # 左侧入队
RPUSH queue task2          # 右侧入队
LPOP queue                 # 左侧出队
RPOP queue                 # 右侧出队
LRANGE queue 0 -1          # 取范围，-1 表示末尾
LLEN queue                 # 长度
LTRIM queue 0 99           # 只保留前 100 个
BRPOP queue 5              # 阻塞式右弹出，最多等 5 秒
```

## 4.2 Go 示例

```go
// 简单队列：左边进，右边出（FIFO）
rdb.LPush(ctx, "queue:email", "mail-1")
task, err := rdb.RPop(ctx, "queue:email").Result()

// 阻塞消费，适合 worker 循环
for {
    res, err := rdb.BRPop(ctx, 5*time.Second, "queue:email").Result()
    if err == redis.Nil {
        continue // 5 秒内没有新任务，继续等
    }
    if err != nil {
        break // 真正的错误
    }
    handle(res[1]) // res[0] 是键名，res[1] 是值
}

// 保留最近 N 条，做"最新动态"列表
rdb.LPush(ctx, "feed:1001", feedID)
rdb.LTrim(ctx, "feed:1001", 0, 99) // 只留 100 条
```

★ **关键认知**：List 是**双端队列**，`LPUSH + RPOP` 是 FIFO 队列，`LPUSH + LPOP` 是 LIFO 栈。用 `BRPOP` 做阻塞消费比轮询省资源，但它**不保证消息不丢**——弹出即删除，消费者崩溃消息就没了。可靠队列要用 Stream。

# 5. Set

## 5.1 常用命令

```bash
SADD tags go redis          # 添加元素
SMEMBERS tags               # 取全部（大集合慎用）
SISMEMBER tags go           # 是否存在
SREM tags go                # 删除
SCARD tags                  # 元素个数
SINTER s1 s2                # 交集
SUNION s1 s2                # 并集
SDIFF s1 s2                 # 差集
```

## 5.2 Go 示例

```go
rdb.SAdd(ctx, "user:1001:tags", "go", "redis", "backend")
ok, err := rdb.SIsMember(ctx, "user:1001:tags", "go").Result() // true

// 共同关注：两个集合求交集
common, err := rdb.SInter(ctx, "follow:1001", "follow:1002").Result()

// 随机抽奖：抽 3 个不重复
winners, err := rdb.SRandMemberN(ctx, "lottery:1001", 3).Result()

// 注意：SMembers 会一次性返回全部，百万元素会卡住
```

★ **关键认知**：Set 的成员判断是 O(1)，非常适合做"是否存在""是否已点赞""标签去重"。但集合很大时不要用 `SMEMBERS`，改用 `SSCAN` 分批遍历，或者用 `SISMEMBER` 单点判断。

# 6. ZSet

## 6.1 常用命令

```bash
ZADD rank 100 alice 90 bob     # 添加，分值为 score
ZINCRBY rank 5 alice           # 分值自增
ZRANGE rank 0 9 WITHSCORES     # 升序取前 10
ZREVRANGE rank 0 9 WITHSCORES  # 降序取前 10
ZRANK rank alice               # 升序排名
ZSCORE rank alice              # 查分值
ZCOUNT rank 80 100             # 分值区间计数
ZRANGEBYSCORE rank 80 100      # 按分值区间取
ZREM rank alice                # 删除
```

## 6.2 Go 示例

```go
// 排行榜：写入分值
rdb.ZAdd(ctx, "rank:article", redis.Z{Score: 100, Member: "article:1"})
rdb.ZIncrBy(ctx, "rank:article", 5, "article:1")

// 取 top 10（降序）
top, err := rdb.ZRevRangeWithScores(ctx, "rank:article", 0, 9).Result()

// 我的排名（ZRank 是升序，名次要换算）
rank, err := rdb.ZRevRank(ctx, "rank:article", "article:1").Result()

// 延时队列：score 存到期时间戳
rdb.ZAdd(ctx, "delay:queue", redis.Z{
    Score:  float64(time.Now().Add(30 * time.Second).Unix()),
    Member: "task:1001",
})
// 取出已到期的任务
due, err := rdb.ZRangeByScore(ctx, "delay:queue", &redis.ZRangeBy{
    Min: "-inf",
    Max: fmt.Sprint(time.Now().Unix()),
    Offset: 0,
    Count:  10,
}).Result()
```

★ **关键认知**：ZSet 的"分值"可以是任意浮点数，所以它能表达三种语义：**排名**（score 是分数）、**时间排序**（score 是时间戳）、**延时队列**（score 是到期时间）。同一个结构，三种用法，这是 ZSet 最被低估的地方。

⚠ **关键认知**：`ZRANGE` 默认按 score 升序，取"排行榜 top N"要用 `ZREVRANGE`。名次从 0 开始，展示时记得加 1。

# 7. HyperLogLog

## 7.1 用法与原理

HyperLogLog 用于**基数估算**：统计一个集合里有多少个不重复元素，误差约 0.81%，但**固定只占 12KB**，无论元素多少。

```go
rdb.PFAdd(ctx, "uv:20261105", "user:1", "user:2", "user:3") // PFADD
n, err := rdb.PFCount(ctx, "uv:20261105").Result()          // PFCOUNT，约 3
rdb.PFMerge(ctx, "uv:week", "uv:d1", "uv:d2")               // PFMERGE 合并多天
```

★ **关键认知**：HLL 只告诉你"大约多少个不同值"，**不保存原始元素**，所以无法判断某个元素是否在里面，也无法列出成员。用它省内存做 UV 统计很划算，但需要精确 ID 时不行。

# 8. Geo

## 8.1 常用命令与 Go 示例

Geo 底层是 ZSet，把经纬度编码成分值，所以能直接做距离计算。

```go
// 添加位置（经度 longitude 在前，纬度 latitude 在后）
rdb.GeoAdd(ctx, "shop:location",
    &redis.GeoLocation{Name: "shop1", Longitude: 121.47, Latitude: 31.23},
    &redis.GeoLocation{Name: "shop2", Longitude: 121.50, Latitude: 31.25},
)

// 计算两点距离，单位 km
dist, err := rdb.GeoDist(ctx, "shop:location", "shop1", "shop2", "km").Result()

// 找附近 5km 内的店，按距离升序，返回距离
nearby, err := rdb.GeoSearchLocation(ctx, "shop:location", &redis.GeoSearchLocationQuery{
    GeoSearchQuery: redis.GeoSearchQuery{
        Longitude: 121.47,
        Latitude:  31.23,
        Radius:    5,
        Unit:      "km",
        Sort:      "ASC",
    },
    WithDist: true,
}).Result()
```

⚠ **关键认知**：Redis 的 Geo 使用**矩形近似**计算距离，误差在 0.5% 左右。它适合"附近的人""门店查询"这类业务，不适合测绘级精度。真正的球面距离计算要用 PostGIS 等专业方案。

# 9. Stream

## 9.1 为什么需要 Stream

List 做队列有两个硬伤：弹出即删除，消费者崩溃会丢消息；没有消费组，多个消费者无法分工。Stream 是 Redis 5.0 加入的**持久化消息队列**，支持消费者组、ACK、Pending 列表。它的核心命令是 `XADD`（追加）、`XGROUP CREATE`（建组）、`XREADGROUP`（读取）、`XACK`（确认）、`XPENDING`（查未确认）。

## 9.2 Go 示例

```go
// 生产消息
msgID, err := rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "orders",
    Values: map[string]interface{}{"order_id": 1001, "amount": 99},
}).Result()

// 创建消费组（已存在会报 BUSYGROUP，可以忽略）
rdb.XGroupCreateMkStream(ctx, "orders", "g1", "0")

// 消费者循环读取
for {
    streams, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
        Group:    "g1",
        Consumer: "consumer-1",
        Streams:  []string{"orders", ">"}, // > 表示只读新消息
        Count:    10,
        Block:    2 * time.Second,
    }).Result()
    if err == redis.Nil {
        continue // 没有新消息
    }
    if err != nil {
        break
    }

    for _, msg := range streams[0].Messages {
        process(msg) // 业务处理
        // 处理成功后 ACK，失败则不 ACK，留在 Pending 里
        rdb.XAck(ctx, "orders", "g1", msg.ID)
    }
}
```

★ **关键认知**：Stream 的可靠投递靠"**读取 → 处理 → ACK**"三步。消息被读取后进入 Pending 列表，只有 `XACK` 才真正移除。消费者崩溃后，其他消费者可以用 `XCLAIM` 接管超时未确认的消息。这让 Redis 具备了接近专业 MQ 的可靠性。

# 10. 选型表

| 需求 | 首选 | 理由 |
|------|------|------|
| 缓存单个对象 | String（JSON） | 简单、一次读写 |
| 对象局部更新 | Hash | 字段级操作 |
| 接口计数/限流 | String + INCR | 原子自增 |
| FIFO 任务队列 | List 或 Stream | 轻量用 List，可靠用 Stream |
| 去重/标签/点赞 | Set | O(1) 成员判断 |
| 排行榜/延时队列 | ZSet | 分值排序 |
| UV 统计 | HyperLogLog | 12KB 固定内存 |
| 附近的人 | Geo | 内置距离计算 |
| 可靠消息 | Stream | 消费组 + ACK |

# 11. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `WRONGTYPE` | 对 Hash 用了 `GET`，或类型被覆盖 | 先 `TYPE key` 确认类型 |
| 内存持续上涨 | `SET` 覆盖时丢了 TTL | 用 `SET ... EX` 或 `KEEPTTL` |
| `SMEMBERS` 卡顿 | 大集合一次全取 | 改 `SSCAN` |
| 排行榜顺序反了 | `ZRANGE` 升序 | 改用 `ZREVRANGE` |
| Stream 消息重复消费 | 忘记 ACK 或重复读取 | 幂等 + 正确 ACK |
| HLL 数量不准 | 估算误差 | 接受 0.81% 误差或用 Set |

# 12. 动手练习

1. 用 Hash 存一个用户对象，实现"只更新年龄"和"读取全部字段"两个操作。
2. 用 ZSet 实现一个日榜，写入 10 个成员，取出前 3 名和某个成员的名次。
3. 用 HyperLogLog 统计 3 天的 UV，再用 `PFMERGE` 算周 UV。

# 13. 小结

| 类型 | 关键能力 | 别踩的坑 |
|------|----------|----------|
| String | 缓存、计数 | `SET` 会丢 TTL |
| Hash | 字段级读写 | 大 Hash 别 `HGETALL` |
| List | 双端队列 | 弹出即删，不保证不丢 |
| Set | 去重、集合运算 | 大集合别 `SMEMBERS` |
| ZSet | 排序、延时队列 | 注意升序/降序 |
| HLL | 基数估算 | 不保存原始元素 |
| Geo | 距离计算 | 矩形近似有误差 |
| Stream | 可靠消息 | 记得 ACK |

下一篇进入缓存模式，讲 Cache Aside、穿透击穿雪崩、空值缓存、布隆过滤器和过期策略——这些是把上面这些结构真正用到生产时必须回答的问题。