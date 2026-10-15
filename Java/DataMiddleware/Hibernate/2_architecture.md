# 1. Hibernate 整体架构

## 1.1 分层架构概览

Hibernate 采用了经典的分层架构设计，从上到下依次划分为五个主要层次：

```
┌─────────────────────────────────────────────────────────┐
│                   应用层 (Application)                     │
│         业务代码 / Service / Repository / DAO             │
│          使用 Session API 或 JPA EntityManager             │
├─────────────────────────────────────────────────────────┤
│                持久化层 (Persistence Layer)                 │
│   Session / EntityManager / Transaction / Query           │
│   一级缓存 (PersistenceContext) / 脏检查 (Dirty Check)     │
│   事件系统 (Event System) / 拦截器 (Interceptor)           │
├─────────────────────────────────────────────────────────┤
│                 映射层 (Mapping Layer)                      │
│          Metadata / EntityPersister / Mapping               │
│   注解处理器 (AnnotationBinder) / XML 解析器                 │
│   HBM 元模型 (DomainModel) / 关系映射 (Association)         │
├─────────────────────────────────────────────────────────┤
│              数据库交互层 (Database Access Layer)            │
│       Dialect (方言) / SQL 生成 / JDBC 抽象                  │
│         JdbcServices / ConnectionProvider                   │
│        类型系统 (Type System) / 参数绑定 (Binder)            │
├─────────────────────────────────────────────────────────┤
│              基础设施层 (Infrastructure Layer)               │
│    二级缓存 (L2 Cache) / 查询缓存 (Query Cache)             │
│    连接池 (Connection Pool) / 统计 (Statistics)             │
│    事务管理器 (TransactionManager) / JMX 监控                │
└─────────────────────────────────────────────────────────┘
```

**各层职责说明：**

- **应用层**：业务代码直接交互的层面。开发者通过调用 `Session` 或 `EntityManager` 的 API 完成持久化操作，无需关心底层 SQL 生成和 JDBC 执行细节。
- **持久化层**：核心 API 层，负责管理实体对象的生命周期（transient → persistent → detached → removed），维护一级缓存（持久化上下文），实现脏检查机制和自动刷新。
- **映射层**：负责将 Java 对象模型与数据库关系模型建立映射。解析注解（`@Entity`, `@Table`, `@Column`, `@OneToMany` 等）和 XML 映射文件，构建运行时元数据（`EntityPersister`, `CollectionPersister`）。
- **数据库交互层**：封装 JDBC 操作，通过 `Dialect` 方言适配不同数据库的 SQL 语法差异，生成并执行 SQL 语句，管理 JDBC 连接的生命周期。
- **基础设施层**：提供横切关注点的支持，包括二级缓存（跨 Session 共享实体）、查询缓存、连接池集成、JMX 监控统计等。

## 1.2 核心架构图

### 1.2.1 组件关系总览

```
                     ┌──────────────┐
                     │ Configuration │  (或 MetadataSources)
                     └──────┬───────┘
                            │ 构建
                 ┌──────────▼──────────┐
                 │ StandardService     │
                 │    Registry         │
                 └──────────┬──────────┘
                            │ 提供
              ┌─────────────▼─────────────┐
              │      SessionFactory        │  (线程安全, 重量级, 单例)
              │  (EntityManagerFactory)    │
              └─────────────┬─────────────┘
                            │ 创建
              ┌─────────────▼─────────────┐
              │         Session            │  (非线程安全, 轻量级)
              │     (EntityManager)        │
              └──────┬──────────┬─────────┘
                     │          │
          ┌──────────▼──┐  ┌───▼───────────┐
          │ Transaction  │  │    Query       │
          │              │  │ (Criteria)     │
          └──────────────┘  └───────────────┘
```

### 1.2.2 数据流

应用代码通过 Session API 发起持久化操作，经过拦截器/事件系统拦截后，进入映射层将实体状态变更转换为 SQL 语句，再经 Dialect 方言适配后，通过 JDBC 驱动执行 SQL 并返回结果：

```
应用代码 → Session API → Interceptor / Event → Mapping (EntityPersister)
→ SQL Generation (Dialect) → JDBC → Database
```

### 1.2.3 请求处理完整链路

以 `session.persist(entity)` 为例，一条持久化请求在 Hibernate 内部的完整处理链路如下：

1. **应用层发起** - 调用 `Session#persist(entity)`
2. **Session 门面** - `SessionImpl#persist()` 委托给内部 `SessionImplementor`
3. **拦截器前置处理** - 若配置了 `Interceptor`，调用 `Interceptor#onSave()`
4. **事件触发** - 触发 `PersistEvent`，由注册的 `PersistEventListener` 处理
5. **持久化上下文** - 检查一级缓存中是否已存在该实体，生成持久化标识
6. **映射元数据** - `EntityPersister` 获取表名、字段映射、主键生成策略
7. **主键生成** - `IdentifierGenerator` 生成主键值（自增/序列/UUID/雪花等）
8. **ActionQueue 入队** - 创建 `EntityInsertAction` 并加入 `ActionQueue`
9. **Flush 时机** - 事务提交或显式 `flush()` 触发
10. **SQL 生成** - `EntityPersister` 结合 `Dialect` 生成 INSERT SQL
11. **参数绑定** - `Binder` 将实体属性值绑定到 JDBC `PreparedStatement` 参数
12. **JDBC 执行** - `JdbcServices` 通过 `ConnectionProvider` 获取连接并执行
13. **拦截器后置处理** - 调用 `Interceptor#onSave()` 的后置逻辑（如 `afterTransactionCompletion`）
14. **返回结果** - 将数据库生成的值回填到实体对象

# 2. 核心接口与组件

## 2.1 Configuration

### 2.1.1 职责

`Configuration` 是 Hibernate 传统的配置引导类（Hibernate 5 及之前版本的主入口），负责加载和解析映射配置（`hibernate.cfg.xml`、`*.hbm.xml` 或注解类），并最终构建 `SessionFactory`。

从 Hibernate 5 开始，推荐使用 `MetadataSources` 替代 `Configuration`，但 `Configuration` 仍然可用。

### 2.1.2 configure() 方法

```java
// 加载类路径下的 hibernate.cfg.xml
Configuration cfg = new Configuration().configure();

// 加载指定路径的配置文件
Configuration cfg = new Configuration().configure("hibernate-mysql.cfg.xml");

// 编程方式配置
Configuration cfg = new Configuration()
    .setProperty("hibernate.connection.url", "jdbc:mysql://localhost:3306/mydb")
    .setProperty("hibernate.connection.driver_class", "com.mysql.cj.jdbc.Driver")
    .setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect")
    .setProperty("hibernate.hbm2ddl.auto", "update")
    .setProperty("hibernate.show_sql", "true");
```

### 2.1.3 addAnnotatedClass() / addPackage()

```java
Configuration cfg = new Configuration()
    .addAnnotatedClass(User.class)
    .addAnnotatedClass(Order.class)
    .addPackage("com.example.entity")
    .configure();
```

### 2.1.4 从 XML 加载配置的流程

1. 解析 `hibernate.cfg.xml` 中的 `<session-factory>` 节点
2. 提取所有 `<property>` 属性键值对
3. 提取所有 `<mapping>` 引用路径
4. 解析每个 `*.hbm.xml` 映射文件，构建 `PersistentClass` 模型
5. 合并编程方式设置的属性

### 2.1.5 属性合并策略

属性优先级从高到低：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 (最高) | 编程方式 `setProperty()` | 代码中直接设置的属性 |
| 2 | `hibernate.properties` 文件 | 类路径下的属性文件 |
| 3 | `hibernate.cfg.xml` | XML 配置文件 |
| 4 (最低) | Hibernate 默认值 | 框架内置默认值 |

### 2.1.6 配置验证阶段

在 `buildSessionFactory()` 调用时，`Configuration` 会执行以下验证：

- 检查必需的数据库连接属性是否完整
- 验证 `hibernate.dialect` 是否与驱动版本兼容
- 检查映射的实体类是否可加载、是否有 `@Id` 注解
- 校验关联映射的双向关系一致性
- 检查命名查询语法合法性

## 2.2 StandardServiceRegistry / ServiceRegistry

### 2.2.1 服务注册中心概念

`ServiceRegistry` 是 Hibernate 的服务定位器（Service Locator），负责管理和发现框架内部的各种服务组件（如连接提供者、事务工厂、JDBC 服务、方言解析器等）。它替代了早期版本中直接依赖具体实现类的硬编码方式，实现了松耦合和可扩展性。

### 2.2.2 层次结构

```
BootstrapServiceRegistry（引导服务注册中心）
    ├── ClassLoaderService          —— 类加载服务
    ├── IntegratorService           —— 集成器（Envers、Bean Validation 等）
    └── StrategySelector            —— 策略选择器
        │
        └── StandardServiceRegistry（标准服务注册中心）
            ├── ConnectionProvider   —— 数据库连接提供者
            ├── TransactionFactory   —— 事务工厂
            ├── JdbcServices         —— JDBC 服务
            ├── DialectResolver      —— 方言解析器
            ├── CachingService       —— 缓存服务
            └── ...
                │
                └── SessionFactoryServiceRegistry（会话工厂服务注册中心）
                    ├── EventListenerRegistry  —— 事件监听器注册表
                    ├── StatisticsImplementor   —— 统计信息实现
                    └── ...
```

### 2.2.3 StandardServiceRegistryBuilder 用法

```java
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .applySettings(configuration.getProperties())  // 从 Configuration 合并属性
    .applySetting("hibernate.connection.datasource", dataSource)
    .applySetting("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect")
    .addService(MyCustomConnectionProvider.class, new MyCustomConnectionProvider())
    .build();
```

### 2.2.4 服务发现机制

Hibernate 使用 Java 标准 `java.util.ServiceLoader` 机制发现扩展服务实现：

```
META-INF/services/org.hibernate.integrator.spi.Integrator
META-INF/services/org.hibernate.boot.registry.selector.StrategyRegistrationProvider
```

代码示例（Hibernate 内部逻辑）：

```java
// 伪代码：服务发现核心逻辑
ServiceLoader<Integrator> loader = ServiceLoader.load(Integrator.class);
for (Integrator integrator : loader) {
    integrator.integrate(metadata, sessionFactory, sessionFactoryServiceRegistry);
}
```

