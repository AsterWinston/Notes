# 1 Spring Kafka 概述

## 1.1 什么是 Spring Kafka

Spring Kafka 是 Spring 生态中用于集成 Apache Kafka 的官方框架，提供模板化的发送、消息监听容器、事务管理和错误处理等能力。

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.1.0</version>  <!-- Spring Boot 3.2 对应 -->
</dependency>
```

```groovy
// Gradle
implementation 'org.springframework.kafka:spring-kafka'
```

# 2 基础配置

## 2.1 application.yml 配置

```yaml
spring:
  kafka:
    bootstrap-servers: broker1:9092,broker2:9092,broker3:9092

    # === Producer 默认配置 ===
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      compression-type: lz4
      batch-size: 16384
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5

    # === Consumer 默认配置 ===
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      group-id: default-group
      enable-auto-commit: false
      auto-offset-reset: earliest
      properties:
        partition.assignment.strategy:
          org.apache.kafka.clients.consumer.CooperativeStickyAssignor

    # === Listener 容器 ===
    listener:
      type: batch           # single / batch
      ack-mode: manual      # 手动确认
      concurrency: 3        # 并发消费者线程数

    # === Admin ===
    admin:
      auto-create: false
```

## 2.2 KafkaTemplate 配置

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate(
            ProducerFactory<String, String> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configs.put(ProducerConfig.ACKS_CONFIG, "all");
        configs.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        configs.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
        return new DefaultKafkaProducerFactory<>(configs);
    }
}
```

# 3 KafkaTemplate 消息发送

## 3.1 基本发送

```java
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    // ① 异步发送（推荐）
    public void sendOrderAsync(String orderId, String orderJson) {
        CompletableFuture<SendResult<String, String>> future =
            kafkaTemplate.send("orders", orderId, orderJson);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to send: {}", ex.getMessage());
            } else {
                log.info("Sent to {}-{} @ offset {}",
                    result.getRecordMetadata().topic(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            }
        });
    }

    // ② 同步发送
    public void sendOrderSync(String orderId, String orderJson) {
        try {
            SendResult<String, String> result =
                kafkaTemplate.send("orders", orderId, orderJson).get(10, TimeUnit.SECONDS);
            log.info("Sent: {}", result.getRecordMetadata());
        } catch (Exception e) {
            log.error("Send failed", e);
        }
    }

    // ③ 发送到指定分区
    public void sendToPartition(String orderId, String orderJson, int partition) {
        kafkaTemplate.send("orders", partition, orderId, orderJson);
    }

    // ④ 使用 ProducerRecord 发送
    public void sendWithHeaders(String orderId, String orderJson) {
        ProducerRecord<String, String> record =
            new ProducerRecord<>("orders", orderId, orderJson);
        record.headers().add("trace-id", UUID.randomUUID().toString().getBytes());
        record.headers().add("source", "order-service".getBytes());
        kafkaTemplate.send(record);
    }
}
```

## 3.2 事务发送

```java
@Service
public class TransactionalOrderService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    // @Transactional 注解驱动事务
    @Transactional(transactionManager = "kafkaTransactionManager")
    public void sendOrderAndPayment(String orderId) {
        kafkaTemplate.send("orders", orderId, toOrderJson(orderId));
        kafkaTemplate.send("payments", orderId, toPaymentJson(orderId));
        // 如果抛出异常 → 两条消息都不会发送（回滚）
    }

    // 手动事务管理
    public void sendWithManualTx(String orderId) {
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send("orders", orderId, toOrderJson(orderId));
            operations.send("payments", orderId, toPaymentJson(orderId));
            return true;
        });
    }

    @Bean
    public KafkaTransactionManager<String, String> kafkaTransactionManager(
            ProducerFactory<String, String> producerFactory) {
        return new KafkaTransactionManager<>(producerFactory);
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        DefaultKafkaProducerFactory<String, String> factory =
            new DefaultKafkaProducerFactory<>(producerConfigs());
        factory.setTransactionIdPrefix("tx-");
        return factory;
    }
}
```

# 4 @KafkaListener 消息消费

## 4.1 基础监听

