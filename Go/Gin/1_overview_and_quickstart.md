Gin 是 Go 生态中最常用的 HTTP Web 框架，适合快速构建 REST API 和中小型服务。本篇从零跑起第一个服务，解释 Gin 解决的问题、项目如何组织，以及初学者最容易卡住的环境与排错步骤。

# 1. 为什么需要 Web 框架

## 1.1 net/http 的能力与不足

Go 标准库 `net/http` 已经能提供完整的 HTTP 服务端能力。一个最小服务只需要下面几行：

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "hello") // 直接向响应写入文本
    })

    http.ListenAndServe(":8080", nil) // 监听 8080 端口
}
```

标准库足以写服务，但真实项目很快会遇到重复劳动：

| 需求 | 只用 net/http 时的做法 | 问题 |
|------|------------------------|------|
| 路径参数 | 自己解析 `r.URL.Path` | 容易写错，缺少类型转换 |
| 请求参数绑定 | 手动读 Query、Form、JSON | 代码重复，错误处理分散 |
| 中间件 | 自己包装 Handler | 嵌套深，顺序不直观 |
| 统一 JSON 响应 | 每个 Handler 手写编码 | 状态码和结构容易不一致 |
| 路由分组 | 自己拼接前缀 | 版本管理混乱 |

★ **关键认知**：框架不是替代标准库，而是把"每个项目都会重复写一遍"的部分封装起来。理解 `net/http` 仍然重要，因为 Gin 的底层就是它。

## 1.2 Gin 的定位与适用场景

Gin 的核心优势是**路由性能好、API 简洁、中间件模型清晰**。它适合：

- REST API、后台管理接口、微服务中的 HTTP 服务。
- 需要快速开发、同时保持较好性能的项目。
- 已经熟悉 Go 基础语法，想学习 Web 工程化的开发者。

它不一定适合：

- 只需要一两个静态页面的极简服务，标准库更轻。
- 需要大量内置企业能力（如完整 ORM、依赖注入）的项目，可能需要组合其他库。
- 对框架抽象极度敏感、希望完全掌控底层链路的项目。

| 框架 | 特点 | 适合场景 |
|------|------|----------|
| `net/http` | 标准库、零依赖 | 学习、极小服务、完全自定义 |
| Gin | 路由快、生态成熟 | REST API、常规后端服务 |
| Echo | API 设计与 Gin 类似 | 需要更多内置组件的项目 |
| Fiber | 基于 fasthttp，API 风格不同 | 追求吞吐、能接受生态差异 |

# 2. 环境准备

## 2.1 Go 版本与模块初始化

先确认 Go 版本。Gin 的新版本通常要求较新的 Go，学习阶段建议使用 Go 1.21 或更高版本：

```bash
go version
```

创建项目目录并初始化模块：

```bash
mkdir gin-demo
cd gin-demo
go mod init example.com/gin-demo
```

模块名不需要是真实域名，但建议使用能表达项目归属的写法。初始化后会生成 `go.mod`：

```text
module example.com/gin-demo

go 1.21
```

## 2.2 安装 Gin

```bash
go get github.com/gin-gonic/gin
```

安装完成后，`go.mod` 会增加依赖，项目根目录会出现 `go.sum`。如果国内网络下载缓慢，可以配置代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

★ **关键认知**：`go get` 只负责把依赖写入模块并下载到本地缓存，真正编译时 `go build` 或 `go run` 会读取 `go.mod` 和 `go.sum`。不要手动编辑 `go.sum`。

# 3. 最小可运行示例

## 3.1 第一个 Hello World

在项目根目录创建 `main.go`：

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default() // 创建引擎，并启用日志与恢复中间件

    r.GET("/hello", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{ // 返回 JSON，状态码 200
            "message": "hello gin",
        })
    })

    r.Run(":8080") // 监听 8080 端口；不写参数时默认也是 :8080
}
```

运行：

```bash
go run main.go
```

看到类似下面的日志，说明服务已经启动：

