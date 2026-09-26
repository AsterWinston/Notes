
# 1. 环境要求

| 组件 | 版本要求 |
|------|---------|
| JDK | 8+（ShardingSphere-JDBC）/ 11+（ShardingSphere-Proxy） |
| MySQL | 5.7+ / 8.0+ |
| Maven | 3.6+（JDBC 模式） |
| ZooKeeper | 3.6+（Proxy 集群模式可选） |

# 2. ShardingSphere-JDBC 入门

## 2.1 Maven 依赖

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>5.4.1</version>
</dependency>
```

## 2.2 YAML 配置

```yaml
# resources/shardingsphere.yaml
dataSources:
  ds0:
    url: jdbc:mysql://localhost:3306/demo_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 10

  ds1:
    url: jdbc:mysql://localhost:3306/demo_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds${0..1}.t_order_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: database_inline
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: table_inline
        keyGenerateStrategy:
          column: order_id
          keyGeneratorName: snowflake

    shardingAlgorithms:
      database_inline:
        type: INLINE
        props:
          algorithm-expression: ds${user_id % 2}
      table_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}

    keyGenerators:
      snowflake:
        type: SNOWFLAKE
        props:
          worker-id: 1

props:
  sql-show: true
```

## 2.3 Java 代码

```java
// 1. 创建 ShardingSphereDataSource
public class DemoApplication {
    public static void main(String[] args) throws SQLException {
        // 加载 YAML 配置
        YamlShardingSphereDataSourceFactory dataSourceFactory =
            new YamlShardingSphereDataSourceFactory(
                DemoApplication.class.getResourceAsStream("/shardingsphere.yaml"));

        DataSource dataSource = dataSourceFactory.createDataSource();

        // 2. 插入数据
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute(
                "INSERT INTO t_order (user_id, address_id, status) VALUES (1, 1, 'OK')");
            conn.createStatement().execute(
                "INSERT INTO t_order (user_id, address_id, status) VALUES (2, 1, 'OK')");
        }

        // 3. 查询数据
        try (Connection conn = dataSource.getConnection()) {
            ResultSet rs = conn.createStatement()
                .executeQuery("SELECT * FROM t_order");
            while (rs.next()) {
                System.out.println(rs.getLong("order_id"));
            }
        }
    }
}
```

# 3. ShardingSphere-Proxy 入门

## 3.1 Docker 部署

```bash
docker run -d \
  --name shardingsphere-proxy \
  -p 3307:3307 \
  -v /path/to/conf:/opt/shardingsphere-proxy/conf \
  apache/shardingsphere-proxy:5.4.1
```

## 3.2 server.yaml

```yaml
# conf/server.yaml
authority:
  users:
    - user: root@%
      password: root
    - user: sharding
      password: sharding
  privilege:
    type: ALL_PERMITTED

mode:
  type: Standalone
  repository:
    type: JDBC

props:
  sql-show: true
  proxy-frontend-database-protocol-type: MySQL
```

## 3.3 config-sharding.yaml

```yaml
# conf/config-sharding.yaml
databaseName: sharding_db

dataSources:
  ds0:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root

  ds1:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds${0..1}.t_order_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: database_inline
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: table_inline
    shardingAlgorithms:
      database_inline:
        type: INLINE
        props:
          algorithm-expression: ds${user_id % 2}
      table_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}
```

## 3.4 DistSQL 配置（推荐）

```sql
-- 连接 Proxy
mysql -h 127.0.0.1 -P 3307 -u root -p

-- 1. 注册数据源
REGISTER STORAGE UNIT ds0 (
    URL="jdbc:mysql://127.0.0.1:3306/demo_ds_0",
    USER="root", PASSWORD="root"
);

REGISTER STORAGE UNIT ds1 (
    URL="jdbc:mysql://127.0.0.1:3306/demo_ds_1",
    USER="root", PASSWORD="root"
);

-- 2. 创建分片规则
CREATE SHARDING TABLE RULE t_order (
    DATANODES("ds${0..1}.t_order_${0..1}"),
    DATABASE_STRATEGY(TYPE="standard", SHARDING_COLUMN=user_id,
        SHARDING_ALGORITHM(TYPE(NAME="inline", PROPERTIES("algorithm-expression"="ds${user_id % 2}")))),
    TABLE_STRATEGY(TYPE="standard", SHARDING_COLUMN=order_id,
        SHARDING_ALGORITHM(TYPE(NAME="inline", PROPERTIES("algorithm-expression"="t_order_${order_id % 2}"))))
);
```

# 4. 验证

```sql
-- 插入数据
INSERT INTO t_order (user_id, order_id, address_id, status)
VALUES (1, 1, 1, 'OK');     -- → ds1.t_order_1
INSERT INTO t_order (user_id, order_id, address_id, status)
VALUES (2, 2, 1, 'OK');     -- → ds0.t_order_0

-- 查询验证
SELECT * FROM t_order;
-- Logic SQL: SELECT * FROM t_order
-- Actual SQL: ds0 :: SELECT * FROM t_order_0
-- Actual SQL: ds0 :: SELECT * FROM t_order_1
-- Actual SQL: ds1 :: SELECT * FROM t_order_0
-- Actual SQL: ds1 :: SELECT * FROM t_order_1

-- 精确查询
SELECT * FROM t_order WHERE order_id = 1;
-- Actual SQL: ds1 :: SELECT * FROM t_order_1 WHERE order_id = 1
```
