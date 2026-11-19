# 1. 本篇读法与调用栈全景

本篇把 `4_flow_control.md` 的规则语义翻译成代码：**一次 entry 请求在流控这条路上到底经过哪几层、每层的判定输入从哪来**。版本基线 1.8.6（`sentinel-core`），涉及的类集中在三个包：

| 层 | 类 | 职责 |
|----|----|------|
| Slot | `com.alibaba.csp.sentinel.slots.block.flow.FlowSlot` | 链上的挂载点，本身无逻辑 |
| Checker | `FlowRuleChecker`（`slots.block.flow`） | 逐条规则判定 + 选统计节点 |
| Rater | `TrafficShapingController` 三实现（`slots.block.flow.controller`） | 具体算法（快速失败/预热/排队） |
| 规则 | `FlowRuleManager` / `FlowRule` | 规则表 + 加载期把 `controlBehavior` 编译成 rater 对象 |
| 统计 | `ClusterNode` / `DefaultNode` / `MetricBucket` / `LeapArray` | 被 rater 读取的数字从哪来（细节在 `18_statistics_engine.md`） |
| 集群 | `ClusterFlowChecker` / `TokenClient`（`slots.cluster`） | `clusterMode=true` 时改走 token server |

一条请求的完整栈（单机、直接模式、QPS）：`SphU.entry` → `CtSph#entryWithPriority` → 取/建 `ProcessorSlotChain` → `ProcessorSlot#entry` 依次 fire → `StatistricSlot`（先记 pass 前的东西）→ `SystemSlot` → `FlowSlot` → `FlowRuleChecker#canPassCheck` → `selectNodeByRequesterAndStrategy` → `rater#canPass` → 通过则继续 fire 到链尾 → 业务；拒绝则抛 `FlowException`（`2_architecture.md` §3.3 的链序，`15_slot_chain_source.md` 的链构建与缓存）。**读源码时最容易走错的直觉**：这里没有"限流器对象"常驻在资源上——限流能力=「规则表里的一条 FlowRule + 它编译出的 rater」，统计在节点上，两者在 Checker 里每请求现拉现配对，这就是"改规则不需要重启、也不用重建代理"的实现根（`14_dynamic_rules.md` §8.1）。

原稿各段代码是教学化骨架，方向对但细节与真实实现有出入；下文每段代码后都给出真实形态（标注"还原"）与勘误点。

# 2. FlowSlot — 入口

```java
public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized,
                      Object... args) throws Throwable {
        // 核心：调用 FlowRuleChecker 检查所有流控规则
        FlowRuleChecker.checkFlow(resourceWrapper, context, node, count, prioritized);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }
}
```

## 2.1 【纠偏】真实形态：实例字段 + 函数式规则提供者

真实 `FlowSlot` 里没有静态方法调用，而是持有一个 checker 实例，并把"按资源名取规则"作为**函数**注入（还原，简化）：

```java
// 还原（简化）
public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {
    private final FlowRuleChecker checker;

    public FlowSlot() {
        this(new FlowRuleChecker());
    }

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node,
                      int count, boolean prioritized, Object... args) throws Throwable {
        checkFlow(resourceWrapper, context, node, count, prioritized);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    void checkFlow(ResourceWrapper resource, Context context, DefaultNode node,
                   int count, boolean prioritized) throws BlockException {
        checker.checkFlow(FlowRuleManager::getRulesOfResource, resource, context, node, count, prioritized);
    }
}
```

这个差异不只是风格：**函数式 `ruleProvider` 让 Checker 与 Manager 解耦**（单测/自定义规则源可注入），同时把"每次判定都现查规则表"钉成事实——规则表变更对 Slot 完全透明，没有缓存、没有失效通知。另一处要留意的实现约束：`FlowSlot` 是**无状态**的（状态全在规则对象与统计节点上），所以链上这个 Slot 实例可以被同资源的所有线程共享、并在 SPI 构建时被 `SlotChainBuilder` 复用——理解这点才能理解"排队模式下 sleep 的是业务线程，不是某个 slot 线程"（§5.7）。链上没有 `exit` 逻辑（`FlowSlot` 不重写 exit），流控是纯 entry 期决策，这与熔断（依赖 exit 上报，`17_circuit_breaker_source.md` §2.1）构成两类规则的根本形态差。

# 3. FlowRuleChecker — 规则校验器

```java
public class FlowRuleChecker {

    public static void checkFlow(ResourceWrapper resource, Context context,
                                  DefaultNode node, int count,
                                  boolean prioritized) throws BlockException {

        // 1. 获取该资源的流控规则
        List<FlowRule> rules = FlowRuleManager.getRules(resource.getName());
        if (rules == null || rules.isEmpty()) {
            return;  // 无规则，直接通过
        }

        for (FlowRule rule : rules) {
            // 2. 判断是否需要检查此规则（limitApp 匹配）
            if (!FlowRuleChecker.passCheck(rule, context, node, count, prioritized)) {
                // 3. 触发流控
                throw new FlowException(rule.getLimitApp(), rule);
            }
        }
    }
}
```

## 3.1 真实骨架与三条语义

