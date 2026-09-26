
# 1. 数据源代理架构

```
DataSource (原始)
  └── DataSourceProxy (Seata 代理)
        ├── ConnectionProxy
        │     └── PreparedStatementProxy
        │           └── ExecuteTemplate
        │                 ├── SELECT → SelectRecognizer → 生成前置/后置镜像
        │                 ├── UPDATE → UpdateRecognizer → 生成前置/后置镜像
        │                 ├── INSERT → InsertRecognizer → 仅后置镜像
        │                 └── DELETE → DeleteRecognizer → 仅前置镜像
        └── StatementProxy
```

# 2. DataSourceProxy

```java
public class DataSourceProxy extends AbstractDataSourceProxy {

    @Override
    public ConnectionProxy getConnection() throws SQLException {
        // 返回代理连接
        return new ConnectionProxy(this, targetDataSource.getConnection());
    }

    @Override
    public ConnectionProxy getConnection(String username, String password)
            throws SQLException {
        return new ConnectionProxy(this,
            targetDataSource.getConnection(username, password));
    }
}
```

# 3. ConnectionProxy

```java
public class ConnectionProxy extends AbstractConnectionProxy {

    // 当前连接的分支事务上下文
    private ConnectionContext context = new ConnectionContext();

    @Override
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        // 构建 SQL 识别器
        String dbType = getDbType();
        
        // 返回 PreparedStatementProxy
        return new PreparedStatementProxy(
            this,
            targetConnection.prepareStatement(sql),
            sql,
            context
        );
    }

    // 提交本地事务的代理
    @Override
    public void commit() throws SQLException {
        // 如果有 Seata 全局事务上下文 → 先注册分支事务
        if (context.inGlobalTransaction()) {
            processGlobalTransactionCommit();
        } else {
            targetConnection.commit();
        }
    }

    private void processGlobalTransactionCommit() throws SQLException {
        // 1. 注册分支事务到 TC
        register();
        
        // 2. 提交本地事务（释放 DB 锁）
        targetConnection.commit();
        
        // 3. 清空上下文（undo_log 已持久化在本地事务中）
        context.reset();
    }
}
```

# 4. PreparedStatementProxy

```java
public class PreparedStatementProxy extends AbstractPreparedStatementProxy {

    @Override
    public int executeUpdate() throws SQLException {
        // 执行前置：生成 before image
        TableRecords beforeImage = buildBeforeImage();
        
        // 执行业务 SQL
        int result = targetPreparedStatement.executeUpdate();
        
        // 执行后置：生成 after image
        TableRecords afterImage = buildAfterImage(beforeImage);
        
        // 构建 undo_log 项
        SQLUndoLog undoLog = buildUndoItem(beforeImage, afterImage);
        
        // 添加到 ConnectionContext.undoItems
        connectionContext.appendUndoItem(undoLog);
        
        return result;
    }
}
```

# 5. 前置/后置镜像构建

## 5.1 SELECT 语句解析

```java
// SelectRecognizer 解析 SELECT FOR UPDATE 等
public class SelectForUpdateRecognizer extends SelectRecognizer {

    @Override
    public TableRecords buildBeforeImage(ConnectionProxy connection,
                                          String dbType) {
        // 将 SELECT 语句转换为 SELECT ... FOR UPDATE
        // 查询当前数据 → before image
        String selectSQL = "SELECT * FROM account WHERE user_id = ? FOR UPDATE";
        PreparedStatement ps = connection.prepareStatement(selectSQL);
        ps.setLong(1, 1);
        ResultSet rs = ps.executeQuery();
        
        TableRecords records = TableRecords.buildRecords(
            tableMeta, rs);
        return records;
    }
}
```

## 5.2 UndoLog 结构

```java
public class SQLUndoLog {

    private SQLType sqlType;                // INSERT / UPDATE / DELETE
    private String tableName;               // 表名
    private TableRecords beforeImage;       // 前置镜像
    private TableRecords afterImage;        // 后置镜像
}

// 持久化到 undo_log 表
INSERT INTO undo_log (
    branch_id, xid, context,
    rollback_info, log_status, log_created, log_modified
) VALUES (
    ?, ?, ?,
    ?, -- rollback_info = JSON序列化后的 SQLUndoLog
    0, NOW(), NOW()
);
```

# 6. 分支注册

```java
// ConnectionProxy 提交时注册分支
private Long register() throws TransactionException {
    // 1. 获取全局事务 ID
    String xid = RootContext.getXID();
    if (xid == null) return null;

    // 2. 生成分支事务 ID
    Long branchId = DefaultResourceManager.get()
        .branchRegister(
            BranchType.AT,          // AT 模式
            getResourceId(),        // JDBC 资源 ID
            null,                   // 客户端 ID（自动生成）
            xid,                    // 全局事务 ID
            null,                   // 应用数据
            getLockKey()            // 全局锁键
        );

    // 3. 保存分支 ID 到上下文
    context.setBranchId(branchId);
    return branchId;
}
```

# 7. 全局锁键生成

```java
// 从 undo_log 中提取锁键
private String getLockKey() {
    // 格式: tableName;pkName:pkValue;pkName:pkValue
    // 示例: account;1;order;100
    StringBuilder lockKeys = new StringBuilder();
    for (SQLUndoLog undoLog : context.getUndoItems()) {
        TableRecords afterImage = undoLog.getAfterImage();
        lockKeys.append(afterImage.getTableMeta().getTableName());
        for (Row row : afterImage.getRows()) {
            for (Field pk : row.primaryKeys()) {
                lockKeys.append(pk.getName())
                        .append(":")
                        .append(pk.getValue())
                        .append(";");
            }
        }
    }
    return lockKeys.toString();
}
```

# 8. 回滚执行

```java
// TC 驱动 RM 执行回滚
public class RMHandlerAT {

    public BranchStatus branchRollback(String xid, long branchId,
                                        String resourceId, String applicationData) {
        // 1. 从 undo_log 读取
        SQLUndoLog undoLog = selectUndoLog(xid, branchId);
        
        // 2. 根据 SQL 类型生成反向 SQL
        String undoSQL = buildUndoSQL(undoLog);
        
        // 3. 执行反向 SQL
        // UPDATE account SET balance = 100
        // WHERE user_id = 1 AND balance = 50;  ← 校验 after image
        executeUndoSQL(undoSQL);
        
        // 4. 删除 undo_log
        deleteUndoLog(xid, branchId);
        
        return BranchStatus.PhaseTwo_Rollbacked;
    }
}
```

# 9. undo_log 清理策略

```java
// TC Server 定时清理过期 undo_log
@Scheduled(fixedDelay = 86400000)  // 每天
public void cleanUndoLog() {
    // 删除 log_status = 1（已提交）且 log_modified > 7 天的记录
    String sql = "DELETE FROM undo_log " +
        "WHERE log_status = 1 " +
        "AND log_modified < DATE_SUB(NOW(), INTERVAL ? DAY)";
    
    jdbcTemplate.update(sql, undoLogSaveDays);
}
```
