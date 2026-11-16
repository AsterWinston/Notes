k6 的压测能力建立在"虚拟用户 + 执行器调度"这套执行模型之上：理解了 VU 到底是什么、六种执行器分别在控制什么变量，才能回答"这个测试测出来的数字到底意味着什么"。本章从运行时结构讲到执行器参数，再到负载模型设计与客户端资源限制。

# 1. 执行模型概述

## 1.1 VU 与调度核心思想

**虚拟用户（VU）的本质是"一个 Go 协程 + 一个独立的 Goja JS 运行时实例"。**k6 本体是 Go 程序，脚本里的 JavaScript 由 Goja（纯 Go 实现的 ECMAScript 引擎）解释执行。每创建一个 VU，k6 就启动一个协程，并在其中初始化一个全新的 Goja 运行时：脚本的 init 上下文代码（import、模块顶层语句）在这个运行时里独立执行一遍，随后该 VU 反复调用自己的默认函数（或场景指定的 exec 函数）。

因为每个 VU 拥有独立运行时，**VU 之间的 JS 变量空间完全隔离**：一个 VU 修改模块级变量，其他 VU 看不到，跨 VU 共享数据必须走 `SharedArray` 这类由 Go 侧托管的机制（见第 8 篇）。每个运行时内部有自己的事件循环，但与浏览器 JS 不同，k6 脚本以同步风格为主——`http.get` 直接返回响应对象，异步只出现在定时器与 Promise 等少数场景。

整个进程的结构可以画成下图：

```text
┌────────────────────────────────────────────────────────────┐
│                        k6 进程（Go）                        │
│  ┌──────────┐    ┌────────────────────────────────────┐    │
│  │ 调度器    │──→ │ 执行器（六种，决定 VU 数 / 到达率）   │    │
│  └──────────┘    └─────────────────┬──────────────────┘    │
│                                    │ 创建 / 复用 / 销毁      │
│         ┌──────────────────────────┼──────────────────┐    │
│         ▼                          ▼                  ▼    │
│  ┌─────────────┐           ┌─────────────┐    ┌─────────────┐
│  │ VU 1        │           │ VU 2        │ …  │ VU N        │
│  │ goroutine   │           │ goroutine   │    │ goroutine   │
│  │ Goja 运行时  │           │ Goja 运行时  │    │ Goja 运行时  │
│  │ 事件循环     │           │ 事件循环     │    │ 事件循环     │
│  └─────────────┘           └─────────────┘    └─────────────┘
└────────────────────────────────────────────────────────────┘
```

**调度器与执行器的分工**是理解 k6 行为的关键：执行器（executor）是一份声明式的"策略配置"，描述负载的形状——维持多少 VU、每秒发起多少次迭代、随时间怎么变化；调度器则是执行这份策略的运行时机制，负责 VU 池的分配、迭代的触发、指标采样与优雅收尾。用户只需要配置执行器，不需要（也无法）直接操作调度器。

## 1.2 开放模型与闭合模型

**闭合模型（closed model）以"并发用户数"为控制变量，开放模型（open model）以"请求到达率"为控制变量。**这是负载测试理论里最重要的一组概念，k6 的六种执行器恰好按这两类模型划分。

闭合模型假设系统里同时存在固定数量的用户：每个 VU 完成一次迭代（含思考时间）后立刻开始下一次。它的隐含行为是**背压**——被测系统变慢时，VU 被阻塞在等待响应上，发起请求的速率自动下降，负载"温柔"了下来。真实世界的多数互联网流量不是这样：新用户不会因为你的服务慢就停止访问，请求持续到达、在服务端排队，这才是系统真正要承受的形态。开放模型按固定到达率发起迭代，系统变慢时 k6 自动增派 VU 承接积压，排队效应被如实暴露。

