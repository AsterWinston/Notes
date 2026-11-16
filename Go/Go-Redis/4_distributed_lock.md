分布式锁看起来只是"抢一个 key"，但真正写对要回答一串问题：锁的过期时间设多长？释放时怎么保证删的是自己的锁？业务没跑完锁过期了怎么办？Redis 主从切换会不会导致两个客户端同时持锁？本篇按"最小可用 → 生产可用 → 争议与边界"的顺序，把这些坑逐个拆开。

# 1. 为什么需要分布式锁

## 1.1 单机锁的边界

`sync.Mutex` 只能保护**同一个进程内**的并发：

```text
┌───────────────────┐     ┌───────────────────┐
│   实例 A          │     │   实例 B          │
│   sync.Mutex ✓    │     │   sync.Mutex ✓    │  ← 各自加锁成功
└─────────┬─────────┘     └─────────┬─────────┘
          │                         │
          └──────────┬──────────────┘
                     ▼
              ┌─────────────┐
              │  共享数据库  │  ← 两个实例同时改，冲突
              └─────────────┘
```

★ **关键认知**：多实例部署时，进程内的锁互相看不见。分布式锁的本质是找一个**所有实例都能访问的共享存储**来做仲裁，Redis 因为"单命令原子 + 高性能"成为最常用的选择。

## 1.2 锁的三个基本要求

| 要求 | 含义 |
|------|------|
| 互斥 | 同一时刻只有一个持有者 |
| 防死锁 | 持有者崩溃后锁能自动释放 |
| 防误删 | 只能释放自己持有的锁 |

# 2. 最小可用版本：SET NX EX

## 2.1 一条命令同时完成加锁和过期

```bash
SET lock:order:1001 <唯一值> NX EX 30
```

- `NX`：只有 key 不存在时才设置，保证互斥。
- `EX 30`：30 秒后自动过期，防止持有者崩溃后死锁。
- `<唯一值>`：每个客户端生成不同的值，用于释放时校验身份。

```go
func TryLock(ctx context.Context, key, token string, ttl time.Duration) (bool, error) {
    // SetNX 在 go-redis v9 中就是 SET key value NX EX ttl
    return rdb.SetNX(ctx, key, token, ttl).Result()
}
```

★ **关键认知**：**绝对不要**用 `SETNX` + `EXPIRE` 两条命令加锁。中间如果进程崩溃，锁就永久不过期，形成死锁。`SET key value NX EX ttl` 是单条命令，原子完成"判断 + 设置 + 过期"。

## 2.2 唯一值从哪来

```go
import "github.com/google/uuid"

token := uuid.NewString() // 例如 8f14e45f-...
```

用 `uuid`、随机数或者"服务名 + 请求 ID"都可以。关键是**每次加锁都不同**，这样释放时才能确认"锁还是我加的那把"。

⚠ **关键认知**：用固定的 `1` 当锁值是最常见的错误。A 的锁过期后 B 拿到锁，A 醒来直接 `DEL`，就把 B 的锁删了。唯一值是为了让 A 在删除前发现"值已经不是我的了"。

# 3. 安全释放：Lua 脚本

## 3.1 为什么不能 GET 再 DEL

```text
A 持有锁（token=A）
   │
   │ 锁过期
   ▼
B 加锁成功（token=B）
   │
   │ A 执行 GET lock → 发现存在（其实是 B 的）→ 判断通过
   ▼
A 执行 DEL lock → 删掉了 B 的锁
```

GET 和 DEL 之间不是原子操作，中间可能发生锁过期和他人加锁。所以必须用 Lua，把"比较值"和"删除"合成一个原子操作：

```lua
-- release.lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```go
var releaseScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
`)

func Unlock(ctx context.Context, key, token string) (bool, error) {
    n, err := releaseScript.Run(ctx, rdb, []string{key}, token).Int64()
    if err != nil {
        return false, err
    }
    return n == 1, nil // 1 表示确实删掉了自己的锁
}
```

★ **关键认知**：Redis 执行 Lua 脚本时是**单线程串行**的，脚本运行期间不会插入其他命令。这就是"GET 比较 + DEL"能变成原子操作的原因。⚠ 但脚本里的命令要短，长脚本会阻塞整个 Redis 实例。

## 3.2 封装成一个锁对象

```go
type RedisLock struct {
    rdb   *redis.Client
    key   string
    token string
    ttl   time.Duration
}

func NewLock(rdb *redis.Client, key string, ttl time.Duration) *RedisLock {
    return &RedisLock{
        rdb:   rdb,
        key:   key,
        token: uuid.NewString(), // 每个锁实例唯一的身份
        ttl:   ttl,
    }
}

func (l *RedisLock) Lock(ctx context.Context) (bool, error) {
    return l.rdb.SetNX(ctx, l.key, l.token, l.ttl).Result()
}

func (l *RedisLock) Unlock(ctx context.Context) (bool, error) {
    n, err := releaseScript.Run(ctx, l.rdb, []string{l.key}, l.token).Int64()
    return n == 1, err
}
```

