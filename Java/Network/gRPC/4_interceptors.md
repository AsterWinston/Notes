gRPC 拦截器就是 gRPC 版的中间件：鉴权、日志、链路追踪、panic 恢复、限流这些横切逻辑都靠它。它的难点不在 API，而在**链式顺序**和**流式接口的特殊性**。顺序写反，日志会记错耗时；只写 Unary 拦截器，流式接口就完全裸奔。

# 1. 拦截器解决什么问题

一次 RPC 调用中，总有一些逻辑与具体业务无关，却要在每个方法上重复：

- 从 Metadata 中取 token 并校验身份。
- 记录方法名、耗时、状态码。
- 生成或透传 trace id。
- 捕获业务异常，转换成统一的 `Status`。
- 统计 QPS、并发数、错误率。

如果把这些代码散落在每个 service 方法里，重复且容易漏。拦截器把这些横切关注点抽出来，在调用进入业务实现之前、返回客户端之前插入统一处理。

# 2. 两套接口

gRPC Java 的拦截器分为客户端与服务端两套，接口签名不同，但思想一致：拿到"下一层"的调用入口，决定是否继续、是否修改参数、是否提前返回。

## 2.1 ClientInterceptor

```java
public interface ClientInterceptor {
    <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next);
}
```

三个参数分别是：

- `method`：方法描述符，包含全限定方法名、请求/响应类型、序列化器。
- `callOptions`：本次调用的选项，如 deadline、压缩、凭据。
- `next`：下一层 Channel。调用 `next.newCall(method, callOptions)` 才会真正发出请求。

返回值是一个 `ClientCall`。常见写法是继承 `ForwardingClientCall.SimpleForwardingClientCall`，只覆写需要干预的方法：

```java
import io.grpc.*;
public class AuthClientInterceptor implements ClientInterceptor {
    private static final Metadata.Key<String> AUTHORIZATION =
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    private final String token;
    public AuthClientInterceptor(String token) {
        this.token = token;
    }
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {
        return new ForwardingClientCall.SimpleForwardingClientCall<>(
                next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                // Metadata 的 key 会被 gRPC 规范化为小写
                headers.put(AUTHORIZATION, "Bearer " + token);
                super.start(responseListener, headers);
            }
        };
    }
}
```

## 2.2 ServerInterceptor

```java
public interface ServerInterceptor {
    <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next);
}
```

`call` 代表当前这次服务端调用，`headers` 是客户端传来的 Metadata，`next` 是真正的业务处理器。返回 `ServerCall.Listener` 后，gRPC 会把消息、半关闭、取消等事件回调给它。

## 2.3 流式接口为什么用同一套

服务端拦截器不区分 Unary 和 Streaming：`interceptCall` 对两种调用都会触发。区别在于业务处理器内部如何处理多个消息。因此服务端只需要一套拦截器，就能覆盖全部方法。客户端也一样，`ClientInterceptor` 对四种调用都生效。

**注意**：有些框架或教程会提供单独的"流式拦截器"概念，但在 gRPC Java 原生 API 里，只有 `ClientInterceptor` 与 `ServerInterceptor` 两个接口。流式的特殊处理体现在 `ServerCall.Listener` 的多次 `onMessage` 和 `onHalfClose` 上。

# 3. 拦截器顺序

## 3.1 推荐顺序

理想顺序是：

1. **Recovery** 放在最外层，才能兜住内层所有异常。
2. **Logging** 放在 Recovery 内层，才能记录到真实业务耗时；如果放在外层，异常请求可能记不到。
3. **Auth** 放在靠近业务的位置，先确认请求合法，再进入业务。
4. **限流/指标** 根据需求放在 Auth 外或内。

## 3.2 Java 的顺序陷阱

gRPC Java 的注册方法有两个：`intercept` 与 `interceptForward`。它们的顺序语义**相反**：

| 方法 | 顺序 |
|------|------|
| `ClientInterceptors.intercept(channel, a, b, c)` | c 最先被调用，a 最靠近真实 Channel |
| `ClientInterceptors.interceptForward(channel, a, b, c)` | a 最先被调用，c 最靠近真实 Channel |
| `ServerInterceptors.intercept(service, a, b, c)` | c 最先被调用，a 最靠近业务 |
| `ServerInterceptors.interceptForward(service, a, b, c)` | a 最先被调用，c 最靠近业务 |

也就是说，`intercept` 是"列表倒序执行"，`interceptForward` 是"列表正序执行"。很多 Go 转 Java 的开发者会按 Go 的直觉写 `intercept`，结果顺序完全反了。

推荐统一使用 `interceptForward`，让代码书写顺序与执行顺序一致：

```java
ServerServiceDefinition intercepted = ServerInterceptors.interceptForward(
        new UserServiceImpl().bindService(),
        new RecoveryServerInterceptor(),   // 最外层，最先执行
        new LoggingServerInterceptor(),    // 第二层
        new AuthServerInterceptor());      // 最内层
Server server = ServerBuilder.forPort(50051)
        .addService(intercepted)
        .build()
        .start();
```

