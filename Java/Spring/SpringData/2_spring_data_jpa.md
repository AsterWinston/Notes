# 1. JPA 概述与核心概念

## 1.1 JPA、Hibernate 与 Spring Data JPA 的关系

理解这三者的关系是学习 Spring Data JPA 的基础：

```
Java EE / Jakarta EE 规范层：
  ┌──────────────────────────────┐
  │   JPA (Jakarta Persistence)   │  ← 接口规范（javax.persistence / jakarta.persistence）
  └──────────────┬───────────────┘
                 │ 实现
  ┌──────────────▼───────────────┐
  │   Hibernate                   │  ← JPA 的最主流实现
  │   EclipseLink                 │  ← 另一个 JPA 实现
  └──────────────┬───────────────┘
                 │ 封装简化
  ┌──────────────▼───────────────┐
  │   Spring Data JPA             │  ← Spring 对 JPA 的进一步封装
  │   (Repository 抽象层)          │     简化 DAO 层开发
  └──────────────────────────────┘
```

- **JPA**：是一套 ORM 规范，定义了注解、接口和 JPQL 查询语言，但并不提供具体实现
- **Hibernate**：是 JPA 规范最主流的实现，还提供了 JPA 规范之外的高级特性（如 HQL、多租户等）
- **Spring Data JPA**：是 Spring 对 JPA 的进一步封装，核心是 Repository 抽象，通过接口方法名自动生成查询

## 1.2 ORM 思想与核心问题

ORM（Object-Relational Mapping）解决的核心问题是面向对象编程与关系数据库之间的"阻抗失配"：

| 面向对象世界 | 关系数据库世界 | 映射方案 |
|------------|-------------|---------|
| 类（Class） | 表（Table） | @Entity + @Table |
| 对象（Object） | 行（Row） | 实体实例 |
| 属性（Field） | 列（Column） | @Column |
| 继承（Inheritance） | 无此概念 | 继承策略映射 |
| 关联（Association） | 外键（Foreign Key） | @OneToMany / @ManyToOne 等 |
| 多态（Polymorphism） | 无此概念 | 鉴别器列 / 联合表 |

## 1.3 为什么要使用 JPA

```
优势：
- 以面向对象方式操作数据库，开发效率极高
- 屏蔽不同数据库的 SQL 方言差异（可移植性）
- 自动生成 DDL（建表语句），适合快速原型开发
- 一级缓存减少数据库访问
- 脏检查机制自动执行 UPDATE，无需手动调用 save
- 延迟加载优化大对象加载性能

劣势与挑战：
- 学习曲线陡峭（持久化上下文、实体状态、级联操作等概念较多）
- 复杂查询不如手写 SQL 灵活
- 脏检查和缓存可能导致意外的 SQL 执行
- N+1 查询问题是常见陷阱
- 对 DBA 不友好（SQL 不可见、执行计划不确定）
```

## 1.4 JPA 核心对象

| 对象 | 说明 |
|------|------|
| **EntityManagerFactory** | 线程安全，通常应用级别单例，负责创建 EntityManager |
| **EntityManager** | 线程不安全，每个事务/请求创建新的，管理持久化上下文 |
| **EntityTransaction** | 事务管理对象（在 Spring 中由 @Transactional 统一管理） |
| **PersistenceContext** | 持久化上下文，管理着一组实体实例的集合（一级缓存） |
| **PersistenceUnit** | 持久化单元，定义了一组实体类与数据库的映射关系 |

# 2. 快速入门

## 2.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

## 2.2 基础配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/jpa_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf-8
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: update        # 开发阶段用 update，生产环境用 validate 或 none
    show-sql: true            # 控制台打印 SQL
    properties:
      hibernate:
        format_sql: true      # 格式化 SQL
        dialect: org.hibernate.dialect.MySQLDialect
```

**ddl-auto 选项说明**：

| 值 | 说明 | 适用场景 |
|----|------|---------|
| `none` | 不自动处理 DDL | 生产环境 |
| `validate` | 只验证实体与表结构是否匹配，不修改 | 生产环境（安全） |
| `update` | 不存在的表/列自动创建，已有的不删除 | 开发环境 |
| `create` | 每次启动删除重建表 | 开发测试 |
| `create-drop` | 启动时建表，关闭时删表 | 单元测试 |

## 2.3 实体类与 Repository

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "t_user")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_name", length = 50, nullable = false)
    private String userName;

    @Column(unique = true)
    private String email;

    @Column(name = "create_time")
    private LocalDateTime createTime;

    public User() {}
}
```