### 2.2.5 关键内置服务

| 服务接口 | 默认实现 | 职责 |
|----------|----------|------|
| `ConnectionProvider` | `DatasourceConnectionProviderImpl` | 提供 JDBC 连接 |
| `TransactionFactory` | `JdbcResourceLocalTransactionCoordinatorBuilderImpl` | 创建事务协调器 |
| `JdbcServices` | `JdbcServicesImpl` | 封装 JDBC 操作和元数据 |
| `DialectResolver` | `StandardDialectResolver` | 自动检测数据库方言 |
| `CachingService` | `CachingServiceImpl` | 二级缓存管理 |
| `BatchBuilder` | `BatchBuilderImpl` | JDBC 批处理 |

### 2.2.6 自定义服务注册

```java
public class MyCustomService implements Service {
    // 实现自定义服务逻辑
}

// 注册自定义服务
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .addService(MyCustomService.class, new MyCustomService())
    .build();
```

### 2.2.7 服务的延迟初始化与生命周期管理

Hibernate 中的服务采用延迟初始化策略（Lazy Init）。`ServiceRegistry` 不会在构建时立刻初始化所有服务，而是在首次使用时才创建服务实例。服务实现接口 `org.hibernate.service.Service`，其生命周期方法包括：

- `configure(Map configurationValues)` - 配置阶段，传递配置参数
- `start()` - 启动阶段，建立资源连接
- `stop()` - 停止阶段，释放资源

当 `ServiceRegistry` 销毁时，会逆序调用所有已初始化服务的 `stop()` 方法。

## 2.3 Metadata / MetadataSources

### 2.3.1 MetadataSources 替代 Configuration 的原因

`MetadataSources` 是 Hibernate 5 引入的引导 API，相比传统的 `Configuration` 有显著改进：

- **关注点分离**：`MetadataSources` 只负责收集映射源（类、XML 文件、包），而配置属性则通过 `StandardServiceRegistry` 管理
- **API 清晰**：`Configuration` 既管理配置属性又收集映射，职责混乱
- **扩展性**：通过 `MetadataBuilder` 可以更细粒度控制元数据构建过程
- **解耦**：`Metadata` 的构建独立于 `SessionFactory` 的创建

### 2.3.2 Metadata 包含的信息

| 信息类别 | 说明 |
|----------|------|
| 实体映射（Entity Bindings） | 实体类到表的映射、属性到列的映射 |
| 命名查询（Named Queries） | `@NamedQuery`、`@NamedNativeQuery` 定义的查询 |
| 过滤器定义（Filter Definitions） | `@FilterDef` 定义的数据过滤器 |
| 类型定义（Type Definitions） | `@TypeDef` 定义的自定义类型别名 |
| 集合映射（Collection Bindings） | `@OneToMany`、`@ManyToMany` 等集合映射 |
| 组件/嵌入（Component Bindings） | `@Embeddable` 组件映射 |
| 继承策略 | `@Inheritance(strategy = ...)` 配置 |
| 命名策略结果 | 应用 `ImplicitNamingStrategy` 和 `PhysicalNamingStrategy` 后的最终名称 |

### 2.3.3 MetadataImplementor 内部实现

`MetadataImplementor` 是 `Metadata` 的内部扩展接口，增加了运行时所需的方法：

```java
// 伪代码示意
public interface MetadataImplementor extends Metadata {
    EntityPersister getEntityPersister(String entityName);
    CollectionPersister getCollectionPersister(String role);
    Type getIdentifierType(String entityName);
    Map<String, NamedQueryDefinition> getNamedQueryDefinitions();
    Map<String, FilterDefinition> getFilterDefinitions();
    // ... 更多内部方法
}
```

### 2.3.4 InFlightMetadataCollector - 元数据收集过程

在 `MetadataBuilder.build()` 阶段，Hibernate 使用 `InFlightMetadataCollector` 逐步收集元数据：

1. **扫描阶段**：`MetadataSources` 注册的所有源（注解类、hbm.xml、orm.xml）被扫描
2. **绑定阶段**：`AnnotationBinder` 或 `HbmBinder` 将映射信息写入 `InFlightMetadataCollector`
3. **验证阶段**：检查实体映射的完整性（必须有主键、关联类型匹配等）
4. **完成阶段**：`InFlightMetadataCollector` 被冻结为不可变的 `Metadata`

### 2.3.5 MetadataBuilder 配置选项

```java
Metadata metadata = new MetadataSources(serviceRegistry)
    .addAnnotatedClass(User.class)
    .addAnnotatedClass(Order.class)
    .getMetadataBuilder()
    .applyImplicitNamingStrategy(ImplicitNamingStrategyJpaCompliantImpl.INSTANCE)
    .applyPhysicalNamingStrategy(new PhysicalNamingStrategyStandardImpl())
    .applyAttributeConverter(MyConverter.class)
    .enableNewIdentifierGeneratorSupport(true)
    .applyIdGenerationTypeInterpreter(new FallbackInterpreter())
    .build();
```

### 2.3.6 启动流程（Hibernate 5+）

```
MetadataSources → MetadataBuilder → Metadata → SessionFactoryBuilder → SessionFactory
```

代码示例：

```java
// 1. 构建服务注册中心
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .configure("hibernate.cfg.xml")
    .build();

// 2. 收集映射源并构建 Metadata
Metadata metadata = new MetadataSources(registry)
    .addAnnotatedClass(User.class)
    .addAnnotatedClass(Order.class)
    .getMetadataBuilder()
    .build();

// 3. 构建 SessionFactory
SessionFactory sessionFactory = metadata
    .getSessionFactoryBuilder()
    .applyInterceptor(new MyInterceptor())
    .applyBeanManager(beanManager)  // CDI
    .build();
```

## 2.4 SessionFactory

### 2.4.1 线程安全与不可变性

`SessionFactory` 是 Hibernate 中最重要的对象之一，具有以下特征：

- **线程安全**：多个线程可以并发访问同一个 `SessionFactory` 实例，获取各自的 `Session`
- **不可变性**：`SessionFactory` 一旦构建完成，其内部元数据（实体映射、查询计划缓存等）是不可变的，保证了线程安全
- **创建成本高**：构建过程需要解析所有映射、初始化缓存、建立连接池等，耗时较长
- **通常为单例**：一个应用通常只为每个数据库创建一个 `SessionFactory` 实例

### 2.4.2 内部结构

```
SessionFactory
├── EntityPersister 缓存（每个实体类一个）
│   ├── SingleTableEntityPersister（单表实体）
│   ├── JoinedSubclassEntityPersister（JOIN 继承）
│   └── UnionSubclassEntityPersister（UNION 继承）
├── 集合元数据缓存（CollectionPersister）
│   ├── OneToManyPersister
│   └── ManyToManyPersister
├── 查询计划缓存（QueryPlanCache）
│   ├── HQL/JPQL 查询计划
│   ├── Criteria 查询计划
│   └── Native SQL 查询计划
├── 类型关联（TypeResolver）
├── 命名查询注册表
├── 过滤器定义注册表
└── SessionFactoryServiceRegistry
```

### 2.4.3 SessionFactoryImplementor 内部接口

```java
public interface SessionFactoryImplementor 
    extends SessionFactory, EntityManagerFactory {
    EntityPersister getEntityPersister(String entityName);
    CollectionPersister getCollectionPersister(String role);
    TypeResolver getTypeResolver();
    QueryPlanCache getQueryPlanCache();
    Dialect getDialect();
    // ... 更多内部方法
}
```

### 2.4.4 getCurrentSession() vs openSession()

| 特性 | `getCurrentSession()` | `openSession()` |
|------|----------------------|-----------------|
| 创建策略 | 绑定到当前事务上下文中复用 | 每次调用都创建新 Session |
| 事务管理 | 必须存在活动事务 | 不强制要求事务 |
| 关闭方式 | 事务提交时自动关闭 | 必须手动调用 `close()` |
| 适用场景 | Web 应用、JTA 环境 | 批处理、后台任务 |
| 配置要求 | 需配置 `hibernate.current_session_context_class` | 无需特殊配置 |

```java
// hibernate.cfg.xml 配置当前 Session 上下文策略
// <property name="hibernate.current_session_context_class">thread</property>

// getCurrentSession() 使用示例
Session session = sessionFactory.getCurrentSession();
// 不需要手动 close()，事务提交后自动关闭

// openSession() 使用示例
Session session = sessionFactory.openSession();
try {
    // 执行操作
} finally {
    session.close();  // 必须手动关闭
}
```

### 2.4.5 获取统计信息

```java
SessionFactory sessionFactory = metadata.getSessionFactoryBuilder().build();
Statistics statistics = sessionFactory.getStatistics();
statistics.setStatisticsEnabled(true);

// 运行时获取数据
long queryCount = statistics.getQueryExecutionCount();
long hitCount = statistics.getSecondLevelCacheHitCount();
long missCount = statistics.getSecondLevelCacheMissCount();
```

### 2.4.6 SessionFactory 销毁与资源释放

```java
sessionFactory.close();  // 释放所有资源、关闭连接池、清理缓存
```

### 2.4.7 SessionFactory 与 JPA EntityManagerFactory 的关系

Hibernate 的 `SessionFactory` 实现了 JPA 的 `EntityManagerFactory` 接口（通过 `SessionFactoryImplementor`）。因此 Hibernate 可以同时作为 Hibernate Native API 和 JPA 标准的实现提供者：

```java
// 作为 JPA EntityManagerFactory 使用
EntityManagerFactory emf = new MetadataSources(registry)
    .addAnnotatedClass(User.class)
    .getMetadataBuilder()
    .build()
    .getSessionFactoryBuilder()
    .build();  // 返回 SessionFactory 但实现了 EntityManagerFactory

EntityManager em = emf.createEntityManager();
// 等价于
Session session = ((SessionFactory) emf).openSession();
```

## 2.5 Session

### 2.5.1 Session 不是线程安全的（绑定到线程）

`Session` 是非线程安全的，设计上应绑定到当前线程使用（通常通过 `ThreadLocal` 或事务上下文绑定）。多个线程不能同时操作同一个 `Session` 实例，否则会导致数据错乱或并发异常。