真实签名（还原，简化）：`public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource, Context context, DefaultNode node, int count, boolean prioritized)`，内部对每条规则调 `canPassCheck(...)`，失败即 `throw new FlowException(rule.getLimitApp(), rule)`。方法名两点纠偏：Manager 侧的取规则 API 是 **`FlowRuleManager.getRulesOfResource(String)`**（`getRules()` 是无参的"全量规则列表"，给控制台/数据源用），Checker 自己的判定方法是 **`canPassCheck`**（不是 `passCheck`）。

三段骨架给出的是流控判定的**全部宏观语义**，值得逐条钉牢：

| 语义 | 代码依据 | 运维后果 |
|------|---------|---------|
| 无规则 → 直接 return（零成本快路径） | `rules == null \|\| isEmpty` 短路 | 大量资源没配规则时，链上开销≈统计本身；"限流器拖慢 RT"的锅多数不在 Checker 遍历 |
| **逐条遍历，第一条不过就抛** | `for` + throw | 多规则串联=最严者胜，且**被哪条拦的取决于列表顺序**：`FlowException(rule.getLimitApp(), rule)` 里带了 rule，但异常 message 只含 limitApp——想定位具体阈值要看 `sentinel-record.log` 的 Block 记录（`21_logging_monitoring.md`） |
| 每条规则独立判定，前面通过≠后面不拦 | 同上 | "同资源配了 100/200 两条"出现的是 100 说话，不是"取最大" |

遍历顺序也有实现细节：`FlowRuleManager` 按资源分组时把 `limitApp="default"` 的规则**排在同资源列表靠前**（还原：加载期会按 limitApp 归属排序，default 优先、具体 origin 其次、`other` 最后），目的是让通用规则先跑、少做无用的 origin 节点查找；这决定了"default 规则和 origin 规则同时超阈值时，异常里看到的是 default 那条"。

## 3.2 canPassCheck 的真实分派：clusterMode 在Checker 里短路

```java
// 还原（简化）：canPassCheck 的分支结构
public boolean canPassCheck(FlowRule rule, Context context, DefaultNode node,
                            int acquireCount, boolean prioritized) {
    String limitApp = rule.getLimitApp();
    if (limitApp == null) {
        return true;                       // 未指定 limitApp：视为不拦（与 Dashboard 表单不同，代码构造可漏）
    }
    if (rule.isClusterMode()) {
        return clusterChecker.canPass(rule, context, node, acquireCount, prioritized);  // §8
    }
    TrafficShapingController rater = rule.getRater();
    if (rater == null) {
        return true;
    }
    Node refNode = selectNodeByRequesterAndStrategy(rule, context, node);
    if (refNode == null) {
        return true;                       // 找不到统计节点 → 放行（fail-open）
    }
    return rater.canPass(refNode, acquireCount, prioritized);
}
```

`limitApp == null` 与 `rater == null` 两处 return true 是**代码构造规则**（非 Dashboard/JSON）才会遇到的"规则写了但完全不拦"来源；`refNode == null → 放行` 是本篇第一条要背下来的 **fail-open 纪律**：关联/链路模式下引用的统计对象不存在（资源从没被调用过、名字写错），规则静默失效且不报错（§4.4、§4.6 展开，`3_quick_start.md` §6.2 的资源名对账纪律就是它的应用层表现）。

# 4. 流控模式匹配（节点选择）

## 4.1 直接模式（STRATEGY_DIRECT）

```java
public static boolean passCheck(FlowRule rule, Context context,
                                 DefaultNode node, int count, boolean prioritized) {
    // 获取对应的统计节点
    Node selectedNode = selectNodeByRequesterAndStrategy(
        rule, context, node);
    if (selectedNode == null) {
        return true;
    }
    // 调用流量整形控制器判断
    return rule.getRater().canPass(selectedNode, count, prioritized);
}
```

## 4.2 真实的选择器：limitApp 与 strategy 是两个正交维度

`selectNodeByRequesterAndStrategy` 的决策矩阵（还原，简化）：

```java
// 还原：真实结构是"先看 limitApp 定归属，再看 strategy 定换哪个节点"
String origin = context.getOrigin();
String limitApp = rule.getLimitApp();
int strategy = rule.getStrategy();

if (limitApp.equals(origin) && !origin.isEmpty()) {
    // 规则针对当前调用方身份：用该 origin 在本资源下的统计节点
    return context.getCurOriginNode();               // DirectNode（origin 维度）
} else if (RuleConstant.LIMIT_APP_OTHER.equals(limitApp)
           && !origin.isEmpty() && !hasRuleForThisOrigin(...)) {
    return context.getCurOriginNode();               // "other" 的兜底语义
} else if (RuleConstant.LIMIT_APP_DEFAULT.equals(limitApp)) {
    if (strategy == RuleConstant.STRATEGY_RELATE) {
        return ClusterBuilderSlot.getClusterNode(rule.getRefResource());
    }
    if (strategy == RuleConstant.STRATEGY_CHAIN) {
        if (!context.getName().equals(rule.getRefResource())) {
            return null;                             // 入口不匹配 → 该规则不适用
        }
        return context.getCurNode();                 // 该 context 下的 DefaultNode
    }
    return node;                                     // 直接模式：本资源的 DefaultNode
}
return null;
```

