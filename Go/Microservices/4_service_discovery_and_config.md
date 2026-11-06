服务实例的地址是动态的：扩缩容、滚动发布、故障迁移都会让 IP 变化。把地址写死在配置文件里，等于每次发布都改配置；而注册中心把"实例在哪"变成一条可订阅的数据。本篇讲清注册发现的模型、Etcd 与 Consul 的差异，以及配置中心怎么做环境隔离和热更新。

# 1. 为什么需要注册中心

## 1.1 静态配置的问题

```yaml
# 写死地址的配置
order_service:
  addr: 10.0.0.5:8000
```

这段配置在三种情况下立即失效：实例扩容到 3 个、实例迁移到另一台机器、实例崩溃后调度到新节点。运维只能改配置重启，发布窗口被迫拉长。

## 1.2 注册中心职责

| 职责 | 说明 |
|------|------|
| 注册 | 实例启动时写入地址、端口、权重、元数据 |
| 健康 | 实例崩溃后自动从列表移除 |
| 发现 | 调用方拉取当前健康实例列表 |
| 通知 | 实例上下线时实时推送变更 |

★ **关键认知**：注册中心不是"服务的数据库"，而是**一份带过期时间的视图**。它允许短时间不一致，但必须保证"死掉的实例最终会消失"。理解这一点，就不会纠结"为什么刚注册的实例偶尔查不到"。

## 1.3 CAP 取舍

| 系统 | 一致性模型 | 特点 |
|------|-----------|------|
| Etcd | CP（Raft） | 强一致，写入需多数派确认 |
| Consul | CP（Raft） | 功能全，自带健康检查与 DNS |
| Nacos | AP/CP 可切换 | 注册默认 AP，配置走 CP |
| Eureka | AP | 可用性优先，允许读到旧列表 |

服务发现场景通常**可用性优先**：宁可短暂拿到一个已下线实例（调用失败后重试），也不要因为注册中心不可写导致新实例无法上线。

# 2. 注册模型

## 2.1 服务、实例、元数据

```text
Service: order-service
├── Instance: order-7f3c9a  {addr: 10.0.0.5:8000, weight: 100}
└── Instance: order-8a1b2c  {addr: 10.0.0.6:8000, weight: 100}
```

**实例 ID** 要稳定且唯一，推荐"服务名 + 主机名 + 随机后缀"或 Pod 名，不要用 IP。容器重建后 IP 会变，但实例 ID 可以复用来做灰度、排空和审计。

**元数据** 用来承载版本、机房、权重、灰度标签，网关和负载均衡器据此做路由。

## 2.2 租约与健康检查

两种健康模型：

| 模型 | 工作方式 | 代表 |
|------|----------|------|
| 客户端心跳 | 实例定期续租，过期删除 | Etcd Lease、Nacos 临时实例 |
| 服务端探测 | 注册中心主动探活 | Consul、Nacos 持久实例 |
| 客户端主动上报 | 实例定时上报状态 | 自研注册中心 |

★ **关键认知**：Etcd 的注册 key **必须绑定 Lease**。普通 Put 的 key 不会因为进程崩溃而消失，调用方会一直把流量打到不存在的地址。Lease 的 TTL 就是"多久没心跳就认为死亡"。

## 2.3 注册与摘除时序

```text
实例启动
   │
   ├─ 1. 监听端口，健康检查返回 200
   ├─ 2. 向注册中心注册（带 Lease/TTL）
   ├─ 3. 开始接收流量
   │
   │  ……运行中定期续租……
   │
实例关闭
   ├─ 4. 先从注册中心摘除（Revoke / Deregister）
   ├─ 5. 等待排空窗口（如 10s），让在途请求完成
   ├─ 6. 停止接收新请求
   └─ 7. 关闭进程
```

⚠ **关键认知**：顺序错了就会 502。先停进程再摘除，调用方会在一段时间内继续把请求发到已关闭的端口；先摘除再立刻停进程，正在处理的请求会被打断。正确做法是**先摘除、再排空、最后关闭**。

# 3. Etcd 实践

## 3.1 key 设计

```text
/services/{service}/{instanceID}  →  {"addr":"10.0.0.5:8000","weight":100}
/config/{env}/{service}/{key}     →  配置值
```

