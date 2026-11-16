APISIX 最核心的差异点不是插件多，而是**配置全动态**：路由、上游、证书、Consumer 都存在 etcd，改完即生效，不需要 reload，也不会断连接。理解 etcd 里的数据长什么样、变更怎么同步到数据面，是排查"配置为什么不生效"和设计灰度发布的基础。

# 1. 动态配置的意义

## 1.1 reload 的代价

Nginx 的 `reload` 已经比较优雅：主进程读新配置，起新 worker，老 worker 处理完存量请求后退出。但它仍有几个问题：

- 每次 reload 都有内存峰值，配置大时更明显。
- 频繁 reload 会产生大量旧 worker，连接和 FD 短暂翻倍。
- 配置在文件里，多实例要逐个分发，容易不一致。
- 回滚靠替换文件，缺少版本和审计。

## 1.2 APISIX 的方案

```text
写配置：CI / Dashboard / curl ──► Admin API ──► etcd
读配置：etcd ──Watch──► APISIX 本地缓存 ──► 处理请求
```

- 写：Admin API 校验 schema 后写入 etcd。
- 读：数据面 Watch etcd，把变更同步到本地缓存。
- 请求路径上不访问 etcd，etcd 延迟不会加到每个请求上。
- 改一条路由只影响这条路由，不重启 worker，不断连接。

★ **关键认知**：APISIX 的动态性来自"etcd Watch + 本地缓存"，不是"更快的 reload"。etcd 是配置的唯一真相，本地缓存只是它的投影。

# 2. etcd 数据模型

## 2.1 key 结构

默认前缀是 `/apisix`，可在 `deployment.etcd.prefix` 改：

| 资源 | etcd key | 说明 |
|------|----------|------|
| Route | `/apisix/routes/{id}` | 路由 |
| Upstream | `/apisix/upstreams/{id}` | 上游 |
| Service | `/apisix/services/{id}` | 服务抽象 |
| Consumer | `/apisix/consumers/{username}` | 消费者，key 是用户名 |
| Consumer Group | `/apisix/consumer_groups/{id}` | 消费者组 |
| SSL | `/apisix/ssl/{id}` | 证书与 SNI |
| Global Rule | `/apisix/global_rules/{id}` | 全局插件 |
| Plugin Config | `/apisix/plugin_configs/{id}` | 可复用插件集合 |
| Plugin Metadata | `/apisix/plugin_metadata/{plugin_name}` | 插件全局配置 |
| Stream Route | `/apisix/stream_routes/{id}` | 四层路由 |
| Proto | `/apisix/proto/{id}` | gRPC proto 文件 |

## 2.2 value 与 revision

value 是一段 JSON，除了你写的字段，Admin API 还会补上元数据：

```json
{
  "id": "100",
  "uri": "/api/orders/*",
  "upstream_id": "200",
  "create_time": 1767225600,
  "update_time": 1767225600,
  "status": 1
}
```

etcd 为每次写入分配全局递增的 `revision`，每个 key 还有自己的 `mod_revision`。它们可以用来判断配置是否同步、审计变更、按 revision 找回旧值。

⚠ **关键认知**：不要用 `etcdctl put` 手写 APISIX 配置来"图快"。Admin API 会做 schema 校验、补默认值、维护时间字段；绕过它写入的坏数据可能让整个资源加载失败，而且很难定位。

# 3. 配置如何到达数据面

## 3.1 Watch 与增量更新

```text
etcd
 │  PUT /apisix/routes/100
 ▼
Watch 事件 ──► 更新共享内存中的配置缓存
                     │
                     ▼
              worker 读取最新配置处理请求
```

- PUT 事件：新增或更新。
- DELETE 事件：删除。
- 事件按 revision 有序，不能乱序应用。

## 3.2 一致性与断线

配置在本地缓存后，请求处理是纯内存操作，带来的结果是：

- **最终一致**：Admin API 返回成功时 etcd 已写入，但数据面可能还没应用，通常延迟在毫秒级。
- **跨地域延迟**：etcd 跨机房部署时，异地数据面的同步延迟取决于网络 RTT。
- **读旧配置**：etcd 短暂不可用时，数据面继续用旧缓存服务，不会立刻失败。

Watch 断开后，APISIX 会重连并从上次 revision 继续。如果断线太久、etcd 已压缩旧 revision，Watch 会返回 `CompactRevision`，此时必须重新全量拉取，再以新 revision+1 重新 Watch。

★ **关键认知**：Admin API 的 200 只代表"etcd 写成功"，不代表"所有网关实例都已生效"。需要强一致的发布流程时，应该在发布后主动验证，而不是写完就认为完成。

# 4. Admin API 写操作

```bash
# 创建或整体替换（PUT 带 ID）
curl -X PUT "http://127.0.0.1:9180/apisix/admin/routes/100" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "uri": "/api/orders/*",
  "upstream_id": "200"
}'

# 创建（POST 自动生成 ID）
curl -X POST "http://127.0.0.1:9180/apisix/admin/routes" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "uri": "/api/users/*",
  "upstream_id": "201"
}'

# 局部更新
curl -X PATCH "http://127.0.0.1:9180/apisix/admin/routes/100" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{ "priority": 10 }'
```

