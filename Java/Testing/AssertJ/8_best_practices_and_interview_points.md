# 1. 选型与落地建议

AssertJ 是服务端 Java 断言的事实标准，但"用不用、怎么混用、怎么迁移"仍是团队级决策。本篇收束整个目录：先给选型与工程纪律，再按面试题形式提炼考察点，最后串起前七篇的知识地图。

## 1.1 选型决策

**默认结论：新项目的 JVM 后端测试，断言层直接选 AssertJ，不需要犹豫**。它在强类型链式 API、JDK 类型覆盖度（日期、IO、并发、Optional）、失败消息质量、软断言与自定义扩展四个维度全面占优，且被 `spring-boot-starter-test` 默认携带——框架都替你选了，逆着来才需要理由。

需要权衡的场景：

| 场景 | 建议 | 理由 |
|------|------|------|
| Spring Boot 后端服务 | AssertJ（随 starter） | 零决策成本，类型覆盖最全 |
| Android / Guava 重度项目 | Google Truth | 与 Android 测试设施、proto 集成更好 |
| 只测纯函数、断言极少 | JUnit 5 原生足够 | 引入库的收益盖不过认知成本 |
| 遗留工程大规模 JUnit4/Hamcrest 断言 | 渐进迁移（见 1.3） | OpenRewrite 批量改写，不推翻重来 |

## 1.2 混用 JUnit 5 断言与 AssertJ 的纪律

同一个工程两套断言并存是现实（尤其迁移期），可以共存但必须有纪律，否则 code review 会退化成风格战争：

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.junit.jupiter.api.Assertions.assertAll;
import static org.junit.jupiter.api.Assertions.assertTimeout;

class DisciplineTest {

    @Test
    void mixed_but_ordered() {
        // 纪律 1: 值断言只用 AssertJ, 不许再出现 assertEquals/assertTrue
        var report = service.build();
        assertThat(report.getRows()).hasSize(120);
        assertThat(report.getTitle()).startsWith("Q3");

        // 纪律 2: AssertJ 没有的能力留在 JUnit(超时、动态跳过、条件执行)
        // assertTimeout / assumeTrue / @DisabledIf 属于执行控制, 不是值断言

        // 纪律 3: 聚合可以借 JUnit assertAll 包住 AssertJ 断言(见第 6 篇),
        // 但同一方法内不要 assertAll、SoftAssertions、原生断言三套并排
        assertAll("汇总行",
                () -> assertThat(report.getTotal()).isPositive(),
                () -> assertThat(report.getFooter()).isNotBlank());
    }
}
```

三条纪律展开：**其一，值断言单一来源**——`assertEquals` 与 `isEqualTo` 并存时，失败消息风格、参数顺序风险也随之并存，最糟的是新人会照抄错误示范；**其二，按能力划界**——超时（`assertTimeout`/`@Timeout`）、假设（`assumeTrue`）、动态测试（`@ParameterizedTest`）归 JUnit，值校验归 AssertJ，各守其位；**其三，禁止反向混用**——不要在 AssertJ 链里塞 `assertTrue(list.contains(x))` 这种降级写法，那是把 AssertJ 当 JUnit 用。

## 1.3 老旧系统自动化重构策略 (OpenRewrite)

JUnit4/5 与 Hamcrest 存量代码基数庞大，人工改写几十万行断言缺乏性价比。**最佳实践：用 OpenRewrite 做 AST（抽象语法树）级批量重构**，在不改变验证语义与类型推导安全的前提下，完成正则根本做不到的上万处改写：

```xml
<plugin>
  <groupId>org.openrewrite.maven</groupId>
  <artifactId>rewrite-maven-plugin</artifactId>
  <version>5.20.0</version>
  <configuration>
    <activeRecipes>
      <!-- 把 JUnit Assert / Hamcrest Matcher 调用智能改写为 AssertJ 流式断言 -->
      <recipe>org.openrewrite.java.testing.assertj.JUnitToAssertj</recipe>
    </activeRecipes>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>org.openrewrite.recipe</groupId>
      <artifactId>rewrite-testing-frameworks</artifactId>
      <version>2.5.0</version>
    </dependency>
  </dependencies>
