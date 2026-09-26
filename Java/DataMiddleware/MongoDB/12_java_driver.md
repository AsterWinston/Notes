
# 1. MongoDB Java Driver概述

MongoDB官方提供Java驱动，主要包含：
- **mongodb-driver-sync**（同步驱动）：Java 8+，基于Callback的异步底层
- **mongodb-driver-reactivestreams**（响应式驱动）：支持Reactive Streams规范
- **mongodb-driver-kotlin-coroutine**：Kotlin协程支持
- **mongodb-driver-scala**：Scala驱动

## 1.1 Maven依赖

```xml
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>5.2.0</version>
</dependency>

<!-- 或使用BOM管理版本 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongodb-driver-bom</artifactId>
            <version>5.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

# 2. MongoClient创建与配置

## 2.1 基本连接

```java
// 最简单的连接方式
MongoClient mongoClient = MongoClients.create("mongodb://localhost:27017");

// 通过连接字符串配置
MongoClient mongoClient = MongoClients.create(
    "mongodb://user:password@host1:27017,host2:27017/?replicaSet=rs0&w=majority");
```

## 2.2 MongoClientSettings详细配置

```java
import com.mongodb.*;
import com.mongodb.connection.*;

MongoClientSettings settings = MongoClientSettings.builder()
    // ===== 集群设置 =====
    .applyToClusterSettings(builder -> builder
        .hosts(Arrays.asList(
            new ServerAddress("mongo1", 27017),
            new ServerAddress("mongo2", 27017)
        ))
        .requiredReplicaSetName("rs0")      // 副本集名称
        .serverSelectionTimeout(30, TimeUnit.SECONDS)
    )
    
    // ===== 连接池设置 =====
    .applyToConnectionPoolSettings(builder -> builder
        .maxSize(100)
        .minSize(10)
        .maxWaitTime(2, TimeUnit.SECONDS)
        .maxConnectionIdleTime(10, TimeUnit.MINUTES)
        .maxConnectionLifeTime(30, TimeUnit.MINUTES)
    )
    
    // ===== Socket设置 =====
    .applyToSocketSettings(builder -> builder
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
    )
    
    // ===== 认证 =====
    .credential(MongoCredential.createScramSha256Credential(
        "username", "admin", "password".toCharArray()))
    
    // ===== 读写偏好 =====
    .readPreference(ReadPreference.secondaryPreferred())
    .readConcern(ReadConcern.MAJORITY)
    .writeConcern(WriteConcern.MAJORITY)
    
    // ===== 压缩 =====
    .compressorList(Arrays.asList(
        MongoCompressor.createZstdCompressor(),
        MongoCompressor.createSnappyCompressor(),
        MongoCompressor.createZlibCompressor()
    ))
    
    // ===== UUID表示 =====
    .uuidRepresentation(UuidRepresentation.STANDARD)
    
    .build();

MongoClient mongoClient = MongoClients.create(settings);
```

## 2.3 连接字符串参数速查

```
mongodb://[user:pass@]host1[:port1][,host2[:port2]]/db[?options]

常用参数：
replicaSet=rs0                副本集名称
w=majority                    写关注（majority/1/2/number）
wtimeoutMS=5000               写关注超时
readPreference=secondary       读偏好
readConcernLevel=majority      读关注
maxPoolSize=100                最大连接数
minPoolSize=10                 最小连接数
maxIdleTimeMS=600000           空闲超时
connectTimeoutMS=10000         连接超时
socketTimeoutMS=0              Socket超时(0=无限)
serverSelectionTimeoutMS=30000 服务器选择超时
compressors=zstd,snappy       压缩算法
authSource=admin               认证数据库
authMechanism=SCRAM-SHA-256    认证机制
tls=true                       启用TLS
retryWrites=true               可重试写入(默认true)
retryReads=true                可重试读取(默认true)
zlibCompressionLevel=6         zlib压缩级别
appName=myapp                  应用名称(诊断用)
```

# 3. 基本CRUD操作

## 3.1 获取数据库和集合

```java
MongoDatabase database = mongoClient.getDatabase("mydb");
MongoCollection<Document> collection = database.getCollection("users");

// 使用泛型集合（配合POJO）
MongoCollection<User> userCollection = database.getCollection("users", User.class);

