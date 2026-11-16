# 1. Extension 模型概览

JUnit 4 的扩展点只有两种：`Runner`（接管整个类的执行，一个类只能有一个）和 `Rule`（包裹单个测试的执行）。两者互不相通，也无法组合——想用两个 Runner 只能靠 `@Delegate` 之类的 hack。JUnit 5 把它们统一成 **Extension SPI**：一组细粒度的回调接口，一个测试类可以注册任意多个 Extension，平台按声明顺序依次回调。**"可组合、可复用"是 Extension 模型相对 Runner/Rule 的核心改进**。

## 1.1 主要扩展接口

| 接口 | 回调时机 | 典型用途 |
|------|---------|---------|
| `BeforeAllCallback` / `AfterAllCallback` | 所有测试前/后 | 启动共享容器、初始化连接池 |
| `BeforeEachCallback` / `AfterEachCallback` | 每个测试前/后 | 准备数据、计时、清理 |
| `BeforeTestExecutionCallback` / `AfterTestExecutionCallback` | 紧贴测试方法体前/后（在 `@BeforeEach` 之后） | 精确测量方法体耗时 |
| `ParameterResolver` | 解析测试方法/构造器参数 | 注入测试数据、注入临时目录 |
| `TestExecutionExceptionHandler` | 测试方法抛异常时 | 统一处理特定异常 |
| `TestInstancePostProcessor` / `TestInstancePreDestroyCallback` | 测试实例创建后/销毁前 | 给 `@Autowired` 风格字段赋值（Spring 就靠它） |
| `ExecutionCondition` | 决定是否执行某容器/测试 | 条件跳过（`@Disabled`、`@EnabledOnOs` 的底层） |
| `InvocationInterceptor` | 环绕拦截所有调用（前置/后置/动态测试） | AOP 式增强、线程上下文切换 |
| `LifecycleMethodExecutionExceptionHandler` | `@BeforeAll` 等生命周期方法抛异常时 | 生命周期异常兜底 |

## 1.2 回调嵌套顺序

理解 Extension 的时机，最好的方式是看一次完整调用的嵌套结构：

```text
BeforeAllCallback
  └─ @BeforeAll
       └─ BeforeEachCallback
            └─ TestInstancePostProcessor
                 └─ @BeforeEach
                      └─ BeforeTestExecutionCallback
                           └─ 测试方法体 ←─ TestExecutionExceptionHandler（异常时）
                      └─ AfterTestExecutionCallback
                 └─ @AfterEach
            └─ AfterEachCallback
       └─ @AfterAll
AfterAllCallback
```

**`BeforeTestExecutionCallback` 比 `BeforeEachCallback` 更贴近方法体**，做耗时统计时应该用前者，否则会把 `@BeforeEach` 的准备时间也算进去。

# 2. 扩展注册方式

## 2.1 @ExtendWith 声明式注册

最常用的方式，可以标注在类或方法上，支持多个：

```java
import org.junit.jupiter.api.extension.ExtendWith;

@ExtendWith(TimingExtension.class)          // 类级：所有测试生效
@ExtendWith({MockDbExtension.class, RandomParametersExtension.class})  // 也可以合并成一个注解
class UserServiceTest {

    @ExtendWith(SlowQueryLogExtension.class)  // 方法级：仅此方法生效
    @Test
    void find_user() { ... }
}
```

`@ExtendWith` 注册的 Extension **由平台反射实例化，必须有无参构造器**，实例默认每个测试类一个。

## 2.2 @RegisterExtension 编程式注册

Extension 需要构造参数或字段配置时用 `@RegisterExtension`，注册在 static 字段上等价于类级，实例字段上等价于方法级：

```java
class DbTest {

    @RegisterExtension
    static PostgresExtension pg = new PostgresExtension("postgres:16")
        .withInitScript("schema.sql");       // 可链式配置

    @Test
    void query() {
        DataSource ds = pg.getDataSource();  // 字段本身也能当资源句柄用
    }
}
```

`@RegisterExtension` 是 Testcontainers 的 `@Container`、各种内嵌服务器扩展的标准注册姿势。

## 2.3 自动注册（ServiceLoader）

把 Extension 全类名写进 `META-INF/services/org.junit.jupiter.api.extension.Extension`，再开启开关，**整个项目的所有测试无需任何注解即可生效**：

