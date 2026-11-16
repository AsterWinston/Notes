# 1. 实体类基本要求

## 1.1 实体类规范

在 Hibernate/JPA 中，一个 POJO 要成为实体类，必须满足以下规范：

- **必须使用 `@Entity` 注解**：标记该类为 JPA 实体，由 JPA 提供者管理其生命周期。
- **必须有无参构造方法**（public 或 protected）：Hibernate 在通过反射实例化实体时需要调用无参构造方法，且不能是 private。
- **不能是 final 类**：Hibernate 使用 CGLIB/ByteBuddy 代理技术来实现懒加载和脏检查，final 类无法被代理。
- **需实现 `Serializable`**（可选但推荐）：当实体需要存储在二级缓存（如 Ehcache、Hazelcast）中或通过 RMI/HTTP Session 传输时，必须实现该接口。即便暂无此需求，也建议实现，以避后续重构成本。
- **持久化字段不能是 final**：代理或反射无法修改 final 字段。
- **实体类不能是内部类**（除非是 static 嵌套类）：普通内部类隐式持有外部类引用，其构造函数依赖外部类实例，Hibernate 反射实例化会出问题。

```java
@Entity
public class User implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    
    // 必须有无参构造方法
    public User() {}
    
    // 可以有带参构造方法
    public User(String username) {
        this.username = username;
    }
    
    // getter / setter 省略
}
```

## 1.2 字段访问 vs 属性访问

Hibernate 访问实体状态的方式由 `@Id` 注解所在位置决定：

- **Field Access**（`@Id` 标注在字段上）：Hibernate 通过反射直接读取/写入字段值，绕过 getter/setter，字段可以是 private。
- **Property Access**（`@Id` 标注在 getter 方法上）：Hibernate 通过 getter/setter 方法访问属性，允许在 setter 中做额外的业务逻辑（如校验、转换）。

```java
// Field Access：@Id 在字段上
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private BigDecimal amount;
    
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
}

// Property Access：@Id 在 getter 上
@Entity
public class Product {
    private Long id;
    private String name;
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long getId() { return id; }
    
    public void setId(Long id) { this.id = id; }
    
    @Column(name = "product_name")
    public String getName() { return name; }
    
    public void setName(String name) {
        // 可在 setter 中添加业务逻辑
        this.name = (name != null) ? name.trim().toUpperCase() : null;
    }
}
```

**`@Access` 显式指定访问类型**：可以在同一个实体中混合使用 Field Access 和 Property Access。

```java
@Entity
@Access(AccessType.FIELD)  // 默认字段访问
public class Employee {
    
    @Id
    private Long id;
    
    @Transient
    private String fullName;  // 不持久化此字段
    
    private String firstName;
    private String lastName;
    
    @Access(AccessType.PROPERTY)  // 此属性使用 getter 访问
    public String getEmployeeCode() {
        return "EMP-" + id;
    }
}
```

| 对比维度 | Field Access | Property Access |
|---------|-------------|----------------|
| 访问方式 | 反射直接访问字段（含 private） | 通过 getter/setter |
| 额外逻辑 | 无法在字段读写时执行额外逻辑 | 可在 setter 中添加校验/转换逻辑 |
| 注解位置 | 注解标注在字段上 | 注解标注在 getter 上 |
| 代理影响 | 字段级别代理，不影响方法调用 | getter 调用可能触发代理逻辑 |
| 推荐度 | **推荐**（更简洁直观） | 特定需求时使用 |

## 1.3 equals() / hashCode() 实现

### 1.3.1 为什么需要正确实现

在 ORM 中，实体的标识符有两种含义：
- **数据库标识**（数据库主键，即 `@Id`）：在数据库中唯一标识一行记录。
- **对象标识**（Java 内存地址，即 `==`）：在 JVM 堆中唯一标识一个对象。

当实体被放入 `Set` 或作为 `Map` 的 key 时，默认的 `equals()` 基于对象内存地址（`==`），这会导致：同一个数据库行对应的两个不同 Java 实例（如从不同 Session 加载）在集合中被视为不同元素，产生重复数据。

```java
// 问题示例
Session session1 = sessionFactory.openSession();
User user1 = session1.get(User.class, 1L);
session1.close();

Session session2 = sessionFactory.openSession();
User user2 = session2.get(User.class, 1L);
session2.close();

Set<User> users = new HashSet<>();
users.add(user1);
users.add(user2);
System.out.println(users.size()); // 2（预期为 1），因为 user1.equals(user2) 为 false
```

### 1.3.2 基于业务键（Business Key / Natural Id）的实现

使用业务上不可变的自然键实现 `equals()/hashCode()` —— **推荐方式**：

```java
@Entity
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;  // 业务键（自然标识符）
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return username != null && username.equals(user.username);
    }
    
    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

### 1.3.3 基于数据库标识符的实现及问题

```java
// 基于 @Id 的实现（有隐患）
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;
    User user = (User) o;
    return id != null && id.equals(user.id);
}

@Override
public int hashCode() {
    return getClass().hashCode();  // 固定值
}
```

**问题所在**：
- 当实体处于 `Transient` 状态（`id == null`，尚未持久化）时，两个不同的新实体将 "相等"。
- 此时若放入 `HashSet`，后添加的实体会被视为重复而丢失。
- `hashCode()` 返回固定值，导致哈希冲突，所有实体落入同一个 hash bucket，集合操作退化为 O(n)。

### 1.3.4 推荐实现模式

```java
@Entity
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NaturalId
    @Column(nullable = false, unique = true)
    private String username;
    
    private String email;
    
    // 推荐：基于 @NaturalId 的 equals/hashCode
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(username, user.username);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(username);
    }
    
    // 备选：使用 Objects.equals / Objects.hash
}
```

# 2. 表映射

## 2.1 @Table 注解

`@Table` 用于指定实体与数据库表的映射细节，当类名与表名不一致时必须使用。

```java
@Entity
@Table(name = "t_user",                          // 表名
       catalog = "app_db",                       // 数据库 catalog
       schema = "public",                        // 数据库 schema
       uniqueConstraints = {
           @UniqueConstraint(name = "uk_username", columnNames = {"username"}),
           @UniqueConstraint(name = "uk_email_phone", columnNames = {"email", "phone"})
       },
       indexes = {
           @Index(name = "idx_create_time", columnList = "create_time"),
           @Index(name = "idx_status_time", columnList = "status, create_time DESC")
       }
)
public class User {
    @Id
    private Long id;
    private String username;
    private String email;
    private String phone;
    private String status;
    private LocalDateTime createTime;
}
```

**命名策略交互**：Hibernate 默认的 `PhysicalNamingStrategy` 会将 camelCase 转换为 snake_case。可通过配置替换：

```properties
# application.properties
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
# 或自定义实现
spring.jpa.hibernate.naming.physical-strategy=com.example.CustomNamingStrategy
```

```java
// 自定义命名策略（Hibernate 6）
public class CustomNamingStrategy extends CamelCaseToUnderscoresNamingStrategy {
    // 默认 CamelCaseToUnderscoresNamingStrategy 就是驼峰转下划线
    // 可重写 toPhysicalTableName / toPhysicalColumnName
}
```

## 2.2 @SecondaryTable / @SecondaryTables

当一个实体的字段需要分散存储在**多张表**中时使用。通常是因为历史遗留数据库设计或特定存储策略（如大字段分离）。

```java
@Entity
@Table(name = "t_customer")
@SecondaryTable(name = "t_customer_detail",
                pkJoinColumns = @PrimaryKeyJoinColumn(name = "customer_id", 
                                                       referencedColumnName = "id"))
