
# 1. Spring Data MongoDB概述

Spring Data MongoDB是Spring Data家族的一员，在MongoDB Java Driver之上提供更高层次的抽象，类似于Spring Data JPA对JDBC的封装。

核心功能：
- MongoTemplate（底层操作模板）
- Repository接口（类似JPA的数据访问层）
- 自动配置（Spring Boot Auto Configuration）
- 注解驱动的实体映射
- 聚合框架DSL
- 事务支持
- 审计功能

## 1.1 Maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

## 1.2 基本配置

```yaml
# application.yml
spring:
  data:
    mongodb:
      uri: mongodb://user:pass@localhost:27017/mydb
      # 或分散配置
      host: localhost
      port: 27017
      database: mydb
      username: user
      password: pass
      authentication-database: admin
      # 副本集
      # uri: mongodb://mongo1:27017,mongo2:27017/mydb?replicaSet=rs0
      # 连接池（不推荐在此配置，在uri中配置）
      # uri: mongodb://localhost:27017/mydb?maxPoolSize=50&minPoolSize=5
```

# 2. 实体映射注解

## 2.1 核心注解

```java
@Document(collection = "users")    // 映射到集合
public class User {
    
    @Id                               // 主键
    private String id;                // MongoDB默认生成ObjectId字符串
    
    @Field("user_name")               // 自定义字段名
    private String userName;
    
    @Field(targetType = FieldType.INT64)
    private Long score;
    
    @Indexed(unique = true)           // 创建唯一索引
    private String email;
    
    @Indexed(expireAfterSeconds = 3600) // TTL索引
    private Date createTime;
    
    @CompoundIndex(name = "idx_status_age",
        def = "{'status': 1, 'age': -1}")
    public class User {
        // ...
    }
    
    @TextIndexed(weight = 10)         // 文本索引（需联合@TextScore）
    private String title;
    
    @GeoIndexed(type = GeoIndexType.GEO_2DSPHERE)
    private GeoJsonPoint location;
    
    @Transient                        // 不持久化
    private String tempData;
    
    @Version                          // 乐观锁版本号
    private Long version;
    
    @ReadOnlyProperty                 // 只读（插入时忽略）
    private Date updatedAt;
    
    // getters and setters...
}
```

## 2.2 复合索引注解

```java
@Document
@CompoundIndexes({
    @CompoundIndex(name = "idx_status_time", 
                    def = "{'status': 1, 'createTime': -1}"),
    @CompoundIndex(name = "idx_dept_age", 
                    def = "{'department': 1, 'age': 1}",
                    unique = true)
})
public class User {
    @Id
    private String id;
    private String status;
    private Date createTime;
    private String department;
    private int age;
}
```

## 2.3 地理空间类型

```java
public class Store {
    @Id
    private String id;
    
    @GeoIndexed(type = GeoIndexType.GEO_2DSPHERE)
    private GeoJsonPoint location;     // Point
    
    private GeoJsonPolygon deliveryArea; // Polygon
    
    // 赋值方式
    // location = new GeoJsonPoint(116.397, 39.908)
}
```

# 3. MongoTemplate

MongoTemplate是Spring Data MongoDB的核心操作类，提供非Repository风格的数据访问。

## 3.1 基本CRUD

```java
@Autowired
private MongoTemplate mongoTemplate;

// ===== 插入 =====
User user = new User("张三", 28);
User saved = mongoTemplate.insert(user);
// insert如果user已有ID则使用该ID

// 批量插入
List<User> users = Arrays.asList(
    new User("李四", 25),
    new User("王五", 30)
);
mongoTemplate.insert(users, User.class);

// insertAll（插入到指定集合）
mongoTemplate.insertAll(users);

// ===== 查询 =====
// 查询所有
List<User> allUsers = mongoTemplate.findAll(User.class);

// 条件查询
Query query = new Query(Criteria.where("age").gt(25));
List<User> result = mongoTemplate.find(query, User.class);

// 查询单个
User user = mongoTemplate.findOne(query, User.class);

// 根据ID查询
User user = mongoTemplate.findById("507f1f77bcf86cd799439011", User.class);

// ===== 更新 =====
Query query = new Query(Criteria.where("name").is("张三"));
Update update = new Update()
    .set("age", 29)
    .inc("visitCount", 1);

// updateFirst：更新第一个
UpdateResult result = mongoTemplate.updateFirst(query, update, User.class);

// updateMulti：更新所有匹配
mongoTemplate.updateMulti(query, update, User.class);

// upsert
UpdateResult result = mongoTemplate.upsert(query, update, User.class);

// findAndModify
User updated = mongoTemplate.findAndModify(
    query, update, 
    new FindAndModifyOptions().returnNew(true), 
    User.class);

// ===== 删除 =====
mongoTemplate.remove(query, User.class);
mongoTemplate.remove(user);    // 根据ID删除
mongoTemplate.findAllAndRemove(query, User.class);
// findAndRemove
User removed = mongoTemplate.findAndRemove(query, User.class);

// 删除集合
mongoTemplate.dropCollection(User.class);
```

