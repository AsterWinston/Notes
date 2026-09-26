# 1. 知识点
- **BIO(Blocking IO)**：同步阻塞IO，一个连接对应一个线程。ServerSocket.accept()阻塞等连接，Socket.read()阻塞等数据。当连接数增加时线程数线性增加，消耗系统资源
- **BIO弊端**：每个连接创建线程造成大量线程开销和上下文切换，无法支撑海量并发连接；线程大部分时间都在等待IO，CPU利用率低
- **NIO(Non-blocking IO)**：同步非阻塞IO，JDK1.4引入，三大核心：Channel（通道）、Buffer（缓冲区）、Selector（选择器/多路复用器）
- **NIO工作原理**：一个Selector线程管理多个Channel，通过轮询就绪事件（OP_READ/OP_WRITE/OP_ACCEPT/OP_CONNECT），只在有事件时才处理，少量线程支撑大量连接
- **AIO(Asynchronous IO)**：异步非阻塞IO，JDK1.7引入（NIO.2），操作系统完成IO操作后通过回调通知。读取完成后再回调处理结果，不需要自己轮询。
- **流(Stream) vs 通道(Channel)**：流单向（InputStream/OutputStream），通道双向（可读可写）；流阻塞，通道默认阻塞但可配非阻塞；流直接读写字节，通道需通过Buffer；Channel是全双工的
- **FileChannel**：文件通道，从FileInputStream/FileOutputStream/RandomAccessFile获取；支持transferTo/transferFrom零拷贝；支持文件锁和内存映射
- **SocketChannel**：TCP客户端通道，open()创建，connect()连接
- **ServerSocketChannel**：TCP服务端通道，bind()监听端口，accept()接受连接
- **DatagramChannel**：UDP通道，send/receive
- **Buffer核心属性**：capacity（容量，固定不变）、limit（可读/写的限制位置）、position（当前读写位置）、mark（标记，reset恢复）
- **Buffer方法**：flip()（读模式: limit=position, position=0）、clear()（写模式: limit=capacity, position=0）、rewind()（重新读: position=0, limit/mark不变）、compact()（压缩未读数据到开头: position=limit-position, limit=capacity）、mark()/reset()
- **allocate vs allocateDirect**：allocate分配JVM堆内存（受GC管理，读写需拷贝到直接内存）；allocateDirect分配堆外直接内存（不受GC管理，零拷贝，创建销毁成本高），适合长生命周期的大Buffer
- **Selector**：多路复用器，通过select()方法阻塞直到有就绪事件，返回就绪Channel数量。register(Selector, ops, att)注册Channel；selectNow()非阻塞
- **select/poll/epoll**：Linux IO多路复用实现。select/poll轮询所有fd效率O(n)（1024/无限制），epoll基于事件驱动回调O(1)，JDK使用epoll（Linux）或kqueue（Mac）
- **SelectionKey**：Channel注册后返回，包含就绪事件集合（OP_READ=1<<0, OP_WRITE=1<<2, OP_CONNECT=1<<3, OP_ACCEPT=1<<4）
- **水平触发 vs 边缘触发**：水平触发(Level Trigger)数据可读时持续通知（不处理完下次select还会返回，epoll默认）；边缘触发(Edge Trigger)只在状态变化时通知一次（必须一次读完直到返回EAGAIN，JDK NIO只支持水平触发）
- **NIO空轮询Bug**：Linux下Selector的select()在无事件时应阻塞，但JDK有Bug会直接返回0（epoll空轮询），导致CPU 100%。Netty通过检测并重建Selector解决
- **Reactor线程模型**：
  - 单Reactor单线程：一个线程NIO做accept+read+process+send（Redis使用，简单高效，但单CPU无法发挥多核）
  - 单Reactor多线程：一个线程accept，线程池处理业务（可能出现业务处理快、Reactor处理慢的瓶颈）
  - 主从Reactor多线程：MainReactor只accept，SubReactor负责read+process+send（Netty采用，Kafka/SLB）
