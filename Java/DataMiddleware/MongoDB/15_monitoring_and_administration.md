
# 1. 监控命令概览

## 1.1 serverStatus

```javascript
// 获取服务器完整状态信息
db.serverStatus()

// 关键指标分组：
db.serverStatus().connections      // 连接信息
db.serverStatus().opcounters       // 操作计数
db.serverStatus().opLatencies      // 操作延迟
db.serverStatus().metrics          // 详细指标
db.serverStatus().wiredTiger       // WiredTiger引擎
db.serverStatus().locks            // 锁信息
db.serverStatus().network          // 网络统计
db.serverStatus().globalLock       // 全局锁
db.serverStatus().mem              // 内存使用
db.serverStatus().extra_info       // 额外信息
db.serverStatus().transactions     // 事务统计
```

## 1.2 currentOp

```javascript
// 查看当前所有操作
db.currentOp()

// 查看运行超过5秒的操作
db.currentOp({ "secs_running": { $gt: 5 } })

// 查看特定数据库的操作
db.currentOp({ "ns": /^mydb\./ })

// 查看等待锁的操作
db.currentOp({ "waitingForLock": true })

// 查看特定类型的操作
db.currentOp({ "op": "query" })

// 查看活跃（非空闲）操作
db.currentOp({ "active": true })

// 终止操作
db.killOp(opid)

// 终止所有特定类型的操作（谨慎使用）
db.currentOp({ "ns": /^mydb\./, "active": true }).inprog.forEach(op => {
    db.killOp(op.opid)
})
```

## 1.3 stats

```javascript
// 数据库统计
db.stats()

// 集合统计（重要）
db.collection.stats()
// 包含字段：
// size: 集合数据总大小
// storageSize: 磁盘分配大小
// count: 文档数
// avgObjSize: 平均文档大小
// nindexes: 索引数量
// totalIndexSize: 索引总大小
// totalSize: 数据+索引总大小
// capped: 是否固定集合
// wiredTiger: WiredTiger特定信息

// 集合延迟统计
db.collection.latencyStats()

// 存储统计（按数据库）
db.runCommand({ storageStats: {}, scale: 1024 * 1024 })

// 查看top（类似mongotop）
db.adminCommand("top")
```

# 2. 性能诊断命令

## 2.1 explain

```javascript
// 查看查询执行计划（已在索引章节详述）
db.collection.find({ field: "value" }).explain("executionStats")

// 聚合管道explain
db.collection.explain("executionStats").aggregate([...])
```

## 2.2 Profiler分析

```javascript
// 查看slow query日志
db.system.profile.find({
    millis: { $gt: 100 }
}).sort({ ts: -1 }).limit(10).pretty()

// 按namespace分组统计慢查询
db.system.profile.aggregate([
    { $match: { millis: { $gt: 100 } } },
    { $group: {
        _id: "$ns",
        count: { $sum: 1 },
        avgTime: { $avg: "$millis" },
        maxTime: { $max: "$millis" }
    }},
    { $sort: { count: -1 } }
])

// 找出最耗时的查询模式
db.system.profile.aggregate([
    { $match: { op: "query" } },
    { $group: {
        _id: "$planSummary",
        count: { $sum: 1 },
        totalTime: { $sum: "$millis" }
    }},
    { $sort: { totalTime: -1 } },
    { $limit: 10 }
])
```

## 2.3 锁分析

```javascript
// 全局锁
db.serverStatus().globalLock
// globalLock.currentQueue.total: 等待锁的操作数
// globalLock.totalTime: 全局锁持有时长

// 锁统计
db.serverStatus().locks
// 按type: Global, Database, Collection, Mutex
// 按mode: r(读), w(写), R(意向读), W(意向写)
```

# 3. 监控工具

## 3.1 mongostat（命令行）

```bash
# 实时监控（每秒刷新）
mongostat --host localhost --port 27017

# 自定义间隔
mongostat 5       # 每5秒输出

# 输出更多列
mongostat -O="host,version,network.numRequests"

# 连接副本集
mongostat --host rs0/mongo1:27017,mongo2:27017

# 输出示例解读：
# insert/query/update/delete  — 每秒操作数
# getmore                     — 游标获取批次
# command                     — 命令操作数
# dirty                       — WiredTiger脏缓存比例(%)
# used                        — WiredTiger缓存使用比例(%)
# vsize                       — 虚拟内存(MB)
# res                         — 物理内存(MB)
# qr|qw                       — 读/写操作队列
# ar|aw                       — 活跃读/写操作
# netIn/netOut                — 网络I/O(KB/s)
# conn                        — 当前连接数
# set                         — 副本集名称
# repl                        — 节点状态(PRI/SEC/ARB)
# time                        — 当前时间
```

