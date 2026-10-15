# 1. Executor 体系概览

```
Executor 是 MyBatis 执行层的核心，负责 SQL 的实际执行。

┌─────────────────────────────────────────────────────────────────┐
│                        Executor (接口)                            │
│  核心方法：query() / update() / commit() / rollback() / close()   │
│  缓存相关：createCacheKey() / isCached() / clearLocalCache()     │
├─────────────────────────────────────────────────────────────────┤
│                          │                                       │
│              ┌───────────┴───────────┐                           │
│              │                       │                           │
│              ▼                       ▼                           │
│      BaseExecutor (抽象类)    CachingExecutor (装饰器)              │
│      实现：一级缓存              持有：另一个 Executor（委托对象）     │
│           事务管理                    实现：二级缓存                  │
│           模板方法 doXxx()                                         │
│              │                                                   │
│    ┌─────────┼─────────┐                                         │
│    │         │         │                                         │
│    ▼         ▼         ▼                                         │
│ Simple    Reuse     Batch                                         │
│ Executor  Executor  Executor                                     │
│ (默认)    (复用)    (批量)                                          │
│                                                                  │
│ 每次创建  复用已有  批量收集                                         │
│ Statement Statement 批量执行                                        │
└─────────────────────────────────────────────────────────────────┘
```

# 2. BaseExecutor — 模板方法模式的核心

