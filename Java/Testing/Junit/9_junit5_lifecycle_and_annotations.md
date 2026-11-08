# 1. 生命周期概览

JUnit 5 的测试生命周期由五个回调注解构成：`@BeforeAll` → `@BeforeEach` → `@Test` → `@AfterEach` → `@AfterAll`，与 JUnit 4 的 `@BeforeClass`/`@Before`/`@Test`/`@After`/`@AfterClass` 一一映射，改名是为了让"作用于每个测试"还是"作用于整个类"从名字上就能读出来。语义上真正的变化有两处：**默认每个测试方法创建一个全新的测试类实例**（与 JUnit 4 相同，但 JUnit 5 提供了改变它的开关），以及**生命周期方法可以出现在继承链与嵌套类的多个层级上并按确定顺序组合**（JUnit 4 做不到）。

## 1.1 生命周期顺序

一个包含两个测试方法的类，完整的执行时序如下：

```text
┌────────────────────────────────────────────┐
│ @BeforeAll（静态，整类一次）                 │
├────────────────────────────────────────────┤
│ 测试方法 A：                                │
│   @BeforeEach → @Test A → @AfterEach       │
│ 测试方法 B：                                │
│   @BeforeEach → @Test B → @AfterEach       │
├────────────────────────────────────────────┤
│ @AfterAll（静态，整类一次）                  │
└────────────────────────────────────────────┘
```

每个 `@Test` 都被一对 `@BeforeEach`/`@AfterEach` 包裹；`@BeforeAll`/`@AfterAll` 在全部测试之外各执行一次。几个关键的异常语义值得单独记住：

- **`@BeforeEach` 抛异常，对应的 `@Test` 不再执行，但 `@AfterEach` 仍会执行**——资源清理不会被前置失败跳过。
- **`@Test` 失败不影响同类的其他测试方法**，后续方法照常走自己的 Before/After。
- **`@AfterEach` 抛异常，该异常会与测试本身的失败一起上报**，不会互相吞掉。
- 父类的 `@BeforeAll` 先于子类执行，`@AfterAll` 则相反（后声明者先执行），继承链上的回调按"由外向内进、由内向外出"的栈式顺序组合。

## 1.2 TestInstance 生命周期

JUnit 5 用 `@TestInstance(Lifecycle.XXX)` 显式声明测试实例的创建策略，两种取值：

**PER_METHOD（默认）**：每个测试方法执行前新建一个测试类实例，方法之间不共享任何实例字段。优点是测试天然隔离，一个方法改了字段不会波及下一个；代价是实例上无法缓存跨方法复用的昂贵对象。

**PER_CLASS**：整个测试类只创建一个实例，所有测试方法共享实例字段。它解锁了两件事——`@BeforeAll`/`@AfterAll` **可以写成非静态实例方法**（能访问依赖注入的字段），以及"先创建、再校验、最后清理"这类有状态的场景化测试成为可能：

```java
package com.example.service;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;

import static org.junit.jupiter.api.Assertions.assertEquals;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)   // 整类共用一个实例
class UserServiceTest {

    private UserService service;                   // 跨方法共享的实例字段

    @BeforeAll
    void setUpAll() {
        // PER_CLASS 下 BeforeAll 可以是非静态方法，直接使用实例字段
        service = new UserService(new FakeUserRepository());
    }

    @Test
    void should_create_user() {
        assertEquals(1, service.create("alice"));
    }

    @AfterAll
    void tearDownAll() {
        service.close();                           // 实例方法里可访问共享资源
    }
}
```

| 维度 | PER_METHOD（默认） | PER_CLASS |
|------|--------------------|-----------|
| 实例数量 | 每个测试方法一个 | 整类一个 |
| 字段共享 | 方法间完全隔离 | 所有方法共享，可能互相污染 |
| @BeforeAll/@AfterAll | 必须 static | 可以是实例方法 |
| 与 Spring 集成 | 每方法注入一次 | 注入一次即可，@BeforeAll 可用注入的 Bean |
| 典型场景 | 绝大多数无状态单元测试 | 有状态场景测试、需要实例级夹具的集成测试 |

