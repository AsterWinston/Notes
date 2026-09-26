
# 1. Trace 数据模型

```
Trace (一次请求的完整生命周期)
  │
  ├── TraceId: 全局唯一标识
  │     └── 格式: {serviceInstance}.{timestamp}.{threadId}
  │
  ├── Segment (单个 JVM 内的 Trace 片段)
  │     ├── SegmentId
  │     └── ServiceInstance
  │
  └── Span (一次操作)
        ├── SpanId: 0, 1, 2, ...
        ├── ParentSpanId: -1 (root), 0, 1, ...
        ├── SpanType: Entry / Exit / Local
        ├── StartTime / EndTime
        ├── ComponentId (SpringMVC/Dubbo/JDBC...)
        ├── Tags (URL, HTTP_METHOD, db.type...)
        ├── Logs (error stacktrace)
        └── Refs (跨进程 Parent TraceSegment 引用)
```

# 2. 单服务内追踪

```
一个 HTTP 请求在 Spring Boot 应用内的 Span 树：

Entry Span (SpanId=0, ParentSpanId=-1)
  GET:/api/user/{id}
  │
  ├── Local Span (SpanId=1, ParentSpanId=0)
  │     UserController.getUser()
  │
  ├── Local Span (SpanId=2, ParentSpanId=0)
  │     UserService.getById()
  │
  └── Exit Span (SpanId=3, ParentSpanId=0)
        JDBC/PreparedStatement/executeQuery()
```

# 3. 跨进程 Trace 传播

```
Service A (HTTP Client)                    Service B (HTTP Server)
        │                                         │
        │ 创建 Exit Span                          │
        │   SpanId = 2                            │
        │                                         │
        │ 注入 Trace 上下文到 Header                │
        │   sw8: 1-MyApp...-0-1-2-3-...           │
        │ ──────────────────────────────────→     │
        │                                         │ 提取 Trace 上下文
        │                                         │ 创建 Entry Span
        │                                         │   SpanId = 0
        │                                         │   ParentRef = ServiceA.Span2
        │                                         │
        │ ←──────────── Response ───────────────  │
        │                                         │
        │ 结束 Exit Span                          │ 结束 Entry Span
```

## 3.1 sw8 协议头

```
sw8: 1-MyAppA..1-MyAppB..1-/api/user-0-...-...

格式: 1-TRACEID-SEGMENTID-SPANID-SERVICE-INSTANCE-ENDPOINT-PEER

示例:
  sw8: 1-MyAppA.abc.123-abc.124-2-
       MyAppB.MyAppB-instance-/api/user-...
```

# 4. ContextManager — 上下文管理

```java
public class ContextManager {

    // ThreadLocal 存储当前活跃 Span
    private static final ThreadLocal<AbstractTracerContext> CONTEXT =
        new ThreadLocal<>();

    // 创建 Entry Span
    public static AbstractSpan createEntrySpan(
            String operationName, ContextCarrier carrier) {
        AbstractSpan span = getOrCreate().createEntrySpan(operationName);
        if (carrier != null) {
            // 从上游提取 Trace 上下文
            span.ref(carrier);
        }
        return span;
    }

    // 创建 Exit Span
    public static AbstractSpan createExitSpan(
            String operationName, ContextCarrier carrier, String remotePeer) {
        AbstractSpan span = CONTEXT.get().createExitSpan(
            operationName, remotePeer);
        // 将上下文注入 carrier（传播给下游）
        span.inject(carrier);
        return span;
    }

    // 停止当前 Span
    public static void stopSpan() {
        CONTEXT.get().stopSpan();
    }
}
```

# 5. Span 生命周期

```
1. createEntrySpan/ExitSpan/LocalSpan → Span 入栈
2. 执行业务逻辑
3. stopSpan() → Span 出栈，计算耗时
4. Segment 关闭时 → 序列化 → 异步发送到 OAP
```

**Span 栈**：同线程内 Span 形成栈结构，`stopSpan()` 按后进先出顺序退栈。

# 6. Tags 与 Logs

```java
// 自定义 Span 标签
@Trace(operationName = "customOperation")
@Tag(key = "userId", value = "arg[0]")
@Tag(key = "orderAmount", value = "returnedObj.amount")
public Order createOrder(Long userId) { ... }

// 编程方式
ActiveSpan.tag("custom_key", "custom_value");
ActiveSpan.error();           // 标记为错误
ActiveSpan.log(new Exception("error"));  // 记录异常
ActiveSpan.info("processing step 1");    // 记录信息
```

# 7. 异步线程追踪

```java
// 线程池追踪
@Async
public CompletableFuture<User> getUserAsync(Long id) {
    // SkyWalking 自动传播 Trace 上下文到异步线程
    return CompletableFuture.completedFuture(userService.getById(id));
}

// 手动跨线程传播
Runnable wrappedRunnable = ContextManager.wrap(() -> {
    // 此 Runnable 自动携带父线程的 Trace 上下文
    userService.getById(id);
});
executorService.submit(wrappedRunnable);
```

# 8. 采样策略

```properties
# agent.config
agent.sample_n_per_3_secs=1000    # 每 3 秒最多 1000 条

# 强制采样（所有 Trace 都记录，开发环境）
agent.force_tracing=true

# 忽略指定端点
trace.ignore_path=/health,/metrics,/actuator/**
```
