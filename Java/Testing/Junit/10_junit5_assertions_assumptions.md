# 1. Assertions 基本能力

JUnit 5 的断言集中在 `org.junit.jupiter.api.Assertions`，是一个纯静态方法的工具类（JUnit 4 的 `org.junit.Assert` 对应物），配合静态导入使用。所有断言失败都抛出 OpenTest4J 规范的 `AssertionFailedError`（`MultipleFailuresError` 用于聚合断言），这使任何构建工具、IDE、报告框架都能以统一方式识别"这是断言失败而不是普通异常"。相对 JUnit 4，Jupiter 断言的三大增强是：**参数顺序统一为 (expected, actual)、几乎所有断言都支持消息重载或 `Supplier<String>` 延迟消息、新增 assertAll/assertThrows/assertTimeout 等组合与结构化断言**。

## 1.1 常用断言全家

按用途归类，常用断言如下表：

| 断言 | 语义 | 备注 |
|------|------|------|
| `assertEquals(expected, actual)` | 相等 | 基本类型精确比较；对象走 equals；浮点需带 delta 或用 assertEquals(double,double,double) |
| `assertNotEquals` | 不相等 | 与上相反 |
| `assertSame` / `assertNotSame` | 引用同一性 | 即 `==`，不走 equals |
| `assertTrue` / `assertFalse` | 布尔 | 失败消息不带实际值，能不用就不用，优先 assertEquals 语义更明确 |
| `assertNull` / `assertNotNull` | 空值 | — |
| `assertArrayEquals` | 数组逐元素相等 | 支持各基本类型数组与 Object[] |
| `assertIterableEquals` | 任意 Iterable 逐元素相等 | List/Set 通用，元素顺序敏感 |
| `assertLinesMatch` | 文本行匹配 | 支持 `>> skip >>` 快进标记与正则行 |
| `assertTimeout` / `assertTimeoutPreemptively` | 超时 | 见 1.5 节 |
| `assertThrows` / `assertDoesNotThrow` | 异常 | 见 1.3 节 |
| `assertAll` | 聚合断言 | 见 1.4 节 |
| `fail` | 主动失败 | 用于"不该到达的分支" |

```java
package com.example.calc;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.assertArrayEquals;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertIterableEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

class CalculatorTest {

    private final Calculator calc = new Calculator();

    @Test
    void common_assertions() {
        // 相等：参数顺序恒为 (expected, actual)，失败消息据此生成
        assertEquals(3, calc.add(1, 2));
        assertEquals(0.1 + 0.2, calc.add(0.1, 0.2), 1e-9);   // 浮点必须给容差

        // 布尔与空值：可附消息，Supplier 形式只在失败时才拼接字符串
        assertTrue(calc.isReady(), () -> "初始化后应为 ready，实际状态: " + calc.state());
        assertNotNull(calc.history());

        // 集合与数组：逐元素比较，assertEquals 对 List 也可但失败消息不如专用断言清晰
        assertIterableEquals(List.of(1, 2, 3), calc.recentResults());
        assertArrayEquals(new int[]{1, 2, 3}, calc.snapshot());
    }
}
```

**消息参数的两种形态要分清**：`assertEquals(expected, actual, "msg")` 是固定字符串，无论断言成功与否都会先构造；`assertEquals(expected, actual, () -> buildMessage())` 是 `Supplier`，只有失败时才求值。消息构造有成本（查数据库、序列化对象）时必须用 Supplier 形式，这是 JUnit 5 相对 JUnit 4 的实用改进。

> **注意**：`assertEquals(Object, Object)` 比较依赖被测类型的 `equals` 实现。值对象没重写 equals 时断言永远失败，且失败消息里两个对象 `toString` 输出一模一样——这是新手最容易困惑的失败形态。排查顺序：先确认 equals/hashCode，再怀疑断言本身。

## 1.2 与 JUnit 4 断言的差异

