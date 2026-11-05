单机 WebSocket 很直观：所有连接都在同一个 JVM 里，广播就是遍历一个 Map。一旦部署两个实例，用户 A 连在节点 1、用户 B 连在节点 2，节点 1 发出的消息永远到不了 B。集群化要解决的不是"连接怎么复制"，而是**连接落在哪里、消息怎么跨节点、会话状态放在哪、顺序怎么保证**这四件事。

# 1. 单机模型的边界

## 1.1 连接是有状态的

HTTP 服务可以随意扩容，因为每个请求自带上下文，落到哪台机器都一样。WebSocket 不同：一条连接从握手到关闭都绑定在**某一个进程**上，连接对象、会话属性、发送缓冲都在那个进程的内存里。

```text
              ┌─────────────┐
客户端 A ───> │  节点 1      │  只有 A 的连接
客户端 B ───> │  节点 2      │  只有 B 的连接
              └─────────────┘
节点 1 想给 B 发消息：找不到 B 的连接
```

## 1.2 集群后暴露的三类问题

集群后会暴露三类问题：消息到不了，因为广播只在本进程内；重连丢状态，因为房间与订阅只存在本地内存；顺序错乱，因为多节点并发发布且缺少统一序号。

**设计考量**：不要试图"把 WebSocket 连接做成无状态"。连接本身必然有状态，能做的是把**业务会话状态**外置，让任意节点都能接管重连。连接是短暂的，会话是持久的，两者要分开设计。

# 2. 多实例广播

## 2.1 整体架构

```text
         ┌──────────┐   ┌──────────┐   ┌──────────┐
         │ 节点 1    │   │ 节点 2    │   │ 节点 3    │
         │ 连接池    │   │ 连接池    │   │ 连接池    │
         └────┬─────┘   └────┬─────┘   └────┬─────┘
              │ publish       │              │
              └───────────────┼──────────────┘
                              ▼
                    ┌──────────────────┐
                    │  Redis Pub/Sub   │
                    │  ws:room:{room}  │
                    └────────┬─────────┘
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
           节点 1          节点 2          节点 3
         （各自投递给本机订阅了该房间的连接）
```

核心思路：**消息不直接找连接，而是发到一个所有节点都能收到的通道**。每个节点收到后，只投递给本机持有的、属于该房间的连接。

## 2.2 Redis Pub/Sub

Redis 的发布订阅模型非常适合这个场景：`PUBLISH channel message` 把消息发给所有订阅者，订阅者用 `SUBSCRIBE` 或 `PSUBSCRIBE` 接收，延迟通常在毫秒级。但它有两个硬限制：**不持久化**，没有订阅者时消息直接丢弃，节点重启期间的消息不会补发；**不保证跨发布者顺序**，多个节点同时 `PUBLISH`，订阅者收到的顺序不固定。因此它适合"在线广播"，不适合"必须送达"的业务消息，后者要用 Stream、Kafka 或 RabbitMQ，并配合第 5 篇的序号与补偿。

## 2.3 发布端实现

```java
@Service
public class ClusterBroadcaster {
    private final StringRedisTemplate redis;
    private final ObjectMapper mapper;
    private final String instanceId;

    public ClusterBroadcaster(StringRedisTemplate redis, ObjectMapper mapper,
                              @Value("${app.instance-id}") String instanceId) {
        this.redis = redis; this.mapper = mapper; this.instanceId = instanceId;
    }

    public void broadcast(String roomId, WsMessage message) throws Exception {
        // msgId 用于去重，origin 用于跳过本实例
        var env = new Envelope(UUID.randomUUID().toString(), instanceId, roomId, message);
        // 按房间分频道，减少无关节点收到的消息量
        redis.convertAndSend("ws:room:" + roomId, mapper.writeValueAsString(env));
    }

    public record Envelope(String msgId, String origin, String roomId, WsMessage payload) { }
    public record WsMessage(String type, String from, String content, long seq) { }
}
```

## 2.4 订阅端实现

房间是动态的，用模式订阅 `ws:room:*` 可以避免为每个房间单独管理监听器：

```java
@Component
public class RoomMessageListener implements MessageListener {
    private final ObjectMapper mapper;
    private final LocalSessionRegistry registry;
    private final String instanceId;

    public RoomMessageListener(ObjectMapper mapper, LocalSessionRegistry registry,
                               @Value("${app.instance-id}") String instanceId) {
        this.mapper = mapper; this.registry = registry; this.instanceId = instanceId;
    }

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            var env = mapper.readValue(message.getBody(), Envelope.class);
            // 发布端已在本地直接发送过，跳过自己的消息
            if (instanceId.equals(env.origin())) return;
            registry.sendToRoom(env.roomId(), env.payload());
        } catch (Exception e) {
            // 单条消息解析失败不能影响后续消息消费
            e.printStackTrace();
        }
    }
}
```