### 2.5.2 Session 作为一级缓存（持久化上下文）

`Session` 维护了一个持久化上下文（PersistenceContext），作为一级缓存：

- **作用范围**：单个 Session 生命周期内有效
- **自动启用**：无法关闭（区别于二级缓存）
- **保证同一 Session 内同一主键返回同一 Java 对象引用**
- **实现 REPEATABLE READ 语义**：同一 Session 内多次查询同一实体，只执行一次 SQL

```java
User user1 = session.get(User.class, 1L);  // 发送 SQL
User user2 = session.get(User.class, 1L);  // 不发送 SQL，直接从一级缓存获取
System.out.println(user1 == user2);         // true，同一引用
```

### 2.5.3 Session 内部结构

```
SessionImpl (SessionImplementor)
├── PersistenceContext（持久化上下文 - 一级缓存）
│   ├── EntityEntryContext  —— 实体状态条目（managed/detached/deleted）
│   ├── entityByKey         —— Map<EntityKey, Object> 实体缓存
│   └── entitySnapshotsByKey—— Map<EntityKey, Object[]> 旧值快照（脏检查用）
├── ActionQueue（操作队列）
│   ├── InsertAction[]      —— 待插入操作
│   ├── UpdateAction[]      —— 待更新操作
│   ├── DeleteAction[]      —— 待删除操作
│   └── CollectionAction[]  —— 集合操作
├── EventManager（事件管理器）
│   ├── PreInsertEventListener[]
│   ├── PostInsertEventListener[]
│   ├── PreUpdateEventListener[]
│   └── ...
└── JdbcCoordinator（JDBC 协调器）
    └── LogicalConnection  —— 逻辑连接句柄
```

### 2.5.4 SessionImplementor 内部接口

```java
public interface SessionImplementor extends Session {
    PersistenceContext getPersistenceContext();
    ActionQueue getActionQueue();
    EventManager getEventManager();
    JdbcCoordinator getJdbcCoordinator();
    EntityPersister getEntityPersister(String entityName);
    // ... 更多内部方法
}
```

### 2.5.5 openSession() 内部流程详解

```
SessionFactoryImpl.openSession()
    │
    ├── 1. 从 ConnectionProvider 获取 JDBC 连接（延迟获取，此时可能为 null）
    ├── 2. 创建 JdbcCoordinator（包装 JDBC Connection）
    ├── 3. 创建 TransactionCoordinator（事务协调器）
    ├── 4. 创建 EventManager（事件管理器）
    ├── 5. 创建 PersistenceContext（持久化上下文仓库）
    ├── 6. 创建 ActionQueue（操作队列）
    ├── 7. 组装为 SessionImpl 实例
    └── 8. 返回 Session
```

### 2.5.6 Session 关闭后的状态

调用 `session.close()` 后，Session 进入不可用状态：

- 一级缓存被清空
- JDBC 连接归还给连接池
- 所有持久化对象变为脱管（detached）状态
- 任何后续操作都会抛出 `SessionException("Session is closed!")`

### 2.5.7 Session 与数据库连接的关系（延迟获取连接）

Hibernate 不会在 `openSession()` 时立即从连接池获取 JDBC 连接，而是采用延迟获取策略：

- **连接获取时机**：首次需要执行 SQL 时才获取连接（如第一次 `get()`/`load()`/`persist()` 调用，或显式 `beginTransaction()`）
- **无事务操作**：如果整个 Session 生命周期内没有任何 SQL 操作，则不会消耗任何数据库连接
- **连接释放**：`session.close()` 时归还连接（如果此时连接尚未释放）
- **连接释放时机优化**：Hibernate 5+ 支持在事务提交后立即释放连接（`hibernate.connection.release_mode=after_transaction`），避免长期占用

## 2.6 Transaction

### 2.6.1 抽象事务接口

Hibernate 的 `Transaction` 接口（`org.hibernate.Transaction`）提供了统一的事务抽象，屏蔽底层 JDBC 本地事务与 JTA 分布式事务的差异。

```java
public interface Transaction {
    void begin();
    void commit();
    void rollback();
    void setTimeout(int seconds);
    int getTimeout();
    boolean isActive();
    boolean wasCommitted();
    boolean wasRolledBack();
    void markRollbackOnly();
    boolean isRollbackOnly();
    void registerSynchronization(Synchronization synchronization);
}
```

### 2.6.2 JdbcResourceLocalTransactionCoordinator（JDBC 本地事务）

`JdbcResourceLocalTransactionCoordinator` 封装了 JDBC 原生的 `Connection.commit()` / `Connection.rollback()`，适用于单数据库场景：

```java
Session session = sessionFactory.openSession();
Transaction tx = null;
try {
    tx = session.beginTransaction();
    session.persist(new User("张三"));
    session.persist(new User("李四"));
    tx.commit();       // 调用 Connection.commit()
} catch (Exception e) {
    if (tx != null) {
        tx.rollback(); // 调用 Connection.rollback()
    }
    throw e;
} finally {
    session.close();
}
```

### 2.6.3 JtaTransactionCoordinator（JTA 分布式事务）

`JtaTransactionCoordinator` 集成了 JTA 事务管理器（如 JBoss、WebLogic、Atomikos、Bitronix），适用于跨多个数据源/消息队列的分布式事务场景：

```xml
<!-- hibernate.cfg.xml -->
<property name="hibernate.transaction.jta.platform">
    org.hibernate.engine.transaction.jta.platform.internal.JBossAppServerJtaPlatform
</property>
```

```java
// JTA 环境下使用 Hibernate
Session session = sessionFactory.getCurrentSession();
// 事务由 JTA 容器管理，无需手动 begin/commit
session.persist(new User("张三"));
// 事务边界由 EJB/JTA 容器决定
```

### 2.6.4 Transaction 的生命周期

```
NOT_ACTIVE ──begin()──► ACTIVE ──commit()/rollback()──► NOT_ACTIVE
```

- `begin()` - 启动事务，底层获取 JDBC 连接并设置 `autoCommit = false`
- `commit()` - 触发 flush，提交 JDBC 连接，释放资源
- `rollback()` - 回滚 JDBC 连接上的所有变更，释放资源

### 2.6.5 事务超时控制

```java
Transaction tx = session.beginTransaction();
tx.setTimeout(30);  // 30 秒超时，超过则自动回滚
```

### 2.6.6 回滚标记

```java
try {
    session.persist(entity);
    // 某条件触发强制回滚
    if (someCondition) {
        tx.markRollbackOnly();  // 标记事务必须回滚
    }
    // 后续即使 commit()，也会实际执行 rollback()
    tx.commit();  // 内部检查 isRollbackOnly()，执行回滚
} catch (Exception e) {
    tx.rollback();
}
```

### 2.6.7 保存点（Savepoint）支持

Hibernate 支持 JDBC 3.0 规范中的保存点特性，允许在事务内设置回滚断点：

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

session.persist(new User("张三"));

// 设置保存点（JDBC Savepoint）
tx.setSavepoint("sp1");

session.persist(new User("李四"));

// 回滚到保存点，李四不会被持久化
tx.rollbackToSavepoint("sp1");

session.persist(new User("王五"));
tx.commit();  // 最终持久化张三和王五，李四不持久化
session.close();
```

## 2.7 Query / TypedQuery

### 2.7.1 org.hibernate.query.Query 接口体系

```
Query<R>（Hibernate 原生接口，继承 JPA TypedQuery & Query）
├── SelectionQuery<R>     —— 查询操作（list, uniqueResult, stream）
├── MutationQuery         —— 变更操作（executeUpdate） Hibernate 6+
├── NativeQuery<R>        —— 原生 SQL 查询扩展
├── Query<R>（JPA 接口）
│   └── TypedQuery<X>（JPA 接口）
└── ScrollableResults<R>  —— 可滚动结果集
```

### 2.7.2 Query 创建方式

```java
// 1. HQL/JPQL 查询
Query<User> query = session.createQuery(
    "from User u where u.age > :age", User.class);
query.setParameter("age", 18);
List<User> users = query.list();

// 2. 原生 SQL 查询
NativeQuery<User> nativeQuery = session.createNativeQuery(
    "SELECT * FROM t_user WHERE age > :age", User.class);
nativeQuery.setParameter("age", 18);

// 3. 命名查询
Query<User> namedQuery = session.createNamedQuery("User.findByAge", User.class);
namedQuery.setParameter("age", 18);

// 4. 命名原生查询
NativeQuery<User> namedNativeQuery = session.createNamedNativeQuery(
    "User.findByAgeNative", User.class);
```

### 2.7.3 TypedQuery 泛型支持（JPA）

```java
TypedQuery<User> typedQuery = session.createQuery(
    "from User where age > :age", User.class);  // 编译期类型安全
List<User> users = typedQuery.getResultList();

// 对比无类型参数的 Query
Query legacyQuery = session.createQuery("from User where age > :age");
List rawList = legacyQuery.list();   // 无类型安全，需手动转型
```

### 2.7.4 Query 接口常用方法

```java
Query<User> query = session.createQuery("from User where age > :minAge", User.class);

// 参数绑定
query.setParameter("minAge", 18);                    // 命名参数
query.setParameter(0, 18);                           // 位置参数（不推荐）
query.setParameter("name", "%张%", StringType.INSTANCE);  // 指定类型

// 分页
query.setMaxResults(20);       // LIMIT 20
query.setFirstResult(0);       // OFFSET 0

// 锁模式
query.setLockMode(LockModeType.PESSIMISTIC_WRITE);   // SELECT ... FOR UPDATE

// 提示
query.setHint("org.hibernate.cacheable", true);       // 启用查询缓存
query.setHint("org.hibernate.readOnly", true);        // 只读

// 刷新模式
query.setFlushMode(FlushModeType.AUTO);