| 维度 | JUnit 4 `Assert` | JUnit 5 `Assertions` |
|------|------------------|----------------------|
| assertTrue 带消息 | `assertTrue("msg", cond)` 消息在前 | `assertTrue(cond, "msg")` 消息在后 |
| 消息形态 | 仅 String | String 或 `Supplier<String>` 延迟构造 |
| 异常断言 | `@Test(expected = X.class)` 注解属性 | `assertThrows` 返回异常对象可继续断言 |
| 组合断言 | 无 | `assertAll` 聚合多个可执行断言 |
| 超时 | `@Test(timeout = ...)` / Timeout Rule | `assertTimeout` 两模式 / `@Timeout` |
| 异常体系 | `java.lang.AssertionError` | OpenTest4J `AssertionFailedError`（带 expected/actual 结构） |
| 集合断言 | assertArrayEquals 仅数组 | 新增 assertIterableEquals/assertLinesMatch |

OpenTest4J 这条值得展开：`AssertionFailedError` 携带结构化的 expected/actual 值（而不只是拼好的消息字符串），IDE 据此能做"左右对照 diff 视图"（IntelliJ 点失败断言弹出的对比窗口），构建工具能把失败数据写进报告供后续分析。**断言失败的机器可读化**是 JUnit 5 选择 OpenTest4J 作为共享异常模型的核心原因，TestNG、Spock 也采用同一规范。

# 2. 异常断言

## 2.1 assertThrows

`assertThrows` 取代了 JUnit 4 的 `@Test(expected = ...)`，后者的缺陷是**验证范围过粗**：expected 作用于整个测试方法，方法里任何一行抛出该异常都算通过，无法确认异常确实来自被测调用，也无法断言异常的消息与字段。`assertThrows` 把"预期抛异常的调用"收窄为一个 lambda，并**返回捕获的异常对象**供继续深挖：

```java
package com.example.service;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

class UserServiceTest {

    private final UserService service = new UserService();

    @Test
    void should_reject_null_name() {
        // 返回值就是被捕获的异常对象，可以继续断言消息、字段、cause
        IllegalArgumentException ex = assertThrows(
                IllegalArgumentException.class,
                () -> service.create(null)          // 只有这个调用允许抛异常
        );
        assertEquals("name is required", ex.getMessage());
    }

    @Test
    void should_wrap_io_failure() {
        // 精确匹配类型：子类也算命中（instanceof 语义），要排除子类需自行加断言
        AppException ex = assertThrows(AppException.class, () -> service.load(42));
        assertTrue(ex.getCause() instanceof java.io.IOException,
                "底层原因应为 IOException，实际: " + ex.getCause());
    }
}
```

两个易错点：**其一**，`assertThrows` 是 `instanceof` 语义，预期父类时子类异常也会通过，需要精确类型时补一句 `assertEquals(ExpectedType.class, ex.getClass())`。**其二**，lambda 里若什么都没抛，`assertThrows` 本身失败并给出清晰消息；若抛了别的类型异常，该异常会**原样冒泡**导致测试失败——两种情况都不会漏报。

## 2.2 assertDoesNotThrow

反向断言"这段代码不应抛出任何异常"，典型用于包装了受检异常的调用（lambda 里调受检方法本需 try-catch，交给它更干净）或验证防御性代码路径的健壮性：

```java
@Test
void parse_should_never_throw_for_blank_input() {
    // 任何异常（含受检异常）都会导致失败，失败消息带上原始异常堆栈
    assertDoesNotThrow(() -> parser.parse(""));
}
```

> **设计争议**：`assertDoesNotThrow` 包裹"本来就不该抛异常"的普通调用属于噪音断言——JUnit 的默认语义就是"抛未捕获异常即失败"，多包一层只会稀释测试重点。它的正当场景只有两个：调用签名声明了受检异常而你不想写 try-catch，以及方法名要显式表达"契约就是不抛"。除此之外不要习惯性包裹。

# 3. 聚合断言 assertAll

普通断言是 fail-fast 的：第一个失败立即中止方法，后面的断言根本执行不到。当一个测试需要验证对象的多个属性时（比如校验接口返回的整个 DTO），fail-fast 意味着修一个字段跑一次、修完再跑再暴露下一个——反馈循环被人为拉长。`assertAll` 把多个断言聚合执行：**全部都会跑完，失败项收集进 `MultipleFailuresError` 一次性上报**：

