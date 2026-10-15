# 1. 源码分析概述

## 1.1 Hibernate 源码结构

### 1.1.1 核心模块概览

Hibernate ORM 采用 Maven 多模块结构，按功能领域划分：

| 模块 | 职责 | 关键内容 |
|------|------|----------|
| `hibernate-core` | 核心 ORM 功能 | SessionFactory、Session、映射元数据、HQL/SQL 生成、类型系统、缓存、事件系统 |
| `hibernate-envers` | 审计与版本管理 | 自动记录实体变更历史，生成 `_AUD` 审计表 |
| `hibernate-spatial` | 地理空间数据支持 | 扩展 Dialect 和 Type，支持 Geometry/Point 等空间类型 |
| `hibernate-jpamodelgen` | JPA 静态元模型生成 | 注解处理器，编译期生成 `EntityName_` 类 |
| `hibernate-c3p0` | C3P0 连接池集成 | 实现 `ConnectionProvider` 接口 |
| `hibernate-hikaricp` | HikariCP 连接池集成 | 实现 `ConnectionProvider` 接口 |
| `hibernate-jcache` | JCache (JSR-107) 二级缓存 | 实现 `RegionFactory` |
| `hibernate-ehcache` | Ehcache 二级缓存 | 实现 `RegionFactory` |
| `hibernate-graalvm` | GraalVM Native Image 支持 | AOT 编译支持 |
| `hibernate-micrometer` | Micrometer 指标集成 | 监控统计导出 |
| `hibernate-ant` | Ant 任务 | Schema Export 等 |
| `hibernate-maven-plugin` | Maven 插件 | Schema Export 等 |

### 1.1.2 关键包的职责

```java
// ========== 配置与启动 ==========
// org.hibernate.cfg -- 配置解析
org.hibernate.cfg.Configuration          // 传统配置入口
org.hibernate.cfg.AvailableSettings      // 所有可配置属性名称常量

// org.hibernate.boot -- 统一启动引导（Hibernate 5.0+ 推荐）
org.hibernate.boot.MetadataSources       // 元数据源集合
org.hibernate.boot.MetadataBuilder       // 元数据构建器
org.hibernate.boot.SessionFactoryBuilder // SessionFactory 构建器
org.hibernate.boot.registry.*            // 注册表构建
org.hibernate.boot.spi.*                 // SPI 扩展接口

// ========== 内部引擎 ==========
// org.hibernate.engine -- 内部引擎
org.hibernate.engine.spi.SessionImplementor       // Session 内部契约
org.hibernate.engine.spi.SessionFactoryImplementor // SessionFactory 内部契约
org.hibernate.engine.spi.PersistenceContext       // 持久化上下文接口
org.hibernate.engine.spi.EntityKey                // 实体缓存键
org.hibernate.engine.spi.CollectionKey            // 集合缓存键
org.hibernate.engine.spi.EntityEntry              // 实体条目（含快照）
org.hibernate.engine.spi.Status                   // 实体状态枚举
org.hibernate.engine.transaction.*                // 事务管理

// ========== 持久化器 ==========
// org.hibernate.persister -- 实体/集合持久化元数据
org.hibernate.persister.entity.EntityPersister          // 实体持久化器接口
org.hibernate.persister.entity.SingleTableEntityPersister    // 单表继承
org.hibernate.persister.entity.JoinedSubclassEntityPersister // 连接子类继承
org.hibernate.persister.entity.UnionSubclassEntityPersister  // 联合子类继承
org.hibernate.persister.collection.CollectionPersister  // 集合持久化器

// ========== 数据加载 ==========
// org.hibernate.loader -- SQL 生成与结果集映射
org.hibernate.loader.Loader                      // 数据加载器基类
org.hibernate.loader.entity.EntityLoader         // 实体加载
org.hibernate.loader.collection.CollectionLoader // 集合加载
org.hibernate.sql.Select    // SELECT 子句生成
org.hibernate.sql.Insert    // INSERT 子句生成
org.hibernate.sql.Update    // UPDATE 子句生成
org.hibernate.sql.Delete    // DELETE 子句生成

// ========== 操作队列 ==========
// org.hibernate.action -- 操作队列（Flush 时按序执行）
org.hibernate.action.internal.EntityInsertAction
org.hibernate.action.internal.EntityUpdateAction
org.hibernate.action.internal.EntityDeleteAction
org.hibernate.action.internal.CollectionRecreateAction
org.hibernate.action.internal.CollectionRemoveAction
org.hibernate.action.internal.CollectionUpdateAction
org.hibernate.action.internal.OrphanRemovalAction
org.hibernate.engine.spi.ActionQueue              // 统一的操作队列门面

// ========== 事件系统 ==========
org.hibernate.event.spi.EventType              // 事件类型枚举
org.hibernate.event.internal.DefaultPersistEventListener
org.hibernate.event.internal.DefaultLoadEventListener
org.hibernate.event.internal.DefaultMergeEventListener
org.hibernate.event.internal.DefaultDeleteEventListener
org.hibernate.event.internal.DefaultFlushEventListener
org.hibernate.event.internal.DefaultFlushEntityEventListener

// ========== 类型系统 ==========
// Hibernate 5:
org.hibernate.type.Type                    // 顶层类型接口
org.hibernate.type.StringType              // String -> VARCHAR
org.hibernate.type.IntegerType             // Integer -> INTEGER
org.hibernate.type.EntityType              // 实体引用类型
org.hibernate.type.CollectionType          // 集合类型
org.hibernate.type.ComponentType           // 组件/嵌入式类型
// Hibernate 6:
org.hibernate.type.descriptor.java.JavaType     // Java 侧类型描述
org.hibernate.type.descriptor.jdbc.JdbcType     // JDBC 侧类型描述

// ========== 方言 ==========
org.hibernate.dialect.Dialect                   // 方言基类
org.hibernate.dialect.MySQLDialect              // MySQL
org.hibernate.dialect.PostgreSQLDialect         // PostgreSQL
org.hibernate.dialect.OracleDialect             // Oracle

// ========== 缓存 ==========
org.hibernate.cache.spi.RegionFactory           // 缓存区域工厂
org.hibernate.cache.spi.EntityRegion            // 实体缓存区域
org.hibernate.cache.spi.QueryResultsRegion      // 查询结果缓存
org.hibernate.cache.internal.StandardQueryCache  // 标准查询缓存

// ========== 查询 ==========
org.hibernate.query.Query                        // 查询接口
org.hibernate.query.sqm.*                        // SQM (Hibernate 6)

// ========== 代理与懒加载 ==========
org.hibernate.proxy.HibernateProxy               // 代理接口
org.hibernate.proxy.LazyInitializer              // 懒加载初始化器
org.hibernate.proxy.pojo.bytebuddy.ByteBuddyProxyFactory
org.hibernate.collection.internal.*              // PersistentSet/List/Bag/Map 实现
```

## 1.2 源码学习方法

学习 Hibernate 源码建议采用两条主线：

**主线一：启动流程（配置 -> SessionFactory）**

从 `MetadataSources` -> `StandardServiceRegistry` -> `Metadata` -> `SessionFactory` 追踪，理解配置转化为运行时元数据和执行器的过程。

```
MetadataSources -> BootstrapServiceRegistry -> StandardServiceRegistry 
-> MetadataBuilder -> Metadata -> SessionFactoryBuilder -> SessionFactoryImpl
```

**主线二：请求处理流程（Session 操作）**

以 `persist()` / `find()` / `merge()` / `remove()` 为入口，跟踪事件触发、ActionQueue 排队、SQL 生成、JDBC 执行、结果集映射的全过程。

```
Session.find() -> LoadEvent -> DefaultLoadEventListener -> Loader.loadEntity()
  -> AbstractEntityPersister.load() -> SQL 生成 -> ResultSet 处理
  -> TwoPhaseLoad.initializeEntity() -> PersistenceContext 注册
```

**辅助方法：**

- 核心类加注释阅读（关键方法逐行理解）
- 调用链跟踪（IDE 的 Call Hierarchy 功能，从 SPI 接口深入到具体实现）
- 关注接口与实现分离的设计（`org.hibernate.*.spi.*` 为 SPI 接口，`org.hibernate.*.internal.*` 为默认实现）
- 利用 Hibernate 源码自带的单元测试作为实际调试入口
- 阅读 `package-info.java` 了解每个包的职责定位

---

# 2. 启动流程源码分析

## 2.1 Configuration / MetadataSources 加载

### 2.1.1 MetadataSources 初始化

`MetadataSources` 是 Hibernate 5 引入的统一配置入口，替代了传统的 `Configuration` 类。它本身不解析内容，而是收集"元数据来源"。

```java
// Hibernate 5/6 推荐用法
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .configure("hibernate.cfg.xml")  // 加载主配置文件
    .build();

MetadataSources sources = new MetadataSources(registry);
sources.addAnnotatedClass(User.class);   // 注册带注解的实体
sources.addAnnotatedClassName("com.example.Order"); // 通过全限定类名注册
sources.addResource("com/example/legacy.hbm.xml"); // 注册 hbm.xml

Metadata metadata = sources.buildMetadata();    // 触发实际构建
SessionFactory sf = metadata.buildSessionFactory();
```

核心内部流程：
1. `MetadataSources` 构造函数接收 `ServiceRegistry`，初始化内部处理组件
2. 调用 `addAnnotatedClass()` 等方法仅将来源存入内部集合，不做任何解析
3. 调用 `buildMetadata()` 时，触发真正的构建流程

### 2.1.2 扫描 @Entity 注解

注解扫描由 `AnnotationMetadataSourceProcessorImpl` 处理：

```java
// 简化调用链
MetadataSources.buildMetadata()
  -> MetadataBuilder.buildMetadata()
    -> MetadataBuildingProcess.complete()
      -> AnnotationMetadataSourceProcessorImpl.prepare()
        -> 遍历所有注册类，对每个类：
          1. 检查是否有 @Entity / @MappedSuperclass / @Embeddable 注解
          2. 使用 ReflectionManager 获取类信息（字段、方法、注解）
          3. 调用 AnnotationBinder.bindClass() 绑定映射信息
```

`AnnotationBinder.bindClass()` 是核心注解解析器：

```java
// 在 org.hibernate.cfg.AnnotationBinder 中
public static void bindClass(
    XClass classToProcess,
    Map<XClass, InheritanceState> inheritanceStatePerClass,
    MetadataBuildingOptions options,
    InFlightMetadataCollector collector
) {
    // 1. 解析类级别注解：@Entity、@Table、@Inheritance、@DiscriminatorColumn 等
    // 2. 解析 @Id / @EmbeddedId
    // 3. 解析属性级注解：@Column、@ManyToOne、@OneToMany、@JoinColumn 等
    // 4. 处理 @IdClass 或 @EmbeddedId
    // 5. 处理级联（cascade）、抓取（fetch）策略
    // 6. 处理 @Version 乐观锁
    // 7. 处理关联映射表（@JoinTable、@CollectionTable）
    // 8. 将解析结果写入 InFlightMetadataCollector
}
```

### 2.1.3 解析 hbm.xml / orm.xml

XML 映射文件通过 `MappingBinder` 绑定：

```java
// hbm.xml 的解析链
MetadataSources.addResource("legacy.hbm.xml")
  -> 将 XML 文件路径存入 sources
    -> buildMetadata()
      -> MappingBinder.bind()
        -> 使用 JAXB 将 XML 反序列化为 HbmHbmMapping 对象树
        -> 遍历 <class>、<subclass>、<union-subclass> 等元素
        -> 逐个调用 collectEntity() / collectTable() / collectColumn()
```

### 2.1.4 InFlightMetadataCollector 元数据收集器

```java
// org.hibernate.boot.spi.InFlightMetadataCollector
interface InFlightMetadataCollector {
    void addEntityBinding(PersistentClass persistentClass);
    void addCollectionBinding(Collection collection);
    void addTable(Table table);
    void addNamedQuery(String name, NamedQueryDefinition definition);
    void addTypeDefinition(TypeDefinition typeDefinition);
    void addFilterDefinition(FilterDefinition filterDefinition);
    void registerIdentifierGenerator(String name, Class<?> generatorClass);
}
```

构建完成后转换为不可变的 `MetadataImpl`：

```java
// 在 MetadataBuildingProcess.complete() 中
MetadataImpl metadata = new MetadataImpl(
    uuid, metadataBuildingOptions, typeConfiguration,
    inFlightMetadataCollector.getEntityBindingMap(),
    inFlightMetadataCollector.getCollectionBindingMap(),
    // ... 更多映射数据
);
```

## 2.2 ServiceRegistry 构建

### 2.2.1 BootstrapServiceRegistryBuilder

`BootstrapServiceRegistry` 是最底层的服务注册表，生命周期与 JVM 内所有 SessionFactory 绑定。

```java
BootstrapServiceRegistry bootstrapRegistry = 
    new BootstrapServiceRegistryBuilder()
        .applyClassLoader(customClassLoader)
        .applyIntegrator(customIntegrator)
        .build();
```

**核心职责：**

1. **ClassLoader 管理** -- `ClassLoaderService`：提供统一的类加载服务。

```java
public class ClassLoaderServiceImpl implements ClassLoaderService {
    public <T> Class<T> classForName(String className) {
        // 按顺序尝试：appCL -> resourceCL -> hibernateCL -> jpaTempCL
    }
}
```

2. **Integrator 加载** -- 利用 Java SPI `ServiceLoader` 机制：

```
META-INF/services/org.hibernate.integrator.spi.Integrator
```

