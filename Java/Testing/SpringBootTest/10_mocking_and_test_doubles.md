# 1. 测试替身（Test Double）分类

测试替身是**用可控的假对象替换真实依赖**的统称，这个概念由 Gerard Meszaros 在《xUnit Test Patterns》中系统化，后来被 Mockito 等框架落地。替身的价值在于切断测试与外部世界（数据库、HTTP 服务、消息队列、时钟）的耦合，让测试快且确定；代价是每引入一个替身，就有一块真实行为不再被验证。理解五种替身的分工，是决定"这个测试该不该 Mock、Mock 到什么程度"的前提。

## 1.1 常见类型

五种经典替身按"参与测试的深度"排列：

| 类型 | 目的 | 特点 | 验证方式 | 典型例子 |
|------|------|------|----------|----------|
| Dummy | 占位 | 只为了凑齐构造参数，从不被调用 | 不验证 | `new OrderService(null)` 里的 null 协作对象 |
| Stub | 固定返回 | 预置"问到什么答什么"，关注**结果** | 状态验证 | `when(repo.findById(1L)).thenReturn(user)` |
| Spy | 代理真实对象 | 默认走真实逻辑，可选择性拦截 | 状态 + 交互 | `@MockitoSpyBean` 包装真实价格计算 |
| Mock | 关注交互 | 预置**期望的调用**，测试结束时核对 | 行为验证 | `verify(mailer).send(any())` |
| Fake | 简化实现 | 有真实可运行的轻量逻辑，但不能上生产 | 状态验证 | 内存版 Repository、内存版文件系统 |

**Stub 与 Mock 的本质区别是验证方向**：Stub 支撑"被测对象算出了正确结果"（状态验证），Mock 支撑"被测对象以正确方式与依赖交互"（行为验证）。实践中 Mockito 的一个 mock 对象同时具备两种能力——既能打桩又能 verify——所以口语里"Mock 一切"，但设计测试时应该先想清楚：我到底在断言返回值，还是在断言调用？断言调用的测试对实现重构极其敏感（改个内部调用顺序就红），应该只用在"调用本身就是契约"的场合，比如发信、扣款、发事件。

> **设计争议**：行为验证（verify）该用多重？经典派（Mockito 作者 Szczepan Faber 早期）主张交互测试是单元测试的正统；后来连他本人都转向"优先状态验证"，因为过度 verify 会把实现细节焊死在测试里。行业共识大致是：**对副作用（发消息、写审计日志）用 verify，对纯计算结果用断言**。

## 1.2 何时用 Fake

Fake 是五种替身里唯一"有真实逻辑"的，适合依赖行为复杂、Stub 打桩打不过来的场景。判断标准：**如果被测代码会对依赖做多次不同输入的读写，且结果依赖之前写入的状态，Stub 就会退化成"把整段业务逻辑用 when/thenReturn 重抄一遍"——这时应该写 Fake**。

```java
// 内存版 Fake：实现真实接口，行为自洽，可被任意读写
public class InMemoryCartRepository implements CartRepository {

    private final Map<Long, Cart> store = new ConcurrentHashMap<>();
    private final AtomicLong seq = new AtomicLong();

    @Override
    public Cart save(Cart cart) {
        Cart toSave = cart.id() == null
                ? new Cart(seq.incrementAndGet(), cart.items())  // 模拟自增主键
                : cart;
        store.put(toSave.id(), toSave);
        return toSave;
    }

    @Override
    public Optional<Cart> findById(Long id) {
        return Optional.ofNullable(store.get(id));              // 语义与真实 JPA 一致
    }
}
```

Fake 的成本是要自己维护，收益是测试可以像对着真库一样自由读写。常见 Fake：内存 Map 版 Repository、`MapStruct` 无关的假对象映射、`Clock.fixed(...)` 假时钟、Spring 生态里 `spring-boot` 测试自带的 H2 内嵌库本质上也是一种"数据库 Fake"。Google 的测试博客把这套选择总结为：**能 Fake 则 Fake，不能 Fake 才 Mock，Dummy 只在编译期需要**。

---

# 2. Mockito 核心用法

Mockito 5.x（Boot 3.x 默认）基于 inline mock maker，能力比 4.x 强得多。这一节按"打桩 → 验证 → 捕获 → 严格度"四个动作过一遍高频用法。

## 2.1 Stubbing（打桩）

打桩就是告诉替身"被这样调用时返回什么"。两种等价风格：

```java
// 经典风格：when / thenReturn
when(userRepo.findById(1L)).thenReturn(Optional.of(user));

// BDD 风格：given / willReturn（Spring 项目更常用，读起来像场景描述）
given(userRepo.findById(1L)).willReturn(Optional.of(user));

// 按调用次序返回不同值：第一次抛异常，第二次成功（测重试逻辑）
given(paymentClient.charge(any())).willThrow(new TimeoutException()).willReturn(OK);

// void 方法：doThrow / doNothing（不能用 when(...) 语法）
doThrow(new IllegalStateException()).when(auditService).close();
doNothing().when(smsSender).send(any());
```