订阅侧用一个 `RedisMessageListenerContainer` 注册 `PatternTopic("ws:room:*")` 即可，一条模式订阅覆盖所有房间频道，不必为每个房间单独管理监听器。

> **注意**：使用 `origin` 跳过自己时，发布端必须保证"本地投递成功"再发布，否则本地投递失败而远程成功，会出现本机用户收不到、其他节点用户收到的诡异现象。更稳妥的做法是**本地不直接发，统一由 Redis 回到所有节点**，用一点延迟换取一致性。

## 2.5 从 Pub/Sub 升级

| 方案 | 可靠性 | 顺序 | 复杂度 | 适用 |
|------|--------|------|--------|------|
| Redis Pub/Sub | 最多一次 | 不保证 | 低 | 在线广播、可丢消息 |
| Redis Stream | 可持久化、可重放 | 单流有序 | 中 | 需要补发的聊天 |
| Kafka | 高吞吐、分区有序 | 分区内有序 | 高 | 大规模消息平台 |
| RabbitMQ | 可持久化、路由灵活 | 队列内有序 | 中 | 企业消息、复杂路由 |

**设计考量**：不要一上来就上 Kafka。日活几万的聊天，Redis Pub/Sub 加数据库补偿已经够用；真正需要的是想清楚"消息丢了业务能不能接受"，而不是盲目追求组件先进。

# 3. 会话共享

## 3.1 哪些状态要外置

| 状态 | 是否外置 | 说明 |
|------|----------|------|
| WebSocket 连接对象 | 否 | 无法序列化，只能留在本机 |
| 用户所在节点 | 是 | 路由消息、踢重复连接要用 |
| 房间成员关系 | 是 | 重连到任意节点都要恢复 |
| 最后序号 `lastSeq` | 是 | 断线补发依赖它 |
| 离线消息缓冲 | 是 | 实例重启后仍能补发 |
| 认证信息与过期时间 | 是 | 避免重连后重复登录 |

连接对象本身不能共享，但"这条连接代表谁、在哪个房间、处理到哪"完全可以放进 Redis。

## 3.2 Redis 结构设计

```text
user:{userId}:conn     ->  hash  { nodeId, sessionId, connectedAt, lastActiveAt }
room:{roomId}:members  ->  set   { userId1, userId2, ... }
session:{sessionId}    ->  hash  { userId, lastSeq, resumeToken, expireAt }
room:{roomId}:recent   ->  stream 最近 N 条消息，用于补发
```

```java
@Service
public class SessionStore {
    private final StringRedisTemplate redis;
    public SessionStore(StringRedisTemplate redis) { this.redis = redis; }

    public void bindConnection(String userId, String nodeId, String sessionId) {
        String key = "user:" + userId + ":conn";
        redis.opsForHash().putAll(key, Map.of(
                "nodeId", nodeId, "sessionId", sessionId,
                "connectedAt", String.valueOf(System.currentTimeMillis())));
        // 设置过期，防止实例崩溃后留下永久脏数据
        redis.expire(key, Duration.ofHours(2));
    }

    public void joinRoom(String roomId, String userId) {
        redis.opsForSet().add("room:" + roomId + ":members", userId);
    }

    public void leaveRoom(String roomId, String userId) {
        redis.opsForSet().remove("room:" + roomId + ":members", userId);
    }
}
```

## 3.3 本地缓存与一致性

每次广播都查 Redis 会很慢，节点通常会在本地缓存"本机连接属于哪个房间"。加入房间时先写 Redis 再更新本地缓存，离开时先删本地再删 Redis；节点崩溃后 Redis 可能残留成员关系，需要靠心跳过期或定时清理。

> **行业实践**：房间成员关系用"本地为准、Redis 汇总"的模型。广播时只发本机连接，Redis 只用于跨节点通知和重连恢复，不要求 Redis 中的成员集合实时精确。这样能避免每次消息都做一次分布式查询。

# 4. 网关与负载均衡

## 4.1 四层与七层

WebSocket 握手是 HTTP，因此七层网关可以正常处理；握手完成后，网关只是透明转发 TCP 字节。四层网关（LVS、云 SLB TCP）只看到 TCP 字节，转发快但不能按路径路由；七层网关（Nginx、Ingress、ALB）能识别 Upgrade 与路径，可以按 Header、Cookie 路由并做鉴权。关键是**升级头要透传，读超时要放大**。

