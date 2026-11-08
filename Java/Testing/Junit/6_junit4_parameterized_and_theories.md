# 1. 参数化测试（Parameterized）

同一段测试逻辑要在几十组输入上反复验证时，复制粘贴方法体是最糟的解法：一旦断言写错，要改几十处。JUnit 4 的 `Parameterized` Runner 提供的思路是**把"数据"从"逻辑"中剥离出来**——测试逻辑只写一遍，数据由一个静态工厂方法集中供给，框架负责为每组数据各跑一遍所有 `@Test` 方法。这是 JUnit 4 时代做数据驱动测试的标准手段，也是后来 JUnit 5 `@ParameterizedTest` 的直接前身。

## 1.1 为什么参数化必须靠 Runner 实现

**参数化的本质是"一个测试类膨胀成 N 个逻辑测试"**，而 JUnit 4 中控制"一个类如何被发现、实例化、执行"的扩展点只有一个：`@RunWith` 指定的 Runner。默认的 `BlockJUnit4ClassRunner` 只会实例化一次测试类、按方法逐个执行；要实现"每组数据一个新实例"，就必须替换整个执行模型，所以参数化能力被做成了 `Parameterized` Runner。

`Parameterized` 在内部其实继承了 `Suite`：它为每组参数生成一个子 Runner，每个子 Runner 用同一批参数构造测试类实例并执行全部 `@Test` 方法。理解了这一点，就能理解它最大的限制——**`@RunWith` 一个类只能写一次**，参数化 Runner 与 Spring 的 `SpringJUnit4ClassRunner` 无法共存，这在需要依赖注入的集成测试里几乎是致命的（第 1.5 节详述）。

## 1.2 基本写法

标准写法由三部分组成：`@RunWith(Parameterized.class)` 声明 Runner、`@Parameters` 静态方法返回数据集、`@Parameter(n)` 字段接收第 n 列数据：

```java
package com.example.calc;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.Parameterized;
import org.junit.runners.Parameterized.Parameter;
import org.junit.runners.Parameterized.Parameters;

import java.util.Arrays;
import java.util.Collection;

import static org.junit.Assert.assertEquals;

@RunWith(Parameterized.class)                 // 替换默认 Runner，启用参数化
public class CalculatorParamTest {

    // 数据集：每行一个 Object[]，按顺序对应 @Parameter 的下标
    @Parameters
    public static Collection<Object[]> data() {
        return Arrays.asList(new Object[][]{
            {1, 2, 3},                        // a=1, b=2, expected=3
            {2, 3, 5},                        // a=2, b=3, expected=5
            {-1, 1, 0},                       // 负数边界
            {Integer.MAX_VALUE, 0, Integer.MAX_VALUE}
        });
    }

    @Parameter(0)
    public int a;                             // 第 0 列数据

    @Parameter(1)
    public int b;                             // 第 1 列数据

    @Parameter(2)
    public int expected;                      // 第 2 列数据

    @Test
    public void add_should_work() {
        // 逻辑只写一遍，框架用 4 组数据各执行一次
        assertEquals(expected, new Calculator().add(a, b));
    }
}
```

**每组数据都会创建一个新的测试类实例**，所以字段注入不存在跨数据污染问题；反过来，测试类必须有可访问的构造方式（public 类 + 默认构造器或参数匹配的构造器）。执行报告里默认显示为 `add_should_work[0]`、`add_should_work[1]` 这样的下标形式。

## 1.3 数据提供方式的几种形态

`@Parameters` 方法的返回类型和签名有几种合法组合，选哪种取决于数据从哪来：

**返回 `Collection<Object[]>`（默认形态）**。每行数组按位置对应参数，写法如上。缺点是纯位置语义——第 2 列是什么全靠读者对照 `@Parameter(2)`，数据一多可读性迅速劣化。

**返回 `Iterable<Parameters>` 或单参数简化形态**。如果测试只接收一个参数，可以让每行直接是值本身而不是数组，省去一层包装。

**构造器注入替代字段注入**。提供一个参数个数与数据列数一致的构造器，`Parameterized` 会自动用它创建实例。构造器注入的好处是字段可以声明为 `final`，语义上更接近不可变的测试夹具：

