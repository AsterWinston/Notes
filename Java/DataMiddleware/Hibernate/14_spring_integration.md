# 1. Spring 与 Hibernate 集成概述

## 1.1 集成演进历史

Spring 与 Hibernate 的集成经历了三个主要阶段，每个阶段反映了框架设计哲学和 Java 生态的演进：

| 阶段 | Spring 版本 | 集成方式 | 核心特征 |
|------|------------|---------|---------|
| **模板阶段** | Spring 2.x | `HibernateTemplate` / `HibernateDaoSupport` | 模板方法模式封装 Session 管理，侵入式 |
| **原生阶段** | Spring 3.x | 原生 `SessionFactory` + `HibernateTransactionManager` | 不再强制使用模板，声明式事务管理成熟 |
| **自动配置阶段** | Spring 4.x+ / Spring Boot | `spring-boot-starter-data-jpa` 自动配置 | 零 XML 配置，约定优于配置，JPA 标准化 |

### 1.1.1 Spring 2.x 时代：HibernateTemplate / HibernateDaoSupport

```java
// Spring 2.x 风格的 DAO（已过时）
@Repository
public class UserDao extends HibernateDaoSupport {

    public User findById(Long id) {
        return (User) getHibernateTemplate().get(User.class, id);
    }

    public void save(User user) {
        getHibernateTemplate().save(user);
    }

    public List<User> findByName(String name) {
        return getHibernateTemplate().find(
            "from User where name = ?", name);
    }
}
```

`HibernateTemplate` 的设计目标是简化 Hibernate 操作：自动打开/关闭 Session、将受检异常 `HibernateException` 转换为 Spring 的 `DataAccessException`（非受检）。但这一模式存在明显问题：
- DAO 与 Spring 框架耦合，单元测试困难
- 模板方法掩盖了底层 Hibernate 行为，不利于精细化控制
- `HibernateCallback` 回调代码冗长，可读性差

### 1.1.2 Spring 3.x 时代：原生 SessionFactory 集成

Spring 3.x 开始鼓励直接使用原生 `SessionFactory`，通过声明式事务管理处理 Session 生命周期：

```java
// Spring 3.x 风格的 DAO（推荐方式）
@Repository
public class UserDao {

    @Autowired
    private SessionFactory sessionFactory;

    public User findById(Long id) {
        return sessionFactory.getCurrentSession().get(User.class, id);
    }

    public void save(User user) {
        sessionFactory.getCurrentSession().persist(user);
    }
}
```

核心改进：
- `HibernateTemplate` 被标记为 `@Deprecated`（但未移除）
- 声明式事务管理（`@Transactional`）接管 Session 创建与关闭
- `SpringSessionContext` 实现了 `CurrentSessionContext` 接口，使 `getCurrentSession()` 在 Spring 中可用

### 1.1.3 Spring 4.x+ / Spring Boot 时代：自动配置

```java
// Spring Boot 应用只需添加依赖和少量配置
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Spring Boot 自动完成：
// 1. 创建 DataSource Bean
// 2. 创建 LocalContainerEntityManagerFactoryBean Bean
// 3. 创建 JpaTransactionManager Bean
// 4. 注册 OpenEntityManagerInViewInterceptor（OSIV）
// 5. 扫描 @Entity 类
// 6. 注册 @Repository 异常转换
```

三种集成方式的对比：

| 对比维度 | HibernateTemplate | 原生 SessionFactory | Spring Boot 自动配置 |
|---------|-------------------|-------------------|---------------------|
| 代码侵入性 | 高（继承/依赖 Spring API） | 低（仅依赖 Hibernate API） | 极低（仅需配置） |
| 配置复杂度 | 中 | 高（手动装配 Bean） | 极低（零配置或少量 yml） |
| Session 管理 | 隐式（模板内部处理） | 显式（事务边界控制） | 隐式（OSIV + 事务） |
| 测试友好度 | 低 | 高 | 高（@DataJpaTest 支持） |
| JPA 兼容性 | 仅 Hibernate | 仅 Hibernate | JPA 标准，可切换实现 |
| 异常处理 | 自动转换 | 需 @Repository 注解 | 自动转换（Spring Boot 自动注册） |

## 1.2 核心依赖

一个完整的 Spring Boot + Hibernate + MySQL 项目依赖配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>hibernate-spring-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Data JPA Starter（核心依赖） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Spring Web（用于 REST API 演示） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 验证 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- H2 数据库（用于单元测试） -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**依赖传递关系说明**：

- `spring-boot-starter-data-jpa` 内部自动引入：
  - `hibernate-core`（Hibernate ORM 核心，基于 JPA 规范实现）
  - `spring-orm`（Spring 对 ORM 框架的支持，`LocalContainerEntityManagerFactoryBean` 等）
  - `spring-tx`（Spring 事务管理，`@Transactional` 注解支持）
  - `spring-data-jpa`（Spring Data JPA 抽象，`JpaRepository` 接口等）
  - `spring-boot-starter-jdbc`（JDBC 支持，自动配置连接池 HikariCP）
  - `jakarta.persistence-api`（JPA 规范接口）

- **连接池**：Spring Boot 2.x+ 默认使用 **HikariCP** 作为连接池。若想使用其他连接池（如 Druid、Tomcat CP），需要排除默认依赖并显式声明：

```xml
<!-- 使用 Druid 连接池替代 HikariCP -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-3-starter</artifactId>
    <version>1.2.23</version>
</dependency>
```

---

# 2. Spring Boot 自动配置原理

## 2.1 HibernateJpaAutoConfiguration

Spring Boot 对 JPA / Hibernate 的自动配置由 `HibernateJpaAutoConfiguration` 类负责，其定义位于 `org.springframework.boot.autoconfigure.orm.jpa` 包下。

### 2.1.1 自动配置的触发条件

```java
@AutoConfiguration
@ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class, EntityManager.class })
@EnableConfigurationProperties(JpaProperties.class)
@Import(HibernateJpaConfiguration.class)
public class HibernateJpaAutoConfiguration {

}
```

触发自动配置的四个条件：

| 条件 | 说明 |
|------|------|
| `@ConditionalOnClass(LocalContainerEntityManagerFactoryBean.class)` | classpath 中存在 `spring-orm` |
| `@ConditionalOnClass(EntityManager.class)` | classpath 中存在 `jakarta.persistence-api` |
| classpath 中存在 Hibernate（`HibernateJpaVendorAdapter`） | 启用 Hibernate 特定配置 |
| 未自定义 `LocalContainerEntityManagerFactoryBean` Bean | 用户未手动声明同名 Bean |

**类继承关系**：

```
HibernateJpaAutoConfiguration
    └── @Import → HibernateJpaConfiguration
            └── extends JpaBaseConfiguration（自动配置的核心抽象类）
                    ├── 定义 EntityManagerFactory 的构建流程
                    ├── 定义 TransactionManager 的创建
                    └── 抽象方法 createJpaVendorAdapter() → HibernateJpaVendorAdapter
```

### 2.1.2 JpaBaseConfiguration 的核心作用

`JpaBaseConfiguration` 是自动配置的真正核心，它使用 `@ConditionalOnMissingBean` 保护用户自定义 Bean 不被覆盖：

```java
public abstract class JpaBaseConfiguration {

    // 核心 Bean 1：LocalContainerEntityManagerFactoryBean
    @Bean
    @ConditionalOnMissingBean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder factoryBuilder) {
        Map<String, Object> vendorProperties = getVendorProperties();
        customizeVendorProperties(vendorProperties);
        return factoryBuilder.dataSource(dataSource)
                .packages(getPackagesToScan())
                .properties(vendorProperties)
                .jta(isJta())
                .build();
    }

    // 核心 Bean 2：JpaTransactionManager
    @Bean
    @ConditionalOnMissingBean
    public PlatformTransactionManager transactionManager(
            AbstractEntityManagerFactoryBean entityManagerFactory) {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(
            entityManagerFactory.getObject());
        return transactionManager;
    }

    // 核心 Bean 3：EntityManagerFactoryBuilder
    @Bean
    @ConditionalOnMissingBean
    public EntityManagerFactoryBuilder entityManagerFactoryBuilder(
            JpaVendorAdapter jpaVendorAdapter,
            ObjectProvider<PersistenceUnitManager> persistenceUnitManager) {
        EntityManagerFactoryBuilder builder = new EntityManagerFactoryBuilder(
                jpaVendorAdapter, getVendorProperties(),
                persistenceUnitManager.getIfAvailable());
        return builder;
    }

    // 抽象方法，由子类提供具体 JPA 实现的 VendorAdapter
    protected abstract AbstractJpaVendorAdapter createJpaVendorAdapter();

    protected abstract Map<String, Object> getVendorProperties();
}
```

## 2.2 自动配置的 Bean

Spring Boot 自动配置启动后，ApplicationContext 中包含以下关键 Bean：

| Bean 类型 | Bean 名称 | 说明 |
|-----------|----------|------|
| `javax.sql.DataSource` | `dataSource` | 数据源，自动根据 `spring.datasource.*` 创建（默认 HikariCP） |
| `LocalContainerEntityManagerFactoryBean` | `entityManagerFactory` | JPA EntityManagerFactory 的 Spring FactoryBean |
| `JpaTransactionManager` | `transactionManager` | JPA 事务管理器（自动偏好，回退到 `DataSourceTransactionManager`） |
| `HibernateJpaVendorAdapter` | (内部 Bean) | Hibernate 的 JPA 适配器，设置 Dialect、showSql 等 |
| `JpaVendorAdapter` | `jpaVendorAdapter` | JPA 适配器接口（通过 `HibernateJpaVendorAdapter` 实现） |
| `JpaProperties` | `spring.jpa-org.springframework.boot.autoconfigure.orm.jpa.JpaProperties` | yml 配置映射类 |

