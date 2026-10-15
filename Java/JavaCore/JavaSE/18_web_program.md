网络编程解决的是"两台机器上的两个程序如何互相通信"的问题：一端把字节写出去，另一端把字节读进来。Java 把最基础的这套能力放在 `java.net` 包里，核心是 **`InetAddress` 定位主机、`Socket`/`ServerSocket` 走 TCP、`DatagramSocket` 走 UDP**，再往上是 **`URL` 与 `URLConnection`** 这套"最原始的 HTTP 客户端"。本篇先建立**网络分层、IP、端口**的坐标系，再把 TCP 与 UDP 两条链路各自写到可运行，最后说明 HTTP 客户端为什么应该换用 `HttpClient`。

# 1. 网络编程的地基：分层、IP 与端口

## 1.1 网络分层与 Java 的位置

网络通信是一个典型的"分层"问题：上层只管自己的语义，下层负责把数据搬到对面。教科书上常讲 OSI 七层，工程上更常用的是 **TCP/IP 四层模型**。

| 分层 | 代表协议 | Java 中对应的 API | 这一层关心什么 |
|------|----------|------------------|---------------|
| 应用层 | HTTP、FTP、SMTP、DNS、自定义协议 | `URL` / `URLConnection` / `HttpClient` | 报文的语义（请求头、响应体） |
| 传输层 | **TCP**、**UDP** | `Socket` / `ServerSocket` / `DatagramSocket` | 端到端的数据怎么送（可靠？有序？） |
| 网络层 | IP、ICMP | `InetAddress`（地址的表示） | 数据包怎么从一台主机路由到另一台 |
| 链路层 | 以太网、Wi-Fi | 无（JVM 不暴露） | 相邻设备之间怎么传帧 |

这张表最重要的结论是：**Java 的 `Socket` 直接坐在传输层之上**，网络层和链路层的活（分组、路由、重传、校验）都由操作系统内核和网卡代劳了。所以 `Socket` 的编程模型非常朴素——你拿到的就是一个**能读能写的字节流**，不用管对端中间隔了几个路由器。

```text
   ┌──────────────────────────────────────────────────────────┐
   │  应用层    HTTP / FTP / 自定义协议                         │
   │           ──▶ URLConnection / HttpClient 在这一层实现协议    │
   ├──────────────────────────────────────────────────────────┤
   │  传输层    TCP ／ UDP                                     │
   │           ──▶ Socket / DatagramSocket 直接坐在这一层        │
   ├──────────────────────────────────────────────────────────┤
   │  网络层    IP                                             │
   ├──────────────────────────────────────────────────────────┤
   │  链路层    以太网 / Wi-Fi                                  │
   └──────────────────────────────────────────────────────────┘
```

> **最佳实践**：写 Socket 程序时不要把心思花在"数据包怎么走"上，那是操作系统的事。应用层要负责的是**自己定义的消息边界**——TCP 只保证字节流有序到达，**不保证"一次 write 对应一次 read"**，这是初学者最常踩的坑（见 3.7）。

## 1.2 TCP 与 UDP 的定位

传输层只有两个主角，Java 也就只提供了两套 API。它们不是"谁更好"的关系，而是**先看业务能不能容忍丢包**。

| 维度 | TCP | UDP |
|------|-----|-----|
| 连接 | **面向连接**，通信前必须三次握手建立连接 | **无连接**，直接往目标地址扔数据包 |
| 可靠性 | 不丢、不重、按序到达（靠确认、重传、序号保证） | 不保证到达、不保证顺序、可能重复 |
| 传输方式 | **字节流**（没有消息边界，像水管） | **数据报**（一次发一个包，像寄信） |
| 开销 | 头 20 字节起，有握手/挥手、连接状态、拥塞控制 | 头仅 8 字节，无连接状态，开销极小 |
| 速度 | 相对慢（要等确认、要重传） | 快，适合"来不及就丢"的场景 |
| 广播/组播 | 不支持 | 支持（`MulticastSocket`） |
| 一对几 | 只能一对一 | 一对一、一对多、多对多都行 |
| 典型应用 | HTTP/HTTPS、文件传输、数据库连接、邮件、SSH | DNS 查询、视频直播、语音通话、游戏帧同步、心跳广播 |
| Java 核心类 | `Socket` / `ServerSocket` | `DatagramSocket` / `DatagramPacket` / `MulticastSocket` |

选择的判断标准很直接：

1. **数据一个都不能少**（转账、文件、聊天记录）→ 用 TCP；
2. **宁可丢一帧也不能卡顿**（直播画面、游戏位置同步）→ 用 UDP，丢包交给应用层自己补；
3. **要一次发给多个接收方**（局域网设备发现、行情推送）→ 用 UDP 组播。

> **注意**：TCP 建立连接需要**三次握手**（SYN → SYN+ACK → ACK）、断开需要**四次挥手**，UDP 什么都不用。本篇只需要记住"存在这两个动作、它们带来了一次往返的延迟"，握手/挥手的细节、状态机、TIME_WAIT 等属于网络协议板块的内容，这里不展开。

## 1.3 IP 地址与 InetAddress

**IP 地址**用来在网络层唯一标识一台主机。IPv4 是 32 位、点分十进制写法（`192.168.1.10`），已经接近耗尽；IPv6 是 128 位、冒号十六进制写法（`2001:db8::1`）。几个必须认识的特殊地址：

| 地址 | 含义 |
|------|------|
| `127.0.0.1` | **回环地址**（loopback），永远指向本机，不经过网卡 |
| `localhost` | 主机名，通常解析为 `127.0.0.1`（IPv6 环境下可能是 `::1`） |
| `0.0.0.0` | 通配地址，服务端绑定它表示"监听本机所有网卡" |
| `10.x.x.x`、`172.16.x.x`~`172.31.x.x`、`192.168.x.x` | **私有地址**，只在局域网内可用 |
| `224.0.0.0` ~ `239.255.255.255` | **组播地址**，用于一对多通信 |
| `255.255.255.255` | 广播地址（仅本网段） |

Java 用 `InetAddress` 表示 IP 地址，它有一个非常关键的设计：**没有 public 构造方法，无法直接 `new`**，只能通过静态工厂方法获取。原因是 `InetAddress` 内部维护了一个缓存，保证同一个主机名解析出的对象可以被复用，同时把 DNS 解析这个可能阻塞、"可能失败"的动作收敛到几个明确的入口上。

常用方法：

