本篇覆盖 Sentinel 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：定位与核心概念、内部机制、限流与熔断、使用与集成、集群与实践。治理概念框架见 15_spring_cloud.md 3.2.3，Feign/Gateway 侧集成见 18_feign.md 4.1、19_spring_cloud_gateway.md 3.1，规则持久化的配置通道依托见 17_nacos.md 3.2。

# 1. 定位与核心概念

## 1.1 是什么

### 1.1.1 Sentinel 是什么？功能矩阵怎么对到真实场景？

**定位**：阿里开源的**流量治理组件**——以"资源"为中心，在调用发生的进程内做**流量整形（限流）、故障止损（熔断降级）、系统兜底（自适应保护）**；SDK 轻量（无独立数据面进程），配 Dashboard 观测与规则管理。概念分工先钉牢（"拦流量、断故障、留底线"的完整论证在 15_spring_cloud.md 3.2.3）：限流防"进来太多"，熔断防"出去太坏"，降级管"被拦之后给什么"。

**功能→场景映射**（面试报场景不报清单）：

| 能力 | 解决的场景 | 深水区 |
|------|-----------|--------|
| 流量控制（QPS/并发线程数） | 接口被打爆、慢调用占满线程池（5_tomcat.md 4.1.2 的账） | 3.1.1 |
| 熔断降级（慢调用/异常比例/异常数） | 下游 B 变慢拖死上游 A 的级联雪崩 | 3.2 |
| 热点参数限流 | 大促爆款商品 ID 独吞流量、单用户刷接口 | 3.1.3 |
| 系统自适应保护 | 整机水位（CPU/load/入口 RT）到极限时全局限流 | 3.1.4 |
| 授权规则（来源黑白名单） | 内部接口只放特定调用方（按 origin） | 4.2.1 |
| 集群流控 | 多实例单机限流=N 倍漂移的精确总闸 | 5.1.1 |
| 生态适配（HTTP/Feign/Gateway/Dubbo/Sentinel annotation） | 存量接口零改造进治理体系 | 4.2.1 |

**一句话收口**："Sentinel 把'容量保护 + 故障隔离'做成**进程内规则引擎**——规则动态可调、维度多样、观测秒级，这是它对 Hystrix 时代最本质的超越（1.2.2）。"

### 1.1.2 什么是"资源"？定义资源有几种方式？粒度怎么把握？

**资源**=Sentinel 保护与统计的单位：一个名字 + 一次进出（entry/exit）之间的行为，指标（通过 QPS、异常、RT、线程数）全按资源聚合。三种定义方式：

```java
// ① 编程式（其他一切方式的底层）
try (Entry entry = SphU.entry("createOrder")) {        // 1.8+ Entry 可 try-with-resources
    return orderService.create(cmd);
} catch (BlockException ex) {          // 被治理规则拦截：限流/熔断/黑名单
    return degraded(cmd);
} finally { /* 老写法：entry.exit() 必须在 finally，且与 entry 成对 */ }

// ② 注解式：@SentinelResource（4.1.1 细讲，含 Spring AOP 边界坑）
// ③ 适配器自动注册：HTTP 端点（spring-mvc-adapter）、Feign 方法、Gateway 路由、
//    Dubbo Filter、MyBatis/Spring Cloud Stream 等——"依赖一开，接口即资源"（4.2.1）
```

**粒度判断（这题的真正考点）**：资源太粗（整个应用一个）——规则没法定向生效；**太细（每行代码一个 entry）——统计对象与责任链开销累积，Dashboard 被淹**。经验线：**以"可独立谈 SLA 的业务动作/接口"为粒度**——一个 Controller 端点、一次远程调用、一段核心流程；同一资源在**不同入口链路**下还要可区分（"链路限流"依赖 Context 记录调用路径，2.1.2/3.1.1），这是资源之上的第二维。

**配套概念 Context 与 origin**：`ContextUtil.enter("入口名", "调用来源")`——Context 表达"从哪个入口进来的"（链路限流按它圈定），origin 表达"谁在调"（授权规则按它黑白）；HTTP adapter 默认把来源解析成 `other`/按 `RequestOriginParser` 自定义——**origin 不解析，授权规则形同虚设**（5.2.2 常见失效场景之一）。

## 1.2 概念辨析与对比

### 1.2.1 限流、熔断、降级到底是什么关系？在 Sentinel 里各由哪条规则实现？

**用"谁在防谁"再拧紧一遍概念**（15_spring_cloud.md 3.2.3 的框架落到 Sentinel 实现）：

| 概念 | 防什么 | Sentinel 落点 | 触发后 |
|------|--------|--------------|--------|
| 限流 | **进来的流量**超过自身容量（预防） | FlowRule（QPS/线程数/热点 ParamFlowRule/系统 SystemRule） | 快速失败/Warm-up 排队/匀速排队（3.1） |
| 熔断 | **出去的对坏依赖的调用**放大故障（止损） | DegradeRule（慢调用比例/异常比例/异常数） | OPEN 期直接 BlockException（3.2） |
| 降级 | **体验与功能的取舍**（补偿策略） | **不是规则、是回调**：blockHandler/fallback/@SentinelResource 的 exit 处 | 兜底值/缓存/裁剪功能 |

