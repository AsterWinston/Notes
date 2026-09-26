# 1. Session 与 Persistence Context

## 1.1 Session 的本质

Session 是 Hibernate 应用程序与数据库交互的核心接口，其本质由两个核心职责构成：

- **Persistence Context（持久化上下文）**：作为一级缓存，管理被加载或持久化的实体实例
- **数据库操作入口**：提供 CRUD 操作、查询创建、事务管理等 API

Session = Persistence Context + 数据库操作入口。每次通过 Session 加载实体时，Hibernate 首先在 Persistence Context 中查找；命中则直接返回，未命中则查数据库并纳入缓存。

```java
// Session 接口继承体系
public interface Session extends SharedSessionContract, EntityManager {
    // JPA EntityManager 方法
    <T> T find(Class<T> entityClass, Object primaryKey);
    void persist(Object entity);
    <T> T merge(T entity);
    void remove(Object entity);
    
    // Hibernate 原生扩展方法
    Serializable save(Object entity);
    void update(Object entity);
    void saveOrUpdate(Object entity);
    void delete(Object entity);
    Object get(String entityName, Serializable id);
    Object load(String entityName, Serializable id);
    SessionFactory getSessionFactory();
    CriteriaBuilder getCriteriaBuilder();
}
```

### JPA EntityManager 与 Hibernate Session 的关系

Session 接口直接继承了 `javax.persistence.EntityManager`，因此 Session 就是 EntityManager。在 Hibernate 5.x+ 中，两者完全等同：

```java
// 以下两种方式获取的对象是同一个类型
EntityManager em = entityManagerFactory.createEntityManager();
// em 实际类型是 SessionImpl（实现了 Session 接口）

Session session = sessionFactory.openSession();
// session 实现类也是 SessionImpl

// 可以相互转换（无成本）
Session session = em.unwrap(Session.class);
EntityManager em = (EntityManager) session;
```

Hibernate 内部的实际实现类是 `SessionImpl`，它同时实现了 `Session` 接口和 `SessionImplementor` 内部接口。`SessionImplementor` 供 Hibernate 内部模块调用，暴露更多底层细节（如 PersistenceContext、JDBC 协调器等），不推荐应用代码直接使用。

### 线程安全性

Session 不是线程安全的。一个 Session 实例同一时刻只能由一个线程访问。多个线程共享同一个 Session 会导致数据混乱、脏检查异常。正确做法是为每个线程/请求创建独立的 Session：

```java
// 错误：多线程共享 Session（不安全）
Session session = sessionFactory.openSession();
ExecutorService executor = Executors.newFixedThreadPool(4);
for (int i = 0; i < 10; i++) {
    executor.submit(() -> {
        // 多个线程并发使用同一个 session，导致异常
        User user = session.get(User.class, 1L);
    });
}

// 正确：每个线程创建自己的 Session
executor.submit(() -> {
    Session session = sessionFactory.openSession();
    try {
        User user = session.get(User.class, 1L);
    } finally {
        session.close();
    }
});
```

## 1.2 SessionFactory

SessionFactory 是线程安全的重量级对象，一个应用通常只需要一个实例（单例模式）。它的创建成本极高，因为初始化过程涉及大量工作：

```java
// 创建 SessionFactory（通常在应用启动时执行一次）
Configuration configuration = new Configuration();
configuration.configure("hibernate.cfg.xml");
configuration.addAnnotatedClass(User.class);
configuration.addAnnotatedClass(Order.class);

SessionFactory sessionFactory = configuration.buildSessionFactory();
```

### 创建成本分析

1. **解析映射元数据（ORM 元模型）**：读取 hbm.xml 或注解，构建 `EntityPersister`、`CollectionPersister`
2. **构建 Persister 对象**：每个实体生成一个 `EntityPersister`（如 `SingleTableEntityPersister`），缓存 CRUD SQL 模板、字段映射
3. **初始化查询计划缓存**：解析命名查询、HQL 查询模板，构建 AST
4. **连接池初始化**：通过 `ConnectionProvider` 初始化数据库连接池
5. **类型系统初始化**：注册 `BasicType` 映射（如 `StringType`、`LongType`）
6. **拦截器与事件监听器注册**：构建事件系统

```java
// SessionFactoryImpl 内部结构示意
public class SessionFactoryImpl implements SessionFactoryImplementor {
    // 每个实体类一个 Persister
    private Map<String, EntityPersister> entityPersisters;
    // 每个集合映射一个 Persister
    private Map<String, CollectionPersister> collectionPersisters;
    // 命名查询缓存（HQL → AST）
    private QueryPlanCache queryPlanCache;
    // 连接提供者
    private ConnectionProvider connectionProvider;
    // 拦截器链
    private Interceptor interceptor;
    // JPA 元模型
    private MetamodelImplementor metamodel;
    // 类型解析器
    private TypeHelper typeHelper;
    // 统计信息（可选）
    private StatisticsImplementor statistics;
}
```

### 与 JPA EntityManagerFactory 的关系

Hibernate 的 `SessionFactory` 接口继承了 `javax.persistence.EntityManagerFactory`，二者完全互通：

```java
// JPA 方式创建
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
// emf 实际类型是 SessionFactoryImpl

// 转回 SessionFactory
SessionFactory sf = emf.unwrap(SessionFactory.class);
sf.getStatistics();  // 使用 Hibernate 原生功能

// Hibernate 原生方式创建
SessionFactory sf = new Configuration().configure().buildSessionFactory();
// sf 可直接当作 EntityManagerFactory 使用
EntityManagerFactory emf = (EntityManagerFactory) sf;
```

### 关闭与资源释放

SessionFactory 关闭时会释放所有底层资源：

```java
// 应用关闭时释放
sessionFactory.close();
```

关闭操作会：
1. Evict 所有二级缓存数据
2. 销毁连接池
3. 清理查询计划缓存
4. 注销 JNDI 绑定
5. 销毁所有 Session 工厂相关的线程资源

```java
// 典型的 SessionFactory 生命周期管理（Servlet/Spring 环境）
@Configuration
public class HibernateConfig {
    
    @Bean(destroyMethod = "close")
    public SessionFactory sessionFactory() {
        LocalSessionFactoryBuilder builder = 
            new LocalSessionFactoryBuilder(dataSource());
        builder.scanPackages("com.example.entity");
        builder.setProperty("hibernate.dialect", 
            "org.hibernate.dialect.MySQL8Dialect");
        builder.setProperty("hibernate.show_sql", "true");
        builder.setProperty("hibernate.hbm2ddl.auto", "validate");
        builder.setProperty("hibernate.current_session_context_class", 
            "thread");
        return builder.buildSessionFactory();
    }
}
```

## 1.3 Persistence Context 的核心职责

### 实体身份保证

在同一个 Persistence Context 范围内，同一数据库行始终映射到同一个 Java 对象实例（按 ID 和实体类型保证唯一性）：

```java
Session session = sessionFactory.openSession();
try {
    User u1 = session.get(User.class, 1L);  // SELECT
    User u2 = session.get(User.class, 1L);  // 不查库，返回相同实例
    System.out.println(u1 == u2);  // true — 对象身份一致
    
    // 通过查询加载也不会违反这一保证
    User u3 = session.createQuery("from User where id = 1", User.class)
        .uniqueResult();
    System.out.println(u1 == u3);  // true
} finally {
    session.close();
}
```

这一特性避免了数据不一致：修改 `u1` 的字段后，通过 `u2` 读到的就是修改后的值（因为是同一个对象），无需担心 `u1` 和 `u2` 分别持有不同版本的数据。

### 脏检查（Dirty Checking）

Persistence Context 在加载实体时保存一份快照（snapshot）。Flush 时，Hibernate 对比实体当前状态与快照，自动生成 UPDATE 语句，开发者无需显式调用 `session.update()`：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);     // 加载 + 保存快照
user.setName("New Name");                     // 只修改属性
user.setEmail("new@example.com");

tx.commit();  // flush → 检测到 name、email 变更 → 自动 UPDATE
session.close();
```

输出 SQL 大致如下：

```sql
SELECT * FROM users WHERE id = 1;
UPDATE users SET name = 'New Name', email = 'new@example.com' WHERE id = 1;
```

### 写后延迟（Write-Behind）

所有持久化操作（INSERT、UPDATE、DELETE）先在 Persistence Context 的内存中缓存，记录到 ActionQueue 中。直到 Flush 时，才将累积的操作按正确顺序批量发送到数据库：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

// 下面三行不会立即执行 SQL
session.persist(new User("Alice"));        // 加入 EntityInsertAction 队列
session.persist(new User("Bob"));          // 加入队列
session.get(User.class, 1L).setName("X");  // 加入 EntityUpdateAction 队列

tx.commit();  // flush → 批量执行 2 条 INSERT + 1 条 UPDATE
session.close();
```

优势：
- 减少数据库往返次数（JDBC Batch 机制）
- 优化 SQL 执行顺序以遵守外键约束
- 合并相邻的同类型操作

### 自动版本检查（乐观锁）

对使用 `@Version` 的实体，Hibernate 在 UPDATE 和 DELETE 时自动在 WHERE 子句中追加版本号条件，并在 SET 中递增版本号：

```java
// 乐观锁 SQL 自动生成
// UPDATE users SET name = 'X', version = 2 WHERE id = 1 AND version = 1;
// 若 version 不匹配（已被其他事务修改），UPDATE 影响 0 行 → OptimisticLockException
```

### 级联操作传播

Persistence Context 根据级联设置（`CascadeType`），将操作从父实体传播到关联子实体。

### 持久化上下文清理（clear）

调用 `session.clear()` 会清空整个 Persistence Context，所有被管理的实体变为游离态，快照也被丢弃：

```java
session.clear();  // 全部脱管

// 通常用于批处理场景，防止内存溢出
for (int i = 0; i < 100000; i++) {
    session.persist(new Record(i));
    if (i % 1000 == 0) {
        session.flush();   // 提交 SQL
        session.clear();   // 释放内存
    }
}
```

# 2. 实体生命周期

## 2.1 四种状态

Hibernate/JPA 定义实体实例在 Persistence Context 中的生命周期有四种状态：

| 状态 | Java 引用 | 数据库行 | Persistence Context 管理 | 脏检查 |
|------|-----------|----------|-------------------------|--------|
| Transient（瞬时态） | 有 | 无 | 否 | 否 |
| Managed/Persistent（托管态） | 有 | 有 | 是 | 是 |
| Detached（游离态） | 有 | 有 | 否 | 否 |
| Removed（删除态） | 有 | 有（待删） | 是 | 否 |

### Transient（瞬时态）

通过 `new` 创建，尚未与任何 Persistence Context 关联，数据库中没有对应记录：

```java
User user = new User("John");
// 此时 user 是瞬时态
// 调用 persist(user) 后转为托管态
```

### Managed/Persistent（托管态）

实体已与 Persistence Context 关联，由 Hibernate 管理其生命周期。所有属性变更都会被脏检查监控，在 Flush 时自动同步到数据库：

```java
User user = session.get(User.class, 1L);  // 托管态
user.setName("Updated");                    // 修改被监控
// flush 时自动 UPDATE
```

### Detached（游离态）

实体曾经与 Persistence Context 关联，但 Session 已关闭或实体被显式脱管（evict/detach）。数据库仍有对应记录，但不再被监控：

