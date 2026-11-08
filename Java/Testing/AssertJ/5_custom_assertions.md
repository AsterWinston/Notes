# 1. 为什么需要自定义断言

标准类型（String、int、Iterable）的断言覆盖不了**业务语境**。当团队反复写 `assertThat(user.getAge()).isGreaterThanOrEqualTo(18)`、`assertThat(order.getStatus().name()).isEqualTo("PAID")` 时，断言里泄漏的是 getter 结构与字符串字面量，而不是领域概念。理想形态是直接说人话：`assertThat(order).isPaid().hasAmountGreaterThan(100)`。

AssertJ 为这种诉求提供了从轻到重的四级扩展点，选型顺序应当**从最便宜的开始**：

| 级别 | 手段 | 成本 | 适用 |
|------|------|------|------|
| 1 | `has/are/allSatisfy` + 既有断言 | 零 | 一次性判断，不值得封装 |
| 2 | `Condition<T>` | 极低 | 可命名、可复用的布尔谓词 |
| 3 | 通用逃生口 `hasFieldOrPropertyWithValue` 等 | 低 | 临时深挖，不建类 |
| 4 | 继承 `AbstractAssert` 建专用 Assert 类 | 中（有代码生成器） | 领域对象在测试里高频出现 |

# 2. 轻量级方案： `Condition<T>`

如果只是想增加某个规则判断而不想写继承类，AssertJ 的 `has(Condition)` 体系是不二之选——概念上等价于 Hamcrest 里写一个 Matcher，但 API 更收敛。

## 2.1 定义与使用

```java
package com.example.condition;

import org.assertj.core.api.Condition;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

public class CustomConditionTest {

    // 泛型为待检查的对象类型; 第二个参数(describeWith)用于失败时高亮展示期望
    static final Condition<User> seniorCitizen = new Condition<>(
            user -> user.getAge() >= 65,
            "senior citizen (age >= 65)");

    static final Condition<User> validName = new Condition<>(
            u -> u.getName() != null && !u.getName().isBlank(),
            "valid name");

    @Test
    void testCondition() {
        User user = new User("Old Joe", 70);

        // 用 is / has / doesNotHave 搭配出自然语言语感
        assertThat(user).is(seniorCitizen);
        assertThat(user).has(seniorCitizen);          // 对象级语义与 is 完全相同

        // 集合上: are 要求全部满足, isSubsetOf 之类可用 anySatisfy 表达存在性
        assertThat(List.of(user, new User("Ann", 80))).are(seniorCitizen);

        // 失败消息: Expecting actual: User[name=Old Joe, age=17] to be: senior citizen (age >= 65)
        // ——describeWith 字符串直接成为可读的期望描述
    }
}
```

## 2.2 组合子：allOf / anyOf / not

`Condition` 是可组合的值对象，布尔逻辑用组合子拼：

```java
import static org.assertj.core.api.Assertions.allOf;
import static org.assertj.core.api.Assertions.anyOf;
import static org.assertj.core.api.Assertions.not;

@Test
void compose_conditions() {
    User user = new User("Alice", 70);

    // 且: 同时是老年用户且姓名合法
    assertThat(user).is(allOf(seniorCitizen, validName));

    // 或 / 非
    assertThat(user).is(anyOf(seniorCitizen, validName));
    assertThat(new User("Bob", 30)).is(not(seniorCitizen));

    // 集合过滤视角: filteredOn 按 Condition 筛出子集继续断言
    assertThat(users).filteredOn(seniorCitizen)
                      .extracting(User::getName)
                      .contains("Old Joe");
}
```

**Condition 的边界**要心里有数：它本质是 `Predicate<T>` + 一个描述字符串，失败消息只会告诉你"没满足这个描述"，**不会指出是哪个字段没满足**。规则一旦复杂到你想在失败消息里看细节（"age 实际是 17，期望 >= 65"），就该升级到专用 Assert 类。

# 3. 企业级方案：继承 `AbstractAssert` 构建领域专属断言

若要彻底的流式语法和多重属性连缀 `assertThat(user).isAdult().hasRole("ADMIN")`，就要构建完整的专属 Assert 类。

## 3.1 继承模板与实现细节

泛型签名是第一道坎：`AbstractAssert<UserAssert, User>` 中**第一个类型参数必须指向当前断言类自身**（递归泛型，保证链式返回 `this` 后静态类型仍是 `UserAssert`，后续能继续点出业务方法），第二个是被检测对象的类型。

