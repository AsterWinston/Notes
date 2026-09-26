# 1. 配置方式概述

Hibernate 提供了三种主要的配置方式：**XML 配置**、**注解配置**和**编程式配置**。三种方式可以单独使用，也可以混合使用，灵活适应不同项目的需求。

历史演变路径：

```
hbm.xml 映射文件（Hibernate 2/3）
→ hibernate.cfg.xml + hbm.xml（Hibernate 3 主流）
→ JPA 注解（Hibernate 3.5+，JPA 1.0）
→ 编程式 Bootstrap（Hibernate 5+，彻底 XML-free）
→ Spring Boot 自动配置（业界主流）
```

**配置优先级规则**（从高到低）：

1. `hibernate.properties` 文件中的属性
2. `hibernate.cfg.xml` 中的 `<property>` 元素
3. 编程式 `Configuration.setProperty()` / `MetadataBuilder.applySetting()`
4. 注解中的默认值
5. Hibernate 内部的默认值

> 注意：properties 文件优先级最高，意味着即使 cfg.xml 或编程式设置了同名属性，properties 文件的值也会覆盖它们。这是初学者常见的困惑点。

**配置的层次模型**：

| 层次 | 作用域 | 典型配置 | 生命周期 |
|------|--------|----------|----------|
| Bootstrap 级 | 全局唯一 | 数据源、Dialect、连接池 | 应用启动时确定 |
| SessionFactory 级 | 全局唯一 | 缓存策略、统计开关、命名策略 | 与 SessionFactory 同生命周期 |
| Session 级 | 每个 Session 独立 | 拦截器、过滤器、FlushMode | 与 Session 同生命周期 |
| Query 级 | 每个查询 | 超时、FetchSize、只读模式 | 单次查询 |

---

# 2. XML 配置方式

## 2.1 hibernate.cfg.xml

XML 配置文件是 Hibernate 最传统、最完整的配置方式。文件默认放置于 classpath 根目录（通常是 `src/main/resources/`）。

### 基本结构

```xml
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
    <session-factory>
        <!-- 数据库连接属性 -->
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/mydb</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">secret</property>

        <!-- 方言与 SQL -->
        <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>

        <!-- 映射文件（HBM 方式） -->
        <mapping resource="com/example/domain/User.hbm.xml"/>
        <mapping resource="com/example/domain/Order.hbm.xml"/>

        <!-- 映射类（注解方式） -->
        <mapping class="com.example.domain.Product"/>
    </session-factory>
</hibernate-configuration>
```

### 属性命名规范

所有 Hibernate 核心属性前缀为 `hibernate.`：
- `hibernate.connection.*` — 数据库连接
- `hibernate.cache.*` — 缓存
- `hibernate.jdbc.*` — JDBC 行为
- `hibernate.transaction.*` — 事务

## 2.2 核心配置属性分类详解

### 2.2.1 数据库连接配置

| 属性 | 说明 | 示例值 |
|------|------|--------|
| `hibernate.connection.driver_class` | JDBC 驱动类 | `com.mysql.cj.jdbc.Driver` |
| `hibernate.connection.url` | 数据库连接 URL | `jdbc:mysql://localhost:3306/mydb` |
| `hibernate.connection.username` | 用户名 | `root` |
| `hibernate.connection.password` | 密码 | `secret` |
| `hibernate.connection.pool_size` | Hibernate 内置连接池大小（仅开发用） | `5` |
| `hibernate.connection.datasource` | JNDI 数据源名 | `java:/comp/env/jdbc/MyDS` |
| `hibernate.connection.isolation` | 事务隔离级别（1-READ_UNCOMMITTED, 2-READ_COMMITTED, 4-REPEATABLE_READ, 8-SERIALIZABLE） | `4` |
| `hibernate.connection.autocommit` | 自动提交 | `false` |
| `hibernate.connection.provider_class` | 自定义 ConnectionProvider 实现 | `org.hibernate.hikaricp.internal.HikariCPConnectionProvider` |
| `hibernate.connection.CharSet` | 字符集 | `UTF-8` |
| `hibernate.connection.characterEncoding` | 字符编码（MySQL 驱动参数） | `UTF-8` |

完整的数据源配置示例：

```xml
<!-- 直接连接方式 -->
<property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
<property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/mydb</property>
<property name="hibernate.connection.username">appuser</property>
<property name="hibernate.connection.password">p@ssw0rd</property>
<property name="hibernate.connection.isolation">2</property>
<property name="hibernate.connection.autocommit">false</property>
```

```xml
<!-- JNDI 方式（应用服务器环境） -->
<property name="hibernate.connection.datasource">java:comp/env/jdbc/MyDataSource</property>
```

### 2.2.2 连接池配置

Hibernate 内置连接池非常简单，**严禁生产环境使用**。生产环境必须集成第三方连接池。

#### C3P0 连接池

```xml
<property name="hibernate.c3p0.min_size">5</property>
<property name="hibernate.c3p0.max_size">50</property>
<property name="hibernate.c3p0.timeout">300</property>
<property name="hibernate.c3p0.max_statements">50</property>
<property name="hibernate.c3p0.idle_test_period">300</property>
<property name="hibernate.c3p0.acquire_increment">5</property>
<property name="hibernate.c3p0.validate">true</property>
```

#### HikariCP 连接池（推荐）

```xml
<property name="hibernate.connection.provider_class">
    org.hikaricp.HikariConnectionProvider
</property>
<property name="hibernate.hikari.dataSourceClassName">com.mysql.cj.jdbc.MysqlDataSource</property>
<property name="hibernate.hikari.maximumPoolSize">20</property>
<property name="hibernate.hikari.minimumIdle">5</property>
<property name="hibernate.hikari.connectionTimeout">30000</property>
<property name="hibernate.hikari.idleTimeout">600000</property>
<property name="hibernate.hikari.maxLifetime">1800000</property>
```

连接池选型建议：
- **HikariCP**：性能最优，Spring Boot 2.x 默认，适合绝大多数场景
- **C3P0**：老牌连接池，稳定但性能中规中矩，存量项目多见
- **DBCP2**：Apache 官方，适合与日过 Apache 生态强绑定的项目

### 2.2.3 Dialect 配置

`hibernate.dialect` 告诉 Hibernate 如何生成适配特定数据库的 SQL 方言（分页语法、类型映射、函数支持等）。

常见数据库 Dialect 类：