**三个常见混淆一次说死**：① "降级=熔断的另一个名字"——错：熔断**必须配**降级动作才有用户语义，但降级也可由容量规划主动触发（大促关评价入口根本不进规则引擎，人工开关）；② "限流就是熔断的一种"——错：限流输入是**自身水位**（我在被冲击），熔断输入是**下游质量**（它在变坏）——同一次 BlockException，语义与处置完全不同（fallback 里按异常类型分流：FlowException 提示"稍后再试"、DegradeException 可切备用源，4.1.1）；③ "熔断器自己会重试恢复"——半开探测放的是**真实请求**试探（3.2.1），不是后台探活；"要不要给失败请求补一次真调用"是另一层（重试，归链路设计管：18_feign.md 2.2.3）。

**答题模板**：概念三句 + "在 Sentinel 里分别对应哪条规则/回调" + "一次调用被拦的完整旅程：Chain 上 System/Flow 先查、Degrade 后查（2.2.1），抛 BlockException → blockHandler → 返回兜底"——层次全出即高分。

### 1.2.2 Sentinel 和 Hystrix、Resilience4j 怎么选？

**Hystrix 遗产题的正确答法（先承认停更，再讲思想）**：Hystrix 2018 官宣维护模式（Netflix 转 4.0 项目），但"断路器模式/舱壁隔离/快速失败"仍是全部后继者的概念基座。

| 维度 | Hystrix | Sentinel | Resilience4j |
|------|---------|----------|--------------|
| 隔离模型 | **线程池舱壁**（默认，每次调用排队到专属池）/信号量 | **并发线程数计数**（FlowRule grade=线程数，进程内无额外池）+ 信号量语义 | 信号量（Bulkhead 模块，也可线程池） |
| 熔断判据 | 异常比例（10s 滚动窗口） | **慢调用比例/异常比例/异常数**三种（3.2.1） | 异常比例/异常数（滑动窗口可计数可时间） |
| 限流 | 无（靠线程池容量硬顶） | **专职能力**：QPS/预热/排队/热点/系统保护/集群 | RateLimiter 模块（注解式，功能克制） |
| 规则运维 | 代码/Archaius 配置 | **控制台动态推送 + DataSource 持久化**（4.3） | 配置属性/代码注册（无官方控制台） |
| 形态 | 重量级（每依赖一池） | 轻量计数 + 丰富维度 | **函数式组合子，最轻**，与 Micrometer 亲 |
| 现状（2026） | 存量 | SCA 栈默认、国内主流 | Spring Cloud Circuit Breaker **默认实现**（15_spring_cloud.md 2.1.1） |

**线程池 vs 信号量的账要会讲（高频追问）**：Hystrix 线程池舱壁——隔离彻底（下游再慢也只烧它自己的池，还能给慢依赖设超时中断），代价是**每依赖一组线程**：百级依赖=千级线程的上下文切换与内存，且响应式/WebFlux 场景根本没法"占线程等回调"；Sentinel/R4J 计数式——调用仍在当前线程执行，**并发数超限即刻拒绝**（不挪线程），轻量，但**拦不住"线程被下游阻塞着不退出"**——线程占用只能靠超时兜底（18_feign.md 3.1.1）+ 并发线程数规则间接约束。一句话："**线程池隔离防的是'不守规矩的下游拖死我'，信号量/计数隔离防的是'我自己发起太多'——前者硬隔离贵，后者软限流省**；现代栈默认后者 + 严格超时预算，个别高危依赖才值得补线程池舱壁。"

**选型收口**：Spring Cloud 体系内熔断的"门票"已被 Circuit Breaker 抽象统一（18_feign.md 4.1.2），**竞争发生在治理维度**：要控制台/热点/系统保护/集群限流这些"平台件"→ Sentinel；只要库级韧性、无治理平台 → Resilience4j 更轻；Hystrix 只剩存量认知。

# 2. 内部机制

## 2.1 统计与节点模型

### 2.1.1 Sentinel 的滑动时间窗口怎么实现？LeapArray 拆开讲。

**为什么先要滑动窗口**：限流判据是"最近 N 秒的 QPS"——固定窗口有**临界翻倍问题**（0.9s 打满 + 1.1s 打满=瞬时 2 倍阈值但两个窗口各自合规）；滑动窗口把时间切片、只统计"现在往回看 interval"的部分，误差随切片变小。

**LeapArray 结构**（默认 1s 统计窗）：

```text
环形数组 WindowWrap[MetricBucket>[]，长度 sampleCount（秒级统计默认 2）
每格 windowLengthInMs = interval/sampleCount（默认 500ms 一片）
定位：idx = (currentTime / windowLengthInMs) % array.length
      ├─ 本格起始时间 == 期望 → 直接用（本格内累加）
      ├─ 本格已过期（旧周期的遗留格）→ CAS 重置起始时间与计数后复用
      └─ 时间跳变过大（进程暂停等）→ 整数组重置
读取（判限时）：currentWindow() 把"当前时刻往回 1s"覆盖到的格子聚合成序列求和
```

**MetricBucket 里记什么**：八个维度的 LongAdder（PASS/BLOCK/SUCCESS/EXCEPTION/RT 累计值等）——**RT 维度存的是窗口内耗时总和**（配合成功数算平均 RT；3.2.2 口径题的根）。**LongAdder** 的选择本身是考点：高并发写场景下比 AtomicLong 分段竞争友好（1_java.md 并发容器的思想复用）。

