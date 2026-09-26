
# 1. Filter 机制概述

Filter 是 Dubbo 的**拦截器链**，在 RPC 调用的各个环节插入横切逻辑。Filter 基于 SPI 机制，通过装饰器模式层层包裹 Invoker。

```java
@SPI
public interface Filter {
    Result invoke(Invoker<?> invoker, Invocation invocation) 
        throws RpcException;

    // Listener 回调（Filter 生命周期）
    default void onResponse(Result result, Invoker<?> invoker, 
            Invocation invocation) {}
    
    default void onError(Throwable t, Invoker<?> invoker, 
            Invocation invocation) {}
}
```

# 2. Filter 链构建

```java
public class ProtocolFilterWrapper implements Protocol {

    private static <T> Invoker<T> buildInvokerChain(
            final Invoker<T> invoker, String key, String group) {
        
        Invoker<T> last = invoker;
        
        // 获取所有激活的 Filter（按 @Activate 排序）
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class)
            .getActivateExtension(invoker.getUrl(), key, group);

        if (!filters.isEmpty()) {
            // 反向构建 Filter 链（装饰器模式）
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {
                    @Override
                    public Result invoke(Invocation invocation) {
                        Result asyncResult;
                        try {
                            asyncResult = filter.invoke(next, invocation);
                        } catch (Exception e) {
                            filter.onError(e, invoker, invocation);
                            throw e;
                        }
                        // 异步响应回调
                        if (asyncResult instanceof AsyncRpcResult) {
                            asyncResult.whenCompleteWithContext((r, t) -> {
                                if (t != null) {
                                    filter.onError(t, invoker, invocation);
                                } else {
                                    filter.onResponse(r, invoker, invocation);
                                }
                            });
                        } else {
                            filter.onResponse(asyncResult, invoker, invocation);
                        }
                        return asyncResult;
                    }
                    // ... 其他方法委托给 next
                };
            }
        }
        return last;
    }
}
```

**构建示意**：

```
原始 Invoker
    │
    ▼
Filter3.invoke(Invoker next, Invocation inv) {
    // 前置逻辑
    Result r = next.invoke(inv);
    // 后置逻辑
    return r;
}
    │
    ▼
Filter2.invoke(Invoker next, Invocation inv) { ... }
    │
    ▼
Filter1.invoke(Invoker next, Invocation inv) { ... }
    │
    ▼
最终 Invoker (DubboInvoker / AbstractProxyInvoker)
```

# 3. @Activate —— 条件激活

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface Activate {
    String[] group() default {};          // consumer / provider
    String[] value() default {};          // URL 参数 key 匹配条件
    int order() default 0;                // 排序（越小越靠前）
    String[] before() default {};          // 排在指定 Filter 之前
    String[] after() default {};           // 排在指定 Filter 之后
}
```

**示例**：

```java
@Activate(group = {PROVIDER, CONSUMER}, order = -10000)
public class MonitorFilter implements Filter {
    // 在 Provider 和 Consumer 两端都激活，优先级最高（先执行监控记录）
}

@Activate(group = {PROVIDER}, order = -8000, 
    value = {"validation", "true"})  // 仅当 URL 中 validation=true 时激活
