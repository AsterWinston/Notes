Netty 是 Java 网络编程里绕不开的一个框架：Dubbo、RocketMQ、Elasticsearch、gRPC-Java、Spring WebFlux 的底层都用它收发字节。上一篇的 Tomcat 解决的是"HTTP 请求进来之后怎么处理"，这一篇解决的是"字节怎么高效地在网络上流动"，两者合起来才算把 JavaWeb 的下半程补齐。

# 1. Netty 是什么

一句话：**Netty 是基于 Java NIO 的异步事件驱动网络框架**，把 JDK 原生 NIO 里那一堆样板代码和坑封装成了一个可复用的编程模型。

它本身不是协议，也不是容器，只是一层"网络 I/O 的抽象"。所以它既能跑 TCP、UDP，也能在其上拼出 HTTP、WebSocket、Redis 协议、自定义二进制协议。

| 维度 | 说明 |
|------|------|
| 定位 | 网络通信框架（不是 Web 容器） |
| 底层 | Java NIO（Selector + Channel），可替换为 Epoll/KQueue 原生传输 |
| 编程模型 | Reactor 主从多线程 + 责任链（Pipeline） |
| 典型使用者 | Dubbo、RocketMQ、Elasticsearch、gRPC、Spring WebFlux |

**设计考量**：Netty 的选择是"小而完整"——只做 I/O 和协议编解码，业务线程模型交给使用方决定。这也是它比 Tomcat 更容易被中间件厂商复用的原因。

# 2. 为什么需要 Netty

JDK 1.4 就有了 NIO，但直接写 NIO 服务端的体验很差。

## 2.1 原生 NIO 的四个痛点

| 痛点 | 具体表现 |
|------|---------|
| API 复杂 | 一个可用的服务端要同时操作 `Selector`、`ServerSocketChannel`、`SelectionKey`、`ByteBuffer` |
| 半包粘包 | `read()` 一次读到的不是一条完整消息，需要自己维护缓冲区拼包 |
| 空轮询 bug | Linux 上 `epoll` 在某些情况下会立刻返回但没有任何就绪事件，导致 CPU 100% |
| 线程模型 | 读写线程、业务线程如何分工，需要自己设计，写错就是性能灾难 |

## 2.2 Netty 解决了什么

- **统一的异步 API**：所有 I/O 操作返回 `ChannelFuture`，回调/监听器风格一致。
- **自动处理半包粘包**：内置 `LengthFieldBasedFrameDecoder`、`LineBasedFrameDecoder` 等。
- **修复空轮询**：Netty 内部统计 `select` 空转次数，超过阈值就重建 Selector。
- **成熟的线程模型**：`EventLoop` 把"一个连接的所有事件"绑定到同一个线程，避免并发问题。
- **内存管理**：`ByteBuf` + 池化分配器（PooledByteBufAllocator）减少 GC 压力。

**注意**：Netty 解决的是网络层问题，不解决业务问题。如果只是写个内部小工具，JDK 的 `HttpServer` 或 Spring Boot 的 Web 容器足够，上 Netty 属于过度设计。

# 3. 核心组件

Netty 的模型可以浓缩成一句话：**一个 Channel 对应一条连接，一条连接绑定一个 EventLoop，一个 Channel 上挂一条 Pipeline，Pipeline 里串着一堆 Handler**。

```
┌─────────────────────────────────────────────────────┐
│                    EventLoopGroup                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │EventLoop │  │EventLoop │  │EventLoop │  ...       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
└───────┼─────────────┼─────────────┼─────────────────┘
        │             │             │
   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
   │ Channel │   │ Channel │   │ Channel │  ← 一条 TCP 连接
   └────┬────┘   └─────────┘   └─────────┘
        │
   ┌────▼──────────────────────────────┐
   │ Pipeline                          │
   │  Inbound:  Decoder → Handler      │
   │  Outbound: Encoder ← Handler      │
   └───────────────────────────────────┘
```

## 3.1 Channel

`Channel` 是对一条网络连接的抽象，可以理解为"增强版的 Socket"。它提供 `read`、`write`、`close`、`connect` 等异步操作，所有方法都返回 `ChannelFuture`。

## 3.2 EventLoop

`EventLoop` 是一个单线程的事件循环，本质是"一个 Selector + 一个任务队列"。

- 一个 `EventLoop` 可以服务多个 `Channel`；
- 一个 `Channel` 从注册开始到关闭，只会绑定一个 `EventLoop`；
- 所以**同一个 Channel 上的事件天然串行**，Handler 里不需要加锁。

## 3.3 ChannelPipeline 与 ChannelHandler

Pipeline 是责任链，Inbound 事件（读、连接建立、异常）从头往后走，Outbound 事件（写、关闭）从尾往前走。

| Handler 类型 | 触发方向 | 典型用途 |
|--------------|---------|---------|
| `ChannelInboundHandler` | 头 → 尾 | 解码、业务处理 |
| `ChannelOutboundHandler` | 尾 → 头 | 编码、限流 |

## 3.4 ByteBuf

