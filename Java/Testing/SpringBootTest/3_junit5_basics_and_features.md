# 1. JUnit 5 架构概览

本目录是 SpringBootTest 专题，JUnit 5 的系统性教程（断言全集、动态测试、迁移指南等）在 `Java/Testing/Junit` 目录另有专篇。这里的定位是：**只讲理解 Spring 测试所必需的 JUnit 5 知识**——尤其是生命周期回调的顺序和 Extension 模型，因为 `SpringExtension` 正是通过这两个机制把容器接进测试执行流程的。

## 1.1 三层架构

**JUnit 5 = Platform + Jupiter + Vintage 三个模块的组合**，理解这个拆分才能明白 Spring 测试挂在哪一层：

- **JUnit Platform**：运行测试的基础平台，定义 `TestEngine` SPI 和 `Launcher` API。IDE、Maven surefire、Gradle 都是通过 Platform 发现并启动引擎的——平台本身不含任何测试注解
- **JUnit Jupiter**：新的测试编程模型（`@Test`、`@BeforeEach`、断言 API）+ 执行它的引擎（Jupiter TestEngine）。Extension 模型也在这一层
- **JUnit Vintage**：兼容 JUnit 4 的引擎，让 `@RunWith` 时代的老测试在 Platform 上继续跑

Spring Boot 默认启用 Jupiter，除非显式引入 Vintage（Boot 2.4 起 starter-test 已移除 Vintage，纯 JUnit 4 项目迁移需手动添加）。**与 Spring 测试直接相关的事实是：`SpringExtension` 是一个 Jupiter Extension，Spring 的容器型测试跑在 Jupiter 引擎上**（JUnit 4 时代的对应物是 `SpringRunner`）；Platform 层对 Spring 透明。

## 1.2 运行模型

**JUnit 5 使用 Extension 机制扩展测试能力，Spring 通过 `SpringExtension` 将测试与容器生命周期连接。** Jupiter 引擎执行测试类时，先收集 `@ExtendWith` 声明的与元注解携带的 Extension（`@SpringBootTest` 元注解自带 `SpringExtension`），再按生命周期顺序逐一回调：`BeforeAllCallback → @BeforeAll → 实例化 → TestInstancePostProcessor（Spring 注入 @Autowired 的时机）→ BeforeEachCallback → @BeforeEach → 参数解析 → @Test → AfterEachCallback → AfterAllCallback → @AfterAll`。

Extension 能挂的点比 JUnit 4 的 `Runner`/`Rule` 细得多，这正是 Spring 选择它的原因——容器初始化、依赖注入、Mock 重置、事务管理各自挂在最合适的回调上，互不干扰。回调与 Spring 侧转译的完整对照见第 6 节。

---

# 2. 生命周期与注解

## 2.1 基础注解

**每个测试方法默认获得一个全新的测试类实例（PER_METHOD 生命周期），这决定了 Spring 注入"每个方法都发生一次"的基本事实。**

| 注解 | 作用 | 执行时机 |
|------|------|---------|
| `@BeforeAll` | 全部测试前执行一次 | 实例化之前（默认要求 static） |
| `@BeforeEach` | 每个测试前执行 | 实例化并注入之后 |
| `@Test` | 测试方法 | — |
| `@AfterEach` | 每个测试后执行 | 测试方法之后（即使失败也执行） |
| `@AfterAll` | 全部测试后执行一次 | 所有方法之后（默认要求 static） |

对 Spring 测试至关重要的顺序保证：**测试实例创建 → Spring 完成依赖注入 → `@BeforeEach` 执行 → 测试方法执行**。所以 `@BeforeEach` 里可以放心使用 `@Autowired` 字段；而 `@BeforeAll` 发生在实例化之前，**拿不到注入的 Bean**——需要在 `@BeforeAll` 里用容器的场合，要么改用 PER_CLASS 生命周期（见 2.2），要么通过 `SpringExtension.getApplicationContext(testClass)` 静态获取。

> **注意**：`@Test` 方法执行抛出异常时 `@AfterEach` 仍会执行，但 `@AfterEach` 自己抛异常会掩盖原始失败。清理逻辑（删测试数据、重置状态）放在 `@AfterEach` 里要做好"它自己也可能失败"的防御。

## 2.2 `@TestInstance` 与生命周期

**默认生命周期为 `PER_METHOD`，每个测试方法创建一个新实例；`PER_CLASS` 则整个测试类共用一个实例。**

- **PER_METHOD**：隔离性更好——实例字段天然不跨用例泄漏；代价是每方法一次实例化 + 一次 Spring 注入（注入本身很便宜，容器是缓存的）
- **PER_CLASS**：可共享状态，适合昂贵初始化；`@BeforeAll`/`@AfterAll` 不再要求 static

