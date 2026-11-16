# 1. 源码与架构概览

## 1.1 核心模块

**k6 是一个纯 Go 项目（AGPL-3.0），脚本能力靠内嵌的 JS 解释器提供，整体遵循"核心稳定 API + internal 实现 + 扩展注册"的三层组织**。v0.5x 的仓库目录结构（master 分支）：

```text
k6/
├── main.go              # 入口：调用 cmd 包的根命令
├── cmd/                 # CLI 根命令（薄壳，实现在 internal/cmd）
├── lib/                 # 核心公共类型：Test、Options、Runner、VU 接口
├── js/                  # JS 运行时对外部分：模块 API（k6、k6/http 等）
├── metrics/             # 指标公共类型：Metric、Sample、Trend/Counter/Rate/Gauge
├── output/              # Output 扩展接口（第三方输出实现的稳定 API）
├── errext/、ext/         # 错误类型（退出码映射）、xk6 扩展注册机制
├── api/、cloudapi/       # REST API 与 k6 Cloud 协议
├── vendor/              # 依赖内联（含 sobek——JS 引擎）
└── internal/            # 全部内部实现
    ├── cmd/             # run/archive/inspect 子命令与 GlobalState
    ├── js/              # 编译器、Bundle、js.Runner、事件循环、内置模块
    ├── lib/executor/    # 7 种执行器实现
    ├── execution/       # 调度器（Scheduler/Ticker）与分布式执行控制
    ├── metrics/engine/  # 指标聚合引擎与阈值判定
    ├── output/          # 内置输出：json、csv、prometheus-rw、cloud
    └── event/、ui/、log/  # 事件系统（TestStart/TestEnd）、进度条、日志
```

三个职责最重的核心模块：**JS 运行时**（`js/` + `internal/js/`，负责编译并执行用户脚本，把 JS 世界桥接到 Go 实现）、**调度器**（`internal/execution` + `internal/lib/executor`，管理 VU 生命周期与负载模型）、**指标系统**（`metrics/` + `internal/metrics/engine`，采集、聚合、阈值判定、分发给各输出端）。

> **注意**：早期版本的 k6 有一个 `core/` 包承载本地执行引擎（Engine），v0.47 之后经过重构，这部分职责被吸收进 `lib.TestRunState`（组装 outputs、指标引擎、执行器）与 `internal/execution`（调度与分布式控制），网上一些旧架构文章提到的 `core.Engine` 在 v0.5x 已不存在，读源码时不要被旧资料带偏。

## 1.2 执行流程

**一次 `k6 run` 的完整生命周期分四个阶段：init → setup → VU 循环 → teardown**，对应到源码是"编译产物逐级装配"的过程：

```text
k6 run script.js
      │
      ▼
┌─────────────┐   解析+转译    ┌──────────────┐
│  Compiler    │ ────────────► │    Bundle    │  脚本+依赖打成单一可执行单元
│ (internal/js)│               │  (含 Program) │
└─────────────┘               └──────┬───────┘
                                     ▼
                              ┌──────────────┐
                              │   js.Runner   │  实现 lib.Runner 接口
                              └──────┬───────┘  init context 在此求值
                                     ▼
        lib.TestRunState：组装 MetricEngine、Outputs、Thresholds、Executors
                                     ▼
   setup() 执行一次（专属 VU）──► Scheduler 启动各场景 Executor
                                     ▼
   Executor 按负载模型驱动 VU 池：NewVU → RunIteration → 释放
                                     ▼
   teardown(data) 收尾 ──► handleSummary ──► 阈值判定 ──► 退出码
```

init 阶段的本质是**每个 VU 创建时都会重新求值一遍模块顶层代码**（所以顶层拿不到运行时状态，VU 专属 API 在 init 里调用会报错）；setup 由一个特殊 VU 执行且只执行一次，返回值序列化后分发给所有 VU；default 函数被 Executor 反复调用产生迭代；teardown 拿到 setup 的返回值做清理。

## 1.3 Runner / Executor / Output 三层抽象

**k6 的执行架构可以概括为三层解耦：Runner 负责"怎么跑一次迭代"，Executor 负责"以什么负载模式驱动迭代"，Output 负责"迭代产生的指标流向哪里"**。三者通过 `lib` 包里的接口互相认识，谁都不关心另外两者的具体实现：

