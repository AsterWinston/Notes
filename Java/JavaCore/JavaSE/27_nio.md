NIO（New I/O）是 JDK 1.4 引入的一套全新 I/O 体系，它用**通道（Channel）**加**缓冲区（Buffer）**取代了传统流式的读写方式，并用**选择器（Selector）**实现了"一个线程管理成千上万个连接"的多路复用能力。这篇讲清 BIO / NIO / AIO 三大 IO 模型的差异与各自适用场景，重点把三块最容易卡住的地方讲透：**Buffer 四个属性的状态变化全过程**、**Channel 的双向性与零拷贝**、**Selector 多路复用的事件循环与经典踩坑**。文件流的字节/字符基础见第 16 篇，`Path`/`Files` 的详细用法见第 15 篇。

# 1. 为什么需要 NIO：三种 IO 模型

## 1.1 BIO：同步阻塞 I/O

**BIO（Blocking I/O）** 是 JDK 1.0 就有的传统模型，基于 `InputStream`/`OutputStream`/`Socket`。它的特点是"**调用即阻塞**"：`accept()` 等新连接会阻塞，`read()` 等数据会阻塞，`write()` 缓冲区满也会阻塞。因此服务端只能写成"**一个连接一个线程**"：

```text
  线程1 ─▶ 连接1：阻塞在 read 上 ────────▶ 有数据才继续
  线程2 ─▶ 连接2：阻塞在 read 上 ────────▶ 有数据才继续
  线程3 ─▶ 连接3：阻塞在 read 上 ────────▶ 有数据才继续
    ……
  线程N ─▶ 连接N

  → 连接数 = 线程数；线程是昂贵资源（默认 1MB 栈），连接上万就撑不住
```

连接数一多，海量线程绝大多数时间都在**空等数据**，CPU 被白白浪费在上下文切换上。

## 1.2 NIO：同步非阻塞 + 多路复用

NIO 把"等数据"这件事整个翻转了：**不再让线程去等连接，而是让一个线程去问内核"谁准备好了"**。

1. 把 `Channel` 设为**非阻塞**（`configureBlocking(false)`），没有数据时 `read()` 立刻返回 0 而不是卡住；
2. 把关心的 `Channel` **注册**到一个 `Selector` 上，声明"我只关心它的可读/可写/可连接事件"；
3. 线程在 `selector.select()` 上**只阻塞一次**，内核返回的就绪列表里才是真正有活干的连接；
4. 遍历就绪列表逐个处理，处理完回到第 3 步。

于是 1000 个连接也只需要 1 个线程（实际会用少量线程 + 线程池处理业务），**这就是多路复用**。

## 1.3 AIO：异步非阻塞 I/O

**AIO（Asynchronous I/O，也叫 NIO 2.0）** 在 JDK 7 引入，核心 API 是 `AsynchronousFileChannel`、`AsynchronousSocketChannel`、`AsynchronousServerSocketChannel`。它和 NIO 的本质差别在于**"谁来完成数据搬运"**：

- NIO 里数据从内核到用户缓冲区这一步，仍然由**调用线程**在 `read()` 里完成（只是不再阻塞着等）；
- AIO 里这一步由**操作系统**完成后主动**回调通知**你，发起 `read()` 之后线程可以彻底去干别的，连"就绪了再去读"都不用管。

## 1.4 三种 IO 模型对比

| 维度 | BIO（同步阻塞） | NIO（同步非阻塞 / 多路复用） | AIO（异步非阻塞，NIO 2.0） |
|------|----------------|---------------------------|--------------------------|
| 出现版本 | JDK 1.0 | JDK 1.4 | JDK 7 |
| 数据单位 | 字节 / 字符流 | 块（`Buffer`） | 块（`Buffer`） |
| 是否阻塞 | `accept` / `read` / `write` 全阻塞 | 可设为非阻塞，无数据立即返回 | 全程不阻塞 |
| 谁完成 IO 搬运 | 调用线程自己 | 调用线程自己（只处理已就绪的连接） | **操作系统**，完成后回调 |
| 编程模型 | 一连接一线程 | **Reactor**（事件驱动 + 轮询） | **Proactor**（回调 / `Future`） |
| 内核机制 | `read` / `write` | `select` / `poll` / **`epoll`**（Linux） | IOCP（Windows）/ `io_uring`（Linux 新） |
| 核心 API | `Socket`、`ServerSocket` | `Channel` / `Buffer` / `Selector` | `Asynchronous*Channel` |
| 连接数上限 | 受线程数限制（千级） | 单线程可支撑**万级** | 同 NIO，理论上更高 |
| 适用场景 | 连接少、数据量大的传统应用 | **高并发、连接多但每连接数据量小**（IM、推送、网关） | 连接多且 IO 耗时长的场景，且平台要支持 IOCP |

> **注意**：Linux 上 AIO 长期只有 `io_uring` 之前的残缺实现，`epoll` 底层仍是同步的，所以 Java 的 AIO 在 Linux 上并未得到真正的内核级异步支持，实际性能优势不明显。**这也是 AIO 在服务端迟迟没有流行起来的原因**。

## 1.5 Netty 为什么选 NIO