```java
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.TestInstance;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class CacheWarmupTest {

    @BeforeAll
    void initOnce() {
        // 非 static：PER_CLASS 下可以访问注入的字段做一次性初始化
    }
}
```

Spring 视角的取舍：**PER_CLASS 让 `@Autowired` 字段在 `@BeforeAll` 里可用，但共享实例意味着一个用例对字段的修改会泄漏给下一个用例**——这正是 Spring 测试容易出 Flaky 的地方。除非初始化确实昂贵（如预热大缓存），保持 PER_METHOD 默认值更安全。Spring 6.1 起 `@BeforeAll` 方法还能直接注入 `TestInfo`、ApplicationContext 等参数，进一步减少了对 PER_CLASS 的需求。

## 2.3 @Nested 与层级组织

**`@Nested` 内部类共享外层类的生命周期回调，Spring 容器配置也沿继承链合并——这是组织"同一个被测对象的多个场景"的利器。**

```java
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest // 容器配置在外层声明一次，内部类共享
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @Nested
    class DiscountCalculation { // 折扣计算的一组场景

        @Test
        void should_apply_5_percent_over_1000() {
            assertThat(orderService.discountRate(new BigDecimal("2000")))
                    .isEqualByComparingTo(new BigDecimal("0.05"));
        }
    }
    // 可并列更多 @Nested 组：VipPolicy、RefundRule……
}
```

要点：`@Nested` 类**不能是 static**；内部类默认也是 PER_METHOD；`@BeforeEach` 顺序是外层先、内层后。对 Spring 测试，`@Nested` 最大的价值是**多个场景组共享同一个上下文缓存 key**——所有内部类复用外层类的容器配置，不会额外创建容器。

---

# 3. 断言与假设

## 3.1 断言

**Jupiter 自带 `Assertions`，但 Spring Boot 项目的事实标准是 starter-test 里的 AssertJ**——流式 API 的失败信息更完整（会打印实际集合内容），且有 IDE 补全引导：

```java
import static org.assertj.core.api.Assertions.assertThat;

// AssertJ：失败信息包含实际值与期望值的差异
assertThat(users).hasSize(3)
                 .extracting(User::getName)
                 .containsExactly("tom", "jerry", "spike"); // 有序精确匹配

// Jupiter 原生（org.junit.jupiter.api.Assertions）：够用但失败信息简陋
assertEquals(expected, actual);
assertAll("user",                          // 聚合断言：一次报告多个失败
        () -> assertEquals("tom", user.getName()),
        () -> assertEquals(1L, user.getId()));
```

两者的选择没有硬性规则，团队内统一即可；混用（一半 `assertThat` 一半 `assertEquals`）才是真正的问题。异常断言用 AssertJ 的 `assertThatThrownBy(...)` 或 Jupiter 的 `assertThrows(...)` 都可，前者能链式断言异常消息与字段。

## 3.2 假设（Assumptions）

**假设用于跳过不满足前置条件的测试——失败的假设不算测试失败，算 skipped。** 最常见的用法是"按环境跳过"：`assumeTrue(System.getProperty("os.name").contains("Windows"))`，前置条件不满足时整个用例标记为 skipped，不红不绿。

> **注意**：假设发生在容器启动之后——`@SpringBootTest` 测试即使所有方法都被 assume 跳过，容器该起还是会起（除非缓存命中）。所以"跳过昂贵环境"的正确层级是构建工具的 tag 过滤（见第 5 节），assume 只适合方法级的细粒度条件。滥用 assume 会让套件"绿得可疑"：大量 skipped 的绿灯不提供任何信心。

---

# 4. 参数化与动态测试

## 4.1 参数化测试

**`@ParameterizedTest` 用一个方法 + 多个数据源替代复制粘贴的相似用例，是边界值测试的首选形态。**

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

class DiscountPolicyTest {

    @ParameterizedTest
    @ValueSource(strings = {"999.99", "1000.00"}) // 单参数字面量
    void should_not_throw(String amount) {
        assertThat(new DiscountPolicy().rate(new BigDecimal(amount))).isNotNull();
    }

