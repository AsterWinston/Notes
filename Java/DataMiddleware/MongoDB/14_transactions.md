
# 1. MongoDB事务概述

MongoDB 4.0引入了多文档事务（Multi-Document Transactions），4.2进一步支持分布式事务（跨分片事务）。事务是MongoDB从NoSQL向全能数据库演进的重要里程碑。

## 1.1 事务的核心特性

| 特性 | 说明 |
|------|------|
| ACID兼容 | 完整支持原子性、一致性、隔离性、持久性 |
| 多文档 | 跨越多个文档、多个集合的操作 |
| 快照隔离 | 事务看到一致的数据快照 |
| 分布式事务 | 跨分片的事务支持（4.2+） |
| 可重试提交 | 网络错误时可自动重试 |
| 时间限制 | 默认60秒事务超时 |

## 1.2 事务的ACID保证

| 属性 | MongoDB实现 |
|------|------------|
| Atomicity（原子性） | 所有操作要么全部提交，要么全部回滚 |
| Consistency（一致性） | Schema Validation + JSON Schema校验 |
| Isolation（隔离性） | Snapshot隔离级别（读快照，写冲突检测） |
| Durability（持久性） | Journal + WriteConcern保证 |

# 2. 单文档原子性的局限

MongoDB天然支持单文档原子性，但在事务支持之前，跨文档操作存在以下问题：

```javascript
// 场景：转账操作
// 从账户A扣款，向账户B加款

// 无事务时的问题：
db.accounts.updateOne({ _id: "A" }, { $inc: { balance: -100 } })
// ↑ 如果这里系统崩溃，A已扣款但B未到账！
db.accounts.updateOne({ _id: "B" }, { $inc: { balance: 100 } })

// 解决方案：
// 1. 4.0之前：two-phase commit模式（应用层实现，复杂且不可靠）
// 2. 4.0之后：使用多文档事务
```

# 3. 事务API

## 3.1 Shell事务

```javascript
// 启动会话
var session = db.getMongo().startSession()

// 开始事务
session.startTransaction()

// 获取事务上下文中的集合
var accounts = session.getDatabase("mydb").accounts

try {
    // 扣款
    accounts.updateOne({ _id: "A" }, { $inc: { balance: -100 } })
    
    // 加款
    accounts.updateOne({ _id: "B" }, { $inc: { balance: 100 } })
    
    // 提交
    session.commitTransaction()
} catch (e) {
    // 回滚
    session.abortTransaction()
    throw e
} finally {
    session.endSession()
}
```

## 3.2 事务选项

```javascript
// 自定义事务选项
var txnOptions = {
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" },
    maxCommitTimeMS: 30000          // 最大提交时间
}

session.startTransaction(txnOptions)

// 事务内读取
accounts.find(
    { _id: "A" },
    { readConcern: { level: "snapshot" } }  // 快照读
).toArray()
```

## 3.3 withTransaction简化写法

```javascript
// 自动处理提交/回滚
session.withTransaction(function() {
    var accounts = session.getDatabase("mydb").accounts
    accounts.updateOne({ _id: "A" }, { $inc: { balance: -100 } })
    accounts.updateOne({ _id: "B" }, { $inc: { balance: 100 } })
    return "转账成功"
}, {
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
})
```

# 4. Java Driver事务

## 4.1 基础事务

```java
MongoClient mongoClient = MongoClients.create(uri);

try (ClientSession session = mongoClient.startSession()) {
    // 方式一：手动控制
    session.startTransaction();
    try {
        MongoCollection<Document> accounts = 
            mongoClient.getDatabase("mydb").getCollection("accounts");
        
        accounts.updateOne(session, 
            Filters.eq("_id", "A"), 
            Updates.inc("balance", -100));
        
        accounts.updateOne(session, 
            Filters.eq("_id", "B"), 
            Updates.inc("balance", 100));
        
        session.commitTransaction();
    } catch (Exception e) {
        session.abortTransaction();
        throw e;
    }
}
```

## 4.2 withTransaction自动管理

```java
// 简化事务处理（自动重试和回滚）
mongoClient.withSession(session -> {
    session.withTransaction(txn -> {
        MongoCollection<Document> accounts = 
            mongoClient.getDatabase("mydb").getCollection("accounts");
        
        accounts.updateOne(session, 
            Filters.eq("_id", "A"), 
            Updates.inc("balance", -100));
        
        accounts.updateOne(session, 
            Filters.eq("_id", "B"), 
            Updates.inc("balance", 100));
        
        return "转账成功";
    }, TransactionOptions.builder()
        .readConcern(ReadConcern.SNAPSHOT)
        .writeConcern(WriteConcern.MAJORITY)
        .build());
    return null;
});
```