| 选择 | 理由 |
|------|------|
| 为什么不选 BIO | 连接上万就需要上万个线程，内存与上下文切换成本直接压垮机器 |
| 为什么不选 AIO | Linux 支持不完善、性能提升不明确，且回调式编程调试困难、易出回调地狱 |
| 为什么选 NIO | `epoll` 是 Linux 上成熟且高性能的多路复用实现，Reactor 模型可扩展性好，能通过"主从多 Reactor"充分利用多核 |

> **行业实践**：Netty 是 NIO 的工业级封装，它在裸 NIO 之上补齐了**空轮询 bug 的规避、半包/粘包解码器、内存池化的 `ByteBuf`、心跳与断线重连**等能力。生产环境的网络服务端**不要直接写裸 NIO**，要么用 Netty，要么用其上的框架（Spring WebFlux、gRPC）。

# 2. BIO 与 NIO 的细节对比

| 维度 | BIO（Blocking I/O） | NIO（Non-blocking I/O） |
|------|--------------------|------------------------|
| 数据流向 | 单向（输入流 / 输出流分开） | **双向**（`Channel` 可读可写） |
| 数据处理单位 | 字节 / 字符 | 块（`Buffer`） |
| 阻塞特性 | 读写时线程必须等待 | 线程可去处理其他就绪的连接 |
| 并发支持 | 一个连接一个线程 | 一个线程管理多个连接（`Selector`） |
| 是否面向缓冲 | 面向流（Stream） | 面向缓冲区（Channel + Buffer） |
| 适用场景 | 连接数少、数据量大的场景 | 高并发、连接数多但单连接数据量小的场景 |

**一句话记忆**：BIO 是"**面向流**"的——数据像水流一样顺序前进，`read()` 一下就读走一点，读过的位置不能回头；NIO 是"**面向缓冲区**"的——数据先整块搬进 `Buffer` 这个容器，**由你决定从哪个位置读、读多少、读完之后要不要回头重读**，这也正是 `Buffer` 那四个属性存在的原因。

# 3. Buffer（缓冲区）

## 3.1 Buffer 是什么

`Buffer` 是一块**内存容器**（底层是数组），所有 NIO 读写都必须经过它：`Channel` 读数据先写进 `Buffer`，`Channel` 写数据也从 `Buffer` 取。`Buffer` 是抽象类，按数据类型分为八种子类，最常用的是 `ByteBuffer`：

| 子类 | 元素类型 | 说明 |
|------|----------|------|
| `ByteBuffer` | `byte` | **最常用**，网络与文件 IO 的基本单位 |
| `CharBuffer` / `ShortBuffer` / `IntBuffer` / `LongBuffer` / `FloatBuffer` / `DoubleBuffer` | 对应基本类型 | 处理结构化数据 |
| `MappedByteBuffer` | `byte` | 内存映射文件的 `ByteBuffer` 子类，见 4.4 |

## 3.2 Buffer 的四个核心属性

`Buffer` 内部就靠四个 int 字段决定"数据从哪来、到哪去"，简化后的结构如下：

```java
public abstract class Buffer {
    private int mark = -1;     // 标记位置，-1 表示未标记
    private int position = 0;  // 下一个要被读取 / 写入的元素索引
    private int limit;         // 第一个不应该被读 / 写的元素索引
    private int capacity;      // 总容量，创建时确定，永不改变
}
```

| 属性 | 含义 | 谁在改动它 |
|------|------|-----------|
| `capacity` | 容量，创建时固定 | **永不改变** |
| `limit` | 界限：写模式下 = `capacity`；读模式下 = 上一次写入的数据量 | `flip()`、`limit(int)` |
| `position` | 下一个要被读 / 写的下标 | `put()` / `get()` 自动 `+1`，`flip()`/`clear()` 归零 |
| `mark` | 备忘位置，配合 `reset()` 回退 | `mark()` 记录，`reset()` 回退，`flip()`/`clear()`/`rewind()` 清除 |

**永远成立的不变式**：

```text
0 <= mark <= position <= limit <= capacity
```

> **陷阱**：`Buffer` 的读模式与写模式是**同一块内存的两种视角**，靠 `flip()` 手动切换。忘记 `flip()` 是新手最常见的错误——`put` 完直接 `get`，此时 `position` 停在末尾，读出来的是 `limit - position == 0` 个字节，表现为"明明写了数据却读到空的"。

## 3.3 四个属性的状态变化全过程（本篇最核心）

下面用 `ByteBuffer.allocate(8)` 走完"写入 → 读取 → 清空"的完整生命周期。**看懂这条链，NIO 的 Buffer 就没有难点了**。

```text
步骤 1：allocate(8) 之后 —— 写模式：position=0, limit=capacity=8, mark=-1
                        position=0                    limit=8
                            │                           │
                            ▼                           ▼
        索引        0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │    │    │    │    │    │    │    │    │
                └────┴────┴────┴────┴────┴────┴────┴────┘
                 capacity = 8
```

```text
步骤 2：put('a') / put('b') / put('c') 之后 —— 每写一个字节 position 自动前移
                                  position=3    limit=8
                                      │            │
                                      ▼            ▼
        索引        0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ a  │ b  │ c  │    │    │    │    │    │
                └────┴────┴────┴────┴────┴────┴────┴────┘
```

```text
步骤 3：flip() 之后 —— 切换读模式：limit=position(=3)，position=0，mark 清除
                position=0    limit=3
                    │            │
                    ▼            ▼
        索引        0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ a  │ b  │ c  │    │    │    │    │    │
                └────┴────┴────┴────┴────┴────┴────┴────┘
                （只能读 0~2 这三个字节，后面 4~7 这轮不允许读）
```

