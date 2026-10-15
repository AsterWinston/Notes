# 1. 定义与作用

## 1.1 什么是 Hibernate

Hibernate 是一个开源的 **对象关系映射（ORM, Object-Relational Mapping）** 框架，用于在 Java 应用程序中实现领域模型对象与关系型数据库表之间的双向映射。其核心思想是：开发者操作 Java 对象，框架自动将这些操作转换为对应的 SQL 语句并执行，从而屏蔽底层数据库操作的细节。

**ORM 概念**：在面向对象编程中，数据以对象图的形式存在；在关系型数据库中，数据以行和列的形式存储。ORM 的目的就是在这两种范式之间架设桥梁，解决"对象-关系阻抗不匹配"问题。

Hibernate 同时也是 **JPA（Jakarta Persistence API，原 Java Persistence API）规范** 的参考实现（Reference Implementation）之一。在 Java EE / Jakarta EE 生态中，JPA 定义了持久化的标准接口，而 Hibernate 提供了这些接口的具体实现。

**核心价值**：
- **透明持久化（Transparent Persistence）**：开发者无需感知底层 SQL，框架自动跟踪对象状态变化并同步到数据库。
- **数据库无关性（Database Independence）**：通过 Dialect 抽象层，同一套代码可运行在 MySQL、PostgreSQL、Oracle 等多种数据库上。
- **减少 SQL 编写**：常见 CRUD 操作完全自动化，开发者聚焦业务逻辑而非 SQL 拼接。

## 1.2 核心能力

Hibernate 提供了一套完整的持久化解决方案，其核心能力包括：

### 1.2.1 对象持久化与 CRUD 自动化

```java
// 保存
session.persist(user);

// 查询
User user = session.get(User.class, 1L);

// 更新（通过脏检查自动同步）
user.setName("New Name");

// 删除
session.remove(user);
```

### 1.2.2 关联管理

支持实体之间的一对一（@OneToOne）、一对多/多对一（@OneToMany / @ManyToOne）、多对多（@ManyToMany）关联映射，并能自动处理级联操作。

```java
@Entity
public class Order {
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;
}
```

### 1.2.3 继承映射

支持单表继承（SINGLE_TABLE）、连接表继承（JOINED）、每个具体类一张表（TABLE_PER_CLASS）三种策略：

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment {
    @Id @GeneratedValue
    private Long id;
    private BigDecimal amount;
}

@Entity
public class CreditCardPayment extends Payment {
    private String cardNumber;
}

@Entity
public class AlipayPayment extends Payment {
    private String accountId;
}
```

### 1.2.4 查询能力

- **HQL（Hibernate Query Language）**：面向对象的查询语言，操作的是实体而非表。

```java
List<User> users = session.createQuery(
    "from User u where u.age > :age and u.status = :status", User.class)
    .setParameter("age", 18)
    .setParameter("status", Status.ACTIVE)
    .getResultList();
```

- **Criteria API**：类型安全的编程式查询。

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.select(root).where(cb.greaterThan(root.get("age"), 18));
List<User> users = session.createQuery(cq).getResultList();
```

- **Native SQL**：直接执行原生 SQL 并映射为实体。

```java
List<User> users = session.createNativeQuery(
    "SELECT * FROM t_user WHERE age > ?1", User.class)
    .setParameter(1, 18)
    .getResultList();
```

### 1.2.5 缓存管理

- **一级缓存（Session 级别）**：与 Session 生命周期绑定，默认开启。
- **二级缓存（SessionFactory 级别）**：跨 Session 共享，需显式配置（如 Ehcache、Hazelcast、Infinispan）。
- **查询缓存**：缓存 HQL / Criteria 查询结果集的标识符。

### 1.2.6 事务管理

通过 `session.getTransaction()` 或集成 JTA 实现本地事务和分布式事务管理。

```java
Transaction tx = session.beginTransaction();
try {
    // 业务操作
    tx.commit();
} catch (Exception e) {
    tx.rollback();
    throw e;
}
```

### 1.2.7 锁机制

- **乐观锁**：通过 `@Version` 注解实现，使用版本号或时间戳检测并发冲突。
- **悲观锁**：使用 `LockModeType.PESSIMISTIC_READ` / `PESSIMISTIC_WRITE` 在数据库层面加锁。

```java
@Entity
public class Product {
    @Version
    private int version;
}

// 悲观锁
Product product = session.find(Product.class, id,
    LockModeType.PESSIMISTIC_WRITE);
```

### 1.2.8 模式生成（SchemaExport）

Hibernate 可以根据映射元数据自动生成 DDL 语句，支持 `validate`、`update`、`create`、`create-drop` 等策略：

```xml
<property name="hibernate.hbm2ddl.auto" value="validate" />
```

### 1.2.9 懒加载与代理

关联属性和集合默认使用懒加载（Lazy Loading），通过字节码增强或代理对象延迟初始化，避免不必要的数据查询。

```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders; // 仅在访问时加载
```

## 1.3 为什么选择 Hibernate

### 1.3.1 数据库无关性（Dialect 机制）

Hibernate 通过 Dialect 接口抽象了不同数据库的 SQL 方言差异，切换数据库只需修改配置文件中的 Dialect 类名和驱动信息，无需修改业务代码：

```xml
<!-- MySQL 8 -->
<property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect" />
<!-- PostgreSQL -->
<property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect" />
```

### 1.3.2 减少样板代码