```java
@Test
void should_return_complete_user_profile() {
    UserProfile profile = api.fetchProfile(1001L);

    // 六个断言全部执行，失败项一起报告，而不是修一个跑一遍
    assertAll("用户档案",                                // 聚合组名，报告中的分组标题
        () -> assertEquals("alice", profile.getName()),
        () -> assertEquals(30, profile.getAge()),
        () -> assertTrue(profile.isActive()),
        () -> assertNotNull(profile.getEmail()),
        () -> assertEquals(2, profile.getRoles().size()),
        () -> assertEquals("CN", profile.getRegion())
    );
}
```

`assertAll` 支持嵌套（可执行断言内部再放一个 `assertAll`），报告中按组名分层展示失败。**适用边界要清楚**：聚合断言适合"多项平等的属性校验"；如果断言之间存在依赖（第二条断言的对象由第一条的结果构造），前面的失败会让后面的断言抛出无意义的 NPE 混进报告，这种情况仍应 fail-fast。

> **行业实践**：一个 `assertAll` 内的断言数建议控制在个位数。十几条属性校验塞进一个测试，说明被测对象的契约太宽，通常应按业务关注点拆成多个测试方法（每个方法一个 `assertAll`），失败时报告直接指出是哪个关注点坏了，而不是给出一张长长的失败清单让人自己归类。

# 4. 超时断言

`assertTimeout` 与 `assertTimeoutPreemptively` 签名几乎相同，**执行模型完全不同**，选错会产生"测试过了但线程泄漏"或"测试随机挂"两类隐蔽问题。

## 4.1 两种模式的区别

**assertTimeout（同线程观察）**：可执行体在**当前测试线程**里运行，计时也在同一线程——如果可执行体是死循环或阻塞 IO，`assertTimeout` 自己也被卡住，**永远不会超时失败**，测试直接挂死直到构建工具的全局超时。它真正能检测的是"能正常返回但返回得太慢"的场景。

**assertTimeoutPreemptively（独立线程抢占）**：可执行体在**另一个线程**执行，主线程计时，超时立即失败返回，不等被测代码结束。能处理死循环与阻塞，但代价有三：被测代码运行在不同线程（ThreadLocal、事务上下文、SecurityContext 都不可见）；超时后那个线程**不会被真正杀死**，只是被抛弃（interrupt 尽力而为，卡在 socket read 上的线程根本不理会中断）；断言失败时可能出现"僵尸线程"继续在后台跑，污染同一 JVM 里的后续测试。

| 维度 | assertTimeout | assertTimeoutPreemptively |
|------|---------------|---------------------------|
| 执行线程 | 当前测试线程 | 独立新线程 |
| 能否检测死循环/阻塞 | 不能，会一起挂死 | 能，超时即返回失败 |
| ThreadLocal/上下文 | 天然可见 | 不可见，需手动传递 |
| 超时后线程处置 | — | 无法强杀，interrupt 尽力而为，可能泄漏 |
| 失败确定性 | 高（同线程无竞态） | 受机器负载影响，边界值易随机失败 |
| 适用场景 | 性能回归护栏、常规慢化检测 | 明确怀疑阻塞/死循环的防御性验证 |

```java
@Test
void both_timeout_modes() {
    // 同线程模式：process() 在测试线程执行，适合"应当快速返回"的常规护栏
    assertTimeout(Duration.ofMillis(200), () -> service.process());

    // 抢占模式：疑似阻塞的调用才用它，超时会立即失败但后台线程可能残留
    assertTimeoutPreemptively(Duration.ofMillis(100), () -> legacyBlockingCall());

    // 带返回值与消息的形态：可执行体可以有返回值，失败消息用 Supplier 延迟构造
    String result = assertTimeout(Duration.ofSeconds(1),
            () -> service.render(),
            () -> "render 应在 1s 内完成");
    assertNotNull(result);
}
```

