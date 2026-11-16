
# 1. TCC 资源注册

```java
// TCCResourceManager — RM 端注册 TCC 资源
public class TCCResourceManager extends AbstractResourceManager {

    @Override
    public void registerResource(Resource resource) {
        // resource = TCCResource，包含 Try/Confirm/Cancel 方法
        tccResources.put(resource.getResourceId(), (TCCResource) resource);
        super.registerResource(resource);
    }
}
```

# 2. TCCResource

```java
public class TCCResource implements Resource {

    private String resourceGroupId;
    private String applicationName;
    
    // 从 @TwoPhaseBusinessAction 注解解析
    private String actionName;           // Try 方法名
    private Method commitMethod;         // Confirm 方法
    private Method rollbackMethod;       // Cancel 方法

    // 分支事务 ID
    private String branchId;

    @Override
    public BranchStatus branchCommit(BranchType branchType, String xid,
                                      long branchId, String resourceId,
                                      String applicationData) {
        // TC 通知执行 Confirm
        BusinessActionContext context = parseContext(applicationData);
        return invokeConfirm(context);
    }

    @Override
    public BranchStatus branchRollback(BranchType branchType, String xid,
                                        long branchId, String resourceId,
                                        String applicationData) {
        // TC 通知执行 Cancel
        BusinessActionContext context = parseContext(applicationData);
        return invokeRollback(context);
    }

    private BranchStatus invokeConfirm(BusinessActionContext context) {
        try {
            Object result = commitMethod.invoke(bean, context);
            return BranchStatus.PhaseTwo_Committed;
        } catch (Exception e) {
            // 重试或失败
            return BranchStatus.PhaseTwo_CommitFailed_Retryable;
        }
    }
}
```

# 3. TCC 分支注册

```java
// RM 端注册 TCC 分支（Try 阶段成功后）
public Long branchRegister(BranchType branchType, String resourceId,
        String clientId, String xid, String applicationData,
        String lockKeys) throws TransactionException {

    // 1. 序列化 BusinessActionContext
    BusinessActionContext context = new BusinessActionContext();
    context.setXid(xid);
    context.setActionName(actionName);
    context.setActionContext(actionContextMap);  // Try 参数

    String contextJson = JSON.toJSONString(context);

    // 2. 向 TC 注册分支
    Long branchId = DefaultResourceManager.get()
        .branchRegister(branchType, resourceId, clientId,
            xid, contextJson, null);

    return branchId;
}
```

# 4. TC 调度 TCC 二阶段

```java
// TC Server: Core 的 commit/rollback 方法
@Override
public GlobalStatus commit(String xid) throws TransactionException {
    GlobalSession session = sessionHolder.findGlobalSession(xid);

    for (BranchSession branch : session.getSortedBranches()) {
        if (branch.getBranchType() == BranchType.TCC) {
            // TCC 分支按顺序确认
            branchCommit(session, branch);
        } else {
            // AT 分支异步提交
            asyncBranchCommit(session, branch);
        }
    }
    return GlobalStatus.Committed;
}

@Override
public GlobalStatus rollback(String xid) throws TransactionException {
    GlobalSession session = sessionHolder.findGlobalSession(xid);

    // TCC 逆序取消
    List<BranchSession> branches = session.getSortedBranches();
    Collections.reverse(branches);

    for (BranchSession branch : branches) {
        if (branch.getBranchType() == BranchType.TCC) {
            branchRollback(session, branch);
        }
    }
    return GlobalStatus.Rollbacked;
}
```

# 5. TCC 幂等与悬挂控制源码

```java
@Component
public class TccActionInterceptor {

    // 拦截 TCC Try/Confirm/Cancel 方法
    public Object intercept(MethodInvocation invocation) throws Throwable {
        // 1. 获取 BusinessActionContext
        BusinessActionContext context = getBusinessActionContext(invocation);

        // 2. 生成幂等 key
        String idempotentKey = context.getXid() + ":" + context.getBranchId();

        // 3. Try 阶段的悬挂检查
        if (isTryMethod(invocation)) {
            // 检查是否已有 Cancel 记录（悬挂控制）
            if (hasCancel(idempotentKey)) {
                log.warn("Cancel already executed, skip Try");
                return false;
            }
        }

        // 4. 幂等检查
        if (isIdempotentExecuted(idempotentKey,
                getActionType(invocation))) {
            log.info("Action already executed, skip");
            return true;
        }

        // 5. 执行
        Object result = invocation.proceed();

        // 6. 记录幂等日志
        recordIdempotent(idempotentKey, getActionType(invocation));

        return result;
    }
}
```

# 6. BusinessActionContext

```java
public class BusinessActionContext implements Serializable {

    private String xid;                         // 全局事务 ID
    private String branchId;                    // 分支 ID
    private String actionName;                  // 动作名称
    private Map<String, Object> actionContext;  // Try 阶段的参数

    // 从 actionContext 中获取 Try 参数
    public <T> T getActionContext(String key, Class<T> clazz) {
        return clazz.cast(actionContext.get(key));
    }

    // 获取 Try 阶段的所有参数名
    public Map<String, Object> getActionContext() {
        return actionContext;
    }
}
```

# 7. TCC 异常处理

```java
// TCCResource 中的异常处理
private BranchStatus invokeConfirm(BusinessActionContext context) {
    try {
        Object result = commitMethod.invoke(bean, context);
        return BranchStatus.PhaseTwo_Committed;
    } catch (InvocationTargetException e) {
        Throwable cause = e.getCause();
        if (cause instanceof SkipCallbackException) {
            // 业务主动跳过 Confirm → 直接标记完成
            return BranchStatus.PhaseTwo_Committed;
        }
        // 其他异常 → 重试
        log.error("Confirm failed, will retry", cause);
        return BranchStatus.PhaseTwo_CommitFailed_Retryable;
    }
}
```

# 8. TCC 超时处理

```java
// Seata TCC 的二阶段超时
// 如果 Confirm/Cancel 重试次数超过阈值 → TC 标记为失败并记录
// 需要人工介入处理

seata:
  client:
    rm:
      tcc:
        retry-count: 5           # Confirm/Cancel 重试次数
        recover-interval: 10000  # 重试间隔(ms)
```