public class ValidationFilter implements Filter {
    // 参数校验 Filter
}
```

# 4. 内置 Filter 详解

## 4.1 Consumer 端 Filter

| Filter | order | 说明 |
|--------|-------|------|
| `MonitorFilter` | -10000 | 记录调用次数、耗时等监控数据 |
| `ActiveLimitFilter` | 0 | 并发限制（actives 参数） |
| `RpcContextFilter` | 0 | 设置 RpcContext 上下文信息 |
| `GenericImplFilter` | 20000 | 泛化调用结果处理 |
| `ConsumerContextFilter` | -1800 | Consumer 端上下文透传 |
| `ConsumerInvokerTraceFilter` | 100000 | 调用链追踪 |
| `FutureFilter` | -20000 | 异步调用结果处理 |

## 4.2 Provider 端 Filter

| Filter | order | 说明 |
|--------|-------|------|
| `MonitorFilter` | -10000 | 监控统计 |
| `EchoFilter` | -11000 | 回声测试 |
| `ClassLoaderFilter` | -30000 | 设置 TCCL |
| `TokenFilter` | -9000 | Token 令牌验证 |
| `TimeoutFilter` | -2000 | 记录调用超时 |
| `ExceptionFilter` | -1 | 异常处理（包装成业务异常） |
| `ExecuteLimitFilter` | 0 | 并发执行限制（executes 参数） |
| `ContextFilter` | -2500 | 从 Attachments 恢复 RpcContext |
| `GenericFilter` | 30000 | 泛化调用处理 |
| `TpsLimitFilter` | -5000 | TPS 限流 |
| `TraceFilter` | 1000 | 追踪 Filter |

# 5. 关键 Filter 源码分析

## 5.1 ExceptionFilter（异常处理）

```java
@Activate(group = PROVIDER)
public class ExceptionFilter implements Filter {
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) 
            throws RpcException {
        Result result = invoker.invoke(invocation);
        
        if (result.hasException() && !(invoker instanceof GenericService)) {
            try {
                Throwable exception = result.getException();
                
                // JDK 内置异常（如 IllegalArgumentException）直接抛出
                if (exception instanceof RuntimeException 
                        && !(exception.getCause() instanceof RuntimeException)) {
                    throw (RuntimeException) exception;
                }
                
                // 非运行时异常 → 检查方法签名是否声明了该异常
                if (!isDeclaredException(invoker, invocation, exception)) {
                    // 未声明的受检异常 → 包装为 RuntimeException
                    return AsyncRpcResult.newDefaultAsyncResult(
                        new RuntimeException(exception.getMessage(), exception),
                        invocation);
                }
            } catch (Throwable e) {
                // 其他异常直接抛出
                return AsyncRpcResult.newDefaultAsyncResult(e, invocation);
            }
        }
        return result;
    }
}
```

## 5.2 ExecuteLimitFilter（并发限制）

```java
@Activate(group = PROVIDER, value = {"executes"}, order = -5000)
public class ExecuteLimitFilter implements Filter {
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) 
            throws RpcException {
        URL url = invoker.getUrl();
        String methodName = invocation.getMethodName();
        
        // 获取方法级最大并发数（默认 0 表示不限制）
        int max = url.getMethodParameter(methodName, "executes", 0);
        
        if (max > 0) {
            // Semaphore 信号量控制
            RpcStatus count = RpcStatus.getStatus(url, methodName);
            if (count.getActive() >= max) {
                throw new RpcException(RpcException.LIMIT_EXCEEDED_EXCEPTION,
                    "max concurrent invoke limit: " + max);
            }
        }
        
        long begin = System.currentTimeMillis();
        try {
            RpcStatus.beginCount(url, methodName); // active + 1
            return invoker.invoke(invocation);
        } finally {
            RpcStatus.endCount(url, methodName, 
                System.currentTimeMillis() - begin, true);
        }
    }
}
```

## 5.3 TokenFilter（令牌验证）

```java
@Activate(group = PROVIDER, value = {"token"})
public class TokenFilter implements Filter {
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) 
            throws RpcException {
        String token = invoker.getUrl().getParameter("token");
        if (StringUtils.isNotEmpty(token)) {
            // 消费端调用时需在 attachments 中传入相同 token
            String remoteToken = invocation.getAttachment("token");
            if (!token.equals(remoteToken)) {
                throw new RpcException("Invalid token! " + remoteToken);
            }
        }
        return invoker.invoke(invocation);
    }
}
```

## 5.4 ContextFilter（上下文恢复）

```java
@Activate(group = PROVIDER, order = -10000)
public class ContextFilter implements Filter {
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        // 从 Attachments 恢复 Consumer 端透传的 RpcContext
        Map<String, String> attachments = invocation.getAttachments();
        
        // 恢复隐式参数
        if (attachments.containsKey(TIME_COUNTDOWN_KEY)) {
            invocation.setAttachment(TIMEOUT_KEY, 
                attachments.get(TIME_COUNTDOWN_KEY));
        }
        
