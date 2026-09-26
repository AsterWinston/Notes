
# 1. 概述

除了 RDB 和 Elasticsearch，Canal Adapter 还支持同步到 HBase、Kudu、以及通过自定义 SPI 扩展到任意目标系统。

```
                     ┌─────────────┐
                     │ Canal Server│
                     └──────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        ┌─────▼────┐ ┌─────▼────┐ ┌─────▼─────┐
        │  RDB     │ │   ES     │ │  HBase    │
        │ Adapter  │ │ Adapter  │ │  Adapter  │
        └──────────┘ └──────────┘ └───────────┘
              │             │             │
        ┌─────▼────┐ ┌─────▼────┐ ┌─────▼─────┐
        │  HBase   │ │  Kudu    │ │  Logger   │
        │  Adapter │ │  Adapter │ │  Adapter  │
        └──────────┘ └──────────┘ └───────────┘
```

# 2. HBase Adapter

## 2.1 配置

```yaml
# application.yml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: hbase
              properties:
                hbase.zookeeper.quorum: 127.0.0.1
                hbase.zookeeper.property.clientPort: 2181
                zookeeper.znode.parent: /hbase
```

```yaml
# conf/hbase/mytest_user.yml
dataSourceKey: defaultDS
destination: example
groupId: g1
hbaseMapping:
  mode: STRING                         # NATIVE / STRING / PHOENIX
  
  # STRING 模式：所有值按字符串存储
  hbaseTable: test:user                # namespace:tablename
  
  # 行键设计
  rowKey: id,type                      # HBase RowKey 由指定列的值组成
  
  # 列族配置
  family: cf                           # 默认列族
  
  # 字段映射
  columns:
    - column: name
      qualifier: n                     # 列限定符（短名节省存储）
    - column: email
      qualifier: e
    - column: age
      qualifier: a
    - column: status
      qualifier: s
    
  # 或使用 etlCondition 进行全量同步
  # etlCondition: "where id > 10000"
  
  commitBatch: 5000
```

## 2.2 HBase 行键设计策略

```java
public class HBaseRowKeyDesigner {
    
    /**
     * 设计 RowKey 的各种策略
     */
    
    // 策略 1: 简单主键
    // rowKey: {id}                    → 适合按 ID 查询
    
    // 策略 2: 时间反转 + 主键
    // rowKey: {Long.MAX_VALUE - ts}_{id}  → 最新数据排前面
    
    // 策略 3: 分区前缀 + 主键（防热点）
    // rowKey: {id % 100}_{id}         → 100个分区
    
    // 策略 4: 复合键
    // rowKey: {region}_{date}_{id}    → 按时间和地域组织
    
    /**
     * 生成 RowKey
     */
    public static byte[] generateRowKey(String config, Map<String, String> row) {
        String[] keyParts = config.split(",");
        
        StringBuilder key = new StringBuilder();
        for (String part : keyParts) {
            String value = row.get(part.trim());
            if (value != null) {
                // 定长填充，保证排序一致性
                key.append(leftPad(value, 20, '0'));
            }
        }
        
        return Bytes.toBytes(key.toString());
    }
    
    /**
     * 盐值前缀（防热点）
     */
    public static byte[] generateSaltedRowKey(String config, Map<String, String> row, 
                                               int saltSize) {
        String[] keyParts = config.split(",");
        String primaryKey = row.get(keyParts[0].trim());
        
        // 加盐
        int salt = Math.abs(primaryKey.hashCode()) % saltSize;
        
        StringBuilder key = new StringBuilder();
        key.append(String.format("%03d", salt));
        key.append("_");
        
        for (String part : keyParts) {
            String value = row.get(part.trim());
            key.append(value != null ? value : "");
        }
        
        return Bytes.toBytes(key.toString());
    }
}
```

## 2.3 HBase 同步实现

