
# 1. 概述

熔断降级（Circuit Breaking / Degradation）用于在依赖服务不稳定时**快速失败**，避免调用方线程堆积，是防止**雪崩效应**的关键机制。

```
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

```
statIntervalMs = 1000ms 内：
  请求总数 = 10
  慢调用数 (RT > 200ms) = 6
  慢调用比例 = 60% > 50% (slowRatioThreshold)
  → 触发熔断，进入 OPEN 状态
```

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
            if (retryTimeoutArrived() && fromOpenToHalfOpen(context)) {
                return true;  // 允许一个探测请求
            }
            return false;  // 熔断中，拒绝
        }
        // HALF_OPEN: 已经放行了一个探测请求
        return false;
    }
}
```

# 7. 熔断恢复流程

```
时间线：
t0: CLOSED（正常）
t1: 慢调用比例超过 50% → OPEN（开始熔断）
t1~t1+10s: 请求直接拒绝，走 fallback
t1+10s: 熔断时间到 → 尝试进入 HALF_OPEN
t1+10s: 放行一个探测请求
  → 成功 → CLOSED（恢复）
  → 失败 → OPEN（重新熔断）
```

# 8. 熔断 vs 降级 vs 流控

| 维度 | 熔断 | 降级 | 流控 |
|------|------|------|------|
| **触发条件** | 下游异常/慢调用 | 下游异常/超时 | 超过 QPS/线程阈值 |
| **行为** | 快速失败 | 返回兜底值 | 拒绝/排队 |
| **作用域** | 调用方侧 | 调用方侧 | 被调用方侧 |
| **恢复机制** | 半开状态探测 | 超时后重试 | 下一秒自动恢复 |
| **目的** | 防止线程堆积 | 提升用户体验 | 保护自身过载 |

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

# 10. 最佳实践

1. **慢调用优先**：生产环境优先使用慢调用比例熔断，异常比例作为补充
2. **合理阈值**：`count` 设为 P99 响应时间的 1.5~2 倍
3. **级联熔断**：上层服务熔断时间应**短于**下层服务
4. **区分核心/非核心**：核心接口熔断时长短（5s），非核心长（30s）
5. **fallback 设计**：始终提供有意义的 fallback 响应（缓存/兜底值/静态页面）