```java
User user = session.get(User.class, 1L);
session.close();   // 或 session.evict(user);
// 此时 user 是游离态
user.setName("Change");  // 不会自动同步到数据库
```

游离态实体的常用场景：在 Web 层编辑、跨请求传输实体 → 再通过 `merge()` 重新同步回数据库。

### Removed（删除态）

实体被标记为待删除，仍然受 Persistence Context 管理，但在 Flush 时将执行 DELETE 语句：

```java
User user = session.get(User.class, 1L);
session.remove(user);  // 进入删除态
// 此时 user 仍在 Persistence Context 中
user.setName("X");  // 通常不应再修改
// flush → DELETE FROM users WHERE id = 1
// flush 后 user 变为瞬时态
```

## 2.2 状态转换操作

### 2.2.1 persist() / save()

**persist()（JPA 标准）** 将瞬时态实体转为托管态：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = new User();
user.setName("Alice");
user.setEmail("alice@example.com");

session.persist(user);  // 瞬时态 → 托管态
// 注意：根据 ID 生成策略，INSERT 可能立即执行，也可能延迟到 flush
// 若 ID 策略为 IDENTITY → 立即 INSERT（需获取自增 ID）
// 若 ID 策略为 SEQUENCE → 延迟到 flush

tx.commit();  // flush 时执行尚未发出的 INSERT
session.close();
```

**save()（Hibernate 原生）** 也执行类似操作，但返回生成的 ID：

```java
Long id = (Long) session.save(user);  // 返回主键值
// persist() 返回 void，不返回 ID

// save() 对 Detached 实体的行为：
User existing = new User();
existing.setId(1L);  // 人为设置了一个已存在的 ID
session.save(existing);  
// → 可能尝试 INSERT，若 ID 已存在则报主键冲突
// persist() 对 Detached 实体 → 抛出 PersistenceException
```

**persist() vs save() 核心区别：**

| 对比维度 | persist() | save() |
|---------|-----------|--------|
| 标准 | JPA 标准 | Hibernate 原生 |
| 返回值 | void | Serializable（生成的 ID） |
| 对 Detached 实体 | 抛异常（EntityExistsException 或 PersistenceException） | 尝试 INSERT（可能主键冲突） |
| 语义 | 持久化"瞬时的"实体 | 保存（不关心瞬时或游离） |

```java
// persist 不会立即 INSERT 的示例（SEQUENCE/ASSIGNED 策略）
session.persist(user);     // 暂不 INSERT，加入 ActionQueue
System.out.println(user.getId());  // 可能为 null 或 sequence 预取值
session.flush();           // 此时才执行 INSERT
```

### 2.2.2 merge()

`merge()` 将游离态实体"合并"回当前 Persistence Context，返回新的托管实例，**原实例依然是游离态**——这是最容易被忽视的行为：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

// 假设 userFromWeb 是 Web 层传回的游离态实体
User detached = new User();
detached.setId(1L);
detached.setName("Updated Name");

User managed = session.merge(detached);  // 返回托管实例

System.out.println(detached == managed);   // false！
System.out.println(detached.getName());    // "Updated Name"（游离态未变）
System.out.println(managed.getName());     // "Updated Name"（托管态已合并）

// 后续修改必须使用 managed（托管实例）
managed.setEmail("new@example.com");  // 会被脏检查监控
detached.setEmail("ignored");          // 不会被监控

tx.commit();  // flush → UPDATE email = 'new@example.com'
session.close();
```

**merge 内部执行流程：**

```java
// 伪代码模拟 merge 内部逻辑
public <T> T merge(T entity) {
    // 1. 按 ID 在 Persistence Context 中查找
    Object existing = persistenceContext.getEntityByKey(entity.getId());
    
    if (existing != null) {
        // 3. 在 Persistence Context 中找到 → 复制属性
        copyProperties(entity, existing);
        return (T) existing;
    }
    
    // 2. Persistence Context 中未找到 → 查数据库
    Object dbEntity = loadFromDatabase(entity.getId());
    
    if (dbEntity != null) {
        // 3. 数据库中找到了 → 加载到 Persistence Context → 复制属性
        copyProperties(entity, dbEntity);
        return (T) dbEntity;
    }
    
    // 4. 数据库中也未找到 → 创建新托管实体（INSERT）
    Object newManaged = createNewInstance(entity);
    copyProperties(entity, newManaged);
    persistenceContext.addEntity(newManaged);
    return (T) newManaged;
}
```

**merge 的常见坑：**

```java
// 坑 1：忽略返回值，后续修改游离实例无效
session.merge(detachedUser);
detachedUser.setName("New Name");  // 无效！游离实例不被监控
// 正确：User managed = session.merge(detachedUser); managed.setName("New Name");

// 坑 2：级联合并时关联集合被整体替换
// 若 parent 有 Cascade.MERGE
Parent detachedParent = new Parent();
detachedParent.setId(1L);
detachedParent.getChildren().add(new Child("NewChild"));
Parent managed = session.merge(detachedParent);  
// → 会导致 children 集合被整体替换，旧 children 可能成为孤儿

// 坑 3：merge 与版本号
// 游离实体携带旧版本号，merge 后托管实体的版本号可能被覆盖
// 在 tx.commit() 时进行乐观锁检查
```

### 2.2.3 remove() / delete()

**remove()（JPA）** 将托管实体标记为删除：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);  // 托管态
session.remove(user);  // 标记删除（进入删除态）
// 此时 user 仍在 Persistence Context 中
// SQL 尚未执行

tx.commit();  // flush → DELETE FROM users WHERE id = 1
session.close();
```

**remove() 要求实体为托管态：**

```java
// 错误：对游离态实体调用 remove
User detached = new User();
detached.setId(1L);
session.remove(detached);  // IllegalArgumentException: Removing a detached instance

// 正确：先合并再删除
User managed = session.merge(detached);
session.remove(managed);
```

**delete()（Hibernate 原生）** 行为类似，但允许对游离实体直接删除（需配置）：

```java
session.delete(user);  // Hibernate 原生 API

// 级联删除配置示例
@Entity
public class Order {
    // ...
    @OneToMany(mappedBy = "order", cascade = CascadeType.REMOVE)
    private List<OrderItem> items;
}

// 删除 Order 时级联删除所有 OrderItem
Order order = session.get(Order.class, 1L);
session.remove(order);  // → DELETE Order + DELETE OrderItem(s)
```

**orphanRemoval=true 的行为：**

```java
@Entity
public class Parent {
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> children = new ArrayList<>();
}

// 从集合中移除子实体 → flush 时自动 DELETE 该子实体
Parent parent = session.get(Parent.class, 1L);
parent.getChildren().remove(0);  // 从集合移除
// flush → DELETE FROM child WHERE id = ?
// 无需显式调用 session.remove(child)，因为 orphanRemoval=true
```

### 2.2.4 refresh()

`refresh()` 从数据库重新加载实体状态，覆盖内存中未提交的修改：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);
// 此时另一个事务修改了数据库中该行的 name 和 version
user.setName("My Modification");  // 内存修改

session.refresh(user);
// → SELECT ... FROM users WHERE id = 1
// user.getName() 现在恢复为数据库中最新值（丢弃"My Modification"）
// LockMode 默认为 READ

tx.commit();
session.close();
```

**LockMode 参数：**

```java
// 基础刷新：读取当前状态
session.refresh(user, LockMode.READ);

// 悲观读锁刷新：获取共享锁
session.refresh(user, LockMode.PESSIMISTIC_READ);
// → SELECT ... FOR SHARE

// 悲观写锁刷新：获取排他锁
session.refresh(user, LockMode.PESSIMISTIC_WRITE);
// → SELECT ... FOR UPDATE
```

**级联刷新：**

```java
@OneToMany(mappedBy = "parent", cascade = CascadeType.REFRESH)
private List<Child> children;

// refresh(parent) 会级联刷新所有 children
session.refresh(parent);
```

### 2.2.5 detach() / evict()

将实体从 Persistence Context 中移除，使其不再受脏检查监控：

```java
// detach() — JPA 标准
session.detach(user);  // user 变为游离态

// evict() — Hibernate 原生，功能相同
session.evict(user);

// 代码等价：detach 实际委托给 evict
```

**clear() 清空整个 Persistence Context：**

```java
session.clear();  // 所有托管实体立即变为游离态
```

**游离态 vs 从上下文中移除的对比：**

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);
session.detach(user);

// 1. 修改不会被同步到数据库
user.setName("Detached Change");
tx.commit();  // flush 不包含此变更
session.close();

// 2. 尝试访问懒加载代理 → LazyInitializationException
session.detach(user);
List<Order> orders = user.getOrders();  // 如果 orders 未初始化
// → LazyInitializationException: could not initialize proxy - no Session

// 3. 重新附加游离实体
Session session2 = sessionFactory.openSession();
Transaction tx2 = session2.beginTransaction();
User managed = session2.merge(user);  // 使用 merge 重新合并
tx2.commit();
session2.close();
```

### 2.2.6 find() / get() / load()

**find()（JPA）/ get()（Hibernate）** 立即查询数据库，返回实体或 null：

```java
// find() — JPA
User user = session.find(User.class, 1L);
// → SELECT * FROM users WHERE id = 1
// 返回托管实体，或 null（无此 ID）

// get() — Hibernate 原生，行为完全相同
User user = session.get(User.class, 1L);
```

**getReference()（JPA）/ load()（Hibernate）** 返回代理对象，不立即查询数据库：

```java
// getReference() — JPA
User userRef = session.getReference(User.class, 1L);
// 不执行 SELECT！返回代理对象

// load() — Hibernate 原生
User user = session.load(User.class, 1L);
// 同样不立即查询

// 代理对象内部结构：
// user.getClass() → User$$HibernateProxy$xxx（CGLIB 代理类）
// user.getId() → 不需要初始化，从代理存储的 ID 直接返回
// user.getName() → 触发初始化 → SELECT * FROM users WHERE id = 1
// 代理初始化后，行为与普通实体完全一致
```

**load() 的懒加载行为详解：**

```java
session = sessionFactory.openSession();
try {
    User user = session.load(User.class, 1L);
    // 此刻无 SQL
    
    Long id = user.getId();  // 无 SQL（代理已知 ID）
    
    String name = user.getName();  // 触发 SQL！
    // SELECT * FROM users WHERE id = 1
    
    List<Order> orders = user.getOrders();  // 如果 FetchType.LAZY
    // 再触发一次 SQL（或已由二级缓存提供）
} finally {
    session.close();
}
```

**Session 关闭后访问未初始化代理 → LazyInitializationException：**

```java
User user;
try (Session session = sessionFactory.openSession()) {
    user = session.load(User.class, 1L);  // 仅代理，未初始化
}
// Session 已关闭

user.getName();  // LazyInitializationException: could not initialize proxy
```

**getReference() 对不存在 ID 的行为差异：**

```java
// 场景：id=999 在数据库中不存在

// 方式 1：find()——直接返回 null
User u1 = session.find(User.class, 999L);
System.out.println(u1 == null);  // true

// 方式 2：getReference()——返回空代理
User u2 = session.getReference(User.class, 999L);
System.out.println(u2 == null);  // false（得到了代理）

u2.getName();  // 访问属性时触发 SELECT → 找不到 → EntityNotFoundException

