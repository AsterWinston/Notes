# 1. 为什么需要专门的异常断言机制

单元测试不仅要验证代码"执行正确"，还要验证它在面对非法输入或异常状态时能**按预期抛出特定的异常**。异常也是一种契约：抛什么类型、消息说什么、包装了什么原因，都是被测代码对外承诺的一部分。断言库对异常的支持程度，直接决定了这类契约能不能被完整锁住。

## 1.1 被历史淘汰的三种姿势

在给出 AssertJ 方案之前，先看要淘汰什么：

```java
// 错误示范一(JUnit 4 时代): 注解式, 验证范围是整个方法体
@Test(expected = IllegalArgumentException.class)
public void invalid_age() {
    User user = loadUser();          // 这行若抛 IAE, 测试同样"通过"
    user.setAge(-5);                 // 真正想验证的调用
}

// 错误示范二: try-catch 手动验证, 忘写最后的 fail 就漏报
@Test
void invalid_age_try_catch() {
    try {
        user.setAge(-5);
        // 忘记 fail("应抛出异常") —— 方法安静通过, 假绿灯!
    } catch (IllegalArgumentException e) {
        assertThat(e).hasMessageContaining("negative");
    }
}

// 错误示范三: JUnit 4 时代想在 try 后补断言, 异常类型还被擦成变量, 链式无从谈起
```

三个方案的共同缺陷：**其一范围过大**——注解式里方法中任何一行抛出目标异常都算通过，可能验证的根本不是你想验证的调用；**其二缺乏详细校验**——只能核对类型，无法精确核对消息（Message）、原因（Cause）与异常字段；**其三有假绿灯风险**——手动 try-catch 忘写 `fail()` 时，"没抛异常"这种最严重的回归反而测不出来。

JUnit 5 的 `assertThrows` 解决了范围与返回值问题，但它返回的是裸异常对象，后续对消息、原因的核对要另起一行行 `assertEquals(e.getMessage(), ...)`，链条是断的。AssertJ 把"捕获 + 类型 + 消息 + 原因"统一成一条流式链，这就是本篇的主题。

# 2. 核心方法：`assertThatThrownBy` （最常用）

**`assertThatThrownBy()` 是 AssertJ 最常用的异常断言入口**，接收一个 `ThrowingCallable`（可抛 checked 异常的 lambda，等价于 `@FunctionalInterface` 版的 `Callable`），执行它并捕获异常，返回 `AbstractThrowableAssert` 供链式深挖。**如果 lambda 没有抛异常，链上第一个断言直接失败，消息是 "Expecting code to raise a throwable"——不存在假绿灯**。

```java
package com.example.calc;

import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DivideTest {

    // 被测服务: loadReport 会把底层 IOException 包装成 AppException 抛出
    private final ReportService service = new ReportService();

    @Test
    void testDivideByZero() {
        // 捕获可能出现异常的逻辑, 范围被 lambda 精确圈定
        assertThatThrownBy(() -> {
            int result = 10 / 0;
        })
        .isInstanceOf(ArithmeticException.class)   // 1. 验证类型
        .hasMessageContaining("/ by zero")         // 2. 验证错误消息
        .hasNoCause();                             // 3. 验证没有包装原因
    }

    @Test
    void should_wrap_io_failure() {
        assertThatThrownBy(() -> service.loadReport(42))
                .isInstanceOf(AppException.class)
                .hasMessage("load report failed: 42")
                .hasCauseInstanceOf(IllegalStateException.class)     // 直接检查 cause 类型
                .hasRootCauseInstanceOf(IOException.class)           // 一直追到根因类型
                .hasRootCauseMessage("File lock")                    // 根因消息
                // hasFieldOrPropertyWithValue: 不写 getter 也能核对异常携带的业务错误码
                .hasFieldOrPropertyWithValue("errorCode", 40012);
    }
}
```

## 2.1 链式错误追查：cause 与 root cause

异常在分层架构里会被层层包装（service 包 repository，repository 包 IO），断言时要能**沿链下钻**。AssertJ 提供两组工具：不切换上下文的 `hasCauseInstanceOf/hasRootCauseInstanceOf/hasRootCauseMessage`，和显式切换断言对象的导航方法 `cause()/rootCause()`（3.23 起替代旧的 `getCause()/getRootCause()`）：

```java
import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ExceptionChainTest {

    @Test
    void walk_the_cause_chain() {
        assertThatThrownBy(() -> {
            throw new RuntimeException("Outer", new IllegalStateException("Middle",
                    new IOException("File lock")));
        })
        .isInstanceOf(RuntimeException.class)
        .hasMessage("Outer")
        .hasCauseInstanceOf(IllegalStateException.class)  // 断言直接下一层
        .rootCause()                                      // 切换上下文到最深层根因
        .isInstanceOf(IOException.class)
        .hasMessage("File lock");                         // 此时断言的是根因的消息
        // 也可用 .cause() 只下钻一层; 注意导航后无法回到外层继续链
    }
}
```

