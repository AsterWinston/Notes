# 1. DegradeSlot — 入口

```java
public class DegradeSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized,
                      Object... args) throws Throwable {
        // 执行熔断检查
        DegradeRuleManager.checkDegrade(resourceWrapper, context, node, count);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper,
                     int count, Object... args) {
        // 资源调用完成后，上报 RT 和异常给 CircuitBreaker
        Entry curEntry = context.getCurEntry();
        if (curEntry.getBlockError() != null) {
            fireExit(context, resourceWrapper, count, args);
            return;
        }
        // 通过 StatisticSlot 的 exit 间接上报
        fireExit(context, resourceWrapper, count, args);
    }
}
```

## 1.1 真实骨架：exit 才是熔断了一半的主战场

`entry` 侧与骨架一致（还原：真实代码是 `DegradeRuleManager.checkDegrade(resourceWrapper, context, node, count)`，抛 `DegradeException`）。`exit` 侧骨架保留了正确的一半（被拦即早退），但漏了中段与末段（还原，简化）：

```java
// 还原（简化）
@Override
public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
    Entry curEntry = context.getCurEntry();
    if (curEntry.getBlockError() != null) {          // 被任何拦截者拒掉的 Entry
        fireExit(context, resourceWrapper, count, args);
        return;                                       // 不参与熔断统计
    }
    List<CircuitBreaker> breakers = DegradeRuleManager.getCircuitBreakers(resourceWrapper.getName());
    if (breakers == null || breakers.isEmpty()) {     // 无规则：零成本快路径
        fireExit(context, resourceWrapper, count, args);
        return;
    }
    // 结算：RT = exit 时刻 - entry 时刻；异常取 curEntry.getError()
    long r = curEntry.getCompleteTimestamp() - curEntry.getCreateTimestamp();   // 真实取 createTs
    for (CircuitBreaker breaker : breakers) {
        breaker.apply(context, resourceWrapper, curEntry.getOrigin(), curEntry.getNodeType(), r,
            curEntry.getError());
    }
    fireExit(context, resourceWrapper, count, args);
}
```

骨架注释里那句"**通过 StatisticSlot 的 exit 间接上报**"要勘误：**熔统计不经过 StatisticSlot**，DegradeSlot 在自己的 exit 里直接把 RT 与 error 喂给 breaker，用的是 breaker 自带的统计表（§7.1）。这一句误读的代价很实在：以为"熔断数字=主统计数字"的人，会在窗口口径、慢调用计数、被拦请求是否计入这些点上做出错误推理——`5_circuit_breaker.md` 里"两套账"的提醒，代码出处就在这一段。

## 1.2 链上的位置与"判定发生在哪两个时刻"

`DegradeSlot` 在链上排在 `FlowSlot`/`ParamFlowSlot` 之后（`2_architecture.md` §3.3，SPI order 决定），含义是：**被流控拒掉的请求根本走不到熔断器**，也就不会被计入异常统计（`blockError != null` 的早退与之呼应）；而熔断的 entry 判定又发生在 `StatisticSlot` 之后，所以被熔断的请求**已经在主统计里记过一次 block**（这就是 `9_dashboard.md` §4.4"拒绝 QPS 单独成列"与"熔断后曲线骤降"的成因——被短路的请求不产生 RT，平均 RT 反而在熔断恢复后看起来变好了）。

# 2. CircuitBreaker — 熔断器

```java
public abstract class CircuitBreaker {

    protected State state = State.CLOSED;           // 当前状态
    protected final String resource;                 // 资源名
    protected volatile long nextRetryTimestamp;      // 下次可重试时间
    protected final int recoveryTimeoutMs;           // 恢复超时(ms)

    public enum State {
        CLOSED,     // 正常：统计指标，正常通过
        OPEN,       // 熔断：直接拒绝
        HALF_OPEN   // 半开：允许探测请求
    }

    // 判断是否允许通过
    public boolean tryPass(Context context) {
        if (currentState() == State.CLOSED) {
            return true;
        }
        if (currentState() == State.OPEN) {
            // 检查是否可以进入 HALF_OPEN
            if (retryTimeoutArrived() && fromOpenToHalfOpen(context)) {
                return true;  // 允许一个探测请求
            }
            return false;     // 拒绝
        }
        // HALF_OPEN: 已经放行了一个探测请求
        return false;
    }

    // 从 OPEN 转为 HALF_OPEN
    protected boolean fromOpenToHalfOpen(Context context) {
        // CAS 保证只有一个请求进入半开状态
        if (state.compareAndSet(State.OPEN, State.HALF_OPEN)) {
            return true;
        }
        return false;
    }

    protected boolean retryTimeoutArrived() {
        return TimeUtil.currentTimeMillis() >= nextRetryTimestamp;
    }
}
```

