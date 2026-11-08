# 1. 扩展机制概览

## 1.1 为什么需要扩展

**k6 的内核刻意保持精简，扩展（Extension）机制是官方指定的能力增长路径**。k6 内置的协议只有 HTTP/1.1、HTTP/2、WebSocket、gRPC 等少数几种，输出后端也有限；而真实压测需求五花八门——压 AMQP 队列、压 MQTT 物联网端点、压私有 TCP 二进制协议、把指标写进公司自研的时序库。对这类需求，k6 的回答不是"往核心里加代码"，而是**通过扩展增强功能而不修改核心**：扩展以 Go 模块的形式存在，在编译期与 k6 链接成一个定制二进制。这个设计的根本原因是 k6 的架构形态——它是**单一 Go 可执行文件**，脚本引擎的宿主能力（协议客户端、输出后端、JS 模块）都编译在二进制里，JS 脚本层无法凭空获得新的宿主能力，只能在编译期注入。

> **设计考量**：编译期注入而非运行时插件（如动态库加载），换来的是**零插件 ABI 兼容负担与可复现的构建**——一个定制 k6 二进制就是它全部能力的完整快照，拷到任何同架构机器上行为一致；代价是每换一次扩展版本都要重新编译，且不同扩展组合出的二进制互不通用。对压测工具而言这个取舍是划算的：压测环境本来就要求"客户端行为绝对确定"。

## 1.2 扩展类型

按注入点划分，k6 扩展主要有三类，各自实现不同的 Go 接口并注册到不同子系统：

| 类型 | 注入点 | 解决的问题 | 代表扩展 |
|------|--------|-----------|---------|
| **JS 模块扩展** | `modules.Register` 注册进脚本引擎 | 在 JS 里暴露新协议/新能力，如 `import kafka from "k6/x/kafka"` | xk6-kafka、xk6-mqtt、xk6-browser（已并入内核） |
| **输出扩展** | `output.RegisterExtension` 注册进输出管线 | 把指标写入新后端，对接 `--out` 参数 | xk6-output-prometheus-remote、xk6-output-kafka |
| **混合/其他** | 两者兼有或扩展 CLI | 既提供 JS API 又提供输出，或改造命令行行为 | xk6-dashboard（实时 Web 面板）、xk6-sql |

JS 模块扩展让脚本获得新的 `import` 路径（惯例是 `k6/x/<name>` 前缀，区别于内置的 `k6/*`）；输出扩展对用户完全透明——编译进二进制后，`--out myext=...` 就能用，脚本一行不改。两类扩展可以组合进同一次 `xk6 build`。

# 2. xk6 的作用

## 2.1 xk6 build 原理与用法

**xk6 是官方构建工具，原理是"生成临时 Go 模块 + 编译期注册 + go build"**。执行 `xk6 build` 时它在临时目录里做四件事：初始化一个新的 Go module；把 k6 本体作为依赖引入（版本由 `--with k6@vX.Y.Z` 或默认最新版决定）；生成一个 `imports.go`，把所有 `--with` 指定的扩展包以空白导入（`_ "github.com/..."`）方式引入——扩展包的 `init()` 函数借此机会调用 `modules.Register` / `output.RegisterExtension` 完成自注册；最后 `go build` 产出单个二进制。整个流程没有任何运行时魔法，**扩展就是被链接进来的普通 Go 代码**：

```text
xk6 build v0.57.0 --with github.com/grafana/xk6-output-prometheus-remote@v0.3.0
        │
        ▼
┌──────────────────────────────────────────┐
│ 临时 Go module（xk6 自动创建）             │
│   go.mod:                                │
│     require go.k6.io/k6 v0.57.0          │
│     require .../xk6-output-... v0.3.0    │
│   imports.go:                            │
│     import _ "go.k6.io/k6"               │
│     import _ ".../xk6-output-..."        │
│            │                             │
│            ▼ 扩展包的 init() 自注册        │
│   go build ──► ./k6（定制二进制）          │
└──────────────────────────────────────────┘
```

常用命令形态：

