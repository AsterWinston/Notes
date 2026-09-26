
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，AMQP 0-9-1 协议，深入讲解 RabbitMQ 每一个核心概念。

---

# 1. 消息模型全景

```
                                  RabbitMQ Broker
┌──────────┐                ┌──────────────────────────┐                ┌──────────┐
│ Producer │────msg────────→│        Exchange          │───msg────────→│ Consumer │
│          │                │   direct/fanout/topic/   │               │          │
│          │                │   headers/default        │               │          │
│          │                └─────────┬──────┬─────────┘               │          │
└──────────┘                          │      │                         └──────────┘
                                   绑定规则  绑定规则
                                      ↓       ↓
                               ┌────────┐  ┌────────┐
                               │ Queue1 │  │ Queue2 │
                               └────────┘  └────────┘
```

**一条消息的生命周期**：

```
Producer 创建消息 → 指定 Exchange + RoutingKey → Broker 接收消息
→ Exchange 根据类型和 BindingKey 匹配 Queue → 消息存入 Queue
→ Consumer 监听 Queue 拉取消息 → 处理完成后发送 ACK → 消息从 Queue 删除
```

---

# 2. Producer（生产者）

生产者是创建并发送消息的应用程序。

```java
// 核心步骤
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
factory.setPort(5672);
factory.setUsername("admin");
factory.setPassword("admin123");
factory.setVirtualHost("/");

try (Connection connection = factory.newConnection();
     Channel channel = connection.createChannel()) {

    // 声明交换机
    channel.exchangeDeclare("order.exchange", BuiltinExchangeType.DIRECT, true);

    // 声明队列
    Map<String, Object> args = new HashMap<>();
    args.put("x-queue-type", "quorum");
    channel.queueDeclare("order.queue", true, false, false, args);

    // 绑定
    channel.queueBind("order.queue", "order.exchange", "order.create");

    // 发送消息
    String message = "Order created: #20240001";
    channel.basicPublish("order.exchange", "order.create",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes(StandardCharsets.UTF_8));
}
```

## 2.1 消息属性（BasicProperties）

| 属性 | 说明 |
|------|------|
| **content-type** | 消息内容类型，如 `application/json`、`text/plain` |
| **content-encoding** | 内容编码方式 |
| **delivery-mode** | 1=非持久化，2=持久化 |
| **priority** | 消息优先级（0~255，越大越高） |
| **correlation-id** | 关联 ID，常用于 RPC 关联请求响应 |
| **reply-to** | 回调队列名，RPC 场景指定回复队列 |
| **expiration** | 消息过期时间（毫秒），过期未被消费则成为死信 |
| **message-id** | 消息唯一标识（业务侧指定） |
| **timestamp** | 消息时间戳 |
| **type** | 消息类型标识 |
| **user-id** | 发布者用户 ID |
| **app-id** | 发布者应用 ID |
| **headers** | 自定义消息头（用于 headers 交换机路由） |

---

# 3. Consumer（消费者）

消费者从队列中获取并处理消息。有两种消费方式：

## 3.1 Push 模式（推荐）

```java
// 注册消费者回调
channel.basicConsume("order.queue", false, "consumer-tag-1",
    (consumerTag, delivery) -> {
        String body = new String(delivery.getBody(), StandardCharsets.UTF_8);
        try {
            processOrder(body);
            // 手动确认（autoAck=false）
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        } catch (Exception e) {
            // NACK 并重回队列（或拒绝）
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);
        }
    },
    consumerTag -> {}  // 消费者取消回调
);
```

## 3.2 Pull 模式（主动拉取）

```java
// 不推荐，性能差
GetResponse response = channel.basicGet("order.queue", false);
if (response != null) {
    String body = new String(response.getBody(), StandardCharsets.UTF_8);
    processOrder(body);
    channel.basicAck(response.getEnvelope().getDeliveryTag(), false);
}
```

## 3.3 确认模式对比

