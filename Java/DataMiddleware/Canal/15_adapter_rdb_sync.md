
# 1. RDB Adapter 概述

RDB Adapter 是 Canal Adapter 中最常用的模块，负责将 MySQL 的增量变更数据同步到关系型数据库。支持的目标数据库包括：

| 目标数据库 | 驱动 | 状态 |
|-----------|------|------|
| MySQL | `mysql-connector-java` | 生产可用 |
| Oracle | `ojdbc` | 生产可用 |
| PostgreSQL | `postgresql` | 生产可用 |
| SQL Server | `mssql-jdbc` | 生产可用 |
| 达梦 (DM) | `dm-jdbc` | 兼容 |
| TiDB | `mysql-connector-java` | 兼容（MySQL 协议） |

# 2. 整体架构

```
┌──────────┐    ┌──────────────┐    ┌─────────────────────────┐
│  MySQL   │───>│ Canal Server │───>│     Canal Adapter        │
│ (Source) │    │              │    │                          │
└──────────┘    └──────────────┘    │  ┌───────────────────┐  │
                                    │  │   RDB Adapter     │  │
                                    │  │                   │  │
                                    │  │ ┌───────────────┐ │  │
                                    │  │ │  INSERT       │ │  │
                                    │  │ │  UPDATE       │ │  │
                                    │  │ │  DELETE       │ │  │
                                    │  │ │  DDL          │ │  │
                                    │  │ └───────────────┘ │  │
                                    │  └────────┬──────────┘  │
                                    └───────────┼─────────────┘
                                                │
                                    ┌───────────▼─────────────┐
                                    │    目标数据库            │
                                    │  (MySQL/Oracle/PG/...)   │
                                    └─────────────────────────┘
```

# 3. 配置详解

## 3.1 主配置文件 (application.yml)

```yaml
server:
  port: 8081

canal.conf:
  mode: tcp
  canalServerHost: 127.0.0.1:11111
  
  # 源数据源（用于全量 ETL 查询表结构）
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/source_db?useUnicode=true
      username: canal
      password: canal_password
  
  # 目标数据源
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: rdb
              key: mysql1       # 目标数据源 key
              properties:
                jdbc.driverClassName: com.mysql.cj.jdbc.Driver
                jdbc.url: jdbc:mysql://127.0.0.1:3306/target_db?useUnicode=true
                jdbc.username: root
                jdbc.password: root_password
                jdbc.maxWait: 60000
                jdbc.maxActive: 20
                jdbc.minIdle: 5
```

## 3.2 表映射配置 (rdb/xxx.yml)

```yaml
# conf/rdb/user_sync.yml

# 基础配置
dataSourceKey: defaultDS      # 对应 srcDataSources 的 key
outerAdapterKey: mysql1       # 对应目标数据源的 key
destination: example          # Canal Instance

# 目标表信息
targetDb: target_db
targetTable: user
targetPk: id                  # 目标表主键（用于 UPDATE/DELETE 定位行）

# 源表信息
sourceDb: source_db
sourceTable: users

# 映射模式
mapMode: ALL                  # ALL / COLUMN / EXCLUDE

# 批量提交配置
commitBatch: 3000             # 每 3000 条提交一次事务

# ETL 全量同步
# etlCondition: "where id > 0 and deleted = 0"
```

## 3.3 一对一映射

```yaml
dataSourceKey: defaultDS
outerAdapterKey: mysql1
destination: example

targetDb: target_db
targetTable: t_order

sourceDb: source_db
sourceTable: orders

# 一对一映射：源表的列名和目标表的列名完全一致
mapMode: ALL
```

## 3.4 列名映射

```yaml
targetDb: target_db
targetTable: t_order

sourceDb: source_db
sourceTable: orders

mapMode: COLUMN
columnMapping:
  order_id: id               # 源 order_id -> 目标 id
  order_status: status       # 源 order_status -> 目标 status
  create_time: created_at    # 源 create_time -> 目标 created_at
  # 未映射的列不会被同步
```

## 3.5 排除列映射

```yaml
targetDb: target_db
targetTable: t_order

sourceDb: source_db
sourceTable: orders

mapMode: EXCLUDE
excludeColumns:
  - password_hash
  - internal_notes
  - debug_flag
  # 同步除上述列之外的所有列
```

# 4. 同步策略

## 4.1 INSERT 同步