```xml
<!-- MySQL 8.x -->
<property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>

<!-- MySQL 5.x -->
<property name="hibernate.dialect">org.hibernate.dialect.MySQL57Dialect</property>

<!-- PostgreSQL 10+ -->
<property name="hibernate.dialect">org.hibernate.dialect.PostgreSQL10Dialect</property>

<!-- Oracle 12c+ -->
<property name="hibernate.dialect">org.hibernate.dialect.Oracle12cDialect</property>

<!-- SQL Server 2012+ -->
<property name="hibernate.dialect">org.hibernate.dialect.SQLServer2012Dialect</property>

<!-- H2 -->
<property name="hibernate.dialect">org.hibernate.dialect.H2Dialect</property>
```

Hibernate 5+ 支持自动 Dialect 解析（通过 `DialectResolver` SPI）：

```xml
<!-- 启用自动检测（默认已启用，无需显式设置） -->
<property name="hibernate.session_factory.statement_inspector">
    org.hibernate.dialect.DialectResolver
</property>
```

> 注意：自动解析依赖数据库 JDBC 元数据，某些驱动版本可能无法准确识别。生产环境建议**显式指定**。

### 2.2.4 SQL 生成与日志

日志控制属性：

```xml
<!-- 显示 SQL -->
<property name="hibernate.show_sql">true</property>
<!-- 格式化 SQL -->
<property name="hibernate.format_sql">true</property>
<!-- SQL 注释（标注生成的 HQL/JPQL 来源） -->
<property name="hibernate.use_sql_comments">true</property>
<!-- 高亮 SQL（Hibernate 6+） -->
<property name="hibernate.highlight_sql">true</property>
```

Hibernate 日志分类器（Logback 示例）：

```xml
<!-- logback.xml -->
<configuration>
    <!-- 打印 SQL 语句 -->
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <!-- 打印 SQL 参数绑定值 -->
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
    <!-- 打印 SQL 结果集提取值 -->
    <logger name="org.hibernate.type.descriptor.sql.BasicExtractor" level="TRACE"/>
    <!-- Hibernate 统计 -->
    <logger name="org.hibernate.stat" level="DEBUG"/>
    <!-- 缓存行为 -->
    <logger name="org.hibernate.cache" level="DEBUG"/>
    <!-- 事务 -->
    <logger name="org.hibernate.engine.transaction" level="DEBUG"/>
</configuration>
```

使用 Log4j2 时的等价配置：

```xml
<!-- log4j2.xml -->
<Loggers>
    <Logger name="org.hibernate.SQL" level="debug" additivity="false">
        <AppenderRef ref="Console"/>
    </Logger>
    <Logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="trace" additivity="false">
        <AppenderRef ref="Console"/>
    </Logger>
</Loggers>
```

### 2.2.5 DDL 自动生成

`hibernate.hbm2ddl.auto` 控制启动时数据库 Schema 的自动操作：

| 值 | 行为 | 适用场景 |
|----|------|----------|
| `none` | 不做任何操作 | 生产环境 |
| `validate` | 仅验证实体映射与 Schema 是否匹配，不匹配则抛异常 | CI/CD 验证阶段 |
| `update` | 自动添加新列/新表，**不删除已有列或表** | 开发环境 |
| `create` | 每次启动重建所有表（先 DROP 再 CREATE） | 集成测试 |
| `create-drop` | 启动时创建，SessionFactory 关闭时删除 | 单元测试 |

validate 验证的项目：
- 每个 `@Entity` 是否在数据库中都有对应表
- 每个 `@Column` 是否在表中都有对应列
- 列的类型是否与 Java 类型兼容
- 是否缺少必要的字段

update 的局限性：
- **不会删除列**：如果 Java 类删除了字段，数据库中的列会保留（产生冗余）
- **不会修改列约束**：不会自动修改 nullable / unique 约束
- **不会处理索引和外键的变更**

```xml
<property name="hibernate.hbm2ddl.auto">validate</property>
```

**生产环境建议**：使用 Flyway 或 Liquibase 管理数据库迁移，将 `hbm2ddl.auto` 设为 `none` 或 `validate`。

```xml
<!-- Flyway 配合 Hibernate -->
<property name="hibernate.hbm2ddl.auto">validate</property>
<!-- Flyway 独立管理 DDL -->
<bean id="flyway" class="org.flywaydb.core.Flyway" init-method="migrate">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

Spring Boot 中的对应配置：

```properties
# Spring Boot 属性（底层等同于 hibernate.hbm2ddl.auto）
spring.jpa.hibernate.ddl-auto=validate
```

### 2.2.6 命名策略

Hibernate 5+ 将命名策略拆分为**隐式命名**和**物理命名**两个阶段：

```
Java 实体字段名 → ImplicitNamingStrategy → 逻辑列名 → PhysicalNamingStrategy → 数据库物理列名
```

#### ImplicitNamingStrategy（隐式命名策略）

决定未显式指定 `@Column(name=...)` 时，如何推导逻辑列名。

五种内置实现：

| 实现类 | 字段命名 | 外键命名 | 索引命名 |
|--------|----------|----------|----------|
| `ImplicitNamingStrategyJpaCompliantImpl`（默认） | 驼峰保持 | `FK_xxx` | `IDX_xxx` |
| `ImplicitNamingStrategyLegacyHbmImpl` | 原始 hbm 规则 | `FKxxx` | 特殊规则 |
| `ImplicitNamingStrategyLegacyJpaImpl` | 早期 JPA 规则 | `FK_xxx` | `IDX_xxx` |
| `ImplicitNamingStrategyComponentPathImpl` | 组件路径（如 `address_city`） | 标准 | 标准 |

命名规则对比：

| 场景 | JpaCompliantImpl | LegacyHbmImpl |
|------|------------------|---------------|
| 实体 → 表名 | `UserProfile` | `UserProfile` (不做转换) |
| 普通字段 | `firstName` → `firstName` | `firstName` → `firstName` |
| 嵌入组件路径 | 不拼路径 | 不拼路径 |
| 连接表 | `User_Group` | 使用 identifier1 / identifier2 |
| 外键列 | `user_id` | `user` |

#### PhysicalNamingStrategy（物理命名策略）

将逻辑列名转换为数据库物理名称：

```xml
<!-- 不转换（默认） -->
<property name="hibernate.physical_naming_strategy">
    org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
