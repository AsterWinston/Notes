# 1. 断言（Assertions）

JUnit 4 的断言工具集中在 `org.junit.Assert`（静态方法类），用于验证预期与实际是否一致。断言的实现原理极其朴素：**条件不满足就抛 `AssertionError`**，Statement 执行链捕获后把测试记为失败——没有魔法，没有字节码增强，这也是 JUnit 断言能力有限、生态里 AssertJ/Hamcrest 长盛不衰的根本原因。本篇覆盖 assert* 全家、浮点 delta、异常断言的三种方案，以及 `Assume` 假设机制。

## 1.1 常用断言

`org.junit.Assert` 的完整方法族（4.13.2），按用途分组：

| 方法 | 语义 | 备注 |
|------|------|------|
| `assertEquals(expected, actual)` | 相等（用 `equals()`） | 最常用；参数顺序是"期望在前" |
| `assertArrayEquals(expected[], actual[])` | 数组逐元素相等 | `assertEquals` 比数组只比引用，必须用这个 |
| `assertTrue(condition)` / `assertFalse(condition)` | 布尔真假 | 失败信息最弱，尽量换 assertEquals |
| `assertNull(obj)` / `assertNotNull(obj)` | 空/非空 | |
| `assertSame(expected, actual)` | 同一引用（`==`） | 验证单例、缓存命中时用 |
| `assertNotSame(unexpected, actual)` | 不同引用 | |
| `assertThat(actual, matcher)` | Hamcrest 匹配器断言 | 见第 2 节 |
| `fail(message)` | 直接失败 | 配合 try-catch 做异常断言、标记不可达分支 |

每个方法都有一个 `String message` 作为第一参数的重载版本，失败时该消息会出现在报告开头。

```java
import org.junit.Test;
import static org.junit.Assert.*;

public class UserTest {

    @Test
    public void user_assertions_demo() {
        User user = userService.find(1L);

        assertEquals("alice", user.getName());       // equals 比较
        assertTrue(user.isActive());                  // 布尔断言
        assertNotNull(user.getCreatedAt());           // 非空断言
        assertSame(User.GUEST, user.getRole());       // 引用同一性断言
        assertArrayEquals(                            // 数组必须用专用方法
                new String[]{"READ", "WRITE"},
                user.getPermissions());
    }
}
```

**参数顺序是"期望值在前、实际值在后"**（`assertEquals(expected, actual)`）。写反了功能上没区别，但失败信息会误导——报告是 `expected:<3> but was:<5>`，写反后你排查时会对着错误的方向找原因。这是 JUnit 断言的第一条纪律。

> **注意**：`assertEquals` 对基本类型走数值比较，对对象走 `equals()`。如果忘了给值对象重写 `equals()`，断言会退化成引用比较而莫名失败；反过来，`Long` 与 `Integer` 类型不同（`equals` 返回 false），`assertEquals(1L, someInt)` 会因自动装箱类型不匹配而编译期就选中 double 重载或直接失败——跨包装类型断言要显式统一类型。

## 1.2 浮点比较

**浮点数比较必须提供误差范围（delta）**，因为二进制浮点无法精确表示大多数十进制小数：

```java
@Test
public void floating_point_assertions() {
    double pi = Math.PI;

    assertEquals(3.1415, pi, 0.0001);        // 正确做法：|expected - actual| <= delta 即通过

    // 错误示范：0.1 + 0.2 = 0.30000000000000004，无 delta 直接失败
    // assertEquals(0.3, 0.1 + 0.2);

    // float 同理
    assertEquals(1.5f, Float.parseFloat("1.5"), 0.0f);
}
```

不带 delta 的 `assertEquals(double, double)` 在 JUnit 4 中**直接抛 `AssertionError` 拒绝对比**（4.4 起的行为，之前是废弃警告）——框架宁可让测试失败，也不让你写出依赖浮点精确相等的脆弱断言。delta 的选取原则：不是越小越好，而是**匹配业务精度**——金额计算到分就取 `0.005`，物理仿真按有效数字取相对误差。

