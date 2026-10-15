# 1. 关联映射概述

## 1.1 对象关联 vs 数据库关联的本质差异

在面向对象编程与关系型数据库之间，存在根本性的"对象-关系阻抗不匹配"（Object-Relational Impedance Mismatch）。理解这两种范式的差异，是掌握关联映射的前提。

| 维度 | 对象模型 | 关系模型 |
|------|---------|---------|
| 关联方向 | 单向引用（指针/引用），天然有方向 | 外键关联，本质上是双向的（通过 JOIN 可任意方向查询） |
| 关联方式 | 内存中的引用（对象指针） | 外键约束（数值匹配） |
| 多重性 | 直接的多对多关联 | 多对多需通过中间表（连接表）实现 |
| 遍历 | 通过引用的 getter 方法遍历 | 通过 JOIN 查询遍历 |
| 状态同步 | 对象关系变更后需显式持久化 | 外键列值变更直接影响查询结果 |

```java
// 对象模型：单向引用
public class Order {
    private Customer customer;          // Order → Customer 单向引用
    // Customer 中没有 Order 的引用（单向）
}

// 数据库模型：外键双向可查
// SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;  -- Order → Customer
// SELECT * FROM customers c JOIN orders o ON c.id = o.customer_id;  -- Customer → Order
```

Hibernate 关联映射的核心任务，就是在这两种范式之间建立桥接——让开发者以面向对象的方式操作关联，而框架自动翻译为正确的 SQL。

## 1.2 关联的方向性

### 1.2.1 单向关联（Unidirectional）

只有一方持有对另一方的引用。数据库中外键在哪一方取决于映射配置。

```java
// 单向 @ManyToOne：多方持有外键，一方无感知
@Entity
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;  // OrderItem → Order 单向
}

@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // 没有 OrderItem 的集合引用，Order 不知道有哪些 OrderItem
}
```

### 1.2.2 双向关联（Bidirectional）

双方互相持有对方的引用。但在数据库中，外键仍然只存在于一方（或中间表）——这就是"拥有方"概念的来源。

```java
// 双向 @OneToMany / @ManyToOne
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;  // Order → OrderItem
}

@Entity
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;  // OrderItem → Order
}
```

## 1.3 关联的拥有方（Owning Side）与反向方（Inverse Side）

这是 Hibernate/JPA 关联映射中**最重要的概念**，理解它才能正确配置双向关联。

- **拥有方（Owning Side）**：持有外键列（`@JoinColumn`）的一方，也称为"关系所有者"。拥有方的状态变更会触发数据库外键列的更新。
- **反向方（Inverse Side）**：通过 `mappedBy` 声明的一方，不持有外键列。反向方的状态变更**不会直接**更新外键列——Hibernate 忽略反向方的变更，仅通过拥有方来决定外键值。

```java
// 拥有方：持有 @JoinColumn
@Entity
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id")     // 外键列在此
    private Order order;               // OrderItem 是拥有方
}

// 反向方：通过 mappedBy 指向拥有方的字段
@Entity
public class Order {
    @OneToMany(mappedBy = "order")     // "order" 是 OrderItem 中指向 Order 的字段名
    private List<OrderItem> items;     // Order 是反向方（非拥有方）
}
```

**核心规则**：

- 外键列由 `@JoinColumn` / `@JoinTable` 所在方定义（即拥有方）
- 只有拥有方的变更才会被同步到数据库
- `mappedBy` 只存在于 `@OneToOne`、`@OneToMany`、`@ManyToMany` 中，`@ManyToOne` 没有 `mappedBy`
- `@ManyToOne` 永远是拥有方

## 1.4 mappedBy 的核心作用与原理

`mappedBy` 告诉 Hibernate："这个关联已经被对方实体中的 `xxx` 字段映射过了，不要再生成额外的外键列或中间表。"

如果不加 `mappedBy`，Hibernate 会认为这是一个**两个独立的单向关联**：
- `Order.orderItems` → 需要中间表或外键列
- `OrderItem.order` → 需要外键列
- 结果：产生两个独立的关联映射，数据可能不一致

加上 `mappedBy` 后：
- `Order.orderItems` → 只是 `OrderItem.order` 的镜像，不产生额外的数据库结构
- `OrderItem.order` → 唯一的外键映射

```sql
-- 错误：未加 mappedBy，Hibernate 创建了两条外键路径
CREATE TABLE order_item (
    id BIGINT PRIMARY KEY,
    order_id BIGINT,              -- 来自 OrderItem.order 的 @JoinColumn
    order_entity_id BIGINT        -- 来自 Order.items 的自动外键（多余）
);

-- 正确：加了 mappedBy，只有一条外键路径
CREATE TABLE order_item (
    id BIGINT PRIMARY KEY,
    order_id BIGINT               -- 仅来自 OrderItem.order 的 @JoinColumn
);
```

`mappedBy` 的值是**拥有方实体中引用本实体的 Java 字段名**（不是数据库列名）。例如 `mappedBy = "order"` 指的是 `OrderItem` 中的 `private Order order;` 字段。

## 1.5 关联映射最佳实践原则

1. **优先使用双向关联**：除非明确只需要单向遍历，否则双向关联更灵活
2. **@ManyToOne 永远设为 LAZY**：JPA 规范默认 `@ManyToOne` 是 EAGER，会导致严重的 N+1 问题
3. **在 @OneToMany 中始终设置 orphanRemoval = true**：避免孤立的子实体残留
4. **在 @ManyToMany 中级联仅设 PERSIST/MERGE**：不要使用 REMOVE 或 ALL，避免误删关联数据
5. **提供关联辅助方法**：在父实体内提供 `addXxx()` / `removeXxx()` 方法，确保双向关联的同步
6. **避免在 equals/hashCode 中包含懒加载关联**：可能导致意外的数据库查询
7. **避免在 toString() 中包含双向关联**：造成循环引用和 StackOverflow
8. **JSON 序列化时注意循环引用**：使用 `@JsonIgnore`、`@JsonManagedReference` / `@JsonBackReference` 或 `@JsonIdentityInfo`
9. **外键列放在子实体（多方）表中**：这是数据库设计的最佳实践，也是 Hibernate 推荐的默认行为
10. **连接表优先用于 @ManyToMany**：一对一和一对多使用外键列而非连接表

---

# 2. @OneToOne

## 2.1 单向 @OneToOne

### 2.1.1 外键关联（@JoinColumn）

外键关联是最常见的 @OneToOne 映射方式：在一方的表中创建一个外键列，指向另一方的主键。

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false, unique = true)
    private String username;

    @Column(name = "email")
    private String email;

    public User() {}
    public User(String username, String email) {
        this.username = username;
        this.email = email;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}

@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name")
    private String fullName;

    @Column(name = "bio", length = 500)
    private String bio;

    @Column(name = "avatar_url")
    private String avatarUrl;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;

    public UserProfile() {}
    public UserProfile(String fullName, String bio) {
        this.fullName = fullName;
        this.bio = bio;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public String getBio() { return bio; }
    public void setBio(String bio) { this.bio = bio; }
    public String getAvatarUrl() { return avatarUrl; }
    public void setAvatarUrl(String avatarUrl) { this.avatarUrl = avatarUrl; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

```sql
-- 生成的 DDL
CREATE TABLE user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255)
);

CREATE TABLE user_profile (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(255),
    bio VARCHAR(500),
    avatar_url VARCHAR(255),
    user_id BIGINT UNIQUE,
    FOREIGN KEY (user_id) REFERENCES user(id)
);
```

### 2.1.2 主键关联（@PrimaryKeyJoinColumn）

两表共享同一主键值，子表的 PK 同时是 FK。

```java
@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @Column(name = "department")
    private String department;

    public Employee() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDepartment() { return department; }
    public void setDepartment(String department) { this.department = department; }
}

@Entity
public class EmployeeDetail {
    @Id
    private Long id;  // 不使用 @GeneratedValue，值来自 Employee

    @OneToOne
    @PrimaryKeyJoinColumn  // 使用主键作为外键
    private Employee employee;

    @Column(name = "phone_number")
    private String phoneNumber;

    @Column(name = "address", length = 500)
    private String address;

    public EmployeeDetail() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Employee getEmployee() { return employee; }
    public void setEmployee(Employee employee) { this.employee = employee; }
    public String getPhoneNumber() { return phoneNumber; }
    public void setPhoneNumber(String phoneNumber) { this.phoneNumber = phoneNumber; }
    public String getAddress() { return address; }
    public void setAddress(String address) { this.address = address; }
}
```

```sql
CREATE TABLE employee (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    department VARCHAR(255)
);

CREATE TABLE employee_detail (
    id BIGINT PRIMARY KEY,
    phone_number VARCHAR(255),
    address VARCHAR(500),
    FOREIGN KEY (id) REFERENCES employee(id)
);
```

### 2.1.3 关联表关联（@JoinTable）

使用中间表来维护一对一关系（不常见，通常用外键就够）。

```java
@Entity
public class WorkStation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "station_code")
    private String stationCode;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinTable(
        name = "employee_workstation",
        joinColumns = @JoinColumn(name = "workstation_id"),
        inverseJoinColumns = @JoinColumn(name = "employee_id", unique = true)
    )
    private Employee employee;

    public WorkStation() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getStationCode() { return stationCode; }
    public void setStationCode(String stationCode) { this.stationCode = stationCode; }
    public Employee getEmployee() { return employee; }
    public void setEmployee(Employee employee) { this.employee = employee; }
}
```

```sql
CREATE TABLE employee_workstation (
    workstation_id BIGINT,
    employee_id BIGINT UNIQUE,
    FOREIGN KEY (workstation_id) REFERENCES work_station(id),
    FOREIGN KEY (employee_id) REFERENCES employee(id)
);
```

## 2.2 双向 @OneToOne

双向 @OneToOne 需要指定 `mappedBy`，让其中一方成为非拥有方。

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false, unique = true)
    private String username;

    // 反向方：mappedBy 指向 UserProfile 中的 "user" 字段
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, optional = false)
    private UserProfile profile;

    public User() {}
    public User(String username) {
        this.username = username;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public UserProfile getProfile() { return profile; }
    public void setProfile(UserProfile profile) { this.profile = profile; }

    // ============ 双向同步辅助方法 ============

    /**
     * 添加关联，同步双向引用
     */
    public void addProfile(UserProfile profile) {
        if (profile == null) return;
        this.profile = profile;
        profile.setUser(this);
    }

    /**
     * 移除关联，同步双向引用
     */
    public void removeProfile() {
        if (this.profile == null) return;
        UserProfile oldProfile = this.profile;
        this.profile = null;
        oldProfile.setUser(null);
    }
}

