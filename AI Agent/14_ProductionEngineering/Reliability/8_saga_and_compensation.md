# 1. 为什么需要 Saga

**结论先行**：Agent 常执行跨多个系统的长事务，分布式事务（2PC）在微服务与 LLM（Large Language Model，大语言模型）场景下不可行，因此需要 Saga（长事务补偿）来保证最终一致性。

## 1.1 长事务的挑战

### 1.1.1 跨系统副作用

Agent 的一次任务往往横跨支付、库存、订单、消息等多个系统，每个系统只支持本地事务，无法用一个数据库事务包住。

```
下单流程：
  ① 扣款（支付系统）
  ② 扣库存（库存系统）
  ③ 创建订单（订单系统）
  ④ 发通知（消息系统）

若 ③ 失败，①② 已执行的副作用必须回滚
```

### 1.1.2 失败场景

| 失败点 | 已产生的副作用 | 需要做的事 |
|--------|----------------|------------|
| ① 扣款失败 | 无 | 直接失败 |
| ② 扣库存失败 | 已扣款 | 退款 |
| ③ 创建订单失败 | 已扣款、已扣库存 | 退款、恢复库存 |
| ④ 发通知失败 | 已扣款、已扣库存、已建单 | 退款、恢复库存、取消订单 |

### 1.1.3 与普通数据库事务的区别

普通事务可以真正"回滚"到过去；Saga 的补偿是"向前抵消"，中间状态已经对外可见，且补偿本身也是业务操作，可能失败。

| 维度 | 数据库事务 | Saga |
|------|------------|------|
| 回滚语义 | 物理回滚 | 业务抵消 |
| 中间态可见 | 否 | 是 |
| 补偿会失败 | 否 | 会 |
| 跨系统 | 否 | 是 |

## 1.2 为什么不用 2PC

### 1.2.1 2PC 的局限

两阶段提交（2PC，Two-Phase Commit）要求所有参与方支持准备/提交协议、持有锁直到全局提交，在微服务与外部 API 场景下几乎不可实现。

| 维度 | 2PC | Saga |
|------|-----|------|
| 一致性 | 强一致 | 最终一致 |
| 锁持有 | 长 | 无全局锁 |
| 外部系统 | 需支持协议 | 只需可补偿 |
| 可用性 | 低（协调者单点） | 高 |
| 适用 | 同库/同中间件 | 微服务、跨系统 |

### 1.2.2 Saga 的取舍

Saga 用"最终一致性 + 补偿"换取可用性。代价是：中间状态对外可见，且补偿不是回滚到过去，而是用一个新操作抵消旧操作。

## 1.3 Saga 的适用边界

- 适合：跨系统、长耗时、每步有明确反向操作的流程；
- 不适合：强一致要求的资金清算核心账务（需专门的对账与冲正体系）；
- 慎用：不可补偿且不可逆的操作（如已送达的实物）。

| 特征 | 是否适合 Saga |
|------|---------------|
| 每步有反向操作 | 适合 |
| 允许最终一致 | 适合 |
| 需要强一致 | 不适合 |
| 存在不可逆操作 | 慎用 |
| 单库单事务可解决 | 不需要 Saga |

# 2. Saga 的核心思想

**结论先行**：把长事务拆成多个本地事务，每个步骤配一个补偿操作；任一步失败，就逆序执行已成功步骤的补偿。

## 2.1 正向与补偿

### 2.1.1 拆分本地事务

```
把长事务拆成多个本地事务
每个步骤配一个"补偿操作"（反向操作）
任一步失败 → 反向执行已成功步骤的补偿
```

### 2.1.2 反向补偿

```
正向：T1 → T2 → T3 → T4
补偿：C1 ← C2 ← C3 ← C4
```

注意：补偿是逆序执行（从失败点往回），且只有**已成功**的步骤才需要补偿。

## 2.2 补偿表设计

### 2.2.1 示例

| 步骤 | 正向操作 | 补偿操作 |
|------|----------|----------|
| 1 | 扣款 | 退款 |
| 2 | 扣库存 | 恢复库存 |
| 3 | 创建订单 | 取消订单 |
| 4 | 发通知 | 无需补偿（幂等通知） |

### 2.2.2 设计要点

- 每个正向步骤都要在设计阶段明确其补偿操作；
- 补偿操作需接收正向步骤产生的业务 ID（如 paymentId、reservationId）；
- 无法补偿的步骤应尽量后置。

## 2.3 补偿与重试的区别

