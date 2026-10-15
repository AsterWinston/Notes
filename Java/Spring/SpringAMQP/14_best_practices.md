# 1. 设计模式与架构最佳实践

## 1.1 消息格式规范
统一消息格式可以减少解析成本和错误：

```java
/**
 * 统一的消息封装
 */
public class BaseMessage<T> {
    private String messageId;        // 消息唯一ID
    private String messageType;      // 消息类型
    private Long timestamp;          // 发送时间戳
    private String source;           // 消息来源服务
    private String traceId;          // 链路追踪ID
    private T data;                  // 消息体
    private String version;          // 消息格式版本号
}

// 发送端封装
rabbitTemplate.convertAndSend("order.exchange", "order.create",
    BaseMessage.of(order, "order_create"));

// 消费端解包
@RabbitListener(queues = "order.queue")
public void handleMessage(BaseMessage<Order> baseMessage) {
    String traceId = baseMessage.getTraceId();
    Order order = baseMessage.getData();
    // 处理业务
}
```

## 1.2 服务间消息目录
建议在项目中建立统一的消息目录（Message Catalog），管理所有消息的定义：

```java
/**
 * 消息目录：定义所有消息的Exchange、Queue、RoutingKey
 */
public final class MessageCatalog {

    // 订单模块
    public static final class Order {
        public static final String EXCHANGE = "order.topic.exchange";
        public static final String QUEUE_CREATE = "order.create.queue";
        public static final String QUEUE_CANCEL = "order.cancel.queue";
        public static final String QUEUE_DLQ = "order.dlq.queue";
        
        public static final String RK_CREATE = "order.create";
        public static final String RK_CANCEL = "order.cancel";
        public static final String RK_DEAD = "order.dead";
    }
    
    // 支付模块
    public static final class Payment {
        public static final String EXCHANGE = "payment.topic.exchange";
        public static final String QUEUE_SUCCESS = "payment.success.queue";
        public static final String RK_SUCCESS = "payment.success";
    }
}

// 使用
rabbitTemplate.convertAndSend(
    MessageCatalog.Order.EXCHANGE,
    MessageCatalog.Order.RK_CREATE,
    order);
```

## 1.3 配置集中管理
```java
@Configuration
public class RabbitMQGlobalConfig {

    /**
     * 所有Queue、Exchange、Binding的统一定义
     */
    
    // ===== 订单模块 =====
    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder.topicExchange("order.topic.exchange")
            .durable(true)
            .withArgument("alternate-exchange", "unroutable.exchange")  // 备用交换机
            .build();
    }
    
    @Bean
    public Queue orderCreateQueue() {
        return QueueBuilder.durable("order.create.queue")
            .deadLetterExchange("dlx.order.exchange")
            .deadLetterRoutingKey("dlx.order")
            .build();
    }
    
    // ===== 全局 =====
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                          Jackson2JsonMessageConverter converter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(converter);
        
        // 全局确认回调
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("消息确认失败: id={}, cause={}", 
                    correlationData != null ? correlationData.getId() : "null", cause);
            }
        });
        
        // 全局退回回调
        template.setReturnsCallback(returned -> {
            log.warn("消息无法路由: exchange={}, routingKey={}, replyText={}",
                returned.getExchange(), returned.getRoutingKey(), 
                returned.getReplyText());
        });
        
        return template;
    }
}
```

# 2. 幂等性实践

## 2.1 为什么需要幂等性
在消息队列中，消息可能会被**重复投递**：
- 消费者处理成功但Ack网络超时，Broker重发
- 生产者因超时重试，发送重复消息
- Broker故障恢复后重发

因此，**消费者必须保证幂等性**：同一消息无论消费多少次，最终结果与消费一次相同。

## 2.2 实现方案

### 2.2.1 基于数据库唯一约束
```java
@RabbitListener(queues = "order.create.queue")
public void handleOrderCreate(OrderMessage msg) {
    try {
        // 使用唯一键约束保证幂等
        orderRepository.insert(msg.toOrder());  // messageId作为唯一键
    } catch (DuplicateKeyException e) {
        // 消息已处理，忽略
        log.info("消息已处理，跳过: {}", msg.getMessageId());
    }
}
```