public class Customer {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;                 // 存储在主表 t_customer
    
    @Column(table = "t_customer_detail")
    private String address;              // 存储在从表 t_customer_detail
    
    @Column(table = "t_customer_detail")
    private String phone;                // 存储在从表 t_customer_detail
    
    @Column(table = "t_customer_detail", columnDefinition = "TEXT")
    private String remarks;              // 大字段放从表，减少主表 IO
}
```

**多张从表**：

```java
@Entity
@Table(name = "t_employee")
@SecondaryTables({
    @SecondaryTable(name = "t_employee_salary", 
                    pkJoinColumns = @PrimaryKeyJoinColumn(name = "emp_id")),
    @SecondaryTable(name = "t_employee_contact", 
                    pkJoinColumns = @PrimaryKeyJoinColumn(name = "emp_id"))
})
public class Employee {
    @Id private Long id;
    private String name;
    
    @Column(table = "t_employee_salary")
    private BigDecimal salary;
    
    @Column(table = "t_employee_contact")
    private String email;
}
```

**查询行为**：Hibernate 默认使用 `LEFT JOIN` 关联所有从表来获取完整实体。这会导致：
- 总是加载所有表的数据，增加不必要的 IO。
- 若从表字段不需要，建议拆分为独立实体 + `@OneToOne` 关联并设为懒加载。

# 3. 主键映射

## 3.1 @Id

`@Id` 标记实体主键，支持简单主键和复合主键，支持以下类型：
- 基本类型：`byte`, `short`, `int`, `long`, `char` 及其包装类
- `String`
- `java.util.Date`, `java.sql.Date`
- `java.math.BigInteger`, `java.math.BigDecimal`
- `java.util.UUID`

```java
@Entity
public class Book {
    @Id
    private String isbn;  // String 作为自然主键
}
```

## 3.2 @GeneratedValue 生成策略

### 3.2.1 GenerationType.AUTO

由 JPA 提供者根据 Dialect 自动选择最合适的策略：

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```

| 数据库 | Hibernate AUTO 实际策略 |
|--------|----------------------|
| MySQL / MariaDB | `IDENTITY`（自增） |
| PostgreSQL | `SEQUENCE`（序列） |
| Oracle 11g- | `SEQUENCE`（序列） |
| Oracle 12c+ | `SEQUENCE`（序列，也可用 IDENTITY） |
| H2 / HSQLDB | `SEQUENCE` |
| SQL Server | `IDENTITY` |
| DB2 | `SEQUENCE` |

### 3.2.2 GenerationType.IDENTITY

自增列，由数据库自动分配值：

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

底层原理：
- MySQL：`AUTO_INCREMENT`
- PostgreSQL：`SERIAL` / `BIGSERIAL` / `GENERATED AS IDENTITY`
- SQL Server：`IDENTITY(1,1)`

**关键限制**：`IDENTITY` 策略下，Hibernate 必须在 `INSERT` 执行后立即获取生成的主键（通过 `Statement.getGeneratedKeys()`），因此无法使用 JDBC 批量插入优化（`addBatch()` + `executeBatch()`）。如果需要高性能批量插入，应避免使用此策略。

```java
// 使用 SEQUENCE 或 TABLE 策略才支持 JDBC 批处理
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq_user")
@SequenceGenerator(name = "seq_user", sequenceName = "user_seq", allocationSize = 50)
private Long id;
```

### 3.2.3 GenerationType.SEQUENCE

使用数据库序列对象生成主键，支持**预分配**以提升性能：

```java
@Entity
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_gen")
    @SequenceGenerator(
        name = "order_gen",           // 生成器名称，供 @GeneratedValue 引用
        sequenceName = "order_seq",   // 数据库中序列的名称
        initialValue = 1000,          // 序列起始值（默认 1）
        allocationSize = 50           // 预分配大小（默认 50）
    )
    private Long id;
}
```

**`allocationSize` 工作机制**：
- `allocationSize = 50` 表示 Hibernate 一次从序列申请 50 个 ID（如 1~50），缓存在应用内存中。
- 这 50 个 ID 用完后再发起一次序列 `NEXTVAL` 调用，大幅减少数据库往返。
- 若应用重启，预分配但未使用的 ID 将**丢失**（导致主键不连续，但无业务影响）。

**序列 vs 自增对比**：

| 维度 | SEQUENCE | IDENTITY |
|------|----------|----------|
| JDBC 批处理 | 支持 | 不支持 |
| 预分配 | 支持（allocationSize） | 不支持 |
| ID 可预知 | 支持（提前获取 ID） | 不支持（插入后才知道） |
| 数据库支持 | PostgreSQL, Oracle, DB2, H2 | MySQL, SQL Server, DB2, PostgreSQL |
| 移植性 | 需数据库支持序列 | 几乎所有数据库都支持 |

### 3.2.4 GenerationType.TABLE

使用一张特殊的数据表来模拟序列，通常**不推荐使用**：

