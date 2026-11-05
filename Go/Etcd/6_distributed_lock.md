用 etcd 做分布式锁的核心不是"抢一个 key"，而是"用租约证明客户端还活着，用 revision 排队保证公平，用事务保证比较和写入不可分割"。写对了，它能提供比 Redis 锁更强的一致性；写错了，同样会出现锁过期、误删、惊群和脑裂下的双持锁。本篇从最小实现讲到生产用法。

# 1. 为什么用 etcd 做锁

## 1.1 锁的基本要求

| 要求 | 含义 | etcd 的对应机制 |
|------|------|-----------------|
| 互斥 | 同一时刻只有一个持有者 | Txn 的 CAS 写入 |
| 防死锁 | 持有者崩溃后锁能释放 | Lease TTL 自动过期 |
| 防误删 | 只能释放自己的锁 | 唯一 token + Compare |
| 公平 | 等待者按顺序获得锁 | 前缀 key + CreateRevision 排序 |
| 一致性 | 故障切换后不出现双持锁 | Raft 多数派 |

## 1.2 etcd 的优势

etcd 的锁基于 Raft，写锁和释放锁都要多数派确认，因此不会出现 Redis 主从异步复制导致的那种"主节点挂了，锁还没同步，新主上又被别人加锁"的问题。代价是性能低一个数量级，适合**正确性优先、竞争不极端激烈**的场景。

★ **关键认知**：etcd 锁解决的是"多个客户端协调"的问题，不是"高吞吐抢购"的问题。如果你的锁每秒要被抢几万次，etcd 会成为瓶颈；这种场景应该用数据库行锁、Redis 锁加业务兜底，或者直接消除共享状态。

# 2. 最小实现：Lease + Txn

## 2.1 加锁

单 key 版本的思路：固定一个 key，谁能在 key 不存在时写入成功，谁就持锁。

```go
type EtcdLock struct {
    cli      *clientv3.Client
    key      string
    token    string
    ttl      int64
    leaseID  clientv3.LeaseID
    keepStop context.CancelFunc
}

func NewEtcdLock(cli *clientv3.Client, key string, ttl int64) *EtcdLock {
    // token 每个锁对象唯一，用于释放时校验身份
    return &EtcdLock{cli: cli, key: key, ttl: ttl, token: uuid.NewString()}
}

func (l *EtcdLock) Lock(ctx context.Context) (bool, error) {
    lease, err := l.cli.Grant(ctx, l.ttl) // 锁的寿命由租约保证
    if err != nil {
        return false, err
    }
    // 只有 key 不存在时才写入，且写入时绑定租约
    resp, err := l.cli.Txn(ctx).
        If(clientv3.Compare(clientv3.CreateRevision(l.key), "=", 0)).
        Then(clientv3.OpPut(l.key, l.token, clientv3.WithLease(lease.ID))).
        Commit()
    if err != nil || !resp.Succeeded {
        _, _ = l.cli.Revoke(ctx, lease.ID) // 加锁失败，及时释放租约
        return false, err
    }
    l.leaseID = lease.ID
    return true, nil
}
```

这里有两个关键点：

- `CreateRevision(key) = 0` 表示 key 从未创建，是 etcd 里判断"不存在"的标准写法。
- `Put` 与 `Compare` 在同一个 Txn 里，不会出现"判断时不存在、写入时已被别人占了"的竞态。

## 2.2 续期

租约不会自动续命，必须持续 KeepAlive：

```go
func (l *EtcdLock) startKeepAlive() {
    ctx, cancel := context.WithCancel(context.Background())
    l.keepStop = cancel
    ch, err := l.cli.KeepAlive(ctx, l.leaseID)
    if err != nil {
        cancel()
        return
    }
    go func() {
        for ka := range ch {
            if ka == nil { // 租约失效：锁已经不属于自己，业务应尽快停止
                log.Printf("锁 %s 的租约已失效", l.key)
                return
            }
        }
    }()
}
```

`KeepAlive` 默认会按 TTL 的三分之一左右发送续期。只要进程活着、网络通、etcd 有 Leader，锁就会一直保持。

## 2.3 释放

释放必须"先确认 key 还是自己的，再删除"：

```go
func (l *EtcdLock) Unlock(ctx context.Context) error {
    // 只有值等于自己的 token 才删除，避免锁过期后误删别人的锁
    _, err := l.cli.Txn(ctx).
        If(clientv3.Compare(clientv3.Value(l.key), "=", l.token)).
        Then(clientv3.OpDelete(l.key)).
        Commit()
    if err != nil {
        return err
    }
    if l.keepStop != nil {
        l.keepStop() // 停止续期
    }
    if l.leaseID != 0 {
        _, _ = l.cli.Revoke(context.Background(), l.leaseID) // 撤销租约，立即回收
    }
    return nil
}
```