</property>
```

`PhysicalNamingStrategyStandardImpl`：直接使用逻辑列名，不做任何转换。字段 `firstName` 生成的列名就是 `firstName`。

Spring Boot 默认使用 `SpringPhysicalNamingStrategy`，将驼峰转换为小写下划线：

```java
// 字段：firstName → 列名：first_name
// 类名：UserProfile → 表名：user_profile
```

自定义物理命名策略：

```java
public class CustomPhysicalNamingStrategy extends PhysicalNamingStrategyStandardImpl {

    @Override
    public Identifier toPhysicalTableName(Identifier name, JdbcEnvironment context) {
        // 所有表名添加 T_ 前缀
        return Identifier.toIdentifier("T_" + name.getText());
    }

    @Override
    public Identifier toPhysicalColumnName(Identifier name, JdbcEnvironment context) {
        // 驼峰转小写下划线
        String newName = name.getText()
            .replaceAll("([a-z])([A-Z])", "$1_$2")
            .toLowerCase();
        return Identifier.toIdentifier(newName);
    }
}
```

配置命名策略：

```xml
<!-- 隐式命名策略 -->
<property name="hibernate.implicit_naming_strategy">
    org.hibernate.boot.model.naming.ImplicitNamingStrategyJpaCompliantImpl
</property>
<!-- 物理命名策略 -->
<property name="hibernate.physical_naming_strategy">
    com.example.CustomPhysicalNamingStrategy
</property>
```

### 2.2.7 缓存配置

```xml
<!-- 启用二级缓存 -->
<property name="hibernate.cache.use_second_level_cache">true</property>
<!-- 启用查询缓存 -->
<property name="hibernate.cache.use_query_cache">true</property>
<!-- 缓存区域工厂 -->
<property name="hibernate.cache.region.factory_class">
    org.hibernate.cache.ehcache.EhCacheRegionFactory
</property>
<!-- 二级缓存提供者配置文件路径 -->
<property name="hibernate.cache.provider_configuration_file_resource_path">
    ehcache.xml
</property>
<!-- 最小化 put 操作（减少缓存写入） -->
<property name="hibernate.cache.use_minimal_puts">true</property>
<!-- 结构化条目（方便调试，轻微性能损耗） -->
<property name="hibernate.cache.use_structured_entries">true</property>
<!-- 自动驱逐集合缓存 -->
<property name="hibernate.cache.auto_evict_collection_cache">true</property>
```

不同缓存提供者的工厂类：

| 缓存提供者 | Factory 类 |
|-----------|-----------|
| Ehcache 2.x | `org.hibernate.cache.ehcache.EhCacheRegionFactory` |
| Ehcache 3.x (JCache) | `org.hibernate.cache.jcache.JCacheRegionFactory` |
| Hazelcast | `org.hibernate.cache.hazelcast.HazelcastCacheRegionFactory` |
| Infinispan | `org.hibernate.cache.infinispan.InfinispanRegionFactory` |

完整示例（Hibernate 5 + Ehcache 3）：

```xml
<property name="hibernate.cache.use_second_level_cache">true</property>
<property name="hibernate.cache.use_query_cache">true</property>
<property name="hibernate.cache.region.factory_class">
    org.hibernate.cache.jcache.internal.JCacheRegionFactory
</property>
<property name="hibernate.javax.cache.provider">
    org.ehcache.jsr107.EhcacheCachingProvider
</property>
<property name="hibernate.javax.cache.uri">ehcache.xml</property>
```

### 2.2.8 事务配置

```xml
<!-- 事务协调器 -->
<property name="hibernate.transaction.coordinator_class">
    org.hibernate.transaction.JTATransactionCoordinator
</property>
<!-- JTA 平台（应用服务器环境） -->
<property name="hibernate.transaction.jta.platform">
    org.hibernate.engine.transaction.jta.platform.internal.JBossAppServerJtaPlatform
</property>
<!-- 当前 Session 上下文 --->
<property name="hibernate.current_session_context_class">thread</property>
```

`hibernate.current_session_context_class` 的值：

| 值 | 说明 |
|----|------|
| `thread` | 通过 `SessionFactory.getCurrentSession()` 获取，Session 绑定到当前线程 |
| `jta` | Session 绑定到当前 JTA 事务 |
| `managed` | 由外部容器管理，`getCurrentSession()` 委托给外部 |
| 自定义类 | 实现 `CurrentSessionContext` 接口 |

事务相关其他属性：

```xml
<!-- 事务完成前 flush -->
<property name="hibernate.transaction.flush_before_completion">true</property>
<!-- 事务完成后自动关闭 Session -->
<property name="hibernate.transaction.auto_close_session">true</property>
```

### 2.2.9 批处理配置

```xml
<!-- JDBC 批处理大小（强烈推荐设置） -->
<property name="hibernate.jdbc.batch_size">30</property>
<!-- 版本化数据也参与批处理 -->
<property name="hibernate.jdbc.batch_versioned_data">true</property>
<!-- 排序 INSERT 语句（减少死锁、提高批处理效率） -->
<property name="hibernate.order_inserts">true</property>
<!-- 排序 UPDATE 语句 -->
<property name="hibernate.order_updates">true</property>
<!-- 批量加载大小 -->
<property name="hibernate.default_batch_fetch_size">16</property>
```

批处理属性详解：

- `hibernate.jdbc.batch_size`（默认 -1 关闭）：批量提交 INSERT/UPDATE/DELETE，极大提升批量操作性能。建议值 20-50。MySQL 需在 JDBC URL 追加 `rewriteBatchedStatements=true`。
- `hibernate.order_inserts`：对 INSERT 按实体类型排序，使相同表的 INSERT 可以被批处理。
- `hibernate.order_updates`：同理对 UPDATE 排序。

```xml
<!-- JDBC Fetch 大小 -->
<property name="hibernate.jdbc.fetch_size">100</property>
<!-- 可滚动结果集 -->
<property name="hibernate.jdbc.use_scrollable_resultset">true</property>
<!-- 使用流处理二进制 -->
<property name="hibernate.jdbc.use_streams_for_binary">true</property>
```

### 2.2.10 统计与监控

```xml
<!-- 生成统计信息 -->
<property name="hibernate.generate_statistics">true</property>
<!-- 会话事件日志 -->
<property name="hibernate.session.events.log">true</property>
<!-- 慢查询阈值（Hibernate 6+，单位毫秒） -->
<property name="hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS">1000</property>
```

JMX 统计：

```xml
<!-- 全局统计 -->
<property name="hibernate.jmx.enabled">true</property>
<!-- SessionFactory 注册名 -->
<property name="hibernate.session_factory_name">MyAppSessionFactory</property>
```

获取统计数据的代码：

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
Statistics statistics = sessionFactory.getStatistics();

long queryCount = statistics.getQueryExecutionCount();
long queryMaxTime = statistics.getQueryExecutionMaxTime();
long hitCount = statistics.getSecondLevelCacheHitCount();
long missCount = statistics.getSecondLevelCacheMissCount();
double hitRatio = statistics.getSecondLevelCacheHitRatio();
```

