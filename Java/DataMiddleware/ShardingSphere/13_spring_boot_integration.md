# 1. 简介

ShardingSphere-JDBC 与 Spring Boot 的集成方式在 5.x 系列内经历过一次**彻底的重构**：5.2.1 及之前通过 `shardingsphere-jdbc-core-spring-boot-starter` 提供 `spring.shardingsphere.*` 配置绑定；5.3.0 起 starter 被官方废弃，5.4/5.5.x 统一改为**标准 JDBC Driver 方式**——`spring.datasource` 照常配置，驱动换成 `ShardingSphereDriver`、URL 指向规则 YAML。理解这条演进线是集成前提：**5.5.x 没有任何 ShardingSphere 专属 Spring 命名空间**，规则写在独立 config.yaml，Spring 只见一个普通数据源。

支持 JPA、MyBatis、MyBatis-Plus 等 ORM 无缝集成，本质是 ShardingSphere 完整实现了 JDBC 4.2 规范，对上层框架透明。

# 2. 依赖

## 2.1 5.5.x 标准依赖（Driver 方式）

```xml
<!-- ShardingSphere 5.5.x：只有一个坐标，starter 已不存在 -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc</artifactId>
    <version>5.5.2</version>
</dependency>

<!-- 连接池：Boot 默认 HikariCP，无需显式引入 -->
<!-- MyBatis-Plus 集成示例（Spring Boot 3 必须用 spring-boot3 专用 starter） -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.9</version>
</dependency>
```

## 2.2 旧版 starter（5.2.1 以前，考古用）

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>5.2.1</version>
</dependency>
```

**为什么官方要砍掉 starter**：starter 时代 Spring 命名空间与 ShardingSphere YAML 规则结构几乎一比一映射，等于同一套配置维护两份 schema，版本迭代互相拖累；且 starter 与特定 Spring Boot 大版本绑定，Boot 2→Boot 3 的 jakarta 迁移让每个中间件都要发双份 starter。改成 Driver 方式后，ShardingSphere 与 Spring 版本彻底解耦——**Spring Boot 3 下不需要任何适配动作，因为它根本不需要被 Spring 感知**，这也是 Boot 3 适配问题的一句话答案。

> **注意**：`shardingsphere-jdbc` 传递依赖 Groovy（行表达式 `$->{}` 引擎）、Netty、calcite（联邦查询）等，与老项目公共库版本冲突时需 `<exclusions>` 精细处理，升级前先 `mvn dependency:tree` 比对。

# 3. application.yml 配置

## 3.1 Spring 侧：只剩标准 datasource 四件套

```yaml
spring:
  application:
    name: sharding-demo
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:config.yaml
    username: # 留空，凭据在 config.yaml 各数据源内
    password:
```

规则来源三种：`classpath:config.yaml`（jar 内）、`file:/data/sharding/config.yaml`（挂载文件）、`absent!`（先建连接再推送，配合 DistSQL 场景）。

## 3.2 config.yaml：分片 + 读写分离 + 加密组合配置

5.5.x 的规则 YAML 顶层是 `mode / dataSources / rules / props` 四段，规则之间可自由叠加——同一张表先加密、再分片、每个分片库内部再做读写分离，链路顺序是**加密改写 → 分片路由 → 主从路由**：

```yaml
mode:
  type: Standalone
  repository:
    type: JDBC

dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_0
    username: root
    password: root
  ds0_replica:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3307/demo_ds_0
    username: root
    password: root
  ds1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_1
    username: root
    password: root
  ds1_replica:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3307/demo_ds_1
    username: root
    password: root

rules:
- !SHARDING
  tables:
    t_order:
      actualDataNodes: pr_$->{0..1}.t_order_$->{0..1}
      databaseStrategy:
        standard:
          shardingColumn: user_id
          shardingAlgorithmName: db-inline
      tableStrategy:
        standard:
          shardingColumn: order_id
          shardingAlgorithmName: tbl-inline
      keyGenerateStrategy:
        column: order_id
        keyGeneratorName: snowflake
  shardingAlgorithms:
    db-inline:
      type: INLINE
      props:
        algorithm-expression: pr_$->{user_id % 2}
    tbl-inline:
      type: INLINE
      props:
        algorithm-expression: t_order_$->{order_id % 2}
  keyGenerators:
    snowflake:
      type: SNOWFLAKE

