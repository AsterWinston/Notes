# 1. NIO 概述

```
NIO（New I/O）于 JDK 1.4 引入，是一种基于 Channel 和 Buffer 的 I/O 模型。
NIO 支持非阻塞 I/O，可以处理大量并发连接（一个线程管理多个连接）。

NIO 三大核心组件：
  Channel（通道）：类似流，但可以同时读写，是双向的
  Buffer（缓冲区）：数据的容器，所有数据都通过 Buffer 处理
  Selector（选择器）：用于监听多个 Channel 的事件，实现单线程管理多连接
```

# 2. BIO 与 NIO 对比

```
                     BIO (Blocking I/O)              NIO (Non-blocking I/O)
  数据流向             单向（输入流/输出流）            双向（Channel 可读可写）
  数据处理单位         字节/字符                        块（Buffer）
  阻塞特性             阻塞（读写时线程等待）            非阻塞（线程可做其他事）
  并发支持             一个连接一个线程                  一个线程管理多个连接（Selector）
  适用场景             连接数少、数据量大的场景           高并发、连接数多但数据量小的场景
  API                  基于流（Stream）                 基于通道和缓冲区（Channel/Buffer）

AIO（Asynchronous I/O，NIO 2.0，JDK7+）：
  真正的异步 I/O，读写操作由操作系统完成，完成后通过回调通知。
  AIO 的 API 包括 AsynchronousFileChannel、AsynchronousSocketChannel 等。
```

# 3. Buffer（缓冲区）

```
Buffer 是一个内存块（底层是数组），用于临时存储读写的数据。
所有数据的读写都通过 Buffer 完成。

Buffer 是抽象类，常用子类：
  ByteBuffer（最常用）、CharBuffer、ShortBuffer、IntBuffer、LongBuffer、FloatBuffer、DoubleBuffer
```

## 3.1Buffer 核心属性

```
capacity（容量）：Buffer 的总大小，创建时确定，不可改变
limit（限制）：可读/可写的最大位置。写模式 limit == capacity，读模式 limit == 之前写入的数据量
position（位置）：下一个要被读取/写入的元素的索引
mark（标记）：用于标记一个位置，配合 reset() 方法回到标记位置

四个属性的大小关系：mark <= position <= limit <= capacity
```

## 3.2Buffer 核心方法

```
3.2.1 基本操作
  capacity()           // 返回容量
  position()           // 返回当前位置
  position(int newPos) // 设置位置
  limit()              // 返回限制
  limit(int newLimit)  // 设置限制
  mark()               // 标记当前位置（将 mark 设为 position）
  reset()              // 回到 mark 标记的位置
  clear()              // 切换到写模式：position=0, limit=capacity, 清除 mark（数据不清除）
  flip()               // 切换到读模式：limit=position, position=0, 清除 mark
  rewind()             // 重新读：position=0, 清除 mark
  remaining()          // 剩余可读/可写元素数 = limit - position
  hasRemaining()       // 是否还有剩余元素

3.2.2 ByteBuffer 常用方法
  static ByteBuffer allocate(int capacity)     // 创建堆内存 Buffer
  static ByteBuffer allocateDirect(int capacity) // 创建直接内存 Buffer（更快，但创建开销大）
  put(byte b)             // 写入单个字节
  put(byte[] src)         // 写入字节数组
  put(int index, byte b)  // 在指定位置写入（position 不变）
  get()                   // 读取单个字节（position 后移）
  get(byte[] dst)         // 读取到字节数组
  get(int index)          // 读取指定位置（position 不变）
  flip()                  // 从写模式切换到读模式
  clear()                 // 从读模式切换到写模式
  compact()               // 将未读数据移到头部，position移到未读数据之后（保留未读数据）
  wrap(byte[] array)      // 将字节数组包装为 ByteBuffer
  asCharBuffer()          // 转换为 CharBuffer 视图（处理文本）
  order(ByteOrder order)  // 设置字节序（大端/小端）

3.2.3 ByteBuffer 使用模式

  写入 → 读取：
    ByteBuffer buf = ByteBuffer.allocate(1024);
    // 写
    buf.put("hello".getBytes());
    // 切换到读模式
    buf.flip();
    // 读
    byte[] data = new byte[buf.remaining()];
    buf.get(data);
    System.out.println(new String(data));  // hello

  clear vs compact 的区别：
    - clear()：position 回到 0，可从头写入（旧数据被覆盖）
    - compact()：将未读数据移到开头，position 移到未读数据末尾（保留旧数据继续写）

3.2.4 直接内存 vs 堆内存 Buffer
  直接内存（allocateDirect）：
    优点：避免 JVM 堆和本地内存之间的拷贝，I/O 效率高
    缺点：创建和销毁成本高，内存不受 GC 直接管理
    适用：长期使用、大容量、高 I/O 场景

  堆内存（allocate）：
    优点：创建成本低，GC 可回收
    缺点：I/O 时需复制到直接内存
    适用：短期使用、小容量场景
```

