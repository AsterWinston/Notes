标准 Jakarta WebSocket 只给你"收发消息"的能力，订阅、广播、用户定向、消息代理全要自己实现。Spring 在它之上加了一层**消息模型**：客户端连到端点，向某个**目的地**（destination）发消息，服务端用 `@MessageMapping` 处理，再把结果送到订阅了某个目的地的所有会话。这层抽象依赖 STOMP 协议，Spring Boot 3.x 用一套注解和配置就能跑起来。

# 1. Spring 的 WebSocket 抽象

## 1.1 两条技术路线

Spring 提供两种使用 WebSocket 的方式：**原生 Handler** 以 `WebSocketHandler` 为核心，贴近协议、消息格式自己定义，适合自定义二进制协议或极简场景；**STOMP 消息** 通过 `@EnableWebSocketMessageBroker` 启用，有目的地、订阅和消息代理，适合聊天、通知、协同。原生 Handler 的写法接近第 3 篇的 Jakarta API，只是换成了 Spring 的接口；STOMP 路线额外引入"消息代理"概念，是业务系统更常用的选择。

## 1.2 依赖与启用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

要启用 STOMP 消息代理，在配置类上加 `@EnableWebSocketMessageBroker`：

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // 客户端握手连接的入口，不是订阅地址
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("https://app.example.com");
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // 服务端到客户端的广播前缀
        registry.enableSimpleBroker("/topic", "/queue");
        // 客户端到服务端的应用前缀，交给 @MessageMapping
        registry.setApplicationDestinationPrefixes("/app");
        // 点对点消息的用户前缀
        registry.setUserDestinationPrefix("/user");
    }
}
```

# 2. STOMP 协议

## 2.1 帧格式

**STOMP**（Simple Text Oriented Messaging Protocol）是一个基于文本的简单消息协议，帧长这样：

```text
SEND
destination:/app/chat.send
content-type:application/json

{"from":"alice","content":"hi"}
^@
```

结构分三部分：命令、若干 `key:value` 头、空行后的消息体，最后以一个 `NULL` 字节（`^@`）结束。它和 WebSocket 是**互补**关系：STOMP 定义"消息发给谁、怎么订阅"，WebSocket 负责"字节怎么传"。

## 2.2 常用命令

| 命令 | 方向 | 作用 |
|------|------|------|
| `CONNECT` | 客户端 → 服务端 | 建立会话，可携带认证头 |
| `CONNECTED` | 服务端 → 客户端 | 握手确认，返回会话信息 |
| `SEND` | 客户端 → 服务端 | 发送一条消息到某目的地 |
| `SUBSCRIBE` | 客户端 → 服务端 | 订阅某目的地 |
| `MESSAGE` | 服务端 → 客户端 | 推送订阅的消息 |
| `ACK`/`NACK` | 客户端 → 服务端 | 确认或拒绝消费 |
| `ERROR` | 服务端 → 客户端 | 错误通知 |

## 2.3 与原生 WebSocket 的关系

```text
浏览器                    Spring 服务端
  │                        │
  │ ── HTTP Upgrade ─────> │  /ws 端点
  │ <─ 101 Switching ───── │
  │                        │
  │ ── STOMP CONNECT ────> │  ChannelInterceptor 鉴权
  │ <─ STOMP CONNECTED ─── │
  │ ── SUBSCRIBE /topic/x > │  注册订阅
  │ ── SEND /app/chat ───> │  @MessageMapping
  │ <─ MESSAGE /topic/x ── │  SimpMessagingTemplate / broker
