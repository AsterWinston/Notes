# 1. 什么是 AssertJ

**AssertJ 是一个为 Java 测试提供强类型、流式（Fluent）、高可读性断言 API 的开源库**。它为标准 Java 类型（字符串、数值、集合、Map、异常、Optional、日期时间、文件路径、并发对象等）准备了一整套语义化断言方法，目标是让测试代码读起来接近自然语言："断言这个字符串，非空、以 Hello 开头、包含 Assert、以 J 结尾"。

在 Java 单元测试生态中，AssertJ 与测试执行框架是**互补而非替代**关系：JUnit 4/5、TestNG 负责"调度和执行"测试用例（生命周期、参数化、并行、报告），AssertJ 只负责"验证结果是否符合预期"并在失败时生成人类可读的诊断信息。因此你可以在同一个测试类里混用 JUnit 的 `@Test`、`@ParameterizedTest` 与 AssertJ 的断言，二者没有冲突。

## 1.1 在测试技术栈中的位置

一个典型的企业级单测技术栈是自上而下分层的，AssertJ 占据的是"断言层"：

```text
┌───────────────────────────────────────────────┐
│ 执行层: JUnit 5 / TestNG                      │  调度、生命周期、并行、报告
├───────────────────────────────────────────────┤
│ 断言层: AssertJ(assertj-core)                 │  结果校验、失败诊断信息
├───────────────────────────────────────────────┤
│ 模拟层: Mockito / BDDMockito                  │  依赖隔离、行为验证
└───────────────────────────────────────────────┘
```

断言失败时，AssertJ 抛出 OpenTest4J 规范的 `org.opentest4j.AssertionFailedError`（由内部工厂反射构造；OpenTest4J 是可选类路径依赖，缺失时回退为普通 `java.lang.AssertionError`），所以 IDE 的失败对照视图、Maven Surefire 报告、CI 平台的测试结果解析都能像处理 JUnit 原生断言一样处理它。

## 1.2 模块家族与版本基线

AssertJ 不是一个孤立的 jar，而是一个家族：

| 构件 | 用途 |
|------|------|
| `assertj-core` | 核心库，覆盖 JDK 标准类型，本篇及后续所有内容的主体 |
| `assertj-guava` | Guava 类型（Multimap、Optional 等）的专用断言 |
| `assertj-db` | 数据库结果集断言，把 `ResultSet` 当表格校验 |
| `assertj-assertions-generator` | 扫描 POJO 包自动生成专用 Assert 类的 Maven/Gradle 插件 |
| `assertj-swing` | 遗留的 Swing UI 测试库，新项目中已很少使用 |

版本基线：**3.x 是当前主线，要求 Java 8 及以上**；3.x 失败异常走 OpenTest4J 异常族（AssertionFailedError/MultipleFailuresError，类路径缺失时自动回退普通 AssertionError），IDE 可展示 expected/actual diff；4.x 处于预发布阶段，面向 Java 17+。团队落地时直接跟随 Spring Boot BOM 管理的版本或取 3.x 最新补丁版即可，API 在 3.x 内保持向后兼容。

> **注意**：`assertThat` 在 `org.assertj.core.api.Assertions`（AssertJ）与 `org.hamcrest.MatcherAssert`、`org.junit.Assert`（JUnit 4）里都存在同名静态方法。IDE 自动导入时导错包，会得到完全不同的方法签名，这是新手最常见的"AssertJ 方法不存在"假象——先检查 import。

# 2. 为什么选择 AssertJ？（与竞品对比）

在 AssertJ 成为当前乃至未来的主流之前，开发者通常使用以下工具进行断言。逐一看它们的痛点，才能理解 AssertJ 的设计动机。

## 2.1 传统 JUnit 5 原生断言 (`org.junit.jupiter.api.Assertions`)

JUnit 5 原生断言的三大缺点：

1. **参数顺序容易混淆**：`assertEquals(expected, actual)` 两个参数同类型时，写反了编译器不会报错，只有失败那一刻消息才会"说谎"——你以为 expected 是实际值，排错方向整个反过来。

