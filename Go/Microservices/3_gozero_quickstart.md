go-zero 是晓黑板开源的 Go 微服务框架，和 Kratos 最大的区别是**把工程规范直接固化进代码生成器**：你写 `.api` 描述 HTTP 接口、写 `.proto` 描述 RPC，`goctl` 就把 handler、logic、config、routes 全部生成好。本篇用一条最短路径跑通 API 服务与 RPC 服务，并讲清它和 Kratos 的取舍。

# 1. go-zero 的工程观

## 1.1 框架定位

go-zero 1.x 面向"高并发、快速落地"的场景，内置了这些能力：

| 能力 | 包 | 说明 |
|------|-----|------|
| HTTP 框架 | `rest` | 路由、中间件、参数解析、超时 |
| RPC 框架 | `zrpc` | 基于 gRPC，内置服务发现与负载均衡 |
| 代码生成 | `goctl` | API、RPC、Model、Dockerfile |
| 熔断限流 | `core/breaker`、`core/load` | 自适应降载、熔断 |
| 缓存与日志 | `core/stores`、`core/logx` | Redis/MySQL 封装、结构化日志 |

★ **关键认知**：go-zero 的核心竞争力不是运行时性能，而是**一致性**。所有团队用同一套目录、同一套配置、同一套错误处理，新人接手成本低。代价是灵活性弱于 Kratos，很多"想自己组织"的地方会被生成器约束。

## 1.2 两套服务形态

```text
HTTP 请求
   │
   ▼
┌─────────────┐      zrpc       ┌─────────────┐
│  api 服务    │ ──────────────▶ │  rpc 服务    │
│  rest       │                 │  zrpc/grpc  │
└─────────────┘                 └─────────────┘
   ▲                                   │
   │ 对外暴露 REST                      ▼
浏览器 / App / 第三方               Etcd 注册中心
```

**api 服务**负责对外协议适配，**rpc 服务**负责领域能力。也可以只写一个 api 服务，规模上来后再拆出 rpc。

# 2. 安装与创建项目

## 2.1 安装 goctl

```bash
go install github.com/zeromicro/go-zero/tools/goctl@latest
goctl --version
```

goctl 还依赖 `protoc`、`protoc-gen-go`、`protoc-gen-go-grpc`，首次使用 RPC 时按提示安装即可。

## 2.2 生成 HTTP 服务

```bash
goctl api new greet
cd greet && go mod tidy
go run greet.go -f etc/greet-api.yaml
curl http://localhost:8888/greet/from/gopher
# {"message":"Hello gopher"}
```

## 2.3 生成目录

```text
greet
├── etc/greet-api.yaml        # 配置
├── greet.api                 # 接口定义
├── greet.go                  # 入口
└── internal
    ├── handler/ logic/       # 路由、handler、业务
    ├── svc/                  # 依赖容器
    └── types/                # 请求响应类型
```

# 3. API 文件

## 3.1 语法

`.api` 是 go-zero 自有的 DSL，长得像 Go 结构体加路由声明：

```go
syntax = "v1"

info (
	title:   "greet api"
	version: "v1"
)

type GreetRequest {
	Name string `path:"name,options=you|me"`
}

type GreetResponse {
	Message string `json:"message"`
}

service greet-api {
	@handler GreetHandler
	get /greet/from/:name (GreetRequest) returns (GreetResponse)
}
```

`path:"name"` 把路径变量绑定到字段；`options=you|me` 是枚举校验，不匹配直接返回 400。

## 3.2 常用标签

| 标签 | 作用 | 示例 |
|------|------|------|
| `path` | 路径参数 | `path:"id"` |
| `json` | JSON 字段名 | `json:"user_name"` |
| `optional` | 可选字段 | `json:"age,optional"` |
| `options` | 枚举 | `json:"type,options=a|b"` |

## 3.3 生成与重生成

```bash
goctl api new greet                    # 首次
goctl api go -api greet.api -dir .     # 修改 .api 后重新生成
```

⚠ **关键认知**：`goctl api go` 会**覆盖** `types.go`、`routes.go`、handler 骨架，但不会覆盖你已写好的 `logic` 文件。所以业务代码要写在 `logic` 里，不要把逻辑塞进 handler，否则每次改接口都被冲掉。

# 4. 业务逻辑

## 4.1 handler 只做转发

生成的 handler 做三件事：解析请求、调用 logic、写响应。核心代码就是：

```go
var req types.GreetRequest
if err := httpx.Parse(r, &req); err != nil {
	httpx.ErrorCtx(r.Context(), w, err)
	return
}
l := logic.NewGreetLogic(r.Context(), svcCtx)
resp, err := l.Greet(&req)
httpx.OkJsonCtx(r.Context(), w, resp) // 省略 err 分支
```

## 4.2 logic 写业务

```go
type GreetLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewGreetLogic(ctx context.Context, svcCtx *svc.ServiceContext) *GreetLogic {
	return &GreetLogic{Logger: logx.WithContext(ctx), ctx: ctx, svcCtx: svcCtx}
}

func (l *GreetLogic) Greet(req *types.GreetRequest) (*types.GreetResponse, error) {
	if req.Name == "" {
		return nil, errors.New("name 不能为空")
	}
	return &types.GreetResponse{Message: "Hello " + req.Name}, nil
}
```