### 2.2.2 基于Redis去重
```java
@RabbitListener(queues = "order.create.queue")
public void handleOrderCreate(OrderMessage msg) {
    String dedupKey = "order:dedup:" + msg.getMessageId();
    
    // 使用SET NX EX（仅当key不存在时设置，并设置过期时间）
    Boolean isDuplicate = redisTemplate.opsForValue()
        .setIfAbsent(dedupKey, "1", 24, TimeUnit.HOURS);
    
    if (Boolean.FALSE.equals(isDuplicate)) {
        log.info("消息已处理，跳过: {}", msg.getMessageId());
        return;
    }
    
    // 处理业务逻辑
    orderService.process(msg.toOrder());
}
```

### 2.2.3 基于数据库去重表
```java
CREATE TABLE message_dedup (
    message_id VARCHAR(64) PRIMARY KEY,
    status VARCHAR(20),
    created_at DATETIME,
    INDEX idx_created_at (created_at)
);

@RabbitListener(queues = "order.create.queue")
public void handleOrderCreate(OrderMessage msg) {
    // 先插入去重表（messageId作为主键）
    try {
        messageDedupRepository.insert(msg.getMessageId(), "PROCESSING");
    } catch (DuplicateKeyException e) {
        log.info("消息已处理，跳过: {}", msg.getMessageId());
        return;
    }
    
    // 处理业务
    try {
        orderService.process(msg.toOrder());
        messageDedupRepository.updateStatus(msg.getMessageId(), "SUCCESS");
    } catch (Exception e) {
        messageDedupRepository.updateStatus(msg.getMessageId(), "FAILED");
        throw e;
    }
}
```

## 2.3 幂等性方案对比
| 方案 | 性能 | 可靠性 | 实现复杂度 | 适用场景 |
|------|------|--------|-----------|---------|
| 数据库唯一约束 | 中 | 高 | 低 | 操作本身有自然唯一ID |
| Redis | 高 | 中 | 中 | 高并发、可容忍少量丢失 |
| 去重表 | 中 | 高 | 中 | 需要完整记录的消息去重 |

# 3. 消息顺序性

## 3.1 RabbitMQ中的消息顺序
RabbitMQ在默认配置下**不保证消息的全局顺序**，原因：
- 多个消费者并发消费时，处理速度不同导致乱序
- 消息重试时可能打乱顺序
- 网络延迟导致到达顺序不确定

## 3.2 保证消息顺序的方案

### 3.2.1 单消费者 + prefetch=1
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 1          # 只有一个消费者
        prefetch: 1             # 每次只取一条消息
```

```java
@RabbitListener(queues = "order.sequence.queue", concurrency = "1")
public void handleSequentialOrder(Order order) {
    // 确保单线程处理，消息按顺序消费
    orderService.processSequentially(order);
}
```

### 3.2.2 按业务Key分区
如果只需要相关消息有序（如同一订单的消息），可以将同一业务Key的消息发送到同一队列：

```java
// 根据订单ID哈希，将同一订单的消息路由到固定队列
int index = Math.abs(orderId.hashCode() % QUEUE_COUNT);
String queueName = "order.partition.queue." + index;
rabbitTemplate.convertAndSend(queueName, orderMessage);
```

# 4. 性能优化

## 4.1 生产者端优化
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);

    // 1. 使用单独的发布连接（与消费连接分离）
    template.setUsePublisherConnection(true);

    // 2. 异步确认（不要同步等待确认结果）
    factory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);

    // 3. 批量发送时使用批量模板（Spring AMQP 2.2+）
    // BatchingRabbitTemplate batchingTemplate = new BatchingRabbitTemplate(...);
    
    return template;
}
```

## 4.2 消费者端优化
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 合理设置并发数
        concurrency: 5
        max-concurrency: 20
        # 每个消费者预取数量（根据处理速度调整）
        prefetch: 100
