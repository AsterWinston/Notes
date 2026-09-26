
# 1. Adapter 简介

Canal Adapter 是 Canal 的**数据同步适配器**，负责将 Canal Server 订阅到的增量变更数据同步到各类目标存储系统。它解决了"数据到了 Canal 之后怎么办"的问题。

```
┌──────────┐     ┌──────────────┐     ┌─────────────────────────┐
│  MySQL   │────>│ Canal Server │────>│     Canal Adapter        │
│ (Source) │     │  (Binlog)    │     │                          │
└──────────┘     └──────────────┘     │  ┌───────────────────┐  │
                                      │  │  RDB Adapter      │  │
                                      │  │  (MySQL/Oracle/   │  │
                                      │  │   PostgreSQL等)   │  │
                                      │  ├───────────────────┤  │
                                      │  │  ES Adapter       │  │
                                      │  │  (Elasticsearch)  │  │
                                      │  ├───────────────────┤  │
                                      │  │  HBase Adapter    │  │
                                      │  ├───────────────────┤  │
                                      │  │  Logger Adapter   │  │
                                      │  │  (日志/调试)      │  │
                                      │  ├───────────────────┤  │
                                      │  │  Custom Adapter   │  │
                                      │  │  (SPI 扩展)       │  │
                                      │  └───────────────────┘  │
                                      └─────────────────────────┘
```

# 2. 架构设计

## 2.1 Adapter 核心组件

```
┌─────────────────────────────────────────────────────────┐
│                   Canal Adapter                          │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │              CanalClient (订阅层)                   │ │
│  │  ┌─────────┐  ┌──────────┐  ┌──────────────────┐ │ │
│  │  │ TCP     │  │  Kafka   │  │ RocketMQ         │ │ │
│  │  │ 直连    │  │ Consumer │  │ Consumer         │ │ │
│  │  └─────────┘  └──────────┘  └──────────────────┘ │ │
│  └───────────────────────┬───────────────────────────┘ │
│                          │                              │
│  ┌───────────────────────▼───────────────────────────┐ │
│  │            OutAdapter (适配层)                      │ │
│  │  ┌─────────┬─────────┬─────────┬────────────────┐ │ │
│  │  │  RDB    │   ES    │  HBase  │  Logger/Other  │ │ │
│  │  │ Adapter │ Adapter │ Adapter │  Adapter       │ │ │
│  │  └─────────┴─────────┴─────────┴────────────────┘ │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │            SPI 扩展机制                             │ │
│  │  支持用户自定义实现 OutAdapter 接口                  │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## 2.2 核心接口

```java
/**
 * 外部适配器接口 — 所有 Adapter 的顶层抽象
 */
public interface OutAdapter {
    
    /**
     * 启动适配器
     */
    void start();
    
    /**
     * 停止适配器
     */
    void stop();
    
    /**
     * 同步数据
     * @param dmls DML 数据列表
     */
    void sync(List<Dml> dmls);
    
    /**
     * 获取目标类型
     */
    String getDestination();
}
```

# 3. 消费模式

## 3.1 TCP 模式（直连 Canal Server）

```yaml
canal.conf:
  mode: tcp
  canalServerHost: 127.0.0.1:11111
  # 可选：ZK 模式
  # zookeeperHosts: 127.0.0.1:2181
```

## 3.2 Kafka 模式

```yaml
canal.conf:
  mode: kafka
  kafka.bootstrap.servers: 127.0.0.1:9092
  kafka.enable.auto.commit: false
  kafka.auto.commit.interval.ms: 1000
  kafka.auto.offset.reset: latest
  kafka.request.timeout.ms: 40000
  kafka.session.timeout.ms: 30000
  kafka.isolation.level: read_committed
  kafka.max.poll.records: 1000