```java
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    List<User> findByUserNameContaining(String keyword);

    boolean existsByEmail(String email);

    long countByCreateTimeAfter(LocalDateTime time);
}
```

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional
    public User createUser(User user) {
        user.setCreateTime(LocalDateTime.now());
        return userRepository.save(user);
    }

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("用户不存在"));
    }
}
```

# 3. 实体映射详解

## 3.1 基础注解

```java
@Entity
@Table(name = "t_product",
       indexes = {@Index(name = "idx_category", columnList = "category"),
                  @Index(name = "idx_name_price", columnList = "name, price")},
       uniqueConstraints = {@UniqueConstraint(columnNames = {"code"})})
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name", length = 100, nullable = false)
    private String name;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @Lob
    private String description;

    @Enumerated(EnumType.STRING)
    private ProductStatus status;

    @Transient
    private String tempField;

    @Temporal(TemporalType.TIMESTAMP)
    private Date legacyDate;
}
```

## 3.2 主键生成策略

```java
// 策略1：数据库自增（MySQL 推荐）
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// 策略2：序列（PostgreSQL / Oracle）
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq_gen")
@SequenceGenerator(name = "seq_gen", sequenceName = "product_seq", allocationSize = 50)
private Long id;

// 策略3：表生成器（兼容性好，性能较差）
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "table_gen")
@TableGenerator(name = "table_gen", table = "id_generator",
                pkColumnName = "gen_name", valueColumnName = "gen_value",
                pkColumnValue = "product_id", allocationSize = 100)
private Long id;

// 策略4：UUID
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private String id;
```

| 策略 | 优点 | 缺点 | 适用数据库 |
|------|------|------|-----------|
| IDENTITY | 简单、无额外表/序列 | 批量插入性能差 | MySQL |
| SEQUENCE | 性能好、支持批量 | 需要数据库支持序列 | PostgreSQL、Oracle |
| TABLE | 跨数据库兼容 | 性能最差、有锁竞争 | 通用 |
| UUID | 分布式友好 | 字符串占用大、索引效率低 | 通用 |

## 3.3 关联关系映射

### 3.3.1 一对一（@OneToOne）

```java
// 主表
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, orphanRemoval = true)
    private UserProfile profile;
}

// 从表（拥有外键的一方）
@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String avatar;
    private String bio;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;
}
```

关键概念：
- **拥有方（Owning Side）**：持有外键列的一方，即 `@JoinColumn` 所在方
- **被拥有方（Inverse Side）**：使用 `mappedBy` 的一方，不持有外键
- **只有拥有方的变更会影响数据库中的外键**

### 3.3.2 一对多 / 多对一（@OneToMany / @ManyToOne）

```java
// 多的一方（拥有方）
@Entity
@Table(name = "t_order_item")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String productName;
    private int quantity;
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
}

// 一的一方（被拥有方）
@Entity
@Table(name = "t_order")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String orderNumber;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    // 辅助方法：维护双向关联
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}
```

默认加载策略：
- `@OneToMany` 默认 `FetchType.LAZY`（延迟加载）
- `@ManyToOne` 默认 `FetchType.EAGER`（立即加载）
- **最佳实践：大多数关联都应使用 LAZY 加载**

### 3.3.3 多对多（@ManyToMany）

```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(name = "student_course",
               joinColumns = @JoinColumn(name = "student_id"),
               inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

推荐做法：将多对多拆为两个一对多，用一个显式的中间实体：

```java
@Entity
@Table(name = "student_course")
public class StudentCourse {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    private Course course;

    private LocalDateTime enrollTime;
    private Integer score;
}
```

### 3.3.4 级联操作（Cascade）

| 级联类型 | 说明 | 何时使用 |
|---------|------|---------|
| `PERSIST` | 保存父实体时，同时保存子实体 | @OneToMany 中很常用 |
| `MERGE` | 合并父实体时，同时合并子实体 | 与 PERSIST 配合使用 |
| `REMOVE` | 删除父实体时，同时删除子实体 | 谨慎使用，配合 orphanRemoval |
| `REFRESH` | 刷新父实体时，同时刷新子实体 | 不常用 |
| `DETACH` | 分离父实体时，同时分离子实体 | 不常用 |
| `ALL` | 包含以上所有级联 | 确认完整拥有子实体生命周期时 |

**orphanRemoval（孤儿删除）**：当子实体从父实体的集合中移除时，自动删除子实体。与 `REMOVE` 不同：REMOVE 只在删除父实体时触发，orphanRemoval 在解除关联时触发。

## 3.4 继承策略映射

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "dtype")
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String brand;
}

