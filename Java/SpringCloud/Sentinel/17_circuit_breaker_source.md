
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

# 6. 状态转换详解

```
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
  │               │
  ▼               ▼
OPEN→HALF_OPEN  拒绝
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

# 8. 与 Hystrix 熔断的对比

| 维度 | Sentinel | Hystrix |
|------|----------|---------|
| 熔断策略 | 慢调用/异常比例/异常数 | 仅异常比例 |
| 半开机制 | CAS 保证单探测 | 单请求探测 |
| 统计窗口 | 滑动窗口（任意 interval） | 固定窗口（10s） |
| 分级熔断 | ✓ 支持多级规则 | ✗ 单一规则 |
| 线程池隔离 | ✗（推荐信号量隔离） | ✓ |
