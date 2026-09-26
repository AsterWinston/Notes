# 1. 知识点
- **Seata 概述**：阿里巴巴开源的分布式事务解决方案，提供高性能、易扩展的分布式事务能力，由TC（事务协调器）、TM（事务管理器）、RM（资源管理器）三大组件组成。
- **四种事务模式**：AT（自动补偿型）、TCC（Try-Confirm-Cancel）、Saga（长事务）、XA（强一致），分别适用于不同的分布式事务场景。
- **AT模式**：基于两阶段提交协议，一阶段自动提交SQL并记录undo_log，二阶段根据全局事务状态执行提交或回滚，对业务代码无侵入。
- **TCC模式**：业务层手动实现Try（预留资源）、Confirm（执行业务）、Cancel（释放资源）三个接口，需要业务配合编写。
- **Saga模式**：长事务方案，由事件驱动的正向调用链组成，每个步骤提供补偿操作，失败时从失败点开始反向补偿。
- **XA模式**：基于数据库XA协议的两阶段提交，第一阶段预提交锁定资源，第二阶段全局提交或回滚，使用数据库原生事务支持。
- **TC（Transaction Coordinator）**：事务协调器，负责维护全局事务和分支事务的状态，驱动全局事务的提交或回滚。
- **TM（Transaction Manager）**：事务管理器，定义全局事务的范围，向TC发起全局事务的开始、提交、回滚指令，通常为注解`@GlobalTransactional`的方法所在服务。
- **RM（Resource Manager）**：资源管理器，管理分支事务，向TC注册、报告分支事务状态，驱动分支事务的提交或回滚，每个参与分布式事务的数据库连接对应一个RM。
- **XID（全局事务ID）**：全局事务的唯一标识，由TC生成，通过微服务间的RPC调用（Header透传）在整条调用链中传递，关联所有分支事务。
- **AT模式一阶段**：各RM执行SQL（INSERT/UPDATE/DELETE），解析SQL生成before-image和after-image，存入undo_log表，提交本地事务并释放数据库连接。
- **AT模式二阶段-提交**：TC通知各RM全局提交，RM异步删除undo_log记录并行提交本地事务（实际上第一阶段已完成提交）。
- **AT模式二阶段-回滚**：TC通知各RM全局回滚，RM根据undo_log中的before-image构建反向补偿SQL，执行补偿将数据恢复。
- **undo_log表**：存放每个分支事务的数据快照，包含xid、branch_id、rollback_info（压缩的before-image/after-image）、log_status、log_created等字段。
- **before-image**：执行SQL前的数据快照（行记录），用于回滚时恢复数据至修改前状态，记录在undo_log中。
- **after-image**：执行SQL后的数据快照，用于回滚前校验当前数据是否与after-image一致（防脏写），也记录在undo_log中。
- **全局锁**：AT模式为了防止并发事务的脏写，在一阶段注册分支事务时获取针对数据行的全局锁，确保回滚前的数据一致性。
- **全局锁检查**：在回滚前检查当前行数据是否与after-image一致，若不一致（被其他事务修改），代表发生了脏写，需人工介入处理或根据策略自动重试。
- **TCC模式-Try阶段**：业务代码预留/冻结资源，如冻结资金、锁定库存，资源尚未实际转移，但该资源对后续Confirm操作可用。
- **TCC模式-Confirm阶段**：执行业务操作，使用Try阶段预留的资源完成正式操作，必须有幂等设计，TC通知失败会重试。
- **TCC模式-Cancel阶段**：取消Try阶段预留的资源，恢复系统到事务开始前的状态，同样需要幂等设计。
- **TCC幂等性**：Confirm和Cancel接口都需实现幂等（通过业务流水号去重等机制），因为TC可能因网络重试多次调用。
- **Saga模式-无全局锁**：每个参与者提交自己的本地事务并发送消息触发下一个参与者，不持有全局锁，性能高但可能存在中间状态。
- **Saga模式-正向恢复**：失败时重新调度失败后续步骤继续推进（如果后续步骤仍然可用）。
- **Saga模式-反向补偿**：失败时执行每个已完成步骤的补偿操作回滚整个事务（Seata默认的反向补偿策略）。
- **XA模式一阶段**：各RM预提交SQL，数据库锁定相关行数据但不提交，数据库资源被占用阻塞直到二阶段结束。
- **XA模式二阶段**：TC通知各RM全局提交或回滚（COMMIT/ROLLBACK），数据库执行最终提交或回滚，释放锁。
- **XA vs AT**：XA需要数据库原生支持，第一阶段持有数据库锁到二阶段结束；AT第一阶段就提交释放锁，不阻塞数据库资源；AT使用undo_log补偿，XA使用数据库两阶段提交。
- **AT vs TCC对比**：AT隔离性靠全局写锁，TCC靠业务保证；AT一致性靠undo_log补偿；TCC性能更好（无锁等待）；AT开发成本低（无侵入），TCC开发成本高（需实现三个接口）。
- **XA vs AT vs TCC**：XA侵入性最低但阻塞数据库、性能最差；AT侵入性低、性能中等、通用性高；TCC侵入性最高但性能最好。