@Entity
@DiscriminatorValue("CAR")
public class Car extends Vehicle {
    private int seatCount;
}

@Entity
@DiscriminatorValue("BIKE")
public class Bike extends Vehicle {
    private boolean hasBasket;
}
```

| 策略 | 数据库结构 | 优点 | 缺点 |
|------|-----------|------|------|
| `SINGLE_TABLE` | 一张大表 + 鉴别器列 | 性能最好（无需 JOIN） | 大量 NULL 列 |
| `JOINED` | 父表 + 每个子类独立表 | 结构清晰，符合范式 | 查询需要 JOIN |
| `TABLE_PER_CLASS` | 每个具体子类一张完整表 | 无 NULL 列 | 多态查询需 UNION |

# 4. 持久化上下文与实体状态

## 4.1 持久化上下文

持久化上下文本质上是一个 Map<ID, Entity>，在同一个 EntityManager 生命周期内，相同 ID 的实体只有一个实例：

```
用户请求开始
  → EntityManager 创建（@Transactional 范围）
    → 查询 User(id=1) → 从数据库加载 → 放入 PersistenceContext
    → 再查询 User(id=1) → 从 PersistenceContext 直接返回（不查库）
    → 修改 user.setName("新名字") → 标记为 dirty
    → 事务提交 → flush → 自动生成 UPDATE SQL → 提交
  → EntityManager 关闭
```

核心特性：
- **一级缓存**：同一 EntityManager 内，相同 ID 的查询只访问一次数据库
- **脏检查（Dirty Checking）**：事务提交时自动比较实体快照，生成 UPDATE 语句
- **写后延迟（Write-Behind）**：INSERT/UPDATE/DELETE 延迟到事务提交时批量执行
- **身份保证**：同一 PC 中，相同 ID 的实体引用相同（user1 == user2）

## 4.2 实体的四种状态

```java
// 1. Transient（瞬时态）：刚 new 出来，与数据库和 PC 都无关联
User user = new User();
user.setName("张三");

// 2. Managed / Persistent（持久态）：被 PersistenceContext 管理
entityManager.persist(user); // 或通过 find/merge 获取
// 此时对 user 的任何修改，在 flush 时都会自动同步到数据库

// 3. Detached（游离态）：曾经是持久态，但 PC 已关闭
entityManager.detach(user);  // 或 PC 关闭后
// 对 user 的修改不会自动同步到数据库

// 4. Removed（移除态）：计划从数据库中删除
entityManager.remove(user);
// 事务提交时生成 DELETE 语句
```

状态转换：

```
         new / find
   Transient ──────────→ Managed ←── merge ─── Detached
                            │                       ↑
                            │ remove                │ detach / close
                            ▼                       │
                         Removed ───────────────────┘
```

## 4.3 Flush 机制

Flush 是将 PersistenceContext 中的变更同步到数据库的过程：

```java
// flush 触发时机：
// 1. 事务提交前（自动）
// 2. 显式调用 entityManager.flush()
// 3. 执行 JPQL/HQL 查询前（取决于 FlushModeType）

// FlushModeType：
// - AUTO（默认）：在查询前自动 flush
// - COMMIT：只在事务提交时 flush
```

flush 执行顺序（重要）：
```
1. 所有 INSERT
2. 所有 UPDATE
3. 集合元素删除
4. 集合元素更新
5. 集合重建
6. 所有 DELETE
```

# 5. Repository 详解

## 5.1 Repository 接口层级

```
Repository<T, ID>                          标记接口
 └── CrudRepository<T, ID>                 基本 CRUD
      └── PagingAndSortingRepository<T, ID> 分页排序
           └── JpaRepository<T, ID>         JPA 扩展功能
                └── JpaSpecificationExecutor<T> 动态查询