```text
步骤 4：get() 两次之后 —— position 前移 2，数据仍在数组里没被删除
                             position=2  limit=3
                                 │         │
                                 ▼         ▼
        索引        0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ a  │ b  │ c  │    │    │    │    │    │
                └────┴────┴────┴────┴────┴────┴────┴────┘
                （a、b 已"读出"，但字节仍留在底层数组里，变的只是 position）

步骤 5：remaining() == limit - position == 1，即还剩 'c' 没读
```

```text
步骤 6：clear() 之后 —— 回到写模式：position=0, limit=capacity=8, mark 清除
        索引        0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ a  │ b  │ c  │    │    │    │    │    │
                └────┴────┴────┴────┴────┴────┴────┴────┘
          position=0                                  limit=8=capacity
                │
                └── clear() 不清数据！a/b/c 还在这块内存里，只是下一次 put 会从 0 开始覆盖
```

```text
步骤 7：若在第 4 步改用 compact() —— 保留未读数据 'c'，把它搬到数组头部
        索引        0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
                │ c  │    │    │    │    │    │    │    │
                └────┴────┴────┴────┴────┴────┴────┴────┘
          position=1（未读数据长度）              limit=8
                │
                └── compact() 之后可以直接继续 put，新数据接在 'c' 后面 —— 这正是"半包"场景需要的
```

> **最佳实践**：`clear()` 适用于"这批数据我全部读完了，下一轮从头写"；`compact()` 适用于"还有没读完的数据，但我得先继续写"。**网络编程里几乎总是用 `compact()`**，因为一次 `read()` 拿到的往往是不完整的报文，剩下半个包必须留着和后面的数据拼起来。

## 3.4 六大切换方法的对比与适用时机

| 方法 | 具体做了什么 | 什么时候调用 |
|------|-------------|-------------|
| `flip()` | `limit = position; position = 0; mark = -1` | **写完之后要读之前**，必调，写读模式的开关 |
| `clear()` | `position = 0; limit = capacity; mark = -1` | 数据全部读完，准备重写一整轮 |
| `compact()` | 把 `position ~ limit` 的未读数据复制到头部 → `position = 未读长度; limit = capacity; mark = -1` | 数据**没读完**就要继续写（半包 / 边读边写） |
| `rewind()` | `position = 0; mark = -1`，`limit` **不变** | 重读同一批数据（比如"把刚收到的消息原样回显"） |
| `mark()` | `mark = position` | 读到不确定的地方先做个书签 |
| `reset()` | `position = mark` | 解析失败时回退到书签处重新解析 |

`rewind()` 与 `flip()` 的区别只需记一句：**`flip()` 顺手把 `limit` 收到 `position`，而 `rewind()` 不动 `limit`**——所以 `rewind()` 之后你仍然只能读到原来那个 `limit`，而 `flip()` 之后能读的范围变成了刚写入的那一段。

## 3.5 ByteBuffer 常用方法

| 方法 | 说明 |
|------|------|
| `static ByteBuffer allocate(int capacity)` | 创建**堆内存** `Buffer`，`position=0`、`limit=capacity` |
| `static ByteBuffer allocateDirect(int capacity)` | 创建**直接内存** `Buffer`，见 3.6 |
| `static ByteBuffer wrap(byte[] array)` | 用一个现成数组包装成 `ByteBuffer` |
| `put(byte b)` / `put(byte[] src)` | 写入，`position` 自动前移 |
| `put(int index, byte b)` | **绝对写**：写入指定下标，`position` **不变** |
| `get()` / `get(byte[] dst)` | 读取，`position` 自动前移 |
| `get(int index)` | **绝对读**：读指定下标，`position` **不变** |
| `remaining()` / `hasRemaining()` | 剩余可读 / 可写元素数；是否还有剩余 |
| `asCharBuffer()` | 转成 `CharBuffer` 视图（处理文本时用） |
| `order(ByteOrder order)` | 设置字节序（大端 `BIG_ENDIAN` / 小端 `LITTLE_ENDIAN`） |
| `asReadOnlyBuffer()` | 得到只读视图，防止意外修改 |

> **注意**：`get(int index)` / `put(int index, byte)` 这组"绝对操作"**不会移动 `position`**，和 `get()` / `put()` 的区别与"索引访问 vs 迭代器访问"完全一样，混用时要格外小心。

## 3.6 直接内存 vs 堆内存

`allocate()` 与 `allocateDirect()` 的区别不只是"快慢"，而是**内存到底分配在哪里**：

| 维度 | `allocate()`（堆内存） | `allocateDirect()`（直接内存） |
|------|----------------------|------------------------------|
| 分配位置 | JVM 堆内 | 堆外的**本地内存**（操作系统内存） |
| 创建 / 销毁成本 | 低，由 JVM 管理 | **高**（要调用系统 API 分配） |
| 是否受 GC 管理 | 是，可正常回收 | `ByteBuffer` 对象本身受 GC 管，但**它背后那块本地内存要等 `Cleaner` 回收**，不可控 |
| IO 时是否要拷贝 | **要**：堆内数据需先复制到一块临时的直接内存再交给内核 | **不要**：内核可直接读写它 |
| 适用场景 | 短期使用、小容量、频繁创建 | 长期存活、大容量、高吞吐 IO（如 Netty 的池化 `ByteBuf`） |

