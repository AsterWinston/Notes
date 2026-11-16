gRPC 服务写完后，马上会遇到三个工程问题：浏览器怎么调用？多个实例怎么分流？链路上的代理认不认 HTTP/2？本篇把 grpc-gateway、服务发现、客户端负载均衡、健康检查、keepalive 和代理配置串起来，讲清一套 gRPC 服务怎么真正接进现有系统。

# 1. grpc-gateway：给 gRPC 加一层 HTTP 门面

## 1.1 为什么需要网关

浏览器不能直接说 gRPC（grpc-web 需要额外代理），移动端、第三方合作方也更习惯 REST。grpc-gateway 的做法是：**起一个 HTTP 反向代理，把 REST 请求翻译成 gRPC 调用**，业务代码只写一份。

```text
浏览器 ──HTTP/JSON──▶ grpc-gateway ──gRPC──▶ 业务服务
                         │
                         └── 用同一份 .proto 生成路由与编解码
```

★ **关键认知**：gateway 不是"再写一套 REST 接口"，而是**根据 proto 注解自动生成映射**。`SayHello` 只需要定义一次，gRPC 客户端和 HTTP 客户端都能调，避免两套接口语义漂移。

## 1.2 在 proto 里声明 HTTP 映射

```protobuf
syntax = "proto3";

package hello.v1;

option go_package = "example.com/gwdemo/gen;hellov1";

import "google/api/annotations.proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      post: "/v1/hello"
      body: "*"                 // 请求体整体映射到 HelloRequest
    };
  }

  rpc GetUser (GetUserRequest) returns (User) {
    option (google.api.http) = {
      get: "/v1/users/{id}"     // 路径参数自动绑定到 id 字段
    };
  }
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}

message GetUserRequest {
  int64 id = 1;
}

message User {
  int64 id = 1;
  string name = 2;
}
```

常用映射规则：

| HTTP | proto 写法 | 说明 |
|------|-----------|------|
| `GET /users/{id}` | `get: "/v1/users/{id}"` | 路径变量绑定同名字段 |
| `POST /users` | `post` + `body: "*"` | 整个 body 映射到 message |
| `POST /users` | `post` + `body: "user"` | body 只映射到 `user` 字段 |
| `PATCH /users/{id}` | `patch` + `body: "user"` | 局部更新 |
| `DELETE /users/{id}` | `delete` | 无 body |
| 查询参数 | 未在路径出现的字段 | 自动从 query 读取 |

⚠ **关键认知**：`body: "*"` 会把整个 JSON 反序列化到请求消息，**包括路径参数以外的所有字段**。如果字段既在路径里又在 body 里，行为容易混乱。推荐路径参数只放标识，body 只放实体。

## 1.3 生成与启动

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest

protoc --proto_path=proto \
  --go_out=gen --go_opt=paths=source_relative \
  --go-grpc_out=gen --go-grpc_opt=paths=source_relative \
  --grpc-gateway_out=gen --grpc-gateway_opt=paths=source_relative \
  --openapiv2_out=docs \
  proto/hello.proto
```

`protoc-gen-openapiv2` 会同时产出 Swagger/OpenAPI 文档，前端可以直接导入。

```go
package main

import (
    "context"
    "log"
    "net/http"

    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    hellov1 "example.com/gwdemo/gen"
)

func main() {
    ctx := context.Background()
    mux := runtime.NewServeMux() // gateway 的路由复用 grpc 的注册函数

    opts := []grpc.DialOption{
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    }

    // 注意：这里连的是真实 gRPC 服务地址
    if err := hellov1.RegisterGreeterHandlerFromEndpoint(ctx, mux, "localhost:50051", opts); err != nil {
        log.Fatalf("register gateway: %v", err)
    }

    log.Println("HTTP gateway on :8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatal(err)
    }
}
```

验证：

```bash
curl -X POST http://localhost:8080/v1/hello \
  -H "Content-Type: application/json" \
  -d '{"name":"gopher"}'
# {"message":"hello gopher"}

curl http://localhost:8080/v1/users/42
```

## 1.4 同端口复用

gateway 和 gRPC 通常想共用一个端口，用 `cmux` 按协议分流：

```go
import "github.com/soheilhy/cmux"

lis, _ := net.Listen("tcp", ":50051")
m := cmux.New(lis)

