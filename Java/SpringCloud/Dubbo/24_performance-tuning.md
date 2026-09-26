
# 1. 性能基准

## 1.1 参考基准

| 场景 | QPS | 平均延迟 | 99% 延迟 |
|------|-----|---------|---------|
| 小消息体 (100B) + Dubbo 协议 | 10w+ | < 1ms | < 3ms |
| 中等消息体 (1KB) + Dubbo 协议 | 5w+ | < 3ms | < 10ms |
| 大消息体 (1MB) + Dubbo 协议 | 2000+ | < 50ms | < 200ms |
| Triple + Protobuf (100B) | 8w+ | < 2ms | < 5ms |
| Triple + JSON (100B) | 5w+ | < 5ms | < 15ms |

> 测试环境：4C8G，千兆网卡，JDK 17

## 1.2 压测工具

```bash
# 使用 Dubbo 自带的性能测试工具
java -jar dubbo-benchmark.jar \
  --protocol dubbo \
  --host 192.168.1.100 \
  --port 20880 \
  --service org.example.UserService \
  --method getUser \
  --threads 100 \
  --duration 60s
```

# 2. 线程池调优

## 2.1 IO 线程

```yaml
dubbo:
  protocol:
    iothreads: 8            # 默认 CPU + 1, 建议 4-16
```

**调优原则**：
- IO 线程主要处理编解码和心跳，不阻塞
- 如果使用 `dispatcher: direct`，需增大 IO 线程以承载业务逻辑
- 过多 IO 线程会导致上下文切换开销

## 2.2 业务线程池

```yaml
dubbo:
  protocol:
    threads: 200            # 默认 200
    queues: 100              # 默认 0 (SynchronousQueue)
    threadpool: fixed        # fixed / cached / eager
```

**线程数计算**：

```
最优线程数 = N_cpu × N_core × (1 + W/C)

N_cpu  = CPU 核数
N_core = 核心数
W      = 线程等待时间 (IO 操作)
C      = 线程计算时间 (CPU)

示例：4C8G, IO 占比 80%
      = 4 × 2 × (1 + 0.8/0.2)
      = 40 线程
```

**实践经验**：

| 业务类型 | 建议线程数 | 队列 |
|---------|-----------|------|
| CPU 密集型（计算、加密） | CPU × 2-4 | 0 |
| IO 密集型（DB、缓存） | CPU × 8-16 | 100-500 |
| 混合型 | CPU × 4-8 | 50-200 |

## 2.3 线程池拒绝排查

当出现 `RejectedExecutionException` 时：

```bash
# 检查线程池状态
curl http://localhost:22222/threadpool

# 输出:
# {
#   "pool-name": "DubboServerHandler-192.168.1.100:20880",
#   "core-size": 200,
#   "max-size": 200,
#   "active-count": 200,       ← 活跃线程已满
#   "queue-size": 100,         ← 队列已满
#   "completed-task": 99999,
#   "largest-size": 200
# }
```

# 3. 序列化优化

## 3.1 序列化选择

| 序列化 | 体积 (100B payload) | 序列化耗时 | 反序列化耗时 | 场景 |
|--------|-------------------|-----------|------------|------|
| Kryo | 80B | 0.2μs | 0.3μs | 极致性能 |
| Protobuf | 70B | 0.3μs | 0.4μs | 跨语言/极致 |
| Hessian2 | 120B | 0.5μs | 0.8μs | 通用默认 |
| FastJSON2 | 150B | 0.8μs | 1.0μs | 调试友好 |
| JDK | 500B | 5μs | 10μs | 不推荐 |

```yaml
# 性能优先
dubbo:
  protocol:
    serialization: kryo

# 跨语言
dubbo:
  protocol:
    serialization: protobuf
```

## 3.2 Kryo 优化

```java
// 避免每次都创建 Kryo 实例（ThreadLocal 复用）
@Activate
public class KryoSerialization implements Serialization {
    // 使用 ThreadLocal 或对象池复用 Kryo 实例
    private static final ThreadLocal<Kryo> KRYO_THREAD_LOCAL = 
        ThreadLocal.withInitial(() -> {
            Kryo kryo = new Kryo();
            // 注册常用类，避免写入全限定类名
            kryo.register(User.class, 1001);
            kryo.register(Order.class, 1002);
            kryo.setRegistrationRequired(true); // 强制注册
            return kryo;
        });
}
```

# 4. 连接管理

## 4.1 连接数

```yaml
dubbo:
  consumer:
    connections: 2    # 每个 Consumer 到每个 Provider 的连接数
```

