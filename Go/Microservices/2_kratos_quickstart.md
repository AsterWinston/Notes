Kratos 是 B 站开源的 Go 微服务框架，v2 的设计目标很明确：**用一套工程约定，把 HTTP 与 gRPC 服务、依赖注入、配置、注册发现、可观测性串起来**，让业务代码只关心领域逻辑。本篇从 `kratos new` 开始，把生成出来的目录逐个拆开讲，再动手改一个能跑通的接口。

# 1. Kratos 的设计取向

## 1.1 框架定位

Kratos v2 不是"大而全"的脚手架，而是一组可组合的库：`transport/http`、`transport/grpc`、`middleware`、`config`、`registry`、`log` 各自独立，最后通过 `kratos.App` 组装。

| 能力 | 包路径 | 作用 |
|------|--------|------|
| 应用生命周期 | `github.com/go-kratos/kratos/v2` | 启动、停止、信号处理 |
| 传输 | `transport/http`、`transport/grpc` | 两种协议同进程共存 |
| 中间件 | `middleware/*` | 日志、恢复、追踪、限流、超时 |
| 配置 | `config` | 多源加载、热更新 |
| 注册发现 | `registry` + contrib | Etcd、Consul、Nacos |

★ **关键认知**：Kratos 的核心是**分层 + 依赖倒置**。`service` 依赖 `biz` 的接口，`data` 实现接口，`biz` 不 import 任何数据库包。这样业务逻辑可以在没有数据库的情况下单测，替换存储也不牵动上层。

## 1.2 分层结构

```text
cmd/server            启动入口，负责装配
   │
   ├── internal/server   传输层：HTTP、gRPC 服务器注册
   ├── internal/service  接口层：proto 请求 → 领域调用
   ├── internal/biz      领域层：用例与领域接口
   └── internal/data     数据层：实现 biz 定义的 Repo
```

依赖方向是单向的：`server → service → biz ← data`。`biz` 定义接口，`data` 反向实现，这就是依赖倒置。

# 2. 安装与创建项目

## 2.1 安装 CLI

```bash
go install github.com/go-kratos/kratos/cmd/kratos/v2@latest
```

如果 `kratos` 不在 PATH 里，把 `$(go env GOPATH)/bin` 加进去。CLI 只负责生成骨架与 proto 代码，运行时不依赖它。

## 2.2 生成项目

```bash
kratos new helloworld
cd helloworld && go mod tidy
kratos run   # 默认监听 HTTP 8000、gRPC 9000
```

## 2.3 目录解读

```text
helloworld
├── api/helloworld/v1/greeter.proto   # 接口契约
├── cmd/server/{main,wire,wire_gen}.go # 入口与装配
├── configs/config.yaml               # 配置
└── internal
    ├── biz/greeter.go                # 用例与 Repo 接口
    ├── data/greeter.go               # Repo 实现
    ├── server/{http,grpc}.go         # 两种传输
    └── service/greeter.go            # 接口实现
```

★ **关键认知**：`api/` 是**对外契约**，`internal/` 是**内部实现**。Go 的 `internal` 目录规则保证外部模块无法 import 你的实现，但可以 import `api/`。这正好对应微服务边界：契约可以共享，实现必须私有。

⚠ **关键认知**：不要把所有逻辑塞进 `service`。它只做三件事：参数校验、调用 `biz`、转换错误。业务规则写在 `biz`，数据库细节写在 `data`，否则分层形同虚设。

# 3. 定义第一个接口

## 3.1 proto 文件

打开 `api/helloworld/v1/greeter.proto`：

```protobuf
syntax = "proto3";

package helloworld.v1;

import "google/api/annotations.proto";

option go_package = "helloworld/api/helloworld/v1;v1";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      get: "/helloworld/{name}"
    };
  }
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

`google.api.http` 注解让同一份定义同时生成 HTTP 路由和 gRPC 方法。路径里的 `{name}` 会自动绑定到 `HelloRequest.name`。

## 3.2 生成代码

```bash
kratos proto client api/helloworld/v1/greeter.proto
```

常用子命令：`kratos proto add` 新建模板，`proto client` 生成 `.pb.go` 与 HTTP 桩，`proto server` 生成 service 模板。生成后 `internal/server/http.go` 里会多一行 `v1.RegisterGreeterHTTPServer(srv, greeter)`，说明路由已经挂上。

## 3.3 实现接口

```go
package service

import (
	"context"

	v1 "helloworld/api/helloworld/v1"
	"helloworld/internal/biz"

	"github.com/go-kratos/kratos/v2/log"
)

type GreeterService struct {
	v1.UnimplementedGreeterServer

	uc  *biz.GreeterUsecase
	log *log.Helper
}

func NewGreeterService(uc *biz.GreeterUsecase, logger log.Logger) *GreeterService {
	return &GreeterService{uc: uc, log: log.NewHelper(logger)}
}

func (s *GreeterService) SayHello(ctx context.Context, req *v1.HelloRequest) (*v1.HelloReply, error) {
	if req.GetName() == "" { // 参数校验放接口层
		return nil, v1.ErrorInvalidArgument("name 不能为空")
	}
	return &v1.HelloReply{Message: "hello " + req.GetName()}, nil
}
```

`v1.ErrorInvalidArgument` 来自生成的错误定义。Kratos 的错误会带上 HTTP 状态码、gRPC code 和业务 reason，第 5 篇会详细讲。

# 4. 业务分层

## 4.1 biz 定义用例

```go
package biz

