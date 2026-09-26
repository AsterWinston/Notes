# 1. 基础概念面试题

## 1.1 什么是 ORM？Hibernate 是如何实现 ORM 的？

**Q: 什么是 ORM？Hibernate 是如何实现 ORM 的？**

ORM（Object-Relational Mapping，对象关系映射）是一种将面向对象编程中的对象与关系数据库中的表进行映射的技术。它解决了面向对象模型与关系模型之间的"阻抗不匹配"问题（如继承、多态在关系表中无直接对应）。

Hibernate 实现 ORM 的核心机制：
- **元数据映射**：通过 XML（hbm.xml）或注解（如 `@Entity`、`@Table`、`@Column`）定义对象属性与表字段的映射关系。
- **持久化上下文**：Session/EntityManager 维护一个持久化上下文（一级缓存），管理实体对象的生命周期。
- **脏检查（Dirty Checking）**：flush 时自动比较快照与当前状态，生成差异 SQL。
- **方言（Dialect）**：抽象不同数据库的 SQL 差异，提供统一的 API。
- **关联管理**：通过外键/中间表自动管理与级联对象之间的关系。

```java
@Entity
@Table(name = "t_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_name", length = 50, nullable = false)
    private String name;

    // getter/setter...
}
```

## 1.2 JPA 和 Hibernate 是什么关系？

**Q: JPA 和 Hibernate 是什么关系？**

- **JPA（Java Persistence API）**：是 Java EE/Jakarta EE 提供的持久化规范，是一套接口和注解标准（如 `javax.persistence.*` / `jakarta.persistence.*`），本身不提供实现。
- **Hibernate**：是一个完整的 ORM 框架，同时也是 JPA 规范最流行的实现。除了实现 JPA 规范外，Hibernate 还提供了许多 JPA 之外的高级特性（如 HQL 多态查询、Criteria API Hibernate 扩展、Filter、Envers 审计等）。
- **类比**：JPA 是 JDBC 接口，Hibernate 是 MySQL 驱动。

```java
// 纯 JPA 方式
EntityManager em = entityManagerFactory.createEntityManager();

// Hibernate 特有方式
Session session = entityManager.unwrap(Session.class);
session.enableFilter("activeFilter");
```

## 1.3 Hibernate 与 MyBatis 的区别？

**Q: Hibernate 与 MyBatis 的区别？**

| 维度 | Hibernate | MyBatis |
|------|-----------|---------|
| ORM 程度 | 全自动 ORM | 半自动（SQL 映射器） |
| SQL 控制 | 自动生成，控制力弱 | 手动编写，完全可控 |
| 移植性 | 强（方言机制） | 弱（SQL 绑定数据库） |
| 缓存 | 内置一/二级缓存 | 需第三方缓存 |
| 对象管理 | 有状态（持久化上下文） | 无状态 |
| 学习曲线 | 陡峭 | 平缓 |
| 适用场景 | 对象模型复杂、CRUD 为主 | SQL 复杂、报表、性能要求高 |
| 延迟加载 | 自动支持 | 需手动配置 |

**选择建议**：如果系统以对象模型驱动、关系复杂，选 Hibernate；如果 SQL 需要高度优化、涉及复杂多表查询和存储过程，选 MyBatis。

## 1.4 实体有哪些生命周期状态？各状态如何转换？

**Q: 实体有哪些生命周期状态？各状态如何转换？**

Hibernate 定义的四种生命周期状态：

- **Transient（瞬时态）**：刚 new 出来，未与 Session 关联，数据库无对应行。
- **Persistent / Managed（持久态）**：与 Session 关联，有标识符，受脏检查管理。
- **Detached（游离态）**：曾经是持久态，但 Session 已关闭，数据库有对应行，但不受管理。
- **Removed（删除态）**：调用了 `remove()` 但尚未 flush，将在提交时删除。

```java
User user = new User("张三");         // transient
session.persist(user);               // → persistent（managed）
Long id = user.getId();
session.close();                     // → detached

Session session2 = sf.openSession();
session2.beginTransaction();
User detachedUser = session2.get(User.class, id);  // persistent
session2.remove(detachedUser);       // → removed
session2.getTransaction().commit();  // DELETE 执行
session2.close();                    // → transient（对象已无意义）
```

状态转换图：

```
        persist()          close()/clear()/evict()
Transient ─────────► Persistent ─────────────────────► Detached
    ▲                    │  ▲                               │
    │                    │  │ merge()                       │
    │   remove()         │  └───────────────────────────────┘
    │   (flush 后)       │
    └──── Removed ◄──────┘
              (flush 前)
```

## 1.5 Session 和 EntityManager 是什么关系？

**Q: Session 和 EntityManager 是什么关系？**

- `Session` 是 Hibernate 原生 API，`EntityManager` 是 JPA 标准 API。
- 在 Hibernate 实现中，`EntityManager` 内部包装了 `Session`：
  ```java
  EntityManager em = emf.createEntityManager();
  Session session = em.unwrap(Session.class);  // 获取底层 Session
  ```
- 功能基本等价，但 Session 提供更多 Hibernate 特有功能（如 `createSQLQuery()`、`enableFilter()`）。
- 推荐：项目中使用 JPA 标准接口（`EntityManager`），特定场景通过 `unwrap()` 获取 Session。

## 1.6 SessionFactory 和 EntityManagerFactory 是什么？线程安全吗？

**Q: SessionFactory 和 EntityManagerFactory 是什么？线程安全吗？**

- **SessionFactory**（Hibernate 原生）：负责创建 `Session` 实例，持有所有映射元数据和缓存配置。创建成本极高（解析映射、构建元模型），一个应用通常只有一个实例。
- **EntityManagerFactory**（JPA 标准）：等价于 SessionFactory，Spring 中通常通过 `LocalContainerEntityManagerFactoryBean` 创建。
- **线程安全**：两者都是**线程安全**的，多个线程可以并发访问。
- **Session / EntityManager** 是**线程不安全的**，每个线程必须使用自己的实例。

```java
// SessionFactory = 线程安全（单例）
// Session = 线程不安全（每次请求新实例）
SessionFactory sf = new Configuration().configure().buildSessionFactory();
Session session = sf.openSession();  // 每个操作创建新 Session
```

## 1.7 openSession() 和 getCurrentSession() 的区别？

**Q: openSession() 和 getCurrentSession() 的区别？**

| 维度 | openSession() | getCurrentSession() |
|------|--------------|---------------------|
| 新建 | 每次调用都创建新 Session | 当前线程没有时才创建 |
| 关闭 | 需手动调用 close() | 事务提交/回滚时自动关闭 |
| 上下文绑定 | 无上下文绑定 | 绑定到当前事务上下文 |
| 配置要求 | 无 | 需配置 `<property name="current_session_context_class">thread</property>` |
| 适用场景 | 需要手动控制生命周期 | 与容器/Spring 事务管理配合 |

```java
// Spring 环境下通常使用 getCurrentSession()
@Autowired
private SessionFactory sessionFactory;

@Transactional
public void doSomething() {
    Session session = sessionFactory.getCurrentSession();  // 自动绑定
    // 不需要手动 close，事务提交后自动 close
}
```

## 1.8 persist() 和 save() 的区别？

**Q: persist() 和 save() 的区别？**

| 维度 | persist() | save() |
|------|-----------|--------|
| 来源 | JPA 标准 | Hibernate 原生 |
| 返回值 | void（无返回值） | Serializable（返回生成的 ID） |
| 执行时机 | 可能延迟到 flush | 立即执行 INSERT 并返回 ID |
| 事务外行为 | 不允许（抛出异常） | 允许（但需要手动回滚） |
| 对游离态 | 抛出 EntityExistsException | 重新分配 ID 或报错 |

```java
// persist() — JPA 风格
em.persist(user);  // 不返回 ID，可能在 flush 时才生成 ID（IDENTITY 除外）

// save() — Hibernate 风格
Long id = (Long) session.save(user);  // 立即 INSERT，返回生成的主键
```

**推荐使用 `persist()`**（JPA 标准），结合 `GenerationType.SEQUENCE` 或 `GenerationType.TABLE` 可以实现批量插入优化。

## 1.9 merge() 和 update() 的区别？

**Q: merge() 和 update() 的区别？**

| 维度 | merge() | update() |
|------|---------|----------|
| 来源 | JPA 标准 | Hibernate 原生 |
| 语义 | 合并：将 detached 对象状态复制到 managed 对象 | 将 detached 对象重新附加（attach） |
| 同一 ID 冲突 | 无问题，返回 managed 对象 | 抛 NonUniqueObjectException |
| 返回值 | 返回 managed 对象副本 | 无返回值 |
| 参数对象 | 原对象保持 detached | 原对象变为 managed |

```java
// merge() 示例
User detachedUser = new User();
detachedUser.setId(1L);
detachedUser.setName("新名字");
User managedUser = em.merge(detachedUser);  // detachedUser 仍是 detached
// managedUser 是 managed，后续修改会被跟踪

// update() 示例
session.update(detachedUser);  // detachedUser 直接变成 managed
detachedUser.setName("再改");   // 会被脏检查捕获
```

**最佳实践**：始终使用 `merge()`，因为它在存在同名 managed 实例时不会报错。

## 1.10 get() 和 load() 的区别？

**Q: get() 和 load() 的区别？**

| 维度 | get() | load() |
|------|-------|--------|
| 加载时机 | 立即查询数据库 | 返回代理对象（延迟加载） |
| 数据不存在 | 返回 null | 访问非 ID 属性时抛 ObjectNotFoundException |
| 性能 | 必然执行 SELECT | 可能不执行 SQL（只用到 ID） |
| 代理 | 直接返回实体或 null | 返回 Hibernate 代理对象 |

```java
// get() — 立即查询
User user = session.get(User.class, 1L);  // SELECT 立即执行
if (user == null) {
    // 不存在
}

// load() — 延迟加载
User user = session.load(User.class, 9999L);  // 无 SELECT，返回代理
// user.getId() 可用（不触发加载）
String name = user.getName();  // 触发 SELECT，若不存在则抛异常

// load() 的使用场景 — 设置关联，仅需 ID
Order order = new Order();
order.setUser(session.load(User.class, userId));  // 不查询 User 表
session.persist(order);  // 仅用 userId 设置外键
```

## 1.11 detach() 和 evict() 的区别？

**Q: detach() 和 evict() 的区别？**

| 维度 | detach() | evict() |
|------|----------|---------|
| 来源 | JPA 标准（EntityManager） | Hibernate 原生（Session） |
| 功能 | 将实体从持久化上下文移除 | 将实体从持久化上下文移除 |
| 本质 | 相同操作，不同 API 名称 | 相同操作，不同 API 名称 |
| 级联 | 支持级联（cascade = DETACH） | 不自动级联（除非配置） |

```java
// 二者本质等价
session.evict(user);   // Hibernate 方式
em.detach(user);       // JPA 方式

// persist → detach → merge 的典型场景
em.persist(user);
em.detach(user);       // user 变为 detached
user.setName("修改");  // 不会被跟踪
User m = em.merge(user); // 合并回数据库中
```

## 1.12 flush() 的作用是什么？FlushMode 有哪些？

**Q: flush() 的作用是什么？FlushMode 有哪些？**

**flush() 作用**：将持久化上下文中的所有挂起 SQL（INSERT、UPDATE、DELETE）同步到数据库，但不提交事务。flush 后可以回滚。

**FlushMode 类型**：
| 模式 | 说明 |
|------|------|
| AUTO（默认） | 查询前自动 flush，确保查询结果包含当前上下文变更 |
| COMMIT | 仅在事务提交前 flush，查询不触发 flush |
| ALWAYS | 每次查询前都 flush |
| MANUAL | 仅当显式调用 `flush()` 时执行 |

```java
session.setHibernateFlushMode(FlushMode.COMMIT);

// flush() 的顺序（重要！）
// 1. EntityInsertAction → 2. EntityUpdateAction → 3. CollectionRemoveAction
// 4. CollectionUpdateAction → 5. CollectionRecreateAction → 6. EntityDeleteAction

session.flush();  // 手动同步，但事务未提交
session.getTransaction().commit();  // 提交事务
```

**flush 的自动触发时机**：
- 事务提交前（AUTO/COMMIT 模式）
- 执行查询前（AUTO/ALWAYS 模式，若挂起操作可能影响查询结果）
- 显式调用 `flush()`

## 1.13 什么是持久化上下文（Persistence Context）？

**Q: 什么是持久化上下文（Persistence Context）？它的核心职责是什么？**

持久化上下文（Persistence Context）是 Hibernate 的核心概念，它是 Session/EntityManager 内部维护的一组 managed 实体实例的集合，类似于一个"对象缓存 + 脏数据追踪器"。

**核心职责**：

1. **同一性保证**：在同一个 Session 内，同一个数据库行对应的实体对象引用唯一。
   ```java
   User u1 = session.get(User.class, 1L);
   User u2 = session.get(User.class, 1L);
   System.out.println(u1 == u2);  // true
   ```

2. **脏检查（Dirty Checking）**：flush 时自动对比实体当前状态与加载时的快照，生成 UPDATE 语句。
   ```java
   User u = session.get(User.class, 1L);
   u.setName("新名字");  // 无需调用 update()，自动检测
   session.flush();       // 自动生成 UPDATE
   ```

3. **写操作延迟**：INSERT、UPDATE、DELETE 不一定立即执行，而是排队到 flush 时批量执行。

4. **一级缓存**：重复查询时避免不必要的 SQL。

5. **循环引用处理**：通过身份保证机制，在对象图中正确处理循环引用。

# 2. 映射相关面试题

## 2.1 @Entity 实体类有哪些要求？

**Q: @Entity 实体类有哪些要求？**

根据 JPA 规范，实体类必须满足：

