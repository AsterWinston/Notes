# 1. 概述

流量控制（Flow Control）是 Sentinel 最核心的能力，通过**QPS**或**并发线程数**两个维度限制资源访问。

```text
                    ┌───────────────┐
   请求 ──────────→ │   FlowSlot    │
                    │               │
                    │  FlowRule     │
                    │  Checker      │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          通过 (PASS)   等待 (WAIT)   拒绝 (BLOCK)
```

## 1.1 两种 grade 读的是两种数据

图里"FlowRule Checker"背后，QPS 与线程数模式的判断依据完全不同源：**QPS 模式**读统计引擎滑动窗口内"已通过 + 正在到达"的准实时计数（`2_architecture.md` §4 的 MetricBucket），有窗口粒度的近似；**线程数模式**读 DefaultNode/ClusterNode 上的**实时 AtomicInteger**（entry 加、exit 减），无窗口、瞬时精确，但要求 exit 严格配对。这个差异决定了故障表现不同：QPS 模式在窗口边缘允许短促超标；线程数模式一旦漏 exit 就**永久虚增**，表现为"限流阈值莫名打满、重启才恢复"（`2_architecture.md` §3.4）。选型时的第一问因此是"我限的是速率还是容量"——保护下游吞吐选 QPS，保护自己线程池水位选线程数，两者可以共存（同资源多条 rule，任一触发即拦）。

# 2. FlowRule 核心属性

```java
public class FlowRule extends AbstractRule {
    private int grade = RuleConstant.FLOW_GRADE_QPS;   // 阈值类型
    private double count;                                // 阈值
    private int strategy = RuleConstant.STRATEGY_DIRECT; // 流控模式
    private String refResource;                          // 关联/链路资源名
    private int controlBehavior = RuleConstant.CONTROL_BEHAVIOR_DEFAULT; // 流控效果
    private int warmUpPeriodSec = 10;                    // 预热时长(秒)
    private int maxQueueingTimeMs = 500;                 // 排队最大等待(ms)
}
```

## 2.1 阈值类型（grade）

| 值 | 常量 | 含义 | 适用场景 |
|----|------|------|---------|
| 1 | `FLOW_GRADE_QPS` | 每秒请求数（**默认**） | 大多数场景 |
| 0 | `FLOW_GRADE_THREAD` | 并发线程数 | 防止线程耗尽 |

**QPS vs 线程数隔离**：

```java
// QPS 限流：适合瞬时流量控制
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);  // 每秒 100 个请求

// 线程数限流：适合慢调用场景，防止线程堆积
rule.setGrade(RuleConstant.FLOW_GRADE_THREAD);
rule.setCount(20);   // 最多 20 个并发线程
```

**设计建议**：优先使用 QPS 限流。线程数限流仅当接口耗时差异巨大（慢接口可能长时间占用线程）时考虑。

## 2.2 线程数阈值的换算与盲区

线程数模式不是"随便设个并发上限"，它有一个必须会用的换算——**利特尔法则：稳态并发线程数 ≈ QPS × RT**。接口 100 QPS、RT 50ms，常态占用 5 个线程；设 count=20 意味着允许 RT 恶化 4 倍（或流量翻 4 倍）才触碰——这就是"下游变慢时优先限住线程堆积"的机制：慢调用场景 count 应贴着 `入口线程池容量 × 0.6` 设，让 Sentinel 比 Tomcat 线程池先饱和，降级动作发生在你手里而不是容器手里。盲区也要知道：它限的是**本资源占用的线程数**，排队效果（§4.3）中睡眠等待的线程同样计数，两者叠加时预算要合并算；集群流控模式下线程数 grade 不支持（token server 只发速率令牌，`16_flow_control_source.md`）。

# 3. 流控模式（strategy）

## 3.1 直接模式（STRATEGY_DIRECT）

对当前资源直接限流，**默认模式**：

```text
请求 → getUser → FlowSlot 检查 → 通过/拒绝
```

```java
rule.setStrategy(RuleConstant.STRATEGY_DIRECT);
```

## 3.2 关联模式（STRATEGY_RELATE）

当关联资源达到阈值时，限流当前资源：

```text
/order/create (触发资源) → 达到 100 QPS
    → 限流 /order/query (被保护资源)
```

```java
// 当 /order/create 超过 100 QPS 时，限制 /order/query
FlowRule rule = new FlowRule("orderQuery");
rule.setStrategy(RuleConstant.STRATEGY_RELATE);
rule.setRefResource("orderCreate");  // 关联的高优先级资源
rule.setCount(100);
```

**适用场景**：写操作优先级高于读操作。当写压力大时，牺牲读来保护系统。

## 3.3 链路模式（STRATEGY_CHAIN）

只限制从某个入口链路进入的请求：

```text
入口A → getUser → (被限流)
入口B → getUser → (正常放行)
```