对比原始 JDBC 编程，Hibernate 消除了大量的 `try-catch-finally`、资源管理、`ResultSet` 遍历和手动映射代码。一个 JDBC 插入需要 15+ 行代码，Hibernate 仅需 1 行。

**JDBC 方式**：
```java
String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
try (PreparedStatement stmt = conn.prepareStatement(sql)) {
    stmt.setString(1, user.getName());
    stmt.setString(2, user.getEmail());
    stmt.executeUpdate();
}
```

**Hibernate 方式**：
```java
session.persist(user);
```

### 1.3.3 性能优化手段丰富

Hibernate 提供了丰富的性能调优手段，详见本书性能优化章节。

### 1.3.4 生态成熟

Hibernate 与 Spring Framework 深度集成（Spring Data JPA / Spring Boot Starter Data JPA），是目前 Java 企业级开发的事实标准。几乎所有面试和招聘要求中，JPA / Hibernate 都是必备技能。

### 1.3.5 社区活跃

拥有庞大的用户群、完善的文档、众多的第三方扩展和活跃的维护团队。Hibernate 6.x 版本仍在持续演进。

# 2. 发展历程

## 2.1 起源与早期版本

Hibernate 由 **Gavin King** 于 **2001 年** 创建。当时 Java 持久化领域主流是 EJB 2.0 的 Entity Bean，但存在性能差、复杂性高、耦合度强等问题。Gavin King 在开发一款保险系统时，对 EJB 2 的局限性感到不满，从而开创性地实现了轻量级 ORM 框架 Hibernate。

- **Hibernate 2.x（2003-2005）**：首个成熟的版本线，引入了基础的 ORM 映射、HQL、Criteria 查询、缓存机制。2003 年加入 JBoss。
- **Hibernate 3.x（2005-2010）**：里程碑式版本，引入 **注解（Annotation）** 支持（3.2+）、Criteria API 重构（3.0+）、Event / Listener 架构（3.0+）、拦截器（Interceptor）机制、过滤器（Filter）、Hibernate Shards 等。成为 Java 持久化领域的事实标准。
- **Hibernate 3.5+**：正式成为 **JPA 2.0** 的参考实现，标志着 Hibernate 从"自有标准"到"行业规范"的转变。首次引入 Envers（数据审计模块）。

## 2.2 Hibernate 4.x（2011-2015）

Hibernate 4 是一次重大架构重构：

- **ServiceRegistry 重构**：将原有的 `Configuration` 拆分为 `StandardServiceRegistry`，实现了 Bootstrap 过程的解耦和更灵活的扩展机制。

```java
// Hibernate 4/5 中的标准启动方式
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .configure()
    .build();
SessionFactory sessionFactory = new MetadataSources(registry)
    .buildMetadata()
    .buildSessionFactory();
```

- **Multi-tenancy 支持**：支持 Database-per-tenant、Schema-per-tenant、Discriminator 等多种多租户策略。
- **二级缓存重构**：与 Infinispan、Ehcache 等的集成更加标准和灵活。
- **弃用 HibernateTemplate**：Hibernate 4 起官方不推荐使用 Spring 的 `HibernateTemplate`，转而推荐原生 Session API。

## 2.3 Hibernate 5.x（2015-2023）

- **JPA 2.1（5.0+）/ JPA 2.2（5.2+）支持**：完整实现存储过程调用（StoredProcedureQuery）、Entity Graph 控制加载策略、`@Repeatable` 注解、`AttributeConverter`、`Stream` 结果集等。
- **Hibernate Spatial**：集成地理空间数据类型（GIS），支持 MySQL、PostgreSQL（PostGIS）、Oracle Spatial。
- **Java 8 集成**：原生支持 `java.time`（LocalDate、LocalDateTime 等）类型映射，无需额外 `@Converter`。

```java
@Entity
public class Event {
    private LocalDateTime startTime;  // 自动映射
    private Duration duration;
}
```

- **Hibernate Reactive 萌芽**：5.3+ 开始实验性支持 Vert.x / Netty 驱动下的非阻塞持久化。
- **5.2 合并 EntityManager**：`hibernate-entitymanager` 模块合并入 `hibernate-core`，简化依赖。

## 2.4 Hibernate 6.x（2023+）

Hibernate 6 是近年来最大的一次技术革新：

- **Jakarta Persistence API（JPA 3.x）**：包名从 `javax.persistence.*` 迁移至 `jakarta.persistence.*`。

```java
// JPA 2.x (Hibernate 5.x)
import javax.persistence.Entity;
import javax.persistence.Id;

// JPA 3.x (Hibernate 6.x)
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
```

- **SQM（Semantic Query Model）**：HQL / JPQL 解析引擎完全重写，支持更丰富的 SQL 函数、类型推断、类型安全检查，性能显著提升。
- **类型系统重构**：引入 `@JavaType`、`@JdbcType` 新的映射注解体系，替代旧的 Type 系统。
- **性能大幅提升**：批量操作优化、更高效的反射替代（通过 ByteBuddy 或 Lambdas）、ZGC 友好的内存模型。
- **JDK 17+ 基线**：要求 JDK 17+，充分利用 Record、Pattern Matching、Sealed Classes 等新特性。支持将 Java Record 映射为实体（Embeddable）。

```java
@Embeddable
public record Address(String street, String city, String zipCode) {}
```

- **GenerationType.UUID 原生支持**：不再需要手动实现 IdentifierGenerator。
- **对 High-Performance 持久化的原生支持**：如 Upsert（INSERT ... ON CONFLICT）支持 Postgres、MySQL 等。