### 2.2.1 EntityManagerFactory → SessionFactory 的转换

`LocalContainerEntityManagerFactoryBean` 内部持有 Hibernate 的 `SessionFactoryImpl`（即 `EntityManagerFactory`）：

```java
@Service
public class UserService {

    @Autowired
    private EntityManagerFactory emf;

    public void doSomething() {
        // 方法一：通过 EntityManagerFactory.unwrap() 获取 SessionFactory
        SessionFactory sessionFactory = emf.unwrap(SessionFactory.class);

        // 方法二：直接注入 SessionFactory（Spring Boot 自动可用）
        // @Autowired private SessionFactory sessionFactory;

        // 方法三：通过 EntityManager 获取
        // EntityManager em = emf.createEntityManager();
        // Session session = em.unwrap(Session.class);
    }
}
```

Spring Boot 使得 `SessionFactory` 可以直接 `@Autowired` 注入（因为 `EntityManagerFactory` 的 `unwrap` 在注入时被自动处理，或者通过 `HibernateJpaVendorAdapter` 的类型匹配）。

## 2.3 自动配置的扫描机制

### 2.3.1 @EntityScan 注解

Spring Boot 默认扫描 `@SpringBootApplication` 所在包及其所有子包下的 `@Entity` 类。若实体类不在主类所在包或子包下，需使用 `@EntityScan` 显式指定：

```java
@SpringBootApplication
@EntityScan(basePackages = {
    "com.example.project.domain",
    "com.example.common.entities"
})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2.3.2 自定义扫描路径

```java
@Configuration
public class JpaScanConfig {

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter) {

        LocalContainerEntityManagerFactoryBean factory =
            new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setJpaVendorAdapter(jpaVendorAdapter);

        // 方式一：手动指定扫描包（会覆盖默认扫描）
        factory.setPackagesToScan(
            "com.example.domain"
        );

        // 方式二：设置持久化单元名称（需要 persistence.xml）
        // factory.setPersistenceUnitName("myPU");

        return factory;
    }
}
```

### 2.3.3 排除特定实体

```java
// 使用 TypeExcludeFilter 排除带有特定注解的实体
@EntityScan(
    basePackages = "com.example.domain",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = ExcludeFromScan.class
    )
)
```

需要注意的是，`@EntityScan` 不区分 `includeFilters` 和 `excludeFilters` 的语义，所有 filter 均用于排除。

---

# 3. 配置详解

## 3.1 application.yml 配置项完整指南

以下是一份包含所有常用 JPA / Hibernate 配置项的完整 `application.yml`：

```yaml
spring:
  # ==================== 数据源配置 ====================
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf-8
    username: root
    password: ${DB_PASSWORD:root}          # 支持环境变量覆盖，默认 root
    driver-class-name: com.mysql.cj.jdbc.Driver

  # ==================== JPA 配置 ====================
  jpa:
    # --- 数据库平台 ---
    database: MYSQL                        # 提示：MYSQL, POSTGRESQL, ORACLE, H2, HSQL 等
    database-platform: org.hibernate.dialect.MySQLDialect

    # --- SQL 日志 ---
    show-sql: true                         # 控制台输出 SQL
    format-sql: true                       # 格式化 SQL 输出

    # --- DDL 策略 ---
    hibernate:
      ddl-auto: validate                   # none | validate | update | create | create-drop
      naming:
        # 物理命名策略（影响最终表名/列名）
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
        # 隐式命名策略（影响未显式命名的列/表）
        implicit-strategy: org.hibernate.boot.model.naming.ImplicitNamingStrategyJpaCompliantImpl

    # --- Open Session In View ---
    open-in-view: false                    # 生产环境建议关闭

    # --- 延迟数据源初始化 ---
    defer-datasource-initialization: true  # 延迟到 EntityManagerFactory 初始化后再初始化数据源

    # --- 自定义 orm.xml 映射文件 ---
    # mapping-resources:
    #   - META-INF/orm/User.xml
    #   - META-INF/orm/Order.xml

    # --- 原生 Hibernate 属性透传 ---
    properties:
      hibernate:
        # 连接
        connection:
          autocommit: false
        # JDBC 批处理
        jdbc:
          batch_size: 20
          batch_versioned_data: true
        # 批量抓取
        default_batch_fetch_size: 16
        # 排序
        order_inserts: true
        order_updates: true
        # SQL 生成
        use_sql_comments: true
        generate_statistics: false
        # 多租户
        # multi_tenant_connection_provider: com.example.MultiTenantProvider
        # 二级缓存
        cache:
          use_second_level_cache: false
          use_query_cache: false
          region:
            factory_class: org.hibernate.cache.jcache.internal.JCacheRegionFactory
        # 查询
        query:
          fail_on_pagination_over_collection_fetch: true
          in_clause_parameter_padding: true
```

### 3.1.1 DDL 策略详解

| 值 | 行为 | 适用环境 |
|---|------|---------|
| `none` | 不做任何操作 | **生产环境**（配合 Flyway/Liquibase 管理 Schema） |
| `validate` | 仅验证表结构是否与实体映射匹配，不修改 | **生产环境**（安全校验，不匹配则启动失败） |
| `update` | 根据实体变更自动更新表结构（不会删除列） | **开发环境** |
| `create` | 每次启动删除旧表并重建新表 | **开发 / 演示** |
| `create-drop` | 启动时建表，关闭时删表 | **集成测试** |

> **警告**：`update` 策略不会删除数据库中已存在但实体中已移除的列，也不会重命名列。长期使用会导致数据库 Schema 累积垃圾字段。生产环境严禁使用。

### 3.1.2 物理命名策略 vs 隐式命名策略

```
Java 实体属性名
    │
    ▼
┌─────────────────────────────┐
│ 隐式命名策略                 │  ← 将属性名转换为逻辑名
│ ImplicitNamingStrategy      │     userFirstName → user_first_name
└───────────────┬─────────────┘
                │ 逻辑名
                ▼
┌─────────────────────────────┐
│ 物理命名策略                 │  ← 将逻辑名转换为物理名
│ PhysicalNamingStrategy      │     user_first_name → USER_FIRST_NAME
└───────────────┬─────────────┘
                │ 物理名（最终数据库中的名称）
                ▼
        数据库列名
```

```java
// 常用物理命名策略

// 1. Spring 默认（SpringPhysicalNamingStrategy）
// userFirstName → user_first_name（驼峰转下划线，小写）

// 2. Hibernate 标准（PhysicalNamingStrategyStandardImpl）
// userFirstName → userFirstName（原样保留）

// 3. 全大写
public class UpperCaseNamingStrategy extends PhysicalNamingStrategyStandardImpl {
    @Override
    public Identifier toPhysicalTableName(Identifier name, JdbcEnvironment context) {
        return Identifier.toIdentifier(name.getText().toUpperCase());
    }
}
```

## 3.2 自定义 Hibernate 属性

所有 `spring.jpa.properties.hibernate.*` 下的配置都会透传给 Hibernate 的 `Configuration` 对象，覆盖其默认属性。以下为常用自定义属性：

```yaml
spring:
  jpa:
    properties:
      hibernate:
        # ====== SQL 相关 ======
        dialect: org.hibernate.dialect.MySQL8Dialect
        show_sql: false
        format_sql: true
        use_sql_comments: true              # 在 SQL 中添加 HQL/Criteria 注释
        highlight_sql: true                 # SQL 语法高亮（Hibernate 6.x+）

        # ====== JDBC 批处理 ======
        jdbc:
          batch_size: 30                    # JDBC 批处理大小
          batch_versioned_data: true        # 批量更新包含版本号的实体
          time_zone: Asia/Shanghai          # JDBC 时区
        order_inserts: true                 # 排序 INSERT 以提高批处理效率
        order_updates: true                 # 排序 UPDATE

        # ====== 抓取策略 ======
        default_batch_fetch_size: 16        # 批量懒加载的默认批次大小
        max_fetch_depth: 3                  # 外连接抓取的最大深度

        # ====== 二级缓存 ======
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.internal.JCacheRegionFactory
          use_structured_entries: true
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
            missing_cache_strategy: fail

        # ====== 连接 ======
        connection:
          autocommit: false
          provider_disables_autocommit: true  # HikariCP 级别已控制 autocommit
          pool_size: 20                       # Hibernate 自带连接池（不推荐，建议用 HikariCP）

        # ====== 查询 ======
        query:
          plan_cache_max_size: 2048           # 查询计划缓存上限
          plan_parameter_metadata_max_size: 128
          fail_on_pagination_over_collection_fetch: true
          in_clause_parameter_padding: true   # 填充 IN 子句参数到 2 的幂

        # ====== 统计 ======
        generate_statistics: false

        # ====== Session 上下文 ======
        current_session_context_class: org.springframework.orm.hibernate5.SpringSessionContext

        # ====== Schema 相关 ======
        hbm2ddl:
          auto: none
          create_namespaces: true             # 自动创建 Schema

        # ====== 多租户 ======
        # multiTenancy: DATABASE
        # multi_tenant_connection_provider: com.example.ConfigurableMultiTenantConnectionProvider