⚠ **关键认知**：Admin API 没有乐观锁。两个人同时 PATCH 同一条路由，后写入的会覆盖先写入的字段，中间没有冲突提示。多人协作时应该用 GitOps 串行发布。数组字段（如 `methods`、`nodes`）在 PATCH 时通常整体替换，不是追加。

备份与恢复：

```bash
curl -s "http://127.0.0.1:9180/apisix/admin/migrate/export" \
  -H "X-API-KEY: $ADMIN_KEY" -o backup-$(date +%Y%m%d).json

curl -X POST "http://127.0.0.1:9180/apisix/admin/migrate/import" \
  -H "X-API-KEY: $ADMIN_KEY" -H "Content-Type: application/json" \
  -d @backup-20261106.json
```

导入会覆盖同 ID 资源，务必先在测试环境验证，跨版本导入尤其要小心 schema 差异。

# 5. 灰度发布

## 5.1 traffic-split

```json
{
  "uri": "/api/orders/*",
  "upstream_id": "100",
  "plugins": {
    "traffic-split": {
      "rules": [
        {
          "match": [
            { "vars": [["http_x_canary", "==", "1"]] }
          ],
          "weighted_upstreams": [
            { "upstream_id": "200", "weight": 100 },
            { "weight": 0 }
          ]
        }
      ]
    }
  }
}
```

- `rules` 是规则数组，按顺序匹配。
- `match` 里是 `vars` 表达式，和第 2 篇的路由匹配语法一致。
- `weighted_upstreams` 里带 `upstream_id` 的走指定上游，不带 `upstream_id` 的走路由默认上游。
- 权重之和建议凑满 100，语义最清晰。

## 5.2 三种切流方式

**按 Header 全量切流**（内部测试）：上面的配置，`x-canary: 1` 走 200，其余走 100。

**按权重 10% 灰度**：

```json
{
  "weighted_upstreams": [
    { "upstream_id": "200", "weight": 10 },
    { "weight": 90 }
  ]
}
```

**按用户 ID 灰度**：把用户 ID 放在 Header 或 Cookie，用 `vars` 匹配固定前缀或范围。更复杂的按比例分流，需要自己算哈希后写入 Header，再由 `traffic-split` 匹配。

## 5.3 灰度与回滚流程

```text
1. 预发新版本 Upstream（200），不接流量
2. 用 Header 规则把内部测试流量切到 200
3. 观察错误率、延迟、日志
4. 按 1% → 10% → 50% → 100% 逐步放量
5. 每步设定观察窗口和回滚阈值
6. 全量后保留旧 Upstream 一段时间
```

回滚只需要把权重改回 `0 / 100`，或删掉 `traffic-split` 插件。配置是 etcd 里的对象，回滚也是一次写入，不需要发版。

★ **关键认知**：灰度不是"把流量切过去"这么简单，关键是**可观测 + 可回滚**。没有独立的上游标识、没有分版本的指标、没有一键回滚，灰度就是盲切。

# 6. 配置版本与回滚

## 6.1 revision 与快照

```bash
etcdctl get /apisix/routes/100 --rev=12345          # 读历史版本
etcdctl get /apisix/routes/100 -w json              # 看 mod_revision
```

APISIX 暴露的 `apisix_etcd_modify_indexes` 可以观察各资源前缀的最新 revision，用来判断数据面是否落后。

| 备份方式 | 优点 | 缺点 |
|----------|------|------|
| Admin API 导出 | 结构完整、可跨实例导入 | 依赖 Admin API 可用 |
| etcd 快照 | 包含所有 key，可整库恢复 | 体积大，恢复粒度粗 |
| Git 仓库 | 可审计、可 review、可回滚 | 需要额外同步机制 |

推荐组合：每次发布前用 Admin API 导出一份快照，同时把配置源文件提交到 Git。etcd 快照用于灾难恢复，Git 用于日常回滚和审计。

## 6.2 Standalone 模式

Standalone 完全不用 etcd：

```yaml
deployment:
  role: traditional
  role_traditional:
    config_provider: yaml
```

```yaml
routes:
  - id: "1"
    uri: /hello
    upstream:
      type: roundrobin
      nodes:
        "127.0.0.1:1980": 1
#END
```

`#END` 必须是文件最后一行，缺少它配置不会生效。Standalone 适合边缘节点、配置变化少的场景；它没有跨实例同步能力，改配置要分发文件并 reload。

⚠ **关键认知**：Standalone 模式下改 `apisix.yaml` 不会自动同步到其他实例，也没有 Admin API 的 schema 校验。它换来"没有 etcd 依赖"，代价是"回到文件分发"。

# 7. 与 Go/Etcd 模块的衔接

