gRPC 的四种调用方式是它区别于 REST 的核心能力。很多人只会在 Unary 里打转，遇到大结果集、实时推送、批量上报就退回 HTTP。本篇用一套完整的 Java 示例把四种模式跑通，并说明每种模式在什么场景下才真正划算。

# 1. 四种调用方式总览

## 1.1 一张表看懂

| 模式 | 客户端发送 | 服务端返回 | 典型场景 |
|------|-----------|-----------|----------|
| Unary（一元） | 1 条消息 | 1 条消息 | 查询、创建、常规 RPC |
| Server Streaming | 1 条消息 | N 条消息 | 日志推送、大结果集、订阅 |
| Client Streaming | N 条消息 | 1 条消息 | 文件上传、批量聚合 |
| Bidirectional Streaming | N 条消息 | N 条消息 | 实时聊天、双向同步、流式处理 |


四种模式的底层都是"一个 HTTP/2 stream 上收发多个 DATA 帧"，区别只在于**谁先结束、谁发几条**。理解这一点，就不会把流式当成"多条独立请求"，而是"一次调用内的消息序列"。

## 1.2 示例 proto

```protobuf
syntax = "proto3";
package stream.v1;
option java_package = "com.example.stream.v1";
option java_multiple_files = true;
option java_outer_classname = "StreamProto";
service StreamDemo {
  rpc Add (AddRequest) returns (AddReply);                          // 一元
  rpc Range (RangeRequest) returns (stream RangeReply);             // 服务端流
  rpc Upload (stream Chunk) returns (UploadReply);                  // 客户端流
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);       // 双向流
}
message AddRequest {
  int64 a = 1;
  int64 b = 2;
}
message AddReply {
  int64 sum = 1;
}
message RangeRequest {
  int32 start = 1;
  int32 end = 2;
}
message RangeReply {
  int32 value = 1;
}
message Chunk {
  bytes data = 1;
}
message UploadReply {
  int64 size = 1;
}
message ChatMessage {
  string user = 1;
  string text = 2;
}
```

# 2. Unary：最常用的一问一答

## 2.1 服务端

```java
package com.example.stream;
import com.example.stream.v1.AddReply;
import com.example.stream.v1.AddRequest;
import com.example.stream.v1.StreamDemoGrpc;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
public class StreamDemoService extends StreamDemoGrpc.StreamDemoImplBase {
    @Override
    public void add(AddRequest request, StreamObserver<AddReply> responseObserver) {
        if (request.getA() == 0 && request.getB() == 0) {
            responseObserver.onError(Status.INVALID_ARGUMENT
                    .withDescription("两个操作数不能同时为 0")
                    .asRuntimeException());
            return;
        }
        responseObserver.onNext(AddReply.newBuilder()
                .setSum(request.getA() + request.getB())
                .build());
        responseObserver.onCompleted();
    }
}
```

## 2.2 客户端

```java
AddReply reply = blockingStub
        .withDeadlineAfter(2, TimeUnit.SECONDS)
        .add(AddRequest.newBuilder().setA(1).setB(2).build());
System.out.println("sum = " + reply.getSum());
```

Unary 不涉及流的生命周期，是最容易调试的模式。绝大多数 CRUD 接口都应该先用 Unary，不要为了"看起来高级"而上流式。

# 3. Server Streaming：一次请求，多次响应

## 3.1 服务端实现

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
        // 每次 onNext 写一条消息，所有消息共享同一个 HTTP/2 stream
        responseObserver.onNext(RangeReply.newBuilder().setValue(i).build());
        try {
            Thread.sleep(10); // 模拟耗时，生产环境不要阻塞 gRPC 执行器线程
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            responseObserver.onError(Status.CANCELLED
                    .withDescription("服务端被中断")
                    .asRuntimeException());
            return;
        }
    }
    responseObserver.onCompleted(); // 正常结束，客户端会收到 onCompleted
}
```

服务端流的结束信号是 `onCompleted()`。如果服务端只 `onNext` 不 `onCompleted`，客户端会一直等，直到 deadline 到期。

## 3.2 客户端：阻塞式迭代器

Java 的 `BlockingStub` 对服务端流提供了 `Iterator` 版本，写法比 Go 更简单：

```java
Iterator<RangeReply> it = blockingStub
        .withDeadlineAfter(10, TimeUnit.SECONDS)
        .range(RangeRequest.newBuilder().setStart(1).setEnd(5).build());