## 4.3 事务选项详解

```java
ClientSessionOptions sessionOptions = ClientSessionOptions.builder()
    .causallyConsistent(true)
    .defaultTransactionOptions(TransactionOptions.builder()
        .readConcern(ReadConcern.SNAPSHOT)      // 快照读
        .writeConcern(WriteConcern.MAJORITY)    // 多数确认
        .readPreference(ReadPreference.primary()) // 事务读必须在主节点
        .maxCommitTime(30, TimeUnit.SECONDS)    // 最大提交等待时间
        .build())
    .build();

try (ClientSession session = mongoClient.startSession(sessionOptions)) {
    session.startTransaction();
    // ...
    session.commitTransaction();
}
```

## 4.4 事务中的操作注意事项

```java
// 事务中创建集合和索引会自动在事务外执行（不会回滚）

// 事务中不能进行以下操作：
// - 创建/删除集合（除固定集合外）
// - 创建/删除索引
// - 非CRUD操作（listCollections, listIndexes等）
// - 对config, admin, local数据库的写入操作
// - 在transaction上下文外操作（传入session获得事务上下文）
```

# 5. Spring Data MongoDB事务

## 5.1 配置事务管理器

```java
@Configuration
public class MongoTransactionConfig {
    
    @Bean
    public MongoTransactionManager transactionManager(
            MongoDatabaseFactory dbFactory) {
        return new MongoTransactionManager(dbFactory);
    }
}
```

## 5.2 声明式事务

```java
@Service
public class TransferService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Transactional
    public void transfer(String fromId, String toId, long amount) {
        Query fromQuery = new Query(Criteria.where("_id").is(fromId)
            .and("balance").gte(amount));
        Update fromUpdate = new Update().inc("balance", -amount);
        
        UpdateResult result = mongoTemplate.updateFirst(
            fromQuery, fromUpdate, Account.class);
        
        if (result.getMatchedCount() == 0) {
            throw new InsufficientBalanceException("余额不足");
        }
        
        Query toQuery = new Query(Criteria.where("_id").is(toId));
        Update toUpdate = new Update().inc("balance", amount);
        mongoTemplate.updateFirst(toQuery, toUpdate, Account.class);
    }
}
```

## 5.3 编程式事务

```java
@Service
public class OrderService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createOrder(Order order) {
        transactionTemplate.execute(status -> {
            try {
                mongoTemplate.insert(order);
                
                // 减库存
                Query query = new Query(Criteria.where("_id")
                    .is(order.getProductId()).and("stock").gte(order.getQty()));
                Update update = new Update().inc("stock", -order.getQty());
                
                UpdateResult result = mongoTemplate.updateFirst(
                    query, update, Product.class);
                
                if (result.getMatchedCount() == 0) {
                    throw new RuntimeException("库存不足");
                }
                
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

## 5.4 @Transactional属性设置

```java
@Service
public class OrderService {
    
    @Transactional(
        transactionManager = "mongoTransactionManager",
        propagation = Propagation.REQUIRED,
        isolation = Isolation.SERIALIZABLE,  // MongoDB中实际效果有限
        timeout = 30,                         // 超时30秒
        rollbackFor = { Exception.class },
        noRollbackFor = { IllegalArgumentException.class }
    )
    public void checkout(Order order) {
        // ...
    }
}
```

# 6. 事务隔离级别

MongoDB目前仅支持**快照隔离（Snapshot Isolation）**级别，实现原理：

```
快照隔离的工作原理：
1. 事务开始时获取一个快照时间戳
2. 事务内所有读操作都看到该时间戳的数据版本
3. 写操作检测写冲突（Write Conflict）
   - 如果两个并发事务修改同一文档，后提交的事务会被回滚
4. MVCC提供非阻塞读
```

隔离级别对比：

| 隔离级别 | SQL标准 | MongoDB |
|---------|---------|---------|
| Read Uncommitted | 脏读可能 | 不支持 |
| Read Committed | 不可重复读可能 | 不支持 |
| Repeatable Read | 幻读可能 | 接近（快照隔离） |
| Serializable | 完全串行化 | 不支持直接串行化 |

## 6.1 写冲突处理

```javascript
// 两个并发事务修改同一个文档
// 事务A和事务B同时读取账户X（余额=1000）

// 事务A:
sessionA.startTransaction()
accounts.updateOne({ _id: "X" }, { $inc: { balance: -200 } })
// 扣款200，balance=800

// 事务B（并发执行）:
sessionB.startTransaction()
accounts.updateOne({ _id: "X" }, { $inc: { balance: -300 } })
// 尝试扣款300，但检测到写冲突

sessionA.commitTransaction()  // 先提交成功 → balance=800

