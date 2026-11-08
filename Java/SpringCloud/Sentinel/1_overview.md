# 1. 概述

Sentinel 是阿里巴巴开源的一款**面向分布式服务架构的流量防护组件**，以流量为切入点，从**流量控制、熔断降级、系统自适应保护**等多个维度保障微服务的稳定性。与只做"熔断器"的库不同，Sentinel 把限流、熔断、降级、隔离、系统保护统一到一套 **Rule 模型 + SlotChain 执行链 + Dashboard 管控面**之下，这是它与 Hystrix/Resilience4j 最本质的定位差异。2018 年开源，2020 年发布 1.8.0 GA，是 Spring Cloud Alibaba 生态的**默认选择**。

核心能力矩阵：

| 能力 | 说明 | 典型场景 |
|------|------|---------|
| **流量控制** | QPS/并发线程数限流，支持 WarmUp、匀速排队 | 秒杀防刷、API 配额 |
| **熔断降级** | 慢调用比例/异常比例/异常数触发熔断 | 下游服务故障隔离 |
| **系统保护** | Load/CPU/RT/线程数/QPS 自适应 | 防止系统雪崩 |
| **热点保护** | 针对热点参数（如商品ID）精细化限流 | 热门商品防刷 |
| **授权控制** | 黑白名单，来源应用控制 | 服务调用方鉴权 |
| **集群限流** | 全局维度统一阈值（1.6.0+） | 分布式配额 |

# 2. 为什么要用 Sentinel

## 2.1 微服务架构的稳定性挑战

某个服务的延迟增加或不可用会沿调用链向上传导：下游 C 变慢，B 的线程被长时间占用，B 线程耗尽后对上游 A 也表现为不可用，导致**雪崩效应**：

```text
服务 A → 服务 B → 服务 C（慢/故障）
   ↑         ↑
   └─ 线程堆积 ── 线程耗尽 ── 整体不可用
```

在足够大的系统里故障是常态，防护的目标不是"消灭故障"而是**把故障影响限制在最小单元内**：

- **流量整形**：将突发流量平滑为均匀请求，避免后端过载
- **熔断降级**：快速失败不可用的下游，释放线程资源
- **系统保护**：监控整体负载，在系统崩溃前主动限流

防线生效的前提是**指标实时性**：分钟级监控远慢于故障传播，Sentinel 内置秒级滑动窗口引擎直接消费本地内存指标。

## 2.2 Sentinel vs Hystrix vs Resilience4j

| 维度 | Sentinel | Hystrix | Resilience4j |
|------|----------|---------|-------------|
| **维护状态** | ✓ 活跃 | ✗ 停止维护 | ✓ 活跃 |
| **限流** | QPS/线程/WarmUp/匀速排队 | 线程池/信号量隔离 | 信号量隔离 |
| **熔断** | 慢调用/异常比例/异常数 | 异常比例 | 异常比例/慢调用 |
| **系统保护** | ✓ 自适应 | ✗ | ✗ |
| **热点参数** | ✓ | ✗ | ✗ |
| **控制台** | ✓ 可视化管理 | ✓ Dashboard | ✗ 无官方 |
| **规则推送** | 拉/推模式，多数据源 | 配置管理 | 代码定义 |

综合来看，**Sentinel 在规则丰富度、可视化、性能上均优于 Hystrix，是 Spring Cloud Alibaba 体系的推荐选择**。

> **行业实践**：Netflix 已宣布 Hystrix 进入维护模式，Spring Cloud 将其标记为不可用；社区以 Resilience4j 为继任者，新项目不应再选型 Hystrix。

### 2.2.1 隔离策略：线程池 vs 信号量 vs 并发数限流

隔离是三者分歧最大的设计点。Hystrix 默认**线程池隔离**：每个依赖独立线程池、调用异步转发，能物理隔离故障并中断超时线程，代价是线程池随依赖膨胀、有上下文切换开销。Resilience4j 只保留**信号量隔离**（Bulkhead），用许可计数限制并发，无切换开销但不能中断已发出的请求。

