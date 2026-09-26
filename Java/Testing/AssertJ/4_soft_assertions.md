
在系统测试和复杂业务流程的验证中，我们可能会对由很多字段组成的同一个业务对象进行超过十几项断言验证。传统断言存在"一旦前面一项失败，程序直接抛出异常，后面的断言代码被拦截执行"的缺陷。AssertJ 提出了 **软断言 (Soft Assertions)** 来解决此痛点。

---

# 1. 什么是软断言？解决了什么痛点？

假设有如下常规的强断言（Hard Assertion）测试：

```java
@Test
void hardAssertionExample() {
    User user = userService.getUser(1001);

    assertThat(user.getName()).isEqualTo("Admin"); // 假设这里期望是 Admin，但实际是 Guest，断言失败
    assertThat(user.getAge()).isEqualTo(30);       // 这里将永不执行
    assertThat(user.getRole()).isEqualTo("SYS");   // 这里将永不执行
}
```
**痛点**：因为第一行失败，抛出了 `AssertionError`，导致我们无法在单次构建中知道该用户的 `age` 和 `role` 是否是对的。我们必须先修复 name，重新运行，才能测出后面的错，极其耽误效率。

**软断言机制：收集所有断言结果，即便中间遇到失败也不会中断代码执行，在代码块最后阶段一次性输出所有失败的明细（Multiple FailuresError）**。

---

# 2. `SoftAssertions` 经典用法

AssertJ 提供了原生的对象 `SoftAssertions`：

```java
import org.assertj.core.api.SoftAssertions;
import org.junit.jupiter.api.Test;

public class SoftAssertionTest {

    @Test
    void testStandardSoftAssertion() {
        User user = new User("Guest", 18, "USER");

        // 1. 创建 SoftAssertions 实例
        SoftAssertions softly = new SoftAssertions();

        // 2. 调用 softly 进行断言，即使错，也不会在这两行立刻抛出异常中断
        softly.assertThat(user.getName()).as("校验用户名").isEqualTo("Admin"); 
        softly.assertThat(user.getAge()).as("校验年龄").isEqualTo(30);

        // 3. 必须包含这句！这告诉引擎去核算刚才收集的所有错误，并在这里统一抛出（若有错）
        softly.assertAll(); 
    }
}
```

如果在上述例子中两个断言都失败了，控制台会一次性打印如下日志说明：
```text
Multiple Failures (2 failures)
-- failure 1 --
[校验用户名] expected: "Admin" but was: "Guest"
-- failure 2 --
[校验年龄] expected: 30 but was: 18
```
> `as("Description")` 是配合软断言的神器。当多个失败出现时，描述可以立刻让你看出哪个业务维度崩了。

---

# 3. 静态 `assertSoftly` 方法结合 Lambda

经常忘记调用 `softly.assertAll()` 是这个方案的痛点（如果不调用，即使断言全错，单元测试也会由于正常返回而判定“绿灯通过”，极其危险！）。
利用 Lambda，AssertJ 提供了自带资源清理的静态辅助：

```java
import static org.assertj.core.api.SoftAssertions.assertSoftly;

@Test
void testLambdaSoftAssertion() {
    User user = new User("Guest", 18, "USER");

    // 不再需要手写 assertAll()，代码块执行完毕自动调用
    assertSoftly(softly -> {
        softly.assertThat(user.getName()).isEqualTo("Admin");
        softly.assertThat(user.getAge()).isEqualTo(30);
    });
}
```

---

# 4. SpringBoot + JUnit 5 环境下的自动化拓展 (`SoftAssertionsExtension`)

如果你不想满屏幕都是 `softly.XXX` 和内部匿名类，AssertJ 也提供了深度的 JUnit 5 集成，完全做到了"无侵入"注入。

需要在类级别打上 `@ExtendWith`。

```java
import org.assertj.core.api.SoftAssertions;
import org.assertj.core.api.junit.jupiter.SoftAssertionsExtension;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

// 1. 添加扩展支持
@ExtendWith(SoftAssertionsExtension.class)
public class AdvancedSoftTest {

    // 2. 测试方法必须注入 SoftAssertions 参数（或 InjectSoftAssertions 注解绑定字段对象）
    @Test
    void testWithExtension(SoftAssertions softly) {
        User user = new User("Guest", 18, "USER");

        softly.assertThat(user.getName()).isEqualTo("Admin");
        softly.assertThat(user.getAge()).isEqualTo(30);
        
        // 3. 重点：连 assertAll() 都不用写了！扩展类会在拦截方法执行结束后自行处理收尾。
    }
}
```
这是工业级别非常推荐的写法，使得代码拥有与 Hard Assert 同样干爽的结构，但保留了批量容错与报告的特点。