// 超时
query.setHint("javax.persistence.query.timeout", 5000); // 5秒
```

### 2.7.5 QueryProducer 接口

`Session` 实现了 `QueryProducer` 接口，提供了统一的查询创建入口：

```java
public interface QueryProducer {
    <R> Query<R> createQuery(String queryString, Class<R> resultClass);
    <R> NativeQuery<R> createNativeQuery(String sqlString, Class<R> resultClass);
    <R> Query<R> createNamedQuery(String name, Class<R> resultClass);
    <R> NativeQuery<R> createNamedNativeQuery(String name, Class<R> resultClass);
}
```

### 2.7.6 ScrollableResults（可滚动结果集）

```java
ScrollableResults<User> scroll = session.createQuery(
    "from User order by id", User.class)
    .scroll(ScrollMode.FORWARD_ONLY);  // 或 SCROLL_INSENSITIVE, SCROLL_SENSITIVE

// 正向遍历
while (scroll.next()) {
    User user = scroll.get();
}

// 大结果集场景下释放游标
scroll.close();
```

### 2.7.7 Query 的流式处理

```java
// Java 8 Stream 流式处理
try (Stream<User> stream = session.createQuery(
        "from User where age > :age", User.class)
        .setParameter("age", 18)
        .stream()) {
    stream.filter(u -> u.getName().startsWith("张"))
          .forEach(System.out::println);
}
```

### 2.7.8 MutationQuery 批量操作接口（Hibernate 6）

Hibernate 6 将更新/删除操作独立为 `MutationQuery` 接口：

```java
// Hibernate 6 批量更新
int updated = session.createMutationQuery(
    "update User set status = :status where age > :age")
    .setParameter("status", "INACTIVE")
    .setParameter("age", 60)
    .executeUpdate();

// Hibernate 6 批量删除
int deleted = session.createMutationQuery(
    "delete from User where status = :status")
    .setParameter("status", "INACTIVE")
    .executeUpdate();
```

## 2.8 CriteriaBuilder / CriteriaQuery（JPA Criteria API）

### 2.8.1 类型安全的查询构建

JPA Criteria API 提供了一种纯面向对象、编译期类型安全的查询构建方式，避免了字符串拼接 HQL 容易出错的问题。

```java
EntityManager em = sessionFactory.createEntityManager();
CriteriaBuilder cb = em.getCriteriaBuilder();

// 构建查询
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);

// 添加条件
Predicate ageGt18 = cb.greaterThan(root.get("age"), 18);
Predicate nameLike = cb.like(root.get("name"), "%张%");
cq.where(cb.and(ageGt18, nameLike));

// 排序
cq.orderBy(cb.desc(root.get("createTime")));

// 执行
TypedQuery<User> query = em.createQuery(cq);
List<User> result = query.getResultList();
```

### 2.8.2 与 Hibernate Criteria（已废弃）的区别

| 特性 | JPA Criteria API | Hibernate Criteria (deprecated) |
|------|-------------------|--------------------------------|
| 标准 | JPA 规范（可移植） | Hibernate 私有 API |
| 类型安全 | 配合 Static Metamodel 实现强类型 | 无编译期类型检查 |
| 查询模型 | `CriteriaQuery<T>` + `Root<T>` | `Criteria` + `Criterion` |
| 状态 | 当前推荐 | Hibernate 5 废弃，6 移除 |
| 关联获取 | `root.fetch("orders", JoinType.LEFT)` | `criteria.setFetchMode("orders", FetchMode.JOIN)` |

### 2.8.3 静态元模型（Static Metamodel）配合使用

通过 Hibernate JPA 元模型生成器（`hibernate-jpamodelgen`）生成静态元模型类，实现完全类型安全的查询：

```java
// 生成的元模型类（编译时生成）
@StaticMetamodel(User.class)
public abstract class User_ {
    public static volatile SingularAttribute<User, Long> id;
    public static volatile SingularAttribute<User, String> name;
    public static volatile SingularAttribute<User, Integer> age;
    public static volatile ListAttribute<User, Order> orders;
}

// 使用元模型进行类型安全查询
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);

// 编译期类型安全：User_.age 是 Integer 类型，只能比较 Integer
cq.where(cb.gt(root.get(User_.age), 18));   // ✅ 编译通过
// cq.where(cb.gt(root.get(User_.age), "18")); // ❌ 编译错误
```

# 3. 关键设计模式

## 3.1 工厂模式

### 3.1.1 SessionFactory 作为 Session 的工厂（Abstract Factory）

`SessionFactory` 是典型的抽象工厂模式实现——它为创建一系列相关的 `Session` 对象提供了统一接口，而不需要指定具体的实现类。客户端只需要使用 `SessionFactory` 接口，底层返回的是 `SessionImpl` 实现。

```java
// 客户端只依赖 SessionFactory 接口
SessionFactory factory = metadata.getSessionFactoryBuilder().build();

// 工厂方法创建不同类别的产品
Session session1 = factory.openSession();           // 普通 Session
Session session2 = factory.getCurrentSession();    // 绑定到上下文的 Session
Session session3 = factory.withOptions()
    .interceptor(new MyInterceptor())
    .connection(customConnection)
    .openSession();                                // 带选项的 Session
```

### 3.1.2 SessionFactoryBuilder 作为 SessionFactory 的 Builder

`SessionFactoryBuilder` 实现了 Builder 模式，逐步构建复杂的 `SessionFactory` 对象：

```java
SessionFactory factory = metadata.getSessionFactoryBuilder()
    .applyInterceptor(new AuditInterceptor())         // 设置拦截器
    .applyStatisticsSettings(new StatisticsSettings()) // 统计配置
    .applyBeanManager(cdiBeanManager)                 // CDI 集成
    .applyValidatorFactory(validatorFactory)          // Bean Validation 集成
    .applyName("my-persistence-unit")                 // 命名
    .build();                                          // 最终构建
```

Builder 模式的优势：
- 将复杂对象的构建过程与表示分离
- 支持链式调用，代码可读性强
- 提供灵活的配置选项，不用为每种组合创建构造函数重载

## 3.2 代理模式

### 3.2.1 懒加载代理对象（Javassist/ByteBuddy 生成）

Hibernate 通过字节码增强库（早期 Javassist，Hibernate 5.3+ 默认 ByteBuddy）在运行时生成实体类的代理子类，实现懒加载：

```java
// 原始实体
@Entity
public class User {
    @Id private Long id;
    private String name;
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private Set<Order> orders;
    // getters & setters
}

// Hibernate 运行时生成的代理类（伪代码）
public class User$HibernateProxy$xyz extends User implements HibernateProxy {
    private LazyInitializer initializer;
    private Set<Order> orders;  // 未被初始化时为 null

    @Override
    public Set<Order> getOrders() {
        if (this.initializer != null) {
            this.initializer.initialize();  // 触发 SQL 加载
        }
        return super.getOrders();
    }
}
```

### 3.2.2 代理拦截原理

```
应用代码: user.getOrders()
    │
    ▼
代理对象: User$HibernateProxy$xyz.getOrders()
    │
    ├── 检查是否已初始化
    │   ├── 已初始化 ──► 返回 orders 值
    │   └── 未初始化 ──► LazyInitializer.initialize()
    │                       │
    │                       ├── 获取 Session（可能需要重新打开）
    │                       ├── 生成 SQL: SELECT * FROM t_order WHERE user_id = ?
    │                       ├── 执行 SQL
    │                       ├── 填充 orders 字段
    │                       └── 标记为已初始化
    │
    └── 返回 orders 值
```

### 3.2.3 HibernateProxy 接口

```java
public interface HibernateProxy {
    LazyInitializer getHibernateLazyInitializer();
    Object writeReplace();  // 序列化支持
}

public interface LazyInitializer {
    void initialize();
    boolean isUninitialized();
    Object getImplementation();
    SessionImplementor getSession();
    // ...
}
```

### 3.2.4 代理与继承关系

Hibernate 代理要求实体类满足以下条件：

- **不能是 `final` 类**：代理需要继承实体类，`final` 阻止了继承
- **至少有一个无参构造器**（可以是 `protected` 或 `public`）：代理类需要调用 `super()`
- **懒加载属性对应的 getter 不能是 `final`**：代理需要重写 getter
- **业务方法不应直接访问字段，应使用 getter**：确保代理能拦截

```java
@Entity
public class Order {  // ✅ 非 final，可以被继承
    @Id
    private Long id;

    public Long getId() {  // ✅ 非 final，可被重写
        return id;
    }
}

// ❌ 错误示例
// public final class Order { }  // 代理无法继承 final 类
// public final Long getId() {}  // 代理无法重写 final 方法
```

## 3.3 拦截器/观察者模式

### 3.3.1 Interceptor 接口

`org.hibernate.Interceptor` 允许在 CRUD 操作前后插入自定义逻辑（如审计日志、权限检查、数据加密等）：

```java
public class AuditInterceptor implements Interceptor {
    @Override
    public boolean onSave(Object entity, Object id, Object[] state,
                          String[] propertyNames, Type[] types) {
        if (entity instanceof Auditable) {
            // 自动设置修改时间
            for (int i = 0; i < propertyNames.length; i++) {
                if ("lastModified".equals(propertyNames[i])) {
                    state[i] = LocalDateTime.now();
                    return true;  // 告知 Hibernate 状态已修改
                }
            }
        }
        return false;
    }

    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState,
                                Object[] previousState, String[] propertyNames,
                                Type[] types) {
        // 更新时设置修改时间
        if (entity instanceof Auditable) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("lastModified".equals(propertyNames[i])) {
                    currentState[i] = LocalDateTime.now();
                    return true;
                }
            }
        }
        return false;
    }

    @Override
    public void onDelete(Object entity, Object id, Object[] state,
                         String[] propertyNames, Type[] types) {
        // 删除时记录审计日志
        System.out.println("Entity " + entity.getClass().getSimpleName()
            + " with id " + id + " deleted");
    }

    @Override
    public boolean onLoad(Object entity, Object id, Object[] state,
                          String[] propertyNames, Type[] types) {
        // 加载时触发（如解密数据）
        return false;
    }
}
```

### 3.3.2 Event/Listener 系统架构

相比 `Interceptor`，Event/Listener 系统更细粒度、更灵活，是 Hibernate 内部使用的主要扩展机制：

```
       事件触发（如 persist(entity)）
                │
                ▼
      ┌─────────────────────┐
      │    EventManager      │  ← 中央调度器
      └─────────────────────┘
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
 Listener   Listener   Listener
 Group 1    Group 2    Group 3
