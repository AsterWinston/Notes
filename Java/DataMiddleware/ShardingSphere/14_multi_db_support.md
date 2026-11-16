# 1. 概述

ShardingSphere 5.x 支持多种数据库，提供方言自动适配和 SQL 翻译能力。**"支持一个数据库"在 ShardingSphere 里意味着三件事同时成立**：有该库的 SQL 方言解析器（Lexer/Parser 语法文件 + AST Visitor）、有该库的 `DatabaseType` 实现（负责保留字、分页语法、主键自增元数据等差异点）、以及在该库上跑通过的集成测试矩阵。任何一项缺失，SQL 都会落到"解析失败降级直通"的路径上——能跑，但分片改写、分页改写、聚合改写等增值能力全部失效。

```text
支持的数据库：
┌──────────┬──────────┬──────────┐
│  MySQL   │PostgreSQL│ openGauss│
│  5.7/8.0 │  9.6-16  │  2.0-5.0 │
└──────────┴──────────┴──────────┘
│  SQLServer │  Oracle  │          │
│ 2008-2022  │ 11g-19c  │          │
└────────────┴──────────┘
```

5.5.x 的支持度分级值得记住：MySQL/PostgreSQL/openGauss 是**全功能方言**（解析、路由、改写、事务、联邦查询全支持，其中 openGauss 复用 PG 语法树并有独立语法分支）；Oracle/SQL Server 是**基础支持**（可作后端数据源跑分片与读写分离，但 DistSQL、SQL Translator、联邦查询覆盖度低）；ClickHouse/Hive 等通过 optional-plugins 提供只读分析场景支持。

# 2. 方言自动适配

ShardingSphere 通过 SQL 方言自动适配不同数据库的 SQL 语法差异。方言识别的优先级链条是：**显式配置（SQL Parser 规则的 `databaseType`）→ 数据源元数据探测（`DatabaseMetaData.getDatabaseProductName()`）→ 默认回退**。JDBC 模式下驱动 URL 与元数据足以判定；Proxy 模式下前端协议类型（客户端连上来说的是哪种协议）与后端存储的方言是**两回事**——前端 MySQL 协议 + 后端 PG 的组合，就是第 5 节 SQL 翻译的主场。

```java
// 自动识别数据源类型
MySQL:
  SQL: SELECT * FROM t_order LIMIT 10 OFFSET 20
  → MySQL 方言解析

PostgreSQL:
  SQL: SELECT * FROM t_order LIMIT 10 OFFSET 20
  → PostgreSQL 方言解析
```

## 2.1 方言差异处理

| 语法 | MySQL | PostgreSQL | Oracle |
|------|-------|-----------|--------|
| 分页 | LIMIT offset, count | LIMIT count OFFSET offset | ROWNUM / FETCH |
| 自增主键 | AUTO_INCREMENT | SERIAL | SEQUENCE |
| 字符串拼接 | CONCAT() | \|\| | CONCAT() / \|\| |
| 当前时间 | NOW() | NOW() | SYSDATE |
| 引用标识符 | 反引号 ` | 双引号 " | 双引号 " |
| 大小写语义 | 表名默认不区分（Linux 区分文件名） | 未加引号折叠为小写 | 未加引号折叠为大写 |

大小写折叠是异构迁移里最阴险的差异：**同一条不带引号的 SQL，在 MySQL 里查 `T_Order` 能命中，在 PG 里折叠成 `t_order`，在 Oracle 里折叠成 `T_ORDER`**。ShardingSphere 的分片算法对逻辑表名大小写敏感地做映射（`t_order` 配成规则后 SQL 写 `T_ORDER`，MySQL 后端没问题，PG 后端直接落到"未找到分片规则"路径按单表处理），跨库项目统一表名大小写规范比任何配置都有效。

## 2.2 各方言的解析语法树组织

方言差异全部收敛在 `shardingsphere-sql-parser-dialect-*` 模块：每种方言一套 ANTLR `.g4` 语法文件 + 独立的 `XxxStatementVisitor` 体系，产出**统一的抽象 SQLStatement**（`SelectStatement`、`InsertStatement` 等），分片/加密等规则引擎只面向抽象语法树编程，不感知具体方言。也就是说方言树是"多入口、单出口"：

```text
MySQL.g4 ──▶ MySQLVisitor ──┐
PG.g4    ──▶ PGVisitor     ──┼──▶ 抽象 SQLStatement（规则引擎唯一可见的模型）
Oracle.g4──▶ OracleVisitor ──┘
```

**设计考量**：

- **新增方言成本可控**：语法文件 + Visitor + DatabaseType 三件套即可接入，不改核心引擎；官方 Dev Manual 的 SQL Parser 章节提供了 SPI 骨架
- **抽象层丢信息**：方言独有语法（MySQL `GROUP_CONCAT` 的 SEPARATOR 子句、Oracle 闪回查询）在抽象树上没有对应节点，改写引擎无法安全处理这类语句，会跳过改写直通——遇到"某函数在分片表上不生效"的反馈，先怀疑这个
- **版本跨度是隐坑**：语法文件按大版本维护（如 PostgreSQL 9.6-16 共用一套），新版本的语法（PG 15 的 `MERGE`）在旧 g4 里不存在，解析失败同样走降级路径

# 3. 数据源配置

数据源配置的通用骨架（5.5.x 驼峰属性名，Spring Boot Driver 方式与 Proxy 方式一致）见 13 篇，此处按数据库给出关键差异项。

## 3.1 MySQL

```yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_0?useSSL=false&rewriteBatchedStatements=true
    username: root
    password: root
