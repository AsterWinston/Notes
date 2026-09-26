
# 1. MetaManager 概述

MetaManager 是 Canal 的**元数据管理器**，核心职责是管理 Canal 的消费位点（Cursor），确保 Canal 在重启、HA 切换等场景下能够从正确的位置继续消费 Binlog，不丢数据、不重复数据。

```
┌─────────────────────────────────────────────────────┐
│                  MetaManager                         │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  位点读取    │  │  位点更新    │  │  位点持久化  │ │
│  │ getCursor() │  │updateCursor │  │ (ZK/File)   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                     │
│  存储的元数据：                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  {                                          │   │
│  │    "identity": {                            │   │
│  │      "slaveId": 1234,                       │   │
│  │      "sourceAddress": "192.168.1.100:3306"  │   │
│  │    },                                       │   │
│  │    "postion": {                             │   │
│  │      "journalName": "mysql-bin.000003",      │   │
│  │      "position": 45678901,                  │   │
│  │      "timestamp": 1672545600000,            │   │
│  │      "gtid": "uuid:1-100"                  │   │
│  │    }                                        │   │
│  │  }                                          │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

# 2. 核心接口与类层次

## 2.1 接口定义

```java
public interface CanalMetaManager extends CanalLifeCycle {
    
    /**
     * 获取当前消费位点
     * @return 当前的 LogPosition，可能为 null
     */
    Position getCursor();
    
    /**
     * 更新消费位点
     * @param position 新的位点信息
     */
    void updateCursor(Position position);
    
    /**
     * 获取客户端订阅信息
     */
    ClientIdentity getClientIdentity();
}
```

## 2.2 关键数据结构

```java
// LogPosition — 完整的位点信息
public class LogPosition extends Position {
    private LogIdentity identity;      // 身份标识
    private EntryPosition postion;     // 位点详情
    
    // 构造函数：从 journal + position 构建
    public LogPosition(String journalName, Long position) {
        this.postion = new EntryPosition(journalName, position);
    }
}

// LogIdentity — 身份标识（确认数据源）
public class LogIdentity {
    private Long slaveId;               // Canal 模拟的 slave id
    private InetSocketAddress sourceAddress;  // MySQL Master 地址
}

// EntryPosition — 具体的位点信息
public class EntryPosition extends Position {
    private String journalName;   // Binlog 文件名
    private Long position;        // Binlog 文件内的偏移量
    private Long timestamp;       // 位点的时间戳
    private String gtid;          // GTID 集合（如 uuid:1-100）
    private Long serverId;        // MySQL Server ID
    private boolean included;     // 起始位点是否包含（true=>=，false=>）
}
```

## 2.3 实现类层次

```
CanalMetaManager (接口)
    │
    ├── MemoryMetaManager (内存模式，不持久化)
    │
    ├── FileMetaManager (本地文件持久化)
    │
    ├── ZooKeeperMetaManager (ZK 持久化)
    │
    └── PeriodMixedMetaManager (定时混合持久化)
