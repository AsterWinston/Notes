
# 1. 概述

Canal 的性能调优涉及多个层面：MySQL 端、Canal Server 端、网络、存储（RingBuffer）、以及下游消费端。合理的调优可以让 Canal 处理数万 TPS 的增量数据。

# 2. MySQL 端优化

## 2.1 Binlog 相关配置

```ini
[mysqld]
# 设置合适的 binlog 文件大小（避免频繁切换）
max_binlog_size = 1024M

# 每 N 次事务提交后刷盘（值越大性能越好，但崩溃丢失风险越大）
sync_binlog = 100                    # 每 100 个事务刷盘

# Binlog 缓存大小
binlog_cache_size = 1M

# 组提交配置（MySQL 5.7+）
binlog_group_commit_sync_delay = 1000  # 等待 1000 微秒
binlog_group_commit_sync_no_delay_count = 100  # 或攒够 100 个事务

# Binlog 行映像模式（minimal 减小传输量）
binlog_row_image = FULL              # Canal 需要 FULL（有完整的前后镜像）
# 如果不需要所有列，可考虑 MINIMAL（但某些场景会丢失数据）
```

## 2.2 网络优化

```bash
# Linux 内核参数优化
# /etc/sysctl.conf

# TCP 缓冲区
net.core.rmem_max = 134217728        # 128MB
net.core.wmem_max = 134217728

# TCP 自动调优
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
```

# 3. Canal Server 端优化

## 3.1 RingBuffer 配置

```properties
# instance.properties

# RingBuffer 大小（必须是 2 的 n 次方）
canal.instance.memory.buffer.size = 32768    # 默认 16384

# 每个槽位的内存单元（字节）
canal.instance.memory.buffer.memunit = 2048  # 默认 1024

# 计算总内存：32768 * 2048 = 64MB
# 根据数据量调整：大流量用大 buffer，小流量用小 buffer
```

**RingBuffer 大小建议**：

| TPS | Buffer Size | Memory |
|-----|-------------|--------|
| < 100 | 8192 | 16MB |
| 100 - 500 | 16384 | 32MB |
| 500 - 2000 | 32768 | 64MB |
| 2000 - 5000 | 65536 | 128MB |
| > 5000 | 131072 | 256MB |

## 3.2 JVM 参数优化

```bash
# Canal Server 启动 JVM 参数（在 startup.sh 中设置）

# 推荐配置（8GB 内存的机器）
JAVA_OPTS="-server
  -Xms4g
  -Xmx4g
  -Xmn2g                                    # 年轻代 2GB
  -XX:MetaspaceSize=256m
  -XX:MaxMetaspaceSize=512m
  -XX:+UseG1GC                              # 使用 G1 垃圾回收器
  -XX:MaxGCPauseMillis=200                  # 最大 GC 停顿 200ms
  -XX:+DisableExplicitGC                    # 禁用 System.gc()
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=./heapdump.hprof
  -Dfile.encoding=UTF-8"
```

## 3.3 线程数调优

```properties
# canal.properties

# Parser 线程数（Binlog 拉取和解析，通常 1 个足够）
canal.instance.parser.threads = 1

# Sink 线程数（事件过滤，通常 1 个足够）
canal.instance.sink.threads = 1

# Store 线程数（数据存储，通常 1 个足够）
canal.instance.store.threads = 1

# Netty Boss 线程数（接受客户端连接）
canal.port.boss.threads = 1

# Netty Worker 线程数（处理客户端请求）
canal.port.worker.threads = 8              # CPU 核心数 * 2

# 注意：Parser/Sink/Store 使用 Disruptor，单线程通常足够
# 瓶颈通常在 Netty Worker 线程和消费者处理速度
```

# 4. 批量处理优化

## 4.1 批量拉取

```java
// 客户端减少获取频率，增大批量大小
int batchSize = 1000;            // 每次获取 1000 条
int timeout = 5000;              // 等待 5 秒

Message message = connector.getWithoutAck(batchSize, timeout, TimeUnit.MILLISECONDS);
```

## 4.2 批量 ACK

```java
// 批量处理多个 Message 后统一 ACK
// 优点：减少 ACK 和位点持久化的开销
// 缺点：失败时回滚的数据更多

int ackBatchSize = 100;         // 每处理 100 个 Message 才 ACK
List<Long> batchIds = new ArrayList<>();

while (running) {
    Message message = connector.getWithoutAck(1000);
    
    processEntries(message.getEntries());
    batchIds.add(message.getId());
    
    if (batchIds.size() >= ackBatchSize) {
        connector.ack(batchIds.get(batchIds.size() - 1));
        batchIds.clear();
    }
}
```

# 5. MQ 模式优化

## 5.1 Kafka 优化

```properties
# Kafka Producer 优化配置
kafka.batch.size = 131072              # 128KB
kafka.linger.ms = 50                   # 50ms 内凑数据批量发
kafka.buffer.memory = 67108864         # 64MB 发送缓冲区
kafka.compression.type = lz4           # LZ4 压缩（平衡速度和压缩率）
kafka.max.in.flight.requests.per.connection = 5
kafka.acks = 1                         # 仅 Leader 确认（平衡可靠性）
kafka.enable.idempotence = true        # 幂等发送（防重）
```

## 5.2 RocketMQ 优化

```properties
rocketmq.enable.batch = true
rocketmq.send.timeout = 3000
rocketmq.retry.times.when.send.failed = 3
```

# 6. 表结构缓存优化

## 6.1 TSDB 配置