        // 设置本地调用信息
        RpcContext.getServiceContext().setConsumerUrl(
            attachments.get(CONSUMER_URL_KEY));
        
        return invoker.invoke(invocation);
    }
}
```

# 6. 自定义 Filter

## 6.1 实现

```java
// 自定义调用日志 Filter
@Activate(group = {PROVIDER, CONSUMER}, order = -9000)
public class InvokeLogFilter implements Filter {
    private static final Logger log = LoggerFactory.getLogger(InvokeLogFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) 
            throws RpcException {
        long start = System.currentTimeMillis();
        String service = invoker.getInterface().getName();
        String method = invocation.getMethodName();
        
        log.info("[Dubbo Invoke] {}.{} start, args: {}", 
            service, method, invocation.getArguments());
        
        try {
            Result result = invoker.invoke(invocation);
            long elapsed = System.currentTimeMillis() - start;
            
            log.info("[Dubbo Invoke] {}.{} end, cost: {}ms, result: {}", 
                service, method, elapsed, result.getValue());
            
            return result;
        } catch (RpcException e) {
            log.error("[Dubbo Invoke] {}.{} error: {}", 
                service, method, e.getMessage());
            throw e;
        }
    }
}
```

## 6.2 注册

文件 `META-INF/dubbo/org.apache.dubbo.rpc.Filter`：

```
invokeLog=com.example.filter.InvokeLogFilter
```

## 6.3 使用

```yaml
dubbo:
  provider:
    filter: invokeLog,exception
  consumer:
    filter: invokeLog,monitor
```

或注解方式：

```java
@DubboService(filter = "invokeLog")
public class UserServiceImpl implements UserService {}
```

```java
@DubboReference(filter = "invokeLog")
private UserService userService;
```

# 7. Filter 与 RpcContext 联动

```java
@Activate(group = CONSUMER)
public class ConsumerContextFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        RpcContext.getServiceContext()
            .setInvoker(invoker)
            .setInvocation(invocation);
        
        // 将 RpcContext 中的 attachments 透传到 Invocation
        Map<String, String> attachments = 
            RpcContext.getServerContext().getAttachments();
        if (attachments != null) {
            invocation.addAttachments(attachments);
        }
        
        return invoker.invoke(invocation);
    }
}
```

# 8. 全链路 Filter 链视图

```
Consumer                                 Provider
┌──────────────────────────────┐     ┌──────────────────────────────┐
│ InvokerInvocationHandler     │     │ DubboExporter                │
│        │                     │     │        │                     │
│   ┌────┴────┐                │     │   ┌────┴────┐                │
│   │ Consumer │                │     │   │ Provider │                │
│   │  Filters │                │     │   │  Filters │                │
│   └────┬────┘                │     │   └────┬────┘                │
│        │                     │     │        │                     │
│ ConsumerContextFilter        │     │ ContextFilter                │
│ MonitorFilter                │     │ MonitorFilter                │
│ ActiveLimitFilter            │     │ ExecuteLimitFilter           │
│ FutureFilter                 │     │ ExceptionFilter              │
│ InvokeLogFilter (自定义)      │     │ InvokeLogFilter (自定义)      │
│        │                     │     │        │                     │
│   ┌────┴────┐                │     │   ┌────┴────┐                │
│   │DubboInvoker              │     │   │AbstractProxyInvoker      │
│   └─────────┘                │     │   └─────────┘                │
└──────────────────────────────┘     └──────────────────────────────┘
```

# 9. Filter vs Interceptor

Dubbo 3 还支持基于 Spring 的拦截器：

```java
@Component
public class DubboServiceInterceptor implements 
        org.apache.dubbo.common.extension.Activate, Filter.Listener {
    
    // 实现原理类似 Filter，但是通过 Spring 自动发现和管理
}
```

Filter 是 Dubbo 原生机制（SPI），更底层、性能更高；Interceptor 基于 Spring 管理，更易使用。一般情况下性能要求不高时，两者可以互换。
