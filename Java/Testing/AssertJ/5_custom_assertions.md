
随着业务工程代码的激增膨胀，标准类型（String, Int, Iterable）的断言已经无法提供足够的业务语境。团队可能希望能够直接针对自己系统中的独特实体类提供专门的断言如：`assertThat(order).isPaid().hasAmountGreaterThan(100)`。

AssertJ 提供了灵活的扩展点和代码生成机制支持这种情况。

---

# 1. 轻量级进阶匹配法则： `Condition<T>` 接口

如果你只是一次性的想要增加某些规则逻辑判断而不像大动干戈去写继承类，AssertJ 的 `has(Condition)` 体系是你的不二之选。
这就类似于在 Hamcrest 里写一个 Matcher。

```java
import org.assertj.core.api.Condition;

public class CustomConditionTest {

    // 1. 定义你的业务 Condition 对象
    // 泛型为待检查的对象类型。第二个参数用于报错时高亮展示
    Condition<User> seniorCitizen = new Condition<>(
        user -> user.getAge() >= 65, 
        "senior citizen (age >= 65)"
    );

    @Test
    void testCondition() {
        User user = new User("Old Joe", 70);

        // 使用 has / is / doesNotHave 配合语义阅读
        assertThat(user).is(seniorCitizen);
        assertThat(user).has(seniorCitizen);
    }
}
```

组合 `Condition` 进行高级并联规则（如 AND / OR 等），AssertJ 允许基于已经存在 `Condition` 执行诸如 `allOf`, `anyOf`, `not`：

```java
Condition<User> validName = new Condition<>(u -> !u.getName().isBlank(), "valid name");
assertThat(user).is(allOf(seniorCitizen, validName));
```

---

# 2. 企业级：继承 `AbstractAssert` 构建领域专属断言

若需要彻底的流式语法和多重属性连缀 `assertThat(user).isAdult().hasRole("ADMIN")`，唯一的方法就是构建完整的专属 Assert 类。

## 2.1 继承模板与实现细节

要断言目标对象类 `User`，我们需要实现 `UserAssert`，注意它的类型签名继承自 `AbstractAssert<UserAssert, User>`。

```java
import org.assertj.core.api.AbstractAssert;

// 泛型第一个必须是当前断言类自己（为了能够流式返回 this），第二个是你的被检测对象
public class UserAssert extends AbstractAssert<UserAssert, User> {

    protected UserAssert(User actual) {
        // "UserAssert.class" 用来处理链式异常输出时候精准到类名关联
        super(actual, UserAssert.class);
    }

    // 提供一个静态入口
    public static UserAssert assertThat(User actual) {
        return new UserAssert(actual);
    }

    // 业务断言 1：校验是不是成年人
    public UserAssert isAdult() {
        // 第一件事永远是校验 actual 是否为空
        isNotNull();
        
        // 核心校验逻辑
        if (actual.getAge() < 18) {
            failWithMessage("Expected user to be adult, but was age %s", actual.getAge());
        }
        
        // Fluent API 的精髓：把自己原封不动 return 出去，以便链式下传
        return this;
    }

    // 业务断言 2：校验其角色
    public UserAssert hasRole(String expectedRole) {
        isNotNull();
        if (!actual.getRole().equals(expectedRole)) {
            failWithMessage("Expected role to be <%s> but was <%s>", expectedRole, actual.getRole());
        }
        return this;
    }
}
```

**使用方式**：

```java
import static com.yourdomain.test.UserAssert.assertThat;

@Test
void testCustomAssert() {
    User user = new User("Alice", 25);
    user.setRole("ADMIN");

    // 一旦写成这样，它比任何 JUnit 工具都要能体现《领域驱动架构与通用语言》的思想
    assertThat(user)
        .isAdult()
        .hasRole("ADMIN");
}
```

---

# 3. 工业化利器：AssertJ Maven Plugin (自动化生成）

手写虽然好，但是遇到 100 张表和 DTO 时，没人愿意维护这种样板代码。AssertJ 官方直接帮你想好了方案 —— 通过 `assertj-assertions-generator-maven-plugin` 工具自动扫描你的 entity 包或者 model 包，由程序读取反射后编译成对应的 CustomAssert。

```xml
<plugin>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-assertions-generator-maven-plugin</artifactId>
    <version>2.2.0</version>
    <executions>
        <execution>
            <goals>
                <!-- 挂载到 test-compile 阶段之前 -->
                <goal>generate-assertions</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!-- 你希望监控与代理哪些包 -->
        <packages>
            <param>com.yourdomain.model</param>
        </packages>
        <!-- 这是生成的 Java 源文件会被放入的地方 -->
        <targetDir>${project.build.directory}/generated-test-sources/assertj</targetDir>
        <!-- 将会为你直接生成个包含所有包下断言的全局门面 Assertions.java 极其方便 -->
        <generateAssertionsForAllClasses>true</generateAssertionsForAllClasses>
    </configuration>
</plugin>
```

只要执行 `mvn clean test` 或者是编译项目，你就可以在 target 源文件中拿到完美的专属 Assert API 框架。这也是很多大型微服务和大数据底层项目中在用的极限自动化测试基础设施。