```java
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "ticket_gen")
@TableGenerator(
    name = "ticket_gen",
    table = "t_sequence",            // 序列表名
    pkColumnName = "seq_name",       // 序列表中的序列名列
    valueColumnName = "seq_value",   // 序列表中的当前值列
    pkColumnValue = "ticket_pk",     // 当前序列的名称（值）
    initialValue = 1,                // 起始值
    allocationSize = 50              // 预分配大小
)
private Long id;
```

生成的序列表结构：

```sql
CREATE TABLE t_sequence (
    seq_name  VARCHAR(255) NOT NULL,
    seq_value BIGINT,
    PRIMARY KEY (seq_name)
);
```

**性能问题**：
- 每次分配 ID 都需要对 `t_sequence` 表进行 `SELECT FOR UPDATE` 行级锁，在高并发下成为瓶颈。
- 相比数据库原生 SEQUENCE/IDENTITY，性能差 10 倍以上。
- **只有在不支持序列也不支持自增的古老数据库上才考虑使用。**

### 3.2.5 Hibernate 扩展生成器

**UUID 生成策略**：

```java
@Entity
public class Document {
    
    @Id
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    private UUID id;  // Hibernate 6 之前使用 UUID
    
    // Hibernate 6 推荐方式
    // @Id
    // @UuidGenerator
    // private UUID id;
}
```

**assigned（手动赋值）**：

```java
@Id
private String id;  // 不由 Hibernate 生成，应用程序自行赋值
```

**自定义 IdentifierGenerator**：

```java
// Hibernate 6 推荐方式：实现 IdentifierGenerator 接口
public class CustomIdGenerator implements IdentifierGenerator {
    
    @Override
    public Object generate(SharedSessionContractImplementor session, Object entity) 
            throws HibernateException {
        return UUID.randomUUID().toString().replace("-", "");
    }
}

// 使用自定义生成器
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "custom_gen")
@GenericGenerator(
    name = "custom_gen",
    strategy = "com.example.CustomIdGenerator"
)
private String id;
```

**雪花算法 ID 生成器**（常用）：

```java
public class SnowflakeIdGenerator implements IdentifierGenerator {
    
    private final Snowflake snowflake = IdUtil.getSnowflake(1, 1);
    
    @Override
    public Serializable generate(SharedSessionContractImplementor session, Object object) 
            throws HibernateException {
        return snowflake.nextId();
    }
}
```

# 4. 列映射

## 4.1 @Column 详解

```java
@Entity
@Table(name = "t_product")
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(
        name = "product_name",               // 列名（默认驼峰转下划线）
        nullable = false,                    // 非空约束（DDL 生成用）
        unique = false,                      // 唯一约束
        length = 100,                        // VARCHAR 长度（默认 255）
        columnDefinition = "VARCHAR(100) COMMENT '产品名称'"  // DDL 片段
    )
    private String name;
    
    @Column(
        precision = 10,                      // 数值总位数
        scale = 2,                           // 小数位数
        columnDefinition = "DECIMAL(10,2) DEFAULT 0.00"
    )
    private BigDecimal price;
    
    @Column(
        insertable = true,                   // 是否参与 INSERT（默认 true）
        updatable = false                    // 是否参与 UPDATE（默认 true）
    )
    private LocalDateTime createTime;        // 创建后不可修改
    
    @Column(table = "t_product_detail")      // 列属于从表（配合 @SecondaryTable）
    private String description;
}
```

## 4.2 @Basic

`@Basic` 是属性到列的默认映射方式，通常无需显式声明，但可用于配置懒加载和 optional：

```java
@Entity
public class Article {
    
    @Id
    private Long id;
    
    @Basic(fetch = FetchType.LAZY)           // 懒加载（需要字节码增强）
    @Column(columnDefinition = "TEXT")
    private String content;                  // 大文本字段，仅需要时才加载
    
    @Basic(optional = false)                 // 不可为 null
    private String title;
}
```

> **注意**：`@Basic(fetch = FetchType.LAZY)` 仅是**提示**，JPA 提供者可以选择忽略。Hibernate 实现此特性需要字节码增强（build-time weaving）才能生效。

## 4.3 @Transient

标记某个字段不是持久化属性，不会映射到数据库列：

```java
@Entity
public class User {
    
    @Id
    private Long id;
    
    private String firstName;
    private String lastName;
    
    @Transient                              // JPA 注解，不持久化
    private String fullName;                // 计算字段
    
    public String getFullName() {
        return firstName + " " + lastName;
    }
    
    // 也可以用 Java 的 transient 关键字
    // 但 transient 还会影响 Java 序列化，而 @Transient 只影响 JPA
}
```

**`@Transient` vs `transient`**：
- `@Transient`：仅标记为 JPA 非持久化字段，**不影响 Java 序列化**。
- `transient`：既标记为 JPA 非持久化（JPA 会忽略 transient 修饰的具体字段），也**影响 Java 序列化**。
- 同时使用时，序列化和持久化均被排除。

## 4.4 @Formula

映射一个只读的计算字段，其值由 SQL 表达式计算得出，无需存储在数据库列中：

```java
@Entity
@Table(name = "t_order")
public class Order {
    
    @Id
    private Long id;
    
    private BigDecimal amount;
    
    private BigDecimal discount;
    
    @Formula("amount * (1 - discount)")
    private BigDecimal finalAmount;          // 计算字段：最终金额
    
    @Formula("(SELECT COUNT(*) FROM t_order_item oi WHERE oi.order_id = id)")
    private int itemCount;                   // 子查询：订单项数量
    
    @Formula("(SELECT COALESCE(SUM(oi.quantity * oi.price), 0) FROM t_order_item oi WHERE oi.order_id = id)")
    private BigDecimal totalPrice;           // 子查询：订单总价（从明细计算）
}
```

**注意事项**：
- `@Formula` 字段是**只读**的，不会参与 INSERT/UPDATE 操作。
- 值在每次 `SELECT` 查询时**实时计算**，不建议用于频繁查询的热数据。
- 子查询中**不能引用当前实体的别名**，只能使用原始列名。
- 不参与脏检查，不会因关联表数据变化而自动更新。

## 4.5 @ColumnTransformer

在 SQL 层面对列的读写值进行转换，常用于透明加解密：