> **注意**：PER_CLASS 把"字段污染"的风险交还给了开发者——一个测试方法修改了共享字段而忘记还原，失败会出现在后续无关的测试里，且随执行顺序漂移，非常难查。Spring 项目常被推荐 PER_CLASS（这样 `@BeforeAll` 能拿到注入的 Bean），但更稳妥的做法是保持 PER_METHOD，把类级资源放进 static 字段，由 `@BeforeAll` 静态方法初始化。

## 1.3 构造器与依赖注入

PER_METHOD 下测试类的构造器会被调用 N 次（N = 测试方法数），构造器里可以放轻量初始化，也可以声明参数由 Extension 注入——`TestInfo` 是最常用的内置注入，能拿到当前测试的元数据：

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInfo;

class OrderTest {

    private final TestInfo testInfo;

    // 构造器参数由 ParameterResolver 扩展注入（TestInfo 是内置支持的）
    OrderTest(TestInfo testInfo) {
        this.testInfo = testInfo;
    }

    @Test
    void should_calc_total() {
        System.out.println("当前测试: " + testInfo.getDisplayName());
    }
}
```

# 2. 常用注解

## 2.1 @Test / @RepeatedTest

`@Test` 来自 `org.junit.jupiter.api`，标注的测试方法**不能是 private 或 static**，不要求 public，返回类型必须为 void。与 JUnit 4 最大的兼容性差异是**没有 expected/timeout 属性**——异常断言改用 `assertThrows`，超时改用 `@Timeout` 注解或 `assertTimeout`，职责从注解参数移到了断言 API（见第 10 篇）。

`@RepeatedTest` 把同一方法重复执行固定次数，每次执行都走完整的 BeforeEach/AfterEach 周期，适合验证偶发性问题（并发缺陷、随机种子敏感性）；`RepetitionInfo` 参数可注入当前轮次：

```java
@RepeatedTest(3)
void should_work_repeatedly(RepetitionInfo info) {
    // 每次执行打印 1/3、2/3、3/3，报告里每轮是独立节点
    System.out.println(info.getCurrentRepetition() + "/" + info.getTotalRepetitions());
}
```

## 2.2 @BeforeAll / @AfterAll

类级前置与后置，默认要求 `static`（PER_METHOD 下类实例尚未创建，静态是唯一选择）；PER_CLASS 下可写实例方法。继承体系里父类先 Before、后 After，顺序确定且文档化，这与 JUnit 4 中 `@BeforeClass` 在继承链上的行为一致，但 JUnit 5 额外允许它出现在接口 default 方法中（配合 PER_CLASS）。

## 2.3 @BeforeEach / @AfterEach

方法级前置与后置，是准备测试夹具的主战场。多个来源的 BeforeEach 按"父类 → 子类 → 嵌套外层 → 嵌套内层"的顺序执行，AfterEach 严格逆序。接口里的 default 方法标注 `@BeforeEach` 也会被发现——JUnit 5 支持"测试夹具接口"模式，把公共的 setup 逻辑抽进接口让测试类实现，这是 JUnit 4 做不到的复用手段。

## 2.4 @DisplayName

为类和方法提供报告中的展示名，可以是空格分隔的自然语言甚至 emoji。它的价值在报告可读性：方法名受标识符约束（`should_return_404_when_user_not_found`），DisplayName 则没有——"用户不存在时应返回 404"。IDE 测试树、Surefire 报告、Allure 里显示的都是 DisplayName：

```java
@DisplayName("订单服务")
class OrderServiceTest {

    @Test
    @DisplayName("库存不足时下单应抛出异常并回滚")
    void should_throw_when_out_of_stock() {
        // 方法名保持英文驼峰，报告名用中文描述行为
    }
}
```

## 2.5 @Nested

`@Nested` 标注非静态内部类，把"同一被测对象在不同上下文中的行为"组织成树。嵌套类**继承外层类的生命周期回调**（外层的 BeforeEach 先于内层执行），并能访问外层实例字段（PER_METHOD 下每个内层测试也会触发外层实例创建），因此非常适合 Given-When-Then 式的场景分层：

```java
package com.example.order;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

