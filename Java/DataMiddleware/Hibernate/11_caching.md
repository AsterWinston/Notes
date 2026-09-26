# 1. 缓存概述

## 1.1 为什么需要缓存

在企业应用中，数据库往往是系统性能的最大瓶颈。每一次数据库访问都涉及网络 I/O、磁盘 I/O、SQL 解析与执行等开销。缓存通过在应用与数据库之间建立一道"缓冲层"，将常用数据暂存于内存中，从而：

- **减少数据库访问**：高频读取的数据直接从内存获取，避免重复查询数据库
- **降低 SQL 执行开销**：减少 SQL 解析、编译、执行的次数
- **降低网络 I/O**：应用与数据库之间的网络往返次数大幅减少
- **提升应用响应速度**：内存读取速度远高于磁盘/网络读取（纳秒级 vs 毫秒级）

典型的缓存分层架构如下：

```
┌─────────────────────────────────────────┐
│              应用层缓存                    │
│   (Spring Cache / Caffeine / Redis)      │
│   业务语义缓存，手动控制，粒度灵活            │
├─────────────────────────────────────────┤
│           二级缓存 (SessionFactory)        │
│   (Ehcache / Hazelcast / Redis)          │
│   跨 Session，按实体类型/ID 组织            │
├─────────────────────────────────────────┤
│           一级缓存 (Session)               │
│   (Persistence Context)                  │
│   Session 级别，自动管理，事务隔离           │
├─────────────────────────────────────────┤
│              数据库                        │
│   (MySQL / PostgreSQL / Oracle)          │
│   持久化存储，数据权威来源                    │
└─────────────────────────────────────────┘
```

每一层缓存都有其独特的职责和生命周期。上层缓存粒度更粗、生命周期更短；下层缓存更贴近数据源，提供最终一致性保障。

## 1.2 Hibernate 缓存层次

Hibernate 提供了三个层次的缓存机制：

| 缓存层次 | 作用域 | 是否默认启用 | 配置复杂度 | 跨 Session 共享 |
|---------|--------|------------|-----------|----------------|
| 一级缓存（Persistence Context） | Session | 是（内置） | 无需配置 | 否 |
| 二级缓存（Second-Level Cache） | SessionFactory | 否 | 中等 | 是 |
| 查询缓存（Query Cache） | SessionFactory | 否 | 中等 | 是（依赖二级缓存） |

**一级缓存**是 Hibernate 内置的，每个 Session 维护自己的 PersistenceContext，自动管理实体状态。它是 Hibernate 脏检查、延迟写入等机制的基础。

**二级缓存**是 SessionFactory 级别的全局缓存，跨 Session 共享。需要通过配置显式启用，并选择合适的缓存提供者（如 Ehcache、Hazelcast、Redis 等）。

**查询缓存**缓存 HQL/JPQL/Criteria 查询的结果集（仅存储 ID 列表），必须依赖二级缓存才能真正减少数据库访问。查询缓存按查询语句和参数作为 key，任何涉及的表发生数据变更都会导致相关查询缓存失效。

# 2. 一级缓存（Persistence Context）

## 2.1 一级缓存原理

每个 Hibernate Session 内部维护一个 **PersistenceContext**（持久化上下文），这是 Hibernate 一级缓存的核心数据结构。PersistenceContext 本质上是一个 `Map<EntityKey, Object>`，其中：

- **EntityKey** 由 **实体类型 + @Id 值** 组合而成
- **Value** 是实体对象的引用（即 Java 对象本身）

当调用 `session.find()`、`session.get()` 或 `session.byId().load()` 等方法时，Hibernate 的执行流程如下：

```
1. 计算 EntityKey = (实体Class, id值)
2. 查找 PersistenceContext Map 中是否存在该 EntityKey
   ├─ 存在 → 直接返回缓存的实体引用（不执行 SQL）
   └─ 不存在 → 执行 SQL 查询数据库
              → 将结果组装为实体对象
              → 放入 PersistenceContext Map
              → 返回实体引用
```

**示例——同一 Session 内重复查询只执行一次 SQL：**

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

// 第一次查询 → 执行 SQL SELECT
Book book1 = session.find(Book.class, 1L);
System.out.println("--- 第一次 find 结束 ---");

// 第二次查询相同 ID → 不执行 SQL，直接从一级缓存返回
Book book2 = session.find(Book.class, 1L);
System.out.println("--- 第二次 find 结束 ---");

// book1 和 book2 是同一个对象引用
System.out.println(book1 == book2); // true

tx.commit();
session.close();
```

控制台输出（仅一次 SQL）：

```sql
Hibernate: select b1_0.id,b1_0.title,b1_0.price from book b1_0 where b1_0.id=?
--- 第一次 find 结束 ---
--- 第二次 find 结束 ---
true
```

## 2.2 一级缓存的作用域

| 操作 | 影响范围 | 说明 |
|------|---------|------|
| `session.close()` | 整个 PersistenceContext | Session 关闭时一级缓存完全清空 |
| `session.clear()` | 整个 PersistenceContext | 手动清空，所有托管实体变为游离态 |
| `session.evict(entity)` | 单个实体 | 从 PersistenceContext 中移除指定实体 |
| `session.flush()` | 不清理缓存 | 仅将脏数据同步到数据库，缓存仍保留 |

**作用域限制**：一级缓存严格限制在单个 Session 内，不同 Session 之间的一级缓存是完全隔离的：

```java
// Session 1
Session session1 = sessionFactory.openSession();
Book book1 = session1.find(Book.class, 1L); // 执行 SQL
session1.close();

// Session 2 —— 即使是同一个 ID，仍需重新查询
Session session2 = sessionFactory.openSession();
Book book2 = session2.find(Book.class, 1L); // 再次执行 SQL
session2.close();
// book1 和 book2 是不同的对象引用（且前一个已脱离 Session 管理）
```

## 2.3 一级缓存的行为特征

### 重复读取优化

同一 Session 内对同一 ID 的多次读取不会重复执行 SQL。这包括 `find()`、`get()`、`load()` 以及通过关联导航加载的实体。

### 写后延迟（Write Behind）

Hibernate 不会在每次 `session.persist()` 或属性修改后立即执行 SQL。而是将变更累积在 PersistenceContext 中，在以下时机统一同步到数据库：

- `transaction.commit()` 前自动调用 `flush()`
- 显式调用 `session.flush()`
- 执行某些查询（HQL/JPQL）前（Hibernate 5 默认 `FlushMode.AUTO`）
- 执行 `session.createNativeQuery()` 且涉及被修改的表时

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

Book book = session.find(Book.class, 1L);
book.setTitle("新标题");        // 仅修改内存中的对象，不执行 SQL
book.setPrice(new BigDecimal("99.00"));
// ... 更多修改 ...

tx.commit();  // flush() → 执行 UPDATE SQL
session.close();
```

### 脏检查（Dirty Checking）

Hibernate 在 flush 时会将 PersistenceContext 中实体的当前状态与加载时的快照（Snapshot）进行比对，只有属性发生变化的实体才会生成 UPDATE 语句。这是通过 `session.flush()` → 脏检查 → 生成必要的 SQL 来实现的。

### HQL/JPQL 查询的自动 Flush

在 `FlushMode.AUTO` 模式下（默认），执行 HQL/JPQL 查询时，Hibernate 会自动先 flush PersistenceContext 中挂起的变更，以保证查询结果的一致性：

```java
session.persist(newBook); // 未 flush，新书尚未写入 DB
List<Book> books = session.createQuery("from Book", Book.class).list();
// 执行 HQL 前自动 flush → INSERT 新书 → 再执行 SELECT → 结果包含新书
```

### 批量操作绕过一级缓存

HQL 的 UPDATE/DELETE 批量操作（`executeUpdate()`）**不会经过一级缓存**，直接在数据库层面执行。这意味着：

```java
// 这不会修改一级缓存中已加载的实体
session.createMutationQuery("update Book set price = price * 1.1").executeUpdate();

// 之前加载的 book 对象仍然持有旧 price 值
Book book = session.find(Book.class, 1L); // 从一级缓存返回旧数据
```

因此，批量操作后必须调用 `session.clear()` 清空一级缓存，再重新加载数据。

## 2.4 一级缓存的问题

### 大数据量时内存溢出

当需要处理大量实体（如批量导入 10 万条记录）时，如果不加以控制，PersistenceContext 会不断膨胀直至 OOM：

```java
// 错误做法——批量导入导致 OOM
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
for (int i = 0; i < 100_000; i++) {
    Book book = new Book("Book" + i, BigDecimal.valueOf(i));
    session.persist(book); // 不断累积在一级缓存中
}
tx.commit(); // 一次性 flush 10 万条 INSERT + 内存中的 10 万个对象
session.close();
```

**正确做法——定期 flush + clear：**

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
int batchSize = 50;
for (int i = 0; i < 100_000; i++) {
    Book book = new Book("Book" + i, BigDecimal.valueOf(i));
    session.persist(book);
    if (i % batchSize == 0) {
        session.flush();  // 将累积的变更同步到数据库
        session.clear();  // 清空一级缓存，释放内存
    }
}
tx.commit();
session.close();
```

### 长 Session 持有过期数据

如果 Session 的生命周期过长（例如在 Web 应用中一个 HTTP 请求对应一个 Session 是合理的，但绝不能跨多个 HTTP 请求共用 Session），会导致一级缓存中的数据与数据库实际数据出现较大偏差。

### StatelessSession 完全不使用一级缓存

`StatelessSession` 是 Hibernate 提供的无状态 Session 实现，它：

- 不维护 PersistenceContext
- 不进行脏检查
- 不提供延迟加载
- 不提供一级缓存
- 每个操作都直接执行 SQL

```java
StatelessSession statelessSession = sessionFactory.openStatelessSession();
Transaction tx = statelessSession.beginTransaction();

// 两次查询相同 ID，都会执行 SQL
Book book1 = statelessSession.get(Book.class, 1L); // 执行 SQL
Book book2 = statelessSession.get(Book.class, 1L); // 再次执行 SQL

