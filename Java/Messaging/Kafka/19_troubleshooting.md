# 1 消费 Lag 过大

## 1.1 症状

- `kafka-consumer-groups.sh --describe` 显示 LAG 持续增长
- 监控告警 `records-lag-max` 超过阈值
- 业务数据延迟增大

## 1.2 排查流程

```
Step 1: 判断 Lag 增长速度
  → 如果 Lag 匀速增长（Consumer 正常只是跟不上）
      → 扩大 Consumer 数量或增加分区
  → 如果 Lag 突然暴增（Consumer 停止消费）
      → 检查 Consumer 日志和状态

Step 2: 检查 Consumer 状态
  kafka-consumer-groups.sh --describe --group my-group
  → 检查 Consumer 成员是否在线
  → 每个 Partition 是否都有 Consumer 在消费

Step 3: 检查 Consumer 日志
  → 是否有频繁 Rebalance
  → 是否有长时间 GC
  → 是否有异常抛出

Step 4: 检查下游系统
  → 下游数据库是否响应慢
  → 是否有锁竞争
```

## 1.3 常见原因与解决

| 原因 | 诊断 | 解决方案 |
|------|------|---------|
| 消费速度 < 生产速度 | Lag 匀速增长 | 增加分区+Consumer, 优化消费逻辑 |
| Consumer 频繁 Rebalance | 日志中 JoinGroup 频繁 | 增大 session.timeout.ms, max.poll.interval.ms |
| 消息处理慢 | poll 日志显示处理时间长 | 异步处理, 增大 max.poll.records |
| Consumer 挂掉 | 成员列表缺失 | 重启 Consumer, 检查 OOM |
| 数据倾斜 | 某些 Partition Lag 特别大 | 重新设计消息 Key, 增加分区 |
| 下游系统瓶颈 | 处理逻辑中 DB 超时 | 批量化写入, 连接池优化 |

# 2 Rebalance 风暴

## 2.1 症状

- Consumer 日志频繁打印 `(Re-)joining group`
- 消费时断时续
- 整体吞吐下降

## 2.2 常见原因

```
① session.timeout.ms 太小
  heartbeat.interval.ms=3s, session.timeout.ms=10s
  GC 暂停 2s → 心跳延迟 → 可能超时 → Rebalance

② max.poll.interval.ms 太小
  poll() 返回 500 条消息, 每条处理 1s
  500 × 1s = 500s > max.poll.interval.ms=300s → Rebalance

③ Consumer 启动顺序
  不设置 group.initial.rebalance.delay.ms
  多个 Consumer 逐个启动 → 每次加入都触发 Rebalance

④ Consumer 异常退出
  Consumer 未调用 close() → session.timeout.ms 后被发现 → Rebalance
```

## 2.3 解决方案

```java
// ① 合理设置超时
props.put("session.timeout.ms", 60000);      // 从 10s → 60s
props.put("heartbeat.interval.ms", 10000);   // 从 3s → 10s (< session/3)
props.put("max.poll.interval.ms", 600000);   // 从 300s → 600s

// ② 使用 Cooperative Rebalance
props.put("partition.assignment.strategy",
    CooperativeStickyAssignor.class.getName());

// ③ Static Membership（避免重启 Rebalance）
props.put("group.instance.id", "my-consumer-1");

// ④ 延迟首次 Rebalance
props.put("group.initial.rebalance.delay.ms", 10000);
```

# 3 Leader 分布不均衡

## 3.1 症状

- 某些 Broker CPU/网络负载远高于其他
- 经过 Controller 切换后 Leader 都集中到少数 Broker

## 3.2 诊断

```bash
# 检查每个 Broker 上的 Leader 数量
kafka-topics.sh --bootstrap-server localhost:9092 --describe \
    | grep "Leader:" \
    | awk '{print $NF}' \
    | sort | uniq -c

# 输出:
#  120 0    ← Broker 0 有 120 个 Leader（过高）
#    5 1    ← Broker 1 只有 5 个 Leader（过低）
#   35 2
```

## 3.3 解决方案

