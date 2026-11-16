# 1. 最佳实践

前面十九篇讲的都是"怎么用"，这一篇讲"怎么用好"。测试代码同样是代码，同样会腐化——**一套失去信任的测试比没有测试更糟糕，因为它给了团队虚假的安全感，还持续收着维护税**。本篇收束整个系列：先立规范，再排陷阱，最后用面试题串起全系列的知识点。

## 1.1 测试命名规范

测试名是失败报告里的第一行信息，**好的名字让 CI 红灯自带诊断**。推荐的模式是 `should_xxx_when_yyy`——"在什么条件下，应该发生什么行为"，失败时不用打开代码就知道哪条业务规则被破坏了：

```java
class OrderServiceTest {

    @Test
    void should_reject_order_when_inventory_insufficient() { }   // 条件与结果都在名字里

    @Test
    void should_apply_ten_percent_discount_when_user_is_vip() { }

    @Test
    void should_throw_IllegalArgumentException_when_amount_is_negative() { }
}
```

配合 `@DisplayName` 可以进一步支持中文与空格描述（IDE 和报告里显示 DisplayName，方法名保留英文规范）：

```java
@Test
@DisplayName("库存不足时下单应被拒绝，且不扣减库存")
void should_reject_order_when_inventory_insufficient() { }
```

> **行业实践**：命名风格（下划线分隔 / 驼峰 / 反引号——Groovy 与 Kotlin 里流行）团队内统一即可，真正的共识是三点：名字里要有**行为**、有**条件**，以及不出现 `test1`/`testSave` 这种无信息命名。JUnit 5 移除了 JUnit 4 "方法必须 public"的限制，包级私有让测试类更干净。

## 1.2 AAA 结构

每个测试方法按 **Arrange（准备）- Act（执行）- Assert（断言）** 三段组织，段间空行分隔。AAA 的价值不是仪式感，而是**强制每个测试只有一个行为焦点**：出现两段 Act 通常意味着该拆成两个测试了。

```java
@Test
void should_calculate_total_with_discount() {
    // Arrange：构造输入与依赖状态，只准备本测试用到的
    Cart cart = new Cart().addItem(item("sku-1", 60)).addItem(item("sku-2", 60));
    when(couponService.discountFor("VIP10")).thenReturn(new Discount(0.10));

    // Act：唯一一次被测调用
    Money total = pricingService.calculateTotal(cart, "VIP10");

    // Assert：验证结果，必要时验证关键副作用
    assertThat(total).isEqualTo(Money.of("108.00"));
    verify(couponService).discountFor("VIP10");
}
```

Arrange 段膨胀（几十行造数据）是可读性杀手，治理手段：ObjectMother / Test Data Builder 模式把造数逻辑抽成可复用的构建器，测试体只声明与本用例相关的那几个字段。

## 1.3 测试独立性

**每个测试必须能单独运行、以任意顺序运行、并行运行而结果不变**。独立性破坏的典型来源与对策：

- **执行顺序依赖**：测试 B 用到测试 A 写入的数据。对策：每个测试自己 Arrange 自己的数据；确需顺序表达的场景用 `@TestMethodOrder` + `@Order`，但这应是极罕见的例外（如演示生命周期），而不是数据依赖的遮羞布。
- **共享可变静态状态**：static 字段、单例缓存跨测试残留。对策：`@AfterEach` 还原，或把状态设计成可注入/可重置。
- **外部资源争用**：并行测试写同一张表、抢同一个端口。对策：并行套件里的有状态测试类标 `@Isolated`，或用 `@ResourceLock` 声明互斥（见第 13 篇）。

验证独立性的手段很朴素：**乱序跑一遍**（`junit.jupiter.testclass.order.default` + `ClassOrderer.Random`，或 CI 里定期开随机顺序），红了就是有问题。

## 1.4 测试即文档

测试套件是系统行为的**可执行规格**：新人读 `should_reject_order_when_inventory_insufficient` 比读需求文档更快且不会过时（过时就红灯）。由此推论：为覆盖率硬凑的、没有业务语义的测试（getter/setter 全覆盖）不仅无价值，还稀释了套件的信噪比。**每个测试都应该回答一个业务问题**。

# 2. 常见陷阱

## 2.1 Flaky Test：根因与治理

