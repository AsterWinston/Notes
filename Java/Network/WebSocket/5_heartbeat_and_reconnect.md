WebSocket 的长连接比 TCP 更容易"假活"：NAT 表项过期、负载均衡摘除后端、手机切网、服务端 GC 长停顿，都可能让连接在两端看起来正常，实际数据已经进黑洞。心跳负责发现这种半开连接，重连负责把连接重新建起来，而会话恢复与消息补偿决定重连之后业务能不能接着走。本篇聚焦 WebSocket 场景，把这几件事串成一条完整链路。

# 1. 为什么 WebSocket 需要心跳

## 1.1 半开连接

**半开连接**（half-open connection）指一端认为连接仍然有效，另一端已经消失或不再响应：

```text
客户端                NAT/防火墙              服务端
  │   ESTABLISHED       映射存在               │
  │ ───────────────────────────────────────> │
  │                     映射超时被清理          │  实例被摘除
  │  （客户端仍以为在线）                       │
  │ ──────── 数据发进黑洞 ─────────────────X  │
```

常见成因：

- 家用路由器或云 NAT 在 5-30 分钟内清理空闲映射。
- 负载均衡把长时间空闲的连接从后端摘掉，但没发 FIN。
- 服务端进程被 `kill -9`、机器断电，没有机会发送关闭帧。
- 服务端 GC 停顿或线程池打满，TCP 仍在，业务已经不响应。

TCP keepalive 默认要两小时才发现，且只能判断链路，判断不了应用是否健康。WebSocket 必须有**应用层或协议层心跳**。

## 1.2 浏览器 API 的限制

浏览器 `WebSocket` 对象**不暴露 ping/pong**：浏览器收到协议级 ping 会自动回 pong，JS 完全感知不到；JS 也不能主动发送 ping 帧；`onclose` 只有在 TCP 断开或收到关闭帧时才触发，半开连接不会触发。所以浏览器侧只能发**业务心跳**（通常是一条 JSON 消息），服务端收到后回一条 pong 业务消息，这也是前端项目里到处能看到 `{"type":"ping"}` 的原因。

## 1.3 协议心跳与应用心跳

| 形态 | 谁发 | 优点 | 缺点 |
|------|------|------|------|
| WebSocket ping/pong | 服务端或原生客户端 | 帧级轻量，不占业务通道 | 浏览器 JS 无法主动发 |
| STOMP 心跳 | 客户端与服务端 | 框架内置，带 `\n` 帧 | 依赖 STOMP，非 STOMP 场景不可用 |
| 业务心跳 | 任意一端 | 可带版本、负载、时间戳 | 需要业务协议配合 |

**设计考量**：服务端应同时开启协议级 ping 与业务层活跃检测。协议级负责"链路是否通"，业务层负责"这个会话是否还在正常交互"。只做一种，都会留下盲区。

# 2. 服务端检测

## 2.1 STOMP 心跳配置

Spring 的简单代理支持 STOMP 心跳，需要显式提供 `TaskScheduler`，否则不生效：

```java
@Bean
public ThreadPoolTaskScheduler wsHeartbeatScheduler() {
    ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
    scheduler.setPoolSize(1);
    scheduler.initialize();
    return scheduler;
}

@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/topic", "/queue")
            // 服务端发心跳间隔、期望客户端心跳间隔（毫秒）
            .setHeartbeatValue(new long[]{10_000, 10_000})
            .setTaskScheduler(wsHeartbeatScheduler());
}
```

配置后，代理会定期发送 STOMP 心跳，并检测客户端是否按约定发回。超过约定时间没有心跳，会话会被关闭。

**注意**：`setHeartbeatValue` 只对 STOMP 生效，对裸 WebSocket 无效。裸连接要么用容器的 ping 配置，要么自己写定时任务。

## 2.2 会话活跃时间扫描

更通用的做法是给每个会话记录 `lastActiveAt`：任何入站消息到达时刷新时间戳，定时任务每 30 秒扫描一次，超过 90 秒没有更新的会话用 `1001` 关闭。清理逻辑要放在 `finally` 里，保证关闭失败时也能把会话从本地表移除。

## 2.3 参数选择

| 场景 | 心跳间隔 | 读空闲超时 | 说明 |
|------|----------|------------|------|
| 网页聊天 | 30 秒 | 90 秒 | 兼顾实时性与后台标签页节流 |
| 移动端 H5 | 60 秒 | 180 秒 | 省电与弱网优先 |
| 桌面协同 | 20 秒 | 60 秒 | 要求快速发现断线 |
| 服务间推送 | 10 秒 | 30 秒 | 内网，追求快速故障转移 |

读空闲超时必须大于心跳间隔，通常取 2.5-3 倍。设成相等会导致"心跳还在路上就被判超时"，表现为每隔一个周期断一次。