```nginx
# Nginx 反向代理 WebSocket 的关键配置
location /ws/ {
    proxy_pass http://ws_backend;
    proxy_http_version 1.1;
    # 透传 Upgrade，缺少这两行握手一定失败
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    # 长连接不能按默认 60 秒超时断开
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

## 4.2 粘性会话

**粘性会话**（sticky session）让同一客户端始终落到同一后端实例，常见实现有按 IP 哈希的 `ip_hash`、网关写入后端标识的 Cookie 粘性、按用户 ID 哈希的 Header 粘性。`ip_hash` 会让同一 NAT 后的用户集中到一台，Cookie 粘性首次请求仍可能落错，Header 哈希则需要网关支持自定义键。

粘性会话能减少重连后的状态恢复，但**不能替代会话共享**：

- 实例故障时，粘性也救不了，用户必须换节点。
- 扩容缩容会重排哈希，部分用户必然迁移。
- 同一 NAT 出口的用户可能全压在一台上，负载不均。

> **设计考量**：粘性会话是优化，不是正确性保证。正确做法是会话状态外置，让任意节点都能接管；粘性只用来减少补发和缓存预热。把正确性寄托在粘性上，滚动发布时一定出问题。

## 4.3 发布时的连接迁移

滚动发布不能直接杀进程，否则大量客户端同时重连。正确做法是先停止接收新连接，再遍历本机会话，用 `1001`（服务端正在离开）关闭并提示客户端按退避重连。再配合网关的**连接排空**（connection draining），给客户端留出重连窗口；分批发布、每批之间留间隔，能显著降低重连风暴。

# 5. 消息顺序

## 5.1 顺序从哪里来

- **单条连接内**：TCP 保证字节有序，WebSocket 帧有序，消息自然有序。
- **同一节点内多线程发送**：需要串行队列，否则 `send` 交错。
- **跨节点**：Redis Pub/Sub 不保证多个发布者之间的顺序，网络延迟也会让先发的后到。

```text
节点 1 ── seq=1 ──┐
                  ├──> Redis ──> 订阅者可能收到 2, 1
节点 2 ── seq=2 ──┘
```

## 5.2 保序方案

保序有四类方案：单写者把同一会话固定给一个节点，简单但有瓶颈；统一序号用 Redis `INCR` 或数据库序列分配 seq，多一次网络往返；分区有序按会话 ID 哈希到 Kafka 同一分区，需要 MQ；客户端重排则带 seq 缓冲乱序消息，代价是延迟与内存。最常用的是**统一序号 + 客户端按 seq 重排**。

```java
public long nextSeq(String conversationId) {
    // Redis INCR 是原子操作，多节点并发也不会拿到重复序号
    Long seq = redis.opsForValue().increment("seq:" + conversationId);
    if (seq == null) {
        throw new IllegalStateException("无法分配消息序号");
    }
    return seq;
}
```

客户端收到消息后：

```javascript
const buffer = new Map();
let expected = 1;

function onMessage(msg) {
  buffer.set(msg.seq, msg);
  while (buffer.has(expected)) {
    render(buffer.get(expected));   // 按序消费
    buffer.delete(expected++);
  }
  // 缺口长时间补不上，主动请求补发
  if (buffer.size > 100) requestResume(expected);
}
```

## 5.3 顺序与扩展性的取舍

严格全局有序意味着单写者，吞吐上不去。大多数业务只需要**单个会话内有序**，不同会话之间可以并行。按 `conversationId` 分区，既保序又能水平扩展。

**注意**：不要用时间戳排序。不同机器的时钟有偏差，同一毫秒内的消息也无法区分先后。时间戳只能用于展示，不能用于业务排序。

# 6. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| 广播只遍历本地 Map | 跨实例用户收不到 | Redis Pub/Sub 或 MQ |
| 会话只存本地内存 | 重连换节点后状态丢失 | 会话元数据外置 Redis |
| 依赖粘性会话保正确 | 实例故障后大量异常 | 状态外置，粘性只做优化 |
| 网关未透传 Upgrade | 握手 400/200 | 配置 `Upgrade` 与 `Connection` 头 |
| 读超时用默认 60 秒 | 连接每分钟被断 | 放大到小时级 |

# 7. 练习与自检

**练习一**：本地启动两个实例（不同端口），用两个浏览器分别连接，验证不加 Redis 广播时消息收不到，加上后能收到。

**练习二**：把 2.3 节的 `origin` 跳过逻辑去掉，观察本机用户是否会收到重复消息，并解释原因。

**练习三**：设计一个"同一用户只允许一处登录"的方案，写出 Redis 键结构和顶替旧连接时的关闭码。

自检问题：为什么 WebSocket 连接不能像 HTTP 请求一样无状态扩容？粘性会话为什么不能替代会话共享？跨节点保序最轻量的做法是什么？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 连接 | 有状态，绑定单个进程，无法复制 |
| 广播 | 发布到共享通道，各节点投递本机连接 |
| Redis | Pub/Sub 适合在线广播，可靠投递用 Stream/MQ |
| 会话 | 连接元数据、房间、lastSeq 外置 |
| 网关 | 透传 Upgrade，放大超时，七层可路由 |
| 粘性 | 优化手段，不是正确性保证 |
| 顺序 | 会话内统一 seq，客户端重排，不用时间戳 |

集群解决了"消息能不能到达"，但能到达不等于安全。最后一篇收束到鉴权、CORS、CSRF、限流、消息大小与生产排错，并给出整个模块的自检题与总结。