Flaky Test（时红时绿、重跑就过）是测试体系的头号癌症——它的危害不在单次失败，而在**训练团队忽略红灯**。常见根因按出现频率排序：

| 根因 | 典型症状 | 治理手段 |
|------|---------|---------|
| 时间依赖 | 月末/跨年/闰秒时段失败 | 注入 `Clock`，测试用固定时钟（`Clock.fixed`） |
| 异步竞态 | 偶发超时或读到中间状态 | 用 Awaitility 轮询断言，禁用 `Thread.sleep` |
| 顺序/共享状态 | 单跑绿、批跑红 | 还原静态状态；`@Isolated`/`@ResourceLock` |
| 随机数 | 概率性越界 | 固定 seed，或 Mock `Random`/用确定性数据 |
| 外部依赖抖动 | CI 特有失败 | Testcontainers 替代真实远端；wait strategy 判就绪 |
| 资源泄漏 | 越跑越慢最后超时 | try-with-resources；容器/连接严格关闭 |

```java
// 反模式：sleep 赌异步完成——短了偶发失败，长了拖慢套件
Thread.sleep(2000);
assertThat(queue.size()).isEqualTo(1);

// 正确：Awaitility 轮询到条件满足，最多等 5 秒，通常几百毫秒即返回
await().atMost(Duration.ofSeconds(5))
       .untilAsserted(() -> assertThat(queue.size()).isEqualTo(1));
```

治理流程上的共识：**Flaky 测试一旦确认，要么 24 小时内修，要么立即隔离（`@Disabled("flaky: 跟踪 issue #123")` + 建单），绝不允许"重跑一下就好"**。CI 自动重试（Surefire `rerunFailingTestsCount`、Gradle Test Retry）只作为过渡，且必须把 flaky 计数显式上报、定期清零，否则重试机制会把问题永久掩埋（详见第 16 篇）。

## 2.2 过度 Mock 反模式

第 18 篇已展开机制，这里给识别清单。以下症状出现两条以上，就是过度 Mock：

- 测试体里 `when(...)` 比业务断言还多；
- 重构（不改行为）导致大批测试红；
- `verify` 精确锁定调用次数与顺序，连内部协作者的私有交互都要验证；
- Mock 了值对象、工具类、纯函数——这些根本没有隔离的必要；
- 测试通过但线上仍出 bug：Mock 把真实依赖的行为假设（null 处理、异常类型、超时语义）全部绕开了。

修正方向：**Mock 只放在信任边界上**（进程外、有副作用、不可控），边界内用真实现；断言优先面向业务结果，交互验证只用于副作用与"不该发生"的场景。

## 2.3 断言粒度

两个极端都常见。**断言过粗**：一个测试十几行 Act 之后只有一条 `assertNotNull(result)`——通过了也不知道对在哪。**断言过细**：对整个响应对象做全字段 `isEqualTo`，任何无关字段变动都引发红灯，测试变成"变更探测器"而非"行为验证器"。

合理粒度：**断言与本测试的业务问题直接相关的字段，用软断言聚合多个检查点**，失败时一次看全所有差异而不是修一个跑一遍：

```java
// AssertJ 软断言：收集全部失败再一起报告
SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(order.getStatus()).isEqualTo(Status.PAID);
    softly.assertThat(order.getTotal()).isEqualTo(Money.of("108.00"));
    softly.assertThat(order.getPaidAt()).isNotNull();
});

// 或 JUnit 5.10+ 原生的聚合断言
assertAll("支付后的订单状态",
        () -> assertEquals(Status.PAID, order.getStatus()),
        () -> assertEquals(Money.of("108.00"), order.getTotal()),
        () -> assertNotNull(order.getPaidAt()));
```

## 2.4 覆盖率解读

覆盖率是**阴性预测值高、阳性预测值低**的指标：覆盖率低的地方大概率没测到（可信），覆盖率高不代表测得好（不可信）——无断言的空跑测试同样贡献 100% 行覆盖。正确姿势：

- 看**增量覆盖率**（新代码的覆盖）而非全量存量数字；
- 看**分支覆盖**而非行覆盖，条件逻辑的漏洞多半藏在未走的分支里；
- 把未覆盖区域当**风险清单**人工过一遍：是"不值得测"（DTO、常量类）还是"忘了测"（金额计算的异常分支）；
- 警惕为达标而写的凑数测试，它们抬高数字、拉低信噪比。