```java
/**
 * BaseExecutor 是 Executor 的抽象基类
 * 源码：org.apache.ibatis.executor.BaseExecutor
 *
 * 它使用模板方法模式：
 *   - 定义执行骨架（query/update 的通用逻辑）
 *   - 子类实现 doQuery/doUpdate/doFlushStatements 等抽象方法
 *
 * 核心功能：
 *   1. 一级缓存（localCache）管理
 *   2. 事务管理（commit/rollback 时清空缓存）
 *   3. 延迟加载管理
 *   4. Statement 超时设置
 */
public abstract class BaseExecutor implements Executor {

    protected Transaction transaction;
    protected Executor wrapper;                          // 包装器（插件包装后的 Executor）

    // 一级缓存：PerpetualCache（本质是 HashMap）
    protected PerpetualCache localCache;

    // 本地输出参数缓存（存储过程）
    protected PerpetualCache localOutputParameterCache;

    protected Configuration configuration;

    // 嵌套查询的深度计数器（防止无限递归）
    protected int queryStack = 0;

    // 是否已关闭
    private boolean closed;

    // ==================== query() 核心流程 ====================

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter,
                             RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        // 1. 获取 BoundSql（在此处将 #{} 替换为 ?）
        BoundSql boundSql = ms.getBoundSql(parameter);

        // 2. 创建缓存 Key
        CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);

        // 3. 执行查询
        return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }

    @SuppressWarnings("unchecked")
    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter,
                             RowBounds rowBounds, ResultHandler resultHandler,
                             CacheKey key, BoundSql boundSql) throws SQLException {

        // 如果当前 Executor 已关闭，抛出异常
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (closed) {
            throw new ExecutorException("Executor was closed.");
        }

        // 如果 flushCache=true 且之前没执行过查询（queryStack=0），清空一级缓存
        if (queryStack == 0 && ms.isFlushCacheRequired()) {
            clearLocalCache();
        }

        List<E> list;
        try {
            queryStack++;
            // ★ 从一级缓存中查询 ★
            list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
            if (list != null) {
                // 一级缓存命中！处理存储过程的输出参数
                handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
            } else {
                // 一级缓存未命中 → 查询数据库
                list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
            }
        } finally {
            queryStack--;
        }

        // 处理延迟加载
        if (queryStack == 0) {
            for (DeferredLoad deferredLoad : deferredLoads) {
                deferredLoad.load();
            }
            deferredLoads.clear();

            // 如果 localCacheScope == STATEMENT，清空一级缓存
            if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
                clearLocalCache();
            }
        }
        return list;
    }

    /**
     * 从数据库查询
     *
     * 模板方法：先调用 doQuery()（子类实现），将结果放入一级缓存
     */
    private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter,
                                          RowBounds rowBounds, ResultHandler resultHandler,
                                          CacheKey key, BoundSql boundSql) throws SQLException {
        List<E> list;
        // 先在缓存中放入一个占位符（防止循环依赖）
        localCache.putObject(key, EXECUTION_PLACEHOLDER);
        try {
            // ★ 调用子类的 doQuery() 真正执行数据库查询 ★
            list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        } finally {
            localCache.removeObject(key);  // 移除占位符
        }
        // 将查询结果放入一级缓存
        localCache.putObject(key, list);

        // 如果是存储过程，缓存输出参数
        if (ms.getStatementType() == StatementType.CALLABLE) {
            localOutputParameterCache.putObject(key, parameter);
        }
        return list;
    }

    // ==================== update() 核心流程 ====================

    /**
     * update() 用于 INSERT / UPDATE / DELETE 操作
     *
     * 核心流程：
     *   1. 清空一级缓存（增删改操作会使缓存数据失效）
     *   2. 调用子类的 doUpdate() 执行
     */
    @Override
    public int update(MappedStatement ms, Object parameter) throws SQLException {
        ErrorContext.instance().resource(ms.getResource())
            .activity("executing an update").object(ms.getId());
        if (closed) {
            throw new ExecutorException("Executor was closed.");
        }
        // ★ 增删改必须清空一级缓存 ★
        clearLocalCache();
        // ★ 调用子类的 doUpdate() 执行数据库操作 ★
        return doUpdate(ms, parameter);
    }

    // ==================== 一级缓存管理 ====================

    @Override
    public void clearLocalCache() {
        if (!closed) {
            localCache.clear();
            localOutputParameterCache.clear();
        }
    }

    // ==================== 事务管理 ====================

    @Override
    public void commit(boolean required) throws SQLException {
        if (closed) {
            throw new ExecutorException("Cannot commit, transaction is already closed");
        }
        // ★ 提交时清空一级缓存 ★
        clearLocalCache();
        flushStatements();
        if (required) {
            transaction.commit();
        }
    }

    @Override
    public void rollback(boolean required) throws SQLException {
        if (!closed) {
            try {
                flushStatements();
            } finally {
                if (required) {
                    transaction.rollback();
                }
            }
            // ★ 回滚时也清空一级缓存 ★
            clearLocalCache();
        }
    }

    /**
     * 创建缓存 Key
     *
     * CacheKey 是 MyBatis 中的一个重要类，用于唯一标识一条缓存记录。
     * 它的 hashCode 基于：
     *   1. MappedStatement 的 ID
     *   2. RowBounds 的 offset 和 limit
     *   3. BoundSql 的 SQL 语句
     *   4. 所有参数值
     *   5. 环境 ID
     *
     * 基于这些值的组合，才能确保"相同的 SQL + 相同的参数"命中缓存
     */
    @Override
    public CacheKey createCacheKey(MappedStatement ms, Object parameterObject,
                                   RowBounds rowBounds, BoundSql boundSql) {
        if (closed) {
            throw new ExecutorException("Executor was closed.");
        }
        CacheKey cacheKey = new CacheKey();
        cacheKey.update(ms.getId());                    // sqlId
        cacheKey.update(rowBounds.getOffset());         // 分页偏移
        cacheKey.update(rowBounds.getLimit());          // 分页大小
        cacheKey.update(boundSql.getSql());             // SQL 语句
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
        for (ParameterMapping parameterMapping : parameterMappings) {
            // 跳过输出参数（存储过程中）
            if (parameterMapping.getMode() != ParameterMode.OUT) {
                Object value;
                String propertyName = parameterMapping.getProperty();
                if (boundSql.hasAdditionalParameter(propertyName)) {
                    value = boundSql.getAdditionalParameter(propertyName);
                } else if (parameterObject == null) {
                    value = null;
                } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                    value = parameterObject;
                } else {
                    MetaObject metaObject = configuration.newMetaObject(parameterObject);
                    value = metaObject.getValue(propertyName);
                }
                cacheKey.update(value);  // 每个参数的值
            }
        }
        if (configuration.getEnvironment() != null) {
            cacheKey.update(configuration.getEnvironment().getId());  // 环境ID
        }
        return cacheKey;
    }

    // ==================== 抽象方法 — 子类实现 ====================

    protected abstract int doUpdate(MappedStatement ms, Object parameter) throws SQLException;

    protected abstract List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException;

    protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter,
                                           RowBounds rowBounds, ResultHandler resultHandler,
                                           BoundSql boundSql) throws SQLException;

    protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter,
                                                   RowBounds rowBounds, BoundSql boundSql) throws SQLException;
}
```