| 方法 | 说明 |
|------|------|
| `static InetAddress getByName(String host)` | 按主机名或 IP 字符串获取对象，**会触发 DNS 解析**，解析失败抛 `UnknownHostException` |
| `static InetAddress getLocalHost()` | 获取本机地址。部分环境下会去反查主机名，可能很慢（有真实案例卡几秒） |
| `static InetAddress[] getAllByName(String host)` | 获取一个域名对应的**所有** IP（CDN、负载均衡场景会返回多个） |
| `static InetAddress getByAddress(byte[] addr)` | 由原始字节数组构造（IPv4 是 4 字节，IPv6 是 16 字节） |
| `String getHostName()` | 获取主机名（可能触发反向 DNS 查询） |
| `String getHostAddress()` | 获取点分十进制的 IP 字符串 |
| `byte[] getAddress()` | 获取原始字节数组形式的 IP |
| `boolean isReachable(int timeout)` | 类似 ping，判断是否可达（依赖系统权限，可能不准） |
| `boolean isLoopbackAddress()` | 是否回环地址 |
| `boolean isSiteLocalAddress()` | 是否私有地址（局域网地址） |
| `boolean isMulticastAddress()` | 是否组播地址 |

```java
import java.net.InetAddress;
import java.net.UnknownHostException;

public class InetAddressDemo {
    public static void main(String[] args) throws UnknownHostException {
        // 拿本机地址：注意在某些 Linux 上这一步会去反查主机名，可能较慢
        InetAddress local = InetAddress.getLocalHost();
        System.out.println("本机 IP: " + local.getHostAddress() + "，主机名: " + local.getHostName());

        // 按域名获取（会真的去查 DNS，断网时会抛 UnknownHostException）
        InetAddress baidu = InetAddress.getByName("www.baidu.com");
        System.out.println("百度 IP: " + baidu.getHostAddress());

        // 一个域名可能对应多个 IP：CDN 就是靠这个做就近接入的
        InetAddress[] all = InetAddress.getAllByName("www.baidu.com");
        for (InetAddress addr : all) {
            System.out.println("  " + addr.getHostAddress() + " 是否私有地址: " + addr.isSiteLocalAddress());
        }

        // 直接由 IP 字符串构造，不会走 DNS
        InetAddress loopback = InetAddress.getByName("127.0.0.1");
        System.out.println("是回环地址吗: " + loopback.isLoopbackAddress());   // true
    }
}
```

> **陷阱**：`getHostName()` 和 `getByName()` 都**可能阻塞在 DNS 查询上**（网络不通时可能等数秒才抛异常），不要把它们放在高频循环或时间敏感的代码路径里。同样地，`getLocalHost()` 在容器/服务器环境下有踩坑记录，需要"本机地址"时更稳妥的做法是遍历 `NetworkInterface.getNetworkInterfaces()`。

## 1.4 端口

IP 地址定位"哪台机器"，**端口**定位"机器上的哪个程序"。端口是一个 16 位无符号整数，因此取值范围是 **0 ~ 65535**，一共 65536 个。按用途划分为三段：

| 范围 | 名称 | 说明 |
|------|------|------|
| 0 ~ 1023 | **知名端口**（well-known） | 分配给系统服务，Linux 下绑定这些端口需要 root 权限 |
| 1024 ~ 49151 | 注册端口 | 应用程序常用区间，我们写的服务端一般挑这里 |
| 49152 ~ 65535 | 动态/私有端口 | 操作系统在客户端发起连接时**临时分配**的源端口 |

常见知名端口值得记一下，排查问题时几乎天天用到：

| 端口 | 协议 | 用途 |
|------|------|------|
| 20 / 21 | TCP | FTP 数据 / 控制 |
| 22 | TCP | SSH |
| 25 | TCP | SMTP（邮件发送） |
| 53 | UDP / TCP | DNS（查询用 UDP，区域传送用 TCP） |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 6379 | TCP | Redis |
| 8080 / 8443 | TCP | 常见的备用 HTTP/HTTPS 端口（Tomcat 默认） |

有两条实践规则必须记住：

1. **一次通信由四元组唯一确定**（源 IP、源端口、目标 IP、目标端口）。所以服务端用固定端口监听时，客户端的源端口是操作系统随机分配的，同一个客户端可以同时开几十条连接到同一个服务端——四元组不同，它们就是不同的连接。
2. **同一台机器上，同一个端口在同一时刻只能被一个进程绑定**（更准确地说，同一个 `IP:端口` 组合只能被绑定一次）。重复绑定会立刻抛 `java.net.BindException: Address already in use`，这也是"服务没停干净就重启"最常见的报错。

> **行业实践**：程序里**不要把端口号写死**，更不要用 0~1023。常见做法是从配置中心或环境变量读取，本地开发给个 8080 之类的默认值。第一次写网络程序时若报 `BindException`，用 `netstat -ano | findstr 端口号`（Windows）或 `lsof -i:端口号`（Linux/macOS）查一下是谁占着。

# 2. UDP 通信

## 2.1 UDP 的编程模型

UDP 的模型和寄信一样：**写清收件人地址，把信投出去，不管对方在不在家**。它没有"连接"这个对象，收发都围绕**数据包**进行，Java 里对应两个类：

| 类 | 职责 |
|----|------|
| `DatagramSocket` | **套接字**：代表本机的一个端口，负责发和收 |
| `DatagramPacket` | **数据包**：里面装着 `byte[]` 数据 + 目标地址端口（发送时）或来源地址端口（接收后） |

一个容易被忽略的关键点：**UDP 的"一次发送"必须对应"一次接收"**。发送方调一次 `send()` 产生一个数据报，接收方就得调一次 `receive()` 来取它；如果一次 `send()` 的数据超过缓冲区大小，接收方会拿到一个被截断的包（多余部分直接丢弃，不会像 TCP 那样还能再读一次）。

## 2.2 DatagramSocket 常用 API

| 方法 / 构造 | 说明 |
|------|------|
| `DatagramSocket()` | 绑定到**任意可用端口**（多用于发送端，端口由系统随机分配） |
| `DatagramSocket(int port)` | 绑定到指定端口（**接收端必须用这个**，否则别人不知道往哪发） |
| `DatagramSocket(int port, InetAddress bindAddr)` | 绑定到指定端口和指定网卡地址 |
| `void send(DatagramPacket p)` | 发送数据包，**不阻塞**（数据交给内核就返回） |
| `void receive(DatagramPacket p)` | 接收数据包，**阻塞**直到收到为止 |
| `void close()` | 关闭套接字，释放端口 |
| `int getLocalPort()` | 获取本机绑定的端口（用无参构造时可以靠它查出系统分配了哪个） |
| `void setSoTimeout(int millis)` | 设置 `receive` 的超时；超时抛 `SocketTimeoutException` 而不是永久阻塞 |
| `void setBroadcast(boolean on)` | 允许发送广播包 |

> **注意**：`receive()` 是**阻塞方法**，不设超时的话，如果永远没有数据到达，这个线程就永远卡在那里。服务端程序里一定要配 `setSoTimeout`，否则线程池会被"卡死的线程"逐渐耗尽——这和 3.8 讲的 TCP 读写超时是同一类问题。

