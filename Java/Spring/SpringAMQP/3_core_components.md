# 1. 核心组件总览

Spring AMQP的核心组件分为以下几个层次：

```
| 组件             | 职责                         | 对应AMQP概念        |
|-----------------|-----------------------------|--------------------|
| ConnectionFactory | 创建和管理Connection         | Connection         |
| RabbitAdmin      | 声明Exchange/Queue/Binding  | AMQP管理命令        |
| RabbitTemplate   | 发送/接收消息的模板           | Publisher/Consumer |
| MessageListenerContainer | 管理消费者线程，分发消息 | Consumer           |
| MessageConverter | 消息体与Java对象的转换         | 消息序列化/反序列化  |
| Message          | 封装AMQP消息                 | AMQP Message       |
```

# 2. ConnectionFactory详解

## 2.1 什么是ConnectionFactory
`ConnectionFactory`是Spring AMQP中最底层的组件，负责创建到RabbitMQ Broker的TCP连接。它实现了`javax.net.SocketFactory`的设计理念，封装了连接的创建过程。

Spring AMQP提供了`CachingConnectionFactory`作为默认实现，底层使用RabbitMQ官方客户端库`com.rabbitmq.client.ConnectionFactory`创建实际的AMQP连接。

## 2.2 CachingConnectionFactory的缓存机制
这是`CachingConnectionFactory`最核心的设计——**连接复用与信道缓存**。

### 2.2.1 缓存模式：Channel模式（默认）
在Channel模式下，`CachingConnectionFactory`**维护一个共享的TCP Connection**，并在其上缓存多个Channel：

```
CachingConnectionFactory
  └── 共享 Connection（TCP连接）
        ├── Channel 1（被线程A使用）
        ├── Channel 2（空闲，缓存中）
        ├── Channel 3（被线程B使用）
        └── ...
```

工作流程：
1. 线程调用`createChannel()`请求一个Channel
2. 如果缓存中有空闲Channel，直接返回
3. 如果缓存中没有空闲Channel，且Channel总数未超上限，则创建新Channel
4. 如果Channel总数已达上限，线程阻塞等待，直到有Channel归还
5. Channel使用完毕后，调用`close()`（实际是归还缓存，而非真正关闭）

```java
// Channel缓存大小配置
factory.setChannelCacheSize(25); // 默认25

// 获取Channel的超时时间
factory.setChannelCheckoutTimeout(10000); // 默认10秒

// 获取Channel的源码示例（简化的逻辑）
Channel getChannel() {
    Channel channel = channelCache.poll();  // 从缓存中取
    if (channel != null) {
        return channel;  // 命中缓存，直接返回
    }
    if (channelCount < channelCacheSize) {
        channel = createNewChannel();  // 未达上限，创建新Channel
        channelCount++;
        return channel;
    }
    // 达到上限，阻塞等待
    return waitForChannel();
}
```

### 2.2.2 缓存模式：Connection模式
在Connection模式下，**每个请求都获取一个独立的Connection**（每个Connection只有一个Channel）：

```java
factory.setCacheMode(CacheMode.CONNECTION);
factory.setConnectionCacheSize(5); // 缓存5个Connection
```

适用场景：
- 需要连接不同的Virtual Host
- 需要使用不同的SSL证书
- 需要严格的连接隔离

## 2.3 ConnectionFactory的创建与配置

### 2.3.1 使用Spring Boot自动配置（最常见）
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
```

Spring Boot会自动创建`CachingConnectionFactory` Bean，并使用上述配置初始化。

### 2.3.2 手动创建CachingConnectionFactory
```java
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("guest");
        factory.setPassword("guest");
        factory.setVirtualHost("/");
        
        // 重要配置项
        factory.setCacheMode(CachingConnectionFactory.CacheMode.CHANNEL);
        factory.setChannelCacheSize(25);
        factory.setConnectionTimeout(30000);
        factory.setRequestedHeartBeat(60);  // 心跳间隔，单位秒
        
        return factory;
    }
}
```

### 2.3.3 连接RabbitMQ集群
```java
@Bean
public ConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    // 集群地址，用逗号分隔
    factory.setAddresses("192.168.1.100:5672,192.168.1.101:5672,192.168.1.102:5672");
    factory.setUsername("admin");
    factory.setPassword("admin");
    
    // 连接恢复间隔
    factory.setConnectionRecoveryInterval(10000); // 10秒
    
    return factory;
}
```

### 2.3.4 使用URI创建
```java
@Bean
public ConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setUri("amqp://guest:guest@localhost:5672/%2F");
    // URI格式: amqp://用户名:密码@主机:端口/虚拟主机(需URL编码)
    return factory;
}
```

## 2.4 ConnectionListener监听连接事件
可以通过`ConnectionListener`监听连接的建立、断开等事件：

```java
@Component
public class MyConnectionListener implements ConnectionListener {
    
