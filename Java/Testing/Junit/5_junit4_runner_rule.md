# 1. Runner 机制

JUnit 4 使用 Runner 驱动测试类执行，`@RunWith` 可替换默认 Runner。第 2 篇从架构角度看过 Runner 在执行链中的位置，本篇聚焦**使用与扩展**：Runner 的解析原理、常用内置 Runner、以及 Rule 机制的完整工具箱。两者共同构成 JUnit 4 的扩展模型——Runner 改执行模型（一个类只能有一个），Rule 加环绕逻辑（可以叠加多个），这是贯穿全篇的选择主线。

## 1.1 `@RunWith` 的原理

Runner 的解析发生在 `Request.getRunner()` 阶段，核心逻辑是**注解驱动的反射实例化**：

```text
JUnitCore.run(TestClass.class)
  │
  ▼
Request.aClass() → AllDefaultPossibilitiesBuilder.runnerForClass()
  │
  ├─ 类上有 @RunWith(Xxx.class)？
  │     ├─ 有 → 反射实例化 Xxx（构造器 (Class) 或 (Class, RunnerBuilder)）
  │     │        失败 → 返回 ErrorReportingRunner（跑起来全是初始化错误）
  │     └─ 无 → 继续向下探测
  │
  ├─ public 类且有 @Test 方法？
  │     └─ 是 → new BlockJUnit4ClassRunner(clazz)   ← 默认 Runner
  │
  └─ 都不满足 → 返回 ErrorReportingRunner
```

`@RunWith` 注解本身是 `@Target(TYPE)`（只能标类）+ `@Inherited`（子类自动继承）+ 单值属性 `Class<? extends Runner> value()`，定义即约束。

三个要点：**只能类级标注**（方法级切换 Runner 不存在）；**`@Inherited` 意味着基类标了 `@RunWith(SpringRunner.class)`，所有子类自动生效**（Spring 项目里测试基类的常见写法）；自定义 Runner 必须提供 `(Class<?>)` 或 `(Class<?>, RunnerBuilder)` 签名的 public 构造器——实例化失败不会立刻报错，而是生成一个 `ErrorReportingRunner`，运行时所有条目显示为初始化错误，这是排查自定义 Runner 的第一线索。

## 1.2 默认 Runner：BlockJUnit4ClassRunner

不标注 `@RunWith` 时使用的默认执行器，职责在第 2 篇拆过：反射扫描 `@Test` 方法 → 构造 Statement 装饰链 → 逐方法执行。从使用者视角，它代表 JUnit 4 的**全部默认约束**：测试类必须 public、非抽象、有 public 无参构造；测试方法必须 public void 无参数；每个测试方法一个新实例；类级钩子必须 static。这些约束是它作为"最保守解释器"的产物——JUnit 5 的 Jupiter 引擎逐一解除了它们。

## 1.3 常见 Runner

| Runner | 用途 | 来源 |
|--------|------|------|
| `BlockJUnit4ClassRunner` | 默认执行器 | junit |
| `Parameterized` | 数据驱动参数化测试 | junit |
| `Suite` / `Categories` / `Enclosed` | 套件聚合 / 按 `@Category` 筛选 / 运行静态内部类 | junit |
| `SpringRunner`（`SpringJUnit4ClassRunner`） | Spring 容器集成、依赖注入、事务回滚 | spring-test |
| `MockitoJUnitRunner` | 自动初始化 `@Mock`/`@InjectMocks`、校验多余打桩 | mockito |
| `PowerMockRunner` | 静态方法/final 类/私有方法 Mock | powermock |

三个高频 Runner 的用法：

