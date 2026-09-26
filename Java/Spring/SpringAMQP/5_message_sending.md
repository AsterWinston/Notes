# 1.消息发送概览

## 1.1两种发送模型
Spring AMQP中的消息发送主要围绕`RabbitTemplate`展开，提供两种发送模型：

| 发送模型 | 说明 | 常用方法 |
|---------|------|---------|
| send() | 发送原始的`Message`对象 | send(exchange, routingKey, message) |
| convertAndSend() | 发送Java对象，自动转换 | convertAndSend(exchange, routingKey, object) |

`convertAndSend`是实际项目中最常用的方法，因为它会自动调用`MessageConverter`将Java对象转换为`Message`。

## 1.2RabbitTemplate的核心架构
```
RabbitTemplate
  ├── ConnectionFactory（获取连接和Channel）
  ├── MessageConverter（对象<->Message转换）
  ├── ConfirmCallback（消息到达Exchange的回调）
  ├── ReturnsCallback（消息无法路由到Queue的回调）
  ├── CorrelationDataPostProcessor（CorrelationData后处理器）
  ├── RetryTemplate（发送重试策略）
  └── 各种配置选项（mandatory、receiveTimeout等）
```

# 2.convertAndSend详解

## 2.1方法签名
```java
// 最基本的：指定Exchange、RoutingKey、消息对象
void convertAndSend(String exchange, String routingKey, Object message);

// 使用默认Exchange
void convertAndSend(String routingKey, Object message);

// 使用默认Exchange和默认RoutingKey
void convertAndSend(Object message);

// 带MessagePostProcessor
void convertAndSend(String exchange, String routingKey, Object message,
                    MessagePostProcessor messagePostProcessor);

// 带CorrelationData（用于Publisher Confirm）
void convertAndSend(String exchange, String routingKey, Object message,
                    CorrelationData correlationData);

// 带类型提示的消息转换（向消息头部添加类型信息）
void convertAndSend(String exchange, String routingKey, Object message,
                    Type type);
```

## 2.2基本使用示例
```java
@Service
public class OrderMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送订单创建消息
     */
    public void sendOrderCreated(Order order) {
        rabbitTemplate.convertAndSend(
            "order.topic.exchange",  // Exchange名称
            "order.created",         // Routing Key
            order                    // 消息体（Java对象）
        );
    }

    /**
     * 发送消息到默认交换机（直接指定Queue名称）
     */
    public void sendToDefaultExchange(String queueName, String message) {
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

## 2.3convertAndSend的内部流程
```
convertAndSend(exchange, routingKey, object)
  |
  ├── 1. 调用 MessageConverter.toMessage(object)
  |       ├── 如果是String, byte[] → 直接包装
  |       ├── 如果是Serializable → 使用SimpleMessageConverter序列化
  |       └── 如果配置了Jackson2JsonMessageConverter → JSON序列化
  |
  ├── 2. 构造CorrelationData（如果配置了ConfirmCallback）
  |
  ├── 3. 调用 doSend(exchange, routingKey, message, correlationData)
  |     |
  |     ├── 3.1 从ConnectionFactory获取Channel
  |     ├── 3.2 设置ConfirmListener（如果启用）
  |     ├── 3.3 设置ReturnListener（如果mandatory=true）
  |     └── 3.4 调用 channel.basicPublish(exchange, routingKey, mandatory, props, body)
  |
  └── 4. 如果启用了事务，提交事务
```

# 3.send详解

## 3.1方法签名
```java
// 发送原始Message
void send(String exchange, String routingKey, Message message);
void send(String routingKey, Message message);
void send(Message message);

// 带CorrelationData
void send(String exchange, String routingKey, Message message,
          CorrelationData correlationData);
