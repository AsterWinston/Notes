# 1. 整体架构

Sentinel 的架构分为**客户端核心库**和**Dashboard 控制台**两部分：

```text
┌─────────────────────────────────────────────────────────┐
│                    Sentinel Dashboard                    │
│              (规则管理 / 监控可视化 / 集群流控)           │
└──────────────────────┬──────────────────────────────────┘
                       │ 规则推送 / 拉取
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   Sentinel Client                        │
│  ┌───────────────────────────────────────────────────┐  │
│  │                 ProcessorSlotChain                 │  │
│  │  ┌──────────┬──────────┬──────────┬──────────┐   │  │
│  │  │NodeSel   │Cluster   │ LogSlot  │Statistic │   │  │
│  │  │Slot      │Builder   │          │Slot      │   │  │
│  │  ├──────────┼──────────┼──────────┼──────────┤   │  │
│  │  │System    │Authority │FlowSlot  │Degrade   │   │  │
│  │  │Slot      │Slot      │          │Slot      │   │  │
│  │  └──────────┴──────────┴──────────┴──────────┘   │  │
│  └───────────────────────────────────────────────────┘  │
│                           │                              │
│  ┌────────────────────────▼───────────────────────────┐  │
│  │                   Statistics Engine                  │  │
│  │        (LeapArray 滑动窗口 + MetricBucket 指标)     │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## 1.1 控制面与数据面：两个进程的分工

Dashboard 是独立的 Spring Boot jar，客户端是嵌进业务进程的库——这是标准的**控制面/数据面分离**。两条通信方向要分清：

| 通信 | 方向 | 通道 | 失败后果 |
|------|------|------|----------|
| 心跳注册 | Client → Dashboard | HTTP POST `/dashboard/register-app`（transport 模块） | Dashboard 看不到应用 |
| 指标拉取 | Dashboard → Client | Command Center 8719 `/metrics` | 监控曲线断点 |
| 规则下发（裸模式） | Dashboard → Client | Command Center `/v1/flow/rules` 等 | 规则不落盘，重启丢失 |
| 规则下发（生产模式） | Dashboard → Nacos → Client | 配置中心 watch | Dashboard 挂了规则照样生效 |

关键的韧性结论：**数据面零依赖控制面**。Dashboard 宕机、Nacos 短暂失联，客户端内存里的规则和统计照常工作（fail-open）——Sentinel 的所有治理动作在业务进程本地完成，检查路径上没有一次远程调用。这决定了它可以"控制台平时不开、出事再开"的运维姿态，也解释了为什么裸模式下改的规则重启就没了：控制台本来就不是存储，只是内存的遥控器。

## 1.2 客户端内部分层在解决什么

三层各司其职且依赖方向单一：**SlotChain 做决策 → 决策只读 Statistics Engine → 规则表由各自 Manager 静态持有**。两个设计取舍值得注意：其一，规则 Manager（`FlowRuleManager` 等）全是**进程级静态单例**，好处是零依赖嵌入、调用路径无容器查找，代价是"一个 JVM 一套规则表"——同一进程跑多租户/多应用上下文的场景规则会互串，测试用例之间也要记得 `FlowRuleManager.loadRules(Collections.emptyList())` 清场；其二，链上每个 Slot 检查时读的都是统计引擎的数据，所以**统计准确性直接决定限流熔断的正确性**（高并发下 LeapArray 的近似窗口语义，`18_statistics_engine.md`），"限流不准"类问题十有八九要下沉到统计引擎找根因。

# 2. 核心概念模型

## 2.1 Resource（资源）

Resource 是 Sentinel 中最核心的概念，代表需要保护的代码点。每个 Resource 由一个唯一的 `resourceName` 标识。

```java
// 方式一：API 硬编码
try (Entry entry = SphU.entry("getUser")) {
    // 业务逻辑
    return userService.getUser(id);
} catch (BlockException e) {
    // 被流控/降级处理
    return "服务繁忙，请稍后再试";
}

// 方式二：注解声明（推荐）
@SentinelResource(value = "getUser", blockHandler = "handleBlock")
public User getUser(Long id) { ... }
```

资源名是**唯一寻址键**，两条隐性纪律：同一应用内不要出现两种命名风格（`/user/get` 与 `getUser` 会裂成两个资源、两份统计）；带参 URL 必须收敛成模板名——裸 `SphU.entry` 用查询串当资源名会让资源树无限膨胀（每个唯一名一个 ClusterNode，内存泄漏 + 控制台失焦），Web 适配器的 context-path 归一化正是为此（`10_spring_cloud_integration.md`）。

## 2.2 Context 与 Entry

```text
Context（调用上下文）
  ├── EntranceNode（入口节点，统计全局流量）
  │     └── Entry (SphU.entry 返回)
  │           ├── Node (DefaultNode，每个 Context 下资源节点)
  │           └── ClusterNode（资源全集群统计，所有 Context 汇总）
  └── Entry.exit() 退出时释放