## 2.1 熔断的两半：entry 期只读状态，exit 期才写状态

这是本篇最该先立起来的结构性认知，也是 `16_flow_control_source.md` §2.1 所说"两类规则的根本形态差"的展开。基类把职责切成三段（真实类图：抽象基类 + `ResponseTimeCircuitBreaker` / `ExceptionCircuitBreaker` 两个子类，1.7 起重构出 `AbstractCircuitBreaker` 承载 `apply()`/`boundNode()` 的公共逻辑，子类名与方法族在不同版本略有出入，本篇按语义读，以所用版本为准）：

| 阶段 | 方法 | 做什么 | 不做什么 |
|------|------|--------|---------|
| entry | `tryPass(context)` | **只读** `state` + `nextRetryTimestamp` 做短路判定 | 不看任何窗口数字、不产生任何计数 |
| exit | `onRequestComplete(context)` | 读本次 RT/error → 更新自己的统计 → 判定是否 CLOSED→OPEN | 不影响本次请求能否通过（本次早已放行） |
| 探测 | `onEntry(context)`（RT 类重写） | HALF_OPEN 下记录探测起点，供 exit 算探测 RT | — |

由此得出三条只有读代码才踏实的行为结论：**① 熔断触发对"当前这一笔"无效**——把比例推过阈值的那个请求正常执行完了才改状态，OPEN 从下一个 entry 生效；所以熔断的"反应时间"至少包含一个请求的 RT + 窗口延迟，`5_circuit_breaker.md` 说"熔断是止血不是急救"的依据在此。**② OPEN 期间的拒绝成本近乎零**：`tryPass` 就是两次 volatile 读 + 时间比较，没有锁、没有窗口遍历——这正是下游雪崩时熔断器能保住本方线程的原因（与 `16_flow_control_source.md` §5.7 排队模式的 sleep 相对照，两种"保护"的线程代价完全相反）。**③ `count`/阈值判定不发生在 entry**，所以"熔断中收到 RT 曲线空白"不是 bug：短路请求不进统计（§1.2）。

## 2.2 状态字段的并发真相：单探测靠的是"近似 CAS"

骨架把 `state` 写成带 `compareAndSet` 的原子引用，而真实实现里这个字段是 `volatile State`，转换走的是基类的一个受锁/原子封装（1.8 用 `AtomicReference<State>` 语义的 `unSafeCompareAndSet`/`compareAndSet` 辅助方法，早期版本源码注释自陈"not thread-safe"，靠的是"迁移条件收敛到少数几个调用点 + 状态本身单调"来保证效果；具体字段形态随版本变化，以所用版本为准）。**读代码时要抓住的不是 CAS 是否严格，而是三件事**：

- **HALF_OPEN 只放一个探测请求**：`tryPass` 在 `state == HALF_OPEN` 时**返回 false**（骨架末行正确），只有把 OPEN 翻成 HALF_OPEN 的那一个线程拿到 true——所以"半开"期间并发打过来时，其余请求全部被拒，探测样本量恒等于 1；这也意味着**探测请求本身可能撞上刚恢复但仍在打冷的下游**，一次失败就再等一个 `timeWindow`（§6.1 讨论它是否合理）。
- **OPEN→HALF_OPEN 的时机是"被动"的**：没有定时器，`retryTimeoutArrived()` 只在有人 entry 时才检查；没有流量的资源即使 timeWindow 早过了也不会"变成半开"，簇点链路里的状态会长时间停在 OPEN 直到下一个请求进来——排障时看到"熔断中但没有请求"不要以为是定时器坏了。
- **CLOSED→OPEN 之后 `nextRetryTimestamp` 立即生效**，OPEN 期间若又观察到新的失败（RT/异常）会怎样？**不会重新计时**：CLOSED→OPEN 的 CAS 在 OPEN 状态失败即返回，`timeWindow` 从触发时刻起算，中途的额外失败不延长熔断（想要"持续恶化就延长"的退避语义，Sentinel 没有，Resilience4j 有指数退避，§8.2）。

## 2.3 字段的真实对应关系（配置 ↔ 代码）