```java
// ① Parameterized：同一逻辑多组数据（完整示例见第 2 篇 3.1 节）
@RunWith(Parameterized.class)
public class AddParameterizedTest { /* @Parameters 提供数据 + 带参构造 */ }

// ② Suite：显式聚合测试类（CI 分阶段执行时用），类体为空，只执行 @SuiteClasses 列出的类
@RunWith(Suite.class)
@SuiteClasses({UserServiceTest.class, OrderServiceTest.class, PayServiceTest.class})
public class SmokeTestSuite { }

// ③ MockitoJUnitRunner：注解驱动的 Mock 初始化
@RunWith(MockitoJUnitRunner.class)
public class OrderServiceMockTest {

    @Mock
    private InventoryClient inventoryClient;   // Runner 自动创建 Mock 对象

    @InjectMocks
    private OrderService orderService;         // Runner 自动把 @Mock 注入被测对象

    @Test
    public void create_order_should_check_inventory() {
        when(inventoryClient.stock("sku-1")).thenReturn(10);   // 打桩
        assertEquals(2, orderService.create("sku-1", 2).getQuantity());
    }
}
```

`MockitoJUnitRunner` 也可以用 Rule 等价替代（`@Rule public MockitoRule rule = MockitoJUnit.rule();`）——这是"Runner 能力大多有 Rule 平替"的典型例子，也是解决 Runner 名额冲突的标准手法。

## 1.4 Runner 的局限

- **一个测试类只能有一个 Runner**。`@RunWith` 的 value 是单值属性，"参数化 + Spring 注入"、"Mockito + PowerMock"这类需求无法通过叠加注解满足，只能靠第三方桥接（JUnitParams、PowerMock 的 `@PowerMockRunnerDelegate` 委托机制）或自写组合 Runner。
- **自定义 Runner 门槛不低**：要理解 Statement 装饰链、Description 树、RunNotifier 事件协议，还要处理初始化错误、过滤、并发等边界。
- **Runner 粒度是类级**，无法"这个测试方法用 A 模型、那个用 B 模型"。

> **行业实践**：JUnit 4 项目里 Runner 的推荐姿态是**尽量不用**：能用 Rule 解决的用 Rule（Mockito 两者皆可），必须用 Runner 的（Spring、Parameterized）在测试基类上标注一次、子类继承。这样大部分测试类保留默认 Runner，未来向 JUnit 5 迁移时只需处理少数几个基类。

# 2. Rule 机制

Rule 是 JUnit 4 的可组合扩展点，基于装饰器模式：Rule 的 `apply()` 方法拿到内层 Statement，返回包装后的新 Statement，在调用 `base.evaluate()` 前后注入自己的逻辑。它与 `@Before/@After` 的本质区别是**可复用、可参数化、可叠加**——`@Before` 里的资源管理代码每个测试类都要重写，Rule 写一次、到处 `@Rule` 一标就生效。

```text
┌────────────────────────────────────────────────┐
│           Rule 的装饰器原理                      │
│                                                │
│   rule.apply(base, description)                │
│          │                                     │
│          ▼                                     │
│   包装 Statement.evaluate() {                   │
│     前置逻辑（申请资源/起线程）                   │
│     try {                                      │
│       base.evaluate();  ──→ 内层 Statement      │
│     } finally {                                │
│       后置逻辑（释放资源/判定）                   │
│     }                                          │
│   }                                            │
│                                                │
│   多个 Rule 依次包装 → 洋葱结构                   │
└────────────────────────────────────────────────┘
```

## 2.1 Rule 类型：MethodRule 与 TestRule 的区别

JUnit 4 历史上有两个 Rule 接口，**`MethodRule` 已废弃（4.7 起），一律使用 `TestRule`**：

| 维度 | `MethodRule`（废弃） | `TestRule`（推荐） |
|------|---------------------|-------------------|
| 接口签名 | `apply(Statement, FrameworkMethod, Object target)` | `apply(Statement, Description)` |
| 能力 | 可直接操作测试实例字段、反射调用方法 | 只能读注解/名称，通过包装 Statement 干预执行 |
| `@ClassRule` 支持 | ✗（实例还不存在，拿不到 target） | ✓（`Description` 类级方法级统一） |
| 设计倾向 | 能力过强，破坏封装，难以推理 | 能力受限但正交、可组合 |

废弃 `MethodRule` 的动机是**收敛扩展点的能力面**：能拿到测试实例意味着 Rule 可以任意篡改测试对象状态，行为难以预测；`TestRule` 通过 `Description` 只暴露元数据，需要读注解时照样能读，但不给你改对象的机会。JUnit 5 的 Extension 模型延续了这个思路。