@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name")
    private String fullName;

    @Column(name = "bio", length = 500)
    private String bio;

    // 拥有方：持有外键列
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;

    public UserProfile() {}
    public UserProfile(String fullName) {
        this.fullName = fullName;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public String getBio() { return bio; }
    public void setBio(String bio) { this.bio = bio; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

**双向同步问题详解**：

```java
// 错误做法：只设置一端
User user = new User("john");
UserProfile profile = new UserProfile("John Doe");
profile.setUser(user);
// user.profile 仍然是 null —— 对象模型不一致！
session.persist(user);
session.persist(profile);

// 正确做法一：使用辅助方法
User user = new User("john");
UserProfile profile = new UserProfile("John Doe");
user.addProfile(profile);  // 内部同时设置双方引用
session.persist(user);     // 如设置了 cascade = CascadeType.ALL，profile 也会被级联保存

// 正确做法二：手动同步两端
User user = new User("john");
UserProfile profile = new UserProfile("John Doe");
profile.setUser(user);
user.setProfile(profile);
session.persist(user);
session.persist(profile);
```

## 2.3 @OneToOne 懒加载问题

### 2.3.1 为什么 @OneToOne 懒加载常失效

JPA 规范规定 `@OneToOne` 的默认加载策略是 **EAGER**。而且即使显式设置 `fetch = FetchType.LAZY`，在许多情况下 Hibernate 仍然无法创建代理对象，导致懒加载无效。

核心原因：**代理对象的限制**。

```java
// 数据库中有 user_profile 表：
// | id | user_id | full_name |
// | 1  | 1       | John      |
// | 2  | NULL    | Orphan    |

// 查询 User
User user = session.get(User.class, 1L);
UserProfile profile = user.getProfile();
```

对于 `user.id = 1`，其 profile 存在。Hibernate 创建代理需要判断：`profile` 到底是 null 还是需要懒加载的对象？但代理必须是对象（不能是 null），所以 Hibernate **必须查询数据库**才能知道外键值——这就导致即使声明了 LAZY，实际上也触发了立即查询。

**`optional = false` 的作用**：

```java
@OneToOne(fetch = FetchType.LAZY, optional = false)  // optional=false 是关键
@JoinColumn(name = "user_id", unique = true)
private UserProfile profile;
```

当 `optional = false` 时，Hibernate 知道关联**必定存在**（外键不为 NULL），因此可以安全地创建一个代理对象而无需查库。这是让 @OneToOne 懒加载生效的最简单方式——但前提是业务上确实允许 NOT NULL 约束。

### 2.3.2 字节码增强方案

通过构建时或运行时字节码增强，Hibernate 可以创建"无状态"代理，不需要先查库。

```xml
<!-- pom.xml：使用 Hibernate Enhancement Plugin -->
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
            </configuration>
            <goals>
                <goal>enhance</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 2.3.3 @LazyToOne 注解

Hibernate 特有的注解，允许更细粒度的懒加载控制。

```java
@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    @LazyToOne(LazyToOneOption.NO_PROXY)
    private User user;
}
```

- `LazyToOneOption.PROXY`：默认行为，使用代理对象（对 null 关联无效）
- `LazyToOneOption.NO_PROXY`：使用字节码增强，即使关联为 null 也不需查库
- `LazyToOneOption.FALSE`：立即加载

注意：`NO_PROXY` 需要字节码增强支持。

### 2.3.4 使用 @ManyToOne 替代 @OneToOne

一个务实的解决方案：在外键方使用 `@ManyToOne`（默认 EAGER，但显式设 LAZY 有效），配合唯一约束实现一对一语义。

```java
@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)  // @ManyToOne 的 LAZY 是真正有效的
    @JoinColumn(name = "user_id", unique = true, nullable = false)
    private User user;
}
```

`@ManyToOne` 的代理机制不同：Hibernate 可以直接从外键值创建代理（`session.load()`），无需查询数据库，因此懒加载能真正生效。

## 2.4 @MapsId 共享主键

`@MapsId` 实现"派生标识符"——子实体的主键同时是外键，且值直接从父实体派生，不需要独立生成。

```java
@Entity
public class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @OneToOne(mappedBy = "person", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, optional = false)
    private PersonPassport passport;

    public Person() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public PersonPassport getPassport() { return passport; }
    public void setPassport(PersonPassport passport) { this.passport = passport; }
}

@Entity
public class PersonPassport {
    @Id
    private Long id;  // 与 Person 共享主键，不使用 @GeneratedValue

    @Column(name = "passport_number", unique = true)
    private String passportNumber;

    @Column(name = "issue_country")
    private String issueCountry;

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId  // 主键值从 Person 的 id 派生
    @JoinColumn(name = "id")  // FK 即 PK
    private Person person;

    public PersonPassport() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getPassportNumber() { return passportNumber; }
    public void setPassportNumber(String passportNumber) { this.passportNumber = passportNumber; }
    public String getIssueCountry() { return issueCountry; }
    public void setIssueCountry(String issueCountry) { this.issueCountry = issueCountry; }
    public Person getPerson() { return person; }
    public void setPerson(Person person) { this.person = person; }
}
```

```java
// 使用示例
Person person = new Person();
person.setName("Alice");

PersonPassport passport = new PersonPassport();
passport.setPassportNumber("E1234567");
passport.setIssueCountry("China");
passport.setPerson(person);  // MapsId 从此获取 id

person.setPassport(passport);

session.persist(person);  // Person 先 insert 获得 id=1
// PersonPassport 的 id 自动设为 1，执行 insert with id=1
```

```sql
CREATE TABLE person (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE person_passport (
    id BIGINT PRIMARY KEY,
    passport_number VARCHAR(255) UNIQUE,
    issue_country VARCHAR(255),
    FOREIGN KEY (id) REFERENCES person(id)
);
```

`@MapsId` 的优势：
- 消除冗余的主键列，表结构更清晰
- 保证两表的一对一对应关系天然约束
- 查询性能更好（不需要额外的 FK 索引）

---

# 3. @ManyToOne / @OneToMany

## 3.1 单向 @ManyToOne

这是**最常见的关联形式**，也是最简单、最符合数据库设计的映射方式。多方持有外键，指向一方。

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;

    @Column(name = "order_date")
    private LocalDateTime orderDate;

    public Order() {}
    public Order(String orderNumber) {
        this.orderNumber = orderNumber;
        this.orderDate = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }
    public LocalDateTime getOrderDate() { return orderDate; }
    public void setOrderDate(LocalDateTime orderDate) { this.orderDate = orderDate; }
}

@Entity
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name")
    private String productName;

    @Column(name = "quantity")
    private Integer quantity;

    @Column(name = "unit_price")
    private BigDecimal unitPrice;

    // 单向 @ManyToOne，OrderItem 持有外键
    @ManyToOne(fetch = FetchType.LAZY)  // 必须显式设 LAZY！
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    public OrderItem() {}
    public OrderItem(String productName, Integer quantity, BigDecimal unitPrice) {
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
}
```

```sql
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_number VARCHAR(255) NOT NULL UNIQUE,
    order_date TIMESTAMP
);

CREATE TABLE order_item (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(255),
    quantity INTEGER,
    unit_price DECIMAL(19, 2),
    order_id BIGINT NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

### 3.1.1 FetchType 对查询的影响

```java
// @ManyToOne 默认 EAGER → 执行 LEFT OUTER JOIN，一条 SQL 查出所有数据
// SELECT o.*, u.* FROM orders o LEFT JOIN users u ON o.user_id = u.id

// @ManyToOne(fetch = FetchType.LAZY) → 先只查 Order，需要时再查 User
// SELECT * FROM orders o WHERE o.id = ?
// SELECT * FROM users u WHERE u.id = ?  -- 仅在访问 order.getUser() 时执行
```

**为什么必须改 @ManyToOne 为 LAZY**：EAGER 会在每次查询主实体时都 JOIN 关联实体，导致：
- 不必要的数据传输（即使不需要关联数据也查出来）
- N+1 问题扩散
- 查询计划复杂化

## 3.2 单向 @OneToMany

### 3.2.1 默认行为：中间表

不加 `@JoinColumn` 时，Hibernate 默认使用中间表来维护关联。

```java
@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "dept_name")
    private String deptName;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();

    public Department() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getDeptName() { return deptName; }
    public void setDeptName(String deptName) { this.deptName = deptName; }
    public List<Employee> getEmployees() { return employees; }
    public void setEmployees(List<Employee> employees) { this.employees = employees; }
}
```

```sql
-- 生成了三张表！中间表 department_employees
CREATE TABLE department (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dept_name VARCHAR(255)
);