```bash
# 自动 Preferred Leader 选举
kafka-leader-election.sh --bootstrap-server localhost:9092 \
    --election-type preferred --all-topic-partitions

# 确保配置:
# auto.leader.rebalance.enable=true
# leader.imbalance.per.broker.percentage=10
```

# 4 ISR 收缩与扩展

## 4.1 症状

- `UnderReplicatedPartitions > 0`
- ISR 频繁变化（抖动）
- 部分 Partition 写入受阻（min.insync.replicas 不满足）

## 4.2 诊断

```bash
# 查看当前 ISR 状态
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --under-replicated-partitions

# 查看 Broker 日志
grep "ISR" /var/log/kafka/server.log | tail -100

# 查看远程复制 Lag
kafka.server:type=ReplicaFetcherManager,name=MaxLag,clientId=Replica
```

## 4.3 常见原因与解决

| 原因 | 解决 |
|------|------|
| Broker 宕机 | 恢复 Broker |
| Follower 拉取跟不上 | 增大 `num.replica.fetchers`, 减少复制限速 |
| GC 导致 Follower 暂停 | 优化 JVM 参数 |
| 磁盘 I/O 慢 | 换 SSD, 增加 `log.dirs` 多目录 |
| 网络抖动 | 检查网络, 增大 `replica.lag.time.max.ms` |

```properties
# 调大 ISR 收缩容忍度
replica.lag.time.max.ms=60000  # 30s → 60s
num.replica.fetchers=4         # 1 → 4
```

# 5 磁盘满

## 5.1 症状

- Broker 日志: `No space left on device`
- 无法创建新 Segment
- Kafka 进程可能直接 crash

## 5.2 应急处理

```bash
# ① 紧急降低 retention（动态生效）
kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config retention.ms=7200000 \
    --entity-type topics --entity-name large-topic

# ② 紧急：手动删除旧 Segment（慎用！）
# 直接删除 .log 和 .index 文件
rm -f /data/kafka-logs/large-topic-0/00000000000000000000.*

# ③ 迁移分区到有空间的 Broker
# 使用 kafka-reassign-partitions.sh

# ④ 扩容磁盘
# 添加 log.dirs 新目录
```

## 5.3 预防

```properties
# 设置 Topic 级别的大小限制
retention.bytes=107374182400   # 100GB 上限

# 监控磁盘使用率
# 告警: 磁盘使用率 > 80%
```

# 6 Producer 性能问题

## 6.1 发送延迟高

```bash
# 诊断
kafka.producer:type=producer-metrics,client-id=*
# request-latency-avg, bufferpool-wait-time

# 原因与解决:
# ① batch.size 太小 → 增大到 128KB+
# ② buffer.memory 不够 → 增大到 64MB+
# ③ 网络延迟 → 检查 Broker 网络
# ④ 压缩 CPU 高 → 改为 lz4 或 none
```

## 6.2 消息丢失

```
场景 A: acks=0 → 不等待确认，Broker 未写入
  解决: 改用 acks=1 或 acks=all

场景 B: acks=1, Leader 宕机
  Producer 收到 ACK, 但 Leader 宕机前 Follower 未同步 → 消息丢失
  解决: acks=all + min.insync.replicas=2

场景 C: Producer 缓冲区溢出
  buffer.memory 满 → send() 阻塞 → max.block.ms 超时
  解决: 增大 buffer.memory, 增大 max.block.ms
```

# 7 Consumer 消费问题

## 7.1 消息丢失

```
场景 A: enable.auto.commit=true
  poll() 自动提交了上一批的 offset
  消费者崩溃 → 当前批未处理 → 消息丢失
  解决: enable.auto.commit=false, 手动提交 → At-Least-Once

场景 B: offset 重置到 latest
  auto.offset.reset=latest
  Group 首次启动 → 只消费新消息，历史消息丢失
  解决: 首次使用 earliest, 重复消费后手动处理去重

场景 C: Rebalance 期间丢消息
  onPartitionsRevoked 中未提交 offset
  解决: 在 Revoke 回调中 commitSync
```

## 7.2 消息重复

