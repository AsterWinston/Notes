`clientv3` 的 API 看似只有 Put/Get/Delete 三个动作，但每个动作都带一组选项：前缀、数量限制、指定 revision、串行化读、返回旧值。选项用错，轻则读到旧数据，重则把整个前缀误删。本篇用一个可运行的小程序，把连接配置、常用读写、分页、revision 与错误处理串起来。

# 1. 连接 etcd

## 1.1 安装客户端

```bash
go mod init example.com/etcd-demo
go get go.etcd.io/etcd/client/v3@v3.5.15
```

`go.etcd.io/etcd/client/v3` 是官方维护的 v3 客户端。不要再用 `github.com/coreos/etcd/clientv3` 这类旧路径，模块已经迁移。

## 1.2 最小连接示例

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
    cli, err := clientv3.New(clientv3.Config{
        Endpoints:   []string{"127.0.0.1:2379"}, // 可写多个地址，客户端自动轮询
        DialTimeout: 5 * time.Second,            // 建立连接的超时
    })
    if err != nil {
        log.Fatalf("连接 etcd 失败: %v", err)
    }
    defer cli.Close() // 必须关闭，否则后台 goroutine 与连接会泄漏

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    if _, err := cli.Put(ctx, "/demo/hello", "world"); err != nil {
        log.Fatalf("写入失败: %v", err)
    }

    resp, err := cli.Get(ctx, "/demo/hello")
    if err != nil {
        log.Fatalf("读取失败: %v", err)
    }
    for _, kv := range resp.Kvs {
        fmt.Printf("%s = %s (mod revision=%d)\n", kv.Key, kv.Value, kv.ModRevision)
    }
}
```

★ **关键认知**：`go run main.go` 会输出 `/demo/hello = world (mod revision=2)`。`clientv3.New` 是**懒连接**，它只构造客户端，第一次 RPC 时才真正建立 gRPC 连接。因此"New 成功"不代表 etcd 可用，第一次 `Put`/`Get` 的错误才是真实连通性。生产上可以在启动时主动 `cli.Sync(ctx)` 或做一次 `Get` 探活。

## 1.3 生产常用配置

```go
cli, err := clientv3.New(clientv3.Config{
    Endpoints:            []string{"10.0.0.1:2379", "10.0.0.2:2379", "10.0.0.3:2379"},
    DialTimeout:          5 * time.Second,
    DialKeepAliveTime:    10 * time.Second, // gRPC 心跳间隔
    DialKeepAliveTimeout: 3 * time.Second,  // 心跳响应超时
    AutoSyncInterval:     30 * time.Second, // 定期从集群同步成员列表
    MaxCallSendMsgSize:   2 * 1024 * 1024,  // 最大发送消息，默认约 2 MB
    MaxCallRecvMsgSize:   4 * 1024 * 1024,  // 最大接收消息

    Username: "root",      // 开启认证后使用
    Password: "secret",
})
```

建议至少写 2-3 个 Endpoints 避免单点，`DialTimeout` 设 3-5 秒，`AutoSyncInterval` 设 30 秒左右；大范围查询时再调大 `MaxCallRecvMsgSize`，生产必须配置 `TLS`。

⚠ **关键认知**：不要为每次请求 `clientv3.New` 一个客户端。客户端内部维护连接池、Watch 流和重连逻辑，频繁创建会耗尽文件描述符，还会让 Watch 反复重建。正确做法是进程内单例，随进程关闭。

# 2. 写入：Put 与 Delete

## 2.1 Put

Put 是**整值覆盖**，不是追加。`resp.Header.Revision` 是本次写事务的全局 revision。value 是字节数组，存 JSON、protobuf、纯文本都可以，但建议控制在 KB 级。

带租约写入：

```go
lease, err := cli.Grant(ctx, 30) // 30 秒 TTL
if err != nil {
    return err
}
_, err = cli.Put(ctx, "/services/order/10.0.0.1:8080", "alive",
    clientv3.WithLease(lease.ID))
