# 1. JUnit 4 架构概览

JUnit 4 的核心思想是**通过 Runner 驱动测试类执行**。测试类被 Runner 转换为一组可执行的 Statement，并在生命周期钩子之间执行。相比 JUnit 3 靠"继承 TestCase + 方法名约定"，JUnit 4 转向"注解 + Runner"模型，这是一次架构上的彻底重写——理解这条执行链，才能理解 `@RunWith`、`Rule`、Spring Test 集成这些扩展点为什么长这个样子。本篇基于 JUnit 4.13.2 分析。

## 1.1 关键角色

JUnit 4 的运行模型由五个核心抽象构成，各司其职：

| 角色 | 类型 | 职责 |
|------|------|------|
| **Request** | 抽象类 | 描述"要跑什么"（一个类、一组类、一个方法），负责产出 Runner |
| **Runner** | 抽象类 | 执行测试的引擎，默认是 `BlockJUnit4ClassRunner` |
| **Statement** | 抽象类 | 可执行的测试节点，唯一方法是 `evaluate()`；前后置逻辑通过装饰 Statement 实现 |
| **Description** | 值对象 | 测试的树形描述信息（类名、方法名、注解），用于报告和筛选 |
| **RunNotifier** | 事件通道 | 事件发布器（`fireTestStarted` / `fireTestFailure` / `fireTestIgnored` / `fireTestFinished`），IDE 和构建工具靠它渲染进度 |

这几个角色的关系可以概括为：**Request 找到 Runner，Runner 把测试组织成 Statement 树，执行 Statement 时通过 RunNotifier 向外广播事件，Description 是事件里标识"这是哪个测试"的元数据**。

```text
┌──────────┐  getRunner()   ┌──────────┐  classBlock()  ┌──────────────┐
│  Request │ ────────────→  │  Runner  │ ────────────→  │  Statement   │
└──────────┘                └────┬─────┘                │  （装饰器树） │
                                 │ run()                └──────┬───────┘
                                 ▼                             ▼ evaluate()
                          ┌────────────┐  fire* 事件    ┌────────────┐
                          │ RunNotifier│ ←────────────  │ 测试结果    │
                          └────────────┘                └────────────┘
```

**Runner** 是最关键的抽象，它只有两个必须实现的方法：

```java
public abstract class Runner {
    // 执行测试，事件通过 notifier 广播
    public abstract void run(RunNotifier notifier);

    // 返回测试的树形描述（哪些类、哪些方法）
    public abstract Description getDescription();
}
```

**Statement** 是 JUnit 4 架构的精髓。它把"一段可执行的测试逻辑"抽象成单一接口：

```java
public abstract class Statement {
    // 执行本语句，抛异常即失败；子类通过装饰 base Statement 注入前后置逻辑
    public abstract void evaluate() throws Throwable;
}
```

`@Before`、`@After`、`@Rule`、超时控制……全都是"用一个新 Statement 包住旧 Statement，在调用 `base.evaluate()` 前后做事情"。这就是**装饰器模式**在测试框架里的教科书级应用，第 5 篇的 Rule 机制完全建立在这个模式上。

## 1.2 典型执行流程

`BlockJUnit4ClassRunner` 继承自 `ParentRunner`，后者提供了"遍历子节点"的模板方法。执行一个测试类的完整链路：

```text
JUnitCore.run(CalculatorTest.class)
  │
  ▼
Request.aClass(clazz) → runnerForClass() → new BlockJUnit4ClassRunner(clazz)
  │
  ▼
ParentRunner.run(notifier)
  │
  ├─ classBlock(notifier)          ← 组装类级 Statement 装饰链
  │    ├─ childrenInvoker()        ← 遍历所有 @Test 方法
  │    ├─ withBeforeClasses()      ← 包上 @BeforeClass
  │    ├─ withAfterClasses()       ← 包上 @AfterClass
  │    └─ withClassRules()         ← 包上 @ClassRule
  │
  ▼
Statement.evaluate()
  │
  ▼ 对每个测试方法 runChild(method, notifier)
  │
  ├─ methodBlock(method)           ← 组装方法级 Statement 装饰链
  │    ├─ invokeMethod()           ← 反射调用 @Test 方法本体
  │    ├─ possiblyExpectingExceptions() ← 处理 @Test(expected=...)
  │    ├─ withPotentialTimeout()   ← 处理 @Test(timeout=...)
  │    ├─ withBefores()            ← 包上 @Before
  │    ├─ withAfters()             ← 包上 @After
  │    └─ withRules()              ← 包上 @Rule
  │
  ▼
Statement.evaluate() → 结果经 RunNotifier 广播
```

**要点**：JUnit 4 的生命周期是通过 Statement 的嵌套组合实现的。最终执行的 Statement 是一棵"洋葱"，从外到内大致是：