```
场景 A: 处理成功但提交失败
  解决: 消费端做幂等处理（唯一 Key 去重）

场景 B: Rebalance 导致双消费
  解决: Cooperative Rebalance + onPartitionsRevoked commitSync

场景 C: Producer 重试导致重复
  解决: enable.idempotence=true
```

# 8 Controller 故障

## 8.1 症状

- `ActiveControllerCount = 0`
- 集群无法创建/删除 Topic
- Leader 选举停止

## 8.2 排查

```bash
# ZK 模式: 查看 /controller 节点
zookeeper-shell.sh localhost:2181 get /controller

# KRaft 模式: 查看 quorum 状态
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status

# 检查 Controller Broker 日志
grep "Controller" /var/log/kafka/server.log | tail -50
```

## 8.3 常见原因

| 原因 | 解决 |
|------|------|
| Controller Broker 宕机 | 恢复 Broker 或等待重新选举 |
| ZK Session 过期 | 检查 ZK 连接，增大 session.timeout |
| 元数据加载缓慢 | 减少 Partition 总数，升级到 KRaft |
| 网络分区 | 检查 Broker 间网络连通性 |

# 9 客户端连接问题

## 9.1 Producer 无法连接

```
错误: org.apache.kafka.common.errors.TimeoutException:
      Failed to update metadata after 60000 ms.

排查:
  ① bootstrap.servers 是否正确? → 检查 IP/端口
  ② 网络是否通? → telnet broker1 9092
  ③ Broker 是否在运行? → kafka-broker-api-versions.sh
  ④ 安全配置是否正确? → SASL/SSL 配置
  ⑤ advertised.listeners 是否正确? → 客户端实际连接的地址
```

## 9.2 Consumer 不能消费

```
错误: org.apache.kafka.common.errors.GroupAuthorizationException:
      Not authorized to access group

排查:
  ① 是否有 Group READ 权限? → kafka-acls.sh --list
  ② Group 名称是否有冲突?
  ③ Broker SASL 认证是否通过?
```

# 10 其他常见问题

## 10.1 Topic 无法删除

```bash
# 确保 delete.topic.enable=true
# 确保没有 Consumer 在消费该 Topic
# 查看 /admin/delete_topics ZK 节点（ZK 模式）

kafka-topics.sh --bootstrap-server localhost:9092 \
    --delete --topic my-topic
```

## 10.2 Offset 越界

```
错误: OffsetOutOfRangeException
原因: 日志被清理了（retention），已提交的 offset 不在任何 Segment 中
解决: 重置 offset 到 earliest/latest
  kafka-consumer-groups.sh --reset-offsets --to-earliest --execute
```

## 10.3 日志文件损坏

```bash
# 查看损坏的段文件
kafka-dump-log.sh --files /data/kafka-logs/topic-0/00000000000000000000.log

# 修复: 手动删除损坏的 Segment（数据会丢失）
rm -f /data/kafka-logs/topic-0/00000000000000000000.*
# 或: 如果损坏不严重，Kafka 会自动截断并跳过损坏部分
```

## 10.4 快速诊断脚本

```bash
#!/bin/bash
# kafka-health-check.sh

BOOTSTRAP_SERVER="localhost:9092"

echo "=== Broker 状态 ==="
kafka-broker-api-versions.sh --bootstrap-server $BOOTSTRAP_SERVER 2>&1 | head -5

echo ""
echo "=== Controller 状态 ==="
# ZK 模式
# zookeeper-shell.sh localhost:2181 get /controller 2>&1 | head -5

echo ""
echo "=== Under-Replicated Partitions ==="
kafka-topics.sh --bootstrap-server $BOOTSTRAP_SERVER --describe --under-replicated-partitions

echo ""
echo "=== Consumer Group Lag ==="
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP_SERVER --list 2>&1 | head -10
for group in $(kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP_SERVER --list 2>/dev/null | head -5); do
    kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP_SERVER --describe --group "$group" | grep -E "LAG|$group" | head -15
done

echo ""
echo "=== Offline Partitions ==="
kafka-topics.sh --bootstrap-server $BOOTSTRAP_SERVER --describe --unavailable-partitions
```
