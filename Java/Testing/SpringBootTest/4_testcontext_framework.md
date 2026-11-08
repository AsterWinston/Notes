# 1. TestContext Framework 概览

TestContext Framework 是 `spring-test` 模块里支撑所有 Spring 容器型测试的基础设施——`@SpringBootTest`、`@WebMvcTest`、`@ContextConfiguration` 全都是它的使用者。理解它，才能回答一串高频问题：测试类上的注解是怎么变成容器的？`@Autowired` 是在哪一刻被填充的？为什么事务会自动回滚？为什么两个测试类能共享同一个容器？本篇按"角色 → 启动流程 → 监听器 → 注入 → 缓存 key"的顺序把这套机制拆开。缓存的性能细节（收益量化、`@DirtiesContext`、调优）在第 5 篇展开。

## 1.1 核心定位

**TestContext Framework 负责把"测试生命周期"与"ApplicationContext 生命周期"连接起来，让测试框架（JUnit/TestNG）与容器框架（Spring）互相解耦。** 它解决三个问题：

- 测试类如何加载 Spring 容器——解析测试类上的注解元数据，选择加载器，构建并缓存 `ApplicationContext`
- 测试方法执行前后如何做依赖注入、事务管理、回滚、SQL 脚本执行——通过 `TestExecutionListener` 监听器链插件化实现
- ApplicationContext 如何缓存以减少启动成本——以"合并后的上下文配置"为 key 的 LRU 缓存

这套设计的历史动机值得记一笔：**Spring 2.5 时代测试支持是一堆抽象基类（`AbstractJUnit4SpringContextTests` 等），继承锁死了测试类的父类名额**；Spring 2.5 引入 TestContext Framework 后改为注解 + 监听器组合，JUnit 4/5、TestNG 都通过薄适配层（`SpringExtension`、`SpringJUnit4ClassRunner`）接入同一套核心——这就是今天 `SpringExtension` 只是"转发器"的原因。

## 1.2 关键角色

**四个核心组件构成一条清晰的职责链：Manager 驱动流程、TestContext 承载状态、Bootstrapper 决定"怎么建"、Loader 真正去建。**

| 组件 | 作用 | 生命周期 |
|------|------|---------|
| `TestContextManager` | 门面与驱动者：持有监听器链，把测试生命周期事件分发下去 | 每个测试类一个 |
| `TestContext` | 测试执行时上下文信息载体：连接"测试类元数据"与"当前 ApplicationContext" | 随 Manager |
| `ContextBootstrapper` | 解析注解、构建 `MergedContextConfiguration`、选择 ContextLoader | 启动期一次性 |
| `ContextLoader` | 负责创建 ApplicationContext | 缓存未命中时 |
| `TestExecutionListener` | 插件式生命周期扩展：注入、事务、脚本、脏化检测 | 全局注册，按序回调 |

---

# 2. 启动与加载流程

## 2.1 ContextBootstrapper

**`ContextBootstrapper` 是启动流程的"决策者"：它读取测试类（含父类、元注解）上的全部上下文注解，合并成一份 `MergedContextConfiguration`（MCC），再据此挑选合适的 `ContextLoader`。** 默认实现是 `DefaultContextBootstrapper`；测试类上有 `@BootstrapWith` 时用它指定的实现——**`@SpringBootTest` 的元注解链上正是 `@BootstrapWith(SpringBootTestContextBootstrapper.class)`**，这是 Boot 测试与纯 Spring 测试走上不同加载路径的分岔口。

`SpringBootTestContextBootstrapper` 相对默认实现的增量行为：

- 没有显式 `@ContextConfiguration` 时，**向上逐级查找 `@SpringBootConfiguration`**（即 `@SpringBootApplication`）作为配置入口——找不到就报 `Unable to find a @SpringBootConfiguration`
- 根据 `webEnvironment` 属性决定容器形态：`MOCK` → Servlet 模拟环境、`RANDOM_PORT`/`DEFINED_PORT` → 真实嵌入式容器、`NONE` → 普通容器
- 把 `properties`、`args` 等 Boot 特有属性并入 MCC，参与缓存 key 计算

MCC 是理解缓存行为的钥匙：**它把散落在多个注解上的配置（`@ContextConfiguration`、`@ActiveProfiles`、`@TestPropertySource`、`@ContextHierarchy`、初始化器、Loader 类型）压平成一个不可变对象，其 `equals`/`hashCode` 就是上下文缓存的 key**（第 5 篇细讲）。

