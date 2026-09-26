
# 1. 概述

本章汇总 Canal 在生产环境中常见的故障现象、原因分析和解决方案。

# 2. 连接类问题

## 2.1 无法连接到 MySQL

**现象**：
```
ERROR c.a.o.c.p.d.rds.RdsBinlogEventParserProxy - connect failed
com.mysql.cj.jdbc.exceptions.CommunicationsException: Communications link failure
```

**排查思路**：

```bash
# 1. 检查 MySQL 是否监听正确端口
netstat -tlnp | grep 3306

# 2. 从 Canal 所在机器 telnet MySQL 端口
telnet 192.168.1.100 3306

# 3. 检查 MySQL 绑定地址
mysql> SHOW VARIABLES LIKE 'bind_address';
# 如果是 127.0.0.1，Canal 在其他机器上时无法连接

# 4. 检查防火墙规则
iptables -L -n | grep 3306

# 5. 检查 MySQL 最大连接数
mysql> SHOW VARIABLES LIKE 'max_connections';
mysql> SHOW STATUS LIKE 'Threads_connected';
```

**解决方案**：
```ini
# my.cnf 修改绑定地址
bind-address = 0.0.0.0  # 或指定 IP

# 或者通过 SSH 隧道
ssh -L 3307:localhost:3306 user@mysql-server
# 然后 Canal 连接 localhost:3307
```

## 2.2 权限不足

**现象**：
```
Access denied; you need (at least one of) the SUPER, REPLICATION CLIENT privilege(s)
```

**解决方案**：
```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

## 2.3 Server ID 冲突

**现象**：
```
A slave with the same server_uuid/server_id as this slave has connected to the master
```

**原因**：多个 Canal 实例或 MySQL Slave 使用了相同的 `server-id`。

**解决方案**：
```properties
# instance.properties — 确保 slaveId 全局唯一
canal.instance.mysql.slaveId = 5678  # 改为其他值

# 检查 MySQL 的 server-id
mysql> SHOW VARIABLES LIKE 'server_id';
```

# 3. Binlog 问题

## 3.1 Binlog 格式不是 ROW

**现象**：
```
Cannot replicate because the master has BINLOG_FORMAT = STATEMENT
```

**解决方案**：
```sql
-- 在线修改（立即生效，但重启后失效）
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';

-- 永久修改 my.cnf
[mysqld]
binlog_format = ROW
binlog_row_image = FULL
```

## 3.2 Binlog 文件被删除

**现象**：
```
Could not find first log file name in binary log index file
```
或
```
Client requested master to start replication from position > file size
```

**原因**：MySQL 自动清理了 Canal 位点所在的 Binlog 文件（过期删除）。

**排查**：
```bash
# 查看 MySQL binlog 保留情况
mysql> SHOW BINARY LOGS;
mysql> SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';

# 查看 Canal 当前位点
# 通过 Admin 界面或 ZK 查看
```

**解决方案**：

**方案 A**：增加 Binlog 保留时间
```sql
-- MySQL 8.0+
SET GLOBAL binlog_expire_logs_seconds = 1209600;  -- 14天
```

**方案 B**：重置 Canal 位点为最新的 Binlog 文件
```properties
# 清空 instance.properties 中的位点配置
canal.instance.master.journal.name =
canal.instance.master.position =
canal.instance.master.timestamp =
# 重启 Canal → 自动从最新位置开始
```

**方案 C**：使用全量+增量恢复（不丢数据）
```bash
# 1. 从备份恢复全量数据到目标
# 2. 记录全量备份时的 Binlog 位点
# 3. 设置 Canal 的位点为备份时的位点
# 4. 启动 Canal 继续增量同步
```

## 3.3 Binlog 损坏

**现象**：
```
Event deserialize error: Failed to deserialize event
Checksum mismatch
```

**解决方案**：
```sql
-- 1. 检查 MySQL binlog checksum
SHOW VARIABLES LIKE 'binlog_checksum';

-- 2. 查看 binlog 是否完好
mysqlbinlog mysql-bin.000123 --start-position=45678901 --stop-position=45678902

-- 3. 跳过损坏的事件
# 在 Canal Admin 中手动将位点跳到下一个可用位置
```

# 4. 性能问题

## 4.1 消费延迟过大

**现象**：
```
canal_instance_subscribe_delay > 300 秒
```

**排查**：

```bash
# 1. 检查 RingBuffer 积压
# Prometheus 指标：produce_seq - consume_seq > 10000