```

**设计考量**：STOMP 带来的最大好处是**解耦**。发送方只管把消息发到 `/app/chat.send`，不需要知道谁在听；接收方只管订阅 `/topic/room.42`，不需要知道消息从哪来。代价是多了一层协议和代理，排查链路更长。

# 3. 配置

## 3.1 端点注册

`addEndpoint("/ws")` 注册的是**握手入口**。客户端先连这个地址，再在 WebSocket 之上发 STOMP 帧。常见配置：

```java
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("https://app.example.com")
            .setSubProtocols("v12.stomp")
            .withSockJS();
}
```

> **行业实践**：现代浏览器都原生支持 WebSocket，新项目直接用原生连接即可。SockJS 是一层兼容垫片，会让消息格式和排错都变复杂，只在必须支持老环境时使用。

## 3.2 消息代理

Spring 提供两种代理：

| 代理 | 配置 | 特点 |
|------|------|------|
| 简单代理 | `enableSimpleBroker("/topic")` | 进程内内存实现，无需外部依赖，不能跨实例 |
| 外部代理 | `enableStompBrokerRelay("/topic")` | 依赖 RabbitMQ 等，支持持久化与集群 |

简单代理的广播只在本 JVM 内有效。部署两个实例、用户分别连到不同实例时，A 实例发出的消息到不了 B 实例的订阅者。第 6 篇会专门讲集群方案。

## 3.3 传输参数

消息大小、发送缓冲、发送超时都要显式配置，默认值往往偏小或无限：

```java
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration registry) {
    registry.setMessageSizeLimit(64 * 1024)        // 单条入站消息上限
            .setSendBufferSizeLimit(512 * 1024)    // 单会话出站缓冲上限
            .setSendTimeLimit(10_000)              // 出站发送超时，毫秒
            .setTimeToFirstMessage(30_000);        // 握手后首条消息超时
}
```

`sendBufferSizeLimit` 是保护服务端内存的关键。慢客户端消费不过来时，消息会堆在出站缓冲区，没有上限就会把堆撑爆。

## 3.4 外部代理

需要持久化、集群或复杂路由时，可以接 RabbitMQ：

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableStompBrokerRelay("/topic", "/queue")
            .setRelayHost("rabbitmq.internal")
            .setRelayPort(61613)
            .setClientLogin("stomp-user")
            .setClientPasscode("stomp-pass");
    registry.setApplicationDestinationPrefixes("/app");
}
```

RabbitMQ 需要启用 `rabbitmq_stomp` 插件，端口通常是 61613。此时 Spring 不再自己维护订阅表，而是把 STOMP 帧转发给 broker，由 broker 负责路由与持久化。

# 4. 订阅与广播

## 4.1 @MessageMapping

```java
@Controller
public class ChatController {
    // 客户端 SEND 到 /app/chat.send.42
    @MessageMapping("/chat.send.{roomId}")
    @SendTo("/topic/room.{roomId}")
    public ChatMessage send(@DestinationVariable Long roomId, ChatMessage message) {
        // 返回值会被自动序列化，并广播到 @SendTo 指定的目的地
        return new ChatMessage(message.from(), message.content(), roomId);
    }
}
```
- `@MessageMapping` 的路径是**应用前缀之后**的部分，客户端实际发到 `/app/chat.send.42`。
- `@DestinationVariable` 取目的地中的路径变量。
- `@SendTo` 指定返回值发往哪里；不写则只把返回值发给当前会话。
- 参数和返回值默认用 Jackson 序列化，消息头 `content-type` 要匹配。

## 4.2 服务端主动推送

业务代码不在消息处理器里时，注入 `SimpMessagingTemplate` 主动发送：

```java
@Service
public class PricePushService {
    private final SimpMessagingTemplate messaging;
    public PricePushService(SimpMessagingTemplate messaging) {
        this.messaging = messaging;
    }

    public void pushPrice(String symbol, double price) {
        // 广播给所有订阅该主题的会话
        messaging.convertAndSend("/topic/price." + symbol, new Price(symbol, price));
    }
}
```

## 4.3 点对点

用户私有消息走 `/user` 前缀：

```java
// 第二个参数是用户标识，Spring 会解析成该用户的所有会话
messaging.convertAndSendToUser("alice", "/queue/notifications", notification);
```

客户端订阅 `/user/queue/notifications` 即可收到。服务端会把 `/user/{userId}/queue/notifications` 映射到实际会话，用户标识来自 `Principal`，所以**必须先完成鉴权并设置 Principal**，否则点对点消息找不到收件人。

# 5. 鉴权

## 5.1 握手拦截

握手阶段可以做第一道校验。实现 `HandshakeInterceptor` 的 `beforeHandshake`，从查询串或 Cookie 里取 token，校验失败就设置 `401` 并返回 `false`，成功则把 `userId` 放进 `attributes`，后续会话可以直接读取。`afterHandshake` 一般只做日志，不需要额外处理。

> **注意**：原生 WebSocket 不能自定义 Header，所以握手阶段只能从查询串、Cookie 或子协议里取凭证。查询串会进日志，优先用短期一次性票据。

## 5.2 CONNECT 拦截