// 创建集合
database.createCollection("newCollection");
```

## 3.2 插入操作

```java
// 插入单个文档
Document doc = new Document("name", "张三")
    .append("age", 28)
    .append("email", "zhangsan@example.com")
    .append("tags", Arrays.asList("Java", "MongoDB"))
    .append("createTime", new Date());

collection.insertOne(doc);
// 返回的ObjectId: doc.getObjectId("_id")

// 插入多个文档
List<Document> docs = Arrays.asList(
    new Document("name", "李四").append("age", 25),
    new Document("name", "王五").append("age", 30)
);
collection.insertMany(docs);

// 插入选项
collection.insertOne(doc, new InsertOneOptions().bypassDocumentValidation(true));
```

## 3.3 查询操作

```java
// 查找所有
FindIterable<Document> cursor = collection.find();

// 条件查询
collection.find(Filters.eq("name", "张三"));
collection.find(Filters.and(
    Filters.eq("status", "active"),
    Filters.gt("age", 25)
));

// 查询单个
Document doc = collection.find(Filters.eq("email", "zhangsan@example.com")).first();

// 遍历结果
collection.find().forEach(doc -> {
    System.out.println(doc.getString("name"));
});

// 返回列表
List<Document> results = collection.find().into(new ArrayList<>());
```

## 3.4 查询选项

```java
collection.find()
    .filter(Filters.gt("age", 20))
    .projection(Projections.include("name", "age"))
    .projection(Projections.excludeId())
    .sort(Sorts.descending("age"))
    .limit(10)
    .skip(20)
    .batchSize(50)
    .comment("按年龄查询")            // 查询注释(诊断用)
    .maxAwaitTime(5, TimeUnit.SECONDS) // change stream
    .noCursorTimeout(true)            // 不自动关闭游标
    .cursorType(CursorType.TailableAwait); // 固定集合tail
```

## 3.5 更新操作

```java
// 更新单个
collection.updateOne(
    Filters.eq("name", "张三"),
    Updates.combine(
        Updates.set("age", 29),
        Updates.currentDate("lastModified")
    )
);

// 更新多个
collection.updateMany(
    Filters.lt("age", 18),
    Updates.set("status", "minor")
);

// 更新不存在则插入
collection.updateOne(
    Filters.eq("name", "新用户"),
    Updates.set("age", 20),
    new UpdateOptions().upsert(true)
);

// 数组操作
collection.updateOne(
    Filters.eq("_id", userId),
    Updates.push("tags", "NewTag")
);

collection.updateOne(
    Filters.eq("_id", userId),
    Updates.addToSet("tags", "UniqueTag")
);

collection.updateOne(
    Filters.eq("_id", userId),
    Updates.pull("tags", "OldTag")
);

// arrayFilters
collection.updateOne(
    Filters.eq("_id", userId),
    Updates.inc("scores.$[elem].score", 5),
    new UpdateOptions().arrayFilters(
        Arrays.asList(Filters.lt("elem.score", 90))
    )
);

// 原子操作：findOneAndUpdate
Document updated = collection.findOneAndUpdate(
    Filters.eq("name", "张三"),
    Updates.inc("visitCount", 1),
    new FindOneAndUpdateOptions()
        .returnDocument(ReturnDocument.AFTER)
        .upsert(false)
);
```

## 3.6 替换操作

```java
// 整体替换（除了_id）
Document replacement = new Document()
    .append("name", "张三")
    .append("age", 30)
    .append("department", "技术部");

collection.replaceOne(Filters.eq("name", "张三"), replacement);

// findOneAndReplace
Document old = collection.findOneAndReplace(
    Filters.eq("name", "张三"),
    replacement,
    new FindOneAndReplaceOptions()
        .returnDocument(ReturnDocument.BEFORE)
);
```

## 3.7 删除操作

```java
// 删除单个
collection.deleteOne(Filters.eq("name", "张三"));

// 删除多个
collection.deleteMany(Filters.lt("age", 18));

