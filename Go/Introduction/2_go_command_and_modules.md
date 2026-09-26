`go` 是一个"瑞士军刀"式的命令行工具：它同时承担了编译、依赖管理、测试、格式化、文档等职责。理解它的设计哲学——**一个命令统一所有环节**——比背每个子命令的用法更重要。本章重点讲构建与模块管理，测试与性能分析留到专门章节。

# 1. go 命令体系总览

```
go <command> [arguments]

构建相关：  go build / go run / go install / go clean
依赖相关：  go mod init / go mod tidy / go mod download / go get / go mod vendor
测试相关：  go test / go vet / go test -bench / go test -race / go test -fuzz
代码质量：  go fmt / gofmt / go vet / go doc
包管理：    go list / go env / go version / go tool
```

★ **关键认知**：`go` 命令本身的实现位于 `GOROOT/src/cmd/go/`，而它调用的编译器、汇编器、链接器在 `GOROOT/src/cmd/compile/`、`cmd/asm/`、`cmd/link/` 下，通过 `go tool` 子命令可以直接访问这些底层工具（见汇编章节）。

# 2. 编译四部曲：go build / run / install / clean

## 2.1 go build —— 编译

```go
// 在任意 module 目录下
go build             # 编译当前目录的包，生成可执行文件（若 package main）
go build ./...       # 编译当前 module 下所有包
go build -o myapp    # 指定输出文件名
go build -gcflags="-m"      # 打开编译优化日志（逃逸分析）
go build -ldflags="-s -w"   # 链接期去除符号表与 DWARF 调试信息，减小体积
go build -tags=integration  # 按构建标签选择性编译（见构建章节）
go build -race              # 开启竞态检测（见测试章节）
go build -trimpath          # 去掉编译机路径，便于可复现构建
```

★ **关键认知**：`go build` 的产物默认不进入 `GOBIN`，而是输出到当前目录；`go build` 编译 `package main` 才产出可执行文件，编译普通包只做类型检查与编译缓存，不产生独立产物。

### 2.1.1 增量编译与构建缓存

★ **关键认知**：Go 的编译是**增量的**。每个包编译后，其产物（`.a` 归档 + 依赖元数据）按"源码内容哈希 + 编译参数"存入 `GOCACHE`。第二次编译时，只有变化包及其下游需要重编，未变化的包直接复用缓存——这是 Go 大型项目编译快的关键，本质上是编译器层面的"内容寻址缓存"。

## 2.2 go run —— 编译并运行

```
go run main.go         # 编译 + 运行，产物放临时目录，运行后删除
go run .               # 运行当前目录的 main 包
go run main.go a b     # a b 会作为 os.Args 传给程序
```

★ **关键认知**：`go run` 不是解释执行。它内部走的是"编译到临时目录 → 执行 → 清理"的完整流程。可以理解为 `go build` 到一个临时位置然后立刻运行。调试期方便，但正式部署一律用 `go build` 产物。

## 2.3 go install —— 编译并安装

```
go install            # 编译当前包，把可执行文件放进 GOBIN（默认 $GOPATH/bin）
go install golang.org/x/tools/cmd/gofumpt@latest   # 从远程安装一个工具
```

★ **关键认知**：`go build` 与 `go install` 的唯一区别在于产物落点——`install` 把二进制放入 `GOBIN`。因此安装第三方 CLI 工具（如 `gofumpt`、`staticcheck`）的标准姿势是 `go install xxx@latest`。

## 2.4 go clean —— 清理缓存

```
go clean              # 清理构建缓存
go clean -cache       # 清理 GOCACHE
go clean -modcache    # ★ 清理整个模块缓存（GOMODCACHE），慎用
```

# 3. 模块机制：go.mod 与 go.sum

Go module 是 Go 1.11 引入、1.16 全面启用的依赖管理方案，取代了 GOPATH 时代混乱的 `vendor` + 版本管理。

## 3.1 初始化与语义化版本

```
go mod init github.com/you/yourproject    # 初始化 module
```

生成的 `go.mod` 结构：

```
module github.com/you/yourproject    // module 路径，等价于命名空间

go 1.23                              // ★ 声明该模块的 Go 版本（GOTOOLCHAIN 据此切换工具链）

require (
    github.com/gin-gonic/gin v1.10.0      // 直接依赖 + 版本
    golang.org/x/sync v0.9.0
)

require (                              // 间接依赖（第三方依赖又依赖的包）
    github.com/bytedance/sonic v1.12.0 // indirect
)
```

### 3.1.1 go.mod 的全部指令