```

**JpaRepository 新增的关键方法**：

```java
List<T> findAll();
List<T> findAllById(Iterable<ID> ids);
<S extends T> List<S> saveAll(Iterable<S> entities);
void flush();
<S extends T> S saveAndFlush(S entity);
void deleteAllInBatch();
void deleteAllByIdInBatch(Iterable<ID> ids);
T getReferenceById(ID id);  // 懒加载代理，不立即查库
```

## 5.2 派生查询方法

Spring Data JPA 解析方法名生成 JPQL 查询，这是其最核心的特性之一：

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    List<User> findByUserNameAndEmail(String userName, String email);

    List<User> findByAgeBetween(int min, int max);

    List<User> findByCreateTimeAfter(LocalDateTime time);

    List<User> findByAgeIn(Collection<Integer> ages);

    List<User> findByUserNameLike(String pattern);           // 需手动拼接 %

    List<User> findByUserNameStartingWith(String prefix);

    List<User> findByUserNameEndingWith(String suffix);

    List<User> findByUserNameContaining(String keyword);

    List<User> findByUserNameIgnoreCase(String userName);

    List<User> findByEmailIsNull();

    List<User> findByActiveTrue();

    List<User> findByAgeGreaterThanOrderByCreateTimeDesc(int age);

    User findFirstByOrderByCreateTimeDesc();

    List<User> findTop5ByAgeGreaterThan(int age);

    List<User> findByAddressCity(String city);  // 嵌套属性查询
}
```

**完整关键字对照表**：

| 关键字 | SQL 片段 | 示例方法名 |
|--------|---------|-----------|
| Is / Equals | `= ?` | `findByNameIs` |
| Not | `<> ?` | `findByNameNot` |
| IsNull | `IS NULL` | `findByNameIsNull` |
| IsNotNull | `IS NOT NULL` | `findByNameIsNotNull` |
| Like | `LIKE ?` | `findByNameLike` |
| NotLike | `NOT LIKE ?` | `findByNameNotLike` |
| StartingWith | `LIKE '?%'` | `findByNameStartingWith` |
| EndingWith | `LIKE '%?'` | `findByNameEndingWith` |
| Containing | `LIKE '%?%'` | `findByNameContaining` |
| NotContaining | `NOT LIKE '%?%'` | `findByNameNotContaining` |
| LessThan | `< ?` | `findByAgeLessThan` |
| LessThanEqual | `<= ?` | `findByAgeLessThanEqual` |
| GreaterThan | `> ?` | `findByAgeGreaterThan` |
| GreaterThanEqual | `>= ?` | `findByAgeGreaterThanEqual` |
| Between | `BETWEEN ? AND ?` | `findByAgeBetween` |
| In | `IN (?)` | `findByAgeIn` |
| NotIn | `NOT IN (?)` | `findByAgeNotIn` |
| True | `= true` | `findByActiveTrue` |
| False | `= false` | `findByActiveFalse` |
| IgnoreCase | `UPPER(x) = UPPER(?)` | `findByNameIgnoreCase` |
| OrderBy | `ORDER BY` | `findByNameOrderByAgeDesc` |
| After | `> ?` | `findByCreateTimeAfter` |
| Before | `< ?` | `findByCreateTimeBefore` |
| Distinct | `DISTINCT` | `findDistinctByName` |
| First / Top | `LIMIT` | `findFirst5ByAge` |

## 5.3 @Query 注解 —— JPQL 与原生 SQL

### 5.3.1 JPQL 查询

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p WHERE p.price > :minPrice AND p.category = :category")
    List<Product> findExpensiveProducts(@Param("minPrice") BigDecimal minPrice,
                                         @Param("category") String category);

    @Query("SELECT p FROM Product p WHERE p.name LIKE %:keyword%")
    List<Product> searchByName(@Param("keyword") String keyword);

    @Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.id = :id")
    Optional<Product> findByIdWithCategory(@Param("id") Long id);

    // 自定义返回对象
    @Query("SELECT new com.example.dto.ProductSummary(p.name, p.price) " +
           "FROM Product p WHERE p.category = :category")
    List<ProductSummary> findSummariesByCategory(@Param("category") String category);
}
```

### 5.3.2 更新与删除

```java
@Modifying
@Query("UPDATE Product p SET p.price = :price WHERE p.id = :id")
int updatePrice(@Param("id") Long id, @Param("price") BigDecimal price);

@Modifying
@Query("DELETE FROM Product p WHERE p.status = :status AND p.createTime < :time")
int deleteOldProducts(@Param("status") String status, @Param("time") LocalDateTime time);

// @Modifying 注意事项：
// 1. 默认不自动 flush，需手动调用 repository.flush() 或设置 flushAutomatically = true
// 2. 执行后不自动 clear，实体可能变为过期状态，设置 clearAutomatically = true
// 3. 返回 int 为影响行数
```

### 5.3.3 原生 SQL

```java
@Query(value = "SELECT * FROM t_product WHERE MATCH(name, description) AGAINST(:keyword IN NATURAL LANGUAGE MODE)",
       nativeQuery = true)
List<Product> fullTextSearch(@Param("keyword") String keyword);

@Modifying
@Query(value = "INSERT INTO t_product_log (product_id, change_type, change_time) " +
               "VALUES (:productId, :type, NOW())",
       nativeQuery = true)
