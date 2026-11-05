单条 Redis 命令再快，也架不住网络往返。一次 RTT 就算只有 0.5ms，100 条命令串行执行也要 50ms。更麻烦的是，有些"读-判断-写"的组合必须一起完成，中途不能被别的客户端插队。本篇讲三件常被混为一谈的东西：Pipeline（减少往返）、事务（MULTI/EXEC + WATCH）、Lua（服务端原子执行），并说清它们各自的原子性边界。

# 1. 三者的定位

先给结论，避免概念串味：

| 机制 | 解决什么 | 原子性 | 能否读中间结果 |
|------|----------|--------|----------------|
| Pipeline | 多条命令一次网络往返 | 否 | 客户端一次收全部结果 |
| 事务 MULTI/EXEC | 命令排队后一起执行 | 执行阶段不被插入 | 不能，结果最后一起返回 |
| WATCH | 乐观锁，检测 key 是否被改 | 配合事务使用 | 不能 |
| Lua 脚本 | 服务端执行逻辑 | 整个脚本原子执行 | 脚本内可以读 |

★ **关键认知**：Pipeline 只省网络往返，**不保证原子**；事务保证"执行阶段连续"，但**不保证失败回滚**；Lua 保证"整个脚本连续执行"，是目前实现复合原子操作的首选。三者不是替代关系，可以组合使用。

# 2. Pipeline

## 2.1 为什么快

```text
串行执行（100 条命令）：
客户端 ──命令1──▶ Redis ──结果1──▶ 客户端
客户端 ──命令2──▶ Redis ──结果2──▶ 客户端
... 100 次 RTT

Pipeline：
客户端 ──命令1,2,...,100──▶ Redis
客户端 ◀──结果1,2,...,100── Redis
... 1 次 RTT
```

```go
pipe := rdb.Pipeline()

// 这些调用只是把命令放进缓冲区，还没有发送
incr := pipe.Incr(ctx, "counter")
pipe.Expire(ctx, "counter", time.Hour)
get := pipe.Get(ctx, "greeting")

// Exec 才真正发送并等待所有结果
_, err := pipe.Exec(ctx)
if err != nil {
    // 注意：err 可能是某条命令的错误，也可能整体失败
    return err
}

fmt.Println(incr.Val()) // 结果在 Exec 之后才能读
fmt.Println(get.Val())
```

★ **关键认知**：Pipeline 中的命令**按顺序发送、按顺序返回**，但 Redis 是逐条执行的，中间可能插入其他客户端的命令。所以 Pipeline 适合"批量读""批量写"这类彼此独立的操作，**不适合**"先读再根据结果写"。

## 2.2 错误处理

Pipeline 的错误分两层：

```go
cmds, err := pipe.Exec(ctx)
if err != nil && !errors.Is(err, redis.Nil) {
    // 1. 整体错误：网络断、连接池超时等，所有命令都可能没执行
}

// 2. 单条命令错误：Exec 返回所有命令，逐条检查
for _, cmd := range cmds {
    if err := cmd.Err(); err != nil {
        // 这条命令失败了，其他命令可能成功
    }
}
```

⚠ **关键认知**：Pipeline **不是事务**。前 50 条执行成功、第 51 条失败、后面 49 条仍然会执行。想要"要么全成功"，得用事务或 Lua，而不是 Pipeline。

## 2.3 批量大小

不要一次塞 10 万条命令：客户端内存会涨，服务端输出缓冲区也可能超限（`client-output-buffer-limit`）。建议按 500-1000 条一批：

```go
const batchSize = 500

for i := 0; i < len(keys); i += batchSize {
    end := i + batchSize
    if end > len(keys) {
        end = len(keys)
    }
    pipe := rdb.Pipeline()
    for _, k := range keys[i:end] {
        pipe.Get(ctx, k)
    }
    if _, err := pipe.Exec(ctx); err != nil {
        return err
    }
}
```

# 3. 事务：MULTI / EXEC

## 3.1 基本用法

```go
// TxPipeline 内部会发送 MULTI ... EXEC
pipe := rdb.TxPipeline()

incr := pipe.Incr(ctx, "balance")
pipe.Decr(ctx, "stock")

_, err := pipe.Exec(ctx) // 发送 EXEC，命令一起执行
```

对应的 Redis 命令序列：

```text
MULTI          ← 开启事务，后续命令进入队列
INCR balance   ← 返回 QUEUED，不执行
DECR stock     ← 返回 QUEUED，不执行
EXEC           ← 按顺序执行队列里的所有命令，返回结果数组
```

