本篇把 Spring Boot 3 + ShardingSphere-JDBC 5.5.x 的最小可运行环境跑通：建库建表、引依赖、写一份可直接粘贴的 `shardingsphere.yaml`、用 JDBC 驱动接入、启动后验证逻辑 SQL 与物理 SQL 的对应关系。配置全部采用 **5.x 新格式**（顶层 `dataSources` + `rules` + `props`），与网上大量 4.x 老教程的 `sharding.jdbc.datasource.*` 写法完全不同，照抄旧格式一定启动失败，这一点贯穿全篇。

# 1. 环境要求

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 8+（JDBC）/ 11+（Proxy） | Spring Boot 3 需 **17+** |
| Spring Boot | 3.x | Jakarta EE 9+，注意 XA 兼容限制 |
| ShardingSphere-JDBC | 5.5.x | artifact 为 `shardingsphere-jdbc` |
| MySQL | 5.7+ / 8.0+ | 需 `mysql-connector-j` |
| 连接池 | HikariCP（默认） | 也可 Druid |
| ZooKeeper | 3.6+ | 仅 Proxy Cluster 模式需要 |

> **注意**：ShardingSphere 官方明确 **XA 分布式事务在 Spring Boot OSS 3（及 Quarkus 3 等 Jakarta EE 9+ 框架）上尚未就绪**。在 Boot 3 上想用 XA，需要引入 jakarta 版事务实现依赖做适配，或先用 LOCAL/BASE 事务。读写分离、分片、加密等功能不受此限制。

# 2. ShardingSphere-JDBC 入门

## 2.1 Maven 依赖

```xml
<dependencies>
    <!-- 5.x 统一入口 artifact，内含内核 + 各功能插件 -->
    <dependency>
        <groupId>org.apache.shardingsphere</groupId>
        <artifactId>shardingsphere-jdbc</artifactId>
        <version>5.5.1</version>
    </dependency>
    <!-- MySQL 驱动（Boot 3 用新坐标） -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- 若用 SNOWFLAKE 之外的主键、或 XA，再按需引入对应插件 -->
</dependencies>
```

## 2.2 准备物理库表

ShardingSphere 只负责路由与改写，**物理库表必须自己先建好**，它不会自动 DDL。以"2 库 2 表、分库键 user_id、分表键 order_id"为例：

```sql
-- 在 demo_ds_0 与 demo_ds_1 两个库中分别执行
-- 库：demo_ds_0 / demo_ds_1
CREATE TABLE t_order_0 (
    order_id BIGINT PRIMARY KEY,
    user_id  INT NOT NULL,
    status   VARCHAR(16)
);
CREATE TABLE t_order_1 (
    order_id BIGINT PRIMARY KEY,
    user_id  INT NOT NULL,
    status   VARCHAR(16)
);
```

> **行业实践**：建表脚本纳入版本管理（Flyway/Liquibase），且**每个分库都执行同一份 DDL**；逻辑表结构变更要人工同步到所有物理分片，漏同步一个分片会在路由到该分片时报"Unknown column"。

## 2.3 YAML 配置（5.x 格式，可直接粘贴）

```yaml
# src/main/resources/shardingsphere.yaml
mode:
  type: Standalone          # 单机模式，无需注册中心
  repository:
    type: JDBC              # 元数据持久化到内嵌 H2

dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_0?serverTimezone=Asia/Shanghai&useSSL=false&allowPublicKeyRetrieval=true
    username: root
    password: root
    maximumPoolSize: 10
    minimumIdle: 1
    connectionTimeout: 30000
  ds1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_1?serverTimezone=Asia/Shanghai&useSSL=false&allowPublicKeyRetrieval=true
    username: root
    password: root
    maximumPoolSize: 10
    minimumIdle: 1

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds${0..1}.t_order_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: db_inline
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: tbl_inline
        keyGenerateStrategy:
          column: order_id
          keyGeneratorName: snowflake
    shardingAlgorithms:
      db_inline:
        type: INLINE
        props:
          algorithm-expression: ds${user_id % 2}
      tbl_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}
    keyGenerators:
      snowflake:
        type: SNOWFLAKE
        props:
          worker-id: 1

props:
  sql-show: true            # 打印逻辑 SQL 与实际 SQL，开发期务必打开
```

**逐块解释关键写法**：