| 模式 | 说明 | 风险 |
|------|------|------|
| **autoAck=true** | Broker 发出消息后立即删除 | **消息可能丢失**（消费者宕机） |
| **autoAck=false** | 消费者显式 ACK 后才删除 | 需手动管理 ACK，可能重复消费 |
| **basicAck** | 确认单条或多条消息（multiple=true） | 安全模式 |
| **basicNack** | 拒绝单条或多条，可选择是否 requeue | 可用作重试 |
| **basicReject** | 拒绝单条消息，可选择是否 requeue | Nack 的单条版本 |

---

# 4. Broker（消息代理）

RabbitMQ Broker 的职责：

```
┌─────────────────────────────────────────────────┐
│                  RabbitMQ Broker                 │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ TCP Acceptor │  │  Erlang VM   │              │
│  │ (ranch/ssl)  │  │  (BEAM)      │              │
│  └──────┬───────┘  └──────┬───────┘              │
│         │                  │                     │
│  ┌──────▼──────────────────▼───────────────┐     │
│  │           AMQP 0-9-1 Engine             │     │
│  │  ┌─────────┐ ┌────────┐ ┌──────────┐   │     │
│  │  │ Exchange │ │ Queue  │ │ Binding  │   │     │
│  │  │ Engine   │ │ Engine │ │ Engine   │   │     │
│  │  └─────────┘ └────────┘ └──────────┘   │     │
│  └──────────────────┬──────────────────────┘     │
│                      │                           │
│  ┌──────────────────▼──────────────────────┐     │
│  │           Storage Layer                 │     │
│  │  ┌──────────┐  ┌──────────────────┐     │     │
│  │  │  Khepri  │  │  Msg Store       │     │     │
│  │  │  (Meta)  │  │  (Segments/Idx)  │     │     │
│  │  └──────────┘  └──────────────────┘     │     │
│  └─────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘
```

---

# 5. Connection（连接）

- Connection 是客户端与 Broker 之间的 **TCP 长连接**。
- 每个 Connection 占用一个 TCP 端口，消耗一定的内存和 CPU。
- 生产环境一个应用通常只创建一个 Connection（或少量）。

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
factory.setPort(5672);
// 设置连接超时
factory.setConnectionTimeout(30000);
// 设置心跳间隔
factory.setRequestedHeartbeat(60);
// 设置自动恢复（网络波动后自动重连）
factory.setAutomaticRecoveryEnabled(true);
// 设置恢复间隔
factory.setNetworkRecoveryInterval(5000);

Connection connection = factory.newConnection();
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| connection-timeout | 60000ms | 连接超时时间 |
| heartbeat | 60s | 心跳间隔，0 表示禁用心跳 |
| automatic-recovery | false | 自动恢复（断线重连） |
| network-recovery-interval | 5000ms | 自动恢复间隔 |

---

# 6. Channel（信道）

Channel 是 Connection 内的**轻量级逻辑连接**，所有 AMQP 操作都通过 Channel 进行。

```
┌───────────────────────────────────────────┐
│              Connection (TCP)             │
│                                           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │ Channel1 │ │ Channel2 │ │ ChannelN │     │
│  │ (生产者)  │ │ (消费者)  │ │ (消费者)  │     │
│  └─────────┘ └─────────┘ └─────────┘     │
└───────────────────────────────────────────┘
```

| 特性 | 说明 |
|------|------|
| **复用 TCP** | 多个 Channel 共享同一个 TCP 连接，减少资源消耗 |
| **隔离性** | 每个 Channel 有独立 ID，互不干扰 |
| **线程不安全** | Channel 实例不能多线程共享（每个线程使用独立 Channel） |
| **连接数上限** | 默认最大 2047 个 Channel/Connection（可配 `channel_max`） |
| **轻量级** | 创建和销毁 Channel 成本极低 |

> **最佳实践**：单 Connection 多 Channel，每个线程持有独立 Channel（ThreadLocal 管理）。不要频繁创建销毁 Channel，也不要多线程共享一个 Channel。