```java
@Entity
@Table(name = "t_employee")
public class Employee {
    
    @Id
    private Long id;
    
    private String name;
    
    @ColumnTransformer(
        read = "AES_DECRYPT(UNHEX(salary_encrypted), 'secret_key')",
        write = "HEX(AES_ENCRYPT(?, 'secret_key'))"
    )
    @Column(name = "salary_encrypted", columnDefinition = "VARBINARY(256)")
    private BigDecimal salary;               // 数据库存密文，Java 中透明存取明文
    
    @ColumnTransformer(
        read = "UPPER(email)",              // 读取时转大写
        write = "LOWER(?)"                  // 写入时转小写
    )
    @Column(name = "email")
    private String email;
}
```

`@ColumnTransformer` 中，`read` 的 SQL 表达式中使用原始列名作为占位符，`write` 的 SQL 表达式中使用 `?` 作为参数占位符。

# 5. 基本类型映射

## 5.1 Java 基本类型与包装类型

| Java 类型 | 默认 JDBC 类型 | 说明 |
|----------|--------------|------|
| `byte` / `Byte` | TINYINT | 0~255 |
| `short` / `Short` | SMALLINT | -32768~32767 |
| `int` / `Integer` | INTEGER | 最常用的整数类型 |
| `long` / `Long` | BIGINT | 主键常用 |
| `float` / `Float` | FLOAT | 单精度浮点 |
| `double` / `Double` | DOUBLE | 双精度浮点 |
| `boolean` / `Boolean` | BIT / BOOLEAN | 布尔值 |

**推荐使用包装类型**：包装类型支持 `null` 值，能明确表示 "未知" 状态。基本类型有默认值（int 默认 0），可能导致误判某个字段是否被赋值。

```java
@Entity
public class User {
    @Id
    private Long id;
    
    private Integer age;     // null 表示未填写，0 表示 0 岁
    // 若用 int age，未填写时默认 0，无法与真实 0 岁区分
}
```

## 5.2 字符串类型

```java
@Entity
public class Post {
    
    @Id
    private Long id;
    
    @Column(length = 128)               // VARCHAR(128)，默认 255
    private String title;
    
    @Lob                                // TEXT / CLOB
    private String content;
    
    @Nationalized                       // NVARCHAR（Unicode）
    @Column(length = 50)
    private String displayName;
}
```

## 5.3 数值类型

```java
@Entity
public class FinancialRecord {
    
    @Id
    private Long id;
    
    private BigInteger bigValue;        // NUMERIC（任意精度整数）
    
    @Column(precision = 18, scale = 4)  // DECIMAL(18,4)
    private BigDecimal taxRate;         // 税率，4 位小数
    
    // BigDecimal 用于金额（避免浮点精度问题）
    @Column(precision = 10, scale = 2)
    private BigDecimal amount;          // DECIMAL(10,2)
}
```

**金额必须使用 `BigDecimal`**，`float`/`double` 存在二进制浮点精度问题（如 0.1 无法精确表示），金额计算会累积误差。

## 5.4 日期时间类型

### 5.4.1 旧 API（不推荐）

```java
@Entity
public class LegacyEvent {
    
    @Temporal(TemporalType.DATE)
    private java.util.Date startDate;       // 仅日期：2024-01-15
    
    @Temporal(TemporalType.TIME)
    private java.util.Date startTime;       // 仅时间：14:30:00
    
    @Temporal(TemporalType.TIMESTAMP)
    private java.util.Date createdAt;       // 日期+时间：2024-01-15 14:30:00.123
    
    private java.sql.Timestamp updatedAt;   // 直接映射 TIMESTAMP
}
```

**旧 API 的问题**：
- `java.util.Date` 将日期、时间混合在一起，语义不清晰。
- 时区处理混乱（看似有值，实际上 JDBC 驱动会按连接时区转换）。
- `java.sql.Date` 继承自 `java.util.Date`，操作时容易混淆。
- `java.util.Date` 是可变的，多线程不安全。

### 5.4.2 Java 8+ java.time API

自 Hibernate 5.3+ 起，`java.time` 包下的类型已被**原生支持**（无需额外依赖）：

```java
@Entity
public class Event {
    
    @Id
    private Long id;
    
    private LocalDate startDate;           // DATE：          2024-01-15
    private LocalTime startTime;           // TIME：          14:30:00
    private LocalDateTime createdAt;       // TIMESTAMP：     2024-01-15 14:30:00.123456
    
    private OffsetTime startOffsetTime;    // TIME_WITH_TIMEZONE：  14:30:00+08:00
    private OffsetDateTime updatedAt;      // TIMESTAMP_WITH_TIMEZONE：2024-01-15T14:30:00+08:00
    
    // 注意：ZonedDateTime 行为因时区规则变化而不同，推荐统一使用 UTC
    private Instant publishedAt;           // TIMESTAMP_UTC： 2024-01-15T06:30:00Z
    
    private Year fiscalYear;               // INTEGER：       2024
    private YearMonth billingPeriod;       // DATE：          2024-06
    private Duration elapsed;              // BIGINT（纳秒）： PT30M → 1800000000000
    private Period validityPeriod;         // VARCHAR（ISO-8601 字符串）： P1Y2M3D
}
```

**时区配置**：

```properties
# 强制 JDBC 连接使用 UTC 时区
hibernate.jdbc.time_zone = UTC

# 或 Spring Boot
spring.jpa.properties.hibernate.jdbc.time_zone=UTC
```

## 5.5 枚举类型

### 5.5.1 @Enumerated(EnumType.ORDINAL)

按枚举**序数**（0, 1, 2, ...）存储为整数：

```java
public enum Status {
    PENDING,    // 0
    ACTIVE,     // 1
    INACTIVE,   // 2
    DELETED     // 3
}

@Entity
public class Account {
    @Id
    private Long id;
    
    @Enumerated(EnumType.ORDINAL)
    private Status status;     // 存整数：PENDING=0, ACTIVE=1...
}
```

> **严重风险**：若将来在 `PENDING` 和 `ACTIVE` 之间插入 `DRAFT`，则原数据库中的 1（ACTIVE）会变成 DRAFT，导致数据错乱。**强烈不推荐 ORDINAL**。

### 5.5.2 @Enumerated(EnumType.STRING)

按枚举**名称**存储为字符串：

```java
@Enumerated(EnumType.STRING)
@Column(length = 20)
private Status status;         // 存字符串："PENDING", "ACTIVE"...

// 推荐始终使用 STRING，安全可读
```

### 5.5.3 Hibernate 扩展与 AttributeConverter

使用 `AttributeConverter` 实现自定义枚举映射（如按 code 值映射）：