# 2. 检查消费者性能
# 观察业务日志，看消费者处理速度

# 3. 检查 GC 情况
jstat -gcutil <pid> 1000

# 4. 检查是否有大事务阻塞
# 查看 canal.log 中是否有大量 DML 堆积
```

**解决方案**：

| 原因 | 解决方案 |
|------|---------|
| RingBuffer 过小 | 增大 `buffer.size` |
| 消费端处理慢 | 增加消费线程数/优化业务逻辑 |
| 大事务阻塞 | MySQL 端避免大事务 |
| GC 频繁 | 增加 JVM 堆内存、使用 G1GC |
| 网络带宽不足 | 启用压缩、限流 |

## 4.2 RingBuffer 阻塞

**现象**：
```
canal_instance_publish_blocking_time > 0
```
Parser 等待 Sink/Store 有空位，出现阻塞。

**原因**：消费跟不上生产，RingBuffer 满。

**解决方案**：
```properties
# 1. 增大 RingBuffer
canal.instance.memory.buffer.size = 65536  # 增加一倍

# 2. 增大批量大小
canal.instance.memory.buffer.memunit = 2048

# 3. 加快消费（在消费端优化）
```

## 4.3 解析性能瓶颈

**现象**：大量 CPU 消耗在 Binlog 解析上。

**解决方案**：
```properties
# 1. 减少订阅的表数量
canal.instance.filter.regex = order_db\\.order,order_db\\.payment  # 缩小范围

# 2. 使用黑名单排除不需要的表
canal.instance.filter.black.regex = .*\\.log_table,.*\\._backup_.*

# 3. 增加 instance 拆分（按库拆分到不同的 Canal Instance）
```

# 5. 数据一致性问题

## 5.1 数据丢失

**可能原因**：

1. Canal 位点被错误重置
2. Binlog 文件被清理
3. 应用层处理逻辑有 bug

**排查**：

```bash
# 对比源库和目标库的数据
# MySQL 源库
mysql> SELECT COUNT(*), MAX(id), SUM(amount) FROM source_db.orders;

# MySQL 目标库
mysql> SELECT COUNT(*), MAX(id), SUM(amount) FROM target_db.orders;

# 通过对比找出差异
```

**解决方案**：
```bash
# 1. 使用 ETL 全量同步补偿
curl http://localhost:8081/etl/rdb/orders_sync.yml -X POST

# 2. 或者通过位点回溯重新消费（如果 binlog 还存在）
```

## 5.2 数据重复

**原因**：Canal 是 At-Least-Once 语义，在故障恢复时可能重复投递。

**解决方案**：消费端实现幂等性

```java
public class IdempotentConsumer {
    
    // 使用 Redis 记录已处理的 Event 唯一标识
    private RedisTemplate<String, String> redis;
    
    public void consume(Message message) {
        for (Entry entry : message.getEntries()) {
            if (entry.getEntryType() != EntryType.ROWDATA) continue;
            
            // 构建唯一标识：binlog文件+偏移量+行号
            String key = String.format("%s:%d:%d",
                entry.getHeader().getLogfileName(),
                entry.getHeader().getLogfileOffset(),
                entry.hashCode());
            
            // 检查是否已处理
            if (redis.opsForValue().get(key) != null) {
                continue;  // 已处理，跳过
            }
            
            // 处理数据
            processEntry(entry);
            
            // 标记已处理（设置过期时间，避免无限增长）
            redis.opsForValue().set(key, "1", 24, TimeUnit.HOURS);
        }
        
        connector.ack(message.getId());
    }
}
```

## 5.3 数据乱序

**原因**：多线程消费时，同一行的不同操作可能被并发执行。

**解决方案 A**：按主键分组，相同主键串行处理

```java
// 保证同一行数据的操作顺序
int threadIndex = Math.abs(rowId.hashCode()) % threadCount;
// 同一行数据始终由同一个线程处理
executors[threadIndex].submit(() -> processRow(row));
```

**解决方案 B**：利用 Kafka 分区保证顺序

```properties
# 相同主键的数据进入同一分区
canal.mq.partitionHash = test\\.user:id
```

# 6. ZooKeeper 问题

## 6.1 ZK 连接断开

**现象**：
```
WARN ... - KeeperErrorCode = ConnectionLoss
WARN ... - Session expired
```

**解决方案**：
```properties
# 增大 ZK 会话超时
canal.zookeeper.session.timeout.ms = 30000
canal.zookeeper.connection.timeout.ms = 30000