## 3.2 Query详解

```java
// 基本查询
Query query = new Query();

// Criteria组合
query.addCriteria(Criteria.where("age").gt(25));
query.addCriteria(Criteria.where("status").is("active"));

// 逻辑组合
query.addCriteria(new Criteria().orOperator(
    Criteria.where("status").is("active"),
    Criteria.where("vipLevel").gt(3)
));

// 正则查询
query.addCriteria(Criteria.where("name").regex("^张"));

// 数组查询
query.addCriteria(Criteria.where("tags").all("Java", "MongoDB"));
query.addCriteria(Criteria.where("tags").size(3));
query.addCriteria(Criteria.where("scores").elemMatch(
    Criteria.where("score").gte(80).lte(90)
));

// 嵌套字段查询
query.addCriteria(Criteria.where("address.city").is("北京"));

// 排序
query.with(Sort.by(Sort.Direction.DESC, "createTime"));
query.with(Sort.by(Sort.Order.desc("createTime"), Sort.Order.asc("name")));

// 分页
query.skip(100).limit(20);

// 投影（字段选择）
query.fields().include("name").include("age").exclude("_id");

// 分页（配合Pageable）
Pageable pageable = PageRequest.of(0, 20, Sort.by("createTime").descending());
query.with(pageable);

// 查询注释（诊断用）
query.comment("查询活跃用户");
```

## 3.3 Update详解

```java
Update update = new Update();

// 基本操作
update.set("age", 30);                  // 设置
update.unset("tempField");              // 删除字段
update.inc("loginCount", 1);            // 自增
update.mul("price", 1.2);               // 乘法
update.min("score", 60);                // 仅设置更小值
update.max("bestScore", 95);            // 仅设置更大值
update.rename("oldName", "newName");    // 重命名
update.currentDate("lastModified");     // 当前时间
update.setOnInsert("createTime", new Date()); // 仅在插入时设置

// 数组操作
update.push("tags", "Spring");          // 追加
update.addToSet("tags", "Spring");      // 去重追加
update.pop("tags", Update.Position.FIRST); // 弹出首元素
update.pop("tags", Update.Position.LAST);  // 弹出尾元素
update.pull("tags", "Spring");          // 移除匹配元素
update.pullAll("tags",                  // 移除多个
    Arrays.asList("Spring", "Java"));

// Bitwise
update.bitwise("permissions", Update.BitwiseOperator.AND, 0x0F);
```

## 3.4 聚合操作

```java
// 聚合构建
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("status").is("completed")),
    Aggregation.group("category")
        .sum("amount").as("totalAmount")
        .avg("amount").as("avgAmount")
        .count().as("orderCount"),
    Aggregation.sort(Sort.Direction.DESC, "totalAmount"),
    Aggregation.limit(10)
);

AggregationResults<CategoryStat> results = 
    mongoTemplate.aggregate(aggregation, "orders", CategoryStat.class);

// 复杂聚合
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("orderDate")
        .gte(LocalDate.of(2024, 1, 1))
        .lt(LocalDate.of(2025, 1, 1))),
    Aggregation.lookup("users", "userId", "_id", "user"),
    Aggregation.unwind("user"),
    Aggregation.group(
        Fields.from(
            Fields.field("year", DateOperators.Year.year("$orderDate")),
            Fields.field("month", DateOperators.Month.month("$orderDate"))))
        .sum("total").as("revenue")
        .count().as("count"),
    Aggregation.sort(Sort.Direction.DESC, "revenue")
).withOptions(AggregationOptions.builder().allowDiskUse(true).build());
```

## 3.5 聚合类型投影

```java
// 使用TypedAggregation获得类型安全
TypedAggregation<Order> aggregation = Aggregation.newAggregation(
    Order.class,
    Aggregation.match(Criteria.where("status").is("completed")),
    Aggregation.group("userId")
        .sum("total").as("totalSpent")
);

AggregationResults<UserSpending> results = 
    mongoTemplate.aggregate(aggregation, UserSpending.class);
```

## 3.6 分页查询