CREATE TABLE employee (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE department_employees (
    department_id BIGINT NOT NULL,
    employees_id BIGINT NOT NULL UNIQUE,
    FOREIGN KEY (department_id) REFERENCES department(id),
    FOREIGN KEY (employees_id) REFERENCES employee(id)
);
```

### 3.2.2 使用 @JoinColumn 避免中间表

```java
@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "dept_name")
    private String deptName;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")  // 外键在 Employee 表中
    private List<Employee> employees = new ArrayList<>();

    public Department() {}
    // getters/setters 省略
}
```

```sql
-- 只有两张表，外键在 Employee 表中
CREATE TABLE department (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dept_name VARCHAR(255)
);

CREATE TABLE employee (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    department_id BIGINT,
    FOREIGN KEY (department_id) REFERENCES department(id)
);
```

### 3.2.3 单向 @OneToMany 的更新问题

```java
// 单向 @OneToMany + @JoinColumn 的持久化行为
Department dept = new Department("Engineering");
Employee emp1 = new Employee("Alice");
Employee emp2 = new Employee("Bob");

dept.getEmployees().add(emp1);
dept.getEmployees().add(emp2);

session.persist(dept);   // INSERT department
session.persist(emp1);   // INSERT employee (department_id = NULL)
session.persist(emp2);   // INSERT employee (department_id = NULL)
session.flush();         // UPDATE employee SET department_id = ? WHERE id = ?
                         // UPDATE employee SET department_id = ? WHERE id = ?
```

因为 `Department` 是反向方（非拥有方），Hibernate 在 INSERT 时不知道 `department_id` 该设什么值，只能先插入 NULL，然后通过额外的 UPDATE 语句设置外键。

**这就是为什么推荐使用双向 @OneToMany + @ManyToOne**——让多方成为拥有方，INSERT 时直接写入正确的外键值。

### 3.2.4 推荐方案对比

```java
// ❌ 不推荐：单向 @OneToMany
@Entity
public class Department {
    @OneToMany
    @JoinColumn(name = "department_id")
    private List<Employee> employees;
}
// 问题：额外的 UPDATE 语句、中间表

// ✅ 推荐：双向 @OneToMany + @ManyToOne
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    private List<Employee> employees = new ArrayList<>();
}
@Entity
public class Employee {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
}
// 优势：一次 INSERT 完成，无额外 UPDATE，外键约束清晰
```

## 3.3 双向 @OneToMany / @ManyToOne（推荐）

完整的双向关联示例，以订单（Order）与订单项（OrderItem）为经典模型。

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;

    @Column(name = "order_date")
    private LocalDateTime orderDate;

    @Column(name = "total_amount")
    private BigDecimal totalAmount;

    // 反向方：mappedBy 指向 OrderItem 中的 "order" 字段
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public Order() {}
    public Order(String orderNumber) {
        this.orderNumber = orderNumber;
        this.orderDate = LocalDateTime.now();
        this.totalAmount = BigDecimal.ZERO;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }
    public LocalDateTime getOrderDate() { return orderDate; }
    public void setOrderDate(LocalDateTime orderDate) { this.orderDate = orderDate; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }

    // ============ 双向关联同步辅助方法 ============

    /**
     * 添加订单项，同步双向引用并重新计算总金额
     */
    public void addOrderItem(OrderItem item) {
        if (item == null) return;
        items.add(item);
        item.setOrder(this);
        recalculateTotal();
    }

    /**
     * 移除订单项，同步双向引用并重新计算总金额
     */
    public void removeOrderItem(OrderItem item) {
        if (item == null) return;
        items.remove(item);
        item.setOrder(null);
        recalculateTotal();
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(item -> item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

@Entity
@Table(name = "order_item")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name", nullable = false)
    private String productName;

    @Column(name = "quantity", nullable = false)
    private Integer quantity;

    @Column(name = "unit_price", nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;

    // 拥有方：持有外键
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    public OrderItem() {}
    public OrderItem(String productName, Integer quantity, BigDecimal unitPrice) {
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
}
```

```java
// 使用示例：在拥有方（OrderItem）设置关联再保存
Order order = new Order("ORD-2024-001");

OrderItem item1 = new OrderItem("Keyboard", 2, new BigDecimal("99.99"));
OrderItem item2 = new OrderItem("Mouse", 1, new BigDecimal("49.99"));

// 方式一：使用辅助方法
order.addOrderItem(item1);  // 内部同步双向引用
order.addOrderItem(item2);

// 方式二：手动在拥有方设置
item1.setOrder(order);
item2.setOrder(order);
order.getItems().add(item1);
order.getItems().add(item2);

// 保存（cascade = CascadeType.ALL 会级联保存 OrderItem）
session.persist(order);
```

```sql
-- 生成的 SQL
INSERT INTO orders (order_number, order_date, total_amount) VALUES (?, ?, ?);

INSERT INTO order_item (product_name, quantity, unit_price, order_id) VALUES (?, ?, ?, ?);
INSERT INTO order_item (product_name, quantity, unit_price, order_id) VALUES (?, ?, ?, ?);

UPDATE orders SET total_amount = ? WHERE id = ?;
```

### 3.3.1 循环引用风险

```java
// ❌ 危险：toString() 循环引用 → StackOverflowError
@Override
public String toString() {
    return "Order{" +
        "id=" + id +
        ", items=" + items +   // 打印 items → 每个 item 又打印 order → 死循环
        '}';
}

// ✅ 安全：排除双向关联
@Override
public String toString() {
    return "Order{" +
        "id=" + id +
        ", orderNumber='" + orderNumber + '\'' +
        ", totalAmount=" + totalAmount +
        '}';
}

// JSON 序列化循环引用解决方案
// 方案一：@JsonIgnore
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    @JsonIgnore  // 序列化 OrderItem 时不包含 Order
    private Order order;
}

// 方案二：@JsonManagedReference / @JsonBackReference
public class Order {
    @OneToMany(mappedBy = "order")
    @JsonManagedReference  // 序列化 Order 时包含 OrderItem
    private List<OrderItem> items;
}
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id")
    @JsonBackReference  // 序列化 OrderItem 时不包含 Order
    private Order order;
}

// 方案三：@JsonIdentityInfo（基于对象标识避免循环）
@Entity
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class Order { ... }

@Entity
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class OrderItem { ... }
```

## 3.4 级联操作

### 3.4.1 CascadeType 枚举详解

```java
public enum CascadeType {
    PERSIST,  // 级联 persist()
    MERGE,    // 级联合并 merge()
    REMOVE,   // 级联删除 remove()
    REFRESH,  // 级联刷新 refresh()（从数据库重新加载）
    DETACH,   // 级联分离 detach()（将实体从持久化上下文分离）
    ALL       // 以上全部
}
```

```java
@Entity
public class Blog {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "title")
    private String title;

    @OneToMany(mappedBy = "blog", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    private List<Comment> comments = new ArrayList<>();

    // CascadeType.PERSIST：persist(blog) → 自动 persist 所有 Comment
    // CascadeType.MERGE：merge(blog) → 自动 merge 所有 Comment
    // 未设置 REMOVE：remove(blog) → 不会自动 remove Comment（需手动或依赖 orphanRemoval）

    public Blog() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public List<Comment> getComments() { return comments; }
    public void setComments(List<Comment> comments) { this.comments = comments; }
}

@Entity
public class Comment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "content")
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "blog_id")
    private Blog blog;

    public Comment() {}
    public Comment(String content) { this.content = content; }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public Blog getBlog() { return blog; }
    public void setBlog(Blog blog) { this.blog = blog; }
}
```

### 3.4.2 级联操作的实际行为

```java
// ===== PERSIST =====
Blog blog = new Blog("My Blog");
Comment c1 = new Comment("Great!");
Comment c2 = new Comment("Nice!");

blog.getComments().add(c1);
blog.getComments().add(c2);
c1.setBlog(blog);
c2.setBlog(blog);

session.persist(blog);  // 级联 persist c1 和 c2
// SQL:
// INSERT INTO blog (title) VALUES (?)
// INSERT INTO comment (content, blog_id) VALUES (?, ?)
// INSERT INTO comment (content, blog_id) VALUES (?, ?)

// ===== REMOVE =====
session.remove(blog);  // 级联 remove 所有 Comment
// SQL (如果设置 cascade.REMOVE):
// DELETE FROM comment WHERE id = ?
// DELETE FROM comment WHERE id = ?
// DELETE FROM blog WHERE id = ?

// ===== MERGE =====
// 传入一个 detached 的 blog（有新的 comments）
Blog detachedBlog = ...;  // 来自上一 session
session.merge(detachedBlog);  // 级联 merge 所有 Comment
```

### 3.4.3 orphanRemoval = true vs CascadeType.REMOVE

这是两种不同的删除机制，经常被混淆。