## 2.2 Spring Boot 的 ContextLoader

**`SpringBootContextLoader` 的关键价值：确保测试与生产走同一条启动路径。** 它不像纯 Spring 的 `AnnotationConfigContextLoader` 那样直接 new 一个 `GenericApplicationContext` 注册配置类，而是构建一个 `SpringApplication` 并调用其启动流程：

1. **构建 `SpringApplication`**：以 MCC 里的配置类（或查找到的 `@SpringBootConfiguration`）为 primary source
2. **应用环境与属性**：合并 `@ActiveProfiles`、`@TestPropertySource` 的内联属性（以 `InlinedTestProperties` 形式置于 Environment 高优先级位置）、`@DynamicPropertySource` 的动态属性
3. **创建 `ApplicationContext`**：按 Web 环境类型选择具体实现——`NONE` 用 `AnnotationConfigApplicationContext`，`MOCK` 用带 Mock Servlet 环境的实现，`RANDOM_PORT` 用 `AnnotationConfigServletWebServerApplicationContext`（Servlet 栈）或 Reactive 对应物
4. **执行自动配置**：`@EnableAutoConfiguration` 照常生效，条件评估、starter 装配与生产启动完全一致；测试切片注解则通过 `@ImportAutoConfiguration` 把候选自动配置裁剪成子集

```text
测试类注解 ──解析──▶ ContextBootstrapper
                        │ 构建 MergedContextConfiguration（MCC）
                        │ 选择 ContextLoader
                        ▼
                 ContextCache 查询 MCC
                   │            │
              命中 ▼            ▼ 未命中
            复用已缓存容器   SpringBootContextLoader
                   │            │ new SpringApplication(...)
                   │            │ 应用 profiles / properties / args
                   │            │ run() → 自动配置 + Bean 实例化
                   │            ▼
                   │      放入缓存（默认上限 32，LRU）
                   └────┬───────┘
                        ▼
              TestContext.getApplicationContext()
                        │
                        ▼
       DependencyInjectionTestExecutionListener → 注入测试实例
```

> **设计考量**：测试复用 `SpringApplication` 启动路径是 Boot Test 最重要的架构决策——它让"自动配置错误"（缺依赖、条件不满足、属性名拼错）在测试阶段就能暴露，而不是只在生产启动时爆炸。代价是任何全量 Boot 测试都背着完整自动配置评估的开销，这正是切片测试存在的理由。

---

# 3. TestExecutionListener 机制

## 3.1 默认监听器（常见）

**监听器链是 TestContext Framework 的插件系统：每个横切关注点（注入、事务、脚本、Mock 重置）都是一个监听器，按注册顺序依次收到生命周期回调。** 默认注册的监听器（Spring Framework 6.2，按 `@Order` 排序）：

| 监听器 | 作用 |
|--------|------|
| `ServletTestExecutionListener` | 为 `MOCK` Web 环境准备 `MockServletContext` 与 `RequestContextHolder` |
| `DirtiesContextBeforeModesTestExecutionListener` | 处理 BEFORE 系列的 `@DirtiesContext` |
| `ApplicationEventsTestExecutionListener` | 支撑 `@RecordApplicationEvents` 的事件捕获 |
| `DependencyInjectionTestExecutionListener` | **为测试实例注入 `@Autowired` 等依赖**（见第 4 节） |
| `DirtiesContextTestExecutionListener` | 处理 AFTER 系列的 `@DirtiesContext`，销毁脏上下文 |
| `TransactionalTestExecutionListener` | 测试事务的开启/提交/回滚（默认回滚） |
| `SqlScriptsTestExecutionListener` | 执行 `@Sql` 注解指定的脚本 |

Boot 侧还会追加自己的监听器（如处理 `@MockitoBean` 重置的机制在 Framework 6.2 中由 Bean Override 基础设施承担、`ResetMocksTestExecutionListener` 处理遗留 `@MockBean`、`MockitoTestExecutionListener` 等），通过 `spring.factories`（现为 `SpringFactoriesLoader` 机制文件）自动发现。**顺序有语义**：注入监听器必须在事务监听器之前完成前置工作，脏化检测必须在注入之前决定是否重建容器——这就是监听器都带 `@Order` 的原因。

## 3.2 自定义监听器

**`@TestExecutionListeners` 用于替换或追加监听器链，是"在每个测试前后统一做环境准备"的正规扩展点：**