读这张矩阵的正确姿势：**limitApp 决定"谁的流量"（default=全部流量看 ClusterNode 口径；具体 origin/other=只看这个来源，用 `DirectNode`），strategy 决定"看哪个资源的账"（直接=本资源；关联=refResource 的 ClusterNode；链路=当前 context 名等于 refResource 时看本资源在该 context 下的 DefaultNode）**。原稿把三者画成并列的三小节，容易读成"三选一实现"，实际是 `2 × 3` 的组合空间，且 limitApp 非 default 时 strategy 仍会参与（先按 origin 归并，再按 strategy 换统计口径）。

## 4.3 关联模式（STRATEGY_RELATE）

```java
// 如果关联资源 refResource 达到阈值，限流当前资源
private static Node selectNodeByRelate(FlowRule rule, Context context) {
    String refResource = rule.getRefResource();
    // 获取关联资源的 ClusterNode
    ClusterNode refNode = ClusterBuilderSlot.getClusterNode(refResource);
    if (refNode == null) {
        return null;
    }
    // 使用关联资源的统计数据进行判断
    return refNode;
}
```

## 4.4 关联模式的三个实现推论

骨架方向准确（真实代码就在 `selectNodeByRequesterAndStrategy` 内部，`ClusterBuilderSlot.getClusterNode(refResource)`），三条推论值得写进配置评审：**① 关联读的是 `ClusterNode`（资源全局账），不是当前 context 的账**——所以关联模式下 `limitApp` 与 origin 归并的细节实际作用在"被关联资源的全局阈值"上；**② 判定用的是 rater 对着 refNode 算**：`count` 语义变成"refResource 的 QPS/线程数达到 count 时，拦当前资源"，`4_flow_control.md` §3.3 的"反向保护"（写满则拒读）在代码里就是这一句，没有任何特殊分支——**"关联资源达到阈值"不等于"达到 refResource 自己的规则阈值"**，它只看本条规则里写的 count，这是关联模式最普遍的语义误读（ref 自己的规则与这条毫无关系）；**③ refResource 从未被调用 → `getClusterNode` 返回 null → §3.2 的 fail-open**。另外 `ClusterBuilderSlot` 的节点表是懒建的（首次 entry 才注册），所以"先启动、立刻打规则、再压 ref 资源"的验证顺序不能反（`3_quick_start.md` §6.2）。

## 4.5 链路模式（STRATEGY_CHAIN）

```java
private static Node selectNodeByChain(FlowRule rule, Context context,
                                       DefaultNode node) {
    String refResource = rule.getRefResource();
    // 只统计从特定入口进入的流量
    DefaultNode originNode = node.getOriginNode(refResource);
    return originNode;
}
```

## 4.6 【勘误】DefaultNode 没有 getOriginNode(refResource)

这段是原稿里最需要纠正的一处：`DefaultNode` 上没有按 context 取子节点的方法，真实实现是**比较 Context 名与 refResource**，命中后用 `context.getCurNode()`（即该资源在"当前入口 context"下的 DefaultNode）：

```java
// 还原
if (strategy == RuleConstant.STRATEGY_CHAIN) {
    if (!context.getName().equals(rule.getRefResource())) {
        return null;          // 当前请求不是从这个入口进来的，规则不适用
    }
    return context.getCurNode();
}
```

由此产生三条只有读代码才知道的行为：**① 匹配是 `String.equals` 全等**，不是前缀也不是模式——入口名来自 `ContextUtil.enter(name, origin)`（Web 集成里=归一化后的入口 URL，`10_spring_cloud_integration.md` §3.5），被调资源与入口资源名之间必须逐字符对得上，`/api/order` 与 `/api/order/` 是两个名字；**② 返回 null 是"规则不适用"而非"拒绝"**（同 §3.2 的 fail-open 家族），所以链路规则对"别的入口进来的流量"完全放行——这是它的正确用法（只限某个入口路径），也是它被误配成"我以为限了这个资源"的成因；**③ 统计载体 `DefaultNode` 天然是 (context, resource) 二维**：同一资源在不同入口下计数互不相干（`18_statistics_engine.md` §3 的节点模型），链路模式就是这一结构的对外暴露。

# 5. TrafficShapingController — 流量整形

```java
public interface TrafficShapingController {

    // 判断是否可以放行
    boolean canPass(Node node, int acquireCount, boolean prioritized);

    // 是否可以继续排队
    boolean canPass(Node node, int acquireCount);
}
```

## 5.1 接口三方法与两个两难设计

真实接口是三个方法（还原）：`boolean canPass(Node node, int acquireCount, boolean prioritized)`、`boolean canPass(Node node, int acquireCount)`（默认语义=不带优先级的便捷重载，内部委托 `canPass(node, count, false)`），以及 `void passUsingToken(...)` 一类给集群/内部复用路径的钩子（不同版本形态略有出入，以所用版本为准）。原稿把双参重载注释成"是否可以继续排队"不准确——**排队能力不在接口层，而在具体实现是否阻塞当前线程**：`RateLimiterController` 的双参/三参版本都会 sleep，`DefaultController` 的双参版本只是不看优先级。

| 实现 | 算法原型 | 读哪个统计 | 是否阻塞 | 生效前提 |
|------|---------|-----------|---------|---------|
| `DefaultController` | 阈值比较（快速失败） | 当前窗口 passQps / 实时线程数 | 否 | 无（默认） |
| `WarmUpController` | Guava SmoothWarmingUp 移植（storedTokens 令牌桶） | `previousPassQps()`（上一秒） | 否 | 仅 grade=QPS |
| `RateLimiterController` | 漏桶（预约通过时间线） | 自己的 `latestPassedTime`，不读窗口 | **是（sleep）** | 仅 grade=QPS |

