阻塞式 Socket 的问题在于"一条连接一个线程"：连接空闲时线程也跟着空转，连接数上到几千，线程切换和栈内存就成了瓶颈。NIO（Non-blocking I/O，非阻塞 I/O）换了个思路——用少量线程通过 `Selector` 统一问内核"哪些连接有数据了"，只处理就绪的，其余一律不占用线程。本篇讲清 Channel、Buffer、Selector 三件套，写出一个不阻塞的事件循环，并说明直接内存与半包处理的注意点。

# 1. 三种 I/O 模型

## 1.1 BIO：阻塞式

BIO（Blocking I/O）就是上一篇的写法：每个 `accept()`、`read()`、`write()` 都可能把线程挂起。

```text
线程 1 ──> 连接 A：read 阻塞 ────────────────>
线程 2 ──> 连接 B：read 阻塞 ────────────────>
线程 3 ──> 连接 C：read 阻塞 ────────────────>
```

优点是编程模型直观，适合连接数少、请求处理耗时的场景。缺点是线程数与连接数成正比，连接数一多，内存和上下文切换成本迅速上升。

## 1.2 NIO：非阻塞 + 多路复用

NIO 的核心是**就绪通知**：把大量连接注册到一个 `Selector` 上，一个线程调用 `select()` 等待，内核告诉它"这几个连接可读、那个连接可接受新连接"，然后只处理这些就绪的。

```text
                ┌──────────────────────────┐
  连接 A ──────>│                          │
  连接 B ──────>│        Selector          │──> 就绪集合
  连接 C ──────>│  （内核维护的 fd 集合）     │      │
  连接 D ──────>│                          │      ▼
                └──────────────────────────┘   单线程处理就绪事件
```

一个线程可以管理成千上万条连接，空闲连接不消耗线程。

## 1.3 AIO：异步回调

AIO（Asynchronous I/O）进一步把"读"也变成异步：调用 `read()` 时传入回调，内核完成拷贝后通知你，连"先 select 再 read"这两步都省了。Java 7 引入 `AsynchronousSocketChannel` 和 `AsynchronousServerSocketChannel`。

## 1.4 对比

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| 阻塞点 | accept/read/write | select 阻塞，读写不阻塞 | 全部异步 |
| 线程模型 | 一连接一线程 | 少量线程管多连接 | 回调线程池 |
| 编程难度 | 低 | 中高 | 高 |
| 连接数上限 | 千级 | 万级以上 | 万级以上 |
| 典型框架 | Tomcat BIO | Netty、Tomcat NIO | 较少使用 |
| 适用场景 | 内部工具、低并发 | 高并发服务端 | 特定平台（Linux epoll AIO） |

**设计考量**：AIO 在 Linux 上长期依赖 epoll 模拟（真正的 io_uring 支持较新），框架生态也远不如 NIO 成熟。Netty 选择 NIO 而非 AIO，说明工程上"够用且可控"比"理论更异步"更重要。

# 2. Channel 与 Buffer

## 2.1 Channel 的类型

`Channel` 是 NIO 的连接抽象，类似 `Socket`，但支持非阻塞模式。

| Channel | 对应 | 用途 |
|---------|------|------|
| `ServerSocketChannel` | `ServerSocket` | 监听、接受连接 |
| `SocketChannel` | `Socket` | 客户端连接、收发数据 |
| `DatagramChannel` | `DatagramSocket` | UDP |
| `FileChannel` | 文件 | 文件读写、`transferTo` 零拷贝 |

TCP 场景只用前两个。把 `SocketChannel` 配置成非阻塞后，`read()` 和 `write()` 立即返回，可能读不到任何数据。

## 2.2 Buffer 的四个指针

`Buffer` 是一块内存加上四个下标。理解这四个下标，就理解了 `ByteBuffer` 的所有古怪 API。