// 对比 load() 的旧版行为
User u3 = session.load(User.class, 999L);
// 旧版 Hibernate：立即返回 ObjectNotFoundException
// 新版 Hibernate：行为对齐 JPA，延迟到属性访问时抛出异常
```

### 2.2.7 lock()

`lock()` 将游离实体重新附加到 Persistence Context，或对托管实体升级锁级别：

```java
// 场景：重新附加游离实体
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User detached = new User();
detached.setId(1L);
detached.setVersion(2);

session.lock(detached, LockMode.NONE);
// → 将 detached 重新纳入 Persistence Context（无实际锁）
// 要求版本号匹配，否则抛出乐观锁异常

tx.commit();
session.close();

// LockMode 参数说明：
// LockMode.NONE      — 不获取锁，仅重新附加
// LockMode.READ       — 乐观读锁
// LockMode.WRITE      — 乐观写锁（自动升级为 LockMode.PESSIMISTIC_WRITE 或做版本检查）
// LockMode.OPTIMISTIC — 乐观锁（版本号检查）
// LockMode.PESSIMISTIC_READ  — 悲观读锁（SELECT ... FOR SHARE）
// LockMode.PESSIMISTIC_WRITE — 悲观写锁（SELECT ... FOR UPDATE）
// LockMode.PESSIMISTIC_FORCE_INCREMENT — 悲观写锁 + 版本号递增
```

## 2.3 状态转换图

完整的实体生命周期状态转换如下（文字描述）：

```
                    new()
                    ──────▶  Transient（瞬时态）
                               │
                     persist() │ save() │ saveOrUpdate()
                               ▼
                    Managed/Persistent（托管态）
                      │          │          │
              remove()│  detach()│   close()│
                      │  evict() │   clear()│
                      ▼          ▼          ▼
                Removed     Detached（游离态）
                 （删除态）      │
                      │   merge() │ update() │ saveOrUpdate()
                      │          ▼
                      ▼   Managed/Persistent（托管态）
                flush/commit
                      │
                      ▼
                 DELETE 执行
                 → 实例变为 Transient
```

**各 API 对应状态变化总结表：**

| 操作 | 输入状态 | 输出状态 | 说明 |
|------|---------|---------|------|
| `persist(entity)` | Transient | Managed | 持久化新实体 |
| `persist(entity)` | Detached | 异常 | EntityExistsException |
| `save(entity)` | Transient | Managed | 返回生成 ID |
| `save(entity)` | Detached | Managed | 可能主键冲突 |
| `merge(entity)` | Detached | Managed（新实例） | 原实例仍为游离态 |
| `merge(entity)` | Transient | Managed（新实例） | 创建新托管实例（INSERT） |
| `remove(entity)` | Managed | Removed | 标记待删除 |
| `delete(entity)` | Managed/Detached | Removed | Hibernate 原生 |
| `refresh(entity)` | Managed | Managed | 覆盖内存修改 |
| `detach/evict(entity)` | Managed | Detached | 移除出上下文 |
| `clear()` | Managed | Detached | 清空整个上下文 |
| `lock(entity, mode)` | Detached | Managed | 重新附加 |
| `find/get(entity, id)` | — | Managed | 查库并返回托管实体 |
| `getReference/load(entity, id)` | — | Managed（代理） | 返回未初始化的代理 |

# 3. Flush 机制

## 3.1 FlushMode

FlushMode 控制 Persistence Context 何时将缓存的变更同步到数据库：

```java
// FlushMode 枚举值（Hibernate 5+）
public enum FlushMode {
    // 查询前 + 事务提交前自动 flush
    AUTO,
    // 仅事务提交前自动 flush
    COMMIT,
    // 仅手动调用 flush() 时才 flush
    MANUAL,
    // 每次查询前都 flush（最严格，性能最差）
    ALWAYS
}
```

**FlushModeType（JPA 标准，只有两种）：**

```java
public enum FlushModeType {
    AUTO,   // 等价于 Hibernate FlushMode.AUTO
    COMMIT  // 等价于 Hibernate FlushMode.COMMIT
}
```

**Session 级别设置：**

```java
// Session 级别
session.setFlushMode(FlushMode.MANUAL);

// JPA 方式
entityManager.setFlushMode(FlushModeType.COMMIT);
```

**Query 级别设置（覆盖 Session 级别）：**

```java
// 对特定查询设置 FlushMode
List<User> users = session.createQuery("from User", User.class)
    .setFlushMode(FlushMode.COMMIT)  // 本查询不触发 flush
    .list();

// JPA 方式
List<User> users = entityManager.createQuery("select u from User u", User.class)
    .setFlushMode(FlushModeType.COMMIT)
    .getResultList();
```

**各模式行为对比：**

```java
// AUTO（默认）：
session.persist(new User("Alice"));           // INSERT 缓存
User u = session.createQuery("from User", User.class).uniqueResult();
// ↑ 查询前自动 flush → Alice 被 INSERT → 查询结果包含 Alice

// COMMIT：
session.persist(new User("Alice"));           // INSERT 缓存
User u = session.createQuery("from User", User.class).uniqueResult();
// ↑ 不自动 flush → Alice 未 INSERT → 查询结果不包含 Alice
tx.commit();  // 此时才 flush

// MANUAL：
session.persist(new User("Alice"));
session.createQuery("from User", User.class).list();  // 结果不含 Alice
tx.commit();  // 也不 flush！（MANUAL 不随 commit 自动 flush）
// 必须手动调用：session.flush();

// ALWAYS：
session.persist(new User("Alice"));
// 即便 Session.get() 也会触发 flush
User u = session.get(User.class, 2L);  // ← 触发 flush
```

## 3.2 Flush 触发时机

Flush 在以下时机触发：

```java
// 1. 事务提交
Transaction tx = session.beginTransaction();
session.persist(new User("A"));
tx.commit();  // commit 内部调用 session.flush()

// 2. 显式调用 flush()
session.persist(new User("B"));
session.flush();  // 立即同步到数据库

// 3. HQL/JPQL 查询执行前（AUTO 模式下）
session.persist(new User("C"));
List<User> users = session.createQuery("from User where name like '%C%'", User.class)
    .list();  // 查询前自动 flush

// 4. 原生 SQL 查询（默认不触发 flush）
session.persist(new User("D"));
List<Object[]> rows = session.createNativeQuery("SELECT * FROM users WHERE name = 'D'")
    .list();  // 不触发 flush → 查不到 D

// 强制原生 SQL 查询前 flush（Hibernate 6+）
List<Object[]> rows = session.createNativeQuery("SELECT * FROM users WHERE name = 'D'")
    .addSynchronizedEntityClass(User.class)  // 告知查询依赖 User 表
    .list();  // 触发 flush
```

**Flush 触发条件完整列表：**

| 触发时机 | AUTO | COMMIT | MANUAL | ALWAYS |
|---------|------|--------|--------|--------|
| `session.flush()` | ✓ | ✓ | ✓ | ✓ |
| `tx.commit()` | ✓ | ✓ | ✗ | ✓ |
| HQL/JPQL 查询前 | ✓ | ✗ | ✗ | ✓ |
| 原生 SQL 查询前 | ✗ | ✗ | ✗ | ✓ |
| `session.get()/find()` 前 | ✗ | ✗ | ✗ | ✓ |

## 3.3 Flush 内部执行流程

Flush 的内部执行分为多个阶段，由 `SessionImpl.flush()` → `AbstractFlushingEventListener` 驱动：

```
1. flushEverythingToExecutions(flushEvent)
   ├── 收集 PersistenceContext 中所有 EntityEntry
   ├── 对比当前状态与快照 → 生成 EntityUpdateAction
   ├── 识别新增实体 → 生成 EntityInsertAction
   └── 识别删除标记 → 生成 EntityDeleteAction
   
2. 收集集合变更 → CollectionRecreateAction / CollectionRemoveAction / CollectionUpdateAction

3. addPostFlushAction 处理级联操作

4. ActionQueue.prepareActions()
   ├── 按外键依赖拓扑排序
   ├── INSERT 操作预执行（生成 ID）
   └── 合并为 JDBC Batch

5. ActionQueue.executeActions()
   ├── 执行 EntityInsertAction(s)
   ├── 执行 EntityRecreateAction(s)（one-to-one 关联重建）
   ├── 执行 EntityUpdateAction(s)
   ├── 执行 CollectionRecreateAction(s)
   ├── 执行 CollectionRemoveAction(s)
   ├── 执行 CollectionUpdateAction(s)
   └── 执行 EntityDeleteAction(s)

6. 提交后清理（post-commit）
   └── 重置所有实体快照为当前状态
```

**代码示例——观察内部执行流程（开启 DEBUG 日志）：**

```java
// log4j2.xml 配置
// logger name="org.hibernate.event.internal" level="DEBUG"

Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

// 1. 新增
User u1 = new User("Alice");
session.persist(u1);

// 2. 修改
User u2 = session.get(User.class, 2L);
u2.setEmail("updated@example.com");

// 3. 删除
User u3 = session.get(User.class, 3L);
session.remove(u3);

// 4. 集合变更
Order order = session.get(Order.class, 10L);
order.getItems().add(new OrderItem("Product X"));

tx.commit();  // 触发 flush，观察日志中的执行顺序
session.close();

// 日志输出（简化）：
// [DEBUG] Flushing session
// [DEBUG] Processing flush-time cascades
// [DEBUG] Dirty checking entity: User#2
// [DEBUG] Found dirty property: email
// [DEBUG] Scheduling entity update: User#2
// [DEBUG] Scheduling collection update: Order#10.items
// [DEBUG] Executing batch size: 1 — INSERT INTO users ...
// [DEBUG] Executing batch size: 1 — UPDATE users SET email = ? WHERE id = ?
// [DEBUG] Executing batch size: 1 — DELETE FROM users WHERE id = ?
// [DEBUG] Executed flush
```

**执行顺序优先级（排序规则）：**

1. `EntityInsertAction` — 优先 INSERT（先有行才能引用）
2. `EntityUpdateAction` — 然后 UPDATE
3. `CollectionRecreateAction` / `CollectionRemoveAction` / `CollectionUpdateAction` — 集合变更
4. `EntityDeleteAction` — 最后 DELETE

## 3.4 脏检查机制（Dirty Checking）

### 原理

脏检查的核心机制：Persistence Context 在加载实体时保存一份属性值快照（snapshot）。在 Flush 时，逐个比较当前实体属性值与快照，发现差异则生成 UPDATE：

```java
// 快照存储在 PersistenceContext 的 EntityEntry 中
// EntityEntry 结构示意
class EntityEntry {
    Object[] loadedState;   // 加载时的属性值数组
    Object[] deletedState;  // 删除时的属性值（软删除用）
    Status status;          // MANAGED / DELETED / GONE / ...
    LockMode lockMode;
    EntityPersister persister;
}

