
# 1. 概述

Canal 支持将增量数据投递到 RocketMQ，适用于使用 RocketMQ 作为消息中间件的架构体系（尤其是阿里云生态）。

```
┌──────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│  MySQL   │───>│ Canal Server │───>│  RocketMQ    │───>│   Consumers  │
│          │    │ (Producer)   │    │   Cluster    │    │ (下游业务)    │
└──────────┘    └──────────────┘    └─────────────┘    └──────────────┘
```

# 2. 配置

## 2.1 canal.properties

```properties
#################################################
#########    Canal Server 模式    ##############
#################################################
canal.serverMode = rocketMQ

#################################################
#########    RocketMQ 连接配置    ##############
#################################################

# RocketMQ NameServer 地址
rocketmq.namesrv.addr = 127.0.0.1:9876

# RocketMQ 命名空间（阿里云 MQ 需要）
rocketmq.namespace = 

# 访问通道
rocketmq.access.channel = LOCAL

# 生产者组名
rocketmq.producer.group = canal_producer_group

# 是否启用消息轨迹
rocketmq.enable.message.trace = false
rocketmq.customized.trace.topic = 

# 消息 TAG
rocketmq.tag = 

# 批量模式
rocketmq.enable.batch = true

# 超时（毫秒）
rocketmq.send.timeout = 3000

# 重试次数
rocketmq.retry.times.when.send.failed = 3

# VipChannel
rocketmq.vip.channel.enabled = false

# ACL 认证（阿里云 MQ 需要）
rocketmq.access.key = 
rocketmq.secret.key = 
```

## 2.2 Instance 级别配置

```properties
# instance.properties — RocketMQ 相关

# 默认 Topic
canal.mq.topic = canal_example

# 动态 Topic
canal.mq.dynamicTopic = .*\\.user:user_topic,.*\\.order:order_topic

# 消息序列化
canal.mq.flatMessage = true

# 分区数
canal.mq.partitionsNum = 4

# 分区哈希
canal.mq.partitionHash = test\\.user:id
```

# 3. RocketMQ 消息模型

## 3.1 消息 TAG

Canal 自动为消息设置 TAG，便于消费者按条件过滤：

```java
// TAG 格式：destination_schema_table_type
// 例如：example_test_users_INSERT
//       example_test_users_UPDATE
//       example_test_users_DELETE

public class CanalRocketMQTagGenerator {
    
    /**
     * 生成消息 TAG
     */
    public static String generateTag(CanalEntry.Entry entry) {
        String destination = getDestination();
        String schema = entry.getHeader().getSchemaName();
        String table = entry.getHeader().getTableName();
        
        EventType eventType = entry.getHeader().getEventType();
        if (eventType == null) {
            eventType = EventType.QUERY;  // DDL
        }
        
        return String.format("%s_%s_%s_%s", 
            destination, schema, table, eventType.name());
    }
}
```

## 3.2 消费者 TAG 过滤

```java
// 消费者：只订阅 INSERT 类型消息
consumer.subscribe("canal_example", 
    "example_test_users_INSERT || example_test_users_UPDATE");

// 消费者：只订阅 user 表的所有操作
consumer.subscribe("canal_example", 
    "example_test_users_*");
```

# 4. Canal RocketMQ Producer

```java
public class CanalRocketMQProducer implements CanalMQProducer {
    
    private DefaultMQProducer producer;
    
    @Override
    public void init(Properties properties) {
        
        String nameSrvAddr = properties.getProperty("rocketmq.namesrv.addr");
        String producerGroup = properties.getProperty("rocketmq.producer.group");
        
        producer = new DefaultMQProducer(producerGroup);
        producer.setNamesrvAddr(nameSrvAddr);
        
        // 批量发送支持
        boolean enableBatch = Boolean.parseBoolean(
            properties.getProperty("rocketmq.enable.batch", "true"));
        
        // 发送超时
        int sendTimeout = Integer.parseInt(
            properties.getProperty("rocketmq.send.timeout", "3000"));
        producer.setSendMsgTimeout(sendTimeout);
        
        // 重试次数
        int retryTimes = Integer.parseInt(
            properties.getProperty("rocketmq.retry.times.when.send.failed", "3"));
        producer.setRetryTimesWhenSendFailed(retryTimes);
        
        // 命名空间
        String namespace = properties.getProperty("rocketmq.namespace");
        if (StringUtils.isNotEmpty(namespace)) {
            producer.setNamespace(namespace);
        }
        
        // ACL 认证
        String accessKey = properties.getProperty("rocketmq.access.key");
        String secretKey = properties.getProperty("rocketmq.secret.key");
        if (StringUtils.isNotEmpty(accessKey)) {
            AclClientRPCHook aclHook = new AclClientRPCHook(
                new SessionCredentials(accessKey, secretKey));
            producer.getDefaultMQProducerImpl().registerRPCHook(aclHook);
        }
        
        try {
            producer.start();
        } catch (MQClientException e) {
            throw new RuntimeException("RocketMQ producer start failed", e);
        }
    }
    
    @Override
    public void send(CanalMQData data, Callback callback) {
        
        String topic = resolveTopic(data);
        String tag = resolveTag(data);
        String key = buildMessageKey(data);
        String body = buildMessageBody(data);
        
        Message message = new Message(topic, tag, key, body.getBytes(StandardCharsets.UTF_8));
        
        try {
            SendResult result = producer.send(message);
            
            if (result.getSendStatus() == SendStatus.SEND_OK) {
                callback.onSuccess(null);  // RocketMQ 不返回 metadata
            } else {
                callback.onFailure(new CanalMQException(
                    "send status: " + result.getSendStatus()));
            }
        } catch (Exception e) {
            logger.error("rocketmq send failed", e);
            callback.onFailure(e);
        }
    }
    
    @Override
    public void stop() {
        if (producer != null) {
            producer.shutdown();
        }
    }
}
```

