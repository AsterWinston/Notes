
# 1. 概述

系统保护（System Protection）从**整体系统负载**维度出发，在系统指标（Load、CPU、RT、线程数、入口 QPS）达到阈值时自动限流，防止系统被流量冲垮。

**与传统限流的区别**：

| 维度 | 资源级限流 | 系统级保护 |
|------|-----------|-----------|
| 粒度 | 单个资源 | 整个应用 |
| 触发条件 | 资源 QPS/线程数 | 系统 Load/CPU/RT |
| 自适应 | 需手动设定阈值 | 部分指标自适应 |
| 适用场景 | 精细化控制 | 整体系统防护 |

# 2. SystemRule 核心属性

```java
public class SystemRule extends AbstractRule {
    private double highestSystemLoad = -1;     // 系统 Load1
    private double highestCpuUsage = -1;       // CPU 使用率 [0, 1]
    private double avgRt = -1;                 // 平均 RT(ms)
    private double maxThread = -1;             // 最大并发线程数
    private double qps = -1;                   // 入口 QPS
}
```

**规则**：上述 5 个指标**独立生效**，任意一个达到阈值即触发系统保护。未配置（= -1）的指标不检查。

# 3. 五种系统保护规则

## 3.1 Load 保护（highestSystemLoad）

```java
SystemRule rule = new SystemRule();
rule.setHighestSystemLoad(4.0);  // 系统 Load1 超过 4.0 触发
```

**触发条件**：`系统 Load1 > highestSystemLoad` **且** `当前线程数 > 1`

**原理**：从操作系统 `/proc/loadavg` 读取 Load1。

**适用场景**：CPU 密集型应用，防止过度排队。

**注意**：Linux 有效，Windows/macOS 不可用。Load 阈值为逻辑 CPU 核心数。

## 3.2 CPU 使用率保护（highestCpuUsage）

```java
SystemRule rule = new SystemRule();
rule.setHighestCpuUsage(0.8);  // CPU > 80% 触发
```

**触发条件**：进程 CPU 使用率超过阈值。

**适用场景**：计算密集型服务。

## 3.3 平均 RT 保护（avgRt）

```java
SystemRule rule = new SystemRule();
rule.setAvgRt(100);  // 平均 RT > 100ms 触发
```

**触发条件**：所有入口流量的平均 RT 超过阈值 **且** 当前 QPS > 5。

**适用场景**：防止下游服务拖慢整体系统。

## 3.4 最大并发线程数（maxThread）

```java
SystemRule rule = new SystemRule();
rule.setMaxThread(200);  // 并发线程数 > 200 触发
```

**触发条件**：当前并发线程数超过阈值。

**适用场景**：长连接、慢调用场景。

## 3.5 入口 QPS（qps）

```java
SystemRule rule = new SystemRule();
rule.setQps(1000);  // 入口 QPS > 1000 触发
```

**触发条件**：所有入口资源的 QPS 总和超过阈值。

**适用场景**：最简单的全局限流。

# 4. BBR 自适应算法

Sentinel 1.8.0+ 引入了类 BBR 的自适应流控算法，无需手动设定阈值：

```java
// 开启 BBR 模式
SystemRule rule = new SystemRule();
rule.setHighestSystemLoad(2.0);  // 仍可作为兜底
// 设置 BBR 策略（通过 SentinelConfig）
```

BBR 算法根据**当前 QPS 与最大 QPS**的比值，动态调整通过量：

```
当前 QPS
  ├── < 最大 QPS * 容忍度 → 通过
  ├── > 最大 QPS * 容忍度 → 按比例放行
  └── 持续过高 → 逐步收紧 → 保护系统
```

# 5. SystemSlot 执行流程

```java
public class SystemSlot extends AbstractLinkedProcessorSlot<DefaultNode> {
    
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized, Object... args) {
        SystemRuleManager.checkSystem(resourceWrapper);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }
}

// SystemRuleManager.checkSystem() 内部逐个检查：
public static void checkSystem(ResourceWrapper resourceWrapper) {
    // 1. QPS 检查
    if (highestQps > 0) { checkQps(); }
    // 2. 线程数检查
    if (maxThread > 0) { checkMaxThread(); }
    // 3. RT 检查
    if (avgRt > 0) { checkAvgRt(); }
    // 4. Load 检查
    if (highestSystemLoad > 0) { checkSystemLoad(); }
    // 5. CPU 检查
    if (highestCpuUsage > 0) { checkCpuUsage(); }
}
```

# 6. 配置示例

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        system:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-system-rules
            group-id: SENTINEL_GROUP
            rule-type: system
```

Nacos JSON：

```json
[
  {
    "highestSystemLoad": 4.0,
    "highestCpuUsage": 0.8,
    "avgRt": 100,
    "maxThread": 200,
    "qps": 1000
  }
]
```

# 7. 最佳实践

1. **互补使用**：系统保护 + 资源级流控是一对防御组合
2. **Load 优先**：在 Linux 环境，Load 是最直接的过载信号
3. **阈值设定**：
   - `highestSystemLoad`：CPU 核心数（如 8 核 → 8.0）
   - `highestCpuUsage`：0.7 ~ 0.85
   - `avgRt`：P50 响应时间 × 2
4. **与流控共存**：系统保护是最后一层兜底，资源级流控在前
5. **K8s 环境注意**：容器内 Load 和 CPU 读取需确认正确性
