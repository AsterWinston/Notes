
# 1. CRUD操作概览

MongoDB的CRUD操作是基于文档的，所有操作针对某个集合（Collection）执行。以下是CRUD操作的核心方法：

| 操作类型 | 单文档 | 多文档 | 批量 |
|---------|--------|--------|------|
| 创建(Create) | insertOne | insertMany | bulkWrite |
| 读取(Read) | findOne, find | — | — |
| 更新(Update) | updateOne, replaceOne | updateMany | bulkWrite |
| 删除(Delete) | deleteOne | deleteMany | bulkWrite |

# 2. 数据库与集合操作

## 2.1 数据库操作

```javascript
// 查看所有数据库
show dbs

// 查看当前数据库
db

// 切换/创建数据库（隐式创建，插入数据时才真正创建）
use mydb

// 查看当前数据库统计信息
db.stats()

// 删除当前数据库（谨慎操作）
db.dropDatabase()
```

数据库命名规范：
- 不能包含空字符、空格、`.`、`$`、`/`、`\`、`\0`
- 应全部小写
- 最多64个字符
- 保留名称：admin、local、config

## 2.2 集合操作

```javascript
// 显式创建集合
db.createCollection("users")
db.createCollection("logs", { capped: true, size: 1048576, max: 1000 })

// 隐式创建集合（插入文档时自动创建）
db.users.insertOne({ name: "张三" })

// 查看所有集合
show collections

// 查看集合详情
db.users.stats()

// 重命名集合
db.users.renameCollection("accounts")

// 删除集合
db.users.drop()
```

集合选项：

| 选项 | 说明 |
|------|------|
| capped: true | 固定大小集合（类似环形缓冲区） |
| size: <bytes> | 固定集合最大字节数 |
| max: <number> | 固定集合最大文档数 |
| validator | 文档校验规则 |
| validationLevel | 校验严格程度（strict/moderate） |
| validationAction | 校验失败行为（error/warn） |

## 2.3 固定集合（Capped Collection）

```javascript
// 创建一个最大1MB、最多1000条日志的固定集合
db.createCollection("system_log", {
    capped: true,
    size: 1048576,  // 1MB
    max: 1000
})
```

特性：
- 插入顺序保证与自然顺序一致
- 磁盘空间固定，达到上限时覆盖最旧文档
- 不支持文档删除（可用drop整个集合）
- 文档更新不能导致文档大小增长
- 天然支持tailable cursor（类似tail -f）

# 3. 插入操作

## 3.1 insertOne

```javascript
// 插入单条文档
db.users.insertOne({
    name: "张三",
    age: 28,
    email: "zhangsan@example.com",
    tags: ["Java", "MongoDB"],
    createTime: new Date()
})

// 返回结果
{
    "acknowledged": true,
    "insertedId": ObjectId("507f1f77bcf86cd799439011")
}
```

## 3.2 insertMany

```javascript
// 插入多条文档
db.users.insertMany([
    { name: "李四", age: 25, tags: ["Python"] },
    { name: "王五", age: 30, tags: ["Go", "Docker"] },
    { name: "赵六", age: 22, tags: ["JavaScript"] }
])

// 可选选项
db.users.insertMany(
    [
        { name: "孙七", age: 35 },
        { name: "周八", age: 28 }
    ],
    { ordered: false }  // 遇到错误继续插入剩余的
)
```

`ordered`参数：
- `true`（默认）：按顺序插入，遇到错误停止
- `false`：忽略错误，继续插入后续文档

## 3.3 插入行为细节

**自动添加_id**：
- 如果没有指定`_id`字段，MongoDB自动生成ObjectId
- 如果指定了`_id`，必须保证集合内唯一
- 不能使用数组作为`_id`值

**原子性**：
- 每个文档的插入操作是原子的（单文档原子性）
- insertMany不是原子操作（4.0+可用事务实现批量原子操作）

**文档大小限制**：
- 单个BSON文档最大16MB
- 超过此限制使用GridFS存储

```javascript
// 自定义_id
db.users.insertOne({
    _id: "user_1001",
    name: "自定义ID用户"
})

// 检查文档大小
Object.bsonsize({ name: "张三", age: 28 })  // 返回字节数
```

## 3.4 Insert性能优化

```javascript
// 无序批量插入（性能最优）
db.users.insertMany(
    largeArrayOfDocuments,
    { ordered: false, writeConcern: { w: 0 } }
)

// 在副本集中减小写关注级别可以大幅提高写入性能
// w: 0  → 不等待确认（最快，但不可靠）
// w: 1  → 等待主节点确认（默认，足够）
// w: "majority" → 等待大多数节点确认（最安全）
```

# 4. 查询操作

## 4.1 find基础

```javascript
// 查询所有文档（返回游标）
db.users.find()

