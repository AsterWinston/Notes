
# 1. 监控体系总览

```
监控层次：
┌──────────────────────────────────────────────┐
│ 应用层    │ 消费延迟、消息堆积、消费成功率    │
│ Broker层  │ TPS、存储容量、磁盘 I/O           │
│ 系统层    │ CPU、内存、网络、JVM GC            │
│ 日志      │ 错误日志、死信消息、事务超时       │
└──────────────────────────────────────────────┘
```

## 1.1 监控手段对比

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| **Dashboard** | RocketMQ 提供 Web 管理界面 | 日常运维、问题排查 |
| **Admin CLI** | `mqadmin` 命令行工具 | 脚本自动化、快速诊断 |
| **Prometheus + Grafana** | 导出 Metrics，Grafana 可视化 | 生产环境持续监控 |
| **Broker Stats** | Broker 内部统计信息 | 性能分析 |
| **应用日志** | Producer/Consumer 日志 | 问题定位 |

# 2. RocketMQ Dashboard

## 2.1 核心功能

```
驾驶舱 (Dashboard)
  ├── Broker 数量
  ├── Topic 数量
  ├── Consumer 数量
  └── 今日消息量

集群 (Cluster)
  ├── Broker 列表 (地址、版本、启动时间)
  ├── Master/Slave 状态
  └── 配置信息

Topic
  ├── 状态（路由信息）
  ├── 消息轨迹（按 Key/MessageId 查询）
  └── 消费者组列表

Consumer
  ├── 消费状态 (TPS、延迟)
  ├── 订阅关系
  └── 消费客户端列表

消息 (Message)
  ├── 按 Topic + 时间 查询
  ├── 按 MessageId 查询
  ├── 按 Key 查询
  └── 消息详情（内容、轨迹）
```

## 2.2 关键监控指标

| 指标 | 含义 | 告警建议 |
|------|------|----------|
| TPS (写入/消费) | 每秒消息条数 | 突降 > 50% |
| 消息堆积 (DIFF) | 未消费的消息数 | > 阈值（如 10万） |
| 消费延迟 | 最后一条消费消息的时间距今 | > 5min |
| 死信队列消息数 | %DLQ% 下的消息数 | > 0 |
| CommitLog 磁盘使用率 | 存储占用百分比 | > 80% |
| Broker 存活状态 | 心跳正常 | 离线 |

# 3. Admin CLI (mqadmin)

## 3.1 常用命令

```bash
# 查看集群信息
sh bin/mqadmin clusterList -n 127.0.0.1:9876

# 查看 Topic 列表及状态
sh bin/mqadmin topicList -n 127.0.0.1:9876
sh bin/mqadmin topicStatus -n 127.0.0.1:9876 -t TopicTest

# 查看 Consumer Group
sh bin/mqadmin consumerProgress -n 127.0.0.1:9876 -g consumer_group

# 查看 Broker 状态
sh bin/mqadmin brokerStatus -n 127.0.0.1:9876 -b broker-a

# 发送测试消息
sh bin/mqadmin sendMessage -n 127.0.0.1:9876 -t TopicTest -p "Hello"

# 查询消息
sh bin/mqadmin queryMsgById -n 127.0.0.1:9876 -i 0A6F2B08070D33B2421800000000

# 重置消费进度（从头消费）
sh bin/mqadmin resetOffsetByTime -n 127.0.0.1:9876 -g group -t TopicTest -s now

# 更新 Broker 配置
sh bin/mqadmin updateBrokerConfig -n 127.0.0.1:9876 -b broker-a -k key -v value
```

## 3.2 consumerProgress 输出解读

```
# 查看消费进度
sh bin/mqadmin consumerProgress -n 127.0.0.1:9876 -g order_consumer_group

ConsumerGroup: order_consumer_group
Topic:        TopicTest
┌────────────┬─────────────┬─────────────┬─────────────┬──────────┐
│ Broker     │ QueueId     │ MinOffset   │ MaxOffset   │ DIFF     │
├────────────┼─────────────┼─────────────┼─────────────┼──────────┤
│ broker-a   │ 0           │ 100         │ 500         │ 400      │
│ broker-a   │ 1           │ 200         │ 550         │ 350      │
│ broker-a   │ 2           │ 150         │ 520         │ 370      │
│ broker-a   │ 3           │ 180         │ 530         │ 350      │
└────────────┴─────────────┴─────────────┴─────────────┴──────────┘
Total DIFF: 1470  ←─ 核心指标：未消费的消息数
```