import (
	"context"

	"github.com/go-kratos/kratos/v2/log"
)

type Greeter struct {
	Message string
}

// GreeterRepo 由 data 层实现，biz 只依赖接口
type GreeterRepo interface {
	Save(ctx context.Context, g *Greeter) (*Greeter, error)
}

type GreeterUsecase struct {
	repo GreeterRepo
	log  *log.Helper
}

func NewGreeterUsecase(repo GreeterRepo, logger log.Logger) *GreeterUsecase {
	return &GreeterUsecase{repo: repo, log: log.NewHelper(logger)}
}

func (uc *GreeterUsecase) CreateGreeter(ctx context.Context, g *Greeter) (*Greeter, error) {
	uc.log.WithContext(ctx).Infof("create greeter: %s", g.Message)
	return uc.repo.Save(ctx, g)
}
```

## 4.2 data 实现接口

`internal/data/greeter.go` 实现 `biz.GreeterRepo`，把接口依赖指回 `biz`：

```go
func NewGreeterRepo(data *Data, logger log.Logger) biz.GreeterRepo {
	return &greeterRepo{data: data, log: log.NewHelper(logger)}
}

func (r *greeterRepo) Save(ctx context.Context, g *biz.Greeter) (*biz.Greeter, error) {
	return g, nil // 换成真正的数据库写入
}
```

★ **关键认知**：依赖倒置的关键在**接口归属**。`GreeterRepo` 必须定义在 `biz` 而不是 `data`，否则 `biz` 就反向依赖了 `data`，分层失效。判断方法：`go list -deps` 里 `biz` 不应出现任何数据库驱动。

## 4.3 Wire 装配

```go
//go:build wireinject
// +build wireinject

package main

import "github.com/google/wire"

func wireApp(*conf.Server, *conf.Data, *conf.Registry, log.Logger) (*kratos.App, func(), error) {
	panic(wire.Build(server.ProviderSet, data.ProviderSet, biz.ProviderSet, service.ProviderSet, newApp))
}
```

改完接口后重新生成：

```bash
wire ./cmd/server
```

⚠ **关键认知**：`wire_gen.go` 是生成文件，不要手改。新增依赖时改 `wire.go` 和各层 `ProviderSet`，再跑 `wire`；否则下次生成会覆盖你的修改。

# 5. 启动与注册

## 5.1 配置文件

```yaml
server:
  http:
    addr: 0.0.0.0:8000
    timeout: 1s
  grpc:
    addr: 0.0.0.0:9000
    timeout: 1s

registry:
  etcd:
    address: 127.0.0.1:2379
```

## 5.2 注册到 Etcd

引入 `etcdreg`、`clientv3` 与 `conf` 后：

```go
func NewRegistrar(c *conf.Registry) registry.Registrar {
	client, _ := clientv3.New(clientv3.Config{Endpoints: []string{c.Etcd.Address}})
	return etcdreg.New(client)
}
```

在 `wire.go` 的 `wire.Build` 里加入它，并在 `newApp` 里传入：

```go
func newApp(gs *grpc.Server, hs *http.Server, r registry.Registrar) *kratos.App {
	return kratos.New(
		kratos.Name("helloworld"),
		kratos.Server(gs, hs),
		kratos.Registrar(r), // 启动时注册，退出时摘除
	)
}
```

★ **关键认知**：`kratos.App` 会监听 `SIGTERM`/`SIGINT`，先摘除注册、再等待在途请求、最后关闭服务器。不要自己写一套 `http.Server.Shutdown`，否则容易和框架的信号处理打架。

## 5.3 验证

```bash
curl http://localhost:8000/helloworld/gopher
# {"message":"hello gopher"}
grpcurl -plaintext -d '{"name":"gopher"}' localhost:9000 helloworld.v1.Greeter/SayHello
etcdctl get --prefix /helloworld
```

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `kratos: command not found` | GOPATH/bin 不在 PATH | 加入环境变量 |
| HTTP 404 | proto 改了但没重新生成 | 重跑 `kratos proto client` |
| `wire_gen.go` 冲突 | 手改了生成文件 | 改 `wire.go` 后重跑 `wire` |
| 启动即退出 | 配置文件路径不对 | `-conf ./configs` 指到目录 |
| Etcd 注册失败 | 地址或 scheme 错误 | 检查 `127.0.0.1:2379` 是否可连 |

# 7. 练习

1. 新增一个 `GetUser(id int64)` 方法，HTTP 路径为 `GET /v1/users/{id}`，返回固定用户信息。
2. 在 `biz` 里加一个内存计数器，记录 `SayHello` 调用次数，通过 gRPC 暴露查询方法。
3. 把 `data` 层的 `Save` 改成写 Redis，观察 `biz` 和 `service` 是否需要改动，体会依赖倒置。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 分层 | `server → service → biz ← data`，biz 定义接口 |
| proto | 一份定义同时生成 gRPC 与 HTTP 路由 |
| 生成 | `kratos proto client` 生成桩，`wire` 生成装配 |
| 启动 | `kratos.App` 统一管理生命周期与信号 |
| 注册 | 实现 `registry.Registrar`，交给 `kratos.New` |
| 错误 | 用生成的错误类型，带 code 与 reason |

下一篇进入 go-zero 1.x，用 `.api` 文件和 `goctl` 走另一条代码生成路线，对比它与 Kratos 在目录约定、RPC 组织和工程化上的差异。