```

# 3. MemoryMetaManager — 内存模式

最简单的实现，仅在内存中维护位点，适用于开发测试环境：

```java
public class MemoryMetaManager extends AbstractCanalLifeCycle 
        implements CanalMetaManager {

    // 内存中的位点
    private Position cursor;
    
    @Override
    public Position getCursor() {
        return cursor;
    }
    
    @Override
    public void updateCursor(Position position) {
        this.cursor = position;
    }
    
    // 问题：重启后位点丢失，会从最新位置开始
}
```

# 4. FileMetaManager — 文件持久化

将位点信息持久化到本地文件系统：

```java
public class FileMetaManager extends AbstractCanalLifeCycle 
        implements CanalMetaManager {

    // 位点文件存储目录
    private String dataDir;
    
    // 位点文件路径
    private File cursorFile;
    
    // 当前位点
    private Position cursor;
    
    @Override
    public void start() {
        super.start();
        
        // 构建位点文件路径
        // 格式: ${dataDir}/destinations/${destination}/cursor.dat
        cursorFile = new File(dataDir, "cursor.dat");
        
        // 如果文件存在，从文件中恢复位点
        if (cursorFile.exists()) {
            cursor = loadCursorFromFile();
        }
    }
    
    @Override
    public void updateCursor(Position position) {
        this.cursor = position;
        // 异步刷盘（平衡性能和安全）
        flushToFileAsync(position);
    }
    
    /**
     * 从文件加载位点
     */
    private Position loadCursorFromFile() {
        try {
            byte[] data = Files.readAllBytes(cursorFile.toPath());
            String json = new String(data, StandardCharsets.UTF_8);
            
            // 支持两种格式：
            // 旧格式: {"journalName":"mysql-bin.000003","position":45678}
            // 新格式: {"identity":{...},"postion":{"journalName":...,"position":...}}
            
            if (json.contains("identity")) {
                return JsonUtils.unmarshalFromString(json, LogPosition.class);
            } else {
                // 兼容旧格式
                EntryPosition entryPosition = JsonUtils.unmarshalFromString(
                    json, EntryPosition.class);
                return new LogPosition(entryPosition);
            }
        } catch (IOException e) {
            logger.error("failed to load cursor from file", e);
            return null;
        }
    }
    
    /**
     * 异步刷盘到文件
     */
    private void flushToFileAsync(Position position) {
        FlushTask task = new FlushTask(cursorFile, position);
        flushExecutor.submit(task);
    }
    
    private static class FlushTask implements Runnable {
        private final File file;
        private final Position position;
        
        @Override
        public void run() {
            try {
                String json = JsonUtils.marshalToString(position);
                Files.write(file.toPath(), json.getBytes(StandardCharsets.UTF_8));
            } catch (IOException e) {
                logger.error("failed to flush cursor to file", e);
            }
        }
    }
}
```

**文件内容示例**：

```json
{
  "identity": {
    "slaveId": 1234,
    "sourceAddress": {
      "address": "192.168.1.100",
      "port": 3306
    }
  },
  "postion": {
    "journalName": "mysql-bin.000003",
    "position": 45678901,
    "timestamp": 1672545600000,
    "gtid": "3e11fa47-71ca-11e1-9e33-c80aa9429562:1-150",
    "included": false
  }
}
```

# 5. ZooKeeperMetaManager — ZK 持久化

HA 模式下，位点信息存储在 ZooKeeper 上：

```java
public class ZooKeeperMetaManager extends AbstractCanalLifeCycle 
        implements CanalMetaManager {

    // ZooKeeper 客户端
    private CuratorFramework zkClient;
    
    // ZK 路径
    // /otter/canal/destinations/{destination}/cursor
    
    private String cursorPath;
    private Position cursor;
    
    @Override
    public void start() {
        super.start();
        
        cursorPath = ZKPaths.CANAL_DESTINATION_ROOT + "/" 
                     + destination + "/cursor";
        
        // 从 ZK 加载位点
        if (zkClient.checkExists().forPath(cursorPath) != null) {
            byte[] data = zkClient.getData().forPath(cursorPath);
            if (data != null && data.length > 0) {
                cursor = parseCursorData(data);
                logger.info("loaded cursor from zk: {}", cursor);
            }
        }
        
        // 监听位点变化
        watchCursorChanges();
    }
    
    @Override
    public void updateCursor(Position position) {
        this.cursor = position;
        
        try {
            // 将位点序列化并写入 ZK
            byte[] data = serializePosition(position);
            
            if (zkClient.checkExists().forPath(cursorPath) != null) {
                zkClient.setData().forPath(cursorPath, data);
            } else {
                zkClient.create()
                        .creatingParentsIfNeeded()
                        .forPath(cursorPath, data);
            }
        } catch (Exception e) {
            logger.error("failed to update cursor to zk", e);
        }
    }
    
    /**
     * 位点序列化
     */
    private byte[] serializePosition(Position position) {
        if (position instanceof LogPosition) {
            return JsonUtils.marshalToString(position).getBytes();
        } else {
            return JsonUtils.marshalToString(position).getBytes();
        }
    }
    
    /**
     * 解析 ZK 中的位点数据
     */
    private Position parseCursorData(byte[] data) {
        String json = new String(data, StandardCharsets.UTF_8);
        return JsonUtils.unmarshalFromString(json, LogPosition.class);
    }
}
```

**ZK 中的位点节点结构**：

```
/otter/canal/
├── cluster/
│   └── ... 
└── destinations/
    └── example/
        ├── cursor    ← 消费位点
        ├── running   ← 运行的 Server 信息
        └── cluster/
            └── ...
