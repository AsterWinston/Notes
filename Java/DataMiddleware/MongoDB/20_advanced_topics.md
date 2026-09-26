
# 1. Change Streams（变更流）

变更流允许应用程序订阅集合、数据库或集群级别的实时数据变更，是实现事件驱动架构的核心组件。

## 1.1 Change Streams原理

Change Streams基于副本集的Oplog实现，从Oplog中读取变更事件并推送给订阅者。

```
应用 → watch() → Change Stream Cursor
                         ↓
                     读取Oplog
                    local.oplog.rs
```

## 1.2 Change Events结构

```javascript
{
    _id: {                           // Resume Token
        _data: "826501a..."
    },
    operationType: "insert",         // insert|update|delete|replace|drop|rename|...
    fullDocument: { ... },           // 完整文档（仅insert和replace）
    ns: { db: "mydb", coll: "users" },
    documentKey: { _id: ... },       // 变更文档的_id
    updateDescription: {             // 仅update
        updatedFields: { ... },
        removedFields: [...],
        truncatedArrays: [...]
    },
    clusterTime: Timestamp(...),
    wallTime: ISODate(...)
}
```

## 1.3 Resume Token

Resume Token是Change Stream的核心机制，允许在连接断开后从断点继续：

```java
// 记录Resume Token
BsonDocument resumeToken = changeStreamDocument.getResumeToken();

// 从断点恢复
MongoCursor<ChangeStreamDocument<Document>> cursor = 
    collection.watch()
        .resumeAfter(resumeToken)
        .iterator();

// 或从指定时间开始
cursor = collection.watch()
    .startAtOperationTime(BsonTimestamp.valueOf(1705651200))
    .iterator();

// 或从集群时间开始
cursor = collection.watch()
    .startAfter(resumeToken)
    .iterator();

// 侦听整个数据库
MongoCursor<ChangeStreamDocument<Document>> dbCursor = 
    database.watch().iterator();

// 侦听整个集群
MongoCursor<ChangeStreamDocument<Document>> clusterCursor = 
    mongoClient.watch().iterator();
```

## 1.4 事件过滤

```javascript
// 过滤特定操作类型
collection.watch(Arrays.asList(
    Aggregates.match(Filters.in("operationType",
        Arrays.asList("insert", "update", "replace")))
))

// 过滤特定字段变更
collection.watch(Arrays.asList(
    Aggregates.match(Filters.eq("updateDescription.updatedFields.status", "active"))
))

// 全文检索过滤
collection.watch(Arrays.asList(
    Aggregates.match(Filters.regex("fullDocument.name", "^张"))
))
```

## 1.5 Spring Boot集成

```java
@Component
public class UserChangeListener {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @EventListener(ApplicationReadyEvent.class)
    public void startWatching() {
        new Thread(() -> {
            MongoCollection<Document> collection = 
                mongoTemplate.getCollection("users");
            
            try (MongoCursor<ChangeStreamDocument<Document>> cursor = 
                    collection.watch().iterator()) {
                while (cursor.hasNext()) {
                    ChangeStreamDocument<Document> change = cursor.next();
                    handleChange(change);
                }
            } catch (Exception e) {
                // 处理异常，实现重连+恢复逻辑
                // 使用resumeToken从断点恢复
            }
        }).start();
    }
    
    private void handleChange(ChangeStreamDocument<Document> change) {
        switch (change.getOperationType()) {
            case INSERT:
                handleInsert(change.getFullDocument());
                break;
            case UPDATE:
                handleUpdate(change.getDocumentKey(), 
                    change.getUpdateDescription());
                break;
            case DELETE:
                handleDelete(change.getDocumentKey());
                break;
        }
    }
}
```

## 1.6 实用场景

- **CDC（Change Data Capture）**：实时同步MongoDB到数仓/搜索引擎
- **CQRS**：命令查询责任分离，变更事件驱动读模型更新
- **跨服务数据同步**：订单变更通知库存服务
- **审计日志**：记录所有数据变更
- **缓存失效**：数据变更时触发缓存刷新