```

## 3.3 HikariCP 连接池配置

HikariCP 是 Spring Boot 2.x+ 的默认连接池，"光速"连接池，以极致性能著称。完整配置如下：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver

    hikari:
      # ====== 连接池大小 ======
      maximum-pool-size: 20               # 最大连接数（含空闲和活跃），默认 10
      minimum-idle: 5                     # 最小空闲连接数，默认等于 maximum-pool-size

      # ====== 超时设置（单位：ms） ======
      connection-timeout: 30000           # 等待连接的最大时间，默认 30000
      idle-timeout: 600000               # 空闲连接最大存活时间，默认 600000（10 分钟）
      max-lifetime: 1800000              # 连接最大生命周期，默认 1800000（30 分钟）
      keepalive-time: 0                  # 保活检查间隔（0 表示不启用）
      validation-timeout: 5000           # 连接验证超时，默认 5000

      # ====== 连接泄漏检测 ======
      leak-detection-threshold: 60000    # 连接泄漏检测阈值（建议 < max-lifetime），0 = 禁用

      # ====== 连接验证 ======
      connection-test-query: SELECT 1    # 连接有效性验证 SQL（MySQL 可不设置，JDBC4 自动检测）

      # ====== 其他 ======
      auto-commit: false                 # 自动提交（JPA 场景下建议 false）
      pool-name: HikariPool-MyDB         # 连接池名称（便于监控）
      read-only: false                   # 只读模式
      connection-init-sql: "SET NAMES utf8mb4"  # 连接初始化 SQL
      allow-pool-suspension: false       # 允许暂停连接池

      # ====== 数据源属性（JDBC URL 参数之外的属性） ======
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        useLocalSessionState: true
        rewriteBatchedStatements: true
        cacheResultSetMetadata: true
        cacheServerConfiguration: true
        elideSetAutoCommits: true
        maintainTimeStats: false
```

### 3.3.1 连接池大小计算公式

```
数据库连接数 = ((核心数 * 2) + 硬盘数)

简化公式（源自 HikariCP 官方 wiki）：
connections = ((core_count * 2) + effective_spindle_count)

示例：
- 8 核 CPU + SSD：16 连接
- 4 核 CPU + HDD：9 连接
```

实际情况中应考虑微服务数量与数据库最大连接数限制的协调。

### 3.3.2 连接泄漏检测

```java
// 当配置 leak-detection-threshold 后，对于超过该时间的连接将输出警告日志
// 日志示例：
// [HikariPool-1 housekeeper] WARN  com.zaxxer.hikari.pool.ProxyLeakTask
// Connection leak detection triggered for conn1 on thread http-nio-8080-exec-3,
// stack trace follows...
```

**调试连接泄漏的典型场景**：忘记关闭手动创建的 `EntityManager`、事务边界设置过宽、OSIV 导致的长时间占用。

---

# 4. 编程式配置（高级定制）

## 4.1 JpaBaseConfiguration 子类化

当需要覆盖自动配置的行为时，可以通过继承 `HibernateJpaConfiguration` 并重写方法来定制：

```java
@Configuration
public class CustomJpaConfiguration extends HibernateJpaConfiguration {

    public CustomJpaConfiguration(DataSource dataSource,
                                   JpaProperties properties,
                                   ObjectProvider<JtaTransactionManager> jtaTransactionManager) {
        super(dataSource, properties, jtaTransactionManager);
    }

    // 重写：返回自定义的 JPA VendorAdapter
    @Override
    protected AbstractJpaVendorAdapter createJpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setDatabase(Database.MYSQL);
        adapter.setShowSql(false);
        adapter.setGenerateDdl(false);
        adapter.setDatabasePlatform("org.hibernate.dialect.MySQL8Dialect");
        return adapter;
    }

    // 重写：供应商属性（透传至 persistence.xml 的 <properties>）
    @Override
    protected Map<String, Object> getVendorProperties() {
        Map<String, Object> props = new HashMap<>(super.getVendorProperties());
        props.put("hibernate.jdbc.batch_size", "50");
        props.put("hibernate.order_inserts", "true");
        props.put("hibernate.order_updates", "true");
        return props;
    }
}
```

> **注意**：直接继承 `HibernateJpaConfiguration` 会完全接管自动配置。通常建议使用更轻量的 `EntityManagerFactoryBuilderCustomizer` 或 `HibernatePropertiesCustomizer`。

## 4.2 EntityManagerFactoryBuilderCustomizer

`EntityManagerFactoryBuilderCustomizer` 是 Spring Boot 提供的扩展钩子，用于在 `EntityManagerFactory` 构建前自定义其配置，非常适合注册 Hibernate 拦截器、自定义类型、命名策略等：

```java
@Configuration
public class HibernateCustomizerConfig {

    /**
     * 注册 Hibernate 拦截器
     */
    @Bean
    public EntityManagerFactoryBuilderCustomizer registerInterceptor(
            MyAuditInterceptor auditInterceptor) {
        return builder -> {
            builder.persistenceUnitProperties(
                Map.of(
                    "hibernate.session_factory.interceptor",
                    auditInterceptor
                )
            );
        };
    }

    /**
     * 注册自定义命名策略
     */
    @Bean
    public EntityManagerFactoryBuilderCustomizer registerNamingStrategy() {
        return builder -> {
            builder.persistenceUnitProperties(
                Map.of(
                    "hibernate.physical_naming_strategy",
                    "com.example.naming.CustomPhysicalNamingStrategy"
                )
            );
        };
    }

    /**
     * 注册自定义类型映射
     */
    @Bean
    public EntityManagerFactoryBuilderCustomizer registerCustomType() {
        return builder -> {
            builder.persistenceUnitProperties(
                Map.of(
                    "hibernate.type_contributors",
                    "com.example.type.MoneyTypeContributor"
                )
            );
        };
    }
}
```

### 4.2.1 自定义 Hibernate 拦截器

```java
@Component
public class MyAuditInterceptor extends EmptyInterceptor {

    @Override
    public boolean onSave(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        if (entity instanceof Auditable) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("createdAt".equals(propertyNames[i]) && state[i] == null) {
                    state[i] = LocalDateTime.now();
                }
            }
        }
        return true;
    }

    @Override
    public boolean onFlushDirty(Object entity, Serializable id,
                                 Object[] currentState, Object[] previousState,
                                 String[] propertyNames, Type[] types) {
        if (entity instanceof Auditable) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("updatedAt".equals(propertyNames[i])) {
                    currentState[i] = LocalDateTime.now();
                }
            }
        }
        return true;
    }
}
```

## 4.3 HibernatePropertiesCustomizer

`HibernatePropertiesCustomizer` 允许动态修改 JPA 属性（在自动配置读取 yml 配置后、`EntityManagerFactory` 创建前执行）：

```java
@Configuration
public class ProfileBasedHibernateConfig {

    @Bean
    public HibernatePropertiesCustomizer devProfileCustomizer(
            @Value("${spring.profiles.active:}") String activeProfile) {

        return hibernateProperties -> {
            if ("dev".equals(activeProfile)) {
                // 开发环境：开启 SQL 输出和统计
                hibernateProperties.put(
                    "hibernate.show_sql", "true");
                hibernateProperties.put(
                    "hibernate.format_sql", "true");
                hibernateProperties.put(
                    "hibernate.generate_statistics", "true");
                hibernateProperties.put(
                    "hibernate.hbm2ddl.auto", "update");
            } else {
                // 生产环境：严格模式
                hibernateProperties.put(
                    "hibernate.show_sql", "false");
                hibernateProperties.put(
                    "hibernate.generate_statistics", "false");
                hibernateProperties.put(
                    "hibernate.hbm2ddl.auto", "none");
            }
        };
    }
}
```

## 4.4 完整自定义配置示例

以下是一个结合命名策略、拦截器、类型注册的完整配置类：

```java
@Configuration
@RequiredArgsConstructor
public class HibernateFullCustomConfig {

    private final DataSource dataSource;

    // 自定义物理命名策略：表名前缀 + 驼峰转下划线
    @Bean
    public EntityManagerFactoryBuilderCustomizer namingStrategyCustomizer() {
        return builder -> builder.persistenceUnitProperties(
            Map.of(
                "hibernate.physical_naming_strategy",
                TablePrefixNamingStrategy.class.getName(),
                "hibernate.implicit_naming_strategy",
                ImplicitNamingStrategyJpaCompliantImpl.class.getName()
            )
        );
    }

    // 注册审计拦截器
    @Bean
    public EntityManagerFactoryBuilderCustomizer interceptorCustomizer(
            AuditInterceptor auditInterceptor) {
        return builder -> builder.persistenceUnitProperties(
            Map.of("hibernate.session_factory.interceptor", auditInterceptor)
        );
    }

    // 注册自定义 JSON 类型
    @Bean
    public EntityManagerFactoryBuilderCustomizer typeContributorCustomizer() {
        return builder -> builder.persistenceUnitProperties(
            Map.of("hibernate.type_contributors",
                   JsonTypeContributor.class.getName())
        );
    }

    // 自定义表名前缀命名策略
    public static class TablePrefixNamingStrategy
            extends PhysicalNamingStrategyStandardImpl {

        private static final String TABLE_PREFIX = "t_";

        @Override
        public Identifier toPhysicalTableName(Identifier name,
                                               JdbcEnvironment context) {
            String newName = TABLE_PREFIX + name.getText();
            return Identifier.toIdentifier(newName);
        }

        @Override
        public Identifier toPhysicalColumnName(Identifier name,
                                                JdbcEnvironment context) {
            return new Identifier(
                CaseFormat.LOWER_CAMEL.to(CaseFormat.LOWER_UNDERSCORE, name.getText()),
                name.isQuoted()
            );
        }
    }

    // 多数据源的主数据源配置（手动声明，覆盖默认自动配置）
    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("primaryDataSource") DataSource dataSource) {

        return builder
                .dataSource(dataSource)
                .packages("com.example.primary.entity")
                .persistenceUnit("primary")
                .properties(getHibernateProperties())
                .build();
    }

    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManagerFactory")
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }

    private Map<String, Object> getHibernateProperties() {
        Map<String, Object> props = new HashMap<>();
        props.put("hibernate.dialect", MySQL8Dialect.class.getName());
        props.put("hibernate.jdbc.batch_size", "30");
        props.put("hibernate.order_inserts", "true");
        props.put("hibernate.order_updates", "true");
        return Collections.unmodifiableMap(props);
    }
}
```