> **行业实践**：涉及金额的场景根本不该用 `double/float`，而应该用 `BigDecimal`，此时断言回到普通的 `assertEquals`，但要注意 `BigDecimal.equals()` 连 scale 一起比较（`1.0` 不等于 `1.00`），比较数值应使用 `assertEquals(0, a.compareTo(b))` 或统一 scale 后再断言。

## 1.3 异常断言（传统方式）

JUnit 4 使用 `@Test(expected = ...)` 或 `ExpectedException` Rule 断言异常：

```java
// 方式一：expected 属性
@Test(expected = IllegalArgumentException.class)
public void parse_null_should_throw() {
    parser.parse(null);
}

// 方式二：try-catch + fail（最原始但控制力最强）
@Test
public void parse_invalid_should_throw() {
    try {
        parser.parse("!!!");
        fail("应抛出 ParseException");          // 没抛异常则走到这里，测试失败
    } catch (ParseException e) {
        assertEquals("非法字符 '!' 位于 0", e.getMessage());   // 可校验消息与字段
    }
}
```

**问题**：`expected` 无法精确校验异常消息与类型层次结构——它只验证"方法体某处抛了这个类型（或子类）的异常"，异常来自哪一行、消息对不对、cause 链是什么，全都无法确认；更糟的是测试方法里**预期抛异常语句之后的断言永远不会执行**（异常一抛方法就退出了），所以 `expected` 方式下你没法"先验证抛异常、再验证状态没被破坏"。

`ExpectedException` Rule（`org.junit.rules.ExpectedException`）补上了一部分能力，可以校验消息和 cause：

```java
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.ExpectedException;

public class ParserExceptionTest {

    @Rule
    public ExpectedException thrown = ExpectedException.none();  // 默认：不期望任何异常

    @Test
    public void parse_invalid_should_throw_with_message() throws Exception {
        thrown.expect(ParseException.class);                      // 期望类型
        thrown.expectMessage("非法字符");                          // 期望消息（子串或 Matcher）
        thrown.expectCause(isA(NumberFormatException.class));     // 期望 cause
        parser.parse("!!!");
        // 注意：此行之后不能再写断言——异常抛出后测试方法即终止
    }
}
```

但 `ExpectedException` 有同样的结构性缺陷（异常后无法继续断言），且**整个 Rule 在 4.13 起被标记废弃**，官方推荐的替代品是 `assertThrows`（JUnit 5 原生提供；JUnit 4 侧用 AssertJ 的 `assertThatThrownBy`，或直接用下面的 AssertJ 风格）。三种方案对比：

| 方案 | 校验类型 | 校验消息/cause | 异常后可继续断言 | 状态 |
|------|---------|---------------|----------------|------|
| `@Test(expected=...)` | ✓（含子类） | ✗ | ✗ | 可用但不推荐 |
| `ExpectedException` Rule | ✓ | ✓ | ✗ | **4.13 起废弃** |
| try-catch + `fail()` | ✓（精确到语句） | ✓ | ✓ | 可用，样板代码多 |
| AssertJ `assertThatThrownBy` | ✓ | ✓（流式） | ✓ | **推荐** |

```java
// 推荐：AssertJ 风格（需引入 assertj-core 依赖）
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@Test
public void parse_invalid_should_throw() {
    assertThatThrownBy(() -> parser.parse("!!!"))
            .isInstanceOf(ParseException.class)       // 类型精确匹配
            .hasMessageContaining("非法字符")          // 消息校验
            .hasCauseInstanceOf(NumberFormatException.class);  // cause 链校验
    // lambda 之外的断言照常执行——这是相对 expected/Rule 的本质优势
    assertFalse(parser.hasPartialResult());
}
```

# 2. Hamcrest Matcher

`assertThat` + Hamcrest 能提高可读性与表达力。JUnit 4 自带 `assertThat(T actual, Matcher<T> matcher)`，配合 `hamcrest-core`（4.13.2 依赖树里自带）和完整的 `hamcrest-library` 使用：

