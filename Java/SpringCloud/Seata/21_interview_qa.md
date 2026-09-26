
# 1. 基础题

## 1.1 什么是分布式事务？为什么需要 Seata？

分布式事务是指事务的参与者分布在不同的服务或数据库中。在微服务架构下，一次业务请求可能跨多个服务和数据库，无法用单一数据库的 ACID 事务保证一致性。

Seata 提供开箱即用的分布式事务解决方案，支持 AT/TCC/Saga/XA 四种模式，与 Spring Cloud/Dubbo 生态无缝集成。

## 1.2 Seata 的 TC/TM/RM 三个角色分别是什么？

| 角色 | 职责 | 实例 |
|------|------|------|
| **TC** (Transaction Coordinator) | 维护全局事务状态，协调分支提交/回滚 | Seata Server |
| **TM** (Transaction Manager) | 定义全局事务边界：开启/提交/回滚 | `@GlobalTransactional` 方法 |
| **RM** (Resource Manager) | 管理分支事务资源，注册分支并上报状态 | 各服务的数据库操作 |

## 1.3 AT/TCC/Saga/XA 四种模式的核心区别？

| 模式 | 回滚方式 | 侵入性 | 性能 | 锁机制 |
|------|---------|--------|------|--------|
| AT | 自动 undo_log | 无 | 中高 | 全局锁 |
| TCC | 手动 Cancel | 高 | 高 | 业务预留 |
| Saga | 手动补偿 | 中 | 高 | 无锁 |
| XA | 数据库 XA 协议 | 无 | 低 | 数据库行锁 |

# 2. 原理题

## 2.1 AT 模式的 undo_log 是如何工作的？

1. **Phase 1**：DataSourceProxy 拦截 SQL → 查前置镜像 → 执行业务 SQL → 查后置镜像 → 生成 undo_log（包含前后镜像） → 注册分支 → 本地提交
2. **Phase 2 Commit**：TC 通知 → 异步删除 undo_log
3. **Phase 2 Rollback**：TC 通知 → 根据 undo_log 生成反向 SQL → 执行补偿 → 删除 undo_log

关键：undo_log 存储了每行数据的前后镜像，回滚时通过对比还原。

## 2.2 AT 模式如何解决脏写问题？

通过**全局锁**：Phase 1 提交前向 TC 注册分支并申请全局锁（记录在 `lock_table` 中），其他全局事务操作同一行数据时需检查全局锁，若已被持有则等待或失败。

## 2.3 TCC 模式的三大难题是什么？

1. **空回滚**：Try 未执行但 Cancel 先到 → Cancel 必须允许空回滚
2. **悬挂**：Cancel 空回滚后，延迟的 Try 到达 → Try 需检查已有 Cancel 记录
3. **幂等**：Confirm/Cancel 可能被重复调用 → 通过唯一键 + 状态表保证幂等

## 2.4 xid 是如何在微服务调用中传播的？

- **Dubbo**：`SeataTransactionPropagationFilter` 在 Consumer 端将 xid 放入 `RpcInvocation.attachment`，Provider 端提取并绑定到 `RootContext`
- **Feign**：`SeataFeignRequestInterceptor` 将 xid 放入 HTTP Header，服务端 `SeataHandlerInterceptor` 提取
- **跨线程**：`xid` 存储在 ThreadLocal 中，异步线程需手动传递

# 3. 实践题

## 3.1 如何选择分布式事务模式？

```
决策流程：
1. 只涉及数据库？
   → 需要强一致？ XA
   → 不想改代码？ AT
   → 追求性能？ TCC（改代码）

2. 涉及非数据库资源？
   → TCC 或 Saga

3. 长事务？
   → Saga
```

## 3.2 如何优化 Seata AT 模式的性能？

1. 批量操作代替逐条操作（减少 undo_log 次数）
2. 定期清理 undo_log 表
3. 启用 AT 模式异步提交
4. 增大连接池（AT 模式额外 SQL 占用连接）
5. 控制事务粒度：只包含核心操作
6. 缩短全局事务超时时间

## 3.3 TCC 的悬挂和空回滚怎么处理？

使用业务状态表：Try 执行前检查是否有 Cancel 记录（防悬挂）；Cancel 执行时检查 Try 是否已执行（防空回滚）。

```sql
CREATE TABLE tcc_action (
    business_key VARCHAR(64),
    action VARCHAR(10),       -- TRY/CONFIRM/CANCEL
    UNIQUE KEY (business_key, action)
);
```

# 4. 架构题

## 4.1 Seata 如何保证高可用？

- **TC 集群**：3 节点通过 Nacos 注册发现，DB 共享存储
- **故障转移**：Client 自动从 Nacos 获取可用 TC 节点列表
- **事务恢复**：新 TC 从 DB 恢复未完成的事务
- **支持 Raft**：Seata 1.5+ 支持 Raft 模式

## 4.2 AT 模式 vs XA 模式的锁差异？

| 维度 | AT | XA |
|------|-----|-----|
| 锁类型 | 全局锁（TC 维护） | 数据库行锁 |
| 锁持有时间 | Phase 1 ~ Phase 2 | Phase 1 Prepare ~ Phase 2 |
| 锁释放 | Phase 2 完成 | Phase 2 完成 |
| 并发性能 | 较高（锁释放后其他事务可读） | 较低（长锁阻塞） |

## 4.3 CAP 在 Seata 中的体现？

Seata 选择 **AP + BASE**：
- 牺牲强一致性（C）：AT/TCC/Saga 都是最终一致性
- 保证可用性（A）：即使 TC 短暂不可用，业务仍可降级运行
- 满足分区容错（P）：TC 集群化部署

XA 模式例外，它倾向于 CP（牺牲部分可用性换取强一致）。

# 5. 对比题

## 5.1 Seata vs 可靠消息最终一致性？

| 维度 | Seata | 可靠消息 |
|------|-------|---------|
| 实时性 | 秒级 | 秒-分钟级 |
| 回滚 | 自动/手动 | 需补偿逻辑 |
| 依赖 | Seata Server | RocketMQ/Kafka |
| 复杂度 | 中 | 中高 |
| 适用 | 同步调用链 | 异步解耦 |

## 5.2 Seata AT vs ShardingSphere-Proxy XA？

| 维度 | Seata AT | ShardingSphere XA |
|------|---------|-------------------|
| 定位 | 分布式事务中间件 | 数据库中间件 |
| 侵入性 | 低（DataSourceProxy） | 无（代理层） |
| 适用范围 | 微服务跨服务 | 分库分表跨库 |
| 事务模式 | AT/TCC/Saga/XA | XA/Seata AT |

# 6. 加分题

## 6.1 Seata 的 GlobalLock 注解有什么作用？

`@GlobalLock` 仅获取全局锁但不参与全局事务。用于需要读已提交隔离级别的场景。它的 SELECT FOR UPDATE 会检查 lock_table 中是否有未提交事务的锁。

## 6.2 Seata AT 模式的 rollback 失败怎么办？

1. TC 接收到 RM 的 rollback 失败通知 → 标记分支状态为 `PhaseTwo_RollbackFailed_Retryable`
2. TC 按 `rollback-retry-count` 重试
3. 全部失败 → 记录到 `branch_table` → 人工介入
4. **预防措施**：开启 `undo.data-validation=true`，校验 after image 避免脏写