| 抽象 | 关键接口 | 实现方 | 职责 |
|------|----------|--------|------|
| Runner | `lib.Runner`：`NewVU()`、`Setup()`、`Teardown()`、`MakeArchive()` | `js.Runner`（JS 脚本）、archive 恢复的 Runner | 生产 VU，定义单次迭代的行为 |
| Executor | `lib.Executor`：`Run(ctx, execFn)`、`GetMaxPlannedVUs()` | `internal/lib/executor` 下 7 种执行器 | 决定何时、用多少 VU 执行迭代 |
| Output | `output.Output`：`AddMetricSamples([]SampleContainer)`、`Start()/Stop()` | json、csv、experimental-prometheus-rw、cloud、dashboard | 消费指标样本流 |

这个三层的价值在组合性：同一个脚本（Runner）可以换执行器改负载模型而不改一行业务代码；同一次运行可以同时挂多个 Output（本地 summary + Prometheus + JSON 文件）互不干扰；xk6 扩展出的新协议模块也只需要产出标准的 `metrics.Sample`，所有 Output 自动可见。**设计考量**：三层之间传递的是纯数据（VU 接口、Sample 结构体），没有回调地狱——这使得分布式执行只需在 Executor 层做文章（把"驱动 VU"的职责切分到多个 instance 进程），Runner 与 Output 完全无感知。

## 1.4 JS 编译链与 sobek（Goja）集成

**k6 的 JS 引擎是 sobek——Grafana 从 dop251/goja 分叉并持续维护的版本（v0.50 起替换，API 与 goja 基本兼容，读旧资料时可以把两者视为同一个东西）**。sobek 是纯 Go 实现的 ES5.1+ 解释器，无 JIT，这直接决定了 k6 的资源模型（见第 17 篇）。脚本进入引擎前要过编译链（`internal/js/compiler`）：

```text
script.js（+ import 的依赖）
      │
      ▼
┌──────────────┐  compatibility-mode=full：
│  Transpiler   │  用内嵌 sobek 实例跑打包进二进制的 Babel，
│               │  把 ES6+ 转译为 ES5.1 并注入 core-js polyfill
└──────┬───────┘  compatibility-mode=base：跳过转译，sobek 原生解析
       ▼
┌──────────────┐  解析全部 import/require，把本地模块、远程模块、
│   Resolver    │  npm 依赖收敛为单一 Program（bundle 产物）
└──────┬───────┘
       ▼
┌──────────────┐  为每个 VU 创建独立的 sobek.Runtime，
│  InitContext  │  注册 k6 内置模块（k6/http、k6/data…），
│               │  求值模块顶层代码，导出 options/setup/default
└──────────────┘
```

JS↔Go 桥接的机制是 sobek 的宿主对象注入：`k6/http` 等内置模块（`internal/js/modules`）在 init context 里以 Go 结构体形式挂到全局对象上，JS 侧调用 `http.get()` 实际执行的是 Go 函数；参数与返回值通过 sobek 的反射映射自动转换。**每个 VU 一个独立 Runtime 实例**是 k6 内存开销的根源，也是它选择"不做跨 VU 共享 JS 对象"（SharedArray 因此只共享底层字节、读取时拷贝）的原因——隔离换取了无锁并发。

## 1.5 事件循环实现

**sobek/goja 本身没有事件循环，k6 在 `internal/js/eventloop` 里自己实现了一个最小化的事件循环来支撑 Promise 与 async/await**。机制：VU 执行 default 函数时，事件循环维护一个任务队列（microtask queue）；JS 代码里 `await` 会把后续逻辑注册为 Promise 回调并让出控制权，Go 侧完成实际工作（HTTP 请求、sleep 定时器）后把结果 resolve 回 Promise，事件循环继续排空队列，直到没有待处理任务才认为"这次迭代结束"。

两个工程含义：其一，k6 的 async/await 不是 Node.js 那种可以无限等待的循环——**迭代结束时队列必须排空，悬而未决的 Promise 会导致 "event loop terminated" 类错误**（见第 18 篇 2.5）；其二，同一 VU 内的事件循环是串行的，`Promise.all` 并发多个请求在 k6 里真并发（底层 Go 同时发请求），但 JS 回调仍按序在单 VU 的循环里执行，不要指望 JS 侧的并行计算。