```bash
# 安装 xk6（需要本机有 Go 工具链）
go install go.k6.io/xk6/cmd/xk6@latest

# 用指定 k6 版本 + 一个输出扩展 + 一个 JS 模块扩展构建
xk6 build v0.57.0 \
  --with github.com/grafana/xk6-output-prometheus-remote@v0.3.0 \
  --with github.com/grafana/xk6-kafka@v0.24.0 \
  --output ./k6-custom

# 验证扩展已编入：版本输出会列出扩展清单
./k6-custom version
# 输出示例：k6 v0.57.0 ... 
# Extensions: github.com/grafana/xk6-kafka@v0.24.0, ...
```

构建出的 `k6-custom` 与官方 k6 完全同构使用，只是多了扩展能力：

```bash
./k6-custom run script.js \
  --out prometheus-rw=http://prom:9090/api/v1/write
```

## 2.2 开发自定义扩展

**开发一个 JS 模块扩展的最小骨架**：实现 `modules.Instance` 接口（`NewModuleInstance` 返回每 VU 一个的模块实例），在 `init()` 里注册，并把要暴露给 JS 的 Go 结构体方法直接映射为 JS 函数。下例实现一个查询 SQL 行数的小扩展（真实项目应使用成熟的 xk6-sql，此处仅演示接口形态）：

```go
// 扩展包 main.go —— 演示 JS 模块扩展的最小结构
package rowcount

import (
	"go.k6.io/js/modules"       // 模块注册接口
)

func init() {
	// 注册模块：JS 侧 import rowcount from "k6/x/rowcount" 即可拿到
	modules.Register("k6/x/rowcount", NewModuleInstance)
}

// Module 是模块级对象，NewModuleInstance 每个 VU 调用一次
type Module struct{}

func NewModuleInstance(vu modules.VU) modules.Instance {
	return &ModuleInstance{vu: vu}
}

// ModuleInstance 上的导出方法会直接暴露为 JS 函数
type ModuleInstance struct {
	vu modules.VU
}

// Exports 返回 JS 侧可见的默认导出对象
func (mi *ModuleInstance) Exports() modules.Exports {
	return modules.Exports{Default: mi}
}

// Greet 在 JS 里是 rowcount.greet("k6") —— Go 方法名首字母大写即导出
func (mi *ModuleInstance) Greet(name string) string {
	return "hello, " + name
}
```

JS 侧使用：

```javascript
import rowcount from "k6/x/rowcount";

export default function () {
  console.log(rowcount.greet("k6"));   // hello, k6
}
```

**开发一个输出扩展**则实现 `output.Output` 接口（核心是 `Start` / `AddMetricSamples` / `Stop` 三个方法），用 `output.RegisterExtension` 注册，之后 `--out <name>=<config>` 就会把指标样本流式送进来。输出扩展的关键设计点是 `AddMetricSamples` 在热路径上被高频调用，**内部必须做缓冲与批量异步落盘**，绝不能同步写网络——否则会反压负载生成器本身，扭曲压测结果。

## 2.3 版本与兼容性

**扩展版本必须与 k6 版本匹配，这是编译期依赖冲突问题而非运行时问题**。扩展的 `go.mod` 声明了它依赖的 k6 版本区间，xk6 build 时若扩展锁定的 k6 API 与你指定的 k6 版本不兼容，Go 编译器直接报错（接口签名变更、包路径迁移等）。实践规则：查看扩展仓库的 release notes 或 `go.mod` 确认其适配的 k6 版本；k6 大版本升级（如 v0.4x → v0.5x）后，先重新构建定制二进制并跑冒烟测试再上正式压测；**编译后需在目标环境验证**——扩展通常含 cgo 或平台相关代码时，本地构建的二进制未必能在压测机（不同 OS/架构）上运行，交叉编译或直接在目标平台构建。

# 3. 知名扩展与选型

## 3.1 主流扩展清单

社区生态里经过大规模验证的扩展主要有：

| 扩展 | 类型 | 用途 | 备注 |
|------|------|------|------|
| **xk6-output-prometheus-remote** | 输出 | 指标经 Remote Write 进 Prometheus/Mimir/VictoriaMetrics | Grafana 官方维护，k6 可观测集成的事实标准（见第 14 篇） |
| **xk6-kafka** | JS 模块 | 压测 Kafka：produce/consume、带认证、Avro/JSON 序列化 | 压消息队列场景首选 |
| **xk6-sql** | JS 模块 | 脚本内直连数据库执行 SQL | 常用于压测前置造数/后置清理，注意连接池配置 |
| **xk6-output-kafka** | 输出 | 指标样本写入 Kafka topic | 对接自研指标管道 |
| **xk6-dashboard** | 混合 | 运行时 Web 实时面板 | 已随内核 `--out web-dashboard` 官方化，新工程可不再用扩展版 |
| **xk6-browser** | JS 模块 | 真实 Chromium 浏览器压测 | v0.52 起已并入 k6 内核（`k6/browser`），无需再自行编译 |