前缀末尾带 `/`，否则 `/services/order` 会误匹配 `/services/order-service/...`。

## 3.2 Kratos 接入

```go
import (
	etcdreg "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
	"github.com/go-kratos/kratos/v2/registry"
	clientv3 "go.etcd.io/etcd/client/v3"
)

func NewRegistrar(endpoints []string) (registry.Registrar, func(), error) {
	client, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		return nil, nil, err
	}
	cleanup := func() { _ = client.Close() }
	return etcdreg.New(client), cleanup, nil
}
```

服务启动时通过 `kratos.New(kratos.Registrar(r))` 注册，退出时自动摘除。调用方用 discovery：

```go
conn, err := grpc.DialInsecure(ctx,
	grpc.WithEndpoint("discovery:///order-service"),
	grpc.WithDiscovery(discovery),
)
```

`discovery:///` 告诉 Kratos 走注册中心解析，后面的名字就是服务名。

## 3.3 go-zero 接入

go-zero 的 zrpc 只需要在 yaml 里填 Etcd：

```yaml
Name: order.rpc
ListenOn: 0.0.0.0:8080
Etcd:
  Hosts:
    - 127.0.0.1:2379
  Key: order.rpc
```

调用方配置同样的 `Hosts` 和 `Key`，`zrpc.MustNewClient` 会自动发现并负载均衡。Key 不一致是最常见的"调用不到"原因。

## 3.4 直接使用 clientv3

不使用框架时，注册逻辑如下：

```go
lease, _ := cli.Grant(ctx, 10) // 10 秒租约
key := "/services/order/order-7f3c9a"
_, _ = cli.Put(ctx, key, `{"addr":"10.0.0.5:8000"}`, clientv3.WithLease(lease.ID))
go func() {
	ch, _ := cli.KeepAlive(ctx, lease.ID)
	for ka := range ch {
		if ka == nil { // 租约失效，需要重新注册
			return
		}
	}
}()
```

★ **关键认知**：`KeepAlive` 返回的 channel 关闭或收到 `nil`，说明租约已经失效。生产代码必须处理这个信号：重新 Grant + Put，而不是假设续租永远成功。

# 4. Consul 对比

## 4.1 Consul 模型

Consul 用 Agent 模式：每个节点跑一个 Consul Agent，服务向本地 Agent 注册，Agent 负责健康检查并同步到集群。

```go
reg := &api.AgentServiceRegistration{
	Name: "order-service", ID: "order-7f3c9a", Port: 8000,
	Check: &api.AgentServiceCheck{HTTP: "http://127.0.0.1:8000/healthz", Interval: "10s"},
}
_ = client.Agent().ServiceRegister(reg)
```

Consul 自带 DNS 接口，可以 `dig order-service.service.consul`，对不支持 SDK 的老系统友好。

## 4.2 对比表

| 维度 | Etcd | Consul |
|------|------|--------|
| 定位 | 强一致 KV | 服务网格 + KV + DNS |
| 健康检查 | 依赖 Lease 心跳 | 内置 TCP/HTTP/gRPC 探测 |
| 服务发现 | SDK / 自建 resolver | SDK + DNS + HTTP API |
| 多数据中心 | 需自行方案 | 原生支持 |
| 运维复杂度 | 较低 | 较高，组件多 |
| Go 生态 | Kratos/go-zero 默认支持好 | 需引入 contrib |
| 适合 | Kubernetes 外的小中型集群 | 多数据中心、混合云 |

# 5. 配置中心

## 5.1 配置分类

| 类型 | 示例 | 变更频率 | 存储 |
|------|------|----------|------|
| 启动配置 | 端口、数据库地址 | 低 | 文件 / 配置中心 |
| 业务开关 | 灰度开关、功能开关 | 中 | 配置中心 |
| 动态参数 | 限流阈值、超时时间 | 高 | 配置中心 |
| 密钥 | 数据库密码、AK/SK | 低 | 密钥管理服务 |

★ **关键认知**：不是所有配置都适合放配置中心。数据库连接串这类"改了必须重启"的配置，放文件更简单；只有**需要不重启生效**的参数才值得引入 Watch 和热更新。

## 5.2 key 设计

