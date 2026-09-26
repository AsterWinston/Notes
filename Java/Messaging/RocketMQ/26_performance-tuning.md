
# 1. 性能调优全景

```
调优目标：
┌────────────────────────────────────────────┐
│ 吞吐 (Throughput)  │ TPS 越高越好          │
│ 延迟 (Latency)     │ P99 < 10ms            │
│ 稳定性 (Stability)  │ 长期高负载不抖动      │
│ 堆积弹性           │ 积压千万消息不影响消费  │
└────────────────────────────────────────────┘

调优层次：
  操作系统 (OS) → JVM → Broker → Producer/Consumer
```

# 2. 操作系统级优化

## 2.1 文件描述符

```bash
# 查看当前限制
ulimit -n

# 永久设置
vim /etc/security/limits.conf
* soft nofile 655360
* hard nofile 655360

# Broker 需要大量 FD (消息文件 + 网络连接)
```

## 2.2 虚拟内存 — vm.max_map_count

```bash
# RocketMQ 使用 MappedByteBuffer (mmap)
# 需要修改 mmap 数量限制
# 临时设置
sysctl -w vm.max_map_count=655360

# 永久设置
echo "vm.max_map_count=655360" >> /etc/sysctl.conf
sysctl -p
```

## 2.3 磁盘调度器

```bash
# 查看当前调度器
cat /sys/block/vdb/queue/scheduler
# [mq-deadline] kyber none

# SSD 推荐 none (noop) 或 mq-deadline
echo none > /sys/block/vdb/queue/scheduler
```

## 2.4 网卡队列与中断

```bash
# 开启网卡多队列
ethtool -L eth0 combined 8

# 检查网卡缓冲
ethtool -g eth0
```

# 3. JVM 调优

## 3.1 Broker JVM 参数

```bash
# runbroker.sh 调整
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC"
JAVA_OPT="${JAVA_OPT} -XX:G1HeapRegionSize=16m"
JAVA_OPT="${JAVA_OPT} -XX:G1ReservePercent=25"
JAVA_OPT="${JAVA_OPT} -XX:InitiatingHeapOccupancyPercent=30"
JAVA_OPT="${JAVA_OPT} -XX:SoftRefLRUPolicyMSPerMB=0"
JAVA_OPT="${JAVA_OPT} -XX:MaxGCPauseMillis=200"
JAVA_OPT="${JAVA_OPT} -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:+PrintGCDateStamps"
JAVA_OPT="${JAVA_OPT} -XX:+PrintGCApplicationStoppedTime"
JAVA_OPT="${JAVA_OPT} -Xloggc:${GC_LOG_DIR}/gc.log"
```

## 3.2 NameServer JVM 参数

```bash
# runserver.sh 调整 (NameServer 内存需求低)
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC"
```

## 3.3 关键 JVM 参数说明

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `-Xms/Xmx` | 相等 | 避免堆内存伸缩带来的 GC |
| `-Xmn` | Xmx/2 | 年轻代大小 |
| `-XX:+UseG1GC` | - | 低延迟 GC，推荐 |
| `-XX:MaxGCPauseMillis` | 200 | 目标 GC 暂停时间 |
| `-XX:InitiatingHeapOccupancyPercent` | 30 | G1 中触发 Mixed GC 的阈值 |
| `-XX:G1HeapRegionSize` | 16m | G1 Region 大小 |
| `-XX:SoftRefLRUPolicyMSPerMB` | 0 | 软引用立即回收 |

# 4. Broker 配置调优

## 4.1 存储相关

```properties
# broker.conf

# ===== 刷盘策略 =====
flushDiskType = ASYNC_FLUSH          # 异步刷盘 (推荐，性能最高)
# flushDiskType = SYNC_FLUSH         # 同步刷盘 (可靠性优先)

# ===== 异步刷盘参数 =====
flushIntervalCommitLog = 500         # 刷盘间隔 (ms)，默认 500
flushLeastPagesWhenWarmMapedFile = 4096  # 最少刷盘页数 4096页≈16MB
commitCommitLogInterval = 200        # Commit 间隔 (ms)

# ===== 文件相关 =====
mapedFileSizeCommitLog = 1073741824  # CommitLog 单文件大小，1GB
mapedFileSizeConsumeQueue = 300000  # ConsumeQueue 单文件大小(条目数)
fileReservedTime = 72               # 文件保留时间 (小时)

# ===== 主从复制 =====
brokerRole = ASYNC_MASTER           # 异步复制 (吞吐优先)
# brokerRole = SYNC_MASTER           # 同步复制 (可靠性优先)

# ===== 线程池 =====
sendMessageThreadPoolNums = 16      # 发送消息线程数 (默认 1)
pullMessageThreadPoolNums = 32      # 拉取消息线程数 (默认 16+brokerId)
queryMessageThreadPoolNums = 16     # 查询消息线程数

# ===== 网络 =====
serverSocketSndBufSize = 131072     # socket 发送缓冲 (128KB)
serverSocketRcvBufSize = 131072     # socket 接收缓冲 (128KB)

# ===== 开启堆积 =====
osPageCacheBusyTimeOutMills = 1000  # PageCache 忙等待超时
useReentrantLockWhenPutMessage = true  # 使用 ReentrantLock (自旋锁，吞吐更高)
```