```

`rewriteBatchedStatements=true` 是对分片影响很大的驱动参数：ShardingSphere 把一条逻辑 INSERT 扇出成多物理连接上的批量语句，没有这个参数 MySQL 驱动会把 `addBatch` 退化为逐条发送，批量插入吞吐差 5-10 倍。

## 3.2 PostgreSQL

```yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: org.postgresql.Driver
    jdbcUrl: jdbc:postgresql://localhost:5432/demo_ds_0
    username: postgres
    password: postgres
```

## 3.3 openGauss

```yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: org.opengauss.Driver
    jdbcUrl: jdbc:opengauss://localhost:5432/demo_ds_0
    username: gaussdb
    password: gaussdb
```

## 3.4 SQL Server

```yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.microsoft.sqlserver.jdbc.SQLServerDriver
    jdbcUrl: jdbc:sqlserver://localhost:1433;DatabaseName=demo_ds_0
    username: sa
    password: sa
```

## 3.5 Oracle

```yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: oracle.jdbc.driver.OracleDriver
    jdbcUrl: jdbc:oracle:thin:@localhost:1521/demo_ds_0
    username: scott
    password: tiger
```

# 4. 特性差异：自增、序列与事务隔离

## 4.1 主键生成语义的三种世界

| 数据库 | 原生机制 | 与 ShardingSphere 的协作方式 |
|--------|---------|------------------------------|
| MySQL | AUTO_INCREMENT（表级计数器） | 分片下**禁用**：各分片独立计数必冲突；用 SNOWFLAKE/COSID 改写补列 |
| PostgreSQL | SERIAL/IDENTITY（序列） | 序列是库级对象，跨分片不连续但可不冲突；生产仍推荐分布式主键 |
| Oracle | SEQUENCE + 触发器 | 序列每分片各建各的会重号；`getGeneratedKeys` 依赖列名约定，配 keyGenerateStrategy 时让 SS 先补值 |

**核心结论**：任何"依赖数据库本地自增"的方案在分片世界都不成立。ShardingSphere 的改写器在 INSERT 缺少主键列值时，把 SNOWFLAKE 生成值**改写补进列与参数**（`INSERT INTO t_order (user_id) VALUES (?)` → `INSERT INTO t_order_0 (order_id, user_id) VALUES (?, ?)`），这是"分布式主键"能力的真实机制，而非数据库端配合。

## 4.2 事务隔离级别与默认值

| 数据库 | 默认隔离级别 | XA 支持 | 备注 |
|--------|-------------|---------|------|
| MySQL(InnoDB) | REPEATABLE READ | ✓ | RR 下有间隙锁，跨片范围 UPDATE 死锁概率高 |
| PostgreSQL | READ COMMITTED | ✓ | 无间隙锁（用 SSI 替代），RC 下改写依赖 MVCC |
| Oracle | READ COMMITTED | ✓ | SERIALIZABLE 开销极大，分片场景避免 |
| SQL Server | READ COMMITTED | ✓（需 MSDTC 或 XA 驱动） | 锁粒度升级（页锁→表锁）影响跨片一致 |

隔离级别差异在**本地事务降级判定**时容易被忽略：ShardingSphere 判定 SQL 只路由到单数据源时会把 XA 降级为本地事务（见 18 篇），此时一致性语义完全由该库的隔离级别兜底——同一条业务 SQL 在 MySQL 集群上是 RR 快照读，迁到 PG 上变成 RC，不可重复读的异常类型随之改变。异构双活（同一规则两套方言后端）的项目，回归用例必须按后端库分别定义。

> **设计争议**：读写分离 + 主从异步复制叠加各库隔离级别差异后，"写后读不到"的表现和修复手段（事务内强制主库、等待位点、hint）在不同库上并不等价，PG 的物理复制延迟语义与 MySQL binlog 位点也不同构，方案文档里要分开写，不能一份模板套到底。

# 5. 异构联邦查询

ShardingSphere 支持跨数据库类型的联合查询，由 SQL Federation 引擎（基于 Apache Calcite）把跨源 JOIN、子查询翻译成 Calcite RelNode 计划下推到各源执行：

```yaml
dataSources:
  ds_mysql:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/order_db
  ds_pg:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: org.postgresql.Driver
    jdbcUrl: jdbc:postgresql://localhost:5432/user_db

rules:
- !SHARDING
  tables:
    t_order:
      actualDataNodes: ds_mysql.t_order
    t_user:
      actualDataNodes: ds_pg.t_user
- !SQL_FEDERATION
  sqlParser:
    databaseType: MySQL