```java
public interface Integrator {
    void integrate(
        Metadata metadata,
        BootstrapSessionFactoryBuilder bootstrapSessionFactoryBuilder,
        SessionFactoryServiceRegistry serviceRegistry
    );
    void disintegrate(
        SessionFactoryImplementor sessionFactory,
        SessionFactoryServiceRegistry serviceRegistry
    );
}
```

3. **StrategySelector** -- 策略选择器：允许用户通过配置切换实现类。

```java
public interface StrategySelector {
    <T> Class<? extends T> selectStrategyImplementor(Class<T> strategyType, String name);
    <T> T resolveStrategy(Class<T> strategyType, Object strategyReference);
}
```

### 2.2.2 StandardServiceRegistryBuilder

```java
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .configure()
    .loadProperties("hibernate.properties")
    .applySetting("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect")
    .applySetting("hibernate.hbm2ddl.auto", "update")
    .build();
```

**核心服务注册过程：**

```java
// StandardServiceRegistryBuilder.build() 内部
// 步骤 1: 构建 BootstrapServiceRegistry
bootstrapRegistry = new BootstrapServiceRegistryBuilder().build();

// 步骤 2: 初始化设置（读取配置文件、合并属性）

// 步骤 3: 按依赖顺序逐个初始化服务
for (StandardServiceInitiator<?> initiator : standardInitiators) {
    serviceRegistry.registerService(initiator.getServiceInitiated(), 
        initiator.initiateService(configValues, serviceRegistry));
}
```

**核心服务列表：**

| 服务接口 | 职责 |
|----------|------|
| `ConnectionProvider` | 获取 JDBC 连接 |
| `DialectResolver` | 自动检测数据库方言 |
| `JdbcServices` | JDBC 操作抽象 |
| `TransactionFactory` | 事务工厂 |
| `JtaPlatform` | JTA 平台集成 |

**方言自动解析：**
```java
public Dialect resolveDialect(DialectResolutionInfo info) {
    String databaseName = info.getDatabaseName();     // 如 "MySQL"
    int majorVersion = info.getDatabaseMajorVersion();
    for (DialectResolver resolver : resolvers) {
        Dialect dialect = resolver.resolveDialect(info);
        if (dialect != null) return dialect;
    }
    return null;
}
```

## 2.3 Metadata 构建

### 2.3.1 MetadataBuilder 工作流

```java
MetadataBuilder builder = sources.getMetadataBuilder();
builder.applyImplicitSchemaName("my_schema");
builder.applyImplicitCatalogName("my_catalog");
builder.applyPhysicalNamingStrategy(new PhysicalNamingStrategyStandardImpl());
builder.enableNewIdentifierGeneratorSupport(true);

Metadata metadata = builder.build();
```

**`MetadataBuilder.build()` 内部调用链：**

```java
MetadataBuilderImpl.build()
  -> MetadataBuildingProcess.complete(sources, options)
    -> 阶段一: ManagedResources -- 收集所有实体类、映射文件
    -> 阶段二: Scanning -- 若启用自动扫描
    -> 阶段三: Binding -- AnnotationBinder.bindClass() 逐个解析注解
    -> 阶段四: 索引创建 -- processIndexes()
    -> 阶段五: 唯一约束创建 -- processUniqueConstraints()
    -> 阶段六: 完成 -- InFlightMetadataCollector -> MetadataImpl
```

### 2.3.2 映射元数据存储结构

```java
public class MetadataImpl implements MetadataImplementor {
    private final Map<String, PersistentClass> entityBindingMap;
    // Key: 实体名  Value: PersistentClass（运行时映射元数据）
    
    private final Map<String, Collection> collectionBindingMap;
    // Key: 集合角色名（如 "com.example.User.addresses"）
    
    private final Map<String, Table> tableMap;
    // Key: 表限定名
    
    private final Map<String, NamedQueryDefinition> namedQueryMap;
    // Key: 查询名（@NamedQuery(name)）
}
```

### 2.3.3 EntityPersister 预创建

Metadata 构建阶段只存储 PersistentClass 及映射信息，实际的 EntityPersister 在 SessionFactory 构建阶段才创建：

```java
// SessionFactory 构建阶段:
SessionFactoryBuilderImpl.build()
  -> SessionFactoryImpl 构造函数
    -> 遍历 entityBindingMap
    -> 调用 EntityPersisterFactory.create()
       -> 根据继承策略选择：
          if (单表继承) -> new SingleTableEntityPersister(...)
          if (联合继承) -> new JoinedSubclassEntityPersister(...)
          if (每类一张表) -> new UnionSubclassEntityPersister(...)
```

### 2.3.4 命名策略应用

```java
// 1. 隐式命名策略 -- 决定未显式指定的名称
public interface ImplicitNamingStrategy {
    Identifier determinePrimaryTableName(ImplicitEntityNameSource source);
    Identifier determineJoinTableName(ImplicitJoinTableNameSource source);
    Identifier determineBasicColumnName(ImplicitBasicColumnNameSource source);
    Identifier determineJoinColumnName(ImplicitJoinColumnNameSource source);
    Identifier determineIdentifierColumnName(ImplicitIdentifierColumnNameSource source);
}

// 2. 物理命名策略 -- 决定最终的数据库标识符
public interface PhysicalNamingStrategy {
    Identifier toPhysicalTableName(Identifier name, JdbcEnvironment jdbcEnvironment);
    Identifier toPhysicalColumnName(Identifier name, JdbcEnvironment jdbcEnvironment);
    Identifier toPhysicalSequenceName(Identifier name, JdbcEnvironment jdbcEnvironment);
}
```

### 2.3.5 属性验证阶段

Metadata 构建最后阶段的验证项：
- 实体类必须有无参构造器（或通过 HibernateProxy 可创建）
- @Id 不能为 final
- @OneToMany / @ManyToMany 集合必须是接口类型（Set/List/Map）
- 关联关系双方 mappedBy 一致性检查
- Identifier 生成策略是否合法
- 类型映射是否有效

## 2.4 SessionFactory 构建

### 2.4.1 SessionFactoryBuilder -> SessionFactoryImpl

```java
SessionFactory sessionFactory = metadata.getSessionFactoryBuilder()
    .applyInterceptor(new MyInterceptor())
    .applyBeanManager(cdiBeanManager)
    .applyValidatorFactory(validatorFactory)
    .build();
```

**`SessionFactoryBuilderImpl.build()` 内部流程：**

```java
SessionFactoryBuilderImpl.build()
  -> SessionFactoryImpl 构造函数:
    1. 创建 SessionFactoryServiceRegistryImpl
       - 注册 SessionFactory 级别的服务
       - 调用所有 Integrator.integrate()
    
    2. 创建所有 EntityPersister
       for (PersistentClass pc : entityBindingMap.values()) {
           EntityPersister ep = factory.createEntityPersister(pc, ...);
       }
    
    3. 创建所有 CollectionPersister
       for (Collection col : collectionBindingMap.values()) {
           new BasicCollectionPersister(col, ...);
       }
    
    4. 初始化查询计划缓存
       queryPlanCache = new QueryPlanCache(this);
    
    5. 初始化二级缓存
       if (regionFactory != null) {
           regionFactory.start(sessionFactoryOptions, properties);
       }
    
    6. 生成 SessionFactory UUID
```

### 2.4.2 EntityPersister 创建

`EntityPersister` 是 Hibernate 最核心的内部组件之一，封装了实体的所有映射知识和 SQL 生成逻辑：

```java
public interface EntityPersister {
    // --- 标识符 ---
    Serializable getIdentifier(Object entity, SharedSessionContractImplementor session);
    void setIdentifier(Object entity, Object id, SharedSessionContractImplementor session);
    
    // --- 属性读写 ---
    Object[] getPropertyValues(Object entity);
    void setPropertyValues(Object entity, Object[] values);
    Object getPropertyValue(Object entity, int i);
    
    // --- CRUD SQL 生成 ---
    String generateSelectString();
    String[] getIdentifierSelectSQL();
    
    // --- 脏检查 ---
    int[] findDirty(Object[] currentState, Object[] previousState,
                     Object owner, SharedSessionContractImplementor session);
    
    // --- ResultSet 处理 ---
    Object[] hydrate(ResultSet rs, Serializable id, Object object, ...);
    Object resolve(Object id, ResultSet rs, Object owner, SharedSessionContractImplementor session);
    
    // --- 锁 ---
    LockMode getLockMode(Object entity);
}
```

### 2.4.3 CollectionPersister 创建

```java
public class BasicCollectionPersister implements CollectionPersister {
    // 构造函数中初始化:
    // 1. 根据 Collection 元数据生成 SELECT/INSERT/UPDATE/DELETE SQL
    // 2. 确定集合元素的类型处理器
    // 3. 确定关联键的映射信息
    
    private final String sqlSelectString;      // 加载所有元素
    private final String sqlSelectRowString;   // 根据元素标识加载
    private final String sqlInsertRowString;   // 插入单个元素
    private final String sqlUpdateRowString;   // 更新单个元素
    private final String sqlDeleteRowString;   // 删除单个元素
    private final String sqlDeleteString;      // 删除所有元素
}
```

### 2.4.4 查询计划缓存初始化

```java
public class QueryPlanCache {
    private final BoundedConcurrentHashMap<HQLQueryPlanKey, HQLQueryPlan> hqlPlanCache;
    private final BoundedConcurrentHashMap<NativeSQLQueryPlanKey, NativeSQLQueryPlan> nativePlanCache;
    private final BoundedConcurrentHashMap<String, FilterQueryPlan> filterPlanCache;
    
    // 缓存大小的默认值: hibernate.query.plan_cache_max_size (默认 2048)
    private final int maxQueryPlanCount;
}
```

### 2.4.5 RegionFactory 初始化（二级缓存）

```java
// 二级缓存的启动流程
if (useSecondLevelCache) {
    RegionFactory regionFactory = 
        strategySelector.resolveStrategy(RegionFactory.class, 
            properties.get(AvailableSettings.CACHE_REGION_FACTORY));
    // 常见实现:
    //   EhcacheRegionFactory (Ehcache 2.x)
    //   JCacheRegionFactory (JSR-107 / Ehcache 3.x)
    
    regionFactory.start(sessionFactoryOptions, properties);
    
    // 构建各种 Region:
    //    EntityRegion -- 存储完整实体对象
    //    CollectionRegion -- 存储集合元素
    //    QueryResultsRegion -- 存储查询结果
    //    TimestampsRegion -- 存储表最后更新时间戳
}
```

### 2.4.6 SessionFactoryServiceRegistry 创建

```java
// 在 SessionFactoryImpl 构造函数内创建
SessionFactoryServiceRegistryImpl serviceRegistry = 
    new SessionFactoryServiceRegistryImpl(standardServiceRegistry, sessionFactory);

serviceRegistry.registerService(EventManager.class, eventManager);
serviceRegistry.registerService(StatisticsImplementor.class, statistics);

// 调用所有 Integrator.integrate()
for (Integrator integrator : integrators) {
    integrator.integrate(metadata, this, serviceRegistry);
}
```

## 2.5 启动时序图

以下用文字调用链描述从零到 SessionFactory 的完整启动过程：

```
[用户代码]
  |
  +- new StandardServiceRegistryBuilder()
  |   |
  |   +- .configure()                     // 加载 hibernate.cfg.xml
  |   +- .build()
  |       |
  |       +- new BootstrapServiceRegistryBuilder().build()
  |       |   +- ClassLoaderServiceImpl 初始化
  |       |   +- IntegratorServiceImpl 初始化
  |       |   |   +- ServiceLoader.load(Integrator.class)
  |       |   |       +- 读取 META-INF/services/...Integrator
  |       |   +- StrategySelectorImpl 初始化
  |       |
  |       +- 逐个调用 StandardServiceInitiator.initiateService()
  |           +- ConnectionProviderInitiator -> ConnectionProvider
  |           +- DialectResolverInitiator -> DialectResolver
  |           +- JdbcServicesInitiator -> JdbcServicesImpl
  |           +- TransactionFactoryInitiator -> TransactionFactory
  |
  +- new MetadataSources(registry)
  |   +- .addAnnotatedClass(User.class)
  |   +- .addResource("queries.hbm.xml")
  |
  +- sources.buildMetadata()
  |   +- MetadataBuildingProcess.complete()
  |       +- [阶段 1] ManagedResources 处理
  |       +- [阶段 2] 扫描
  |       +- [阶段 3] 绑定
  |       |   +- AnnotationBinder.bindClass()
  |       |   |   +- 解析 @Entity / @Table / @Id / @Column
  |       |   |   +- 解析 @ManyToOne / @OneToMany 等关联
  |       |   |   +- 写入 InFlightMetadataCollector
  |       |   +- MappingBinder.bind() -> 解析 hbm.xml
  |       +- [阶段 4] 索引处理
  |       +- [阶段 5] 唯一约束处理
  |       +- [阶段 6] 完成 -> new MetadataImpl(...)
  |
  +- metadata.buildSessionFactory()
      +- SessionFactoryBuilderImpl.build()
          +- new SessionFactoryImpl(...)
              +- 创建 SessionFactoryServiceRegistryImpl
              +- 遍历 entityBindingMap
              |   +- EntityPersisterFactory.create()
              |       +- new SingleTableEntityPersister()
              |           +- 生成 SELECT/INSERT/UPDATE/DELETE SQL
              +- 遍历 collectionBindingMap
              |   +- new BasicCollectionPersister()
              +- new QueryPlanCache()
              +- regionFactory.start()
              +- 调用 Integrator.integrate()
              +- 返回 SessionFactoryImpl
```