```java
@Component
public class OrderConsumer {

    // ① 简单监听
    @KafkaListener(topics = "orders", groupId = "order-group")
    public void onMessage(String message) {
        log.info("Received: {}", message);
    }

    // ② 带 ConsumerRecord 元数据
    @KafkaListener(topics = "orders", groupId = "order-group")
    public void onMessage(ConsumerRecord<String, String> record) {
        log.info("topic={}, partition={}, offset={}, key={}, value={}",
            record.topic(), record.partition(), record.offset(),
            record.key(), record.value());
    }

    // ③ 多 Topic
    @KafkaListener(topics = {"orders", "payments"}, groupId = "finance-group")
    public void onMultiTopics(ConsumerRecord<String, String> record) {
        switch (record.topic()) {
            case "orders": processOrder(record); break;
            case "payments": processPayment(record); break;
        }
    }

    // ④ Topic 正则匹配
    @KafkaListener(topicPattern = "order-.*", groupId = "order-pattern-group")
    public void onPatternTopic(String message) {
        log.info("Pattern matched: {}", message);
    }

    // ⑤ 指定分区
    @KafkaListener(topicPartitions =
        @TopicPartition(topic = "orders", partitions = {"0", "1"}),
        groupId = "order-p0p1-group")
    public void onSpecificPartition(ConsumerRecord<String, String> record) {
        log.info("Partition {}: {}", record.partition(), record.value());
    }
}
```

## 4.2 批量消费

```java
@Configuration
public class BatchConsumerConfig {
    @Bean
    public KafkaListenerContainerFactory<?> batchFactory(
            ConsumerFactory<String, String> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);  // 开启批量监听
        return factory;
    }
}

@Component
public class BatchConsumer {

    // 批量消费
    @KafkaListener(topics = "orders", groupId = "batch-group",
                   containerFactory = "batchFactory")
    public void onBatch(List<ConsumerRecord<String, String>> records) {
        log.info("Received {} records", records.size());
        for (ConsumerRecord<String, String> record : records) {
            process(record);
        }
    }

    // 批量消费 + 手动确认
    @KafkaListener(topics = "orders", groupId = "batch-group",
                   containerFactory = "batchFactory")
    public void onBatchWithAck(List<ConsumerRecord<String, String>> records,
                                Acknowledgment ack) {
        try {
            for (ConsumerRecord<String, String> record : records) {
                process(record);
            }
            ack.acknowledge(); // 手动确认
        } catch (Exception e) {
            log.error("Batch processing failed", e);
        }
    }
}
```

# 5 手动确认（Acknowledgment）

## 5.1 确认模式

```yaml
spring:
  kafka:
    listener:
      ack-mode: manual  # 手动确认
      # 可选值:
      #   RECORD   : 逐条自动确认
      #   BATCH    : 批量自动确认（默认）
      #   TIME     : 定时自动确认
      #   COUNT    : 按数量自动确认
      #   COUNT_TIME: 数量或时间满足自动确认
      #   MANUAL   : 手动调用 acknowledge()
      #   MANUAL_IMMEDIATE: 手动立即确认
```

## 5.2 手动确认示例

```java
@Component
public class ManualAckConsumer {

    // MANUAL: 调用 acknowledge() 后所有之前的 offset 一起提交
    @KafkaListener(topics = "orders", groupId = "manual-ack-group")
    public void onMessage(ConsumerRecord<String, String> record,
                          Acknowledgment ack) {
        try {
            process(record);
            ack.acknowledge(); // 提交 offset
        } catch (Exception e) {
            // 不确认 → 消息会被重新消费
            log.error("Processing failed, will retry", e);
        }
    }

    // MANUAL_IMMEDIATE: 立即提交（不影响同一批其他 offset）
    @KafkaListener(topics = "orders", groupId = "manual-immediate-group")
    public void onMessageImmediate(ConsumerRecord<String, String> record,
                                    Acknowledgment ack) {
        process(record);
        ack.acknowledge(); // 立即提交此条 offset
    }
}
```

# 6 错误处理

## 6.1 ErrorHandler 基础配置

