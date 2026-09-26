
# 1. 副本集概述

副本集（Replica Set）是一组维护相同数据集的MongoDB进程，提供**数据冗余**和**高可用性**。副本集是MongoDB生产部署的最基本形式。

副本集的核心功能：
- **数据冗余**：多份数据副本，防止单点故障
- **自动故障转移**：主节点不可用时自动选举新主节点
- **读写分离**：读操作可以路由到从节点，分散负载
- **灾难恢复**：异地部署从节点，应对数据中心故障
- **备份不影响性能**：在从节点进行备份不影响主节点性能

```
┌──────────────────────────────────────────┐
│              副本集架构                    │
│                                          │
│     ┌────────┐     Heartbeat             │
│     │ Primary │◄──────────────────┐       │
│     │ (读写)  │                   │       │
│     └───┬────┘                   │       │
│         │ Oplog同步               │       │
│    ┌────▼───┐              ┌─────▼──┐    │
│    │Secondary│              │Secondary│    │
│    │ (只读)  │              │ (只读)  │    │
│    └────────┘              └────────┘    │
│         ▲                      ▲         │
│         └─────── Arbiter ──────┘         │
│               (仅投票)                    │
└──────────────────────────────────────────┘
```

# 2. 副本集成员角色

| 角色 | 说明 |
|------|------|
| Primary | 主节点，唯一接收写操作的节点，通过oplog向从节点同步 |
| Secondary | 从节点，复制主节点的oplog并应用，默认可读 |
| Arbiter | 仲裁节点，仅参与选举投票，不存储数据，不提供服务 |
| Hidden | 隐藏节点，不参与选举，对客户端不可见，用于备份/报表 |
| Delayed | 延迟节点，故意延迟同步（用于防止人为误删） |
| Priority 0 | 优先级为0的节点，永远不能成为Primary |

# 3. 副本集架构模式

## 3.1 标准三节点（推荐）

```
P(主) + S(从) + S(从)
具备完整高可用能力，自动故障转移，奇数投票
```

## 3.2 两节点+仲裁节点

```
P(主) + S(从) + Arbiter(仲裁)
最少硬件实现高可用，但实际数据冗余只有2份
仲裁节点不存储数据，资源消耗极低
```

## 3.3 多数据中心部署

```
数据中心A: P + S(priority=2)
数据中心B: S(priority=0.5) + S(hidden)
跨机房高可用，P0节点确保不会自动切换到特定数据中心
```

# 4. Oplog（操作日志）

## 4.1 Oplog机制

Oplog（Operations Log）是副本集数据同步的核心。它是一个特殊的**固定集合（Capped Collection）**，记录主节点上所有修改数据的操作。

```javascript
// oplog存储在local数据库中
use local
db.oplog.rs.find().limit(5).pretty()

// oplog条目结构
{
    "ts": Timestamp(1705651200, 1),     // 操作时间戳
    "t": NumberLong(1),                 // term（选举任期）
    "h": NumberLong("..."),             // 操作哈希（唯一标识）
    "v": 2,                             // oplog版本
    "op": "i",                          // 操作类型
    "ns": "mydb.users",                 // 命名空间（库.集合）
    "o": { _id: ..., name: "张三" }     // 操作内容
}

// 操作类型(op)：
// "i" - insert
// "u" - update
// "d" - delete
// "c" - 命令（如创建集合、创建索引）
// "n" - no-op（空操作，用于心跳）
```

## 4.2 Oplog大小配置

Oplog大小的选择直接影响从节点能落后主节点多久：

```bash
# 启动时指定（MB）
mongod --oplogSize 2048 --replSet rs0

# 配置文件中
replication:
  oplogSizeMB: 2048
```

Oplog大小计算建议：
- 默认：磁盘空间的5%（最小990MB，最大50GB）
- 建议：至少能容纳**24-48小时的写操作量**
- 公式：`oplogSizeMB ≈ 每小时写操作产生的oplog字节数 × 期望恢复时间(小时) / (1024 × 1024)`

```javascript
// 查看oplog信息
rs.printReplicationInfo()

// 输出示例：
// configured oplog size: 2048MB
// log length start to end: 34567 secs (9.6 hrs)
// oplog first event time: Mon Jan 15 2024 00:00:00
// oplog last event time:  Mon Jan 15 2024 09:36:07
// now:                     Mon Jan 15 2024 09:36:10
```

## 4.3 Oplog窗口（Oplog Window）