Sentinel 走第三条路：**不做线程转发，用并发线程数限流实现隔离**——`FlowRule` 的 `grade=0`（线程数模式）统计资源当前并发数、超限直接拒绝，语义上等价于信号量许可，但统一收敛进流控规则体系：

| 维度 | 线程池隔离（Hystrix） | 信号量隔离（Resilience4j） | 并发数限流（Sentinel） |
|------|----------------------|---------------------------|----------------------|
| **超时中断能力** | 可中断池中线程 | 不可中断 | 不可中断（依赖调用方超时设置） |
| **阈值动态调整** | 需改配置重建池 | 部分支持 | ✓ 规则推送即时生效 |

> **设计考量**：Sentinel 不用线程池隔离，是因为微服务调用多为 IO 密集型，转发只增加一次上下文切换、并不减少等待；"并发数"以零切换开销表达同样的隔离意图，且与 QPS 限流同属 FlowRule，可叠加规则、动态推送。

### 2.2.2 降级能力对比

Hystrix 用 `getFallback()`、Resilience4j 用函数式 API 组合降级；Sentinel 把**两类降级入口严格分开**：`blockHandler` 只处理 `BlockException`（流控/熔断拒绝），`fallback` 处理业务异常（见 `25_interview_qa.md`）。

### 2.2.3 实时指标统计：滑动窗口

三者熔断判断都基于滑动窗口：Hystrix 10s 滚动窗口、Resilience4j 计数/时间双模式、Sentinel **LeapArray** 秒级 1s（500ms × 2），且**统计引擎同时供规则判断与监控消费**，口径天然一致（`18_statistics_engine.md`）。

# 3. 核心设计理念

## 3.1 Resource（资源）

**Resource 是 Sentinel 中一切需要保护的代码点**，构成防护的最小单元，以唯一的 `resourceName` 标识，通过 API 或注解声明：

```java
// 方式一：原生 API 硬编码
try (Entry entry = SphU.entry("getUser")) {
    return userService.getUser(id);      // 被保护的业务逻辑
} catch (BlockException ex) {
    return "服务繁忙，请稍后再试";        // 被流控/降级
}

// 方式二：注解声明（Spring Cloud 环境推荐）
@SentinelResource(value = "getUser", blockHandler = "handleBlock")
public User getUser(Long id) { ... }
```

## 3.2 SlotChain（责任链）

围绕每个资源，Sentinel 构建一条 **ProcessorSlot 责任链**，每个 Slot 负责一类检查，链式执行、单一职责：

```text
SphU.entry("getUser")
        │
        ▼
ProcessorSlotChain（按序执行 entry 检查）
        │
        ├── NodeSelectorSlot     # 构建 DefaultNode，组织调用树
        ├── ClusterBuilderSlot   # 构建 ClusterNode（全局统计节点）
        ├── LogSlot              # 记录拦截日志
        ├── StatisticSlot        # 先执行后续 Slot，再按结果统计 pass/block/RT
        ├── SystemSlot           # 系统保护检查
        ├── AuthoritySlot        # 授权检查（黑/白名单）
        ├── FlowSlot             # 流量控制检查
        ├── DegradeSlot          # 熔断降级检查
        └── ParamFlowSlot        # 热点参数流控检查
```

责任链 + SPI 使扩展非常干净：新增一类防护只需实现一个 Slot 并注册进链（见 `19_spi_extension.md`）。链中 **StatisticSlot 的位置最特殊**——先放行后续检查、再按结果统计（源码见 `15_slot_chain_source.md`）。

## 3.3 滑动窗口统计

基于 **LeapArray** 的统计引擎以秒级精度采集 RT/QPS/异常数，窗口过期重置复用，单次检查微秒级，规则判断与控制台监控共用这一套指标。

## 3.4 规则即配置