1. 类上标注 `@Entity` 注解，必须有**无参构造器**（public 或 protected）。
2. 必须有一个主键字段，使用 `@Id` 标注。
3. 不能是 `final` 类（影响代理生成）。
4. 持久化字段不能是 `final`（Hibernate 6 中放宽部分限制）。
5. 需要实现 `Serializable`（建议但不强制，尤其在集群/二级缓存场景）。
6. 枚举、集合等类型需有对应映射注解。

```java
@Entity
@Table(name = "t_product")
public class Product implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    protected Product() {}  // JPA 要求的无参构造器

    public Product(String name) {
        this.name = name;
    }
    // getter/setter...
}
```

## 2.2 字段访问和属性访问有什么区别？

**Q: 字段访问和属性访问有什么区别？**

- **字段访问（Field Access）**：`@Id` 注解放在字段上，Hibernate 通过反射直接访问字段，不经过 getter/setter。
- **属性访问（Property Access）**：`@Id` 注解放在 getter 上，Hibernate 通过 getter/setter 访问，可以在 getter/setter 中添加逻辑。

```java
// 字段访问（推荐）
@Entity
public class User {
    @Id private Long id;  // @Id 在字段上 → 字段访问
}

// 属性访问
@Entity
public class User {
    private Long id;

    @Id  // @Id 在 getter 上 → 属性访问
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
}

// 混合访问（不推荐，易混乱）
@Entity
public class User {
    @Id private Long id;  // 字段访问

    @Access(AccessType.PROPERTY)  // 单独指定属性访问
    @Column
    private String name;
    public String getName() { return "Mr. " + name; }
}
```

**推荐使用字段访问**，因为：
- 更直观，避免 getter/setter 副作用
- 序列化框架（如 Jackson）通常访问字段或 getter，减少冲突
- 代理对象可能影响 getter 行为

## 2.3 主键生成策略有哪些？IDENTITY 和 SEQUENCE 的区别？

**Q: 主键生成策略有哪些？IDENTITY 和 SEQUENCE 的区别？**

JPA 主键生成策略（`GenerationType`）：

| 策略 | 说明 | 适用数据库 |
|------|------|-----------|
| AUTO | 由 Hibernate 根据方言自动选择 | 通用 |
| IDENTITY | 依赖数据库自增列（AUTO_INCREMENT） | MySQL、SQL Server |
| SEQUENCE | 使用数据库序列 | PostgreSQL、Oracle |
| TABLE | 用单独的表模拟序列 | 所有（性能差） |
| UUID | Hibernate 6 新增，生成 UUID | 通用 |

**IDENTITY vs SEQUENCE 关键区别**：

```java
// IDENTITY — INSERT 后才知道 ID，禁用 JDBC 批处理
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;  // 执行 INSERT 后从数据库获取
}

// SEQUENCE — 预分配 ID，支持 JDBC 批处理
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(name = "user_seq", sequenceName = "user_seq", allocationSize = 50)
    private Long id;  // 一次从序列获取 50 个 ID，减少数据库往返
}
```

| 维度 | IDENTITY | SEQUENCE |
|------|----------|----------|
| ID 获取时机 | INSERT 执行后 | INSERT 执行前（内存预分配） |
| JDBC 批处理 | 禁用（需立即知道 ID） | 支持 |
| 数据库支持 | MySQL 等 | PostgreSQL、Oracle 等 |
| 性能 | 批量插入性能差 | 批量插入性能好 |
| 事务回滚 | ID 不会被回滚 | ID 不会被回滚（序列不回滚） |

## 2.4 allocationSize 的作用？

**Q: allocationSize 的作用？**

`allocationSize` 是 `@SequenceGenerator` 的重要参数，控制每次从数据库序列获取的 ID 数量。它实现了**池化优化（Hi/Lo 算法）**，减少数据库往返。

```java
// allocationSize = 1：每次 INSERT 都要访问序列
@GeneratedValue(strategy = GenerationType.SEQUENCE)
// 10 万条数据 → 10 万次序列访问

// allocationSize = 50：一次获取 50 个 ID 缓存
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_seq", allocationSize = 50)
// 10 万条数据 → 2000 次序列访问
```

**关键规则**：
- `allocationSize` 必须与数据库序列的 `INCREMENT BY` 一致。
- 服务器重启时，未使用的 ID 会丢失（即产生 ID 空洞），但 ID 仍保持唯一递增。
- 新版 Hibernate 默认 `allocationSize = 50`（JPA 规范默认为 1，但 Hibernate 优化了默认值）。

```sql
-- 数据库序列必须与 allocationSize 对齐
CREATE SEQUENCE user_seq START WITH 1 INCREMENT BY 50;
```

## 2.5 复合主键如何映射？@IdClass vs @EmbeddedId 的区别？

**Q: 复合主键如何映射？@IdClass vs @EmbeddedId 的区别？**

**@IdClass**：主键字段保留在实体类中，通过独立的主键类映射。

```java
public class OrderItemPK implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() + hashCode() + 无参构造器
}

@Entity
@IdClass(OrderItemPK.class)
public class OrderItem {
    @Id private Long orderId;
    @Id private Long productId;
    private Integer quantity;
}
```

**@EmbeddedId**：主键字段封装在嵌入式类中。

```java
@Embeddable
public class OrderItemPK implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() + hashCode() + 无参构造器
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemPK id;
    private Integer quantity;
}
```

| 维度 | @IdClass | @EmbeddedId |
|------|----------|-------------|
| JPQL 访问 | `em.find(OrderItem.class, new OrderItemPK(1L, 2L))` | `em.find(OrderItem.class, new OrderItemPK(1L, 2L))` |
| JPQL 引用 | 直接写 `o.orderId` | 需写 `o.id.orderId` |
| 使用方便 | JPQL 更方便 | 对象模型更内聚 |
| 适用场景 | 主键字段需要单独查询 | 主键作为整体传递 |

**推荐 @EmbeddedId**，对象模型更清晰。

## 2.6 @Embeddable 和 @MappedSuperclass 的区别？

**Q: @Embeddable 和 @MappedSuperclass 的区别？**

| 维度 | @Embeddable | @MappedSuperclass |
|------|-------------|-------------------|
| 本质 | 值对象（Value Object） | 实体父类（不可独立持久化） |
| 生命周期 | 绑定到所属实体，无独立 ID | 字段被子类继承映射 |
| 数据库表现 | 字段嵌入同一张表 | 字段分别映射到各子类表 |
| 共享性 | 不可跨实体共享（每次嵌入新实例） | 子类继承字段定义 |

```java
// @Embeddable — 值对象
@Embeddable
public class Address {
    private String city;
    private String street;
}

@Entity
public class User {
    @Embedded
    private Address homeAddress;  // 字段嵌入 t_user 表
}

// @MappedSuperclass — 公共字段继承
@MappedSuperclass
public abstract class BaseEntity {
    @Id @GeneratedValue
    private Long id;

    @CreationTimestamp
    private Instant createdAt;
}

@Entity
public class User extends BaseEntity {
    private String name;  // 继承 id、createdAt 字段
}
```

**选择**：
- 值对象（本身不需要独立标识）→ `@Embeddable`
- 实体共享公共字段和映射 → `@MappedSuperclass`

## 2.7 怎样映射枚举类型？ORDINAL vs STRING？

**Q: 怎样映射枚举类型？ORDINAL vs STRING？**

使用 `@Enumerated` 注解映射枚举：

```java
public enum Status {
    ACTIVE,     // ordinal = 0
    INACTIVE,   // ordinal = 1
    DELETED     // ordinal = 2
}

@Entity
public class User {
    @Enumerated(EnumType.ORDINAL)  // 存 0, 1, 2（默认）
    private Status status1;

    @Enumerated(EnumType.STRING)   // 存 "ACTIVE", "INACTIVE", "DELETED"
    private Status status2;
}
```

| 维度 | ORDINAL | STRING |
|------|---------|--------|
| 存储值 | 序号（整数） | 枚举名称（字符串） |
| 性能 | 稍好（整数比较） | 稍差（字符串比较） |
| 枚举顺序变更 | **数据损坏**（严重 Bug） | 无影响 |
| 可读性 | 差 | 好 |

**强烈推荐使用 `EnumType.STRING`**。ORDINAL 在枚举中间插入新值时会导致已有数据映射错误。

```java
// 进阶：使用 AttributeConverter 自定义映射
@Converter(autoApply = true)
public class StatusConverter implements AttributeConverter<Status, Integer> {
    @Override
    public Integer convertToDatabaseColumn(Status status) {
        return status.getCode();  // 自定义编码
    }

    @Override
    public Status convertToEntityAttribute(Integer code) {
        return Status.fromCode(code);
    }
}
```

## 2.8 Java 8 日期时间类型如何映射？

**Q: Java 8 日期时间类型如何映射？**

在 Hibernate 5.x 中，需要添加 `hibernate-java8` 依赖，或使用 `@Temporal`（仅限 `java.util.Date`）。**Hibernate 5.3+ / 6.x 原生支持 Java 8 时间类型**，无需额外配置。

```java
@Entity
public class Event {
    @Column(columnDefinition = "DATE")
    private LocalDate startDate;       // 日期（无时间）

    @Column(columnDefinition = "TIMESTAMP")
    private LocalDateTime startTime;   // 日期+时间（无时区）

    @Column(columnDefinition = "TIMESTAMP WITH TIME ZONE")
    private ZonedDateTime zonedTime;   // 日期+时间+时区

    @Column(columnDefinition = "TIME")
    private LocalTime time;            // 时间

    private Instant createdAt;         // 时间戳（UTC），映射为 TIMESTAMP
    private Duration duration;         // Hibernate 6 原生支持
    private YearMonth yearMonth;       // Hibernate 6 原生支持
}
```

**推荐**：
- 业务日期（生日）→ `LocalDate`
- 业务时间（营业时间）→ `LocalTime`
- 业务日期时间（会议时间，固定时区）→ `LocalDateTime`
- 绝对时间点（创建时间、修改时间）→ `Instant`
- 跨时区场景 → `ZonedDateTime` 或 `OffsetDateTime`

Hibernate 6 的 JDBC 映射：
```java
// Hibernate 6 可以指定标准 JDBC 类型
@JdbcTypeCode(SqlTypes.TIMESTAMP_UTC)
private Instant createdAt;
```

## 2.9 大字段（CLOB/BLOB）如何映射？

**Q: 大字段（CLOB/BLOB）如何映射？**

```java
@Entity
public class Article {
    @Lob
    @Column(columnDefinition = "CLOB")
    private String content;        // 大文本，映射 CLOB

    @Lob
    @Column(columnDefinition = "BLOB")
    private byte[] image;          // 二进制，映射 BLOB

    @Lob
    @Nationalized                    // 支持 Unicode（NVARCHAR/NCLOB）
    private String unicodeContent;
}
```

**懒加载大字段**：
```java
@Entity
public class Article {
    @Lob
    @Basic(fetch = FetchType.LAZY)  // 懒加载，仅在访问时查询
    private String content;
}
```

**注意事项**：
- `@Basic(fetch = FetchType.LAZY)` 对大字段的懒加载依赖**字节码增强**，否则在 ToOne 关联上可能失效。
- PostgreSQL 中 `@Lob` 字符串映射为 TEXT/OID，需使用 `@Column(columnDefinition = "TEXT")` 或 `@JdbcTypeCode(SqlTypes.LONG32VARCHAR)`。
- 对于超大文件（GB 级别），不应存入数据库，应存储文件路径或使用 OID。

# 3. 关联映射面试题

## 3.1 mappedBy 的作用是什么？

**Q: mappedBy 的作用是什么？**

`mappedBy` 在双向关联中指定**关系的拥有方（Owner）**。它出现在**非拥有方（Inverse Side）**，值指向拥有方中对应的字段名。

```java
// 拥有方（Owner）— 持有外键
@Entity
public class Order {
    @ManyToOne
    @JoinColumn(name = "user_id")  // 外键字段
    private User user;
}

// 非拥有方（Inverse）— mappedBy 指向拥有方的字段
@Entity
public class User {
    @OneToMany(mappedBy = "user")  // "user" 是 Order 中的字段名
    private List<Order> orders = new ArrayList<>();
}
```

**核心理解**：
- 只有拥有方的外键列会被持久化，非拥有方的修改**不影响数据库**。
- 不使用 `mappedBy` 时，Hibernate 会以为 `@OneToMany` 是单向的，创建一个**中间表**。
- 设置 `mappedBy` 后，Hibernate 知道外键在对方表，不会创建额外表。

**最佳实践**：双向关联中总是设置 `mappedBy`，同时在设置关联时双向维护：
```java
public void addOrder(Order order) {
    orders.add(order);
    order.setUser(this);  // 双向维护
}
```

## 3.2 单向和双向关联有什么区别？如何选择？

**Q: 单向和双向关联有什么区别？如何选择？**

| 维度 | 单向关联 | 双向关联 |
|------|---------|---------|
| 导航方向 | 仅一端可导航到另一端 | 两端可互相导航 |
| 维护复杂度 | 低 | 高（需同步两端） |
| 数据库表 | 无额外表（外键在拥有方） | 同上（mappedBy 确保不生成中间表） |
| JSON 序列化 | 无循环引用问题 | 需处理循环引用 |

**选择原则**：
- **优先使用单向关联**：大多数场景下，只有一端需要导航。
- **仅在确实需要双向导航时**才使用双向关联（如从 User 查 Orders 和从 Order 查 User 都频繁）。
- `@OneToMany` 单向会在数据库中多出一个中间表，若无特殊需求应使用双向（`mappedBy`）。

```java
// 单向：订单知道用户，用户不知道订单
// 适用：大多数场景（查询用户时不需要知道他所有订单）
@Entity
public class Order {
    @ManyToOne
    private User user;
}

// 双向：两端都需要导航
// 适用：领域驱动设计（User 是聚合根，管理其 Orders）
@Entity
public class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}
```

## 3.3 CascadeType 有哪些？orphanRemoval 和 CascadeType.REMOVE 的区别？

**Q: CascadeType 有哪些？orphanRemoval 和 CascadeType.REMOVE 的区别？**

