
# 1. Transport 层概述

Transport 层是 Dubbo 的网络传输抽象层，封装了底层 NIO 框架（Netty/Mina/Grizzly），向上层提供统一的通道（Channel）模型。

```
Exchange Layer (请求/响应模型)
    │
    ▼
Transport Layer (网络传输抽象)
    │
    ├── Transporter (SPI)      → Server / Client 工厂
    ├── Channel / ChannelHandler (通道模型)
    ├── Codec2 (编解码器)
    └── Dispatcher (线程分发)
    │
    ▼
Netty / Mina / Grizzly (底层 NIO 框架)
```

# 2. Transporter SPI

```java
@SPI("netty")
public interface Transporter {
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    RemotingServer bind(URL url, ChannelHandler handler) 
        throws RemotingException;

    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) 
        throws RemotingException;
}
```

内置实现：

| Transporter | 底层框架 | 说明 |
|-------------|---------|------|
| **netty** (默认) | Netty 4 | 高性能、稳定，推荐 |
| netty3 | Netty 3 | 老版本兼容 |
| mina | Apache Mina | 与 Netty 类似的 NIO 框架 |
| grizzly | Grizzly | GlassFish 的 NIO 框架 |

# 3. Channel 模型

## 3.1 核心接口

```java
public interface Channel extends Endpoint {
    InetSocketAddress getRemoteAddress();
    boolean isConnected();
    boolean hasAttribute(String key);
    Object getAttribute(String key);
    void setAttribute(String key, Object value);
    void removeAttribute(String key);
}

public interface Endpoint {
    URL getUrl();
    ChannelHandler getChannelHandler();
    void send(Object message) throws RemotingException;
    void close();
    void close(int timeout);
    boolean isClosed();
}
```

## 3.2 ChannelHandler

```java
public interface ChannelHandler {
    void connected(Channel channel) throws RemotingException;
    void disconnected(Channel channel) throws RemotingException;
    void sent(Channel channel, Object message) throws RemotingException;
    void received(Channel channel, Object message) throws RemotingException;
    void caught(Channel channel, Throwable exception) throws RemotingException;
}
```

## 3.3 NettyChannel —— Netty 适配

```java
public class NettyChannel implements Channel {
    private final Channel channel; // Netty 的 Channel
    private final Map<String, Object> attributes = new ConcurrentHashMap<>();

    public NettyChannel(Channel channel, URL url, 
            ChannelHandler handler) {
        this.channel = channel;
        this.url = url;
        this.handler = handler;
    }

    @Override
    public void send(Object message) throws RemotingException {
        // 将 Dubbo 消息编码为字节流并发送
        channel.writeAndFlush(message);
    }
}
```

# 4. NettyClient —— 消费者端

```java
public class NettyClient extends AbstractClient {
    private Bootstrap bootstrap;
    private volatile Channel channel;

    @Override
    protected void doOpen() {
        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
            .channel(NioSocketChannel.class)
            .option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getConnectTimeout())
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    // NettyClientHandler 适配 Dubbo ChannelHandler
                    NettyClientHandler handler = 
                        new NettyClientHandler(getUrl(), NettyClient.this);
                    
                    ch.pipeline()
                        // 1. 解码器（Dubbo 协议帧解析）
                        .addLast("decoder", new DubboCountCodec())
                        // 2. 编码器
                        .addLast("encoder", new InternalEncoder())
                        // 3. 业务处理器
                        .addLast("handler", handler);
                }
            });
    }

    @Override
    protected void doConnect() {
        ChannelFuture future = bootstrap.connect(getConnectAddress());
        // 获取 Netty Channel，包装为 NettyChannel
        this.channel = future.channel();
    }
}
```

## 4.1 AbstractClient 基类

```java
public abstract class AbstractClient extends AbstractPeer 
        implements Client {
    // 重连定时器
    private final HashedWheelTimer reconnectTimer = new HashedWheelTimer();

    @Override
    public void send(Object message) throws RemotingException {
        // 1. 检查连接，必要时重连
        if (!isConnected()) {
            connect();
        }
        
        // 2. 获取 Channel
        Channel channel = getChannel();
        
        // 3. 发送消息
        channel.send(message);
    }

    // 连接断开后自动重连
    protected void reconnect() {
        if (isClosed()) return;
        
        reconnectTimer.newTimeout(timeout -> {
            try {
                connect();
                logger.info("Successfully reconnect to " + getRemoteAddress());
            } catch (Exception e) {
                logger.warn("Reconnect failed, will retry");
                reconnect(); // 递归重连
            }
        }, reconnectPeriod, TimeUnit.MILLISECONDS);
    }
}
```