```text
[GIN-debug] GET    /hello --> main.main.func1 (3 handlers)
[GIN-debug] Listening and serving HTTP on :8080
```

## 3.2 代码逐行解释

- `gin.Default()`：创建一个 `*gin.Engine`，并自动挂载 Logger 和 Recovery 两个中间件。学习阶段可以直接用；生产环境有时会改用 `gin.New()` 再按需挂载。
- `r.GET("/hello", handler)`：注册一个只响应 GET 方法的路由。同一个路径注册不同方法时，会分别进入不同的处理函数。
- `c *gin.Context`：一次请求的上下文，负责读取请求、写入响应、传递中间件数据。它由 Gin 复用，不能跨 goroutine 长期持有。
- `c.JSON(status, obj)`：把对象序列化为 JSON 并写入响应，同时设置 `Content-Type: application/json; charset=utf-8`。
- `gin.H`：本质是 `map[string]any`，适合临时构造小响应；正式项目建议定义结构体，字段更稳定。
- `r.Run(":8080")`：内部创建 `http.Server` 并监听地址。启动失败会返回错误，正式项目应检查返回值。

## 3.3 启动与验证

用 curl 验证：

```bash
curl http://localhost:8080/hello
```

返回：

```json
{"message":"hello gin"}
```

也可以直接用浏览器访问。若返回 404，先检查路径大小写和是否带尾斜杠；若连接被拒绝，先确认服务是否真的启动成功。

# 4. 项目目录建议

## 4.1 学习期单文件结构

初学阶段不要急着分层。先把路由、Handler、参数绑定跑通，单文件就足够：

```text
gin-demo/
├── go.mod
├── go.sum
└── main.go
```

当 `main.go` 超过两三百行，或者开始出现数据库、配置、鉴权时，再拆分。

## 4.2 工程化目录结构

一个常见的演进方向如下：

```text
gin-demo/
├── cmd/server/main.go        # 程序入口
├── internal/
│   ├── router/               # 路由注册
│   ├── handler/              # HTTP 处理层
│   ├── service/              # 业务逻辑
│   ├── repository/           # 数据访问
│   └── middleware/           # 中间件
├── config/                   # 配置文件
├── go.mod
└── go.sum
```

★ **关键认知**：分层的目的是隔离变化，不是追求目录好看。Handler 只负责解析请求和返回响应；业务规则放 service；数据库细节放 repository。小项目强行分层反而增加阅读成本。

# 5. 常见错误与排错

| 现象 | 常见原因 | 解决方式 |
|------|----------|----------|
| `go get` 超时 | 网络或代理问题 | 配置 `GOPROXY` 后重试 |
| 启动报端口占用 | 8080 已被其他程序使用 | 换端口或结束占用进程 |
| 访问返回 404 | 路径不匹配、方法不匹配 | 检查注册路径和请求方法 |
| `undefined: gin` | 依赖未下载或导入路径错误 | 确认 `go.mod` 中有 Gin，执行 `go mod tidy` |
| 修改代码后行为没变 | 旧进程仍在运行 | 停止旧进程后重新启动 |

# 6. 动手练习

1. 新增 `/health` 接口，返回 `{"status":"ok"}`，并用 curl 验证。
2. 把服务端口改成 `9090`，观察启动日志和访问地址的变化。
3. 注册 `POST /echo`，先不处理请求体，只返回固定 JSON，熟悉方法注册。

# 7. 小结

| 主题 | 要点 |
|------|------|
| Gin 的定位 | 基于 `net/http` 的 Web 框架，简化路由、参数、中间件和响应 |
| 最小流程 | 初始化模块 → 安装 Gin → 创建 Engine → 注册路由 → Run |
| 学习顺序 | 先把单文件跑通，再逐步引入分层 |
| 排错思路 | 先看启动日志，再看路由注册，最后看请求方法与路径 |

下一篇进入路由，重点讲清路径参数、查询参数、路由分组和匹配优先级。