| DegradeRule 字段 | 基类/子类成员 | 语义 |
|------------------|--------------|------|
| `timeWindow`（秒） | `recoveryTimeoutMs = timeWindow * 1000` | OPEN 的时长，`nextRetryTimestamp = 触发时刻 + recoveryTimeoutMs` |
| `grade` | 决定实例化哪个子类 + `strategy` | 0=慢调用比例、1=异常比例、2=异常数 |
| `count` | `maxAllowedRt` / `maxAllowedErrorRatio` / 异常数阈值 | **一个字段三种含义**（`5_circuit_breaker.md` §10 口径表） |
| `slowRatioThreshold` | 子类比例阈值 | 仅 RT 模式用 |
| `minRequestAmount` | `threshold` | 请求数不足时**完全不判定**（连 OPEN 也不会维持判定） |
| `statIntervalMs` | 统计窗口长度 | 决定"这套账本"覆盖多久（§7.1） |

`DegradeException` 的抛出点在 `checkDegrade`（`throw new DegradeException(breaker.getRule().getLimitApp(), breaker.getRule())`），异常里带着触发的那条 rule——日志与 `/api/degrade/rules` 对账时 `ex.getRule()` 能拿到阈值现场（`21_logging_monitoring.md`）。

# 3. ResponseTimeCircuitBreaker（慢调用熔断）

```java
public class ResponseTimeCircuitBreaker extends CircuitBreaker {

    private long maxAllowedRt;             // 最大允许 RT(ms)
    private double maxSlowRequestRatio;    // 慢调用比例阈值
    private int minRequestAmount;          // 最小请求数

    public ResponseTimeCircuitBreaker(DegradeRule rule) {
        this(rule.getCount(), rule.getTimeWindow() * 1000,
             rule.getMinRequestAmount(), rule.getSlowRatioThreshold());
    }

    @Override
    public void onRequestComplete(Context context) {
        // 1. 获取滑动窗口统计
        long slowCount = slidingCounter.getSlowCount();   // 慢调用数
        long totalCount = slidingCounter.getTotalCount(); // 总调用数

        if (totalCount < minRequestAmount) {
            return;  // 请求数不足，不判断
        }

        // 2. 计算慢调用比例
        double slowRatio = (double) slowCount / totalCount;

        // 3. 判断是否触发熔断
        if (slowRatio > maxSlowRequestRatio) {
            // CAS 变更状态：CLOSED → OPEN
            if (state.compareAndSet(State.CLOSED, State.OPEN)) {
                nextRetryTimestamp = TimeUtil.currentTimeMillis()
                    + recoveryTimeoutMs;
                log.warn("[Degrade] resource={} slowRatio={} > threshold={}, OPEN",
                    resource, slowRatio, maxSlowRequestRatio);
            }
        }
    }
}
```

## 3.1 两套统计表：比例判定 + maxRt 快速回弹

骨架只画了"一张比例窗口"，真实 RT 熔断器多维护一套**每窗口的最大 RT**统计（还原，简化）：

```java
// 还原：RT 熔断的两块账
private final LeapArray<SlowRequestCounter> slidingWindow;   // 总数 + 慢调用数（算比例）
private final LeapArray<MaxRt> maxRtStatistic;              // 窗口内最大 RT（OPEN/探测期的快速判定）
```

`maxRtStatistic` 的用处是这段逻辑（还原语义）：**当状态已经是 OPEN 时，若有请求（例如探测请求）的 RT 再次超过 `maxAllowedRt`，不等比例算出来，立刻把状态打回 OPEN 并重新计时**。这就是为什么 RT 模式的 HALF_OPEN 探测"慢即回退"的响应是单请求级的，而 CLOSED 状态的进入熔断才需要攒够 `minRequestAmount` 算比例。工程含义两条：**探测失败不需要"凑样本"，一次慢就够**（比异常比例模式反应快得多）；同时**OPEN 期间偶尔漏进去的探测请求，其 RT 会立刻续期熔断**，运维上看到"熔断时长比 timeWindow 长"多半是探测撞上慢响应续的命，不是 timeWindow 配错。

## 3.2 【纠偏】构造签名与阈值比较的边界

