
# 1. 概述

RocketMQ 为 Spring 生态提供了完善的集成支持：

| 项目 | 版本 | 说明 |
|------|------|------|
| `rocketmq-spring-boot-starter` | 2.x | Spring Boot 自动配置 |
| `rocketmq-spring` | 2.x | Spring 框架集成 (非 Boot) |
| `spring-cloud-starter-stream-rocketmq` | - | Spring Cloud Stream Binder |

# 2. Spring Boot 集成 (rocketmq-spring-boot-starter)

## 2.1 依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.3</version>
</dependency>
```

## 2.2 配置文件

```yaml
# application.yml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: spring-boot-producer-group
    send-message-timeout: 3000
    retry-times-when-send-failed: 2
    max-message-size: 4194304
  consumer:
    group: spring-boot-consumer-group
    topic: test-topic
    consume-thread-min: 5
    consume-thread-max: 20
```

## 2.3 Producer 示例

```java
@Slf4j
@SpringBootApplication
public class SpringBootProducer implements CommandLineRunner {

    @Resource
    private RocketMQTemplate rocketMQTemplate;

    public static void main(String[] args) {
        SpringApplication.run(SpringBootProducer.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        // 1. 同步发送
        SendResult result = rocketMQTemplate.syncSend("test-topic", 
            MessageBuilder.withPayload("Hello RocketMQ").build());
        System.out.println(result);

        // 2. 发送带 Tag 的消息
        rocketMQTemplate.syncSend("test-topic:tagA", 
            MessageBuilder.withPayload("Hello with Tag").build());

        // 3. 异步发送
        rocketMQTemplate.asyncSend("test-topic", 
            MessageBuilder.withPayload("Async Message").build(), 
            new SendCallback() {
                @Override
                public void onSuccess(SendResult result) {
                    log.info("发送成功: {}", result);
                }
                @Override
                public void onException(Throwable e) {
                    log.error("发送失败", e);
                }
            });

        // 4. 顺序发送
        rocketMQTemplate.syncSendOrderly("order-topic",
            MessageBuilder.withPayload("Order Message").build(), "orderId_001");

        // 5. 延迟消息
        Message<String> delayMsg = MessageBuilder.withPayload("Delay Message").build();
        rocketMQTemplate.syncSend("delay-topic", delayMsg, 3000, 3); // delayLevel=3

        // 6. 事务消息
        rocketMQTemplate.sendMessageInTransaction("tx-topic",
            MessageBuilder.withPayload("Tx Message").build(), "args");
    }
}
```

## 2.4 Consumer 示例

```java
@Slf4j
@Service
@RocketMQMessageListener(
    topic = "test-topic",
    consumerGroup = "spring-boot-consumer-group",
    selectorExpression = "tagA || tagB",       // Tag 过滤
    consumeMode = ConsumeMode.CONCURRENTLY,    // 并发消费
    messageModel = MessageModel.CLUSTERING,    // 集群模式
    consumeThreadMax = 10,
    maxReconsumeTimes = 5
)
public class SpringBootConsumer implements RocketMQListener<MessageExt> {

    @Override
    public void onMessage(MessageExt msg) {
        String body = new String(msg.getBody());
        log.info("收到消息: topic={}, tags={}, body={}", 
            msg.getTopic(), msg.getTags(), body);
        // 业务处理...
        // 如果抛异常 → 消息会重试
    }
}

// 顺序消费
@Service
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-consumer-group",
    consumeMode = ConsumeMode.ORDERLY           // 顺序消费
)
public class OrderlyConsumer implements RocketMQListener<MessageExt> {
    @Override
    public void onMessage(MessageExt msg) {
        // 同一 Queue 的消息会串行进入此方法
    }
}
```

## 2.5 多 Topic 消费

```java
// 方案1: 多个 @RocketMQMessageListener 注解（不同类）
// 方案2: 使用 ExtRocketMQConsumer 动态订阅

@Component
public class MultiTopicConsumer implements RocketMQPushConsumerLifecycleListener {
    @Override
    public void prepareStart(DefaultMQPushConsumer consumer) {
        consumer.subscribe("topic-a", "*");
        consumer.subscribe("topic-b", "*");
        consumer.setInstanceName("multi-topic-consumer");
    }
}
```

# 3. Spring Cloud Stream 集成

## 3.1 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

## 3.2 配置

```yaml
spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
      bindings:
        output:
          destination: test-topic
          content-type: application/json
        input:
          destination: test-topic
          content-type: application/json
          group: consumer-group
          consumer:
            max-attempts: 3   # 最大重试次数