- **Netty高性能原因**：NIO Reactor线程模型、零拷贝（堆外直接内存+FileRegion+CompositeByteBuf）、锁优化（无锁串行化设计/Pipeline）、内存池（PooledByteBufAllocator复用ByteBuf）、高效序列化（Protobuf）、TCP参数优化
- **Netty核心组件**：
  - Bootstrap/ServerBootstrap：客户端/服务端启动引导，配置EventLoopGroup、Channel类型、Handler
  - EventLoopGroup：事件循环组（BossGroup处理accept, WorkerGroup处理IO读写）
  - NioEventLoop：执行所有Channel的IO操作和任务，一个NioEventLoop绑定多个Channel（Selector轮询）
  - ChannelPipeline：双向链表，责任链模式，ChannelHandler处理入站(inbound/ChannelInboundHandlerAdapter)和出站(outbound/ChannelOutboundHandlerAdapter)事件
  - ChannelHandlerContext：Handler上下文，pipeline前后通知和传递事件、属性读写
  - ChannelFuture：异步通知，Netty异步操作返回的Future（addListener注册回调）
- **Netty线程模型**：BossGroup中NioEventLoop轮询ServerSocketChannel的ACCEPT事件 → 接受连接后把SocketChannel注册到WorkerGroup的某个NioEventLoop（轮询分配） → WorkerGroup轮询已注册的SocketChannel的READ/WRITE事件 → 数据经过Pipeline的入站Handler链处理 → 写回数据经过Pipeline的出站Handler链
- **TCP粘包拆包**：
  - 原因：TCP面向流无边界、发送端Nagle算法合并小包、接收端不及时读取导致缓冲区堆积、发送数据大于接收缓冲区
  - 粘包：多个小包合并为一个TCP段
  - 拆包：一大包被分多个TCP段
- **Netty拆包解码器**：LineBasedFrameDecoder（行分隔符\r\n）、DelimiterBasedFrameDecoder（自定义分隔符如$）、FixedLengthFrameDecoder（固定长度）、LengthFieldBasedFrameDecoder（头部指定长度字段，最常用）
- **Netty适配编码器**：ByteToMessageDecoder（字节→消息，decode方法）、MessageToByteEncoder（消息→字节，encode方法）、MessageToMessageDecoder/Encoder（消息转换）；结合编解码(Codec)：ByteToMessageCodec、MessageToMessageCodec
- **零拷贝技术**：
  - mmap（内存映射）：文件映射到用户空间，减少内核到用户空间拷贝
  - sendfile：数据从文件直接发送到socket，不经过用户空间，DMA拷贝
  - Netty CompositeByteBuf：多个ByteBuf虚拟组合，避免物理复制合并
  - Netty FileRegion：包装FileChannel.transferTo实现文件传输零拷贝
- **Java序列化的问题**：不能跨语言（Java专属）、码流大（携带类元数据）、速度慢（反射遍历）。不适合分布式通信场景
- **Protobuf**：Google开源，跨语言（支持Java/C++/Python/Go等），体积小（二进制编码/varint压缩），性能高（无反射/预编译），通过.proto文件IDL定义消息结构后生成代码。Netty内置支持ProtobufVarint32FrameDecoder+ProtobufDecoder
- **Kryo**：Java高性能序列化框架，比Java序列化快10倍以上，体积更小；线程不安全，需配合对象池（如KryoPool）或在ThreadLocal中使用
- **Hessian**：轻量级跨语言RPC序列化，基于HTTP传输，二进制格式，Dubbo早起默认序列化方案

# 2. 面试题

## 2.1 BIO、NIO、AIO的区别？
| 特性 | BIO | NIO | AIO |
|------|-----|-----|-----|
| IO模型 | 同步阻塞 | 同步非阻塞 | 异步非阻塞 |
| 线程模型 | 1连接:1线程 | 1线程:N连接(Selector) | 回调/通知 |
| 触发方式 | 主动读写 | Selector轮询 | OS回调 |
| 编程复杂度 | 简单 | 复杂 | 一般 |
| 并发能力 | 低（线程数限制） | 高（十万级连接） | 高 |
| 适用场景 | 连接少且固定 | 连接多且短（聊天/Live） | 连接多且长 |

BIO是"你点的菜没做好，服务员一直等着"，NIO是"服务员定期巡查每桌需求"，AIO是"菜好了服务员会端过来告诉你"。Java AIO实际使用较少，Netty等框架更成熟。

## 2.2 NIO的三大核心组件是什么？各自的职责？
**Channel（通道）**：双向数据传输通道，负责连接和传输数据。常见实现：FileChannel（文件）、SocketChannel（TCP客户端）、ServerSocketChannel（TCP服务端）、DatagramChannel（UDP）。Channel注册到Selector进行事件监听。

**Buffer（缓冲区）**：存储数据的容器，本质是读写共用的内存块。所有数据必须经过Buffer（Channel写入Buffer → Buffer读取给程序）。属性：capacity/limit/position/mark。方法：flip()切换读写模式。