---

# 3. Session 创建与实现

## 3.1 SessionFactory.openSession()

### 3.1.1 SessionFactoryImpl.openSession() 入口

```java
// SessionFactoryImpl.openSession()
public Session openSession() {
    return withOptions().openSession();
}

// SessionBuilderImpl.openSession()
public Session openSession() {
    final JdbcConnectionAccess jdbcConnectionAccess = 
        new JdbcConnectionAccessImpl(sessionFactory);
    
    return new SessionImpl(
        sessionFactory,            // SessionFactoryImplementor
        jdbcConnectionAccess,      // 管理 JDBC 连接生命周期
        interceptor,               // 用户自定义拦截器
        jdbcBatchSize,             // JDBC 批处理大小
        isRepresentsNewTransaction,
        persistenceUnitName,       // 持久化单元名称（JPA）
        exceptionHandler           // 异常处理器
    );
}
```

### 3.1.2 SessionImpl 的创建过程

```java
public SessionImpl(
    SessionFactoryImpl factory,
    JdbcConnectionAccess jdbcConnectionAccess,
    Interceptor interceptor,
    Integer jdbcBatchSize,
    boolean openNewTransactions,
    String persistenceUnitName,
    ExceptionHandler exceptionHandler
) {
    // 1. 持有 SessionFactory 引用
    this.factory = factory;
    
    // 2. 初始化持久化上下文
    this.persistenceContext = new StatefulPersistenceContext(this);
    
    // 3. 初始化操作队列
    this.actionQueue = new ActionQueue(this);
    
    // 4. 初始化 JDBC 协调器（连接暂时不获取）
    this.jdbcCoordinator = new JdbcCoordinatorImpl(jdbcConnectionAccess, this);
    
    // 5. 初始化事务协调器
    this.transactionCoordinator = factory.getTransactionFactory()
        .createTransactionCoordinator(this);
    
    // 6. 设置拦截器
    this.interceptor = interceptor != null ? interceptor : EmptyInterceptor.INSTANCE;
    
    // 7. 设置刷新模式默认值
    this.flushMode = FlushMode.AUTO;
    
    // 8. 关联事件管理器
    this.eventManager = factory.getEventManager();
}
```

### 3.1.3 连接延迟获取策略

```java
public class JdbcConnectionAccessImpl implements JdbcConnectionAccess {
    private final ConnectionProvider connectionProvider;
    
    public Connection obtainConnection() {
        // 尝试从当前 JTA 事务获取连接
        // 若没有，通过 ConnectionProvider 获取新连接
        return connectionProvider.getConnection();
    }
}

// 连接的实际获取时机:
//   1. 首次 flush()
//   2. 首次执行查询
//   3. 手动调用 session.getTransaction().begin()
// 延迟获取避免了不必要的连接占用
```

### 3.1.4 ActionQueue 初始化

```java
public class ActionQueue {
    private final ExecutableList<EntityInsertAction> insertions;
    private final ExecutableList<EntityUpdateAction> updates;
    private final ExecutableList<EntityDeleteAction> deletions;
    private final ExecutableList<CollectionRecreateAction> collectionCreations;
    private final ExecutableList<CollectionUpdateAction> collectionUpdates;
    private final ExecutableList<CollectionRemoveAction> collectionRemovals;
    
    private transient List<AfterTransactionCompletionProcess> afterTransactionProcesses;
    private transient List<BeforeTransactionCompletionProcess> beforeTransactionProcesses;
    
    public ActionQueue(SessionImplementor session) {
        this.insertions = new ExecutableList<>(true);
        this.updates = new ExecutableList<>(true);
        this.deletions = new ExecutableList<>(true);
        // ...
    }
}
```

### 3.1.5 EventManager 关联

```java
// EventManager 维护了 EventType -> EventListener 的映射:
// EventType.PERSIST  -> DefaultPersistEventListener
// EventType.LOAD    -> DefaultLoadEventListener
// EventType.MERGE   -> DefaultMergeEventListener
// EventType.DELETE  -> DefaultDeleteEventListener
// EventType.FLUSH   -> DefaultFlushEventListener
```

## 3.2 SessionImpl 内部结构

### 3.2.1 核心属性

```java
public final class SessionImpl 
    extends AbstractSessionImpl 
    implements SessionImplementor, EventSource 
{
    private transient PersistenceContext persistenceContext;   // 一级缓存
    private transient ActionQueue actionQueue;                // 操作队列
    private transient JdbcCoordinatorImpl jdbcCoordinator;    // JDBC 协调器
    private transient TransactionCoordinatorImpl transactionCoordinator;
    private transient Interceptor interceptor;
    private FlushMode flushMode = FlushMode.AUTO;
    private transient BatchBuilder batchBuilder;
    private transient CacheMode cacheMode = CacheMode.NORMAL;
    private transient LoadQueryInfluencers loadQueryInfluencers;
}
```

### 3.2.2 SessionImpl 继承层次

```
java.lang.Object
  +-- org.hibernate.internal.AbstractSessionImpl
        implements Serializable, SharedSessionContractImplementor
        (提供基础属性: factory, exceptionConverter)
        |
        +-- org.hibernate.internal.SessionImpl
              implements SessionImplementor, EventSource
              (持久化操作的核心实现)
              |
              +-- Session (对外 API, EntityManager)
                    +-- SharedSessionContract (内部契约)
                          +-- SessionImplementor (引擎可见)
                                +-- EventSource (事件源)
```

**关键接口职责：**

```java
// Session -- 用户直接调用的 API
interface Session extends SharedSessionContract, EntityManager {
    void persist(Object entity);
    <T> T find(Class<T> entityClass, Object primaryKey);
    <T> T merge(T entity);
    void remove(Object entity);
    void flush();
}

// SessionImplementor -- Hibernate 内部组件可见
interface SessionImplementor extends SharedSessionContractImplementor {
    PersistenceContext getPersistenceContext();
    ActionQueue getActionQueue();
    JdbcCoordinator getJdbcCoordinator();
}
```

## 3.3 PersistenceContext 实现

### 3.3.1 StatefulPersistenceContext 的数据结构

```java
public class StatefulPersistenceContext implements PersistenceContext {
    
    // 1. 按主键索引的实体缓存 (EntityKey = EntityPersister + 实体 ID)
    private Map<EntityKey, Object> entityByKey;
    
    // 2. 按唯一键索引的实体缓存（用于 @NaturalId）
    private Map<EntityUniqueKey, Object> entityByUniqueKey;
    
    // 3. 代理对象缓存（已创建但尚未初始化的代理对象）
    private Map<EntityKey, Object> proxyByKey;
    
    // 4. 实体快照缓存（脏检查核心）
    //    Key: EntityKey, Value: Object[] -- 实体加载时的原始属性值数组
    private Map<EntityKey, Object[]> entitySnapshotsByKey;
    
    // 5. 可空外键实体 Key 集合
    private HashSet<EntityKey> nullifiableEntityKeys;
    
    // 6. 集合缓存
    //    CollectionKey = CollectionPersister.getRole() + Owner EntityKey
    private Map<CollectionKey, PersistentCollection> collectionsByKey;
    
    // 7. 父子关联索引（用于级联操作）
    private HashMap<Object, HashSet<Object>> childrenByParent;
    
    // 8. 批量抓取队列
    private BatchFetchQueue batchFetchQueue;
    
    private int cascading;             // 当前级联操作深度
    private int loadCounter;           // 加载计数器
    private boolean flushing;          // 是否正在执行 flush
}
```

### 3.3.2 EntityKey 构成

```java
public final class EntityKey implements Serializable {
    private final Serializable identifier;        // 实体 ID 值
    private final String entityName;              // 实体名称
    private final String rootEntityName;          // 继承根实体名称
    private final Type identifierType;            // ID 的类型处理器
    private final int hashCode;                   // 预计算 hashCode
    
    public EntityKey(Serializable id, EntityPersister persister) {
        this.identifier = id;
        this.entityName = persister.getEntityName();
        this.rootEntityName = persister.getRootEntityName();
        this.identifierType = persister.getIdentifierType();
        this.hashCode = generateHashCode();
    }
}
```

### 3.3.3 EntityEntry 结构

```java
public final class EntityEntry {
    private final Status status;
    // MANAGED / READ_ONLY / DELETED / GONE / LOADING / SAVING
    
    private Object[] loadedState;
    // 实体被加载时的所有属性值数组（Object[]）
    // 脏检查时，将当前状态与此快照逐项比较
    
    private Object[] deletedState;
    // 实体被删除时的属性值数组（用于事务回滚时还原）
    
    private final Object version;           // @Version 乐观锁版本号
    private final LockMode lockMode;        // 加载时获取的锁模式
    private final EntityPersister persister; // 对应实体的持久化器
    private final EntityKey entityKey;       // 当前实体的 key
    private final boolean existsInDatabase;  // 在数据库中是否存在
}
```

## 3.4 Session 的代理机制

### 3.4.1 ByteBuddy 代理生成

Hibernate 5.3+ 默认使用 ByteBuddy 作为字节码增强和代理生成的工具。

```java
public class ByteBuddyProxyFactory implements ProxyFactory {
    
    public HibernateProxy getProxy(
        Serializable id,
        SharedSessionContractImplementor session
    ) {
        // 1. 创建 LazyInitializer
        LazyInitializer li = new LazyInitializer(entityName, id, session);
        
        // 2. 使用 ByteBuddy 创建代理子类
        Class<?> proxyClass = generateProxyClass(entityClass);
        HibernateProxy proxy = proxyClass.getConstructor(LazyInitializer.class)
            .newInstance(li);
        
        return proxy;
    }
    
    private Class<?> generateProxyClass(Class<?> entityClass) {
        return new ByteBuddy()
            .subclass(entityClass)                        // 继承实体类
            .implement(HibernateProxy.class)              // 实现 HibernateProxy 接口
            .method(any())                                // 拦截所有方法
            .intercept(MethodDelegation.to(interceptor))  // 委托给拦截器
            .make()
            .load(entityClass.getClassLoader())
            .getLoaded();
    }
}
```

代理要求：
- 不能是 `final` 类
- 必须有无参构造器（至少 `protected`）
- 所有持久化属性的访问方法不能是 `final`

### 3.4.2 HibernateProxy 和 LazyInitializer 接口

```java
public interface HibernateProxy extends Serializable {
    LazyInitializer getHibernateLazyInitializer();
    Object writeReplace();
}

public interface LazyInitializer {
    void initialize();                        // 触发加载
    Object getImplementation();               // 获取被代理的真实实体
    Serializable getIdentifier();             // 获取 ID（不触发加载）
    String getEntityName();
    boolean isUninitialized();
    SharedSessionContractImplementor getSession();
    void setSession(SharedSessionContractImplementor session);
}
```

### 3.4.3 懒加载触发流程

```
用户代码: user.getAddress().getCity()
  |
  +- user 是代理对象，getAddress() 触发属性访问
  |
  +- ByteBuddy 方法拦截器捕获调用
  |   +- ByteBuddyInterceptor.handleInvocation()
  |
  +- 检查 LazyInitializer.isUninitialized() -> true
  |
  +- 调用 LazyInitializer.initialize()
  |   |
  |   +- 获取目标实体的 EntityKey
  |   +- 调用 persister.load(id, targetObject, lockMode, session)
  |   |   +- 生成 SELECT SQL
  |   |   +- JDBC 执行
  |   |   +- ResultSet -> Object[] fields (hydrate)
  |   |   +- Object[] -> 实体对象 (resolve)
  |   +- 注入真实实体，设置 initialized = true
  |
  +- 后续方法调用不再拦截，直接委托给真实对象
  |
  +- 调用原始方法（反射执行真实对象的 getAddress()）
```

### 3.4.4 PersistentCollection 包装

集合关联的懒加载通过 `PersistentCollection` 包装实现：

```java
// 集合包装的对应关系:
//   Set -> PersistentSet
//   List -> PersistentList
//   Bag  -> PersistentBag
//   Map  -> PersistentMap

public abstract class AbstractPersistentCollection 
    implements Serializable, PersistentCollection {
    
    protected CollectionPersister persister;    // 集合的持久化器
    protected Serializable key;                 // 所有者实体 ID
    protected Object owner;                     // 所有者实体
    protected SharedSessionContractImplementor session;
    protected boolean initialized;              // 是否已加载
    protected boolean dirty;                    // 是否被修改
    protected List<Object> storedSnapshot;      // 加载时的快照数据
    
    // 读操作必须调用（确保已初始化）
    protected final void read() {
        if (!initialized) initialize();
    }
    
    // 写操作必须调用（标记 dirty）
    protected final void write() {
        initialize(true);
        dirty = true;
    }
}
```

```java
// PersistentSet 读写示例
public class PersistentSet extends AbstractPersistentCollection implements Set {
    private Set set;
    
    public boolean add(Object value) {
        write();                       // 标记 dirty
        return set.add(value);
    }
    
    public int size() {
        read();                        // 触发加载
        return set.size();
    }
    
    public Iterator iterator() {
        read();
        return new IteratorProxy(set.iterator());
    }
}
```

---

# 4. 持久化操作源码分析

## 4.1 persist() 流程

### 4.1.1 入口与事件触发

```java
// SessionImpl.persist()
public void persist(Object object) {
    checkOpen();
    persist(null, object);
}

public void persist(String entityName, Object object) {
    firePersist(new PersistEvent(entityName, object, this));
}

private void firePersist(PersistEvent event) {
    for (PersistEventListener listener : eventManager.getPersistEventListeners()) {
        listener.onPersist(event);
    }
}
```

