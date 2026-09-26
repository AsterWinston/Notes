
使用第三方库停留在 API 层面固然够用，但如果要应对极端的排错场景（甚至二次开发），掌握其底层拦截与抛错设计则大有裨益。

---

# 1. AssertJ 门面模式：为什么入口全是 `Assertions`

AssertJ 是一个极端使用 **门面模式 (Facade Pattern)** 与 **工厂模式** 的范例。代码里永远通过 `org.assertj.core.api.Assertions.assertThat()` 作为统一触发点。

如果你追溯它的 `Assertions` 类的源码，就会看到几百个基于类型的重载方法，这通过多态为强类型流式体系奠定基础：

```java
public class Assertions implements AssertionsForClassTypes, AssertionsForInterfaceTypes {
    
    // 返回对应的 AbstractStringAssert
    public static AbstractStringAssert<?> assertThat(String actual) {
        return new StringAssert(actual);
    }

    // 返回 AbstractListAssert
    public static <ELEMENT> checkList assertThat(List<? extends ELEMENT> actual) {
        return ListAssert.newInstance(actual);
    }
}
```

所有的这些专属类型 Assert（`StringAssert`, `ListAssert`）全部都会统归单线继承于一条核心骨带 `AbstractAssert<SELF, ACTUAL>`。

这个泛型设计极其精妙：
- `SELF`: 是当前断言类型的递归边界引用。用来支持子类调用各种 API 后仍然能原汁原味地返还自己（Fluent 风格）。
- `ACTUAL`: 是被断言的承载对象的源类型限制（确保你在数字处理上绝对不能点出字符串的 `startsWith` 联想）。

---

# 2. 何为 `AssertionError` (JVM 核心异常触发源)

由于在 Java 生态里，所有的测试框架底层都是遵循 Junit/TestNG 的报错共识。不论是 AssertJ 还是其他，核心就只有一点：遇到不符期望就抛出 `AssertionError`。

在每一次流式链的执行内部（比如你调了 `isNotNull()`），底层大概发生了如下事情：

```java
// AbstractAssert 源码节选简述
public SELF isNotNull() {
    // 调用内部核心工具 Failures
    objects.assertNotNull(info, actual);
    // 返回泛型定义的自己以供后续链条
    return myself;
}
```
此时 `objects.assertNotNull` 的深层源码逻辑如下：
```java
public void assertNotNull(AssertionInfo info, Object actual) {
    if (actual == null) {
        throw Failures.instance().failure(info, ShouldNotBeNull.shouldNotBeNull());
    }
}
```
最终 `Failures.instance().failure()` 调用，就是利用工厂构建一条纯粹的 `java.lang.AssertionError` (一种非常严重的 `Error` 而非 `Exception`)。
因为是 `Error`，它在方法栈向上溢出的时一般不被业务的 `try-catch(Exception c)` 阻拦，只有跑在最外圈的 JUnit Runner 进行专门的基于 `Throwable` 层面的拦截，并进而报告给 IDE 和测试报告生成器。

---

# 3. `SoftAssertions` 软断言的“黑魔法” (ByteBuddy 代理增强)

为什么使用 SoftAssertions 之后，代码原本因为调用 `failWithMessage` 的时候不再中断抛出了，而是一直憋到了 `assertAll()` 手动调用时统一引爆？

其实如果你翻开带依赖的源代码体系，你会惊讶发现：SoftAssertion 底层大面积使用了 **AOP 面向切面编程思想与动态代理技术（CGLIB 及其继承者 ByteBuddy）**！

## 3.1 拦截原理剖析

当你书写这么一段代码的时候：
```java
SoftAssertions softly = new SoftAssertions();
softly.assertThat(val).isNotNull(); 
```

`softly.assertThat(val)` 交回给你的并没有真正的原版的 `StringAssert` 或者 `ObjectAssert`。它通过底层的字节码代理技术（目前默认使用的是 ByteBuddy 衍生）为你交出了一个被包裹过的“间谍 Assert 实例”。

在这个代理类实例中，它切面截获了你的所有方法调用。如果捕捉到底层被触发并且抛出 `AssertionError`：
1. 拦截该 `Error`，而不是让它进入普通的 JVM 回溯区。
2. 将这条 Error，提取出它的文本或堆栈。
3. 存入到一个内置注册表 `List<Throwable> errorsCollected` 去。
4. 提供伪装机制，继续骗 JVM 方法顺利执行，返还代理自身供你链上下一个方法。

当你最终调用 `softly.assertAll()` 的时机：
其实就是去 `errorsCollected` 看，为空万事大吉。如果不为空，则将这个 List 进行格式化遍历重组，融合成一个自定义派生的超巨大 `MultipleFailuresError` 抛出！

这段代理实现在 AssertJ 的 `ErrorCollector` 内部以及相关 Proxy 工厂核心类极具观赏性，也是很多框架用来做静默采集的底层教科书。