这张表是 `4_flow_control.md` §4"流控效果"三种选择的代码真身，也是 §7.1 那张 `generateRater` 分派表的依据。

## 5.2 DefaultController（快速失败）

```java
public class DefaultController implements TrafficShapingController {

    private double count;  // 阈值

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        // 获取当前 QPS/线程数
        int curCount = avgUsedTokens(node);
        // 超过阈值则拒绝
        if (curCount + acquireCount > count) {
            // 是否有优先级等待（保留给高优先级请求）
            if (prioritized && grade == RuleConstant.FLOW_GRADE_QPS) {
                long currentTime = TimeUtil.currentTimeMillis();
                long waitTime = node.tryOccupyNext(currentTime,
                    acquireCount, count);
                if (waitTime < OccupyTimeoutProperty.getOccupyTimeout()) {
                    node.addOccupiedPass(acquireCount);
                    return true;  // 预占未来的令牌
                }
            }
            return false;  // 拒绝
        }
        return true;  // 通过
    }
}
```

## 5.3 avgUsedTokens 与"占位"的真实语义

真实 `avgUsedTokens`（还原）体现 grade 的分派：**`grade == FLOW_GRADE_THREAD ? node.curThreadNum() : (int) node.passQps()`**。两条语义要分开读：线程数模式读的是**此刻活跃 Entry 数**（`StatisticSlot.exit` 会减，`2_architecture.md` §4.3），是瞬时并发，不看窗口；QPS 模式读的是**当前窗口已完成桶的 passQps**，其中就包含别的 controller 预占进来的量。骨架缺的一行很关键：真实代码在 `waitInMs < occupyTimeout` 分支里除了 `addOccupiedPass` 还会 **`node.addWaitingRequest(currentTime + waitInMs, acquireCount)` 并 `sleep(waitInMs)`**——即"预占未来令牌"不是空头支票，而是**当前线程真的睡到那个时刻再往下走**（`ClusterNode#tryOccupyNext` 会在未来窗口上挂 `futureOccupiedBuckets`）。

由此得到 `prioritized` 的完整账：它**只在 `SphU.entryWithPriority(...)` 时才为 true**（普通 `SphU.entry` 恒 false，所以这段代码在绝大多数应用里从未执行——Feign 系列里给"重要接口"留路的正确姿势是显式用 entryWithPriority，Web/注解资源默认都吃不到）。等待上限由 `OccupyTimeoutProperty` 提供（默认 20ms 量级，可通过 Config `csp.sentinel.occupy.timeout` 调整，以版本为准），超过则拒绝。抢占的代价也必须写进结论：优先级请求把额度预支到未来窗口，**同资源的普通请求随后会因此更早被拒**，这是"保障关键流量"的账单形式；且被 sleep 的线程与排队模式一样占着容器线程（§5.7 同款风险）。

## 5.4 WarmUpController（预热）

```java
public class WarmUpController implements TrafficShapingController {

    protected double count;              // 稳定阈值
    private int coldFactor;              // 冷启动因子（默认 3）
    protected int warningToken = 0;      // 警戒令牌数
    private int maxToken;                // 最大令牌数
    protected double slope;              // 斜率

    // 基于 Guava SmoothWarmingUp 算法
    public WarmUpController(double count, int warmUpPeriodInSec,
                            int coldFactor) {
        this.count = count;
        this.coldFactor = coldFactor;
        
        // 计算令牌桶容量
        this.maxToken = warmUpPeriodInSec * count;
        this.slope = (coldFactor - 1.0) * count / (maxToken - warningToken);
    }

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        long previousQps = node.previousPassQps();
        // 同步令牌桶
        syncToken(previousQps);

        long restToken = storedTokens.get();
        if (restToken >= warningToken) {
            // 预热阶段：限制通过量
            long aboveToken = restToken - warningToken;
            double warningQps = Math.nextUp(
                1.0 / (aboveToken * slope + 1.0 / count));
            if (previousQps + acquireCount <= warningQps) {
                return true;
            }
        } else {
            // 稳定阶段：正常放行
            if (previousQps + acquireCount <= count) {
                return true;
            }
        }
        return false;
    }
}
```

## 5.5 构造公式纠偏与"机制/效果"两面

**① 公式纠偏**（还原，真实 `construct`）：骨架里 `maxToken = warmUpPeriodInSec * count`、`slope = (coldFactor-1)*count/(maxToken-warningToken)` 两项都不准确，且 `warningToken` 停在初始 0 没算。真实的是：

```java
// 还原
this.warningToken = (int) (warmUpPeriodInSec * count) / (coldFactor - 1);
this.maxToken = warningToken + (int) (2 * warmUpPeriodInSec * count / (1.0 + coldFactor));
this.slope = (coldFactor - 1.0) / count / (maxToken - warningToken);
```

