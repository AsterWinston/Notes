# 1. 深度对象比较：`usingRecursiveComparison()` 家族

微服务分层体系里，`Entity`、`DTO`、`VO` 之间的映射与深拷贝（MapStruct、BeanUtils）是重灾区。传统 `isEqualTo()` 依赖每个类都正确重写 `equals()`，而 VO/DTO 层常常压根不想写这些胶水方法。AssertJ 的**递归字段比较**用反射遍历对象图，按字段名逐个比到底，完全不要求被测类型配合——这是 3.x 系列（3.16 引入 `usingRecursiveComparison`）相对旧版最大的生产力升级。

## 1.1 三兄弟的分工

同一个诉求有三个入口，粒度与语义各不相同，选错会得到意外结果：

| 方法 | 作用层级 | 语义 | 典型场景 |
|------|----------|------|----------|
| `usingRecursiveComparison()` | 单个对象 vs 另一个对象（**可不同类型**） | 按字段名递归比，叶子值走 equals | DTO 与 Entity 对账 |
| `usingFieldByFieldElementComparator()` | 集合的**元素**比较器 | 元素按顶层字段比（不深入） | 元素字段是 String/int 的列表 |
| `usingRecursiveFieldByFieldElementComparator()` | 集合的元素比较器 | 元素按字段**递归**比 | 嵌套对象列表对账 |

```java
package com.example.compare;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class RecursiveComparisonTest {

    record Address(String city, String zip) {}

    @Test
    void dto_against_entity_without_equals() {
        UserDTO dto = new UserDTO(null, "Alice", new Address("NYC", "10001"));
        UserEntity entity = new UserEntity(42L, "Alice", new Address("NYC", "10001"));

        // 错误示范: 两个类都没有 equals, isEqualTo 必挂
        // assertThat(dto).isEqualTo(entity);

        // 正确做法: 递归比较不关心类型是否相同, 只按字段名对齐取值
        // id 一个为 null 一个为 42 —— 默认也会比出来, 见 1.2 的忽略策略
        assertThat(dto).usingRecursiveComparison()
                       .ignoringFields("id")
                       .isEqualTo(entity);
    }

    @Test
    void collection_elements_field_by_field() {
        List<UserDTO> actual = service.exportUsers();
        List<UserDTO> expected = fixtureUsers();

        // 元素仍是 UserDTO, 但比较方式换成"逐字段"
        assertThat(actual).usingFieldByFieldElementComparator()
                          .isEqualTo(expected);

        // 需要深入嵌套的 Address 时, 换递归版元素比较器
        assertThat(actual).usingRecursiveFieldByFieldElementComparator()
                          .containsExactlyInAnyOrderElementsOf(expected);
    }
}
```

## 1.2 精细化控制：忽略、过滤、指定比较策略

真实业务对象上总有一批"不可断言"的属性：自增主键、UUID、创建时间、审计字段。`RecursiveComparisonAssert` 提供成体系的调节旋钮：

```java
assertThat(dto)
        .usingRecursiveComparison()
        // 1. 按名字/正则忽略字段
        .ignoringFields("id", "createdAt", "updatedAt")
        .ignoringFieldsMatchingRegexes(".*Audit$")
        // 2. 只比较白名单(与忽略互斥, 二选一)
        // .comparingOnlyFields("name", "address.city")
        // 3. 期望侧为 null 的字段跳过(常用于"只验证产出对象非空的公共部分")
        .ignoringExpectedNullFields()
        // 4. 类型不同但字段兼容时默认宽松; 开启严格类型检查后 name: String vs 5 直接判负
        // .withStrictTypeChecking()
        // 5. 对特定类型强制用给定比较策略而不是拆字段(如 BigDecimal 按 compareTo 屏蔽 scale 差异)
        .withEqualsForType((java.math.BigDecimal a, java.math.BigDecimal b) -> a.compareTo(b) == 0,
                java.math.BigDecimal.class)
        // 6. 忽略所有重写过 equals 的类(把 java.time、BigDecimal 等交还给 equals)
        .ignoringAllOverriddenEquals()
        // 7. 集合/Map 默认顺序敏感, 可全局关闭
        .ignoringCollectionOrder()
        .isEqualTo(expected);
```

