
# 1. 部署架构

## 1.1 标准生产部署

```
                        ┌─────────────────────┐
                        │      ZooKeeper      │
                        │      Cluster         │
                        │  (可选，单机可不用)   │
                        └──────────┬──────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
   ┌─────▼─────┐            ┌─────▼─────┐            ┌─────▼─────┐
   │ Canal      │            │ Canal      │            │ Canal      │
   │ Server 1   │            │ Server 2   │            │ Server 3   │
   │ (Active)   │            │ (Standby)  │            │ (Standby)  │
   └─────┬─────┘            └─────┬─────┘            └─────┬─────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                        ┌─────────▼─────────┐
                        │      Kafka        │
                        │     Cluster       │
                        │ (Topic: canal_xxx)│
                        └─────────┬─────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
   ┌─────▼─────┐           ┌─────▼─────┐           ┌─────▼─────┐
   │ Canal      │           │  Flink     │           │ Custom     │
   │ Adapter    │           │  Consumer  │           │ Consumer   │
   │ (RDB/ES)   │           │            │           │ (业务逻辑) │
   └───────────┘           └─────────────┘           └─────────────┘
```

## 1.2 部署建议

1. **Canal Server 至少 2 台**（HA 模式），建议 3 台
2. **ZooKeeper 集群 3-5 台**（奇数节点）
3. **Canal Server 与 MySQL 尽量同机房**（减小网络延迟）
4. **日志和数据存储使用 SSD**
5. **Canal Admin 独立部署**（不要在 Canal Server 上）

# 2. 配置规范

## 2.1 Server 端配置清单

```properties
# canal.properties — 生产配置检查清单

# 1. 监听端口
canal.port = 11111

# 2. ZK 配置（HA 模式）
canal.zkServers = zk1:2181,zk2:2181,zk3:2181
canal.zookeeper.session.timeout.ms = 30000

# 3. 注册信息
canal.register.ip = <服务器真实IP>   # 重要！
canal.register.cluster = production

# 4. Admin 关联
canal.admin.manager = admin-server:8089
canal.admin.port = 11110
canal.admin.user = admin
canal.admin.passwd = <SHA1 加密的密码>

# 5. 实例模式
canal.instance.global.mode = manager
canal.instance.global.spring.xml = classpath:spring/default-instance.xml

# 6. 服务模式
canal.serverMode = kafka              # 生产推荐 Kafka 模式

# 7. Kafka 配置
kafka.bootstrap.servers = kafka1:9092,kafka2:9092,kafka3:9092
kafka.acks = all
kafka.compression.type = lz4
kafka.batch.size = 131072
kafka.retries = 3

# 8. TSDB 配置
canal.instance.tsdb.enable = true

# 9. 告警配置
canal.alarm.mode = email
canal.alarm.email.host = smtp.example.com
canal.alarm.email.to = ops-team@example.com
```

## 2.2 Instance 配置清单

```properties
# instance.properties — 生产配置检查清单

# 1. Slave ID（必须唯一）
canal.instance.mysql.slaveId = 1234

# 2. MySQL 连接
canal.instance.master.address = mysql-master:3306
canal.instance.dbUsername = canal
canal.instance.dbPassword = <密码>
canal.instance.connectionCharset = UTF-8

# 3. GTID 模式（强烈推荐）
canal.instance.gtidon = true

# 4. 过滤规则
canal.instance.filter.regex = order_db\\..*,user_db\\..*
canal.instance.filter.black.regex = .*\\._backup_.*,.*\\._temp_.*

# 5. RingBuffer 配置
canal.instance.memory.buffer.size = 32768
canal.instance.memory.buffer.memunit = 2048

# 6. Kafka Topic 配置
canal.mq.topic = canal_production
canal.mq.dynamicTopic = .*\\.order:canal_order,.*\\.user:canal_user
canal.mq.flatMessage = true
canal.mq.partitionHash = order_db\\.order:order_id,user_db\\.user:user_id
```

# 3. 缓存一致性方案

## 3.1 经典架构：Canal + MQ + 缓存更新