```java
// 只限制入口 "orderEntrance" 进入的 getUser 请求
ContextUtil.enter("orderEntrance", "orderApp");
try (Entry e = SphU.entry("getUser")) { ... }

FlowRule rule = new FlowRule("getUser");
rule.setStrategy(RuleConstant.STRATEGY_CHAIN);
rule.setRefResource("orderEntrance");  // 入口资源名
```

## 3.4 relate 与 chain 的匹配对象不同，是最高频错配点

两种模式都写 `refResource`，但查找的东西不是一类：**relate 的 refResource 是"兄弟资源名"**——checker 拿它的 ClusterNode（全局统计）当触发信号，所以两个资源必须**同进程**且被真实调用过（没统计=永远不触发），跨服务的资源名写了也白写；**chain 的 refResource 是"入口 Context 名"**——checker 读的是"当前资源在**那个 context 的 DefaultNode**"上的通过流量（`2_architecture.md` §2.4 的节点粒度在这里兑现），Web 场景 context 名是归一化后的入口 URL（`10_spring_cloud_integration.md` 的 normalize 规则），不是"GET:xxx"资源名。两个连锁坑：链路上 Context 丢失（异步线程，§3.3 例子的 enter 没跟到工作线程）时 chain 规则**整体失明**；chain 限的是"该入口贡献的那份流量"，不是"入口本身的总量"——想限入口总量应该把规则直接配在入口资源上（Web filter 资源天然就是一个 context 的 entrance，多数"按入口限流"需求用 `GET:/api/hello` 直接配 direct 即可，不需要 chain 模式）。

# 4. 流控效果（controlBehavior）

## 4.1 快速失败（CONTROL_BEHAVIOR_DEFAULT）

超过阈值直接拒绝，**默认效果**：

```text
请求 → 检查 QPS → 超过阈值 → BlockException
                  → 未超过 → PASS
```

```java
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);
```

## 4.2 WarmUp（预热/冷启动）

系统启动初期允许的 QPS 从 `count/coldFactor` 缓慢增长到 `count`，避免冷系统被高峰流量击垮：

```text
QPS
 ▲
 │                          ╭────────────
 │                    ╭─────╯
 │              ╭─────╯
 │        ╭─────╯              count = 100
 │  ╭─────╯
 │──╯───── count/3 = 33
 └─────────────────────────────────────► 时间
       预热期 (warmUpPeriodSec)
```

```java
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
rule.setWarmUpPeriodSec(30);  // 30 秒预热期
rule.setCount(100);           // 稳定后阈值 100 QPS
```

**原理**：基于**令牌桶算法**的变体 — Guava RateLimiter 的 SmoothWarmingUp。

## 4.3 匀速排队（CONTROL_BEHAVIOR_RATE_LIMITER）

以恒定速率通过请求，超过阈值的请求排队等待（而非直接拒绝）：

```text
请求到达时刻：
t0: ●         → 立即通过
t1:     ●     → 立即通过（在速率内）
t2:   ●●●●●  → 部分立即通过，部分排队
t3: ●●●●●●●● → 部分排队超时 → 拒绝

最大排队时间 = maxQueueingTimeMs
```

```java
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
rule.setMaxQueueingTimeMs(500);  // 最大排队 500ms
rule.setCount(10);               // 每秒 10 个请求
```

**原理**：**漏桶算法**，将突发流量整形为匀速流量。适合处理间隔性突发流量（如消息队列消费）。

## 4.4 效果对比

| 效果 | 原理 | 拒绝方式 | 适用场景 |
|------|------|---------|---------|
| 快速失败 | 超过阈值立即拒绝 | BlockException | 通用场景 |
| WarmUp | 令牌桶预热，逐步放量 | BlockException | 冷启动保护 |
| 匀速排队 | 漏桶算法，恒定速率 | 超过排队时间拒绝 | 突发流量整形 |

## 4.5 两处原理表述的纠偏（以 1.8.x 实现为准）

**WarmUp 不是令牌桶**。§4.2 末行"Guava RateLimiter 的 SmoothWarmingUp 变体"是概念类比不是实现描述，纠偏如下：Sentinel 的 `WarmUpFlowChecker` 做的是**放行阈值随时间爬升**——维护一个 `recoverTime`（由最近一次统计到的 pass QPS 反推"系统当前处于冷启动曲线的哪一点"），当前允许令牌数 = `count/3` 沿线性增速爬向 `count`，增速 `increasingRate = count / (coldFactor - 1) / warmUpPeriodSec`。两点与令牌桶的本质差异：① 没有"积攒的桶"，突发容忍度由**历史流量**决定——系统刚被 80 QPS 打过，recoverTime 就接近全热，此时突发不会被误杀；空转很久后才真正冷，曲线从 33 重爬；② `coldFactor` 硬编码为 3，**不可配置**，想要更陡的冷启动只能调小 count 或拉长 warmUpPeriodSec。Guava 的 SmoothWarmingUp 是"存令牌 + 按曲线等待"，行为形似而内核不同，面试与代码评审都按前者表述。