# 3. 客户端心跳

## 3.1 浏览器实现

业务心跳要同时解决"定期发"和"多久没回就判死"：

```javascript
class Heartbeat {
  constructor(ws, { interval = 30000, timeout = 10000 } = {}) {
    this.ws = ws;
    this.interval = interval;
    this.timeout = timeout;
    this.pongTimer = null;
  }

  start() {
    // 定期发 ping，并在 timeout 内等 pong
    this.timer = setInterval(() => {
      if (this.ws.readyState !== WebSocket.OPEN) return;
      this.ws.send(JSON.stringify({ type: 'ping', ts: Date.now() }));
      this.pongTimer = setTimeout(() => this.ws.close(4000, 'heartbeat timeout'), this.timeout);
    }, this.interval);
  }

  onMessage() {
    // 收到任何消息（包括 pong）都清掉超时定时器
    clearTimeout(this.pongTimer);
    this.pongTimer = null;
  }

  stop() { clearInterval(this.timer); clearTimeout(this.pongTimer); }
}
```

服务端收到 `type=ping` 后应立刻回 `{"type":"pong","ts":原时间戳}`。带回原时间戳可以算 RTT，用于监控而不是业务判断。

> **行业实践**：浏览器在后台标签页会节流定时器，移动端切后台还可能直接冻结页面。心跳间隔不要设得太短，并在 `visibilitychange` 回到前台时立即补发一次心跳，避免"刚回来就判死"。

## 3.2 Java 客户端

Java 客户端同样建议独立心跳线程，并记录连续失败次数：

```java
public class ClientHeartbeat {
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
    private volatile long lastPongAt = System.currentTimeMillis();

    public void start(Session session) {
        scheduler.scheduleAtFixedRate(() -> {
            if (!session.isOpen()) return;
            if (System.currentTimeMillis() - lastPongAt > 90_000) {
                // 超过 3 个心跳周期没有回包，主动关闭触发重连
                closeQuietly(session, "heartbeat timeout");
                return;
            }
            try {
                session.getBasicRemote().sendText("{\"type\":\"ping\"}");
            } catch (Exception e) {
                closeQuietly(session, "send failed");
            }
        }, 0, 30, TimeUnit.SECONDS);
    }

    public void onPong() { lastPongAt = System.currentTimeMillis(); }
    public void stop() { scheduler.shutdownNow(); }

    private void closeQuietly(Session session, String reason) {
        try {
            session.close(new CloseReason(CloseReason.CloseCodes.GOING_AWAY, reason));
        } catch (Exception ignored) {
        }
    }
}
```

**注意**：心跳发送不要和业务发送共用一把锁，否则业务消息大批量发送时心跳会被堵住，误判为超时。心跳应走独立路径或至少独立队列。

# 4. 断线重连

## 4.1 哪些情况该重连

| 触发事件 | 是否重连 | 说明 |
|----------|----------|------|
| 心跳超时 | 是 | 典型半开连接 |
| `1006` 异常关闭 | 是 | 未收到关闭帧，网络中断 |
| `1001` 服务端下线 | 是 | 等退避后重连到新实例 |
| `1011` 服务端内部错误 | 视情况 | 连续出现应告警 |
| `1008` 策略违规 | 否 | 鉴权或权限问题，重连也没用 |
| 认证过期 | 先刷新 token | 刷新成功再重连 |
| `4001` 被新连接顶替 | 否 | 否则会形成互踢死循环 |

**设计考量**：重连判断应基于**关闭码加业务原因**，不要只看 `onclose`。把所有关闭都当可重连，会在鉴权失败时无限刷请求；把所有关闭都当不可重连，又会在服务端滚动发布时掉线不恢复。

## 4.2 指数退避与抖动

固定间隔重连会在服务端故障时形成持续压力，恢复瞬间又会惊群。标准做法是**指数退避 + 抖动 + 上限**：

```text
第 n 次失败等待 = min(base × 2^(n-1), max) × (0.5 + random × 0.5)
```

`base` 取 1 秒保证首次重连快，`max` 取 30-60 秒避免用户等太久，抖动系数取 0.5-1.0 打散同时重连的客户端；连接成功后必须重置计数，否则下次断线会直接等最大值。

## 4.3 浏览器完整示例

```javascript
class ReconnectingSocket {
  constructor(url, onMessage) {
    this.url = url; this.onMessage = onMessage;
    this.attempt = 0; this.closedByUser = false; this.connect();
  }
  connect() {
    this.ws = new WebSocket(this.url);
    this.ws.onopen = () => { this.attempt = 0; this.hb = new Heartbeat(this.ws); this.hb.start(); };
    this.ws.onmessage = e => { this.hb?.onMessage(); this.onMessage?.(e.data); };
    this.ws.onclose = e => {
      this.hb?.stop();
      // 4001 被顶替、1008 策略拒绝，都不应重连
      if (this.closedByUser || e.code === 4001 || e.code === 1008) return;
      const backoff = Math.min(30000, 1000 * 2 ** Math.min(this.attempt++, 5));
      setTimeout(() => this.connect(), backoff * (0.5 + Math.random() * 0.5));
    };
  }
  close() { this.closedByUser = true; this.ws.close(1000, 'user closed'); }
}
```