    @Override
    public void onCreate(Connection connection) {
        System.out.println("新连接创建: " + connection);
    }
    
    @Override
    public void onClose(Connection connection) {
        System.out.println("连接关闭: " + connection);
    }
}

// 注册到ConnectionFactory
@Bean
public ConnectionFactory connectionFactory(MyConnectionListener listener) {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    // ...基本配置...
    factory.addConnectionListener(listener);
    return factory;
}
```

## 2.5 ChannelListener监听信道事件
```java
@Component
public class MyChannelListener implements ChannelListener {
    
    @Override
    public void onCreate(Channel channel, boolean transactional) {
        System.out.println("新Channel创建: " + channel.getChannelNumber());
    }
}

// 注册
factory.addChannelListener(new MyChannelListener());
```

# 3. RabbitAdmin详解

## 3.1 什么是RabbitAdmin
`RabbitAdmin`是Spring AMQP中用于**管理AMQP实体（Exchange、Queue、Binding）**的核心组件。它可以：

- 自动在Broker上声明（declare）Spring容器中定义的Exchange、Queue、Binding Bean
- 编程式地创建、删除、清空Exchange、Queue、Binding
- 查询Queue中的消息数量

## 3.2 RabbitAdmin的工作原理
当Spring容器刷新（refresh）时，RabbitAdmin会执行以下操作：

```
1. 从容器中收集所有 Exchange 类型的 Bean
2. 从容器中收集所有 Queue 类型的 Bean
3. 从容器中收集所有 Binding 类型的 Bean
4. 获取一个 Channel
5. 逐个声明 Exchange（调用 channel.exchangeDeclare()）
6. 逐个声明 Queue（调用 channel.queueDeclare()）
7. 逐个声明 Binding（调用 channel.queueBind()）
8. 归还 Channel
```

关键点：
- 声明顺序：**先Exchange，再Queue，最后Binding**
- 如果Broker上已存在同名组件，RabbitAdmin会比较属性是否匹配
- 属性不匹配时，默认行为是**抛出`IOException`**（Broker端的限制，不允许修改已存在组件的属性）

## 3.3 RabbitAdmin的核心方法

```java
// 声明Exchange
rabbitAdmin.declareExchange(Exchange exchange);

// 声明Queue
rabbitAdmin.declareQueue(Queue queue);

// 声明Binding
rabbitAdmin.declareBinding(Binding binding);

// 批量声明
rabbitAdmin.declareExchanges(exchanges);
rabbitAdmin.declareQueues(queues);
rabbitAdmin.declareBindings(bindings);

// 删除（第二个参数表示是否仅当未使用时删除）
rabbitAdmin.deleteExchange("my.exchange", true);
rabbitAdmin.deleteQueue("my.queue", true, true); // (名称, 是否unused, 是否empty)

// 清空队列
rabbitAdmin.purgeQueue("my.queue", true); // 第二个参数表示是否等待完成

// 获取队列信息
Properties props = rabbitAdmin.getQueueProperties("my.queue");
int messageCount = (Integer) props.get(RabbitAdmin.QUEUE_MESSAGE_COUNT);

// 获取Queue信息（包含消费者数、消息数等）
QueueInformation info = rabbitAdmin.getQueueInfo("my.queue");
```
## 3.4 声明规则与注意事项
### 3.4.1 同名组件属性冲突
如果Broker上已存在同名Exchange/Queue，但属性不同（如Queue的durable属性），再次声明会抛出异常。

解决方案一：删除后重建（开发环境）
```bash
# 通过管理界面删除，或代码删除
rabbitAdmin.deleteQueue("my.queue");
rabbitAdmin.declareQueue(new Queue("my.queue", true));
```

解决方案二：使用Policy（生产环境推荐）
RabbitMQ的Policy机制可以在不重新声明队列的情况下修改部分属性。

### 3.4.2 多个项目共享同一个Broker
当多个微服务共享同一个RabbitMQ时，要注意命名冲突：
- 使用有意义的前缀，如`order.create.queue`、`payment.notify.queue`
- 使用不同Virtual Host实现完全隔离（更彻底）

# 4. RabbitTemplate详解

## 4.1 什么是RabbitTemplate
`RabbitTemplate`是Spring AMQP中发送和接收消息的核心模板类，类似于JdbcTemplate、RedisTemplate的设计。它封装了：

- 连接的获取和释放
- Channel的创建和关闭
- 消息的序列化和反序列化（通过MessageConverter）
- 消息的发送和接收
- 确认和回调机制

## 4.2 内部工作流程

### 4.2.1 发送消息的流程
```java
// convertAndSend 的内部调用链：
rabbitTemplate.convertAndSend(exchange, routingKey, object)
    → MessageConverter.toMessage(object)           // 1. 对象转Message
    → doSend(exchange, routingKey, message)        // 2. 执行发送
        → execute(channel -> {                     // 3. 获取Channel
              channel.basicPublish(                 // 4. AMQP发送命令
                  exchange, routingKey, mandatory, immediate,
                  basicProperties, message.getBody()
              );
          });