tx.commit();
statelessSession.close();
```

适用于批量导入、批量导出等不需要管理实体状态的场景。

# 3. 二级缓存

## 3.1 二级缓存架构

二级缓存是 **SessionFactory 级别**的全局缓存，其核心特征：

- **跨 Session 共享**：所有由同一个 SessionFactory 创建的 Session 共享同一份二级缓存数据
- **按实体类型组织**：每种实体类型对应一个独立的缓存区域（Cache Region）
- **数据结构**：`CacheKey → 缓存值`，CacheKey 由实体类型 + ID + 某些上下文信息组成

```
SessionFactory
  ├── 二级缓存
  │    ├── Region: com.example.Book
  │    │    ├── [Book#1] → {id:1, title:"Java入门", price:59}
  │    │    ├── [Book#2] → {id:2, title:"Hibernate实战", price:79}
  │    │    └── ...
  │    ├── Region: com.example.Author
  │    │    ├── [Author#1] → {id:1, name:"张三"}
  │    │    └── ...
  │    ├── Region: com.example.Department.employees  (集合缓存)
  │    │    └── ...
  │    ├── Region: org.hibernate.cache.internal.StandardQueryCache  (查询缓存)
  │    │    └── ...
  │    └── Region: org.hibernate.cache.spi.TimestampsCache  (时间戳缓存)
  │         └── ...
  ├── Session 1
  │    └── 一级缓存 (PersistenceContext)
  ├── Session 2
  │    └── 一级缓存 (PersistenceContext)
  └── Session N ...
```

**缓存区域命名规范**：

| 区域 | 命名格式 | 示例 |
|------|---------|------|
| 实体缓存 | 实体全限定类名 | `com.example.Book` |
| 集合缓存 | 实体全限定类名 + `.` + 集合属性名 | `com.example.Department.employees` |
| 查询缓存（默认） | `org.hibernate.cache.internal.StandardQueryCache` | — |
| 查询缓存（自定义） | 用户指定的区域名 | `myQueryRegion` |
| 时间戳缓存 | `org.hibernate.cache.spi.TimestampsCache` | — |

## 3.2 二级缓存的启用与配置

### 3.2.1 基础配置

在 `application.properties`（Spring Boot）或 `hibernate.properties` / `persistence.xml` 中配置：

```properties
# ========== 二级缓存开关 ==========
hibernate.cache.use_second_level_cache=true

# ========== 缓存区域工厂类 ==========
# Ehcache 2.x（Hibernate 5.x 原生支持）
hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory

# Ehcache 2.x 单例模式（非 JTA 环境推荐）
# hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory

# Ehcache 3.x / JCache（JSR-107）
# hibernate.cache.region.factory_class=jcache

# Hazelcast
# hibernate.cache.region.factory_class=com.hazelcast.hibernate.HazelcastCacheRegionFactory

# Infinispan
# hibernate.cache.region.factory_class=org.hibernate.cache.infinispan.InfinispanRegionFactory

# ========== 辅助选项 ==========
# 使用结构化缓存条目（更易于调试，但占用更多内存）
hibernate.cache.use_structured_entries=true

# 减少缓存写入：仅在缓存中不存在时才写入（避免重复写入）
hibernate.cache.use_minimal_puts=true

# 查询缓存开关
hibernate.cache.use_query_cache=true

# ========== 统计信息 ==========
hibernate.generate_statistics=true
```

在 Spring Boot 的 `application.yml` 中：

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.ehcache.EhCacheRegionFactory
          use_structured_entries: true
          use_minimal_puts: true
        generate_statistics: true
```

### 3.2.2 @Cacheable 与 @Cache

Hibernate 提供了两个层面的缓存注解：

- **`@Cacheable`**：JPA 2.0 标准注解，标记实体"可使用缓存"，但具体是否缓存由 `shared-cache-mode` 决定
- **`@Cache`**：Hibernate 扩展注解，精确定义缓存策略（`CacheConcurrencyStrategy`）

```java
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;
import jakarta.persistence.Cacheable;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;

@Entity
@Cacheable                              // JPA 标准：标记该实体可被缓存
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Hibernate 扩展：指定缓存策略
public class Book {
    @Id
    private Long id;
    private String title;
    // ...
}
```

**`shared-cache-mode` 配置**（在 `persistence.xml` 中）：

```xml
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
             http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">

    <persistence-unit name="myPU">
        <!-- 缓存模式 -->
        <shared-cache-mode>ENABLE_SELECTIVE</shared-cache-mode>

        <properties>
            <property name="hibernate.cache.use_second_level_cache" value="true"/>
            <property name="hibernate.cache.region.factory_class"
                      value="org.hibernate.cache.ehcache.EhCacheRegionFactory"/>
        </properties>
    </persistence-unit>
</persistence>
```

| shared-cache-mode | 含义 |
|-------------------|------|
| `ENABLE_SELECTIVE` | 仅 `@Cacheable(true)` 的实体启用缓存（**默认值**） |
| `DISABLE_SELECTIVE` | 除 `@Cacheable(false)` 的实体外，全部启用缓存 |
| `ALL` | 所有实体默认启用缓存 |
| `NONE` | 所有实体均不启用缓存（等同于关闭二级缓存） |
| `UNSPECIFIED` | 由提供商决定（Hibernate 行为等同于 `ENABLE_SELECTIVE`） |

## 3.3 缓存并发策略（CacheConcurrencyStrategy）

### 3.3.1 READ_ONLY

**只读缓存**，适用于永远不会被修改的数据。

- 性能最佳——无并发控制开销
- 仅适用于字典表、国家列表、常量配置等静态数据
- 任何对 READ_ONLY 实体的修改尝试都会抛出 `UnsupportedOperationException`

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id
    private String code;   // CN, US, JP...
    private String name;   // 中国, 美国, 日本...
}
```

### 3.3.2 NONSTRICT_READ_WRITE

**非严格读写缓存**，允许缓存与数据库之间存在短暂不一致。

- 适合**更新频率极低**的数据，对一致性要求不敏感的场景
- 缓存更新与数据库更新之间无严格顺序保证
- 可能出现短暂的数据"脏读"（缓存返回旧值）

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
public class ConfigParam {
    @Id
    private String key;
    private String value;
    private LocalDateTime lastModified;
}
```

**适用场景**：系统配置参数（更新频率极低），统计报告数据（可接受秒级延迟）。

### 3.3.3 READ_WRITE

**读写缓存**，最常用的缓存策略。通过"软锁"（Soft Lock）机制保证缓存与数据库的一致性。

**更新流程**：

```
1. 事务开始
2. 从缓存中读取实体
3. 修改实体属性
4. flush 时：
   a. 在缓存中为该条目设置 Soft Lock（阻止其他事务读取缓存中的旧数据）
   b. 执行 SQL UPDATE 更新数据库
   c. 提交事务后，从缓存中移除 Soft Lock（或直接更新缓存条目）
5. 事务结束
```

**Soft Lock 机制**：在事务提交前，缓存条目被标记为 Soft Lock。其他事务读取时若遇到 Soft Lock：
- 不会返回缓存中的旧数据
- 而是绕过缓存直接查询数据库
- 直到 Soft Lock 被移除

**要求**：
- 数据库事务隔离级别必须为 `READ_COMMITTED` 或更高
- 不适用于 `READ_UNCOMMITTED` 隔离级别

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;
}
```

### 3.3.4 TRANSACTIONAL

**事务型缓存**，缓存参与数据库事务，提供强一致性保证。

- 缓存操作与数据库操作在同一事务中进行（两阶段提交）
- 只有支持 JTA 事务的缓存提供者才能使用（如 Infinispan + JTA 环境）
- 性能开销最大，使用场景有限

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL)
public class Account {
    @Id
    private Long id;
    private BigDecimal balance; // 余额——必须强一致
}
```

**各策略对比总结**：

| 策略 | 一致性 | 性能 | 并发控制 | 适用场景 |
|------|--------|------|---------|---------|
| READ_ONLY | 不适用（不可写） | 最高 | 无 | 字典、静态数据 |
| NONSTRICT_READ_WRITE | 弱（最终一致） | 高 | 无 | 极少更新的配置数据 |
| READ_WRITE | 强（Soft Lock） | 中 | 有 | **最常用**，大多数实体 |
| TRANSACTIONAL | 最强 | 低 | 有（XA） | 金融、订单余额等 |

## 3.4 实体缓存配置

### 注解方式配置

```java
@Entity
@Cacheable
@Cache(
    usage = CacheConcurrencyStrategy.READ_WRITE,
    region = "bookCache",     // 自定义缓存区域名称
    include = "all"           // "all" 或 "non-lazy"
)
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;

    // getters and setters...
}
```

**`include` 属性**：

| 值 | 说明 |
|----|------|
| `"all"` | 缓存实体时包含所有属性（包括 LAZY 关联的代理对象）——**默认** |
| `"non-lazy"` | 缓存实体时跳过 LAZY 关联属性（节省缓存空间） |

**`region` 属性**：自定义区域名称，多个实体可以共享同一个缓存区域（不推荐，通常一个实体一个区域）。

### XML 方式配置

在 `hibernate.cfg.xml` 或 `persistence.xml` 中：

```xml
<!-- 实体缓存 -->
<class-cache class="com.example.Book" usage="read-write" region="bookCache"/>

<!-- 集合缓存 -->
<collection-cache collection="com.example.Department.employees" usage="read-write"/>
```

### 哪些实体适合缓存

| 适合缓存 | 不适合缓存 |
|---------|-----------|
| 频繁读取、很少修改 | 频繁修改（INSERT/UPDATE/DELETE） |
| 主数据（Master Data） | 事务流水数据（订单、日志） |
| 字典/枚举/配置数据 | 实时统计指标 |
| 数据量小、结构简单 | 大对象、BLOB/CLOB 字段 |
| 用户、部门等基础信息 | 临时/会话相关数据 |

**决策参考**：

```
读取频率 / 写入频率 > 10 → 强烈推荐缓存
读取频率 / 写入频率 在 3~10 之间 → 值得缓存
读取频率 / 写入频率 < 3 → 不建议缓存
```

## 3.5 集合缓存

**关键规则**：集合默认**不会**被缓存，即使其所属的实体已经启用了二级缓存。必须显式在集合字段上标注 `@Cache`。

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Department {
    @Id
    private Long id;
    private String name;

    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // 显式启用集合缓存
    private Set<Employee> employees = new HashSet<>();

    // getters and setters...
}
```

**集合缓存与实体缓存的关系**：

- 集合缓存存储的是**关联的 ID 集合**，而非实体对象本身
- 当访问 `department.getEmployees()` 时：
  1. 从集合缓存获取该部门关联的 Employee ID 列表
  2. 对每个 ID，尝试从二级缓存（实体缓存 Region）获取 Employee 对象
  3. 如果 Employee 的二级缓存未命中，再查询数据库

**集合缓存的失效**：当向集合添加或移除元素时，相关的集合缓存会被清除。删除集合中某个关联实体时，集合缓存也会失效。

**完整示例**——同时缓存实体和集合：

```java
// 父端
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Department {
    @Id
    private Long id;
    private String name;

    @OneToMany(mappedBy = "department")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    private List<Employee> employees = new ArrayList<>();
}

// 子端（也应启用缓存）
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Employee {
    @Id
    private Long id;
    private String name;

    @ManyToOne
    @JoinColumn(name = "dept_id")
    private Department department;
}
```

## 3.6 缓存区域管理

每个实体类型默认对应一个独立的缓存区域（Cache Region），可以对不同区域配置不同的缓存策略（容量、TTL、驱逐算法等）。

### 区域命名规则

| 区域类型 | 命名规则 | 示例 |
|---------|---------|------|
| 实体（默认） | 实体全限定类名 | `com.example.Book` |
| 实体（自定义） | `@Cache(region="xxx")` | `bookCache` |
| 集合（默认） | 全限定类名.属性名 | `com.example.Department.employees` |
| 查询缓存（默认） | `StandardQueryCache` | — |
| 查询缓存（自定义） | `query.setCacheRegion("xxx")` | `reportQueryRegion` |
| 更新时间戳 | `TimestampsCache` | — |

### Ehcache 配置示例——不同区域不同策略

**ehcache.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
         updateCheck="false">

    <!-- 默认缓存区域配置 -->
    <defaultCache
        maxEntriesLocalHeap="10000"
        eternal="false"
        timeToIdleSeconds="600"
        timeToLiveSeconds="1800"
        memoryStoreEvictionPolicy="LRU">
    </defaultCache>

    <!-- Book 实体缓存区域：读多写少，缓存 50000 条，TTL 1小时 -->
    <cache name="com.example.Book"
        maxEntriesLocalHeap="50000"
        eternal="false"
        timeToIdleSeconds="1200"
        timeToLiveSeconds="3600"
        memoryStoreEvictionPolicy="LRU">
    </cache>

    <!-- Country 实体缓存区域：只读静态数据，永不过期 -->
    <cache name="com.example.Country"
        maxEntriesLocalHeap="500"
        eternal="true"
        memoryStoreEvictionPolicy="LFU">
    </cache>

    <!-- Department.employees 集合缓存 -->
    <cache name="com.example.Department.employees"
        maxEntriesLocalHeap="10000"
        eternal="false"
        timeToIdleSeconds="600"
        timeToLiveSeconds="1800"
        memoryStoreEvictionPolicy="LRU">
    </cache>

    <!-- 查询缓存（默认区域） -->
    <cache name="org.hibernate.cache.internal.StandardQueryCache"
        maxEntriesLocalHeap="5000"
        eternal="false"
        timeToIdleSeconds="600"
        timeToLiveSeconds="1200"
        memoryStoreEvictionPolicy="LRU">
    </cache>

    <!-- 时间戳缓存：小型缓存，数据很小（仅存表名→时间戳映射），永不过期 -->
    <cache name="org.hibernate.cache.spi.TimestampsCache"
        maxEntriesLocalHeap="500"
        eternal="true">
    </cache>

    <!-- 自定义查询缓存区域：报表查询 -->
    <cache name="reportQueryRegion"
        maxEntriesLocalHeap="1000"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="600"
        memoryStoreEvictionPolicy="LRU">
    </cache>

</ehcache>
```

### 动态修改缓存区域配置

大多数缓存提供者支持通过 JMX 或管理 API 动态调整缓存配置（如 TTL、最大条目数等）。例如：

- **Ehcache**：通过 `CacheManager` MBean 动态修改
- **Hazelcast**：通过 Management Center 或 REST API
- **Infinispan**：通过 JMX 或 CLI

但通常不建议在运行时大幅调整缓存配置，应在设计阶段根据业务特征合理规划。

# 4. 查询缓存

## 4.1 查询缓存原理

查询缓存缓存 HQL/JPQL/Criteria 查询的**结果集**。与实体缓存不同，查询缓存存储的是**ID 列表**而非完整的实体对象。

**工作流程**：

```
第一次执行查询：
  ┌─────────────────────────────────────────────┐
  │ 1. 执行 SQL，获取结果集                        │
  │ 2. 提取结果集中每个实体的 ID                    │
  │ 3. 将 (查询语句+参数) → [ID1, ID2, ID3...]    │
  │    存入查询缓存区域                             │
  │ 4. 将实体对象按 ID 存入二级缓存（实体缓存区域）    │
  │ 5. 返回实体列表                                │
  └─────────────────────────────────────────────┘

第二次执行相同查询：
  ┌─────────────────────────────────────────────┐
  │ 1. 从查询缓存获取 ID 列表                      │
  │ 2. 遍历每个 ID：                               │
  │    ├─ 二级缓存命中 → 返回实体                   │
  │    └─ 二级缓存未命中 → 查询数据库加载实体         │
  │ 3. 返回实体列表                                │
  └─────────────────────────────────────────────┘
```

**关键关系**：查询缓存**依赖二级缓存**。如果实体没有启用二级缓存，查询缓存的作用非常有限：

```
无二级缓存时的查询缓存：
  → 从查询缓存拿到 ID 列表
  → 每个 ID 都无法从二级缓存拿到实体
  → 逐个查询数据库（N+1 问题）
  → 查询缓存形同虚设
```

因此启用查询缓存的实体**必须**同时启用二级缓存。

### 查询缓存的失效机制

查询缓存的失效是**表级别**的：一旦某个表发生任何 INSERT、UPDATE 或 DELETE 操作，**所有**涉及该表的查询缓存条目都会失效。这是通过 Hibernate 的时间戳机制来实现的：

1. Hibernate 在 `TimestampsCache` 区域中维护每个表的最后更新时间戳
2. 每个查询缓存条目存储时，会记录该查询涉及的所有表及其当前时间戳
3. 下次查询时，Hibernate 对比存储的时间戳与 `TimestampsCache` 中的当前时间戳
4. 如果不一致 → 缓存失效，重新执行查询并更新缓存

## 4.2 查询缓存的启用

### 全局启用

```properties
hibernate.cache.use_query_cache=true
```

### 按查询启用

```java
// HQL 查询
List<Book> books = session.createQuery(
    "from Book b where b.price > :minPrice", Book.class)
    .setParameter("minPrice", new BigDecimal("50"))
    .setCacheable(true)                    // 启用查询缓存
    .setCacheRegion("bookQueryRegion")     // 指定缓存区域（可选）
    .getResultList();

// Criteria 查询
List<Book> books = session.createQuery(
    "from Book b where b.price > :minPrice", Book.class)
    .setParameter("minPrice", new BigDecimal("50"))
    .setCacheable(true)
    .getResultList();
```

### @NamedQuery 方式

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@NamedQuery(
    name = "Book.findExpensive",
    query = "from Book b where b.price > :minPrice",
    hints = @QueryHint(name = "org.hibernate.cacheable", value = "true")
)
public class Book {
    // ...
}
```

### 在 Spring Data JPA 中启用

```java
public interface BookRepository extends JpaRepository<Book, Long> {

    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    List<Book> findByPriceGreaterThan(BigDecimal minPrice);
}
```

## 4.3 查询缓存的适用场景

| 适合查询缓存 | 不适合查询缓存 |
|-------------|--------------|
| 查询结果相对稳定（表很少更新） | 高频更新的表（订单、流水） |
| 相同查询参数频繁执行 | 查询参数变化极多（几乎每次不同） |
| 查询涉及的表量少 | 结果集很大（数万条） |
| 报表类查询（周期性生成） | 实时性要求极高（毫秒级） |
| 字典/配置查询 | 动态过滤的组合爆炸查询 |

**典型案例**：

```java
// 适合——查询国家列表（表不变，查询频繁）
@NamedQuery(
    name = "Country.findAll",
    query = "from Country order by code",
    hints = @QueryHint(name = "org.hibernate.cacheable", value = "true")
)

// 不适合——查询最近的订单（表频繁更新）
@NamedQuery(
    name = "Order.findRecent",
    query = "from Order o where o.createdAt > :since"
    // 不要启用查询缓存——订单表频繁更新导致缓存不断失效
)
```

## 4.4 查询缓存的失效

### 时间戳缓存区域（TimestampsCache）

Hibernate 内部维护一个 `TimestampsCache` 区域，结构如下：

```
TimestampsCache = {
    "com.example.Book"    → 1716000000000 (该表最近更新时间戳)
    "com.example.Author"  → 1716000000000
    "com.example.Order"   → 1716000120000
    ...
}
```

每个查询缓存条目存储时记录：

```
QueryCacheEntry {
    queryKey: "from Book where price > ?__[50.00]",
    result: [1L, 5L, 8L, 12L],
    timestamps: {
        "com.example.Book" → 1716000000000  // 当时的时间戳
    }
}
```

### 失效过于频繁的问题

如果一个表频繁更新（例如每秒数十条 UPDATE），查询缓存条目会不断失效，导致：

1. 缓存命中率极低
2. 每次查询都要重新执行 SQL
3. 缓存维护开销反而成为负担

**解决方案**：
- 对此类表不启用查询缓存
- 使用更精细的实体缓存替代（按 ID 索引，失效粒度更细）
- 考虑使用应用层缓存（如 Redis），手动控制失效策略

## 4.5 查询缓存与实体缓存的区别

| 维度 | 查询缓存 | 实体缓存 |
|------|---------|---------|
| **存储内容** | 查询结果的 **ID 列表** | **完整实体对象**（反序列化后的数据） |
| **索引方式** | 按查询语句 + 参数值 | 按实体类型 + 主键 ID |
| **失效粒度** | 粗粒度——表级别（任何数据变更都使整个区域失效） | 细粒度——按主键精确失效 |
| **适用查询** | 特定 HQL/JPQL/Criteria | 按 ID 加载（`find()`/`get()`） |
| **依赖关系** | 依赖二级缓存 | 独立 |
| **空间占用** | 较小（仅 ID 列表） | 较大（完整对象图） |
| **配置方式** | 按查询逐个或在查询定义时启用 | 按实体类型配置 |

# 5. 常见二级缓存提供者

## 5.1 Ehcache

Ehcache 是 Hibernate 二级缓存最流行的实现，历史悠久、成熟稳定。Ehcache 2.x 是 Hibernate 5.x 默认支持的缓存提供者（需单独引入依赖），Ehcache 3.x 通过 JCache 接口集成。

### 依赖配置

**Maven（Ehcache 2.x + Hibernate 5.x）：**

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-ehcache</artifactId>
    <version>5.6.15.Final</version>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>2.10.9.2</version>
</dependency>
```

**Maven（Ehcache 3.x + Hibernate 5.x 通过 JCache）：**

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-jcache</artifactId>
    <version>5.6.15.Final</version>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>3.10.8</version>
</dependency>
<dependency>
    <groupId>javax.cache</groupId>
    <artifactId>cache-api</artifactId>
    <version>1.1.1</version>
</dependency>
```

### 完整配置示例（Ehcache 2.x）

**application.properties**：

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory
hibernate.cache.use_structured_entries=true
hibernate.cache.use_minimal_puts=true
hibernate.generate_statistics=true
```

**ehcache.xml**（放在 `src/main/resources` 下）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
         updateCheck="false"
         monitoring="autodetect"
         dynamicConfig="true">

    <!--
        ========== 磁盘持久化配置 ==========
        当 JVM 重启后，缓存数据可以从磁盘恢复
    -->
    <diskStore path="java.io.tmpdir/hibernate-ehcache"/>

    <!-- ========== 默认缓存区域 ========== -->
    <defaultCache
        maxEntriesLocalHeap="10000"
        maxEntriesLocalDisk="100000"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="600"
        memoryStoreEvictionPolicy="LRU"
        diskSpoolBufferSizeMB="30"
        diskExpiryThreadIntervalSeconds="120">
        <persistence strategy="localTempSwap"/>
    </defaultCache>

    <!-- ========== 实体缓存区域 ========== -->

    <!-- Book 实体：读多写少，堆内 100K 条目，TTL 30 分钟 -->
    <cache name="com.example.Book"
        maxEntriesLocalHeap="100000"
        eternal="false"
        timeToIdleSeconds="600"
        timeToLiveSeconds="1800"
        memoryStoreEvictionPolicy="LRU">
        <persistence strategy="localTempSwap"/>
    </cache>

    <!-- Country 实体：静态数据，永不过期，仅 500 条 -->
    <cache name="com.example.Country"
        maxEntriesLocalHeap="500"
        eternal="true"
        memoryStoreEvictionPolicy="LFU">
    </cache>

    <!-- Order 实体：高频更新，不做二级缓存（在 ehcache.xml 中不配置该区域即可） -->

    <!-- ========== 集合缓存区域 ========== -->
    <cache name="com.example.Department.employees"
        maxEntriesLocalHeap="5000"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="900"
        memoryStoreEvictionPolicy="LRU">
    </cache>

    <!-- ========== Hibernate 内部缓存区域 ========== -->

    <!-- 查询缓存区域（默认） -->
    <cache name="org.hibernate.cache.internal.StandardQueryCache"
        maxEntriesLocalHeap="5000"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="600"
        memoryStoreEvictionPolicy="LRU">
    </cache>

    <!-- 时间戳缓存：数据量极小，但永不过期 -->
    <cache name="org.hibernate.cache.spi.TimestampsCache"
        maxEntriesLocalHeap="500"
        eternal="true">
    </cache>

    <!-- 自定义查询缓存区域：报表查询 -->
    <cache name="reportQueryRegion"
        maxEntriesLocalHeap="2000"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="1200"
        memoryStoreEvictionPolicy="LRU">
    </cache>

    <!-- 自定义查询缓存区域：高频实时查询 -->
    <cache name="frequentQueryRegion"
        maxEntriesLocalHeap="3000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="300"
        memoryStoreEvictionPolicy="LFU">
    </cache>

</ehcache>
```

### 完整配置示例（Ehcache 3.x + JCache）

**ehcache3.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns="http://www.ehcache.org/v3"
        xmlns:jsr107="http://www.ehcache.org/v3/jsr107">

    <service>
        <jsr107:defaults enable-management="true" enable-statistics="true"/>
    </service>

    <!-- 持久化目录 -->
    <persistence directory="${java.io.tmpdir}/ehcache3-hibernate"/>

    <!-- ========== 缓存模板 ========== -->

    <cache-template name="entityCacheTemplate">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>
        <resources>
            <heap unit="entries">50000</heap>
            <offheap unit="MB">100</offheap>
        </resources>
    </cache-template>

    <cache-template name="queryCacheTemplate">
        <expiry>
            <ttl unit="minutes">10</ttl>
        </expiry>
        <resources>
            <heap unit="entries">10000</heap>
            <offheap unit="MB">20</offheap>
        </resources>
    </cache-template>

    <!-- ========== 实体缓存 ========== -->
    <cache alias="com.example.Book" uses-template="entityCacheTemplate"/>
    <cache alias="com.example.Author" uses-template="entityCacheTemplate"/>
    <cache alias="com.example.Country">
        <expiry><none/></expiry>
        <resources><heap unit="entries">500"/></resources>
    </cache>

    <!-- ========== 集合缓存 ========== -->
    <cache alias="com.example.Department.employees" uses-template="entityCacheTemplate"/>

    <!-- ========== Hibernate 内部缓存 ========== -->
    <cache alias="org.hibernate.cache.internal.StandardQueryCache" uses-template="queryCacheTemplate"/>
    <cache alias="org.hibernate.cache.spi.TimestampsCache">
        <expiry><none/></expiry>
        <resources><heap unit="entries">500"/></resources>
    </cache>

</config>
```

**application.properties（JCache 模式）**：

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=jcache
hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
hibernate.javax.cache.uri=ehcache3.xml
hibernate.generate_statistics=true
```

## 5.2 Hazelcast

Hazelcast 是分布式缓存解决方案，适合多节点集群部署。所有节点共享同一个缓存集群，天然支持跨 JVM 的缓存一致性。

### 依赖配置

```xml
<!-- Hibernate 5.x + Hazelcast 集成 -->
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-hazelcast</artifactId>
    <version>5.6.15.Final</version>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast</artifactId>
    <version>5.3.7</version>
</dependency>
```

### 完整配置示例

**application.properties**：

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=com.hazelcast.hibernate.HazelcastLocalCacheRegionFactory
# 可选：使用分布式缓存工厂（所有节点共享）
# hibernate.cache.region.factory_class=com.hazelcast.hibernate.HazelcastCacheRegionFactory
hibernate.cache.use_minimal_puts=true
hibernate.generate_statistics=true
```

**hazelcast.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<hazelcast xmlns="http://www.hazelcast.com/schema/config"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.hazelcast.com/schema/config
           http://www.hazelcast.com/schema/config/hazelcast-config-5.3.xsd">

    <cluster-name>hibernate-cluster</cluster-name>

    <network>
        <join>
            <multicast enabled="true"/>
            <!-- 或使用 TCP/IP 静态成员列表 -->
            <!--
            <tcp-ip enabled="true">
                <member>192.168.1.101</member>
                <member>192.168.1.102</member>
                <member>192.168.1.103</member>
            </tcp-ip>
            -->
        </join>
    </network>

    <!-- 默认缓存配置 -->
    <map name="default">
        <time-to-live-seconds>600</time-to-live-seconds>
        <max-idle-seconds>300</max-idle-seconds>
        <eviction eviction-policy="LRU" max-size-policy="PER_NODE" size="50000"/>
    </map>

    <!-- 实体缓存 -->
    <map name="com.example.Book">
        <time-to-live-seconds>1800</time-to-live-seconds>
        <max-idle-seconds>600</max-idle-seconds>
        <eviction eviction-policy="LRU" max-size-policy="PER_NODE" size="100000"/>
    </map>

    <map name="com.example.Country">
        <time-to-live-seconds>0</time-to-live-seconds>  <!-- 永不过期 -->
        <eviction eviction-policy="LFU" max-size-policy="PER_NODE" size="500"/>
    </map>

    <!-- 查询缓存 -->
    <map name="org.hibernate.cache.internal.StandardQueryCache">
        <time-to-live-seconds>600</time-to-live-seconds>
        <max-idle-seconds>300</max-idle-seconds>
        <eviction eviction-policy="LRU" max-size-policy="PER_NODE" size="10000"/>
    </map>

    <!-- 时间戳缓存 -->
    <map name="org.hibernate.cache.spi.TimestampsCache">
        <time-to-live-seconds>0</time-to-live-seconds>
    </map>

</hazelcast>
```

### 分布式环境下的缓存一致性

Hazelcast 通过以下机制保证集群缓存一致性：

- **失效广播**：节点更新数据时，向集群广播失效消息，其他节点移除对应缓存条目
- **近缓存（Near Cache）**：每个节点维护本地近缓存，减少网络往返
- **分区备份**：数据在集群中分片存储，每个分片有备份副本

```java
// HazelcastLocalCacheRegionFactory vs HazelcastCacheRegionFactory
// LocalCacheRegionFactory：
//   - 在本地维护一份近缓存，读取速度快
//   - 更新时广播失效消息
//   - 适合读多写少的场景

// HazelcastCacheRegionFactory：
//   - 无本地近缓存，每次读取都从分布式 map 获取
//   - 一致性更强，但延迟更高
//   - 适合读写都频繁的场景
```

## 5.3 Infinispan

Infinispan 是 Red Hat/JBoss 提供的缓存方案，与 JBoss/WildFly 深度集成。支持事务型缓存（TRANSACTIONAL 策略），需要 JTA 事务管理器。

### 依赖配置

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-infinispan</artifactId>
    <version>5.6.15.Final</version>
</dependency>
<dependency>
    <groupId>org.infinispan</groupId>
    <artifactId>infinispan-core</artifactId>
    <version>14.0.21.Final</version>
</dependency>
<!-- 如果需要 JTA 事务支持 -->
<dependency>
    <groupId>org.infinispan</groupId>
    <artifactId>infinispan-jcache</artifactId>
    <version>14.0.21.Final</version>
</dependency>
```

### 完整配置示例

**application.properties**：

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=org.hibernate.cache.infinispan.InfinispanRegionFactory
hibernate.cache.infinispan.cfg=infinispan.xml
hibernate.generate_statistics=true
```

**infinispan.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<infinispan xmlns="urn:infinispan:config:14.0">

    <cache-container name="hibernate" default-cache="entity">
        <!-- JGroups 传输配置（集群通信） -->
        <transport cluster="hibernate-cluster" node-name="${jboss.node.name}"/>

        <!-- ========== 序列化 ========== -->
        <serialization>
            <context-initializer class="org.hibernate.cache.infinispan.ExternalizerBridge"/>
        </serialization>

        <!-- ========== 复制缓存（适合小型集群，读多写少） ========== -->
        <replicated-cache-configuration name="replicated-entity">
            <locking isolation="READ_COMMITTED"/>
            <transaction mode="NONE"/>
            <expiration max-idle="600000"/>
            <memory max-count="100000" when-full="REMOVE"/>
        </replicated-cache-configuration>

        <!-- ========== 分布式缓存（适合大型集群） ========== -->
        <distributed-cache-configuration name="distributed-entity" owners="2">
            <locking isolation="READ_COMMITTED"/>
            <transaction mode="NONE"/>
            <expiration max-idle="600000"/>
            <memory max-count="50000" when-full="REMOVE"/>
        </distributed-cache-configuration>

        <!-- ========== 失效缓存（适合写多读少的场景） ========== -->
        <invalidation-cache-configuration name="invalidation-entity">
            <locking isolation="READ_COMMITTED"/>
            <transaction mode="NONE"/>
            <expiration max-idle="600000"/>
        </invalidation-cache-configuration>

        <!-- ========== 事务型缓存（支持 TRANSACTIONAL 策略） ========== -->
        <distributed-cache-configuration name="transactional-entity" owners="2">
            <locking isolation="REPEATABLE_READ"/>
            <transaction mode="FULL_XA"
                          locking="PESSIMISTIC"
                          transaction-manager-lookup="org.infinispan.transaction.lookup.JBossStandaloneJTAManagerLookup"/>
            <expiration max-idle="600000"/>
            <memory max-count="100000"/>
        </distributed-cache-configuration>

        <!-- 缓存实例定义 -->
        <distributed-cache name="entity" configuration="distributed-entity"/>
        <replicated-cache name="immutable-entity" configuration="replicated-entity"/>

        <!-- Hibernate 内部缓存 -->
        <local-cache name="org.hibernate.cache.internal.StandardQueryCache">
            <expiration max-idle="300000" lifespan="600000"/>
            <memory max-count="10000"/>
        </local-cache>
        <local-cache name="org.hibernate.cache.spi.TimestampsCache">
            <memory max-count="500"/>
        </local-cache>

        <!-- 实体缓存覆写 -->
        <distributed-cache name="com.example.Book" configuration="distributed-entity"/>
        <replicated-cache name="com.example.Country" configuration="immutable-entity"/>
    </cache-container>

</infinispan>
```

**TRANSACTIONAL 策略实体示例**：

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL, region = "com.example.Account")
public class Account {
    @Id
    private Long id;
    private String owner;
    private BigDecimal balance;