# 3. JPA 与 Hibernate 的关系

## 3.1 JPA 规范概述

**JPA（Jakarta Persistence API，原 Java Persistence API）** 是由 JCP（Java Community Process）制定的 Java 持久化标准规范。注意：JPA 本身只是一组接口和注解（`jakarta.persistence` 包），不包含任何实现代码。它定义了 Java 持久化的契约，具体的 SQL 生成、缓存、事务管理等工作由各实现提供。

**JPA 核心 API**：

| API | 说明 |
| --- | --- |
| **EntityManagerFactory** | 线程安全的工厂，用于创建 EntityManager，通常一个应用一个实例 |
| **EntityManager** | 持久化上下文的核心操作接口，管理实体生命周期 |
| **EntityTransaction** | 资源本地事务管理接口 |
| **PersistenceUnit** | 持久化单元配置，定义实体类和数据库连接 |
| **TypedQuery / Query** | JPQL 和 Criteria 查询的执行接口 |
| **Metamodel** | 持久化单元的元模型，用于类型安全查询 |

**核心规范文件 - persistence.xml**：

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
                 https://jakarta.ee/xml/ns/persistence/persistence_3_1.xsd"
             version="3.1">
    <persistence-unit name="myPU" transaction-type="RESOURCE_LOCAL">
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
        <class>com.example.User</class>
        <properties>
            <property name="jakarta.persistence.jdbc.url"
                      value="jdbc:mysql://localhost:3306/mydb" />
            <property name="jakarta.persistence.jdbc.user" value="root" />
            <property name="jakarta.persistence.jdbc.password" value="secret" />
        </properties>
    </persistence-unit>
</persistence>
```

## 3.2 Hibernate 与 JPA 的关系

**本质关系**：JPA 是规范（Specification），Hibernate 是 JPA 的参考实现（Implementation）之一。类比：JDBC 是规范，MySQL Connector/J 是实现；Servlet 是规范，Tomcat 是实现。

**Hibernate 的 JPA 扩展**：

除了实现 JPA 标准 API 外，Hibernate 还提供了大量超出 JPA 规范的能力：

| 能力 | JPA 标准支持 | Hibernate 扩展 |
| --- | :---: | :---: |
| HQL（增强查询） | 无 | 完整支持，远超 JPQL 语法 |
| 多租户 | 无 | 完整支持 |
| 过滤器（@Filter） | 无 | 支持动态条件过滤 |
| 批量操作优化 | 基本支持 | JDBC Batch、StatelessSession |
| Envers（数据审计） | 无 | 完整的数据版本管理 |
| 二级缓存控制 | @Cacheable | 更细粒度的缓存策略 |
| Native SQL 命名查询 | @NamedNativeQuery | 增强的结果映射 |
| 多态查询 | 有限支持 | 完整支持 |
| Entity Graph 扩展 | @NamedEntityGraph | FetchProfile 等补充 |

**演进历程**：

- **hibernate-core**：Hibernate 原生 API（Session、SessionFactory、Transaction），不依赖 JPA 规范。
- **hibernate-entitymanager**：在 hibernate-core 之上包装的 JPA 层（EntityManagerImpl 内部包装了 Session）。**Hibernate 5.2 起该模块合并入 hibernate-core**，用户无需单独引入。
- **Hibernate 6.x**：全面实现 Jakarta Persistence 3.x，原生 API 和 JPA API 深度整合。

**Native Hibernate 启动 vs JPA Mode 启动**：

JPA 方式（通过 persistence.xml）：
```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();
em.persist(user);
em.getTransaction().commit();
em.close();
```

Native Hibernate 方式（通过 hibernate.cfg.xml 或编程）：
```java
SessionFactory sf = new Configuration()
    .configure("hibernate.cfg.xml")
    .buildSessionFactory();