# 3. SimpleExecutor — 默认执行器

```java
/**
 * SimpleExecutor 是 MyBatis 的默认执行器
 * 源码：org.apache.ibatis.executor.SimpleExecutor
 *
 * 特点：每次执行 SQL 都创建新的 PreparedStatement
 *
 * 适用场景：绝大多数常规操作
 */
public class SimpleExecutor extends BaseExecutor {

    public SimpleExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    /**
     * doUpdate 执行更新操作
     *
     * 流程：
     *   1. 创建 StatementHandler
     *   2. 通过 StatementHandler 创建 PreparedStatement
     *   3. 通过 ParameterHandler 设置参数
     *   4. 执行 statement.executeUpdate()
     *   5. 返回影响行数
     */
    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            // 1. 创建 StatementHandler
            StatementHandler handler = configuration.newStatementHandler(
                this, ms, parameter, RowBounds.DEFAULT, null, null);
            // 2. 准备 Statement（创建 + 设参）
            stmt = prepareStatement(handler, ms.getStatementLog());
            // 3. 执行更新并返回影响行数
            return handler.update(stmt);
        } finally {
            closeStatement(stmt);  // ★ SimpleExecutor 每次都关闭 Statement
        }
    }

    /**
     * doQuery 执行查询操作
     *
     * 流程与 doUpdate 类似，区别在于：
     *   - 执行的是 statement.executeQuery()
     *   - 需要通过 ResultSetHandler 映射结果集
     */
    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameter,
                               RowBounds rowBounds, ResultHandler resultHandler,
                               BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            // 1. 创建 StatementHandler
            StatementHandler handler = configuration.newStatementHandler(
                wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
            // 2. 准备 Statement
            stmt = prepareStatement(handler, ms.getStatementLog());
            // 3. 执行查询并映射结果
            return handler.query(stmt, resultHandler);
        } finally {
            closeStatement(stmt);  // ★ SimpleExecutor 每次都关闭 Statement
        }
    }

    /**
     * prepareStatement 准备 JDBC Statement
     *
     * 核心步骤：
     *   1. 获取 JDBC Connection
     *   2. 通过 StatementHandler 创建 Statement（通常是 PreparedStatement）
     *   3. 通过 StatementHandler 设置参数（ParameterHandler.setParameters()）
     *   4. 设置查询超时时间
     *   5. 设置 fetchSize
     */
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Statement stmt;
        // 获取 Connection
        Connection connection = getConnection(statementLog);
        // 创建 Statement（内部创建 PreparedStatement 或 CallableStatement）
        stmt = handler.prepare(connection, transaction.getTimeout());
        // 设置参数（给 ? 占位符赋值）
        handler.parameterize(stmt);
        return stmt;
    }
}
```

# 4. ReuseExecutor — 复用 Statement 的执行器

```java
/**
 * ReuseExecutor 复用 PreparedStatement
 * 源码：org.apache.ibatis.executor.ReuseExecutor
 *
 * 特点：
 *   - 缓存已创建的 PreparedStatement
 *   - 相同的 SQL 语句可以复用，减少 Statement 创建开销
 *   - 通过 Map<String, Statement> 存储
 *     key = SQL 语句，value = 对应的 PreparedStatement
 *
 * 注意：虽然 Statement 对象被复用，但每个请求仍然需要重新设置参数
 */
public class ReuseExecutor extends BaseExecutor {

    /** Statement 缓存：key=SQL, value=Statement */
    private final Map<String, Statement> statementMap = new HashMap<>();

    public ReuseExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(
            this, ms, parameter, RowBounds.DEFAULT, null, null);
        // ★ 使用 prepareStatement 复用机制
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.update(stmt);
    }

    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameter,
                               RowBounds rowBounds, ResultHandler resultHandler,
                               BoundSql boundSql) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(
            wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.query(stmt, resultHandler);
    }

    /**
     * 从缓存获取或创建 Statement
     */
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        BoundSql boundSql = handler.getBoundSql();
        String sql = boundSql.getSql();
        Statement stmt;
        if (hasStatementFor(sql)) {
            // ★ 缓存命中：复用已有的 Statement
            stmt = getStatement(sql);
            // 重新设置事务超时
            applyTransactionTimeout(stmt);
        } else {
            // ★ 缓存未命中：创建新的 Statement
            Connection connection = getConnection(statementLog);
            stmt = handler.prepare(connection, transaction.getTimeout());
            // 放入缓存
            putStatement(sql, stmt);
        }
        // 无论如何都要重新设置参数（参数值每次都可能不同）
        handler.parameterize(stmt);
        return stmt;
    }

    // 事务提交/回滚/关闭时必须清空 Statement 缓存并关闭所有 Statement
    @Override
    public void doFlushStatements(boolean isRollback) throws SQLException {
        closeAllStatements();  // 遍历缓存，逐个关闭
    }
}
```