**CascadeType 取值**：
| 类型 | 作用 |
|------|------|
| PERSIST | 级联保存（persist 父 → persist 子） |
| MERGE | 级联合并（merge 父 → merge 子） |
| REMOVE | 级联删除（remove 父 → remove 子） |
| REFRESH | 级联刷新（refresh 父 → refresh 子） |
| DETACH | 级联游离（detach 父 → detach 子） |
| ALL | 以上全部 |

**orphanRemoval vs CascadeType.REMOVE**：
```java
// CascadeType.REMOVE：删除父时级联删除子
em.remove(parent);  // 子也被移除

// orphanRemoval：子从集合中移除时自动删除
parent.getChildren().remove(child);  // 自动 DELETE FROM children WHERE id = ?
```

| 维度 | CascadeType.REMOVE | orphanRemoval |
|------|-------------------|---------------|
| 触发时机 | 调用 `remove(父)` | 子从父集合中移除 |
| 本质含义 | 级联删除 | 孤儿删除（没有父的子自动删除） |
| 合用 | 通常一起使用 | `orphanRemoval = true` 时隐式包含 `CascadeType.REMOVE` |

```java
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Child> children = new ArrayList<>();
// CascadeType.ALL + orphanRemoval：完整的父子生命周期管理
```

## 3.4 FetchType.LAZY 和 EAGER 的区别？@ManyToOne 默认是什么？

**Q: FetchType.LAZY 和 EAGER 的区别？@ManyToOne 默认是什么？**

| 维度 | LAZY（懒加载） | EAGER（立即加载） |
|------|---------------|-------------------|
| 加载时机 | 首次访问关联属性时 | 与父实体同时加载 |
| SQL 执行 | 延迟到访问时 | 立即执行（JOIN 或额外 SELECT） |
| 性能影响 | 可能导致 N+1 问题 | 可能加载不必要的数据 |
| 默认 | `@OneToMany`、`@ManyToMany` | `@ManyToOne`、`@OneToOne` |

```java
// @ManyToOne 和 @OneToOne 默认 EAGER（这是常见的坑！）
@Entity
public class Order {
    @ManyToOne  // 默认 FetchType.EAGER
    private User user;   // 查询 Order 时总是同时 JOIN User
}

// 最佳实践：所有 ToOne 关联显式设为 LAZY
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)  // 显式设为 LAZY
    @JoinColumn(name = "user_id")
    private User user;
}
```

**重要原则**：**所有 `@ManyToOne` 和 `@OneToOne` 都应显式设为 `FetchType.LAZY`**，需要立即加载时用 `JOIN FETCH`。

## 3.5 为什么 @OneToOne 懒加载经常失效？如何解决？

**Q: 为什么 @OneToOne 懒加载经常失效？如何解决？**

**原因**：Hibernate 无法仅用外键判断关联对象是否为 null（不查表无法区分 null 和代理），因此即使配置了 `FetchType.LAZY`，Hibernate 通常也会**立即查询**。

```java
// 这种写法下懒加载通常失效
@Entity
public class User {
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")
    private Profile profile;  // Hibernate 仍会立即 SELECT Profile
}
```

**解决方案**：

1. **使用字节码增强**（Build-Time / Runtime Bytecode Enhancement）：
   ```xml
   <!-- Maven 插件启用字节码增强 -->
   <plugin>
       <groupId>org.hibernate.orm.tooling</groupId>
       <artifactId>hibernate-enhance-maven-plugin</artifactId>
       <configuration>
           <enableLazyInitialization>true</enableLazyInitialization>
       </configuration>
   </plugin>
   ```

2. **设为 optional = false**（约束非空时无需判断 null）：
   ```java
   @OneToOne(fetch = FetchType.LAZY, optional = false)
   @JoinColumn(name = "profile_id", nullable = false)
   private Profile profile;  // 此时代理有效，无需查询
   ```

3. **使用 @MapsId 共享主键**（Profile 的 ID 即 User 的 ID）：
   ```java
   @Entity
   public class User {
       @Id private Long id;
       @OneToOne(fetch = FetchType.LAZY, mappedBy = "user", optional = false)
       private Profile profile;
   }
   @Entity
   public class Profile {
       @Id private Long id;  // 与 User 共享 ID
       @OneToOne @MapsId
       @JoinColumn(name = "id")
       private User user;
   }
   ```

## 3.6 @ManyToMany 的局限性？如何扩展？

**Q: @ManyToMany 的局限性？如何扩展？**

`@ManyToMany` 的中间表由 Hibernate 自动管理，开发者**无法直接操作**中间表。局限性：
- 中间表只能有**两个外键列**，无法添加额外字段（如 `created_at`、`role`）。
- 无法对中间表执行独立查询。
- 批量操作效率低。

**解决方案**：拆分为两个 `@OneToMany` 关联 + 显式中间实体。

```java
// 原始 ManyToMany（限制了中间表）
@ManyToMany
@JoinTable(name = "user_role",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id"))
private Set<Role> roles;

// 升级方案：显式中间实体
@Entity
@Table(name = "user_role")
public class UserRole {
    @EmbeddedId
    private UserRoleId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("userId")
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("roleId")
    private Role role;

    private LocalDateTime assignedAt;   // 中间表自有字段
    private String assignedBy;          // 中间表自有字段
}

@Embeddable
public class UserRoleId implements Serializable {
    private Long userId;
    private Long roleId;
}
```

这样中间表就变成了一个完整的实体，可以添加任意额外字段、独立查询和批量操作。

## 3.7 @OrderColumn 和 @OrderBy 的区别？

**Q: @OrderColumn 和 @OrderBy 的区别？**

| 维度 | @OrderColumn | @OrderBy |
|------|-------------|----------|
| 实现方式 | 数据库多一个排序列（`int`） | SQL 的 ORDER BY 子句 |
| 持久化顺序 | 通过列值维护持久化顺序 | 内存排序，不持久化 |
| 性能 | 更新顺序需 UPDATE 排序列 | 无额外 UPDATE |
| 灵活性 | 支持任意顺序调整 | 依赖字段值排序 |
| 空值处理 | 不友好 | SQL 语义处理 |

```java
// @OrderColumn — 数据库多一个 orders_index 列维护顺序
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
@OrderColumn(name = "orders_index")  // 数据库维护排列顺序
private List<Order> orders;

// @OrderBy — 查询时 ORDER BY 排序
@OneToMany(mappedBy = "user")
@OrderBy("createdAt DESC")  // 仅影响查询，不持久化
private List<Order> orders;

// 等价于
@OrderBy("createdAt DESC")
// SQL: SELECT ... FROM orders WHERE user_id = ? ORDER BY created_at DESC
```

**推荐**：大多数场景使用 `@OrderBy`（简单、高效），除非需要用户自定义排序并持久化。

## 3.8 什么是 N+1 问题？如何解决？

**Q: 什么是 N+1 问题？如何解决？**

**N+1 问题**：查询 N 个父实体后，访问每个父实体的懒加载关联时各执行 1 次查询，总共 N+1 次 SQL。

```java
// 问题代码
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class)
                        .getResultList();  // 1 次查询，返回 N 条
for (Order order : orders) {
    System.out.println(order.getUser().getName());  // N 次查询（每次懒加载 User）
}
// 总计：1 + N 次 SQL
```

**解决方案**：

1. **JOIN FETCH**（最常用）：
   ```java
   List<Order> orders = em.createQuery(
       "SELECT o FROM Order o JOIN FETCH o.user", Order.class)
       .getResultList();  // 1 次 SQL := SELECT o.*, u.* FROM orders o JOIN users u
   ```

2. **@EntityGraph**（声明式，更灵活）：
   ```java
   @EntityGraph(attributePaths = {"user", "items"})
   List<Order> findAll();
   ```

3. **@BatchSize**（批量加载）：
   ```java
   @BatchSize(size = 100)
   @OneToMany(mappedBy = "user")
   private List<Order> orders;
   // N 次查询变为 N/100 次
   ```

4. **次优方案**：当不需要关联数据时，使用 DTO 投影而非实体查询。

## 3.9 双向关联的 JSON 序列化循环引用如何解决？

**Q: 双向关联的 JSON 序列化循环引用如何解决？**

双向关联在 Jackson 序列化时会产生无限递归（`User → Orders → Order → User → ...`），导致 `StackOverflowError`。

**解决方案**：

1. **@JsonIgnore**（简单粗暴）：
   ```java
   @OneToMany(mappedBy = "user")
   @JsonIgnore  // 完全忽略，不序列化
   private List<Order> orders;
   ```

2. **@JsonManagedReference / @JsonBackReference**（仅显示一端）：
   ```java
   public class User {
       @JsonManagedReference  // 被序列化
       @OneToMany(mappedBy = "user")
       private List<Order> orders;
   }
   public class Order {
       @JsonBackReference  // 不序列化
       @ManyToOne
       private User user;
   }
   ```

3. **@JsonIdentityInfo**（推荐）：
   ```java
   @JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
   public class User {
       @OneToMany(mappedBy = "user")
       private List<Order> orders;
   }
   // 输出：{"id":1, "orders":[{"id":100, "user":1}]}  — 第二次出现用 ID 替代
   ```

4. **DTO 投影**（最佳实践）：
   ```java
   public class OrderDTO {
       private Long id;
       private Long userId;  // 不包含整个 User 对象
   }
   // 完全避免实体参与序列化
   ```

# 4. 继承映射面试题

## 4.1 四种继承策略是什么？各自的优缺点？

**Q: 四种继承策略是什么？各自的优缺点？**

| 策略 | 表结构 | 说明 |
|------|--------|------|
| SINGLE_TABLE | 1 张表 | 所有子类字段合并到一张表，使用鉴别器列区分 |
| JOINED | N+1 张表 | 父类一张表，每个子类各一张表，通过 JOIN 获取全部字段 |
| TABLE_PER_CLASS | N 张表 | 每个具体子类一张表，包含所有字段（含父类字段） |
| MappedSuperclass | 无父类表 | 父类不是实体，仅字段复用（严格说不算继承策略） |

```java
// SINGLE_TABLE
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Payment { @Id Long id; BigDecimal amount; }

@Entity @DiscriminatorValue("CARD")
public class CardPayment extends Payment { String cardNumber; }

@Entity @DiscriminatorValue("WX")
public class WechatPayment extends Payment { String openId; }

// JOINED
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment { @Id Long id; BigDecimal amount; }

@Entity @PrimaryKeyJoinColumn(name = "payment_id")
public class CardPayment extends Payment { String cardNumber; }

// TABLE_PER_CLASS
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment { @Id @GeneratedValue(strategy = GenerationType.TABLE) Long id; }
```

| 策略 | 优点 | 缺点 |
|------|------|------|
| SINGLE_TABLE | 查询最快（无需 JOIN） | 大量 NULL 列，NOT NULL 约束困难 |
| JOINED | 数据规范，支持 NOT NULL | 多表 JOIN 性能差，插入需多表 |
| TABLE_PER_CLASS | 每张表干净 | 多态查询需 UNION ALL，IDENTITY 主键不可用 |

**推荐**：
- 子类字段少 → `SINGLE_TABLE`
- 强调数据完整性 → `JOINED`
- 几乎不执行多态查询 → `TABLE_PER_CLASS`

## 4.2 SINGLE_TABLE 策略的鉴别器列如何处理？

**Q: SINGLE_TABLE 策略的鉴别器列如何处理？**

鉴别器列（Discriminator Column）用于在同一张表中区分不同子类类型。

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type",
                     discriminatorType = DiscriminatorType.STRING,
                     length = 10)
@DiscriminatorValue("BASE")  // 父类自己的鉴别值（当父类非 abstract 时）
public class Payment {
    @Id @GeneratedValue
    private Long id;
}

@Entity
@DiscriminatorValue("CARD")  // 子类鉴别值
public class CardPayment extends Payment {
    private String cardNumber;
}

@Entity
@DiscriminatorValue("WECHAT")
public class WechatPayment extends Payment {
    private String openId;
}
```

**DiscriminatorType**：
- `STRING`：推荐，可读性好
- `INTEGER`：性能稍好
- `CHAR`：单字符

**Hibernate 特有**：支持 `@DiscriminatorFormula` 自定义 SQL 表达式：
```java
@DiscriminatorFormula(
    "CASE WHEN card_number IS NOT NULL THEN 'CARD' " +
    "     WHEN open_id IS NOT NULL THEN 'WECHAT' " +
    "     ELSE 'BASE' END"
)
```

## 4.3 TABLE_PER_CLASS 的主键策略有什么限制？

**Q: TABLE_PER_CLASS 的主键策略有什么限制？**

**核心限制**：`TABLE_PER_CLASS` 策略**不能使用 `IDENTITY` 主键生成策略**，因为多个子类表各自有自增列会导致 ID 冲突。

```java
// 错误：IDENTITY 在 TABLE_PER_CLASS 中不可用
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 错误！子类表 ID 会冲突
    private Long id;
}
```

**解决方案**：
1. **TABLE 策略**（Hibernate 自动生成序列表）：
   ```java
   @GeneratedValue(strategy = GenerationType.TABLE)
   private Long id;
   // CREATE TABLE hibernate_sequences (sequence_name VARCHAR, next_val BIGINT);
   ```

2. **SEQUENCE 策略**（支持 SEQUENCE 的数据库）：
   ```java
   @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "payment_seq")
   @SequenceGenerator(name = "payment_seq", sequenceName = "payment_seq", allocationSize = 100)
   private Long id;
   ```

3. **UUID 策略**（Hibernate 6+）：
   ```java
   @Id
   @GeneratedValue(strategy = GenerationType.UUID)
   private UUID id;
   ```

# 5. 查询相关面试题

## 5.1 HQL 和 SQL 的区别？

**Q: HQL 和 SQL 的区别？**

| 维度 | HQL（Hibernate Query Language） | SQL |
|------|--------------------------------|-----|
| 操作目标 | 实体对象 | 数据库表和列 |
| 面向对象 | 支持多态、继承、关联导航 | 面向关系 |
| 可移植性 | 高（方言翻译） | 低（绑定数据库） |
| 完整 SQL | 仅支持 SELECT（Hibernate 6 支持 INSERT/UPDATE/DELETE） | 全部 DML/DDL |
| 特殊函数 | 不支持数据库特有函数 | 完全支持 |

```java
// HQL — 操作实体和属性
Query q = em.createQuery(
    "SELECT o FROM Order o JOIN o.user u WHERE u.name = :name");