// 脏检查伪代码
public void dirtyCheck(EntityEntry entry, Object entity) {
    Object[] currentState = entry.getPersister().getPropertyValues(entity);
    Object[] loadedState = entry.getLoadedState();
    
    // 深度比较（使用 Type.equals → 最终调用 equals/hashCode）
    int[] dirtyProperties = entry.getPersister().findDirty(
        currentState, loadedState, entity, session);
    
    if (dirtyProperties != null && dirtyProperties.length > 0) {
        // 生成 EntityUpdateAction
        scheduleUpdate(entity, dirtyProperties, currentState, loadedState);
    }
}
```

### 比较策略

**默认：全字段深度比较**——遍历每个属性，调用 Hibernate Type 的 `isEqual()` 方法：

```java
// Hibernate Type 的等值比较
public class StringType extends AbstractSingleColumnStandardBasicType<String> {
    @Override
    public boolean isEqual(String x, String y) {
        return x == y || (x != null && x.equals(y));
    }
}
// 对于关联属性（@ManyToOne），比较外键值而非整个对象
```

**字节码增强优化（@OptimisticLocking）：**

使用字节码增强后，Hibernate 在实体 setter 被调用时记录"脏属性索引"，Flush 时只比较这些已标记的字段：

```java
@Entity
@OptimisticLocking(type = OptimisticLockType.DIRTY)
public class User {
    @Id private Long id;
    private String name;
    private String email;
    
    // 字节码增强后，setName() 内部记录字段 index 0 已修改
    public void setName(String name) {
        // Hibernate 插入的字节码：
        // $$_hibernate_tracker.trackChange(propertyIndex);
        this.name = name;
    }
}

// 启用字节码增强（Maven 配置）
// <plugin>
//   <groupId>org.hibernate.orm.tooling</groupId>
//   <artifactId>hibernate-enhance-maven-plugin</artifactId>
//   <executions>
//     <execution>
//       <configuration>
//         <enableDirtyTracking>true</enableDirtyTracking>
//       </configuration>
//     </execution>
//   </executions>
// </plugin>
```

### 性能影响与优化

**问题场景：** 每个托管实体 Flush 时都要遍历所有属性进行等值比较，当 Persistence Context 中有大量实体时开销显著。

**优化手段：**

```java
// 1. @DynamicUpdate — 只 UPDATE 变更的列（减少 SQL 大小）
@Entity
@DynamicUpdate
public class User {
    // 只修改 name 时：
    // UPDATE users SET name = ? WHERE id = ?   (而非全字段 UPDATE)
}

// 2. @Immutable — 实体不可变，跳过脏检查
@Entity
@Immutable
public class Country {
    @Id private String code;
    private String name;
    // 永不 UPDATE，flush 时完全跳过此类实体
}

// 3. read-only Session
session.setDefaultReadOnly(true);
// 该 Session 加载的所有实体跳过脏检查

// 4. 单个实体设为只读
session.setReadOnly(user, true);

// 5. HQL 只读查询
List<User> users = session.createQuery("from User", User.class)
    .setReadOnly(true)
    .list();
```

**动态更新对比示例：**

```java
// 无 @DynamicUpdate
User user = session.get(User.class, 1L);
user.setName("NewName");
// flush → UPDATE users SET name=?, email=?, phone=?, address=? WHERE id=?
// ↑ 所有列都出现在 SET 子句中

// 有 @DynamicUpdate
User user = session.get(User.class, 1L);
user.setName("NewName");
// flush → UPDATE users SET name=? WHERE id=?
// ↑ 仅包含变更的列
```

## 3.5 ActionQueue 详解

ActionQueue 是 Persistence Context 中暂存待执行 SQL 操作的核心数据结构：

```java
// ActionQueue 内部组成
public class ActionQueue {
    private ExecutableList<EntityInsertAction> insertions;
    private ExecutableList<EntityUpdateAction> updates;
    private ExecutableList<EntityDeleteAction> deletions;
    private ExecutableList<CollectionRecreateAction> collectionCreations;
    private ExecutableList<CollectionUpdateAction> collectionUpdates;
    private ExecutableList<CollectionRemoveAction> collectionRemovals;
    private List<BeforeTransactionCompletionProcess> beforeTransactionProcesses;
    private List<AfterTransactionCompletionProcess> afterTransactionProcesses;
}
```

**各操作类型说明：**

| 操作类型 | 触发条件 | SQL 类型 |
|---------|---------|---------|
| `EntityInsertAction` | `persist()` / `save()` | INSERT |
| `EntityUpdateAction` | 脏检查发现变更 | UPDATE |
| `EntityDeleteAction` | `remove()` / `delete()` | DELETE |
| `CollectionRecreateAction` | 集合被整体替换（新集合赋给 inverse 端） | DELETE + INSERT |
| `CollectionRemoveAction` | 集合元素被移除（仅关联记录删除） | DELETE |
| `CollectionUpdateAction` | 集合元素被新增 | INSERT |

**操作排序算法（拓扑排序）：**

```java
// 按外键依赖关系排序，确保 INSERT 先于引用它的 UPDATE/INSERT
// 示例：
Order order = new Order();
session.persist(order);  // EntityInsertAction[order]

OrderItem item = new OrderItem();
item.setOrder(order);    // item 的 order_id 外键依赖 order 的 id
session.persist(item);   // EntityInsertAction[item]

// flush 时排序：
// 1. EntityInsertAction[order] — 先 INSERT 主表
// 2. EntityInsertAction[item]  — 后 INSERT 从表（外键 order_id 可用）

// EntityInsertAction 实现了 Comparable 接口
// 比较规则：基于外键依赖的深度
public int compareTo(EntityInsertAction other) {
    // 分析自身外键引用了哪些实体
    // 引用越多 → 排序越靠后
}
```

**Batch 合并：**

```java
// 配置 JDBC Batch
hibernate.jdbc.batch_size = 20

// flush 时，ActionQueue 将相邻同类型 SQL 合并为 PreparedStatement batch
session.persist(new User("u1"));
session.persist(new User("u2"));
// ...
session.persist(new User("u20"));
// flush → 1 次 JDBC executeBatch()（20 条 INSERT 在一个 batch）

// IDENTITY 策略不参与 batch（需要立即获取自增 ID）
```

# 4. Session 获取与管理

## 4.1 openSession() vs getCurrentSession()

```java
// openSession() — 总是创建新的 Session
Session s1 = sessionFactory.openSession();
Session s2 = sessionFactory.openSession();
System.out.println(s1 == s2);  // false（每次返回新对象）
// 需要手动关闭：s1.close(); s2.close();

// getCurrentSession() — 获取当前上下文绑定的 Session
Session s3 = sessionFactory.getCurrentSession();
Session s4 = sessionFactory.getCurrentSession();
System.out.println(s3 == s4);  // true（同一上下文同一实例）
// 事务提交后自动关闭，无需手动 close()
```

**核心区别对比：**

| 特性 | openSession() | getCurrentSession() |
|------|---------------|---------------------|
| 每次调用 | 创建新 Session | 返回上下文绑定的 Session |
| 关闭 | 需手动 close() | 事务结束时自动关闭 |
| 事务要求 | 无（可无事务使用） | 必须在事务中使用 |
| 用途 | 独立、手动管理生命周期 | 绑定到事务上下文 |
| 配置 | 无需特殊配置 | 需配置 `current_session_context_class` |

## 4.2 CurrentSessionContext 策略

### 4.2.1 thread（ThreadLocalSessionContext）

Session 绑定到当前线程，事务提交后自动关闭：

```java
// hibernate.cfg.xml 配置
<property name="hibernate.current_session_context_class">thread</property>

// 或 Java 配置
configuration.setProperty("hibernate.current_session_context_class", "thread");

// 使用示例
Session session = sessionFactory.getCurrentSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);

tx.commit();  // 事务提交 → Session 自动 flush + close
// 注意：此后不能再次使用 session（已关闭）

// 下一个事务获取新的 Session
Session session2 = sessionFactory.getCurrentSession();  // 新的 Session 实例
Transaction tx2 = session2.beginTransaction();
// ...
tx2.commit();
```

**ThreadLocalSessionContext 核心逻辑：**

```java
public class ThreadLocalSessionContext implements CurrentSessionContext {
    private static final ThreadLocal<Session> CONTEXT_TL = new ThreadLocal<>();
    
    @Override
    public Session currentSession() {
        Session current = CONTEXT_TL.get();
        if (current == null) {
            Session session = sessionFactory.openSession();
            CONTEXT_TL.set(session);
            return session;
        }
        return current;
    }
    
    // 事务完成后，由 Hibernate 内部自动调用
    public static void unbind(SessionFactory factory) {
        CONTEXT_TL.remove();
    }
}
```

### 4.2.2 jta（JTASessionContext）

适用于 Java EE 容器环境（WildFly/JBoss、WebLogic 等），Session 绑定到当前 JTA 事务：

```xml
<!-- persistence.xml 配置 -->
<persistence-unit name="myPU" transaction-type="JTA">
    <jta-data-source>java:/MyDS</jta-data-source>
    <properties>
        <property name="hibernate.current_session_context_class" 
                  value="jta"/>
        <property name="hibernate.transaction.jta.platform" 
                  value="org.hibernate.service.jta.platform.internal.JBossAppServerJtaPlatform"/>
    </properties>
</persistence-unit>
```

```java
// 容器管理中 Session 的获取
@PersistenceContext
private EntityManager entityManager;

// 同一 JTA 事务内多次获取返回同一 Session
Session s1 = entityManager.unwrap(Session.class);
Session s2 = entityManager.unwrap(Session.class);
System.out.println(s1 == s2);  // true

// JTA 事务由容器管理，应用代码不手动 begin/commit
```

### 4.2.3 managed（ManagedSessionContext）

需要开发者手动管理 Session 的绑定与解绑：

```java
// 配置
<property name="hibernate.current_session_context_class">managed</property>

// 使用
Session session = sessionFactory.openSession();
ManagedSessionContext.bind(session);  // 手动绑定到当前线程

// 后续代码可通过 getCurrentSession() 获取
Session current = sessionFactory.getCurrentSession();  // 与上面同一个 session

// 使用完毕后手动解绑
ManagedSessionContext.unbind(sessionFactory);
session.close();
```

## 4.3 Spring 中的 Session 管理

### SpringSessionContext

Spring 框架提供了自己的 `CurrentSessionContext` 实现——`SpringSessionContext`，与 Spring 事务管理器集成：

```java
// Spring 自动配置（application.properties/yaml）
spring.jpa.properties.hibernate.current_session_context_class = 
    org.springframework.orm.hibernate5.SpringSessionContext

// 或在 Java 配置中
@Bean
public LocalSessionFactoryBean sessionFactory(DataSource dataSource) {
    LocalSessionFactoryBean factoryBean = new LocalSessionFactoryBean();
    factoryBean.setDataSource(dataSource);
    factoryBean.setPackagesToScan("com.example.entity");
    
    Properties props = new Properties();
    props.put("hibernate.current_session_context_class", 
        "org.springframework.orm.hibernate5.SpringSessionContext");
    factoryBean.setHibernateProperties(props);
    
    return factoryBean;
}
```

### @Transactional 管理 Session

```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    public User createUser(String name) {
        // getCurrentSession() 返回当前 Spring 事务绑定的 Session
        Session session = sessionFactory.getCurrentSession();
        User user = new User(name);
        session.persist(user);
        return user;  // @Transactional 提交时自动 flush + commit
    }
    
    public User findUser(Long id) {
        Session session = sessionFactory.getCurrentSession();
        return session.get(User.class, id);
    }
}
```

### HibernateTransactionManager

Spring 的 `HibernateTransactionManager` 负责协调 Hibernate Session 与 Spring 事务：

```java
@Configuration
@EnableTransactionManagement
public class AppConfig {
    
    @Bean
    public LocalSessionFactoryBean sessionFactory(DataSource dataSource) {
        LocalSessionFactoryBean factoryBean = new LocalSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        factoryBean.setPackagesToScan("com.example.entity");
        return factoryBean;
    }
    
