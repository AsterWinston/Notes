长连接最危险的时刻不是断开，而是"看起来没断"。对端机房断电、NAT 表项超时、中间设备静默丢包，TCP 连接在本地依然是 `ESTABLISHED`，写操作也不报错，可数据永远不会到达。心跳、空闲检测和断线重连就是为这个场景准备的三件套；会话恢复和重复连接控制，则决定重连之后业务能不能无缝继续。

# 1. 为什么长连接需要心跳

## 1.1 TCP 保活的局限

TCP 协议层有 keepalive：长时间没有数据往来时，内核发探测包，若干次无响应就判定连接死亡。但它的默认参数几乎不适合业务使用：

```bash
# Linux 默认值：空闲 7200 秒后才开始探测
sysctl net.ipv4.tcp_keepalive_time    # 7200 秒
sysctl net.ipv4.tcp_keepalive_intvl   # 75 秒
sysctl net.ipv4.tcp_keepalive_probes  # 9 次
```

按默认值，一条死连接要 **2 小时 11 分钟** 才被发现。即使调小参数，TCP keepalive 也只能检测"链路是否还通"，无法判断"对端业务进程是否还健康"。

| 维度 | TCP keepalive | 应用层心跳 |
|------|---------------|------------|
| 检测对象 | 链路和内核 | 对端应用是否存活 |
| 默认时间 | 2 小时 + | 秒级到分钟级 |
| 携带信息 | 无 | 可带负载、版本、时间戳 |
| 跨语言 | 系统调用，需逐连接开启 | 协议层实现，完全可控 |
| 中间设备 | 可能过滤探测包 | 走正常数据通道 |

**设计考量**：`SO_KEEPALIVE` 可以作为兜底打开，但业务探活必须靠应用层心跳。两者不是替代关系，而是"最后一层保险"与"主检测手段"的关系。

## 1.2 半开连接与中间设备

所谓**半开连接**（half-open connection），是指一端认为连接还在，另一端已经消失或不再响应。常见成因：

```text
客户端             NAT/防火墙            服务端
  │   ESTABLISHED     表项存在            │
  │ ───────────────────────────────────> │
  │                   表项超时被清理       │  服务端重启
  │  （客户端仍以为是通的）                │
  │ ──────── 数据发进黑洞 ──────────────X │
```

- **NAT 表项超时**：家用路由器通常 5-30 分钟清理一次空闲映射，之后客户端发出的包找不到出口。
- **中间设备静默丢包**：负载均衡、防火墙会丢弃长时间空闲的连接状态。
- **对端进程崩溃**：操作系统会发 RST，这种情况客户端能感知；但机器断电、网络分区不会。
- **GC 停顿或线程池打满**：TCP 层还活着，业务已经无法响应。

心跳正是为了解决"TCP 层看起来正常、业务层已经失联"这一类问题。

# 2. 心跳设计

## 2.1 间隔与超时的取舍

心跳太频繁浪费带宽和 CPU，太稀疏则故障发现慢。经验公式：

```text
心跳间隔 T
读空闲超时 ≈ 2.5T ~ 3T
断线判定时间 ≈ 读空闲超时 + 重连等待
```

| 场景 | 心跳间隔 | 读空闲超时 | 说明 |
|------|----------|------------|------|
| 即时通讯 | 30 秒 | 90 秒 | 兼顾耗电与实时性 |
| 移动端后台 | 4-5 分钟 | 15 分钟 | 受 NAT 与省电策略限制 |
| 服务间 RPC | 5-10 秒 | 30 秒 | 内网，追求快速发现 |
| 物联网设备 | 60 秒 | 180 秒 | 设备多、网络差 |

**注意**：读空闲超时必须大于心跳间隔，否则正常心跳还没到就被判定超时。第 5.2 节的 Netty 示例里，`IdleStateHandler` 的读空闲时间常被误设成与心跳相同，结果是"每隔一个心跳就断一次"。

## 2.2 ping/pong 与业务心跳

心跳有两种形态：

| 形态 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| 协议级 ping/pong | 独立控制帧，不带业务语义 | 轻量、易识别 | 只能证明进程活着 |
| 业务心跳 | 复用业务消息，带状态字段 | 顺便上报指标、版本 | 需要业务配合 |

一个最小的心跳协议通常长这样：

```text
客户端                          服务端
  │ ─── PING(seq=1, ts=...) ───> │
  │ <── PONG(seq=1, ts=...) ──── │
  │                              │
  │  （超过 3 个周期没有 PONG，视为断线）
```

带上 `seq` 和时间戳有两个好处：可以计算 RTT 监控链路质量，也能在重连后识别迟到的心跳响应。

## 2.3 双向检测

只让客户端发心跳是不够的。服务端也需要检测客户端是否失联，否则大量死连接会一直占着文件描述符和内存。

```text
┌──────────┐   心跳请求    ┌──────────┐
│  客户端   │ ───────────> │  服务端   │
│          │ <─────────── │          │
└──────────┘   心跳响应    └──────────┘
     │                          │
     └─ 检测响应超时              └─ 检测读空闲
        （客户端视角）               （服务端视角）
```