void insertLog(@Param("productId") Long productId, @Param("type") String type);
```

## 5.4 分页与排序

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    Page<Product> findByCategory(String category, Pageable pageable);

    Slice<Product> findByNameContaining(String name, Pageable pageable);  // 不查总数

    List<Product> findByPriceGreaterThan(BigDecimal price, Sort sort);
}

// 使用示例
@Service
public class ProductService {

    public Page<Product> getProducts(int page, int size, String category) {
        Pageable pageable = PageRequest.of(
            page, size,
            Sort.by(Sort.Direction.DESC, "createTime")
                .and(Sort.by("price").ascending())
        );
        Page<Product> result = productRepository.findByCategory(category, pageable);

        // result.getTotalElements()  // 总记录数
        // result.getTotalPages()     // 总页数
        // result.getNumber()         // 当前页码
        // result.hasNext()           // 是否有下一页
        // result.getContent()        // 当前页数据列表
        return result;
    }
}
```

## 5.5 Projection（投影查询）

```java
// 基于接口的投影
public interface ProductSummary {
    String getName();
    BigDecimal getPrice();
    CategoryInfo getCategory();

    interface CategoryInfo {
        String getName();
    }
}

// 基于 record 的投影（推荐，Java 14+）
public record ProductDto(String name, BigDecimal price, String categoryName) {}

// Repository 使用
@Query("SELECT new com.example.dto.ProductDto(p.name, p.price, c.name) " +
       "FROM Product p JOIN p.category c WHERE p.id = :id")
ProductDto findDtoById(@Param("id") Long id);

// 动态投影
<T> List<T> findByCategory(String category, Class<T> type);
// 调用：productRepository.findByCategory("电子", ProductSummary.class);
```

# 6. JPA 高级查询

## 6.1 Specification（动态查询）

当查询条件不确定、需要动态拼接 WHERE 条件时使用：

```java
// Repository 需额外继承 JpaSpecificationExecutor
public interface ProductRepository extends JpaRepository<Product, Long>,
        JpaSpecificationExecutor<Product> {
}

@Service
public class ProductService {

    public Page<Product> searchProducts(String name, BigDecimal minPrice,
                                         BigDecimal maxPrice, String category,
                                         Pageable pageable) {
        Specification<Product> spec = (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            if (name != null && !name.isEmpty()) {
                predicates.add(cb.like(root.get("name"), "%" + name + "%"));
            }
            if (minPrice != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("price"), minPrice));
            }
            if (maxPrice != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get("price"), maxPrice));
            }
            if (category != null && !category.isEmpty()) {
                predicates.add(cb.equal(root.get("category").get("name"), category));
            }

            return cb.and(predicates.toArray(new Predicate[0]));
        };

        return productRepository.findAll(spec, pageable);
    }
}
```

封装可复用的 Specification：

```java
public class ProductSpecifications {

    public static Specification<Product> hasCategory(String category) {
        return (root, query, cb) ->
            category == null ? null : cb.equal(root.get("category").get("name"), category);
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (min != null) predicates.add(cb.ge(root.get("price"), min));
            if (max != null) predicates.add(cb.le(root.get("price"), max));
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }

    public static Specification<Product> nameContains(String keyword) {
        return (root, query, cb) ->
            keyword == null ? null : cb.like(root.get("name"), "%" + keyword + "%");
    }
}

// 组合使用
Specification<Product> spec = Specification
    .where(ProductSpecifications.hasCategory("电子"))
    .and(ProductSpecifications.priceBetween(new BigDecimal("100"), new BigDecimal("1000")))
    .and(ProductSpecifications.nameContains("手机"));
productRepository.findAll(spec);
```

## 6.2 Querydsl 集成

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
    <classifier>jakarta</classifier>
</dependency>
```

```java
@Service
public class ProductQueryService {

    @PersistenceContext
    private EntityManager em;

    public List<Product> search(String name, BigDecimal minPrice, BigDecimal maxPrice) {
        JPAQueryFactory queryFactory = new JPAQueryFactory(em);
        QProduct product = QProduct.product;

        BooleanBuilder builder = new BooleanBuilder();

        if (name != null && !name.isEmpty()) {
            builder.and(product.name.containsIgnoreCase(name));
        }
        if (minPrice != null) {
            builder.and(product.price.goe(minPrice));
        }
        if (maxPrice != null) {
            builder.and(product.price.loe(maxPrice));
        }

        return queryFactory
            .selectFrom(product)
            .where(builder)
            .orderBy(product.createTime.desc())
            .fetch();
    }
}
```

# 7. N+1 问题与加载策略

## 7.1 N+1 问题剖析

N+1 问题是 JPA/Hibernate 中最常见也最严重的性能问题：

```java
// 假设 Order 与 OrderItem 是 @OneToMany 关系
List<Order> orders = orderRepository.findAll();
// 执行 1 条 SQL：SELECT * FROM orders