```

### 4.2.2 接收消息的流程
```java
// receiveAndConvert 的内部调用链：
rabbitTemplate.receiveAndConvert(queueName)
    → doReceive(queueName, timeout)
        → execute(channel -> {
              GetResponse response = channel.basicGet(queueName, autoAck);  // AMQP拉取命令
              return response;
          });
    → MessageConverter.fromMessage(message)       // Message转对象
```

## 4.3 核心方法分类

### 4.3.1 发送消息
```java
// 发送原始Message
void send(String exchange, String routingKey, Message message);
void send(String routingKey, Message message); // 使用默认Exchange
void send(Message message); // 使用默认Exchange和默认RoutingKey

// 发送Java对象（自动转换）
void convertAndSend(String exchange, String routingKey, Object object);
void convertAndSend(String routingKey, Object object);
void convertAndSend(Object object);

// 带MessagePostProcessor的发送
void convertAndSend(String exchange, String routingKey, Object object,
                    MessagePostProcessor messagePostProcessor);

// 带CorrelationData的发送（用于发布确认）
void convertAndSend(String exchange, String routingKey, Object object,
                    CorrelationData correlationData);
```
（此章保持对核心组件的概述级详解，RabbitTemplate的完整用法将在"消息发送详解"章节深入展开）

### 4.3.2 接收消息
```java
// 同步接收（拉模式）
Message receive(String queueName);
Message receive(String queueName, long timeoutMillis);
Object receiveAndConvert(String queueName);
Object receiveAndConvert(String queueName, long timeoutMillis);

// RPC风格的发送并等待回复
Message sendAndReceive(Message message);
Object convertSendAndReceive(Object message);
```

## 4.4 MessagePostProcessor的使用
`MessagePostProcessor`可以在消息发送前对Message（或其MessageProperties）进行修改：

```java
rabbitTemplate.convertAndSend("order.exchange", "order.create", order, 
    new MessagePostProcessor() {
        @Override
        public Message postProcessMessage(Message message) throws AmqpException {
            MessageProperties props = message.getMessageProperties();
            props.setExpiration("10000");        // 设置过期时间10秒
            props.setPriority(5);                // 设置优先级
            props.setHeader("source", "order-service");  // 自定义头部
            return message;
        }
    }
);

// Java 8+ Lambda简化
rabbitTemplate.convertAndSend("order.exchange", "order.create", order, msg -> {
    msg.getMessageProperties().setExpiration("10000");
    return msg;
});
```

# 5. MessageListenerContainer详解

## 5.1 什么是MessageListenerContainer
`MessageListenerContainer`是Spring AMQP中管理消息消费者的核心组件，它负责：

- 管理消费者线程的生命周期
- 维持与Broker的连接
- 分发消息到MessageListener
- 处理消费者异常和恢复
- 管理Consumer的Ack/Nack

## 5.2 两种实现对比
Spring AMQP提供了两种消息监听容器：

| 特性 | SimpleMessageListenerContainer | DirectMessageListenerContainer |
|------|-------------------------------|-------------------------------|
| 底层模型 | 基于Consumer线程池 | 每个Queue一个Consumer，通过Channel直接消费 |
| 消费者模型 | 共享线程池处理多个Queue | 每个Queue独立Consumer实例 |
| 线程模型 | 使用TaskExecutor线程池 | 直接在amqp-client线程中调用Listener |
| prefetch管理 | 全局配置 | 每个Consumer独立配置 |
| 动态伸缩 | 支持根据负载动态调整消费者线程数 | 不支持动态伸缩 |
| 性能 | 中等 | 较高（减少线程切换开销） |
| 消息顺序 | 不保证 | 在单Consumer+prefetch=1时保证顺序 |
| 适用场景 | 通用场景，需要线程池管理的场景 | 高性能场景，需要明确消费者数量的场景 |
| 引入版本 | Spring AMQP 1.0 | Spring AMQP 2.1 |

## 5.3 SimpleMessageListenerContainer详解
Spring Boot默认使用`SimpleMessageListenerContainer`。

### 5.3.1 核心工作原理
```
SimpleMessageListenerContainer
  └── BlockingQueueConsumer（内部消费者实现）
        ├── 从AMQP Channel中拉取消息（通过basicConsume注册回调）
        ├── 将消息放入内部的BlockingQueue
        └── TaskExecutor线程池中的线程从BlockingQueue中取出消息
              └── 调用MessageListener.onMessage()