**参数与精度**：秒级窗口默认 `1000ms / 2 片 × 500ms`——**粒度可配置**（`SampleCountProperty` 运行期可改，精度与内存的折中：片多→边界误差小→对象多）；分钟级统计（异常数规则）是另一独立 LeapArray 实例。**面试加分点**：① 窗口边界误差"最多差一片"——Sentinel 用**更短的片 + 排队/预热缓冲**消化临界毛刺，而不是追求绝对精确（"限流是概率艺术不是计量学"）；② 时钟回拨/进程 FGC 停顿（2_jvm.md 场景）会让窗口重置逻辑兜底（时间跳变整数组 reset）——"统计失真 5 秒"比"抛异常"更符合治理件自我定位。

### 2.1.2 DefaultNode、ClusterNode、OriginNode 都是什么？Dashboard 的调用链路图从哪来？

**一棵资源树 + 三个视图**：

```text
调用树（NodeSelectorSlot 维护）：
  entrance "unknown" → /api/orders（HTTP 入口）→ createOrder（资源）
    每个 (资源 × 入口路径) 组合挂一个 DefaultNode —— 记录"这条链路"的统计
ClusterNode（ClusterBuilderSlot 维护）：
  每个资源全局唯一 —— 跨所有链路聚合的"这个资源本身"的统计
OriginNode：
  资源 × 调用来源(origin) —— "谁在调我"的分来源统计（授权规则的数据源）
```

**三者怎么被规则消费（这题的机制增量）**：
- **普通流控/熔断看 ClusterNode**（资源全局水位）；
- **链路限流看 DefaultNode**——同一 `createOrder`，从 `/api/orders` 入口的量超限、从 `/admin` 入口的量没超——规则带 `restrictRef`（入口上下文名）时判定对象切换为该入口下的 DefaultNode（3.1.1 "链路"模式）；
- **授权规则看 OriginNode**（黑白名单匹配 origin，4.2.1）。

**Dashboard 实时曲线与"调用链路"标签页的数据来源**：控制台经 **command API（客户端 8719 起的端口）实时拉**这些节点的窗口序列（4.3.2）——所以链路图不是猜的，是 DefaultNode 树的直接投影。**易错**：把"资源树"说成"调用链追踪"——它只是**进程内**入口→资源的嵌套关系，跨进程的 trace 是 Micrometer Tracing/OTel 的事（15_spring_cloud.md 4.2.1），两套体系别在面试里握手。

## 2.2 处理链

### 2.2.1 Slot 责任链怎么工作？各 Slot 的顺序和职责？

**一次 entry 的流水线**（`ProcessorSlotChain`，CtSpu 资源元数据上挂链）：

```text
SphU.entry(resource, args...)
  → NodeSelectorSlot：按 (资源×上下文路径) 建/取 DefaultNode，挂进调用树
  → ClusterBuilderSlot：取/建资源的全局 ClusterNode 与 OriginNode，绑定到本链路
  → LogSlot：（被拦时）记录 block 日志
  → StatisticSlot：先累加 PASS 计数 → fireEntry 放行下游链 → 回调时按结果记 SUCCESS/EXCEPTION/RT
  → AuthoritySlot：授权规则（黑白名单 origin）
  → SystemSlot：系统自适应规则（整机水位，入口流量才检）
  → FlowSlot：流控规则（QPS/线程数/直接/关联/链路/效果）
  → ParamFlowSlot：热点参数规则（classpath 带参数限流模块时插在 Flow 之后）
  → DegradeSlot：熔断规则（查 CircuitStateController 三态）
  任一 Slot 不过 → 抛 BlockException 子类 → 链终止 → 业务侧走降级
```

**三个链设计的考点**：
1. **StatisticSlot 在规则 Slot 之前挂链、但统计"结果"发生在调用返回后**——`chain.fireEntry` 回调模式让它同时拿到"通过数（entry 时）"与"成功/RT/异常（exit 时）"——**限流用 PASS 口径（拦前计数）、熔断用 SUCCESS+RT 口径（调用结果）**，两个数据源的差异直接解释 3.2.2 的口径题；
2. **顺序即语义**：授权（不让谁调）→ 系统（整机兜底）→ 流控（自身容量）→ 热点（精细维度）→ 熔断（依赖健康）——**便宜的粗筛在前、依赖统计的在后**；熔断排最后合理吗？合理：熔断判的是"这个资源最近的表现"，前面几步都过了才轮到"要不要为它的历史劣化买单"；
3. **责任链是 SPI 可插拔的**（SlotChainBuilder）——自定义 Slot 插链（审计、染色）是扩展姿势；**链按资源惰性构建**，规则不存在时对应 Slot 直穿（开销≈一次判空）——"接入 Sentinel 但没配规则=近乎零成本"的依据。

**易错**：exit 不对称导致**线程数统计泄漏**（并发线程数规则下"卡死不还"的调用会永久占用配额，触发假限流——3.1.1 线程数模式的排障点，异步资源要用 `entry.exit()` 的异步变体/AsyncEntry，4.1.1 补充位）。

# 3. 限流与熔断

## 3.1 流量控制

### 3.1.1 流控规则的三要素是什么？各自怎么实现？

**FlowRule 一张表拆三要素**（resource/count 之外，规则性格由这三个字段决定）：

**① 阈值类型（grade）**：
- **QPS**：对 ClusterNode 的滑动窗口 PASS 数判限（2.1.1 的 LeapArray，"通过请求"口径在 entry 时就累加）；
- **并发线程数**：对 ClusterNode 的 `threadNum`（AtomicInteger，entry +1 / exit -1）判限——**慢调用堆积的直接保险丝**：下游 RT 恶化时先掐并发（比 QPS 更早反映"我快撑不住了"，因为线程占用=RT×QPS 的小定律，5_tomcat.md 4.1.2 利特尔法则的治理应用）；前提是 exit 必须配对归还（2.2.1 的泄漏坑）。

