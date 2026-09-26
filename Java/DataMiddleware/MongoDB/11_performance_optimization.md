
# 1. 性能优化方法论

MongoDB性能优化遵循"测量 → 分析 → 优化 → 验证"的循环。在优化之前，首先要明确瓶颈在哪个层面。

## 1.1 性能瓶颈识别框架

```
┌──────────────────────────────────┐
│          应用层                    │
│  连接池、序列化、查询设计、N+1     │
├──────────────────────────────────┤
│          网络层                    │
│  延迟、带宽、连接数               │
├──────────────────────────────────┤
│         MongoDB服务层              │
│  查询优化器、锁、游标、聚合内存    │
├──────────────────────────────────┤
│         存储引擎层                 │
│  WiredTiger缓存、磁盘IO、压缩     │
├──────────────────────────────────┤
│         硬件/OS层                  │
│  CPU、内存、磁盘、文件系统、NUMA   │
└──────────────────────────────────┘
```

# 2. 慢查询分析

## 2.1 开启慢查询日志

```yaml
# mongod.conf
operationProfiling:
  mode: slowOp                    # off | slowOp | all
  slowOpThresholdMs: 100          # 慢查询阈值（ms）
  slowOpSampleRate: 1.0           # 采样率（0.0~1.0，1.0=全部）

# 动态修改（无需重启）
db.setProfilingLevel(1, { slowms: 100 })
```

```javascript
// 查看慢查询
db.system.profile.find()
    .sort({ ts: -1 })
    .limit(10)
    .pretty()

// 分析特定时间范围的慢查询
db.system.profile.find({
    ts: {
        $gte: ISODate("2024-01-15T08:00:00Z"),
        $lt: ISODate("2024-01-15T09:00:00Z")
    },
    millis: { $gt: 100 }
}).sort({ millis: -1 })

// 查找没有使用索引的慢查询（COLLSCAN）
db.system.profile.find({
    "planSummary": { $regex: /COLLSCAN/ }
}).sort({ millis: -1 })

// 清空profile集合
db.setProfilingLevel(0)    // 先关闭
db.system.profile.drop()
db.setProfilingLevel(1)    // 重新开启
```

## 2.2 Profile输出解读

```javascript
{
    "op": "query",                        // 操作类型
    "ns": "mydb.users",                   // 命名空间
    "command": {                          // 实际查询
        "find": "users",
        "filter": { "name": "张三" },
        "sort": { "createTime": -1 }
    },
    "keysExamined": 0,                    // 扫描索引条目数
    "docsExamined": 100000,               // 扫描文档数
    "nreturned": 10,                      // 返回文档数
    "millis": 350,                        // 执行时间(ms)
    "planSummary": "COLLSCAN",            // 执行计划
    "locks": {                            // 锁信息
        "Global": { "acquireCount": { "r": NumberLong(1) } },
        "Database": { "acquireCount": { "r": NumberLong(1) } },
        "Collection": { "acquireCount": { "r": NumberLong(1) } }
    }
}
```

关键指标：
- `keysExamined / nreturned` 过高 → 索引效率低
- `docsExamined / nreturned` 过高 → 过滤条件不精准或缺少复合索引
- `keysExamined: 0` 且大量 `docsExamined` → COLLSCAN，缺少索引
- `millis` 高 → 需要优化

# 3. 查询性能优化

## 3.1 索引优化

```javascript
// 1. 识别缺失索引的查询
// 在慢查询中找planSummary: "COLLSCAN"

// 2. 创建合适的索引（遵循ESR规则）
db.collection.createIndex({ 
    equalityField: 1,    // 等值条件
    sortField: 1,        // 排序字段
    rangeField: 1        // 范围条件
})

// 3. 使用覆盖查询减少FETCH阶段
// 索引包含所有查询和返回的字段时，无需读取文档
db.collection.find(
    { status: "active", age: { $gt: 25 } },
    { name: 1, email: 1, _id: 0 }
)
// 配合索引: { status: 1, age: 1, name: 1, email: 1 }

// 4. 检查冗余索引
db.collection.aggregate([{ $indexStats: {} }])
// 如果{ a: 1 }和{ a: 1, b: 1 }同时存在且{ a: 1 }从未被使用 → 可能冗余
```

## 3.2 投影优化（减少传输）

