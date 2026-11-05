gRPC 调用失败时，客户端看到的往往只有一句 `UNAVAILABLE` 或 `UNKNOWN`。如果服务端不主动返回结构化错误，排查就只能靠猜。本篇把 `Status` 体系、错误详情、Deadline、取消和重试串起来，讲清"错误如何产生、如何传播、什么时候该重试"。

# 1. Status 模型

## 1.1 状态码

gRPC 不使用 HTTP 状态码表达业务结果，而是用 `io.grpc.Status.Code`。HTTP 层通常始终是 200，真实结果在 Trailers 的 `grpc-status` 里。

| 状态码 | 含义 | 典型原因 |
|--------|------|----------|
| `OK` | 成功 | 正常返回 |
| `INVALID_ARGUMENT` | 参数非法 | 参数校验失败、格式错误 |
| `NOT_FOUND` | 资源不存在 | 查询不到记录 |
| `ALREADY_EXISTS` | 资源已存在 | 重复创建 |
| `PERMISSION_DENIED` | 无权限 | 已认证但无权访问 |
| `UNAUTHENTICATED` | 未认证 | 缺少或无效 token |
| `RESOURCE_EXHAUSTED` | 资源耗尽 | 限流、配额、消息过大 |
| `FAILED_PRECONDITION` | 前置条件不满足 | 状态机不允许当前操作 |
| `ABORTED` | 并发冲突 | 乐观锁失败、事务冲突 |
| `OUT_OF_RANGE` | 越界 | 分页参数越界 |
| `UNIMPLEMENTED` | 方法未实现 | 服务未注册、版本不匹配 |
| `INTERNAL` | 服务端内部错误 | 未捕获异常、bug |
| `UNAVAILABLE` | 服务不可用 | 连接断开、服务未启动 |
| `DATA_LOSS` | 数据丢失 | 解码失败、存储损坏 |
| `DEADLINE_EXCEEDED` | 超时 | 调用超过 deadline |
| `CANCELLED` | 被取消 | 客户端主动取消 |

**设计考量**：状态码是跨语言的契约，不能随意自定义。业务错误应尽量映射到标准码，再用 `description` 和错误详情补充上下文。把"用户不存在"返回成 `INTERNAL`，会让客户端无法区分"业务未命中"和"服务挂了"。

## 1.2 异常类型

gRPC Java 有两个异常类：

| 异常 | 是否受检 | 典型来源 |
|------|----------|----------|
| `StatusRuntimeException` | 否 | 阻塞 stub、异步 `onError`、默认转换 |
| `StatusException` | 是 | 需要受检异常的 API |

两者都提供：

```java
Status status = e.getStatus();
Metadata trailers = e.getTrailers(); // 可能为 null
```

从任意异常还原状态：

```java
catch (Exception e) {
    Status status = Status.fromThrowable(e);
    System.err.println(status.getCode() + ": " + status.getDescription());
}
```

如果异常不是 gRPC 抛出的，`Status.fromThrowable` 会返回 `UNKNOWN`。因此看到 `UNKNOWN` 时，第一反应应该是"服务端抛了未被转换的异常"，而不是"gRPC 坏了"。

## 1.3 错误信息不是异常栈

`Status.description` 会被发送到客户端，**不要往里塞堆栈、SQL、内部地址、密钥**。堆栈应记录在服务端日志，客户端只给可读的、脱敏的错误说明。

```java
// 错误示范：泄露内部细节
throw Status.INTERNAL.withDescription(e.toString()).asRuntimeException();
// 正确做法：对外通用信息，对内保留 cause
throw Status.INTERNAL
        .withDescription("internal server error")
        .withCause(e)
        .asRuntimeException();
```

`withCause` 只在服务端进程内保留堆栈，不会完整传给客户端。

# 2. 服务端如何返回错误

## 2.1 一元方法

最直接的方式是抛 `StatusRuntimeException`：

