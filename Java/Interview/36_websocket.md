本篇覆盖 WebSocket 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：协议与握手、服务端实现、集群与性能、安全与运维、选型与演进。HTTP 协议基础与升级机制见 28_http.md（2.1.2），Nginx 代理配置见 12_nginx.md（2.2.1/2.2.3），网关的 WebSocket 路由见 19_spring_cloud_gateway.md（1.2.1），长连接的容器线程模型见 5_tomcat.md。

# 1. 协议与握手

## 1.1 定位与握手

### 1.1.1 WebSocket 是什么？和 HTTP、SSE、轮询的本质区别？

**一句话**：WebSocket 是**在一条 TCP 连接上做全双工消息通信**的协议——**握手借 HTTP（Upgrade），之后脱离 HTTP 语义**，双方可随时互发"帧"。

| 方案 | 方向 | 连接 | 实时性 | 开销 | 适用 |
|------|------|------|--------|------|------|
| **短轮询** | 客户端拉 | 每次新连接 | 差（间隔） | 高（每次 HTTP 头） | 兼容性兜底 |
| **长轮询** | 客户端拉（挂起） | 长连接（反复重建） | 中 | 中 | 老环境降级 |
| **SSE** | **服务器→客户端单向** | HTTP 长连接（`text/event-stream`） | 高 | 低（文本流） | 通知/行情/日志流 |
| **WebSocket** | **双向** | 一条 TCP 长连接 | 高 | 低（帧头 2~14 字节） | IM/协同/游戏/实时交互 |

**本质差异（答题要有层次）**：① **协议层**：HTTP 是"请求-响应"（服务端不能主动发），WebSocket 是"消息帧"（双方对等）；② **连接层**：WebSocket 握手后**复用同一条 TCP**，没有 HTTP 的头开销与队头语义；③ **语义层**：WebSocket 只提供"有序的字节/文本消息流"，**可靠性（ACK/重发）、路由、鉴权、心跳**都要应用层自己做——这也是它"简单又难"的地方。

**WebSocket 的代价**：**有状态**（连接在哪台机器、内存里存会话）——水平扩展、发布、负载均衡都比无状态 HTTP 难（块 3 展开）；**代理/防火墙**对长连接不友好（超时/不支持 Upgrade）；**重连/心跳/背压**全要自己实现。

### 1.1.2 握手过程与 Sec-WebSocket-Accept 的算法？101 之后发生了什么？

**握手（HTTP/1.1 Upgrade）**：