```javascript
// 只返回需要的字段
db.users.find(
    { status: "active" },
    { name: 1, email: 1, _id: 0 }  // 只返回name和email
)

// 排除大字段
db.articles.find({}, { fullContent: 0, images: 0 })
```

## 3.3 排序优化

```javascript
// 避免内存排序（SORT stage）
// 创建支持排序的索引
db.collection.createIndex({ status: 1, createTime: -1 })

// 查询利用索引排序
db.collection.find({ status: "active" }).sort({ createTime: -1 })

// 监控是否发生内存排序失败
db.serverStatus().metrics.query.sortStageCount
```

## 3.4 分页优化

```javascript
// 避免大offset的skip分页
// 不推荐：
db.collection.find().sort({ _id: -1 }).skip(100000).limit(20)

// 推荐：游标分页（基于排序字段的范围查询）
var lastId = ObjectId("...")  // 上一页最后一个文档的_id
db.collection.find({ _id: { $lt: lastId } }).sort({ _id: -1 }).limit(20)

// 当排序字段非唯一时，添加_id作为ties-breaker
db.collection.find({
    $or: [
        { createTime: { $lt: lastTime } },
        { createTime: lastTime, _id: { $lt: lastId } }
    ]
}).sort({ createTime: -1, _id: -1 }).limit(20)
```

## 3.5 避免N+1查询

```javascript
// 不推荐：循环查询（N+1问题）
var orders = db.orders.find().toArray()
orders.forEach(function(order) {
    var user = db.users.findOne({ _id: order.userId })
    print(user.name)
})

// 推荐1：嵌入式文档（数据总是一起读取）
// 推荐2：使用$lookup一次性关联
db.orders.aggregate([
    {
        $lookup: {
            from: "users",
            localField: "userId",
            foreignField: "_id",
            as: "user"
        }
    }
])

// 推荐3：分批次IN查询（先收集所有ID再批量查）
var userIds = db.orders.distinct("userId")
var users = db.users.find({ _id: { $in: userIds } }).toArray()
var userMap = {}
users.forEach(u => { userMap[u._id] = u })
```

# 4. 写入性能优化

## 4.1 批量写入

```javascript
// 单条插入 → 慢（每次网络往返）
db.collection.insertOne(doc)

// 批量插入 → 快
db.collection.insertMany(docs, { ordered: false })

// bulkWrite → 最灵活
db.collection.bulkWrite([
    { insertOne: { document: doc1 } },
    { updateOne: { filter: {...}, update: {...} } },
    { deleteOne: { filter: {...} } }
], { ordered: false })
```

## 4.2 写关注（Write Concern）调整

```javascript
// 追求写入速度：w: 0（不等待确认）
db.collection.insertOne(doc, { writeConcern: { w: 0 } })

// 权衡安全和速度：w: 1（等待主节点，默认）
db.collection.insertOne(doc, { writeConcern: { w: 1 } })

// 最高安全性：w: "majority"
db.collection.insertOne(doc, { writeConcern: { w: "majority" } })

// 避免同时设置j: true（刷journal），除非绝对不能丢数据
```

## 4.3 减少索引数量

每个索引都会拖慢写入速度。对写入密集型场景，应精确控制索引数量。

关于索引对写入性能的影响：
- 每个B树索引：每次insert/update/delete都要维护
- 多键索引：数组字段的索引维护成本更高
- 文本索引：需要额外的词条划分和存储
- 估算：每增加一个索引，写入吞吐量降低约5%~15%

## 4.4 数据模型影响

```javascript
// 嵌入式文档 vs 引用式文档的写入影响

// 嵌入式：一次I/O更新所有数据（原子操作）
db.orders.updateOne(
    { _id: "ORD001" },
    { $push: { items: newItem } }
)

// 引用式：可能需要多次I/O
db.orders.updateOne({ _id: "ORD001" }, { $push: { items: newItem } })
db.products.updateOne({ _id: productId }, { $inc: { stock: -1 } })
```

# 5. WiredTiger存储引擎优化

## 5.1 缓存大小优化

WiredTiger缓存是影响MongoDB性能的最关键配置：

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8    # 根据服务器内存大小确定
```

```javascript
// 查看缓存使用情况
db.serverStatus().wiredTiger.cache

// 关键指标：
// "bytes currently in the cache" — 当前缓存占用
// "maximum bytes configured" — 配置的最大缓存
// "pages evicted by application threads" — 应用线程被迫等待换页的次数
// 如果evicted by application threads很高 → 缓存不足

