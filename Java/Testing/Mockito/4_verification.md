# 1. 行为验证

打桩回答"依赖应该返回什么"，验证回答"被测代码到底对依赖做了什么"。Mockito 的 `verify` 系列 API 可以检查方法是否被调用、调用了几次、参数是什么、先后顺序如何。验证很强大，也最容易被滥用：一旦测试开始关心实现细节，重构就会变得寸步难行。

## 1.1 verify 的基本语义

最基础的验证是 `verify(mock).method(args)`，它表示"该方法恰好被调用一次，并且参数匹配"：

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class NotificationServiceTest {

    @Mock
    private EmailClient emailClient;

    @Test
    void should_send_welcome_email() {
        NotificationService service = new NotificationService(emailClient);

        service.welcome("ada@example.com");

        // 验证邮件客户端被调用一次，参数精确匹配
        verify(emailClient).send("ada@example.com", "欢迎注册");
    }
}
```

`verify(emailClient)` 默认等价于 `verify(emailClient, times(1))`。如果方法被调用零次或两次以上，测试都会失败。参数匹配遵循上一篇的规则：可以和 `eq`、`any`、`argThat` 混用，但同一个调用里不能混用原始值。

**设计考量**：验证的落脚点应该是"可观察的协作"，而不是"内部实现步骤"。发送邮件、扣减库存、发布事件属于外部可见的副作用，适合验证；至于服务内部先查缓存还是先查数据库，通常不应该成为测试的断言对象。

## 1.2 调用次数与验证模式

Mockito 提供了一组 `VerificationMode`，用来表达不同的次数约束：

| 模式 | 含义 | 典型场景 |
|------|------|----------|
| `times(n)` | 恰好调用 `n` 次 | 精确控制重试、批量次数 |
| `never()` | 从未调用，等价于 `times(0)` | 校验短路、权限拒绝、幂等 |
| `atLeast(n)` | 至少 `n` 次 | 至少触发一次告警 |
| `atMost(n)` | 至多 `n` 次 | 防止重复提交 |
| `atLeastOnce()` | 至少一次 | 大多数"应该发生"的场景 |
| `only()` | 该交互只发生一次 | 需要确认没有其他同类调用 |
| `calls(n)` | 恰好 `n` 次，语义更强调"调用次数" | 可读性偏好 |

```java
import static org.mockito.Mockito.atLeast;
import static org.mockito.Mockito.atMost;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;

// 恰好两次
verify(repository, times(2)).save(any(Order.class));

// 至少一次、至多三次
verify(alertService, atLeast(1)).send(anyString());
verify(rateLimiter, atMost(3)).tryAcquire();

// 明确表达"不应该发生"
verify(repository, never()).deleteById(anyLong());
```

`never()` 在测试防御性逻辑时特别有用：用户没有权限时，不应该调用删除接口；订单金额为负时，不应该调用支付网关。它比断言返回值为 `null` 更能说明"没有产生副作用"。

> **注意**：`verifyNoMoreInteractions` 和 `never()` 不是一回事。`never()` 针对某个具体方法，`verifyNoMoreInteractions` 针对整个 Mock 对象上的所有未验证调用。前者通常安全，后者容易过度约束。

## 1.3 verifyNoInteractions 与 verifyNoMoreInteractions

`verifyNoInteractions(mock)` 断言这个 Mock 完全没有被调用过：

```java
import static org.mockito.Mockito.verifyNoInteractions;

@Test
void should_not_touch_audit_log_when_validation_fails() {
    service.process(invalidOrder);

    // 校验失败时，审计日志不应被写入
    verifyNoInteractions(auditLog);
}
```

`verifyNoMoreInteractions(mock)` 则断言"除了前面已经验证过的调用之外，没有其他调用"：

```java
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoMoreInteractions;

@Test
void should_only_save_once() {
    service.create(order);

    verify(repository).save(order);
    verifyNoMoreInteractions(repository);
}
```

这两种写法在小型测试里可以防止"多做了事情"，但在业务测试里非常脆弱。只要被测代码以后新增一次合法调用（例如保存后发一条领域事件），测试就会失败，而业务行为可能完全正确。

> **行业实践**：`verifyNoMoreInteractions` 更适合基础设施代码或协议客户端测试，用来确保没有发送多余请求。普通业务服务测试不建议默认加上它，否则会把测试变成"调用序列快照"。

## 1.4 InOrder 顺序验证

有些协作顺序具有业务含义：必须先扣库存再生成订单，必须先写主表再写明细表。`InOrder` 可以验证多个 Mock 之间的调用顺序：

```java
import org.mockito.InOrder;
import static org.mockito.Mockito.inOrder;

