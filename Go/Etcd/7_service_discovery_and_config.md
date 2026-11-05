服务发现和配置中心是 etcd 最经典的两个落地场景：前者解决"服务实例在哪里"，后者解决"配置怎么在不重启的情况下安全变更"。两者都建立在同一组能力上——Lease 负责实例存活，前缀查询负责批量读取，Watch 负责实时变更。本篇给出可直接改造的注册、发现、配置发布与灰度实现。

# 1. 服务发现模型

## 1.1 注册中心要做什么

一个可用的服务发现至少包含四件事：注册（实例启动时写入地址和元数据）、健康（崩溃后注册信息自动消失）、发现（调用方拿到当前健康实例）、变更通知（上下线实时感知）。etcd 用 Lease 解决健康，用前缀查询解决发现，用 Watch 解决变更通知。

## 1.2 key 设计

推荐按"服务名/实例 ID"分层，实例 ID 用唯一值而不是 IP，例如 `/services/order/order-7f3c9a` 的值是 `{"addr":"10.0.0.1:8080","weight":10}`。为什么 key 用实例 ID 而不是地址：同一台机器可能跑多个实例，端口不同；容器 IP 会变，实例 ID 可以稳定复用；发布时还可以用实例 ID 做灰度、排空和审计。

## 1.3 与 Kubernetes Service 的区别

如果服务已经跑在 Kubernetes 里，优先用 Service/Endpoints/ConfigMap，不要再自建一套 etcd 注册中心。自建方案适合非 Kubernetes 环境、跨集群/跨云服务，或需要自定义元数据、权重、灰度规则，以及把服务发现与配置中心放在同一套强一致存储里的场景。

# 2. 服务注册

## 2.1 最小注册

```go
type Instance struct {
    ID       string            `json:"id"`
    Addr     string            `json:"addr"`
    Weight   int               `json:"weight"`
    Metadata map[string]string `json:"metadata"`
}

type Registry struct {
    cli     *clientv3.Client
    leaseID clientv3.LeaseID
    key     string
    stop    context.CancelFunc
}

func Register(cli *clientv3.Client, service string, inst Instance, ttl int64) (*Registry, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    lease, err := cli.Grant(ctx, ttl) // 实例活着，租约就续着
    if err != nil {
        return nil, err
    }
    raw, err := json.Marshal(inst)
    if err != nil {
        _, _ = cli.Revoke(ctx, lease.ID)
        return nil, err
    }
    key := fmt.Sprintf("/services/%s/%s", service, inst.ID)
    if _, err := cli.Put(ctx, key, string(raw), clientv3.WithLease(lease.ID)); err != nil {
        _, _ = cli.Revoke(ctx, lease.ID)
        return nil, err
    }
    return &Registry{cli: cli, leaseID: lease.ID, key: key}, nil
}
```

★ **关键认知**：注册 key **必须绑定租约**。如果只是普通 Put，实例崩溃后 key 会永久残留，调用方会一直把流量打到已经不存在的地址。租约是"健康检查"的替代品，比应用层心跳表更可靠。

## 2.2 续期与自动下线

```go
func (r *Registry) StartKeepAlive() {
    ctx, cancel := context.WithCancel(context.Background())
    r.stop = cancel
    ch, err := r.cli.KeepAlive(ctx, r.leaseID) // 通道关闭说明租约已失效
    if err != nil {
        cancel()
        return
    }
    go func() {
        for ka := range ch {
            if ka == nil {
                log.Printf("实例 %s 的租约已失效", r.key)
                return
            }
        }
    }()
}
```

注册成功后要调用 `StartKeepAlive`，否则实例会在 TTL 后消失。进程崩溃时 KeepAlive 自然停止；TTL 到期后 etcd 删除 key，发现方通过 Watch 收到 DELETE，把实例从本地列表移除。

## 2.3 优雅下线

```go
func (r *Registry) Close() error {
    if r.stop != nil {
        r.stop() // 停止续期
    }
    _, err := r.cli.Revoke(context.Background(), r.leaseID) // 主动撤销，立即摘除
    return err
}
```

优雅下线的顺序是：先从负载均衡/注册中心摘除自己（Revoke 租约），再等待一个"排空窗口"让已有请求完成，然后停止接收新请求，最后关闭进程。如果先停进程再摘除，调用方会在一段时间内继续把请求发到已关闭的实例，造成 502。

# 3. 服务发现

## 3.1 前缀查询