```text
客户端 → 服务端：
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==      # 16 字节随机数 Base64
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat           # 可选：子协议
Origin: https://example.com                       # 浏览器自动带，服务端要校验

服务端 → 客户端（同意升级）：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Accept 的计算（必背）**：`Base64(SHA1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`——**固定 GUID 拼接 + SHA-1 + Base64**；作用是**证明服务端真的懂 WebSocket**（不是误转发的 HTTP 服务器/缓存），同时避免代理缓存把"普通响应"当握手成功。

**101 之后**：**同一条 TCP 连接**切换为 WebSocket 帧协议——不再有 HTTP 请求行/头，**双方随时可发帧**；连接的生命周期由**关闭帧 + TCP FIN** 结束（1.2.2）。**HTTP/2 的 WebSocket**（RFC 8441 Extended CONNECT）让 WS 跑在 h2 流上（多路复用/复用 TLS），但**浏览器/代理支持仍不普遍**，主流仍是 h1 升级。

**三个易错**：① 握手是 **HTTP GET**（不是 POST），且 **101 不是 200**；② `Sec-WebSocket-Key` 是**随机数**（不是密钥，不参与加密），服务端**不能**拿它当认证；③ 鉴权放在**握手阶段**（Cookie/Token/子协议），升级后没有 HTTP 头可用（4.1.1）。

## 1.2 帧与连接管理

### 1.2.1 帧结构、Opcode、分片（FIN）、掩码（MASK）各是什么？

**帧头（2~14 字节，按位）**：

```text
FIN(1) RSV1-3(3) Opcode(4) | MASK(1) PayloadLen(7)
[扩展长度 16/64 位] [Masking Key 4 字节（仅客户端→服务端）] [Payload]
```

**Opcode（帧类型）**：`0x0` 续帧（分片的后续）、`0x1` 文本、`0x2` 二进制、`0x8` 关闭、`0x9` Ping、`0xA` Pong；**控制帧（8/9/A）不能分片、负载 ≤125 字节**。

**分片（Fragmentation）**：一条大消息可拆成"首帧（FIN=0）+ 若干续帧（Opcode=0）+ 末帧（FIN=1）"——用于**流式发送/避免一次性大缓冲**；接收方按"同一条消息的所有帧 Opcode 连续"重组。**控制帧可穿插在分片中间**（心跳不被大消息阻塞）。

**掩码（MASK）**：**客户端→服务端的帧必须掩码**（4 字节随机 key 与 payload 逐字节 XOR），**服务端→客户端禁止掩码**。**为什么**：防止**缓存投毒/代理混淆**——早期中间代理可能把 WS 帧误当 HTTP 请求缓存，掩码让攻击者**无法构造可预测的字节**去污染共享缓存/CDN（RFC 6455 的安全设计）。

**Payload 长度**：`≤125` 直接用 7 位；`126` 后跟 16 位；`127` 后跟 64 位——**协议支持超大消息，但生产要设上限**（防内存打爆，4.1.2）。

**易错**：认为"帧=消息"（一条消息可能多帧）；服务端给客户端发帧也加掩码（协议禁止，客户端会断）；控制帧分片/超过 125 字节（协议错误）；不校验 RSV 位（扩展协商后才有意义）。

### 1.2.2 关闭握手、Ping/Pong、超时与重连怎么做？

**关闭握手（优雅关闭）**：一方发 **Close 帧（0x8，带状态码/原因）** → 另一方回 Close → **再关 TCP**；发起方等对方 Close 或超时后强制关闭。**状态码**：`1000` 正常、`1001` 服务端下线/离开、`1002` 协议错误、`1003` 不支持的数据类型、`1006` **异常关闭（无 Close 帧，只能由本地推断）**、`1011` 服务端内部错误。

**Ping/Pong（协议级心跳）**：任一方可发 **Ping（0x9）**，对端**必须尽快回 Pong（0xA）**（可带相同 payload）；用途：**探测死连接**（半开连接：NAT/代理超时、客户端崩溃但 TCP 未断）+ **保活**（让中间的 LB/代理不因 idle 超时断连）。**浏览器的 JS API 不能主动发 Ping**（只能由服务端发、浏览器自动回 Pong），所以**应用层心跳**（业务 JSON `ping/pong`）在前端也很常见；**STOMP 有自己的 heart-beat 协商**（2.1.2）。

**超时与重连（客户端侧）**：
```text
连接：ws:// → wss://（TLS）；握手带 token/cookie
心跳：每 30s 发应用层 ping；连续 2~3 次无 pong → 判定断线
重连：指数退避（1s→2s→4s…封顶 30s）+ 抖动；网络恢复/页面可见时立即重试
恢复：带 lastMessageId/seq 拉取离线消息（避免"重连丢消息"，2.2.2）
```

**服务端侧**：`IdleStateHandler`（Netty）检测读写空闲——**读空闲超时发 Ping/关连接，写空闲发 Ping**（2.1.3 的代码）；Spring 的 `WebSocketSession` 有 `setMaxIdleTimeout`（容器相关）/STOMP 心跳；**别只依赖 TCP keepalive**（默认 2 小时，太慢）。

**易错**：只做 TCP keepalive（代理早就断了）；心跳太频繁（流量/CPU）或太稀疏（死连接占资源）；重连无退避（服务端雪崩时全员同时重连——**重连风暴**，20_sentinel.md 的重试风暴同源）；重连不做消息补偿（丢消息）。

# 2. 服务端实现

## 2.1 Java 生态

### 2.1.1 Spring WebSocket 的两种用法（原生 Handler vs STOMP）？

**方式一：原生 WebSocketHandler（贴近协议）**：
```java
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/ws")
                .addInterceptors(new AuthHandshakeInterceptor())
                .setAllowedOrigins("https://app.example.com");   // 别用 *
    }
}

