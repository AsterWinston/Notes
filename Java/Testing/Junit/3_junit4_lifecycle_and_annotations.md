# 1. JUnit 4 生命周期概览

JUnit 4 的生命周期围绕测试类与测试方法展开，默认**每个测试方法都会创建新的测试类实例**（per-method 生命周期）。这个默认决定了几乎所有生命周期规则的形状：为什么 `@Before` 里初始化的字段不会污染下一个测试、为什么 `@BeforeClass` 必须 static、为什么重量级资源要用 `@Rule` 的 `ExternalResource` 管理——都源于此。本篇基于 JUnit 4.13.2，把四个钩子注解的执行顺序、继承语义、异常处理和 JUnit 5 对照一次讲清。

## 1.1 生命周期顺序

一个包含两个测试方法的测试类，完整执行序列如下：

```text
┌─────────────────────────────────────────────┐
│ 测试类实例化流程（JUnit 4.13.2）              │
│                                             │
│  @BeforeClass        ← static，全类只执行一次 │
│      │                                      │
│      ▼                                      │
│  ┌─ new TestClass()  ← 每个测试方法一个新实例 │
│  │      │                                   │
│  │      ▼                                   │
│  │  @Before          ← 每个测试方法前执行     │
│  │      │                                   │
│  │      ▼                                   │
│  │  @Test            ← 测试方法本体           │
│  │      │                                   │
│  │      ▼                                   │
│  │  @After           ← 每个测试方法后执行     │
│  └──────┘  （测试失败/异常也保证执行）         │
│      │        ↑ 两个测试方法就循环两次          │
│      ▼                                      │
│  @AfterClass         ← static，全类只执行一次 │
└─────────────────────────────────────────────┘
```

顺序总结为五步：

1. `@BeforeClass`
2. `@Before`
3. `@Test`
4. `@After`
5. `@AfterClass`

**注意**：`@BeforeClass/@AfterClass` 必须是 `public static void` 无参方法。static 的原因在第 2 篇讲过——类级钩子执行时还不存在（或已销毁全部）测试类实例，只能挂在 Class 上。如果标注在非 static 方法上，Runner 构造时直接抛 `InitializationError`，整个类一个测试都跑不了。

## 1.2 per-method 实例化的含义

每个测试方法独享一个新实例，带来两个直接推论：

- **实例字段天然隔离**：`@Before` 里 `service = new OrderService()`，每个测试拿到的都是全新对象，测试之间不可能通过实例字段互相污染。
- **无法用实例字段共享昂贵资源**：想跨测试复用（如嵌入式数据库、解析好的大文件），只能走 `@BeforeClass` 的 static 字段，而 static 状态又要求使用者自己保证只读或每次重置，否则就埋下顺序依赖的雷。

```java
public class ReportServiceTest {

    private static EmbeddedDatabase db;      // static：类级共享
    private ReportService service;           // 实例字段：每个测试独享

    @BeforeClass
    public static void startDb() {
        db = new EmbeddedDatabaseBuilder().build();  // 昂贵的资源只建一次
    }

    @Before
    public void setUp() {
        db.execute("DELETE FROM report");            // 每个测试前重置共享状态
        service = new ReportService(db);
    }

    @AfterClass
    public static void stopDb() {
        db.shutdown();                               // 类结束时释放
    }
}
```

> **设计争议**：per-method 实例化 + static 类钩子的组合是 JUnit 4 被诟病最多的设计之一——"想共享就不能用实例字段，想用实例字段就不能共享"。JUnit 5 用 `@TestInstance(Lifecycle.PER_CLASS)` 解决了这个问题：整个测试类只创建一个实例，`@BeforeAll/@AfterAll` 可以是非 static 方法，Fixture 组织方式灵活得多。

# 2. 常用注解详解

## 2.1 `@Test`

`@Test` 标记测试方法，要求 `public void`、无参数。它有两个可选属性：