| 维度 | 闭合模型 | 开放模型 |
|------|---------|---------|
| 控制变量 | VU 数量 | 到达率（迭代数 / 时间单位） |
| 对应执行器 | constant-vus、ramping-vus、shared-per-vu-iterations、per-vu-iterations | constant-arrival-rate、ramping-arrival-rate |
| 系统变慢时 | 迭代速率下降，负载自动降低 | VU 数上升，积压与排队被保留 |
| 延迟测量 | 存在协调遗漏（coordinated omission），结果偏乐观 | 更接近真实用户体验 |
| VU 数量 | 由配置直接决定 | 由"到达率 × 响应时长"动态决定，受 maxVUs 封顶 |
| 适用场景 | 固定席位类系统、长连接、企业内部应用 | 互联网对外服务、容量评估、SLA 验证 |

> **行业实践**：面向公众的 Web/API 服务做容量评估时，优先使用到达率类执行器；只有在模拟"固定数量的操作员持续在线"（客服系统、交易席位、WebSocket 长连接）时，闭合模型才是更贴近现实的抽象。

# 2. 执行器体系

## 2.1 常见执行器分类

k6 v0.5x 提供六种执行器，全部在 `scenarios` 中声明（不写 `scenarios` 时，`--vus`/`--duration` 会生成一个默认的 shared-per-vu-iterations 场景）。总览如下：

| 执行器 | 模型 | 核心参数 | 一句话定位 |
|--------|------|---------|-----------|
| shared-per-vu-iterations | 闭合 | vus、iterations、maxDuration | 固定 VU 数瓜分一个共享迭代池 |
| per-vu-iterations | 闭合 | vus、iterations、maxDuration | 每个 VU 独立完成固定次数迭代 |
| constant-vus | 闭合 | vus、duration | 恒定并发，最常用的长稳负载 |
| ramping-vus | 闭合 | startVUs、stages | 并发数随时间阶梯/线性变化 |
| constant-arrival-rate | 开放 | rate、timeUnit、duration、preAllocatedVUs、maxVUs | 恒定到达率 |
| ramping-arrival-rate | 开放 | startRate、timeUnit、stages、preAllocatedVUs、maxVUs | 到达率随时间变化 |

## 2.2 执行器选择策略

**执行器的选择本质上是在回答"我要控制的变量是并发数、迭代总量还是到达率"。**几条经验规则：

- **验证并发极限、找系统拐点**：用 `ramping-vus` 或 `ramping-arrival-rate` 阶梯加压，观察吞吐-延迟曲线的拐点位置（方法论见第 9 篇）。
- **贴近真实流量做容量评估**：用 `constant-arrival-rate`，把生产峰值 QPS 直接翻译成 rate。
- **长稳/浸泡测试**：`constant-vus` 或 `constant-arrival-rate` 配长 duration，关注资源泄漏而非吞吐。
- **一次性批量任务**（跑完 N 个样本即结束）：`shared-per-vu-iterations`。
- **CI 冒烟**：`per-vu-iterations` 配 1-2 个 VU、少量迭代，时间可预期、退出码确定。

## 2.3 迭代驱动：shared-per-vu-iterations 与 per-vu-iterations

**shared-per-vu-iterations 让固定数量的 VU 竞争消费一个共享的迭代池，per-vu-iterations 则给每个 VU 派发等量的独立任务。**两者都以"总迭代数"而非"时长"作为结束条件：

```javascript
export const options = {
  scenarios: {
    shared: {
      executor: "shared-per-vu-iterations",
      vus: 10,               // 并发 VU 数
      iterations: 200,       // 全部 VU 共享 200 次迭代，先空闲先领取
      maxDuration: "5m",     // 兜底时长，超时未跑完则强制结束
      gracefulStop: "10s",   // 结束时等待进行中迭代收尾的宽限期
    },
    perVU: {
      executor: "per-vu-iterations",
      vus: 10,               // 并发 VU 数
      iterations: 20,        // 每个 VU 独立跑 20 次，共 10 × 20 = 200 次
      maxDuration: "5m",
    },
  },
};

export default function () {
  // 迭代体
}
```

**设计考量**：