```java
@RunWith(Parameterized.class)
public class ConstructorStyleTest {

    private final String input;
    private final int expected;

    // 构造器注入：参数顺序必须与 @Parameters 返回的数组列顺序一致
    public ConstructorStyleTest(String input, int expected) {
        this.input = input;
        this.expected = expected;
    }

    @Parameters(name = "{index}: parse(\"{0}\") = {1}")   // 自定义报告名称
    public static Iterable<Object[]> data() {
        return Arrays.asList(new Object[][]{
            {"42", 42},
            {"-7", -7},
            {"0", 0}
        });
    }

    @Test
    public void should_parse_int() {
        assertEquals(expected, Integer.parseInt(input));
    }
}
```

**`name` 属性是强烈建议配置的**。`name` 支持 `{index}`（数据下标）和 `{0}`、`{1}`（第 n 列的值）占位符，失败报告里会直接显示 `parse("-7") = -7` 而不是冷冰冰的 `[1]`，定位失败数据组的成本大幅下降。

数据源也不必硬编码在方法里。`@Parameters` 只是普通静态方法，可以从 CSV、JSON 文件或数据库读取后组装成 `Collection<Object[]>`：

```java
@Parameters(name = "{index}: {0}")
public static Collection<Object[]> data() throws IOException {
    // 从 classpath 下的 CSV 加载数据，每行拆成数组
    List<Object[]> rows = new ArrayList<>();
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(
            Files.newInputStream(Paths.get("src/test/resources/cases.csv")),
            StandardCharsets.UTF_8))) {
        String line;
        while ((line = reader.readLine()) != null) {
            rows.add(line.split(","));        // 列即参数，注意全是 String 需自行转型
        }
    }
    return rows;
}
```

> **注意**：外置数据文件绕过了编译期类型检查，CSV 里的一列写错类型要到运行时才炸，且失败堆栈里只有字符串。数据文件方案适合"用例由非开发人员维护"（如验收用例表）的场景；纯开发维护的小数据集，写在代码里反而更安全。

## 1.4 执行模型：一组数据一个实例

Parameterized 的运行结构可以画成下面这张图，它与 Suite 的关系一目了然：

```text
┌─────────────────────────────────────────┐
│      Parameterized Runner（本质是 Suite） │
└───────┬────────────┬────────────┬───────┘
        ▼            ▼            ▼
   子 Runner[0]  子 Runner[1]  子 Runner[2]   ← 每组数据一个
        │            │            │
        ▼            ▼            ▼
   new Test(a,b)  new Test(a,b)  new Test(a,b)
        │            │            │
        ▼            ▼            ▼
   全部 @Test 方法 全部 @Test 方法 全部 @Test 方法
```

**实例数 = 数据组数 × 1，方法执行数 = 数据组数 × @Test 方法数**。类中每多一个 `@Test` 方法，都会对全部数据组各跑一遍——如果两个方法只想用不同的数据子集，就必须拆成两个测试类。

## 1.5 优缺点

**优点**是消除重复：逻辑与断言只写一遍，加一组数据只改数据表；报告按数据组展开，哪组失败一目了然；配合 `name` 属性，失败信息可自解释。

**缺点**都源自"它是一个 Runner"这个事实：

- **Runner 独占**：`@RunWith(Parameterized.class)` 占用了唯一的 Runner 槽位，无法再叠加 Spring 注入、PowerMock 等其他 Runner。集成测试想参数化，只能自己写复合 Runner 或放弃。
- **类型不安全**：数据以 `Object[]` 传递，列数、列序、列类型全靠约定，编译器完全不设防。
- **粒度受限**：所有 `@Test` 方法共享同一份数据集，无法按方法定制。
- **可读性一般**：不配 `name` 时失败报告只有下标；数据列多了以后 `{2}` 是什么需要来回翻代码。

> **行业实践**：在必须留在 JUnit 4 的老项目里，如果测试类同时需要 Spring 上下文与参数化，常见做法是不用 `Parameterized` Runner，而是用普通 `@Test` 方法内部循环 + 每轮显式断言（配合消息前缀标明第几组数据）。丑，但能同时保住 `SpringJUnit4ClassRunner` 的注入能力。

# 2. Theories

