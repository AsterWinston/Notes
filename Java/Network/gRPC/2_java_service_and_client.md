上一篇把 proto 与代码生成跑通了，但生成代码只是零件。本篇把零件装成一台能运转的机器：定义服务、实现服务端、用三种 stub 发起调用，并处理连接与停机这些工程问题。

# 1. 从 proto 到工程骨架

## 1.1 proto 定义

先准备一份用户查询服务，文件放在 `src/main/proto/user.proto`：

```protobuf
syntax = "proto3";
package user.v1;
option java_package = "com.example.user.v1";
option java_multiple_files = true;
option java_outer_classname = "UserProto";
service UserService {
  rpc GetUser (GetUserRequest) returns (UserReply);
  rpc CreateUser (CreateUserRequest) returns (UserReply);
}
message GetUserRequest {
  int64 id = 1;
}
message CreateUserRequest {
  string name = 1;
  string email = 2;
}
message UserReply {
  int64 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at_millis = 4;
}
```

`package user.v1` 决定了全限定服务名 `user.v1.UserService`，最终映射到 HTTP/2 路径 `/user.v1.UserService/GetUser`。客户端和服务端的包名必须一致，否则会得到 `UNIMPLEMENTED`。

## 1.2 模块划分

单仓库里通常拆成三个模块，避免客户端被迫依赖服务端实现：

```text
grpc-user-demo/
├── user-proto/                 # 只放 .proto 和生成代码
│   └── src/main/proto/user.proto
├── user-server/                # 依赖 user-proto，实现并启动服务
│   └── src/main/java/.../UserServiceImpl.java
└── user-client/                # 依赖 user-proto，只调用
    └── src/main/java/.../UserClient.java
```

**设计考量**：把 proto 单独做成一个 jar，客户端就不需要拉入服务端的 Spring、数据库、业务依赖。契约模块越薄，跨团队协作的耦合越小。很多团队会把 proto 发布到内部 Maven 仓库，客户端直接引用版本号。

# 2. 服务端实现

## 2.1 继承 ImplBase

生成代码里的 `UserServiceGrpc.UserServiceImplBase` 是服务端入口。即使是一元方法，签名也是 `void` 加 `StreamObserver`：

```java
package com.example.user;
import com.example.user.v1.GetUserRequest;
import com.example.user.v1.UserReply;
import com.example.user.v1.UserServiceGrpc;
import io.grpc.stub.StreamObserver;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    private final Map<Long, UserReply> store = new ConcurrentHashMap<>();
    @Override
    public void getUser(GetUserRequest request, StreamObserver<UserReply> responseObserver) {
        UserReply user = store.get(request.getId());
        if (user == null) {
            // 业务上"查不到"要用 NOT_FOUND，而不是返回空对象
            responseObserver.onError(
                    io.grpc.Status.NOT_FOUND
                            .withDescription("用户不存在: " + request.getId())
                            .asRuntimeException());
            return;
        }
        responseObserver.onNext(user);
        responseObserver.onCompleted();
    }
}
```

三条硬性规则：

- `onNext` 之后必须调用 `onCompleted`，否则客户端会一直等。
- `onError` 与 `onCompleted` 只能调用其中一个，且只能调用一次。
- 业务异常不要直接向外抛 `RuntimeException`，应转换成 `StatusRuntimeException`，否则客户端只能看到 `UNKNOWN`。

## 2.2 注册与启动

```java
public class UserServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder.forPort(50051)
                .addService(new UserServiceImpl())
                // 生产环境建议显式指定线程池，避免默认缓存线程池无限增长
                .executor(java.util.concurrent.Executors.newFixedThreadPool(32))
                .build()
                .start();
        System.out.println("gRPC server started on 50051");
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("shutting down gRPC server");
            server.shutdown(); // 停止接收新请求，等待在途请求结束
            try {
                if (!server.awaitTermination(5, TimeUnit.SECONDS)) {
                    server.shutdownNow(); // 超时后强制取消在途请求
                }
            } catch (InterruptedException e) {
                server.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }));
        server.awaitTermination(); // 阻塞主线程，直到服务关闭
    }
}
```

`ServerBuilder` 还有几个常用配置：

| 配置 | 作用 |
|------|------|
| `addService(...)` | 注册一个 `BindableService`，可多次调用 |
| `executor(...)` | 业务处理线程池 |
| `maxInboundMessageSize(...)` | 单条入站消息上限，默认 4 MiB |
| `keepAliveTime(...)` | 服务端 keepalive 探测间隔 |
| `handshakeTimeout(...)` | TLS 握手超时 |

## 2.3 优雅停机

`shutdown()` 与 `shutdownNow()` 的区别经常被忽略：

