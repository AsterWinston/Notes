"服务端扛不住"很少是单一原因。可能是 Nagle 让延迟莫名多了 40ms，可能是线程池被慢连接占满，也可能是接收缓冲区太小导致窗口收缩，还可能是没人消费的写队列把内存撑爆。本篇按"内核参数 → 线程模型 → 缓冲与背压 → 零拷贝 → 压测监控"的顺序，把 TCP 服务端的性能问题拆成可逐项检查的清单。

# 1. TCP 层参数

## 1.1 Nagle 算法与 TCP_NODELAY

Nagle 算法会缓存小数据，等到凑够一个 MSS 或收到前一个包的 ACK 再发送，目的是减少网络上的小包数量。代价是**延迟**：发一条 20 字节的请求，可能要等到 40ms 后收到 ACK 才真正发出，后续数据才能跟上，这就是经典的"Nagle + 延迟 ACK"组合问题。

```text
应用 write(20B) ──> 内核缓存（未达 MSS，等 ACK）
                      │
                      │ 40ms 后对端延迟 ACK 到达
                      ▼
                   真正发出
```

对延迟敏感的场景（RPC、游戏、交互式请求）应关闭：

```java
Socket socket = new Socket();
socket.setTcpNoDelay(true);   // 关闭 Nagle，每个 write 尽快发出

// Netty 中
bootstrap.childOption(ChannelOption.TCP_NODELAY, true);
```

| 场景 | 建议 |
|------|------|
| RPC、游戏、即时通讯 | `TCP_NODELAY = true` |
| 大文件传输、批量上报 | 保持默认，利用合并提高带宽利用率 |
| 小请求高频交互 | 关闭 Nagle，并考虑应用层批量合并 |

**设计考量**：关闭 Nagle 会增加小包数量，在带宽紧张或按包计费的链路上有成本。更好的做法是应用层攒批：把多个小消息合并成一次 `write`，既减少系统调用，又不牺牲延迟。

## 1.2 keepalive

`SO_KEEPALIVE` 是链路层兜底，默认 2 小时才开始探测，见第 5 篇。开启它不会带来明显开销，可以保留：

```java
socket.setKeepAlive(true);
```

但不要指望它做业务探活，真正的检测靠应用层心跳。

## 1.3 SO_REUSEADDR 与 SO_BACKLOG

`SO_REUSEADDR` 允许在 `TIME_WAIT` 状态下重新绑定端口，服务重启时不用等 2MSL：

```java
ServerSocket server = new ServerSocket();
server.setReuseAddress(true);          // 必须在 bind 之前设置
server.bind(new InetSocketAddress(9000));
```

**注意**：`SO_REUSEADDR` 不等于 `SO_REUSEPORT`。前者解决"重启时端口被 TIME_WAIT 占用"，后者允许多个进程/线程同时绑定同一端口做负载均衡（Linux 3.9+）。两者语义不同，不要混用。

`SO_BACKLOG` 控制全连接队列长度，队列满了新连接会被丢弃：

```java
ServerSocket server = new ServerSocket(9000, 4096);   // backlog = 4096
// 仍需配合内核参数 somaxconn
```

```bash
# 查看与调整内核上限
sysctl net.core.somaxconn
sysctl -w net.core.somaxconn=4096
```

| 参数 | 作用 | 调大后的收益 | 风险 |
|------|------|--------------|------|
| `SO_BACKLOG` | 已完成握手待 accept 的连接数 | 抗突发连接 | 内存占用略增 |
| `somaxconn` | backlog 的内核上限 | 同上 | 无 |
| `tcp_max_syn_backlog` | 半连接队列 | 抗 SYN Flood | 内存略增 |

## 1.4 缓冲区

`SO_RCVBUF` 和 `SO_SNDBUF` 决定单连接能缓冲多少数据。太小会限制吞吐，太大则增加延迟和内存占用。

```java
socket.setReceiveBufferSize(256 * 1024);
socket.setSendBufferSize(256 * 1024);
```

Linux 默认开启自动调优，`tcp_rmem` / `tcp_wmem` 会动态调整。手动设置会关闭自动调优，通常只在明确知道带宽时延积时才做。

```text
所需缓冲区 ≈ 带宽 × RTT
例：1Gbps × 40ms = 5MB
```

