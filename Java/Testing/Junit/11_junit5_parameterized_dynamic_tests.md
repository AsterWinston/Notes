# 1. 参数化测试总览

参数化测试解决的是"**同一套断言逻辑，跑在一批不同输入上**"的问题。JUnit 4 时代要靠 `@RunWith(Parameterized.class)` 加一个静态数据方法实现，整个测试类只能服务于一个参数化场景；JUnit 5 把它变成了 `junit-jupiter-params` 模块里的一个普通注解 `@ParameterizedTest`，**同一个测试类里可以混排普通 `@Test` 和多个不同数据源的参数化测试**，这是两代 API 最本质的差别。

## 1.1 依赖与基本结构

参数化能力不在 `junit-jupiter-api` 里，需要单独引入 `junit-jupiter-params`（引入聚合包 `junit-jupiter` 时已包含）：

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

一个最小的参数化测试：

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.assertTrue;

class NumberTest {

    @ParameterizedTest                  // 替代 @Test，必须搭配至少一个数据源注解
    @ValueSource(ints = {1, 2, 3})      // 数据源：三个 int 值，跑三次
    void is_positive(int n) {
        assertTrue(n > 0);
    }
}
```

**`@ParameterizedTest` 方法必须声明参数**，且参数要么能从数据源直接消费，要么有对应的转换器/聚合器，否则运行时抛 `JUnitException`。每次调用在报告中是一个独立的"调用（invocation）"，失败互不影响，报告里显示为 `[1] 1`、`[2] 2` 这样的索引加参数值。

## 1.2 自定义显示名

默认显示名是 `[index] argument0, argument1`，可读性一般。用 `name` 属性配合占位符定制：

```java
@ParameterizedTest(name = "[{index}] {0} 岁 {1} 应判定为 {2}")
@CsvSource({"17, alice, 未成年", "18, bob, 成年"})
void check_adult(int age, String name_, String expect) { ... }

// 也可以用 Arguments.displayName 单独指定某条数据的显示名
static Stream<Arguments> cases() {
    return Stream.of(Arguments.of("边界值", 18, true));
}
```

可用占位符：`{index}` 调用序号、`{0}`/`{1}` 第 N 个参数、`{displayName}` 方法显示名。显示名值得认真对待——**参数化测试失败时，你第一眼看到的就是它**，`[3] 18, true` 远不如 `[3] 边界值：18 岁应为成年` 直观。

# 2. 数据源注解详解

数据源是参数化测试的核心，内置六种注解覆盖了绝大多数场景。

| 数据源注解 | 数据形态 | 典型场景 |
|-----------|---------|---------|
| `@ValueSource` | 单参数的字面量数组 | 一组 int/String/Class 快速枚举 |
| `@CsvSource` | 内联 CSV 字符串，多参数 | 少量多列数据，写在注解里 |
| `@CsvFileSource` | classpath 或文件系统的 CSV 文件 | 大批量表格数据，由测试/产品同学维护 |
| `@EnumSource` | 枚举全部或部分常量 | 遍历枚举做契约测试 |
| `@MethodSource` | 静态工厂方法返回 `Stream<Arguments>` | 复杂对象、需要代码构造的数据 |
| `@ArgumentsSource` | 自定义 `ArgumentsProvider` 类 | 数据源本身要复用到多个测试类 |

## 2.1 @ValueSource

只支持**单参数**方法，支持 `shorts/bytes/ints/longs/floats/doubles/chars/booleans/strings/classes` 十种属性：

```java
@ParameterizedTest
@ValueSource(strings = {"racecar", "level", "madam"})
void palindrome(String word) {
    assertTrue(isPalindrome(word));
}
```

## 2.2 @CsvSource 与 @CsvFileSource

CSV 数据源支持多参数，按逗号切分后逐个做类型转换。`@CsvSource` 写在注解里，`@CsvFileSource` 从文件加载：

```java
@ParameterizedTest
@CsvSource({
    "alice, 18",           // 每行一条数据，逗号分隔
    "bob,   20"
})
void parse_user(String name, int age) {
    assertNotNull(name);
    assertTrue(age > 0);
}

@ParameterizedTest
@CsvFileSource(
    resources = "/users.csv",       // 从 classpath 读取
    numLinesToSkip = 1,             // 跳过表头行
    delimiter = ';'                 // 自定义分隔符（默认逗号）
)
void load_users(String name, int age, String city) { ... }
```

`@CsvSource` 还有几个实用属性：`nullValues = {"NIL"}` 把指定字面量转成 `null`、`emptyValue = ""` 控制空串行为、`ignoreLeadingAndTrailingWhitespace = true`（默认）忽略首尾空白。**CSV 里的值默认都按字符串处理再隐式转换**，含逗号的字段要用单引号包裹（`"'hello, world', 1"`）。

## 2.3 @MethodSource

数据需要代码构造时（复杂对象、依赖计算的值）用 `@MethodSource` 指向一个返回 `Stream<Arguments>` 的工厂方法：

```java
@ParameterizedTest
@MethodSource("userProvider")          // 省略 value 时按"同名静态方法"约定查找
void create_user(User user, boolean expectValid) {
    assertEquals(expectValid, validator.isValid(user));
}