> **注意**：`onerror` 里不要重复调用 `connect`。浏览器在大多数错误后都会触发 `onclose`，两处都重连会造成连接风暴。

# 5. 会话恢复与消息补偿

## 5.1 恢复握手

重连建立的是全新连接，服务端默认不知道"你就是刚才那个人"。协议里需要跨连接稳定的标识：

```text
首次连接：
  客户端 ── login(token) ──> 服务端
  服务端 ── welcome(sessionId=A1, resumeToken=R1) ──> 客户端

断线重连：
  客户端 ── resume(sessionId=A1, token=R1, lastSeq=1024) ──> 服务端
  服务端 ── resumeOk(fromSeq=1025)，并补发 1025 之后的消息 ──> 客户端
```

| 字段 | 作用 |
|------|------|
| `sessionId` | 标识业务会话，跨多条连接保持不变 |
| `resumeToken` | 防止别人冒用 sessionId |
| `lastSeq` | 客户端最后处理到的序号，用于补发 |

## 5.2 序号与幂等

补发只能做到**至少一次**（at-least-once），客户端必须能处理重复：

- 每条消息带 `msgId` 与递增的 `seq`。
- 客户端按 `msgId` 去重，按 `seq` 检测缺口。
- 处理完成后回 `ack(seq)`，服务端据此清理离线缓存。
- 发现缺口时主动请求 `resume(fromSeq=缺口)`，而不是干等。

服务端把最近消息放进 Redis List 或 Stream，按会话维度保留一段时间。**不要**只放在内存里，否则实例重启后无法补发。

## 5.3 重复连接

移动网络下经常出现"旧连接还没死、新连接已经建好"。不处理会导致同一用户在线两次，消息发到旧连接上：

```java
public void register(String userId, WebSocketSession newSession) {
    WebSocketSession old = byUser.put(userId, newSession);
    if (old != null && old.isOpen() && old != newSession) {
        try {
            // 先发业务层通知，再关闭，避免客户端立刻重连
            old.sendMessage(new TextMessage("{\"type\":\"kicked\"}"));
            old.close(new CloseStatus(4001, "replaced"));
        } catch (Exception ignored) {
        }
    }
}
```

**注意**：被顶替的客户端必须**不再自动重连**。这就是前面把 `4001` 排除在重连之外的原因。如果产品允许多端在线，则改用 `(userId, deviceId)` 作为键，并让消息按设备路由。

# 6. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| 只依赖 TCP keepalive | 断线两小时才发现 | 应用层心跳 30-90 秒 |
| 读空闲等于心跳间隔 | 周期性误杀 | 读空闲取 2.5-3 倍 |
| 浏览器发协议 ping | 无此 API，代码报错 | 发业务心跳 JSON |
| 心跳与业务共用锁 | 忙时心跳发不出 | 独立线程或队列 |
| 所有关闭都重连 | 鉴权失败无限刷 | 按关闭码区分 |
| 重连成功后不重置退避 | 下次断线等很久 | `onopen` 重置计数 |
| 被顶替后继续重连 | 两个客户端互踢 | 识别 4001 并停止重连 |

# 7. 练习与自检

**练习一**：用浏览器建立连接后，在 Chrome DevTools 的 Network 面板切到 Offline，观察 `onclose` 的关闭码，再恢复网络，验证重连逻辑是否按退避执行。

**练习二**：把服务端心跳设为 10 秒、客户端 30 秒，观察会发生什么；再调整为服务端 30 秒、客户端 10 秒，解释差异。

自检问题：浏览器为什么不能主动发 ping？`1006` 和 `1000` 在重连策略上有什么区别？`sessionId` 和 WebSocket 的 `Session.getId()` 是不是一回事？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 心跳 | 协议 ping 保链路，业务心跳保会话 |
| 参数 | 间隔 30-90 秒，读空闲为 2.5-3 倍 |
| 重连 | 按关闭码分类，指数退避 + 抖动 + 上限 |
| 恢复 | sessionId + resumeToken + lastSeq |
| 补偿 | 至少一次投递，客户端按 msgId 幂等 |
| 顶替 | 新连接踢旧连接，旧连接收到 4001 不再重连 |

单机做到这里已经足够，但一旦部署多个实例，消息只会到达连在同一台机器上的用户。下一篇进入集群与会话共享，讲清 Redis Pub/Sub、网关、粘性会话和消息顺序。