for (Order order : orders) {
    // 每次访问 order.getItems() 触发一次 SQL
    System.out.println(order.getItems().size());
}
// 总共执行了 N+1 条 SQL（N = orders 数量）
```

原因：延迟加载机制——关联集合默认不立即加载，首次访问时才发送 SQL 查询。

## 7.2 解决方案

### 7.2.1 JOIN FETCH

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customerId = :customerId")
List<Order> findOrdersWithItems(@Param("customerId") Long customerId);

// 多级 FETCH
@Query("SELECT o FROM Order o " +
       "JOIN FETCH o.items i " +
       "JOIN FETCH i.product " +
       "WHERE o.id = :id")
Optional<Order> findOrderDetail(@Param("id") Long id);
```

### 7.2.2 @EntityGraph

```java
// 方式1：实体上定义命名 EntityGraph
@Entity
@NamedEntityGraph(
    name = "Order.withItems",
    attributeNodes = @NamedAttributeNode("items")
)
public class Order { ... }

@EntityGraph("Order.withItems")
List<Order> findByCustomerId(Long customerId);

// 方式2：直接在方法上声明
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByCustomerId(Long customerId);

// 方式3：动态创建
@EntityGraph(type = EntityGraph.EntityGraphType.LOAD,
             attributePaths = {"items"})
Optional<Order> findById(Long id);
```

**EntityGraph 的 type 属性**：

| 类型 | 说明 |
|------|------|
| `FETCH`（默认） | 图中指定的属性使用 EAGER，其他遵循默认策略 |
| `LOAD` | 图中指定的属性使用 EAGER，其他全部使用 LAZY |

### 7.2.3 @BatchSize

```java
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 10)  // 批量加载，每次最多 10 个
    private Order order;
}

@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 50)
    private List<OrderItem> items;
}
```

### 7.2.4 FetchMode（Hibernate 特有）

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @Fetch(FetchMode.SUBSELECT)  // 使用子查询一次性加载
    // @Fetch(FetchMode.JOIN)   // 使用 JOIN
    // @Fetch(FetchMode.SELECT) // 默认（N+1）
    private List<OrderItem> items;
}
```

| 模式 | SQL 生成 | 适用场景 |
|------|---------|---------|
| `SELECT`（默认） | N+1 条 | 数据量极小 |
| `JOIN` | 1 条 LEFT JOIN | 数据量适中 |
| `SUBSELECT` | 2 条（主查询 + 子查询） | 数据量大，避免笛卡尔积 |

## 7.3 加载策略选择指南

```
1. 只查父实体，不涉及子实体 → 保持默认 LAZY
2. 少量数据查询，需要子实体数据 → @EntityGraph 或 JOIN FETCH
3. 大量数据查询，需要子实体 → @BatchSize 或 FetchMode.SUBSELECT
4. 只需要个别字段 → Projection / DTO 投影
5. 复杂嵌套 → 拆分为多个简单查询，应用层组装
```

# 8. 缓存机制

## 8.1 一级缓存

与 EntityManager 生命周期绑定，无法关闭：

```
特性：
- 范围：EntityManager 级别（通常等于一次事务）
- 自动启用，无法禁用
- 保证同一 EntityManager 内相同 ID 的实体对象唯一
- flush 时将变更同步到数据库

注意事项：大规模批量操作时应定期 clear() 释放内存
```

## 8.2 二级缓存

跨 EntityManager 的进程级缓存：

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: jcache
          use_query_cache: true
```

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Category {
    @Id
    private Long id;
    private String name;
}

// 关联缓存
@Entity
public class Product {
    @ManyToOne(fetch = FetchType.LAZY)
    @Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
    private Category category;
}
```

**缓存并发策略**：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `READ_ONLY` | 只读缓存，最高效 | 永不修改的静态数据 |
| `NONSTRICT_READ_WRITE` | 读写缓存，不保证严格一致性 | 很少更新的数据 |
| `READ_WRITE` | 读写缓存，软锁定保证一致性 | 经常更新的数据 |
| `TRANSACTIONAL` | 全事务缓存 | 需 JTA 环境 |

# 9. 锁机制

## 9.1 乐观锁

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @Version
    private Integer version;

    private Integer stock;
}

// 并发更新 + 重试机制
@Transactional
public void safeDecreaseStock(Long productId, int quantity) {
    int retries = 3;
    while (retries > 0) {
        try {
            productRepository.decreaseStock(productId, quantity);
            return;
        } catch (OptimisticLockException e) {
            retries--;
            if (retries == 0) throw e;
            Thread.sleep(50);
        }
    }
}
```