**DefaultPersistEventListener.onPersist()：**

```java
public void onPersist(PersistEvent event) {
    EntityPersister persister = getPersister(event);
    
    // 1. 检查是否是 detached 实体（报错）
    if (persister.isTransient(event.getObject(), event.getSession())) {
        // 2. 生成 ID
        Serializable id = persister.getIdentifierGenerator()
            .generate(event.getSession(), event.getObject());
        
        // 3. 设置 ID
        persister.setIdentifier(event.getObject(), id, event.getSession());
        
        // 4. 向 PersistenceContext 注册实体
        event.getSession().getPersistenceContext()
            .addEntity(event.getObject(), Status.SAVING, persister, id);
        
        // 5. 级联处理关联实体
        cascadeOnPersist(event);
        
        // 6. 创建 EntityInsertAction 并加入 ActionQueue
        event.getSession().getActionQueue().addAction(
            new EntityInsertAction(
                id,
                persister.getPropertyValues(event.getObject()),
                event.getObject(),
                persister,
                event.getSession()
            )
        );
    }
}
```

### 4.1.2 ID 生成器调用

```java
public interface IdentifierGenerator {
    Serializable generate(SharedSessionContractImplementor session, Object object);
}

// 常见实现:
//   SequenceStyleGenerator   -- @GeneratedValue(strategy = SEQUENCE)
//   TableGenerator           -- @TableGenerator
//   IdentityGenerator        -- @GeneratedValue(strategy = IDENTITY)
//   UUIDGenerator            -- UUID.randomUUID()
//   Assigned                 -- 用户手动赋值
```

### 4.1.3 EntityInsertAction.execute()

```java
public class EntityInsertAction extends EntityAction {
    private final Object[] state;      // 实体所有属性值
    private final Object version;      // 乐观锁版本号
    
    public void execute() {
        // 1. 生成 INSERT SQL
        String insertSQL = persister.generateInsertString();
        // 如: INSERT INTO T_USER (name, age, version, id) VALUES (?, ?, ?, ?)
        
        // 2. 绑定参数 (dehydrate: 实体属性值 -> PreparedStatement 参数)
        persister.dehydrate(id, state, st, session);
        
        // 3. 执行 SQL
        if (usingBatch) {
            st.addBatch();  // 批处理模式
        } else {
            st.executeUpdate();
        }
        
        // 4. 如果是 IDENTITY 主键，获取生成的值
        if (isIdentityInsert) {
            ResultSet generatedKeys = st.getGeneratedKeys();
            generatedKeys.next();
            id = persister.getIdentifierType().nullSafeGet(generatedKeys, 1, session);
            persister.setIdentifier(instance, id, session);
        }
        
        // 5. 注册 AfterTransactionCompletionProcess
        session.getActionQueue().registerProcess(new AfterTransactionCompletionProcess() {
            public void doAfterTransactionCompletion(boolean success, SharedSessionContractImplementor session) {
                if (success) {
                    persister.cacheAfterInsert(entity, session);  // 更新二级缓存
                }
            }
        });
    }
}
```

## 4.2 find() / load() 流程

### 4.2.1 SessionImpl.find() 流程

```java
// SessionImpl.find() -> immediateLoad()
public <T> T immediateLoad(String entityName, Serializable id) {
    // 1. 首先检查一级缓存 (PersistenceContext)
    PersistenceContext context = getPersistenceContext();
    EntityKey ekey = session.generateEntityKey(id, persister);
    Object entity = context.getEntity(ekey);
    
    if (entity != null) {
        return (T) entity;  // 缓存命中
    }
    
    // 2. 检查二级缓存（如果启用）
    if (persister.hasCache()) {
        entity = persister.getCacheAccessStrategy()
            .get(session, ekey, session.getTimestamp());
        if (entity != null) {
            context.addEntity(ekey, entity);
            return (T) entity;
        }
    }
    
    // 3. 一级、二级缓存均未命中，执行数据库查询
    return (T) loadEntityUsingLoadEventListener(id, persister);
}
```

### 4.2.2 DefaultLoadEventListener 处理流程

```java
// DefaultLoadEventListener.load()
public void load(LoadEvent event, LoadType loadType) {
    EntityPersister persister = getPersister(event);
    Serializable id = event.getEntityId();
    
    // 执行实际的数据库加载
    Object entity = loadFromDatasource(event, persister, ekey, loadType);
    event.setResult(entity);
}

private Object loadFromDatasource(LoadEvent event, EntityPersister persister, 
                                   EntityKey keyToLoad, LoadType options) {
    // 委托给 Loader 执行 SQL
    Object entity = persister.load(
        event.getEntityId(), null, event.getLockOptions(), event.getSession()
    );
    
    // 注册到 PersistenceContext
    event.getSession().getPersistenceContext()
        .addEntity(keyToLoad, entity, Status.MANAGED);
    
    return entity;
}
```

### 4.2.3 Loader.loadEntity() 核心逻辑

```java
// org.hibernate.loader.Loader
public Object loadEntity(
    final Loadable persister,
    final Serializable id,
    final LockOptions lockOptions,
    final SharedSessionContractImplementor session
) {
    // 1. 生成 SELECT SQL
    String sql = persister.getIdentifierSelectSQL();
    // 如: SELECT ... FROM user WHERE id = ?
    
    // 2. 创建 PreparedStatement
    PreparedStatement st = session.getJdbcCoordinator()
        .getStatementPreparer().prepareStatement(sql);
    
    // 3. 绑定 ID 参数
    Type idType = persister.getIdentifierType();
    idType.nullSafeSet(st, id, 1, session);
    
    // 4. 执行查询
    ResultSet rs = st.executeQuery();
    
    // 5. 处理结果集
    if (rs.next()) {
        return processResultSet(rs, persister, session);
    }
    return null;
}
```

### 4.2.4 load() vs find() 的源码差异

```java
// find() -- 立即加载
// loadType = LoadType.INTERNAL_LOAD_EAGER

// load() -- 延迟加载（返回代理）
public <T> T load(Class<T> entityClass, Serializable id) {
    return byId(entityClass).getReference(id);
}

// DefaultLoadEventListener.proxyOrLoad()
private Object proxyOrLoad(LoadEvent event, EntityPersister persister, 
                            EntityKey keyToLoad, LoadType options) {
    if (isLoadOfLazyEntity) {
        // 创建代理对象（不立即执行 SQL）
        return persister.createProxy(event.getEntityId(), event.getSession());
    } else {
        // 立即执行 SQL 加载
        return loadFromDatasource(event, persister, keyToLoad, options);
    }
}
```

**关键差异总结：**

| 方法 | 返回类型 | 找不到时 | 是否创建代理 | LoadType |
|------|---------|---------|-------------|----------|
| `find()` | 实体实例或 null | 返回 null | 否 | `INTERNAL_LOAD_EAGER` |
| `load()` | 代理对象 | 访问时抛异常 | 是 | `INTERNAL_LOAD_LAZY` |

## 4.3 merge() 流程

### 4.3.1 SessionImpl.merge() 入口

```java
// SessionImpl.merge()
public <T> T merge(T object) {
    return fireMerge(new MergeEvent(null, object, this));
}

// DefaultMergeEventListener.onMerge()
public void onMerge(MergeEvent event) {
    EntityPersister persister = getPersister(event);
    Serializable id = persister.getIdentifier(event.getEntity(), session);
    
    Object target = null;  // 托管的目标实体
    
    if (id != null) {
        // 1.1 在 PersistenceContext 中查找
        EntityKey key = session.generateEntityKey(id, persister);
        target = session.getPersistenceContext().getEntity(key);
        
        if (target == null) {
            // 1.2 在二级缓存中查找
            if (persister.hasCache()) {
                target = persister.getCacheAccessStrategy().get(session, key, ...);
            }
        }
    }
    
    if (target == null) {
        // 1.3 从数据库加载
        target = session.get(entityName, id);
        
        if (target == null) {
            // 1.4 数据库中也不存在 -> 创建新实体（相当于 persist）
            target = instantiate(persister, id);
            session.persist(target);
            return;
        }
    }
    
    // 阶段 2: 属性拷贝（detached -> managed）
    copyValues(persister, event.getEntity(), target, session, mergeContext);
    
    // 阶段 3: 返回托管实体
    event.setResult(target);
}
```

### 4.3.2 属性拷贝机制

```java
// AbstractEntityPersister.copyValues()
public void copyValues(Object source, Object target, 
    SharedSessionContractImplementor session, CopyContext copyContext) {
    Object[] sourceValues = getPropertyValues(source);
    Object[] targetValues = getPropertyValues(target);
    Type[] types = getPropertyTypes();
    
    for (int i = 0; i < types.length; i++) {
        if (types[i].isEntityType()) {
            // 级联合并关联实体
            targetValues[i] = types[i].replace(sourceValues[i], targetValues[i], 
                session, target, copyContext);
        } else if (types[i].isCollectionType()) {
            // 集合的合并
            targetValues[i] = types[i].replace(sourceValues[i], targetValues[i], 
                session, target, copyContext);
        } else {
            // 普通属性直接覆盖
            targetValues[i] = sourceValues[i];
        }
    }
    
    setPropertyValues(target, targetValues);
}
```

## 4.4 remove() 流程

### 4.4.1 SessionImpl.remove() 入口

```java
// SessionImpl.remove()
public void remove(Object object) {
    fireDelete(new DeleteEvent(object, entityName, false, this));
}

// SessionImpl.internalRemove()
public void internalRemove(Object entity) {
    EntityPersister persister = getEntityPersister(null, entity);
    EntityEntry entry = persistenceContext.getEntry(entity);
    
    // 1. 检查实体状态
    if (entry == null) {
        throw new IllegalArgumentException("attempt to delete a detached instance");
    }
    
    // 2. 标记实体为 DELETED
    entry.setStatus(Status.DELETED);
    
    // 3. 级联删除关联实体（cascade = REMOVE / ALL）
    cascadeOnDelete(entity, persister);
    
    // 4. 将 EntityDeleteAction 加入 ActionQueue
    actionQueue.addAction(
        new EntityDeleteAction(id, state, version, entity, persister, session)
    );
}
```

### 4.4.2 EntityDeleteAction.execute()

```java
public void execute() {
    // 1. 生成 DELETE SQL
    String deleteSQL = persister.generateDeleteString();
    // 如: DELETE FROM T_USER WHERE id = ? AND version = ?
    
    // 2. 绑定参数（ID 和 version 用于乐观锁）
    persister.getIdentifierType().nullSafeSet(st, id, 1, session);
    if (persister.isVersioned()) {
        persister.getVersionType().nullSafeSet(st, version, 2, session);
    }
    
    // 3. 执行删除
    st.executeUpdate();
    
    // 4. 从 PersistenceContext 移除实体
    session.getPersistenceContext().removeEntity(entityKey);
    
    // 5. 注册事务完成后的处理
    session.getActionQueue().registerProcess(new AfterTransactionCompletionProcess() {
        public void doAfterTransactionCompletion(boolean success, ...) {
            if (success) {
                persister.cacheAfterDelete(entity, session);
            }
        }
    });
}
```

### 4.4.3 级联删除处理

```java
private void cascadeBeforeDelete(EventSource session, EntityPersister persister, 
                                  Object entity, EntityEntry entry) {
    for (Type type : persister.getPropertyTypes()) {
        if (type.isEntityType()) {
            if (mapping.hasCascadeDelete()) {
                Object child = persister.getPropertyValue(entity, index);
                if (child != null) {
                    session.remove(child);
                }
            }
        } else if (type.isCollectionType()) {
            Collection collection = persister.getCollection(index);
            if (collection.isOrphanDelete()) {
                // orphanRemoval = true -> 产生 OrphanRemovalAction
            }
            if (collection.hasCascadeDelete()) {
                Object children = persister.getPropertyValue(entity, index);
                for (Object child : (Iterable) children) {
                    session.remove(child);
                }
            }
        }
    }
}
```

---

# 5. Flush 与脏检查源码分析

## 5.1 Flush 入口

### 5.1.1 SessionImpl.flush() -> SessionImpl.doFlush()

```java
// SessionImpl.flush()
public void flush() {
    checkOpen();
    if (transactionCoordinator != null) {
        transactionCoordinator.pulse();  // 检查事务状态
    }
    doFlush();
}

protected void doFlush() {
    fireFlush(new FlushEvent(this));
}
```

### 5.1.2 DefaultFlushEventListener.onFlush()

```java
public void onFlush(FlushEvent event) {
    EventSource source = event.getSession();
    PersistenceContext context = source.getPersistenceContext();
    
    // 1. 准备操作队列（排序 + 外键依赖解析）
    source.getActionQueue().prepareActions();
    
    // 2. 遍历所有受管实体，执行脏检查
    flushEntities(event);
    
    // 3. 遍历所有受管集合，检查集合变更
    flushCollections(source);
    
    // 4. 执行操作队列中的所有 Action
    source.getActionQueue().executeActions();
    
    // 5. 后处理（更新快照等）
    postFlush(source);
}
```

## 5.2 脏检查实现

### 5.2.1 DefaultFlushEntityEventListener.onFlushEntity()

