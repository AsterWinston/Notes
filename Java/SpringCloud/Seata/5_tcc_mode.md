
# 1. TCC 概述

TCC（Try-Confirm-Cancel）是**手动实现**的分布式事务模式，将每个参与方的业务操作拆分为三个阶段：

```
┌──────────────────────────────────────────────────────┐
│                    TCC 三阶段                          │
│                                                       │
│  Phase 1: TRY（预留资源）                              │
│    - 检查资源是否可用                                   │
│    - 预留/冻结业务资源                                  │
│    - 不真正执行操作                                     │
│                                                       │
│  Phase 2: CONFIRM（确认）                              │
│    - 所有 Try 成功 → 执行 Confirm                       │
│    - 使用预留资源完成实际操作                             │
│    - 幂等设计（允许多次调用）                            │
│                                                       │
│  Phase 3: CANCEL（取消）                               │
│    - 任一 Try 失败 → 执行 Cancel                       │
│    - 释放预留资源，回滚状态                              │
│    - 幂等设计 + 允许空回滚                               │
└──────────────────────────────────────────────────────┘
```

# 2. TCC 接口示例

```java
// 账户服务的 TCC 接口
@LocalTCC  // Seata TCC 注解
public interface AccountTccService {

    /**
     * Try：冻结余额
     * @param businessKey 业务标识，用于幂等和悬挂控制
     */
    @TwoPhaseBusinessAction(
        name = "accountDebit",
        commitMethod = "confirm",
        rollbackMethod = "cancel"
    )
    boolean tryDebit(
        @BusinessActionContextParameter(paramName = "userId") Long userId,
        @BusinessActionContextParameter(paramName = "amount") BigDecimal amount,
        @BusinessActionContextParameter(paramName = "businessKey") String businessKey
    );

    /**
     * Confirm：确认扣款
     * @param context 包含 Try 阶段的参数
     */
    boolean confirm(BusinessActionContext context);

    /**
     * Cancel：解冻余额
     * @param context 包含 Try 阶段的参数
     */
    boolean cancel(BusinessActionContext context);
}
```

## 2.1 实现类

```java
@Service
public class AccountTccServiceImpl implements AccountTccService {

    @Autowired
    private AccountMapper accountMapper;
    @Autowired
    private FreezeLogMapper freezeLogMapper;

    @Override
    @Transactional
    public boolean tryDebit(Long userId, BigDecimal amount, String businessKey) {
        // 1. 悬挂控制：如果 Cancel 先于 Try 到达（网络延迟）
        //    检查 freeze_log 是否已被 Cancel
        int exists = freezeLogMapper.exists(businessKey, "CANCEL");
        if (exists > 0) {
            return false;  // Cancel 已执行，Try 无需处理
        }

        // 2. 幂等控制：检查是否已执行过 Try
        int frozen = freezeLogMapper.exists(businessKey, "TRY");
        if (frozen > 0) {
            return true;   // 已冻结，幂等返回
        }

        // 3. 冻结余额
        int rows = accountMapper.freezeBalance(userId, amount);
        if (rows == 0) {
            throw new RuntimeException("余额不足");
        }

        // 4. 记录冻结日志（用于幂等和悬挂判断）
        freezeLogMapper.insert(businessKey, "TRY", userId, amount);

        return true;
    }

    @Override
    @Transactional
    public boolean confirm(BusinessActionContext context) {
        String businessKey = context.getActionContext("businessKey", String.class);
        Long userId = context.getActionContext("userId", Long.class);

        // 幂等控制
        int exists = freezeLogMapper.exists(businessKey, "CONFIRM");
        if (exists > 0) {
            return true;
        }

        // 确认扣款（从冻结余额转为实际扣款）
        accountMapper.confirmDebit(userId);

        freezeLogMapper.insert(businessKey, "CONFIRM", userId, null);
        return true;
    }

    @Override
    @Transactional
    public boolean cancel(BusinessActionContext context) {
        String businessKey = context.getActionContext("businessKey", String.class);
        Long userId = context.getActionContext("userId", Long.class);

        // 幂等控制 + 空回滚标记
        int exists = freezeLogMapper.exists(businessKey, "CANCEL");
        if (exists > 0) {
            return true;
        }

        // 解冻余额
        accountMapper.unfreezeBalance(userId);

        // 记录 Cancel 日志（防止后续 Try 到达：悬挂控制）
        freezeLogMapper.insert(businessKey, "CANCEL", userId, null);
        return true;
    }
}
```

