# 1. Spring Data JDBC 概述

## 1.1 什么是 Spring Data JDBC

Spring Data JDBC 是 Spring Data 家族的一员，它提供了一种基于纯 JDBC 的数据访问方式，但与直接使用 JDBC 不同，它引入了 DDD（领域驱动设计）中 Aggregate（聚合）和 Repository（仓库）的概念，简化了数据库操作。

Spring Data JDBC 的设计目标是：
- **简单直接**：不引入 JPA 那样的复杂映射机制（懒加载、脏检查、一级缓存等）
- **DDD 聚合根理念**：以聚合为操作单元，符合 DDD 设计思想
- **可预测的 SQL**：没有魔法，执行的 SQL 语句一目了然
- **Spring Data 编程模型**：使用 Repository 接口模式，与 Spring Data JPA 保持一致的编程体验

## 1.2 为什么需要 Spring Data JDBC —— 与其他方案对比

| 特性 | Spring Data JDBC | Spring Data JPA/Hibernate | MyBatis | 原生 JDBC |
|------|------------------|---------------------------|---------|-----------|
| 学习成本 | 低 | 高 | 中 | 低 |
| SQL 可控性 | 高 | 低（自动生成） | 高 | 完全控制 |
| 懒加载支持 | 不支持 | 支持 | 手动实现 | 无 |
| 脏检查/自动更新 | 不支持 | 支持 | 无 | 无 |
| DDD 支持 | 原生支持 | 可配合使用 | 无 | 无 |
| ORM 映射 | 简单对象映射 | 完整 ORM | XML/注解映射 | 手动映射 |
| 缓存机制 | 无内置 | 一级/二级缓存 | 可集成 | 无 |

**适用场景**：
- 领域驱动设计（DDD）项目，需要明确聚合边界
- 对 SQL 有精确控制需求，但不想手写所有样板代码
- 数据库操作相对简单，不需要复杂的 ORM 特性
- 追求高性能，避免 ORM 框架带来的性能开销

## 1.3 Spring Data JDBC 与 Spring Data JPA 的核心区别

**架构理念差异**：
- JPA/Hibernate 试图将面向对象的世界与关系数据库进行无缝映射，引入了持久化上下文、懒加载、脏检查等复杂机制
- Spring Data JDBC 认为对象模型与关系模型天然不匹配，不应该强行映射，而是以更简单直接的方式进行数据库交互

**聚合加载策略差异**：
- JPA 默认懒加载，需要时通过代理延迟获取——容易产生 N+1 问题
- Spring Data JDBC 加载聚合时一次性加载所有依赖实体——避免了 N+1 问题但要注意聚合粒度

**写入策略差异**：
- JPA 通过脏检查自动发现变更并生成 UPDATE 语句
- Spring Data JDBC 比较聚合中所有实体来判断是否需要写入——如果你更新了聚合根中的某个字段，整个聚合可能需要重新写入

# 2. 核心概念

## 2.1 聚合（Aggregate）

聚合是 DDD 中的一个核心概念，它由一组不可分割的关联对象组成，通过聚合根（Aggregate Root）来访问。

在 Spring Data JDBC 中：
- 一个聚合对应数据库中的一个或多个表
- 聚合根是一个带有 `@Id` 注解的实体
- 通过聚合根来执行所有 CRUD 操作
- 聚合内部的对象之间通过引用关联，而不是通过数据库外键

```
聚合的设计原则：
- 聚合根的 ID 是全局唯一标识
- 聚合内部实体通过聚合根引用，不跨聚合直接引用
- 聚合是一个事务一致性边界
- 尽量设计小聚合
```

## 2.2 聚合设计示例

```java
// 聚合根
public class Order {
    @Id
    private Long id;
    private String orderNumber;
    private Long customerId;          // 跨聚合引用：只引 ID，不引对象
    private LocalDateTime createTime;
    // 一对多：聚合内部实体，与聚合根生命周期一致
    private Set<OrderItem> items = new HashSet<>();
    // 一对一：聚合内部实体
    private OrderAddress address;
}

// 聚合内部实体
public class OrderItem {
    private Long productId;           // 跨聚合引用
    private int quantity;
    private BigDecimal price;
}

// 聚合内部实体（值对象）
public class OrderAddress {
    private String province;
    private String city;
    private String detail;
}
```

## 2.3 跨聚合引用规则

在 Spring Data JDBC 中，跨聚合引用只存储 ID 值，不存储对象引用：