```

```java
// 使用异步非阻塞处理
@RabbitListener(queues = "high.throughput.queue")
public CompletableFuture<Void> handleAsync(Message message) {
    return CompletableFuture.runAsync(() -> {
        // 非阻塞处理
        processMessageAsync(message);
    });
}
```

## 4.3 Broker端优化
```bash
# 1. 消息持久化优化：批量确认
# 配置 broker.conf
queue_index_embed_msgs_below = 4096  # 小于4KB的消息嵌入索引

# 2. 内存优化：合理设置watermark
vm_memory_high_watermark.relative = 0.4

# 3. 磁盘IO优化：使用SSD
# 4. 网络优化：增大TCP缓冲区
```

## 4.4 性能基准参考
| 指标 | 参考值（单节点） |
|------|----------------|
| 简单消息吞吐（非持久化） | 5-10万 msg/s |
| 简单消息吞吐（持久化） | 1-2万 msg/s |
| 带确认的持久化消息 | 5000-10000 msg/s |
| 事务消息 | 200-1000 msg/s |

# 5. 常见问题排查

## 5.1 消息丢失
```
现象：消息发送了但消费者没收到
排查步骤：
1. 检查ConfirmCallback是否返回ack=false
2. 检查ReturnsCallback是否被触发（RoutingKey不匹配）
3. 查看RabbitMQ管理界面：消息是否在Queue中堆积
4. 检查Exchange和Queue的配置是否正确（名称、类型、Binding）
5. 检查消息是否被设置了TTL而过期
6. 检查是否有其他消费者消费了该消息
7. 检查消费者是否正确处理了Ack
```

## 5.2 消息重复消费
```
现象：同一条消息被消费多次
排查步骤：
1. 检查消费确认模式（acknowledge-mode）
2. 如果是auto模式，检查方法是否抛异常导致Nack+requeue
3. 检查消费者是否配置了重试
4. 检查网络是否超时（消费者处理成功但Ack超时）
解决方案：消费端实现幂等性
```

## 5.3 消费者不消费
```
现象：队列中有消息但消费者不消费
排查步骤：
1. 检查消费者的连接状态（管理界面Connections tab）
2. 检查消费者是否已启动（RabbitListenerEndpointRegistry）
3. 检查消费者的prefetch是否耗尽
4. 检查消费线程是否死锁或阻塞
5. 检查Channel是否被关闭（ShutdownSignalException）
6. 检查autoStartup配置是否为false
```

## 5.4 消息堆积
```
现象：队列中消息数量持续增长
排查步骤：
1. 检查消费者处理速度：增加concurrency
2. 检查消费者是否异常停止
3. 检查消息是否都发到了一个队列（应该分散到多个队列）
4. 检查是否有按时间分发消息的需求（可以增加更多消费者）
5. 临时方案：增加消费者数量，提高prefetch
```

## 5.5 连接断开
```
现象：应用日志中出现Connection reset
排查步骤：
1. 检查RabbitMQ Broker是否重启
2. 检查防火墙/安全组是否拦截了连接
3. 检查网络延迟是否超时（requested-heartbeat）
4. 检查Channel缓存是否耗尽（等待获取Channel超时）
5. 检查Broker的连接数限制是否达到上限
```

# 6. 开发调试技巧

## 6.1 本地开发环境
```java
// 使用TestRabbitTemplate进行测试
@SpringBootTest
class MessageTest {

    @Autowired
    private TestRabbitTemplate testRabbitTemplate;

    @Test
    void testSendAndReceive() {
        Order order = new Order();
        order.setId("123");
        
        // 发送消息
        testRabbitTemplate.convertAndSend("order.exchange", "order.create", order);
        
        // 等待并确认消费
        Order received = (Order) testRabbitTemplate.receiveAndConvert("order.queue", 5000);
        assertThat(received.getId()).isEqualTo("123");
    }
}
```

## 6.2 RabbitListenerTestHarness
```java
@SpringBootTest
class ListenerTest {

    @Autowired
    private RabbitListenerTestHarness harness;

    @Test
    void testListener() {
        // 获取监听器
        MessageListenerContainer container = harness.getSpy("orderListener");
        assertThat(container.isRunning()).isTrue();
    }
}
```

## 6.3 日志配置
```yaml
logging:
  level:
    org.springframework.amqp: DEBUG
    org.springframework.amqp.rabbit: DEBUG
    com.rabbitmq.client: DEBUG