q.setParameter("name", "张三");
List<Order> orders = q.getResultList();

// SQL — 操作表和列
Query q = em.createNativeQuery(
    "SELECT o.* FROM t_order o JOIN t_user u ON o.user_id = u.id WHERE u.name = :name");
```

**使用原则**：优先 HQL/JPQL，需数据库特定特性时使用原生 SQL。

## 5.2 JOIN FETCH 的作用是什么？

**Q: JOIN FETCH 的作用是什么？**

`JOIN FETCH` 一次性将关联实体加载到父实体的属性中，将 LAZY 变为 EAGER，用于解决 N+1 问题。

```java
// 不使用 JOIN FETCH：N+1 问题
List<User> users = em.createQuery("SELECT u FROM User u", User.class).getResultList();

// 使用 JOIN FETCH：1 次 SQL 完成
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders", User.class)
    .getResultList();
// 生成的 SQL：SELECT u.*, o.* FROM t_user u INNER JOIN t_order o ON u.id = o.user_id
```

**注意**：
- `JOIN FETCH` 要求关联的 `fetch` 类型必须是 `LAZY`（持久化上下文不考虑 fetch 设置）。
- 可连续 FETCH 多层：`JOIN FETCH u.orders o JOIN FETCH o.items`。
- 不能与 `@OneToMany` 集合的 `JOIN FETCH` + 分页同时使用（会在内存中分页）。

## 5.3 JOIN FETCH 和 JOIN 有什么区别？

**Q: JOIN FETCH 和 JOIN 有什么区别？**

```java
// JOIN：仅用于条件过滤，不加载关联实体
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN u.orders o WHERE o.amount > 100", User.class)
    .getResultList();
// 返回的 User 中 orders 仍是 LAZY（未初始化）
// 生成 SQL：SELECT u.* FROM t_user u JOIN t_order o ON ... WHERE o.amount > 100

// JOIN FETCH：加载关联实体到属性中
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders o WHERE o.amount > 100", User.class)
    .getResultList();
// 返回的 User 中 orders 已被初始化
// 生成 SQL：SELECT u.*, o.* FROM t_user u JOIN t_order o ON ...
```

| 维度 | JOIN | JOIN FETCH |
|------|------|------------|
| 目的 | 过滤、关联条件 | 预加载关联数据 |
| 关联是否初始化 | 否 | 是 |
| SELECT 子句 | 只查主表 | 同时查主表和关联表 |
| 结果集 | 父实体（关联可能为代理） | 父实体（关联已加载） |

## 5.4 JOIN FETCH 分页的问题及解决方案？

**Q: JOIN FETCH 分页的问题及解决方案？**

**问题**：对 `@OneToMany` 集合使用 `JOIN FETCH` + `setFirstResult()/setMaxResults()` 时，Hibernate 会发出警告，并在**内存中进行分页**（将全部数据取回再截断），可能导致 OOM。

```
HHH000104: firstResult/maxResults specified with collection fetch;
applying in memory!
```

**原因**：`JOIN FETCH @OneToMany` 使 SQL 结果集的行数是**被关联表的行数**（而非父表行数），数据库层的 LIMIT/OFFSET 无法正确对应父实体的分页。

**解决方案**：

方案一（推荐）— **两阶段查询**：
```java
// 第一步：不 FETCH，只获取分页的 ID
List<Long> ids = em.createQuery(
    "SELECT u.id FROM User u ORDER BY u.id", Long.class)
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();

// 第二步：基于 ID 做 FETCH
List<User> users = em.createQuery(
    "SELECT DISTINCT u FROM User u JOIN FETCH u.orders WHERE u.id IN :ids", User.class)
    .setParameter("ids", ids)
    .setHint(QueryHints.PASS_DISTINCT_THROUGH, false)  // 让 DISTINCT 传递到 SQL
    .getResultList();
```

方案二 — 使用 `@BatchSize`：
```java
@BatchSize(size = 20)
@OneToMany(mappedBy = "user")
private List<Order> orders;
```

方案三 — 改用 `@ManyToOne` 端开始查询：
```java
// 从 Order 端（Many 端）查询，可以安全分页
List<Order> orders = em.createQuery(
    "SELECT o FROM Order o JOIN FETCH o.user", Order.class)
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();
```

## 5.5 MultipleBagFetchException 的原因和解决方案？

**Q: MultipleBagFetchException 的原因和解决方案？**

**原因**：在同一个查询中同时对**多个 `List`（Bag）类型集合**使用 `JOIN FETCH` 时抛出。因为 Bag 是无序集合，多个笛卡尔积的组合会让 Hibernate 无法正确去重和分配结果。

```java
// 错误：同时 FETCH 两个 List
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders JOIN FETCH u.addresses", User.class)
    .getResultList();
// 抛出 MultipleBagFetchException
```

**解决方案**：

1. **将 List 改为 Set**（推荐）：
   ```java
   @OneToMany(mappedBy = "user")
   private Set<Order> orders = new HashSet<>();  // Set 允许同时 FETCH

   @OneToMany(mappedBy = "user")
   private Set<Address> addresses = new HashSet<>();
   ```

2. **拆分为多次查询**：
   ```java
   List<User> users = em.createQuery(
       "SELECT DISTINCT u FROM User u JOIN FETCH u.orders", User.class)
       .getResultList();
   // 再单独加载 addresses（利用一级缓存，query cache 避免重复查 User）
   users = em.createQuery(
       "SELECT DISTINCT u FROM User u JOIN FETCH u.addresses WHERE u IN :users", User.class)
       .setParameter("users", users)
       .getResultList();
   ```

3. **使用 @Fetch(FetchMode.SUBSELECT) 或 @BatchSize**。

## 5.6 EntityGraph 是什么？loadgraph 和 fetchgraph 的区别？

**Q: EntityGraph 是什么？loadgraph 和 fetchgraph 的区别？**

`@EntityGraph` 是 JPA 2.1 提供的声明式方法，用于定义查询时要加载的关联属性。它是解决 N+1 问题的声明式方案。

```java
@Entity
@NamedEntityGraph(
    name = "User.orders",
    attributeNodes = @NamedAttributeNode("orders")
)
public class User { ... }

// 使用时
EntityGraph<User> graph = em.createEntityGraph(User.class);
graph.addAttributeNodes("orders");

Map<String, Object> hints = new HashMap<>();
hints.put("jakarta.persistence.loadgraph", graph);
User user = em.find(User.class, 1L, hints);
```

**loadgraph vs fetchgraph**：

| 维度 | loadgraph | fetchgraph |
|------|-----------|------------|
| 未在 graph 中的 EAGER 属性 | 仍按 EAGER 加载（保留默认行为） | 强制变为 LAZY（只有 graph 中属性加载） |
| 适用场景 | 额外加载一些懒属性 | 精确控制加载哪些属性（更像 DTO） |
| 提示键 | `jakarta.persistence.loadgraph` | `jakarta.persistence.fetchgraph` |

```java
// fetchgraph — 只加载指定的属性
EntityGraph<User> graph = em.createEntityGraph(User.class);
graph.addAttributeNodes("id", "name");  // 只加载 id 和 name

Map<String, Object> hints = new HashMap<>();
hints.put("jakarta.persistence.fetchgraph", graph);
User user = em.find(User.class, 1L, hints);
// user.getEmail() 可能为 null（未加载），即使是 EAGER 也不会加载
```

**Spring Data JPA 中的使用**：
```java
@EntityGraph(attributePaths = {"orders", "orders.items"})
@Query("SELECT u FROM User u")
List<User> findAllWithOrders();
```

## 5.7 Criteria API 和 JPQL 如何选择？

**Q: Criteria API 和 JPQL 如何选择？**

| 维度 | JPQL / HQL | Criteria API |
|------|-----------|--------------|
| 可读性 | 好（类 SQL 语法） | 差（链式调用冗长） |
| 动态查询 | 需拼接字符串（易出错） | 自然支持条件组合 |
| 编译期检查 | 无（字符串） | 部分（元模型/类型安全） |
| 重构友好 | 差 | 好（基于元模型） |
| 学习成本 | 低 | 高 |

```java
// JPQL — 适合固定查询
String jpql = "SELECT u FROM User u WHERE u.age > :age";
List<User> users = em.createQuery(jpql, User.class)
                      .setParameter("age", 18)
                      .getResultList();

// Criteria API — 适合动态查询
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);

List<Predicate> predicates = new ArrayList<>();
if (name != null) predicates.add(cb.equal(root.get("name"), name));
if (age != null) predicates.add(cb.greaterThan(root.get("age"), age));

cq.where(predicates.toArray(new Predicate[0]));
List<User> users = em.createQuery(cq).getResultList();
```

**选择原则**：
- 固定查询 → JPQL
- 动态条件查询（如搜索表单多条件组合）→ Criteria API
- 项目统一使用 JPA Specification（Spring Data JPA）可结合两者优点

## 5.8 原生 SQL 查询的结果如何映射？

**Q: 原生 SQL 查询的结果如何映射？**

```java
// 方式 1：映射为实体
List<User> users = em.createNativeQuery(
    "SELECT * FROM t_user WHERE age > ?1", User.class)
    .setParameter(1, 18)
    .getResultList();

// 方式 2：映射为 DTO（通过 @SqlResultSetMapping 或构造器表达式）
@SqlResultSetMapping(
    name = "UserDTOMapping",
    classes = @ConstructorResult(
        targetClass = UserDTO.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "name", type = String.class),
            @ColumnResult(name = "email", type = String.class)
        }
    )
)
// 使用
List<UserDTO> dtos = em.createNativeQuery("SELECT id, name, email FROM t_user", "UserDTOMapping")
                        .getResultList();

// 方式 3：Hibernate 特有 — ResultTransformer（Hibernate 5）/ TupleTransformer（Hibernate 6）
// Hibernate 6
List<UserDTO> dtos = session.createNativeQuery("SELECT id, name, email FROM t_user", Object[].class)
    .setTupleTransformer((tuple, aliases) -> new UserDTO(
        (Long) tuple[0], (String) tuple[1], (String) tuple[2]))
    .getResultList();

// 方式 4：接口投影（Spring Data JPA）
public interface UserProjection {
    Long getId();
    String getName();
    String getEmail();
}
@Query(value = "SELECT id, name, email FROM t_user WHERE age > :age", nativeQuery = true)
List<UserProjection> findByAge(@Param("age") int age);
```

## 5.9 @NamedNativeQuery 如何配合 @SqlResultSetMapping 使用？

**Q: @NamedNativeQuery 如何配合 @SqlResultSetMapping 使用？**

```java
// 1. 定义结果映射
@SqlResultSetMapping(
    name = "OrderSummaryMapping",
    entities = {
        @EntityResult(entityClass = Order.class, fields = {
            @FieldResult(name = "id", column = "order_id"),
            @FieldResult(name = "amount", column = "order_amount")
        })
    },
    columns = {
        @ColumnResult(name = "user_name", type = String.class),
        @ColumnResult(name = "total_count", type = Long.class)
    }
)
// 2. 定义命名原生查询
@NamedNativeQuery(
    name = "Order.findSummaryByUser",
    query = "SELECT o.id AS order_id, o.amount AS order_amount, " +
            "u.name AS user_name, COUNT(*) AS total_count " +
            "FROM t_order o JOIN t_user u ON o.user_id = u.id " +
            "WHERE u.id = :userId GROUP BY o.id, o.amount, u.name",
    resultSetMapping = "OrderSummaryMapping"
)
@Entity
public class Order { ... }

// 3. 使用
List<Object[]> results = em.createNamedQuery("Order.findSummaryByUser")
                            .setParameter("userId", 1L)
                            .getResultList();
for (Object[] row : results) {
    Order order = (Order) row[0];
    String userName = (String) row[1];
    Long count = (Long) row[2];
}
```

**更推荐的做法**：使用 `@ConstructorResult` 映射到 DTO，而非混合实体和标量。

## 5.10 存储过程如何调用？

**Q: 存储过程如何调用？**

```java
// 方式 1：JPA 标准 StoredProcedureQuery
StoredProcedureQuery spq = em.createStoredProcedureQuery("calculate_total");

// 注册参数
spq.registerStoredProcedureParameter("p_user_id", Long.class, ParameterMode.IN);
spq.registerStoredProcedureParameter("p_start_date", LocalDate.class, ParameterMode.IN);
spq.registerStoredProcedureParameter("p_total", BigDecimal.class, ParameterMode.OUT);

// 设置参数
spq.setParameter("p_user_id", 1L);
spq.setParameter("p_start_date", LocalDate.now().minusMonths(1));

// 执行
spq.execute();
BigDecimal total = (BigDecimal) spq.getOutputParameterValue("p_total");

// 方式 2：命名存储过程查询
@NamedStoredProcedureQuery(
    name = "User.calculateTotal",
    procedureName = "calculate_total",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "p_user_id", type = Long.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "p_total", type = BigDecimal.class)
    }
)
// 使用
StoredProcedureQuery spq = em.createNamedStoredProcedureQuery("User.calculateTotal");

// 方式 3：返回结果集（游标）
spq.registerStoredProcedureParameter("p_cursor", void.class, ParameterMode.REF_CURSOR);
spq.execute();
List<Object[]> rows = spq.getResultList();
```

**Hibernate 6 对于 REF_CURSOR 的参数名**：PostgreSQL 使用 `refcursor`，Oracle 使用 `SYS_REFCURSOR`。

## 5.11 @Formula 的作用和使用场景？

**Q: @Formula 的作用和使用场景？**

`@Formula` 用于在实体中定义一个**计算属性**，其值由 SQL 表达式计算得出，不持久化，是只读的。

```java
@Entity
public class Account {
    @Id
    private Long id;

