# 1. 定义与目标

Spring Boot Test 是理解整个 Spring 测试体系的入口。它不是一个新的测试框架，而是一套"组装方案"：把 JUnit 5、Spring Test、Mockito、AssertJ、MockMvc、TestRestTemplate 等十几个组件按约定粘合起来，让开发者只写一个 `@SpringBootTest` 就能获得接近生产形态的测试环境。本篇给出这套体系的全景图——它包含什么、各类测试如何分工、和纯 JUnit 测试的边界在哪里。后续篇章（TestContext 框架、上下文缓存、切片、MockMvc 等）都是对这张全景图某个局部的放大。

## 1.1 Spring Boot Test 是什么

**Spring Boot Test 是 Spring Boot 为测试提供的一整套能力集合，核心目标是让测试在"接近真实运行环境"的前提下依然保持可控、可读、可维护。** 它由两个 Maven 工件承载：`spring-boot-test`（核心注解与工具，如 `@SpringBootTest`、`TestRestTemplate`）和 `spring-boot-test-autoconfigure`（测试切片的自动配置，如 `@WebMvcTest`、`@DataJpaTest`）。这两个工件加上 Spring Framework 自带的 `spring-test`，再通过 `spring-boot-starter-test` 打包了第三方断言与 Mock 库，构成了完整的测试技术栈：

- **Spring Test**：提供底层机制——TestContext Framework（容器加载、缓存、注入）、MockMvc、`@Transactional` 测试事务、`@Sql` 脚本执行
- **JUnit 5**：测试的发现与执行引擎（JUnit Platform + Jupiter API）
- **Mockito、AssertJ、JSONassert、Hamcrest、Awaitility、JsonPath、XMLUnit**：Mock、断言与数据格式校验的第三方组件
- **Spring Boot 自动配置的测试延伸**：`@SpringBootTest` 复用生产启动路径，测试切片复用自动配置的条件装配机制

一个常被混淆的点是"Spring Test"与"Spring Boot Test"的关系：**Spring Test 属于 Spring Framework，不依赖 Boot 也能用**（`@ContextConfiguration` + `SpringExtension` 就能跑纯 Spring 测试）；Spring Boot Test 在其上叠加了 Boot 特有的能力——自动配置装载、切片注解、嵌入式 Web 环境、`@MockitoBean` 式的 Bean 替换。所以面试里问"@SpringBootTest 和 @ContextConfiguration 什么关系"，答案是前者内部通过 `SpringBootContextLoader` 走 `SpringApplication.run` 的完整路径，后者只加载显式指定的配置。

## 1.2 设计目标与价值

### 1.2.1 快速可靠

**测试体系的第一约束是速度——跑得慢的测试等于没有测试。** Spring Boot Test 在三个层面控制成本：TestContext 框架按"配置指纹"缓存 `ApplicationContext`，几百个测试类可能只启动三五次容器；测试切片（slice）只加载某一层需要的 Bean，`@WebMvcTest` 的启动时间通常是全量容器的几分之一；注解式配置（`@ActiveProfiles`、`@TestPropertySource`）把测试初始化压缩到声明层面，避免每个测试类手写 setup 代码。可靠性则来自确定性：内嵌数据库、随机端口、事务自动回滚，让测试之间互不干扰、可以并行。

### 1.2.2 接近真实

**"测试能过、生产挂"的根源是测试环境与生产环境的装配路径不一致，Spring Boot Test 的对策是让测试走同一条启动路径。** `@SpringBootTest` 内部调用 `SpringApplication.run`，自动配置、条件装配、profile 合并、`@ConfigurationProperties` 绑定全部照常发生；`webEnvironment = RANDOM_PORT` 进一步启动真实的嵌入式 Tomcat/Netty，用真实 HTTP 栈验证序列化、过滤器链、异常映射。这一点的价值在大型项目里尤其明显：一个 Bean 因为缺少 `@ConditionalOnProperty` 在生产没被装配，只有走完整启动路径的测试才能暴露。

### 1.2.3 可控可替换

接近真实不等于照单全收——**外部依赖（支付网关、第三方 API、消息队列）必须可替换，否则测试无法在 CI 里稳定运行**。Boot 提供的替换手段从"轻"到"重"排列：

- `@MockitoBean` / `@MockitoSpyBean`（Boot 3.4+，取代已废弃的 `@MockBean` / `@SpyBean`）：在容器内用 Mockito 替身替换真实 Bean
- `@TestConfiguration`：为测试提供专用配置，注入 Fake 实现（如内存版邮件发送器）
- `@DynamicPropertySource`：把运行期才能确定的值（Testcontainers 的随机端口）注入 Environment
- `properties` / `args` 属性：内联覆盖任意配置项