```

```sql
-- 跨 MySQL 和 PostgreSQL 的关联查询
SELECT o.*, u.name
FROM t_order o
JOIN t_user u ON o.user_id = u.id;
-- ShardingSphere 在内存层做 JOIN（联邦引擎）
-- 注意：性能远低于单库 JOIN
```

## 5.1 联邦查询的边界与代价

开启 `sql-federation-enabled: true` 后，原先"不支持 SQL99、无法路由"的跨库 JOIN 变成可执行，但代价模型必须清楚：Calcite 将 JOIN 一侧的数据**全量拉到 Proxy/JDBC 内存侧**做 hash join，无索引、无统计信息收集（5.5 支持部分下推 filter），百万行侧驱动直接 OOM 是常态。官方限制清单：MySQL 8.x/PostgreSQL 支持最完整，Oracle/SQL Server 后端不支持联邦；聚合下推仅在确定单源命中时生效。**因此联邦查询的定位是管理后台、数据核对这类低频 SQL 的"保底能力"**，绝不允许进入交易链路。

> **行业实践**：跨异构库的关联需求，成熟做法是数据侧解决——ETL/CDC 把维表同步到主库本地，或者接入 ES/ClickHouse 做宽表；用联邦查询硬算跨源的，基本都是没做数据规划的临时方案。

# 6. SQL 翻译

ShardingSphere 可以在不同方言之间做 SQL 翻译（`!SQL_TRANSLATOR` 规则，内置 TYPE 转换器，基于 JSQLParser），典型场景是 **Proxy 前端 MySQL 协议 → 后端 PostgreSQL**：应用不改 SQL，Proxy 把它翻译成后端方言：

```text
场景：应用使用 MySQL 语法，后端是 PostgreSQL

MySQL SQL: SELECT * FROM t_order LIMIT 0, 10
     ↓ SQL 翻译
PG SQL:    SELECT * FROM t_order LIMIT 10 OFFSET 0
```

翻译能力边界要提前验证：反引号标识符、`IFNULL`、`DATE_FORMAT`、`ON DUPLICATE KEY UPDATE` 这类 MySQL 专有语法**不在基础翻译覆盖范围内**，Translator 是文本级 AST 转换，不是语义级重写引擎，遇到无法安全转换的结构会原样透传给后端并报错。"MySQL 语法访问 PG"可行的前提是应用 SQL 自律使用标准 SQL——这对存量 MySQL 系统往往不成立，所以 Translator 更多用于**Proxy 统一网关 + 少量新 PG 服务**，而不是 MySQL 全量平迁 PG 的救星。

# 7. 存储过程支持度

| 数据库 | 支持度 | 说明 |
|--------|--------|------|
| MySQL | 部分 | 简单存储过程可用 |
| PostgreSQL | 部分 | 函数调用可用 |
| SQL Server | 有限 | — |
| Oracle | 有限 | — |
| openGauss | 有限 | — |

**注意**：跨分片的存储过程调用的结果集合并是复杂的，建议存储过程仅操作单分片数据。

补充机制原因：存储过程/函数调用语句（`{CALL p(?)}`、`SELECT f()`）在解析器里是独立语句类型，**没有表段可提取分片值**，路由只能广播到全部分片执行——副作用类过程（有 UPDATE）会被执行 N 次，直接数据灾难。带 `?` 出参的调用在部分方言上还会与占位符体系混淆。存量系统大量存储过程的，分片改造前先做过程清单盘点，把有副作用的过程改造成"先路由查询、再单片调用"或干脆应用层重写。

# 8. 最佳实践

1. **同构优先**：各分片使用相同数据库类型，避免方言转换损耗
2. **联邦查询谨慎使用**：跨库 JOIN 性能低下，仅在管理后台等低频场景使用
3. **Proxy 模式异构更友好**：Proxy 的 MySQL 协议层屏蔽了后端差异
4. **驱动版本**：确保使用数据库官方推荐的 JDBC 驱动版本（PG 用最新 pgjdbc，Oracle 19c 起用 ojdbc11）
5. **测试覆盖**：每种数据库方言的 SQL 都应在目标数据库上验证
6. **把"降级直通"纳入监控**：方言解析失败不抛异常只少改写，生产要监控 SQL 解析失败计数，发现某类语句频繁失败就针对性简化 SQL 或升级语法文件
7. **迁移异构库先对齐大小写与保留字**：逻辑表名列名全小写、避开各库保留字，比事后加引号可靠得多

# 9. 小结

1. **"支持"= 语法文件 + DatabaseType + 测试矩阵三者齐备**，缺项即降级直通
2. **本地自增在分片世界集体出局**，主键必须交给 SNOWFLAKE 等分布式生成器
3. **隔离级别默认值随库而异**，本地事务降级后的语义由后端库兜底，异构迁移要分库定义回归用例
4. **联邦查询是保底能力不是方案**，跨源关联优先 ETL/CDC 数据侧解决
5. **存储过程/方言函数先盘点再上分片**，解析覆盖盲区静默丢能力