- !READWRITE_SPLITTING
  dataSources:
    pr_0:
      staticStrategy:
        writeDataSourceName: ds0
        readDataSourceNames:
          - ds0_replica
      loadBalancerName: round-robin
    pr_1:
      staticStrategy:
        writeDataSourceName: ds1
        readDataSourceNames:
          - ds1_replica
      loadBalancerName: round-robin
  loadBalancers:
    round-robin:
      type: ROUND_ROBIN

- !ENCRYPT
  encryptors:
    aes_encryptor:
      type: AES
      props:
        aes-key-value: 123456abc123456a
        digest-algorithm-name: sha-inventory
  tables:
    t_order:
      columns:
        receiver_phone:
          cipher:
            name: receiver_phone_encrypted
            encryptorName: aes_encryptor

props:
  sql-show: true
```

三个易错点：**①** 读写分离的 `dataSources` 名（pr_0/pr_1）必须与分片 `actualDataNodes` 里的库名对齐——分片路由先落到 pr_0，再由读写分离规则展开成 ds0/ds0_replica；**②** 5.5.x 的数据源属性名是 `dataSourceClassName`/`driverClassName`/`jdbcUrl`（驼峰），不是旧 starter 时代的 `type`/`driver-class-name`/`jdbc-url`，老配置直接搬会启动报 "Can not find driver"；**③** 加密列改写后 `SELECT receiver_phone` 实际执行的是 `SELECT receiver_phone_encrypted AS receiver_phone`，与 MyBatis 结果映射兼容，与原生 JDBC 按下标取值也兼容，但按**列名**查加密辅助列的场景要注意改名。

> **注意**：YAML 行表达式必须用 `$->{}`。Spring Boot 的 `application.yml` 里 `${}` 会被 Spring 占位符解析器抢走，而 config.yaml 是 ShardingSphere 自己加载的，**不经过 Spring 占位符体系**——想在 config.yaml 里引用环境变量写 `${DB_PASSWORD}` 反而是不行的，需要用 `!!SYSTEM_ENV` 或在启动前做模板渲染，这是迁移旧工程最常见的坑。

# 4. MyBatis-Plus 集成

```java
@Entity
@TableName("t_order")
public class Order {
    @TableId(type = IdType.ASSIGN_ID)  // 见下方警示：与 ShardingSphere 主键生成二选一
    private Long orderId;
    private Long userId;
    private String status;
}

@Mapper
public interface OrderMapper extends BaseMapper<Order> { }

@Service
public class OrderService {
    @Autowired
    private OrderMapper orderMapper;

    @Transactional
    public void createOrder(Order order) {
        // MyBatis-Plus 自动使用 ShardingSphere 数据源
        orderMapper.insert(order);
        // ShardingSphere 自动分片路由 + 主键生成
    }

    public Order getById(Long orderId) {
        return orderMapper.selectById(orderId);
    }
}
```

**主键生成的冲突规则**：如果实体保留 `IdType.ASSIGN_ID`，MyBatis-Plus 会在 SQL 里显式写入 order_id，ShardingSphere 检测到主键列已有值就不再触发 `keyGenerateStrategy`，两套雪花互不相干——功能正确但治理混乱（MP 的 workerId 与 SS 的 workerId 独立，重复概率翻倍）。**纪律是二选一**：要么 MP 用 `IdType.INPUT` 交给 ShardingSphere 生成，要么干脆不配 `keyGenerateStrategy`。另外 `selectById(orderId)` 只带分片表键不带分片库键时，会触发**全库路由、单库内精确定位**，性能上是 N 次点查而非一次，跨库维度尽量带上 user_id。

# 5. JPA 集成

```java
@Entity
@Table(name = "t_order")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long orderId;
    private Long userId;
    private String status;
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> { }
```

**注意**：使用 JPA 时，`@GeneratedValue` 与 ShardingSphere 的 `keyGenerateStrategy` 协同工作，ShardingSphere 优先。

## 5.1 JPA 的三个必配项

Driver 方式下 Hibernate 拿到的 `DatabaseMetaData` 是 ShardingSphere 包装后的产品名，**方言自动探测会失败或探测错**，必须显式指定：

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.MySQLDialect
    hibernate:
      ddl-auto: none   # 分片物理表由 DBA/迁移工具建，禁止 Hibernate 自动建表
  properties:
    hibernate.jdbc.batch_size: 50
```