### 2.2.11 其他常用配置

```xml
<!-- 默认 Schema / Catalog -->
<property name="hibernate.default_schema">public</property>
<property name="hibernate.default_catalog">mydb</property>

<!-- 最大外连接抓取深度 -->
<property name="hibernate.max_fetch_depth">3</property>

<!-- 字节码增强提供者（默认 bytebuddy，Hibernate 5.3+） -->
<property name="hibernate.bytecode.provider">bytebuddy</property>
<!-- 字节码 reflection optimizer -->
<property name="hibernate.bytecode.use_reflection_optimizer">true</property>

<!-- 无事务懒加载（不推荐开启，可能导致 N+1） -->
<property name="hibernate.enable_lazy_load_no_trans">false</property>

<!-- 分页与集合抓取冲突时失败（false 则回退到内存分页） -->
<property name="hibernate.query.fail_on_pagination_over_collection_fetch">true</property>

<!-- IN 子句参数填充优化（提高语句缓存复用率） -->
<property name="hibernate.query.in_clause_parameter_padding">true</property>

<!-- JOINED 继承忽略显式鉴别器列 -->
<property name="hibernate.discriminator.ignore_explicit_for_joined">true</property>

<!-- 启用空组件/嵌入对象 -->
<property name="hibernate.create_empty_composites.enabled">true</property>
```

---

# 3. 注解配置方式

## 3.1 JPA 标准注解与配置文件

JPA 标准注解（`@Entity`、`@Table`、`@Id`、`@GeneratedValue` 等）是注解配置的基石。它们属于 `javax.persistence.*` 包（JPA 2.x）或 `jakarta.persistence.*` 包（JPA 3.x / Spring Boot 3）。

在配置层面，JPA 有自身的配置文件体系：

### persistence.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
             http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">

    <!-- 持久化单元：逻辑隔离的一组实体类 -->
    <persistence-unit name="myPU" transaction-type="RESOURCE_LOCAL">
        <!-- JPA 标准属性 -->
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

        <!-- 显式列出实体类 -->
        <class>com.example.domain.User</class>
        <class>com.example.domain.Order</class>

        <!-- 是否排除未列出的类 -->
        <exclude-unlisted-classes>false</exclude-unlisted-classes>

        <!-- 从外部 JAR 加载实体 -->
        <jar-file>lib/domain.jar</jar-file>

        <!-- 数据源（JTA 模式） -->
        <jta-data-source>java:/comp/env/jdbc/MyDS</jta-data-source>

        <!-- 属性透传给 Hibernate -->
        <properties>
            <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQL10Dialect"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.hbm2ddl.auto" value="validate"/>
        </properties>
    </persistence-unit>
</persistence>
```

### orm.xml — XML 覆盖/补充注解

`orm.xml` 可以覆盖或补充实体注解中的映射信息（高优先级）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entity-mappings xmlns="http://xmlns.jcp.org/xml/ns/persistence/orm"
                 version="2.2">
    <persistence-unit-metadata>
        <xml-mapping-metadata-complete/>
    </persistence-unit-metadata>
    <!-- 如果设置 metadata-complete=true，则完全忽略注解，只参考 XML -->
</entity-mappings>
```

JPA 注解扫描机制：
1. 查找 `persistence.xml` 中 `class` 元素显式列出的实体
2. 若 `exclude-unlisted-classes=false`（默认），扫描 persistence-unit root 下的 `@Entity` 类
3. 扫描 `jar-file` 元素指定的 JAR 中的实体

## 3.2 Hibernate 扩展注解

以下为 Hibernate 专有注解（`org.hibernate.annotations.*`），提供 JPA 标准之外的扩展能力。

### 过滤器注解

```java
// 定义过滤器
@FilterDef(name = "activeFilter",
           parameters = @ParamDef(name = "status", type = String.class))
@Entity
public class User {

    // 应用过滤器
    @Filter(name = "activeFilter",
            condition = "status = :status")
    private String status;
}

// 使用
session.enableFilter("activeFilter")
       .setParameter("status", "ACTIVE");
```

### 抓取策略注解

```java
@Entity
public class Order {
    @ManyToOne
    @Fetch(FetchMode.JOIN)  // JOIN 抓取
    @BatchSize(size = 10)   // 批量抓取
    private Customer customer;
}
```

### 自定义加载器

```java
@Entity
@Loader(namedQuery = "findUserById")
@NamedNativeQuery(name = "findUserById",
                  query = "SELECT * FROM users WHERE id = ? AND deleted = false",
                  resultClass = User.class)
public class User { ... }
```

### 自定义 DML SQL

```java
@Entity
@Table(name = "users")
@SQLInsert(sql = "INSERT INTO users (id, name, version) VALUES (?, ?, ?)")
@SQLUpdate(sql = "UPDATE users SET name = ?, version = ? WHERE id = ? AND version = ?")
@SQLDelete(sql = "DELETE FROM users WHERE id = ? AND version = ?")
@SQLSelect(sql = "SELECT * FROM users WHERE id = ?")
public class User {

    @Id
    private Long id;

    @Version
    private int version;

    private String name;
}
```

> `@SQLDelete` 常用于实现软删除：`UPDATE users SET deleted = true WHERE id = ?`。

### 扩展表注解

```java
@Entity
@org.hibernate.annotations.Table(
    appliesTo = "users",
    indexes = {
        @Index(name = "idx_name", columnList = "last_name, first_name")
    },
    foreignKeys = {
        @ForeignKey(name = "fk_user_dept",
                    inverseColumns = @JoinColumn(name = "department_id",
                                                  referencedColumnName = "id"))
    },
    comment = "用户表"
)
@javax.persistence.Table(name = "users")
public class User { ... }
```

> 注意区分 `@jakarta.persistence.Table` 与 `@org.hibernate.annotations.Table`。

### 公式计算列

```java
@Entity
public class Product {
    private BigDecimal price;
    private BigDecimal taxRate;

    @Formula("price * tax_rate / 100")
    private BigDecimal taxAmount;
}
```

### 类型映射

