
# 1. 基础概念

## Q1: 什么是MongoDB？与传统关系型数据库有何区别？

MongoDB是一个面向文档的NoSQL数据库，使用类JSON的BSON格式存储数据。

核心区别：
- **数据模型**：文档 vs 表+行，MongoDB更灵活，无强制Schema
- **扩展方式**：MongoDB天然支持水平扩展（分片），RDBMS以垂直扩展为主
- **事务**：MongoDB 4.0+支持多文档ACID事务，但RDBMS的事务更成熟高效
- **JOIN**：MongoDB通过嵌入式文档和$lookup实现关联，RDBMS有成熟的JOIN机制
- **适用场景**：MongoDB适合非结构化/半结构化数据、快速迭代，RDBMS适合强一致事务场景

## Q2: 解释BSON是什么？与JSON的区别？

BSON（Binary JSON）是二进制序列化格式：
- JSON是文本格式，BSON是二进制格式 → 解析更快
- BSON支持更多数据类型（Date、ObjectId、Binary等）
- BSON有空间开销（字段名重复），但有更好的遍历性
- BSON设计目标是轻量、可遍历、高效编解码

## Q3: MongoDB中的集合（Collection）和文档（Document）是什么？

- **文档（Document）**：MongoDB的基本数据单元，相当于RDBMS中的一行
- **集合（Collection）**：文档的分组，相当于RDBMS中的表
- 集合中的文档可以有不同结构（Schema-free）
- 集合在插入第一个文档时隐式创建

## Q4: ObjectId的结构是什么？为什么用它作为默认主键？

ObjectId是12字节的BSON类型：

| 部分 | 字节数 | 说明 |
|------|--------|------|
| Timestamp | 4 | Unix时间戳，天然按时间排序 |
| Random | 5 | 机器标识+进程ID，全局唯一 |
| Counter | 3 | 自增计数器 |

优点：客户端生成无需协调、天然时间排序、分片友好、全局唯一

# 2. 索引相关

## Q5: MongoDB支持哪些索引类型？

- 单字段索引、复合索引、多键索引（数组）
- 文本索引、地理空间索引（2dsphere/2d）
- 哈希索引（分片）、TTL索引（过期自动删除）
- 唯一索引、稀疏索引、部分索引、通配符索引
- 隐藏索引（4.4+）

## Q6: 什么是ESR规则？

设计复合索引的黄金法则：
1. **Equality**（等值查询字段）放在最前面
2. **Sort**（排序字段）放在中间
3. **Range**（范围查询字段）放在最后

```
查询: find({status:"active", age:{$gt:25}}).sort({createTime:-1})
最佳索引: {status:1, createTime:-1, age:1}
           ↑EQ      ↑Sort          ↑Range
```

## Q7: explain()输出的COLLSCAN和IXSCAN的含义？

- **COLLSCAN**：全集合扫描，未使用索引或索引效率极低，需要优化
- **IXSCAN**：索引扫描，使用了索引
- **FETCH**：通过索引获取完整文档
- **PROJECTION_COVERED**：覆盖查询（最优），全部从索引获取

## Q8: 覆盖查询是什么？有什么优势？

覆盖查询（Covered Query）是指查询所需字段全部在索引中，无需读取实际文档。

```javascript
// 索引: {name: 1, age: 1}
// 覆盖查询：
db.users.find({name: "张三"}, {name: 1, age: 1, _id: 0})
// stage: PROJECTION_COVERED — 不需要FETCH
```

优势：零磁盘文档读取，速度极快，内存友好

# 3. 聚合框架

## Q9: MongoDB聚合管道（Aggregation Pipeline）是什么？

聚合管道是一组数据处理阶段（Stage），文档流经管道逐步处理：

```
输入文档 → $match → $group → $sort → $project → 输出
```

常用阶段：$match, $group, $sort, $project, $unwind, $lookup, $limit, $skip, $facet, $bucket, $addFields, $replaceRoot, $out, $merge

## Q10: $lookup与SQL JOIN的区别？

`$lookup`是MongoDB的左外连接（LEFT OUTER JOIN）：

```javascript
db.orders.aggregate([{
    $lookup: {
        from: "users",
        localField: "userId",
        foreignField: "_id",
        as: "user"
    }
}])
```

区别：
- $lookup是左外连接，SQL支持更多连接类型
- $lookup返回数组，需要$unwind展开
- $lookup在分片集群中有限制
- 推荐使用嵌入式文档避免$lookup

## Q11: $facet的用途？

`$facet`在同一组文档上并行执行多个聚合管道，适合多维度分析报表：

```javascript
{ $facet: {
    "byCategory": [{ $group: { _id: "$category", count: { $sum: 1 } } }],
    "priceStats": [{ $group: { _id: null, avg: { $avg: "$price" } } }]
}}
```

# 4. 副本集与高可用

## Q12: 副本集（Replica Set）的工作原理？