@Test
void should_deduct_stock_before_creating_order() {
    service.placeOrder(order);

    InOrder inOrder = inOrder(inventoryService, orderRepository);

    // 先扣库存
    inOrder.verify(inventoryService).deduct(order.sku(), order.quantity());
    // 再落订单
    inOrder.verify(orderRepository).save(any(Order.class));
}
```

`InOrder` 只验证你列出的调用之间的相对顺序，不要求它们连续发生。也就是说，两次被验证的调用之间可以夹杂其他 Mock 的其他调用，只要顺序正确即可。

```java
// 同一个 Mock 上也可以验证顺序
InOrder inOrder = inOrder(repository);
inOrder.verify(repository).save(any());
inOrder.verify(repository).flush();
```

如果需要确认"没有更多未验证的调用"，可以调用 `inOrder.verifyNoMoreInteractions()`。但同样要谨慎，它会把后续新增的合法调用也判为失败。

**设计考量**：顺序验证应该只用于**业务上确实有先后依赖**的场景。如果两个调用只是恰好按当前实现顺序执行，用 `InOrder` 会把实现细节固化成契约，增加重构成本。

## 1.5 超时验证

异步代码中，方法调用可能发生在另一个线程。普通 `verify` 在调用发生前就会执行，导致偶发失败。Mockito 提供了 `timeout` 和 `after` 两种时间相关的验证模式：

```java
import static org.mockito.Mockito.after;
import static org.mockito.Mockito.timeout;
import static org.mockito.Mockito.verify;

@Test
void should_deliver_async_event() {
    service.publishAsync(event);

    // 在 500ms 内等待事件被投递一次
    verify(eventBus, timeout(500)).publish(event);
}

@Test
void should_not_retry_after_success() throws Exception {
    service.execute();

    // 至少等待 200ms，确认没有发生额外调用
    verify(gateway, after(200).never()).retry(any());
}
```

时间参数的单位是毫秒。`timeout(500).times(2)` 表示在 500ms 的等待窗口内等待两次调用出现，超时仍未达到两次则验证失败。`timeout` 只负责等待，次数语义仍由 `times` 决定，不能拿它替代真正的同步机制。

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `timeout(ms)` | 在超时前等待调用发生，默认一次 | 异步投递、回调 |
| `timeout(ms).times(n)` | 在超时前等待至少 `n` 次 | 批量异步处理 |
| `after(ms)` | 先等待指定时间，再执行普通验证 | 确认"没有发生" |
| `after(ms).never()` | 等待后确认从未调用 | 防重试、防重复消费 |

> **警告**：`timeout` 不是同步原语。它只能缓解"验证执行得太早"的问题，不能修复被测代码里的竞态条件。真正稳定的异步测试应该使用 `CountDownLatch`、`CompletableFuture` 或 Awaitility 等工具等待明确的状态，而不是靠固定毫秒数碰运气。

## 1.6 使用 ArgumentCaptor 捕获参数

有时参数对象是动态生成的，无法在验证时精确构造。`ArgumentCaptor` 可以先捕获实际参数，再用断言库检查其内容：

```java
import org.mockito.ArgumentCaptor;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.verify;

@Test
void should_capture_saved_order() {
    service.createOrder("BOOK-1", 2);

    ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
    verify(orderRepository).save(captor.capture());

    Order saved = captor.getValue();
    assertEquals("BOOK-1", saved.sku());
    assertEquals(2, saved.quantity());
    assertEquals(OrderStatus.CREATED, saved.status());
}
```

如果方法被调用多次，可以用 `getAllValues()` 获取全部参数：

```java
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(orderRepository, times(2)).save(captor.capture());

List<Order> orders = captor.getAllValues();
assertEquals(2, orders.size());
```

在 JUnit 5 扩展下，还可以用 `@Captor` 自动创建：

```java
import org.mockito.Captor;

@Captor
private ArgumentCaptor<Order> orderCaptor;