```java
// 堆内存：适合临时用一下
ByteBuffer heapBuf = ByteBuffer.allocate(1024);

// 直接内存：适合长期复用的大缓冲区
ByteBuffer directBuf = ByteBuffer.allocateDirect(1024 * 1024);

// 判断一个 Buffer 是否为直接内存
System.out.println(directBuf.isDirect());   // true
```

> **陷阱**：直接内存**不受 `-Xmx` 约束**，它由 `-XX:MaxDirectMemorySize` 单独控制（默认约等于堆最大值）。如果大量创建 `allocateDirect()` 而不复用，很容易出现"堆还很空闲，程序却先 `OutOfMemoryError: Direct buffer memory` 崩了"的诡异现象——而且它**不参与常规 GC 统计**，用 `jmap` 也看不出来，只能靠 `-XX:MaxDirectMemorySize` 限制和 `BufferPoolMXBean` 监控。

## 3.7 完整的写读示例

```java
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;

public class BufferDemo {
    public static void main(String[] args) {
        ByteBuffer buf = ByteBuffer.allocate(16);
        print("初始", buf);                 // position=0, limit=16, capacity=16

        buf.put("hello".getBytes(StandardCharsets.UTF_8));   // 写入 5 个字节
        print("写入后", buf);               // position=5, limit=16

        buf.flip();                          // 写 → 读，必须调用
        print("flip 后", buf);              // position=0, limit=5

        byte[] data = new byte[buf.remaining()];   // remaining() = 5
        buf.get(data);
        System.out.println("读出: " + new String(data, StandardCharsets.UTF_8));  // 读出: hello
        print("读完后", buf);               // position=5, limit=5

        buf.rewind();                        // position 归零，limit 保持 5，可以再读一遍
        System.out.println("重读第一个字节: " + (char) buf.get());   // h

        buf.clear();                         // 回到写模式，position=0, limit=16
        print("clear 后", buf);
    }

    // 打印四个核心属性，方便直观看到每一步的变化
    private static void print(String stage, ByteBuffer buf) {
        System.out.printf("%-8s position=%d, limit=%d, capacity=%d, remaining=%d%n",
                stage, buf.position(), buf.limit(), buf.capacity(), buf.remaining());
    }
}
```

```text
初始      position=0, limit=16, capacity=16, remaining=16
写入后    position=5, limit=16, capacity=16, remaining=11
flip 后   position=0, limit=5,  capacity=16, remaining=5
读完后    position=5, limit=5,  capacity=16, remaining=0
重读第一个字节: h
clear 后  position=0, limit=16, capacity=16, remaining=16
```

# 4. Channel（通道）

## 4.1 Channel 与流的本质区别：它是双向的

`Channel` 表示一个**连接**，数据必须通过 `Buffer` 中转，方向只有两个：

```text
  读：Channel ──▶ Buffer      写：Buffer ──▶ Channel
     channel.read(buf)             channel.write(buf)
```

它和传统流最大的区别是 **`Channel` 是双向的**——一个 `Channel` 既能读又能写，而 `InputStream` / `OutputStream` 是两条独立的单向管道。这意味着**同一连接上不需要开两个对象**，这在网络编程里尤其重要（一个 TCP 连接本来就是一个双向的全双工通道）。

常用实现：

| Channel | 用途 |
|---------|------|
| `FileChannel` | 文件 IO，**不支持非阻塞**，只能配合 `FileInputStream`/`FileOutputStream`/`RandomAccessFile` 获取 |
| `SocketChannel` | TCP **客户端**通道，支持非阻塞，可注册到 `Selector` |
| `ServerSocketChannel` | TCP **服务端**监听通道，只监听 `OP_ACCEPT` |
| `DatagramChannel` | UDP 通道，支持非阻塞 |

## 4.2 FileChannel 常用方法

| 方法 | 说明 |
|------|------|
| `int read(ByteBuffer dst)` | 从通道读数据到缓冲区，返回实际读取字节数，返回 `-1` 表示到达文件末尾 |
| `int write(ByteBuffer src)` | 把缓冲区的数据写入通道 |
| `long position()` / `FileChannel position(long)` | 获取 / 设置文件指针位置 |
| `long size()` | 文件大小 |
| `FileChannel truncate(long size)` | 截断文件到指定长度 |
| `void force(boolean metaData)` | **强制刷盘**（类似 `flush`），`metaData=true` 连文件元数据一起刷 |
| `long transferTo(long position, long count, WritableByteChannel target)` | **零拷贝**：本通道数据直接传给目标通道 |
| `long transferFrom(ReadableByteChannel src, long position, long count)` | **零拷贝**：从源通道拉数据到本通道 |
| `FileLock lock()` / `FileLock tryLock()` | 排他文件锁（阻塞 / 非阻塞） |
| `MappedByteBuffer map(FileChannel.MapMode mode, long position, long size)` | **内存映射文件** |

> **注意**：`force(true)` 是 `FileChannel` **唯一**保证数据真正落到磁盘的手段，`write()` 返回只代表写进了操作系统页缓存。写关键数据（比如数据库的 WAL、消息队列的刷盘）必须显式 `force`。

## 4.3 文件复制：三种写法