```java
import org.springframework.test.context.TestExecutionListeners;
import org.springframework.test.context.support.AbstractTestExecutionListener;
import org.springframework.test.context.TestContext;

// 自定义监听器：为每个测试方法设置线程上下文（如租户 ID）
public class TenantContextListener extends AbstractTestExecutionListener {

    @Override
    public void beforeTestMethod(TestContext testContext) {
        TenantContext.set("test-tenant"); // 模拟多租户请求上下文
    }

    @Override
    public void afterTestMethod(TestContext testContext) {
        TenantContext.clear();            // 防止跨用例污染线程
    }
}

@TestExecutionListeners(
    value = TenantContextListener.class,
    mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS // 与默认链合并
)
class CustomListenerTest {
}
```

`mergeMode` 两个取值语义完全不同：`MERGE_WITH_DEFAULTS` 把自定义监听器**并入**默认链（按 `@Order` 排位）；默认的 `REPLACE_DEFAULTS` 则**整体替换**——忘记写 mergeMode 会导致 `@Autowired` 突然失效（注入监听器被替换掉了），这是该注解最经典的坑。全局注册（不想每个类都写注解）可以走 `SpringFactoriesLoader`：在 `META-INF/spring.factories` 里登记 `org.springframework.test.context.TestExecutionListener` 键。

使用场景：需要在测试前后设置线程上下文、审计信息、时钟偏移或外部依赖状态的横切逻辑——比在每个测试类写 `@BeforeEach`/`@AfterEach` 更收敛。

> **注意**：自定义监听器里不要持有跨测试类的可变状态。监听器实例在整条链上是共享的，一个监听器字段里的 Map 就是所有测试类的公共黑板，Flaky 之源。

---

# 4. 依赖注入与测试实例

## 4.1 测试类注入

**测试实例的依赖注入由 `DependencyInjectionTestExecutionListener` 完成：它在 `prepareTestInstance` 回调里把测试实例交给容器的 `AutowireCapableBeanFactory` 走一遍标准的注入后处理**——效果等同于把测试类当成一个（不注册进容器的）Bean：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationContext;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;        // 字段注入：由注入监听器填充

    @Autowired
    private ApplicationContext context;     // 容器本身也可以注入

    // 构造器注入同样支持（Spring Framework 5.x 起对测试类生效）
    // @Autowired 构造函数可省略注解，多构造器时需显式标注
}
```

机制细节决定了几个常见现象：

- **注入发生在实例创建后、`@BeforeEach` 前**（Jupiter 侧由 `SpringExtension` 的 `TestInstancePostProcessor` 触发 `prepareTestInstance`），所以 `@BeforeEach` 里字段已可用；而 `@BeforeAll` 在实例化之前，拿不到注入字段
- **默认 PER_METHOD 生命周期下，每个测试方法一个新实例、注入一次**——注入的是缓存容器里的同一批单例 Bean，代价只是反射赋值，很便宜
- **测试实例本身不进容器**：它是被容器"处理"的对象，不是容器管理的 Bean。所以测试类上的 `@Transactional`、`@Value` 由测试框架特殊支持，而不是走 Bean 后处理器
- `@Value("${...}")` 在测试字段上可用；但**纯 Jupiter 测试（无 `SpringExtension`）里 `@Value` 静默不生效**，字段是 null——"半容器半纯单测"混写是隐蔽 bug 源

Jupiter 还支持**测试方法参数注入**：`SpringExtension` 实现了 `ParameterResolver`，可以把 `ApplicationContext`、`Environment` 等直接声明为方法参数。

## 4.2 测试专用配置

**`@TestConfiguration` 提供"只属于测试的 Bean 定义"**，两种用法：内部静态类（只对当前测试类生效、随其 MCC 参与缓存 key）或独立顶层类（可被多个测试类 `@Import` 共享）：

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;

@SpringBootTest
@Import(UserServiceTest.TestConfig.class) // 顶层用法需显式 Import；内部类会被自动发现
class UserServiceTest {

    @TestConfiguration(proxyBeanMethods = false)
    static class TestConfig {

        @Bean
        public Clock fixedClock() {
            // 固定时钟：让所有依赖时间的断言可复现
            return Clock.fixed(Instant.parse("2024-01-01T00:00:00Z"), ZoneOffset.UTC);
        }
    }
}
```