## 2.3 DatagramPacket 常用 API

发送用的包和接收用的包构造方式不同，这是初学最容易写反的地方：

| 用途 | 构造方法 | 说明 |
|------|----------|------|
| **发送** | `DatagramPacket(byte[] buf, int length, InetAddress address, int port)` | 指定数据、有效长度、目标地址、目标端口 |
| **发送** | `DatagramPacket(byte[] buf, int offset, int length, InetAddress address, int port)` | 带偏移量的版本，只发 `buf` 中从 `offset` 开始的 `length` 个字节 |
| **接收** | `DatagramPacket(byte[] buf, int length)` | **只指定缓冲区**，地址和端口留空，收完以后由 JVM 填进去 |

| 方法 | 说明 |
|------|------|
| `byte[] getData()` | 拿到内部缓冲区。返回的**永远是整个缓冲区**，不是实际收到的内容 |
| `int getLength()` | **实际收到的字节数**，解析数据必须用它，不能直接用 `buf.length` |
| `int getOffset()` | 数据在缓冲区中的起始偏移，通常为 0 |
| `InetAddress getAddress()` | 发送方的 IP（接收时才有值） |
| `int getPort()` | 发送方的端口（接收时才有值） |
| `void setData(byte[] buf)` | 替换缓冲区，配合 `setLength()` 可复用一个包对象 |

> **陷阱**：接收端如果写 `new String(packet.getData())`，缓冲区尾部没用到的字节（通常是 `0x00`）也会被当成字符解出来，出现一堆乱码或空白。**正确写法必须带长度**：`new String(packet.getData(), packet.getOffset(), packet.getLength(), StandardCharsets.UTF_8)`。

## 2.4 UDP 完整可运行示例

**接收端**（先启动，占住 8888 端口）：

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.nio.charset.StandardCharsets;

public class UdpReceiver {
    public static void main(String[] args) throws Exception {
        // 接收端必须绑定固定端口，否则发送方无处可投
        try (DatagramSocket socket = new DatagramSocket(8888)) {
            System.out.println("UDP 接收端已启动，监听 8888 端口，本地端口 = " + socket.getLocalPort());
            socket.setSoTimeout(30_000);   // 30 秒收不到就抛 SocketTimeoutException，避免永久阻塞

            while (true) {
                // 缓冲区大小由程序自己决定，不是协议协商出来的：一次最多收 1024 字节
                byte[] buf = new byte[1024];
                DatagramPacket packet = new DatagramPacket(buf, buf.length);
                try {
                    socket.receive(packet);   // 阻塞等待，直到有包到达
                } catch (java.net.SocketTimeoutException e) {
                    System.out.println("30 秒没有新数据，退出监听");
                    break;
                }
                // 必须用 getLength() 截取有效部分，否则会把缓冲区的填充字节也读出来
                String msg = new String(packet.getData(), packet.getOffset(),
                        packet.getLength(), StandardCharsets.UTF_8);
                System.out.println("收到来自 " + packet.getAddress().getHostAddress()
                        + ":" + packet.getPort() + " 的消息: " + msg);
            }
        }
    }
}
```

**发送端**（在接收端启动后再运行）：

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.nio.charset.StandardCharsets;

public class UdpSender {
    public static void main(String[] args) throws Exception {
        // 无参构造：源端口交给操作系统随机分配，省得和别的程序撞端口
        try (DatagramSocket socket = new DatagramSocket()) {
            String msg = "你好，UDP 接收端！";
            byte[] data = msg.getBytes(StandardCharsets.UTF_8);
            InetAddress target = InetAddress.getByName("127.0.0.1");
            // 发送用数据包：数据 + 有效长度 + 目标地址 + 目标端口
            DatagramPacket packet = new DatagramPacket(data, data.length, target, 8888);
            socket.send(packet);   // 不阻塞，扔出去就返回，不关心对方收没收到
            System.out.println("已向 " + target.getHostAddress() + ":8888 发送 "
                    + data.length + " 字节，本地源端口 = " + socket.getLocalPort());
        }
    }
}
```

运行顺序和观察要点：

1. 先跑 `UdpReceiver`，再跑 `UdpSender`，接收端会打印发送方的 `IP:端口`；
2. 如果先跑发送端再跑接收端，这一条消息**就彻底丢了**，不会有任何报错——这就是 UDP"不保证可达"的直观体现；
3. 把发送端的字符串改长（超过 1024 字节），会看到接收端收到的内容被**截断**。

## 2.5 UDP 组播（MulticastSocket）

普通 UDP 只能一对一。如果要把同一份数据发给局域网里的 100 台机器，逐个发效率很低；**组播**允许发送方只发一份，由网络设备复制给所有"加入了组播组"的接收者。`MulticastSocket` 继承自 `DatagramSocket`，额外提供加入/离开组播组的能力。组播地址范围是 **224.0.0.0 ~ 239.255.255.255**。

| 方法 | 说明 |
|------|------|
| `void joinGroup(InetAddress mcastaddr)` | 加入组播组，之后发往该组地址的数据包本机就能收到 |
| `void leaveGroup(InetAddress mcastaddr)` | 离开组播组 |
| `void setTimeToLive(int ttl)` | 设置数据包的 TTL（跳数）；0 只在同一主机，1 只在同一网段，>1 才可能被路由转发 |

**接收端**（先启动）：

```java
import java.net.DatagramPacket;
import java.net.InetAddress;
import java.net.MulticastSocket;
import java.nio.charset.StandardCharsets;

public class MulticastReceiver {
    public static void main(String[] args) throws Exception {
        InetAddress group = InetAddress.getByName("224.0.0.1");   // 组播组地址
        try (MulticastSocket ms = new MulticastSocket(8888)) {     // 组播端口
            ms.joinGroup(group);                                   // 加入组，之后才能收到
            System.out.println("已加入组播组 " + group.getHostAddress() + "，等待消息...");

            byte[] buf = new byte[1024];
            DatagramPacket dp = new DatagramPacket(buf, buf.length);
            ms.receive(dp);                                        // 阻塞接收
            System.out.println("组播收到: " + new String(dp.getData(), dp.getOffset(),
                    dp.getLength(), StandardCharsets.UTF_8));

            ms.leaveGroup(group);                                  // 离开组，释放资源
        }
    }
}
```

**发送端**：

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.nio.charset.StandardCharsets;

