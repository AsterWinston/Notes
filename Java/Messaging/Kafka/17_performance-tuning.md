# 1. 性能调优总览

## 1.1 调优金字塔

```
                 ┌─────────────────┐
                 │  4. 应用层调优    │ 分区策略、序列化、压缩、幂等
                 │                  │
                 ┌───────────────────┐
                 │  3. JVM 调优       │ GC策略、堆大小、线程
                 │                    │
                 ┌─────────────────────┐
                 │  2. OS 调优           │ Page Cache、文件描述符、CPU
                 │                       │
                 ┌─────────────────────────┐
                 │  1. 硬件/网络调优           │ SSD、网络带宽、拓扑
                 └─────────────────────────────┘
```

# 2. Broker 调优

## 2.1 线程模型调优

```properties
##############################
# 关键线程配置
##############################

# 网络线程数（Processor 数）
num.network.threads=8
# 建议: CPU 核数，处理大量连接时增大

# I/O 线程数（RequestHandler 数）
num.io.threads=16
# 建议: CPU 核数 × 2，处理大量请求时增大
# 监控: kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent < 0.3 需增大

# 后台线程数
background.threads=10

# Follower 拉取线程数
num.replica.fetchers=4
# 建议: 分区数较多时增大，默认 1 可能不够
# 多线程可并行从不同 Leader 拉取数据

# Log Cleaner 线程数
log.cleaner.threads=2
```

## 2.2 网络调优

```properties
##############################
# Socket 配置
##############################

# 每个连接最大排队请求
queued.max.requests=1000
# 增大可缓冲更多请求，但内存占用增加

# Socket 缓冲区（OS 优先）
socket.send.buffer.bytes=1048576    # 1MB
socket.receive.buffer.bytes=1048576 # 1MB

# 最大请求大小（从客户端到 Broker）
socket.request.max.bytes=104857600  # 100MB

# 最大连接数
max.connections=10000

##############################
# 网络线程相关
##############################

# 监听器 Backlog
socket.listen.backlog=128
# 当监听队列满时控制新连接请求排队数

# 连接超时
connections.max.idle.ms=600000      # 10分钟
```

## 2.3 日志存储调优

```properties
##############################
# 日志目录配置
##############################

# 多磁盘目录（提高并发 I/O）
log.dirs=/data1/kafka-logs,/data2/kafka-logs,/data3/kafka-logs
# 每个目录放在不同物理磁盘，提升 I/O 带宽

# Segment 大小
log.segment.bytes=1073741824  # 1GB
# 减小 → 更多 Segment 文件，查找变慢
# 增大 → 文件太大，Compact 清理变慢
# 推荐: 512MB~2GB

# Segment 滚动时间
log.segment.ms=604800000      # 7天
# 用于控制 Segment 生命周期

##############################
# 索引配置
##############################

# 索引大小上限
log.index.size.max.bytes=10485760   # 10MB
# 增大索引 → 更密集的索引，查找更快但内存/磁盘占用增加

# 索引间隔
log.index.interval.bytes=4096       # 4KB
# 减小 → 更密集索引，写入时索引开销略大，查找更快

##############################
# 预分配
##############################

# 预分配日志文件（避免实时分配的开销）
log.preallocate=false
# 开启: 创建新 Segment 时提前占满磁盘空间
# 优点: 减少磁盘碎片和分配时间
# 缺点: 增加磁盘空间占用（空洞文件）
```

## 2.4 数据保留与清理

```properties
# 基于时间的保留
log.retention.ms=604800000      # 7天
log.retention.check.interval.ms=300000  # 5分钟检查一次

# 基于大小的保留
log.retention.bytes=-1          # -1 表示不限制

# Compact 清理配置
log.cleaner.threads=2
log.cleaner.io.max.bytes.per.second=1.7976931348623157E308 # 不限制
log.cleaner.backoff.ms=15000
log.cleaner.min.cleanable.ratio=0.5  # 脏率超过50%才触发
log.cleaner.delete.retention.ms=86400000  # tombstone保留
```

