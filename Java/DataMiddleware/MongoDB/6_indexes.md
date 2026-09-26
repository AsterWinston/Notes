
# 1. 索引概述

MongoDB的索引在概念上与传统关系型数据库的索引相似，都是通过维护一种特定的数据结构来提高查询效率。没有索引时，MongoDB必须执行**集合扫描（COLLSCAN）**，扫描集合中的每个文档。

核心要点：
- 索引极大地提高查询性能，但会降低写入速度（需要维护索引）
- 每个索引至少占用8KB磁盘空间
- 每个集合最多64个索引
- 索引键最大1024字节

# 2. 默认_id索引

MongoDB为每个集合的`_id`字段自动创建唯一索引：

```javascript
// 等同于
db.collection.createIndex({ _id: 1 }, { unique: true })

// 该索引不能被删除
```

# 3. 单字段索引

## 3.1 创建单字段索引

```javascript
// 升序索引
db.users.createIndex({ age: 1 })

// 降序索引（单字段索引中升序降序区别不大）
db.users.createIndex({ createTime: -1 })

// 后台创建（不阻塞读写）
db.users.createIndex({ name: 1 }, { background: true })

// 指定索引名称
db.users.createIndex({ email: 1 }, { name: "idx_email" })

// 查看集合所有索引
db.users.getIndexes()

// 查看索引使用统计
db.users.aggregate([{ $indexStats: {} }])
```

## 3.2 单字段索引查询行为

```javascript
// 精准匹配：使用索引
db.users.find({ age: 25 })

// 范围查询：使用索引
db.users.find({ age: { $gte: 20, $lte: 30 } })

// 排序：使用索引（索引本身就是有序的）
db.users.find().sort({ age: 1 })

// 不包含索引字段的查询：不使用索引
db.users.find({ name: "张三" })  // 如果只有age索引
```

# 4. 复合索引

## 4.1 创建复合索引

```javascript
// 创建组合索引（字段顺序很重要！）
db.users.createIndex({ status: 1, age: -1, name: 1 })
```

## 4.2 复合索引的前缀原则

复合索引支持使用索引前缀的查询：

```javascript
// 索引: { status: 1, age: -1, name: 1 }

// 这些查询会使用索引：
db.users.find({ status: "active" })                        // 使用前缀[status]
db.users.find({ status: "active", age: { $gt: 25 } })      // 使用前缀[status, age]
db.users.find({ status: "active", age: 30, name: "张三" })  // 使用全部字段

// 这些查询可能不会使用索引：
db.users.find({ age: 30 })                                 // 不满足最左前缀
db.users.find({ name: "张三" })                             // 不满足最左前缀
db.users.find({ age: 30, name: "张三" })                    // 不满足最左前缀

// 排序也可以利用索引
db.users.find({ status: "active" }).sort({ age: -1 })       // 使用索引排序
db.users.find({ status: "active" }).sort({ age: -1, name: 1 }) // 使用索引排序
db.users.find({ status: "active" }).sort({ name: 1 })       // 无法用索引排序（跳过了age）
```

## 4.3 复合索引中的排序优化

```javascript
// 索引: { status: 1, age: -1 }

// 有效（等值在前，排序字段紧跟且方向一致）
db.users.find({ status: "active" }).sort({ age: -1 })  ✓

// 无效（排序字段前不是等值条件）
db.users.find({ status: "active" }).sort({ name: 1 })  ✗

// 有效（所有前缀字段都是等值条件）
db.users.find({ status: "active", age: 30 }).sort({ name: 1 })  ✓

// 范围查询后的排序需要索引支持
db.users.find({ status: "active", age: { $gt: 25 } }).sort({ name: 1 })  ✓

// 注意：范围查询字段后的排序方向需要匹配
db.users.find({ status: "active", age: { $gt: 25 } }).sort({ name: -1 })  
// 如果索引是 {status:1, age:1, name:1}，可以在内存排序后返回
```

## 4.4 ESR规则（Equality, Sort, Range）

设计复合索引时的黄金法则——ESR规则：

| 顺序 | 规则 | 说明 |
|------|------|------|
| 1. Equality | 先放等值查询字段 | 精确匹配字段放在最前面 |
| 2. Sort | 再放排序字段 | 可以避免内存排序（SORT stage） |
| 3. Range | 最后放范围查询字段 | 范围查询后面的字段无法用于排序 |