// 查看缓存命中率
var cacheStats = db.serverStatus().wiredTiger.cache
var hitRatio = 1 - (cacheStats["pages read into cache"] / 
                     (cacheStats["pages requested from the cache"] + 1))
// 目标 > 95%
```

## 5.2 压缩算法选择

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy    # snappy | zlib | zstd | none
```

- **snappy**（默认）：压缩比适中，CPU消耗低，适合大多数场景
- **zlib**：压缩比高，CPU消耗高，适合存储密集型
- **zstd**：压缩比高，CPU消耗适中（6.0+），zlib的现代替代
- **none**：无压缩，适合纯内存或极致性能要求

## 5.3 检查点调优

```javascript
// 查看检查点统计
db.serverStatus().wiredTiger.checkpoint

// 调整检查点间隔（不推荐轻易修改）
// 默认60秒或2GB journal

// 增加间隔 → 减少IO峰值，但增加恢复时间
// 减少间隔 → 减少恢复时间，但增加IO
```

## 5.4 存储引擎统计监控

```javascript
// 查看WiredTiger会话统计
db.serverStatus().wiredTiger.concurrentTransactions

// 查看锁等待
db.serverStatus().wiredTiger.lock
// 如果lock.wait.time很高，说明存在并发竞争

// 查看阻塞操作
db.currentOp({ "waitingForLock": true })
```

# 6. 连接池优化

## 6.1 连接池配置（Java Driver）

```java
MongoClientSettings settings = MongoClientSettings.builder()
    .applyToConnectionPoolSettings(builder -> 
        builder
            .maxSize(100)          // 最大连接数
            .minSize(10)           // 最小连接数（保持热连接）
            .maxWaitTime(2, TimeUnit.SECONDS)  // 等待连接超时
            .maxConnectionIdleTime(10, TimeUnit.MINUTES)  // 空闲超时
            .maxConnectionLifeTime(30, TimeUnit.MINUTES)  // 连接最大生命周期
    )
    .build();
```

连接池大小建议：
- 默认maxSize: 100
- 一般设为 `(CPU核心数 × 2) ~ 100` 之间
- 过大 → 更多的MongoDB服务端开销
- 过小 → 请求排队等待连接

## 6.2 Spring Boot连接池配置

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mydb?maxPoolSize=50&minPoolSize=5&maxIdleTimeMS=600000&waitQueueTimeoutMS=2000
      # maxPoolSize: 最大连接数(默认100)
      # minPoolSize: 最小连接数(默认0)
      # maxIdleTimeMS: 空闲连接最大存活时间(ms)
      # maxLifeTimeMS: 连接最大生命周期(ms)
      # waitQueueTimeoutMS: 等待连接的队列超时(ms)
      # connectTimeoutMS: 连接超时(ms, 默认10000)
      # socketTimeoutMS: Socket超时(ms, 默认0=无限)
```

# 7. 聚合管道性能优化

## 7.1 聚合管道优化原则

```javascript
// 1. $match必须在管道开头（利用索引）
db.collection.aggregate([
    { $match: { status: "active" } },   // ✅ 索引利用
    { $group: { _id: "$category", count: { $sum: 1 } } }
])

// 2. $project尽早裁剪字段
db.collection.aggregate([
    { $project: { status: 1, category: 1, amount: 1 } },  // ✅ 减少后续内存
    { $group: { _id: "$category", total: { $sum: "$amount" } } }
])

// 3. 大数据量聚合使用allowDiskUse
db.collection.aggregate([...], { allowDiskUse: true })

// 4. 使用$limit提前限制数据量
db.collection.aggregate([
    { $sort: { createTime: -1 } },
    { $limit: 100 },                           // ✅ 限制后续处理
    { $lookup: { from: "users", ... } }
])
```

## 7.2 聚合管道内存限制

```javascript
// $group, $sort, $bucket等阶段默认内存限制100MB
// 超出限制抛错: "exceeded memory limit"

// 解决方案：
// 1. allowDiskUse（性能下降但不会失败）
db.collection.aggregate([...], { allowDiskUse: true })

// 2. 创建索引（$sort阶段）
db.collection.createIndex({ sortField: 1 })

