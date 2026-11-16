gRPC 是 Go 微服务里最常见的服务间通信方案，但它并不是"更快的 HTTP 接口"这么简单。本篇先把 RPC 与 REST 的定位讲清楚，再落到 HTTP/2 这条地基上，最后用一个能跑通的示例走完 Protocol Buffers 的编写与编译流程。

# 1. 从 RPC 到 gRPC

## 1.1 什么是 RPC

**RPC（Remote Procedure Call，远程过程调用）** 的目标是让调用远程服务"看起来像调用本地函数"：

```go
// 本地调用
user, err := repo.GetUser(ctx, 42)

// RPC 调用：语法几乎一样，但背后是网络请求
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
```

调用方只需要关心方法名、参数和返回值，序列化、连接管理、超时传递这些脏活由框架完成。

★ **关键认知**：RPC 的核心抽象是**接口契约**。客户端和服务端共享同一份 `.proto` 或 `.thrift` 定义，编译器据此生成两边的桩代码（stub）。契约一旦确定，双方可以独立开发、独立发布，不必像 REST 那样靠文档口头约定字段。

## 1.2 RPC 与 REST 的对比

REST 不是协议，而是一种**以资源为中心的架构风格**；gRPC 是**以方法调用为中心的框架**。两者解决的问题重叠，但侧重点不同：

| 维度 | REST over HTTP/JSON | gRPC |
|------|---------------------|------|
| 抽象模型 | 资源 + 动词（GET /users/42） | 服务 + 方法（GetUser） |
| 传输格式 | JSON 文本 | Protobuf 二进制 |
| 传输协议 | HTTP/1.1 或 HTTP/2 | HTTP/2（必须） |
| 接口契约 | OpenAPI/Swagger，通常后补 | `.proto` 强约束，先定契约 |
| 浏览器支持 | 原生支持 | 需要 grpc-web 或网关转码 |
| 流式能力 | 弱（SSE、WebSocket 另说） | 原生四种流 |
| 可读性 | 抓包直接看 | 需要工具解码 |
| 典型场景 | 对外 API、前后端交互 | 内部服务、低延迟、强类型 |

★ **关键认知**：选型的分界线通常是**调用方是不是浏览器**。面向公网、需要被任意客户端调用的接口，REST/JSON 的通用性无可替代；而服务之间的东西向流量，gRPC 的类型安全、性能和流式能力更划算。两者不是替代关系，生产系统里常常是"边缘 REST，内部 gRPC"。

## 1.3 gRPC 的组成

```text
┌──────────────────────────────────────────────┐
│                 应用代码                      │
├──────────────────────────────────────────────┤
│  生成代码（stub）：Client / Server 接口        │
├──────────────────────────────────────────────┤
│  gRPC 运行时：编解码、拦截器、负载均衡、重试     │
├──────────────────────────────────────────────┤
│  HTTP/2：多路复用、头部压缩、流控、Trailers     │
├──────────────────────────────────────────────┤
│  TCP / TLS                                    │
└──────────────────────────────────────────────┘
```

一次 gRPC 调用在 HTTP/2 上表现为：

```text
POST /helloworld.Greeter/SayHello HTTP/2
content-type: application/grpc
te: trailers
grpc-timeout: 1S
<length-prefixed protobuf body>
```

响应分为 **Headers + 若干 DATA 帧 + Trailers**，`grpc-status` 放在 Trailers 里。这一点很关键：HTTP 状态码通常都是 200，真正的业务结果要看 `grpc-status`。

# 2. HTTP/2 是 gRPC 的地基

## 2.1 二进制分帧与多路复用

HTTP/1.1 的报文是文本，一条连接同一时刻只能处理一个请求。HTTP/2 把报文拆成**帧（frame）**，每个帧带有 `stream id`，同一条 TCP 连接上可以交错发送多个流：

```text
一条 TCP 连接
┌─────────────────────────────────────────────┐
│ stream 1: HEADERS ─ DATA ─ DATA ─ END       │
│ stream 3:   HEADERS ─ DATA ─ END            │
│ stream 5: HEADERS ─ DATA ─ DATA ─ DATA ─ END│
└─────────────────────────────────────────────┘
     帧按 stream id 归属，接收端重组
```

★ **关键认知**：gRPC 的四种调用方式能成立，靠的就是 HTTP/2 的"流"概念。一个 gRPC 调用占一个 stream，客户端流、服务端流、双向流都只是在同一个 stream 上多次收发 DATA 帧，而不是建立多条连接。

## 2.2 gRPC 依赖的 HTTP/2 特性

| 特性 | gRPC 的用途 |
|------|-------------|
| 多路复用 | 单连接并发大量 RPC，避免连接数爆炸 |
| 头部压缩（HPACK） | 重复的 `content-type`、`:path` 只传一次 |
| 流控 | 防止发送方压垮接收方 |
| Trailers | 在响应体之后携带 `grpc-status`、`grpc-message` |
| 取消（RST_STREAM） | 客户端取消时精确终止单个调用，不影响同连接其他流 |

