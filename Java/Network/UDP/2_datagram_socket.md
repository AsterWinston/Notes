上一篇讲清了 UDP 的协议语义，这一篇把它落到 Java 代码上。UDP 编程比 TCP 简单得多：没有 `accept()`，没有连接，只有两个类——`DatagramSocket` 负责收发，`DatagramPacket` 承载一个数据报。但简单不等于没有坑：缓冲区被截断、报文长度用错、编码不一致、接收线程永久阻塞，这些问题在 UDP 里不会以异常的形式出现，而是表现为"数据莫名其妙少了几个字节"。

# 1. DatagramSocket 与 DatagramPacket

## 1.1 两个核心类

Java 把 UDP 的收发抽象成两个类，职责划分非常清晰：

| 类 | 职责 | 关键方法 |
|----|------|----------|
| `DatagramSocket` | 绑定本地端口、发送与接收数据报 | `send()`、`receive()`、`setSoTimeout()`、`close()` |
| `DatagramPacket` | 承载一个数据报，含数据、长度、远端地址 | `getData()`、`getLength()`、`getAddress()`、`getPort()` |
| `InetAddress` | IP 地址的抽象 | `getByName()`、`getLocalHost()` |

`DatagramSocket` 相当于一个"信箱"，绑定在某个本地端口上；`DatagramPacket` 是一封信，信封上写着收件地址，里面装着字节数据。发信时把信交给信箱，收信时从信箱里取出一封完整的信。

```text
┌────────────────────┐          ┌────────────────────┐
│  DatagramPacket    │  send()  │  DatagramSocket    │
│  data + 目标地址    │ ───────► │  绑定本地端口        │
└────────────────────┘          └─────────┬──────────┘
                                          │ receive()
                                          ▼
                                ┌────────────────────┐
                                │  新的 DatagramPacket │
                                │  data + 来源地址     │
                                └────────────────────┘
```

**注意**：`DatagramSocket` 和 TCP 的 `Socket` 名字相似，但没有连接状态。调用 `socket.connect(address)` 只是设置一个默认远端地址，让后续 `send()` 不必每次指定目标，并让内核过滤掉其他来源的报文，它不是握手。

## 1.2 DatagramPacket 的两个身份

同一个 `DatagramPacket` 类，在发送和接收时的语义完全不同。发送时你是构造者，要提供数据和目标地址：

```java
byte[] data = "hello".getBytes(StandardCharsets.UTF_8);
InetAddress addr = InetAddress.getByName("127.0.0.1");
// 构造时指定：数据、长度、目标 IP、目标端口
DatagramPacket packet = new DatagramPacket(data, data.length, addr, 9000);
socket.send(packet);
```

接收时只提供一个空缓冲区，地址和端口由 `receive()` 填充：

```java
byte[] buf = new byte[2048];
DatagramPacket packet = new DatagramPacket(buf, buf.length);
socket.receive(packet);   // 阻塞，直到收到一个数据报

// 必须用 getLength() 取实际收到的字节数，不能用 buf.length
String text = new String(packet.getData(), packet.getOffset(), packet.getLength(),
                         StandardCharsets.UTF_8);
```

**设计考量**：`getData()` 返回整个底层数组，可能比实际数据长；保留 `getOffset()` 和 `getLength()` 是为了支持从大缓冲区切包的零拷贝用法。初学者最容易在这里出错，因为 `new String(buf)` 不会抛异常，只会静默地把尾部空字节也解成字符。

## 1.3 与 TCP Socket 的差异

| 维度 | TCP `Socket` | UDP `DatagramSocket` |
|------|--------------|----------------------|
| 建立连接 | `connect()` 触发握手 | 不需要 |
| 收发接口 | `InputStream` / `OutputStream` | `send()` / `receive()` |
| 数据单位 | 字节流 | 数据报 |
| 消息边界 | 需应用层处理 | 天然保留 |
| 服务端 | `ServerSocket.accept()` | 一个 socket 收所有来源 |
| 超时 | `setSoTimeout` 影响 `read` | `setSoTimeout` 影响 `receive` |
| 对端感知 | 连接断开可感知 | 无感知，需心跳 |