# 2. 指标管线

## 2.1 采集

**每次请求、每个 check、每次迭代都会生成带标签的 `metrics.Sample`，由 VU 本地缓冲后批量上送**。Sample 的结构决定了 k6 指标系统的全部表达力：

```go
// metrics/sample.go（简化）
type Sample struct {
    TimeSeries                   // = Metric（指标定义）+ TagsAndMeta（标签集）
    Time     time.Time           // 采样时刻
    Value    float64             // 数值：耗时、字节数、0/1
    Metadata map[string]string   // 事件类元数据
}
```

HTTP 模块每完成一个请求会一次性产出一组样本：`http_req_duration` 及其分段（blocked/connecting/sending/waiting/receiving）、`http_req_failed`（Rate 型）、`http_reqs`（Counter 型）等，共享同一组标签（`name`、`method`、`status`、`scenario`、`group`、用户自定义 tag）。采集是 VU 本地无锁的——样本先写进 VU 自己的缓冲区，迭代结束或缓冲满时通过 channel 发给聚合层，这是高 VU 数下指标系统不成为瓶颈的关键。

## 2.2 聚合

**聚合发生在 `internal/metrics/engine`：按"指标 + 标签集"（TimeSeries）为键累积，不同指标类型用不同算法**。四种基础类型的聚合语义：

| 类型 | 聚合方式 | 典型指标 | 内存特征 |
|------|----------|----------|----------|
| Counter | 求和 | `http_reqs`、`data_received` | O(1) |
| Gauge | 保留最后值（附带 max/min） | `vus` | O(1) |
| Rate | 布尔计数比 | `http_req_failed`、checks | O(1) |
| Trend | 全部值进直方图/采样结构算分位数 | `http_req_duration` | O(桶数) |

Trend 值得多说一句：精确分位数需要保存所有样本，百万级请求下内存不可接受，k6 用的是**近似直方图算法**（`internal/ds/histogram`，对数分桶 + 桶内插值），P95/P99 是估算值——与 Prometheus 的 histogram_quantile 同理。所以 summary 里的分位数在数据量极大时可能与其他工具算出的精确值有小偏差，这是设计取舍而非 bug。用户自定义的 `new Trend()` / `new Counter()` 走完全相同的管线。

## 2.3 分发与阈值判定

**聚合层是扇出点：同一份样本流一路进阈值判定，一路复制给所有 Output**。阈值判定也在 `internal/metrics/engine`——每收到样本就增量更新对应 TimeSeries 的阈值表达式状态（`p(95)<500` 这类表达式在测试启动时编译为可增量求值的结构），配置了 `abortOnFail` 且判定窗口满足时触发中止事件。Output 侧只实现 `AddMetricSamples`，收到的是批量样本容器（`SampleContainer`，可以是单条也可以是 HTTP 请求的一组），各输出自行决定落盘格式与频率。测试结束时引擎向事件系统（`internal/event`）广播 TestEnd，Output 依次 flush、`handleSummary()` 拿到聚合结果生成报告、阈值做最终判定决定退出码——第 16 篇的退出码 99 正是从这里产生的。

# 3. 执行器实现思路

## 3.1 闭合模型执行器

**闭合模型（closed model）执行器的核心逻辑是"维持 VU 数量恒定，VU 用完一轮迭代立即开始下一轮"**。`per-vu-iterations`（每 VU 固定迭代数）与 `constant-vus`（固定 VU 数持续跑）都属此类，实现上共享同一套骨架：

```text
Executor.Run(ctx, execFn) 内部：
  1. 向 VU 池申请 N 个 VU（池不够则调 runner.NewVU() 新建）
  2. 每个 VU 起一个 goroutine，循环执行 execFn(vu) —— 即一次迭代
  3. 迭代 = 事件循环跑一遍 default 函数，产出样本
  4. 迭代间由负载配置决定节奏：立即下一轮 or 直到 iterations 用尽 / duration 到期
  5. 结束后 VU 归还池（不销毁，供 ramping 场景复用）
```