★ **关键认知**：事务的原子性指"**EXEC 执行期间不会被其他客户端的命令插入**"，类似一段不可分割的批处理。它**不是**关系型数据库那种"失败自动回滚"——某条命令运行时出错，其他命令照样执行，Redis 不会撤销。

## 3.2 事务的两种错误

| 错误类型 | 发生时机 | 行为 |
|----------|----------|------|
| 入队错误 | 命令语法错、命令不存在 | `EXEC` 时整个事务被拒绝，一条都不执行 |
| 执行错误 | 类型不匹配（对 String 用 `LPUSH`） | 该条失败，其他命令继续执行 |

```text
入队错误示例：
MULTI
SET k v
INVALIDCMD        ← 语法错误，返回 error
EXEC              ← 返回 EXECABORT，所有命令都不执行
```

```text
执行错误示例：
MULTI
SET k v
LPUSH k x         ← k 是 String，运行时报 WRONGTYPE
EXEC              ← SET 成功，LPUSH 失败，事务不回滚
```

⚠ **关键认知**：Redis 事务不保证"全部成功或全部失败"。需要强原子语义时，用 Lua 脚本，或者把校验逻辑放到应用层用 WATCH 实现乐观锁。

## 3.3 事务中的 DISCARD

```go
pipe := rdb.TxPipeline()
pipe.Set(ctx, "k", "v", 0)
// 不想执行了：Discard 会发送 DISCARD 丢弃队列
pipe.Discard()
```

在 go-redis 里通常不需要手动 `Discard`，`Exec` 失败或连接关闭时会自动清理。

# 4. WATCH：乐观锁

## 4.1 机制

WATCH 监控一个或多个 key，如果在 `EXEC` 之前这些 key 被**其他客户端**修改过，事务就会被放弃（`EXEC` 返回 nil）。

```go
func IncrWithLimit(ctx context.Context, key string, limit int64) error {
    return rdb.Watch(ctx, func(tx *redis.Tx) error {
        // 1. 在 WATCH 之后读当前值
        val, err := tx.Get(ctx, key).Int64()
        if err != nil && err != redis.Nil {
            return err
        }
        if val >= limit {
            return errors.New("超过上限")
        }

        // 2. 开启事务并执行；期间 key 被改过则 Exec 返回 redis.TxFailedErr
        _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.Incr(ctx, key)
            return nil
        })
        return err
    }, key)
}
```

★ **关键认知**：WATCH 是**乐观锁**——它假设冲突很少，先读再写，提交时检查有没有人改过。冲突了就重试。这比"先加分布式锁再操作"轻量，但冲突频繁时会大量重试。

## 4.2 重试循环

```go
for i := 0; i < 3; i++ { // 最多重试 3 次
    err := IncrWithLimit(ctx, "counter", 100)
    if err == nil {
        return nil
    }
    if errors.Is(err, redis.TxFailedErr) {
        continue // key 被改了，重新读-改-写
    }
    return err // 其他错误直接返回
}
return errors.New("重试次数耗尽")
```

⚠ **关键认知**：WATCH 监控的 key 一旦被修改，整个事务作废。所以 WATCH 的粒度要小，只监控真正需要校验的 key；监控太多 key 会让冲突概率飙升，重试成本反而更高。

# 5. Lua 脚本

## 5.1 为什么 Lua 能保证原子

Redis 执行 Lua 脚本时，会把脚本当成一个整体，在**单线程**里连续执行完，期间不处理其他客户端命令。所以脚本内的"读 → 判断 → 写"不会被插队。

```lua
-- 原子地"仅当值等于旧值时才更新"
if redis.call("GET", KEYS[1]) == ARGV[1] then
    redis.call("SET", KEYS[1], ARGV[2])
    return 1
else
    return 0
end
```

```go
var casScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
    redis.call("SET", KEYS[1], ARGV[2])
    return 1
else
    return 0
end
`)

func CompareAndSet(ctx context.Context, key, oldVal, newVal string) (bool, error) {
    n, err := casScript.Run(ctx, rdb, []string{key}, oldVal, newVal).Int64()
    return n == 1, err
}
```

## 5.2 脚本缓存与 EVALSHA

`redis.NewScript` 第一次 `Run` 时会用 `EVALSHA`，如果服务端没缓存过这个脚本，会返回 `NOSCRIPT`，go-redis 自动降级用 `EVAL` 发送整个脚本并缓存。之后都走 `EVALSHA`，只传 SHA1，节省带宽。

```text
首次调用：
EVALSHA <sha1> ... → NOSCRIPT
EVAL <完整脚本> ... → 执行并缓存