- `mode.type: Standalone`：5.x 用 Mode 取代了 4.x 的治理中心配置，Standalone 用 JDBC（H2）存元数据，适合本地与单机；生产多实例改 Cluster + ZooKeeper（见 10_governance）。
- `dataSources` 是**扁平结构**：每个数据源直接写 `dataSourceClassName`/`driverClassName`/`jdbcUrl`/`username`/`password` + 连接池原生属性（Hikari 的 `maximumPoolSize` 等），不再有 4.x 的 `type: HikariDataSource` + `props:` 嵌套。
- `actualDataNodes: ds${0..1}.t_order_${0..1}`：行表达式内联，展开为 4 个数据节点 `ds0.t_order_0/ds0.t_order_1/ds1.t_order_0/ds1.t_order_1`。
- **5.x 分片算法新写法**：不再写 `databaseShardingStrategy/complexKeysShardingAlgorithm` 那一套，统一为 `databaseStrategy.standard.shardingColumn + shardingAlgorithmName`，算法在独立的 `shardingAlgorithms` 段声明，通过名字引用。这是从 4.x 迁移最容易踩的格式差异。

## 2.4 通过 Spring Boot 3 接入

Boot 3 下最省事的方式是把它当成**一个 JDBC 驱动**用，不额外定义 DataSource Bean：

```properties
# application.properties
spring.datasource.driver-class-name=org.apache.shardingsphere.driver.ShardingSphereDriver
spring.datasource.url=jdbc:shardingsphere:classpath:shardingsphere.yaml
spring.datasource.username=
spring.datasource.password=
```

`ShardingSphereDriver` 读取 `jdbc:shardingsphere:` 后的地址（`classpath:` 或 `file:`）加载那份 YAML，构造出内部数据源。之后 `@Autowired DataSource` 或 JPA/MyBatis 拿到的就是被 ShardingSphere 增强的连接，业务代码零改动。

## 2.5 纯 Java（非 Boot）方式

```java
public class DemoApplication {
    public static void main(String[] args) throws SQLException {
        // 方式一：直接喂 YAML 文件
        Map<String, DataSource> dataSourceMap = createDataSources();
        Collection<ShardingSphereRule> rules = List.of(createShardingRule());
        DataSource dataSource = ShardingSphereDataSourceFactory
            .createDataSource(dataSourceMap, rules, new Properties());

        // 方式二：YamlShardingSphereDataSourceFactory 从 yaml 反序列化
        // DataSource ds = new YamlShardingSphereDataSourceFactory()
        //     .createDataSource(resourceBytes);

        try (Connection conn = dataSource.getConnection();
             Statement st = conn.createStatement()) {
            st.execute("INSERT INTO t_order (user_id, status) VALUES (1, 'OK')");
            ResultSet rs = st.executeQuery("SELECT * FROM t_order");
            while (rs.next()) {
                System.out.println(rs.getLong("order_id"));
            }
        }
    }
}
```

# 3. ShardingSphere-Proxy 入门

Proxy 不依赖 Java 应用，任何 MySQL 客户端都能连。适合多语言、或想让 DBA 用 SQL 直接运维的场景。

## 3.1 Docker 部署

```bash
# 挂载 conf 目录（放 server.yaml 等），下载 MySQL 驱动放入 ext-lib
docker run -d \
  --name shardingsphere-proxy \
  -p 3307:3307 \
  -v /path/to/conf:/opt/shardingsphere-proxy/conf \
  -v /path/to/mysql-driver:/opt/shardingsphere-proxy/ext-lib \
  apache/shardingsphere-proxy:5.5.1
```

> **注意**：Proxy 镜像**不内置 MySQL JDBC 驱动**（License 原因），必须把 `mysql-connector-j` jar 放进 `extensions/`（或挂载到 `ext-lib`）再启动，否则连不上后端库并报找不到驱动类。这是 Proxy 入门最常见的坑。

## 3.2 server.yaml

```yaml
# conf/server.yaml（Proxy 全局配置，不含具体分库分表规则）
authority:
  users:
    - user: root@%
      password: root
    - user: sharding@%
      password: sharding
      admin: true                   # 管理员可执行全部 DistSQL
  privilege:
    type: ALL_PERMITTED             # 默认；推荐改用 DATABASE_PERMITTED（见 10_governance）

mode:
  type: Standalone
  repository:
    type: JDBC

props:
  sql-show: true
```

## 3.3 DistSQL 建规则（推荐，免改文件重启）

连上 Proxy 后，用 SQL 在线建库建规则，无需编辑 `config-sharding.yaml`：

