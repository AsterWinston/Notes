WebSocket 一旦建立，连接会长期停留，传统的"每个请求过一遍过滤器"的安全模型不再成立。鉴权要覆盖握手与消息两个阶段，浏览器侧要防跨站 WebSocket 劫持，服务端还要限制连接数、消息大小和发送频率。本篇把这些点串起来，再给出一张生产排错表和整个模块的自检题。

# 1. 鉴权

## 1.1 两个鉴权时机

| 时机 | 能拿到什么 | 适合做什么 |
|------|------------|------------|
| 握手阶段 | HTTP 头、Cookie、查询参数、路径 | 拒绝未认证连接，绑定用户身份 |
| 消息阶段 | STOMP 头、消息体、会话属性 | 校验 token 是否过期、逐条授权 |

只在握手阶段校验是不够的：连接可能持续几小时，token 早已过期；只在消息阶段校验也不行：未认证连接会白白占用资源。正确做法是**握手时粗筛，消息时复核**。

## 1.2 握手阶段

浏览器原生 `WebSocket` 不支持自定义 Header，这是鉴权最麻烦的地方。三种常见方案：

| 方案 | 做法 | 安全性 | 说明 |
|------|------|--------|------|
| Cookie | 依赖浏览器自动携带 | 中 | 必须校验 Origin，防 CSWSH |
| 查询参数 token | `wss://host/ws?token=xxx` | 低 | 会进日志和浏览器历史 |
| 一次性票据 | 先换短期 ticket，握手时消费 | 高 | 推荐，ticket 用后即焚 |

一次性票据的流程：

```text
1. 前端用正常 HTTP 接口换取 ticket
   POST /api/ws-ticket  (Authorization: Bearer ...)
   <-- { "ticket": "t-8f3a...", "expiresIn": 30 }

2. 前端用 ticket 建连
   wss://host/ws?ticket=t-8f3a...

3. 服务端校验并删除 ticket（Redis GETDEL），再放行
```

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;
import java.util.Map;

public class TicketHandshakeInterceptor implements HandshakeInterceptor {

    private final StringRedisTemplate redis;

    public TicketHandshakeInterceptor(StringRedisTemplate redis) {
        this.redis = redis;
    }

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) {
        String ticket = extractQuery(request, "ticket");
        if (ticket == null) {
            response.setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
            return false;
        }
        // GETDEL 保证票据只能使用一次，防止重放
        String userId = redis.opsForValue().getAndDelete("ws:ticket:" + ticket);
        if (userId == null) {
            response.setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
            return false;
        }
        attributes.put("userId", userId);
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception exception) {
    }

    private String extractQuery(ServerHttpRequest request, String name) {
        String query = request.getURI().getQuery();
        if (query == null) return null;
        for (String pair : query.split("&")) {
            String[] kv = pair.split("=", 2);
            if (kv.length == 2 && kv[0].equals(name)) return kv[1];
        }
        return null;
    }
}
```

## 1.3 消息阶段

握手通过后，应把用户身份绑定到会话，并在每条消息上检查授权与过期：

```java
import jakarta.websocket.CloseReason;
import jakarta.websocket.Session;
import java.time.Instant;

public class MessageAuthorizer {

    public boolean check(Session session, IncomingMessage message) {
        String userId = (String) session.getUserProperties().get("userId");
        Instant expiresAt = (Instant) session.getUserProperties().get("tokenExpiresAt");

        if (userId == null || expiresAt == null) {
            close(session, "unauthenticated");
            return false;
        }
        if (Instant.now().isAfter(expiresAt)) {
            // token 过期后不能继续用旧身份收消息
            close(session, "token expired");
            return false;
        }
        // 业务级授权：能不能进这个房间、能不能发这类消息
        return canAccess(userId, message.roomId(), message.type());
    }

    private void close(Session session, String reason) {
        try {
            // 1008 表示策略违规，客户端不应自动重连
            session.close(new CloseReason(
                    CloseReason.CloseCodes.VIOLATION_OF_POLICY, reason));
        } catch (Exception ignored) {
        }
    }

    private boolean canAccess(String userId, String roomId, String type) {
        return true;   // 实际接入权限系统
    }