- **expected**：声明测试应当抛出的异常类型（含子类），抛了算通过，没抛或抛错类型算失败。
- **timeout**：超时毫秒数，超过即失败。**不推荐作为异常断言和超时控制的首选方式**——它把测试方法放到独立线程执行，破坏 ThreadLocal（数据库连接绑定、MDC 日志上下文都会失效），且与 expected 的交互在旧版本有 Bug。

```java
@Test(expected = IllegalArgumentException.class, timeout = 100)
public void should_throw_exception() {
    // expected：抛出 IllegalArgumentException（或其子类）才算通过
    // timeout：整个方法必须在 100ms 内结束（含抛异常）
    service.parse(null);
}
```

`expected` 的另一个硬伤是**粒度太粗**：它验证的是"整个测试方法抛了这个异常"，无法确认异常来自哪一行调用，也无法校验异常消息。一个方法里三行代码都可能抛 `IllegalArgumentException` 时，测试通过并不代表被测的那行抛的。精确的异常断言方案（try-catch + fail、`ExpectedException` Rule、AssertJ 的 `assertThatThrownBy`）在第 4 篇对比。

## 2.2 `@Before` / `@After`

用于每个测试方法的前置与后置处理，例如数据初始化、资源释放：

```java
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

public class CacheTest {

    private LocalCache cache;

    @Before
    public void setUp() {
        cache = new LocalCache();          // 每个测试前重建缓存，避免脏数据
        cache.put("key", "value");         // 预置公共测试数据
    }

    @After
    public void tearDown() {
        cache.clear();                     // 释放资源；测试失败也会执行
    }

    @Test
    public void get_should_return_cached_value() {
        assertEquals("value", cache.get("key"));
    }
}
```

三条容易踩坑的语义规则：

- **可以有多个 `@Before`/`@After` 方法**，JUnit 4 不保证同类中多个 `@Before` 之间的执行顺序（官方文档明确说"不保证"，实现上按反射返回顺序，不同 JDK 可能不同）。有先后依赖的逻辑写进同一个方法。
- **`@After` 在 finally 语义下执行**：即使 `@Test` 抛异常失败、甚至 `@Before` 抛异常，已成功进入执行链的 `@After` 依然会跑（对应 Statement 装饰链里 `RunAfters` 的 finally 块）。
- **继承时的顺序**：父类的 `@Before` 先于子类执行，子类的 `@After` 先于父类执行（构造顺序、析构逆序，和 C++ 一致）。且**不需要也不能**手动调用 `super.setUp()`——JUnit 会反射收集整条继承链上的所有钩子方法，手动调 super 反而导致执行两次。这是从 JUnit 3 迁移过来最常犯的错。

```java
public class BaseDaoTest {
    @Before
    public void initConnection() { /* 1. 父类 @Before 先执行 */ }
    @After
    public void closeConnection() { /* 4. 父类 @After 最后执行 */ }
}

public class UserDaoTest extends BaseDaoTest {
    @Before
    public void insertFixtures() { /* 2. 子类 @Before 后执行 */ }
    @After
    public void cleanFixtures() { /* 3. 子类 @After 先于父类 */ }
}
```

## 2.3 `@BeforeClass` / `@AfterClass`

用于类级别的初始化和清理，例如启动嵌入式数据库或共享资源。约束：`public static void`、无参、每个类各只执行一次。

```java
public class IntegrationStyleTest {

    private static EmbeddedServer server;    // 类级共享资源必须是 static 字段

    @BeforeClass
    public static void startServer() throws Exception {
        server = EmbeddedServer.start(8080); // 启动一次，全类测试共用
    }

    @AfterClass
    public static void stopServer() {
        if (server != null) {
            server.stop();                   // 判空防御：startServer 失败时 server 为 null
        }
    }
}
```

两个实践要点：`@AfterClass` 里要做**判空防御**，因为 `@BeforeClass` 抛异常时 `@AfterClass` 依然会被调用（`RunAfters` 的 finally 语义同样适用于类级钩子），此时资源字段还是 null；`@BeforeClass` 一旦失败，**整个测试类的所有方法都被记为失败**，不会继续执行——所以类级初始化要尽量轻、尽量稳。