> **注意**：`when(...)` 语法对 Spy 有坑——`when(spy.method())` 会**先真实执行一次** method，再打桩；对 Spy 一律用 `doReturn(...).when(spy).method()`。这是 Mockito 文档里明确写的例外，面试也常考。

## 2.2 验证交互

验证回答"依赖被按预期调用了吗"：

```java
verify(userRepo, times(1)).save(any(User.class));   // 恰好一次（verify 默认就是 times(1)）
verify(mailSender, never()).send(any());             // 从未调用
verify(userRepo, atLeastOnce()).findById(1L);        // 至少一次
verifyNoMoreInteractions(userRepo);                  // 之后不许再有其他调用（慎用，太脆）

// 顺序验证：先扣款、后发货是业务契约
InOrder inOrder = inOrder(paymentService, deliveryService);
inOrder.verify(paymentService).charge(any());
inOrder.verify(deliveryService).ship(any());
```

## 2.3 参数捕获

当"调用发生了"不够、还要断言"调用时传了什么"，用 `ArgumentCaptor`：

```java
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(userRepo).save(captor.capture());
assertThat(captor.getValue().getName()).isEqualTo("tom");   // AssertJ 继续深挖字段

// Mockito 5 的 Lambda 风格，少一个变量声明
verify(userRepo).save(withArg(u -> assertThat(u.getName()).isEqualTo("tom")));
```

参数捕获的合理使用是断言**被测代码组装出来的对象**（比如把 DTO 转成实体再 save），它比 `eq(new User(...))` 依赖 equals 实现要稳。但捕获后再逐字段断言七八个，往往说明该给实体补 `equals` 或改用 AssertJ 的 `usingRecursiveComparison()`。

## 2.4 严格度与宽松模式

JUnit 5 下 `@ExtendWith(MockitoExtension.class)` 默认开启 **STRICT_STUBS**，它能抓两类错误：

```java
@ExtendWith(MockitoExtension.class)   // 严格桩：多余的打桩直接让测试失败
class PureUnitTest {

    @Mock
    private UserRepo userRepo;

    @Test
    void demo() {
        given(userRepo.findById(2L)).willReturn(Optional.of(user)); // 打桩了
        service.doSomethingWith(1L);                                 // 但实际只查了 1L
        // 测试结束 → UnnecessaryStubbingException：这条桩从未被命中
    }
}
```

| 严格度 | 行为 | 适用 |
|--------|------|------|
| `STRICT_STUBS`（默认） | 未命中的桩报错；桩与实参不匹配时报 `PotentialStubbingProblem` | 新代码一律用这个 |
| `LENIENT` | 多余桩静默忽略 | 共享 setup 里确实有分支用不到的桩 |
| `WARN` | 只打警告日志 | 老项目迁移过渡期 |

个别桩确实需要在多分支共享时豁免，用 `lenient().when(...)` 单点放开，而不是把整个类降级为 LENIENT——**严格度是防止"桩与实现漂移"的护栏，拆护栏之前先确认不是代码写错了**。

---

# 3. Spring Boot 集成

纯 Mockito 的 `@Mock` 活在测试类字段里，与容器无关；一旦被测对象是从 Spring 容器里 `@Autowired` 出来的，它依赖的 Bean 也是容器装配的——想替换这些依赖，就需要让替身"进容器"。

## 3.1 `@MockBean`

`@MockBean`（Boot 1.4 - 3.3，现已废弃）把 mock 注册进 Spring 容器，按类型替换同类型真实 Bean：

```java
@SpringBootTest
class OrderServiceTest {

    @MockBean                                   // Boot ≤3.3 写法；3.4+ 换 @MockitoBean
    private PaymentGateway paymentGateway;      // 容器里的真实网关被替换为 Mock

    @Autowired
    private OrderService orderService;          // 它注入到的 gateway 就是上面这个 Mock

    @Test
    void should_create_order_when_payment_ok() {
        given(paymentGateway.charge(any())).willReturn(PaymentResult.ok());
        assertThat(orderService.create(order).isPaid()).isTrue();
    }
}
```

切片测试（`@WebMvcTest` 等）里 `@MockBean`/`@MockitoBean` 几乎是必需品：切片不加载 Service 层 Bean，不 Mock 掉，控制器注入就会失败。

## 3.2 `@SpyBean`

Spy 包装真实 Bean：默认执行真实逻辑，只对个别方法打桩或验证。适合"真实行为大部分正确、只想干扰一小块"的场景：