public class MulticastSender {
    public static void main(String[] args) throws Exception {
        InetAddress group = InetAddress.getByName("224.0.0.1");
        // 发送端用普通 DatagramSocket 就够，不必是 MulticastSocket
        try (DatagramSocket ms = new DatagramSocket()) {
            byte[] data = "组播消息：大家好".getBytes(StandardCharsets.UTF_8);
            DatagramPacket dp = new DatagramPacket(data, data.length, group, 8888);
            ms.send(dp);   // 一份数据，所有加入该组的机器都能收到（由网络设备负责复制）
            System.out.println("组播消息已发送");
        }
    }
}
```

> **注意**：组播在**公网基本不可用**，路由器默认不转发组播包。它主要用于局域网场景：集群节点发现、行情广播、配置下发。跨网段时需要交换机/路由器显式开启组播路由（IGMP），这一点在部署时经常被忽略。

## 2.6 UDP 的三个固有短板

| 短板 | 表现 | 应对 |
|------|------|------|
| **会丢包** | 网络拥塞、缓冲区满、对端没在监听，包就没了，且不通知发送方 | 应用层自己编号 + 超时重传，或接受丢失（直播场景） |
| **会乱序** | 后发的包可能先到 | 在包头带自增序号，接收端按序号重排 |
| **有大小限制** | 单个 UDP 数据报理论最大 64KB，实际受 MTU 限制通常不超过 1472 字节，超了会被 IP 层分片，任一分片丢失整包作废 | 控制单包大小，超过就应用层自己分片 |

> **行业实践**：真实的高实时场景（如游戏、音视频）几乎都是在 UDP 之上**自己实现一套可靠传输**（FEC 前向纠错 + 选择性重传 + 序号去重），因为 TCP 的"重传 + 阻塞窗口"在丢包时会引发整体停顿，反而体验更差。这类方案属于进阶话题，这里只需建立"UDP 的坑要自己填"的意识。

# 3. TCP 通信（Socket / ServerSocket）

## 3.1 TCP 通信流程

TCP 是**面向连接**的：通信之前必须先建立连接，之后双方各自持有一个 `Socket`，通过它的输入/输出流双向读写字节。服务端和客户端的角色分工是固定的：

```text
   ┌─ 服务端（Server）───────────────────┐
   │  1. new ServerSocket(9999)          │  绑定端口，进入监听状态
   │  2. accept() 阻塞                    │  等待客户端来连接
   │  3. accept() 返回【一个新的 Socket】  │  这个 Socket 才代表"这一条"连接
   │  4. 用该 Socket 的流读写              │  与客户端双向通信
   │  5. close()                          │  关闭这条连接
   └─────────────────────────────────────┘

   ┌─ 客户端（Client）───────────────────┐
   │  1. new Socket(host, 9999)          │  构造时即发起连接（三次握手）
   │  2. 用 socket 的流读写               │  双向通信
   │  3. close()                          │  关闭连接
   └─────────────────────────────────────┘
```

时序上按"谁在写"排开看最清楚（箭头始终从写的一方指向读的一方）：

```text
  ①  客户端 new Socket(host, 9999)        ──────▶  三次握手完成，服务端 accept() 返回新 Socket
  ②  客户端 getOutputStream() 写          ──────▶  服务端 getInputStream() 读
  ③  服务端 getOutputStream() 写          ──────▶  客户端 getInputStream() 读（可反复来回多次）
  ④  某一方 close()                       ──────▶  四次挥手，连接释放
```

**最需要理解的一点是 `accept()` 返回的是一个新的 `Socket`。** `ServerSocket` 只负责"听"，它本身不传输数据；每来一个客户端，`accept()` 就为这条连接创建（准确说是取出内核已完成三次握手的连接并包装出）一个**独立的 `Socket` 对象**，这个对象绑定着 `（客户端 IP:端口 ↔ 服务端 IP:端口）` 这组四元组。所以服务端会同时存在"一个 `ServerSocket` + N 个 `Socket`"，前者持续监听，后者各自服务一个客户端。

> **陷阱**：常见错误是"把 `Socket` 定义在 `while` 外面，只在 `accept()` 之后复用同一个变量"——这不是问题；真正的错误是**把 `ServerSocket` 和 `Socket` 混淆**，比如试图用 `ServerSocket` 去拿输入流。`ServerSocket` 上根本没有 `getInputStream()`。

## 3.2 Socket（TCP 客户端）API

| 构造 / 方法 | 说明 |
|------|------|
| `Socket(String host, int port)` | 创建并**立刻连接**，失败抛 `ConnectException`；这是最常用的构造 |
| `Socket(InetAddress address, int port)` | 用 `InetAddress` 对象连接 |
| `Socket()` + `connect(SocketAddress, int timeout)` | **先建对象再连接**，可以指定连接超时（首选写法） |
| `InputStream getInputStream()` | 获取输入流，用于**读**对端发来的数据 |
| `OutputStream getOutputStream()` | 获取输出流，用于**写**数据给对端 |
| `void close()` | 关闭连接，同时关闭两个流 |
| `void shutdownOutput()` | **半关闭**：禁用输出流，向对方发送 FIN，表示"我不再发数据了" |
| `void shutdownInput()` | 禁用输入流，表示"我不再读了"；残余数据被丢弃 |
| `InetAddress getInetAddress()` / `int getPort()` | 获取**对端**的 IP / 端口 |
| `int getLocalPort()` | 获取**本机**分配的源端口 |
| `boolean isConnected()` / `isClosed()` | 是否已连接 / 是否已关闭 |
| `void setSoTimeout(int millis)` | 设置读操作的超时（**只影响读**，不影响写） |
| `void setTcpNoDelay(boolean on)` | 开启 TCP_NODELAY，关闭 Nagle 算法，降低小包延迟 |
| `void setKeepAlive(boolean on)` | 开启 TCP 保活探测，用于发现"对端已经消失但没发 FIN"的情况 |

## 3.3 ServerSocket（TCP 服务端）API

| 构造 / 方法 | 说明 |
|------|------|
| `ServerSocket(int port)` | 在指定端口创建并开始监听 |
| `ServerSocket(int port, int backlog)` | `backlog`：**已完成握手但还没被 `accept()` 取走**的连接队列长度上限 |
| `ServerSocket(int port, int backlog, InetAddress bindAddr)` | 绑定到指定网卡地址，例如只监听内网 |
| `Socket accept()` | **阻塞**等待客户端连接，返回代表这条连接的**新 `Socket`** |
| `void close()` | 关闭 `ServerSocket`；**不会关闭已 `accept` 出来的那些 `Socket`** |
| `int getLocalPort()` | 获取绑定的端口（传 0 让系统分配时可用它查出来） |
| `void setSoTimeout(int millis)` | 设置 `accept()` 的超时，超时抛 `SocketTimeoutException` |
| `boolean isClosed()` | 是否已关闭 |

> **注意**：`backlog` 不是"最大并发连接数"，而是**内核已完成三次握手、等待应用调用 `accept()` 取走的队列长度**。如果应用 `accept()` 得太慢，队列满了以后新的连接会被拒绝或被丢弃（Linux 默认行为），表现为客户端偶发 `Connection reset`。真正的并发能力取决于后面 3.6 讲的线程模型。

## 3.4 TCP 完整示例：一次请求一次响应

**服务端**（先启动）：

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class TcpServer {
    public static void main(String[] args) throws IOException {
        // try-with-resources 保证 ServerSocket 一定会被关闭
        try (ServerSocket server = new ServerSocket(9999)) {
            System.out.println("服务端已启动，监听 " + server.getLocalPort() + " 端口...");

            Socket socket = server.accept();   // 阻塞在这里，直到有客户端连接上
            System.out.println("客户端 " + socket.getInetAddress().getHostAddress()
                    + ":" + socket.getPort() + " 已接入，本机服务端口 = " + socket.getLocalPort());

            try (BufferedReader reader = new BufferedReader(
                         new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
                 // 第二个参数 true = autoFlush：调用 println 时自动 flush，避免数据卡在缓冲区
                 PrintWriter writer = new PrintWriter(
                         new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true)) {

                String line = reader.readLine();   // readLine 会阻塞到读到一行（遇到 \n）
                System.out.println("收到: " + line);

                writer.println("服务端已收到: " + line);   // 回写响应，println 自带换行符
            }
            socket.close();   // 关闭这条连接（ServerSocket 仍在监听）
        }
    }
}
```

