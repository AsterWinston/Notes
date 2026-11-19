# 1. 概述

系统保护（System Protection）从**整体系统负载**维度出发，在系统指标（Load、CPU、RT、线程数、入口 QPS）达到阈值时自动限流，防止系统被流量冲垮。

**与传统限流的区别**：

| 维度 | 资源级限流 | 系统级保护 |
|------|-----------|-----------|
| 粒度 | 单个资源 | 整个应用 |
| 触发条件 | 资源 QPS/线程数 | 系统 Load/CPU/RT |
| 自适应 | 需手动设定阈值 | 部分指标自适应 |
| 适用场景 | 精细化控制 | 整体系统防护 |

## 1.1 系统保护的三个隐含语义

表之外必须先钉死三条边界，否则"系统规则没生效"会查错方向：**① 只作用于入口型流量**——`checkSystem` 内部对 `EntryType.OUT` 直接返回，且 QPS/RT/线程数取的是**入口资源（EntranceNode 维度，`2_architecture.md` §2.4 的 machine-root 聚合）**的指标；对内部方法级资源配了系统规则也不会拦"它自己"，只拦"经由入口进来的那一请求"。**② 拦的是"下一个请求"**——Load/CPU 是全局瞬时读数，判定发生在 entry 时刻，已经在跑的业务不会被中止（与线程数模式的区别：它数"正在跑的几个"，Load 看"机器喘不喘气"）。**③ 它不是容量规划工具**——五个阈值全部是"过载保护线"，触发即代表系统已经或即将劣化，资源级流控（§7.4 的定位）才是日常护栏，系统规则的设计目标是"资源级没配到、配不全的病"兜底。

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

## 2.1 "只有一条规则"的存储结构

SystemRule 与其他规则的一个显著差异：`SystemRuleManager` 虽然收 List，但**只取第一条**作为全局生效配置（`List<SystemRule> rules` → `convertAndInit` 里 `SystemRule rule = rules.get(0)`），其余条目直接忽略。所以：配多条系统规则是**无效操作**（Dashboard 也只让你维护一份），所有想用的指标写进同一条的各字段；"我加了第二条 load 规则怎么没排队生效"这类问题在 §24 排障表里属于配置姿势错误。另注意 -1 的表示是"不检查"，但阈值**填 0 同样等于不检查**（代码判断 `> 0`），想表达"一律拦截"不能用 0，得用极小正数并接受误配观感。

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

## 3.6 五项指标的细节纠偏与适用账

| 指标 | 数据源 | 原文核对 | 适用与坑 |
|------|--------|---------|---------|
| Load | JVM `getSystemLoadAverage()`（Linux 下内核值即来自 /proc/loadavg 一族） | "macOS 不可用"**要纠偏**：Unix 系（含 macOS）该 API 均可用，真正不可用的是 Windows（恒为 -1，判 `> 0` 自动跳过）；容器里的可用性见 §7.1 | 唯一含"线程数 > 1"前置的指标，防止低并发误拦 |
| CPU | 进程 CPU 时间片采样（后台采样线程 1s 一次，读数滞后最多一个采样周期） | "进程使用率"准确——是**本进程**占整机/配额的比例，不是机器 CPU | 对 IO 等待型服务不敏感（CPU 空闲但线程全挂），别指望它兜慢 SQL |
| avgRt | 入口流量滑动窗口平均 | "QPS > 5"前置正确（低流量下均值噪声大） | **平均值是被长尾稀释的**：一个 10s 超时会把均值瞬间拉爆但也可能均值仍达标——avgRt 适合当"系统性劣化"粗闸，精控用资源级 + 慢调用熔断（`5_circuit_breaker.md` §3） |
| maxThread | 入口 DefaultNode 实时线程计数 | 与资源级线程数模式同源（`4_flow_control.md` §1.1），exit 泄漏同样中招 | 语义接近"给自己线程池上闸"，值 = 容器线程池×0.7 比资源级更统 |
| qps | 入口 QPS 总和 | 正确 | 与"每个资源各配 QPS"相比只防总量不防结构，单接口打爆时它可能还没过线 |

## 3.7 触发后的异常形态

五项拦截抛的都是同一个 `SystemBlockException`（`2_architecture.md` §3.3 的链首拦截者），**区分靠 message 里的 ruleType 字段**（load/cpu/rt/thread/qps）——告警按异常类分桶时 SystemBlockException 是一个桶，看板下钻要解析 message 或看 `sentinel-record.log` 的 BlockException 记录，否则"系统保护在拦什么"永远是一笔糊涂账。对外的 HTTP 表现统一走 Web 适配器的 BlockExceptionHandler——**默认实现对所有 Block 家族一律 429**（含授权类异常），想在状态码上区分"授权 403/降级 503"属于应用自定义（`10_spring_cloud_integration.md` §7.1 的双通道说明）。

# 4. BBR 自适应算法

Sentinel 1.8.0+ 引入了类 BBR 的自适应流控算法，无需手动设定阈值：

```java
// 开启 BBR 模式
SystemRule rule = new SystemRule();
rule.setHighestSystemLoad(2.0);  // 仍可作为兜底
// 设置 BBR 策略（通过 SentinelConfig）
```

BBR 算法根据**当前 QPS 与最大 QPS**的比值，动态调整通过量：

```text
当前 QPS
  ├── < 最大 QPS * 容忍度 → 通过
  ├── > 最大 QPS * 容忍度 → 按比例放行
  └── 持续过高 → 逐步收紧 → 保护系统
```