# 检查 ZK 是否健康
echo stat | nc localhost 2181
```

## 6.2 脑裂问题

**现象**：两个 Canal Server 同时运行同一个 Instance。

**排查**：
```bash
# 查看 ZK 中的 running 节点
zkCli.sh get /otter/canal/destinations/example/running

# 查看两台服务器的日志，确认是否同时在消费
tail -f /opt/canal/logs/example/example.log
```

**解决方案**：
1. 先停止其中一台的 Instance
2. 检查 ZK 会话是否正常
3. 检查网络是否有分区

# 7. Adapter 问题

## 7.1 目标库连接失败

**现象**：
```
Caused by: com.mysql.cj.jdbc.exceptions.CommunicationsException
```

**解决方案**：
```yaml
# application.yml — 增加连接池配置
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: rdb
              key: mysql1
              properties:
                jdbc.url: jdbc:mysql://127.0.0.1:3306/target_db?useUnicode=true&autoReconnect=true&failOverReadOnly=false
                jdbc.maxWait: 60000
                jdbc.maxActive: 20
                jdbc.minIdle: 5
                jdbc.initialSize: 5
                jdbc.validationQuery: SELECT 1
                jdbc.testOnBorrow: true
                jdbc.testWhileIdle: true
                jdbc.timeBetweenEvictionRunsMillis: 60000
```

## 7.2 ES 写入失败

**现象**：
```
org.elasticsearch.client.ResponseException: Failed to deserialize exception
```

**常见原因和解决方案**：

| 原因 | 解决方案 |
|------|---------|
| ES 版本不兼容 | 升级 Adapter ES 客户端版本 |
| 字段类型冲突 | 检查 Mapping，统一字段类型 |
| 文档过大 | 拆分文档或增加 `http.max_content_length` |
| 索引只读 | 调整磁盘水位线或清理旧数据 |

# 8. 调试技巧

## 8.1 开启详细日志

```xml
<!-- logback.xml — 临时开启 DEBUG 日志 -->
<logger name="com.alibaba.otter.canal" level="DEBUG"/>
<logger name="com.alibaba.otter.canal.parse" level="DEBUG"/>
<logger name="com.alibaba.otter.canal.sink" level="DEBUG"/>
```

## 8.2 查看 Binlog 原始内容

```bash
# 使用 mysqlbinlog 查看 binlog 中指定位置的数据
mysqlbinlog --verbose --base64-output=DECODE-ROWS \
  --start-position=45678901 --stop-position=45678902 \
  mysql-bin.000123

# 结合 Canal 日志中的位点信息确认 Canal 收到的内容是否正确
```

## 8.3 使用 Logger Adapter 调试

```yaml
# 临时启用 Logger Adapter 查看 Canal 收到的数据
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: debug
          outerAdapters:
            - name: logger
```

## 8.4 检查 RingBuffer 状态

```java
// 通过 JMX 查看 RingBuffer 状态
MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
ObjectName name = new ObjectName("canal:type=Store,name=MemoryEventStore");
Long produceSeq = (Long) mbs.getAttribute(name, "ProduceSeq");
Long consumeSeq = (Long) mbs.getAttribute(name, "ConsumeSeq");
System.out.printf("RingBuffer: produce=%d, consume=%d, lag=%d%n",
    produceSeq, consumeSeq, produceSeq - consumeSeq);
```

# 9. 常见错误码

| 错误信息 | 原因 | 解决 |
|---------|------|------|
| `CommunicationsException` | MySQL 连接断开 | 检查网络和 MySQL 状态 |
| `Access denied` | 权限不足 | 授予 REPLICATION 权限 |
| `server_uuid conflict` | Server ID 冲突 | 修改 slaveId |
| `Found 1 prepared transactions` | 有未提交的 XA 事务 | 提交或回滚 XA 事务 |
| `binlog format != ROW` | Binlog 格式不对 | 改为 ROW 格式 |
| `log file not found` | Binlog 文件被删除 | 增加保留时间或重置位点 |
| `ring buffer full` | RingBuffer 满了 | 增大 buffer 或加快消费 |
| `Can't find checkpoint` | 位点文件丢失 | 重置位点重新开始 |
| `Table 'xxx' doesn't exist` | 目标表不存在 | 在目标库创建对应表 |
| `Duplicate entry` | 主键冲突 | 检查处理逻辑或使用 REPLACE |
