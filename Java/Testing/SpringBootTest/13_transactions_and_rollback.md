# 1. 测试事务模型

Spring Test 给事务性测试提供了一个"默认帮你收拾残局"的模型：**测试方法运行在一个由测试框架管理的事务里，方法结束自动回滚**。这让数据库测试可以随便写数据而不留垃圾，但代价是——回滚也顺手抹掉了"真实提交"这一段被测范围。理解这个模型的能力边界，比会用注解更重要。

## 1.1 默认事务行为

在 Spring Test 中，标注 `@Transactional` 的测试默认会在**测试结束后回滚**：

```java
@Transactional           // 测试方法包进事务，结束后回滚
@SpringBootTest
class TransactionTest {

    @Test
    void should_save_order() {
        orderRepository.save(new Order(null, "A-1"));  // 事务内可见
        assertThat(orderRepository.count()).isEqualTo(1);
    }   // 方法返回 → 框架回滚 → 数据库无痕
}
```

几个必须精确的点：

- **`@SpringBootTest` 本身不带事务**。默认回滚的前提是测试类或方法上有 `@Transactional`；`@DataJpaTest`、`@JdbcTest` 这类数据切片是**元注解里自带了 `@Transactional`**，所以"切片测试默认回滚"。
- 事务范围是**整个测试方法**：`@BeforeEach`/`@AfterEach` 里的数据库操作默认也在同一事务内（`SqlScriptsTestExecutionListener` 的 `@Sql` 脚本同样默认加入）。
- 测试代码与生产代码**共享这个事务**：被测 Service 的 `@Transactional(REQUIRED)` 方法会加入测试事务，而不是另开一个。这正是"测试里调 Service，数据没提交也能查到"的原因。

## 1.2 事务监听器

`TransactionalTestExecutionListener` 是 TestContext 框架默认的 `TestExecutionListener` 之一（无需注册，`spring.factories`/`TestExecutionListeners` 默认清单里），负责：

- 在 `beforeTestMethod` 时通过 `PlatformTransactionManager` **开启事务**（`TransactionDefinition` 取自测试类/方法上的 `@Transactional`）
- 在 `afterTestMethod` 时**默认回滚**，处理 `@Rollback` 与 `@Commit` 声明
- 维护 `TransactionContext`，让 `TransactionSynchronizationManager` 上绑定的资源（连接、EntityManager）随测试生命周期走

```text
┌───────────────────────────────────────────────────────────┐
│         TransactionalTestExecutionListener 生命周期         │
│                                                           │
│  beforeTestMethod                                         │
│    │                                                      │
│    ├─ 有 @Transactional?                                   │
│    │    ├─ 无 → 不开事务，测试裸跑（写完自己负责清理）          │
│    │    └─ 有 → 用 TransactionManager 开启事务              │
│    │             绑定连接到当前线程                          │
│    ▼                                                      │
│  测试方法执行（生产代码 REQUIRED 传播 → 加入该事务）            │
│    │                                                      │
│    ▼                                                      │
│  afterTestMethod                                          │
│    ├─ @Commit / @Rollback(false) → 提交                    │
│    └─ 默认 → 回滚，连接解绑归还                               │
└───────────────────────────────────────────────────────────┘
```

---

# 2. `@Rollback` 与 `@Commit`

## 2.1 强制回滚

回滚是默认行为，`@Rollback` 显式写出来通常是为了对抗类级别的提交声明：

```java
@Commit                       // 类级：整个测试类默认提交
@SpringBootTest
class CommitByDefaultTest {

    @Rollback                 // 方法级：这一个测试仍回滚
    @Test
    void rollbackTest() { }
}
```

`@Rollback` 是 `@Transactional` 事务收尾策略的独立开关，可以标在类或方法上，方法级覆盖类级。

## 2.2 强制提交

`@Commit` 等价于 `@Rollback(false)`，让测试事务真提交：

```java
@Commit
@Test
void commitTest() {
    orderRepository.save(new Order(null, "A-1"));
}   // 数据真的落库——你必须自己清理！
```

适合验证提交后行为：数据库触发器、`@TransactionalEventListener(AFTER_COMMIT)` 监听器、依赖唯一索引落库后生效的场景。**代价是数据残留**，配套动作：

```java
@Commit
@Sql(scripts = "/sql/cleanup-orders.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)  // 方法后清表
@Test
void should_fire_trigger_after_commit() { }
```

> **行业实践**：提交型测试集中在少数"必须见真提交"的用例上，且每个都配 AFTER 清理脚本或用容器化一次性数据库；把整套测试都改成 `@Commit` + 清表，等于放弃了框架送的隔离性，flaky 概率立刻上升。

---

# 3. 传播与隔离

## 3.1 常见传播行为