```

## 3.3 RocketMQ 模式

```yaml
canal.conf:
  mode: rocketMQ
  rocketmq.namespace:
  rocketmq.namesrv.addr: 127.0.0.1:9876
  rocketmq.batch.size: 1000
  rocketmq.enable.message.trace: false
  rocketmq.customized.trace.topic:
  rocketmq.access.channel:
  rocketmq.subscribe.filter:
```

# 4. 配置文件详解

## 4.1 主配置文件 (application.yml)

```yaml
server:
  port: 8081

spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8

canal.conf:
  # ========== 数据源 ==========
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/source_db?useUnicode=true
      username: canal
      password: canal_password
  
  # ========== Canal 连接 ==========
  canalServerHost: 127.0.0.1:11111
  mode: tcp
  
  # ========== 批量处理 ==========
  flatMessage: true          # 使用扁平化消息
  batchSize: 500            # 每批 500 条
  syncBatchSize: 1000       # 同步批量大小
  retries: 3                # 重试次数
  timeout: 60000            # 超时时间(ms)
  accessKey:                # 访问密钥
  secretKey:
  
  # ========== Adapter 配置 ==========
  canalAdapters:
    - instance: example       # Canal Instance 名称
      groups:
        - groupId: g1         # 分组 ID
          outerAdapters:
            - name: logger    # 日志适配器（调试用）
            - name: rdb       # 关系型数据库适配器
```

## 4.2 表映射配置 (rdb/xxx.yml)

每个目标表都需要一个 YAML 配置文件：

```yaml
# conf/rdb/mytest_user.yml
dataSourceKey: defaultDS      # 数据源 key
destination: example          # Canal Instance
groupId: g1                   # 分组 ID
outerAdapterKey: mysql1       # 目标库 key

# 是否使用 GTID 模式
# enableGTID: true

# 映射配置
targetDb: target_db           # 目标库名
targetTable: user             # 目标表名
targetPk: id                  # 目标主键

# 映射模式：ALL（全量映射）/ COLUMN（按列映射）/ EXCLUDE（排除）
mapMode: ALL

# 字段映射（mapMode=COLUMN 时生效）
# columnMapping:
#   source_column: target_column

# 提交批次大小
commitBatch: 3000

# 是否启用 etl（全量同步）
# etlCondition: "where id > 0"

# 条件过滤
# where: "status != 'deleted'"

# 源表映射配置
sourceDb: source_db           # 源库名
sourceTable: users            # 源表名
```

# 5. 数据处理流程

## 5.1 Dml 数据模型

```java
public class Dml {
    
    private String destination;     // Canal Instance
    private String database;        // 库名
    private String table;           // 表名
    private String type;            // INSERT / UPDATE / DELETE / DDL
    private Long es;                // binlog 执行时间
    private Long ts;                // 处理时间戳
    private String sql;             // DDL SQL
    
    // 列元数据
    private Map<String, String> mysqlType;
    private Map<String, Integer> sqlType;
    
    // 行数据
    private List<Map<String, String>> data;  // 变更后数据
    private List<Map<String, String>> old;   // 变更前数据
    
    private List<String> pkNames;            // 主键列名
    private Boolean isDdl;
}
```

## 5.2 同步流程

```java
public abstract class AbstractOutAdapter implements OutAdapter {
    
    @Override
    public void sync(List<Dml> dmls) {
        if (dmls == null || dmls.isEmpty()) {
            return;
        }
        
        for (Dml dml : dmls) {
            try {
                // 1. 查找对应的映射配置
                MappingConfig config = findMappingConfig(dml.getDatabase(), dml.getTable());
                if (config == null) {
                    logger.warn("no mapping config for {}.{}", dml.getDatabase(), dml.getTable());
                    continue;
                }
                
                // 2. DDL 特殊处理
                if (dml.getIsDdl() != null && dml.getIsDdl()) {
                    handleDdl(config, dml);
                    continue;
                }
                
                // 3. 按操作类型处理
                switch (dml.getType().toUpperCase()) {
                    case "INSERT":
                        syncInsert(config, dml);
                        break;
                    case "UPDATE":
                        syncUpdate(config, dml);
                        break;
                    case "DELETE":
                        syncDelete(config, dml);
                        break;
                    default:
                        logger.warn("unknown dml type: {}", dml.getType());
                }
                
            } catch (Exception e) {
                logger.error("sync failed for {}.{}: {}", 
                    dml.getDatabase(), dml.getTable(), e.getMessage());
                // 失败处理：重试 or 记录 or 抛出
                handleSyncError(dml, e);
            }
        }
    }
    