# 2. 面试题

## 2.1 Seata AT 模式的核心原理是什么？一阶段和二阶段分别做了什么？
AT模式是Seata最主流的分布式事务模式，基于两阶段提交协议实现自动回滚补偿。

**一阶段**：
1. 解析业务SQL，获取SQL类型（UPDATE/INSERT/DELETE）、表名、条件等信息
2. 在SQL执行前查询并保存数据快照（before-image）
3. 执行原SQL（业务数据写入数据库）
4. 在SQL执行后查询并保存数据快照（after-image）
5. 将before-image和after-image（压缩后）存入undo_log表
6. 注册分支事务到TC，获取该行数据的**全局锁**（防止并发脏写）
7. **提交本地事务**（undo_log和业务数据在同一个本地事务中提交）
8. 将本地事务提交结果上报TC

**二阶段（提交）**：
1. TC判定全局事务为提交状态
2. TC异步通知各RM执行提交
3. RM删除对应XID和BranchId的undo_log记录
4. 该分支事务完成

**二阶段（回滚）**：
1. TC判定全局事务为回滚状态
2. TC异步通知各RM执行回滚
3. RM根据undo_log中记录的after-image数据校验当前数据是否一致
4. 若一致，根据before-image构建反向补偿SQL并执行
5. 删除undo_log记录
6. 若after-image校验失败（数据被篡改），抛出异常需要人工介入

## 2.2 Seata AT 模式的全局锁与回滚校验机制
AT模式的全局锁主要用于防止分布式事务并发写导致的脏写问题。

**全局锁获取时机**：
- 一阶段注册分支事务时，RM向TC申请全局行锁
- TC检查该数据行是否已被其他全局事务锁定
- 如果未被锁定，获取锁成功，完成分支注册
- 如果已被锁定，等待锁释放或超时

**回滚时的after-image校验**：
- 二阶段回滚时，RM读取当前数据行
- 与undo_log中的after-image做比对
- 若一致：说明数据未被其他事务修改，执行补偿SQL恢复数据
- 若不一致：说明发生了脏写（数据被其他业务修改），抛出异常
- 此时需要人工介入或根据配置策略（如重试、告警）处理

**全局锁的特点**：
- 全局锁粒度基于数据行的主键值
- 锁由TC管理，跨数据库实例
- 默认全局锁超时时间为60秒
- 目的是在AT模式下提供读已提交级别的隔离性

## 2.3 Seata TCC 模式详解
TCC（Try-Confirm-Cancel）是业务层两阶段提交，由业务方手动实现三个阶段。

**Try阶段 — 资源预留**：
- 冻结/锁定业务资源，不执行真正的业务操作
- 例如：冻结账户金额（可用余额减少，冻结金额增加）
- 检查业务条件是否满足（余额是否充足、库存是否够用）

**Confirm阶段 — 执行业务**：
- 使用Try阶段预留的资源完成真正的业务操作
- 例如：正式扣减冻结金额、减少库存
- 必须是幂等的（多次执行结果相同）

**Cancel阶段 — 释放资源**：
- 回滚Try阶段预留的资源
- 例如：解冻被冻结的金额，恢复可用余额
- 必须是幂等的

**TCC示例（转账）**：
```java
// Try: A账户冻结100元，B账户不变
public boolean tryTransfer(String from, String to, BigDecimal amount) {
    return accountMapper.freeze(from, amount); // SQL: UPDATE balance=balance-amount, freeze=freeze+amount WHERE balance>=amount
}

// Confirm: A账户扣减冻结金额，B账户增加金额
public boolean confirmTransfer(String from, String to, BigDecimal amount) {
    boolean fromResult = accountMapper.confirmFreeze(from, amount); // UPDATE freeze=freeze-amount
    boolean toResult = accountMapper.addBalance(to, amount);      // UPDATE balance=balance+amount
    return fromResult && toResult;
}

// Cancel: A账户解冻金额
public boolean cancelTransfer(String from, String to, BigDecimal amount) {
    return accountMapper.unfreeze(from, amount); // UPDATE balance=balance+amount, freeze=freeze-amount
}
```