// 查询所有文档（返回数组，最多20条）
db.users.find().toArray()

// 条件查询
db.users.find({ age: 28 })

// 查询第一条匹配文档
db.users.findOne({ name: "张三" })

// 指定返回字段（投影）
db.users.find({}, { name: 1, age: 1, _id: 0 })
```

## 4.2 游标（Cursor）

`find()`返回的是一个游标，惰性求值，仅在迭代时才真正执行查询。

```javascript
// 基础游标操作
var cursor = db.users.find({ age: { $gte: 25 } })

while (cursor.hasNext()) {
    printjson(cursor.next())
}

// 游标方法链
db.users.find()
    .limit(10)           // 限制返回10条
    .skip(20)            // 跳过前20条
    .sort({ age: -1 })   // 按年龄降序
    .projection({ name: 1, age: 1 })

// 遍历游标
db.users.find().forEach(doc => {
    print(doc.name)
})
```

游标生命周期：
- 默认10分钟无活动后自动关闭（`noCursorTimeout`标志可覆盖）
- 服务端游标批次大小默认101条文档
- 游标耗尽后自动关闭
- 主动关闭：`cursor.close()`

```javascript
// 设置批次大小
db.users.find().batchSize(50)

// 查看游标状态
db.serverStatus().metrics.cursor
```

## 4.3 投影（Projection）

投影用于控制返回哪些字段，通过第二个参数指定：

```javascript
// 只包含name和age字段
db.users.find({}, { name: 1, age: 1 })

// 排除email字段
db.users.find({}, { email: 0 })

// 不能混合包含和排除（_id除外）
db.users.find({}, { name: 1, email: 0 })  // 报错！

// _id可以单独控制
db.users.find({}, { name: 1, age: 1, _id: 0 })

// 投影嵌套字段
db.users.find({}, { "address.city": 1, _id: 0 })

// 投影数组（$slice, $elemMatch）
db.users.find({}, { tags: { $slice: 2 } })               // 数组前2个元素
db.users.find({}, { tags: { $slice: -2 } })              // 数组后2个元素
db.users.find({}, { tags: { $slice: [1, 3] } })          // 跳过1个，取3个
db.users.find({}, { scores: { $elemMatch: { $gte: 80 } } })  // 匹配条件的数组元素
```

# 5. 更新操作

## 5.1 updateOne

```javascript
// 更新匹配到的第一条文档
db.users.updateOne(
    { name: "张三" },              // 过滤条件
    { $set: { age: 29 } }          // 更新操作
)

// 更新不存在则插入（upsert）
db.users.updateOne(
    { name: "新用户" },
    { $set: { age: 20, email: "new@example.com" } },
    { upsert: true }
)
```

## 5.2 updateMany

```javascript
// 更新所有匹配的文档
db.users.updateMany(
    { age: { $lt: 30 } },          // 过滤条件
    { $inc: { age: 1 } }           // 所有匹配文档age+1
)
```

## 5.3 replaceOne

```javascript
// 完全替换文档（除了_id）
db.users.replaceOne(
    { name: "张三" },
    {
        name: "张三",
        age: 30,
        email: "new_email@example.com",
        department: "技术部"
    }
)
```

注意：`replaceOne`是整体替换，如果新文档缺少某个字段，该字段会被删除。不能包含更新操作符（如$set）。

## 5.4 更新操作符大全

**字段更新操作符**：

| 操作符 | 功能 | 示例 |
|--------|------|------|
| $set | 设置字段值 | {$set: { age: 30 }} |
| $unset | 删除字段 | {$unset: { email: "" }} |
| $inc | 数值增减 | {$inc: { age: 1, score: -5 }} |
| $mul | 数值乘法 | {$mul: { price: 1.2 }} |
| $rename | 字段重命名 | {$rename: { "old_name": "new_name" }} |
| $min | 仅在小于当前值更新 | {$min: { score: 60 }} |
| $max | 仅在大于当前值更新 | {$max: { score: 90 }} |
| $currentDate | 设为当前时间 | {$currentDate: { lastModified: true }} |
| $setOnInsert | 仅upsert时设置 | {$setOnInsert: { createTime: new Date() }} |

```javascript
// 组合使用多个操作符
db.users.updateOne(
    { name: "张三" },
    {
        $set: { "address.city": "上海" },        // 设置嵌套字段
        $unset: { tempField: "" },               // 删除字段
        $inc: { loginCount: 1 },                 // 自增
        $currentDate: { lastLogin: true },       // 当前时间
        $rename: { "nick": "nickname" }          // 重命名
    }
)
```

**数组更新操作符**：

| 操作符 | 功能 | 示例 |
|--------|------|------|
| $push | 向数组追加元素 | {$push: { tags: "Redis" }} |
| $push (with $each) | 批量追加 | {$push: { tags: { $each: ["a","b"] }}} |
| $addToSet | 向数组添加不重复元素 | {$addToSet: { tags: "Java" }} |
| $pop | 移除数组首/尾元素 | {$pop: { tags: 1 }} (1尾-1首) |
| $pull | 移除匹配条件的元素 | {$pull: { tags: "Java" }} |
| $pullAll | 移除指定的多个元素 | {$pullAll: { tags: ["a","b"] }} |
| $each + $slice | 追加后截断数组 | 见下方示例 |
| $each + $sort | 追加后排序 | 见下方示例 |
| $each + $position | 追加到指定位置 | 见下方示例 |
| $\<operator\> | 更新匹配条件的数组元素 | {$set: { "scores.$": 100 }} |

```javascript
// $push高级用法：$each + $slice + $sort + $position
// 添加多个元素，保留最后5个，按score降序排列
db.users.updateOne(
    { name: "张三" },
    {
        $push: {
            scores: {
                $each: [
                    { subject: "数学", score: 95 },
                    { subject: "英语", score: 88 }
                ],
                $sort: { score: -1 },
                $slice: -5
            }
        }
    }
)

