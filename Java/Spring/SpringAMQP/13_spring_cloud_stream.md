# 1. Spring Cloud Stream概述

## 1.1 什么是Spring Cloud Stream
Spring Cloud Stream是Spring Cloud生态中用于构建**消息驱动微服务**的框架。它在Spring AMQP等消息中间件之上提供了更高层次的抽象，使开发者可以用统一的编程模型来操作不同的消息中间件。

核心设计理念：**开发者只需要关注业务逻辑，消息中间件的细节由框架处理**。

## 1.2 与Spring AMQP的关系
```
Spring Cloud Stream（高层抽象，统一的编程模型）
          ↓
Spring AMQP          Spring Kafka       Spring Pulsar
（AMQP实现）          （Kafka实现）        （Pulsar实现）
          ↓
RabbitMQ              Kafka              Pulsar
```

Spring AMQP和Spring Cloud Stream的关系：
- **Spring AMQP**是底层框架，直接操作RabbitMQ的API
- **Spring Cloud Stream**是上层抽象，通过Binder适配不同的消息中间件
- Spring Cloud Stream集成RabbitMQ时，底层仍然使用Spring AMQP
- Spring Cloud Stream屏蔽了不同MQ的差异，但损失了一些特定MQ的高级特性

## 1.3 适用场景
| 场景 | 推荐 | 原因 |
|------|------|------|
| 只使用RabbitMQ | Spring AMQP | 更灵活，功能更完整 |
| 需要未来切换MQ | Spring Cloud Stream | 统一抽象，更换Binder即可 |
| 多MQ并存 | Spring Cloud Stream | 统一的编程模型 |
| 使用RabbitMQ高级特性（死信、延迟等） | Spring AMQP | Stream可能不支持或配置复杂 |

**建议**：大多数项目中，如果确定了使用RabbitMQ且不需要切换，直接使用Spring AMQP更简单直接。

# 2. Spring Cloud Stream核心概念

## 2.1 Binder（绑定器）
Binder是Spring Cloud Stream最核心的概念，它是与具体消息中间件集成的适配器。

```xml
<!-- RabbitMQ Binder -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>

<!-- Kafka Binder（如果需要的话） -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
```

## 2.2 Binding（绑定）
Binding是应用程序和Binder之间的桥梁：

```
Application
  ├── @Input (Sink)    → Binding → Binder → [Broker Topic/Queue]
  └── @Output (Source) → Binding → Binder → [Broker Topic/Queue]
```

## 2.3 核心编程模型（函数式）
Spring Cloud Stream 3.x推荐使用函数式编程模型（`java.util.function`）：

```java
@SpringBootApplication
public class OrderStreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderStreamApplication.class, args);
    }

    // 消息消费者：接收订单创建消息
    @Bean
    public Consumer<Order> orderCreateConsumer() {
        return order -> {
            System.out.println("收到订单: " + order);
        };
    }

    // 消息生产者：发送订单创建消息
    @Bean
    public Supplier<Order> orderCreateSupplier() {
        return () -> {
            // 从数据库/缓存中获取待发送的数据
            return null;
        };
    }

    // 消息处理器：接收消息并处理后发送新消息
    @Bean
    public Function<Order, OrderResult> orderProcessor() {
        return order -> {
            return orderService.process(order);
        };
    }
}
```

## 2.4 配置
```yaml
spring:
  cloud:
    stream:
      # 配置Binder
      binders:
        rabbit:
          type: rabbit
      # 配置Binding
      bindings:
        orderCreateConsumer-in-0:
          destination: order.exchange
          group: order-service
          binder: rabbit
        orderCreateSupplier-out-0:
          destination: order.exchange
          binder: rabbit
        orderProcessor-in-0:
          destination: order.input.queue
          group: processor-group
        orderProcessor-out-0:
          destination: order.output.queue
      # RabbitMQ Binder特定配置
      rabbit:
        bindings:
          orderCreateConsumer-in-0:
            consumer:
              binding-routing-key: order.create
              exchange-type: topic
              acknowledge-mode: auto
              max-concurrency: 10
              prefetch: 100
```

# 3. 从Spring AMQP迁移到Spring Cloud Stream

## 3.1 发送消息对比
Spring AMQP:
```java
@Autowired
private RabbitTemplate rabbitTemplate;

rabbitTemplate.convertAndSend("order.exchange", "order.create", order);
```

Spring Cloud Stream:
```java
@Autowired
private StreamBridge streamBridge;

streamBridge.send("orderCreateSupplier-out-0", order);
```

## 3.2 接收消息对比
Spring AMQP:
```java
@RabbitListener(queues = "order.create.queue")
public void handleOrder(Order order) {
    // 处理订单
}
```

Spring Cloud Stream:
```java
@Bean
public Consumer<Order> orderConsumer() {
    return order -> {
        // 处理订单
    };
}
```

# 4. Spring Cloud Stream的高级特性

## 4.1 分区支持
```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: order.process
          group: processor
          consumer:
            partitioned: true
        processOrder-out-0:
          destination: order.process
          producer:
            partitioned: true
            partition-key-expression: headers['partitionKey']
            partition-count: 3
```

## 4.2 死信队列（DLQ）
```yaml
spring:
  cloud:
    stream:
      rabbit:
        bindings:
          orderConsumer-in-0:
            consumer:
              auto-bind-dlq: true          # 自动绑定死信队列
              dlq-ttl: 60000               # 死信队列TTL
              dlq-dead-letter-exchange: ""  # 死信消息回投的交换机
              republish-to-dlq: true        # 将失败消息重新发布到DLQ
              max-attempts: 3               # 最大重试次数
```

## 4.3 消费者重试
```yaml
spring:
  cloud:
    stream:
      bindings:
        orderConsumer-in-0:
          consumer:
            max-attempts: 3              # 最大重试
            back-off-initial-interval: 1000  # 初始退避间隔
            back-off-max-interval: 10000     # 最大退避间隔
            back-off-multiplier: 2.0         # 退避乘数
```

# 5. 何时使用Spring Cloud Stream vs Spring AMQP

## 5.1 选择Spring AMQP的场景
- 只需要使用RabbitMQ一种消息中间件
- 需要RabbitMQ的高级特性（死信队列、延迟队列、事务消息等）
- 需要精确控制消息属性（TTL、优先级、Headers等）
- 需要`@RabbitListener`的动态监听
- 团队对RabbitMQ和Spring AMQP已很熟悉

## 5.2 选择Spring Cloud Stream的场景
- 未来可能切换消息中间件（RabbitMQ → Kafka等）
- 需要同时支持多种消息中间件的统一抽象
- 项目中已大量使用Spring Cloud其他组件
- 消息需求比较简单（发送/接收），不需要MQ的高级特性

## 5.3 总结
对于大多数基于RabbitMQ的项目，**Spring AMQP是更好的选择**：
- 功能更完整
- 配置更直观
- 性能更好（少一层抽象）
- 问题排查更容易

Spring Cloud Stream更适合需要**消息中间件无感**的场景，但以牺牲灵活性和一些高级特性为代价。