| 传播级别 | 说明 | 测试事务内的表现 |
|----------|------|------------------|
| `REQUIRED` | 默认，加入现有事务 | 加入测试事务，被测逻辑与测试同生共死 |
| `REQUIRES_NEW` | 新建事务，挂起外部事务 | **新事务独立提交/回滚，与测试事务隔离** |
| `NESTED` | 保存点嵌套 | 基于 savepoint，回滚到保存点不影响外层 |
| `NOT_SUPPORTED` | 非事务执行 | 挂起测试事务，裸连接执行 |
| `SUPPORTS` | 有事务就用，没有就裸跑 | 加入测试事务 |
| `NEVER` | 有事务就抛异常 | **在测试事务里必抛 `IllegalTransactionStateException`** |
| `MANDATORY` | 没事务就抛异常 | 依赖测试事务才能跑 |

## 3.2 测试中的意义

- 使用 `REQUIRES_NEW` 验证事务边界（见 3.3 的坑）；
- 使用 `NOT_SUPPORTED` 验证非事务查询——注意它会**挂起**测试事务，此时被测代码看到的是"已提交数据"，测试事务里未提交的写入它看不到；
- 隔离级别（`@Transactional(isolation = ...)`）在测试事务上声明后，整个测试方法都用该隔离级别，但**内嵌 H2 对隔离级别的支持很弱**（基本只有 READ_COMMITTED 语义），要测隔离行为必须上真库。

## 3.3 REQUIRES_NEW 的坑

`REQUIRES_NEW` 是测试事务模型里最经典的陷阱。测试事务回滚不了它：

```java
@Service
class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(String msg) {
        auditRepository.save(new Audit(msg));   // 独立事务，立即提交
    }
}

@Transactional                 // 测试事务
@DataJpaTest
class AuditLeakTest {

    @Autowired
    private AuditService auditService;

    @Test
    void demo() {
        auditService.record("leak-me");
    }   // 测试事务回滚了，但 "leak-me" 已经真提交进库——数据残留！
}
```

机制：`REQUIRES_NEW` 挂起外层测试事务，用**另一个连接**开新事务，方法返回即提交。回滚只作用于外层事务，内层的写入永久生效。推论：

- 被测代码含 `REQUIRES_NEW` 时，回滚型测试**并不零残留**；
- 反过来，`REQUIRES_NEW` 内也**看不到**测试事务里未提交的数据（不同连接、READ_COMMITTED 下互不可见）——"测试里先 save 再调 REQUIRES_NEW 方法查它"会查不到，这不是 bug，是隔离语义。

应对：这类测试改用容器化一次性库，或显式 `@Commit` + AFTER 清理，或把 `REQUIRES_NEW` 部分 Mock 掉单独测。

---

# 4. `@Sql` 脚本支持

`@Sql` 与事务的交互规则决定了要不要写清理脚本：

```java
@Sql(scripts = "classpath:seed.sql")                          // 默认 BEFORE，加入测试事务
@Sql(scripts = "classpath:cleanup.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)   // 方法后清理
class SqlCleanupTest { }
```

- **回滚型测试**：BEFORE 脚本的写入随事务回滚，**无需清理脚本**；
- **提交型测试**：BEFORE 脚本也提交（`SqlConfig` 默认 `INFERRED`——有测试事务就加入，没有就独立事务执行并立即生效），必须配 AFTER 清理；
- 想让脚本**不参与**测试事务（比如种子数据要跨测试方法可见），用 `@SqlConfig(transactionMode = TransactionMode.ISOLATED)`。

---

# 5. 事务与 `@DirtiesContext`

两者解决的是不同层面的"残留"：

| 手段 | 清理对象 | 成本 |
|------|----------|------|
| 事务回滚 | **数据**残留 | 近乎零（一次 rollback） |
| `@DirtiesContext` | **容器状态**污染（单例 Bean 内存状态） | 高（销毁并重建 ApplicationContext） |

事务回滚通常比 `@DirtiesContext` 成本低得多——数据残留在数据库层，回滚一下就没了；只有当测试**修改了 Bean 状态**（往单例缓存里塞数据、改了内存配置）导致容器污染时才使用 `@DirtiesContext`，且优先考虑在 `@AfterEach` 里手动还原状态代替销毁容器。用 `@DirtiesContext` 清数据残留是典型的杀鸡用牛刀。

---

# 6. 多线程与异步场景

**事务上下文与线程绑定**（`TransactionSynchronizationManager` 用的是 ThreadLocal），这带来两个测试盲区：

- 异步线程默认不会继承事务：测试方法里调 `@Async` 方法，异步线程**看不到**测试事务未提交的数据，它开自己的连接、自己的事务；
- 测试事务回滚时，异步线程已提交的数据**不会**被回滚——又一处数据残留来源。