# 5. NettyServer —— 提供者端

```java
public class NettyServer extends AbstractServer {
    private ServerBootstrap bootstrap;
    private Channel channel;

    @Override
    protected void doOpen() {
        bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    NettyServerHandler handler = 
                        new NettyServerHandler(getUrl(), NettyServer.this);
                    
                    ch.pipeline()
                        .addLast("decoder", new DubboCountCodec())
                        .addLast("encoder", new InternalEncoder())
                        .addLast("handler", handler);
                }
            });

        ChannelFuture future = bootstrap.bind(getBindAddress());
        this.channel = future.channel();
    }
}
```

## 5.1 AbstractServer 基类

```java
public abstract class AbstractServer extends AbstractPeer 
        implements RemotingServer {

    // 客户端连接池（用于服务端向消费端反向通知）
    private final Map<String, Channel> channels = new ConcurrentHashMap<>();

    @Override
    public Collection<Channel> getChannels() {
        return Collections.unmodifiableCollection(channels.values());
    }

    @Override
    public void connected(Channel ch) {
        channels.put(ch.getRemoteAddress().toString(), ch);
    }

    @Override
    public void disconnected(Channel ch) {
        channels.remove(ch.getRemoteAddress().toString());
    }
}
```

# 6. 编解码 Pipeline

```
Netty Pipeline:
┌─────────────────────────────────────┐
│  DubboCountCodec (帧解码器)           │  ← 处理 TCP 粘包/拆包
│  ┌───────────────────────────────┐  │
│  │ 1. 读取 16 字节 Header         │  │
│  │ 2. 从 Header 获取 Body Length  │  │
│  │ 3. 读取 Body Length 字节数据   │  │
│  │ 4. 合并为完整帧               │  │
│  └───────────────────────────────┘  │
├─────────────────────────────────────┤
│  InternalEncoder / InternalDecoder  │  ← 序列化/反序列化消息体
├─────────────────────────────────────┤
│  NettyServerHandler / NettyClientHandler │ ← 业务处理（分发到 Dubbo 逻辑）
└─────────────────────────────────────┘
```

## 6.1 DubboCountCodec —— 帧解码

```java
public class DubboCountCodec implements ChannelHandler {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 1. 读取 16 字节 Header
        // 2. 解析 Magic (0xdabb) 验证协议
        // 3. 解析 Body Length (12-15 字节)
        // 4. 读取完整 Body
        // 5. 调用下一个 Handler 处理完整消息
    }
}
```

## 6.2 DubboCodec —— 序列化编解码

```java
public class DubboCodec extends ExchangeCodec {

    @Override
    protected Object decodeBody(Channel channel, InputStream is, 
            byte[] header) throws IOException {
        byte flag = header[2];
        byte serializationId = getSerializationId(flag);
        long requestId = Bytes.bytes2long(header, 4);
        
        // 1. 根据 serializationId 获取序列化器
        Serialization serialization = 
            CodecSupport.getSerialization(channel.getUrl(), serializationId);
        
        // 2. 反序列化 Body
        if (isRequest(flag)) {
            Request req = new Request(requestId);
            req.setData(serialization.deserialize(channel.getUrl(), is)
                .readObject());  // 读取 Invocation
            return req;
        } else {
            Response res = new Response(requestId);
            byte status = header[3];
            if (status == Response.OK) {
                res.setResult(serialization.deserialize(channel.getUrl(), is)
                    .readObject());  // 读取返回值
            } else {
                res.setStatus(status);
            }
            return res;
        }
    }
}
```

# 7. 心跳机制（Heartbeat）

## 7.1 心跳检测模型

```
Consumer ←──→ Provider
   │             │
   │  每 60s     │
   │  ←──→      │  (双向心跳)
   │             │
   ├─ 连续 3 次无响应 → 判定断开 → 重连
   │             │
```

## 7.2 心跳处理器