```

### 3.3.3 事件类型枚举与对应监听器接口

| 事件类型 | 监听器接口 | 触发时机 |
|----------|------------|----------|
| `PreInsertEvent` | `PreInsertEventListener` | `persist()` 后、INSERT SQL 执行前 |
| `PostInsertEvent` | `PostInsertEventListener` | INSERT SQL 执行后、事务提交前 |
| `PreUpdateEvent` | `PreUpdateEventListener` | 更新实体后、UPDATE SQL 执行前 |
| `PostUpdateEvent` | `PostUpdateEventListener` | UPDATE SQL 执行后 |
| `PreDeleteEvent` | `PreDeleteEventListener` | `remove()` 后、DELETE SQL 执行前 |
| `PostDeleteEvent` | `PostDeleteEventListener` | DELETE SQL 执行后 |
| `PreLoadEvent` | `PreLoadEventListener` | 从数据库加载实体前 |
| `PostLoadEvent` | `PostLoadEventListener` | 从数据库加载实体后 |
| `FlushEvent` | `FlushEventListener` | `flush()` 开始时 |
| `PostCommitInsertEvent` | `PostInsertEventListener` | 事务提交后 |
| `PostCommitUpdateEvent` | `PostUpdateEventListener` | 事务提交后 |
| `PostCommitDeleteEvent` | `PostDeleteEventListener` | 事务提交后 |

### 3.3.4 事件注册与分发流程

```java
// 自定义事件监听器
public class MyPostInsertListener implements PostInsertEventListener {
    @Override
    public void onPostInsert(PostInsertEvent event) {
        Object entity = event.getEntity();
        // 后处理逻辑（发送消息、更新缓存等）
    }

    @Override
    public boolean requiresPostCommitHandling(EntityPersister persister) {
        return true;  // 需要事务提交后才执行
    }

    @Override
    public boolean requiresPostCommitHandlingAfterFlush(EntityPersister persister) {
        return true;
    }
}

// 手动注册监听器（通过 Integrator）
public class MyIntegrator implements Integrator {
    @Override
    public void integrate(Metadata metadata, SessionFactoryImplementor sessionFactory,
                          SessionFactoryServiceRegistry serviceRegistry) {
        EventListenerRegistry listenerRegistry =
            serviceRegistry.getService(EventListenerRegistry.class);
        listenerRegistry.appendListeners(
            EventType.POST_INSERT, new MyPostInsertListener());
    }
}
```

## 3.4 策略模式

策略模式在 Hibernate 中广泛应用，允许在运行时替换具体行为，实现可插拔架构。

### 3.4.1 Dialect（数据库方言）

不同数据库的 SQL 语法、分页方式、序列获取等各不相同，`Dialect` 将这些差异封装为可替换的策略：

```java
// 策略的使用
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .applySetting("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect")
    .build();
// 或自动检测
// Hibernate 根据 JDBC 元数据自动选择匹配的 Dialect
```

### 3.4.2 IdentifierGenerator（主键生成策略）

不同实体可以使用不同的主键生成策略：

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // IDENTITY 策略
    private Long id;
}

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "order_seq")  // SEQUENCE 策略
    @SequenceGenerator(name = "order_seq", sequenceName = "order_sequence")
    private Long id;
}

@Entity
public class AuditLog {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)  // UUID 策略
    private String id;
}
```

### 3.4.3 CacheProvider（缓存提供者）

```java
// EHCache 策略
hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory

// Redis 策略（通过第三方适配器）
hibernate.cache.region.factory_class=org.hibernate.cache.redis.RedisRegionFactory

// Infinispan 策略
hibernate.cache.region.factory_class=org.hibernate.cache.infinispan.InfinispanRegionFactory

// Hazelcast 策略
hibernate.cache.region.factory_class=org.hibernate.cache.hazelcast.HazelcastCacheRegionFactory
```

### 3.4.4 ConnectionProvider（连接提供者）

```java
// HikariCP 连接池策略
hibernate.connection.provider_class=org.hibernate.hikaricp.internal.HikariCPConnectionProvider

// C3P0 连接池策略
hibernate.connection.provider_class=org.hibernate.c3p0.internal.C3P0ConnectionProvider

// JNDI 数据源策略
hibernate.connection.datasource=java:comp/env/jdbc/MyDS
```

### 3.4.5 BatchBuilder（批处理策略）

```java
// 启用 JDBC 批处理
hibernate.jdbc.batch_size=30
hibernate.jdbc.batch_versioned_data=true
hibernate.order_inserts=true
hibernate.order_updates=true
```

## 3.5 装饰器模式

### 3.5.1 Session 的装饰器链

Hibernate 允许通过装饰器模式在 Session 上动态附加行为（如拦截器、统计收集等）：

```
调用链: Client → SessionWrapper → SessionWrapper → SessionImpl → JDBC
```

### 3.5.2 SessionWrapper / SessionDelegatorBaseImpl

`org.hibernate.Session` 被设计为可以包装的接口，`SessionDelegatorBaseImpl` 提供了默认的委托实现：

```java
public class CustomSession extends SessionDelegatorBaseImpl {
    public CustomSession(SessionImplementor delegate) {
        super(delegate);
    }

    @Override
    public void persist(Object entity) {
        // 装饰前处理：记录日志
        System.out.println("Before persist: " + entity);
        // 委托给真正的 Session
        super.persist(entity);
        // 装饰后处理：刷新缓存
        System.out.println("After persist: " + entity);
    }
}

// 使用装饰器
Session rawSession = sessionFactory.openSession();
Session decoratedSession = new CustomSession((SessionImplementor) rawSession);
```

### 3.5.3 拦截器如何包装 Session

当通过 `SessionFactory.withOptions().interceptor(interceptor).openSession()` 创建 Session 时，Hibernate 内部将拦截器织入 Session 的事件处理链：

```java
// 内部逻辑示意（简化）
Session session = sessionFactory.withOptions()
    .interceptor(new AuditInterceptor())
    .openSession();

// 实际效果等价于：
// SessionImpl 内部持有 Interceptor 引用
// 每次 CRUD 操作时调用 interceptor.onXxx() 方法
```

## 3.6 享元模式

### 3.6.1 查询计划缓存（QueryPlanCache）

HQL/JPQL 查询的解析和 SQL 生成是计算密集型操作。通过享元模式，Hibernate 将查询计划（QueryPlan）缓存并共享：

```
SessionFactory
└── QueryPlanCache（全局单例，跨 Session 共享）
    ├── "from User"          → QueryPlan_A（HQL AST + SQL 模板）
    ├── "from Order"         → QueryPlan_B
    ├── "from User u join..."→ QueryPlan_C
    └── ...
```

```java
// 内部逻辑（简化）
public class QueryPlanCache {
    private final Map<String, HQLQueryPlan> hqlPlanCache = new ConcurrentHashMap<>();

    public HQLQueryPlan getHQLQueryPlan(String queryString, Map<String, Filter> filters) {
        String cacheKey = buildCacheKey(queryString, filters);
        return hqlPlanCache.computeIfAbsent(cacheKey, key -> {
            // 解析 HQL → AST → SQL，开销大，只做一次
            HqlParser parser = new HqlParser(queryString);
            // ... 解析和生成 SQL
            return new HQLQueryPlan(sql, parameterMetadata);
        });
    }
}
```

### 3.6.2 类型元数据共享

实体类型的映射元数据（`EntityPersister`、`Type`）在 `SessionFactory` 级别共享，所有 `Session` 共享同一份，避免重复解析：

```
SessionFactory（共享）
├── EntityPersister[User]  ── 所有 Session 共享这一份
├── EntityPersister[Order] ── 所有 Session 共享这一份
├── Type[String]           ── 所有 Session 共享这一份
└── Type[Integer]          ── 所有 Session 共享这一份

Session_A ──── 引用 ──► EntityPersister[User]
Session_B ──── 引用 ──► EntityPersister[User]  （同一对象引用）
```

# 4. 工作流程详解

## 4.1 启动流程

Hibernate 的启动过程是一个多阶段的初始化流水线，每一阶段完成特定的初始化任务：

### 4.1.1 完整启动时序

```
1. 引导阶段
   │
   ├── BootstrapServiceRegistryBuilder.build()
   │   ├── 确定 ClassLoader
   │   ├── 加载 Integrator（ServiceLoader 发现）
   │   └── 创建 StrategySelector
   │
2. 服务注册阶段
   │
   ├── StandardServiceRegistryBuilder.configure()
   │   ├── 解析 hibernate.cfg.xml
   │   ├── 合并 hibernate.properties
   │   └── 合并编程方式设置的属性
   │
   ├── StandardServiceRegistryBuilder.build()
   │   ├── 创建 ConnectionProvider
   │   ├── 创建 TransactionFactory
   │   ├── 创建 JdbcServices（连接数据库获取元数据）
   │   ├── 解析 Dialect（手动指定 或 自动检测）
   │   └── 初始化所有内置服务
   │
3. 元数据收集阶段
   │
   ├── new MetadataSources(registry)
   │   ├── addAnnotatedClass(User.class)
   │   ├── addAnnotatedClass(Order.class)
   │   └── addPackage("com.example.entity")
   │
   ├── metadataSources.getMetadataBuilder()
   │   ├── 扫描实体类注解（@Entity, @Table, @Column...）
   │   ├── 解析 orm.xml / hbm.xml（如有）
   │   ├── 构建 DomainModel（内部领域模型）
   │   └── InFlightMetadataCollector 收集
   │
4. 命名策略应用
   │
   ├── applyImplicitNamingStrategy(...)
   │   └── 生成未显式指定的表名/列名/外键名
   │
   ├── applyPhysicalNamingStrategy(...)
   │   └── 转换物理名称（加前缀、大小写转换等）
   │
5. 映射验证阶段
   │
   ├── MetadataBuilder.build()
   │   ├── 检查每个实体是否有 @Id 或 @EmbeddedId
   │   ├── 检查关联映射一致性（mappedBy 匹配）
   │   ├── 检查命名冲突（表名、列名重复）
   │   ├── 检查继承映射配置合法性
   │   ├── 检查 @SecondaryTable 引用完整性
   │   └── 生成最终 Metadata（不可变对象）
   │
6. SessionFactory 构建阶段
   │
   ├── metadata.getSessionFactoryBuilder().build()
   │   ├── 为每个实体创建 EntityPersister
   │   ├── 为每个集合创建 CollectionPersister
   │   ├── 初始化 QueryPlanCache（查询计划缓存）
   │   ├── 初始化 L2 缓存区域（EHCache/Redis 等）
   │   ├── 创建 SessionFactoryServiceRegistry
   │   ├── 执行 Integrator.integrate()（Envers 集成等）
   │   └── 返回 SessionFactoryImpl（单例持有）
```