# 4. Channel（通道）

```
Channel 是双向的数据传输通道，可以同时进行读写操作。
Channel 配合 Buffer 使用：读时 Channel → Buffer，写时 Buffer → Channel。

常用 Channel：
  FileChannel：文件 I/O 通道
  SocketChannel：TCP 客户端通道
  ServerSocketChannel：TCP 服务端通道（监听连接）
  DatagramChannel：UDP 通道
```

## 4.1FileChannel（文件通道）

```
FileChannel 只能通过 FileInputStream/FileOutputStream/RandomAccessFile 获取。

4.1.1 常用方法
  int read(ByteBuffer dst)          // 从通道读数据到缓冲区，返回读取字节数（-1 表示结束）
  int write(ByteBuffer src)         // 从缓冲区写数据到通道
  long position()                   // 获取文件指针位置
  FileChannel position(long pos)    // 设置文件指针位置
  long size()                       // 获取文件大小
  FileChannel truncate(long size)   // 截断文件
  void force(boolean metaData)     // 强制刷盘（类似于 flush）
  long transferTo(position, count, targetChannel)  // 零拷贝：本通道数据传到目标通道
  long transferFrom(srcChannel, position, count)   // 零拷贝：从源通道传入本通道
  FileLock lock()                   // 文件锁（排他锁）
  FileLock tryLock()                // 尝试获取文件锁（非阻塞）
  MappedByteBuffer map(mode, position, size)  // 内存映射文件

4.1.2 文件复制示例（NIO）
  FileInputStream fis = new FileInputStream("source.txt");
  FileOutputStream fos = new FileOutputStream("dest.txt");
  FileChannel srcChannel = fis.getChannel();
  FileChannel destChannel = fos.getChannel();

  ByteBuffer buf = ByteBuffer.allocate(1024);
  while (srcChannel.read(buf) != -1) {
      buf.flip();       // 切换为读模式
      destChannel.write(buf);
      buf.clear();      // 切换为写模式
  }
  srcChannel.close();
  destChannel.close();

4.1.3 零拷贝文件复制（transferTo/transferFrom）
  FileChannel srcChannel = new FileInputStream("source.txt").getChannel();
  FileChannel destChannel = new FileOutputStream("dest.txt").getChannel();

  // transferTo：从本通道传输到目标通道
  srcChannel.transferTo(0, srcChannel.size(), destChannel);
  // 或 transferFrom：
  // destChannel.transferFrom(srcChannel, 0, srcChannel.size());

  // 效率极高，底层利用操作系统的 sendfile 系统调用，减少数据拷贝
  srcChannel.close();
  destChannel.close();
```

## 4.2SocketChannel / ServerSocketChannel（TCP NIO）

```
ServerSocketChannel：监听 TCP 连接的通道。
SocketChannel：实际的 TCP 通信通道。

4.2.1 ServerSocketChannel
  static ServerSocketChannel open()     // 打开
  ServerSocketChannel bind(SocketAddress)  // 绑定端口
  ServerSocketChannel configureBlocking(boolean)  // 设置阻塞/非阻塞模式
  SocketChannel accept()                // 接收连接（阻塞模式等待，非阻塞模式立即返回）

4.2.2 SocketChannel
  static SocketChannel open()           // 打开
  static SocketChannel open(SocketAddress)  // 打开并连接
  boolean connect(SocketAddress)        // 连接（非阻塞模式可能未完成）
  boolean finishConnect()               // 完成连接
  int read(ByteBuffer dst)              // 读数据
  int write(ByteBuffer src)             // 写数据
  SocketChannel configureBlocking(boolean)  // 设置阻塞/非阻塞模式

4.2.3 TCP NIO 通信示例
  见第 5 节 Selector 示例。
```

# 5. Selector（选择器 / 多路复用器）

```
Selector 是 NIO 非阻塞编程的核心。一个 Selector 可以监听多个 Channel 的事件，
当 Channel 有事件（可读、可写、可连接等）时，Selector 会通知应用程序。

工作原理（类似于 Linux 的 epoll/select 机制）：
  1. 创建 Selector
  2. 将 Channel 注册到 Selector，并指定要监听的事件
  3. 调用 select() 方法阻塞等待事件
  4. 遍历已就绪的事件进行处理
```

## 5.1Selector 基本 API