`@TestConfiguration` 与 `@Configuration` 的差异只有一点：**它不会被主应用的组件扫描捡到**（`@SpringBootConfiguration` 的扫描显式排除它），保证测试配置不泄漏进生产容器。语义上它支持"覆盖同名 Bean"（Spring Boot 默认禁止 Bean 定义覆盖，但对测试配置放行），常用来提供 Fake 实现替换真实外部依赖。事务与回滚行为由 `TransactionalTestExecutionListener` 支撑，详见第 13 篇。

---

# 5. 源码视角理解

## 5.1 关键对象关系

**整条链上每个对象只认识自己的下一环，这个"逐级委托"结构是读懂源码的地图：**

```text
TestContextManager                    ← 门面：JUnit/TestNG 适配层只调它
  ├── List<TestExecutionListener>     ← 插件链：注入/事务/脚本/脏化
  └── TestContext（DefaultTestContext）
        ├── Class<?> testClass        ← 当前测试类
        ├── MergedContextConfiguration（MCC）
        │     ├── locations / classes          ← @ContextConfiguration
        │     ├── activeProfiles               ← @ActiveProfiles
        │     ├── propertySourceProperties     ← @TestPropertySource / properties
        │     ├── contextInitializerClasses    ← ApplicationContextInitializer
        │     ├── parent MergedContextConfiguration ← @ContextHierarchy 父级
        │     └── contextLoaderClass + 自定义属性（Boot 的 webEnvironment 等）
        ├── ContextLoader（SpringBootContextLoader）
        └── ApplicationContext（来自 ContextCache，可能为 null 直到首次访问）
```

调用时序上，`SpringExtension` 收到 Jupiter 回调后全部转发给 `TestContextManager`；Manager 先让监听器链依次处理，涉及容器访问时通过 `TestContext.getApplicationContext()` 触发"查缓存 → 未命中则 Loader 加载 → 放入缓存"的惰性流程。**容器不是测试类一加载就启动的，而是第一次真正被需要时才启动**——全部方法都被条件跳过的测试类可以完全不付容器成本。

## 5.2 设计动机

- **通过 TestContext 把"测试元数据"与"上下文实例"解耦**：元数据（MCC）是不可变的、可比较的，因此能当缓存 key；上下文实例是有状态、可脏化、可销毁的。两者分离后，"哪些测试共享容器"变成一个纯粹的元数据比较问题
- **通过监听器链提供扩展点，避免测试框架与业务框架强耦合**：事务、SQL 脚本、Mock 重置这些能力对 Spring 核心是可选的，做成监听器后按需插拔；第三方（如 Testcontainers 的 `@Testcontainers` 集成、Spring Security 的测试支持）也走同一扩展机制
- **通过 Bootstrapper 隔离"框架差异"**：Boot 的特殊性（自动配置查找、Web 环境选择）全部收在 `SpringBootTestContextBootstrapper` 里，核心框架对 Boot 一无所知——纯 Spring、Spring Boot、甚至自定义框架都能在同一套 TestContext 上生长

> **设计争议**：监听器链的"隐式默认注册"方便了开箱即用，但代价是行为不透明——测试类上看不见任何事务配置，回滚却发生了（`TransactionalTestExecutionListener` 在链上）。排查"数据为什么没提交"这类问题必须知道默认链的构成，这也是本篇把默认监听器表列全的原因。

---

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 核心角色 | `TestContextManager` 驱动、`TestContext` 承载、Bootstrapper 决策、Loader 建容器、Listener 插件化横切 |
| Boot 分岔口 | `@SpringBootTest` 元注解携带 `@BootstrapWith(SpringBootTestContextBootstrapper.class)`，触发"向上找 `@SpringBootConfiguration`"与 Web 环境选择 |
| 启动路径 | `SpringBootContextLoader` 内部走 `SpringApplication` 流程，测试与生产装配一致；容器惰性启动（首次访问才建） |
| 监听器链 | 注入 → 事务 → 脚本按 `@Order` 排序；`@TestExecutionListeners` 默认 REPLACE_DEFAULTS，追加要写 MERGE_WITH_DEFAULTS |
| 测试注入 | `DependencyInjectionTestExecutionListener` 在实例创建后注入；测试类不进容器；`@BeforeAll` 拿不到注入字段 |
| MCC | 注解元数据压平成的不可变对象，`equals`/`hashCode` 即上下文缓存 key——任何属性差异都是新容器 |
| @TestConfiguration | 不被生产扫描捡到的测试专用配置；内部类随宿主类参与缓存 key |

下一篇聚焦 MCC 作为缓存 key 的性能意义：缓存收益量化、`@DirtiesContext` 的各种模式与滥用治理。