```java
@Entity
@TypeDefs({
    @TypeDef(name = "jsonb", typeClass = JsonBinaryType.class),
    @TypeDef(name = "money", typeClass = MonetaryAmountType.class)
})
public class Order {

    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> attributes;

    @Type(type = "money")
    private MonetaryAmount total;
}
```

### 子查询映射（视图实体）

```java
@Entity
@Subselect("SELECT u.id, u.name, count(o.id) as order_count " +
           "FROM users u LEFT JOIN orders o ON u.id = o.user_id " +
           "GROUP BY u.id, u.name")
@Synchronize({"users", "orders"})
public class UserOrderSummary {

    @Id
    private Long id;
    private String name;
    private Long orderCount;
}
```

`@Synchronize` 告诉 Hibernate 该视图表依赖于哪些基表，以便在 DML 后刷新相关缓存。

### 乐观锁注解

```java
@Entity
@OptimisticLocking(type = OptimisticLockType.DIRTY) // 仅 dirty 字段参与版本校验
public class Document {

    @Version
    private int version;

    @OptimisticLock(excluded = true) // 排除该字段
    private String contentHash;
}
```

并发锁类型：
- `OptimisticLockType.VERSION`（默认）：基于 @Version 字段
- `OptimisticLockType.DIRTY`：仅比较修改过的字段
- `OptimisticLockType.ALL`：比较所有字段

### 动态 SQL 注解

```java
@Entity
@DynamicInsert  // INSERT 只包含非 null 字段
@DynamicUpdate  // UPDATE 只包含变化的字段
@SelectBeforeUpdate // UPDATE 前先 SELECT 比较
public class User {
    // ...
}
```

`@DynamicInsert` / `@DynamicUpdate` 适合包含大量可空字段的表，减少 SQL 长度，提高性能。但在字段数少时可能降低效率（需动态分析）。

### 约束与条件过滤

```java
@Entity
@Check(constraints = "age >= 0 AND age <= 150")
@Where(clause = "deleted = false")
public class User {

    @ColumnTransformer(
        read = "AES_DECRYPT(ssn, 'secret')",
        write = "AES_ENCRYPT(?, 'secret')"
    )
    private String ssn;
}
```

- `@Check`：生成数据库 CHECK 约束（DDL 生成时）
- `@Where`：默认给所有查询加 WHERE 条件
- `@ColumnTransformer`：读写时对列值做转换（如加解密）

### 集合与映射类型

```java
@Entity
public class ShoppingCart {

    @ElementCollection
    @CollectionId(
        column = @Column(name = "item_id"),
        type = @Type(type = "long"),
        generator = "sequence"
    )
    @CollectionType(type = "com.example.CustomBagType")
    private Collection<CartItem> items;
}
```

### Any 类型引用

```java
@Entity
public class Comment {
    // 多态关联到任意实体
    @Any(metaDef = "PropertyMetaDef", metaColumn = @Column(name = "property_type"))
    @JoinColumn(name = "property_id")
    @AnyMetaDef(idType = "long", metaType = "string",
                metaValues = {
                    @MetaValue(value = "PR", targetEntity = Product.class),
                    @MetaValue(value = "AR", targetEntity = Article.class)
                })
    private Object property;
}
```

### 代理与延迟加载

```java
@Entity
@Proxy(lazy = true, proxyClass = UserProxy.class)
public class User implements UserProxy {
    // lazy=true: 默认延迟加载
    // lazy=false: 不使用代理，立即加载
}
```

### 多态注解

```java
@Entity
@Polymorphism(type = PolymorphismType.EXPLICIT)
public class AuditLog {
    // EXPLICIT: 只在直接查询当前类时返回
    // IMPLICIT（默认）: 查询父类时也返回子类
}
```

### 缓存注解

```java
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "users")
public class User { ... }
```

缓存并发策略：
- `READ_ONLY`：只读缓存，永不修改的数据
- `NONSTRICT_READ_WRITE`：很少修改，允许短暂不一致
- `READ_WRITE`：读多写少，需保证数据一致性（须启用事务）
- `TRANSACTIONAL`：全事务隔离（需 JTA + 事务缓存实现）

### 自然 ID

```java
@Entity
public class User {

    @NaturalId
    @Column(unique = true, nullable = false)
    private String email;

    @Id
    @GeneratedValue
    private Long id;
}

// 查询（绕过一级缓存，直接走二级缓存或数据库）
User user = session.byNaturalId(User.class)
                   .using("email", "user@example.com")
                   .load();
```

### 不可变注解

```java
@Entity
@Immutable
public class Country {
    @Id
    private String code;
    private String name;
    // Hibernate 不会对该实体发出 UPDATE/DELETE
}
```

### @NotFound 注解

```java
@Entity
public class Order {

    @ManyToOne
    @JoinColumn(name = "customer_id")
    @NotFound(action = NotFoundAction.IGNORE)
    private Customer customer;
}
```

- `NotFoundAction.EXCEPTION`（默认）：找不到关联实体时抛异常
- `NotFoundAction.IGNORE`：找不到时设为 null（适合数据一致性较弱的老系统）

## 3.3 注解 vs XML 选择建议

**注解优先的场景**：
- 映射关系稳定，不经常变动
- 团队依赖 IDE 导航和重构功能
- 新项目或中等复杂度项目
- 注解可以与代码放在一起，减少文件个数和上下文切换

**XML 优先的场景**：
- 需要在**不重新编译**时修改映射
- 复杂查询（NamedQuery）用 XML 更易管理
- 不同部署目标需要不同映射配置
- 遗留数据库结构复杂，用 XML 描述更清晰

**混合使用的推荐策略**：
1. 结构映射（表、列、关联）使用注解，贴近代码
2. 查询定义使用 XML（`orm.xml` 中的 `named-native-query` 等），便于 DBA 审查
3. 性能调优参数使用 XML/properties，便于环境切换

---

# 4. 编程式配置方式

## 4.1 Hibernate 原生 Bootstrap（Hibernate 5+）

Hibernate 5 重构了启动流程，将配置类拆分为 `StandardServiceRegistry` 和 `MetadataSources`。这是完全脱离 XML 的方式。