```java
@SpyBean
private PriceService priceService;   // 真实的价格计算仍在跑

@Test
void should_use_discount_but_real_tax() {
    doReturn(Money.of(100)).when(priceService).basePrice(any()); // 只桩掉基础价
    // 税费计算走真实逻辑——这正是我们想顺带验证的部分
    assertThat(orderService.checkout(cart).total()).isEqualTo(expected);
    verify(priceService).basePrice(any());
}
```

> **注意**：被 Spy 的 Bean 如果被其他 Bean 持有引用，替换后旧引用仍指向原对象吗？不会——`@SpyBean` 在 Bean 定义层面完成包装，容器内所有注入点拿到的都是 Spy。但对**容器外**手动 new 出来的对象无效，这点和 `@MockBean` 一致。

## 3.3 `@Mock` 与 `@MockBean` 区别

| 注解 | 归属 | 作用范围 | 是否进入容器 | 上下文缓存影响 |
|------|------|----------|--------------|----------------|
| `@Mock` | Mockito（配 `MockitoExtension`） | 测试类字段 | 否 | 无（不启动容器） |
| `@MockBean` | Spring Boot Test（≤3.3，已废弃） | 容器内 Bean | 是 | 不同 Mock 组合 → 不同上下文 |
| `@MockitoBean` | Spring Framework 6.2+ | 容器内 Bean 或测试字段 | 是 | 同上 |

选型规则很直接：**测试不需要容器就用 `@Mock`；需要容器、要替换容器里的 Bean 才用 `@MockitoBean`**。前者毫秒级，后者至少一次上下文加载。

## 3.4 `@MockitoBean` 与 `@MockitoSpyBean`（Boot 3.4+）

Boot 3.4 把 Bean 替换能力从 Boot 自家下沉到了 Spring Framework 6.2，新注解是 `@MockitoBean` / `@MockitoSpyBean`（包 `org.springframework.test.context.bean.override.mockito`），旧的 `@MockBean` / `@SpyBean` 标记 `@Deprecated`，并计划在 **Boot 4.0 移除**。新注解不只是换名：

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;

@SpringBootTest
class OrderServiceTest {

    @MockitoBean
    private PaymentGateway paymentGateway;        // 按类型替换；容器没有则新增一个 Mock

    @MockitoBean(name = "primaryDataSource")
    private DataSource auditDataSource;           // 3.4+ 新能力：按 Bean 名字精确替换

    @MockitoSpyBean
    private AuditLogger auditLogger;              // Spy 包装真实 Bean