```go
func ListInstances(ctx context.Context, cli *clientv3.Client, service string) ([]Instance, int64, error) {
    prefix := fmt.Sprintf("/services/%s/", service)
    resp, err := cli.Get(ctx, prefix, clientv3.WithPrefix())
    if err != nil {
        return nil, 0, err
    }
    var out []Instance
    for _, kv := range resp.Kvs {
        var inst Instance
        if json.Unmarshal(kv.Value, &inst) == nil { // 跳过非法实例，避免一个坏数据拖垮全量
            out = append(out, inst)
        }
    }
    return out, resp.Header.Revision, nil
}
```

前缀末尾一定要带 `/`。`/services/order` 会匹配到 `/services/order-service/...`，把无关服务也捞进来。

## 3.2 Watch 增量变更

```go
func WatchInstances(cli *clientv3.Client, service string, onChange func([]Instance)) error {
    prefix := fmt.Sprintf("/services/%s/", service)
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    list, rev, err := ListInstances(ctx, cli, service) // 先全量读，拿快照 revision
    if err != nil {
        return err
    }
    rch := cli.Watch(ctx, prefix, // 从 rev+1 开始，保证不漏掉 Get 之后的变化
        clientv3.WithPrefix(), clientv3.WithRev(rev+1), clientv3.WithPrevKV())

    state := make(map[string]Instance) // 以实例 ID 为 key，天然支持 upsert
    for _, inst := range list {
        state[inst.ID] = inst
    }
    onChange(flatten(state))

    for wresp := range rch {
        if wresp.Canceled {
            if wresp.CompactRevision > 0 {
                return fmt.Errorf("watch 被压缩，需要重建: compactRev=%d", wresp.CompactRevision)
            }
            return fmt.Errorf("watch 取消: %v", wresp.Err())
        }
        changed := false
        for _, ev := range wresp.Events {
            id := path.Base(string(ev.Kv.Key))
            if ev.Type == mvccpb.DELETE {
                delete(state, id)
                changed = true
                continue
            }
            var inst Instance
            if err := json.Unmarshal(ev.Kv.Value, &inst); err != nil {
                continue
            }
            state[id] = inst
            changed = true
        }
        if changed {
            onChange(flatten(state))
        }
    }
    return nil
}
```

★ **关键认知**：发现方的本地状态应该按"实例 ID → 实例"存成 map，PUT 就覆盖、DELETE 就删除。这样即使断线重连后重复收到事件，结果也一致（幂等）。不要用 slice 追加，否则重复事件会产生重复实例。

## 3.3 压缩后的全量重建

Watch 返回 `CompactRevision > 0` 时，说明断线期间的历史已经被压缩，必须重新全量：调用 `ListInstances` 重建 map，拿到新的 `rev`，再把 `rev+1` 作为 Watch 起点。生产上自动压缩保留时间要大于"最长可接受断线时间"，比如保留 1 小时。

# 4. 配置中心

## 4.1 key 设计

配置的 key 要同时考虑"隔离""版本""原子发布"：

```text
/config/order-service/prod/current/{timeout,retry,version}
/config/order-service/prod/versions/{v1,v2,v3}/{timeout,retry}
```

`current/` 是当前生效配置，消费者只 Watch 这个前缀；`versions/` 是不可变历史，用于审计和回滚；环境（prod/staging）与服务名分开，避免不同环境串配置。

## 4.2 版本发布与回滚

```go
func publish(cli *clientv3.Client, app, env, version string, cfg map[string]string) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    var ops []clientv3.Op
    for k, v := range cfg { // 1. 新版本写成不可变历史
        ops = append(ops, clientv3.OpPut(
            fmt.Sprintf("/config/%s/%s/versions/%s/%s", app, env, version, k), v))
    }
    for k, v := range cfg { // 2. 原子更新 current/ 下的所有 key
        ops = append(ops, clientv3.OpPut(
            fmt.Sprintf("/config/%s/%s/current/%s", app, env, k), v))
    }
    ops = append(ops, clientv3.OpPut(
        fmt.Sprintf("/config/%s/%s/current/version", app, env), version))
    _, err := cli.Txn(ctx).Then(ops...).Commit()
    return err
}
```

一次 Txn 里写 `current/` 的所有 key，它们会落在同一个 revision。消费者收到同一批 Watch 事件，可以整体替换配置，不会读到"timeout 已是新值、retry 还是旧值"的中间状态。

回滚就是把 `versions/v2` 的值重新写回 `current/`，或者把 `current/version` 指回 v2 后再读对应版本。前者更直接，后者更省写入但需要消费者理解指针。

## 4.3 消费配置