// 指定位置插入
db.users.updateOne(
    { name: "张三" },
    { $push: { tags: { $each: ["Python"], $position: 0 } } }
)

// $操作符：更新数组中第一个匹配的元素
db.users.updateOne(
    { name: "张三", "scores.subject": "数学" },
    { $set: { "scores.$.score": 100 } }
)

// $[]操作符：更新数组中所有元素
db.users.updateOne(
    { name: "张三" },
    { $set: { "scores.$[].status": "updated" } }
)

// $[identifier]操作符（arrayFilters）：条件更新数组元素
db.users.updateOne(
    { name: "张三" },
    { $inc: { "scores.$[elem].score": 5 } },
    { arrayFilters: [{ "elem.score": { $lt: 90 } }] }
)
```

# 6. 删除操作

## 6.1 deleteOne

```javascript
// 删除匹配到的第一条文档
db.users.deleteOne({ name: "张三" })
```

## 6.2 deleteMany

```javascript
// 删除所有匹配的文档
db.users.deleteMany({ age: { $lt: 18 } })

// 删除集合所有文档（清空集合）
db.users.deleteMany({})

// 更快的清空方式（删除集合和索引后重建）
db.users.drop()
```

## 6.3 findOneAndDelete

```javascript
// 查找并删除，返回被删除的文档
var deleted = db.users.findOneAndDelete(
    { status: "expired" },
    { sort: { createTime: 1 } }   // 删除最早的
)
```

# 7. 原子操作：findAndModify系列

`findOneAndUpdate`、`findOneAndReplace`、`findOneAndDelete`是原子操作，查找并修改/删除一个文档，可以返回修改前或修改后的文档。

```javascript
// findOneAndUpdate：原子查找并更新
var result = db.users.findOneAndUpdate(
    { name: "张三" },
    { $inc: { visitCount: 1 } },
    {
        returnDocument: "after",   // "before" | "after" (默认before)
        projection: { visitCount: 1 },
        sort: { _id: -1 },
        upsert: false
    }
)

// findOneAndReplace：原子查找并替换
var result = db.users.findOneAndReplace(
    { name: "张三" },
    { name: "张三", age: 30, replaced: true },
    { returnDocument: "after" }
)
```

经典应用场景：

```javascript
// 1. 实现原子自增序列号
function getNextSequence(name) {
    var result = db.counters.findOneAndUpdate(
        { _id: name },
        { $inc: { seq: 1 } },
        { returnDocument: "after", upsert: true }
    )
    return result.seq
}