### 4.1.2 命名策略示例

```java
// 隐式命名策略：当未显式指定表名、列名时，如何生成默认名称
ImplicitNamingStrategyJpaCompliantImpl.INSTANCE;
// User 类 → 表名 "User"

// 物理命名策略：所有表名、列名应用统一的物理转换
PhysicalNamingStrategyStandardImpl.INSTANCE;
// 不做转换，保持原样

// 自定义：加前缀 + 下划线分隔
public class CustomPhysicalNamingStrategy extends PhysicalNamingStrategyStandardImpl {
    @Override
    public Identifier toPhysicalTableName(Identifier name, JdbcEnvironment context) {
        return Identifier.toIdentifier("t_" + name.getText().toLowerCase());
    }
    @Override
    public Identifier toPhysicalColumnName(Identifier name, JdbcEnvironment context) {
        return Identifier.toIdentifier(name.getText().toLowerCase());
    }
}
```

### 4.1.3 映射验证示例

启动时 Hibernate 会执行以下验证（如果 `hibernate.hbm2ddl.auto` 不为 `none`，可能创建或验证表结构）：

```java
// 验证失败示例
// @Entity class User { // 缺少 @Id }  → 抛出 AnnotationException

// @Entity class Order {
//     @ManyToOne
//     private User user;  // mappedBy="user" 在 User 侧，但 User 侧没有 @OneToMany(mappedBy="user")
// }
// → 抛出 MappingException: "broken column mapping"
```

## 4.2 请求处理流程

以 `session.persist(entity)` 为例，描述完整的请求处理流程：

### 4.2.1 完整流程图

```
Client Code: session.persist(user)
    │
    ▼
SessionImpl.persist(...)
    │
    ├── 1. 检查 Session 状态（已关闭则抛异常）
    ├── 2. 检查实体是否已关联到当前 Session
    │      ├── 已关联（managed）→ 无操作
    │      └── 未关联 → 继续
    │
    ├── 3. 触发 Interceptor.onSave(entity, id, state, propertyNames, types)
    │      └── 拦截器可修改 state 数组
    │
    ├── 4. 触发 PersistEvent → PersistEventListener#onPersist()
    │      ├── 调用 EntityPersister#getIdentifier() 获取/生成主键
    │      ├── 将实体注册到 PersistenceContext（一级缓存）
    │      └── 创建 EntityInsertAction → 加入 ActionQueue
    │
    ├── 5. 【此时 SQL 尚未执行】实体进入 ActionQueue 排队
    │
6. Flush 时机（以下任一触发）
    │
    ├── 显式调用 session.flush()
    ├── 事务提交 tx.commit()
    ├── 查询前（FlushMode.AUTO 且查询表与未持久化实体关联）
    │
    ▼
7. AbstractFlushingEventListener#onFlush()
    │
    ├── 脏检查（Dirty Check）
    │   ├── 比较实体当前状态与加载时的快照（snapshot）
    │   ├── 标记已修改的实体
    │   └── 生成 UpdateAction
    │
    ├── ActionQueue 排序
    │   ├── InsertAction 最前
    │   ├── UpdateAction 中间
    │   ├── CollectionAction（重建/删除/更新）
    │   └── DeleteAction 最后
    │
    ├── SQL 生成
    │   ├── EntityPersister#generateInsertString()  → "INSERT INTO ..."
    │   ├── EntityPersister#generateUpdateString()  → "UPDATE ... SET ..."
    │   └── EntityPersister#generateDeleteString()  → "DELETE FROM ..."
    │
    ├── SQL 执行
    │   ├── JdbcCoordinator → ConnectionProvider → JDBC Connection
    │   ├── Binder 绑定参数 → PreparedStatement.executeUpdate()
    │   └── 提取生成的主键值 → 回填到实体对象
    │
    └── 后处理
        ├── 更新 PersistenceContext 中的快照
        └── 触发 PostFlushEvent
    │
    ▼
8. Transaction.commit()（由客户端调用）
    │
    ├── beforeCompletion() - 回调
    ├── Connection.commit()
    ├── 触发 PostCommitEvents
    └── afterCompletion() - 回调
    │
    ▼
9. Session.close()
    │
    ├── 清理 PersistenceContext（一级缓存清空，实体变 detached）
    ├── 归还 JDBC 连接给连接池
    ├── 清理 ActionQueue
    └── 标记 Session 为已关闭
```

### 4.2.2 关键环节代码示例

```java
// 1. 获取 Session
Session session = sessionFactory.getCurrentSession();
// 或
Session session = sessionFactory.openSession();

// 2. 开启事务
Transaction tx = session.beginTransaction();

// 3. 执行操作
User user = new User("张三", 25);
session.persist(user);  // 此时尚未执行 INSERT

User existingUser = session.get(User.class, 1L); // 发送 SELECT
existingUser.setName("李四");  // 标记为脏（dirty）

Order order = new Order();
order.setUser(existingUser);
session.persist(order);  // 尚未执行 INSERT

// 4. 事务提交时触发 flush → 脏检查 → SQL 执行
tx.commit();  // 此时才真正执行 INSERT INTO t_user, UPDATE t_user, INSERT INTO t_order

// 5. 关闭 Session
session.close();
```

## 4.3 Flush 机制

### 4.3.1 FlushMode 枚举值

| 模式 | 说明 |
|------|------|
| `AUTO` | 默认模式。查询前自动 flush（仅当查询表与待持久化的实体表有关联时）|
| `COMMIT` | 仅在事务提交时 flush，查询前不自动 flush |
| `ALWAYS` | 每次查询前都强制 flush |
| `MANUAL` | 完全手动控制，只在显式调用 `flush()` 时执行 |

### 4.3.2 各模式下 Flush 触发时机

```java
// FlushMode.AUTO（默认）
Session session = sessionFactory.openSession();
session.setFlushMode(FlushModeType.AUTO);  // 或 FlushMode.AUTO
session.persist(new User());
// 以下查询可能触发 flush（取决于是否与未持久化的实体有关）
session.createQuery("from User", User.class).list();  // ✅ 触发 flush（User 表有未持久化实体）
session.createQuery("from Order", Order.class).list(); // ❌ 可能不触发（Order 表无关）

// FlushMode.COMMIT
session.setFlushMode(FlushModeType.COMMIT);
session.persist(new User());
session.createQuery("from User", User.class).list();  // ❌ 不触发 flush，查不到新实体
tx.commit();  // ✅ 此时才 flush

// FlushMode.MANUAL（JPA 批处理场景）
session.setFlushMode(FlushModeType.MANUAL);
for (int i = 0; i < 100; i++) {
    session.persist(new User("user" + i));
    if (i % 50 == 0) {
        session.flush();   // 手动控制
        session.clear();   // 清理一级缓存，避免内存溢出
    }
}
```

### 4.3.3 Flush 的内部步骤

**步骤 1：收集脏实体**

```java
// 伪代码
List<EntityEntry> dirtyEntities = new ArrayList<>();
for (EntityEntry entry : persistenceContext.reentrantSafeEntityEntries()) {
    if (entry.requiresDirtyCheck(flushEntityEvent)) {
        Object[] currentState = persister.getPropertyValues(entity);
        Object[] loadedState = entry.getLoadedState();
        if (!Arrays.equals(currentState, loadedState)) {
            dirtyEntities.add(entry);  // 实体发生了变更
        }
    }
}
```

**步骤 2：确定 SQL 执行顺序**

```
1. EntityInsertAction         ← 最先执行（插入）
2. EntityUpdateAction         ← 更新已有记录
3. CollectionRecreateAction   ← 集合重建
4. CollectionRemoveAction     ← 集合元素删除
5. CollectionUpdateAction     ← 集合元素变更
6. EntityDeleteAction         ← 最后执行（删除）
```

这个顺序保证了外键依赖的正确性：
- 先 INSERT 父实体，再 INSERT 子实体（子实体引用父实体的 ID）
- 先 DELETE 子实体，再 DELETE 父实体（解除外键引用）
- 集合操作在 DELETE 之前，避免了约束冲突

**步骤 3：按外键依赖排序**

Hibernate 内部使用 `ActionSorter` 对 `ActionQueue` 中的操作按外键依赖关系进行拓扑排序：

```java
// 伪代码示意
public interface ActionSorter {
    void sort(List<EntityInsertAction> insertions);
    void sort(List<EntityUpdateAction> updates);
    void sort(List<EntityDeleteAction> deletions);
}
```

**步骤 4：预执行（Pre-insert）生成 ID**

在执行 INSERT SQL 之前，需要先生成主键值：

```java
// IDENTITY 自增策略 → 不提前生成，INSERT 后从数据库获取
// SEQUENCE 策略 → 提前调用 NEXTVAL 获取序列值
// UUID 策略 → 在应用侧生成 UUID
// ASSIGNED 策略 → 由应用代码赋值，Hibernate 不做处理
```

**步骤 5：执行 SQL**

通过 JDBC 批处理（如果启用）执行 SQL：

```java
for (EntityInsertAction action : sortedInsertions) {
    PreparedStatement ps = jdbcCoordinator.getStatementExecutor().prepare(sql);
    binder.bindValues(ps, action.getState());  // 参数绑定
    ps.addBatch();  // 批处理模式
}
jdbcCoordinator.executeBatch();  // 一次性提交所有 INSERT
```

**步骤 6：后处理（Post-commit 事件、清理）**

- 更新 `PersistenceContext` 中的快照（更新后的属性值变为新的 loaded state）
- 触发 `PostFlushEvent` 和各 `PostCommitEvent`
- 清理不再需要的 Action

### 4.3.4 ActionQueue 的结构与排序算法