```java
@Transactional
@SpringBootTest
class AsyncVisibilityTest {

    @Test
    void async_thread_cannot_see_uncommitted_data() throws Exception {
        orderRepository.save(new Order(null, "A-1"));       // 测试事务内，未提交

        CompletableFuture<Long> count = asyncCounter.countOrders();  // 另一线程、另一连接
        // 大概率返回 0：异步线程看不到未提交的 "A-1"
        assertThat(count.get(2, TimeUnit.SECONDS)).isZero();
    }
}
```

若需验证异步提交效果，应明确等待异步执行完成（Awaitility/latch），并且**放弃测试事务**（不加 `@Transactional`），让主流程真实提交，异步线程才能看到数据：

```java
@SpringBootTest                       // 无测试事务
class AsyncCommitIT {

    @Test
    void should_process_after_commit() {
        orderService.createAndPublish(order);              // 真实提交 + AFTER_COMMIT 发事件

        await().atMost(3, SECONDS)
               .untilAsserted(() ->
                   assertThat(projectionRepository.findByOrderId(order.id())).isPresent());
    }

    @AfterEach
    void cleanup() { jdbc.update("DELETE FROM orders"); } // 提交型测试自备清理
}
```

> **设计争议**：`@TransactionalEventListener(phase = AFTER_COMMIT)` 的测试到底该不该带测试事务？带事务 → AFTER_COMMIT 永远不触发（回滚没有 commit），监听器逻辑测不到；不带事务 → 要自己清理数据。**共识是这类"提交后行为"必须用非事务测试 + 清理策略来测**，带事务版本只能测到"事件已注册待提交"这一半。

---

# 7. 回滚的盲区：什么时候禁用测试事务

回滚模型的舒适区是有代价的。以下场景**测不到**，必须禁用测试事务（不加 `@Transactional`，或 `@Commit`）：

1. **真实提交路径**：提交时触发的约束、触发器、`AFTER_COMMIT` 监听器、乐观锁版本号落库；
2. **跨连接可见性**：另一线程/另一数据源能否读到数据（见第 6 节）;
3. **DDL 相关行为**：多数数据库 DDL 隐式提交，事务回滚包不住它；
4. **`REQUIRES_NEW` 的独立性**：外层回滚后内层是否还在（见 3.3）；
5. **批量 flush 时序**：Hibernate 在提交时机的 flush 行为与测试内手动 flush 不完全等价。

禁用事务后的多表清理策略，按可靠性排序：

| 策略 | 做法 | 适用 |
|------|------|------|
| 一次性容器库 | 每类/每套 Testcontainers 新实例 | 集成测试首选，零清理代码 |
| `@Sql` AFTER 清表 | `DELETE FROM ...` 按外键逆序 | 共享库、表数量可控 |
| `@BeforeEach` 重建 | `TRUNCATE ... CASCADE` | PG/MySQL 支持级联 truncate |
| `DatabaseCleaner` 工具类 | 遍历元数据删全表 | 表多、想一劳永逸 |

```java
@Component
class DatabaseCleaner {                       // 提交型测试套件的通用清理器

    private final JdbcTemplate jdbc;
    private final List<String> tables = List.of("order_item", "orders", "users"); // 外键逆序

    DatabaseCleaner(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    void clean() {
        tables.forEach(t -> jdbc.execute("DELETE FROM " + t));  // 注意顺序，先子表后父表
    }
}
```

**何时禁用测试事务的决策线**：默认带回滚（快、隔离好）；测试目标涉及"提交那一刻发生的事"或"跨连接可见性"时禁用，并立刻补上清理策略。两条规则冲突时（比如一个测试既要回滚的隔离又要验证提交），拆成两个测试。

# 8. 小结

| 主题 | 关键结论 |
|------|---------|
| 默认行为 | `@Transactional` 测试方法级事务 + 结束回滚；切片注解元注解自带事务 |
| 管理者 | `TransactionalTestExecutionListener` 开启/回滚，生产代码 REQUIRED 加入测试事务 |
| 提交开关 | `@Commit` = `@Rollback(false)`，用于触发器/AFTER_COMMIT 等提交后行为，需自备清理 |
| REQUIRES_NEW | 独立连接立即提交，测试回滚管不住它，也看不到测试事务未提交数据 |
| @Sql | 默认加入测试事务；提交型测试必须配 AFTER 清理 |
| 异步 | 事务 ThreadLocal 绑定，异步线程看不到未提交数据；测异步要放弃测试事务 + Awaitility |
| 禁用事务 | 验证提交路径/跨连接可见性/DDL 时禁用，配一次性容器库或清表策略 |

下一篇进入安全测试：`spring-security-test` 如何在 MockMvc 与方法级安全里构造认证主体。