> **设计争议**：递归比较的"按字段名对齐"是双刃剑——**它默认允许 DTO.name 与 Entity.name 类型不同且宽松匹配**，一旦两侧字段名巧合等价但语义不同（比如 `version` 一边是乐观锁计数、一边是schema 版本），测试会给出虚假的安全感；反过来，字段改名后测试悄悄从"比较"退化成"忽略"，也需要 review 兜底。合理姿势是：公共 DTO 契约测试开启 `withStrictTypeChecking()`，并让忽略清单显式、集中、可 review，禁止 `ignoringAllOverriddenEquals()` 一把梭。

# 2. Java 8+ 类型断言：Optional / Stream / 并发

## 2.1 Optional 与 Stream

```java
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

class Jdk8TypesTest {

    @Test
    void optional_assertions() {
        Optional<String> optional = Optional.of("Game of Thrones");

        assertThat(optional)
                .isPresent()                       // 是否有值
                .contains("Game of Thrones")       // 拆包并核对值
                .hasValueSatisfying(value -> {     // 拆包后下钻更丰富的验证
                    assertThat(value).startsWith("Game");
                });

        assertThat(Optional.<String>empty()).isEmpty();
        // 失败消息区分度很好: "Expecting Optional to contain a value but it was empty"
    }

    @Test
    void stream_assertions() {
        Stream<String> stream = Stream.of("Spring", "MyBatis", "Redis");

        // assertThat(Stream) 直接返回 ListAssert, 集合全家桶方法全部可用
        assertThat(stream)
                .hasSize(3)
                .extracting(String::length)
                .containsExactly(6, 7, 5);
    }
}
```

**Stream 是一次性消费资源**，AssertJ 在入口处立刻把它物化成列表，所以下面示例里对同一 stream 的多次失败重跑不会遇到"已关闭"问题，但也别指望断言后还能拿原 stream 继续操作。

## 2.2 CompletableFuture：非阻塞状态 + 限时取值

异步链路的断言分两类：**对 Future 状态本身**的断言（不阻塞）与**对 Future 结果**的断言（限时阻塞）。AssertJ 3.17+ 的 `succeedsWithin/failsWithin` 把后者变成了一等公民：

```java
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;

class FutureAssertionTest {

    @Test
    void completable_future() {
        CompletableFuture<String> future = CompletableFuture.completedFuture("Done");

        assertThat(future)
                .isDone()                              // 断言已完成(不等待)
                .isNotCancelled()
                .isNotCompletedExceptionally()
                // 主动等待最多 1 秒并取出结果, 返回值是结果的 ObjectAssert, 链继续
                .succeedsWithin(Duration.ofSeconds(1))
                .isEqualTo("Done");

        CompletableFuture<String> boom = new CompletableFuture<>();
        boom.completeExceptionally(new IllegalStateException("cb down"));
        assertThat(boom)
                .failsWithin(Duration.ofMillis(100))
                .withThrowableOfType(IllegalStateException.class)   // 拆出的就是导致失败的那条异常
                .withMessageContaining("cb down");
    }
}
```

> **注意**：`succeedsWithin` 会**真的阻塞测试线程**至超时；而 `isDone()` 只是看一眼状态。对"还没提交任务就先断言 isDone"的写法，得到的只会是 flaky。如果等待逻辑更复杂（轮询数据库、等 MQ 消费），用 Awaitility，见第 4 节。

# 3. 展示控制与类型收窄

## 3.1 `withRepresentation`：自定义对象在失败消息里的长相

失败消息里对象的 `toString()` 质量决定排错速度。AssertJ 把"怎么展示一个值"抽成 `Representation` 接口，提供从全局到单链的三级控制：