| 现象 | 可能原因 | 调整方向 |
|------|----------|----------|
| 高 RTT 链路吞吐上不去 | 窗口小于带宽时延积 | 增大接收缓冲区 |
| 延迟抖动大 | 缓冲区过大产生 bufferbloat | 减小缓冲区，启用 AQM |
| 内存随连接数暴涨 | 每连接缓冲区过大 | 降低单连接缓冲或共享池 |

# 2. 线程模型

## 2.1 三种常见模型

```text
① 每连接一线程
  连接A → 线程1
  连接B → 线程2
  连接C → 线程3

② 线程池 + 阻塞 I/O
  连接(排队) → 工作线程池(固定 N 个) → 处理

③ Reactor（NIO/Netty）
  I/O 线程(少量) → 事件 → 业务线程池
```

| 模型 | 线程数 | 适用连接数 | 主要问题 |
|------|--------|------------|----------|
| 每连接一线程 | 与连接数相同 | 数百 | 内存与切换成本高 |
| 线程池 + 阻塞 I/O | 固定 | 数百到数千 | 慢连接占住线程，池满排队 |
| Reactor | 固定且少 | 万级以上 | 编程复杂，需处理半包与背压 |

## 2.2 线程池大小怎么算

业务线程池的大小取决于"每条请求的 CPU 时间"和"等待 I/O 的比例"，可以用 Little 定律估算：

```text
按并发估算：线程数 ≈ QPS × 平均处理时间（秒）
按资源估算：线程数 ≈ 核数 × (1 + 等待时间 / 计算时间)
```

前者算出"同一时刻有多少请求在途"，后者算出"多少线程能把 CPU 喂饱"，取两者中较小的合理值，再用压测校准。

| 负载类型 | 粗略配置 |
|----------|----------|
| CPU 密集 | 核数 + 1 |
| I/O 密集 | 核数 × (1 + 平均等待/平均计算)，常见 2-8 倍 |
| 混合 | 按业务拆分多个池，分别配置 |

**注意**：线程数不是越多越好。线程过多会导致上下文切换和缓存失效，吞吐反而下降。必须用压测找拐点，而不是凭公式拍脑袋。

## 2.3 I/O 线程与业务线程分离

Netty 的 EventLoop 线程负责读写和编解码，业务逻辑应丢到独立线程池：

```java
EventLoopGroup business = new DefaultEventExecutorGroup(16);

bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline()
          .addLast(new FrameDecoder())
          // 业务 Handler 绑定到独立线程组，避免阻塞 I/O 线程
          .addLast(business, new BusinessHandler());
    }
});
```

**注意**：在 EventLoop 里做数据库查询、同步 HTTP 调用或 `Thread.sleep`，会阻塞该 EventLoop 上的所有连接，症状是"少数慢请求导致整批连接超时"。

# 3. 连接数与端口

## 3.1 文件句柄上限

每条 TCP 连接占一个文件描述符。默认 `ulimit -n` 常见 1024，远不够。

```bash
ulimit -n              # 查看当前限制
ulimit -n 65535       # 临时调整
```

```ini
# /etc/security/limits.conf 持久化
* soft nofile 65535
* hard nofile 65535
```

## 3.2 客户端端口耗尽

主动发起连接的机器受**临时端口范围**限制，默认约 28000 个。大量短连接会让 `TIME_WAIT` 占满端口：

```bash
sysctl net.ipv4.ip_local_port_range      # 如 32768 60999
sysctl -w net.ipv4.ip_local_port_range="10240 65000"
```

| 问题 | 表现 | 处理 |
|------|------|------|
| 连接数上不去 | `Too many open files` | 调 `ulimit -n` |
| 端口耗尽 | `Cannot assign requested address` | 连接复用、扩大端口范围、多 IP |
| 全连接队列溢出 | 客户端超时、服务端无日志 | 调大 backlog、加快 `accept()` |

> **行业实践**：网关类服务通常把"监听端口"和"出站连接"分到不同机器或不同 IP，因为出站连接数受端口限制，而监听连接数只受文件句柄限制。这是一个容易被忽略的容量规划点。

# 4. 缓冲区与批处理

## 4.1 应用层缓冲

每次 `write` 都是一次系统调用。小数据频繁写，系统调用开销会超过实际传输时间。

```java
// 错误示范：每条消息一次 write + flush
for (String msg : messages) {
    out.write(msg);
    out.flush();
}

// 正确做法：攒一批，一次写出
BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(out, StandardCharsets.UTF_8));
for (String msg : messages) {
    writer.write(msg);
    writer.newLine();
}
writer.flush();
```