```properties
# 开启 TSDB 表结构快照（减少查询 INFORMATION_SCHEMA）
canal.instance.tsdb.enable = true
canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
```

## 6.2 TableMetaCache 优化

```java
// 增加表结构缓存大小
canal.instance.table.meta.cache.size = 10000

// 表结构缓存过期时间
canal.instance.table.meta.cache.expire = 3600
```

# 7. 网络与序列化

## 7.1 FlatMessage 模式

```properties
# 使用 FlatMessage 模式（推荐）
canal.mq.flatMessage = true

# FlatMessage 优势：
# 1. 消息体积更小（JSON 扁平化）
# 2. 不需要 Protobuf 解析，消费端处理更快
# 3. 可读性好，便于调试和直接消费

# 对比：
# Protobuf 格式: ~2KB/条
# FlatMessage 格式: ~500B/条（减少 75%）
```

## 7.2 数据压缩

```java
// Canal Server 对 Protobuf 消息进行自动压缩
public class MessageCompressor {
    
    // 压缩阈值：超过 64KB 的消息才压缩
    private static final int COMPRESSION_THRESHOLD = 65536;
    
    public byte[] compress(byte[] data) {
        if (data.length < COMPRESSION_THRESHOLD) {
            return data;  // 太小，不压缩
        }
        
        // GZIP 压缩
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (GZIPOutputStream gzip = new GZIPOutputStream(baos)) {
            gzip.write(data);
        }
        
        byte[] compressed = baos.toByteArray();
        
        // 只有确实变小时才使用压缩
        return compressed.length < data.length ? compressed : data;
    }
}
```

# 8. 消费端优化

## 8.1 多线程并行消费

```java
public class ParallelConsumer {
    
    // 单线程获取，多线程处理
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public void consume() {
        while (running) {
            Message message = connector.getWithoutAck(1000);
            
            // 按表名分组，相同表的数据顺序处理（保证一致性）
            Map<String, List<Entry>> grouped = groupByTable(message.getEntries());
            
            // 不同表之间可以并行处理
            List<CompletableFuture<Void>> futures = grouped.values().stream()
                .map(entries -> CompletableFuture.runAsync(() -> 
                    processEntries(entries), executor))
                .collect(Collectors.toList());
            
            // 等待所有表处理完成
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            
            connector.ack(message.getId());
        }
    }
}
```

## 8.2 批量写入目标

```java
// 批量 INSERT 优化（JDBC batch）
String sql = "INSERT INTO target_table (col1, col2) VALUES (?, ?)";

List<Object[]> batch = new ArrayList<>();
for (Entry entry : entries) {
    batch.add(new Object[]{val1, val2});
    
    if (batch.size() >= 1000) {
        jdbcTemplate.batchUpdate(sql, batch);
        batch.clear();
    }
}

if (!batch.isEmpty()) {
    jdbcTemplate.batchUpdate(sql, batch);
}
```

# 9. 性能监控指标

## 9.1 关键指标

```java
public class CanalMetrics {
    
    // 1. 消费位点延迟
    // 当前 binlog 位点 vs MySQL 最新位点
    
    // 2. Canal 处理 TPS
    // 每秒处理的 DML 事件数
    
    // 3. RingBuffer 使用率
    // 已用槽位 / 总槽位
    
    // 4. 消息积压量
    // RingBuffer 中待消费的消息数量
    
    // 5. GC 频率与耗时
    // Full GC 次数和停顿时间
    
    // 6. 网络 I/O
    // 入站/出站流量
    
    /**
     * 计算消费延迟
     */
    public long calculateDelay() {
        long currentPosition = store.getCurrentPosition();
        long latestPosition = getMySQLMasterPosition();
        
        return latestPosition - currentPosition;
    }
    
    /**
     * 计算 TPS
     */
    public double calculateTps() {
        long currentCount = totalEventCount.get();
        long now = System.currentTimeMillis();
        
        if (lastCheckTime > 0) {
            double elapsed = (now - lastCheckTime) / 1000.0;
            double tps = (currentCount - lastEventCount) / elapsed;
            
            lastEventCount = currentCount;
            lastCheckTime = now;
            
            return tps;
        }
        
        lastEventCount = currentCount;
        lastCheckTime = now;
        return 0;
    }
}
```

## 9.2 性能压测

```bash
# 使用 sysbench 生成大量数据变更
sysbench oltp_write_only \
  --mysql-host=127.0.0.1 \
  --mysql-port=3306 \
  --mysql-user=root \
  --mysql-password=root \
  --mysql-db=test \
  --tables=10 \
  --table-size=100000 \
  --threads=16 \
  --time=300 \
  run

# 同时监控 Canal 的表现
# 观察 RingBuffer 使用率、延迟、TPS
```

# 10. 常见性能瓶颈

| 瓶颈 | 现象 | 解决方案 |
|------|------|---------|
| **慢 SQL** | Canal 启动慢、表结构获取慢 | 为 INFORMATION_SCHEMA 加索引或开启 TSDB |
| **RingBuffer 满** | Parser 阻塞、`put` 等待 | 增大 buffer.size 或加快消费 |
| **消费跟不上** | 位点延迟持续增大 | 增加消费线程、优化业务逻辑 |
| **GC 频繁** | 吞吐量抖动 | 增大堆内存，使用 G1GC |
| **网络瓶颈** | 带宽打满 | 启用压缩、减小 batch_size |
| **MySQL binlog 量大** | 数据变更太快 | 添加过滤规则、减少不必要订阅 |
| **位点持久化慢** | ACK 慢 | 减小持久化频率、使用 SSD |
