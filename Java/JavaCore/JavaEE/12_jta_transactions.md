
# 1. 概述

## 1.1 什么是 JTA

JTA（Java Transaction API，Java 事务 API）是 JavaEE 的**分布式事务规范**（原 JSR-907），定义事务管理器、应用接口与资源间的标准协议。核心目标：让**多个资源**（多个数据库、数据库 + 消息队列）参与同一个原子事务。

## 1.2 事务四大特性（ACID）

| 特性 | 说明 |
|------|------|
| 原子性（Atomicity） | 全部成功或全部回滚 |
| 一致性（Consistency） | 事务前后数据一致 |
| 隔离性（Isolation） | 事务之间互不干扰 |
| 持久性（Durability） | 提交后数据持久保存 |

## 1.3 事务类型对比

```
JDBC 本地事务：单个数据源，Connection.commit()/rollback()
JTA 全局事务：多个资源，由事务管理器统一协调（XA 两阶段提交）
```

# 2. JTA 核心 API

## 2.1 事务管理器接口

| 接口 | 说明 |
|------|------|
| `jakarta.transaction.TransactionManager` | 事务管理器（面向系统组件） |
| `jakarta.transaction.UserTransaction` | 面向应用的手动事务控制入口 |
| `jakarta.transaction.Transaction` | 单个事务对象 |
| `jakarta.transaction.Status` | 事务状态常量 |
| `jakarta.transaction.TransactionSynchronizationRegistry` | 注册同步回调 |

## 2.2 UserTransaction 使用

```java
@Resource
private UserTransaction utx;

public void doWork() {
    utx.begin();
    try {
        // 多个资源操作
        ds1.getConnection();   // 数据库 A
        ds2.getConnection();   // 数据库 B
        jmsProducer.send(...); // 消息队列
        utx.commit();
    } catch (Exception e) {
        utx.setRollbackOnly();
        // 或 utx.rollback();
        throw e;
    }
}
```

## 2.3 事务状态

```
ACTIVE → MARKED_ROLLBACK → ...
   ↓
  COMMITTING → COMMITTED
   ↓
  ROLLING_BACK → ROLLEDBACK
```

# 3. 声明式事务（CMT）

## 3.1 EJB 容器管理事务

```java
@Stateless
public class TransferService {

    @TransactionAttribute(TransactionAttributeType.REQUIRED)
    public void transfer(Long from, Long to, double amount) {
        // 方法正常返回 → 容器自动提交
        // 抛出运行时异常 → 容器自动回滚
    }
}
```

## 3.2 事务属性类型

| 属性 | 说明 |
|------|------|
| `REQUIRED` | **默认**。有事务则加入，无则新建 |
| `REQUIRES_NEW` | 总是新建事务，挂起外层事务 |
| `SUPPORTS` | 有事务则参与，无事务则以无事务方式运行 |
| `NOT_SUPPORTED` | 挂起当前事务，以无事务方式运行 |
| `MANDATORY` | 必须存在事务，否则抛异常 |
| `NEVER` | 必须无事务，否则抛异常 |

## 3.3 事务边界

```
调用链：A（REQUIRED）→ B（REQUIRED）→ C（REQUIRES_NEW）
                  └──────── 同一事务 ────────┘
                                               └── 独立新事务（B 的回滚不影响 C）
```

- 事务边界默认在**业务方法**处。
- `REQUIRES_NEW` 用于：日志记录、发送消息、需要独立提交的内部操作。

# 4. 回滚控制

## 4.1 自动回滚规则

| 异常类型 | 默认行为 |
|----------|----------|
| 运行时异常（RuntimeException 子类） | **自动回滚** |
| 受检异常（如 Exception） | **自动提交**（不回滚） |
| 错误（Error） | 自动回滚 |

## 4.2 显式标记回滚

```java
@Resource
private SessionContext ctx;

public void doWork() {
    try {
        // ...
    } catch (BusinessException e) {
        ctx.setRollbackOnly();   // 标记当前事务回滚
        throw e;
    }
}
```

## 4.3 指定回滚规则（ejb-jar.xml）

```xml
<assembly-descriptor>
    <container-transaction>
        <method>
            <ejb-name>TransferService</ejb-name>
            <method-name>transfer</method-name>
        </method>
        <trans-attribute>Required</trans-attribute>
    </container-transaction>
</assembly-descriptor>
```

# 5. Bean 管理事务（BMT）

```java
@Stateless
@TransactionManagement(TransactionManagementType.BEAN)
public class ManualService {

    @Resource
    private UserTransaction utx;

    public void doWork() throws Exception {
        utx.begin();
        try {
            // 业务操作
            utx.commit();
        } catch (Exception e) {
            utx.rollback();
            throw e;
        }
    }
}
```

# 6. XA 与两阶段提交（2PC）