Session session = sf.openSession();
session.beginTransaction();
session.persist(user);
session.getTransaction().commit();
session.close();
```

**选择建议**：
- **优先使用 JPA API**：当项目需要保持 JPA 实现的可替换性时（例如未来可能切换到 EclipseLink）。
- **使用 Hibernate 扩展**：当需要使用 Hibernate 特有功能（Filter、Multi-tenancy、Envers、批量优化等）时。
- **实际最佳实践**：多数 Spring Boot 项目通过 JPA 接口编码，但利用 Hibernate 扩展能力——`EntityManager.unwrap(Session.class)` 即可获取底层 Session。

```java
// 在 JPA 环境中获取 Hibernate Session
EntityManager em = emf.createEntityManager();
Session session = em.unwrap(Session.class);
session.enableFilter("activeFilter").setParameter("status", "ACTIVE");
```

**Hibernate 对 JPA 的增强项一览**：

1. 更丰富的 ID 生成策略（`@GenericGenerator`，Hibernate 6 中为 `@IdGeneratorType`）
2. 多租户支持（`MultiTenantConnectionProvider`）
3. Envers 审计模块
4. Hibernate Validator（Bean Validation 集成，超出 JPA 的验证能力）
5. 更灵活的 Fetch 策略（Batch Fetching、Subselect Fetching）
6. StatelessSession（无状态会话，适合批量处理）
7. 自定义类型映射（`@Type`）和数据类型扩展
8. 乐观锁失败重试机制
9. 集成 Infinispan / Ehcache 等缓存框架的丰富配置

## 3.3 其他 JPA 实现对比

| 特性 | Hibernate | EclipseLink | OpenJPA | DataNucleus |
| --- | :---: | :---: | :---: | :---: |
| **创建者/维护者** | Red Hat | Eclipse Foundation | Apache | DataNucleus Team |
| **社区规模** | 最活跃 | 活跃 | 停滞 | 较小 |
| **Spring 集成** | 深度集成 | 兼容 | 兼容 | 兼容 |
| **扩展能力** | 非常丰富 | 中等 | 较少 | 丰富（支持 NoSQL） |
| **性能** | 优秀 | 良好 | 一般 | 良好 |
| **JPA 版本** | 3.1（Jakarta） | 4.0（Jakarta） | 停滞于 JPA 2.x | JDO + JPA |

- **EclipseLink**：TopLink 的开源后继者，曾为 JPA 2.x 参考实现。支持更丰富的映射策略（如 MOXy）。某些高级特性（如复合持久化单元、外部连接池集成）领先于 Hibernate。
- **OpenJPA**：Apache 项目，源自 IBM WebSphere 的持久化引擎。社区活跃度低，基本不再更新。
- **DataNucleus**：支持 JDO 和 JPA 双标准，特别之处在于同时支持 RDBMS 和 NoSQL 数据库（如 MongoDB、Cassandra、Neo4j），适合混合数据存储场景。

# 4. 核心概念

## 4.1 ORM 原理

**对象-关系阻抗不匹配（Object-Relational Impedance Mismatch）** 是 ORM 要解决的核心问题。以下是不匹配的五个主要维度：

### 4.1.1 粒度问题（Granularity）

Java 模型可以有细粒度的类型（如 Address 是一个 Embeddable 独立类型），但数据库中地址信息通常与用户同表。Hibernate 通过 `@Embeddable` / `@Embedded` 或 `@ElementCollection` 来解决：

```java
@Embeddable
public class Address {
    private String street;
    private String city;
}

@Entity
public class User {
    @Id private Long id;
    @Embedded
    private Address address; // 与 User 同表存储
}
```

### 4.1.2 继承问题（Subtypes / Inheritance）

Java 支持继承，关系数据库不支持。Hibernate 提供三种继承映射策略（单表、连接表、每个具体类一张表）。

### 4.1.3 同一性问题（Identity）

Java 中通过 `==` 判断对象同一性（内存地址），通过 `equals()` 判断等价性；数据库中通过主键值判断同一性。Hibernate 在持久化上下文中保证了同一个数据库行在同一个 Session 中只对应一个 Java 对象实例（Identity Map 模式）：

```java
User u1 = session.get(User.class, 1L);
User u2 = session.get(User.class, 1L);
System.out.println(u1 == u2); // true
```

### 4.1.4 关联问题（Associations）

Java 对象通过引用表示关联（`user.getOrders()` 返回集合引用），数据库通过外键表示关联。ORM 需要将二者相互转换，并处理级联操作、外键约束、关联方向、集合类型（Set vs List）等细节。

### 4.1.5 导航问题（Navigation）

Java 中通过对象图导航访问关联数据（`user.getOrders().get(0).getItems()` 链式调用），数据库中需要 JOIN 查询。Hibernate 的 Fetch 策略（Eager / Lazy / Batch / Subselect）决定了何时以何种方式加载关联数据。

## 4.2 核心术语

| 术语 | 说明 |
| --- | --- |
| **Entity（实体）** | 使用 `@Entity` 注解的 Java 类，对应数据库中的一张表，其实例代表表中的一行数据 |
| **Persistent Context（持久化上下文）** | 由 Session / EntityManager 管理的一组托管实体实例集合，是 Hibernate 的核心状态管理机制 |
| **Session / EntityManager** | 持久化操作的入口，管理实体生命周期，提供 CRUD 操作、查询和事务控制。Session 是 Hibernate 原生 API，EntityManager 是 JPA API |
| **SessionFactory / EntityManagerFactory** | 线程安全的重型工厂对象，创建代价高但内部状态不可变，整个应用通常只有一个实例 |
| **Transaction** | 逻辑工作单元，通过 `beginTransaction()` / `commit()` / `rollback()` 控制，底层可以是 JDBC 事务或 JTA 事务 |
| **Mapping Metadata（映射元数据）** | 描述实体类与表、属性与列之间映射关系的配置信息，可通过注解、XML 或编程方式定义 |
| **Detached / Managed / Transient 状态** | 实体的三种生命周期状态（详见 4.3 节） |
| **Flush（刷新）** | 将持久化上下文中的变更同步到数据库的过程。发生时机：事务提交前、执行查询前（FlushMode.AUTO）、显式调用 `flush()` |
| **Dirty Checking（脏检查）** | Hibernate 自动检测托管实体属性变更的机制。在 Flush 时比较实体的当前快照与初始快照，对变更的属性生成 UPDATE 语句 |

**脏检查原理**：

```java
User user = session.get(User.class, 1L);
// Hibernate 在加载时保存了一份实体状态的快照（snapshot）
user.setName("New Name");
// flush 时，Hibernate 对比当前状态与快照，发现 name 已变更，生成：
// UPDATE t_user SET name = 'New Name' WHERE id = 1
session.getTransaction().commit();
```

## 4.3 实体生命周期状态

实体在 Hibernate 中的生命周期分为四种状态（Six States in Extended Model）：

### 4.3.1 Transient（瞬时态）

- 通过 `new` 关键字创建、尚未与任何 Session 关联的实体。
- **无数据库对应行**，无持久化标识（ID 为 null 或默认值）。
- 状态变更不会被跟踪。
- 垃圾回收即消失，删除无痕迹。

```java
User user = new User();
user.setName("Alice");
// user 当前为 Transient 状态
```

### 4.3.2 Persistent / Managed（持久态 / 托管态）

- 已经通过 `persist()` / `save()` / `merge()` / `find()` 等方法与 Session 建立关联的实体。
- **有数据库对应行**（或即将有，由 ID 生成策略决定）。
- 任何属性变更都会被脏检查（Dirty Checking）捕获，并在 flush 时自动生成 UPDATE 语句同步到数据库。

```java
session.persist(user); // Transient → Managed
User loaded = session.find(User.class, 1L); // loaded 即为 Managed 状态
loaded.setName("Bob"); // 脏检查将追踪此修改
// transaction.commit() 时自动执行 UPDATE
```

### 4.3.3 Detached（游离态）

- Session 关闭后，原先的托管实体变为游离态。
- **有数据库对应行**，但不再受任何 Session 管理。
- 属性变更不会自动同步到数据库，需通过 `merge()` / `update()` 重新关联。

```java
session.close(); // session 关闭后，user 和 loaded 均为 Detached 状态
loaded.setEmail("bob@example.com"); // 不会被自动持久化
Session session2 = sf.openSession();
session2.merge(loaded); // Detached → Managed，变更同步
```

### 4.3.4 Removed（删除态）

- 通过 `remove()` / `delete()` 标记为待删除的实体。
- 在 flush 时生成 DELETE 语句。
- 删除后实体回归 Transient 状态。

```java
session.remove(user); // Managed → Removed
// flush 时生成 DELETE FROM t_user WHERE id = ?
```

### 4.3.5 状态转换图

```
    new / close
  ┌──────────────┐
  │  TRANSIENT   │ ←──────────────────┐
  └──────┬───────┘                    │
         │ persist() / save()         │
         ▼                            │
  ┌──────────────┐      remove()      │
  │   MANAGED    │ ──────────────────→│ REMOVED
  └──────┬───────┘                    │
         │ close() / clear() / evict()│
         ▼                            │
  ┌──────────────┐                    │
  │  DETACHED    │                    │
  └──────┬───────┘                    │
         │ merge() / update()         │
         └────────────────────────────┘
