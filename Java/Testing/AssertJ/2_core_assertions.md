# 1. 基础类型断言

AssertJ 之所以被称为"开发者的瑞士军刀"，很大程度上归功于其对 Java 原生类型提供了非常细粒度且语义化的 API。本篇系统剖析各断言族的常用方法与语义边界，以及集合的高级玩法。所有入口都来自 `org.assertj.core.api.Assertions` 的静态导入，方法返回值都是当前类型的 Assert 对象，可以无限续链。

## 1.1 通用对象 (Object) 断言

适用于所有 Java 对象，核心方法用于校验相等性、为空性和类型。其中**最容易用错的是 isEqualTo 与 isSameAs 的区别**——前者走 `equals()`，后者走 `==` 引用比较：

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ObjectAssertionTest {

    @Test
    void equality_vs_identity() {
        Object obj = new Object();
        Object sameObj = obj;                 // 同一个引用
        Object equalObj = "abc";
        Object otherRef = new String("abc");  // 不同引用, equals 相同

        assertThat(obj).isNotNull();                   // 确保不为空
        assertThat(obj).isSameAs(sameObj);             // == 引用比较(内存地址一致)
        assertThat(equalObj).isEqualTo(otherRef);      // equals() 比较, 内容一致即通过
        // 错误示范: 想验证缓存返回同一实例, 却写了 isEqualTo
        // assertThat(cache.get(1L)).isEqualTo(instance); // 只要 equals 相同就漏过"返回了新对象"的 bug
        // 正确做法: 单例/缓存场景验证的是引用同一性
        assertThat(sameObj).isSameAs(obj);

        assertThat(obj).isInstanceOf(Object.class);          // 类型检查, 子类也算
        assertThat(obj).isExactlyInstanceOf(Object.class);   // 必须精确类, 子类不行
        assertThat(obj).isOfAnyClassIn(String.class, Object.class); // 多类型白名单
        assertThat(obj).hasSameClassAs(sameObj);             // 与另一对象的 Class 相同
        assertThat(obj).isIn(obj, new Object(), new Object()); // 在给定候选中(按 equals)
        assertThat(obj).isNotIn(java.util.List.of("x"));       // 不在候选中
    }
}
```

**isEqualTo 的陷阱在于被测类型的 equals 实现**。值对象没重写 equals 时断言永远失败，且失败消息里两个对象 `toString` 输出一模一样——排查顺序：先确认 equals/hashCode，再怀疑断言本身。对于"不重写 equals 但想按字段比"的场景，不要改用 `hasFieldOrPropertyWithValue` 一个个贴，直接上 `usingRecursiveComparison()`（见第 6 篇）。

## 1.2 字符串 (String) 断言

字符串断言在 Web 开发（JSON 报文、日志、SQL 语句断言）中极为常用，AssertJ 提供了非常丰富的语义化 API：

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class StringAssertionTest {

    @Test
    void string_assertions() {
        String text = "Spring Boot 3.x is awesome!";

        assertThat(text)
                .isNotBlank()                          // 非 null、非空串、不只有空白
                .startsWith("Spring")                  // 前缀校验
                .endsWith("awesome!")                  // 后缀校验
                .contains("Boot", "3.x")               // 包含全部指定子串
                .containsOnlyOnce("Boot")              // 恰好出现一次
                .doesNotContain("2.x")                 // 不包含子串
                .containsIgnoringCase("spring boot")   // 忽略大小写包含
                .matches(".*Boot \\d\\.x.*")           // 整串正则匹配
                .containsPattern("(?i)awesome")        // 部分匹配正则(比 matches 常用)
                .hasSize(27)                           // 长度
                .hasLineCount(1)                       // 行数(多行文本很有用)
                .isSubstringOf("Look! Spring Boot 3.x is awesome! Yeah") // 是谁的子串
                .isEqualToNormalizingWhitespace("Spring  Boot 3.x is awesome!"); // 空白归一后相等
    }
}
```

> **注意**：`isEqualTo` 对字符串是**逐字符严格比较**，换行符、行尾空格、不可见 Unicode 字符都会造成"肉眼一样但断言失败"。比较 JSON/XML/SQL 这类格式化文本时，优先 `isEqualToNormalizingWhitespace` 或 `isEqualToNormalizingNewlines`，再不行才考虑解析成对象后 `usingRecursiveComparison`，不要为了过断言去 trim 被测产物。