```java
public void onFlushEntity(FlushEntityEvent event) {
    EntityEntry entry = event.getEntityEntry();
    EntityPersister persister = entry.getPersister();
    Object entity = event.getEntity();
    
    // 1. 获取实体当前所有属性值
    Object[] currentState = persister.getPropertyValues(entity);
    
    // 2. 获取加载时的原始快照
    Object[] loadedState = entry.getLoadedState();
    
    // 3. 调用 findDirty() 执行比较
    int[] dirtyProperties = persister.findDirty(
        currentState, loadedState, entity, event.getSession()
    );
    
    if (dirtyProperties != null && dirtyProperties.length > 0) {
        // 4. 有脏数据，创建 EntityUpdateAction
        EntityUpdateAction action = new EntityUpdateAction(
            entry.getId(), currentState, dirtyProperties,
            entry.getVersion(), entity, entry.getRowId(),
            persister, event.getSession()
        );
        event.getSession().getActionQueue().addAction(action);
        
        // 5. 更新快照（以备下次脏检查使用）
        entry.postUpdate(entity, currentState, persister.getVersion(entity));
    }
}
```

### 5.2.2 AbstractEntityPersister.findDirty()

```java
// 默认策略: 比较当前所有属性值与快照中的对应值
public int[] findDirty(Object[] currentState, Object[] previousState, 
                        Object entity, SharedSessionContractImplementor session) {
    int[] dirtyProps = null;
    int count = 0;
    Type[] types = getPropertyTypes();
    
    for (int i = 0; i < types.length; i++) {
        // 跳过不可更新的属性（如 ID、version）
        if (!types[i].isMutable()) continue;
        
        if (!types[i].isEqual(currentState[i], previousState[i])) {
            if (dirtyProps == null) {
                dirtyProps = new int[types.length];
            }
            dirtyProps[count++] = i;
        }
    }
    
    if (count == 0) return null;
    return Arrays.copyOf(dirtyProps, count);
}

// 增强策略 (Bytecode Enhancement):
// 如果实体使用了字节码增强并开启 dirtyTracking，
// 每个属性 setter 中会自动追踪变化，脏检查只需检查标记位
```

### 5.2.3 快照存储与比较

```java
// EntityEntry 中的快照
public class EntityEntry {
    private Object[] loadedState;
    // 实体加载/刷新时的属性值快照
    // 索引对应 persister.getPropertyTypes() 的顺序
    // 例如: [123, "Alice", 30, addressProxy, photoCollection]
    
    public void postUpdate(Object entity, Object[] updatedState, Object nextVersion) {
        this.loadedState = updatedState;  // 替换为新快照
        this.version = nextVersion;
    }
}

// Type.isEqual() 深度比较
// AbstractStandardBasicType.isEqual():
//   1. 引用相等 -> true
//   2. 任意为 null -> false
//   3. 调用 x.equals(y)
//
// EntityType.isEqual() -- 只比较 ID，不比较整个实体
//   比较双方的 identifier 值是否相等
```

## 5.3 ActionQueue 排序

### 5.3.1 ActionQueue.prepareActions()

```java
public void prepareActions() {
    prepareActions(insertions);
    prepareActions(updates);
    prepareActions(deletions);
    prepareActions(collectionCreations);
    prepareActions(collectionUpdates);
    prepareActions(collectionRemovals);
}

private void prepareActions(ExecutableList<?> queue) {
    // 按外键依赖排序
    Collections.sort(queue, new ActionSorter());
}
```

### 5.3.2 操作类型执行优先级

```java
// ActionQueue.executeActions() -- 按固定顺序执行
public void executeActions() {
    // 1. OrphanRemovalAction（孤立删除）
    //    必须先执行，因为后续 INSERT/UPDATE 可能依赖这些清理
    executeActions(orphanRemovals);
    
    // 2. EntityInsertAction（实体插入）
    //    在 UPDATE/DELETE 之前，确保新实体存在
    executeActions(insertions);
    
    // 3. EntityUpdateAction（实体更新）
    //    插入完成后才能更新
    executeActions(updates);
    
    // 4. CollectionRecreateAction（集合重建）
    executeActions(collectionCreations);
    
    // 5. CollectionRemoveAction（集合删除）
    executeActions(collectionRemovals);
    
    // 6. CollectionUpdateAction（集合更新）
    executeActions(collectionUpdates);
    
    // 7. EntityDeleteAction（实体删除）
    //    最后删除，因为其他操作可能还在引用
    executeActions(deletions);
}
```

### 5.3.3 外键依赖排序算法

```java
// ActionSorter 按外键依赖排序
// 示例: OrderItem 的外键指向 Order 的主键
//  -> Order 的插入必须在 OrderItem 之前

public class ActionSorter implements Comparator<Executable> {
    public int compare(Executable o1, Executable o2) {
        // 检查 o1 是否依赖于 o2
        if (dependsOn(o1, o2)) return 1;   // o1 排在 o2 之后
        // 检查 o2 是否依赖于 o1
        if (dependsOn(o2, o1)) return -1;  // o2 排在 o1 之后
        return 0;
    }
}
```

## 5.4 JDBC Batch 处理

### 5.4.1 BatchBuilder 构建

```java
// BatchBuilder -- 为每次 flush 创建批处理对象
public interface BatchBuilder {
    // 若配置了 jdbcBatchSize，则创建 BatchingBatch
    // 否则创建 NonBatchingBatch
    Batch buildBatch(BatchKey key, JdbcCoordinator jdbcCoordinator);
}

// BatchKey 用于区分不同类型的批处理:
//   BatchKey(key = INSERT INTO T_USER ..., type = INSERT)
//   BatchKey(key = UPDATE T_USER SET ..., type = UPDATE)
//   相同 BatchKey 的 SQL 才会被合并到一个 Batch 中
```

### 5.4.2 Batch 实现

```java
// BatchingBatch -- 支持批处理的实现
public class BatchingBatch implements Batch {
    private final int batchSize;           // hibernate.jdbc.batch_size
    private final PreparedStatement statement;
    private int currentBatchCount = 0;
    
    public void addToBatch(String sql, Object[] parameters) {
        for (int i = 0; i < parameters.length; i++) {
            parameterBinders[i].bind(statement, parameters[i], i + 1);
        }
        statement.addBatch();
        currentBatchCount++;
        
        if (currentBatchCount >= batchSize) {
            statement.executeBatch();
            currentBatchCount = 0;
        }
    }
    
    public void execute() {
        if (currentBatchCount > 0) {
            statement.executeBatch();  // 执行剩余 SQL
        }
    }
}

// NonBatchingBatch -- 逐条执行
public class NonBatchingBatch implements Batch {
    public void addToBatch(String sql, Object[] parameters) {
        // 每次 addToBatch 立即执行
        PreparedStatement ps = jdbcCoordinator.getStatementPreparer()
            .prepareStatement(sql);
        // ...绑定参数...
        ps.executeUpdate();
    }
}
```

### 5.4.3 批处理注意事项

- 批处理生效条件：`hibernate.jdbc.batch_size > 0`，且相邻操作 SQL 完全相同
- IDENTITY 主键生成策略下，INSERT 批处理被禁用（MySQL 的 getGeneratedKeys() 只返回最后一条的 ID）
- 批处理大小建议：20-50，过大可能因参数缓存占用过多内存

---

# 6. 查询执行源码分析

## 6.1 HQL/JPQL 查询流程

### 6.1.1 从 HQL 到 SQL 的完整链路

```java
// 用户代码
Query<User> query = session.createQuery(
    "from User u where u.age > :minAge", User.class
);
query.setParameter("minAge", 18);
List<User> results = query.list();

// 内部调用链:
// 1. SessionImpl.createQuery() -> AbstractProducedQuery
// 2. Query.list() -> getQueryPlan() -> performList() -> processResultSet()
```

### 6.1.2 HQL 解析（ANTLR）

```java
// Hibernate 5 使用 ANTLR 进行 HQL 语法解析
public class QueryTranslatorImpl implements QueryTranslator {
    public void compile(Map replacements, boolean scalar) {
        // 步骤 1: 词法+语法分析
        HqlParser parser = HqlParser.getInstance(hql);
        parser.statement();  // 生成 HQL AST
        AST hqlAst = parser.getAST();
        
        // 步骤 2: 语义分析
        //   解析实体名 -> EntityPersister
        //   解析属性名 -> 属性类型
        //   解析关联路径 -> EntityType / CollectionType
        HqlSqlWalker walker = new HqlSqlWalker(hqlAst, factory, ...);
        walker.statement();
        
        // 步骤 3: HQL AST -> SQL AST
        // 步骤 4: SQL AST -> SQL 字符串
        SqlGenerator sqlGen = new SqlGenerator(factory, dialect);
        sqlGen.statement(walker.getAST());
        this.sqlString = sqlGen.getSQL();
    }
}
```

```
HQL AST 示例:
HQL: from User u where u.age > :minAge order by u.name

HQL AST 结构（简化）:
QUERY
  +-- SELECT_FROM
      +-- FROM
      |   +-- RANGE (alias=u, class=User)
      +-- SELECT
          +-- IDENT (u)
  +-- WHERE
  |   +-- GT
  |       +-- DOT (u, age)
  |       +-- PARAM (minAge)
  +-- ORDER_BY
      +-- DOT (u, name)
```

### 6.1.3 SQL 生成（SqlGenerator）

```java
// SqlGenerator 遍历 SQL AST 生成最终 SQL 字符串
public class SqlGenerator {
    // 1. 表名映射（实体名 -> 数据库表名）
    //    "User" -> "T_USER"
    
    // 2. 列名映射（属性名 -> 数据库列名）
    //    "age" -> "AGE_COLUMN"
    
    // 3. 关联 JOIN 生成
    //    "u.address.city" ->
    //    "LEFT JOIN T_ADDRESS a ON u.id = a.user_id
    //     LEFT JOIN T_CITY c ON a.city_id = c.id"
    
    // 4. 鉴别器列处理（继承映射）
    //    自动添加 "AND u.DTYPE = 'Manager'"
    
    // 5. 分页方言适配
    //    MySQL: "LIMIT ? OFFSET ?"
    //    Oracle: "SELECT * FROM (SELECT rownum rn, t.* FROM (...) t
    //             WHERE rownum <= ?) WHERE rn > ?"
}
```

## 6.2 SQM（Semantic Query Model）-- Hibernate 6+

### 6.2.1 SQM 概述

Hibernate 6 用全新的 SQM 架构替换了旧的 ANTLR 解析器。

```java
// 旧架构: HQL -> ANTLR -> HQL AST -> SQL AST -> SQL 字符串
// 新架构: HQL -> SQM 解析 -> SQM 语义树 -> SQM -> SQL (SqmTranslator)

// 关键类:
//   SemanticQueryInterpreter -- SQM 语义查询解释器
//   SqmTranslator -- SQM -> SQL 转换器
//   org.hibernate.query.sqm.tree.SqmStatement -- SQM 语句根节点
//   org.hibernate.query.sqm.tree.select.SqmSelectStatement -- SELECT 语句
//   org.hibernate.query.sqm.tree.from.SqmRoot -- FROM 子句根
//   org.hibernate.query.sqm.tree.predicate.SqmPredicate -- WHERE 条件
```

### 6.2.2 SemanticQueryInterpreter

```java
public class SemanticQueryInterpreter {
    public static <R> SqmStatement<R> interpret(
        String hql,
        Class<R> resultType,
        SessionFactoryImplementor factory
    ) {
        // 1. 使用 HQL 的 ANTLR4 语法解析器
        HqlParseTreeBuilder builder = new HqlParseTreeBuilder(hql);
        HqlParseTree parseTree = builder.build();
        
        // 2. 遍历解析树，构建 SqmStatement
        SqmStatementBuilder<R> sqmBuilder = new SqmStatementBuilder<>(
            factory.getMetamodel(), resultType
        );
        return sqmBuilder.visitStatement(parseTree);
    }
}
```

### 6.2.3 SqmTranslator（SQM -> SQL）

```java
public class SqmTranslator<T> {
    public JdbcSelect translate(SqmSelectStatement<T> sqmStatement) {
        // 步骤 1: 确定 SELECT 列
        // 步骤 2: 生成 FROM 子句（遍历 SqmRoot + SqmJoin）
        // 步骤 3: 生成 WHERE 子句（SqmPredicate -> SQL 条件）
        // 步骤 4: 生成 GROUP BY / HAVING / ORDER BY
        // 步骤 5: 应用分页方言
        // 步骤 6: 封装为 JdbcSelect 对象
        
        return new JdbcSelect(
            sqlString, parameterBinders, 
            jdbcParameterExtractors, resultSetMapping
        );
    }
}
```

### 6.2.4 查询优化

```java
// SQM 层的查询优化包括:
// 1. 常量折叠 -- 编译期计算常量表达式
//    WHERE u.age > 10 + 5 -> WHERE u.age > 15

// 2. 冗余 JOIN 消除
//    如果 SELECT 和 WHERE 中未引用 JOIN 的表，则移除该 JOIN

// 3. 子查询展开
//    FROM (SELECT ...) sub -> 尝试合并到外层查询

// 4. Fetch 策略优化
//    根据 @FetchProfile 和实体元数据决定是否需额外 JOIN
```

### 6.2.5 SQM 的优点

| 对比维度 | 旧架构 (ANTLR2/3) | SQM (ANTLR4) |
|----------|-------------------|--------------|
| 解析器 | 手写语法 + ANTLR2/3 | ANTLR4 自动生成 |
| 语法支持 | HQL 历史遗留语法 | 完全 JPA 2.2+ Criteria 兼容 |
| 扩展性 | 自定义方言困难 | 清晰的 Visitor 模式扩展点 |
| 语义分析 | 混在 AST 遍历中 | 独立的 SQM 语义树 |
| SQL 生成 | 单步 SQL AST -> SQL | SQM -> SQL 两步转换 |
| 窗口函数 | 不支持 | 支持 (ROW_NUMBER, RANK) |
| CTE | 不支持 | 支持 (WITH 子句) |