    // Infinispan transitional 模式下，balance 的更新与缓存
    // 在同一 JTA 事务中提交或回滚，保证强一致性
}
```

## 5.4 Redis（通过 Redisson）

Redis 作为二级缓存适用于已有 Redis 基础设施的项目，支持集群部署。

### 依赖配置

**Maven（Hibernate 5.x）**：

```xml
<!-- Redisson 的 Hibernate 5.x 集成模块 -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-hibernate-53</artifactId>
    <version>3.28.0</version>
</dependency>
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.28.0</version>
</dependency>
```

**Maven（Hibernate 6.x）**：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-hibernate-6</artifactId>
    <version>3.28.0</version>
</dependency>
```

### 完整配置示例

**application.properties**：

```properties
# ========== Hibernate 二级缓存 ==========
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=org.redisson.hibernate.RedissonRegionFactory
# 或使用本地缓存包装（Redis + 本地缓存双层）
# hibernate.cache.region.factory_class=org.redisson.hibernate.RedissonLocalCachedRegionFactory
hibernate.cache.use_minimal_puts=true
hibernate.generate_statistics=true

# Redisson 本地缓存配置（仅在 RedissonLocalCachedRegionFactory 下生效）
# hibernate.cache.redisson.local_cache.max_size=10000
# hibernate.cache.redisson.local_cache.time_to_live=300000
# hibernate.cache.redisson.local_cache.eviction_policy=LRU
```

