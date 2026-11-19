# 1. 概述

熔断降级（Circuit Breaking / Degradation）用于在依赖服务不稳定时**快速失败**，避免调用方线程堆积，是防止**雪崩效应**的关键机制。

```text
正常状态 (CLOSED)                  熔断状态 (OPEN)
   ┌──────────────┐               ┌──────────────┐
   │   请求正常通过  │   触发熔断    │   直接拒绝    │
   │   统计指标     │ ──────────→  │   走 fallback │
   └──────┬───────┘               └──────┬───────┘
           │                               │
           │          半开状态              │ 熔断时长到
           │      (HALF_OPEN)              │
           │    ┌──────────────┐           │
           └─── │ 探测请求通过  │ ←────────┘
                │ 判断是否恢复  │
                └──────────────┘
```

## 1.1 熔断保护的是"调用人"

与流控（保护自己不被打爆）相反，熔断的立场是**保护自己不被下游拖死**，OPEN 期间做三件事：不发起下游调用（切断线程占用与故障放大）、立即抛 `DegradeException`（拒绝成本 ≈ 0，这正是"防止线程堆积"的机制——慢调用占线程 2s，熔断后占线程 0ms）、让调用方有时间恢复。Sentinel 术语里"熔断"与"降级"是一对动作：**熔断是决策（DegradeRule），降级是善后（fallback/blockHandler 回调）**，图里 OPEN 分支的"走 fallback"就是这对组合；这也是它和 Resilience4j 语境的映射关系（cb.run 的 fallback ≈ blockHandler 挂 DegradeException）。另一个架构事实：**熔断状态是单机进程内的**——每个实例独立统计、独立开关，不协商、不同步，10 台机器可能 3 台 OPEN 7 台 CLOSED（各自看到的下游切片不同），这是与集群流控相对的设计，排障时别拿一台的控制台状态推断全集群。

# 2. DegradeRule 核心属性

```java
public class DegradeRule extends AbstractRule {
    private double count;                  // 阈值
    private int timeWindow;               // 熔断时长(秒)
    private int grade = RuleConstant.DEGRADE_GRADE_RT;       // 熔断策略
    private int rtSlowRequestAmount = RuleConstant.DEGRADE_DEFAULT_SLOW_REQUEST_AMOUNT;
    private int minRequestAmount = RuleConstant.DEGRADE_DEFAULT_MIN_REQUEST_AMOUNT;
    private int statIntervalMs = 1000;    // 统计时长(ms)
}
```

## 2.1 三种熔断策略（grade）

| 值 | 常量 | 含义 | 默认最小请求数 | 适用场景 |
|----|------|------|--------------|---------|
| 0 | `DEGRADE_GRADE_RT` | **慢调用比例** | 5 | 下游响应变慢 |
| 1 | `DEGRADE_GRADE_EXCEPTION_RATIO` | **异常比例** | 5 | 下游错误率高 |
| 2 | `DEGRADE_GRADE_EXCEPTION_COUNT` | **异常数** | 5 | 下游异常数量 |

## 2.2 count 是个"多义词"：按 grade 换含义的字段表

DegradeRule 的字段语义强依赖 grade，这是熔断配置错误率最高的地方，一张表钉死：

| 字段 | grade=0 慢调用 | grade=1 异常比例 | grade=2 异常数 |
|------|---------------|-----------------|---------------|
| `count` | **慢调用 RT 门槛（ms）** | 异常比例阈值 (0~1) | 窗口内异常个数 |
| `slowRatioThreshold` | 触发熔断的慢调用占比 | 不使用 | 不使用 |
| `rtSlowRequestAmount` | 判定参与的最小慢调用数 | 不使用 | 不使用 |
| `minRequestAmount` | 窗口内总请求门槛 | 同左 | 同左 |
| `timeWindow` | OPEN 持续秒数 | 同左 | 同左 |
| `statIntervalMs` | 统计窗口（默认 1000ms） | 同左 | 同左 |

两个必须记住的细节：**RT 模式下 `slowRatioThreshold` 默认是 1.0**——只设 `count=200` 不设比例，意味着"100% 慢才熔断"，几乎永不触发，Dashboard 表单会帮你填 0.5 而 JSON 手写不会，这是"配了慢调用熔断没反应"的第一归因；`minRequestAmount` 是低流量保护闸——窗口内请求数不到 5，哪怕全错也不熔断（防止 1 个请求 1 个错=100% 比例的统计噪声），代价是低 QPS 资源熔断迟钝，必要时调小它并接受误熔断。另注意 grade 编号与 FlowRule 的 grade **不同序**（流控 1=QPS、0=线程数；熔断 0=RT、1=比例、2=数量），跨规则类型抄 JSON 时逐字段重读 §2.1。