## 6.3 查询计划缓存

### 6.3.1 QueryPlanCache 结构

```java
public class QueryPlanCache {
    private final BoundedConcurrentHashMap<HQLQueryPlanKey, HQLQueryPlan> hqlPlanCache;
    private final BoundedConcurrentHashMap<NativeSQLQueryPlanKey, NativeSQLQueryPlan> nativePlanCache;
    private final BoundedConcurrentHashMap<String, FilterQueryPlan> filterPlanCache;
    
    // 缓存最大条目数: hibernate.query.plan_cache_max_size (默认 2048)
    private final int maxQueryPlanCount;
}
```

### 6.3.2 缓存 Key 的构造

```java
// HQLQueryPlanKey 的组成
public class HQLQueryPlanKey {
    private final String hql;                    // HQL 查询字符串
    private final Type[] positionalParameterTypes;
    private final Map<String, Type> namedParameterTypes;
    private final Integer firstRow;              // 分页起始行
    private final Integer maxRows;               // 最大返回行数
    private final boolean cacheable;
    private final String resultingClassName;
    
    // 特别注意: 参数数量会影响 hash！
    // 例如: IN (:ids) 中不同长度的集合产生不同的 Key
    // 这是查询计划缓存膨胀的主要原因之一
}
```

### 6.3.3 缓存大小限制与风险

```java
// 配置: hibernate.query.plan_cache_max_size (默认 2048)
// 配置: hibernate.query.plan_parameter_metadata_max_size (默认 128)

// 风险场景 1: IN 子句参数数量变化
//   每次不同长度的 IN 列表产生独立查询计划，缓存快速膨胀

// 风险场景 2: 动态拼接 HQL
//   每个不同的 HQL 字符串产生不同的查询计划

// 解决方案:
//   - 使用 Criteria API 替代字符串拼接
//   - 对 IN 子句使用固定大小的分段处理
//   - 适当调大 plan_cache_max_size
```

## 6.4 结果集处理

### 6.4.1 Loader.processResultSet()

```java
public List processResultSet(
    ResultSet resultSet,
    QueryParameters queryParameters,
    SharedSessionContractImplementor session,
    boolean returnProxies,
    ResultTransformer forcedResultTransformer,
    int maxRows,
    List<AfterLoadAction> afterLoadActions
) {
    List results = new ArrayList();
    ResultSetProcessor processor = new ResultSetProcessor(resultSet, session, getReturnTypes());
    
    while (resultSet.next()) {
        Object result = processor.extractRow(
            resultSet, session, queryParameters, returnProxies, afterLoadActions
        );
        results.add(result);
        
        if (maxRows > 0 && results.size() >= maxRows) break;
    }
    return results;
}
```

### 6.4.2 EntityPersister.hydrate()

```java
// hydrate() -- 将 ResultSet 拆解为属性值数组 (Object[])
// 此阶段不创建实体对象，只是提取原始数据

public Object[] hydrate(
    ResultSet rs, Serializable id, Object object,
    Loadable rootLoadable, String[][] suffixedPropertyColumns,
    boolean allProperties, SharedSessionContractImplementor session
) {
    Type[] types = getPropertyTypes();
    Object[] values = new Object[types.length];
    
    for (int i = 0; i < types.length; i++) {
        // 找到此属性在 ResultSet 中的列索引
        // 提取列值: IntegerType -> rs.getInt(), StringType -> rs.getString()
        values[i] = types[i].hydrate(rs, columnAliases, session, object);
    }
    return values;
}
```

### 6.4.3 EntityPersister.resolve()

```java
// resolve() -- 将 hydrate 阶段提取的属性值数组组装成完整的实体对象
public Object resolve(Object id, ResultSet rs, Object owner,
                       SharedSessionContractImplementor session) {
    // 1. 通过无参构造函数创建实体实例
    Object entity = getEntityTuplizer().instantiate();
    
    // 2. 分解属性值（解析关联代理、集合初始化器）
    Object[] resolvedValues = resolveValues(id, rawValues, session);
    
    // 3. 设置所有属性值到实体实例
    setPropertyValues(entity, resolvedValues);
    
    // 4. 注册到 PersistenceContext
    session.getPersistenceContext().addEntity(
        generateEntityKey(id, this), entity, Status.LOADING
    );
    
    return entity;
}
```

### 6.4.4 TwoPhaseLoad.initializeEntity()

```java
// TwoPhaseLoad.initializeEntity() -- 实体两阶段加载
// 第一阶段: hydrate -- 从 ResultSet 提取属性值
// 第二阶段: resolve + initialize -- 组装实体并注入关联

public static void initializeEntity(
    Object entity, boolean readOnly,
    SharedSessionContractImplementor session, PreLoadEvent preLoadEvent
) {
    // 1. 创建并关联 EntityEntry
    EntityEntry entry = session.getPersistenceContext().addEntry(
        entity, Status.LOADING, loadedState, rowId, id, version, ...);
    
    // 2. 处理关联属性（EntityType / CollectionType）
    //    - 创建代理对象（延迟加载）
    //    - 创建 PersistentCollection 包装
    
    // 3. 注册实体到 PersistenceContext
    //    entityByKey.put(entityKey, entity)
    //    entitySnapshotsByKey.put(entityKey, loadedState)
    
    // 4. 调用 @PostLoad 回调
}
```

---

# 7. 懒加载与代理源码分析

## 7.1 代理生成

### 7.1.1 ProxyFactory 的选择

```java
// ProxyFactory 的选择依据（Hibernate 5）
// 1. 检查配置: hibernate.bytecode.provider
//    - "bytebuddy" (默认, 5.3+) -> ByteBuddyProxyFactory
//    - "javassist" -> JavassistProxyFactory

// 2. 检查类路径中是否有对应库
//    ByteBuddy: net.bytebuddy.ByteBuddy
//    Javassist: javassist.util.proxy.ProxyFactory

// EntityPersister 在构造时选择 ProxyFactory
if (bytecodeProvider instanceof ByteBuddyProvider) {
    this.proxyFactory = new ByteBuddyProxyFactory(...);
} else {
    this.proxyFactory = new JavassistProxyFactory(...);
}
```

### 7.1.2 ByteBuddyProxyFactory 详解

```java
public class ByteBuddyProxyFactory implements ProxyFactory {
    // 缓存的代理类（每个实体类只生成一次）
    private final ConcurrentHashMap<Class<?>, Class<?>> proxyClassCache;
    
    public Object getProxy(Serializable id, SharedSessionContractImplementor session) {
        // 1. 获取或创建代理类
        Class<?> proxyClass = getProxyClass(session.getFactory(), entityClass);
        
        // 2. 创建 LazyInitializer
        LazyInitializer li = new LazyInitializer(entityName, id, session);
        
        // 3. 实例化代理对象
        return proxyClass.getConstructor(LazyInitializer.class).newInstance(li);
    }
    
    private Class<?> getProxyClass(SessionFactoryImplementor factory, Class<?> entityClass) {
        return proxyClassCache.computeIfAbsent(entityClass, clazz -> {
            return new ByteBuddy()
                .subclass(clazz)
                .implement(HibernateProxy.class)
                // 添加 LazyInitializer 字段
                .defineField("$$_hibernate_lazy_initializer", LazyInitializer.class, ...)
                // 实现 getHibernateLazyInitializer
                .method(named("getHibernateLazyInitializer"))
                .intercept(FieldAccessor.ofField("$$_hibernate_lazy_initializer"))
                // 拦截所有其他 public 方法
                .method(isPublic().and(not(named("finalize"))))
                .intercept(MethodDelegation.to(ByteBuddyInterceptor.class))
                .make()
                .load(factory.getServiceRegistry().getService(ClassLoaderService.class)
                    .getClassLoader())
                .getLoaded();
        });
    }
}
```

### 7.1.3 代理类的结构

以 `User` 实体类为例，生成的代理类结构如下：

```java
// 生成的代理类（伪代码）
public class User$HibernateProxy$abc123 
    extends User 
    implements HibernateProxy {

    private LazyInitializer $$_hibernate_lazy_initializer;
    
    public User$HibernateProxy$abc123(LazyInitializer li) {
        this.$$_hibernate_lazy_initializer = li;
    }
    
    public LazyInitializer getHibernateLazyInitializer() {
        return $$_hibernate_lazy_initializer;
    }
    
    public Object writeReplace() {
        return new HibernateProxySerializer(this);
    }
    
    // 所有业务方法被拦截
    public String getName() {
        return (String) ByteBuddyInterceptor.intercept(
            this, "getName", new Object[0], $$_hibernate_lazy_initializer
        );
    }
}
```

### 7.1.4 代理的序列化处理

```java
// 当代理对象被序列化时:
// 1. writeReplace() 被调用，返回 HibernateProxySerializer
// 2. 序列化 LazyInitializer 信息（entityName + id）
// 3. 反序列化时 readResolve() 通过当前线程的 Session 重新创建代理

public class HibernateProxySerializer implements Serializable {
    private String entityName;
    private Serializable id;
    
    public HibernateProxySerializer(HibernateProxy proxy) {
        LazyInitializer li = proxy.getHibernateLazyInitializer();
        this.entityName = li.getEntityName();
        this.id = li.getIdentifier();
    }
    
    private Object readResolve() {
        SessionImplementor session = getCurrentSessionFromJtaOrContext();
        if (session != null) {
            return session.internalLoad(entityName, id, false, false);
        }
        return this; // 无 Session 时返回包装对象
    }
}
```

## 7.2 懒加载触发

### 7.2.1 代理拦截器

```java
// ByteBuddyInterceptor
public class ByteBuddyInterceptor implements MethodHandler {
    
    public static Object intercept(
        Object proxy, String methodName, Object[] args,
        LazyInitializer lazyInitializer
    ) throws Throwable {
        
        // 1. 特殊方法直接处理（不触发初始化）
        if ("getHibernateLazyInitializer".equals(methodName)) {
            return lazyInitializer;
        }
        if ("writeReplace".equals(methodName)) {
            return new HibernateProxySerializer((HibernateProxy) proxy);
        }
        if ("finalize".equals(methodName)) return null;
        
        Method method = findMethodOnTargetClass(proxy.getClass(), methodName, args);
        
        // 2. 检查是否需要触发初始化
        if (lazyInitializer.isUninitialized()) {
            // 如果方法来自 Object 类（hashCode, equals, toString）
            if (method.getDeclaringClass() == Object.class) {
                // 不触发初始化，使用代理自身的行为
                // hashCode -> 基于 entityName + id 计算
                // equals -> 比较 entityName + id
                // toString -> "Proxy[entityName#id]"
                return handleObjectMethod(proxy, method, args, lazyInitializer);
            }
            
            // 方法来自实体类 -> 触发懒加载
            lazyInitializer.initialize();
        }
        
        // 3. 调用真实对象的方法
        Object target = lazyInitializer.getImplementation();
        return method.invoke(target, args);
    }
}
```

### 7.2.2 LazyInitializer.initialize() 详细流程

```java
public class LazyInitializer {
    private String entityName;
    private Serializable id;
    private SharedSessionContractImplementor session;
    private Object target;           // 真实的实体实例
    private boolean initialized;
    
    public final void initialize() {
        if (initialized) return;
        
        if (session == null) {
            throw new LazyInitializationException(
                "could not initialize proxy - no Session");
        }
        if (!session.isOpen()) {
            throw new LazyInitializationException(
                "could not initialize proxy - the owning Session was closed");
        }
        
        // 获取实体持久化器
        EntityPersister persister = session.getFactory()
            .getMetamodel().entityPersister(entityName);
        
        // 加载实体
        target = persister.load(id, null, LockOptions.NONE, session);
        
        initialized = true;
    }
    
    public final Object getImplementation() {
        if (!initialized) initialize();
        return target;
    }
}
```

### 7.2.3 初始化后的行为

```java
// 一旦初始化完成:
// 1. initialized 标记为 true
// 2. 后续方法调用直接转发到 target 对象
// 3. 代理的拦截器检查 initialized == true -> 跳过初始化步骤

// 判断对象是否是未初始化的代理:
Hibernate.isInitialized(proxy);        // 返回 false
Hibernate.unproxy(proxy);              // 返回真实对象（触发加载）
```

## 7.3 PersistentCollection 懒加载

### 7.3.1 AbstractPersistentCollection 核心

```java
public abstract class AbstractPersistentCollection 
    implements Serializable, PersistentCollection {
    
    protected transient SharedSessionContractImplementor session;
    protected Object owner;
    protected boolean initialized;              // 是否已从数据库加载
    protected boolean dirty;                    // 是否被修改过
    protected transient List<Object> storedSnapshot;
    protected Serializable key;
    protected CollectionPersister persister;
    
    // 所有读操作必须调用此方法
    protected final void read() {
        initialize(false);
    }
    
    // 所有写操作必须调用此方法
    protected final void write() {
        initialize(true);
        dirty = true;
    }
    
    // 从数据库加载集合元素
    public final void forceInitialization() {
        if (!initialized) {
            if (session == null) {
                throw new LazyInitializationException(
                    "failed to lazily initialize a collection, no session");
            }
            session.initializeCollection(this, false);
        }
    }
}
```

### 7.3.2 集合访问触发流程

