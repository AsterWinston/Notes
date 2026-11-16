上一篇讲的是 TCP 协议本身，这一篇把协议落到代码上：用 `Socket` 和 `ServerSocket` 建立连接、收发字节、正确关闭。阻塞式 Socket 是最古老也最直观的 Java 网络编程方式，理解了它，再看 NIO 的 `Channel` 和 Netty 的 `Channel`，就会明白它们只是把同样的动作换了一种调度方式。

# 1. Socket 是什么

## 1.1 从文件描述符到 Socket

在操作系统眼里，网络连接就是一个**文件描述符**（file descriptor，进程打开文件或套接字时内核返回的整数编号）。读写网络和读写文件用的是同一套系统调用。Java 的 `Socket` 就是这个描述符的面向对象封装。

一条 TCP 连接由四元组唯一标识：

```text
(源 IP, 源端口, 目的 IP, 目的端口)
```

服务端通常固定目的 IP 和目的端口（监听端口），靠源 IP 与源端口区分不同客户端。

## 1.2 Socket 与 TCP 连接的对应

| Java 类 | 对应协议阶段 | 主要职责 |
|---------|--------------|----------|
| `ServerSocket` | 监听 | 绑定端口、等待连接、返回 `Socket` |
| `Socket` | 一条已建立的连接 | 发起连接、读写字节、关闭 |
| `InetSocketAddress` | 地址 + 端口 | 连接目标或本地绑定地址 |
| `InputStream` / `OutputStream` | 字节流 | 实际的数据收发 |

**设计考量**：`ServerSocket` 只负责"生产连接"，真正传数据的是 `accept()` 返回的 `Socket`。把监听与通信拆成两个类，让服务端可以用一个监听端口服务无数条连接，这也是后面 NIO 里 `ServerSocketChannel` 能注册 `OP_ACCEPT`、普通 `SocketChannel` 注册 `OP_READ` 的历史源头。

# 2. 阻塞式服务端

## 2.1 最小可运行示例

先看一个能跑起来、能回显的服务端。它接收一行文本，转成大写后回写：

```java
import java.io.*;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class EchoServer {
    public static void main(String[] args) throws IOException {
        // try-with-resources 保证监听 socket 在退出时关闭
        try (ServerSocket server = new ServerSocket(9000)) {
            System.out.println("监听 9000 端口");
            while (true) {
                // accept() 阻塞，直到有客户端完成三次握手
                Socket socket = server.accept();
                handle(socket);
            }
        }
    }

    private static void handle(Socket socket) {
        // 每条连接独立关闭，避免异常时句柄泄漏
        try (socket;
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
             BufferedWriter out = new BufferedWriter(
                     new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8))) {
            String line;
            // readLine 返回 null 表示对端关闭了输出方向
            while ((line = in.readLine()) != null) {
                out.write(line.toUpperCase());
                out.newLine();   // 换行是这里约定的消息边界
                out.flush();     // 不 flush，数据可能一直留在缓冲区
            }
        } catch (IOException e) {
            // 单个连接出错不应拖垮整个服务端
            System.err.println("连接处理失败: " + e.getMessage());
        }
    }
}
```

## 2.2 accept 与连接队列

`server.accept()` 的行为依赖上一篇讲的全连接队列。当队列里已有完成握手的连接，`accept()` 立即返回；队列为空时，线程挂起。这意味着：

- 单线程 `accept()` + 单线程处理，同一时刻只能服务一条连接。
- 如果 `handle()` 里耗时很长，新连接会在队列里排队，超过 backlog 就被丢弃。

```text
         ┌───────────────────────┐
客户端 → │ 全连接队列（backlog）  │ → accept() → 业务处理
         └───────────────────────┘
```

调用 `new ServerSocket(9000)` 时，backlog 默认是 50。生产环境一般显式指定：

```java
// backlog 是"已完成握手但未被 accept 的连接"上限
ServerSocket server = new ServerSocket(9000, 1024);
```

## 2.3 为什么单线程不够用

上面的写法一次只能处理一条连接。改造方向有两种，本篇先给最直接的一种——**每连接一个线程**：

```java
Socket socket = server.accept();
// 把处理逻辑丢给线程池，主线程继续 accept
executor.submit(() -> handle(socket));
```

| 方案 | 优点 | 缺点 |
|------|------|------|
| 单线程串行 | 简单，无并发问题 | 吞吐极低 |
| 每连接一线程 | 编程简单，隔离性好 | 线程数随连接数线性增长 |
| 线程池处理连接 | 控制线程上限 | 连接空闲时也占线程，池满则排队 |

> **行业实践**：每连接一线程在连接数几百时完全够用，Tomcat 的传统 BIO 模式就是这样。连接数上万时必须换 NIO，否则光是线程栈（每线程约 1MB）就能把内存吃光。第 3 篇会详细比较这三种模型。

# 3. 客户端

## 3.1 创建与连接

