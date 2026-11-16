Jakarta WebSocket（原 Java EE 的 `javax.websocket`）是 Java 平台的标准 WebSocket API，既支持注解式的 `@ServerEndpoint`/`@ClientEndpoint`，也支持编程式的 `Endpoint` 与 `WebSocketContainer`。Spring Boot 3.x 已全部迁移到 `jakarta.*` 命名空间，网上大量 `javax.websocket` 的示例不能直接粘贴使用。本篇给出一套能跑起来的最小服务端与客户端，再讲编码器、解码器、异常和并发发送的坑。

# 1. Jakarta WebSocket 概览

## 1.1 标准 API 与实现

`jakarta.websocket-api` 只定义接口，真正干活的是容器提供的实现：Spring Boot 内嵌 Tomcat 最常用、集成简单；Jetty 轻量，适合独立部署；Undertow 性能好但配置项不同；Netty 不走 Jakarta API，控制力最强。业务系统用内嵌 Tomcat 就够了，需要精细控制背压、内存和线程模型时再考虑 Netty。

## 1.2 依赖

Spring Boot 3.x 项目引入 starter 即可，它会把 WebSocket 实现和 Spring 的封装一起带进来：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

普通 Jakarta EE 项目则引入 `jakarta.websocket-api`，并把 `scope` 设为 `provided`：编译时需要，运行时由容器提供，打进 WAR 反而可能和容器自带版本冲突。

# 2. 服务端 ServerEndpoint

## 2.1 最小可运行服务端

一个能广播消息的端点，核心代码不到 60 行：

```java
import jakarta.websocket.*;
import jakarta.websocket.server.PathParam;
import jakarta.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@ServerEndpoint("/chat/{room}")
public class ChatEndpoint {

    // 每个连接一个 Session，用并发集合保存，便于广播
    private static final Set<Session> SESSIONS = ConcurrentHashMap.newKeySet();

    @OnOpen
    public void onOpen(Session session, @PathParam("room") String room) {
        // 把房间号存进会话属性，关闭时还要用
        session.getUserProperties().put("room", room);
        SESSIONS.add(session);
        broadcast(room, "系统: " + session.getId() + " 加入");
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        String room = (String) session.getUserProperties().get("room");
        // 简单回显并广播，生产环境应做鉴权、限流与内容校验
        broadcast(room, session.getId() + ": " + message);
    }

    @OnClose
    public void onClose(Session session, CloseReason reason) {
        SESSIONS.remove(session);
        String room = (String) session.getUserProperties().get("room");
        broadcast(room, "系统: " + session.getId() + " 离开，原因 " + reason.getCloseCode());
    }

    private static void broadcast(String room, String text) {
        for (Session s : SESSIONS) {
            if (!s.isOpen() || !room.equals(s.getUserProperties().get("room"))) {
                continue;
            }
            try {
                // BasicRemote 是同步发送，多线程广播时需要自己加锁
                synchronized (s) {
                    s.getBasicRemote().sendText(text);
                }
            } catch (IOException e) {
                // 单个连接发送失败不应影响其他连接
                e.printStackTrace();
            }
        }
    }
}
```

## 2.2 生命周期回调

`@ServerEndpoint` 标注的类由容器为**每个连接创建一个实例**，这一点和 Spring 的单例 Bean 完全不同。四个回调的职责如下：

| 注解 | 触发时机 | 常见用途 |
|------|----------|----------|
| `@OnOpen` | 握手成功、Session 建立 | 鉴权后的会话登记、加入房间 |
| `@OnMessage` | 收到一条完整消息 | 业务处理，注意这是消息而非帧 |
| `@OnClose` | 连接关闭 | 清理登记、通知房间 |
| `@OnError` | 处理过程中抛异常 | 记录日志、主动关闭 |

**注意**：`@OnMessage` 一次回调对应一条**完整消息**。协议层可能用多个分片传输，容器已经替你拼好；不要在这里再按 64KB 切分，除非业务自己做了应用层分块。

## 2.3 路径参数与配置

`@ServerEndpoint` 支持路径模板、子协议与编解码器声明：`value = "/chat/{room}"` 定义路径，`@PathParam` 取路径变量，`subprotocols`、`encoders`、`decoders` 分别声明协商协议与消息转换器，`configurator` 则是握手鉴权的钩子，`modifyHandshake` 里可以读 Cookie、Header、查询参数，也能直接拒绝握手。第 7 篇会展开。

