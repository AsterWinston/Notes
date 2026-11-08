# 1. Spring Boot Test 体系

Spring Boot 把 JUnit 5 作为默认测试框架，并通过 `spring-boot-starter-test` 一次性打包了整个测试工具链。**理解这个体系的关键是分清两层：JUnit Jupiter 负责测试的发现与执行，Spring TestContext Framework 负责容器的构建与注入，两者靠 `SpringExtension` 桥接**。分不清这一层，就会在"要不要加 `@ExtendWith`""为什么 `@Autowired` 是 null"这类问题上反复踩坑。

## 1.1 依赖引入

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

starter-test 传递引入的核心依赖：

| 依赖 | 用途 |
|------|------|
| JUnit Jupiter（junit-jupiter） | 测试框架本体，5.x 全家桶 |
| Spring Test / Spring Boot Test | `@SpringBootTest`、MockMvc、TestRestTemplate |
| Mockito（mockito-core + mockito-junit-jupiter） | Mock 框架 |
| AssertJ | 流式断言 `assertThat(...)` |
| Hamcrest | 匹配器风格断言（历史遗留，新代码推荐 AssertJ） |
| JSONassert / JsonPath | JSON 结构断言 |
| Awaitility | 异步结果轮询断言 |
| XMLUnit | XML 对比 |

> **注意**：Spring Boot 2.4 起 starter-test 移除了 JUnit 4 的 vintage 引擎，纯 JUnit 5；老项目迁移时如需继续跑 JUnit 4 测试，要手动加 `junit-vintage-engine` 并 exclude `junit:junit`（见第 14 篇迁移指南）。

## 1.2 与 JUnit Jupiter 的集成原理

Spring Boot 2.1 之后，`@SpringBootTest` 元注解上带了 `@ExtendWith(SpringExtension.class)`——这是 JUnit 5 扩展模型（Extension）对 JUnit 4 `SpringRunner` 的替代。所以**写 `@SpringBootTest` 的测试类不需要再显式声明任何 `@ExtendWith`**，Spring 容器已经被接入了 Jupiter 的生命周期：

```text
┌────────────────────────────────────────────────────────────┐
│              JUnit Jupiter 执行一个 Spring 测试             │
│                                                            │
│  Launcher 发现 @SpringBootTest 测试类                       │
│    │                                                       │
│    ▼                                                       │
│  SpringExtension（实现 BeforeAllCallback、                  │
│                   PostProcessTestInstanceCallback 等）      │
│    │                                                       │
│    ├─ beforeAll       → 初始化 TestContextManager           │
│    ├─ postProcessTestInstance                               │
│    │      → 从 TestContext 拿到 ApplicationContext          │
│    │      → 对测试实例执行依赖注入（@Autowired 生效点）        │
│    ├─ beforeTestMethod → 执行 TestExecutionListener 前置逻辑 │
│    ├─ 测试方法执行                                           │
│    └─ afterTestMethod → 事务回滚、Mock 重置等                │
│                                                            │
│  ApplicationContext 来源：TestContext 框架的                 │
│  ContextLoader + MergedContextConfiguration + 缓存          │
└────────────────────────────────────────────────────────────┘
```

如果用的是**纯 Spring（非 Boot）测试**，例如直接测 Spring MVC 配置，才需要显式写 `@ExtendWith(SpringExtension.class)` 加 `@ContextConfiguration`。这也是面试题"`@SpringBootTest` 和 `@ContextConfiguration` 什么关系"的答案基础：前者是 Boot 封装，内部通过 `SpringBootContextLoader` 加载完整 Boot 容器；后者是 Spring Test 原生注解，指定普通的配置类/XML。

## 1.3 @ContextConfiguration 与 @SpringBootTest 的分工

```java
// 纯 Spring Test：手动指定配置类，不启动 Boot 自动装配
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {DataSourceConfig.class, UserDaoConfig.class})
class UserDaoTest {
    @Autowired
    private UserDao userDao;
}

// Spring Boot：自动向上查找 @SpringBootApplication 作为配置入口
@SpringBootTest
class OrderServiceTest {
    @Autowired
    private OrderService orderService;
}
```

`@SpringBootTest` 的查找逻辑：从当前测试类所在包**向上逐级查找** `@SpringBootConfiguration`（`@SpringBootApplication` 的元注解之一），找到即作为容器配置入口。因此测试类的包结构必须位于主应用类的包或其子包下，否则会报 `Unable to find a @SpringBootConfiguration`——多模块项目里这个错误非常常见，解法是显式指定 `@SpringBootTest(classes = Application.class)`。

`@SpringBootTest` 常用属性：