```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class FileCopyDemo {
    public static void main(String[] args) throws Exception {
        try (FileChannel src = new FileInputStream("source.txt").getChannel();
             FileChannel dest = new FileOutputStream("dest.txt").getChannel()) {

            // 写法一：通用循环（Buffer 需要自己 flip / clear 轮换）
            ByteBuffer buf = ByteBuffer.allocate(1024);
            while (src.read(buf) != -1) {   // 读满或读不到就返回 -1
                buf.flip();                 // 写完 → 转读模式
                dest.write(buf);            // 把 buf 中 0~limit 的数据写出
                buf.clear();                // 读完 → 回写模式，准备下一轮
            }

            // 写法二：零拷贝（推荐，大文件下差距极其明显）
            src.transferTo(0, src.size(), dest);
        }
    }
}
```

> **陷阱**：写法一里 `read(buf)` 返回后 `buf` 处于**写模式**，直接 `dest.write(buf)` 会写出 0 个字节（`position` 已在末尾、`limit` 在 capacity）；必须先 `flip()`。这个"忘了 flip"的错误在 NIO 代码里出现频率最高。

## 4.4 零拷贝：transferTo / transferFrom 与内存映射

**传统文件读写的四次拷贝**（以"读文件再写出去"为例）：

```text
  磁盘 ─▶ 内核页缓存 ─▶ 用户缓冲区 ─▶ 内核 socket 缓冲区 ─▶ 网卡
         （DMA 拷贝）   （CPU 拷贝）    （CPU 拷贝）        （DMA 拷贝）
                       position 1     position 2
```

`transferTo` / `transferFrom` 底层走的是操作系统的 **`sendfile` 系统调用**，数据**完全不经过用户态**：

```text
  磁盘 ─▶ 内核页缓存 ─────────────────▶ 内核 socket 缓冲区 ─▶ 网卡
         （DMA 拷贝）                    （只传描述符，近乎零拷贝）
```

```java
FileChannel src  = new FileInputStream("source.txt").getChannel();
FileChannel dest = new FileOutputStream("dest.txt").getChannel();

// transferTo：从本通道的 [0, size) 区间传输到目标通道，返回实际传输字节数
long transferred = src.transferTo(0, src.size(), dest);
System.out.println("传输字节数: " + transferred);

// transferFrom：反向，从源通道拉数据到本通道
// dest.transferFrom(src, 0, src.size());
```

**内存映射（`MappedByteBuffer`）** 是另一种零拷贝思路：把文件的一段直接映射成进程的虚拟内存，读写这块内存就等于读写文件，由操作系统负责按页换入换出。

```java
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

public class MappedDemo {
    public static void main(String[] args) throws Exception {
        try (RandomAccessFile raf = new RandomAccessFile("big.dat", "rw");
             FileChannel channel = raf.getChannel()) {

            // 把整个文件映射到内存
            MappedByteBuffer mapped = channel.map(
                    FileChannel.MapMode.READ_WRITE, 0, channel.size());

            mapped.put(0, (byte) 'X');        // 绝对写：直接改文件首字节
            byte first = mapped.get(0);       // 绝对读
            System.out.println("首字节: " + (char) first);   // 首字节: X

            mapped.force();                   // 手动刷盘（等价于 msync）
        }
    }
}
```

| 方式 | 底层机制 | 优点 | 适用场景 |
|------|---------|------|---------|
| 普通 Buffer 循环 | `read` + `write` | 通用，逻辑可控 | 小文件、需要在传输过程中加工数据 |
| `transferTo` / `transferFrom` | `sendfile` | 零拷贝，代码极简 | **文件 → 文件、文件 → 网络的整块搬运** |
| `MappedByteBuffer` | `mmap` | 随机访问大文件像访问数组 | 大文件索引、数据库式随机读写 |

> **陷阱**：`MappedByteBuffer` 映射的内存**只能靠 GC 或 `Unmapper` 释放**，在 Windows 上文件被映射期间**无法删除**（Linux 上可以 `unlink`）；且映射大小受虚拟地址空间限制（32 位 JVM 上就是个大坑）。映射一个 10GB 文件不会立刻占 10GB 物理内存，但一旦大量随机访问触发缺页，物理内存会被迅速吃满。

## 4.5 SocketChannel / ServerSocketChannel

| 类 | 关键方法 | 说明 |
|----|---------|------|
| `ServerSocketChannel` | `static open()` | 打开监听通道 |
| | `bind(SocketAddress)` | 绑定端口（等价于 `ServerSocket.bind`） |
| | `configureBlocking(boolean)` | 设置阻塞 / 非阻塞，**注册到 `Selector` 前必须设为 `false`** |
| | `SocketChannel accept()` | 接收连接；非阻塞模式下**没有新连接会返回 `null`** |
| `SocketChannel` | `static open()` / `open(SocketAddress)` | 打开 / 打开并连接 |
| | `boolean connect(SocketAddress)` | 连接；非阻塞模式下可能立即返回 `false`，表示连接进行中 |
| | `boolean finishConnect()` | 配合 `OP_CONNECT` 事件完成连接 |
| | `int read(ByteBuffer)` / `int write(ByteBuffer)` | 读写，非阻塞下可能返回 `0` |
| | `shutdownInput()` / `shutdownOutput()` | 半关闭（只关一个方向） |

> **注意**：`accept()` 在非阻塞模式下返回 `null` 是**正常情况**（事件来了但被别的线程抢先 `accept` 掉了），必须判空，否则后面一句 `clientChannel.configureBlocking(false)` 直接 NPE。这是 NIO 服务端最常见的一类崩溃。