UDP 服务端不需要为每个客户端创建新 socket。所有客户端的数据报都进入同一个 socket 的接收队列，靠 `packet.getAddress()` 和 `getPort()` 区分来源。

# 2. 最小可运行示例

## 2.1 回声服务端

先写一个能跑起来的回声服务：收到任意文本，转成大写后原样发回。

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.nio.charset.StandardCharsets;

public class UdpEchoServer {
    public static void main(String[] args) throws Exception {
        // 绑定 9000 端口；不指定地址表示监听所有网卡
        try (DatagramSocket socket = new DatagramSocket(9000)) {
            System.out.println("UDP 回声服务启动在 9000");
            byte[] buf = new byte[2048];
            while (true) {
                // 复用同一个 packet，receive 会重置长度和来源地址
                DatagramPacket in = new DatagramPacket(buf, buf.length);
                socket.receive(in);

                // 必须用 getLength()，不能直接用 buf.length
                String text = new String(in.getData(), in.getOffset(), in.getLength(),
                                         StandardCharsets.UTF_8);
                byte[] outData = text.toUpperCase().getBytes(StandardCharsets.UTF_8);
                // 回包地址就是刚才收到包的来源地址
                socket.send(new DatagramPacket(outData, outData.length,
                        in.getAddress(), in.getPort()));
                System.out.println("来自 " + in.getAddress() + ":" + in.getPort()
                        + " -> " + text);
            }
        }
    }
}
```

`in.getAddress()` 和 `in.getPort()` 在 `receive()` 之后才有效。发送时如果不指定目标地址，数据报会被丢弃或抛出异常。

## 2.2 客户端

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.nio.charset.StandardCharsets;

public class UdpEchoClient {
    public static void main(String[] args) throws Exception {
        // 不指定端口，内核分配一个临时端口用于接收回包
        try (DatagramSocket socket = new DatagramSocket()) {
            socket.setSoTimeout(2000);   // 2 秒读超时，避免永久阻塞

            InetAddress server = InetAddress.getByName("127.0.0.1");
            byte[] data = "hello udp".getBytes(StandardCharsets.UTF_8);
            socket.send(new DatagramPacket(data, data.length, server, 9000));

            byte[] buf = new byte[2048];
            DatagramPacket response = new DatagramPacket(buf, buf.length);
            socket.receive(response);    // 超时会抛 SocketTimeoutException

            String reply = new String(response.getData(), response.getOffset(),
                                      response.getLength(), StandardCharsets.UTF_8);
            System.out.println("服务端返回: " + reply);
        }
    }
}
```

客户端**必须绑定端口**才能收到回包——不显式指定时，`new DatagramSocket()` 会让内核分配一个**临时端口**（ephemeral port，通常是 49152 到 65535），`receive()` 就在这个端口上等待。分别用 `javac` 和 `java` 运行两个类即可测试，也可以用 `printf 'hello udp' | nc -u -w1 127.0.0.1 9000` 快速验证。

# 3. 发送与接收的细节

## 3.1 send 与 receive 的语义

`send()` 是**非阻塞**的：它把数据报交给内核发送缓冲区就返回，不代表报文已经离开网卡，更不代表对方收到。发送缓冲区满时，UDP 的默认行为是**丢弃新报文**，而不是阻塞等待。`receive()` 是**阻塞**的：接收队列为空时线程挂起，直到有报文到达或超时；接收队列满时，新到达的报文被内核直接丢弃，双方都不会收到通知。

```text
发送路径：应用 ── send() ──► 内核发送缓冲区 ──► 网卡 ──► 网络
                             满了就丢，不阻塞
接收路径：网络 ──► 网卡 ──► 内核接收队列 ──► receive() ──► 应用
                             满了就丢，不通知
```

**设计考量**：UDP 的"丢"发生在两个地方——网络中和本机队列中。后者常被忽略：如果 `receive()` 处理太慢（比如在循环里做数据库写入），接收队列会迅速填满，后续报文还没出网卡就被丢掉。解决方式是把接收和处理解耦：接收线程只负责收包并投递到队列，业务线程池消费。

## 3.2 缓冲区长度与截断

`receive()` 时传入的缓冲区如果小于报文长度，会发生**截断**：多余的数据被丢弃，`getLength()` 返回缓冲区大小，报文剩余部分永久丢失，且没有任何异常。