量纲读法：`storedTokens` 的单位是**"秒 × QPS"=可积压的请求数**，`warningToken` 是"冷启动警戒线"——**桶里存量高于 warningToken 才进入预热限流区**（存量高=系统闲=刚启动/刚恢复），低于则直接按 `count` 放行。`coldFactor` 决定起步速度：初始满桶时 `aboveToken` 最大，`warningQps = 1/(aboveToken*slope + 1/count)` 退化为 `count / coldFactor`，也就是**预热第一拍的实际允许 QPS = 阈值 ÷ coldFactor（默认 3）**，随流量消耗令牌沿 slope 爬到 count——这正是 `4_flow_control.md` §4.3"冷启动阈值 = count/coldFactor"的推导处。

**② "机制 vs 效果"两面**：`4_flow_control.md` 里那句"WarmUp 不是令牌桶而是阈值爬升"要和这里合起来读，别当成矛盾：**机制上它确实是 Guava SmoothWarmingUp 的移植**（storedTokens 令牌桶 + slope 计算动态上限，这是源码事实），**对外效果是允许 QPS 从 count/coldFactor 线性爬到 count**（这是配置者看到的语义）。两者不冲突，冲突的是"它会像令牌桶那样容忍突发"这种误期——它对**突发流量没有任何容忍**（突发只会更快被限），它容忍的是"低历史流量之后逐步放量"。

**③ 输入口径纠偏**：判定读的是 `previousPassQps()`（**上一个 1s 窗口的 pass QPS**），不是当前窗口累计——所以预热看的是"历史一秒的负载"，冷启动判定天然滞后 1s；桶的补充（`syncToken`/`coolDownTokens`，还原）按 `(currentTime - lastFillTime)/1000 × count` 逐秒填充并在空闲时把存量抬向 maxToken，同时把 `lastFillTime` 用 CAS 推到整秒，避免同一秒重复补。三个工程后果：**warmUpPeriodSec 配短了基本无效**（爬升期没走完就进稳定区）；**统计为 0 的长时间空闲后满桶**，恰好在冷态下最保守（这是设计意图：给 JIT/连接池/本地缓存时间）；**对定时任务式脉冲流量它会把每一波都当冷启动**，这类场景要用"预热 + 排队"组合（§7.1 的第四种 controller）或干脆用普通阈值。

**④ 集群模式不支持预热排队**：`clusterMode=true` 时判定交给 token server 的 `ClusterFlowChecker`（只实现快速失败语义，`4_flow_control.md` §6.2 已提），`rule.getRater()` 在这条路上不被使用——配了"QPS 预热 + 集群全局阈值"等于预热被静默忽略。

## 5.6 RateLimiterController（匀速排队）

```java
public class RateLimiterController implements TrafficShapingController {

    private final int maxQueueingTimeMs;  // 最大排队时间
    private final double count;           // 每秒允许通过的数量
    
    // 基于漏桶算法
    private final AtomicLong latestPassedTime = new AtomicLong(-1);

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        // 计算每个请求之间的最小间隔
        long costTime = Math.round(1.0 * acquireCount / count * 1000);
        
        long expectedTime = costTime + latestPassedTime.get();
        long currentTime = TimeUtil.currentTimeMillis();

        if (expectedTime <= currentTime) {
            // 预期时间已过，可以立即通过
            latestPassedTime.set(currentTime);
            return true;
        } else {
            // 需要排队等待
            long waitTime = costTime + latestPassedTime.get() - currentTime;
            if (waitTime > maxQueueingTimeMs) {
                return false;  // 超过最大排队时间，拒绝
            }
            // 通过 CAS 更新预期通过时间
            long oldTime = latestPassedTime.addAndGet(costTime);
            waitTime = oldTime - currentTime;
            if (waitTime > maxQueueingTimeMs) {
                latestPassedTime.addAndGet(-costTime);  // 回退
                return false;
            }
            // 睡眠等待
            if (waitTime > 0) {
                Thread.sleep(waitTime);
            }
            return true;
        }
    }
}
```

## 5.7 "排队"两个字的真相：一条预约时间线

真实实现与骨架基本一致（`latestPassedTime = new AtomicLong(-1)`、`costTime = Math.round(1000.0 * acquireCount / count)`、CAS 追加后二次校验超时就 `addAndGet(-costTime)` 回退并拒绝、最后 `sleep(waitTime)`），需要把三件事说穿：

**① 没有队列数据结构。** 全部状态是一个 `AtomicLong`——它记录"下一个请求应该被放行的时刻"。并发请求靠 CAS 把自己预约的时间往后推，**排的是"时间"而不是"线程"**：谁拿到靠前的预约时刻谁先睡醒，这带来近似 FIFO 的效果但没有公平性保证（CAS 重试顺序、系统调度、GC 停顿都能改变实际通过顺序），也没有任何"队列长度"可观测——你能看到的只有 `passQps` 平稳和被拒的 `FlowException` 计数，看不到"现在有几请求在排队"。

