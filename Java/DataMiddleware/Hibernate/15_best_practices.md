# 1. 实体设计最佳实践

## 1.1 实体设计原则

### 1.1.1 包装类型优于基本类型

实体属性应使用包装类型（`Long`、`Integer`、`Boolean`）而非基本类型（`long`、`int`、`boolean`）。包装类型可以表达 `null`，在以下场景至关重要：

- 字段未赋值时与数据库 `NULL` 语义一致
- 查询条件中区分"未指定"与"指定为 0/false"
- 避免基本类型默认值（0、false）被错误持久化

```java
// ❌ 错误：基本类型无法表达 null
@Entity
public class Product {
    @Id
    private long id;           // 默认值 0，可能与合法 ID 混淆
    private int stock;         // 默认值 0，无法区分"未设置"和"库存为 0"
    private double price;      // 默认值 0.0
    private boolean active;    // 默认值 false
}

// ✅ 正确：使用包装类型
@Entity
public class Product {
    @Id
    private Long id;           // null 表示未持久化
    private Integer stock;     // null 表示未设置，0 表示库存为零
    private BigDecimal price;  // BigDecimal 替代浮点数（精度要求）
    private Boolean active;    // null 表示未设置
}
```

### 1.1.2 实现 Serializable 接口

实体类应实现 `java.io.Serializable` 接口，原因包括：

- 二级缓存可能将实体序列化到磁盘或网络（如 Redis、Hazelcast）
- 分布式环境需要通过网络传输实体
- Session 钝化/激活需要序列化
- DTO 与实体分离场景涉及序列化

```java
@Entity
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    // 当实体结构稳定后固定 serialVersionUID
    // 反序列化时 JVM 会校验该值，确保类版本一致
}
```

> **注意**：`serialVersionUID` 应在实体结构稳定后明确设定，避免默认生成的 UID 因微小改动而改变。

### 1.1.3 非 final 类 + 非 final 方法

Hibernate 通过运行时代理（CGLIB/ByteBuddy）实现懒加载，因此：

- 实体类不能声明为 `final`
- 懒加载的方法（getter）不能声明为 `final`
- 无参构造方法至少为 `protected`（不能是 `private`）

```java
// ❌ 错误：final 类无法被代理
@Entity
public final class Order { ... }

// ❌ 错误：final 方法阻止代理拦截懒加载
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;

    public final Customer getCustomer() {  // Lazy 加载将失败
        return customer;
    }
}

// ✅ 正确：非 final 类与非 final getter
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;

    // 无参构造方法至少为 protected
    protected Order() {}

    // Getter 非 final，允许代理拦截
    public Customer getCustomer() {
        return customer;
    }
}
```

### 1.1.4 equals/hashCode 实现规范

实体中 `equals`/`hashCode` 的实现是 Hibernate 中最容易出错的地方之一。核心原则：

**永远不要使用数据库自增 ID 作为 equals/hashCode 的基础**，因为：
- 新建实体未持久化时 ID 为 `null`，持久化后才有值
- 同一个实体在持久化前后 `hashCode` 会变化，导致 Set/Map 行为异常

正确做法：使用**不可变的业务键**（如 `ISBN`、`email`）或**约定策略**。

```java
// ✅ 方案一：基于业务键（推荐，需确保业务键不为 null）
@Entity
public class Book {
    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, unique = true)
    private String isbn;  // 不可变的业务键

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Book)) return false;
        Book book = (Book) o;
        return Objects.equals(isbn, book.isbn);
    }

    @Override
    public int hashCode() {
        return Objects.hash(isbn);
    }
}

// ✅ 方案二：无业务键时，使用实体类型 + ID（约定 ID 赋值后不再变化）
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return id != null && Objects.equals(id, order.id);
    }

    @Override
    public int hashCode() {
        // 使用固定值确保持久化前后一致，但牺牲了 hash 分布的均匀性
        return getClass().hashCode();
    }
}

// ❌ 错误：基于自增 ID（持久化前后 hashCode 变化）
@Entity
public class BadEntity {
    @Id
    @GeneratedValue
    private Long id;

    @Override
    public int hashCode() {
        return Objects.hash(id);  // id 为 null 时 hash=0，持久化后 hash 改变
    }
}
```

### 1.1.5 toString() 避免懒加载关联

在 `toString()` 中引用懒加载的关联属性会导致意外的数据库查询，甚至在日志输出中触发 `LazyInitializationException`。

```java
// ❌ 错误：toString() 触发懒加载
@Entity
public class Employee {
    @ManyToOne(fetch = FetchType.LAZY)
    private Department department;

    @Override
    public String toString() {
        return "Employee{id=" + id + ", dept=" + department + "}";  // 触发查询
    }
}

// ✅ 正确：toString() 只包含基本属性
@Entity
public class Employee {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Department department;

    @Override
    public String toString() {
        return "Employee{id=" + id + ", name='" + name + "', deptId="
            + (department != null ? department.getId() : null) + "}";
    }
}
```

### 1.1.6 equals/hashCode 避免包含关联实体

在 `equals`/`hashCode` 中包含 `@ManyToOne` 或 `@OneToOne` 关联可能导致：
- 递归调用（A contains B, B contains A）
- 懒加载触发（访问关联时产生额外 SQL）
- 代理对象比较的复杂性

```java
// ❌ 错误：equals 中包含关联实体
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    private Order order;

    @Override
    public boolean equals(Object o) {
        // 访问 order 可能触发懒加载，且可能导致递归
        return Objects.equals(id, other.id) && Objects.equals(order, other.order);
    }
}

// ✅ 正确：仅比较自身属性或业务键
@Entity
public class OrderItem {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Order order;

    private String productCode;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderItem)) return false;
        OrderItem that = (OrderItem) o;
        return Objects.equals(productCode, that.productCode)
            && order != null && that.order != null
            && Objects.equals(order.getId(), that.order.getId());
    }
}
```

### 1.1.7 Lombok @Data 的陷阱

`@Data` 注解自动生成 `equals`、`hashCode`、`toString`、getter/setter，但在 JPA 实体中使用时存在严重问题：

| 问题 | 原因 | 后果 |
|------|------|------|
| `equals/hashCode` 包含所有字段 | 包括懒加载关联和自增 ID | 持久化前后 hash 变化、触发懒加载 |
| `toString` 包含所有字段 | 懒加载关联被访问 | LazyInitializationException 或意外查询 |
| 双向关联的 `toString` | A.toString() 调用 B.toString() 调用 A.toString()... | StackOverflowError |

```java
// ❌ 错误：@Data 在 JPA 实体中的典型问题
@Entity
@Data  // 危险！对所有字段生成 equals/hashCode/toString
public class User {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;  // toString() 会触发懒加载
}

// ✅ 正确：仅使用必要的 Lombok 注解
@Entity
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User {
    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;

    // 手动实现 equals/hashCode 基于业务键
    @Column(unique = true, nullable = false)
    private String email;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return Objects.equals(email, user.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(email);
    }

    @Override
    public String toString() {
        return "User{id=" + id + ", email='" + email + "'}";
    }
}
```

### 1.1.8 实体作为 DTO 的利弊讨论

| 维度 | 实体直接作为 DTO | 独立 DTO |
|------|----------------|---------|
| **开发效率** | 高（无需额外类） | 低（需要转换代码） |
| **安全性** | 差（密码等敏感字段易泄露） | 好（精确控制暴露字段） |
| **灵活性** | 差（受 Jackson 注解耦合影响） | 好（独立设计序列化格式） |
| **懒加载** | 差（序列化时触发大量 SQL） | 好（DTO 不含懒加载关联） |
| **维护性** | 差（实体变更影响 API） | 好（API 与实体解耦） |

**建议**：生产项目中始终使用独立 DTO。只有极简单的 CRUD 或内部工具可考虑实体直接暴露。

## 1.2 主键设计

### 1.2.1 SEQUENCE vs IDENTITY

| 特性 | SEQUENCE | IDENTITY |
|------|----------|----------|
| 批处理支持 | 支持 | **不支持**（每条 INSERT 后需立即获取 ID） |
| JDBC batch + hibernate.order_inserts | 正常 | 被静默禁用 |
| 数据库可移植性 | 支持数据库有限 | 多数数据库支持 |
| ID 预分配 | 支持（allocationSize） | 不支持 |
| MySQL 支持 | 不支持 | 支持（AUTO_INCREMENT） |

```java
// ✅ 推荐：PostgreSQL/Oracle 使用 SEQUENCE
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
    private Long id;
}

// ⚠️ MySQL 场景下的最佳可用方案（IDENTITY 为无奈选择）
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}

// ✅ MySQL 替代方案：使用 TABLE 生成器（支持批处理，但性能略低）
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "order_gen")
    @TableGenerator(name = "order_gen", table = "id_generator",
                    pkColumnName = "gen_name", valueColumnName = "gen_value",
                    allocationSize = 50)
    private Long id;
}
```

### 1.2.2 UUID 作为业务主键

在分布式系统中，UUID 作为主键的优势：

- **离线生成**：无需访问数据库即可生成 ID
- **全局唯一**：数据合并/同步无冲突
- **安全性**：不可猜测的 ID，避免遍历攻击

缺点：

- 存储空间大（36 字符 vs 8 字节 Long）
- 随机 UUID 导致索引页分裂，写入性能下降
- 可读性差，调试不便

```java
// ✅ UUID 主键实现（注意：使用 TSID 或时间戳优先的 UUID v7 更好）
@Entity
public class DistributedEntity {
    @Id
    @Column(length = 36)
    private String id;

    @PrePersist
    public void generateId() {
        if (this.id == null) {
            this.id = UUID.randomUUID().toString();
        }
    }
}

// ✅ 更优方案：使用 ULID 或 TSID（时间戳有序，适合索引）
// 需要引入第三方库（如 tsid-creator）
@Entity
public class BetterDistributedEntity {
    @Id
    @Column(length = 26)
    private String id;

    @PrePersist
    public void generateId() {
        if (this.id == null) {
            this.id = TsidCreator.getTsid().toString();  // 时间戳前缀 + 随机后缀
        }
    }
}
```