```java
// 错误示范：缓冲区只有 16 字节，收到 100 字节的报文会被截断
byte[] small = new byte[16];
socket.receive(new DatagramPacket(small, small.length));
// getLength() == 16，后面 84 字节已经丢了

// 正确做法：按协议最大报文长度分配，并在应用层加长度字段识别异常
byte[] buf = new byte[2048];
DatagramPacket p = new DatagramPacket(buf, buf.length);
socket.receive(p);
if (p.getLength() == buf.length) {
    System.err.println("缓冲区已满，可能被截断");   // 无法百分百断定
}
```

**注意**：无法通过 `DatagramPacket` 判断是否真的发生了截断。`getLength() == buf.length` 既可能是报文刚好这么长，也可能是被截断。所以协议设计时应在应用层加长度字段或版本号，让接收方能识别异常报文。

## 3.3 消息边界从哪里来

UDP 保留边界，但**不告诉你一条业务消息在哪里结束**。如果一次业务消息被拆成多个数据报发送，接收方收到的仍然是多个独立的数据报，需要自己拼。反过来，如果一次 `send()` 发了 3000 字节，IP 层会分片，接收方要么收到完整的 3000 字节，要么什么都收不到。

```text
应用层消息：[ 业务消息 3000 字节 ]
                 │
                 ▼ 应用层自行分片（推荐）
        [包1 1400B][包2 1400B][包3 200B]
                 │
                 ▼ 每个包独立发送，接收方按序号重组

如果不分片：send(3000B) → IP 分片 → 丢一片 → 整个数据报丢弃
```

**设计考量**：把应用层消息控制在 MTU 以内（以太网约 1472 字节），可以避免 IP 分片。分片的风险在于只要丢一片，整个数据报都会被丢弃，而 IP 层没有重传。应用层分片则可以让重传只针对丢失的那一小片，效率高得多。

# 4. 超时与阻塞

## 4.1 setSoTimeout

`setSoTimeout(ms)` 设置 `receive()` 的最长阻塞时间，超时抛出 `SocketTimeoutException`。设为 0 表示无限等待（默认值）。

```java
DatagramSocket socket = new DatagramSocket(9000);
socket.setSoTimeout(3000);   // 3 秒
try {
    socket.receive(packet);
} catch (SocketTimeoutException e) {
    // 超时不是致命错误，socket 仍然可用，可以继续下一轮 receive
    System.out.println("3 秒内没有收到报文");
}
```

**注意**：`setSoTimeout` 影响的是**每次 `receive()` 调用**，不是整个 socket 的生命周期。超时后 socket 不会关闭，可以继续使用。

## 4.2 接收循环的写法

一个健壮的接收循环需要同时处理超时、关闭和业务异常：

```java
try (DatagramSocket socket = new DatagramSocket(9000)) {
    socket.setSoTimeout(1000);
    byte[] buf = new byte[2048];
    while (true) {
        DatagramPacket packet = new DatagramPacket(buf, buf.length);
        try {
            socket.receive(packet);
        } catch (SocketTimeoutException e) {
            continue;                 // 用超时作为检查退出标志的机会
        } catch (SocketException e) {
            if (socket.isClosed()) break;   // 正常关闭导致的中断
            throw e;                        // 真正的 socket 错误
        }
        handle(packet);              // 处理逻辑应尽量短，或投递到线程池
    }
}
```

用超时加退出标志（或者用另一个线程调用 `close()` 打断阻塞）是 UDP 服务端优雅退出的标准做法。没有超时的话，`receive()` 会一直挂着，进程无法响应关闭信号。

> **行业实践**：生产环境的 UDP 服务通常把接收与处理分离：一个或少数几个线程专职 `receive()`，收到包后放进 `ArrayBlockingQueue`，业务线程池消费。这样即使业务处理变慢，接收队列也不至于瞬间被内核丢弃，同时可以用队列长度做背压指标。

## 4.3 关闭时的异常

调用 `close()` 后，阻塞中的 `receive()` 会立即返回并抛出 `SocketException: Socket closed`。多线程环境下要把它和真正的网络错误区分开：