两个方向各设一个超时，任何一侧判定死亡都主动关闭连接。这样即使一侧代码有 bug，另一侧也能兜底清理。

# 3. 断线重连

## 3.1 触发条件

不是所有异常都该重连，也不是所有关闭都值得重试：

| 触发事件 | 是否重连 | 说明 |
|----------|----------|------|
| 读空闲超时 | 是 | 典型半开连接 |
| `Connection reset` | 是 | 对端异常退出 |
| 心跳连续无响应 | 是 | 业务失联 |
| 服务端主动关闭（业务下线） | 视协议而定 | 若带"不再重连"标志则停止 |
| 认证失败 | 否 | 重连也过不了鉴权，应先刷新凭证 |
| 客户端主动关闭 | 否 | 正常退出 |
| 网络不可用 | 等待网络恢复 | 监听网络状态，避免无效重试 |

## 3.2 指数退避与抖动

固定间隔重连有两个问题：服务端故障期间，所有客户端每秒重试一次，等于持续 DDoS；服务端恢复瞬间，大量重连同时涌入，可能再次把它压垮。

**指数退避**（exponential backoff）让等待时间随失败次数增长：

```text
第 1 次失败 → 1 秒
第 2 次失败 → 2 秒
第 3 次失败 → 4 秒
第 4 次失败 → 8 秒
...
第 n 次失败 → min(base × 2^(n-1), maxDelay)
```

**抖动**（jitter）在等待时间上加随机量，打散同时重连的客户端：

```text
实际等待 = 退避时间 × (0.5 + random × 0.5)
```

```java
public class BackoffPolicy {
    private final long baseMs;
    private final long maxMs;
    private int attempt = 0;

    public BackoffPolicy(long baseMs, long maxMs) {
        this.baseMs = baseMs;
        this.maxMs = maxMs;
    }

    public long nextDelayMs() {
        // 指数增长，防止 long overflow：先限制指数上限
        int exp = Math.min(attempt, 20);
        long delay = Math.min(maxMs, baseMs * (1L << exp));
        // 抖动：0.5 ~ 1.0 倍，避免重连风暴
        delay = (long) (delay * (0.5 + Math.random() * 0.5));
        attempt++;
        return delay;
    }

    public void reset() {
        attempt = 0;   // 连接成功后必须重置，否则下次断线直接等最大值
    }
}
```

| 策略 | 优点 | 缺点 |
|------|------|------|
| 固定间隔 | 实现简单 | 故障期压力恒定，恢复时惊群 |
| 线性退避 | 压力渐进 | 大故障下等待仍偏短 |
| 指数退避 | 快速缓解压力 | 恢复后重连偏慢 |
| 指数 + 抖动 | 兼顾压力与分散 | 实现稍复杂 |

> **行业实践**：几乎所有成熟客户端（Kafka、gRPC、WebSocket 库）都采用"指数退避 + 抖动 + 最大上限"。纯指数退避在服务端长时间不可用时会把间隔拉到几分钟，用户体感很差，所以要设上限（常见 30 秒到 1 分钟）。

## 3.3 重连风暴

重连风暴指大量客户端在同一时刻重连，压垮刚恢复的服务端。除了抖动，还可以用这些手段：

- 在服务端做**接入限流**，按 IP 或租户限制每秒新建连接数。
- 客户端重连前先做一次**健康探测**，确认服务端可用再建连。
- 服务端扩缩容时采用**分批重启**，不要一次性摘掉所有实例。
- 重连成功后**延迟注册**，错开业务请求洪峰。

# 4. 会话恢复

## 4.1 连接 ID 与 token

断线重连会建立一条全新的 TCP 连接，服务端默认无法知道"这个人刚才就在"。要恢复会话，协议里需要一个跨连接稳定的标识。

```text
首次连接：
  客户端 ─── login(token=xxx) ───> 服务端
  服务端 ─── welcome(sessionId=A1, resumeToken=R1) ───> 客户端

断线重连：
  客户端 ─── resume(sessionId=A1, token=R1, lastSeq=1024) ───> 服务端
  服务端 ─── resumeOk(fromSeq=1025) ───> 客户端（补发 1025 之后的消息）
```

| 字段 | 作用 |
|------|------|
| `sessionId` | 标识一次业务会话，跨多条 TCP 连接 |
| `resumeToken` | 防止他人冒用 `sessionId` |
| `lastSeq` | 客户端最后处理到的消息序号，用于补发 |
| `expireAt` | 会话保留时间，过期后只能重新登录 |

## 4.2 重复连接

移动网络下常见"旧连接还没死、新连接已经建好"的情况，如果不处理，服务端会认为同一用户在线两次，消息可能发到旧连接上。

处理方式有两种：

- **新连接踢掉旧连接**：以 `sessionId` 为 key，注册新连接时关闭同 key 的旧连接。适合"同一账号只能一处登录"。
- **多端并存**：按 `(userId, deviceId)` 维护连接集合，允许多端同时在线，但消息要按设备路由。