`ByteBuf` 用**读指针 + 写指针**替代了 `ByteBuffer` 的 `flip()`，读写不需要切换模式，也不会因为忘记 `flip()` 出错。它支持池化和引用计数，用完必须 `release()`（或用 `SimpleChannelInboundHandler` 自动释放）。

## 3.5 ChannelFuture

Netty 的所有 I/O 都是异步的。`ChannelFuture` 用来拿到"还没完成的操作"的结果，可以加监听器，也可以 `sync()` 阻塞等待。

**设计考量**：异步 API 的代价是"回调地狱"。Netty 4 用 `ChannelFuture` + `ChannelHandler` 缓解，Netty 5 曾尝试 `Future`/`Promise` 统一模型，最终项目被放弃，说明这套模型的分寸拿捏得比较微妙。

# 4. 线程模型：Reactor 与 EventLoop

Netty 支持三种 Reactor 模型，实际项目里几乎只用第三种。

| 模型 | 结构 | 适用场景 |
|------|------|---------|
| 单线程 Reactor | 一个线程处理 accept + 读写 + 业务 | 连接少、业务轻 |
| 多线程 Reactor | 一个 accept 线程 + 一组 I/O 线程 | 通用服务端 |
| 主从多线程 Reactor | boss 组 accept，worker 组读写 | 高并发服务端（Netty 默认用法） |

```
bossGroup（1~2 个线程）       workerGroup（CPU 核数 × 2）
      │                              │
   接受连接  ────注册 Channel────▶  读写事件 / 编解码 / 业务
```

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);      // 只负责 accept
EventLoopGroup workerGroup = new NioEventLoopGroup();     // 默认 CPU × 2
```

**注意**：业务逻辑如果阻塞在 Handler 里，会卡住整个 EventLoop 上的所有 Channel。CPU 密集型或耗时任务必须丢到独立业务线程池，否则 worker 线程数再大也没用。

# 5. 一个最小的 TCP 服务端

## 5.1 启动代码

```java
public class EchoServer {
    public static void main(String[] args) throws InterruptedException {
        // boss 负责接收连接，worker 负责读写
        EventLoopGroup boss = new NioEventLoopGroup(1);
        EventLoopGroup worker = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(boss, worker)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)          // 全连接队列长度
                    .childOption(ChannelOption.SO_KEEPALIVE, true)  // TCP 保活
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline()
                              .addLast(new StringDecoder())          // 入站解码
                              .addLast(new StringEncoder())          // 出站编码
                              .addLast(new EchoHandler());
                        }
                    });
            ChannelFuture future = bootstrap.bind(8080).sync();
            future.channel().closeFuture().sync();                   // 阻塞到服务端关闭
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}
```

## 5.2 业务 Handler

```java
@ChannelHandler.Sharable
public class EchoHandler extends SimpleChannelInboundHandler<String> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        // 原样回写，Netty 会把 write 转成异步操作
        ctx.writeAndFlush("echo: " + msg + "\n");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();   // 打印异常
        ctx.close();               // 出异常就关连接，避免半死不活
    }
}
```

**注意**：`SimpleChannelInboundHandler` 会自动释放消息，而继承 `ChannelInboundHandlerAdapter` 时需要自己 `ReferenceCountUtil.release(msg)`，否则池化的 `ByteBuf` 会泄漏。

# 6. 编解码与粘包、半包

TCP 是字节流，没有消息边界。发两次 `"abc"`、`"def"`，接收端可能收到 `"abcdef"`，也可能只收到 `"ab"`。

## 6.1 三种常见解决思路

| 方案 | 原理 | 示例 |
|------|------|------|
| 定长 | 每条消息长度固定 | 简单但浪费，不适用于变长业务 |
| 分隔符 | 用特定字节标记结尾 | 文本协议（Redis、HTTP 头） |
| 长度字段 | 消息头写明 body 长度 | 二进制协议（Dubbo、RocketMQ） |

## 6.2 长度字段解码器

```java
// 协议：4 字节长度 + 4 字节版本 + 消息体
ch.pipeline()
  .addLast(new LengthFieldBasedFrameDecoder(
          1024 * 1024,   // 单帧最大长度，防止内存被打爆
          0,             // 长度字段的偏移
          4,             // 长度字段占 4 字节
          0,             // 长度字段之后、消息体之前还有多少字节
          4))            // 解码后跳过前面 4 字节长度字段
  .addLast(new MyProtocolDecoder());
```

**注意**：`maxFrameLength` 一定要设，否则对端发一个伪造的超大长度头就能把堆内存撑爆。这是最容易被忽略的安全问题。

## 6.3 编解码器的选择

| 场景 | 推荐 |
|------|------|
| 内部服务 | Protobuf / FlatBuffers |
| 需要可读性 | JSON（用 `JsonObjectDecoder` 或自定义） |
| 兼容老系统 | 自定义二进制协议 + 版本号字段 |

# 7. 零拷贝与内存管理

Netty 的"零拷贝"和操作系统说的零拷贝不是一回事，要分开看。

| 层次 | 手段 | 效果 |
|------|------|------|
| 应用层 | `CompositeByteBuf` 合并逻辑视图 | 避免多次拷贝拼包 |
| 应用层 | `slice()` / `duplicate()` | 共享底层内存，不复制 |
| 应用层 | `FileRegion` | 文件传输走 `sendfile`，不经过用户态 |
| 系统层 | `transferTo` / `sendfile` | 减少内核态与用户态拷贝 |

## 7.1 池化分配器

Netty 默认使用 `PooledByteBufAllocator`，按 arena + chunk 的方式复用内存，避免频繁 `malloc` 和 GC。

```java
// 显式指定池化分配器（生产环境推荐）
bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