闭合模型下**实际 RPS = VU 数 ÷ 单次迭代耗时**，迭代耗时包含响应时间与 sleep——服务端一慢，RPS 自动下降，这就是协调遗漏的结构性来源（面试篇细讲）。ramping 类执行器（`ramping-vus`）在这套骨架上加了一个时间轴插值器：`internal/execution` 的 Scheduler 按 stage 边界推进目标 VU 数，执行器对比当前 VU 数与目标值，多退少补。

## 3.2 开放模型执行器

**开放模型（open model）执行器的核心逻辑反过来："按固定到达率发射迭代，VU 只是执行迭代的资源池"**。`constant-arrival-rate` 与 `ramping-arrival-rate` 的实现要点：

- 启动时预分配 `preAllocatedVUs` 个 VU 常驻；
- 一个定时器按 `rate/timeUnit` 计算发射间隔（内部用精确的 ticker 补偿而非简单 sleep，避免累积漂移）；
- 到点就从空闲 VU 池取一个执行迭代；**没有空闲 VU 且未达 `maxVUs` 时临时新建，达到 `maxVUs` 仍无可用 VU 则丢弃这次迭代并递增 `dropped_iterations` 指标**；
- 迭代完成慢于发射速率时，未完成迭代的数量堆积——这是服务端过载在开放模型下的正确表现（延迟上升被如实记录，而不是被 VU 数量掩盖）。

**设计考量**：`dropped_iterations` 是开放模型最重要的自监控指标——它大于 0 说明要么 maxVUs 不够、要么服务端已经慢到 VU 周转不过来，两种情况都意味着"实际到达率 < 目标到达率"，此时的吞吐数据要打折解读。

## 3.3 VU 调度与分布式执行

VU 池（`internal/lib/executor` 内的 VU 管理）解决的是"VU 创建昂贵（要新建 sobek Runtime + 求值 init 代码）、销毁浪费"的问题——池化复用让 ramping 场景升降 VU 数时不必反复付 init 成本。`internal/execution` 的 Scheduler 负责多场景并发时的全局协调：各场景独立持有执行器，共享一个进度 UI 与中止信号（任何一个场景触发 abortOnFail，全部场景一起停）。分布式执行（v0.49+）也在这一层：controller 进程持有 Scheduler，把"目标 VU 数/到达率"按 instance 数量切分下发，各 instance 本地跑自己的执行器、指标回流 controller 聚合——对上层的 Runner 与 Output 完全透明。

# 4. 扩展点

## 4.1 输出扩展

**给 k6 加一个新的指标目的地，只需实现 `output.Output` 接口并用 `ext.Register` 注册**：

```go
package myoutput

import (
    "go.k6.io/k6/ext"
    "go.k6.io/k6/output"
)

func init() {
    ext.Register(ext.Name("output-myoutput"), output.NewOutput, ext.Priority(ext.PriorityHigh))
}

type Output struct {
    output.SampleBuffer          // 内嵌缓冲实现，攒批消费
}

func (o *Output) Description() string { return "my custom output" }

func (o *Output) Start() error {
    go func() {
        for range o.PeriodicFlush(1 * time.Second) {  // 每秒刷一批
            samples := o.GetBufferedSamples()
            // 写入目标存储……
        }
    }()
    return nil
}
```

用 `xk6 build --with k6-output-myoutput=./myoutput` 编出定制二进制后，`k6 run --out myoutput script.js` 即可使用。`SampleBuffer` + `PeriodicFlush` 是官方给的标准骨架，内置的 json/csv 输出也是同一套模式，抄 `internal/output` 里的实现最省事。

## 4.2 协议扩展

**协议扩展的入口是 JS 模块系统：用 Go 实现一个新模块（如 k6-sql、k6-kafka、xk6-grpc 的思路），注册进 sobek Runtime，产出标准 Sample，就自动获得 checks/thresholds/全部 Output 的支持**。xk6 扩展的通用形态：

```go
// 扩展模块骨架：实现 modules.Module 接口
func init() {
    modules.Register("k6/mysql", func() modules.Module { return &RootModule{} })
}

// RootModule 负责实例化，InstanceModule 提供每个 VU 的方法集
// JS 侧：import sql from 'k6/mysql';
```