```java
@Override
public void getUser(GetUserRequest request, StreamObserver<UserReply> responseObserver) {
    if (request.getId() <= 0) {
        throw Status.INVALID_ARGUMENT
                .withDescription("id 必须大于 0")
                .asRuntimeException();
    }
    UserReply user = store.get(request.getId());
    if (user == null) {
        throw Status.NOT_FOUND
                .withDescription("用户不存在: " + request.getId())
                .asRuntimeException();
    }
    responseObserver.onNext(user);
    responseObserver.onCompleted();
}
```

也可以显式调用 `onError`：

```java
responseObserver.onError(Status.NOT_FOUND
        .withDescription("用户不存在")
        .asRuntimeException());
```

两种写法的效果一样：gRPC 会把异常转换成 `grpc-status` 和 `grpc-message`。区别是抛异常会中断当前方法，`onError` 之后要继续写代码时必须自己 `return`。

## 2.2 流式方法

流式方法里，错误通过 `responseObserver.onError` 或请求观察者的回调传播：

```java
@Override
public void range(RangeRequest request, StreamObserver<RangeReply> responseObserver) {
    if (request.getEnd() < request.getStart()) {
        responseObserver.onError(Status.INVALID_ARGUMENT
                .withDescription("end 必须大于等于 start")
                .asRuntimeException());
        return;
    }
    for (int i = request.getStart(); i <= request.getEnd(); i++) {
        if (Context.current().isCancelled()) {
            // 客户端已经取消，不必再写
            return;
        }
        responseObserver.onNext(RangeReply.newBuilder().setValue(i).build());
    }
    responseObserver.onCompleted();
}
```

**重要**：`onNext` 之后再 `onError` 是允许的，但 `onCompleted` 之后不能再 `onError`。终态只能有一个。

## 2.3 错误详情

标准状态码只能表达"大类"，复杂的参数校验需要结构化详情。gRPC 使用 `google.rpc.Status` 加 `Any` 承载扩展信息：

```java
import com.google.protobuf.Any;
import com.google.rpc.BadRequest;
import io.grpc.Status;
import io.grpc.protobuf.StatusProto;
BadRequest badRequest = BadRequest.newBuilder()
        .addFieldViolations(BadRequest.FieldViolation.newBuilder()
                .setField("email")
                .setDescription("邮箱格式不正确")
                .build())
        .build();
com.google.rpc.Status rpcStatus = com.google.rpc.Status.newBuilder()
        .setCode(io.grpc.Status.Code.INVALID_ARGUMENT.value())
        .setMessage("参数校验失败")
        .addDetails(Any.pack(badRequest))
        .build();
responseObserver.onError(StatusProto.toStatusRuntimeException(rpcStatus));
```

需要额外依赖：


`google.rpc` 里常用的详情类型：


客户端解析：


# 3. 客户端如何处理

客户端处理错误的原则是：**先分类，再决定重试、降级还是上报**。

```java
try {
    return stub.withDeadlineAfter(2, TimeUnit.SECONDS).getUser(request);
} catch (StatusRuntimeException e) {
    switch (e.getStatus().getCode()) {
        case NOT_FOUND: return null; // 业务上可以接受"查不到"
        case INVALID_ARGUMENT: throw new IllegalArgumentException(e.getStatus().getDescription(), e);
        case UNAVAILABLE: case DEADLINE_EXCEEDED: throw new RetryableException(e);
        case UNAUTHENTICATED: throw new AuthException(e);
        default: throw new DownstreamException("调用用户服务失败", e);
    }
}
```

直接对 `ExecutionException` 调 `Status.fromThrowable` 会得到 `UNKNOWN`，这是很常见的排查误区。

# 4. 超时与截止时间

## 4.1 Deadline

gRPC 使用 **Deadline（截止时间）** 而不是简单的"读超时"。Deadline 是整条调用链共享的绝对时间点，会随着请求传播到下游。

```java
UserReply reply = stub
        .withDeadlineAfter(200, TimeUnit.MILLISECONDS) // 相对当前时间
        .getUser(request);
```

也可以使用绝对时间：