sessionB.commitTransaction()  // 抛出写冲突异常！→ 回滚
// WriteConflict error: error encountered during commit
```

处理写冲突：

```java
@Override
@Retryable(
    value = {MongoCommandException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional
public void transfer(String from, String to, long amount) {
    // 自动重试写冲突
}

// 或在应用层手动重试
public void transferWithRetry(String from, String to, long amount) {
    int maxRetries = 3;
    for (int i = 0; i < maxRetries; i++) {
        try {
            transferInternal(from, to, amount);
            return;
        } catch (MongoCommandException e) {
            if (e.getCode() == 112 && i < maxRetries - 1) { // WriteConflict
                Thread.sleep((long) (Math.random() * 100 * (i + 1)));
                continue;
            }
            throw e;
        }
    }
}
```

# 7. 分布式事务（跨分片）

MongoDB 4.2支持跨分片的分布式事务，实现基于**两阶段提交（2PC）**：

```
分布式事务流程：
1. Prepare阶段：协调者通知所有参与者准备提交
2. Commit阶段：协调者通知所有参与者提交

┌──────────────┐
│   Mongos     │ ← 事务协调者
│  (Coordinator)│
└─┬─────────┬──┘
  │         │
  ▼         ▼
┌────┐   ┌────┐
│分片1│   │分片2│  ← 事务参与者
└────┘   └────┘
```

分布式事务限制：
- 跨分片事务性能开销明显大于单分片事务
- 默认事务超时60秒
- 建议不要在跨分片事务中做大量操作

# 8. 事务最佳实践

## 8.1 何时使用事务

**适合使用事务**：
- 跨文档/跨集合的原子性要求（如转账、下单扣库存）
- 数据一致性要求高，无法容忍部分成功
- 短小精悍的操作

**不适合使用事务**：
- 单个文档的操作（MongoDB天然单文档原子性）
- 批量操作追求高吞吐量（事务会降低写入性能）
- 长时间运行的操作（事务有60秒超时）
- 可以用嵌入式文档解决的场景

## 8.2 事务设计建议

1. **缩短事务时间**：事务中的操作尽量少、尽量快
2. **避免在事务中执行网络调用**：如调用外部API
3. **使用正确的读写关注**：事务需要`w: "majority"`和`r: "snapshot"`
4. **处理写冲突**：实现重试逻辑
5. **监控事务指标**：

```javascript
// 查看事务统计
db.serverStatus().transactions

// 查看当前活跃事务
db.currentOp({ "transaction": { $exists: true } })

// 终止超时事务
db.adminCommand({
    killSessions: [
        { id: UUID("...") }
    ]
})
```

## 8.3 事务与性能

```
事务性能影响:
- 单分片事务: 吞吐量降低约 10%~20%
- 跨分片事务: 吞吐量降低约 20%~40%
- 事务中大量写操作: 性能可能显著下降

建议:
- 事务中操作的文档数 < 1000
- 事务持续时间 < 5秒
- 避免在事务中做全表扫描
```

## 8.4 容错与重试模式

```java
@Component
public class ResilientTransactionService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    /**
     * 带自动重试的事务执行
     */
    public <T> T executeInTransaction(TransactionCallback<T> callback) {
        int maxRetries = 3;
        int retryCount = 0;
        
        while (true) {
            try {
                return mongoTemplate.executeInTransaction(
                    session -> callback.doInTransaction(mongoTemplate, session));
            } catch (MongoCommandException e) {
                if (isTransientError(e) && retryCount < maxRetries) {
                    retryCount++;
                    sleepBackoff(retryCount);
                    continue;
                }
                throw e;
            }
        }
    }
    
    private boolean isTransientError(MongoCommandException e) {
        // TransientTransactionError, WriteConflict, Network errors
        return e.hasErrorLabel(
            org.bson.BsonString.valueOf("TransientTransactionError"))
            || e.getCode() == 112;  // WriteConflict
    }
    
    private void sleepBackoff(int retry) {
        try {
            // 指数退避
            Thread.sleep((long) (100 * Math.pow(2, retry - 1)));
        } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
        }
    }
}

@FunctionalInterface
interface TransactionCallback<T> {
    T doInTransaction(MongoTemplate template, ClientSession session);
}
```

## 8.5 事务checklist

- [ ] 确认事务中操作的所有集合都已存在（事务中不能创建集合）
- [ ] 所有写入操作都使用`session`参数
- [ ] 使用合适的写关注和读关注
- [ ] 处理`TransientTransactionError`（可重试）和`UnknownTransactionCommitResult`
- [ ] 设置合理的超时时间
- [ ] 监控事务的持续时间和数量
- [ ] 在分片集群中测试事务行为