```text
server.shutdown()
    │
    ├── 停止 accept 新连接
    ├── 允许在途 RPC 继续执行
    └── 在途 RPC 结束后 server 终止
server.shutdownNow()
    │
    ├── 立即关闭连接
    ├── 在途 RPC 被取消，客户端收到 CANCELLED/UNAVAILABLE
    └── 用于超时兜底，不用于正常发布
```

> **行业实践**：Kubernetes 滚动发布时，先给 Pod 发 `SIGTERM`，应用收到后调用 `server.shutdown()`，同时从服务注册中心摘除实例。等待时间要大于最长的在途请求，否则客户端会看到一批 `UNAVAILABLE`。

# 3. 客户端 Stub

## 3.1 ManagedChannel

`ManagedChannel` 是客户端到服务端的逻辑连接，负责 DNS、连接、负载均衡、重连和流控。创建方式：

```java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
ManagedChannel channel = ManagedChannelBuilder
        .forAddress("localhost", 50051)
        .usePlaintext() // 明文，仅本地开发；生产必须用 TLS
        .build();
```

`ManagedChannel` 是**线程安全**的，一个进程通常只保留一个到目标服务的 channel。它内部会复用 TCP 连接，并通过 HTTP/2 多路复用承载并发 RPC。不要每次请求都 `build()` 一个 channel，那会导致连接数暴涨、握手开销巨大。

## 3.2 三种 Stub

`UserServiceGrpc` 提供了三种客户端：

| Stub | 创建方式 | 调用风格 | 适用场景 |
|------|----------|----------|----------|
| BlockingStub | `newBlockingStub(channel)` | 同步阻塞，直接返回结果 | 普通业务调用、脚本、测试 |
| Stub | `newStub(channel)` | 异步回调，`StreamObserver` | 流式调用、需要非阻塞 |
| FutureStub | `newFutureStub(channel)` | 返回 `ListenableFuture` | 需要组合异步任务 |

三种 stub 都由同一个 channel 创建，彼此独立、线程安全，可以缓存为单例。

## 3.3 最小可运行客户端

```java
public class UserClient {
    public static void main(String[] args) throws InterruptedException {
        ManagedChannel channel = ManagedChannelBuilder
                .forAddress("localhost", 50051)
                .usePlaintext()
                .build();
        try {
            UserServiceGrpc.UserServiceBlockingStub stub =
                    UserServiceGrpc.newBlockingStub(channel);
            UserReply reply = stub
                    .withDeadlineAfter(2, TimeUnit.SECONDS) // 必须设置 deadline
                    .getUser(GetUserRequest.newBuilder().setId(1L).build());
            System.out.println("id=" + reply.getId() + ", name=" + reply.getName());
        } catch (StatusRuntimeException e) {
            System.err.println("gRPC failed: " + e.getStatus().getCode()
                    + " - " + e.getStatus().getDescription());
        } finally {
            channel.shutdown();
            if (!channel.awaitTermination(5, TimeUnit.SECONDS)) {
                channel.shutdownNow();
            }
        }
    }
}
```

运行顺序：先启动 `UserServer`，再运行 `UserClient`。如果没有创建过用户，`getUser(1)` 会返回 `NOT_FOUND`，这正好验证了错误码链路。

# 4. 阻塞、异步与 Future 的取舍

## 4.1 BlockingStub

阻塞 stub 最接近本地方法调用，代码可读性最好。它内部会阻塞当前线程，直到收到响应或 deadline 到期。

**注意**：不要在 gRPC 服务端的业务线程里调用阻塞 stub 去访问另一个服务，除非你清楚线程池大小。大量阻塞调用会把线程池占满，形成级联阻塞。异步 stub 或 Future stub 更适合服务间扇出调用。

## 4.2 异步 Stub

异步 stub 的每个方法都接收一个 `StreamObserver`：

```java
UserServiceGrpc.UserServiceStub asyncStub = UserServiceGrpc.newStub(channel);
asyncStub.getUser(
        GetUserRequest.newBuilder().setId(1L).build(),
        new io.grpc.stub.StreamObserver<UserReply>() {
            @Override
            public void onNext(UserReply value) {
                System.out.println("收到: " + value.getName());
            }
            @Override
            public void onError(Throwable t) {
                io.grpc.Status status = io.grpc.Status.fromThrowable(t);
                System.err.println("失败: " + status.getCode());
            }
            @Override
            public void onCompleted() {
                System.out.println("调用结束");
            }
        });
```

回调线程由 gRPC 的执行器提供，**不要在回调里做长时间阻塞操作**，否则会拖慢同一执行器上的其他回调。需要重活时，把任务提交到业务线程池。

## 4.3 FutureStub

Future stub 返回 Guava 的 `ListenableFuture`：

