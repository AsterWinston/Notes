
# 1. 开发最佳实践

## 1.1 数据建模

- **优先嵌入式文档**：数据一起访问就一起存储，减少JOIN
- **避免文档无限增长**：数组字段会持续增长（如用户评论），使用引用或桶模式
- **字段命名一致**：统一使用camelCase或snake_case，不要混用
- **避免大文档**：单个文档控制在1MB以内，超大文档考虑拆分
- **使用Schema Validation**：生产环境务必开启校验

```javascript
// ✅ 好的数据模型
{
    _id: ObjectId,
    name: "张三",
    orders: [
        { orderId: "ORD001", amount: 299 }
    ],
    orderCount: 1,      // 冗余计数，避免$size
    lastOrderDate: Date  // 冗余，避免每次排序
}

// ❌ 不好的数据模型
{
    _id: ObjectId,
    name: "张三",
    orderIds: ["ORD001", "ORD002"],  // 每次查询需要$lookup
    order_details: /* 无限增长的数组 */
}
```

## 1.2 查询设计

- **所有查询都应有索引覆盖**：定期检查慢查询日志，确保新查询有索引
- **$match放在聚合管道最前面**：充分利用索引
- **使用投影减少传输**：只返回需要的字段
- **游标分页代替skip分页**：大offset的分页性能极差
- **$in代替$or（同字段时）**：$in的查询计划更优
- **覆盖查询优化**：高QPS查询尽量使用覆盖索引

```javascript
// ✅ 游标分页
db.collection.find({ _id: { $lt: lastId } })
    .sort({ _id: -1 }).limit(20)

// ❌ skip分页
db.collection.find().sort({ _id: -1 }).skip(100000).limit(20)

// ✅ 只返回需要的字段
db.users.find({}, { name: 1, email: 1, _id: 0 })

// ❌ 返回所有字段
db.users.find({})
```

## 1.3 写入优化

- **批量写入代替逐条写入**：使用insertMany/bulkWrite
- **ordered: false**：无序批量操作，提高并发
- **合理使用Write Concern**：非关键数据可适当降低写关注级别
- **避免在高峰期创建索引**：使用background模式（4.2+默认）
- **写入峰值时不触发Balancer迁移**：分片集群设置迁移窗口

## 1.4 事务使用

- **能用单文档原子操作就不用事务**：单文档操作天然原子性
- **能用嵌入式文档就不用跨文档事务**：数据一起访问就一起存储
- **事务要短小精悍**：控制在1秒内，文档数<1000
- **处理写冲突重试**：实现TransientTransactionError重试逻辑

# 2. 运维最佳实践

## 2.1 部署架构

- **生产环境必须使用副本集**：最少3个数据节点
- **合理配置Oplog大小**：至少覆盖24小时写操作
- **使用奇数个投票节点**：避免脑裂
- **从节点部署在异地**：增强容灾
- **mongos与服务同机部署**：减少延迟
- **监控Config Server**：分片集群的元数据中心

## 2.2 安全加固

- **开启认证**：生产环境必须启用authorization
- **使用TLS加密通信**：内部和外部通信都加密
- **最小权限原则**：为每个应用创建独立用户
- **定期轮换密码**：至少每90天一次
- **禁用服务端JavaScript**（如不需要）：`javascriptEnabled: false`
- **绑定内网IP**：不要暴露到公网
- **使用防火墙限制端口访问**

## 2.3 容量规划

| 指标 | 建议 |
|------|------|
| 文档大小 | < 1MB |
| 集合文档数 | 非分片 < 1亿 |
| 单集合大小 | 非分片 < 200GB |
| 索引数 | 每个集合 5~10个 |
| 连接数 | < 10000 |
| 磁盘IOPS | SSD > 3000 |
| 内存 | 热数据 + 索引 < 缓存大小 |

## 2.4 监控指标

关键指标需要持续监控和告警：
- 连接使用率
- 操作延迟（p99）
- 复制延迟
- Oplog窗口
- WiredTiger缓存命中率
- 磁盘空间使用率
- 慢查询比例
- 异常错误率

# 3. 应用层最佳实践

## 3.1 连接管理

```java
// ✅ MongoClient作为单例，全局复用
@Bean(destroyMethod = "close")
public MongoClient mongoClient() {
    return MongoClients.create(settings);
}

// ✅ 合理配置连接池
.applyToConnectionPoolSettings(b -> b
    .maxSize(50)
    .minSize(5)
    .maxWaitTime(2, TimeUnit.SECONDS))

// ❌ 每次请求创建新连接
public void doSomething() {
    MongoClient client = MongoClients.create();  // 错误！
}
```

## 3.2 异常处理