```text
ClassRule → AfterClass → BeforeClass
              └─ 对每个方法：
                 Rule → After → Before → Timeout → Expected → invokeMethod
```

每一层只关心自己的职责，调用 `base.evaluate()` 把控制权交给内层，并在 `finally` 块里保证后置逻辑一定执行。这种结构的好处是**扩展点天然正交**——加一个新钩子就是多包一层 Statement，不用改动已有逻辑；坏处是调试时栈很深，且各层的先后顺序由包装顺序决定，不直观（这也是面试题常考的"Rule 和 @Before 谁先执行"的根源，答案：Rule 在外层，先于 `@Before`）。

## 1.3 测试发现与过滤

**发现**：`BlockJUnit4ClassRunner` 使用反射扫描测试类，通过 `computeTestMethods()` 找出所有标注 `@Test` 的 public 无参方法，并做合法性校验——类必须有 public 无参构造、不能是抽象类、方法必须 public void 且无参数，否则构造 Runner 时直接抛 `InitializationError`。这就是为什么 JUnit 4 的测试类约束比 JUnit 5 严格得多。

**过滤**：`Request` 与 `Filter` 支持选择性执行。`Filter` 是一个谓词，判断某个 `Description` 是否应该运行：

```java
// 只运行方法名包含 "add" 的测试（IDE 里单跑一个方法的原理）
Request request = Request.aClass(CalculatorTest.class)
        .filterWith(Filter.matchMethodDescription(
                Description.createTestDescription(CalculatorTest.class, "add_should_sum_two_numbers")));
new JUnitCore().run(request);
```

**增强发现**：Suite 与 Category 通过额外 Runner 实现。`@RunWith(Suite.class)` + `@SuiteClasses` 把多个测试类聚合成一个"虚拟测试类"；`@Category` 配合 `Categories` Runner 实现按标签分组运行（JUnit 5 用 `@Tag` 更优雅地解决了同一问题）。

# 2. 核心概念与模型

## 2.1 测试类（Test Class）

JUnit 4 对测试类的硬性约束（违反即 `InitializationError`）：

- **公共构造函数、无参构造**：Runner 通过 `ReflectiveCallable` 反射 `newInstance()` 创建实例，且**每个测试方法都会创建一个全新的测试类实例**（per-method 生命周期）。
- **测试方法为 `public void`，且标注 `@Test`**：无参数，方法体抛出任何异常即失败，正常返回即通过。

per-method 实例化是一个重要设计决策：它保证了测试方法之间**天然不共享实例字段**，`@Before` 里初始化的字段不会泄漏到下一个测试。代价是无法在实例字段里缓存昂贵资源——这正是 `@BeforeClass` 必须 static 的原因（类级别只有一份，挂在 Class 上而非实例上）。JUnit 5 保留了 per-method 默认，但新增了 `@TestInstance(PER_CLASS)` 选项，解除了 static 限制。

## 2.2 Fixture 与生命周期

**Fixture**（测试夹具）是测试执行所需的环境和数据——被测对象、测试数据、临时文件、数据库连接等。JUnit 4 用四个注解管理 Fixture 的生命周期：

```java
import org.junit.After;
import org.junit.AfterClass;
import org.junit.Before;
import org.junit.BeforeClass;
import org.junit.Test;

public class OrderServiceTest {

    private OrderService service;                 // 每个测试方法独享的实例字段

    @BeforeClass
    public static void initClass() {
        // 类级别执行一次：启动嵌入式数据库、加载重量级配置
        // 必须是 static——此时还没有任何测试类实例
    }

    @Before
    public void setUp() {
        service = new OrderService();              // 每个测试方法前重建，保证隔离
    }

    @Test
    public void create_order_should_success() {
        assertNotNull(service.create("item-1"));
    }

    @After
    public void tearDown() {
        // 每个测试方法后执行（即使测试失败也执行，放在 finally 里）
    }

    @AfterClass
    public static void destroyClass() {
        // 类级别执行一次：关闭数据库、释放端口
    }
}
```

`@Before/@After` 负责每个测试方法的前后置，`@BeforeClass/@AfterClass` 负责测试类级别。生命周期细节（继承时的执行顺序、异常处理语义）在第 3 篇展开。

## 2.3 断言与假设

- **断言（Assert）**：验证结果是否符合预期，失败抛 `AssertionError`（Error 而非 Exception——语义上"测试失败"不是程序异常）。断言的实现极其简单：条件不满足就抛错，框架在 Statement 层捕获并记为 failure。
- **假设（Assume）**：在条件不满足时**跳过**测试，失败抛 `AssumptionViolatedException`，Runner 捕获后将其标记为 ignored 而非 failed。典型用途：某测试只在 Linux 上有意义，Windows 上直接跳过而不算失败。

