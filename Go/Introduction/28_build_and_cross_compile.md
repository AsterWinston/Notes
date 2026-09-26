构建与交叉编译是 Go 工程化的最后一块拼图。Go 的交叉编译能力（一次编译、多平台运行）是其"一次编写、处处部署"的关键支撑。这一章讲清楚 build tags、代码生成、资源嵌入，以及交叉编译的完整原理。

# 1. 构建标签（build tags）

## 1.1 为什么要 build tags

★ **关键认知**：build tags 用于**条件编译**——根据目标平台、架构或自定义标签，决定哪些文件参与编译。典型场景：不同操作系统有不同的实现文件。

```
场景：某功能在 Linux 和 Windows 实现不同
├── util_linux.go     // 只在 Linux 编译
├── util_windows.go   // 只在 Windows 编译
└── util.go           // 通用代码
```

## 1.2 build tags 的两种写法

```go
// 写法1：文件顶部注释（注意：后面必须空一行）
//go:build linux && amd64
// +build linux,amd64        // 旧版写法（1.17 前）

package util

// 写法2：文件名后缀（最简单）
// 文件名：util_linux.go  → 只在 linux 编译
// 文件名：util_amd64.go  → 只在 amd64 编译
// 文件名：util_linux_amd64.go → linux 且 amd64
```

★ **关键认知**：Go 1.17 起统一用 `//go:build` 语法（放在文件顶部，紧跟包声明前，且**必须留一个空行**）。文件名后缀方式（`_linux.go`、`_amd64.go`）是最简单常用的方式，无需注释。

## 1.3 自定义标签

```go
//go:build integration
package test

// 编译时指定：
go test -tags=integration
```

★ **关键认知**：除了平台/架构，还能自定义标签（如 `integration`、`debug`），用 `-tags` 参数激活。常用于区分"单元测试 vs 集成测试"、或"正式版 vs 调试版"。

# 2. 代码生成：go:generate

## 2.1 基本用法

```go
// 在源文件中写注释：
//go:generate stringer -type=Status

type Status int

const (
    StatusPending Status = iota
    StatusActive
    StatusDone
)
```

```bash
go generate ./...    # 执行所有 go:generate 指令
```

★ **关键认知**：`go:generate` 是**代码生成**的入口——执行注释中指定的命令，自动生成代码。上例用 `stringer` 工具为 Status 类型生成 String() 方法。它是"用编译期成本换运行时性能"（替代反射）的核心手段。

## 2.2 代码生成 vs 反射

★ **关键认知**：代码生成是反射的高性能替代方案（见反射章节）：

```
反射：运行时解析类型，灵活但慢
代码生成：编译期生成专用代码，零运行时开销，但需额外构建步骤
```

典型应用：`easyjson`（生成专用 JSON 序列化代码）、`gorm`（生成模型代码）、`stringer`（生成 String 方法）。

# 3. 资源嵌入：go:embed

## 3.1 基本用法

```go
import _ "embed"

//go:embed config.json
var config string              // 嵌入为字符串

//go:embed templates/*
var templates embed.FS         // 嵌入为文件系统
```

★ **关键认知**：`go:embed`（Go 1.16+）把外部文件**编译进二进制**，运行时无需外部文件依赖。这对"单文件部署"意义重大——模板、配置文件、静态资源都能打包进可执行文件：

```
支持嵌入：
- 单个文件（嵌入为 string 或 []byte）
- 多个文件/目录（嵌入为 embed.FS）
- 支持通配符和模式匹配
```

# 4. 交叉编译

## 4.1 原理

★ **关键认知**：Go 交叉编译的核心是 `GOOS` 和 `GOARCH` 两个环境变量——它们指定**目标**操作系统和架构，让编译器生成对应平台的机器码：

```bash
# 在 Windows 上编译 Linux 程序
GOOS=linux GOARCH=amd64 go build -o app-linux main.go

# 在 Linux 上编译 Windows 程序
GOOS=windows GOARCH=amd64 go build -o app.exe main.go

# 编译 macOS 程序
GOOS=darwin GOARCH=arm64 go build -o app-mac main.go
```

## 4.2 支持的平台

```
GOOS:    linux, windows, darwin, freebsd, android, js, ...
GOARCH:  amd64, arm64, 386, arm, mips, riscv64, ...

组合示例：
linux/amd64   Windows/amd64   darwin/arm64(M1 Mac)   linux/arm64
```

★ **关键认知**：Go 之所以能轻松交叉编译，是因为它的工具链**内置了所有目标平台的代码生成器**，且运行时用 Go 编写（不依赖目标平台）。只需两个环境变量，无需像 C 那样配置交叉工具链。

## 4.3 cgo 的限制

★ **关键认知**：**纯 Go** 代码可任意交叉编译，但**用了 cgo** 的代码交叉编译很困难：

```
CGO_ENABLED=0   禁用 cgo（纯 Go），可自由交叉编译
CGO_ENABLED=1   启用 cgo，交叉编译需要目标平台的 C 交叉编译器
```

★ **关键认知**：需要交叉编译的项目应尽量避免 cgo（如 `net` 包用纯 Go 解析器 `netgo`、`os/user` 用 `osusergo`）。这是"跨平台部署优先"还是"用 C 库优先"的取舍。

# 5. 构建优化与发布

## 5.1 减小二进制体积

```bash
go build -ldflags="-s -w"        # 去掉符号表和调试信息
go build -trimpath              # 去掉编译路径（可复现构建）
```

```
-s   去除符号表
-w   去除 DWARF 调试信息
组合使用可减小 30%~50% 的体积
```

## 5.2 可复现构建

★ **关键认知**：可复现构建（reproducible build）指"相同源码永远产出字节级相同的二进制"。关键参数：

```
1. -trimpath：去掉编译机绝对路径
2. 固定 CGO_ENABLED、GOOS、GOARCH
3. 固定 Go 版本（或使用 GOTOOLCHAIN 锁定）
```

这对供应链安全（验证构建产物未被篡改）至关重要。

## 5.3 容器化部署

```
Go 部署的黄金搭档是 Docker 多阶段构建：

FROM golang:1.23 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o app

FROM scratch              # 或 alpine
COPY --from=builder /app/app /app
ENTRYPOINT ["/app"]
```

★ **关键认知**：Go 的静态链接让它能用 `scratch`（空镜像）或 `alpine` 部署，最终镜像可小到几 MB。这是 Go 在云原生/微服务领域碾压性优势的来源。

# 6. 构建缓存与提速

★ **关键认知**：Go 的构建缓存（GOCACHE）让增量编译极快：

```
1. 每个包按"源码哈希 + 编译参数"缓存产物
2. 未变化的包直接复用缓存，不重编
3. 配合多核并行编译（默认自动），大型项目编译秒级完成
```

```
加速技巧：
1. 保持依赖精简（减少编译包数量）
2. 用 go build ./... 而非重复单文件编译
3. 合理使用 GOMAXPROCS（默认自动）
```

# 7. 小结

| 概念 | 要点 |
|------|------|
| build tags | 条件编译，//go:build 或文件名后缀 |
| go:generate | 代码生成入口 |
| go:embed | 资源嵌入二进制 |
| GOOS/GOARCH | 交叉编译目标 |
| CGO_ENABLED | cgo 影响交叉编译 |
| -ldflags -s -w | 减小体积 |
| -trimpath | 可复现构建 |

至此，Go 语言从入门到深入的完整学习路径已经走完。从环境配置、基础语法，到接口、并发、反射，再到运行时、GC、内存分配、编译器优化，你已经建立了对 Go 的完整认知体系。