    private BigDecimal balance;                 // 当前余额

    @Formula("(SELECT COALESCE(SUM(t.amount), 0) FROM t_transaction t WHERE t.account_id = id)")
    private BigDecimal totalTransactionAmount;   // 累计交易额（只读计算字段）

    @Formula("balance * (SELECT rate FROM t_interest_rate WHERE type = 'SAVING')")
    private BigDecimal projectedInterest;        // 预计利息

    @Formula("(SELECT COUNT(*) FROM t_order o WHERE o.account_id = id AND o.status = 'PENDING')")
    private Integer pendingOrderCount;           // 待处理订单数
}

// 使用
Account account = em.find(Account.class, 1L);
System.out.println(account.getTotalTransactionAmount());  // 由子查询计算
```

**使用场景**：
- 计算属性（如余额、计数、汇总）
- 附加关联表的数据而不建立实体关联
- 子查询结果映射到实体的只读字段

**限制**：
- 只读（不能 SET/UPDATE 此字段）
- 依赖原生 SQL，可能影响数据库移植性
- 每次查询实体都执行 `@Formula` SQL，可能影响性能

# 6. 缓存相关面试题

## 6.1 一级缓存和二级缓存的区别？

**Q: 一级缓存和二级缓存的区别？**

| 维度 | 一级缓存（L1 Cache） | 二级缓存（L2 Cache） |
|------|---------------------|----------------------|
| 作用域 | Session / EntityManager | SessionFactory / EntityManagerFactory |
| 生命周期 | Session 关闭时清除 | SessionFactory 关闭时清除 |
| 共享性 | 仅当前 Session 内 | 跨 Session、跨事务 |
| 默认启用 | 是（不可禁用） | 否（需显式配置） |
| 存储内容 | 实体实例引用 | 实体属性值（反序列化后重建） |
| 一致性 | 强一致 | 可能读到旧数据（事务隔离） |
| 实现 | Hibernate 内置 | 需第三方（Ehcache、Hazelcast、Redis） |

```java
// 一级缓存示例
Session session = sf.openSession();
User u1 = session.get(User.class, 1L);  // SELECT
User u2 = session.get(User.class, 1L);  // 无 SELECT（一级缓存命中）
System.out.println(u1 == u2);            // true
session.close();

// 二级缓存示例
Session session1 = sf.openSession();
User u1 = session1.get(User.class, 1L);  // SELECT
session1.close();

Session session2 = sf.openSession();
User u2 = session2.get(User.class, 1L);  // 无 SELECT（二级缓存命中）
System.out.println(u1 == u2);            // false（不同实例，值相同）
session2.close();
```

## 6.2 二级缓存的并发策略有哪些？各自的特点？

**Q: 二级缓存的并发策略有哪些？各自的特点？**

| 策略 | 隔离级别 | 数据一致性 | 性能 |
|------|---------|-----------|------|
| READ_ONLY | 最高 | 数据永不变更 | 最高 |
| NONSTRICT_READ_WRITE | 低 | 极少写入，容忍短暂不一致 | 很高 |
| READ_WRITE | 中 | 更新时加锁，防止脏读 | 中 |
| TRANSACTIONAL | 最高 | 完全支持事务语义 | 最低 |

```java
// READ_ONLY — 适用于字典表、配置表
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
@Entity
public class DictItem { ... }

// NONSTRICT_READ_WRITE — 很少更新的缓存（如用户信息）
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
@Entity
public class User { ... }

// READ_WRITE — 需要保证不脏读的数据
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Order { ... }

// TRANSACTIONAL — 分布式事务环境（需 JTA）
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL)
@Entity
public class Account { ... }
```

**选择优先级**：`READ_ONLY` > `NONSTRICT_READ_WRITE` > `READ_WRITE` > `TRANSACTIONAL`

## 6.3 什么数据适合放在二级缓存？

**Q: 什么数据适合放在二级缓存？**

**适合放入二级缓存的数据**：
- **字典表**（国家、省市、行业分类等）：几乎不变，`READ_ONLY` 策略。
- **变更极少的参考数据**（币种汇率表、配置参数表）。
- **读多写少**的数据（如历史订单、已完成的交易记录）。
- **小规模高频率访问**的数据（常用分类、标签）。

**不适合放入二级缓存的数据**：
- **高频写入**的数据（计数器、状态机）。
- **跨应用共享**的数据（银行账户余额）— 缓存不一致风险高。
- **大数据量**的表（占用内存过大，缓存命中率低）。
- **私有数据**（用户个人信息随 user_id 变化，缓存收益不大）。
- **与时间强相关**的数据（实时日志、监控数据）。

## 6.4 查询缓存的工作原理？什么时候会失效？

**Q: 查询缓存的工作原理？什么时候会失效？**

**查询缓存**缓存的是 JPQL/HQL 查询的**结果实例 ID 列表**（而非实体本身），结合二级缓存才能避免数据库查询。

```xml
<!-- 启用查询缓存 -->
<property name="hibernate.cache.use_query_cache" value="true"/>
```

```java
// 查询缓存工作流程
Query query = session.createQuery("FROM User WHERE age > 18");
query.setCacheable(true);  // 启用查询缓存
query.setCacheRegion("userQueries");
List<User> users = query.list();

// 缓存内容：{ "FROM User WHERE age > 18": [1, 2, 3] }  — 只存 ID
// 实体数据：{ 1: {id:1, name:"张三"}, 2: {...}, 3: {...} }  — 二级缓存（需实体启用缓存）
```

**查询缓存何时失效**：

| 操作 | 失效范围 |
|------|---------|
| 目标实体执行 INSERT | 目标实体所有查询缓存失效 |
| 目标实体执行 UPDATE | 目标实体所有查询缓存失效 |
| 目标实体执行 DELETE | 目标实体所有查询缓存失效 |
| 手动 `evictQueries()` | 指定区域的缓存失效 |

```java
// 缓存失效机制 — 任何 User 表变更，所有 User 查询缓存都失效
session.createQuery("UPDATE User SET age = 20 WHERE id = 1").executeUpdate();
// 触发所有 User 查询缓存的失效（即使查询条件不相关）
```

**生产环境慎用查询缓存**，因为任何表的写操作都会导致该表所有查询缓存失效，收益通常为负。

## 6.5 实体缓存和查询缓存的区别？

**Q: 实体缓存和查询缓存的区别？**

| 维度 | 实体缓存（二级缓存） | 查询缓存 |
|------|---------------------|---------|
| 缓存键 | 实体 ID + 类名 | 查询语句 + 参数值 |
| 缓存值 | 实体属性值 | 实体 ID 列表 |
| 单独可用 | 是（实体缓存独立工作） | 否（必须配合实体缓存才能避免 SQL） |
| 失效粒度 | 单条实体变更 | 整张表任何变更 |
| 适用场景 | 按 ID 查找 | 重复的条件查询 |
| 推荐 | 推荐 | 谨慎使用 |

```java
// 实体缓存：按 ID 查，命中率高
User u = session.get(User.class, 1L);  // 实体缓存命中

// 查询缓存：条件查询，命中率低
List<User> users = session.createQuery("FROM User WHERE status = :s", User.class)
                           .setParameter("s", "ACTIVE")
                           .setCacheable(true)
                           .getResultList();
// 任何 User 的增删改都会使此查询缓存失效
```

## 6.6 二级缓存在集群环境如何处理？

**Q: 二级缓存在集群环境如何处理？**

集群环境的核心挑战是**缓存一致性**：当一个节点的数据被修改后，其他节点的缓存需要失效或更新。

**解决方案**：

1. **分布式缓存**（推荐）— 使用 Hazelcast、Redis、Infinispan：
   ```properties
   # Hazelcast 作为二级缓存
   hibernate.cache.region.factory_class=com.hazelcast.hibernate.HazelcastLocalCacheRegionFactory
   hibernate.cache.hazelcast.instance_name=hzInstance
   ```
   ```yaml
   # Redisson (Redis) 作为二级缓存
   hibernate.cache.region.factory_class=org.redisson.hibernate.RedissonRegionFactory
   ```

2. **缓存同步策略** — JGroups 集群通信 + 本地缓存：
   ```properties
   hibernate.cache.infinispan.cachemanager_config=infinispan.xml
   ```

3. **缓存失效消息** — 通过消息队列广播缓存失效通知：
   ```java
   // 应用层方案：通过 Redis Pub/Sub 公告缓存失效
   public void updateUser(User user) {
       userRepository.save(user);
       redisTemplate.convertAndSend("cache:invalidate:User", user.getId());
   }
   // 各节点监听并清除本地二级缓存
   sessionFactory.getCache().evictEntity(User.class, id);
   ```

4. **数据库级一致性** — 使用 `@Version` 乐观锁 + 二级缓存 READ_WRITE 策略。

**推荐**：使用 Hazelcast（与 Hibernate 集成最好）或 Redis，配合 `READ_WRITE` 策略。

# 7. 事务与锁面试题

## 7.1 乐观锁和悲观锁的区别？

**Q: 乐观锁和悲观锁的区别？**

| 维度 | 乐观锁 | 悲观锁 |
|------|--------|--------|
| 锁获取时机 | 提交时检测冲突 | 读取时就加锁 |
| 实现方式 | 版本号/时间戳 | 数据库行锁 |
| 并发性能 | 高（无锁等待） | 低（持有锁） |
| 冲突处理 | 重试或抛异常 | 等待或抛异常 |
| 适用场景 | 冲突少的场景（读多写少） | 冲突多的场景 |
| 死锁风险 | 无 | 有 |

**乐观锁**：假定冲突很少发生，提交时通过版本号检测。
```java
// 乐观锁实现
@Entity
public class Product {
    @Version
    private Integer version;  // 每次 UPDATE 时 version + 1
}
// UPDATE product SET stock = 50, version = 3 WHERE id = 1 AND version = 2
// 若 version 不匹配（被其他事务修改），抛 OptimisticLockException
```

**悲观锁**：假定冲突一定发生，读取时锁定行。
```java
// 悲观锁实现
Product product = em.find(Product.class, 1L, LockModeType.PESSIMISTIC_WRITE);
// SELECT * FROM product WHERE id = 1 FOR UPDATE  -- 行级锁
product.setStock(product.getStock() - 1);
```

## 7.2 @Version 如何实现乐观锁？

**Q: @Version 如何实现乐观锁？**

```java
@Entity
public class Account {
    @Id private Long id;

    private BigDecimal balance;

    @Version
    private Long version;  // 版本号（可用 int、long、Timestamp、Instant）
}