| 维度 | orphanRemoval = true | CascadeType.REMOVE |
|------|---------------------|-------------------|
| 触发条件 | 子实体从父集合中**移除** | 父实体被**删除** |
| 行为 | 从集合移除 → 子实体被 DELETE | 删除父实体 → 子实体被 DELETE |
| JPA 规范 | JPA 2.0 新增 | JPA 1.0 已有 |
| 适用场景 | 清除子实体（不再需要） | 删除父实体（整棵树删除） |
| 数据库操作 | DELETE FROM child WHERE id = ? | DELETE FROM child WHERE id = ? |

```java
// orphanRemoval 示例：从集合中移除 → 数据库 DELETE
Blog blog = session.get(Blog.class, 1L);
Comment comment = blog.getComments().get(0);
blog.getComments().remove(comment);  // 从集合中移除
// session.flush() 后：
// DELETE FROM comment WHERE id = ?  -- orphanRemoval 触发

// CascadeType.REMOVE 示例：删除父实体 → 级联删除
Blog blog = session.get(Blog.class, 1L);
session.remove(blog);  // 删除父实体
// DELETE FROM comment WHERE id = ?  -- CascadeType.REMOVE 触发
// DELETE FROM comment WHERE id = ?
// DELETE FROM blog WHERE id = ?
```

**最佳实践**：在 `@OneToMany` 中通常都设置 `orphanRemoval = true`，因为：

```java
@OneToMany(mappedBy = "blog", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Comment> comments = new ArrayList<>();
```

这样既支持从集合中移除子实体时自动删除，也支持删除父实体时级联删除子实体——两全其美。

## 3.5 集合类型选择

### 3.5.1 List + @OrderColumn

`@OrderColumn` 在数据库表中维护一个额外的索引列，保证元素顺序与 Java List 完全一致。

```java
@Entity
public class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "article", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderColumn(name = "display_order")  // 在 Section 表中额外添加 display_order 列
    private List<Section> sections = new ArrayList<>();

    // 操作 List 时 Hibernate 会维护 display_order 的值
    // 删除中间元素 → 重新编号后续元素（可能产生大量 UPDATE）
}
```

```sql
CREATE TABLE section (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    article_id BIGINT,
    display_order INTEGER,  -- @OrderColumn 产生的索引列
    FOREIGN KEY (article_id) REFERENCES article(id)
);
```

**性能注意**：在 List 中间删除/插入元素时，Hibernate 会生成大量 UPDATE 语句重新编号后续元素。

### 3.5.2 List + @OrderBy

`@OrderBy` 在 SQL 查询时添加 ORDER BY 子句，不维护额外的列。顺序是 "读取时排序"，不是持久化顺序。

```java
@Entity
public class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "article")
    @OrderBy("createdAt ASC, title DESC")
    private List<Section> sections = new ArrayList<>();
}
```

```sql
-- @OrderBy 影响的 SQL
SELECT s.* FROM section s
WHERE s.article_id = ?
ORDER BY s.created_at ASC, s.title DESC;
```

### 3.5.3 Set

使用 `Set` 集合，元素唯一（基于 `equals/hashCode`），无重复。

```java
@Entity
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "product_tag",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();  // Set：每个 Tag 只能出现一次

    // 重新加载集合时依赖 equals/hashCode
    // 确保 Tag 的 equals/hashCode 设计正确
}
```

### 3.5.4 Map

```java
@Entity
public class Library {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    // @MapKey：用关联实体的某个字段作为 Map 的 key
    @OneToMany(mappedBy = "library")
    @MapKey(name = "isbn")  // Book.isbn 作为 Map key
    private Map<String, Book> booksByIsbn = new HashMap<>();

    // @MapKeyColumn：用额外的列作为 key
    @ElementCollection
    @CollectionTable(name = "library_settings", joinColumns = @JoinColumn(name = "library_id"))
    @MapKeyColumn(name = "setting_key")
    @Column(name = "setting_value")
    private Map<String, String> settings = new HashMap<>();

    // @MapKeyJoinColumn：用关联实体作为 key
    @OneToMany(mappedBy = "library")
    @MapKeyJoinColumn(name = "category_id")
    private Map<Category, Book> booksByCategory = new HashMap<>();
}
```

### 3.5.5 Bag（无排序语义的集合）

当 `@OneToMany` 使用 `List` 但**不加** `@OrderColumn` 时，Hibernate 将其视为 **Bag**——允许重复、无顺序保证的集合。

```java
@OneToMany(mappedBy = "order")
// 没有 @OrderColumn 也没有 @OrderBy
private List<OrderItem> items = new ArrayList<>();  // 这是 Bag！
```

Bag 的特点：
- 从数据库重新加载时元素顺序不保证
- 可以有重复元素
- 不使用索引列（性能比 @OrderColumn 好）
- 使用 `PersistentBag` 包装

```java
// 各集合类型对比
// 类型              数据库结构          重复  null  顺序         重新加载
// List+@OrderColumn  额外的索引列       Y    Y     List 顺序    保持顺序（但有大量 UPDATE）
// List+@OrderBy      无额外列           Y    Y     ORDER BY 排序 按 ORDER BY 排序
// Set                无额外列           N    N     无序         无序（依赖 equals）
// Bag (默认)         无额外列           Y    Y     无序         不保证顺序
// Map                无额外列或索引列    N    N     无序         无序
```

## 3.6 排序

### 3.6.1 @OrderBy（SQL 级排序）

通过 SQL 的 ORDER BY 子句排序，排序发生在数据库层面。

```java
@Entity
public class Category {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @OneToMany(mappedBy = "category")
    @OrderBy("price ASC, name DESC")  // 数据库级排序（字段名是实体属性名，不是列名）
    private List<Product> products = new ArrayList<>();
}
```

```sql
SELECT p.* FROM product p
WHERE p.category_id = ?
ORDER BY p.price ASC, p.name DESC;
```

### 3.6.2 @OrderColumn（持久化索引列）

```java
@Entity
public class Playlist {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "playlist", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderColumn(name = "track_order")
    private List<Track> tracks = new ArrayList<>();
}

@Entity
public class Track {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "title")
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "playlist_id")
    private Playlist playlist;
}
```

```sql
CREATE TABLE track (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255),
    playlist_id BIGINT,
    track_order INTEGER,
    FOREIGN KEY (playlist_id) REFERENCES playlist(id)
);
```

### 3.6.3 @SortNatural（JVM 内存排序）

实体实现 `Comparable` 接口，数据加载后在 JVM 内存中排序。

```java
@Entity
public class Chapter implements Comparable<Chapter> {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "title")
    private String title;

    @Column(name = "chapter_number")
    private Integer chapterNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id")
    private Book book;

    @Override
    public int compareTo(Chapter other) {
        return this.chapterNumber.compareTo(other.chapterNumber);
    }
}

@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "title")
    private String title;

    @OneToMany(mappedBy = "book")
    @SortNatural  // 使用 Comparable 在 JVM 内存中排序
    private SortedSet<Chapter> chapters = new TreeSet<>();
}
```

### 3.6.4 @SortComparator（自定义比较器）

```java
@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "book")
    @SortComparator(ChapterByTitleComparator.class)
    private SortedSet<Chapter> chapters = new TreeSet<>();
}

public class ChapterByTitleComparator implements Comparator<Chapter> {
    @Override
    public int compare(Chapter c1, Chapter c2) {
        return c1.getTitle().compareToIgnoreCase(c2.getTitle());
    }
}
```

### 3.6.5 SQL 级排序 vs JVM 级排序对比

| 维度 | @OrderBy（SQL 排序） | @SortNatural / @SortComparator（JVM 排序） |
|------|---------------------|------------------------------------------|
| 排序位置 | 数据库（ORDER BY 子句） | JVM 内存 |
| 性能 | 利用数据库索引，大数据量性能好 | 所有数据加载后排序，大数据量内存/CPU 压力大 |
| 集合类型 | List、Set | 必须 SortedSet（或使用 @SortNatural 的 Set） |
| 分页影响 | 数据库排序后分页，符合预期 | 先查询所有，再排序，再分页（分页不正确） |
| 动态性 | 静态（编译时确定） | 可通过代码动态改变 |
| 推荐场景 | 绝大多数场景 | 排序逻辑复杂、需要自定义比较规则 |

---

# 4. @ManyToMany

## 4.1 单向 @ManyToMany

```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "student_no", unique = true)
    private String studentNo;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",                    // 连接表名
        joinColumns = @JoinColumn(name = "student_id"),            // 本实体 FK
        inverseJoinColumns = @JoinColumn(name = "course_id")       // 对方实体 FK
    )
    private Set<Course> courses = new HashSet<>();

    public Student() {}
    public Student(String name, String studentNo) {
        this.name = name;
        this.studentNo = studentNo;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getStudentNo() { return studentNo; }
    public void setStudentNo(String studentNo) { this.studentNo = studentNo; }
    public Set<Course> getCourses() { return courses; }
    public void setCourses(Set<Course> courses) { this.courses = courses; }
}

@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "course_name", nullable = false)
    private String courseName;

    @Column(name = "credit")
    private Integer credit;

    public Course() {}
    public Course(String courseName, Integer credit) {
        this.courseName = courseName;
        this.credit = credit;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getCourseName() { return courseName; }
    public void setCourseName(String courseName) { this.courseName = courseName; }
    public Integer getCredit() { return credit; }
    public void setCredit(Integer credit) { this.credit = credit; }
}
```

