
# 1. 事务管理器架构

```java
// ShardingSphere 事务管理器 SPI
public interface ShardingSphereTransactionManager extends AutoCloseable {

    // 初始化
    void init(DatabaseType databaseType,
              Collection<ResourceTransactionManager> managers);

    // 开启事务
    void begin();

    // 提交
    void commit() throws TransactionException;

    // 回滚
    void rollback() throws TransactionException;
}
```

# 2. XA 事务管理器

```java
public class XAShardingSphereTransactionManager
        implements ShardingSphereTransactionManager {

    private final XATransactionManager xaManager;  // Atomikos / Narayana

    @Override
    public void begin() {
        // 1. 获取所有参与的数据源
        Collection<String> xaDataSources = getParticipatedDataSources();

        // 2. 为每个数据源注册 XAResource
        for (String dsName : xaDataSources) {
            XADataSource xaDataSource = getXADataSource(dsName);
            XAResource xaResource = xaDataSource.getXAConnection()
                .getXAResource();

            // 3. 登记到 JTA 事务管理器
            xaManager.getTransaction().enlistResource(xaResource);
        }
    }

    @Override
    public void commit() throws TransactionException {
        try {
            xaManager.getTransaction().commit();
        } catch (Exception e) {
            throw new TransactionException(e);
        }
    }
}
```

# 3. XA 执行流程追踪

```java
// ShardingSphere 内部的事务判断逻辑
public class TransactionContext {

    private TransactionType transactionType;

    public void determineTransactionType(
            RouteContext routeContext) {

        // 1. 如果 SQL 只涉及一个数据源 → 本地事务
        if (routeContext.getRouteUnits().size() == 1) {
            transactionType = TransactionType.LOCAL;
            return;
        }

        // 2. 如果涉及多个数据源 → XA 事务
        if (routeContext.getRouteUnits().size() > 1) {
            transactionType = TransactionType.XA;
            return;
        }
    }
}
```

# 4. Seata AT 集成

```java
public class SeataATShardingSphereTransactionManager
        implements ShardingSphereTransactionManager {

    @Override
    public void begin() {
        // Seata 的 begin 由 GlobalTransactionScanner 处理
        // ShardingSphere 只需要确保数据源被 Seata 代理
        // DataSourceProxy 自动接管
    }

    @Override
    public void commit() throws TransactionException {
        // Seata TM 驱动提交，ShardingSphere 不干预
    }

    @Override
    public void rollback() throws TransactionException {
        // Seata TM 驱动回滚
    }
}
```

Seata 集成核心：ShardingSphere 的数据源被 Seata `DataSourceProxy` 包装，所有 SQL 自动记录 undo_log。

# 5. 事务拦截流程

```java
// ShardingSphere-JDBC 事务拦截
public class ShardingSphereConnection extends AbstractConnectionAdapter {

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {
        if (!autoCommit) {
            // 开启事务
            transactionManager.begin();
        }
        super.setAutoCommit(autoCommit);
    }

    @Override
    public void commit() throws SQLException {
        try {
            // 1. 先提交各个物理连接
            super.commit();
            // 2. 再调事务管理器提交（XA COMMIT）
            transactionManager.commit();
        } catch (Exception e) {
            transactionManager.rollback();
            throw e;
        }
    }
}
```

# 6. XA 恢复（Crash Recovery）

```java
public class XARecovery {

    public void recover() {
        // 1. 查询所有 XADataSource 中的 prepared 事务
        for (XADataSource ds : xaDataSources) {
            XAResource xaResource = ds.getXAConnection()
                .getXAResource();
            Xid[] preparedXids = xaResource.recover(
                XAResource.TMSTARTRSCAN | XAResource.TMENDRSCAN);

            // 2. 对每个 prepared 事务决定 COMMIT 或 ROLLBACK
            for (Xid xid : preparedXids) {
                if (transactionLog.exists(xid)) {
                    // 有提交日志 → COMMIT
                    xaResource.commit(xid, false);
                } else {
                    // 无提交日志 → ROLLBACK
                    xaResource.rollback(xid);
                }
            }
        }
    }
}
```

# 7. 性能对比

```
测试：2 个数据源，各 10 万行，批量 INSERT 100 条

| 模式       | TPS   | AVG RT |
|-----------|--------|--------|
| 本地事务   | 5000   | 20ms   |
| XA (Atomikos) | 2500 | 40ms |
| Seata AT   | 3800 | 26ms |
```
