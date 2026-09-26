
# 1. Hystrix 在 Zuul 中的角色

Hystrix 是 Netflix 的熔断器库，在 Zuul 网关中提供以下核心能力：

- **熔断（Circuit Breaker）**：后端服务连续失败时自动熔断，快速失败
- **降级（Fallback）**：服务不可用时提供兜底响应
- **隔离（Isolation）**：线程池或信号量隔离，防止故障扩散
- **超时控制**：设置请求超时时间，避免长时间等待
- **监控（Metrics）**：收集熔断器状态和请求指标

```
Zuul 请求 → HystrixCommand → 正常：转发到后端 → 收到响应
                            → 异常/超时/熔断 → Fallback → 降级响应
```

# 2. 两种隔离策略

## 2.1 信号量隔离（SEMAPHORE，默认）

信号量隔离下，HTTP 调用**在当前 Tomcat 线程中执行**，通过信号量控制并发数：

```yaml
zuul:
  ribbon-isolation-strategy: SEMAPHORE

hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: SEMAPHORE
          semaphore:
            maxConcurrentRequests: 100  # 最大并发请求数
```

信号量隔离的优缺点：

| 优点 | 缺点 |
|------|------|
| 无线程切换开销，延迟更低 | 调用线程被阻塞，会占用 Tomcat 线程 |
| 无需额外线程池管理 | 不适合耗时长的调用 |
| 适合高并发低延迟场景 | 无法处理超时中断 |

## 2.2 线程隔离（THREAD）

线程隔离下，HTTP 调用在**Hystrix 线程池**中执行，Tomcat 线程可以释放：

```yaml
zuul:
  ribbon-isolation-strategy: THREAD

hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: THREAD
          thread:
            timeoutInMilliseconds: 5000
            interruptOnTimeout: true
```

线程隔离的优缺点：

| 优点 | 缺点 |
|------|------|
| 完全隔离，故障不扩散 | 线程切换有额外开销 |
| 支持超时中断 | 需要管理多个线程池 |
| 适合耗时较长的调用 | `ThreadLocal` 上下文需手动传递 |

## 2.3 策略选择建议

```
                  是否 IO 密集？
                  /          \
               是              否
               │               │
         响应时间 < 10ms?    线程隔离
          /         \
       是            否
       │             │
    信号量隔离    线程隔离
```

对于 Zuul 网关场景，路由转发属于 IO 密集型操作，响应时间通常大于 10ms，推荐使用**线程隔离**。但需要注意 `RequestContext`（基于 ThreadLocal）的上下文传递问题。

# 3. Hystrix 超时配置

## 3.1 超时配置关系

Hystrix 超时与 Ribbon 超时的关系：

```
Hystrix 超时 > Ribbon (ConnectTimeout + ReadTimeout) * (MaxAutoRetries + 1) * (MaxAutoRetriesNextServer + 1)
```

```yaml
# 推荐配置
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 20000  # Hystrix 超时（必须大于 Ribbon 总超时）

user-service:
  ribbon:
    ConnectTimeout: 2000
    ReadTimeout: 5000
    MaxAutoRetries: 1
    MaxAutoRetriesNextServer: 1

# 计算：Ribbon 总超时 = (2000+5000) * (1+1) * (1+1) = 28000ms
# 因此 Hystrix 超时应 >= 30000ms
```

## 3.2 禁用 Hystrix 超时

```yaml
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: false  # 禁用 Hystrix 超时，仅依赖 Ribbon 超时
```

# 4. 熔断器配置

## 4.1 熔断器参数

```yaml
hystrix:
  command:
    default:
      circuitBreaker:
        enabled: true                      # 启用熔断器
        requestVolumeThreshold: 20         # 时间窗口内的最小请求数
        errorThresholdPercentage: 50       # 错误率阈值（50%）
        sleepWindowInMilliseconds: 5000    # 熔断后休眠时间（ms）
```

熔断器状态转换：