```java
// 正确做法：只存 ID
public class Order {
    private Long customerId;    // 引用 Customer 聚合
}

// 错误做法：存储对象引用（Spring Data JDBC 无此概念）
public class Order {
    private Customer customer; 
}
```

# 3. 快速入门

## 3.1 Maven 依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<!-- 数据库驱动 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

## 3.2 数据源配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
```

## 3.3 实体定义与表结构映射

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.relational.core.mapping.Column;
import java.time.LocalDateTime;

@Table("t_user")           // 映射到表 t_user
public class User {

    @Id                     // 标记主键
    private Long id;

    @Column("user_name")    // 映射字段名（类属性名与列名不一致时使用）
    private String userName;

    private String email;

    private LocalDateTime createTime;

    // getters and setters...
}
```

**自动映射规则**：
- 类名通过 `NamingStrategy` 转换为表名（默认驼峰转下划线：`UserProfile` → `user_profile`）
- 属性名通过 `NamingStrategy` 转换为列名
- 可自定义 `NamingStrategy` Bean 改变命名规则

## 3.4 Repository 接口定义

```java
import org.springframework.data.repository.CrudRepository;
import java.util.List;

public interface UserRepository extends CrudRepository<User, Long> {
    // CrudRepository 内置方法：
    //   save(S entity)       - 新增或更新
    //   saveAll(Iterable<S>) - 批量保存
    //   findById(ID)         - 按ID查询，返回 Optional
    //   existsById(ID)       - 判断ID是否存在
    //   findAll()            - 查询全部
    //   findAllById(Iterable<ID>) - 批量按ID查询
    //   count()              - 统计总数
    //   deleteById(ID)       - 按ID删除
    //   delete(S entity)     - 按实体删除
    //   deleteAllById(Iterable<ID>) - 批量按ID删除
    //   deleteAll(Iterable<S>) - 批量删除
    //   deleteAll()          - 删除全部

    // 派生查询（方法命名约定自动生成 SQL）
    List<User> findByUserName(String userName);

    List<User> findByEmailContaining(String keyword);

    boolean existsByEmail(String email);

    long countByCreateTimeAfter(LocalDateTime time);
}
```

## 3.5 在 Service 层使用

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(User user) {
        user.setCreateTime(LocalDateTime.now());
        return userRepository.save(user);
    }

    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }

    public List<User> findByName(String name) {
        return userRepository.findByUserName(name);
    }
}
```

# 4. 实体映射详解

## 4.1 主键策略

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;

@Table("t_product")
public class Product {

    @Id  // 主键标记，必需
    private Long id;

    // 其他字段...
}
```

Spring Data JDBC 判断 insert vs update 的规则：
- 如果 `@Id` 字段值为 `null` 或原始类型默认值（如 `0`），执行 **INSERT**
- 如果 `@Id` 字段值非空且非默认值，执行 **UPDATE**

使用聚合根引用作为 ID 时，需确保 insert 时 ID 为 null：

```java
// insert 操作
User user = new User();
user.setId(null);  // 明确设为 null，框架会认为这是新增
user.setUserName("张三");
userRepository.save(user);

// update 操作
User user = userRepository.findById(1L).orElseThrow();
user.setEmail("new@example.com");
userRepository.save(user);  // id 不为 null，执行 update
```

## 4.2 嵌套实体与一对多关系

Spring Data JDBC 的嵌套实体通过对象引用实现，不依赖数据库外键约束：

```java
import java.util.Set;

public class Article {
    @Id
    private Long id;
    private String title;
    private String content;

    // 一对多：Article 是聚合根，Comments 是其内部实体
    //  要求 comments 表中有 article 字段作为关联键
    @MappedCollection(idColumn = "article", keyColumn = "comment_idx")
    private Set<Comment> comments = new HashSet<>();

    // 一对一：嵌套值对象
    private ArticleMeta meta;
}

public class Comment {
    private String content;
    private LocalDateTime createTime;
}

public class ArticleMeta {
    private String author;
    private String source;
}
```

**@MappedCollection 详解**：

| 属性 | 说明 | 示例 |
|------|------|------|
| `idColumn` | 子表中引用父表主键的列名（外键列） | `"article"` 表示子表 `article` 列指向父表 `id` |
| `keyColumn` | 集合排序键列名，存储元素在集合中的位置/序号 | `"comment_idx"` 用于保证元素顺序 |

对应的数据库表结构：