- **shared 版的吞吐更快、样本分布不均**：快的 VU 会多做迭代，适合"尽快采满 N 个样本"的回归测试；但如果被测系统对某些 VU 特别慢，总时长会被拖长。
- **per-vu 版的节奏均匀、总时长可预期**：每个 VU 做的事情一样多，适合与数据分片配合——第 i 个 VU 固定消费第 i 片数据（见第 8 篇的 `vu.idInTest` 分片）。
- `maxDuration` 默认 10 分钟，迭代没跑完也会被切断，报告里会看到 `dropped_iterations` 之外的截断现象，批量任务要把它调大。

## 2.4 并发驱动：constant-vus 与 ramping-vus

**constant-vus 在指定时长内维持恒定数量的 VU，ramping-vus 则按 stages 定义的折线让 VU 数随时间变化。**这是闭合模型的主力执行器：

```javascript
export const options = {
  scenarios: {
    steady: {
      executor: "constant-vus",
      vus: 50,                // 恒定 50 个并发 VU
      duration: "10m",        // 持续 10 分钟
      gracefulStop: "30s",    // 结束时给进行中的迭代 30 秒收尾
    },
    ramp: {
      executor: "ramping-vus",
      startVUs: 0,            // 起始 VU 数（可省略，默认 0）
      stages: [
        { duration: "2m", target: 100 },  // 2 分钟内线性升到 100 VU
        { duration: "5m", target: 100 },  // 稳态保持 5 分钟
        { duration: "1m", target: 0 },    // 1 分钟内线性降回 0
      ],
      gracefulRampDown: "30s", // 削减 VU 时，给被移除 VU 的迭代收尾时间
      gracefulStop: "30s",
    },
  },
};
```

两个"graceful"参数的区别值得单独记住：

| 参数 | 生效时机 | 作用 |
|------|---------|------|
| gracefulRampDown | ramping-vus 缩减 VU 数时 | 被移除的 VU 若正在执行迭代，允许它跑完（最长等待该时长）再销毁 |
| gracefulStop | 场景/测试结束时 | 仍活跃的 VU 获得一段宽限期完成当前迭代，超时则强制中断 |

> **注意**：gracefulRampDown 默认 30 秒。如果迭代里有很长的 `sleep`（比如模拟用户思考 2 分钟），降载阶段 VU 会被强制中断，产生一批被截断的迭代样本，污染 `iteration_duration` 指标——长迭代场景要把这两个参数调到不小于单次迭代时长。

## 2.5 到达率驱动：constant-arrival-rate 与 ramping-arrival-rate

**到达率执行器把"迭代发起节奏"与"VU 复用"解耦，是 k6 模拟开放系统流量的关键。**调度器严格按 `rate / timeUnit` 的节奏发起迭代：优先复用 `preAllocatedVUs` 里空闲的 VU；如果所有预分配 VU 都在忙（系统变慢的直接后果），临时扩容新 VU，上限是 `maxVUs`；到达上限后仍无空闲 VU，这次迭代被**丢弃**并计入 `dropped_iterations` 指标。

```javascript
export const options = {
  scenarios: {
    api: {
      executor: "ramping-arrival-rate",
      startRate: 10,           // 初始到达率：每 timeUnit 发起 10 次迭代
      timeUnit: "1s",          // 到达率的时间单位
      preAllocatedVUs: 50,     // 预分配 VU：测试前就初始化好，可直接复用
      maxVUs: 500,             // VU 扩容上限：预分配不够用时最多扩到这里
      stages: [
        { target: 100, duration: "2m" },  // 2 分钟线性升到 100 迭代/秒
        { target: 100, duration: "5m" },  // 稳态 5 分钟（采样窗口）
        { target: 0, duration: "30s" },   // 降载收尾
      ],
      gracefulStop: "10s",
    },
  },
};
```

**maxVUs 的意义是安全阀，而不是负载参数**——真正的负载由 rate 决定，maxVUs 只限制 k6 愿意为兑现这个 rate 付出多少客户端资源。估算方法来自 Little 定律：稳态所需并发 ≈ 到达率 × 平均迭代时长。例如目标 100 it/s、平均迭代耗时 0.3 秒，理论上 30 个 VU 就够；`preAllocatedVUs` 按 2-3 倍余量设为 100，`maxVUs` 再放大数倍以容纳系统劣化时的积压。