```java
@Configuration
public class ErrorHandlerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        // 默认错误处理
        CommonErrorHandler errorHandler =
            new DefaultErrorHandler(
                // 死信 Topic 发布策略
                new DeadLetterPublishingRecoverer(kafkaTemplate()),
                // 固定间隔重试
                new FixedBackOff(3000L, 3L)  // 3s 间隔, 重试 3 次
            );
        // 忽略某些异常（不重试）
        errorHandler.addNotRetryableExceptions(IllegalArgumentException.class);

        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

## 6.2 重试与死信队列

```java
@Component
public class RetryableConsumer {

    @KafkaListener(topics = "orders", groupId = "retry-group")
    @RetryableTopic(
        attempts = "4",               // 总共尝试 4 次（1 原始 + 3 重试）
        backoff = @Backoff(
            delay = 3000,              // 初始延迟 3s
            multiplier = 2.0,          // 指数退避: 3s, 6s, 12s
            maxDelay = 15000           // 最大延迟 15s
        ),
        autoCreateTopics = "false",
        dltTopicSuffix = "-dlt",       // 死信 Topic: orders-dlt
        dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    public void onMessage(String message) {
        if (message.contains("error")) {
            throw new RuntimeException("Simulated error");
        }
        process(message);
    }

    // 死信队列消费者（最后一道防线）
    @KafkaListener(topics = "orders-dlt", groupId = "dlt-group")
    public void onDltMessage(ConsumerRecord<String, String> record) {
        log.error("Dead letter: key={}, value={}, offset={}",
            record.key(), record.value(), record.offset());
        // 告警、人工处理、归档
    }
}
```

## 6.3 自定义错误拦截器

```java
@Component
public class CustomErrorHandler implements CommonErrorHandler {

    @Override
    public boolean handleOne(Exception thrownException,
                             ConsumerRecord<?, ?> record,
                             Consumer<?, ?> consumer,
                             MessageListenerContainer container) {
        log.error("Error on record: key={}, value={}, partition={}, offset={}",
            record.key(), record.value(), record.partition(), record.offset());

        // 返回 true → 消息被"消耗"（不重试，offset 继续推进）
        // 返回 false → 消息被重新投递（根据 SeekUtils 策略）
        return false;
    }
}
```

# 7 事务集成

## 7.1 Kafka + 数据库事务（ChainedTransactionManager）

```java
@Configuration
public class TransactionConfig {

    @Bean
    public KafkaTransactionManager<String, String> kafkaTransactionManager(
            ProducerFactory<String, String> producerFactory) {
        return new KafkaTransactionManager<>(producerFactory);
    }

    @Bean
    @Primary
    public ChainedKafkaTransactionManager<String, String> chainedTransactionManager(
            KafkaTransactionManager<String, String> kafkaTm,
            DataSourceTransactionManager dbTm) {
        // 注意: ChainedKafkaTransactionManager 在 3.0 中已废弃
        // 替代: 使用 @Transactional + KafkaTemplate.sendOffsetsToTransaction
        return new ChainedKafkaTransactionManager<>(kafkaTm, dbTm);
    }
}

@Service
public class OrderServiceWithTx {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    // Spring Boot 2.5+ / 3.x 推荐做法:
    @KafkaListener(topics = "input", groupId = "eos-group")
    @Transactional  // DB 事务
    public void consumeAndProduce(ConsumerRecord<String, String> record) {
        // 1. 写入数据库
        jdbcTemplate.update("INSERT INTO orders VALUES (?, ?)",
            record.key(), record.value());

        // 2. 发送消息（Kafka 事务）
        kafkaTemplate.send("output", record.key(), process(record.value()));

        // 3. Kafka offset 在事务提交时自动同步（Spring Kafka 3.x）
        //     需要配置:
        //     spring.kafka.producer.transaction-id-prefix=tx-
        //     spring.kafka.consumer.isolation-level=read_committed
    }
}
```

# 8 并发控制

## 8.1 Listener 容器并发

```java
@Configuration
public class ConcurrencyConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        // 并发度 = Consumer 线程数
        factory.setConcurrency(3);
        // ≤ Topic 分区数（超过的部分闲置）

        // 自动启动
        factory.setAutoStartup(true);

