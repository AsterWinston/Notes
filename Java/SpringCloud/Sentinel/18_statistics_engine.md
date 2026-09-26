
# 1. 概述

Sentinel 的实时统计能力基于**滑动窗口（LeapArray）**实现，是流控、熔断等所有规则判断的数据基础。

# 2. LeapArray — 滑动窗口核心

```java
public abstract class LeapArray<T> {

    protected int windowLengthInMs;   // 单个窗口时长 (ms)
    protected int sampleCount;        // 窗口数量
    protected int intervalInMs;       // 总统计时长 (ms)

    // 窗口数组（环形复用）
    protected final AtomicReferenceArray<WindowWrap<T>> array;

    // 构造函数
    public LeapArray(int sampleCount, int intervalInMs) {
        this.sampleCount = sampleCount;
        this.intervalInMs = intervalInMs;
        this.windowLengthInMs = intervalInMs / sampleCount;
        this.array = new AtomicReferenceArray<>(sampleCount);
    }

    // 抽象方法：子类实现窗口内数据的创建
    public abstract T newEmptyBucket(long timeMillis);

    // 窗口重置
    protected abstract WindowWrap<T> resetWindowTo(
        WindowWrap<T> windowWrap, long startTime);
}
```

# 3. 窗口定位与滚动

```java
// 获取当前时间所在的窗口
public WindowWrap<T> currentWindow() {
    return currentWindow(TimeUtil.currentTimeMillis());
}

public WindowWrap<T> currentWindow(long timeMillis) {
    long timeId = timeMillis / windowLengthInMs;
    int idx = (int) (timeId % array.length());

    // 无限循环，确保获取到正确窗口
    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            // 窗口不存在，创建新窗口
            WindowWrap<T> window = new WindowWrap<>(
                windowLengthInMs, timeId * windowLengthInMs, newEmptyBucket(timeMillis));
            if (array.compareAndSet(idx, null, window)) {
                return window;
            }
            Thread.yield();
        } else if (timeId == old.windowStart() / windowLengthInMs) {
            // 命中当前窗口，直接返回
            return old;
        } else if (timeId > old.windowStart() / windowLengthInMs) {
            // 窗口已过期，需要重置
            if (updateLock.tryLock()) {
                try {
                    return resetWindowTo(old, timeId * windowLengthInMs);
                } finally {
                    updateLock.unlock();
                }
            }
            Thread.yield();
        } else {
            // 时间回拨异常
            return new WindowWrap<>(windowLengthInMs,
                timeId * windowLengthInMs, newEmptyBucket(timeMillis));
        }
    }
}
```

# 4. 窗口值统计

```java
// 获取整个统计周期的值
public T values(long timeMillis) {
    List<T> validBuckets = new ArrayList<>();
    long currentTime = timeMillis;
    
    for (int i = 0; i < array.length(); i++) {
        WindowWrap<T> window = array.get(i);
        if (window == null) continue;
        
        // 检查窗口是否在当前统计周期内
        if (currentTime - window.windowStart() <= intervalInMs) {
            validBuckets.add(window.value());
        }
    }
    // 聚合所有有效窗口的值
    return aggregate(validBuckets);
}
```

# 5. MetricBucket — 指标桶

```java
public class MetricBucket {

    // 使用 LongAdder 记录各项指标
    private final LongAdder[] counters;

    // 指标类型（MetricEvent）
    public enum MetricEvent {
        PASS,           // 通过
        BLOCK,          // 阻塞
        EXCEPTION,      // 异常
        SUCCESS,        // 成功
        RT,             // 响应时间
        OCCUPIED_PASS   // 预占通过（优先级）
    }

    public MetricBucket() {
        this.counters = new LongAdder[MetricEvent.values().length];
        for (MetricEvent event : MetricEvent.values()) {
            counters[event.ordinal()] = new LongAdder();
        }
    }

    // 各项指标的累加
    public void add(MetricEvent event, long n) {
        counters[event.ordinal()].add(n);
    }

    public long get(MetricEvent event) {
        return counters[event.ordinal()].sum();
    }

    // 重置（窗口过期时调用）
    public void reset() {
        for (MetricEvent event : MetricEvent.values()) {
            counters[event.ordinal()].reset();
        }
    }
}
```

