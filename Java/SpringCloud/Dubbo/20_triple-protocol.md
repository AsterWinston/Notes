
# 1. Triple 协议概述

Triple 是 Dubbo 3 设计的新一代 RPC 协议，基于 HTTP/2 构建，完全兼容 gRPC 协议。

```
┌──────────────────────────────────────────┐
│             Triple Protocol               │
├──────────────────────────────────────────┤
│  HTTP/2 Transport                         │
│  (Streams, Multiplexing, Flow Control)    │
├───────────────┬──────────────────────────┤
│  Protobuf     │  Wrapped Serialization    │
│  (原生)       │  (Triple Wrapper)          │
├───────────────┴──────────────────────────┤
│  Streaming: Client / Server / Bi-di       │
└──────────────────────────────────────────┘
```

# 2. Triple vs gRPC vs Dubbo

| 特性 | Triple | gRPC | Dubbo |
|------|--------|------|-------|
| 传输层 | HTTP/2 | HTTP/2 | TCP 自定义 |
| 序列化 | Protobuf + 扩展 | Protobuf | Hessian2 等 |
| 流式通信 | 支持 | 支持 | 不支持 |
| 多路复用 | 原生支持 | 原生支持 | 无 |
| 与 gRPC 互调 | 双向 | - | 否 |
| 浏览器访问 | 支持 (HTTP/2/1.1) | 需 gRPC-Web | 否 |
| 服务治理 | Dubbo 全部能力 | 有限 | Dubbo 全部能力 |
| 跨语言 | 好 | 好 | 有限 |

# 3. Triple 协议帧格式

## 3.1 请求格式

```
HTTP/2 HEADERS Frame:
  :method: POST
  :scheme: http
  :path: /org.example.UserService/getUser
  :authority: 192.168.1.100:50051
  content-type: application/grpc+proto
  te: trailers

HTTP/2 DATA Frame:
  0x00 0x00 0x00 0x00 0x05   ← Length-Prefixed Message (5 字节压缩标志 + 4 字节长度)
  [protobuf serialized body]

HTTP/2 HEADERS Frame (trailers):
  grpc-status: 0
  grpc-message: OK
```

## 3.2 Triple Wrapper 格式

不使用 Protobuf 时，Triple 使用 Wrapper 格式支持任意序列化：

```
Length-Prefixed Message:
┌────────────────┬──────────────────────┐
│ first byte 0x0 │ 4 bytes length (大端) │ ← 前缀 (5 字节)
├────────────────┼──────────────────────┤
│ serialization   │  body bytes          │ ← 数据
└────────────────┴──────────────────────┘
  1 byte: 序列化类型
    - 0: protobuf
    - 2: hessian2
    - 4: json
```

# 4. 流式通信

## 4.1 流式类型

| 类型 | 类名 | 说明 |
|------|------|------|
| Unary | `UnaryCall` | 一次请求，一次响应 |
| Server Stream | `ServerStream` | 一次请求，多次响应 |
| Client Stream | `ClientStream` | 多次请求，一次响应 |
| Bi-directional Stream | `BiDiStream` | 双向流 |

## 4.2 接口定义

```java
// Unary (普通 RPC)
User getUser(Long id);

// Server Stream
void listUsers(Empty request, StreamObserver<User> responseObserver);

// Client Stream
StreamObserver<User> createUsers(StreamObserver<Result> responseObserver);

// Bi-directional Stream
StreamObserver<ChatMessage> chat(StreamObserver<ChatMessage> responseObserver);
```

## 4.3 Server Stream 示例

```java
// 接口
public interface UserService {
    void listUsers(Empty request, StreamObserver<User> responseObserver);
}

// Provider 实现
@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public void listUsers(Empty request, 
            StreamObserver<User> responseObserver) {
        
        // 模拟分批返回大量数据
        for (int page = 0; page < 10; page++) {
            List<User> users = loadPage(page);
            for (User user : users) {
                responseObserver.onNext(user);  // 发送每个 User
            }
        }
        responseObserver.onCompleted();  // 完成流
    }
}

// Consumer 使用
@DubboReference
private UserService userService;

public void demo() {
    userService.listUsers(Empty.getDefaultInstance(), 
        new StreamObserver<User>() {
            @Override
            public void onNext(User user) {
                System.out.println("Received: " + user.getName());
            }
            
            @Override
            public void onError(Throwable t) {
                t.printStackTrace();
            }
            
            @Override
            public void onCompleted() {
                System.out.println("Stream completed");
            }
        });
}
```

