
# 1. 基础题

## 1.1 什么是 Sentinel？为什么需要它？

Sentinel 是阿里开源的**分布式系统流量防护组件**，核心能力包括流量控制、熔断降级、系统自适应保护、热点参数限流和授权控制。

**为什么需要**：在微服务架构中，单个服务故障可能引发雪崩效应。Sentinel 从多维度保护系统稳定性——限制入口流量保护自身，熔断故障下游释放线程资源，系统保护防止整体崩溃。

## 1.2 流控、熔断、降级有什么区别？

| 维度 | 流控 | 熔断 | 降级 |
|------|------|------|------|
| 触发条件 | 超过 QPS/线程阈值 | 下游异常/慢调用 | 下游异常/超时 |
| 行为 | 拒绝/排队/预热 | 快速失败 | 返回兜底值 |
| 作用域 | 被调用方 | 调用方 | 调用方 |
| 恢复方式 | 下一秒自动恢复 | 半开探测恢复 | 超时后重试 |
| 何时使用 | 保护自身过载 | 防止线程堆积 | 提升用户体验 |

**Sentinel 中**：流控通过 `FlowSlot` 实现，熔断通过 `DegradeSlot` 实现，降级通过 `fallback` 机制实现。

## 1.3 Sentinel vs Hystrix vs Resilience4j？

| 维度 | Sentinel | Hystrix | Resilience4j |
|------|----------|---------|-------------|
| 限流模型 | QPS/线程数/WarmUp/匀速排队 | 线程池/信号量隔离 | 信号量隔离 |
| 熔断策略 | 慢调用/异常比例/异常数 | 异常比例 | 异常比例/慢调用 |
| 可视化 | ✓ Dashboard | ✓ Dashboard | ✗ 无官方 |
| 规则动态推送 | ✓ Nacos/Apollo/ZK | ✗ | ✗ |
| 系统保护 | ✓ | ✗ | ✗ |
| 热点参数 | ✓ | ✗ | ✗ |
| 维护状态 | ✓ 活跃 | ✗ 停止维护 | ✓ 活跃 |

# 2. 原理题

## 2.1 Sentinel 的 SlotChain 是什么？执行顺序是怎样的？

SlotChain 是 Sentinel 的核心执行引擎，基于**责任链模式**将规则检查串联：

```
1. NodeSelectorSlot    → 构建调用树 Node
2. ClusterBuilderSlot  → 构建全局 ClusterNode
3. LogSlot             → 记录拦截日志
4. StatisticSlot       → 统计（先执行后续 + 再统计指标）
5. SystemSlot          → 系统保护检查
6. AuthoritySlot       → 授权（黑白名单）检查
7. FlowSlot            → 流量控制检查
8. DegradeSlot         → 熔断降级检查
```

**关键设计**：`StatisticSlot` 先执行后续 Slot，再根据结果决定统计"通过"还是"阻塞"。

## 2.2 Sentinel 的滑动窗口是怎么实现的？

使用 `LeapArray` 数据结构：
- 环形数组 + 时间分片
- 默认 2 样本 / 1 秒（500ms 窗口 × 2）
- 每个窗口内用 `MetricBucket` 记录 pass/block/exception/rt 等指标
- 窗口过期时**重置复用**，避免频繁创建对象
- 使用 `LongAdder` 替代 `AtomicLong` 降低竞争

## 2.3 Sentinel 的熔断状态机是怎样的？

三态模型：
- **CLOSED**：正常统计指标，判断是否触发熔断
- **OPEN**：直接拒绝，经过 `timeWindow` 后尝试进入 HALF_OPEN
- **HALF_OPEN**：通过 CAS 保证只有一个探测请求，成功则恢复到 CLOSED，失败则回到 OPEN

## 2.4 blockHandler 和 fallback 的区别？

| 维度 | blockHandler | fallback |
|------|-------------|----------|
| 触发异常 | `BlockException` 及其子类 | 所有 `Throwable`（业务异常） |
| 设计目的 | 处理流量管控拒绝 | 处理业务异常兜底 |
| 参数签名 | 原参数 + `BlockException` | 原参数 + `Throwable`（或仅 `Throwable`） |
| 典型场景 | "请求过多，请稍后重试" | "服务异常，返回缓存数据" |

