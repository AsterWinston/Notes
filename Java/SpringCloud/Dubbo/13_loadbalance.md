
# 1. 负载均衡概述

负载均衡（LoadBalance）负责从多个 Provider 中选择一个进行调用。Dubbo 内置 5 种负载均衡策略，支持基于 SPI 自定义扩展。

```java
@SPI(RandomLoadBalance.NAME)  // 默认随机
public interface LoadBalance {
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, 
        Invocation invocation) throws RpcException;
}
```

# 2. RandomLoadBalance（随机，默认）

## 2.1 算法原理

基于**权重随机**，权重大的 Provider 被选中的概率高。

```java
public class RandomLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "random";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        int length = invokers.size();
        boolean sameWeight = true;
        int[] weights = new int[length];
        
        // 1. 计算第一个权重
        int firstWeight = getWeight(invokers.get(0), invocation);
        weights[0] = firstWeight;
        int totalWeight = firstWeight;
        
        // 2. 计算总权重 + 判断是否所有权重相同
        for (int i = 1; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            weights[i] = weight;
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) {
                sameWeight = false;
            }
        }
        
        // 3. 权重不相同 → 按权重随机选择
        if (totalWeight > 0 && !sameWeight) {
            int offset = ThreadLocalRandom.current()
                .nextInt(totalWeight);
            for (int i = 0; i < length; i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        
        // 4. 所有权重相同 → 完全随机
        return invokers.get(
            ThreadLocalRandom.current().nextInt(length));
    }
}
```

## 2.2 权重计算

```java
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    int weight = invoker.getUrl().getMethodParameter(
        invocation.getMethodName(), "weight", 100);
    
    if (weight > 0) {
        long timestamp = invoker.getUrl().getParameter("timestamp", 0L);
        if (timestamp > 0L) {
            // 启动时长（ms）
            long uptime = System.currentTimeMillis() - timestamp;
            // 预热时间（默认 10 分钟）
            int warmup = invoker.getUrl().getParameter("warmup", 600000);
            
            if (uptime > 0 && uptime < warmup) {
                // 预热期间，按启动时长比例降低权重
                weight = calculateWarmupWeight(uptime, warmup, weight);
            }
        }
    }
    return Math.max(weight, 0);
}

// 预热权重计算：使权重随时间平滑增长
static int calculateWarmupWeight(long uptime, int warmup, int weight) {
    int ww = (int) (uptime / ((float) warmup / weight));
    return ww < 1 ? 1 : (Math.min(ww, weight));
}
```

**预热曲线**：

```
权重
100% ┤                               ,-------
 90% ┤                        ,----'
 70% ┤                  ,----'
 50% ┤            ,----'
 30% ┤      ,----'
 10% ┤,----'
     └─────┬─────┬─────┬─────┬─────→ 启动时间
          2min  4min  6min  8min  10min
```

# 3. RoundRobinLoadBalance（轮询）

## 3.1 算法原理