**客户端**：

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class TcpClient {
    public static void main(String[] args) throws IOException {
        // 构造这一步就在做三次握手，连不上直接抛 ConnectException
        try (Socket socket = new Socket("127.0.0.1", 9999);
             PrintWriter writer = new PrintWriter(
                     new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);
             BufferedReader reader = new BufferedReader(
                     new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8))) {

            socket.setSoTimeout(5000);              // 5 秒收不到服务端响应就超时，别死等
            writer.println("你好，服务端！");          // 必须带换行，否则对方的 readLine 一直等
            String resp = reader.readLine();
            System.out.println("服务端响应: " + resp);
        }   // 离开 try 块自动 close，触发四次挥手
    }
}
```

这段代码里有三个初学者必踩的坑：

| 坑 | 现象 | 原因 |
|----|------|------|
| 忘了 `flush()` | 服务端一直 `readLine` 等不到数据，两边一起卡死 | 输出流默认带缓冲，数据还躺在用户态缓冲区里没发出 |
| 用 `write()` 而不是 `println()`，且不发换行 | 对方 `readLine()` 永远阻塞 | `readLine()` 以换行符作为消息边界 |
| 中文乱码 | 收到的中文变成问号 | 两端字符集不一致，必须显式指定 `StandardCharsets.UTF_8` |

## 3.5 为什么服务端必须多线程

上面这个服务端的致命问题是：**`accept()` 拿到一个 `Socket` 后，主线程就陷在读写这个客户端上，没法回去再 `accept()`**。也就是说，只要第一个客户端连上来了但不发数据，第二个客户端根本连不进来——它会在 `backlog` 队列里干等，直到第一个客户端断开。

结论很直接：**服务端处理并发请求的能力 = 同时能有多少个线程各自持有一个 `Socket` 在跑**。所以标准做法是 `accept()` 之后立刻把这条连接交给一个新线程（或线程池）去处理，主线程马上回到 `accept()` 等待下一个客户端：

```text
   主线程                            工作线程 1              工作线程 2
     │                                  │                      │
     ▼                                  │                      │
 accept() ───── 返回 Socket A ────────▶│ 处理 A（读写...）      │
     │                                  │                      │
     ▼                                  │                      │
 accept() ───── 返回 Socket B ───────────────────────────────▶│ 处理 B（读写...）
     │                                  │                      │
     ▼                                  │                      │
 accept() 继续等待下一个...              │                      │
```

> **陷阱**：`new Thread(...).start()` 这种"来一个客户端开一个线程"的写法在**生产环境是禁止的**——连接数一多就会创建海量线程导致 OOM。真实项目必须用**线程池**（第 17 篇讲了它与 `Executors` 的坑），并且要有连接数上限。下面 3.6 给出的示例直接用 `ThreadPoolExecutor`。

## 3.6 多线程 / 线程池服务端示例

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TcpThreadPoolServer {
    // 显式构造线程池：有界队列 + 有界线程数，拒绝时由提交者自己执行形成背压
    private static final ExecutorService POOL = new ThreadPoolExecutor(
            4, 32, 60L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(128),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.CallerRunsPolicy());

    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(9999, 50)) {
            System.out.println("服务端已启动，端口 " + server.getLocalPort() + "，等待连接...");
            while (true) {
                // 只在主线程做 accept，一拿到连接就扔给线程池，自己立刻回去等下一个
                Socket socket = server.accept();
                POOL.execute(() -> handle(socket));
            }
        }
    }

    private static void handle(Socket socket) {
        String client = socket.getInetAddress().getHostAddress() + ":" + socket.getPort();
        try (socket;   // JDK9+ 支持 try-with-resources 引用已存在的变量，处理完自动关闭连接
             BufferedReader reader = new BufferedReader(
                     new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
             PrintWriter writer = new PrintWriter(
                     new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true)) {

            socket.setSoTimeout(10_000);            // 防止某个客户端占着连接不发数据导致线程永久阻塞
            String line;
            // 循环读取，支持一个客户端发多行（这里约定"bye"表示结束）
            while ((line = reader.readLine()) != null) {
                System.out.println("[" + client + "] " + line);
                if ("bye".equals(line)) {
                    writer.println("再见！");
                    break;
                }
                writer.println("已收到: " + line);
            }
        } catch (IOException e) {
            // 单个客户端异常不能影响整个服务端，记日志即可
            System.err.println("处理客户端 " + client + " 时出错: " + e.getMessage());
        }
    }
}
```

配套客户端（起多个实例可以观察到服务端同时处理多个连接）：

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.Socket;
import java.nio.charset.StandardCharsets;
import java.util.Scanner;