# 3. Producer 调优

## 3.1 吞吐优先配置

```java
Properties props = new Properties();

// === 网络 ===
props.put("bootstrap.servers", "broker1:9092,broker2:9092,broker3:9092");

// === 批量处理 ===
props.put("batch.size", 131072);                // 128KB
props.put("linger.ms", 10);                     // 10ms 额外等待聚合
props.put("buffer.memory", 67108864);           // 64MB 缓冲区

// === 压缩 ===
props.put("compression.type", "lz4");           // 高速压缩
// none: 不压缩, 延迟最低
// gzip: 压缩率最高, CPU 开销最大
// snappy: 平衡压缩率和速度
// lz4: 速度最快, 压缩率中等（推荐）
// zstd: 新一代压缩, 压缩率高且速度较快

// === 发送策略 ===
props.put("acks", "1");                         // 或 all
props.put("max.in.flight.requests.per.connection", 5);
props.put("max.request.size", 10485760);        // 10MB

// === 可靠性 ===
props.put("enable.idempotence", true);
props.put("retries", Integer.MAX_VALUE);
props.put("delivery.timeout.ms", 120000);
```

## 3.2 低延迟优先配置

```java
Properties props = new Properties();
props.put("batch.size", 0);                     // 不聚合
props.put("linger.ms", 0);                      // 不等待
props.put("compression.type", "none");          // 不压缩（省 CPU）
props.put("buffer.memory", 33554432);           // 32MB
props.put("acks", "1");                         // 或 0
props.put("max.in.flight.requests.per.connection", 1); // 严格有序
```

## 3.3 Producer 参数影响矩阵

| 参数 | 增大后吞吐 | 增大后延迟 | CPU | 内存 |
|------|----------|-----------|-----|------|
| batch.size | ↑↑ | ↑ | → | ↑ |
| linger.ms | ↑ | ↑↑ | → | → |
| buffer.memory | ↑ | → | → | ↑↑ |
| compression.type=lz4 | ↑↑ | ↑ | ↑↑ | ↓ |
| acks=0→1→all | ↓ | ↑↑ | → | → |
| max.in.flight.requests | ↑ | → | → | ↑ |

# 4. Consumer 调优

## 4.1 通用配置

```java
Properties props = new Properties();

// === 拉取策略 ===
props.put("fetch.min.bytes", 1048576);          // 1MB 最小拉取量
props.put("fetch.max.wait.ms", 500);            // 500ms 最大等待
props.put("fetch.max.bytes", 52428800);         // 50MB 总上限
props.put("max.partition.fetch.bytes", 10485760);// 10MB 单分区
props.put("max.poll.records", 1000);            // 单次 poll 1000 条

// === Offset 提交 ===
props.put("enable.auto.commit", false);         // 手动提交
props.put("auto.commit.interval.ms", 5000);

// === Session 与心跳 ===
props.put("session.timeout.ms", 60000);         // 适当放大
props.put("heartbeat.interval.ms", 10000);
props.put("max.poll.interval.ms", 600000);      // 10分钟

// === 分配策略 ===
props.put("partition.assignment.strategy",
        "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");

// === 接收缓冲区 ===
props.put("receive.buffer.bytes", 262144);      // 256KB (放大减少系统调用)
```

## 4.2 高吞吐消费配置

```java
props.put("fetch.min.bytes", 10485760);       // 10MB 批量拉取
props.put("fetch.max.wait.ms", 1000);         // 1秒等待
props.put("max.partition.fetch.bytes", 52428800); // 50MB
props.put("max.poll.records", 5000);          // 5000 条
props.put("receive.buffer.bytes", 1048576);   // 1MB
```

## 4.3 Consumer 消费模型调优

