
# 1. 全局事务入口

Seata 通过 Spring AOP 拦截 `@GlobalTransactional` 注解方法：

```java
// GlobalTransactionalInterceptor
public class GlobalTransactionalInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 1. 获取注解信息
        GlobalTransactional anno = getAnnotation(invocation.getMethod());
        
        // 2. 获取或创建事务信息
        TransactionInfo txInfo = new TransactionInfo();
        txInfo.setTimeOut(anno.timeoutMills());
        txInfo.setName(anno.name());

        // 3. 执行全局事务模板
        return transactionalTemplate.execute(
            new TransactionalExecutor() {
                @Override
                public Object execute() throws Throwable {
                    return invocation.proceed();
                }

                @Override
                public TransactionInfo getTransactionInfo() {
                    return txInfo;
                }
            }
        );
    }
}
```

# 2. TransactionalTemplate

```java
public class TransactionalTemplate {

    public Object execute(TransactionalExecutor executor) throws Throwable {
        // 1. 获取全局事务
        GlobalTransaction tx = GlobalTransactionContext
            .getCurrentOrCreate();

        try {
            // 2. 开启全局事务（向 TC 发送 begin 请求）
            tx.begin(
                executor.getTransactionInfo().getTimeOut(),
                executor.getTransactionInfo().getName()
            );
            
            // 3. 执行业务方法
            Object result = executor.execute();

            // 4. 提交全局事务
            tx.commit();
            return result;

        } catch (TransactionException e) {
            // 5. 事务框架异常
            tx.rollback();
            throw e;
        } catch (Throwable e) {
            // 6. 业务异常 → 回滚
            tx.rollback();
            throw e;
        }
    }
}
```

# 3. GlobalTransaction — TM 端

```java
public class DefaultGlobalTransaction implements GlobalTransaction {

    private String xid;
    private GlobalTransactionStatus status;

    @Override
    public void begin(int timeout, String name) throws TransactionException {
        // 1. 检查是否已存在全局事务
        String currentXid = RootContext.getXID();
        if (currentXid != null) {
            throw new TransactionException("Global transaction already exists");
        }

        // 2. 向 TC 发起 begin 请求
        xid = transactionManager.begin(
            null, null, name, timeout
        );

        // 3. 绑到当前线程
        RootContext.bind(xid);
        status = GlobalTransactionStatus.Begin;
    }

    @Override
    public void commit() throws TransactionException {
        // 向 TC 发起 commit 请求
        transactionManager.commit(xid);
        status = GlobalTransactionStatus.Committed;
    }

    @Override
    public void rollback() throws TransactionException {
        // 向 TC 发起 rollback 请求
        transactionManager.rollback(xid);
        status = GlobalTransactionStatus.Rollbacked;
    }
}
```

# 4. TC Server — 事务协调

## 4.1 begin 处理

```java
public class DefaultCore implements Core {

    @Override
    public String begin(String applicationId, String txServiceGroup,
                         String name, int timeout) {
        // 1. 生成全局事务 ID
        String xid = UUIDGenerator.generateUUID();
        
        // 2. 创建 GlobalSession
        GlobalSession session = new GlobalSession(
            applicationId, txServiceGroup, name, timeout
        );
        session.setXid(xid);
        session.setStatus(GlobalStatus.Begin);
        
        // 3. 持久化到 storage
        sessionHolder.addGlobalSession(session);
        
        // 4. 启动超时定时器
        sessionManager.addGlobalSession(session);
        
        return xid;
    }
}
```

## 4.2 commit 处理

```java
@Override
public GlobalStatus commit(String xid) throws TransactionException {
    // 1. 获取全局事务会话
    GlobalSession session = sessionHolder.findGlobalSession(xid);
    if (session == null) {
        throw new TransactionException("Global session not found");
    }

    // 2. 标记为 Committing
    session.changeStatus(GlobalStatus.Committing);

    // 3. 遍历所有分支，逐个通知 commit
    for (BranchSession branch : session.getSortedBranches()) {
        branchCommit(session, branch);
    }

    // 4. 标记为 Committed
    session.changeStatus(GlobalStatus.Committed);

    // 5. 清理资源
    sessionHolder.removeGlobalSession(session);
    return GlobalStatus.Committed;
}
```

## 4.3 rollback 处理

```java
@Override
public GlobalStatus rollback(String xid) throws TransactionException {
    GlobalSession session = sessionHolder.findGlobalSession(xid);

    // 1. 标记为 Rollbacking
    session.changeStatus(GlobalStatus.Rollbacking);

    // 2. 逆序遍历分支，逐个通知 rollback
    List<BranchSession> branches = session.getSortedBranches();
    Collections.reverse(branches);
    for (BranchSession branch : branches) {
        branchRollback(session, branch);
    }

    // 3. 标记为 Rollbacked
    session.changeStatus(GlobalStatus.Rollbacked);
    sessionHolder.removeGlobalSession(session);
    return GlobalStatus.Rollbacked;
}
```

# 5. 状态机

```
                            begin()
                               │
                               ▼
                          ┌────────┐
                          │  Begin  │
                          └───┬────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                成功 ▼              异常 ▼
              ┌──────────┐       ┌──────────┐
              │Comitting │       │Rollbacking│
              └────┬─────┘       └────┬─────┘
                   │                  │
                   ▼                  ▼
              ┌──────────┐       ┌──────────┐
              │Committed │       │Rollbacked│
              └──────────┘       └──────────┘

超时处理：
  Begin/Comitting/Rollbacking 超时 → 触发定时任务 rollback/commit
```

# 6. 超时处理

```java
// TC Server 中的 SessionManager
public class SessionManager {

    @Scheduled(fixedDelay = 1000)  // 每秒检查
    public void handleTimeoutSession() {
        for (GlobalSession session : allSessions()) {
            if (session.isTimeout()) {
                switch (session.getStatus()) {
                    case Begin:
                        // 超时未提交 → 自动回滚
                        core.rollback(session.getXid());
                        break;
                    case Committing:
                    case Rollbacking:
                        // Phase 2 执行超时 → 重试
                        retryCommitOrRollback(session);
                        break;
                }
            }
        }
    }
}
```