public class MyHandler extends TextWebSocketHandler {
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    @Override public void afterConnectionEstablished(WebSocketSession s) {
        sessions.put(userId(s), s);            // 会话注册（单机）
    }
    @Override protected void handleTextMessage(WebSocketSession s, TextMessage msg) {
        // 所有异常要自己兜；sendMessage 非线程安全 → 用 ConcurrentWebSocketSessionDecorator
    }
    @Override public void afterConnectionClosed(WebSocketSession s, CloseStatus st) {
        sessions.remove(userId(s));
    }
}
```
**适用**：自定义协议、二进制、极致控制；**代价**：路由/订阅/心跳/并发发送都要自己写。

**方式二：STOMP（消息语义，2.1.2）**：`@EnableWebSocketMessageBroker` + `@MessageMapping` + `@SendTo`——**发布订阅/用户目标/拦截器**开箱即用，适合"业务消息"（聊天/通知）；**代价**：多一层协议、SimpleBroker 单机、要理解目的地（destination）语义。

**HandshakeInterceptor**：握手前从 **HTTP 请求**取 Cookie/Token/Header → 校验 → 放入 `attributes`（`WebSocketSession.getAttributes()`）——**这是 WebSocket 鉴权的标准入口**（升级后没有 Header，4.1.1）。

**并发发送的坑**：`WebSocketSession.sendMessage` **不是线程安全**——多线程同时推同一会话会抛 `IllegalStateException`（TEXT_PARTIAL_WRITING）；Spring 提供 **`ConcurrentWebSocketSessionDecorator`**（包装 + 发送超时 + 缓冲上限）——"消息偶发发送失败"的常见根因。

### 2.1.2 STOMP 的帧、订阅、用户目标消息怎么工作？

**STOMP 帧格式**：`COMMAND\nheader:value\n\nBody\0`——常用命令：`CONNECT`/`CONNECTED`、`SUBSCRIBE`/`UNSUBSCRIBE`、`SEND`、`MESSAGE`（服务端推）、`ACK`/`NACK`、`DISCONNECT`、`ERROR`。

**Spring 的目的地模型（三个前缀）**：
```java
registry.enableSimpleBroker("/topic", "/queue");     // 客户端订阅的 broker 前缀
registry.setApplicationDestinationPrefixes("/app");  // 客户端 SEND 到服务端 @MessageMapping
registry.setUserDestinationPrefix("/user");          // 用户私有
```
- 客户端 `SEND /app/chat.send` → `@MessageMapping("/chat.send")` 方法；
- 方法 `@SendTo("/topic/room/1")` → 广播给订阅该目的地的客户端；
- **用户目标**：`@SendToUser("/queue/notifications")` 或 `SimpMessagingTemplate.convertAndSendToUser(userId, "/queue/notifications", msg)`——**按 Principal 路由到该用户的会话**（多端/多会话都能收到），客户端订阅 `/user/queue/notifications`。

**认证与授权**：**握手时**用 `HandshakeInterceptor`/`ChannelInterceptor` 解析 Token → 设置 `Principal`（`StompHeaderAccessor.setUser`）；**订阅时**用 `ChannelInterceptor` 校验"这个用户能否订阅这个目的地"（否则任何人都能订阅别人的 `/user/{id}/...`）。

**Broker 的两种**：**SimpleBroker（内存）**——单机、不支持集群（多实例各自广播，消息不互通）；**外部 Broker（RabbitMQ/ActiveMQ 的 STOMP relay）**——多实例共享订阅/路由，生产集群必选（3.1.2 的另一种解）。

**SockJS**：给"不支持 WebSocket 的旧浏览器/代理"提供 **HTTP 降级**（xhr-streaming/xhr-polling）——**现代浏览器已普遍支持 WS，SockJS 更多是历史兼容**；新项目优先原生 WS + 应用层重连（Spring 文档也建议评估）。

**易错**：SimpleBroker 上生产多实例（A 实例推的消息 B 实例的用户收不到）；`/user` 目的地不配 Principal（发送失败/广播）；订阅不做权限校验（越权订阅）；STOMP 心跳没开（长连接被代理断）。

### 2.1.3 Netty 怎么实现 WebSocket 服务？

**Pipeline 骨架**：
```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() {
   protected void initChannel(SocketChannel ch) {
     ch.pipeline()
       .addLast(new HttpServerCodec())                       // HTTP 编解码（握手用）
       .addLast(new HttpObjectAggregator(65536))             // 聚合 HTTP 请求
       .addLast(new IdleStateHandler(60, 30, 0))             // 读空闲 60s / 写空闲 30s
       .addLast(new WebSocketServerProtocolHandler("/ws", null, true, 65536))
       // ↑ 处理握手、升级、帧编解码、Ping/Pong、Close
       .addLast(new MyWebSocketFrameHandler());              // 业务帧处理
   }
 });
