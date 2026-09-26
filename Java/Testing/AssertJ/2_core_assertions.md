
AssertJ 之所以被称为"开发者的瑞士军刀"，很大程度上归功于其对 Java 原生类型提供了非常细粒度且语义化的 API。本篇将系统剖析常见类型的断言方法，以及集合的高级玩法。

---

# 1. 基础类型断言

## 1.1 通用对象 (Object) 断言

适用于所有的 Java 对象。核心方法用于校验相等性、为空性和类型。

```java
Object obj = new Object();
Object sameObj = obj;

assertThat(obj)
    .isNotNull()             // 确保不为空
    .isEqualTo(sameObj)      // equals() 比较
    .isSameAs(sameObj)       // == 引用比较（内存地址一致）
    .isInstanceOf(Object.class) // 类型检查
    .isExactlyInstanceOf(Object.class) // 不能是子类，必须是精确类
    .hasSameClassAs(sameObj) // 获取其 Class 对象做对比
    .isIn(obj, new Object(), new Object()); // 在给定范围中
```

## 1.2 字符串 (String) 断言

字符串断言在 Web 开发（如 JSON 断言、日志信息断言）中极为常用，AssertJ 提供了非常丰富的语义化 API。

```java
String text = "Spring Boot 3.x is awesome!";

assertThat(text)
    .isNotBlank()                         // 不允许为空且不能只含空格
    .startsWith("Spring")                 // 前缀校验
    .endsWith("awesome!")                 // 后缀校验
    .contains("Boot", "3.x")              // 包含指定子串
    .doesNotContain("2.x")                // 不包含子串
    .containsIgnoringCase("spring boot")  // 忽略大小写包含
    .matches(".*Boot \\d\\.x.*")          // 正则表达式匹配
    .hasSize(27)                          // 检查长度
    .isSubstringOf("Look! Spring Boot 3.x is awesome! Yeah"); // 是谁的子串
```

## 1.3 数字 (Numbers) 与可比较对象 (Comparable)

任何实现了 `Comparable` 接口的数据（包含 `int`, `Double`, `BigDecimal` 及 `Date` 等）都可以进行大小断言。

```java
int age = 18;

assertThat(age)
    .isPositive()              // > 0
    .isGreaterThan(17)         // > 17
    .isGreaterThanOrEqualTo(18)// >= 18
    .isBetween(10, 20)         // 10 <= age <= 20
    .isCloseTo(20, within(2)); // age 在区间 [20-2, 20+2] 内，对浮点数尤其有用
```

> **注意点：浮点数比较机制**
> 由于浮点数的精度损失，`isEqualTo()` 在处理 double/float 时可能不稳定，必须使用精度容差（Offset）：
> ```java
> assertThat(0.1 + 0.2).isCloseTo(0.3, within(0.0001));
> ```

---

# 2. 集合与数组高级 API (Collections/Arrays)

AssertJ 对集合框架的支持非常逆天，几乎把常见的流式校验都包含在了一个断言语句内，无需我们在外部手写 `forEach` 循环或者借助 Stream API。

## 2.1 基本包含与大小断言

```java
List<String> frameworkList = Arrays.asList("Spring", "MyBatis", "Redis");

assertThat(frameworkList)
    .isNotEmpty()
    .hasSize(3)
    .contains("MyBatis")                   // 必须包含此元素
    .containsExactly("Spring", "MyBatis", "Redis") // 包含且顺序必须绝对一致
    .containsExactlyInAnyOrder("Redis", "Spring", "MyBatis") // 包含但忽略顺序
    .containsOnly("Spring", "MyBatis", "Redis") // 包含这些，且只能是这些（忽略由于重复导致的个数差异带来的报错）
    .doesNotContain("Hibernate");           // 绝对不能有
```

---

# 3. `extracting` 属性级联验证大法 (高频使用)

测试中，我们往往拿到的是一组 `UserDTO` 对象，但我们只想验证这些 DTO 里的 `userId` 或者是 `name` 是否符合我们的预期集合。传统做法是，我们先提取出 `List<Long> ids` 然后断言，而 AssertJ 提供了提取器。

```java
class User {
    String name;
    int age;
    // ... getter setters 及构造函数省略
}

List<User> users = Arrays.asList(
    new User("Alice", 25),
    new User("Bob", 30),
    new User("Charlie", 35)
);

// 魔法特性：直接提取 users 里所有对象的 name 属性建立新 List 并比对！
assertThat(users)
    .extracting("name") // 根据反射读取 public field 或者是 getName()
    .containsExactly("Alice", "Bob", "Charlie");

// 也可以基于多个属性一次性校验出 Tuple5!
import static org.assertj.core.api.Assertions.tuple;

assertThat(users)
    .extracting("name", "age") // 取出 name 和 age 的组合
    .containsExactly(
        tuple("Alice", 25),
        tuple("Bob", 30),
        tuple("Charlie", 35)
    );
```

## 3.1 类型安全的 `extracting`

使用字符串如 `"name"` 反射提取会在重构字段被修改时失败，AssertJ 支持强类型的 Function 提取：

```java
assertThat(users)
    .extracting(User::getName)   // 编译期即可发现错误，非常推荐！
    .containsExactly("Alice", "Bob", "Charlie");
```

---

# 4. Map 对象断言

处理键值对结构也非常方便。

```java
Map<String, Object> config = new HashMap<>();
config.put("timeout", 5000);
config.put("retry", true);

assertThat(config)
    .isNotEmpty()
    .hasSize(2)
    .containsKey("timeout")
    .containsEntry("retry", true)
    .doesNotContainKey("username");
```