**② 流控策略（strategy）**：
- **直接**：检自己的水位（默认）；
- **关联（refResource）**：**别资源达到阈值，限我**——经典账：下单与库存扣减共享 DB 容量，`deductStock` 线程数打满时限流 `createOrder`（保护共同的下游，而非各自为政）；
- **链路（restrictRef）**：同一资源不同入口分开算账——检的是该入口 Context 下的 **DefaultNode**（2.1.2），"admin 后台批量拉取不许挤兑 C 端查询"的实现基础。

**③ 控制行为（controlBehavior）**：
- **快速失败**（默认）：超线即抛 FlowException；
- **Warm Up**：见 3.1.2；
- **匀速排队**：见 3.1.2。

**追问预备**："规则怎么匹配到这次 entry"——FlowSlot 里按资源查 `Map<resource, List<FlowRule>>`（规则变更由 DataSource 回调重建，4.3.1），逐条 check 任一不过即拦；**一个资源可挂多条规则**（QPS+线程数+链路各一条），**最严者生效**——"配了大阈值没生效"先查有没有第二条小阈值规则。

### 3.1.2 Warm Up 和匀速排队背后的算法是什么？分别什么时候用？

**两个效果都由 RateLimiterController 实现**（存"上一通过请求的预期放行时刻"，非经典令牌桶容器，但语义等价可讲）：

**匀速排队＝漏桶**：请求到来按其消耗令牌数推算"应该被放行的时刻"，早于当前时刻立即过、晚于则 `Thread.sleep` 排到那个点——**出口速率恒定=count/s**；`maxQueueingTimeMs` 是排队上限：预计等待超过它就拒绝（否则线程无限堆睡）。**场景**：MQ 消费平滑（13_rocketmq.md 3.3.2 中转搬迁后接的正是这招）、写入削峰、定时任务打散——**"能等"的异步/后台流量专属**；同步 HTTP 慎用（排队线程=占用 Tomcat 工作线程，5_tomcat.md 排队账换个地方重演；且排队叠加进 RT，链路超时预算必须含它，18_feign.md 3.1.1）。

**Warm Up＝冷启动斜率爬升**：阈值不从 count 起步，而是从 **count / coldFactor（默认 3）** 起、随"系统持续通过的速率历史"逐步爬到 count——判定式基于"最近一次令牌生成时刻"：资源长期低载（静默超过估算的预热时长 10s×coldFactor 量级）后来流量，先按冷速率限住、给 JIT/连接池/本地缓存一段热身期（5_tomcat.md 4.1.1 JIT 预热、12_nginx.md 3.1.2"新节点摘除后瞬间全量打回"同款问题）。**场景**：突发型服务（秒杀开闸、cron 唤醒的检索进程、刚扩容完的新 Pod）；**长期恒载服务用它是错的**（没有"冷"可热，白抬拒绝率）。

**记忆锚（面试收口）**：**漏桶管"别急"（整形出口速率），预热管"慢来"（保护脆弱入口）**；两者都改变"超限时怎么办"，不改"超没超限"的判定输入——与快速失败互斥三选一，和 QPS/线程数阈值正交。

### 3.1.3 热点参数限流怎么实现？和大促场景怎么配合？

**语义**：对**同一资源、按参数值**分别限流——`queryItem(id)`：爆款 id=8888 单独 100 QPS、其余所有 id 合计 1000 QPS。声明时 entry 带 args：`SphU.entry("res", EntryType.IN, 1, itemId, uid)`（@SentinelResource 自动把方法入参带上，4.1.1）。

**规则（ParamFlowRule）**：`paramIdx`（第几个参数）+ 全局 count（**参数值未命中特例时的合计阈值**）+ **ParamFlowItem 特例列表**（指定值单独阈值）+ `exclusionItems`（**排除值**：这些值不受限——"免费商品随便刷"的反向白名单）+ 阈值类型也支持线程数。

**实现**：`sentinel-parameter-flow-control` 模块为资源维护 **ParameterMetric：每个参数位一个"参数值 → 独立滑动窗口序列"的 LRU Map**（默认容量 200，防高基数参数撑爆内存——**这即是能力也是红线**：拿 userId 这类近无界维度开热点限流，200 槽 LRU 会疯狂换入换出，统计失真且 CPU 白花——**热点参数只用于"有限基数 + 少数尖峰"的维度**：商品 ID、活动 ID、租户 ID，这是本题最重要的工程判断）。检查时：命中的每个参数值各自窗口独立计数、全部通过才放行，任一超限即 ParamFlowException。

**大促配合姿势**：① 爆款预热清单→参数特例规则**批量动态下发**（规则来自 Nacos，运营圈品即改规则，4.3.1 通道）；② 与"单机 vs 集群"阈值换算想清楚（5.1.1：热点规则集群模式支持按参数值的集群统计）；③ 参数维度与用户维度分两资源打（接口一个 entry 带两个参数，各限各的位）。

### 3.1.4 系统自适应保护是什么？和接口级限流怎么分工？

**SystemRule 视角换到整机**：检的是**入口流量**（Context 为 entry 的才检，中间资源不检——保护"总闸"而非"分闸"），五维指标任一越线即全局收紧：
- **load**（系统平均负载 load1，仅 Linux/Unix 的 `getSystemLoadAverage` 可得——Windows/部分容器拿不到即失效）；
- **平均 RT**（所有入口资源的 ClusterNode 平均耗时，ms）；
- **入口线程数 / 入口 QPS**（所有入口合计）；
- **CPU 使用率**（0~1，依赖 `com.sun.management` 扩展 MXBean，非所有 JVM 发行版可用——容器里要核对 cgroup 感知）。