```

| 对象 | 职责 | 生命周期 |
|------|------|---------|
| **Context** | 一次请求的调用上下文，包含调用链信息 | 请求开始创建，结束销毁 |
| **Entry** | 一次资源调用记录，关联 Node | entry() 创建，exit() 销毁 |
| **Node** | 资源的统计节点，记录 RT/QPS/异常数 | 资源首次调用创建，常驻内存 |
| **ClusterNode** | 汇聚所有 Context 中同名资源的统计 | 全局唯一 |

## 2.3 Rule（规则）

所有防护策略以 Rule 对象表达，各规则类型独立管理：

```java
// FlowRule — 流量控制规则
FlowRule rule = new FlowRule();
rule.setResource("getUser");    // 资源名
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS 模式
rule.setCount(100);             // 阈值 100 QPS
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT); // 快速失败
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

规则分类：

| 规则类型 | 配置类 | 管理类 | 核心场景 |
|---------|--------|--------|---------|
| 流控 | `FlowRule` | `FlowRuleManager` | QPS/线程数限制 |
| 熔断 | `DegradeRule` | `DegradeRuleManager` | 异常/慢调用熔断 |
| 系统 | `SystemRule` | `SystemRuleManager` | 系统级自适应 |
| 授权 | `AuthorityRule` | `AuthorityRuleManager` | 黑白名单 |
| 热点 | `ParamFlowRule` | `ParamFlowRuleManager` | 热点参数限流 |

## 2.4 同一个资源的四个统计视角

§2.2 表里只讲了两种 Node，完整图谱其实有四层——这是理解"关联流控/来源限流/系统保护为什么能各管各的"的钥匙：

```text
machine-root (EntranceNode, 全局根)
  └── Context: web-unit (EntranceNode)
        └── getUser (DefaultNode ← 资源×context 粒度)
  └── Context: rpc-source (EntranceNode)
        └── getUser (另一个 DefaultNode，统计独立)

getUser (ClusterNode ← 资源全局汇总，仅一份)
  ├── OriginNode: appA (调用来源维度)
  └── OriginNode: appB
```

| 节点 | 粒度 | 谁在读它 |
|------|------|----------|
| EntranceNode | 每个 Context 一个 | 系统规则（machine-root 聚合全局入口流量/负载） |
| DefaultNode | 资源 × Context | 链路上下文流控（`relateDataId`）、context 维度监控 |
| ClusterNode | 资源全局（单机内） | 普通 QPS/线程数流控、熔断降级、控制台簇点链路 |
| OriginNode | 资源 × 来源(origin) | 来源区分限流（limitApp）、授权黑白名单 |

控制台排障直接受益于此：簇点链路里看到的曲线是 ClusterNode，"按来源限流不生效"时要查的是 OriginNode 的 origin 值对不对（origin 由 `ContextUtil.enter(name, origin)` 的第二参或 Web 适配器的 origin parser 决定，默认取 `x-sentinel-request-origin` 头）——同一资源四个数字不一致是**特性不是 bug**。

# 3. ProcessorSlotChain — 责任链模式

## 3.1 架构原理

所有规则检查通过**责任链**串联执行，每个 Slot 负责单一职责：

```text
SphU.entry(resourceName)
  │
  ▼
ProcessorSlotChain.entry()
  │
  ├── NodeSelectorSlot       # 构建 Node 节点和调用树
  ├── ClusterBuilderSlot     # 构建 ClusterNode（全局统计节点）
  ├── LogSlot                # 记录拦截日志
  ├── StatisticSlot          # 先执行后续 Slot，再根据结果统计指标
  ├── SystemSlot             # 系统保护检查
  ├── AuthoritySlot          # 授权检查（黑/白名单）
  ├── FlowSlot               # 流量控制检查
  ├── DegradeSlot            # 熔断降级检查
  └── ParamFlowSlot          # 热点参数流控检查
```

## 3.2 关键设计