```
**业务 Handler**：`SimpleChannelInboundHandler<TextWebSocketFrame>`——处理文本帧；`ChannelGroup` 管理所有连接（广播）；`AttributeKey` 存 userId；`ctx.channel().writeAndFlush(new TextWebSocketFrame(json))` 推送；**写操作要在 EventLoop 线程**（或 `channel.eventLoop().execute`），**别在业务线程直接写**（线程安全/顺序）。

**心跳**：`IdleStateHandler` + `userEventTriggered`（2.1 的代码）——**READER_IDLE 关连接、WRITER_IDLE 发 Ping**；也可应用层 JSON 心跳（前端可控）。

**优雅关闭**：`channel.close()` 前发 `CloseWebSocketFrame(1001, "server shutdown")` → 等待 → 关 TCP；**先摘 LB/注册中心**，再逐连接关闭（4.2.2）。

**易错**：`WebSocketServerProtocolHandler` 的路径与握手 URL 不一致（404/400）；消息太大没设 `maxFramePayloadLength`（默认 64KB，大消息被拒）；`ChannelGroup` 里残留已关闭连接（广播异常）；在 IO 线程做阻塞业务（拖死 EventLoop，23_docker.md 的响应式纪律同源）。

## 2.2 连接与消息治理

### 2.2.1 心跳、空闲检测、会话管理怎么做？

**三层心跳（按可靠性）**：**TCP keepalive**（内核级，默认 2 小时，太慢）；**协议 Ping/Pong**（WebSocket 原生，服务端可发、浏览器自动回）；**应用层心跳**（业务 JSON，前端可控、可带时间戳/RTT）——**生产通常"协议/应用层 + 代理 idle 超时对齐"**。

**会话管理（单机 vs 集群）**：
- 单机：`Map<userId, WebSocketSession>` + `ConcurrentHashMap`；**用户多端**用 `Map<userId, Set<session>>`；
- 集群：**连接注册表**（Redis：`userId → nodeId` / `sessionId → userId,nodeId`，带 TTL）——**消息路由靠它**（3.1.2）；**不要让"会话"成为唯一真相**（节点挂了要能清理）；
- **会话与业务解耦**：连接只负责"收/发"，业务状态（在线/离线、最后消息）放 Redis/DB；**节点无状态化**（尽量）才能弹性扩缩。

**空闲检测参数**：`IdleStateHandler(readerIdle, writerIdle, allIdle)`——读空闲=客户端没发任何帧（含 Pong），写空闲=服务端没发；**读空闲超时要大于"客户端心跳间隔 × 容忍次数"**（别误杀弱网）。

**易错**：心跳间隔小于代理 idle 超时（代理先断，客户端还以为连着）；只检测"连接存在"不检测"业务可用"（半开连接）；会话 Map 不清理（内存泄漏/广播到死连接）；多端会话只存一个（另一端的消息丢）。

### 2.2.2 消息可靠性与顺序（ACK/重发/幂等）怎么设计？

**先认清**：WebSocket 只保证"**连接内、帧级有序**"，**不保证"消息必达"**（断线/进程崩溃/网络抖动都会丢）——业务可靠性要自己做。

**可靠消息的四件套**：
1. **序号（seq）**：每条消息带**单调递增 seq**（会话级/用户级），接收方按 seq 检测**空洞/重复**；重连时带 `lastSeq` 请求补发（**离线消息/补偿**）；
2. **ACK**：接收方收到后回 `ack(seq)`；发送方维护"未确认窗口"，超时重发（**至少一次**）；
3. **幂等**：消息带 `messageId`（UUID/业务键），接收方去重（3.2.1 的幂等表/唯一键）；
4. **顺序**：**同一会话/同一房间单线程发送**（或按 key 哈希到同一队列），避免"多线程写导致乱序"；跨节点用**单分区 MQ/Redis Stream** 保序。

**离线与多端**：在线走 WS，离线走 **Push（APNs/FCM）** + 上线拉取（`seq > lastAck`）；多端同步用"**每端独立游标**"（同一用户不同设备的已读位置不同）。

**背压（Backpressure）**：客户端消费慢时，服务端**不能无限缓冲**——`ConcurrentWebSocketSessionDecorator` 的 buffer 上限 + 发送超时；超限**断开/降级**（丢非关键消息/提示重连）——"服务端 OOM 因为某个慢客户端"是真实事故。

**易错**：把 TCP 可靠当"消息必达"（进程崩溃/重连就丢）；只重发不幂等（重复消费）；seq 全局共享（多会话乱序）；无背压（慢客户端拖垮服务端）；离线消息只存内存（重启即丢）。

# 3. 集群与性能

## 3.1 水平扩展

### 3.1.1 百万连接怎么支撑？单机瓶颈与系统参数怎么调？

**先算账（单机能扛多少）**：连接数上限由**内存（每连接会话/缓冲区，KB 级）、fd、CPU（TLS/帧解析）、带宽、GC**共同决定——Netty NIO 单机 **10 万级**是常见基线（调优后可更高），**百万连接=横向多节点**，不是"一台机器硬扛"。

**六层优化（从应用到内核）**：
1. **线程模型**：**NIO/Epoll + 少量 EventLoop 线程**（≈CPU 核数），**绝不一连接一线程**（10 万线程直接崩）；业务处理丢到业务线程池（别阻塞 EventLoop，2.1.3）；
2. **内存**：每连接的对象/缓冲尽量小；**避免每连接大 ByteBuf**；用池化（Netty PooledByteBufAllocator）；**背压**防慢客户端堆积（2.2.2）；
3. **fd 与端口**：`ulimit -n`（百万级）、`fs.file-max`、`net.ipv4.ip_local_port_range`（主动连接场景）、`tcp_max_syn_backlog`/`somaxconn`（握手队列）；
4. **TCP 参数**：`tcp_rmem/wmem`（缓冲）、`tcp_tw_reuse`（客户端侧）、`tcp_keepalive_*`（配合应用心跳）；**TIME_WAIT/CLOSE_WAIT** 的治理（30_linux.md 2.3.2）；
5. **TLS**：wss 的握手/加解密吃 CPU——**AES-NI**、会话复用（session ticket）、TLS 1.3；能内网明文就内网（入口做 TLS 终结，12_nginx.md）；
6. **GC/对象**：避免每消息创建大对象/字符串拼接；堆外缓冲；G1/ZGC 按延迟选（2_jvm.md 2.3）。

**压测口径**：**连接数 + 消息速率**两个维度分别压（Gatling/k6/tsung/wscat 集群）；观察**握手成功率、P99 推送延迟、GC、fd、带宽**；**单机基线 × 节点数 × 0.7** 做容量规划（35_project_experience.md 1.1.2）。

**易错**：只压"连接数"不压"消息速率"（连接挂着不费 CPU，消息才费）；忽略**代理/LB 的连接上限**（Nginx `worker_connections`、云 LB 并发）；百万连接但**广播**（O(N) 扇出）把 CPU 打满——广播要走 MQ/分片。

### 3.1.2 多实例下消息怎么路由/广播？（连接注册表、Pub/Sub、MQ）

**问题**：用户 A 连在节点 1、用户 B 连在节点 2——B 给 A 发消息，节点 2 怎么找到 A 的连接？

**三种方案（按规模）**：
1. **连接注册表 + 定向转发**：Redis 维护 `userId → nodeId`（多端：`Set<sessionId>`），消息到节点 2 → 查注册表 → **调节点 1 的内部接口**（HTTP/gRPC）→ 节点 1 推给本地会话；**优点**：精准、无广播；**代价**：多一次内部调用、注册表要 TTL/心跳清理；
2. **Redis Pub/Sub（按节点频道）**：每个节点订阅 `ws:node:{nodeId}`；任意节点`PUBLISH` 到目标节点频道——**简单**，但 **Redis Pub/Sub 不持久**（节点短暂断连丢消息）、无 ACK；
3. **MQ（按用户/节点分区）**：消息发到 MQ（Kafka/RocketMQ），消费者按 `userId` 分区——**同一用户的消息有序**、可重放/削峰；**代价**：多一跳延迟、要处理重复（2.2.2 的幂等）。

**广播/群聊**：**小群**（<几百）可"查成员 → 逐个定向"；**大群/直播间**：**不能 O(N) 逐连接推**——走 **MQ 广播 + 节点本地扇出**（每个节点只推自己持有的成员），或 **边缘推流/CDN**；**在线状态（presence）**用 Redis Set/Hash + 心跳 TTL。

**节点故障**：注册表 TTL 过期 + 客户端重连到新节点重新注册；**未送达消息**靠"离线消息 + seq 补偿"（2.2.2），而不是指望注册表。

**答题收口**："**路由靠注册表、扇出靠 MQ、可靠靠补偿**——三者分工清楚，集群版 WebSocket 就成立了。"

### 3.1.3 负载均衡与粘滞会话怎么配？（Nginx/网关/云 LB）

**核心矛盾**：WebSocket 是**长连接 + 有状态**——连接建立后**必须一直落在同一后端**（粘滞），否则消息推错节点。

**四层/七层做法**：
- **Nginx**：`upstream` + **`ip_hash`**（按客户端 IP 粘滞）或 `hash $arg_token consistent`（按用户/Token 一致性哈希）；**`proxy_http_version 1.1` + `Upgrade/Connection` 头**（3.2.1）；`proxy_read_timeout` 大于心跳间隔；
- **云 LB**：**ALB/NLB 的 sticky session**（cookie）或 **TCP 模式**（四层转发，天然粘滞）；注意**空闲超时**（ALB 默认 60s，要调大或靠心跳保活）；
- **K8s Ingress/Gateway**：`nginx.ingress.kubernetes.io/affinity: cookie` + `proxy-read-timeout` 注解；Spring Cloud Gateway 用 `ws://` 路由（19_spring_cloud_gateway.md 1.2.1）；
- **无粘滞方案**：**连接注册表 + 定向转发**（3.1.2）——连接落在哪台都行，消息按注册表路由；**扩容/发布更灵活**，但多一次内部调用。