```java
// 缓存失效示例
case UPDATE:
    String updatedId = change.getDocumentKey()
        .getObjectId("_id").getValue().toString();
    // 清除相关缓存
    redisTemplate.delete("user:" + updatedId);
    redisTemplate.delete("userList");
    break;
```

# 2. GridFS

GridFS是MongoDB的大文件存储规范，用于存储超过16MB BSON限制的文件。

## 2.1 GridFS架构

```
GridFS将大文件拆分为两个集合：
- fs.files: 文件元数据（文件名、大小、上传时间、自定义metadata等）
- fs.chunks: 文件数据块（每个chunk默认255KB，可配置）

fs.files文档:
{
    _id: ObjectId,
    filename: "report.pdf",
    length: 1234567,           // 文件大小（字节）
    chunkSize: 261120,         // 块大小
    uploadDate: ISODate,
    metadata: { type: "pdf" }  // 自定义元数据
}

fs.chunks文档:
{
    _id: ObjectId,
    files_id: ObjectId,        // 关联的files._id
    n: 0,                      // 块序号
    data: BinData(...)         // 二进制数据
}
```

## 2.2 GridFS Java操作

```java
// 上传
GridFSBucket bucket = GridFSBuckets.create(database, "fs");

GridFSUploadOptions options = new GridFSUploadOptions()
    .chunkSizeBytes(261120)
    .metadata(new Document("contentType", "application/pdf")
        .append("uploader", "system")
        .append("department", "finance"));

ObjectId fileId;
try (InputStream stream = new FileInputStream("/path/to/large_file.pdf")) {
    fileId = bucket.uploadFromStream("report.pdf", stream, options);
}

// 下载
try (OutputStream stream = new FileOutputStream("/path/to/output.pdf")) {
    bucket.downloadToStream(fileId, stream);
}

// 按文件名下载
bucket.downloadToStream("report.pdf", stream);

// 查找文件
GridFSFindIterable files = bucket.find(
    Filters.and(
        Filters.gt("uploadDate", oneWeekAgo),
        Filters.eq("metadata.department", "finance")
    ));
files.forEach(file -> {
    System.out.println(file.getFilename() + " - " + file.getLength());
});

// 删除文件
bucket.delete(fileId);

// 重命名
bucket.rename(fileId, "new_name.pdf");
```

## 2.3 GridFS使用场景vs建议

**适合GridFS**：
- 文件超过16MB
- 需要利用MongoDB的复制和分片特性
- 文件需要与关联数据原子操作（事务）
- 小文件数量非常大且需要快速访问

**不适合GridFS**：
- 静态文件建议使用对象存储（S3/MinIO）
- 频繁部分读取的大型文件
- 已有成熟的文件存储方案时

# 3. 时序集合（Time Series Collections）

MongoDB 5.0引入原生时序集合，针对时序数据进行了存储和查询的优化。

## 3.1 创建时序集合

```javascript
db.createCollection("sensorData", {
    timeseries: {
        timeField: "timestamp",
        metaField: "sensor",
        granularity: "seconds"    // seconds | minutes | hours
    },
    expireAfterSeconds: 2592000   // 30天自动过期
})
```

## 3.2 时序集合存储优化

```
时序集合内部实现：
- 按metaField分组
- 每组内的测量值按时间排序紧密存储
- 大幅减少索引大小和存储空间
- 自动创建_time_和_meta_索引

性能对比（vs 普通集合）：
- 存储空间减少约 50%~70%
- 写入吞吐提升约 30%~50%
- 聚合查询提升约 2~5倍
```

## 3.3 时序集合Java操作

```java
@Document("sensorData")
public class SensorReading {
    @Id
    private String id;
    private Date timestamp;      // timeField
    private SensorInfo sensor;   // metaField
    private double temperature;
    private double humidity;
}

// 批量写入时序数据
List<SensorReading> readings = generateReadings();
mongoTemplate.insert(readings, "sensorData");

// 查询特定传感器一段时间内的数据
Query query = new Query()
    .addCriteria(Criteria.where("sensor.sensorId").is("S001")
        .and("timestamp").gte(startTime).lte(endTime))
    .limit(1000);
List<SensorReading> data = mongoTemplate.find(query, SensorReading.class);

// 聚合：每小时平均温度
Aggregation agg = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("sensor.sensorId").is("S001")),
    Aggregation.group(
        DateOperators.DateToString.dateOf("timestamp").toString("%Y-%m-%d %H"),
        Accumulators.avg("avgTemp", "$temperature")
    ),
    Aggregation.sort(Sort.by("_id"))
);
```

