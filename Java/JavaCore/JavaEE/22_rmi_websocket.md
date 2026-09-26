
# 1. RMI（Remote Method Invocation）

## 1.1 什么是 RMI

RMI（Java Remote Method Invocation）是 Java 的**远程方法调用**机制，允许运行在不同 JVM 上的对象互相调用方法，是 JavaEE 远程调用的基础设施（EJB `@Remote` 底层基于 RMI/RMI-IIOP）。

```
┌───────────────┐   序列化+网络    ┌───────────────┐
│  客户端 JVM    │ ──────────────► │   服务端 JVM   │
│  Stub（存根）  │ ◄───────────── │  Skeleton     │
└───────────────┘                └──────┬────────┘
                                        │
                                    远程对象实现
```

## 1.2 RMI 架构组件

| 组件 | 作用 |
|------|------|
| `Remote` | 远程接口标记接口（必须继承） |
| `RemoteException` | 远程调用异常（必须抛出） |
| `UnicastRemoteObject` | 远程对象基类 |
| `Registry` | 注册表（RMI 服务注册/查找） |
| `LocateRegistry` | 获取/创建注册表 |
| Stub / Skeleton | 客户端存根 / 服务端骨架 |

## 1.3 定义远程接口

```java
import java.rmi.Remote;
import java.rmi.RemoteException;

public interface HelloService extends Remote {
    String sayHello(String name) throws RemoteException;
}
```

## 1.4 实现远程对象

```java
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class HelloServiceImpl extends UnicastRemoteObject implements HelloService {

    public HelloServiceImpl() throws RemoteException {
        super();
    }

    @Override
    public String sayHello(String name) throws RemoteException {
        return "Hello, " + name;
    }
}
```

## 1.5 服务端注册

```java
public class Server {
    public static void main(String[] args) throws Exception {
        HelloService service = new HelloServiceImpl();

        // 创建注册表（1099 为默认端口）
        Registry registry = LocateRegistry.createRegistry(1099);

        // 绑定服务
        registry.rebind("HelloService", service);

        System.out.println("RMI 服务已启动");
        // 保持运行
    }
}
```

## 1.6 客户端调用

```java
public class Client {
    public static void main(String[] args) throws Exception {
        // 查找注册表
        Registry registry = LocateRegistry.getRegistry("localhost", 1099);

        // 获取远程对象存根
        HelloService service = (HelloService) registry.lookup("HelloService");

        String result = service.sayHello("RMI");
        System.out.println(result);
    }
}
```

## 1.7 RMI 重要机制

| 机制 | 说明 |
|------|------|
| 参数传递 | 基本类型/可序列化对象**按值**传递 |
| 远程对象传递 | 远程引用**按引用**传递（返回存根） |
| 动态类加载 | 客户端可通过 codebase 加载缺少的类 |
| 安全 | 默认受 RMISecurityManager 控制（JDK 17 后废弃） |

## 1.8 RMI-IIOP

- RMI-IIOP 让 RMI 对象可以通过 **IIOP 协议**与 CORBA 互操作。
- EJB `@Remote` 传统上依赖 RMI-IIOP。
- 现代实现（如 WildFly）已转向其他远程协议。

## 1.9 RMI vs RPC vs 现代 RPC

| 对比 | RMI | Java RPC（传统） | Dubbo/gRPC |
|------|-----|------------------|-----------|
| 语言 | Java 专属 | Java | 跨语言（gRPC）/Java 为主（Dubbo） |
| 序列化 | Java 序列化 | XML/二进制 | Protobuf/自定义 |
| 注册中心 | RMI Registry | RMI Registry | Nacos/Zookeeper 等 |
| 生态 | 老旧 | 老旧 | 微服务主流 |

> 现代分布式调用主要用 gRPC、Dubbo、HTTP（REST/JSON），详见 `SpringCloud/Dubbo`、`Interview/42_rpc_dubbo.md`。

# 2. Jakarta WebSocket

## 2.1 什么是 WebSocket

WebSocket 提供浏览器与服务器的**全双工长连接**，实现实时通信（聊天、推送、实时监控）。

```
HTTP（请求-响应，半双工）
浏览器 ──请求──► 服务器
浏览器 ◄─响应── 服务器

WebSocket（长连接，全双工）
浏览器 ⇄ 服务器（双向随时发送）
```

## 2.2 核心注解与 API

