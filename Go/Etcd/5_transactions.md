单个 key 的 Put 是原子的，但真实业务经常要求"先判断、再修改"，或者"多个 key 一起改"。如果判断和修改分成两次请求，中间就会被其他客户端插队。etcd 的 Txn 用 If/Then/Else 把条件判断和一批读写合成一次 Raft 提案，是 CAS、乐观锁、选主和配置原子更新的基础。

# 1. 为什么需要事务

## 1.1 单 key 原子不够

假设要实现"配置只有没被人改过才更新"：客户端 A 和 B 都 Get 到 version=3，A 先 Put 成功，B 再 Put 也成功，A 的更新被静默覆盖。两次 Put 各自原子，但"比较版本 + 写入"整体不原子。要解决这个问题，必须让服务端在**同一个原子操作**里完成比较和写入。

## 1.2 etcd 事务的定位

| 能力 | 说明 |
|------|------|
| 条件判断 | 一组 Compare，全部满足才走 Then，否则走 Else |
| 原子写入 | Then/Else 里的多个 Put/Delete 在同一 revision 提交 |
| 一致快照 | 所有 Compare 在同一个状态快照上求值 |
| 无交互 | 不能"先读结果再决定下一步"，所有操作必须提前构造好 |
| 有限规模 | 操作数受 `--max-txn-ops` 限制，默认 128 |

★ **关键认知**：etcd 的事务是**批处理式**的，不是数据库里那种可以 `SELECT` 后根据结果再 `UPDATE` 的交互式事务。你能做的是"把可能的分支都写出来"：If 条件决定走 Then 还是 Else，而不是在事务中间读一个值再动态拼 SQL。

# 2. Txn 的结构

## 2.1 If / Then / Else

```go
resp, err := cli.Txn(ctx).
    If(
        clientv3.Compare(clientv3.ModRevision("/config/order/timeout"), "=", 42),
    ).
    Then(
        clientv3.OpPut("/config/order/timeout", "5s"),
    ).
    Else(
        clientv3.OpGet("/config/order/timeout"),
    ).
    Commit()
if err != nil {
    return err
}
fmt.Println("条件是否成立:", resp.Succeeded)
```

执行逻辑：所有 Compare 在同一个状态快照上求值，全部成立走 Then，任一不成立走 Else；选中的操作在同一个 revision 原子提交。`If` 可以写多个 Compare，它们之间是 **AND** 关系。没有 OR，需要 OR 时拆成多个 Txn 或调整 key 设计。

## 2.2 最小 CAS 示例

CAS（Compare-And-Swap）是"值等于旧值时才写入新值"：

```go
func compareAndSwap(cli *clientv3.Client, key, oldValue, newValue string) (bool, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    resp, err := cli.Txn(ctx).
        If(clientv3.Compare(clientv3.Value(key), "=", oldValue)).
        Then(clientv3.OpPut(key, newValue)).
        Commit()
    if err != nil {
        return false, err
    }
    return resp.Succeeded, nil
}
```

- 返回 `Succeeded=true`：条件成立，Put 已提交。
- 返回 `Succeeded=false`：条件不成立，Then 没执行，key 未被修改。
- 返回 `err != nil`：请求本身失败，**不能假设没写成功**，需要按幂等逻辑处理。

## 2.3 返回结构

```go
resp, err := cli.Txn(ctx).
    If(clientv3.Compare(clientv3.CreateRevision("/lock/leader"), "=", 0)).
    Then(clientv3.OpPut("/lock/leader", "node-1")).
    Else(clientv3.OpGet("/lock/leader")).
    Commit()
if resp.Succeeded {
    fmt.Println("抢注成功")
} else {
    getResp := resp.Responses[0].GetResponseRange() // 与 Else 的操作一一对应
    for _, kv := range getResp.Kvs {
        fmt.Printf("当前持有者: %s\n", kv.Value)
    }
}
```

| 字段 | 含义 |
|------|------|
| `Succeeded` | 走的是 Then 还是 Else |
| `Responses` | 每个操作的响应，顺序与 Then/Else 中的操作一致 |
| `Header.Revision` | 本次事务提交的全局 revision |

读取响应时用 `GetResponseRange()`、`GetResponsePut()`、`GetResponseDeleteRange()` 这些方法，不要直接断言具体类型。

# 3. Compare 条件

## 3.1 五种比较目标

| 构造函数 | 比较对象 | 常用场景 |
|----------|----------|----------|
| `clientv3.Version(key)` | key 被修改的次数 | 判断是否被改过 |
| `clientv3.CreateRevision(key)` | key 创建时的 revision | 判断 key 是否存在 |
| `clientv3.ModRevision(key)` | key 最近修改的 revision | 精确 CAS |
| `clientv3.Value(key)` | key 的原始字节值 | 值匹配 |
| `clientv3.LeaseValue(key)` | 绑定的租约 ID | 判断租约归属 |

