# 1. 什么是软断言？解决了什么痛点？

在系统测试和复杂业务流程的验证中，我们经常需要对**同一个业务对象的十几个字段**分别断言。传统断言是 fail-fast 的：前面一项一失败就抛出 `AssertionError`，后面的断言代码全部不被执行。AssertJ 提出 **软断言 (Soft Assertions)** 机制来消除这个反馈循环。

## 1.1 硬断言的"修一个跑一遍"困境

假设有一如下常规的强断言（Hard Assertion）测试：

```java
@Test
void hardAssertionExample() {
    User user = userService.getUser(1001);

    assertThat(user.getName()).isEqualTo("Admin"); // 期望 Admin 实际 Guest, 失败即抛出
    assertThat(user.getAge()).isEqualTo(30);       // 永不执行
    assertThat(user.getRole()).isEqualTo("SYS");   // 永不执行
}
```

**痛点**：因为第一行失败抛出 `AssertionError`，方法立即中止，我们**无法在单次构建中得知 age 和 role 是否也错了**。必须修好 name、重跑，才能暴露下一个错——三个字段错要跑三轮。若被测入口本身昂贵（起容器、调远程、造数据），每轮几分钟，这种反馈延迟会被放大到不可接受。

**软断言机制：收集所有断言结果，即便中间遇到失败也不中断代码执行，在收尾阶段一次性输出所有失败明细（`AssertJMultipleFailuresError`）**。官方文档那个"晚宴派对"比喻很传神：硬断言像进门发现少了一位客人就报警，厨房乱没乱、凶器在不在全都不知道；软断言是把整座宅子检查完，一份报告列出全部四处异常——于是能看出"教授袖口有血、书房凌乱"之间的因果。

# 2. `SoftAssertions` 经典用法

AssertJ 提供了原生的 `SoftAssertions` 类：

```java
import org.assertj.core.api.SoftAssertions;
import org.junit.jupiter.api.Test;

public class SoftAssertionTest {

    @Test
    void testStandardSoftAssertion() {
        User user = new User("Guest", 18, "USER");

        // 1. 创建 SoftAssertions 实例(有状态, 一个测试方法一个实例, 绝不能做成成员变量跨用例复用)
        SoftAssertions softly = new SoftAssertions();

        // 2. 通过 softly 开链: 失败不立刻抛出, 而是被代理拦截后暂存
        softly.assertThat(user.getName()).as("校验用户名").isEqualTo("Admin");
        softly.assertThat(user.getAge()).as("校验年龄").isEqualTo(30);
        softly.assertThat(user.getRole()).as("校验角色").isEqualTo("ADMIN");

        // 3. 必须调用! 核算所有收集到的错误, 统一抛出(若有)
        softly.assertAll();
    }
}
```

若两个断言都失败，控制台一次性打印：

```text
org.assertj.core.error.AssertJMultipleFailuresError:
Multiple Failures (2 failures)
-- failure 1 --
[校验用户名]
expected: "Admin"
 but was: "Guest"
-- failure 2 --
[校验年龄]
expected: 30
 but was: 18
```

> **注意**：`as("描述")` 是配合软断言的神器。多条链的 expected/but was 结构高度雷同，聚合报告里没有描述前缀，你根本分不清哪条失败对应哪个业务维度。**用软断言却不写 `as()`，等于自愿读天书**。

## 2.1 忘记 assertAll() 的假绿灯风险

这是手搭 `SoftAssertions` 最危险的坑：**如果不调用 `assertAll()`，即使收集了一堆失败，测试方法也会因正常返回而被判定绿灯通过**——代理对象吞掉了所有 `AssertionError`，没人引爆它们。CI 上一片绿，线上才发现字段全错。这类"沉默的成功"比失败可怕得多。

# 3. 资源式写法：`assertSoftly` 与 `AutoCloseableSoftAssertions`

为了根除忘调 `assertAll()` 的问题，AssertJ 提供两个自动收尾的形态。

## 3.1 Lambda 形态 `assertSoftly`（推荐默认）

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.SoftAssertions.assertSoftly;

public class LambdaSoftAssertionTest {

