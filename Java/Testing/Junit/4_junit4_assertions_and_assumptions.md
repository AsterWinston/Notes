# 1. 断言（Assertions）

JUnit 4 的断言工具集中在 `org.junit.Assert`，用于验证预期与实际是否一致。

## 1.1 常用断言

- `assertEquals(expected, actual)`
- `assertTrue(condition)` / `assertFalse(condition)`
- `assertNull(obj)` / `assertNotNull(obj)`
- `assertSame(expected, actual)`（比较引用）

```java
assertEquals("alice", user.getName());
assertTrue(user.isActive());
```

## 1.2 浮点比较

浮点数比较必须提供误差范围：

```java
assertEquals(3.1415, pi, 0.0001);
```

## 1.3 异常断言（传统方式）

JUnit 4 使用 `@Test(expected = ...)` 或 `ExpectedException` 规则：

```java
@Test(expected = IllegalArgumentException.class)
public void should_throw_exception() { }
```

**问题**：无法精确校验异常消息与类型层次结构。

# 2. Hamcrest Matcher

`assertThat` + Hamcrest 能提高可读性与表达力：

```java
import static org.hamcrest.CoreMatchers.*;
import static org.junit.Assert.*;

assertThat(user.getName(), is("alice"));
assertThat(list, hasItem("admin"));
```

## 2.1 常用 Matcher

- `is` / `equalTo`
- `not` / `nullValue`
- `instanceOf`
- `hasItem` / `hasSize`

# 3. 假设（Assumptions）

Assumption 用于在前置条件不满足时跳过测试（不是失败）。

## 3.1 常用方法

- `Assume.assumeTrue(condition)`
- `Assume.assumeNotNull(obj)`

```java
Assume.assumeTrue(System.getProperty("os.name").contains("Linux"));
```

# 4. 断言设计建议

## 4.1 保持断言与意图一致

- 一个测试只验证一个核心行为。
- 避免在断言中包含过多业务逻辑。

## 4.2 错误信息要清晰

```java
assertEquals("用户名不匹配", "alice", user.getName());
```

## 4.3 避免过度依赖 Hamcrest

Hamcrest 可读性强，但过度嵌套会让测试变得难懂，需平衡表达力与简洁性。