```java
public enum Gender {
    MALE(1, "男"),
    FEMALE(2, "女"),
    UNKNOWN(0, "未知");
    
    private final int code;
    private final String description;
    
    Gender(int code, String description) {
        this.code = code;
        this.description = description;
    }
    
    public int getCode() { return code; }
    
    public static Gender fromCode(int code) {
        for (Gender g : Gender.values()) {
            if (g.code == code) return g;
        }
        throw new IllegalArgumentException("Unknown code: " + code);
    }
}

@Converter(autoApply = false)
public class GenderConverter implements AttributeConverter<Gender, Integer> {
    
    @Override
    public Integer convertToDatabaseColumn(Gender gender) {
        return gender == null ? null : gender.getCode();
    }
    
    @Override
    public Gender convertToEntityAttribute(Integer code) {
        return code == null ? null : Gender.fromCode(code);
    }
}

// 使用
@Entity
public class User {
    @Id
    private Long id;
    
    @Convert(converter = GenderConverter.class)
    private Gender gender;     // 数据库存 0/1/2，Java 对象是枚举
}
```

## 5.6 大对象类型

### 5.6.1 @Lob

```java
@Entity
public class Document {
    
    @Id
    private Long id;
    
    @Lob
    @Basic(fetch = FetchType.LAZY)           // 懒加载大对象（需字节码增强）
    @Column(columnDefinition = "MEDIUMTEXT")
    private String content;                  // CLOB：文本数据
    
    @Lob
    @Basic(fetch = FetchType.LAZY)
    private byte[] attachment;               // BLOB：二进制数据
    
    @Lob
    private Serializable metadata;           // 序列化 BLOB：Java 对象
}
```

**JDBC 流式读取**：Hibernate 对 `@Lob` 字段默认使用基于 JDBC `Clob`/`Blob` 的流式读取，只有在第一次调用 getter 时才从数据库加载数据。

### 5.6.2 @Nationalized

用于国际化字符数据（多语言支持）：

```java
@Entity
public class I18nMessage {
    
    @Id
    private Long id;
    
    @Nationalized
    @Column(length = 500)
    private String message;                  // NVARCHAR(500)
    
    @Lob
    @Nationalized
    private String fullText;                 // NCLOB
}
```

对应的 DDL：
- `CHAR` → `NCHAR`
- `VARCHAR` → `NVARCHAR`（MySQL 中用 `utf8mb4` 字符集等效）
- `CLOB` → `NCLOB`

## 5.7 UUID

```java
@Entity
public class Resource {
    
    @Id
    @GeneratedValue
    private UUID id;                         // Hibernate 6 自动使用 UUID 生成器
    
    // 若需二进制存储（更高效）
    // @Column(columnDefinition = "BINARY(16)")
    // private UUID id;
}
```

| 存储方式 | 数据库类型 | 长度 | 性能 |
|---------|-----------|------|------|
| 字符串 | `VARCHAR(36)` | 36 bytes | 较差（字符串比较） |
| 二进制 | `BINARY(16)` | 16 bytes | 较好（原始字节比较） |
| PostgreSQL 原生 UUID | `UUID` | 16 bytes | 最好（数据库原生优化） |

```java
// PostgreSQL 原生 UUID 支持
@Entity
@Table(name = "t_resource")
public class Resource {
    @Id
    @GeneratedValue
    @Column(columnDefinition = "UUID")       // PostgreSQL 使用 uuid 类型
    private UUID id;
}
```

## 5.8 JSON / XML 类型

### 5.8.1 Hibernate 6 JSON 映射

Hibernate 6 对 JSON 类型提供了官方支持，可配合 `@JdbcTypeCode` 或专门的 `@JsonFormat` 等注解：

```java
@Entity
@Table(name = "t_product")
public class Product {
    
    @Id
    private Long id;
    
    // Hibernate 6 + PostgreSQL jsonb
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> attributes;  // JSON 对象
}
```

### 5.8.2 第三方扩展（Vlad Mihalcea 的 hibernate-types）

```java
// 依赖：io.hypersistence:hypersistence-utils-hibernate-60
@Entity
@Table(name = "t_event")
@TypeDef(name = "jsonb", typeClass = JsonBinaryType.class)
public class Event {
    
    @Id
    private Long id;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, String> properties;
}
```

## 5.9 AttributeConverter（属性转换器）

JPA 2.1 标准特性，用于自定义 Java 属性与数据库列之间的转换逻辑：

```java
// 布尔值 → 'Y' / 'N' 字符转换器
@Converter(autoApply = true)                // autoApply=true：所有 Boolean 字段自动应用
public class BooleanToYNConverter implements AttributeConverter<Boolean, String> {
    
    @Override
    public String convertToDatabaseColumn(Boolean attribute) {
        return (attribute != null && attribute) ? "Y" : "N";
    }
    
    @Override
    public Boolean convertToEntityAttribute(String dbData) {
        return "Y".equals(dbData);
    }
}

// 加密转换器
@Converter
public class CryptoConverter implements AttributeConverter<String, String> {
    
    private static final String SECRET_KEY = "AES_KEY_32_BYTES_LONG_STRING!!";
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        // 加密逻辑（使用 AES）
        return CryptoUtils.encrypt(attribute, SECRET_KEY);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        return CryptoUtils.decrypt(dbData, SECRET_KEY);
    }
}

// 手动指定使用哪个转换器
@Entity
public class User {
    @Id
    private Long id;
    
    @Convert(converter = CryptoConverter.class)
    private String phone;                    // 数据库中加密存储
}
```

**`@Converter(autoApply = true)` vs 手动 `@Convert`**：

| 方式 | 作用范围 | 示例 |
|------|---------|------|
| `autoApply = true` | 全局所有该类型的字段 | 所有 `Boolean` → `CHAR(1)` |
| `@Convert(converter = ...)` | 仅指定字段 | 仅 `phone` 字段加密 |

**`AttributeConverter` vs `@Type`**：
- `AttributeConverter` 是 **JPA 标准**（JPA 2.1），可移植性好。
- `@Type` 是 **Hibernate 扩展**，功能更强大但绑定 Hibernate。
- 推荐优先使用 `AttributeConverter`，当处理复杂类型（如 JSON、UUID）时使用 Hibernate 扩展。

# 6. 组件映射（Embedded）

## 6.1 @Embeddable / @Embedded