### 1.2.3 自然键 vs 代理键

| 维度 | 自然键 | 代理键（自增/序列） |
|------|--------|---------------------|
| 业务含义 | 有 | 无 |
| 变更难度 | 高（业务规则可能变） | 低（内部使用，不变） |
| 存储大小 | 可能大 | 固定小 |
| 索引效率 | 可能差 | 最优 |
| 分布式友好 | 是 | 否 |

**建议**：代理键作为主键，自然键加唯一约束。在实体 `equals/hashCode` 中使用自然键。

### 1.2.4 主键类型选择

```java
// ✅ 推荐顺序：Long > Integer > String > UUID
// Long 足够大（9,223,372,036,854,775,807），基本不会溢出
// Integer 最大值仅 21 亿，可能不够
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE)
private Long id;  // 最佳选择
```

### 1.2.5 雪花算法 / 分布式 ID 生成器集成

```java
// 集成雪花算法（Snowflake）ID 生成器
@Entity
public class Order {
    @Id
    private Long id;  // 不依赖数据库生成

    @PrePersist
    public void generateId() {
        if (this.id == null) {
            this.id = SnowflakeIdGenerator.nextId();
        }
    }
}

// Snowflake ID 生成器（简化示例）
public class SnowflakeIdGenerator {
    private static final long EPOCH = 1609459200000L;  // 2021-01-01
    private static final long WORKER_ID_BITS = 5L;
    private static final long SEQUENCE_BITS = 12L;
    private static final long MAX_SEQUENCE = (1L << SEQUENCE_BITS) - 1;

    private static final long workerId = 1L;
    private static long sequence = 0L;
    private static long lastTimestamp = -1L;

    public static synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("时钟回拨，拒绝生成 ID");
        }
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {
                // 当前毫秒序列号用完，等待下一毫秒
                while (timestamp <= lastTimestamp) {
                    timestamp = System.currentTimeMillis();
                }
            }
        } else {
            sequence = 0L;
        }
        lastTimestamp = timestamp;
        return ((timestamp - EPOCH) << (WORKER_ID_BITS + SEQUENCE_BITS))
             | (workerId << SEQUENCE_BITS)
             | sequence;
    }
}
```

## 1.3 审计字段设计

### 1.3.1 基类模式（@MappedSuperclass）

将审计字段抽取到基类中，所有实体通过继承复用，减少重复代码：

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedBy
    private Long lastModifiedBy;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;

    @Version
    private Long version;

    @Column(nullable = false)
    private Boolean deleted = false;
}

// 子实体继承
@Entity
public class Product extends BaseEntity {
    private String name;
    private BigDecimal price;
}

@Entity
public class Customer extends BaseEntity {
    private String customerName;
    private String email;
}
```

### 1.3.2 Spring Data JPA 审计功能配置

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    public AuditorAware<Long> auditorProvider() {
        // 通常从 SecurityContext 或 Token 中获取当前用户 ID
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
                .map(ctx -> ctx.getAuthentication())
                .filter(Authentication::isAuthenticated)
                .map(Authentication::getPrincipal)
                .filter(principal -> principal instanceof UserDetails)
                .map(UserDetails.class::cast)
                .map(userDetails -> Long.valueOf(userDetails.getUsername()));
    }
}
```

### 1.3.3 Hibernate Envers 完整审计追踪

```java
// 引入 Hibernate Envers 依赖后，只需标注 @Audited
@Entity
@Audited
public class BankAccount {
    @Id
    @GeneratedValue
    private Long id;

    private String accountNumber;
    private BigDecimal balance;

    // @NotAudited 标注不需要审计的字段
    @NotAudited
    private String internalNote;
}

// Envers 自动创建 _AUD 表和 REVINFO 表
// 查询历史版本
public class AuditQueryExample {
    @PersistenceContext
    private EntityManager em;

    public List<BankAccount> findRevisions(Long accountId) {
        AuditReader reader = AuditReaderFactory.get(em);
        List<Number> revisions = reader.getRevisions(BankAccount.class, accountId);
        return revisions.stream()
            .map(rev -> reader.find(BankAccount.class, accountId, rev))
            .collect(Collectors.toList());
    }

    public BankAccount findAtTime(Long accountId, LocalDateTime dateTime) {
        AuditReader reader = AuditReaderFactory.get(em);
        long timestamp = dateTime.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli();
        return reader.find(BankAccount.class, accountId, timestamp / 1000);
    }
}
```

---

# 2. 关联映射最佳实践

## 2.1 关联方向选择

### 2.1.1 优先使用单向关联

每个双向关联都是一笔"技术债务"：需要维护两边一致性，增加代码复杂度。

```java
// ✅ 推荐：单向关联（简单、清晰）
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;  // Order 知道 Customer，但 Customer 不需要知道所有 Order
}

@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;
    // 不持有 Order 集合 — 需要查询时通过 Repository 查询
}

// ❌ 仅在必要时使用双向关联（如确实需要从 Customer 访问 Orders）
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer")
    private Set<Order> orders = new HashSet<>();  // 仅在确实需要时添加
}
```

### 2.1.2 双向关联以 @ManyToOne 方为拥有方

```java
// ✅ 标准双向关联模式
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)  // 拥有方
    @JoinColumn(name = "customer_id")
    private Customer customer;
}

@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer")  // 反方向，非拥有方
    private Set<Order> orders = new HashSet<>();
}
```

## 2.2 FetchType 使用规范

### 2.2.1 @ManyToOne / @OneToOne 必须设为 LAZY

JPA 规范规定 `@ManyToOne` 和 `@OneToOne` 默认 `FetchType.EAGER`，这是性能问题的头号来源。

```java
// ❌ 错误：EAGER 是 @ManyToOne 的默认值
@Entity
public class OrderItem {
    @ManyToOne  // 默认为 EAGER！每次查询 OrderItem 都会 JOIN Product
    private Product product;

    @ManyToOne  // 每次查询 OrderItem 都会 JOIN Order
    private Order order;
}

// ✅ 正确：显式指定 LAZY
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
}
```

> **绝对规则**：项目中所有 `@ManyToOne` 和 `@OneToOne` 必须显式标注 `fetch = FetchType.LAZY`。

### 2.2.2 @OneToMany / @ManyToMany 保持 LAZY

这两个注解默认就是 `LAZY`，保持默认即可。**绝不要改为 EAGER**。

```java
// ✅ 保持默认 LAZY — 不额外标注即可
@Entity
public class Department {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "department")  // 默认 LAZY，正确
    private Set<Employee> employees = new HashSet<>();
}
```

### 2.2.3 避免 FetchType.EAGER

EAGER 的问题清单：
1. 即使不需要关联数据也会加载 — 浪费资源
2. 多个 EAGER 关联导致笛卡尔积 — 结果集指数膨胀
3. JOIN 查询可能比多次查询更慢 — 无差别使用
4. 开发者无法在调用层控制 — 失去灵活性

## 2.3 级联操作规范

### 2.3.1 CascadeType 使用矩阵

| 关联类型 | 推荐级联 | orphanRemoval | 说明 |
|---------|---------|---------------|------|
| @OneToOne（父子） | ALL | 推荐 | 生命周期一致 |
| @OneToMany（父子） | ALL | 推荐 | 子实体由父实体独占 |
| @ManyToMany | PERSIST + MERGE | 绝对不 | 多对多共享引用 |
| @ManyToOne | PERSIST | 可选 | 通常不级联删除父方 |

```java
// ✅ @OneToMany 父子关系：ALL + orphanRemoval
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);  // orphanRemoval=true 会触发子实体删除
    }
}

// ✅ @ManyToMany：仅 PERSIST + MERGE
@Entity
public class Student {
    @Id @GeneratedValue
    private Long id;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(name = "student_course",
               joinColumns = @JoinColumn(name = "student_id"),
               inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}

// ❌ 错误：@ManyToMany 级联 REMOVE — 删除学生时删除所有关联课程
@Entity
public class BadStudent {
    @ManyToMany(cascade = CascadeType.ALL)  // 危险！可能级联删除 Course
    private Set<Course> courses = new HashSet<>();
}
```

### 2.3.2 多对多拆分为中间实体

多对多关系往往携带额外属性（加入时间、状态等），拆分中间实体是更好的实践：

```java
// ✅ 拆分为中间实体（推荐）
@Entity
public class Student {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "student", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Enrollment> enrollments = new HashSet<>();
}

@Entity
public class Course {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "course")
    private Set<Enrollment> enrollments = new HashSet<>();
}

@Entity
public class Enrollment {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    private Course course;

    private LocalDateTime enrolledAt;  // 额外属性
    private String status;             // 额外属性
}
```

## 2.4 双向关联同步辅助方法

双向关联必须有同步方法确保两端一致性，否则数据不一致的可能性很高。

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    public void setCustomer(Customer customer) {
        // 从旧关联中移除
        if (this.customer != null) {
            this.customer.getOrders().remove(this);
        }
        this.customer = customer;
        // 在新关联中注册
        if (customer != null) {
            customer.getOrders().add(this);
        }
    }
}

@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Order> orders = new HashSet<>();

    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this);
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
}
```

> **重要**：数据库中最终以**拥有方**（`@ManyToOne` 方，即 `Order.customer`）为准。永远在拥有方设置关联后再保存。

## 2.5 集合选择指南

### 2.5.1 场景决策矩阵

| 场景 | 推荐集合 | 原因 |
|------|---------|------|
| 无重复、无排序需求 | `Set` | 语义明确，删除后不会重新插入全量 |
| 需要按某字段排序（只读） | `List` + `@OrderBy` | 数据库排序，无额外列 |
| 需要可变排序位置 | `List` + `@OrderColumn` | 维护一个排序列 |
| 按键快速查找 | `Map` + `@MapKey` | 按 key 直接获取关联实体 |

```java
// ✅ Set：最通用，避免重复和重新插入问题
@Entity
public class Department {
    @OneToMany(mappedBy = "department", cascade = ALL, orphanRemoval = true)
    private Set<Employee> employees = new HashSet<>();
}