Theories 是 JUnit 4.4 引入的实验性模块（`org.junit.experimental.theories` 包），它换了个问问题的方式：Parameterized 回答"这几组具体输入的输出对不对"，**Theories 回答"对任意满足条件的输入，某条性质是否恒成立"**。写参数化时你枚举用例，写 Theory 时你陈述规律，数据点由框架自动组合喂进来。

## 2.1 示例

`@Theory` 方法的参数不需要显式数据集，框架从测试类里所有 `@DataPoint` 标注的静态成员中，**按类型匹配**挑值、做笛卡尔积组合：

```java
package com.example.math;

import org.junit.experimental.theories.DataPoint;
import org.junit.experimental.theories.Theories;
import org.junit.experimental.theories.Theory;
import org.junit.runner.RunWith;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertTrue;

@RunWith(Theories.class)
public class MathTheoryTest {

    @DataPoint
    public static int zero = 0;               // 数据点：特殊值 0

    @DataPoint
    public static int one = 1;                // 数据点：单位元 1

    @DataPoint
    public static int negative = -5;          // 数据点：负数

    @Theory
    public void add_should_be_commutative(int a, int b) {
        // 性质陈述：加法交换律对任意数据点组合成立
        // 框架会用 3×3=9 种 (a,b) 组合各执行一次
        assertEquals(a + b, b + a);
    }

    @Theory
    public void add_zero_is_identity(int a) {
        // 性质陈述：0 是加法单位元，3 种组合各执行一次
        assertEquals(a, a + zero);
    }
}
```

**类型匹配是唯一的筛选标准**：`add_should_be_commutative(int, int)` 会拿到所有 `int` 型数据点的两两组合；如果一个 `@Theory` 同时声明 `int` 和 `String` 参数，框架分别从两个类型的池子里取值组合。这意味着数据点之间**没有"行"的概念**——Parameterized 里 `{1, 2, 3}` 是一个整体，Theories 里它们是独立散落的值。

## 2.2 @DataPoints 与相关性数据

单个 `@DataPoint` 一次只能给一个值，成批供给要用 `@DataPoints`（复数），它可以标注返回数组的静态方法或数组字段：

```java
@DataPoints
public static int[] manyValues() {
    return new int[]{2, 4, 8, 16, 32};        // 一次性提供 5 个 int 数据点
}
```

当两个参数**必须相关**（比如"账户"和"该账户的密码"）而不能任意组合时，用对象数组形式的 `@DataPoints` 表达成对数据，再让 `@Theory` 直接接收该对象：

```java
@DataPoints
public static Account[] accounts() {
    return new Account[]{
        new Account("alice", "pwd1"),
        new Account("bob", "pwd2")
    };
}

@Theory
public void login_should_succeed_with_correct_password(Account acc) {
    assertTrue(service.login(acc.name, acc.password));   // 相关数据打包传入
}
```

如果框架找不到某个参数类型的任何数据点，该 `@Theory` 会因"参数无法赋值"被跳过而不是失败——**静默跳过是 Theories 最容易踩的坑**：重命名了数据点字段的类型、忘了加 `static`，测试就悄悄不跑了，报告里却看不到红色。

## 2.3 适用场景

Theories 适合的是**性质恒真、数据可自由组合**的领域：

- **数学与算法性质**：交换律、结合律、幂等性、`encode(decode(x)) == x` 这类往返性质。
- **规则型业务逻辑**：费率计算对任意合法金额单调不减、排序结果对任意输入有序且长度不变。
- **API 契约的普适断言**：对任意合法参数组合都不抛未检查异常、不返回 null。

它本质上是一种轻量级的"穷举 + 性质检查"，思想接近 QuickCheck 一类的基于性质测试（property-based testing），但没有随机生成器与用例收缩（shrinking）能力，只能吃人工声明的有限数据点。

## 2.4 局限

- **组合爆炸**：类型池里 m 个 `int`、n 个 `String`，一个双参 `@Theory` 就要跑 m×n 次；数据点稍多、参数稍长，执行时间呈乘法增长，且没有内置的采样或上限控制。
- **失败定位成本高**：报告里失败的是"某个数据点组合"，但数据点是散落声明的，反查是哪两个值组合出的问题需要自己算笛卡尔积。
- **无关组合制造假失败**：类型匹配不看语义，"负数金额"和"折扣率"可能被组合成业务上不存在的输入，`@Theory` 失败反映的往往是数据点声明得太随意。
- **长期停留在 experimental 包**：API 十几年几乎没有演进，社区资料稀少，`PotentialAssignment`、`AssignabilityFailure` 这些内部概念文档寥寥。