    @ParameterizedTest
    @CsvSource({                          // 多参数 CSV：金额 → 期望折扣
            "999.99, 0.00",
            "1000.00, 0.05",
            "5000.00, 0.05"
    })
    void should_calculate_discount_by_amount(String amount, String expected) {
        assertThat(new DiscountPolicy().rate(new BigDecimal(amount)))
                .isEqualByComparingTo(new BigDecimal(expected));
    }
}
```

常见数据来源速查：`@ValueSource`（单参数字面量）、`@CsvSource`（内联多参数）、`@CsvFileSource`（外部 CSV 文件）、`@MethodSource`（复杂对象走静态工厂方法）、`@EnumSource`（遍历枚举全部取值）、`@ArgumentsSource`（自定义 ArgumentsProvider）。**Spring 容器型测试同样支持参数化**——`@SpringBootTest` + `@ParameterizedTest` 组合合法，容器照常缓存复用，每组参数只是重新执行方法体，不会重复启动容器。

> **行业实践**：参数化用例失败时报告会带上参数索引与实参（`[2] amount=1000.00, expected=0.05`），配合 `@ParameterizedTest(name = "...")` 自定义显示名可大幅提升失败可读性。数据超过二十行时移到 `@CsvFileSource`，测试类里塞大 CSV 表会淹没测试意图。

## 4.2 动态测试

**`@TestFactory` 在运行期生成测试用例，适合"用例集合本身来自数据"的场景**（如遍历配置表验证每条规则）：

```java
import org.junit.jupiter.api.DynamicTest;
import org.junit.jupiter.api.TestFactory;

import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.DynamicTest.dynamicTest;

class RuleValidationTest {

    @TestFactory
    Stream<DynamicTest> dynamicTests() {
        return Stream.of("a", "b")
                .map(v -> dynamicTest("test:" + v,
                        () -> assertThat(v).isNotBlank())); // 每个 DynamicTest 独立报告
    }
}
```

与参数化测试的区别：动态测试的用例是运行期才确定的 `Executable`，`@BeforeEach`/`@AfterEach` **只对 `@TestFactory` 方法本身执行一次**，而参数化测试每组参数都完整走一遍前后置。**需要 Spring 事务回滚、Mock 重置这类"每用例前后"语义时用参数化，不要用动态测试**——动态测试的多个用例共享一次前后置，容易互相污染。

---

# 5. 标签与条件执行

## 5.1 标签

**`@Tag` 给测试打上可过滤的元数据，是 CI 分层执行（第 2 篇）的落地机制。** 类级标签作用于整个类，方法级标签与类级叠加：`@Tag("slow")` 标在类上、`@Tag("nightly")` 标在方法上即可。

构建工具中按标签过滤：surefire/failsafe 用 `<excludedGroups>slow</excludedGroups>`（或 `<groups>` 白名单），Gradle 用 `useJUnitPlatform { excludeTags("slow") }`。典型标签体系：`fast`（纯单元）、`slice`（切片）、`integration`（容器集成）、`slow`/`nightly`（端到端、性能）。标签比"按类名 `*IT` 过滤"更灵活——一个类里可以混有快慢用例。

## 5.2 条件执行

**条件注解让用例按环境自动启用/禁用，替代手写 assume：**

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Timeout;
import org.junit.jupiter.api.condition.EnabledIfSystemProperty;
import org.junit.jupiter.api.condition.EnabledOnOs;
import org.junit.jupiter.api.condition.OS;

class PlatformSpecificTest {

    @EnabledOnOs(OS.WINDOWS) // 仅 Windows 执行，其他平台标记 skipped
    @Test
    void windowsOnly() {
    }

    @EnabledIfSystemProperty(named = "env", matches = "ci") // 仅 CI 环境执行
    @Test
    void ciOnly() {
    }

    @Timeout(2) // 单位默认秒；超时即失败，独立线程执行
    @Test
    void shouldFinishInTime() {
        // 2 秒内完成
    }
}
```

内置条件：`@EnabledOnOs`/`@DisabledOnOs`、`@EnabledIfEnvironmentVariable`、`@EnabledIfSystemProperty`、`@EnabledForJreRange`，以及通用的 `@EnabledIf("方法名")`。Spring 还提供容器相关的 `@EnabledIf`（SpEL 读取 Environment 属性），用于"配置开关决定测试是否执行"的场景。

---

# 6. 扩展模型与 Spring 集成

## 6.1 Extension 机制

**Extension 是 Jupiter 的核心扩展点：一切"横切测试执行流程"的能力（注入、Mock、计时、条件）都以 Extension 形态存在，通过 `@ExtendWith` 注册。** 与 JUnit 4 相比，它取代了 `Runner`（一个类只能有一个，不可组合）和 `Rule`（能力有限）：Jupiter 里 `@ExtendWith` 可以声明任意多个、还能通过元注解层层携带——`@SpringBootTest` 自带 `SpringExtension` 正是"元注解携带 Extension"的应用。

## 6.2 SpringExtension