```java
// 基础分页
Pageable pageable = PageRequest.of(0, 20, Sort.by("createTime").descending());
Query query = new Query().with(pageable);
List<User> users = mongoTemplate.find(query, User.class);

// 获取总数用于分页
long count = mongoTemplate.count(query, User.class);

// 完整分页（一次查询获取数据和总数）
Pageable pageable = PageRequest.of(0, 20);
Query query = new Query().with(pageable);
List<User> users = mongoTemplate.find(query, User.class);
long count = mongoTemplate.count(Query.of(query).limit(-1).skip(-1), User.class);
Page<User> page = new PageImpl<>(users, pageable, count);

// 游标分页（大偏移量场景）
Query query = new Query(Criteria.where("_id").lt(lastId))
    .with(Sort.by(Sort.Direction.DESC, "_id"))
    .limit(20);
```

# 4. Repository接口

Spring Data MongoDB Repository提供了类似于Spring Data JPA的声明式数据访问。

## 4.1 基础Repository

```java
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    // 继承的方法：
    // save, saveAll, findById, findAll, findAllById
    // count, delete, deleteById, deleteAll
    // existsById
}
```

## 4.2 方法命名查询

```java
public interface UserRepository extends MongoRepository<User, String> {
    
    // 等值查询
    List<User> findByName(String name);
    User findByEmail(String email);
    
    // 模糊匹配
    List<User> findByNameLike(String pattern);     // name LIKE '%pattern%'
    List<User> findByNameStartingWith(String prefix);
    
    // 比较
    List<User> findByAgeGreaterThan(int age);
    List<User> findByAgeBetween(int min, int max);
    
    // 多条件
    List<User> findByStatusAndAgeGreaterThan(String status, int age);
    
    // 排序
    List<User> findByStatusOrderByCreateTimeDesc(String status);
    
    // Top/First + 限制数量
    List<User> findTop5ByStatus(String status);
    User findFirstByOrderByCreateTimeDesc();
    
    // 存在性
    boolean existsByEmail(String email);
    
    // 计数
    long countByStatus(String status);
    
    // 删除
    long deleteByStatus(String status);
    
    // 数组查询
    List<User> findByTagsIn(List<String> tags);
    List<User> findByTagsAll(List<String> tags);      // 包含所有
    
    // 嵌套字段
    List<User> findByAddressCity(String city);
    
    // 正则
    List<User> findByNameRegex(String regex);
}
```

## 4.3 @Query注解查询

```java
public interface UserRepository extends MongoRepository<User, String> {
    
    // JSON查询
    @Query("{ 'name': ?0 }")
    List<User> findByUserName(String name);
    
    // 多参数
    @Query("{ 'status': ?0, 'age': { $gt: ?1 } }")
    List<User> findByStatusAndAgeGreaterThan(String status, int age);
    
    // 排序
    @Query(value = "{ 'status': ?0 }", sort = "{ 'createTime': -1 }")
    List<User> findByStatusSorted(String status);
    
    // 字段投影
    @Query(value = "{ 'age': { $gt: ?0 } }", 
           fields = "{ 'name': 1, 'email': 1 }")
    List<User> findByAgeProjected(int age);
    
    // exists判断
    @Query(exists = true, value = "{ 'status': 'active' }")
    boolean hasActiveUsers();
    
    // 计数
    @Query(count = true, value = "{ 'status': ?0 }")
    long countByStatusQuery(String status);
    
    // 删除
    @Query(delete = true, value = "{ 'status': 'expired' }")
    long removeAllExpired();
    
    // 复杂条件（SpEL）
    @Query("{ $or: [ { 'name': ?0 }, { 'email': ?0 } ] }")
    List<User> findByNameOrEmail(String keyword);
}
```

## 4.4 自定义Repository实现

```java
// 1. 定义自定义方法接口
public interface UserRepositoryCustom {
    List<User> findComplexUsers(Criteria criteria);
    void batchUpdateStatus(List<String> ids, String status);
}

// 2. 实现接口
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Override
    public List<User> findComplexUsers(Criteria criteria) {
        Query query = new Query(criteria);
        return mongoTemplate.find(query, User.class);
    }
    
    @Override
    public void batchUpdateStatus(List<String> ids, String status) {
        Query query = new Query(Criteria.where("_id").in(ids));
        Update update = new Update().set("status", status);
        mongoTemplate.updateMulti(query, update, User.class);
    }
}

// 3. Repository继承自定义接口
public interface UserRepository extends 
        MongoRepository<User, String>, 
        UserRepositoryCustom {
    // ...
}
```

## 4.5 分页与排序

```java
// 方法级别
List<User> findByStatus(String status, Pageable pageable);
List<User> findByAgeGreaterThan(int age, Sort sort);

// 使用示例
PageRequest pageRequest = PageRequest.of(0, 20, 
    Sort.by(Sort.Direction.DESC, "createTime")
        .and(Sort.by("name")));

Page<User> page = userRepository.findByStatus("active", pageRequest);
long totalElements = page.getTotalElements();
int totalPages = page.getTotalPages();
List<User> content = page.getContent();
```