**redisson.yaml**（放在 `src/main/resources`）：

```yaml
# ========== 单节点模式 ==========
singleServerConfig:
  address: "redis://127.0.0.1:6379"
  connectionPoolSize: 20
  connectionMinimumIdleSize: 10
  idleConnectionTimeout: 10000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  subscriptionsPerConnection: 5
  subscriptionConnectionMinimumIdleSize: 1
  subscriptionConnectionPoolSize: 10

# ========== 集群模式（可选） ==========
# clusterServersConfig:
#   nodeAddresses:
#     - "redis://192.168.1.101:6379"
#     - "redis://192.168.1.102:6379"
#     - "redis://192.168.1.103:6379"
#   loadBalancer: org.redisson.connection.balancer.RoundRobinLoadBalancer
#   slaveConnectionMinimumIdleSize: 10
#   slaveConnectionPoolSize: 20
#   masterConnectionMinimumIdleSize: 10
#   masterConnectionPoolSize: 20

# ========== 哨兵模式（可选） ==========
# sentinelServersConfig:
#   sentinelAddresses:
#     - "redis://192.168.1.10:26379"
#     - "redis://192.168.1.11:26379"
#     - "redis://192.168.1.12:26379"
#   masterName: "mymaster"
#   database: 0

# ========== 编码配置 ==========
codec: !<org.redisson.codec.Kryo5Codec> {}
# 或使用 JSON 编码（更易调试但占用空间大）
# codec: !<org.redisson.codec.JsonJacksonCodec> {}

# ========== 线程配置 ==========
threads: 8
nettyThreads: 8
```

**Spring Boot 自动配置**：

```java
@Configuration
public class HibernateCacheConfig {

    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient() throws IOException {
        Config config = Config.fromYAML(
            new ClassPathResource("redisson.yaml").getInputStream()
        );
        return Redisson.create(config);
    }

    @Bean
    public HibernatePropertiesCustomizer hibernateCacheCustomizer(
            RedissonClient redissonClient) {
        return properties -> {
            properties.put("hibernate.cache.region.factory_class",
                "org.redisson.hibernate.RedissonRegionFactory");
            // 将 RedissonClient 注入 Hibernate（重要！）
            properties.put("hibernate.cache.redisson.config", redissonClient);
        };
    }
}
```

### Redis 缓存的优缺点