```text
 0        position          limit        capacity
 ├──────────┼────────────────┼─────────────┤
 │ 已读/已写 │   可读写区域    │   不可用     │
 └──────────┴────────────────┴─────────────┘
            mark 可以在其中某个位置被记住
```

| 属性 | 含义 |
|------|------|
| `capacity` | 缓冲区总容量，创建后不变 |
| `position` | 下一个读或写的位置 |
| `limit` | 可读写的边界 |
| `mark` | 临时记住的位置，`reset()` 回到这里 |

刚分配时 `position=0`、`limit=capacity`，处于**写模式**。写入 n 个字节后 `position=n`。要让别人读这段数据，必须调用 `flip()`。

## 2.3 flip、clear 与 compact

这是初学者最容易混淆的三个方法：

```java
ByteBuffer buf = ByteBuffer.allocate(1024);

buf.put("hello".getBytes(StandardCharsets.UTF_8)); // position=5, limit=1024
buf.flip();                                        // limit=5, position=0，切到读模式
while (buf.hasRemaining()) {
    byte b = buf.get();                            // 读完 position=5
}
buf.clear();                                        // position=0, limit=1024，切回写模式
```

| 方法 | 做了什么 | 何时用 |
|------|----------|--------|
| `flip()` | `limit=position; position=0` | 写完之后准备读 |
| `clear()` | `position=0; limit=capacity` | 数据已读完，准备重新写 |
| `compact()` | 把未读数据移到开头，`position=剩余长度` | 只读了一部分，剩下要留着继续拼 |

**注意**：`clear()` 不会清空内存内容，只是重置指针。忘记 `flip()` 直接读，会读到 `position` 之后的空白；忘记 `clear()` 继续写，会得到 `BufferOverflowException`。

# 3. Selector 与事件循环

## 3.1 注册与就绪事件

```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(9000));
server.configureBlocking(false);
// 注册"接受连接"事件，attachment 可以挂任意上下文对象
server.register(selector, SelectionKey.OP_ACCEPT);
```

四种事件：

| 事件 | 常量 | 触发条件 |
|------|------|----------|
| 接受连接 | `OP_ACCEPT` | 有新的三次握手完成 |
| 可读 | `OP_READ` | 接收缓冲区有数据或对端关闭 |
| 可写 | `OP_WRITE` | 发送缓冲区有空闲 |
| 连接完成 | `OP_CONNECT` | 客户端握手完成 |

`ServerSocketChannel` 只能注册 `OP_ACCEPT`，`SocketChannel` 不能注册 `OP_ACCEPT`。

## 3.2 SelectionKey

每次 `select()` 返回后，`selectedKeys()` 里是就绪的 key。每个 key 携带三种信息：

- `channel()`：对应的 Channel。
- `readyOps()`：这次就绪的事件。
- `attachment()`：注册时挂的对象，通常用来保存该连接的读写缓冲区与状态。

## 3.3 事件循环骨架

事件循环（event loop）是一个 `while` 循环：等待就绪、分发处理、清理。所有 NIO 框架的核心都是它。

```text
       ┌──────────────────────────────────────┐
       │                                      │
       ▼                                      │
  select() 阻塞等待 ──> 遍历就绪 key ──> 分发处理 ─┘
       │
       └──> 处理完毕，回到 select
```

```java
while (running) {
    int n = selector.select();        // 阻塞直到有事件，或超时
    if (n == 0) {
        continue;                     // 超时没有事件
    }
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();                  // 必须移除，否则下次还会被处理
        if (!key.isValid()) {
            continue;
        }
        if (key.isAcceptable()) {
            handleAccept(key);
        } else if (key.isReadable()) {
            handleRead(key);
        }
    }
}
```

**注意**：`selectedKeys()` 返回的集合不会自动清空，处理完必须 `it.remove()`。漏掉这一行，同一个 key 会被反复处理，典型症状是 CPU 飙高、日志重复打印。

# 4. 非阻塞服务端示例

## 4.1 完整代码