## 1.3 数字 (Numbers) 与可比较对象 (Comparable)

任何实现了 `Comparable` 接口的数据（`int`、`Double`、`BigDecimal`、`Date` 等）都可以做大小与区间断言：

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class NumberAssertionTest {

    @Test
    void comparable_assertions() {
        int age = 18;

        assertThat(age)
                .isPositive()                 // > 0
                .isNotZero()
                .isGreaterThan(17)            // > 17
                .isGreaterThanOrEqualTo(18)   // >= 18
                .isLessThan(100)
                .isBetween(10, 20)            // 10 <= age <= 20, 闭区间
                .isCloseTo(20, within(2))     // age 落在 [18, 22], 容差思想
                .isNotIn(1, 2, 3);

        // BigDecimal 专用: isZero/isNegative/isNotNegative, 语义化更直接
        assertThat(new java.math.BigDecimal("0.00")).isZero();
    }
}
```

**浮点数比较必须用容差而不是 isEqualTo**。由于 IEEE 754 精度损失，`0.1 + 0.2` 不等于 `0.3`：

```java
// 错误示范: 永远失败的断言
// assertThat(0.1 + 0.2).isEqualTo(0.3);

// 正确做法: 显式给出可接受的误差
assertThat(0.1 + 0.2).isCloseTo(0.3, within(0.0001));
// withinPercentage 表达相对容差, 金额类断言更稳
assertThat(1.005 * 100).isCloseTo(100.5, org.assertj.core.data.Percentage.withPercentage(0.01));
```

同族的 `Date`/`Instant`/`LocalDateTime` 断言在第 6 篇时间部分展开，核心思想一致：**时间断言永远带容差或对齐单位**，不要 `isEqualTo` 裸比微秒。

# 2. 集合、数组与 Map 断言

AssertJ 对集合框架的支持几乎把常见的流式校验都收进了一个断言语句内，无需在外部手写 `forEach` 循环或借助 Stream API 收集。

## 2.1 包含语义家族：顺序敏感与否是核心分界

集合断言的重头戏是**几个 contains 系列方法的语义差异**，混用会写出"碰巧通过"或"碰巧失败"的测试：

| 方法 | 顺序 | 多余元素 | 重复元素 | 一句话语义 |
|------|------|----------|----------|------------|
| `contains(a, b)` | 不敏感 | 允许 | 允许 | 至少有这些 |
| `containsExactly(a, b)` | **敏感** | 不允许 | 不允许 | 精确等于这个序列 |
| `containsExactlyInAnyOrder(a, b)` | 不敏感 | 不允许 | 不允许 | 精确等于这个多重集合 |
| `containsOnly(a, b)` | 不敏感 | 不允许 | 允许 | 只有这些（不关心个数） |
| `containsSequence(a, b)` | 敏感且**相邻** | 允许 | 允许 | 中间的一段连续子序列 |
| `containsSubsequence(a, b)` | 敏感**不要求相邻** | 允许 | 允许 | 相对先后顺序正确 |

```java
import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class CollectionAssertionTest {

    @Test
    void containment_semantics() {
        List<String> frameworkList = Arrays.asList("Spring", "MyBatis", "Redis");

        assertThat(frameworkList)
                .isNotEmpty()
                .hasSize(3)
                .doesNotHaveDuplicates();                    // 无重复
        assertThat(frameworkList).allSatisfy(f -> assertThat(f).isNotBlank()); // 每个元素都满足
        assertThat(frameworkList).anyMatch(f -> f.startsWith("Re"));           // 至少一个满足
        assertThat(frameworkList).noneMatch(f -> f.equals("Hibernate"));       // 一个都没有

        assertThat(frameworkList).contains("MyBatis");                          // 必须包含
        assertThat(frameworkList).containsExactly("Spring", "MyBatis", "Redis"); // 顺序必须绝对一致
        assertThat(frameworkList).containsExactlyInAnyOrder("Redis", "Spring", "MyBatis"); // 忽略顺序
        assertThat(frameworkList).containsOnly("Spring", "MyBatis", "Redis");    // 只能是这些, 个数不管
        assertThat(frameworkList).containsSequence("MyBatis", "Redis");          // 相邻子序列
        assertThat(frameworkList).containsSubsequence("Spring", "Redis");        // 只要求相对顺序

        // 错误示范: 查询接口本不保证顺序, 用 containsExactly 会因数据库返回顺序抖动而偶发红
        // assertThat(userDao.findByName("a")).containsExactly(u1, u2, u3);
        // 正确做法: 顺序无契约时一律 containsExactlyInAnyOrder; 有 ORDER BY 契约才用 containsExactly
        assertThat(frameworkList).containsExactlyInAnyOrder("Redis", "MyBatis", "Spring");
    }
}
```

> **设计考量**：**containsExactly 的顺序敏感是特性不是缺陷**——它把"顺序本身是契约"（分页排序、优先级队列、编译产物行序）显式写进测试。反过来说，对无顺序契约的查询结果用它，就是在给 CI 埋随机失败的雷。判断标准只有一条：**这个顺序是被测需求承诺的，还是实现碰巧给的？**

`startsWith`/`endsWith` 对 List 同样可用（断言头尾元素），`hasSizeBetween`、`extractingResultOf` 等可与上面任意链组合。数组走 `assertThat(int[])` 返回 `ArrayAssert`，方法与 List 基本同构，不再单列。

## 2.2 元素级条件：allSatisfy / filteredOn / 原始 isTrue 的反面

对"每个元素满足某复杂条件"，有三种写法，可读性与失败信息质量递减：

```java
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ElementConditionTest {

    record Account(String owner, long balance) {}

    @Test
    void element_conditions() {
        List<Account> accounts = List.of(new Account("a", 10), new Account("b", 20));

        // 推荐: 失败时精确指出是第几个元素、违反了哪条子断言
        assertThat(accounts).allSatisfy(acc -> {
            assertThat(acc.owner()).isNotBlank();
            assertThat(acc.balance()).isPositive();
        });

        // 数据驱动: filteredOn 谓词筛出满足条件的元素再整体断言
        assertThat(accounts).filteredOn(acc -> acc.balance() > 15)
                            .extracting(Account::owner)
                            .containsExactly("b");

        // 不推荐: 把逻辑塞进一个布尔表达式, 失败消息只有 "Expecting actual to be true"
        assertThat(accounts.stream().allMatch(acc -> acc.balance() > 0)).isTrue();
    }
}
```

**`isTrue` 是可读性的黑洞**。`assertThat(x.stream().anyMatch(...)).isTrue()` 把整个业务语义折叠成一个 boolean，失败时既看不到期望也看不到实际。AssertJ 保留 `isTrue` 是给它"本来就是布尔字段"的正当场景（开关位、状态标志），而不是让它当集合量词的替身——`anyMatch/noneMatch/allMatch` 在集合 Assert 上都有同名断言，永远优先用。

## 2.3 Map 断言

键值结构的断言核心是 `entry(...)` 辅助器：

```java
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.entry;