```java
public class HBaseSyncService {
    
    private Connection hbaseConnection;
    private Map<String, Table> tableCache = new ConcurrentHashMap<>();
    
    /**
     * INSERT 同步
     */
    public void syncInsert(HBaseMappingConfig config, Dml dml) {
        Table table = getTable(config.getHbaseTable());
        
        List<Put> puts = new ArrayList<>();
        
        for (Map<String, String> row : dml.getData()) {
            byte[] rowKey = HBaseRowKeyDesigner.generateRowKey(config.getRowKey(), row);
            Put put = new Put(rowKey);
            
            // 添加列数据
            for (ColumnMapping col : config.getColumns()) {
                String value = row.get(col.getColumn());
                if (value != null) {
                    put.addColumn(
                        Bytes.toBytes(config.getFamily()),
                        Bytes.toBytes(col.getQualifier()),
                        Bytes.toBytes(value)
                    );
                }
            }
            
            puts.add(put);
        }
        
        // 批量写入
        table.put(puts);
        logger.debug("hbase insert {} rows to {}", puts.size(), config.getHbaseTable());
    }
    
    /**
     * UPDATE 同步
     */
    public void syncUpdate(HBaseMappingConfig config, Dml dml) {
        // HBase 的 Put 天然是 upsert
        // UPDATE 等同于 INSERT（覆盖写入）
        syncInsert(config, dml);
    }
    
    /**
     * DELETE 同步
     */
    public void syncDelete(HBaseMappingConfig config, Dml dml) {
        Table table = getTable(config.getHbaseTable());
        
        List<Delete> deletes = new ArrayList<>();
        
        for (Map<String, String> row : dml.getData()) {
            byte[] rowKey = HBaseRowKeyDesigner.generateRowKey(config.getRowKey(), row);
            Delete delete = new Delete(rowKey);
            
            // 可选：删除特定列族或列
            if (config.isDeleteFamily()) {
                delete.addFamily(Bytes.toBytes(config.getFamily()));
            } else {
                for (ColumnMapping col : config.getColumns()) {
                    delete.addColumn(
                        Bytes.toBytes(config.getFamily()),
                        Bytes.toBytes(col.getQualifier())
                    );
                }
            }
            
            deletes.add(delete);
        }
        
        table.delete(deletes);
        logger.debug("hbase delete {} rows from {}", deletes.size(), config.getHbaseTable());
    }
    
    private Table getTable(String tableName) {
        return tableCache.computeIfAbsent(tableName, tn -> {
            try {
                return hbaseConnection.getTable(TableName.valueOf(tn));
            } catch (IOException e) {
                throw new RuntimeException("failed to get hbase table: " + tn, e);
            }
        });
    }
}
```

# 3. Kudu Adapter

```yaml
# application.yml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: kudu
              properties:
                kudu.master.address: 127.0.0.1:7051
                kudu.worker.count: 2
                kudu.operation.timeout.ms: 30000
                kudu.admin.operation.timeout.ms: 30000
```

```yaml
# conf/kudu/mytest_user.yml
dataSourceKey: defaultDS
destination: example
groupId: g1
kuduMapping:
  kuduTable: impala::default.user_kudu
  
  # 主键列
  targetPk: id
  
  # 列映射
  columns:
    - column: id
      targetColumn: user_id
      type: long
    - column: name
      targetColumn: user_name
      type: string
    - column: age
      targetColumn: user_age
      type: int
    - column: status
      targetColumn: status_flag
      type: string
  
  commitBatch: 5000
```

# 4. Logger Adapter（调试工具）

Logger Adapter 是最简单的适配器，将收到的数据打印到日志：

```yaml
# application.yml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: logger            # 日志适配器
```

```java
// Logger Adapter 实现
public class LoggerOutAdapter extends AbstractOutAdapter {
    
    @Override
    protected void syncInsert(MappingConfig config, Dml dml) {
        log("INSERT", dml);
    }
    
    @Override
    protected void syncUpdate(MappingConfig config, Dml dml) {
        log("UPDATE", dml);
    }
    
    @Override
    protected void syncDelete(MappingConfig config, Dml dml) {
        log("DELETE", dml);
    }
    
    private void log(String type, Dml dml) {
        logger.info("[{}] [{}.{}] pk={} data={} old={}",
            type,
            dml.getDatabase(), dml.getTable(),
            dml.getPkNames(),
            dml.getData(),
            dml.getOld());
    }
}
```

