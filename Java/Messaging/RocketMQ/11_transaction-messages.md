
# 1. 事务消息概述

RocketMQ 的事务消息用于解决**分布式事务最终一致性**问题，基于**两阶段提交 + 事务回查**机制实现。

```
典型场景：电商下单
下单服务 → Producer
├── 发送事务消息（订单创建事件）
├── 执行本地事务（写入订单表）
└── 提交/回滚 → 消息对下游可见/不可见

下游：库存服务、积分服务、通知服务 → Consumer
```

## 1.1 核心角色

| 角色 | 说明 |
|------|------|
| **Half Message (半消息)** | 暂不可见的消息，等待事务提交/回滚 |
| **TransactionStatus** | Commit / Rollback / Unknown |
| **TransactionListener** | 执行本地事务 + 回查 |
| **TransactionalMessageService** | Broker 端事务消息管理 |

# 2. 事务消息流程

```
Producer                    Broker                        Consumer
    │                         │                              │
    │  1.发送 Half Message    │                              │
    │ ──────────────────────→ │                              │
    │                         │  (存储但不可见)               │
    │  2.响应 (Half OK)      │                              │
    │ ←────────────────────── │                              │
    │                         │                              │
    │  3.执行本地事务         │                              │
    │  ┌─────────────────┐    │                              │
    │  │ INSERT 订单表    │    │                              │
    │  │ (本地事务)       │    │                              │
    │  └─────────────────┘    │                              │
    │                         │                              │
    │  4.提交 Commit/Rollback │                              │
    │ ──────────────────────→ │                              │
    │                         │                              │
    │                         │  Commit: 消息对 Consumer 可见  │
    │                         │ ─────────────────────────────→│
    │                         │  Rollback: 逻辑删除            │
```

## 2.1 超时回查

```
如果 Producer 在步骤4之前宕机：

Broker 定时扫描 Half Message（超时未提交）
    │
    ├── 60s 超时 → 发起回查
    │
    ▼
Producer 回查 (TransactionListener.checkLocalTransaction())
    │
    ├── 本地事务已提交 → 返回 COMMIT
    ├── 本地事务已回滚 → 返回 ROLLBACK
    └── 状态未知       → 返回 UNKNOWN（等待下次回查）
```

# 3. 代码实现

## 3.1 TransactionListener 实现

```java
public class OrderTransactionListener implements TransactionListener {

    // 执行本地事务（半消息发送成功后的回调）
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 执行业务操作 (如：写入订单表、扣减库存)
            Long orderId = (Long) arg;
            orderService.createOrder(orderId);
            System.out.println("本地事务执行成功");
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            System.err.println("本地事务执行失败: " + e.getMessage());
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    // Broker 回查回调（本地事务状态超时未确认时触发）
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        String orderId = msg.getKeys();
        try {
            // 查询本地事务状态（如：检查订单表是否写入成功）
            Order order = orderService.getOrder(Long.valueOf(orderId));
            if (order != null) {
                System.out.println("回查: 订单已创建, 提交消息");
                return LocalTransactionState.COMMIT_MESSAGE;
            }
            System.out.println("回查: 订单不存在, 回滚消息");
            return LocalTransactionState.ROLLBACK_MESSAGE;
        } catch (Exception e) {
            System.err.println("回查异常: " + e.getMessage());
            return LocalTransactionState.UNKNOW; // 下次再回查
        }
    }
}
```

## 3.2 TransactionMQProducer 使用

