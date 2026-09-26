
# 1. 概述

Seata 通过 Dubbo SPI 机制自动集成，实现 xid 在 RPC 调用中**透明传播**，无需修改任何 Dubbo 配置。

```
Consumer (TM)                         Provider (RM)
      │                                     │
      │ Dubbo RPC Call                      │
      │ RpcInvocation.attachment:           │
      │   TX_XID = 192.168.1.100:8091:xxx   │
      │ ────────────────────────────────→   │
      │                                     │ SeataDubboFilter
      │                                     │ 提取 xid → RootContext.bind()
      │                                     │ 注册分支事务
      │                                     │ 执行业务逻辑
```

# 2. 依赖与配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

**无需额外依赖**，`spring-cloud-starter-alibaba-seata` 已包含 Dubbo 集成所需的 Filter。

# 3. Filter 注册机制

Seata 通过 Dubbo SPI 注册两个 Filter：

```
META-INF/dubbo/org.apache.dubbo.rpc.Filter:

seata=org.apache.dubbo.seata.SeataTransactionPropagationFilter
```

## 3.1 Consumer 端传播

```java
@Activate(group = {CommonConstants.CONSUMER}, order = 100)
public class SeataTransactionPropagationFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation)
            throws RpcException {
        // 1. 从 RootContext 获取 xid
        String xid = RootContext.getXID();
        if (xid != null) {
            // 2. 将 xid 放入 Dubbo 的 RpcInvocation.attachments
            invocation.setAttachment(RootContext.KEY_XID, xid);
            invocation.setAttachment(RootContext.KEY_XID.toLowerCase(), xid);
        }
        return invoker.invoke(invocation);
    }
}
```

## 3.2 Provider 端接收

```java
@Activate(group = {CommonConstants.PROVIDER}, order = -100)
public class SeataTransactionPropagationProviderFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation)
            throws RpcException {
        // 1. 从 Dubbo attachment 中提取 xid
        String xid = invocation.getAttachment(RootContext.KEY_XID);
        if (xid == null) {
            xid = invocation.getAttachment(RootContext.KEY_XID.toLowerCase());
        }
        
        if (xid != null) {
            // 2. 绑定到 RootContext（当前线程）
            RootContext.bind(xid);
        }

        try {
            // 3. 执行 Provider 业务逻辑
            //    DataSourceProxy 检测到 RootContext 中有 xid
            //    → 自动注册分支事务
            return invoker.invoke(invocation);
        } finally {
            // 4. 清理 xid
            if (xid != null) {
                RootContext.unbind();
            }
        }
    }
}
```

# 4. Provider 端完整配置

```yaml
dubbo:
  application:
    name: account-service
  registry:
    address: nacos://localhost:8848
  protocol:
    name: dubbo
    port: 20880

seata:
  tx-service-group: default_tx_group
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: seata
      application: seata-server
      group: SEATA_GROUP
  config:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: seata
      group: SEATA_GROUP
      data-id: seata.properties
  data-source-proxy-mode: AT
```

# 5. 分布式事务示例

## 5.1 Consumer（TM 发起方）

```java
@Service
public class OrderService {

    @DubboReference
    private AccountService accountService;

    @DubboReference
    private InventoryService inventoryService;

    @Autowired
    private OrderMapper orderMapper;

    @GlobalTransactional
    public void createOrder(Order order) {
        // 1. 本地 AT 分支
        orderMapper.insert(order);

        // 2. Dubbo RPC → 远程 AT 分支
        //    SeataDubboFilter 自动传播 xid
        accountService.debit(order.getUserId(), order.getAmount());

        // 3. Dubbo RPC → 远程 AT 分支
        inventoryService.deduct(order.getProductId(), order.getQuantity());
    }
}
```

## 5.2 Provider（RM 参与方）

```java
@DubboService
public class AccountServiceImpl implements AccountService {

    @Autowired
    private AccountMapper accountMapper;

    @Override
    public void debit(Long userId, BigDecimal amount) {
        // DataSourceProxy 检测到 RootContext 中有 xid
        // → 自动生成 undo_log，注册分支事务到 TC
        accountMapper.debit(userId, amount);

        // 业务校验
        BigDecimal balance = accountMapper.getBalance(userId);
        if (balance.compareTo(BigDecimal.ZERO) < 0) {
            throw new InsufficientBalanceException(userId);
        }
    }
}
```

# 6. TCC + Dubbo 集成

```java
// Consumer 调用 TCC
@GlobalTransactional
public void createOrder(Order order) {
    orderMapper.insert(order);

    // TCC Try
    accountTccService.tryDebit(
        order.getUserId(), order.getAmount(),
        "ORDER_" + order.getId()
    );
}

// Provider 实现 TCC
@DubboService
public class AccountTccServiceImpl implements AccountTccService {

    @Override
    @Transactional
    public boolean tryDebit(Long userId, BigDecimal amount, String businessKey) {
        // 检查是否已存在 Cancel（悬挂控制）
        // 冻结余额
        // 记录冻结日志（幂等）
        // ...
    }
}
```

# 7. 嵌套调用 xid 传播

```
TM (OrderService)
  │ @GlobalTransactional
  │ xid: 192.168.1.100:8091:2039718144
  │
  ├──→ RM (AccountService / Dubbo)
  │      ├── RootContext: xid=192.168.1.100:8091:2039718144
  │      └── 注册分支事务 branchId=2039718145
  │           │
  │           └──→ RM (NotificationService / Dubbo)
  │                  └── RootContext: xid=192.168.1.100:8091:2039718144
  │                      注册分支事务 branchId=2039718146
  │
  └── 全局提交 → TC 驱动所有分支提交
```

# 8. 超时配置

```yaml
seata:
  client:
    tm:
      commit-retry-count: 5
      rollback-retry-count: 5
      default-global-transaction-timeout: 300000    # 5 分钟

    rm:
      report-retry-count: 5
      async-commit-buffer-limit: 10000

dubbo:
  provider:
    timeout: 5000       # Dubbo 超时应大于 Seata 事务处理时间
  consumer:
    timeout: 5000
```

**注意事项**：Dubbo 的超时时间应充分考虑 Seata 的全局事务跨度，避免 RPC 超时先于 Seata 事务完成导致异常。

# 9. 最佳实践

1. **Filter 顺序**：确保 `SeataTransactionPropagationFilter` 在 Consumer 端 Filter 链的最前面执行
2. **超时对齐**：`Dubbo:timeout > Seata:事务超时 > 业务方法执行时间`
3. **异常透传**：RM 端业务异常会通过 Dubbo 透传到 Consumer，触发全局回滚
4. **Context 清理**：Provider 端必须 `RootContext.unbind()` 防止内存泄漏
5. **undo_log 独立**：每个参与全局事务的数据库都需要 `undo_log` 表