```javascript
// 查询: find({ status: "active", age: { $gt: 25 } }).sort({ createTime: -1 })
// 最佳索引: { status: 1, createTime: -1, age: 1 }
//                    ↑EQ        ↑Sort         ↑Range
```

# 5. 多键索引

当对数组字段创建索引时，MongoDB自动创建多键索引。多键索引为数组中的每个元素创建一个索引条目。

```javascript
// tags是数组字段
db.users.createIndex({ tags: 1 })

// 这些查询可以使用多键索引：
db.users.find({ tags: "Java" })
db.users.find({ tags: { $in: ["Java", "Python"] } })
db.users.find({ tags: { $all: ["Java", "MongoDB"] } })

// 注意：一个复合索引中最多只能有一个数组字段
// 因为多键索引需要为每个数组元素创建条目，两个数组会产生笛卡尔积
```

多键索引限制：
- 一个复合索引中只能包含一个数组字段
- 不能创建`{ a: 1, b: 1 }`如果a和b都是数组（插入时会报错）
- 分片键不能是多键索引

# 6. 文本索引

## 6.1 创建文本索引

```javascript
// 单字段文本索引
db.articles.createIndex({ content: "text" })

// 多字段文本索引（一个集合只能有一个文本索引）
db.articles.createIndex({ title: "text", content: "text" })

// 指定权重
db.articles.createIndex(
    { title: "text", content: "text" },
    { weights: { title: 10, content: 5 } }
)

// 指定默认语言
db.articles.createIndex(
    { content: "text" },
    { default_language: "chinese" }
)
```

## 6.2 全文搜索查询

```javascript
// 基本文本搜索
db.articles.find({ $text: { $search: "MongoDB 索引" } })

// 精确短语搜索
db.articles.find({ $text: { $search: "\"MongoDB 索引优化\"" } })

// 排除词
db.articles.find({ $text: { $search: "MongoDB -MySQL" } })

// 按相关度排序
db.articles.find(
    { $text: { $search: "MongoDB 索引" } },
    { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })

// 多语言搜索
db.articles.find({ $text: { $search: "database", $language: "english" } })
```

文本索引限制：
- 一个集合最多一个文本索引
- 文本索引占用空间较大
- 中文分词默认效果一般，生产环境建议配合Atlas Search或Elasticsearch

# 7. 地理空间索引

## 7.1 dsphere索引（地球表面）

```javascript
// 创建2dsphere索引
db.locations.createIndex({ location: "2dsphere" })

// 插入数据（GeoJSON格式）
db.locations.insertOne({
    name: "天安门",
    location: { type: "Point", coordinates: [116.397, 39.908] }
})

// 查询附近位置
db.locations.find({
    location: {
        $nearSphere: {
            $geometry: { type: "Point", coordinates: [116.397, 39.908] },
            $maxDistance: 5000
        }
    }
})
```

## 7.2 d索引（平面坐标）

```javascript
// 创建2d索引
db.places.createIndex({ location: "2d" })

// 插入数据（数组格式）
db.places.insertOne({
    name: "位置A",
    location: [116, 39]
})

// 查询矩形范围内
db.places.find({
    location: { $geoWithin: { $box: [[116, 39], [117, 40]] } }
})
```

# 8. 哈希索引

哈希索引存储索引字段的哈希值，仅支持等值查询，不支持范围查询。主要用于分片集群中实现数据均匀分布。

```javascript
// 创建哈希索引
db.users.createIndex({ userId: "hashed" })

// 仅支持等值查询
db.users.find({ userId: "U001" })  // 使用哈希索引
db.users.find({ userId: { $gt: "U001" } })  // 不使用哈希索引
```

# 9. 其他索引类型

## 9.1 TTL索引

TTL（Time-To-Live）索引自动过期和删除文档，适用于会话、缓存、日志等有时效性的数据。

```javascript
// 创建TTL索引（expireAfterSeconds单位：秒）
db.sessions.createIndex(
    { createTime: 1 },
    { expireAfterSeconds: 3600 }   // 1小时后自动删除
)

// TTL索引限制：
// - 字段必须是Date类型或Date数组
// - 只能是单字段索引
// - 不支持复合索引
// - _id字段不支持TTL索引
// - 后台线程每60秒运行一次
```

## 9.2 唯一索引

```javascript
// 单字段唯一索引
db.users.createIndex({ email: 1 }, { unique: true })

// 复合唯一索引
db.users.createIndex({ groupId: 1, userId: 1 }, { unique: true })

// 稀疏唯一索引（允许多个null值或缺失字段）
db.users.createIndex(
    { phone: 1 },
    { unique: true, sparse: true }
)
```