```properties
# src/test/resources/junit-platform.properties
junit.jupiter.extensions.autodetection.enabled=true
```

> **行业实践**：自动注册适合放"全项目统一"的横切扩展（计时、日志、安全上下文），但它是隐式魔法，新人读测试类看不到任何注册痕迹。多数团队宁愿在每个基类上显式 `@ExtendWith`，只在基础测试框架 jar 里用 ServiceLoader 自动注册。

## 2.4 继承

父类上的 `@ExtendWith` 会被子类继承；实现 Extension 接口的抽象基类也直接生效。Spring 的 `@SpringBootTest` 元注解里就带 `@ExtendWith(SpringExtension.class)`，所以业务测试类什么都不用写。

# 3. ExtensionContext 与 Store

## 3.1 ExtensionContext 提供什么

**每个回调方法都会收到一个 `ExtensionContext`，它是扩展与被测代码之间唯一的桥梁**，能拿到：当前测试类/方法/容器信息、`TestInfo`、唯一 ID、显示名、tag 集合、配置参数（`getConfigurationParameter`），以及最重要的 **Store**。

## 3.2 Store：跨回调共享数据

同一个 Extension 的多个回调（如 `beforeEach` 和 `afterEach`）之间要传递数据，不能用实例字段（实例可能被多测试类共享、并发下会串数据），要用 Store：

```java
public void beforeEach(ExtensionContext context) {
    context.getStore(ExtensionContext.Namespace.GLOBAL)
        .put("start", System.nanoTime());
}

public void afterEach(ExtensionContext context) {
    long start = context.getStore(ExtensionContext.Namespace.GLOBAL)
        .get("start", long.class);
}
```

Store 是**按 context 分层**的：测试方法级的 context 是类的子 context，查找时向父级回溯，所以 `@BeforeAll` 里存的数据测试方法回调也能读到。`Namespace` 用于隔离不同 Extension 的 key，规范做法是 `Namespace.create(MyExtension.class)` 而不是图省事全用 `GLOBAL`。Store 还支持 `getOrComputeIfAbsent` 和 `CloseableResource`——把实现了 `close()` 的资源放进 Store，**context 关闭时自动释放**，这是扩展里管理容器/连接生命周期的标准手段。

# 4. 实战：自定义扩展

## 4.1 TimingExtension：最小组合示例

组合两个回调接口实现计时：

```java
import org.junit.jupiter.api.extension.*;

public class TimingExtension
        implements BeforeTestExecutionCallback, AfterTestExecutionCallback {

    private static final ExtensionContext.Namespace NS =
        ExtensionContext.Namespace.create(TimingExtension.class);

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        context.getStore(NS).put("start", System.nanoTime());
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        long start = context.getStore(NS).get("start", long.class);
        long costMs = (System.nanoTime() - start) / 1_000_000;
        System.out.printf("%s cost=%dms%n", context.getDisplayName(), costMs);
    }
}
```

## 4.2 ParameterResolver：注入测试数据

`ParameterResolver` 让测试方法直接"要"一个参数，扩展负责给——这是**依赖注入式测试**的基础，Spring 的 `@Autowired` 注入测试字段、Mockito 的 `@Mock` 参数注入本质都是它：

```java
import org.junit.jupiter.api.extension.*;
import org.junit.jupiter.api.Test;
import java.lang.reflect.Parameter;

public class RandomUserResolver implements ParameterResolver {

    @Override
    public boolean supportsParameter(ParameterContext pc, ExtensionContext ec) {
        // 只解析带 @RandomUser 注解的 User 类型参数
        return pc.getParameter().getType() == User.class
            && pc.isAnnotated(RandomUser.class);
    }

    @Override
    public Object resolveParameter(ParameterContext pc, ExtensionContext ec) {
        // 每个测试方法拿到独立的新对象，避免用例间污染
        return User.random();
    }
}

// 使用：参数由扩展注入，测试签名即文档
@ExtendWith(RandomUserResolver.class)
class OrderServiceTest {

    @Test
    void create_order(@RandomUser User user) {
        Order order = service.create(user);
        assertEquals(user.getId(), order.getUserId());
    }
}
```