```
ActionQueue
├── InsertAction[]           —— ArrayList + 排序
├── UpdateAction[]           —— ArrayList + 排序
├── DeleteAction[]           —— ArrayList + 排序
├── CollectionRecreateAction[]
├── CollectionRemoveAction[]
└── CollectionUpdateAction[]
```

排序算法：当提交 `hibernate.order_inserts=true` 和 `hibernate.order_updates=true` 时，Hibernate 会按外键依赖图进行拓扑排序，确保 INSERT/UPDATE 语句的顺序不违反外键约束。

# 5. JDBC 抽象层

## 5.1 JdbcServices

### 5.1.1 JdbcServices 接口职责

`JdbcServices` 是 Hibernate 与 JDBC 之间的核心抽象层，封装了所有与 JDBC 驱动直接交互的操作：

```java
public interface JdbcServices extends Service {
    Dialect getDialect();                    // 方言
    SqlStatementLogger getSqlStatementLogger();  // SQL 日志
    SqlExceptionHelper getSqlExceptionHelper();  // 异常转换
    ExtractedDatabaseMetaData getExtractedMetaData();  // 数据库元数据
}
```

### 5.1.2 JdbcEnvironment（方言、SQL 类型映射）

```java
public interface JdbcEnvironment {
    Dialect getDialect();
    SqlTypeDescriptorRegistry getSqlTypeDescriptorRegistry();
    IdentifierHelper getIdentifierHelper();  // 标识符处理（大小写、引号）
    NameQualifierSupport getNameQualifierSupport();  // catalog/schema 支持
}
```

### 5.1.3 SqlStatementLogger

`SqlStatementLogger` 负责 SQL 日志的输出（由 `hibernate.show_sql` 控制）：

```java
// 配置
hibernate.show_sql=true
hibernate.format_sql=true
hibernate.highlight_sql=true
hibernate.use_sql_comments=true

// 效果
/* insert com.example.entity.User
   */ insert
    into
        t_user
        (name, age, create_time, id)
    values
        (?, ?, ?, ?)
```

### 5.1.4 JdbcCoordinator（逻辑连接管理）

`JdbcCoordinator` 管理逻辑连接（`LogicalConnection`），封装物理连接的获取与释放策略：

```java
public interface JdbcCoordinator {
    LogicalConnection getLogicalConnection();
    StatementPreparer getStatementPreparer();
    ResultSetReturn getResultSetReturn();
    BatchKey getBatchKey();
    void executeBatch();
    void close();  // 释放连接
}
```

## 5.2 ConnectionProvider

### 5.2.1 连接获取策略

| 策略 | 实现类 | 说明 |
|------|--------|------|
| DriverManager | `DriverManagerConnectionProviderImpl`（已废弃）| 直接使用 JDBC 驱动，每次获取新连接 |
| DataSource | `DatasourceConnectionProviderImpl` | 通过标准 `javax.sql.DataSource` 获取连接 |
| JNDI | `JndiDatasourceConnectionProvider` | 从应用服务器 JNDI 查找 DataSource |
| C3P0 | `C3P0ConnectionProvider` | 集成 C3P0 连接池 |
| HikariCP | `HikariCPConnectionProvider` | 集成 HikariCP 连接池（推荐） |

### 5.2.2 DatasourceConnectionProvider（推荐）

```java
// hibernate.cfg.xml
// <property name="hibernate.connection.datasource">java:comp/env/jdbc/MyDS</property>

// 编程方式配置
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .applySetting("hibernate.connection.datasource", myDataSource)
    .build();
```

### 5.2.3 自定义 ConnectionProvider 实现

```java
public class MyConnectionProvider implements ConnectionProvider, Configurable {
    private DataSource dataSource;

    @Override
    public void configure(Map<String, Object> configValues) {
        // 从配置中获取连接池参数，初始化 DataSource
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl((String) configValues.get("hibernate.connection.url"));
        config.setUsername((String) configValues.get("hibernate.connection.username"));
        config.setPassword((String) configValues.get("hibernate.connection.password"));
        this.dataSource = new HikariDataSource(config);
    }

    @Override
    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }

    @Override
    public void closeConnection(Connection conn) throws SQLException {
        conn.close();  // 归还给连接池
    }

    @Override
    public boolean supportsAggressiveRelease() {
        return true;  // 支持提前释放连接
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return DataSource.class.isAssignableFrom(unwrapType);
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        return (T) dataSource;
    }
}
```

### 5.2.4 连接释放与连接池配合

Hibernate 的连接释放模式通过 `hibernate.connection.release_mode` 配置：

| 模式 | 行为 |
|------|------|
| `ON_CLOSE` | Session 关闭时释放连接（Hibernate 3.x 默认） |
| `AFTER_TRANSACTION` | 事务提交后立即释放连接（Hibernate 4+ 默认） |
| `AFTER_STATEMENT` | 每次 Statement 执行后释放（已废弃，多数连接池不支持） |

```xml
<property name="hibernate.connection.release_mode">after_transaction</property>
```

## 5.3 Dialect（方言）

### 5.3.1 Dialect 核心职责

| 职责 | 说明 |
|------|------|
| 分页 SQL 生成 | `getLimitString(String sql, int offset, int limit)` |
| 序列获取 | `getSequenceNextValString(String sequenceName)` |
| 自增主键 | `getIdentityInsertString()` |
| DDL 生成 | `getTableTypeString()`, `getAddColumnString()` |
| 函数注册 | `registerFunction(String name, SQLFunction function)` |
| 锁策略 | `getForUpdateString()`, `getReadLockString()` |

### 5.3.2 分页 SQL 方言差异

```sql
-- MySQL / PostgreSQL
SELECT * FROM t_user LIMIT 20 OFFSET 0;

-- Oracle 12c+
SELECT * FROM t_user OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;

-- Oracle 11g
SELECT * FROM (
    SELECT t.*, ROWNUM rn FROM (SELECT * FROM t_user ORDER BY id) t
    WHERE ROWNUM <= 20
) WHERE rn > 0;

-- SQL Server 2012+
SELECT * FROM t_user ORDER BY id OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;

-- SQL Server 2008
SELECT TOP 20 * FROM t_user WHERE id NOT IN (
    SELECT TOP 0 id FROM t_user ORDER BY id
) ORDER BY id;

-- DB2
SELECT * FROM t_user ORDER BY id OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
```

### 5.3.3 DialectResolver 自动检测机制

Hibernate 通过 `DialectResolver` 自动检测数据库类型，无需手工指定：

```java
public interface DialectResolver {
    Dialect resolveDialect(DialectResolutionInfo info);
}

// 标准实现：通过 JDBC DatabaseMetaData 检测
public class StandardDialectResolver implements DialectResolver {
    // getDatabaseName() 获取 "MySQL", "PostgreSQL" 等
    // getDatabaseMajorVersion() 获取主版本号
    // 匹配到具体的版本化 Dialect
}
```

### 5.3.4 常用方言列表

| 数据库 | Hibernate 5 方言 | Hibernate 6 方言 |
|--------|------------------|------------------|
| MySQL 5.7 | `MySQL57Dialect` | `MySQLDialect(5)` |
| MySQL 8 | `MySQL8Dialect` | `MySQLDialect(8)` |
| PostgreSQL 10+ | `PostgreSQL10Dialect` | `PostgreSQLDialect(10)` |
| Oracle 12c+ | `Oracle12cDialect` | `OracleDialect(12)` |
| SQL Server 2016 | `SQLServer2016Dialect` | `SQLServerDialect(14)` |
| H2 | `H2Dialect` | `H2Dialect` |
| DB2 | `DB2Dialect` | `DB2Dialect` |
| MariaDB 10.3+ | `MariaDB103Dialect` | `MariaDBDialect(10)` |

### 5.3.5 自定义 Dialect

```java
public class CustomMySQL8Dialect extends MySQL8Dialect {

    public CustomMySQL8Dialect() {
        super();
        // 注册自定义 SQL 函数
        registerFunction("group_concat",
            new StandardSQLFunction("group_concat", StandardBasicTypes.STRING));
        registerFunction("date_format",
            new StandardSQLFunction("date_format", StandardBasicTypes.STRING));

        // 注册自定义类型映射
        registerColumnType(Types.BLOB, "LONGBLOB");
        registerColumnType(Types.CLOB, "LONGTEXT");
    }

    @Override
    public String getTableTypeString() {
        // 强制使用 InnoDB 引擎
        return " ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci";
    }

    @Override
    public String getLimitString(String sql, int offset, int limit) {
        // 自定义分页逻辑（如果需要）
        return super.getLimitString(sql, offset, limit);
    }
}
```

### 5.3.6 Hibernate 5 vs 6 Dialect 变化

- **Hibernate 5**：每个数据库大版本有独立的类，如 `MySQL5Dialect`, `MySQL55Dialect`, `MySQL57Dialect`, `MySQL8Dialect`
- **Hibernate 6**：统一为一个类，通过构造函数传入版本号，如 `new MySQLDialect(8)`, `new MySQLDialect(5)`，减少了类爆炸

### 5.3.7 多数据库适配时的 Dialect 自动选择原理

```
Hibernate 启动
    │
    ├── 获取 JDBC Connection
    ├── 获取 DatabaseMetaData
    │   ├── databaseProductName = "MySQL"
    │   └── databaseMajorVersion = 8
    │
    ├── DialectResolver.resolveDialect(info)
    │   ├── 遍历已注册的 DialectResolver
    │   ├── StandardDialectResolver 匹配到 MySQL
    │   └── 返回 MySQLDialect(8)
    │
    └── 使用该 Dialect 生成所有 SQL
```

# 6. 类型系统

## 6.1 Type 接口体系

### 6.1.1 org.hibernate.type.Type 顶层接口

```java
public interface Type extends Serializable {
    SqlTypeDescriptor getSqlTypeDescriptor();      // Hibernate 6
    JavaTypeDescriptor getJavaTypeDescriptor();    // Hibernate 6
    String getName();
    Class<?> getReturnedClass();
    boolean isMutable();
    boolean isAssociationType();
    boolean isCollectionType();
    boolean isEntityType();
    // ... SQL 参数绑定和结果提取方法
}
```

### 6.1.2 Type 实现体系