```java
@SpringBootTest(
    classes = Application.class,              // 显式指定配置类（跨模块时必要）
    webEnvironment = WebEnvironment.RANDOM_PORT, // NONE / MOCK / RANDOM_PORT / DEFINED_PORT
    args = "--spring.profiles.active=it",     // 传给应用启动的命令行参数
    properties = "app.feature-x.enabled=true" // 内联属性，等价于 @TestPropertySource
)
class PaymentFlowIT { }
```

`webEnvironment` 决定了 Web 层的形态：`MOCK`（默认）不启动真实服务器、配合 MockMvc 使用；`RANDOM_PORT` 启动真实容器（随机端口），配合 `TestRestTemplate` / `WebTestClient` 做端到端测试。

# 2. 测试切片（Slice Test）

## 2.1 切片测试一览

`@SpringBootTest` 加载**全量**容器，而切片测试只加载某一层需要的 Bean——自动装配被限制到相关子集，启动快、隔离好，是控制器测试和仓储测试的首选：

| 切片注解 | 加载范围 | 自动 Mock/内嵌 | 典型注入 |
|----------|---------|---------------|---------|
| `@WebMvcTest` | Controller、ControllerAdvice、Filter、WebMvc 配置、Jackson | Service/Repository 需 `@MockitoBean` | `MockMvc` |
| `@WebFluxTest` | WebFlux 控制器与相关组件 | 同上 | `WebTestClient` |
| `@DataJpaTest` | Entity、Spring Data JPA Repository | 内嵌内存数据库（可用 `@AutoConfigureTestDatabase(replace = NONE)` 关掉） | `TestEntityManager` |
| `@JdbcTest` | DataSource、JdbcTemplate | 内嵌内存数据库 | `JdbcTemplate` |
| `@DataRedisTest` | RedisTemplate、连接工厂 | 内嵌 Redis | `RedisTemplate` |
| `@JsonTest` | Jackson 序列化器/反序列化器 | 无 | `JsonTester` |
| `@RestClientTest` | RestTemplate/RestClient 相关 | `MockRestServiceServer` 拦截 HTTP | `RestTemplate` |