后续调用：
EVALSHA <sha1> ... → 直接执行
```

★ **关键认知**：脚本缓存是按 **SHA1** 索引的，内容一变 SHA1 就变。所以不要在脚本里拼接动态内容（比如把 key 名写死进脚本文本），否则每次都是新脚本，缓存永远命中不了。

## 5.3 Lua 的边界

| 限制 | 说明 |
|------|------|
| 单线程阻塞 | 脚本执行期间 Redis 不处理其他命令，脚本要短 |
| 无回滚 | 脚本中途报错，已执行的命令不会撤销 |
| 不能跨槽 | Cluster 模式下所有 key 必须同槽（用 hash tag） |
| 随机性受限 | 早期版本要求脚本对同一输入结果确定 |
| 调试困难 | 没有断点，只能靠返回值和日志 |

```lua
-- 反例：循环百万次，阻塞整个 Redis
for i = 1, 1000000 do
    redis.call("INCR", KEYS[1])
end
```

⚠ **关键认知**：Lua 的"原子"是**执行不被打断**，不是"失败可回滚"。脚本里如果先 `SET` 再对一个错误类型的 key 执行 `LPUSH`，`SET` 已经生效且不会撤销。写脚本时要先校验参数，把可能失败的操作放在前面。

## 5.4 批量删除示例

```lua
-- 按前缀批量删除（注意：生产慎用，可能阻塞）
local keys = redis.call("KEYS", ARGV[1])
for i = 1, #keys do
    redis.call("DEL", keys[i])
end
return #keys
```

★ **关键认知**：`KEYS` 在大库上极慢，会阻塞整个实例。生产环境删除大量 key 应该用 `SCAN` 在客户端分批迭代，而不是在 Lua 里 `KEYS *`。

# 6. 如何选择

```text
需要减少网络往返？
   │
   ├── 命令彼此独立 ──→ Pipeline
   │
   └── 命令有依赖
         │
         ├── 需要读中间结果做判断 ──→ Lua
         │
         ├── 冲突少、可重试 ──→ WATCH + 事务
         │
         └── 只是批量执行、不要求全成功 ──→ TxPipeline
```

| 需求 | 推荐 | 注意 |
|------|------|------|
| 批量读 100 个 key | Pipeline | 分批，别一次太多 |
| 扣库存（读后写） | Lua | 脚本要短 |
| 乐观并发控制 | WATCH | 配重试循环 |
| 多条命令连续执行 | MULTI/EXEC | 不保证回滚 |
| 跨 key 的复合操作 | Lua + hash tag | 必须同槽 |

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| Pipeline 结果为空 | 没调用 `Exec` | 命令只是入队，必须 `Exec` |
| `EXECABORT` | 事务中有语法错误命令 | 检查命令拼写和参数 |
| 事务部分成功 | 执行阶段类型错误 | 用 Lua 或应用层校验 |
| `TxFailedErr` 频繁 | WATCH 的 key 冲突多 | 缩小监控范围或改用锁 |
| `NOSCRIPT` 反复出现 | 脚本内容动态拼接 | 固定脚本文本 |
| `CROSSSLOT` | Cluster 下多 key 不同槽 | 用 `{tag}` 保证同槽 |
| Redis 被脚本卡住 | 脚本里 `KEYS` 或大循环 | 拆分、改用 `SCAN` |

# 8. 动手练习

1. 用 Pipeline 批量写入 1000 个 key，对比逐条写入的耗时。
2. 用 WATCH 实现"仅当余额足够时扣款"，模拟两个并发请求，验证只有一个成功。
3. 用 Lua 实现"如果 key 存在则删除并返回 1，否则返回 0"，并解释为什么它比 `GET` + `DEL` 安全。

# 9. 小结

| 机制 | 原子性 | 适用场景 | 关键坑 |
|------|--------|----------|--------|
| Pipeline | 否 | 批量独立命令 | 必须 `Exec`，部分失败 |
| 事务 | 执行连续 | 多条命令批处理 | 不回滚 |
| WATCH | 乐观锁 | 读改写、冲突少 | 需要重试 |
| Lua | 整体连续 | 复合原子操作 | 不能太长、无回滚 |

下一篇进入消息通信，讲 Pub/Sub 与 Stream 的区别、消费者组、ACK、Pending 列表，以及如何用 Stream 搭一套可靠消息方案。