```java
package com.example.assertj;

import org.assertj.core.api.AbstractAssert;

// 泛型第一个必须是当前断言类自己(为了流式返回 this), 第二个是被检测对象
public class UserAssert extends AbstractAssert<UserAssert, User> {

    public UserAssert(User actual) {
        // 第二参数用于失败时定位"这是哪个断言类", 也是 as/describedAs 上下文的载体
        super(actual, UserAssert.class);
    }

    /** 静态入口工厂: 让调用方写 assertThat(user) 而不是 new UserAssert(user) */
    public static UserAssert assertThat(User actual) {
        return new UserAssert(actual);
    }

    /** 业务断言 1: 成年人 */
    public UserAssert isAdult() {
        isNotNull();                                   // 第一件事: 恒先校验 actual 非空

        if (actual.getAge() < 18) {
            // 失败消息要同时给出期望与实际, %s 由 AssertJ 做 representable 格式化
            failWithMessage("Expected user to be adult, but age was <%s>.", actual.getAge());
        }
        return this;                                   // Fluent 精髓: 原样返回自己
    }

    /** 业务断言 2: 角色 */
    public UserAssert hasRole(String expectedRole) {
        isNotNull();
        if (!expectedRole.equals(actual.getRole())) {  // 常量在前, 顺手规避 expectedRole 为 null
            failWithMessage("Expected role to be <%s> but was <%s>.",
                    expectedRole, actual.getRole());
        }
        return this;
    }

    /** 支持"取反": AssertJ 约定用 public 的 negative 变体表达, 而不是内部 if-else 塞两个语义 */
    public UserAssert isNotAdult() {
        isNotNull();
        if (actual.getAge() >= 18) {
            failWithMessage("Expected user NOT to be adult, but age was <%s>.", actual.getAge());
        }
        return this;
    }
}
```

## 3.2 入口对接与使用

自定义入口与全局 `Assertions.assertThat` 是**同名方法**，但两者签名不同——对两个类分别做单静态导入是合法的，调用点靠**重载决议**各归各位：`User` 参数命中 `UserAssert.assertThat`（更 specific），其余类型落回 AssertJ 标准入口。

```java
package com.example.assertj;

import org.junit.jupiter.api.Test;

import static com.example.assertj.UserAssert.assertThat;  // 领域入口
import static org.assertj.core.api.Assertions.assertThat;  // 标准入口(签名不同, 共存合法)

class UserAssertUsageTest {

    @Test
    void testCustomAssert() {
        User user = new User("Alice", 25);
        user.setRole("ADMIN");

        // User 参数命中 UserAssert.assertThat: 领域语言级的断言, 读起来就是需求文档
        assertThat(user)
                .as("注册用户应已成年且为管理员")   // 继承自 AbstractAssert, 前缀照用
                .isAdult()
                .hasRole("ADMIN");

        // 非 User 参数自动走 AssertJ 标准入口
        assertThat(user.getName()).startsWith("Ali");
    }
}
```

> **注意**：如果两个入口存在**签名相同的重载**（例如自定义类照抄了 `public static <T> ObjectAssert<T> assertThat(T)`），调用点就会报 reference is ambiguous——自定义入口只声明自己领域的精确类型（`assertThat(User)`），永远不要复制泛型兜底重载，这是共存的前提。

**写法纪律**——这三条决定自定义断言是资产还是负债：

- **不要在业务断言里省略 `isNotNull()`**：actual 为 null 时直接 NPE，堆栈指向断言库内部，消息毫无可读性。
- **`failWithMessage` 必须成对给出期望与实际**：你的消息是凌晨三点唯一能读到的线索。用 `<%s>` 占位让 AssertJ 格式化（能正确处理 null 与引号），别手拼字符串。
- **一个方法只测一个概念并返回 this**：把 `isAdultAndAdmin` 做成一个方法，失败时你就失去了定位能力——链式的价值恰恰在于每环独立报错。

## 3.3 进阶：让自定义断言也能"下钻"

需要返回子对象继续链式（如 `assertThat(order).items().hasSize(3)`）时，直接 new 子断言对象即可，但要**手动搬运描述**——AssertJ 内置导航方法（cause()/extracting()）靠包私有的 `withAssertionState` 复制上下文，第三方断言类拿不到这个机制：

```java
public OrderItemAssert itemAt(int index) {
    isNotNull();
    OrderItemAssert itemAssert = new OrderItemAssert(actual.getItems().get(index));
    // as() 的描述不会自动跨界, 需要显式带过去
    String desc = getWritableAssertionInfo().descriptionText();
    return desc.isEmpty() ? itemAssert : itemAssert.describedAs(desc);
}
```

> **注意**：自定义断言的下钻对象是**裸实例**，不会被软断言代理包裹——`softly.assertThat(order).itemAt(0).hasPrice(...)` 里第二跳失败会直接抛出而不是进收集器。要参与软断言收集，用 `softly.check(() -> assertThat(order).itemAt(0).hasPrice(...))` 包一层。

# 4. 通用逃生口：不建类的情况下深挖字段

领域对象只出现一两次、不值得写 Assert 类时，AssertJ 的**反射式通用断言**是够用的逃生口：

```java
@Test
void generic_reflective_escapes() {
    Order order = service.create(cart);

    // 1. 按属性/字段名断言值(先找 getter, 再找字段, 含继承链与私有成员)
    assertThat(order).hasFieldOrPropertyWithValue("status", OrderStatus.PAID);

    // 2. 存在性: 只要求有该字段/属性
    assertThat(order).hasFieldOrProperty("totalAmount");

    // 3. 提取字段值继续断言, 支持点号链式路径
    assertThat(order).extracting("customer.address.city")
                     .isEqualTo("Shanghai");

    // 4. returns: 用函数表达"这个方法返回什么", 失败消息自带函数描述
    assertThat(order)
            .returns(OrderStatus.PAID, Order::getStatus)
            .returns(3, o -> o.getItems().size());

    // 5. usingRecursiveComparison: 整棵对象图按字段名递归比, 见第 6 篇
    // assertThat(order).usingRecursiveComparison().ignoringFields("id").isEqualTo(expectedOrder);
}
```