> **注意**：`@DataRedisTest`、`@JsonTest` 这类注解来自 `spring-boot-test-autoconfigure`，其中部分（如 `@DataRedisTest`）是 Boot 3.x 才有的（Boot 2.x 时代 Redis 测试没有官方切片，需第三方或自建 `@TestConfiguration`）。写之前先确认所用 Boot 版本的注解清单。

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(UserController.class) // 只加载 UserController 及其 Web 层协作者
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;              // MOCK 环境下自动配置

    @MockitoBean                          // Boot 3.4+；旧版本用 @MockBean
    private UserService userService;      // 切片之外的 Bean 必须 Mock，否则启动失败

    @Test
    void should_return_user_when_get_by_id() throws Exception {
        given(userService.findById(1L)).willReturn(new User(1L, "tom"));

        mockMvc.perform(get("/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("tom"));
    }
}
```

## 2.2 @MockBean/@SpyBean 与 @MockitoBean/@MockitoSpyBean

切片测试几乎总是伴随 Bean 替换：Web 切片里的 Service、JPA 切片里的外部客户端，都要以 Mock 或 Spy 的形态注入容器。历史上这件事由 Spring Boot 自己的 `@MockBean`/`@SpyBean`（`org.springframework.boot.test.mock.mockito` 包）完成，**Boot 3.4 起该职责移交给了 Spring Framework 6.2**，新注解是：

| 旧（Boot ≤ 3.3，已标记废弃） | 新（Boot 3.4+ / Spring Framework 6.2+） | 语义 |
|------------------------------|------------------------------------------|------|
| `@MockBean` | `@MockitoBean`（`org.springframework.test.context.bean.override.mockito`） | 用 Mockito Mock 替换容器中的 Bean |
| `@SpyBean` | `@MockitoSpyBean` | 用 Spy 包装真实 Bean，可打桩也可放行 |

两者的行为差异不只是改包名：`@MockitoBean` 支持**按名字或按限定符替换**（`@MockitoBean(name = "primaryDataSource")`）、可以在字段上替换测试类自身依赖，且当容器里没有匹配 Bean 时会**新增**一个 Mock Bean；旧的 `@MockBean` 语义相同但实现绑定在 Boot 的测试自动配置里。

```java
@SpringBootTest
class OrderServiceTest {

    @MockitoBean
    private PaymentGateway paymentGateway; // 容器中的真实 Bean 被替换为 Mock

    @MockitoSpyBean
    private AuditLogger auditLogger;       // 真实实现仍在，可 verify 也可 stub

    @Autowired
    private OrderService orderService;
}
```

> **设计争议**：`@MockBean`/`@MockitoBean` 是"集成测试里的单元测试化"手段——它让切片测试可行，但也让测试离真实装配越来越远。凡是 Mock 掉的边界，就意味着该边界的真实行为（超时、序列化、事务传播）没有被任何测试覆盖。行业实践是：**切片内 Mock、跨切片用 Testcontainers 真实依赖、发布前跑一遍全量 `@SpringBootTest`**，三层配合而不是全靠 Mock。

# 3. Context 缓存与性能

## 3.1 缓存机制

Spring 容器启动一次动辄数秒（组件扫描、自动装配、连接池初始化），如果每个测试类都重启容器，套件耗时会不可接受。Spring TestContext 框架的解法是**按"上下文配置指纹"缓存 ApplicationContext**：两个测试类如果 `@ContextConfiguration`、`@ActiveProfiles`、`@TestPropertySource`、`webEnvironment` 等影响容器内容的配置完全一致，就复用同一个容器实例，只重新注入测试实例字段。

```text
TestContextManager
    │
    ▼
ContextCache（默认上限 32 个上下文，LRU 淘汰）
    ├── key = MergedContextConfiguration（配置类 + profiles + properties 的指纹）
    │     ├── ctx-A ← @SpringBootTest 无属性的测试类们（共享）
    │     ├── ctx-B ← @SpringBootTest(properties="x=1")（属性不同 → 新上下文）
    │     └── ctx-C ← @WebMvcTest(UserController.class)
    ▼
缓存命中 → 直接注入；未命中 → 启动新容器并放入缓存
```

由此得到一条重要的工程纪律：**让尽可能多的测试类共享同一套上下文配置**。每多一个不同的 `properties`、每多一个 `@ActiveProfiles` 组合、每换一组 `@MockitoBean` 字段，都可能导致一个新容器被创建。一个 500 个测试类的工程，上下文配置收敛到 3-5 种和发散到 50 种，CI 总耗时可以差一个数量级。

## 3.2 @DirtiesContext

如果测试**修改了容器状态**（往单例 Bean 里塞了数据、改了配置中心的内存值），缓存的容器就"脏"了，后续复用它的测试会被污染。`@DirtiesContext` 用于显式声明"我弄脏了上下文，用完请销毁重建"：

```java
@SpringBootTest
@DirtiesContext(classMode = ClassMode.AFTER_CLASS) // 类执行完后销毁上下文
class CacheWarmerTest {
    // 该测试直接操作了单例 CacheRegistry 的内部状态，无法用还原逻辑清理
}
```

`classMode` 可选 `BEFORE_CLASS` / `AFTER_CLASS` / `BEFORE_EACH_TEST_METHOD` / `AFTER_EACH_TEST_METHOD`。

> **行业实践**：`@DirtiesContext` 应该是**最后手段而非常规操作**——它的代价是缓存失效与容器重建。更好的做法是让测试不弄脏容器：用 `@MockitoBean` 替换有状态 Bean、在 `@AfterEach` 里还原数据、把可变状态设计到容器外。老代码里成片的 `@DirtiesContext` 往往就是测试套件越来越慢的元凶。

## 3.3 性能优化建议

- **能用切片就不用全量**：`@WebMvcTest` 只装配 Web 层，启动时间通常是 `@SpringBootTest` 的几分之一。
- **纯逻辑测试不要进容器**：没有 `@Autowired` 需求的测试就是普通 Jupiter 测试，加 `@SpringBootTest` 纯属浪费（还占了上下文缓存名额）。
- **收敛上下文配置种类**：把公共测试属性放进统一的 `application-test.yml` 或一个共享的 `@TestPropertySource` 基类，而不是每个类各写一份 `properties`。
- **延迟初始化**：`spring.main.lazy-initialization=true`（测试 profile 下开启）让未被用到的 Bean 不初始化，代价是首次访问变慢、部分装配错误延迟暴露。
- **并行执行**：JUnit 5 并行 + Spring 上下文缓存是兼容的，但共享可变状态的测试类要用 `@Isolated` 或资源锁隔离（见第 13 篇）。

# 4. 测试配置与 Profile

## 4.1 @ActiveProfiles 与测试专用配置

`@ActiveProfiles("test")` 激活测试 profile，使 `application-test.yml` 覆盖默认配置。工程上的常见组织方式：`src/test/resources/application-test.yml` 存放测试专用配置（内存库地址、关闭邮件发送、缩短超时），真实环境配置绝不进入 test resources。

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest {
    // 容器加载 application.yml + application-test.yml（后者覆盖前者）
}
```

Spring Boot 2.4+ 的多文档配置（`spring.config.activate.on-profile`）让一个 `application.yml` 内即可分 profile 段落，但**独立文件仍是可读性更好的选择**。

## 4.2 @TestPropertySource 与内联属性

比 profile 更细粒度的是 `@TestPropertySource`——为单个测试类覆盖属性，优先级高于任何配置文件：

```java
@SpringBootTest
@TestPropertySource(properties = {
        "order.discount.enabled=true",     // 只为这个测试类开启折扣功能
        "order.max-items=5"
})
class DiscountOrderTest { }
```

`@SpringBootTest(properties = ...)` 是它的简写等价物。属性进入 Environment 后照常可用 `@Value`、`@ConfigurationProperties` 读取。反向需求——**把测试时才能确定的值（如 Testcontainers 的随机端口）注入 Environment**——用 `@DynamicPropertySource`（见第 19 篇）。

## 4.3 测试实例内读取配置

```java
@SpringBootTest
@ActiveProfiles("test")
class BillingPropertiesTest {

    @Autowired
    private BillingProperties billingProperties; // @ConfigurationProperties Bean 照常注入

    @Value("${billing.retry.max-attempts}")
    private int maxAttempts;                     // 单值注入

    @Autowired
    private Environment env;                     // 兜底：编程式读取

    @Test
    void should_load_test_profile_values() {
        assertThat(billingProperties.getRetry().getMaxAttempts()).isEqualTo(3);
        assertThat(env.getActiveProfiles()).contains("test");
    }
}
```

> **注意**：`@Value` 注入测试字段要求测试实例经过 Spring 后处理（即容器型测试）。普通 Jupiter 测试里 `@Value` 不会生效，也不会报错——字段静默为 null/0，这类"半容器半纯单测"的混写是隐蔽 bug 源。

# 5. 常见陷阱

## 5.1 过度使用 @SpringBootTest

最常见的反模式：**给纯逻辑类写测试也挂 `@SpringBootTest`**。一个计算器工具类的测试如果启动了完整容器，单测耗时会从 1ms 变成 5000ms+，几百个这样的测试足以让 CI 从 2 分钟膨胀到 20 分钟。判断标准很简单——**测试类里有没有 `@Autowired`？没有就不该进容器**。有，也要问一句：能不能用切片替代全量？能不能把依赖 Mock 掉退化为纯 Jupiter + Mockito（见第 18 篇）？

## 5.2 MockBean 滥用

`@MockitoBean` 每次的字段组合都会参与上下文指纹计算，**不同 Mock 组合的测试类无法共享上下文**，滥用会同时带来两个问题：容器数量爆炸（性能）与真实装配未验证（正确性）。治理手段：把 Mock 集合收敛为固定的几种组合；对"必须真实"的依赖（数据库、消息队列）改用 Testcontainers 而不是 Mock。

## 5.3 事务与数据残留

`@DataJpaTest`、`@Transactional` 标注的测试默认**在每个测试方法后回滚**，这带来一个经典困惑：测试里 `save` 的数据在数据库断言（如用 JdbcTemplate 另开连接查询）时看不到——因为数据从未提交。需要验证真实提交行为时，用 `@Commit`/`@Rollback(false)` 并自行清理数据，或者干脆不加 `@Transactional`。

## 5.4 测试类包位置错误

如 1.3 所述，`@SpringBootTest` 依赖"向上查找 `@SpringBootApplication`"。测试类放在主类包之外（如 `com.example.util` 而主类在 `com.example.app`）会直接启动失败。多模块项目中每个可测模块要么有自己的 `@SpringBootConfiguration` 测试配置类，要么显式 `classes = ...`。

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 集成原理 | `@SpringBootTest` 元注解自带 `@ExtendWith(SpringExtension.class)`，注入发生在 `postProcessTestInstance` |
| 切片测试 | `@WebMvcTest`/`@DataJpaTest` 等只装配单层 Bean，切片外依赖用 `@MockitoBean` |
| Bean 替换 | Boot 3.4+ 用 Spring Framework 的 `@MockitoBean`/`@MockitoSpyBean` 替代废弃的 `@MockBean`/`@SpyBean` |
| 上下文缓存 | 按配置指纹缓存（默认上限 32），收敛配置种类是最大的性能杠杆 |
| @DirtiesContext | 销毁重建上下文的最后手段，能不用就不用 |
| 配置注入 | `@ActiveProfiles` → 测试 yml；`@TestPropertySource`/`properties` → 类级覆盖；`@DynamicPropertySource` → 运行期动态值 |
| 陷阱 | 无 `@Autowired` 不进容器；Mock 组合爆炸；测试事务默认回滚 |

下一篇进入 Mockito 与 MockMvc 的协同，解决"切片里 Mock 什么、怎么 Mock"的具体手艺。