## 9.3 部分索引

仅对满足筛选条件的文档创建索引，减少索引大小和维护开销：

```javascript
// 仅对active状态用户创建email索引
db.users.createIndex(
    { email: 1 },
    {
        partialFilterExpression: { status: "active" }
    }
)

// 部分索引+唯一约束组合
db.users.createIndex(
    { email: 1 },
    {
        unique: true,
        partialFilterExpression: { email: { $exists: true, $ne: null } }
    }
)
```

## 9.4 稀疏索引

稀疏索引只包含有索引字段的文档，缺失该字段的文档不进入索引：

```javascript
db.users.createIndex({ phone: 1 }, { sparse: true })

// 查询会使用稀疏索引
db.users.find({ phone: "13800138000" })

// 查询可能不会使用稀疏索引（需要sort）
db.users.find().sort({ phone: 1 })

// hint可强制使用
db.users.find().sort({ phone: 1 }).hint({ phone: 1 })
```

## 9.5 通配符索引

MongoDB 4.2+支持通配符索引，自动为文档中所有字段或其子集创建索引：

```javascript
// 为所有字段创建通配符索引
db.collection.createIndex({ "$**": 1 })

// 为嵌套字段创建通配符索引
db.collection.createIndex({ "attributes.$**": 1 })

// 排除特定字段
db.collection.createIndex(
    { "$**": 1 },
    { wildcardProjection: { "_id": 0, "secretField": 0 } }
)
```

适用场景：属性模式（Attribute Pattern），其中文档结构不确定但需要查询能力。

## 9.6 隐藏索引

隐藏索引对查询优化器不可见，但仍在维护：

```javascript
// 创建隐藏索引
db.collection.createIndex({ field: 1 }, { hidden: true })

// 隐藏/取消隐藏现有索引
db.collection.hideIndex("indexName")
db.collection.unhideIndex("indexName")
```

用途：评估删除索引的影响，先隐藏观察，没问题再彻底删除。

# 10. 索引属性详解

## 10.1 索引创建选项

```javascript
db.collection.createIndex({ field: 1 }, {
    name: "my_custom_name",           // 自定义索引名
    background: true,                 // 后台创建
    unique: true,                     // 唯一索引
    sparse: true,                     // 稀疏索引
    expireAfterSeconds: 3600,         // TTL
    partialFilterExpression: {...},   // 部分索引
    hidden: true,                     // 隐藏索引
    collation: { locale: "zh" },      // 排序规则
    storageEngine: { wiredTiger: { configString: "..." } },  // 引擎特定配置
    weights: {...}                    // 文本索引权重
})
```

## 10.2 索引名称自动生成规则

- 默认名称格式：`fieldname_direction_fieldname_direction`
- 示例：`{ name: 1, age: -1 }` → `"name_1_age_-1"`
- 名称最大127字符
- 通过`name`选项自定义名称

# 11. 索引管理

```javascript
// 查看所有索引
db.collection.getIndexes()

// 查看索引大小
db.collection.totalIndexSize()
db.collection.stats().indexSizes

// 删除索引
db.collection.dropIndex("indexName")
db.collection.dropIndex({ field: 1 })

// 删除所有非_id索引
db.collection.dropIndexes()

// 重建索引（整理碎片）
db.collection.reIndex()

// 查看索引构建状态
db.currentOp({ "command.createIndexes": { $exists: true } })
```

# 12. 执行计划分析（explain）

## 12.1 explain概览

```javascript
// explain三种模式
db.collection.find({ field: "value" }).explain("queryPlanner")     // 默认：查询计划
db.collection.find({ field: "value" }).explain("executionStats")   // 执行统计
db.collection.find({ field: "value" }).explain("allPlansExecution") // 所有计划的执行
```

## 12.2 explain输出解读

```javascript
// 关键输出结构
{
    queryPlanner: {
        winningPlan: {
            stage: "FETCH",              // 根阶段
            inputStage: {
                stage: "IXSCAN",         // 索引扫描
                keyPattern: { age: 1 },
                indexName: "age_1",
                direction: "forward"
            }
        },
        rejectedPlans: [...]            // 被拒绝的计划
    },
    executionStats: {
        nReturned: 100,                 // 返回文档数
        totalKeysExamined: 100,         // 扫描的索引条目
        totalDocsExamined: 100,         // 扫描的文档数
        executionTimeMillis: 2,         // 执行时间(ms)
        executionStages: {...}          // 详细阶段信息
    }
}
```