**粘滞的代价**：**负载不均**（大客户/热点 IP 集中）、**节点故障时连接全丢**（要重连）、**发布时连接迁移**（4.2.2 的优雅下线）；所以**"注册表 + 无粘滞"是更可扩展的方向**，粘滞只适合小规模/简单场景。

**易错**：LB 空闲超时小于心跳（连接被静默断）；只做 IP hash（NAT 后大量用户同 IP → 全落一台）；扩容后旧连接不迁移（新节点闲置）；健康检查把"能握手"当"业务可用"（要探活业务）。

## 3.2 代理与网络

### 3.2.1 Nginx/网关代理 WebSocket 的配置与坑？

**Nginx 最小配置**：
```nginx
location /ws/ {
    proxy_pass http://ws_backend;
    proxy_http_version 1.1;                          # 必须 1.1 才支持 Upgrade
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";           # 或 $connection_upgrade（map 处理）
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_read_timeout 3600s;                        # 大于心跳间隔，别用默认 60s
    proxy_send_timeout 3600s;
    proxy_buffering off;                             # 长连接/流式不要缓冲
}
```
**五个坑**：① `proxy_http_version` 默认 1.0（不带 Upgrade，握手失败）；② `Connection` 头写死 `upgrade` 会让普通 HTTP 请求也带（用 `map $http_upgrade $connection_upgrade`）；③ **`proxy_read_timeout` 默认 60s**——没有心跳/心跳间隔大于它，连接被 Nginx 断（表现为"每分钟断一次"）；④ 多级代理（CDN/WAF/LB）都要支持 Upgrade 且超时对齐；⑤ **K8s Ingress 注解**要配 `proxy-read-timeout`/`proxy-buffering`（不同 Controller 注解不同）。

**网关（Spring Cloud Gateway）**：路由 `uri: ws://ws-service` + `lb://`（WebSocket 负载均衡），**注意 `spring.cloud.gateway` 的 WebSocket 路由与普通 HTTP 路由分开**；网关的**响应式线程不能被阻塞**（19_spring_cloud_gateway.md 2.2.3 的纪律）。

**调试**：`curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Key: ..." -H "Sec-WebSocket-Version: 13" http://host/ws` 看是否 101；`wscat`/浏览器 Network 的 **101 帧**；抓包看握手头（30_linux.md 2.3.3）。

### 3.2.2 弱网/移动端/断线重连怎么优化？

**移动端的三个现实**：**网络切换**（WiFi↔4G，IP 变）、**后台挂起**（iOS 会冻结/断 WS）、**NAT 超时**（运营商网关几分钟无流量就断）——**"长连接永远在线"是幻觉**，必须按"随时会断"设计。