```

这样 key 会在租约到期且没有续期时自动删除，服务注册和分布式锁都依赖这个特性。

## 2.2 Delete

```go
// 删除单个 key
resp, err := cli.Delete(ctx, "/demo/hello")
// 删除整个前缀，并返回被删掉的旧值
resp, err = cli.Delete(ctx, "/services/order/",
    clientv3.WithPrefix(),
    clientv3.WithPrevKV(),
)
fmt.Println("删除数量:", resp.Deleted)
for _, kv := range resp.PrevKvs {
    fmt.Printf("被删除: %s = %s\n", kv.Key, kv.Value)
}
```

★ **关键认知**：`WithPrefix()` 的匹配是**字节前缀**，不是"目录"。`/service` 会匹配 `/service2/x`，`/service/` 只匹配带斜杠的子路径。前缀末尾是否带 `/` 必须统一，否则很容易误删或漏查。

## 2.3 选项速查

| 选项 | 适用操作 | 作用 |
|------|----------|------|
| `WithPrefix()` | Get/Delete | 按前缀匹配 |
| `WithRange(end)` | Get/Delete | 匹配 `[key, end)` 区间 |
| `WithLimit(n)` | Get | 最多返回 n 个 key |
| `WithRev(rev)` | Get | 读取指定 revision 的历史快照 |
| `WithSerializable()` | Get | 串行化读，允许陈旧 |
| `WithSort(...)` | Get | 按 key 或 revision 排序 |
| `WithCountOnly()` | Get | 只返回数量，不返回值 |
| `WithPrevKV()` | Put/Delete | 返回修改前的旧值 |
| `WithLease(id)` | Put | 绑定租约 |
| `WithIgnoreLease()` | Put | 保留 key 原有租约，只改值 |

# 3. 查询：Get 的常见形态

## 3.1 单 key 与前缀

```go
// 单 key
resp, err := cli.Get(ctx, "/config/order/timeout")

// 前缀查询，最多 10 条，按 key 升序
resp, err = cli.Get(ctx, "/config/order/",
    clientv3.WithPrefix(),
    clientv3.WithLimit(10),
    clientv3.WithSort(clientv3.SortByKey, clientv3.SortAscend),
)
for _, kv := range resp.Kvs {
    fmt.Printf("%s = %s\n", kv.Key, kv.Value)
}
fmt.Println("是否还有更多:", resp.More, "总数:", resp.Count)
```

`resp.Count` 是满足条件的总数（在 `WithCountOnly()` 时尤其有用），`resp.More` 表示是否因为 `WithLimit` 被截断。

## 3.2 分页

etcd 没有游标，分页要靠"上一页最后一个 key + 一个零字节"：

```go
func listPage(ctx context.Context, cli *clientv3.Client, prefix string, limit int64) ([]*mvccpb.KeyValue, error) {
    var all []*mvccpb.KeyValue
    start, end := prefix, clientv3.GetPrefixRangeEnd(prefix)
    for {
        resp, err := cli.Get(ctx, start, clientv3.WithRange(end), clientv3.WithLimit(limit))
        if err != nil {
            return nil, err
        }
        all = append(all, resp.Kvs...)
        if !resp.More {
            return all, nil
        }
        // 从最后一个 key 的下一个字节继续；\x00 比任何可见字符都小
        start = string(resp.Kvs[len(resp.Kvs)-1].Key) + "\x00"
    }
}
```

★ **关键认知**：分页不是快照。两次 `Get` 之间如果有写入，第二页可能包含新 key，也可能漏掉被删除的 key。需要强一致快照时，应该用 `WithRev(rev)` 固定 revision，或者一次性取完（并控制前缀规模）。

## 3.3 指定 revision 读历史

```go
now, err := cli.Get(ctx, "/config/order/timeout")
if err != nil {
    return err
}
// 读取当前 revision 之前 10 个版本的状态
old, err := cli.Get(ctx, "/config/order/timeout",
    clientv3.WithRev(now.Header.Revision-10))
if err != nil {
    return err
}
```

- `WithRev(0)` 等价于不指定，读当前最新。
- 读一个已经被 compact 的 revision 会返回 `ErrCompacted`。
- 读一个未来的 revision 会返回 `ErrFutureRev`，客户端会等待还是报错取决于调用类型；普通 Get 直接报错。

## 3.4 线性一致读与串行化读

```go
// 默认：线性一致读；WithSerializable：从本地成员读，可能落后
resp, err := cli.Get(ctx, "/config/order/timeout")
resp, err = cli.Get(ctx, "/config/order/timeout", clientv3.WithSerializable())
```

| 模式 | 一致性 | 延迟 | 适用 |
|------|--------|------|------|
| 默认线性一致 | 最强 | 需要 Leader 确认多数派 | 配置、锁、任何要求最新的读 |
| `WithSerializable` | 可能陈旧 | 本地读，低 | 监控、批量扫描、可容忍延迟 |

⚠ **关键认知**：分布式锁、CAS、服务发现的关键读**绝对不能用** `WithSerializable()`。你看到的 ModRevision 可能已经过期，基于它做 Compare 会得到错误结果。

# 4. revision 与 MVCC 的配合

## 4.1 一次写入返回什么

etcd 的 revision 是**事务级**的，不是 key 级。`resp.Header.Revision` 是这次写事务的全局 revision；一次 Txn 写了三个 key，这三个 key 的 `ModRevision` 都等于本次事务的 revision。

## 4.2 用 revision 做一致性快照

```go
// 先读一次，固定快照点
first, err := cli.Get(ctx, "/config/", clientv3.WithPrefix())
if err != nil {
    return err
}
rev := first.Header.Revision
// 后续所有读都用同一个 revision，看到的是同一时刻的状态
a, _ := cli.Get(ctx, "/config/a", clientv3.WithRev(rev))
b, _ := cli.Get(ctx, "/config/b", clientv3.WithRev(rev))
```

★ **关键认知**：revision 是 etcd 的"逻辑时钟"。需要多个 key 的**一致快照**时，先取一个 revision，再让所有读都固定在这个 revision 上；不要连续用默认读，否则中间发生的写入会让结果互相矛盾。

## 4.3 错误处理

```go
import (
    "errors"
    "go.etcd.io/etcd/api/v3/v3rpc/rpctypes"
)