---

# 5. 事务管理集成

## 5.1 @Transactional 与 Hibernate

Spring Boot 自动配置 `JpaTransactionManager`，与 `@Transactional` 无缝集成：

```java
@Service
@Transactional(readOnly = true)  // 类级别默认只读
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    // 继承类级别设置：readOnly = true
    public List<Order> findAll() {
        return orderRepository.findAll();
    }

    // 覆盖为读写事务
    @Transactional
    public Order createOrder(Order order) {
        return orderRepository.save(order);
    }

    // 新事务：挂起当前事务
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditLog(String message) {
        // 独立事务，不受外部事务回滚影响
    }

    // 嵌套事务（需数据库支持 SavePoint）
    @Transactional(propagation = Propagation.NESTED)
    public void processOrder(Order order) {
        try {
            orderRepository.save(order);
        } catch (Exception e) {
            // 仅回滚到 savepoint，外部事务继续
        }
    }

    // 事务超时 5 秒
    @Transactional(timeout = 5)
    public void batchProcess() {
        // 超过 5 秒自动回滚
    }

    // 指定回滚异常
    @Transactional(
        rollbackFor = BusinessException.class,
        noRollbackFor = ValidationException.class
    )
    public void complexOperation() throws BusinessException {
        // BusinessException 触发回滚
        // ValidationException 不触发回滚
    }
}
```

### 5.1.1 事务传播行为对照表

| 传播行为 | Hibernate 行为 | 典型场景 |
|---------|---------------|---------|
| `REQUIRED`（默认） | 有事务则加入，无事务则新建 | 90% 的业务场景 |
| `REQUIRES_NEW` | 总是新建事务，挂起当前事务 | 日志记录、审计、异步通知 |
| `NESTED` | 嵌套事务，创建 Savepoint（需 JDBC 驱动支持） | 部分失败可恢复的批量操作 |
| `SUPPORTS` | 有事务则加入，无事务则非事务运行 | 纯查询，兼顾性能 |
| `NOT_SUPPORTED` | 总是以非事务方式运行，挂起当前事务 | 不需事务保护的耗时操作 |
| `MANDATORY` | 必须在事务中运行，否则抛异常 | 对事务有严格要求的操作 |
| `NEVER` | 不能在事务中运行，否则抛异常 | 防止被误放在事务中的操作 |

### 5.1.2 readOnly 优化

`@Transactional(readOnly = true)` 对 Hibernate 产生以下影响：

```java
// readOnly 标志的实际效果
public class ReadOnlyEffects {

    @Transactional(readOnly = true)
    public void queryOnly() {
        // 1. FlushMode 设置为 MANUAL
        //    不会自动 flush 脏检查，减少数据库同步
        //    显式调用 flush() 会抛异常

        // 2. Hibernate 不维护快照（5.x+）
        //    减少内存占用，因为无需对比新旧状态

        // 3. JDBC Connection 设置只读提示
        //    某些数据库（如 PostgreSQL）会优化查询计划

        // 4. 脏检查被跳过
        //    实体属性修改不会被同步到数据库
    }
}
```

## 5.2 Session 与事务绑定

Spring 使用 `SpringSessionContext` 实现 `CurrentSessionContext` 接口，将 Hibernate Session 与 Spring 事务绑定：

```
Spring 事务周期：
┌──────────────────────────────────────────────────────────────┐
│ @Transactional                                                │
│  ├── 事务开始（AOP 环绕通知）                                   │
│  │   ├── 创建/获取 Connection → 绑定到 DataSourceTransactionManager │
│  │   ├── 创建 Session → 绑定到 SpringSessionContext              │
│  │   ├── Session 绑定到当前线程的 TransactionSynchronizationManager │
│  │   │                                                          │
│  │   ├── 业务方法执行                                            │
│  │   │   └── getCurrentSession() → 返回绑定的 Session             │
│  │   │                                                          │
│  │   ├── 事务提交前：flush() Session（脏数据同步到 DB）             │
│  │   ├── 提交数据库事务（commit）                                 │
│  │   └── 关闭 Session，释放绑定资源                                │
│  ├── 事务回滚时：Session 被 close()，Connection 回滚               │
└──────────────────────────────────────────────────────────────┘
```

```java
@Service
public class SessionBindingDemo {

    @Autowired
    private SessionFactory sessionFactory;

    @Transactional
    public void demonstrateSessionBinding() {
        // getCurrentSession() 返回与当前事务绑定的 Session
        Session session1 = sessionFactory.getCurrentSession();
        Session session2 = sessionFactory.getCurrentSession();

        System.out.println(session1 == session2);  // true

        // 等价于 EntityManager（在 Spring 中）
        // EntityManager em = entityManagerFactory.createEntityManager();
        // 但在事务内，直接注入的 EntityManager 也是事务绑定的代理
    }

    // 非事务方法中调用 getCurrentSession() 会抛出异常
    // HibernateException: No CurrentSessionContext configured!
    public void withoutTransaction() {
        // 以下调用将失败，因为没有事务 → 没有 Session
        // Session session = sessionFactory.getCurrentSession(); // ERROR!
    }
}
```

**Session 生命周期规则**：

| 场景 | Session 创建时机 | Session 关闭时机 |
|------|-----------------|-----------------|
| `@Transactional` 方法 | 事务开始时（首个需要 Session 的操作） | 事务提交/回滚时 |
| OSIV 开启 | HTTP 请求开始时 | HTTP 请求结束时 |
| 手动 `EntityManager` | `emf.createEntityManager()` 调用时 | 显式 `em.close()` |

## 5.3 事务与异常的转换

Hibernate 访问数据库时抛出的异常属于受检异常体系（`HibernateException` 及其子类）。Spring 通过异常翻译机制将其转换为 Spring 的非受检 `DataAccessException` 体系。

### 5.3.1 异常转换原理

```
Hibernate 异常层次                   Spring 异常层次
┌──────────────────┐               ┌──────────────────────┐
│ HibernateException│  ──转换──→  │ DataAccessException   │
│  ├── Constraint   │              │  ├── DataIntegrity    │
│  │   Violation    │              │  │   ViolationException│
│  ├── StaleState   │              │  ├── OptimisticLock   │
│  │   Exception    │              │  │   FailureException  │
│  ├── Query        │              │  ├── InvalidDataAccess│
│  │   Exception    │              │  │   ApiUsageException │
│  ├── Lazy         │              │  ├── DataRetrieval    │
│  │   Initialization│             │  │   FailureException  │
│  │   Exception    │              │  └── CannotAcquire    │
│  └── ...          │              │      LockException    │
└──────────────────┘               └──────────────────────┘
```

### 5.3.2 @Repository 自动翻译

```java
// @Repository 注解自动注册 PersistenceExceptionTranslationPostProcessor
@Repository
public class UserDao {

    @PersistenceContext
    private EntityManager em;

    public User findById(Long id) {
        try {
            return em.find(User.class, id);
        } catch (Exception e) {
            // HibernateException 自动转换为 Spring DataAccessException
            throw e;  // 传给 AOP 代理处理
        }
    }
}
```

`@Repository` 注解的原理：Spring 在解析 `@Repository` 时自动注册一个 `PersistenceExceptionTranslationAdvisor`，环绕目标对象的所有方法，捕获 JPA / Hibernate 异常并转换为 Spring 异常体系。

### 5.3.3 手动启用异常翻译

```java
@Configuration
public class PersistenceConfig {

    // 如果某 Bean 没有 @Repository 注解，可以手动注册
    @Bean
    public PersistenceExceptionTranslationPostProcessor exceptionTranslation() {
        return new PersistenceExceptionTranslationPostProcessor();
    }
}
```

---

# 6. Open Session in View（OSIV）

## 6.1 OSIV 原理

OSIV 是 Web 应用中最容易被忽视但也最具争议的配置。Spring Boot 默认开启 OSIV（`spring.jpa.open-in-view = true`）。

### 6.1.1 工作流程