```java
// 服务端注册连接时的去重逻辑
public void register(String sessionId, Channel newChannel) {
    Channel old = sessions.put(sessionId, newChannel);
    if (old != null && old.isActive() && old != newChannel) {
        old.writeAndFlush("KICKED: duplicate login");   // 通知旧连接
        old.close();                                     // 主动关闭，释放资源
    }
}
```

**注意**：关闭旧连接时不要简单粗暴地发 RST。先发一条业务层的"被踢下线"消息，让客户端知道原因，避免它立刻又发起重连，形成"踢了重连、重连又被踢"的死循环。

# 5. Netty 的 IdleStateHandler

Netty 内置了空闲检测处理器，能在指定时间内没有读、写或读写事件时触发 `IdleStateEvent`。

```java
import io.netty.channel.*;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;
import io.netty.handler.timeout.IdleStateHandler;
import java.util.concurrent.TimeUnit;

public class HeartbeatInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) {
        ch.pipeline()
          // 参数依次为：读空闲、写空闲、读写空闲（秒）
          // 这里读空闲 90 秒，写空闲 30 秒
          .addLast(new IdleStateHandler(90, 30, 0, TimeUnit.SECONDS))
          .addLast(new HeartbeatHandler());
    }
}

class HeartbeatHandler extends ChannelInboundHandlerAdapter {
    private static final byte[] PING = "PING\n".getBytes();
    private int idleCount = 0;

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (!(evt instanceof IdleStateEvent)) {
            super.userEventTriggered(ctx, evt);
            return;
        }
        IdleStateEvent e = (IdleStateEvent) evt;
        if (e.state() == IdleState.WRITER_IDLE) {
            // 写空闲：主动发心跳，同时累计无响应次数
            if (++idleCount > 3) {
                ctx.close();              // 连续多次没有收到响应，断开重连
            } else {
                ctx.writeAndFlush(PING);
            }
        } else if (e.state() == IdleState.READER_IDLE) {
            // 读空闲：超过 90 秒没收到任何数据，认为对端失联
            ctx.close();
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        idleCount = 0;                    // 收到任何数据（包括 PONG）都重置计数
        ctx.fireChannelRead(msg);
    }
}
```

参数含义：

| 参数 | 触发事件 | 典型设置 |
|------|----------|----------|
| `readerIdleTime` | 多久没读到数据 | 心跳间隔 × 3 |
| `writerIdleTime` | 多久没写出数据 | 心跳间隔 |
| `allIdleTime` | 多久既没读也没写 | 一般设为 0，前两个够用 |

**注意**：`IdleStateHandler` 的读空闲判断的是"没收到**任何**数据"，不是"没收到心跳包"。如果这条连接上还有其他高频业务数据，读空闲要设得比心跳间隔大得多，否则正常业务流量会让检测永远不触发，反而失去作用。

# 6. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| 只依赖 TCP keepalive | 死连接两小时才发现 | 应用层心跳 30-90 秒 |
| 读空闲 ≤ 心跳间隔 | 正常连接被误杀 | 读空闲 = 2.5~3 倍心跳 |
| 固定间隔重连 | 服务端恢复瞬间被打垮 | 指数退避 + 抖动 |
| 成功后不重置退避 | 下次断线等几分钟 | 连接成功即 `reset()` |
| 重连不做鉴权 | 旧 token 失效后无限重试 | 区分认证失败与网络失败 |
| 不做重复连接处理 | 同一用户收到重复消息 | 按 sessionId 去重或踢旧连接 |
| 踢旧连接不发通知 | 客户端立即重连形成死循环 | 先发 kick 消息再关闭 |
| 心跳包走业务队列 | 业务繁忙时心跳发不出 | 心跳走独立通道或高优先级 |

# 7. 练习与自检

**练习一**：用 `BackoffPolicy` 模拟 10 次连接失败，打印每次等待时间，观察上限与抖动的效果。

**练习二**：写一个客户端，连接后每隔 30 秒发 `PING`，服务端 90 秒读空闲。用防火墙或 `iptables` 丢弃服务端返回包，观察客户端多久判定断线并开始重连。

**练习三**：用 Netty 搭一服务端，把 `IdleStateHandler` 的读空闲设成 10 秒、客户端心跳 10 秒，复现"每个周期断一次"的问题，再改成 30 秒对比。

自检问题：TCP keepalive 为什么不能替代应用层心跳？指数退避为什么必须加抖动？`sessionId` 和 `connectionId` 的区别是什么？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 心跳的必要性 | TCP 只保证链路，不保证对端业务存活 |
| 心跳参数 | 间隔 30-90 秒，读空闲为间隔的 2.5-3 倍 |
| 空闲检测 | 读写双向检测，任一方向失联都能发现 |
| 重连策略 | 指数退避 + 抖动 + 上限，成功即重置 |
| 会话恢复 | sessionId + token + 序号，实现断点续传 |
| 重复连接 | 按 sessionId 去重，踢旧连接前先通知 |

连接能自动恢复之后，新的问题变成"为什么吞吐上不去、为什么延迟抖动"。下一篇进入性能调优，从 Nagle、缓冲区、线程模型一直讲到背压、零拷贝与压测方法。