**①** 真实构造是 `public ResponseTimeCircuitBreaker(DegradeRule rule)`（内部自己拆 `count`/`timeWindow*1000`/`slowRatioThreshold`/`minRequestAmount`/`statIntervalMs`），不是骨架那个四参数形态——骨架的四参版在 1.8 已不存在（早期版本有过，注意资料代际）。**② 比较方向**：真实判定是 `slowRatio >= slowRatioThreshold` 与 `RT > maxAllowedRt`（`count` 是"严格大于才算慢"），骨架写 `>` 阈值；边界值上一笔 200ms 的调用在 `count=200` 下**不算慢调用**。这个细节决定"慢调用定义线"贴着 P99 配时的抖动方向，值得记住。**③ RT 模式下 `statIntervalMs`/窗口长度影响的是比例分母的可信度**：窗口太短（默认 1s）时 `minRequestAmount=5` 很容易被一次瞬时抖动满足，比例在样本少时方差极大（`5_circuit_breaker.md` §5.1 的"小样本陷阱"）；反过来窗口拉长（如 10s）会让"熔断反应"变慢——**RT 熔断的窗口是灵敏度/误杀率的旋钮**，不是越大越稳。

# 4. ExceptionCircuitBreaker（异常熔断）

```java
public class ExceptionCircuitBreaker extends CircuitBreaker {

    private double maxExceptionRatio;     // 异常比例阈值
    private int minRequestAmount;

    @Override
    public void onRequestComplete(Context context) {
        Entry entry = context.getCurEntry();
        if (entry == null) return;

        // 只统计业务异常，BlockException 不计入
        Throwable error = entry.getError();
        if (error == null || error instanceof BlockException) {
            return;  // 非异常或流控异常不统计
        }

        // 统计异常数
        slidingCounter.addException();

        long totalCount = slidingCounter.getTotalCount();
        long exceptionCount = slidingCounter.getExceptionCount();

        if (totalCount < minRequestAmount) {
            return;
        }

        double exceptionRatio = (double) exceptionCount / totalCount;

        if (exceptionRatio > maxExceptionRatio) {
            if (state.compareAndSet(State.CLOSED, State.OPEN)) {
                nextRetryTimestamp = TimeUtil.currentTimeMillis()
                    + recoveryTimeoutMs;
                log.warn("[Degrade] resource={} exceptionRatio={} > threshold={}, OPEN",
                    resource, exceptionRatio, maxExceptionRatio);
            }
        }
    }
}
```

## 4.1 BlockException 不计入：一条被反复验证的纪律

这段"非异常或流控异常不统计"是熔断生态里最容易被自己踩坏的地方，代码上有**两道**防线（链序上熔断在流控之后是第一道，见 `2_architecture.md` §3.3）：**被拦请求压根不进 DegradeSlot.exit 的统计分支**（`blockError != null` 早退）+ **进来了也认得 `BlockException` 家族并跳过**。为什么这么重要：熔断一旦把"自己拒绝的请求"当失败，就会形成正反馈自杀（限流→异常→熔断→更多拒绝→更久熔断），这条短路是它的保险丝。对应到实践：`@SentinelResource` 的 blockHandler **不要 throw 一个非 Block 的异常**（把 Block 转成业务 RuntimeException 会绕过这道防线，熔断统计被污染，`5_circuit_breaker.md` §10.2 的"降级路径里别再抛异常"）。骨架另一处要留意：`entry.getError() == null` 时**直接 return**——真实实现不是这样，异常为 null 的请求仍要计入**总请求数（分母）**，骨架的写法会让"全部成功"的资源永远算不出比例（分母恒 0）。真实结构是：先 `addPass/total` 再按 error 决定是否 `addException`，然后才做比例判定（还原语义）。

## 4.2 异常比例与异常数共用一个类，差别只在判定式

`ExceptionCircuitBreaker` 靠 `strategy`（`ERROR_RATIO` / `ERROR_COUNT`）分派两条判定式，异常数模式**不看分母**：

```java
// 还原（简化）
if (strategy == CircuitBreakerStrategy.ERROR_COUNT) {
    if (exceptionCount >= rule.getCount()) { trip(context); }   // 窗口内异常数达阈值
} else {
    if (totalCount >= minRequestAmount && exceptionRatio >= rule.getCount()) { trip(context); }
}
```