    public record IncomingMessage(String roomId, String type, String content) {
    }
}
```

> **行业实践**：长连接上不做逐消息远程鉴权（每次都调权限服务），而是握手时把权限快照放进会话，权限变更通过事件通知或短 TTL 刷新。否则每条消息多一次 RPC，延迟和故障面都不可接受。

# 2. 浏览器侧安全

## 2.1 CORS 管不了 WebSocket

很多人以为配了 CORS 就安全了，这是一个误解：

- WebSocket 握手是 `GET`，浏览器会带 `Origin`，但**不走 CORS 预检**。
- 响应里的 `Access-Control-Allow-Origin` 对 WebSocket 连接**没有约束力**。
- 浏览器不会因为跨域就阻止 WebSocket 建连，服务端必须自己校验 `Origin`。

```java
private static final Set<String> ALLOWED_ORIGINS =
        Set.of("https://app.example.com", "https://admin.example.com");

public boolean originAllowed(ServerHttpRequest request) {
    String origin = request.getHeaders().getOrigin();
    // 没有 Origin 的可能是非浏览器客户端，按业务决定是否放行
    return origin != null && ALLOWED_ORIGINS.contains(origin);
}
```

**注意**：不要用 `startsWith("https://app.example.com")` 做校验，`https://app.example.com.evil.com` 也能通过。必须精确匹配，或者解析后比较 scheme、host、port。

## 2.2 跨站 WebSocket 劫持

**CSWSH**（Cross-Site WebSocket Hijacking）是 WebSocket 版的 CSRF：用户登录了 `bank.com`，又打开恶意页面 `evil.com`，恶意脚本直接 `new WebSocket("wss://bank.com/ws")`，浏览器会自动带上 `bank.com` 的 Cookie。同源策略不会阻止 WebSocket 建连，服务端若不校验 `Origin`，就把攻击者当成了合法用户。

三道防线：

| 防线 | 做法 | 说明 |
|------|------|------|
| 校验 Origin | 白名单精确匹配 | 最直接，必须做 |
| 不用自动携带的凭证 | 一次性 ticket 或 Header token | 攻击者拿不到 ticket |
| Cookie 加 SameSite | `SameSite=Strict` 或 `Lax` | 降低跨站自动携带概率，不能单独依赖 |

```java
// 握手时同时校验 Origin 与票据，两者缺一不可
if (!originAllowed(request) || !consumeTicket(request, attributes)) {
    response.setStatusCode(HttpStatus.FORBIDDEN);
    return false;
}
```

## 2.3 输入与输出

WebSocket 消息和 HTTP 参数一样不可信：

- 消息体要做**长度、类型、枚举值**校验，不能直接反序列化后落库。
- 文本消息可能包含 HTML，前端渲染时必须转义，防 XSS。
- 不要把服务端异常堆栈原样发回客户端，避免泄露类名、SQL、内网地址。
- 二进制消息要校验魔数与大小，防止伪装成图片的可执行内容。

**设计考量**：消息协议应有明确版本与 `type` 枚举，未知类型直接拒绝并记录，而不是"尽力解析"。宽松解析是很多逻辑漏洞的源头。

# 3. 限流与资源保护

## 3.1 连接数限流

WebSocket 每个连接都占文件描述符和内存，必须限制：

- 单 IP 并发连接数（防单机压测式攻击）。
- 单用户并发连接数（防同一账号多端刷连接）。
- 全局连接总数（保护服务端容量）。

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import java.time.Duration;

public class ConnectionLimiter {

    private final StringRedisTemplate redis;

    public ConnectionLimiter(StringRedisTemplate redis) {
        this.redis = redis;
    }

    public boolean tryAcquire(String ip, int perIpLimit) {
        String key = "ws:conn:ip:" + ip + ":" + currentMinute();
        // INCR 是原子操作，多实例共享同一个计数
        Long count = redis.opsForValue().increment(key);
        if (count != null && count == 1) {
            redis.expire(key, Duration.ofMinutes(2));
        }
        return count != null && count <= perIpLimit;
    }