```java
import org.junit.Test;
import java.util.Arrays;
import java.util.List;
import static org.hamcrest.CoreMatchers.*;
import static org.hamcrest.collection.IsIterableContainingInOrder.contains;
import static org.junit.Assert.assertThat;

public class MatcherDemoTest {

    @Test
    public void hamcrest_demo() {
        User user = userService.find(1L);
        List<String> roles = Arrays.asList("admin", "user");

        assertThat(user.getName(), is("alice"));            // is 是 equalTo 的语法糖
        assertThat(roles, hasItem("admin"));                // 集合包含
        assertThat(roles, hasSize(2));                      // 集合大小
        assertThat(roles, contains("admin", "user"));       // 顺序敏感的逐元素匹配
        assertThat(user.getAge(), allOf(                    // 组合匹配器
                greaterThan(18), lessThan(60)));
        assertThat(user.getNickname(), is(nullValue()));    // 空值匹配
    }
}
```

Hamcrest 的核心优势是**失败信息质量**：`assertThat(list, hasItem("admin"))` 失败时会打印整个列表内容和期望的匹配器描述，而 `assertTrue(list.contains("admin"))` 失败时只有一句 "expected true"。Matcher 是可组合的一等对象（`allOf/anyOf/not` 任意嵌套），还能复用为字段常量。

## 2.1 常用 Matcher

| Matcher | 语义 | 所在包 |
|---------|------|--------|
| `is(x)` / `equalTo(x)` | 相等（is 只是包装，改善读感） | `CoreMatchers` |
| `not(x)` | 取反 | `CoreMatchers` |
| `nullValue()` / `notNullValue()` | 空/非空 | `CoreMatchers` |
| `instanceOf(Class)` | 类型判断 | `CoreMatchers` |
| `sameInstance(x)` | 引用同一 | `CoreMatchers` |
| `allOf(...)` / `anyOf(...)` | 与/或组合 | `CoreMatchers` |
| `containsString(s)` / `startsWith` / `endsWith` | 字符串子串/前后缀 | `CoreMatchers` |
| `hasItem(x)` / `hasItems(...)` | 集合包含（不关心顺序） | `IsCollectionContaining` |
| `hasSize(n)` | 集合大小 | `IsCollectionWithSize` |
| `contains(a, b)` | 顺序敏感的逐元素匹配 | `IsIterableContainingInOrder` |
| `containsInAnyOrder(...)` | 顺序无关的全量匹配 | `IsIterableContainingInAnyOrder` |
| `hasEntry(k, v)` / `hasKey(k)` | Map 条目/键 | `IsMapContaining` |
| `closeTo(x, delta)` | 浮点近似 | `IsCloseTo` |

## 2.2 自定义 Matcher

Matcher 是接口，业务对象可以写专用匹配器，把断言意图沉淀为可复用组件：

```java
import org.hamcrest.Description;
import org.hamcrest.TypeSafeMatcher;

// 自定义匹配器：校验用户是否处于"激活且已实名"状态
public class IsActiveVerified extends TypeSafeMatcher<User> {

    @Override
    protected boolean matchesSafely(User user) {
        return user.isActive() && user.isVerified();      // 匹配逻辑
    }

    @Override
    public void describeTo(Description description) {
        description.appendText("一个已激活且已实名的用户");  // 期望描述（失败时显示）
    }

    @Override
    protected void describeMismatchSafely(User user, Description description) {
        description.appendText("实际 active=").appendValue(user.isActive())
                   .appendText(", verified=").appendValue(user.isVerified());
    }
}

// 使用
assertThat(user, new IsActiveVerified());
```

> **设计争议**：Hamcrest 还是 AssertJ？两者解决同一问题（富断言），但风格相反：Hamcrest 是"匹配器对象 + assertThat"，AssertJ 是"链式流式 API"（`assertThat(user).isActive().hasName("alice")`）。Hamcrest 胜在可组合、可复用为字段；AssertJ 胜在 IDE 补全友好、不需要记 Matcher 类名、错误信息更详尽、对集合/异常/日期有专用入口。当前社区趋势明显偏向 AssertJ——Spring Boot 的 `spring-boot-starter-test` 两者都带，但官方示例以 AssertJ 为主。JUnit 4 项目建议：存量 Hamcrest 不动，新代码用 AssertJ。

# 3. 假设（Assumptions）