// 3. 使用$match减少进入管道的文档数
// 4. 拆分聚合为多步执行
```

# 8. 硬件与操作系统优化

## 8.1 磁盘

| 指标 | 建议 |
|------|------|
| 类型 | SSD（强烈推荐），NVMe更佳 |
| 文件系统 | XFS（推荐），EXT4也可 |
| RAID | RAID-10（性能+冗余）或不做RAID（副本集处理冗余） |
| 数据与日志分离 | dbPath和journal分不同物理磁盘 |
| 磁盘调度器 | noop或deadline |

## 8.2 内存

- 为操作系统保留至少1~2GB内存
- WiredTiger缓存设为 (总内存 - 1GB) × 50%
- 大量聚合/排序操作时降低缓存大小，为操作系统文件缓存留空间

## 8.3 CPU

- MongoDB的查询处理和压缩/解压需要CPU
- WiredTiger压缩（尤其是zlib）增加CPU消耗
- 一般8~16核心足够，观测CPU使用率决定

## 8.4 操作系统配置

```bash
# Linux内核优化
# /etc/sysctl.conf

# 减少脏页比例（MongoDB有WiredTiger自己管理）
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# 禁用透明大页（THP影响MongoDB性能）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 增大文件句柄限制
# /etc/security/limits.conf
mongod soft nofile 64000
mongod hard nofile 64000

# 禁用atime（减少文件访问时间的IO）
# /etc/fstab: 添加 noatime 选项
UUID=xxx /data xfs defaults,noatime 0 0

# NUMA: 如果服务器有多个NUMA节点，使用numactl绑定
numactl --interleave=all mongod --config /etc/mongod.conf
```

# 9. 性能监控指标

## 9.1 关键监控指标

```javascript
// 连接数
db.serverStatus().connections

// 操作计数器（ops/sec）
db.serverStatus().opcounters

// 操作延迟
db.serverStatus().opLatencies

// 锁等待
db.serverStatus().locks

// 网络吞吐
db.serverStatus().network

// WiredTiger缓存
db.serverStatus().wiredTiger.cache

// 复制延迟
rs.printSecondaryReplicationInfo()

// 当前操作
db.currentOp({ "secs_running": { $gt: 5 } })
```

## 9.2 外部监控工具

| 工具 | 说明 |
|------|------|
| mongostat | 实时opcounters、连接数等 |
| mongotop | 每个集合的读写时间占比 |
| MongoDB Compass | GUI性能分析和可视化 |
| Ops Manager | 企业级监控（图表、报警、自动优化建议） |
| Prometheus + Grafana | 开源监控栈，通过mongodb_exporter |
| mtools | MongoDB日志分析工具集 |

```bash
# mongostat 实时监控
mongostat --host localhost --port 27017 -u admin -p pass --authenticationDatabase admin

# 输出列说明：
# insert/query/update/delete: 每秒操作数
# getmore: 游标获取更多批次的次数
# command: 命令操作数
# dirty: WiredTiger脏缓存比例
# used: WiredTiger缓存使用比例
# vsize/res: 虚拟/物理内存
# qrw/arw: 读写队列长度
# net_in/net_out: 网络输入/输出(KB/s)
# conn: 连接数

# mongotop 集合级别分析
mongotop 5   # 每5秒刷新
```

# 10. 常见性能问题诊断

## 10.1 查询突然变慢

```javascript
// 1. 检查是否有长时间运行的操作阻塞
db.currentOp({ "secs_running": { $gt: 5 } })

// 2. 检查计划缓存是否选择了坏计划
db.collection.getPlanCache().listQueryShapes()

// 3. 清除计划缓存
db.collection.getPlanCache().clear()

// 4. 检查是否有大批量写入或索引构建
db.currentOp({ "msg": { $regex: /index/ } })

// 5. 检查磁盘IO是否饱和
// 查看WiredTiger是否在主动换页
db.serverStatus().wiredTiger.cache["pages evicted by application threads"]
```

## 10.2 写入变慢

```javascript
// 1. 检查写关注级别
// w: "majority"比w: 1慢

// 2. 检查是否有太多索引
// 每个索引都要维护，多键索引维护成本更高

// 3. 检查oplog是否满了
rs.printReplicationInfo()

// 4. 检查磁盘空间
db.stats().storageSize

// 5. 检查Balancer是否在迁移chunk（分片集群）
sh.isBalancerRunning()
```

## 10.3 内存不足的表现

```
- pages evicted by application threads 持续增加
- WiredTiger缓存使用率接近100%
- 操作系统swap使用增加
- 查询延迟波动大
```