// 2. 实现分布式锁
function acquireLock(lockName, owner, ttlSeconds) {
    var now = new Date()
    return db.locks.findOneAndUpdate(
        {
            _id: lockName,
            $or: [
                { locked: false },
                { expireAt: { $lt: now } }
            ]
        },
        {
            $set: {
                locked: true,
                owner: owner,
                expireAt: new Date(now.getTime() + ttlSeconds * 1000)
            }
        },
        { upsert: true, returnDocument: "after" }
    )
}
```

# 8. 批量操作bulkWrite

`bulkWrite`可以将多个操作打包成一组批量执行，减少网络往返，提高性能。

```javascript
db.users.bulkWrite([
    { insertOne: { document: { name: "用户A", age: 20 } } },
    { insertOne: { document: { name: "用户B", age: 25 } } },
    { updateOne: {
        filter: { name: "张三" },
        update: { $set: { age: 30 } }
    }},
    { updateMany: {
        filter: { age: { $lt: 18 } },
        update: { $set: { status: "minor" } }
    }},
    { replaceOne: {
        filter: { name: "李四" },
        replacement: { name: "李四", age: 26, replaced: true }
    }},
    { deleteOne: { filter: { name: "过期用户" } }},
    { deleteMany: { filter: { status: "deleted" } }}
], { ordered: false })
```

bulkWrite支持的操作类型：
- `insertOne`
- `updateOne`
- `updateMany`
- `replaceOne`
- `deleteOne`
- `deleteMany`

注意事项：
- 每个bulkWrite最多1000个操作（实际限制由BSON文档大小决定）
- ordered: false效率更高，最多并行8个批次
- 分批执行大量操作以防止内存溢出

```javascript
// 分批执行大量写操作的辅助函数
function batchBulkWrite(collection, operations, batchSize = 500) {
    for (let i = 0; i < operations.length; i += batchSize) {
        let batch = operations.slice(i, i + batchSize)
        collection.bulkWrite(batch, { ordered: false })
    }
}
```

# 9. 分布式锁实现详解

```java
// Java实现MongoDB分布式锁
@Service
public class MongoDistributedLock {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    /**
     * 获取锁
     */
    public boolean tryLock(String lockKey, String owner, long expireSeconds) {
        Date now = new Date();
        Query query = new Query(Criteria.where("_id").is(lockKey)
            .orOperator(
                Criteria.where("locked").is(false),
                Criteria.where("expireAt").lt(now)
            ));
        Update update = new Update()
            .set("locked", true)
            .set("owner", owner)
            .set("expireAt", new Date(now.getTime() + expireSeconds * 1000));
        FindAndModifyOptions options = new FindAndModifyOptions()
            .upsert(true).returnNew(true);
        
        LockDoc doc = mongoTemplate.findAndModify(
            query, update, options, LockDoc.class, "locks");
        return doc != null && owner.equals(doc.getOwner());
    }
    
    /**
     * 释放锁
     */
    public void unlock(String lockKey, String owner) {
        Query query = new Query(Criteria.where("_id").is(lockKey)
            .and("owner").is(owner));
        Update update = new Update()
            .set("locked", false)
            .unset("owner");
        mongoTemplate.updateFirst(query, update, "locks");
    }
}

@Data
@Document("locks")
class LockDoc {
    @Id
    private String id;
    private Boolean locked;
    private String owner;
    private Date expireAt;
}
```

# 10. 写入与读取关注实践

```javascript
// ========== 写关注配置 ==========

// 快速写入（不等待确认）
db.users.insertOne({ name: "test" }, { writeConcern: { w: 0 } })

// 等待主节点确认（默认）
db.users.insertOne({ name: "test" }, { writeConcern: { w: 1 } })

// 等待大多数节点确认（推荐的生产环境配置）
db.users.insertOne({ name: "test" }, { writeConcern: { w: "majority" } })

// 等待日志刷盘
db.users.insertOne({ name: "test" }, { writeConcern: { w: "majority", j: true } })

// 设置超时
db.users.insertOne({ name: "test" }, { writeConcern: { w: "majority", wtimeout: 5000 } })

// ========== 读关注配置 ==========

// 读取本地最新数据（默认）
db.users.find().readConcern("local")

// 读取大多数节点已确认的数据
db.users.find().readConcern("majority")

// 线性读取（最严格，单文档）
db.users.findOne({ _id: id }).readConcern("linearizable")
```

# 11. MongoDB SQL对照表

| SQL | MongoDB |
|-----|---------|
| INSERT INTO users VALUES(1, "张三") | db.users.insertOne({ _id: 1, name: "张三" }) |
| SELECT * FROM users | db.users.find() |
| SELECT name, age FROM users | db.users.find({}, { name: 1, age: 1 }) |
| SELECT * FROM users WHERE age > 25 | db.users.find({ age: { $gt: 25 } }) |
| SELECT * FROM users WHERE age > 25 AND status = 'active' | db.users.find({ age: { $gt: 25 }, status: "active" }) |
| SELECT * FROM users WHERE age > 25 ORDER BY age DESC | db.users.find({ age: { $gt: 25 } }).sort({ age: -1 }) |
| SELECT * FROM users LIMIT 10 OFFSET 20 | db.users.find().limit(10).skip(20) |
| SELECT COUNT(*) FROM users | db.users.countDocuments({}) |
| UPDATE users SET age = 30 WHERE name = '张三' | db.users.updateOne({ name: "张三" }, { $set: { age: 30 } }) |
| DELETE FROM users WHERE age < 18 | db.users.deleteMany({ age: { $lt: 18 } }) |
| SELECT DISTINCT status FROM users | db.users.distinct("status") |