```sql
-- 连 Proxy：mysql -h 127.0.0.1 -P 3307 -u root -p
-- 1. 创建逻辑库并切上下文
CREATE DATABASE sharding_db;
USE sharding_db;

-- 2. 注册存储单元（5.x 术语，取代旧 data source）
REGISTER STORAGE UNIT ds0 (
    HOST="localhost", PORT=3306,
    DB="demo_ds_0", USER="root", PASSWORD="root",
    PROPERTIES("maximumPoolSize"="10")
);
REGISTER STORAGE UNIT ds1 (
    HOST="localhost", PORT=3306,
    DB="demo_ds_1", USER="root", PASSWORD="root"
);

-- 3. 创建自动分片规则（hash_mod 无需 SQL 携带分片键）
CREATE SHARDING TABLE RULE t_order (
    STORAGE_UNITS(ds0,ds1),
    SHARDING_COLUMN=user_id,
    TYPE(NAME="hash_mod", PROPERTIES("sharding-count"="2")),
    KEY_GENERATE_STRATEGY(COLUMN=order_id, TYPE(NAME="snowflake"))
);
```

DistSQL 完整语法与更多规则类型见 9_distsql；这里只证明"Proxy 全程可以不碰 yaml"。

# 4. 启动验证

无论 JDBC 还是 Proxy，开 `sql-show: true` 后，一次全表查询日志会打印逻辑 SQL 与实际 SQL 的映射：

```text
Logic SQL: SELECT * FROM t_order
Actual SQL: ds0 ::: SELECT * FROM t_order_0
Actual SQL: ds0 ::: SELECT * FROM t_order_1
Actual SQL: ds1 ::: SELECT * FROM t_order_0
Actual SQL: ds1 ::: SELECT * FROM t_order_1
```

带分片键的精确查询只落到单个节点，这正是分片收益的来源：

```sql
-- 插入：user_id=1, order_id 由 SNOWFLAKE 生成
INSERT INTO t_order (user_id, status) VALUES (1, 'OK');
-- → 改写为 INSERT INTO t_order? (order_id, user_id, status) VALUES (?,1,'OK')
--   并按 order_id % 2 落到某张 t_order；按 user_id % 2 落到 ds1

-- 精确查询：分片键都在
SELECT * FROM t_order WHERE user_id = 1 AND order_id = 8888888888;
-- Actual SQL: ds1 ::: SELECT * FROM t_order_0 WHERE user_id = 1 AND order_id = 8888888888
--（仅 1 条物理 SQL → 单库单表，最优路径）
```

**验证清单**：

- 启动无 `Failed to create data source` / `Cannot load JDBC driver`：确认驱动 jar 到位、`driverClassName` 正确。
- 打印的 `Actual SQL` 数量 = 期望路由分片数：全表查询应为 `分库数 × 分表数`，带全分片键应为 1。
- 插入返回自增/雪花 ID：`getGeneratedKeys` 能取到 order_id，说明主键改写生效。
- 跨分片查询不报错且结果完整：证明归并引擎工作正常。

> **设计考量**：
>
> - **开发期一律 `sql-show: true`**：90% 的"结果不对/性能差"问题，看一眼 Actual SQL 就知道是不是路由降级成全分片扫描了。生产环境可关掉或配合 `sql-simple`（只打 SQL 不打参数）降低日志量。
> - **先 JDBC 验证再上生产规模**：用 Standalone + 少量分片把逻辑跑通，再切 Cluster 模式与真实分片数，比一开始就上 Proxy 集群更容易定位配置错误。

# 5. 小结

| 步骤 | 要点 |
|------|------|
| 依赖 | 5.x 用 `shardingsphere-jdbc`（不是 `shardingsphere-jdbc-core`） |
| 建表 | 物理分片表自己建，ShardingSphere 不做 DDL |
| YAML | 顶层 `dataSources` + `rules(!SHARDING)` + `props`，扁平数据源写法 |
| 分片算法 | `databaseStrategy.standard` + `shardingAlgorithms` 命名引用 |
| Boot 3 | `ShardingSphereDriver` + `jdbc:shardingsphere:classpath:xxx.yaml` |
| Proxy | 补 MySQL 驱动、`authority.privilege.type: ALL_PERMITTED`、DistSQL 在线建规则 |
| 验证 | `sql-show` 看 Actual SQL 数量是否等于期望路由数 |

下一篇（4_sharding）深入分片：分片键选取、算法家族、绑定表/广播表与跨分片查询的代价分级。
