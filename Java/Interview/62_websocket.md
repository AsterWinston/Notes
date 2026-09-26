# 1. 知识点
- **WebSocket概述**: 基于TCP的全双工通信协议，客户端和服务器之间建立持久连接，双向实时推送数据。由HTML5标准定义
- **WebSocket vs HTTP**: HTTP是请求-响应模式(客户端主动)，WebSocket是全双工(双向主动推送)。WebSocket首次通过HTTP升级建立连接
- **协议升级(Upgrade)**: 客户端HTTP请求携带`Connection: Upgrade`和`Upgrade: websocket`头请求协议切换。服务器同意返回101 Switching Protocols状态码
- **握手请求头**: 客户端发送Sec-WebSocket-Key(随机16字节Base64编码)、Sec-WebSocket-Version(版本号通常是13)
- **握手响应**: 服务端用固定的Magic String("258EAFA5-E914-47DA-95CA-C5AB0DC85B11")拼接客户端的Sec-WebSocket-Key，计算SHA-1再Base64得到Sec-WebSocket-Accept返回
- **握手目的**: 防止误连接(保证对方确实是WebSocket服务器而非误转发)、防止缓存代理返回错误的缓存响应
- **数据帧格式**: FIN(1bit，是否为最后一帧)、RSV(3bit保留)、Opcode(4bit，帧类型)、MASK(1bit，客户端→服务器需掩码)、Payload Length(7/16/64bit，数据长度)、Masking Key(4字节，掩码密钥)、Payload Data
- **Opcode帧类型**: 0x0(后续帧)、0x1(文本帧)、0x2(二进制帧)、0x8(关闭连接)、0x9(Ping心跳)、0xA(Pong心跳响应)
- **客户端掩码(MASK)**: 客户端发送到服务器的数据必须做掩码处理(防止缓存投毒攻击Proxy Cache Poisoning)。Masking Key为4字节随机数，Payload每个字节与Masking Key对应字节XOR
- **Payload Length编码**: 如果length<=125，直接7bit表示。如果length=126，后续2字节(16bit)表示实际长度。如果length=127，后续8字节(64bit)表示实际长度。即支持超大消息(最大2^64-1)
- **心跳保活(Ping/Pong)**: 服务端或客户端可以发送Ping帧(Opcode=0x9)，对端必须回复Pong帧(Opcode=0xA)。用于检测连接是否存活和保持代理不超时
- **Netty IdleStateHandler**: `pipeline.addLast(new IdleStateHandler(0, 0, 30, TimeUnit.SECONDS))`检测读写空闲。读空闲超过30秒触发userEventTriggered，可发送Ping帧或断开连接
- **Netty WebSocket支持**: WebSocketServerProtocolHandler处理握手和帧解析、WebSocketFrame编解码、自动处理Ping/Pong
- **轮询(Polling)**: 客户端定时(如每2秒)发送HTTP请求查询是否有新消息。缺点：浪费带宽和服务器资源、实时性不高
- **长轮询(Long Polling / Comet)**: 客户端发送HTTP请求，服务器保持连接不立即返回，有新消息时响应。客户端处理后立即再次发起请求。比轮询实时性好但仍然HTTP开销大
- **SSE(Server-Sent Events)**: 服务器向客户端单向推送的HTTP长连接，客户端通过EventSource API接收。格式为`text/event-stream`，简单文本流。缺点：单向(仅服务器→客户端)、不支持二进制
- **WebSocket优势**: 全双工(双向实时)、二进制支持(文本/二进制帧)、开销小(建立后无HTTP头)、延迟低(持续连接)
- **WebSocket劣势**: 需要特殊协议(代理/防火墙可能不支持)、需要保持连接(服务器资源)、重连机制需自行实现
- **Spring WebSocket**: 实现WebSocketHandler接口(TextWebSocketHandler/BinaryWebSocketHandler)，注册到WebSocketConfigurer
- **HandshakeInterceptor**: 握手拦截器，在握手前后执行。可用于从HTTP Session获取用户信息放入WebSocketSession attributes(用于权限控制)
- **SockJS**: Spring提供的浏览器降级方案。当浏览器不支持WebSocket时自动降级为轮询/长轮询等方案。客户端使用SockJS JavaScript库，服务端使用withSockJS()启用
- **STOMP(Simple Text Oriented Messaging Protocol)**: 基于帧的简单文本消息协议，建立在WebSocket之上。提供发布-订阅模型，类似MQ的功能
- **STOMP帧格式**: COMMAND + headers + 空行 + Body + NULL字符。常用命令：CONNECT、SEND、SUBSCRIBE、UNSUBSCRIBE、DISCONNECT、MESSAGE、ERROR
- **Spring STOMP**: @EnableWebSocketMessageBroker开启消息代理。SimpleBroker(内存消息代理)或外部代理(RabbitMQ/ActiveMQ)转发消息
- **@MessageMapping**: 类似@RequestMapping，处理STOMP客户端发送到/app/destination的消息
- **@SendTo**: 方法返回值发送到指定目的地(broker的/topic/xxx)，订阅者收到
- **用户目标消息(User Destinations)**: `/user/queue/notifications`，ConvertAndSendToUser发送给特定用户，利用simpUserRegistry管理
- **Nginx代理WebSocket**: 需要配置Upgrade和Connection头传递：
```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;  # 长连接超时
}
```
- **负载均衡**: 使用IP Hash或用户ID一致性Hash保证同一用户WebSocket连接到同一服务器(有状态)
- **适用场景**: 在线聊天(IM)、实时推送通知、协作编辑(如文档协同)、实时监控大盘、游戏实时交互、实时交易行情

# 2. 面试题