```

### 4.3.6 各状态下的行为特征

| 特征 | Transient | Managed | Detached | Removed |
| --- | :---: | :---: | :---: | :---: |
| 有无数据库对应行 | 无 | 有 | 有 | 有（待删） |
| Session 是否管理 | 否 | 是 | 否 | 是 |
| Dirty Checking 是否生效 | 否 | 是 | 否 | 否 |
| ID 是否存在 | 否 | 是 | 是 | 是 |
| `persist()` | → Managed | 异常 | 异常 | 异常 |
| `merge()` | → Managed (return) | → 返回 Managed | → Managed (return) | → 返回 Managed |
| `remove()` | 异常 | → Removed | 异常 | 无变化 |
| `refresh()` | 异常 | 从数据库刷新 | 异常 | 异常 |

# 5. 与其他 ORM 框架对比

## 5.1 MyBatis

**MyBatis** 是一款半自动化的 **SQL Mapping 框架**，定位与 Hibernate 有本质差异：

| 维度 | Hibernate | MyBatis |
| --- | --- | --- |
| **设计理念** | 全自动 ORM，面向对象 | 半自动化 SQL Mapping，面向 SQL |
| **SQL 控制** | 自动生成，开发中可手写覆盖 | 完全由开发者编写 SQL |
| **对象映射** | 通过注解/XML 自动映射，透明持久化 | 需要手动编写 ResultMap 映射规则 |
| **关联查询** | `@OneToMany` 等自动化处理 | 手动 Join 查询并组装 |
| **缓存** | 内置一级/二级缓存 | 两级缓存需配置但更轻量 |
| **SQL 优化空间** | 底层生成 SQL，调优需理解引擎行为 | 开发者可直接优化每条 SQL |
| **学习曲线** | 陡峭（需理解 ORM 原理） | 平缓（懂 SQL 即可上手） |
| **适用场景** | 对象模型复杂、CRUD 规范的应用 | SQL 密集、查询灵活、性能要求高的应用 |

**选型思路**：如果你的应用以 CRUD 为主、需要领域建模和对象导航，选 Hibernate；如果你的应用需要大量复杂查询和报表、SQL 高度定制化，选 MyBatis。很多项目中两者共存也很常见。

## 5.2 JOOQ

**JOOQ（Java Object Oriented Querying）** 是一种基于数据库逆向工程的 **类型安全 SQL 构建框架**：

- JOOQ 将数据库的表、字段、存储过程全部生成为 Java 类型，编写 SQL 时有编译期检查。
- 与 Hibernate Criteria API 相比，JOOQ 更接近原生 SQL 的思维方式，表达能力更强。
- 更适用于 **SQL 密集型应用**（报表、数据分析、复杂查询），而 Hibernate 更适合 **对象驱动** 的应用。

```java
// JOOQ 方式：类型安全的 SQL 构建
Result<Record> result = dsl.select()
    .from(T_USER)
    .where(T_USER.AGE.gt(18))
    .fetch();

