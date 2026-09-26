
# 1. 算法概览

| 算法 | 类名 | 适用场景 | 特点 |
|------|------|---------|------|
| **轮询** | `RoundRobinLoadBalancer`（默认） | 实例性能均等 | 简单、公平 |
| **随机** | `RandomLoadBalancer` | 实例性能均等 | 无状态 |
| **Nacos 加权** | `NacosLoadBalancer` | 实例性能不均 | 按权重分配 |
| **Nacos 集群优先** | `NacosLoadBalancer` | 多集群/多机房 | 同集群优先 |
| **自定义** | 实现 `ReactorLoadBalancer` | 特殊需求 | 完全定制 |

# 2. RoundRobinLoadBalancer（轮询）

## 2.1 算法原理

```java
public class RoundRobinLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final AtomicInteger position;              // 自增计数器
    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request)
                .next()                                // 取 Flux 第一个元素
                .map(serviceInstances -> processInstanceResponse(
                        supplier, serviceInstances));
    }

    private Response<ServiceInstance> processInstanceResponse(
            ServiceInstanceListSupplier supplier,
            List<ServiceInstance> instances) {
        Response<ServiceInstance> response = getInstanceResponse(instances);
        if (supplier instanceof SelectedInstanceCallback) {
            ((SelectedInstanceCallback) supplier)
                    .selectedServiceInstance(response.getServer());
        }
        return response;
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            log.warn("No servers available for service: {}", serviceId);
            return new EmptyResponse();
        }

        // 原子自增 + 取模 = 轮询
        int pos = Math.abs(this.position.incrementAndGet());
        ServiceInstance instance = instances.get(pos % instances.size());
        return new DefaultResponse(instance);
    }
}
```

## 2.2 轮询示意图

```
请求序号  position   取模结果  选中实例
────────────────────────────────────
  1        1           1 % 3 = 1  → 实例 B
  2        2           2 % 3 = 2  → 实例 C
  3        3           3 % 3 = 0  → 实例 A
  4        4           4 % 3 = 1  → 实例 B
  5        5           5 % 3 = 2  → 实例 C
  ...

position 使用 AtomicInteger 保证线程安全，无锁高并发。
```

## 2.3 使用方式（默认）

无需任何配置，默认即为 `RoundRobinLoadBalancer`。

# 3. RandomLoadBalancer（随机）

## 3.1 算法原理

```java
public class RandomLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;
    private final ThreadLocalRandom random;           // 高性能线程随机

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request)
                .next()
                .map(instances -> getInstanceResponse(supplier, instances));
    }

    private Response<ServiceInstance> getInstanceResponse(
            ServiceInstanceListSupplier supplier,
            List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }

        // 从 0 ~ size-1 随机选取
        int index = random.nextInt(instances.size());
        ServiceInstance instance = instances.get(index);
        return new DefaultResponse(instance);
    }
}
```

## 3.2 使用方式

```java
@Configuration
@LoadBalancerClient(name = "user-service",
        configuration = RandomLoadBalancerConfig.class)
public class RandomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment, LoadBalancerClientFactory factory) {
        String name = environment.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

# 4. NacosLoadBalancer（Nacos 加权）

## 4.1 加权轮询原理

```java
public class NacosLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request)
                .next()
                .map(instances -> getInstanceResponse(instances));
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }

        // 转换为 Nacos NamingService 的 Instance
        Instance instanceToChoose = getInstanceByWeight(
                hostToWeight(instances));

        // 从 Nacos Instance 反查 Spring ServiceInstance
        ServiceInstance selected = instances.stream()
                .filter(i -> match(i, instanceToChoose))
                .findFirst()
                .orElse(null);
        return new DefaultResponse(selected);
    }
}
```

**权重配置**（在 Nacos 实例的元数据中）：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          nacos.weight: 10       # 权重 1~100，越大分配越多
```

**集群优先**：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: SH          # 当前服务所在集群
        # NacosLoadBalancer 默认同集群优先

    loadbalancer:
      nacos:
        enabled: true
```

## 4.2 加权随机示意图

```
实例权重: A=1, B=2, C=3

权重区间: [0,1): A   [1,3): B   [3,6): C

随机数 0~5:
  0 → A
  1 → B       B 的概率 = 2/6 = 33%
  2 → B
  3 → C       C 的概率 = 3/6 = 50%
  4 → C
  5 → C