# 5. BatchExecutor — 批量执行器

```java
/**
 * BatchExecutor 用于批量操作
 * 源码：org.apache.ibatis.executor.BatchExecutor
 *
 * 特点：
 *   - 将所有 Statement 操作先收集起来
 *   - 通过 JDBC 的 addBatch() 添加批处理
 *   - 在 doFlushStatements() 或事务提交时统一执行
 *
 * 流程：
 *   1. 每次 update() 只做 addBatch()，不实际执行
 *   2. 调用 flushStatements() 或事务提交时执行 executeBatch()
 *   3. 返回 BatchResult 列表
 *
 * 使用方式：
 *   SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
 *   CarMapper mapper = sqlSession.getMapper(CarMapper.class);
 *   for (Car car : carList) {
 *       mapper.insert(car);  // 只 addBatch，不执行
 *   }
 *   sqlSession.flushStatements();  // 统一执行
 *   sqlSession.commit();
 */
public class BatchExecutor extends BaseExecutor {

    /** 当前正在处理的 SQL 语句 */
    private String currentSql;

    /** 当前正在处理的 MappedStatement */
    private MappedStatement currentStatement;

    /** 批处理结果列表 */
    private final List<BatchResult> batchResultList = new ArrayList<>();

    public BatchExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        final Configuration configuration = ms.getConfiguration();
        final StatementHandler handler = configuration.newStatementHandler(
            this, ms, parameter, RowBounds.DEFAULT, null, null);
        final BoundSql boundSql = handler.getBoundSql();
        final String sql = boundSql.getSql();

        final Statement stmt;
        // 如果当前 SQL 和上一条相同 → 复用 Statement 和 batch
        if (sql.equals(currentSql) && ms.equals(currentStatement)) {
            int last = batchResultList.size() - 1;
            BatchResult batchResult = batchResultList.get(last);
            final List<Object> parameterObjects = batchResult.getParameterObjects();
            parameterObjects.add(parameterObject);  // 只添加参数，不执行
            stmt = batchResult.getStatement();
            handler.parameterize(stmt);  // 重新设置参数
        } else {
            // 新的 SQL → 创建新的 Statement
            Connection connection = getConnection(ms.getStatementLog());
            stmt = handler.prepare(connection, transaction.getTimeout());
            handler.parameterize(stmt);  // 设置参数
            currentSql = sql;
            currentStatement = ms;

            // 创建新的 BatchResult
            BatchResult batchResult = new BatchResult(ms, sql, parameterObject, stmt);
            batchResultList.add(batchResult);
        }

        // ★ 核心：调 JDBC 的 addBatch() ★
        handler.batch(stmt);
        return BATCH_UPDATE_RETURN_VALUE;  // 返回特殊标记，不返回实际行数
    }

    @Override
    public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        try {
            List<BatchResult> results = new ArrayList<>();
            if (isRollback) {
                return Collections.emptyList();
            }
            for (BatchResult batchResult : batchResultList) {
                Statement stmt = batchResult.getStatement();
                // ★ 执行 executeBatch()，获取批量结果 ★
                int[] updateCounts = stmt.executeBatch();
                batchResult.setUpdateCounts(updateCounts);
                results.add(batchResult);
            }
            return results;
        } finally {
            closeAllStatements();
            currentSql = null;
            currentStatement = null;
            batchResultList.clear();
        }
    }

    /**
     * BatchExecutor 不支持查询操作
     * 如果强行查询，会抛出异常
     */
    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameterObject,
                               RowBounds rowBounds, ResultHandler resultHandler,
                               BoundSql boundSql) throws SQLException {
        // ★ BatchExecutor 不支持查询！会先 flush 再查询 ★
        // 实际源码中会先 flushStatements 再创建 SimpleExecutor 执行查询
        throw new ExecutorException("BatchExecutor does not support select queries.");
    }
}
```

