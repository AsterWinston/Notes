# 1. Mockito 定位与工程接入

Mockito 是 Java 生态里最常用的**测试替身（Test Double）框架**。它不负责运行测试，也不负责断言结果，而是负责在单元测试中创建"假的依赖对象"，让被测类可以在隔离的环境里被观察和控制。JUnit 提供测试的执行骨架，AssertJ 或 JUnit Assertions 提供结果断言，Mockito 则补上中间那层"依赖隔离"。

## 1.1 为什么单元测试需要测试替身

一个典型的业务类往往依赖数据库、远程接口、消息队列、时钟或文件系统。如果每次测试都启动真实依赖，测试会变慢、变脆，而且很难构造异常分支。**测试替身**就是为了解决这三个问题：加快执行速度、隔离外部状态、精确控制返回值。

业界通常把测试替身分为五类，Mockito 主要覆盖其中的 Stub、Mock 和 Spy：

| 类型 | 作用 | 典型行为 |
|------|------|----------|
| Dummy | 只为填参数，不会被真正使用 | 传入一个空对象即可 |
| Stub | 预设返回值，让被测代码走特定分支 | `when(repo.findById(1L)).thenReturn(user)` |
| Mock | 记录调用，用于验证交互 | `verify(repo).save(user)` |
| Spy | 包装真实对象，可选择性地打桩 | 未打桩的方法执行真实逻辑 |
| Fake | 轻量真实实现，如内存仓库 | `InMemoryUserRepository` |

Mockito 里的 Mock 和 Stub 在 API 上并没有严格分家：同一个对象既可以被 `when` 打桩，也可以被 `verify` 验证。真正需要区分的是**测试关注点**：Stub 关注"给出什么输入"，Mock 关注"发生了什么交互"。

## 1.2 Mockito 在测试工具链中的位置

很多初学者会把 JUnit、Mockito、AssertJ 混为一谈，实际上它们的职责边界很清楚：

| 工具 | 主要职责 | 不应该承担 |
|------|----------|------------|
| JUnit 5 | 发现测试、生命周期、参数化、断言入口 | 创建替身、验证交互 |
| Mockito | 创建替身、打桩、验证调用 | 运行测试、写业务断言 |
| AssertJ | 提供流畅、强类型的断言 | 控制依赖行为 |
| Spring Test | 启动容器、测试切片、事务回滚 | 替代单元测试设计 |

一个最小单元测试的调用链可以画成这样：

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   JUnit 5    │ ──▶ │   被测对象    │ ──▶ │   Mock 依赖   │
│  负责调度     │     │  UserService │     │ UserRepository│
└──────────────┘     └──────┬───────┘     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   断言结果    │
                     │ AssertJ/JUnit│
                     └──────────────┘
```

**设计考量**：单元测试的核心不是"用上 Mockito"，而是"把被测单元与不可控依赖隔开"。如果一个类只有纯计算逻辑，完全不需要 Mockito。只有当依赖会访问网络、磁盘、时间、随机数或全局状态时，替身才真正产生价值。

## 1.3 依赖引入与版本选择

在 Spring Boot 项目中，`spring-boot-starter-test` 已经聚合了 JUnit 5、Mockito、AssertJ、Hamcrest 等常用测试库，通常不需要单独声明 Mockito 版本。Spring Boot 3.4 管理的 Mockito 版本是 5.x，要求 Java 11 及以上。

如果项目没有使用 Spring Boot，可以手动引入下面三个坐标：`junit-jupiter` 提供测试引擎与 API，`mockito-core` 负责创建 Mock、打桩和验证，`mockito-junit-jupiter` 则让 `@Mock` / `@InjectMocks` 在 JUnit 5 下自动生效。

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.11.3</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.14.2</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.14.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

使用 Gradle 时等价写法如下：

```groovy
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.11.3'
    testImplementation 'org.mockito:mockito-core:5.14.2'
    testImplementation 'org.mockito:mockito-junit-jupiter:5.14.2'
}
```

> **行业实践**：Mockito 5 起默认使用 **inline mock maker**，可以 Mock final 类、final 方法和静态方法。过去常见到的 `mockito-inline` 依赖在 5.x 中已不再需要，继续引入反而容易造成版本冲突。老项目如果还停留在 Mockito 3/4，需要显式换成 inline 才能获得这些能力。

## 1.4 JUnit 5 集成方式

Mockito 提供了两种 JUnit 5 集成方式。第一种是手动调用 `MockitoAnnotations.openMocks(this)`，第二种是使用 `MockitoExtension`。生产代码里几乎总是推荐后者，因为它把初始化、严格模式、参数解析和资源清理都交给了扩展。

### 1.4.1 使用 MockitoExtension

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository; // 由 Mockito 自动创建并注入

    @Test
    void should_return_user_when_repository_finds_one() {
        User user = new User(1L, "Ada");
        when(repository.findById(1L)).thenReturn(Optional.of(user));

        UserService service = new UserService(repository);
        User result = service.getUser(1L);

        assertEquals("Ada", result.getName());
    }
}
```

`@Mock` 会创建一个实现或继承目标类型的代理对象，所有方法默认返回"空值"：对象返回 `null`，基本类型返回 `0` 或 `false`，`Optional` 返回 `Optional.empty()`，集合返回空集合。这一点很重要，因为很多 `NullPointerException` 并不是 Mockito 的 bug，而是开发者忘了给关键方法打桩。

### 1.4.2 手动初始化