⚠ **关键认知**：很多"gRPC 连不上"的问题其实是中间代理只支持 HTTP/1.1。Nginx 需要 `grpc_pass`，Envoy 需要明确配置 HTTP/2 upstream，传统七层负载均衡可能直接破坏帧格式。上线前先确认链路上每一跳都懂 HTTP/2。

# 3. Protocol Buffers 基础

## 3.1 第一个 proto 文件

新建 `proto/hello.proto`：

```protobuf
syntax = "proto3";                 // 必须声明版本，否则默认 proto2

package hello.v1;                  // 包名，参与生成代码与全限定方法名

option go_package = "example.com/grpcdemo/gen;hellov1";
// go_package 告诉 protoc-gen-go 生成到哪个导入路径，分号后是包别名

message HelloRequest {
  string name = 1;                 // 字段编号 1，一旦上线不可随意改
}

message HelloReply {
  string message = 1;
}

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}
```

★ **关键认知**：`.proto` 是**接口的唯一事实来源**。服务端和客户端都从它生成代码，字段名、类型、编号完全一致。编号才是线上传输的标识，字段名只影响生成代码，改名不会破坏兼容性，改编号会。

## 3.2 字段编号与类型

字段编号（field number）的规则：

- 范围 `1 ~ 536870911`（2^29 - 1）。
- `19000 ~ 19999` 被 Protobuf 内部保留，不能使用。
- `1 ~ 15` 编码后只占 1 个字节的 tag，适合高频字段。
- 编号一旦发布就**不能复用**，删除字段时应把编号 `reserved` 掉。

常用标量类型与 Go 的映射：

| proto 类型 | Go 类型 | 说明 |
|------------|---------|------|
| `double` / `float` | `float64` / `float32` | 浮点 |
| `int32` / `int64` | `int32` / `int64` | 变长编码，负数效率低 |
| `uint32` / `uint64` | `uint32` / `uint64` | 无符号 |
| `sint32` / `sint64` | `int32` / `int64` | ZigZag 编码，适合负数 |
| `fixed32` / `fixed64` | `uint32` / `uint64` | 定长，适合大整数 |
| `bool` | `bool` | 布尔 |
| `string` | `string` | 必须是 UTF-8 |
| `bytes` | `[]byte` | 任意字节 |

⚠ **关键认知**：`int32` 存负数时会编码成 10 字节（补码按 64 位处理），如果字段经常为负，应该用 `sint32`/`sint64`。这是 Protobuf 编码里最容易忽略的性能细节。

## 3.3 proto3 的变化

| 特性 | proto2 | proto3 |
|------|--------|--------|
| 字段存在性 | 默认有 `has_xxx` | 标量默认无，需 `optional` 显式开启 |
| 默认值 | 可自定义 | 只能是零值，不能自定义 |
| `required` | 支持 | 已移除 |
| 枚举首值 | 任意 | 必须为 0 |
| 未知字段 | 部分丢弃 | 保留 |

★ **关键认知**：proto3 里普通标量字段**无法区分"没传"和"传了零值"**。`int32 age = 1` 收到 0 时，你不知道对方是没填还是真的 18 岁减 18。需要区分时加 `optional`（生成指针或 `HasAge()`），或者用 `google.protobuf.Int32Value` 这类 wrapper。

# 4. proto 编译流程

## 4.1 安装工具链

```bash
# 1. 安装 protoc 编译器（以 Windows + winget 为例，其他平台用包管理器）
winget install protobuf

# 2. 安装 Go 代码生成插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 3. 确认插件在 PATH 中
protoc-gen-go --version
protoc-gen-go-grpc --version
```

三个工具的分工：

```text
hello.proto
    │
    ▼
protoc（解析 + 校验）
    │  调用插件
    ├── protoc-gen-go      → hello.pb.go      消息结构体、序列化
    └── protoc-gen-go-grpc → hello_grpc.pb.go 客户端/服务端接口
```

## 4.2 生成 Go 代码

在项目根目录执行：

```bash
protoc --proto_path=proto \
  --go_out=gen --go_opt=paths=source_relative \
  --go-grpc_out=gen --go-grpc_opt=paths=source_relative \
  proto/hello.proto
```

参数说明：

- `--proto_path`：import 的搜索根目录，可以写多次。
- `--go_out`：消息代码输出目录。
- `--go-grpc_out`：服务代码输出目录。
- `paths=source_relative`：按 proto 的相对路径生成，避免目录结构错乱。

生成结果：

```text
gen/
└── hello.pb.go        # HelloRequest、HelloReply 及其编解码
└── hello_grpc.pb.go   # GreeterClient、GreeterServer、注册函数
```