补偿（Compensation）是"反向操作"，重试（Retry）是"再试一次正向操作"。二者不可混淆：失败后应先重试正向，仍失败才触发补偿。

```
正向失败 → 重试正向(可幂等) → 仍失败 → 补偿
```

重试要求正向操作幂等；若正向不幂等，重试本身就会产生重复副作用，此时应直接进入补偿。

# 3. 两种协调方式

**结论先行**：Saga 有两种协调方式——编排式（中央协调器）与协同式（事件驱动）；Agent 场景推荐编排式。

## 3.1 编排式

### 3.1.1 结构

由中央协调器控制流程，协调器负责调用各服务并记录补偿栈。

```
        ┌──────────────┐
        │  OrderSaga   │
        └──┬───┬───┬───┘
           │   │   │
      ┌────▼┐ ┌▼───┐ ┌▼────┐
      │支付 │ │库存│ │订单 │
      └─────┘ └────┘ └─────┘
```

### 3.1.2 完整实现

```java
public class OrderSaga {

    private final Deque<Runnable> compensations = new ArrayDeque<>();

    public OrderResult execute(OrderRequest req) {
        try {
            // ① 扣款
            String paymentId = paymentService.charge(req);
            compensations.push(() -> paymentService.refund(paymentId));

            // ② 扣库存
            String reservationId = inventoryService.reserve(req);
            compensations.push(() -> inventoryService.release(reservationId));

            // ③ 创建订单
            Order order = orderService.create(req);
            compensations.push(() -> orderService.cancel(order.id()));

            // ④ 发通知
            notificationService.notify(order);

            return OrderResult.ok(order);

        } catch (Exception e) {
            compensate();
            throw new SagaFailedException("下单失败，已回滚", e);
        }
    }

    private void compensate() {
        while (!compensations.isEmpty()) {
            try {
                compensations.pop().run();
            } catch (Exception e) {
                // 补偿失败 → 必须告警，人工介入
                log.error("补偿失败，需人工处理", e);
                alertService.alert("SAGA_COMPENSATION_FAILED", e);
            }
        }
    }
}
```

## 3.2 协同式

### 3.2.1 事件流

各服务通过事件协作，无中央协调器：

```
订单服务 → 发出 OrderCreated 事件
   ↓
支付服务 → 监听，执行扣款 → 发出 PaymentCharged
   ↓
库存服务 → 监听，扣库存 → 发出 InventoryReserved
   ↓ 失败
支付服务 → 监听 InventoryFailed → 退款
```

### 3.2.2 对比

| 方式 | 优点 | 缺点 |
|------|------|------|
| 编排式 | 流程清晰、易调试 | 协调器可能成为瓶颈 |
| 协同式 | 松耦合 | 流程分散、难追踪 |

## 3.3 选型建议

> Agent 场景推荐**编排式**：流程显式、易审计、易补偿。

| 场景 | 推荐方式 |
|------|----------|
| 步骤少、跨服务少 | 协同式 |
| 步骤多、需审计 | 编排式 |
| Agent 工具链 | 编排式 |
| 高吞吐、松耦合 | 协同式 |

# 4. 补偿的设计原则

**结论先行**：补偿必须幂等、逆序、有超时、失败必告警；这四条是补偿正确性的底线。

## 4.1 幂等

### 4.1.1 为什么必须幂等

补偿可能因重试或恢复而被多次执行。若补偿非幂等，重复退款会导致资金错误。

| 原则 | 说明 |
|------|------|
| 补偿必须幂等 | 补偿可能被重试 |
| 补偿不能失败 | 尽量设计为不会失败的操作 |
| 补偿失败要告警 | 人工介入 |
| 补偿要考虑顺序 | 逆序执行 |
| 补偿要有超时 | 避免卡住 |
| 补偿要可审计 | 记录补偿过程 |

### 4.1.2 实现

```java
public record CompensationStep(
        String name,
        Runnable action,
        int maxRetries
) {}
```

幂等实现通常依赖业务幂等键（如 refundId = "refund:" + paymentId），重复调用返回同一结果。

```java
public RefundResult refund(String paymentId) {
    String idempotencyKey = "refund:" + paymentId;
    RefundResult existing = refundStore.findByKey(idempotencyKey);
    if (existing != null) {
        return existing;                 // 已退款，直接返回
    }
    RefundResult result = paymentGateway.refund(paymentId);
    refundStore.save(idempotencyKey, result);
    return result;
}
```