class MapAssertionTest {

    @Test
    void map_assertions() {
        Map<String, Object> config = new HashMap<>();
        config.put("timeout", 5000);
        config.put("retry", true);

        assertThat(config)
                .isNotEmpty()
                .hasSize(2)
                .containsKeys("timeout", "retry")
                .containsValue(5000)
                .containsEntry("retry", true)
                .containsOnly(entry("timeout", 5000), entry("retry", true)) // 精确整表校验
                .doesNotContainKey("username");

        // 值本身是集合时, extracting 可以继续下钻
        Map<String, List<String>> roles = Map.of("admin", List.of("r1", "r2"));
        assertThat(roles).extractingByKey("admin")
                         .asInstanceOf(org.assertj.core.api.InstanceOfAssertFactories.LIST)
                         .hasSize(2);
    }
}
```

> **注意**：`containsEntry` 的 value 比较同样走 `equals`；不可变 `List.of(...)` 与 `ArrayList` 之间 equals 是内容语义没问题，但**自定义 value 类型忘写 equals 时整表断言必挂**，报错形态与 1.1 节的对象断言一致。

# 3. `extracting` 属性级联验证大法 （高频使用）

测试中拿到的往往是一组 `UserDTO`，但只想验证它们的 `name` 或 `age` 是否符合预期。传统做法是先手写 Stream 提取成 `List<String>` 再断言；AssertJ 把提取这一步内建进了断言链。

## 3.1 字符串属性提取与 tuple 多属性

```java
import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.tuple;

class ExtractingTest {