但缓冲也有代价：数据留在用户态，没及时发出会增加延迟。**批处理的本质是"用延迟换吞吐"**，需要按业务在两者间取平衡。

## 4.2 合并写与 writev

一次 `write` 传多个缓冲区，可以用 `writev` 减少系统调用和拷贝。Java 的 `SocketChannel`（实现 `GatheringByteChannel`）、`FileChannel` 和 Netty 的 `CompositeByteBuf` 都能做到：

```java
// 头部和正文分属两个 ByteBuffer，用 writev 一次交给内核
ByteBuffer header = ByteBuffer.wrap(headerBytes);
ByteBuffer payload = ByteBuffer.wrap(body);
// SocketChannel 实现了 GatheringByteChannel，可一次传多个缓冲区
ByteBuffer[] buffers = {header, payload};
while (header.hasRemaining() || payload.hasRemaining()) {
    socketChannel.write(buffers);   // 返回值可能小于总长度，循环直到写完
}
```

# 5. 背压

## 5.1 没有背压会发生什么

如果生产者发得比消费者处理得快，数据会堆在某个队列里。队列无界，内存最终被撑爆；队列有界，就要决定"满了怎么办"。

```text
生产者（快） ──> 无界队列 ──> 消费者（慢）
                    │
                    └── 队列无限增长 → OOM
```

## 5.2 背压的三种手段

| 手段 | 做法 | 适用 |
|------|------|------|
| 阻塞生产者 | 队列满时 `put()` 阻塞 | 线程模型简单、可接受反压传导 |
| 丢弃/降级 | 队列满时丢最旧或返回错误 | 实时性优先、允许丢数据 |
| 通知上游降速 | TCP 窗口、`isWritable`、响应式流 | 长连接、流式处理 |

Netty 用**高低水位线**实现可写性通知：

```java
// 当待发送字节超过高水位，Channel 变为不可写；低于低水位恢复可写
bootstrap.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
        new WriteBufferWaterMark(32 * 1024, 64 * 1024));

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    if (!ctx.channel().isWritable()) {
        // 下游写不动了，暂停读取，让 TCP 窗口自然收缩
        ctx.channel().config().setAutoRead(false);
        return;
    }
    process(msg);
}
```

**设计考量**：背压必须逐层传导。如果中间某一层用了无界队列，上游就感知不到下游的压力，背压形同虚设。设计时要问一句："这条链路上，从最慢的一环到最快的一环，压力怎么传回去？"

# 6. 零拷贝

## 6.1 传统读写的四次拷贝

```text
磁盘/网卡 ──> 内核缓冲区 ──> 用户缓冲区 ──> 内核 socket 缓冲区 ──> 网卡
              ①            ②              ③                    ④
```

传统 `read` + `write` 至少经历四次拷贝和四次上下文切换。零拷贝的目标是减少这些拷贝。

## 6.2 常见零拷贝手段

| 层次 | 手段 | 减少的拷贝 |
|------|------|------------|
| 系统调用 | `sendfile` | 跳过用户态，内核直接送网卡 |
| 系统调用 | `splice` | 管道间零拷贝 |
| Java NIO | `FileChannel.transferTo/From` | 底层走 `sendfile` |
| Java NIO | `MappedByteBuffer` | mmap，用户态与内核共享页 |
| Netty | `FileRegion` | 文件传输走 `sendfile` |
| Netty | `CompositeByteBuf` | 逻辑合并，避免拼包拷贝 |
| Netty | `slice()/duplicate()` | 共享底层内存，不复制 |

```java
// 文件下载：直接在内核里把文件送到 socket，不经过用户态
try (FileChannel file = FileChannel.open(Paths.get("big.zip"));
     SocketChannel socket = SocketChannel.open(address)) {
    long position = 0;
    long size = file.size();
    while (position < size) {
        // transferTo 返回实际传输字节数，可能小于请求量，必须循环
        position += file.transferTo(position, size - position, socket);
    }
}
```

**注意**：`transferTo` 不保证一次传完，返回值可能小于请求长度，必须循环直到传完。另外，开启 TLS 后数据要加密，`sendfile` 无法直接使用，零拷贝收益会消失——这也是 HTTPS 下载比 HTTP 更吃 CPU 的原因之一。

# 7. 压测与监控

## 7.1 压测要看的指标