**② 等待是 `Thread.sleep`，成本记在容器线程账上。** 这是"匀速排队"最贵的隐含项：Tomcat/Undertow 的工作线程在 slot 链里睡着，`maxQueueingTimeMs=500` 意味着每个请求最多多占 500ms 线程时间。换算式（务必在评审时算一遍）：**排队所需并发线程 ≈ 目标 QPS × (平均业务耗时 + 最大排队时长)/1000**。想把突发削到 1000 QPS 匀速、排队上限 1s，就需要 ~1000 个线程同时挂着——超过容器 `maxThreads` 时，排队还没开始就已经在连接队列里失败了，"削峰填谷"变成"自建雪崩"。正确姿势：`maxQueueingTimeMs` 取得**远小于**上游超时（Feign 的 readTimeout、网关的 responseTimeout），并把排队只用于**低频但必须成功**的写路径（MQ 消费入口、回调），而不是高并发读接口——后者该用 MQ 自己排队（`4_flow_control.md` §4.5 的"削峰的正确层次"）。

**③ 排队时间不计入统计口径。** 骨架里的 sleep 发生在 `FlowSlot.entry` 内部、`StatisticSlot` 已经记过 pass 之后（链序，`2_architecture.md` §3.3），也就是说**通过 QPS 曲线会看到请求在窗口里正常计数，而 RT 曲线变长**；另外 `latestPassedTime` 初值 -1 与 `expectedTime <= currentTime` 分支的处理，使系统空闲后的第一个请求不会被排队（直接放行并把时间线推到 now），**但空闲期间累积的"未发出预约"不会被补偿**——漏桶没有"允许攒额度"的概念，长时间静默后的第一波仍然按 1/count 秒的节奏过（这一点与 `WarmUpController` 的满桶加速恰好相反，两种模式选错方向的现象就来自这里）。

**④ grade 前提与集群前提**：只在 `FLOW_GRADE_QPS` 下被装配（§7.1）；`clusterMode=true` 时全局阈值不支持排队（`4_flow_control.md` §6.2）。

# 6. FlowRuleManager — 规则管理

```java
public class FlowRuleManager {

    // 规则以 Map 形式存储，Key = ResourceName
    private static volatile Map<String, List<FlowRule>> flowRules
        = new HashMap<>();

    public static void loadRules(List<FlowRule> rules) {
        Map<String, List<FlowRule>> newRuleMap = new HashMap<>();
        for (FlowRule rule : rules) {
            // 按资源名分组
            newRuleMap.computeIfAbsent(rule.getResource(), 
                k -> new ArrayList<>()).add(rule);
        }
        flowRules = newRuleMap;
    }

    public static List<FlowRule> getRules(String resource) {
        return flowRules.getOrDefault(resource, Collections.emptyList());
    }
}
```

## 6.1 【纠偏】loadRules 走的是 Property 层，不直接改 Map

骨架把 `loadRules` 写成"直接重建 Map"，这与真实实现差了一层关键结构（还原，简化）：

```java
// 还原
private static volatile List<FlowRule> rules = new ArrayList<>();
private static final ReadWriteLock READ_WRITE_LOCK = new ReentrantReadWriteLock();

private static final SentinelProperty<List<FlowRule>> currentProperty = new DynamicSentinelProperty<>();

static {
    currentProperty.addListener(new FlowPropertyListener());   // 内置监听器负责建表
}

public static void loadRules(List<FlowRule> rules) {
    currentProperty.updateValue(rules);       // ① 只是发布事件
}

static void onRuleChange(List<FlowRule> rulesCollection) {   // ② 由 listener 调
    READ_WRITE_LOCK.writeLock().lock();
    try {
        Map<String, List<FlowRule>> newRuleMap = new HashMap<>();
        for (FlowRule rule : rulesCollection) {
            if (rule == null || rule.getLimitApp() == null) { continue; }
            if (!isValidRule(rule)) { LogUtil.warn("Ignoring invalid flow rule: {}", rule); continue; }
            FlowRule copy = new FlowRule(rule);
            copy.setRater(generateRater(copy));            // ③ 关键：加载期编译 rater
            // ④ 同资源同 limitApp 收敛（见 §6.2）
            newRuleMap.computeIfAbsent(copy.getResource(), k -> new ArrayList<>()).add(copy);
        }
        flowRules = newRuleMap;                            // ⑤ volatile 整体替换
        rules = convertedList;
    } finally {
        READ_WRITE_LOCK.writeLock().unlock();
    }
}
```

这一层的差别不是风格，是 `14_dynamic_rules.md` 整篇接线方式的原因：**代码里 `loadRules` 与数据源 `register2Property` 是同一条流水线的两个入口**（都落到 `currentProperty`），所以两者并存时后到的整表覆盖先到的（`14_dynamic_rules.md` §2.1 的"三股势力汇合点"），而不是"数据源优先"或有合并逻辑。`FlowPropertyListener` 的两个回调（`configLoad`/`configUpdate`）都只调 `onRuleChange`——这也是自定义观测 listener 只配打日志、不碰 Manager 的依据。

**取规则 API 的名字纠偏**：给 Checker 用的是 `getRulesOfResource(String resource)`（返回该资源的 `List<FlowRule>`，读锁保护），`getRules()` 无参返回全量 `List<FlowRule>` 供控制台/导出。骨架写的 `getRules(resource)` 在源码中不存在（照抄会编译不过）。

## 6.2 加载期做的四件"隐形的事"