</plugin>
```

终端触发 `mvn rewrite:dryRun` 先看 diff，确认后 `mvn rewrite:run` 落盘。注意 recipe 对 `assertTrue(a.equals(b))` 这类"翻译后语义等价但可读性未必更好"的改写要人工抽检，机器负责量，人负责质。

> **设计争议**：也有团队反对全库迁移——JUnit 原生断言零依赖、OpenTest4J 消息结构近年也补齐了 expected/actual，而迁移会产生一次巨大的 diff 噪声，污染 git blame。折中方案是"增量转换"：新代码强制 AssertJ，存量只转换断言失败消息难读的热点文件，不追求一次性全量。

## 1.4 失败信息可读性即文档

**断言库的第一用户是半年后凌晨三点读 CI 日志的人**。这个视角能推出四条工程约定：

- **复合断言必写 `as("业务语义")`**：失败前缀 `[余额应已扣减]` 值十行 stacktrace；软断言聚合报告里 as 更是唯一的路标（第 4 篇）。
- **别用 `assertTrue(复杂表达式)` 折叠语义**：失败消息只有 expected true but was false，什么也没说（第 2 篇）。
- **失败消息太裸的对象先治理展示**：给类型注册 formatter 或改善 toString，而不是给断言加 `overrideErrorMessage` 打补丁（第 6 篇）。
- **把 `as()` 当用例步骤文档**：开启 printAssertionsDescription 后，断言描述流就是自然语言写的验收清单。

> **行业实践**：不少团队在 checkstyle/ArchUnit 层加两条硬约束：测试代码禁止导入 `org.junit.jupiter.api.Assertions` 的值断言方法（白名单 assertAll/超时/假设），`assertTrue` 仅允许直接作用于布尔 getter 结果。纪律靠工具固化，不靠 review 时的心情。

# 2. 与 Mockito 的组合分工

**AssertJ 管"数据"（结果状态），Mockito 管"行为"（交互与替身）**——verify 回答"调没调、调几次、参数什么"，assertThat 回答"拿到的值对不对"，企业测试里两者成对出现。经典组合是 ArgumentCaptor 捕获下游请求后交给 AssertJ 深检：

```java
package com.example.order;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class OrderCreationTest {

    @Mock PaymentClient paymentClient;
    @InjectMocks OrderService orderService;

    @Test
    void should_pay_exact_amount() {
        given(paymentClient.pay(org.mockito.ArgumentMatchers.any()))
                .willReturn("PAY-OK");

        orderService.createOrder(cartWithTotal("199.90"));

        ArgumentCaptor<PaymentRequest> captor = ArgumentCaptor.forClass(PaymentRequest.class);
        verify(paymentClient, times(1)).pay(captor.capture());   // 行为: 恰好调用一次

        // 数据: 捕获的 DTO 全维度流式审查(错误示范: assertEquals(req.getAmount().toString(), "199.9"))
        assertThat(captor.getValue())
                .extracting(PaymentRequest::getAmount, PaymentRequest::getCurrency)
                .containsExactly(new BigDecimal("199.90"), "CNY");
    }
}
```

> **注意**：`verify(...).pay(argThat(...))` 也能验参数，但 Hamcrest 式内联 argThat 失败消息极差，Mockito 的 `argThat` 混进多个 matcher 后报错只说"Wanted but not invoked"。成熟做法是**行为 verify 只管次数与目标，参数一律 Captor 捕获后移出 Mockito 交给 AssertJ**——报错可读性立刻不同。

# 3. 面试题精选

## 3.1 isEqualTo 和 isSameAs 有什么区别？各在什么场景用？

`isEqualTo` 走 `equals()` 比较内容，`isSameAs` 走 `==` 比较引用同一性。断言值对象（金额、DTO、集合内容）用 isEqualTo；断言单例、缓存命中（"两次 get 必须返回同一实例"）、对象复用池这类**身份契约**时必须 isSameAs——用 isEqualTo 会漏掉"返回了内容相同的新对象"这个真 bug。反过来，对没重写 equals 的类用 isEqualTo 会永远失败，且失败消息里两个对象 toString 一模一样，这是新手最常见困惑，排查顺序是先看被测类的 equals 实现，再怀疑断言。

## 3.2 containsExactly、contains、containsOnly 三者的语义差异？

按"顺序是否敏感、允不允许多余元素、是否在意重复"三个维度区分：contains 只要求"至少有这些"，顺序与多余元素都允许；containsExactly 要求元素、顺序、个数完全一致，顺序本身是被测契约（分页排序、编译产物行序）时才用；containsOnly 要求"只能是这些"但不检查顺序与重复次数；还有 containsExactlyInAnyOrder 是"精确多重集合、忽略顺序"。对无顺序契约的查询结果写 containsExactly，等于给 CI 埋随机失败的雷——判断标准只有一条：顺序是需求承诺的，还是实现碰巧给的。

## 3.3 AssertJ 和 Hamcrest 都是断言库，面试怎么答差异？

两者同源（Hamcrest 首创 matcher 思想，JUnit 的 assertThat 即桥接 Hamcrest），分野在工程体验：Hamcrest 的期望逻辑封装为 Matcher 对象，靠泛型静态方法组合，IDE 补全差、复杂条件 allOf/anyOf 嵌套成括号地狱、类型安全靠 TypeSafeMatcher 约定；AssertJ 把 actual 前置进 assertThat 入口，返回按 actual 类型特化的 Assert 对象，链上方法由类型系统给出，补全即文档，失败消息带结构化 expected/actual（走 OpenTest4J，IDE 可弹 diff 视图），且独有软断言、异常链式断言、递归字段比较等能力。一句话：Hamcrest 是 matcher 组合子库，AssertJ 是把"断言"做成了一门按类型组织的小语言。

## 3.4 assertThatThrownBy 和 JUnit4 的 @Test(expected=...) 比好在哪？

三个层次：范围——注解作用于整个测试方法，任何一行抛出该类型异常都算通过，可能验证的根本不是目标调用，assertThatThrownBy 用 lambda 把"允许抛异常"收窄到被测那一行；深度——注解只能验类型，AssertJ 拿到异常对象后可以继续 hasMessageContaining、hasCauseInstanceOf、rootCause()、hasFieldOrPropertyWithValue 核对消息、原因链与携带字段，这些本身就是契约；假绿灯——lambda 没抛异常时链上立即失败（"Expecting code to raise a throwable"），而手写 try-catch 忘调 fail 会让"没抛异常"静默通过。JUnit5 的 assertThrows 解决了范围与返回值，但对消息/原因要另写多行断言，链式一体感仍是 AssertJ 占优。

## 3.5 软断言是什么？底层怎么实现？什么时候不该用？

软断言把 fail-fast 改造成"收集—合并—统一引爆":SoftAssertions 用 ByteBuddy 为真实 Assert 对象生成代理，断言失败产生的 AssertionError 被代理边界的 ErrorCollector 拦截，存入 errorsCollected 列表，链照常返回代理自身继续执行；assertAll() 时若非空，把各条失败拼装成 AssertJMultipleFailuresError（继承 OpenTest4J MultipleFailuresError）一次性抛出。软/硬的区分不在断言上下文里，而在调用的是代理对象还是裸对象——代理吞错收集，裸对象直接上抛。该用：同一对象多个平行字段、循环内逐元素、昂贵冒烟检查要一次看清全部偏差。不该用：断言间有依赖（短路正是想要的）、前置条件（必须先硬断言非空，否则级联 NPE 污染报告）、以及拿它容忍已知失败——那是掩盖问题不是聚合报告。手搭形态忘调 assertAll 会假绿灯，所以默认用 assertSoftly lambda 或 JUnit5 的 SoftAssertionsExtension。

## 3.6 isEqualTo 比较自定义对象总是失败，为什么？有哪些解法？

因为 isEqualTo 委托 equals，而该类没重写 equals（还是 Object 的引用比较）。解法按代价排序：给值对象补 equals/hashCode（或 record/Lombok @Data，治本）；DTO 与 Entity 跨类型对账用 usingRecursiveComparison().isEqualTo()，反射按字段名递归比，必要时 ignoringFields 剔除 id/createdAt 这类不稳定字段，配合 withStrictTypeChecking 防类型漂移；集合元素级用 usingFieldByFieldElementComparator（只一层）或 usingRecursiveFieldByFieldElementComparator（递归）；只关心个别属性时 extracting(User::getName) 投影后再断言——报"哪个字段错"最精确。反模式是给被测类塞一个只为测试服务的 toString 然后 assertTrue(a.toString().equals(b.toString()))。

## 3.7 如何为领域对象写自定义断言？泛型为什么是 AbstractAssert<UserAssert, User>？

继承 AbstractAssert，第一个泛型 SELF 指向子类自身（递归泛型），保证基类方法（isNotNull、as）返回后静态类型不降级，链上能继续点出业务方法；第二个泛型 ACTUAL 是被测对象类型，构成编译期"类型即文档"。模板：构造器里 super(actual, UserAssert.class)；提供静态 assertThat(User) 工厂入口；每个业务断言先 isNotNull() 防 NPE，失败用 failWithMessage 同时给期望与实际，成功返回 this。若返回子对象继续链，要手动搬运 as 描述——内置导航用的上下文桥接方法是包私有的，第三方断言够不着。一次性规则用 Condition 更轻，上百实体交给 assertj-assertions-generator 生成器，但生成的是机械字段断言，领域语义（如 isPaid 含状态机）仍要手写。

## 3.8 AssertJ 断言失败抛出的是什么异常？为什么测试框架都能接住？

3.x 主要抛 org.opentest4j.AssertionFailedError：内部工厂反射构造，OpenTest4J 不在类路径时回退普通 java.lang.AssertionError，两者都是 AssertionError 的后裔。能被任意框架识别的原因：JUnit/TestNG/Surefire/IDE 约定在 Throwable 层拦截，AssertionError 谱系会被归类为"断言失败（FAILURE）"而非"执行错误（ERROR）"，报告与红绿标识据此生成；OpenTest4J 的异常还携带结构化 expected/actual 值对象，所以 IDE 能渲染左右 diff 视图。选择 Error 而非 Exception 也刻意为之——业务代码的 catch(Exception) 不会误吞断言失败。

## 3.9 assertThat 编译报 ambiguous method call，怎么回事？

被测对象是"有界泛型且同时可能匹配多个 assertThat 重载"的类型（如 T extends Throwable & List 一类，或 lambda 结果），Java 8 决议无法在 assertThat(Throwable) 与 assertThat(Iterable) 之间选择。这正是官方把入口拆成 AssertionsForClassTypes 与 AssertionsForInterfaceTypes 再由 Assertions 聚合的原因。解法：换用无歧义的具名入口 assertThatObject/assertThatList/assertThatIterable/assertThatPredicate，或强转收敛静态类型 (Object) value，异常场景直接走 assertThatThrownBy。

## 3.10 浮点数/时间/金额断言怎么写才稳？

浮点禁 isEqualTo(0.1+0.2 != 0.3)，用 isCloseTo(0.3, within(1e-4)) 或 byLessThan 表达严格不等；金额用 BigDecimal 并认清 equals 与 compareTo 的分裂（1.0 与 1.00 equals 为 false 但 compareTo 为 0），断言首选 isEqualByComparingTo，递归比较场景用 withEqualsForType 把 BigDecimal 比较策略换成 compareTo 屏蔽 scale 差异（1.0 与 1.00 equals 为 false 但 compareTo 为 0）；时间禁裸比时钟，用 isCloseTo(expected, within(2, ChronoUnit.MINUTES))、isBetween 或截断到秒后比较；异步结果用 succeedsWithin(Duration) 给等待上限，共享状态轮询交给 Awaitility 的 untilAsserted 包住 AssertJ 断言。

# 4. 小结：八篇知识地图

本目录从"为什么"走到"怎么运转"再走到"怎么用对"：第 1 篇立论——AssertJ 以强类型 + 流式 + 结构化失败消息三板斧成为断言层事实标准；第 2 篇铺 API 主干道——isEqualTo/isSameAs 的身份与内容之辨、contains 家族的顺序语义、extracting 投影与 isTrue 反模式；第 3 篇解决"失败的契约"——assertThatThrownBy 默认、类型前置变体、catchThrowable 只留给分支控制流；第 4 篇把 fail-fast 改造成收集-引爆——三档收尾护栏与"平行字段簇才许软化"的边界；第 5 篇给出 Condition → AbstractAssert → 生成器的四级扩展阶梯；第 6 篇覆盖深水区——递归比较、表示器、与 assertAll/BDD/Awaitility 的协作；第 7 篇回收所有"为什么"：门面分派、递归泛型、AssertionInfo 上下文、Should* 错误工厂与代理+ErrorCollector 的软断言实现；本篇收敛为选型纪律与面试考点。**一句话贯穿全目录：断言库的价值不在"能判断"，而在"判断错了的时候，消息能不能替凌晨三点的你省下三十分钟"——写断言时按这个标准自律，读 AssertJ 源码时按这个标准理解它的所有设计。**