**Assumption 用于在前置条件不满足时跳过测试（不是失败）**。机制上，`Assume.assumeTrue(false)` 抛出 `AssumptionViolatedException`，Runner 在 Statement 层专门捕获这个异常，把测试标记为 **ignored**（灰色）而非 failed（红色）。断言失败和假设不满足是两种完全不同的信号：前者说"代码错了"，后者说"这个环境跑不了这个测试"。

## 3.1 常用方法

`org.junit.Assume` 的方法族与 `Assert` 一一对应，但语义是"不满足则中止"：

| 方法 | 不满足时的行为 |
|------|--------------|
| `assumeTrue(condition)` / `assumeFalse(condition)` | 抛 `AssumptionViolatedException`，测试跳过 |
| `assumeNotNull(obj...)` | 同上 |
| `assumeThat(actual, matcher)` | 同上（Hamcrest 风格） |
| `assumeNoException(e)` | 传入异常非 null 时跳过（典型：环境探测失败） |

```java
import org.junit.Assume;
import org.junit.Test;
import static org.junit.Assert.assertTrue;

public class OsSpecificTest {

    @Test
    public void should_use_epoll_on_linux() {
        // 前置条件：仅在 Linux 上运行；Windows/macOS 上跳过而不算失败
        Assume.assumeTrue(System.getProperty("os.name").contains("Linux"));

        Transport transport = Transport.create();         // 只有 Linux 会执行到这里
        assertTrue(transport instanceof EpollTransport);
    }
}
```

## 3.2 典型用途

Assume 的适用场景有一条清晰的分界线：**条件的满足与否与被测代码的正确性无关**时才用 Assume。

- **环境相关测试**：依赖特定 OS、JDK 版本、时区、文件编码、网络可达性的测试。CI 矩阵里 Linux 跑全量、Windows 跑兼容子集，就靠 Assume 过滤。
- **可选依赖探测**：测试某个可选集成（如"装了 Redis 才测缓存路径"），连接失败时 `assumeNoException(e)` 跳过。
- **数据前置条件**：参数化测试中，某组数据不适用于当前分支（如"负数开平方"用例只对有复数支持的实现有意义）。
- **第三方服务的可用性**：外部沙箱偶尔宕机，探测失败即跳过，避免把环境问题误报为代码缺陷。

```java
// 探测式跳过：连不上外部沙箱就跳过，而不是让 CI 红一片
private static final String SANDBOX_URL = "https://sandbox.example.com";

@Test
public void should_call_sandbox_api() {
    try {
        ping(SANDBOX_URL);                               // 环境探测
    } catch (IOException e) {
        Assume.assumeNoException("沙箱不可达，跳过", e);    // 环境问题 → 跳过
    }
    SandboxResponse resp = client.call(SANDBOX_URL);     // 环境正常才执行
    assertEquals("OK", resp.getStatus());
}
```

> **注意**：Assume 是双刃剑。跳过不报警的特性意味着——如果条件写错（比如把 `assumeTrue` 的条件写反），**所有测试会静默跳过，CI 全绿，而你什么都没测**。工程上要盯两个指标：构建报告里的 ignored/skipped 数量趋势，以及"某测试类 skipped 比例突然变成 100%"的告警。Maven Surefire 的报告会区分 Tests run / Failures / Errors / **Skipped**，Skipped 异常升高必须排查。

## 3.3 Assume 与 @Ignore 的选择

| 维度 | `@Ignore` | `Assume` |
|------|-----------|----------|
| 决策时机 | 编译期（写死在注解上） | 运行时（动态判断条件） |
| 跳过范围 | 整个方法/类 | 方法内任意位置中止 |
| 适用场景 | 已知问题待修复、功能未实现 | 环境不满足、条件不适用 |
| 风险 | 无声腐化（忘了删） | 条件写反导致全量静默跳过 |
| 报告体现 | ignored | ignored（Vintage/JUnit 4 下同为跳过） |

经验法则：**能用 Assume 表达的条件就不要用 @Ignore**——`@Ignore("Windows 上跑不了")` 应该改成 `assumeFalse(isWindows())`，前者在 Linux CI 上也白白跳过了有效测试。