// ✅ List + @OrderBy：需要固定排序
@Entity
public class Article {
    @OneToMany(mappedBy = "article")
    @OrderBy("createdAt DESC")
    private List<Comment> comments = new ArrayList<>();
}

// ✅ List + @OrderColumn：可拖拽排序
@Entity
public class Playlist {
    @OneToMany(cascade = ALL, orphanRemoval = true)
    @OrderColumn(name = "track_index")
    private List<Track> tracks = new ArrayList<>();
}

// ✅ Map：按类型快速查找
@Entity
public class Configuration {
    @OneToMany(mappedBy = "config", cascade = ALL, orphanRemoval = true)
    @MapKey(name = "settingKey")
    private Map<String, ConfigEntry> settings = new HashMap<>();
}
```

### 2.5.2 Bag 的问题

当 `List` 没有 `@OrderColumn` 时，Hibernate 视为 **Bag**（无序、允许重复）：

- Hibernate 删除后再全部重新插入（性能差）
- 元素顺序不保证
- 如果实体未正确实现 `equals`/`hashCode`，会导致重复元素

```java
// ❌ 问题：List 无 @OrderColumn → Bag 语义
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    private List<Employee> employees = new ArrayList<>();
    // 删除一个 Employee 时，Hibernate 可能先删除全部，再插入剩下的
}

// ✅ 解决方案一：使用 Set（推荐）
@OneToMany(mappedBy = "department")
private Set<Employee> employees = new HashSet<>();

// ✅ 解决方案二：确需 List 时添加 @OrderColumn
@OneToMany(mappedBy = "department")
@OrderColumn(name = "sort_order")
private List<Employee> employees = new ArrayList<>();
```

### 2.5.3 关联量大时的分页策略

```java
// ✅ 使用 @BatchSize 限制一次加载数量
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    @BatchSize(size = 20)  // 每次最多初始化 20 个 Employee
    private Set<Employee> employees = new HashSet<>();
}

// ✅ 更好的做法：不映射大集合，通过 Repository 分页查询
// 不在 Department 中持有 employees 字段
// 需要时通过 EmployeeRepository 分页查询
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    Page<Employee> findByDepartmentId(Long departmentId, Pageable pageable);
}
```

---

# 3. 查询最佳实践

## 3.1 查询选择策略

| 查询类型 | 推荐方案 | 示例 |
|---------|---------|------|
| 简单精确查询 | Spring Data JPA 方法命名 | `findByEmail(String email)` |
| 简单范围查询 | Spring Data JPA 方法命名 | `findByAgeBetween(int min, int max)` |
| 静态多条件查询 | `@Query` + JPQL | `@Query("SELECT u FROM User u WHERE u.status = :status")` |
| 动态多条件查询 | Criteria API / Specification | 搜索表单（条件可选） |
| 复杂报表查询 | 原生 SQL | 多表聚合、窗口函数 |
| 全文搜索 | 搜索引擎（Elasticsearch） | 不是 Hibernate 的职责 |

```java
// 方法命名查询 — 最简单场景
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByAgeBetweenAndStatus(int minAge, int maxAge, UserStatus status);
    boolean existsByEmail(String email);
}

// @Query + JPQL — 静态复杂查询
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o "
         + "JOIN FETCH o.customer "
         + "WHERE o.status = :status "
         + "ORDER BY o.createdDate DESC")
    List<Order> findByStatusWithCustomer(@Param("status") OrderStatus status);

    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.customer.name, o.totalAmount) "
         + "FROM Order o WHERE o.createdDate BETWEEN :start AND :end")
    List<OrderSummary> findOrderSummaries(@Param("start") LocalDate start,
                                          @Param("end") LocalDate end);
}

// Specification — 动态查询
public class UserSpecification {
    public static Specification<User> withFilters(String name, String email, UserStatus status) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (name != null && !name.isEmpty()) {
                predicates.add(cb.like(root.get("name"), "%" + name + "%"));
            }
            if (email != null) {
                predicates.add(cb.equal(root.get("email"), email));
            }
            if (status != null) {
                predicates.add(cb.equal(root.get("status"), status));
            }
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

## 3.2 避免 N+1 问题的系统方法

### 3.2.1 识别 N+1 问题

```java
// 日志配置（application.yml）
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # 显示参数绑定
```

```java
// 单元测试断言 SQL 执行次数
@Test
public void testNoNPlusOne() {
    // 假设有 5 个 Author，每个 Author 有 3 本书
    // 预期：1 条查询获取 Authors + 1 条批量查询获取 Books = 2 条 SQL

    // 使用 Hypersistence Utils 统计 SQL
    SQLStatementCountValidator.reset();
    List<Author> authors = authorRepository.findAllWithBooks();
    SQLStatementCountValidator.assertSelectCount(2);
}
```

### 3.2.2 解决方案工具包

```java
// 方案一：JOIN FETCH — 单条 JOIN 查询
@Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books WHERE a.id = :id")
Optional<Author> findByIdWithBooks(@Param("id") Long id);

// 方案二：@NamedEntityGraph — 声明式抓取计划
@Entity
@NamedEntityGraph(name = "Author.books",
    attributeNodes = @NamedAttributeNode("books"))
public class Author {
    @OneToMany(mappedBy = "author")
    private Set<Book> books = new HashSet<>();
}

public interface AuthorRepository extends JpaRepository<Author, Long> {
    @EntityGraph(value = "Author.books", type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findAll();
}

// 方案三：@BatchSize — 批量懒加载
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    @BatchSize(size = 50)  // 初始化关联时每批加载 50 个
    private Set<Book> books = new HashSet<>();
}

// 方案四：@Fetch(SUBSELECT) — 子查询一次性加载
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    @Fetch(FetchMode.SUBSELECT)  // SELECT * FROM book WHERE author_id IN (SELECT id FROM author)
    private Set<Book> books = new HashSet<>();
}

// 方案五：全局批量抓取大小（兜底配置）
// application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50  # 全局兜底，所有 LAZY 关联默认批量加载
```

### 3.2.3 场景选择决策流程

```
查询是否总是需要关联数据？
├── 是 → JOIN FETCH（单次查询，最省数据库往返）
│         ⚠️ 不能同时 FETCH 多个 List（Bag）集合 → MultipleBagFetchException
│         ⚠️ 不能同时 FETCH + 分页 → 使用两步查询法
│
└── 否（有时需要、有时不需要）
    └── 使用 @BatchSize 或 global default_batch_fetch_size
        依赖懒加载，触发时批量加载而非逐条加载
```

## 3.3 分页查询注意事项

### 3.3.1 基本分页

```java
// Spring Data JPA 标准分页
Page<Order> findByStatus(OrderStatus status, Pageable pageable);

// 使用示例
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdDate").descending());
Page<Order> orderPage = orderRepository.findByStatus(OrderStatus.PENDING, pageable);

List<Order> orders = orderPage.getContent();        // 当前页数据
long totalElements = orderPage.getTotalElements();  // 总记录数
int totalPages = orderPage.getTotalPages();          // 总页数
```

### 3.3.2 JOIN FETCH + 分页的两步查询法

```java
// ❌ 问题：JOIN FETCH + 分页会全量加载到内存再分页
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
Page<Order> findWithItems(@Param("status") OrderStatus status, Pageable pageable);
// Hibernate 警告: HHH000104: firstResult/maxResults specified with collection fetch

// ✅ 方案：两步查询法
@Repository
public class OrderRepositoryImpl implements OrderRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    public Page<Order> findWithItemsPaged(OrderStatus status, Pageable pageable) {
        // 第一步：查询 ID 列表（轻量）
        Long total = em.createQuery(
                "SELECT COUNT(o) FROM Order o WHERE o.status = :status", Long.class)
            .setParameter("status", status)
            .getSingleResult();

        List<Long> ids = em.createQuery(
                "SELECT o.id FROM Order o WHERE o.status = :status ORDER BY o.createdDate DESC", Long.class)
            .setParameter("status", status)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();

        if (ids.isEmpty()) {
            return new PageImpl<>(Collections.emptyList(), pageable, total);
        }

        // 第二步：用 IN 查询 + JOIN FETCH 加载完整数据
        List<Order> orders = em.createQuery(
                "SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id IN :ids", Order.class)
            .setParameter("ids", ids)
            .getResultList();

        // 按原顺序排序
        Map<Long, Order> orderMap = orders.stream()
            .collect(Collectors.toMap(Order::getId, Function.identity()));
        List<Order> orderedResult = ids.stream()
            .map(orderMap::get)
            .collect(Collectors.toList());

        return new PageImpl<>(orderedResult, pageable, total);
    }
}
```

### 3.3.3 COUNT 查询优化

```java
// ❌ 问题：大表 COUNT 极慢
long count = orderRepository.count();  // 可能有几千万行

// ✅ 方案一：使用数据库的近似计数（如 PostgreSQL 的 pg_class）
@Query(value = "SELECT reltuples::bigint AS estimate FROM pg_class WHERE relname = 'orders'",
       nativeQuery = true)
long estimatedCount();

// ✅ 方案二：使用缓存维护计数（不要求实时精确）
// 通过 Redis 或应用缓存维护一个近似计数

// ✅ 方案三：限制 COUNT 的范围
@Query("SELECT COUNT(o) FROM Order o WHERE o.createdDate > :since")
long countSince(@Param("since") LocalDate since);
```

## 3.4 DTO 使用策略

### 3.4.1 SELECT NEW 构造器表达式

```java
// DTO 定义
public class OrderSummary {
    private Long orderId;
    private String customerName;
    private BigDecimal totalAmount;
    private LocalDateTime orderDate;

    public OrderSummary(Long orderId, String customerName,
                        BigDecimal totalAmount, LocalDateTime orderDate) {
        this.orderId = orderId;
        this.customerName = customerName;
        this.totalAmount = totalAmount;
        this.orderDate = orderDate;
    }
}

// Repository 查询
@Query("SELECT new com.example.dto.OrderSummary("
     + "o.id, o.customer.name, o.totalAmount, o.createdDate) "
     + "FROM Order o WHERE o.status = :status")
List<OrderSummary> findOrderSummaries(@Param("status") OrderStatus status);
```

### 3.4.2 MapStruct 实体-DTO 转换

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface OrderMapper {

    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);

