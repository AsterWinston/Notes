
# 1. Saga 模式概述

Saga 模式是**长事务**解决方案，将一个大事务拆分为多个有序的**本地事务**，每个本地事务有对应的**补偿操作**。如果某个步骤失败，则逆序执行已成功步骤的补偿操作。

```
正向流程:
  Step1(创建订单) → Step2(扣减库存) → Step3(扣减余额) → 完成

异常回滚（逆序补偿）:
  Step1(创建订单) → Step2(扣减库存) → Step3(扣减余额) ✗ 失败
       ▲                ▲                │
       │                │                ▼
  Compensate1       Compensate2      (无需补偿，本地事务已回滚)
  (取消订单)        (恢复库存)
```

# 2. Saga 实现方式

Seata 提供两种 Saga 实现方式：

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| **状态机（JSON DSL）** | 通过 JSON 配置定义状态流转 | 推荐，可编排、可视化 |
| **注解** | `@SagaTransactional` + `@Compensable` | 简单场景 |

# 3. 状态机 JSON DSL（推荐）

## 3.1 状态机定义

```json
{
  "Name": "createOrderSaga",
  "Comment": "创建订单 Saga",
  "StartState": "CreateOrder",
  "Version": "1.0",
  "States": {
    "CreateOrder": {
      "Type": "ServiceTask",
      "ServiceName": "orderService.create()",
      "CompensateState": "CancelOrder",
      "Next": "DeductInventory"
    },
    "DeductInventory": {
      "Type": "ServiceTask",
      "ServiceName": "inventoryService.deduct()",
      "CompensateState": "RestoreInventory",
      "Next": "DebitBalance"
    },
    "DebitBalance": {
      "Type": "ServiceTask",
      "ServiceName": "accountService.debit()",
      "CompensateState": "RefundBalance",
      "Next": "Succeed"
    },
    "CancelOrder": {
      "Type": "ServiceTask",
      "ServiceName": "orderService.cancel()",
      "Next": "Fail"
    },
    "RestoreInventory": {
      "Type": "ServiceTask",
      "ServiceName": "inventoryService.restore()",
      "Next": "Fail"
    },
    "RefundBalance": {
      "Type": "ServiceTask",
      "ServiceName": "accountService.refund()",
      "Next": "Fail"
    },
    "Succeed": {
      "Type": "Succeed"
    },
    "Fail": {
      "Type": "Fail",
      "ErrorCode": "ORDER_CREATE_FAILED",
      "Message": "订单创建失败，已执行补偿"
    }
  }
}
```

## 3.2 状态机引擎 API

```java
// 启动状态机
@Autowired
private StateMachineEngine stateMachineEngine;

public void createOrder(Order order) {
    // 通过状态机引擎执行 Saga
    StateMachineInstance instance = stateMachineEngine.start(
        "createOrderSaga",       // 状态机名称
        null,                     // 租户 ID
        order.getId(),            // 业务键
        new HashMap<String, Object>() {{
            put("order", order);
        }}
    );

    // 获取执行状态
    ExecutionStatus status = instance.getStatus();
    if (status == ExecutionStatus.FA) {
        throw new SagaException(instance.getCompensationStatus());
    }
}
```

## 3.3 服务实现

```java
@Service
public class OrderSagaService {

    @Autowired
    private OrderMapper orderMapper;

    // 正向操作
    public void create(Order order) {
        orderMapper.insert(order);
    }

    // 补偿操作
    public void cancel(Order order) {
        orderMapper.updateStatus(order.getId(), OrderStatus.CANCELLED);
    }
}
```

# 4. 注解方式

```java
@Service
public class OrderSagaService {

    @SagaTransactional(
        applicationId = "order-service",
        transactionName = "createOrder"
    )
    public void createOrder(Order order) {
        // Step 1
    }

    @Compensable(
        compensationBean = OrderSagaService.class,
        compensationMethod = "cancelOrder"
    )
    public void create(Order order) {
        orderMapper.insert(order);
    }

    public void cancelOrder(Order order) {
        orderMapper.updateStatus(order.getId(), OrderStatus.CANCELLED);
    }
}
```

# 5. Saga 的核心特征

## 5.1 ACD 特性

| 特性 | Saga | ACID 事务 |
|------|------|----------|
| **A**tomicity | 最终原子（通过补偿） | 严格原子 |
| **C**onsistency | 最终一致 | 强一致 |
| **I**solation | 无隔离（中间状态可见） | 完全隔离 |
| **D**urability | 持久 | 持久 |

## 5.2 补偿的设计原则

1. **幂等**：补偿操作可被重复调用
2. **可交换**：正向操作和补偿操作的执行顺序不影响最终结果
3. **最终成功**：补偿操作应设计为必然成功（重试 + 人工干预兜底）

# 6. Saga vs AT vs TCC

| 维度 | AT | TCC | Saga |
|------|-----|-----|------|
| 事务时长 | 短（秒级） | 短-中 | 长（分钟/小时级） |
| 锁持有 | 全局锁（AT锁） | 业务预留 | 无锁 |
| 中间状态可见 | 否（AT锁隔离） | 是（资源已预留） | 是 |
| 补偿方式 | 自动 undo_log | 手动 Cancel | 手动补偿方法 |
| 性能 | 中 | 高 | 高 |
| 一致性 | 读未提交（默认） | 业务定义 | 最终一致 |

# 7. 适用场景

- ✓ 长事务（跨分钟/小时/天）
- ✓ 多服务编排，流程复杂
- ✓ 无法用 AT/TCC 的异构系统（如调用外部 HTTP API）
- ✓ 不需要隔离性的场景（如日志、统计分析）
- ✗ 需要强隔离性的金融交易
- ✗ 简单的事务（用 AT 更省事）