> **设计争议**：Theories 的思路（陈述性质而非枚举用例）被普遍认为是对的方向，但 JUnit 4 这套实现被普遍认为是不成功的落地——类型匹配过于粗放、失败反馈过于原始。它最终没有等到成熟，JUnit 5 直接放弃了对应迁移路径，性质测试的生态位让给了 jqwik、junit-quickcheck 这类专门的第三方引擎。

# 3. 实践建议

## 3.1 JUnit 4 内的取舍

在必须留在 JUnit 4 的项目里，选择依据很简单：**数据是"成行的用例表"就用 Parameterized，性质是"对任意值恒成立"且组合数可控才考虑 Theories**。绝大多数业务测试属于前者。Theories 即使要用，也应把数据点控制在个位数、给 `@Theory` 方法名写清所验证的性质，并定期检查测试报告确认 Theory 真的被执行了而不是被静默跳过。

数据规模大时，优先把数据外置（CSV/JSON + 静态加载方法），并坚持配置 `@Parameters(name = ...)`，否则失败报告的排查成本会吞掉参数化省下的编写时间。

## 3.2 为什么 JUnit 5 用 @ParameterizedTest 取代

JUnit 5 没有迁移 `Parameterized` Runner，也没有迁移 Theories，而是重新设计了 `@ParameterizedTest`，逐项化解了 JUnit 4 方案的结构性缺陷：

| 维度 | JUnit 4 Parameterized | JUnit 5 @ParameterizedTest |
|------|----------------------|---------------------------|
| 实现机制 | 占用 `@RunWith` 的 Runner | 普通注解 + Extension 模型，不独占任何槽位 |
| 与其他能力共存 | 不能与 Spring Runner 等叠加 | 可与 `@ExtendWith(SpringExtension.class)` 自由组合 |
| 粒度 | 整个类共享一份数据，所有 @Test 都参数化 | 单个方法级，同类中参数化与普通测试混用 |
| 数据源 | 只有 `@Parameters` 静态方法一种 | `@ValueSource`/`@CsvSource`/`@MethodSource`/`@EnumSource` 等多种 |
| 类型安全 | `Object[]` 全凭约定 | 方法签名直接声明类型，参数隐式转换 |
| 失败报告 | 默认只有下标 | 默认带参数值，且可自定义 `name` |

关键差异在第一条：**Runner 独占是 JUnit 4 架构层面的死结**，任何在 JUnit 4 内的改良都绕不开它。JUnit 5 把扩展模型从"替换整个执行器"改为"在执行流程的多个回调点上挂载插件"，参数化才终于变成一种可以和其他能力（Spring 注入、Mockito、超时控制）自由叠加的普通注解。这也是读这篇笔记的实际意义：理解 JUnit 4 参数化为什么长这个样子，才能理解 JUnit 5 的扩展模型解决了什么（详见第 12 篇 Extensions 与第 11 篇 JUnit 5 参数化）。

# 4. 小结

| 要点 | 结论 |
|------|------|
| Parameterized 的本质 | 一个继承 Suite 的 Runner，每组数据一个测试类实例，全部 @Test 各跑一遍 |
| 数据提供 | `@Parameters` 静态方法返回 `Collection<Object[]>`；字段注入（`@Parameter(n)`）或构造器注入二选一 |
| 必配项 | `@Parameters(name = "{index}: ...")`，否则失败报告只有下标 |
| 最大限制 | Runner 独占，无法与 SpringJUnit4ClassRunner 等叠加 |
| Theories 的思想 | 陈述"对任意数据点恒成立的性质"，框架按类型匹配自动组合 |
| Theories 的坑 | 组合爆炸、无关组合假失败、数据点类型不匹配时静默跳过 |
| 选型 | JUnit 4 内优先 Parameterized；新项目直接用 JUnit 5 `@ParameterizedTest` |
| 迁移方向 | JUnit 5 未迁移两者，以方法级注解 + 多数据源 + Extension 可组合性全面取代 |