| 优点 | 缺点 |
|------|------|
| 独立于应用 JVM 进程，重启不丢缓存 | 网络延迟高于本地缓存（~1ms vs ~ns） |
| 支持集群，天然多节点共享 | 需额外维护 Redis 集群 |
| 内存弹性扩展 | 持久化（RDB/AOF）开销 |
| 丰富的监控和运维工具 | 序列化/反序列化开销 |

## 5.5 JCache（JSR-107）

JCache（JSR-107）是 Java 缓存标准 API。任何实现了 JCache 规范的缓存提供者都可以通过 Hibernate 的 `hibernate-jcache` 模块无缝集成。

### 依赖配置

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-jcache</artifactId>
    <version>5.6.15.Final</version>
</dependency>
<dependency>
    <groupId>javax.cache</groupId>
    <artifactId>cache-api</artifactId>
    <version>1.1.1</version>
</dependency>
<!-- 选择某一个 JCache 实现 -->
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>3.10.8</version>
</dependency>
```

### 配置示例

**application.properties**：

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=jcache
hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
hibernate.javax.cache.uri=ehcache3.xml
# 或对于 Hazelcast：
# hibernate.javax.cache.provider=com.hazelcast.cache.HazelcastMemberCachingProvider
# 或对于 Infinispan：
# hibernate.javax.cache.provider=org.infinispan.jcache.embedded.JCachingProvider
```

**JCache 编程式配置**（无需 XML）：

```java
@Configuration
public class JCacheConfig {

    @Bean
    public javax.cache.CacheManager cacheManager() {
        CachingProvider provider = Caching.getCachingProvider(
            "org.ehcache.jsr107.EhcacheCachingProvider"
        );
        javax.cache.CacheManager cacheManager = provider.getCacheManager();

        // 编程式创建缓存配置
        MutableConfiguration<Object, Object> entityConfig =
            new MutableConfiguration<Object, Object>()
                .setTypes(Object.class, Object.class)
                .setStoreByValue(false)
                .setExpiryPolicyFactory(
                    CreatedExpiryPolicy.factoryOf(
                        new Duration(TimeUnit.MINUTES, 30)
                    )
                );

        cacheManager.createCache("com.example.Book", entityConfig);
        cacheManager.createCache("com.example.Country",
            new MutableConfiguration<Object, Object>()
                .setTypes(Object.class, Object.class)
                .setStoreByValue(false)
                .setExpiryPolicyFactory(
                    AccessedExpiryPolicy.factoryOf(
                        new Duration(TimeUnit.HOURS, 24)
                    )
                )
        );

        // Hibernate 内部缓存
        cacheManager.createCache(
            "org.hibernate.cache.internal.StandardQueryCache", entityConfig
        );
        cacheManager.createCache(
            "org.hibernate.cache.spi.TimestampsCache",
            new MutableConfiguration<Object, Object>()
                .setTypes(Object.class, Object.class)
                .setStoreByValue(false)
                .setExpiryPolicyFactory(
                    EternalExpiryPolicy.factoryOf()
                )
        );

        return cacheManager;
    }
}
```

## 5.6 Caffeine

Caffeine 是高性能本地缓存库，基于 W-TinyLFU 驱逐算法，性能优于 Guava Cache 和 Ehcache 2.x。

### 依赖配置

```xml
<!-- 第三方 hibernate-caffeine 集成 -->
<dependency>
    <groupId>com.github.mxab</groupId>
    <artifactId>hibernate-caffeine</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
<!-- JCache API（hibernate-caffeine 通过 JCache 接口适配） -->
<dependency>
    <groupId>javax.cache</groupId>
    <artifactId>cache-api</artifactId>
    <version>1.1.1</version>
</dependency>
```

### 完整配置示例

**application.properties**：

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=jcache
hibernate.javax.cache.provider=com.github.mxab.caffeine.CaffeineCachingProvider
hibernate.generate_statistics=true
```

**编程式配置（推荐方式）**：

```java
@Configuration
public class CaffeineCacheConfig {

    @Bean
    public HibernatePropertiesCustomizer hibernateCacheCustomizer() {
        return properties -> {
            properties.put("hibernate.cache.use_second_level_cache", "true");
            properties.put("hibernate.cache.use_query_cache", "true");
            properties.put("hibernate.cache.region.factory_class", "jcache");
            properties.put("hibernate.javax.cache.provider",
                "com.github.mxab.caffeine.CaffeineCachingProvider");
        };
    }

    @Bean
    public javax.cache.CacheManager jCacheManager() {
        CachingProvider provider = Caching.getCachingProvider(
            "com.github.mxab.caffeine.CaffeineCachingProvider"
        );
        javax.cache.CacheManager manager = provider.getCacheManager();

        // 实体缓存：最大 50K 条目，写入后 30 分钟过期，访问后 10 分钟过期
        Cache<Object, Object> bookCache = Caffeine.newBuilder()
            .maximumSize(50_000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .expireAfterAccess(10, TimeUnit.MINUTES)
            .recordStats()
            .build();
        CaffeineCacheWrapper<Object, Object> bookWrapper =
            new CaffeineCacheWrapper<>("com.example.Book", bookCache);
        // 注册到 JCache Manager（简化示例，实际需通过 MutableConfiguration）

        // 静态数据缓存：最大 500 条，永不过期
        Cache<Object, Object> countryCache = Caffeine.newBuilder()
            .maximumSize(500)
            .recordStats()
            .build();
        // 注册...

        return manager;
    }

    /**
     * 创建指定配置的 Caffeine 缓存
     */
    private <K, V> Cache<K, V> createCache(
            long maxSize,
            long ttlMinutes,
            long ttiMinutes) {
        return Caffeine.newBuilder()
            .maximumSize(maxSize)
            .expireAfterWrite(ttlMinutes, TimeUnit.MINUTES)
            .expireAfterAccess(ttiMinutes, TimeUnit.MINUTES)
            .recordStats()
            .build();
    }
}
```

### Caffeine vs Ehcache 3.x 本地缓存性能对比

| 指标 | Caffeine | Ehcache 3.x |
|------|----------|-------------|
| 驱逐算法 | W-TinyLFU（命中率最优） | LRU / LFU |
| 读吞吐量（单线程） | ~150M ops/s | ~60M ops/s |
| 写吞吐量（单线程） | ~100M ops/s | ~30M ops/s |
| 内存开销 | 低（自适应） | 中等 |
| 过期策略 | 写后/访问后/自定义 | 写后/访问后/自定义 |
| 异步支持 | 原生 CompletableFuture | 有限支持 |
| 持久化 | 不支持 | 支持磁盘持久化 |

**选择建议**：
- 纯本地单机缓存，追求极致性能 → **Caffeine**
- 需要磁盘持久化（重启恢复缓存） → **Ehcache 3.x**
- 需要分布式/集群缓存 → **Hazelcast / Redisson(Redis)**

# 6. 缓存操作 API

## 6.1 SessionFactory.getCache()

Hibernate 提供了 `SessionFactory.getCache()` 方法获取 Cache 引用，用于程序化管理二级缓存：

```java
import org.hibernate.Cache;
import org.hibernate.SessionFactory;

Cache cache = sessionFactory.getCache();
```

**Cache 接口主要方法**：

| 方法 | 说明 |
|------|------|
| `evictEntityData(Class entityClass, Object id)` | 从二级缓存中移除指定实体 |
| `evictEntityData(Class entityClass)` | 从二级缓存中移除该类型的所有实体 |
| `evictCollectionData(String role)` | 从二级缓存中移除指定集合（role 格式：全限定类名.属性名） |
| `evictCollectionData(String role, Object id)` | 移除指定实体的指定集合缓存 |
| `evictQueryRegions()` | 清除**所有**查询缓存区域 |
| `evictDefaultQueryRegion()` | 清除默认查询缓存区域 |
| `evictAllRegions()` | 清除**所有**二级缓存（实体+集合+查询+时间戳） |
| `containsEntity(Class entityClass, Object id)` | 检查实体是否在二级缓存中 |
| `containsCollection(String role, Object id)` | 检查集合是否在二级缓存中 |
| `evictNaturalIdRegion(Class entityClass)` | 清除自然 ID 缓存 |

**使用示例**：

```java
@Service
public class CacheManagementService {

    private final SessionFactory sessionFactory;

    public CacheManagementService(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }

    /**
     * 清除指定 Book 实体的缓存
     */
    public void evictBook(Long bookId) {
        sessionFactory.getCache()
            .evictEntityData(Book.class, bookId);
    }

    /**
     * 清除所有 Book 缓存
     */
    public void evictAllBooks() {
        sessionFactory.getCache()
            .evictEntityData(Book.class);
    }

    /**
     * 清除指定 Department 的 employees 集合缓存
     */
    public void evictDepartmentEmployees(Long deptId) {
        sessionFactory.getCache()
            .evictCollectionData(
                "com.example.Department.employees", deptId
            );
    }

    /**
     * 清除所有自定义查询缓存区域
     */
    public void evictReportQueryCache() {
        sessionFactory.getCache()
            .evictQueryRegion("reportQueryRegion");
    }

    /**
     * 清除所有查询缓存
     */
    public void evictAllQueryCaches() {
        sessionFactory.getCache()
            .evictQueryRegions();
    }

    /**
     * 紧急清空所有缓存（慎用）
     */
    public void evictAllCaches() {
        sessionFactory.getCache()
            .evictAllRegions();
    }

    /**
     * 检查实体是否已缓存
     */
    public boolean isCached(Class<?> entityClass, Object id) {
        return sessionFactory.getCache()
            .containsEntity(entityClass, id);
    }
}
```

## 6.2 Session 级别缓存操作

Session（一级缓存）的缓存操作方法：

```java
@Service
public class SessionCacheService {

    @PersistenceContext
    private EntityManager entityManager;

    /**
     * 从一级缓存中移除单个实体
     */
    public void evictEntity(Object entity) {
        Session session = entityManager.unwrap(Session.class);
        session.evict(entity);
        // 该实体变为游离态，后续修改不会被自动同步到数据库
    }

    /**
     * 清空整个一级缓存
     */
    public void clearPersistenceContext() {
        Session session = entityManager.unwrap(Session.class);
        session.clear();
        // 所有托管实体变为游离态
    }

    /**
     * 检查实体是否在一级缓存中
     */
    public boolean isInPersistenceContext(Object entity) {
        Session session = entityManager.unwrap(Session.class);
        return session.contains(entity);
    }

    /**
     * 批量导入示例：定期清空一级缓存
     */
    public void batchImport(List<Book> books) {
        Session session = entityManager.unwrap(Session.class);
        int count = 0;
        for (Book book : books) {
            session.persist(book);
            if (++count % 50 == 0) {
                session.flush();
                session.clear();  // 清空一级缓存
            }
        }
    }
}
```

## 6.3 手动控制缓存

### 定时任务批量清除缓存

```java
@Component
public class CacheScheduledTasks {

    private final SessionFactory sessionFactory;

    public CacheScheduledTasks(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }

    /**
     * 每小时清除一次过期查询缓存
     */
    @Scheduled(fixedRate = 3600_000)
    public void evictQueryCaches() {
        Cache cache = sessionFactory.getCache();
        cache.evictQueryRegions();
        log.info("查询缓存已定时清理");
    }

    /**
     * 每天凌晨 3 点全量刷新主数据缓存
     */
    @Scheduled(cron = "0 0 3 * * ?")
    public void refreshMasterDataCache() {
        Cache cache = sessionFactory.getCache();
        cache.evictEntityData(Country.class);
        cache.evictEntityData(Currency.class);
        cache.evictEntityData(Language.class);
        log.info("主数据缓存已刷新");
    }
}
```

### 外部消息队列触发缓存失效

```java
@Component
public class CacheInvalidationListener {

    private final SessionFactory sessionFactory;

    public CacheInvalidationListener(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }

    /**
     * 监听数据变更消息，精确失效缓存
     * 适用于多服务共享数据库的场景
     */
    @RabbitListener(queues = "cache.invalidation")
    public void handleCacheInvalidation(CacheInvalidationMessage message) {
        Cache cache = sessionFactory.getCache();

        switch (message.getType()) {
            case "ENTITY":
                cache.evictEntityData(
                    message.getEntityClass(),
                    message.getEntityId()
                );
                break;
            case "COLLECTION":
                cache.evictCollectionData(
                    message.getCollectionRole(),
                    message.getOwnerId()
                );
                break;
            case "REGION":
                cache.evictQueryRegion(message.getRegionName());
                break;
            case "ALL":
                cache.evictAllRegions();
                break;
            default:
                log.warn("未知缓存失效类型: {}", message.getType());
        }
        log.info("缓存已失效: {}", message);
    }
}
```

### Hibernate Statistics API 监控

```java
@Service
public class CacheStatisticsService {

    private final SessionFactory sessionFactory;

    public CacheStatisticsService(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }

    public CacheStats getCacheStats() {
        Statistics stats = sessionFactory.getStatistics();

        CacheStats result = new CacheStats();
        // 二级缓存统计
        result.setSecondLevelCacheHitCount(
            stats.getSecondLevelCacheHitCount()
        );
        result.setSecondLevelCacheMissCount(
            stats.getSecondLevelCacheMissCount()
        );
        result.setSecondLevelCachePutCount(
            stats.getSecondLevelCachePutCount()
        );

        // 查询缓存统计
        result.setQueryCacheHitCount(
            stats.getQueryCacheHitCount()
        );
        result.setQueryCacheMissCount(
            stats.getQueryCacheMissCount()
        );
        result.setQueryCachePutCount(
            stats.getQueryCachePutCount()
        );

        // 计算命中率
        long totalHits = stats.getSecondLevelCacheHitCount();
        long totalAccess = totalHits + stats.getSecondLevelCacheMissCount();
        result.setHitRatio(totalAccess > 0
            ? (double) totalHits / totalAccess * 100
            : 0.0
        );

        // 区域级别统计
        String[] regionNames = stats.getSecondLevelCacheRegionNames();
        Map<String, RegionStats> regionStatsMap = new HashMap<>();
        for (String regionName : regionNames) {
            SecondLevelCacheStatistics regionStats =
                stats.getSecondLevelCacheStatistics(regionName);
            RegionStats rs = new RegionStats();
            rs.setName(regionName);
            rs.setHitCount(regionStats.getHitCount());
            rs.setMissCount(regionStats.getMissCount());
            rs.setPutCount(regionStats.getPutCount());
            rs.setElementCountInMemory(
                regionStats.getElementCountInMemory()
            );
            rs.setSizeInMemory(regionStats.getSizeInMemory());
            regionStatsMap.put(regionName, rs);
        }
        result.setRegionStats(regionStatsMap);

        return result;
    }

    /**
     * 重置统计信息
     */
    public void resetStats() {
        sessionFactory.getStatistics().clear();
    }

    /**
     * 打印缓存统计报告
     */
    public String printReport() {
        CacheStats stats = getCacheStats();
        StringBuilder sb = new StringBuilder();
        sb.append("========== Hibernate 缓存统计报告 ==========\n");
        sb.append(String.format(
            "二级缓存命中次数: %d\n", stats.getSecondLevelCacheHitCount()
        ));
        sb.append(String.format(
            "二级缓存未命中次数: %d\n", stats.getSecondLevelCacheMissCount()
        ));
        sb.append(String.format(
            "二级缓存命中率: %.2f%%\n", stats.getHitRatio()
        ));
        sb.append(String.format(
            "查询缓存命中次数: %d\n", stats.getQueryCacheHitCount()
        ));
        sb.append(String.format(
            "查询缓存未命中次数: %d\n", stats.getQueryCacheMissCount()
        ));
        sb.append("\n--- 各区域统计 ---\n");
        for (RegionStats rs : stats.getRegionStats().values()) {
            long regionTotal = rs.getHitCount() + rs.getMissCount();
            double regionRatio = regionTotal > 0
                ? (double) rs.getHitCount() / regionTotal * 100
                : 0.0;
            sb.append(String.format(
                "  [%s] 命中率: %.2f%% 内存条目: %d\n",
                rs.getName(), regionRatio, rs.getElementCountInMemory()
            ));
        }
        sb.append("=============================================\n");
        return sb.toString();
    }
}
```

### JMX 暴露统计信息

```java
@Configuration
@EnableMBeanExport
public class HibernateJmxConfig {

    @Bean
    public HibernateStatisticsMBean hibernateStatisticsMBean(
            EntityManagerFactory entityManagerFactory) {
        SessionFactory sf = entityManagerFactory
            .unwrap(SessionFactory.class);
        sf.getStatistics().setStatisticsEnabled(true);

        return new HibernateStatisticsMBean(sf);
    }

    @ManagedResource(objectName = "com.example:name=HibernateCacheStatistics")
    public static class HibernateStatisticsMBean {

        private final SessionFactory sessionFactory;

        public HibernateStatisticsMBean(SessionFactory sessionFactory) {
            this.sessionFactory = sessionFactory;
        }

        @ManagedAttribute
        public double getSecondLevelCacheHitRatio() {
            Statistics stats = sessionFactory.getStatistics();
            long hits = stats.getSecondLevelCacheHitCount();
            long total = hits + stats.getSecondLevelCacheMissCount();
            return total > 0 ? (double) hits / total * 100 : 0.0;
        }

        @ManagedAttribute
        public long getSecondLevelCacheHitCount() {
            return sessionFactory.getStatistics()
                .getSecondLevelCacheHitCount();
        }

        @ManagedAttribute
        public long getSecondLevelCacheMissCount() {
            return sessionFactory.getStatistics()
                .getSecondLevelCacheMissCount();
        }

        @ManagedOperation
        public void clearAllCaches() {
            sessionFactory.getCache().evictAllRegions();
        }

        @ManagedOperation
        public void clearStatistics() {
            sessionFactory.getStatistics().clear();
        }
    }
}
```

### Spring Boot Actuator 集成

如果引入了 Spring Boot Actuator，可以通过 `/actuator/metrics` 端点查看 Hibernate 缓存指标：

```properties
# application.properties
management.endpoints.web.exposure.include=health,info,metrics
management.metrics.export.simple.enabled=true
```

```java
@Component
public class HibernateCacheMetricsBinder {

    private final MeterRegistry meterRegistry;
    private final SessionFactory sessionFactory;

    public HibernateCacheMetricsBinder(
            MeterRegistry meterRegistry,
            EntityManagerFactory entityManagerFactory) {
        this.meterRegistry = meterRegistry;
        this.sessionFactory = entityManagerFactory
            .unwrap(SessionFactory.class);
        bindMetrics();
    }

    private void bindMetrics() {
        Statistics stats = sessionFactory.getStatistics();

        Gauge.builder("hibernate.cache.hit.ratio",
                () -> {
                    long hits = stats.getSecondLevelCacheHitCount();
                    long total = hits + stats.getSecondLevelCacheMissCount();
                    return total > 0 ? (double) hits / total : 0.0;
                })
            .description("Hibernate 二级缓存命中率")
            .register(meterRegistry);
    }
}
```

# 7. 缓存最佳实践

## 7.1 缓存对象选择

### 优先缓存

| 数据类型 | 示例 | 缓存策略 |
|---------|------|---------|
| 主数据 | 国家、货币、语言、省份 | READ_ONLY |
| 字典/枚举映射 | 订单状态、用户角色、权限 | READ_ONLY 或 NONSTRICT_READ_WRITE |
| 配置参数 | 系统参数、功能开关 | NONSTRICT_READ_WRITE |
| 不常变的业务数据 | 产品目录、部门信息、用户基本信息 | READ_WRITE |
| 层级数据 | 组织架构树、分类树 | READ_WRITE（连同集合缓存） |

### 谨慎缓存

| 数据类型 | 风险 | 缓解措施 |
|---------|------|---------|
| 频繁更新的数据 | 缓存命中率低，维护开销大 | 降低 TTL，或仅用一级缓存 |
| 大对象（BLOB/CLOB） | 占用大量缓存空间 | 排除在缓存外，单独查询 |
| 关联深度大的对象图 | 缓存序列化复杂、空间大 | 只缓存必要的实体和集合 |
| 高并发更新的热点数据 | Soft Lock 竞争严重 | 考虑应用层分布式锁 + 缓存 |

### 不应缓存

| 数据类型 | 原因 |
|---------|------|
| 临时数据（验证码、Token） | 生命周期极短，无缓存价值 |
| 实时要求极高的数据（股票价格、库存） | 缓存延迟不可接受 |
| 用户会话数据 | 与 Session 绑定，无需二级缓存 |
| 一次性的查询结果 | 不会重复使用 |

## 7.2 缓存策略选择决策树

```
开始
  │
  ├─ 数据是否会被修改？
  │    │
  │    ├─ 否 → 使用 READ_ONLY 策略
  │    │       │
  │    │       └─ 在 ehcache.xml 中配置 eternal="true"
  │    │
  │    └─ 是 ──→ 修改频率如何？
  │              │
  │              ├─ 极低（天/周级别）→ NONSTRICT_READ_WRITE
  │              │       │
  │              │       └─ 适合配置参数、系统设置等
  │              │
  │              ├─ 低～中等（分钟/小时级别）→ READ_WRITE
  │              │       │
  │              │       ├─ TTL 设置：30 ~ 60 分钟
  │              │       └─ 大多数业务实体
  │              │
  │              └─ 频繁（秒级别）→ 需要更高级方案
  │                      │
  │                      ├─ 是否需要强一致性？
  │                      │    ├─ 是 → TRANSACTIONAL（Infinispan + JTA）
  │                      │    └─ 否 → 应用层缓存（Redis/Caffeine）
  │                      │           + 手动失效控制
  │                      │
  │                      └─ 或放弃二级缓存，仅用一级缓存
  │
  └─ 是否需要查询缓存？
       │
       ├─ 是 → 确保相关实体已启用二级缓存
       │       │
       │       ├─ 查询参数变化少 → 启用查询缓存
       │       │    └─ TTL 设置：5 ~ 15 分钟（短于实体缓存 TTL）
       │       │
       │       └─ 查询参数变化多 → 不启用查询缓存
       │            └─ 每个不同参数值生成独立条目
       │
       └─ 否 → 仅使用实体缓存 + 按 ID 查询
```

## 7.3 缓存大小管理

### 容量配置原则

| 缓存区域 | 推荐最大条目数 | 依据 |
|---------|--------------|------|
| 实体缓存（大表） | 100,000 | 按实体大小 × 条目数估算内存 |
| 实体缓存（小表） | 所有记录数 | 主数据表通常几百到几千条 |
| 集合缓存 | 10,000 ~ 50,000 | 按关联数估算 |
| 查询缓存 | 5,000 ~ 10,000 | 每个查询是一个条目 |
| 时间戳缓存 | 500 | 每个表一个时间戳，量极小 |

### 内存估算公式

```
缓存内存占用 ≈ 条目数 × (实体序列化后大小 + 缓存 Key 开销)
              + 集合缓存条目数 × (ID 列表大小)
              + 查询缓存条目数 × (ID 列表大小 + 查询键大小)
```

### 驱逐策略选择

| 策略 | 适用场景 |
|------|---------|
| **LRU**（Least Recently Used） | 一般场景，最近访问的数据更可能再次访问 |
| **LFU**（Least Frequently Used） | 热点数据明确，长期低访问频率的冷数据可被驱逐 |
| **FIFO**（First In First Out） | 简单场景，按写入时间顺序驱逐 |

### TTL 与 TTI 配置

```properties
# Ehcache 2.x 配置
eternal=false           # 非永久缓存
timeToLiveSeconds=1800  # TTL：创建后 30 分钟过期
timeToIdleSeconds=600   # TTI：10 分钟未访问则过期
```

```java
// Caffeine 配置
Caffeine.newBuilder()
    .expireAfterWrite(30, TimeUnit.MINUTES)   // TTL
    .expireAfterAccess(10, TimeUnit.MINUTES)   // TTI
    .build();
```

**推荐配置参考**：

| 数据类型 | TTL（写后过期） | TTI（空闲过期） |
|---------|---------------|---------------|
| 静态字典 | eternal（永不过期） | — |
| 主数据 | 30～60 分钟 | 10～20 分钟 |
| 业务实体 | 15～30 分钟 | 5～10 分钟 |
| 查询缓存 | 5～15 分钟 | 3～5 分钟 |

### 监控内存使用

```java
// 获取各区域内存占用
SecondLevelCacheStatistics regionStats =
    sessionFactory.getStatistics()
        .getSecondLevelCacheStatistics("com.example.Book");

System.out.println("内存中条目数: " + regionStats.getElementCountInMemory());
System.out.println("内存占用(近似): " + regionStats.getSizeInMemory());
System.out.println("磁盘条目数: " + regionStats.getElementCountOnDisk());
```

## 7.4 集群环境中的缓存一致性

### 分布式缓存的挑战

在多节点集群中，每个节点可能持有同一实体的旧版本缓存数据：

```
节点 A：缓存中 Book#1.title = "旧版本"
节点 B：缓存中 Book#1.title = "新版本"（刚更新了数据库）
用户请求负载到节点 A → 读到旧数据
```

### 缓存同步策略

#### 策略一：失效广播（Invalidation Broadcasting）

```
更新流程：
  节点A 更新 Book#1
    → 更新数据库
    → 向所有节点广播："Book#1 已失效"
    → 其他节点移除 Book#1 的缓存条目
    → 下次读取时从数据库重新加载
```

**优点**：实现简单，仅广播失效消息，开销小
**缺点**：失效广播虽有短暂延迟窗口但不影响最终一致性

**适用提供者**：Ehcache（RMI/JMS 复制）、Hazelcast、Infinispan

#### 策略二：更新广播（Update Broadcasting）

```
更新流程：
  节点A 更新 Book#1
    → 更新数据库
    → 将新的 Book#1 数据广播到所有节点
    → 所有节点直接更新缓存
```

**优点**：节点无需重新查询数据库
**缺点**：网络传输开销大（对象序列化），大对象影响性能

#### 策略三：外部消息队列通知

```
更新流程：
  节点A 更新 Book#1
    → 更新数据库
    → 发送消息到 MQ（Kafka/RabbitMQ）："Book#1 已更新"
    → 所有节点消费消息
    → 各自清除本地缓存中的 Book#1
```

**优点**：解耦、可靠（MQ 保证消息不丢）、跨数据中心
**缺点**：引入额外组件（MQ），增加延迟

```java
// 示例——监听 CDC 事件触发缓存失效
@Component
public class DebeziumEventListener {