```

# 5. 自定义负载均衡算法

## 5.1 一致性哈希负载均衡

```java
public class ConsistentHashLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;
    private final TreeMap<Integer, ServiceInstance> ring = new TreeMap<>();
    private final int virtualNodes = 160;      // 虚拟节点数

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable();
        return supplier.get(request)
                .next()
                .map(instances -> {
                    if (instances.isEmpty()) return new EmptyResponse();

                    // 根据请求上下文计算哈希 key
                    RequestDataContext ctx = (RequestDataContext) request.getContext();
                    String hashKey = ctx.getClientRequest().getHeaders()
                            .getFirst("X-User-Id");

                    return new DefaultResponse(
                            selectByHash(instances, hashKey));
                });
    }

    private ServiceInstance selectByHash(
            List<ServiceInstance> instances, String hashKey) {
        rebuildRing(instances);
        int hash = hash(hashKey);
        if (!ring.containsKey(hash)) {
            Map.Entry<Integer, ServiceInstance> entry =
                    ring.ceilingEntry(hash);             // 顺时针找下一个
            if (entry == null) {
                entry = ring.firstEntry();               // 回到起始
            }
            return entry.getValue();
        }
        return ring.get(hash);
    }

    private void rebuildRing(List<ServiceInstance> instances) {
        ring.clear();
        for (ServiceInstance instance : instances) {
            for (int i = 0; i < virtualNodes; i++) {
                ring.put(hash(instance.getHost() + "#" + i), instance);
            }
        }
    }

    private int hash(String key) {
        // MurmurHash / Ketama 等一致性哈希实现
        return key.hashCode() & 0x7fffffff;
    }
}
```

## 5.2 灰度发布负载均衡

```java
public class GrayLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;
    private final AtomicInteger position = new AtomicInteger(0);

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable();
        return supplier.get(request)
                .next()
                .map(instances -> {
                    if (instances.isEmpty()) return new EmptyResponse();

                    // 从请求头获取灰度标记
                    RequestDataContext ctx = (RequestDataContext) request.getContext();
                    String grayTag = ctx.getClientRequest().getHeaders()
                            .getFirst("X-Gray-Version");

                    // 按灰度标记过滤实例
                    List<ServiceInstance> candidates;
                    if (grayTag != null) {
                        candidates = instances.stream()
                                .filter(i -> grayTag.equals(
                                        i.getMetadata().get("version")))
                                .collect(Collectors.toList());
                    } else {
                        candidates = instances.stream()
                                .filter(i -> !"gray".equals(
                                        i.getMetadata().get("version")))
                                .collect(Collectors.toList());
                    }

                    if (candidates.isEmpty()) {
                        candidates = instances;  // fallback
                    }

                    int pos = Math.abs(position.incrementAndGet());
                    return new DefaultResponse(
                            candidates.get(pos % candidates.size()));
                });
    }
}
```

## 5.3 最小响应时间负载均衡

```java
public class LeastResponseTimeLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    private final Map<String, MovingAverage> rtStats = new ConcurrentHashMap<>();

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable();
        return supplier.get(request)
                .next()
                .map(instances -> {
                    if (instances.isEmpty()) return new EmptyResponse();

                    // 选平均响应时间最小的实例
                    ServiceInstance best = instances.stream()
                            .min(Comparator.comparingDouble(
                                    i -> rtStats.computeIfAbsent(
                                            getKey(i),
                                            k -> new MovingAverage())
                                            .getAverage()))
                            .orElse(instances.get(0));
                    return new DefaultResponse(best);
                });
    }

    public void recordResponseTime(ServiceInstance instance, long rtMs) {
        rtStats.computeIfAbsent(getKey(instance),
                k -> new MovingAverage()).add(rtMs);
    }

    private String getKey(ServiceInstance instance) {
        return instance.getHost() + ":" + instance.getPort();
    }

    // 滑动平均
    private static class MovingAverage {
        private double average;
        private final double alpha = 0.3;   // 衰减因子

        void add(long value) {
            average = alpha * value + (1 - alpha) * average;
        }

        double getAverage() {
            return average;
        }
    }
}
```

# 6. 算法选择指南

| 场景 | 推荐算法 | 原因 |
|------|---------|------|
| 实例性能相同 | RoundRobin / Random | 简单可靠 |
| 实例性能不同 | NacosLoadBalancer（加权） | 按能力分配流量 |
| 多机房部署 | NacosLoadBalancer（集群优先） | 就近路由 |
| 需要会话粘滞 | ConsistentHash | 同一用户路由到同一实例 |
| 灰度发布 | GrayLoadBalancer | 按版本标签路由 |
| 多网卡/权重动态 | LeastResponseTime | 自适应最优实例 |