```
                     ┌─────────┐
                     │  MySQL  │
                     └────┬────┘
                          │ Binlog
                     ┌────▼────┐
                     │  Canal  │
                     └────┬────┘
                          │
                    ┌─────▼─────┐
                    │   Kafka   │
                    └─────┬─────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
        ┌─────▼────┐ ┌───▼────┐ ┌───▼────┐
        │ 缓存更新  │ │ ES 更新 │ │ 其他   │
        │ Service  │ │ Service │ │ Service│
        └─────┬────┘ └─────────┘ └─────────┘
              │
        ┌─────▼────┐
        │  Redis   │
        │ (Cache)  │
        └──────────┘
```

## 3.2 缓存同步策略

**策略 1：删缓存（Cache-Aside）**

```java
// 最简单的缓存同步方案：数据库变更后删除缓存
@Component
public class CacheInvalidateConsumer {
    
    private RedisTemplate<String, Object> redis;
    
    @KafkaListener(topics = "canal_production")
    public void onMessage(String messageJson) {
        FlatMessage msg = JSON.parseObject(messageJson, FlatMessage.class);
        
        if (msg.getIsDdl()) return;
        
        String table = msg.getTable();
        
        switch (msg.getType()) {
            case "INSERT":
            case "UPDATE":
            case "DELETE":
                // 删除相关缓存 Key
                for (Map<String, String> row : msg.getData()) {
                    String cacheKey = buildCacheKey(table, row);
                    redis.delete(cacheKey);
                }
                break;
        }
    }
    
    private String buildCacheKey(String table, Map<String, String> row) {
        // 假设主键是 id
        return "cache:" + table + ":" + row.get("id");
    }
}
```

**策略 2：更新缓存（Read-Through/Write-Through）**

```java
// 数据库变更后，从数据库重新加载最新数据到缓存
@Component
public class CacheRefreshConsumer {
    
    @KafkaListener(topics = "canal_production")
    public void onMessage(String messageJson) {
        FlatMessage msg = JSON.parseObject(messageJson, FlatMessage.class);
        
        if (msg.getIsDdl()) return;
        if ("DELETE".equals(msg.getType())) {
            // DELETE: 删除缓存
            for (Map<String, String> row : msg.getData()) {
                String key = buildCacheKey(msg.getTable(), row);
                redis.delete(key);
            }
        } else {
            // INSERT/UPDATE: 更新缓存
            for (Map<String, String> row : msg.getData()) {
                String key = buildCacheKey(msg.getTable(), row);
                // 从数据库重新查询最新数据
                Object latestData = queryFromDatabase(msg.getTable(), row);
                redis.opsForValue().set(key, latestData, 30, TimeUnit.MINUTES);
            }
        }
    }
}
```

**策略 3：延迟双删**

```java
// 解决缓存与数据库最终一致性问题的经典方案
public class DelayedDoubleDelete {
    
    public void updateWithConsistency(String table, Map<String, String> data) {
        // 第一步：先删除缓存
        redis.delete(buildCacheKey(table, data));
        
        // 第二步：更新数据库
        updateDatabase(table, data);
        
        // 第三步：延迟一段时间再次删除缓存
        scheduler.schedule(() -> {
            redis.delete(buildCacheKey(table, data));
        }, 500, TimeUnit.MILLISECONDS);  // 500ms 延迟
    }
}
```

# 4. 数据同步架构

## 4.1 MySQL 异构数据同步

```
场景：将 MySQL 数据同步到 OLAP 数据库（ClickHouse/Doris）

MySQL (OLTP) ──> Canal ──> Kafka ──> Flink ──> ClickHouse (OLAP)
                                                 Doris (OLAP)
```

```sql
-- Flink SQL 处理 Canal 消息并写入 ClickHouse
INSERT INTO clickhouse_user_behavior
SELECT
    data['id'] AS user_id,
    data['action'] AS action,
    CAST(data['ts'] AS TIMESTAMP) AS event_time,
    CAST(data['amount'] AS DECIMAL(10,2)) AS amount
FROM canal_kafka_source
WHERE `table` = 'user_behavior'
  AND type IN ('INSERT', 'UPDATE');
```

## 4.2 微服务数据同步