    OrderDTO toDto(Order order);

    @Mapping(target = "items", ignore = true)
    Order toEntity(OrderDTO dto);

    List<OrderDTO> toDtoList(List<Order> orders);
}

// 使用
@Service
@Transactional(readOnly = true)
public class OrderService {
    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;

    public OrderDTO getOrder(Long id) {
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
        return orderMapper.toDto(order);
    }
}
```

### 3.4.3 DTO 投影的性能优势

```java
// ❌ 查询全字段实体
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class).getResultList();
// 加载 30 个字段，全部纳入 PersistenceContext，全部做脏检查

// ✅ DTO 投影 — 仅加载 4 个字段，跳过持久化上下文
List<OrderSummary> summaries = em.createQuery(
    "SELECT new com.example.dto.OrderSummary(o.id, o.customer.name, o.totalAmount, o.createdDate) "
    + "FROM Order o", OrderSummary.class).getResultList();
// 实体不被管理，跳过脏检查，节省内存和 CPU
```

## 3.5 查询日志与性能监控

### 3.5.1 开发环境日志配置

```yaml
# application-dev.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        highlight_sql: true          # 语法高亮（Hibernate 6+）
        generate_statistics: true     # 开启统计信息

logging:
  level:
    org.hibernate.SQL: DEBUG                          # SQL 语句
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # 参数绑定
    org.hibernate.stat: DEBUG                         # 统计摘要
```

### 3.5.2 慢查询检测

```java
// 方案一：Hibernate 拦截器记录慢查询
@Component
public class SlowQueryInterceptor extends EmptyInterceptor {

    @Override
    public void afterTransactionCompletion(Transaction tx) {
        SessionFactory sf = sessionFactory();
        if (sf != null) {
            Statistics stats = sf.getStatistics();
            String[] queries = stats.getQueries();
            for (String query : queries) {
                long maxTime = stats.getQueryStatistics(query).getExecutionMaxTime();
                if (maxTime > 1000) {  // 超过 1 秒
                    log.warn("慢查询: {} 最大执行时间: {}ms", query, maxTime);
                }
            }
        }
    }
}

// 方案二：数据库层面慢查询日志（推荐）
// MySQL: SET GLOBAL slow_query_log = 'ON';
//        SET GLOBAL long_query_time = 1;
// PostgreSQL: ALTER SYSTEM SET log_min_duration_statement = 1000;
```

### 3.5.3 连接池监控

```yaml
# HikariCP 连接池监控配置
spring:
  datasource:
    hikari:
      pool-name: MainPool
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000  # 60s 未归还视为泄漏

logging:
  level:
    com.zaxxer.hikari: DEBUG    # 连接池分配/归还日志
    com.zaxxer.hikari.HikariConfig: DEBUG
```

```java
// 运行时获取连接池指标
@Component
public class HikariMonitor implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        if (dataSource instanceof HikariDataSource) {
            HikariPoolMXBean poolBean = ((HikariDataSource) dataSource).getHikariPoolMXBean();
            int active = poolBean.getActiveConnections();
            int idle = poolBean.getIdleConnections();
            int total = poolBean.getTotalConnections();
            int waiting = poolBean.getThreadsAwaitingConnection();

            if (waiting > 0) {
                return Health.down().withDetail("waitingThreads", waiting).build();
            }
            return Health.up()
                .withDetail("active", active)
                .withDetail("idle", idle)
                .withDetail("total", total)
                .build();
        }
        return Health.unknown().build();
    }
}
```

---

# 4. 事务管理最佳实践

## 4.1 事务边界设计

### 4.1.1 @Transactional 标注位置

```java
// ❌ 错误：Controller 层标注事务
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Transactional  // 错误！事务应在 Service 层
    @PostMapping
    public OrderDTO createOrder(@RequestBody CreateOrderRequest request) {
        // ...
    }
}

// ✅ 正确：Service 层标注事务
@Service
public class OrderService {

    @Transactional
    public OrderDTO createOrder(CreateOrderRequest request) {
        // 1. 获取必要的只读数据（已缓存或已在事务外获取）
        // 2. 执行业务逻辑
        // 3. 持久化
        return orderMapper.toDto(savedOrder);
    }
}
```

### 4.1.2 事务尽量短小

```java
// ❌ 错误：事务中包含外部调用和文件 IO
@Transactional
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setStatus(OrderStatus.PROCESSING);

    // 以下操作不应在事务中执行
    sendNotificationEmail(order);       // 发送邮件（可能耗时）
    generatePDFInvoice(order);          // 文件 IO
    callExternalPaymentGateway(order);  // 外部 API 调用（网络不可靠）
    saveToObjectStorage(order);         // 对象存储

    order.setStatus(OrderStatus.COMPLETED);
}

// ✅ 正确：事务仅包含数据库操作
@Transactional
public void processOrderCore(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setStatus(OrderStatus.PROCESSING);
}

// 非事务方法编排整体流程
public void processOrderFull(Long orderId) {
    processOrderCore(orderId);           // 事务内：数据库状态变更

    Order order = orderRepository.findById(orderId).orElseThrow();
    CompletableFuture.runAsync(() -> {
        sendNotificationEmail(order);    // 异步发送邮件
        generatePDFInvoice(order);
    });
}
```

### 4.1.3 读写分离

```java
@Service
public class OrderQueryService {

    @Transactional(readOnly = true)  // 只读事务优化
    public OrderDTO getOrder(Long id) {
        // readOnly=true 的优化效果：
        // 1. 跳过脏检查（Hibernate 不维护快照）
        // 2. JDBC 层面只读连接（某些数据库优化器利用此信息）
        // 3. 不会触发自动 flush
        return orderMapper.toDto(orderRepository.findById(id).orElseThrow());
    }
}
```

### 4.1.4 长事务的替代方案

```java
// ❌ 避免：长时间事务
@Transactional
public void massUpdatePrices(List<Long> productIds, BigDecimal factor) {
    List<Product> products = productRepository.findAllById(productIds);
    for (Product product : products) {  // 假设 10 万条，事务持续数分钟
        product.setPrice(product.getPrice().multiply(factor));
    }
}

// ✅ 方案一：分段提交（每 100 条一个事务）
@Service
public class BatchUpdateService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updatePriceBatch(List<Product> batch, BigDecimal factor) {
        for (Product product : batch) {
            product.setPrice(product.getPrice().multiply(factor));
        }
    }

    public void massUpdatePrices(List<Long> productIds, BigDecimal factor) {
        List<List<Long>> partitions = partition(productIds, 100);
        for (List<Long> batch : partitions) {
            List<Product> products = productRepository.findAllById(batch);
            self.updatePriceBatch(products, factor);  // 通过 self 代理触发事务
        }
    }
}

// ✅ 方案二：异步处理 + 任务状态追踪
@Transactional
public Long submitMassUpdateJob(List<Long> ids, BigDecimal factor) {
    Job job = new Job(JobType.MASS_UPDATE, JobStatus.PENDING);
    job = jobRepository.save(job);
    
    massUpdateAsyncProcessor.processAsync(job.getId(), ids, factor);
    return job.getId();  // 立即返回任务 ID，客户端轮询
}
```

## 4.2 事务传播行为

| 传播行为 | 场景 | 说明 |
|---------|------|------|
| REQUIRED（默认） | 大多数场景 | 加入已有事务或新建 |
| REQUIRES_NEW | 审计日志、异步操作 | 总是新建事务，挂起当前事务 |
| NESTED | 部分回滚 | JDBC 保存点实现 |
| MANDATORY | 强制要求已有事务 | 调用方不传事务则报错 |
| SUPPORTS | 有事务就用，没有也行 | 很少使用 |
| NOT_SUPPORTED | 明确不需要事务 | 运行在没有事务的上下文中 |
| NEVER | 禁止事务 | 当前有事务则报错 |

```java
// REQUIRES_NEW：审计日志独立事务（即使用户操作回滚，审计记录也保留）
@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOperation(String action, String details) {
        AuditLog log = new AuditLog();
        log.setAction(action);
        log.setDetails(details);
        log.setTimestamp(LocalDateTime.now());
        auditLogRepository.save(log);
        // 当前事务立即提交，不依赖外部事务
    }
}

// NESTED：购物车结算 — 某件商品失败不影响其他商品
@Transactional
public void checkout(List<CartItem> items) {
    for (CartItem item : items) {
        try {
            cartService.reserveStock(item);  // NESTED，失败只回滚此商品
        } catch (InsufficientStockException e) {
            failedItems.add(item);
        }
    }
    // 成功的商品继续处理
}

@Transactional(propagation = Propagation.NESTED)
public void reserveStock(CartItem item) {
    // 设置保存点，失败时回滚到此处
    Product product = productRepository.findByIdWithLock(item.getProductId());
    product.reduceStock(item.getQuantity());
}
```

## 4.3 乐观锁策略

```java
// ✅ 所有易并发冲突的实体使用 @Version
@Entity
public class Inventory {
    @Id @GeneratedValue
    private Long id;

    @Version
    private Long version;  // Long 永远不会溢出（9e18）

    private Integer availableQuantity;