> **行业实践**：门禁设在增量覆盖（如新代码 ≥ 80%，SonarQube "Coverage on New Code"），全量覆盖率只作趋势观察。把全量覆盖率当 KPI 考核的团队，最终收获的一定是古德哈特定律——指标变成目标后就不再是好指标。

## 2.5 大而慢的测试

测试套件超过 10 分钟后，开发者的"提交前跑一遍"习惯就会消失，套件名存实亡。治理组合拳（细节分散在前面各篇）：纯逻辑测试不进 Spring 容器（第 17 篇）；能用切片不用全量；集成测试打 `@Tag("integration")` 与单测分级执行（第 13 篇）；Testcontainers 单例复用（第 19 篇）；开 JUnit 5 并行（第 13 篇）。**目标是单测套件分钟级以内、全量套件十分钟级以内。**

# 3. 面试高频问题

## 3.1 JUnit 4 与 JUnit 5 的区别是什么？

架构上 JUnit 4 是单体 jar，由 Runner 驱动执行（`@RunWith`），扩展只能靠 Runner 继承与 Rule 组合；JUnit 5 拆成 Platform（基础设施与 Launcher API）+ Jupiter（编程模型与引擎）+ Vintage（兼容运行 JUnit 4 测试）三模块，扩展模型换成组合式的 Extension（`@ExtendWith`，可同时挂多个）。编程模型上：JUnit 5 测试类与方法不再要求 public、断言参数顺序改为 `assertEquals(expected, actual, message)` 且 message 挪到末位并支持 Supplier 惰性求值、新增 `assertAll`/`assertThrows`/`assertTimeout`、`@Before/@After/@BeforeClass/@AfterClass` 更名为 `@BeforeEach/@AfterEach/@BeforeAll/@AfterAll`、`@Ignore` 换成 `@Disabled`、Category 换成 `@Tag`、`@RunWith(Parameterized)` 换成原生的 `@ParameterizedTest`、并新增 `@Nested`、`@DisplayName`、动态测试与原生并行执行支持。

## 3.2 @Before 与 @BeforeEach 有什么区别？

它们语义等价——都在每个测试方法执行前运行——区别属于两代框架：`@Before` 是 JUnit 4 注解（org.junit 包），要求方法 public，配对的类级注解是 `@BeforeClass`（要求 static）；`@BeforeEach` 是 JUnit 5 注解（org.junit.jupiter.api 包），方法可以是包级私有甚至 protected（供 `@Nested` 内部类继承覆写），配对的 `@BeforeAll` 默认仍要求 static，但配合 `@TestInstance(Lifecycle.PER_CLASS)` 后可以是实例方法，从而能访问实例状态。混用时注意执行顺序：若 Vintage 引擎与 Jupiter 并存，两套注解各管各的测试，不会交叉生效。

## 3.3 assertThrows 怎么用？与 JUnit 4 的 expected 属性相比好在哪？

`assertThrows(异常类型.class, 可执行体)` 断言代码块抛出指定类型（含子类）的异常，并返回异常对象供进一步断言：`OrderException ex = assertThrows(OrderException.class, () -> service.submit(emptyOrder)); assertEquals("EMPTY_ORDER", ex.getCode()); assertTrue(ex.getMessage().contains("订单为空"));`。JUnit 4 的 `@Test(expected = X.class)` 有三个缺陷：粒度是整个方法（异常可能从 Arrange 段意外抛出也算通过）、拿不到异常对象无法断言 message 与错误码、无法验证"没抛异常"以外的组合场景。另外 JUnit 5 的 `assertThrows` 要求异常精确匹配声明类型或子类，配合 `assertDoesNotThrow` 还能显式断言不抛异常。

## 3.4 如何在 JUnit 5 中开启并行执行？

两步：在 `src/test/resources/junit-platform.properties` 里设 `junit.jupiter.execution.parallel.enabled=true`，并选择并行策略——`junit.jupiter.execution.parallel.mode.default=concurrent`（方法级并行）或 `junit.jupiter.execution.parallel.mode.classes.default=concurrent`（类级并行，更常用也更安全），两者可组合；并发度用 `junit.jupiter.execution.parallel.config.strategy=fixed|dynamic` 控制，dynamic 按 CPU 核数乘因子。前提条件是测试相互独立：共享外部资源的类要标 `@Isolated` 强制串行，或用 `@ResourceLock("DB")` 声明资源互斥（同一资源的 READ 锁可并行、WRITE 锁互斥）。Spring 测试与并行兼容，但注意上下文缓存大小与共享容器的数据隔离。