// 更新时的 SQL：
// em.find() 获取 {id:1, balance:100, version:1}
// account.setBalance(200);
// em.flush() 生成：
// UPDATE account SET balance = 200, version = 2
// WHERE id = 1 AND version = 1
// 若 affected rows = 0 → 说明版本已变（其他事务修改过）→ OptimisticLockException
```

**@Version 支持的类型**：
- `int` / `Integer`
- `long` / `Long`
- `short` / `Short`
- `java.sql.Timestamp`
- `java.time.Instant`（Hibernate 6）

**乐观锁的两种模式**：
- **OPTIMISTIC**：读取时获取版本号，仅 UPDATE 时检查，多用于防止更新丢失。
- **OPTIMISTIC_FORCE_INCREMENT**：即使没修改也强制递增版本号，用于管理父子关系时防止并发修改。

```java
em.lock(account, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
// 等同于 account.setVersion(account.getVersion() + 1); 即使没改 balance
```

## 7.3 LockModeType 有哪些？PESSIMISTIC_READ 和 PESSIMISTIC_WRITE 的区别？

**Q: LockModeType 有哪些？PESSIMISTIC_READ 和 PESSIMISTIC_WRITE 的区别？**

| LockModeType | 描述 | SQL 表现 |
|--------------|------|---------|
| NONE | 无锁 | - |
| OPTIMISTIC | 乐观读锁，提交时检查版本号 | WHERE version = ? |
| OPTIMISTIC_FORCE_INCREMENT | 乐观锁+强制递增版本号 | 同上 + UPDATE version |
| PESSIMISTIC_READ | 共享锁 | `SELECT ... FOR SHARE` |
| PESSIMISTIC_WRITE | 排他锁 | `SELECT ... FOR UPDATE` |
| PESSIMISTIC_FORCE_INCREMENT | 悲观锁+版本号递增 | `SELECT ... FOR UPDATE NOWAIT` |

**PESSIMISTIC_READ vs PESSIMISTIC_WRITE**：

```sql
-- PESSIMISTIC_READ（共享锁）— 其他事务可读，不可写
SELECT * FROM product WHERE id = 1 FOR SHARE;  -- PostgreSQL
SELECT * FROM product WHERE id = 1 LOCK IN SHARE MODE;  -- MySQL 8.0-
-- 允许其他事务也获取 PESSIMISTIC_READ 锁
-- 不允许其他事务获取 PESSIMISTIC_WRITE 锁

-- PESSIMISTIC_WRITE（排他锁）— 其他事务不可读、不可写（视隔离级别）
SELECT * FROM product WHERE id = 1 FOR UPDATE;  -- 所有数据库通用
-- 不允许任何事务再获取该行的任何锁
```

**使用 PESSIMISTIC_WRITE 的技巧** — 添加超时避免死锁：
```java
Map<String, Object> hints = new HashMap<>();
hints.put("jakarta.persistence.lock.timeout", 5000L);  // 毫秒
Product p = em.find(Product.class, 1L, LockModeType.PESSIMISTIC_WRITE, hints);
```

## 7.4 事务隔离级别有哪些？各解决什么问题？

**Q: 事务隔离级别有哪些？各解决什么问题？**

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 并发性能 |
|---------|------|-----------|------|---------|
| READ_UNCOMMITTED | 是 | 是 | 是 | 最高 |
| READ_COMMITTED | 否 | 是 | 是 | 高 |
| REPEATABLE_READ | 否 | 否 | 是* | 中 |
| SERIALIZABLE | 否 | 否 | 否 | 最低 |

*注：MySQL InnoDB 的 REPEATABLE_READ 通过间隙锁避免了幻读。

**问题定义**：
- **脏读**：读到其他事务未提交的修改。
- **不可重复读**：同一事务内两次读取同一行，得到不同数据（其他事务提交了 UPDATE）。
- **幻读**：同一事务内两次查询同一条件，结果集的行数不同（其他事务提交了 INSERT）。

```java
// Hibernate 中设置隔离级别
@Transactional(isolation = Isolation.READ_COMMITTED)  // Spring 方式

// Hibernate 配置
hibernate.connection.isolation=2  // 2 = READ_COMMITTED
```

**大部分应用的推荐级别**：`READ_COMMITTED`（性能与一致性之间的最佳平衡）。

## 7.5 Spring @Transactional 和 Hibernate Session 的关系？

**Q: Spring @Transactional 和 Hibernate Session 的关系？**

Spring 通过 `@Transactional` 管理 Hibernate Session 的生命周期：

1. 进入 `@Transactional` 方法时，Spring 绑定一个 Session 到当前线程（`getCurrentSession()`）。
2. Session 的 `beginTransaction()` / `commit()` / `rollback()` 由 Spring 事务管理器控制。
3. 退出 `@Transactional` 方法时，Spring 刷新（flush）并关闭 Session。

```java
@Transactional  // Spring 管理事务 + Session 生命周期
public void createOrder(OrderDTO dto) {
    Session session = sessionFactory.getCurrentSession();  // 同一个 Session
    // ...
    User user = session.get(User.class, dto.getUserId());  // 懒加载可用
    // 方法结束时：flush → commit → close session
}

// 等价于手动管理：
Session session = sf.openSession();
Transaction tx = session.beginTransaction();
try {
    // ... 业务逻辑
    tx.commit();
} catch (Exception e) {
    tx.rollback();
} finally {
    session.close();
}
```

**关键理解**：
- `@Transactional` 的边界即 Session 的边界。
- 在 `@Transactional` 方法内，懒加载正常工作。
- 在 `@Transactional` 方法外，访问懒加载属性会抛 `LazyInitializationException`。

## 7.6 只读事务如何优化 Hibernate？

**Q: 只读事务如何优化 Hibernate？**

只读事务（`@Transactional(readOnly = true)`）为 Hibernate 提供优化提示：

```java
@Transactional(readOnly = true)
public List<User> findAll() {
    return userRepository.findAll();
}
```

**优化机制**：
1. **禁用脏检查**：Hibernate 跳过 flush 时的脏检查（Dirty Checking），因为不会发生写操作。
2. **禁用快照**：Hibernate 不保留实体的加载快照，节省内存。
3. **JDBC 连接**：Spring 将 JDBC 连接的 `readOnly` 设为 true，数据库可据此优化（如 MySQL 主从读写分离）。
4. **数据库层面**：部分数据库（如 PostgreSQL）对只读事务允许更高并发度。

```java
// Spring 建议：默认方法上标注只读
@Transactional(readOnly = true)  // 类级别默认只读
public class UserService {
    public List<User> findAll() { ... }      // 继承只读

    @Transactional(readOnly = false)         // 写操作覆盖为读写
    public User save(User user) { ... }
}
```

**注意**：`readOnly = true` 仅是一个**提示**（hint），不会阻止你执行写操作（但应避免这样做）。

# 8. 性能优化面试题

## 8.1 Hibernate 有哪些常见的性能问题？

**Q: Hibernate 有哪些常见的性能问题？**

1. **N+1 查询**：懒加载导致循环查询关联实体（最常见）。
2. **EAGER 加载过多数据**：`@ManyToOne` 默认 EAGER，导致每次查询都 JOIN 不需要的表。
3. **批量插入/更新慢**：未配置 `hibernate.jdbc.batch_size` 导致每次 INSERT 都是一次网络往返。
4. **IDENTITY 主键禁用批处理**：`GenerationType.IDENTITY` 导致 JDBC 批处理无法生效。
5. **Open Session in View 过长**：Session 持续到视图渲染，导致数据库连接长时间占用。
6. **脏检查开销**：大对象图在 flush 时需要比较所有属性。
7. **不必要的数据加载**：查询整个实体，实际只需几个字段。
8. **笛卡尔积**：同时 JOIN FETCH 多个集合导致巨大结果集。
9. **查询缓存频繁失效**：任何写操作都清空该表所有查询缓存。
10. **日志过多**：SQL 日志输出导致 I/O 阻塞和字符串拼接开销。

## 8.2 如何发现和解决 N+1 问题？

**Q: 如何发现和解决 N+1 问题？**

**发现方法**：

1. **SQL 日志分析**：开启 SQL 统计和日志。
   ```properties
   hibernate.show_sql=true
   hibernate.format_sql=true
   hibernate.generate_statistics=true
   ```

2. **JavaMelody / Glowroot** 等 APM 工具自动检测 N+1。

3. **Hypersistence Optimizer** 工具（Vlad Mihalcea 开发）：
   ```xml
   <dependency>
       <groupId>io.hypersistence</groupId>
       <artifactId>hypersistence-optimizer</artifactId>
   </dependency>
   ```

4. **数据源代理**（如 datasource-proxy）：
   ```java
   @Bean
   public DataSource dataSource() {
       return ProxyDataSourceBuilder.create(actualDataSource)
           .logQueryBySlf4j(SLF4JLogLevel.INFO)
           .countQuery()
           .build();
   }
   ```

**解决步骤**（按优先级）：
1. 需要关联数据 → `JOIN FETCH` 或 `@EntityGraph`。
2. 不需要关联数据 → DTO 投影（`SELECT new com.example.UserDTO(u.id, u.name)`）。
3. 频繁访问的关联 → 设置 `@BatchSize(size = 50)`。
4. 大量通用查询 → 启用二级缓存（`@Cache`）。

## 8.3 批处理如何配置和优化？

**Q: 批处理如何配置和优化？**

```properties
# JDBC 批处理配置
hibernate.jdbc.batch_size=50                    # 批量 INSERT/UPDATE/DELETE
hibernate.jdbc.fetch_size=50                    # 批量 SELECT
hibernate.order_inserts=true                    # 排序 INSERT 语句以提高批处理效率
hibernate.order_updates=true                    # 排序 UPDATE 语句
hibernate.batch_versioned_data=true             # 对 @Version 实体也使用批处理
hibernate.jdbc.batch_versioned_data=true        # JDBC 级别支持版本化数据批处理
```

**批处理实现模式**：

```java
@Transactional
public void batchImport(List<User> users) {
    Session session = sessionFactory.getCurrentSession();
    int i = 0;
    for (User user : users) {
        session.persist(user);
        i++;
        if (i % 50 == 0) {  // 与 batch_size 对齐
            session.flush();   // 执行批处理 SQL
            session.clear();   // 清空一级缓存，防止 OOM
        }
    }
    // 最后一批
    session.flush();
    session.clear();
}
```

**IDENTITY 主键与批处理的冲突**：
```java
// IDENTITY 主键会禁用 JDBC 批处理
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // 批处理失效
private Long id;

// 解决方案：使用 SEQUENCE（PostgreSQL）或 TABLE（MySQL）策略
@Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_seq", allocationSize = 100)
private Long id;
```

## 8.4 StatelessSession 和 Session 的区别？

**Q: StatelessSession 和 Session 的区别？**

| 维度 | Session | StatelessSession |
|------|---------|-----------------|
| 持久化上下文 | 有（一级缓存） | 无 |
| 脏检查 | 有 | 无 |
| 级联操作 | 支持 | 不支持 |
| 延迟加载 | 支持 | 不支持 |
| 自动 flush | 支持 | 不 flush（直接执行 SQL） |
| 性能 | 低（有缓存/脏检查开销） | 高 |
| 使用场景 | 常规 CRUD | 批量导入/导出、ETL |

```java
// StatelessSession 示例
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

for (User user : users) {
    session.insert(user);  // 直接执行 INSERT，无脏检查，无缓存
}
tx.commit();
session.close();

// 批量更新
session.createNativeQuery("UPDATE t_user SET status = :s WHERE id = :id")
       .setParameter("s", "ACTIVE")
       .setParameter("id", id)
       .executeUpdate();
```

**注意事项**：
- `StatelessSession` 返回的实体是 detached 状态，修改后不会自动 UPDATE。
- 不支持关联和集合的懒加载（无代理）。
- 适合单纯的批量插入/更新/删除场景。

## 8.5 字节码增强的作用？

**Q: 字节码增强的作用？**

字节码增强（Bytecode Enhancement）是在编译期或类加载期修改 Hibernate 实体类的字节码，以支持高级特性。

```xml
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
                <enableDirtyTracking>true</enableDirtyTracking>
                <enableAssociationManagement>true</enableAssociationManagement>
            </configuration>
            <goals><goal>enhance</goal></goals>
        </execution>
    </executions>
</plugin>
```

**增强项**：

1. **懒加载增强**（`enableLazyInitialization`）：使 `@OneToOne` 和 `@Basic(fetch = LAZY)` 的懒加载真正生效，无需代理对象。

2. **脏检查增强**（`enableDirtyTracking`）：实体自己追踪属性变化，避免 flush 时与快照的全量比较。
   ```java
   // 不使用增强：flush 时比较所有字段
   // 使用增强后：实体内部记录 $$_hibernate_setName("新值") 标记为脏
   ```

3. **关联管理增强**（`enableAssociationManagement`）：自动维护双向关联。
   ```java
   // 增强前：需手动维护双向关联
   order.setUser(user);
   user.getOrders().add(order);

   // 增强后：自动同步
   order.setUser(user);
   // user.getOrders() 自动包含此 order
   ```

## 8.6 DTO 投影相比实体查询有什么优势？

**Q: DTO 投影相比实体查询有什么优势？**

```java
// 实体查询：查询所有字段，返回 managed 实体
List<User> users = em.createQuery("SELECT u FROM User u", User.class).getResultList();
// SELECT id, name, email, password, created_at, updated_at, ... FROM t_user

// DTO 投影：只查询需要的字段，返回 DTO
List<UserDTO> dtos = em.createQuery(
    "SELECT new com.example.UserDTO(u.id, u.name, u.email) FROM User u", UserDTO.class)
    .getResultList();
// SELECT id, name, email FROM t_user
```

| 维度 | 实体查询 | DTO 投影 |
|------|---------|---------|
| SQL 字段 | 所有字段 | 按需选择 |
| 内存开销 | 大（managed 实体+快照） | 小（普通对象） |
| 脏检查 | 参与 | 不参与 |
| 一级缓存 | 占用 | 不占用 |
| 可修改 | 是（跟踪变更） | 否（只读） |
| 适用场景 | 需要修改并保存 | 只读展示/报表 |

**性能差异**：对 10000 条记录，DTO 投影比实体查询快 2-5 倍（减少字段传输和对象管理开销）。

## 8.7 Open Session in View 的优缺点？

**Q: Open Session in View 的优缺点？**

OSIV 模式让 Hibernate Session 的生命周期延长到视图渲染阶段（超出 `@Transactional` 范围），从而在视图层可以使用懒加载。

```properties
# Spring Boot 默认启用（已废弃警告）
spring.jpa.open-in-view=true
```

**优点**：
- 简化开发：视图层可以透明访问懒加载属性，无需手动处理。
- 适合快速原型和简单 CRUD 应用。

**缺点**：
- 数据库连接在**整个 HTTP 请求期间**都被占用（包括视图渲染），降低连接池利用率。
- 高并发下容易耗尽连接池，导致应用不可用。
- 视图层触发 N+1 查询难以追踪和优化。
- 破坏分层：视图层代码隐式依赖数据库连接。

**推荐**：生产环境**关闭 OSIV**，使用 DTO 在 Service 层完成数据组装：
```properties
spring.jpa.open-in-view=false
```

```java
// 关闭 OSIV 后的最佳实践
@Transactional(readOnly = true)
public OrderDetailDTO getOrderDetail(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    return OrderDetailDTO.from(order);  // 在事务内完成所有数据加载和转换
}
```

## 8.8 IDENTITY 主键为什么影响批处理？

**Q: IDENTITY 主键为什么影响批处理？**

`GenerationType.IDENTITY` 依赖数据库自增列（`AUTO_INCREMENT`），INSERT 语句执行后，Hibernate 必须立即获取数据库生成的 ID 值，才能将实体放入持久化上下文。这使得 JDBC 无法将多条 INSERT 语句批量发送。

```java
// IDENTITY 主键时的执行序列
session.persist(user1);  // 立即执行 INSERT，获取 ID
session.persist(user2);  // 立即执行 INSERT，获取 ID
session.persist(user3);  // 立即执行 INSERT，获取 ID
// 每条都是独立数据库往返（无批处理）

// SEQUENCE 主键时的执行序列
session.persist(user1);  // 不执行 SQL，仅在内存分配 ID
session.persist(user2);  // 不执行 SQL
session.persist(user3);  // 不执行 SQL
session.flush();         // 一次性发送 3 条 INSERT 的批处理
```

**技术原因**：JDBC 批处理要求在同一次 `executeBatch()` 调用前准备好所有参数，而 IDENTITY 要求每次 INSERT 后立即获取返回值（`Statement.getGeneratedKeys()`），破坏了批处理模式。

**解决方案**：
- PostgreSQL/Oracle → 使用 `SEQUENCE`
- MySQL → 使用 `TABLE` 策略或 `UUID` 策略
- 或使用 Hibernate 的 `hibernate.jdbc.batch_size` + MySQL 的 `rewriteBatchedStatements=true`

# 9. 高级特性面试题

## 9.1 Interceptor 和 JPA EntityListeners 的区别？

**Q: Interceptor 和 JPA EntityListeners 的区别？**

| 维度 | Hibernate Interceptor | JPA EntityListeners |
|------|----------------------|---------------------|
| 来源 | Hibernate 原生 | JPA 标准 |
| 范围 | 全局（SessionFactory 级别） | 类级别（`@EntityListeners`） |
| 回调时机 | 更细粒度（onLoad、onFlushDirty 等） | 标准生命周期（`@PrePersist` 等） |
| 审计 | 可修改状态和获取 Session | 适合简单审计和数据填充 |

```java
// Hibernate Interceptor — 全局，可访问 Session
public class AuditInterceptor extends EmptyInterceptor {
    @Override
    public boolean onSave(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        // setValue(state, propertyNames, "createdAt", Instant.now());
        return true;
    }
}
// 注册：sessionFactory.withOptions().interceptor(new AuditInterceptor()).openSession();

// JPA EntityListener — 指定类，不能注入 Session
public class AuditListener {
    @PrePersist
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            ((Auditable) entity).setCreatedAt(Instant.now());
        }
    }
}
@Entity @EntityListeners(AuditListener.class)
public class User { ... }
```

**动态注入依赖到 EntityListener** — 使用 Spring `@Component` + `@Bean`：
```java
@Component
public class AuditListener {
    private static SecurityContext securityContext;  // 或通过 Spring 注入