// Hibernate Criteria 方式
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.select(root).where(cb.gt(root.get("age"), 18));
```

**适用场景**：
- JOOQ：SQL 第一思维、动态查询极为复杂、需要精确控制 SQL 生成的场景。
- Hibernate：对象模型驱动、CRUD 为主、需要高级持久化特性（缓存、事件、级联）的场景。

## 5.3 JDBC 与 Spring JDBC

| 维度 | JDBC | Spring JDBC (JdbcTemplate) | Hibernate |
| --- | :---: | :---: | :---: |
| **抽象层级** | 最低（Connection, Statement, ResultSet） | 低（JdbcTemplate 封装模板） | 高（对象模型） |
| **SQL 控制** | 完全手动 | 手动编写 SQL | 自动生成（可覆盖） |
| **结果映射** | 手动 ResultSet 遍历 | RowMapper / BeanPropertyRowMapper | 自动映射 |
| **资源管理** | 手动 try-finally | 框架管理 | 框架管理 |
| **开发效率** | 低 | 中 | 高 |
| **运行时性能** | 最高 | 高 | 中-高（取决于使用方式） |
| **代码量** | 极多 | 中等 | 少 |

**Hibernate 如何封装 JDBC**：

Hibernate 底层仍然使用 JDBC 与数据库通信。每次 Session 操作最终都会通过 `ConnectionProvider` 获取 JDBC 连接、构建 `PreparedStatement`、执行 SQL、处理 `ResultSet`。Hibernate 的责任是在 JDBC 之上增加映射层、缓存层、状态管理层和查询翻译层。

**开发效率与性能权衡**：使用 Hibernate 的开发效率显著高于裸 JDBC，但运行时有额外的开销（代理、脏检查、缓存管理）。对于大多数业务系统，这种开销可以接受；对于高性能要求的场景，可以在关键路径使用 Native SQL 或 JDBC 兜底。

## 5.4 选型建议

| 场景 | 推荐方案 |
| --- | --- |
| 标准企业应用（CRUD + 简单查询） | Hibernate / Spring Data JPA |
| 复杂查询、报表系统 | MyBatis 或 JOOQ |
| 混合场景（CRUD + 复杂查询） | Hibernate + MyBatis（或 JOOQ） 共存 |
| 极简单体应用、原型 | Spring JDBC (JdbcTemplate) |
| 高并发、对性能极度敏感 | JDBC 或 R2DBC（响应式） |

# 6. 适用场景与局限性

## 6.1 适用场景

- **OLTP 系统**：在线事务处理系统（如电商订单、用户管理、内容管理），以短事务和 CRUD 操作为主，Hibernate 的自动脏检查、缓存和乐观锁非常契合。
- **复杂对象模型**：实体之间存在大量关联（一对多、多对多）和继承关系，使用 Hibernate 可以显著降低手动处理 Join 的复杂度。
- **CRUD 密集型应用**：后台管理系统、表单驱动的应用，大部分数据操作是标准增删改查。
- **需要数据库移植性**：产品需要在不同客户的不同数据库上部署（MySQL、PostgreSQL、Oracle），Dialect 机制非常关键。

## 6.2 不适用场景

- **批量 ETL 数据处理**：数百万行的导入导出、数据迁移，Hibernate 的脏检查和管理开销过大。推荐使用 Spring Batch + JdbcTemplate 或数据库原生工具。
- **报表系统**：大量多表 Join、聚合查询、复杂统计，自动生成的 SQL 难以优化。推荐 JOOQ 或 MyBatis + 原生 SQL。
- **对延迟极度敏感的场景**：Hibernate 的初始化开销、代理创建、反射操作不适合亚毫秒级延迟要求的系统。
- **无领域模型的简单查询**：只做键值对查询的场景，Hibernate 过于笨重。

## 6.3 常见误解

### 6.3.1 "Hibernate 很慢"

**事实**：Hibernate 本身不会让应用变慢。变慢的原因通常是不当使用：

1. **N+1 查询问题**：Lazy 加载导致的循环查询。解决方案：`JOIN FETCH`、`@BatchSize`、`@Fetch(FetchMode.SUBSELECT)`、Entity Graph。
2. **不合理的 Fetch 策略**：Eager 加载过多不必要的数据，或 Lazy 导致频繁单条查询。
3. **未使用批量操作**：批量插入/更新未开启 JDBC Batch。
4. **未合理使用缓存**：频繁查询相同数据而未配置二级缓存或查询缓存。

一个经过良好调优的 Hibernate 应用，性能可以与手写 JDBC 相差在 5-10% 以内，而开发效率提升数倍。

### 6.3.2 "Hibernate 不适合大项目"

**事实**：全球众多大型企业在关键系统中使用 Hibernate（Red Hat、Atlassian、Liferay 等）。问题不在框架规模适用性，而在于团队是否理解 ORM 原理并遵循最佳实践。

- 核心策略：分层架构明确（Service 层不泄露 Session）、合理的事务边界、严格的 Fetch 策略控制、禁用自动 DDL。
- 对于非常复杂的查询，可以在 Hibernate 项目中使用 Native SQL 或 JOOQ 作为补充，而非全盘否定。

### 6.3.3 "使用 Hibernate 不需要懂 SQL"

**事实**：这是错误的。使用 Hibernate 更需要深入理解 SQL，因为：

1. 需要审查和优化 Hibernate 生成的 SQL（通过日志输出）。
2. N+1 问题、笛卡尔积问题需要 SQL 知识来诊断。
3. Native SQL 是兜底手段，复杂查询最终还是要手写 SQL。
4. 索引设计、查询计划分析等 DBA 工作同样需要 SQL 功底。

**正确认知**：Hibernate 让你从 80% 的重复 SQL 编写中解放出来，但剩下的 20% 复杂场景仍需要 SQL 技能。

# 7. 环境搭建快速入门

## 7.1 Maven 依赖

**Hibernate 5.x**：

```xml
<dependencies>
    <!-- Hibernate Core (已包含 JPA 支持) -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>5.6.15.Final</version>
    </dependency>

    <!-- MySQL 驱动 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.0.33</version>
    </dependency>

    <!-- 连接池 (HikariCP) -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.0.1</version>
    </dependency>