```

## 3.2send vs convertAndSend
| 对比维度 | send() | convertAndSend() |
|---------|--------|------------------|
| 参数类型 | Message对象 | Java对象 |
| 序列化 | 需要自己构建Message | 自动通过MessageConverter |
| 使用场景 | 需要精确控制Message属性 | 日常业务消息发送 |

## 3.3send的使用示例
```java
@Service
public class CustomMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendWithCustomProperties(String messageBody) {
        // 手动构建Message
        MessageProperties props = new MessageProperties();
        props.setContentType("text/plain");
        props.setContentEncoding("UTF-8");
        props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
        props.setMessageId(UUID.randomUUID().toString());
        props.setTimestamp(new Date());
        props.setExpiration("60000");  // 60秒过期
        props.setPriority(5);
        props.setHeader("x-source", "order-service");
        props.setHeader("x-trace-id", "trace-12345");

        Message message = new Message(messageBody.getBytes(StandardCharsets.UTF_8), props);

        rabbitTemplate.send("order.exchange", "order.create", message);
    }
}
```

# 4.同步接收消息

## 4.1拉模式（Pull）API
RabbitTemplate也支持从Queue中主动拉取（Pull）消息：

```java
// 拉取一个Message（不会自动Ack）
Message receive(String queueName);
Message receive(String queueName, long timeoutMillis);

// 拉取并自动转换
Object receiveAndConvert(String queueName);
Object receiveAndConvert(String queueName, long timeoutMillis);
```

## 4.2使用示例
```java
@Service
public class PullMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 同步拉取一条消息
     */
    public void pullMessage() {
        Message message = rabbitTemplate.receive("order.queue", 5000); // 5秒超时
        if (message != null) {
            System.out.println("收到消息: " + new String(message.getBody()));
        } else {
            System.out.println("超时未收到消息");
        }
    }

    /**
     * 拉取并自动转换为Java对象
     */
    public void pullAndConvert() {
        Order order = (Order) rabbitTemplate.receiveAndConvert("order.queue", 5000);
        if (order != null) {
            processOrder(order);
        }
    }

    private void processOrder(Order order) {
        // 处理订单
    }
}
```

## 4.3Push vs Pull
| 维度 | Push模式（@RabbitListener） | Pull模式（RabbitTemplate.receive） |
|------|---------------------------|-----------------------------------|
| 消息获取方式 | Broker主动推送 | 应用主动拉取 |
| 延迟 | 低（消息到达即推送） | 取决于轮询频率 |
| CPU占用 | 低（事件驱动） | 如果频繁轮询则较高 |
| 流量控制 | 通过prefetch控制 | 应用自己控制拉取频率 |
| 使用场景 | 常规消费场景 | 按需消费、定时批量处理、兼容非AMQP协议 |
| Spring AMQP实现 | MessageListenerContainer | RabbitTemplate |

> 注意：Push模式是RabbitMQ和Spring AMQP的推荐使用方式，Pull模式主要适用于特殊场景。

# 5.RPC模式

## 5.1什么是RPC模式
RabbitMQ的RPC模式允许客户端发送请求消息后，等待服务端处理并返回响应消息。

```
Client                          Server
  |                                |
  |-- 发送请求到 request.queue --> |
  |                                |-- 处理请求
  | <-- 返回结果到 replyTo.queue -- |
  |                                |
```

## 5.2使用sendAndReceive
Spring AMQP通过`RabbitTemplate.sendAndReceive()`支持RPC模式：

```java
// 发送原始Message并等待回复
Message sendAndReceive(String exchange, String routingKey, Message message);
Message sendAndReceive(Message message);

// 发送Java对象并接收转换后的回复
Object convertSendAndReceive(String exchange, String routingKey, Object message);
Object convertSendAndReceive(Object message);

// 带类型参数
<T> T convertSendAndReceiveAsType(String exchange, String routingKey,
                                   Object message, ParameterizedTypeReference<T> type);
```

## 5.3RPC服务端实现
```java
@Component
public class RpcServer {

    @RabbitListener(queues = "rpc.request.queue")
    public String handleRpcRequest(String requestMessage) {
        System.out.println("收到RPC请求: " + requestMessage);
        
        // 处理业务逻辑
        String result = "处理结果: " + requestMessage.toUpperCase();
        
        return result;  // 返回值自动发送到replyTo指定的队列
    }
}
```

## 5.4RPC客户端实现
```java
@Service
public class RpcClientService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public String callRemoteService(String message) {
        // 发送请求并等待响应
        Object response = rabbitTemplate.convertSendAndReceive(
            "",                    // 使用默认Exchange
            "rpc.request.queue",   // Routing Key = 请求队列名
            message
        );
        