    public void reduceQuantity(int amount) {
        if (this.availableQuantity < amount) {
            throw new InsufficientStockException();
        }
        this.availableQuantity -= amount;
    }
}
```

### 4.3.1 乐观锁冲突处理策略

```java
// 策略一：自动重试（Spring Retry）
@Service
public class InventoryService {

    @Retryable(
        value = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void reduceStock(Long productId, int quantity) {
        Inventory inventory = inventoryRepository.findById(productId).orElseThrow();
        inventory.reduceQuantity(quantity);
        // 提交时若 version 不匹配，Spring 抛出 ObjectOptimisticLockingFailureException
    }

    @Recover
    public void recover(ObjectOptimisticLockingFailureException e,
                        Long productId, int quantity) {
        log.warn("扣减库存失败，已达最大重试次数: productId={}, quantity={}", productId, quantity);
        throw new ConcurrencyException("库存扣减冲突，请稍后重试");
    }
}

// 策略二：用户重新操作（前端提示）
@ControllerAdvice
public class OptimisticLockExceptionHandler {

    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ResponseEntity<ErrorResponse> handleOptimisticLock(
            ObjectOptimisticLockingFailureException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("数据已被其他用户修改，请刷新后重试"));
    }
}
```

## 4.4 事务回滚规则

```java
// 默认行为：RuntimeException + Error 回滚，Checked Exception 不回滚
@Transactional
public void createOrder(OrderRequest request) {
    // RuntimeException → 自动回滚
    // throw new IllegalArgumentException();  // → 回滚

    // Checked Exception → 不回滚（需显式指定 rollbackFor）
    // throw new IOException();  // → 提交！（常见坑）
}

// ✅ 推荐：始终指定 rollbackFor = Exception.class
@Transactional(rollbackFor = Exception.class)
public void createOrderSafe(OrderRequest request) throws OrderCreationException {
    // 任何 Exception 都回滚，避免意外提交
}

// 业务异常封装
public class OrderCreationException extends RuntimeException {
    private final String errorCode;

    public OrderCreationException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}
```

---

# 5. 缓存策略最佳实践

## 5.1 缓存适用场景判断

### 5.1.1 优先考虑不做缓存

在引入缓存之前，依次确认：
1. 是否优化了 SQL 和索引？（这是最大的性能杠杆）
2. 是否消除了 N+1 问题？
3. 是否使用了连接池和批处理？
4. 以上都做了，性能仍不满足要求？

### 5.1.2 缓存适合/不适合的数据

| 适合缓存 | 不适合缓存 |
|---------|-----------|
| 字典数据（国家、省份） | 股票价格、汇率（实时变化） |
| 配置参数（很少变更） | 用户钱包余额（强一致性） |
| 用户权限（登录周期有效） | 秒杀库存（高并发写入） |
| 统计汇总（小时/天级别） | 聊天消息（实时性要求） |

## 5.2 二级缓存决策树

```
是否需要跨 Session 共享数据？
├── 否 → 只用一级缓存（PersistenceContext），无需配置
└── 是 → 考虑二级缓存
    └── 数据变更频率？
        ├── 几乎不变（如国家列表） → CacheConcurrencyStrategy.READ_ONLY
        ├── 偶尔变化（如商品分类）   → CacheConcurrencyStrategy.NONSTRICT_READ_WRITE
        └── 频繁变化（如商品库存）   → CacheConcurrencyStrategy.READ_WRITE
                                     → 但应反思：频繁变更的数据值得缓存吗？
```

```java
// 二级缓存配置示例
@Entity
@Cacheable
@org.hibernate.annotations.Cache(
    usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id
    private String code;
    private String name;
}

@Entity
@Cacheable
@org.hibernate.annotations.Cache(
    usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
public class ProductCategory {
    @Id @GeneratedValue
    private Long id;
    private String name;
    // 偶尔增删分类，NONSTRICT 可接受短暂不一致
}

@Entity
@Cacheable
@org.hibernate.annotations.Cache(
    usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Version  // READ_WRITE 必须使用乐观锁
    private Long version;
}
```

### 5.2.1 查询缓存策略

```java
// 查询缓存 — 仅适用于条件和结果都极其稳定的查询
// ⚠️ 任何涉及的表发生变更，所有相关查询缓存都会失效！

// application.yml 开启查询缓存
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_query_cache: true

// 使用示例
@Query("SELECT c FROM Country c")
@QueryHints(@QueryHint(name = HINT_CACHEABLE, value = "true"))
List<Country> findAllCached();

// ⚠️ 高频更新表绝对不用查询缓存
// 假设 Product 表每秒有 10 次更新，查询缓存命中率接近 0%
```

## 5.3 缓存监控与调优

```java
// 生产环境缓存统计
@Component
public class CacheStatisticsMonitor {

    @PersistenceContext
    private EntityManager em;

    @Scheduled(fixedRate = 60000)  // 每分钟输出
    public void logCacheStats() {
        SessionFactory sf = em.getEntityManagerFactory()
            .unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();

        log.info("=== Hibernate 二级缓存统计 ===");
        log.info("实体缓存命中: {}", stats.getSecondLevelCacheHitCount());
        log.info("实体缓存未命中: {}", stats.getSecondLevelCacheMissCount());
        log.info("实体缓存放入: {}", stats.getSecondLevelCachePutCount());
        log.info("查询缓存命中: {}", stats.getQueryCacheHitCount());
        log.info("查询缓存未命中: {}", stats.getQueryCacheMissCount());
        log.info("查询缓存放入: {}", stats.getQueryCachePutCount());

        double entityHitRate = safeDivide(
            stats.getSecondLevelCacheHitCount(),
            stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount());
        log.info("实体缓存命中率: {:.2f}%", entityHitRate * 100);
    }

    private double safeDivide(long numerator, long denominator) {
        return denominator == 0 ? 0 : (double) numerator / denominator;
    }
}
```

```yaml
# Ehcache 配置示例（缓存大小 + TTL）
# ehcache.xml
config:
  default-cache:
    max-entries-local-heap: 10000
    time-to-live-seconds: 3600
    time-to-idle-seconds: 1800

  cache:
    - name: com.example.entity.Country
      max-entries-local-heap: 500
      time-to-live-seconds: 86400     # 24 小时

    - name: com.example.entity.ProductCategory
      max-entries-local-heap: 1000
      time-to-live-seconds: 3600      # 1 小时
```

---

# 6. 批量处理最佳实践

## 6.1 大批量数据处理

### 6.1.1 flush + clear 模式

```java
// ✅ 标准批量处理：定期 flush + clear
@Transactional
public void batchImportUsers(List<UserDTO> userDTOs) {
    int batchSize = 50;
    for (int i = 0; i < userDTOs.size(); i++) {
        User user = convertToEntity(userDTOs.get(i));
        entityManager.persist(user);

        if (i > 0 && i % batchSize == 0) {
            entityManager.flush();    // 同步到数据库
            entityManager.clear();    // 清空持久化上下文，释放内存
        }
    }
    entityManager.flush();
    entityManager.clear();
}
```

### 6.1.2 JDBC 批处理配置

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50              # 每批 50 条批量执行
        order_inserts: true           # 对 INSERT 排序，增大批次
        order_updates: true           # 对 UPDATE 排序
```

```java
// 批处理验证：观察日志中是否出现多 VALUES 的批量 INSERT
// 如：INSERT INTO user (name, email) VALUES (?, ?), (?, ?), (?, ?) ...

// ⚠️ IDENTITY 策略破坏批处理
// 如果使用 @GeneratedValue(strategy = GenerationType.IDENTITY)
// Hibernate 必须每条 INSERT 后立即获取 ID，批处理被禁用
```

### 6.1.3 StatelessSession 的使用

```java
// StatelessSession：无一级缓存、无脏检查、无级联、无拦截器
// 性能接近原生 JDBC，但功能受限
public void statelessBatchImport(List<UserDTO> userDTOs) {
    Session session = entityManager.unwrap(Session.class);
    SessionFactory sessionFactory = session.getSessionFactory();
    StatelessSession statelessSession = sessionFactory.openStatelessSession();
    Transaction tx = statelessSession.beginTransaction();

    try {
        int batchSize = 50;
        for (int i = 0; i < userDTOs.size(); i++) {
            User user = convertToEntity(userDTOs.get(i));
            // 直接 INSERT，无持久化上下文
            statelessSession.insert(user);

            // batchSize 配合 flush 但无需 clear（无持久化上下文）
        }
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    } finally {
        statelessSession.close();
    }
}
```

### 6.1.4 批量处理策略选择

| 数据量 | 策略 | 理由 |
|-------|------|------|
| < 1000 条 | Session + 无 flush/clear | 持久化上下文足够小 |
| 1000 ~ 10 万 | Session + flush/clear + batch | 定期清理防止 OOM |
| 10 万 ~ 100 万 | StatelessSession + JDBC batch | 跳过 Hibernate ORM 开销 |
| > 100 万 | 数据库导入工具（COPY/LOAD DATA） | 绕过 JDBC，直接到存储引擎 |

## 6.2 导入/导出策略

```java
// 策略矩阵
@Component
public class DataImportService {

    public void importData(List<ImportDTO> data) {
        if (data.size() > 1_000_000) {
            // 超大数据量：写入临时 CSV 文件，执行数据库导入命令
            importViaDatabaseTool(data);
        } else if (data.size() > 10_000) {
            // 中等数据量：StatelessSession + JDBC batch
            importViaStatelessSession(data);
        } else {
            // 小数据量且需要业务逻辑：Session + flush/clear + batch
            importViaSession(data);
        }
    }

    @Transactional
    public void importViaSession(List<ImportDTO> data) {
        int i = 0;
        for (ImportDTO dto : data) {
            // 可能包含业务验证逻辑
            Product product = validateAndConvert(dto);
            entityManager.persist(product);
            if (++i % 50 == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }
    }

    public void importViaStatelessSession(List<ImportDTO> data) {
        StatelessSession ss = sessionFactory.openStatelessSession();
        Transaction tx = ss.beginTransaction();
        try {
            for (ImportDTO dto : data) {
                Product product = fastConvert(dto);
                ss.insert(product);
            }
            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        } finally {
            ss.close();
        }
    }

    // PostgreSQL COPY 示例（超大数据量）
    public void importViaDatabaseTool(List<ImportDTO> data) {
        // 1. 将数据写入 CSV 文件
        // 2. 执行: COPY products FROM '/tmp/products.csv' WITH CSV HEADER;
        // 对于 MySQL: LOAD DATA INFILE '/tmp/products.csv' INTO TABLE products
    }
}
```

---

# 7. 安全性最佳实践

## 7.1 SQL 注入防范

### 7.1.1 参数化查询

```java
// ❌ 危险：字符串拼接
String username = request.getParameter("username");
Query query = entityManager.createQuery(
    "SELECT u FROM User u WHERE u.username = '" + username + "'");
// 输入: ' OR '1'='1' --
// 结果: SELECT u FROM User u WHERE u.username = '' OR '1'='1' --'

// ✅ 安全：参数绑定（HQL/JPQL）
String username = request.getParameter("username");
TypedQuery<User> query = entityManager.createQuery(
    "SELECT u FROM User u WHERE u.username = :username", User.class);
query.setParameter("username", username);

// ✅ 安全：参数绑定（原生 SQL）
String username = request.getParameter("username");
Query query = entityManager.createNativeQuery(
    "SELECT * FROM users WHERE username = :username", User.class);
query.setParameter("username", username);

// ✅ Spring Data JPA 自动参数绑定
@Query("SELECT u FROM User u WHERE u.username = :username")
Optional<User> findByUsername(@Param("username") String username);
```

### 7.1.2 动态排序字段的白名单校验

```java
// ❌ 危险：ORDER BY 参数无法通过 setParameter 绑定
@Query("SELECT p FROM Product p ORDER BY :sortField :sortDirection")
List<Product> findAllSorted(@Param("sortField") String field,
                             @Param("sortDirection") String dir);
// JPQL/HQL 不支持 ORDER BY 参数绑定

// ✅ 安全：白名单校验
private static final Set<String> ALLOWED_SORT_FIELDS = Set.of("id", "name", "price", "createdDate");
private static final Set<String> ALLOWED_DIRECTIONS = Set.of("ASC", "DESC");

public List<Product> findAllSorted(String sortField, String sortDirection) {
    if (sortField == null || !ALLOWED_SORT_FIELDS.contains(sortField)) {
        throw new IllegalArgumentException("非法的排序字段: " + sortField);
    }
    if (sortDirection == null || !ALLOWED_DIRECTIONS.contains(sortDirection.toUpperCase())) {
        throw new IllegalArgumentException("非法的排序方向: " + sortDirection);
    }
    // 白名单通过后安全拼装
    return entityManager.createQuery(
        "SELECT p FROM Product p ORDER BY p." + sortField + " " + sortDirection.toUpperCase(),
        Product.class).getResultList();
}

// ✅ 更好：使用 Criteria API 完全消除字符串拼接
public List<Product> findAllSortedSafe(String sortField, String sortDirection) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<Product> cq = cb.createQuery(Product.class);
    Root<Product> root = cq.from(Product.class);

    if ("ASC".equalsIgnoreCase(sortDirection)) {
        cq.orderBy(cb.asc(root.get(sortField)));  // root.get() 内部使用元模型，安全
    } else {
        cq.orderBy(cb.desc(root.get(sortField)));
    }
    return entityManager.createQuery(cq).getResultList();
}
```

### 7.1.3 动态查询使用 Criteria API

```java
// ✅ Criteria API — 无字符串拼接
public List<User> searchUsers(UserSearchCriteria criteria) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<User> cq = cb.createQuery(User.class);
    Root<User> root = cq.from(User.class);

    List<Predicate> predicates = new ArrayList<>();

    if (criteria.getName() != null) {
        predicates.add(cb.like(root.get("name"), "%" + criteria.getName() + "%"));
    }
    if (criteria.getStatus() != null) {
        predicates.add(cb.equal(root.get("status"), criteria.getStatus()));
    }
    if (criteria.getMinAge() != null) {
        predicates.add(cb.greaterThanOrEqualTo(root.get("age"), criteria.getMinAge()));
    }

    cq.where(predicates.toArray(new Predicate[0]));
    return entityManager.createQuery(cq).getResultList();
}
```

## 7.2 敏感数据保护

### 7.2.1 数据库密码加密

```java
// Jasypt 配置文件加密
// application.yml
spring:
  datasource:
    password: ENC(encrypted_password_here)

// Jasypt 配置
@Configuration
public class JasyptConfig {

    @Bean
    public static StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword(System.getenv("JASYPT_ENCRYPTOR_PASSWORD"));  // 密钥来自环境变量
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setPoolSize("1");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

### 7.2.2 用户密码不可逆加密

```java
// ✅ BCrypt — 不可逆哈希
@Service
public class UserService {

    private final PasswordEncoder passwordEncoder = new BCryptPasswordEncoder();

    public User createUser(CreateUserRequest request) {
        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));  // BCrypt 哈希
        return userRepository.save(user);
    }

    public boolean verifyPassword(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}