while (it.hasNext()) {
    System.out.println("value = " + it.next().getValue());
}
```

`Iterator.hasNext()` 会阻塞等待下一条消息；流正常结束时返回 `false`，不会抛异常。如果服务端返回错误，`hasNext()` 或 `next()` 会抛出 `StatusRuntimeException`。

## 3.3 客户端：异步观察者

需要自定义背压或取消时，用异步 stub：

```java
StreamDemoGrpc.StreamDemoStub asyncStub = StreamDemoGrpc.newStub(channel);
asyncStub.withDeadlineAfter(10, TimeUnit.SECONDS)
        .range(RangeRequest.newBuilder().setStart(1).setEnd(5).build(),
                new StreamObserver<RangeReply>() {
                    @Override
                    public void onNext(RangeReply value) {
                        System.out.println("收到: " + value.getValue());
                    }
                    @Override
                    public void onError(Throwable t) {
                        Status status = Status.fromThrowable(t);
                        System.err.println("流失败: " + status.getCode());
                    }
                    @Override
                    public void onCompleted() {
                        System.out.println("流正常结束");
                    }
                });
```

`onError` 与 `onCompleted` 互斥，且只触发一次。收到 `onError` 后不会再有 `onNext`。

# 4. Client Streaming：多次请求，一次响应

## 4.1 服务端实现

```java
@Override
public StreamObserver<Chunk> upload(StreamObserver<UploadReply> responseObserver) {
    return new StreamObserver<Chunk>() {
        private long total = 0;
        @Override
        public void onNext(Chunk chunk) {
            // 每收到一块就累加，内存占用恒定
            total += chunk.getData().size();
        }
        @Override
        public void onError(Throwable t) {
            // 客户端取消或网络中断；这里只记录，不再向 responseObserver 写
            System.err.println("upload failed: " + Status.fromThrowable(t).getCode());
        }
        @Override
        public void onCompleted() {
            // 客户端半关闭，返回唯一响应
            responseObserver.onNext(UploadReply.newBuilder().setSize(total).build());
            responseObserver.onCompleted();
        }
    };
}
```

## 4.2 客户端实现

```java
StreamDemoGrpc.StreamDemoStub asyncStub = StreamDemoGrpc.newStub(channel);
StreamObserver<UploadReply> responseObserver = new StreamObserver<UploadReply>() {
    @Override public void onNext(UploadReply v) { System.out.println("服务端收到字节数: " + v.getSize()); }
    @Override public void onError(Throwable t) { System.err.println("上传失败: " + Status.fromThrowable(t).getCode()); }
    @Override public void onCompleted() { System.out.println("上传完成"); }
};
StreamObserver<Chunk> requestObserver =
        asyncStub.withDeadlineAfter(30, TimeUnit.SECONDS).upload(responseObserver);
try {
    for (byte[] part : parts) {
        requestObserver.onNext(Chunk.newBuilder()
                .setData(ByteString.copyFrom(part))
                .build());
    }
    requestObserver.onCompleted(); // 必须显式半关闭，服务端才会回响应
} catch (RuntimeException e) {
    // onNext 可能因流已关闭而抛异常，此时要取消整个流
    requestObserver.onError(e);
}
```

客户端流的收尾动作是 `requestObserver.onCompleted()`。如果只 `onNext` 不 `onCompleted`，服务端的 `onCompleted` 永远不会触发，最后被 deadline 杀掉。

# 5. Bidirectional Streaming：双向实时

## 5.1 服务端实现

```java
@Override
public StreamObserver<ChatMessage> chat(StreamObserver<ChatMessage> responseObserver) {
    return new StreamObserver<ChatMessage>() {
        @Override
        public void onNext(ChatMessage message) {
            // 收到一条就回一条；收发在同一个 stream 上交错进行
            responseObserver.onNext(ChatMessage.newBuilder()
                    .setUser("server")
                    .setText("echo: " + message.getText())
                    .build());
        }
        @Override
        public void onError(Throwable t) {
            System.err.println("chat stream error: " + Status.fromThrowable(t).getCode());
        }
        @Override
        public void onCompleted() {
            responseObserver.onCompleted(); // 客户端关闭发送方向，服务端也结束
        }
    };
}
```

## 5.2 客户端实现

```java
StreamObserver<ChatMessage> responseObserver = new StreamObserver<ChatMessage>() {
    @Override public void onNext(ChatMessage v) { System.out.println(v.getUser() + ": " + v.getText()); }
    @Override public void onError(Throwable t) { System.err.println(Status.fromThrowable(t).getCode()); }
    @Override public void onCompleted() { System.out.println("对话结束"); }
};
StreamObserver<ChatMessage> requestObserver =
        asyncStub.withDeadlineAfter(60, TimeUnit.SECONDS).chat(responseObserver);