选择 xk6 而不是提 PR 进主仓库是社区惯例：**主仓库对内置模块的准入门槛极高（维护成本、API 稳定性承诺），绝大多数协议扩展以 xk6 生态库存在**，官方维护的 k6-browser、k6 各 experimental 模块是少数例外。扩展列表见 github.com/grafana/xk6-* 系列仓库。

## 4.3 二次开发切入点

按改动目的选切入点：

- **加自定义指标输出/对接自研平台** → 实现 Output 扩展（4.1），零侵入。
- **支持新协议**（Dubbo、私有 TCP 协议）→ JS 模块扩展 + xk6 构建（4.2）。
- **改负载模型**（如按泊松过程到达）→ 仿照 `internal/lib/executor/constant_arr_rate.go` 实现新 Executor，注册进 options 解析——侵入 internal 的改动，需要 fork 维护。
- **JS API 行为调整**（如给 http 模块加钩子）→ `internal/js/modules/k6/http`，`http.get/post/request` 的全部实现（含连接池与重定向策略）都在这里；改指标语义则去 `metrics/` + `internal/metrics/engine`，注意与 threshold 表达式的兼容。

> **行业实践**：企业内二次开发最常见的形态是"fork + 私有 Output 扩展 + xk6 构建脚本"三件套，核心引擎几乎不动——k6 的扩展面设计使得绝大多数需求不需要碰调度与指标内核。真要改 internal 的，先在 issue 里与维护者对齐，AGPL 协议也要求 fork 后的修改对外开源。

# 5. 阅读源码的建议

## 5.1 关注关键路径

不要平铺着读，沿一次真实运行的调用链走一遍最有效。推荐的三条路径：**启动路径**（`main.go → internal/cmd` → `internal/js/compiler` → Bundle → `js.Runner` → `lib.TestRunState`），搞清楚"一个 .js 文件如何变成可执行的 Test"；**调度路径**（`internal/execution` 的 Scheduler → `internal/lib/executor`，挑 per_vu_iterations 与 constant_arr_rate 对比读，一个最短一个最典型），搞清楚 VU 从哪来、迭代怎么被驱动；**指标路径**（`internal/js/modules/k6/http` 产样本 → `internal/metrics/engine` 聚合与阈值 → `internal/output/json` 消费），搞清楚一个请求的耗时怎么变成 summary 里的一行。三条路径走完，架构图上的每个框都有了着落。

## 5.2 结合实际问题阅读

**带着排错需求读源码效率最高**，几个高频问题的对应模块：压测端性能瓶颈（第 17 篇）→ 读 `internal/js`（init 求值与 VU 创建成本）与 sobek 的 Runtime 结构；`http_req_blocked` 异常 → 读 `internal/js/modules/k6/http` 里的连接池与 transport 配置；分布式执行不稳定 → 读 `internal/execution` 的 controller/instance 协议与断连处理；阈值判定行为与文档不符 → 读 `internal/metrics/engine` 的 thresholds 求值（abortOnFail 的 delay 逻辑常有误解）。读的时候配合 `go test ./internal/lib/executor/...` 这类包级测试——k6 的执行器与指标引擎测试覆盖率很高，测试用例本身就是最好的行为文档。

# 6. 小结

k6 的源码架构围绕三层抽象展开：Runner（`lib.Runner`，由 `js.Runner` 实现）定义单次迭代的行为，Executor（`internal/lib/executor` 的 7 种实现）决定负载如何驱动迭代，Output（`output.Output` 接口）消费指标样本流，三层之间只传递纯数据，因而可以独立替换——分布式执行只需切分 Executor 层的职责。JS 侧由 sobek（goja 的 Grafana 分叉）承载，编译链在 full 模式下用内嵌的 Babel 转译为 ES5.1，每个 VU 持有独立 Runtime 与自行实现的事件循环（`internal/js/eventloop`）支撑 async/await。指标管线是"VU 本地无锁采集 → engine 按 TimeSeries 聚合（Trend 用近似直方图）→ 扇出到阈值判定与多个 Output"。扩展的正规姿势是 Output 扩展与 xk6 JS 模块扩展，几乎不需要触碰 internal 内核；读源码沿启动、调度、指标三条路径走一遍，再结合具体排错问题深入对应模块即可。