    @PrePersist
    public void prePersist(Object entity) { ... }
}
```

## 9.2 @Filter 和 @Where 的区别？

**Q: @Filter 和 @Where 的区别？**

| 维度 | @Filter | @Where |
|------|---------|--------|
| 来源 | Hibernate 原生 | Hibernate 原生（已废弃，推荐 @SQLRestriction） |
| 开关 | Session 级别动态启用/禁用 | 静态，始终生效 |
| 参数 | 支持运行时参数 | 不支持参数（静态条件） |
| 条件 | 可在运行时改变 | 写死在注解中 |
| 继承 | 子类自动继承 | 子类不继承（Hibernate 6 中 @SQLRestriction 可继承） |

```java
// @FilterDef — 定义过滤器
@FilterDef(name = "activeFilter", parameters = @ParamDef(name = "status", type = String.class))
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = Long.class))
@Entity
public class User {
    @Filter(name = "activeFilter", condition = "status = :status")
    @Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
    private Long id;
}

// 使用时动态启用
session.enableFilter("activeFilter").setParameter("status", "ACTIVE");
session.enableFilter("tenantFilter").setParameter("tenantId", currentTenantId);

// 所有 User 查询自动添加 WHERE status = 'ACTIVE' AND tenant_id = 123
List<User> users = session.createQuery("FROM User", User.class).getResultList();

// 禁用
session.disableFilter("activeFilter");
```

```java
// @Where（已废弃，Hibernate 6 用 @SQLRestriction）
@Entity
@SQLRestriction("status = 'ACTIVE'")  // Hibernate 6 替代 @Where
public class User {
    private String status;
}
// 所有简单查询自动加上 AND status = 'ACTIVE'
```

## 9.3 如何实现软删除？

**Q: 如何实现软删除？**

**方案一：@SQLDelete + @SQLRestriction**（推荐，Hibernate 6）：

```java
@Entity
@SQLDelete(sql = "UPDATE t_user SET deleted = true, deleted_at = CURRENT_TIMESTAMP WHERE id = ?")
@SQLRestriction("deleted = false")
public class User {
    @Id private Long id;
    private String name;
    private boolean deleted;
    private LocalDateTime deletedAt;
}

// 使用
em.remove(user);
// 不执行 DELETE，而是执行 UPDATE t_user SET deleted = true WHERE id = ?
// 查询时自动添加 AND deleted = false
```

**方案二：@Filter 实现**（需要灵活控制时）：

```java
@FilterDef(name = "softDeleteFilter", defaultCondition = "deleted = false")
@Entity
@Filter(name = "softDeleteFilter")
public class User {
    private boolean deleted;
}

// 默认启用软删除过滤器
session.enableFilter("softDeleteFilter");

// 需要查已删除数据时暂时禁用
session.disableFilter("softDeleteFilter");
List<User> deletedUsers = session.createQuery("FROM User", User.class).getResultList();
session.enableFilter("softDeleteFilter");
```

## 9.4 多租户的实现方案有哪些？

**Q: 多租户的实现方案有哪些？**

Hibernate 从 4.0 开始支持多租户（Multi-Tenancy），主要有三种方案：

| 方案 | 数据库 | 表 | 隔离性 | 成本 |
|------|--------|-----|--------|------|
| DATABASE | 每个租户独立数据库 | 独立 | 最高 | 最高 |
| SCHEMA | 共用数据库，独立 Schema | 独立 | 中 | 中 |
| DISCRIMINATOR | 共用数据库和 Schema | 共享（tenant_id 列） | 低 | 最低 |

```java
// DATABASE 方案 — 根据租户选择不同数据库连接
@Component
public class TenantConnectionProvider implements MultiTenantConnectionProvider<String> {
    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        DataSource ds = tenantDataSourceMap.get(tenantId);  // 每个租户独立数据源
        return ds.getConnection();
    }
}

// SCHEMA 方案 — 同一数据库不同 Schema
@Component
public class SchemaMultiTenantConnectionProvider implements MultiTenantConnectionProvider<String> {
    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        Connection conn = defaultDataSource.getConnection();
        conn.setSchema(tenantId);  // PostgreSQL 设置 Schema
        return conn;
    }
}

// DISCRIMINATOR 方案 — 使用 @Filter 过滤租户数据
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = Long.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
@Entity
public class Order {
    @Column(name = "tenant_id")
    private Long tenantId;
}
```

**配置**：
```properties
hibernate.multi_tenancy=DATABASE  # DATABASE | SCHEMA | DISCRIMINATOR
hibernate.tenant_identifier_resolver=com.example.TenantResolver
hibernate.multi_tenant_connection_provider=com.example.TenantConnectionProvider
```

## 9.5 如何自定义类型映射？

**Q: 如何自定义类型映射？**

通过实现 JPA 的 `AttributeConverter` 或 Hibernate 的 `UserType` 来进行自定义类型映射。

**JPA AttributeConverter（推荐，简单场景）**：

```java
// 将 JSON 字符串映射为 Java 对象
@Converter
public class JsonConverter implements AttributeConverter<Map<String, Object>, String> {

    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Map<String, Object> attribute) {
        try {
            return mapper.writeValueAsString(attribute);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public Map<String, Object> convertToEntityAttribute(String dbData) {
        try {
            return mapper.readValue(dbData, new TypeReference<>() {});
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}

@Entity
public class Order {
    @Convert(converter = JsonConverter.class)
    @Column(columnDefinition = "JSONB")  // PostgreSQL JSONB
    private Map<String, Object> extraData;  // 自动序列化/反序列化
}

// 也可全局启用 autoApply
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, BigDecimal> { ... }
```

**Hibernate UserType（复杂场景，需要更细粒度控制）**：

```java
public class MoneyType implements UserType<Money> {
    @Override
    public int getSqlType() { return Types.NUMERIC; }

    @Override
    public Class<Money> returnedClass() { return Money.class; }

    @Override
    public Money nullSafeGet(ResultSet rs, int position, SharedSessionContractImplementor session, Object owner) throws SQLException {
        BigDecimal value = rs.getBigDecimal(position);
        return value != null ? new Money(value) : null;
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Money value, int index, SharedSessionContractImplementor session) throws SQLException {
        st.setBigDecimal(index, value != null ? value.getAmount() : null);
    }
    // ... 其他方法
}
```

## 9.6 Hibernate Envers 的工作原理？

**Q: Hibernate Envers 的工作原理？**

Hibernate Envers 是一个审计（Audit）模块，自动记录实体的历史变更。

```java
@Entity
@Audited  // 对该实体启用审计
public class Product {
    @Id private Long id;

    @Column(name = "product_name")
    private String name;

    private BigDecimal price;

    @NotAudited  // 排除此字段不审计
    private String internalNote;
}

// Envers 自动创建 PRODUCT_AUD 审计表
// PRODUCT_AUD(id, REV, REVTYPE, name, price)
// REV: 修订号（全球递增）
// REVTYPE: 0=ADD, 1=MOD, 2=DELETE
```

**查询审计历史**：
```java
// 查询某个时间点的快照
AuditReader reader = AuditReaderFactory.get(session);
Product oldVersion = reader.find(Product.class, productId, revisionNumber);

// 查询某实体的所有历史版本
List<Number> revisions = reader.getRevisions(Product.class, productId);

// 查询指定版本的变更
List<Object[]> modifiedEntities = reader.createQuery()
    .forRevisionsOfEntity(Product.class, true, true)
    .getResultList();
```

**Envers 的核心表**：
- `REVINFO`：存储每次修订的元信息（修订号、时间戳、用户等）。
- `*_AUD`：每个被 `@Audited` 实体的审计表（字段与原表一致 + `REV` + `REVTYPE`）。

**工作原理**：在 insert/update/delete 时，通过 Hibernate 的事件监听器（PreInsert/PreUpdate/PreDelete）向审计表插入历史记录。

## 9.7 Hibernate 6 有哪些重要变化？

**Q: Hibernate 6 有哪些重要变化？**

1. **Jakarta EE 命名空间**：包名从 `javax.persistence.*` 迁移到 `jakarta.persistence.*`。

2. **语义查询模型（SQM）**：全新的 HQL/Criteria 编译器，性能更好，错误提示更清晰。
   ```java
   // Hibernate 6 Criteria 更简洁
   CriteriaBuilder cb = em.getCriteriaBuilder();
   CriteriaQuery<User> cq = cb.createQuery(User.class);
   Root<User> root = cq.from(User.class);
   cq.select(root).where(cb.like(root.get("name"), "%张%"));
   ```

3. **更好的 SQL 类型系统**：引入 `@JdbcTypeCode`，更精确地映射 JDBC 类型。
   ```java
   @JdbcTypeCode(SqlTypes.JSON)  // 替代 @Type(type = "jsonb")
   private JsonNode metadata;
   ```

4. **UUID 主键生成**：内置 `@GeneratedValue(strategy = GenerationType.UUID)`。

5. **HBM XML 被替代**：Java 注解为首选映射方式。

6. **新的分页 API**：`@SQLRestriction` 替代 `@Where`，`@Filter` 改进。

7. **to-one 懒加载改进**：不再需要字节码增强即可实现可靠的 `@OneToOne` 懒加载。

8. **时间映射**：原生支持 `Instant`、`Duration`、`YearMonth` 等。

9. **批量操作**：INSERT/UPDATE/DELETE 在 HQL 中已支持（以前只能用原生 SQL）。

10. **移除 Session.save() / update()** 等老旧 API（6.0 仍保留但 deprecated，6.3+ 移入 Legacy API）。

# 10. 设计与实战面试题

## 10.1 如何设计一个通用的审计字段？

**Q: 如何设计一个通用的审计字段？**

```java
// 方案一：@MappedSuperclass
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @CreatedDate
    @Column(updatable = false)
    private Instant createdDate;

    @LastModifiedBy
    private String lastModifiedBy;

    @LastModifiedDate
    private Instant lastModifiedDate;

    @Version
    private Long version;
}

@Entity
public class Order extends BaseEntity {
    // 业务字段
}
```

```java
// 配置 Spring Data JPA 审计
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext()
            .getAuthentication().getName());
    }
}
```

```java
// 方案二：@Embeddable — 更灵活，可嵌入任何实体
@Embeddable
public class AuditFields {
    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    private String createdBy;
    private String updatedBy;
}

@Entity
public class Order {
    @Embedded
    private AuditFields audit = new AuditFields();  // 默认值避免 NPE
}
```

## 10.2 大数据量导入如何优化？

**Q: 大数据量导入如何优化？**

```java
@Transactional
public void bulkImport(List<User> users, int batchSize) {
    // 1. 使用无状态 Session
    StatelessSession session = sessionFactory.openStatelessSession();
    Transaction tx = session.beginTransaction();

    int count = 0;
    for (User user : users) {
        session.insert(user);
        count++;
        if (count % batchSize == 0) {
            session.getTransaction().commit();
            session.getTransaction().begin();  // 分段提交，避免长事务
        }
    }
    tx.commit();
    session.close();
}
```

**优化措施清单**：

1. **关闭验证**：`hibernate.javax.persistence.validation.mode=none`
2. **增大批处理**：`hibernate.jdbc.batch_size=200`
3. **排序 INSERT**：`hibernate.order_inserts=true`
4. **关闭二级缓存**：`hibernate.cache.use_second_level_cache=false`
5. **关闭 SQL 日志**：`hibernate.show_sql=false`
6. **使用无主键预检查**：先用 SQL 查询已存在 ID，只插入新数据
7. **使用 JDBC 直接的批量 API**（最快）：
   ```java
   jdbcTemplate.batchUpdate("INSERT INTO t_user (name, email) VALUES (?, ?)",
       users, batchSize, (ps, user) -> {
           ps.setString(1, user.getName());
           ps.setString(2, user.getEmail());
       });
   ```
8. **数据库配置**：MySQL `rewriteBatchedStatements=true`、PostgreSQL `reWriteBatchedInserts=true`

## 10.3 数据库迁移（Schema Evolution）如何管理？

**Q: 数据库迁移（Schema Evolution）如何管理？**

推荐使用版本化的数据库迁移工具，而非依赖 Hibernate 的 `ddl-auto`。

1. **Flyway**（推荐，简单易用）：
   ```sql
   -- V1__init.sql
   CREATE TABLE t_user (id BIGINT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));

   -- V2__add_email.sql
   ALTER TABLE t_user ADD COLUMN email VARCHAR(100);
   ```
   ```properties
   spring.flyway.enabled=true
   spring.jpa.hibernate.ddl-auto=validate  # 验证而非自动创建
   ```

2. **Liquibase**（功能更丰富，支持回滚）：
   ```xml
   <changeSet id="1" author="dev">
       <createTable tableName="t_user">
           <column name="id" type="BIGINT" autoIncrement="true">
               <constraints primaryKey="true"/>
           </column>
           <column name="name" type="VARCHAR(50)"/>
       </createTable>
   </changeSet>
   ```

**Hibernate ddl-auto 生产环境设置**：
| 值 | 行为 | 生产环境 |
|----|------|---------|
| `create` | 每次启动重建表（数据丢失） | 禁止 |
| `create-drop` | 启动创建、关闭删除 | 禁止 |
| `update` | 自动更新表结构 | 不推荐（不可控） |
| `validate` | 校验映射与表结构一致性 | 推荐 |
| `none` | 不做任何操作 | 配合 Flyway 使用 |

## 10.4 Hibernate 项目分层架构建议？

**Q: Hibernate 项目分层架构建议？**

```
Controller/Resource 层
    ↓ DTO (数据传输对象)