static Stream<Arguments> userProvider() {
    return Stream.of(
        Arguments.of(new User("alice", 18), true),
        Arguments.of(new User("", 18), false),           // 空名字非法
        Arguments.of(new User("bob", -1), false)         // 负年龄非法
    );
}
```

工厂方法**默认必须是 static**，因为参数在测试实例创建之前就要解析。JUnit 5.7 起如果测试类标注了 `@TestInstance(Lifecycle.PER_CLASS)`，工厂方法可以是实例方法。跨类引用写成 `@MethodSource("com.example.Users#provider")`。注意 `Stream<Arguments>` 是惰性消费的，但 JUnit 会先物化再统计数量，不要在工厂里产生无限流。

## 2.4 @EnumSource

遍历枚举常量，可用 `mode` 和 `names` 做筛选：

```java
@ParameterizedTest
@EnumSource(TimeUnit.class)                       // 全部常量
void to_nanos_not_negative(TimeUnit unit) {
    assertTrue(unit.toNanos(1) > 0);
}

@ParameterizedTest
@EnumSource(
    value = OrderStatus.class,
    mode = EnumSource.Mode.EXCLUDE,               // INCLUDE（默认）/EXCLUDE/MATCHES_ALL
    names = {"PAID", "SHIPPED"}                   // EXCLUDE：排除这两个终态
)
void cancelable(OrderStatus status) {
    assertTrue(status.isCancelable());
}
```

`names` 支持正则（如 `names = "^FAILED_.*"`），配合 `MATCHES_ALL` 使用。**新增枚举常量时 `@EnumSource` 默认全量的用例会立刻覆盖到它**，这是它对枚举契约测试特别有价值的原因——不用记得去补数据。

## 2.5 @ArgumentsSource 与自定义 ArgumentsProvider

当同一份数据要喂给多个测试类，或者数据来自数据库/外部系统时，实现 `ArgumentsProvider`：

```java
import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.ArgumentsProvider;
import java.util.stream.Stream;

public class UserArgumentsProvider implements ArgumentsProvider {
    @Override
    public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
        // context 可拿到测试类/方法信息，按需生成数据
        return Stream.of(
            Arguments.of("alice", 18),
            Arguments.of("bob", 20)
        );
    }
}

// 使用
@ParameterizedTest
@ArgumentsSource(UserArgumentsProvider.class)
void parse_user(String name, int age) { ... }
```

> **行业实践**：内置六种数据源足够覆盖 95% 的场景，`ArgumentsProvider` 只在数据源需要跨类复用或来自外部系统时才值得写。多数团队更倾向用 `@MethodSource` 指向一个集中的 TestDataFactory 静态方法，可读性更好。

# 3. 类型转换与参数聚合

## 3.1 隐式转换

CSV 等数据源给出的都是字符串，JUnit 会自动做**隐式转换**：目标是基本类型/包装类、`String`、枚举（按 `valueOf`）、`java.time` 的常用类型（`LocalDate`、`Instant` 等，按 ISO 格式）、以及**实现了 `ArgumentConverter` 或带单参 `String` 构造器/静态 `valueOf`、`of`、`from` 工厂方法的类型**。这意味着自定义值对象常常不需要写任何转换代码：

```java
class Money {
    private Money(String text) { ... }   // 有单参 String 构造器即可被隐式转换
}

@ParameterizedTest
@CsvSource({"CNY 100.00, USD 14.50"})
void convert(Money from, Money to) { ... }
```

## 3.2 @ConvertWith 显式转换

隐式规则不够用时，实现 `ArgumentConverter` 并用 `@ConvertWith` 挂到参数上：

```java
class StringToUserConverter implements SimpleArgumentConverter {
    @Override
    protected Object convert(Object source, Class<?> targetType) {
        String[] parts = ((String) source).split(":");   // 约定格式 "name:age"
        return new User(parts[0], Integer.parseInt(parts[1]));
    }
}

@ParameterizedTest
@ValueSource(strings = {"alice:18", "bob:20"})
void login(@ConvertWith(StringToUserConverter.class) User user) { ... }
```

## 3.3 @AggregateWith 参数聚合

把**多个 CSV 列聚合成一个对象**，内置的 `ArgumentsAccessor` 是最简单的聚合器：

```java
@ParameterizedTest
@CsvSource({"alice, 18, beijing"})
void register(ArgumentsAccessor args) {              // 按索引取值
    User user = new User(args.getString(0), args.getInteger(1));
    assertEquals("beijing", args.getString(2));
}