> **注意**：给毫秒级阈值写超时断言在 CI 上极易随机失败——共享构建机的 GC 停顿、CPU 争抢都可能把 50ms 的调用拖到 200ms。**超时断言的阈值应留至少一个数量级的余量**（防的是"秒级死锁/网络挂起"，不是"快了 10ms"），真正的性能回归应该交给 JMH 基准测试而非单元测试里的 assertTimeout。类级统一时限用 `@Timeout` 注解更简洁（见第 9 篇）。

# 5. Assumptions

断言失败是"测试红"，**假设（Assumption）失败是"测试跳过"**——这是两者的本质区别。`org.junit.jupiter.api.Assumptions` 用于表达"本测试只在满足某前提时才有意义"：前提不成立时，测试标记为 skipped（aborted），不算失败也不算通过。它把"环境相关的测试"从 if-return 的手工跳过升级为框架级语义，报告里能清楚区分"跳过的"与"通过的"。

## 5.1 基本用法

三个核心方法：`assumeTrue`/`assumeFalse`（条件成立才继续，否则中止）与 `assumingThat`（条件成立才执行块内断言，但**测试本身不中止**）：

```java
package com.example.tool;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assumptions.assumeTrue;
import static org.junit.jupiter.api.Assumptions.assumingThat;

class FileLockTest {

    @Test
    void should_use_flock_on_linux() {
        // 假设不成立时：抛出 TestAbortedException，测试标记为 skipped
        assumeTrue(System.getProperty("os.name").contains("Linux"),
                "仅在 Linux 上验证 flock 行为");

        assertEquals(0, locker.lockViaFlock("/tmp/demo.lock"));
    }

    @Test
    void config_should_work_everywhere() {
        Config config = Config.load();
        assertEquals("default", config.getMode());        // 所有环境都要过的断言

        // assumingThat：条件块内的断言只在 CI 上执行，
        // 但块外的断言照常执行，测试不会因条件不成立而跳过
        assumingThat(System.getenv("CI") != null,
                () -> assertEquals(4, config.getWorkers()));  // CI 环境 workers 固定为 4
    }
}
```

`assumeTrue` 中止测试走的是 `TestAbortedException`，Surefire/Gradle/IDE 都把它渲染成 skipped；`assumingThat` 内部断言失败则是真正的测试失败。**两者语义差异极大**：前者是"整个测试在此环境下无意义"，后者是"测试的某一部分只在特定环境下验证"，混用会把"应该失败"的测试悄悄变成"跳过"。

## 5.2 适用场景与滥用边界

正当场景集中在**环境依赖**：只在特定 OS/JDK 版本上有效的行为、依赖外部服务且该服务未配置时应跳过（连接字符串为空则 assume）、依赖测试数据文件的可选用例。

> **行业实践**：Assumption 的第一反模式是**用它掩盖 flaky 测试**——"这个测试偶尔失败，加个 assumeTrue(服务可用) 吧"，结果是测试在 CI 上永远 skipped，等于删除了测试但报告还是绿的。第二反模式是用它替代 `@Disabled`/条件注解：整个类在某环境不该跑，应该用 `@EnabledOnOs`/`@EnabledIfSystemProperty`（见第 13 篇）在发现阶段就排除，而不是进了方法体再 abort——前者报告里是"不适用"，后者是"跑了一半放弃"，语义与维护成本都更差。判断标准：**假设应该表达"数据/环境的客观前提"，而不是"我对这个测试没信心"**。

# 6. 与 AssertJ / Hamcrest 的取舍

JUnit 自带断言之外，社区两大主流断言库是 Hamcrest（matcher 风格，JUnit 4 时代标配）与 AssertJ（流式 API，当前事实标准）。三者不是互斥关系——它们抛的都是 OpenTest4J/AssertionError 体系的失败，混用没有技术障碍——但团队应当定一个主风格。