| 指标 | 含义 | 关注点 |
|------|------|--------|
| QPS/TPS | 每秒处理请求数 | 拐点在哪里 |
| P50/P95/P99 延迟 | 分位延迟 | 平均值会掩盖长尾 |
| 错误率 | 超时、重置比例 | 先于吞吐恶化出现 |
| CPU/内存/GC | 资源水位 | 是否成为瓶颈 |
| 连接数 | 建立/活跃/等待 | 是否泄漏 |
| 重传率 | `netstat -s` 中 retrans | 网络质量问题 |

**注意**：压测只看平均值没有意义。P99 从 20ms 涨到 2s，平均值可能只从 20ms 涨到 50ms，但用户体验已经崩了。

## 7.2 常用工具与命令

```bash
# 连接状态统计
ss -s
ss -tan state established | wc -l
ss -tan state time-wait | wc -l

# TCP 重传统计
netstat -s | grep -i retrans

# 队列溢出
netstat -s | grep -i "listen overflows\|listen drops"

# 网卡与带宽
sar -n DEV 1

# JVM GC
jstat -gcutil <pid> 1000
```

```bash
# 用 wrk 做 HTTP 压测（长连接 + 多线程）
wrk -t8 -c1000 -d30s --latency http://127.0.0.1:8080/api/ping
```

| 工具 | 适用 | 特点 |
|------|------|------|
| `wrk` | HTTP 压测 | 轻量、高并发、支持长连接 |
| `JMeter` | 复杂场景 | 图形化、支持多种协议 |
| `ab` | 简单基准 | 功能少，不适合高并发 |
| 自定义客户端 | 私有协议 | 完全可控，需自己做统计 |

## 7.3 排错顺序

性能问题不要一上来就调参数，按这个顺序排查：

```text
① 现象确认：哪个指标异常？P99 还是吞吐？
② 分段定位：客户端、网络、服务端、下游，哪一段慢？
③ 资源检查：CPU、内存、GC、句柄、连接数
④ 协议检查：重传、窗口、队列溢出
⑤ 应用检查：线程池、锁、慢查询、序列化
⑥ 最后才是调参
```

> **行业实践**：绝大多数"网络慢"最终定位到的是应用层：线程池打满、锁竞争、下游超时、日志同步刷盘。先排除应用问题，再怀疑内核参数。

# 8. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 所有场景都关 Nagle | 小包增多、带宽浪费 | 按延迟敏感度决定 |
| 手动设巨大缓冲区 | 延迟升高、内存暴涨 | 按带宽时延积设置 |
| 无界队列做缓冲 | OOM | 有界队列 + 背压 |
| 在 I/O 线程做阻塞操作 | 所有连接被拖慢 | 业务线程池隔离 |
| `transferTo` 只调一次 | 文件传输不完整 | 循环直到传完 |
| 压测只看平均值 | 长尾问题被掩盖 | 看 P95/P99 |
| 先调参后定位 | 掩盖真实问题 | 先分段定位 |
| 忽略重传率 | 误判为应用慢 | 检查 `netstat -s` |

# 9. 练习与自检

**练习一**：用同一个小请求分别开启和关闭 `TCP_NODELAY`，各压测 30 秒，比较 P99 延迟差异。

**练习二**：把 `SO_RCVBUF` 设成 4KB，在 100ms RTT 的链路上传输大文件，用 `ss -tin` 观察窗口变化，解释吞吐受限的原因。

**练习三**：给 Netty 服务端设置 `WriteBufferWaterMark(8KB, 16KB)`，让下游消费变慢，观察 `isWritable()` 何时变为 false，验证背压是否生效。

自检问题：Nagle 与延迟 ACK 为什么会叠加出 40ms？线程数公式里"等待/计算"比值代表什么？`transferTo` 在 HTTPS 下为什么失效？

# 10. 小结

| 主题 | 要点 |
|------|------|
| Nagle | 延迟敏感关闭，带宽敏感保留，优先应用层攒批 |
| 套接字参数 | `REUSEADDR` 解决重启，`BACKLOG` 抗突发，缓冲区按 BDP 算 |
| 线程模型 | Reactor 管 I/O，业务线程池隔离阻塞操作 |
| 容量 | 文件句柄与临时端口是两套上限，分别规划 |
| 背压 | 有界队列 + 水位线，压力必须逐层传导 |
| 零拷贝 | `sendfile`/`transferTo` 省拷贝，TLS 下失效 |
| 压测 | 看 P99 与错误率，先定位再调参 |

参数调优只能把系统推到它应有的上限，真正决定上限的是架构与代码。最后一篇把整个 TCP 模块串成知识地图，补上抓包排错与生产检查清单，并给出高频自检题。