## 2.1 WebSocket的握手过程是怎样的？
WebSocket握手通过HTTP升级完成：
1) 客户端发送HTTP请求：
```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```
2) 服务端解析Sec-WebSocket-Key，将其与Magic String("258EAFA5-E914-47DA-95CA-C5AB0DC85B11")拼接
3) 对拼接字符串计算SHA-1，得到20字节摘要
4) 对摘要进行Base64编码得到Sec-WebSocket-Accept
5) 服务端返回101响应：
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
之后TCP连接升级为WebSocket协议，双方可以发送WebSocket帧进行通信。

## 2.2 WebSocket和HTTP轮询、长轮询、SSE有什么区别？
| 维度 | 短轮询 | 长轮询 | SSE | WebSocket |
|------|--------|--------|-----|-----------|
| 通信方向 | 客户端→服务器 | 客户端→服务器 | 服务器→客户端 | 双向 |
| 连接类型 | 短连接(每次新建) | 保持/新建 | 长连接 | 长连接 |
| 实时性 | 低(取决于轮询间隔) | 中(有延迟) | 高(实时推送) | 高(实时双向) |
| 带宽开销 | 高(每次HTTP头) | 中 | 低(文本流) | 低(帧头极小) |
| 二进制支持 | 可以(但开销大) | 可以 | 不支持(仅文本) | 支持 |
| 兼容性 | 全兼容 | 全兼容 | 高(IE不支持) | 高(IE10+) |

WebSocket适合需要双向实时交互的场景(IM、游戏)；SSE适合只有服务器推送的场景(股价、新闻)；长轮询适合不支持WebSocket的老环境降级。

## 2.3 WebSocket的数据帧结构是怎样的？
WebSocket帧格式(按bit排列)：
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|     Extended payload length continued, if payload len=126/127 |
+-------------------------------+-------------------------------+
|                               |  Masking-key (if MASK set)    |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------+-------------------------------+
```
- FIN(1bit)：1表示最后一帧(消息结束)
- Opcode(4bit)：0x1文本、0x2二进制、0x8关闭、0x9 Ping、0xA Pong
- MASK(1bit)：1表示Payload被掩码(客户端必须mask)
- Payload Len(7bit)：<=125直接长度；126则后2字节表示长度；127则后8字节表示长度

## 2.4 为什么客户端发送WebSocket帧需要掩码(MASK)？
掩码是为了防止缓存投毒攻击(Cache Poisoning)。某些中间代理/缓存服务器会将客户端发送的WebSocket帧误认为是HTTP请求，代理响应可能被缓存。攻击者可以构造恶意WebSocket帧使其看起来像HTTP请求，污染公共CDN/代理缓存的响应。掩码使用随机4字节Masking Key与Payload XOR，确保客户端每次发送的数据看起来随机，消除可预测性，避免缓存投毒。RFC 6455规范强制要求客户端到服务器的帧必须掩码，服务器到客户端禁止掩码。

## 2.5 如何实现WebSocket心跳保活机制？
WebSocket协议内置Ping/Pong帧用于心跳：
- 服务端定期(如30秒)发送Ping帧(Opcode=0x9)
- 客户端收到Ping必须立即回复Pong帧(Opcode=0xA)，可带相同Payload
- 服务端超时未收到Pong判定连接断开，主动关闭

Netty实现：
```java
pipeline.addLast(new IdleStateHandler(60, 30, 0));
// 当reader_idle=60秒触发，或writer_idle=30秒时发送PingFrame
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent) {
        IdleState state = ((IdleStateEvent) evt).state();
        if (state == IdleState.READER_IDLE) {
            ctx.close();  // 读超时太久，主动关闭
        } else if (state == IdleState.WRITER_IDLE) {
            ctx.writeAndFlush(new PingWebSocketFrame());  // 定期发Ping
        }
    }
}
```
客户端侧同样需要心跳检测，收不到服务端Ping/Pong时主动重连。

## 2.6 Spring中如何集成WebSocket和STOMP？
STOMP提供发布-订阅模型，简化WebSocket消息路由：
1) 配置：
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue"); // 客户端订阅的broker前缀
        registry.setApplicationDestinationPrefixes("/app"); // 客户端发送到服务端的前缀
        registry.setUserDestinationPrefix("/user"); // 用户私有消息前缀
    }
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS(); // 端点+降级
    }
}
```
2) Controller：
```java
@MessageMapping("/chat.send")
@SendTo("/topic/public")
public ChatMessage sendMessage(ChatMessage message) {
    return message;
}
```
3) 前端stomp.js：
```javascript
let stompClient = Stomp.over(new SockJS('/ws'));
stompClient.connect({}, () => {
    stompClient.subscribe('/topic/public', message => { ... });
    stompClient.send('/app/chat.send', {}, JSON.stringify(msg));
});
```

## 2.7 WebSocket如何处理大规模的连接数(百万级)？
1) **集群部署+负载均衡**：多台服务器水平扩展WebSocket服务，Nginx/LVS负载均衡
2) **连接绑定(Sticky Session)**：同一用户的WebSocket始终路由到同一台服务器(IP Hash/用户ID Hash)，避免广播消息到所有服务器
3) **消息广播**: 使用Redis Pub/Sub或MQ(Topic/Kafka)在服务器间广播消息，只有持有目标用户连接的服务器才推送
4) **Netty优化**: Netty使用NIO+Epoll(linux)，单机支持10万+长连接。调整系统参数(fd限制、tcp参数)
5) **连接编码**: 链接信息使用轻量协议，连接元数据存Redis(而非服务内存)，服务无状态更易水平扩展
6) **消息队列异步**: 推送通知异步入MQ，在消费者线程处理，非IO线程阻塞
7) **限流保护**: 同一用户连接数限制、消息频率限制