Oplog窗口 = oplog最早记录到当前时间的间隔，表示从节点能离线多长时间仍能追赶上主节点。如果从节点离线时间超过oplog窗口，需要**全量同步（Initial Sync）**。

# 5. 数据同步机制

## 5.1 初始同步（Initial Sync）

当新节点加入副本集时，执行初始同步：

```
初始同步流程（6.0+）：
1. 克隆所有数据库（不包括local库）
2. 应用同步期间产生的oplog变更
3. 构建索引（可并行）
4. 完成并切换到Secondary状态

初始同步流程（4.4+）：
1. 创建集合和索引
2. 复制文档
3. 应用oplog
```

```javascript
// 查看初始同步状态
rs.status().members.find(m => m.stateStr === "STARTUP2")
```

## 5.2 增量同步

初始同步完成后，从节点持续从主节点（或其它从节点）拉取oplog并应用：

```
从节点 → 拉取oplog → 应用oplog操作 → 保持与主节点一致
               ↓(通过链式复制)
         从节点A → 从节点B → 从节点C
```

```javascript
// 设置从节点同步源
rs.syncFrom("hostname:27017")

// 查看从节点延迟
rs.printSecondaryReplicationInfo()

// 输出：
// source: mongo2:27017
//     syncedTo: Mon Jan 15 2024 10:00:00
//     0 secs (0 hrs) behind the primary
```

# 6. 选举机制

## 6.1 Raft协议

MongoDB使用Raft共识协议的变体实现选举：
- 节点间通过心跳（Heartbeat）维持通信（每2秒）
- 如果从节点10秒内收不到主节点心跳，发起选举
- 获得大多数票（N/2+1）的节点成为新主节点

## 6.2 选举触发条件

| 触发条件 | 说明 |
|---------|------|
| 主节点宕机 | 最常见情况 |
| 主节点网络隔离 | 无法与大多数节点通信 |
| 手动触发 | `rs.stepDown()`命令 |
| 主节点降级 | 节点priority变化 |

## 6.3 Priority（优先级）

```javascript
// 每个节点可以设置优先级（0~1000，默认1）
var cfg = rs.conf()
cfg.members[0].priority = 2     // 更高优先级优先选为主节点
cfg.members[1].priority = 1
cfg.members[2].priority = 0     // 永远不成为主节点
rs.reconfig(cfg)
```

## 6.4 electionTimeoutMillis等参数

```javascript
rs.conf()

// settings相关参数：
// electionTimeoutMillis: 10000 (默认10秒)
// heartbeatTimeoutSecs: 10
// catchUpTimeoutMillis: -1 (无限制)
```

# 7. 读写关注与读写分离

## 7.1 读偏好（Read Preference）

```javascript
// primary: 只从主节点读（默认）
db.collection.find().readPref("primary")

// primaryPreferred: 优先主节点，主不可用时从从节点读
db.collection.find().readPref("primaryPreferred")

// secondary: 只从从节点读
db.collection.find().readPref("secondary")

// secondaryPreferred: 优先从节点，从不可用时从主节点读
db.collection.find().readPref("secondaryPreferred")

// nearest: 读取网络延迟最低的节点
db.collection.find().readPref("nearest")

// Java连接字符串方式
// mongodb://host1,host2,host3/?readPreference=secondaryPreferred&maxStalenessSeconds=120
```

## 7.2 写关注（Write Concern）

```javascript
// w: 需要确认的节点数
{ writeConcern: { w: 1 } }            // 只等主节点（默认）
{ writeConcern: { w: "majority" } }   // 等大多数节点
{ writeConcern: { w: 3 } }            // 等3个节点

// j: 是否等待写入journal
{ writeConcern: { w: "majority", j: true } }

// wtimeout: 超时时间(ms)
{ writeConcern: { w: "majority", wtimeout: 5000 } }
```

## 7.3 读关注（Read Concern）

```javascript
// "local": 读节点本地数据（默认）
db.collection.find().readConcern("local")

// "available": 读节点上可用的数据（分片集群，比local更宽松）
db.collection.find().readConcern("available")

// "majority": 读已被大多数节点确认的数据（防止脏读/回滚）
db.collection.find().readConcern("majority")

// "linearizable": 线性一致性读（单文档，最严格）
db.collection.findOne({ _id: x }).readConcern("linearizable")

// "snapshot": 快照读（事务中使用）
// 需要事务上下文
```

# 8. 副本集管理操作

## 8.1 副本集初始化与配置