## 2.4 `@Ignore`

跳过测试方法或测试类，被跳过的测试在报告中标记为 ignored（灰色），不算失败：

```java
@Ignore("等待外部依赖稳定，见 JIRA-1234")   // 必须写原因 + 追踪单号
@Test
public void flaky_test() { }
```

建议给出原因，避免"永久忽略"。`@Ignore` 最大的风险不是跳过本身，而是**无声的腐化**：一个没有原因、没有单号的 `@Ignore`，三年后没人知道它为什么存在、还该不该存在。团队实践上应配合构建插件（如 Maven 的 `maven-surefire-report-plugin`）定期审计 ignored 数量，超过阈值报警。

## 2.5 注解与 Rule 的相对顺序

`@Rule` 也是生命周期的一部分，且位于钩子注解的**外层**（见第 2 篇的 Statement 装饰链）：

```text
Rule.apply() 前置逻辑
  → @Before
    → @Test
  → @After
Rule.apply() 后置逻辑（finally）
```

所以 `Timeout` Rule 的计时**不包含** `@Before/@After` 的耗时（它包在更内层的位置由框架决定，4.13.2 中 `withRules` 在 `withBefores/withAfters` 之外），而 `TemporaryFolder` 创建的临时目录在 `@After` 执行完之后才删除——`@After` 里还能访问临时文件。这类"谁包着谁"的细节，用装饰链图推一遍比死记硬背可靠。

# 3. 测试执行顺序

## 3.1 默认顺序

**JUnit 4 不保证测试方法的执行顺序**。官方文档的原话是"顺序确定但不可预测"（deterministic but not predictable）——同一 JDK 版本上多次运行顺序通常一致（依赖 `MethodSorters.DEFAULT` 的 hashCode 排序），但换 JDK 版本就可能变。因此：

- 测试之间**绝不能有顺序依赖**，任何"B 必须在 A 之后跑"的设计都是缺陷；
- 每个测试要能单独运行（IDE 里右键单跑一个方法是日常操作）；
- 共享可变状态（static 字段、外部数据库）必须在 `@Before` 里重置，而不是指望上一个测试留下干净现场。

方法级的无序之外，**类级钩子的顺序是严格保证的**：`@BeforeClass` 一定在所有测试之前，`@AfterClass` 一定在所有测试之后。

## 3.2 `@FixMethodOrder`

确实需要固定顺序时（典型场景：有状态的集成测试、遗留系统迁移期），用 `@FixMethodOrder` 显式声明排序策略：

```java
import org.junit.FixMethodOrder;
import org.junit.Test;
import org.junit.runners.MethodSorters;

@FixMethodOrder(MethodSorters.NAME_ASCENDING)   // 按方法名字典序执行
public class OrderedTest {

    @Test
    public void step1_create() { /* 第一个执行 */ }

    @Test
    public void step2_verify() { /* 第二个执行 */ }

    @Test
    public void step3_cleanup() { /* 第三个执行 */ }
}
```

三种策略：

| 策略 | 排序依据 | 适用场景 |
|------|---------|---------|
| `MethodSorters.DEFAULT` | 方法名 hashCode，确定但不可预测 | 默认值，等价于"别依赖顺序" |
| `MethodSorters.NAME_ASCENDING` | 方法名字典序 | 步骤式测试，用 `step1_/step2_` 前缀控制 |
| `MethodSorters.JVM` | JVM 反射返回顺序 | 基本不用，跨版本最不稳定 |

**建议**：只在无法避免的情况下使用，不作为常规策略。方法名里嵌 `step1/step2` 是一种代码坏味道——它承认了测试之间有状态传递，正确的重构方向通常是把状态准备内联到每个测试，或改用 `@FixMethodOrder` 之外的组织方式（一个测试方法覆盖完整流程）。

# 4. 典型示例

把前面所有概念放进一个完整可运行的例子——一个用户服务测试类，覆盖类级钩子、方法级钩子、异常测试、忽略测试：