```sql
CREATE TABLE article (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    title      VARCHAR(255),
    content    TEXT,
    author     VARCHAR(100),
    source     VARCHAR(100)
);

CREATE TABLE comment (
    article     BIGINT,          -- 外键，关联 article 表
    comment_idx INT,             -- 集合索引
    content     TEXT,
    create_time DATETIME
);
```

`@MappedCollection` 支持的集合类型：`List`、`Set`、`Map`。

**List 集合要求 keyColumn 连续**：

```java
@MappedCollection(idColumn = "order_id", keyColumn = "item_seq")
private List<OrderItem> items;
```

删除 List 中某个元素会导致后续元素的 keyColumn 重新编号。

**Set 集合使用 Map 实现**：

```java
@MappedCollection(idColumn = "order_id", keyColumn = "item_id")
private Set<OrderItem> items;
```

Set 中删除元素不要求 keyColumn 连续，比 List 更灵活。

**Map 集合（键为 keyColumn 值）**：

```java
@MappedCollection(idColumn = "order_id", keyColumn = "item_type")
private Map<String, OrderItem> items;
```

## 4.3 值对象与 @Embedded 注解

从 Spring Data JDBC 2.0 开始，支持使用 `@Embedded` 将嵌套对象的字段扁平化到父表中：

```java
public class User {
    @Id
    private Long id;
    private String name;

    @Embedded(onEmpty = Embedded.OnEmpty.USE_NULL)
    private Address address;
}

public class Address {
    private String province;
    private String city;
    private String street;
}
```

使用 `@Embedded` 后，`address` 的字段会直接映射到 `user` 表的 `province`、`city`、`street` 列上，而非单独建表。

`@Embedded` 属性：
- `onEmpty`：当所有嵌套字段为 null 时的处理策略
  - `USE_NULL`：写入 null
  - `USE_EMPTY`：使用缺省值（由 `prefix` 配合使用）

## 4.4 自定义列映射与 NamingStrategy

```java
import org.springframework.data.relational.core.mapping.NamingStrategy;
import org.springframework.data.relational.core.mapping.RelationalPersistentProperty;

@Component
public class CustomNamingStrategy implements NamingStrategy {

    @Override
    public String getTableName(Class<?> type) {
        // 自定义表名规则
        return "t_" + NamingStrategy.super.getTableName(type);
    }

    @Override
    public String getColumnName(RelationalPersistentProperty property) {
        // 自定义列名规则：全部大写
        return NamingStrategy.super.getColumnName(property).toUpperCase();
    }
}
```

# 5. 数据操作详解

## 5.1 Save 的内部机制

Spring Data JDBC 的 save 方法不是简单的 INSERT/UPDATE：

```
save(aggregateRoot) 执行流程：

1. 判断聚合根是否为新实体（@Id 是否为 null/默认值）
2. 如果是新实体 → 执行 INSERT
3. 如果非新实体：
   a. 从数据库加载当前聚合数据
   b. 比较新旧数据差异
   c. 根据差异生成 INSERT、UPDATE、DELETE 语句
4. 递归处理聚合内的嵌套实体
5. 所有操作在同一事务中执行完成
```

**关键区别**：Spring Data JDBC 不是比较单个字段的脏数据，而是比较整个实体对象的差异，决定新增、更新还是删除。

**示例：更新聚合时对嵌套集合的操作**：

```java
// 假设 Order 已存在，包含 items: [item1, item2, item3]
Order order = orderRepository.findById(1L).orElseThrow();

// 修改 items 集合：删除 item2，新增 item4
order.getItems().removeIf(item -> item.getProductId().equals(2L));
OrderItem newItem = new OrderItem();
newItem.setProductId(4L);
newItem.setQuantity(10);
order.getItems().add(newItem);

orderRepository.save(order);
// 框架比较旧集合 [1,2,3] 与新集合 [1,3,4]
//   DELETE item WHERE order_id=1 AND item_seq=1  (删除 item2)
//   UPDATE item SET ... WHERE order_id=1 AND item_seq=0  (item1 不变)
//   UPDATE item SET ... WHERE order_id=1 AND item_seq=1  (item3 移到位置1)
//   INSERT item (order_id, item_seq, ...) VALUES (1, 2, ...)  (新增 item4)
```

## 5.2 自定义查询 —— @Query 注解

