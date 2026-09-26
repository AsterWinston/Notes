
# 1. 概述

ShardingSphere 5.x 支持多种数据库，提供方言自动适配和 SQL 翻译能力。

```
支持的数据库：
┌──────────┬──────────┬──────────┐
│  MySQL   │PostgreSQL│ openGauss│
│  5.7/8.0 │  9.6-16  │  2.0-5.0 │
└──────────┴──────────┴──────────┘
│  SQLServer │  Oracle  │          │
│ 2008-2022  │ 11g-19c  │          │
└────────────┴──────────┘
```

# 2. 方言自动适配

ShardingSphere 通过 SQL 方言自动适配不同数据库的 SQL 语法差异：

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

# 3. 数据源配置

## 3.1 MySQL

```yaml
dataSources:
  ds0:
    driver-class-name: com.mysql.cj.jdbc.Driver
    jdbc-url: jdbc:mysql://localhost:3306/demo_ds_0
    username: root
    password: root
```

## 3.2 PostgreSQL

```yaml
dataSources:
  ds0:
    driver-class-name: org.postgresql.Driver
    jdbc-url: jdbc:postgresql://localhost:5432/demo_ds_0
    username: postgres
    password: postgres
```

## 3.3 openGauss

```yaml
dataSources:
  ds0:
    driver-class-name: org.opengauss.Driver
    jdbc-url: jdbc:opengauss://localhost:5432/demo_ds_0
    username: gaussdb
    password: gaussdb
```

## 3.4 SQL Server

```yaml
dataSources:
  ds0:
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    jdbc-url: jdbc:sqlserver://localhost:1433;DatabaseName=demo_ds_0
    username: sa
    password: sa
```

## 3.5 Oracle

```yaml
dataSources:
  ds0:
    driver-class-name: oracle.jdbc.driver.OracleDriver
    jdbc-url: jdbc:oracle:thin:@localhost:1521/demo_ds_0
    username: scott
    password: tiger
```

# 4. 异构联邦查询

ShardingSphere 支持跨数据库类型的联合查询：

```yaml
dataSources:
  ds_mysql:
    driver-class-name: com.mysql.cj.jdbc.Driver
    jdbc-url: jdbc:mysql://localhost:3306/order_db
  ds_pg:
    driver-class-name: org.postgresql.Driver
    jdbc-url: jdbc:postgresql://localhost:5432/user_db

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_mysql.t_order
      t_user:
        actualDataNodes: ds_pg.t_user
```

```sql
-- 跨 MySQL 和 PostgreSQL 的关联查询
SELECT o.*, u.name
FROM t_order o
JOIN t_user u ON o.user_id = u.id;
-- ShardingSphere 在 Proxy 层做面向内存的 JOIN
-- 注意：性能远低于单库 JOIN
```

# 5. SQL 翻译

ShardingSphere 可以在不同方言之间做 SQL 翻译：

```
场景：应用使用 MySQL 语法，后端是 PostgreSQL

MySQL SQL: SELECT * FROM t_order LIMIT 0, 10
     ↓ SQL 翻译
PG SQL:    SELECT * FROM t_order LIMIT 10 OFFSET 0
```

# 6. 存储过程支持度

| 数据库 | 支持度 | 说明 |
|--------|--------|------|
| MySQL | 部分 | 简单存储过程可用 |
| PostgreSQL | 部分 | 函数调用可用 |
| SQL Server | 有限 | — |
| Oracle | 有限 | — |
| openGauss | 有限 | — |

**注意**：跨分片的存储过程调用的结果集合并是复杂的，建议存储过程仅操作单分片数据。

# 7. 最佳实践

1. **同构优先**：各分片使用相同数据库类型，避免方言转换损耗
2. **联邦查询谨慎使用**：跨库 JOIN 性能低下，仅在管理后台等低频场景使用
3. **Proxy 模式异构更友好**：Proxy 的 MySQL 协议层屏蔽了后端差异
4. **驱动版本**：确保使用数据库官方推荐的 JDBC 驱动版本
5. **测试覆盖**：每种数据库方言的 SQL 都应在目标数据库上验证