# 6. CachingExecutor — 二级缓存装饰器

```java
/**
 * CachingExecutor 是 Executor 的装饰器
 * 源码：org.apache.ibatis.executor.CachingExecutor
 *
 * 装饰器模式：
 *   - CachingExecutor 包装另一个 Executor（delegate）
 *   - 在 delegate 的基础上增加二级缓存功能
 *
 * 核心流程（以 query 为例）：
 *   1. 从 MappedStatement 获取二级缓存对象（如果开启了）
 *   2. 创建 CacheKey
 *   3. 查询二级缓存
 *   4. 命中 → 直接返回
 *   5. 未命中 → delegate.query() → 结果放入二级缓存
 */
public class CachingExecutor implements Executor {

    private final Executor delegate;
    private final TransactionalCacheManager tcm = new TransactionalCacheManager();

    public CachingExecutor(Executor delegate) {
        this.delegate = delegate;
        // 让 delegate 的 wrapper 指向 this（插件链）
        delegate.setExecutorWrapper(this);
    }

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameterObject,
                             RowBounds rowBounds, ResultHandler resultHandler,
                             CacheKey key, BoundSql boundSql) throws SQLException {
        // 获取二级缓存
        Cache cache = ms.getCache();
        if (cache != null) {
            // 如果需要刷新缓存，清空
            flushCacheIfRequired(ms);

            if (ms.isUseCache() && resultHandler == null) {
                // 处理存储过程的输出参数
                ensureNoOutParams(ms, parameterObject, boundSql);

                // ★ 查询二级缓存 ★
                List<E> list = (List<E>) tcm.getObject(cache, key);
                if (list == null) {
                    // 未命中 → 委托给被装饰的 Executor 查询数据库
                    list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                    // ★ 结果放入二级缓存 ★
                    tcm.putObject(cache, key, list);
                }
                return list;
            }
        }
        // 没有二级缓存 → 直接委托
        return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

    /**
     * 更新操作：清空二级缓存 → 委托 delegate 执行
     */
    @Override
    public int update(MappedStatement ms, Object parameterObject) throws SQLException {
        // ★ 先清空二级缓存 ★
        flushCacheIfRequired(ms);
        // ★ 委托给 delegate 执行更新（delegate 的 update 会清空一级缓存） ★
        return delegate.update(ms, parameterObject);
    }

    @Override
    public void commit(boolean required) throws SQLException {
        // ★ 提交事务时：刷新 TransactionalCache 到真正的二级缓存 ★
        delegate.commit(required);
        tcm.commit();
    }

    @Override
    public void rollback(boolean required) throws SQLException {
        try {
            delegate.rollback(required);
        } finally {
            if (required) {
                // ★ 回滚时：清空 TransactionalCache ★
                tcm.rollback();
            }
        }
    }

    // ... 其他方法委托给 delegate
}
```

# 7. StatementHandler 体系

```
StatementHandler 负责 JDBC Statement 的创建、参数设置、SQL 执行和结果处理。

┌──────────────────────────────────────────────────────────────┐
│                   StatementHandler (接口)                      │
│   prepare() — 创建 Statement                                 │
│   parameterize() — 设置参数                                  │
│   update() / query() — 执行 SQL                              │
│   batch() — 批量操作                                         │
├──────────────────────────────────────────────────────────────┤
│                          │                                    │
│              RoutingStatementHandler (路由)                    │
│              根据 MappedStatement.statementType 路由：         │
│                STATEMENT → SimpleStatementHandler             │
│                PREPARED  → PreparedStatementHandler (默认)     │
│                CALLABLE  → CallableStatementHandler           │
│                          │                                    │
│        ┌─────────────────┼─────────────────┐                  │
│        ▼                 ▼                 ▼                  │
│  SimpleStatementHandler PreparedStmtHandler CallableStmtHandler│
│  (Statement)           (PreparedStatement) (CallableStatement) │
└──────────────────────────────────────────────────────────────┘
```