</dependencies>
```

**Hibernate 6.x（Jakarta 命名空间）**：

```xml
<dependencies>
    <dependency>
        <groupId>org.hibernate.orm</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>6.4.4.Final</version>
    </dependency>

    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.0.33</version>
    </dependency>

    <!-- HikariCP 连接池 -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.0.1</version>
    </dependency>
</dependencies>
```

**注意**：Hibernate 5.2+ 及 6.x 中 `hibernate-entitymanager` 已合并到 `hibernate-core`，不再需要单独引入。如果使用 Spring Boot，直接引入 `spring-boot-starter-data-jpa` 即可，它会自动引入合适的 Hibernate 版本。

## 7.2 第一个 Hibernate 程序

### 7.2.1 实体类编写

```java
package com.example.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "t_user")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", length = 50, nullable = false)
    private String name;

    @Column(name = "email", unique = true)
    private String email;

    private Integer age;

    // 必须提供无参构造器
    public User() {}

    public User(String name, String email, Integer age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }

    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', email='" + email + "', age=" + age + "}";
    }
}
```

### 7.2.2 Hibernate 配置 - hibernate.cfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <!-- 数据库连接 -->
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">
            jdbc:mysql://localhost:3306/hibernate_demo?useSSL=false&amp;serverTimezone=Asia/Shanghai
        </property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">your_password</property>

        <!-- 连接池配置 (Hibernate 内置连接池，生产环境应使用 HikariCP) -->
        <property name="hibernate.connection.pool_size">10</property>

        <!-- SQL 方言 -->
        <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>

        <!-- 自动建表策略：create | update | validate | create-drop -->
        <property name="hibernate.hbm2ddl.auto">update</property>

        <!-- 打印 SQL（开发调试用） -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>

        <!-- 映射实体类 -->
        <mapping class="com.example.entity.User" />
    </session-factory>
</hibernate-configuration>
```

### 7.2.3 创建 SessionFactory（工具类）

```java
package com.example.util;

import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

public class HibernateUtil {

    private static final SessionFactory sessionFactory = buildSessionFactory();

    private static SessionFactory buildSessionFactory() {
        try {
            return new Configuration()
                .configure("hibernate.cfg.xml")
                .buildSessionFactory();
        } catch (Throwable ex) {
            System.err.println("SessionFactory 创建失败: " + ex);
            throw new ExceptionInInitializerError(ex);
        }
    }

    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }

    public static void shutdown() {
        getSessionFactory().close();
    }
}
```

### 7.2.4 增删改查完整示例

```java
package com.example;

import com.example.entity.User;
import com.example.util.HibernateUtil;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.Transaction;

import java.util.List;

public class Application {

    public static void main(String[] args) {
        SessionFactory sf = HibernateUtil.getSessionFactory();

        // ====== 新增 (Create) ======
        Session session = sf.openSession();
        Transaction tx = session.beginTransaction();

        User user1 = new User("张三", "zhangsan@example.com", 25);
        User user2 = new User("李四", "lisi@example.com", 30);
        session.persist(user1);
        session.persist(user2);

        tx.commit();
        session.close();

        // ====== 查询 (Read) ======
        session = sf.openSession();
        tx = session.beginTransaction();

        // 按主键查询
        User found = session.get(User.class, user1.getId());
        System.out.println("按ID查询: " + found);

        // HQL 查询
        List<User> users = session.createQuery(
            "from User u where u.age > :age", User.class)
            .setParameter("age", 20)
            .getResultList();
        System.out.println("年龄大于20的用户数量: " + users.size());

        tx.commit();
        session.close();

        // ====== 更新 (Update) ======
        session = sf.openSession();
        tx = session.beginTransaction();

        User toUpdate = session.get(User.class, user1.getId());
        toUpdate.setAge(26); // 脏检查自动生成 UPDATE
        toUpdate.setName("张三丰");

        tx.commit(); // flush 发生时 UPDATE 语句发送到数据库
        session.close();

        // ====== 删除 (Delete) ======
        session = sf.openSession();
        tx = session.beginTransaction();

        User toDelete = session.get(User.class, user2.getId());
        session.remove(toDelete);

        tx.commit();
        session.close();

        // ====== 验证 ======
        session = sf.openSession();
        List<User> allUsers = session.createQuery("from User", User.class).getResultList();
        allUsers.forEach(System.out::println);
        session.close();

        // 关闭资源
        HibernateUtil.shutdown();
    }
}
```

**运行输出预期**（SQL 日志）：

```sql
-- 插入
INSERT INTO t_user (name, email, age) VALUES ('张三', 'zhangsan@example.com', 25)
INSERT INTO t_user (name, email, age) VALUES ('李四', 'lisi@example.com', 30)

-- 查询
SELECT * FROM t_user WHERE id = 1
SELECT * FROM t_user WHERE age > 20

-- 更新（脏检查触发）
UPDATE t_user SET name = '张三丰', age = 26 WHERE id = 1

-- 删除
DELETE FROM t_user WHERE id = 2

-- 最终查询
SELECT * FROM t_user
```

