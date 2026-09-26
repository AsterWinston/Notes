
# 1. 概述

## 1.1 什么是 JPA

JPA（Java Persistence API，Java 持久化 API）是 JavaEE 的 **ORM（对象关系映射）标准规范**，统一了 Java 领域对象与关系数据库表之间的映射、CRUD 与查询方式。

- 参考实现（RI）：**EclipseLink**
- 最流行实现：**Hibernate**（参见 `DataMiddleware/Hibernate` 笔记）
- 其他实现：OpenJPA、Apache OpenJPA

## 1.2 JPA 解决的问题

```
传统 JDBC：
  写 SQL + ResultSet 手动映射到 Java 对象 + 手动管理批量操作...

JPA：
  操作 Java 对象，框架自动生成 SQL 并完成映射
```

## 1.3 JPA 与 Hibernate 的关系

```
JPA（规范，接口 + 注解 + JPQL）
   ▲
   │ 实现
Hibernate（具体实现，额外提供更丰富的能力）
```

- 面向 JPA 标准编程可以保持可移植性。
- Hibernate 在 JPA 之上提供了二级缓存、原生 SQL、Session API 等增强能力。

# 2. JPA 核心组成

| 组件 | 说明 |
|------|------|
| `Entity` | 持久化实体类（POJO + `@Entity`） |
| `EntityManager` | 实体管理器：CRUD 入口（Session/Transaction 管理） |
| `EntityManagerFactory` | EntityManager 工厂（应用级单例，对应持久化单元） |
| `persistence.xml` | 持久化单元配置文件 |
| JPQL | 面向对象的查询语言 |
| Criteria API | 类型安全查询 |
| `@PersistenceContext` | 容器注入 EntityManager 的注解 |

# 3. 持久化单元（Persistence Unit）

## 3.1 persistence.xml

```xml
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
                                 http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">
    <persistence-unit name="defaultPU" transaction-type="JTA">
        <jta-data-source>java:comp/env/jdbc/MyDS</jta-data-source>
        <class>com.example.User</class>
        <properties>
            <property name="jakarta.persistence.schema-generation.database.action"
                      value="update"/>
        </properties>
    </persistence-unit>
</persistence>
```

| 事务类型 | 说明 |
|----------|------|
| `JTA` | 容器管理事务（JavaEE 环境） |
| `RESOURCE_LOCAL` | 应用管理事务（JavaSE / 独立运行） |

## 3.2 获取 EntityManager

**JavaEE（容器注入，推荐）**

```java
@Stateless
public class UserService {
    @PersistenceContext
    private EntityManager em;
}
```

**JavaSE（工厂获取）**

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("defaultPU");
EntityManager em = emf.createEntityManager();
```

# 4. 实体映射

## 4.1 基础映射注解

```java
@Entity
@Table(name = "t_user")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_name", nullable = false, length = 50)
    private String name;

    @Column(precision = 10, scale = 2)
    private BigDecimal salary;

    @Temporal(TemporalType.TIMESTAMP)
    private Date createdAt;

    @Enumerated(EnumType.STRING)
    private UserStatus status;

    @Lob
    private String description;

    // getter/setter...
}
```

## 4.2 主键生成策略

| 策略 | 说明 |
|------|------|
| `IDENTITY` | 数据库自增 |
| `SEQUENCE` | 数据库序列（Oracle、PostgreSQL） |
| `TABLE` | 模拟序列表 |
| `AUTO` | 由持久化提供者选择 |
| `UUID` | UUID 字符串（Hibernate 扩展） |

## 4.3 关联映射

| 关系 | 注解 |
|------|------|
| 一对一 | `@OneToOne` |
| 一对多 | `@OneToMany` |
| 多对一 | `@ManyToOne` |
| 多对多 | `@ManyToMany` |

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "dept_id")
private Department department;

@OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Employee> employees = new ArrayList<>();
```

> 详细映射与继承策略见 `DataMiddleware/Hibernate` 笔记。

# 5. EntityManager 的 CRUD

| 方法 | 作用 |
|------|------|
| `persist(entity)` | 持久化（INSERT） |
| `find(Class, id)` / `getReference(Class, id)` | 查询（按主键） |
| `merge(entity)` | 合并（UPDATE / 分离实体合并） |
| `remove(entity)` | 删除 |
| `flush()` | 强制同步到数据库 |
| `refresh(entity)` | 用数据库刷新实体 |
| `detach(entity)` | 分离实体（脱离持久化上下文） |
| `clear()` | 清空持久化上下文 |
| `contains(entity)` | 是否受管理 |
| `createQuery / createNamedQuery / createNativeQuery` | 查询 |

```java
public User findById(Long id) {
    return em.find(User.class, id);
}

public User save(User user) {
    if (user.getId() == null) {
        em.persist(user);        // 新增
    } else {
        user = em.merge(user);   // 更新
    }
    return user;
}

public void delete(Long id) {
    User user = em.find(User.class, id);
    if (user != null) {
        em.remove(user);
    }
}
```

# 6. 实体生命周期与状态

## 6.1 四种状态

```
                         persist()
              ┌──────────────────────────┐
              │                          ▼
   new(瞬时)                          managed(受管)
              │                          │
              └──────── merge() ◄────────┘
                      │                  │ 事务提交/clear/detach
                      ▼                  ▼
                  detached(游离)     removed(删除)  ◄── remove() 从 managed 进入
```