    @Bean
    public HibernateTransactionManager transactionManager(SessionFactory sessionFactory) {
        HibernateTransactionManager tm = new HibernateTransactionManager();
        tm.setSessionFactory(sessionFactory);
        return tm;
    }
}
```

### 完整 Spring + Hibernate Session 管理示例

```java
@Configuration
@EnableTransactionManagement
@ComponentScan("com.example")
public class HibernateConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        ds.setUsername("root");
        ds.setPassword("password");
        ds.setMaximumPoolSize(20);
        return ds;
    }
    
    @Bean
    public LocalSessionFactoryBean sessionFactory(DataSource dataSource) {
        LocalSessionFactoryBean sf = new LocalSessionFactoryBean();
        sf.setDataSource(dataSource);
        sf.setPackagesToScan("com.example.entity");
        
        Properties props = new Properties();
        props.put("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        props.put("hibernate.show_sql", "true");
        props.put("hibernate.hbm2ddl.auto", "validate");
        props.put("hibernate.jdbc.batch_size", "20");
        // SpringSessionContext 确保 getCurrentSession() 返回事务绑定的 Session
        props.put("hibernate.current_session_context_class",
            "org.springframework.orm.hibernate5.SpringSessionContext");
        sf.setHibernateProperties(props);
        return sf;
    }
    
    @Bean
    public HibernateTransactionManager transactionManager(SessionFactory sessionFactory) {
        return new HibernateTransactionManager(sessionFactory);
    }
    
    @Bean
    public HibernateTemplate hibernateTemplate(SessionFactory sessionFactory) {
        return new HibernateTemplate(sessionFactory);
    }
}

// ==================== Repository 示例 ====================
@Repository
public class UserRepository {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    public User findById(Long id) {
        Session session = sessionFactory.getCurrentSession();
        return session.get(User.class, id);
    }
    
    public void save(User user) {
        Session session = sessionFactory.getCurrentSession();
        session.persist(user);
    }
    
    public void update(User user) {
        Session session = sessionFactory.getCurrentSession();
        session.merge(user);
    }
    
    public void delete(User user) {
        Session session = sessionFactory.getCurrentSession();
        session.remove(user);
    }
    
    public List<User> findAll() {
        Session session = sessionFactory.getCurrentSession();
        return session.createQuery("from User", User.class).list();
    }
    
    // 批处理：需要手动管理 Session（不能用 getCurrentSession）
    public void batchInsert(List<User> users) {
        Session session = sessionFactory.openSession();
        Transaction tx = session.beginTransaction();
        try {
            for (int i = 0; i < users.size(); i++) {
                session.persist(users.get(i));
                if (i % 50 == 0) {
                    session.flush();
                    session.clear();
                }
            }
            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        } finally {
            session.close();
        }
    }
}

// ==================== Service 示例 ====================
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(String name, String email) {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        userRepository.save(user);
        return user;  // ID 可能在此返回时已填充（取决于 ID 策略）
    }
    
    @Transactional(readOnly = true)
    public List<User> getActiveUsers() {
        return userRepository.findAll()
            .stream()
            .filter(u -> u.isActive())
            .collect(Collectors.toList());
    }
    
    @Transactional(rollbackFor = Exception.class)
    public void transferPoints(Long fromId, Long toId, int points) {
        User from = userRepository.findById(fromId);
        User to = userRepository.findById(toId);
        
        if (from.getPoints() < points) {
            throw new InsufficientPointsException("积分不足");
        }
        
        from.setPoints(from.getPoints() - points);
        to.setPoints(to.getPoints() + points);
        // @Transactional 方法结束 → flush → commit
    }
}
```

# 5. 事务管理

## 5.1 事务基础概念

### ACID 属性

| 属性 | 含义 | Hibernate 中的体现 |
|------|------|-------------------|
| Atomicity（原子性） | 全部成功或全部回滚 | `tx.commit()` / `tx.rollback()` |
| Consistency（一致性） | 事务前后数据满足业务约束 | 级联操作、版本检查 |
| Isolation（隔离性） | 并发事务互不干扰 | 隔离级别 + 乐观锁/悲观锁 |
| Durability（持久性） | 提交后永久保存 | 数据库保证（WAL、Redo Log） |

### 事务隔离级别

```java
// JDBC 定义的四种隔离级别
Connection.TRANSACTION_READ_UNCOMMITTED  // 1 — 最低隔离
Connection.TRANSACTION_READ_COMMITTED    // 2 — 防脏读
Connection.TRANSACTION_REPEATABLE_READ   // 4 — 防脏读 + 不可重复读
Connection.TRANSACTION_SERIALIZABLE      // 8 — 最高隔离，防所有并发问题
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ_UNCOMMITTED | ✓ | ✓ | ✓ |
| READ_COMMITTED | ✗ | ✓ | ✓ |
| REPEATABLE_READ | ✗ | ✗ | ✓（部分数据库不防） |
| SERIALIZABLE | ✗ | ✗ | ✗ |

### 在 Hibernate 中配置隔离级别

```java
// 方式 1：Hibernate 属性
hibernate.connection.isolation = 2  // READ_COMMITTED

// 方式 2：数据源配置（推荐）
// HikariCP 示例
HikariConfig config = new HikariConfig();
config.setTransactionIsolation("TRANSACTION_READ_COMMITTED");

// 方式 3：编程方式
Session session = sessionFactory
    .withOptions()
    .connection()
    .setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ)
    .openSession();
```

## 5.2 JPA 事务管理

```java
// Resource-local 事务（最常用）
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();

try {
    tx.begin();
    
    User user = new User("Alice");
    em.persist(user);
    em.flush();  // 可选：手动同步
    
    tx.commit();  // 成功提交
} catch (Exception e) {
    if (tx.isActive()) {
        tx.rollback();  // 回滚
    }
    throw e;
} finally {
    em.close();
}
```

**事务超时设置：**

```java
// 通过 Hint 设置事务超时（依赖 JPA 提供者实现）
Map<String, Object> props = new HashMap<>();
props.put("javax.persistence.lock.timeout", 5000);  // 5 秒
EntityManager em = emf.createEntityManager(props);
EntityTransaction tx = em.getTransaction();
tx.begin();
// ...
tx.commit();

// Hibernate 原生设置超时（秒）
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
tx.setTimeout(10);  // 10 秒超时
```

**事务回滚标记：**

```java
EntityTransaction tx = em.getTransaction();
tx.begin();

try {
    // 业务逻辑
    doSomething(em);
    
    if (someBusinessCondition) {
        tx.setRollbackOnly();  // 标记为回滚
        // commit 时不会真正提交，而是回滚
    }
    
    tx.commit();  // 如果 setRollbackOnly 被调用，此 commit 执行回滚
} catch (Exception e) {
    tx.rollback();
}

// 检查回滚标记
boolean markedForRollback = tx.getRollbackOnly();
```

**事务中异常的恢复策略：**

```java
// 事务异常后，EntityManager 可能处于不一致状态
EntityTransaction tx = em.getTransaction();
tx.begin();

try {
    User user = new User("Bob");
    user.setEmail(null);  // 假设 NOT NULL 约束
    em.persist(user);
    em.flush();  // → ConstraintViolationException
    tx.commit();
} catch (PersistenceException e) {
    // 事务已标记为 rollback-only，必须回滚
    tx.rollback();
    
    // 重要：异常后是否可重用 EntityManager？
    // 取决于异常类型。通常建议关闭重建：
    em.close();
    em = emf.createEntityManager();  // 新的 EntityManager
    // 然后用新的 em 重试或执行后续操作
}
```

## 5.3 Hibernate Transaction API

```java
// Session.beginTransaction() 返回 Hibernate Transaction
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

try {
    User user = new User("Charlie");
    session.persist(user);
    tx.commit();
} catch (Exception e) {
    if (tx != null && tx.isActive()) {
        tx.rollback();
    }
    throw e;
} finally {
    session.close();
}
```

**Transaction 接口：**

```java
public interface Transaction extends EntityTransaction {
    void commit();
    void rollback();
    boolean isActive();
    void setTimeout(int seconds);
    int getTimeout();
    void registerSynchronization(Synchronization synchronization);
}
```

**保存点（Savepoint）支持：**

```java
// 保存点在事务中创建"回滚点"，可部分回滚
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

try {
    // 第一阶段操作
    session.persist(new User("User1"));
    session.flush();
    
    // 创建保存点
    session.setSavepoint("sp1");
    
    // 第二阶段操作
    session.persist(new User("User2"));
    session.persist(new User("User3"));
    
    // User2 和 User3 的持久化出问题，回到 sp1
    if (someCondition) {
        session.rollbackToSavepoint("sp1");
        // User1 保留，User2 和 User3 被撤销
    }
    
    // 释放保存点（不再需要）
    session.releaseSavepoint("sp1");
    
    session.persist(new User("User4"));
    
    tx.commit();  // User1 + User4 被提交
} catch (Exception e) {
    tx.rollback();  // 全部回滚
} finally {
    session.close();
}
```

**保存点注意事项：**

```java
// 保存点的限制：
// 1. MySQL InnoDB 支持保存点
// 2. JTA 事务中通常不支持保存点
// 3. 保存点不释放已持有的数据库锁
// 4. 保存点名称在同一事务中必须唯一
```

## 5.4 JTA（Java Transaction API）

JTA 用于分布式事务场景，协调多个资源（多个数据库、消息队列等）在同一事务中操作。

### 两阶段提交（2PC）简介

```
Phase 1 — 准备阶段（Prepare）：
  TransactionManager 向所有资源管理器（RM）发送 prepare 指令
  每个 RM 执行操作并写入 Undo/Redo 日志，返回 prepared 或 fail

Phase 2 — 提交阶段（Commit）：
  若所有 RM 都 prepared → TransactionManager 发送 commit 指令
  任一 RM 返回 fail → TransactionManager 发送 rollback 指令
```

### Hibernate 与 JTA 集成配置

```xml
<!-- persistence.xml -->
<persistence-unit name="myJtaPU" transaction-type="JTA">
    <jta-data-source>java:/jdbc/MyDS</jta-data-source>
    <properties>
        <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect"/>
        <property name="hibernate.transaction.jta.platform" 
                  value="org.hibernate.service.jta.platform.internal.JBossAppServerJtaPlatform"/>
        <!-- 或使用 Atomikos -->
        <!-- <property name="hibernate.transaction.jta.platform" 
                 value="org.hibernate.service.jta.platform.internal.AtomikosJtaPlatform"/> -->
    </properties>
</persistence-unit>
```

### 常见 JTA 实现

**Atomikos（独立 JTA 实现，适合 Spring Boot）：**

```xml
<!-- Maven 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jta-atomikos</artifactId>
</dependency>
```