# 5. 批量发送

## 5.1 批量发送配置

```properties
# 启用批量发送模式
rocketmq.enable.batch = true

# 批量大小（消息条数）
canal.mq.batch.size = 100

# 批量发送的最大字节数
# rocketmq.max.message.size = 4194304  # 4MB
```

## 5.2 批量发送实现

```java
public class CanalRocketMQBatchProducer {
    
    private DefaultMQProducer producer;
    private List<Message> batchBuffer = new ArrayList<>();
    private int batchSize = 100;
    
    /**
     * 批量发送
     */
    public void sendBatch(List<CanalMQData> dataList) {
        
        List<Message> messages = new ArrayList<>();
        
        for (CanalMQData data : dataList) {
            String topic = resolveTopic(data);
            String tag = resolveTag(data);
            String body = buildMessageBody(data);
            
            Message message = new Message(topic, tag, body.getBytes(StandardCharsets.UTF_8));
            messages.add(message);
        }
        
        try {
            // RocketMQ 批量发送
            SendResult result = producer.send(messages);
            
            if (result.getSendStatus() == SendStatus.SEND_OK) {
                logger.debug("batch send success: {} messages", messages.size());
            }
        } catch (Exception e) {
            logger.error("batch send failed", e);
            
            // 失败时逐条重试
            for (Message message : messages) {
                try {
                    producer.send(message);
                } catch (Exception ex) {
                    logger.error("single retry failed", ex);
                }
            }
        }
    }
    
    /**
     * 将 List<Message> 拆分为不超过 4MB 的批次
     */
    private List<List<Message>> splitBatch(List<Message> messages) {
        List<List<Message>> batches = new ArrayList<>();
        List<Message> currentBatch = new ArrayList<>();
        int currentSize = 0;
        
        for (Message message : messages) {
            int msgSize = message.getBody().length;
            
            if (currentSize + msgSize > 4 * 1024 * 1024) {  // 4MB
                batches.add(currentBatch);
                currentBatch = new ArrayList<>();
                currentSize = 0;
            }
            
            currentBatch.add(message);
            currentSize += msgSize;
        }
        
        if (!currentBatch.isEmpty()) {
            batches.add(currentBatch);
        }
        
        return batches;
    }
}
```

# 6. 消费者示例

## 6.1 Java Consumer

```java
public class CanalRocketMQConsumer {
    
    public static void main(String[] args) throws Exception {
        
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("canal_consumer_group");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        
        // 订阅 Topic 和 TAG
        consumer.subscribe("canal_example", "*");  // 订阅所有消息
        // consumer.subscribe("canal_example", "example_test_users_INSERT");
        
        // 注册消息监听器
        consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
            
            for (MessageExt msg : msgs) {
                try {
                    String body = new String(msg.getBody(), StandardCharsets.UTF_8);
                    
                    // 解析 FlatMessage
                    FlatMessage flatMessage = JSON.parseObject(body, FlatMessage.class);
                    
                    System.out.printf("Topic=%s Tag=%s Keys=%s%n", 
                        msg.getTopic(), msg.getTags(), msg.getKeys());
                    System.out.printf("db=%s table=%s type=%s%n",
                        flatMessage.getDatabase(), flatMessage.getTable(), 
                        flatMessage.getType());
                    
                    // 业务处理
                    processFlatMessage(flatMessage);
                    
                } catch (Exception e) {
                    logger.error("process message error, msgId={}", msg.getMsgId(), e);
                    // 返回 RECONSUME_LATER 触发重试
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            }
            
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });
        
        consumer.start();
        System.out.println("Canal RocketMQ Consumer started.");
    }
}
```

## 6.2 Spring Cloud Stream 消费

```yaml
# application.yml
spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
        bindings:
          canal-input:
            consumer:
              subscription: canal_example||*
              delay-level-when-next-consume: 0
      bindings:
        canal-input:
          destination: canal_example
          content-type: application/json
          group: canal-consumer-group
```