// 或者自定义 ArgumentsAggregator
@ParameterizedTest
@CsvSource({"alice, 18, beijing"})
void register(@AggregateWith(UserAggregator.class) User user) {
    assertNotNull(user.getName());
}
```

> **注意**：`ArgumentsAccessor` 用魔法索引取列，重构 CSV 列顺序时容易悄悄错位；列数超过 3 建议用 `@AggregateWith` 或直接在 `@MethodSource` 里构造对象，把"列到字段"的映射收敛到一处。

# 4. 动态测试（Dynamic Test）

参数化测试的数据在**执行前**就确定了（注解字面量或工厂方法）；动态测试则把用例的生成推迟到**运行时**，由一个 `@TestFactory` 方法返回一批 `DynamicTest`。适合"用例数量和内容取决于外部状态"的场景：扫描目录对每个文件生成一个解析用例、从数据库读出所有配置项逐个校验、根据输入规模生成一组性能用例。

## 4.1 @TestFactory 与 DynamicTest

```java
import org.junit.jupiter.api.DynamicTest;
import org.junit.jupiter.api.TestFactory;
import java.util.stream.Stream;
import static org.junit.jupiter.api.DynamicTest.dynamicTest;
import static org.junit.jupiter.api.Assertions.assertTrue;

class CodecTest {

    @TestFactory
    Stream<DynamicTest> decode_all_fixtures() {
        // 运行时扫描 fixtures 目录，每个文件生成一个用例
        return Files.list(Path.of("src/test/fixtures"))
            .map(path -> dynamicTest(
                "decode " + path.getFileName(),        // 用例显示名
                () -> assertTrue(codec.decode(path).isValid())  // Executable
            ));
    }
}
```

工厂方法可以返回 `Collection`、`Iterable`、`Iterator`、`Stream` 或数组，元素是 `DynamicTest`。用 `Iterator` 时 JUnit 边迭代边执行，**可以处理无法一次载入内存的海量用例**，且失败不影响后续迭代。

## 4.2 DynamicContainer 组织层级

`DynamicContainer` 把动态用例组织成树，报告里呈现为嵌套结构：

```java
@TestFactory
Stream<DynamicContainer> grouped_by_type() {
    return Stream.of(
        DynamicContainer.dynamicContainer("图片", Stream.of(
            dynamicTest("png 解码", () -> decode("a.png")),
            dynamicTest("jpg 解码", () -> decode("b.jpg"))
        )),
        DynamicContainer.dynamicContainer("音频", Stream.of(
            dynamicTest("mp3 解码", () -> decode("c.mp3"))
        ))
    );
}
```

```text
CodecTest
 └─ grouped_by_type()
     ├─ 图片
     │   ├─ png 解码   ✔
     │   └─ jpg 解码   ✔
     └─ 音频
         └─ mp3 解码   ✔
```

## 4.3 动态测试的局限

动态测试灵活，但代价不小，决定用它之前先对照这张表：

| 能力 | @ParameterizedTest | @TestFactory |
|------|-------------------|--------------|
| 每个用例独立生命周期回调 | 支持（每次调用前后都触发 `@BeforeEach`/`@AfterEach`） | **不支持**，回调只作用于工厂方法本身 |
| 每个用例独立测试实例 | 支持 | 不支持，所有用例共享工厂方法所在的实例与上下文 |
| 类型转换/聚合 | 内置丰富 | 全部手工 |
| 被 `--tests`、tag 单独过滤 | 可以（按方法过滤） | 只能过滤到工厂方法，无法过滤单个动态用例 |
| `@Tag`/`@Disabled` 挂到单个用例 | 可以（方法级） | **不可以**，动态节点上没有注解元数据 |
| 失败报告 | 每次调用独立展示 | 独立展示，但无扩展回调可介入 |

**最关键的一条**：动态测试的 `@BeforeEach`/`@AfterEach` 只在工厂方法前后各执行一次，**不会**在每个 `DynamicTest` 执行前后重复触发。如果需要每个用例独立的资源准备/清理，只能在 `Executable` lambda 内部自己写 try-finally。需要每用例完整生命周期的场景，`@ParameterizedTest` 配合 `@MethodSource` 几乎总是更好的选择——动态测试留给"用例集合本身只能运行时确定"的场景。

# 5. 小结

| 主题 | 要点 |
|------|------|
| 依赖 | 参数化在 `junit-jupiter-params` 模块，聚合包已含 |
| 数据源 | `@ValueSource` 单参字面量；`@CsvSource`/`@CsvFileSource` 多列表格；`@EnumSource` 枚举遍历；`@MethodSource` 代码构造；`@ArgumentsSource` 复用 Provider |
| 工厂方法 | 默认 static；`PER_CLASS` 生命周期下可非 static（5.7+） |
| 转换 | 隐式转换覆盖基本类型/枚举/时间/单参构造器；`@ConvertWith` 显式；`@AggregateWith` 多列聚合 |
| 显示名 | `name` 属性 + `{index}`/`{0}` 占位符，失败排查靠它 |
| 动态测试 | `@TestFactory` + `DynamicTest`/`DynamicContainer`，运行时生成用例 |
| 动态测试局限 | 无每用例生命周期回调、无注解元数据、无法单独过滤——优先参数化，动态测试兜底 |