`supportsParameter` 返回 true 后平台才会调 `resolveParameter`；多个扩展都声明支持同一参数时按注册顺序取第一个。**解析逻辑要保证幂等且无副作用**，因为 `supportsParameter` 可能被平台调用多次。

## 4.3 ExecutionCondition：条件执行

返回 `ConditionEvaluationResult` 决定容器/测试是否执行：

```java
public class DockerCondition implements ExecutionCondition {
    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext context) {
        return isDockerAvailable()
            ? ConditionEvaluationResult.enabled("docker 可用")
            : ConditionEvaluationResult.disabled("docker 不可用，跳过");
    }
}
```

被禁用的测试在报告里显示为 skipped 并附带原因字符串，`@Disabled`、`@EnabledOnOs`、`assumeTrue` 的底层机制都与之同源。

# 5. 与 JUnit 4 Runner/Rule 对比

| 维度 | JUnit 4 Runner | JUnit 4 Rule | JUnit 5 Extension |
|------|---------------|--------------|-------------------|
| 作用范围 | 整个测试类 | 单个方法（`@ClassRule` 为类级） | 类级/方法级/参数级/实例级任选 |
| 数量限制 | **每类只能一个** | 可多个但类型受限 | 任意多个，自由组合 |
| 扩展点粒度 | 接管整个执行流，全有或全无 | 只有"环绕"一个点 | 十余个细粒度回调点 |
| 参数注入 | Parameterized Runner 专属 | 不支持 | `ParameterResolver` 通用能力 |
| 共享数据 | 无标准机制 | 字段传递 | `ExtensionContext.Store` 分层隔离 |
| 注册方式 | `@RunWith` | `@Rule` 字段 | `@ExtendWith`/`@RegisterExtension`/ServiceLoader |

Runner 常见的替代路径：`SpringRunner` → `SpringExtension`（已内置于 `@SpringBootTest`）、`MockitoJUnitRunner` → `MockitoExtension`、`Parameterized` → `@ParameterizedTest`、自定义 Runner → 拆解为一组回调接口的组合实现。Rule 的 `TestWatcher`、`ExpectedException`、`TemporaryFolder` 分别对应 `TestWatcher` 扩展、`assertThrows`、`@TempDir`。

# 6. 扩展顺序与冲突

多个 Extension 的回调顺序：**`before*` 类回调按注册顺序执行，`after*` 类回调按注册逆序执行**（类似洋葱模型）。注册顺序由 `@ExtendWith` 声明顺序、字段顺序（`@RegisterExtension`）、继承层级（父类先于子类）共同决定。需要精确控制时给 Extension 类加 `@Order`：

```java
@Order(1)   // 数值小的先执行 before 回调、后执行 after 回调
public class SecurityContextExtension implements BeforeEachCallback { ... }
```

两个 `ParameterResolver` 都声明支持同一参数、或两个 `TestExecutionExceptionHandler` 都处理了异常时，**第一个胜出，后面的被忽略**——异常处理扩展尤其要注意注册顺序，兜底型的必须放最后。

> **设计争议**：Extension 数量多了以后执行顺序难以推理，洋葱模型加上继承和自动注册，实际顺序可能和直觉不符。经验法则是：单个测试类上的 Extension 不超过 3-4 个，公共能力收敛到一个组合扩展或测试基类里，出问题时用 `-Djunit.platform.execution.listeners.deactivate` 排查或打印 `context.getUniqueId()` 确认层级。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 模型定位 | Extension 统一替代 Runner/Rule，细粒度回调、可任意组合 |
| 核心接口 | Before/AfterEachCallback、ParameterResolver、ExecutionCondition、InvocationInterceptor、TestInstancePostProcessor |
| 注册方式 | `@ExtendWith`（声明式）、`@RegisterExtension`（编程式）、ServiceLoader 自动注册、继承 |
| 上下文 | `ExtensionContext` 按类/方法分层，Store 是跨回调共享数据的唯一正道 |
| 参数注入 | `ParameterResolver.supportsParameter` + `resolveParameter`，注入对象要每次新建 |
| 顺序 | before 按注册序、after 逆序；`@Order` 精调；同能力扩展第一个胜出 |
| 迁移对照 | SpringRunner→SpringExtension，MockitoJUnitRunner→MockitoExtension，TemporaryFolder→`@TempDir` |