```
HTTP Request 生命周期（OSIV 开启）：

1. 请求到达 DispatcherServlet
       │
2. OpenEntityManagerInViewInterceptor.preHandle()
       │── EntityManagerFactory.createEntityManager()
       │── 将 EntityManager 绑定到当前线程
       │── 此阶段 Session 已打开，但未真正获取数据库连接
       │
3. Controller 处理请求
       │── 调用 Service 层
       │── Service 层有 @Transactional → 开启数据库事务
       │   └── 获取 Connection，关联到已有 Session
       │
4. View 渲染（如 Thymeleaf、JSP）
       │── 可以懒加载实体关联（因为 Session 仍打开）
       │── user.getOrders().size() → 自动触发查询
       │
5. OpenEntityManagerInViewInterceptor.afterCompletion()
       │── EntityManager.close()
       │── Session 关闭
```

```java
// OpenEntityManagerInViewInterceptor 核心逻辑（简化）
public class OpenEntityManagerInViewInterceptor {

    @Override
    public void preHandle(WebRequest request) {
        EntityManager em = createEntityManager();
        EntityManagerHolder holder = new EntityManagerHolder(em);
        TransactionSynchronizationManager.bindResource(
            emf, holder);
    }

    @Override
    public void afterCompletion(WebRequest request, Exception ex) {
        EntityManagerHolder holder = (EntityManagerHolder)
            TransactionSynchronizationManager.unbindResource(emf);
        EntityManagerFactoryUtils.closeEntityManager(
            holder.getEntityManager());
    }
}
```

## 6.2 OSIV 的问题

### 6.2.1 问题一：数据库连接长时间占用

```
请求时间线（OSIV 开启）：

时间 ──────────────────────────────────────────────────→
        │←── 连接获取 ──→│
        │  @Transactional  │
        │  方法执行          │←── View 渲染 ──────────→│
        │  (持有连接)        │  (仍在事务外，              │
        │                   │   但如有懒加载，              │
        │                   │   会再次申请连接)            │
        
问题：长请求占用连接，连接池可能耗尽
```

### 6.2.2 问题二：N+1 查询难以控制

```java
@Controller
public class OrderController {

    @GetMapping("/orders")
    public String list(Model model) {
        List<User> users = userService.findAll();  // SELECT * FROM users
        model.addAttribute("users", users);
        return "order-list";
    }
}
```

```html
<!-- Thymeleaf 模板（OSIV 导致 N+1） -->
<div th:each="user : ${users}">
    <h3 th:text="${user.name}"></h3>
    <ul>
        <!-- 每次访问 user.orders 触发一次 SQL！ -->
        <li th:each="order : ${user.orders}"
            th:text="${order.title}"></li>
    </ul>
</div>
```

渲染时对每个 `user` 触发一次 `SELECT * FROM orders WHERE user_id = ?`，产生 N+1 查询。而开发者难以在模板中优化此行为。

### 6.2.3 问题三：连接池耗尽

```
高并发场景：
- 100 个并发请求
- 连接池大小：20
- 每个请求持有连接时间：2 秒（含视图渲染和懒加载）

结果：20 个请求获取到连接，剩余 80 个排队等待 → 响应时间急剧增加
```

## 6.3 关闭 OSIV

### 6.3.1 配置方式

```yaml
spring:
  jpa:
    open-in-view: false
```

关闭 OSIV 后，`OpenEntityManagerInViewInterceptor` 不会被注册。Session 仅在 `@Transactional` 方法期间打开，事务结束后 Session 立即关闭。

### 6.3.2 关闭后的影响与解决方案

关闭 OSIV 后，视图层访问未加载的懒加载属性将抛出 `LazyInitializationException`：

```java
// 错误示例（OSIV 关闭后）
@GetMapping("/users")
public String list(Model model) {
    List<User> users = userRepository.findAll();
    model.addAttribute("users", users);
    return "user-list";  // 视图访问 user.orders 时爆炸！
}
// 异常：LazyInitializationException: could not initialize proxy - no Session
```

**解决方案一：DTO 投影（推荐）**

```java
// 在 Service 层完成数据组装，返回专用 DTO
@Service
@Transactional(readOnly = true)
public class UserService {

    public List<UserDto> findAllUserDtos() {
        return userRepository.findAll().stream()
            .map(user -> UserDto.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .orders(user.getOrders().stream()
                    .map(OrderDto::fromEntity)
                    .toList())
                .build())
            .toList();
    }
}
```

**解决方案二：@EntityGraph / JOIN FETCH**

```java
public interface UserRepository extends JpaRepository<User, Long> {

    @EntityGraph(attributePaths = {"orders"})
    List<User> findAll();

    @Query("SELECT u FROM User u JOIN FETCH u.orders")
    List<User> findAllWithOrders();
}
```

**解决方案三：事务内完成懒加载**

```java
@Service
@Transactional(readOnly = true)
public class UserService {

    public List<User> getUsersWithOrders() {
        List<User> users = userRepository.findAll();
        // 在事务结束前，手动触发懒加载
        users.forEach(user -> user.getOrders().size());
        return users;
    }
}
```

**方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| DTO 投影 | 精确控制数据，减少传输量，解耦视图与实体 | 需额外编写映射代码 |
| @EntityGraph | 简单声明式，ORM 透明 | 不能 JOIN FETCH 多个集合（笛卡尔积） |
| 事务内加载 | 实现简单 | 可能加载不必要的数据，N+1 隐患 |
| 保持 OSIV | 开发简单快 | 生产风险大，连接池可能耗尽 |

---

# 7. Spring Data JPA 集成

## 7.1 Repository 基础

Spring Data JPA 通过 `JpaRepository` 接口提供开箱即用的 CRUD 和分页能力：

```java
// 基础 Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

自动提供的标准方法：

| 方法 | 说明 |
|------|------|
| `save(S entity)` | 保存（新增或更新） |
| `saveAll(Iterable<S> entities)` | 批量保存 |
| `findById(ID id)` | 按主键查找，返回 Optional |
| `existsById(ID id)` | 判断是否存在 |
| `findAll()` | 查询所有 |
| `findAllById(Iterable<ID> ids)` | 按 ID 批量查找 |
| `count()` | 总数 |
| `deleteById(ID id)` | 按 ID 删除 |
| `delete(T entity)` | 删除实体 |
| `deleteAll()` | 删除所有 |
| `findAll(Pageable pageable)` | 分页查询 |
| `findAll(Sort sort)` | 排序查询 |

### 7.1.1 方法命名查询（Derived Query Methods）

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // ====== 精确匹配 ======
    Optional<User> findByEmail(String email);

    // ====== 多条件 AND ======
    List<User> findByAgeAndStatus(Integer age, String status);

    // ====== OR 条件 ======
    List<User> findByAgeOrName(Integer age, String name);

    // ====== 比较操作 ======
    List<User> findByAgeGreaterThan(Integer age);
    List<User> findByAgeLessThanEqual(Integer age);
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    // ====== 模糊匹配 ======
    List<User> findByNameLike(String pattern);
    List<User> findByNameStartingWith(String prefix);
    List<User> findByEmailContaining(String keyword);

    // ====== NULL 判断 ======
    List<User> findByDeletedAtIsNull();
    List<User> findByDeletedAtIsNotNull();

    // ====== IN 条件 ======
    List<User> findByStatusIn(Collection<String> statuses);

    // ====== 排序 ======
    List<User> findByAgeOrderByNameDesc(Integer age);

    // ====== 分页 ======
    Page<User> findByStatus(String status, Pageable pageable);

    // ====== 限制 ======
    User findFirstByOrderByCreatedAtDesc();
    List<User> findTop3ByStatusOrderByAgeAsc(String status);

    // ====== count / exists / delete ======
    long countByStatus(String status);
    boolean existsByEmail(String email);
    void deleteByStatus(String status);
}
```

### 7.1.2 @Query 注解

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // JPQL 查询
    @Query("SELECT o FROM Order o WHERE o.user.id = :userId AND o.status = :status")
    List<Order> findByUserAndStatus(
        @Param("userId") Long userId,
        @Param("status") OrderStatus status);

    // JPQL + 动态排序
    @Query("SELECT o FROM Order o WHERE o.amount > :amount")
    List<Order> findExpensiveOrders(
        @Param("amount") BigDecimal amount, Sort sort);

    // 原生 SQL
    @Query(value =
        "SELECT * FROM orders o WHERE o.created_at >= :since",
        nativeQuery = true)
    List<Order> findRecentOrders(@Param("since") LocalDateTime since);

    // JDBC 风格参数绑定（按位置）
    @Query("SELECT o FROM Order o WHERE o.status = ?1 AND o.amount > ?2")
    List<Order> findByStatusAndAmountGreaterThan(
        OrderStatus status, BigDecimal amount);

    // @Modifying：UPDATE / DELETE 操作
    @Modifying
    @Query("UPDATE Order o SET o.status = :newStatus WHERE o.id IN :ids")
    int batchUpdateStatus(
        @Param("ids") List<Long> ids,
        @Param("newStatus") OrderStatus newStatus);

    @Modifying
    @Query("DELETE FROM Order o WHERE o.status = :status AND o.createdAt < :before")
    int deleteOldOrders(
        @Param("status") OrderStatus status,
        @Param("before") LocalDateTime before);
}
```

`@Modifying` 注意事项：
1. 必须配合 `@Transactional` 使用（UPDATE / DELETE 必须在事务中执行）
2. 不会自动触发持久化上下文刷新，需要手动 `clear()`
3. 返回值为受影响的行数（不是实体）

### 7.1.3 @EntityGraph 声明抓取策略

```java
// 在实体上预定义 EntityGraph
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "User.withOrders",
        attributeNodes = @NamedAttributeNode("orders")
    ),
    @NamedEntityGraph(
        name = "User.withOrdersAndAddress",
        attributeNodes = {
            @NamedAttributeNode("orders"),
            @NamedAttributeNode("address")
        }
    )
})
public class User {
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;