**六条优化**：
1. **重连策略**：指数退避 + 抖动（1s→2s→4s…封顶 30s），**网络恢复/页面可见（visibilitychange）/前台**立即重试；**避免全员同时重连**（抖动）；
2. **心跳自适应**：前台 30s、弱网/后台拉长或停（省电），**心跳间隔 < NAT/LB 超时**；
3. **断点续传**：重连带 `lastSeq`，服务端补发（2.2.2）；**离线消息** + Push 兜底（APNs/FCM）；
4. **消息压缩/二进制**：`permessage-deflate`（CPU 换带宽，弱网值得）+ **Protobuf/MessagePack** 替代 JSON（体积/解析）；
5. **减少往返**：批量消息/合并推送、**增量更新**（只传变化字段）、避免大消息（分片/对象存储引用，25_minio.md）；
6. **省电/保活**：移动端后台不维持 WS（系统会杀），用 **Push 唤醒**；前台再重连拉历史。

**服务端配合**：**连接粒度限流**（每用户最大连接/消息速率）、**慢客户端背压**（2.2.2）、**多端会话独立**（手机/PC 各自游标）。

**易错**：心跳太短（耗电/流量）、太长（被 NAT 断）；重连不带 seq（丢消息）；iOS 后台还指望 WS（系统挂起）；只做 WS 不做 Push（离线用户收不到）。

# 4. 安全与运维

## 4.1 安全

### 4.1.1 握手鉴权、Origin 校验、CSRF/WSS 怎么做？

**握手鉴权的三种载体（浏览器限制：JS 不能自定义 Header）**：
1. **Cookie**（`HttpOnly; Secure; SameSite`）——浏览器自动带，最自然；但**CSRF/CSWSH** 面（4.1.2）；
2. **URL query token**（`wss://host/ws?token=...`）——简单但**会进日志/Referer/代理**，要用**短时效一次性 ticket**（先 HTTP 换票，再握手用票）；
3. **Sec-WebSocket-Protocol（子协议）**——把 token 放子协议里（有些客户端支持），较少用。

**服务端流程**：`HandshakeInterceptor` 解析凭证 → 校验（JWT 验签/Redis 查票）→ 设置 `Principal`/attributes → **拒绝未认证的升级**（返回 401/403，而不是先升级再断）；**升级后**用 `ChannelInterceptor` 校验**订阅/发送权限**（不能只信客户端消息）。

**Origin 校验（防 CSWSH）**：浏览器发起 WS 会带 `Origin`，**服务端必须校验白名单**——否则恶意站点可以"借用用户 Cookie"建立 WS（**跨站 WebSocket 劫持**，本质是 WS 版的 CSRF）；**不能只靠 CORS**（CORS 不覆盖 WS）。

**WSS**：**生产必须 `wss://`**（TLS）——否则 token/消息明文；证书/SNI/ALPN 与 HTTPS 同（28_http.md 3.2.2）；**入口 TLS 终结**（Nginx/云 LB），内网可明文但要隔离。

**易错**：把 token 放 URL 且长期有效（日志泄露）；只校验握手不校验消息（越权操作）；`setAllowedOrigins("*")`（CSWSH）；相信客户端传的 userId（必须从凭证推导）。

### 4.1.2 常见攻击（连接耗尽/消息洪泛/跨站劫持）怎么防？

| 攻击 | 手法 | 防护 |
|------|------|------|
| **连接耗尽** | 大量假连接占满 fd/内存 | **握手前鉴权**（未认证直接拒）、**每 IP/用户连接数上限**、握手速率限制、LB/WAF 限流 |
| **消息洪泛** | 单连接高频发消息打满 CPU/下游 | **每连接/用户消息速率限制**（令牌桶）、**消息大小上限**（帧/业务层）、慢客户端背压/断开 |
| **跨站 WS 劫持（CSWSH）** | 恶意页面借用户 Cookie 连 WS | **Origin 白名单 + token 校验**（4.1.1）、Cookie `SameSite` |
| **大帧/内存打爆** | 超大 payload/分片 | `maxFramePayloadLength`、业务消息大小上限、**分片上限** |
| **慢握手（Slowloris）** | 握手不完成占连接 | 握手超时、连接数上限、WAF |
| **注入/越权** | 消息内容注入、订阅他人主题 | 输入校验/参数化、**订阅授权**（2.1.2）、最小权限 |
| **重放** | 截获消息重发 | 消息 ID/时间戳/nonce + 幂等（2.2.2） |

**通用底线**：**认证在握手、授权在每条消息/订阅、限流在连接与消息两层、大小有上限、日志可审计**——五件事齐了，WS 的安全面就收敛了。

## 4.2 运维

### 4.2.1 监控、日志、压测怎么做？

**四组指标（按"连接→消息→业务"）**：
1. **连接**：当前连接数（按节点/用户）、**握手成功率/失败原因**、连接建立/断开速率、平均连接时长；
2. **消息**：收发速率（msg/s、bytes/s）、**推送延迟 P99**（从产生到送达）、发送失败/超时、**背压丢弃数**；
3. **资源**：fd 使用率、内存（每连接）、GC、CPU（TLS/编解码）、带宽、EventLoop 队列；
4. **业务**：在线用户数、消息必达率、重连率、离线消息积压。