★ **关键认知**：生成的 `.pb.go` **不要手改**，每次编译都会覆盖。需要扩展行为时用 Go 的包装类型或在服务实现里做，而不是往生成文件里塞方法。

## 4.3 用 buf 管理

项目变大后，手写 `protoc` 命令会失控。`buf` 提供格式化、lint、breaking change 检测和依赖管理：

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-go
    out: gen
    opt: paths=source_relative
  - local: protoc-gen-go-grpc
    out: gen
    opt: paths=source_relative
```

```bash
buf lint          # 检查命名与编号规范
buf generate      # 按 buf.gen.yaml 生成代码
buf breaking --against '.git#branch=main'  # 检测不兼容改动
```

# 5. 最小可运行示例

## 5.1 项目结构

```text
grpcdemo/
├── go.mod
├── proto/hello.proto
├── gen/hello.pb.go
├── gen/hello_grpc.pb.go
├── server/main.go
└── client/main.go
```

```bash
go mod init example.com/grpcdemo
go get google.golang.org/grpc@latest
go get google.golang.org/protobuf@latest
```

## 5.2 服务端

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"

    hellov1 "example.com/grpcdemo/gen"
)

type server struct {
    hellov1.UnimplementedGreeterServer // 必须内嵌，保证前向兼容
}

func (s *server) SayHello(ctx context.Context, req *hellov1.HelloRequest) (*hellov1.HelloReply, error) {
    return &hellov1.HelloReply{Message: "hello " + req.GetName()}, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("listen: %v", err)
    }

    s := grpc.NewServer()                          // 创建 gRPC 服务端
    hellov1.RegisterGreeterServer(s, &server{})    // 注册服务实现

    log.Println("gRPC server listening on :50051")
    if err := s.Serve(lis); err != nil {           // 阻塞直到退出
        log.Fatalf("serve: %v", err)
    }
}
```

## 5.3 客户端

```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    hellov1 "example.com/grpcdemo/gen"
)

func main() {
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()), // 明文，仅本地调试
    )
    if err != nil {
        log.Fatalf("dial: %v", err)
    }
    defer conn.Close()

    client := hellov1.NewGreeterClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    resp, err := client.SayHello(ctx, &hellov1.HelloRequest{Name: "gopher"})
    if err != nil {
        log.Fatalf("SayHello: %v", err)
    }
    log.Println(resp.GetMessage())
}
```

运行：

```bash
go run ./server &
go run ./client
# 输出：hello gopher
```

# 6. 常见错误与排错

| 现象 | 常见原因 | 解决方式 |
|------|----------|----------|
| `protoc-gen-go: program not found` | 插件不在 PATH | 把 `$(go env GOPATH)/bin` 加入 PATH |
| `unimplemented` 错误 | 服务端没内嵌 `UnimplementedXxxServer` 或没注册 | 检查 `RegisterXxxServer` 与内嵌 |
| `connection refused` | 服务端未启动或端口不对 | 先用 `netstat`/`telnet` 验证端口 |
| `transport: authentication handshake failed` | 客户端用了 TLS，服务端是明文 | 两边凭据配置保持一致 |
| `code = Unavailable desc = ...` | 连接断开、服务未就绪 | 检查服务端日志与网络 |
| 编译报重复注册 | 同名 proto 被多个包生成 | 统一 `go_package` 与 import 路径 |

⚠ **关键认知**：`grpc.NewClient` 是"惰性连接"，它不会立刻建连，第一次 RPC 才真正握手。所以 dial 成功不代表服务可用，必须用 `conn.Connect()` 或实际调用来验证。

# 7. 练习与自检

1. 在 `hello.proto` 里新增 `int32 age = 2`，重新生成代码，观察 `GetAge()` 方法以及零值行为。
2. 把 `SayHello` 改成返回当前时间戳，练习 `google.protobuf.Timestamp` 的 import 与生成。
3. 故意把客户端端口改成 `50052`，观察错误码是 `Unavailable` 还是别的，并解释原因。

# 8. 小结

| 主题 | 要点 |
|------|------|
| RPC 定位 | 以方法调用为中心，共享 `.proto` 契约，适合内部服务 |
| HTTP/2 | 多路复用、HPACK、Trailers 是 gRPC 四种流的基础 |
| Protobuf | 编号是线上标识，字段名只影响代码；负数用 `sint` |
| 编译链 | `protoc` 调 `protoc-gen-go` 与 `protoc-gen-go-grpc` |
| 最小闭环 | 定义 proto → 生成代码 → 注册服务 → 客户端调用 |

下一篇进入服务定义，系统讲字段编号规则、`optional`/`repeated`/`map`/`oneof`、枚举、包名版本化，以及向后兼容的取舍。