## 1.3 与 Spring Test / JUnit 的关系

**三者的分工可以概括为：JUnit 5 管"执行"，Spring Test 管"容器"，Spring Boot Test 管"装配约定"。** JUnit Platform 负责发现测试、调度引擎；Jupiter 提供 `@Test`、`@BeforeEach`、断言与 Extension 扩展点；Spring Test 通过 `SpringExtension` 挂进 Jupiter 的生命周期，在测试实例创建后完成容器加载与依赖注入；Spring Boot Test 则决定"容器长什么样"——用哪个配置类、开不开 Web 环境、装不装某个自动配置。

```text
JUnit Platform（Launcher / Engine SPI）
    │  发现并调度测试
    ▼
JUnit Jupiter（@Test、Extension 模型）
    │  SpringExtension 挂载到生命周期回调
    ▼
Spring Test（TestContext Framework）
    │  TestContextManager → ContextLoader → 容器缓存
    ▼
Spring Boot Test（@SpringBootTest、切片、MockMvc 装配）
    │  复用 SpringApplication 启动路径
    ▼
ApplicationContext（被测系统的真实容器）
```

> **注意**：正因为 `@SpringBootTest` 的元注解上已经带了 `@ExtendWith(SpringExtension.class)`（Boot 2.1 起），写了 `@SpringBootTest` 的测试类**不需要也不应该**再手动声明 `@ExtendWith`。只有在纯 Spring（非 Boot）测试里才需要显式写它。

---

# 2. 测试类型与覆盖面

## 2.1 测试金字塔在 Boot 场景的映射

**测试类型谱系是一条"覆盖范围递增、数量递减、速度递减"的连续带，而不是几个孤立的盒子。** 从纯逻辑测试到端到端测试，每一档在 Spring Boot 里都有对应的惯用手段：

| 层级 | 目标 | Spring Boot 典型手段 | 容器状态 | 单个用例典型耗时 |
|------|------|---------------------|---------|----------------|
| 单元测试 | 逻辑验证、快速反馈 | JUnit 5 + Mockito（无 Spring 注解） | 不启动 | 毫秒级 |
| 切片测试 | 局部组件验证 | `@WebMvcTest` / `@DataJpaTest` / `@JsonTest` | 部分容器 | 百毫秒级 |
| 集成测试 | 多组件协作 | `@SpringBootTest` + `@MockitoBean` | 完整容器（MOCK 环境） | 秒级 |
| 端到端测试 | 真实链路验证 | `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate` / `WebTestClient` | 完整容器 + 真实 HTTP | 秒级 |

这个谱系里最容易犯错的位置是"单元"与"集成"的边界：**判断标准不是被测类的大小，而是测试是否需要容器**。一个几百行的 Service，只要依赖都能用构造函数传进去，它的测试就是纯 Jupiter + Mockito 的单元测试；反过来，一个三行的配置类，要验证它在自动配置条件下是否正确装配，就得进容器。

## 2.2 选择测试类型的核心准则

选择测试类型时按优先级权衡四条准则：

- **速度优先**：能不启 Spring 就不启。纯逻辑挂 `@SpringBootTest` 是最常见的浪费——单测从 1ms 变成 5000ms，几百个这样的测试足以让 CI 从 2 分钟膨胀到 20 分钟
- **边界清晰**：只验证"我关心的那一层"。控制器测试不该因为 Service 的 Mock 桩没配好而失败；仓储测试不该关心 JSON 序列化
- **失败可定位**：失败信息应指向单一问题。切片测试失败基本锁定在本层，全量集成测试失败则可能是装配、配置、数据、网络中任何一环
- **复用真实配置**：测试与生产的配置分叉越小越好。测试专用的 yml 只应覆盖"必须不同"的项（数据库地址、外部服务开关），业务参数保持一致

> **行业实践**：成熟团队的分布大致是"单元测试占 70% 以上、切片测试 20%、集成/端到端 10%"（详见第 2 篇金字塔讨论）。但比例是结果不是目标——真正的纪律是**每写一个测试先问：这个断言最低可以落在哪一层？** 能落在纯 JUnit 就不进切片，能落在切片就不进全量。

## 2.3 典型组合

三种最常见、几乎每个 Boot 项目都会出现的组合：