## 3.2 比较符

| 目标类型 | 支持的比较符 |
|----------|--------------|
| revision 类（Create/Mod） | `=`、`!=`、`>`、`<` |
| Version | `=`、`!=`、`>`、`<` |
| Value | `=`、`!=` |
| LeaseValue | `=`、`!=` |

⚠ **关键认知**：`Value` 比较是**字节字典序**，不是数字比较。用 `Value("/stock") >= "10"` 判断库存时，字符串 `"9"` 会被认为大于 `"10"`（因为字符 `9` 大于 `1`）。库存、计数这类数值不要用 Value 比较，改用 CAS 循环或固定宽度的零填充字符串。

## 3.3 判断 key 是否存在

```go
// 正确：CreateRevision 为 0 表示 key 从未创建
clientv3.Compare(clientv3.CreateRevision(key), "=", 0)

// 不可靠：值本来就是空字符串时，无法区分"不存在"和"存在但为空"
clientv3.Compare(clientv3.Value(key), "=", "")
```

同理，判断"key 存在"用 `CreateRevision(key) > 0`。不要用 `Version(key) > 0` 吗？Version 对存在的 key 至少为 1，也能判断，但 CreateRevision 更直观。

## 3.4 多个条件

```go
resp, err := cli.Txn(ctx).
    If(
        clientv3.Compare(clientv3.ModRevision("/order/1001"), "=", rev),
        clientv3.Compare(clientv3.Value("/order/1001/status"), "=", "pending"),
    ).
    Then(clientv3.OpPut("/order/1001/status", "paid")).
    Commit()
```

两个条件必须同时成立。注意：两个 key 的读取发生在同一个 revision 快照上，不会出现"第一个条件成立、第二个条件读到更新值"的撕裂。

# 4. 原子性与一致性边界

## 4.1 一次事务怎么执行

```text
1. 客户端把 Compare + Then/Else 操作打包成一个 Txn 请求
2. 请求被转发给 Leader，进入 Raft 日志
3. 多数派持久化后提交
4. 状态机在同一个 BoltDB 写事务里：
   a. 按当前状态求值所有 Compare
   b. 选择 Then 或 Else
   c. 依次应用选中的操作
   d. 所有改动共享同一个 revision
5. 返回 Succeeded 与每个操作的响应
```

★ **关键认知**：etcd 事务的隔离级别可以理解为**可串行化**：所有事务按日志顺序执行，Compare 看到的是执行到它之前的最新状态。它比数据库的 Read Committed 更强，但代价是每个事务都要走一次 Raft。

### 4.1.1 不是交互式事务

想在事务里读一个值、再根据值决定写什么，etcd 做不到，因为 Then/Else 必须在发请求前就构造好。正确做法有两种：先 Get 拿到 revision，再用这个 revision 做 Compare 发起 CAS，冲突就重试；或者把"根据值决定写什么"的逻辑放到客户端，但用 revision 保证"读到的值在写之前没变"。

## 4.2 事务内多操作

```go
resp, err := cli.Txn(ctx).
    If(clientv3.Compare(clientv3.ModRevision("/config/order/version"), "=", oldRev)).
    Then(
        clientv3.OpPut("/config/order/version", "v2"),
        clientv3.OpPut("/config/order/timeout", "5s"),
        clientv3.OpPut("/config/order/retry", "3"),
    ).
    Else(
        clientv3.OpGet("/config/order/version"),
    ).
    Commit()
```

三个 Put 要么一起成功，要么一起不执行（条件不成立时走 Else）。它们属于同一个 revision，消费者用 Watch 会收到同一批事件。

⚠ **关键认知**：不要在同一个事务里对同一个 key 连续写多次并依赖执行顺序。事务是"一批操作的原子提交"，不是给你做中间状态推导的脚本。需要多步依赖时，拆成多个事务，用 revision 做前后置条件。

## 4.3 失败与重试

| 情况 | 是否已提交 | 处理 |
|------|------------|------|
| `Succeeded=false` | 没走 Then，事务本身成功 | 读 Else 响应，决定重试或放弃 |
| 请求超时/连接断开 | 不确定 | 用 Get 检查结果，或设计幂等写 |
| `ErrCompacted` | 请求未执行 | 重新取 revision 再试 |
| `no leader` | 未提交 | 等集群恢复，有限重试 |
| 操作数超限 | 未提交 | 拆分事务 |

★ **关键认知**：**CAS 失败不是错误，是并发冲突的正常信号**。上层要有"重新读取 → 重新计算 → 再 CAS"的循环，并设置最大重试次数。无界重试在热点 key 上会把 CPU 打满。

# 5. 典型用法

## 5.1 乐观并发更新配置