**Selector（选择器）**：多路复用器，单线程通过一个Selector管理多个Channel。Channel注册到Selector（需非阻塞模式），Selector轮询所有注册Channel的就绪事件（OP_ACCEPT/OP_READ/OP_WRITE/OP_CONNECT），然后由程序处理就绪的Channel。

三者协作流程：Channel注册到Selector → Selector.select()阻塞等待 → 有就绪事件返回SelectionKey集合 → 遍历SelectionKey → 获取就绪的Channel → Channel.read(Buffer) → 处理Buffer数据 → 清除SelectionKey防止重复处理。

## 2.3 Buffer中flip、clear、rewind的区别？
以容量为10的Buffer为例，写入5个字节后状态：
- 写入后：position=5, limit=10, capacity=10
- **flip()**（写→读切换）：limit=5, position=0, mark=-1。准备读取刚写入的5个字节数据
- **rewind()**（重读）：position=0, limit=5（不变）, mark=-1。重新读取当前limit内的数据
- **clear()**（读→写切换）：position=0, limit=10, mark=-1。准备重新写入（数据并未清除，只是位置重置）
- **compact()**（部分读取后切换写）：将未读数据(position到limit)前移到0，position=未读数据长度，limit=capacity。如position=3,limit=5 → compact后 position=2,limit=10，且原buffer[3]移到buffer[0]，buffer[4]移到buffer[1]

## 2.4 Reactor线程模型是什么？Netty用的是哪种？
Reactor模式是一种事件驱动的IO多路复用设计模式。核心是一个Reactor通过Selector轮询IO事件，派发到对应的Handler处理。

三种模型：
1. **单Reactor单线程**：一个NIO线程做accept + read + process + send，所有操作串行。Redis使用此模型（内存操作很快，不考虑IO）。缺点：无法发挥多核CPU，一个Handler阻塞影响全局
2. **单Reactor多线程**：一个Reactor线程做accept + read + send，业务process交给线程池。但Reactor线程在高并发下可能成为瓶颈（要处理accept+read+send）
3. **主从Reactor多线程**：MainReactor只处理accept，SubReactor处理read+process+send。Netty采用此模型，BossGroup=MainReactor，WorkerGroup=多个SubReactor

Netty可以进一步扩展：一个BossGroup对应多个SubReactor，SubReactor数量通常为CPU核数×2，每个SubReactor管理一组Channel。

## 2.5 Netty的核心组件和工作流程？
**核心组件**：
- **Bootstrap/ServerBootstrap**：启动引导类，用于配置和启动Netty服务
- **EventLoopGroup**：事件循环线程池，BossGroup接收连接，WorkerGroup处理IO
- **ChannelPipeline**：处理链，包含入站和出站Handler，责任链模式
- **ChannelHandler**：业务处理器，inbound处理读事件，outbound处理写事件
- **ChannelHandlerContext**：上下文，在Handler间传递数据
- **ChannelFuture**：异步操作的结果容器

**工作流程**：
1. 启动Bootstrap，配置EventLoopGroup、Channel类型、Handler
2. Bind端口，BossGroup中NioEventLoop监听OP_ACCEPT事件
3. 有连接到来，BossGroup接受连接，将SocketChannel注册到WorkerGroup
4. WorkerGroup中NioEventLoop轮询OP_READ/OP_WRITE事件
5. 事件到达，数据经过Pipeline的入站Handler链处理（如：ByteToMessageDecoder → 自定义解码 → 业务处理Handler）
6. 出站时数据经过出站Handler链编码后写回客户端

Pipeline中Handler的ctx.fireChannelRead()方法将数据传递给下一个inbound Handler。

## 2.6 TCP粘包/拆包的原因和Netty解决方案？
**原因**：
- 应用层写入数据大小 > Socket发送缓冲区大小（拆包）
- 发送数据 < MSS（TCP最大报文段长度），且有连续小数据写入，Nagle算法合并发送（粘包）
- 接收方不及时读取，多条数据在接收缓冲区堆积（粘包）
- TCP是面向流的协议，数据之间无边界

**Netty解决方案（编解码框架）**：
- **LineBasedFrameDecoder**：以换行符(\n或\r\n)为分隔符，逐行解析
- **DelimiterBasedFrameDecoder**：自定义分隔符（如$、|等特殊字符）
- **FixedLengthFrameDecoder**：固定长度消息，每次读取固定字节数
- **LengthFieldBasedFrameDecoder**：消息头包含长度字段（最常用），通过长度字段确定消息边界。参数：lengthFieldOffset(长度字段偏移)、lengthFieldLength(长度字段字节数)、lengthAdjustment(长度修正值)、initialBytesToStrip(忽略头字节数)