```javascript
// 初始化副本集
rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "mongo1:27017", priority: 2 },
        { _id: 1, host: "mongo2:27017", priority: 1 },
        { _id: 2, host: "mongo3:27017", priority: 1 }
    ]
})

// 添加节点
rs.add("mongo4:27017")
rs.add({ host: "mongo5:27017", priority: 0, hidden: true })

// 移除节点
rs.remove("mongo4:27017")

// 查看配置
rs.conf()

// 重新配置
var cfg = rs.conf()
cfg.members[1].priority = 0.5
cfg.members[2].hidden = true
rs.reconfig(cfg)

// 强制重新配置（仅在大多数节点不可用时使用）
rs.reconfig(cfg, { force: true })
```

## 8.2 监控副本集状态

```javascript
// 查看副本集状态
rs.status()

// 关键字段：
// stateStr: PRIMARY / SECONDARY / ARBITER
// health: 1 (健康) / 0 (不健康)
// optimeDate: 最后应用oplog的时间
// pingMs: 心跳延迟

// 查看复制信息
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()

// 查看oplog窗口
db.getReplicationInfo()

// 监控oplog大小
db.oplog.rs.stats()
```

## 8.3 手动故障转移

```javascript
// 主节点降级（60秒内不能重新选举）
rs.stepDown()

// 降级并指定时长
rs.stepDown(120, 30)  // 120秒内不参选，等待30秒给从节点追数据

// 冻结节点（不参与选举）
rs.freeze(60)   // 60秒内不参与选举

// 解冻
rs.freeze(0)
```

## 8.4 从节点延迟监控

```javascript
// 在从节点上查询复制延迟
db.printReplicationInfo()

// 或者在主节点上
rs.printSecondaryReplicationInfo()
```

# 9. 副本集故障场景与恢复

## 9.1 主节点故障

```
场景：主节点宕机
流程：
1. 从节点检测到心跳超时(10秒)
2. 符合条件的从节点发起选举
3. 获得大多数票的节点成为新主节点
4. 其他从节点切换到新主节点同步
5. 旧主节点恢复后以Secondary身份加入
```

## 9.2 从节点故障

```
场景：从节点宕机
影响：读负载可能增加，但不影响写入
恢复：
1. 从节点重启
2. 自动追赶oplog
3. 如果落后太多（超过oplog窗口），触发Initial Sync
```

## 9.3 网络分区

```
场景：网络隔离导致副本集分裂
结果：
- 包含大多数节点的分区 → 选举新主节点，继续服务
- 少数节点分区 → 无法选举，旧主节点降级为Secondary
- 网络恢复后 → 少数节点分区追赶oplog同步
```

## 9.4 Rollback（回滚）

当旧主节点恢复后，发现有自己提交了但新主节点没有的操作，需要回滚：

```javascript
// 回滚文件存储位置
// <dbPath>/rollback/
// 回滚的文档保存在rollback目录中的BSON文件

// 可配置回滚数据保留时间
db.adminCommand({
    setParameter: 1,
    rollbackTimeLimitSecs: 86400  // 1天
})
```

# 10. 副本集最佳实践

1. **使用奇数个投票节点**：避免脑裂
2. **保证大多数节点存活**：确保可以选举
3. **使用writeConcern: "majority"**：防止数据回滚
4. **合理配置oplog大小**：至少24小时的更改量
5. **从节点不可用时添加节点**：使用initialSync重同步
6. **监控复制延迟**：确保从节点及时跟上
7. **在不同物理位置部署**：增强容灾能力
8. **使用密钥文件保护通信**：security.keyFile

# 11. Spring Boot连接副本集

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://mongo1:27017,mongo2:27017,mongo3:27017/mydb?replicaSet=rs0&readPreference=secondaryPreferred&w=majority
```

```java
// MongoClient配置方式
@Configuration
public class MongoConfig {
    
    @Bean
    public MongoClient mongoClient() {
        MongoClientSettings settings = MongoClientSettings.builder()
            .applyToClusterSettings(builder ->
                builder.hosts(Arrays.asList(
                    new ServerAddress("mongo1", 27017),
                    new ServerAddress("mongo2", 27017),
                    new ServerAddress("mongo3", 27017)
                ))
                .requiredReplicaSetName("rs0")
            )
            .readPreference(ReadPreference.secondaryPreferred())
            .writeConcern(WriteConcern.MAJORITY)
            .build();
        return MongoClients.create(settings);
    }
}
```