组件映射用于将一组相关属性封装为值对象（Value Object），这些属性映射到**同一张表**：

```java
@Embeddable
public class Address {
    
    @Column(name = "province", length = 20)
    private String province;
    
    @Column(name = "city", length = 50)
    private String city;
    
    @Column(name = "district", length = 50)
    private String district;
    
    @Column(name = "detail", length = 200)
    private String detail;
    
    @Column(name = "zip_code", length = 10)
    private String zipCode;
    
    // getter / setter / equals / hashCode 省略
}

@Entity
@Table(name = "t_user")
public class User {
    
    @Id
    private Long id;
    
    private String username;
    
    @Embedded
    private Address homeAddress;            // 家庭地址（映射到同表）
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "province", column = @Column(name = "work_province")),
        @AttributeOverride(name = "city",     column = @Column(name = "work_city")),
        @AttributeOverride(name = "district", column = @Column(name = "work_district")),
        @AttributeOverride(name = "detail",   column = @Column(name = "work_detail")),
        @AttributeOverride(name = "zipCode",  column = @Column(name = "work_zip_code"))
    })
    private Address workAddress;             // 工作地址（列名覆盖）
}
```

最终生成的 `t_user` 表包含列：`id`, `username`, `province`, `city`, `district`, `detail`, `zip_code`, `work_province`, `work_city`, `work_district`, `work_detail`, `work_zip_code`。

**嵌套组件**：

```java
@Embeddable
public class Amount {
    
    @Column(name = "value", precision = 10, scale = 2)
    private BigDecimal value;
    
    @Embedded
    private Currency currency;              // 嵌套组件
    
    @Embeddable
    public static class Currency {
        @Column(name = "currency_code", length = 3)
        private String code;
        
        @Column(name = "currency_symbol", length = 5)
        private String symbol;
    }
}

@Entity
public class Invoice {
    @Id
    private Long id;
    
    @Embedded
    private Amount totalAmount;            
    // 生成列：value, currency_code, currency_symbol
}
```

## 6.2 组件集合

`@ElementCollection` 用于映射一组值类型到单独的集合表：

```java
@Entity
@Table(name = "t_user")
public class User {
    
    @Id
    private Long id;
    
    // 基本类型集合
    @ElementCollection
    @CollectionTable(
        name = "t_user_nicknames",           // 集合表名
        joinColumns = @JoinColumn(name = "user_id")  // 关联外键
    )
    @Column(name = "nickname", length = 50)
    private Set<String> nicknames = new HashSet<>();
    
    // 组件类型集合
    @ElementCollection
    @CollectionTable(
        name = "t_user_address",
        joinColumns = @JoinColumn(name = "user_id")
    )
    private Set<Address> addresses = new HashSet<>();
    
    // 带顺序的集合
    @ElementCollection
    @CollectionTable(
        name = "t_user_tags",
        joinColumns = @JoinColumn(name = "user_id")
    )
    @OrderColumn(name = "sort_order")        // 持久化元素的索引位置
    @Column(name = "tag", length = 30)
    private List<String> tags = new ArrayList<>();
}
```

**`@ElementCollection` vs `@OneToMany`**：

| 维度 | @ElementCollection | @OneToMany |
|------|-------------------|------------|
| 元素类型 | 基本类型、@Embeddable | 实体（@Entity） |
| 生命周期 | 随拥有者自动删除（级联） | 需显式管理 |
| 独立标识 | 无（值类型） | 有（`@Id`） |
| 独立查询 | 不支持 | 支持 |
| 适用场景 | 颜色列表、标签、地址簿 | 订单项、评论、复杂关联 |

# 7. 复合主键

## 7.1 @IdClass

实体类中包含多个 `@Id` 字段，对应的 ID 类单独定义：

```java
// ID 类：必须实现 Serializable 和 equals()/hashCode()
public class OrderItemId implements Serializable {
    
    private Long orderId;      // 必须与实体中的 @Id 字段名和类型完全一致
    private Long productId;
    
    public OrderItemId() {}
    
    public OrderItemId(Long orderId, Long productId) {
        this.orderId = orderId;
        this.productId = productId;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        OrderItemId that = (OrderItemId) o;
        return Objects.equals(orderId, that.orderId) 
            && Objects.equals(productId, that.productId);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(orderId, productId);
    }
}

@Entity
@Table(name = "t_order_item")
@IdClass(OrderItemId.class)
public class OrderItem {
    
    @Id
    private Long orderId;
    
    @Id
    private Long productId;
    
    private Integer quantity;
    private BigDecimal price;
}

// 使用 find()
OrderItemId itemId = new OrderItemId(100L, 200L);
OrderItem item = entityManager.find(OrderItem.class, itemId);
```

## 7.2 @EmbeddedId

复合主键封装为 `@Embeddable` 组件：

```java
@Embeddable
public class OrderItemPk implements Serializable {
    
    @Column(name = "order_id")
    private Long orderId;
    
    @Column(name = "product_id")
    private Long productId;
    
    public OrderItemPk() {}
    
    public OrderItemPk(Long orderId, Long productId) {
        this.orderId = orderId;
        this.productId = productId;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        OrderItemPk that = (OrderItemPk) o;
        return Objects.equals(orderId, that.orderId) 
            && Objects.equals(productId, that.productId);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(orderId, productId);
    }
}

@Entity
@Table(name = "t_order_item")
public class OrderItem {
    
    @EmbeddedId
    private OrderItemPk id;
    
    private Integer quantity;
    private BigDecimal price;
}

// 使用 find()
OrderItemPk pk = new OrderItemPk(100L, 200L);
OrderItem item = entityManager.find(OrderItem.class, pk);
```

**`@EmbeddedId` vs `@IdClass`**：

| 维度 | @IdClass | @EmbeddedId |
|------|----------|-------------|
| ID 字段访问 | 直接访字段（`order.getId()`） | 通过 ID 对象（`order.getId().getOrderId()`） |
| JPQL 查询 | `SELECT o.orderId FROM OrderItem o` | `SELECT o.id.orderId FROM OrderItem o` |
| 代码简洁度 | ID 类字段与实体字段冗余 | ID 封装为一个对象 |
| 推荐场景 | 复合主键中部分字段有独立映射需求 | 复合主键作为整体操作更自然 |

## 7.3 派生标识符（Derived Identifier）