## 7.1 RoutingStatementHandler

```java
/**
 * RoutingStatementHandler 是 StatementHandler 的路由代理
 * 源码：org.apache.ibatis.executor.statement.RoutingStatementHandler
 *
 * 根据 MappedStatement 的 statementType
 * 将请求路由到实际的 StatementHandler 实现
 */
public class RoutingStatementHandler implements StatementHandler {

    private final StatementHandler delegate;

    public RoutingStatementHandler(Executor executor, MappedStatement ms,
                                   Object parameter, RowBounds rowBounds,
                                   ResultHandler resultHandler, BoundSql boundSql) {
        switch (ms.getStatementType()) {
            case STATEMENT:
                delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case PREPARED:
                delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case CALLABLE:
                delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            default:
                throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
        }
    }

    // 所有方法都委托给 delegate
    @Override
    public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
        return delegate.prepare(connection, transactionTimeout);
    }
    // ...
}
```

## 7.2 PreparedStatementHandler — 最常用的 Handler

```java
/**
 * PreparedStatementHandler 是最常用的 StatementHandler
 * 源码：org.apache.ibatis.executor.statement.PreparedStatementHandler
 *
 * 核心方法：
 *   - prepare():  创建 PreparedStatement，设置 fetchSize、maxRows、timeout
 *   - parameterize(): 委托 ParameterHandler 设置 SQL 参数
 *   - update():   执行 executeUpdate()
 *   - query():    执行 executeQuery()，委托 ResultSetHandler 映射结果
 */
public class PreparedStatementHandler extends BaseStatementHandler {

    public PreparedStatementHandler(Executor executor, MappedStatement ms, Object parameter,
                                    RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        super(executor, ms, parameter, rowBounds, resultHandler, boundSql);
    }

    /**
     * 创建 PreparedStatement
     */
    @Override
    protected Statement instantiateStatement(Connection connection) throws SQLException {
        String sql = boundSql.getSql();
        // keyGenerator：主键生成策略
        if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
            // 自增主键：使用 RETURN_GENERATED_KEYS
            return connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
        } else if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
            return connection.prepareStatement(sql);
        } else {
            return connection.prepareStatement(sql,
                mappedStatement.getResultSetType().getValue(),
                ResultSet.CONCUR_READ_ONLY);
        }
    }

    /**
     * 设置 SQL 参数
     * 委托给 ParameterHandler
     */
    @Override
    public void parameterize(Statement statement) throws SQLException {
        parameterHandler.setParameters((PreparedStatement) statement);
    }

    /**
     * 执行 UPDATE/INSERT/DELETE
     */
    @Override
    public int update(Statement statement) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        int rows = ps.getUpdateCount();
        // 处理自增主键
        Object parameterObject = boundSql.getParameterObject();
        KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
        keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
        return rows;
    }

    /**
     * 执行 SELECT 查询
     */
    @Override
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        // ★ 委托 ResultSetHandler 进行结果映射 ★
        return resultSetHandler.handleResultSets(ps);
    }
}
```

# 8. ParameterHandler — 参数设置