| 方式 | 语义 | 建议 |
|------|------|------|
| `withDeadlineAfter` | 从当前时刻起算 | 业务代码首选 |
| `withDeadline(Deadline)` | 使用指定绝对 deadline | 需要向下游传播时使用 |
| 不设置 | 无限等待 | 禁止在生产环境出现 |

**设计考量**：不设 deadline 的调用，一旦服务端卡住，客户端线程会永久挂起，最终拖垮连接池和线程池。所有 RPC 调用都应该有超时，包括内部调用和批处理。

## 4.2 服务端感知与传播

服务端通过 `Context.current().getDeadline()` 读取剩余时间，向下游调用时用 `withDeadline(deadline)` 继承，而不是重新给一个完整超时。长任务要检查 `Context.current().isCancelled()`，因为 deadline 到期后业务代码不会自动停止。

# 5. 取消

## 5.1 客户端主动取消

`Context` 提供了可取消的上下文：

import io.grpc.Context;
Context.CancellableContext ctx = Context.current().withCancellation();
ExecutorService executor = Executors.newSingleThreadExecutor();
executor.submit(() -> ctx.run(() -> stub.getUser(request)));
// 用户离开页面或任务被中止
ctx.cancel(new RuntimeException("user navigated away"));

`CancellableContext.cancel` 是线程安全的，可以从另一个线程调用。取消会向服务端发送 `RST_STREAM`，只终止当前这次调用，不影响同一 channel 上的其他流。

## 5.2 服务端感知取消

一元方法可以通过 `Context` 监听：


流式方法更适合用 `ServerCallStreamObserver`：

```java
ServerCallStreamObserver<RangeReply> observer =
        (ServerCallStreamObserver<RangeReply>) responseObserver;
observer.setOnCancelHandler(() -> {
    // 设置标志位，让生产循环尽快退出
    cancelled.set(true);
});
```

取消不是错误，不要把它当成 `INTERNAL` 上报。客户端取消后，服务端通常只需要记录指标并释放资源。

# 6. 重试与幂等

## 6.1 哪些错误可以重试

| 状态码 | 是否可重试 | 说明 |
|--------|-----------|------|
| `UNAVAILABLE` | 通常可以 | 连接断开、实例重启 |
| `DEADLINE_EXCEEDED` | 谨慎 | 服务端可能已经执行，取决于幂等性 |
| `RESOURCE_EXHAUSTED` | 谨慎 | 限流可退避重试，配额不足不应重试 |
| `ABORTED` | 可以 | 并发冲突，重试可能成功 |
| `INVALID_ARGUMENT` | 不可以 | 参数错误，重试无用 |
| `NOT_FOUND` | 不可以 | 资源不存在 |
| `PERMISSION_DENIED` | 不可以 | 权限问题 |
| `UNAUTHENTICATED` | 刷新凭据后一次 | 不要无限重试 |

**核心原则**：只有**幂等**操作才适合自动重试。查询、删除（按 id）通常幂等；创建、扣款、发消息通常不幂等，除非有幂等键。

## 6.2 gRPC Java 的重试配置

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import java.util.List;
import java.util.Map;
Map<String, Object> policy = Map.of(
        "maxAttempts", 4.0,
        "initialBackoff", "0.1s",
        "maxBackoff", "1s",
        "backoffMultiplier", 2.0,
        "retryableStatusCodes", List.of("UNAVAILABLE", "DEADLINE_EXCEEDED"));
Map<String, Object> methodConfig = Map.of(
        "name", List.of(Map.of("service", "user.v1.UserService", "method", "GetUser")),
        "retryPolicy", policy);
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 50051)
        .usePlaintext().enableRetry().maxRetryAttempts(4)
        .defaultServiceConfig(Map.of("methodConfig", List.of(methodConfig)))
        .build();

`maxAttempts` 包含首次调用，所以 `4` 表示最多再重试 3 次。退避采用指数增长并带随机抖动，避免大量客户端在同一时刻重试造成"重试风暴"。