```java
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;

class OrderAmountTest {

    // 被测: 订单服务返回的实际金额(本例中是 120)
    private int orderTotal() {
        return 120;
    }

    @Test
    void wrong_argument_order() {
        // 错误示范: 把 actual 写在了前面, 类型相同编译通过
        assertEquals(orderTotal(), 100);
        // 失败消息: expected: <120> but was: <100>
        // 读的人会以为"期望是 120", 而真正的契约值是 100, 排错方向被误导
    }

    @Test
    void right_argument_order() {
        // 正确做法: 恒为 (expected, actual)
        assertEquals(100, orderTotal());
    }
}
```

2. **缺乏链式调用**：验证一个复杂对象的五六个属性，就要写五行 `assertEquals`，每行重复对象名，噪音大且首个失败即中断，修一个跑一遍。
3. **提示信息有限**：集合与嵌套对象的对比，`assertEquals` 只会打印两边的 `toString`，上百个元素的 List 差异要人眼找。

## 2.2 Hamcrest (`org.hamcrest.MatcherAssert`)

它引入了 Matcher 模式，把"期望"做成可组合的对象，如 `assertThat(actual, is(equalTo(expected)))`。思路先进，但落地有三个缺点：

1. **API 碎片化**：需要记住大量静态匹配器——`is`、`equalTo`、`hasItem`、`hasSize`、`containsInAnyOrder`……而且它们散落在 `CoreMatchers`、`Matchers`（JUnit 桥接）、自定义 `TypeSafeMatcher` 三处。
2. **缺乏代码提示（IDE 自动补全不友好）**：核心 API 是 `assertThat(T, Matcher<? super T>)`，点出 `.` 之后没有类型引导，全凭记忆。
3. **学习曲线**：语法英语化，但构造复杂逻辑时嵌套严重，一层 `allOf` 套三层 `anyOf`，括号地狱：

```java
// 错误示范(Hamcrest): 判断列表存在两个元素满足组合条件, 嵌套三层
assertThat(employees, containsInAnyOrder(
        allOf(hasProperty("name", is("Alice")), hasProperty("age", greaterThan(60))),
        allOf(hasProperty("name", is("Bob")), hasProperty("age", lessThan(30)))));

// 正确做法(AssertJ): 同一语义, 平铺直叙
// assertThat(employees).extracting("name", "age")
//         .containsExactlyInAnyOrder(tuple("Alice", 65), tuple("Bob", 25));
```

## 2.3 Google Truth

Google 开源的断言库，定位与 AssertJ 高度重合（`assertThat(actual).isEqualTo(expected)` 同样流式），在 Android 与 Guava 系项目里常见。它的强项是与 Android 测试基础设施、proto 类型的集成以及更克制的 API 面积；弱项是对**服务端 Java 类型族的覆盖度**——日期时间、文件/Path、CompletableFuture、Map 的细粒度断言，AssertJ 明显更全；并且 Truth 的自定义断言要求写 `Subject` 子类并处理平台相关的失败描述，门槛比 AssertJ 的 `AbstractAssert` 高一截。**后端 JVM 项目首选 AssertJ，Android 项目选 Truth**，这是社区大体形成的分工。

## 2.4 AssertJ 胜出的原因

综合对比：

| 维度 | JUnit 5 Assertions | Hamcrest | Google Truth | AssertJ |
|------|--------------------|----------|--------------|---------|
| 风格 | 静态方法，参数列表 | Matcher 嵌套 | 流式 | 流式 |
| 编译期防呆（参数写反） | 防不住 | 不适用 | 天然规避 | 天然规避（actual 在入口） |
| IDE 补全引导 | 一般 | 差 | 好 | 极好（按类型给方法） |
| 覆盖的 JDK 类型 | 基本 | 需自行扩展 | 中等 | 极广（含 IO/并发/时间） |
| 失败消息质量 | 中 | 中 | 好 | 极好（expected/actual diff、截断、高亮） |
| 软断言（聚合失败） | assertAll | 无 | 有 | SoftAssertions/assertSoftly |
| 递归深度比较 | 无 | 无 | 无 | usingRecursiveComparison |
| 自定义断言成本 | 无此概念 | TypeSafeMatcher | Subject | AbstractAssert + 代码生成器 |

