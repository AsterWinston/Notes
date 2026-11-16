Lease 和 Watch 是 etcd 从"键值存储"变成"分布式协调组件"的两块基石：Lease 让 key 能随会话自动消失，Watch 让客户端能实时感知变化。两者组合起来，就是服务注册、配置热更新、分布式锁的标准骨架。本篇把租约的生命周期、KeepAlive 的正确用法、Watch 的事件结构与断线恢复讲透。

# 1. Lease 租约

## 1.1 租约解决什么问题

普通 key 写入后会一直存在，除非显式删除。服务注册场景需要一个"进程活着 key 就在，进程挂了 key 自动消失"的语义，这就是租约：客户端先 `Grant(ttl)` 拿到租约，再用 `WithLease` 把 key 绑定上去，并持续 `KeepAlive` 续期；停止续期后，TTL 到期会删除该租约下的所有 key。

★ **关键认知**：租约是**一组 key 的公共生命周期**。一个租约可以绑定任意多个 key，租约过期或被撤销时，这些 key 会一起被删除。不要把租约当成"单个 key 的过期时间"，它是会话级的。

## 1.2 Grant、绑定与撤销

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

lease, err := cli.Grant(ctx, 30) // 申请一个 30 秒的租约
if err != nil {
    return err
}
_, err = cli.Put(ctx, "/services/order/10.0.0.1:8080", `{"weight":10}`,
    clientv3.WithLease(lease.ID)) // 到期后 key 自动删除
if err != nil {
    return err
}
_, err = cli.Revoke(ctx, lease.ID) // 主动撤销，立刻删除其下所有 key
```

常用操作：`Grant(ctx, ttl)` 申请（TTL 单位秒）、`Put(..., WithLease(id))` 绑定（一个租约可绑多个 key）、`KeepAlive`/`KeepAliveOnce` 续期、`TimeToLive` 查剩余 TTL、`Revoke` 立即删除、`Leases` 列出所有租约。

TTL 不是精确闹钟。过期检查由 Leader 周期性执行，实际删除可能比 TTL 晚一点；Leader 切换、网络抖动也会影响检查节奏。因此**不要依赖"第 30 秒整一定消失"**，只能依赖"停止续期后最终会消失"。

## 1.3 KeepAlive 续期

```go
// KeepAlive 会启动后台 goroutine，自动按 TTL/3 左右发送续期请求
keepCtx, keepCancel := context.WithCancel(context.Background())
defer keepCancel()
ch, err := cli.KeepAlive(keepCtx, lease.ID)
if err != nil {
    return err
}
go func() {
    for ka := range ch {
        if ka == nil { // 通道关闭：租约已失效或续期被停止
            log.Println("租约已失效，需要重新注册")
            return
        }
        log.Printf("续期成功: lease=%x ttl=%d", ka.ID, ka.TTL)
    }
}()
```

`KeepAlive` 返回一个只读通道，每次续期成功会收到新的 TTL；通道关闭通常表示租约已经过期或被撤销。如果只想手动续一次，用 `cli.KeepAliveOnce(ctx, lease.ID)`，适合短任务或测试；生产环境的服务注册一般还是用长期通道，避免自己写定时器。

★ **关键认知**：**同一个租约只开一个 KeepAlive 通道**。每次调用 `KeepAlive` 都会新建一个后台 goroutine 和流；在请求处理函数里反复调用，会迅速积累 goroutine 和连接。正确模式是进程启动时注册一次，把通道交给一个专门的后台协程消费。

## 1.4 租约过期后会发生什么

租约 123 绑定了 key a 和 key b；TTL 到期未续期时，etcd 删除租约 123，两个 key 一起被删除，Watch 会收到两条 DELETE 事件。几个容易忽略的点：

- 过期删除会产生 Watch 的 `DELETE` 事件，`Kv` 里通常只有 key 和 revision；需要旧值就加 `WithPrevKV()`。
- 用已过期租约执行 `Put` 会返回 `etcdserver: requested lease not found`。
- 租约 ID 只保证在当前存活期间有效，不要把它写进业务数据当永久标识。
- 撤销租约是幂等的：重复 `Revoke` 通常返回成功或 not found，按客户端版本略有差异。

# 2. Watch

## 2.1 最小 Watch

```go
import mvccpb "go.etcd.io/etcd/api/v3/mvccpb"

watchCtx, watchCancel := context.WithCancel(context.Background())
defer watchCancel()
rch := cli.Watch(watchCtx, "/services/", clientv3.WithPrefix(), clientv3.WithPrevKV())