# 3. 慢调用比例熔断（DEGRADE_GRADE_RT）

**触发条件**：在 `statIntervalMs` 统计周期内，**慢调用**（RT > `count` ms）的比例超过阈值。

```java
DegradeRule rule = new DegradeRule();
rule.setResource("getUser");
rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
rule.setCount(200);                     // 最大 RT 200ms
rule.setTimeWindow(10);                 // 熔断时长 10s
rule.setMinRequestAmount(5);            // 最少 5 个请求
rule.setSlowRatioThreshold(0.5);        // 慢调用比例 50%
```

**熔断逻辑**：

```text
statIntervalMs = 1000ms 内：
  请求总数 = 10
  慢调用数 (RT > 200ms) = 6
  慢调用比例 = 60% > 50% (slowRatioThreshold)
  → 触发熔断，进入 OPEN 状态
```

## 3.1 决策发生在"下一次请求"

RT 是事后指标：慢调用统计挂在 **exit 路径**（`2_architecture.md` §3.4，StatisticSlot.exit 才结算 RT 进窗口），而熔断判断挂在 **entry 路径**（DegradeSlot → `CircuitBreaker.tryPass` 读上一窗口的快照）。时序因此是："窗口 N 内慢比例达标 → N 期间最后一个请求 exit 时把 breaker 翻 OPEN → N+1 的第一个请求在 entry 被拒"——熔断生效有最多一个统计窗口的滞后，且**触发那一批请求本身全部执行成功了**（它们进来时还是 CLOSED）。这解释了两类"诡异"现象：下游抖动 800ms 内我方 QPS 先冲高后骤降（先放行后拒绝）；下游瞬时全挂时 RT 模式反应可能慢于异常数模式（错误在 exit 计数同理，但超时的 RT 要等 read-timeout 到点才结算——依赖超时才失败的故障，RT 熔断永远比异常熔断慢一拍，两种 grade 各配一条是成熟做法）。

# 4. 异常比例熔断（DEGRADE_GRADE_EXCEPTION_RATIO）

**触发条件**：每秒异常比例超过阈值。

```java
DegradeRule rule = new DegradeRule();
rule.setResource("createOrder");
rule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
rule.setCount(0.5);                     // 异常比例 50%
rule.setTimeWindow(10);
rule.setMinRequestAmount(5);
```

**注意**：`BlockException`（流控/熔断拒绝）不计入异常统计。只有**业务异常**计入。

## 4.1 异常统计的完整进出规则

"BlockException 不计入"的机制在 `2_architecture.md` §3.2：被拦请求走的是 `increaseBlockQps` 分支，根本不产生 error 事件——所以**限流风暴不会把自己熔断**（熔断器的输入只有"真正打到业务并抛出的异常"）。补三条同样关键的规则：**① 计不计什么异常由 TraceContext 决定**——`ErrorEntryFreeException` 级别的重复 exit 不重复计数；`@SentinelResource` 的 `exceptionsToIgnore` 让某类异常（参数校验 IllegalArgumentException 这类"不算故障"的）不进 error 统计，等于对熔断器的投票权白名单（`3_quick_start.md` §7.1）；**② exit 漏掉 = 异常失明**——业务抛异常后没走 exit（手写 try-catch 吞了、异步没注册回调），`setError` 的结算不发生，熔断器"下游明明在炸就是不熔断"，先查 exit 配对（§3.1 的结算路径）；**③ fallback 吞掉的异常不再参与统计**——异常已经计过一次，降级返回值不影响计数，不存在"降级成功把比例稀释"的问题，但也意味着熔断 OPEN 期间下游的真实恢复要靠半开探测而不是流量。

# 5. 异常数熔断（DEGRADE_GRADE_EXCEPTION_COUNT）

**触发条件**：每分钟异常数超过阈值。

```java
DegradeRule rule = new DegradeRule();
rule.setResource("queryDB");
rule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT);
rule.setCount(100);                     // 异常数 ≥ 100
rule.setTimeWindow(20);                 // 熔断 20s
rule.setMinRequestAmount(5);
```

## 5.1 【纠偏】"每分钟"是 1.6 之前的旧口径