```java
import org.junit.After;
import org.junit.AfterClass;
import org.junit.Before;
import org.junit.BeforeClass;
import org.junit.Ignore;
import org.junit.Test;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.fail;

public class UserServiceTest {

    private static InMemoryUserRepository repository;  // 类级共享：static
    private UserService service;                       // 方法级独享：实例字段

    @BeforeClass
    public static void beforeClass() {
        repository = new InMemoryUserRepository();     // 类级别资源初始化（只做一次）
    }

    @Before
    public void setUp() {
        repository.clear();                            // 关键：重置共享状态，切断测试间依赖
        service = new UserService(repository);         // 每个测试拿到全新的 service
    }

    @Test
    public void create_user_should_success() {
        User user = service.create("alice");
        assertNotNull(user.getId());                   // 断言：ID 已生成
        assertEquals("alice", user.getName());         // 断言：名称正确
    }

    @Test
    public void create_duplicate_user_should_throw() {
        service.create("bob");
        try {
            service.create("bob");                     // 第二次创建同名用户
            fail("应抛出 DuplicateUserException");      // 走到这里说明没抛异常，测试失败
        } catch (DuplicateUserException e) {
            assertEquals("bob", e.getExistingName());  // 还能进一步校验异常内容
        }
    }

    @Ignore("用户改名接口尚未实现，见 JIRA-2048")
    @Test
    public void rename_user_should_update_name() { }

    @After
    public void tearDown() {
        // 每次测试后的清理
    }

    @AfterClass
    public static void afterClass() {
        repository = null;                             // 类级别资源释放，帮助 GC
    }
}
```

> **行业实践**：状态重置放 `@Before` 还是 `@After`？两种都有人用，但**放 `@Before` 更稳**：如果测试失败，`@After` 清掉现场后你就无法在 IDE 里断点复现失败时的状态；而且新加的测试类忘了写 `@After` 时，`@Before` 的重置能保证它拿到的仍是干净环境。`@After` 只用于释放**必须显式关闭的资源**（文件句柄、连接、线程）。

# 5. 常见问题与建议

## 5.1 静态资源的副作用

`@BeforeClass` 创建的资源需要保证线程安全和可复用，避免污染后续测试。具体检查三件事：

- **可变共享状态必须每测试重置**：static 的 Map、List、数据库表，在 `@Before` 里清空或回滚，否则测试顺序一变结果就变（而 JUnit 4 恰恰不保证顺序）。
- **并行执行时类间也会互相干扰**：Surefire/Gradle 并行跑多个测试类时，若两个测试类共享同一基类的 static 资源就可能冲突。重量级共享资源考虑动态端口分配，避免写死。
- **失败路径也要能清理**：`@AfterClass` 里判空 + try-catch，一个资源释放失败不应阻止其他资源释放。

## 5.2 与依赖注入框架的协作

JUnit 4 本身不提供 DI，通常通过 Spring Test 或自定义 Runner 实现。Spring 的接入方式是**替换 Runner**：

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)                    // Spring 4.3+ 提供；旧版叫 SpringJUnit4ClassRunner
@ContextConfiguration(classes = AppConfig.class) // 指定 Spring 配置
public class SpringUserServiceTest {

    @Autowired
    private UserService service;                 // 由 Spring 容器注入，而非手动 new