    @ManyToOne(fetch = FetchType.LAZY)
    private Address address;
}

// 在 Repository 方法上引用
public interface UserRepository extends JpaRepository<User, Long> {

    @EntityGraph(value = "User.withOrders", type = EntityGraph.EntityGraphType.FETCH)
    List<User> findByStatus(String status);

    // 动态声明 ad-hoc EntityGraph
    @EntityGraph(attributePaths = {"orders", "orders.items"})
    Optional<User> findWithOrdersById(Long id);
}
```

## 7.2 JpaSpecificationExecutor

`JpaSpecificationExecutor` 结合 Criteria API 实现类型安全的动态查询：

```java
public interface UserRepository extends
        JpaRepository<User, Long>,
        JpaSpecificationExecutor<User> {
}
```

```java
@Service
@RequiredArgsConstructor
public class UserQueryService {

    private final UserRepository userRepository;

    public List<User> searchUsers(String name, Integer minAge,
                                   Integer maxAge, String status) {
        Specification<User> spec = Specification.where(null);

        if (name != null && !name.isBlank()) {
            spec = spec.and((root, query, cb) ->
                cb.like(root.get("name"), "%" + name + "%"));
        }
        if (minAge != null) {
            spec = spec.and((root, query, cb) ->
                cb.ge(root.get("age"), minAge));
        }
        if (maxAge != null) {
            spec = spec.and((root, query, cb) ->
                cb.le(root.get("age"), maxAge));
        }
        if (status != null) {
            spec = spec.and((root, query, cb) ->
                cb.equal(root.get("status"), status));
        }

        return userRepository.findAll(spec);
    }
}
```

### 7.2.1 封装通用动态查询工具

```java
public class SpecificationBuilder<T> {

    private final List<Specification<T>> specs = new ArrayList<>();

    public static <T> SpecificationBuilder<T> of(Class<T> clazz) {
        return new SpecificationBuilder<>();
    }

    public SpecificationBuilder<T> like(String field, String value) {
        if (value != null && !value.isBlank()) {
            specs.add((root, query, cb) ->
                cb.like(root.get(field), "%" + value + "%"));
        }
        return this;
    }

    public SpecificationBuilder<T> eq(String field, Object value) {
        if (value != null) {
            specs.add((root, query, cb) ->
                cb.equal(root.get(field), value));
        }
        return this;
    }

    public SpecificationBuilder<T> in(String field, Collection<?> values) {
        if (values != null && !values.isEmpty()) {
            specs.add((root, query, cb) ->
                root.get(field).in(values));
        }
        return this;
    }

    public SpecificationBuilder<T> between(String field,
                                            Comparable<?> from,
                                            Comparable<?> to) {
        if (from != null && to != null) {
            specs.add((root, query, cb) ->
                cb.between(root.get(field), from, to));
        }
        return this;
    }

    public Specification<T> build() {
        return specs.stream()
            .reduce(Specification.where(null), Specification::and);
    }
}

// 使用
List<User> users = userRepository.findAll(
    SpecificationBuilder.of(User.class)
        .like("name", keyword)
        .eq("status", activeStatus)
        .between("age", 20, 40)
        .build()
);
```

## 7.3 自定义 Repository

### 7.3.1 Fragment 组合模式

```java
// 1. 定义自定义接口
public interface UserRepositoryCustom {
    List<User> findUsersWithComplexCriteria(Map<String, Object> filters);
    void batchUpdateStatus(List<Long> ids, String status);
}

// 2. 实现自定义接口（类名必须为：接口名 + Impl）
@RequiredArgsConstructor
public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    private final EntityManager em;

    @Override
    public List<User> findUsersWithComplexCriteria(Map<String, Object> filters) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);

        List<Predicate> predicates = new ArrayList<>();
        filters.forEach((key, value) -> {
            if (value != null) {
                predicates.add(cb.equal(root.get(key), value));
            }
        });
        query.where(predicates.toArray(new Predicate[0]));

        return em.createQuery(query).getResultList();
    }

    @Override
    @Transactional
    public void batchUpdateStatus(List<Long> ids, String status) {
        em.createQuery(
            "UPDATE User u SET u.status = :status WHERE u.id IN :ids")
            .setParameter("status", status)
            .setParameter("ids", ids)
            .executeUpdate();
        em.flush();
        em.clear();
    }
}

// 3. 组合到主 Repository
public interface UserRepository extends
        JpaRepository<User, Long>,
        UserRepositoryCustom {  // 继承自定义接口
}
```

### 7.3.2 自定义后缀名

```java
// 默认后缀是 Impl，可自定义
@SpringBootApplication
@EnableJpaRepositories(
    repositoryImplementationPostfix = "CustomImpl"
)
public class Application {
}

// 此时实现类应命名为：UserRepositoryCustomImpl → 匹配 "CustomImpl"
```

## 7.4 审计功能

### 7.4.1 实体审计注解

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedBy
    private String lastModifiedBy;

    @LastModifiedDate
    private LocalDateTime lastModifiedAt;
}
```

### 7.4.2 启用审计 + AuditorAware

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class AuditConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            // 从 SecurityContext 获取当前登录用户
            Authentication authentication =
                SecurityContextHolder.getContext().getAuthentication();
            if (authentication == null || !authentication.isAuthenticated()) {
                return Optional.of("SYSTEM");
            }
            return Optional.of(authentication.getName());
        };
    }
}
```

### 7.4.3 提取公共审计基类

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseAuditEntity {

    @CreatedBy
    @Column(name = "created_by", updatable = false, length = 50)
    private String createdBy;

    @CreatedDate
    @Column(name = "created_at", updatable = false, nullable = false)
    private LocalDateTime createdAt;

    @LastModifiedBy
    @Column(name = "updated_by", length = 50)
    private String updatedBy;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}

@Entity
public class Product extends BaseAuditEntity {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
}
```

---

# 8. 多数据源配置

## 8.1 基础多数据源

以下是双数据源（主库 + 副库）的完整配置：

```java
@Configuration
public class DataSourceConfig {

    // ==================== 主数据源 ====================
    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("primaryDataSource") DataSource dataSource) {

        return builder
                .dataSource(dataSource)
                .packages("com.example.domain.primary")
                .persistenceUnit("primary")
                .build();
    }

    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManagerFactory") EntityManagerFactory emf) {

        return new JpaTransactionManager(emf);
    }

    // ==================== 副数据源 ====================
    @Bean(name = "secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean secondaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("secondaryDataSource") DataSource dataSource) {

        return builder
                .dataSource(dataSource)
                .packages("com.example.domain.secondary")
                .persistenceUnit("secondary")
                .build();
    }

    @Bean(name = "secondaryTransactionManager")
    public PlatformTransactionManager secondaryTransactionManager(
            @Qualifier("secondaryEntityManagerFactory") EntityManagerFactory emf) {

        return new JpaTransactionManager(emf);
    }
}
```

### 8.1.1 分别指定 Repository 扫描

```java
// 主库 Repository
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository.primary",
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryJpaConfig {
}

// 副库 Repository
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository.secondary",
    entityManagerFactoryRef = "secondaryEntityManagerFactory",
    transactionManagerRef = "secondaryTransactionManager"
)
public class SecondaryJpaConfig {
}
```

### 8.1.2 多数据源 yml 配置

```yaml
spring:
  datasource:
    primary:
      jdbc-url: jdbc:mysql://localhost:3306/primary_db?useSSL=false
      username: root
      password: ${PRIMARY_DB_PASSWORD}
      driver-class-name: com.mysql.cj.jdbc.Driver
      hikari:
        maximum-pool-size: 20
        pool-name: PrimaryHikariPool

    secondary:
      jdbc-url: jdbc:postgresql://localhost:5432/secondary_db
      username: postgres
      password: ${SECONDARY_DB_PASSWORD}
      driver-class-name: org.postgresql.Driver
      hikari:
        maximum-pool-size: 10
        pool-name: SecondaryHikariPool

  jpa:
    primary:
      properties:
        hibernate:
          dialect: org.hibernate.dialect.MySQL8Dialect
    secondary:
      properties:
        hibernate:
          dialect: org.hibernate.dialect.PostgreSQLDialect
```

> 注意：多数据源情况下，Spring Boot 的 JPA 自动配置会退避，需要手动配置 EntityManagerFactory 和 TransactionManager。

## 8.2 读写分离

基于 `AbstractRoutingDataSource` 实现动态数据源路由：