Service/Facade 层 (@Transactional)
    ↓ Entity ↔ DTO 转换
Repository/DAO 层 (Spring Data JPA)
    ↓ Entity (JPA 实体)
Database
```

**关键原则**：
1. **Entity 不出 Service 层**：实体不应暴露到 Controller 或序列化到 JSON（使用 DTO）。
2. **Repository 接口继承 Spring Data JPA**：
   ```java
   @Repository
   public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
       @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
       Optional<User> findByIdWithOrders(@Param("id") Long id);
   }
   ```
3. **Service 层控制事务边界**：所有数据操作在一个 `@Transactional` 中完成。
   ```java
   @Service
   @Transactional
   public class OrderService {
       public OrderDTO createOrder(OrderRequest req) {
           // 业务逻辑在事务内完成
           Order order = new Order(...);
           return OrderDTO.from(orderRepository.save(order));
       }
   }
   ```
4. **读写分离**：读操作 `@Transactional(readOnly = true)`，写操作默认。
5. **避免在 Controller 层直接使用 EntityManager/Session**。
6. **使用 MapStruct 或 ModelMapper 做 Entity ↔ DTO 转换，避免手动 set/get**。

## 10.5 如何避免 LazyInitializationException？

**Q: 如何避免 LazyInitializationException？**

`LazyInitializationException` 发生在 Session 已关闭后访问懒加载属性时。

**解决方案**（按推荐程度排序）：

1. **DTO 投影**（最佳）— Service 层内完成所有数据加载并返回 DTO：
   ```java
   @Transactional(readOnly = true)
   public OrderDTO getOrder(Long id) {
       Order order = orderRepository.findById(id).orElseThrow();
       return new OrderDTO(order);  // 在事务内完成所有访问
   }
   ```

2. **JOIN FETCH / @EntityGraph** — 在查询时立即加载所需关联：
   ```java
   @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
   Optional<Order> findByIdWithItems(@Param("id") Long id);
   ```

3. **Open Session in View** — 延长 Session 到视图层（不推荐生产环境）：
   ```properties
   spring.jpa.open-in-view=true
   ```

4. **Hibernate.initialize()** — 手动初始化懒加载属性：
   ```java
   @Transactional
   public User getUser(Long id) {
       User user = userRepository.findById(id).orElseThrow();
       Hibernate.initialize(user.getOrders());
       return user;
   }
   ```

5. **@NamedEntityGraph** — 在实体类上预定义获取策略。

## 10.6 生产环境中 ddl-auto 应该设置为什么？

**Q: 生产环境中 ddl-auto 应该设置为什么？**

**强烈推荐：`validate` 或 `none`**。

```properties
# 生产环境
spring.jpa.hibernate.ddl-auto=validate  # 校验映射与数据库结构一致

# 配合 Flyway 管理 Schema
spring.jpa.hibernate.ddl-auto=none
spring.flyway.enabled=true
```

| 设置 | 风险 |
|------|------|
| `create` / `create-drop` | 启动时**删除并重建所有表**，生产灾难 |
| `update` | Hibernate 自动 ALTER TABLE，可能不预期地修改表结构；删除列不会同步 |

**`validate` 的优点**：
- 启动时发现映射与数据库不一致，立即失败（快速失败原则）。
- 不会执行任何 DDL 操作，不会意外修改数据库结构。
- 配合 CI/CD 管道的迁移脚本，确保 DBA 审查和版本化管理。

## 10.7 连接池大小如何确定？

**Q: 连接池大小如何确定？**

**经典公式**（来源：《HikariCP 文档》和 PostgreSQL 文档）：

```
connections = ((core_count * 2) + effective_spindle_count)
```

- `core_count`：CPU 核心数
- `effective_spindle_count`：磁盘数（SSD 近似为 0-1）

**经验值**：

| CPU 核心数 | 推荐连接池大小 |
|-----------|--------------|
| 4 核 | 10 ~ 20 |
| 8 核 | 16 ~ 32 |
| 16 核 | 20 ~ 40 |
| 32 核 | 30 ~ 60 |

```properties
# 基于 HikariCP 配置
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.connection-timeout=20000
spring.datasource.hikari.max-lifetime=1800000
```

**关键原则**：
- 连接数不是越大越好：64+ 连接时数据库调度开销显著增加。
- 主要瓶颈在磁盘 I/O 而非网络。
- 监控指标：`active_connections`、`pending_connections`、`connection_timeout_count`。

**Hibernate 相关连接池参数**：
```properties
hibernate.connection.provider_disables_autocommit=true  # 连接池管理事务
hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
```

# 11. 代码题

## 11.1 订单-订单项-商品实体设计

**Q: 给一个场景，要求写实体和关联映射代码（如订单-订单项-商品）**

```java
@Entity
@Table(name = "t_order")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String orderNo;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("id ASC")
    private List<OrderItem> items = new ArrayList<>();

    @Column(nullable = false)
    private BigDecimal totalAmount = BigDecimal.ZERO;

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    @CreationTimestamp
    private Instant createdAt;

    @Version
    private Long version;

    // 辅助方法 — 双向维护
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }

    public void calculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

```java
@Entity
@Table(name = "t_order_item")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false)
    private BigDecimal unitPrice;

    @Column(nullable = false)
    private BigDecimal subtotal;

    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

```java
@Entity
@Table(name = "t_product")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;

    @Version
    private Long version;
}
```

```java
// DTO（用于接口返回）
public class OrderDTO {
    private Long id;
    private String orderNo;
    private Long userId;
    private String userName;
    private BigDecimal totalAmount;
    private String status;
    private List<OrderItemDTO> items;
    private Instant createdAt;

    public static OrderDTO from(Order order) {
        OrderDTO dto = new OrderDTO();
        dto.id = order.getId();
        dto.orderNo = order.getOrderNo();
        dto.userId = order.getUser().getId();
        dto.userName = order.getUser().getName();
        dto.totalAmount = order.getTotalAmount();
        dto.status = order.getStatus().name();
        dto.items = order.getItems().stream().map(OrderItemDTO::from).toList();
        dto.createdAt = order.getCreatedAt();
        return dto;
    }
}

@Data
@AllArgsConstructor
public class OrderItemDTO {
    private Long productId;
    private String productName;
    private Integer quantity;
    private BigDecimal unitPrice;
    private BigDecimal subtotal;

    public static OrderItemDTO from(OrderItem item) {
        return new OrderItemDTO(
            item.getProduct().getId(),
            item.getProduct().getName(),
            item.getQuantity(),
            item.getUnitPrice(),
            item.getSubtotal()
        );
    }
}
```

## 11.2 N+1 问题优化

**Q: 给一个 N+1 问题的代码，要求优化**

```java
// ------ 问题代码（N+1）------
@Transactional(readOnly = true)
public List<User> findActiveUsersWithOrders() {
    List<User> users = em.createQuery(
        "SELECT u FROM User u WHERE u.status = :status", User.class)
        .setParameter("status", Status.ACTIVE)
        .getResultList();  // 1 次查询，假设返回 100 条

    for (User user : users) {
        // 每次循环触发 1 次 SELECT（懒加载）
        List<Order> orders = user.getOrders();  // 触发第 1 次：SELECT ... FROM t_order WHERE user_id = ?
        for (Order order : orders) {
            System.out.println(order.getOrderNo());
        }
    }
    // 总计：1 + 100 = 101 次 SQL（N+1 问题）
    return users;
}
```

```java
// ------ 优化方案一：JOIN FETCH ------
@Transactional(readOnly = true)
public List<User> findActiveUsersWithOrders() {
    return em.createQuery(
        "SELECT DISTINCT u FROM User u " +
        "LEFT JOIN FETCH u.orders " +
        "WHERE u.status = :status", User.class)
        .setParameter("status", Status.ACTIVE)
        .setHint(QueryHints.PASS_DISTINCT_THROUGH, false)
        .getResultList();
    // 1 次 SQL：SELECT DISTINCT u.*, o.* FROM t_user u LEFT JOIN t_order o ...
}
```

```java
// ------ 优化方案二：@BatchSize ------
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    @BatchSize(size = 50)  // 批量加载
    private List<Order> orders;
}
// 结果：1 次 User 查询 + 100/50 = 2 次 Order 查询 = 3 次 SQL

// ------ 优化方案三：DTO 投影（不关心 Order 详情时）------
public record UserSummary(Long id, String name, Long orderCount) {}

@Transactional(readOnly = true)
public List<UserSummary> findActiveUserSummaries() {
    return em.createQuery(
        "SELECT new com.example.UserSummary(u.id, u.name, COUNT(o)) " +
        "FROM User u LEFT JOIN u.orders o " +
        "WHERE u.status = :status GROUP BY u.id, u.name", UserSummary.class)
        .setParameter("status", Status.ACTIVE)
        .getResultList();
    // 1 次 SQL，只加载必要字段
}

// ------ 优化方案四：@EntityGraph ------
@EntityGraph(attributePaths = {"orders"})
@Query("SELECT u FROM User u WHERE u.status = :status")
List<User> findActiveUsersWithOrdersGraph(@Param("status") Status status);
```

## 11.3 Criteria API 动态查询

**Q: 给一个 Criteria API 动态查询需求，要求实现**

需求：实现一个商品搜索接口，支持按名称（模糊）、价格区间、分类、状态多条件组合筛选，支持分页和排序。

```java
@Service
@Transactional(readOnly = true)
public class ProductSearchService {

    @Autowired
    private EntityManager em;

    public Page<ProductDTO> search(ProductSearchCriteria criteria, Pageable pageable) {
        CriteriaBuilder cb = em.getCriteriaBuilder();

        // 查询数据
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);
        cq.select(root);

        // 动态构建查询条件
        List<Predicate> predicates = buildPredicates(criteria, cb, root);
        cq.where(predicates.toArray(new Predicate[0]));

        // 排序
        if (pageable.getSort().isSorted()) {
            pageable.getSort().forEach(order -> {
                Path<Object> field = root.get(order.getProperty());
                cq.orderBy(order.isAscending() ? cb.asc(field) : cb.desc(field));
            });
        }

        // 分页查询
        List<Product> products = em.createQuery(cq)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();

        // 计数查询
        Long total = getTotalCount(criteria, cb);

        return new PageImpl<>(
            products.stream().map(ProductDTO::from).toList(),
            pageable,
            total
        );
    }

    private List<Predicate> buildPredicates(ProductSearchCriteria criteria,
                                             CriteriaBuilder cb, Root<Product> root) {
        List<Predicate> predicates = new ArrayList<>();

        // 默认不查已删除
        predicates.add(cb.equal(root.get("deleted"), false));

        // 名称模糊搜索
        if (criteria.getName() != null && !criteria.getName().isBlank()) {
            predicates.add(cb.like(
                cb.lower(root.get("name")),
                "%" + criteria.getName().toLowerCase() + "%"
            ));
        }

        // 价格区间
        if (criteria.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), criteria.getMinPrice()));
        }
        if (criteria.getMaxPrice() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), criteria.getMaxPrice()));
        }

        // 分类筛选
        if (criteria.getCategoryIds() != null && !criteria.getCategoryIds().isEmpty()) {
            Join<Product, Category> categoryJoin = root.join("category");
            predicates.add(categoryJoin.get("id").in(criteria.getCategoryIds()));
        }

        // 状态筛选
        if (criteria.getStatus() != null) {
            predicates.add(cb.equal(root.get("status"), criteria.getStatus()));
        }

        // 库存阈值
        if (criteria.getMinStock() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("stock"), criteria.getMinStock()));
        }

        // 创建时间范围
        if (criteria.getStartDate() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("createdAt"), criteria.getStartDate()));
        }
        if (criteria.getEndDate() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("createdAt"), criteria.getEndDate()));
        }

        return predicates;
    }

    private Long getTotalCount(ProductSearchCriteria criteria, CriteriaBuilder cb) {
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<Product> root = countQuery.from(Product.class);
        countQuery.select(cb.count(root));

        List<Predicate> predicates = buildPredicates(criteria, cb, root);
        countQuery.where(predicates.toArray(new Predicate[0]));

        return em.createQuery(countQuery).getSingleResult();
    }
}
```

```java
// 搜索条件 DTO
@Data
@Builder
public class ProductSearchCriteria {
    private String name;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private List<Long> categoryIds;
    private ProductStatus status;
    private Integer minStock;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
}

// 调用示例
ProductSearchCriteria criteria = ProductSearchCriteria.builder()
    .name("手机")
    .minPrice(new BigDecimal("1000"))
    .maxPrice(new BigDecimal("5000"))
    .status(ProductStatus.ON_SALE)
    .build();

Pageable pageable = PageRequest.of(0, 20, Sort.by(Sort.Direction.DESC, "createdAt"));
Page<ProductDTO> result = productSearchService.search(criteria, pageable);
```

---

> **参考资源**：
> - Hibernate 官方文档：https://hibernate.org/orm/documentation/
> - JPA 2.2 规范：https://jakarta.ee/specifications/persistence/
> - Vlad Mihalcea 博客：https://vladmihalcea.com/
> - Thorben Janssen 博客（Thoughts on Java）：https://thorben-janssen.com/
> - 性能优化指南：https://docs.jboss.org/hibernate/orm/6.6/topical/html_single/performance/Performance.html