```

# 6. PeriodMixedMetaManager — 定时混合持久化

这是生产环境最常用的模式，在内存模式下定期将位点持久化到本地文件 + ZK：

```java
public class PeriodMixedMetaManager extends MemoryMetaManager {
    
    // 持久化间隔（默认 1000ms）
    private long period = 1000;
    
    // 同步服务（定时任务）
    private ScheduledExecutorService syncService;
    
    // 上次持久化的位点
    private Position lastFlushedPosition;
    
    @Override
    public void start() {
        super.start();
        
        // 启动定时持久化任务
        syncService.scheduleAtFixedRate(() -> {
            try {
                if (cursor != null && !cursor.equals(lastFlushedPosition)) {
                    flushPosition(cursor);
                    lastFlushedPosition = cursor;
                }
            } catch (Exception e) {
                logger.error("period flush position failed", e);
            }
        }, period, period, TimeUnit.MILLISECONDS);
    }
    
    private void flushPosition(Position position) {
        // 1. 写入本地文件
        fileMetaManager.updateCursor(position);
        
        // 2. 写入 ZK（HA 模式时）
        if (zkMetaManager != null) {
            zkMetaManager.updateCursor(position);
        }
    }
}
```

# 7. 位点管理策略

## 7.1 位点查找优先级

Canal 在启动时，通过以下优先级查找起始位点：

```
1. MetaManager 中持久化的位点
   ↓ 不存在
2. Instance 配置中指定的 journalName + position
   ↓ 未指定  
3. Instance 配置中指定的 timestamp
   ↓ 未指定
4. MySQL SHOW MASTER STATUS（当前最新位点）
```

## 7.2 位点更新时机

```
EventParser 拉取 Binlog Event
        │
        ▼
解析并转换为 Entry
        │
        ▼
Sink 过滤
        │
        ▼
Store 存储（此时位点仅更新在内存中）
        │
        ▼
Client 消费并 ACK
        │
        ▼
MetaManager.updateCursor()  ← 这里才持久化位点
```

> **关键**：Canal 在 Client ACK 时才更新持久化位点，而不是在 Parser 拉取时。这保证了即使 Canal 重启，未确认的消息可以被重新消费。

## 7.3 位点一致性问题

```
时间线：
t1: Parser 拉取事件 E1 (pos=100) → Store
t2: Parser 拉取事件 E2 (pos=200) → Store
t3: Client 消费 E1、E2
t4: Client ACK (pos=200) → MetaManager 更新位点
t5: Canal 宕机

重启后:
MetaManager 中的 cursor = pos=200
从 pos=200 开始拉取 → 不会丢数据
```

```
时间线（异常情况）：
t1: Parser 拉取事件 E1 (pos=100) → Store
t2: Client 消费 E1 → MetaManager 更新位点 (pos=100)
t3: Parser 拉取事件 E2 (pos=200) → Store  
t4: Canal 宕机（E2 未被消费）