// findOneAndDelete
Document deleted = collection.findOneAndDelete(
    Filters.eq("status", "expired"),
    new FindOneAndDeleteOptions().sort(Sorts.ascending("createTime"))
);
```

## 3.8 批量操作

```java
List<WriteModel<Document>> operations = Arrays.asList(
    new InsertOneModel<>(new Document("name", "用户A")),
    new InsertOneModel<>(new Document("name", "用户B")),
    new UpdateOneModel<>(
        Filters.eq("name", "张三"),
        Updates.set("age", 30)
    ),
    new UpdateManyModel<>(
        Filters.lt("age", 18),
        Updates.set("status", "minor")
    ),
    new DeleteOneModel<>(Filters.eq("name", "过期用户"))
);

BulkWriteResult result = collection.bulkWrite(operations,
    new BulkWriteOptions().ordered(false));

System.out.println("插入: " + result.getInsertedCount());
System.out.println("更新: " + result.getModifiedCount());
System.out.println("删除: " + result.getDeletedCount());
```

# 4. Filters构建器

Filters是查询条件的构建器（类似Spring Data的Criteria），提供类型安全的查询条件构造。

```java
import static com.mongodb.client.model.Filters.*;

// 比较
eq("field", value)              // =
ne("field", value)              // !=
gt("field", value)              // >
gte("field", value)             // >=
lt("field", value)              // <
lte("field", value)             // <=
in("field", v1, v2, v3)        // IN
nin("field", v1, v2)           // NOT IN

// 逻辑
and(filter1, filter2, ...)     // AND
or(filter1, filter2, ...)      // OR
nor(filter1, filter2)          // NOR
not(filter)                     // NOT

// 元素
exists("field")                 // 字段存在
exists("field", false)          // 字段不存在
type("field", BsonType.STRING) // 类型判断

// 评估
regex("name", "^张")            // 正则
text("搜索关键词")               // 全文搜索
expr(expr)                      // 聚合表达式

// 数组
all("tags", "Java", "MongoDB") // 包含所有
size("tags", 3)                 // 数组长度
elemMatch(filter)               // 元素匹配
// 示例：至少有一个成绩在80~90之间
elemMatch(
    eq("score", 80),
    gte("score", 80),
    lte("score", 90)
)

// 地理空间
geoWithin("location", polygon)  // 在区域内
near("location", point, maxDist, minDist) // 附近
geoIntersects("location", geometry) // 相交
```

# 5. 聚合操作

```java
// 聚合管道
List<Bson> pipeline = Arrays.asList(
    Aggregates.match(Filters.eq("status", "completed")),
    Aggregates.group("$category",
        Accumulators.sum("totalAmount", "$amount"),
        Accumulators.avg("avgAmount", "$amount"),
        Accumulators.sum("count", 1)
    ),
    Aggregates.sort(Sorts.descending("totalAmount")),
    Aggregates.limit(10)
);

AggregateIterable<Document> results = collection.aggregate(pipeline);

// 复杂聚合
collection.aggregate(Arrays.asList(
    Aggregates.match(Filters.gte("orderDate", startDate)),
    Aggregates.lookup("users", "userId", "_id", "user"),
    Aggregates.unwind("$user"),
    Aggregates.group(
        new Document("year", new Document("$year", "$orderDate"))
            .append("month", new Document("$month", "$orderDate")),
        Accumulators.sum("total", "$amount"),
        Accumulators.sum("count", 1)
    ),
    Aggregates.sort(Sorts.descending("_id.year", "_id.month"))
)).allowDiskUse(true);

// $facet多维度分析
collection.aggregate(Arrays.asList(
    Aggregates.facet(
        new Facet("byCategory",
            Aggregates.group("$category", Accumulators.sum("count", 1))),
        new Facet("priceStats",
            Aggregates.group(null,
                Accumulators.avg("avg", "$price"),
                Accumulators.min("min", "$price"))),
        new Facet("top5",
            Aggregates.sort(Sorts.descending("price")),
            Aggregates.limit(5))
    )
));
```

# 6. POJO映射

## 6.1 基础映射

```java
// 实体类
public class User {
    @BsonId
    private ObjectId id;
    private String name;
    private int age;
    private String email;
    private List<String> tags;
    
    @BsonProperty("create_time")    // 映射不同字段名
    private Date createTime;
    
    @BsonIgnore                     // 忽略字段
    private String tempData;
    
    // getters and setters...
}

// 使用CodecRegistry注册
CodecRegistry pojoCodecRegistry = CodecRegistries.fromRegistries(
    MongoClientSettings.getDefaultCodecRegistry(),
    CodecRegistries.fromProviders(PojoCodecProvider.builder().automatic(true).build())
);