```
         ┌────────────────┐
         │    CLOSED       │  ← 正常状态，请求正常通过
         │  (熔断器关闭)    │
         └───────┬────────┘
                 │ 错误率达到阈值 (50%) 且请求数 > 20
                 ▼
         ┌────────────────┐
         │     OPEN        │  ← 熔断状态，直接拒绝请求
         │  (熔断器打开)    │
         └───────┬────────┘
                 │ 休眠窗口过后 (5s)
                 ▼
         ┌────────────────┐
         │   HALF_OPEN     │  ← 尝试恢复，允许少量请求
         │  (半开状态)      │
         └───────┬────────┘
          /      │      \
     成功        失败      超时
      │          │        │
      ▼          ▼        ▼
   CLOSED     OPEN     OPEN
```

## 4.2 按服务配置熔断策略

```yaml
hystrix:
  command:
    # 默认配置
    default:
      circuitBreaker:
        requestVolumeThreshold: 20
        errorThresholdPercentage: 50
        sleepWindowInMilliseconds: 5000

    # user-service 更严格
    user-service:
      circuitBreaker:
        requestVolumeThreshold: 10
        errorThresholdPercentage: 30
        sleepWindowInMilliseconds: 10000

    # order-service 较宽松（关键业务）
    order-service:
      circuitBreaker:
        requestVolumeThreshold: 50
        errorThresholdPercentage: 70
        sleepWindowInMilliseconds: 3000
```

# 5. 降级处理（Fallback）

## 5.1 FallbackProvider 接口

```java
public interface FallbackProvider {
    // 为哪些路由提供降级
    String getRoute();

    // 降级响应
    ClientHttpResponse fallbackResponse(String route, Throwable cause);
}
```

## 5.2 实现自定义降级

```java
@Component
public class UserServiceFallback implements FallbackProvider {

    @Override
    public String getRoute() {
        return "user-service";  // 指定降级的目标路由
        // return "*";          // 为所有路由提供降级
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        // 不同异常返回不同降级响应
        if (cause instanceof HystrixTimeoutException) {
            return buildResponse(504, "请求超时，请稍后重试");
        }
        if (cause.getCause() instanceof ConnectException) {
            return buildResponse(503, "服务暂时不可用");
        }
        return buildResponse(500, "系统繁忙，请稍后重试");
    }

    private ClientHttpResponse buildResponse(int status, String message) {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() {
                return HttpStatus.valueOf(status);
            }

            @Override
            public int getRawStatusCode() {
                return status;
            }

            @Override
            public String getStatusText() {
                return HttpStatus.valueOf(status).getReasonPhrase();
            }

            @Override
            public InputStream getBody() {
                // 返回 JSON 格式的降级消息
                String json = String.format(
                    "{\"code\":%d,\"message\":\"%s\"}", status, message);
                return new ByteArrayInputStream(
                    json.getBytes(StandardCharsets.UTF_8));
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }

            @Override
            public void close() {}
        };
    }
}
```

## 5.3 配置降级响应内容类型

```yaml
zuul:
  routes:
    user-service:
      path: /user/**
      serviceId: user-service

# Hystrix 降级策略
hystrix:
  command:
    user-service:
      fallback:
        enabled: true  # 启用降级（默认 true）
```

## 5.4 降级触发场景

降级（Fallback）在以下场景被触发：

1. **熔断器打开**（CIRCUIT_OPEN）：后端连续失败，熔断器跳闸
2. **线程池/信号量满**（REJECTED）：并发数达到上限
3. **请求超时**（TIMEOUT）：后端响应超过 Hystrix 超时时间
4. **执行异常**（FAILURE）：`run()` 方法抛出异常
5. **短路**（SHORT_CIRCUITED）：请求被熔断器直接拒绝

# 6. 线程池配置

## 6.1 线程池隔离策略