```

### 5.3.2 配置示例
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 5                # 最小消费者线程数
        max-concurrency: 20           # 最大消费者线程数
        prefetch: 250                 # 每个消费者预取数量
        acknowledge-mode: auto        # 确认模式
        default-requeue-rejected: false  # 拒绝后是否重新入队
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
          multiplier: 2.0
```

### 5.3.3 动态伸缩机制
SimpleMessageListenerContainer支持根据队列中的积压消息数量动态增加或减少消费者线程：

```
- 当BlockingQueue中的未处理消息 >= maxConcurrency * prefetch * 0.5 时，增加消费者
- 当消费者线程空闲超过一定时间（默认60秒）时，减少消费者
- 消费者线程数范围：[concurrency, max-concurrency]
```

## 5.4 DirectMessageListenerContainer详解
`DirectMessageListenerContainer`采用了更简单直接的设计：每个Queue被一个（或多个）独立的Consumer直接消费。

### 5.4.1 核心工作原理
```
DirectMessageListenerContainer
  └── 为每个Queue创建N个独立的Consumer（N = consumersPerQueue）
        ├── 每个Consumer使用独立的AMQP Channel
        ├── AMQP Client线程直接调用Listener（无中间BlockingQueue）
        └── Listener在amqp-client的线程中同步执行
```

### 5.4.2 配置示例
```yaml
spring:
  rabbitmq:
    listener:
      type: direct                          # 指定使用Direct类型
      direct:
        consumers-per-queue: 5              # 每个队列的消费者数量
        prefetch: 100                       # 每个消费者预取量
        acknowledge-mode: auto
```

### 5.4.3 选择建议
- 大多数场景下使用默认的`SimpleMessageListenerContainer`即可
- 如果追求极致的消息吞吐量，并且Listener中没有阻塞操作，可以尝试`DirectMessageListenerContainer`
- 如果需要严格的消息顺序保证，使用`DirectMessageListenerContainer` + `consumers-per-queue=1` + `prefetch=1`
- 如果需要动态伸缩消费者数量，只能使用`SimpleMessageListenerContainer`

# 6. Message消息对象详解

## 6.1 Message结构
Spring AMQP中的`Message`对象是AMQP消息的Java表示：

```java
public class Message implements Serializable {
    private final MessageProperties messageProperties;
    private final byte[] body;
    
    public Message(byte[] body, MessageProperties messageProperties) {
        this.body = body;
        this.messageProperties = messageProperties;
    }
}
```

## 6.2 MessageProperties详解
```java
MessageProperties props = new MessageProperties();

// 基本属性
props.setMessageId(UUID.randomUUID().toString());    // 消息ID
props.setTimestamp(new Date());                       // 时间戳
props.setContentType("application/json");             // 内容类型
props.setContentEncoding("UTF-8");                    // 内容编码

// 投递控制
props.setDeliveryMode(MessageDeliveryMode.PERSISTENT); // 持久化
props.setPriority(5);                                  // 优先级 0-9
props.setExpiration("60000");                          // 过期时间 毫秒

// 消息路由
props.setReceivedExchange("order.exchange");  // 接收时的Exchange
props.setReceivedRoutingKey("order.create");  // 接收时的RoutingKey

// RPC相关
props.setCorrelationId("corr-" + UUID.randomUUID());  // 关联ID
props.setReplyTo("reply.queue");                      // 回复队列

// 自定义头部
props.setHeader("x-retry-count", 3);
props.setHeader("x-source", "order-service");
props.setHeader("x-custom-metadata", "value");
```

# 7. 线程模型与性能考量

## 7.1 Channel的线程安全
一个关键的设计约束：**AMQP Channel不是线程安全的**。

RabbitMQ的Java客户端明确说明，Channel实例不能被多个线程同时使用。如果有两个线程同时在同一个Channel上发送命令，消息可能被交叉写入导致协议错误。

Spring AMQP通过以下方式解决：
- `CachingConnectionFactory`为每个线程/事务分配独立的Channel
- `RabbitTemplate`在每次操作时通过`execute()`获取独立的Channel
- 使用Channel缓存池，复用Channel但保证同一时刻一个Channel只被一个线程使用

## 7.2 性能优化要点
- **合理设置Channel缓存大小**：太小会导致等待，太大会占用过多连接资源。通常设置为最大并发数的1.2~1.5倍
- **合理设置Prefetch值**：太小吞吐量低，太大可能内存溢出和消息分配不均
- **选择合适的并发数**：根据CPU核心数和消息处理耗时计算，一般 `并发数 = CPU核心数 * (1 + 平均IO等待时间 / 平均CPU处理时间)`
- **避免在Listener中进行长时间阻塞操作**：会阻塞amqp-client线程，影响其他消息的处理