**日志输出示例**：

```
[INSERT] [test.users] pk=[id] data=[{id=1, name=Alice, email=alice@example.com}] old=[]
[UPDATE] [test.users] pk=[id] data=[{id=1, name=Bob, status=0}] old=[{name=Alice}]
[DELETE] [test.users] pk=[id] data=[{id=1}] old=[]
```

# 5. 自定义 Adapter 开发

## 5.1 实现 OutAdapter 接口

```java
// 自定义 MongoDB Adapter 示例
@SPI("mongodb")
public class MongoOutAdapter extends AbstractOutAdapter {
    
    private MongoClient mongoClient;
    private Map<String, MongoCollection<Document>> collections = new HashMap<>();
    
    @Override
    public void start() {
        // 读取配置
        String connectionString = getProperties().getProperty("mongodb.uri");
        String databaseName = getProperties().getProperty("mongodb.database");
        
        mongoClient = MongoClients.create(connectionString);
        this.database = mongoClient.getDatabase(databaseName);
        
        logger.info("MongoDB adapter started");
    }
    
    @Override
    protected void syncInsert(MappingConfig config, Dml dml) {
        MongoCollection<Document> collection = getCollection(dml.getTable());
        
        List<Document> documents = new ArrayList<>();
        for (Map<String, String> row : dml.getData()) {
            Document doc = new Document();
            for (Map.Entry<String, String> entry : row.entrySet()) {
                doc.append(entry.getKey(), entry.getValue());
            }
            documents.add(doc);
        }
        
        collection.insertMany(documents);
        logger.info("MongoDB insert {} docs into {}", documents.size(), dml.getTable());
    }
    
    @Override
    protected void syncUpdate(MappingConfig config, Dml dml) {
        MongoCollection<Document> collection = getCollection(dml.getTable());
        
        for (int i = 0; i < dml.getData().size(); i++) {
            Map<String, String> newRow = dml.getData().get(i);
            Map<String, String> oldRow = dml.getOld().get(i);
            
            // 构建更新条件（基于主键）
            Bson filter = buildPkFilter(dml.getPkNames(), newRow);
            
            // 构建更新内容
            Document update = new Document("$set", new Document(newRow));
            
            collection.updateOne(filter, update, new UpdateOptions().upsert(true));
        }
    }
    
    @Override
    protected void syncDelete(MappingConfig config, Dml dml) {
        MongoCollection<Document> collection = getCollection(dml.getTable());
        
        List<Bson> filters = new ArrayList<>();
        for (Map<String, String> row : dml.getData()) {
            filters.add(buildPkFilter(dml.getPkNames(), row));
        }
        
        collection.deleteMany(Filters.or(filters));
    }
    
    private Bson buildPkFilter(List<String> pkNames, Map<String, String> row) {
        Document filter = new Document();
        for (String pk : pkNames) {
            filter.append(pk, row.get(pk));
        }
        return filter;
    }
    
    private MongoCollection<Document> getCollection(String tableName) {
        return collections.computeIfAbsent(tableName, database::getCollection);
    }
    
    @Override
    public void stop() {
        if (mongoClient != null) {
            mongoClient.close();
        }
    }
}
```

## 5.2 注册 SPI

```
# META-INF/services/com.alibaba.otter.canal.adapter.out.OutAdapter
com.example.MongoOutAdapter
```

## 5.3 配置使用

```yaml
# application.yml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: mongodb
              properties:
                mongodb.uri: mongodb://localhost:27017
                mongodb.database: canal_sync
```

# 6. Adapter 的 retry 和容错

## 6.1 重试策略

```java
public class SyncRetryHandler {
    
    private static final int MAX_RETRIES = 3;
    private static final long BASE_DELAY = 1000;  // 1s
    
    /**
     * 带重试的同步执行
     */
    public void executeWithRetry(Runnable syncTask) {
        int retries = 0;
        
        while (true) {
            try {
                syncTask.run();
                return;  // 成功
                
            } catch (Exception e) {
                retries++;
                
                if (retries >= MAX_RETRIES) {
                    // 重试耗尽
                    logger.error("sync failed after {} retries", MAX_RETRIES, e);
                    throw new RetryExhaustedException("sync failed", e);
                }
                
                // 指数退避等待
                long delay = BASE_DELAY * (1L << (retries - 1));  // 1s, 2s, 4s
                logger.warn("sync failed, retry {}/{} after {}ms",
                    retries, MAX_RETRIES, delay, e);
                
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
    }
}
```