```java
@SpringBootApplication
@EnableBinding({CanalSink.class})
public class CanalStreamConsumer {
    
    @StreamListener(Sink.INPUT)
    public void handleCanalMessage(String messageJson) {
        
        FlatMessage flatMessage = JSON.parseObject(messageJson, FlatMessage.class);
        
        System.out.printf("Received: %s.%s [%s]%n",
            flatMessage.getDatabase(),
            flatMessage.getTable(),
            flatMessage.getType());
        
        // 业务处理
    }
    
    interface CanalSink {
        @Input("canal-input")
        SubscribableChannel input();
    }
}
```

# 7. 事务消息支持

RocketMQ 的事务消息可以保证 Canal 消息的发送与消费位点的事务一致性：

```java
public class CanalRocketMQTransactionProducer {
    
    private TransactionMQProducer producer;
    
    public void init() {
        producer = new TransactionMQProducer("canal_transaction_group");
        producer.setNamesrvAddr("127.0.0.1:9876");
        
        // 设置事务检查监听器
        producer.setTransactionCheckListener(msg -> {
            // 检查消息的事务状态
            Long transactionId = Long.parseLong(msg.getTransactionId());
            
            // 查询本地事务状态（如查询位点是否已持久化）
            boolean committed = metaManager.isPositionCommitted(transactionId);
            
            if (committed) {
                return LocalTransactionState.COMMIT_MESSAGE;
            } else {
                return LocalTransactionState.UNKNOW;
            }
        });
        
        producer.start();
    }
    
    public void sendInTransaction(CanalMQData data) {
        
        Message message = buildMessage(data);
        
        SendResult result = producer.sendMessageInTransaction(message, 
            // 本地事务执行器
            (msg, arg) -> {
                try {
                    // 1. 持久化 Canal 位点
                    metaManager.updateCursor(data.getPosition());
                    
                    // 2. 返回提交状态
                    return LocalTransactionState.COMMIT_MESSAGE;
                    
                } catch (Exception e) {
                    // 位点持久化失败，回滚消息
                    return LocalTransactionState.ROLLBACK_MESSAGE;
                }
            },
            null  // 附加参数
        );
    }
}
```

# 8. 阿里云 RocketMQ 适配

```properties
# 阿里云 RocketMQ (ONS) 配置
rocketmq.namesrv.addr = http://MQ_INST_xxx.mq-internet-access.mq-internet.aliyuncs.com:80
rocketmq.access.channel = CLOUD
rocketmq.namespace = MQ_INST_xxx%topic_tag_prefix  # 阿里云 MQ 实例 ID
rocketmq.access.key = your_access_key
rocketmq.secret.key = your_secret_key
rocketmq.producer.group = GID_canal_producer
rocketmq.enable.message.trace = true
```

# 9. 消息重试与死信

## 9.1 Canal 端的重试

```java
public class CanalRocketMQRetryHandler {
    
    private static final int MAX_RETRIES = 3;
    
    public void sendWithRetry(CanalMQData data) {
        int retries = 0;
        
        while (retries < MAX_RETRIES) {
            try {
                Message message = buildMessage(data);
                SendResult result = producer.send(message);
                
                if (result.getSendStatus() == SendStatus.SEND_OK) {
                    return;  // 成功
                }
                
                retries++;
                logger.warn("send retry {}/{}: status={}", 
                    retries, MAX_RETRIES, result.getSendStatus());
                
            } catch (Exception e) {
                retries++;
                logger.error("send failed, retry {}/{}", retries, MAX_RETRIES, e);
                
                if (retries >= MAX_RETRIES) {
                    // 重试耗尽 → 记录到失败队列
                    failureQueue.offer(data);
                    throw new MaxRetryException("send exhausted after " + MAX_RETRIES, e);
                }
            }
        }
    }
}
```

## 9.2 消费者死信处理

RocketMQ 消费重试机制：
- 消费失败返回 `RECONSUME_LATER` 后，消息进入重试队列
- 重试间隔递增：10s → 30s → 1min → 2min → 3min → ...
- 重试 16 次后进入死信队列（DLQ）

```java
// 死信队列命名：%DLQ%consumerGroup
// 消费者可单独订阅死信队列进行人工处理
consumer.subscribe("%DLQ%canal_consumer_group", "*");
```

# 10. Kafka vs RocketMQ 对比

| 维度 | Kafka | RocketMQ |
|------|-------|----------|
| 协议 | 自有协议 | 自有协议 |
| 分区模型 | Partition | MessageQueue |
| 消费方式 | 拉模式 | 拉+推模式 |
| TAG过滤 | 不支持原生 TAG | 支持 TAG/SQL92 过滤 |
| 事务消息 | Kafka 0.11+ | 原生支持 |
| 消息顺序 | 分区内有序 | 队列内有序 |
| 延迟消息 | 不支持 | 支持（18个等级） |
| 死信队列 | 需自建 | 内置 %DLQ% |
| 运维复杂度 | 较高 | 中等 |
| 阿里云托管 | 有 | 有（RocketMQ 原生） |
| Canal 集成 | 成熟 | 成熟 |