# 3. 三大难题

## 3.1 空回滚（Empty Rollback）

**场景**：Try 阶段网络超时，TM 认为 Try 失败发起 Cancel，但 RM 实际上收到了 Try 请求并执行。

```
TM 发起 Try ──×──（网络超时）──→ RM
TM 认为 Try 失败，发起 Cancel ──→ RM (Cancel 先到达)
                                    ├── RM 尚未执行 Try
                                    └── 此时执行 Cancel = 空回滚
```

**解决**：Cancel 操作必须允许**空回滚**——即使 Try 未执行，Cancel 也能正常运行（返回成功，不报错）。

## 3.2 悬挂（Suspension）

**场景**：空回滚之后，延迟的 Try 请求到达。

```
TM 发起 Try ────────（网络延迟）──────→ RM (Try 晚到达)
TM 发起 Cancel ──→ RM (Cancel 先到达，空回滚)
                   ├── RM 记录 Cancel 日志
                   └── 之后 Try 到达 → 检查到已有 Cancel
                                    └── 拒绝执行（悬挂控制）
```

**解决**：Try 执行前检查是否已有 Cancel 记录，若有则拒绝。

## 3.3 幂等控制

**场景**：Confirm/Cancel 因网络异常被重复调用。

**解决**：通过 `businessKey` 唯一标识 + 操作日志表判断是否已执行。

# 4. 资源表设计

```sql
-- 账户表
CREATE TABLE account (
    user_id    BIGINT PRIMARY KEY,
    balance    DECIMAL(10,2) NOT NULL,
    frozen     DECIMAL(10,2) NOT NULL DEFAULT 0  -- 冻结金额
);

-- 冻结操作日志（幂等 + 悬挂控制）
CREATE TABLE freeze_log (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    business_key  VARCHAR(64) NOT NULL,         -- 业务唯一键
    action        VARCHAR(10) NOT NULL,         -- TRY / CONFIRM / CANCEL
    user_id       BIGINT,
    amount        DECIMAL(10,2),
    create_time   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_business_action (business_key, action)
);
```

# 5. 调用方（TM）

```java
@Service
public class OrderService {

    @DubboReference
    private AccountTccService accountTccService;

    @DubboReference
    private InventoryTccService inventoryTccService;

    @GlobalTransactional
    public void createOrder(Order order) {
        // 1. 本地操作
        orderMapper.insert(order);

        // 2. TCC Try（冻结资源）
        accountTccService.tryDebit(
            order.getUserId(), order.getAmount(),
            "ORDER_" + order.getId()
        );

        // 3. TCC Try（预占库存）
        inventoryTccService.tryDeduct(
            order.getProductId(), order.getQuantity(),
            "ORDER_" + order.getId()
        );

        // Seata 自动：
        // - 所有 Try 成功 → 调用每个服务的 Confirm
        // - 任一 Try 失败 → 调用每个服务的 Cancel
    }
}
```

# 6. TCC vs AT 对比

| 维度 | AT 模式 | TCC 模式 |
|------|---------|---------|
| 侵入性 | 低（仅代理数据源） | 中（需实现 TCC 接口） |
| 回滚方式 | 自动（undo_log） | 手动（Cancel 方法） |
| 性能 | 中 | 高（无 undo_log 开销） |
| 适用资源 | 仅数据库 | 数据库 + 外部 RPC + 消息 |
| 实现复杂度 | 低 | 高（需处理幂等/空回滚/悬挂） |
| 锁粒度 | 全局锁（AT 锁） | 业务层预留（更灵活） |

# 7. 适用场景

- ✓ 对性能要求极高的场景
- ✓ 涉及非数据库资源（缓存、消息队列、第三方 API）
- ✓ 可以改造成 "预留-确认-取消" 模型的业务
- ✗ 不想修改业务代码的场景（用 AT）
- ✗ 业务逻辑难以拆分为三阶段的场景