**专用 Assert 类 vs 通用反射断言的取舍**：

| 维度 | 专用 UserAssert | hasFieldOrPropertyWithValue / extracting |
|------|-----------------|--------------------------------------------|
| 重构安全 | 编译期（方法名+类型） | **运行期才炸**（字段改名即失败） |
| IDE 补全 | 有，业务词汇 | 无，字符串裸写 |
| 失败消息 | 自定义、精确 | 通用、可用但笼统 |
| 编写成本 | 一个类 | 一行 |
| 适合 | 高频领域对象 | 一次性验证、第三方不可改类型 |

> **设计争议**：`hasFieldOrPropertyWithValue("status", PAID)` 这类写法在 code review 里常被当作**坏味道**：它绕过了类型系统，把重构期错误推迟到运行期，而且读的人看不出 `status` 是字段还是 getter 还是 map key。它的正当场景是**验证异常对象携带的私有字段**（异常类你不便改）与**测试未暴露 getter 的内部状态**——即"目标类型不配合"时的补偿手段，而不是常规字段断言的替代品。

# 5. 工业化利器：AssertJ Assertions Generator（自动生成）

手写虽然好，但遇到上百个实体与 DTO 时没人愿意维护样板代码。官方提供 `assertj-assertions-generator-maven-plugin`：扫描 entity/model 包，按反射结构生成 `XxxAssert` 类与一个聚合入口 `Assertions`。

```xml
<plugin>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-assertions-generator-maven-plugin</artifactId>
    <version>2.2.0</version>
    <executions>
        <execution>
            <goals>
                <!-- 挂载到 generate-test-sources 阶段, 先于 test-compile -->
                <goal>generate-assertions</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!-- 为哪些包下的类型生成断言 -->
        <packages>
            <param>com.yourdomain.model</param>
        </packages>
        <!-- 生成的 Java 源文件位置, 需被 build-helper 或 IDE 识别为测试源码根 -->
        <targetDir>${project.build.directory}/generated-test-sources/assertj-assertions</targetDir>
        <!-- 生成一个聚合门面类, 一次静态导入拿到全部 assertThat -->
        <generateBddAssertions>false</generateBddAssertions>
        <generateSoftAssertions>true</generateSoftAssertions>
        <entryPointClassPackage>com.yourdomain.model.assertj</entryPointClassPackage>
        <!-- 排除不需要断言的类型, 支持正则 -->
        <excludes>
            <param>.*\.package-info.*</param>
        </excludes>
    </configuration>
</plugin>
```

执行 `mvn clean test`（或单独触发 `mvn assertj:generate-assertions`）后，在 target 下得到每个实体的 `XxxAssert`，以及自动生成的 `assertThat` 聚合入口与对应的 **SoftAssertions 变体**——领域断言与软断言同时免费获得，这是很多大型微服务与数据平台项目在用的测试基础设施。

**工程实践要点**：

> **注意**：生成器产出的是**按 getter 一一映射的机械断言**（`hasName`、`hasAge`），它解决的是"有没有 API 可用"，不解决"业务语义是什么"。真正有价值的断言（`isPaid()` 内含状态机判断）必须手写。成熟做法是**分层**：生成器覆盖 DTO 字段级对账，手写 `XxxAssert` 子类或工具类承载领域概念，两者入口分开导入避免重载歧义。

> **行业实践**：引入生成器前先问三个问题——这些实体在测试里出现的频率够高吗？字段级断言真的比 `usingRecursiveComparison()` 一行更有价值吗？生成代码进入版本库还是每次构建产出（推荐后者，并把 target 目录配为 generated test sources）？多数团队试过之后会把生成器换成"少量手写核心聚合根断言 + recursiveComparison 兜底"，维护成本立降一个量级。

# 6. 小结

自定义断言的选型阶梯：能用 `allSatisfy` + 标准断言就不新建抽象；需要**可命名、可复用、可组合**的谓词用 `Condition`（`is/has/are` + `allOf/anyOf/not`），它的短板是失败消息只报描述不报字段；需要**领域语言的流式链**才继承 `AbstractAssert<SELF, ACTUAL>`，纪律是 `isNotNull()` 开头、`failWithMessage` 给全期望与实际、单概念单方法、返回 this，子对象下钻手动搬运描述（内置的上下文桥接方法是包私有的）；一次性的字段窥探用 `hasFieldOrPropertyWithValue`/`extracting`/`returns` 这些反射逃生口，但要认清它们是运行期才校验的重构黑洞；上百实体的机械字段断言交给 `assertj-assertions-generator-maven-plugin` 生成，领域语义仍须手写。