## 3.5 JUnit 4 的 Rule 与 JUnit 5 的 Extension 有什么区别？

Rule（`@Rule`/`@ClassRule`）是 JUnit 4 的扩展机制，本质是包装测试执行的 Statement 拦截器（临时文件夹、超时、ExpectedException 都曾是 Rule），一个 Rule 只管"包裹方法执行"这一件事，且能力面窄（拿不到测试实例、无法介入实例创建）。Extension 是 JUnit 5 的回调式扩展模型，接口按生命周期切分得很细：BeforeAllCallback、BeforeEachCallback、PostProcessTestInstanceCallback（能改测试实例字段，Spring 注入、Mockito 注解都在这实现）、BeforeTestExecutionCallback、AfterEachCallback、ParameterResolver（还能给测试方法注入参数）、TestTemplateInvocationContextProvider（支撑 @RepeatedTest/@ParameterizedTest）、ExecutionCondition（动态决定跳过）、InvocationInterceptor（环绕执行）等，一个类可实现多个接口、一个测试可挂多个 Extension（`@ExtendWith` 可组合，还支持自动注册）。迁移上官方提供 `junit-jupiter-migrationsupport` 模块的 `@EnableRuleMigrationSupport` 让旧 Rule 过渡期可用。

## 3.6 assumeTrue 与 assertTrue 的区别是什么？

`assertTrue` 失败会抛 AssertionFailedError，测试记为**失败（failed）**，红灯阻断构建；`assumeTrue` 失败抛 TestAbortedException，测试记为**跳过/中止（aborted/skipped）**，不算失败——语义是"前提条件不满足，本测试不适用，主动弃权"。典型用法是把测试与运行环境解耦：`assumeTrue(DockerClientFactory.instance().isDockerAvailable(), "无 Docker 环境，跳过集成测试")`、`@EnabledOnOs(OS.LINUX)` 这类条件注解本质也是 assumption。使用纪律：assumption 应该表达**环境性前提**（OS、Docker、网络、数据库版本），而不是掩盖业务断言——把 assertTrue 换成 assumeTrue 来"消灭"红灯是自欺欺人，报告里 skipped 数量激增就是信号。

## 3.7 单元测试与集成测试的边界怎么划分？

单元测试验证最小逻辑单元（类/方法）在隔离状态下的行为：依赖全部 Mock、不启动容器、毫秒级、开发者每次提交都跑，归属测试金字塔底座；集成测试验证多个组件与真实外部依赖（数据库、消息队列、HTTP 服务）协作的正确性：用 Spring 容器（全量或切片）+ Testcontainers 真依赖、秒级到分钟级、按流水线阶段跑。划分口径不在"测试了什么类"而在"信任边界放在哪"：边界内是真实现、边界外是替身。工程上常用命名（`*Test` vs `*IT`）+ 构建插件（Surefire vs Failsafe）+ `@Tag` 把两层物理分开，避免慢测试拖垮快反馈。

## 3.8 @SpringBootTest、@WebMvcTest、@DataJpaTest 怎么选？

按被测层选：验证 Service/组件在完整装配下的端到端行为用 `@SpringBootTest`（可用 `webEnvironment = RANDOM_PORT` 起真端口配 TestRestTemplate）；验证 Controller 的 HTTP 契约（路由、参数绑定、状态码、JSON 结构、异常处理器）用 `@WebMvcTest(XxxController.class)`，它只装配 Web 层，Service 用 `@MockitoBean` 替换，注入 MockMvc；验证 JPA 实体映射与 Repository 查询用 `@DataJpaTest`，默认内嵌内存库（对 SQL 方言敏感时应 `@AutoConfigureTestDatabase(replace = NONE)` 换真库/Testcontainers）。优先级顺序是"能用小的就不用大的"：切片测试启动快、失败定位准，`@SpringBootTest` 只留给真正需要全量装配的少数场景。

## 3.9 Mockito 的 @Mock、@Spy、@InjectMocks 分别是什么？