`@MapsId` 用于共享主键或复合主键中某部分来自关联实体：

```java
// 场景1：共享主键（一对一）
@Entity
@Table(name = "t_person")
public class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
}

@Entity
@Table(name = "t_person_detail")
public class PersonDetail {
    
    @Id
    private Long id;                         // 主键即外键（与 Person 的 id 相同）
    
    @OneToOne
    @MapsId                                  // id 值从 person 派生
    @JoinColumn(name = "id")
    private Person person;
    
    private String address;
    private String phone;
}

// 场景2：复合主键中部分来自关联实体
@Entity
@Table(name = "t_payment")
@IdClass(PaymentId.class)
public class Payment {
    
    @Id
    private Long userId;
    
    @Id
    private Long paymentSeq;
    
    @ManyToOne
    @MapsId("userId")                        // userId 值从 user 的 id 派生
    @JoinColumn(name = "user_id")
    private User user;
    
    private BigDecimal amount;
}

// 场景3：@EmbeddedId + @MapsId
@Embeddable
public class PaymentId implements Serializable {
    private Long userId;
    private Long paymentSeq;
}

@Entity
public class Payment {
    
    @EmbeddedId
    private PaymentId id;
    
    @ManyToOne
    @MapsId("userId")
    @JoinColumn(name = "user_id")
    private User user;
}
```

# 8. 实体特性配置

## 8.1 @DynamicInsert / @DynamicUpdate

动态生成 INSERT/UPDATE SQL，只包含非 null 或已变化的列：

```java
@Entity
@DynamicInsert                           // INSERT 时只包含非 null 字段
@DynamicUpdate                           // UPDATE 时只包含变化的字段
@Table(name = "t_report")
public class Report {
    
    @Id
    private Long id;
    
    private String title;
    
    @Column(columnDefinition = "TEXT")
    private String content;
    
    private Long viewCount;
    
    @Column(columnDefinition = "TEXT")
    private String exportPath;
    
    // 40+ 个业务字段
}
```

```sql
-- 默认行为（所有字段参与 UPDATE）
UPDATE t_report SET title=?, content=?, view_count=?, export_path=?, ... WHERE id=?

-- @DynamicUpdate（仅变化的字段）
UPDATE t_report SET title=? WHERE id=?
```

**权衡**：

| 维度 | 默认 | @DynamicInsert/@DynamicUpdate |
|------|------|----------------------------|
| SQL 体积 | 大（所有列） | 小（仅需要的列） |
| 预编译语句缓存 | 高效（同结构 SQL 可复用） | 低效（不同组合产生不同 SQL） |
| 数据库默认值 | 会被 null 覆盖 | 保留数据库默认值 |
| 适用场景 | 列少、频繁全量更新 | 宽表、大量可选字段、JSON 存储 |

## 8.2 @SelectBeforeUpdate

更新前先查询当前快照，与快照不一致时才执行 UPDATE：

```java
@Entity
@SelectBeforeUpdate
@OptimisticLocking(type = OptimisticLockType.DIRTY)
@Table(name = "t_config")
public class AppConfig {
    
    @Id
    private String configKey;
    
    @Lob
    private String configValue;
    
    // 配置表读取远多于写入，避免无意义的 UPDATE
}
```

**适用场景**：
- 实体属性很多但实际修改很少的场景。
- 需配合乐观锁使用，避免并发更新覆盖。
- 对写入性能要求不高的配置、元数据类表。

## 8.3 @Immutable

标记实体不可变，Hibernate 会跳过脏检查、不执行 UPDATE：

```java
@Entity
@Immutable
@Table(name = "t_audit_log")
public class AuditLog {
    
    @Id
    private Long id;
    
    @Column(length = 50)
    private String operation;
    
    private String operator;
    
    private LocalDateTime operateTime;
    
    @Column(columnDefinition = "JSON")
    private String changeDetails;
    
    // 审计日志只增不改
}

// 集合也可标记不可变
@Entity
public class Role {
    @Id private Long id;
    
    @ManyToMany
    @Immutable
    private Set<Permission> permissions;     // 权限集合不允许变更
}
```

## 8.4 @Proxy

控制 Hibernate 懒加载代理的行为：

```java
@Entity
@Proxy(lazy = false)                         // 禁用懒加载代理（总是立即加载）
public class Country {
    @Id
    private String code;
    private String name;
}

// Hibernate 6 中的 proxyClass（指定代理接口）
@Entity
@Proxy(proxyClass = Auditable.class)
public class Product implements Auditable {
    @Id
    private Long id;
    // ...
}
```

**Hibernate 6 行为变化**：
- Hibernate 6 允许在 proxy 初始化前将被代理引用赋值给不同类型变量（此前版本会抛出 `LazyInitializationException`）。
- 通过 `bytecode-enhancement` 避免部分代理场景。

## 8.5 @Polymorphism

控制实体在多态查询中的可见性：

```java
// EXPLICIT：只有明确引用该类时才会被查询（类似 JPA 的 non-polymorphic）
@Entity
@Polymorphism(type = PolymorphismType.EXPLICIT)
@Table(name = "t_archived_order")
public class ArchivedOrder {
    @Id
    private Long id;
    // ...
}

// 查询 Order 时不会包含 ArchivedOrder
List<Order> orders = session.createQuery("FROM Order", Order.class).list();

// IMPLICIT（默认）：多态查询覆盖子类
```

## 8.6 @OptimisticLocking

配置乐观锁检查的策略：

```java
@Entity
@OptimisticLocking(type = OptimisticLockType.ALL)  // 检查所有字段
public class Document {
    @Id
    private Long id;
    private String title;
    private String content;
}

@Entity
@OptimisticLocking(type = OptimisticLockType.DIRTY) // 仅检查变化的字段
public class Article {
    @Id
    private Long id;
    private String title;
    private String content;
}

@Entity
@OptimisticLocking(type = OptimisticLockType.VERSION) // 基于 @Version 字段（默认）
public class Product {
    @Id
    private Long id;
    
    @Version
    private Integer version;                 // 乐观锁版本号
    
    private String name;
    private BigDecimal price;
}

// 排除某个字段不参与乐观锁检查
@OptimisticLock(excluded = true)
@Column(columnDefinition = "TEXT")
private String remarks;
```

# 9. 属性验证

## 9.1 Bean Validation 集成

Hibernate Validator 与 Hibernate ORM 紧密集成，可在持久化操作时自动触发验证：