```java
import org.springframework.data.jdbc.repository.query.Query;
import org.springframework.data.repository.query.Param;

public interface OrderRepository extends CrudRepository<Order, Long> {

    @Query("SELECT * FROM orders WHERE customer_id = :customerId")
    List<Order> findByCustomerId(@Param("customerId") Long customerId);

    @Query("SELECT COUNT(*) FROM orders WHERE customer_id = :customerId")
    long countByCustomerId(@Param("customerId") Long customerId);

    @Query("UPDATE orders SET status = :status WHERE id = :id")
    @Modifying
    void updateStatus(@Param("id") Long id, @Param("status") String status);

    @Query("DELETE FROM orders WHERE status = :status AND create_time < :time")
    @Modifying
    int deleteExpiredOrders(@Param("status") String status, @Param("time") LocalDateTime time);
}
```

`@Modifying` 注解标识方法为修改操作（UPDATE、DELETE），Spring Data JDBC 会将其作为写操作执行。

**Spring Data JDBC 3.x 新增的特性**：
- 支持返回值绑定（如 `returning *`）
- 支持 `@Query` 中的命名参数和 SpEL 表达式

## 5.3 分页与排序

```java
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

public interface ProductRepository extends PagingAndSortingRepository<Product, Long> {

    // 使用 Pageable 分页参数
    Page<Product> findByCategory(String category, Pageable pageable);

    // 使用 Sort 排序参数
    List<Product> findByPriceGreaterThan(BigDecimal price, Sort sort);
}

// 调用示例
@Service
public class ProductService {
    // ...

    public Page<Product> getProductsByPage(String category, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("price").descending());
        return productRepository.findByCategory(category, pageable);
    }
}
```

# 6. 派生查询方法详解

## 6.1 方法命名规则

Spring Data JDBC 支持根据方法名自动生成查询 SQL，规则如下：

```
方法名 = 前缀 + 主体 + 后缀

前缀：find / read / get / query / count / exists / delete / remove
主体：[条件表达式][逻辑操作符 [条件表达式]...]
```

**完整语法**：

```
find...By        — 标准查询
read...By        — 等同于 find...By
get...By         — 等同于 find...By
query...By       — 等同于 find...By
count...By       — 计数查询
exists...By      — 存在性判断
delete...By      — 按条件删除
remove...By      — 等同于 delete...By

...First...      — 限制结果为第一条
...Top...        — 限制结果为前 N 条
...Distinct...   — 去重查询
```

## 6.2 条件关键字

| 关键字 | SQL 片段 | 示例方法名 | 生成的 WHERE 条件 |
|--------|---------|-----------|------------------|
| Is / Equals | `= ?` | `findByNameIs` | `name = ?` |
| Not | `<> ?` | `findByNameNot` | `name <> ?` |
| IsNull | `IS NULL` | `findByNameIsNull` | `name IS NULL` |
| IsNotNull | `IS NOT NULL` | `findByNameIsNotNull` | `name IS NOT NULL` |
| Like | `LIKE ?` | `findByNameLike` | `name LIKE ?` |
| NotLike | `NOT LIKE ?` | `findByNameNotLike` | `name NOT LIKE ?` |
| StartingWith | `LIKE '?%'` | `findByNameStartingWith` | `name LIKE '?%'` |
| EndingWith | `LIKE '%?'` | `findByNameEndingWith` | `name LIKE '%?'` |
| Containing | `LIKE '%?%'` | `findByNameContaining` | `name LIKE '%?%'` |
| NotContaining | `NOT LIKE '%?%'` | `findByNameNotContaining` | `name NOT LIKE '%?%'` |
| LessThan | `< ?` | `findByAgeLessThan` | `age < ?` |
| LessThanEqual | `<= ?` | `findByAgeLessThanEqual` | `age <= ?` |
| GreaterThan | `> ?` | `findByAgeGreaterThan` | `age > ?` |
| GreaterThanEqual | `>= ?` | `findByAgeGreaterThanEqual` | `age >= ?` |
| Between | `BETWEEN ? AND ?` | `findByAgeBetween` | `age BETWEEN ? AND ?` |
| In | `IN (?)` | `findByAgeIn` | `age IN (?)` |
| NotIn | `NOT IN (?)` | `findByAgeNotIn` | `age NOT IN (?)` |
| True | `= true` | `findByActiveTrue` | `active = true` |
| False | `= false` | `findByActiveFalse` | `active = false` |
| IgnoreCase | `UPPER(x) = UPPER(?)` | `findByNameIgnoreCase` | `UPPER(name) = UPPER(?)` |
| OrderBy | `ORDER BY` | `findByNameOrderByAgeDesc` | `... ORDER BY age DESC` |
| After | `> ?` | `findByCreateTimeAfter` | `create_time > ?` |
| Before | `< ?` | `findByCreateTimeBefore` | `create_time < ?` |

