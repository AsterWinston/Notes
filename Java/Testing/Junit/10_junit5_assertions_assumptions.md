# 1. Assertions 基本能力

JUnit 5 使用 `org.junit.jupiter.api.Assertions`，提供更强的断言能力。

## 1.1 常用断言

```java
assertEquals(3, calc.add(1, 2));
assertTrue(result.isOk());
assertNotNull(user);
```

## 1.2 异常断言

```java
IllegalArgumentException ex = assertThrows(
    IllegalArgumentException.class,
    () -> service.create(null)
);
assertEquals("name is required", ex.getMessage());
```

## 1.3 组合断言

```java
assertAll(
    () -> assertEquals("alice", user.getName()),
    () -> assertTrue(user.isActive())
);
```

## 1.4 超时断言

```java
assertTimeout(Duration.ofMillis(50), () -> service.process());
```

**提示**：`assertTimeoutPreemptively` 会在超时后中断线程，可能影响线程上下文。

# 2. Assumptions

## 2.1 基本用法

```java
assumeTrue(System.getProperty("os.name").contains("Linux"));
```

## 2.2 适用场景

- 仅在特定环境下执行的测试
- 与外部依赖强绑定的测试

# 3. 与 JUnit 4 的差异

- JUnit 5 引入 `assertAll`、更友好的异常断言。
- 使用 OpenTest4J 统一异常模型，便于工具处理失败原因。