public class TcpInteractiveClient {
    public static void main(String[] args) throws Exception {
        try (Socket socket = new Socket("127.0.0.1", 9999);
             PrintWriter writer = new PrintWriter(
                     new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);
             BufferedReader reader = new BufferedReader(
                     new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
             Scanner scanner = new Scanner(System.in, StandardCharsets.UTF_8)) {

            System.out.println("已连接到服务端，输入内容后回车发送，输入 bye 退出");
            while (true) {
                String input = scanner.nextLine();
                writer.println(input);              // 发给服务端
                System.out.println("服务端: " + reader.readLine());   // 读回响应
                if ("bye".equals(input)) {
                    break;
                }
            }
        }
    }
}
```

这套模型就是 Tomcat 早期的 **BIO（Blocking IO）** 模型：一个请求一个线程，线程会阻塞在读写上。它的瓶颈很明确——线程数上不去时，并发连接数就上不去。现代服务端用 **NIO / Netty**（第 27 篇）以少量线程支撑海量连接，原理是用 `Selector` 做多路复用，而不是靠"堆线程"。

## 3.7 文件上传示例（含三个关键要点）

文件上传是 TCP 的经典练习，但它比"发一行文本"多了三个必须处理的细节，写漏任何一个都会出问题：

| 要点 | 不处理会怎样 |
|------|-------------|
| **客户端要先把文件名告诉服务端** | 服务端只能盲写成一个固定名字，多个客户端同时上传会互相覆盖 |
| **服务端要加唯一前缀** | 两个用户传了同名文件（都叫 `头像.jpg`），后传的把先传的覆盖掉 |
| **客户端要 `shutdownOutput()`** | 服务端 `while(read != -1)` 永远等不到流结束，双方一起卡死 |

第三点最需要讲清楚：TCP 是**字节流**，服务端在读文件字节时怎么知道"读完了"？如果客户端只是 `flush()`，那只是把缓冲区里的数据推出去，连接还开着，服务端会继续阻塞等待。`shutdownOutput()` 的语义是**半关闭**——"我这边不再写数据了"，内核会往对端发一个 FIN，对端读到这里就得到 `-1`（EOF），于是知道传输结束。但它不影响读：客户端仍然可以从输入流读服务端回写的"上传成功"。

```text
   ①  客户端 writeUTF("头像.jpg")     ──────▶  服务端读到原始文件名
   ②  客户端 write(文件字节...)       ──────▶  服务端加 UUID 前缀后写入磁盘
   ③  客户端 shutdownOutput()        ──────▶  发 FIN（半关闭），服务端读到 -1，知道收完了
   ④  服务端 writeUTF("上传成功")     ──────▶  客户端 readUTF() 拿到结果
   ⑤  客户端 close()                 ──────▶  连接释放（四次挥手）
```

> **陷阱**：文件名不能直接用 `BufferedReader.readLine()` 来传。因为 `BufferedReader` 会**预读**超过一行的字节到自己的缓冲区里，这些字节本来属于后面的文件内容，一旦被它吃掉就再也拿不回来了。这里改用 `DataOutputStream.writeUTF()` / `DataInputStream.readUTF()`：它先写 2 字节长度再写内容，读的时候**精确读固定字节数，不会多读一个字节**，因此和后面的原始字节流能安全衔接。

**服务端**：

```java
import java.io.BufferedOutputStream;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID;

public class FileUploadServer {
    private static final Path UPLOAD_DIR = Paths.get("uploads");

    public static void main(String[] args) throws IOException {
        Files.createDirectories(UPLOAD_DIR);   // 目录不存在就创建，避免 FileNotFoundException
        try (ServerSocket server = new ServerSocket(9999)) {
            System.out.println("文件上传服务端已启动，保存目录: " + UPLOAD_DIR.toAbsolutePath());
            while (true) {
                Socket socket = server.accept();
                new Thread(() -> handle(socket)).start();   // 演示用；生产环境请换成线程池
            }
        }
    }