    protected abstract void syncInsert(MappingConfig config, Dml dml);
    protected abstract void syncUpdate(MappingConfig config, Dml dml);
    protected abstract void syncDelete(MappingConfig config, Dml dml);
    protected abstract void handleDdl(MappingConfig config, Dml dml);
}
```

# 6. 同步模式

## 6.1 增量同步（默认）

只同步 binlog 中的增量变更数据：

```yaml
# Adapter 默认就是增量模式，不需要特殊配置
# 启动即开始消费 Canal Server 的增量数据
```

## 6.2 ETL 全量同步

在配置中添加 `etlCondition` 即可启用全量同步：

```yaml
# conf/rdb/mytest_user.yml
dataSourceKey: defaultDS
outerAdapterKey: mysql1
targetDb: target_db
targetTable: user

# ETL 全量同步条件
etlCondition: "where id > 0"
```

```bash
# 通过 REST API 触发全量同步
# ETL (Extract, Transform, Load)
curl http://localhost:8081/etl/rdb/mytest_user.yml \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}'

# 查看 ETL 进度
curl http://localhost:8081/etl/rdb/mytest_user.yml
```

## 6.3 ETL 实现原理

```java
public class EtlService {
    
    /**
     * ETL 全量同步
     * 从源库全量查询数据，写入目标库
     */
    public EtlResult etl(String configFile, Map<String, String> params) {
        
        MappingConfig config = loadConfig(configFile);
        
        // 1. 从源库查询全量数据
        String sourceSql = buildSourceQuery(config);
        List<Map<String, Object>> sourceData = sourceJdbcTemplate.queryForList(sourceSql);
        
        long total = sourceData.size();
        long success = 0;
        
        // 2. 分批写入目标库
        int batchSize = config.getCommitBatch();
        for (int i = 0; i < sourceData.size(); i += batchSize) {
            int end = Math.min(i + batchSize, sourceData.size());
            List<Map<String, Object>> batch = sourceData.subList(i, end);
            
            try {
                writeToTarget(config, batch);
                success += batch.size();
            } catch (Exception e) {
                logger.error("etl batch write failed at offset {}", i, e);
                return EtlResult.fail(i, total, e.getMessage());
            }
        }
        
        return EtlResult.success(total, success);
    }
    
    private String buildSourceQuery(MappingConfig config) {
        StringBuilder sql = new StringBuilder();
        sql.append("SELECT * FROM ");
        sql.append(config.getSourceDb()).append(".").append(config.getSourceTable());
        
        if (StringUtils.isNotEmpty(config.getEtlCondition())) {
            sql.append(" ").append(config.getEtlCondition());
        }
        
        return sql.toString();
    }
}
```

# 7. 数据过滤与转换

## 7.1 条件过滤 (where)

```yaml
# 只同步 status != 'deleted' 的数据
sourceDb: source_db
sourceTable: users
targetDb: target_db
targetTable: users
where: "status != 'deleted'"
```

## 7.2 字段映射 (columnMapping)

```yaml
# 将源表的 user_name 映射到目标表的 name
sourceDb: source_db
sourceTable: users
targetDb: target_db
targetTable: users
mapMode: COLUMN
columnMapping:
  user_name: name         # 源列名 -> 目标列名
  create_time: created_at
  update_time: updated_at