```text
/config/{env}/{service}/current/{key}
/config/{env}/{service}/versions/{version}/{key}
```

- `env` 隔离开发、测试、生产。
- `current/` 是当前生效值，客户端只 Watch 这个前缀。
- `versions/` 保存不可变历史，用于审计和回滚。

发布时用一次 Txn 写 `current/` 下的所有 key，保证它们落在同一个 revision，消费者不会读到半新半旧的状态。

## 5.3 Kratos 多源配置

```go
c := config.New(
	config.WithSource(
		file.NewSource("configs"), // 本地兜底
		etcd.New(client, etcd.WithPath("/config/prod/order-service")),
	),
)
defer c.Close()
if err := c.Load(); err != nil { panic(err) }

var bc conf.Bootstrap
if err := c.Scan(&bc); err != nil { panic(err) } // 反序列化到强类型
```

多个 Source 按顺序合并，后面的覆盖前面的。本地文件保证配置中心不可用时仍能启动，远程配置负责动态覆盖。

## 5.4 热更新

```go
c.Watch("server.http.addr", func(key string, value config.Value) {
	addr, err := value.String()
	if err != nil {
		log.Errorf("invalid addr: %v", err)
		return
	}
	log.Infof("http addr changed to %s", addr)
	// 这里通常只更新内存中的参数，不重建监听端口
})
```

⚠ **关键认知**：热更新不是"所有字段都能热改"。端口、数据库连接池这类资源必须在启动时确定；能热更的通常是超时时间、限流阈值、开关。哪些字段可热更，要在配置结构里明确标注，否则会出现"改了没生效"的困惑。

# 6. 环境隔离

## 6.1 隔离维度

| 维度 | 做法 | 隔离强度 |
|------|------|----------|
| key 前缀 | `/config/prod/...` | 弱，靠约定 |
| Etcd 命名空间 | 不同 prefix + RBAC | 中 |
| 独立集群 | 每个环境一套 Etcd | 强 |
| 独立账号 | 每环境独立凭证 | 强 |

## 6.2 推荐方案

```text
生产：独立 Etcd 集群 + 独立 VPC
预发：独立 prefix + 只读账号
测试：共享集群 + prefix 隔离
本地：文件配置，不连远程
```

★ **关键认知**：环境隔离的第一原则是**生产凭证不出生产**。开发机上的配置里不应出现生产数据库密码，哪怕前缀写对了。用独立账号 + RBAC 把"可能误操作"变成"物理上做不到"。

# 7. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 实例下线后仍被调用 | key 未绑 Lease | `WithLease` + 合理 TTL |
| 刚启动就被打满 | 未完成预热就注册 | 健康检查通过后再注册 |
| 发布时大量 502 | 先停进程后摘除 | 先摘除、再排空 |
| 配置改了没生效 | Watch 的 key 写错 | 打印实际 key 与 revision |
| 配置半新半旧 | 分多次 Put | 一次 Txn 写多个 key |
| 多环境串配置 | 前缀拼接漏了 env | 统一封装 key 构造函数 |
| Etcd 压力大 | 每次心跳都写 value | 只续租，value 只写一次 |

# 8. 练习

1. 启动一个 Etcd，注册两个实例，用 `etcdctl get --prefix /services/` 观察 key 与 Lease 的关系；kill 一个实例，记录它多久后消失。
2. 用 Kratos 的 etcd config source 加载一份 YAML，修改 Etcd 中的值，验证 `Watch` 回调被触发。
3. 设计一套 `/config/{env}/{service}/` 的 key 规范，写出发布、回滚、灰度三种操作的 key 变化。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 注册中心 | 提供带过期时间的实例视图，允许短暂不一致 |
| 健康 | Etcd 用 Lease，Consul 用主动探测 |
| 摘除 | 先摘除、再排空、最后关进程 |
| 发现 | `discovery:///服务名`，不要写死 IP |
| 配置 | 只有需要热更的参数才进配置中心 |
| 原子性 | 一次 Txn 写多个 key，共享 revision |
| 隔离 | 生产用独立集群与独立凭证 |

下一篇进入 RPC 与网关，讲 gRPC 服务间调用、HTTP 网关的两种实现、超时如何跨进程传播、错误码设计，以及 proto 的版本管理。