## 4.2 消息大小限制

```properties
maxMessageSize = 4194304            # 最大消息 4MB (默认)
```

# 5. Producer 调优

```java
DefaultMQProducer producer = new DefaultMQProducer("producer_group");

// ===== 连接 =====
producer.setNamesrvAddr("127.0.0.1:9876");

// ===== 发送参数 =====
producer.setSendMsgTimeout(3000);               // 发送超时 (ms)
producer.setCompressMsgBodyOverHowmuch(4096);   // 压缩阈值 (字节)，4KB+
producer.setRetryTimesWhenSendFailed(2);        // 同步发送重试次数
producer.setRetryAnotherBrokerWhenNotStoreOK(false); // 换 Broker 重试

// ===== 提高吞吐 =====
producer.setSendLatencyFaultEnable(false);  // 关闭故障延迟 (吞吐优先)
producer.setMaxMessageSize(4194304);        // 最大消息

// ===== 生产级配置建议 =====
// 1. 异步发送优于同步（吞吐量提升 3~5 倍）
// 2. 批量发送优于单条发送
// 3. 消息压缩（4KB 以上自动启用）
// 4. 关闭故障延迟（高吞吐场景）
// 5. Producer 建议单例复用
```

# 6. Consumer 调优

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");

// ===== 消费线程 =====
consumer.setConsumeThreadMin(20);     // 最小消费线程
consumer.setConsumeThreadMax(64);     // 最大消费线程
// 线程数建议：Queue 数 × 每Queue并发数

// ===== 拉取参数 =====
consumer.setPullBatchSize(32);        // 每次拉取消息数
consumer.setConsumeMessageBatchMaxSize(1);  // 每次消费最大消息数
// 提高 pullBatchSize = 提高网络效率
// 提高 consumeMessageBatchMaxSize = 减少线程开销

// ===== 流量控制 =====
consumer.setPullInterval(0);          // 拉取间隔，建议 0
consumer.setConsumeThreadMax(20);     // 控制消费并发

// ===== 生产级配置建议 =====
// 1. Queue 数量 = Consumer 实例数 × 消费线程数
// 2. 并发消费优于顺序消费（吞吐高 5~10 倍）
// 3. 批量消费优于单条
// 4. 避免消费阻塞
// 5. 合理配置消费线程池大小
```

# 7. Topic 与 Queue 规划

## 7.1 Queue 数量建议

```
读 Queue=4, 写 Queue=8, Broker=1
↓
Consumer 最多 4 个实例活跃（平均分配）
↓
建议：writeQueue > readQueue，方便后续动态扩容
```

| Queue 数 | 适用场景 | 注意 |
|----------|----------|------|
| 1 | 全局有序 | 无水平扩展 |
| 4-8 | 普通业务 | 默认配置 |
| 16-32 | 高吞吐 | Consumer 实例需匹配 |
| 64+ | 超高吞吐 | 过多可能导致 Rebalance 慢 |

# 8. 性能基准参考

| 场景 | 配置 | TPS | 延迟(P99) |
|------|------|-----|-----------|
| 小消息 100B | ASYNC_MASTER + ASYNC_FLUSH | ~10万/s | < 5ms |
| 中等消息 1KB | ASYNC_MASTER + ASYNC_FLUSH | ~8万/s | < 10ms |
| 大消息 4KB | ASYNC_MASTER + ASYNC_FLUSH | ~5万/s | < 20ms |
| 同步双写 | SYNC_MASTER + ASYNC_FLUSH | ~3万/s | < 30ms |
| 可靠模式 | SYNC_MASTER + SYNC_FLUSH | ~3千/s | < 50ms |

*以上数据仅供参考，实际性能取决于硬件配置和消息大小*

# 9. 调优检查清单

- [ ] OS: 文件描述符 > 65535
- [ ] OS: `vm.max_map_count` >= 655360
- [ ] OS: 磁盘调度器 = `none` (SSD)
- [ ] JVM: Xms = Xmx，使用 G1GC
- [ ] JVM: GC 日志开启并定期分析
- [ ] Broker: `flushDiskType=ASYNC_FLUSH`（吞吐优先）
- [ ] Broker: 线程池大小根据 CPU 核数调整
- [ ] Producer: 异步发送 + 压缩开启
- [ ] Consumer: 线程数 = Queue 数 × 系数
- [ ] Topic: Queue 数 >= Consumer 实例数

# 10. 参考

- [RocketMQ 最佳实践 — 性能调优](https://github.com/apache/rocketmq/blob/master/docs/cn/best_practice.md)