## 12.3 执行计划阶段（Stage）含义

| Stage | 含义 | 优化建议 |
|-------|------|---------|
| COLLSCAN | 全集合扫描 | 创建索引 |
| IXSCAN | 索引扫描 | 正常（用到了索引） |
| FETCH | 通过索引获取文档 | 正常 |
| SORT | 内存排序 | 优化索引方向或添加排序字段到索引 |
| SORT_KEY_GENERATOR | 生成排序键 | 同SORT |
| LIMIT | 限制返回数量 | 正常 |
| SKIP | 跳过文档 | 避免大offset，使用游标分页 |
| SHARD_MERGE | 合并分片结果 | 分片集群正常行为 |
| TEXT | 全文搜索 | 正常 |
| PROJECTION_COVERED | 覆盖查询（最少） | 已是最优 |
| COUNT_SCAN | count扫描 | 正常 |
| OR | OR条件合并 | 确保OR各分支有索引 |

## 12.4 覆盖查询

当查询所需的所有字段都在索引中时，MongoDB不需要访问实际文档：

```javascript
// 索引: { name: 1, age: 1 }
// 覆盖查询：只返回索引中的字段
db.users.find(
    { name: "张三" },
    { name: 1, age: 1, _id: 0 }
)
// stage: PROJECTION_COVERED — 最优！

// 非覆盖查询（需要email字段）
db.users.find(
    { name: "张三" },
    { name: 1, email: 1 }
)
// stage: FETCH — 需要读取文档
```

## 12.5 糟糕的查询示例及优化

```javascript
// 糟糕的查询
db.orders.find({
    $or: [
        { customerId: "C001" },
        { orderDate: { $gte: ISODate("2024-01-01") } }
    ]
}).sort({ total: -1 }).skip(1000).limit(20)

// explain("executionStats") 输出分析：
// - totalDocsExamined: 50000（扫描了5万文档）
// - totalKeysExamined: 50000
// - executionTimeMillis: 350
// - stage: SORT（内存排序）

// 优化方案：
// 1. 为$or的每个分支创建索引
db.orders.createIndex({ customerId: 1 })
db.orders.createIndex({ orderDate: 1 })
// 2. 避免大offset分页（使用游标分页）
// 3. 考虑创建覆盖索引避免FETCH阶段
```

# 13. 索引使用技巧与最佳实践

## 13.1 用$indexStats分析索引使用

```javascript
// 查看索引使用统计（哪些索引被使用、哪些从未使用）
db.collection.aggregate([{ $indexStats: {} }])

// 返回示例解读：
// accesses: 索引被使用的次数
// ops: 上次使用以来的操作次数
// since: 索引被使用的最后时间
```

## 13.2 强制使用/忽略索引

```javascript
// 强制使用指定索引
db.collection.find({ field: "value" }).hint({ field: 1 })
db.collection.find({ field: "value" }).hint("indexName")

// 忽略所有索引（走全表扫描）
db.collection.find({ field: "value" }).hint({ $natural: 1 })
```

## 13.3 索引建议清单

1. **所有查询都应该有索引覆盖**：通过explain确认是IXSCAN而非COLLSCAN
2. **遵循ESR规则设计复合索引**：Equality → Sort → Range
3. **避免冗余索引**：如已有`{a:1, b:1}`，则`{a:1}`是冗余的
4. **定期分析和清理无用索引**：使用`$indexStats`检查
5. **注意索引选择性**：字段值差异越大，索引效率越高
6. **控制索引数量**：每个集合最好不超过5-10个索引
7. **使用background:true**：在生产环境避免阻塞读写（4.2+已是默认行为）
8. **覆盖查询优于FETCH**：返回字段尽量让索引覆盖
9. **监控慢查询**：定期检查system.profile集合
10. **避免索引字段上的运算**：`{ $inc: { field: 1 } }`会正确维护索引，但`{ $mul }`也会维护

## 13.4 索引的代价

```
写入性能损耗 ≈ 0.1~0.3ms × 索引数量（每个文档每次写入）
磁盘开销 ≈ 索引大小通常为数据大小的10%~30%

示例：
- 100GB数据，5个索引 → 索引约10-30GB
- 每秒写入10000文档，3个索引 → 额外开销约3-9ms/文档
```