# 4. 锁续期

## 4.1 为什么需要续期

锁的 TTL 是加锁时固定的。如果业务执行时间不确定，就陷入两难：

| TTL 设置 | 风险 |
|----------|------|
| 太短 | 业务没跑完锁就过期，别的实例进来 |
| 太长 | 持有者崩溃后，其他人要等很久才能拿到锁 |

续期（watchdog）的思路：加锁成功后起一个后台 goroutine，每隔 `TTL/3` 检查一次"锁还是我的吗"，如果是就重置 TTL。

```go
func (l *RedisLock) startRenew(ctx context.Context) (stop func()) {
    ctx, cancel := context.WithCancel(ctx)
    go func() {
        ticker := time.NewTicker(l.ttl / 3) // 每 1/3 TTL 续一次
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                // 只有值仍是我的才续期，避免给别人的锁续命
                ok, err := renewScript.Run(ctx, l.rdb,
                    []string{l.key}, l.token, int(l.ttl.Seconds())).Int64()
                if err != nil || ok == 0 {
                    return // 锁已丢失，停止续期
                }
            }
        }
    }()
    return cancel
}
```

```lua
-- renew.lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("EXPIRE", KEYS[1], ARGV[2])
else
    return 0
end
```

★ **关键认知**：续期只能延长"仍然属于自己"的锁。如果锁已经被别人拿走，续期必须失败并让业务感知。⚠ 后台续期不能掩盖"业务已经卡死"的事实——如果 goroutine 一直续命，锁就永远不释放，别的实例会一直等。

## 4.2 续期不是万能药

带 watchdog 的锁（如 Java 的 Redisson）看起来很美，但有两个前提：

1. 续期进程和业务进程同生共死。进程被杀，续期也停，锁才会过期。
2. 业务必须能被及时取消。GC 停顿、网络阻塞时，业务线程可能已经"假死"，但续期线程还在跑。

所以生产上更稳妥的做法是：**给业务设明确的超时**，TTL 设为"业务最大耗时的 2-3 倍"，而不是无限续期。

# 5. 锁超时与业务超时

## 5.1 两者的关系

```text
时间轴
0s          10s                30s
├───────────┼──────────────────┤
│ 业务执行   │  锁已过期，他人可进入
│ 持锁       │
└───────────┘
```

★ **关键认知**：锁的 TTL 必须**大于业务的最坏执行时间**。否则会出现"A 还在写数据库，B 已经拿到锁也在写"，锁形同虚设。正确做法是把业务操作包在带超时的 Context 里，并让 TTL 明显大于这个超时：

```go
const businessTimeout = 10 * time.Second
const lockTTL = 30 * time.Second // 3 倍余量

ctx, cancel := context.WithTimeout(context.Background(), businessTimeout)
defer cancel()

lock := NewLock(rdb, "lock:order:1001", lockTTL)
ok, err := lock.Lock(ctx)
if err != nil || !ok {
    return errors.New("获取锁失败")
}
defer lock.Unlock(ctx)

return doBusiness(ctx) // 业务自己有 10 秒上限，锁 30 秒
```

## 5.2 锁过期后的补救

即使 TTL 足够长，也可能出现锁已过期但业务仍在执行。此时需要一个**兜底校验**：写数据库时带上版本号或前置条件，让"过期后重复执行"变成幂等操作，而不是直接覆盖。

```go
// 乐观兜底：只有状态还是"待处理"才更新
res := db.Exec("UPDATE orders SET status='done' WHERE id=? AND status='pending'", id)
if res.RowsAffected == 0 {
    return nil // 已被别人处理过，直接返回成功
}
```

⚠ **关键认知**：分布式锁**不能保证业务绝对互斥**，它只是"降低并发冲突概率"。真正的正确性要靠数据库的唯一约束、版本号、状态机来兜底。

# 6. 可重入锁

## 6.1 同一个线程重复加锁的问题

如果一段代码里两次调用 `Lock`，第二次会失败，因为 key 已经存在。可重入锁要记录"当前持有者加了几次锁"：

```text
lock:order:1001
┌──────────────────────────────┐
│ field: token                 │  ← 持有者身份
│ field: count                 │  ← 重入次数
│ expire: 30s                  │
└──────────────────────────────┘
```

用 Hash 存 token 和 count：

```lua
-- 加锁：同一 token 重入则计数 +1，否则新建
if redis.call("EXISTS", KEYS[1]) == 0 then
    redis.call("HSET", KEYS[1], "token", ARGV[1], "count", 1)
    redis.call("PEXPIRE", KEYS[1], ARGV[2])
    return 1
end
if redis.call("HGET", KEYS[1], "token") == ARGV[1] then
    redis.call("HINCRBY", KEYS[1], "count", 1)
    redis.call("PEXPIRE", KEYS[1], ARGV[2])
    return 1
end
return 0
```

```lua
-- 解锁：计数减到 0 才真正删除
if redis.call("HGET", KEYS[1], "token") == ARGV[1] then
    local count = redis.call("HINCRBY", KEYS[1], "count", -1)
    if count == 0 then
        redis.call("DEL", KEYS[1])
        return 1
    end
    return 1
end
return 0
```

