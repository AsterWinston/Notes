服务拆开之后，最常写的代码不是业务逻辑，而是"调别人"和"被别人调"。调用写不好，超时会层层放大、错误码会互相污染、接口一升级就全线崩。本篇把 RPC 调用、HTTP 网关、超时传播、错误码与版本管理放在一起讲，因为它们共同决定了一条调用链是否可控。

# 1. RPC 调用基础

## 1.1 连接必须复用

```go
// 错误做法：每次请求都建连
func GetOrder(id int64) (*Order, error) {
	conn, _ := grpc.NewClient("10.0.0.5:9000", creds)
	defer conn.Close()
	return orderv1.NewOrderClient(conn).Get(ctx, &orderv1.GetRequest{Id: id})
}
```

每次建连意味着 TCP 握手、TLS 握手、HTTP/2 设置帧，延迟从毫秒涨到几十毫秒，还会耗尽端口。正确做法是**进程内复用一个 `ClientConn`**，把它放在依赖容器里。

```go
type Clients struct {
	Order orderv1.OrderClient
	User  userv1.UserClient
}

func NewClients(conn *grpc.ClientConn) *Clients {
	return &Clients{
		Order: orderv1.NewOrderClient(conn),
		User:  userv1.NewUserClient(conn),
	}
}
```

★ **关键认知**：`grpc.ClientConn` 是**并发安全**的，内部维护连接池、名称解析和负载均衡状态。一个下游服务一个 `ClientConn` 足够，不要为每个请求、每个 goroutine 新建。

## 1.2 Kratos 客户端

```go
conn, err := grpc.DialInsecure(ctx,
	grpc.WithEndpoint("discovery:///order-service"), // 走注册中心
	grpc.WithDiscovery(discovery),
)
if err != nil {
	return nil, err
}
client := orderv1.NewOrderClient(conn)
```

`WithEndpoint` 支持三种形式：

| 形式 | 示例 | 说明 |
|------|------|------|
| 直连 | `127.0.0.1:9000` | 调试用 |
| 注册中心 | `discovery:///order-service` | 生产常用 |
| 多地址 | `127.0.0.1:9000,127.0.0.1:9001` | 静态集群 |

## 1.3 go-zero 客户端

```go
type ServiceContext struct {
	Config config.Config
	Order  orderclient.Order // zrpc.MustNewClient(c.OrderRpc) 负责发现与负载均衡
}
```

zrpc 已经封装了发现、负载均衡、熔断、超时，业务侧只需 `l.svcCtx.Order.Get(ctx, req)`。

## 1.4 一次调用的完整链路

```text
HTTP → API 网关 → order-service (HTTP) → inventory-service (gRPC)
                     │ metadata: x-request-id, traceparent, timeout
                     └─ 每跳都透传请求 ID、超时预算、调用链上下文
```

每一跳都要传递**请求 ID、超时预算、调用链上下文**，缺一项就会在排障时变成黑盒。

# 2. HTTP 网关

## 2.1 两种实现路线

| 路线 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| 框架内置 HTTP | Kratos/go-zero 同时暴露 HTTP 与 gRPC | 一份业务代码，部署简单 | HTTP 与 gRPC 语义需自己对齐 |
| grpc-gateway | 独立进程把 REST 翻译成 gRPC | 严格由 proto 生成，适合多语言 | 多一层进程与网络跳转 |

## 2.2 Kratos 内置 HTTP

Kratos 的 proto 注解会同时生成 HTTP 路由与 gRPC 方法：

```protobuf
service Order {
  rpc GetOrder (GetOrderRequest) returns (OrderReply) {
    option (google.api.http) = {
      get: "/v1/orders/{id}"
    };
  }
}
```

`internal/server/http.go` 里 `v1.RegisterOrderHTTPServer(srv, svc)` 就把路由挂上了。同一个 `OrderService` 同时实现 gRPC 与 HTTP 接口，业务逻辑只有一份。

★ **关键认知**：内置 HTTP 的本质是**同一份 proto 生成两套桩**，不是"再写一个 REST 层"。这样字段名、错误码、参数校验规则天然一致，避免 REST 与 gRPC 两套语义漂移。

