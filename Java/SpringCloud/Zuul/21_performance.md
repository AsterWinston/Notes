
# 1. 性能瓶颈分析

Zuul 1.x 基于 Servlet 阻塞模型，性能瓶颈主要集中在：

```
┌─────────────────────────────────────────┐
│  性能瓶颈区域                             │
│                                          │
│  1. Tomcat 线程池 (线程数 = 并发数)        │
│  2. HTTP 连接池 (后端连接创建/回收)        │
│  3. Hystrix 隔离策略 (信号量 vs 线程)      │
│  4. Ribbon 负载均衡 (实例选择延迟)         │
│  5. 超时配置 (过长的超时导致线程积压)       │
└─────────────────────────────────────────┘
```

# 2. Tomcat 线程池调优

```yaml
server:
  tomcat:
    threads:
      max: 200              # 最大工作线程数（建议 = 期望 QPS * 平均响应时间）
      min-spare: 20          # 最小空闲线程数
    accept-count: 200        # 请求等待队列长度
    max-connections: 20000   # 最大连接数（BIO 模式）
    connection-timeout: 20000 # 连接超时（ms）

    # 对于高并发场景，使用 NIO 模式
    protocol: org.apache.coyote.http11.Http11NioProtocol
```

线程数计算公式：

```
线程数 = QPS × 平均响应时间(秒)

例如: QPS=500, 平均响应时间=0.2s
线程数 = 500 × 0.2 = 100 → 配置 max=150 (留余量)
```

## 2.1 禁用 access log 阀门

```yaml
server:
  tomcat:
    accesslog:
      enabled: false  # 访问日志会占用 I/O，高并发下建议关闭或异步处理
```

# 3. HTTP 连接池调优

Zuul 通过 Apache HttpClient 管理到后端服务的连接：

```yaml
zuul:
  host:
    # 连接池配置
    max-total-connections: 500       # 全局最大连接数
    max-per-route-connections: 200   # 每个路由（后端服务）的最大连接数
    time-to-live: 600000            # 连接最大存活时间（ms），10分钟
    time-unit: MILLISECONDS

    # 超时配置
    socket-timeout-millis: 10000    # 读超时
    connect-timeout-millis: 2000    # 连接超时

    # 连接管理
    connection-request-timeout: 3000 # 从连接池获取连接的超时

# Spring Cloud Edgware 之前的写法
ribbon:
  MaxTotalConnections: 500
  MaxConnectionsPerHost: 200
  ReadTimeout: 10000
  ConnectTimeout: 2000
```

## 3.1 连接池大小计算

```
连接池大小 = 期望并发请求数

例如: QPS=500, 每个请求平均耗时 200ms
并发请求数 = 500 × 0.2 = 100

连接池建议配置: max-per-route-connections=200 (2倍余量)
```

# 4. 超时配置策略

## 4.1 超时层级关系

```
客户端超时 > Hystrix 超时 > Ribbon 超时 > 连接超时

正确配置层级：
  浏览器:              30s
  Zuul(Hystrix):       20s  → hystrix.command.default.timeoutInMilliseconds
  Ribbon 总超时:        10s  → (ConnectTimeout + ReadTimeout) × (重试次数)
  Ribbon ReadTimeout:   8s
  Ribbon ConnectTimeout: 2s
```

## 4.2 超时配置

```yaml
# Hystrix 超时（最终兜底）
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 20000

# Ribbon 超时
ribbon:
  ConnectTimeout: 2000
  ReadTimeout: 8000
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 1
  # 总超时 = (2000 + 8000) * (1+1) * (1+1) = 40000
  # 已经大于 Hystrix 超时 20000，需要调整
```

**修正后的配置**：

```yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 30000  # 必须 > Ribbon 总超时

ribbon:
  ConnectTimeout: 2000
  ReadTimeout: 5000
  MaxAutoRetries: 0           # 同实例不重试
  MaxAutoRetriesNextServer: 1  # 只换一次实例
  # 总超时 = (2000 + 5000) * (1+1) = 14000 < 30000 ✓
```

## 4.3 按服务配置超时

```yaml
# 核心服务：宽松超时
user-service:
  ribbon:
    ConnectTimeout: 2000
    ReadTimeout: 10000

hystrix:
  command:
    user-service:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 25000

# 非核心服务：快速失败
log-service:
  ribbon:
    ConnectTimeout: 1000
    ReadTimeout: 3000

hystrix:
  command:
    log-service:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 6000
```