重启后:
MetaManager 中的 cursor = pos=100
从 pos=100 开始拉取 → E2 可能被重复收到
```

这说明 Canal 的位点机制是 **At-Least-Once** 语义，而不是 Exactly-Once。消费者需要自己实现幂等处理。

# 8. GTID 模式下的位点管理

## 8.1 GTID 位点格式

当启用 GTID 模式时，位点信息增加 GTID 字段：

```java
public class EntryPosition extends Position {
    private String journalName;   // 传统位点：binlog 文件名
    private Long position;        // 传统位点：偏移量
    private String gtid;          // GTID 集合：uuid:1-100:102:105-200
    // ...
}
```

**位点 JSON 示例（GTID 模式）**：

```json
{
  "identity": {
    "slaveId": 1234,
    "sourceAddress": "192.168.1.100:3306"
  },
  "postion": {
    "journalName": "mysql-bin.000003",
    "position": 45678901,
    "timestamp": 1672545600000,
    "gtid": "3e11fa47-71ca-11e1-9e33-c80aa9429562:1-150"
  }
}
```

## 8.2 GTID 模式下的位点查找

```java
protected EntryPosition findStartPosition() {
    
    EntryPosition startPosition = new EntryPosition();
    
    // 1. 先尝试从 MetaManager 恢复
    Position cursor = metaManager.getCursor();
    if (cursor != null && cursor.getIdentity() != null) {
        EntryPosition savedPosition = ((LogPosition) cursor).getPostion();
        
        if (isGtidMode()) {
            // GTID 模式：验证源地址是否一致
            if (isSameMaster(cursor.getIdentity())) {
                // 同一台 Master，直接恢复
                startPosition.setGtid(savedPosition.getGtid());
                return startPosition;
            } else {
                // 不同 Master（可能发生了 HA 切换）
                // 使用 GTID 集合重新定位
                startPosition.setGtid(savedPosition.getGtid());
                // 让 MySQL 根据 GTID 自动找到正确的 binlog 位置
                return startPosition;
            }
        } else {
            // 传统模式：直接恢复
            startPosition.setJournalName(savedPosition.getJournalName());
            startPosition.setPosition(savedPosition.getPosition());
            return startPosition;
        }
    }
    
    // 2. 降级处理...
    return findEndPosition();  // 从最新位置开始
}
```

## 8.3 GTID 模式的优势

**传统位点的 HA 切换困境**：

```
Master A (mysql-bin.000001)  ←→  Master B (mysql-bin.000005)
         ↓                                    ↓
    pos=12345                              pos=?

当 Master A 故障，切换到 Master B 时：
Master B 的 binlog 文件和位置完全不同
无法直接从传统位点恢复
```

**GTID 模式解决**：

```
Master A: uuid_a:1-100
Master B: uuid_b:1-50

Canal 的位点 = uuid_a:1-100
切换到 Master B 时，发送 GTID = uuid_a:1-100

MySQL 会自动查找包含这些 GTID 的所有 binlog
并定位到正确的起始位置继续推送
```

# 9. 位点数据清理

## 9.1 Binlog 过期导致位点失效

当 MySQL 端删除了 Canal 位点对应的 Binlog 文件时：

```java
protected void handleMissingBinlogFile(String missingFile) {
    
    logger.warn("binlog file {} has been purged, try to find next available", 
                missingFile);
    
    // 策略 1：自动跳过到下一个可用文件
    try {
        List<File> binlogFiles = getAvailableBinlogFiles();
        if (!binlogFiles.isEmpty()) {
            File nextFile = binlogFiles.get(0);  // 最旧的可用的文件
            resetPosition(nextFile.getName(), 4L);  // 从头开始
            logger.info("reset position to {}:4", nextFile.getName());
            return;
        }
    } catch (Exception e) {
        // 降级处理
    }
    
    // 策略 2：如果所有策略都失败，从最新位置开始
    EntryPosition latest = findEndPosition();
    resetPosition(latest.getJournalName(), latest.getPosition());
    
    // 策略 3：发送告警
    alarmHandler.sendAlarm("Binlog file purged, position reset to latest");
}
```

## 9.2 位点安全性保障

```properties
# 设置合理的 binlog 过期时间，防止数据丢失
# MySQL 端：
binlog_expire_logs_seconds = 604800  # 保留 7 天

# 确保 Canal 在 7 天内至少 ACK 一次
# 如果 Canal 可能长时间停机，需要做 binlog 备份或增加过期时间
```

# 10. MetaManager 与 HA 协同

在 ZooKeeper HA 模式下，MetaManager 与 HA 机制协同工作：

```java
// CanalHAController 中使用 MetaManager
public class CanalHAController {
    
    private ZooKeeperMetaManager metaManager;
    