        return factory;
    }
}
```

## 8.2 动态启停监听器

```java
@Service
public class ListenerManager {

    @Autowired
    private KafkaListenerEndpointRegistry registry;

    // 暂停所有监听器
    public void pauseAll() {
        registry.getListenerContainers().forEach(container -> {
            container.pause();
            log.info("Paused: {}", container.getListenerId());
        });
    }

    // 恢复所有监听器
    public void resumeAll() {
        registry.getListenerContainers().forEach(container -> {
            container.resume();
            log.info("Resumed: {}", container.getListenerId());
        });
    }

    // 启停指定监听器
    public void toggleListener(String listenerId, boolean start) {
        MessageListenerContainer container =
            registry.getListenerContainer(listenerId);
        if (container != null) {
            if (start) container.start();
            else container.stop();
        }
    }
}
```

# 9 AdminClient 集成

```java
@Component
public class KafkaAdminService {

    @Autowired
    private KafkaAdmin kafkaAdmin;

    // Spring Boot 自动创建 AdminClient
    // 也可手动创建:
    // @Bean
    // public KafkaAdmin kafkaAdmin() {
    //     Map<String, Object> configs = new HashMap<>();
    //     configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    //     return new KafkaAdmin(configs);
    // }

    // 编程式创建 Topic
    @Bean
    public NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
                .partitions(6)
                .replicas(3)
                .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
                .config(TopicConfig.RETENTION_MS_CONFIG, "604800000")
                .build();
    }

    // 编程式操作
    public void adminOperations() {
        try (AdminClient adminClient = AdminClient.create(
                kafkaAdmin.getConfigurationProperties())) {

            // 列出 Topic
            adminClient.listTopics().names().get()
                .forEach(System.out::println);

            // 查看 Consumer Group
            adminClient.describeConsumerGroups(
                Collections.singletonList("order-group"))
                .all().get()
                .forEach((id, desc) ->
                    log.info("Group {} members: {}", id, desc.members().size()));

            // 查看 Topic
            adminClient.describeTopics(
                Collections.singletonList("orders"))
                .allTopicNames().get()
                .forEach((name, desc) ->
                    log.info("Topic {}: {} partitions, isInternal={}",
                        name, desc.partitions().size(), desc.isInternal()));
        } catch (Exception e) {
            log.error("Admin operation failed", e);
        }
    }
}
```

# 10 完整示例：Spring Boot Kafka 应用

```java
// ========== application.yml ==========
/*
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      transaction-id-prefix: tx-
      enable-idempotence: true
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      group-id: app-group
      enable-auto-commit: false
      isolation-level: read_committed
      properties:
        spring.json.trusted.packages: "com.example.dto"
    listener:
      ack-mode: manual
*/
```

```java
// ========== 消息实体 ==========
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderEvent {
    private String orderId;
    private String eventType;  // CREATED, PAID, SHIPPED
    private BigDecimal amount;
    private Instant timestamp;
}

// ========== Producer ==========
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(String orderId, BigDecimal amount) {
        OrderEvent event = new OrderEvent(orderId, "CREATED", amount, Instant.now());
        kafkaTemplate.send("order-events", orderId, event);
    }
}

// ========== Consumer ==========
@Component
public class OrderEventConsumer {

    private final ConcurrentHashMap<String, String> processedIds = new ConcurrentHashMap<>();

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 2000, multiplier = 2.0)
    )
    public void onOrderEvent(ConsumerRecord<String, OrderEvent> record,
                             Acknowledgment ack) {
        OrderEvent event = record.value();

        // 幂等去重
        if (processedIds.putIfAbsent(event.getOrderId() + event.getEventType(),
                record.value().toString()) != null) {
            log.warn("Duplicate event: {}", event.getOrderId());
            ack.acknowledge();
            return;
        }

        try {
            processEvent(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process: {}", event.getOrderId(), e);
            throw e; // 触发重试
        }
    }

    private void processEvent(OrderEvent event) {
        switch (event.getEventType()) {
            case "CREATED" -> orderService.createOrder(event);
            case "PAID"    -> orderService.markAsPaid(event);
            case "SHIPPED" -> orderService.markAsShipped(event);
        }
    }
}
```