```java
/**
 * DefaultParameterHandler 负责给 PreparedStatement 设置参数
 * 源码：org.apache.ibatis.executor.parameter.DefaultParameterHandler
 *
 * 核心逻辑：
 *   遍历 BoundSql 中的 parameterMappings 列表，
 *   每个 ParameterMapping 对应一个 ?
 *   从 parameterObject 中获取对应的值，通过 TypeHandler 设置到 PreparedStatement
 */
public class DefaultParameterHandler implements ParameterHandler {

    private final TypeHandlerRegistry typeHandlerRegistry;
    private final MappedStatement mappedStatement;
    private final Object parameterObject;
    private final BoundSql boundSql;
    private final Configuration configuration;

    @Override
    public void setParameters(PreparedStatement ps) {
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        if (parameterMappings != null) {
            for (int i = 0; i < parameterMappings.size(); i++) {
                ParameterMapping parameterMapping = parameterMappings.get(i);

                // 跳过输出参数（存储过程中使用）
                if (parameterMapping.getMode() != ParameterMode.OUT) {
                    Object value;
                    String propertyName = parameterMapping.getProperty();

                    // ★ 获取参数值 ★
                    // 1. 首先尝试从 additionalParameters 中获取
                    //    （用于 foreach 场景，#{item} 的值存在 additionalParameters 中）
                    if (boundSql.hasAdditionalParameter(propertyName)) {
                        value = boundSql.getAdditionalParameter(propertyName);
                    }
                    // 2. 如果参数对象本身是 null
                    else if (parameterObject == null) {
                        value = null;
                    }
                    // 3. 如果参数对象有对应的 TypeHandler，直接使用
                    else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                        value = parameterObject;
                    }
                    // 4. 否则通过 MetaObject 反射获取属性值
                    else {
                        MetaObject metaObject = configuration.newMetaObject(parameterObject);
                        value = metaObject.getValue(propertyName);
                    }

                    // ★ 获取 TypeHandler ★
                    TypeHandler typeHandler = parameterMapping.getTypeHandler();
                    JdbcType jdbcType = parameterMapping.getJdbcType();
                    if (value == null && jdbcType == null) {
                        jdbcType = configuration.getJdbcTypeForNull();
                    }

                    // ★ 设置参数 ★
                    typeHandler.setParameter(ps, i + 1, value, jdbcType);
                }
            }
        }
    }
}

// ParameterHandler 接口
public interface ParameterHandler {
    Object getParameterObject();
    void setParameters(PreparedStatement ps) throws SQLException;
}
```

# 9. ResultSetHandler — 结果集映射

```java
/**
 * DefaultResultSetHandler 负责将 JDBC ResultSet 映射为 Java 对象
 * 源码：org.apache.ibatis.executor.resultset.DefaultResultSetHandler
 *
 * 这是 MyBatis 中最复杂的组件之一，处理：
 *   1. 简单映射（列名 → 属性名）
 *   2. 嵌套映射（association、collection）
 *   3. 嵌套查询（select 属性）
 *   4. 鉴别器（discriminator）
 *   5. 自动映射（autoMapping）
 *   6. 存储过程结果集处理
 *
 * handleResultSets() 核心流程：
 *   1. 获取第一个 ResultSet
 *   2. 遍历所有 ResultMap（一个 MappedStatement 可以有多个 ResultMap）
 *   3. 对每个 ResultMap：
 *      a. 遍历 ResultSet 中的每一行
 *      b. 创建目标 Java 对象（ObjectFactory.create()）
 *      c. applyAutomaticMappings() — 自动映射（列名匹配）
 *      d. applyPropertyMappings() — 处理 <result> 标签定义的映射
 *      e. 处理嵌套对象（association、collection）
 *         - 如果是嵌套查询（有 select 属性）：
 *           延迟加载：放入代理对象中的未解析引用
 *           立即加载：执行嵌套查询
 *         - 如果是嵌套结果映射（无 select 属性）：
 *           递归处理嵌套的 ResultMap
 *   4. 返回映射好的对象列表
 */
public class DefaultResultSetHandler implements ResultSetHandler {

    // 自动映射的 3 个级别
    public enum AutoMappingBehavior {
        NONE,       // 不做自动映射
        PARTIAL,    // 只做不嵌套的自动映射（默认）
        FULL        // 全自动映射（包括嵌套）
    }

    @Override
    public List<Object> handleResultSets(Statement stmt) throws SQLException {
        final List<Object> multipleResults = new ArrayList<>();

        int resultSetCount = 0;
        ResultSetWrapper rsw = getFirstResultSet(stmt);

        List<ResultMap> resultMaps = mappedStatement.getResultMaps();
        int resultMapCount = resultMaps.size();

        while (rsw != null && resultMapCount > resultSetCount) {
            ResultMap resultMap = resultMaps.get(resultSetCount);
            // ★ 核心：处理结果集 ★
            handleResultSet(rsw, resultMap, multipleResults, null);
            rsw = getNextResultSet(stmt);
            resultSetCount++;
        }
        // ... 处理多个 ResultSet 的情况
        return collapseSingleResultList(multipleResults);
    }

    // ResultSetHandler 接口
    public interface ResultSetHandler {
        <E> List<E> handleResultSets(Statement stmt) throws SQLException;
        <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;
        void handleOutputParameters(CallableStatement cs) throws SQLException;
    }
}
```

