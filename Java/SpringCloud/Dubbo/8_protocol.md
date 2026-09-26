
# 1. 协议概述

Dubbo 的 Protocol 层负责 RPC 协议的实现，定义请求和响应的序列化/反序列化和网络传输格式。

```java
@SPI("dubbo")
public interface Protocol {
    int getDefaultPort();
    
    // Provider 端：暴露服务
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    
    // Consumer 端：引用服务
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
    
    void destroy();
}
```

# 2. 支持的协议

| 协议 | 协议头值 | 特点 | 适用场景 |
|------|---------|------|---------|
| **dubbo** | 0xdabb | 单一长连接、NIO 异步传输、适合小数据量高并发 | 服务调用 |
| **triple** | HTTP/2 | HTTP/2、Protobuf、流式、兼容 gRPC | 跨语言、云原生 |
| **rest** | HTTP | 标准 RESTful，基于 JAX-RS | 对外 API |
| **grpc** | HTTP/2 | gRPC 原生协议 | 多语言环境 |
| **hessian** | HTTP | Hessian 协议，基于 HTTP POST | 内网低延迟 |
| **http** | HTTP | 基于 Spring HTTP Invoker | 老系统兼容 |
| **redis** | Redis | 通过 Redis 模拟 RPC | 特殊场景 |
| **thrift** | Thrift | Apache Thrift 原生协议 | 跨语言 |

# 3. Dubbo 协议（默认协议）

## 3.1 特点

- **单一长连接**：Consumer 与 Provider 之间维护一条 TCP 长连接（连接复用）
- **NIO 异步通信**：依赖 Netty 实现 NIO，线程模型高效
- **请求/响应模型**：每个请求带唯一 ID，支持同步/异步/回调
- **TCP 心跳**：心跳检测自动剔除不健康连接

## 3.2 帧格式（Dubbo 协议帧）

```
┌──────────────────────────────────────────────────────────────┐
│  0-1   │   2   │   3    │  4-11  │ 12-15 │   16-...          │
├────────┼───────┼───────┼────────┼───────┼──────────────────┤
│ Magic  │ Flag  │Status │   ID   │Length │  Body             │
│(0xdabb)│       │       │ (8B)   │ (4B)  │  (序列化数据)       │
└──────────────────────────────────────────────────────────────┘
```

| 字段 | 字节 | 说明 |
|------|------|------|
| Magic | 2 | 魔数 0xdabb，用于协议识别 |
| Flag | 1 | 标志位：request/response | two-way/one-way | heartbeat | serialization_id |
| Status | 1 | 响应状态：20=OK, 30=CLIENT_TIMEOUT, 31=SERVER_TIMEOUT 等 |
| ID | 8 | 请求 ID，long 类型，用于关联请求与响应 |
| Length | 4 | 消息体长度（不含帧头 16 字节） |
| Body | N | 序列化后的数据 |

## 3.3 编解码实现

```java
// DubboCodec —— 编解码入口
public class DubboCodec extends ExchangeCodec {

    @Override
    protected Object decodeBody(Channel channel, InputStream is, 
            byte[] header) throws IOException {
        byte flag = header[2];
        // 获取序列化方式（从 Flag 中提取 serialization_id）
        Serialization serialization = CodecSupport.getSerialization(
            channel.getUrl(), getSerializationId(flag)
        );
        
        // 反序列化 Body
        long requestId = Bytes.bytes2long(header, 4);
        if (isRequest(flag)) {
            // 解码为 Request 对象
            return decodeRequest(channel, is, header, serialization, requestId);
        } else {
            // 解码为 Response 对象
            return decodeResponse(channel, is, header, serialization, requestId);
        }
    }

    @Override
    protected void encodeData(Channel channel, 
            ChannelBuffer buffer, Object message) throws IOException {
        // 根据消息类型（Request/Response）序列化 Body
        if (message instanceof Request) {
            encodeRequestData(channel, buffer, (Request) message);
        } else {
            encodeResponseData(channel, buffer, (Response) message);
        }
    }
}
```

## 3.4 线程模型

```
Netty IO Worker Thread
    │
    ├── 接收 TCP 数据 → 解码 → Request
    │
    ├── Dispatcher (选择线程分发策略)
    │   ├── all:         所有消息到线程池（默认）
    │   ├── direct:      所有消息在 IO 线程处理
    │   ├── message:     只有请求响应消息到线程池
    │   ├── execution:   只有请求消息到线程池
    │   └── connection:  连接事件按顺序执行，其他到线程池
    │
    └── ThreadPool (业务线程池)
        └── 调用 ServiceImpl → 返回 Response → 编码 → 发送
```

## 3.5 配置参考

```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
    serialization: hessian2
    transporter: netty
    dispatcher: all
    threadpool: fixed
    threads: 200
    iothreads: 8
    queues: 0
    buffer: 8192
    payload: 8388608         # 8MB，最大请求大小
    heartbeat: 60000          # 心跳间隔 ms
    charset: UTF-8
```