`ddl-auto` 设成 `update` 是危险动作：Hibernate 只会对着"逻辑表 t_order"生成 DDL 并随机落到某个分片，物理表结构从此不一致。兼容矩阵如下：

| 框架 | 版本要求 | 关键点 |
|------|---------|--------|
| MyBatis | 任意（Boot 3 需 mybatis-spring 3.x） | 完全兼容，PageHelper 慎用（见 6.2） |
| MyBatis-Plus | Boot 2 用 boot-starter；Boot 3 用 spring-boot3-starter | 主键策略二选一；逻辑删除无冲突 |
| Spring Data JPA | Hibernate 6.x（Boot 3） | 显式方言 + ddl-auto=none |
| Spring JDBC / JdbcTemplate | 任意 | 完全透明 |
| XA 场景（Atomikos/JTA） | spring-boot-starter-jta-atomikos | Boot 的 JtaTransactionManager 与 SS 事务规则叠加时以 SS 的 !TRANSACTION 为准 |

# 6. 多数据源整合

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource shardingDataSource() {
        // 自动加载 shardingsphere 配置
        return ShardingSphereDataSourceFactory.createDataSource();
    }

    @Bean
    @Qualifier("otherDb")
    public DataSource otherDataSource() {
        // 非分片的普通数据源
        return new HikariDataSource(config);
    }
}
```

5.5.x 的工厂方法签名是 `ShardingSphereDataSourceFactory.createDataSource(Map<String, DataSource>, Collection<RuleConfiguration>, Properties)`（代码构建规则）与 `createDataSource("file:/config.yaml")`（文件构建规则）。**编程式与 Driver 式混用时的坑**：Driver 方式下 Spring 的 `DataSourceBuilder` 已经建了一个 Hikari 外壳池包住 ShardingSphereConnection（双池嵌套，外壳池参数几乎不起作用但占连接统计），编程式则直接暴露 SS DataSource——同一应用里两种都建，Actuator 的 health/metrics 会因多个 DataSource bean 报错，需要 `management.endpoint.health.components.db` 逐个放行或关闭自动配置 `DataSourcePoolMetricsAutoConfiguration`。

## 6.1 与 dynamic-datasource 共存

baomidou `dynamic-datasource-spring-boot-starter` 与 ShardingSphere 都是"数据源管理者"，共存模型应当是**外层动态路由、内层分片**：把 ShardingSphere DataSource 注册为 dynamic-datasource 的一个 group（如 `@DS("sharding")`），而不是两个各自独立再靠 `@Primary` 抢。**坑点**：`@DS` 切换与 `@Transactional` 的绑定发生在事务开启时，方法内先开事务后切 DS 是无效的；以及 dynamic-datasource 的 seata 模式若开启，会再包一层 `DataSourceProxy`，与 SS 的 `!TRANSACTION` 规则叠加成双重代理，事务类型判定错乱，只能保留一层。

## 6.2 PageHelper 与归并的冲突

PageHelper 的工作方式是把 SQL 改写成 `LIMIT` 并自动生成 `SELECT COUNT(*)` 包裹查询，它与 ShardingSphere 的分页归并叠加后：count 语句被广播到全分片求和（结果正确但代价大），而深分页 offset 被每个分片放大取 offset+limit 行——**两层改写互相不可见，问题只能靠约定规避**：分片表的分页交给 ShardingSphere（SQL 写标准 LIMIT），PageHelper 只用于单库表；或对分片表禁用 count 查询、用 ES 提供总数。

# 7. 动态配置加载

```java
// 从注册中心动态加载配置（Cluster 模式）
@Configuration
public class DynamicShardingConfig {