**匀速排队的"队列"不存在**。`RateLimiterFlowChecker` 的实现是维护一个"下一个允许通过的时钟点"：每个请求到来时算 `等待 = 预计放行时刻 - now`，等待 ≤ maxQueueingTimeMs 就**当前线程 sleep(parkNanos) 等满再放行**，超了就抛 `FlowLimitException`。由此三条硬约束：① 它不排队、只占线程——**Web 同步链路慎用**（等待计入 servlet 线程占用，还会撞上请求侧超时，线程数模式与它叠加时预算合并，§2.2）；② QPS 需 > 0.5（间隔 < 2s 才有意义，官方推荐上限排队 20s）；③ 放行时刻是全局单调时钟点，**多实例各排各的**，全局速率 = 单实例 rate × 实例数，MQ 削峰用它的前提是消费组内实例数稳定（否则"匀速"的总量在漂）。它真正的适用形态就是 §4.3 说的间隔性突发：秒杀订单落 MQ、消费端匀速灌 DB。

## 4.6 效果选择决策矩阵

| 特征 | 选择 |
|------|------|
| 同步 Web/HTTP 入口，要求快速失败 | 快速失败 |
| 刚发布/扩容、JIT 与缓存未热、瞬时高流量 | WarmUp（配 §4.5 的冷启动语义理解） |
| 下游吞吐恒定（DB 写入/MQ 消费），宁可等不可丢 | 匀速排队（且同步入口禁用，改由上游等待） |
| 流量既冷启动又怕丢 | 无银弹——排队+WarmUp 不能叠加（同 rule 单选），用快速失败 + 上游重试补偿 |

# 5. YAML 配置示例

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow

# Nacos 中存储的 JSON 规则
# [
#   {
#     "resource": "getUser",
#     "grade": 1,
#     "count": 100,
#     "strategy": 0,
#     "controlBehavior": 0,
#     "limitApp": "default"
#   }
# ]
```

## 5.1 JSON 字段与 Dashboard 表单的对照

数据源 JSON 用 Java 字段名（camelCase），Dashboard 表单用中文语义，两套心智必须对齐才不会"界面上设的没写进文件"：`grade`（QPS=1/线程=0，别与 degrade 的 grade 编号混——`5_circuit_breaker.md` §2.2）、`count`（阈值，double，支持小数是为排队场景的亚秒速率）、`strategy`（0 direct/1 relate/2 chain）、`refResource`、`controlBehavior`（0 快失/1 WarmUp/2 排队）、`warmUpPeriodSec`、`maxQueueingTimeMs`、`limitApp`（default/具体来源/other），以及集群流控子对象 `clusterConfig`（`clusterMode`、`thresholdType` 全局/单机、`fallbackToLocalWhenFail`——它的语义在 `14_dynamic_rules.md` §10 治理）。手写规则文件最常见的事故是把 Dashboard 显示的中文语义当枚举值填（`"grade": "QPS"`），watch 回调 JSON 反序列化直接失败且**旧规则继续服役**，改完必须验证：日志找 `load rules` 输出或 `FlowRuleManager.getRules()` 快照。

# 6. 最佳实践

1. **阈值设定**：先在测试环境压测确定基准 QPS，生产阈值设为基准的 80%
2. **接口分级**：核心接口（支付/下单）QPS 阈值高于非核心接口
3. **WarmUp**：JDK 的 C2 编译需要预热期，建议配合使用 WarmUp
4. **匀速排队**：适合下游处理能力固定的场景（如数据库写入）
5. **避免跨资源共用阈值**：不同接口的流量模型不同，分别设定阈值

## 6.1 阈值从集群到单机的换算

第 1 条"压测基准 80%"落地时有个单位陷阱：count 是**单机语义**。换算公式：`单机 count = 集群总容量目标 / 实例数 × 0.8`，配之前先看负载均衡是否均匀（RoundRobin 均匀、带 session 粘性不均匀）。两个衍生风险：**缩容不设防**——10 台缩到 5 台，规则不改则集群阈值翻倍，正确姿势是用全局阈值的集群流控（thresholdType=GLOBAL，`16_flow_control_source.md` 的 token server）或把规则与实例数联动进发布流水线；**压测基准本身**——单机压测出的 QPS 峰值受压测机、数据分布影响，80% 系数在"压测环境=生产规格"时才可信，容器化后 CPU limit 变化需要重新压。另外给"限住之后"留出路：阈值触发率（block 指标）超过日常 0.1% 就该告警而不是等人发现，规则值写进版本库与代码同审（`14_dynamic_rules.md`）。

# 7. 相关阅读

规则属性逐个的消费点在源码篇 `16_flow_control_source.md`（`DefaultController.passChecking` 的滑动窗口近似与排队实现）；统计口径与窗口粒度误差的地基在 `18_statistics_engine.md`；限住之后的世界（异常类型分桶、与熔断 SystemSlot 的抢跑顺序）见 `2_architecture.md` §3.3 与 `24_troubleshooting.md`；系列阅读地图在 `1_overview.md`。