## 4.3 ServiceContext

`internal/svc/servicecontext.go` 是依赖容器，数据库、Redis、RPC 客户端都挂在这里：

```go
type ServiceContext struct {
	Config config.Config
	User   userclient.User // 下游 RPC 客户端
}

func NewServiceContext(c config.Config) *ServiceContext {
	return &ServiceContext{Config: c, User: userclient.NewUser(zrpc.MustNewClient(c.UserRpc))}
}
```

★ **关键认知**：go-zero 没有依赖注入框架，靠 `ServiceContext` 手工装配。好处是依赖关系一目了然，坏处是依赖多了构造函数会变长。保持 ServiceContext 只放"基础设施客户端"，业务对象在用的时候再创建。

# 5. RPC 服务

## 5.1 生成骨架

```bash
goctl rpc new user
```

生成 `etc/`、`internal/`、`user.proto`、`user/`（pb 桩）、`userclient/`（客户端）和入口 `user.go`。

## 5.2 proto 定义

```protobuf
syntax = "proto3";

package user;

option go_package = "./user";

message GetUserRequest {
  int64 id = 1;
}

message UserReply {
  int64  id   = 1;
  string name = 2;
}

service User {
  rpc GetUser (GetUserRequest) returns (UserReply);
}
```

修改后重新生成：

```bash
goctl rpc protoc user.proto \
  --go_out=types --go-grpc_out=types --zrpc_out=.
```

## 5.3 配置与注册

`etc/user.yaml`：

```yaml
Name: user.rpc
ListenOn: 0.0.0.0:8080
Etcd:
  Hosts:
    - 127.0.0.1:2379
  Key: user.rpc
```

启动时 zrpc 会自动向 Etcd 注册，进程退出自动摘除。

## 5.4 调用方

在 api 服务的配置里声明下游，并在 `config.go` 加 `UserRpc zrpc.RpcClientConf`：

```yaml
UserRpc:
  Etcd:
    Hosts: ["127.0.0.1:2379"]
    Key: user.rpc
```

```go
func (l *GreetLogic) Greet(req *types.GreetRequest) (*types.GreetResponse, error) {
	u, err := l.svcCtx.User.GetUser(l.ctx, &user.GetUserRequest{Id: 1})
	if err != nil {
		return nil, err
	}
	return &types.GreetResponse{Message: "Hello " + u.Name}, nil
}
```

★ **关键认知**：`zrpc.MustNewClient` 内部已经做了**服务发现、负载均衡、熔断、超时**。你不需要自己 new 一个 `grpc.ClientConn`，也不需要在每次请求时重新建连。客户端应该作为长生命周期对象放在 `ServiceContext` 里。

⚠ **关键认知**：`MustNewClient` 在初始化时不会真正连上所有实例，第一次调用才建连。所以服务刚启动时第一次 RPC 可能稍慢，这属于正常现象；如果要预热，可以在启动后主动发一次健康检查。

# 6. 常用 goctl 命令

| 命令 | 作用 |
|------|------|
| `goctl api new <name>` | 新建 HTTP 服务 |
| `goctl api go -api x.api -dir .` | 从 `.api` 生成代码 |
| `goctl rpc new <name>` | 新建 RPC 服务 |
| `goctl rpc protoc x.proto --zrpc_out=.` | 从 proto 生成 RPC |

# 7. 与 Kratos 的对比

| 维度 | go-zero | Kratos |
|------|---------|--------|
| 接口描述 | `.api` + `.proto` | 统一 `.proto` |
| 代码生成 | goctl，约定强 | kratos CLI + wire，组合自由 |
| 依赖注入 | 手工 ServiceContext | Google Wire |
| 学习曲线 | 低，照生成写 | 中，要理解分层与 wire |
| 适合团队 | 追求统一规范、快速交付 | 追求架构自由度、长期演进 |

# 8. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `goctl: command not found` | GOPATH/bin 不在 PATH | 加入环境变量 |
| 改了 `.api` 路由 404 | 没重新生成 | `goctl api go -api x.api -dir .` |
| logic 被覆盖 | 把业务写进了 handler | 业务只写 logic |
| RPC 调用超时 | Etcd Key 不一致 | 两端 `Key` 必须相同 |
| 找不到 `userclient` | 没执行 rpc protoc | 重新生成客户端 |
| 配置解析失败 | yaml 字段名与结构体不符 | 检查大小写与嵌套 |

# 9. 练习

1. 给 `greet.api` 加一个 `POST /greet` 接口，请求体包含 `name` 和 `age`，返回拼接后的问候语。
2. 新建一个 `order.rpc`，提供 `CreateOrder` 方法，让 `greet-api` 调用它并返回订单号。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 约定优于配置，代码生成保证工程一致 |
| API | `.api` 定义类型与路由，goctl 生成 handler |
| 分层 | handler 只转发，logic 写业务，svc 管依赖 |
| RPC | proto + goctl 生成服务端与客户端 |
| 注册 | zrpc 内置 Etcd 注册与发现 |
| 取舍 | 规范强、上手快，灵活性弱于 Kratos |

下一篇讲服务发现与配置中心，把 Etcd、Consul 的注册模型、Kratos 与 go-zero 的接入方式、环境隔离和配置热更新串起来。