```java
// 线程安全地使用 Channel（ThreadLocal 模式）
public class ChannelPool {
    private final Connection connection;
    private final ThreadLocal<Channel> channelHolder = new ThreadLocal<>();

    public ChannelPool(Connection connection) {
        this.connection = connection;
    }

    public Channel getChannel() throws IOException {
        Channel channel = channelHolder.get();
        if (channel == null || !channel.isOpen()) {
            channel = connection.createChannel();
            channelHolder.set(channel);
        }
        return channel;
    }

    public void close() throws IOException {
        Channel channel = channelHolder.get();
        if (channel != null && channel.isOpen()) {
            channel.close();
        }
        channelHolder.remove();
    }
}
```

---

# 7. Exchange（交换机）

Exchange 是消息路由的入口，根据**类型**和**规则**将消息分发到一个或多个 Queue。

## 7.1 交换机类型对比

| 类型 | 路由规则 | 适用场景 |
|------|----------|----------|
| **Direct** | RoutingKey == BindingKey（完全匹配） | 精确路由，如错误级别分发 |
| **Fanout** | 忽略 RoutingKey，广播到所有绑定队列 | 广播通知，如配置刷新 |
| **Topic** | RoutingKey 与 BindingKey 通配符匹配（`*` 匹配一个词 / `#` 匹配零或多个词） | 灵活的多条件路由，如日志分级 |
| **Headers** | 根据 Header 属性匹配（忽略 RoutingKey） | 复杂的 Header 条件路由 |
| **Default** | 每个 Queue 自动以 QueueName 为 RoutingKey 绑定到默认 Exchange（`""`） | 最简单的点对点 |

## 7.2 交换机属性

| 属性 | 说明 | 默认值 |
|------|------|--------|
| **Name** | 交换机名称 | 必填 |
| **Type** | 交换机类型 | direct |
| **Durability** | 持久化（Durable=true 重启后不丢失） | 取决于声明参数 |
| **Auto-delete** | 当没有绑定队列时自动删除 | false |
| **Internal** | 内部交换机（只接受其他交换机的消息，不接受 Producer 直接发布） | false |
| **Arguments** | 扩展参数（如替代交换机 alternate-exchange） | 空 |

```java
// 声明不同类型的交换机
channel.exchangeDeclare("order.direct", BuiltinExchangeType.DIRECT, true);      // 持久化的 Direct 交换机
channel.exchangeDeclare("order.fanout", BuiltinExchangeType.FANOUT, true);      // 持久化的 Fanout 交换机
channel.exchangeDeclare("order.topic", BuiltinExchangeType.TOPIC, true);        // 持久化的 Topic 交换机
channel.exchangeDeclare("order.headers", BuiltinExchangeType.HEADERS, true);    // 持久化的 Headers 交换机
```

## 7.3 Direct Exchange — 精确路由

```
                      Direct Exchange
                    ┌────────────────┐
   RoutingKey="info"│                │─ BindingKey="info" ──→ [queue_info]
 ──────────────────→│                │─ BindingKey="error" ──→ [queue_error]
   RoutingKey="error"│                │─ BindingKey="warn"  ──→ [queue_warn]
 ──────────────────→│                │
                    └────────────────┘
```

## 7.4 Fanout Exchange — 广播

```
                      Fanout Exchange
                    ┌────────────────┐
   RoutingKey=忽略  │                │────→ [queue1]
 ──────────────────→│                │────→ [queue2]
                    │                │────→ [queue3]
                    └────────────────┘
```

## 7.5 Topic Exchange — 通配符匹配

```
                      Topic Exchange
                    ┌────────────────┐
   "order.create.cn"│                │─ "order.#"      → [order_queue]
 ──────────────────→│                │─ "*.create.*"   → [create_queue]
   "order.update.us"│                │─ "order.*.us"   → [us_order_queue]
 ──────────────────→│                │
                    └────────────────┘
```

| 通配符 | 含义 | 示例 |
|--------|------|------|
| `*` | 匹配**一个**单词 | `order.*.cn` 匹配 `order.create.cn`，不匹配 `order.create.sync.cn` |
| `#` | 匹配**零个或多个**单词 | `order.#` 匹配 `order`、`order.create`、`order.create.cn` |

按 `.` 分割单词，如 `"order.create.cn"` 分割为 `["order", "create", "cn"]`。

## 7.6 Headers Exchange — Header 属性匹配

