
# 1. FlowSlot — 入口

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

# 2. FlowRuleChecker — 规则校验器

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

# 3. 流控模式匹配

## 3.1 直接模式（STRATEGY_DIRECT）

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

## 3.2 关联模式（STRATEGY_RELATE）

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

## 3.3 链路模式（STRATEGY_CHAIN）

```java
private static Node selectNodeByChain(FlowRule rule, Context context,
                                       DefaultNode node) {
    String refResource = rule.getRefResource();
    // 只统计从特定入口进入的流量
    DefaultNode originNode = node.getOriginNode(refResource);
    return originNode;
}
```

# 4. TrafficShapingController — 流量整形

```java
public interface TrafficShapingController {

    // 判断是否可以放行
    boolean canPass(Node node, int acquireCount, boolean prioritized);

    // 是否可以继续排队
    boolean canPass(Node node, int acquireCount);
}
```

## 4.1 DefaultController（快速失败）

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

## 4.2 WarmUpController（预热）

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

## 4.3 RateLimiterController（匀速排队）

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

# 5. FlowRuleManager — 规则管理

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

# 6. TrafficShaping 选择策略

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