## 4.4 Client Stream 示例

```java
// 批量创建用户
@DubboReference
private UserService userService;

public void batchCreateUsers() {
    StreamObserver<Result> responseObserver = new StreamObserver<>() {
        @Override
        public void onNext(Result result) {
            System.out.println("Batch result: " + result.getMessage());
        }

        @Override
        public void onError(Throwable t) {
            t.printStackTrace();
        }

        @Override
        public void onCompleted() {
            System.out.println("All users created");
        }
    };

    StreamObserver<User> requestObserver = 
        userService.createUsers(responseObserver);

    // 流式发送多个请求
    for (User user : userList) {
        requestObserver.onNext(user);
    }
    requestObserver.onCompleted();  // 请求发送完毕
}
```

## 4.5 Bi-directional Stream 示例

```java
// 实时聊天
@DubboReference
private ChatService chatService;

public void chat() {
    StreamObserver<ChatMessage> responseObserver = new StreamObserver<>() {
        @Override
        public void onNext(ChatMessage msg) {
            System.out.println("Received: " + msg.getContent());
        }
        
        @Override
        public void onError(Throwable t) {}
        
        @Override
        public void onCompleted() {}
    };

    StreamObserver<ChatMessage> requestObserver = 
        chatService.chat(responseObserver);

    // 发送消息
    requestObserver.onNext(ChatMessage.newBuilder()
        .setContent("Hello").build());
    requestObserver.onNext(ChatMessage.newBuilder()
        .setContent("How are you?").build());
    
    // 接收响应在 responseObserver.onNext() 中处理
    requestObserver.onCompleted();
}
```

# 5. gRPC 互操作性

## 5.1 Dubbo 调用 gRPC

```yaml
# Dubbo Consumer → gRPC Provider
dubbo:
  consumer:
    protocol: tri    # 指定 Triple 协议
```

```java
@DubboReference(protocol = "tri")
private GreeterService greeterService;
```

Dubbo Consumer 可以直接调用标准的 gRPC 服务。

## 5.2 gRPC 调用 Dubbo

```java
// gRPC Client → Dubbo Provider (Triple 协议)
// 标准的 gRPC Client 代码，无需修改

ManagedChannel channel = ManagedChannelBuilder
    .forAddress("localhost", 50051)
    .usePlaintext()
    .build();

GreeterGrpc.GreeterBlockingStub stub = 
    GreeterGrpc.newBlockingStub(channel);

HelloReply response = stub.sayHello(
    HelloRequest.newBuilder().setName("world").build());
```

## 5.3 Protobuf IDL 定义

```protobuf
syntax = "proto3";

option java_multiple_files = true;
option java_package = "org.example.api";
option java_outer_classname = "UserServiceProto";

service UserService {
    rpc getUser (GetUserRequest) returns (User) {}
    rpc listUsers (Empty) returns (stream User) {}
    rpc createUsers (stream User) returns (Result) {}
    rpc chat (stream ChatMessage) returns (stream ChatMessage) {}
}

message GetUserRequest {
    int64 id = 1;
}

message User {
    int64 id = 1;
    string name = 2;
    int32 age = 3;
}

message Empty {}
message Result { string message = 1; }
message ChatMessage { string content = 1; }
```

# 6. HTTP 访问支持

Triple 协议支持通过 HTTP/1.1 访问（无流式），使 Dubbo 服务可以被浏览器、curl 直接调用：

```bash
# curl 调用 Triple 服务 (JSON 格式)
curl -X POST \
  -H "Content-Type: application/json" \
  -d '["1"]' \
  http://localhost:50051/org.example.UserService/getUser
```