```java
// 声明
channel.exchangeDeclare("order.headers", BuiltinExchangeType.HEADERS, true);

Map<String, Object> headers = new HashMap<>();
headers.put("x-match", "all");  // all=全部匹配, any=任一匹配
headers.put("type", "order");
headers.put("region", "cn");
channel.queueBind("order.queue", "order.headers", "", headers);

// 发送消息时携带 Header
Map<String, Object> msgHeaders = new HashMap<>();
msgHeaders.put("type", "order");
msgHeaders.put("region", "cn");
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
        .headers(msgHeaders)
        .build();
channel.basicPublish("order.headers", "", props, message.getBytes());
```

---

# 8. Queue（队列）

Queue 是消息的存储容器。RabbitMQ 支持多种队列类型：

## 8.1 队列类型对比

| 类型 | 引入版本 | 数据安全 | 吞吐量 | 适用场景 |
|------|----------|:---:|--------|----------|
| **Classic** | 初始版本 | 低（需镜像有冗余） | 高 | 低价值消息、快速消费、无持久化需求 |
| **Quorum** | 3.8 | **高**（Raft 共识） | 中等 | 关键业务消息、金融/订单、数据不丢失 |
| **Stream** | 3.9 | 高 | **极高** | 大吞吐量流式数据、日志、IoT |
| **Lazy/Classic Lazy** | 3.6 | 低 | 中等 | 大消息堆积、节省内存 |

## 8.2 队列属性

| 属性 | 说明 | 建议 |
|------|------|------|
| **Name** | 队列名称（唯一） | 按业务命名，如 `order.created.queue` |
| **Durable** | 持久化队列（true=重启保留） | 生产环境建议 true |
| **Exclusive** | 独占队列（仅当前 Connection 可用，断开即删除） | 临时队列、RPC 回调队列 |
| **Auto-delete** | 无消费者时自动删除 | 临时队列 |
| **Arguments** | 扩展参数（TTL、DLX、队列类型、最大长度等） | 详见下表 |

## 8.3 关键扩展参数（Arguments）

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `x-message-ttl` | 消息 TTL（毫秒），过期未消费则成为死信 | `60000`（60秒） |
| `x-expires` | 队列 TTL（毫秒），空闲超时自动删除 | `3600000`（1小时） |
| `x-max-length` | 最大消息条数 | `10000` |
| `x-max-length-bytes` | 最大消息总大小（字节） | `104857600`（100MB） |
| `x-max-priority` | 最大优先级（启用优先级队列） | `10` |
| `x-dead-letter-exchange` | 死信交换机 | `"dlx.exchange"` |
| `x-dead-letter-routing-key` | 死信路由键 | `"dlx.order"` |
| `x-queue-type` | 队列类型 | `"quorum"` / `"stream"` |
| `x-queue-mode` | 队列模式（3.6~3.7 用此参数启用 Lazy） | `"lazy"` |
| `x-single-active-consumer` | 单活跃消费者 | `true` |
| `x-overflow` | 溢出策略 | `"drop-head"` / `"reject-publish"` |
| `x-queue-leader-locator` | Quorum 队列 Leader 定位策略 | `"client-local"` / `"balanced"` |

```java
// 声明一个 Quorum 队列
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
args.put("x-dead-letter-exchange", "dlx.exchange");
args.put("x-dead-letter-routing-key", "dlx.order");
args.put("x-max-length", 100000);
channel.queueDeclare("order.queue", true, false, false, args);
```

## 8.4 经典队列演进

```
Classic Queue
  └─→ Mirror Queue（镜像，quorum 出来前的高可用方案，3.13 已不推荐）
  └─→ Quorum Queue（基于 Raft，3.8+ 推荐）
  └─→ Stream Queue（3.9+，日志式流存储，支持重复消费）
```

---

# 9. Binding（绑定）

Binding 是 Exchange 和 Queue 之间的**路由规则连接**。

```
Exchange  ←──Binding──→  Queue
   (路由规则 = BindingKey + Arguments)
```