```java
// 1. 数据源类型枚举
public enum DataSourceType {
    MASTER, SLAVE
}

// 2. 数据源上下文持有者（ThreadLocal）
public class DataSourceContextHolder {

    private static final ThreadLocal<DataSourceType> CONTEXT = new ThreadLocal<>();

    public static void set(DataSourceType type) {
        CONTEXT.set(type);
    }

    public static DataSourceType get() {
        return CONTEXT.get() != null ? CONTEXT.get() : DataSourceType.MASTER;
    }

    public static void clear() {
        CONTEXT.remove();
    }
}

// 3. 动态数据源
@Component
public class DynamicDataSource extends AbstractRoutingDataSource {

    public DynamicDataSource(
            @Qualifier("masterDataSource") DataSource master,
            @Qualifier("slaveDataSource") DataSource slave) {

        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER, master);
        targetDataSources.put(DataSourceType.SLAVE, slave);

        super.setDefaultTargetDataSource(master);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.get();
    }
}

// 4. 切面：根据 @Transactional(readOnly) 自动路由
@Aspect
@Component
@Order(Integer.MIN_VALUE)  // 确保在 @Transactional 之前执行
public class ReadOnlyRouteInterceptor {

    @Around("@annotation(transactional)")
    public Object determineRoute(ProceedingJoinPoint joinPoint,
                                  Transactional transactional) throws Throwable {
        try {
            if (transactional.readOnly()) {
                DataSourceContextHolder.set(DataSourceType.SLAVE);
            } else {
                DataSourceContextHolder.set(DataSourceType.MASTER);
            }
            return joinPoint.proceed();
        } finally {
            DataSourceContextHolder.clear();
        }
    }
}
```

```java
@Service
public class ReadWriteDemo {

    @Transactional(readOnly = true)  // → 从库
    public List<User> findAll() {
        return userRepository.findAll();
    }

    @Transactional                     // → 主库（写）
    public void createUser(User user) {
        userRepository.save(user);
    }
}
```

## 8.3 多数据源事务管理

### 8.3.1 分布式事务（JTA + Atomikos）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jta-atomikos</artifactId>
</dependency>
```

```yaml
spring:
  jta:
    atomikos:
      properties:
        log-base-dir: ./data/tx-logs
        max-timeout: 600000
        default-jta-timeout: 300000

  # Atomikos 会自动管理多个数据源的事务协调
  datasource:
    db1:
      xa-data-source-class-name: com.mysql.cj.jdbc.MysqlXADataSource
      xa-properties:
        url: jdbc:mysql://localhost:3306/db1
        user: root
        password: ${DB1_PASSWORD}
    db2:
      xa-data-source-class-name: com.mysql.cj.jdbc.MysqlXADataSource
      xa-properties:
        url: jdbc:mysql://localhost:3306/db2
        user: root
        password: ${DB2_PASSWORD}
```

### 8.3.2 非 XA 的多数据源事务方案

| 方案 | 描述 | 适用场景 |
|------|------|---------|
| **Best Efforts 1PC** | 先提交第一个，再提交第二个（无协调） | 对一致性要求不高的场景 |
| **补偿事务（TCC）** | Try-Confirm-Cancel 模式，手动编写补偿逻辑 | 要求最终一致性 |
| **Saga 模式** | 本地事务 + 事件驱动的补偿 | 微服务间分布式事务 |
| **JTA（XA）** | 两阶段提交，强一致性 | 对一致性要求极高的场景 |

> 警告：`ChainedTransactionManager` 在 Spring Boot 2.x 已被标记为 Deprecated，在 3.x 中移除。不建议在生产环境使用。应改用 JTA（Atomikos / Narayana）或最终一致性方案。

---

# 9. 测试支持

## 9.1 @DataJpaTest

`@DataJpaTest` 是 Spring Boot 为 JPA 测试提供的专用注解，默认配置：

- 扫描 `@Entity` 类，配置 `LocalContainerEntityManagerFactoryBean`
- 自动使用嵌入式内存数据库（H2 / Derby / HSQL）
- 每个测试方法自动开启事务并在结束后回滚
- 仅加载 JPA 相关 Bean（`@Repository`、`EntityManager`、`DataSource` 等）

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindByEmail() {
        // TestEntityManager 是 @DataJpaTest 提供的轻量级 EntityManager
        User user = new User();
        user.setName("张三");
        user.setEmail("zhangsan@example.com");
        user.setAge(25);
        entityManager.persistAndFlush(user);  // 立即写入，绕过回滚

        Optional<User> found = userRepository.findByEmail("zhangsan@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("张三");
    }

    @Test
    void shouldSaveAndFindById() {
        User user = new User();
        user.setName("李四");
        user.setEmail("lisi@example.com");
        User saved = userRepository.save(user);

        Optional<User> found = userRepository.findById(saved.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getEmail()).isEqualTo("lisi@example.com");
    }
}
```

### 9.1.1 @AutoConfigureTestDatabase 控制测试数据库

```java
// 使用真实数据库（非 H2）
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class RealDatabaseTest {

    // 此时使用 application.yml 中配置的真实数据库
    // 注意：需要确保测试数据库可用，且测试数据被正确清理
}

// 显示指定测试数据源配置
@DataJpaTest
@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.H2)
class H2SpecificTest {
}

@DataJpaTest
@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.HSQL)
class HsqlSpecificTest {
}
```

### 9.1.2 事务不回滚

```java
@DataJpaTest
class PersistenceTest {

    @Test
    @Commit  // 或 @Rollback(false)
    void shouldPersistAndNotRollback() {
        User user = new User();
        user.setName("持久化用户");
        userRepository.save(user);

        // 事务提交，数据持久化到数据库
    }
}
```

## 9.2 集成测试

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@Transactional  // 事务回滚，保持测试隔离
class UserControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldCreateUser() throws Exception {
        String json = """
            {
                "name": "测试用户",
                "email": "test@example.com",
                "age": 30
            }
            """;

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(json))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("测试用户"));

        assertThat(userRepository.count()).isEqualTo(1);
    }
}
```

## 9.3 性能测试注意事项

### 9.3.1 Hibernate Statistics 断言查询次数

```java
@DataJpaTest
class QueryPerformanceTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private EntityManagerFactory emf;

    @BeforeEach
    void setUp() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();
        stats.setStatisticsEnabled(true);
        stats.clear();
    }

    @Test
    void shouldNotCauseNPlusOneQueries() {
        // 准备数据：1 个 User，N 个 Order
        User user = new User();
        user.setName("Test");
        userRepository.save(user);

        for (int i = 0; i < 10; i++) {
            Order order = new Order();
            order.setUser(user);
            order.setAmount(BigDecimal.valueOf(100));
            // ... save orders
        }

        // 清除 Hibernate Session 和二级缓存
        entityManager.flush();
        entityManager.clear();

        // 执行查询并断言 SQL 语句数
        Statistics stats = emf.unwrap(SessionFactory.class)
            .getStatistics();
        long beforeCount = stats.getQueryExecutionCount();

        @SuppressWarnings("unchecked")
        List<User> users = entityManager.getEntityManager()
            .createQuery(
                "SELECT u FROM User u JOIN FETCH u.orders")
            .getResultList();
        users.get(0).getOrders().size();

        long afterCount = stats.getQueryExecutionCount();
        long executedQueries = afterCount - beforeCount;

        // 期望只有 1 条 SQL（JOIN FETCH 一条查询搞定）
        assertThat(executedQueries)
            .as("JOIN FETCH 应只产生一次查询，而非 N+1")
            .isEqualTo(1);
    }
}
```

### 9.3.2 测试间的数据隔离

```java
@DataJpaTest
class IsolatedRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    // 方式一：每个测试后手动清理
    @AfterEach
    void tearDown() {
        userRepository.deleteAll();
    }

    // 方式二：使用 @Sql 初始化/清理脚本
    @Test
    @Sql(scripts = "/sql/init-users.sql")
    @Sql(scripts = "/sql/cleanup.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void shouldWorkWithPreloadedData() {
        assertThat(userRepository.count()).isGreaterThan(0);
    }
}
```

---

# 10. 常见问题与故障排查

## 10.1 常见启动错误

### 10.1.1 错误一：Unable to determine Dialect

```
Caused by: org.hibernate.HibernateException:
    Access to DialectResolutionInfo cannot be null when
    'hibernate.dialect' not set
```

**原因**：Hibernate 无法自动检测数据库方言。

**解决**：

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.MySQL8Dialect
```

或在自动配置中显式设置：

```java
@Bean
public HibernatePropertiesCustomizer dialectCustomizer() {
    return props -> props.put(
        "hibernate.dialect", MySQL8Dialect.class.getName());
}
```

### 10.1.2 错误二：No EntityManager with actual transaction available

```
org.springframework.dao.InvalidDataAccessApiUsageException:
    No EntityManager with actual transaction available for current
    thread - cannot reliably process 'persist' call
```

**原因**：在非事务上下文中执行写操作（`persist`、`merge`、`remove`）。

**解决**：给调用方法加上 `@Transactional` 注解。

```java
// 错误
public void save(User user) {
    em.persist(user);
}

// 正确
@Transactional
public void save(User user) {
    em.persist(user);
}
```

### 10.1.3 错误三：Multiple representations of the same entity

```
java.lang.IllegalStateException:
    Multiple representations of the same entity [...] are being merged.
```

**原因**：在同一个 Session 中试图 merge 一个实体两次，或者实体主键/`equals`/`hashCode` 实现有问题。

**解决**：

```java
// 避免：在同一个事务中重复 merge 同一实体
// 如果是 detached entity，应使用 find + setProperty 模式
@Transactional
public User updateUser(Long id, UserDto dto) {
    User user = em.find(User.class, id);     // 先从数据库加载
    user.setName(dto.getName());              // 修改属性（自动脏检查）
    user.setEmail(dto.getEmail());
    return user;  // 事务提交时自动更新
    // 不要这样做：em.merge(userFromDto);
}
```

## 10.2 常见运行时错误

### 10.2.1 LazyInitializationException

```
org.hibernate.LazyInitializationException:
    could not initialize proxy [com.example.User#1] - no Session
```