```

### 7.2.3 实体序列化排除敏感字段

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;

    @JsonIgnore  // Jackson 序列化时排除
    private String password;

    @JsonIgnore
    private String ssn;  // 社会安全号码

    @Transient  // 不持久化到数据库
    private String temporaryToken;

    @OneToMany(mappedBy = "user")
    @JsonIgnore  // 序列化时排除关联，避免循环和泄露
    private Set<Order> orders;
}
```

### 7.2.4 日志脱敏

```java
@Component
public class SensitiveDataMaskingInterceptor extends EmptyInterceptor {

    @Override
    public String onPrepareStatement(String sql) {
        // ⚠️ 生产环境不要开启参数绑定日志，或使用专门的脱敏方案
        return sql;
    }

    // 更推荐：日志配置层面控制
    // application-prod.yml
    // logging.level.org.hibernate.type.descriptor.sql.BasicBinder: OFF
}
```

```java
// ✅ DTO 日志脱敏
@Service
public class PaymentService {

    public void processPayment(PaymentRequest request) {
        log.info("处理支付: orderId={}, amount={}, cardNo={}",
            request.getOrderId(),
            request.getAmount(),
            maskCardNumber(request.getCardNumber()));  // ****1234
    }

    private String maskCardNumber(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) return "****";
        return "****" + cardNumber.substring(cardNumber.length() - 4);
    }
}
```

---

# 8. 常见错误与反模式

## 8.1 十大常见错误

### 8.1.1 错误 1：@ManyToOne/@OneToOne 未设 LAZY

```java
// ❌ N+1 查询爆炸
@Entity
public class OrderItem {
    @ManyToOne  // EAGER！加载 100 个 OrderItem = 1 + 100 条 SQL
    private Product product;
}

// ✅ 修复
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;
}
```

### 8.1.2 错误 2：双向关联只更新一方

```java
// ❌ 导致内存与数据库数据不一致
Order order = new Order();
order.setCustomer(customer);
orderRepository.save(order);
// customer.getOrders() 中没有这个 order → 同一个 Session 内数据不一致

// ✅ 使用同步方法
customer.addOrder(order);
orderRepository.save(order);
```

### 8.1.3 错误 3：Set/Map 中实体未实现 equals/hashCode

```java
// ❌ 重复数据
@OneToMany(mappedBy = "order", cascade = ALL, orphanRemoval = true)
private Set<OrderItem> items = new HashSet<>();

// 如果 OrderItem 未正确实现 equals/hashCode
// session.merge(order) 时，Set 认为所有项都是"新"的 → 重复 INSERT

// ✅ 必须在 OrderItem 中实现 equals/hashCode（基于业务键或 id）
```

### 8.1.4 错误 4：session.save/update 替代 persist/merge

```java
// ❌ Hibernate 专有 API，迁移困难
Session session = entityManager.unwrap(Session.class);
session.save(entity);    // Hibernate API
session.update(entity);  // Hibernate API

// ✅ JPA 标准 API
entityManager.persist(entity);  // 插入
entityManager.merge(entity);    // 更新/保存
```

### 8.1.5 错误 5：事务外部访问懒加载属性

```java
// ❌ LazyInitializationException
@Transactional(readOnly = true)
public UserDTO getUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    return userMapper.toDto(user);  // toDto 中需要访问 user.orders → 懒加载触发
}
// 方法返回后事务结束 → Session 关闭 → LazyInitializationException

// ✅ 解决方案一：事务内完成所有懒加载访问
@Transactional(readOnly = true)
public UserDTO getUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    UserDTO dto = userMapper.toDto(user);   // 在事务内完成
    return dto;
}

// ✅ 解决方案二：使用 JOIN FETCH 在查询时加载关联
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);
```

### 8.1.6 错误 6：merge() 返回值不使用

```java
// ❌ merge() 返回新托管对象，但错误地继续使用旧的游离对象
@Transactional
public void updateUser(User detachedUser) {
    entityManager.merge(detachedUser);  // 返回托管实体，但被忽略
    // detachedUser 仍是游离对象，不参与脏检查
    // 后续修改 detachedUser 不会持久化！
}

// ✅ 使用 merge() 返回的托管实体
@Transactional
public User updateUser(User detachedUser) {
    User managedUser = entityManager.merge(detachedUser);  // 获取托管实体
    managedUser.setLastModifiedDate(LocalDateTime.now());   // 托管实体的修改会持久化
    return managedUser;
}
```

### 8.1.7 错误 7：JOIN FETCH 与分页同时使用

```java
// ❌ HHH000104 警告 + 全量加载内存分页
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);

// ✅ 两步查询法（参见 3.3 节）
```