现行 1.8.x 里异常数模式的窗口同样是 `statIntervalMs`（**默认 1 秒**），不是分钟——"每分钟异常数"的说法来自早期版本基于 `MinuteStatListener` 的实现，1.7 统一滑动窗口后已废。直接影响上例：`count=100` 配默认 1s 窗口意味着"单秒 100 错"才熔断，高 QPS 服务合理，中低流量场景基本永不触发——想表达"每分钟 100 个异常"应显式 `statIntervalMs=60000, count=100`，且意识到 timeWindow=20 的恢复探测也会变钝（窗口越大，OPEN→恢复的判定越滞后，慢调用/异常数模式在大窗口下都有这个通病）。异常数模式相对比例模式的价值在**绝对量兜底**：大流量接口比例被健康流量稀释（10000 QPS 里 500 错=5% 不触发 50% 阈值）但 500/s 已是事故量，两种模式各配一条是标准做法；小流量资源慎用纯异常数（一次发布全错可能凑不满 minRequestAmount，比例模式更灵敏）。

# 6. 熔断状态机源码

```java
// CircuitBreaker 状态枚举
public enum State {
    CLOSED,     // 正常：请求正常通过，统计指标
    OPEN,       // 熔断：直接拒绝，不调用下游
    HALF_OPEN   // 半开：允许探测请求，判断是否恢复
}

// 状态转换逻辑
public class CircuitBreaker {
    
    public boolean tryPass(Context context) {
        if (state == State.CLOSED) {
            return true;  // 正常通过
        }
        if (state == State.OPEN) {
            // 检查是否到了半开时间
            if (retryTimeoutArrived() && fromOpenToHalfOpen(context) ) {
                return true;  // 允许一个探测请求
            }
            return false;  // 熔断中，拒绝
        }
        // HALF_OPEN: 已经放行了一个探测请求
        return false;
    }
}
```

## 6.1 真实实现的三处细节（还原注记）

1.8.x 的 `CircuitBreaker.tryPass` 骨架与示例一致（CLOSED 放行 / OPEN 判时点 / 其余拒绝），值得补进心智模型的三个实现细节：**① 状态翻转的原子性**——`fromOpenToHalfOpen` 用 CAS 比较 `nextAllowedProbeTimestamp` 保证**同一时刻只有一个请求拿到探测资格**（拿到者把状态改 HALF_OPEN），其余并发请求继续被拒——半开期不是"放一小撮流量"而是**严格单请求探测**，与 Envoy/R4j 的 half-open 令牌数可配不同；**② CLOSED→OPEN 不发生在 tryPass**——breaker 自己订阅统计事件（`onReport`，即 §3.1 的 exit 结算路径回调），状态是"事件驱动翻转、entry 只读"，读操作零竞争；**③ 探测失败的重开是整窗口重来**——`fromHalfOpenToOpen` 直接把 `nextAllowedProbeTimestamp` 推到 now+timeWindow，**没有指数退避**，探测成功则 `fromHalfOpenToClosed` 并**重置统计窗口**（旧的慢调用快照清零，避免上一轮的账拖累新状态）。对比 Resilience4j 的 `permittedNumberOfCallsInHalfOpenState` 与自动退避，Sentinel 语义更"保守恢复"：探测窗口单请求、失败惩罚固定——对"下游时好时坏"的病，timeWindow 就是限流的节流阀，别设 1s。

# 7. 熔断恢复流程

```text
时间线：
t0: CLOSED（正常）
t1: 慢调用比例超过 50% → OPEN（开始熔断）
t1~t1+10s: 请求直接拒绝，走 fallback
t1+10s: 熔断时间到 → 尝试进入 HALF_OPEN
t1+10s: 放行一个探测请求
  → 成功 → CLOSED（恢复）
  → 失败 → OPEN（重新熔断）
```

## 7.1 "成功"的判定比你想的严格

探测请求活着返回不等于恢复：RT 模式的判据是**这次探测本身 RT < count**（下游从 2s 恢复成 1.5s 仍算失败——半开探的是"慢病"不是"死病"），异常模式的判据是不抛异常。若探测被**下游的流控/熔断拒绝**（429/降级响应），在调用方会走 `errorDecoder`/非 2xx 计为异常 → 探测失败重开——级联故障里"下游还在自我保护、上游探测反复失败"造成恢复互相拖后，解法见 §10.1 第 3 条勘误。另一句实话：OPEN 期间业务请求全走 fallback，**fallback 的质量决定熔断的业务观感**——返回空列表 vs 缓存旧值 vs 抛错给用户，是比调 timeWindow 收益更大的工作（Feign 侧的接线在 Feign 系列 `16_fallback.md`，Sentinel 原生在 `20_annotation_source.md`）。

# 8. 熔断 vs 降级 vs 流控