    @Test
    void testLambdaSoftAssertion() {
        User user = new User("Guest", 18, "USER");

        // 不再手写 assertAll(): lambda 正常结束后框架自动调用
        assertSoftly(softly -> {
            softly.assertThat(user.getName()).as("用户名").isEqualTo("Admin");
            softly.assertThat(user.getAge()).as("年龄").isEqualTo(30);
        });
    }
}
```

作用域由 lambda 边界保证，几乎不可能忘。缺点是所有软断言必须写在同一个代码块内，块内若抛出**非断言类异常**（空指针等），`assertSoftly` 会先丢弃已收集的断言失败、让原始异常冒泡（3.x 行为，不同小版本对 suppressed 处理略有差异）。

## 3.2 try-with-resources 形态 `AutoCloseableSoftAssertions`

断言点分散在多个方法、甚至跨越 setUp/tearDown 时，用 `AutoCloseable` 版本把生命周期交给 JVM：

```java
import org.assertj.core.api.AutoCloseableSoftAssertions;
import org.junit.jupiter.api.Test;

public class TryWithResourcesSoftTest {

    @Test
    void auto_close_soft_assertions() {
        User user = new User("Guest", 18, "USER");

        // try 块退出时 close() 自动触发 assertAll(), 忘不掉
        try (AutoCloseableSoftAssertions softly = new AutoCloseableSoftAssertions()) {
            softly.assertThat(user.getName()).as("用户名").isEqualTo("Admin");
            checkAge(softly, user);                  // 可以把 softly 传给辅助方法继续收集
            softly.assertThat(user.getRole()).as("角色").isEqualTo("ADMIN");
        }
    }

    private void checkAge(AutoCloseableSoftAssertions softly, User user) {
        softly.assertThat(user.getAge()).as("年龄").isEqualTo(30);
    }
}
```

三种形态的收尾责任对照：

| 形态 | 收尾方式 | 忘记引爆的风险 | 适用 |
|------|----------|----------------|------|
| `new SoftAssertions()` | 手写 `assertAll()` | **高**（假绿灯） | 需要 `assertAlso` 组合多个收集器 |
| `assertSoftly(softly -> ...)` | lambda 结束自动 | 无 | 断言集中在一段代码内（默认选择） |
| `AutoCloseableSoftAssertions` | try 块退出自动 | 无 | 断言跨多个私有方法 |
| `@ExtendWith(SoftAssertionsExtension)` | 扩展回调自动 | 无 | 整个测试类大量使用软断言 |

# 4. JUnit 5 集成：`SoftAssertionsExtension` 与替代方案

## 4.1 扩展注入式（类内高频使用的工业写法）

AssertJ 自带 JUnit 5 扩展 `org.assertj.core.api.junit.jupiter.SoftAssertionsExtension`，做到"无侵入"注入：

```java
import org.assertj.core.api.SoftAssertions;
import org.assertj.core.api.junit.jupiter.InjectSoftAssertions;
import org.assertj.core.api.junit.jupiter.SoftAssertionsExtension;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

// 1. 注册扩展(Boot 2.2+ / JUnit 5.1+ 均可用)
@ExtendWith(SoftAssertionsExtension.class)
public class AdvancedSoftTest {

    // 方式 B: 字段注入, 供 @BeforeEach 等生命周期方法共享
    @InjectSoftAssertions
    private SoftAssertions softlyInField;

    // 方式 A: 参数注入, 每个测试方法拿到全新实例
    @Test
    void testWithExtension(SoftAssertions softly) {
        User user = new User("Guest", 18, "USER");

        softly.assertThat(user.getName()).isEqualTo("Admin");
        softly.assertThat(user.getAge()).isEqualTo(30);

        // 重点: 连 assertAll() 都不用写!
        // 扩展在测试方法结束回调里自动执行 assertAll, 统一引爆收集到的失败
    }
}
```

参数注入保证**每方法一个新实例**，天然规避状态泄漏；字段注入（`@InjectSoftAssertions`）允许在 `@BeforeEach` 里收集，但要求你克制住把它当"线程安全的日志器"乱用的冲动。

> **行业实践**：这是工业代码里最推荐的形态——代码结构与硬断言一样干爽，又保留批量容错与聚合报告。落地前提是团队统一引 `assertj-core` 3.20+ 并对测试基类集中挂扩展，零散混用反而增加认知负担。

## 4.2 与 JUnit 5 原生 `assertAll` 的取舍

JUnit Jupiter 自带的 `assertAll` 干的是同一件事（聚合执行、统一上报 `MultipleFailuresError`），两条路线对照：

| 维度 | AssertJ SoftAssertions | JUnit 5 assertAll |
|------|------------------------|-------------------|
| 写法 | `softly.assertThat(x).isEqualTo(y)` 一条链任意深度 | 每个断言要包一个 `Executable` lambda |
| 断言 API | AssertJ 全家桶（集合/时间/递归比较） | 可与 AssertJ 混用，也可用原生 |
| 失败类型 | `AssertJMultipleFailuresError` | `MultipleFailuresError` |
| 动态收集 | 实例可随时 `assertAlso` 合并、循环内追加 | 组装 `Stream<Executable>` 也能动态 |
| 忘引爆风险 | 手搭形态有（见 3 节） | 无（不调 assertAll 里的断言根本不执行） |
| 心智成本 | 记住"最后要收尾" | 记住"每个断言要包 lambda" |

```java
// JUnit assertAll 的等价写法: 每个断言一个 lambda, 嵌套深时括号噪声明显
assertAll("用户档案",
        () -> assertThat(user.getName()).isEqualTo("Admin"),
        () -> assertThat(user.getAge()).isEqualTo(30));