三个由此而来的口径：**① 异常数模式的窗口就是 `statIntervalMs`（默认 1000ms）**，所以"`count=5` 熔断"的真实含义是"1 秒内 5 个异常"，不是旧文档流传的"每分钟 5 个"（`5_circuit_breaker.md` §10.1 的勘误同一条）；想用 1 分钟口径就把 `statIntervalMs` 设为 60000，代价是误判恢复也要等一个分钟级窗口（`minRequestAmount` 在异常数模式下不参与判定，这是它的隐性优势：低频接口也能熔断）。**② 异常数模式没有"最小请求数"保护**，一个本来就持续报错的接口会稳定地"熔断→探测→失败→熔断"循环，曲线表现为规律的锯齿——这是正常形态，不是抖动。**③ 异常判定用的是 `Entry.exit(..., Throwable)` 传进来的 error**：只有 `Tracer.trace(err)` / `SphU.entry` 的 try-with-resources 配 `entry.setError` 能填上它，**吞掉异常不 trace 的代码会让熔断器永远看不到失败**（`5_circuit_breaker.md` §10.3 的"异常必须上报"纪律，代码依据在此；Web 集成里 `SentinelWebInterceptor.afterCompletion` 会代填，注解切面也代填，手写 API 埋点最容易漏）。

# 5. DegradeRuleManager

```java
public class DegradeRuleManager {

    // 规则 Map: ResourceName → List<CircuitBreaker>
    private static volatile Map<String, List<CircuitBreaker>> circuitBreakers
        = new ConcurrentHashMap<>();

    public static void checkDegrade(ResourceWrapper resource, Context context,
                                     DefaultNode node, int count)
                                     throws BlockException {
        // 获取该资源的熔断器列表
        List<CircuitBreaker> breakers = circuitBreakers.get(resource.getName());
        if (breakers == null || breakers.isEmpty()) {
            return;  // 无规则
        }

        for (CircuitBreaker breaker : breakers) {
            if (!breaker.tryPass(context)) {
                // 熔断器中，直接抛出 DegradeException
                throw new DegradeException(breaker.getRule().getLimitApp(),
                    breaker.getRule());
            }
        }
    }

    // 加载规则：根据 DegradeRule 创建 CircuitBreaker
    public static void loadRules(List<DegradeRule> rules) {
        Map<String, List<CircuitBreaker>> newBreakers = new ConcurrentHashMap<>();
        for (DegradeRule rule : rules) {
            CircuitBreaker breaker;
            switch (rule.getGrade()) {
                case RuleConstant.DEGRADE_GRADE_RT:
                    breaker = new ResponseTimeCircuitBreaker(rule);
                    break;
                case RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO:
                case RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT:
                    breaker = new ExceptionCircuitBreaker(rule);
                    break;
                default:
                    continue;
            }
            newBreakers.computeIfAbsent(rule.getResource(),
                k -> new ArrayList<>()).add(breaker);
        }
        circuitBreakers = newBreakers;
    }
}
```

## 5.1 骨架对了大半，补三层结构

`checkDegrade` 的形态基本准确（真实取 breaker 走 `getCircuitBreakers(resource)`，且**同资源多 breaker 时逐个 tryPass，任一短路即抛**——这就是"同一资源可以配多条不同策略的熔断规则并联"的实现，最严者胜，`5_circuit_breaker.md` §2 的多规则账）。`loadRules` 需要补三层：**① 数据入口是 Property**（`loadRules` 实为 `currentProperty.updateValue(rules)`，真正建表在 listener 回调 `onRuleChange` 里，与 `16_flow_control_source.md` §6.1 完全同构——`14_dynamic_rules.md` 的"三股势力汇合点"在熔断这边是另一个实例）；**② 建表前逐条 `isValidRule`**（grade 合法、count ≥ 0、RT 模式要求 `count > 0` 且 `slowRatioThreshold ∈ (0,1]`、`statIntervalMs > 0`），非法条目**丢弃 + warn**，不会整批失败；**③ 双表结构**：`ruleMap`（资源→规则列表）与 `circuitBreakers`（资源→breaker 实例列表）同批重建，`getCircuitBreakers` 读后者、控制台 `/rules` 类接口读前者——**两套东西同批生成，是"规则改了、状态没了"的结构性原因**（§5.2）。

## 5.2 【关键】规则重建会清零正在 OPEN 的熔断

`onRuleChange` 每次都 `new ResponseTimeCircuitBreaker(rule)`/`new ExceptionCircuitBreaker(rule)` 并整体替换 Map，**breaker 里的 state、nextRetryTimestamp、统计窗口全在新对象上重新初始化**（CLOSED、窗口空）。后果是一条纯运维性质但极难自查的故障模式：

```text
t0  下游变慢 → RT 熔断 OPEN（timeWindow=30s）
t5  有人改了一条无关资源的规则 / Nacos 配置被全量重推 / Dashboard 保存了一次规则
t5+ 该资源的 breaker 被换成新对象 → state=CLOSED，熔断"凭空解除"
t6  流量打向仍然慢的下游 → 重新攒 minRequestAmount 样本 → 再次 OPEN
```