**自适应的味道（BBR 借鉴，追问显深度）**：指标超阈值**不是无条件硬拦**——还比较当前全局 QPS 与"最高历史通过量"的关系：负载高但当前量还没到曾经过的底线时选择性放行，避免"抖一下全拒"的次生雪崩——**"保护"与"限流"的差别就在这一档**。

**分工**：接口级限流按 SLA 逐条定阈值（精细、要人维护）；系统规则是**兜底保险丝**（CPU 90% 时所有入口一起收，防的是"没配到接口的流量/意料外的热点"把机器打穿）——生产标配**两层都上**：接口层按压测值（5.2.1），系统层 CPU/load 红线 + 入口总 QPS 粗闸。**易错**：把系统规则当限流主力（维度太粗，一坏全坏），它只在"未知流量冲击"时出场。

## 3.2 熔断降级

### 3.2.1 熔断降级有哪几种策略？断路器状态机怎么转？

**三策略（1.8 起统一为 CircuitBreakerStrategy，窗口=statIntervalMs，默认 1s）**：

| 策略 | 判据 | 典型场景 |
|------|------|---------|
| **慢调用比例** SLOW_REQUEST_RATIO | 窗口内 **RT > count(ms)** 的调用占比 ≥ 比例阈值 | 下游变慢但还在吐 200（超时线之下的"温水煮青蛙"） |
| **异常比例** ERROR_RATIO | 窗口内异常数/请求数 ≥ 阈值 | 下游持续报错但偶有成功 |
| **异常数** ERROR_COUNT | 窗口内异常绝对数 ≥ 阈值 | 低频高价值接口（一次挂即熔断） |

**两个公共门槛**：窗口请求数 **minRequestAmount（默认 5）** 不满足时**不判熔断**——防小样本误伤（2 次请求挂 1 次就 50% 异常率的荒唐）；timeWindow（熔断时长，秒）为 OPEN 期的"冷静期"。

**状态机（把"半开"说准是这题的分水岭）**：

```text
CLOSED ──(窗口内判据命中 + 样本达标)──▶ OPEN：全部快速失败（DegradeException）
   ▲                                        │ timeWindow 到期
   │                          下一个真实请求到达（仅放这一个去试探）
   │                                        ▼
   └────── 试探调用成功（未超 RT/异常判据）──── HALF_OPEN
                     失败 → 重新 OPEN，再等一个 timeWindow
```

- **半开=单请求试探**（CircuitStateController 对"恢复窗口内第一个 entry"授权通行、其余仍拒）——旧资料"放 1~5 个探测请求"**不准确**，Sentinel 没有后台探测线程，放的是真实流量（和 K8s readiness 探活、12_nginx.md 3.1.2"恢复渐进"的话题接得上：**试探成功即刻全量放开，同样有"刚恢复被打回原形"的风险**——配 timeWindow 别太短 + 依赖侧预热保护（3.1.2 Warm Up 是下游服务的保险，熔断恢复是上游视角的保险，一句辨析即高分）；
- 熔断是**资源级**的（ClusterNode 维度）：A 实例熔断不影响 B 实例的断路器（各自进程内状态，不共享）——"要不要集群共享熔断状态"是开放话题（现状不支持，集群只做了限流的 token server，5.1.1）。

**旧口径纠偏两条（面试别沿袭）**：① "异常数=1 分钟内的数量"——1.8+ 统一用 statIntervalMs 窗口（默认 1s，可调），老文档是旧版实现；② 旧 grade=RT（平均 RT 超阈值直接断）——1.8 起被"慢调用比例"取代（平均值一个长尾就顶爆，比例稳健），新版 Dashboard 里就是慢调用比例。

### 3.2.2 熔断的统计口径里，"异常"和"RT"各是什么来源？哪些坑会造成"该熔断不熔断"？

**RT 口径**：StatisticSlot 在 exit 时把本次耗时累进窗口 RT 桶（2.1.1 MetricBucket），慢调用比例按"单次 RT > count"计数——**注意它测的是"业务代码执行段"**：`try-with-resources` 关闭（exit）发生在业务返回时，**Feign/RestTemplate 的远程耗时算得进，前提是那段调用在 entry/exit 区间内**；异步返回（future 立刻回来、真活在回调里）的 RT **测不到**——异步资源必须用 AsyncEntry/ContextCallback 把 exit 挪到真实完成点（2.2.1 exit 配对的异步版），否则"下游明明 5 秒，Sentinel 记 5 毫秒"——慢调用熔断全线失灵的隐性根因。

**异常口径**：SUCCESS/EXCEPTION 计数来自 **Tracer**：entry 块内**向外抛出的 Throwable** 由模板代码捕获计入；手动风格要 `Tracer.trace(ex)` 上报——**被业务 try-catch 咽掉的异常不产生异常统计**（对熔断"不可见"）：这是"异常比例熔断从不触发"排查清单第一条。两个细则：① BlockException（被限流/熔断）**不计入异常统计**——否则熔断触发→拒绝→拒绝也算异常→熔断永不恢复的死循环；② `exceptionsToIgnore`（@SentinelResource）可排除特定异常（参数校验类的"业务正常拒绝"不该拉高错误率——**"熔断该统计故障还是统计失败"的口径设计**，答出即资深）。