```java
// 多线程消费决策树:

// 1. 如果分区数 >= 线程数 → 使用一个线程一个 Consumer
// 2. 如果分区数 < 线程数或需要更复杂模型

// 推荐方案:
// Consumer 线程: 只负责 poll + offset 提交
// Worker 线程池: 负责消息处理

// 解耦消费和处理:
// 避免处理慢影响下次 poll，造成 max.poll.interval.ms 超时
```

# 5. OS 调优

## 5.1 文件系统与磁盘

```bash
# /etc/sysctl.conf

# === 页缓存调优 ===
# 脏页刷新阈值（降低延迟）
vm.dirty_background_ratio=5   # 后台刷脏页比率（%）
vm.dirty_ratio=10             # 强制刷脏页比率（%）
# 含义: 当脏页占内存 5% 时后台开始刷，占 10% 时阻塞写入强制刷

# 脏页过期时间（避免老旧脏页）
vm.dirty_expire_centisecs=3000  # 30秒
vm.dirty_writeback_centisecs=500 # 5秒

# === 虚拟内存 ===
# 降低 swappiness（优先用内存而非 swap）
vm.swappiness=1

# === 文件描述符 ===
# 每个 Broker 需要打开至少 (Partitions × 每个分区的文件数)
# 安全值: 100000+
fs.file-max=1000000

# 用户级文件描述符限制
# /etc/security/limits.conf
# kafka soft nofile 100000
# kafka hard nofile 200000
```

## 5.2 网络调优

```bash
# TCP 缓冲区
net.core.rmem_max = 134217728       # 接收缓冲最大值 (128MB)
net.core.wmem_max = 134217728       # 发送缓冲最大值
net.core.rmem_default = 268435456
net.core.wmem_default = 268435456
net.core.optmem_max = 134217728

# TCP 积压队列和连接重用
net.core.somaxconn = 32768          # 连接队列大小
net.core.netdev_max_backlog = 5000  # 接收队列大小

# TCP 连接复用
net.ipv4.tcp_tw_reuse = 1

# 网络设备积压
net.ipv4.tcp_max_syn_backlog = 32768

# 孤儿 socket 数量
net.ipv4.tcp_max_orphans=262144
```

## 5.3 挂载选项（ext4/xfs）

```bash
# /etc/fstab 示例
# 使用 noatime 减少写操作
/dev/sdb1 /data/kafka-logs xfs defaults,noatime,nodiratime 0 0

# noatime: 不记录文件访问时间（减少写入）
# nodiratime: 不记录目录访问时间
```

# 6. JVM 调优

## 6.1 GC 策略

```bash
# KAFKA_HEAP_OPTS 示例
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g \
    -XX:MetaspaceSize=128m \
    -XX:MaxMetaspaceSize=256m \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=20 \
    -XX:InitiatingHeapOccupancyPercent=35 \
    -XX:G1HeapRegionSize=16M \
    -XX:MinMetaspaceFreeRatio=50 \
    -XX:MaxMetaspaceFreeRatio=80 \
    -XX:+ExplicitGCInvokesConcurrent \
    -XX:+ParallelRefProcEnabled \
    -XX:+UnlockExperimentalVMOptions \
    -XX:G1NewSizePercent=5 \
    -XX:+DisableExplicitGC \
    -Djava.awt.headless=true"

# 说明:
# Xms = Xmx: 堆大小固定，避免动态调整
# UseG1GC: 推荐 G1（比 CMS 更可预测）
# MaxGCPauseMillis=20: 最大 20ms GC 暂停
# InitiatingHeapOccupancyPercent=35: 堆占用 35% 触发并发标记
```

## 6.2 GC 日志与分析

```bash
# GC 日志配置
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100M
-Xloggc:/var/log/kafka/gc.log

# 分析方法
# 1. 查看 GC 暂停次数和时间
grep "GC pause" /var/log/kafka/gc.log | tail -50

# 2. 使用 GCViewer / gceasy.io 在线分析
```