| 动作 | 代码位置 | 配置侧后果 |
|------|---------|-----------|
| 合法性校验 | `isValidRule`：count ≥ 0、grade 合法、threshold 与 clusterConfig 一致等 | **非法条目被丢弃并打 warn**，不会让整批失败（`3_quick_start.md` §6.2 看日志的第三类形态） |
| rater 编译 | `copy.setRater(generateRater(copy))` | `controlBehavior` 改一次就要整条规则重建对象；**运行期改 FlowRule 字段无效**（rater 已按旧值生成）——这就是"必须重新 load 一份完整列表"的原因 |
| 规则拷贝 | `new FlowRule(rule)`（还原） | Manager 内部持有副本，外部改传进来的对象不影响生效规则 |
| 同资源重复收敛 | limitApp 相同的两条只留更严的一条（1.8.x 行为，具体策略以所用版本为准） | **"第二条规则莫名不生效"的根因**：同资源 + 同 limitApp + 同 grade 组合的多条规则不会并联，需要区分就改 limitApp 或拆资源 |

集群相关还有第五件：`clusterMode=true` 的规则在加载时会把 `flowId` 注册进 `ClusterStateManager`/server 侧配置同步链路（§8），Dashboard 侧的 flowId 必须与 client 一致，否则出现"有 flowId 无配置"的拒绝/退化（`9_dashboard.md` §6.2）。

# 7. TrafficShaping 选择策略

```java
// 根据 controlBehavior 选择对应的 Controller
private static TrafficShapingController generateRater(FlowRule rule) {
    switch (rule.getControlBehavior()) {
        case RuleConstant.CONTROL_BEHAVIOR_WARM_UP:
            return new WarmUpController(rule.getCount(), 
                rule.getWarmUpPeriodSec(), ColdFactorProperty.coldFactor);
        
        case RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER:
            return new RateLimiterController(rule.getMaxQueueingTimeMs(),
                rule.getCount());
        
        case RuleConstant.CONTROL_BEHAVIOR_DEFAULT:
        default:
            return new DefaultController(rule.getCount(), rule.getGrade());
    }
}
```

## 7.1 【勘误】外层还有一个 grade 判断，且漏了第四种组合

真实结构（还原）是"先判 grade，再在 QPS 分支里 switch behavior"：

```java
// 还原
private static TrafficShapingController generateRater(FlowRule rule) {
    if (rule.getGrade() == RuleConstant.FLOW_GRADE_QPS) {
        switch (rule.getControlBehavior()) {
            case RuleConstant.CONTROL_BEHAVIOR_WARM_UP:
                return new WarmUpController(rule.getCount(), rule.getWarmUpPeriodSec(),
                    ColdFactorProperty.coldFactor);
            case RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER:
                return new RateLimiterController(rule.getMaxQueueingTimeMs(), rule.getCount());
            case RuleConstant.CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER:
                return new WarmUpRateLimiterController(rule.getCount(), rule.getWarmUpPeriodSec(),
                    rule.getMaxQueueingTimeMs(), ColdFactorProperty.coldFactor);
            default:
                break;
        }
    }
    return new DefaultController(rule.getCount(), rule.getGrade());
}
```

两个后果要记：**① 线程数模式（grade=1）下 `controlBehavior` 被完全忽略**——控制台"线程数 + 排队等待"这种组合能保存、不报错，运行期就是快速失败，`4_flow_control.md` §4.6 那张"效果 × 阈值类型"有效性矩阵的代码出处在此；**② 还有第四种 `CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER`（预热 + 匀速排队）**，`WarmUpRateLimiterController extends RateLimiterController`：在漏桶的 costTime 上叠一个"冷启动期把 costTime 放大 coldFactor 倍"的系数，实现"慢速放量 + 平滑通过"，适合重启后依赖冷缓存的重写接口——它同时吃下 §5.5④ 与 §5.7② 两条代价（既有 sleep 占用线程，又有爬升期低吞吐）。原稿 switch 结构看着只有三种，会让"配了 behavior 4 却按漏桶跑"看起来像 bug。

`ColdFactorProperty.coldFactor`：早期版本硬编码 3，1.8 起从 Config 读取（`csp.sentinel.warmup.cold.factor` 一类键名，默认仍是 3，控制台不提供）；它是**静态全局量**，一改影响进程内所有预热规则——想按资源差异化只能拆成不同资源或多实例部署。

# 8. 集群流控分支（源码位置与 fail-over）

`canPassCheck` 在 `rule.isClusterMode()` 时把判定交给 `ClusterFlowChecker`（`slots.cluster.flow` 包，还原主干）：

```java
// 还原（简化）
public boolean canPass(FlowRule rule, Context context, DefaultNode node,
                       int acquireCount, boolean prioritized) {
    // 自身是 token server 时，直接走 server 侧本地判定
    if (ClusterStateManager.isServer()) {
        return ClusterFlowChecker.passCheck(flowConfig(rule), context, node, acquireCount, prioritized);
    }
    TokenResult result = TokenClient.currentTokenClient().requestToken(rule.getClusterConfig().getFlowId(), acquireCount);
    return checkTokenResult(result, rule, context, node, acquireCount);   // 失败时按配置退化/拒绝
}
```

