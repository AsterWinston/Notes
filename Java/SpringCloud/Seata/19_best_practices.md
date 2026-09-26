
# 1. 分布式事务使用原则

## 1.1 避免过度使用

```
✓ 必须用分布式事务：
  - 跨数据库的资金操作
  - 跨服务的核心业务数据变更
  - 强一致性要求的场景

✗ 不需要分布式事务：
  - 最终一致性可接受的场景 → 使用消息队列 + 本地事务
  - 可以合并到同一个数据库的 → 重构数据存储
  - 非关键数据的同步 → 异步补偿
```

## 1.2 事务拆分原则

```java
// ❌ 大事务：包含不必要操作
@GlobalTransactional(timeoutMills = 300000)
public void createOrder(Order order) {
    orderMapper.insert(order);
    accountClient.debit(userId, amount);       // 核心：必须
    inventoryClient.deduct(productId, qty);     // 核心：必须
    notificationClient.sendEmail(userId);       // 非核心：不应在事务中
    logClient.record(order);                    // 非核心：不应在事务中
}

// ✓ 拆分：核心操作用分布式事务，非核心操作异步
@GlobalTransactional(timeoutMills = 60000)
public void createOrder(Order order) {
    orderMapper.insert(order);
    accountClient.debit(userId, amount);
    inventoryClient.deduct(productId, qty);
}

// 事务外异步处理
completableFuture.runAsync(() -> {
    notificationClient.sendEmail(userId);
    logClient.record(order);
});
```

# 2. 模式选择决策流程

```
1. 是否所有操作都是关系型数据库？
   ├── 是 → 2
   └── 否 → 3

2. 是否需要强一致性？
   ├── 是 → XA 模式
   └── 否 → AT 模式（首选）

3. 是否可以改造成 TCC 三阶段？
   ├── 是 → TCC 模式
   └── 否 → Saga 模式
```

# 3. 超时配置

```yaml
# 超时配置黄金法则：
# 业务方法执行时间 < Seata 超时 < Dubbo/Feign 超时 < 上游超时

@GlobalTransactional(timeoutMills = 30000)  # 30s Seata 事务超时

dubbo:
  consumer:
    timeout: 40000    # 40s RPC 超时

spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 5000
            read-timeout: 40000
```

# 4. 异常处理

```java
// ✓ 正确：抛出让 Seata 感知的异常
@Service
public class AccountService {

    public void debit(Long userId, BigDecimal amount) {
        int rows = accountMapper.debit(userId, amount);
        if (rows == 0) {
            // 抛出业务异常 → Seata 检测 → TC 驱动回滚
            throw new InsufficientBalanceException(userId);
        }
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            // 大额交易需要审批 → 通知 TM 回滚
            throw new LargeAmountException(userId, amount);
        }
    }
}

// ✗ 错误：吞掉异常
public void debit(Long userId, BigDecimal amount) {
    try {
        accountMapper.debit(userId, amount);
    } catch (Exception e) {
        log.error("debit failed", e);
        // Seata 无法感知异常 → 不会回滚！
    }
}
```

# 5. undo_log 管理

```sql
-- 每个参与 AT 模式的业务库都需要
CREATE TABLE `undo_log` ( ... );

-- 定期清理已提交的 undo_log（建议每天定时任务）
DELETE FROM undo_log
WHERE log_status = 1
  AND log_modified < DATE_SUB(NOW(), INTERVAL 7 DAY)
LIMIT 5000;

-- 监控 undo_log 表大小
SELECT
  table_name,
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_name = 'undo_log';
```

# 6. 生产配置检查清单

```yaml
# ========== 必须配置 ==========
seata:
  registry:
    type: nacos                         # 注册中心类型
  config:
    type: nacos                         # 配置中心类型
  store:
    mode: db                            # 存储模式：db
  data-source-proxy-mode: AT            # 代理模式

# ========== 建议配置 ==========
seata:
  client:
    tm:
      default-global-transaction-timeout: 60000  # 60s 全局超时
    rm:
      async-commit-buffer-limit: 10000
    lock:
      retry-interval: 10
      retry-times: 30
    undo:
      data-validation: true
      only-care-update-columns: true
```

# 7. 监控指标

```
建议监控以下指标：

1. TC Server
   - global_table 行数（活跃事务数）
   - branch_table 行数（活跃分支数）
   - lock_table 行数（持锁数）
   - undo_log 行数（待清理数）
   - TC JVM 堆内存/GC

2. Client
   - @GlobalTransactional 方法执行时间
   - 全局事务提交/回滚比率
   - Branch 注册失败率
   - RootContext.bind() / unbind() 匹配情况
```

# 8. 不要做的事

- ✗ 不要在分布式事务中调用外部不可控的 HTTP API
- ✗ 不要在 `@GlobalTransactional` 方法中起异步线程（需手动传递 xid）
- ✗ 不要设置过长的超时时间（会导致全局锁长时间持有）
- ✗ 不要在生产环境使用 file 存储模式的 TC
- ✗ 不要在异常处理中吞掉业务异常
- ✗ 不要在事务中包含大量非核心操作
- ✗ 不要混合使用不同版本的 Seata Client 和 TC Server