# 4. 全文搜索进阶

## 4.1 Atlas Search

MongoDB Atlas Search是基于Apache Lucene的全文搜索服务（仅Atlas云服务），相比内置文本索引有显著提升：

| 特性 | 内置文本索引 | Atlas Search |
|------|------------|--------------|
| 分词 | 基础 | Apache Lucene |
| 中文分词 | 效果差 | 支持多种分析器 |
| 模糊搜索 | 不支持 | 支持fuzzy query |
| 自动补全 | 不支持 | 支持autocomplete |
| 同义词 | 不支持 | 支持 |
| 高亮 | 不支持 | 支持 |
| 权重 | 基础 | 灵活 |
| 性能 | 一般 | 高性能 |

## 4.2 Atlas Search聚合管道

```javascript
// $search 阶段
db.articles.aggregate([
    {
        $search: {
            index: "articleSearch",
            text: {
                query: "MongoDB 索引优化",
                path: ["title", "content"],
                fuzzy: { maxEdits: 1 }     // 允许1个编辑距离的模糊
            },
            highlight: {
                path: "content"
            }
        }
    },
    {
        $project: {
            title: 1,
            score: { $meta: "searchScore" },
            highlights: { $meta: "searchHighlights" }
        }
    },
    { $limit: 20 }
])
```

## 4.3 自建全文搜索方案

对于不使用Atlas的场景，可以采用Elasticsearch同步方案：

```
MongoDB (主存储) → Change Stream → Elasticsearch (搜索)
                                              ↑
                                        应用搜索请求
```

```java
@Component
public class MongoToESListener {
    
    @EventListener(ApplicationReadyEvent.class)
    public void syncChanges() {
        new Thread(() -> {
            MongoCursor<ChangeStreamDocument<Document>> cursor = 
                collection.watch().iterator();
            while (cursor.hasNext()) {
                ChangeStreamDocument<Document> change = cursor.next();
                switch (change.getOperationType()) {
                    case INSERT, REPLACE:
                        esClient.index(change.getFullDocument());
                        break;
                    case UPDATE:
                        esClient.update(change.getDocumentKey(), 
                            change.getUpdateDescription());
                        break;
                    case DELETE:
                        esClient.delete(change.getDocumentKey());
                        break;
                }
            }
        }).start();
    }
}
```

# 5. 多租户架构

## 5.1 多租户实现方式

MongoDB实现多租户（Multi-Tenancy）的三种方式：

**方式一：数据库级隔离**

```java
// 每个租户使用独立数据库
MongoDatabase tenantDb = mongoClient.getDatabase("tenant_" + tenantId);
// 优点：完全隔离，备份恢复独立
// 缺点：连接数随租户增长，管理复杂
```

**方式二：集合前缀隔离**

```java
// collection名称带租户前缀
MongoCollection<Document> orders = database
    .getCollection("tenant_" + tenantId + "_orders");
// 优点：单数据库，共享连接
// 缺点：大数据量时集合膨胀
```

**方式三：字段级隔离（推荐）**

```java
// 所有租户共用集合，通过tenantId字段区分
@Document("orders")
public class Order {
    @Id
    private String id;
    @Indexed
    private String tenantId;  // 租户标识
    // ... 其他字段
}

// 创建索引（tenantId放最前）
collection.createIndex(Indexes.compoundIndex(
    Indexes.ascending("tenantId"),
    Indexes.descending("createTime")
));

// 所有查询带tenantId
Query query = new Query(Criteria.where("tenantId").is(currentTenant));
// 如果分片，使用{ tenantId: 1, ... }作为分片键
```

## 5.2 多租户分片策略

