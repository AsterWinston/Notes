
# 1. 概述

SkyWalking Profiling 提供**方法级性能诊断**，回答 APM 传统数据回答不了的问题：端点耗时 200ms，**这 200ms 到底花在哪个方法、哪一行代码上**。普通 Trace 只埋了"有插件的地方"（HTTP、RPC、JDBC），业务方法内部是黑盒；Profiling 通过在指定条件下抓取**线程栈快照（Thread Dump Snapshot）**，把黑盒剖开：

```text
传统 Trace 只能看到：
  /api/order 端点耗时 200ms
    ├── Span: jdbc/select (100ms)
    └── Span: redis/get   (30ms)
    剩余 70ms 去哪了？—— 不知道

Profiling 剖开后：
  /api/order 耗时 200ms（含线程栈快照序列）
    ├── UserService.getById() 120ms
    │     └── JDBC select 100ms
    ├── AccountService.debit() 60ms
    └── OrderMapper.convert() 20ms   ← 热点现形（纯 CPU，无任何 Span 覆盖）
```

9.x 生态里有**三条剖析路线**，边界必须分清：

| 机制 | 载体 | 触发方式 | 产物 |
|------|------|---------|------|
| **Trace Profiling** | skywalking-java agent 内置 | OAP 下发任务，按 Trace 条件采样线程栈 | 单条 Trace 的线程栈快照序列（调用树） |
| **Async-profiler** | agent 8.12+ 内置 async-profiler | UI 创建按需任务（CPU/Memory/Lock） | 进程级火焰图 |
| **Continuous Profiling** | SkyWalking Rover（eBPF） | 进程级持续采样 | oncpu/offcpu、网络、内核级剖析 |

本文以 Trace Profiling 为主线（它是 agent 体系的一部分），async-profiler 与 Rover 在第 4 节展开关系。

# 2. Profiling 任务创建

任务由 UI 或 GraphQL 创建，本质是**给 OAP 注册一个匹配规则**：哪个服务、哪个端点、什么条件（耗时阈值/采样数/时长）满足后，让对应实例的 agent 开始抓栈：

```bash
# GraphQL mutation：针对 /api/order 端点，抓耗时 >100ms 的 Trace，任务 10 分钟
curl -s http://localhost:12800/graphql -H "Content-Type: application/json" -d '{
  "query": "mutation create($task: NewProfileTask!) { createProfileTask(input: $task) { id status errorLog } }",
  "variables": {
    "task": {
      "service": "order-service",
      "endpointName": "POST:/api/order",
      "startTime": "2024-01-15 1000",
      "duration": 10,
      "minDurationThreshold": 100,
      "dumpPeriod": 10,
      "maxSamplingCount": 50
    }
  }
}'
```

| 参数 | 说明 |
|------|------|
| `duration` | 任务持续时间（分钟），到点自动停止 |
| `minDurationThreshold` | Trace 最小耗时阈值（ms），低于此值不抓栈 |
| `dumpPeriod` | 线程栈快照周期间隔（ms），控制单条 Trace 内快照密度 |
| `maxSamplingCount` | 任务最大采样 Trace 数（agent 侧并行额度另由 `profile.max_parallel` 约束） |

**任务通过 gRPC 下发**：agent 周期性（`collector.get_profile_task_interval`，默认 20s）向 OAP 查询待执行任务（`ProfileTaskQueryService`），拿到任务后开始抓栈。这意味着任务生效有**最长约 20 秒 + 一个上报周期**的延迟，创建完立刻打流量可能抓不到，属正常时序。

> **注意**：任务只对**命中采样的 Trace** 生效。如果 agent 侧 `sample_n_per_3_secs` 采样率很低，大量 Trace 根本不会进入分析，Profiling 也抓不到——**排查 Profiling 无数据，先查采样率**。

# 3. 采样式线程栈剖析原理

Trace Profiling 不是方法级字节码计时（那开销无法接受），而是**对目标线程做离散栈快照**——与采样式 profiler（JFR/async-profiler 的 wall-clock 模式）同源的思想：

```text
Trace 命中任务条件（端点匹配 + 耗时 > 阈值）
  │
  ▼
Span 生命周期内，独立采样线程按 dumpPeriod 周期执行：
  ┌───────────────────────────────────────────┐
  │ StackTraceElement[] stack = thread.getStackTrace()  │ ← 快照点 1
  │ ... 10ms 后 ... snapshot                  │ ← 快照点 2
  │ ... 10ms 后 ... snapshot                  │ ← 快照点 3
  └───────────────────────────────────────────┘
  快照序列附加进当前 TraceSegment，随 segment 一起上报
  │
  ▼
OAP 将快照序列还原为"时间轴上的方法调用树" → UI 展示
```