**TCC的幂等性保证**：
- 通过事务日志表 + 唯一事务ID去重
- 每个操作前先查询事务日志表，已处理则直接返回成功
- Confirm和Cancel操作都需检查并记录处理状态

## 2.4 Seata Saga 模式的适用场景和原理
Saga模式适用于**长事务**场景，不持有全局锁，由事件驱动。

**核心原理**：
1. 将长事务拆分为多个有序的本地事务
2. 每个本地事务有对应的**补偿操作**（反向操作）
3. 按顺序执行各本地事务，每个事务提交后发布事件触发下一步
4. 如果某步骤失败，反向执行已成功步骤的补偿操作

**Seata Saga的实现方式**：
- 使用状态机引擎，通过JSON定义状态流转
- JSON定义包含：状态名、要调用的服务、输入参数、补偿服务、状态转移条件等
- Seata 根据状态机JSON自动执行正向操作或反向补偿

**补偿 vs 回滚**：
- 补偿是业务层面的反向操作（如取消订单而非物理删除）
- 不同于数据库回滚，补偿后的数据可能与原始数据不一致（如事件通知已发出）
- 补偿操作也需要幂等性保证

**优点**：
- 无全局锁，性能高
- 事务可长时间运行（小时/天数级别）
- 数据不阻塞，适合大并发场景

**缺点**：
- 无隔离性，存在中间状态（可能被外部读取到不完整的数据）
- 补偿逻辑复杂，需要仔细设计
- 调试困难

## 2.5 Seata AT、TCC、Saga、XA 四种模式对比
| 维度 | AT | TCC | Saga | XA |
|------|-----|------|------|-----|
| 一致性 | 最终一致性 | 最终一致性 | 最终一致性 | 强一致性 |
| 隔离性 | 全局写锁（读未提交） | 业务保证 | 无（中间态可见） | 数据库锁（读已提交） |
| 性能 | 中（一阶段提交快，全局锁有开销） | 高（无锁） | 高（无锁、异步） | 低（阻塞到二阶段） |
| 对业务侵入 | 低（只需代理数据源） | 高（需实现3个接口） | 中（需提供补偿接口） | 低（需数据库支持XA） |
| 开发成本 | 低（自动回滚补偿） | 高（手动实现） | 中（需定义补偿状态机） | 低 |
| 适用场景 | 同数据库或简单跨库 | 性能要求高且可控 | 长事务/异步流程 | 对强一致性有要求 |
| 全局锁 | 有 | 无 | 无 | 有（数据库锁） |
| 数据库支持 | 任意关系数据库 | 任意数据库 | 任意数据库 | 需支持XA协议 |
| 回滚方式 | 自动undo_log补偿 | 手动Cancel | 状态机反向补偿 | 数据库ROLLBACK |

## 2.6 Seata 的 XA 模式与 AT 模式的核心区别
**XA模式**：
- 基于数据库原生XA协议（JTA）
- 一阶段：各RM预提交（PREPARE），锁定行数据但未提交
- 一阶段结束时：数据库锁依然持有，资源被阻塞
- 二阶段：执行COMMIT或ROLLBACK，释放锁
- 数据补偿由数据库完成，无需undo_log
- 问题：锁持有时间长，阻塞其他并发操作
- 适用：单一应用多数据源（如多库多表），或已有JTA基础的项目

**AT模式**：
- 基于undo_log的自动补偿
- 一阶段：本地事务直接提交，释放数据库锁，undo_log写入同事务中
- 一阶段结束时：数据库锁已释放，其他事务可并发操作
- 二阶段：通过undo_log的before-image反向补偿实现回滚
- 需要全局锁防止脏写
- 问题：存在"读未提交"的隔离性问题
- 适用：跨服务微服务分布式事务场景，性能敏感的绝大多数场景

## 2.7 Seata 的部署架构和核心组件
**三大核心组件**：