# 6. StatisticNode — 统计节点

```java
public class StatisticNode implements Node {

    // 滑动窗口（2 个窗口，共 1000ms）
    private transient volatile Metric rollingCounterInSecond
        = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT,
            IntervalProperty.INTERVAL);  // 默认 2 样本 / 1 秒

    // 分钟级统计（60 个窗口，共 60000ms）
    private transient Metric rollingCounterInMinute
        = new ArrayMetric(60, 60 * 1000, false);

    // 各项指标查询
    @Override
    public long totalPass() {
        return rollingCounterInSecond.pass();
    }

    @Override
    public long totalBlock() {
        return rollingCounterInSecond.block();
    }

    @Override
    public long totalException() {
        return rollingCounterInSecond.exception();
    }

    @Override
    public double avgRt() {
        long success = rollingCounterInSecond.success();
        long rt = rollingCounterInSecond.rt();
        return success == 0 ? 0 : (double) rt / success;
    }

    // 累加统计
    @Override
    public void addPassRequest(int count) {
        rollingCounterInSecond.addPass(count);
        rollingCounterInMinute.addPass(count);
    }
}
```

# 7. ArrayMetric — 指标聚合

```java
public class ArrayMetric implements Metric {

    private final LeapArray<MetricBucket> data;

    public ArrayMetric(int sampleCount, int intervalInMs) {
        this.data = new MetricLeapArray(sampleCount, intervalInMs);
    }

    // 内部类：实现 LeapArray 的抽象方法
    private static class MetricLeapArray extends LeapArray<MetricBucket> {
        @Override
        public MetricBucket newEmptyBucket(long time) {
            return new MetricBucket();
        }
        @Override
        protected WindowWrap<MetricBucket> resetWindowTo(
                WindowWrap<MetricBucket> w, long startTime) {
            w.resetTo(startTime);
            w.value().reset();  // 重置 MetricBucket
            return w;
        }
    }

    // 指标计算
    @Override
    public long pass() {
        data.currentWindow();  // 先确保窗口存在
        long pass = 0;
        for (MetricBucket bucket : data.values()) {
            pass += bucket.get(MetricEvent.PASS);
        }
        return pass;
    }
}
```

# 8. 时间窗口示意图

```
intervalInMs = 1000ms, sampleCount = 2

时间线 ──────────────────────────────────►
       0ms          500ms        1000ms
        │            │            │
   ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
   │ Window0 │ │ Window1 │ │ Window0 │  ← 环形复用
   │ [0,500) │ │[500,1000)│ │[1000,…) │
   └─────────┘ └─────────┘ └─────────┘
        ▲                        ▲
        └── 过期，被重置 ────────┘
```

# 9. ClusterNode — 全局统计

```java
public class ClusterNode extends StatisticNode {

    // 全局唯一，所有 Context 中的同名资源汇聚于此
    private final String name;
    private final int resourceType;

    @Override
    public void addPassRequest(int count) {
        super.addPassRequest(count);
        // ClusterNode 的统计是所有 Context 的汇总
    }
}
```

**与 DefaultNode 的关系**：
- `DefaultNode`：每个 Context 中同名资源的独立统计
- `ClusterNode`：所有 DefaultNode 的聚合，全局唯一

# 10. 性能优化设计

1. **LongAdder**：替代 AtomicLong，高并发下 CAS 竞争更低
2. **窗口复用**：环形数组轮转，避免频繁创建对象
3. **无锁读取**：通过 CAS 更新窗口引用，读取无需加锁
4. **细粒度锁**：仅窗口过期重置时加 `updateLock`
5. **懒创建**：窗口按需创建，不预分配数据