for wresp := range rch {
    if wresp.Canceled { // 上下文结束、权限变化、revision 被压缩等
        log.Printf("watch 取消: %v, compactRev=%d", wresp.Err(), wresp.CompactRevision)
        return
    }
    for _, ev := range wresp.Events {
        switch ev.Type {
        case mvccpb.PUT:
            fmt.Printf("新增/修改: %s = %s\n", ev.Kv.Key, ev.Kv.Value)
        case mvccpb.DELETE:
            fmt.Printf("删除: %s，旧值=%s\n", ev.Kv.Key, ev.PrevKv.GetValue())
        }
    }
}
```

没有事件时，`for range rch` 会阻塞；这正是 Watch 比轮询省资源的地方。不要在循环里做耗时操作，否则事件会积压。

## 2.2 事件结构

| 字段 | 含义 |
|------|------|
| `ev.Type` | `mvccpb.PUT` 或 `mvccpb.DELETE` |
| `ev.Kv.Key` | 发生变化的 key |
| `ev.Kv.Value` | PUT 时的新值；DELETE 时通常为空 |
| `ev.Kv.ModRevision` | 本次事件的 revision |
| `ev.Kv.CreateRevision` | key 创建时的 revision |
| `ev.PrevKv` | 变化前的值，只有 `WithPrevKV()` 才有 |
| `wresp.Header.Revision` | 该批事件对应的全局 revision |
| `wresp.CompactRevision` | 非 0 表示因压缩被取消 |

同一次事务修改多个 key，会作为同一批事件推送，`Header.Revision` 相同。事件在单个 Watch 内是**有序**的，但不要假设不同 Watch 之间的到达顺序一致。

## 2.3 常用 Watch 选项

常用选项有 `WithPrefix()`（监听前缀）、`WithRev(rev)`（从指定 revision 回放）、`WithPrevKV()`（DELETE 带旧值）、`WithProgressNotify()`（定期收到只有 Header 的进度通知）、`WithFilterPut()`/`WithFilterDelete()`（过滤事件类型）；另外可以用 `clientv3.WithRequireLeader(ctx)` 包装 context，让集群没有 Leader 时 Watch 失败而不是静默等待。`WithProgressNotify` 适合做"连接还活着吗"的检测，即使没有业务事件，也能借进度通知推进本地记录的 revision。

## 2.4 监听取消

Watch 的取消靠 `context`：

```go
ctx, cancel := context.WithCancel(context.Background())
rch := cli.Watch(ctx, "/config/", clientv3.WithPrefix())
go func() { time.Sleep(10 * time.Second); cancel() }() // 10 秒后关闭
for resp := range rch {
    if resp.Canceled {
        break // 上下文取消后通道会关闭
    }
}
```

⚠ **关键认知**：`cancel()` 之后要继续把通道读到关闭，否则客户端内部的发送协程可能阻塞。最简单的写法是 `for range rch {}` 直到通道关闭，再退出 goroutine。不要只调用 `cancel()` 就丢弃通道。

# 3. Revision、压缩与断线重连

## 3.1 先列后听：不要漏事件

错误写法是 Get 后直接 `Watch(ctx, prefix, WithPrefix())`，会漏掉两者之间发生的写入。正确写法是从读到的 revision 的下一个开始：

```go
resp, err := cli.Get(ctx, "/services/", clientv3.WithPrefix())
if err != nil {
    return err
}
rch := cli.Watch(ctx, "/services/",
    clientv3.WithPrefix(),
    clientv3.WithRev(resp.Header.Revision+1), // 覆盖 Get 之后、Watch 建立之前的事件
)
```

★ **关键认知**：`WithRev(0)` 表示从 Watch 建立时的当前 revision 开始，**不包含**建立之前的事件。标准恢复流程必须是"Get 拿当前 revision → 用 revision + 1 建立 Watch → 先处理 Watch 缓冲，再决定是否用 Get 结果初始化"。顺序反了就会丢事件。

## 3.2 压缩导致的失败

如果 Watch 的起点已经被 compact：

```text
最后处理到 rev=100
服务端已 compact 到 rev=500
Watch(rev=101) → Canceled=true, CompactRevision=500
```

处理方式是：重新全量拉取并重建本地状态，再从新的 `revision + 1` 重新 Watch：

```go
if wresp.Canceled && wresp.CompactRevision > 0 {
    resp, err := cli.Get(ctx, prefix, clientv3.WithPrefix())
    if err != nil {
        return err
    }
    rebuild(resp.Kvs) // 用全量结果重建本地状态
    rch = cli.Watch(ctx, prefix, clientv3.WithPrefix(),
        clientv3.WithRev(resp.Header.Revision+1))
    continue
}
```

生产上建议把自动压缩保留时间设得比"最长断线时间"长，例如保留 1 小时，避免短暂网络抖动就触发全量重建。

## 3.3 断线重连

`clientv3` 会自动重连，并在内部重新发送 Watch 请求，通常从上次收到的 revision 继续。应用层不需要自己重建 Watch，但必须处理两件事：**重复事件**（重连后可能重放少量已处理事件，逻辑要幂等）和**压缩失败**（断线超过历史保留窗口，Watch 以 `Canceled` 结束，必须全量重建）。

★ **关键认知**：Watch 的语义更接近"至少一次"而不是"恰好一次"。把事件处理写成幂等的 upsert/delete（按 key 覆盖本地状态），比追求"绝不重复"更可靠。

# 4. 组合模式

## 4.1 服务注册与自动下线

```go
type Registration struct {
    cli     *clientv3.Client
    leaseID clientv3.LeaseID
}