    private static void handle(Socket socket) {
        try (socket;
             DataInputStream in = new DataInputStream(socket.getInputStream());
             DataOutputStream out = new DataOutputStream(socket.getOutputStream())) {

            // 1) 先读客户端发的文件名（writeUTF 保证了不会多读后面的文件字节）
            String originalName = in.readUTF();

            // 2) 加唯一前缀，避免同名文件互相覆盖（UUID 去掉横线更紧凑）
            String savedName = UUID.randomUUID().toString().replace("-", "") + "_" + originalName;
            Path target = UPLOAD_DIR.resolve(savedName);

            long total = 0;
            try (BufferedOutputStream fileOut = new BufferedOutputStream(new FileOutputStream(target.toFile()))) {
                byte[] buf = new byte[8192];       // 8KB 缓冲区：比一次一个字节快几个数量级
                int len;
                // 客户端 shutdownOutput 后这里会读到 -1，循环自然结束
                while ((len = in.read(buf)) != -1) {
                    fileOut.write(buf, 0, len);
                    total += len;
                }
            }

            // 3) 回写结果，让客户端知道成功了（不回写的话客户端也无法确认）
            out.writeUTF("上传成功: " + savedName + "，共 " + total + " 字节");
            out.flush();
            System.out.println("已保存 " + target.toAbsolutePath() + "（" + total + " 字节）");
        } catch (IOException e) {
            System.err.println("上传失败: " + e.getMessage());
        }
    }
}
```

**客户端**：

```java
import java.io.BufferedInputStream;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.FileInputStream;
import java.io.File;
import java.net.Socket;

public class FileUploadClient {
    public static void main(String[] args) throws Exception {
        File file = new File(args.length > 0 ? args[0] : "file_to_upload.txt");
        if (!file.exists()) {
            System.out.println("文件不存在: " + file.getAbsolutePath());
            return;
        }

        try (Socket socket = new Socket("127.0.0.1", 9999);
             DataOutputStream out = new DataOutputStream(socket.getOutputStream());
             DataInputStream in = new DataInputStream(socket.getInputStream());
             BufferedInputStream fileIn = new BufferedInputStream(new FileInputStream(file))) {

            // 1) 先发文件名，服务端好知道存成什么
            out.writeUTF(file.getName());

            // 2) 再发文件内容
            byte[] buf = new byte[8192];
            int len;
            long total = 0;
            while ((len = fileIn.read(buf)) != -1) {
                out.write(buf, 0, len);
                total += len;
            }
            out.flush();

            // 3) 关键：告诉服务端"我发完了"，服务端才能从 read() 拿到 -1 跳出循环
            socket.shutdownOutput();

            // 4) 半关闭只关了输出，输入还能读——服务端的回应正常收到
            System.out.println("服务端回应: " + in.readUTF() + "（本地文件 " + total + " 字节）");
        }
    }
}
```

## 3.8 超时设置：不加超时等于埋雷

网络是不可靠的，**"等一个永远不来的响应"是网络程序最典型的故障形态**。不加超时的后果不是"慢"，而是线程被永久占住：线程池的线程被一个个卡死，最终整个服务不可用。

TCP 相关的时间有三个，作用范围完全不同，必须分清楚：

| 超时设置 | 作用对象 | 不设置的后果 | 写法 |
|----------|----------|-------------|------|
| **连接超时** | `Socket` 建立连接（三次握手）阶段 | 对端 IP 不可达时，卡在内核默认超时（可能几十秒到几分钟） | `socket.connect(new InetSocketAddress(host, port), 3000)` |
| **读超时** `setSoTimeout` | 所有 `read()` 操作 | 对端连上了却不发数据，线程永久 `RUNNABLE` 卡在 `read` | `socket.setSoTimeout(5000)`，超时抛 `SocketTimeoutException` |
| **`accept` 超时** | `ServerSocket.accept()` | 主线程可能一直等不到客户端，无法做定期任务 | `server.setSoTimeout(1000)`，在循环里捕获超时 |

```java
// 连接超时 + 读超时的完整写法
Socket socket = new Socket();
try {
    // 3 秒内连不上就抛 SocketTimeoutException，不会无限等
    socket.connect(new InetSocketAddress("127.0.0.1", 9999), 3000);
    socket.setSoTimeout(5000);   // 之后每次读最多等 5 秒

    BufferedReader reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
    System.out.println(reader.readLine());
} catch (java.net.SocketTimeoutException e) {
    System.out.println("连接或读取超时: " + e.getMessage());
} finally {
    socket.close();
}
```

> **最佳实践**：**每一个网络操作都必须有超时**，且超时时间要分层。经验值是：内网调用 100ms ~ 500ms，跨机房 1s ~ 2s，第三方接口 3s ~ 5s，再加上重试与熔断。`new Socket(host, port)` 这种"裸构造"没有连接超时参数，生产代码一律用 `new Socket()` + `connect(SocketAddress, timeout)`。

> **注意**：`setSoTimeout` **只影响读，不影响写**。写操作也可能阻塞（对端接收窗口满了，发送缓冲区写不进去），这一点在阻塞式 API 里没有办法单独设超时，只能靠业务层的超时框架（如 `Future.get(timeout)`）兜底。

# 4. URL 与 URLConnection

## 4.1 URL：资源定位符

`URL`（Uniform Resource Locator，统一资源定位符）描述的是"在哪个协议下、到哪台主机的哪个端口、取哪个资源"。它比 `Socket` 高一个层次——`Socket` 只给你一条字节管道，`URLConnection` 则**替你实现了 HTTP 协议的细节**（拼请求行、解析响应头）。

| 构造 / 方法 | 说明 |
|------|------|
| `URL(String spec)` | 由一个完整字符串构造，例如 `new URL("https://www.example.com/a/b?x=1#top")` |
| `URL(String protocol, String host, int port, String file)` | 分别指定协议、主机、端口、资源路径 |
| `String getProtocol()` | 协议（`http`、`https`、`ftp` 等） |
| `String getHost()` | 主机名 |
| `int getPort()` | 端口号，**未显式指定时返回 -1**（不是默认端口） |
| `int getDefaultPort()` | 该协议的默认端口（http 是 80，https 是 443） |
| `String getPath()` | 路径部分（`/a/b`） |
| `String getFile()` | 路径 + 查询串（`/a/b?x=1`） |
| `String getQuery()` | 查询串，即 `?` 后面的部分（`x=1`） |
| `String getRef()` | 锚点，即 `#` 后面的部分（`top`） |
| `URLConnection openConnection()` | 打开连接，返回 `URLConnection` |
| `InputStream openStream()` | 简化版，等价于 `openConnection().getInputStream()`，快速读取内容 |

```java
import java.net.URI;
import java.net.URL;

public class UrlDemo {
    public static void main(String[] args) throws Exception {
        // 推荐用 URI 构造再转 URL（JDK20 起 new URL(String) 已标记为废弃）
        URL url = URI.create("https://www.example.com:8443/a/b?x=1&y=2#top").toURL();

        System.out.println("协议: "   + url.getProtocol());        // https
        System.out.println("主机: "   + url.getHost());            // www.example.com
        System.out.println("端口: "   + url.getPort());            // 8443
        System.out.println("默认端口: " + url.getDefaultPort());     // 443
        System.out.println("路径: "   + url.getPath());            // /a/b
        System.out.println("文件: "   + url.getFile());            // /a/b?x=1&y=2
        System.out.println("查询串: " + url.getQuery());           // x=1&y=2
        System.out.println("锚点: "   + url.getRef());             // top
    }
}
```

> **注意**：`getPort()` 在 URL 里没写端口时返回 **-1**，需要真实端口应该写成 `url.getPort() == -1 ? url.getDefaultPort() : url.getPort()`。这是个很低级的坑，但排查起来很费时间。

## 4.2 URLConnection / HttpURLConnection

`openConnection()` 返回的是 `URLConnection`，当协议是 HTTP/HTTPS 时，实际对象是它的子类 **`HttpURLConnection`**，可以强转后使用 HTTP 专有 API。

| 方法 | 说明 |
|------|------|
| `void connect()` | 显式建立连接。通常不必调用，`getInputStream()` 会自动触发 |
| `InputStream getInputStream()` | 读取**响应体**（状态码 2xx/3xx 时用） |
| `InputStream getErrorStream()` | 读取**错误响应体**（状态码 4xx/5xx 时，`getInputStream` 会抛异常，要用这个） |
| `OutputStream getOutputStream()` | 写入**请求体**（POST/PUT 用），需先 `setDoOutput(true)` |
| `void setRequestMethod(String method)` | 设置请求方法：GET / POST / PUT / DELETE / HEAD … |
| `void setRequestProperty(String key, String value)` | 设置**请求头** |
| `void setDoOutput(boolean doOutput)` | 是否要写请求体。**POST 必须设 `true`**，否则 `getOutputStream()` 直接抛异常 |
| `void setDoInput(boolean doInput)` | 是否要读响应体，默认 `true` |
| `int getResponseCode()` | 获取 HTTP 状态码（200、404、500…），**会触发发送请求** |
| `String getResponseMessage()` | 状态码对应的描述文本 |
| `String getHeaderField(String name)` | 获取某个**响应头** |
| `Map<String, List<String>> getHeaderFields()` | 获取所有响应头 |
| `String getContentType()` | 响应体的 MIME 类型 |
| `int getContentLength()` | 响应体长度，未知返回 -1 |
| `long getLastModified()` | 最后修改时间 |
| `void setConnectTimeout(int ms)` / `setReadTimeout(int ms)` | 连接 / 读取超时 |
| `void disconnect()` | 断开连接，释放底层 socket（建议在 `finally` 里调用） |

> **注意**：`URLConnection` 是"**最原始的 HTTP 客户端**"——它只提供最基础的请求/响应能力，没有连接池（虽然底层有 keep-alive 缓存，但控制能力很弱）、没有重试、没有自动重定向策略（3xx 默认自动跟随，但跨协议跳转不跟随）、没有 JSON 序列化、写请求体要自己拼字符串。**它就是"能跑"的下限**，而不是工程上的选择。

## 4.3 HTTP 请求示例（GET 与 POST）

**GET 请求**：

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URI;
import java.net.URL;
import java.nio.charset.StandardCharsets;

public class HttpGetDemo {
    public static void main(String[] args) throws Exception {
        URL url = URI.create("https://httpbin.org/get?name=zhangsan").toURL();
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();

        conn.setRequestMethod("GET");
        conn.setConnectTimeout(5000);                 // 连接超时
        conn.setReadTimeout(5000);                    // 读取超时
        conn.setRequestProperty("User-Agent", "Notes-Demo/1.0");            // 设置请求头
        conn.setRequestProperty("Accept", "application/json;charset=UTF-8");

        try {
            int code = conn.getResponseCode();        // 这一步才真正发出请求
            System.out.println("状态码: " + code + " " + conn.getResponseMessage());

            // 关键：状态码 >= 400 时 getInputStream() 会抛 IOException，
            // 错误响应体必须从 getErrorStream() 里读
            BufferedReader reader = new BufferedReader(new InputStreamReader(
                    code >= 400 ? conn.getErrorStream() : conn.getInputStream(),
                    StandardCharsets.UTF_8));
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                sb.append(line).append('\n');
            }
            reader.close();
            System.out.println("响应体:\n" + sb);
        } finally {
            conn.disconnect();                        // 释放连接，务必放在 finally
        }
    }
}
```

**POST 发送 JSON**：

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URI;
import java.net.URL;
import java.nio.charset.StandardCharsets;

public class HttpPostDemo {
    public static void main(String[] args) throws Exception {
        URL url = URI.create("https://httpbin.org/post").toURL();
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();

        conn.setRequestMethod("POST");
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);
        conn.setDoOutput(true);                       // 必须开启，否则 getOutputStream() 抛异常
        conn.setRequestProperty("Content-Type", "application/json;charset=UTF-8");

        String json = "{\"name\":\"张三\",\"age\":25}";
        // try-with-resources 保证流被关闭并 flush
        try (OutputStream os = conn.getOutputStream()) {
            os.write(json.getBytes(StandardCharsets.UTF_8));
        }

        try (BufferedReader reader = new BufferedReader(new InputStreamReader(
                conn.getInputStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        }
        conn.disconnect();
    }
}
```