表现为"熔断时长被反复重置、曲线像在抖动"，或"明明配了熔断却总拦不住"。**判定方法**：对比 `sentinel-record.log` 里同资源的 OPEN 记录间隔与规则推送日志（`14_dynamic_rules.md` §7.1 的观测 listener 正好用来打这条）——OPEN 之后紧跟一次 `configUpdate` 即中招。**治理**：改规则要改整份文件里的目标条目并保持发布频率可控；禁止"为了看效果反复保存同一条规则"；熔断类规则的变更走数据源、走流水线，不要 Dashboard 手点（`9_dashboard.md` §4.4 的"立即推送"就是整表重推）。

## 5.3 无熔断规则的资源零开销，但有"半规则"形态

`circuitBreakers` 里没有该资源时 exit 早退（§1.1 的快路径）。值得注意的反向形态：**规则存在、资源从没有真实调用**（全部被流控拦掉，§4.1）→ 熔断统计恒空 → 熔断永不触发，Dashboard 上"配了熔断但曲线是 0/0"的合法解释之一；同理"资源名对不上"（`3_quick_start.md` §6.2 的头号坑）在熔断这里表现为"breaker 存在但 tryPass 从不被调"，比流控更难发现——流控失效能从拒绝数看出，熔断失效看起来只是"没熔断过"。

# 6. 状态转换详解

```text
                    onRequestComplete()
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       正常统计      超过阈值?        无变化
          │              │
          │         CLOSED→OPEN
          │         (记录 nextRetryTimestamp)
          │
  ┌───────┴───────┐
  │               │
  ▼               ▼
 下次请求          下次请求
 tryPass()        tryPass()
   │               │
   ▼               ▼
 recoveryTimeout  recoveryTimeout
 已到达?          未到达?
   │               ▼
   ▼               拒绝
 OPEN→HALF_OPEN
   │
   ▼
 允许探测请求
   │
   ├── 成功 → onRequestComplete()
   │            正常统计 → HALF_OPEN→CLOSED
   │
   └── 失败 → onRequestComplete()
              统计异常 → HALF_OPEN→OPEN
```

## 6.1 图之外必须补的三条边

**① 探测请求自己也进统计**：HALF_OPEN 放行那一个请求后，它的 RT/error 会照常走 `onRequestComplete`——**这就是"成功即 CLOSED"的判定源**（真实实现里 RT 模式成功路径是"探测 RT < maxAllowedRt → HALF_OPEN→CLOSED"，异常模式是"探测无异常 → CLOSED"）。但注意：CLOSED 之后统计窗口并没有被清空（还原：新窗口随时间自然滚动），**刚恢复时旧窗口里的高异常数据仍在分母/分子里**，可能立刻再次把比例推过阈值→秒回 OPEN。这个"恢复即复发"现象在低 QPS 资源上尤其常见，缓解手段是拉长 `statIntervalMs`（让历史数据更快滚出）或提高 `minRequestAmount`（`5_circuit_breaker.md` §9.3 的"反复熔断"配方）。

**② HALF_OPEN→OPEN 有两条来路**：探测失败（正常语义），以及 RT 模式下"探测慢"经 `maxRtStatistic` 走的快速通道（§3.1）——两条都会重新设 `nextRetryTimestamp`，等价于熔断时长被续期。

**③ 状态机没有"手动关闭"入口**，只有 `DegradeRuleManager.loadRules` 隐式重置（§5.2）——所以运维上"想提前恢复"的唯一手段是改规则触发重建。这条既是应急技巧（临时把 `timeWindow` 改成 1 秒推下去，等于强制 CLOSED 并缩短后续熔断），也是误操作高发点（改完忘了改回来，保护就此失效，`24_troubleshooting.md` 的案例族）。

# 7. 滑动窗口统计

```java
public class SimpleCircuitBreakerMetric {
    private final LeapArray<LongAdder> passCounter;       // 通过计数
    private final LeapArray<LongAdder> blockCounter;      // 阻塞计数
    private final LeapArray<LongAdder> exceptionCounter;  // 异常计数
    private final LeapArray<LongAdder> rtCounter;         // RT 累计
    private final LeapArray<LongAdder> slowCounter;       // 慢调用计数

    public SimpleCircuitBreakerMetric(int sampleCount, int intervalInMs) {
        this.passCounter = new LeapArray<>(sampleCount, intervalInMs) {
            @Override public LongAdder newEmptyBucket(long time) {
                return new LongAdder();
            }
        };
        // ... 类似初始化其他计数器
    }
}
```