## 2.3 grpc-gateway

当网关需要独立部署、或需要给非 Go 服务提供 REST 入口时，用 grpc-gateway：

```go
mux := runtime.NewServeMux(runtime.WithIncomingHeaderMatcher(func(key string) (string, bool) {
	return key, key == "X-Request-Id" // 只透传白名单头
}))
_ = orderv1.RegisterOrderHandlerFromEndpoint(ctx, mux, "order-service:9000", opts)
```

## 2.4 网关该做什么

```text
鉴权 → 限流 → 路由 → 协议转换 → 错误映射 → 日志/追踪
```

网关不该做业务计算，也不该成为"什么都往里塞"的胖服务。它只解决**横切关注点**：认证、配额、灰度路由、协议适配。

# 3. 超时传播

## 3.1 context deadline 是唯一正确方式

```go
ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
defer cancel() // 必须调用，否则 timer 泄漏

resp, err := client.GetOrder(ctx, &orderv1.GetOrderRequest{Id: id})
if err != nil {
	return nil, err // 可能是 context.DeadlineExceeded
}
```

gRPC 会把 deadline 编码进 `grpc-timeout` 头，服务端拿到的 `ctx` 自带同样的截止时间。**不要用自定义头传超时**，框架已经做了。

★ **关键认知**：`context.WithTimeout` 的 `cancel` 必须 `defer` 调用。它释放的是定时器资源，不调用会导致内存泄漏；同时，提前 cancel 能让下游立刻停止无用计算。

## 3.2 服务端读取剩余预算

```go
deadline, ok := ctx.Deadline()
if !ok {
	deadline = time.Now().Add(time.Second)
}
remaining := time.Until(deadline)
if remaining < 50*time.Millisecond {
	return nil, errors.DeadlineExceeded("BUDGET_EXHAUSTED", "剩余时间不足")
}
childCtx, cancel := context.WithTimeout(ctx, remaining/2)
defer cancel()
_, _ = s.inventory.Deduct(childCtx, &inventory.DeductRequest{})
```

## 3.3 预算分配原则

| 层级 | 建议预算 | 说明 |
|------|----------|------|
| 接入层 | 3s | 面向用户的总预算 |
| 聚合服务 | 1s | 预留重试与序列化开销 |
| 单次下游 RPC | 300ms | 不超过父预算的一半 |

⚠ **关键认知**：超时不是"设一个数就完事"。如果上游 1s、下游每个都 1s，串行调三个下游就会超时。正确做法是**逐层递减**：父级 500ms，子调用最多 250ms，孙调用最多 100ms，让最外层始终能拿到结果或明确的超时错误。

# 4. 错误码

## 4.1 gRPC 状态码

| code | 数值 | 语义 | HTTP |
|------|------|------|------|
| `INVALID_ARGUMENT` | 3 | 参数错误 | 400 |
| `UNAUTHENTICATED` | 16 | 未认证 | 401 |
| `PERMISSION_DENIED` | 7 | 无权限 | 403 |
| `NOT_FOUND` | 5 | 资源不存在 | 404 |
| `RESOURCE_EXHAUSTED` | 8 | 限流/配额 | 429 |
| `INTERNAL` | 13 | 服务内部错误 | 500 |
| `UNAVAILABLE` | 14 | 暂时不可用 | 503 |
| `DEADLINE_EXCEEDED` | 4 | 超时 | 504 |

## 4.2 Kratos 错误定义

用 proto 枚举声明业务错误：

```protobuf
syntax = "proto3";

package order.v1;

import "errors/errors.proto";

option go_package = "demo/api/order/v1;v1";

enum ErrorReason {
  option (errors.default_code) = 500;

  ORDER_NOT_FOUND   = 0 [(errors.code) = 404];
  STOCK_INSUFFICIENT = 1 [(errors.code) = 409];
  INVALID_ARGUMENT  = 2 [(errors.code) = 400];
}
```

使用：

```go
return nil, v1.ErrorOrderNotFound("order %d not found", id)
```

生成的方法会带上 code、reason 和 message，HTTP 层自动映射成对应状态码。