MongoClientSettings settings = MongoClientSettings.builder()
    .codecRegistry(pojoCodecRegistry)
    .build();

MongoCollection<User> collection = database.getCollection("users", User.class);

// 现在可以直接操作User对象
User user = new User();
user.setName("张三");
user.setAge(28);
collection.insertOne(user);

// 查询自动映射为User对象
User found = collection.find(Filters.eq("name", "张三")).first();
```

## 6.2 自定义Codec

```java
// 自定义编码器（复杂类型映射）
public class AddressCodec implements Codec<Address> {
    @Override
    public void encode(BsonWriter writer, Address value, EncoderContext context) {
        writer.writeStartDocument();
        writer.writeString("city", value.getCity());
        writer.writeString("street", value.getStreet());
        writer.writeEndDocument();
    }
    
    @Override
    public Address decode(BsonReader reader, DecoderContext context) {
        Address addr = new Address();
        reader.readStartDocument();
        while (reader.readBsonType() != BsonType.END_OF_DOCUMENT) {
            String fieldName = reader.readName();
            switch (fieldName) {
                case "city": addr.setCity(reader.readString()); break;
                case "street": addr.setStreet(reader.readString()); break;
                default: reader.skipValue();
            }
        }
        reader.readEndDocument();
        return addr;
    }
    
    @Override
    public Class<Address> getEncoderClass() { return Address.class; }
}

// 注册自定义Codec
CodecRegistry registry = CodecRegistries.fromRegistries(
    CodecRegistries.fromCodecs(new AddressCodec()),
    MongoClientSettings.getDefaultCodecRegistry()
);
```

## 6.3 常用注解参考

| 注解 | 说明 |
|------|------|
| @BsonId | 标记_id字段 |
| @BsonProperty | 自定义字段名映射 |
| @BsonIgnore | 忽略字段（不持久化也不读取） |
| @BsonCreator | 标记构造函数用作反序列化 |
| @BsonDiscriminator | 多态类型的鉴别器 |
| @BsonRepresentation | 类型表示映射（如ObjectId↔String） |

```java
public class User {
    @BsonId
    @BsonRepresentation(BsonType.OBJECT_ID)
    private String id;              // 存储为ObjectId，Java中是String
    
    @BsonProperty("email_address")
    private String email;
    
    @BsonCreator
    public User(@BsonProperty("name") String name,
                @BsonProperty("age") int age) {
        this.name = name;
        this.age = age;
    }
}
```

# 7. 事务支持

```java
ClientSession session = mongoClient.startSession();

try {
    session.startTransaction();
    
    collection1.insertOne(session, new Document("name", "事务测试"));
    collection2.updateOne(session, 
        Filters.eq("_id", "counter"),
        Updates.inc("value", 1));
    
    session.commitTransaction();
} catch (Exception e) {
    session.abortTransaction();
    throw e;
} finally {
    session.close();
}

// 使用模板简化
mongoClient.withSession(session -> {
    session.withTransaction(txn -> {
        collection1.insertOne(session, doc1);
        collection2.updateOne(session, filter, update);
        return null;
    });
    return null;
});

// 事务选项
TransactionOptions txnOptions = TransactionOptions.builder()
    .readConcern(ReadConcern.SNAPSHOT)
    .writeConcern(WriteConcern.MAJORITY)
    .build();

session.startTransaction(txnOptions);
```

# 8. Change Streams（变更流）

```java
// 监听集合变更
MongoCursor<ChangeStreamDocument<Document>> cursor = 
    collection.watch().iterator();

while (cursor.hasNext()) {
    ChangeStreamDocument<Document> change = cursor.next();
    OperationType operation = change.getOperationType();
    Document fullDocument = change.getFullDocument();
    BsonDocument documentKey = change.getDocumentKey();
    
    switch (operation) {
        case INSERT:
            System.out.println("插入: " + fullDocument);
            break;
        case UPDATE:
            System.out.println("更新: " + documentKey + " → " + fullDocument);
            break;
        case DELETE:
            System.out.println("删除: " + documentKey);
            break;
        case REPLACE:
            System.out.println("替换: " + fullDocument);
            break;
    }
}