**日志/追踪**：每条连接带 `sessionId/userId`，日志结构化；**消息链路带 traceId**（跨 HTTP→WS→下游，15_spring_cloud.md 4.2.1），否则"消息丢了"无法定位；**采样**（高频消息不全量打日志）。

**压测**：**连接数 + 消息速率**分开压（3.1.1）；用 Gatling/k6/自研 WS 客户端；**长稳测试**（24h 看内存/GC/fd 泄漏）；**故障注入**（杀节点/断网/慢客户端）。

**告警**：连接数突降（LB/节点故障）、握手失败率、推送延迟、fd/内存水位、离线积压——**按症状告警**（用户视角），不是"CPU 高"（35_project_experience.md 3.2.2）。

### 4.2.2 灰度发布/优雅下线怎么处理长连接？

**问题**：发布要重启节点，但**长连接不能"一刀切"**——否则大量用户同时断线重连（重连风暴）+ 消息丢失。

**优雅下线五步**：
```text
① 从 LB/注册中心摘除该节点（不再接新连接）
② 通知客户端"本节点将下线"（业务消息/Close 1001 "going away"）
③ 等待客户端重连到其他节点（或服务端主动关闭 + 客户端退避重连）
④ 排空在途消息（等 ACK/超时）
⑤ 关闭连接 → 停止进程
```
**K8s 配合**：`preStop` 里执行"摘流 + sleep"、`terminationGracePeriodSeconds` 要覆盖排空时间（32_kubernetes.md 4.2.2）；**滚动发布**一次只动一个节点/一批；**注册表 TTL** 让路由尽快更新（3.1.2）。

**客户端配合**：收到 Close 1001 后**立即重连（带抖动）**；**重连到新节点后拉离线消息**（2.2.2）；前端展示"连接恢复中"而不是报错。

**灰度**：按用户/比例把新版本节点接入 LB（权重），观察**连接成功率/推送延迟/错误率**；**蓝绿**需要两套入口（连接切换更复杂，通常用滚动+排空）。

**易错**：直接 `kill -9`（连接全断、消息丢）；preStop 没摘流（新连接还进来）；所有节点同时重启（全站断连）；客户端重连无抖动（瞬间打满新节点）。

# 5. 选型与演进

## 5.1 对比

### 5.1.1 WebSocket、SSE、长轮询怎么选？

**决策表（按需求排序）**：

| 需求 | 首选 | 理由 |
|------|------|------|
| **双向实时**（聊天/协同/游戏/交易） | **WebSocket** | 全双工、低开销、二进制 |
| **只推不收**（通知/行情/日志/进度） | **SSE** | 普通 HTTP、自动重连、过代理简单 |
| **老环境/代理不支持 WS** | **长轮询**（或 SockJS 降级） | 兼容性最好，代价是开销 |
| **服务间流式**（非浏览器） | **gRPC streaming** | 多路复用、强类型、生态 |
| **P2P/低延迟音视频** | **WebRTC** | 媒体通道/数据通道，NAT 穿透 |

**SSE 被低估的三个优点**：**基于 HTTP**（不需要 Upgrade，代理/网关友好）、**自动重连 + `Last-Event-ID` 续传**（浏览器原生）、**HTTP/2 下多路复用**（不占额外连接）；**缺点**：单向、只文本（二进制要 Base64）、连接数受浏览器限制（HTTP/1.1 下 6 个/域）。

**WebSocket 的"隐性成本"**：有状态（3.1）、心跳/重连/背压自己写、代理/防火墙支持不一、移动端后台限制（3.2.2）——**"能双向"不等于"该双向"**：很多"推送"需求用 SSE + HTTP 上行就够，简单一个量级。

**收口**："**要双向选 WS，只要推选 SSE，兼容兜底长轮询**——先把'方向'和'环境'两个问题答了，再谈性能。"

### 5.1.2 WebTransport/HTTP3、MQTT 这些新方案怎么定位？

**WebTransport（基于 HTTP/3/QUIC）**：提供**双向流 + 数据报（unreliable datagram）**——**无 TCP 队头阻塞、0-RTT、连接迁移**（1.2.2 的 QUIC 特性在 Web 的落地）；适合**实时游戏/音视频/低延迟双向**；**现状**：浏览器支持不完整（Chrome/Edge 较好，Safari/Firefox 滞后）、服务端/代理生态早期——**生产要评估兼容性与回退**（回退 WS）。

**MQTT（over WebSocket）**：**轻量发布订阅协议**（QoS 0/1/2、保留消息、遗嘱消息、主题层级）——**IoT/移动弱网**的标准；浏览器通过 **MQTT over WebSocket** 接入（MQTT broker 如 EMQX/Mosquitto）；**与业务 WS 的区别**：MQTT 是"协议级 pub/sub + 可靠性语义"，WebSocket 只是"传输"——**IoT/大量设备**选 MQTT，**业务 IM/通知**通常自建 WS + 应用层协议（或 STOMP）。