**`SpringExtension` 是 Spring TestContext Framework 在 Jupiter 世界的"翻译官"：它把 Jupiter 的每个生命周期回调转译成 `TestContextManager` 的对应调用。** 纯 Spring（非 Boot）测试需要显式声明 `@ExtendWith(SpringExtension.class)` 配 `@ContextConfiguration`；Boot 测试注解（`@SpringBootTest`、`@WebMvcTest`、`@DataJpaTest` 等）的元注解链上已带 `@ExtendWith(SpringExtension.class)`，**无需手动声明任何 `@ExtendWith`**。

转译关系（理解 Spring 测试行为的关键地图）：

```text
Jupiter 回调                    SpringExtension 转发到 TestContextManager
─────────────────────────────  ─────────────────────────────────────────
BeforeAllCallback          →   beforeTestClass() / 准备上下文
TestInstancePostProcessor  →   prepareTestInstance()  ← @Autowired 注入发生在这里
BeforeEachCallback         →   beforeTestMethod()     ← 事务开启、监听器前置
InvocationInterceptor      →   包裹测试方法执行（事务边界）
AfterEachCallback          →   afterTestMethod()      ← 事务回滚、Mock 重置
AfterAllCallback           →   afterTestClass()       ← @DirtiesContext 检查点
```

> **设计考量**：Spring 团队把注入时机选在 `TestInstancePostProcessor`（实例创建后立即执行）而非 `BeforeEachCallback`，保证了注入先于所有用户回调——这就是"`@BeforeEach` 里 `@Autowired` 字段一定可用"的机制来源。同理，`@BeforeAll` 早于实例创建，天然拿不到注入字段，两件事在时间轴上根本不相交。

---

# 7. 常见实践建议

## 7.1 命名规范

- **类名以 `*Test` 或 `*IT` 结尾**：surefire 默认拾取 `*Test`/`Test*`/`*Tests`/`*TestCase`，failsafe 默认拾取 `*IT`，命名即分层（见第 2 篇 CI 策略）
- **方法名描述行为与期望结果**：`should_apply_discount_when_amount_over_1000` 或 `greetReturnsHello`，失败列表里不点开代码就知道测的是什么。仓库里两种风格并存时跟随现有代码

## 7.2 避免共享状态

- **尽量避免使用静态可变字段**：PER_METHOD 生命周期保证实例字段隔离，但 static 字段跨实例、跨测试类存活，是"单独跑通过、全量跑失败"的头号元凶
- **如需共享，明确生命周期与重置策略**：容器 Bean 是单例，往单例里塞测试数据等于共享可变状态——要么用 `@MockitoBean` 替换该 Bean，要么在 `@AfterEach` 里还原，要么接受并显式声明 `@DirtiesContext`（最后手段）

## 7.3 控制超时

**`@Timeout` 给用例设置硬性时限，防止死锁或失控循环拖死整个套件**（用法见 5.2 示例）。也可以在 `junit-platform.properties` 里全局配置（`junit.jupiter.execution.timeout.default=10s`）。**注意超时不等于 Flaky 修复**：一个因为资源竞争偶尔超过 2 秒的测试，加 `@Timeout(2)` 只是把"偶尔挂起"变成"偶尔失败"，根因还在。超时应该设成"正常耗时的 5-10 倍"，作为失控兜底而非性能断言。

---

# 8. 小结

| 主题 | 关键结论 |
|------|---------|
| 架构 | Platform（引擎 SPI）/ Jupiter（编程模型）/ Vintage（JUnit 4 兼容）；`SpringExtension` 是 Jupiter Extension |
| 生命周期 | 实例创建 → Spring 注入 → `@BeforeEach` → 测试方法；`@BeforeAll` 拿不到注入字段（发生在实例化前） |
| PER_METHOD vs PER_CLASS | 默认 PER_METHOD 隔离性好；PER_CLASS 共享实例，字段泄漏风险换一次性初始化 |
| @Nested | 内部类共享外层容器配置与缓存 key，组织多场景不换容器 |
| 参数化 vs 动态测试 | 参数化每组走完整前后置（事务回滚、Mock 重置生效）；动态测试共享一次，慎用 |
| 标签与条件 | `@Tag` + surefire/failsafe 的 groups 过滤落地 CI 分层；条件注解替代手写 assume |
| Extension 桥接 | `SpringExtension` 把 Jupiter 回调逐一转译为 `TestContextManager` 调用，注入发生在 `TestInstancePostProcessor` |
| 实践 | `*Test`/`*IT` 命名即分层；static 可变字段是 Flaky 之源；`@Timeout` 是失控兜底不是性能断言 |

下一篇深入 Spring Test 的地基——TestContext Framework：`TestContextManager` 的生命周期、ContextLoader、缓存 key 的构成与 `TestExecutionListener` 体系。
