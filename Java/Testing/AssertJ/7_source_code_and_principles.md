# 1. 总体架构：门面、断言类层次、内部校验器

停留在 API 层面使用 AssertJ 固然够用，但要应对极端排错场景（失败消息为什么长这样、软断言为什么不炸、二义性报错怎么治），甚至做二次开发，就必须理解它的分层。AssertJ-core 的架构是教科书级的**门面 + 递归泛型模板 + 策略化错误工厂**组合。

## 1.1 三层结构总览

```text
┌────────────────────────────────────────────────────────────┐
│ 入口门面层: org.assertj.core.api.Assertions                │
│   数百个静态 assertThat 重载 + 辅助静态方法(within/tuple) │
│   BDDAssertions extends Assertions(then 版入口)           │
├────────────────────────────────────────────────────────────┤
│ 断言对象层: AbstractAssert<SELF, ACTUAL> 及其子类树        │
│   StringAssert / ListAssert / MapAssert / ThrowableAssert  │
│   每个类型专属断言方法都定义在本层, 返回 SELF 续链         │
├────────────────────────────────────────────────────────────┤
│ 内部校验层: org.assertj.core.internal.Objects/Strings/...  │
│   真正干活的无状态单例; 失败时经 Failures + Should* 工厂    │
│   产出 OpenTest4J/AssertionError 抛出或上交给代理收集      │
└────────────────────────────────────────────────────────────┘
```

`Assertions` 是极端的门面模式 + 工厂模式范例：每个 `assertThat` 重载本质是一个**按参数静态类型分派的产品工厂**（`StringAssert`、`ListAssert`、`ObjectAssert`……）。类上同时实现 `InstanceOfAssertFactories` 等接口，使得 `LIST`、`MAP` 这类常量也能从 `Assertions` 一次导入。之所以曾经把入口接口拆成 `AssertionsForClassTypes` 与 `AssertionsForInterfaceTypes`（并保留至今作为并列入口），是为了治疗 Java 8 对一个**有界泛型对象**（如 `T extends Comparable & List`）同时匹配多个 assertThat 重载时的 ambiguous 编译错误——需要绕开时直接换用那两个具名入口类。

## 1.2 递归泛型 `AbstractAssert<SELF, ACTUAL>`

所有类型专属 Assert 统统收敛在一条核心继承带下：

```java
public abstract class AbstractAssert<SELF extends AbstractAssert<SELF, ACTUAL>, ACTUAL> {

    protected final ACTUAL actual;           // 被断言对象
    protected final SELF myself;             // this 的类型安全视图, 链式返回用
    public WritableAssertionInfo info;       // 本次断言的上下文(描述/表示器/覆盖消息)

    protected AbstractAssert(ACTUAL actual, Class<?> selfType) {
        this.myself = (SELF) selfType.cast(this);   // selfType 只为把 this 安全降型为 SELF
        this.actual = actual;
        this.info = new WritableAssertionInfo(customRepresentation);
    }
}
```

这个泛型设计极其精妙：

- **`SELF` 是指向自身类型的递归引用**，作用是让定义在基类上的通用方法（`isNotNull`、`isEqualTo`、`as`）返回后**静态类型不降级**：`assertThat(user).isNotNull().isAdult()` 里 `isNotNull()` 声明返回 `SELF`，对 `UserAssert` 而言 SELF=UserAssert，业务方法继续可点。若没有 SELF 参数，基类方法只能返回 `AbstractAssert`，自定义断言的链式体验当场破产。
- **`ACTUAL` 锁定被断言对象类型**，构成"类型即文档"的编译期边界——数字上点不出 `startsWith`。
- 子类构造器传入的 `selfType`（如 `UserAssert.class`）只用于把 `this` 安全 cast 成 `SELF`；`cause()`、`extracting(..., assertFactory)` 这类**切换断言对象**的方法靠 `newObjectAssert`/`newListAssertInstance` 工厂造新实例，再用包私有的 `withAssertionState` 把表示器、描述、覆盖消息复制过去。

## 1.3 内部校验器与断言类的分工

> **注意**：internal 执行器是**无状态单例**，比较策略（如自定义 Comparator）不存放在执行器上，而是存放在每个断言实例的 `info`/`objects` 副本里——所以 `usingComparator` 只影响当前链，不会泄漏到别的测试线程；排错时看到"另一处断言也变了行为"，先怀疑是不是误用了 `setCustomRepresentation` 这类全局静态开关。

`StringAssert.contains` 并不自己实现子串检查，而是转手给无状态的内部执行器：

```java
// AbstractCharSequenceAssert 源码节选(骨架)
public SELF contains(CharSequence... values) {
    strings.assertContains(info, actual.toString(), values);   // 委托 org.assertj.core.internal.Strings
    return myself;
}
```

internal 包（`Objects`、`Strings`、`Iterables`、`Maps`、`Comparables`……）以**单例**形式存在，集中实现比较逻辑；断言类只是把这些执行器按类型语法糖化。好处：同一套 `assertContains` 逻辑被 `IterableAssert`、`ListAssert`、`ObjectArrayAssert`、软断言代理、生成的自定义断言**复用**，API 层的组合爆炸不会带来实现层的重复。