## 7.1 类名与结构纠偏：这是"第二套账"

真实结构接近这个形态（1.7/1.8 的熔断器内部统计类，命名在不同版本有出入，如 `SimpleCircuitBreakerMetric` / 各 breaker 私有的 `SlowRequestCounter` + `MaxRt`，本篇按语义描述，以所用版本为准），但三点必须钉清：**① 它与主统计体系完全独立**——`ClusterNode`/`DefaultNode` 的 `MetricBucket`（`18_statistics_engine.md`）是"业务视角的账"，熔断器手里这张是"熔断判定的账"，两者窗口长度可以不同（主统计固定 1s×2 桶语义，熔断跟随 `statIntervalMs`），这也是"控制台 QPS 与熔断判定用的总数对不上"的合法解释之一。**② 桶长自适应**：窗口很长时（比如 `statIntervalMs=60000`）不会真开 120 个 500ms 桶，实现按窗口长度放大 bucket 粒度（还原：桶数量级固定在个位数到十几个，避免桶对象与滚动成本爆炸）；含义是**拉长窗口的代价不是内存，而是分辨率变粗**——60s 窗口 / 10 桶=每桶 6s，判定粒度 6 秒级。**③ `rtCounter` 不是"平均 RT"**：熔断侧存的是累计/最大值语义的 RT 计数（RT 模式真正用到的是"是否超阈值"的布尔归类和 maxRt，不做分位数计算），想拿 P99 必须走主统计或 APM（`9_dashboard.md` §4.4 的"P99 要打个折"同源）。

## 7.2 异常数模式为什么"窗口越大越钝"

`statIntervalMs` 同时是**分子分母的观测期**和**滚出周期**：窗口 60s 时，"异常数达阈值"要攒满 60s 的数据（慢），但一旦 OPEN，恢复后旧异常也要 60s 才滚出（快回到 OPEN，§6.1 ① 的复发概率上升）。**换算建议**：低频接口（QPS<1）用较大窗口换取"能触发"，高频接口保持 1s 换取"快速止血"；异常数模式配长窗口时把 `timeWindow` 压短，避免"长窗口触发 + 长窗口恢复"的双重迟钝。

# 8. 与 Hystrix 熔断的对比

| 维度 | Sentinel | Hystrix |
|------|----------|---------|
| 熔断策略 | 慢调用/异常比例/异常数 | 仅异常比例 |
| 半开机制 | CAS 保证单探测 | 单请求探测 |
| 统计窗口 | 滑动窗口（任意 interval） | 固定窗口（10s） |
| 分级熔断 | ✓ 支持多级规则 | ✗ 单一规则 |
| 线程池隔离 | ✗（推荐信号量隔离） | ✓ |

## 8.1 表的逐行校准

**"仅异常比例"** 不准确：Hystrix 是"异常比例 + 最小请求量阈值"（`errorThresholdPercentage` 默认 50%、`requestVolumeThreshold` 默认 20），且**没有慢调用维度**——它把超时算作异常（`execution.isolation.thread.timeoutInMilliseconds` 触发后走 `markTimeout`/异常计数），所以 Sentinel 的"慢调用比例"在 Hystrix 语境里要靠"缩短超时→转成异常"间接表达，这是两者最本质的能力差。**"半开机制"行区分度低**：Hystrix 也靠 `allowSingleTest` 的 AtomicBoolean CAS 保证一次只放一个探测请求，与 Sentinel 的 HALF_OPEN 语义一致；真实差异在探测失败后的处理（Sentinel 直接回 OPEN 并重新计 `timeWindow`，Hystrix 由事件流决定，且 `sleepWindowInMilliseconds` 固定不指数退避）。**"固定窗口 10s"** 应写作"默认 10s、可配但下限 10s，桶数固定 10 个"（`metrics.rollingStats.timeInMilliseconds` ≥ 10000）；Sentinel 侧窗口是 `statIntervalMs` 可任意（默认 1s），灵敏度可高一个数量级。**"分级熔断"** 表述过头：Sentinel 的能力是"同资源多条不同策略规则并联、各自独立状态机、任一 OPEN 即拒"（§5.1），不是按流量/故障等级分层的语义化分级；把它写成"多级规则并联"更准确。**"线程池隔离"** 这行是对的，且要加一句 Sentinel 的等价物：Sentinel 不做资源级隔离，靠 `FLOW_GRADE_THREAD`（并发线程数上限，`16_flow_control_source.md` §5.3 读的是 `curThreadNum`）实现"轻量舱壁"——没有独立线程池，也就没有 Hystrix 的线程池调参税与跨线程上下文丢失问题，这是当代主流选择它的主要理由。