```java
// RdbAdapter 中的 INSERT 处理
@Override
protected void syncInsert(MappingConfig config, Dml dml) {
    
    if (dml.getData() == null || dml.getData().isEmpty()) {
        return;
    }
    
    String targetDb = config.getTargetDb();
    String targetTable = config.getTargetTable();
    
    // 构建 INSERT SQL
    // INSERT INTO target_db.target_table (col1, col2, ...) VALUES (?, ?, ...)
    String insertSql = buildInsertSql(config, dml.getData().get(0));
    
    // 批量插入
    List<Object[]> batchParams = new ArrayList<>();
    for (Map<String, String> row : dml.getData()) {
        Object[] params = buildInsertParams(config, row);
        batchParams.add(params);
    }
    
    // 分批执行
    for (int i = 0; i < batchParams.size(); i += config.getCommitBatch()) {
        int end = Math.min(i + config.getCommitBatch(), batchParams.size());
        List<Object[]> batch = batchParams.subList(i, end);
        
        targetJdbcTemplate.batchUpdate(insertSql, batch);
    }
    
    logger.debug("INSERT sync: {} rows to {}.{}", dml.getData().size(), targetDb, targetTable);
}

private String buildInsertSql(MappingConfig config, Map<String, String> row) {
    StringBuilder sql = new StringBuilder();
    sql.append("INSERT IGNORE INTO ");  // IGNORE 处理主键冲突
    sql.append(config.getTargetDb()).append(".").append(config.getTargetTable());
    sql.append(" (");
    
    List<String> targetColumns = resolveTargetColumns(config, row);
    sql.append(String.join(", ", targetColumns));
    sql.append(") VALUES (");
    sql.append(String.join(", ", Collections.nCopies(targetColumns.size(), "?")));
    sql.append(")");
    
    return sql.toString();
}
```

## 4.2 UPDATE 同步

```java
@Override
protected void syncUpdate(MappingConfig config, Dml dml) {
    
    if (dml.getData() == null || dml.getData().isEmpty()) {
        return;
    }
    
    String targetDb = config.getTargetDb();
    String targetTable = config.getTargetTable();
    
    // 构建 UPDATE SQL
    // UPDATE target_db.target_table SET col1=?, col2=? WHERE pk1=? AND pk2=?
    String updateSql = buildUpdateSql(config, dml.getData().get(0));
    
    List<Object[]> batchParams = new ArrayList<>();
    for (int i = 0; i < dml.getData().size(); i++) {
        Map<String, String> newRow = dml.getData().get(i);
        Map<String, String> oldRow = dml.getOld().get(i);
        
        Object[] params = buildUpdateParams(config, newRow, oldRow);
        batchParams.add(params);
    }
    
    // 分批执行
    for (int i = 0; i < batchParams.size(); i += config.getCommitBatch()) {
        int end = Math.min(i + config.getCommitBatch(), batchParams.size());
        List<Object[]> batch = batchParams.subList(i, end);
        
        targetJdbcTemplate.batchUpdate(updateSql, batch);
    }
    
    logger.debug("UPDATE sync: {} rows to {}.{}", dml.getData().size(), targetDb, targetTable);
}

private String buildUpdateSql(MappingConfig config, Map<String, String> row) {
    StringBuilder sql = new StringBuilder();
    sql.append("UPDATE ");
    sql.append(config.getTargetDb()).append(".").append(config.getTargetTable());
    sql.append(" SET ");
    
    List<String> targetColumns = resolveTargetColumns(config, row);
    List<String> targetPks = resolveTargetPks(config);
    
    // SET 子句（排除主键）
    List<String> setClauses = new ArrayList<>();
    for (String col : targetColumns) {
        if (!targetPks.contains(col)) {
            setClauses.add(col + " = ?");
        }
    }
    sql.append(String.join(", ", setClauses));
    
    // WHERE 子句
    sql.append(" WHERE ");
    List<String> whereClauses = new ArrayList<>();
    for (String pk : targetPks) {
        whereClauses.add(pk + " = ?");
    }
    sql.append(String.join(" AND ", whereClauses));
    
    return sql.toString();
}
```

## 4.3 DELETE 同步