func Register(cli *clientv3.Client, service, addr, meta string, ttl int64) (*Registration, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    lease, err := cli.Grant(ctx, ttl)
    if err != nil {
        return nil, err
    }
    key := "/services/" + service + "/" + addr
    if _, err := cli.Put(ctx, key, meta, clientv3.WithLease(lease.ID)); err != nil {
        return nil, err
    }

    // 后台续期；进程退出时 cancel 停止续期，租约到期后 key 自动删除
    keepCtx, keepCancel := context.WithCancel(context.Background())
    ch, err := cli.KeepAlive(keepCtx, lease.ID)
    if err != nil {
        keepCancel()
        return nil, err
    }
    go func() {
        defer keepCancel()
        for ka := range ch {
            if ka == nil {
                log.Printf("服务 %s 的租约已失效", key)
                return
            }
        }
    }()
    return &Registration{cli: cli, leaseID: lease.ID}, nil
}

func (r *Registration) Close() error {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    _, err := r.cli.Revoke(ctx, r.leaseID) // 主动下线，立即删除 key
    return err
}
```

消费者侧用 `Watch` 监听 `/services/order/` 前缀：收到 PUT 就把实例加入本地列表，收到 DELETE 就移除。这样服务上线、下线、崩溃都能在秒级被发现。

## 4.2 配置热更新

```go
func watchConfig(cli *clientv3.Client, prefix string, apply func(key, value string)) error {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    resp, err := cli.Get(ctx, prefix, clientv3.WithPrefix())
    if err != nil {
        return err
    }
    for _, kv := range resp.Kvs {
        apply(string(kv.Key), string(kv.Value))
    }
    rch := cli.Watch(ctx, prefix, clientv3.WithPrefix(),
        clientv3.WithRev(resp.Header.Revision+1))
    for wresp := range rch {
        if wresp.Canceled {
            return fmt.Errorf("watch 取消: %v", wresp.Err())
        }
        for _, ev := range wresp.Events {
            if ev.Type == mvccpb.DELETE {
                apply(string(ev.Kv.Key), "")
            } else {
                apply(string(ev.Kv.Key), string(ev.Kv.Value))
            }
        }
    }
    return nil
}
```

配置项建议拆成多个 key，一次事务改多个 key，消费者按批应用，避免读到"改了一半"的配置。

# 5. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| key 到期不消失 | 没有 KeepAlive，或 KeepAlive 的 ctx 被提前取消 | 检查通道是否关闭，进程是否还在续期 |
| goroutine 越来越多 | 每个请求都调用 `KeepAlive` | 每个租约只开一个续期协程 |
| Watch 收不到事件 | 前缀不匹配，或 `WithRev` 从旧 revision 开始但已压缩 | 检查前缀边界与 `CompactRevision` |
| Watch 突然结束 | ctx 被取消、权限变化、revision 被压缩 | 读 `wresp.Err()` 与 `CompactRevision`，走重建 |
| 事件处理重复 | 断线重连重放 | 处理逻辑做成幂等 upsert/delete |
| 内存持续上涨 | Watch 通道消费太慢，事件积压 | 循环里只做入队，重活交给 worker |

⚠ **关键认知**：Watch 循环里不要做网络调用、数据库写入等耗时操作。etcd 客户端会缓冲一部分事件，但消费速度长期低于产生速度时，内存会持续增长，最终 OOM。正确做法是"Watch 只负责收事件并放入有界队列，后台 worker 异步处理"。

# 6. 动手练习

1. 申请一个 5 秒租约，绑定一个 key，不做续期，用 Watch 观察它过期时收到的 DELETE 事件，打印 `CompactRevision` 与 `ModRevision`。
2. 启动一个 Watch，处理 10 秒后调用 `cancel()`，验证通道关闭、goroutine 正常退出，没有泄漏。
3. 模拟"先列后听"：先 Get 前缀，再在 1 秒后写入一个新 key，确认用 `Revision+1` 能收到事件，而用 `WithRev(0)` 会漏掉。

# 7. 小结

| 主题 | 要点 |
|------|------|
| Lease | 一组 key 的公共 TTL，过期或撤销时一起删除 |
| KeepAlive | 每个租约一个后台通道，TTL/3 左右续期一次 |
| 过期 | 不精确，最终删除；不要依赖秒级边界 |
| Watch | 按 key/前缀订阅，事件有序，PUT/DELETE 两类 |
| 先列后听 | Get 的 revision + 1 作为 Watch 起点 |
| 压缩 | 旧 revision 不可回放，必须全量重建 |

下一篇进入事务，讲清楚 Txn 的 If/Then/Else 结构、Compare 的五种条件、CAS 与乐观并发、事务内多操作的原子性边界，以及最常见的误用方式。