# 5. Hystrix 隔离策略优化

## 5.1 信号量 vs 线程

```yaml
# 信号量隔离（默认、低延迟）：适合大部分场景
zuul:
  ribbon-isolation-strategy: SEMAPHORE
  semaphore:
    max-semaphores: 200   # 最大并发请求数

# 线程隔离：适合高延迟或需要超时控制的场景
zuul:
  ribbon-isolation-strategy: THREAD
  thread-pool:
    use-separate-thread-pools: true

hystrix:
  threadpool:
    default:
      coreSize: 20
      maximumSize: 40
      maxQueueSize: -1   # 无界队列（不推荐）
      queueSizeRejectionThreshold: 100
```

## 5.2 信号量调优

```yaml
# 信号量大小推导:
# max-semaphores = Tomcat 线程数 × 1.5
# 因为 Route 阶段在每个 Tomcat 线程中执行，信号量控制的就是并发转发数

zuul:
  semaphore:
    max-semaphores: 300  # 对应 Tomcat max-threads=200
```

# 6. Ribbon 性能优化

## 6.1 饥饿加载

```yaml
ribbon:
  eager-load:
    enabled: true
    clients:
      - user-service
      - order-service
```

饥饿加载将 Ribbon 客户端的初始化提前到应用启动阶段，避免首次请求的延迟（通常可减少 200-500ms）。

## 6.2 缓存 Ribbon 服务列表

```yaml
user-service:
  ribbon:
    ServerListRefreshInterval: 30000  # 默认 30s，可适当调大减少刷新频率
```

## 6.3 禁用不必要的能力

```yaml
# 如果不需要 Hystrix，禁用可以减小开销
hystrix:
  command:
    default:
      circuitBreaker:
        enabled: false
```

> 注意：此配置仅适用于纯透传场景，正常生产环境应保持熔断器开启。

# 7. Zuul 层的优化

## 7.1 关闭 Debug 模式

```yaml
zuul:
  # 生产环境不要开启
  debug:
    request: false       # 调试请求追迹
    parameter: false     # 调试参数
```

## 7.2 精简过滤器链

不用的内置过滤器可以通过排除自配置的方式移除：

```java
@Configuration
public class ZuulFilterOptimization {

    // 移除不用的过滤器
    @Bean
    @Primary
    public FilterConfigurationEndpoint filterConfigurationEndpoint(
            List<ZuulFilter> filters) {
        // 过滤掉不必要的过滤器
        return new FilterConfigurationEndpoint(
            filters.stream()
                .filter(f -> !(f instanceof DebugFilter))  // 移除 DebugFilter
                .collect(Collectors.toList()));
    }
}
```

## 7.3 减少 RequestContext 的数据复制

```java
// 不要将大对象存入 RequestContext
// 坏示例
ctx.set("userList", loadAllUsers());  // 可能返回上万条数据

// 好示例
ctx.set("userId", extractUserId(request));
ctx.set("userName", extractUserName(request));
```

# 8. JVM 调优

```bash
# 推荐 JVM 参数
java -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:ParallelGCThreads=4 \
     -XX:ConcGCThreads=2 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     -jar zuul-gateway.jar
```

- 堆内存大小取决于并发量和请求体大小，一般 2-4G 足够
- G1GC 是 JDK 8+ 的推荐选择，延迟可控
- `MaxGCPauseMillis=200` 避免长时间 GC 停顿

# 9. 性能压测方法

## 9.1 压测工具

```bash
# Apache Bench
ab -n 10000 -c 100 http://gateway:8080/user/list

# wrk
wrk -t 4 -c 100 -d 30s http://gateway:8080/user/list
```

## 9.2 关注指标

| 指标 | 健康值 | 告警值 |
|------|--------|--------|
| QPS | 依赖配置 | 下降 30% |
| P99 延迟 | < 500ms | > 1s |
| 错误率 | < 0.1% | > 1% |
| 熔断器打开 | 0 | > 0 |
| 线程池排队 | < 10 | > 50 |

## 9.3 常见性能问题排查

1. **延迟突然升高** → 检查后端服务响应时间、GC 停顿、网络延迟
2. **线程池耗尽** → 检查 Hystrix 超时是否小于 Ribbon 超时、是否有慢查询
3. **连接池耗尽** → 检查 `max-per-route-connections` 是否足够、后端是否保持了长连接
4. **频繁 GC** → 检查 `RequestContext.unset()` 是否正确调用、是否有大对象频繁创建