```
static Selector open()                  // 创建 Selector
int select()                            // 阻塞，直到有通道就绪（返回就绪通道数）
int select(long timeout)                // 阻塞指定时间（毫秒）
int selectNow()                         // 非阻塞，立即返回
Set<SelectionKey> selectedKeys()        // 获取已就绪的事件集合
Selector wakeup()                       // 唤醒 select() 阻塞
void close()                            // 关闭 Selector
```

## 5.2SelectionKey（选择键）

```
Channel 注册到 Selector 后返回 SelectionKey，表示 Channel 和 Selector 的关联。

5.2.1 注册事件常量
  SelectionKey.OP_READ    (1)  // 可读
  SelectionKey.OP_WRITE   (4)  // 可写
  SelectionKey.OP_CONNECT (8)  // 连接就绪（客户端）
  SelectionKey.OP_ACCEPT  (16) // 接收连接就绪（服务端）

5.2.2 常用方法
  boolean isReadable()      // 是否可读
  boolean isWritable()      // 是否可写
  boolean isConnectable()   // 是否可连接
  boolean isAcceptable()    // 是否有新连接
  SelectableChannel channel() // 获取关联的 Channel
  Selector selector()       // 获取关联的 Selector
  Object attach(Object)     // 附加对象
  Object attachment()       // 获取附加对象
  void cancel()             // 取消注册
```

## 5.3Selector 完整示例（TCP 非阻塞服务器）

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void main(String[] args) throws Exception {
        // 1. 创建 ServerSocketChannel，绑定端口，设置为非阻塞
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(9999));
        serverChannel.configureBlocking(false);

        // 2. 创建 Selector
        Selector selector = Selector.open();

        // 3. 将 ServerSocketChannel 注册到 Selector，监听 ACCEPT 事件
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO 服务器已启动，监听端口 9999...");

        // 4. 事件循环
        while (true) {
            // 阻塞等待事件
            if (selector.select() == 0) continue;

            // 获取就绪事件集合
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> it = keys.iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();  // 处理完移除（必须操作）

                if (key.isAcceptable()) {
                    // 有新连接
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    SocketChannel clientChannel = ssc.accept();
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                    System.out.println("新客户端连接: " + clientChannel.getRemoteAddress());
                }
                else if (key.isReadable()) {
                    // 有数据可读
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(1024);
                    int len = clientChannel.read(buf);

                    if (len == -1) {
                        // 客户端断开
                        System.out.println("客户端断开: " + clientChannel.getRemoteAddress());
                        clientChannel.close();
                    } else {
                        buf.flip();
                        String msg = new String(buf.array(), 0, buf.remaining());
                        System.out.println("收到: " + msg);
                        // 回复
                        buf.rewind();
                        clientChannel.write(buf);
                    }
                }
            }
        }
    }
}
```

## 5.4Selector 工作流程图

```
  单线程                     Selector                    Channel
    |                          |                           |
    |-- 创建 Selector -------->|                           |
    |                          |                           |
    |-- 创建 Channel ----------|-------------------------->|
    |-- 设为非阻塞             |                           |
    |-- 注册到 Selector ------>|<----- OP_READ/OP_WRITE ---|
    |                          |                           |
    |                          |                           |
    |-- select() 阻塞等待 ---->|                           |
    |                          |-- 检测事件就绪             |
    |<-- 返回就绪 keys ------  |                           |
    |                          |                           |
    |-- 遍历 keys 处理事件------------------------------->|
    |                          |                           |
```

# 6. NIO 与 BIO 网络编程对比示例

```
BIO 多线程服务器（每个连接一个线程）：
  ServerSocket server = new ServerSocket(9999);
  while (true) {
      Socket socket = server.accept();  // 阻塞
      new Thread(() -> {
          // 每个连接独占一个线程
          handleClient(socket);  // read/write 都阻塞
      }).start();
  }

NIO 单线程服务器（一个线程管理所有连接）：
  Selector selector = Selector.open();
  serverChannel.register(selector, SelectionKey.OP_ACCEPT);
  while (true) {
      selector.select();  // 阻塞等待事件
      for (SelectionKey key : selector.selectedKeys()) {
          // 一个线程处理所有连接的事件
          processKey(key);
      }
  }

优势对比：
  BIO：1000 个连接 = 1000 个线程（内存和 CPU 调度开销大）
  NIO：1000 个连接 = 1 个线程（只需处理有事件的连接）
```

# 7. Path 和 Files（java.nio.file，JDK7+）

```
NIO 2.0 引入了 java.nio.file 包，提供了更现代的文件操作 API。

核心类：
  Path：文件路径接口，替代 java.io.File
  Paths：获取 Path 对象的工厂类
  Files：文件操作的工具类

详细用法见 15_file.md 第 2 节。
```