- 默认 `1` 个连接已足够大多数场景
- 消息量极大（5w+ QPS）时可增加到 `2-4`
- 注意：connections × providers × consumers 会指数增长

## 4.2 连接复用

```
Consumer A ──→ Provider B:
  ├── connection 1 (多个 service 共享)
  └── connection 2

同一 host:port 上即使有 100 个不同接口的服务
也只建立 1 (或 connections 配置的数量) 条 TCP 连接
```

# 5. 数据压缩

```yaml
dubbo:
  protocol:
    payload: 8388608      # 最大 payload (8MB)
    buffer: 16384          # 缓冲区大小
```

大数据传输优化：

```java
// 1. 避免传输大对象，改为传输引用或分页
// 2. 使用 Protobuf 获得更小的序列化体积
// 3. 启用压缩（如果需要）
@DubboService(
    parameters = {"compaction", "gzip"}
)
```

# 6. 批量请求优化

## 6.1 客户端合并请求

```java
// 多个独立请求合并为一次批量请求
CompletableFuture<User> userFuture = userService.getUser(1L);
CompletableFuture<Order> orderFuture = orderService.getOrder(100L);

// 并行执行
CompletableFuture.allOf(userFuture, orderFuture).join();

User user = userFuture.get();
Order order = orderFuture.get();
```

## 6.2 服务端批量接口

```java
// 设计批量接口，减少 RPC 次数
public interface UserService {
    // 不推荐：逐个查询 (N 次 RPC)
    User getUser(Long id);

    // 推荐：批量查询 (1 次 RPC)
    List<User> batchGetUsers(List<Long> ids);
}
```

# 7. Provider 端优化

## 7.1 避免 Provider 端成为瓶颈

```yaml
dubbo:
  provider:
    executes: 500           # 并发上限
    actives: 300            # Consumer 端并发上限
    accepts: 1000           # 最大连接数
    payload: 8388608         # 8MB payload 限制
```

## 7.2 结果缓存

```java
// 对于读多写少的接口
@DubboReference(cache = "lru", parameters = {"cache.size", "10000"})
private UserService userService;
```

## 7.3 异步 Provider

```java
@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public CompletableFuture<User> getUser(Long id) {
        // Provider 端异步处理（不阻塞 Tomcat/Dubbo 线程）
        return CompletableFuture.supplyAsync(
            () -> userRepository.findById(id),
            customThreadPool
        );
    }
}
```

# 8. Consumer 端优化

## 8.1 超时设置

```java
@DubboReference(
    timeout = 3000,           // 总超时 3s
    methods = {
        @Method(name = "getUser", timeout = 500),   // 快速查询 500ms
        @Method(name = "batchProcess", timeout = 10000) // 慢接口 10s
    }
)
```

## 8.2 重试策略

```java
@DubboReference(
    retries = 0,              // 写操作不重试
    cluster = "failfast"       // 快速失败
)
private UserService userService;
```

## 8.3 懒加载

```java
@DubboReference(lazy = true)  // 第一次使用时才初始化连接
private UserService userService;
```

# 9. 注册中心优化

```yaml
dubbo:
  registry:
    address: nacos://127.0.0.1:8848
    session: 60000
    timeout: 30000
    # 应用级服务发现（Dubbo 3）
  application:
    register-mode: instance
```

- 使用应用级服务发现降低注册中心压力
- 合理设置 session 超时（避免心跳过于频繁）

# 10. JVM 和操作系统调优

```bash
# JVM 参数优化
-server
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=50
-XX:+ParallelRefProcEnabled
-XX:+DisableExplicitGC

# Netty 堆外内存
-Dio.netty.maxDirectMemory=0  # 使用 -XX:MaxDirectMemorySize 控制
```

```bash
# 操作系统优化 (Linux)
# 文件描述符
ulimit -n 65535

# TCP 参数
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.tcp_tw_reuse=1
```

# 11. 性能检查清单

| 维度 | 检查项 | 推荐值 |
|------|--------|--------|
| 序列化 | 是否使用 Kryo/Protobuf | Kryo / Protobuf |
| 线程池 | threads 是否满足并发 | 根据 (1+W/C) 计算 |
| 连接 | connections 是否合理 | 1-2 |
| 超时 | timeout 避免堆积 | 区分快/慢接口 |
| 重试 | 写操作 retries | 0 |
| 负载均衡 | 策略是否匹配 | 默认 random |
| 路由 | 规则是否影响性能 | 评估必要性 |
| 注册 | 是否应用级发现 | instance (3.x) |
| 监听 | 是否必装所有 Filter | 移除冗余 |
| GC | GC 停顿是否在预期 | < 50ms |