## 2.4 在 Spring Boot 中注册

Spring Boot 内嵌容器不会自动扫描 `@ServerEndpoint`，需要显式注册 `ServerEndpointExporter`：

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

@Configuration
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        // 只在以 jar 方式运行时需要；打成 war 交给外部 Tomcat 部署时不要声明
        return new ServerEndpointExporter();
    }
}
```

**设计考量**：`@ServerEndpoint` 实例不是 Spring Bean，`@Autowired` 注入的业务服务会是 `null`。常见解法有三种：把依赖做成静态字段并提供 setter、用 `Configurator` 手动注入、或者干脆改用 Spring 的 `WebSocketHandler`（下一篇）。团队内应统一一种，不要混着写。

# 3. 客户端 ClientEndpoint

## 3.1 注解式客户端

```java
import jakarta.websocket.*;
import java.net.URI;

@ClientEndpoint(
    subprotocols = {"chat.v1"},
    decoders = {ChatMessageDecoder.class}
)
public class ChatClient {

    @OnOpen
    public void onOpen(Session session) {
        System.out.println("已连接: " + session.getId());
    }

    @OnMessage
    public void onMessage(String text) {
        // 收到服务端推送，注意这里在容器 IO 线程上回调
        System.out.println("收到: " + text);
    }

    @OnClose
    public void onClose(Session session, CloseReason reason) {
        System.out.println("连接关闭: " + reason.getCloseCode());
    }

    @OnError
    public void onError(Throwable error) {
        error.printStackTrace();
    }
}
```

连接入口只需要两行：

```java
WebSocketContainer container = ContainerProvider.getWebSocketContainer();
Session session = container.connectToServer(ChatClient.class, URI.create("ws://localhost:8080/chat/42"));
```

`connectToServer` 会阻塞到握手完成，返回的 `Session` 处于 `OPEN` 状态。若不想为每种客户端建类，也可以传匿名 `Endpoint`，在 `onOpen` 里用 `addMessageHandler` 注册处理器；同一类型只能注册一次，重复注册会抛 `IllegalStateException`。

# 4. 编码器与解码器

## 4.1 消息的三种形态

`@OnMessage` 支持的参数形态有三种：`String` 对应文本帧，适合 JSON；`ByteBuffer` 或 `byte[]` 对应二进制帧，适合 Protobuf、图片；任意 POJO 则要配合编解码器。Ping/pong 由容器处理，业务代码一般不需要手写，除非要做应用层心跳。

## 4.2 自定义对象

消息体用 record 定义，字段里带 `type` 便于多态分发：

```java
public record ChatMessage(String type, String from, String content) { }
```

编码器与解码器分别实现 `Encoder.Text` 与 `Decoder.Text`，统一用 Jackson 序列化：

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.websocket.*;

public class ChatMessageEncoder implements Encoder.Text<ChatMessage> {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public String encode(ChatMessage message) throws EncodeException {
        try {
            return MAPPER.writeValueAsString(message);
        } catch (Exception e) {
            // 编码失败抛 EncodeException，容器会走 @OnError
            throw new EncodeException(message, "序列化失败", e);
        }
    }
}

public class ChatMessageDecoder implements Decoder.Text<ChatMessage> {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public ChatMessage decode(String text) throws DecodeException {
        try {
            return MAPPER.readValue(text, ChatMessage.class);
        } catch (Exception e) {
            // 非法 JSON 属于客户端错误，可考虑用 1007 关闭
            throw new DecodeException(text, "JSON 解析失败", e);
        }
    }

    @Override
    public boolean willDecode(String text) {
        // 返回 false 时容器不会调用 decode，而是走错误处理
        return text != null && !text.isBlank();
    }
}
```

把两个类注册到 `@ServerEndpoint` 的 `encoders`、`decoders` 后，`@OnMessage` 可以直接声明 `ChatMessage` 参数，`sendObject` 也能直接发送对象。

> **行业实践**：消息体里一定带 `type` 字段做多态分发。只靠"字段长什么样"去猜类型，版本一升级就会解析错。编解码器内部也应做字段校验，不要信任客户端传来的 JSON。

# 5. 异常与关闭

## 5.1 @OnError 与主动关闭