消费配置就是 `Get(prefix, WithPrefix())`，把每个 key 的 basename 作为配置名，同时返回 `resp.Header.Revision`。拿到 `rev` 后同样用 `rev+1` 建立 Watch。配置更新时先校验（类型、范围、依赖关系），校验通过再 `publish`；校验失败直接拒绝，不要让坏配置进入 `current/`。

## 4.4 灰度配置

灰度有两类做法：

| 做法 | key 示例 | 优点 | 缺点 |
|------|----------|------|------|
| 百分比 | `/config/app/gray` = `{"version":"v2","percent":10}` | 简单 | 需要稳定的分流键 |
| 白名单 | `/config/app/gray/users` = `["1001","1002"]` | 精确 | 名单大时 value 大 |
| 按实例 | `/config/app/instances/10.0.0.1` = `v2` | 可控 | 需要实例标识 |

```json
{"version":"v2","percent":10,"match":{"header":"x-gray","value":"1"}}
```

消费者拿到灰度配置后，用请求里的稳定标识（用户 ID、设备 ID）做一致性哈希，决定走 v1 还是 v2。不要用随机数，否则同一个用户在两个版本之间来回跳。

★ **关键认知**：灰度配置本身也是配置，必须走 `current/` 的原子发布流程。不要用一个 key 存整段 JSON 再让所有实例解析，除非你能接受 value 变大和字段演进带来的兼容问题。小配置可以 JSON，大配置拆 key。

# 5. 注意事项

## 5.1 Watch 规模

- 每个进程对每个服务前缀只开一个 Watch，不要每个请求或每个实例单独开。
- Watch 回调只做入队，重活交给 worker；大规模集群中 Watch 数量是重要容量指标，监控 `etcd_debugging_mvcc_watcher_total`。

## 5.2 一致性

| 场景 | 建议 |
|------|------|
| 启动时加载全量 | 默认线性一致读 |
| 批量扫描/监控 | 可用 `WithSerializable()` |
| 多 key 配置快照 | 固定 `WithRev(rev)` |
| 服务实例列表 | 默认读，避免拿到过期实例 |

## 5.3 安全

生产环境要开启 TLS 和 RBAC：

```bash
# 创建只读用户，只允许读 /services/ 前缀
etcdctl user add watcher:password
etcdctl role add service-reader
etcdctl role grant-permission service-reader --prefix=true read /services/
etcdctl user grant-role watcher service-reader
```

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 实例下线后仍被调用 | key 没绑租约，或 TTL 太长 | `WithLease` + 合理 TTL |
| 实例列表重复 | 用 slice 追加事件，没有按 ID 去重 | 用 map 做 upsert |
| 配置更新后部分实例没生效 | Watch 断开后没重建 | 处理 `Canceled` 与 `CompactRevision` |
| 读到半新半旧配置 | 分多次 Put，没有用 Txn | 一次 Txn 写同一 revision |

⚠ **关键认知**：**不要用 etcd 存心跳时间戳并每秒更新**。每次更新都是一次 Raft 写，成本很高。实例存活只需要 KeepAlive 续租约，value 在注册时写一次即可；确实需要更新时间，可以拉长到分钟级并接受精度损失。

# 7. 动手练习

1. 写一个注册器和一个发现器：启动 3 个模拟实例，观察发现方能否在 2 秒内看到全部实例；kill 其中一个，观察它多久后从列表消失。
2. 实现配置的原子发布：一次 Txn 更新 `/config/demo/current/timeout` 与 `/config/demo/current/retry`，用 Watch 验证两个 key 的 `ModRevision` 相同。
3. 设计一个按用户 ID 灰度 10% 的配置结构，写代码用哈希决定版本，并保证同一用户多次请求命中同一版本。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 注册 | 实例 ID 做 key，value 存地址与元数据，必须绑 Lease |
| 健康 | KeepAlive 续租，进程崩溃后 TTL 自动摘除 |
| 下线 | 先 Revoke 摘除，再排空，最后停进程 |
| 发现 | 前缀查询 + `rev+1` Watch，本地用 map 幂等更新 |
| 配置 | `current/` 生效，`versions/` 保留历史 |
| 原子发布 | 一次 Txn 写多个 key，共享同一 revision |
| 灰度 | 稳定分流键 + 百分比/白名单，避免用户来回跳 |
| 规模 | 每服务一个 Watch，回调只入队，监控 Watch 数量 |

下一篇进入运维，讲清楚成员增删、快照备份与恢复、compact 与 defrag、关键监控指标、容量规划，以及一套可以真正执行的故障演练清单。
