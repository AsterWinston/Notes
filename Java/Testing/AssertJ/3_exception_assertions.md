
在单元测试中，我们往往不仅需要验证代码"执行正确"，还需要验证它在面对非法输入或异常状态时，能"按预期抛出特定的异常"。本篇详细探讨 AssertJ 提供的高阶异常处理 API，并将其与 JUnit 进行对比。

---

# 1. 为什么需要特殊的异常断言机制？

在 JUnit 4 时期，开发者习惯使用 `@Test(expected = IllegalArgumentException.class)` 来捕获预期异常。
这存在两个致命缺陷：
1. **范围过大**：该方法中任何代码抛出的 `IllegalArgumentException` 都会导致测试通过，可能掩盖了真正的问题（比如你希望这行代码抛出，结果却是另一行抛出了）。
2. **缺乏详细校验**：只能校验异常类型，无法对异常消息内容（Message）或原因（Cause）进行精确核对。

尽管 JUnit 5 引入了 `assertThrows()` 解决了范围问题，但对于后续返回 Exception 的链式验证，AssertJ 则提供了更优雅、更一气呵成的方案。

---

# 2. 核心方法：`assertThatThrownBy` (最推荐)

`assertThatThrownBy()` 是 AssertJ 暴露的最常用的异常断言入口，它接收一个 Java 8+ 的 `ThrowingCallable` (可以理解为一个 lambda 实现的 Runnable)。它能把代码块抛出的异常捕获并返回成一个流式断言。

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@Test
void testDivideByZero() {
    // 捕获可能出现异常的逻辑
    assertThatThrownBy(() -> {
        int result = 10 / 0;
    })
    .isInstanceOf(ArithmeticException.class)     // 1. 验证类型
    .hasMessageContaining("/ by zero")           // 2. 验证错误信息是否包含这段内容
    .hasNoCause();                               // 3. 验证是否有源头 cause
}
```

## 2.1 链式错误追查

它完全支持链式追踪 `cause`，以及校验其堆栈根源（`Root Cause`）：

```java
assertThatThrownBy(() -> { throw new RuntimeException("Outer", new IOException("File lock")); })
    .isInstanceOf(RuntimeException.class)
    .hasMessage("Outer")
    .hasCauseInstanceOf(IOException.class) // 对内部 Cause 进行断言
    .getCause()                            // 切换断言上下文，现在的上下文进入内部 Cause 了
    .hasMessage("File lock");              // 此时断言的是 Cause 的消息
```

---

# 3. `assertThatExceptionOfType` 变体

这是 AssertJ 提供的用来加强**类型安全**与**阅读流畅度**的另一种写法，尤其对于 `IllegalArgumentException` 等标准异常，使用时可读性更高：

```java
import static org.assertj.core.api.Assertions.assertThatExceptionOfType;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;

@Test
void testInvalidInput() {
    User user = new User();
    
    // 方法 1: 使用 assertThatExceptionOfType
    assertThatExceptionOfType(IllegalArgumentException.class)
        .isThrownBy(() -> user.setAge(-5))
        .withMessageContaining("Age cannot be negative");
        
    // 方法 2: AssertJ 内置的最常用的部分预设 Wrapper（比如 NPE，非法参数，非法状态等）
    assertThatIllegalArgumentException()
        .isThrownBy(() -> user.setAge(-5))
        .withMessage("Age cannot be negative");
}
```
**这种写法的特点刚好倒装了主谓语：由原来的"发生这段行为期望抛出这样的异常" 变成了 "这样的异常预期将在下面这段行为中被抛出"**。这完全取决于团队的编码命名倾向。

---

# 4. `catchThrowable`：先捕获，再断言

在某些极复杂场景下，如果仅仅是链式 API 不满足多重分支的要求，我们可以单独把异常对象提取出来成一个普通对象，然后再做断言。

```java
import static org.assertj.core.api.Assertions.catchThrowable;

@Test
void testCatchThrowable() {
    // 当没有异常抛出时，throwable 会是 null
    Throwable thrown = catchThrowable(() -> {
        throw new IllegalStateException("Business state is wrong");
    });
    
    // 把 thrown 当作一个普通对象拿去断言
    assertThat(thrown).isNotNull()
                      .isInstanceOf(IllegalStateException.class)
                      .hasMessageStartingWith("Business");
}
```

# 5. 配合 `doesNotThrowAnyException`

在证明某段代码“极其稳定，绝对不会抛出异常”时，这是一种显式宣告的好方法：

```java
import static org.assertj.core.api.Assertions.assertThatCode;

@Test
void testSafeCode() {
    // 明确告诉读代码的人：这不应该抛任何异常
    assertThatCode(() -> {
        // ... 一堆业务代码
        Thread.sleep(100);
    }).doesNotThrowAnyException();
}
```