```java
UserServiceGrpc.UserServiceFutureStub futureStub =
        UserServiceGrpc.newFutureStub(channel);
com.google.common.util.concurrent.ListenableFuture<UserReply> future =
        futureStub.withDeadlineAfter(2, TimeUnit.SECONDS)
                .getUser(GetUserRequest.newBuilder().setId(1L).build());
future.addListener(() -> {
    try {
        UserReply reply = future.get(); // 回调触发时已完成，get 不会长时间阻塞
        System.out.println(reply.getMessage());
    } catch (Exception e) {
        System.err.println("失败: " + io.grpc.Status.fromThrowable(e));
    }
}, com.google.common.util.concurrent.MoreExecutors.directExecutor());
```


## 4.4 三者对比

| 维度 | BlockingStub | Stub | FutureStub |
|------|--------------|------|------------|
| 返回值 | 直接返回 | 回调 | `ListenableFuture` |
| 是否阻塞 | 是 | 否 | 否，但 `get()` 会阻塞 |
| 流式支持 | 仅一元 | 全部 | 仅一元 |
| 异常形式 | `StatusRuntimeException` | `onError(Throwable)` | `Future.get()` 抛出 |
| 代码复杂度 | 低 | 中 | 中高 |
| 推荐场景 | 业务主流程 | 流式、回调驱动 | 异步编排、批量扇出 |

**设计考量**：gRPC Java 没有提供"响应式"原生 API，`StreamObserver` 是最底层的回调模型。Reactor/Reactor gRPC 是对它的封装，引入前先确认团队是否愿意维护这层抽象。

# 5. 连接管理

## 5.1 Channel 复用

一个 `ManagedChannel` 对应一个逻辑目标（host:port 或服务名），内部可以维护多条连接。复用方式：


如果目标地址会变化，使用 `forTarget("dns:///user-service:50051")` 或服务发现组件，而不是每次重建 channel。

## 5.2 状态与 Keepalive


常用配置：

```java
ManagedChannel channel = ManagedChannelBuilder
        .forAddress("localhost", 50051)
        .usePlaintext()
        .keepAliveTime(30, TimeUnit.SECONDS)       // 空闲时发送 PING
        .keepAliveTimeout(5, TimeUnit.SECONDS)     // PING 等待响应的超时
        .idleTimeout(10, TimeUnit.MINUTES)         // 空闲后进入 IDLE
        .build();
```

**注意**：keepalive 太激进会被服务端或中间代理判定为滥用，可能收到 `ENHANCE_YOUR_CALM` 或连接被断。客户端 keepalive 间隔通常不低于 30 秒，且要与网关的空闲超时匹配。

## 5.3 关闭顺序

正确顺序是：先停止业务调用，再关闭 channel，最后关闭线程池。

```java
channel.shutdown();                                   // 拒绝新调用
if (!channel.awaitTermination(5, TimeUnit.SECONDS)) {
    channel.shutdownNow();                            // 取消在途调用
}
```

**常见错误**：在 `finally` 里只调用 `shutdown()` 不等待，JVM 退出时在途请求被强行中断；或者先关线程池再关 channel，导致回调无处执行。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `UNAVAILABLE: io exception` | 服务未启动、端口错误、TLS 不匹配 | 先用 `telnet`/`nc` 确认端口 |
| `UNIMPLEMENTED` | 服务未注册或全限定名不一致 | 检查 `addService` 与 proto `package` |
| 客户端一直挂起 | 没设 deadline，服务端也没返回 | 所有调用加 `withDeadlineAfter` |
| `onError` 后仍有 `onNext` | 业务代码没 return | 出错后立即 `return` |
| `CANCELLED` | 客户端 deadline 到期或主动取消 | 检查超时配置与服务端耗时 |
| 连接数持续增长 | 每次请求新建 channel | 改为复用单例 channel |
| 服务端线程池耗尽 | 在业务线程里同步调用下游 | 改异步或隔离线程池 |

# 7. 练习与自检

1. 给 `UserService` 增加 `ListUsers` 一元方法，返回 `repeated UserReply`，观察生成代码中 `List<UserReply>` 的用法。
2. 把客户端改成异步 stub，统计从发起到 `onCompleted` 的耗时，并与阻塞 stub 对比。
3. 故意把服务端 `executor` 设为 1 个线程，在 `getUser` 里 `Thread.sleep(3s)`，用两个并发客户端观察排队现象，并解释 keepalive 与 deadline 的作用。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 服务端 | 继承 `ImplBase`，`onNext` + `onCompleted`，错误用 `Status` |
| 启动 | `ServerBuilder.forPort().addService().build().start()` |
| 停机 | `shutdown()` 等在途请求，超时再 `shutdownNow()` |
| 客户端 | 复用 `ManagedChannel`，按场景选 blocking/async/future |
| 超时 | 每个调用都要有 deadline，不要依赖默认值 |
| 生命周期 | channel 与 server 都要在应用关闭时显式释放 |

下一篇进入四种调用模式，把服务端流、客户端流和双向流的 Java 写法与适用场景讲清楚。