```java
// PersistentSet 示例
public class PersistentSet extends AbstractPersistentCollection implements java.util.Set {
    private Set set;
    
    // 读操作（触发加载）
    public int size() { read(); return set.size(); }
    public boolean isEmpty() { read(); return set.isEmpty(); }
    public boolean contains(Object o) { read(); return set.contains(o); }
    public Iterator iterator() {
        read();
        return new IteratorProxy(set.iterator());
    }
    public Object[] toArray() { read(); return set.toArray(); }
    
    // 写操作（标记 dirty）
    public boolean add(Object value) { write(); return set.add(value); }
    public boolean remove(Object value) { write(); return set.remove(value); }
    public boolean addAll(Collection c) { write(); return set.addAll(c); }
    public void clear() { write(); set.clear(); }
}
```

### 7.3.3 集合加载与 SQL 生成

```java
public void initializeCollection(PersistentCollection collection, boolean writing) {
    CollectionPersister persister = collection.getPersister();
    Serializable key = collection.getKey();
    
    // 生成加载 SQL:
    // 一对一关系: SELECT * FROM child_table WHERE id = ?
    // 一对多关系: SELECT * FROM child_table WHERE parent_id = ?
    // 多对多关系: SELECT ... FROM join_table jt 
    //             JOIN child_table ct ON jt.child_id = ct.id 
    //             WHERE jt.parent_id = ?
    
    Loader loader = new CollectionLoader(persister, factory, ...);
    List elements = loader.loadCollection(session, key, writing);
    collection.injectLoadedState(elements);
}
```

---

# 8. 类型系统源码分析

## 8.1 Type 接口层次（Hibernate 5）

### 8.1.1 类型继承体系

```java
// ===== 顶层接口 =====
public interface Type extends Serializable {
    boolean isAssociationType();     // 是否为关联类型
    boolean isCollectionType();
    boolean isEntityType();
    boolean isComponentType();
    boolean isMutable();             // 属性值是否可变
    
    // 值比较
    boolean isEqual(Object x, Object y);
    int getHashCode(Object x);
    
    // JDBC 操作
    int[] sqlTypes(Mapping mapping);
    int getColumnSpan(Mapping mapping);
}

// ===== 可空类型 =====
public abstract class NullableType extends AbstractType {
    public abstract Object nullSafeGet(ResultSet rs, String[] names, 
        SharedSessionContractImplementor session, Object owner);
    public abstract void nullSafeSet(PreparedStatement st, Object value, 
        int index, SharedSessionContractImplementor session);
}

// ===== 具体基本类型 =====
public class StringType extends AbstractStandardBasicType<String>    { /* String -> VARCHAR */ }
public class IntegerType extends AbstractStandardBasicType<Integer>  { /* Integer -> INTEGER */ }
public class LongType extends AbstractStandardBasicType<Long>        { /* Long -> BIGINT */ }
public class BooleanType extends AbstractStandardBasicType<Boolean>  { /* Boolean -> BOOLEAN */ }
public class DateType extends AbstractStandardBasicType<Date>        { /* Date -> TIMESTAMP */ }
public class BigDecimalType extends AbstractStandardBasicType<BigDecimal> { /* -> NUMERIC */ }
public class ByteArrayType extends AbstractStandardBasicType<byte[]> { /* -> BLOB/VARBINARY */ }
public class UUIDBinaryType extends AbstractStandardBasicType<UUID>  { /* UUID -> BINARY */ }
public class UUIDCharType extends AbstractStandardBasicType<UUID>    { /* UUID -> CHAR(36) */ }

// ===== 关联类型 =====
// EntityType -- 实体引用（@ManyToOne / @OneToOne）
public class EntityType extends AbstractType implements AssociationType {
    private final String associatedEntityName;
    private final EntityPersister persister;
}

// CollectionType -- 集合关联（@OneToMany / @ManyToMany）
public class CollectionType extends AbstractType implements AssociationType {
    private final String role;
    private final CollectionPersister persister;
}

// ComponentType -- 嵌入式组件（@Embeddable）
public class ComponentType extends AbstractType implements CompositeType {
    private final String[] propertyNames;
    private final Type[] propertyTypes;
    private final boolean[] propertyNullability;
}
```

### 8.1.2 TypeResolver 解析流程

```java
public class TypeResolver {
    public Type heuristicType(String typeName) {
        // 1. 先查基本类型映射表
        //    "string" -> StringType, "int" -> IntegerType
        
        // 2. 尝试作为 Java 类加载
        //    java.lang.String -> 由 BasicTypeRegistry 解析
        //    com.example.Address -> ComponentType (若 @Embeddable)
        
        // 3. 用户自定义类型
        //    "myapp.MyCustomType" -> 实例化并返回
    }
}
```

## 8.2 Hibernate 6 新类型系统

### 8.2.1 JavaType + JdbcType 分离设计

```java
// ===== Hibernate 5: 单一体 =====
// Type 同时负责 Java 侧和 JDBC 侧的所有职责

// ===== Hibernate 6: 分离设计 =====

// 1. JavaType<T> -- 描述 Java 侧的类型
public interface JavaType<T> {
    Class<T> getJavaTypeClass();
    boolean areEqual(T one, T another);
    int extractHashCode(T value);
    String toString(T value);
    T fromString(CharSequence string);
    <X> X unwrap(T value, Class<X> type, WrapperOptions options);
    <X> T wrap(X value, WrapperOptions options);
}

// 2. JdbcType -- 描述 JDBC 侧的类型
public interface JdbcType {
    int getJdbcTypeCode();               // java.sql.Types 中的值
    String getFriendlyName();
    <T> T extract(ResultSet rs, int paramIndex, 
                  Class<T> javaType, SharedSessionContractImplementor session);
    <T> void bind(PreparedStatement st, T value, 
                  int paramIndex, SharedSessionContractImplementor session);
}

// 3. BasicType<T> -- 组合 JavaType + JdbcType
public interface BasicType<T> extends Type {
    JavaType<T> getJavaTypeDescriptor();
    JdbcType getJdbcTypeDescriptor();
}

// 4. 具体实现示例
public class StringJavaType implements JavaType<String> {
    public boolean areEqual(String one, String another) {
        return Objects.equals(one, another);
    }
}

public enum VarcharJdbcType implements JdbcType {
    INSTANCE;
    public int getJdbcTypeCode() { return Types.VARCHAR; }
    public <T> T extract(ResultSet rs, int paramIndex, Class<T> javaType, ...) {
        return javaType.cast(rs.getString(paramIndex));
    }
    public <T> void bind(PreparedStatement st, T value, int paramIndex, ...) {
        st.setString(paramIndex, (String) value);
    }
}
```

### 8.2.2 类型映射注册与查找

```java
public class TypeConfiguration {
    // 基本类型注册表
    private final Map<Class<?>, JdbcType> jdbcTypeMapping;
    
    // 类型映射查询
    public BasicType<?> resolveBasicType(Class<?> javaType, JdbcType jdbcType) {
        return basicTypeRegistry.getOrCreate(javaType, jdbcType);
    }
}

// 自动类型映射规则:
//   String   -> VARCHAR(255)
//   int/long -> INTEGER / BIGINT
//   boolean  -> BOOLEAN
//   Date     -> TIMESTAMP
//   UUID     -> BINARY(16) 或 CHAR(36)
//   Enum     -> VARCHAR (ORDINAL 可改为 TINYINT)
//   byte[]   -> VARBINARY / BLOB
```

---

# 9. 缓存源码分析

## 9.1 一级缓存实现

### 9.1.1 数据结构回顾

一级缓存即 `StatefulPersistenceContext`，其核心数据结构：

```java
// 一级缓存的本质
// - 作用域: 单个 Session
// - 存储: 内存 Map 集合
// - 生命周期: Session 打开 -> Session 关闭
// - 强制开启: 无法关闭

// 核心查找逻辑 (按 ID 查找)
public Object getEntity(EntityKey key) {
    // 1. 先查 entityByKey Map
    Object entity = entityByKey.get(key);
    if (entity != null) return entity;
    
    // 2. 检查是否存在代理
    entity = proxiesByKey.get(key);
    if (entity != null && Hibernate.isInitialized(entity)) {
        return entity;
    }
    return null; // 未命中
}
```

### 9.1.2 一级缓存清理时机

```java
// 一级缓存被清理（清空）的时机:
// 1. Session.clear() -- 手动清理
//    persistenceContext.clear()
//      -> entityByKey.clear()
//      -> entitySnapshotsByKey.clear()
//      -> proxiesByKey.clear()
//      -> collectionsByKey.clear()

// 2. Session.evict(entity) -- 逐出一个实体
//    persistenceContext.removeEntity(entityKey)
//      -> entityByKey.remove(key)
//      -> entitySnapshotsByKey.remove(key)

// 3. Session.close() -- 关闭 Session，释放所有缓存

// 注意：flush() 不清理一级缓存！
// flush() 只是将缓存中的变更同步到数据库
```

## 9.2 二级缓存架构

### 9.2.1 RegionFactory -> Region 层次结构

```java
// ===== RegionFactory -- 缓存区域工厂 =====
public interface RegionFactory {
    void start(SessionFactoryOptions settings, Map<String,String> properties);
    void stop();
    
    EntityRegion buildEntityRegion(String regionName, 
        Map<String,String> properties, CacheDataDescription metadata);
    CollectionRegion buildCollectionRegion(String regionName, 
        Map<String,String> properties, CacheDataDescription metadata);
    QueryResultsRegion buildQueryResultsRegion(String regionName, 
        Map<String,String> properties);
    TimestampsRegion buildTimestampsRegion(String regionName, 
        Map<String,String> properties);
}

// ===== EntityRegion -- 实体缓存区域 =====
public interface EntityRegion extends TransactionalDataRegion {
    EntityRegionAccessStrategy getAccessStrategy();
    // 提供 get/put/evict/update 等操作
}

// ===== CollectionRegion -- 集合缓存区域 =====
public interface CollectionRegion extends TransactionalDataRegion {
    CollectionRegionAccessStrategy getAccessStrategy();
}

// ===== QueryResultsRegion -- 查询结果缓存 =====
public interface QueryResultsRegion extends Region {
    Object get(Object key);
    void put(Object key, Object value);
    void evict(Object key);
}

// ===== TimestampsRegion -- 时间戳缓存区域 =====
public interface TimestampsRegion extends Region {
    Object get(Object key);
    void put(Object key, Object value);
}
```

### 9.2.2 CacheKey 的构造

```java
public class CacheKey implements Serializable {
    private final Serializable id;           // 实体 ID
    private final String entityOrRoleName;   // 实体名或集合角色名
    private final String tenantIdentifier;    // 多租户标识
    private final int hashCode;
    
    public CacheKey(Serializable id, String entityOrRoleName, String tenantIdentifier) {
        this.id = id;
        this.entityOrRoleName = entityOrRoleName;
        this.tenantIdentifier = tenantIdentifier;
        this.hashCode = calculateHashCode();
    }
}

// CacheKey 由 CacheKeysFactory 创建:
public interface CacheKeysFactory {
    CacheKey createEntityKey(Object id, EntityPersister persister);
    CacheKey createCollectionKey(Object id, CollectionPersister persister);
}
```

### 9.2.3 缓存写入时机

```java
// 二级缓存的写入发生在事务成功提交后的 afterCompletion 阶段

// EntityInsertAction 中注册的 AfterTransactionCompletionProcess:
new EntityInsertAction(...) {
    public void doAfterTransactionCompletion(boolean success, SessionImplementor session) {
        if (success) {
            EntityPersister persister = getPersister();
            EntityRegionAccessStrategy cache = persister.getCacheAccessStrategy();
            if (cache != null) {
                CacheKey key = session.generateCacheKey(id, persister);
                cache.putFromLoad(key, instance, session.getTimestamp(), null);
            }
        }
    }
}

// EntityUpdateAction 类似:
// -> cache.putFromLoad(key, instance, session.getTimestamp(), version);

// EntityDeleteAction 类似:
// -> cache.evict(key);  // 从二级缓存中移除
```

### 9.2.4 缓存读取时机

```java
// Loader.loadFromSecondLevelCache()
protected Object loadFromSecondLevelCache(
    LoadEvent event, EntityPersister persister,
    EntityKey keyToLoad, LoadType options
) {
    if (!persister.hasCache()) return null;
    
    EntityRegionAccessStrategy cache = persister.getCacheAccessStrategy();
    CacheKey cacheKey = session.generateCacheKey(event.getEntityId(), persister);
    
    // 1. 从缓存中读取
    Object cachedEntity = cache.get(session, cacheKey, session.getTimestamp());
    
    if (cachedEntity != null) {
        // 2. 缓存命中 -> 注册到一级缓存
        session.getPersistenceContext().addEntity(keyToLoad, cachedEntity, Status.MANAGED);
        return cachedEntity;
    }
    return null; // 缓存未命中
}
```

### 9.2.5 缓存一致性保证

```java
// Hibernate 二级缓存的一致性策略:
// 1. READ_ONLY -- 只读缓存（适合不变化的参考数据）
// 2. NONSTRICT_READ_WRITE -- 非严格读写
//    更新时简单地使缓存失效（evict）
//    可能在极短时间内读到旧数据
// 3. READ_WRITE -- 读写缓存（使用软锁机制）
//    更新时: 获取软锁 -> 使缓存失效 -> 数据库更新 -> 释放软锁
//    读取时: 若发现软锁 -> 直接查数据库（绕过缓存）
// 4. TRANSACTIONAL -- 完全事务性缓存（需支持 XA 的提供者）

// UpdateTimestampsCache 用于辅助缓存失效:
//   记录每个表最后更新的时间戳
//   查询缓存命中时，对比查询涉及的表的时间戳，
//   若任何相关表在查询结果缓存后有过更新，则使缓存失效
```