```java
@Entity
@Table(name = "t_user")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotNull(message = "用户名不能为空")
    @Size(min = 3, max = 50, message = "用户名长度须在 3~50 之间")
    @Column(length = 50, nullable = false)
    private String username;
    
    @Email(message = "邮箱格式不正确")
    @Column(length = 100)
    private String email;
    
    @Min(value = 0, message = "年龄不能为负数")
    @Max(value = 150, message = "年龄不能超过 150")
    private Integer age;
    
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    @Column(length = 11)
    private String phone;
    
    @NotNull
    @DecimalMin(value = "0.00", inclusive = false)
    @DecimalMax(value = "99999999.99")
    @Column(precision = 10, scale = 2)
    private BigDecimal amount;
    
    @NotEmpty
    @Column(nullable = false)
    private String password;
}
```

**自动验证时机**：默认在以下生命周期事件前触发：
- `pre-persist`（插入前）
- `pre-update`（更新前）
- `pre-remove`（删除前）

**DDL 约束自动生成**：

```properties
# 根据 Bean Validation 注解自动在 DDL 生成 NOT NULL、长度等约束
hibernate.validator.apply_to_ddl = true
```

| Validation 注解 | 自动生成的 DDL 约束 |
|-----------------|-------------------|
| `@NotNull` | `NOT NULL` |
| `@Size(max = 100)` | `VARCHAR(100)` |
| `@Min(0)` / `@Max(150)` | `CHECK (age >= 0 AND age <= 150)` |
| `@NotEmpty` | `NOT NULL` |
| `@Email` / `@Pattern` | 不生成 DDL 约束（仅应用层校验） |

**验证分组（Grouping）**：

```java
// 定义验证分组
public interface CreateGroup {}
public interface UpdateGroup {}

@Entity
public class Article {
    
    @Id
    @Null(groups = CreateGroup.class, message = "新建时 ID 必须为空")
    @NotNull(groups = UpdateGroup.class, message = "更新时 ID 不能为空")
    private Long id;
    
    @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
    @Column(nullable = false, length = 200)
    private String title;
}

// 在 Service 层指定验证分组
@Component
public class ArticleService {
    
    @PersistenceContext
    private EntityManager em;
    
    private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
    
    public Article create(Article article) {
        // 手动指定分组验证
        Set<ConstraintViolation<Article>> violations = 
            validator.validate(article, CreateGroup.class);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
        em.persist(article);
        return article;
    }
}
```

# 10. 命名与数据库对象

## 10.1 视图映射

使用 `@Subselect` 将数据库视图（或任意 SQL 查询）映射为只读实体：

```java
@Entity
@Subselect(
    "SELECT u.id, u.username, " +
    "       (SELECT COUNT(*) FROM t_order o WHERE o.user_id = u.id) AS order_count, " +
    "       (SELECT COALESCE(SUM(o.amount), 0) FROM t_order o WHERE o.user_id = u.id) AS total_amount " +
    "FROM t_user u"
)
@Synchronize({"t_user", "t_order"})          // 告知 Hibernate 哪些表变化后需刷新此视图
@Immutable                                   // 视图不可更新
public class UserSummary {
    
    @Id
    private Long id;
    
    private String username;
    
    @Column(name = "order_count")
    private Long orderCount;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
}
```

**`@Synchronize` 的作用**：
- 当一个事务中先修改 `t_order` 再查询 `UserSummary` 时，Hibernate 需要知道是否要 flush。
- 指定 `@Synchronize({"t_user", "t_order"})` 后，Hibernate 在查询 `UserSummary` 前会自动 flush `t_user` 和 `t_order` 的变化。

**物化视图映射**：与普通视图相同，但建议定期手动刷新（REFRESH MATERIALIZED VIEW）。

```java
@Entity
@Subselect("SELECT * FROM mv_user_stats")     // PostgreSQL 物化视图
@Immutable
public class UserStats {
    @Id
    private Long id;
    private String metricName;
    private BigDecimal metricValue;
}
```

## 10.2 同义词映射

通过 `@Table` 的 `schema`/`catalog` 间接映射 Oracle 同义词：

```java
@Entity
@Table(name = "V_EMPLOYEE", schema = "HR")   // 可能是同义词或视图
public class EmployeeView {
    @Id
    private Long id;
    private String fullName;
}
```

## 10.3 自定义 CRUD SQL

使用 `@SQLInsert`、`@SQLUpdate`、`@SQLDelete`、`@SQLSelect` 覆盖默认的 CRUD SQL：

```java
@Entity
@Table(name = "t_product")
@SQLInsert(sql = "INSERT INTO t_product(id, name, price, created_at) VALUES(?, ?, ?, NOW())")
@SQLUpdate(sql = "UPDATE t_product SET name = ?, price = ?, updated_at = NOW() WHERE id = ?")
@SQLDelete(sql = "UPDATE t_product SET deleted = 1, deleted_at = NOW() WHERE id = ?")
// ↑ 逻辑删除：不真正 DELETE，只标记删除状态
// 注意：这里的 @SQLDelete 不真正 DELETE，但 Hibernate 认为实体已删除，后续认为状态为 REMOVED
// 如果需要实体仍存在，应用 @Where(clause = "deleted = 0") 过滤
public class Product {
    
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @Column(name = "deleted")
    private Boolean deleted = false;
    
    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;
}

// 存储过程调用
@Entity
@NamedStoredProcedureQuery(
    name = "User.activate",
    procedureName = "sp_activate_user",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN,  name = "p_user_id", type = Long.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "p_result",   type = Integer.class)
    }
)
public class User {
    @Id
    private Long id;
    private String username;
}

// 调用存储过程
@Service
public class UserService {
    
    @PersistenceContext
    private EntityManager em;
    
    public int activateUser(Long userId) {
        StoredProcedureQuery query = em.createNamedStoredProcedureQuery("User.activate");
        query.setParameter("p_user_id", userId);
        query.execute();
        return (Integer) query.getOutputParameterValue("p_result");
    }
}
```

---

> **参考资源**：
> - [Hibernate ORM 官方文档 - Domain Model Mapping](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#domain-model)
> - [Vlad Mihalcea - High-Performance Java Persistence](https://vladmihalcea.com/)
> - [JPA 2.2 规范](https://jakarta.ee/specifications/persistence/2.2/)