grpcL := m.MatchWithWriters(cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"))
httpL := m.Match(cmux.Any())

go grpcServer.Serve(grpcL)
go httpServer.Serve(httpL)

m.Serve() // 根据请求特征分发到不同 listener
```

★ **关键认知**：`cmux` 的分流依据是**首包特征**，不是端口。gRPC 请求带 `content-type: application/grpc`，普通 HTTP 不带。分流规则写错，会把 gRPC 请求送进 HTTP 处理器，表现为莫名的 404 或协议错误。

# 2. 服务发现

## 2.1 gRPC 的 target 语法

```text
dns:///user-service:50051
etcd:///services/user
passthrough:///10.0.0.5:50051
```

格式是 `scheme://authority/endpoint`。默认 `passthrough` 直接当地址用；`dns` 会把域名解析成多个 IP，交给负载均衡器。

## 2.2 DNS 发现

Kubernetes 里最常用：

```go
conn, err := grpc.NewClient(
    "dns:///user-service.default.svc.cluster.local:50051",
    grpc.WithTransportCredentials(creds),
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
)
```

★ **关键认知**：`grpc.NewClient` 用 `dns:///` 时，resolver 会定期重新解析域名，地址变化会自动更新到负载均衡器。用 `passthrough` 则只解析一次，Pod 扩缩容后客户端还在打旧 IP。

## 2.3 注册中心（etcd/Consul）

标准库没有 etcd resolver，需要引入第三方实现或自己写：

```go
import _ "github.com/grpc-ecosystem/grpc-gateway/v2/..." // 示例占位
```

更常见的做法是用服务网格（Istio、Linkerd）把发现和负载均衡下沉到 sidecar，应用代码仍然连 `localhost:50051`。这样业务不用感知注册中心，但要多维护一层基础设施。

| 方案 | 优点 | 缺点 |
|------|------|------|
| DNS（K8s Service） | 零额外组件 | 解析有延迟，粒度粗 |
| etcd/Consul resolver | 实时、可控 | 要自己维护 |
| 服务网格 | 应用无感、功能全 | 运维复杂、有 sidecar 开销 |

# 3. 客户端负载均衡

## 3.1 pick_first vs round_robin

| 策略 | 行为 | 适用 |
|------|------|------|
| `pick_first` | 连第一个可用地址，全压一个实例 | 单实例、调试 |
| `round_robin` | 轮询所有就绪地址 | 无状态服务，最常用 |
| `least_request`（部分实现） | 选负载最低的 | 请求耗时差异大 |

```go
grpc.WithDefaultServiceConfig(`{
  "loadBalancingConfig": [{"round_robin": {}}]
}`)
```

★ **关键认知**：gRPC 默认是 **`pick_first`**。很多人以为配了 DNS 就会自动分流，结果所有流量打到一个 Pod。必须显式设置 `round_robin`，并且 target 使用 `dns:///` 拿到多个地址，轮询才有意义。

## 3.2 负载均衡发生在客户端

```text
        ┌──────────────┐
        │   Client     │
        │  ┌────────┐  │
        │  │  LB    │  │  ← 在进程内选地址
        │  └───┬────┘  │
        └──────┼───────┘
        ┌──────┼───────┐
        ▼      ▼       ▼
     Server1 Server2 Server3
```

gRPC 的长连接特性决定了**服务端 L4 负载均衡效果很差**：连接建立后一直复用，新实例分不到流量。所以 gRPC 生态更依赖客户端 LB 或 L7 代理（Envoy）。

⚠ **关键认知**：K8s 默认的 Service（kube-proxy）是 L4 负载均衡，按**连接**分流。gRPC 客户端一条长连接只会在一个 Pod 上，扩缩容后流量分布极不均匀。解决方案：客户端 round_robin + headless Service，或者上 Envoy/Istio。

## 3.3 连接状态

```go
conn, _ := grpc.NewClient(target, opts...)

conn.Connect()                       // 主动触发建连
state := conn.GetState()             // IDLE / CONNECTING / READY / TRANSIENT_FAILURE / SHUTDOWN
conn.WaitForStateChange(ctx, state)  // 阻塞等待状态变化
```

★ **关键认知**：`NewClient` 是惰性的，创建时不建连。做健康探测或预热时，应该调 `Connect()` 并 `WaitForStateChange` 等到 `READY`，而不是创建完就认为服务可用。

# 4. 健康检查

## 4.1 标准健康协议

gRPC 定义了 `grpc.health.v1.Health` 服务，几乎所有框架都实现了：

```go
import (
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

hs := health.NewServer()
healthpb.RegisterHealthServer(s, hs)

// 为每个服务设置状态；空字符串表示整体服务
hs.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
hs.SetServingStatus("hello.v1.Greeter", healthpb.HealthCheckResponse_SERVING)
```

客户端可以主动探测：

```go
resp, err := healthpb.NewHealthClient(conn).Check(ctx,
    &healthpb.HealthCheckRequest{Service: "hello.v1.Greeter"})
if err == nil && resp.GetStatus() == healthpb.HealthCheckResponse_SERVING {
    // 可以放流量
}
```

## 4.2 客户端自动健康检查

```go
grpc.WithDefaultServiceConfig(`{
  "healthCheckConfig": {"serviceName": "hello.v1.Greeter"},
  "loadBalancingConfig": [{"round_robin": {}}]
}`)
```

开启后，客户端会定期调用健康检查，把不健康的地址从 LB 池里摘掉。这比"连接建立成功"更可靠——端口通不代表业务可用。

# 5. Keepalive

## 5.1 为什么需要

长连接可能被 NAT 超时、防火墙静默丢弃、负载均衡空闲回收。keepalive 通过周期性 PING 探测连接是否还活着。

```go
import "google.golang.org/grpc/keepalive"

// 服务端
grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle: 5 * time.Minute,  // 空闲多久关闭
        MaxConnectionAge:  30 * time.Minute, // 强制重连，配合 LB 打散
        Time:              2 * time.Minute,  // 多久发一次 PING
        Timeout:           10 * time.Second, // PING 超时
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             30 * time.Second, // 客户端最小 PING 间隔
        PermitWithoutStream: true,             // 无活跃流也允许 PING
    }),
)

// 客户端
grpc.WithKeepaliveParams(keepalive.ClientParameters{
    Time:                30 * time.Second,
    Timeout:             10 * time.Second,
    PermitWithoutStream: true,
})
```

## 5.2 默认策略的坑

⚠ **关键认知**：服务端默认 `MinTime` 是 5 分钟。客户端如果设了 30 秒 PING，会被判定为"too many pings"，服务端发送 `GOAWAY` 直接断开。现象是客户端日志里反复出现 `connection closed`，但业务偶发成功。两边的 keepalive 参数必须对齐。

# 6. 代理兼容

## 6.1 Nginx

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location /hello.v1.Greeter/ {
        grpc_pass grpc://backend:50051;      # 必须是 grpc_pass，不是 proxy_pass
        grpc_set_header X-Real-IP $remote_addr;
        grpc_read_timeout 300s;              # 流式调用要放大
    }
}
```

## 6.2 Envoy

```yaml
clusters:
  - name: greeter
    type: STRICT_DNS
    http2_protocol_options: {}   # upstream 必须启用 HTTP/2