```bash
# curl 调用 Triple 服务 (Protobuf 格式)  
curl -X POST \
  -H "Content-Type: application/grpc+proto" \
  -d @request.bin \
  http://localhost:50051/org.example.UserService/getUser
```

# 7. Triple 协议源码分析

## 7.1 TripleProtocol

```java
public class TripleProtocol extends AbstractProtocol {
    
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) {
        URL url = invoker.getUrl();
        String key = serviceKey(url);
        
        // 构建 TripleInvoker (支持 Protobuf 和 Wrapper 双模)
        TripleInvoker<T> tripleInvoker = new TripleInvoker<>(
            invoker, url, serialization);
        
        Exporter<T> exporter = new TripleExporter<>(
            invoker, key, exporterMap);
        exporterMap.put(key, exporter);
        
        // 启动 HTTP/2 Server (基于 Netty Http2)
        openServer(url);
        
        return exporter;
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) {
        // 创建 TripleInvoker，连接 Provider 的 HTTP/2 Server
        return new TripleInvoker<>(type, url, getClients(url));
    }
}
```

## 7.2 TripleHttp2Protocol —— HTTP/2 实现

```java
public class TripleHttp2Protocol {
    
    void export(Http2Server server, Invoker<?> invoker) {
        // 注册 HTTP/2 路由
        // path: /{service}/{method}
        String path = "/" + invoker.getInterface().getName() 
            + "/" + methodName;
        
        server.register(path, (headers, data) -> {
            // 1. 解析 HTTP/2 HEADERS → Request
            // 2. 解析 HTTP/2 DATA → Invocation
            // 3. 调用 invoker.invoke(invocation)
            // 4. 构建 HTTP/2 Response → HEADERS + DATA
        });
    }
}
```

## 7.3 双模序列化支持

```java
public class TripleInvoker<T> extends AbstractInvoker<T> {

    @Override
    protected Result doInvoke(Invocation invocation) {
        // 1. 检查 Content-Type
        String contentType = getContentType(invocation);
        
        if (contentType.startsWith("application/grpc")) {
            // Protobuf 模式
            return invokeWithProtobuf(invocation);
        } else {
            // Triple Wrapper 模式
            return invokeWithWrapper(invocation);
        }
    }

    private Result invokeWithWrapper(Invocation invocation) {
        // 1. 使用 Triple Wrapper 格式打包
        //    [0x0][4 bytes length][1 byte serial-id][body]
        // 2. 通过 HTTP/2 DATA 帧发送
        // 3. 接收 HTTP/2 DATA + TRAILERS
        // 4. 解析返回结果
    }
}
```

# 8. 配置 Triple 协议

```yaml
dubbo:
  protocol:
    name: tri
    port: 50051
    serialization: protobuf    # 或 hessian2
    transporter: netty
```

```java
// 服务同时暴露 Dubbo + Triple
@DubboService(protocol = "dubbo,tri")
public class UserServiceImpl implements UserService {
}
```

Consumer 使用：

```java
@DubboReference(protocol = "tri")
private UserService userService;
```

# 9. Triple + Spring MVC 集成

Dubbo 3.2+ 支持与 Spring MVC 无缝集成，直接将 Triple 服务注册为 Spring Controller：

```java
@DubboService(protocol = "tri")
@RestController
@RequestMapping("/api/users")
public class UserServiceImpl implements UserService {

    @Override
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // 同一方法同时提供 Triple RPC 和 REST API
        return new User(id, "user-" + id, 25);
    }
}
```

调用方式：

```
# REST 方式
curl http://localhost:50051/api/users/1

# Triple RPC 方式
(通过 Dubbo Consumer 调用)
```

# 10. Triple 性能特点

- **HTTP/2 多路复用**：一个 TCP 连接同时承载多个请求，消除连接瓶颈
- **头部压缩**：HPACK 压缩 HTTP Headers，减少带宽
- **流量控制**：Stream 级别的流控，防止单一流占满带宽
- **服务端推送**：Server Stream 支持推送模式
- **二进制帧**：比 HTTP/1.1 文本协议解析更快