下面是一个把收到的数据原样回写的 NIO 服务端：

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;

public class NioEchoServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();

        ServerSocketChannel server = ServerSocketChannel.open();
        server.bind(new InetSocketAddress(9000));
        server.configureBlocking(false);                 // 必须非阻塞才能注册
        server.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO 服务端启动在 9000");

        while (true) {
            // 阻塞等待就绪事件；没有事件时线程睡眠，不占 CPU
            selector.select();
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();
                try {
                    if (key.isAcceptable()) {
                        accept(selector, key);
                    } else if (key.isReadable()) {
                        read(key);
                    }
                } catch (IOException e) {
                    // 单条连接异常，关闭它，不影响其他连接
                    key.cancel();
                    closeQuietly(key.channel());
                }
            }
        }
    }

    private static void accept(Selector selector, SelectionKey key) throws IOException {
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel client = server.accept();   // 非阻塞，没有连接时返回 null
        if (client == null) {
            return;
        }
        client.configureBlocking(false);
        // 每条连接独立一个缓冲区，挂在 attachment 上
        client.register(selector, SelectionKey.OP_READ, ByteBuffer.allocateDirect(4096));
        System.out.println("新连接: " + client.getRemoteAddress());
    }

    private static void read(SelectionKey key) throws IOException {
        SocketChannel client = (SocketChannel) key.channel();
        ByteBuffer buf = (ByteBuffer) key.attachment();
        buf.clear();                               // 准备写入新数据
        int n = client.read(buf);
        if (n == -1) {                             // 对端关闭
            key.cancel();
            client.close();
            System.out.println("连接关闭: " + client.getRemoteAddress());
            return;
        }
        if (n == 0) {
            return;                                // 没有数据，等下次事件
        }
        buf.flip();                                // 切到读模式
        client.write(buf);                         // 回写（简单场景下通常一次写完）
        // 若 write 返回 0，说明发送缓冲区满，需要注册 OP_WRITE 稍后继续
    }

    private static void closeQuietly(Channel ch) {
        try {
            ch.close();
        } catch (IOException ignored) {
            // 关闭失败无需处理
        }
    }
}
```

## 4.2 半包处理

上面的代码直接回写"这一次读到的字节"，在真实协议里并不安全。TCP 是字节流，一次 `read()` 可能只读到半条消息，也可能读到一条半。

```text
发送方: [消息1][消息2][消息3]
接收方第一次 read: [消息1][消息2的前半]
接收方第二次 read: [消息2的后半][消息3]
```

NIO 里的正确做法是给每条连接配一个**累积缓冲区**：读到的数据先追加进去，再循环尝试解析出完整消息，解析不出就保留到下次。

```java
private static void read(SelectionKey key) throws IOException {
    SocketChannel client = (SocketChannel) key.channel();
    ByteBuffer buf = (ByteBuffer) key.attachment();
    int n = client.read(buf);              // 写模式，追加在已有数据之后
    if (n == -1) { key.cancel(); client.close(); return; }

    buf.flip();
    while (buf.remaining() >= 4) {         // 协议：4 字节长度 + 消息体
        buf.mark();
        int len = buf.getInt();
        if (len < 0 || len > 1024 * 1024) { // 长度字段必须校验
            key.cancel(); client.close(); return;
        }
        if (buf.remaining() < len) {
            buf.reset();                   // 不足一条完整消息，回退等待
            break;
        }
        byte[] body = new byte[len];
        buf.get(body);
        System.out.println("收到: " + new String(body, StandardCharsets.UTF_8));
    }
    buf.compact();                         // 把未解析的半包移到开头，切回写模式
}
```

`mark/reset` 与 `compact` 的配合是半包处理的标准套路，第 4 篇会展开成完整的编解码器。

# 5. 直接内存

## 5.1 堆内与堆外

`ByteBuffer.allocate(n)` 分配在 Java 堆上，受 GC 管理；`ByteBuffer.allocateDirect(n)` 分配在**直接内存**（堆外内存），不受堆大小限制，但仍受 `-XX:MaxDirectMemorySize` 约束。

| 维度 | 堆内 Buffer | 直接 Buffer |
|------|-------------|-------------|
| 分配/释放成本 | 低，GC 负责 | 高，需显式或 Cleaner 释放 |
| 读写性能 | 需要一次拷贝到内核 | 可省去用户态到内核态的一次拷贝 |
| 适用 | 短生命周期、小数据 | 频繁 I/O、长连接缓冲区 |
| 监控 | `jmap`/堆 Dump 可见 | 需看 `BufferPoolMXBean` 或 NMT |

**设计考量**：直接内存之所以快，是因为 JVM 在做 I/O 时可以把堆外地址直接交给内核，避免"堆内 → 临时直接内存 → 内核"的拷贝。代价是分配昂贵、回收不可控，所以适合池化复用，而不是每次请求都 `allocateDirect`。

## 5.2 释放与泄漏排查

直接内存由 `Cleaner` 在 Buffer 对象被 GC 时释放。问题在于：只要还有引用，GC 就不会发生，直接内存就一直占着。

```java
// 错误示范：反复分配直接内存却不释放引用
List<ByteBuffer> cache = new ArrayList<>();
while (true) {
    cache.add(ByteBuffer.allocateDirect(1024 * 1024));   // 很快 OOM: Direct buffer memory
}
```

排查手段：

```bash
# 观察直接内存池使用量
jcmd <pid> VM.native_memory summary