| 维度 | 熔断 | 降级 | 流控 |
|------|------|------|------|
| **触发条件** | 下游异常/慢调用 | 下游异常/超时 | 超过 QPS/线程阈值 |
| **行为** | 快速失败 | 返回兜底值 | 拒绝/排队 |
| **作用域** | 调用方侧 | 调用方侧 | 被调用方侧 |
| **恢复机制** | 半开状态探测 | 超时后重试 | 下一秒自动恢复 |
| **目的** | 防止线程堆积 | 提升用户体验 | 保护自身过载 |

## 8.1 表里"降级"是行为列不是规则列

注意 Sentinel 的规则体系里**没有独立的"降级规则"**——表第二列是 DegradeRule 触发后的善后动作（blockHandler/fallback），"熔断(决策)+降级(善后)"在 API 层是一对不可分组合；把它单列一栏是概念教学习惯，配规则时别去找 DegradationRuleManager。决策维度一句话版：**流控问"我该不该收这个请求"（对自己），熔断问"这个下游还配不配被调"（对依赖）**，两者在 SlotChain 里一前一后（Flow 先于 Degrade，`2_architecture.md` §3.3），同一请求理论上可能"流控放行、熔断拒绝"，异常类型分桶告警时两个前缀（FlowException/DegradeException）要分开看才能区分"我忙"和"下游病"。

# 9. YAML 配置

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        degrade:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            rule-type: degrade
```

Nacos JSON 示例：

```json
[
  {
    "resource": "getUser",
    "grade": 0,
    "count": 200,
    "timeWindow": 10,
    "minRequestAmount": 5,
    "slowRatioThreshold": 0.5,
    "statIntervalMs": 1000
  }
]
```

## 9.1 手写 JSON 的三条核对项

这份示例恰好是"配得对"的样子，对照 §2.2 各字段都有值且 `slowRatioThreshold` 显式给出。高频翻车三条：RT 模式漏 `slowRatioThreshold`（默认 1.0 永不熔断，§2.2）；异常数模式漏调 `statIntervalMs`（§5.1）；同资源想配多条规则——**1.8.x 支持**（`Map<String, List<CircuitBreaker>>`，任一 OPEN 即拒），1.6 及以前一条资源只活一个 breaker，老教程"同资源只能一条降级规则"的说法已过时，混用新旧资料时以 §6.1 的类结构为准。

# 10. 最佳实践

1. **慢调用优先**：生产环境优先使用慢调用比例熔断，异常比例作为补充
2. **合理阈值**：`count` 设为 P99 响应时间的 1.5~2 倍
3. **级联熔断**：上层服务熔断时间应**短于**下层服务
4. **区分核心/非核心**：核心接口熔断时长短（5s），非核心长（30s）
5. **fallback 设计**：始终提供有意义的 fallback 响应（缓存/兜底值/静态页面）

## 10.1 第 1、3 条各纠一处

**第 1 条方向对、理由要升级**："慢调用优先"的真理由是它比异常类信号**早**——下游病态通常先 RT 漂移（GC、慢查询、重试风暴）后错误冒头，RT 熔断是唯一在"错都没错但已经在变慢"阶段就动手的模式（§3.1 说它超时场景慢一拍，两者互补而非替代：慢调用+异常比例各一条，异常数在大流量做绝对量兜底，§5.1）。**第 3 条方向存疑，按恢复逻辑应为"下层短、上层长"**：下层资源离故障源最近、信号最准，应快速熔断快速试恢复（小 timeWindow）；上层聚合链路误杀成本高，窗口应更长且少配——反例推演在 §7.1：上层 timeWindow=3s 先开探测，打到还 OPEN 着的下层被拒绝、计为异常，上层重开 3s 再来一轮，形成**探测风暴**，下层恢复节奏被上游的急脾气反复打扰。若上层统计的是"入口整体健康度"（聚合多个依赖），长窗口还能避免单依赖抖动放大成入口震荡。第 2 条的"1.5~2×P99"注意口径：count 是**慢调用判定线**，不是超时线——它必须小于调用侧 read-timeout（否则慢调用永远统计不到，请求先超时走了异常路径），`RT count < timeout` 是 RT 熔断的隐藏前置约束。第 5 条的执行落点：fallback 里禁止再发起远程调用（降级打下游=雪崩放大器，Feign 系列 `16_fallback.md` 的同款纪律）。

# 11. 相关阅读

状态机与 `CircuitBreakerStrategy` 三种实现类的源码还原在 `17_circuit_breaker_source.md`；统计输入（RT/异常怎么进窗口）在 `18_statistics_engine.md`；Slot 位置与链上优先级在样板篇 `15_slot_chain_source.md`；与 Feign/RestTemplate/Gateway 的降级接线见 `13_feign_integration.md`、`12_gateway_integration.md`；排障速查"熔断器不打开/不恢复"在 `24_troubleshooting.md`。