基于**加权轮询 + Nginx 平滑算法**，使请求分布更加平滑。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "roundrobin";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        String key = invokers.get(0).getUrl().getServiceKey() 
            + "." + invocation.getMethodName();
        
        // 每个方法维护独立的轮询状态
        ConcurrentMap<String, WeightedRoundRobin> map = 
            methodWeightMap.computeIfAbsent(key, 
                k -> new ConcurrentHashMap<>());
        
        int totalWeight = 0;
        long maxCurrent = Long.MIN_VALUE;
        WeightedRoundRobin selectedWRR = null;
        Invoker<T> selectedInvoker = null;
        
        for (Invoker<T> invoker : invokers) {
            String identifyString = invoker.getUrl().toIdentityString();
            int weight = getWeight(invoker, invocation);
            
            WeightedRoundRobin weightedRoundRobin = map.get(identifyString);
            if (weightedRoundRobin == null) {
                weightedRoundRobin = new WeightedRoundRobin(weight);
                map.putIfAbsent(identifyString, weightedRoundRobin);
            }
            
            // 权重变化时更新
            if (weight != weightedRoundRobin.getWeight()) {
                weightedRoundRobin.setWeight(weight);
            }
            
            // current += weight (核心: 每次循环增加权重)
            long cur = weightedRoundRobin.increaseCurrent();
            weightedRoundRobin.setLastUpdate(System.currentTimeMillis());
            
            // 选择 current 最大的 Invoker
            if (cur > maxCurrent) {
                maxCurrent = cur;
                selectedWRR = weightedRoundRobin;
                selectedInvoker = invoker;
            }
            totalWeight += weight;
        }
        
        if (selectedInvoker != null) {
            // selected current -= totalWeight (核心: 选中者减去总权重)
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        
        return invokers.get(0);
    }

    protected static class WeightedRoundRobin {
        private int weight;
        private final AtomicLong current = new AtomicLong(0);
        private long lastUpdate;

        long increaseCurrent() {
            return current.addAndGet(weight);
        }

        void sel(int total) {
            current.addAndGet(-1 * total);
        }
    }
}
```

## 3.2 Nginx 平滑算法示例

假设 Provider A(weight=5), B(weight=1), C(weight=1)，totalWeight=7：

| 轮次 | 选择前 current | 选中 | 选择后 current |
|------|---------------|------|---------------|
| 1 | A=5, B=1, C=1 | A(5) | A=-2, B=1, C=1 |
| 2 | A=3, B=2, C=2 | A(3) | A=-4, B=2, C=2 |
| 3 | A=1, B=3, C=3 | B(3) | A=1, B=-4, C=3 |
| 4 | A=6, B=-3, C=4 | A(6) | A=-1, B=-3, C=4 |
| 5 | A=4, B=-2, C=5 | C(5) | A=4, B=-2, C=-2 |
| 6 | A=9, B=-1, C=-1 | A(9) | A=2, B=-1, C=-1 |
| 7 | A=7, B=0, C=0 | A(7) | A=0, B=0, C=0 |

结果：A 出现 5 次，B 1 次，C 1 次，分布均匀。

# 4. LeastActiveLoadBalance（最少活跃数）

## 4.1 算法原理

选择**活跃调用数最少**的 Provider，活跃数相同则按权重随机。

```java
public class LeastActiveLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "leastactive";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        int length = invokers.size();
        int leastActive = -1;         // 最小活跃数
        int leastCount = 0;           // 最小活跃数的 Invoker 个数
        int[] leastIndexes = new int[length];
        int[] weights = new int[length];
        int totalWeight = 0;
        int firstWeight = 0;
        boolean sameWeight = true;
        
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            
            // 获取活跃数（当前正在处理的请求数）
            int active = RpcStatus.getStatus(
                invoker.getUrl(), invocation.getMethodName()).getActive();
            int afterWarmup = getWeight(invoker, invocation);
            weights[i] = afterWarmup;
            
            if (leastActive == -1 || active < leastActive) {
                // 发现更小的活跃数，重置
                leastActive = active;
                leastCount = 1;
                leastIndexes[0] = i;
                totalWeight = afterWarmup;
                firstWeight = afterWarmup;
                sameWeight = true;
                
            } else if (active == leastActive) {
                // 活跃数相同，加入候选
                leastIndexes[leastCount++] = i;
                totalWeight += afterWarmup;
                if (sameWeight && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        
        // 活跃数最低的只有一个 → 直接返回
        if (leastCount == 1) {
            return invokers.get(leastIndexes[0]);
        }
        
        // 活跃数最低的有多个 → 按权重随机
        if (!sameWeight && totalWeight > 0) {
            int offsetWeight = ThreadLocalRandom.current()
                .nextInt(totalWeight);
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        
        // 权重相同 → 随机
        return invokers.get(leastIndexes[
            ThreadLocalRandom.current().nextInt(leastCount)]);
    }
}
```

## 4.2 活跃数统计

```java
public class RpcStatus {
    private static final ConcurrentMap<String, RpcStatus> SERVICE_STATISTICS = 
        new ConcurrentHashMap<>();
    private static final ConcurrentMap<String, ConcurrentMap<String, RpcStatus>> 
        METHOD_STATISTICS = new ConcurrentHashMap<>();

    private final AtomicInteger active = new AtomicInteger();     // 当前活跃

    public static void beginCount(URL url, String methodName) {
        // Filter 调用 beginCount → active + 1
        getStatus(url, methodName).active.incrementAndGet();
    }

    public static void endCount(URL url, String methodName, 
            long elapsed, boolean succeeded) {
        // Filter 调用 endCount → active - 1
        getStatus(url, methodName).active.decrementAndGet();
    }

    public int getActive() {
        return active.get();
    }
}
```

# 5. ConsistentHashLoadBalance（一致性哈希）

## 5.1 算法原理

将请求参数（第一个参数）哈希到虚拟节点环上，顺时针找到最近的 Provider。

```java
public class ConsistentHashLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "consistenthash";

    // 每个方法维护独立的哈希环，缓存避免重复构建
    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = 
        new ConcurrentHashMap<>();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        String methodName = invocation.getMethodName();
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
        
        // 计算 invokers 的 hash 值，检测是否变化
        int identityHashCode = System.identityHashCode(invokers);
        
        ConsistentHashSelector<T> selector = selectors.get(key);
        if (selector == null 
                || selector.identityHashCode != identityHashCode) {
            selectors.put(key, new ConsistentHashSelector<>(
                invokers, methodName, identityHashCode));
            selector = selectors.get(key);
        }
        
        return selector.select(invocation);
    }

    private static final class ConsistentHashSelector<T> {
        // 虚拟节点（TreeMap 实现一致性哈希环）
        private final TreeMap<Long, Invoker<T>> virtualInvokers;
        private final int replicaNumber; // 默认 160 个虚拟节点/Provider
        private final int identityHashCode;
        private final int[] argumentIndex; // 参与哈希的参数索引

        ConsistentHashSelector(List<Invoker<T>> invokers, 
                String methodName, int identityHashCode) {
            
            this.virtualInvokers = new TreeMap<>();
            this.identityHashCode = identityHashCode;
            this.replicaNumber = 160;
            
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                for (int i = 0; i < replicaNumber / 4; i++) {
                    // 4 种哈希算法，每种生成 40 个虚拟节点
                    byte[] digest = Bytes.getMD5(address + i);
                    for (int h = 0; h < 4; h++) {
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }

        public Invoker<T> select(Invocation invocation) {
            // 将第一个参数值作为哈希 key
            String key = toKey(invocation.getArguments());
            byte[] digest = Bytes.getMD5(key);
            return selectForKey(hash(digest, 0));
        }

        private Invoker<T> selectForKey(long hash) {
            // TreeMap.tailMap：找到 >= hash 的第一个虚拟节点
            Map.Entry<Long, Invoker<T>> entry = 
                virtualInvokers.tailMap(hash, true).firstEntry();
            if (entry == null) {
                entry = virtualInvokers.firstEntry(); // 环形，从头开始
            }
            return entry.getValue();
        }

        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                  | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                  | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                  | (digest[number * 4] & 0xFF))
                  & 0xFFFFFFFFL;
        }
    }
}
```

**虚拟节点环示意**：

```
        0
        │
    ┌───┼───┐
    │       │
    │  Hash │   Provider A 的虚拟节点 (×40)
    │  Ring │   Provider B 的虚拟节点 (×40)
    │       │   Provider C 的虚拟节点 (×40)
    │       │
    └───┼───┘
      2^32-1