# 5. Selector（选择器 / 多路复用器）

## 5.1 多路复用的核心思想

`Selector` 是 NIO 的灵魂。它要解决的问题是：**如何用一个线程照看成千上万个连接？**

```text
   一连接一线程（BIO）                           Selector 轮询（NIO 多路复用）

   线程1 ─▶ 连接1   （阻塞 read，大部分时间空闲）      ┌──────────┐
   线程2 ─▶ 连接2   （阻塞 read，大部分时间空闲）      │ Selector │── 注册 连接1~连接N
   线程3 ─▶ 连接3   （阻塞 read，大部分时间空闲）      └────┬─────┘
    ……                                                  │ select() 阻塞
   线程N ─▶ 连接N                                      ▼
                                                返回"就绪"的连接列表
   1000 个连接 = 1000 个线程                          │
   每个线程各自持有一份栈内存，                       ├─▶ 连接 3 可读 → 读它
   CPU 大量时间花在线程切换上                         ├─▶ 连接 7 可读 → 读它
                                                     └─▶ 连接 9 可连 → 收它
                                              1000 个连接 = 1 个线程轮询
                                              空闲的连接不消耗任何线程
```

它的底层对应 Linux 的 **`epoll`**（JDK 在 Linux 上优先使用 `epoll`，其他平台退化到 `poll`/`select`）。`select()` 并不是"一个一个挨着问"，而是**把待监听的事件集合交给内核，由内核在事件发生时把它标记为就绪**，所以即使有十万个连接，线程也只被唤醒在"真正有事发生"的时候。

## 5.2 Selector 基本 API

| 方法 | 说明 |
|------|------|
| `static Selector open()` | 创建 `Selector` |
| `int select()` | **阻塞**，直到至少有一个通道就绪，返回就绪通道数 |
| `int select(long timeout)` | 阻塞最多 `timeout` 毫秒 |
| `int selectNow()` | **非阻塞**，立即返回（可能为 0） |
| `Set<SelectionKey> selectedKeys()` | 获取已就绪的事件集合（**必须手动移除已处理的 key**） |
| `Set<SelectionKey> keys()` | 获取全部已注册的 key（含未就绪的） |
| `Selector wakeup()` | 唤醒阻塞在 `select()` 上的线程 |
| `void close()` | 关闭 `Selector`，其上的所有 key 一并失效 |

## 5.3 SelectionKey（选择键）

`Channel` 注册到 `Selector` 时返回 `SelectionKey`，它表示"**某个 Channel 在某个 Selector 上的注册关系**"，同时也是挂载业务数据的地方。

**四种事件常量**（值是位掩码，可用 `|` 组合）：

| 常量 | 值 | 含义 | 用在谁身上 |
|------|----|------|-----------|
| `SelectionKey.OP_READ` | 1 | 有数据可读 | 所有通道 |
| `SelectionKey.OP_WRITE` | 4 | 可写 | 所有通道 |
| `SelectionKey.OP_CONNECT` | 8 | 连接就绪（非阻塞 `connect` 完成） | 客户端 `SocketChannel` |
| `SelectionKey.OP_ACCEPT` | 16 | 有新连接可以接收 | 服务端 `ServerSocketChannel` |

**常用方法**：

| 方法 | 说明 |
|------|------|
| `int interestOps()` / `SelectionKey interestOps(int)` | **感兴趣的事件集合**：我关心什么，注册时指定 |
| `int readyOps()` | **已经就绪的事件集合**：内核告诉我现在什么可做，由 `Selector` 更新 |
| `boolean isReadable()` / `isWritable()` / `isConnectable()` / `isAcceptable()` | `readyOps` 的便捷判断 |
| `SelectableChannel channel()` | 取出关联的 `Channel` |
| `Selector selector()` | 取出关联的 `Selector` |
| `Object attach(Object)` / `Object attachment()` | 挂载 / 取出**附件对象**（可存放 `ByteBuffer`、会话状态、心跳时间等） |
| `boolean isValid()` | 该 key 是否仍然有效 |
| `void cancel()` | 取消注册（实际在下一次 `select()` 时才真正移除） |

> **最佳实践**：`interestOps` 与 `readyOps` 的区别是**理解 `SelectionKey` 的关键**。`interestOps` 是"我要什么"（自己设置，注册后一般不动），`readyOps` 是"现在有什么"（每次 `select()` 由 `Selector` 重写）。另外，**别为每个连接新建一个 `ByteBuffer`**，把它通过 `attach()` 挂到 key 上复用，能显著减少 GC 压力——这正是 3.6 里"直接内存 + 长期存活"的典型用途。

## 5.4 Selector 完整示例（TCP 非阻塞服务器）

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.Set;

public class NioServer {
    public static void main(String[] args) throws Exception {
        // 1. 打开服务端监听通道并绑定端口
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(9999));
        serverChannel.configureBlocking(false);   // 必须在 register 之前调用

        // 2. 打开 Selector
        Selector selector = Selector.open();

        // 3. 注册 ACCEPT 事件：我只关心"有没有新连接进来"
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO 服务器已启动，监听 9999 ...");