> **设计考量**：**优先断言 cause 链而不是只断言顶层类型**。微服务里把底层异常统一翻译成 `ServiceException` 是常态，如果测试只写 `isInstanceOf(ServiceException.class)`，那么数据库连接失败、参数校验失败、空指针全都会"通过"——因为都被翻译成了同一个壳。补一句 `hasRootCauseInstanceOf(DeadlockLoserDataAccessException.class)`，才能锁住"确实是死锁重试路径"这条契约。

## 2.2 消息断言家族

消息是异常契约里最容易腐烂的部分（改文案就破坏测试），所以**默认用包含匹配而不是全等**：

| 方法 | 语义 | 适用 |
|------|------|------|
| `hasMessage(String)` | 全等 | 消息含参数化占位结果、就是契约本身 |
| `hasMessageContaining / ContainingAll` | 包含（可多个） | 默认选择，对文案改动鲁棒 |
| `hasMessageNotContaining` | 不包含 | 断言没泄露敏感信息 |
| `hasMessageMatching / Pattern` | 正则 | 消息里有可变 ID/时间戳 |
| `hasMessageStartingWith / EndingWith` | 前缀/后缀 | 前缀即错误类别编码 |
| `message()` | 下钻成 StringAssert，复用字符串归一化家族 | 跨平台换行、大小写噪声（如 `.message().isEqualToNormalizingNewlines(...)`） |

```java
// 错误示范: 全等锁死整句文案, 产品改一个字测试就红
// assertThatThrownBy(...).hasMessage("用户 age 非法: -5, 必须介于 0 与 150 之间");

// 正确做法: 锁住业务关键词 + 参数值, 文案其余部分允许演化
assertThatThrownBy(() -> user.setAge(-5))
        .hasMessageContainingAll("age", "-5");
```

# 3. `assertThatExceptionOfType` 家族：类型前置的写法

这是 AssertJ 提供的另一种入口，把**期望异常类型前置**到入口方法名上，再回头指定触发代码，读起来是"断言 IllegalArgumentException 将在 setAge 中被抛出"：

```java
package com.example.model;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.AccessDeniedException;

import static org.assertj.core.api.Assertions.assertThatExceptionOfType;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

class UserValidationTest {

    private final User user = new User();
    private final FileService fileService = new FileService();

    @Test
    void testInvalidInput() {
        // 方法 1: 通用形态, 任意异常类型
        assertThatExceptionOfType(IllegalArgumentException.class)
                .isThrownBy(() -> user.setAge(-5))
                .withMessageContaining("Age cannot be negative");   // 注意动词变成 withXxx

        // 方法 2: 内置高频异常预设入口(NPE/IAE/ISE/IOException/ReflectiveOperationException...)
        assertThatIllegalArgumentException()
                .isThrownBy(() -> user.setAge(-5))
                .withMessage("Age cannot be negative")
                .withNoCause();

        assertThatNullPointerException()
                .isThrownBy(() -> user.getName().isEmpty())
                .withMessageContaining("name");

        // 多态匹配: instanceof 语义, 抛子类 AccessDeniedException 也能命中父类 IOException 断言
        assertThatExceptionOfType(IOException.class)
                .isThrownBy(() -> {
                    throw new java.nio.file.AccessDeniedException("/etc/shadow");
                })
                .withMessageContaining("/etc/shadow");
    }
}
```

两种风格对照：

| 维度 | `assertThatThrownBy(callable).isInstanceOf(X.class)` | `assertThatExceptionOfType(X.class).isThrownBy(callable)` |
|------|------|------|
| 阅读顺序 | 先行为后期望（陈述句） | 先期望后行为（契约句） |
| 类型不匹配时 | 作为链上一环失败，消息对比类型 | 入口即校验类型，后续 withXxx 专注消息/cause |
| 断言方法名 | `hasMessage/hasCause...` | `withMessage/withCause...` |
| 适用 | 异常类型本身是被验证信息 | 类型已知、重点是消息与原因的长串校验 |

**这两种写法倒装了主谓语**：由"发生这段行为、期望抛出这样的异常"变成"这样的异常、预期将在下面这段行为中被抛出"。功能完全等价，选择取决于团队编码语感，但同一个工程里最好统一，review 时两类风格混排会增加阅读成本。

# 4. `catchThrowable` 家族：先捕获，再当普通对象断言

当链式 API 表达不了控制流需求——**同一个调用要根据异常类型走不同断言分支**，或者要把异常对象交给第三方工具（比如喂给错误码解析器）——就把异常先抓出来成普通变量：

