
# 1. 概述

流量控制（Flow Control）是 Sentinel 最核心的能力，通过**QPS**或**并发线程数**两个维度限制资源访问。

```
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

# 3. 流控模式（strategy）

## 3.1 直接模式（STRATEGY_DIRECT）

对当前资源直接限流，**默认模式**：

```
请求 → getUser → FlowSlot 检查 → 通过/拒绝
```

```java
rule.setStrategy(RuleConstant.STRATEGY_DIRECT);
```

## 3.2 关联模式（STRATEGY_RELATE）

当关联资源达到阈值时，限流当前资源：

```
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

```
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

# 4. 流控效果（controlBehavior）

## 4.1 快速失败（CONTROL_BEHAVIOR_DEFAULT）

超过阈值直接拒绝，**默认效果**：

```
请求 → 检查 QPS → 超过阈值 → BlockException
                  → 未超过 → PASS
```

```java
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);
```

## 4.2 WarmUp（预热/冷启动）

系统启动初期允许的 QPS 从 `count/coldFactor` 缓慢增长到 `count`，避免冷系统被高峰流量击垮：

```
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

```
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

# 6. 最佳实践

1. **阈值设定**：先在测试环境压测确定基准 QPS，生产阈值设为基准的 80%
2. **接口分级**：核心接口（支付/下单）QPS 阈值高于非核心接口
3. **WarmUp**：JDK 的 C2 编译需要预热期，建议配合使用 WarmUp
4. **匀速排队**：适合下游处理能力固定的场景（如数据库写入）
5. **避免跨资源共用阈值**：不同接口的流量模型不同，分别设定阈值
