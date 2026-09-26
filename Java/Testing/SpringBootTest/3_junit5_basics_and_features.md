# 1. JUnit 5 架构概览

## 1.1 三层架构

JUnit 5 由三个模块组成：

- **JUnit Platform**：运行测试的基础平台
- **JUnit Jupiter**：新的测试编程模型（注解与 API）
- **JUnit Vintage**：兼容 JUnit 4 的引擎

Spring Boot 默认启用 Jupiter，除非显式引入 Vintage。

## 1.2 运行模型

JUnit 5 使用 **Extension** 机制扩展测试能力，Spring 通过 `SpringExtension` 将测试与容器生命周期连接。

---

# 2. 生命周期与注解

## 2.1 基础注解

| 注解 | 作用 |
|------|------|
| `@Test` | 测试方法 |
| `@BeforeEach` | 每个测试前执行 |
| `@AfterEach` | 每个测试后执行 |
| `@BeforeAll` | 全部测试前执行 |
| `@AfterAll` | 全部测试后执行 |

## 2.2 `@TestInstance` 与生命周期

默认生命周期为 `PER_METHOD`，每个测试方法创建新实例。

- **PER_METHOD**：隔离性更好，启动成本更高
- **PER_CLASS**：可共享状态，适合昂贵初始化

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class CacheWarmupTest {

    @BeforeAll
    void initOnce() {
        // 适合昂贵初始化
    }
}
```

---

# 3. 断言与假设

## 3.1 断言

JUnit 5 自带断言，Spring Boot 默认集成 AssertJ。

```java
assertThat(actual).isEqualTo(expected);
```

## 3.2 假设（Assumptions）

用于跳过不满足前置条件的测试。

```java
assumeTrue(System.getProperty("os.name").contains("Windows"));
```

---

# 4. 参数化与动态测试

## 4.1 参数化测试

```java
@ParameterizedTest
@ValueSource(strings = {"a", "b"})
void shouldAcceptString(String input) {
    assertThat(input).isNotBlank();
}
```

常见数据来源：`@CsvSource`、`@MethodSource`、`@EnumSource`。

## 4.2 动态测试

用于运行期生成测试用例，适合批量规则验证。

```java
@TestFactory
Stream<DynamicTest> dynamicTests() {
    return Stream.of("a", "b")
        .map(v -> DynamicTest.dynamicTest("test:" + v, () -> assertThat(v).isNotBlank()));
}
```

---

# 5. 标签与条件执行

## 5.1 标签

```java
@Tag("slow")
@Test
void slowTest() {}
```

在构建工具中可按标签过滤执行。

## 5.2 条件执行

```java
@EnabledOnOs(OS.WINDOWS)
@Test
void windowsOnly() {}
```

---

# 6. 扩展模型与 Spring 集成

## 6.1 Extension 机制

JUnit 5 通过 `@ExtendWith` 增强测试行为，典型功能：

- 参数解析
- 生命周期回调
- 条件执行

## 6.2 SpringExtension

```java
@ExtendWith(SpringExtension.class)
class PlainSpringTest {}
```

在 Spring Boot 中，`@SpringBootTest` 会自动包含 `SpringExtension`，因此通常无需手动声明。

---

# 7. 常见实践建议

## 7.1 命名规范

- 类名以 `*Test` 或 `*IT` 结尾
- 方法名描述行为与期望结果

## 7.2 避免共享状态

- 尽量避免使用静态可变字段
- 如需共享，明确生命周期与重置策略

## 7.3 控制超时

```java
@Timeout(2)
@Test
void shouldFinishInTime() {
    // 2 秒内完成
}
```