通常做法：LengthFieldBasedFrameDecoder解帧 + MessageToMessageDecoder/ProtobufDecoder反序列化。

## 2.7 什么是零拷贝？Java和Netty如何实现？
零拷贝不是0次拷贝，而是避免在内核空间和用户空间之间来回复制数据。

**传统IO流程**（4次copy、4次上下文切换）：磁盘→内核缓冲区(read)→用户缓冲区→内核socket缓冲区(write)→协议栈发送

**Java NIO零拷贝**：
- **FileChannel.transferTo()**（底层sendfile系统调用）：磁盘→内核缓冲区→DMA直接到网卡（2次copy，2次切换，不经过用户空间）
- **MappedByteBuffer（内存映射mmap）**：将文件直接映射到用户空间虚拟内存，读写内存即操作文件，减少一次拷贝

**Netty零拷贝**：
- **DirectByteBuf堆外内存**：数据不经过JVM堆(不GC)，直接操作操作系统内存，Socket读写直接在直接内存上进行
- **FileRegion**：包装FileChannel.transferTo()，文件直接传输不经过用户空间
- **CompositeByteBuf**：多个ByteBuf虚拟组合，逻辑上是一个完整的ByteBuf，物理上是多个，避免memcpy合并

## 2.8 Netty为什么高性能？
1. **NIO Reactor线程模型**：非阻塞IO + 事件驱动 + 少量线程管理大量连接
2. **零拷贝**：DirectByteBuf + FileRegion + CompositeByteBuf，减少数据拷贝
3. **内存池(PooledByteBufAllocator)**：复用ByteBuf减少GC压力，jemalloc算法
4. **无锁串行化设计**：一个Channel的所有事件在同一个EventLoop线程处理，无需synchronized
5. **高效的序列化**：支持Protobuf等高性能序列化框架
6. **TCP参数优化**：TCP_NODELAY关闭Nagle算法、SO_BACKLOG连接队列大小、SO_KEEPALIVE心跳
7. **规避JDK NIO Bug**：自动检测和处理epoll空轮询（触发阈值65536次后重建Selector）
8. **对象池**：Recycler轻量级对象池复用Handler/Context等对象

## 2.9 Java序列化有什么问题？有哪些替代方案？
**Java序列化的问题**：
- **不能跨语言**：必须都是Java环境
- **码流大**：序列化后携带类的元数据（包名、类名、字段名等），体积大
- **性能差**：大量反射调用，序列化速度慢
- **版本兼容难**：类结构变化后序列化兼容性差（serialVersionUID管理麻烦）
- **安全风险**：readObject()可能有反序列化漏洞

**替代方案**：

| 方案 | 跨语言 | 性能 | 码流大小 | 特点 |
|------|--------|------|----------|------|
| Protobuf | 是 | 极高 | 极小 | IDL定义，预编译，Netty内置支持 |
| Kryo | 否 | 高 | 小 | Java专用，需线程安全处理 |
| Hessian | 是 | 中 | 小 | 二进制，跨语言，Dubbo早期默认 |
| Fastjson/Jackson | 是 | 中 | 中 | JSON文本，可读性好，调试方便 |
| FST | 否 | 高 | 小 | 完全兼容Java Serialization API，性能好 |

## 2.10 Netty的ChannelPipeline责任链模式是怎样的？
ChannelPipeline是Handler的双向链表结构。Head → Handler1 → Handler2 → ... → Tail。每个节点包含一个ChannelHandler和一个ChannelHandlerContext。

**入站事件（inbound）**：从Head到Tail方向传播
- channelRegistered → channelActive → channelRead → channelReadComplete → ...
- 触发：调用`ctx.fireChannelRead(msg)`通知下一个inbound Handler

**出站事件（outbound）**：从Tail到Head方向传播
- write → flush → close → ...
- 触发：调用`ctx.write(msg, promise)`通知前一个outbound Handler

**HandlerContext**：责任链中的"节点"，控制事件传播
- `ctx.fire*()`：传播给下一个Handler处理
- `ctx.write*()`：从当前Handler向前传递出站事件
- `ctx.channel().write*()`：从Tail开始走完整出站链

Handler分为ChannelInboundHandlerAdapter（处理入站事件）和ChannelOutboundHandlerAdapter（处理出站事件），也可以一个Handler同时处理（但通常分离更清晰）。