| 状态 | 说明 |
|------|------|
| **new（瞬时）** | 新建对象，未与 EntityManager 关联 |
| **managed（受管）** | 已关联持久化上下文，变更会被跟踪并同步 |
| **detached（游离）** | 曾关联但上下文关闭/分离，变更不再自动同步 |
| **removed（删除）** | 已标记删除，事务提交时执行 DELETE |

## 6.2 持久化上下文（Persistence Context）

- 是 JPA 的**一级缓存**（Entity 缓存）。
- 一个持久化上下文维护实体实例与数据库记录的对应。
- 同一事务中 `find` 同一条记录返回同一实例（同一性保证）。
- Hibernate 中即 `Session`。

# 7. JPQL（Java Persistence Query Language）

面向实体对象（而非表）的查询语言：

```java
// 查询所有
TypedQuery<User> q = em.createQuery(
        "SELECT u FROM User u WHERE u.status = :status", User.class);
q.setParameter("status", UserStatus.ACTIVE);
List<User> users = q.getResultList();

// 分页
q.setFirstResult(0);
q.setMaxResults(10);

// 聚合
Long count = em.createQuery("SELECT COUNT(u) FROM User u", Long.class)
        .getSingleResult();

// 命名查询
@NamedQuery(name = "User.findByName",
        query = "SELECT u FROM User u WHERE u.name = :name")
```

## 7.1 JPQL vs SQL

| 对比 | JPQL | SQL |
|------|------|-----|
| 操作对象 | 实体对象 / 字段 | 表 / 列 |
| 大小写敏感 | 关键字不敏感，实体名/属性名敏感 | 不敏感 |
| 跨数据库 | 可移植 | 依赖具体数据库方言 |
| 关联查询 | 通过对象属性导航 | 显式 JOIN |

## 7.2 原生 SQL

```java
Query query = em.createNativeQuery("SELECT * FROM t_user WHERE age > ?1", User.class);
query.setParameter(1, 18);
List<User> users = query.getResultList();
```

# 8. Criteria API（类型安全查询）

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);

Predicate agePred = cb.greaterThan(root.get("age"), 18);
Predicate statusPred = cb.equal(root.get("status"), UserStatus.ACTIVE);
cq.select(root).where(cb.and(agePred, statusPred));

List<User> users = em.createQuery(cq).getResultList();
```

- 类型安全：字段名以方法引用方式访问，编译期可检查。
- 适合动态条件构建（MyBatis-Plus 的 LambdaQueryWrapper 也是类似思路）。

# 9. 级联与加载策略

## 9.1 级联操作（Cascade）

| 级联类型 | 说明 |
|----------|------|
| `PERSIST` | 保存父时保存子 |
| `MERGE` | 合并父时合并子 |
| `REMOVE` | 删除父时删除子 |
| `REFRESH` | 刷新父时刷新子 |
| `DETACH` | 分离父时分离子 |
| `ALL` | 以上全部 |

## 9.2 加载策略（Fetch）

| 策略 | 说明 | 适用 |
|------|------|------|
| `LAZY`（默认 for 集合） | 延迟加载，使用时才查询 | 一对多/多对多 |
| `EAGER`（默认 for 单值） | 立即加载 | 多对一/一对一 |

- 过度使用 `EAGER` 会导致 N+1 查询。
- 延迟加载在事务外访问会抛 `LazyInitializationException`（Hibernate）。

# 10. 乐观锁与悲观锁

## 10.1 乐观锁

```java
@Entity
public class Account {
    @Id
    private Long id;

    @Version
    private long version;   // 版本号，每次更新自动 +1

    private BigDecimal balance;
}
```

- 更新时比较版本号，不匹配抛 `OptimisticLockException`。
- 适合并发冲突较少的场景。

## 10.2 悲观锁

```java
User user = em.find(User.class, id, LockModeType.PESSIMISTIC_WRITE);
// 或 em.lock(user, LockModeType.PESSIMISTIC_WRITE);
```

- 数据库行级锁，适合并发冲突严重或跨实体事务。

# 11. JPA 与 JTA 结合

- JavaEE 容器中默认使用 JTA 事务，JPA 自动参与容器事务。
- 无事务时 JPA 操作可能报错（需要事务），事务提交时自动 `flush`。
- 事务回滚时持久化上下文状态被丢弃，需重建。

# 12. JPA 性能要点

| 问题 | 解决方案 |
|------|----------|
| N+1 查询 | 使用 `JOIN FETCH`、`@NamedEntityGraph`、`@BatchSize` |
| 大批量插入 | `em.flush(); em.clear();` 分批执行 |
| 全字段加载 | 使用投影（DTO 查询）只取所需字段 |
| 大结果集 | 游标/流式查询 `getResultStream()` |
| 缓存 | 二级缓存（`@Cacheable`）用于只读热点数据 |

# 13. 与 MyBatis / Spring Data JPA 对照

| 对比 | JPA/Hibernate | MyBatis | Spring Data JPA |
|------|---------------|---------|-----------------|
| SQL 控制 | JPQL/Criteria 自动生成 | 完全手写 SQL | 基于 JPA 自动生成 |
| 灵活度 | 中 | 高 | 中（可用 @Query 补充） |
| 学习成本 | 中高 | 低 | 中 |
| 缓存 | 一级/二级缓存 | 一级缓存 + 需集成第三方 | 继承 JPA 能力 |
| 适用 | 快速 CRUD、标准 ORM | 复杂 SQL、报表、SQL 优化要求高 | JPA + Repository 快速开发 |