```

# 6. ShortestResponseLoadBalance（最短响应时间）

## 6.1 算法原理

基于**滑动窗口**统计每个 Provider 的响应时间，选择预估响应时间最短的节点。

```java
public class ShortestResponseLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "shortestresponse";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        int length = invokers.size();
        long shortestResponse = Long.MAX_VALUE;
        int shortestCount = 0;
        int[] shortestIndexes = new int[length];
        int[] weights = new int[length];
        int totalWeight = 0;
        int firstWeight = 0;
        boolean sameWeight = true;
        
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            
            // 获取滑动窗口内平均成功响应时间
            RpcStatus rpcStatus = RpcStatus.getStatus(
                invoker.getUrl(), invocation.getMethodName());
            
            long succeededAverageElapsed = 
                rpcStatus.getSucceededAverageElapsed();
            int active = rpcStatus.getActive();
            
            // 预估响应时间 = 平均响应时间 × 活跃请求数
            long estimateResponse = succeededAverageElapsed * active;
            int afterWarmup = getWeight(invoker, invocation);
            weights[i] = afterWarmup;
            
            if (estimateResponse < shortestResponse) {
                shortestResponse = estimateResponse;
                shortestCount = 1;
                shortestIndexes[0] = i;
                totalWeight = afterWarmup;
                firstWeight = afterWarmup;
                sameWeight = true;
            } else if (estimateResponse == shortestResponse) {
                shortestIndexes[shortestCount++] = i;
                totalWeight += afterWarmup;
                if (sameWeight && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        
        // 选择逻辑同 LeastActiveLoadBalance
        if (shortestCount == 1) {
            return invokers.get(shortestIndexes[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            int offsetWeight = ThreadLocalRandom.current()
                .nextInt(totalWeight);
            for (int i = 0; i < shortestCount; i++) {
                offsetWeight -= weights[shortestIndexes[i]];
                if (offsetWeight < 0) {
                    return invokers.get(shortestIndexes[i]);
                }
            }
        }
        return invokers.get(shortestIndexes[
            ThreadLocalRandom.current().nextInt(shortestCount)]);
    }
}
```

# 7. 自适应负载均衡（Dubbo 3 新特性）

Dubbo 3.2+ 引入了基于**启发式算法**的自适应负载均衡（AdaptiveLoadBalance），综合考虑 CPU 使用率、内存、响应时间等指标。

```java
public class AdaptiveLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "adaptive";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers,
            URL url, Invocation invocation) {
        
        // 采集每个 Provider 的实时负载指标
        // 1. 响应时间
        // 2. 并发数
        // 3. CPU 负载
        // 4. 内存使用率
        // 综合计算得分，选择得分最高的 Provider
        // ...
    }
}
```

# 8. 策略选择指南

| 策略 | 适用场景 | 特点 |
|------|---------|------|
| **random** | 通用场景（默认） | 简单高效，权重可配 |
| **roundrobin** | 机器性能均匀 | 严格平均，请求平滑 |
| **leastactive** | 长连接、耗时不均 | 自动感知 Provider 负载 |
| **consistenthash** | 有状态服务、缓存亲和 | 相同参数总是路由到同一 Provider |
| **shortestresponse** | 响应时间敏感 | 动态选择最快节点 |

```yaml
dubbo:
  consumer:
    loadbalance: leastactive
```

```java
@DubboReference(loadbalance = "consistenthash")
private UserService userService;
```
