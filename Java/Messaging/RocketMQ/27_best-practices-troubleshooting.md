
# 1. 消息可靠性最佳实践

## 1.1 发送端保证

```
端到端可靠性链：
Producer → Broker → Consumer

┌──────────────────────────────────────────────┐
│ 发送端保证：                                  │
│ 1. 使用同步发送 + 判断 SendResult             │
│ 2. 开启重试 retryTimesWhenSendFailed ≥ 2      │
│ 3. 开启换 Broker 重试                         │
│ 4. 业务侧记录 sendResult (msgId/offset)       │
│ 5. 落本地 DB 后再发消息 → 事务消息            │
└──────────────────────────────────────────────┘
```

```java
// 推荐的发送代码模板
public boolean sendWithRetry(String topic, String body, String key) {
    try {
        Message msg = new Message(topic, "TagA", key, body.getBytes());
        msg.setKeys(key);  // 设置业务 Key，方便追踪
        
        SendResult result = producer.send(msg, 3000);  // 同步发送，超时3s
        
        if (result.getSendStatus() == SendStatus.SEND_OK) {
            // 记录发送结果到 DB/日志
            log.info("发送成功: msgId={}, offset={}", result.getMsgId(), result.getQueueOffset());
            return true;
        }
        // 处理其他状态
        // FLUSH_DISK_TIMEOUT / FLUSH_SLAVE_TIMEOUT → 根据业务决定
        return false;
    } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {
        log.error("发送失败: topic={}, key={}", topic, key, e);
        // 落 DB 补偿表，异步重试
        saveToRetryTable(topic, body, key);
        return false;
    }
}
```

## 1.2 消费端保证

```
┌──────────────────────────────────────────────┐
│ 消费端保证：                                  │
│ 1. 消费幂等 (去重)                            │
│ 2. 异常不吞，返回 RECONSUME_LATER             │
│ 3. 死信队列监控 + 人工处理                    │
│ 4. 先落业务 DB，再返回 SUCCESS                │
│ 5. Ultra：try-catch + 业务回滚               │
└──────────────────────────────────────────────┘
```

# 2. 架构设计最佳实践

## 2.1 Topic 规划原则

```
1. 一个业务域一个 Topic
   ✗ orderTopic, userTopic, paymentTopic → 混在一个 Topic
   ✓ order-create-topic, order-pay-topic, user-register-topic

2. Tag 用于子类型分类
   Topic: order-topic
   ├── Tag: ORDER_CREATED
   ├── Tag: ORDER_PAID
   ├── Tag: ORDER_SHIPPED
   └── Tag: ORDER_COMPLETED

3. Keys 设置业务唯一标识
   msg.setKeys("orderId_" + orderId)

4. Queue 数量 ≥ Consumer 最大实例数
   (方便后续水平扩容)

5. 为 DLQ / RETRY / TRACE 预留 Topic
```

## 2.2 Consumer 设计原则

```
原则 1: 拆分粒度
  ✗ 一个 Consumer 消费所有业务
  ✓ 每个业务有独立的 Consumer Group

原则 2: 消息过滤在 Broker 端
  ✗ 订阅全部 + Consumer 端 filter
  ✓ 使用 Tag 或 SQL92 在 Broker 端过滤

原则 3: 水平扩展
  新加 Consumer 实例 → 自动 Rebalance → 加速消费

原则 4: 处理顺序
  先写 DB/Redis → 再返回 SUCCESS
  
原则 5: 幂等性
  所有 Consumer 必须幂等
```

## 2.3 分布式事务设计

```
使用事务消息处理跨服务数据一致性：

订单服务:
  1. 发送 Half Message (订单创建事件)
  2. 执行本地事务 (INSERT 订单表)
  3. 提交 Commit / Rollback

库存服务:
  → 消费 订单创建事件
  → 扣减库存
  → 发送 "库存扣减成功" 事件

通知服务:
  → 消费 订单创建事件
  → 发送通知
```

# 3. 避坑指南

## 3.1 Producer 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `No route info of this topic` | Topic 不存在或 Broker 未注册 | `autoCreateTopicEnable=true` 或手动创建 |
| `connect to null` | NameServer 不可达 | 检查防火墙/IP/端口 |
| 消息丢失 | Oneway 发送 + Broker 宕机 | 改用同步发送 + 重试 |
| 发送缓慢 | 同步发送阻塞、Broker 负载高 | 更换异步发送、扩容 Broker |
| `MQClientException: Send [xxx] times` | 多次重试均失败 | 检查 Broker 存活、网络 |