    @Test
    public void create_user_should_success() {
        assertNotNull(service.create("alice"));
    }
}
```

这暴露了 Runner 模型的组合困境：`@RunWith(SpringRunner.class)` 用掉唯一的 Runner 名额后，`Parameterized` 就不能再用了。Spring 的补救方案是 `SpringClassRule` + `SpringMethodRule`（把 Spring 上下文管理从 Runner 挪到 Rule，腾出 Runner 名额），这是 JUnit 4 时代"Runner 不够用、Rule 来凑"的典型案例。JUnit 5 用 `@ExtendWith(SpringExtension.class)` 彻底解决——Extension 可以任意叠加。

## 5.3 钩子方法抛异常的语义

生命周期钩子抛异常时的行为经常被问：

| 抛异常的位置 | 后果 |
|-------------|------|
| `@BeforeClass` | 整个类所有测试记为失败（error），`@AfterClass` 仍会执行 |
| `@Before` | 当前测试记为失败（error），`@Test` 不执行，`@After` 仍会执行 |
| `@Test` | 当前测试记为失败（failure/error） |
| `@After` | 当前测试记为失败（error），即使 `@Test` 本来是通过的 |
| `@AfterClass` | 记一个类级 error，不影响已完成的测试结果 |

注意 `@After` 抛异常会把**本来通过的测试标红**——tearDown 里的清理逻辑要防御性编写（判空、catch 非关键异常），不要让清理失败掩盖测试本身的正确性。

## 5.4 与 JUnit 5 生命周期对照

| 语义 | JUnit 4（4.13.2） | JUnit 5（Jupiter 5.10+） | 差异说明 |
|------|------------------|------------------------|---------|
| 测试方法 | `@Test`（public void） | `@Test`（可 package-private，可抛受检异常） | JUnit 5 解除 public 强制 |
| 每方法前置 | `@Before` | `@BeforeEach` | 改名，语义相同 |
| 每方法后置 | `@After` | `@AfterEach` | 改名，语义相同 |
| 类级前置 | `@BeforeClass`（必须 static） | `@BeforeAll`（默认 static；`PER_CLASS` 下可非 static） | JUnit 5 更灵活 |
| 类级后置 | `@AfterClass`（必须 static） | `@AfterAll`（同上） | 同上 |
| 跳过 | `@Ignore("原因")` | `@Disabled("原因")` | 语义相同 |
| 实例生命周期 | 固定 per-method | `@TestInstance(PER_METHOD / PER_CLASS)` 可选 | JUnit 5 新增 |
| 执行顺序 | `@FixMethodOrder` | `@TestMethodOrder` + `@Order` | JUnit 5 可按注解精确指定单个方法次序 |
| 条件执行 | Assume（运行时跳过） | `@EnabledOnOs` 等注解 + Assumptions | JUnit 5 增加声明式条件 |

迁移时最容易错的三点：注解包名从 `org.junit` 换成 `org.junit.jupiter.api`；`@Before` 不会自动映射（Vintage 引擎跑旧注解没问题，但新写的 Jupiter 测试必须用 `@BeforeEach`）；混用时**一个测试方法上不能同时出现两代注解**，否则 Jupiter 引擎视而不见、Vintage 引擎重复发现。

# 6. 小结

| 主题 | 核心要点 |
|------|---------|
| 默认生命周期 | `@BeforeClass` → (`@Before` → `@Test` → `@After`) × N → `@AfterClass` |
| 实例化策略 | per-method：每个测试方法一个新实例，实例字段天然隔离 |
| 类级钩子 | 必须 `public static void`；失败则全类测试记为失败 |
| 继承语义 | 父类 `@Before` 先执行、父类 `@After` 后执行；不要手动调 super |
| 异常语义 | `@After` 抛异常会把通过的测试标红；钩子失败不阻断后续钩子 |
| 执行顺序 | 方法顺序默认不保证；`@FixMethodOrder(NAME_ASCENDING)` 可固定但慎用 |
| `@Test` 属性 | `expected` 粒度粗、`timeout` 破坏 ThreadLocal，均非首选方案 |
| `@Ignore` | 必须写原因和单号，定期审计，防止无声腐化 |
| Rule 顺序 | Rule 在钩子注解外层：Rule 前置 → @Before → @Test → @After → Rule 后置 |
| JUnit 5 对照 | `@BeforeEach/@AfterEach/@BeforeAll/@AfterAll/@Disabled`，`PER_CLASS` 解除 static 限制 |

下一篇进入断言与假设：`org.junit.Assert` 全家、浮点 delta、异常断言的三种方案对比，以及 `Assume` 的跳过机制。