@DisplayName("订单状态机")
class OrderTest {

    private Order order;

    @BeforeEach
    void setUp() {
        order = new Order();                    // 外层夹具，所有嵌套类共享此逻辑
    }

    @Nested
    @DisplayName("已支付时")
    class WhenPaid {

        @BeforeEach
        void payOrder() {
            order.pay();                        // 内层夹具在外层之后执行
        }

        @Test
        @DisplayName("可以发货")
        void can_ship() {
            assertTrue(order.canShip());
        }

        @Test
        @DisplayName("不能取消")
        void cannot_cancel() {
            assertFalse(order.canCancel());
        }
    }

    @Nested
    @DisplayName("已发货时")
    class WhenShipped {

        @BeforeEach
        void shipOrder() {
            order.pay();
            order.ship();
        }

        @Test
        @DisplayName("不能再次发货")
        void cannot_ship_again() {
            assertFalse(order.canShip());
        }
    }
}
```

报告中的测试树与代码结构同形（订单状态机 → 已支付时 → 可以发货），这是 `@Nested` 的核心收益。注意嵌套类**不能是 static**（static 内部类不会被当作 Nested 处理），且嵌套深度不限，但超过两层通常意味着该拆成独立测试类了。

## 2.6 @Disabled

跳过测试并可附带原因，报告里显示为 skipped 而非静默消失：

```java
@Test
@Disabled("等待支付网关 mock 就绪，见 JIRA-1234")   // 原因必填是好习惯，关联工单更佳
void should_call_payment_gateway() { }
```

`@Disabled` 可标注在类上（跳过整类）。**没有原因的 @Disabled 是测试腐化的开始**——半年后没人知道它为什么被禁用、能不能删，所以团队规范里通常要求附工单号。更进一步的方案是用 `@DisabledIfSystemProperty`、`@EnabledOnOs` 等条件注解替代无期限禁用（见第 13 篇）。

## 2.7 @Timeout

给方法设置执行时限，超时即失败，取代 JUnit 4 的 `@Test(timeout = ...)`：

```java
@Test
@Timeout(value = 500, unit = TimeUnit.MILLISECONDS)   // 超过 500ms 判定失败
void should_respond_fast() {
    service.query();
}
```

超时检测在独立线程观察、被测代码仍在原线程运行（与 `assertTimeoutPreemptively` 的强制中断不同），因此不会破坏线程上下文，适合给整个类的慢化退化设防线——标在类上即对所有方法生效。

# 3. 执行顺序

## 3.1 @TestMethodOrder

JUnit 5 默认**不保证**测试方法的执行顺序（只承诺"确定但有意不透明"，防止测试间隐式依赖）。确实需要固定顺序时（有状态的场景链、按成本递增排序），用 `@TestMethodOrder` 显式声明策略：

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)   // 按 @Order 值升序
class MigrationTest {

    @Test @Order(1)
    void create_table() { }

    @Test @Order(2)
    void backfill_data() { }

    @Test @Order(3)
    void switch_traffic() { }
}
```

内置策略有四种：`OrderAnnotation`（按 `@Order` 数值）、`MethodName`（按方法名字典序）、`DisplayName`（按显示名）、`Random`（每次随机——用来**暴露**顺序依赖，排查"单独跑绿、一起跑红"的问题非常好用）。类级顺序则用 `@ClassOrderer`。

> **设计争议**：官方立场是"测试应当互相独立，需要排序本身就是坏味道"，`MethodOrderer` 的 Javadoc 也明说 Random 之外都应谨慎使用。但现实中数据库迁移脚本测试、有状态工作流测试确实依赖顺序，硬拆反而扭曲领域表达。折中实践是：允许 `@Order` 表达顺序，但要求每个测试自带前置状态构造（不依赖前一个测试的副作用），顺序只影响可读性不影响正确性。