```

## 7.3 字段排除 (EXCLUDE)

```yaml
# 排除某些列不同步
sourceDb: source_db
sourceTable: users
targetDb: target_db
targetTable: users
mapMode: EXCLUDE
excludeColumns:
  - password_hash
  - secret_key
  - internal_flag
```

# 8. 批量与性能配置

```yaml
canal.conf:
  # 批量大小（每次从 Canal 拉取的消息数）
  batchSize: 500
  
  # 同步批量大小（每次写入目标的记录数）
  syncBatchSize: 1000
  
  # 重试次数
  retries: 3
  
  # 超时时间（毫秒）
  timeout: 60000
  
  # 使用线程池并行处理
  syncThreadSize: 4
  
  # 消费线程数
  consumerThreadSize: 2
```

## 8.1 性能优化建议

```java
// Adapter 内部使用线程池处理批次
public class BatchSyncExecutor {
    
    private ThreadPoolExecutor executor;
    
    public BatchSyncExecutor(int threadSize) {
        executor = new ThreadPoolExecutor(
            threadSize,
            threadSize,
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 队列满了用调用线程处理
        );
    }
    
    /**
     * 按表分组，并行同步
     */
    public void syncByTable(List<Dml> dmls) {
        // 按 (database, table) 分组
        Map<String, List<Dml>> grouped = dmls.stream()
            .collect(Collectors.groupingBy(d -> d.getDatabase() + "." + d.getTable()));
        
        // 并行处理每张表的数据（表与表之间无依赖）
        List<Future<?>> futures = new ArrayList<>();
        for (Map.Entry<String, List<Dml>> entry : grouped.entrySet()) {
            futures.add(executor.submit(() -> {
                syncTable(entry.getKey(), entry.getValue());
            }));
        }
        
        // 等待所有表同步完成
        for (Future<?> future : futures) {
            try {
                future.get();
            } catch (Exception e) {
                logger.error("sync table failed", e);
                throw new SyncException(e);
            }
        }
    }
}
```

# 9. 监控与管理 API

```bash
# Adapter 管理接口（默认端口 8081）

# 查看所有任务状态
curl http://localhost:8081/syncStatus

# 手动触发 ETL
curl http://localhost:8081/etl/rdb/mytest_user.yml -X POST

# 查看 ETL 进度
curl http://localhost:8081/etl/rdb/mytest_user.yml

# 查询 Adapter 版本信息
curl http://localhost:8081/version
```

# 10. SPI 扩展机制

Canal Adapter 支持通过 Java SPI 扩展自定义适配器：

```java
// 1. 实现 OutAdapter 接口
@SPI("my-custom")
public class MyCustomAdapter extends AbstractOutAdapter {
    
    @Override
    public void start() {
        // 初始化连接
        connectToCustomStore();
    }
    
    @Override
    public void stop() {
        // 清理连接
        disconnect();
    }
    
    @Override
    protected void syncInsert(MappingConfig config, Dml dml) {
        for (Map<String, String> row : dml.getData()) {
            // 写入自定义存储
            customStore.insert(dml.getTable(), row);
        }
    }
    
    @Override
    protected void syncUpdate(MappingConfig config, Dml dml) {
        for (int i = 0; i < dml.getData().size(); i++) {
            Map<String, String> newData = dml.getData().get(i);
            Map<String, String> oldData = dml.getOld().get(i);
            customStore.update(dml.getTable(), oldData, newData);
        }
    }
    
    @Override
    protected void syncDelete(MappingConfig config, Dml dml) {
        for (Map<String, String> row : dml.getData()) {
            customStore.delete(dml.getTable(), row);
        }
    }
}

// 2. 在 META-INF/services 下注册
// 文件：META-INF/services/com.alibaba.otter.canal.adapter.out.OutAdapter
// 内容：com.example.MyCustomAdapter

// 3. 配置使用自定义适配器
// application.yml
// canalAdapters:
//   - instance: example
//     groups:
//       - groupId: g1
//         outerAdapters:
//           - name: my-custom
```