        // 4. 事件循环
        while (true) {
            // 阻塞等待事件；返回 0 说明只是超时或被 wakeup 唤醒，直接下一轮
            if (selector.select() == 0) continue;

            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> it = keys.iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                // 必须在这里移除：selectedKeys() 返回的是内部集合，
                //    Selector 只管往里加，不会替你清；不移除 → 下一轮重复处理 → 空轮询
                it.remove();

                if (!key.isValid()) continue;

                if (key.isAcceptable()) {
                    // 有新连接进来
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    SocketChannel client = ssc.accept();   // 非阻塞下可能为 null！
                    if (client == null) continue;

                    client.configureBlocking(false);       // 同样必须在注册前调用
                    // 给这个连接挂一个专属缓冲区，随 key 一起存活，避免每次读都新建
                    client.register(selector, SelectionKey.OP_READ,
                            ByteBuffer.allocateDirect(1024));
                    System.out.println("新客户端连接: " + client.getRemoteAddress());

                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    ByteBuffer buf = (ByteBuffer) key.attachment();   // 取出复用的缓冲区

                    int len;
                    try {
                        len = client.read(buf);
                    } catch (Exception e) {
                        // 客户端非正常断开时会抛 IOException / Connection reset
                        closeClient(key, client);
                        continue;
                    }

                    if (len == -1) {
                        // -1 表示对端已关闭连接（正常挥手）
                        closeClient(key, client);
                    } else if (len > 0) {
                        buf.flip();                       // 写 → 读
                        String msg = StandardCharsets.UTF_8.decode(buf).toString();
                        System.out.println("收到: " + msg);

                        // 原样回显：把刚读到的数据重新写回客户端
                        buf.rewind();                     // position 归零，limit 不变 → 重读一遍
                        client.write(buf);
                        buf.clear();                      // 回写模式，为下一次 read 做准备
                    }
                    // len == 0 属正常（非阻塞下暂无数据），什么都不做
                }
            }
        }
    }

    private static void closeClient(SelectionKey key, SocketChannel client) throws Exception {
        System.out.println("客户端断开: " + client.getRemoteAddress());
        key.cancel();          // 从 Selector 注销
        client.close();        // 关闭通道，避免 fd 泄漏
    }
}
```

## 5.5 三个必须记住的关键点

**① `iterator.remove()` 不是可选项，是必选项**

`selector.selectedKeys()` 返回的是 `Selector` **内部集合的视图**，它只负责"往里加"，从不负责"往外清"。如果处理完不 `remove()`：

```text
  第 1 轮：key A 就绪 → 处理 A → 不移除
  第 2 轮：集合里仍有 A，即使 A 这次没就绪 → 再处理一次 A（此时 read 返回 0，白跑）
  第 3 轮、第 4 轮 …… 永远重复
  → CPU 100%，select() 每次都立刻返回非 0，形成"空轮询"
```

> **陷阱**：这个 bug 的可怕之处在于**程序不会报错、不会崩，只是 CPU 悄悄打满**，而且是 NIO 编程最经典的坑。Netty 专门为此写了防御代码（记录 `select` 次数，超过阈值就重建 `Selector`）。此外注意：**遍历时必须用 `Iterator` 而不能用 for-each**，因为 for-each 里删除元素会抛 `ConcurrentModificationException`。

**② `configureBlocking(false)` 必须在 `register()` 之前调用**

`Selector` 只能注册非阻塞通道，顺序反了会直接抛 `IllegalBlockingModeException`。同样地，**已经注册的通道不能改回阻塞模式**，会抛 `IllegalBlockingModeException`。

**③ `select()` 返回 0 不是错误**

`select()` 可能因为**超时**（`select(timeout)`）或**被 `wakeup()` 唤醒**而返回 0；而 `selectedKeys()` 在 `select()` 返回 0 时也可能非空（比如 `wakeup()` 与 `register` 竞争）。所以更稳妥的写法是：

```java
int n = selector.select();          // 阻塞等待
if (n > 0) {
    // 只有确认有就绪事件才遍历，避免无意义地走空循环
}
```

## 5.6 Selector 工作流程

```text
   单线程应用              Selector                   Channel / 内核
       │                      │                            │
       │─ Selector.open() ───▶│                            │
       │─ Channel.open() ─────│───────────────────────────▶│
       │─ configureBlocking(false) ───────────────────────▶│
       │─ register(sel, OP_ACCEPT) ───────────────────────▶│
       │                      │──── 返回 SelectionKey ────│
       │                      │                            │
       │─ select() 阻塞等待 ─▶│                            │
       │                      │── epoll_wait 交给内核 ────▶│
       │                      │                            │（事件发生）
       │                      │─── 内核标记就绪的 fd ──────│
       │── 返回就绪 key 集合 ─│                            │
       ▼                      │                            │
   遍历 selectedKeys()        │                            │
       │                      │                            │
       ├─ isAcceptable() ─────│── accept() ───────────────▶│ 新连接
       │                      │   register(sel, OP_READ) ─▶│
       │                      │                            │
       ├─ isReadable() ───────│── read(buf) ──────────────▶│ 数据到达
       │   buf.flip() → 处理业务 → buf.clear()             │
       │                      │                            │
       ├─ isWritable() ───────│── write(buf) ─────────────▶│ 发送缓冲可写
       │                      │                            │
       └─ it.remove()  移除本轮已处理的 key                 │
       │                      │                            │
       └─── 回到 select() 继续下一轮 ───────────────────────┘