# 4. DubboProtocol —— 核心实现

```java
public class DubboProtocol extends AbstractProtocol {

    // key = host:port, value = Netty Server
    private final Map<String, ProtocolServer> serverMap = new ConcurrentHashMap<>();

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) {
        // 1. 构造 serviceKey: group/interface:version
        String key = serviceKey(invoker.getUrl());
        
        // 2. 创建 DubboExporter 存入 exporterMap
        DubboExporter<T> exporter = new DubboExporter<>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);
        
        // 3. 启动 Netty Server（复用已有连接）
        openServer(invoker.getUrl());
        
        return exporter;
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) {
        // 1. 获取/创建 Netty Client 连接
        ExchangeClient[] clients = getClients(url);
        
        // 2. 构造 DubboInvoker
        DubboInvoker<T> invoker = new DubboInvoker<>(type, url, clients);
        invokers.add(invoker);
        
        return invoker;
    }
}
```

# 5. 协议选择指南

## 5.1 场景适配

```
               长连接 → dubbo 协议（默认推荐）
高并发小数据量
               连接复用 → dubbo 协议

大数据量       → dubbo 协议 + kryo/hessian2 序列化

跨语言调用     → Triple 协议 (HTTP/2 + Protobuf)

对外 REST API → rest 协议 (JAX-RS)

流式调用       → Triple 协议 (Streaming)

gRPC 生态     → grpc 协议 或 Triple 协议
```

## 5.2 多协议暴露

```java
@DubboService(protocol = "dubbo,tri")
public class UserServiceImpl implements UserService {
}
```

或配置：

```yaml
dubbo:
  protocols:
    dubbo:
      name: dubbo
      port: 20880
    tri:
      name: tri
      port: 50051
```

同一服务同时通过 Dubbo 协议和 Triple 协议对外暴露。

# 6. Exchange 层 —— 请求/响应模型

Exchange 层在 Protocol 与 Transport 之间，负责将单向消息转换为 Request-Response 模型：

```java
public interface ExchangeClient {
    ResponseFuture request(Object request) throws RemotingException;
    ResponseFuture request(Object request, int timeout) throws RemotingException;
}

// HeaderExchangeClient —— 默认实现
public class HeaderExchangeClient implements ExchangeClient {
    
    @Override
    public ResponseFuture request(Object request, int timeout) {
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);          // 需要响应
        req.setData(request);          // RpcInvocation
        req.setId(ID_GENERATOR.getAndIncrement()); // 唯一 ID
        
        // 创建 DefaultFuture，以 requestId 为 key 存入 Map
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
        
        channel.send(req);
        
        return future;
    }
}
```

## 6.1 DefaultFuture —— 异步转同步

```java
public class DefaultFuture implements ResponseFuture {
    private static final Map<Long, DefaultFuture> FUTURES = 
        new ConcurrentHashMap<>();
    
    private final long id;           // 请求 ID
    private final Lock lock = new ReentrantLock();
    private final Condition done = lock.newCondition();
    private volatile Response response;

    public DefaultFuture(Channel channel, Request request, int timeout) {
        this.id = request.getId();
        FUTURES.put(id, this); // 以 ID 注册到全局 Map
    }

    @Override
    public Object get(int timeout) throws RemotingException {
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                while (!isDone()) {
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } finally {
                lock.unlock();
            }
        }
        return returnFromResponse();
    }

    // Netty 线程收到响应时回调
    public static void received(Channel channel, Response response) {
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            future.doReceived(response); // 唤醒阻塞线程
        }
    }

    private void doReceived(Response res) {
        lock.lock();
        try {
            response = res;
            done.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

**流程**：Consumer 线程调用 `future.get()` 阻塞等待 → Provider 响应到达 Netty 线程 → `received()` 根据 ID 找到 `DefaultFuture` → `done.signalAll()` 唤醒 Consumer 线程。

# 7. 协议 SPI 扩展点

```java
// 协议编解码扩展点
@SPI
public interface Codec2 {
    @Adaptive("codec")
    void encode(Channel channel, ChannelBuffer buffer, Object message) 
        throws IOException;
    
    @Adaptive("codec")
    Object decode(Channel channel, ChannelBuffer buffer) throws IOException;
}

// 序列化扩展点
@SPI("hessian2")
public interface Serialization {
    byte getContentTypeId();
    String getContentType();
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}
```

可通过 SPI 自定义协议：

```
META-INF/dubbo/org.apache.dubbo.rpc.Protocol
myprotocol=com.example.MyProtocol
```

# 8. 性能对比

| 协议 | 单机 QPS | 平均延迟 | 连接模型 | 序列化 |
|------|---------|---------|---------|--------|
| dubbo | 10w+ | < 1ms | 单长连接 | Hessian2 |
| triple | 8w+ | < 2ms | 多路复用 | Protobuf |
| rest | 3w+ | < 5ms | 短连接/Keep-Alive | JSON |
| grpc | 7w+ | < 3ms | 多路复用 | Protobuf |