**StatisticSlot 的特殊位置**：它**先调用后续 Slot**，再根据结果统计：

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper,
                  DefaultNode node, int count, boolean prioritized, Object... args) {
    try {
        // 1. 先让后续 Slot 执行（流控、熔断等检查）
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
        // 2. 通过 → 统计请求通过
        node.increaseThreadNum();
        node.addPassRequest(count);
    } catch (BlockException e) {
        // 3. 被拦截 → 统计阻塞
        node.increaseBlockQps(count);
        throw e;
    } catch (Throwable e) {
        // 4. 异常 → 统计异常
        context.getCurEntry().setError(e);
        throw e;
    }
}
```

## 3.3 顺序的三个实操含义

上图的顺序不是文档图画出来的，是 **1.8.x `DefaultSlotChainBuilder` 硬编码**的（8 个核心 Slot 逐个 `addLast`，扩展 Slot 如 `ParameterFlowSlot` 经 `SpiLoader.loadOrdered(ProcessorSlot.class)` 追加——早期版本核心链也走 SPI+排序，文档里 Authority 在 System 前的老图即源于此，1.8 实际以代码为准）。顺序带来的行为差异有三处要记住：**① 谁先抛异常**——链上 System→Authority→Flow→Degrade→ParamFlow，一个请求同时违反系统规则和流控规则时，拿到的是 `SystemBlockException`，控制台与日志里的异常类型按"最靠前者"归因，配置监控告警按异常类分桶时要考虑这一优先级；**② 被拦截的请求不计通过**——StatisticSlot 在检查类 Slot 的外层，后续抛 `BlockException` 只进 block 计数，所以熔断器的异常比例统计不含被拦掉的请求（拦截风暴不会"把自己熔断"，`17_circuit_breaker_source.md`）；**③ 检查类 Slot 只读不写统计**——FlowSlot 读 ClusterNode 的 pass QPS 做判断，它自己不加计数，写入全在 StatisticSlot，这保证"统计口径唯一"，自定义 Slot 也应当守这条纪律。

## 3.4 exit 方向：链是双向的

`entry()` 顺链而下，`entry.exit()` 同样要过一遍链（`StatisticSlot.exit` 递减并发线程数、结算 RT 进滑动窗口；`NodeSelectorSlot` 弹调用树栈帧）。exit 的重要性常被低估：**漏 exit 等于线程数泄漏**——grade=THREAD 的并发线程数限流会一路涨到 count 后把所有请求拦死，且"重启才恢复"，这是新手最常见的 Sentinel 事故形态（SphO 的 while 循环写法和手动 entry 后异常路径漏 exit 都会中招，`4_flow_control.md` §5.2）。exit 与 entry 必须同线程配对，异步场景把收尾动作挂进 `asyncEntry` 的 completion 回调，别让 exit 跟着业务线程跑丢。链式细节与 `DefaultProcessorSlotChain` 的双向链表实现见样板篇 `15_slot_chain_source.md`。

# 4. 滑动窗口统计引擎

## 4.1 LeapArray 数据结构

Sentinel 使用 `LeapArray` 实现高效滑动窗口：

```text
当前时间线 ────────────────────────────────────►
┌──────────┬──────────┬──────────┬──────────┐
│ Window 0 │ Window 1 │ Window 2 │ Window 3 │  ← 每个窗口 500ms
│ (过期)   │ (当前)   │ (未来)   │ (未来)   │
└──────────┴──────────┴──────────┴──────────┘
             │
             ▼
       ┌──────────────┐
       │ MetricBucket  │  ← 存储 pass/block/error/rt 等指标
       │ long[] counters│
       └──────────────┘
```

## 4.2 窗口生命周期

```java
// 样本数 = intervalInMs / windowLengthInMs
// 例如：1 秒统计周期 / 500ms 窗口 = 2 个窗口
public class LeapArray<T> {
    protected int windowLengthInMs;   // 单个窗口长度 (默认 500ms)
    protected int sampleCount;        // 窗口样本数 (默认 2)
    protected int intervalInMs;       // 统计周期 (默认 1000ms)
    protected final AtomicReferenceArray<WindowWrap<T>> array;
}
```

**窗口滚动逻辑**：当当前时间超出当前窗口范围时，复用或重置旧窗口，避免频繁创建对象。

## 4.3 图里"未来窗口"的澄清与选型账

原图把 Window 2/3 标成"未来"，需要纠正一下读法：**数组里任何时刻都不存在未来数据**，那只是环形缓冲的空闲槽位——`currentWindow()` 按 `time / windowLengthInMs % sampleCount` 定位，落在过期槽位就 `resetTo` 原地复用。环形复用是这个引擎的魂：1 秒窗口 ×2 样本的 MetricBucket 对象在进程生命周期内基本不重新分配，统计路径的 GC 压力近似为零。为什么是 500ms×2 而不是 1s×1 或 100ms×10，一本账说尽：

| 方案 | 精度 | 内存 | 遍历成本 |
|------|------|------|----------|
| 1s×1（固定窗口） | 有临界翻倍问题（两窗口边缘 6 倍流量） | 最小 | O(1) |
| **500ms×2（默认）** | 秒级误差 ≤2 倍窗口粒度内的突发 | 16 字节×桶数 | O(2) |
| 100ms×10 | 更平滑 | 5 倍槽位 | O(10) 每次读汇总 |

拆成子窗口的本质是用"窗口粒度的突发容忍"换"临界效应的连续统计"——所以 Sentinel 的 QPS 限流**天生允许秒边界附近短促超标**（`16_flow_control_source.md` 里 `passChecking` 对这一点有显式注释）；对精度敏感的场景用 Warm-Up/RateLimiter 平滑策略兜底，而不是改小窗口。完整实现（`ArrayBucket`、MetricBucket 位压缩、滑动窗口读锁）在 `18_statistics_engine.md`。

# 5. 调用链路模型

```java
// 构建调用树
ContextUtil.enter("user-center", "appA");
Entry e1 = SphU.entry("getUser");
  Entry e2 = SphU.entry("getUserDetail");
    Entry e3 = SphU.entry("queryDB");
    e3.exit();
  e2.exit();