```java
import org.junit.jupiter.api.Test;

import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;

class RepresentationTest {

    @Test
    void per_type_formatter() {
        // 全局: 为某类型注册专用格式化函数, 只影响该类型的展示, 不动 equals 语义
        Function<Money, String> moneyFormatter =
                m -> m.getCurrency() + " " + m.getAmount().toPlainString();
        org.assertj.core.api.Assertions.registerFormatterForType(Money.class, moneyFormatter);

        // 效果: 失败消息里出现 "USD 199.90" 而不是 "Money@1a2b3c" 或整对象 JSON
        assertThat(price()).isEqualTo(new Money("USD", new java.math.BigDecimal("199.9")));
    }

    @Test
    void per_assertion_override() {
        // 单条链临时换表示器, 不污染全局(也可换自定义 Representation 实现)
        org.assertj.core.presentation.Representation standard =
                org.assertj.core.presentation.StandardRepresentation.STANDARD_REPRESENTATION;
        assertThat(hugeMap)
                .withRepresentation(standard)
                .containsKey("k");
    }
}
```

同一族的工具还有 `setMaxElementsForPrinting`（消息里最多打印多少个元素）、`setMaxLengthForSingleLineDescription`（超过则换行）、`setRemoveAssertJRelatedElementsFromStackTrace`（栈里剔除 AssertJ 内部帧）。**只在需要时动全局开关，且测试并行执行时全局配置存在跨用例可见性，优先选链上 `withRepresentation`。**

## 3.2 `asInstanceOf` 与 `extracting` 的强类型收窄

断言一个 `Object`/泛型槽位里的值时，先收窄再继续链，避免降级成 ObjectAssert 后方法消失：

```java
assertThat(payload)                                   // payload 是 Object
        .asInstanceOf(org.assertj.core.api.InstanceOfAssertFactories.MAP)
        .containsEntry("code", "OK");

// ObjectAssert 的字符串提取同样可指定工厂收窄类型
assertThat(order).extracting("items",
        org.assertj.core.api.InstanceOfAssertFactories.list(OrderItem.class))
        .flatExtracting(OrderItem::getName)
        .contains("sku-1");
```

`InstanceOfAssertFactories` 接口列全了所有现成工厂（LIST、MAP、STRING、OPTIONAL、THROWABLE、JSON 无关……），`Assertions` 直接实现它，所以静态导入 `Assertions.*` 后可裸用 `LIST`、`MAP` 这些常量（第 1 篇的入口家族图中已列）。

## 3.3 条件与逃生分支：`satisfiesAnyOf` / `assumeThat`

```java
// 满足任一分支即通过: 状态机里 "终态有两个" 的合法表达
assertThat(order).satisfiesAnyOf(
        o -> assertThat(o.getStatus()).isEqualTo(OrderStatus.SETTLED),
        o -> assertThat(o.getStatus()).isEqualTo(OrderStatus.CANCELLED));

// 前提不满足 => 跳过(ABORTED)而非失败: 环境性条件(依赖服务未启动、OS 不匹配)
org.assertj.core.api.Assumptions.assumeThat(featureFlag.enabled()).isTrue();
assertThat(result).isEqualTo("v2");
```

**assumeThat 是 JUnit `assumeTrue` 的 AssertJ 镜像**，失败语义是"跳过"，滥用会制造"CI 全灰但没人发现"的事故；它只回答"环境够不够格"，不回答"对不对"。

# 4. 生态协作：JUnit 5 / BDDMockito / Awaitility

## 4.1 与 JUnit 5 `assertAll` 协作

AssertJ 断言就是普通方法调用，天然可以塞进 `assertAll` 的 `Executable` 里获得聚合能力——这相当于"轻量级软断言"，断言数量不多时比管理 SoftAssertions 生命周期更省心：

```java
assertAll("下单结果",
        () -> assertThat(order.getId()).isNotNull(),
        () -> assertThat(order.getItems()).hasSize(2),
        () -> assertThat(order.getTotal()).isCloseTo(new BigDecimal("59.7"),
                org.assertj.core.api.Assertions.within(new BigDecimal("0.01"))));
```

与第 4 篇 `SoftAssertions` 的分工：**断言点固定、数量有限 → assertAll（参数即断言，不可能忘收尾）；断言点动态（循环、跨方法）→ SoftAssertions。**

## 4.2 BDD 风格与 Mockito 的 then/given 链

AssertJ 提供 `BDDAssertions`，把 `assertThat` 改名 `then`，与 Mockito 的 `given`(stub)/`when`（调用）/`then`（断言）节奏对齐：