## 3.2 mongotop（命令行）

```bash
# 集合级别读写时间分析
mongotop 5     # 每5秒输出

# 输出示例：
# ns        total    read    write
# mydb.users  450ms  300ms   150ms
# 含义：users集合每秒花450ms在读取和写入上
```

## 3.3 MongoDB Compass（GUI）

Compass是MongoDB官方的GUI管理工具，提供：

- 可视化查询构建器
- 聚合管道构建器（所见即所得）
- 执行计划可视化
- 索引管理
- Schema分析
- 实时性能面板

## 3.4 mtools工具集

```bash
# mtools安装
pip install mtools

# 日志分析
mlogfilter mongod.log --from "2024-01-15 08:00" --to "2024-01-15 12:00" --slow 100

# 日志可视化
mplotqueries mongod.log

# 日志重放（性能分析）
mloginfo mongod.log --queries
```

# 4. 日常管理操作

## 4.1 连接管理

```javascript
// 查看当前连接数
db.serverStatus().connections
// current: 当前连接数
// available: 可用连接数
// totalCreated: 历史创建总数

// 查看每个连接的详细信息
db.currentOp(true)

// 断开连接（需要有相应权限）
db.killOp(opid)

// 强制断开所有客户端连接（维护用）
db.adminCommand({
    "dropConnections": 1,
    "hostAndPort": "mongod:27017"
})
```

## 4.2 数据库管理

```javascript
// 查看所有数据库和大小
show dbs
db.adminCommand("listDatabases")

// 查看数据库详情
db.stats(1024 * 1024)  // 以MB为单位

// 修复数据库（谨慎使用，会阻塞）
db.repairDatabase()

// 查看数据库存储信息
db.runCommand({ dbStats: 1, scale: 1024 * 1024 })
```

## 4.3 集合管理

```javascript
// 压缩集合（回收碎片空间）
db.runCommand({ compact: "collectionName" })
// compact会阻塞操作，建议在维护窗口执行

// 压缩的替代方案：重新同步数据
// 1. 创建新集合并迁移数据
// 2. 删除旧集合并重命名新集合

// 收缩数据库空间
db.runCommand({ reIndex: "collectionName" })  // 重建索引、释放空间

// 查看集合碎片程度
db.collection.stats().wiredTiger["block-manager"]
// 关注 file bytes available for reuse
```

## 4.4 缓存管理

```javascript
// 清理WiredTiger缓存（仅用于测试，生产环境慎用）
db.runCommand({ closeAllConnections: 1 })

// 查看计划缓存
db.collection.getPlanCache().listQueryShapes()

// 查看特定查询的计划缓存
db.collection.getPlanCache().getPlansByQuery({
    query: { field: "value" },
    sort: { createTime: -1 },
    projection: {}
})

// 清除计划缓存
db.collection.getPlanCache().clear()

// 清除所有集合的计划缓存
db.adminCommand({ planCacheClear: "collectionName" })
```

## 4.5 参数动态修改

```javascript
// 查看可配置参数
db.adminCommand({ getParameter: "*" })

// 修改参数（无需重启）
db.adminCommand({ setParameter: 1, logLevel: 2 })
db.adminCommand({ setParameter: 1, 
    failedAuthenticationDelayMS: 3000 })

// 常用可动态修改的参数：
// logLevel (0-5)
// slowOpThresholdMs
// notablescan (禁止COLLSCAN)
// transactionLifetimeLimitSeconds
// maxTransactionLockRequestTimeoutMillis
// diagnosticDataCollectionEnabled
```

# 5. 副本集管理

## 5.1 副本集状态

```javascript
// 查看副本集状态
rs.status()

// 关键解读：
// members[].stateStr: PRIMARY / SECONDARY / ARBITER
// members[].health: 1(健康) / 0(宕机)
// members[].optimeDate: 最后应用oplog时间
// members[].pingMs: 心跳延迟
// members[].uptime: 运行时长

// 副本集配置
rs.conf()

// 查看oplog信息
rs.printReplicationInfo()

// 从节点同步状态
rs.printSecondaryReplicationInfo()

// 重新配置
var cfg = rs.conf()
cfg.settings.electionTimeoutMillis = 5000
rs.reconfig(cfg)
```