## 2.2 `@Rule` 与 `@ClassRule`

- **`@Rule`**：实例字段（public 非 static），每个测试方法生效，Runner 在方法执行前读取字段值构建装饰链。
- **`@ClassRule`**：public static 字段，包装整个测试类（在 `@BeforeClass` 之外再包一层），全类共享一次前后置。适合昂贵资源：嵌入式数据库、Testcontainers 容器、共享的 HTTP 服务器。

执行顺序上，`@ClassRule` 在最外层（先于 `@BeforeClass`），`@Rule` 在方法级装饰链的最外层（先于 `@Before`）——对照第 2 篇的 Statement 洋葱图即可推出。

多个 `@Rule` 字段之间的顺序默认不保证，需要确定顺序时用 **`RuleChain`**：`RuleChain.outerRule(a).around(b).around(c)` 使 a 最先进入、最后退出（洋葱外层），c 最接近测试方法本体。

## 2.3 常用内置 Rule

`org.junit.rules` 包提供的内置 Rule 全家：

| Rule | 用途 | 状态（4.13.2） |
|------|------|---------------|
| `TemporaryFolder` | 临时文件/目录，测试后自动删除 | 可用 |
| `Timeout` | 方法级/类级超时 | 可用（推荐代替 `@Test(timeout)`） |
| `ExternalResource` | 外部资源生命周期模板（before/after 钩子） | 可用 |
| `ExpectedException` | 异常断言 | **已废弃**（见第 4 篇 1.3 节） |
| `ErrorCollector` | 收集多个错误，测试跑完统一报告（软断言） | 可用 |
| `Verifier` / `TestWatcher` / `TestName` | 通过后追加校验 / 观察结果做日志 / 获取当前方法名 | 可用 |
| `RuleChain` | 显式控制多 Rule 顺序 | 可用 |

**TemporaryFolder**——最高频的内置 Rule：

```java
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.TemporaryFolder;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import static org.junit.Assert.assertEquals;

public class ConfigLoaderTest {

    @Rule
    public TemporaryFolder folder = new TemporaryFolder();

    @Test
    public void load_should_parse_yaml_file() throws Exception {
        // newFile：在系统临时目录下建唯一子目录并创建文件，测试结束整个目录被递归删除
        File config = folder.newFile("app.yaml");
        Files.write(config.toPath(), "port: 8080".getBytes(StandardCharsets.UTF_8));

        AppConfig loaded = ConfigLoader.load(config);   // 被测逻辑读真实文件，无需 Mock IO
        assertEquals(8080, loaded.getPort());

        File sub = folder.newFolder("data", "cache");   // newFolder：创建多级目录
    }
}
```

它的价值在于**让文件 IO 测试既真实又无残留**：不 Mock 文件系统（测的是真实 `java.io` 行为），也不用手写 `File.deleteOnExit()` 和目录递归删除（Rule 在 finally 里保证清理，测试失败也会删）。

**Timeout**——推荐的超时控制方式：

```java
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.Timeout;

public class RetryClientTest {

    @Rule
    public Timeout globalTimeout = Timeout.seconds(5);   // 类中所有测试方法限时 5 秒

    @Test
    public void fetch_should_retry_within_limit() {
        // 相比 @Test(timeout=5000)：Timeout Rule 失败信息更清晰，
        // 且 Builder 形式可配置"超时后是否中断线程"等细节
        client.fetch("http://example.com/data");
    }
}
```

> **注意**：`Timeout` Rule 默认也在**独立线程**中执行测试方法（与 `@Test(timeout)` 同机制），ThreadLocal 失效的问题依然存在；`Timeout.builder().withLookingForStuckThread(true)` 可以在超时后识别并报告卡死的线程，便于定位死锁。对"整条流水线限时"的需求，更应该用 Surefire 的 `forkedProcessTimeoutInSeconds` 或 CI 层的 job timeout。

**ExternalResource**——外部资源生命周期模板，语义等价于 `@Before/@After` 但可复用：