### 8.1.8 错误 8：多个 List（Bag）同时 JOIN FETCH

```java
// ❌ MultipleBagFetchException
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    private List<Employee> employees;  // Bag

    @OneToMany(mappedBy = "department")
    private List<Project> projects;    // 另一个 Bag
}

// ❌ 不能同时 FETCH 两个 Bag
@Query("SELECT d FROM Department d JOIN FETCH d.employees JOIN FETCH d.projects")
List<Department> findAllDepts();

// ✅ 修复方案一：将其中一个 List 改为 Set
@OneToMany(mappedBy = "department")
private Set<Employee> employees;  // Set

@OneToMany(mappedBy = "department")
private List<Project> projects;   // 保留一个 Bag（但推荐也改为 Set）

// ✅ 修复方案二：分开查询，使用 @BatchSize 加载第二个集合
@Query("SELECT d FROM Department d JOIN FETCH d.employees")
List<Department> findAllWithEmployees();
// 通过 @BatchSize(size = 50) 在 departments.projects 上批量加载
```

### 8.1.9 错误 9：HQL 批量更新/删除后不清除 Session 缓存

```java
// ❌ 脏数据问题
@Transactional
public void deactivateExpiredUsers(LocalDateTime threshold) {
    // HQL 批量更新 — 绕过一级缓存
    entityManager.createQuery(
        "UPDATE User u SET u.active = false WHERE u.lastLogin < :threshold")
        .setParameter("threshold", threshold)
        .executeUpdate();  // 不更新一级缓存中的实体

    // 一级缓存中的 User 实体 active 仍为 true！
    User cachedUser = entityManager.find(User.class, 1L);
    // cachedUser.active == true（一级缓存中的旧值）
}

// ✅ 批量更新后清除缓存
@Transactional
public void deactivateExpiredUsers(LocalDateTime threshold) {
    entityManager.createQuery(
        "UPDATE User u SET u.active = false WHERE u.lastLogin < :threshold")
        .setParameter("threshold", threshold)
        .executeUpdate();

    entityManager.flush();
    entityManager.clear();  // 清空一级缓存，后续查询重新从数据库加载
}
```

### 8.1.10 错误 10：生产环境使用 ddl-auto=update

```yaml
# ❌ 生产环境危险配置
spring:
  jpa:
    hibernate:
      ddl-auto: update  # 自动修改 Schema，不可控！

# ✅ 生产环境安全配置
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # 仅校验映射与表结构一致性，不一致时启动失败
      # 或 ddl-auto: none — 不做任何操作
```

> Schema 变更必须通过数据库迁移工具（Flyway / Liquibase）执行，记录版本并可回滚。

## 8.2 反模式列表

### 8.2.1 反模式 1：DAO 层直接暴露实体给 Controller

```java
// ❌ 反模式
@RestController
public class UserController {

    @Autowired
    private UserRepository userRepository;  // Repository 直接注入 Controller

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {  // 返回实体！
        return userRepository.findById(id).orElseThrow();
        // 问题：
        // 1. 密码字段暴露（除非 @JsonIgnore）
        // 2. 懒加载关联无法序列化（Session 已关闭）
        // 3. 序列化所有字段，API 不稳定
    }
}
```

### 8.2.2 反模式 2：实体中写业务逻辑（务实讨论）

```java
// ❌ 过度"充血模型"的问题
@Entity
public class Order {
    // 实体需要依赖外部服务时：
    @Transient
    private EmailService emailService;  // 实体依赖外部服务 → 难以注入

    public void cancel() {
        this.setStatus(OrderStatus.CANCELLED);
        emailService.sendCancellationEmail(this);  // 实体依赖基础设施
        // 问题：Hibernate 管理的 entity 不应持有外部依赖
    }
}

// ✅ 务实贫血模型 + 领域服务
@Entity
public class Order {
    // 仅包含数据 + 自己状态相关的简单方法
    public boolean isCancelable() {
        return this.status == OrderStatus.PENDING
            && this.createdDate.isAfter(LocalDateTime.now().minusDays(1));
    }

    public void markAsCancelled() {
        if (!isCancelable()) {
            throw new IllegalStateException("订单不可取消");
        }
        this.status = OrderStatus.CANCELLED;
    }
}

@Service  // 业务逻辑在 Service 层
public class OrderCancellationService {
    private final EmailService emailService;

    @Transactional
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.markAsCancelled();               // 实体自身状态变更
        emailService.sendCancellationEmail(order);  // 外部依赖由 Service 处理
    }
}
```

### 8.2.3 反模式 3：过度使用 CascadeType.ALL

```java
// ❌ 危险级联
@Entity
public class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private Set<Order> orders;
    // 删除 User 时，级联删除所有 Order（可能不想这样）
}

// ✅ 明确级联范围
@Entity
public class User {
    @OneToMany(mappedBy = "user", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    private Set<Order> orders;
    // 删除 User 时需要先手动处理 Orders
}
```

### 8.2.4 反模式 4：IDENTITY + 批处理

```java
// ❌ IDENTITY 禁用 JDBC 批处理
@Entity
public class LogEntry {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;  // MySQL AUTO_INCREMENT
}

// 批量 INSERT 1000 条 = 1000 条独立的 INSERT 语句（无法批量）

// ✅ PostgreSQL/Oracle：使用 SEQUENCE
@Entity
public class LogEntry {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "log_seq")
    @SequenceGenerator(name = "log_seq", sequenceName = "log_seq", allocationSize = 50)
    private Long id;
}
// 批量 INSERT = INSERT INTO ... VALUES (...), (...), (...) ...
```

### 8.2.5 反模式 5：所有 @OneToMany 都用 List（Bag）

参见 2.5 节 Bag 问题。

### 8.2.6 反模式 6：不设置 hibernate.jdbc.batch_size

```yaml
# ❌ 默认 batch_size = 0（禁用批处理）
# 每条 INSERT/UPDATE/DELETE 单独的数据库往返

# ✅ 必须设置
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

---

# 9. 测试最佳实践

## 9.1 ORM 行为测试

### 9.1.1 使用 @DataJpaTest 隔离数据库层

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Autowired
    private TestEntityManager testEntityManager;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldSaveAndFindOrder() {
        // 使用 TestEntityManager 快速设置测试数据
        Customer customer = new Customer();
        customer.setEmail("test@example.com");
        testEntityManager.persistAndFlush(customer);

        Order order = new Order();
        order.setCustomer(customer);
        order.setTotalAmount(new BigDecimal("99.90"));
        Order saved = orderRepository.save(order);

        Optional<Order> found = orderRepository.findById(saved.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getTotalAmount()).isEqualByComparingTo(new BigDecimal("99.90"));
    }
}
```

### 9.1.2 测试 N+1 问题

```java
// 使用 hypersistence-utils 断言 SQL 执行次数
@Test
public void testFindAllWithBooks_noNPlusOne() {
    // 准备数据：5 个 Author，每个 3 本 Book
    for (int i = 0; i < 5; i++) {
        Author author = new Author("Author " + i);
        for (int j = 0; j < 3; j++) {
            author.addBook(new Book("Book " + i + "-" + j));
        }
        testEntityManager.persistAndFlush(author);
    }

    SQLStatementCountValidator.reset();

    List<Author> authors = authorRepository.findAllWithBooks();

    // 期望：1 条 JOIN FETCH 查询（或 1 + 1 条 batch 查询）
    assertSelectCount(5);  // 如果未优化，会触发 N+1，此处应失败
    // 正确优化后，应该是 1 或 2 条 SELECT

    assertThat(authors).hasSize(5);
    // 确保在 SQL 断言之后访问懒加载属性（否则额外 SQL 会被计算在内）
    authors.forEach(author ->
        assertThat(author.getBooks()).hasSize(3));
}
```

### 9.1.3 测试懒加载行为

```java
@Test
public void testOrderItemProductLazyLoading() {
    // 准备
    Product product = new Product("Test Product");
    testEntityManager.persistAndFlush(product);

    OrderItem item = new OrderItem();
    item.setProduct(product);
    testEntityManager.persistAndFlush(item);

    // 清除持久化上下文，强制从数据库加载
    testEntityManager.clear();

    // 查询 OrderItem（不 JOIN FETCH product）
    OrderItem loaded = testEntityManager.find(OrderItem.class, item.getId());
    // 此时 product 是懒加载代理对象，未触发 SQL

    assertThat(Hibernate.isInitialized(loaded.getProduct())).isFalse();

    // 访问 product 触发 SQL
    String productName = loaded.getProduct().getName();

    assertThat(Hibernate.isInitialized(loaded.getProduct())).isTrue();
    assertThat(productName).isEqualTo("Test Product");
}
```

### 9.1.4 测试级联操作

```java
@Test
public void testCascadePersist() {
    Order order = new Order();
    OrderItem item1 = new OrderItem("Product A");
    OrderItem item2 = new OrderItem("Product B");
    order.addItem(item1);
    order.addItem(item2);

    // 仅持久化 Order
    Order saved = orderRepository.save(order);

    // 清除并重新加载，验证级联持久化
    testEntityManager.clear();
    Order reloaded = testEntityManager.find(Order.class, saved.getId());
    assertThat(reloaded.getItems()).hasSize(2);
}

@Test
public void testOrphanRemoval() {
    Order order = new Order();
    OrderItem item = new OrderItem("Product X");
    order.addItem(item);
    Long orderId = testEntityManager.persistAndFlush(order).getId();

    testEntityManager.clear();

    // 删除孤儿
    Order reloaded = testEntityManager.find(Order.class, orderId);
    OrderItem orphanItem = reloaded.getItems().iterator().next();
    reloaded.removeItem(orphanItem);
    testEntityManager.flush();

    testEntityManager.clear();

    // 验证孤儿被删除
    Order afterRemoval = testEntityManager.find(Order.class, orderId);
    assertThat(afterRemoval.getItems()).isEmpty();
}
```

## 9.2 集成测试模式