1. **TC（Transaction Coordinator）— Seata Server**：
   - 独立部署的服务进程
   - 负责维护全局事务和分支事务状态
   - 驱动全局事务的提交或回滚
   - 可以单机部署（file存储）或集群部署（DB存储+Nacos注册）
   - 集群部署时需要连接同一个数据库存储全局事务日志

2. **TM（Transaction Manager）— 事务发起方**：
   - 注解`@GlobalTransactional`标注的方法所在服务
   - 向TC发起全局事务的开启（获得XID）
   - 向TC发起全局事务的提交或回滚
   - 将XID通过RPC调用透传给下游服务

3. **RM（Resource Manager）— 事务参与方**：
   - 包装了数据源（`DataSourceProxy`）
   - 向TC注册分支事务（获得BranchId）
   - 报告分支事务执行状态
   - 执行分支事务的二阶段提交或回滚

**部署步骤**：
```yaml
# 1. 下载seata-server，修改配置
# registry.conf - 注册中心
registry:
  type: nacos
  nacos:
    server-addr: localhost:8848
    namespace: seata

# file.conf / application.yml - 存储模式
seata:
  store:
    mode: db  # db模式（推荐），或file模式
    db:
      datasource: druid
      db-type: mysql
```

```yaml
# 2. 业务服务配置（application.yml）
spring:
  cloud:
    alibaba:
      seata:
        tx-service-group: my_tx_group  # 事务分组
seata:
  tx-service-group: my_tx_group
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      application: seata-server
      group: SEATA_GROUP
```

## 2.8 @GlobalTransactional 注解的工作流程
```java
@Service
public class OrderService {
    @GlobalTransactional  // TM发起全局事务
    public void createOrder(OrderRequest request) {
        // 1. 本地操作：创建订单（RM-1）
        orderMapper.insert(order);
        
        // 2. 远程调用：扣减库存（RM-2）
        stockClient.deduct(request.getProductId(), request.getQuantity());
        
        // 3. 远程调用：扣减金额（RM-3）
        accountClient.debit(request.getUserId(), request.getAmount());
        
        // 如果任何步骤失败（抛异常），TC通知所有RM回滚
    }
}
```

**执行流程**：
1. 调用`createOrder`方法，TM（事务管理器）向TC（事务协调器）请求开启全局事务，获得XID
2. 将XID绑定到当前线程（通过RootContext）
3. 执行`orderMapper.insert(order)`：
   - RM拦截SQL，解析并保存before-image
   - 执行SQL写入
   - 保存after-image至undo_log
   - 向TC注册分支事务（获得BranchId），获取全局行锁
   - 提交本地事务
   - 上报分支事务状态（成功）
4. 调用`stockClient.deduct()`（Feign远程调用）：
   - 从RootContext中获取XID
   - 通过请求Header（`TX_XID`）将XID传递给下游服务
   - 下游服务同样经过"拦截SQL→before/after-image→注册分支→提交本地事务→上报TC"的流程
5. 调用`accountClient.debit()`：同步骤4
6. 所有分支事务成功：TM向TC发起全局提交，TC通知所有RM删除undo_log
7. 任何步骤失败：TM向TC发起全局回滚，TC通知所有RM执行undo_log补偿回滚

## 2.9 Seata AT 模式的优缺点及使用注意事项
**优点**：
1. 对业务代码零侵入，只需添加一个`@GlobalTransactional`注解
2. 自动生成undo_log回滚脚本，无需手动编写补偿逻辑
3. 一阶段就提交本地事务释放锁，数据库并发性能优于XA
4. 对数据库无特殊要求（不需要XA协议支持）
5. 支持多种框架（Spring Cloud、Dubbo、gRPC等）

**缺点**：
1. 需要额外维护undo_log表，有存储开销
2. 全局锁可能造成短暂的写冲突等待
3. 存在读未提交场景（一阶段提交后数据立即可见，全局事务仍在进行中）
4. 不适合高频写场景（全局锁、undo_log的写入和清除消耗性能）
5. after-image校验失败时需人工介入

**使用注意事项**：
1. 每个参与分布式事务的数据库都需要创建undo_log表
2. 确保undo_log和业务表在同一个数据库中
3. 业务SQL使用的表必须有主键（用于全局锁定位）
4. 生产环境TC必须集群部署并使用DB存储
5. 建议只在确实需要分布式事务的方法上加`@GlobalTransactional`
6. 监控undo_log表大小，定期清理过期记录
7. 设置合理的全局锁超时时间