| 注解/类 | 说明 |
|---------|------|
| `@ServerEndpoint("/path")` | 服务端端点 |
| `@ClientEndpoint` | 客户端端点 |
| `@OnOpen` | 连接建立时回调 |
| `@OnMessage` | 收到消息时回调 |
| `@OnClose` | 连接关闭时回调 |
| `@OnError` | 异常回调 |
| `Session` | 当前连接会话 |
| `EndpointConfig` | 端点配置 |

## 2.3 服务端示例

```java
@ServerEndpoint("/chat/{room}")
public class ChatEndpoint {

    private static final Map<String, Set<Session>> ROOMS = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(Session session,
                       @PathParam("room") String room) {
        ROOMS.computeIfAbsent(room, k -> ConcurrentHashMap.newKeySet())
             .add(session);
        System.out.println("加入房间：" + room);
    }

    @OnMessage
    public void onMessage(String message, Session session,
                          @PathParam("room") String room) {
        // 广播给同房间的所有会话
        for (Session s : ROOMS.getOrDefault(room, Set.of())) {
            if (s.isOpen()) {
                s.getAsyncRemote().sendText(message);
            }
        }
    }

    @OnClose
    public void onClose(Session session, @PathParam("room") String room) {
        Set<Session> sessions = ROOMS.get(room);
        if (sessions != null) {
            sessions.remove(session);
        }
    }

    @OnError
    public void onError(Session session, Throwable error) {
        error.printStackTrace();
    }
}
```

## 2.4 端点编码器/解码器

```java
// 自定义对象消息
@ServerEndpoint(value = "/user", encoders = UserEncoder.class,
        decoders = UserDecoder.class)
public class UserEndpoint { }

// 编码器：对象 → WebSocket 消息
public class UserEncoder implements Encoder.Text<User> {
    @Override
    public String encode(User user) {
        return user.getName() + "," + user.getAge();
    }
}

// 解码器：WebSocket 消息 → 对象
public class UserDecoder implements Decoder.Text<User> {
    @Override
    public User decode(String s) {
        String[] p = s.split(",");
        return new User(p[0], Integer.parseInt(p[1]));
    }
}
```

## 2.5 客户端示例（Java）

```java
public class Client {

    @ClientEndpoint
    public static class MyClient {
        @OnMessage
        public void onMessage(String msg) {
            System.out.println("收到：" + msg);
        }
    }

    public static void main(String[] args) throws Exception {
        WebSocketContainer container = ContainerProvider.getWebSocketContainer();
        Session session = container.connectToServer(
                MyClient.class, URI.create("ws://localhost:8080/chat/general"));

        session.getBasicRemote().sendText("hello");
        session.close();
    }
}
```

## 2.6 浏览器端

```javascript
const ws = new WebSocket('ws://localhost:8080/chat/general');

ws.onopen = () => ws.send('joined');
ws.onmessage = (e) => console.log('收到：', e.data);
ws.onclose = () => console.log('连接关闭');
```

## 2.7 WebSocket vs SSE vs 轮询

| 方式 | 方向 | 场景 |
|------|------|------|
| HTTP 轮询 | 单向（客户端拉） | 低频、简单 |
| SSE | 服务器 → 客户端单向 | 通知推送（比分、新闻） |
| WebSocket | 全双工 | 聊天、实时协作、游戏 |

## 2.8 WebSocket 实践要点

| 要点 | 说明 |
|------|------|
| 连接管理 | 维护 `Map<Session>` 或房间 Map，及时清理失效会话 |
| 心跳 | 定期发送 Ping 检测连接，超时关闭 |
| 并发安全 | 广播用 `ConcurrentHashMap`，避免并发修改 |
| 消息量控制 | 限制消息大小、频率，防打爆 |
| 负载均衡 | 多实例需共享会话状态（Redis 发布订阅 / 网关转发） |
| 认证 | 建连时校验 Token，拒绝未授权连接 |

# 3. Spring 对照

| Jakarta 概念 | Spring 等价 |
|--------------|-------------|
| RMI | Spring Remoting（RMI 支持） |
| `@ServerEndpoint` | `@ServerEndpoint`（Spring Boot 也支持 WebSocket） |
| `@OnMessage` | `WebSocketHandler` / `@MessageMapping`（STOMP） |
| WebSocket 会话管理 | Spring 封装的 `WebSocketSession` |
| 实时推送 | Spring 可配合 `SimpMessagingTemplate` |