```java
import org.junit.jupiter.api.BeforeEach;
import org.mockito.MockitoAnnotations;

class ManualInitTest {

    private UserRepository repository;

    @BeforeEach
    void setUp() {
        // 手动打开注解，返回的 AutoCloseable 在测试结束后应关闭
        MockitoAnnotations.openMocks(this);
    }
}
```

手动方式适合无法使用 JUnit 5 扩展的极端场景，但会丢失严格桩检查和自动清理，不建议在新项目中使用。

## 1.5 第一个可运行示例

下面把完整的被测代码和测试放在一起，确保复制到 IDE 后可以直接运行。先定义领域对象与仓库接口：

```java
public record User(Long id, String name) {
}

public interface UserRepository {
    User save(User user);
    java.util.Optional<User> findById(Long id);
}
```

再定义业务服务。注意它依赖接口而不是具体实现，这正是后面能被 Mock 替换的前提：

```java
import java.util.Optional;

public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public User getUser(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("用户不存在: " + id));
    }

    public User rename(Long id, String newName) {
        User existing = getUser(id);
        User updated = new User(existing.id(), newName);
        return repository.save(updated);
    }
}
```

测试覆盖两条分支：找到用户时返回对象，找不到时抛出异常：

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @InjectMocks
    private UserService service;

    @Test
    void should_return_user_when_found() {
        when(repository.findById(1L))
                .thenReturn(Optional.of(new User(1L, "Ada")));

        assertEquals("Ada", service.getUser(1L).name());
    }

    @Test
    void should_throw_when_user_missing() {
        when(repository.findById(99L)).thenReturn(Optional.empty());

        IllegalArgumentException ex = assertThrows(
                IllegalArgumentException.class,
                () -> service.getUser(99L)
        );
        assertEquals("用户不存在: 99", ex.getMessage());
    }
}
```

这里有两个值得注意的点。第一，`@InjectMocks` 会自动把 `@Mock` 创建的仓库注入 `UserService` 的构造器参数。第二，每个测试方法只打自己需要的桩，避免触发 Mockito 的严格桩检查。

## 1.6 严格模式与常见配置排错

`MockitoExtension` 默认开启 **STRICT_STUBS**。它的规则是：测试方法里声明的桩必须被实际调用，否则测试失败并抛出 `UnnecessaryStubbingException`。这能帮助你发现"写完桩但代码根本没走到"的无效测试，但也会让从 `@BeforeEach` 里统一打桩的写法变得麻烦。

| 现象 | 常见原因 | 处理方式 |
|------|----------|----------|
| `@Mock` 字段为 `null` | 忘记加 `@ExtendWith(MockitoExtension.class)`，或缺少 `mockito-junit-jupiter` | 补扩展注解和依赖 |
| `UnnecessaryStubbingException` | 桩声明了但没被调用 | 把桩移到真正使用它的测试里 |
| `MockitoException: Cannot mock/spy final class` | 使用了 Mockito 4 及以下，或禁用了 inline | 升级 Mockito 5 / 启用 inline |
| `NoSuchMethodError` / `NoClassDefFoundError` | Mockito 与 Byte Buddy 版本不匹配 | 用 BOM 或统一管理测试依赖 |
| `InvalidUseOfMatchersException` | 参数里混用了原始值和匹配器 | 全部参数统一用匹配器 |
| 测试类跑不起来 | Maven Surefire 版本太旧，未识别 JUnit 5 | 升级到 Surefire 3.x |

如果确实需要放宽严格模式，可以显式声明，但不要把全局宽松当成默认方案：

```java
import org.mockito.junit.jupiter.MockitoSettings;
import org.mockito.quality.Strictness;

@MockitoSettings(strictness = Strictness.LENIENT)
@ExtendWith(MockitoExtension.class)
class LegacyTest {
    // 仅用于迁移期，长期应删掉 LENIENT
}
```

> **注意**：JDK 21 及以上版本在动态加载 Java agent 时会打印 `WARNING: A Java agent has been loaded dynamically`。这不影响 Mockito 5 的正常工作，但如果你在 CI 里希望消除告警，可以在测试 JVM 参数中加入 `-XX:+EnableDynamicAgentLoading`，或者使用 `-javaagent` 显式挂载 Byte Buddy agent。

# 2. 练习与自检

1. 去掉 `@ExtendWith(MockitoExtension.class)` 后，`@Mock` 字段会变成什么？为什么会出现空指针？
2. 为 `UserRepository.findById` 写一个抛出 `DataAccessResourceFailureException` 的桩，并断言 `UserService.getUser` 会把异常传播出去。
3. 运行测试时遇到 `UnnecessaryStubbingException`，写出两种定位思路和一种修复方式。

# 3. 小结

Mockito 的定位很明确：在单元测试中创建可控的依赖替身，让测试聚焦于被测对象的行为。接入时要记住三件事：使用 `mockito-junit-jupiter` 与 `MockitoExtension`，优先用构造器注入配合 `@Mock`，并理解严格模式带来的"桩必须被使用"约束。

| 关注点 | 推荐做法 |
|--------|----------|
| 依赖声明 | Spring Boot 项目直接用 `spring-boot-starter-test` |
| JUnit 集成 | `@ExtendWith(MockitoExtension.class)` |
| 依赖创建 | `@Mock`，必要时 `@Spy` |
| 依赖注入 | 优先构造器注入，少用字段反射注入 |
| 版本 | Mockito 5.x + Java 11 及以上 |
| 严格性 | 默认 STRICT_STUBS，不要随意 LENIENT |

下一篇进入 `2_stubbing.md`，从最基础的 `when(...).thenReturn(...)` 开始，系统梳理打桩的返回值、异常、连续调用和 Answer 机制。
