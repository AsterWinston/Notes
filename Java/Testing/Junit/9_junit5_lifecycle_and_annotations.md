# 1. 生命周期概览

JUnit 5 默认每个测试方法创建新的测试实例，生命周期与 JUnit 4 类似但更灵活。

## 1.1 生命周期顺序

1. `@BeforeAll`
2. `@BeforeEach`
3. `@Test`
4. `@AfterEach`
5. `@AfterAll`

## 1.2 TestInstance 生命周期

- `PER_METHOD`（默认）：每个测试方法一个实例。
- `PER_CLASS`：一个测试类一个实例，可用于共享状态。

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class UserServiceTest { }
```

# 2. 常用注解

## 2.1 `@Test` / `@RepeatedTest`

- `@Test`：普通测试
- `@RepeatedTest`：重复测试

```java
@RepeatedTest(3)
void should_work_repeatedly() { }
```

## 2.2 `@BeforeAll` / `@AfterAll`

- 默认要求 `static`，当使用 `PER_CLASS` 时可为实例方法。

## 2.3 `@BeforeEach` / `@AfterEach`

方法级前置与后置处理。

## 2.4 `@DisplayName`

为测试提供友好名称，便于报告阅读。

## 2.5 `@Nested`

用于嵌套测试类，表达上下文与场景。

```java
class OrderTest {
    @Nested
    class WhenPaid { }
}
```

## 2.6 `@Disabled`

跳过测试（可提供原因）。

# 3. 组织结构建议

## 3.1 AAA 结构

- Arrange（准备）
- Act（执行）
- Assert（断言）

## 3.2 命名规则

推荐 `should_xxx_when_yyy` 或中文描述搭配 `@DisplayName`。