```java
@Override
protected void syncDelete(MappingConfig config, Dml dml) {
    
    if (dml.getData() == null || dml.getData().isEmpty()) {
        return;
    }
    
    // 软删除 vs 硬删除
    if (config.isSoftDelete()) {
        syncSoftDelete(config, dml);
    } else {
        syncHardDelete(config, dml);
    }
}

private void syncHardDelete(MappingConfig config, Dml dml) {
    String deleteSql = buildDeleteSql(config);
    
    List<Object[]> batchParams = new ArrayList<>();
    for (Map<String, String> row : dml.getData()) {
        Object[] params = buildDeleteParams(config, row);
        batchParams.add(params);
    }
    
    for (int i = 0; i < batchParams.size(); i += config.getCommitBatch()) {
        int end = Math.min(i + config.getCommitBatch(), batchParams.size());
        List<Object[]> batch = batchParams.subList(i, end);
        
        targetJdbcTemplate.batchUpdate(deleteSql, batch);
    }
    
    logger.debug("DELETE sync: {} rows from {}.{}", 
        dml.getData().size(), config.getTargetDb(), config.getTargetTable());
}

private String buildDeleteSql(MappingConfig config) {
    StringBuilder sql = new StringBuilder();
    sql.append("DELETE FROM ");
    sql.append(config.getTargetDb()).append(".").append(config.getTargetTable());
    sql.append(" WHERE ");
    
    List<String> targetPks = resolveTargetPks(config);
    List<String> whereClauses = new ArrayList<>();
    for (String pk : targetPks) {
        whereClauses.add(pk + " = ?");
    }
    sql.append(String.join(" AND ", whereClauses));
    
    return sql.toString();
}
```

# 5. 主键处理

## 5.1 主键识别

```java
public class PkResolver {
    
    /**
     * 确定目标表的主键
     * 优先级：配置的 targetPk > 源表主键映射 > 自动检测
     */
    public List<String> resolveTargetPks(MappingConfig config, TableMeta sourceMeta) {
        
        // 1. 配置指定
        if (StringUtils.isNotEmpty(config.getTargetPk())) {
            return Arrays.asList(config.getTargetPk().split(","));
        }
        
        // 2. 通过源表主键映射
        List<String> sourcePks = sourceMeta.getPrimaryKeys();
        List<String> targetPks = new ArrayList<>();
        for (String sourcePk : sourcePks) {
            String targetCol = config.getColumnMapping().getOrDefault(sourcePk, sourcePk);
            targetPks.add(targetCol);
        }
        
        return targetPks;
    }
    
    /**
     * 从目标数据库查询表的主键
     */
    public List<String> detectPks(DataSource targetDs, String db, String table) {
        
        String sql = "SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS " +
                     "WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ? AND COLUMN_KEY = 'PRI' " +
                     "ORDER BY ORDINAL_POSITION";
        
        List<String> pks = targetJdbcTemplate.queryForList(sql, String.class, db, table);
        
        return pks;
    }
}
```

## 5.2 复合主键支持

```yaml
# 复合主键配置
targetPk: order_id,item_id    # 多个主键用逗号分隔
```

```java
// 复合主键的 UPDATE WHERE 子句
// WHERE order_id = ? AND item_id = ?
private Object[] buildUpdateParams(MappingConfig config, 
                                    Map<String, String> newRow,
                                    Map<String, String> oldRow) {
    
    List<String> targetPks = resolveTargetPks(config);
    List<String> targetColumns = resolveTargetColumns(config, newRow);
    
    List<Object> params = new ArrayList<>();
    
    // SET 值（排除主键）
    for (String col : targetColumns) {
        if (!targetPks.contains(col)) {
            params.add(convertValue(col, newRow));
        }
    }
    
    // WHERE 主键值
    for (String pk : targetPks) {
        params.add(convertValue(pk, newRow));  // 或 oldRow
    }
    
    return params.toArray();
}
```

# 6. 数据类型转换

```java
public class DataTypeConverter {
    
    /**
     * MySQL → 目标数据库的类型转换
     */
    public Object convertValue(MappingConfig config, String columnName,
                                String value, String mysqlType, int targetDbType) {
        
        if (value == null) {
            return null;
        }
        
        switch (targetDbType) {
            case TargetDbType.MYSQL:
                return value;  // MySQL → MySQL，直接使用
                
            case TargetDbType.ORACLE:
                return convertForOracle(columnName, value, mysqlType);
                
            case TargetDbType.POSTGRESQL:
                return convertForPostgreSQL(columnName, value, mysqlType);
                
            default:
                return value;
        }
    }
    
    private Object convertForOracle(String name, String value, String mysqlType) {
        // Oracle 特殊处理
        if ("datetime".equalsIgnoreCase(mysqlType) || "timestamp".equalsIgnoreCase(mysqlType)) {
            // 转为 Oracle DATE/TIMESTAMP
            return Timestamp.valueOf(value);
        }
        if (mysqlType.toLowerCase().startsWith("tinyint(1)")) {
            // MySQL boolean → Oracle NUMBER(1)
            return "1".equals(value) ? 1 : 0;
        }
        return value;
    }
    
    private Object convertForPostgreSQL(String name, String value, String mysqlType) {
        // PostgreSQL 特殊处理
        if ("datetime".equalsIgnoreCase(mysqlType)) {
            return Timestamp.valueOf(value);
        }
        if (mysqlType.toLowerCase().contains("unsigned")) {
            // PG 不支持 unsigned，转成 long
            return Long.parseLong(value);
        }
        return value;
    }
}
```