## 7.3 日志配置

Hibernate 使用 JBoss Logging 作为底层日志门面，可以桥接到 SLF4J 再输出到 Logback、Log4j2 等日志框架。

### 7.3.1 核心日志分类

| 日志类别 | 说明 |
| --- | --- |
| `org.hibernate.SQL` | 输出生成的 SQL 语句（对应 `hibernate.show_sql`） |
| `org.hibernate.type.descriptor.sql.BasicBinder` | 输出 SQL 参数绑定值（Trace 级别） |
| `org.hibernate.type.descriptor.sql.BasicExtractor` | 输出 ResultSet 提取值（Trace 级别） |
| `org.hibernate.stat` | 输出统计信息（需开启 `hibernate.generate_statistics`） |
| `org.hibernate.cache` | 输出缓存命中/未命中日志 |
| `org.hibernate.engine.transaction` | 事务相关的调试信息 |

### 7.3.2 Logback 完整配置示例

**pom.xml 中引入依赖**：

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.12</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.5.3</version>
</dependency>
```

**resources/logback.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Hibernate 特定日志配置 -->
    <!-- 输出 SQL 语句 -->
    <logger name="org.hibernate.SQL" level="DEBUG" />

    <!-- 输出 SQL 参数绑定值（如：binding parameter [1] as [VARCHAR] - [zhangsan]） -->
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE" />

    <!-- 可选：输出 JDBC 结果集提取值 -->
    <logger name="org.hibernate.type.descriptor.sql.BasicExtractor" level="TRACE" />

    <!-- 统计信息 -->
    <logger name="org.hibernate.stat" level="DEBUG" />

    <!-- 二级缓存日志 -->
    <logger name="org.hibernate.cache" level="DEBUG" />

    <!-- 事务调试 -->
    <logger name="org.hibernate.engine.transaction" level="DEBUG" />

    <!-- 根日志级别 -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

### 7.3.3 Hibernate 配置中开启 SQL 格式化与统计

在 `hibernate.cfg.xml` 中补充：

```xml
<!-- SQL 格式化（多行缩进） -->
<property name="hibernate.format_sql">true</property>

<!-- 高亮注释（标识 Hibernate 生成的 SQL） -->
<property name="hibernate.use_sql_comments">true</property>

<!-- 开启统计信息收集 -->
<property name="hibernate.generate_statistics">true</property>
```

### 7.3.4 日志输出示例

```
16:23:45.123 [main] DEBUG org.hibernate.SQL -
    /* insert com.example.entity.User */
    insert
        into
            t_user
            (name, email, age)
        values
            (?, ?, ?)
16:23:45.126 [main] TRACE o.h.type.descriptor.sql.BasicBinder -
    binding parameter [1] as [VARCHAR] - [张三]
16:23:45.126 [main] TRACE o.h.type.descriptor.sql.BasicBinder -
    binding parameter [2] as [VARCHAR] - [zhangsan@example.com]
16:23:45.127 [main] TRACE o.h.type.descriptor.sql.BasicBinder -
    binding parameter [3] as [INTEGER] - [25]
```

### 7.3.5 生产环境日志建议

- **开发环境**：开启 `show_sql`、`format_sql`、BasicBinder TRACE，方便调试。
- **测试环境**：开启统计信息（`generate_statistics`），分析 N+1 和缓存命中率。
- **生产环境**：关闭 SQL 日志输出（避免 I/O 开销和敏感数据泄露），仅在排查问题时临时开启。建议关闭 `hibernate.show_sql`，通过日志框架的级别控制代替。

```xml
<!-- 生产环境 hibernate.cfg.xml -->
<property name="hibernate.show_sql">false</property>
<property name="hibernate.format_sql">false</property>
<property name="hibernate.generate_statistics">false</property>
```

### 7.3.6 使用 DataSource 方式替代自建连接（推荐）

生产环境推荐使用连接池（如 HikariCP），并通过 `hibernate.connection.datasource` 指定 JNDI 名称或直接注入 DataSource：

```xml
<!-- HikariCP 连接池（Hibernate 原生支持） -->
<property name="hibernate.connection.provider_class">
    com.zaxxer.hikari.hibernate.HikariConnectionProvider
</property>
<property name="hibernate.hikari.dataSourceClassName">
    com.mysql.cj.jdbc.MysqlDataSource
</property>
<property name="hibernate.hikari.dataSource.url">
    jdbc:mysql://localhost:3306/hibernate_demo
</property>
<property name="hibernate.hikari.dataSource.user">root</property>
<property name="hibernate.hikari.dataSource.password">your_password</property>
<property name="hibernate.hikari.maximumPoolSize">20</property>
<property name="hibernate.hikari.minimumIdle">5</property>
```

或者在代码中以编程方式配置：

```java
public static SessionFactory buildSessionFactoryWithHikariCP() {
    HikariConfig hikariConfig = new HikariConfig();
    hikariConfig.setJdbcUrl("jdbc:mysql://localhost:3306/hibernate_demo");
    hikariConfig.setUsername("root");
    hikariConfig.setPassword("your_password");
    hikariConfig.setMaximumPoolSize(20);

    HikariDataSource dataSource = new HikariDataSource(hikariConfig);

    SessionFactory sessionFactory = new Configuration()
        .configure()
        .getStandardServiceRegistryBuilder()
        .applySetting("hibernate.connection.datasource", dataSource)
        .build();

    return sessionFactory;
}
```