```go
func updateConfig(cli *clientv3.Client, key, newValue string) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    for attempt := 0; attempt < 5; attempt++ {
        resp, err := cli.Get(ctx, key)
        if err != nil {
            return err
        }
        var modRev int64
        if len(resp.Kvs) > 0 {
            modRev = resp.Kvs[0].ModRevision
        }
        txnResp, err := cli.Txn(ctx).
            If(clientv3.Compare(clientv3.ModRevision(key), "=", modRev)).
            Then(clientv3.OpPut(key, newValue)).
            Commit()
        if err != nil {
            return err
        }
        if txnResp.Succeeded {
            return nil // 更新成功
        }
        // 冲突：key 在 Get 与 Txn 之间被别人改了，重读再试
    }
    return errors.New("配置更新冲突次数过多")
}
```

这个模式就是 etcd 版的乐观锁：读的时候记下 `ModRevision`，写的时候要求它没变。Kubernetes 的 `resourceVersion` 用的就是同一套思路。

## 5.2 抢注与选主

```go
func tryAcquire(cli *clientv3.Client, key, candidate string, leaseID clientv3.LeaseID) (bool, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    resp, err := cli.Txn(ctx).
        If(clientv3.Compare(clientv3.CreateRevision(key), "=", 0)).
        Then(clientv3.OpPut(key, candidate, clientv3.WithLease(leaseID))).
        Commit()
    return err == nil && resp.Succeeded, err
}
```

- key 不存在时抢注成功。
- 已有 Leader 时抢注失败，可以 Watch 这个 key，等它被删除后再试。
- 绑定的租约到期后 key 自动删除，下一个候选者才能抢到。

这实现了**互斥**，但没有**公平性**：所有候选者同时抢，谁先到谁赢。需要 FIFO 公平锁时，要用"前缀 + CreateRevision 排序"，这在分布式锁篇展开。

## 5.3 多 key 原子更新

```go
resp, err := cli.Txn(ctx).
    If(clientv3.Compare(clientv3.ModRevision("/config/order/current"), "=", rev)).
    Then(
        clientv3.OpPut("/config/order/v2/timeout", "5s"),
        clientv3.OpPut("/config/order/v2/retry", "3"),
        clientv3.OpPut("/config/order/current", "v2"),
    ).Commit()
```

三个 key 在同一 revision 生效。消费者 Watch `/config/order/` 前缀时，会一次性收到三件事，可以按 revision 判断这是一次完整发布。

# 6. 常见误用

| 误用 | 后果 | 正确做法 |
|------|------|----------|
| 先 Get 再无条件 Put | 并发覆盖 | 用 ModRevision 做 Compare |
| 用 `Value(key)=""` 判断不存在 | 空值 key 被误判 | 用 `CreateRevision(key)=0` |
| 事务里对同一 key 多次写 | 行为难预期 | 一个事务只写一次同一 key |
| 把 Txn 当交互式事务 | 读结果无法回流 | 先读后 CAS，或拆分逻辑 |
| 认为超时等于没写 | 可能已提交 | 幂等设计 + 事后读校验 |
| 用 Txn 包住外部调用 | etcd 无法回滚外部副作用 | 外部操作放事务外，用补偿 |

⚠ **关键认知**：etcd 事务只保证 **etcd 内部 key 的原子性**。事务里发 HTTP、写数据库、发消息，都不受事务保护；失败时也不会自动回滚。跨系统的原子性要靠本地消息表、Saga、对账等方案。

# 7. 动手练习

1. 用 Txn 实现一个 CAS：只有 `/counter` 的 `ModRevision` 等于读到的值时，才把它加一。启动 20 个 goroutine 并发执行，统计成功与冲突次数。
2. 用 `CreateRevision(key) = 0` 实现一个"只允许创建一次"的初始化逻辑，验证第二次执行返回 `Succeeded=false`。
3. 写一个事务，同时更新 `/config/app/version` 与 `/config/app/timeout`，用 Watch 观察两个 key 是否落在同一个 revision。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Txn 结构 | If 条件、Then 成功分支、Else 失败分支 |
| 条件 | 多条件 AND，全部在同一个快照上求值 |
| Compare | Version、CreateRevision、ModRevision、Value、LeaseValue |
| 原子性 | 选中分支的操作在同一 revision 提交 |
| 隔离 | 可串行化，但不支持交互式读后决策 |
| CAS | 读 revision → Compare → 冲突重试 |
| 存在性 | 用 CreateRevision 判断，不要用空值 |
| 边界 | 只管 etcd key，不管外部副作用 |

下一篇进入分布式锁，讲清楚 Lease + Txn 如何实现互斥、为什么需要唯一租约与续期、前缀排序如何保证公平、会话失效后锁会怎样，以及 etcd 锁与 Redis 锁在一致性上的本质差异。