```java
public class HeartbeatHandler extends AbstractChannelHandlerDelegate {

    @Override
    public void connected(Channel channel) {
        // 设置读写时间戳
        setReadTimestamp(channel);
        setWriteTimestamp(channel);
    }

    @Override
    public void received(Channel channel, Object message) {
        setReadTimestamp(channel);
        
        if (isHeartbeatRequest(message)) {
            // 收到心跳请求 → 回复心跳响应
            Response response = new Response(
                ((Request) message).getId());
            response.setEvent(true); // 标记为心跳事件
            channel.send(response);
            return;
        }
        
        if (isHeartbeatResponse(message)) {
            // 收到心跳响应 → 更新读时间戳
            return;
        }
        
        handler.received(channel, message);
    }

    @Override
    public void sent(Channel channel, Object message) {
        // 发送消息后更新写时间戳
        if (!isHeartbeatRequest(message)) {
            setWriteTimestamp(channel);
        }
        handler.sent(channel, message);
    }
}
```

## 7.3 心跳定时检测

```java
public class HeartBeatTask implements Runnable {
    private final ChannelProvider channelProvider;
    private final int heartbeat;      // 心跳间隔 (默认 60s)
    private final int heartbeatTimeout; // 心跳超时 (默认 3 倍心跳间隔)

    @Override
    public void run() {
        long now = System.currentTimeMillis();
        
        for (Channel channel : channelProvider.getChannels()) {
            Long lastRead = (Long) channel.getAttribute(KEY_READ_TIMESTAMP);
            Long lastWrite = (Long) channel.getAttribute(KEY_WRITE_TIMESTAMP);
            
            // 1. 长时间没有读 → 发送心跳
            if (lastRead != null 
                    && now - lastRead > heartbeat) {
                Request heartbeat = new Request();
                heartbeat.setTwoWay(true);
                heartbeat.setEvent(true); // 标记 Heartbeat
                channel.send(heartbeat);
            }
            
            // 2. 心跳超时 → 关闭连接
            if (lastRead != null 
                    && now - lastRead > heartbeatTimeout) {
                logger.warn("Close channel " + channel + 
                    " due to heartbeat timeout");
                channel.close();
            }
        }
    }
}
```

# 8. 连接管理

## 8.1 连接模式

```yaml
dubbo:
  protocol:
    connects: 1         # 每个 Provider 建立几条连接（默认 1）
  consumer:
    connections: 1      # 每个 Consumer 对每个 Provider 建几条连接
```

## 8.2 多连接场景

```java
@DubboReference(parameters = {"connections", "4"})
private UserService userService;
```

```java
// DubboProtocol 中的多连接处理
private ExchangeClient[] getClients(URL url) {
    int connections = url.getParameter("connections", 1);
    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        clients[i] = initClient(url);
    }
    return clients;
}
```

## 8.3 连接池复用

```java
// 同一个 host:port 上的所有服务共享同一个 Client 连接
public class HeaderExchangeClient implements ExchangeClient {
    // 内部维护一个共享的 Netty Channel
    // 多个 DubboInvoker 可共享同一个 ExchangeClient
}
```

# 9. 配置

```yaml
dubbo:
  protocol:
    transporter: netty
    server: netty           # netty/mina/grizzly
    client: netty           # netty/mina
    iothreads: 8            # IO 线程数 (默认 CPU + 1)
    buffer: 8192            # 网络缓冲区大小
    heartbeat: 60000         # 心跳间隔 ms
    payload: 8388608         # 最大请求大小 (默认 8MB)
    
  provider:
    timeout: 3000
    payload: 8388608

  consumer:
    reconnect: 3000          # 重连间隔
    sticky: false            # 粘滞连接（同一 Consumer 始终调用同一 Provider）
```

# 10. 自定义 Transport

实现 `Transporter` 接口并注册 SPI：

```java
public class MyTransporter implements Transporter {
    @Override
    public RemotingServer bind(URL url, ChannelHandler handler) {
        return new MyServer(url, handler);
    }

    @Override
    public Client connect(URL url, ChannelHandler handler) {
        return new MyClient(url, handler);
    }
}
```

SPI 注册（`META-INF/dubbo/org.apache.dubbo.remoting.Transporter`）：

```
my=com.example.MyTransporter
```