## 4.2 顺序与超时

### 4.2.1 逆序执行

补偿必须与正向严格逆序，否则可能违反业务约束（如先恢复库存再退款，会导致库存短暂虚增被他人占用）。

### 4.2.2 超时控制

每个补偿操作都要设超时，避免单个补偿卡死整条 Saga。超时后记录并继续，最终统一告警。

```java
CompletableFuture.runAsync(step.action(), compensationExecutor)
        .get(step.timeoutSeconds(), TimeUnit.SECONDS);
```

## 4.3 补偿失败处理

补偿失败无法自动恢复，必须：记录失败步骤、写入死信表、触发告警、进入人工处理队列。绝不能静默吞掉。

死信表至少记录 sagaId、失败步骤、补偿操作、错误信息与重试次数，供人工重放。

# 5. 补偿的类型

**结论先行**：补偿不总是"撤销"，也可能是"部分恢复"或"语义抵消"；理解类型才能正确排序。

## 5.1 分类

### 5.1.1 四种类型

| 类型 | 说明 | 例子 |
|------|------|------|
| 完全补偿 | 恢复到初始状态 | 退款 |
| 部分补偿 | 只能部分恢复 | 已发出的邮件无法撤回 |
| 不可补偿 | 无法撤销 | 已送达的通知 |
| 语义补偿 | 用新操作抵消 | 反向记账 |

### 5.1.2 排序原则

> 对**不可补偿**的操作，应尽量**放到流程最后**（如发通知放最后）。

```
可补偿步骤（扣款/扣库存/建单）
        ↓
不可补偿步骤（发通知/发实物）
```

## 5.2 不可补偿操作的处理

若不可补偿操作必须在中间，则需业务兜底：如发送"更正通知"、人工回访、对账冲正。

```
正向：T1(扣款) → T2(发短信) → T3(建单)
                    ↑ 不可补偿，失败时短信已发出

补救：T1 退款，短信无法撤回，追加一条更正短信
```

# 6. 与 Agent 工具的结合

**结论先行**：Agent 的工具调用链天然是 Saga——每个工具是一次本地事务，失败时按注册表逆序补偿。

## 6.1 执行与补偿

### 6.1.1 AgentSagaExecutor

```java
public class AgentSagaExecutor {

    private final List<ExecutedAction> executed = new ArrayList<>();

    public String executeWithCompensation(List<ToolCall> calls) {
        try {
            for (ToolCall call : calls) {
                ToolResult result = toolExecutor.execute(call);
                if (!result.success()) {
                    throw new ToolFailedException(call.name(), result.error());
                }
                executed.add(new ExecutedAction(call, result));
            }
            return "全部执行成功";

        } catch (Exception e) {
            compensateAll();
            return "执行失败，已回滚：" + e.getMessage();
        }
    }

    private void compensateAll() {
        // 逆序补偿
        for (int i = executed.size() - 1; i >= 0; i--) {
            ExecutedAction action = executed.get(i);
            ToolCall compensation = compensationRegistry.compensationFor(action.call());

            if (compensation == null) continue;      // 无补偿操作

            try {
                toolExecutor.execute(compensation);
            } catch (Exception e) {
                log.error("补偿失败: {}", compensation.name(), e);
                alertService.alert("COMPENSATION_FAILED", compensation.name());
            }
        }
    }
}
```

### 6.1.2 补偿注册表

```java
@Component
public class CompensationRegistry {

    private final Map<String, Function<Map<String, Object>, ToolCall>> registry;

    public CompensationRegistry() {
        registry = Map.of(
                "charge",      args -> ToolCall.of("refund", args),
                "reserveStock", args -> ToolCall.of("releaseStock", args),
                "createOrder",  args -> ToolCall.of("cancelOrder", args)
        );
    }

    public ToolCall compensationFor(ToolCall call) {
        Function<Map<String, Object>, ToolCall> fn = registry.get(call.name());
        return fn == null ? null : fn.apply(call.arguments());
    }
}
```

## 6.2 工具补偿的边界

- LLM 生成的工具调用参数可能不完整，补偿前需校验必填 ID；
- 只读工具无需补偿，注册表返回 null 即可；
- 补偿工具本身也要有超时与幂等。

# 7. 持久化 Saga 状态

**结论先行**：长流程必须持久化，否则进程崩溃后既无法继续正向，也无法继续补偿。

## 7.1 表设计

### 7.1.1 表结构