⚠ **关键认知**：释放时只做 `Delete(key)` 是不够的。锁可能已经因为 TTL 过期而被别人重新持有，此时直接 Delete 会删掉别人的锁。必须用 token 做 Compare，确认"锁还是我加的那把"。

## 2.4 自动释放的边界

正常退出时 `Unlock` 删除 key 并撤销租约；进程被 kill 或网络分区时，KeepAlive 停止，TTL 到期后 key 自动删除；etcd 整体不可用时，客户端无法确认，锁状态由集群恢复后的日志决定。

★ **关键认知**：租约过期删除是**最终发生**的，不是精确到秒。TTL 到期后，Leader 还要把"撤销租约"走一次 Raft，期间客户端可能仍以为自己持锁。因此 TTL 必须明显大于业务最长执行时间，并且业务侧要有 fencing 兜底。

# 3. 公平锁：前缀 + revision 队列

## 3.1 单 key 为什么不公平

单 key 版本中，所有等待者都在抢同一个 key。锁释放的瞬间，所有等待者同时发起 Txn，只有一个成功，其他全部失败并重试。这会造成：

- **惊群**：N 个客户端同时打 etcd。
- **饥饿**：某个客户端可能连续多次抢不到。
- **无顺序**：先来的不一定先拿到。

## 3.2 排队算法

etcd 官方的公平锁做法是给每个等待者创建自己的 key，按创建 revision 排队：每个客户端用自己的租约创建唯一 key，按 `CreateRevision` 升序排列，排第 1 的持锁，排第 N 的只 Watch 排第 N-1 的 key，前一个被删除时才唤醒后一个。这样释放锁只会唤醒一个等待者，避免惊群；CreateRevision 全局递增，天然保证先到先得。

## 3.3 用 concurrency 包

官方已经封装好了，生产上优先用它：

```go
import "go.etcd.io/etcd/client/v3/concurrency"

func doWithLock(cli *clientv3.Client, lockName string) error {
    // 一个 Session 对应一个租约，内部自动 KeepAlive
    session, err := concurrency.NewSession(cli, concurrency.WithTTL(10))
    if err != nil {
        return err
    }
    defer session.Close() // 停止续期并撤销租约，锁自动释放

    mu := concurrency.NewMutex(session, lockName)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := mu.Lock(ctx); err != nil {
        return fmt.Errorf("获取锁失败: %w", err)
    }
    defer func() {
        if err := mu.Unlock(context.Background()); err != nil {
            log.Printf("释放锁失败: %v", err)
        }
    }()
    return doBusiness()
}
```

`concurrency.Mutex` 的关键行为：

| 方法 | 行为 |
|------|------|
| `Lock(ctx)` | 阻塞直到拿到锁，或 ctx 取消 |
| `TryLock(ctx)` | 非阻塞，锁被占用返回 `concurrency.ErrLocked` |
| `Unlock(ctx)` | 删除自己的排队 key，唤醒下一个 |
| `IsOwner()` | 返回当前持有的租约 ID，0 表示不持有 |

```go
if err := mu.TryLock(ctx); err != nil {
    if errors.Is(err, concurrency.ErrLocked) {
        return errors.New("锁已被占用，稍后再试")
    }
    return err
}
```

★ **关键认知**：`concurrency.Mutex` 不是可重入锁。同一个 Session 对同一个 mutex 再次 `Lock`，会创建第二个排队 key，然后等自己释放，形成死锁。需要嵌套加锁时，改造代码或换用可重入设计，不要直接重入。

# 4. 续期与会话失效

## 4.1 Session 是什么

`concurrency.Session` 把一个租约和一组锁绑定在一起：Session 失效时，租约撤销，所有排队 key 删除，所有锁一起释放。这也是为什么**不要把一个 Session 用在无关的业务上**：一个 Session 失效，它持有的所有锁会同时丢失。反过来，如果多个锁本来就属于同一个业务生命周期，共享 Session 是合理的。

## 4.2 续期失败要立刻停手

```go
select {
case <-session.Done():
    // 会话已结束：租约失效，锁不再属于自己
    log.Println("session 已失效，停止当前工作")
    return errors.New("锁已丢失")
case <-workDone:
    // 正常完成
}
```

`Session.Done()` 在以下情况关闭：

- KeepAlive 连续失败，客户端放弃续期。
- 租约被服务端撤销或过期。
- 主动调用 `session.Close()`。

⚠ **关键认知**：`Done()` 关闭不等于"业务线程立刻知道"。如果进程被长时间 GC 暂停或网络分区，客户端可能还没意识到锁已丢失，业务却已经继续执行。所以不能只靠"检查 Done"来保证安全，关键资源还要有 fencing。

## 4.3 fencing token

假设客户端 A 持锁后发生长时间停顿，租约过期，客户端 B 拿到锁开始写。A 恢复后仍以为自己持锁，继续写，两个客户端就同时修改了资源。