```text
到达率 100 it/s，平均迭代耗时 0.3s → 稳态并发 ≈ 30 VU（Little 定律）

到达节奏：  ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼   每秒固定发起 100 次迭代，不管响应快慢
                 │
                 ▼
        ┌───────────────────────────┐
        │ preAllocatedVUs = 50      │ → 空闲则直接复用（零创建开销）
        ├───────────────────────────┤
        │ 扩容区（至 maxVUs = 500）  │ → 预分配全忙时才新建 VU
        └───────────────────────────┘
                 │
        仍然全忙 → 迭代被丢弃，dropped_iterations + 1（负载发生器已达上限）
```

**设计考量**：

- **preAllocatedVUs 要一次给足**：临时扩容的 VU 需要跑一遍 init 上下文（加载模块、解析数据），在高到达率下扩容会带来延迟毛刺；预分配则把这个成本移到测试开始前。
- **maxVUs 给太大**的代价是内存（每个 VU 一份 Goja 运行时），**给太小**的代价是静默丢迭代——目标到达率兑现不了，测出的吞吐低于设定值。跑完务必检查 `dropped_iterations` 是否为 0。
- 到达率执行器没有 gracefulRampDown（VU 不是按折线削减的，而是随积压自然回收）。

# 3. 负载模型设计

## 3.1 目标拆解

**负载模型必须从业务指标倒推：先有 SLO（目标 QPS、p95 延迟上限、错误率容忍度），再换算成执行器参数，而不是反过来。**换算的核心工具还是 Little 定律：`并发数 = 到达率 × 平均响应时长`，例如 500 QPS × 0.2s = 100 并发，preAllocatedVUs 按 2-3 倍余量设为 200-300。如果业务方给的是"峰值 5000 用户在线"这类闭合口径的数字，要先翻译成到达率：在线用户数 ÷（平均思考时间 + 平均响应时长）≈ 到达率。跳过这一步直接用 5000 VU 压测，负载强度会严重失真（通常是被放大数倍）。

## 3.2 预热与稳定段

**一次可信的负载测试应该有三段式时间结构：预热（ramp-up）、稳态（steady state）、降载（ramp-down）。**预热段的意义是让双方进入热状态——被测系统的 JIT 编译、连接池扩张、各级缓存填充，负载发生器侧的 VU 预分配与 DNS 缓存，都发生在最初几十秒；把这段的样本混进结论，p95 会被冷启动毛刺拉高。稳态段是唯一样本窗口，其时长应占整个测试的主导（建议 ≥ 70%），否则汇总指标失真。降载段让系统平稳回收资源，也便于在服务端监控上观察"负载撤掉后指标是否回归基线"——这是判断资源泄漏的第一信号。

> **行业实践**：k6 的运行结束汇总（end-of-test summary）统计的是全程样本，无法原生按时间窗口过滤。要么把稳态段设计得足够长以稀释预热影响，要么把指标输出到 Prometheus/Grafana 等时序系统后框选稳态窗口分析（见第 13、14 篇）。

# 4. 资源与执行限制

## 4.1 VU 的成本

**每个 VU 都是一个携带完整 Goja 运行时的协程，内存成本不可忽略。**空脚本的 VU 增量在数百 KB 量级，但真实脚本会把它推高到数 MB：init 上下文在每个 VU 的运行时里各执行一遍，模块顶层解析的 JSON 数组、加载的数据文件，**每个 VU 都持有一份独立拷贝**——1000 个 VU × 每 VU 10MB 数据 = 10GB，这是压测机 OOM 的最常见原因。对策是 `SharedArray`：数据只在 Go 侧存一份，所有 VU 共享（原理与代价见第 8 篇）。

另一个隐性成本是**保留的响应体**：k6 默认把每个响应体读入内存以支持 `res.json()`/`res.body`，高到达率下这是主要的内存放大器，不需要的场景应开启 `discardResponseBodies: true`。