```java
import org.hibernate.boot.MetadataSources;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;
import org.hibernate.boot.Metadata;
import org.hibernate.SessionFactory;

public class HibernateBootstrap {

    public static SessionFactory createSessionFactory() {
        // 1. 构建 ServiceRegistry（负责管理所有 Hibernate 基础服务）
        StandardServiceRegistry standardRegistry = new StandardServiceRegistryBuilder()
            .applySetting("hibernate.connection.driver_class", "org.h2.Driver")
            .applySetting("hibernate.connection.url", "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1")
            .applySetting("hibernate.connection.username", "sa")
            .applySetting("hibernate.connection.password", "")
            .applySetting("hibernate.dialect", "org.hibernate.dialect.H2Dialect")
            // 连接池
            .applySetting("hibernate.hikari.dataSourceClassName", "org.h2.jdbcx.JdbcDataSource")
            .applySetting("hibernate.hikari.maximumPoolSize", "10")
            // SQL
            .applySetting("hibernate.show_sql", "true")
            .applySetting("hibernate.format_sql", "true")
            // 命名策略
            .applySetting("hibernate.physical_naming_strategy",
                "org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl")
            // 缓存
            .applySetting("hibernate.cache.use_second_level_cache", "false")
            .build();

        try {
            // 2. 构建 Metadata（解析实体类生成映射元数据）
            Metadata metadata = new MetadataSources(standardRegistry)
                .addAnnotatedClass(com.example.domain.User.class)
                .addAnnotatedClass(com.example.domain.Order.class)
                .addAnnotatedClass(com.example.domain.Product.class)
                .getMetadataBuilder()
                .applyImplicitNamingStrategy(
                    ImplicitNamingStrategyLegacyJpaImpl.INSTANCE)
                .build();

            // 3. 获取 SessionFactory
            return metadata.getSessionFactoryBuilder()
                .applyInterceptor(new AuditInterceptor())
                .applyStatelessInterceptor(new AuditStatelessInterceptor())
                .build();
        } catch (Exception e) {
            StandardServiceRegistryBuilder.destroy(standardRegistry);
            throw new RuntimeException("Failed to create SessionFactory", e);
        }
    }
}
```

**优势**：
- 类型安全：IDE 自动提示属性名（虽然值仍是字符串）
- 编译时检查：不会因 XML 拼写错误导致运行时失败
- 无需外部文件：适合嵌入式或测试场景
- 灵活条件配置

### 从属性文件加载后覆盖

```java
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
    .loadProperties("hibernate-production.properties") // 基础配置
    .applySetting("hibernate.show_sql", "false")       // 编程式覆盖
    .build();
```

## 4.2 JPA Bootstrap

JPA 标准方法通过 `Persistence.createEntityManagerFactory()` 创建：

```java
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;
import java.util.HashMap;
import java.util.Map;

public class JpaBootstrap {

    public static EntityManagerFactory createEntityManagerFactory() {
        Map<String, Object> properties = new HashMap<>();
        // JPA 属性
        properties.put("jakarta.persistence.jdbc.driver", "org.postgresql.Driver");
        properties.put("jakarta.persistence.jdbc.url", "jdbc:postgresql://localhost:5432/mydb");
        properties.put("jakarta.persistence.jdbc.user", "appuser");
        properties.put("jakarta.persistence.jdbc.password", "p@ssw0rd");
        // Hibernate 属性（前缀 hibernate.）
        properties.put("hibernate.dialect", "org.hibernate.dialect.PostgreSQL10Dialect");
        properties.put("hibernate.show_sql", "true");
        properties.put("hibernate.hbm2ddl.auto", "validate");
        properties.put("hibernate.hikari.maximumPoolSize", "20");

        return Persistence.createEntityManagerFactory("myPU", properties);
    }
}
```

### PersistenceUnitInfo 编程式配置

```java
import org.hibernate.jpa.HibernatePersistenceProvider;
import jakarta.persistence.spi.PersistenceUnitInfo;
import jakarta.persistence.EntityManagerFactory;
import java.util.*;

public class ProgrammaticJpaConfig {

    public static EntityManagerFactory build() {
        // 自定义 PersistenceUnitInfo（JPA 层面最底层配置方式）
        PersistenceUnitInfoImpl pui = new PersistenceUnitInfoImpl(
            "myPU",
            Arrays.asList("com.example.domain.User", "com.example.domain.Order"),
            getDatabaseProperties()
        );

        HibernatePersistenceProvider provider = new HibernatePersistenceProvider();
        return provider.createContainerEntityManagerFactory(pui, pui.getProperties());
    }

    private static Map<String, Object> getDatabaseProperties() {
        Map<String, Object> props = new HashMap<>();
        props.put("hibernate.connection.driver_class", "org.h2.Driver");
        props.put("hibernate.connection.url", "jdbc:h2:mem:test");
        props.put("hibernate.dialect", "org.hibernate.dialect.H2Dialect");
        return props;
    }
}
```

## 4.3 Spring Boot 自动配置

Spring Boot 通过 `HibernateJpaAutoConfiguration` 自动配置 Hibernate。

### 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### application.properties / application.yml 配置

```properties
# ---- JPA / Hibernate 核心配置 ----
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQL10Dialect
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true

# DDL 自动生成
spring.jpa.hibernate.ddl-auto=validate
# 等价于
spring.jpa.properties.hibernate.hbm2ddl.auto=validate

# 连接池（Spring Boot 自动配置 HikariCP）
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=appuser
spring.datasource.password=p@ssw0rd
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000

# ---- Hibernate 扩展属性透传 ----
spring.jpa.properties.hibernate.jdbc.batch_size=30
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.default_batch_fetch_size=16
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.internal.JCacheRegionFactory
spring.jpa.properties.hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
spring.jpa.properties.hibernate.javax.cache.uri=ehcache.xml

# 统计
spring.jpa.properties.hibernate.generate_statistics=false
```

等效 YAML 配置：

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQL10Dialect
    show-sql: true
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        jdbc:
          batch_size: 30
        order_inserts: true
        order_updates: true
        cache:
          use_second_level_cache: true
          use_query_cache: true
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: appuser
    password: p@ssw0rd
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
```

### 自动配置原理

`HibernateJpaAutoConfiguration` 流程：
1. 检测 `DataSource` bean 的存在
2. 解析 `spring.jpa.*` 属性绑定到 `JpaProperties`
3. 创建 `HibernatePropertiesCustomizer` bean（收集所有 `customizer`）
4. 构建 `EntityManagerFactoryBuilder` → `LocalContainerEntityManagerFactoryBean`
5. `hibernate.properties.*` 通过 `spring.jpa.properties.*` 透传

### 自定义 HibernatePropertiesCustomizer

```java
import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer;
import org.springframework.stereotype.Component;
import java.util.Map;