        return response != null ? response.toString() : null;
    }
}
```

## 5.5RPC的底层机制
当调用`convertSendAndReceive`时，Spring AMQP在底层做了以下事情：

1. 创建一个临时的匿名排他队列作为回复队列（replyTo）
2. 设置请求消息的`replyTo`属性为回复队列名称
3. 生成`correlationId`并设置到消息属性中
4. 发送请求消息
5. 启动一个消费者监听回复队列（阻塞等待）
6. 收到回复消息时，匹配`correlationId`确认是此请求的响应
7. 删除临时回复队列

## 5.6RPC配置
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    
    // 设置回复超时时间（默认5秒）
    template.setReplyTimeout(10000);  // 10秒
    
    // 也可以使用固定的回复队列（而不是每次创建临时队列）
    template.setReplyAddress("rpc.reply.queue");
    
    return template;
}
```

## 5.7RPC模式的适用场景
- 需要同步响应的远程调用（但在微服务中已被HTTP/gRPC取代）
- 遗留系统整合：需要异步传输但业务逻辑要求同步等待
- 特殊网络环境：HTTP不可用但AMQP可用的情况

# 6.MessagePostProcessor使用

## 6.1在发送时修改消息属性
```java
rabbitTemplate.convertAndSend("order.exchange", "order.create", order,
    new MessagePostProcessor() {
        @Override
        public Message postProcessMessage(Message message) throws AmqpException {
            MessageProperties props = message.getMessageProperties();
            props.setExpiration("30000");
            props.setPriority(3);
            props.setHeader("x-request-id", UUID.randomUUID().toString());
            return message;
        }
    }
);

// Lambda简化
rabbitTemplate.convertAndSend("order.exchange", "order.create", order, msg -> {
    msg.getMessageProperties().setExpiration("30000");
    return msg;
});
```

## 6.2全局MessagePostProcessor
可以通过配置全局的`beforePublishPostProcessors`对所有发送消息进行统一处理：

```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    
    // 添加全局后处理器
    template.addBeforePublishPostProcessors(message -> {
        // 为所有发送的消息添加统一的服务标识
        message.getMessageProperties().setHeader("x-service-name", "order-service");
        message.getMessageProperties().setHeader("x-service-instance",
                UUID.randomUUID().toString());
        return message;
    });
    
    return template;
}
```

# 7.消息持久化配置

## 7.1消息持久化的三个条件
消息要保证在Broker重启后不丢失，需要同时满足三个条件：

1. **Exchange持久化**：`new DirectExchange(name, true, false)` - 第二个参数
2. **Queue持久化**：`new Queue(name, true)` - 第二个参数
3. **消息持久化**：设置`deliveryMode = MessageDeliveryMode.PERSISTENT`

## 7.2配置消息持久化
```java
// 方式一：通过MessagePostProcessor
rabbitTemplate.convertAndSend("order.exchange", "order.create", order, msg -> {
    msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
    return msg;
});

// 方式二：全局配置（RabbitTemplate默认deliveryMode）
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    // 设置默认投递模式为持久化
    template.setMessageConverter(new Jackson2JsonMessageConverter());
    // 注意：RabbitTemplate没有直接设置默认deliveryMode的API
    // 需要通过全局MessagePostProcessor实现
    template.addBeforePublishPostProcessors(msg -> {
        msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
        return msg;
    });
    return template;
}
```

# 8.动态Exchange和Queue

## 8.1场景说明
在某些场景下，需要在运行时动态创建Exchange、Queue、Binding，而不是在应用启动时通过@Bean声明。例如：
- 多租户系统：每个租户需要独立的Exchange和Queue
- 动态业务流程：业务规则引擎动态创建处理链路
- 临时处理通道：为某些临时任务创建专用的消息通道