**其他定位**：**gRPC bidirectional streaming**（HTTP/2）——服务间/桌面端，不适合浏览器直连；**SSE over HTTP/2**——服务端推送的现代基线；**WebRTC DataChannel**——P2P/低延迟，NAT 穿透复杂；**SignalR/Socket.IO**——带降级/重连/房间的封装（生态换依赖）。

**收口**："**WebSocket 是'浏览器双向'的默认答案，MQTT 是'设备/弱网 pub-sub'的答案，WebTransport 是'QUIC 时代'的候选**——选型看终端类型与网络环境，不看热度。"

## 5.2 收口

### 5.2.1 设计一个 IM/实时通知系统怎么落地？

**分层架构（按消息流）**：
```text
客户端（Web/App）──wss──▶ 接入层（WS Gateway，无状态尽量）
                             │ ① 握手鉴权（ticket/JWT）
                             │ ② 连接注册（userId/sessionId → node，Redis）
                             ▼
                        消息服务（Message Service）
                             │ ③ 分配 seq/messageId（幂等）
                             │ ④ 持久化（MySQL/Cassandra：会话/消息/已读）
                             │ ⑤ 在线路由（注册表→目标节点）或离线（Push + 拉取）
                             ▼
                        扇出/推送（小群直接、大群 MQ/广播）
                             │ ⑥ 客户端 ACK → 服务端标记送达/重试
                             ▼
                        存储/分析（历史、未读、审计、监控）
```

**六个关键设计**：
1. **消息 ID 与顺序**：**会话级 seq**（单调递增）+ 全局 messageId（雪花/UUID）；**同一会话单分区/单线程**保序（2.2.2）；
2. **可靠投递**：**至少一次 + ACK + 重发 + 幂等**；离线消息存 DB/Redis，上线按 `lastSeq` 拉取；**Push 兜底**（APNs/FCM）；
3. **在线状态**：Redis `userId → nodeId`（TTL/心跳）、`presence` 集合；节点故障靠 TTL + 重连清理；
4. **群聊扇出**：**小群**逐成员路由；**大群**（万人）**不要逐连接推**——MQ 广播到节点、节点本地扇出（3.1.2），或读扩散（拉模式）；
5. **多端同步**：每端独立游标（已读/未读）、**漫游**（历史分页）、**端到端加密**（如需要，密钥管理是另一层）；
6. **安全与治理**：握手鉴权、订阅授权、消息大小/频率限制、敏感词/审核、审计日志（4.1）。

**通知系统（更轻）**：**在线 WS 推 + 离线 Push + 站内信（DB）**；**模板/去重/聚合**（避免轰炸）；**送达/点击回执**；**灰度/开关**（4.2.2）。

**易错**：消息只存内存（重启全丢）；群聊逐连接推（大群打爆）；ACK 只到服务端不到发送方（发送方不知道送达）；多端已读串（游标不独立）；离线只靠 Push（用户关闭通知就丢）。

### 5.2.2 WebSocket 面试高频追问清单（自测表）

| 追问 | 一句话答案 | 详看 |
|------|-----------|------|
| 握手为什么需要 Key/Accept？ | 证明服务端懂 WS、防代理误缓存 | 1.1.2 |
| 为什么客户端要掩码？ | 防缓存投毒/代理混淆 | 1.2.1 |
| 一条消息多大？分片？ | 帧可分片；生产设消息上限 | 1.2.1/4.1.2 |
| 怎么检测死连接？ | Ping/Pong + 应用心跳 + 读空闲 | 1.2.2/2.1.3 |
| 断线怎么不丢消息？ | seq + ACK + 离线拉取 + 幂等 | 2.2.2 |
| 怎么支撑百万连接？ | NIO/EventLoop、fd/内核调优、横向扩展 | 3.1.1 |
| 多实例怎么找连接？ | Redis 注册表 + 定向转发/MQ | 3.1.2 |
| 要不要粘滞会话？ | 小规模可；大规模用注册表无粘滞 | 3.1.3 |
| Nginx 怎么配？ | 1.1 + Upgrade/Connection + 超时 | 3.2.1 |
| 怎么鉴权？ | 握手 ticket/JWT + Origin 校验 + 订阅授权 | 4.1.1 |
| 怎么防 CSWSH？ | Origin 白名单 + token（CORS 不管 WS） | 4.1.2 |
| 发布怎么不断连？ | 摘流 → Close 1001 → 排空 → 重启 | 4.2.2 |
| Spring 用原生还是 STOMP？ | 自定义协议用原生；业务 pub/sub 用 STOMP | 2.1.1/2.1.2 |
| Netty 和 Spring 怎么选？ | 高并发/底层控制 Netty；业务快速 Spring | 2.1.3 |
| WS 还是 SSE？ | 双向 WS，单向 SSE | 5.1.1 |
| 弱网/移动端？ | 退避重连、心跳自适应、Push 兜底 | 3.2.2 |

**答题纪律**：**先给结论（一句话）→ 再讲机制（2~3 点）→ 最后讲代价/边界**——WebSocket 的题最容易"背协议细节但讲不清工程取舍"，**把"有状态带来的扩展/发布/安全成本"主动说出来**，就超过了大多数候选人。