```java
// 组合一：控制器层——Web 切片 + MockMvc + Mock 掉 Service
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;              // MOCK 环境自动装配
    @MockitoBean
    private UserService userService;      // 切片之外的 Bean 必须 Mock
}

// 组合二：仓储层——JPA 切片 + 内嵌数据库 + 事务自动回滚
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private TestEntityManager entityManager; // 切片提供的便捷入口
}

// 组合三：系统回归——完整容器 + 随机端口 + 真实 HTTP
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderFlowE2eTest {
    @Autowired
    private TestRestTemplate restTemplate;   // 自动指向随机端口
}
```

---

# 3. 版本与生态关系

## 3.1 Spring Boot 3.x 与 2.x 差异

**Boot 3.x 对测试代码的影响集中在包名迁移和注解换代两处。** 大版本差异：

| 维度 | Boot 2.x | Boot 3.x（3.2+/3.4） |
|------|----------|---------------------|
| 基线 JDK | 8/11 | 17+ |
| Servlet API | `javax.servlet.*` | `jakarta.servlet.*` |
| Spring Framework | 5.x | 6.x（3.4 对应 6.2） |
| JUnit | 5 已默认（2.4 起移除 vintage） | 5.10+ 深度集成 |
| Bean 替换注解 | `@MockBean` / `@SpyBean`（Boot 自有） | `@MockitoBean` / `@MockitoSpyBean`（Spring Framework 6.2 提供，Boot 3.4 起旧注解废弃） |
| 观察性测试 | Sleuth | Micrometer Tracing |

测试视角的具体影响点：

- **Jakarta 迁移**：所有 `javax.servlet`、`javax.validation` 的 import 要改；MockMvc 的 `MockHttpServletRequest` 等类型随之换包
- **`@MockBean` → `@MockitoBean`**：Boot 3.4 起新测试一律用 `org.springframework.test.context.bean.override.mockito.MockitoBean`，旧注解标记为 deprecated 且将在 Boot 4.0 移除。新注解还支持按名字/限定符替换、无匹配 Bean 时新增 Mock
- **监听器链调整**：Spring Test 6.x 对 `TestExecutionListener` 的默认注册顺序与 `@MockitoBean` 的重置机制做了重构（Mock 重置从 Boot 的监听器移到了 Framework 的 Bean Override 机制里）

> **注意**：从 Boot 2.x 迁移的老项目如果还依赖 JUnit 4 测试，需要手动加 `junit-vintage-engine`；Boot 2.4 起 starter-test 已不再默认携带它。

## 3.2 核心依赖

### 3.2.1 Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

`spring-boot-starter-test` 传递引入的内容（以 Boot 3.4 为例）：

| 依赖 | 用途 |
|------|------|
| junit-jupiter | 测试框架本体（API + 引擎 + Platform 启动器） |
| spring-test / spring-boot-test / spring-boot-test-autoconfigure | TestContext、MockMvc、`@SpringBootTest`、切片自动配置 |
| mockito-core + mockito-junit-jupiter | Mock 框架及其 Jupiter 扩展 |
| assertj-core | 流式断言 `assertThat(...)` |
| hamcrest | 匹配器断言（历史遗留，新代码推荐 AssertJ） |
| jsonassert / json-path | JSON 结构断言（MockMvc 的 `jsonPath` 依赖后者） |
| awaitility | 异步结果轮询断言 |
| xmlunit-core | XML 对比 |

### 3.2.2 Gradle