@Test
void should_capture_with_annotation() {
    service.createOrder("BOOK-1", 2);

    verify(orderRepository).save(orderCaptor.capture());
    assertEquals("BOOK-1", orderCaptor.getValue().sku());
}
```

捕获器有几个使用限制。第一，`captor.capture()` 返回 `null`，所以它只能出现在 `verify` 或 `when` 的参数位置。第二，如果同一个方法有多个参数，使用捕获器的同时其他参数也必须用匹配器。第三，捕获到的是对象引用，如果被测代码在调用后继续修改该对象，断言看到的是修改后的状态。

## 1.7 过度验证与测试脆弱性

验证是一把双刃剑。下面这段测试看似严谨，实际上把实现细节全部锁死了：

```java
// 过度验证：任何一次重构都会让测试失败
verify(cache).get("user:1");
verify(repository).findById(1L);
verify(mapper).toDto(user);
verify(logger).info("查询用户 {}", 1L);
verify(metrics).increment("user.query");
```

这段测试的问题不在于断言错了，而在于它把"先查缓存、再查数据库、再转换、再打日志、再记指标"当成了业务契约。将来把缓存改成 `CacheManager`，或者把日志级别从 `info` 调到 `debug`，测试都会失败，但业务行为没有变化。

**判断标准**：如果一次重构只改变内部实现、不改变可观察行为，测试是否应该失败？答案是否。基于这个标准，优先断言返回值、持久化状态、发送出去的消息内容，而不是内部调用的每一个步骤。

| 反模式 | 问题 | 替代方案 |
|--------|------|----------|
| 验证每个内部方法调用 | 与实现强耦合 | 只验证跨边界协作 |
| 到处 `verifyNoMoreInteractions` | 新增合法调用就失败 | 只验证关键副作用 |
| 用 `times(1)` 显式写出默认值 | 冗余，噪音 | 直接 `verify(mock)` |
| 用 `InOrder` 验证无业务意义的顺序 | 固化实现 | 只对强顺序依赖使用 |
| 验证日志、指标等观测代码 | 观测实现经常调整 | 用集成测试或单独测试覆盖 |
| 验证 Mock 自己的桩调用 | 循环论证 | 断言被测方法结果 |

## 1.8 常见错误与排错

| 现象 | 原因 | 修正 |
|------|------|------|
| `Wanted but not invoked` | 方法没被调用，或调用的是另一个 Mock 实例 | 检查注入是否成功，确认 `verify` 的对象就是被测类持有的对象 |
| `TooManyActualInvocations` | 实际调用次数超过预期 | 用 `atLeast` 表达下限，或检查是否重复调用 |
| `NeverWantedButInvoked` | 用了 `never()` 但发生了调用 | 检查短路逻辑和异常分支 |
| 参数匹配失败 | 使用精确值但对象不相等 | 改用 `argThat` 或 `ArgumentCaptor` |
| `InvalidUseOfMatchersException` | `verify` 中混用匹配器和原始值 | 全部参数统一用匹配器 |
| 异步测试偶发失败 | `verify` 执行早于异步调用 | 用 `timeout`/Awaitility，不要用 `Thread.sleep` |
| 验证 Spy 时触发真实逻辑 | Spy 未打桩的方法会执行真实代码 | 先 `doReturn` 打桩，或改用 Mock |
| 测试顺序变化导致失败 | 共享了 Mock 状态 | 每个测试独立初始化，避免静态共享 |

还有一个常见误区：**用 `verify` 验证查询方法的返回值**。如果测试关心"返回了什么"，应该直接断言被测方法的返回值；`verify` 只适合验证"发生了什么副作用"。两者混用会让测试同时绑定结果和实现，维护成本翻倍。

# 2. 练习与自检

1. 验证 `repository.save` 恰好调用两次，同时验证 `repository.delete` 从未调用。
2. 用 `InOrder` 验证"先扣库存、再保存订单"的顺序，并说明为什么只在有业务顺序要求时使用它。
3. 为什么 `verifyNoMoreInteractions` 容易让测试变脆？给出一个更稳妥的替代断言。

# 3. 小结

验证的价值在于确认"副作用是否发生"，而不是"代码是否按我设想的方式执行"。使用时应优先选择语义明确的次数约束，只在业务确实要求顺序时使用 `InOrder`，只在异步场景使用 `timeout`，并用 `ArgumentCaptor` 把动态参数取出来做结构化断言。

| 需求 | 推荐 API |
|------|----------|
| 验证调用一次 | `verify(mock).method(args)` |
| 验证调用次数 | `times(n)` / `atLeast(n)` / `atMost(n)` |
| 验证从未调用 | `never()` |
| 验证完全没有交互 | `verifyNoInteractions(mock)` |
| 验证顺序 | `InOrder` |
| 验证异步调用 | `timeout(ms)` / `after(ms)` |
| 捕获动态参数 | `ArgumentCaptor` |

下一篇 `5_spy_and_inject_mocks.md` 转向测试替身的创建与注入，重点讨论 `@Mock`、`@Spy`、`@InjectMocks` 的差异，以及 Spy 在部分 Mock 场景下的真实调用风险。