    private final SessionFactory sessionFactory;

    @EventListener
    public void onDataChange(DataChangeEvent event) {
        String tableName = event.getTableName();
        Object entityId = event.getPrimaryKey();
        Class<?> entityClass = resolveEntityClass(tableName);

        if (entityClass != null) {
            sessionFactory.getCache()
                .evictEntityData(entityClass, entityId);
            log.info("CDC 触发缓存失效: {}#{}", entityClass.getSimpleName(), entityId);
        }
    }
}
```

### 推荐方案

对于大多数场景，推荐以下组合：

```
数据库变更 → Debezium CDC → Kafka → 消费者 → 调用 Cache.evict()
                                       ↓
                               所有节点清除缓存
```

或更简单的方案（单数据源 + 多应用节点）：

```
应用节点 A 写入成功 → 发送缓存失效事件（Redis Pub/Sub / MQ）
                    → 所有节点（包括自身）清除相关缓存
```

## 7.5 缓存性能监控

### 关键监控指标

| 指标 | 含义 | 健康值 | 告警阈值 |
|------|------|--------|---------|
| 二级缓存命中率 | 命中 / (命中 + 未命中) | > 80% | < 50% |
| 查询缓存命中率 | 同上（查询缓存维度） | > 60% | < 30% |
| 缓存条目数 | 内存中缓存的条目数量 | 在容量上限 50%-80% | > 90% |
| 缓存内存占用 | 近似内存占用（字节） | 在堆内存 10% 以内 | > 堆内存 20% |
| 缓存驱逐次数 | 因容量满被驱逐的次数 | 接近 0 | 持续增长 |

### 通过 Spring Boot Actuator 自定义端点

```java
@Component
@Endpoint(id = "hibernate-cache")
public class HibernateCacheEndpoint {

    private final SessionFactory sessionFactory;

    public HibernateCacheEndpoint(
            EntityManagerFactory entityManagerFactory) {
        this.sessionFactory = entityManagerFactory
            .unwrap(SessionFactory.class);
    }

    @ReadOperation
    public Map<String, Object> getCacheStats() {
        Statistics stats = sessionFactory.getStatistics();
        Map<String, Object> result = new LinkedHashMap<>();

        // 二级缓存统计
        long slHits = stats.getSecondLevelCacheHitCount();
        long slMisses = stats.getSecondLevelCacheMissCount();
        long slTotal = slHits + slMisses;
        double slRatio = slTotal > 0 ? (double) slHits / slTotal * 100 : 0.0;

        result.put("secondLevelCache", Map.of(
            "hits", slHits,
            "misses", slMisses,
            "puts", stats.getSecondLevelCachePutCount(),
            "hitRatio", String.format("%.2f%%", slRatio)
        ));

        // 查询缓存统计
        long qHits = stats.getQueryCacheHitCount();
        long qMisses = stats.getQueryCacheMissCount();
        long qTotal = qHits + qMisses;
        double qRatio = qTotal > 0 ? (double) qHits / qTotal * 100 : 0.0;

        result.put("queryCache", Map.of(
            "hits", qHits,
            "misses", qMisses,
            "puts", stats.getQueryCachePutCount(),
            "hitRatio", String.format("%.2f%%", qRatio)
        ));

        // 各区域详情
        String[] regionNames = stats.getSecondLevelCacheRegionNames();
        Map<String, Map<String, Object>> regions = new LinkedHashMap<>();
        for (String name : regionNames) {
            SecondLevelCacheStatistics rs =
                stats.getSecondLevelCacheStatistics(name);
            long rHits = rs.getHitCount();
            long rMisses = rs.getMissCount();
            long rTotal = rHits + rMisses;
            double rRatio = rTotal > 0 ? (double) rHits / rTotal * 100 : 0.0;
            regions.put(name, Map.of(
                "hits", rHits,
                "misses", rMisses,
                "puts", rs.getPutCount(),
                "hitRatio", String.format("%.2f%%", rRatio),
                "elementsInMemory", rs.getElementCountInMemory(),
                "sizeInMemory", rs.getSizeInMemory()
            ));
        }
        result.put("regions", regions);

        return result;
    }

    @WriteOperation
    public Map<String, String> evictAll() {
        sessionFactory.getCache().evictAllRegions();
        return Map.of("result", "所有缓存已清空");
    }

    @WriteOperation
    public Map<String, String> evictRegion(@Selector String region) {
        sessionFactory.getCache().evictQueryRegion(region);
        return Map.of("result", "缓存区域已清空: " + region);
    }
}
```

## 7.6 常见缓存问题

### 缓存穿透（Cache Penetration）

**定义**：查询的数据在缓存和数据库中都不存在，每次请求都穿透缓存直接查询数据库。

```
客户端 → 查询不存在的数据 → 缓存未命中 → 查询数据库 → 也不存在 → 返回空
                                                                    ↓
客户端 → 再次查询相同不存在数据 → 缓存仍未命中 → 再次查询数据库（穿透！）
```

**解决方案**：

**方案一——缓存空值**：

```java
public Book findBook(Long id) {
    Book book = cache.get("book:" + id);
    if (book != null) {
        if (book == Book.NULL_OBJECT) {  // 特殊标记对象
            return null;  // 确认不存在，不查数据库
        }
        return book;
    }

    book = bookRepository.findById(id).orElse(null);
    if (book == null) {
        cache.put("book:" + id, Book.NULL_OBJECT, Duration.ofMinutes(5));
    } else {
        cache.put("book:" + id, book, Duration.ofMinutes(30));
    }
    return book;
}
```

**方案二——布隆过滤器**：

```java
@Component
public class BookExistenceFilter {

    private BloomFilter<Long> bloomFilter;
    private final BookRepository bookRepository;

    public BookExistenceFilter(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
        // 初始化时加载所有 Book ID
        List<Long> allIds = bookRepository.findAllIds();
        bloomFilter = BloomFilter.create(
            Funnels.longFunnel(), allIds.size(), 0.01
        );
        allIds.forEach(bloomFilter::put);
    }

    public Optional<Book> findById(Long id) {
        // 布隆过滤器说不存在 → 直接返回，不查 DB 和缓存
        if (!bloomFilter.mightContain(id)) {
            return Optional.empty();
        }
        // 可能存在，查缓存/DB
        return bookRepository.findById(id);
    }

    /**
     * 新增书籍时更新布隆过滤器
     */
    public void addBookId(Long id) {
        bloomFilter.put(id);
    }

    /**
     * 定时全量重建布隆过滤器
     */
    @Scheduled(fixedRate = 600_000)
    public void rebuildFilter() {
        List<Long> allIds = bookRepository.findAllIds();
        bloomFilter = BloomFilter.create(
            Funnels.longFunnel(), allIds.size(), 0.01
        );
        allIds.forEach(bloomFilter::put);
    }
}
```

### 缓存击穿（Cache Breakdown / Hotspot Invalid）

**定义**：热点数据缓存到期，大量并发请求同时涌入数据库。

```
时间线：
  T0: 热门 Book#1 的缓存在 TTL 到期后被清除
  T1: 1000 个并发请求同时查询 Book#1
        → 都发现缓存未命中
        → 同时查询数据库
        → 数据库压力骤升（可能打垮数据库）
```

**解决方案——互斥锁（Mutex Lock）**：

```java
@Service
public class CacheBreakdownProtectionService {

    private final ConcurrentHashMap<Long, ReentrantLock> locks =
        new ConcurrentHashMap<>();