## 6.3 逻辑组合

```java
// AND 组合（默认）
List<User> findByUserNameAndEmail(String userName, String email);

// OR 组合
List<User> findByEmailOrUserName(String email, String userName);

// 复杂组合
List<User> findByAgeGreaterThanAndCreateTimeAfterOrEmailContaining(
    int age, LocalDateTime time, String keyword);
```

## 6.4 限制结果数量

```java
// 取前 N 条
List<User> findTop5ByEmailContaining(String keyword);

// 取第一条
Optional<User> findFirstByEmail(String email);

// 配合排序
List<User> findTop3ByOrderByCreateTimeDesc();
```

## 6.5 嵌套属性查询

```java
public class User {
    @Id
    private Long id;
    private Address address;   // @Embedded 或嵌套对象
}

public class Address {
    private String city;
    private String province;
}

// 查询嵌套属性（假定 address 字段映射到同一张表）
List<User> findByAddressCity(String city);
List<User> findByAddressProvinceAndAddressCity(String province, String city);
```

注意：嵌套属性查询仅适用于 `@Embedded` 的嵌入对象（即映射到同一张表）。对于 `@MappedCollection` 的一对多关系，Spring Data JDBC 不支持通过聚合根的直接嵌套属性查询（需要自行编写 @Query）。

# 7. 事务管理