# 10. Executor 的创建和拦截器包装

```java
/**
 * Configuration 中创建 Executor 的方法
 * 源码：org.apache.ibatis.session.Configuration
 *
 * newExecutor() 展示了 Executor 的完整创建流程：
 *   1. 根据 ExecutorType 创建基础 Executor
 *   2. 如果有二级缓存 → 用 CachingExecutor 装饰
 *   3. 应用插件拦截器链 → 生成代理对象
 */
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;

    Executor executor;
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);  // ★ 默认
    }

    // ★ 如果开启了二级缓存，用 CachingExecutor 包装 ★
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }

    // ★ 应用插件拦截器链 ★
    // 插件可以拦截 Executor 的所有方法
    executor = (Executor) interceptorChain.pluginAll(executor);

    return executor;
}
```

# 11. 三种 Executor 对比

```
┌───────────────────┬────────────────────┬──────────────────────┬──────────────────────┐
│      特性          │   SimpleExecutor   │   ReuseExecutor      │   BatchExecutor       │
├───────────────────┼────────────────────┼──────────────────────┼──────────────────────┤
│  Statement 创建    │ 每次创建新的        │ 相同 SQL 复用         │ 相同 SQL 复用          │
│  SQL 执行时机      │ 立即执行            │ 立即执行              │ commit 时批量执行       │
│  Statement 关闭    │ 每次用完即关        │ commit/close 时关     │ commit/close 时关      │
│  update 返回值     │ 实际影响行数        │ 实际影响行数          │ 特殊标记值              │
│  select 支持       │ 支持               │ 支持                 │ 不支持（会 flush 后查询）│
│  内存占用          │ 低                 │ 中等（缓存 Statement）│ 低（只批量）            │
│  适用场景          │ 常规操作（默认）     │ 频繁相同 SQL 的场景   │ 批量 INSERT/UPDATE     │
│  JDBC 实现         │ Statement 不复用    │ Statement 缓存复用    │ Statement.addBatch()   │
└───────────────────┴────────────────────┴──────────────────────┴──────────────────────┘

ExecutorType 选择示例：

  // 默认（SimpleExecutor）
  SqlSession sqlSession = sqlSessionFactory.openSession();

  // 复用 Statement
  SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.REUSE);

  // 批量操作
  SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
```

# 12. 一级缓存深入分析

```
一级缓存（localCache）是 SqlSession 级别的缓存

数据结构：
  PerpetualCache → 内部是 HashMap<CacheKey, Object>

缓存的生命周期：
  ┌──────────────┐
  │  创建 SqlSession  │ → localCache 创建
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  执行查询       │ → 查询 localCache，未命中则查库，结果放入 localCache
  └──────┬───────┘
         │
         ▼
  ┌───────────┐      ┌─────────────┐      ┌─────────────┐
  │ 提交/回滚   │  or  │ 执行增删改    │  or  │ clearCache()│
  └─────┬─────┘      └──────┬──────┘      └──────┬──────┘
         │                  │                     │
         └──────────────────┼─────────────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │  清空 localCache │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ 关闭 SqlSession │ → localCache 销毁
                    └──────────────┘

一级缓存失效的场景总结：
  1. 不同的 SqlSession
  2. 同一个 SqlSession 但查询条件（参数）不同
     → CacheKey 不同，无法命中
  3. 同一个 SqlSession 但两次查询期间执行了增删改操作
     → BaseExecutor.update() 中 clearLocalCache()
  4. 同一个 SqlSession 但手动清空了缓存
     → sqlSession.clearCache()
  5. 同一个 SqlSession 但 localCacheScope = STATEMENT
     → 每次查询后自动清空
  6. <select> 标签中设置了 flushCache="true"
     → 每次查询前清空缓存
```