e1.exit();
ContextUtil.exit();
```

对应调用树：

```text
machine-root (EntranceNode)
  └── user-center (EntranceNode)
        └── getUser (DefaultNode)
              └── getUserDetail (DefaultNode)
                    └── queryDB (DefaultNode)
```

## 5.1 关联流控的调用树

调用树使得**关联流控**成为可能：当 `/order/create` 资源达到阈值时，可以限制 `/order/query` 的流量。

## 5.2 Context 是线程绑定的

`ContextUtil.enter` 把 Context 压进 **ThreadLocal**（`NodeSelectorSlot` 靠它拼调用树），两个连锁后果：① 嵌套 `SphU.entry` 不显式 enter 时，会**自动创建一个名为 `sentinel_default_context` 的兜底 Context**——所有"没人管"的资源挤在同一个 EntranceNode 下，context 维度流控与监控失真，规范做法是在入口（Web Filter/RPC server filter）统一 enter/exit；② **异步线程不继承 Context**，`@Async`/线程池/响应式链路上的 `SphU.entry` 全部落到默认 context，且 exit 跨线程会抛错——要么显式在目标线程 `ContextUtil.enter(name, origin)` 复刻上下文（origin 也要带过去，否则来源限流/黑白名单在异步段直接失明），要么接受降级为默认 context 的统计口径。§5.1 的关联流控读的正是这棵树上**另一资源在其所属 context 的 DefaultNode** 的实时占用，跨线程丢 context 会让它"永远看不到对面"。

# 6. 规则推送模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **原始模式** | API 代码硬编码规则 | 开发测试 |
| **Pull 模式** | 客户端定时从数据源拉取规则 | 简单生产环境 |
| **Push 模式** | 数据源主动推送规则到客户端 | 生产环境（推荐） |
| **控制台推送** | Dashboard 通过 API 推送 | 配合控制台使用 |

推荐的生产架构：**Nacos + Sentinel Dashboard Push 模式**，规则持久化到 Nacos，Dashboard 修改后推送至客户端。

## 6.1 生产闭环的方向图与两个陷阱

完整的规则流转其实是一条环，两个方向都要通：

```text
① 改规则：运维 → Dashboard → Nacos 配置 →(watch 推送)→ 客户端 RuleManager
② 兜底：Nacos 不可达 → 客户端用内存中最后一次规则继续工作（fail-open）
```

陷阱一：**Dashboard 的"控制台推送"与"Nacos 持久化"不能混用同一资源**——裸推只改内存，Nacos 里的旧版本在下次 watch 回调时把你的紧急改动静默覆盖回去（`loadRules` 是**整表替换**语义，不是合并）；SCA 2023.x 的 `SentinelDashBoardDataSourceServiceImpl`/mode 为 `async` 的改造能写回，但多数团队直接用默认：控制台改 → 落 Nacos → 客户端收敛，**控制台裸推当临时开关用完必须回写 Nacos**。陷阱二：Pull 模式的"定时"意味着生效延迟（默认 `property.write-data-source` 拉取周期秒级），紧急止血时先想到链路的传播时延，别把治理期望压在分钟级轮询上。各数据源的代码接线与规则文件格式在 `14_dynamic_rules.md`。

## 6.2 系列阅读地图

```text
本篇 2_architecture（概念模型与分层）
  ├─ 用：3_quick_start → 4~8 各规则 → 9_dashboard → 10~13 集成
  ├─ 原理纵深：15_slot_chain_source（链）→ 18_statistics_engine（数据）
  ├─ 规则纵深：16_flow_control_source / 17_circuit_breaker_source / 20_annotation_source
  ├─ 治理：14_dynamic_rules（规则流转）→ 21~24（观测/调优/实践/排障）
  └─ 面试：25_interview_qa
```

本篇的四个统计节点（§2.4）和 Slot 顺序（§3.3）是全系列引用最多的两张"地图底图"：后面每一篇规则类文档都是往这两张图上挂自己的检查逻辑。