## 7.1 声明式事务

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final JdbcTemplate jdbcTemplate;

    @Transactional
    public void createOrder(Order order) {
        // Spring Data JDBC 的 save 已在事务中操作聚合

        // 多个 Repository 操作也在同一事务中
        orderRepository.save(order);

        // 可以混合使用 JdbcTemplate
        jdbcTemplate.update(
            "UPDATE inventory SET stock = stock - ? WHERE product_id = ?",
            order.getQuantity(), order.getProductId()
        );
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderCreation(Order order) {
        // 独立事务记录日志，不受外部事务回滚影响
        jdbcTemplate.update(
            "INSERT INTO order_log (order_id, event, event_time) VALUES (?, ?, ?)",
            order.getId(), "CREATED", LocalDateTime.now()
        );
    }

    @Transactional(isolation = Isolation.READ_COMMITTED, timeout = 30)
    public Order findByIdWithLock(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}
```

## 7.2 事务传播行为

| 传播行为 | 说明 |
|---------|------|
| REQUIRED（默认） | 当前存在事务则加入，不存在则新建 |
| REQUIRES_NEW | 总是新建事务，挂起当前事务 |
| SUPPORTS | 当前存在事务则加入，不存在则以非事务方式运行 |
| NOT_SUPPORTED | 以非事务方式运行，挂起当前事务 |
| MANDATORY | 必须在事务中运行，无事务则抛异常 |
| NEVER | 必须在非事务中运行，有事务则抛异常 |
| NESTED | 嵌套事务（需要 JDBC savepoint 支持） |

## 7.3 事务失效的常见场景

```java
// 场景1：方法非 public
@Transactional
private void doSomething() { }  // 失效！事务注解在 private 方法上不生效（CGLIB 代理限制）

// 场景2：同类内部调用
public void outerMethod() {
    this.innerMethod();  // 失效！直接 this 调用不走代理
}

@Transactional
public void innerMethod() { }

// 场景3：异常被捕获
@Transactional
public void doSomething() {
    try {
        jdbcTemplate.update("INSERT INTO ...");
        throw new RuntimeException("error");
    } catch (Exception e) {
        // 异常在此被捕获，事务不会回滚！如需回滚，手动设置：
        // TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}

// 场景4：非 RuntimeException 不触发回滚
@Transactional
public void doSomething() throws Exception {
    throw new Exception("checked exception");  // 默认不回滚！
}

// 解决方法：指定 rollbackFor
@Transactional(rollbackFor = Exception.class)
public void doSomething() throws Exception { }
```

# 8. 生命周期事件

## 8.1 支持的聚合根事件

Spring Data JDBC 提供了聚合根级别的生命周期回调：

```java
import org.springframework.data.relational.core.mapping.event.*;

@Component
public class UserEventListener extends AbstractRelationalEventListener<User> {

    @Override
    protected void onBeforeSave(BeforeSaveEvent<User> event) {
        User user = event.getEntity();
        System.out.println("保存前：" + user);
    }

    @Override
    protected void onAfterSave(AfterSaveEvent<User> event) {
        User user = event.getEntity();
        System.out.println("保存后（ID已生成）：" + user.getId());
    }

    @Override
    protected void onBeforeDelete(BeforeDeleteEvent<User> event) {
        System.out.println("删除前");
    }

    @Override
    protected void onAfterDelete(AfterDeleteEvent<User> event) {
        System.out.println("删除后");
    }
}
```

完整事件列表：

| 事件类型 | 触发时机 | 说明 |
|---------|---------|------|
| `BeforeConvertEvent` | 实体转换为数据库行之前 | 可用于修改实体属性 |
| `AfterConvertEvent` | 数据库行转换为实体之后 | 实体已构建完成 |
| `BeforeSaveEvent` | 执行 INSERT/UPDATE 之前 | 可用于设置 timestamps |
| `AfterSaveEvent` | 执行 INSERT/UPDATE 之后 | 实体 ID 已赋值 |
| `BeforeDeleteEvent` | 执行 DELETE 之前 | — |
| `AfterDeleteEvent` | 执行 DELETE 之后 | — |
| `AfterLoadEvent` | 从数据库加载实体之后 | — |

## 8.2 BeforeConvert 回调接口

```java
import org.springframework.data.relational.core.mapping.event.BeforeConvertCallback;

@Component
public class UserBeforeConvertCallback implements BeforeConvertCallback<User> {

    @Override
    public User onBeforeConvert(User user) {
        if (user.getCreateTime() == null) {
            user.setCreateTime(LocalDateTime.now());
        }
        user.setUpdateTime(LocalDateTime.now());
        return user;
    }
}
```

## 8.3 在实体内部实现回调接口

```java
import org.springframework.data.relational.core.mapping.event.BeforeConvertCallback;
import org.springframework.data.relational.core.mapping.event.BeforeSaveCallback;

public class User implements BeforeConvertCallback<User>, BeforeSaveCallback<User> {

    @Override
    public User onBeforeConvert(User aggregate) {
        // 在转换为数据库行之前
        return aggregate;
    }

    @Override
    public User onBeforeSave(User aggregate, MutableAggregateChange<User> change) {
        // 在保存之前，可以检查变更
        return aggregate;
    }
}
```

# 9. 源码与底层机制解析

## 9.1 整体架构

```
┌──────────────────────────────────────────────────────┐
│                   Application Code                    │
│          (UserRepository, UserService, ...)           │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│              Spring Data JDBC 框架层                   │
│  ┌─────────────────────────────────────────────────┐ │
│  │         Repository Proxy (JDK Proxy)             │ │
│  │  拦截 Repository 方法调用，分发到具体执行逻辑       │ │
│  └───────────────┬─────────────────────────────────┘ │
│                  │                                    │
│  ┌───────────────▼─────────────────────────────────┐ │
│  │         JdbcAggregateTemplate                    │ │
│  │  聚合级别的 CRUD 操作模板                          │ │
│  │  - insert(aggregate)                             │ │
│  │  - update(aggregate)                             │ │
│  │  - delete(id, type)                              │ │
│  │  - findById(id, type)                            │ │
│  └───────────────┬─────────────────────────────────┘ │
│                  │                                    │
│  ┌───────────────▼─────────────────────────────────┐ │
│  │         AggregateChange 变更检测                  │ │
│  │  比较新旧聚合差异，生成变更集                       │ │
│  └───────────────┬─────────────────────────────────┘ │
│                  │                                    │
│  ┌───────────────▼─────────────────────────────────┐ │
│  │         SqlGenerator                             │ │
│  │  根据变更集生成具体 SQL 语句                       │ │
│  │  (SimpleJdbcRepository / DefaultJdbcRepository)   │ │
│  └───────────────┬─────────────────────────────────┘ │
│                  │                                    │
│  ┌───────────────▼─────────────────────────────────┐ │
│  │         DataAccessStrategy                      │ │
│  │  数据访问策略，负责数据库交互                       │ │
│  └─────────────────────────────────────────────────┘ │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│              Spring Framework 核心层                   │
│  ┌───────────────┐  ┌─────────────┐  ┌────────────┐  │
│  │  JdbcTemplate │  │ NamedParam.. │  │  TxManager │  │
│  └───────────────┘  └─────────────┘  └────────────┘  │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│                 JDBC Driver / Database                │
└──────────────────────────────────────────────────────┘
```

## 9.2 Repository 代理创建机制

Spring Data JDBC 在启动时通过 `@EnableJdbcRepositories`（或自动配置）扫描所有 Repository 接口，为每个接口创建 JDK 动态代理：

```
@EnableJdbcRepositories 触发
  → JdbcRepositoryConfigExtension
    → JdbcRepositoryFactoryBean
      → JdbcRepositoryFactory
        → 获取 Repository 接口的 Class 信息
        → 解析方法签名（派生查询 or @Query）
        → 生成 JdkDynamicAopProxy
        → 注册为 Spring Bean
```

每个 Repository 方法调用流程：

```
userRepository.findByEmail(email)
  → JdkDynamicAopProxy.invoke()
    → RepositoryMethodInvoker
      → QueryExecutorMethodInterceptor
        → 解析方法：findByEmail → SELECT * FROM user WHERE email = ?
        → JdbcTemplate.query(sql, rowMapper, email)
        → 返回结果
```

## 9.3 SimpleJdbcRepository 实现

`SimpleJdbcRepository` 是 `CrudRepository` 接口的默认实现：

```java
// 简化示意（非实际源码）
public class SimpleJdbcRepository<T, ID> implements CrudRepository<T, ID> {

    private final JdbcAggregateOperations entityOperations;
    private final RelationalPersistentEntity<T> entity;
    private final JdbcConverter converter;

    @Override
    public <S extends T> S save(S instance) {
        // 1. 判断是新实体还是已存在实体
        Assert.notNull(instance, "Aggregate must not be null");

        // 2. 如果是新实体（ID 为空），调用 insert
        if (entity.isNew(instance)) {
            return insert(instance);
        }

        // 3. 已存在实体：先从数据库加载旧值，比较差异
        //    然后生成 INSERT、UPDATE、DELETE 语句
        return update(instance);
    }

    @Override
    public Optional<T> findById(ID id) {
        // 构建 SQL: SELECT ... FROM table WHERE id = ?
        // 执行查询，返回聚合根（包括嵌套实体）
    }

    // ...
}
```

## 9.4 SQL 生成流程

Spring Data JDBC 不使用运行时反射动态生成 SQL，而是在启动时预解析 SQL 并缓存：

```java
// 启动时
1. 扫描 Repository 方法
2. 解析方法签名，构建 SqlQuery 对象
3. 缓存 SqlQuery → method 的映射

// 运行时
1. 查找缓存的 SqlQuery
2. 绑定参数值
3. 通过 JdbcTemplate 执行
4. 使用 RowMapper 映射结果
```

**INSERT 语句生成逻辑**：

```java
// 简化逻辑
public String getInsert(RelationalPersistentEntity<?> entity) {
    // 遍历所有非 ID 列
    // 构建：INSERT INTO table (col1, col2, ...) VALUES (?, ?, ...)
    // ID 字段如为自增则不包含在 VALUES 中
}
```

**UPDATE 语句生成逻辑**：

```java
// 简化逻辑
public String getUpdate(RelationalPersistentEntity<?> entity) {
    // 遍历所有列（包括 ID 作为 WHERE 条件）
    // 构建：UPDATE table SET col1 = ?, col2 = ? WHERE id = ?
}
```

# 10. MyBatis 整合

## 10.1 场景说明

Spring Data JDBC 适用于简单的 CRUD 操作和聚合管理，但当遇到复杂的动态查询、多表关联查询时，可以配合 MyBatis 使用。

**分工建议**：
- Spring Data JDBC：处理聚合根的 CRUD、简单查询
- MyBatis：处理复杂查询、报表统计、动态条件查询

## 10.2 Maven 依赖

```xml

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

## 10.3 分工示例

```java
// Spring Data JDBC 负责聚合根操作
public interface OrderRepository extends CrudRepository<Order, Long> {
    List<Order> findByCustomerId(Long customerId);
}

// MyBatis Mapper 负责复杂查询
@Mapper
public interface OrderQueryMapper {
    List<OrderSummaryDTO> findTopOrders(@Param("customerId") Long customerId,
                                         @Param("startDate") LocalDate startDate,
                                         @Param("endDate") LocalDate endDate);
}
```

# 11. 最佳实践

## 11.1 聚合设计原则

1. **设计小聚合**：聚合内部实体不宜过多，尽量控制在 3-5 个实体以内
2. **跨聚合引用只存 ID**：只存储 `customerId` 而非 `Customer` 对象
3. **聚合是一致性边界**：一个事务只应修改一个聚合
4. **优先使用 Set 而非 List**：Set 的删除操作不会引起索引重排
5. **慎用深层嵌套**：不要超过两层嵌套（聚合根 → 内部实体 → 值对象）

## 11.2 性能优化建议

```
1. 避免加载不需要的聚合
   - 对于只需部分数据的查询，使用 @Query 返回投影而非完整聚合

2. 批量操作使用 saveAll
   - userRepository.saveAll(userList) 利用批处理提升性能

3. 控制聚合加载深度
   - 避免设计过大的聚合导致加载过多无关数据

4. 合理使用分页
   - findAll(Pageable) 添加 LIMIT/OFFSET 避免全表扫描

5. 数据库索引
   - 为派生查询的条件字段和关联字段创建索引
```

## 11.3 常见问题与解决方案

**问题1：save 时嵌套实体被意外删除再新增**

原因：集合元素使用 List 类型，keyColumn 不连续导致框架认为元素已删除。

解决：优先使用 Set 替代 List，或维护 keyColumn 的连续性。

**问题2：ID 自动生成后无法获取**

Spring Data JDBC 的 save 方法返回的实体已包含生成的 ID：

```java
User saved = userRepository.save(newUser);
Long generatedId = saved.getId();  // 已包含生成的 ID
```

**问题3：派生查询方法名过长**

过于复杂的方法名应改用 `@Query` 注解：

```java
// 不推荐：方法名过长
List<Order> findByCustomerIdAndStatusAndCreateTimeBetweenOrderByCreateTimeDesc(
    Long customerId, String status, LocalDateTime start, LocalDateTime end);

// 推荐：使用 @Query
@Query("SELECT * FROM orders WHERE customer_id = :customerId " +
       "AND status = :status AND create_time BETWEEN :start AND :end " +
       "ORDER BY create_time DESC")
List<Order> findOrders(@Param("customerId") Long customerId,
                        @Param("status") String status,
                        @Param("start") LocalDateTime start,
                        @Param("end") LocalDateTime end);
```

## 11.4 面试常见考点

1. **Spring Data JDBC 与 Spring Data JPA 的本质区别？**
   - JDBC 无懒加载、无脏检查、无缓存、无 EntityManager
   - JDBC 基于聚合概念，JPA 基于完整的 ORM 映射

2. **为什么 Spring Data JDBC 没有懒加载？**
   - 设计哲学：聚合应按需一次性加载，避免运行时代理带来的复杂性和 N+1 问题

3. **Spring Data JDBC 如何判断 insert 还是 update？**
   - 通过 `@Id` 字段的值判断：null/默认值为 insert，非空为 update

4. **如何实现复杂多表联合查询？**
   - 使用 `@Query` 手写 SQL
   - 或整合 MyBatis 处理复杂查询场景

# 12. 版本演进与新特性

| 版本 | Spring Boot 版本 | 主要特性 |
|------|-----------------|---------|
| 1.0 | 2.0 | 初始版本，基础 CRUD 支持 |
| 1.1 | 2.1 | 派生查询增强，@Query 支持 |
| 2.0 | 2.4 | @Embedded 支持，NamingStrategy 改进 |
| 2.1 | 2.4 | 引用加载优化 |
| 2.2 | 2.5 | 生命周期回调增强 |
| 2.3 | 2.6 | ID 生成策略扩展 |
| 2.4 | 2.7 | 批量操作优化 |
| 3.0 | 3.0 | Jakarta EE 迁移，Java 17 基线 |
| 3.1 | 3.1 | 更多 SQL 方言支持 |
| 3.2 | 3.2 | 查询增强，返回集合参数绑定 |
| 3.3 | 3.3 | 当前最新版本 |

# 13. H2/HSQLDB 集成测试

## 13.1 测试依赖

```xml

<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

## 13.2 测试配置

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=MySQL
    driver-class-name: org.h2.Driver
  sql:
    init:
      mode: always
      schema-locations: classpath:test-schema.sql
```

## 13.3 Repository 集成测试

```java
@DataJdbcTest
@Sql(scripts = "classpath:test-data.sql")
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFindUser() {
        User user = new User();
        user.setUserName("测试用户");
        user.setEmail("test@example.com");

        User saved = userRepository.save(user);
        assertThat(saved.getId()).isNotNull();

        Optional<User> found = userRepository.findById(saved.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getEmail()).isEqualTo("test@example.com");
    }

    @Test
    void shouldFindByEmailContaining() {
        List<User> users = userRepository.findByEmailContaining("example");
        assertThat(users).isNotEmpty();
    }
}
```