```sql
CREATE TABLE student (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    student_no VARCHAR(255) UNIQUE
);

CREATE TABLE course (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    course_name VARCHAR(255) NOT NULL,
    credit INTEGER
);

CREATE TABLE student_course (
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES student(id),
    FOREIGN KEY (course_id) REFERENCES course(id)
);
```

```java
// 使用示例
Student student = new Student("Alice", "S2024001");
Course math = new Course("Mathematics", 4);
Course physics = new Course("Physics", 3);

student.getCourses().add(math);
student.getCourses().add(physics);

session.persist(student);  // cascade = PERSIST 会级联保存 Course
// INSERT INTO student ...
// INSERT INTO course ...
// INSERT INTO course ...
// INSERT INTO student_course (student_id, course_id) VALUES (?, ?)
// INSERT INTO student_course (student_id, course_id) VALUES (?, ?)
```

## 4.2 双向 @ManyToMany

双向 @ManyToMany 需要一方通过 `mappedBy` 声明为非拥有方。

```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "student_no", unique = true)
    private String studentNo;

    // 拥有方：持有 @JoinTable
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    public Student() {}
    public Student(String name, String studentNo) {
        this.name = name;
        this.studentNo = studentNo;
    }

    // ============ 双向同步辅助方法 ============

    public void addCourse(Course course) {
        if (course == null) return;
        this.courses.add(course);
        course.getStudents().add(this);
    }

    public void removeCourse(Course course) {
        if (course == null) return;
        this.courses.remove(course);
        course.getStudents().remove(this);
    }

    // getters / setters 省略
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getStudentNo() { return studentNo; }
    public void setStudentNo(String studentNo) { this.studentNo = studentNo; }
    public Set<Course> getCourses() { return courses; }
    public void setCourses(Set<Course> courses) { this.courses = courses; }
}

@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "course_name", nullable = false)
    private String courseName;

    @Column(name = "credit")
    private Integer credit;

    // 反向方：mappedBy 指向 Student 中的 "courses" 字段
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();

    public Course() {}
    public Course(String courseName, Integer credit) {
        this.courseName = courseName;
        this.credit = credit;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getCourseName() { return courseName; }
    public void setCourseName(String courseName) { this.courseName = courseName; }
    public Integer getCredit() { return credit; }
    public void setCredit(Integer credit) { this.credit = credit; }
    public Set<Student> getStudents() { return students; }
    public void setStudents(Set<Student> students) { this.students = students; }
}
```

**@ManyToMany 中级联注意事项**：

```java
// ✅ 推荐：仅使用 PERSIST 和 MERGE
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private Set<Course> courses;

// ❌ 危险：使用 REMOVE 或 ALL
@ManyToMany(cascade = CascadeType.ALL)  // 删除 Student 时可能意外删除 Course
private Set<Course> courses;

// 如果 Alice 和 Bob 都选了 "Math" 课程：
// session.remove(alice)  →  DELETE FROM course WHERE id = ?  ← Math 被删除了！
// Bob 再查询时发现 Math 课程不存在了！
```

## 4.3 @ManyToMany 的局限性

### 4.3.1 无法添加额外列

```java
// 需求：需要记录学生选课的 enrollDate 和 score
// @ManyToMany 无法实现！连接表只能有两个 FK 列
//
// student_course 表：
// +------------+-----------+-------------+-------+
// | student_id | course_id | enroll_date | score |
// +------------+-----------+-------------+-------+
// | 1          | 101       | 2024-09-01  | 85    |
// +------------+-----------+-------------+-------+
//
// @ManyToMany 只能映射前两列，无法映射 enroll_date 和 score
```

### 4.3.2 解决方案：中间实体模式

将 `@ManyToMany` 拆分为两个 `@OneToMany` + 一个中间实体，连接表升级为独立实体。

```java
// 中间实体：代替 student_course 连接表
@Entity
@Table(name = "enrollment")
public class Enrollment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id", nullable = false)
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id", nullable = false)
    private Course course;

    @Column(name = "enroll_date")
    private LocalDate enrollDate;

    @Column(name = "score")
    private Integer score;

    @Enumerated(EnumType.STRING)
    @Column(name = "status")
    private EnrollmentStatus status;

    public Enrollment() {}
    public Enrollment(Student student, Course course, LocalDate enrollDate) {
        this.student = student;
        this.course = course;
        this.enrollDate = enrollDate;
        this.status = EnrollmentStatus.ACTIVE;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Student getStudent() { return student; }
    public void setStudent(Student student) { this.student = student; }
    public Course getCourse() { return course; }
    public void setCourse(Course course) { this.course = course; }
    public LocalDate getEnrollDate() { return enrollDate; }
    public void setEnrollDate(LocalDate enrollDate) { this.enrollDate = enrollDate; }
    public Integer getScore() { return score; }
    public void setScore(Integer score) { this.score = score; }
    public EnrollmentStatus getStatus() { return status; }
    public void setStatus(EnrollmentStatus status) { this.status = status; }
}

public enum EnrollmentStatus {
    ACTIVE, COMPLETED, DROPPED
}

@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @OneToMany(mappedBy = "student", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Enrollment> enrollments = new ArrayList<>();

    // 便捷方法
    public void enroll(Course course, LocalDate date) {
        Enrollment enrollment = new Enrollment(this, course, date);
        enrollments.add(enrollment);
    }

    public Student() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public List<Enrollment> getEnrollments() { return enrollments; }
    public void setEnrollments(List<Enrollment> enrollments) { this.enrollments = enrollments; }
}

@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "course_name")
    private String courseName;

    @Column(name = "credit")
    private Integer credit;

    @OneToMany(mappedBy = "course")
    private List<Enrollment> enrollments = new ArrayList<>();

    public Course() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getCourseName() { return courseName; }
    public void setCourseName(String courseName) { this.courseName = courseName; }
    public Integer getCredit() { return credit; }
    public void setCredit(Integer credit) { this.credit = credit; }
    public List<Enrollment> getEnrollments() { return enrollments; }
    public void setEnrollments(List<Enrollment> enrollments) { this.enrollments = enrollments; }
}
```

```sql
CREATE TABLE enrollment (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    enroll_date DATE,
    score INTEGER,
    status VARCHAR(255),
    FOREIGN KEY (student_id) REFERENCES student(id),
    FOREIGN KEY (course_id) REFERENCES course(id)
);
```

### 4.3.3 @ManyToMany 的性能问题（全量删除再插入）

```java
// 当 @ManyToMany 使用 List（而非 Set）时，Hibernate 删除集合再重新插入
Student student = session.get(Student.class, 1L);
student.getCourses().remove(0);  // 从 List 中移除一个课程

// Hibernate 生成的 SQL：
// DELETE FROM student_course WHERE student_id = 1;       ← 全量删除！
// INSERT INTO student_course VALUES (1, remaining_course_1);
// INSERT INTO student_course VALUES (1, remaining_course_2);
// ......
```

```java
// ✅ 解决方案：使用 Set
@ManyToMany
@JoinTable(name = "student_course", ...)
private Set<Course> courses = new HashSet<>();

// 此时移除一个元素，Hibernate 只删除对应行：
// DELETE FROM student_course WHERE student_id = 1 AND course_id = 101;
```

**Set vs List in @ManyToMany**：

| 维度 | Set | List |
|------|-----|------|
| 删除行为 | 仅删除移除的元素对应行 | 全量删除后重新插入所有保留元素 |
| 重复元素 | 不支持 | 支持（会导致连接表无主键） |
| 排序 | 无序（可配合 @OrderBy） | 需 @OrderColumn（会产生大量 UPDATE） |
| 推荐度 | ✅ 推荐 | ❌ 不推荐 |

---

# 5. 关联加载策略

## 5.1 FetchType.LAZY 与 FetchType.EAGER

### 5.1.1 各关联类型的默认 FetchType

JPA 规范对各关联类型的默认加载策略有明确规定，但不同 JPA 提供者的实现可能不同。以下是 Hibernate 的实际行为：

| 关联类型 | JPA 规范默认 | Hibernate 实际 | 推荐设置 |
|----------|-------------|----------------|---------|
| @OneToOne | EAGER | EAGER | LAZY + optional=false |
| @ManyToOne | EAGER | EAGER | **必须改为 LAZY** |
| @OneToMany | LAZY | LAZY | LAZY（保持默认） |
| @ManyToMany | LAZY | LAZY | LAZY（保持默认） |
| @Basic（普通字段） | EAGER | EAGER | EAGER（保持默认） |
| @ElementCollection | LAZY | LAZY | LAZY（保持默认） |

### 5.1.2 为什么必须改 @ManyToOne 为 LAZY

```java
// ❌ 默认 EAGER 的灾难性后果

@Entity
public class Order {
    @Id private Long id;

    // 以下三个 @ManyToOne 默认全是 EAGER
    @ManyToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @ManyToOne
    @JoinColumn(name = "shipping_address_id")
    private Address shippingAddress;

    @ManyToOne
    @JoinColumn(name = "billing_address_id")
    private Address billingAddress;
}

// 查询所有订单：
List<Order> orders = session.createQuery("FROM Order", Order.class).list();

// 生成的 SQL（假设 3 个订单）：
// SELECT * FROM orders;                                     -- 1 条
// SELECT * FROM customers WHERE id = 1;                      -- N 条各 1 次
// SELECT * FROM customers WHERE id = 2;
// SELECT * FROM customers WHERE id = 3;
// SELECT * FROM addresses WHERE id = 10;                    -- N 条各 1 次
// SELECT * FROM addresses WHERE id = 11;
// SELECT * FROM addresses WHERE id = 12;
// SELECT * FROM addresses WHERE id = 13;                    -- 又来 N 条
// ...
// 总共 1 + 3 + 3 + 3 = 10 条 SQL！这就是 N+1 问题
```