**"该断不断"五查**（排障模板）：① 资源有没有真实进出（适配器没生效/埋点错资源名）；② 异常咽没咽（Tracer 没见）；③ minRequestAmount 够不够（低频接口 1s 窗口样本永远不足——改异常数策略或拉长 statIntervalMs）；④ 判据单位（count 在慢调用里是**毫秒**、在异常比例里是 0~1——控制台填 50 当 50% 结果是 5000%）；⑤ 规则没到进程（4.3 推送链断了）。

# 4. 使用与集成

## 4.1 注解式接入

### 4.1.1 @SentinelResource 的 blockHandler、fallback、defaultFallback 怎么分工？有哪些 AOP 坑？

**三条出口管三种命运**：

```java
@SentinelResource(value = "getStock",
    blockHandler = "stockBlock",                  // 管 BlockException：限流/熔断/黑名单
    blockHandlerClass = StockHandlers.class,      //   外部类时方法必须 public static
    fallback = "stockFallback",                   // 管业务异常（Throwable，除排除项）
    defaultFallback = "defaultFallback",          // fallback 的通用兜底（签名：() 或 (Throwable)）
    exceptionsToIgnore = {BizCheckException.class}) // 这些异常不记错、不触发 fallback
public Stock getStock(Long itemId) { ... }

// 签名匹配是硬约束：
public static Stock stockBlock(Long itemId, BlockException ex) { ... }  // 原参数 + BlockException
public Stock stockFallback(Long itemId, Throwable ex) { ... }           // 原参数 + 可选 Throwable
```

**分发规则**：BlockException 族 → blockHandler（**签名不匹配/未提供 → BlockException 原样抛给调用方**——"配了 fallback 为什么限流异常还是炸出来"的答案：fallback 不接 BlockException，两者井水不犯河水）；其余异常 → fallback，未配则退 defaultFallback，再无则原样抛。**blockHandler 里按异常类型分流**（FlowException→"稍后再试"、DegradeException→切备用）是 1.2.1 概念区分的落地处。

**参数与热点**：方法入参自动成为 args（3.1.3 热点规则吃它）；不想暴露的参数放 `args` 之外（`@SentinelResource` 无排除入参位之说，热点规则用 paramIdx 精确选位）。

**Spring AOP 边界四坑（这题的另一半分值）**：
1. **同类自调用失效**——`this.method()` 不过代理，注解白贴（Spring AOP 通用病，7_spring.md、8_spring_mvc.md 讲过的代理模型；解法：拆类或 AopContext）；
2. **private/ final 方法**不被代理；
3. **裸用 sentinel-core 忘注册切面**——`@Bean SentinelResourceAspect`（SCA starter 已自动配，手写 core 接入时最容易漏——"注解完全没反应"第一查）；
4. **异步方法**：`@Async`/CompletableFuture 场景 entry/exit 在提交段就配对完成，异常与 RT 都在回调线程里发生——切面看不见（熔断失灵的 3.2.2 异步根因在注解场景的原样复刻；需手动 AsyncEntry 或把异步边界包成同步资源）。

**全局收口**：`@SentinelResource` 适合"核心少数方法精细控制降级语义"；**海量接口的资源定义与统一 429 响应交给适配器和 BlockExceptionHandler**（4.2.1）——逐方法写 handler 的团队三个月后一定烂尾，答出这句即工程化视角。

## 4.2 生态集成

### 4.2.1 HTTP/Feign/Gateway 接口是怎么"自动变资源"的？来源黑白名单怎么生效？

**统一的适配模式**（记住一个公式，全家桶都好答）：**适配器 = 自动 entry/exit + BlockException→框架降级出口的桥**——
- **HTTP（SCA 自动装配）**：所有 MVC/WebFlux 端点自动注册为资源，资源名=URL 模式（如 `/api/products/{id}` 经 urlCleaner 归并，防路径参数打爆资源表）；被拦时默认抛 `BlockException` → 全局 `BlockExceptionHandler` 统一渲染（自定义它返回 429 JSON——比每接口写 handler 省力）；
- **Feign**：开 `sentinel.enabled` 后每个 Feign 方法成资源（名如 `GET:http://order-service/orders/{id}`），BlockException 桥接到 fallback/fallbackFactory（机制与选型账在 18_feign.md 4.1.2，此处只补资源名口径）；
- **Gateway**：路由级资源（Route id）+ API 分组聚合，拦截发生在转发前（19_spring_cloud_gateway.md 3.1.2）；
- Dubbo/Spring Cloud Stream 同款套路——**面试报"适配器统一模式"再举两例，胜过背四张表**。

**origin 与授权规则**（AuthorityRule 的落地链）：资源被调时适配器把"调用来源"解析进 Context（HTTP 默认取请求参数 `origin`，自定义 `RequestOriginParser` 改从 header/证书取）→ OriginNode 分来源统计（2.1.2）→ AuthorityRule 按 origin 白/黑名单直接拒（策略 0 白 1 黑）。**两个必踩坑**：① 来源没解析出来（全是默认 `default`/`other`）→ 黑白名单全员误伤或全员放行；② **调用方不经过适配器时 origin 缺省**——内部服务互调要在 Feign 拦截器统一带上来源标识（18_feign.md 4.2.1 的头透传在此复用，鉴权层级的最内圈）。