    private String currentMinute() {
        return String.valueOf(System.currentTimeMillis() / 60_000);
    }
}
```

> **注意**：`INCR` 加 `EXPIRE` 不是原子操作，极端情况下可能留下永不过期的键。生产环境应使用 Lua 脚本或 Redis 的滑动窗口实现，保证计数与过期一起完成。

## 3.2 消息大小

| 层级 | 配置 | 作用 |
|------|------|------|
| 容器 | `maxTextMessageBufferSize` / `maxBinaryMessageBufferSize` | 单帧/单消息缓冲上限 |
| Spring | `setMessageSizeLimit` | STOMP 入站消息上限 |
| 网关 | `client_max_body_size` | HTTP 握手阶段限制 |
| 应用 | 业务字段长度校验 | 防超长文本与嵌套 JSON |

超过限制应主动关闭，关闭码用 `1009`。不要只依赖容器默认值，不同实现默认值差异很大。

## 3.3 消息频率

按会话做令牌桶，防止单连接刷消息：

```java
public class TokenBucket {
    private final double capacity;
    private final double refillPerSecond;
    private double tokens;
    private long lastRefillNanos;

    public TokenBucket(double capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerSecond = refillPerSecond;
        this.tokens = capacity;
        this.lastRefillNanos = System.nanoTime();
    }

    public synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillNanos) / 1_000_000_000.0;
        tokens = Math.min(capacity, tokens + elapsed * refillPerSecond);
        lastRefillNanos = now;
        if (tokens < 1) {
            return false;   // 超限，调用方应回错误或关闭连接
        }
        tokens -= 1;
        return true;
    }
}
```

超限处理要分层：轻微超限回错误消息；持续超限用 `1008` 或 `1013` 关闭，并记录用户 ID，方便风控。

## 3.4 慢客户端

服务端广播时，如果某个客户端网络差、消费慢，消息会堆在出站缓冲区。没有上限就会拖垮整个 JVM：

- 设置 `sendBufferSizeLimit` 与 `sendTimeLimit`。
- 超限时优先断开慢客户端，而不是无限缓存。
- 对实时性要求高的场景（行情、游戏），旧消息可以直接丢弃，只保留最新状态。

**设计考量**：广播系统的容量由**最慢的消费者**决定。不要为了照顾一个弱网用户，让所有用户一起承担内存增长。断开并用 1013 提示稍后重连，是更负责任的做法。

# 4. 生产排错

## 4.1 症状对照表

| 症状 | 常见原因 | 排查入口 |
|------|----------|----------|
| 握手返回 400 | 缺少 `Upgrade` 头、版本不对 | `curl -i` 看原始请求与响应 |
| 握手返回 403 | Origin 不在白名单、票据无效 | 服务端鉴权日志 |
| 返回 200 而非 101 | 网关未升级、路径被普通路由接管 | Nginx/Ingress 配置 |
| 连上就断 | 鉴权失败、消息超限、被顶替 | 关闭码与 `@OnError` 日志 |
| 收不到广播 | 跨实例未共享、订阅前缀错 | Redis `PUBSUB CHANNELS` |
| 定时断线 | 网关读超时、心跳间隔不匹配 | 代理超时与心跳配置 |
| 内存持续上涨 | 发送缓冲无上限、会话未清理 | 堆 dump、连接数指标 |
| CPU 飙高 | 压缩、JSON 序列化、广播扇出 | 线程栈、火焰图 |
| 文件描述符耗尽 | 连接泄漏、未关闭会话 | `ss -s`、`lsof -p` |

## 4.2 排查顺序

```text
1. 浏览器 DevTools → Network → WS
   ├─ 看握手状态码与响应头
   ├─ 看 Frames 里帧是否收发
   └─ 看 Close 的 code 与 reason

2. 网关
   ├─ 是否透传 Upgrade / Connection
   ├─ 读超时是否大于心跳间隔
   └─ 是否有连接数或请求体限制

3. 服务端
   ├─ 鉴权日志：谁在何时被拒绝
   ├─ 关闭码分布：1000/1001/1006/1008/1009
   └─ 线程栈与堆：是否有阻塞、泄漏

4. 跨实例
   ├─ Redis 频道是否有消息
   ├─ 各节点是否都订阅成功
   └─ 消息序号是否连续
```

常用命令：

```bash
# 手工观察握手响应
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  "https://app.example.com/ws"

# 查看连接与文件描述符
ss -tnp | grep ':443'
ss -s

# 查看 Java 线程与堆
jcmd <pid> Thread.print
jcmd <pid> GC.heap_info