```

## 6.3 常见不兼容

| 中间件 | 问题 | 处理 |
|--------|------|------|
| Nginx `proxy_pass` | 按 HTTP/1.1 转发，破坏帧 | 改用 `grpc_pass` |
| 传统 L7 网关 | 不认识 Trailers | 换支持 gRPC 的网关 |
| 旧版 Envoy | upstream 未开 HTTP/2 | 显式 `http2_protocol_options` |
| 公司透明代理 | 只允许 80/443 | gRPC 也走 443 + TLS |
| 负载均衡空闲超时 | 长连接被静默断开 | 配 keepalive 对齐超时 |

★ **关键认知**：gRPC 对代理的要求不是"能转发 TCP"，而是"**理解并保留 HTTP/2 帧语义**"。L4 代理通常没问题，L7 代理必须显式支持 gRPC，否则会出现"偶发成功、大响应失败、流式卡死"这类难查的问题。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| gateway 返回 404 | proto 注解路径与实际不符 | 检查 `google.api.http` |
| gateway 502 | 连不上 gRPC 后端 | 检查地址、TLS、端口 |
| 流量全打一个 Pod | 默认 `pick_first` | 配 `round_robin` + `dns:///` |
| `too many pings` | keepalive 间隔小于服务端 MinTime | 对齐参数 |
| 健康检查一直不健康 | service 名不匹配 | `SetServingStatus` 用全限定名 |
| 流式经 Nginx 中断 | `grpc_read_timeout` 太短 | 放大超时 |
| 扩缩容后旧连接不释放 | 没有 `MaxConnectionAge` | 配置最大连接寿命 |

# 8. 练习与自检

1. 给 `Greeter` 加一个 `GET /v1/users/{id}` 映射，用 curl 验证路径参数绑定。
2. 用两个服务端实例 + `dns:///` + `round_robin`，观察请求是否轮流命中。
3. 故意把客户端 keepalive 设成 5 秒，观察服务端是否返回 `GOAWAY`，再调回合规值。

# 9. 小结

| 主题 | 要点 |
|------|------|
| grpc-gateway | proto 注解生成 REST 映射，一份定义两种协议 |
| 服务发现 | `dns:///` 支持动态地址，注册中心按需引入 |
| 负载均衡 | 默认 `pick_first`，要分流必须显式 `round_robin` |
| 健康检查 | 标准 `grpc.health.v1`，客户端可自动摘除 |
| keepalive | 两端参数必须对齐，否则 `too many pings` |
| 代理 | L7 必须支持 HTTP/2 与 Trailers，Nginx 用 `grpc_pass` |

下一篇进入测试，讲清 bufconn 内存连接、表驱动测试、mock 服务端、流式接口测试，以及超时和错误码断言。