```java
// ✅ 正确做法
@Entity
public class Order {
    @Id private Long id;

    @ManyToOne(fetch = FetchType.LAZY)  // 显式设 LAZY
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "shipping_address_id")
    private Address shippingAddress;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "billing_address_id")
    private Address billingAddress;
}
```

## 5.2 懒加载原理

### 5.2.1 代理对象（Proxy）拦截

对于 `@ManyToOne` 和 `@OneToOne` 等单值关联，Hibernate 使用字节码生成的**代理对象**来实现懒加载。

```java
// 代理对象的本质
// session.get(User.class, 1L) 实际查询到的 user 类型：
// 对于 EAGER：com.example.User —— 真实的 User 对象
// 对于 LAZY：com.example.User$HibernateProxy$xyz —— CGLIB/ByteBuddy 代理

Customer customer = order.getCustomer();  // 此时获取到的是代理对象
// 代理对象中只包含 id，没有其他字段值

String name = customer.getName();  // ← 触发懒加载！
// 代理拦截 getName() → 执行 SELECT * FROM customers WHERE id = ?
```

**Hibernate.initialize()** 强制初始化懒加载关联：

```java
Customer customer = order.getCustomer();  // 代理，未加载
Hibernate.initialize(customer);            // 强制执行 SQL 查询
// 等同于 customer.getName(); // 也会触发

// 检查是否已初始化
boolean initialized = Hibernate.isInitialized(customer);
```

### 5.2.2 集合包装器

对于 `@OneToMany` 和 `@ManyToMany` 等集合关联，Hibernate 使用专门的集合包装器：

| Java 类型 | Hibernate 包装器 | 说明 |
|-----------|-----------------|------|
| List（无 @OrderColumn） | `PersistentBag` | Bag 语义 |
| List + @OrderColumn | `PersistentList` | 有序 List |
| Set | `PersistentSet` | 唯一集 |
| SortedSet | `PersistentSortedSet` | 排序集 |
| Map | `PersistentMap` | 键值对 |
| SortedMap | `PersistentSortedMap` | 排序映射 |

```java
// 集合包装器的行为
List<OrderItem> items = order.getItems();
// items 的实际类型是 PersistentBag（不是 ArrayList）
// 此时集合中还没有数据

int size = items.size();  // ← 触发懒加载！
// SELECT * FROM order_item WHERE order_id = ?

OrderItem first = items.get(0);  // 如果已经加载过，不会再次查询
```

### 5.2.3 懒加载异常（LazyInitializationException）

```java
// 典型的 LazyInitializationException 场景
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

Order order = session.get(Order.class, 1L);
// Customer 是 LAZY 关联，此时只是代理对象

tx.commit();
session.close();  // Session 关闭！

// 访问懒加载关联 → 异常
String customerName = order.getCustomer().getName();
// org.hibernate.LazyInitializationException:
// could not initialize proxy [com.example.Customer#1] - no Session
```

**解决方案**：

```java
// 方案一：在 Session 关闭前触发懒加载
session.getTransaction().begin();
Order order = session.get(Order.class, 1L);
Hibernate.initialize(order.getCustomer());       // 强制加载
Hibernate.initialize(order.getItems());          // 强制加载
session.getTransaction().commit();
session.close();
String name = order.getCustomer().getName();  // OK

// 方案二：使用 JOIN FETCH 一次性加载
Order order = session.createQuery(
    "SELECT o FROM Order o JOIN FETCH o.customer WHERE o.id = :id", Order.class)
    .setParameter("id", 1L)
    .uniqueResult();

// 方案三：Open Session in View（不推荐，但常用）
// 将 Session 绑定到请求生命周期，在视图渲染完成后关闭
```

## 5.3 抓取策略（@Fetch）

Hibernate 提供了四种 `FetchMode`，控制关联数据的抓取方式。

### 5.3.1 @Fetch(FetchMode.JOIN)

使用 SQL LEFT OUTER JOIN 一次性加载主实体和关联实体。

```java
@Entity
public class Order {
    @Id private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @Fetch(FetchMode.JOIN)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

```sql
-- FetchMode.JOIN 生成的 SQL（覆盖 LAZY，总是立即加载）
SELECT o.*, c.*
FROM orders o
LEFT OUTER JOIN customers c ON o.customer_id = c.id
WHERE o.id = ?;
```

**注意**：`FetchMode.JOIN` 会**覆盖** `FetchType.LAZY` 设置，变为立即加载。通常不推荐，因为它使懒加载失效。

### 5.3.2 @Fetch(FetchMode.SELECT)

默认的抓取策略：先查主实体，再为每个实体单独查询关联（N+1 问题）。

```java
@ManyToOne(fetch = FetchType.LAZY)
@Fetch(FetchMode.SELECT)  // 默认行为，可以省略
@JoinColumn(name = "customer_id")
private Customer customer;
```

```sql
SELECT * FROM orders WHERE ...;           -- 1 条
SELECT * FROM customers WHERE id = ?;     -- 每个 order 一条
SELECT * FROM customers WHERE id = ?;
SELECT * FROM customers WHERE id = ?;
```

### 5.3.3 @Fetch(FetchMode.SUBSELECT)

使用子查询方式批量加载关联：先查出所有主实体，再用一条子查询查出所有关联。

```java
@Entity
public class Order {
    @Id private Long id;

    @OneToMany(mappedBy = "order")
    @Fetch(FetchMode.SUBSELECT)  // 使用子查询
    private List<OrderItem> items;
}
```

```sql
-- FetchMode.SUBSELECT 生成的 SQL
SELECT * FROM orders WHERE status = 'PENDING';  -- 查出 3 个订单

-- 访问第一个 order.getItems() 时：
SELECT * FROM order_item
WHERE order_id IN (
    SELECT id FROM orders WHERE status = 'PENDING'
);
-- 所有订单的 OrderItem 一次性加载完成
```

### 5.3.4 @Fetch(FetchMode.BATCH) + @BatchSize

批量加载：当访问第 N 个代理时，同时加载前 N-1 个代理。

```java
@Entity
@BatchSize(size = 10)
public class Customer {
    @Id private Long id;
    // ...
}