## 9.3 查询缓存实现

### 9.3.1 StandardQueryCache

```java
public class StandardQueryCache implements QueryCache {
    private final QueryResultsRegion cacheRegion;
    private final UpdateTimestampsCache updateTimestampsCache;
    
    // 存储查询结果
    public boolean put(QueryKey key, Type[] returnTypes, List<Object> result,
                        boolean isNaturalKeyLookup, SharedSessionContractImplementor session) {
        // 1. 从查询结果中提取 EntityKey 列表
        List<EntityKey[]> entityKeyList = extractEntityKeys(result, returnTypes);
        
        // 2. 获取当前时间戳
        long timestamp = session.getTimestamp();
        
        // 3. 存储: QueryKey -> (entityKeyList, timestamp)
        cacheRegion.put(key, new CachedQueryResult(entityKeyList, timestamp));
        return true;
    }
    
    // 获取查询结果
    public List<Object> get(QueryKey key, Type[] returnTypes, 
                            boolean isNaturalKeyLookup,
                            Set<EntityPersister> spaces,
                            SharedSessionContractImplementor session) {
        // 1. 从缓存获取
        CachedQueryResult cached = (CachedQueryResult) cacheRegion.get(key);
        if (cached == null) return null;
        
        // 2. 时间戳验证（是否有表在查询之后被更新）
        if (!isUpToDate(spaces, cached.getTimestamp())) {
            cacheRegion.evict(key);  // 缓存失效
            return null;
        }
        
        // 3. 根据 EntityKey 列表从一级/二级缓存中组装结果
        return assembleResult(cached.getEntityKeyList(), returnTypes, session);
    }
}
```

### 9.3.2 UpdateTimestampsCache

```java
public class UpdateTimestampsCache {
    private final TimestampsRegion region;
    
    // 当表发生更新时，记录当前时间戳
    public void preInvalidate(Serializable[] spaces) {
        long timestamp = region.nextTimestamp() + region.getTimeout();
        for (Serializable space : spaces) {
            region.put(space, timestamp);
        }
    }
    
    // 检查从某个时间点以来是否有表被更新过
    public boolean isUpToDate(Set<Serializable> spaces, Long timestamp,
                               SharedSessionContractImplementor session) {
        for (Serializable space : spaces) {
            Long lastUpdate = (Long) region.get(space);
            if (lastUpdate != null && lastUpdate > timestamp) {
                return false;  // 有更新的数据，缓存失效
            }
        }
        return true;
    }
}
```

### 9.3.3 QueryKey 的构造

```java
public class QueryKey {
    private final String sqlQueryString;       // SQL 或 HQL 查询字符串
    private final Type[] positionalParameterTypes;
    private final Map<String, TypedValue> namedParameters;
    private final Integer firstRow;
    private final Integer maxRows;
    private final Set<EntityPersister> filterSpaces;
    private final String tenantIdentifier;
    private final int hashCode;
    
    // QueryKey 和 HQLQueryPlanKey 的区分:
    // - HQLQueryPlanKey: 用于查询计划缓存（解析后的 SQL 模板）
    // - QueryKey:       用于查询结果缓存（带具体参数值的查询）
}
```

---

# 10. 源码调试技巧

## 10.1 环境搭建

### 10.1.1 获取源码

```bash
# 方式一: 从 GitHub clone
git clone https://github.com/hibernate/hibernate-orm.git
cd hibernate-orm
git checkout 5.6.15.Final   # 或检查到目标版本 Tag

# 方式二: IDE 中直接下载源码
# Maven: 在 pom.xml 中点击 Download Sources
# Gradle: 右键 -> Download Sources
```

Hibernate 源码使用 Gradle 构建，核心模块在 `hibernate-core` 目录下。

### 10.1.2 构建与导入

```bash
# 构建整个项目（需要较长时间）
./gradlew build -x test

# 只构建核心模块
./gradlew :hibernate-core:build -x test
```

导入 IDE（IntelliJ IDEA）：
1. File -> New -> Project from Existing Sources
2. 选择 `build.gradle` 文件
3. IDE 会自动解析 Gradle 项目结构

## 10.2 关键断点位置建议

### 10.2.1 启动流程断点

```java
// 1. 配置入口
org.hibernate.boot.registry.StandardServiceRegistryBuilder
  -> build() 方法                                     // 行: 约 L280

// 2. 元数据构建
org.hibernate.boot.internal.MetadataBuilderImpl
  -> build() 方法                                     // 行: 约 L170

// 3. 注解绑定
org.hibernate.cfg.AnnotationBinder
  -> bindClass() 方法                                 // 行: 约 L520

// 4. SessionFactory 构建
org.hibernate.internal.SessionFactoryImpl
  -> 构造函数                                         // 行: 约 L320

// 5. EntityPersister 创建
org.hibernate.persister.entity.EntityPersisterFactoryImpl
  -> createEntityPersister() 方法                     // 行: 约 L80
```

### 10.2.2 持久化操作断点

```java
// 1. persist 流程
org.hibernate.internal.SessionImpl
  -> firePersist() 方法                               // 行: 约 L1400
org.hibernate.event.internal.DefaultPersistEventListener
  -> onPersist() 方法                                 // 行: 约 L140
org.hibernate.action.internal.EntityInsertAction
  -> execute() 方法                                   // 行: 约 L85

// 2. find/load 流程
org.hibernate.internal.SessionImpl
  -> immediateLoad() 方法                             // 行: 约 L2150
org.hibernate.loader.Loader
  -> loadEntity() 方法                                // 行: 约 L1900

// 3. merge 流程
org.hibernate.event.internal.DefaultMergeEventListener
  -> onMerge() 方法                                   // 行: 约 L90

// 4. remove 流程
org.hibernate.event.internal.DefaultDeleteEventListener
  -> onDelete() 方法                                  // 行: 约 L65

// 5. flush 流程
org.hibernate.event.internal.DefaultFlushEventListener
  -> onFlush() 方法                                   // 行: 约 L55

// 6. 脏检查
org.hibernate.event.internal.DefaultFlushEntityEventListener
  -> onFlushEntity() 方法                             // 行: 约 L75
org.hibernate.persister.entity.AbstractEntityPersister
  -> findDirty() 方法                                 // 行: 约 L4100
```

### 10.2.3 懒加载/代理断点

```java
// 1. 代理创建
org.hibernate.proxy.pojo.bytebuddy.ByteBuddyProxyFactory
  -> getProxy() 方法                                  // 行: 约 L45

// 2. 代理初始化
org.hibernate.proxy.LazyInitializer
  -> initialize() 方法                                // 行: 约 L175

// 3. 集合初始化
org.hibernate.collection.internal.AbstractPersistentCollection
  -> read() 方法                                      // 行: 约 L380
  -> write() 方法                                     // 行: 约 L395
```

### 10.2.4 查询执行断点

```java
// 1. HQL 解析（Hibernate 5）
org.hibernate.hql.internal.ast.QueryTranslatorImpl
  -> compile() 方法                                   // 行: 约 L220

// 2. SQL 生成
org.hibernate.hql.internal.ast.SqlGenerator
  -> statement() 方法                                 // 行: 约 L110

// 3. SQM 解析（Hibernate 6）
org.hibernate.query.sqm.internal.SemanticQueryInterpreter
  -> interpret() 方法                                 // 行: 约 L60

// 4. 查询计划缓存
org.hibernate.engine.query.spi.QueryPlanCache
  -> getHQLQueryPlan() 方法                           // 行: 约 L70

// 5. 结果集处理
org.hibernate.loader.Loader
  -> processResultSet() 方法                          // 行: 约 L1450

// 6. 两阶段加载
org.hibernate.engine.internal.TwoPhaseLoad
  -> initializeEntity() 方法                          // 行: 约 L180
```

### 10.2.5 缓存断点

```java
// 1. 二级缓存读取
org.hibernate.event.internal.DefaultLoadEventListener
  -> loadFromSecondLevelCache() 方法                  // 行: 约 L240

// 2. 二级缓存写入
org.hibernate.action.internal.EntityInsertAction
  -> doAfterTransactionCompletion() 方法              // 行: 约 L115

// 3. 查询缓存
org.hibernate.cache.internal.StandardQueryCache
  -> get() 方法                                       // 行: 约 L120
  -> put() 方法                                       // 行: 约 L70
```

## 10.3 日志配合调试

### 10.3.1 关键日志配置

```xml
<!-- logback.xml 或 log4j2.xml -->
<!-- 开启 SQL 日志 -->
<logger name="org.hibernate.SQL" level="DEBUG"/>

<!-- 开启 SQL 参数绑定 -->
<logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>

<!-- 开启二级缓存日志 -->
<logger name="org.hibernate.cache" level="DEBUG"/>

<!-- 开启事务日志 -->
<logger name="org.hibernate.engine.transaction" level="DEBUG"/>

<!-- 开启 JDBC 连接日志 -->
<logger name="org.hibernate.engine.jdbc.connections" level="DEBUG"/>

<!-- 开启 Session 指标日志 -->
<logger name="org.hibernate.stat" level="DEBUG"/>

<!-- 开启懒加载日志 -->
<logger name="org.hibernate.proxy" level="DEBUG"/>

<!-- 开启事件日志 -->
<logger name="org.hibernate.event.internal" level="DEBUG"/>
```

### 10.3.2 Hibernate 内置统计

```java
// 开启统计信息收集
// 配置: hibernate.generate_statistics = true

// 获取统计信息
Statistics stats = sessionFactory.getStatistics();

// 关键统计指标:
stats.getEntityFetchCount();        // 实体加载次数
stats.getSecondLevelCacheHitCount(); // 二级缓存命中次数
stats.getSecondLevelCacheMissCount(); // 二级缓存未命中次数
stats.getQueryExecutionCount();     // 查询执行次数
stats.getQueryExecutionMaxTime();   // 慢查询最大时间
stats.getFlushCount();              // flush 次数
stats.getSessionOpenCount();        // Session 打开次数
stats.getSessionCloseCount();       // Session 关闭次数
stats.getConnectCount();            // JDBC 连接获取次数

// 重置统计
stats.clear();
```

### 10.3.3 调试最佳实践

1. **使用条件断点**：在大循环或频繁调用的方法中，设置条件断点（如 `id.equals("target-id")`）减少中断次数。

2. **栈帧分析**：在断点处查看调用栈（Call Stack），完整理解调用链。

3. **变量监视**：关注以下核心变量：
   - `PersistenceContext.entitiesByKey.size()` — 一级缓存中的实体数量
   - `ActionQueue.insertions.size()` — 待执行的 INSERT 操作数量
   - `EntityEntry.loadedState` — 实体的快照数据
   - `LazyInitializer.initialized` — 代理是否已初始化

4. **源码版本一致**：确保调试时使用的 Hibernate 版本与阅读的源码一致。不同版本（尤其是 5.x 和 6.x）的内部实现差异很大。

5. **从测试类入手**：Hibernate 官方仓库包含大量单元测试，在测试类中设置断点可以快速理解特定功能的行为。找到 `hibernate-core/src/test/java` 中对应的测试，直接断点调试。

6. **使用 Arthas 或 Byteman**：对于无法修改源码的生产环境，可以使用 Arthas 或 Byteman 这类字节码级别的诊断工具，在不重启应用的情况下注入监控代码。

7. **模拟调用链**：除了断点调试，还可以使用 IDEA 的 Diagrams 功能查看类继承层次和方法调用关系图：
   - 右键类名 -> Diagrams -> Show Diagram
   - 右键方法名 -> Show Usages 或 Call Hierarchy

8. **关注 SPI 接口**：Hibernate 大量使用了 SPI 模式。追踪代码时，从 SPI 接口的调用者开始，逐步深入到 `internal` 包下的默认实现。例如：
   - `SessionImplementor` (SPI) -> `SessionImpl` (internal)
   - `EntityPersister` (SPI) -> `AbstractEntityPersister` -> `SingleTableEntityPersister`
   - `RegionFactory` (SPI) -> `EhcacheRegionFactory` / `JCacheRegionFactory`

## 10.4 常见调试场景速查

| 调试场景 | 断点位置（类 -> 方法） | 关注变量 |
|----------|----------------------|----------|
| 实体未持久化 | `DefaultPersistEventListener.onPersist()` | `event.getObject()` / `ActionQueue.insertions` |
| N+1 查询问题 | `Loader.loadEntity()` | 调用次数 / `ResultSet` |
| 懒加载异常 | `LazyInitializer.initialize()` | `session` 是否 null / `session.isOpen()` |
| 脏检查未触发 | `DefaultFlushEntityEventListener.onFlushEntity()` | `currentState` vs `loadedState` |
| 更新丢失 | `EntityUpdateAction.execute()` | `version` / 影响行数 |
| 查询慢 | `Loader.processResultSet()` | SQL / 参数 / ResultSet 大小 |
| 缓存不命中 | `StandardQueryCache.get()` | `QueryKey` / `UpdateTimestampsCache.isUpToDate()` |
| Session 泄漏 | `SessionImpl` 构造函数 / `close()` | 打开/关闭计数器 |
| 死锁 | `ActionQueue.prepareActions()` | `ActionSorter` 排序结果 |
| 批处理不生效 | `BatchingBatch.addToBatch()` | `batchSize` / `currentBatchCount` |