```java
try {
    socket.receive(packet);
} catch (SocketException e) {
    if (!socket.isClosed()) log.error("接收失败", e);   // 关闭导致的中断静默退出
}
```

# 5. 文本编码与二进制协议

## 5.1 字符串与字节数组互转

UDP 只认字节，文本必须显式编码。两端不一致就会出现乱码，而 UDP 没有 TCP 那样的连接上下文可以协商编码。

```java
// 发送端：显式 UTF-8
byte[] data = "中文消息".getBytes(StandardCharsets.UTF_8);

// 接收端：必须用同样的编码，且只取有效长度
String text = new String(packet.getData(), packet.getOffset(), packet.getLength(),
                         StandardCharsets.UTF_8);
```

| 写法 | 问题 |
|------|------|
| `"text".getBytes()` | 依赖平台默认编码，跨环境不一致 |
| `new String(bytes)` | 同样依赖默认编码，尾部空字节会变成 `\0` |
| `new String(buf)` 忽略 offset/length | 把缓冲区剩余内容一并解码 |
| 显式 `StandardCharsets.UTF_8` | 可预测，推荐 |

**设计考量**：UTF-8 是变长编码，一个中文字符占 3 字节。如果接收缓冲区刚好在一个字符中间被截断，解码会得到替换字符 `�` 而不抛异常。这说明**编码正确不等于消息完整**，完整性要靠长度字段和校验来保证。

## 5.2 长度前缀与二进制帧

文本协议用换行或分隔符定界，但 UDP 已经保留报文边界，更常见的做法是二进制帧加长度字段：

```text
┌──────────┬──────────┬──────────────┐
│ 魔数 4B   │ 长度 4B   │ 载荷 N 字节    │
│ 0xCAFEBABE│ 大端整数  │ 业务数据       │
└──────────┴──────────┴──────────────┘
```

```java
import java.nio.ByteBuffer;

// 组帧：魔数 + 长度 + 载荷
ByteBuffer buffer = ByteBuffer.allocate(8 + payload.length);
buffer.putInt(0xCAFEBABE).putInt(payload.length).put(payload);
byte[] frame = buffer.array();

// 解帧：先校验魔数和长度，再取载荷
ByteBuffer in = ByteBuffer.wrap(packet.getData(), packet.getOffset(), packet.getLength());
if (in.remaining() < 8 || in.getInt() != 0xCAFEBABE) {
    throw new IllegalArgumentException("非法报文");
}
int len = in.getInt();
if (len < 0 || len > in.remaining()) {   // 长度字段可能被伪造或损坏
    throw new IllegalArgumentException("长度字段与实际不符");
}
byte[] body = new byte[len];
in.get(body);
```

**注意**：长度字段本身也可能被伪造或损坏。解析时必须先检查 `len` 是否超过剩余字节数，否则 `new byte[len]` 可能直接触发 `OutOfMemoryError` 或 `NegativeArraySizeException`。这是二进制协议最常见的漏洞入口。

## 5.3 不要用流式思维处理 UDP

有人习惯用 `DataOutputStream` 序列化，这本身没问题，但不要把它和 TCP 的流式思维混在一起：

```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();
try (DataOutputStream out = new DataOutputStream(baos)) {
    out.writeInt(messageId);
    out.writeUTF(text);
}
byte[] data = baos.toByteArray();
socket.send(new DatagramPacket(data, data.length, address, port));
```

问题出在接收端：`DataInputStream.readUTF()` 会按需读取，如果数据报被截断，它会抛 `EOFException`。这在 UDP 里是常态而非异常，必须捕获并丢弃该报文，而不是让整个接收循环退出。

# 6. 资源管理与并发

## 6.1 DatagramSocket 的关闭

`DatagramSocket` 实现了 `AutoCloseable`，用 try-with-resources 保证释放。关闭会释放本地端口。如果关闭后立即重启服务，可能遇到 `BindException: Address already in use`——UDP 没有 `TIME_WAIT`，但端口可能被其他进程占用。排查时用 `netstat -ano | findstr :9000`（Windows）或 `ss -lunp | grep 9000`（Linux）。

## 6.2 一个 socket 能不能多线程收发