@Entity
public class Order {
    @Id private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

```sql
-- 查询 15 个 Order，访问第 10 个 order.getCustomer() 时：
-- 一次性加载前 10 个 Customer：
SELECT * FROM customers WHERE id IN (1, 2, 3, ..., 10);

-- 访问第 11 个时：
SELECT * FROM customers WHERE id IN (11, 12, ..., 15);
```

### 5.3.5 抓取策略对比

| 策略 | SQL 次数 | 适用场景 | 注意事项 |
|------|---------|---------|---------|
| JOIN | 1 条 | 总是需要关联数据 | 覆盖 LAZY，可能返回大量不必要的字段，笛卡尔积问题 |
| SELECT | 1 + N | 偶尔访问关联 | N+1 问题严重 |
| SUBSELECT | 1 + 1 | 批量访问集合关联 | 子查询可能返回大量数据 |
| BATCH | 1 + ⌈N/B⌉ | 批量访问单值关联 | 需要设置合理的 batch size |

## 5.4 解决 N+1 问题

### 5.4.1 JOIN FETCH（JPQL/HQL）

```java
// JPQL JOIN FETCH：一次性加载主实体和关联实体
List<Order> orders = session.createQuery(
    "SELECT DISTINCT o FROM Order o " +
    "JOIN FETCH o.customer " +
    "JOIN FETCH o.items " +
    "WHERE o.status = :status", Order.class)
    .setParameter("status", "PENDING")
    .list();

// Criteria API 的 fetch
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> root = cq.from(Order.class);
root.fetch("customer", JoinType.LEFT);
root.fetch("items", JoinType.LEFT);
cq.select(root).distinct(true);
cq.where(cb.equal(root.get("status"), "PENDING"));
List<Order> orders = session.createQuery(cq).list();
```

### 5.4.2 EntityGraph（JPA 2.1）

```java
// 定义命名 EntityGraph
@Entity
@NamedEntityGraph(
    name = "Order.withCustomerAndItems",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode("items")
    }
)
public class Order {
    @Id private Long id;
    @ManyToOne(fetch = FetchType.LAZY) private Customer customer;
    @OneToMany(mappedBy = "order") private List<OrderItem> items;
}

// 使用命名 EntityGraph
EntityGraph<Order> graph = session.createEntityGraph(Order.class);
graph.addSubgraph("items");  // 加载子图

List<Order> orders = session.createQuery("FROM Order", Order.class)
    .setHint("jakarta.persistence.fetchgraph", graph)
    .list();

// 动态创建 EntityGraph
EntityGraph<Order> dynamicGraph = session.createEntityGraph(Order.class);
dynamicGraph.addAttributeNodes("customer");
Subgraph<OrderItem> itemSubgraph = dynamicGraph.addSubgraph("items");
itemSubgraph.addAttributeNodes("product");

List<Order> orders = session.createQuery("FROM Order", Order.class)
    .setHint("jakarta.persistence.loadgraph", dynamicGraph)
    .list();
```

**fetchgraph vs loadgraph**：

- `jakarta.persistence.fetchgraph`：**仅**加载 EntityGraph 中指定的属性，未指定的即使是 EAGER 也按 LAZY 处理
- `jakarta.persistence.loadgraph`：加载 EntityGraph 指定的属性 + 所有原本为 EAGER 的属性

### 5.4.3 @BatchSize

```java
// 类级别 @BatchSize
@Entity
@BatchSize(size = 20)
public class Customer {
    @Id private Long id;
    private String name;
}

// 集合级别 @BatchSize
@Entity
public class Order {
    @Id private Long id;

    @OneToMany(mappedBy = "order")
    @BatchSize(size = 50)
    private List<OrderItem> items;
}
```

### 5.4.4 全局配置 default_batch_fetch_size

在 `persistence.xml` 或 `application.yml` 中全局设置：

```xml
<!-- persistence.xml -->
<property name="hibernate.default_batch_fetch_size" value="16"/>
```

```yaml
# application.yml (Spring Boot)
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 16
```

这个配置影响所有懒加载关联的批量加载行为，无需为每个实体单独配置 `@BatchSize`。推荐值：16 或 32（经验数值，平衡内存与查询次数）。

### 5.4.5 N+1 解决方案对比

| 方案 | 优点 | 缺点 | 推荐场景 |
|------|-----|------|---------|
| JOIN FETCH | 1 条 SQL，明确清晰 | 笛卡尔积，可能返回大量重复数据 | 需要多层关联且数据量可控 |
| EntityGraph | 声明式、可复用 | 定义较繁琐，不支持动态过滤 | 不同业务场景需要不同加载策略 |
| @BatchSize | 透明、无需改查询 | 仍然分多批查询 | 懒加载场景，偶尔访问关联 |
| SUBSELECT | 2 条 SQL | 子查询可能很大 | 集合关联批量加载 |
| default_batch_fetch_size | 全局生效、零配置 | 无法精细控制 | 作为默认兜底方案 |

---

# 6. 关联高级特性

## 6.1 @NotFound

适用于**遗留数据库**中外键约束不完整（FOREIGN KEY 引用可能缺失）的场景。

```java
@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    @NotFound(action = NotFoundAction.IGNORE)  // 外键值对应的部门不存在时，设为 null
    private Department department;

    // 如果 department_id 的值在 department 表中不存在：
    // IGNORE：  department 字段设为 null（不抛异常）
    // EXCEPTION：抛出 EntityNotFoundException（默认行为）
}
```

```sql
-- 遗留数据库中的数据：
-- employee: id=1, name='Alice', department_id=999
-- department: 没有 id=999 的记录（外键悬空）

-- 使用 @NotFound(IGNORE) 后：
-- employee.getDepartment() 返回 null，不抛异常
```

**风险**：
- 关联值可能为 null，业务代码需要空判断
- EAGER 加载时如果外键缺失，LEFT JOIN 也会返回 null

## 6.2 @JoinFormula

基于 SQL 表达式（而非简单的外键等值匹配）进行关联。

```java
@Entity
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "balance")
    private BigDecimal balance;

    @Column(name = "currency")
    private String currency;

    // 不是直接的 FK 关联，而是通过币种匹配
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinFormula(
        "(" +
        "  SELECT r.id FROM exchange_rate r " +
        "  WHERE r.from_currency = currency " +
        "  AND r.to_currency = 'CNY' " +
        "  ORDER BY r.rate_date DESC LIMIT 1" +
        ")"
    )
    private ExchangeRate exchangeRate;
}

@Entity
public class ExchangeRate {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "from_currency")
    private String fromCurrency;

    @Column(name = "to_currency")
    private String toCurrency;

    @Column(name = "rate")
    private BigDecimal rate;

    @Column(name = "rate_date")
    private LocalDate rateDate;
}
```

## 6.3 @Any / @ManyToAny（多态关联）

关联到不同表的不同实体类型（类似 Rails 的 Polymorphic Association）。

```java
// 使用场景：评论系统
// Comment 的 target 可以是 Article、Video、Image 等不同实体

// 先定义元数据
@AnyMetaDefs({
    @AnyMetaDef(
        name = "CommentTargetMetaDef",
        idType = "long",
        metaType = "string",
        metaValues = {
            @MetaValue(value = "ARTICLE", targetEntity = Article.class),
            @MetaValue(value = "VIDEO", targetEntity = Video.class),
            @MetaValue(value = "IMAGE", targetEntity = Image.class)
        }
    )
})
// 可以放在 package-info.java 中

@Entity
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "content")
    private String content;

    @Any(metaDef = "CommentTargetMetaDef", metaColumn = @Column(name = "target_type"))
    @JoinColumn(name = "target_id")
    private Object target;  // 可以是 Article、Video 或 Image

    public Comment() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public Object getTarget() { return target; }
    public void setTarget(Object target) { this.target = target; }
}
```

```sql
CREATE TABLE comment (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content VARCHAR(255),
    target_type VARCHAR(255),  -- 'ARTICLE', 'VIDEO', 'IMAGE'
    target_id BIGINT,
    -- 注意：没有 FOREIGN KEY 约束！因为 target_id 指向不同表
);
```

**不推荐原因**：
- 失去外键约束，数据完整性无法保证
- 无法使用 JOIN 查询（JPQL 也不支持）
- 类型不安全（target 是 Object）
- 可替代方案：继承映射（单表/联合表）、单独建表

## 6.4 @Where 与 @Filter

### 6.4.1 @Where（静态过滤）

在 SQL 查询中自动附加 WHERE 条件，常用于软删除。

```java
@Entity
@Table(name = "product")
@Where(clause = "deleted = false")  // 对所有 Product 查询自动追加此条件
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @Column(name = "deleted")
    private Boolean deleted = false;

    // 关联也自动过滤
    @OneToMany(mappedBy = "product")
    @Where(clause = "deleted = false")
    private List<ProductImage> images = new ArrayList<>();
}

@Entity
@Table(name = "product_image")
@Where(clause = "deleted = false")
public class ProductImage {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "url")
    private String url;

    @Column(name = "deleted")
    private Boolean deleted = false;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;
}
```

```sql
-- product.getImages() 自动生成：
SELECT * FROM product_image
WHERE product_id = ? AND deleted = false;  -- @Where 自动追加
```

### 6.4.2 @Filter / @FilterDef（动态过滤）

运行时按需启用/禁用的过滤条件，比 @Where 更灵活。

```java
@Entity
@Table(name = "product")
@FilterDef(
    name = "tenantFilter",
    parameters = @ParamDef(name = "tenantId", type = Long.class)
)
@Filter(
    name = "tenantFilter",
    condition = "tenant_id = :tenantId"
)
@FilterDef(
    name = "activeFilter",
    parameters = @ParamDef(name = "isActive", type = Boolean.class)
)
@Filter(
    name = "activeFilter",
    condition = "active = :isActive"
)
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @Column(name = "tenant_id")
    private Long tenantId;

    @Column(name = "active")
    private Boolean active = true;
}
```

```java
// 运行时启用过滤器
Session session = sessionFactory.openSession();

// 启用多租户过滤
session.enableFilter("tenantFilter")
       .setParameter("tenantId", 100L);

// 启用活跃状态过滤
session.enableFilter("activeFilter")
       .setParameter("isActive", true);

// 后续所有 Product 查询都自动带上 tenant_id = 100 AND active = true
List<Product> products = session.createQuery("FROM Product", Product.class).list();

// 查询完毕后可以禁用
session.disableFilter("tenantFilter");
```

```sql
-- enableFilter 后自动在 WHERE 子句追加条件
SELECT p.* FROM product p
WHERE p.tenant_id = 100 AND p.active = true;
```

### 6.4.3 @FilterJoinTable

针对 @ManyToMany 的中间表进行过滤。

```java
@Entity
public class Student {
    @Id private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    @FilterJoinTable(
        name = "activeCourseFilter",
        condition = "status = 'ACTIVE'"
    )
    private Set<Course> courses;
}
```

## 6.5 @FetchProfile

定义命名获取策略，在运行时选择使用。

```java
@FetchProfile(name = "order-with-customer", fetchOverrides = {
    @FetchProfile.FetchOverride(entity = Order.class, association = "customer", mode = FetchMode.JOIN)
})
@FetchProfile(name = "order-with-items", fetchOverrides = {
    @FetchProfile.FetchOverride(entity = Order.class, association = "items", mode = FetchMode.JOIN)
})
@FetchProfile(name = "order-with-everything", fetchOverrides = {
    @FetchProfile.FetchOverride(entity = Order.class, association = "customer", mode = FetchMode.JOIN),
    @FetchProfile.FetchOverride(entity = Order.class, association = "items", mode = FetchMode.JOIN)
})
// 通常定义在 package-info.java 中
```

```java
// 运行时启用不同的获取策略
Session session = sessionFactory.openSession();

// 场景一：仅需要订单 + 客户信息
session.enableFetchProfile("order-with-customer");
List<Order> orders1 = session.createQuery("FROM Order", Order.class).list();
// SQL: SELECT o.*, c.* FROM orders o LEFT JOIN customers c ON ...

// 场景二：仅需要订单 + 订单项
session.disableFetchProfile("order-with-customer");
session.enableFetchProfile("order-with-items");
List<Order> orders2 = session.createQuery("FROM Order", Order.class).list();
// SQL: SELECT o.*, i.* FROM orders o LEFT JOIN order_item i ON ...
```

---

# 7. 关联映射常见问题

## 7.1 级联操作陷阱

### 7.1.1 CascadeType.ALL + @ManyToMany 的意外删除

```java
// ❌ 危险配置
@Entity
public class Student {
    @ManyToMany(cascade = CascadeType.ALL)  // 包含 REMOVE！
    @JoinTable(name = "student_course", ...)
    private Set<Course> courses;
}

// 场景：Alice 和 Bob 都选了 Math 课程
Student alice = session.get(Student.class, 1L);
session.remove(alice);  // 级联删除 Math 课程！
// Bob 再次查询 → Math 课程已消失
// 连接表中的 student_id=1 的记录也被删除（这是正确的），但 Math 课程也被删了！
```

```java
// ✅ 正确配置
@Entity
public class Student {
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(name = "student_course", ...)
    private Set<Course> courses;
}

// 现在 remove(student) 只删除学生记录和连接表记录，不删除课程
```

### 7.1.2 级联 REMOVE 与 orphanRemoval 的配合

```java
// 典型配置：@OneToMany 中两者都设
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Child> children = new ArrayList<>();

// 行为矩阵：
//                        | cascade.REMOVE    | orphanRemoval=true
// -----------------------|-------------------|--------------------
// parent.children.remove(child) | 不删除 child | 删除 child（DELETE SQL）
// session.remove(parent)        | 删除 children | 删除 children
```

### 7.1.3 级联操作的持久化顺序

```java
// cascade = CascadeType.PERSIST
// persist 的顺序是：父 → 子（递归）
Parent parent = new Parent();
Child child = new Child();
parent.addChild(child);

session.persist(parent);  // 先 INSERT parent，再 INSERT child

// cascade = CascadeType.REMOVE
// remove 的顺序是：子 → 父（递归）—— 因为外键约束
session.remove(parent);   // 先 DELETE children, 再 DELETE parent
```

## 7.2 双向关联陷阱

### 7.2.1 忘记同步两端

```java
// ❌ 只设置一端——对象模型不一致
Order order = new Order("ORD-001");
OrderItem item = new OrderItem("Product", 1, BigDecimal.TEN);
item.setOrder(order);               // 只设置了 OrderItem → Order
// order.getItems() 仍为 空集合！     // 没有设置 Order → OrderItem
session.persist(order);             // 级联 persist item
session.persist(item);

// 事务提交后，order.getItems() 为 空（同一个 Session 中）—— 对象状态不一致！

// ✅ 必须同步两端
Order order = new Order("ORD-001");
OrderItem item = new OrderItem("Product", 1, BigDecimal.TEN);
order.addOrderItem(item);  // 辅助方法同步两端
// 内部：items.add(item); item.setOrder(this);
```

### 7.2.2 JSON 序列化循环引用

```java
// ❌ 无任何处理 → 序列化时无限递归
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;
}
@Entity
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
}
// JSON: { "id": 1, "items": [ { "id": 1, "order": { "id": 1, "items": [ ... ] } } ] }

// ✅ 方案一：@JsonIgnore（简单粗暴）
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id")
    @JsonIgnore
    private Order order;
}

// ✅ 方案二：@JsonManagedReference / @JsonBackReference（保留正向）
public class Order {
    @OneToMany(mappedBy = "order")
    @JsonManagedReference("order-items")  // 正向：序列化时包含
    private List<OrderItem> items;
}
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id")
    @JsonBackReference("order-items")  // 反向：序列化时排除
    private Order order;
}
// JSON: { "id": 1, "items": [ { "id": 1, "product": "..." }, ... ] }

// ✅ 方案三：@JsonIdentityInfo（使用对象标识消除循环）
@Entity
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class Order { ... }

@Entity
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class OrderItem { ... }
// JSON: { "id": 1, "items": [ { "id": 1, "order": 1 } ] }  ← order 只用 id 引用
```

### 7.2.3 toString() 循环引用 → StackOverflow

```java
// ❌ 危险
@Override
public String toString() {
    return "Order{id=" + id + ", items=" + items + '}';
}

// ✅ 安全
@Override
public String toString() {
    return "Order{id=" + id +
           ", orderNumber='" + orderNumber + '\'' +
           ", totalAmount=" + totalAmount +
           ", itemCount=" + items.size() + '}';
}
```

### 7.2.4 Lombok @Data 与双向关联冲突

```java
// ❌ @Data 生成的 equals/hashCode/toString 包含所有字段
@Data  // 等于 @Getter + @Setter + @EqualsAndHashCode + @ToString
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;
    // toString() 包含 items → items 包含 order → 循环
    // equals()/hashCode() 包含 items → 检查时触发懒加载
}

// ✅ 选择性使用 Lombok
@Entity
@Getter
@Setter
@ToString(exclude = "items")          // 排除双向关联
@EqualsAndHashCode(onlyExplicitlyIncluded = true)  // 仅包含指定字段
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @EqualsAndHashCode.Include
    private Long id;

    @OneToMany(mappedBy = "order")
    @ToString.Exclude
    private List<OrderItem> items;
}
```

## 7.3 集合性能陷阱

### 7.3.1 @OneToMany 的 List 默认是 Bag

```java
// 默认的 List（无 @OrderColumn）是 Bag，重加载后顺序不保证
@OneToMany(mappedBy = "order")
private List<OrderItem> items = new ArrayList<>();

// 场景：
Order order = session.get(Order.class, 1L);
List<OrderItem> items = order.getItems();  // items[0] = 'Keyboard', items[1] = 'Mouse'
session.clear();

// 重新加载
order = session.get(Order.class, 1L);
items = order.getItems();  // items[0] = 'Mouse', items[1] = 'Keyboard' ← 顺序变了！
```

### 7.3.2 @OrderColumn 的 UPDATE 性能问题

```java
// 有 100 个元素的 @OrderColumn List
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
@OrderColumn(name = "position")
private List<Child> children = new ArrayList<>();

// 在位置 5 插入一个元素：
children.add(5, newChild);
// Hibernate 生成：1 条 INSERT + 95 条 UPDATE！
// UPDATE child SET position = 6 WHERE id = 6;
// UPDATE child SET position = 7 WHERE id = 7;
// ...
// UPDATE child SET position = 100 WHERE id = 100;

// 解决方案：只在"顺序不常变化"或"元素数量不大"时使用 @OrderColumn
```

### 7.3.3 大集合的计数操作

```java
// ❌ 性能陷阱
@OneToMany(mappedBy = "parent")
private List<Child> children;

// children.size() 会加载整个集合！
int count = parent.getChildren().size();
// SELECT * FROM child WHERE parent_id = ?  ← 加载所有子实体到内存

// ✅ 使用专门的 count 查询
Long count = session.createQuery(
    "SELECT COUNT(c) FROM Child c WHERE c.parent.id = :pid", Long.class)
    .setParameter("pid", parent.getId())
    .uniqueResult();

// ✅ 或使用 @Formula 计算计数值
@Entity
public class Parent {
    @Id private Long id;

    @Formula("(SELECT COUNT(*) FROM child c WHERE c.parent_id = id)")
    private int childCount;  // 每次查询 Parent 时自动计算
}
```

## 7.4 equals/hashCode 与关联

### 7.4.1 关联实体作为 Set 元素时的重要性

```java
// ❌ 错误的 equals/hashCode 实现
@Entity
public class Tag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tag)) return false;
        Tag tag = (Tag) o;
        return Objects.equals(id, tag.id);  // 只基于 id
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// 问题：Set.add() 发生在 persist 前，id 为 null
Tag tag = new Tag();
tag.setName("Java");
Set<Tag> tags = new HashSet<>();
tags.add(tag);          // hashCode 基于 null → 0
session.persist(tag);   // id 变为 1
tags.contains(tag);     // hashCode 基于 1 → 但之前存的时候是 0 → false!
```

```java
// ✅ 稳定的 equals/hashCode 实现
@Entity
public class Tag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", unique = true)
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tag)) return false;
        Tag tag = (Tag) o;
        // 使用业务键（business key），而非数据库 id
        return Objects.equals(name, tag.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);  // 基于业务键
    }
}
```

### 7.4.2 避免在 equals/hashCode 中包含懒加载关联

```java
// ❌ 错误：equals 中包含 @ManyToOne 关联
@EqualsAndHashCode  // Lombok → 包含所有字段
@Entity
public class OrderItem {
    @Id private Long id;
    private String productName;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;  // 懒加载！
}
// 调用 item1.equals(item2) → 可能触发对 Order 的懒加载！

// ✅ 正确：eq/hash 仅基于数据库标识符
@Entity
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderItem)) return false;
        return id != null && Objects.equals(id, ((OrderItem) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();  // 或 EntityClass.hashCode()
    }
}
```

**核心原则**：

1. **实体标识符（id）**：最简单可靠的 equals/hashCode 基础（需处理 id 为 null 的情况）
2. **业务键（Business Key）**：如 `username`、`email` 等不可变的唯一业务字段
3. **永远不要包含**：懒加载的关联字段、自动生成的值（如 `createdAt`）、可变字段
4. **Set vs List**：使用 Set 时必须正确实现 equals/hashCode；使用 Bag（List 无 @OrderColumn）可以依赖实体标识