## 8.2 现役对照：与 Resilience4j 的源码级差异

| 维度 | Sentinel（本篇） | Resilience4j |
|------|-----------------|--------------|
| 状态字段 | `volatile State` + 受保护的比较置位 | `AtomicReference<State>`，另有 `DISABLED`/`FORCED_OPEN`/`METRICS_ONLY` |
| 半开放行量 | 恒 1 个探测请求 | `permittedNumberOfCallsInHalfOpenState` 可配（默认 1，可调多个） |
| 恢复等待 | 固定 `timeWindow` | `waitIntervalFunctionInOpenState` 可指数退避 |
| 窗口模型 | 时间窗口（`statIntervalMs`，桶长自适应） | 计数窗口或时间窗口二选一（`slidingWindowType`） |
| 降级返回值 | 熔断只负责"拦"，兜底由 fallback/blockHandler 承担 | 同一装饰器链里 fallback 与熔断绑在一起 |
| 失败口径 | `BlockException` 不计数、被拦不进统计（§4.1） | 由 `recordExceptions`/`ignoreExceptions` 白名单精确控制 |
| 源码复杂度 | 单文件状态机 + 两张表 | 状态机 + EventPublisher + Metrics 快照（可观测性更强） |

R4 的两个能力恰好是 Sentinel 的已知短板：**半开多探测**（单探测撞上冷下游就再等一个窗口的运气问题，§6.1）与**指数退避**（下游持续故障时不必反复探测）。反过来说 Sentinel 的优势在这里也具体：`ignoreExceptions` 那类精确口径要在 Sentinel 里做，只能在业务侧把不该计的异常包成 Block 语义或干脆换资源名（没有配置位）；跨笔记对照见 Feign 系列 `14_circuit-breaker.md`（Resilience4j 路线的接线）与 `16_fallback.md`（"降级不计失败"语义的有无）。

# 9. 源码视角的常见坑（速查）

| 现象 | 源码归因 | 定位动作 |
|------|---------|---------|
| 熔断时长比 timeWindow 长 | 探测请求 RT 再次超阈值 → 快速回 OPEN 续期（§3.1） | 看 record.log 的连续 OPEN 记录 |
| 熔断反复触发-恢复（锯齿） | 恢复后旧窗口异常数据仍在分母/分子（§6.1①） | 拉长 `statIntervalMs` 或提高 `minRequestAmount` |
| 配了熔断却从不触发 | 请求全被流控先拦（§4.1/§5.3）/ 异常没 trace（§4.2③）/ 资源名不符（§5.3） | 确认 `exceed` 的入口：先看拒绝数是否已在流控列 |
| 明明 OPEN 却很快放行 | 规则被重推 → breaker 重建、状态清零（§5.2） | 对比规则推送日志与 OPEN 日志时间戳 |
| 低频接口熔断永不生效 | 异常比例模式受 `minRequestAmount` 卡死（§3.2③） | 改异常数模式（不受最小请求数约束） |
| 控制台平均 RT 在熔断期变好 | 被短路请求不产生 RT 样本（§1.2/§2.1③） | 看 block 指标列而非 RT 列 |
| 降级方法把熔断"打得更死" | blockHandler/fallback 抛非 Block 异常被计入失败（§4.1） | 降级路径 return 而非 throw |
| 想提前解除熔断 | 状态机无手动入口（§6.1③） | 临时改规则触发重建，事后务必改回 |

# 10. 相关阅读

链序与"被拦不计数"的第一道防线在 `2_architecture.md` §3.3/§5；配置语义与阈值配方在 `5_circuit_breaker.md`；主统计体系（与 §7 的第二套账相对照）在 `18_statistics_engine.md`；规则加载与 `loadRules` 整表替换时序在 `16_flow_control_source.md` §6、`14_dynamic_rules.md` §7；SPI 与 Slot 装配在 `19_spi_extension.md`/`15_slot_chain_source.md`；RPC/Feign 场景的熔断落地在 `11_dubbo_integration.md` §5、`13_feign_integration.md` §9；OPEN 期间的机器/曲线表现见 `9_dashboard.md` §4.4。