@Component
public class CustomHibernateProperties implements HibernatePropertiesCustomizer {

    @Override
    public void customize(Map<String, Object> hibernateProperties) {
        hibernateProperties.put("hibernate.jdbc.batch_size", "50");
        hibernateProperties.put("hibernate.order_inserts", "true");
        hibernateProperties.put("hibernate.order_updates", "true");
        hibernateProperties.put("hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS", "500");
    }
}
```

### 覆盖自动配置

方式一：通过 `application.properties` 覆盖（最常用）

方式二：通过 `JpaBaseConfiguration` 子类化：

```java
import org.springframework.boot.autoconfigure.orm.jpa.JpaBaseConfiguration;
import org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CustomJpaConfiguration extends JpaBaseConfiguration {

    @Override
    protected void customizeVendorProperties(Map<String, Object> vendorProperties) {
        vendorProperties.put("hibernate.jdbc.batch_size", "50");
        vendorProperties.put("hibernate.generate_statistics", "true");
    }
}
```

方式三：通过 `HibernateJpaAutoConfiguration` 排除再自建：

```java
@SpringBootApplication(exclude = HibernateJpaAutoConfiguration.class)
public class MyApplication {
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource, JpaVendorAdapter jpaVendorAdapter) {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setJpaVendorAdapter(jpaVendorAdapter);
        em.setPackagesToScan("com.example.domain");

        Properties jpaProperties = new Properties();
        jpaProperties.put("hibernate.dialect", "org.hibernate.dialect.PostgreSQL10Dialect");
        jpaProperties.put("hibernate.hbm2ddl.auto", "validate");
        em.setJpaProperties(jpaProperties);

        return em;
    }
}
```

### 多数据源下的 Hibernate 配置

```java
@Configuration
public class MultiDataSourceConfig {

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            DataSource primaryDataSource, JpaVendorAdapter jpaVendorAdapter) {

        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(primaryDataSource);
        em.setJpaVendorAdapter(jpaVendorAdapter);
        em.setPackagesToScan("com.example.domain.primary");
        em.setPersistenceUnitName("primaryPU");

        Map<String, Object> props = new HashMap<>();
        props.put("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        props.put("hibernate.hbm2ddl.auto", "validate");
        props.put("hibernate.show_sql", "false");
        em.setJpaPropertyMap(props);

        return em;
    }

    @Bean(name = "secondaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean secondaryEntityManagerFactory(
            @Qualifier("secondaryDataSource") DataSource secondaryDataSource,
            JpaVendorAdapter jpaVendorAdapter) {

        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(secondaryDataSource);
        em.setJpaVendorAdapter(jpaVendorAdapter);
        em.setPackagesToScan("com.example.domain.secondary");
        em.setPersistenceUnitName("secondaryPU");

        Map<String, Object> props = new HashMap<>();
        props.put("hibernate.dialect", "org.hibernate.dialect.PostgreSQL10Dialect");
        props.put("hibernate.hbm2ddl.auto", "validate");
        em.setJpaPropertyMap(props);

        return em;
    }
}
```

对应的 properties：

```properties
# 主数据源
spring.datasource.primary.url=jdbc:mysql://localhost:3306/primary_db
spring.datasource.primary.username=root
spring.datasource.primary.password=secret

# 次数据源
spring.datasource.secondary.url=jdbc:postgresql://localhost:5432/secondary_db
spring.datasource.secondary.username=appuser
spring.datasource.secondary.password=password
```

---

# 5. 多租户配置

## 5.1 多租户策略

Hibernate 5+ 内置支持三种多租户策略：

| 策略 | 枚举值 | 原理 | 隔离性 |
|------|--------|------|--------|
| DATABASE | `MultiTenancyStrategy.DATABASE` | 每个租户连接不同数据库 | 最强（物理隔离） |
| SCHEMA | `MultiTenancyStrategy.SCHEMA` | 同数据库不同 Schema | 中等 |
| DISCRIMINATOR | `MultiTenancyStrategy.DISCRIMINATOR` | 同表加 tenant_id 列过滤 | 较弱（逻辑隔离） |

## 5.2 多租户配置属性与实现

### 配置属性

```xml
<!-- 多租户策略 -->
<property name="hibernate.multiTenancy">DATABASE</property>
<!-- 多租户连接提供者 -->
<property name="hibernate.multi_tenant_connection_provider">
    com.example.multitenant.SchemaMultiTenantConnectionProvider
</property>
<!-- 租户标识解析器 -->
<property name="hibernate.tenant_identifier_resolver">
    com.example.multitenant.CurrentTenantIdentifierResolverImpl
</property>
```

### CurrentTenantIdentifierResolver 实现

```java
import org.hibernate.context.spi.CurrentTenantIdentifierResolver;
import org.springframework.stereotype.Component;

@Component
public class CurrentTenantIdentifierResolverImpl
        implements CurrentTenantIdentifierResolver {

    private static final String DEFAULT_TENANT = "public";

    @Override
    public String resolveCurrentTenantIdentifier() {
        // 从当前请求上下文获取租户 ID（如 JWT token、请求头等）
        String tenantId = TenantContext.getCurrentTenant();
        return tenantId != null ? tenantId : DEFAULT_TENANT;
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }

    @Override
    public boolean isRoot(Object tenantIdentifier) {
        return DEFAULT_TENANT.equals(tenantIdentifier);
    }
}
```

### MultiTenantConnectionProvider 实现

按 Schema 策略的实现：

```java
import org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider;
import org.hibernate.service.spi.Stoppable;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;