## 6.1 XA 是什么

XA 是 X/Open 组织定义的**分布式事务处理规范**，JavaEE 通过 `javax.transaction.xa` 实现：

```
┌────────────┐         ┌────────────────┐
│ 事务管理器  │ ──────► │ 资源管理器 A(XA)│  ──► 数据库A
│ (TM)       │         ├────────────────┤
│            │ ──────► │ 资源管理器 B(XA)│  ──► 数据库B
└────────────┘         └────────────────┘
```

## 6.2 XA 资源接口

| 接口 | 说明 |
|------|------|
| `XAResource` | 参与全局事务的资源（数据库驱动实现） |
| `XAConnection` | XA 连接（`XADataSource.getXAConnection()`） |
| `Xid` | 全局事务标识 |

## 6.3 两阶段提交（2PC）

```
准备阶段（Prepare）：
  TM 通知所有资源 prepare()，各资源锁定资源并回报就绪

提交阶段（Commit）：
  全部就绪 → TM 通知各资源 commit()
  任一失败 → TM 通知各资源 rollback()
```

| 阶段 | 动作 | 失败处理 |
|------|------|----------|
| 1. 准备 | 每个 RM 预提交并记录 | 任一 RM 不能提交则回滚全部 |
| 2. 提交 | 所有 RM 正式提交 | 提交阶段宕机需要协调者恢复（3PC 改进） |

## 6.4 2PC 的缺陷

- **阻塞**：prepare 后 RM 持有锁，任一节点故障会阻塞。
- **协调者单点**：协调者故障导致事务悬挂。
- 现代分布式事务更倾向：TCC、本地消息表、Saga、Seata（见 `Interview/21_seata.md`、`Interview/50_distributed_theory.md`）。

# 7. 事务传播与隔离

## 7.1 EJB 无显式隔离级别控制

- EJB 规范**不提供事务隔离级别注解**（容器通常用默认隔离级别，如 `READ_COMMITTED`）。
- 隔离级别通常由数据库/连接池层配置（如 `hibernate.connection.isolation`）。

## 7.2 隔离级别（数据库标准）

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|----------|------|-----------|------|
| READ_UNCOMMITTED | 可能 | 可能 | 可能 |
| READ_COMMITTED | 避免 | 可能 | 可能 |
| REPEATABLE_READ | 避免 | 避免 | 可能 |
| SERIALIZABLE | 避免 | 避免 | 避免 |

# 8. 事务同步回调

在事务提交/回滚后执行逻辑（如清理缓存、发送通知）：

```java
@Resource
private TransactionSynchronizationRegistry tsr;

public void doWork() {
    tsr.registerInterposedSynchronization(new Synchronization() {
        @Override
        public void beforeCompletion() { }

        @Override
        public void afterCompletion(int status) {
            if (status == Status.STATUS_COMMITTED) {
                // 事务提交后执行
            }
        }
    });
}
```

# 9. 分布式事务方案对比

| 方案 | 一致性 | 适用场景 | 复杂度 |
|------|--------|----------|--------|
| XA/2PC（JTA） | 强一致 | 短事务、多个关系型数据库 | 高 |
| 本地消息表 | 最终一致 | 订单+消息可靠性 | 中 |
| TCC | 最终一致 | 需要强业务补偿控制 | 高 |
| Saga | 最终一致 | 长事务、微服务 | 中 |
| Seata AT | 最终一致 | 微服务 + 数据库 | 低（框架解决） |
| 消息最终一致性 | 最终一致 | 异步解耦 | 低 |

# 10. JTA vs Spring 事务

| JTA | Spring |
|-----|--------|
| `@TransactionAttribute` | `@Transactional` |
| `UserTransaction` | `TransactionTemplate` |
| `SessionContext.setRollbackOnly()` | `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| XA 两阶段提交 | 默认单数据源本地事务；多数据源可配 JTA 或 Seata |
| EJB 容器管理 | Spring AOP 代理管理 |
| `@Transactional` 属性对应 | `propagation`、`isolation`、`rollbackFor` |

## 10.1 经典陷阱：`@Transactional` 自调用失效

- Spring 事务基于代理，同类内 `this.method()` 自调用不经过代理，事务失效。
- EJB 中不存在此问题（容器基于字节码增强）。

# 11. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| 事务未回滚 | 检查异常是否为受检异常（需 `rollbackFor`）、是否被吞掉 |
| 长事务导致锁等待 | 缩小事务边界，避免事务内远程调用/IO |
| `TransactionRolledbackException` | 事务已被标记回滚，检查是否抛了运行时异常 |
| XA 悬挂 | 协调者日志中定位，手工 `xa recover` 恢复 |
| 事务内多数据源不一致 | 确认是否真正使用 JTA/XA 数据源，而非各回各的 |