客户端同理：

```java
Channel intercepted = ClientInterceptors.interceptForward(
        channel,
        new LoggingClientInterceptor(),       // 最外层，记录总耗时
        new AuthClientInterceptor(token),     // 注入 token
        new MetricsClientInterceptor());      // 最内层，统计真实请求
```

**设计考量**：`intercept` 的倒序语义来自内部包装实现，属于历史包袱。新代码没有理由使用它，统一 `interceptForward` 能减少认知负担。如果使用 Spring gRPC，全局拦截器按 `@Order` 排序，不需要关心这个陷阱。

# 4. 实战：日志拦截器

## 4.1 服务端日志

```java
import io.grpc.*;
import java.util.concurrent.TimeUnit;
public class LoggingServerInterceptor implements ServerInterceptor {
    private static final Metadata.Key<String> TRACE_ID =
            Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER);
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        long start = System.nanoTime();
        String traceId = headers.get(TRACE_ID);
        if (traceId == null || traceId.isBlank()) {
            traceId = java.util.UUID.randomUUID().toString();
        }
        String method = call.getMethodDescriptor().getFullMethodName();
        String finalTraceId = traceId;
        // 包装 ServerCall，在 close 时能拿到真实状态码
        ServerCall<ReqT, RespT> wrappedCall =
                new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
                    @Override
                    public void close(Status status, Metadata trailers) {
                        long costMs = TimeUnit.NANOSECONDS.toMillis(
                                System.nanoTime() - start);
                        System.out.printf("trace=%s method=%s code=%s cost=%dms%n",
                                finalTraceId, method, status.getCode(), costMs);
                        super.close(status, trailers);
                    }
                };
        return next.startCall(wrappedCall, headers);
    }
}
```

只包装 Listener 也能记录耗时，但 `onComplete` 只表示流正常结束，不代表业务成功。要拿到 `INVALID_ARGUMENT`、`NOT_FOUND` 这类真实状态码，必须包装 `ServerCall.close`。

## 4.2 客户端日志

客户端日志拦截器同样包装 `ClientCall.start`，在 `ForwardingClientCallListener.onClose(Status, Metadata)` 里记录方法名、状态码和耗时。写法与服务端对称，不再重复完整代码。
# 5. 实战：鉴权拦截器

## 5.1 服务端校验 token

```java
public class AuthServerInterceptor implements ServerInterceptor {
    public static final Context.Key<String> USER_ID = Context.key("user-id");
    private static final Metadata.Key<String> AUTHORIZATION =
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    private final TokenVerifier verifier;
    public AuthServerInterceptor(TokenVerifier verifier) {
        this.verifier = verifier;
    }
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        String header = headers.get(AUTHORIZATION);
        if (header == null || !header.startsWith("Bearer ")) {
            return reject(call, "缺少 Bearer token");
        }
        String userId;
        try {
            userId = verifier.verify(header.substring("Bearer ".length()));
        } catch (Exception e) {
            return reject(call, "token 无效或已过期");
        }
        // 把身份放进 Context，业务层通过 USER_ID.get() 读取
        Context ctx = Context.current().withValue(USER_ID, userId);
        return Contexts.interceptCall(ctx, call, headers, next);
    }
    private <ReqT, RespT> ServerCall.Listener<ReqT> reject(
            ServerCall<ReqT, RespT> call, String message) {
        call.close(Status.UNAUTHENTICATED.withDescription(message), new Metadata());
        return new ServerCall.Listener<>() {}; // 空 Listener，不再进入业务
    }
}
```

业务层读取身份：

```java
String userId = AuthServerInterceptor.USER_ID.get();
if (userId == null) {
    throw Status.UNAUTHENTICATED.withDescription("未认证").asRuntimeException();
}
```

**注意**：`Context` 只在当前进程内有效，不会自动跨网络传递。跨进程传递身份靠 Metadata；服务端把 Metadata 解析成 Context 后，下游代码才能直接读取。

## 5.2 客户端注入 token

客户端拦截器已经在第 2 节展示。生产环境不要每次请求都去远端拉 token，应在内存中缓存，并在过期前刷新。token 也不要写进日志。

# 6. 异常与恢复

## 6.1 把业务异常转成 Status

最稳妥的做法是在业务方法里显式抛出 `StatusRuntimeException`：

```java
if (request.getId() <= 0) {
    throw Status.INVALID_ARGUMENT
            .withDescription("id 必须大于 0")
            .asRuntimeException();
}
```

如果业务代码使用了自定义异常，拦截器可以统一转换：

```java
try {
    return next.startCall(call, headers);
} catch (IllegalArgumentException e) {
    call.close(Status.INVALID_ARGUMENT.withDescription(e.getMessage()), new Metadata());
    return new ServerCall.Listener<>() {};
}
```

但 `interceptCall` 只能捕获"开始调用"阶段的异常。业务方法在 `onHalfClose` 里执行，此时抛出的异常不会回到 `interceptCall` 的 try 块，需要包装 Listener。

## 6.2 恢复拦截器

