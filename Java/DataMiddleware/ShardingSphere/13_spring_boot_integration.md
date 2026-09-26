
# 1. 简介

ShardingSphere-JDBC 通过 `shardingsphere-jdbc-core-spring-boot-starter` 提供 Spring Boot 自动配置能力，支持与 JPA、MyBatis、MyBatis-Plus 等 ORM 框架无缝集成。

# 2. 依赖

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>5.4.1</version>
</dependency>

<!-- 数据库连接池 -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
</dependency>

<!-- MyBatis-Plus 集成示例 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

# 3. application.yml 配置

```yaml
spring:
  application:
    name: sharding-demo

  # ShardingSphere 数据源配置
  shardingsphere:
    mode:
      type: Standalone
      repository:
        type: JDBC

    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/demo_ds_0
        username: root
        password: root
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/demo_ds_1
        username: root
        password: root

    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds$->{0..1}.t_order_$->{0..1}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: database-inline
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table-inline
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake

        sharding-algorithms:
          database-inline:
            type: INLINE
            props:
              algorithm-expression: ds$->{user_id % 2}
          table-inline:
            type: INLINE
            props:
              algorithm-expression: t_order_$->{order_id % 2}

        key-generators:
          snowflake:
            type: SNOWFLAKE

    props:
      sql-show: true
```

# 4. MyBatis-Plus 集成

```java
@Entity
@TableName("t_order")
public class Order {
    @TableId(type = IdType.ASSIGN_ID)  // ShardingSphere 主键生成
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

# 7. 动态配置加载

```java
// 从注册中心动态加载配置（Cluster 模式）
@Configuration
public class DynamicShardingConfig {

    @Bean
    public DataSource dataSource() {
        ModeConfiguration modeConfig = new ModeConfiguration(
            "Cluster",
            new ClusterRepositoryConfiguration("ZooKeeper",
                new Properties() {{
                    setProperty("server-lists", "zk1:2181,zk2:2181");
                    setProperty("namespace", "shardingsphere");
                }}
            )
        );
        return ShardingSphereDataSourceFactory.createDataSource(modeConfig);
    }
}
```

# 8. 测试配置

```yaml
# application-test.yml
spring:
  shardingsphere:
    datasource:
      names: ds0
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: org.h2.Driver
        jdbc-url: jdbc:h2:mem:testdb
        username: sa
        password:

    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds0.t_order_$->{0..1}
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: inline
        sharding-algorithms:
          inline:
            type: INLINE
            props:
              algorithm-expression: t_order_$->{order_id % 2}

    props:
      sql-show: false
```

# 9. 最佳实践

1. **使用 Spring Boot Starter**：避免手动构造 DataSource
2. **YAML 格式用 `$->{}`**：Spring Boot YAML 不支持 `${}`，需用行表达式
3. **事务管理**：跨分片自动升级为 XA，需引入对应依赖
4. **连接池选 HikariCP**：性能最好
5. **配置版本管理**：YAML 配置纳入 Git，DistSQL 脚本化管理