```

## 3.3 Producer

```java
@RestController
public class MessageController {

    @Autowired
    private Source source;       // 或自定义 Binding

    @GetMapping("/send")
    public String send(@RequestParam String msg) {
        source.output().send(MessageBuilder.withPayload(msg).build());
        return "OK";
    }
}
```

## 3.4 Consumer

```java
@EnableBinding(Sink.class)
public class StreamConsumer {

    @StreamListener(Sink.INPUT)
    public void receive(String message) {
        System.out.println("收到: " + message);
        // 抛异常 → 自动重试 (根据 max-attempts)
    }
}
```

# 4. 纯 Spring 集成 (非 Boot)

```java
@Configuration
public class RocketMQConfig {

    @Value("${rocketmq.name-server}")
    private String nameServer;

    @Bean(initMethod = "start", destroyMethod = "shutdown")
    public DefaultMQProducer defaultMQProducer() {
        DefaultMQProducer producer = new DefaultMQProducer("spring-producer-group");
        producer.setNamesrvAddr(nameServer);
        producer.setRetryTimesWhenSendFailed(3);
        return producer;
    }

    @Bean(initMethod = "start", destroyMethod = "shutdown")
    public DefaultMQPushConsumer defaultMQPushConsumer() {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("spring-consumer-group");
        consumer.setNamesrvAddr(nameServer);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        consumer.subscribe("test-topic", "*");
        consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
            for (MessageExt msg : msgs) {
                System.out.println(new String(msg.getBody()));
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });
        return consumer;
    }
}
```

# 5. RocketMQTemplate 核心方法速查

| 方法 | 说明 | 返回 |
|------|------|------|
| `syncSend(topic, message)` | 同步发送 | SendResult |
| `syncSend(topic, message, timeout)` | 同步发送(带超时) | SendResult |
| `asyncSend(topic, message, callback)` | 异步发送 | void |
| `sendOneWay(topic, message)` | 单向发送 | void |
| `syncSendOrderly(topic, message, hashKey)` | 顺序发送 | SendResult |
| `sendMessageInTransaction(topic, message, arg)` | 事务发送 | TransactionSendResult |
| `syncSend(topic, message, timeout, delayLevel)` | 延迟发送 | SendResult |
| `convertAndSend(topic, payload)` | 发送对象(自动JSON) | void |

# 6. 自动配置原理

```java
// rocketmq-spring-boot-autoconfigure 中的自动配置
@Configuration
@EnableConfigurationProperties(RocketMQProperties.class)
@ConditionalOnClass({MQAdmin.class})
@Import({ListenerContainerConfiguration.class, ExtProducerResetConfiguration.class})
public class RocketMQAutoConfiguration {

    @Bean(destroyMethod = "destroy")
    @ConditionalOnMissingBean
    public DefaultMQProducer defaultMQProducer(RocketMQProperties props) {
        DefaultMQProducer producer = new DefaultMQProducer(props.getProducer().getGroup());
        producer.setNamesrvAddr(props.getNameServer());
        // ... 设置其他属性
        return producer;
    }

    @Bean(destroyMethod = "destroy")
    @ConditionalOnMissingBean
    public RocketMQTemplate rocketMQTemplate(DefaultMQProducer mqProducer) {
        RocketMQTemplate template = new RocketMQTemplate();
        template.setProducer(mqProducer);
        return template;
    }
}
```

# 7. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| RocketMQTemplate 无法注入 | 未配置 rocketmq.name-server | 检查配置 |
| 消费者不消费 | @RocketMQMessageListener 未被 Spring 管理 | 添加 @Service/@Component |
| 事务消息不生效 | 未配置 RocketMQLocalTransactionListener | 实现并注册对应事务监听 |
| JSON 消息消费异常 | ContentType 不匹配 | 配置 `content-type: application/json` |

# 8. 参考

- [rocketmq-spring GitHub](https://github.com/apache/rocketmq-spring)