## 6.3 Kafka 特有 JVM 配置

```bash
# Kafka 自身 JVM 参数
export KAFKA_JVM_PERFORMANCE_OPTS="\
    -server \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=20 \
    -XX:InitiatingHeapOccupancyPercent=35 \
    -XX:+DisableExplicitGC \
    -Djava.awt.headless=true"

# 如果 Broker 崩溃时想 dump heap
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/kafka/heap-dump.hprof
```

# 7. 压力测试

## 7.1 kafka-producer-perf-test

```bash
# Producer 性能测试
kafka-producer-perf-test.sh \
    --topic perf-test \
    --num-records 10000000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props \
        bootstrap.servers=broker1:9092,broker2:9092 \
        acks=1 \
        batch.size=65536 \
        linger.ms=5 \
        compression.type=lz4 \
        buffer.memory=67108864

# 输出:
# 500000 records sent, 98039.2 records/sec (95.70 MB/sec), 15.2 ms avg latency
```

## 7.2 kafka-consumer-perf-test

```bash
# Consumer 性能测试
kafka-consumer-perf-test.sh \
    --topic perf-test \
    --messages 10000000 \
    --bootstrap-server broker1:9092,broker2:9092 \
    --group perf-test-group \
    --fetch-size 1048576 \
    --threads 1

# 输出:
# start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec
```

## 7.3 端到端延迟测试

```bash
kafka-producer-perf-test.sh \
    --topic e2e-test \
    --num-records 100000 \
    --record-size 512 \
    --throughput 1000 \
    --producer-props \
        bootstrap.servers=broker1:9092 \
        acks=all \
        linger.ms=0
```

# 8. 性能基准参考

## 8.1 典型硬件配置

```
配置 A (标准):
  CPU: 16 核
  内存: 64GB (JVM 32GB)
  磁盘: 4 × 1TB SSD (RAID 0 / 多目录)
  网络: 10Gbps

配置 B (高吞吐):
  CPU: 32 核
  内存: 128GB (JVM 64GB)
  磁盘: 8 × 1TB NVMe
  网络: 25Gbps
```

## 8.2 基准吞吐量（参考值）

| 场景 | 吞吐量 | 延迟 (P99) |
|------|--------|-----------|
| 1 Producer, acks=0, 无压缩 | ~200 MB/s | < 5ms |
| 1 Producer, acks=1, lz4 | ~180 MB/s | < 10ms |
| 1 Producer, acks=all, lz4, 3副本 | ~150 MB/s | < 20ms |
| 多 Producer (8个) acks=1 | ~800 MB/s | < 15ms |
| 单 Consumer | ~300 MB/s | - |

> 注：实际吞吐取决于硬件、网络、记录大小、分区数等多种因素。

# 9. 调优决策流程图

```
性能瓶颈排查:

① 观察系统指标
   CPU > 85%? → 考虑是否压缩/序列化开销大
   磁盘 IO > 80%? → 增加 log.dirs 多目录
   网络 > 70%? → 增大 batch.size, 开启压缩

② 观察 Broker 指标
   RequestQueueSize 持续增长? → 增加 num.io.threads
   NetworkProcessorAvgIdlePercent < 0.3 → 增加 num.network.threads
   UnderReplicatedPartitions > 0 → 检查副本同步

③ 观察 Producer 指标
   buffer-available-bytes 接近 0? → 增大 buffer.memory
   request-latency-avg 高? → 增大 batch.size + linger.ms
   record-error-rate 高? → 增大 retries / 检查网络

④ 观察 Consumer 指标
   records-lag-max 持续增长? → 增加 Consumer 数 / 优化处理逻辑
   fetch-latency-avg 高? → 增大 fetch.min.bytes / fetch.max.bytes
   rebalance-rate-per-hour 高? → 增大 session.timeout.ms / 使用 Static Membership
```