requestObserver.onNext(ChatMessage.newBuilder().setUser("client").setText("hi").build());
requestObserver.onNext(ChatMessage.newBuilder().setUser("client").setText("bye").build());
requestObserver.onCompleted();
```

双向流允许收发交错，但 **`StreamObserver` 不是线程安全的**。如果多个线程同时调用 `requestObserver.onNext`，必须自己加锁或串行化到一个发送线程。gRPC 只保证响应回调的串行性，不保证你调用请求观察者时的线程安全。

# 6. 流式调用的关键细节

## 6.1 背压与可写性

gRPC Java 通过 `CallStreamObserver` 暴露流控能力：

```java
// 服务端：判断当前是否可写，避免无节制 onNext
ServerCallStreamObserver<RangeReply> serverObserver =
        (ServerCallStreamObserver<RangeReply>) responseObserver;
if (serverObserver.isReady()) {
    serverObserver.onNext(reply);
}
```

客户端控制入站消息的拉取节奏：

```java
ClientCallStreamObserver<RangeReply> clientObserver =
        (ClientCallStreamObserver<RangeReply>) responseObserver;
clientObserver.disableAutoInboundFlowControl(); // 关闭自动拉取
clientObserver.request(1);                      // 每次只请求一条
// 在 onNext 处理完后继续 request(1)，形成拉模式
```

**设计考量**：默认情况下 gRPC 会自动请求消息，应用层感知不到背压。数据量大或处理耗时长时，手动流控能防止消息在内存里堆积，但也会增加代码复杂度。只有确实出现内存压力时才值得开启。

## 6.2 取消

客户端可以主动取消一个流：

```java
ClientCallStreamObserver<Chunk> requestObserver = ...;
requestObserver.cancel("用户取消上传", null);
```

服务端可以感知取消：

```java
ServerCallStreamObserver<RangeReply> serverObserver =
        (ServerCallStreamObserver<RangeReply>) responseObserver;
serverObserver.setOnCancelHandler(() -> {
    System.out.println("客户端已取消，停止生产数据");
});
```

长时间运行的循环应该定期检查 `Context.current().isCancelled()` 或依赖 `setOnCancelHandler` 设置标志位，否则客户端早已离开，服务端还在空转烧 CPU。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 客户端一直等不到结束 | 服务端只 `onNext` 没 `onCompleted` | 补 `onCompleted` |
| 服务端一直等不到结束 | 客户端只 `onNext` 没 `onCompleted` | 补半关闭 |
| `StreamObserver` 并发异常 | 多线程同时 `onNext` | 单线程发送或加锁 |
| 流被 deadline 杀掉 | 单条消息超时设得过短 | 流式调用用更长的总超时 |
| 内存暴涨 | 服务端无节制生产数据 | 用 `isReady`/`request(n)` 做背压 |
| `CANCELLED` 但没有业务异常 | 客户端主动取消或超时 | 检查客户端 deadline 与取消逻辑 |
| 服务端日志刷屏 | 在回调里同步写大量日志 | 异步日志、采样 |

# 8. 练习与自检

1. 实现 `Countdown(n)` 服务端流，从 n 递减到 0，客户端用 `Iterator` 打印每个值。
2. 实现客户端流版"求平均值"：客户端发多条数字，服务端返回平均值，并处理客户端中途 `onError` 的情况。
3. 把双向流示例改成"客户端 3 秒后主动取消"，观察服务端 `onError` 收到什么状态码，并解释 `RST_STREAM` 的作用。

# 9. 小结

| 模式 | 关键 API | 记住一点 |
|------|----------|----------|
| Unary | `stub.method(req, responseObserver)` | 出错用 `Status`，别抛裸异常 |
| Server Streaming | 服务端多次 `onNext` + `onCompleted` | 阻塞 stub 用 `Iterator` 消费 |
| Client Streaming | `onNext` 多次 + `onCompleted` 半关闭 | 不半关闭服务端不会回响应 |
| Bidirectional | 两个 `StreamObserver` 交错收发 | 单线程发送，回调里别阻塞 |
| 流控 | `isReady`、`request(n)` | 大数据量才手动背压 |
| 取消 | `cancel`、`setOnCancelHandler` | 长循环必须响应取消 |

下一篇进入拦截器，讲清 Client/Server 两套拦截器、链式顺序、日志与鉴权，以及 Context 如何跨拦截器传递。