**注意**：gRPC 的重试只适用于一元调用和"尚未收到任何消息"的流式调用。对于已经收到部分消息的流，框架无法自动续传，需要业务层自己设计断点恢复。

## 6.3 幂等键

创建类操作如果要支持重试，必须让服务端能够识别重复请求：

```java
import io.grpc.Metadata;
import io.grpc.stub.MetadataUtils;
// 客户端：同一个业务请求复用同一个 key
Metadata headers = new Metadata();
headers.put(Metadata.Key.of("idempotency-key", Metadata.ASCII_STRING_MARSHALLER),
        orderRequestId);
stub.withInterceptors(MetadataUtils.newAttachHeadersInterceptor(headers))
        .createOrder(request);
```

服务端用 `idempotency-key` 做去重：

```text
收到请求
  │
  ├── key 已存在且已完成 → 直接返回上次结果
  ├── key 已存在且处理中 → 返回 ABORTED 或让客户端轮询
  └── key 不存在 → 执行业务，记录 key 与结果
```

去重表要设置 TTL，并保证"记录 key"与"业务写入"在同一个事务或原子操作里，否则仍可能重复执行。

## 6.4 重试的边界

| 场景 | 建议 |
|------|------|
| 查询用户信息 | 可自动重试 `UNAVAILABLE` |
| 创建订单 | 必须带幂等键才可重试 |
| 扣减库存 | 用幂等键或业务去重，不能裸重试 |
| 流式上传 | 记录已上传偏移，断点续传 |
| 支付回调 | 以交易号去重，重试必须安全 |

> **行业实践**：重试应该放在"最了解业务语义"的那一层。底层 SDK 无脑重试会把非幂等操作放大成重复写入；上层业务根据幂等性、用户可见性和 SLA 决定是否重试，才是安全的。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `UNKNOWN: java.lang.RuntimeException` | 业务抛了未转换的异常 | 转成 `Status` 或加恢复拦截器 |
| `UNKNOWN` 出现在 `Future.get()` | 没解包 `ExecutionException` | 用 `e.getCause()` 再取 Status |
| `DEADLINE_EXCEEDED` 但服务端仍在跑 | 服务端没检查取消 | 长任务检查 `Context.isCancelled()` |
| 重试导致重复订单 | 对非幂等操作开了自动重试 | 加幂等键或关闭重试 |
| 重试风暴 | 固定间隔、无抖动 | 指数退避 + 抖动 + 重试预算 |
| `UNAUTHENTICATED` 无限重试 | token 过期后没刷新 | 刷新一次，失败即终止 |
| 客户端收不到错误详情 | 服务端只设了 description | 用 `google.rpc.Status` + `Any` |
| `onError` 后再次 `onNext` | 终态后继续写 | 出错立即 `return` |

# 8. 练习与自检

1. 实现一个参数校验失败的服务端方法，返回 `INVALID_ARGUMENT` 加 `BadRequest` 详情，客户端解析出具体字段。
2. 给客户端设置 100ms deadline，服务端 `Thread.sleep(1s)`，观察客户端状态码，并验证服务端是否能感知取消。
3. 配置一个只对 `GetUser` 生效的重试策略，重试 `UNAVAILABLE`，用两次启动/停止服务端验证重试次数与退避时间。

# 9. 小结

| 主题 | 要点 |
|------|------|
| Status | 标准状态码是跨语言契约，不要自定义 |
| 异常 | 阻塞用 `StatusRuntimeException`，Future 要解包 `ExecutionException` |
| 详情 | `google.rpc.Status` + `Any` 表达字段级错误 |
| Deadline | 所有调用都要设，且要向下游传播剩余时间 |
| 取消 | `CancellableContext`、`setOnCancelHandler`，长任务主动检查 |
| 重试 | 只重试幂等操作，指数退避，避免风暴 |
| 幂等 | 创建类操作靠 `idempotency-key` 去重 |

下一篇进入 TLS 与认证，讲清服务端证书、mTLS、Spring SSL bundle、JWT 拦截器，以及常见的握手错误。