```gradle
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

> **设计考量**：starter-test 把断言库（AssertJ + Hamcrest）和 Mock 库（Mockito）都替团队做了选型，这是 Spring Boot"约定优于配置"在测试领域的延伸。代价是升级受制于 Boot 的依赖管理版本；需要更新版本的 Mockito 时可在自己的 `dependencyManagement` 里显式覆盖版本号。

---

# 4. 核心术语

## 4.1 ApplicationContext 与 TestContext

**ApplicationContext 是 Spring 容器的运行时模型，TestContext 则是测试框架对这个容器的"托管包装"。** TestContext 框架（详见第 4 篇）负责三件事：按测试类的注解元数据决定加载哪个容器；把加载好的容器放进缓存供配置相同的测试类复用；在每个测试实例创建后执行依赖注入。测试代码里通过 `@Autowired` 拿到的 Bean、通过 `ApplicationContext` 参数拿到的容器，都来自 TestContext 的托管结果——这意味着测试里的容器和生产 `main` 方法启动的容器"内容一致但生命周期不同"：前者跨测试类共享、默认不关闭，后者进程级唯一。

## 4.2 测试切片（Slice）

**切片是"只加载某一层所需 Bean 的部分容器"，本质是对自动配置的条件裁剪。** `@WebMvcTest` 并非启动完整容器后过滤 Bean，而是通过 `@TypeExcludeFilters` 限制组件扫描范围、通过 `@ImportAutoConfiguration` 只导入与 Web MVC 相关的自动配置类——数据库、消息队列的自动配置根本不会参与条件评估。因此切片测试既快又隔离：容器里没有 DataSource，就不可能因为数据库连不上而失败。各切片的装载范围对照见第 7 篇。

## 4.3 Test Double（替身）

**Test Double（测试替身）是所有"替代真实依赖"手段的统称**，出自 xUnit Test Patterns 的分类学：

| 类型 | 说明 | Boot 典型用法 |
|------|------|--------------|
| Dummy | 只为填参数占位，不被调用 | 传 `null` 或空对象给用不到的依赖 |
| Stub | 提供固定返回值，关注"状态" | Mockito `when(...).thenReturn(...)` |
| Mock | 预设交互期望，关注"行为" | `@MockitoBean` + `verify(...)` |
| Spy | 代理真实对象，可选择性打桩 | `@MockitoSpyBean` |
| Fake | 简化的真实实现 | `@TestConfiguration` 注入内存版实现 |

区分 Stub 与 Mock 的实践意义：Stub 验证"返回值对不对"，Mock 验证"调用发生了没有"。过度使用 Mock 的交互验证会让测试变脆——实现细节一重构，测试就红。行业倾向是**状态验证优先于交互验证**，Mock 只用在真正需要断言"副作用发生了"的场景（发事件、调外部 API）。替身的完整用法见第 10 篇。

---

# 5. 最小示例

## 5.1 业务代码

以一个最小的问候服务为例，展示从业务类到测试的完整闭环：

```java
package com.example.demo;

import org.springframework.stereotype.Service;

@Service
public class GreetingService {

    public String greet(String name) {
        return "hello, " + name; // 纯逻辑，无外部依赖
    }
}
```

## 5.2 测试代码

```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest // 加载完整容器；元注解自带 SpringExtension，无需再声明
class GreetingServiceTest {

    @Autowired
    private GreetingService greetingService; // 由 TestContext 框架注入

    @Test
    void greet_shouldReturnHello() {
        assertThat(greetingService.greet("spring"))
                .isEqualTo("hello, spring"); // AssertJ 流式断言
    }
}
```

值得说明的是：**这个例子其实是"高射炮打蚊子"**——`GreetingService` 没有任何需要容器的依赖，更合理的写法是纯 Jupiter 测试：

```java
class GreetingServiceTest {

    private final GreetingService service = new GreetingService(); // 直接 new，不进容器

    @Test
    void greet_shouldReturnHello() {
        assertThat(service.greet("spring")).isEqualTo("hello, spring");
    }
}
```

两个版本的断言完全相同，耗时却差三个数量级。这里保留 `@SpringBootTest` 版本是为了演示容器型测试的最小形态——当你需要验证"这个 Bean 确实被装配了、配置确实生效了"时才值得付出容器成本。

## 5.3 执行方式

```bash
mvn -q test                      # Maven：执行 surefire 绑定的测试
gradle test                      # Gradle：执行 test 任务
```

IDE 中直接运行测试类或方法即可。构建期的测试分层执行策略（单元测试与 `*IT` 集成测试分开跑）见第 2 篇和第 18 篇。

---

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 体系构成 | Spring Boot Test = spring-boot-test + spring-boot-test-autoconfigure，叠加在 Spring Test 与 JUnit 5 之上，由 starter-test 统一打包 |
| 三层分工 | JUnit 5 管执行，Spring Test 管容器，Boot Test 管装配约定；`SpringExtension` 是桥 |
| 设计目标 | 快速（缓存 + 切片）、真实（复用 `SpringApplication.run` 路径）、可控（Mock/替身/属性覆盖） |
| 类型谱系 | 单元 → 切片 → 集成 → 端到端，覆盖范围与成本递增；判断进不进容器的标准是"有没有 `@Autowired` 需求" |
| 版本要点 | Boot 3.x：JDK 17+、jakarta 包名、3.4 起 `@MockitoBean` 取代 `@MockBean` |
| 核心术语 | TestContext 托管容器生命周期；切片是自动配置的条件裁剪；替身五分类中状态验证优先于交互验证 |

下一篇讨论测试策略：金字塔与奖杯模型之争、各层比例怎么定、CI 上如何分层执行。