所有防护策略通过 **Rule 对象**表达（FlowRule/DegradeRule/SystemRule/AuthorityRule/ParamFlowRule），与代码解耦、支持动态推送，修改阈值**无需重启应用**；规则与管理类、消费 Slot 的映射见 `2_architecture.md`。

## 3.5 流量效果多样化

传统限流只有"拒绝"一种反馈，Sentinel 把**超限后的流量形态**也做成配置项：除快速失败外支持 WarmUp 预热与 RateLimiter 匀速排队，见 4.1。

# 4. 三大能力一览

**流控保自己、熔断保线程、系统保底线**，三者可在同一资源上叠加生效，任一 Slot 抛出 `BlockException` 即拒绝本次请求。

## 4.1 流量控制

流控是最核心的能力，通过 **QPS** 或 **并发线程数**控制资源访问，除阈值外还有**调用关系**（直接/关联/链路）与**流控效果**（超限后怎么处置）两个配置轴，细节见 `4_flow_control.md`、`16_flow_control_source.md`：

| 控制行为 | 常量 | 效果 | 典型场景 |
|---------|------|------|---------|
| 快速失败 | `CONTROL_BEHAVIOR_DEFAULT` | 立即抛 `BlockException` | API 配额、防刷 |
| Warm Up | `CONTROL_BEHAVIOR_WARM_UP` | 阈值从 `count/coldFactor` 缓慢爬升 | 冷缓存、DB 连接池冷启动 |
| 匀速排队 | `CONTROL_BEHAVIOR_RATE_LIMITER` | 按固定间隔排队，超过 `maxQueueingTimeMs` 拒绝 | 秒杀削峰、消费平滑 |

## 4.2 熔断降级

熔断器在下游服务不稳定时**快速失败**，避免调用方线程被慢调用拖死。Sentinel 提供**慢调用比例**（1.8.0 起）、**异常比例**、**异常数**三种策略，状态机为 CLOSED → OPEN → HALF_OPEN 三态，半开只放一个探测请求验证下游恢复。慢调用比例对"下游没挂但已变慢"这种劣化前兆比 Hystrix 的纯异常比例反应更快，详见 `5_circuit_breaker.md`。

## 4.3 系统保护

系统保护是 Sentinel **独有**的能力：`SystemRule` 支持 Load、CPU 使用率、平均 RT、并发线程数、总 QPS 五个入口指标，任一达到阈值即限制所有流量，在系统崩溃前主动收缩而不是等资源耗尽。资源限流保护"某个接口"，系统保护保护"这个进程还能不能活"，详见 `6_system_protection.md`。

# 5. 控制台与规则推送

## 5.1 一句话架构

```text
                 ┌──────────────────────┐
                 │   Sentinel Dashboard  │  ← 规则管理 & 监控可视化
                 └──────────┬───────────┘
                            │ 规则推送 / 拉取
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
 ┌──────────┐       ┌──────────────┐      ┌──────────┐
 │ 服务 A    │       │   服务 B      │      │ 服务 C    │
 │ Sentinel │       │   Sentinel   │      │ Sentinel │
 │ Client   │       │   Client     │      │ Client   │
 └──────────┘       └──────────────┘      └──────────┘
```

每个服务内嵌 Sentinel Client，通过 SlotChain 在**本地**执行规则判断——防护路径上没有任何远程依赖，Dashboard 挂了也不影响线上流控。Dashboard 只做集中规则管理与监控，是**管控面**而非数据面。

## 5.2 Dashboard 工作原理

Client 与 Dashboard 之间是**拉模式通信**：Dashboard 定时向 Client 的 command 端口（默认 8719）拉取指标与规则快照，Client 主动发心跳注册。由此带来两个限制：**Dashboard 无状态**，秒级监控数据只存在各 Client 内存中；Dashboard 上改规则走原始模式、直写 Client 内存，**应用重启后规则丢失**。

> **注意**：Dashboard 手改规则仅供调试。生产必须配置持久化数据源、规则变更走配置中心发布流程，否则会出现"控制台显示与实际生效不一致"的排障陷阱。