    /**
     * HA 切换时，从 ZK 获取最新的位点
     */
    public void onMasterChange(String newMaster) {
        // 1. 停止当前 Parser
        eventParser.stop();
        
        // 2. 从 ZK 获取最新位点
        Position cursor = metaManager.getCursor();
        
        // 3. 更新 Parser 的连接信息
        eventParser.setMasterAddress(newMaster);
        
        // 4. 从断点位置重新启动 Parser
        eventParser.setStartPosition(cursor);
        eventParser.start();
        
        logger.info("HA switch complete, restart from position: {}", cursor);
    }
}
```

# 11. 位点手动管理

## 11.1 通过 Admin 管理位点

在 Canal Admin 界面中，可以对指定位点进行手动操作：

- **查看当前位点**：显示 Instance 当前的消费位点
- **重置位点**：手动设置位点为指定的 binlog 文件+偏移量
- **跳跃位点**：跳过某些有问题的 binlog 事件
- **回溯位点**：回退到历史位点重新消费

## 11.2 通过 Canal Admin API 管理

```bash
# 查看实例的位点信息
curl http://localhost:8089/api/v1/canal/instance/log/example

# 返回：
# {
#   "code": 20000,
#   "data": {
#     "cursor": {
#       "identity": {...},
#       "postion": {
#         "journalName": "mysql-bin.000003",
#         "position": 45678901,
#         "gtid": "uuid:1-100"
#       }
#     }
#   }
# }

# 手动修改位点
curl -X PUT http://localhost:8089/api/v1/canal/instance/log/example \
  -H "Content-Type: application/json" \
  -d '{
    "journalName": "mysql-bin.000003",
    "position": 100000
  }'
```

## 11.3 通过 ZK 直接操作（HA 模式）

```bash
# 连接 ZK 客户端
zkCli.sh -server 127.0.0.1:2181

# 查看位点
get /otter/canal/destinations/example/cursor

# 手动修改位点（格式: {"journalName":"mysql-bin.000003","position":45678,"timestamp":0}）
set /otter/canal/destinations/example/cursor '{"postion":{"journalName":"mysql-bin.000003","position":100000,"timestamp":0}}'
```

# 12. 最佳实践

## 12.1 位点管理建议

1. **生产环境启用 GTID**：避免主从切换时位点定位问题
2. **合理设置位点持久化间隔**：默认 1000ms 即可，不要过于频繁
3. **定期备份位点信息**：在 ZK 或文件之外，保留一份位点备份
4. **监控位点延迟**：通过 Canal Admin 或 Prometheus 监控消费延迟

## 12.2 位点延迟监控

```java
// 计算位点延迟
public long calculateDelay(EntryPosition currentPosition, MysqlConnection conn) {
    try {
        // 获取 MySQL 当前最新位点
        ResultSet rs = conn.query("SHOW MASTER STATUS");
        if (rs.next()) {
            String latestFile = rs.getString("File");
            long latestPosition = rs.getLong("Position");
            
            if (latestFile.equals(currentPosition.getJournalName())) {
                return latestPosition - currentPosition.getPosition();
            }
            
            // 不同文件，按文件大小估算
            return estimateDelay(currentPosition, latestFile, latestPosition);
        }
    } catch (Exception e) {
        logger.error("calculate delay error", e);
    }
    return -1;
}
```

## 12.3 数据一致性保障

由于 Canal 是 At-Least-Once 语义，消费端需要实现：

```java
public class IdempotentConsumer {
    
    // Redis 记录已处理的位点
    private RedisClient redis;
    
    public void consume(Message message) {
        long batchId = message.getId();
        
        // 检查是否已经处理过
        if (redis.exists("canal:consumed:" + batchId)) {
            connector.ack(batchId);  // 已处理，直接 ACK
            return;
        }
        
        try {
            // 业务处理
            processEntries(message.getEntries());
            
            // 标记已处理
            redis.setex("canal:consumed:" + batchId, 3600, "1");
            
            // ACK
            connector.ack(batchId);
            
        } catch (Exception e) {
            connector.rollback(batchId);
        }
    }
}
```