    static class User {
        private final String name;
        private final int age;
        User(String name, int age) { this.name = name; this.age = age; }
        public String getName() { return name; }
        public int getAge() { return age; }
    }

    @Test
    void extract_single_and_multiple_fields() {
        List<User> users = Arrays.asList(
                new User("Alice", 25),
                new User("Bob", 30),
                new User("Charlie", 35));

        // 按属性名提取(反射读取 getter 或 public 字段), 得到提取后的 ListAssert
        assertThat(users)
                .extracting("name")
                .containsExactly("Alice", "Bob", "Charlie");

        // 多属性一次校验, 每个元素被打包成 Tuple
        assertThat(users)
                .extracting("name", "age")
                .containsExactly(
                        tuple("Alice", 25),
                        tuple("Bob", 30),
                        tuple("Charlie", 35));
    }
}
```

## 3.2 类型安全的 `extracting`

字符串 `"name"` 靠反射解析，**字段重构改名后要到运行期才炸**。AssertJ 支持方法引用提取，编译期即可发现错误，是团队项目的默认选择：

```java
assertThat(users)
        .extracting(User::getName)          // 编译期检查, 重构安全, 非常推荐
        .containsExactly("Alice", "Bob", "Charlie");

// 嵌套下钻: 先提取子对象列表, 再对子对象继续 extracting 其属性
assertThat(orders).extracting(Order::getCustomer).extracting(Customer::getCity)
                  .contains("Shanghai");
```

与 `usingFieldByFieldElementComparator` 的关系要分清：**extracting 是"投影"**——把元素映射成新列表，后续断言作用在投影结果上；**字段比较器是"换比较器"**——元素仍是原对象，只是 `isEqualTo/contains` 改为逐字段 equals：

```java
@Test
void field_by_field_comparison() {
    List<User> expected = List.of(new User("Alice", 25));
    // 运行时代理出的实例没有重写 equals, isEqualTo 必挂
    // assertThat(service.query()).isEqualTo(expected);     // 错误示范
    assertThat(service.query())
            .usingFieldByFieldElementComparator()          // 元素按字段递归比较
            .isEqualTo(expected);
}
```

代价是失败消息只报"元素不匹配"笼统信息，丢失了 extracting 那种"哪个字段错"的定位精度。**优先 extracting（报得准），对象图复杂到投影写不动时才换字段比较器。**

# 4. 失败信息的结构与调优

理解失败消息的组成，才能在 CI 日志里三秒钟定位问题。有 `as()` 前缀时，失败输出恒为"前缀行 + expected 行 + but was 行"的结构：

```text
org.opentest4j.AssertionFailedError:
[Twitter 昵称]                                   ← as("...") 添加的语义前缀(可无)
expected: "kdannybusch"                          ← 期望值
 but was: "dany"                                 ← 实际值
```

三个常用调节手段：

```java
// 1. as: 给链加业务前缀, 软断言多链聚合时必加
assertThat(user.getNickname()).as("Twitter 昵称").isEqualTo("kdannybusch");

// 2. describedAs: 前缀会覆盖继承链, 行为同 as
assertThat(user.getAge()).describedAs("年龄必须已成年").isGreaterThanOrEqualTo(18);

// 3. withFailMessage(或 overridingErrorMessage): 完全接管消息文本(慎用, 丢失 expected/actual 结构)
assertThat(token).withFailMessage("令牌过期, 请检查时钟偏移").isEqualTo(expected);
```

`Assertions.useRepresentation(...)` 全局更换表示器、`useDefaultRepresentation()` 复位，这两个静态方法在 3.x 的 `AbstractAssert`/`Assertions` 上都有对应物，第 6 篇 withRepresentation 一节继续。

# 5. 小结

核心断言篇的要点：通用对象族里 **isEqualTo（equals 语义）与 isSameAs（== 语义）** 的选择取决于契约是内容还是身份；字符串断言优先语义化方法而非 matches 大正则；数值与时间断言**永远带容差**；集合断言的关键是认清 contains 家族的顺序/多余/重复三维语义，顺序无契约就不许写 containsExactly；**extracting 投影优于原始 isTrue**，方法引用提取优于反射字符串，失败消息的可读性应该作为断言写法的首要评审标准。这些 API 在源码层全部收敛到 `AbstractAssert<SELF, ACTUAL>` 的泛型设计，原理见第 7 篇。