★ **关键认知**：Redis 里没有"线程"概念，"可重入"只能靠**调用方传入同一个 token**来识别。所以 Go 里要保证同一个业务链路复用同一个 `RedisLock` 实例，而不是每次 `NewLock` 都生成新 token。

⚠ **关键认知**：可重入锁增加了一层 Hash 操作和计数逻辑，也增加了死锁风险（忘记成对解锁会导致计数永远不归零）。除非业务确实需要嵌套加锁，否则优先改造代码避免重入。

# 7. Redlock 与它的争议

## 7.1 Redlock 是什么

单机 Redis 锁有一个根本问题：**主从异步复制**。A 在主节点加锁成功，主节点还没把锁同步给从节点就宕机了，哨兵把从节点提升为主，B 在新主节点上加锁也成功——两个客户端同时持锁。

Redlock 的方案是用 **5 个独立的 Redis 主节点**（不是主从关系），客户端依次加锁，只有在**多数派（≥3 个）**成功且总耗时小于锁有效期时才认为加锁成功。

```text
        客户端
          │
   ┌──────┼──────┬──────┬──────┐
   ▼      ▼      ▼      ▼      ▼
  R1     R2     R3     R4     R5
  ✓      ✓      ✓      ✗      ✗
          │
          ▼
   3/5 成功 → 加锁成功
```

## 7.2 争议在哪

分布式系统研究者 Martin Kleppmann 提出过著名质疑：

| 质疑点 | 说明 |
|--------|------|
| 时钟依赖 | 续期和过期依赖各节点时钟，时钟漂移会破坏安全性 |
| GC 停顿 | 客户端拿到锁后发生长时间 GC，醒来时锁已过期 |
| 无法保证线性一致 | 网络分区时，多数派和少数派可能各自认为操作合法 |

Redis 作者 antirez 的回应是：Redlock 适用于"效率型锁"（避免重复劳动），不适用于"正确性型锁"（必须互斥的资源）。

★ **关键认知**：**如果你的锁用来保证正确性（如扣款、库存），不要只依赖 Redis 锁**。用数据库行锁、唯一约束、状态机做最终防线。Redis 锁（无论单机还是 Redlock）更适合"避免重复计算""减少并发压力"这类效率场景。

## 7.3 选型建议

| 场景 | 推荐方案 |
|------|----------|
| 单实例、可接受极小概率失效 | SET NX EX + Lua 释放 |
| 多实例、效率型锁 | 单机 Redis 锁 + 业务兜底 |
| 强一致、金额相关 | 数据库锁 / 事务 / 唯一约束 |
| 已有 ZooKeeper/etcd | 用它们的临时顺序节点 |
| 必须用 Redis 且要高可用 | Redlock，但配合业务幂等 |

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 锁永久不释放 | `SETNX` 后进程崩溃，没设过期 | 用 `SET NX EX` 原子命令 |
| 误删别人的锁 | 释放时没校验唯一值 | Lua 比较 token 后再删 |
| 两个客户端同时持锁 | 业务超时 > 锁 TTL | TTL 设为业务超时的 2-3 倍 |
| 锁刚加就没了 | TTL 单位写错（秒/毫秒） | 确认 `EX` 与 `PX` |
| 续期线程泄漏 | 解锁后没停 ticker | `defer cancel()` |
| 集群下锁行为异常 | 单机客户端连集群、键落在不同槽 | 用 Cluster 客户端并固定 key |

⚠ **关键认知**：分布式锁的 key 必须落在**同一个槽位**。Cluster 模式下如果用多 key 脚本，所有 key 要用 hash tag（如 `lock:{order}:1001`）保证同槽，否则会报 `CROSSSLOT`。

# 9. 动手练习

1. 用 `SET NX EX` + Lua 释放实现一个锁，起两个 goroutine 竞争，验证同一时刻只有一个能进入临界区。
2. 把锁 TTL 设为 1 秒、业务 sleep 3 秒，观察锁过期后另一个客户端能否拿到锁，并解释这意味着什么。
3. 写一个续期 goroutine，让业务执行 10 秒而 TTL 只有 3 秒，验证续期能避免锁提前失效。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 加锁 | `SET key token NX EX ttl`，一条命令原子完成 |
| 唯一值 | 每次加锁生成新 token，防止误删 |
| 释放 | Lua 比较 token 后再 DEL |
| 续期 | 每 TTL/3 检查并续命，锁丢失就停 |
| 超时 | 锁 TTL > 业务最大耗时，业务本身要有超时 |
| Redlock | 多节点多数派，但仍有争议，正确性靠数据库兜底 |
| 可重入 | Hash 存 token + count，需显式复用同一实例 |

下一篇进入 Pipeline、事务、Lua，讲清楚"批量发命令如何减少往返""MULTI/EXEC 的原子性边界在哪""Lua 脚本为什么能保证原子"这些容易混淆的问题。