## 8.2动态创建实现
```java
@Service
public class DynamicQueueService {

    @Autowired
    private AmqpAdmin amqpAdmin;

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 为指定租户动态创建隔离的消息通道
     */
    public void setupTenantChannel(String tenantId) {
        String exchangeName = "tenant." + tenantId + ".exchange";
        String queueName = "tenant." + tenantId + ".queue";
        String routingKey = "tenant." + tenantId + ".event";

        // 创建Exchange
        Exchange exchange = ExchangeBuilder
                .topicExchange(exchangeName)
                .durable(true)
                .build();
        amqpAdmin.declareExchange(exchange);

        // 创建Queue
        Queue queue = QueueBuilder.durable(queueName).build();
        amqpAdmin.declareQueue(queue);

        // 创建Binding
        Binding binding = new Binding(queueName, Binding.DestinationType.QUEUE,
                exchangeName, routingKey, null);
        amqpAdmin.declareBinding(binding);
    }

    /**
     * 发送消息到动态创建的通道
     */
    public void sendToTenant(String tenantId, Object message) {
        String exchange = "tenant." + tenantId + ".exchange";
        String routingKey = "tenant." + tenantId + ".event";
        rabbitTemplate.convertAndSend(exchange, routingKey, message);
    }

    /**
     * 删除租户的通道（租户注销时）
     */
    public void teardownTenantChannel(String tenantId) {
        amqpAdmin.deleteExchange("tenant." + tenantId + ".exchange");
        amqpAdmin.deleteQueue("tenant." + tenantId + ".queue");
    }
}
```

# 9.批量消息发送

## 9.1场景说明
在某些场景下需要批量发送多条消息，如数据迁移、批量通知等。

## 9.2BatchingRabbitTemplate
Spring AMQP 2.2+提供了`BatchingRabbitTemplate`，可以将多条消息打包成一个批次发送：

```java
@Bean
public BatchingRabbitTemplate batchingRabbitTemplate(
        ConnectionFactory connectionFactory) {
    
    BatchingStrategy batchingStrategy = new SimpleBatchingStrategy(
        100,      // 每个批次的最大消息数
        4096,     // 每个批次的最大字节数
        10000     // 批次超时（毫秒），达到超时即使未满也发送
    );
    
    // 使用BatchingRabbitTemplate需要配置特殊的MessageConverter
    // 将多条消息序列化为一个大的字节数组
    
    BatchingRabbitTemplate template = new BatchingRabbitTemplate(
        batchingStrategy,
        new TaskScheduler() { /* ... */ },
        connectionFactory
    );
    
    return template;
}
```

## 9.3批量发送的实际使用
BatchingRabbitTemplate的使用比较复杂，实际项目中更常见的做法是：
- 批量发送场景使用Kafka代替RabbitMQ
- 在应用层聚合消息，将多条消息打包成一条消息发送（如JSON数组）

```java
// 应用层聚合示例
public void batchNotify(List<Notification> notifications) {
    // 将多条通知打包成一条消息
    String json = objectMapper.writeValueAsString(notifications);
    rabbitTemplate.convertAndSend("notify.exchange", "notify.batch", json);
}
```

# 10.线程安全与并发发送

## 10.1RabbitTemplate的线程安全
`RabbitTemplate`是**线程安全**的，可以安全地在多个线程中并发使用同一个实例。

线程安全的实现原理：
- `RabbitTemplate`本身不持有状态（除了配置属性）
- 每次`send()`/`convertAndSend()`操作都会通过`ConnectionFactory`获取独立的Channel
- 多个线程各自使用独立的Channel，互不干扰

## 10.2高并发发送的优化
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    
    // 使用异步确认，避免等待阻塞
    factory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
    
    // 增加Channel缓存大小（根据并发量调整）
    factory.setChannelCacheSize(100);
    
    // 是否需要在发送时确认（设置为false提高吞吐）
    template.setUsePublisherConnection(true);  // 使用独立的连接进行发布
    
    return template;
}
```

`setUsePublisherConnection(true)`的作用是使用一个专用的Connection来处理消息发送操作，与消息消费的Connection分离，避免相互影响。