Hibernate 生成的 SQL：`UPDATE product SET stock = ?, version = version + 1 WHERE id = ? AND version = ?`

## 9.2 悲观锁

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForRead(@Param("id") Long id);
}
```

| 锁类型 | SQL 生成 | 说明 |
|--------|---------|------|
| `PESSIMISTIC_READ` | `SELECT ... FOR SHARE` | 共享锁，可读不可写 |
| `PESSIMISTIC_WRITE` | `SELECT ... FOR UPDATE` | 排他锁 |
| `PESSIMISTIC_FORCE_INCREMENT` | `SELECT ... FOR UPDATE` + version 递增 | 强制版本号变化 |

**乐观锁 vs 悲观锁**：

| 场景 | 推荐 | 原因 |
|------|------|------|
| 冲突概率低（编辑资料） | 乐观锁 | 性能好，无数据库锁开销 |
| 冲突概率高（秒杀） | 悲观锁 | 避免频繁重试 |
| 长事务 | 乐观锁 | 悲观锁长时间持有会造成阻塞 |

# 10. 性能优化

## 10.1 批量操作

```java
@Transactional
public void batchInsert(List<Product> products) {
    int batchSize = 50;
    for (int i = 0; i < products.size(); i++) {
        entityManager.persist(products.get(i));
        if (i % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();  // 清除一级缓存防止 OOM
        }
    }
}
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

## 10.2 只读优化

```java
@Transactional(readOnly = true)  // Hibernate 跳过脏检查
public List<Product> searchProducts(String keyword) {
    return productRepository.findByNameContaining(keyword);
}

@QueryHints(@QueryHint(name = "org.hibernate.readOnly", value = "true"))
List<Product> findReadOnlyByCategory(String category);
```

## 10.3 避免不必要的查询

```java
// 仅需判断存在性时使用 exists
boolean existsByEmail(String email);

// 使用 getReferenceById 获取代理，不查数据库
@Transactional
public void addOrderItem(Long orderId, OrderItem item) {
    Order order = orderRepository.getReferenceById(orderId);  // 不查库
    item.setOrder(order);
    orderItemRepository.save(item);
}

// 使用 @Modifying 避免先查后改
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id AND p.stock >= :qty")
int decreaseStock(@Param("id") Long id, @Param("qty") int qty);
```

## 10.4 SQL 日志与监控

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true
        session.events.log: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    org.hibernate.stat: DEBUG
```

# 11. Auditing（审计）

## 11.1 启用审计

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig { }
```

## 11.2 审计实体基类

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createTime;

    @LastModifiedDate
    private LocalDateTime updateTime;

    @CreatedBy
    @Column(updatable = false)
    private Long createBy;

    @LastModifiedBy
    private Long updateBy;
}

@Entity
public class Product extends BaseEntity {
    // 自动记录 createTime、updateTime、createBy、updateBy
}

// AuditorAware 实现
@Component
public class SpringSecurityAuditorAware implements AuditorAware<Long> {
    @Override
    public Optional<Long> getCurrentAuditor() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            return Optional.empty();
        }
        return Optional.of(((UserDetails) auth.getPrincipal()).getId());
    }
}
```

# 12. 源码与底层机制

## 12.1 Repository 代理创建流程

```
@EnableJpaRepositories (或自动配置)
  → JpaRepositoriesRegistrar
    → JpaRepositoryConfigExtension
      → JpaRepositoryFactoryBean → getObject()
        → JpaRepositoryFactory.getRepository(UserRepository.class)
          → 创建 SimpleJpaRepository (CrudRepository 默认实现)
          → 创建 JDK 动态代理
            → RepositoryMethodInterceptor → 方法分发：
              ├── 派生查询 → PartTreeJpaQuery
              ├── @Query    → DeclaredQuery
              └── 默认方法  → SimpleJpaRepository.xxx()
```

## 12.2 SimpleJpaRepository 核心实现

```java
// save 方法简化逻辑
@Transactional
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);      // 瞬时态 → 持久态
        return entity;
    } else {
        return em.merge(entity); // 游离态 → 持久态
    }
}