```
module <path>         模块路径
go <version>          该模块使用的 Go 语言版本
require <mod> <ver>   依赖声明（可分组）
replace <mod> => <target>  依赖替换（本地路径或另一个版本）
exclude <mod> <ver>   排除某个特定版本
retract [<ver>]       声明已发布版本有严重问题，应回退
toolchain <name>      建议的工具链版本
```

★ **关键认知**：Go 的版本号遵循语义化版本（SemVer，`v主.次.补丁`）。Go 官方引入的"最小版本选择（MVS）"算法，是 Go module 与其他语言（npm、Maven）最大的设计差异——Go 不是选"最新版本"，而是选"能满足所有约束的最低版本"。

## 3.2 最小版本选择（MVS）—— 深入理解

```
假设：
A 依赖 B v1.3.0
C 依赖 B v1.5.0
项目同时依赖 A 和 C

npm/Maven 策略：可能选 B 的最新版（或最靠近根依赖的版本）
Go MVS 策略：  选 v1.5.0（满足"A>=v1.3.0"和"C>=v1.5.0"两个约束的最低版本）

用图表示（选各约束下界的最大值，而非全局最新）：
        B v1.3.0 ── A 要求 >= 1.3.0
        B v1.5.0 ── C 要求 >= 1.5.0
        ────────────
        结果：B v1.5.0（所有约束下界的最大值，可复现）
```

### 3.2.1 为什么 MVS 不选最新版

★ **关键认知**：MVS 的核心价值是**确定性**——同样的 `go.mod` 永远解析出同样的版本，不受"先下载了谁"的影响。这是为了满足大规模团队协作与可复现构建的硬需求。代价是它不会主动升级到更新版本，需要你手动 `go get -u` 或编辑版本号。这种"保守"是刻意的取舍：可复现性优先于"追新"。

### 3.2.2 模块图裁剪（graph pruning）

★ **关键认知**：Go 1.17 起，`go.mod` 会记录完整的间接依赖列表（模块图被"裁剪"），这样 `go mod tidy` 和依赖解析不再需要下载所有依赖的完整 `go.mod` 链，大幅提升大型项目的依赖解析速度。这就是上面示例里 `require` 块同时包含直接依赖和 `// indirect` 间接依赖的原因。

## 3.3 go.sum —— 校验和清单

```
go.sum 记录每个依赖（module + 版本 + go.mod 文件）的哈希，用于完整性校验：

github.com/gin-gonic/gin v1.10.0 h1:XXX...      // 模块 zip 包的哈希（h1:）
github.com/gin-gonic/gin v1.10.0/go.mod h1:YYY... // go.mod 文件的哈希
```

★ **关键认知**：`go.sum` 要提交到版本库（不是临时文件）。它配合 `GOSUMDB` 实现供应链安全——构建时逐条比对哈希，任何依赖内容被篡改都会导致构建失败。`h1:` 前缀表示使用 SHA-256 哈希。

## 3.4 依赖操作命令

```
go mod tidy        # ★ 最常用：拉取缺失依赖、移除多余依赖、更新 go.sum
go mod download    # 只下载依赖到 GOMODCACHE，不修改 go.mod
go get pkg@v1.2.3  # 添加/升级/降级依赖
go get pkg@latest  # 升级到最新
go get -u          # 升级所有直接依赖
go mod verify      # 校验缓存中的依赖与 go.sum 是否一致
go mod graph       # 打印依赖图
go mod vendor      # 把依赖复制到 ./vendor 目录（离线构建）
```

### 3.4.1 go get 的版本语法

```
go get pkg               # 取该依赖当前声明的版本
go get pkg@v1.2.3        # 精确版本
go get pkg@latest        # 最新版本
go get pkg@upgrade       # 升级到更高版本
go get pkg@patch         # 升级到更高补丁版本
go get pkg@none          # ★ 移除该依赖
```

★ **关键认知**：依赖真实存放位置是 `GOMODCACHE`（默认 `$GOPATH/pkg/mod`），路径形如 `github.com/!gin-gonic/gin@v1.10.0`（大写字母会转义为 `!x` 形式，因为 Windows/macOS 文件系统不区分大小写）。源码引用这些路径时可以直接点进去读，是研究第三方库实现的入口。

### 3.4.2 vendor 模式

```
go mod vendor     # 生成 ./vendor 目录，把依赖源码复制进来

当项目根存在 vendor 目录，且 go 版本 >= 1.14 时，go build 会默认使用 vendor 目录，不联网。
```

