
# 1. 整体架构

Sentinel 的架构分为**客户端核心库**和**Dashboard 控制台**两部分：

```
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

## 2.2 Context 与 Entry

```
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

# 3. ProcessorSlotChain — 责任链模式

## 3.1 架构原理

所有规则检查通过**责任链**串联执行，每个 Slot 负责单一职责：

```
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

# 4. 滑动窗口统计引擎

## 4.1 LeapArray 数据结构

Sentinel 使用 `LeapArray` 实现高效滑动窗口：

```
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

```
machine-root (EntranceNode)
  └── user-center (EntranceNode)
        └── getUser (DefaultNode)
              └── getUserDetail (DefaultNode)
                    └── queryDB (DefaultNode)
```

## 5.1 关联流控的调用树

调用树使得**关联流控**成为可能：当 `/order/create` 资源达到阈值时，可以限制 `/order/query` 的流量。

# 6. 规则推送模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **原始模式** | API 代码硬编码规则 | 开发测试 |
| **Pull 模式** | 客户端定时从数据源拉取规则 | 简单生产环境 |
| **Push 模式** | 数据源主动推送规则到客户端 | 生产环境（推荐） |
| **控制台推送** | Dashboard 通过 API 推送 | 配合控制台使用 |

推荐的生产架构：**Nacos + Sentinel Dashboard Push 模式**，规则持久化到 Nacos，Dashboard 修改后推送至客户端。