```java
// Spring Boot + Atomikos 多数据源事务配置
@Configuration
public class AtomikosConfig {
    
    @Bean(initMethod = "init", destroyMethod = "close")
    public AtomikosDataSourceBean dataSourceA() {
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        ds.setUniqueResourceName("dsA");
        ds.setXaDataSourceClassName("com.mysql.cj.jdbc.MysqlXADataSource");
        Properties props = new Properties();
        props.setProperty("url", "jdbc:mysql://localhost:3306/db_a");
        props.setProperty("user", "root");
        props.setProperty("password", "password");
        ds.setXaProperties(props);
        ds.setPoolSize(5);
        return ds;
    }
    
    @Bean(initMethod = "init", destroyMethod = "close")
    public AtomikosDataSourceBean dataSourceB() {
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        ds.setUniqueResourceName("dsB");
        ds.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");
        Properties props = new Properties();
        props.setProperty("serverName", "localhost");
        props.setProperty("portNumber", "5432");
        props.setProperty("databaseName", "db_b");
        props.setProperty("user", "postgres");
        props.setProperty("password", "password");
        ds.setXaProperties(props);
        ds.setPoolSize(5);
        return ds;
    }
    
    @Bean
    public JtaTransactionManager jtaTransactionManager() {
        return new JtaTransactionManager();
    }
}
```

```java
// 跨数据源事务示例
@Service
public class CrossDbService {
    
    @Autowired
    @Qualifier("userRepository")
    private UserRepository userRepository;  // DB-A
    
    @Autowired
    @Qualifier("orderRepository")
    private OrderRepository orderRepository;  // DB-B
    
    @Transactional  // JTA 分布式事务
    public void createUserWithOrder(User user, Order order) {
        userRepository.save(user);       // DB-A 操作
        orderRepository.save(order);     // DB-B 操作
        // 两个数据库操作在同一个分布式事务中，原子提交或回滚
    }
}
```

## 5.5 Spring 事务集成

### @Transactional 注解

```java
@Service
public class OrderService {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    // 默认：REQUIRED 传播 + RuntimeException 回滚
    @Transactional
    public void placeOrder(Order order) {
        Session session = sessionFactory.getCurrentSession();
        session.persist(order);
        // 方法返回后 Spring 自动 commit
    }
    
    // 自定义回滚异常
    @Transactional(rollbackFor = {BusinessException.class, SQLException.class})
    public void processPayment(Payment payment) throws BusinessException {
        // 抛 BusinessException 时也回滚
    }
    
    // 指定不触发回滚的异常
    @Transactional(noRollbackFor = {ValidationException.class})
    public void validateOrder(Order order) {
        // 抛 ValidationException 不触发回滚
    }
}
```

### 事务传播行为

| 传播行为 | 说明 | 典型场景 |
|---------|------|---------|
| `REQUIRED` | 支持当前事务，无则新建（默认） | 大多数场景 |
| `REQUIRES_NEW` | 总是新建事务，挂起当前事务 | 日志记录、审计 |
| `NESTED` | 嵌套事务，使用保存点 | 批量处理中的部分回滚 |
| `SUPPORTS` | 支持当前事务，无非事务执行 | 可选事务的查询 |
| `NOT_SUPPORTED` | 非事务执行，挂起当前事务 | 不需要事务的操作 |
| `MANDATORY` | 必须在事务中，否则抛异常 | 强制事务要求的操作 |
| `NEVER` | 非事务执行，存在事务则抛异常 | 禁止在事务中的操作 |

```java
@Service
public class OrderService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void checkout(Long cartId) {
        // 主业务逻辑
        Order order = createOrder(cartId);
        decreaseInventory(order);
        applyCoupon(order);  // 即使 coupons 方法新建事务，此方法仍在自己的事务中
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logCheckoutAttempt(Long cartId) {
        // 日志表独立事务，不被主事务回滚影响
        AuditLog log = new AuditLog("Checkout", cartId);
        sessionFactory.getCurrentSession().persist(log);
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public void processBatch(List<Order> orders) {
        for (Order order : orders) {
            try {
                processSingle(order);  // NESTED
            } catch (Exception e) {
                // 单个失败不影响其他
                logger.warn("Order {} failed: {}", order.getId(), e.getMessage());
            }
        }
    }
}
```

### 只读事务优化

```java
@Service
@Transactional(readOnly = true)
public class ReportService {
    
    // 只读标志传递到 Hibernate：
    // 1. FlushMode 设为 MANUAL（不自动 flush）
    // 2. Session 标记为只读（跳过脏检查）
    // 3. JDBC Connection 设置为只读
    
    public List<Report> generateMonthlyReport(int month) {
        Session session = sessionFactory.getCurrentSession();
        return session.createQuery(
            "from Report r where r.month = :month", Report.class)
            .setParameter("month", month)
            .list();
    }
}

// 读写方法与只读方法组合
@Service
public class MixedService {
    
    @Transactional(readOnly = true)
    public User getUser(Long id) {
        return sessionFactory.getCurrentSession().get(User.class, id);
    }
    
    @Transactional  // 默认 readOnly=false
    public void updateUser(User user) {
        sessionFactory.getCurrentSession().merge(user);
        // flush → UPDATE
    }
    
    @Transactional
    public void processUser(Long id) {
        User user = getUser(id);  // 同一事务内，readOnly=true 被外层覆盖
        user.setStatus(Status.PROCESSED);
        // 方法返回时 flush
    }
}
```

# 6. 锁机制

## 6.1 乐观锁（Optimistic Locking）

乐观锁假设并发冲突概率低，不在数据库层加锁，而是在 UPDATE/DELETE 时通过版本号/字段值检测并发修改。

### 6.1.1 版本号机制（@Version）

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @Version
    private Long version;  // 版本号字段
    
    // 支持的类型：
    // int/Integer, long/Long, short/Short
    // java.sql.Timestamp（高并发不推荐）
    // 不支持：String、Date、float/double
}
```

**版本号递增时机：**

```java
// 操作流程
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);
// SELECT id, name, email, version FROM users WHERE id = 1
// version = 3

user.setName("New Name");
tx.commit();  // flush
// UPDATE users SET name = 'New Name', email = ..., version = 4
// WHERE id = 1 AND version = 3
// ↑ 关键：WHERE 包含 version = 3，带版本号检查

session.close();
```

**并发冲突检测：**

```java
// 线程 A
Session sessionA = sessionFactory.openSession();
User userA = sessionA.get(User.class, 1L);  // version = 5
userA.setName("A修改");

// 线程 B（同时进行）
Session sessionB = sessionFactory.openSession();
User userB = sessionB.get(User.class, 1L);  // version = 5
userB.setEmail("B修改@example.com");

// 线程 B 先提交
sessionB.getTransaction().commit();  
// UPDATE users SET email = 'B修改@example.com', version = 6 
// WHERE id = 1 AND version = 5  → 成功，1 行受影响

// 线程 A 后提交
sessionA.getTransaction().commit();
// UPDATE users SET name = 'A修改', version = 6 
// WHERE id = 1 AND version = 5  → 失败，0 行受影响
// → OptimisticLockException（或 StaleStateException）→ 事务回滚
```

### 6.1.2 非版本号乐观锁

**ALL 策略：WHERE 子句包含所有字段旧值**

```java
@Entity
@OptimisticLocking(type = OptimisticLockType.ALL)
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;
}

// UPDATE 时：
// UPDATE products SET name = ?, price = ?, stock = ?
// WHERE id = ? AND name = ? AND price = ? AND stock = ?
// ↑ 每个字段旧值都出现在 WHERE 中
```

**DIRTY 策略：WHERE 子句只包含修改字段旧值**

```java
@Entity
@OptimisticLocking(type = OptimisticLockType.DIRTY)
public class Document {
    @Id
    private Long id;
    private String title;
    private String content;
    private Date updatedAt;
}

// 只修改 title 时：
// UPDATE documents SET title = ?, updated_at = ?
// WHERE id = ? AND title = ?
// ↑ 只有 title 的旧值出现在 WHERE 中

// 排除某些字段不参与乐观锁
public class Document {
    @OptimisticLock(excluded = true)
    private Date updatedAt;  // updatedAt 不出现在 WHERE 条件中
}
```

### 6.1.3 乐观锁异常处理

```java
@Service
public class OptimisticLockHandler {
    
    // 策略 1：捕获异常，通知用户重试
    public void updateWithUserNotification(User updated) {
        try {
            doUpdate(updated);
        } catch (OptimisticLockException e) {
            throw new BusinessException("数据已被其他用户修改，请刷新后重试", e);
        }
    }
    
    // 策略 2：自动重试（有限次数）
    @Transactional
    public void updateWithRetry(User updated) {
        int maxRetries = 3;
        int attempt = 0;
        
        while (attempt < maxRetries) {
            try {
                Session session = sessionFactory.getCurrentSession();
                User managed = session.get(User.class, updated.getId());
                managed.setName(updated.getName());
                managed.setEmail(updated.getEmail());
                return;  // 成功退出
            } catch (OptimisticLockException e) {
                attempt++;
                if (attempt >= maxRetries) {
                    throw new BusinessException("更新失败，已重试 " + maxRetries + " 次", e);
                }
                // 隐式重试：@Transactional 回滚后，Spring 重新开始事务
            }
        }
    }
    
    // 策略 3：调用 refresh + 重新应用修改
    public void updateWithRefresh(User updated) {
        Session session = sessionFactory.getCurrentSession();
        User managed = session.get(User.class, updated.getId());
        
        try {
            managed.setName(updated.getName());
            session.flush();
        } catch (OptimisticLockException e) {
            // 回滚到数据库最新状态
            session.refresh(managed);
            // 重新应用修改（可能需要合并策略）
            managed.setName(updated.getName());
        }
    }
    