# 3. 实践题

## 3.1 如何确定限流阈值？

1. **压测**：单机压测获得最大 QPS
2. **安全系数**：阈值 = 最大 QPS × 80%
3. **核心分级**：核心接口保留全量，非核心接口减半
4. **动态调整**：上线后根据 Dashboard 监控微调

## 3.2 如何设计多级降级方案？

```
一级：流控拒绝 → 快速返回"排队中"
二级：熔断触发 → 返回缓存数据（Redis/本地缓存）
三级：系统保护 → 返回静态兜底（本地文件）
四级：完全降级 → 返回固定响应（如"系统维护中"）
```

## 3.3 生产环境如何部署 Sentinel？

```
┌──────────┐   规则拉取   ┌──────────┐
│  Nacos   │ ←────────── │ Sentinel │
│ (持久化) │ ──────────→ │ Client   │
└──────────┘   规则推送   └────┬─────┘
                               │ 心跳+监控
                         ┌─────▼──────┐
                         │ Dashboard  │ (仅监控)
                         └────────────┘
```

**关键配置**：Nacos 持久化规则 + Push 模式、Dashboard 仅用于监控。

# 4. 架构题

## 4.1 Sentinel 的 Context/Entry/Node 关系？

```
一次请求:
  Context（调用上下文）
    └── Entry（资源调用记录）
          ├── Node（每个 Context 中的资源节点）
          ├── DefaultNode（Context 维度统计）
          └── ClusterNode（全局维度统计，所有 Context 聚合）
```

- **Context**：线程级，一次请求中共享
- **Entry**：方法调用级，entry() 创建、exit() 销毁
- **ClusterNode**：全局唯一，跨所有 Context 汇聚

## 4.2 为什么 StatisticSlot 要放在链的前面？

`StatisticSlot` 的特殊设计决定了它的位置：
1. 它需要**先执行后续 Slot**（fireEntry）
2. 根据后续结果决定统计 pass 还是 block
3. 放在前面可以统计所有检查的结果，包括后续 Slot 可能抛出的异常

## 4.3 Sentinel 的数据源 Push 模式如何实现？

```
Nacos ConfigService
  │
  ├── Client 注册 Listener
  │     └── NacosDataSource.readSource()
  │           └── 解析 JSON → List<FlowRule>
  │
  ├── 管理员更新 Nacos 配置
  │     └── Nacos 推送变更通知
  │           └── PropertyListener.configUpdate()
  │                 └── FlowRuleManager.loadRules(rules)
  │
  └── 规则生效
```

# 5. 性能面试题

## 5.1 Sentinel 的性能开销大吗？

Sentinel 设计非常轻量：
- 单次 SlotChain 执行耗时约 **5-15 μs**（正常配置）
- 使用 `LongAdder` 替代 `AtomicLong`，高并发写入降低竞争
- 无锁读取：读取统计时无需加锁
- 对象复用：LeapArray 窗口重置复用

在 100 并发连接场景下，Sentinel 的性能损耗约 **3-5%**。

## 5.2 如何优化 Sentinel 在高 QPS 下的性能？

1. 控制规则数量（< 200 条）
2. 使用默认的 2 样本统计窗口
3. 避免在 blockHandler/fallback 中做远程调用
4. 关闭 Debug 日志
5. Context 复用而非重复创建

# 6. 场景题

## 6.1 秒杀场景如何用 Sentinel 保护？

```
1. 匀速排队（RateLimiter）: 平滑突发流量
   controlBehavior = CONTROL_BEHAVIOR_RATE_LIMITER, count = 500

2. 热点参数：对热门商品差异化限流
   ParamFlowItem: object="SKU001", count = 10000

3. 系统保护：兜底
   highestSystemLoad = 4.0

4. 多级缓存：Sentinel + Redis + 本地缓存
   fallback 返回缓存数据
```

## 6.2 如何实现灰度发布中的流量切换？

使用 `limitApp`（来源应用）区分：
- 灰度版本 origin = `my-service-canary`，阈值 = 10%
- 正式版本 origin = `default`，阈值 = 90%
- 逐步提升灰度版本阈值直到 100%