```sql
CREATE TABLE saga_instance (
    saga_id      VARCHAR(64) PRIMARY KEY,
    status       VARCHAR(16) NOT NULL,   -- RUNNING / COMPLETED / COMPENSATING / FAILED
    steps        JSONB       NOT NULL,   -- 步骤与状态
    created_at   TIMESTAMP   NOT NULL,
    updated_at   TIMESTAMP   NOT NULL
);
```

### 7.1.2 步骤状态

steps 字段记录每一步的名称、状态（PENDING/DONE/COMPENSATED）、业务 ID，用于恢复时判断从何处继续。

| 状态 | 含义 |
|------|------|
| RUNNING | 正向执行中 |
| COMPLETED | 全部成功 |
| COMPENSATING | 正在补偿 |
| FAILED | 补偿失败，需人工 |

## 7.2 恢复

### 7.2.1 恢复流程

```java
public void resumeSaga(String sagaId) {
    SagaInstance saga = sagaStore.load(sagaId);

    if (saga.status() == SagaStatus.RUNNING) {
        // 继续执行未完成步骤（幂等）
        continueForward(saga);
    } else if (saga.status() == SagaStatus.COMPENSATING) {
        // 继续补偿
        continueCompensation(saga);
    }
}
```

### 7.2.2 幂等恢复

恢复时必须依赖每步的业务幂等键，避免重复扣款或重复退款。这与状态持久化中的幂等要求一致。

## 7.3 并发与恢复

同一 Saga 可能被多个恢复线程同时处理，需加分布式锁或乐观锁，保证同一 sagaId 只有一个执行者。

```java
public void resumeSaga(String sagaId) {
    withSagaLock(sagaId, () -> {
        SagaInstance saga = sagaStore.load(sagaId);
        if (saga.status() == SagaStatus.RUNNING) {
            continueForward(saga);
        }
    });
}
```

# 8. 与工作流引擎结合

**结论先行**：复杂 Saga 建议交给工作流引擎，自己实现只适合简单场景。

## 8.1 引擎能力

复杂 Saga 建议交给工作流引擎（Flowable/Temporal），它们原生支持：

- 补偿（BPMN 补偿事件 / Temporal Saga）；
- 状态持久化；
- 重试与超时；
- 可视化。

## 8.2 选型

| 引擎 | 特点 | 适用 |
|------|------|------|
| Temporal | 代码即工作流、强持久化 | 微服务长流程 |
| Flowable | BPMN 标准、可视化 | 业务流程编排 |
| Camunda | BPMN + 运营工具 | 企业流程 |

> 自己实现 Saga 只适合简单场景；复杂流程用引擎更稳。

## 8.3 自研与引擎的取舍

| 维度 | 自研 | 工作流引擎 |
|------|------|------------|
| 上手成本 | 低 | 高 |
| 复杂流程 | 难维护 | 强 |
| 持久化 | 需自建 | 原生 |
| 可视化 | 无 | 有 |
| 适用 | 3~4 步简单流程 | 多步、需审计 |

# 9. 生产注意事项

**结论先行**：Saga 的坑集中在补偿幂等、补偿失败、状态持久化三点，必须逐一防守。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 补偿非幂等 | 补偿必须幂等 |
| 补偿失败静默 | 告警 + 人工介入 |
| 不可补偿操作放中间 | 放到流程最后 |
| 状态不持久化 | 长 Saga 必须持久化 |
| 补偿无超时 | 设超时避免卡住 |
| 无审计 | 记录每步与补偿 |
| 并发恢复 | 加锁防重复执行 |
| 中间态被误读 | 对外标注处理中 |

## 9.2 反例与检查清单

反例：补偿栈只存内存，进程崩溃后无法补偿；补偿接口无幂等键，重试导致重复退款；把发通知放在扣款之前，失败时无法撤回。

| 检查项 | 要求 |
|--------|------|
| 每步补偿 | 已定义 |
| 补偿幂等 | 有幂等键 |
| 补偿超时 | 已设置 |
| 状态持久化 | 已落库 |
| 告警 | 补偿失败有告警 |
| 审计 | 每步与补偿可追溯 |
| 恢复 | 崩溃后可续跑 |

# 10. 小结

- Saga 用"正向 + 补偿"实现长事务回滚。
- 编排式流程清晰，Agent 场景推荐。
- 补偿必须幂等、有超时、失败要告警。
- 不可补偿操作尽量放到流程最后。
- 复杂 Saga 交给工作流引擎。
- 恢复与并发必须加锁，防止重复执行。
