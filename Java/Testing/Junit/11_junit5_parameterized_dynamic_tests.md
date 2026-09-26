# 1. 参数化测试（Parameterized Test）

JUnit 5 内置强大的参数化能力，支持多种数据源。

## 1.1 基础示例

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})
void is_positive(int n) {
    assertTrue(n > 0);
}
```

## 1.2 常用数据源

- `@ValueSource`
- `@CsvSource`
- `@CsvFileSource`
- `@EnumSource`
- `@MethodSource`
- `@ArgumentsSource`

```java
@ParameterizedTest
@CsvSource({"alice, 18", "bob, 20"})
void parse_user(String name, int age) { }
```

## 1.3 参数转换与聚合

```java
@ParameterizedTest
@CsvSource({"alice,18"})
void parse_user(@AggregateWith(UserAggregator.class) User user) { }
```

# 2. 动态测试（Dynamic Test）

动态测试适合运行时生成用例，而不是在编译期固定。

## 2.1 示例

```java
@TestFactory
Stream<DynamicTest> dynamic_tests() {
    return Stream.of("a", "b")
        .map(s -> DynamicTest.dynamicTest("len=" + s, () -> assertTrue(s.length() > 0)));
}
```

## 2.2 适用场景

- 基于数据库/配置动态生成用例
- 根据输入规模或外部文件生成用例

## 2.3 注意点

- 动态测试不支持 `@BeforeEach` 直接包裹每个动态节点。
- 需要自己控制资源与上下文生命周期。