## 3.2 @TestFactory 简介

`@TestFactory` 把一个方法的返回值变成运行时动态生成的测试节点，返回 `DynamicTest` 流。它与 `@ParameterizedTest` 的区别是：**参数化测试的数据集在发现阶段就确定，动态测试的用例数量和内容可以直到执行时才计算出来**——适合"遍历目录下所有样本文件各生成一个用例"这类编译期无法枚举的场景：

```java
@TestFactory
Stream<DynamicTest> should_parse_all_fixtures() throws IOException {
    // 执行时扫描 fixture 目录，每个文件生成一个动态测试
    return Files.list(Path.of("src/test/resources/fixtures"))
            .map(path -> DynamicTest.dynamicTest(
                    path.getFileName().toString(),            // 报告中的用例名
                    () -> assertNotNull(parser.parse(path)))); // 用例体
}
```

注意动态测试内部**不走 BeforeEach/AfterEach 之外的实例级回调语义**（每个 DynamicTest 共享宿主方法的一次生命周期），复杂场景优先考虑 `@ParameterizedTest`，`@TestFactory` 留给真正运行时才能确定用例集的场景（第 11 篇详述）。

# 4. 组织结构建议

## 4.1 AAA 结构

测试方法体推荐 Arrange-Act-Assert 三段式：准备夹具、执行被测行为、断言结果，段落之间空行分隔。三段式最大的价值是**一眼看出"这个测试到底在验证什么"**——Act 只应有一个动作，Assert 只围绕该动作的结果；如果 Assert 部分膨胀到验证十几个无关属性，说明测试职责过载，应该拆分或改用 `assertAll` 分组表达：

```java
@Test
void should_apply_discount_for_vip() {
    // Arrange：准备夹具
    User vip = new User("alice", Level.VIP);
    Cart cart = new Cart().add(item(100));

    // Act：执行唯一被测行为
    Order order = orderService.place(vip, cart);

    // Assert：断言行为结果
    assertEquals(90, order.getTotal());
}
```

## 4.2 命名规则

方法名推荐 `should_xxx_when_yyy` 或 `行为_条件_期望` 的完整语义命名，配合 `@DisplayName` 提供人类可读版本（尤其中文团队直接写中文描述）。命名要回答"什么条件下做什么得到什么"，避免 `test1`、`testMethod` 这类无信息名——测试失败时，报告里唯一能看的就是名字，名字即文档。

# 5. 小结

| 注解/概念 | 作用 | 关键点 |
|-----------|------|--------|
| @BeforeAll / @AfterAll | 类级前置后置 | 默认 static；PER_CLASS 下可实例方法 |
| @BeforeEach / @AfterEach | 方法级前置后置 | BeforeEach 失败不跳过 AfterEach；继承链父先子后 |
| @Test | 测试方法 | 无 expected/timeout 属性，改用 assertThrows/@Timeout |
| @RepeatedTest | 重复执行 | 每轮独立走完整生命周期，可注入 RepetitionInfo |
| @TestInstance | 实例策略 | PER_METHOD 默认隔离；PER_CLASS 共享字段但引入污染风险 |
| @DisplayName | 报告展示名 | 自然语言/中文，方法名保持代码风格 |
| @Nested | 嵌套场景分层 | 非静态内部类，继承外层生命周期，报告呈树形 |
| @Disabled | 跳过 | 必须附原因/工单号，防测试腐化 |
| @Timeout | 方法时限 | 独立线程观察，不中断被测线程 |
| @TestMethodOrder | 方法顺序 | 默认不保证顺序；OrderAnnotation 显式排序，Random 用于暴露依赖 |
| @TestFactory | 动态测试 | 用例集执行时才确定，常规参数化优先 @ParameterizedTest |
| AAA 结构 | 方法体组织 | Arrange-Act-Assert，Act 单一动作，Assert 聚焦结果 |