# 4. 断言设计建议

## 4.1 保持断言与意图一致

- **一个测试只验证一个核心行为**。测试名说"创建用户应成功"，就断言创建的结果，不要顺手断言日志格式、缓存内容、无关字段——断言越多，失败时的信号越模糊，测试对无关变更越敏感（过度约束）。
- **避免在断言中包含过多业务逻辑**。`assertEquals(complexRecalculate(input), result)` 里如果 `complexRecalculate` 是实现代码的镜像复制，测试就退化成"用同一套逻辑验证同一套逻辑"。期望值应该用独立途径得出：手算常量、查表、另一个已知正确的实现。
- **断言强度要够**。只断言 `assertNotNull(result)` 是弱断言——返回任何非空垃圾都能通过。断言到"能区分正确实现和常见错误实现"的粒度为止。

## 4.2 错误信息要清晰

JUnit 4 的每个断言都有带 message 的重载，失败时消息置顶显示：

```java
// 无消息：失败报告只有 "expected:<alice> but was:<bob>"，多断言测试里难以定位
assertEquals("alice", user.getName());

// 有消息：一眼看出是哪个环节、哪条数据的问题
assertEquals("用户名不匹配，userId=" + user.getId(), "alice", user.getName());
```

消息的价值与测试规模成正比：一个测试方法里有 8 个断言、CI 报告折叠成长列表时，"第 3 个断言失败"和"用户名不匹配 userId=1001"的排查效率差一个量级。但要克制——**消息写业务语境，不要重复断言本身**（`assertEquals("期望 3 实际是 result", 3, result)` 是噪音，框架已经会打印期望值和实际值）。

## 4.3 避免过度依赖 Hamcrest

Hamcrest 可读性强，但过度嵌套会让测试变得难懂，需平衡表达力与简洁性。反例：

```java
// 三层嵌套 + 组合匹配器：失败信息冗长，读的人要在脑子里解析语法树
assertThat(order.getItems(),
        hasItem(allOf(
                hasProperty("sku", is("A-100")),
                hasProperty("qty", greaterThan(2)))));

// 拆开写反而更清楚：每个断言一个意图，失败信息精确
Item item = findItem(order, "A-100");
assertNotNull("订单应包含 SKU A-100", item);
assertTrue("A-100 数量应大于 2，实际 " + item.getQty(), item.getQty() > 2);
```

选择标准：**一层匹配器用 Hamcrest/AssertJ（信息质量高），复杂逻辑先提取再简单断言**。断言代码首先是给人读的，其次才是给机器执行的。

# 5. 小结

| 主题 | 核心要点 |
|------|---------|
| assert* 全家 | `assertEquals/assertTrue/assertNull/assertSame/assertArrayEquals/fail`；期望值在前 |
| 实现原理 | 条件不满足抛 `AssertionError`，框架捕获记为失败 |
| 浮点比较 | 必须带 delta；无 delta 的 double 比较在 4.4+ 直接拒绝执行；金额用 `BigDecimal` + `compareTo` |
| 异常断言 | `@Test(expected)` 粒度粗；`ExpectedException` Rule 已废弃；推荐 try-catch+fail 或 AssertJ `assertThatThrownBy` |
| Hamcrest | `assertThat` + Matcher，失败信息质量高、可组合；趋势上被 AssertJ 取代 |
| Assume 机制 | 抛 `AssumptionViolatedException` → 测试记为 ignored（跳过），不是失败 |
| Assume 用途 | OS/JDK/时区等环境条件、可选依赖探测、参数化数据过滤；警惕条件写反导致静默全跳 |
| Assume vs @Ignore | 运行时条件用 Assume，已知问题待修才用 @Ignore |
| 断言设计 | 一测一行为、期望值独立得出、消息写业务语境、避免深度嵌套匹配器 |

下一篇是 JUnit 4 系列的收官：`@RunWith` 的原理与常用 Runner、`TestRule`/`MethodRule`/`@ClassRule` 的机制差异，以及 TemporaryFolder、Timeout、ExternalResource 等内置 Rule 的用法与自定义 Rule 的写法。
