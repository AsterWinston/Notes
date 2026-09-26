
# 1. 什么是 AssertJ

AssertJ 是一个用于编写强类型、流畅、高可读性测试断言的 Java 库。它为标准 Java 类型（如集合、字符串、异常、Optional、日期等）提供了极其丰富且直观的 API。

在 Java 单元测试生态中，通常需要配合测试执行框架（如 JUnit 4/5 或 TestNG）使用。测试框架负责“调度和执行”测试用例，而 AssertJ 则专门负责“验证结果是否符合预期”。

# 2. 为什么选择 AssertJ？（与竞品对比）

在 AssertJ 成为当前乃至未来的主流之前，开发者通常使用以下工具进行断言：

## 2.1 传统 JUnit 5 原生断言 (`org.junit.jupiter.api.Assertions`)
**缺点**：
1. **参数顺序容易混淆**：`assertEquals(expected, actual)` 和 `assertEquals(actual, expected)` 在编译期无法区分，容易写反，导致报错提示相反。
2. **缺乏链式调用**：对于复杂对象的多个属性，需要写多行晦涩的代码。
3. **提示信息有限**：默认的失败提示不够直观，尤其是集合与复杂对象对比。

## 2.2 Hamcrest (`org.hamcrest.MatcherAssert`)
它引入了 Matcher 模式，如 `assertThat(actual, is(equalTo(expected)))`。
**缺点**：
1. **API 碎片化**：需要记住大量的 `CoreMatchers`，如 `hasItem`, `is`, `equalTo` 等。
2. **缺乏代码提示（IDE 自动补全不友好）**：由于核心逻辑依赖于泛型和静态方法组合，在 IDE 里打出 `.` 之后无法很好的给出提示。
3. **学习曲线**：语法比较英语化，但构造复杂逻辑时嵌套严重（如 `allOf`, `anyOf`）。

## 2.3 AssertJ 胜出的原因
AssertJ 通过 **流畅型 API (Fluent API)** 解决了以上问题。
在 AssertJ 中，所有断言都从一个入口开始：`assertThat(actual)`，接下来你只需键入 `.`，IDE 就会根据 `actual` 的类型（String, List, Exception 等）给出该类型所有可用且绝对安全的断言方法。

---

# 3. 核心设计哲学

AssertJ 的核心设计理念可以总结为三点：

1. **链式流式 API (Fluent API)**：
   让断言读起来像自然语言。
   例如：`assertThat(frodo.getName()).startsWith("Fro").endsWith("do").isEqualTo("Frodo");`
2. **强类型安全 (Strong Type Safety)**：
   `assertThat("String")` 返回的是 `StringAssert`，只能调用针对字符串的方法（如 `contains`）；`assertThat(List)` 返回 `ListAssert`，只能调用集合的方法（如 `hasSize`）。在编译阶段直接卡住非法调用。
3. **极佳的失败提示语**：
   当测试失败时，AssertJ 会清楚打印出 "Expect... but was..."，且包含差异高亮，极大地节约了排查问题的时间。

---

# 4. 环境搭建与依赖引入

## 4.1 Maven 引入

在 `pom.xml` 的 `<dependencies>` 块中加入：

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.24.2</version> <!-- 推荐使用 3.x 最新版本，要求 Java 8+ -->
    <scope>test</scope>
</dependency>
```

> **提示**：如果使用的是包含 `spring-boot-starter-test` 的 Spring Boot 项目，通常已经默认包含了 `assertj-core` 的依赖，无需再单独声明版本。

## 4.2 静态导入说明

为了让代码更简洁，使用 AssertJ 必须**静态导入**核心入口类 `Assertions`：

```java
import static org.assertj.core.api.Assertions.assertThat;
// 亦可以静态导入所有的工具以备复杂场景使用
import static org.assertj.core.api.Assertions.*; 
```

## 4.3 第一个快速体验

```java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

public class AssertJFirstTest {

    @Test
    void testBasicAssertion() {
        String result = "Hello, AssertJ";

        // Fluent API 的体现
        assertThat(result)
            .isNotNull()
            .startsWith("Hello")
            .contains("Assert")
            .endsWith("J");
    }
}
```