`DatagramSocket` 是线程安全的，多个线程可以同时调用 `send()` 和 `receive()`。但**多个线程同时 `receive()` 会导致报文被随机分给其中一个线程**，业务上通常不希望这样。推荐的并发模型是：

```text
┌──────────────┐   receive()   ┌──────────────┐
│ 接收线程（1 个）│ ────────────► │ 有界阻塞队列   │
└──────────────┘               └──────┬───────┘
                                      │ take()
                       ┌──────────────┼──────────────┐
                       ▼              ▼              ▼
                   工作线程 1     工作线程 2     工作线程 N
```

单接收线程保证收包顺序与去重逻辑简单；有界队列在满时可以选择丢弃或阻塞接收，形成背压；多工作线程处理业务，避免慢操作阻塞收包。

> **行业实践**：不要为了"提高吞吐"起多个 `receive()` 线程。UDP 的瓶颈通常在业务处理或网络带宽，而不是单线程收包。多个接收线程反而会让"按来源维护状态"变得复杂，比如每个客户端的序号窗口需要额外加锁。

## 6.3 端口复用

`DatagramSocket` 支持 `setReuseAddress(true)`，允许在端口仍处于关闭过程中时重新绑定，快速重启服务时有用：

```java
DatagramSocket socket = new DatagramSocket(null);
socket.setReuseAddress(true);               // 必须在 bind 之前设置
socket.bind(new InetSocketAddress(9000));
```

**注意**：`SO_REUSEADDR` 对 UDP 的语义和 TCP 不同。UDP 下它主要解决"上一个 socket 刚关闭、端口还没完全释放"的问题，不能让两个进程同时稳定地监听同一端口并各自收到全部报文（不同操作系统行为不同，不要依赖）。

# 7. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 用 `buf.length` 当实际长度 | 尾部空字节被解码，数据变脏 | 用 `packet.getLength()` |
| 忽略 `getOffset()` | 切片缓冲区时数据错位 | 用三参数构造 `String` |
| 接收缓冲区太小 | 报文被静默截断 | 按协议上限分配，加长度校验 |
| 不设 `setSoTimeout` | 接收线程永久阻塞 | 设置超时并检查退出标志 |
| 在接收循环里做重业务 | 接收队列满，内核丢包 | 收包与处理解耦 |
| 两端编码不一致 | 中文乱码 | 协议固定 UTF-8 |
| 发送超过 MTU 的大包 | IP 分片，丢一片整包失效 | 控制在 1472 字节内或应用层分片 |

# 8. 练习与自检

**练习一**：把回声服务改成支持并发客户端。用两个客户端同时发送不同内容，验证服务端能把回包正确发给各自的来源端口。

**练习二**：把客户端接收缓冲区从 2048 改成 8，发送一段 100 字节的中文，解释为什么没有异常但数据不对；再改成先读长度字段、按需分配缓冲区。

**练习三**：给服务端加关闭钩子，主线程收到 Ctrl+C 时调用 `socket.close()`，验证阻塞中的 `receive()` 能立即退出，且日志里不出现"接收失败"的误报。

自检问题：`DatagramPacket` 接收后为什么要用 `getLength()`？`setSoTimeout` 超时后 socket 还能用吗？为什么 UDP 服务端通常只需要一个接收线程？

# 9. 小结

| 主题 | 要点 |
|------|------|
| 核心类 | `DatagramSocket` 收发，`DatagramPacket` 承载数据报 |
| 发送 | 非阻塞，缓冲区满则丢弃，成功返回不代表送达 |
| 接收 | 阻塞，队列满则丢弃，必须用 `getLength()` 取实际长度 |
| 边界 | 一次发送对应一次接收，大包会被 IP 分片 |
| 超时 | `setSoTimeout` 只影响单次 `receive()`，超时后 socket 仍可用 |
| 编码 | 两端显式 UTF-8，二进制协议加魔数和长度字段 |
| 并发 | 单接收线程 + 有界队列 + 工作线程池 |

`DatagramSocket` 只提供了"发出去"和"收进来"两个动作，丢包、乱序、重复全都要业务自己面对。下一篇进入可靠 UDP 的设计：如何用序号、确认、超时重传和滑动窗口，在这层不可靠的传输之上，重新搭出一条能用的可靠通道。