# 或通过 JMX 查看 java.nio:type=BufferPool,name=direct
```

> **行业实践**：Netty 用池化的 `PooledByteBufAllocator` 管理直接内存，按 arena 复用，避免频繁分配。自己写 NIO 服务端时，为每条连接复用一个直接 Buffer，并在连接关闭时释放引用，是最简单有效的做法。

# 6. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| Channel 未设非阻塞就注册 | `IllegalBlockingModeException` | 先 `configureBlocking(false)` |
| 忘记 `it.remove()` | 事件被重复处理，CPU 高 | 遍历时立即移除 |
| 忘记 `flip()` | 读到空白或旧数据 | 写后读前 flip |
| 用 `clear()` 丢弃半包 | 消息截断 | 用 `compact()` 保留未读部分 |
| `write()` 返回 0 未处理 | 大数据发送不完整 | 注册 `OP_WRITE` 续写 |
| 直接内存不释放 | `OutOfMemoryError: Direct buffer memory` | 复用 + 及时释放引用 |
| 在事件循环里做阻塞操作 | 所有连接卡住 | 耗时任务交给业务线程池 |

# 7. 练习与自检

**练习一**：用 `nc 127.0.0.1 9000` 连上 `NioEchoServer`，再开第二个 `nc`，验证两条连接互不阻塞。

**练习二**：把每条连接的 `ByteBuffer` 改成 `allocate(8)`，发送一条超过 8 字节的消息，观察多次 `read` 才拼出完整数据，理解半包。

**练习三**：用 `jcmd <pid> VM.native_memory summary` 或 JMX 观察直接内存随连接数增长的情况。

自检问题：为什么 `Selector` 叫多路复用？`compact()` 和 `clear()` 的区别是什么？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 三种模型 | BIO 一连接一线程，NIO 多路复用，AIO 全异步 |
| 三件套 | Channel 传输、Buffer 承载、Selector 通知就绪 |
| Buffer | 四个指针，flip 切读、clear 重置、compact 保留残留 |
| 事件循环 | select → 遍历 → 分发 → 移除 key |
| 半包 | 累积缓冲 + 循环解析 + compact |
| 直接内存 | 省一次拷贝但分配贵，适合池化复用 |

NIO 只解决了"字节怎么高效流动"，没有解决"一条消息到哪里结束"。下一篇专门讲粘包与拆包，把字节流还原成一条条业务消息。