_, err := cli.Get(ctx, key, clientv3.WithRev(oldRev))
switch {
case err == nil:
    // 正常
case errors.Is(err, rpctypes.ErrCompacted):
    // 历史已被压缩，重新取当前值
case errors.Is(err, rpctypes.ErrFutureRev):
    // revision 还没发生，通常是客户端算错了
case errors.Is(err, context.DeadlineExceeded):
    // 超时：网络、无 Leader、磁盘慢
default:
    return err
}
```

| 错误 | 含义 | 处理 |
|------|------|------|
| `context deadline exceeded` | 超时 | 检查网络、quorum、磁盘；有限重试 |
| `etcdserver: mvcc: required revision has been compacted` | 读的历史被压缩 | 重新读当前值 |
| `etcdserver: mvcc: required revision is a future revision` | revision 超过当前 | 修正 revision 计算 |
| `etcdserver: no leader` | 集群失去多数派 | 恢复节点，不要无限重试 |
| `etcdserver: permission denied` | 认证/权限不足 | 检查用户与角色 |

## 4.4 重试的边界

★ **关键认知**：重试只对可重试错误（超时、临时网络故障、无 Leader）有意义，权限、压缩、参数错误重试也没用。实现上用指数退避并设最大次数，每次重试前检查 `ctx.Done()`。**读可以放心重试，写要小心**：一次 `Put` 返回超时，并不代表没写成功，请求可能已经在 Leader 提交，只是响应丢了。写操作重试前要么保证幂等（同样的 key/value 覆盖），要么用 Txn + revision 做条件写，避免重复扣减、重复创建。

# 5. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 程序启动后第一个请求超时 | Endpoints 写错、防火墙、TLS 未配 | 先用 `etcdctl endpoint health` 验证 |
| 文件描述符耗尽 | 每次请求都 New 客户端，或 Watch 未取消 | 客户端单例，`defer cli.Close()` |
| 查到多余 key | 前缀没带 `/`，如 `/service` 匹配 `/service2` | 统一前缀边界 |
| 删除范围过大 | `Delete` 加了 `WithPrefix()` 但前缀太短 | 先 `Get` 确认，再删 |
| 读到旧值 | 用了 `WithSerializable()` | 关键路径改默认读 |
| `message larger than max` | value 或响应超过 gRPC 上限 | 减小 value，或调大 `MaxCallRecvMsgSize` |
| 上下文永远不结束 | 用 `context.Background()` 且无超时 | 每次请求都设 deadline |
| 写入超时后重复执行 | 没做幂等，直接重试 | 用 CAS、唯一键或业务去重 |

⚠ **关键认知**：etcd 的 value 不是"越大越好"。默认最大请求约 1.5 MB，且大 value 会放大 WAL、快照和网络复制的成本。超过几十 KB 的对象，建议把正文放对象存储，etcd 只存指针和元数据。

# 6. 动手练习

1. 写一个程序，向 `/demo/users/` 前缀写入 25 个 key，然后用 `WithLimit(10)` 分页全部读出来，打印每页的 key 数量和 `More`。
2. 连续修改同一个 key 三次，记录每次 `Put` 返回的 revision，再用 `WithRev` 读回中间版本，观察 `ModRevision` 与 `Version` 的变化。
3. 写一个带超时和指数退避的 `Get` 封装，故意把 Endpoints 指向一个不存在的地址，观察错误类型与重试次数。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 连接 | `clientv3.New` 懒连接，进程内单例，必须 Close |
| Put | 整值覆盖，可用 `WithLease` 绑定租约 |
| Get | 默认线性一致，`WithPrefix`/`WithLimit`/`WithRev` 组合使用 |
| revision | 事务级全局版本，可用于一致快照和历史读 |
| 串行化读 | 本地读、可能陈旧，不能用于锁与 CAS |
| 错误 | 区分可重试与不可重试，写重试必须幂等 |

下一篇进入 Lease 与 Watch，讲清楚租约如何自动过期、KeepAlive 通道怎么用、Watch 事件里有哪些字段、断线重连与历史压缩如何处理，以及监听如何正确取消。