三个关键推论由此而来：**其一**，`getStackTrace()` 会触发 safepoint 式的全栈遍历，单线程开销约几十微秒，**只对被采样的那一条请求生效**，不影响同实例其他流量——这就是"agent 触发"的低成本基础。**其二**，采样密度决定分辨率：10ms 周期看不到 5ms 的方法，热点方法被"看到"的概率正比于其占用时间，短小高频方法的栈占比在快照序列里会被低估。**其三**，产出是**单 Trace 视角**的调用树而非全进程火焰图——它回答"这一条慢请求为什么慢"，不回答"整个进程 CPU 都被谁吃了"，后者要等 Continuous Profiling。

# 4. 与 async-profiler / Rover 的关系

方法级、进程级的**聚合火焰图**有两条路：**agent 内置 async-profiler**（skywalking-java 8.12 起集成，由 OAP 下发 CPU/Memory/Lock 按需任务，在目标 JVM 上短时运行 async-profiler 并回传折叠栈）与 **Rover 的 eBPF 连续剖析**（独立部署的 K8s DaemonSet/Sidecar 组件，内核级 oncpu/offcpu、网络剖析）：

```text
┌──────────────────────────┐        ┌────────────────────┐
│ 业务 Pod                  │        │ SkyWalking Rover    │
│  ┌────────────────────┐  │ attach │  (每节点/每 Pod)     │
│  │ JVM                │◄─┼────────┤ async-profiler      │
│  │  · CPU (itimer/    │  │ ondemand│  · cpu/alloc/lock  │
│  │    perf_events)    │  │        │  · 火焰图 jfr/折叠栈 │
│  └────────────────────┘  │        │ eBPF probes         │
│  skywalking-java agent   │        │  · oncpu/offcpu     │
│  (trace profiling 任务)   │        │  · 网络/TLS          │
│  内置 async-profiler      │        └─────────┬───────────┘
└───────────┬──────────────┘                   │ gRPC profile 流
            │ gRPC segment                     ▼
            ▼                          OAP (11800)
                  OAP  →  UI: Trace 调用树 / 火焰图
```

分工一句话记牢：**Trace Profiling 抓"某条慢 Trace 的栈序列"，async-profiler/Rover 抓"进程一段时间的 CPU/内存/锁分布"**。Rover 的 eBPF 路径能剖到内核态与 off-cpu（锁等待、I/O 等待）——这是纯用户态 Java 探针做不到的；agent 内置 async-profiler 则胜在零额外部署，两者产物在 UI 同属 Profiling 菜单。

# 5. 开销评估

| 机制 | 常态开销 | 任务期间开销 | 说明 |
|------|---------|-------------|------|
| Trace Profiling 栈快照 | 0（无任务时零开销） | 单条被采样请求 +5–15% | `getStackTrace()` 快照成本随栈深增长 |
| agent 内置 async-profiler | 0（按需启停） | 进程 CPU 1–3% | perf_events 事件驱动，itimer 模式更高 |
| Rover eBPF | 持续 1% 内 | — | 内核侧过滤后上报，采样率可调 |

**设计考量**：

- **为什么不做全方法字节码计时**：对每个方法插桩计时需要增强所有类，JIT 内联失效 + 计时本身开销可达两位数百分比，且栈深大时误差放大；采样式以"概率正确"换"成本可控"
- **快照不是免费的**：`dumpPeriod` 设 1ms 会把快照频率拉满，单请求栈遍历次数 = 耗时/周期，一条 200ms 请求将产生 200 次快照——阈值与周期组合要有节制
- **对 GC 的影响路径**：栈快照持有 `StackTraceElement[]` 数组，高频快照在 young gen 制造垃圾，配合上报缓冲可能抬高 YGC 频率，堆小且流量大的实例慎用短周期

# 6. 性能剖析可视化

UI 的 Profiling 报告提供两种视图：按单条 Trace 的**栈快照时间轴**与聚合后的**调用树**：

```text
┌──────────────────────────────────────────────┐
│  Profile: POST:/api/order  snapshot #3/#7      │
│  OrderController.create          [200ms]       │
│  ├─ OrderService.submit           [95ms]       │
│  │   ├─ InventoryClient.deduct    [60ms]  ← 快照命中 6/7 次
│  │   └─ PriceCalculator.calc      [30ms]       │
│  ├─ OrderMapper.convertToEntity   [25ms]  ← 无 Span 覆盖的黑盒
│  └─ Json序列化                     [15ms]       │
└──────────────────────────────────────────────┘
```