```

## 6.4 Mock测试
不使用真实RabbitMQ的单元测试：

```java
@ExtendWith(MockitoExtension.class)
class OrderMessageServiceTest {

    @Mock
    private RabbitTemplate rabbitTemplate;

    @InjectMocks
    private OrderMessageService messageService;

    @Test
    void shouldSendOrderCreatedMessage() {
        Order order = new Order();
        order.setId("123");

        messageService.sendOrderCreated(order);

        verify(rabbitTemplate).convertAndSend(
            "order.exchange", "order.create", order);
    }
}
```

# 7. 安全最佳实践

## 7.1 最小权限原则
```bash
# 为不同服务创建独立的用户，分配最小权限
rabbitmqctl add_user order_service password
rabbitmqctl set_permissions -p / order_service "^order\." "^order\." "^order\."
# 只允许 order_service 用户访问以 "order." 开头的Exchange/Queue
```

## 7.2 使用TLS加密
```yaml
spring:
  rabbitmq:
    ssl:
      enabled: true
      algorithm: TLSv1.2
      key-store: classpath:client.p12
      key-store-password: password
      trust-store: classpath:truststore.p12
```

## 7.3 消息体敏感信息处理
```java
// 发送前脱敏
@RabbitListener(queues = "user.queue")
public void handleUser(UserMessage msg) {
    // 对敏感信息进行脱敏处理后再记录日志
    log.info("处理用户消息: id={}, phone={}", 
        msg.getUserId(), maskPhone(msg.getPhone()));
}

private String maskPhone(String phone) {
    if (phone == null || phone.length() < 7) return phone;
    return phone.substring(0, 3) + "****" + phone.substring(7);
}
```

# 8. 面试高频知识点

## 8.1 消息队列常见面试题
以下是用Spring AMQP知识可以回答的高频面试题：

**Q1: RabbitMQ如何保证消息不丢失？**
A: 三阶段保障——生产端Publisher Confirm + Broker端持久化(Exchange/Queue/Message) + 消费端Consumer Ack。

**Q2: RabbitMQ如何保证消息的顺序性？**
A: 单消费者+prefetch=1；或按业务Key排序后分段发送到不同队列。

**Q3: 消息队列如何实现延迟队列？**
A: TTL+死信队列（队列级别TTL或消息级别TTL）；或使用rabbitmq_delayed_message_exchange插件。

**Q4: 如何处理消息的重复消费？**
A: 消费端实现幂等性——数据库唯一约束、Redis去重、去重表三种方案。

**Q5: RabbitMQ的集群模式有哪些？**
A: 普通集群（仅元数据复制）、节点镜像（已废弃）、仲裁队列（推荐，基于Raft）。

**Q6: Spring AMQP的RabbitTemplate是线程安全的吗？**
A: 是的，每次操作通过ConnectionFactory获取独立Channel。

**Q7: RabbitMQ有哪些交换机类型？分别适用于什么场景？**
A: Direct(精确路由)、Topic(模式匹配)、Fanout(广播)、Headers(头部匹配，较少使用)。

**Q8: 如何保证DB和MQ的数据一致性？**
A: 本地消息表 + 定时补偿，或使用支持事务消息的MQ（RocketMQ）。

## 8.2 常见配置清单（面试笔试）
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    
    publisher-confirm-type: correlated     # 发布确认
    publisher-returns: true                # 退回处理
    
    listener:
      simple:
        concurrency: 5                     # 最小消费者
        max-concurrency: 20                # 最大消费者
        prefetch: 100                      # 预取数
        acknowledge-mode: auto             # 确认模式
        retry:
          enabled: true                    # 重试
          max-attempts: 3
          initial-interval: 1000
```

# 9. 学习资源推荐
- RabbitMQ官方文档：https://www.rabbitmq.com/documentation.html
- Spring AMQP官方文档：https://spring.io/projects/spring-amqp
- Spring AMQP Reference：https://docs.spring.io/spring-amqp/reference/
- RabbitMQ in Action（书籍）
- AMQP 0-9-1 Protocol Specification：https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf
