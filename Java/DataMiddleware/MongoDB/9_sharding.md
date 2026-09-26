
# 1. 分片概述

分片是将数据水平拆分到多个服务器上的过程。当一个MongoDB集合的数据量过大时（如数百GB或数TB），单台服务器可能无法承载，分片提供了水平扩展的能力。

## 1.1 何时需要分片

| 信号 | 说明 |
|------|------|
| 磁盘空间不足 | 数据超过单机磁盘容量 |
| 内存不足 | 工作集超过服务器内存 |
| 写入瓶颈 | 单机无法承受写入吞吐量 |
| 数据局部性 | 需要将特定地域数据存储在特定服务器 |

## 1.2 分片集群架构

```
┌─────────────────────────────────────────────────────┐
│                    客户端 (Driver)                    │
└──────────────────────┬──────────────────────────────┘
                       │
               ┌───────▼────────┐
               │   mongos 路由器  │  (多个，无状态)
               │   (Query Router) │
               └─┬────────────┬──┘
                 │            │
        ┌────────▼──┐  ┌─────▼─────────┐
        │ Config     │  │ Config         │
        │ Server     │  │ Server  (副本集)│
        │ (副本集)    │  │                │
        └────────────┘  └────────────────┘
        ┌────────────────────────────────┐
        │          Shard 1 (副本集)        │
        │  ┌──────┐ ┌──────┐ ┌──────┐   │
        │  │P     │ │S     │ │S     │   │
        │  └──────┘ └──────┘ └──────┘   │
        ├────────────────────────────────┤
        │          Shard 2 (副本集)        │
        │  ┌──────┐ ┌──────┐ ┌──────┐   │
        │  │P     │ │S     │ │S     │   │
        │  └──────┘ └──────┘ └──────┘   │
        ├────────────────────────────────┤
        │          Shard N (副本集)        │
        └────────────────────────────────┘
```

## 1.3 核心组件

| 组件 | 说明 |
|------|------|
| Shard（分片） | 存储实际数据子集，每个分片是一个副本集 |
| Config Server | 存储集群元数据（chunk路由信息、分片键范围） |
| Mongos | 查询路由器，客户端连接的入口，负责将请求路由到正确的分片 |

# 2. 分片键（Shard Key）

## 2.1 分片键的重要性

分片键是分片集群设计中**最重要的决策**，一旦选定几乎不可更改（7.0前需要reshardCollection，5.0+支持实时重新分片）。

分片键决定了：
- 数据在分片间的分布是否均匀
- 查询是否能被定向到特定分片（targeted query）
- 写入是否能在分片间均衡分布

## 2.2 分片键类型与选择

**范围分片（Range Sharding）**：

MongoDB根据分片键的值将数据划分为多个chunk（数据块），每个chunk存储一个连续范围的数据。

```javascript
// 创建范围分片
sh.shardCollection("mydb.users", { age: 1 })

// age范围分片示意：
// Shard1: [0, 25)
// Shard2: [25, 50)
// Shard3: [50, ∞)

// 特点：
// ✅ 范围查询效率高
// ✅ 相邻数据存储在同一个分片
// ❌ 写入热点（如按时间戳递增的分片键，所有写入集中在当前时间段的分片）
```

**哈希分片（Hash Sharding）**：

MongoDB计算分片键的哈希值，根据哈希值范围分配chunk。

```javascript
// 创建哈希分片
sh.shardCollection("mydb.users", { userId: "hashed" })

// 特点：
// ✅ 数据分布均匀（没有热点）
// ✅ 写入均匀分散
// ❌ 范围查询效率低（需要访问所有分片）
// ❌ 无法使用唯一哈希索引外的唯一索引
```

## 2.3 分片键选择标准

| 标准 | 说明 |
|------|------|
| 高基数（High Cardinality） | 分片键的值足够多，支持chunk均匀分布 |
| 低频率（Low Frequency） | 每个分片键值对应的文档数不要过于集中 |
| 非单调递增 | 避免写入热点（单调递增 → 所有新写入集中在最大值的chunk） |
| 符合查询模式 | 大多数查询应能定向到特定分片 |

```javascript
// ✅ 好的分片键
{ userId: "hashed" }            // 基数高、均匀、分布式查询
{ orderId: "hashed" }           // 同上
{ region: 1, userId: 1 }       // 复合键，优先使用region定向
{ companyId: 1, _id: 1 }       // 先按租户隔离，内部再分配

// ❌ 不好的分片键
{ status: 1 }                   // 基数极低（只有几种状态）
{ isDeleted: 1 }               // 基数极低（true/false）
{ createTime: 1 }              // 单调递增、写入热点
{ gender: "hashed" }           // 基数低，即使哈希也无法均匀
```

## 2.4 复合分片键

```javascript
// 复合分片键：多层分组
sh.shardCollection("mydb.orders", {
    companyId: 1,
    createTime: 1
})

// 优势：
// - companyId定向查询到特定分片
// - createTime在分片内实现范围查询
// 劣势：
// - companyId基数不够高时数据倾斜
```

## 2.5 Chunk（数据块）

Chunk是MongoDB分片集群中数据分布的最小单位。每个chunk包含一个分片键范围内的文档集合。

```javascript
// 默认chunk大小：64MB
// 可以调整
sh.setBalancerState(true)
db.settings.updateOne(
    { _id: "chunksize" },
    { $set: { value: 256 } }    // 256MB
)

// Chunk分裂：
// 当chunk超过指定大小，MongoDB自动分裂为两个chunk
// 分裂只改变元数据，不实际移动数据
```

**Chunk迁移流程**：
1. Balancer判断分片负载不均
2. 从源分片复制chunk数据到目标分片
3. 更新config server元数据
4. 删除源分片上的旧chunk数据

# 3. 分片集群设置

## 3.1 Config Server设置

```bash
# Config Server副本集
mongod --configsvr --replSet configReplSet --port 27019 --dbpath /data/configdb
```

```javascript
// 初始化Config Server副本集
rs.initiate({
    _id: "configReplSet",
    configsvr: true,
    members: [
        { _id: 0, host: "config1:27019" },
        { _id: 1, host: "config2:27019" },
        { _id: 2, host: "config3:27019" }
    ]
})
```

## 3.2 分片节点设置

```bash
# 每个分片是一个副本集
mongod --shardsvr --replSet shard1ReplSet --port 27018 --dbpath /data/shard1
```

```javascript
// 初始化分片副本集
rs.initiate({
    _id: "shard1ReplSet",
    members: [
        { _id: 0, host: "shard1a:27018" },
        { _id: 1, host: "shard1b:27018" },
        { _id: 2, host: "shard1c:27018" }
    ]
})
```

## 3.3 Mongos路由设置

```bash
# 启动mongos（通常也需要多个实例）
mongos --configdb configReplSet/config1:27019,config2:27019,config3:27019 --port 27017
```

## 3.4 添加分片与启用分片

```javascript
// 通过mongos连接
mongosh --port 27017

// 添加分片
sh.addShard("shard1ReplSet/shard1a:27018")
sh.addShard("shard2ReplSet/shard2a:27018")
sh.addShard("shard3ReplSet/shard3a:27018")

// 查看分片状态
sh.status()

// 为数据库启用分片
sh.enableSharding("mydb")

// 为集合启用分片（指定分片键）
sh.shardCollection("mydb.users", { userId: "hashed" })
sh.shardCollection("mydb.orders", { companyId: 1, createTime: 1 })

// 预分片（提前将chunk分配到各分片，避免初始热点）
sh.shardCollection("mydb.users", { userId: "hashed" })
// 预分割chunk
for (let i = 0; i < 100; i++) {
    sh.splitAt("mydb.users", { userId: MinKey })
}
```

# 4. 查询路由机制

## 4.1 Targeted Query（定向查询）

当查询包含分片键时，mongos可以将查询定向到特定分片：

```javascript
// 分片键: { userId: "hashed" }
// 定向查询（只在1个分片上执行）
db.users.find({ userId: "U001" })

// 分片键: { companyId: 1, createTime: 1 }
// 定向查询
db.orders.find({ companyId: "公司A", createTime: { $gt: ... } })
```

## 4.2 Scatter-Gather Query（广播查询）

当查询不包含分片键时，mongos需要将查询广播到所有分片：

```javascript
// 广播查询（在所有分片上执行，然后合并结果）
db.users.find({ name: "张三" })            // name不是分片键
db.users.find({})                          // 无过滤条件
db.users.find({ status: "active" })        // status不是分片键
```

广播查询性能远差于定向查询，这就是为什么分片键需要与查询模式匹配。

## 4.3 分片集群中的排序、分页

```javascript
// 排序：mongos从各分片获取排序结果后合并排序
db.users.find({}).sort({ createTime: -1 }).limit(20)

// 分页陷阱：
// skip+limit在分片集群中效率极低
db.users.find({}).sort({ _id: -1 }).skip(100000).limit(20)
// 每个分片都要扫描至少100020条文档，然后发送给mongos合并！

// 推荐：游标分页
db.users.find({ _id: { $lt: lastId } }).sort({ _id: -1 }).limit(20)
```

# 5. Balancer（均衡器）

## 5.1 Balancer工作机制

Balancer负责在分片之间迁移chunk，使数据分布均匀：

```javascript
// 查看Balancer状态
sh.getBalancerState()
sh.isBalancerRunning()

// 停止Balancer
sh.stopBalancer()

// 启动Balancer
sh.startBalancer()

// 设置Balancer运行时间窗口（避开业务高峰期）
db.settings.updateOne(
    { _id: "balancer" },
    {
        $set: {
            activeWindow: {
                start: "02:00",
                stop: "06:00"
            }
        }
    }
)
```

## 5.2 Chunk迁移控制

```javascript
// 手动迁移chunk
sh.moveChunk(
    "mydb.users",
    { userId: "U001" },
    "shard2ReplSet"
)

// 查看迁移状态
sh.status()
```

# 6. 分片集群管理

## 6.1 常用管理命令

```javascript
// 查看集群状态
sh.status()

// 查看分片分布
db.collection.getShardDistribution()

// 查看chunk分布
db.collection.getShardVersion()

// 合并空chunk
db.adminCommand({ mergeAllChunksOnShard: "mydb.users" })

// 查看config数据库内容
use config
db.shards.find().pretty()       // 分片信息
db.databases.find().pretty()    // 数据库分片状态
db.collections.find().pretty()  // 集合分片状态
db.chunks.find().pretty()       // chunk分布
db.mongos.find().pretty()       // mongos实例
```

## 6.2 重新分片（5.0+）

```javascript
// 5.0+支持的实时重新分片
db.adminCommand({
    reshardCollection: "mydb.users",
    key: { userId: 1 }          // 新的分片键
})

// 查看重新分片进度
db.adminCommand({ currentOp: true, desc: "Resharding" })
```

## 6.3 区域（Zone）分片

将特定范围的分片键值指定到特定分片（如按地理位置分片）：

```javascript
// 为分片分配区域标签
sh.addShardTag("shard1ReplSet", "ASIA")
sh.addShardTag("shard2ReplSet", "EUROPE")
sh.addShardTag("shard3ReplSet", "AMERICA")

// 为分片键值范围分配区域
sh.addTagRange(
    "mydb.users",
    { region: "CN" }, { region: "CN"+"\uffff" },  // 中国范围
    "ASIA"
)
sh.addTagRange(
    "mydb.users",
    { region: "US" }, { region: "US"+"\uffff" },
    "AMERICA"
)

// 移除配置
sh.removeTagRange("mydb.users", { region: "CN" }, { region: "CN"+"\uffff" })
sh.removeShardTag("shard1ReplSet", "ASIA")
```

# 7. 分片集群限制与注意事项

| 限制 | 说明 |
|------|------|
| 分片键不可变 | 文档的分片键值创建后不可更改 |
| 分片键大小 | 分片键索引条目不超过512字节 |
| 唯一索引 | 唯一索引必须包含分片键（哈希分片除外） |
| $lookup | 被关联的集合如果是非分片集合，不受影响 |
| 事务 | 跨分片事务有一定性能开销 |
| geoSearch | 不支持geoSearch命令 |
| group | 不支持group命令（使用$group聚合） |

## 7.1 分片键不可变的影响

```javascript
// 不允许的更新操作
db.users.updateOne(
    { _id: "U001" },
    { $set: { userId: "U002" } }   // 尝试修改分片键 → 报错！
)
```

# 8. 分片集群最佳实践

1. **分片键选择是重中之重**：一旦选定很难更改
2. **使用哈希分片降低热点**：如果查询多是等值查询
3. **确保大多数查询是定向查询**：查询必须包含分片键
4. **预分片策略**：预先创建空chunk，避免初始阶段数据聚集
5. **监控chunk平衡**：确保数据均匀分布
6. **合理设置chunk大小**：一般64MB，大数据集可适当增大
7. **避免频繁chunk迁移**：高峰期停止Balancer
8. **使用Zone分片实现数据本地性**：GDPR合规等场景
9. **mongos数量充足**：每个应用服务器可部署一个mongos
10. **监控config server**：config server不可用会导致集群元数据无法更新