读图要点：快照命中次数 ≈ 时间占比，命中密集的帧就是热点；"无 Span 覆盖的黑盒"正是 Profiling 的增量价值——它出现在两个 Exit Span 的间隙里，Trace 视图只能看到"空白时间"。

# 7. 使用场景：线上热点定位

典型的三步漏斗排障路径——**指标告警 → Trace 定位端点 → Profiling 定位方法**：

```text
① service_resp_time 告警 / TopN 慢端点榜单
        │ 锁定 POST:/api/order P99 = 800ms
        ▼
② Trace 查询：按端点过滤慢请求，看 Span 树
        │ JDBC 合计 300ms，仍有 500ms 说不清
        ▼
③ 对该端点创建 Profiling 任务（阈值 500ms）
        │ 抓到栈快照：JSON 反序列化 + 正则编译占 400ms
        ▼
④ 修复验证：同一任务再跑一轮对比快照分布
```

**经验边界**：CPU 密集热点（序列化、正则、加解密、循环计算）Profiling 立竿见影；**锁竞争/线程池排队/I/O 等待**这类 off-cpu 问题，in-process 快照只能看到"栈停在哪"却分不出 running/blocked 的占比细分，此时优先上 Rover 的 lock/offcpu 剖析或直接 jstack 对照。

> **行业实践**：把"Trace 锁定端点 → Profiling 抓栈 → 修复 → 同参数任务复测"固化为团队 SOP，热点类工单按此闭环处理；Profiling 任务默认设 10 分钟自动过期，避免遗留任务长期占住 `profile.max_parallel` 并行额度导致新任务排队不生效。

# 8. 自定义方法追踪

不想起任务、只想永久观察某几个关键方法时，用 toolkit 的 `@Trace` 注解生成 **Local Span**，把方法耗时常规化纳入 Trace（这才是"每请求都记录"的机制，注意开销常驻）：

```java
@Service
public class UserService {

    @Trace(operationName = "getUserById")     // 生成 Local Span 记录方法耗时
    @Tag(key = "userId", value = "arg[0]")    // 入参写入 span tag
    @Tag(key = "result", value = "returnedObj.name")
    public User getUserById(Long userId) {
        return userMapper.selectById(userId);
    }
}
```

`@Trace` 依赖 `apm-toolkit-trace` 包 + agent 的注解插件生效（两者缺一不可，只引包不挂 agent 时注解是无操作的）。与 Profiling 的取舍：**高频方法别用 @Trace**（每请求一个 span 的成本叠加 `span_limit_per_segment` 配额），低频关键节点才值得常驻埋点。

# 9. 最佳实践

1. **按需开启、用完即停**：任务都有 duration 上限，不要把 Profiling 当常驻监控
2. **阈值筛选**：`minDurationThreshold` 过滤正常耗时，只抓"值得剖"的慢 Trace
3. **采样控制**：限制 maxSamplingCount 与 dumpPeriod，避免 OAP 存储与目标实例压力
4. **先 Trace 后 Profiling**：先在链路里锁定可疑端点，再对该端点起任务，别对整个服务盲剖
5. **对照基线**：修复前后各跑一次同参数任务，用快照命中分布的变化验证优化效果
6. **进程级热点用 async-profiler/Rover**：全进程 CPU/内存/锁火焰图交给按需 async-profiler 任务或 continuous profiling，三套工具别互相冒充

# 10. 小结

| 主题 | 要点 |
|------|------|
| 三条路线 | Trace Profiling（单 Trace 栈快照，按需任务）、agent 内置 async-profiler（进程火焰图）、Rover eBPF（连续/on-cpu/off-cpu） |
| 原理 | 离散 `getStackTrace()` 快照 ≈ 采样 profiler，命中概率正比于耗时 |
| 触发 | OAP 任务经 gRPC 下发，agent 周期查询生效；采样率过低会抓不到 |
| 开销 | 无任务零开销；任务期单请求 +5–15%，快照周期是唯一旋钮 |
| 场景 | CPU 热点立竿见影；off-cpu/锁竞争用 async-profiler 或 Rover；`@Trace` 管常驻关键节点 |

剖析解决"方法内为什么慢"，下一篇《9. Spring Boot 集成实战》回到工程现场，把 agent 在 Spring Boot 上挂稳、配全。