## 6.2 失败记录

```java
// 将同步失败的记录保存到"失败队列表"中，供后续补偿
public class FailedRecordHandler {
    
    /**
     * 将失败的 DML 记录保存到失败队列表
     */
    public void saveFailedRecord(Dml dml, String error) {
        
        String sql = "INSERT INTO canal_sync_failed_record " +
                     "(destination, database_name, table_name, type, data_json, old_json, " +
                     " error_message, created_time) " +
                     "VALUES (?, ?, ?, ?, ?, ?, ?, NOW())";
        
        jdbcTemplate.update(sql,
            dml.getDestination(),
            dml.getDatabase(),
            dml.getTable(),
            dml.getType(),
            JSON.toJSONString(dml.getData()),
            JSON.toJSONString(dml.getOld()),
            error
        );
    }
    
    /**
     * 补偿同步失败的记录
     */
    public void compensate() {
        List<FailedRecord> records = queryPendingRecords();
        
        for (FailedRecord record : records) {
            try {
                Dml dml = reconstructDml(record);
                adapter.sync(Collections.singletonList(dml));
                
                // 标记为已补偿
                markCompensated(record.getId());
            } catch (Exception e) {
                // 增加失败计数
                incrementFailedCount(record.getId());
                
                if (record.getFailedCount() >= 10) {
                    // 失败太多次，标记为人工处理
                    markManualRequired(record.getId());
                }
            }
        }
    }
}
```

# 7. MQ 数据源模式下的 Adapter

除了 TCP 直连 Canal Server，Adapter 也支持从 MQ 消费数据：

## 7.1 Kafka 消费模式

```yaml
canal.conf:
  mode: kafka
  kafka.bootstrap.servers: 127.0.0.1:9092
  kafka.enable.auto.commit: false
  kafka.auto.offset.reset: latest
  kafka.max.poll.records: 500
  
  canalDestinations:
    - instance: example
      topic: example              # Kafka Topic
      groupId: group-1
      
  canalAdapters:
    - instance: example
      groups:
        - groupId: group-1
          outerAdapters:
            - name: rdb
              key: mysql1
              properties:
                # ... 目标数据库配置
```

## 7.2 RocketMQ 消费模式

```yaml
canal.conf:
  mode: rocketMQ
  rocketmq.namesrv.addr: 127.0.0.1:9876
  rocketmq.batch.size: 1000
  
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: rdb
              key: mysql1
              properties:
                # ... 
```

# 8. 多目标同步

Canal Adapter 支持将同一份数据同步到多个目标：

```yaml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: rdb                    # 同步到 MySQL
              key: mysql1
              properties:
                jdbc.driverClassName: com.mysql.cj.jdbc.Driver
                jdbc.url: jdbc:mysql://127.0.0.1:3306/target_db
                jdbc.username: root
                jdbc.password: root
            
            - name: es                     # 同步到 ES
              hosts: 127.0.0.1:9200
              properties:
                cluster.name: elasticsearch
            
            - name: logger                 # 同时输出日志（调试用）
```

# 9. Adapter 监控

```bash
# Adapter 内置的监控接口
curl http://localhost:8081/syncStatus

# 返回示例：
{
  "code": 20000,
  "data": {
    "destination": "example",
    "groupStatus": [
      {
        "groupId": "g1",
        "outerAdapters": [
          {
            "name": "rdb",
            "status": "running",
            "syncCount": 150000,
            "errorCount": 3,
            "lastSyncTime": "2024-01-01 10:30:00"
          },
          {
            "name": "es", 
            "status": "running",
            "syncCount": 150000,
            "errorCount": 0,
            "lastSyncTime": "2024-01-01 10:30:00"
          }
        ]
      }
    ]
  }
}
```