```java
import java.io.*;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class EchoClient {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket();
        try (socket) {
            // 连接超时 3 秒；不设置会使用系统默认的较长超时
            socket.connect(new InetSocketAddress("127.0.0.1", 9000), 3000);
            // 读超时 5 秒；0 表示无限等待
            socket.setSoTimeout(5000);

            BufferedWriter out = new BufferedWriter(
                    new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8));
            BufferedReader in = new BufferedReader(
                    new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));

            out.write("hello tcp");
            out.newLine();
            out.flush();   // 发送边界由换行体现

            String reply = in.readLine();
            System.out.println("服务端返回: " + reply);
        }
    }
}
```

## 3.2 两个超时

`socket.connect(address, timeout)` 的参数是**连接超时**，只覆盖三次握手阶段。`socket.setSoTimeout(ms)` 是**读超时**，作用于 `read()`，超时抛 `SocketTimeoutException`，但连接不会因此关闭。

| 超时 | 作用点 | 不设置的后果 |
|------|--------|--------------|
| connect timeout | 握手阶段 | 网络不通时阻塞到系统默认值（可能数十秒） |
| SO_TIMEOUT | `read()` | 对端不发数据时永挂起 |
| 写超时 | 无原生 API | 对端不读、发送缓冲区满时阻塞 |

**注意**：Java 的阻塞 Socket 没有直接的写超时。写入可能因为发送缓冲区满而长时间阻塞，实际项目通常靠整体业务超时或异步化来解决，而不是指望 `Socket` 参数。

# 4. 输入输出流

## 4.1 read 的返回值

`InputStream.read()` 有三个重要约定，直接决定循环怎么写：

- 返回 `-1` 表示流结束（对端关闭了写方向，即收到 FIN）。
- 返回 `0` 在阻塞流中通常不会出现。
- 返回大于 0 表示读到的字节数，**不保证填满数组**。

```java
byte[] buf = new byte[1024];
int n;
while ((n = in.read(buf)) != -1) {
    // 必须用 n，不能假设一次读满 1024 字节
    out.write(buf, 0, n);
}
```

**注意**：下面这种写法是典型错误，会丢数据或出错：

```java
// 错误示范：忽略返回值，把整个缓冲区都当成有效数据
int n = in.read(buf);
out.write(buf);
```

## 4.2 字符编码

`InputStream` 和 `OutputStream` 是字节流。要处理文本，需要用 `InputStreamReader` / `OutputStreamWriter` 指定字符集。

```java
// 正确做法：两端显式约定 UTF-8
Reader r = new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8);
Writer w = new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8);
```

| 写法 | 问题 |
|------|------|
| `new InputStreamReader(in)` | 使用平台默认编码，跨环境乱码 |
| `new String(bytes)` | 同样依赖默认编码，且忽略多字节字符被截断 |
| 显式 `StandardCharsets.UTF_8` | 两端一致，可预测 |

**设计考量**：TCP 是字节流，编码是应用层的约定。如果客户端用 GBK 发、服务端用 UTF-8 解，中文就会变成乱码。安全做法是把编码写进协议文档，并在代码里显式指定，绝不依赖默认值。

## 4.3 缓冲与 flush

`BufferedWriter` 只有在缓冲区满、调用 `flush()` 或 `close()` 时才真正写出。忘了 flush，表现是"客户端发了，服务端一直收不到"。

```java
out.write("data");
out.flush();  // 没有这一行，数据可能还在用户态缓冲区
```

反过来，服务端读取时用 `BufferedReader` 可以减少系统调用次数，但它会一次读入较多数据，可能把后续消息也读进缓冲区，这属于正常现象，不改变字节流语义。

# 5. 资源关闭与异常

## 5.1 try-with-resources

`Socket`、`ServerSocket`、各条流都实现了 `AutoCloseable`。用 try-with-resources 保证异常路径也关闭：

```java
try (Socket socket = new Socket("127.0.0.1", 9000);
     InputStream in = socket.getInputStream();
     OutputStream out = socket.getOutputStream()) {
    // 使用连接
} catch (IOException e) {
    // 关闭顺序是声明的逆序，先关流再关 socket
    log.error("通信失败", e);
}
```

关闭顺序有讲究：先关输出流并 flush，再关输入流，最后关 `Socket`。try-with-resources 自动按逆序关闭正好符合这个要求。

## 5.2 close 会引发什么

对一个已经通过 HTTP 或自定义协议完成交互的连接，`close()` 会走四次挥手。如果本地还有未发送的数据，默认行为是尽量发完再关（优雅关闭）。若要立即中断，可调用 `setSoLinger(true, 0)`，内核会直接发 RST。

**注意**：调用 `close()` 之后，另一个线程里正在阻塞的 `read()` 会抛 `SocketException: Socket closed`。多线程共享同一 Socket 时要处理这个异常，不能把它当成真正的错误。