**常见场景与解决方案**：

```java
// 场景 1：事务外访问懒加载关联
// 错误代码
@Transactional
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}
// 在 Controller 层：user.getOrders().size() → LazyInitializationException

// 解决方案 A：事务内触发懒加载
@Transactional
public User getUserWithOrders(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    Hibernate.initialize(user.getOrders());  // 触发加载
    return user;
}

// 解决方案 B：JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findWithOrdersById(@Param("id") Long id);

// 解决方案 C：@EntityGraph
@EntityGraph(attributePaths = "orders")
Optional<User> findWithOrdersById(Long id);

// 解决方案 D：DTO 投影
@Query("SELECT new com.example.dto.UserDto(" +
       "u.id, u.name, u.email) FROM User u WHERE u.id = :id")
Optional<UserDto> findUserDtoById(@Param("id") Long id);
```

### 10.2.2 OptimisticLockException

```
org.hibernate.StaleObjectStateException:
    Row was updated or deleted by another transaction
```

**原因**：两个并发事务修改了同一条记录，后提交的事务检测到版本号已变化。

```java
// 乐观锁重试机制
@Service
public class OptimisticLockRetryService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100)
    )
    public void updateWithRetry(Long orderId, BigDecimal amount) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow();
        order.setAmount(amount);
        orderRepository.save(order);
    }
}
```

### 10.2.3 NonUniqueResultException

```
org.hibernate.NonUniqueResultException:
    query did not return a unique result: 5
```

**解决**：使用 `getResultList()` + 手动校验，或将 `getSingleResult()` 替换为带分页的查询：

```java
// 不要写这样的代码
@Query("SELECT u FROM User u WHERE u.email LIKE :pattern")
User findByEmailPattern(@Param("pattern") String pattern);
// ↑ 如果匹配多条记录 → NonUniqueResultException

// 应改为
@Query("SELECT u FROM User u WHERE u.email LIKE :pattern")
List<User> findByEmailPattern(@Param("pattern") String pattern);

// 或限制返回第一条
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
```

## 10.3 排查工具

### 10.3.1 SQL 日志配置（开发环境）

```yaml
# 方式一：Spring JPA 配置
spring:
  jpa:
    show-sql: true
    format-sql: true
    properties:
      hibernate:
        use_sql_comments: true
        highlight_sql: true

# 方式二：Logging 级别
logging:
  level:
    org.hibernate.SQL: DEBUG                           # SQL 语句
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # 参数绑定
    org.hibernate.orm.jdbc.bind: TRACE                 # Hibernate 6.x 参数绑定
    org.hibernate.stat: DEBUG                          # 统计信息
    org.hibernate.cache: DEBUG                         # 二级缓存
    org.hibernate.engine.transaction: DEBUG            # 事务
```

### 10.3.2 p6spy 插件（SQL 日志 + 慢查询）

```xml
<dependency>
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>p6spy-spring-boot-starter</artifactId>
    <version>1.9.1</version>
</dependency>
```

```yaml
decorator:
  datasource:
    p6spy:
      enable-logging: true
      logging: slf4j
      log-format: |
        Execution Time: %(executionTime) ms
        Connection ID: %(connectionId)
        SQL: %(sql)
      multiline: true
      exclude:
        - "SELECT 1"
```

### 10.3.3 datasource-proxy（SQL 监控 + 慢查询告警）

```xml
<dependency>
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>datasource-proxy-spring-boot-starter</artifactId>
    <version>1.9.1</version>
</dependency>
```

```yaml
decorator:
  datasource:
    datasource-proxy:
      logging: slf4j
      query:
        enable-logging: true
        slow-query:
          enable-logging: true
          threshold: 1000  # 慢查询阈值（毫秒）
          log-level: WARN
```

### 10.3.4 Hibernate Statistics 编程式查询

```java
@Component
public class HibernateStatisticsReporter {

    @Autowired
    private EntityManagerFactory emf;

    @Scheduled(fixedRate = 60000)  // 每分钟输出一次
    public void reportStatistics() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();

        System.out.printf("""
            ====== Hibernate Statistics ======
            Sessions Opened:    %d
            Sessions Closed:    %d
            Transactions:       %d
            Queries Executed:   %d
            Cache Hit Rate:     %.2f%%
            Flushes:            %d
            Entities Loaded:    %d
            Collections Loaded: %d
            ==================================
            """,
            stats.getSessionOpenCount(),
            stats.getSessionCloseCount(),
            stats.getTransactionCount(),
            stats.getQueryExecutionCount(),
            stats.getSecondLevelCacheHitCount() * 100.0 /
                Math.max(1, stats.getSecondLevelCacheHitCount()
                    + stats.getSecondLevelCacheMissCount()),
            stats.getFlushCount(),
            stats.getEntityLoadCount(),
            stats.getCollectionLoadCount()
        );

        stats.clear();  // 重置统计（使每次报告的是增量数据）
    }
}
```

---

# 11. 部署与运维

## 11.1 DDL 策略

### 11.1.1 环境策略矩阵

| 环境 | ddl-auto | 数据库版本管理 | 说明 |
|------|----------|--------------|------|
| **本地开发** | `update` | 可选 Flyway/Liquibase | 方便快速迭代，但注意表结构残留问题 |
| **集成测试** | `create-drop` | Flyway/Liquibase | 每次测试干净启停，验证迁移脚本正确性 |
| **预发布** | `validate` | Flyway/Liquibase | 严格验证实体与表结构一致性 |
| **生产** | `none` 或 `validate` | Flyway/Liquibase | Schema 变更由 DBA 审核后通过迁移工具执行 |

### 11.1.2 Flyway 集成

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 1
    validate-on-migrate: true
    clean-disabled: true         # 生产环境必须禁用 clean

  jpa:
    hibernate:
      ddl-auto: validate         # 仅校验，不修改
```

```sql
-- db/migration/V1__init_schema.sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INT,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
);

-- db/migration/V2__add_status_column.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'ACTIVE';
```

### 11.1.3 Liquibase 集成

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    drop-first: false            # 生产环境必须 false
```

## 11.2 配置文件 Profile 分离

### 11.2.1 完整的多环境配置

**application.yml（公共配置）**：

```yaml
spring:
  application:
    name: hibernate-spring-demo

  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc:
          batch_size: 20
          time_zone: Asia/Shanghai
        order_inserts: true
        order_updates: true
        default_batch_fetch_size: 16

  flyway:
    enabled: true
    validate-on-migrate: true
```

**application-dev.yml（开发环境）**：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb_dev
    username: dev_user
    password: dev_pass
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 5
      pool-name: HikariPool-Dev

  jpa:
    show-sql: true
    format-sql: true
    hibernate:
      ddl-auto: update

  flyway:
    clean-disabled: false  # 开发环境允许 clean

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    com.example: DEBUG
```

**application-prod.yml（生产环境）**：

```yaml
spring:
  datasource:
    url: jdbc:mysql://prod-db-host:3306/mydb_prod?useSSL=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 5000
      idle-timeout: 300000
      max-lifetime: 600000
      leak-detection-threshold: 30000
      pool-name: HikariPool-Prod

  jpa:
    show-sql: false
    format-sql: false
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        generate_statistics: false

  flyway:
    clean-disabled: true

logging:
  level:
    org.hibernate.SQL: WARN
    com.example: INFO
```

### 11.2.2 环境变量覆盖敏感配置

```bash
# Kubernetes / Docker Compose 环境变量
export DB_USERNAME=prod_app_user
export DB_PASSWORD=SecureP@ssw0rd2024
export JAVA_OPTS="-Xms512m -Xmx2g"
```

### 11.2.3 敏感配置加密（Jasypt）

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

```yaml
# 加密后的配置
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: ENC(3nG7kT6x0q8HyP6OFrG1Cw==)
    password: ENC(AZ6RLw5ntJCkX5JLRm0RTyBJ6N1yq7pO)

jasypt:
  encryptor:
    algorithm: PBEWithMD5AndDES
    iv-generator-classname: org.jasypt.iv.NoIvGenerator
    password: ${JASYPT_ENCRYPTOR_PASSWORD}  # 加密密钥从环境变量读取
```

加密命令：

```bash
# 对敏感值进行加密
java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI \
  input="my-secret-password" \
  password="master-key" \
  algorithm=PBEWithMD5AndDES
```

---

# 12. 结语

Spring 与 Hibernate 的集成经历了从模板封装到原生注入再到自动配置的演进。现代 Spring Boot 项目中，开发者只需关注以下几点即可构建健壮的持久化层：

1. **依赖声明**：`spring-boot-starter-data-jpa` 一站式引入所有必要组件
2. **配置管理**：通过 `application.yml` 按 Profile 分离各环境配置
3. **Repository 层**：利用 Spring Data JPA 的命名查询、`@Query`、`@EntityGraph` 减少模板代码
4. **事务边界**：合理使用 `@Transactional` 控制 Session 生命周期
5. **OSIV 策略**：生产环境关闭 OSIV，在 Service 层完成数据组装
6. **测试保障**：`@DataJpaTest` 提供开箱即用的持久化测试支持
7. **运维安全**：生产环境 `ddl-auto=none`，配合 Flyway/Liquibase 实现可信赖的 Schema 变更

合理地理解和运用这些集成机制，可以使 Hibernate 在 Spring 生态中发挥最大的效能，同时避免常见的性能陷阱和维护风险。