// 条件过滤
collection.watch(
    Arrays.asList(Aggregates.match(
        Filters.in("operationType", 
            Arrays.asList("insert", "update", "replace"))))
);

// 从指定时间开始
collection.watch()
    .startAtOperationTime(BsonTimestamp.valueOf(1705651200));

// Resume Token恢复
collection.watch()
    .resumeAfter(lastResumeToken);
```

# 9. GridFS文件操作

```java
// GridFS上传
GridFSBucket gridFSBucket = GridFSBuckets.create(database, "fs");

try (InputStream stream = new FileInputStream("/path/to/file.pdf")) {
    GridFSUploadOptions options = new GridFSUploadOptions()
        .chunkSizeBytes(261120)        // 255KB块大小
        .metadata(new Document("type", "pdf")
            .append("uploader", "system"));
    
    ObjectId fileId = gridFSBucket.uploadFromStream("report.pdf", stream, options);
}

// GridFS下载
try (OutputStream stream = new FileOutputStream("/path/to/output.pdf")) {
    gridFSBucket.downloadToStream(
        new ObjectId("fileId"),
        stream);
}

// 按文件名下载
gridFSBucket.downloadToStream("report.pdf", stream);

// 查找文件
GridFSFindIterable fileInfo = gridFSBucket.find(
    Filters.eq("metadata.uploader", "system"));

// 删除文件
gridFSBucket.delete(new ObjectId("fileId"));

// 重命名
gridFSBucket.rename(new ObjectId("fileId"), "new_filename.pdf");
```

# 10. 索引管理

```java
// 创建索引
collection.createIndex(Indexes.ascending("email"),
    new IndexOptions().unique(true));

collection.createIndex(Indexes.compoundIndex(
    Indexes.ascending("status"),
    Indexes.descending("createTime")
));

// 文本索引
collection.createIndex(Indexes.text("content"));

// TTL索引
collection.createIndex(Indexes.ascending("createTime"),
    new IndexOptions().expireAfter(3600L, TimeUnit.SECONDS));

// 查看索引
for (Document index : collection.listIndexes()) {
    System.out.println(index.toJson());
}

// 删除索引
collection.dropIndex("indexName");
collection.dropIndex(Indexes.ascending("email"));

// 通过explain分析查询
Document explanation = collection.find(filter)
    .modifiers(new Document("$explain", true)).first();
// 或使用runCommand
Document explainResult = database.runCommand(new Document("explain",
    new Document("find", "users")
        .append("filter", new Document("name", "张三"))));
```

# 11. 客户端最佳实践

## 11.1 MongoClient生命周期

```java
// MongoClient是线程安全且管理连接池的，应作为单例使用
// ✅ 正确：应用启动时创建，关闭时销毁
@Component
public class MongoConfig {
    
    @Bean(destroyMethod = "close")
    public MongoClient mongoClient() {
        return MongoClients.create(settings);
    }
}
```

## 11.2 异常处理

```java
try {
    collection.insertOne(doc);
} catch (MongoWriteException e) {
    if (e.getCode() == 11000) {
        // 唯一索引冲突（Duplicate Key Error）
        System.out.println("文档已存在: " + e.getMessage());
    }
} catch (MongoTimeoutException e) {
    // 操作超时，可能需要重试
} catch (MongoExecutionTimeoutException e) {
    // 查询执行超时（如设置了maxTime）
} catch (MongoCommandException e) {
    // 命令执行错误
    System.out.println("错误代码: " + e.getCode());
    System.out.println("错误消息: " + e.getErrorMessage());
}
```

## 11.3 配置建议

```java
// 生产环境推荐配置
MongoClientSettings.builder()
    .applyToClusterSettings(b -> b
        .hosts(hosts)
        .requiredReplicaSetName("rs0"))
    .applyToConnectionPoolSettings(b -> b
        .maxSize(50)
        .minSize(5))
    .applyToSocketSettings(b -> b
        .connectTimeout(5, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS))
    .credential(credential)
    .readPreference(ReadPreference.secondaryPreferred())
    .writeConcern(WriteConcern.MAJORITY)
    .readConcern(ReadConcern.MAJORITY)
    .compressorList(Arrays.asList(
        MongoCompressor.createZstdCompressor()))
    .applicationName("my-service-v1")
    .build();
```