两者的机制差异在第 4 篇详细展开，这里先记住结论：**断言失败 = 测试红；假设不满足 = 测试灰（跳过）**。

# 3. 扩展模型

JUnit 4 提供两个正交的扩展点：Runner 改"执行引擎"，Rule 改"执行过程"。

## 3.1 Runner 扩展

通过 `@RunWith` 切换 Runner。Runner 拿到测试类的完整控制权——它可以决定实例化几次、方法执行几遍、结果怎么算：

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.Parameterized;
import org.junit.runners.Parameterized.Parameters;
import java.util.Arrays;
import java.util.Collection;
import static org.junit.Assert.assertEquals;

// Parameterized Runner：同一测试方法用不同数据跑多遍
@RunWith(Parameterized.class)
public class AddParameterizedTest {

    private final int a;
    private final int b;
    private final int expected;

    // Parameterized 要求带参构造，参数由 @Parameters 方法提供
    public AddParameterizedTest(int a, int b, int expected) {
        this.a = a;
        this.b = b;
        this.expected = expected;
    }

    @Parameters(name = "{0} + {1} = {2}")   // name 定制报告显示
    public static Collection<Object[]> data() {
        return Arrays.asList(new Object[][]{
                {1, 2, 3},                // 正常相加
                {0, 0, 0},                // 边界：零
                {-1, 1, 0},               // 边界：负数
        });
    }

    @Test
    public void add() {
        assertEquals(expected, Calculator.add(a, b));
    }
}
```

Runner 的致命局限是**一个测试类只能有一个 Runner**：`@RunWith(Parameterized.class)` 和 `@RunWith(SpringJUnit4ClassRunner.class)` 不能同时标注。想"参数化 + Spring 注入"就必须自己写一个组合 Runner，这是 JUnit 4 时代大量框架（如 JUnitParams、Spring 的 `SpringClassRule` 折衷方案）出现的直接原因。

## 3.2 Rule 扩展

`Rule` 基于装饰器模式插入额外逻辑——本质是实现 `TestRule` 接口，在 `apply()` 里返回一个包装后的 Statement：

- 临时文件与资源管理（`TemporaryFolder`）
- 超时控制（`Timeout`）
- 期望异常（`ExpectedException`）
- 外部资源启动与销毁（`ExternalResource`）

```java
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.TemporaryFolder;
import java.io.File;
import static org.junit.Assert.assertTrue;

public class FileParserTest {

    @Rule
    public TemporaryFolder folder = new TemporaryFolder();  // 自动建目录、测试后自动删除