## 3.2 Consumer 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 消费堆积 | 消费速度 < 生产速度 | 增加 Consumer 实例、优化消费逻辑 |
| 某个队列不消费 | Rebalance 不均 | 检查 Queue 数与实例数配比 |
| 重复消费 | 重试 + Rebalance 导致 | 实现消费幂等 |
| 消费慢 | 消费逻辑阻塞 | 优化业务处理、提高线程数 |
| 不消费 | Topic 或 Tag 不匹配、Group 冲突 | 检查 subscribe 表达式 |
| `subscription not latest` | 订阅关系不一致 | 统一 Consumer Group 内订阅 |

## 3.3 Broker 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 磁盘写满 | 消费堆积或保留时间过长 | 扩容磁盘、调整 `fileReservedTime` |
| OOM | JVM 内存不够 | 增大 `-Xmx`、限制 `maxMessageSize` |
| CPU 高 | 刷盘/复制频次过高 | 降低 `flushInterval`、优化参数 |
| 主从不一致 | 异步复制延迟大 | 同步复制、DLedger |
| Master 宕机后不可写 | 未开启自动选主 | DLedger 或人工切换 |

# 4. 故障排查流程

## 4.1 消息发送失败排查

```
1. 检查 NameServer 是否可达
   telnet {ns-ip} 9876

2. 检查 Broker 是否存活
   sh bin/mqadmin brokerStatus -n {ns-ip}:9876 -b {broker-name}

3. 检查 Topic 路由信息
   sh bin/mqadmin topicStatus -n {ns-ip}:9876 -t {topic}

4. 检查客户端配置
   - namesrvAddr 是否正确
   - Producer Group 名称是否冲突
   - 版本是否匹配

5. 查看客户端日志
   tail -f ~/logs/rocketmqlogs/rocketmq_client.log
```

## 4.2 消费堆积排查

```
1. 查看消费堆积量
   sh bin/mqadmin consumerProgress -n {ns-ip}:9876 -g {group}

2. 检查消费者是否在线
   sh bin/mqadmin consumerStatus -n {ns-ip}:9876 -g {group}

3. 检查消费逻辑是否有阻塞
   - 卡在数据库查询
   - 外部 HTTP 调用超时
   - 异常导致频繁重试

4. 检查 Rebalance 是否正常
   - 实例数 > Queue 数 → 部分实例空闲
   - 实例频繁上下线 → Rebalance 震荡

5. 紧急处理
   - 增加 Consumer 实例
   - 临时切换消费逻辑（跳过/简化处理）
   - 手动重置消费进度 (如果允许跳过)
```

## 4.3 消息丢失排查

```
1. Producer 端
   - sendResult 是否判断成功
   - 是否使用 Oneway 发送
   - 是否有本地补偿表

2. Broker 端
   - 刷盘策略 (ASYNC_FLUSH 可能丢未刷盘数据)
   - 主从复制策略 (ASYNC_MASTER 可能丢未同步数据)
   - 文件是否被误删

3. Consumer 端
   - 是否吞掉了异常
   - 是否返回了 RECONSUME_LATER 后未重试
   - 是否消费成功后未正确落业务数据

4. 排查工具
   - 按 Key 查询消息
   - 按 MessageId 查询消息
   - 消息轨迹追踪
```

# 5. 生产环境检查清单

```
部署前检查：
□ Namesrv 至少 2 台（高可用）
□ Broker Master + Slave（主从部署）
□ 生产环境建议 DLedger（自动故障转移）
□ 开启 ACL 访问控制
□ 配置合适的 JVM 参数
□ 开启 GC 日志
□ 开启消息轨迹 (可选)
□ 配置 Prometheus + Grafana 监控
□ 配置告警规则（消费堆积、Broker 宕机）
□ 配置日志清理策略（防止磁盘耗尽）
□ 文档化 Topic、Group 使用规范
□ 定期备份配置文件
```

# 6. 参考

- [RocketMQ 最佳实践](https://rocketmq.apache.org/docs/best-practice/)
- [RocketMQ FAQ](https://rocketmq.apache.org/docs/faq/)