## 5.3 规则持久化与 Nacos 推送

生产标准姿势是 **Nacos + Push 模式**：规则以 JSON 存放在 Nacos，Client 注册监听器，配置变更由 Nacos 主动推送生效，典型配置：

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080       # 监控地址
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow
```

推送模式三档：**原始模式**（写 Client 内存，重启丢失）、**Pull 模式**（定时拉取，有秒级延迟）、**Push 模式**（配置中心即时推送，生产推荐）；完整配置见 `14_dynamic_rules.md`。

# 6. 开源版与商业版

Sentinel 开源版（Apache 2.0）覆盖全部核心能力；阿里云 **AHAS（应用高可用服务）** 是与之同源的托管商业产品，规则模型兼容：

| 维度 | 开源 Sentinel | 阿里云 AHAS |
|------|--------------|-------------|
| **部署形态** | Client 内嵌 + 自建 Dashboard | 全托管，SDK/Java Agent 无侵入接入 |
| **规则存储** | 需自行对接 Nacos/Apollo/ZK | 平台托管，控制台修改即持久化 |
| **监控告警** | 秒级数据存内存，需自建持久化 | 长期存储、告警、与 PTS 压测联动 |

需要自主可控、精细规则管控选开源；不想维护 Dashboard 与配置中心联动改造、希望开箱即得告警选商业版，注解与 API 层面迁移成本很低。

# 7. 发展历程

| 版本 | 时间 | 里程碑 |
|------|------|--------|
| 0.x | 2012 | 阿里内部诞生，前身为"哨兵系统" |
| 0.1.0 | 2018.7 | 正式开源 |
| 0.2.0 | 2018.9 | 支持异步调用链路 |
| 1.4.0 | 2019.3 | 完善控制台功能 |
| 1.6.0 | 2019.10 | 支持集群流控 |
| 1.8.0 | 2020.8 | 完善 Spring Cloud Gateway 适配 |
| 1.8.5 | 2023 | 增强 JDK 17+ 兼容、虚拟线程支持 |

# 8. 选型判断

**推荐 Sentinel 的场景**：

- 技术栈已是 Spring Cloud Alibaba（Nacos + Sentinel 一套打通，适配成本最低）
- 需要**动态规则 + 可视化管控**：阈值随大促频繁调整，非开发人员也要能看懂曲线
- 存在热点参数限流（爆款 SKU）、系统自适应保护、集群流控等 Hystrix/Resilience4j 覆盖不到的需求

**可考虑 Resilience4j 的场景**：纯 Spring Cloud 开源系技术栈，只需熔断 + 重试 + 超时，不想引入配置中心和 Dashboard。

**可能都不需要的场景**：单体或接口很少，网关层限流足够；限流已在统一接入层做完、服务间只需超时重试兜底。

> **注意**：Spring Cloud CircuitBreaker 的门面抽象只覆盖熔断语义，Sentinel 的流控、系统保护、热点参数都在其之外——选 Sentinel 实际是选择它的**整套管控面**，迁移成本应按"换产品"评估。

# 9. 小结

- Sentinel 定位为**流量防护组件**：流控保自己、熔断保线程、系统保护保底线，配合热点/授权形成完整能力矩阵
- 与 Hystrix 的根本分歧在**隔离策略**：Sentinel 用并发线程数限流替代线程池转发，零切换开销且统一进 FlowRule 体系、支持动态推送
- 四大核心概念：**Resource**（保护单元）、**Rule**（策略配置）、**SlotChain**（责任链执行引擎）、**LeapArray**（滑动窗口统计引擎），规则判断与监控共用一套秒级指标
- Dashboard 属管控面，Client 本地执行规则、防护路径无远程依赖；生产必须 **Nacos 持久化 + Push 模式**，Dashboard 手改规则重启即丢
- 选型：Spring Cloud Alibaba 体系默认 Sentinel；轻量纯熔断场景可考虑 Resilience4j；需要托管运维与告警联动可评估 AHAS