### 9.2.1 测试独立性

```java
@SpringBootTest
@Transactional  // 每个测试方法在一个事务中执行，完成后自动回滚
class OrderServiceIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private TestEntityManager testEntityManager;

    @Test
    void testCreateOrder() {
        // 每个测试独立，互不影响
        CreateOrderRequest request = new CreateOrderRequest();
        request.setCustomerId(1L);
        request.addItem("Product-A", 2);

        OrderDTO result = orderService.createOrder(request);

        assertThat(result.getId()).isNotNull();
        assertThat(result.getItems()).hasSize(1);
    }
    // 测试结束时自动回滚，不污染数据库
}
```

### 9.2.2 @Sql 注入测试数据

```java
@SpringBootTest
@Sql(scripts = "/sql/init-test-data.sql")  // 测试前执行
@Sql(scripts = "/sql/cleanup.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)  // 测试后清理
class ProductServiceTest {

    @Autowired
    private ProductService productService;

    @Test
    void testFindActiveProducts() {
        List<ProductDTO> products = productService.findActiveProducts();
        assertThat(products).isNotEmpty();
    }
}

// src/test/resources/sql/init-test-data.sql
// INSERT INTO product (id, name, price, active) VALUES (1, 'Test Product A', 19.99, true);
// INSERT INTO product (id, name, price, active) VALUES (2, 'Test Product B', 29.99, false);
```

### 9.2.3 避免测试间数据污染

```java
// ✅ 方案一：@Transactional + 自动回滚（推荐）
@SpringBootTest
@Transactional
class MyServiceTest {
    // 每个测试完成后自动回滚，数据不持久化
}

// ✅ 方案二：@DirtiesContext 重置 Spring 上下文
@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
class HeavyweightTest {
    // 适用于有复杂缓存状态的测试（但较慢）
}

// ✅ 方案三：每次测试前 TRUNCATE TABLE
@BeforeEach
void setUp() {
    testEntityManager.getEntityManager()
        .createNativeQuery("TRUNCATE TABLE orders CASCADE").executeUpdate();
    testEntityManager.getEntityManager()
        .createNativeQuery("TRUNCATE TABLE products CASCADE").executeUpdate();
}
```

## 9.3 性能测试

```java
@SpringBootTest
class HibernatePerformanceTest {

    @Autowired
    private EntityManagerFactory emf;

    @BeforeEach
    void setUp() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        sf.getStatistics().setStatisticsEnabled(true);
    }

    @AfterEach
    void tearDown() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();
        log.info("=== 测试统计 ===");
        log.info("SQL 执行次数: {}", stats.getQueryExecutionCount());
        log.info("二级缓存命中: {}", stats.getSecondLevelCacheHitCount());
        log.info("二级缓存未命中: {}", stats.getSecondLevelCacheMissCount());
        log.info("Session 打开数: {}", stats.getSessionOpenCount());
        log.info("事务数: {}", stats.getTransactionCount());
        stats.clear();
    }

    @Test
    void testQueryPerformance() {
        // ... 执行业务操作

        Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
        assertThat(stats.getQueryExecutionCount()).isLessThanOrEqualTo(3);
    }
}

// 慢查询检测
@Test
void testNoSlowQuery() {
    long start = System.currentTimeMillis();
    List<OrderSummary> results = orderRepository.findOrderSummaries(
        LocalDate.now().minusMonths(1), LocalDate.now());
    long elapsed = System.currentTimeMillis() - start;

    assertThat(elapsed).isLessThan(1000);  // 必须 1 秒内完成
    assertThat(results).isNotEmpty();
}
```

---

# 10. 项目结构建议

## 10.1 包结构

```
com.example.project
├── entity/                     # 实体类（JPA 映射模型）
│   ├── User.java
│   ├── Order.java
│   ├── Product.java
│   └── base/                   # 实体基类
│       └── BaseEntity.java
│
├── repository/                 # 数据访问层（Spring Data JPA）
│   ├── UserRepository.java
│   ├── OrderRepository.java
│   └── custom/                 # 自定义 Repository 实现
│       ├── OrderRepositoryCustom.java
│       └── OrderRepositoryImpl.java
│
├── dto/                        # 数据传输对象
│   ├── request/                # 请求 DTO
│   │   └── CreateOrderRequest.java
│   ├── response/               # 响应 DTO
│   │   └── OrderDTO.java
│   └── query/                  # 查询 DTO（投影）
│       └── OrderSummary.java
│
├── mapper/                     # Entity <-> DTO 转换器
│   ├── OrderMapper.java
│   └── UserMapper.java
│
├── service/                    # 业务逻辑层
│   ├── OrderService.java
│   ├── UserService.java
│   └── impl/                   # 可选：接口实现
│       └── OrderServiceImpl.java
│
├── config/                     # Hibernate/JPA 配置
│   ├── JpaConfig.java
│   ├── HibernateProperties.java
│   └── AuditorConfig.java
│
├── enums/                      # 持久化枚举类型
│   ├── OrderStatus.java
│   └── UserRole.java
│
├── types/                      # 自定义 Hibernate 类型
│   └── JsonType.java
│
├── listener/                   # JPA Entity Listeners
│   └── AuditEntityListener.java
│
├── interceptor/                # Hibernate Interceptors
│   └── SlowQueryInterceptor.java
│
└── exception/                  # 领域异常
    ├── EntityNotFoundException.java
    └── OptimisticLockException.java
```

## 10.2 分层职责

| 层 | 职责 | 不应做什么 |
|---|------|-----------|
| **Controller** | 接收 HTTP 请求、参数校验（`@Valid`）、调用 Service、返回 ResponseEntity | 不包含业务逻辑、不直接访问 Repository |
| **Service** | 业务逻辑编排、事务管理（`@Transactional`）、权限校验 | 不处理 HTTP 相关对象 |
| **Repository** | 数据持久化与查询、提供 `Page` 分页支持 | 不包含业务逻辑、不抛出业务异常 |
| **Entity** | 数据模型定义、JPA 映射注解、简单状态校验 | 不依赖 Service/Repository、不包含基础设施代码 |
| **DTO** | 数据格式定义、Jackson 注解、校验注解（`@NotNull` 等） | 不包含业务逻辑、不与数据库交互 |
| **Mapper** | Entity ↔ DTO 双向转换 | 不含业务逻辑 |

```java
// Controller：薄层
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        OrderDTO result = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrder(id));
    }
}

// Service：业务逻辑 + 事务
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final OrderMapper orderMapper;

    @Transactional
    public OrderDTO createOrder(CreateOrderRequest request) {
        // 1. 业务验证
        Product product = productRepository.findById(request.getProductId())
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
        if (product.getStock() < request.getQuantity()) {
            throw new InsufficientStockException();
        }

        // 2. 创建实体
        Order order = new Order();
        order.setProduct(product);
        order.setQuantity(request.getQuantity());
        // ... 业务计算

        // 3. 持久化
        Order saved = orderRepository.save(order);

        // 4. 返回 DTO
        return orderMapper.toDto(saved);
    }

    @Transactional(readOnly = true)
    public OrderDTO getOrder(Long id) {
        return orderRepository.findById(id)
            .map(orderMapper::toDto)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
    }
}

// Repository：数据访问
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerId(Long customerId);

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);
}

// Entity：纯数据模型
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    private Integer quantity;
    private LocalDateTime createdDate;

    @Version
    private Long version;

    // 仅包含实体自身状态的简单方法（不依赖外部服务）
    public void validate() {
        if (quantity == null || quantity <= 0) {
            throw new IllegalArgumentException("数量必须大于 0");
        }
    }
}
```

## 10.3 依赖方向

```
┌──────────────────────────────────────────┐
│                Controller                │
│          (接收请求、参数校验)              │
└──────────────┬───────────────────────────┘
               │ 依赖（注入 Service）
               ▼
┌──────────────────────────────────────────┐
│                 Service                   │
│          (业务逻辑、事务边界)              │
└──────────────┬──────────────┬────────────┘
               │              │ 依赖（注入 Mapper）
               ▼              ▼
┌──────────────────┐ ┌──────────────────┐
│   Repository     │ │     Mapper       │
│  (数据访问)       │ │ (Entity ↔ DTO)  │
└────────┬─────────┘ └─────┬────────────┘
         │                 │ 依赖（读取字段）
         ▼                 ▼
┌──────────────────────────────────────────┐
│               Entity                     │
│        (持久化模型、数据定义)              │
└──────────────────────────────────────────┘

┌──────────────────┐
│      DTO         │  ← 在 Controller 和 Service 之间传递
│  (表现层数据)     │     定义在 Service 接口的返回值/参数
└──────────────────┘

依赖规则：
✅ Controller → Service → Repository → Entity
✅ Service → Mapper → Entity (Mapper 读取 Entity 字段)
✅ Service → Mapper → DTO   (Mapper 组装 DTO)
❌ Entity → Repository (禁止)
❌ Entity → Service    (禁止)
❌ Entity → Controller (禁止)
❌ DTO → Repository    (禁止)
```

> **核心原则**：依赖方向从外向内。外层（Controller、Service）依赖内层（Entity、Repository），内层不依赖外层。这确保核心领域逻辑的解耦和可测试性。

---

## 10.4 参考资源

| 资源 | 说明 |
|------|------|
| Hibernate ORM User Guide | 官方文档 [hibernate.org/orm/documentation](https://hibernate.org/orm/documentation/) |
| Vlad Mihalcea's Blog | Hibernate 深度文章 [vladmihalcea.com](https://vladmihalcea.com/) |
| Thorben Janssen's Thoughts on Java | Hibernate 教程 [thorben-janssen.com](https://thorben-janssen.com/) |
| Spring Data JPA Reference | Spring 官方文档 |
| Hypersistence Utils | Hibernate 测试/工具库 [github.com/vladmihalcea/hypersistence-utils](https://github.com/vladmihalcea/hypersistence-utils) |