**@SentinelResource vs 适配器怎么选**：要参数维度（热点）、精细 fallback → 注解；要"全站接口默认受保护、统一响应" → 适配器——**生产是叠加使用**：适配器铺面、注解点睛（2.2.1 链模型下两者进的是同一条 Slot 链，无冲突）。

## 4.3 控制面

### 4.3.1 规则怎么下发和持久化？"推模式到 Nacos"的真实边界在哪？

**规则即内存对象 + 可插拔数据源**：全部规则（Flow/Degrade/ParamFlow/System/Authority）本质是 JVM 内存里的 List + 变更监听器（`FlowRuleManager.loadRules(...)`）。**DataSource SPI** 把"从哪拿规则、怎么感知变更"标准化：实现 `ReadableDataSource`（pull 定时读）或自带 watch 的 push 源（Nacos/Apollo/ZK 客户端注册监听），配 `Converter`（JSON→规则对象）即可接入。

**SCA 属性化接入（常用件）**：

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow-rules:                     # 任意名字，可多源
          nacos:
            server-addr: nacos:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow             # flow/degrade/param-flow/system/authority
```

Nacos 改 JSON → 长连接推送（17_nacos.md 3.1.2 通道）→ DataSource 回调 loadRules 热生效——**规则动态化的完整闭环就这三步**；规则 JSON 字段与 FlowRule 属性同名（3.1.1 的三要素、5.1.1 的 clusterConfig 都在 JSON 里）。

**必须说清的边界（这题的高分区）**：默认 Dashboard 的"改规则"按钮**写的是目标应用的内存**（command API `/setRules`）——**不写 Nacos，应用重启规则蒸发**；资料里"控制台修改→自动推送 Nacos"的推模式**需要二选一才成立**：① 改造/替换 Dashboard 的 push 实现（官方给过示例模块，开源主仓默认不带）；② **以配置中心为唯一事实源**：规则 JSON 直接在 Nacos 维护（或平台/流水线写入），Dashboard 只作监控与临时试验——**2026 生产共识是②**（Git/配置中心审计链 > 控制台点按钮；"谁在控制台手滑改了大促限流值"必须可回溯，17_nacos.md 3.2.1 监听查询/历史回滚全适用）。多源并存注意**加载顺序与互相覆盖**：同名资源规则来自两源=后加载者赢，团队要定"规则只进一个源"的纪律。

### 4.3.2 Sentinel Dashboard 的原理？为什么重启控制台就查不到历史？

**两个通道的星型架构**：
1. **心跳上报（应用→控制台）**：客户端 transport 模块定期（约 10s）向 `csp.sentinel.dashboard.server` 注册实例信息（应用名/IP/命令端口）——控制台的"机器列表/簇点链路"来自这份注册表；
2. **指标拉取（控制台→应用）**：点开实时监控时，控制台经**命令端口（默认 8719，占用自增）**按需拉取该实例滑动窗口的秒级数据（2.1.1 的 MetricBucket 序列化返回）——曲线是**拉出来的快照序列，不是推出来的历史**。

**"历史没了"的答案**：控制台**不落库**——指标缓存在其内存，重启清零；规则本来就存在应用侧内存（4.3.1）——**Dashboard 从设计上只是"观测面板 + 临时遥控器"，不是管控平台**：生产要长期曲线/告警 → 指标导出接 Prometheus/自建存储（exporter/云 AHAS）；要规则审计/多环境 → Nacos 为源（4.3.1 收口句）。**追问三连**：① "看不到应用"查心跳（dashboard 地址、网络、版本 transport 配置）；② "实时曲线空"查命令端口连通（防火墙只开业务端口的经典事故，同 17_nacos.md 9848 之坑）；③ "集群维度 QPS 与单机之和不等"——origin/采样窗口对齐差，别拿控制台当计量系统。

# 5. 集群与实践

## 5.1 集群流控

### 5.1.1 集群流控怎么实现全局精确限流？Token Server 的账怎么算？

**问题回看**：单机规则下"阈值 100 × 20 实例"=全局最坏 2000 QPS，且流量在实例间不均（有的撑死有的闲死）——**全局精确需要唯一计数点**（12_nginx.md 3.2.1 单实例视角问题的最终答案，另一条路是 Redis 计数：19_spring_cloud_gateway.md 3.1.1——两条路线对照着答）。

**机制（Token Server/Client）**：每条集群规则有 flowId；client 收到请求先**远程申请 token**（netty 长连接，携带规则与本次计数），server 端维护该 flowId 的全局滑动窗口并判定发/拒；被拒走正常 BlockException 流程。**集群阈值类型**可选"全局额度"或"均摊/单机保底组合"（thresholdType：全局总量 vs 每机独占；失败行为：连不上 server 时按本地回退阈值降级为单机限流——**这是设计里最重要的容错位**：控制面故障不得拖垮数据面）。

**两种部署形态**：
- **内嵌（embedded）**：集群里选/指定一台业务实例兼任 server——省机器，但**业务 GC/宕机直接抖动控制面**（该实例 RT 恶化→全员申请 token 超时→集体回退本地限流，5.2.2 症状）；
- **独立（alone）**：专职 token server 进程/小集群——稳定但要运维；开源版**单点无法定数副本**（不像 Nacos 的 Distro/JRaft 分级，17_nacos.md 2.2），HA 靠"快速故障回退本地"兜可用性而非精确性——**精确全局限流与强高可用在开源 Sentinel 集群模式里不可兼得，要么接受回退窗口，要么买 AHAS/自建仲裁**。

**选型判断（这题的满分段）**：① 先问"要不要全局精确"——**大多数接口用"单机阈值=压测值"已足够**，全局限流的刚需场景是：总容量硬约束（库存扣减、外部配额 API、按租户合同限总额）；. 要精确时先评估 **Redis Lua 方案**（团队已养 Redis、量级不大——网关层已这么干，19_spring_cloud_gateway.md 3.1.1）vs token server（高频申请的性能账与多一跳延迟，但对热点场景的内存粒度更细）；③ 弹性伸缩/容器化下实例频繁生死，集群流控的连接治理成本会显性上涨——报得出这句即真踩过。

## 5.2 落地实践

### 5.2.1 限流阈值怎么定？给一套大促全链路防护方案。

**阈值不是拍的是压的（方法链）**：
1. **单机压测定线**：以"P99 不劣化"为约束找单机最大 QPS（13_rocketmq.md 3.3.2 容量基线同法）——**并发线程数阈值**按利特尔法则反推：`线程数 ≈ 目标 QPS × 平均 RT`（5_tomcat.md 4.1.2），RT 0.2s × 200 QPS=40 并发起步再调；
2. **乘安全系数**：规则值=压测线 ×0.7~0.8（给 GC 抖动、依赖劣化留缓冲，2_jvm.md）；
3. **换算层间预算**：入口 Nginx 粗限 > 网关总限 > 本服务单机限 > 各接口限（逐层收紧、维度递减，15_spring_cloud.md 3.2.3 预算链的"量版"）；
4. **压测后回填热点清单**：爆款商品/头部租户单独 ParamFlowItem（3.1.3）。

**分层防护矩阵（把组件各就各位）**：

| 层 | 件 | 职责 |
|----|----|------|
| 入口 | Nginx/云 LB | 无差别洪水（爬虫/攻击/瞬时尖峰）粗限，429 兜底页（12_nginx.md 3.2.1/4.2.1） |
| 网关 | SCG + Sentinel | 按路由/用户维度全站细限、熔断下游、灰度分流（19_spring_cloud_gateway.md 3.1） |
| 应用 | Sentinel SDK | 接口 QPS/线程数、热点参数、系统保险丝、对下游 Feign 的熔断（本篇） |
| 消息 | RocketMQ | 削峰蓄水池 + 匀速排队消费（13_rocketmq.md 3.3.2、3.1.2） |
| 数据 | DB/Redis | 慢 SQL 防护、连接池上限（16_mysql.md、27_redis.md）——治理的最后一米常在这里 |

**降级预案是限流的另一半**（只限不降=把 429 甩给用户）：分级清单——非核心功能开关（评价/推荐置灰）、读接口回源缓存/静态兜底、写接口异步化排队（MQ 承接）、核心链路保底扩容；**每个 BlockException 出口都有对应的降级 UX**（1.2.1 分工表的演练版）。规则与开关全部走配置中心动态下发（4.3.1），**大促前全员演练一遍"推 Nacos 看生效"**（17_nacos.md 3.2.1 监听查询在此成为验收工具）。

### 5.2.2 Sentinel 上线后常见的"玄学症状"各对应什么根因？

| 症状 | 根因清单 | 定位动作 |
|------|---------|---------|
| 资源压根不出现在控制台 | 应用没进链（适配器没生效/裸 core 忘 Aspect）、心跳断（dashboard 地址/网络） | 查启动日志 transport、curl 心跳接口 |
| 规则配了不拦截 | 资源名不匹配（Feign 是 `GET:http://svc/path` 模板、HTTP 经 urlCleaner——4.2.1）、阈值大错单位、多源覆盖（4.3.1）、集群规则没配 clusterConfig 却期望全局限 | 控制台"簇点链路"核对真实资源名，规则按名重填 |
| 偶发大量拒绝（远低于阈值的 QPS） | **滑动窗口边界毛刺**（2.1.1）、重试风暴叠加（18_feign.md 2.2.3）、热点 LRU 换入换出虚高（3.1.3） | 看秒级曲线找尖刺形状，对齐窗口/重试参数 |
| 熔断反复开合（flapping） | timeWindow 过短、依赖处于"半死不活"（恢复即慢）、平均 RT 类旧策略长尾敏感（3.2.1 已纠偏） | 拉长恢复窗口、改比例策略、加 minRequestAmount |
| 限流后雪崩反而更快 | 快速失败 + 调用方**无退避疯狂重试**（把被拒流量放大）；排队模式把 RT 顶穿上游超时（3.1.2） | 退避+抖动写进调用方，排队只给能等的异步流量 |
| 线程数规则"卡死后永久限流" | exit 泄漏（异常分支没归还/异步未 AsyncEntry，2.2.1/4.1.1） | 看 ClusterNode threadNum 是否只增不减 |
| 集群模式抖动、集体退化单机 | token server 内嵌宿主 GC/宕机（5.1.1）、8719/命令端口不通 | 压测断网演练回退路径 |
| 控制台数据"不新鲜" | 拉取型监控按需刷新（4.3.2）、心跳 10s 粒度——**它本来就不是 Prometheus** | 长期观测换指标导出 |

**通用排查顺序总结**：资源在不在（链与注册）→ 名字对不对（资源名匹配）→ 数据真不真（口径：异常咽没咽、RT 测没测到、窗口毛刺）→ 拓扑稳不稳（dashboard/命令端口/心跳/token server 三个非常规端口连通性）——**"治理件失效通常不是算法错了，是名字、端口、口径三件事错了"**，这句收口即实战派。