# 查看 Redis 频道订阅情况
redis-cli PUBSUB CHANNELS 'ws:room:*'
```

## 4.3 必须监控的指标

| 指标 | 用途 |
|------|------|
| 当前连接数 / 峰值 | 容量规划与泄漏发现 |
| 新建连接速率 | 重连风暴预警 |
| 鉴权失败率 | 攻击或客户端 bug |
| 消息收发速率 | 容量与异常流量 |
| 发送缓冲使用量 | 慢客户端与内存风险 |
| 关闭码分布 | 区分正常下线与异常 |
| 心跳超时次数 | 网络与中间设备问题 |

> **行业实践**：关闭码分布是最容易被忽视的指标。`1006` 突然升高通常意味着网关或网络问题；`1008` 升高可能是 token 过期策略或攻击；`4001` 升高说明多端登录冲突。把它们做成看板，比事后翻日志快得多。

# 5. 自检题

**问题一**：为什么说"配了 CORS 就安全了"是错的？

答：CORS 约束的是 `fetch`/`XMLHttpRequest` 的跨域读取，WebSocket 握手不经过 CORS 预检，响应头也不限制连接建立。防护必须靠服务端校验 `Origin`、使用不可自动携带的凭证，以及 Cookie 的 `SameSite`。

**问题二**：握手阶段已经校验了 token，为什么消息阶段还要检查？

答：WebSocket 连接可能持续数小时，token 会在连接存活期间过期；权限也可能被回收。握手只证明"建立连接那一刻合法"，消息阶段要保证"当前这条消息仍然有权处理"。

**问题三**：Redis Pub/Sub 能不能当作可靠消息队列？

答：不能。它不持久化，没有订阅者时消息直接丢弃，节点重启期间的消息无法补发，也不保证跨发布者顺序。可靠投递要用 Redis Stream、Kafka 或 RabbitMQ，并配合序号与客户端幂等。

**问题四**：为什么粘性会话不能替代会话共享？

答：粘性会话只保证"尽量落到同一实例"，实例故障、扩缩容、NAT 集中都会打破粘性。会话状态必须外置到 Redis 等共享存储，任意节点都能接管重连，粘性只用于减少补发和缓存预热。

**问题五**：收到 `1006` 应该重连吗？收到 `4001` 呢？

答：`1006` 表示没有收到关闭帧的异常断开，通常应退避重连；`4001` 是应用自定义的"被新连接顶替"，继续重连会形成互踢死循环，必须停止并提示用户。

**问题六**：消息顺序应该用什么保证？

答：单连接内靠 TCP；跨节点靠统一分配的递增序号或按会话分区，客户端按序号重排并检测缺口。不要用时间戳排序，时钟偏差会让顺序不可靠。

# 6. 模块总结

| 篇目 | 核心问题 | 关键结论 |
|------|----------|----------|
| 1 概述与握手 | 为什么需要 WebSocket | HTTP Upgrade 到 101，双向全双工 |
| 2 帧协议 | 数据怎么组织 | 帧头 + opcode + 掩码 + 分片 |
| 3 Java API | 标准 API 怎么用 | `@ServerEndpoint` 每连接一实例，发送要串行 |
| 4 Spring 与 STOMP | 订阅与广播怎么抽象 | 目的地 + 代理，`/app` 入站、`/topic` 出站 |
| 5 心跳与重连 | 连接怎么保持可用 | 双向心跳、指数退避、序号补偿 |
| 6 集群与会话 | 多实例怎么协同 | Redis 广播、状态外置、会话内保序 |
| 7 安全与排错 | 怎么安全上线 | Origin 校验、票据鉴权、限流与指标 |

如果只记住三句话：

- **握手是 HTTP，之后是帧**：排错时先分清问题发生在升级阶段还是数据阶段。
- **连接有状态，会话要外置**：不要让正确性依赖某一台机器。
- **能连上不等于可靠**：心跳、重连、序号、幂等缺一不可。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 鉴权 | 握手粗筛 + 消息复核，优先一次性票据 |
| 浏览器安全 | 校验 Origin，防 CSWSH，Cookie 加 SameSite |
| 资源保护 | 连接数、消息大小、消息频率、慢客户端 |
| 排错 | 先看握手与关闭码，再查网关、服务端、跨实例 |
| 监控 | 连接数、重连率、鉴权失败、缓冲、关闭码分布 |
| 模块主线 | 握手 → 帧 → API → 订阅 → 保活 → 集群 → 安全 |

WebSocket 的学习到这里形成一个闭环：先理解它解决了 HTTP 的什么问题，再往下看帧、API、框架、可靠性、集群与安全。真正落地时，建议从一个最小可运行端点开始，逐层加上心跳、鉴权和广播，而不是一开始就堆齐所有组件。