## 5.3 常见异常对照

| 异常 | 典型原因 | 处理 |
|------|----------|------|
| `ConnectException: Connection refused` | 端口没监听、被防火墙拒绝 | 检查服务端与端口 |
| `SocketTimeoutException` | 连接或读取超时 | 区分是握手还是读超时 |
| `SocketException: Connection reset` | 对端发 RST，如进程崩溃 | 重连或上报 |
| `SocketException: Broken pipe` | 对端已关闭，本地还在写 | 停止写入并关闭 |
| `BindException: Address already in use` | 端口被占用 | 改端口或排查残留进程 |

# 6. 一个简单聊天室

把上面的知识组合起来，做一个支持多客户端的广播聊天室。服务端为每条连接起一个读线程，收到消息后转发给所有在线者。

```java
import java.io.*;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ChatServer {
    // 并发集合保存所有在线连接，key 是客户端地址
    private static final Set<Socket> CLIENTS = ConcurrentHashMap.newKeySet();
    private static final ExecutorService POOL = Executors.newCachedThreadPool();

    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(9001, 1024)) {
            System.out.println("聊天室启动在 9001");
            while (true) {
                Socket socket = server.accept();
                CLIENTS.add(socket);
                POOL.submit(() -> serve(socket));
            }
        }
    }

    private static void serve(Socket socket) {
        String name = socket.getRemoteSocketAddress().toString();
        try (socket;
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = in.readLine()) != null) {
                broadcast(name + ": " + line);
            }
        } catch (IOException e) {
            System.err.println(name + " 断开: " + e.getMessage());
        } finally {
            // 无论正常退出还是异常，都要从在线列表移除
            CLIENTS.remove(socket);
            broadcast(name + " 离开了聊天室");
        }
    }

    private static void broadcast(String text) {
        byte[] data = (text + "\n").getBytes(StandardCharsets.UTF_8);
        for (Socket client : CLIENTS) {
            try {
                OutputStream out = client.getOutputStream();
                out.write(data);
                out.flush();
            } catch (IOException e) {
                // 单个客户端发送失败不影响其他人
                CLIENTS.remove(client);
            }
        }
    }
}
```

这段代码有两个值得注意的地方：

- `CLIENTS` 用了 `ConcurrentHashMap.newKeySet()`，因为多个读线程会并发增删。
- `broadcast` 里每个客户端单独 try-catch，一条坏连接不能导致整个广播中断。

**注意**：这个聊天室没有加锁保护多线程同时写同一个 `OutputStream`。若两个线程同时 `write`，字节可能交错。真实实现应为每个客户端配一个发送队列和单独写线程，或对 `out` 加同步。这也正是第 3 篇 NIO 与第 4 篇编解码要解决的问题。

# 7. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 漏写 `flush()` | 数据滞留，对端收不到 | 写完立即 flush |
| 用 `read()` 返回值当数组长度 | 数据错位、丢字节 | 只用前 n 个字节 |
| 不设 `SO_TIMEOUT` | 单连接挂死拖住线程 | 设置合理读超时 |
| 依赖平台默认编码 | 跨平台乱码 | 显式 UTF-8 |
| `accept()` 后不关闭 `Socket` | 句柄泄漏，`CLOSE_WAIT` 堆积 | try-with-resources |
| 在 `accept()` 线程里处理业务 | 新连接全部排队 | 交给线程池 |
| 多线程直接写同一输出流 | 字节交错 | 串行化或加锁 |

# 8. 练习与自检

**练习一**：改写 `EchoServer`，支持同时处理 10 个客户端。用 `telnet 127.0.0.1 9000` 或 `nc` 开多个窗口测试，观察是否互相阻塞。

**练习二**：给客户端设置 1 秒读超时，连接一个从不回数据的服务端，捕获 `SocketTimeoutException`，并验证超时后连接仍然可用。

**练习三**：故意在 `handle()` 里不关闭 `Socket`，用 `ss -tan state close-wait` 观察堆积，再改成 try-with-resources 对比。

自检问题：`connect` 超时和 `SO_TIMEOUT` 分别覆盖哪个阶段？为什么 `read(buf)` 一次不一定读满？

# 9. 小结

| 主题 | 要点 |
|------|------|
| Socket 模型 | 监听 socket 生产连接，连接 socket 负责收发 |
| 阻塞语义 | `accept()`、`read()` 都会挂起线程 |
| 流处理 | 必须使用 `read()` 返回值，文本显式指定编码 |
| 超时 | 连接超时与读超时是两回事，写超时需业务兜底 |
| 关闭 | try-with-resources 逆序关闭，避免句柄泄漏 |
| 并发 | 单线程串行吞吐低，每连接一线程受连接数限制 |

阻塞式模型简单直接，但"一条连接一个线程"的天花板很低。下一篇进入 NIO，用 `Channel`、`Buffer` 和 `Selector` 让一个线程管理成千上万条连接。