AssertJ 通过 **流畅型 API (Fluent API)** 解决了以上问题。所有断言都从一个入口开始：`assertThat(actual)`。这个入口做了两件别人没同时做好的事：**actual 只出现一次且位置固定**，从根上消灭了参数写反；**返回值是 actual 类型专属的 Assert 对象**，接下来只需键入 `.`，IDE 就会根据 `actual` 的类型（String、List、Exception……）给出该类型所有可用且绝对安全的断言方法。

# 3. 核心设计哲学

AssertJ 的核心设计理念可以总结为三点，它们分别回答"怎么写、怎么防错、怎么排错"三个问题。

## 3.1 链式流式 API (Fluent API)

让断言读起来像自然语言，一个主题一行读完：

```java
assertThat(frodo.getName())
        .isNotNull()
        .startsWith("Fro")
        .endsWith("do")
        .isEqualTo("Frodo");
```

技术上每个断言方法都返回 `this`（泛型 SELF），链上任意一环失败即抛错终止——所以链式表达的是**同一主题的多个约束**，不是多个独立用例。这与软断言形成互补：独立维度要用软断言聚合（见第 4 篇）。

## 3.2 强类型安全 (Strong Type Safety)

`assertThat("String")` 返回的是 `StringAssert`，只能调用针对字符串的方法（如 `contains`）；`assertThat(List)` 返回 `ListAssert`，只能调用集合的方法（如 `hasSize`）。在编译阶段直接卡住非法调用：

```java
assertThat("hello").hasSize(5);      // 编译错误: 字符串断言上没有集合语义的 hasSize
assertThat(List.of("a")).contains("b"); // 通过编译, contains 是集合专属
```

这种"类型即文档"的设计让 IDE 补全成为可能：你不需要背 API，**按下点号就是当前类型可用的全部断言**。这也是它对 Hamcrest 的代际优势。

## 3.3 极佳的失败提示语

当测试失败时，AssertJ 会清楚打印出 "expected ... but was ..."，且包含差异高亮，极大节约排查时间。集合对比还会指出多余与缺失的元素：

```text
Expecting actual:
  ["Spring", "MyBatis", "Redis"]
to contain:
  ["Spring", "Hibernate"]
but could not find the following element(s):
  ["Hibernate"]
```

> **设计考量**：断言库的第一用户是**半年后凌晨三点看 CI 失败日志的人**，而不是写测试的人。AssertJ 把预算花在失败消息上——结构化 expected/actual（3.23 起走 OpenTest4J，IDE 能弹左右对照 diff）、长输出自动截断、`as("描述")` 标注业务语义——这些都是 JUnit 4 时代断言没有的。选型时，"失败消息可不可读"应该权重最高，API 顺手只是次级理由。

# 4. 环境搭建与依赖引入

## 4.1 Maven 引入

在 `pom.xml` 的 `<dependencies>` 块中加入：

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.27.3</version> <!-- 3.x 最新补丁版, 要求 Java 8+ -->
    <scope>test</scope>
</dependency>
```

> **行业实践**：Spring Boot 项目的 `spring-boot-starter-test` 已默认传递依赖 `assertj-core` 并由 BOM 统一管版本，**不要再手写 `<version>`**，否则 Boot 升级时会出现测试依赖与 starter 版本打架。让 BOM 管理是这类"仅测试期使用的工具库"的通用纪律。

## 4.2 Gradle 引入

```groovy
dependencies {
    // 仅参与测试编译与运行, 不进入生产 classpath
    testImplementation 'org.assertj:assertj-core:3.27.3'
}
```

Groovy DSL 里也可以用 `testImplementation(platform(SpringBoot-dependencies))` 省略版本；Kotlin DSL 对应 `testImplementation("org.assertj:assertj-core")`。

## 4.3 静态导入说明

为了让代码更简洁，使用 AssertJ 必须**静态导入**核心入口类 `Assertions`：

```java
import static org.assertj.core.api.Assertions.assertThat;
// 亦可以静态导入所有的工具以备复杂场景使用(within/tuple/entry/assertThatThrownBy 等)
import static org.assertj.core.api.Assertions.*;
```

单类导入利于 code review 时看清 API 出处，`*` 导入则少维护成本；团队二选一保持统一即可，但**切忌同时 `import static` 两套同名 `assertThat` 的来源**（如 JUnit 4/Hamcrest 的 `Assert.*`、`MatcherAssert.*` 与 AssertJ 的 `Assertions.*`）——JUnit 4 的 `assertThat(T, Matcher)` 与 AssertJ 的 `assertThat(T)` 参数个数不同，混导后单参/双参调用会**静默落到不同库**，失败消息与补全行为完全两样，排查半天发现导错了包。

## 4.4 第一个快速体验

```java
package com.example.assertj;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