```

经验法则：**断言本身就用 AssertJ 的项目，软断言优先 assertSoftly/扩展；只有零散几处要聚合、不想引入第二套生命周期概念时，用 assertAll 包住普通断言更轻**。两者不互斥，assertAll 的 lambda 里放 AssertJ 硬断言是完全合法的混搭。

# 5. 何时该用、何时不该用

软断言不是"更高级的断言"，它是**报告策略**。用错场景会实打实地损害测试质量。

**适合软断言的场景**：

- **同一 DTO/实体的多字段校验**：接口返回 15 个字段的画像，聚合报告让一次运行暴露全部偏差。
- **循环内对每个元素断言**：遍历 100 条导出记录核对格式，fail-fast 会让你修一条跑一遍。
- **昂贵的端到端冒烟检查**：一次部署 20 分钟，检查点 30 个，绝不能"第一个挂就全停"。
- **迁移/重构对账**：老新两套实现输出逐字段 diff，目的就是"一次看清差多少"。

**不适合的场景（此时软断言在掩盖真实问题）**：

```java
// 错误示范: 前置条件也用软断言 —— 用户不存在时, 后面每条链都 NPE, 聚合出 8 条无意义失败
assertSoftly(softly -> {
    softly.assertThat(user.getName()).isEqualTo("Admin");   // user 为 null!
    softly.assertThat(user.getAge()).isEqualTo(30);
});

// 正确做法: 前置条件硬断言(不满足立即止损), 平行的业务字段才软断言
assertThat(user).as("被测用户必须存在").isNotNull();
assertSoftly(softly -> {
    softly.assertThat(user.getName()).as("用户名").isEqualTo("Admin");
    softly.assertThat(user.getAge()).as("年龄").isEqualTo(30);
});
```

- **断言之间存在依赖**：后续检查建立在前面结果之上时，fail-fast 的短路正是你想要的——继续执行只会制造误导性级联失败。
- **测试只有一两个断言**：软断言的收尾仪式与聚合报告纯属多余噪音。
- **用它来"容忍已知失败"**：今天想少看一条红就摘掉 assertAll 的时机，明天就没人记得为什么这条链一直是错的。已知失败应该用 `@Disabled`/`@Tag` 显式标记，**不允许用软断言静默降格**。

> **设计争议**：软断言会削弱"一个测试一个概念"的单一断言原则——聚合报告里 6 条失败对应 6 个概念，失败归因比拆分成 6 个小测试更难。反方观点认为参数化（@ParameterizedTest + @CsvSource）可以把多字段拆成多用例从而兼得聚合（JUnit 报告天然是列表）与隔离。成熟团队的折中：**同一对象同一快照的字段簇用软断言；跨场景、跨输入的规则矩阵用参数化**。

# 6. 小结

软断言把"一个失败即中止"的硬断言改造成"收集—合并—统一引爆"的报告模式，核心 API 三件套：`new SoftAssertions()+assertAll()`、`assertSoftly(lambda)`、JUnit 5 的 `SoftAssertionsExtension`（+`@InjectSoftAssertions`）。纪律有两条：**其一，永远保证引爆步骤不会被忘记**（手搭形态是假绿灯高危区，默认选 lambda 或扩展）；**其二，只在平行独立的字段簇上使用**，前置条件必须硬断言，有依赖链的断言不许软化——软断言解决的是反馈循环长度，不是测试质量问题，更不能当遮羞布。其"代理拦截 AssertionError 存入 ErrorCollector"的实现机制，见第 7 篇源码解析。