# 2. 一次链式调用的完整生命周期

以最小的链 `assertThat(value).as("描述").isNotNull()` 走一遍全流程：

```text
 Assertions.assertThat(obj)
     │ 按静态类型工厂化
     ▼
 ObjectAssert(actual=obj, info=new WritableAssertionInfo)
     │ .as("描述")  info 记入 description, 返回 myself(同一实例)
     ▼
 ObjectAssert.isNotNull()
     │ 委托 internal/Objects.assertNotNull(info, actual)
     ├─ actual != null ──► 正常返回, 链继续
     └─ actual == null ──► ShouldNotBeNull.shouldNotBeNull() 生成 ErrorMessageFactory
                 │
                 ▼
             Failures.instance().failure(info, factory)
                 ├─ info 里有 overridingErrorMessage? 有则直接用它造错
                 └─ 否则 factory.newAssertionError(description, representation)
                         │ AssertionErrorCreator 反射尝试:
                         ├─ OpenTest4J 在类路径 ──► 抛 AssertionFailedError(带 expected/actual)
                         └─ 不在 ─────────────────► 抛普通 java.lang.AssertionError
```

## 2.1 `AssertionInfo`：一条链的"病历夹"

`WritableAssertionInfo` 对象携带三样东西，解释了 AssertJ 大半的行为：

- **description**：`as()/describedAs()` 写入，失败消息渲染时前置为 `[描述]`；
- **representation**：决定 actual/expected 在消息里的长相（第 3.2 节）；
- **overridingErrorMessage**：`withFailMessage()/overridingErrorMessage()` 写入，`Failures` 造错时优先采用。

注意 3.x 里**软/硬断言的区别不在 info 上，而在调用对象本身**：`softly.assertThat(x)` 返回的是代理实例，`assertThat(x)` 返回裸实例——同一行 `.isEqualTo(...)` 的行为差异由代理拦截制造（第 4 节）。内置导航方法靠包私有的 `withAssertionState` 搬运上下文；第三方自定义断言下钻子对象时要自己搬描述（第 5 篇）。

## 2.2 错误消息工厂：`Should*` 命名族

AssertJ 不为每种失败写 try-catch 拼字符串，而是把"失败文案"对象化：`org.assertj.core.error` 包内每个 `ShouldBeEqual`、`ShouldNotBeNull`、`ShouldContain` 都继承 `BasicErrorMessageFactory`、以 `ErrorMessageFactory` 身份按 actual/expected/description 产出消息文本，`Failures.instance().failure(info, factory)` 是统一的造错出口（软断言场景则由代理层拦截异常改走收集器，见第 4 节）。**设计收益**：文案集中管理（改措辞不动断言逻辑）、工厂类可独立单测、失败得以机器可读化——`AssertionErrorCreator` 构造错误时反射探测 OpenTest4J：类路径上有就产出 `AssertionFailedError`，它除了拼好的 message 还**携带 expected/actual 的 ValueWrapper**，IDE 的 diff 视图与报告平台的高亮对比都消费这份结构化数据；OpenTest4J 缺失则回退普通 `java.lang.AssertionError`，只有文本没有结构。

因为是 `AssertionError` 谱系（`Error` 而非 `Exception`），业务代码里的 `catch (Exception e)` 不会误吞断言失败，只有最外圈 JUnit Runner 在 `Throwable` 层面拦截并上报——这也是**永远不要在生产主流程里 catch Throwable 吞错误**这条纪律能保护测试的原因。

# 3. `Condition` 机制与表示器 `Representation`

## 3.1 Condition：谓词 + 描述的可组合断言单元

`Condition<T>` 内部只有两个成员：`Predicate<T> predicate` 与 `description`。`assertThat(x).is(cond)` 委托 `internal/Conditions.assertIs(info, actual, condition)`（`has()` 走 `assertHas`，殊途同归），通过时无事发生，失败时把 `condition.description()` 塞进 `ShouldBe/ShouldHave` 消息工厂——这解释了第 5 篇说的"Condition 失败消息只有描述没有细节"：机制上它只能产出"to be: <description>"这一句话。`allOf/anyOf/not` 组合子本身也是 Condition 的装饰器，递归求值并拼接描述。

## 3.2 Representation：值到文本的翻译官

所有进入失败消息的 actual/expected 都要经 `Representation` 接口（`toStringOf(Object)`、`unrepresentable()` 标记不可展示的类如 ClassLoader）。默认实现 `StandardRepresentation` 维护一张**类型 → 格式化函数注册表**：String 加引号、char 显示 `'a'`、集合按 `[e1, e2]` 打印且受 `maxElementsForPrinting`（默认 1000）与单行阈值控制、Date/时间按注册的 pattern 渲染。第 6 篇的 `registerFormatterForType(Money.class, f)` 正是往这张表里插条目；`withRepresentation(...)` 则是给 `info.representation` 临时换实现。**排错现场看到"对象打印太啰嗦/太裸奔"，第一反应该是动表示器，而不是给领域类塞巨型 toString。**