# 7. DDL 同步

## 7.1 DDL 处理

```java
@Override
protected void handleDdl(MappingConfig config, Dml dml) {
    
    String sql = dml.getSql();
    if (StringUtils.isEmpty(sql)) {
        return;
    }
    
    String upperSql = sql.toUpperCase().trim();
    
    if (upperSql.startsWith("ALTER TABLE")) {
        handleAlterTable(config, dml);
    } else if (upperSql.startsWith("CREATE TABLE")) {
        handleCreateTable(config, dml);
    } else if (upperSql.startsWith("DROP TABLE")) {
        handleDropTable(config, dml);
    } else if (upperSql.startsWith("RENAME TABLE")) {
        handleRenameTable(config, dml);
    } else {
        logger.info("unsupported DDL in sync mode: {}", sql);
        // 直接执行（仅限 MySQL → MySQL）
        if (config.isSyncDdl()) {
            String targetSql = translateDdl(config, sql);
            targetJdbcTemplate.execute(targetSql);
        }
    }
}

private String translateDdl(MappingConfig config, String sourceSql) {
    // DDL 翻译：将源表名替换为目标表名
    String result = sourceSql
        .replace(config.getSourceTable(), config.getTargetTable())
        .replace(config.getSourceDb(), config.getTargetDb());
    
    return result;
}
```

# 8. 全量同步 (ETL)

```yaml
# ETL 配置
etlCondition: "where id > 0 and status = 'active'"
# etlCondition 会作为 WHERE 条件附加到 SELECT * FROM source_table 后面
```

```bash
# 触发全量同步
# type: rdb / es / hbase
# key: 配置文件名（不含路径前缀）
curl http://localhost:8081/etl/rdb/mytest_user.yml -X POST

# 输出：
# {
#   "succeeded": true,
#   "resultMessage": "导入 RDB 数据：200000 条"
# }
```

## 8.1 ETL 实现

```java
@PostMapping("/etl/rdb/{task}")
public EtlResult etlRdb(@PathVariable String task, @RequestBody Map<String, String> params) {
    
    // 1. 加载配置
    MappingConfig config = configLoader.load("rdb/" + task);
    
    // 2. 从源库读取
    String sql = "SELECT * FROM " + config.getSourceDb() + "." + config.getSourceTable();
    if (StringUtils.isNotEmpty(config.getEtlCondition())) {
        sql += " " + config.getEtlCondition();
    }
    
    JdbcTemplate sourceTemplate = getSourceJdbcTemplate(config.getDataSourceKey());
    List<Map<String, Object>> rows = sourceTemplate.queryForList(sql);
    
    // 3. 转换为目标格式并写入
    int total = rows.size();
    int success = 0;
    int batchSize = config.getCommitBatch();
    JdbcTemplate targetTemplate = getTargetJdbcTemplate(config.getOuterAdapterKey());
    
    String insertSql = buildEtlInsertSql(config);
    
    for (int i = 0; i < total; i += batchSize) {
        int end = Math.min(i + batchSize, total);
        List<Object[]> batch = rows.subList(i, end).stream()
            .map(row -> buildEtlParams(config, row))
            .collect(Collectors.toList());
        
        targetTemplate.batchUpdate(insertSql, batch);
        success += batch.size();
    }
    
    return EtlResult.success(total, success);
}
```

# 9. 冲突处理策略

## 9.1 INSERT 冲突