```java
// 绑定 Queue 到 Exchange，指定 BindingKey
channel.queueBind("order.queue", "order.exchange", "order.create");

// 带参数的绑定（用于 Headers Exchange）
Map<String, Object> bindArgs = new HashMap<>();
bindArgs.put("x-match", "any");
channel.queueBind("order.queue", "order.headers", "", bindArgs);

// Fanout 模式下 BindingKey 无效（可填任意值，但建议写有意义的标识）
channel.queueBind("queue1", "order.fanout", "");
```

---

# 10. Virtual Host（虚拟主机）

VHost 是 RabbitMQ 的**逻辑隔离单元**，类似命名空间或数据库。

```
┌─────────────────────────────────────────┐
│            RabbitMQ Broker              │
│                                         │
│  ┌──── VHost "/" ─────┐ ┌── VHost "/app" ──┐
│  │ exchange_A queue_1 │ │ exchange_B queue_2 │
│  │ user_1: 全部权限   │ │ user_2: 只读权限  │
│  └────────────────────┘ └──────────────────┘
└─────────────────────────────────────────┘
```

| 特性 | 说明 |
|------|------|
| **资源隔离** | 不同 VHost 的 Exchange、Queue 完全独立 |
| **权限隔离** | 用户粒度控制每个 VHost 的 configure/write/read 权限 |
| **默认 VHost** | 系统默认创建 `/`，通常按环境划分（`/dev`、`/test`、`/prod`） |

```bash
# 创建 VHost
rabbitmqctl add_vhost /dev

# 用户授予 VHost 权限
rabbitmqctl set_permissions -p /dev myuser ".*" ".*" ".*"
```

---

# 11. Routing Key 与 Binding Key

| 概念 | 谁定义 | 用途 |
|------|--------|------|
| **Routing Key** | Producer 发布消息时指定 | 告诉 Exchange 如何路由消息 |
| **Binding Key** | Binding 绑定时指定 | Exchange 匹配规则 |

## 11.1 匹配规则

| Exchange 类型 | 匹配方式 |
|---------------|----------|
| Direct | RoutingKey **完全等于** BindingKey |
| Fanout | 忽略 RoutingKey，**无条件广播** |
| Topic | RoutingKey 与 BindingKey 按 `.` 分割后**通配符匹配** |
| Headers | 忽略 RoutingKey，按 **Header 属性** 匹配（`x-match: all/any`） |

---

# 12. 消息流转完整过程（七个步骤）

```
1. Producer 建立与 Broker 的 TCP Connection
2. Producer 在 Connection 内创建 Channel
3. Producer 声明 Exchange（指定名称、类型、持久化等属性）
4. Producer 声明 Queue（指定名称、持久化、类型、参数等属性）
5. Producer 通过 BindingKey 将 Queue 绑定到 Exchange
6. Producer 通过 basicPublish(Exchange, RoutingKey, Properties, Body) 发送消息
7. Exchange 根据类型和 BindingKey 匹配规则，将消息路由到一个或多个 Queue
8. Consumer 通过 basicConsume 从 Queue 拉取消息（或 Broker Push）
9. Consumer 处理消息后发送 basicAck 确认
10. Broker 收到 ACK 后从 Queue 中删除消息
```

> **注意**：步骤 3~5 的顺序可以调整。如果 Exchange/Queue 已存在，Producer 可跳过声明步骤。在生产中通常由应用启动时统一声明，或通过管理界面提前配置。

---

# 13. 架构设计最佳实践

## 13.1 推荐的资源命名规范

```
Exchange:  {domain}.{type}.exchange     例: order.direct.exchange
Queue:     {domain}.{action}.queue       例: order.created.queue
BindingKey:{domain}.{action}             例: order.created
VHost:     /{environment}                例: /prod
```

## 13.2 Channel 与 Connection 管理

```
单例 Connection
  └─ ThreadLocal Channel（每个线程独立Channel）
  └─ 或者连接池 ConnectionFactory（Spring AMQP 已封装 CachingConnectionFactory）
```

## 13.3 声明资源的策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **应用声明** | 应用启动时自动声明 | 开发/测试环境 |
| **手动声明** | 管理员通过 UI/CLI 预先声明 | 生产环境（权限分离） |
| **幂等声明** | 重复声明不报错（相同参数幂等） | Spring AMQP 默认行为 |