public class AssertJFirstTest {

    @Test
    void testBasicAssertion() {
        String result = "Hello, AssertJ";

        // Fluent API 的体现: 一个主题, 一条链, 逐环约束
        assertThat(result)
                .isNotNull()
                .startsWith("Hello")
                .contains("Assert")
                .endsWith("J")
                .hasSize(14);
    }

    @Test
    void testCollectionAssertion() {
        // 集合入口返回 ListAssert, 补全列表里自然只出现集合语义
        assertThat(java.util.List.of("Spring", "MyBatis", "Redis"))
                .hasSize(3)
                .contains("MyBatis")
                .doesNotContain("Hibernate");
    }
}
```

跑通这两个用例后，把 IDE 里某个断言故意改错（如 `endsWith("A")`），观察失败消息的形态——**先学会读失败输出，再学写断言**，这是用好任何断言库的正确顺序。

# 5. assertThat 家族入口速查

`Assertions` 是一个巨大的门面，入口方法按用途分四族，后面各篇逐一展开：

| 入口族 | 代表方法 | 用途 |
|--------|----------|------|
| 类型断言入口 | `assertThat(String/List/Map/Optional/Path/...)` | 数百个重载，按 actual 类型返回专属 Assert |
| 异常断言入口 | `assertThatThrownBy`、`assertThatExceptionOfType`、`assertThatCode`、`catchThrowable(OfType)` | 见第 3 篇 |
| 软断言入口 | `SoftAssertions.assertSoftly`、`new SoftAssertions()`、JUnit 5 的 `SoftAssertionsExtension` | 见第 4 篇 |
| 前提假设入口 | `Assumptions.assumeThat(...)` | 不满足则**跳过**测试而非失败，对应 JUnit 的 assumption |

除入口外还有常用**静态辅助**，全部来自 `org.assertj.core.api.Assertions`：

```java
import static org.assertj.core.api.Assertions.byLessThan;   // 数值容差(Offset)
import static org.assertj.core.api.Assertions.within;       // 同上, 语义更好读
import static org.assertj.core.api.Assertions.tuple;        // extracting 多属性结果
import static org.assertj.core.api.Assertions.entry;        // Map 键值对断言单元
import static org.assertj.core.api.Assertions.allOf;        // Condition 组合(与)
import static org.assertj.core.api.Assertions.anyOf;        // Condition 组合(或)
import static org.assertj.core.api.Assertions.not;          // Condition 取反
```

> **注意**：`assumeThat` 的失败不是测试失败，而是 ABORTED 状态；把它误用在"必须成立"的校验上，CI 会把本该红灯的用例标成灰色跳过。**契约用 assertThat，环境前提才用 assumeThat**，与 JUnit `assumeTrue` 的纪律一致。

# 6. 小结

AssertJ 在测试栈中专职"断言层"，以 `assertThat(actual)` 为唯一入口，凭**强类型 + 流式链 + 结构化失败消息**三板斧，解决了 JUnit 原生断言的参数顺序陷阱、Hamcrest 的补全与嵌套问题，成为服务端 Java 事实上的断言标准。落地要点：跟随 Boot BOM 引依赖、统一静态导入纪律、认真读失败消息。后续七篇依次展开核心断言族（第 2 篇）、异常断言（第 3 篇）、软断言（第 4 篇）、自定义断言（第 5 篇）、高级特性（第 6 篇）、源码原理（第 7 篇）与最佳实践/面试（第 8 篇）。