> **陷阱**：`HttpURLConnection` 的响应体用完之后**必须关闭流**（或用 `disconnect()`），否则连接不会归还到底层缓存，大量请求会把连接数耗尽，表现为在 `getResponseCode()` 处"莫名卡住"。这是一个非常隐蔽的资源泄漏点。

## 4.4 现在应该用什么：HttpClient

JDK 11 引入了全新的 **`java.net.http.HttpClient`**，它支持 HTTP/2、异步、连接池、WebSocket，API 也更现代。JDK 自带的 HTTP 客户端里，**新代码应该直接用 `HttpClient`**；如果需要连接池精细控制、拦截器、超时重试等更强的能力，业界更常用 **OkHttp**（Android 与 Spring Cloud 生态），高层封装则是 Spring 的 `RestTemplate` / `WebClient` / `Feign`。

| 对比 | `HttpURLConnection` | `HttpClient`（JDK 11+） | OkHttp（第三方） |
|------|---------------------|------------------------|-----------------|
| 连接池 | 弱，仅 keep-alive 缓存 | **有**，可配 `Executor` 与连接池 | 有，可精细配置 |
| HTTP/2 | 不支持 | **支持** | 支持 |
| 异步 | 不支持（要自己开线程） | `sendAsync()` 返回 `CompletableFuture` | 支持 |
| 超时 | 连接 / 读两个 | 连接 / 请求 / 整体 | 连接 / 读 / 写 / 整体 |
| 请求体 / 响应体 | 手工写流 | `BodyPublishers` / `BodyHandlers` | 内置多种转换器 |
| 推荐度 | 只用于简单脚本 | **JDK 项目首选** | Android、需高级功能时 |

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.Duration;

public class HttpClientDemo {
    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5))
                .followRedirects(HttpClient.Redirect.NORMAL)   // 自动跟随重定向
                .build();                                       // 线程安全，可复用，不要每次都 new

        String json = "{\"name\":\"张三\",\"age\":25}";
        HttpRequest request = HttpRequest.newBuilder(URI.create("https://httpbin.org/post"))
                .timeout(Duration.ofSeconds(10))                // 整个请求的超时
                .header("Content-Type", "application/json;charset=UTF-8")
                .POST(HttpRequest.BodyPublishers.ofString(json, StandardCharsets.UTF_8))
                .build();

        HttpResponse<String> response = client.send(request,
                HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
        System.out.println("状态码: " + response.statusCode());
        System.out.println("响应体: " + response.body());
    }
}
```

> **行业实践**：`HttpClient` 是**线程安全**的，一个应用应该复用一个实例（内部自带连接池），而不是每个请求 `new` 一个。这和 `HttpURLConnection` 每次 `openConnection()` 都新建的开销完全不同。更上层的选择上，Java 后端项目里 HTTP 调用通常交给 Spring 体系（`RestClient` / `WebClient`）或 `Feign` 统一管理，但底层实现无非还是这几个客户端。

# 5. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 分层 | Java 的 `Socket` 直接坐在 **TCP/UDP** 之上，网络层/链路层由操作系统代劳；`URLConnection` 再往上封装了 HTTP |
| IP 与端口 | 端口 16 位，0~65535；0~1023 是知名端口；`InetAddress` **不能 `new`**，只能用 `getByName` / `getLocalHost` 等静态方法 |
| TCP 与 UDP | TCP 面向连接、可靠、有序、有开销；UDP 无连接、可能丢包乱序、开销小。**看业务能不能容忍丢包**来选 |
| TCP 流程 | `ServerSocket` 监听 → `accept()` 阻塞并返回**代表该连接的新 `Socket`** → 双向读写 → `close()`；连接需要三次握手、断开需要四次挥手 |
| TCP 并发 | 单线程 `accept` 后直接读写的服务端只能服务一个客户端；**必须把每条连接交给线程池**，生产环境禁用 `new Thread` |
| 文件上传 | 三要点：客户端先传文件名（用 `writeUTF` 避开 `BufferedReader` 预读）、服务端加 UUID 前缀防覆盖、客户端 `shutdownOutput()` 触发对方 EOF |
| 超时 | 连接超时用 `connect(SocketAddress, timeout)`，读超时用 `setSoTimeout`；**不加超时会导致线程永久阻塞**，是线上事故的常见成因 |
| UDP 细节 | 缓冲区大小自己定、`receive` 会阻塞、必须用 `getLength()` 截取有效数据、会丢包、组播地址 224.0.0.0~239.255.255.255 |
| HTTP | `URL` + `HttpURLConnection` 是最原始的 HTTP 客户端；新代码用 **`HttpClient`（JDK11+）** 或 OkHttp，生产项目多用 Spring 的封装 |

下一篇（第 19 篇）讲**反射与动态代理**——它是 Spring 这类框架能够"读配置、建对象、加增强"的底层支撑；网络 IO 的下一站是非阻塞模型，第 27 篇的 NIO `Selector` 会解释 Netty 为什么能用一个线程扛住上万连接。