    @Test
    public void parse_should_read_file() throws Exception {
        File input = folder.newFile("data.txt");            // 在临时目录里创建文件
        // ... 写入并解析
        assertTrue(input.exists());
    }
}
```

与 Runner 不同，**Rule 可以叠加多个**（多个 `@Rule` 字段 + `RuleChain` 控制顺序），可组合性好得多。Runner/Rule 的完整用法与自定义方式见第 5 篇。

# 4. 架构优缺点

## 4.1 优点

- **扩展点清晰，Runner/Rule 能覆盖多数需求**：换执行模型用 Runner，加环绕逻辑用 Rule，两个抽象各自单一职责。
- **Statement 装饰链设计优雅**：每个生命周期钩子是一层独立装饰器，新增钩子不动旧代码，符合开闭原则。
- **与 Java 生态工具兼容性好**：Maven Surefire、Gradle、各代 IDE 对 JUnit 4 的支持打磨了近二十年，`RunNotifier` 事件模型简单稳定，接入成本极低。
- **约束即规范**：public 类、public 方法、无参构造这些"死板"约束，换来了工具链零配置就能发现和运行测试。

## 4.2 局限

- **Runner 只能单一生效，组合困难**。`@RunWith` 是类级唯一注解，"参数化 + Spring"、"参数化 + Mockito"都需要第三方桥接方案，且各方案的组合规则互不相同。
- **Rule 体系与 Runner 体系分离，模型不统一**。两套扩展机制解决的问题有重叠（比如超时既能用 `@Test(timeout=...)` 也能用 `Timeout` Rule），行为细节却不一致（`@Test(timeout=...)` 在单独线程跑测试，会破坏 ThreadLocal 和预期异常校验），使用者需要记忆大量"哪个在哪个场景下有效"的隐性知识。
- **对并发/参数化/动态测试支持有限**。没有内建的动态测试（运行时生成测试）、嵌套测试组织能力弱（没有 `@Nested`）、并行执行需要靠 Surefire/Gradle 在框架外实现且 Fixture 隔离不彻底。
- **语法约束过时**。测试类和方法强制 public 是 JUnit 3 时代 `TestCase` 继承模型的遗留；`@BeforeClass` 强制 static 让"类级 Fixture 复用实例状态"变得别扭。

> **设计争议**：`@Test(timeout=100)` 的实现是把测试方法放到独立线程执行、主线程限时等待。这带来两个隐蔽问题：一是测试方法里的 `ThreadLocal`（包括某些数据库连接绑定、MDC 日志上下文）失效；二是超时失败与 `expected` 异常校验存在交互 Bug（4.12 之前尤甚）。社区后来的共识是：**超时控制用 `Timeout` Rule 或干脆放到构建工具层（Surefire 的 forkedProcessTimeoutInSeconds），不要用注解属性**。

# 5. 与 JUnit 3 的对比及演进

JUnit 4 是对 JUnit 3 的重写。对比两者能看清"注解驱动"相对"继承驱动"的本质优势：

| 维度 | JUnit 3 | JUnit 4（4.13.2） |
|------|---------|------------------|
| 测试识别 | 继承 `TestCase`，方法名 `test*` 前缀 | `@Test` 注解，方法名任意 |
| 前置/后置 | 覆写 `setUp()` / `tearDown()`，忘调 `super` 就静默失效 | `@Before` / `@After`，多个方法均可标注，父类方法自动执行 |
| 类级钩子 | 无（需自己写静态逻辑） | `@BeforeClass` / `@AfterClass` |
| 异常测试 | 只能 try-catch + `fail()` | `@Test(expected=...)`、`ExpectedException` Rule |
| 扩展机制 | 覆写 `runTest()`，几乎无扩展点 | Runner + Rule + Filter 三层扩展 |
| 跳过测试 | 无原生支持 | `@Ignore`、Assume |
| 套件 | `TestSuite` 手工组装 | `@RunWith(Suite.class)` 声明式 |

JUnit 3 的继承模型有两个硬伤：**Java 单继承**（测试类继承了 `TestCase` 就不能继承自己的基类）和**字符串方法名约定**（把 `testAdd` 拼错成 `tsetAdd`，测试静默不执行，没有任何报错）。JUnit 4 用注解一次性解决了这两个问题，同时通过 Runner 打开了扩展空间——Spring Test、Mockito、PowerMock 的 JUnit 4 集成全部构建在 `@RunWith` 之上。

JUnit 5（Jupiter 5.10+ / Platform 1.10+）则用**模块化架构**取代单一 Runner 模式：

- **JUnit Platform**：测试发现与执行的底座，定义 `TestEngine` SPI，IDE 和构建工具只对接 Platform。
- **JUnit Jupiter**：新的编程模型（`@Test` 来自 `org.junit.jupiter.api`）+ 引擎实现，Extension 模型统一了 Runner 和 Rule（可组合、支持类/方法/参数级扩展）。
- **JUnit Vintage**：在 Platform 上运行存量 JUnit 4（及 JUnit 3）测试的引擎——这就是 JUnit 4 项目"不改一行代码、只加依赖"就能被 JUnit 5 体系运行的原因。

> **行业实践**：截至 JUnit 4.13.2（2021 年发布，也是 JUnit 4 的收官版本），官方已停止功能开发只留安全修复。新项目应直接使用 JUnit 5；存量 JUnit 4 项目通过 `junit-vintage-engine` 平滑共存，逐步迁移。Spring Boot 2.4+ 默认排除 Vintage，Spring Boot 3.x 的 `spring-boot-starter-test` 只带 Jupiter——生态的重心转移已经完成。

# 6. 小结

| 概念 | 要点 |
|------|------|
| 执行模型 | Request → Runner → Statement 树 → RunNotifier 广播事件 |
| Runner | 执行引擎，默认 `BlockJUnit4ClassRunner`；`@RunWith` 替换；一类只能一个 |
| ParentRunner | 模板方法：`classBlock()` 组装类级装饰链，`runChild()` 逐方法执行 |
| Statement | 单一方法 `evaluate()`；生命周期 = 装饰器洋葱（Rule→After→Before→Timeout→Expected→invoke） |
| Description | 测试树元数据，供报告与 Filter 使用 |
| 测试发现 | 反射扫描 `@Test`；约束：public 类、public 无参方法、无参构造 |
| Fixture | per-method 实例化保证隔离；`@BeforeClass` 必须 static |
| 扩展点 | Runner 改执行模型（不可组合），Rule 加环绕逻辑（可叠加） |
| 对比 JUnit 3 | 注解取代继承与方法名约定，解决单继承限制与静默失效 |
| 演进 | JUnit 5 = Platform（底座）+ Jupiter（新模型）+ Vintage（跑存量 JUnit 4） |

下一篇聚焦生命周期：四个钩子注解的执行顺序、继承与异常语义、以及和 JUnit 5 生命周期注解的逐项对照。