# 4. `SoftAssertions` 的黑魔法：ByteBuddy 代理 + ErrorCollector

软断言为什么能把本该抛出的 `AssertionError` 憋到 `assertAll()` 才引爆？答案是**代理 + 上下文注入**双保险，`ErrorCollector` 是核心部件。

## 4.1 代理创建路径

`AbstractSoftAssertions` 继承 `DefaultAssertionErrorCollector`，后者持有核心列表 `List<Throwable> errorsCollected`；`SoftAssertions` 内部再持有一个 `SoftProxies` 代理工厂。当你调用 `softly.assertThat(str)`：

1. 工厂用 **ByteBuddy** 为目标 Assert 类（如 `StringAssert`）动态生成**子类代理**，并给代理类织入两个私有字段：`ProxifyMethodChangingTheObjectUnderTest`（处理返回新断言对象的导航方法）与 `ErrorCollector`（方法拦截器）；
2. `ErrorCollector.intercept` 包裹每一次方法调用：**先照常执行真实断言逻辑**——失败时真实对象照样抛 `AssertionError`，由拦截器捕获后 `collectAssertionError` 存入 errorsCollected（嵌套调用只让最外层收集一次）；
3. 拦截器返回代理自身（或按返回类型给 stub 值），链得以继续。

所以第 4 篇说"忘记 assertAll 就假绿灯"有了机器解释：**AssertionError 在代理边界就被吞掉，测试方法自然返回，Runner 判绿**。`assertAll()` 检查 errorsCollected：为空万事大吉，非空则拼装成 `AssertJMultipleFailuresError`（继承 OpenTest4J 的 `MultipleFailuresError`，失败分隔符正是 `-- failure N --`）抛出。`assertSoftly`、`AutoCloseableSoftAssertions`、JUnit5 扩展都只是**保证 assertAll 必然被执行**的三种护栏。

## 4.2 `assertAlso` 组合收集器

多个 SoftAssertions 实例可经 `softly.assertAlso(other)` 把对方的 errorsCollected 并入自己——原理即两个 List 的 addAll，用于"各子检查器自扫一摊，主测试统一引爆"的分层结构。

> **设计考量**：用字节码代理换来了"软断言与硬断言 API 完全同形"的体验，开发者不用学第二套方法名；代价是 assertj-core 附带 ByteBuddy 依赖（仅测试类路径，生产零影响）、每个被代理的 Assert 类要多生成一个子类（有 TypeCache 缓存摊销），以及"final 类/final 方法无法被继承式代理拦截"的机制上限——`satisfies(Consumer...)` 这类 final 方法走的是另一条 `satisfiesForProxy` 受保护桥接。对比 4.x 开发线：官方已在试验**去代理化**方案（断言体内置 executeAssertion 钩子直接上报 handler），说明代理是 3.x 在不破坏二进制兼容前提下的权衡，而非终点。

# 5. 一次调用生成的对象图与调试提示

`assertThat(users).extracting(User::getName).contains("Alice")` 静态类型流转：

```text
Assertions ──► ListAssert<User>(actual=users, info)
                   │ extracting(Function) : 内部用 Fields/Extractor 逐元素求值
                   ▼
               ListAssert<String>(actual=["Alice","Bob"], info′←info 拷贝)
                   │ contains : 委托 Iterables.assertContains
                   ▼
               成功→返回 myself;  失败→ShouldContain 产文案→Failures→抛/收集
```

> **行业实践**：读 AssertJ 源码排测试问题时按这条路径走——失败消息在 `org.assertj.core.error.Should*` 里 grep 模板句，回到 `internal` 执行器看判定逻辑，最后才到 API 层找是谁传错了参数；三级结构稳定，两小时能建立完整地图，比在测试栈里盲猜快一个数量级。

调试断言内部行为时两个开关有用：`setRemoveAssertJRelatedElementsFromStackTrace(true)`（默认开启，剔除 AssertJ 内部栈帧，让报错第一行落在你的测试代码）与 `Assertions.setDescriptionConsumer(...)`，后者可以把每条断言的 description 落盘，配合 `printAssertionsDescription` 实现**测试步骤审计日志**——把 `as()` 当业务步骤名用，这份日志就是免费的用例执行轨迹。

# 6. 小结

AssertJ 的架构记忆点收敛成一句话：**门面负责按类型造对象，AbstractAssert<SELF, ACTUAL> 用递归泛型撑住链式与自定义扩展，internal 执行器集中实现比较，Should* 错误工厂 + Failures 通道统一产出 OpenTest4J 结构化异常，AssertionInfo 作为贯穿一条链的上下文把描述、表示器与软断言收集器带到每一个失败点**。据此可以推出大量日常现象的解释：as 前缀为什么对整条链生效、软断言为什么必须收尾、下钻子断言为什么要手动搬描述、IDE diff 视图为什么能认出 AssertJ 的失败。带着这张图去读 `AbstractAssert` 与 `Failures` 两个类，其余部分皆是同构重复。