```java
public class ConflictResolver {
    
    /**
     * 处理 INSERT 主键冲突的策略
     */
    public String buildInsertSql(MappingConfig config, Map<String, String> row) {
        switch (config.getInsertStrategy()) {
            case IGNORE:  // INSERT IGNORE（忽略冲突）
                return buildInsertIgnoreSql(config, row);
                
            case REPLACE: // REPLACE INTO（覆盖旧数据）
                return buildReplaceSql(config, row);
                
            case UPDATE:  // ON DUPLICATE KEY UPDATE（存在则更新）
                return buildInsertOnDuplicateUpdateSql(config, row);
                
            default:
                return buildInsertSql(config, row);
        }
    }
    
    private String buildInsertIgnoreSql(MappingConfig config, Map<String, String> row) {
        // INSERT IGNORE INTO target (col1, col2) VALUES (?, ?)
        return "INSERT IGNORE INTO " + config.getTargetTable() + " ...";
    }
    
    private String buildReplaceSql(MappingConfig config, Map<String, String> row) {
        // REPLACE INTO target (col1, col2) VALUES (?, ?)
        return "REPLACE INTO " + config.getTargetTable() + " ...";
    }
    
    private String buildInsertOnDuplicateUpdateSql(MappingConfig config, 
                                                     Map<String, String> row) {
        // INSERT INTO target (col1, col2) VALUES (?, ?)
        // ON DUPLICATE KEY UPDATE col1 = VALUES(col1), col2 = VALUES(col2)
        StringBuilder sql = new StringBuilder();
        sql.append(buildInsertSql(config, row));
        sql.append(" ON DUPLICATE KEY UPDATE ");
        
        List<String> updates = new ArrayList<>();
        for (String col : row.keySet()) {
            updates.add(col + " = VALUES(" + col + ")");
        }
        sql.append(String.join(", ", updates));
        
        return sql.toString();
    }
}
```

# 10. 多表同步配置

```yaml
# 一个 Adapter 可以配置多个表映射
canalAdapters:
  - instance: example
    groups:
      - groupId: g1
        outerAdapters:
          - name: rdb
            key: mysql1
            properties:
              jdbc.driverClassName: com.mysql.cj.jdbc.Driver
              jdbc.url: jdbc:mysql://127.0.0.1:3306/target_db?useUnicode=true
              jdbc.username: root
              jdbc.password: root_password
```

```yaml
# conf/rdb/user.yml
dataSourceKey: defaultDS
outerAdapterKey: mysql1
destination: example
targetDb: target_db
targetTable: t_user
sourceDb: source_db
sourceTable: users
targetPk: user_id
mapMode: COLUMN
columnMapping:
  user_id: id
  user_name: name
  age: age_num
```

```yaml
# conf/rdb/order.yml
dataSourceKey: defaultDS
outerAdapterKey: mysql1
destination: example
targetDb: target_db
targetTable: t_order
sourceDb: source_db
sourceTable: orders
targetPk: order_id
mapMode: ALL
```

# 11. 事务控制

```java
public class TransactionManager {
    
    /**
     * 在事务中执行批量同步
     * 如果任何一条失败，整个事务回滚
     */
    public void executeSync(MappingConfig config, List<Object[]> params, 
                            String sql, JdbcTemplate targetTemplate) {
        
        int commitBatch = config.getCommitBatch();
        
        for (int i = 0; i < params.size(); i += commitBatch) {
            int end = Math.min(i + commitBatch, params.size());
            List<Object[]> batch = params.subList(i, end);
            
            try {
                targetTemplate.batchUpdate(sql, batch);
            } catch (DataAccessException e) {
                logger.error("sync batch failed at offset {}", i, e);
                throw new SyncException("sync failed at offset " + i, e);
            }
        }
    }
}
```

# 12. 实战示例

## 12.1 MySQL → MySQL 实时同步

```
场景：将生产库 prod_db 的数据实时同步到分析库 analytics_db

Canal Server 订阅 prod_db binlog
        │
Canal Adapter RDB
        │
MySQL analytics_db（目标库）
```

```yaml
# conf/rdb/product_sync.yml
dataSourceKey: defaultDS
outerAdapterKey: mysql_analytics    # 指向分析库
destination: example
targetDb: analytics_db
targetTable: product_snapshot
sourceDb: prod_db
sourceTable: product
mapMode: COLUMN
columnMapping:
  id: product_id
  name: product_name
  price: unit_price
  created_at: created_time
targetPk: product_id
```

## 12.2 MySQL → Oracle 数据同步

```yaml
# application.yml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: rdb
              key: oracle_target
              properties:
                jdbc.driverClassName: oracle.jdbc.OracleDriver
                jdbc.url: jdbc:oracle:thin:@192.168.1.200:1521:ORCL
                jdbc.username: canal
                jdbc.password: canal_password

# conf/rdb/user_oracle.yml
dataSourceKey: defaultDS
outerAdapterKey: oracle_target
destination: example
targetDb: CANAL_USER               # Oracle 用户名作为 schema
targetTable: T_USER                # Oracle 表名大写
sourceDb: source_db
sourceTable: users
mapMode: COLUMN
columnMapping:
  id: ID
  name: USER_NAME
  email: EMAIL_ADDR
targetPk: ID
commitBatch: 1000
```