```
场景：多个微服务之间的数据同步

Service A 写入 MySQL-A ──> Canal ──> Kafka
                                         │
                              ┌──────────┼──────────┐
                              │          │          │
                         Service B   Service C   Service D
                         (订阅订单)   (订阅用户)   (订阅支付)
```

# 5. 安全实践

## 5.1 Canal 账号安全

```sql
-- 最小权限原则
CREATE USER 'canal'@'192.168.1.%' IDENTIFIED BY 'StrongP@ssw0rd';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'192.168.1.%';
FLUSH PRIVILEGES;

-- 限制 Canal 连接 IP
-- 不要在公网暴露 Canal 端口
```

## 5.2 Canal Server 安全配置

```properties
# canal.properties — 启用认证
canal.user = canal_admin
canal.passwd = E10ADC3949BA59ABBE56E057F20F883E

# 启用管理端口认证
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441

# Kafka SASL 认证
kafka.security.protocol = SASL_PLAINTEXT
kafka.sasl.mechanism = PLAIN
kafka.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required username="canal" password="xxx";
```

# 6. 运维规范

## 6.1 日常检查清单

```
每日检查：
├── Canal Server 运行状态（进程是否存在）
├── Instance 运行状态（消费是否正常）
├── 消费延迟（延迟是否 < 5s）
├── RingBuffer 积压（积压是否 < 5000）
├── 磁盘空间（日志和数据目录）
└── JVM 内存和 GC 状态

每周检查：
├── Binlog 保留情况
├── Kafka 消息积压
├── 错误日志分析
├── 性能指标趋势
└── 配置备份确认
```

## 6.2 升级流程

```
1. 【测试环境验证】
   - 搭建与生产一致的 Canal 版本
   - 验证功能正常

2. 【灰度发布】
   - 先升级一台 Standby 节点
   - 观察 24 小时无异常

3. 【切换与升级】
   - 在低峰期进行
   - 停止要升级的 Server
   - 备份配置文件
   - 替换部署包（保留 conf 目录）
   - 启动升级后的 Server
   - 观察消费是否正常恢复

4. 【全量升级】
   - 逐台完成升级
   - 每次间隔 1 天以上
```

## 6.3 回滚方案

```bash
# 1. 保留上一版本的部署包
mv canal canal.bak.$(date +%Y%m%d)

# 2. 如果有配置变更，保留配置备份
cp -r conf conf.bak.$(date +%Y%m%d)

# 3. 快速回滚
./bin/stop.sh
rm -rf canal
mv canal.bak.20240101 canal
./bin/startup.sh
```

# 7. 常见架构模式

## 7.1 模式 1：同构数据同步

```
MySQL (源) ──> Canal ──> MySQL (目标)
用途：读写分离、跨机房同步、数据库迁移
```

## 7.2 模式 2：实时数仓

```
MySQL (OLTP) ──> Canal ──> Kafka ──> Flink ──> 数仓 (Hudi/Iceberg)
用途：实时数据入湖、ODS 层构建
```

## 7.3 模式 3：搜索索引

```
MySQL ──> Canal ──> Kafka ──> Consumer ──> Elasticsearch
用途：商品搜索、日志检索、全文检索
```

## 7.4 模式 4：事件驱动解耦

```
MySQL ──> Canal ──> Kafka ──> 多个下游微服务
用途：订单状态变更通知、库存扣减、积分计算等
```

# 8. 关键经验总结

1. **启用 GTID 模式**：主从切换时自动定位，避免位点错误
2. **使用 Kafka 投递模式**：解耦最彻底，支持多消费者组独立消费
3. **开启 FlatMessage**：消息体积减小 75%，消费端解析更简单
4. **合理设置 RingBuffer**：根据 TPS 调整，避免频繁阻塞
5. **消费端必须实现幂等**：Canal 是 At-Least-Once 语义
6. **按主键分区投递**：保证同一条数据的变更有序
7. **定期监控消费延迟**：延迟 > 60s 就需要关注
8. **配置 Binlog 保留时间足够长**：建议 7-14 天，防止位点过期
9. **使用 Canal Admin 统一管理**：可视化管理多台服务器
10. **做好降级方案**：当 Canal 不可用时，有手动补偿数据的机制