```
Type（顶层接口）
├── BasicType           —— 基本类型映射（String ↔ VARCHAR, int ↔ INTEGER）
│   ├── StringType      —— java.lang.String ↔ VARCHAR
│   ├── IntegerType     —— java.lang.Integer ↔ INTEGER
│   ├── LongType        —— java.lang.Long ↔ BIGINT
│   ├── BooleanType     —— java.lang.Boolean ↔ BOOLEAN
│   ├── DateType        —— java.util.Date ↔ TIMESTAMP
│   ├── BigDecimalType  —— java.math.BigDecimal ↔ NUMERIC
│   ├── BinaryType      —— byte[] ↔ BLOB
│   └── EnumType        —— Java Enum ↔ VARCHAR / ORDINAL
├── CompositeType（组件类型 / @Embeddable）
├── EntityType（实体类型 / @ManyToOne / @OneToOne）
├── CollectionType（集合类型 / @OneToMany / @ManyToMany）
└── AnyType（多态类型 / @Any / @ManyToAny）
```

### 6.1.3 JavaTypeDescriptor + SqlTypeDescriptor 分离（Hibernate 6 重构）

Hibernate 6 将 Type 拆分为两个独立维度，实现了更清晰的关注点分离：

```java
// Java 侧：JavaTypeDescriptor - 描述 Java 类型的特征
public interface JavaTypeDescriptor<T> {
    Class<T> getJavaTypeClass();                     // Java 类型
    MutabilityPlan<T> getMutabilityPlan();           // 可变性
    <X> T extract(ResultSet rs, String name);        // 从 ResultSet 提取值
    <X> void bind(PreparedStatement st, T value);    // 绑定到 PreparedStatement
}

// SQL 侧：SqlTypeDescriptor - 描述 SQL 类型的特征
public interface SqlTypeDescriptor {
    int getSqlType();                                // JDBC Types 常量
    boolean canBeRemapped();                         // 是否可重映射
    <X> ValueBinder<X> getBinder(JavaTypeDescriptor<X> javaType);  // 获取绑定器
    <X> ValueExtractor<X> getExtractor(JavaTypeDescriptor<X> javaType); // 获取提取器
}
```

这种分离的好处：
- Java 类型和 SQL 类型可以自由组合（如 `String` 可以映射为 `VARCHAR`, `TEXT`, `CLOB` 等）
- 自定义类型时不需要同时理解 Java 和 SQL 两侧的细节
- 更灵活的用户类型注册机制

## 6.2 类型解析流程

### 6.2.1 注解扫描 → TypeResolver → Dialect → SQL

```
实体属性 int age;
    │
    ▼
注解扫描: @Column, 无 @Type → 默认规则
    │
    ▼
TypeResolver: int → IntegerType (BasicType)
    │
    ▼
Dialect.getTypeName(Types.INTEGER): "INTEGER" (MySQL) / "NUMBER(10,0)" (Oracle)
    │
    ▼
DDL 生成: "age INTEGER"
SQL 绑定: PreparedStatement.setInt(?, age)
SQL 读取: ResultSet.getInt("age")
```

### 6.2.2 @Type 显式指定

```java
// 使用已注册的类型名
@Type(type = "yes_no")          // Boolean → 'Y'/'N'
@Type(type = "true_false")      // Boolean → 'T'/'F'
@Type(type = "numeric_boolean") // Boolean → 1/0
private Boolean active;

// 使用自定义类型
@Type(type = "com.example.type.JsonType")
private Map<String, Object> attributes;

// Hibernate 6 新语法
@Type(JsonType.class)
private Map<String, Object> attributes;
```

### 6.2.3 @Basic 对类型的影响

```java
@Basic(fetch = FetchType.LAZY)   // 属性懒加载（按需从数据库获取该列）
@Lob                              // 大对象类型（CLOB/BLOB）
private String content;
// 类型解析：String → MaterializedClobType

@Basic(optional = false)          // 非空（DDL 生成 NOT NULL）
private String email;

@Basic(fetch = FetchType.EAGER)   // 立即加载（默认）
private String name;
```

## 6.3 自定义类型

### 6.3.1 UserType 接口（完全自定义）

实现 `UserType` 可以将任意 Java 类型映射到一个或多个 JDBC 列：

```java
public class JsonType implements UserType<Map<String, Object>> {

    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public int getSqlType() {
        return Types.VARCHAR;  // 或 Types.LONGVARCHAR
    }

    @Override
    public Class<Map<String, Object>> returnedClass() {
        return (Class) Map.class;
    }

    @Override
    public boolean equals(Map<String, Object> x, Map<String, Object> y) {
        return Objects.equals(x, y);
    }

    @Override
    public int hashCode(Map<String, Object> x) {
        return x != null ? x.hashCode() : 0;
    }

    @Override
    public Map<String, Object> nullSafeGet(ResultSet rs, int position,
            SharedSessionContractImplementor session, Object owner)
            throws SQLException {
        String json = rs.getString(position);
        if (json == null) return null;
        try {
            return mapper.readValue(json, Map.class);
        } catch (JsonProcessingException e) {
            throw new HibernateException("Failed to parse JSON", e);
        }
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Map<String, Object> value,
            int index, SharedSessionContractImplementor session)
            throws SQLException {
        if (value == null) {
            st.setNull(index, Types.VARCHAR);
        } else {
            try {
                st.setString(index, mapper.writeValueAsString(value));
            } catch (JsonProcessingException e) {
                throw new HibernateException("Failed to serialize JSON", e);
            }
        }
    }

    @Override
    public Map<String, Object> deepCopy(Map<String, Object> value) {
        if (value == null) return null;
        try {
            return mapper.readValue(mapper.writeValueAsString(value), Map.class);
        } catch (JsonProcessingException e) {
            throw new HibernateException("Failed to copy JSON", e);
        }
    }

    @Override
    public boolean isMutable() {
        return true;  // Map 是可变的
    }

    @Override
    public Serializable disassemble(Map<String, Object> value) {
        return (Serializable) deepCopy(value);
    }

    @Override
    public Map<String, Object> assemble(Serializable cached, Object owner) {
        return deepCopy((Map<String, Object>) cached);
    }

    @Override
    public Map<String, Object> replace(Map<String, Object> original,
            Map<String, Object> target, Object owner) {
        return deepCopy(original);
    }
}
```

### 6.3.2 CompositeUserType（复合自定义类型）

当一个 Java 类型需要映射到**多个数据库列**时，使用 `CompositeUserType`：

```java
// 值对象：金额（包含数值 + 币种）
@Embeddable
public class Money {
    private BigDecimal amount;
    private String currency;
    // ...
}

// 映射到两列：amount NUMERIC + currency VARCHAR
public class MoneyType implements CompositeUserType<Money> {

    @Override
    public Class<Money> returnedClass() {
        return Money.class;
    }

    @Override
    public String[] getPropertyNames() {
        return new String[]{"amount", "currency"};
    }

    @Override
    public Type[] getPropertyTypes() {
        return new Type[]{StandardBasicTypes.BIG_DECIMAL, StandardBasicTypes.STRING};
    }

    @Override
    public Object getPropertyValue(Money component, int propertyIndex)
            throws HibernateException {
        if (component == null) return null;
        switch (propertyIndex) {
            case 0: return component.getAmount();
            case 1: return component.getCurrency();
            default: throw new IllegalArgumentException("Invalid index: " + propertyIndex);
        }
    }

    @Override
    public void setPropertyValue(Money component, int propertyIndex, Object value) {
        switch (propertyIndex) {
            case 0: component.setAmount((BigDecimal) value); break;
            case 1: component.setCurrency((String) value); break;
        }
    }

    @Override
    public Money nullSafeGet(ResultSet rs, String[] names,
            SharedSessionContractImplementor session, Object owner)
            throws SQLException {
        BigDecimal amount = rs.getBigDecimal(names[0]);
        String currency = rs.getString(names[1]);
        if (amount == null && currency == null) return null;
        return new Money(amount, currency);
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Money value, int index,
            SharedSessionContractImplementor session) throws SQLException {
        if (value == null) {
            st.setNull(index, Types.NUMERIC);
            st.setNull(index + 1, Types.VARCHAR);
        } else {
            st.setBigDecimal(index, value.getAmount());
            st.setString(index + 1, value.getCurrency());
        }
    }

    @Override
    public Money deepCopy(Money value) {
        return value != null ? new Money(value.getAmount(), value.getCurrency()) : null;
    }

    @Override
    public boolean isMutable() {
        return true;
    }
}
```

### 6.3.3 EnhancedUserType（扩展 UserType，支持 XML 序列化）

`EnhancedUserType` 扩展了 `UserType`，额外支持在 `hbm.xml` 文件中以字符串形式配置值：

```java
public interface EnhancedUserType<J> extends UserType<J> {
    String toSqlLiteral(J value);          // 将值转换为 SQL 字面量
    String toString(J value);              // 用于 XML 序列化
    J fromStringValue(CharSequence sequence); // 从 XML 反序列化
}
```

### 6.3.4 ParameterizedType（支持参数化配置）

`ParameterizedType` 允许在使用 `@Type` 时传递参数：

```java
@Type(type = "com.example.type.EncryptedStringType",
      parameters = @Parameter(name = "algorithm", value = "AES"))
private String secretData;
```

```java
public class EncryptedStringType implements UserType<String>, ParameterizedType {
    private String algorithm;

    @Override
    public void setParameterValues(Properties parameters) {
        this.algorithm = parameters.getProperty("algorithm", "AES");
    }

    // nullSafeGet / nullSafeSet 中使用 algorithm 参数
}
```

### 6.3.5 注册自定义类型

```java
// 方式一：通过 @TypeDef 在实体上注册
@Entity
@TypeDef(name = "json", typeClass = JsonType.class)
public class Config {
    @Type(type = "json")
    private Map<String, Object> settings;
}

// 方式二：通过 MetadataBuilder 全局注册
Metadata metadata = new MetadataSources(registry)
    .addAnnotatedClass(Config.class)
    .getMetadataBuilder()
    .applyBasicType(new JsonType())  // 全局注册
    .build();

// 方式三：通过 package-info.java 包级别注册
@TypeDefs({
    @TypeDef(name = "json", typeClass = JsonType.class),
    @TypeDef(name = "encrypted", typeClass = EncryptedStringType.class)
})
package com.example.entity;
```