```yaml
zuul:
  ribbon-isolation-strategy: THREAD
  thread-pool:
    use-separate-thread-pools: true  # 每个服务使用独立线程池
    thread-pool-key-prefix: zuul-    # 线程池名前缀

hystrix:
  threadpool:
    default:
      coreSize: 10                   # 核心线程数
      maximumSize: 20                # 最大线程数
      maxQueueSize: 100              # 等待队列大小
      queueSizeRejectionThreshold: 50  # 队列拒绝阈值

    # user-service 独占线程池
    user-service:
      coreSize: 20
      maximumSize: 40
      maxQueueSize: 200
```

## 6.2 线程隔离模式下的 RequestContext 传递

由于 Hystrix 线程隔离会导致 `ThreadLocal` 上下文丢失，Spring Cloud Zuul 通过 `HystrixRequestContext` 实现了上下文的传递：

```java
// RibbonCommand 内部
public HystrixCommand<ClientHttpResponse> create(SetRequestContext requestContext) {
    // 1. 从当前线程的 RequestContext 中提取必要信息
    String serviceId = requestContext.getServiceId();
    String verb = requestContext.getVerb();
    String uri = requestContext.getUri();
    // ... 其他参数

    // 2. Hystrix 在线程池线程中执行，但参数已通过闭包传递
    return new HystrixCommand<ClientHttpResponse>(setter) {
        @Override
        protected ClientHttpResponse run() {
            // 在线程池线程中执行时，serviceId/verb/uri 等参数
            // 已通过构造方法闭包传入，不依赖 ThreadLocal
            return forward(serviceId, verb, uri, ...);
        }
    };
}
```

Spring Cloud Zuul 在 `RibbonCommandFactory` 中将 `RequestContext` 的关键数据提取为构造参数，从而绕过了 ThreadLocal 的问题。

# 7. Hystrix 配置全览

```yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: THREAD          # 隔离策略
          thread:
            timeoutInMilliseconds: 10000  # 超时时间
            interruptOnTimeout: true      # 超时中断
            interruptOnCancel: false      # 取消中断
          semaphore:
            maxConcurrentRequests: 100    # 最大并发（信号量模式）
      circuitBreaker:
        enabled: true
        requestVolumeThreshold: 20        # 熔断最小请求数
        errorThresholdPercentage: 50      # 熔断错误率
        sleepWindowInMilliseconds: 5000   # 休眠窗口
      fallback:
        enabled: true
      metrics:
        rollingStats:
          timeInMilliseconds: 10000       # 滚动窗口大小
          numBuckets: 10                  # 滚动窗口桶数
        healthSnapshotIntervalInMilliseconds: 500

    # 针对特定服务
    user-service:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000

  threadpool:
    default:
      coreSize: 10
      maximumSize: 20
      allowMaximumSizeToDivergeFromCoreSize: true
      maxQueueSize: 100
      queueSizeRejectionThreshold: 50
      keepAliveTimeMinutes: 1

    user-service:
      coreSize: 20
      maximumSize: 50

# 全局 Hystrix 线程池属性
hystrix:
  threadpool:
    default:
      coreSize: 10
```

# 8. 配合 Sentinel 使用

由于 Hystrix 已停更，推荐在新项目中配合**Sentinel**使用：

```java
@Component
public class SentinelDegradeFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 0; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        String serviceId = (String) ctx.get("serviceId");
        String resourceName = "zuul_" + serviceId;

        try (Entry entry = SphU.entry(resourceName)) {
            // Sentinel 检查通过，继续执行
            return null;
        } catch (BlockException e) {
            // 被限流或降级
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(429);
            ctx.setResponseBody("{\"code\":429,\"message\":\"请求过于频繁\"}");
            return null;
        }
    }
}
```

# 9. Hystrix 监控端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

访问 `http://localhost:8080/actuator/hystrix.stream` 可以获取 Hystrix 的实时指标数据（SSE 格式的文本流）。配合 Hystrix Dashboard 可可视化监控熔断器状态。

Hystrix Dashboard 访问示例：

```bash
# 启动 Hystrix Dashboard（独立服务）
java -jar hystrix-dashboard.jar

# 在 Dashboard 界面输入监控流地址
http://zuul-gateway:8080/actuator/hystrix.stream
```