副本集是维护相同数据集的多个MongoDB进程：
- 1个Primary：接收所有写操作
- 多个Secondary：复制Primary的oplog
- 可选Arbiter：仅投票不存数据
- 通过Heartbeat维持心跳（2秒一次）
- 主节点宕机10秒内自动选举新主

## Q13: Oplog是什么？有什么作用？

Oplog（Operations Log）是一个固定集合（Capped Collection），记录所有修改数据的操作：

- 存储在local.oplog.rs中
- 记录insert/update/delete和DDL操作
- 从节点通过读取和应用oplog与主节点同步
- Oplog窗口 = 最早~最新oplog的时间跨度
- 如果从节点离线超过oplog窗口，需要全量同步

## Q14: 选举是什么？什么条件下触发？

基于Raft协议的选举机制：
- **触发条件**：主节点10秒无心跳、手动stepDown、主节点降级
- **投票条件**：节点priority>0、能与大多数节点通信、数据足够新
- **成功条件**：获得N/2+1票（N=投票节点数）
- 使用奇数个投票节点防止脑裂

## Q15: Read Preference有哪些类型？

| 类型 | 说明 |
|------|------|
| primary | 只读主节点（默认） |
| primaryPreferred | 优先主，可读从 |
| secondary | 只读从节点 |
| secondaryPreferred | 优先从，可读主 |
| nearest | 读延迟最低的节点 |

# 5. 分片

## Q16: 分片（Sharding）是什么？什么时候需要？

分片是水平拆分数据到多个服务器：

**需要分片的信号**：
- 磁盘空间不足（数据量>单机容量）
- 内存不足（热数据>单机内存）
- 写入吞吐量瓶颈
- 需要数据局部性存储

**架构**：Config Server（元数据）+ Mongos（路由）+ Shard（数据节点）

## Q17: 什么是好的分片键？

好分片键的标准：
1. **高基数（High Cardinality）**：值足够多（如userId）
2. **低频率（Low Frequency）**：每个值对应文档数均匀
3. **非单调递增**：避免写入热点
4. **匹配查询模式**：大多数查询能定向到特定分片

```javascript
// ✅ 好的分片键
{ userId: "hashed" }               // 均匀分布
{ companyId: 1, createTime: 1 }    // 定向查询+范围

// ❌ 差的分片键
{ status: 1 }                      // 基数低
{ isDeleted: 1 }                   // 基数极低
{ createTime: 1 }                  // 写入热点
```

## Q18: 范围分片与哈希分片的区别？

| 特性 | 范围分片 | 哈希分片 |
|------|---------|---------|
| 分布 | 按分片键值范围 | 按hash值均匀分布 |
| 范围查询 | 高效（定向） | 低效（广播） |
| 等值查询 | 高效 | 高效（定向） |
| 写入热点 | 可能（单调递增键） | 低（均匀分布） |
| 唯一索引 | 可跨非分片键 | 仅分片键 |

# 6. 事务

## Q19: MongoDB 4.0后支持ACID事务？它和RDBMS的事务有何不同？

MongoDB 4.0+支持多文档事务，4.2+支持分布式事务：

**相同点**：
- 都支持ACID特性
- 都有提交/回滚机制
- 都有事务隔离级别

**不同点**：
- MongoDB使用快照隔离（Snapshot），RDBMS有多种隔离级别
- MongoDB事务有60秒超时限制
- MongoDB事务性能不如单文档原子操作
- RDBMS的事务更成熟，适合高频、复杂事务场景

## Q20: 事务中的写冲突如何处理？

MongoDB快照隔离下，两个并发事务修改同一文档时：
- 先提交的事务成功
- 后提交的事务检测到写冲突（WriteConflict, 错误码112）
- 应用层需要捕获并重试

```java
@Retryable(value = MongoCommandException.class, maxAttempts = 3)
@Transactional
public void transfer(String from, String to, long amount) {
    // ...
}
```

# 7. 存储引擎

## Q21: WiredTiger存储引擎有哪些特点？

- **文档级并发控制（MVCC）**：写不阻塞读
- **压缩**：支持snappy/zlib/zstd，通常压缩到20%-30%
- **独立缓存**：可控的WiredTiger Cache，不依赖OS文件缓存
- **Checkpoint + Journal**：混合持久化策略
- **文档级锁** vs MMAPv1的集合级锁

## Q22: WiredTiger的Checkpoint和Journal是什么？

- **Checkpoint（检查点）**：每60秒将内存修改刷到磁盘，创建数据快照
- **Journal（预写日志 WAL）**：所有写操作先写到Journal，最多丢失100ms数据
- 恢复流程：最近Checkpoint + 重放Journal = 完整数据

# 8. 设计与优化

## Q23: 嵌入式文档 vs 引用式文档，如何选择？

| 选择嵌入 | 选择引用 |
|---------|---------|
| 数据总是一起访问 | 数据独立访问 |
| 一对极少（<几百） | 一对多/多对多 |
| 子数据无独立意义 | 子数据有独立生命周期 |
| 需要单文档原子操作 | 数据会无限增长 |