```java
try {
    collection.insertOne(doc);
} catch (MongoWriteException e) {
    if (e.getCode() == 11000) {
        // 唯一索引冲突 → 幂等处理或提示用户
    }
    throw e;
} catch (MongoTimeoutException e) {
    // 超时 → 重试或降级
} catch (MongoExecutionTimeoutException e) {
    // 查询超时 → 优化查询或增加maxTime
} catch (MongoSocketException e) {
    // 网络异常 → 重试（利用驱动的自动重试）
}
```

## 3.3 N+1问题避免

```java
// ❌ N+1查询
List<Order> orders = orderRepo.findAll();
for (Order order : orders) {
    User user = userRepo.findById(order.getUserId()); // 每条order一次查询
    order.setUserName(user.getName());
}

// ✅ 使用$lookup或预先批量查询
// 方案1: 聚合$lookup
// 方案2: 先收集所有userId，再一次性查询所有user
Set<String> userIds = orders.stream()
    .map(Order::getUserId).collect(Collectors.toSet());
Map<String, User> userMap = userRepo.findByIdIn(userIds).stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));
```

## 3.4 分页实现

```java
// ✅ 游标分页（推荐）
@GetMapping("/users")
public PageResult<User> listByCursor(
        @RequestParam(required = false) String lastId,
        @RequestParam(defaultValue = "20") int size) {
    
    Query query = new Query();
    if (lastId != null) {
        query.addCriteria(Criteria.where("_id").lt(new ObjectId(lastId)));
    }
    query.with(Sort.by(Sort.Direction.DESC, "_id")).limit(size);
    
    List<User> users = mongoTemplate.find(query, User.class);
    String nextCursor = users.isEmpty() ? null : 
        users.get(users.size() - 1).getId();
    
    return new PageResult(users, nextCursor);
}
```

# 4. 版本升级建议

## 4.1 升级策略

```
升级不能跨大版本：
4.0 → 4.2 → 4.4 → 5.0 → 6.0 → 7.0

升级步骤：
1. 升级所有mongos
2. 升级所有config server（先升级从节点）
3. 升级所有分片（先升级从节点）
4. 执行setFeatureCompatibilityVersion
```

## 4.2 兼容性版本

```javascript
// 查看当前featureCompatibilityVersion
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })

// 设置兼容性版本（允许降级）
db.adminCommand({ setFeatureCompatibilityVersion: "6.0" })

// 升级完成后设置新版本（不可降级）
db.adminCommand({ setFeatureCompatibilityVersion: "7.0" })
```

# 5. 反模式清单

| 反模式 | 描述 | 后果 | 正确做法 |
|--------|------|------|---------|
| 数组无限增长 | 评论、订单历史嵌入文档 | 文档超16MB、更新缓慢 | 引用或溢出桶 |
| 全量大文档 | 将整个页面数据存为一个文档 | IO浪费、缓存效率低 | 按访问模式拆分 |
| 大量索引 | 每个查询都建索引 | 写入极慢、内存不足 | 仅高频查询建索引 |
| $regex无前缀 | /pattern/而非/^prefix/ | 全表扫描 | 前缀匹配或全文索引 |
| 大量skip分页 | skip(100000).limit(20) | 扫描10万+文档 | 游标分页 |
| 硬编码_id | 使用业务键可能导致分布不均 | 分片热点 | 使用ObjectId或哈希 |
| in子句过大 | $in数组成千上万元素 | 查询性能差 | 分批查询 |
| 频繁compact | 定期日常compact | 阻塞操作 | 仅在必要时执行 |

# 6. 编码规范建议

## 6.1 Java实体设计

```java
// ✅ 推荐的实体设计
@Document("users")
@CompoundIndexes({
    @CompoundIndex(name = "idx_status_time",
                   def = "{'status': 1, 'createTime': -1}")
})
public class User {
    
    @Id
    private String id;
    
    @Field("user_name")
    @NotBlank(message = "用户名不能为空")
    private String userName;
    
    @Indexed(unique = true)
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Field(targetType = FieldType.INT32)
    @Min(0) @Max(150)
    private Integer age;
    
    @CreatedDate
    private LocalDateTime createTime;
    
    @Version
    private Long version;  // 乐观锁
    
    // 始终排除大字段
    @DocumentReference  // Spring Data 3.0+
    private UserProfile profile;  // 懒加载引用
    
    // 使用嵌入式时注意大小
    private Address address;  // 小对象可以嵌入
}
```

## 6.2 Repository接口规范

```java
// ✅ 推荐的Repository定义
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    
    // 方法命名清晰，语义明确
    Optional<User> findByEmail(String email);
    
    // 批量查询避免多次RTT
    @Query("{ '_id': { $in: ?0 } }")
    List<User> findByIds(Collection<String> ids);
    
    // 使用projection减少传输
    @Query(value = "{ 'status': ?0 }", 
           fields = "{ 'name': 1, 'email': 1 }")
    List<User> findNamesByStatus(String status);
    
    // 明确返回类型
    boolean existsByEmail(String email);
    long countByStatus(String status);
    void deleteByStatus(String status);
}
```