```java
public class TransactionProducer {

    public static void main(String[] args) throws Exception {
        // 1. 创建事务消息 Producer
        TransactionMQProducer producer = 
            new TransactionMQProducer("transaction_group");
        producer.setNamesrvAddr("127.0.0.1:9876");

        // 2. 设置事务监听器
        producer.setTransactionListener(new OrderTransactionListener());

        // 3. 设置线程池（用于回查）
        ExecutorService executorService = new ThreadPoolExecutor(
            2, 5, 100, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(2000),
            new ThreadFactoryBuilder().setNameFormat("transaction-check-%d").build()
        );
        producer.setExecutorService(executorService);

        // 4. 启动 Producer
        producer.start();

        // 5. 发送事务消息
        for (int i = 0; i < 10; i++) {
            Message msg = new Message("TransactionTopic", "TagA",
                ("Transaction Msg " + i).getBytes());
            msg.setKeys("order_" + i);  // 设置 Key 方便回查定位

            TransactionSendResult result = producer.sendMessageInTransaction(msg, (long) i);
            System.out.printf("发送结果: %s%n", result);
        }

        // 保持运行，等待回查
        Thread.sleep(60000);
        producer.shutdown();
    }
}
```

# 4. Broker 端事务消息存储

## 4.1 CommitLog 中的事务标记

```java
// MessageSysFlag 中的事务标记位
public static final int TRANSACTION_NOT_TYPE         = 0;  // 非事务消息
public static final int TRANSACTION_PREPARED_TYPE   = 0x1 << 2;  // Half Message
public static final int TRANSACTION_COMMIT_TYPE     = 0x2 << 2;  // 已提交
public static final int TRANSACTION_ROLLBACK_TYPE   = 0x3 << 2;  // 已回滚
```

## 4.2 事务消息存储流程

```
1. Half Message → CommitLog (标记 TRANSACTION_PREPARED_TYPE)
                  → CommitLog 中存储 preparedTransactionOffset

2. OP Message   → CommitLog (标记 COMMIT_TYPE / ROLLBACK_TYPE)
                  → CommitLog 中存储 queueOffset (指向 Half Message)

3. 回查时:
   - 扫描 Half Message，超时未找到 OP Message → 回查
   - 找到 OP Message → 更新 Half Message 状态
```

## 4.3 事务回查扫描

```java
// TransactionalMessageCheckService (Broker 启动后运行的定时服务)
public class TransactionalMessageCheckService extends ServiceThread {
    @Override
    public void run() {
        // 定期检查超时的 Half Message（超时时间可配置）
        long timeout = brokerConfig.getTransactionTimeOut();
        int checkMax = brokerConfig.getTransactionCheckMax();

        // 遍历 CommitLog，查找超时 Half Message
        // 过滤条件：PREPARED && 无对应的 OP
        // 发起回查 RPC
    }
}
```

# 5. 事务消息对比

| 维度 | RocketMQ 事务消息 | Kafka 事务 | Seata MQ 模式 |
|------|------------------|------------|---------------|
| 实现方式 | Half Message + 回查 | 幂等 + 事务生产者 | AT/TCC + MQ 集成 |
| 侵入性 | 低（仅 Producer 端） | 低 | 中（需引入 Seata） |
| 回查机制 | Broker 主动回查 | 无 | 依赖 TM |
| 隔离性 | 未 COMMIT 前不可见 | 隔离性由消费者保证 | 由 Seata 控制 |
| 适用场景 | 异步最终一致 | 流处理 | 强一致 + MQ |

# 6. 关键配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `transactionTimeOut` | 6000ms | Half Message 超时时间（超时后可回查） |
| `transactionCheckMax` | 15 | 最大回查次数 |
| `transactionCheckInterval` | 60000ms | 回查间隔 |
| `checkTransactionMessageEnable` | true | 是否启用事务检查服务 |

# 7. 最佳实践

1. **checkLocalTransaction 必须幂等**：可能被多次回查
2. **回查超时次数限制**：防止无限的 UNKNOW
3. **本地事务状态存储**：回查时可直接查 DB 判断状态
4. **业务标识**：使用 `msg.setKeys()` 存储业务唯一 ID，方便回查时定位
5. **线程池隔离**：回查使用独立线程池，避免阻塞主业务

# 8. 参考

- [RocketMQ 事务消息](https://rocketmq.apache.org/docs/4.x/producer/05message3/)