## 4.3 业务错误设计

业务错误至少包含三段信息：`Code`（稳定标识，如 `ORDER_NOT_FOUND`）、`Message`（面向调用方，可展示）、`Detail`（排查用，不返回给用户）。

三条原则：

1. **code 稳定**：调用方按 code 分支，不解析 message。
2. **message 可变**：可以优化文案，但不能改变语义。
3. **内部细节不外泄**：数据库错误、堆栈只写日志，不返回给前端。

★ **关键认知**：错误码是**接口契约的一部分**，一旦发布就不能随意改。新增错误码要向后兼容，删除或改语义等同于破坏性变更，必须走版本管理。

## 4.4 错误映射

```go
order, err := s.repo.Find(ctx, req.GetId())
if errors.Is(err, sql.ErrNoRows) {
	return nil, v1.ErrorOrderNotFound("order %d not found", req.GetId())
}
if err != nil {
	s.log.WithContext(ctx).Errorf("query order: %v", err) // 细节进日志
	return nil, v1.ErrorInternal("INTERNAL", "系统繁忙")
}
```

⚠ **关键认知**：不要在服务间调用时把下游错误原样透传。下游的 `INTERNAL` 对上游可能是"依赖不可用"，应该转换成自己的语义，并决定是否重试。透传会让错误码在链路上层层失真。

# 5. 版本管理

## 5.1 proto 兼容规则

| 变更 | 是否兼容 | 说明 |
|------|----------|------|
| 新增字段 | 是 | 用新字段号 |
| 删除字段 | 是（需保留编号） | `reserved 5;` 防止复用 |
| 改字段类型 | 否 | 例如 int32 → string |
| 改字段编号 | 否 | 序列化错乱 |

删除字段时要用 `reserved 4, 7;` 锁住编号、`reserved "old_status";` 锁住字段名，防止后续误复用导致序列化错乱。

## 5.2 路径版本

```text
/api/order/v1/orders
/api/order/v2/orders
```

proto 的 package 也带版本：`package order.v1;`。这样 v1 与 v2 可以同时注册在同一个进程：

```go
orderv1.RegisterOrderServer(srv, v1Impl)
orderv2.RegisterOrderServer(srv, v2Impl)
```

## 5.3 并行与灰度

```text
请求 ──▶ 网关按版本路由 ──┬──▶ order-v1
                          └──▶ order-v2
```

★ **关键认知**：版本不是"复制一份代码改改"。优先做**向后兼容的增量演进**（加字段、加方法），只有语义发生不兼容变化时才开新版本。每开一个大版本，都要承担双倍维护成本。

# 6. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 调用延迟高、连接数暴涨 | 每次请求新建连接 | 进程内复用 ClientConn |
| 上游超时下游还在跑 | 没传 deadline | `context.WithTimeout` |
| 错误码全是 500 | 未映射业务错误 | 用框架错误类型 |
| 新增字段后老客户端报错 | 修改了字段编号 | 只加不改，保留编号 |

# 7. 练习

1. 用 Kratos 定义 `Order` 服务，同时暴露 `GET /v1/orders/{id}` 与 gRPC 方法，用 curl 和 grpcurl 各调一次。
2. 给调用链设置 800ms 总预算，order 服务调用 inventory 时只给 300ms，观察超时后上游拿到的错误。
3. 设计一套错误码：参数错误、库存不足、订单不存在、下游不可用，分别对应哪个 gRPC code 与 HTTP 状态码。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 连接 | ClientConn 并发安全，进程内复用 |
| 网关 | 内置 HTTP 与 grpc-gateway 按场景选 |
| 超时 | 用 context deadline，框架自动跨进程传播 |
| 预算 | 逐层递减，子调用不超过父预算一半 |
| 错误 | code 稳定、message 可变、细节不外泄 |
| 版本 | 只加不改，保留编号，必要时开 v2 |

下一篇进入可观测性，讲日志、指标、链路追踪三件套怎么落地，请求 ID 如何贯穿全链路，以及 OpenTelemetry 在 Go 服务中的最小接入方式。