解决办法是给每次加锁分配一个单调递增的 fencing token，资源端拒绝旧 token 的写入：

```text
A 持锁，fencing token = 101
A 停顿，租约过期
B 持锁，fencing token = 102
B 写入资源，资源记录 last_token = 102
A 恢复，带着 token=101 写入
资源端检查 101 <= 102，拒绝
```

etcd 里可以用锁 key 的 `CreateRevision` 作为 fencing token：每次重新加锁，revision 一定比上一次大。

```go
resp, err := cli.Get(ctx, lockKey)
if err != nil {
    return err
}
fencingToken := resp.Kvs[0].CreateRevision // 单调递增，传给下游存储
```

★ **关键认知**：**分布式锁本身不能保证资源的绝对互斥**，它只能降低冲突概率。真正的正确性要靠资源端的 fencing、版本号、唯一约束或状态机。任何"拿到锁就绝对安全"的假设，在进程停顿和网络分区面前都不成立。

# 5. 与 Redis 锁对比

| 维度 | etcd 锁 | Redis 锁 |
|------|---------|----------|
| 一致性基础 | Raft 多数派，线性一致 | 主从异步复制，故障切换可能丢锁 |
| 过期机制 | Lease，服务端统一管理 | TTL，依赖各节点时间 |
| 释放安全 | Txn Compare token 后删除 | Lua Compare token 后删除 |
| 公平性 | 前缀 + CreateRevision 排队 | 无内置公平队列 |
| 惊群 | 只唤醒前一个等待者 | 常见，需要额外设计 |
| 性能 | 每次加锁/释放都要 Raft | 单命令，延迟低、吞吐高 |
| 可重入 | 默认不支持 | 可用 Hash 自行实现 |
| 运维 | 需要奇数节点、磁盘、quorum | 相对简单 |
| 适用 | 正确性优先、竞争不激烈 | 效率优先、竞争激烈 |

★ **关键认知**：etcd 锁和 Redis 锁不是"谁替代谁"。etcd 适合选主、任务调度、配置变更这类**低频但要求正确**的协调；Redis 适合防重复提交、缓存重建这类**高频且能容忍极小概率失效**的场景。两者都需要业务侧幂等兜底。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 锁永远不释放 | 没启动 KeepAlive，或 TTL 设得太大 | 用 Session 自动续期；TTL 与业务匹配 |
| 误删别人的锁 | 释放时没 Compare token | 用 Value 比较后再 Delete |
| 两个客户端同时持锁 | 业务执行时间超过 TTL，且无 fencing | TTL 放大 + 资源端 fencing |
| 等待者全部被唤醒 | 用了单 key 轮询，没有排队 | 用前缀 + revision 或 concurrency 包 |
| `ErrLocked` 频繁 | 竞争激烈 | 退避重试，或改用队列削峰 |
| goroutine 泄漏 | 每次加锁都新建 Session/KeepAlive | Session 复用，Close 时停止 |
| 锁 key 不释放 | Unlock 没放在 defer 里 | `defer mu.Unlock(...)` |
| 重入死锁 | 同一 mutex 连续 Lock | 避免重入，或换可重入设计 |
| 释放时报 not found | 锁已过期，属于正常现象 | 记录日志，不要当致命错误 |

⚠ **关键认知**：TTL 不是越长越安全。TTL 太长，进程崩溃后别人要等很久；TTL 太短，业务没跑完锁就丢了。推荐 TTL 设为业务最大耗时的 2-3 倍，并给业务本身设置超时，让"持锁时间"有明确上界。

# 7. 动手练习

1. 用 `EtcdLock` 起 10 个 goroutine 竞争同一把锁，每个持锁 1 秒，统计总耗时，验证互斥。
2. 把 TTL 设为 2 秒、业务 sleep 5 秒，观察锁过期后另一个客户端能否拿到锁，并解释为什么必须加 fencing。
3. 用 `concurrency.NewMutex` 实现公平锁，记录 5 个等待者的获得顺序，验证与创建 revision 的顺序一致。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 互斥 | Txn + `CreateRevision=0` 抢 key |
| 自动释放 | Put 绑定 Lease，停止续期后自动删除 |
| 防误删 | 唯一 token，释放前 Compare |
| 续期 | Session 内部 KeepAlive，TTL/3 左右 |
| 公平 | 前缀 key + CreateRevision 排序，只唤醒前一个 |
| 会话失效 | `Done()` 关闭后立即停手，但别只依赖它 |
| fencing | 用锁 key 的 CreateRevision 做单调 token |
| 选型 | 正确性优先用 etcd，吞吐优先用 Redis，都要幂等 |

下一篇进入服务发现与配置中心，讲清楚如何用 Lease 注册实例、用前缀查询和 Watch 发现变更、如何设计配置版本与灰度发布，以及大规模 Watch 下的性能与一致性注意事项。