★ **关键认知**：`vendor` 是"自带依赖"的离线构建方案，适合需要完全离线、或对供应链有极严格审计要求的场景。现代 Go 项目更常用 `GOMODCACHE` + `go.sum` 而非 vendor，但某些企业 CI 仍强制 vendor。

# 4. 工作区（Go Workspace，Go 1.18+）

## 4.1 初始化与使用

```
go work init ./moduleA ./moduleB    # 创建 go.work
```

`go.work` 让你在**本地同时开发多个 module** 时，无需反复 `replace`，工作区内的模块互相引用会直接指向本地源码：

```
go 1.23

use (
    ./service-a        // 本地模块 A
    ./shared-lib       // 本地模块 B
)

replace example.com/dep => ../dep-local   // 也可以本地替换
```

★ **关键认知**：`go.work` 是**纯本地开发工具**，不应提交到版本库（`.gitignore` 里通常排除）。它解决了"改库代码 → 立刻在依赖它的服务里生效"的联调痛点，是 `replace` 的现代化替代。

# 5. 包与导入路径

```
import 的路径 = module 路径 + 模块内相对目录路径

module github.com/you/app
├── go.mod
├── main.go              // package main，import "github.com/you/app/internal/util"
└── internal/
    └── util/
        └── util.go      // package util
```

## 5.1 internal 目录的访问控制

★ **关键认知**：Go 没有"相对导入"，import 永远是完整的模块路径。`internal` 目录有特殊语义——只有 `internal` 的**父目录树内**的包能导入它，外部包无法导入。这是 Go 语言层面提供的"仅内部可见"边界，用于防止公开包的内部实现被外部依赖，是模块化的核心手段。

# 6. 代码质量三件套：fmt / vet / doc

## 6.1 go fmt / gofmt

```
go fmt ./...      # 格式化代码（底层调用 gofmt -w）
gofmt -w file.go  # 单个文件格式化
gofmt -d file.go  # 只打印差异，不写文件
gofmt -l file.go  # 列出格式不规范的代码
```

★ **关键认知**：`go fmt` 作用于包、`gofmt` 作用于文件。Go 的格式化是**强制的、唯一的**——`gofmt` 输出是标准，不存在"风格之争"，这是 Go 社区能保持代码风格高度统一的技术手段（没有缩进用空格还是 Tab 的争论，统一 Tab）。

## 6.2 go vet —— 静态检查

```
go vet ./...      # 静态检查：可疑结构、Printf 参数不匹配、锁拷贝等
```

★ **关键认知**：`go vet` 属于"编译器不会报错但很可能是 bug"的启发式检查。典型如 `fmt.Printf("%d", "string")` 这种格式串与参数不匹配、`sync.Mutex` 被按值拷贝等。它是 `go test` 的默认前置步骤之一。

## 6.3 go doc —— 文档

```
go doc fmt.Println    # 查看函数文档
go doc -all fmt       # 查看包全部文档
go doc -src fmt.Println   # 查看实现源码
```

Go 的注释即文档：包注释、函数注释遵循固定格式（以声明名开头），`go doc` 与 pkg.go.dev 都据此渲染。写高质量注释是 Go 社区的基本礼仪。

# 7. go tool —— 底层工具链入口

```
go tool compile      # 编译器（将 .go 编译为 .o 中间产物）
go tool asm          # 汇编器
go tool link         # 链接器
go tool objdump      # 反汇编
go tool nm           # 查看符号表
go tool pprof        # 性能剖析
go tool trace        # 运行时跟踪
```

★ **关键认知**：`go build` 本质是 `compile` → `asm` → `link` 的封装。理解这条流水线，是后续研究"Go 代码如何变成机器码"、"逃逸分析发生在哪一步"的基础（程序结构、逃逸分析、汇编章节会反复用到）。

# 8. 小结

| 命令 | 作用 | 一句话记忆 |
|------|------|-----------|
| go run | 编译并运行 | 临时编译，跑完就删 |
| go build | 编译 | 产物在当前目录 |
| go install | 编译并安装到 GOBIN | 装工具用这个 |
| go mod init | 初始化模块 | 生成 go.mod |
| go mod tidy | 整理依赖 | 最常用，保 go.mod/go.sum 干净 |
| go get | 添加/升级依赖 | 精确控制版本 |
| go vet | 静态检查 | 编译器不管的 bug |
| go fmt | 格式化 | 风格唯一化 |
| go tool | 底层工具链 | 深入原理的入口 |

下一篇进入程序结构，讲清楚 `package` / `main` / `import` 的语义，以及一个 Go 程序从源码到可执行文件的完整编译链接过程。