```java
public class RecoveryServerInterceptor implements ServerInterceptor {
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call, Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        ServerCall.Listener<ReqT> delegate = next.startCall(call, headers);
        return new ServerCall.Listener<ReqT>() {
            @Override public void onMessage(ReqT m) { try { delegate.onMessage(m); } catch (RuntimeException e) { fail(e); } }
            @Override public void onHalfClose() { try { delegate.onHalfClose(); } catch (RuntimeException e) { fail(e); } }
            @Override public void onCancel() { delegate.onCancel(); }
            @Override public void onComplete() { delegate.onComplete(); }
            @Override public void onReady() { delegate.onReady(); }
            private void fail(RuntimeException e) {
                call.close(Status.INTERNAL.withDescription("internal server error")
                        .withCause(e), new Metadata());
            }
        };
    }
}
```

**重要**：gRPC 服务端不会自动把业务 panic 转成错误响应。没有恢复拦截器，一次空指针可能导致连接被重置，客户端只看到 `UNAVAILABLE`，服务端日志却可能什么都没有。生产环境必须安装恢复逻辑。

# 7. Context 传递

`io.grpc.Context` 是 gRPC 的请求上下文，类似 `ThreadLocal`，但能在异步回调之间传递。它承载 deadline、取消信号和自定义值。

```java
// 定义 key：用 public static final，业务层直接引用
public static final Context.Key<String> TENANT_ID = Context.key("tenant-id");
// 服务端写入
Context ctx = Context.current().withValue(TENANT_ID, "acme");
return Contexts.interceptCall(ctx, call, headers, next);
// 业务层读取
String tenant = TENANT_ID.get();
```

`Context` 的几个关键点：

- `Context.current()` 取当前上下文；没有则返回 ROOT。
- `withValue` 返回新 Context，不修改原对象。
- `Contexts.interceptCall` 把 Context 绑定到本次调用的生命周期，调用结束后自动恢复。
- 跨进程只传 Metadata，不传 Context 对象。

**常见误区**：在客户端拦截器里用 `Context.current().withValue(...)` 并不会把值发给服务端。客户端要把值写进 `Metadata`，服务端再从 `Metadata` 读出来放进 Context。

# 8. 流式拦截器的特殊点

流式调用与一元调用共用同一套拦截器，但 Listener 会被多次回调：

```text
Unary:   onMessage → onHalfClose → onComplete
Stream:  onMessage → onMessage → ... → onHalfClose → onComplete
```

因此日志拦截器不能假设 `onMessage` 只来一次。统计消息数时，在 `onMessage` 里累加；统计流的结束状态时，在 `onComplete` 或包装的 `close` 里记录。

另外，`onCancel` 可能在任何时刻触发，业务代码要保证取消后不再 `onNext`。服务端可以通过 `ServerCallStreamObserver.setOnCancelHandler` 设置标志位，避免继续生产数据。

# 9. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 拦截器顺序与预期相反 | 用了 `intercept` 的倒序语义 | 改用 `interceptForward` |
| 日志耗时偏大 | 日志放在 Recovery 外层，包含异常处理时间 | 调整顺序，Logging 放 Recovery 内层 |
| 业务取不到用户身份 | Context 没有向下传递 | 用 `Contexts.interceptCall` 返回新 Context |
| 客户端设置了 Context 但服务端收不到 | Context 不会跨网络 | 写入 Metadata，服务端再解析 |
| token 字段取不到 | key 大小写或名称不一致 | 统一小写，使用同一个 `Metadata.Key` |
| 流式接口没鉴权 | 只包了一元方法 | 服务端拦截器本身覆盖流式，检查是否漏注册 |
| panic 后客户端只看到 `UNAVAILABLE` | 没有恢复拦截器 | 加 Recovery 并放最外层 |
| `onError` 后仍写响应 | 业务没 return | 出错后立即返回 |

# 10. 练习与自检

1. 写一个服务端拦截器，统计每个方法的调用次数、失败次数和 P99 耗时，用 `ConcurrentHashMap` 加简单直方图实现。
2. 用 `intercept` 和 `interceptForward` 分别注册 Logging 与 Auth，打印执行顺序，验证两者的差异。
3. 在流式方法中让业务代码抛异常，确认 Recovery 拦截器能返回 `INTERNAL`，并且客户端不会一直挂起。

# 11. 小结

| 主题 | 要点 |
|------|------|
| 两套接口 | `ClientInterceptor` 与 `ServerInterceptor`，都覆盖四种调用 |
| 顺序 | `intercept` 倒序，`interceptForward` 正序；推荐后者 |
| 服务端 | 鉴权、日志、恢复、限流 |
| 客户端 | 注入 token、超时、埋点、重试 |
| Context | 进程内传递，跨进程必须走 Metadata |
| 恢复 | 必须包装 Listener 才能接住业务异常 |
| 流式 | 多次 `onMessage`，取消可能随时发生 |

下一篇进入 Spring gRPC，讲清服务如何注册为 Bean、客户端如何注入、配置项怎么写，以及 gRPC 服务与 Spring Boot 生命周期的配合。