## 4.1 机制还原与三个使用前提

实现（`BbrFlowCheckController`，还原注记）：维护**历史最大 QPS 的滑动窗口记录**（按秒统计进 `maxQpsStatistics`，默认保留 60s 峰值），每次入口检查算负载比 `ρ = 当前入口QPS × 并发线程需求 / 历史最大QPS`（工程化近似：`ρ = curQps / maxQps`），`ρ ≤ k0`（**0.6，硬编码不可配**）直接放行，超过则按 `1 - ρ` 系数的概率丢弃——本质是"**用系统自己的吞吐历史当动态阈值**"，QPS 与 RT 耦合恶化时收紧。三个前提决定它值不值得开：**① 峰值参照需要时间建立**——新启动实例/低谷期刚过的实例 maxQps 失真（偏低→误杀，长期空闲后偏高→形同虚设），冷启动阶段 BBR 是"接近全放"的；**② 历史峰值本身可能是"病态吞吐"**——系统在劣化前刚好冲过一波高峰，那个峰值会保护劣化流量；**③ 概率性通过意味着相同请求可能时过时不过**，压测曲线会毛糙。**默认策略仍是五项阈值**，BBR 需要显式开启（属性开关，1.8.x 不同小版本属性名有出入，以当前版本 release note 为准）；混合建议：高并发入口型应用可开 BBR 做第一层，`highestSystemLoad` 留作硬顶兜底，RT/线程类精细保护仍靠资源级规则——BBR 不是五项指标的替代品，它只替代 qps 维度且换成了自适应阈值。

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

## 5.1 骨架之外的两个真实分支

真实 `checkSystem` 在五项之前还有两段示例没画：**① 入口类型过滤**（OUT 直接 return，§1.1）；**② 指标读数非法即跳过**——`load < 0`（Windows/采集失败）、CPU 采样未就绪时对应检查静默不执行，这是系统保护的 **fail-open 本性：采不到数=不保护，而不是全拦截**。由此推出排障铁律：系统规则"没拦住"先查读数可得性（日志里 metric 值、OS 类型、容器可见性），而不是先怀疑阈值设高了。检查顺序（QPS→线程→RT→Load→CPU，与真实代码一致）决定多项同时超标时报出的第一项，§3.7 的 message 归因以它为准；另注意 `checkSystem` 拿的 DefaultNode 是 context 入口节点（`NodeSelectorSlot` 挂的那棵树，§1.1 入口语义的出处）。

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

## 6.1 这条示例 JSON 的读法

注意它是"五闸全开"的激进形态：任一命中即拦，实际部署通常**只开 1~2 个最有把握的维度**（§7.1 的容器环境建议：qps + maxThread 是最不依赖 OS 读数的两项），其余留 -1。另与资源级规则的量纲核对：`qps=1000` 是单机入口总量，若实例 10 台、集群目标 1 万，这里应该恰好写单机值 1000——"全局阈值"是集群流控的概念（`4_flow_control.md` §6.1），系统规则**永远是单机语义**，扩容缩容时必须联动改这份 JSON（发布流水线里给它留一个模板变量位）。

# 7. 最佳实践

1. **互补使用**：系统保护 + 资源级流控是一对防御组合
2. **Load 优先**：在 Linux 环境，Load 是最直接的过载信号
3. **阈值设定**：
   - `highestSystemLoad`：CPU 核心数（如 8 核 → 8.0）
   - `highestCpuUsage`：0.7 ~ 0.85
   - `avgRt`：P50 响应时间 × 2
4. **与流控共存**：系统保护是最后一层兜底，资源级流控在前
5. **K8s 环境注意**：容器内 Load 和 CPU 读取需确认正确性

## 7.1 K8s 那条要展开成可执行结论

容器里 `getSystemLoadAverage()` 读的是 **cgroup 之外、宿主机内核的 load**（无 lxcfs 时所有容器共享同一份数字）——"Load 优先"（第 2 条）在物理机/VM 语境成立，在 K8s 上恰恰是最不可信的一项：邻居容器打爆宿主机，你的应用被无辜限流；你自己的应用过载，load 也可能被宿主机上百个进程的噪声稀释。**K8s 上的推荐组合**：`qps`（纯应用自测指标）+ `maxThread`（同上）为主，`highestCpuUsage` 经实测确认 JDK 版本口径（JDK 11+ 的进程 CPU 计算与 `availableProcessors` 配额感知有关，不同版本语义漂移，必须用自己的压测验证读数）后再启用，`highestSystemLoad` 直接不配。宿主机级过载信号交给 K8s 自己的机制（HPA 扩缩、requests/limits），Sentinel 专注进程内可精确度量的维度——各管各的层，别用应用中间件重造调度器的活。第 3 条的 `avgRt = P50×2` 补一句核对：它和熔断的 RT 阈值（`5_circuit_breaker.md` §2.2，1.5~2×P99）**量纲对象不同**（全局平均 vs 单资源慢判定），两套数字别互相抄。

# 8. 相关阅读

系统规则在链上的位置与 fail-open 全局面在 `15_slot_chain_source.md`；指标读数的采集实现（`OperatingSystemMetricLoader` 一族、采样线程）在 `18_statistics_engine.md`；触发后对外行为（Web 适配器 BlockException 响应）在 `10_spring_cloud_integration.md`；与资源级规则的分工事故案例在 `24_troubleshooting.md`。