判断瓶颈在负载发生器而非被测系统的信号：k6 进程 CPU 打满、`dropped_iterations` 激增、实际到达率低于设定值、`vus_max` 顶到 maxVUs。出现任一信号，先修客户端（精简脚本、SharedArray、调 GOGC）或改分布式压测（见第 15 篇），此时的测试结论一律作废。

## 4.2 迭代节奏控制

**闭合模型下迭代节奏完全由"响应时长 + sleep"决定，不加 sleep 的脚本等价于对服务端的紧循环攻击。**constant-vus 场景中，每个 VU 拿到响应立刻发起下一次迭代——服务越快，被打得越狠，同一段代码在不同性能的环境上产生的负载完全不同，测试结果不可复现。正确做法是在迭代尾部加 `sleep()` 模拟思考时间，让"每 VU 的迭代周期"成为一个受控常量：

```javascript
import { sleep } from "k6";

export default function () {
  // …业务请求…
  sleep(Math.random() * 3 + 1); // 思考时间 1-4 秒，节奏由脚本而非服务端决定
}
```

开放模型天然没有这个问题：到达率由调度器控制，sleep 只影响单次迭代时长（进而影响所需 VU 数），不影响整体发起速率。这也是到达率执行器"负载可复现"优势的一部分。

# 5. 常见误区

## 5.1 只追求高 VU

**VU 数不是吞吐，更不是容量结论。**闭合模型里吞吐 = 并发数 ÷ 单次迭代时长，服务端变慢时 VU 数不变而吞吐自动下降——"我们压到了 10000 并发"这句话本身不携带任何关于系统能力的信息。有效的容量结论只有一种表述："系统在 p95 < X ms、错误率 < Y% 的前提下，能持续承受 Z RPS"。VU 只是制造这个 RPS 的手段之一，而且在开放模型里它连手段都算不上，只是调度器的内部实现细节。

## 5.2 不区分开放/闭合模型

**模型选错时，测出的数字回答不了你想问的问题。**闭合模型的背压效应会系统性地掩盖排队延迟——系统卡住 10 秒，闭合模型里 VU 全都阻塞着，这 10 秒本该产生的请求样本直接消失，汇总出的 p99 反而好看（这就是协调遗漏，第 9 篇展开）。用闭合模型的数字去承诺"能扛住大促流量"，等于用一个会自动减压的负载去验证一个不会自动减压的世界。反过来，模拟固定席位类系统时硬套到达率模型，则会制造出该场景下不存在的瞬时并发。

# 6. 小结

| 执行器 | 模型 | 结束条件 | 关键参数 | 典型用途 |
|--------|------|---------|---------|---------|
| shared-per-vu-iterations | 闭合 | 共享迭代池耗尽 | vus、iterations、maxDuration | 批量回归、尽快采满样本 |
| per-vu-iterations | 闭合 | 每 VU 迭代数达标 | vus、iterations | CI 冒烟、数据分片场景 |
| constant-vus | 闭合 | duration 到期 | vus、duration | 恒定并发长稳 |
| ramping-vus | 闭合 | stages 走完 | startVUs、stages、gracefulRampDown | 并发阶梯加压 |
| constant-arrival-rate | 开放 | duration 到期 | rate、timeUnit、preAllocatedVUs、maxVUs | 恒定 RPS 容量验证 |
| ramping-arrival-rate | 开放 | stages 走完 | startRate、stages、preAllocatedVUs、maxVUs | 真实流量形状、找拐点 |

要点回顾：VU = 协程 + 独立 Goja 运行时，天然隔离；闭合模型控并发、开放模型控到达率，容量评估优先开放模型；到达率执行器靠 preAllocatedVUs/maxVUs 兑现 rate，`dropped_iterations` 非零说明负载发生器到顶；负载模型从 SLO 用 Little 定律倒推，时间结构按预热/稳态/降载三段设计。下一篇进入 scenarios 的多场景编排与五类典型负载曲线。