    @Bean
    public DataSource dataSource() {
        ModeConfiguration modeConfig = new ModeConfiguration(
            "Cluster",
            new ClusterPersistRepositoryConfiguration(
                "ZooKeeper",                    // repository type
                "governance-sharding-db",       // namespace
                "zk1:2181,zk2:2181",            // server-lists
                new Properties() {{
                    setProperty("overwrite", "true"); // 首次冷启动用，稳态改 false
                }}
            )
        );
        return ShardingSphereDataSourceFactory.createDataSource(
            modeConfig, dataSources, ruleConfigs, props);
    }
}
```

Cluster 模式下的 JDBC 实例加入注册中心后，**规则变更由注册中心 watch 推送，无需重启即热生效**——这是对"JDBC 改配置必须重启"旧印象的修正（前提是集群模式 + 远程 repository）。Driver 方式同样支持，把 config.yaml 的 `mode` 段写成 Cluster 即可。`overwrite: true` 表示本地 YAML 覆盖远端规则，只应在集群首次构建或灾难重建时打开，稳态必须关闭，否则新实例上线就把别人用 DistSQL 改过的规则打回旧版。

# 8. 测试配置

```yaml
# application-test.yml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:config-test.yaml
```

```yaml
# config-test.yml —— 单库 H2，保留分片语义（列名按 5.5.x 驼峰规范）
mode:
  type: Standalone
  repository:
    type: JDBC
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: org.h2.Driver
    jdbcUrl: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=MySQL
    username: sa
    password:
rules:
- !SHARDING
  tables:
    t_order:
      actualDataNodes: ds0.t_order_$->{0..1}
      tableStrategy:
        standard:
          shardingColumn: order_id
          shardingAlgorithmName: inline
  shardingAlgorithms:
    inline:
      type: INLINE
      props:
        algorithm-expression: t_order_$->{order_id % 2}
props:
  sql-show: false
```

> **行业实践**：测试环境用 H2 跑通"分片语义"（路由、改写、归并逻辑都被执行），生产用真实 MySQL——同一份 config.yaml 模板化生成，仅替换 dataSources 段。H2 需开 `MODE=MySQL`，否则反引号、`LIMIT` 方言解析失败。单测里断言"路由对不对"可直接用 `ShardingSphereDataSource` 拿连接后 `PREVIEW` 不可用（那是 DistSQL），改为开启 sql-show 断言日志，或调用 `ShardingRouteEngine` 级别单测。

# 9. 最佳实践

1. **使用标准 Driver 接入**：5.3+ 不要再找 starter，`spring.datasource.url` 指向 config.yaml 即可
2. **YAML 行表达式用 `$->{}`**：config.yaml 不经过 Spring 占位符体系，`${}` 不会按预期工作
3. **事务管理**：跨分片自动升级为 XA，需引入对应依赖；Boot 的 JTA 自动配置与 SS 事务规则只留一层
4. **连接池选 HikariCP**：性能最好；注意 Driver 方式存在外壳池嵌套，外层池容量给足避免二次排队
5. **配置版本管理**：config.yaml 纳入 Git，DistSQL 脚本化管理
6. **ORM 兼容性自检清单**：JPA 显式方言 + ddl-auto=none；MyBatis-Plus 主键策略二选一；PageHelper 与分片分页不叠加

# 10. 小结

1. **5.3+ 接入方式统一为 ShardingSphereDriver**：spring.datasource 指向 config.yaml，无专属 starter 命名空间
2. **规则与 Spring 解耦**：Boot 2/Boot 3 零适配成本，代价是 config.yaml 不走 Spring 占位符体系
3. **ORM 兼容三原则**：JPA 显式方言+禁 ddl-auto、MP 主键策略二选一、PageHelper 不与分片分页叠加
4. **多数据源共存看层级**：dynamic-datasource 在外、SS 在内；Seata 代理在外、SS 在内，禁止双 TM
5. **测试环境用 H2 跑通分片语义**，与生产同一份 YAML 模板生成