# 5. 审计功能

```java
// 1. 启用审计
@Configuration
@EnableMongoAuditing
public class MongoConfig {
}

// 2. 实体类
@Document
public class User {
    @Id
    private String id;
    
    @CreatedDate
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
}

// 3. 实现AuditorAware
@Component
public class AuditorAwareImpl implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        // 从SecurityContext获取当前用户
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            return Optional.empty();
        }
        return Optional.of(auth.getName());
    }
}
```

# 6. 事务支持

```java
// 1. 启用事务
@Configuration
public class MongoConfig {
    
    @Bean
    MongoTransactionManager transactionManager(MongoDatabaseFactory factory) {
        return new MongoTransactionManager(factory);
    }
}

// 2. 使用@Transactional
@Service
public class OrderService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Transactional
    public void createOrder(Order order) {
        mongoTemplate.insert(order);
        
        // 减库存
        Query query = new Query(Criteria.where("_id").is(order.getProductId())
            .and("stock").gt(0));
        Update update = new Update().inc("stock", -order.getQuantity());
        
        UpdateResult result = mongoTemplate.updateFirst(query, update, Product.class);
        if (result.getMatchedCount() == 0) {
            throw new RuntimeException("库存不足");
        }
    }
}

// 3. ClientSession手动控制
@Autowired
private MongoTemplate mongoTemplate;

public void manualTransaction() {
    try (ClientSession session = mongoTemplate.getDb().getMongoClient()
            .startSession()) {
        session.startTransaction();
        try {
            // 操作...
            session.commitTransaction();
        } catch (Exception e) {
            session.abortTransaction();
            throw e;
        }
    }
}
```

# 7. ZonedDateTime支持

MongoDB不原生支持Java的ZonedDateTime，需要自定义转换：

```java
@Configuration
public class MongoConfig {
    
    @Bean
    public MongoCustomConversions mongoCustomConversions() {
        return new MongoCustomConversions(Arrays.asList(
            new ZonedDateTimeReadConverter(),
            new ZonedDateTimeWriteConverter()
        ));
    }
    
    static class ZonedDateTimeWriteConverter 
            implements Converter<ZonedDateTime, Date> {
        @Override
        public Date convert(ZonedDateTime source) {
            return Date.from(source.toInstant());
        }
    }
    
    static class ZonedDateTimeReadConverter 
            implements Converter<Date, ZonedDateTime> {
        @Override
        public ZonedDateTime convert(Date source) {
            return ZonedDateTime.ofInstant(
                source.toInstant(), ZoneId.systemDefault());
        }
    }
}
```

# 8. GridFS操作

```java
@Autowired
private GridFsTemplate gridFsTemplate;

// 上传
public String uploadFile(InputStream inputStream, String filename) {
    GridFSFile file = gridFsTemplate.store(
        inputStream, 
        filename, 
        new Document("contentType", "image/png")
    );
    return file.getId().toString();
}

// 下载
public GridFSFile getFile(String fileId) {
    return gridFsTemplate.findOne(
        new Query(Criteria.where("_id").is(fileId)));
}

// 删除
public void deleteFile(String fileId) {
    gridFsTemplate.delete(
        new Query(Criteria.where("_id").is(fileId)));
}

@Autowired
private GridFsOperations gridFsOperations;

// 更多操作通过GridFsOperations
```

# 9. 配置参考

```java
@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {
    
    @Value("${spring.data.mongodb.uri}")
    private String uri;
    
    @Override
    protected String getDatabaseName() {
        return "mydb";
    }
    
    @Override
    public MongoClient mongoClient() {
        return MongoClients.create(uri);
    }
    
    @Bean
    public MongoTransactionManager transactionManager(
            MongoDatabaseFactory dbFactory) {
        return new MongoTransactionManager(dbFactory);
    }
    
    @Bean
    public MongoCustomConversions mongoCustomConversions() {
        return new MongoCustomConversions(Arrays.asList(
            // 自定义转换器...
        ));
    }
    
    @Override
    protected void configureClientSettings(
            MongoClientSettings.Builder builder) {
        builder
            .applicationName("my-app")
            .compressorList(Arrays.asList(
                MongoCompressor.createZstdCompressor()))
            .applyToConnectionPoolSettings(b -> b
                .maxSize(50).minSize(5));
    }
}
```

# 10. Spring Boot 3.x兼容性

Spring Boot 3.x使用了Jakarta EE，需要注意：
- MongoDB Driver版本需 ≥ 4.9
- Spring Data MongoDB版本需 ≥ 4.2
- Java版本需 ≥ 17

```xml
<!-- Spring Boot 3.x 默认使用兼容版本，无需额外配置 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>
```