```java
// 同一个校验用三种风格的写法对比
// JUnit 5 原生
assertEquals("alice", user.getName());
assertTrue(user.getRoles().contains("ADMIN"));

// Hamcrest：matcher 组合，可读性依赖 matcher 词汇量
assertThat(user.getName(), is(equalTo("alice")));
assertThat(user.getRoles(), hasItem("ADMIN"));

// AssertJ：流式链式，IDE 补全引导，失败消息自动带实际值
assertThat(user.getName()).isEqualTo("alice");
assertThat(user.getRoles()).contains("ADMIN").doesNotContain("GUEST");
```

| 维度 | JUnit Assertions | Hamcrest | AssertJ |
|------|------------------|----------|---------|
| 依赖 | 零（框架自带） | hamcrest-core/library | assertj-core |
| 风格 | 静态方法，(expected, actual) | matcher 组合子 | 流式链式 API |
| IDE 支持 | 一般 | 差（matcher 需记忆） | 极好（`assertThat(x).` 后补全按类型收窄） |
| 失败消息 | 结构化 expected/actual | 较友好 | 最详细，自动展示集合差异、字段级 diff |
| 集合/Map 断言 | 基础（assertIterableEquals） | 丰富但组合啰嗦 | 最丰富（containsExactly/extracting/filteredOn） |
| 异常断言 | assertThrows 返回对象 | 需额外 matcher | assertThatExceptionOfType 链式 |
| 软断言 | assertAll | SoftAssert 支持弱 | SoftAssertions 一等公民 |
| 现状 | 持续演进 | 基本停止演进，存量为主 | 活跃，Spring 官方测试栈默认 |

取舍的经验法则：

**只用 JUnit 断言也完全成立**——简单相等/空值/异常校验它都覆盖，且零依赖、零学习成本，`assertAll` 补上了软断言，`assertThrows` 补上了异常链。项目断言逻辑简单、或严格控制依赖数量时，原生够用。

**复杂对象图与集合的重度校验选 AssertJ**。`extracting`、`containsOnlyOnce`、`usingRecursiveComparison` 这类能力原生断言完全没有，且失败消息质量差距在集合场景是数量级的（原生只报"两个 List 不等"，AssertJ 精确指出第几个元素哪个字段差在哪）。Spring Boot 测试栈默认引入 AssertJ，Spring 项目顺水推舟即可。

**Hamcrest 主要是存量**。它的 matcher 可组合性理论上最优雅，但实践中 matcher 词汇难以记忆、编译错误信息晦涩、库本身多年无演进，新项目很少再选它；老项目里 `assertThat(x, is(...))` 与 JUnit 5 静态方法混存很常见，不必专门迁移，新写的测试统一到一种风格即可。

> **行业实践**：多数团队的落地规范是"JUnit 断言做基础校验 + AssertJ 做对象/集合校验"的混用，并约定**一个测试方法内不混两种风格**，避免同一个失败报告里两种消息格式交替出现。全量统一到 AssertJ 也常见，代价仅是多一个 test 依赖。

# 7. 小结

| 能力 | API | 关键结论 |
|------|-----|----------|
| 基础断言 | assertEquals/assertTrue/assertNull/assertArrayEquals/assertIterableEquals | 参数序 (expected, actual)；消息用 Supplier 延迟构造；浮点必带 delta |
| 异常断言 | assertThrows / assertDoesNotThrow | assertThrows 返回异常对象可深挖，instanceof 语义注意子类；DoesNotThrow 只用于受检异常与契约表达 |
| 聚合断言 | assertAll | 全部执行、失败合并上报；断言间有依赖时不要聚合 |
| 超时断言 | assertTimeout / assertTimeoutPreemptively / @Timeout | 前者同线程测不了死循环，后者独立线程但上下文丢失且线程可能泄漏；阈值留数量级余量 |
| 假设 | assumeTrue / assumingThat | 假设失败 = skipped；assumingThat 块内失败 = 真失败；不要用假设掩盖 flaky |
| 失败体系 | OpenTest4J AssertionFailedError | 结构化 expected/actual，IDE diff 视图与报告工具的基础 |
| 断言库选型 | JUnit / AssertJ / Hamcrest | 简单场景原生够用，复杂对象图与集合用 AssertJ，Hamcrest 视为存量；单方法内不混风格 |