```java
import org.junit.rules.ExternalResource;

// 自定义资源 Rule：管理一个嵌入式 HTTP 服务器的启停
public class MockServerResource extends ExternalResource {

    private final int port;
    private Server server;

    public MockServerResource(int port) {
        this.port = port;
    }

    @Override
    protected void before() throws Throwable {
        server = Server.create(port);      // 测试（或测试类）开始前启动
        server.start();
    }

    @Override
    protected void after() {
        if (server != null) {
            server.stop();                 // 结束后停止；失败路径也会执行
        }
    }

    public String baseUrl() {
        return "http://localhost:" + port; // 对外暴露资源句柄
    }
}
```

使用时 `@Rule` 标注则每个测试方法启停一次（隔离最好、最慢），`@ClassRule` static 标注则整个类启停一次（快，但需保证测试不修改共享状态）。`ExternalResource` 是编写资源型 Rule 的捷径——不用自己实现 `TestRule` 接口和 Statement 包装，只覆写 `before()/after()` 两个模板方法。Testcontainers 早期的 JUnit 4 支持（`GenericContainer` 实现 `TestRule`）本质就是这个模式的加强版。

**ErrorCollector**——软断言：`collector.checkThat(user.getName(), is("alice"))` 失败时不中断，继续执行后续校验，测试结束时统一报告全部不匹配项，适合"一次校验多个字段"的场景。JUnit 5 的 `assertAll()` 是它的现代替代。

# 3. 自定义 Rule

所有内置 Rule 都只是 `TestRule` 的实现，自定义 Rule 就是"写一个类实现 `apply()`，返回包装后的 Statement"。以计时 Rule 为例：

```java
// 自定义 Rule：打印每个测试方法的耗时
public class StopwatchRule implements TestRule {

    @Override
    public Statement apply(Statement base, Description description) {
        // apply 在测试执行前被调用；返回的 Statement 才是真正的执行体
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                long start = System.nanoTime();
                try {
                    base.evaluate();                 // 执行内层（@Before → @Test → @After）
                } finally {
                    long costMs = (System.nanoTime() - start) / 1_000_000;
                    System.out.printf("%s cost=%dms%n",
                            description.getDisplayName(), costMs);   // 成败都打印
                }
            }
        };
    }
}
```

使用时 `@Rule public StopwatchRule stopwatch = new StopwatchRule();` 即可。写自定义 Rule 的四个要点：

- **后置逻辑必须放 finally**：测试失败（内层抛 Throwable）时清理和报告逻辑依然要执行——资源型 Rule 的铁律。
- **`apply()` 与 `evaluate()` 时机不同**：`apply()` 在装饰链组装期调用（此时测试还没跑），真正的环绕逻辑写在返回的匿名 Statement 的 `evaluate()` 里。把资源申请写进 `apply()` 是新手最常见的错误。
- **通过 `Description` 读测试元数据**：`description.getAnnotation(...)` 判断当前测试是否标注某注解，实现"按注解开关"的 Rule。
- **需要参数化就写构造器**：Rule 是普通对象，`new Timeout(5000)`——这正是 Rule 相对 `@Before` 的核心优势，钩子方法无法携带配置。

**一个进阶的"重试 Rule"**，展示 `Description.getAnnotation()` 按注解开关的用法：

```java
// 重试 Rule：标注 @Retry(times=3) 的测试失败后自动重跑（治理 Flaky Test 的临时手段）
public class RetryRule implements TestRule {

    @Override
    public Statement apply(Statement base, Description description) {
        Retry retry = description.getAnnotation(Retry.class);   // 读取测试方法上的自定义注解
        int times = (retry == null) ? 1 : retry.times();        // 未标注则不重试
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                Throwable last = null;
                for (int i = 1; i <= times; i++) {
                    try {
                        base.evaluate();                        // 成功即返回
                        return;
                    } catch (Throwable t) {
                        last = t;                               // 记录失败，继续重试
                    }
                }
                throw last;                                     // 全部失败，抛出最后一次异常
            }
        };
    }
}
```

> **设计争议**：重试 Rule 该不该用？它能立刻止住 Flaky Test 引发的 CI 出血，但也**掩盖了不确定性本身**——偶发失败往往指向真实的并发 Bug 或环境竞态。业界共识是：重试只作为过渡手段，必须在重试发生时打日志报警，让"重跑才绿"的测试可被统计和追责，限期修复根因。