// isNew 判断逻辑：
// 1. @Id 字段为 null → 新实体
// 2. @Version 字段为 null → 新实体
// 3. 实现了 Persistable 接口 → 调用 isNew() 方法
// 4. ID 非 null 且非 0 → 假设已存在实体
```

## 12.3 派生查询解析过程

```
findByUserNameAndAgeGreaterThanOrderByCreateTimeDesc

解析为 PartTree：
  ├── 前缀：find
  ├── 主题：
  │   ├── Predicate: UserName (And)
  │   ├── Predicate: Age (GreaterThan)
  │   └── OrderBy: CreateTime (Desc)
  └── 结果限制：无

转换为 JPQL：
  SELECT user FROM User user
  WHERE user.userName = :userName AND user.age > :age
  ORDER BY user.createTime DESC

转换为 SQL（Hibernate）：
  SELECT u.id, u.user_name, u.age, u.create_time
  FROM t_user u
  WHERE u.user_name = ? AND u.age > ?
  ORDER BY u.create_time DESC
```

# 13. 常见问题与面试题

## 13.1 save vs persist vs merge 的区别

```java
// persist：将瞬时态 → 持久态，不返回 ID（void）
//   如果实体已有 ID（被 Hibernate 认为是 detached），抛异常
entityManager.persist(user);

// merge：将游离态 → 持久态，返回持久态副本
User managedUser = entityManager.merge(detachedUser);
// merge 返回新对象，detachedUser 本身仍为游离态

// save（Spring Data JPA）：
//   内部判断 entityInformation.isNew(entity)
//     → true: 调用 persist
//     → false: 调用 merge
userRepository.save(user);
```

## 13.2 实体需要 hashCode/equals 吗？

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User user)) return false;
        return id != null && Objects.equals(id, user.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();  // 固定 hash，避免 ID 变化导致 Set 丢失元素
    }
}
```

## 13.3 OpenSessionInView 问题

```
默认启用（spring.jpa.open-in-view=true）：
- 好处：Controller 层可延迟加载关联对象
- 坏处：数据库连接在整个请求期间保持（影响连接池）
- 推荐：生产环境设为 false
- 设为 false 后所有关联数据必须在 @Transactional 内加载完毕
```

## 13.4 面试高频考点

**1. N+1 问题如何排查和解决？**
- 排查：开启 SQL 日志，观察重复的 SELECT 语句
- 解决：JOIN FETCH、@EntityGraph、@BatchSize、FetchMode.SUBSELECT

**2. 一级缓存和二级缓存的区别？**
- 一级：EntityManager 级别，默认开启不可关闭，事务范围
- 二级：SessionFactory 级别，需配置启用，跨 Session 共享

**3. getReferenceById 和 findById 的区别？**
- `getReferenceById`：返回代理对象，不立即查库，访问非 ID 属性时才发 SQL
- `findById`：立即查库，返回真实对象或 Optional.empty()

**4. @Transactional 在 JPA 中的作用？**
- 定义事务边界，保证 PersistenceContext 在整个事务中保持
- 控制 flush 和提交时机
- 不使用 @Transactional 时每个 Repository 方法独立事务

**5. 为什么 @OneToMany 默认 LAZY，@ManyToOne 默认 EAGER？**
- 一对多：子集合可能很大，EAGER 加载导致大量不必要的数据查询
- 多对一：只加载一个对象，开销较小（实践中也建议改为 LAZY）

# 14. 测试

## 14.1 Repository 集成测试

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindByEmail() {
        User user = new User();
        user.setUserName("测试");
        user.setEmail("test@example.com");
        entityManager.persistAndFlush(user);

        Optional<User> found = userRepository.findByEmail("test@example.com");
        assertThat(found).isPresent();
    }

    @Test
    void shouldNotFindByNonExistentEmail() {
        Optional<User> found = userRepository.findByEmail("no@user.com");
        assertThat(found).isEmpty();
    }
}
```

## 14.2 @Sql 注解

```java
@SpringBootTest
@Sql(scripts = "/test-data.sql")
@Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    void shouldCreateUser() {
        User user = userService.createUser(new User());
        assertThat(user.getId()).isNotNull();
    }
}
```

# 15. 多数据源配置

```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository.primary",
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryDataSourceConfig {

    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return builder
            .dataSource(dataSource)
            .packages("com.example.entity.primary")
            .persistenceUnit("primary")
            .build();
    }

    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager transactionManager(
            @Qualifier("primaryEntityManagerFactory") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```