更推荐的做法是在 STOMP `CONNECT` 帧上鉴权，因为此时可以读取原生头，也能把 `Principal` 绑定到会话：

```java
public class AuthChannelInterceptor implements ChannelInterceptor {
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        // 只对 CONNECT 做凭证校验，后续帧复用已绑定的用户
        if (accessor.getCommand() == StompCommand.CONNECT) {
            String token = accessor.getFirstNativeHeader("Authorization");
            if (token == null || !token.startsWith("Bearer ")) {
                throw new IllegalArgumentException("missing token");
            }
            String userId = TokenVerifier.verify(token.substring(7));
            // 设置用户后，/user 目的地才能正确路由
            accessor.setUser(() -> userId);
        }
        return message;
    }
}
```

注册拦截器只需重写 `configureClientInboundChannel`，调用 `registration.interceptors(new AuthChannelInterceptor())`。

**注意**：异常会让 STOMP 返回 `ERROR` 帧并关闭连接，客户端需要处理这种情况。不要在校验失败时静默返回 `message`，否则未认证用户会继续留在连接上。

## 5.3 前端携带 token

浏览器原生 `WebSocket` 构造函数不能设置自定义 Header，所以 token 通常放在查询串或 STOMP `CONNECT` 头里。用 `@stomp/stompjs` 时：

```javascript
const client = new Client({
  brokerURL: 'wss://app.example.com/ws',   // 生产环境只用 wss
  reconnectDelay: 5000,                    // 配合第 5 篇的退避策略
  connectHeaders: { Authorization: 'Bearer ' + localStorage.getItem('access_token') }
});

client.onConnect = () => {
  client.subscribe('/topic/room.42', frame => console.log('收到', JSON.parse(frame.body)));
  client.publish({ destination: '/app/chat.send.42', body: JSON.stringify({ from: 'alice', content: 'hi' }) });
};

client.activate();
```

> **注意**：把 token 放在 URL 查询串会进入网关日志和浏览器历史，安全性较差。优先用 STOMP `CONNECT` 头；如果必须走查询串，至少使用短期一次性票据，并在握手后立即作废。

# 6. 常见配置与坑

| 问题 | 症状 | 正确做法 |
|------|------|----------|
| 订阅了但收不到 | 目的地前缀不匹配 | `@SendTo` 必须以代理前缀开头 |
| `/app` 消息 404 | 忘记应用前缀 | 客户端发到 `/app/xxx` |
| 点对点收不到 | Principal 为空 | 在 CONNECT 帧鉴权并 `setUser` |
| 跨域失败 | `setAllowedOrigins("*")` 被拒 | 用 `setAllowedOriginPatterns` 明确来源 |
| 大消息被断 | 默认消息上限太小 | 调整 `setMessageSizeLimit` 与缓冲区 |
| 多实例消息丢失 | 简单代理只在单 JVM 内 | 外部 broker 或 Redis 广播 |
| 慢客户端拖垮服务 | 出站缓冲无上限 | 设置 `sendBufferSizeLimit` |

# 7. 练习与自检

**练习一**：搭一个 `/ws` 端点，用浏览器控制台连接，订阅 `/topic/demo`，再写一个 `@Scheduled` 每秒向该主题推送时间戳，验证能收到。

**练习二**：把 4.1 节的 `@SendTo` 改成用 `SimpMessagingTemplate` 手动发送，观察两种写法在返回值和异常处理上的差异。

**练习三**：在 `CONNECT` 拦截器里故意抛异常，用浏览器观察收到的 `ERROR` 帧内容，并写出客户端应如何处理。

自检问题：STOMP 的 `SEND` 和 `SUBSCRIBE` 有什么区别？为什么 `/user` 目的地依赖 `Principal`？简单代理为什么不能跨实例？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 启用 | `@EnableWebSocketMessageBroker` + 配置类 |
| 端点 | `/ws` 是握手入口，不是订阅地址 |
| 前缀 | `/app` 入站，`/topic`、`/queue` 出站，`/user` 点对点 |
| 广播 | `@SendTo` 或 `SimpMessagingTemplate` |
| 鉴权 | 握手拦截做粗筛，CONNECT 拦截绑定 Principal |
| 代理 | 简单代理单机，外部 broker 支持集群与持久化 |

STOMP 解决了"消息发给谁"的问题，但连接总会断。下一篇专门处理心跳、空闲检测、断线重连与消息补偿，把"连接可用"变成"业务连续"。