public class SchemaMultiTenantConnectionProvider
        implements MultiTenantConnectionProvider, Stoppable {

    private final DataSource dataSource;

    public SchemaMultiTenantConnectionProvider(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSource.getConnection();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        Connection connection = dataSource.getConnection();
        // 切换到租户对应的 Schema
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("SET SCHEMA '" + tenantIdentifier + "'");
        }
        return connection;
    }

    @Override
    public void releaseConnection(String tenantIdentifier, Connection connection)
            throws SQLException {
        // 切换回默认 Schema
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("SET SCHEMA 'public'");
        }
        connection.close();
    }

    @Override
    public boolean supportsAggressiveRelease() {
        return true;
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return false;
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        return null;
    }

    @Override
    public void stop() {
        // 清理资源（因实现 Stoppable 接口）
    }
}
```

按 DATABASE 策略的实现：

```java
public class DatabaseMultiTenantConnectionProvider
        implements MultiTenantConnectionProvider, Stoppable {

    private final Map<String, DataSource> tenantDataSources = new HashMap<>();

    public DatabaseMultiTenantConnectionProvider() {
        tenantDataSources.put("tenant1", createDataSource("jdbc:mysql://host1:3306/tenant1_db"));
        tenantDataSources.put("tenant2", createDataSource("jdbc:mysql://host2:3306/tenant2_db"));
    }

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        DataSource ds = tenantDataSources.get(tenantIdentifier);
        if (ds == null) {
            throw new SQLException("Unknown tenant: " + tenantIdentifier);
        }
        return ds.getConnection();
    }

    @Override
    public Connection getAnyConnection() throws SQLException {
        return tenantDataSources.values().iterator().next().getConnection();
    }

    @Override
    public void releaseConnection(String tenantIdentifier, Connection connection)
            throws SQLException {
        connection.close();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public boolean supportsAggressiveRelease() {
        return true;
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return false;
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        return null;
    }

    @Override
    public void stop() {
        tenantDataSources.forEach((tenant, ds) -> {
            if (ds instanceof Closeable) {
                try { ((Closeable) ds).close(); } catch (Exception ignored) {}
            }
        });
    }

    private DataSource createDataSource(String url) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername("root");
        config.setPassword("secret");
        return new HikariDataSource(config);
    }
}
```

### Spring Boot 中的多租户集成

```java
@Configuration
public class MultiTenancyConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public MultiTenantConnectionProvider multiTenantConnectionProvider() {
        return new SchemaMultiTenantConnectionProvider(dataSource);
    }

    @Bean
    public CurrentTenantIdentifierResolver currentTenantIdentifierResolver() {
        return new CurrentTenantIdentifierResolverImpl();
    }

    @Bean
    public HibernatePropertiesCustomizer multiTenancyCustomizer(
            MultiTenantConnectionProvider connectionProvider,
            CurrentTenantIdentifierResolver tenantResolver) {

        return hibernateProperties -> {
            hibernateProperties.put("hibernate.multiTenancy", "SCHEMA");
            hibernateProperties.put("hibernate.multi_tenant_connection_provider", connectionProvider);
            hibernateProperties.put("hibernate.tenant_identifier_resolver", tenantResolver);
        };
    }
}
```

---

# 6. 配置最佳实践

## 6.1 配置分离原则

按环境划分配置文件：

```
resources/
├── application.properties           # 公共 + 开发默认配置
├── application-dev.properties       # 开发环境
├── application-test.properties      # 测试环境
├── application-staging.properties   # 预发布环境
└── application-prod.properties      # 生产环境
```

各环境差异化配置：

```properties
# application-dev.properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.highlight_sql=true
spring.jpa.hibernate.ddl-auto=update

# application-prod.properties
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.generate_statistics=true
```

## 6.2 敏感信息管理

**禁止在配置文件中明文写密码**，推荐方案：

### 方式 1：Jasypt 加密

```properties
# application.properties
spring.datasource.username=ENC(encrypted_username)
spring.datasource.password=ENC(encrypted_password)
# 密钥通过环境变量注入
jasypt.encryptor.password=${JASYPT_ENCRYPTOR_PASSWORD}
```

### 方式 2：环境变量

```properties
spring.datasource.password=${DB_PASSWORD}
```

### 方式 3：外部密钥管理（HashiCorp Vault）

```properties
spring.config.import=vault://
spring.cloud.vault.token=${VAULT_TOKEN}
```

## 6.3 配置验证（Fail-Fast）

在启动时验证关键配置：

```java
@Component
public class HibernateConfigValidator {

    @EventListener(ApplicationReadyEvent.class)
    public void validateHibernateConfig(EntityManagerFactory emf) {
        SessionFactory sf = emf.unwrap(SessionFactory.class);

        // 验证 DDL 模式
        String ddlMode = sf.getProperties().getProperty("hibernate.hbm2ddl.auto");
        if ("create".equals(ddlMode) || "create-drop".equals(ddlMode)) {
            throw new IllegalStateException(
                "hibernate.hbm2ddl.auto=" + ddlMode + " is not allowed in production!");
        }

        // 验证 SQL 日志在生产环境已关闭
        String showSql = sf.getProperties().getProperty("hibernate.show_sql");
        if ("true".equalsIgnoreCase(showSql)) {
            throw new IllegalStateException(
                "hibernate.show_sql must be false in production!");
        }
    }
}
```

## 6.4 推荐配置组合

### 开发环境（功能导向）

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.highlight_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true
spring.jpa.properties.hibernate.hbm2ddl.auto=update
spring.jpa.properties.hibernate.generate_statistics=false
spring.jpa.properties.hibernate.jdbc.batch_size=10
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### 测试环境（快速 + 可靠）

```properties
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.hbm2ddl.auto=create-drop
spring.jpa.properties.hibernate.generate_statistics=false
spring.jpa.properties.hibernate.jdbc.batch_size=0
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
spring.datasource.driver-class-name=org.h2.Driver
```

### 生产环境（性能优先）

```properties
# SQL 不输出
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false

# Schema 仅验证
spring.jpa.properties.hibernate.hbm2ddl.auto=validate

# 批处理优化 —— 核心性能配置
spring.jpa.properties.hibernate.jdbc.batch_size=30
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true
spring.jpa.properties.hibernate.default_batch_fetch_size=16

# 连接池
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.leak-detection-threshold=2000

# 二级缓存
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.internal.JCacheRegionFactory

# 统计与监控
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=1000

# 查询优化
spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch=true
spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true

# 其他
spring.jpa.properties.hibernate.enable_lazy_load_no_trans=false
spring.jpa.properties.hibernate.discriminator.ignore_explicit_for_joined=true
spring.jpa.properties.hibernate.max_fetch_depth=3
```

### 配置排查清单

配置出问题时按以下顺序排查：

1. **属性名是否正确**：Hibernate 属性名区分大小写，必须是全小写加下划线格式
2. **配置文件是否被加载**：确认 `hibernate.cfg.xml` / `hibernate.properties` 在 classpath 根目录
3. **优先级覆盖**：检查是否有 properties 文件覆盖了编程式设置
4. **Spring Boot 属性透传**：`spring.jpa.properties.hibernate.xxx` 是否正确（注意中间的 `.properties` 而不是直接 `spring.jpa.hibernate.xxx`）
5. **驱动兼容性**：Dialect 版本与数据库版本是否匹配
6. **日志分类器级别**：确认 logback/log4j2 配置的日志级别正确