```javascript
// 分片键设计
sh.shardCollection("mydb.orders", { tenantId: 1, createTime: 1 })
// tenantId确保同一租户数据在同一分片
// createTime保证时间范围查询效率

// Zone分片实现数据隔离
// 大租户数据分配到特定分片
sh.addShardToZone("shard1", "large_tenant_zone")
sh.updateZoneKeyRange(
    "mydb.orders",
    { tenantId: "LARGE_TENANT" },
    { tenantId: "LARGE_TENANT\uffff" },
    "large_tenant_zone"
)
```

# 6. 数据归档与生命周期管理

## 6.1 TTL索引自动化

```javascript
// 自动删除30天前的日志
db.logs.createIndex(
    { createTime: 1 },
    { expireAfterSeconds: 2592000 }
)
```

## 6.2 在线归档策略

```javascript
// 定期将冷数据迁移到归档集合
function archiveOldOrders() {
    var cutoff = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000)  // 90天前
    
    // 1. 复制到归档集合
    db.orders.aggregate([
        { $match: { createTime: { $lt: cutoff } } },
        { $merge: { into: "orders_archive" } }
    ])
    
    // 2. 删除原集合中的归档数据
    db.orders.deleteMany({ createTime: { $lt: cutoff } })
}
```

## 6.3 分片归档

```
- 按时间分片，新数据存入快速SSD分片
- 历史数据迁移到低成本HDD分片
- 通过Zone分片实现冷热数据物理分离

sh.addShardToZone("hot_shard", "hot")
sh.addShardToZone("cold_shard", "cold")
sh.updateZoneKeyRange(
    "mydb.data",
    { date: MinKey }, { date: ISODate("2024-01-01") },
    "cold"
)
sh.updateZoneKeyRange(
    "mydb.data",
    { date: ISODate("2024-01-01") }, { date: MaxKey },
    "hot"
)
```

# 7. $vectorSearch与AI集成

MongoDB 7.0+的Atlas Vector Search支持向量搜索，用于AI/ML场景：

```javascript
// 向量搜索（语义搜索、相似性搜索）
db.articles.aggregate([{
    $vectorSearch: {
        index: "vector_index",
        path: "embedding",             // 存储embedding的字段
        queryVector: [0.12, -0.34, ...], // 查询向量
        numCandidates: 100,
        limit: 10
    }
}])

// 配合Embedding模型实现语义搜索
// 1. 使用OpenAI/HuggingFace将文本转为embedding
// 2. 存储embedding到MongoDB
// 3. 使用$vectorSearch进行语义相似性搜索
// 4. 结合传统$match + $vectorSearch实现混合搜索
```

```java
// Java集成向量搜索
@Document("articles")
public class Article {
    @Id
    private String id;
    private String title;
    private String content;
    private List<Double> embedding;  // 768维向量
}
```

# 8. 生产级部署架构

## 8.1 高可用部署架构

```
应用层：
┌─────────┐ ┌─────────┐ ┌─────────┐
│  App 1  │ │  App 2  │ │  App 3  │
│ +mongos │ │ +mongos │ │ +mongos │
└────┬────┘ └────┬────┘ └────┬────┘
     └──────────┬┴───────────┘
                │
    ┌───────────┴───────────┐
    │   Config Server 副本集  │ (3节点，高可用)
    └───────────────────────┘
    ┌───────────────────────────────┐
    │  Shard 1 副本集 (3节点)        │
    │  Shard 2 副本集 (3节点)        │
    │  Shard 3 副本集 (3节点)        │
    └───────────────────────────────┘
```

## 8.2 连接字符串最佳配置

```
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/admin?
  replicaSet=rs0
  &readPreference=secondaryPreferred
  &w=majority
  &wtimeoutMS=5000
  &maxPoolSize=50
  &minPoolSize=5
  &connectTimeoutMS=5000
  &socketTimeoutMS=30000
  &serverSelectionTimeoutMS=30000
  &compressors=zstd,snappy
  &retryWrites=true
  &retryReads=true
```

## 8.3 Kubernetes部署要点

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  clusterIP: None             # Headless Service（副本集通信必需）
  selector:
    app: mongodb
  ports:
    - port: 27017
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0
          command:
            - mongod
            - "--replSet=rs0"
            - "--bind_ip_all"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
          resources:
            requests:
              memory: "4Gi"
              cpu: "2"
            limits:
              memory: "8Gi"
              cpu: "4"
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
```