四条源码级结论：**① 全局窗口在 server 侧**：`ClusterFlowChecker` 用 server 自己的 `Map<Long, ClusterNode>`（按 flowId）做 LeapArray 统计，client 只上报"我要占 N 个令牌"——所以全局阈值的精度取决于 server 的窗口（仍是 1s 粒度，`18_statistics_engine.md` 的结构复用）而非实例数换算；**② `fallbackToLocalWhenFail` 在 `checkTokenResult` 里生效**：连接失败/超时/server 未就绪等状态码会走**本地 rater**（此时 `rule.getRater()` 才重登舞台，用单机阈值），这条 fail-open 分支是集群流控"失联不炸"的保证，代价是短时全局量放大为 `单机阈值 × 实例数`；**③ 判定路径全程是一次 RPC + 结果解析**，`tokenRequestTimeoutMs`（默认很小，几十 ms 量级）到点即退化——**这也是集群模式不支持预热/排队的实现原因**：排队需要阻塞，而阻塞语义在"向远端申请令牌"上无法与 fail-over 时限共存；**④ `ClusterStateManager` 的 client/server 角色是启动期属性**（`csp.sentinel.cluster.server.mode`/应用接入 Dashboard 分配 namespace set），角色不对时 `isClient=false` 且没配 server 地址，集群规则整体退化到本地分支——`9_dashboard.md` §6.2 的"拓扑页看不到 server"对应这里。压测/演练时"集群规则看起来没生效"的排障顺序：Dashboard 集群页 → `curl :8719/cluster/token` 状态 → 机器是否都拿到同一 flowId 配置 → `fallbackToLocalWhenFail` 是否在掩盖连不上的事实。

# 9. 一次"通过"的完整时间线（把本篇串起来）

```text
t0  业务线程 SphU.entry("getUser")
     └─ CtSph: 取 chain（缓存命中）→ 建 Context/Entry → chain.entry(count=1, prioritized=false, args)
t1  ProcessorSlot 链依次 fire
     ├─ NodeSlot：定位 ClusterNode + DefaultNode（context 维度）
     ├─ StatisticsSlot：pass++（先记后判，失败再冲销）；curThreadNum++
     ├─ SystemSlot：入口且非 OUT 时按全局 load/RT/线程/QPS 判定
     ├─ FlowSlot → FlowRuleChecker：
     │    getRulesOfResource("getUser") → [rule_default_qps100]
     │    limitApp=default & strategy=direct → refNode = node（DefaultNode/ClusterNode 口径）
     │    rater = DefaultController：avgUsedTokens = passQps(当前窗口)
     │    passQps(98) + 1 > 100? 否 → return true
     ├─ ParamFlowSlot / DegradeSlot：各自判定（本篇不展开）
     └─ 链尾：Entry 就绪
t2  业务执行
t3  Entry.exit → 链 exit fire：StatisticsSlot 结算 RT、pass 数落到已完成桶、线程数 -1
t4  窗口滚动：passQps 由 LeapArray 的 completed buckets 算出 → 下一个请求看到新值
```

三个"时序坑"就在 t1→t3 的距离里：**判定用的是已完成窗口的数字**（当前桶未结算，所以瞬时并发突刺会短暂超过阈值，`4_flow_control.md` §1.2 的"QPS 判定精度"由此而来）；**线程数模式靠 exit 归还**（业务线程泄漏/漏 exit 会让并发额度永久占着，`2_architecture.md` §5.2 的纪律）；**排队/优先级模式在 t1 内 sleep**，把 t2 前的耗时直接拉长（§5.7②）。

# 10. 源码视角的常见坑（速查）

| 现象 | 源码归因 | 定位动作 |
|------|---------|---------|
| 规则加载了但完全不拦 | `limitApp == null` / `rater == null` / refNode==null（§3.2 三处 fail-open） | 打印 `FlowRuleManager.getRulesOfResource(name)` 看内容，别只看 `getRules().size()` |
| 关联/链路模式不生效 | ref 资源名不匹配（链路要 `context.getName().equals(refResource)`，§4.6） | 从簇点链路复制 context 名与资源名，逐字符对 |
| 线程数规则配了排队没效果 | `generateRater` 只在 QPS 分支看 behavior（§7.1） | 改回快速失败或改用 QPS |
| 预热没感觉 | 读的是 `previousPassQps`；period 短于实际爬升需要（§5.5①③） | 压测时用"从 0 拉起"的流量曲线验证 |
| 排队导致 RT 暴涨/线程耗尽 | `Thread.sleep` 在 slot 内（§5.7②） | 核对 `QPS × 排队上限` 与 `maxThreads` |
| 同资源第二条同 limitApp 规则没用 | 加载期重复收敛（§6.2） | 合并成一条或换 limitApp |
| 改了 FlowRule 对象字段没效果 | rater 在 load 期编译、Manager 持副本（§6.2） | 重新 `loadRules` 完整列表 |
| 集群规则退化到单机 | token server 不可达 + `fallbackToLocalWhenFail`（§8②） | 看 Dashboard 集群页与 `/cluster/token` |

# 14. 相关阅读

链构建与 Slot 顺序在 `15_slot_chain_source.md`；被读的统计结构（LeapArray/MetricBucket/节点树）在 `18_statistics_engine.md`；熔断这条依赖 exit 上报的路在 `17_circuit_breaker_source.md`；SPI 装配（`FlowSlot` 如何进链）在 `19_spi_extension.md`；规则语义与配置实践在 `4_flow_control.md`；数据源与 `loadRules` 覆盖竞态在 `14_dynamic_rules.md`；集群部署形态在 `9_dashboard.md` §6.2。