# 4. Prometheus + Grafana 监控

## 4.1 暴露 Metrics

RocketMQ 5.0 原生支持 Prometheus 格式：

```properties
# broker.conf
metricsExporterType = prometheus
exporterPromHost = 0.0.0.0
exporterPromPort = 5557
```

对于 RocketMQ 4.x，可通过 **RocketMQ Exporter**：

```bash
docker run -d \
  -p 5557:5557 \
  -e JAVA_OPTS="-Drocketmq.namesrv.addr=127.0.0.1:9876" \
  apacherocketmq/rocketmq-exporter:latest
```

## 4.2 核心 Prometheus Metrics

```promql
# Broker TPS
rocketmq_broker_tps_total{broker="broker-a"}

# 消息写入量 (字节)
rocketmq_broker_put_size_total{broker="broker-a"}

# 消费延迟
rocketmq_consumer_offset_diff{group="order_consumer_group", topic="TopicTest"}

# 死信消息数
rocketmq_dlq_count{group="order_consumer_group"}

# JVM 堆内存
jvm_memory_used_bytes{area="heap"}

# CommitLog 磁盘使用率
rocketmq_broker_commitlog_diskutil
```

## 4.3 Grafana Dashboard

推荐使用社区 `RocketMQ Dashboard` (ID: 10477)：
- 导入后可直接查看 Broker TPS、消费延迟、存储容量等

# 5. 消息轨迹 (Message Trace)

## 5.1 启用消息轨迹

```java
// Producer 端
producer.setEnableMsgTrace(true);
producer.setCustomizedTraceTopic("MSG_TRACE_TOPIC");

// Consumer 端
consumer.setEnableMsgTrace(true);
```

## 5.2 轨迹信息

```
消息轨迹包含：
├── Producer 端：
│    ├── 发送时间、发送耗时
│    ├── 消息 ID、Topic、Key
│    └── 状态：成功/失败
├── Broker 端：
│    ├── 存储时间、存储主机
│    └── 消息大小、偏移量
└── Consumer 端：
     ├── 消费时间、消费耗时
     ├── 消费主机、消费组
     └── 状态：成功/失败
```

## 5.3 查询消息轨迹

```bash
# 使用 mqadmin 查询
sh bin/mqadmin queryMsgTraceById -n 127.0.0.1:9876 -i <messageId>
```

# 6. Broker 内部统计

```java
// BrokerStatsManager — 内部统计
// 通过 Admin API 获取

// 获取 Broker 运行时统计
StatsTable stats = adminBroker.getBrokerRuntimeStats();
stat.getStats().get("PUT_TPS");       // 写入 TPS
stat.getStats().get("GET_TPS");       // 读取 TPS
stat.getStats().get("PUT_SIZE");      // 写入字节数
stat.getStats().get("GET_SIZE");      // 读取字节数
```

# 7. 告警策略建议

| 告警项 | 阈值 | 级别 | 处理 |
|--------|------|------|------|
| Broker 宕机 | 心跳缺失 | P0 | 立即处理 |
| 消费堆积 > 100万 | 按业务定 | P1 | 扩容 Consumer |
| 消费延迟 > 10min | 按业务定 | P1 | 检查 Consumer |
| DLQ 消息 > 0 | > 0 | P2 | 排查消费逻辑 |
| CommitLog 磁盘 > 85% | > 85% | P1 | 扩容或清理 |
| TPS 突降 > 50% | > 50% | P1 | 检查网络/应用 |
| JVM GC > 10% | > 10% | P2 | 调优 JVM 参数 |
| NameServer 宕机 | 单节点不可用 | P2 | 仍有冗余 |
| NameServer 全部宕机 | 全部不可用 | P0 | 立即恢复 |

# 8. 参考

- [RocketMQ Dashboard](https://github.com/apache/rocketmq-dashboard)
- [RocketMQ Exporter](https://github.com/apache/rocketmq-exporter)