`@Mock` 创建完全虚假的对象：所有方法默认返回 null/0/空集合，行为全靠 stubbing 编程，用于替换信任边界外的依赖；`@Spy` 包装真实对象：未打桩的方法走真实现，打桩的走假行为，适合遗留代码的部分替换，注意打桩语法必须用 `doReturn().when(spy).method()`（`when(spy.method())` 会先真实执行一次）；`@InjectMocks` 创建被测真实对象并把 `@Mock`/`@Spy` 注入进去，注入策略依次尝试构造器、setter、字段，同类型多 Mock 按字段名匹配。三者都由 `MockitoExtension`（或 `openMocks`）初始化，`@InjectMocks` 不会递归处理被测对象内部 new 出来的依赖。

## 3.10 什么是 Testcontainers 的单例容器模式？为什么需要它？

单例容器模式指把容器声明为跨测试类共享的 static 实例，在静态初始化块中 `start()`、不显式 `stop()`（靠 Ryuk 在 JVM 退出时回收），所有集成测试通过基类或工具类引用同一容器。需要它的原因：默认的 `@Container static` 只是类级复用，N 个测试类会启停 N 次容器，而一次 Postgres/Kafka 启动要 10-20 秒，套件规模一大启动开销成为大头；单例化后容器整个 JVM 只启一次，配合 Spring 上下文缓存，测试的边际成本趋近于纯执行时间。代价是容器状态跨测试共享，每个测试必须自己保证数据隔离（事务回滚、truncate、独立数据集），不能假设环境是干净的。

## 3.11 遇到 Flaky Test 应该怎么排查和处理？

先复现定位根因：单跑与批跑对比（批跑红→顺序/共享状态问题）、开随机顺序与并行放大竞态、看失败是否聚集在特定时间/特定 CI 机器（→时间依赖/资源抖动）、检查失败堆栈是否总在异步边界（→竞态）。常见根因依次是：`Thread.sleep` 赌异步（换 Awaitility）、直接读系统时钟（注入 `Clock.fixed`）、静态状态跨测试泄漏（`@AfterEach` 还原或 `@Isolated`）、真实外部依赖抖动（换 Testcontainers/Mock）、容器就绪前访问（补 wait strategy）。处理纪律：确认 flaky 后要么限期修复，要么 `@Disabled` 加 issue 隔离，禁止靠 CI 自动重试长期掩盖；重试机制若开，必须把 flaky 计数上报并定期清零。

## 3.12 如何评价一个测试套件的质量？

四个维度：**速度**（单测套件是否分钟级内、开发者是否愿意每次提交都跑）、**稳定性**（flaky 率是否接近零、红灯是否都被当回事）、**可读性**（测试名是否自带业务语义、AAA 结构是否清晰、失败信息能否直接定位问题）、**有效性**（变异测试视角：故意改坏生产代码测试是否会红；覆盖率只作风险信号不作目标）。反指标同样重要：为覆盖率凑数的空断言测试、verify 实现细节的"复读机测试"、成片的 `@DirtiesContext` 与 `@Disabled`、跑一次要半小时的全量套件——这些都在提示套件正在失去信任价值，需要按第 2 节的陷阱清单逐项治理。

# 4. 小结

| 主题 | 一句话结论 |
|------|-----------|
| 命名 | `should_xxx_when_yyy`：名字里有行为、有条件，失败报告自带诊断 |
| 结构 | AAA 三段式，一个测试一个行为焦点；造数抽 Builder |
| 独立性 | 单独跑、乱序跑、并行跑结果不变；静态状态必还原 |
| Flaky | 根因表驱动排查（时间/异步/顺序/随机/外部/泄漏），限期修或隔离，禁止重试掩盖 |
| Mock | 只 Mock 信任边界；断言业务结果优先于 verify 调用 |
| 断言 | 粒度对准业务问题，软断言/`assertAll` 聚合失败 |
| 覆盖率 | 看增量与分支，当风险清单用，不当 KPI 用 |
| 面试 | 5 vs 4 架构、Extension vs Rule、assume vs assert、并行开启、切片选型是最高频的五组题 |

至此 JUnit 系列二十篇完结：从测试理念（1）、JUnit 4 体系（2-7）、JUnit 5 架构与新特性（8-13）、迁移（14）、构建与运行生态（15-16），到 Spring Boot / Mockito / Testcontainers 三大集成（17-19），最后以本篇的工程实践收束。测试的终极目标不是绿灯，而是**让每一次红灯都值得被认真对待**。