```java
package com.example.bdd;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;

import static org.assertj.core.api.BDDAssertions.then;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class OrderServiceBddTest {

    @Mock PaymentGateway gateway;
    @InjectMocks OrderService service;

    @Test
    void create_order_pays_gateway() {
        given(gateway.pay(BigDecimal.valueOf(100))).willReturn("PAY-1");   // given: 布好替身

        String orderNo = service.createOrder(100);                          // when:  触发行为

        then(orderNo).isNotBlank();                                         // then:  断言结果
        then(gateway).isNotNull();                                          // then 同样适用任何类型
        verify(gateway).pay(BigDecimal.valueOf(100));                       // 行为验证仍归 Mockito
    }
}
```

还有 `BDDSoftAssertions`（then 版软断言）与 `BDDAssumptions.given`（前提假设的 BDD 形态）。

> **注意**：同一测试类里 `BDDAssertions` 与 `Assertions` 双静态导入是合法的（方法名 then 与 assertThat 不冲突），但一个用例内两种语感混排会打断阅读节奏——要么整类 BDD 化，要么只把 given/when 留给 Mockito 的 `BDDMockito`、断言坚持 assertThat。

要不要全面 BDD 化取决于团队：它让 AAA(Arrange-Act-Assert)结构在视觉上更整齐，代价是全员适应一套新动词。

## 4.3 Awaitility：异步系统的等待式断言

轮询等待（消息已消费、缓存已刷新、最终一致读）不要手写成环 sleep，交给 Awaitility，断言体里照常写 AssertJ：

```java
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

class AwaitilityComboTest {

    @Test
    void consumer_should_process_all_messages() {
        AtomicInteger consumed = new AtomicInteger();
        publisher.publishBatch(10, consumed::incrementAndGet);   // 异步投递, 回调累加

        // 轮询直到断言通过或超时; 断言体里照常写 AssertJ
        await().atMost(Duration.ofSeconds(5))
               .pollInterval(Duration.ofMillis(100))
               .untilAsserted(() -> assertThat(consumed.get()).isEqualTo(10));
    }
}
```

分工原则：`CompletableFuture`/`Future` 的超时断言用 AssertJ 自带的 `succeedsWithin`（单目标、语义内聚）；**任意共享状态的轮询收敛**用 Awaitility + AssertJ（可观测、可诊断线程堆栈）。

# 5. 时间与文件断言速查

两个高频领域，API 多而碎，列关键项备用：

```java
// java.time: 一律带容差或对齐单位, 禁止裸 isEqualTo 比时钟
LocalDateTime now = LocalDateTime.now();
assertThat(now).isBetween(now.minusMinutes(1), now.plusMinutes(1));
assertThat(eventTime).isCloseTo(expected,
        org.assertj.core.api.Assertions.within(2, java.time.temporal.ChronoUnit.MINUTES));
assertThat(deadline).isAfterOrEqualTo(now);

// File/Path: 内容级比较是杀手锏, 适合报表/配置回归
File file = new File("src/test/resources/out.csv");
assertThat(file).exists().isFile().canRead().hasExtension("csv");
assertThat(file).hasSameTextualContentAs(expectedFile);     // 文本逐行一致
assertThat(file.toPath()).hasSameContentAs(expectedPath);   // NIO 等价物
assertThat(path).exists().isRegularFile().startsWith(baseDir);
// 二进制用 hasBinaryContent(byte[])/hasDigest("MD5", "a1b2...")
```

# 6. 小结

高级特性的共同主题是**让框架替你做反射与等待，但控制权留在你手里**：`usingRecursiveComparison()` 家族用反射遍历对象图，三个入口（对象级递归、元素级浅比、元素级递归）按比较深度选，忽略清单要显式可 review，类型漂移靠 `withStrictTypeChecking()` 兜底；Optional/Stream/CompletableFuture 断言把 `isPresent/succeedsWithin` 变成链式原语，记住 `succeedsWithin` 是真阻塞；`withRepresentation` 与 `registerFormatterForType` 管理失败消息的"长相"，单链覆写优先于全局开关；`assertAll`、`BDDAssertions`、Awaitility 分别解决聚合、语感、异步等待三个协作问题，与 AssertJ 本体组合而非替换。