    public Book getBook(Long id) {
        // 1. 尝试从缓存获取
        Book cached = cache.get("book:" + id);
        if (cached != null) {
            return cached;
        }

        // 2. 缓存未命中 → 获取该 ID 的专用锁
        ReentrantLock lock = locks.computeIfAbsent(
            id, k -> new ReentrantLock()
        );

        try {
            if (lock.tryLock(5, TimeUnit.SECONDS)) {
                try {
                    // 双重检查——可能其他线程已经加载了
                    cached = cache.get("book:" + id);
                    if (cached != null) {
                        return cached;
                    }

                    // 3. 从数据库加载
                    Book book = bookRepository.findById(id)
                        .orElse(null);
                    if (book != null) {
                        // 设置较长 TTL + 随机偏移防止同时过期
                        long ttl = 30 + ThreadLocalRandom.current().nextInt(10);
                        cache.put("book:" + id, book,
                            Duration.ofMinutes(ttl));
                    }
                    return book;
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 锁获取失败 → 稍后重试或返回空
        return null;
    }
}
```

**方案二——永不过期 + 异步更新**：

```java
@Component
public class NeverExpireCacheService {

    private final Map<Long, CacheEntry<Book>> localCache =
        new ConcurrentHashMap<>();

    /**
     * 缓存条目永不过期，但通过后台线程异步刷新
     */
    public Book getBook(Long id) {
        CacheEntry<Book> entry = localCache.get(id);
        if (entry == null) {
            // 首次加载
            Book book = bookRepository.findById(id).orElse(null);
            localCache.put(id, new CacheEntry<>(book));
            return book;
        }

        // 检查数据是否"逻辑过期"（缓存仍可用但需要刷新）
        if (entry.isExpired()) {
            // 异步刷新（不阻塞当前请求）
            CompletableFuture.runAsync(() -> refreshEntry(id));
        }

        // 返回可能稍旧的数据（而非阻塞等待）
        return entry.getData();
    }

    private void refreshEntry(Long id) {
        Book fresh = bookRepository.findById(id).orElse(null);
        if (fresh != null) {
            localCache.put(id, new CacheEntry<>(fresh));
        }
    }

    @Data
    @AllArgsConstructor
    static class CacheEntry<T> {
        private T data;
        private long loadTime = System.currentTimeMillis();

        public CacheEntry(T data) {
            this.data = data;
        }

        public boolean isExpired() {
            return System.currentTimeMillis() - loadTime > 30_000; // 30 秒逻辑过期
        }
    }
}
```

### 缓存雪崩（Cache Avalanche）

**定义**：大量缓存 key 在同一时间点过期，所有请求同时涌向数据库。

```
原因：
  - 缓存预热时所有 key 设置了相同的 TTL
  - 缓存服务重启导致所有 key 同时失效
  - 定时任务批量刷新缓存

后果：
  → 数据库瞬时压力骤升
  → 数据库响应变慢
  → 应用线程池耗尽
  → 系统雪崩
```

**解决方案——随机化 TTL**：

```java
@Service
public class AvalancheProtectionService {

    /**
     * 放入缓存时添加随机偏移
     */
    public void cacheWithRandomTTL(String key, Object value, long baseMinutes) {
        // 基础 TTL ± 20% 的随机偏移
        long minOffset = (long) (baseMinutes * 0.8);
        long maxOffset = (long) (baseMinutes * 1.2);
        long actualTTL = minOffset
            + ThreadLocalRandom.current().nextLong(maxOffset - minOffset + 1);

        cache.put(key, value, Duration.ofMinutes(actualTTL));
    }

    /**
     * 缓存预热时使用错峰加载
     */
    @PostConstruct
    public void warmUpCache() {
        List<Long> hotBookIds = getHotBookIds();
        ExecutorService executor = Executors.newFixedThreadPool(4);

        for (int i = 0; i < hotBookIds.size(); i++) {
            Long id = hotBookIds.get(i);
            int delay = i * 100; // 每个 key 间隔 100ms 加载
            executor.submit(() -> {
                try {
                    Thread.sleep(delay);
                    Book book = bookRepository.findById(id).orElse(null);
                    if (book != null) {
                        cacheWithRandomTTL("book:" + id, book, 30);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        executor.shutdown();
    }

    /**
     * 多级缓存降级策略
     */
    public Book getBookWithFallback(Long id) {
        // L1: 本地缓存（Caffeine）
        Book cached = localCache.getIfPresent("book:" + id);
        if (cached != null) return cached;

        // L2: 分布式缓存（Redis / 二级缓存）
        cached = distributedCache.get("book:" + id);
        if (cached != null) {
            localCache.put("book:" + id, cached); // 回填 L1
            return cached;
        }

        // L3: 数据库
        try {
            Book book = bookRepository.findById(id).orElse(null);
            if (book != null) {
                distributedCache.put("book:" + id, book,
                    Duration.ofMinutes(30 + ThreadLocalRandom.current().nextInt(10)));
                localCache.put("book:" + id, book);
            }
            return book;
        } catch (Exception e) {
            // L4: 降级——返回本地缓存中的过期数据（如果可用）
            Book stale = localCache.getStaleIfPresent("book:" + id);
            log.warn("数据库查询失败，使用过期缓存数据: id={}", id);
            return stale;
        }
    }
}
```

### 缓存与数据库一致性

#### Cache-Aside（旁路缓存）——最常用

```
读流程：
  1. 查询缓存
  2. 命中 → 返回
  3. 未命中 → 查询数据库 → 写入缓存 → 返回

写流程：
  1. 更新数据库
  2. 删除（失效）缓存
```

```java
// Hibernate 的 READ_WRITE 策略即实现了类 Cache-Aside 模式
// 手动控制示例
@Transactional
public Book updateBook(Long id, BookDto dto) {
    Book book = bookRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("Book not found"));
    book.setTitle(dto.getTitle());
    book.setPrice(dto.getPrice());
    book = bookRepository.save(book);  // 更新数据库

    // 失效相关缓存
    sessionFactory.getCache().evictEntityData(Book.class, id);

    return book;
}
```

#### Read-Through（读穿透）

```
读流程：
  1. 查询缓存
  2. 命中 → 返回
  3. 未命中 → 缓存层自动查询数据库 → 写入缓存 → 返回

（应用代码只与缓存交互，缓存层负责加载数据）
```

Hibernate 的二级缓存+实体缓存近似实现了 Read-Through 模式（通过 `find()` 按 ID 查询时）。

#### Write-Through（写穿透）

```
写流程：
  1. 更新缓存
  2. 缓存层自动同步更新数据库
  3. 两者同时完成

（应用代码只与缓存交互）
```

Hibernate 的 `READ_WRITE` 策略部分近似（先更新 DB，再更新缓存，但顺序不是缓存优先）。

#### Write-Behind（异步写回）

```
写流程：
  1. 只更新缓存
  2. 异步批量同步到数据库
  3. 可能有数据丢失风险
```

Hibernate 的一级缓存 + `flush()` 机制类似于 Write-Behind（累积变更后批量写入数据库）。但 Hibernate 不推荐在二级缓存层面使用 Write-Behind，因为存在事务一致性风险。

### 一致性总结

| 模式 | 读流程责任 | 写流程责任 | 一致性 | Hibernate 对应 |
|------|-----------|-----------|--------|---------------|
| Cache-Aside | 应用代码 | 应用代码 | 最终一致 | `evictEntityData()` 手动控制 |
| Read-Through | 缓存层 | — | 最终一致 | 二级缓存+`find()` |
| Write-Through | — | 缓存层 | 强一致 | `READ_WRITE` 策略近似 |
| Write-Behind | — | 缓存层（异步） | 弱一致 | 一级缓存 flush 机制 |

# 8. 附录：完整项目配置示例

## 8.1 Spring Boot + Hibernate + Ehcache 2.x 完整配置

**pom.xml**：

```xml
<dependencies>
    <!-- Spring Boot Starter Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Hibernate Ehcache 集成 -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-ehcache</artifactId>
        <version>5.6.15.Final</version>
    </dependency>
    <dependency>
        <groupId>net.sf.ehcache</groupId>
        <artifactId>ehcache</artifactId>
        <version>2.10.9.2</version>
    </dependency>

    <!-- 数据库驱动 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**application.yml**：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory
          use_structured_entries: true
          use_minimal_puts: true
        generate_statistics: true
        format_sql: true
```

**ehcache.xml**：（同上第 5.1 节完整示例，略）

**实体示例**：

```java
@Entity
@Table(name = "books")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "com.example.Book")
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;

    @Version
    private Integer version;

    // getters and setters...
}

@Entity
@Table(name = "authors")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "author")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    private Set<Book> books = new HashSet<>();

    // getters and setters...
}

@Entity
@Table(name = "countries")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id
    @Column(length = 2)
    private String code;

    @Column(nullable = false)
    private String name;

    // getters and setters...
}
```

**Repository**：

```java
@Repository
public interface BookRepository extends JpaRepository<Book, Long> {

    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    List<Book> findByPriceGreaterThan(BigDecimal price);

    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    List<Book> findByAuthorId(Long authorId);
}
```

**缓存管理服务**：

```java
@Service
public class CacheAdminService {

    @PersistenceContext
    private EntityManager entityManager;

    /**
     * 获取详细的缓存统计信息
     */
    public Map<String, Object> getDetailedStats() {
        SessionFactory sf = entityManager.getEntityManagerFactory()
            .unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("entityFetchCount", stats.getEntityFetchCount());
        result.put("secondLevelCacheHitCount", stats.getSecondLevelCacheHitCount());
        result.put("secondLevelCacheMissCount", stats.getSecondLevelCacheMissCount());
        result.put("secondLevelCachePutCount", stats.getSecondLevelCachePutCount());
        result.put("queryCacheHitCount", stats.getQueryCacheHitCount());
        result.put("queryCacheMissCount", stats.getQueryCacheMissCount());
        result.put("queryCachePutCount", stats.getQueryCachePutCount());
        result.put("sessionOpenCount", stats.getSessionOpenCount());
        result.put("sessionCloseCount", stats.getSessionCloseCount());
        result.put("queryExecutionCount", stats.getQueryExecutionCount());
        result.put("transactionCount", stats.getTransactionCount());
        result.put("flushCount", stats.getFlushCount());

        // 计算命中率
        long l2Hits = stats.getSecondLevelCacheHitCount();
        long l2Total = l2Hits + stats.getSecondLevelCacheMissCount();
        result.put("secondLevelCacheHitRatio",
            l2Total > 0 ? String.format("%.2f%%", (double) l2Hits / l2Total * 100) : "N/A");

        long qHits = stats.getQueryCacheHitCount();
        long qTotal = qHits + stats.getQueryCacheMissCount();
        result.put("queryCacheHitRatio",
            qTotal > 0 ? String.format("%.2f%%", (double) qHits / qTotal * 100) : "N/A");

        return result;
    }

    /**
     * 使指定实体的缓存失效
     */
    @Transactional
    public void invalidateEntity(Class<?> entityClass, Object id) {
        SessionFactory sf = entityManager.getEntityManagerFactory()
            .unwrap(SessionFactory.class);
        sf.getCache().evictEntityData(entityClass, id);
    }
}
```

## 8.2 关键参数速查表

| 参数 | 类型 | 默认值 | 推荐值 | 说明 |
|------|------|--------|--------|------|
| `hibernate.cache.use_second_level_cache` | boolean | false | true | 启用二级缓存 |
| `hibernate.cache.use_query_cache` | boolean | false | true（按需） | 启用查询缓存 |
| `hibernate.cache.region.factory_class` | String | — | 依提供者 | 缓存工厂类 |
| `hibernate.cache.use_structured_entries` | boolean | false | true（开发环境） | 结构化缓存条目 |
| `hibernate.cache.use_minimal_puts` | boolean | false | true | 最小化缓存写入 |
| `hibernate.cache.default_cache_concurrency_strategy` | String | — | read-write | 默认缓存策略 |
| `hibernate.generate_statistics` | boolean | false | true（开发/调试） | 启用统计 |
| `hibernate.cache.keys_factory` | String | simple | simple | 缓存键生成工厂 |
| `hibernate.cache.auto_evict_collection_cache` | boolean | false | false | 自动驱逐集合缓存 |
| `javax.persistence.sharedCache.mode` | String | ENABLE_SELECTIVE | ENABLE_SELECTIVE | JPA 共享缓存模式 |

## 8.3 故障排查清单

| 症状 | 可能原因 | 排查步骤 | 解决方法 |
|------|---------|---------|---------|
| 缓存命中率为 0 | 未启用二级缓存或统计信息 | 检查 `generate_statistics=true` | 确保实体有 `@Cache` + `@Cacheable` 注解 |
| `NullPointerException` 在缓存访问时 | 实体未正确实现 `equals()`/`hashCode()` | 检查实体类的 `@Id` 字段的 equals/hashCode | 基于业务键或 `@Id` 实现 |
| 内存溢出 | 缓存条目过多无上限 | 检查 ehcache.xml 的 `maxEntriesLocalHeap` | 设置合理的最大条目数 + TTL |
| 缓存中读到旧数据 | 缓存未失效、TTL 太长 | 检查是否有外部直接更新数据库 | 设置合理的 TTL，引入失效机制 |
| 查询缓存频繁失效 | 表更新频繁导致查询缓存区域清空 | 检查 `TimestampsCache` 变化频率 | 对此表不启用查询缓存 |
| `SecondLevelCacheStatistics` 为 null | 实体所在区域未被缓存 | 检查 `@Cache(region=...)` 是否正确 | 确认 ehcache.xml 中存在对应名称的 `<cache>` |
| Soft Lock 获取超时 | 并发更新同一实体 | 检查事务隔离级别 | 使用 `READ_COMMITTED` 或更高，或降低此实体的并发更新频率 |
| 集群节点间缓存不一致 | 缓存同步机制失效 | 检查网络、配置 | 使用分布式缓存提供者 + 失效广播 |