xk6-browser 并入内核这件事值得注意：**Grafana 把验证成熟的扩展收编进主发行版是既定路线**，选型时优先看能力是否已官方化，避免维护不必要的定制二进制。

## 3.2 选型原则

**优先选用活跃维护、许可证明确、来源可信的扩展**。具体判断维度：看仓库最近提交时间与 issue 响应（超过一年无提交的扩展要假设它已死）；看许可证（k6 生态主流是 AGPL-3.0，与 k6 本体一致——AGPL 对"内部使用压测工具"没有传染风险，但如果你要把扩展代码合入对外分发的产品则需法务评估）；看维护方（Grafana 官方 org 下的扩展 > 大厂团队维护 > 个人项目）；**安全风险**方面，扩展是编译进二进制的任意 Go 代码，能读写文件、发起网络请求，引入前至少通读其主要源文件，个人小众扩展建议在隔离环境构建。

**生产使用建议**：把定制二进制的构建固化为 CI 产物——扩展清单（哪些扩展、哪些版本、基于哪个 k6 版本）写进仓库的构建脚本，**构建可复现的二进制**并归档，压测报告里记录所用二进制的版本指纹。这样半年后复跑同一场景，客户端能力与当时完全一致。

# 4. 常见问题

## 4.1 编译失败

**编译失败九成是 Go 环境问题，一成是版本冲突**。按序排查：本机未装 Go 或版本过低（xk6 与新版 k6 要求 Go 1.21+，`go version` 确认）；企业网络下 `go mod download` 拉不到依赖（配置 `GOPROXY`，如 `go env -w GOPROXY=https://goproxy.cn,direct`）；扩展与 k6 版本不兼容时错误信息是 Go 编译错误（`undefined: xxx`、接口不匹配），解法回到 2.3 节的版本对齐；磁盘/tmp 权限问题导致 xk6 临时目录创建失败。构建慢属于正常现象——首次要下载并编译整个 k6 依赖树，后续有 Go build cache 会快很多。

## 4.2 运行时崩溃

**扩展不兼容导致的运行时异常，典型症状是 panic 或指标静默缺失**。JS 模块扩展 panic 多因扩展对 k6 内部 API 的调用假设失效（编译过了但语义变了），升级 k6 后未同步升级扩展是高发原因；输出扩展"崩溃"更常见的形态是不崩但**不工作**——`--out name=` 里的名字拼错、扩展注册的名字与文档不符（用 `k6 version` 输出核对扩展清单）、或配置串格式不对导致 Start 失败后 k6 降级继续跑。另一类隐蔽问题：**用官方 k6 二进制跑了依赖扩展的脚本**，报 `unknown module: k6/x/kafka`——这不是脚本 bug，是拿错了二进制，团队内应统一分发定制二进制并在 README 里写明构建命令。

> **行业实践**：成熟团队的压测平台通常维护"标准定制二进制"：一个固定配方（k6 版本 + prometheus-rw 输出 + 业务所需协议扩展）由 CI 每周构建、打版本 tag、分发到所有压测机与镜像仓库；脚本仓库只声明"需要二进制 >= 某版本"，不允许个人随意加扩展。这把扩展管理从"每人一个环境"收敛成"平台一个产物"，可复现性和排障效率都高一个量级。

# 5. 小结

k6 扩展是**编译期注入的 Go 模块**：JS 模块扩展给脚本新增协议能力（`k6/x/*`），输出扩展给 `--out` 新增指标后端。xk6 的构建原理是生成临时 Go module、空白导入触发自注册、`go build` 出单一二进制，因此**扩展与 k6 的版本匹配是编译期强约束**。开发扩展分别实现 `modules.Instance` 与 `output.Output` 接口，输出扩展务必异步缓冲以免反压负载生成。选型看活跃度、许可证（生态主流 AGPL-3.0）与代码可信度；生产环境把定制二进制的配方固化为 CI 产物，保证可复现。xk6-browser 并入内核的先例提醒我们：先确认能力是否已官方化，再决定是否维护定制构建。