```java
package com.example.branch;

import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.catchThrowable;
import static org.assertj.core.api.Assertions.catchThrowableOfType;

class BranchingExceptionTest {

    @Test
    void testCatchThrowable() {
        // 当没有异常抛出时, thrown 为 null(注意: 这是特性也是陷阱)
        Throwable thrown = catchThrowable(() -> {
            throw new IllegalStateException("Business state is wrong");
        });

        // 把 thrown 当普通对象拿去断言
        assertThat(thrown)
                .isNotNull()
                .isInstanceOf(IllegalStateException.class)
                .hasMessageStartingWith("Business");
    }

    @Test
    void catch_typed_and_branch() throws IOException {
        // 3.24+ 签名: 类型在前, callable 在后(旧参数序已废弃)
        AppException ex = catchThrowableOfType(AppException.class, () -> service.load(42));
        if (ex == null) {
            return; // 正常返回路径, 不视为失败
        }
        // 错误示范: 期望异常却没抛时 ex 为 null, 直接解引用会 NPE 且消息难读
        // assertThat(ex.getErrorCode()).isEqualTo(40012);
        // 正确做法: 先断言非空再下钻, 或干脆用 assertThatThrownBy 让框架兜底
        assertThat(ex).isNotNull();
        assertThat(ex.getErrorCode()).isEqualTo(40012);
    }
}
```

配套还有语义化预设 `catchNullPointerException()`、`catchIOException()`、`catchIllegalArgumentException()` 等，以及兜底工具 `failBecauseExceptionWasNotThrown(Class)`——手动 try-catch 风格里补"没抛即失败"的最后一环。

> **注意**：`catchThrowable` 返回 null 不失败——它只是捕获器，不是断言。用它就必须紧跟着 `assertThat(thrown).isNotNull()`，否则"没抛异常"这个最危险的场景会静默通过。**能用 assertThatThrownBy 表达的一律不用 catchThrowable**，后者只留给真正需要分支控制流的场景。

# 5. 反向断言：`assertThatCode` 与 `assertThatNoException`

证明某段代码"极其稳定、绝对不会抛出异常"时，显式宣告比让它裸跑更能表达意图：

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatNoException;

class SafeCodeTest {

    @Test
    void testSafeCode() throws InterruptedException {
        // 明确告诉读代码的人: 这段的契约就是"不抛"
        assertThatCode(() -> {
            parser.parse("");            // 空输入不应炸
            Thread.sleep(100);           // lambda 里可直接调受检方法, 免写 try-catch
        }).doesNotThrowAnyException();

        // 3.18+ 的另一入口, 语义完全相同
        assertThatNoException().isThrownBy(() -> parser.parse(null));
    }
}
```

> **设计争议**：给"本来就不该抛异常"的普通调用套 `assertThatCode(...).doesNotThrowAnyException()` 属于噪音——JUnit 的默认语义就是未捕获异常即失败，多包一层只会稀释测试重点。它的正当场景只有两个：**方法签名声明了受检异常而你不想写 try-catch**（lambda 吞掉编译错误），以及**契约本身就是"必须不抛"**（空值防御、幂等重试的第二次调用）。除此之外不要习惯性包裹。

# 6. 受检异常与 AssertJ 的入口选择：那层模糊重载

受检异常场景曾是 `assertThat` 家族的一个著名编译坑：当被测对象是 `T extends Exception` 这类**有界泛型**时，它既可能匹配 `assertThat(Throwable)` 又可能匹配 `assertThat(Iterable)`（万一一边实现里带了集合接口），Java 8 的重载决议会报 ambiguous。官方因此把入口拆成 `AssertionsForClassTypes` 与 `AssertionsForInterfaceTypes` 两个接口，由 `Assertions` 同时实现——这也是源码里 `public class Assertions implements ...` 结构的由来（原理见第 7 篇）。日常写法的对应关系：

```java
// 通用对象断言拿不准用哪个入口时, 这些具名变体永远无歧义:
assertThatObject(value);        assertThatList(list);       assertThatCollection(coll);
assertThatIterable(iterable);   assertThatStream(stream);   assertThatCharSequence(cs);
assertThatComparable(cmp);      assertThatPath(path);       assertThatPredicate(p);

// 异常对象本身: assertThat(throwable) 会按静态类型走 ThrowableAssert 分支,
// 静态类型仍模糊时退一步强转: assertThat((Object) throwable)
```

**记住一条就够：IDE 报 assertThat 二义性时，不是 AssertJ 坏了，是类型边界模糊——换用上述具名入口，或用 `(Object)` 强转收敛。**

# 7. 小结

异常断言的三种姿势与适用边界：`assertThatThrownBy(callable).isInstanceOf(...).hasMessageContaining(...).rootCause()` 是**默认选择**，一条链完成"捕获—类型—消息—原因"四层校验，没抛即失败，杜绝假绿灯；`assertThatExceptionOfType(X).isThrownBy(...).withMessage(...)` 把类型前置，**契约句式**更适合类型已知、重点核消息的场景，两者按团队语感二选一别混用；`catchThrowable/catchThrowableOfType` 返回可空对象，**只留给需要分支控制流**的场景，且必须显式补 `isNotNull()`。反向断言 `assertThatCode(...).doesNotThrowAnyException()` 仅在"不抛本身是契约"或受检异常免写 try-catch 时使用。所有入口底层都收敛到 `ThrowableAssert` 与统一的失败工厂，机制见第 7 篇。