APISIX 的配置就是普通 etcd key，用 Go 直接读没有任何魔法。下面列出所有路由，并从当前 revision 开始 Watch：

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
        Endpoints:   []string{"127.0.0.1:2379"},
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        log.Fatal(err)
    }
    defer cli.Close()

    ctx := context.Background()

    // 1. 先全量读取，拿到当前 revision 作为快照点
    resp, err := cli.Get(ctx, "/apisix/routes/", clientv3.WithPrefix())
    if err != nil {
        log.Fatal(err)
    }
    for _, kv := range resp.Kvs {
        fmt.Printf("route key=%s value=%s\n", kv.Key, kv.Value)
    }

    // 2. 从 rev+1 开始 Watch，保证不漏掉 Get 之后的变化
    rev := resp.Header.Revision
    watch := cli.Watch(ctx, "/apisix/routes/",
        clientv3.WithPrefix(),
        clientv3.WithRev(rev+1),
        clientv3.WithPrevKV(),
    )

    for wresp := range watch {
        if wresp.Canceled {
            log.Printf("watch canceled: %v, compact_rev=%d",
                wresp.Err(), wresp.CompactRevision)
            return
        }
        for _, ev := range wresp.Events {
            fmt.Printf("type=%s key=%s value=%s\n",
                ev.Type, ev.Kv.Key, ev.Kv.Value)
        }
    }
}
```

这段代码的结构和第 7 篇 etcd 笔记里的服务发现完全一致：**先全量、记 revision、从 rev+1 Watch、本地用 map 幂等更新**。

有了直接读 etcd 的能力，就可以做配置审计、配置巡检、自建发布平台、多集群同步。读配置的结构体可以这样定义：

```go
type Route struct {
    ID         string                     `json:"id"`
    URI        string                     `json:"uri"`
    UpstreamID string                     `json:"upstream_id"`
    ServiceID  string                     `json:"service_id"`
    Plugins    map[string]json.RawMessage `json:"plugins"` // 保留原始 JSON
    Status     int                        `json:"status"`
}
```

`Plugins` 用 `json.RawMessage` 保留原始 JSON，避免每加一个插件就改结构体，记得引入 `encoding/json`。

⚠ **关键认知**：读可以随便读，**写要谨慎**。直接写 etcd 绕过 schema 校验，容易把坏配置推给所有实例；写的时候不要只写 value，APISIX 还依赖 key 的命名和 `id` 字段一致。Watch 要处理 `CompactRevision`，否则断线久了会永久卡住。

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| Admin 写成功但请求不生效 | 数据面还没同步，或 Watch 断开 | 等一秒重试，看 `apisix_etcd_modify_indexes` |
| 启动后没有路由 | etcd 为空，或 prefix 不匹配 | 检查 `deployment.etcd.prefix` |
| Watch 报 compacted | 断线超过 etcd 压缩保留期 | 重新全量，再从新 revision Watch |
| 手工 `etcdctl put` 后加载失败 | JSON 缺字段或格式错误 | 删掉坏 key，改用 Admin API |
| Standalone 配置不生效 | `apisix.yaml` 缺 `#END` | 补上最后一行 |
| 灰度切换后全部 502 | 新 Upstream 没建就切了权重 | 先建上游、验证健康，再切流量 |
| 导入后配置被覆盖 | import 会覆盖同 ID 资源 | 导入前备份，测试环境先验证 |
| 多实例配置不一致 | 实例连了不同的 etcd 集群 | 统一 etcd 地址和 prefix |

# 9. 练习

1. 用 `etcdctl get /apisix/routes/ --prefix` 观察 Admin API 创建路由后 etcd 里的真实结构，对比 `GET /apisix/admin/routes/{id}` 的返回。
2. 用 Go 写一个小程序，Watch `/apisix/routes/`，在 Admin API 修改路由时打印 PUT 事件的 key、value 和 revision。
3. 配一个 `traffic-split` 的 10% 灰度，分别用 Header 和权重两种方式切流，再用权重改回 0/100 完成回滚。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 存储 | 路由、上游、Consumer 等都在 etcd，前缀默认 `/apisix` |
| 同步 | Watch 增量更新本地缓存，请求路径不访问 etcd |
| 一致性 | Admin API 返回只代表 etcd 写入成功，数据面最终一致 |
| 写操作 | 优先 Admin API，schema 校验和默认值都由它完成 |
| 灰度 | `traffic-split` 按 Header 或权重切流，回滚只改权重 |
| 版本 | revision 审计，导出快照，Git 保存配置源 |
| Standalone | 不依赖 etcd，用 `apisix.yaml`，末尾必须有 `#END` |
| Go 衔接 | clientv3 前缀读取 + `rev+1` Watch，结构同第 7 篇 etcd |
| 原子性 | Admin API 无跨资源事务，先建新资源再切引用 |

下一篇进入部署与性能，讲清传统模式、Standalone、Kubernetes 三种部署形态，Worker 与连接池配置，缓存与共享内存，压测方法和容量规划，以及常见故障的定位路径。