    @MockitoBean(reset = MockReset.NONE)
    private ConfigHolder configHolder;            // 可控制重置时机（默认每个方法后重置）
}
```

与旧注解的行为差异要点：

- **按名字/限定符替换**：`@MockitoBean(name = "...")` 可以在同类型多 Bean 时精确指定，旧 `@MockBean` 靠 `name`/`@Qualifier` 组合，语义模糊。
- **可作用于测试类自身字段注入**，也可以声明在 `@TestConfiguration` 的方法上。
- **容器无匹配 Bean 时直接新增**一个 Mock 定义，行为与旧注解一致但实现更规范。
- 迁移期两套注解**不可对同一 Bean 混用**，会抛重复覆盖异常。

## 3.5 Mock 替换容器 Bean 的原理

`@MockBean` 时代由 Boot 的 `MockitoPostProcessor` 完成，它是一个 **`BeanFactoryPostProcessor` + `BeanDefinitionRegistryPostProcessor`**；Spring Framework 6.2 的 `@MockitoBean` 换成 `BeanOverrideRegistryPostProcessor` + `BeanOverrideBeanFactoryPostProcessor`，机制同源。理解这条链，就能解释"为什么 Mock 会影响上下文缓存""为什么每个测试方法后 Mock 自动清零"：

```text
┌─────────────────────────────────────────────────────────────┐
│           @MockitoBean 替换容器 Bean 的生命周期               │
│                                                             │
│  容器 refresh()                                              │
│    │                                                        │
│    ▼                                                        │
│  BeanFactoryPostProcessor 介入                               │
│    ├─ 找到被 Mock 的 BeanDefinition → 替换为 Mock 工厂定义     │
│    └─ 找不到 → 注册一个新的 Mock BeanDefinition               │
│    │                                                        │
│    ▼                                                        │
│  单例实例化：注入点拿到的已是 Mockito Mock 对象                 │
│    │                                                        │
│    ▼                                                        │
│  每个测试方法 before/after：                                  │
│    ResetMocksTestExecutionListener（Boot）/                   │
│    MockitoResetTestExecutionListener（Framework 6.2）        │
│    → 按 MockReset.BEFORE / AFTER 清空打桩与调用记录            │
└─────────────────────────────────────────────────────────────┘
```

两个重要推论：

1. **Mock 集合参与上下文缓存 Key**。替换发生在 BeanDefinition 层，"哪些类被 Mock"直接决定容器内容，所以 `MergedContextConfiguration` 把 Bean 覆盖配置计入指纹——两个测试类哪怕其他配置全同，`@MockitoBean` 字段不同就各起一个容器。
2. **重置而非重建**。每个测试方法后只 reset Mock（清 stub、清调用记录），容器不动。这是缓存能命中的前提，也解释了为什么你在 `@BeforeEach` 之前看到的 Mock 永远是干净的。

---

# 4. 常见问题与陷阱

## 4.1 过度 Mock

过度使用 `@MockBean` 会导致：

- **真实装配无法验证**：Bean 之间怎么接线、`@Conditional` 是否生效、事务代理是否织入，全被 Mock 挡在测试之外；
- **集成问题被隐藏**：真实依赖的超时、序列化、并发行为与 Stub 假设不符时，测试全绿、上线即炸；
- **上下文数量爆炸**：每个 Mock 组合一个容器，CI 时间线性上涨（见 3.5）。

判断一个 Mock 是否过度的土办法：**把测试里所有 when/thenReturn 连起来读，如果它已经把"正确答案"描述了一遍，那这个测试只是在验证 Mock 框架本身**。治理方向是分层：切片内 Mock 边界，跨切片用 Testcontainers 真实依赖，发布前全量 `@SpringBootTest` 冒烟。

## 4.2 Final 类与静态方法

Mockito 5.x 默认启用 **inline mock maker**（靠 Java Instrumentation 改字节码），final 类、final 方法已可直接 Mock，不再需要 4.x 时代的 `mockito-inline` 依赖。静态方法也能 Mock，但要显式开启作用域：

```java
try (MockedStatic<IdGenerator> mocked = mockStatic(IdGenerator.class)) {
    mocked.when(IdGenerator::next).thenReturn("fixed-id");   // 仅在 try 块内生效
    assertThat(service.create().id()).isEqualTo("fixed-id");
}   // 出块自动还原——静态 Mock 是线程绑定 + 作用域绑定的
```

> **设计争议**：静态 Mock 是"设计坏味道的止痛药"。需要 `mockStatic` 往往说明生产代码直接依赖静态工具（时间、UUID、全局单例），更好的解法是注入 `Clock`、注入 ID 生成器接口。测试可以偶尔用静态 Mock 救急，但把它当常规手段会让生产代码的可测性永远得不到改善。

## 4.3 深度 Stubbing

```java
when(order.getCustomer().getAddress().getCity()).thenReturn("SZ");  // RETURNS_DEEP_STUBS 风格
```

过深的调用链（迪米特法则违背）通常是设计问题：Mock 框架替你把链上每一环都造假，掩盖了"被测对象伸手太长"的事实。优先考虑把链上取值收敛成一个参数传入，或者给整条链写一个 Fake 对象树。

---

# 5. 实践建议

把前面各节的取舍收敛成可执行的规则：

- **单元测试优先使用 `@Mock`**：不进容器、毫秒级执行，配合 `MockitoExtension` 的严格桩。
- **切片测试使用 `@MockitoBean` 隔离切片外依赖**：`@WebMvcTest` Mock 掉 Service，`@DataJpaTest` Mock 掉外部客户端。
- **集成测试避免过多替身**：数据库、消息队列用 Testcontainers 真实实例，只 Mock 真正不可控的第三方 HTTP。
- **状态依赖复杂的协作对象写 Fake**，不要连环打桩。
- **verify 只留给副作用型契约**（发信、扣款、发事件），纯计算一律断言返回值。
- Boot 3.4+ 项目**统一迁移到 `@MockitoBean` / `@MockitoSpyBean`**，避免与废弃注解混用。

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 五种替身 | Dummy 占位、Stub 管返回、Mock 管交互、Spy 半真半假、Fake 轻量真实实现 |
| Stub vs Mock | 状态验证 vs 行为验证；verify 只用于副作用契约 |
| 严格度 | `MockitoExtension` 默认 STRICT_STUBS，多余桩直接失败；单点豁免用 `lenient()` |
| 注解选型 | 无容器用 `@Mock`；容器内替换用 `@MockitoBean`（3.4+）/`@MockBean`（≤3.3，废弃） |
| 替换原理 | BeanFactoryPostProcessor 在 BeanDefinition 层替换；Mock 集合计入上下文缓存 Key |
| 方法后重置 | `MockReset.AFTER` 默认清桩清记录，容器保持可缓存 |
| 反模式 | 过度 Mock 隐藏装配问题、静态 Mock 掩盖设计缺陷、深度打桩纵容迪米特违背 |