**注意**：池化带来引用计数问题。`retain()` 和 `release()` 必须成对，跨线程传递 `ByteBuf` 时尤其容易多释放或少释放，表现为内存泄漏或 `IllegalReferenceCountException`。

# 8. 心跳与空闲检测

长连接必须处理"对端已经死了但 TCP 还没感知"的情况，通常用空闲检测 + 心跳包。

```java
// 10 秒没读、10 秒没写、10 秒没读写，就触发 IdleStateEvent
ch.pipeline().addLast(new IdleStateHandler(10, 10, 10, TimeUnit.SECONDS));
ch.pipeline().addLast(new HeartbeatHandler());
```

```java
public class HeartbeatHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            if (e.state() == IdleState.READER_IDLE) {
                ctx.close();               // 读空闲，认为对端已失联
            } else if (e.state() == IdleState.WRITER_IDLE) {
                ctx.writeAndFlush(PING);   // 写空闲，主动发心跳
            }
        }
    }
}
```

| 策略 | 说明 |
|------|------|
| 只做读空闲检测 | 最简单，对端不吭声就断开 |
| 读写都检测 | 双向保活，适合 NAT 环境 |
| 业务层心跳 | 携带状态信息，顺便做探活 |

**注意**：`IdleStateHandler` 的读空闲是"多久没收到数据"，不是"多久没收到心跳包"。如果有其他业务数据流，读空闲要设得比心跳间隔大。

# 9. 常见集成场景

## 9.1 与 Spring Boot 集成

```java
@Component
public class NettyServer implements SmartLifecycle {
    private EventLoopGroup boss;
    private EventLoopGroup worker;
    private Channel channel;

    @Override
    public void start() {
        boss = new NioEventLoopGroup(1);
        worker = new NioEventLoopGroup();
        try {
            channel = new ServerBootstrap()
                    .group(boss, worker)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new YourHandler());
                        }
                    })
                    .bind(9000).sync().channel();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public void stop() {
        if (channel != null) {
            channel.close();
        }
        boss.shutdownGracefully();
        worker.shutdownGracefully();
    }
}
```

## 9.2 与 WebSocket

Netty 提供 `WebSocketServerProtocolHandler`，握手、帧解析都由它完成，业务只需要处理 `TextWebSocketFrame`。

## 9.3 与 HTTP

`HttpServerCodec` + `HttpObjectAggregator` 可以快速做一个轻量 HTTP 服务，但**不要用它替代 Tomcat**：HTTP/2、连接管理、Servlet 规范这些活，Web 容器做得更完整。

# 10. 常见坑与调优

| 问题 | 原因 | 处理 |
|------|------|------|
| 内存泄漏 | `ByteBuf` 未释放 | 用 `SimpleChannelInboundHandler` 或 `ResourceLeakDetector` 排查 |
| CPU 100% | 业务阻塞在 EventLoop | 业务逻辑丢到独立线程池 |
| 连接数上不去 | 文件句柄限制 | 调 `ulimit -n`，同时检查 `SO_BACKLOG` |
| 偶发丢消息 | 半包处理错误 | 检查解码器的 `maxFrameLength` 和长度字段偏移 |
| 客户端频繁重连 | 心跳间隔与读空闲不匹配 | 读空闲 > 心跳间隔 × 2 |

调优优先级：**先修正确性，再调参数**。`SO_BACKLOG`、`SO_RCVBUF`、`writeBufferWaterMark` 这些参数在业务逻辑有问题时调了也没意义。

# 11. 小结

| 主题 | 记住这一句 |
|------|-----------|
| 定位 | Netty 是 NIO 的工程化封装，不是 Web 容器 |
| 线程模型 | 一个 Channel 绑定一个 EventLoop，事件天然串行 |
| Pipeline | 入站从头到尾，出站从尾到头 |
| 粘包半包 | 定长、分隔符、长度字段，三选一 |
| 内存 | 池化 + 引用计数，`release()` 必须成对 |
| 长连接 | 空闲检测 + 心跳包，缺一不可 |

Netty 的难点不在 API，而在"异步 + 引用计数 + 线程模型"三者的组合。把这三块吃透，再看 Dubbo 的 `NettyServer`、RocketMQ 的 `NettyRemotingServer`，会发现它们只是在这套骨架上填业务。

回到本模块的主线：前面几篇解决的是 Servlet 规范下的请求处理，[10. Tomcat](10_tomcat.md) 解决的是"请求如何被接收"，这一篇解决的是"字节如何被收发"。三者合起来，才是一条完整的 JavaWeb 链路。