    // 策略 4：使用悲观锁规避乐观锁冲突（见下节）
}
```

**完整乐观锁冲突处理示例（Spring 重试）：**

```java
@Retryable(
    value = {OptimisticLockException.class, StaleObjectStateException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional
public void processOrder(Long orderId, OrderModification modification) {
    Session session = sessionFactory.getCurrentSession();
    Order order = session.get(Order.class, orderId);
    order.apply(modification);
    // commit 时若版本冲突 → 自动重试
}
```

## 6.2 悲观锁（Pessimistic Locking）

悲观锁直接在数据库层面加锁，阻止其他事务修改或读取：

### 6.2.1 LockModeType

```java
// 通过 find() 方法指定锁
Map<String, Object> hints = new HashMap<>();
hints.put("javax.persistence.lock.timeout", 5000);  // 超时 5000ms

User user = entityManager.find(
    User.class, 
    1L, 
    LockModeType.PESSIMISTIC_WRITE,  // SELECT ... FOR UPDATE
    hints
);
```

| LockModeType | SQL 表现 | 说明 |
|-------------|---------|------|
| `PESSIMISTIC_READ` | `SELECT ... FOR SHARE` / `LOCK IN SHARE MODE` | 共享锁，其他事务可读不可写 |
| `PESSIMISTIC_WRITE` | `SELECT ... FOR UPDATE` | 排他锁，其他事务不可读（视隔离级别）也不可写 |
| `PESSIMISTIC_FORCE_INCREMENT` | `SELECT ... FOR UPDATE` + `version + 1` | 排他锁 + 强制版本递增 |

### 6.2.2 使用方式

```java
// 方式 1：entityManager.find(entityClass, id, lockMode)
User user = entityManager.find(User.class, 1L, LockModeType.PESSIMISTIC_WRITE);
// → SELECT * FROM users WHERE id = 1 FOR UPDATE

// 方式 2：Session.byId 方式
User user = session.byId(User.class)
    .with(LockOptions.UPGRADE)
    .load(1L);

// 方式 3：entityManager.lock() 对已有托管实体升级锁级别
User user = session.get(User.class, 1L);
session.lock(user, LockMode.PESSIMISTIC_WRITE);
// → 此方式不会重新查询（除非配置了 refresh）

// 方式 4：Query.setLockMode()
List<User> users = session.createQuery("from User where status = :status", User.class)
    .setParameter("status", Status.ACTIVE)
    .setLockMode(LockModeType.PESSIMISTIC_WRITE)
    .list();
// → SELECT ... FROM users WHERE status = ? FOR UPDATE

// 方式 5：@NamedQuery 注解
@Entity
@NamedQuery(
    name = "User.findByEmail",
    query = "select u from User u where u.email = :email",
    lockMode = LockModeType.PESSIMISTIC_WRITE
)
public class User { ... }
```

### 6.2.3 锁超时

```java
// JPA 标准方式（毫秒）
Map<String, Object> hints = new HashMap<>();
hints.put("javax.persistence.lock.timeout", 3000);  // 3 秒

User user = entityManager.find(User.class, 1L, LockModeType.PESSIMISTIC_WRITE, hints);

// Hibernate 原生方式
User user = session.byId(User.class)
    .with(new LockOptions(LockMode.PESSIMISTIC_WRITE)
        .setTimeOut(LockOptions.WAIT_FOREVER)  // 或毫秒数
    )
    .load(1L);
```

**不同数据库超时行为：**

```sql
-- MySQL：全局/会话变量
SET innodb_lock_wait_timeout = 3;  -- 默认 50 秒

-- PostgreSQL：语句级锁超时
SET lock_timeout = '3s';

-- Oracle：行级等待语法
SELECT * FROM users WHERE id = 1 FOR UPDATE WAIT 3;     -- 等待 3 秒
SELECT * FROM users WHERE id = 1 FOR UPDATE NOWAIT;     -- 立即返回
-- 配置 Hibernate Dialect 以支持 Oracle 的 WAIT/NOWAIT 语法
```

### 6.2.4 锁范围与高级特性

```java
// @Lock 注解（JPA）
@Lock(LockModeType.PESSIMISTIC_WRITE)
User findByEmail(String email);

// Hibernate 扩展：@NamedQuery 中指定 LockOptions
@NamedQuery(
    name = "User.lockByEmail",
    query = "select u from User u where u.email = :email"
)
// 调用时：
User user = session.createNamedQuery("User.lockByEmail", User.class)
    .setParameter("email", email)
    .setLockOptions(new LockOptions(LockMode.PESSIMISTIC_WRITE)
        .setTimeOut(LockOptions.SKIP_LOCKED))  // 跳过已锁定的行
    .uniqueResult();
```

**SKIP LOCKED 模式（队列消费场景）：**

```java
// 跳过已被其他事务锁定的行，适用于高并发任务处理
List<Task> tasks = session.createQuery(
    "from Task where status = :status order by priority", Task.class)
    .setParameter("status", TaskStatus.PENDING)
    .setLockOptions(new LockOptions(LockMode.PESSIMISTIC_WRITE)
        .setTimeOut(LockOptions.SKIP_LOCKED))
    .setMaxResults(10)
    .list();

// 实际 SQL（MySQL 8+ / PostgreSQL 9.5+）：
// SELECT * FROM tasks WHERE status = 'PENDING' 
// ORDER BY priority LIMIT 10 FOR UPDATE SKIP LOCKED
```

**NOWAIT 模式：**

```java
// 遇到锁定立即返回错误，不等待
LockOptions lockOptions = new LockOptions(LockMode.PESSIMISTIC_WRITE)
    .setTimeOut(LockOptions.NO_WAIT);

User user = session.byId(User.class)
    .with(lockOptions)
    .load(1L);
// → SELECT * FROM users WHERE id = 1 FOR UPDATE NOWAIT
// 若行已锁定 → 立即抛出 LockAcquisitionException
```

# 7. 只读会话优化

## 7.1 只读实体

```java
// 方式 1：Session 级别 — 该 Session 加载的所有实体默认只读
Session session = sessionFactory.openSession();
session.setDefaultReadOnly(true);

User u1 = session.get(User.class, 1L);  // 只读
User u2 = session.get(User.class, 2L);  // 只读
// flush 时跳过 u1、u2 的脏检查

// 方式 2：单个实体级别
session.setDefaultReadOnly(false);
User u3 = session.get(User.class, 3L);  // 默认读写
session.setReadOnly(u3, true);          // 将此实体标记为只读

// 方式 3：@Immutable 注解 → 该类所有实例自动只读
@Entity
@Immutable
public class ReadOnlyConfig {
    @Id
    private Long id;
    private String key;
    private String value;
    // 所有操作都自动只读
}

// 方式 4：查询级别
List<User> users = session.createQuery("from User", User.class)
    .setReadOnly(true)
    .list();
// 本次查询返回的所有实体都是只读的
```

## 7.2 只读优化效果

**效果一：跳过脏检查**

```java
// 正常 Session：flush 时遍历所有托管实体对比快照
Session session = sessionFactory.openSession();
session.setDefaultReadOnly(false);
List<User> users = session.createQuery("from User", User.class).list();
// 所有 user 的当前状态与快照逐一比较 → 开销 O(n * m)
session.flush();

// 只读 Session：跳过所有只读实体的脏检查
session.setDefaultReadOnly(true);
List<User> users = session.createQuery("from User", User.class).list();
session.flush();  // 完全跳过脏检查
```

**效果二：跳过快照创建**

```java
// 只读实体不创建快照复制，节省内存
// 正常实体：加载时复制属性数组作为快照 → 双倍内存开销
// 只读实体：不创建快照 → 仅对象本身内存
```

**效果三：跳过 UPDATE 生成**

```java
// 即使修改了只读实体的属性，flush 也不会发出 UPDATE
session.setReadOnly(user, true);
user.setName("Changed");  // 修改
session.flush();          // 不产生 UPDATE 语句
```

## 7.3 @Transactional(readOnly=true) 与 Hibernate 的交互

当 Spring 的 `@Transactional(readOnly=true)` 与 Hibernate 结合使用时，Spring 自动进行以下优化：

```java
// Spring HibernateTransactionManager 内部逻辑（简化）
protected void doBegin(Object transaction, TransactionDefinition definition) {
    // ...
    Session session = obtainSession();
    
    if (definition.isReadOnly()) {
        // 1. FlushMode 设为 MANUAL
        session.setFlushMode(FlushMode.MANUAL);
        
        // 2. Session 标记为只读（跳过脏检查）
        session.setDefaultReadOnly(true);
        
        // 3. JDBC Connection 设为只读
        Connection connection = session.unwrap(Connection.class);
        connection.setReadOnly(true);
        
        // 4. 优化 HQL 查询计划
        //    （某些数据库对只读连接不获取写意向锁）
    }
}
```

```java
// 完整示例
@Service
public class ReadOnlyOptimizationService {
    
    @Transactional(readOnly = true)  // Spring + Hibernate 双重只读优化
    public List<User> getAllUsers() {
        Session session = sessionFactory.getCurrentSession();
        // Session 已自动设为：
        //   - FlushMode.MANUAL
        //   - defaultReadOnly = true
        //   - Connection.readOnly = true
        return session.createQuery("from User", User.class).list();
    }
    
    @Transactional(readOnly = true)
    public ReportDTO generateReport() {
        Session session = sessionFactory.getCurrentSession();
        // 大型只读查询在此上下文中性能最优
        List<User> users = session.createQuery("from User u join fetch u.orders", User.class)
            .list();
        // 如果无意中修改了 user，flush 也不会发出 UPDATE
        return buildReport(users);
    }
}
```

# 8. 会话中的常见模式

## 8.1 Session-per-Request 模式

Web 应用中一个 HTTP 请求对应一个 Session 实例，请求开始时获取，结束时关闭：

```java
// 典型的 Servlet Filter 实现
@WebFilter("/*")
public class SessionPerRequestFilter implements Filter {
    
    private SessionFactory sessionFactory;
    
    @Override
    public void init(FilterConfig config) {
        this.sessionFactory = HibernateUtil.getSessionFactory();
    }
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, 
                         FilterChain chain) throws IOException, ServletException {
        Session session = sessionFactory.openSession();
        Transaction tx = session.beginTransaction();
        
        try {
            req.setAttribute("hibernateSession", session);
            chain.doFilter(req, resp);   // 执行业务逻辑
            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        } finally {
            session.close();  // 请求结束务必关闭
        }
    }
}
```

### Open Session in View（OSIV）模式

**原理：** 请求开始打开 Session，请求完全结束（包括视图渲染）后才关闭。使得视图层可以触发懒加载：

```java
// OpenSessionInViewFilter（Hibernate 提供）
@WebFilter("/*")
public class OpenSessionInViewFilterConfig {
    // org.springframework.orm.hibernate5.support.OpenSessionInViewFilter
}

// Spring Boot 配置（默认开启）
spring.jpa.open-in-view = true  // 默认 true
```

**启用 OSIV 的工作流：**

```java
// Controller 示例
@Controller
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public String viewOrder(@PathVariable Long id, Model model) {
        Order order = orderService.findById(id);  // Session 仍在
        model.addAttribute("order", order);
        return "order-detail";  // 视图渲染
    }
}

// 视图模板（Thymeleaf）
// <html xmlns:th="http://www.thymeleaf.org">
// <body>
//     <h1 th:text="${order.customer.name}">Customer Name</h1>
//     <!-- ↑ 即使 customer 是 LAZY，也能在此触发 SQL 加载 -->
//     <ul>
//         <li th:each="item : ${order.items}">
//             <span th:text="${item.product.name}">Product</span>
//         </li>
//     </ul>
// </body>
// </html>
```

**OSIV 的缺点与风险：**

```java
// 问题 1：数据库连接长时间占用
// Session 在 Filter 层打开 → 贯穿整个 Controller → Service → View 渲染
// 若视图渲染复杂（调用外部服务、生成图片），连接被长期占用

// 问题 2：事务边界模糊
// Commit 一般发生在 Service 层，但 Session 延伸到 View 层
// View 层触发懒加载相当于在事务外部读取数据，无事务保护

// 问题 3：N+1 问题难以控制
// 视图层懒加载往往是无意识的，每访问一个 LAZY 属性就多一条 SQL

// 问题 4：连接池耗尽风险
// 慢视图渲染导致连接不归还，高峰期连接池迅速耗尽
```

**现代最佳实践——关闭 OSIV，显式处理关联：**

```java
// 关闭 OSIV
spring.jpa.open-in-view = false

// 实践 1：DTO 投影（推荐）
@Transactional(readOnly = true)
public OrderDTO findOrderDTO(Long id) {
    Order order = sessionFactory.getCurrentSession().get(Order.class, id);
    // 在 Service 层主动组装所需数据
    OrderDTO dto = new OrderDTO();
    dto.setOrderId(order.getId());
    dto.setCustomerName(order.getCustomer().getName());  // 显式访问
    dto.setItems(order.getItems().stream()
        .map(i -> new OrderItemDTO(i.getProductName(), i.getQuantity()))
        .collect(Collectors.toList()));
    return dto;  // 所有数据已在事务内加载完毕
}

// 实践 2：JPQL FETCH JOIN 明确加载
@Transactional(readOnly = true)
public Order findOrderWithDetails(Long id) {
    return sessionFactory.getCurrentSession()
        .createQuery(
            "select o from Order o " +
            "join fetch o.customer " +
            "join fetch o.items i " +
            "join fetch i.product " +
            "where o.id = :id", Order.class)
        .setParameter("id", id)
        .uniqueResult();
}