## Q24: 如何优化慢查询？

1. 使用explain()分析执行计划
2. 创建合适的索引（遵循ESR规则）
3. 使用覆盖查询（避免FETCH）
4. 优化分页（游标分页代替skip）
5. 减少返回字段（Projection）
6. 使用$match在聚合管道前过滤
7. 检查并清除计划缓存

## Q25: MongoDB如何做分页？

```javascript
// ❌ 大offset性能差
db.collection.find().skip(100000).limit(20)

// ✅ 游标分页
db.collection.find({ _id: { $lt: lastSeenId } })
    .sort({ _id: -1 })
    .limit(20)

// 非唯一排序字段需要tie-breaker
db.collection.find({
    $or: [
        { createTime: { $lt: lastTime } },
        { createTime: lastTime, _id: { $lt: lastId } }
    ]
}).sort({ createTime: -1, _id: -1 }).limit(20)
```

# 9. 安全管理

## Q26: MongoDB有哪些安全机制？

- **认证**：SCRAM-SHA-256、X.509证书、LDAP、Kerberos
- **授权**：RBAC基于角色的访问控制
- **传输加密**：TLS/SSL
- **审计**：操作审计日志（企业版）
- **字段级加密**：客户端字段级加密（4.2+ FLE）
- **网络隔离**：bindIp限制、防火墙规则

## Q27: 副本集内部节点如何安全通信？

使用**KeyFile密钥文件**或**X.509证书**进行节点间内部认证：

```bash
openssl rand -base64 756 > keyfile
chmod 600 keyfile
```

配置中指定：`security.keyFile: /path/to/keyfile`

# 10. 场景设计题

## Q28: 设计一个博客系统的MongoDB数据模型

```javascript
// 用户集合
users: {
    _id, username, email, avatar, bio, followerCount, followingCount,
    createdAt, updatedAt
}

// 文章集合（嵌入作者信息、标签、评论元数据）
posts: {
    _id, title, content, summary,
    author: { id, username, avatar },        // 嵌入式（很少变）
    tags: ["Java", "MongoDB"],
    category: "技术",
    viewCount, likeCount, commentCount,       // 预计算字段
    createdAt, updatedAt
}

// 评论集合（引用式，评论数量可能很大）
comments: {
    _id, postId, userId, content,
    parentId,                  // 支持嵌套回复
    likeCount,
    createdAt
}
```

## Q29: 设计一个电商订单系统，如何保证下单扣库存的原子性？

方案一（推荐）：使用事务

```java
@Transactional
public void createOrder(Order order) {
    mongoTemplate.insert(order);
    
    // 减库存（带版本号检查，乐观锁）
    Query query = new Query()
        .addCriteria(Criteria.where("_id").is(productId)
            .and("stock").gte(order.getQty())
            .and("version").is(currentVersion));
    Update update = new Update()
        .inc("stock", -order.getQty())
        .inc("version", 1);
    
    UpdateResult result = mongoTemplate.updateFirst(query, update, Product.class);
    if (result.getMatchedCount() == 0) {
        throw new RuntimeException("库存不足或版本冲突");
    }
}
```

方案二：嵌入式文档（订单和库存在同一文档中，单文档原子性）

## Q30: 亿级数据的用户表如何设计分片策略？

```javascript
// 1. 分片键选择
sh.shardCollection("mydb.users", { userId: "hashed" })
// 理由：userId基数高、等值查询为主、哈希确保均匀分布

// 2. 预分片（避免初始阶段数据倾斜）
// 提前创建空chunk分到各分片

// 3. 查询模式设计
// 所有查询必须带userId（定向查询）
db.users.find({ userId: "U001" })  // ✅ 定向
db.users.find({ name: "张三" })    // ❌ 广播

// 4. 如果必须按name查询
// 创建单独的搜索服务或使用Elasticsearch
```

## Q31: 副本集主节点宕机后的完整处理流程？

1. 心跳超时检测（10秒内）
2. 从节点发起选举，获得多数票的节点成为新主
3. 驱动自动检测新主并重连（serverSelectionTimeoutMS内）
4. 旧主恢复后以Secondary身份加入
5. 如果旧主数据落后超过oplog窗口 → Initial Sync
6. 如果旧主数据领先（有未复制的事务） → Rollback

应用层需要：
- 使用replicaSet连接字符串，驱动自动故障转移
- 设置合适的serverSelectionTimeoutMS（默认30秒）
- 处理TransientTransactionError重试
- 使用w: "majority"写关注防止回滚

## Q32: Spring Data MongoDB与传统MongoDB Driver如何选？

- **Spring Data MongoDB**：适合Spring项目，自动配置、Repository、声明式事务
- **MongoDB Driver**：更底层的控制、非Spring项目、更高的灵活性
- 实际项目中Spring Data MongoDB是主流选择，配合MongoTemplate应对复杂查询