## 5.2 副本集维护操作

```javascript
// 手动主节点降级
rs.stepDown(60)   // 60秒内不参与选举

// 强制重新配置（大多数节点宕机时）
var cfg = rs.conf()
cfg.members = [cfg.members[0], cfg.members[1]]
// 移除不可达成员后
rs.reconfig(cfg, { force: true })

// 添加新节点（会触发initial sync）
rs.add("newNode:27017")

// 添加仲裁节点
rs.addArb("arbiter:27017")

// 移除节点
rs.remove("host:27017")

// 冻结节点（维护期间）
rs.freeze(300)  // 300秒内不参与选举
rs.freeze(0)    // 解冻
```

## 5.3 Oplog监控

```javascript
// 检查oplog大小是否足够
var local = db.getSiblingDB("local")
var oplogStats = local.oplog.rs.stats()
var oplogSizeMB = oplogStats.maxSize / (1024 * 1024)

var first = local.oplog.rs.find().sort({ $natural: 1 }).limit(1).next()
var last = local.oplog.rs.find().sort({ $natural: -1 }).limit(1).next()
var timeDiff = (last.ts.getTime() - first.ts.getTime()) / 1000 / 60 / 60  // 小时

print("Oplog大小: " + oplogSizeMB + " MB")
print("Oplog窗口: " + timeDiff + " 小时")
```

# 6. 分片集群管理

```javascript
// 分片集群状态
sh.status()
// 或
db.printShardingStatus()

// 查看Balancer
sh.getBalancerState()    // 是否启用
sh.isBalancerRunning()   // 是否正在运行
sh.setBalancerState(true/false)

// 查看chunk分布
db.collection.getShardDistribution()

// 查看分片信息
use config
db.shards.find().pretty()
db.chunks.find().pretty()
db.databases.find().pretty()

// 手动迁移chunk
sh.moveChunk("mydb.collection", { shardKey: "value" }, "targetShard")

// 合并空chunk
sh.mergeChunks("mydb.collection", 
    { shardKey: MinKey }, { shardKey: MaxKey })

// 刷新mongos配置缓存
db.adminCommand("flushRouterConfig")
```

# 7. 告警设置建议

| 指标 | 警告阈值 | 严重阈值 | 说明 |
|------|---------|---------|------|
| 连接使用率 | > 70% | > 90% | connections.current / connections.available |
| 复制延迟 | > 10s | > 60s | 从节点落后时间 |
| Oplog窗口 | < 24h | < 1h | 可能引发全量同步 |
| WiredTiger缓存 | > 90% | > 95% | 缓存不足，可能频繁换页 |
| 磁盘使用率 | > 75% | > 90% | 需要扩容 |
| 慢查询比例 | > 5% | > 20% | profiler采样比例 |
| 连接错误 | > 0 | > 10/min | 网络或认证问题 |
| Oplog增长 | > 正常2倍 | > 正常5倍 | 可能有异常写入 |
| 事务超时 | > 0 | > 10/min | 事务过慢或死锁 |

# 8. 日常巡检清单

```javascript
// === 每日检查 ===

// 1. 连接数
db.serverStatus().connections

// 2. 操作延迟
db.serverStatus().opLatencies.reads
db.serverStatus().opLatencies.writes

// 3. 存储使用
db.stats()
db.collection.stats()

// 4. 副本集健康
rs.status().members.forEach(m => print(m.name, m.health, m.stateStr))

// 5. Oplog窗口
rs.printReplicationInfo()

// 6. 错误日志
// 检查mongod.log文件中的错误

// === 每周检查 ===

// 1. 索引使用情况（删除未使用索引）
db.collection.aggregate([{ $indexStats: {} }])

// 2. 慢查询趋势
db.system.profile.aggregate([
    { $match: { ts: { $gte: weekAgo } } },
    { $group: { _id: "$ns", count: { $sum: 1 }, avgTime: { $avg: "$millis" } } }
])

// 3. 碎片程度（需compact的集合）
// 关注 storageSize vs size 的差异

// 4. 用户和权限审查
db.getUsers()

// === 每月检查 ===

// 1. 备份恢复测试
// 2. 灾难恢复演练
// 3. 版本更新评估
// 4. 容量规划
```