# 4. Runner 与 Rule 的选择

两者的分工可以用一句话概括：**Runner 决定"测试如何被发现和组织"，Rule 决定"单个测试的执行过程被如何环绕"**。

| 维度 | Runner（`@RunWith`） | Rule（`@Rule`/`@ClassRule`） |
|------|---------------------|------------------------------|
| 控制粒度 | 整个测试类的执行模型 | 单个测试方法（或类，用 ClassRule）的执行过程 |
| 数量限制 | 一类一个 | 可多个，`RuleChain` 控制顺序 |
| 典型能力 | 参数化、套件、容器集成、Mock 初始化 | 资源管理、超时、异常断言、观察结果 |
| 自定义门槛 | 高（Statement 树 + Description + 事件协议） | 低（实现一个 `apply` 方法） |
| 组合性 | 差（冲突需第三方桥接） | 好（天然叠加） |
| JUnit 5 对应物 | `TestEngine` / `Extension`（类级） | `Extension`（统一模型） |

选择原则：

- **改执行模型才用 Runner**：参数化（一个方法跑 N 组数据）、Spring 容器（测试实例要经过容器后处理）、套件聚合——这些是"测试怎么被组织"层面的事，Rule 做不到。
- **环绕执行过程一律用 Rule**：超时、临时目录、外部资源、结果日志、重试——写一次到处复用，多个能力自由叠加。
- **两者都可选时优先 Rule**：Mockito 初始化既可用 `MockitoJUnitRunner` 也可用 `MockitoRule`，选后者能保住 Runner 名额给 Spring 或 Parameterized。Spring 也提供 `SpringClassRule/SpringMethodRule` 作为 Runner 的平替，都是官方为"腾出 Runner 名额"准备的。

JUnit 5 用 **Extension 模型统一了这两个体系**：`@ExtendWith` 可叠加任意多个扩展，扩展点细分到实例创建、测试前后、参数解析、条件判断等十几个回调——这正是对 JUnit 4 双轨扩展模型十年实践中暴露的问题（组合困难、语义重叠）的系统性解决。

# 5. 小结

| 主题 | 核心要点 |
|------|---------|
| `@RunWith` 原理 | 类级注解、反射实例化；`@Inherited` 使基类标注对子类生效；构造器签名 `(Class)` 或 `(Class, RunnerBuilder)` |
| 默认 Runner | `BlockJUnit4ClassRunner`：反射发现 `@Test`，组装 Statement 装饰链 |
| 常用 Runner | `Parameterized`（参数化）、`Suite`（套件）、`SpringRunner`（容器）、`MockitoJUnitRunner`（Mock 初始化） |
| Runner 局限 | 一类一个、类级粒度、自定义门槛高——生态里大量 Rule 平替由此而来 |
| Rule 原理 | `TestRule.apply()` 返回包装 Statement，装饰器洋葱；后置逻辑必须在 finally |
| MethodRule vs TestRule | MethodRule 已废弃（能拿测试实例、能力过强、不支持 ClassRule）；一律用 TestRule |
| `@Rule` vs `@ClassRule` | 方法级 public 实例字段 vs 类级 public static 字段；ClassRule 适合昂贵共享资源 |
| 内置 Rule | `TemporaryFolder`、`Timeout`、`ExternalResource`、`ErrorCollector`、`TestWatcher`；`ExpectedException` 已废弃 |
| 自定义 Rule | 实现 `apply()`；`apply` 组装期调用、`evaluate` 执行期调用；`Description` 读注解实现按标注开关 |
| 选择原则 | 改执行模型用 Runner，环绕执行用 Rule；冲突时优先 Rule 保住 Runner 名额 |

至此 JUnit 4 系列五篇完结：理念（第 1 篇）→ 架构（第 2 篇）→ 生命周期（第 3 篇）→ 断言与假设（第 4 篇）→ Runner 与 Rule（本篇）。JUnit 4 双轨扩展模型的全部经验，都将在 JUnit 5 的 Platform/Jupiter/Extension 体系里找到对应的升级答案。