// 实践 3：使用 @EntityGraph
@Transactional(readOnly = true)
public Order findOrderWithGraph(Long id) {
    EntityGraph<Order> graph = sessionFactory.getCurrentSession()
        .createEntityGraph(Order.class);
    graph.addAttributeNodes("customer", "items");
    graph.addSubgraph("items").addAttributeNodes("product");
    
    Map<String, Object> hints = new HashMap<>();
    hints.put("javax.persistence.fetchgraph", graph);
    
    return sessionFactory.getCurrentSession().find(Order.class, id, hints);
}
```

## 8.2 Session-per-Application（不推荐）

整个应用生命周期使用一个 Session：

```java
// 严重反模式！！！切勿使用！！！

// 问题：
// 1. 内存泄漏：Persistence Context 无限增长
// 2. 线程安全：多线程并发访问一个 Session
// 3. 脏数据：缓存永远不过期
// 4. Stale State：缓存与数据库严重不一致
// 5. 连接泄漏：Session 持有连接不放

Session globalSession = sessionFactory.openSession();
// 应用所有数据库操作都通过此 Session → 灾难性后果
```

## 8.3 Session-per-Conversation

跨多个 HTTP 请求的扩展会话模式，适用于"向导式"业务（如多步注册、购物车）：

**方案一：Detached Entities + Merge**

```java
// 请求 1：加载实体
@Transactional
public OrderDTO startOrder(long productId) {
    Order order = new Order();
    order.addItem(productRepository.find(productId));
    sessionFactory.getCurrentSession().persist(order);
    return OrderDTO.from(order);  // 返回 DTO 包含 orderId
}
// Session 关闭 → order 变为游离态

// 请求 2～N：更新游离实体（Web 层传回 DTO）
@Transactional
public OrderDTO updateOrder(OrderDTO dto) {
    Order detached = dto.toEntity();  // 从 Web 传回的数据构建游离实体
    Order managed = sessionFactory.getCurrentSession().merge(detached);
    return OrderDTO.from(managed);
}
// merge 将游离实体重新合并

// 最后一请求：提交订单
@Transactional
public void confirmOrder(long orderId) {
    Order order = sessionFactory.getCurrentSession().get(Order.class, orderId);
    order.setStatus(OrderStatus.CONFIRMED);
    // commit → flush
}
```

**方案二：session.lock() 重新附加**

```java
// 请求 1
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
User detached = new User();
detached.setId(1L);
detached.setVersion(2);
session.lock(detached, LockMode.NONE);
// detached 被重新附加到 Persistence Context
// 后续修改会被监控
tx.commit();
session.close();
```

**方案三：扩展持久化上下文（Java EE）**

```java
// Stateful Session Bean + 扩展持久化上下文
@Stateful
public class ShoppingCartBean implements ShoppingCart {
    
    @PersistenceContext(type = PersistenceContextType.EXTENDED)
    private EntityManager em;  // 生命周期与 Bean 一致，跨多个方法调用
    
    private Order order;
    
    public void startOrder() {
        order = new Order();
        em.persist(order);
    }
    
    public void addItem(Long productId, int quantity) {
        // order 仍是托管态（同一 EntityManager）
        Product product = em.find(Product.class, productId);
        order.addItem(product, quantity);
    }
    
    @Remove
    public void checkout() {
        // Bean 销毁时 EntityManager 关闭
        order.setStatus(OrderStatus.CONFIRMED);
    }
}
```

# 9. 连接管理

## 9.1 延迟连接获取

Hibernate 默认采用"延迟连接获取"策略，直到需要执行 SQL 时才从连接池获取 JDBC 连接：

```java
// 以下操作不会立即获取连接：
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
User user = new User("Alice");
session.persist(user);  // 写入 ActionQueue，不获取连接

// 以下操作会触发连接获取：
session.flush();        // flush → 获取连接 → 执行 SQL
tx.commit();            // commit → flush → 获取连接
session.get(User.class, 1L);  // 查询 → 获取连接
```

**连接获取的触发时机：**

| 操作 | 触发连接获取？ | 说明 |
|------|-------------|------|
| `openSession()` | 否 | 仅创建 Session 对象 |
| `beginTransaction()` | 否 | 仅创建 Transaction 对象 |
| `persist()` / `merge()` / `remove()` | 否 | 仅写入 ActionQueue |
| `get()` / `find()` | 是 | 需要执行 SELECT |
| `flush()` | 是 | 需要执行 SQL |
| `commit()` | 是（若需要 flush） | 事务提交可能触发 flush |
| `createQuery().list()` | 是 | 需要执行 SELECT |

## 9.2 连接释放策略

```java
// hibernate.connection.release_mode 配置
// 方式 1：hibernate.cfg.xml
<property name="hibernate.connection.release_mode">after_transaction</property>

// 方式 2：编程配置
configuration.setProperty("hibernate.connection.release_mode", "after_transaction");
```

**四种释放模式：**

```java
// 1. after_transaction（默认，推荐）
//    JTA 环境：事务结束后释放连接
//    Resource-local：事务提交/回滚后释放连接
//    下一次需要连接时重新获取
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
session.get(User.class, 1L);      // 获取连接 → 执行 SELECT
session.get(User.class, 2L);      // 复用同一连接
tx.commit();                      // 释放连接！之后无连接持有
session.get(User.class, 3L);      // 重新获取连接（新事务或自动提交）

// 2. after_statement
//    每条语句执行完毕后立即释放连接
//    适用于 JDBC 连接池资源极度紧张的环境
//    缺点：频繁获取/释放连接 → 性能下降
session.get(User.class, 1L);      // 获取 → SELECT → 释放
session.get(User.class, 2L);      // 获取 → SELECT → 释放

// 3. on_close
//    Session 关闭时才释放连接
//    Session 生命周期内始终持有连接
//    适用于连接数不受限的环境
Session session = sessionFactory.openSession();  // 获取连接
// ... 所有操作使用同一连接
session.close();  // 释放连接

// 4. JTA 环境专用
//    BEFORE_TRANSACTION_COMPLETION — JTA 回调 beforeCompletion 时释放
//    AFTER_TRANSACTION_COMPLETION  — JTA 回调 afterCompletion 时释放
```

## 9.3 连接提供者（ConnectionProvider）

```java
// 默认：DriverManagerConnectionProvider（不推荐生产使用）
configuration.setProperty("hibernate.connection.url", "jdbc:mysql://localhost:3306/db");
configuration.setProperty("hibernate.connection.driver_class", "com.mysql.cj.jdbc.Driver");
configuration.setProperty("hibernate.connection.username", "root");
configuration.setProperty("hibernate.connection.password", "secret");
// → Hibernate 使用 DriverManager.getConnection() 创建连接，无连接池

// 推荐：通过 DataSource 配置（配合连接池）
// HikariCP 示例
@Bean
public DataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:mysql://localhost:3306/db");
    ds.setUsername("root");
    ds.setPassword("secret");
    ds.setMaximumPoolSize(20);
    ds.setMinimumIdle(5);
    ds.setConnectionTimeout(30000);
    ds.setIdleTimeout(600000);
    ds.setMaxLifetime(1800000);
    return ds;
}

@Bean
public LocalSessionFactoryBean sessionFactory(DataSource dataSource) {
    LocalSessionFactoryBean sf = new LocalSessionFactoryBean();
    sf.setDataSource(dataSource);  // 使用连接池数据源
    // Hibernate 内部使用 DatasourceConnectionProvider
    return sf;
}
```

**自定义 ConnectionProvider（特殊场景）：**

```java
public class CustomConnectionProvider implements ConnectionProvider {
    
    @Override
    public Connection getConnection() throws SQLException {
        // 自定义连接获取逻辑（如带审计日志的连接包装）
        Connection raw = DriverManager.getConnection(url, user, password);
        return new AuditedConnection(raw);
    }
    
    @Override
    public void closeConnection(Connection conn) throws SQLException {
        if (conn instanceof AuditedConnection) {
            ((AuditedConnection) conn).getWrapped().close();
        } else {
            conn.close();
        }
    }
    
    @Override
    public boolean supportsAggressiveRelease() {
        return true;
    }
}

// 注册自定义 ConnectionProvider
configuration.getServiceRegistry()
    .getService(ConnectionProvider.class);
// 或通过 hibernate.cfg.xml:
// <property name="hibernate.connection.provider_class">
//     com.example.CustomConnectionProvider
// </property>
```

# 10. 扩展持久化上下文（Java EE）

在 Java EE 环境中，`@PersistenceContext(type = PersistenceContextType.EXTENDED)` 创建扩展持久化上下文，其生命周期超过单次 JTA 事务，与 Stateful Session Bean 绑定：

```java
@Stateful
public class OrderEditorBean {
    
    // EXTENDED：EntityManager 生命周期 = Bean 生命周期（跨多个方法调用和事务）
    @PersistenceContext(type = PersistenceContextType.EXTENDED)
    private EntityManager em;
    
    private Order order;
    
    public void loadOrder(Long id) {
        order = em.find(Order.class, id);  // 托管态
        // 此时没有活跃的 JTA 事务（方法可能无事务注解）
    }
    
    @TransactionAttribute(TransactionAttributeType.REQUIRED)
    public void addItem(Long productId, int qty) {
        // 在 JTA 事务中
        // order 仍由扩展持久化上下文管理
        Product p = em.find(Product.class, productId);
        order.addItem(p, qty);
        // JTA 事务提交 → 脏检查发现 order 变更 → 自动 UPDATE
        // 但 EntityManager 不关闭（EXTENDED）
    }
    
    @TransactionAttribute(TransactionAttributeType.REQUIRED)
    public void updateShipping(String address) {
        // 新的 JTA 事务
        order.setShippingAddress(address);
        // 事务提交 → 自动 UPDATE
    }
    
    @Remove
    public void confirm() {
        order.setStatus(OrderStatus.CONFIRMED);
        // Bean 销毁 → EntityManager 关闭
    }
    
    @Remove
    public void cancel() {
        em.close();  // 显式关闭
    }
}
```

**与传统 Detached + Merge 模式的对比：**

| 维度 | Detached + Merge | Extended Persistence Context |
|------|-----------------|------------------------------|
| 托管性 | 实体在方法间变为游离态 | 实体在 Bean 生命周期内保持托管 |
| 数据同步 | 需手动 merge 回持久化上下文 | 脏检查自动同步（事务提交时） |
| 乐观锁 | merge 时需注意版本号 | 版本号持续跟踪 |
| 复杂度 | 需理解游离态/merge 语义 | 需理解扩展上下文与事务交互 |
| 适用场景 | 无状态服务（Spring） | 有状态对话（Java EE） |
| 内存 | 短生命周期上下文 | 长生命周期上下文 |

**注意事项：**

```java
// 扩展持久化上下文不自动 flush，必须显式调用或在事务提交时
@PersistenceContext(type = PersistenceContextType.EXTENDED)
private EntityManager em;

public void someMethod() {
    Order order = em.find(Order.class, 1L);
    order.setStatus(Status.SHIPPED);
    // 没有事务 → 修改不会自动同步到数据库
    // 即使有事务但方法正常返回（无异常）→ 也不会自动 flush
    // 必须：1) 事务提交时自动 flush，或 2) 显式 em.flush()
}
```