```

# 6. NIO 与 BIO 网络编程对比示例

```java
// BIO 多线程服务器：一个连接一个线程，1000 个连接 = 1000 个线程
ServerSocket server = new ServerSocket(9999);
while (true) {
    Socket socket = server.accept();      // 阻塞：没有新连接就一直等
    new Thread(() -> {
        // 这个线程被这条连接独占；read/write 期间同样阻塞
        handleClient(socket);
    }).start();
}
```

```java
// NIO 单线程服务器：一个线程管理所有连接，只在"有事件"时才干活
Selector selector = Selector.open();
serverChannel.register(selector, SelectionKey.OP_ACCEPT);
while (true) {
    selector.select();                    // 阻塞一次，等待任意通道就绪
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();
        processKey(key);                  // 一个线程按顺序处理所有就绪事件
    }
}
```

| 指标 | BIO | NIO |
|------|-----|-----|
| 1000 个连接需要 | 1000 个线程 | 1 个线程（+ 业务线程池） |
| 内存开销 | 每线程约 1MB 栈，合计约 1GB | 只有缓冲区，几 MB 量级 |
| CPU 开销 | 大量时间在线程调度 / 上下文切换 | 只在有事件时工作，空闲几乎零成本 |
| 编程难度 | **低**，顺序化代码，容易理解和调试 | **高**，事件驱动、状态机、半包粘包都要自己处理 |
| 连接数上限 | 千级 | 万级乃至十万级 |

> **行业实践**：选型上有个简单判据——**连接数少（几百以内）且每个连接数据量大，用 BIO 反而更简单更稳**（比如内部管理后台、定时批处理）；**连接数多但每连接数据量小，才用 NIO**（IM、推送、API 网关）。而真正的高并发服务端不会自己去写裸 NIO，而是用 Netty，把事件驱动、编解码、连接管理的复杂度都交给成熟框架。

# 7. Path 和 Files（java.nio.file，JDK 7+）

NIO 2.0 引入的 `java.nio.file` 包提供了更现代的文件操作 API，日常开发中用得比 `FileChannel` 频繁得多：

| 类 / 接口 | 作用 |
|-----------|------|
| `Path` | 文件路径的抽象，**替代 `java.io.File`** |
| `Paths` | 获取 `Path` 的工厂类（JDK 11 起也可直接用 `Path.of(...)`） |
| `Files` | 静态工具类：读写、复制、移动、遍历、属性查询 |
| `FileVisitor` / `Files.walkFileTree` | 递归遍历目录树 |
| `WatchService` | 监听目录变化（文件新增 / 修改 / 删除） |

最常用的几个方法：

```java
import java.nio.file.*;

Path path = Path.of("D:/data", "hello.txt");     // JDK 11+ 推荐写法

Files.writeString(path, "你好，NIO");             // JDK 11+，一行写入
String content = Files.readString(path);          // JDK 11+，一行读出
System.out.println(content);                       // 你好，NIO

Files.copy(path, Path.of("D:/data/copy.txt"), StandardCopyOption.REPLACE_EXISTING);
Files.move(path, Path.of("D:/data/moved.txt"), StandardCopyOption.ATOMIC_MOVE);
System.out.println(Files.exists(path));            // false（已经 move 走了）

// 遍历目录下所有 .txt 文件
try (var stream = Files.walk(Path.of("D:/data"), 1)) {   // 深度 1，只看本层
    stream.filter(p -> p.toString().endsWith(".txt"))
          .forEach(System.out::println);
}
```

> **注意**：`Path` / `Files` 的**完整用法（路径操作、属性读取、目录遍历、`WatchService`）集中在第 15 篇**，那里有专门一节展开；本节只保留与 NIO 通道体系的衔接，避免与第 15 篇重复。

# 8. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 三种 IO 模型 | BIO 同步阻塞、一连接一线程；NIO 同步非阻塞 + `Selector` 多路复用；AIO 由内核完成 IO 后回调，Linux 支持不完善 |
| 选型 | 连接少数据量大用 BIO；**高并发小包用 NIO**；生产环境用 Netty 而非裸 NIO |
| Buffer 四属性 | `capacity` 不变，`limit` 是读写边界，`position` 是下一个读写位，`mark` 是书签；恒有 `mark <= position <= limit <= capacity` |
| 状态变化 | 写完 `flip()` 转读；读一半要写用 `compact()`；全部读完重写用 `clear()`；重读一遍用 `rewind()` |
| 直接内存 | `allocateDirect` 免去一次堆到本地的拷贝、IO 更快，但**不受 `-Xmx` 管、创建销毁贵**，适合长期复用的大缓冲区 |
| Channel | **双向**，必须经 `Buffer` 中转；`FileChannel` 不支持非阻塞；`transferTo` 走 `sendfile` 实现零拷贝，`map` 走 `mmap` 实现内存映射 |
| Selector | 一个线程轮询多个连接；`interestOps` 是"我要什么"，`readyOps` 是"现在有什么"；`attachment` 用来挂每个连接的会话数据 |
| 三个必踩的坑 | 遍历后**必须 `iterator.remove()`**（否则空轮询打满 CPU）；`configureBlocking(false)` 必须在 `register()` **之前**；`accept()` 非阻塞下可能返回 `null` |

下一篇（第 28 篇）讲 `Optional`，看 Java 8 如何用容器类型把"可能为空"这件事从运行时异常变成编译期就能看见的类型信息；文件与目录的完整 API 见第 15 篇，`Record` 与密封类等语法新特性见第 26、29 篇。