`@OnError` 捕获的是**消息处理过程中**抛出的异常，包括解码失败、业务代码异常。它不会自动关闭连接，是否关闭要自己决定：

```java
@OnError
public void onError(Session session, Throwable error) {
    log.error("会话 {} 处理失败", session.getId(), error);
    try {
        // 协议级错误用 1002，业务级错误用 1008
        session.close(new CloseReason(
                CloseReason.CloseCodes.PROTOCOL_ERROR, "invalid message"));
    } catch (IOException ignored) {
        // 连接可能已经断了，关闭失败无需再处理
    }
}
```

## 5.2 CloseReason

服务端和客户端都应读取 `CloseReason`，用 `getCloseCode()` 区分 `NORMAL_CLOSURE` 与异常关闭，用 `getReasonPhrase()` 记录原因，而不是只打印一句"连接关闭"。

**注意**：不要在 `@OnClose` 里再调用 `session.close()`，连接已经关闭，重复关闭会抛 `IllegalStateException`。清理逻辑要写得幂等，因为容器可能因网络异常只触发一次 `@OnError` 而不再触发 `@OnClose`。

# 6. 并发发送

`Session.getBasicRemote()` 返回的 `RemoteEndpoint.Basic` **不是线程安全的**。多个线程同时 `sendText`，会出现消息交错甚至 `IllegalStateException`。三种处理方式：

| 方式 | 写法 | 适用 |
|------|------|------|
| 同步加锁 | `synchronized (session) { ... }` | 连接数少、消息量小 |
| 异步发送 | `session.getAsyncRemote().sendText(...)` | 追求吞吐，需处理回调与背压 |
| 发送队列 | 每连接一个单线程队列 | 高并发、要求严格保序 |

> **设计考量**：`getAsyncRemote()` 并不等于"无阻塞"。底层 socket 缓冲区满时，异步发送也会排队占内存。高吞吐场景必须配合队列长度上限和丢弃/降级策略，否则慢客户端会拖垮整个服务端。

# 7. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| 用 `javax.websocket` | Spring Boot 3 编译失败 | 全部改为 `jakarta.websocket` |
| 忘了 `ServerEndpointExporter` | 端点 404 | 内嵌容器下显式注册 |
| 在端点里 `@Autowired` | 注入为 `null` | 用 Configurator 或静态 setter |
| 多线程直接 `sendText` | 消息交错、抛异常 | 加锁、异步或串行队列 |
| 客户端挂多个同类型 handler | `IllegalStateException` | 一条消息内用 type 分流 |
| `@OnClose` 里再 close | `IllegalStateException` | 只做清理，不再关闭 |
| 不设缓冲区上限 | 大消息 OOM | 设置 max buffer 并按需拒绝 |

# 8. 练习与自检

**练习一**：把 2.1 节的广播端点改成"只广播给同一房间"，启动两个浏览器页面验证隔离是否生效。

**练习二**：实现 `ChatMessage` 的编码器与解码器，用 `session.getBasicRemote().sendObject(...)` 发送一条对象消息，观察服务端是否收到反序列化后的对象。

**练习三**：写一个客户端，连续用 10 个线程各发 100 条消息，观察服务端收到的顺序和内容是否完整；再用 `synchronized (session)` 修复并对比。

自检问题：`@ServerEndpoint` 为什么不能直接注入 Spring Bean？`getBasicRemote` 和 `getAsyncRemote` 的本质区别是什么？`@OnError` 之后连接会自动关闭吗？

# 9. 小结

| 主题 | 要点 |
|------|------|
| 命名空间 | Spring Boot 3 使用 `jakarta.websocket` |
| 服务端 | `@ServerEndpoint` 每连接一个实例，需 `ServerEndpointExporter` |
| 客户端 | `@ClientEndpoint` 或编程式 `Endpoint` |
| 编解码 | `Encoder.Text`/`Decoder.Text`，对象消息带 type 字段 |
| 关闭 | 读取 `CloseReason`，清理逻辑要幂等 |
| 并发 | `BasicRemote` 非线程安全，需加锁或串行队列 |

标准 API 适合小而直接的场景，但订阅、广播、鉴权、消息代理这些能力都要自己搭。下一篇进入 Spring 的 WebSocket 与 STOMP，看看框架如何把"目的地"和"订阅"抽象出来。
