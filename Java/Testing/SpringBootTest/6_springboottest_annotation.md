# 1. `@SpringBootTest` 的定位

`@SpringBootTest` 是 Spring Boot 测试体系里"最重也最全"的注解：它启动一个与生产同构的 `ApplicationContext`，把自动配置、Bean 装配、配置绑定、Web 环境全部拉起来。用它之前必须先回答——**这个断言是否只有在完整容器里才成立？** 本篇拆解注解的每个属性、webEnvironment 四种模式、自动配置装载过程、配套注解与启动优化。

## 1.1 适用场景

**`@SpringBootTest` 的适用判据是"验证跨越多个组件或依赖真实装配的行为"**：验证多组件协作（业务动作穿过 Controller → Service → Repository → 事件发布的完整链路）；验证配置正确加载（自动配置条件装配、`@ConfigurationProperties` 绑定、profile 合并）；需要真实容器或 Web 环境（嵌入式 Tomcat 端口监听、真实 HTTP 栈、`@ConditionalOnXxx` 最终判定）；验证"应用能不能启动"本身（每个项目都该有的 `contextLoads()` 冒烟测试）。

反过来，**不该用它的场景**：被测对象依赖可全部手工构造（纯逻辑，用 Jupiter + Mockito）；只测一层（用切片，见第 7 篇）；只想验证某个配置类（用轻量 `AnnotationConfigApplicationContext`）。

## 1.2 与其他注解的关系

**从"容器装载范围"看，各注解构成一条从全量到零容器的谱系：**

| 注解 | 定位 | 是否启动完整上下文 | Web 环境 | 典型耗时 |
|------|------|------------------|---------|---------|
| `@SpringBootTest` | 集成测试 | 是（全量自动配置） | 由 `webEnvironment` 决定 | 秒级 |
| `@WebMvcTest` / `@WebFluxTest` | Web 层切片 | 否（裁剪后的自动配置） | MOCK | 百毫秒级 |
| `@DataJpaTest` / `@JdbcTest` | 数据层切片 | 否 | NONE | 百毫秒级 |
| `@JsonTest` / `@RestClientTest` | 更窄的功能切片 | 否 | NONE | 百毫秒级 |
| 无 Spring 注解（纯 Jupiter） | 单元测试 | 不启动 | — | 毫秒级 |

机制共性：**都携带 `@BootstrapWith` + `@ExtendWith(SpringExtension.class)`，都走 TestContext Framework**（第 4 篇）；差异只在 Bootstrapper 对自动配置的裁剪程度与 Web 环境的选择。所以"`@SpringBootTest` 换成 `@WebMvcTest` 之后 `@Autowired` 突然找不到 Bean"不是 bug，是切片本来就没装配它。

---

# 2. WebEnvironment 选项

**`webEnvironment` 决定容器的 Web 形态，它是 MCC 缓存 key 的一部分——同一组配置类在 MOCK 与 RANDOM_PORT 下是两个不同的容器。** 四种取值：

| 取值 | 是否启动真实服务器 | 容器实现 | 配套客户端 | 用途 |
|------|------------------|---------|-----------|------|
| `MOCK`（默认） | 否 | `GenericWebApplicationContext`（Mock Servlet 环境） | `MockMvc` | 控制器测试、Web 层集成 |
| `RANDOM_PORT` | 是（随机可用端口） | `ServletWebServerApplicationContext`（或 Reactive 对应物） | `TestRestTemplate` / `WebTestClient` | 端到端、真实 HTTP 栈 |
| `DEFINED_PORT` | 是（`server.port` 指定端口） | 同上 | 同上 | 极少用；外部系统回调需要固定地址时 |
| `NONE` | 否 | `AnnotationConfigApplicationContext`（无 Web 环境） | — | 非 Web 应用、纯服务层集成 |

## 2.1 `MOCK`

**默认值 `MOCK` 创建"带 Mock Servlet 环境的 Web 容器"：Spring MVC 的全部基础设施（DispatcherServlet、HandlerMapping、消息转换器、过滤器）都装配了，但没有真实 HTTP 传输层。** 请求由 `MockHttpServletRequest` 直接喂给 DispatcherServlet，不经过 socket 与 Tomcat 线程池。带来速度（无端口绑定、无网络栈）与盲区（Tomcat 层面的 URL 解码、连接超时、filter 注册顺序测不到）两个后果。配合 `@AutoConfigureMockMvc` 使用（见 5.1）。

## 2.2 `RANDOM_PORT`

**`RANDOM_PORT` 启动真实的嵌入式容器并绑定一个随机可用端口（`server.port=0` 的语义），是端到端测试的标准配置。**

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort; // Boot 3.x 位置

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpE2eTest {

    @LocalServerPort
    private int port;                       // 实际绑定的随机端口

    @Autowired
    private TestRestTemplate restTemplate;  // 已预置 baseUrl = http://localhost:{port}
}
```

随机端口是并行安全的关键：多个测试类（或多个 CI 任务）同时跑不会撞端口。真实 HTTP 意味着**请求走完整栈**：序列化/反序列化、Content-Type 协商、过滤器链、安全框架、错误页面映射都被覆盖——正是 MOCK 模式的盲区。细节见第 9 篇。

> **注意**：`RANDOM_PORT` 下测试代码与应用在同一 JVM，但应用跑在自己的 Tomcat 线程上。**"测试线程设置的 ThreadLocal 在 Controller 里读不到"**——`RequestContextHolder`、MDC、`SecurityContextHolder` 都不跨线程，这是随机端口测试最常见的困惑。

## 2.3 `DEFINED_PORT`

**使用 `server.port` 配置的固定端口启动，仅适合"外部系统必须回调已知地址"的特殊场景**（支付异步通知、Webhook 联调）。常规测试禁用的原因：并行执行必撞端口、CI 端口可能被占用。真需要固定地址时优先考虑"随机端口 + 反向隧道"。

## 2.4 `NONE`

**`NONE` 不创建 Web 环境——即使 classpath 上有 spring-web，容器也是普通 `AnnotationConfigApplicationContext`。** 适用：命令行应用、批处理任务（Spring Batch）、纯消息消费者，以及"被测 Service 与 Web 无关但需要完整自动配置"的场景。副作用：Web 相关 Bean（`MockMvc`、`DispatcherServlet`）都不存在。相比 MOCK 省掉 Web 层装配开销，是"全量但非 Web"场景更快的选择。

---

# 3. 常用属性

**`@SpringBootTest` 的属性分成三类：指定"用什么配置"（classes）、注入"什么属性"（properties/args）、决定"什么形态"（webEnvironment）。** 核心属性：`args`（启动命令行参数）、`classes`（显式配置类，覆盖自动查找）、`properties`（内联属性）、`initializerClasses`（上下文初始化器）、`webEnvironment`（默认 MOCK）。

## 3.1 classes

**`classes` 显式指定容器的配置入口，绕过"向上查找 `@SpringBootConfiguration`"的默认行为。**

```java
@SpringBootTest(classes = MinimalTestConfig.class) // 只装配这个配置类定义的 Bean
class MinimalIntegrationTest {
}
```

三类使用场景：**跨模块测试**（测试类不在主应用类的包下，自动查找失败并报 `Unable to find a @SpringBootConfiguration`）；**缩小上下文规模**（传只 `@Import` 少数配置的类，跳过全量自动配置）；**替换启动配置**（用测试专用的 `@SpringBootApplication` 等价类排除生产组件）。

> **注意**：`classes` 传的不是"追加的配置"而是"配置入口"。传了 `classes = SomeConfig.class` 后主应用类的组件扫描就不再发生——想"在主容器基础上加一个配置"应该用 `@Import(SomeConfig.class)`。这是高频错误来源。

## 3.2 properties

**`properties` 以最高优先级注入内联属性，等价于类级的 `@TestPropertySource(properties = ...)`：**

```java
@SpringBootTest(properties = {
        "app.featureX=true",                  // 开启被测特性开关
        "spring.mail.host=localhost",         // 覆盖外部依赖地址
        "logging.level.com.example=DEBUG"     // 临时提高日志级别便于排查
})
class FeatureXIntegrationTest {
}
```

优先级顺序（从高到低）：`@SpringBootTest(properties)` / `@TestPropertySource` 内联属性 → `@TestPropertySource` 的 properties 文件 → `@DynamicPropertySource` 动态属性 → `application-{profile}.yml` → `application.yml` → 默认属性。

**代价必须记住：properties 进 MCC 缓存 key（第 5 篇）**。每个测试类各写一份 properties，等于每个类一个容器。共享的测试属性应沉到 `src/test/resources/application-test.yml` 并用统一的 `@ActiveProfiles("test")`，只有"这一类独有"的开关才写 `properties`。

## 3.3 args

**`args` 模拟命令行参数注入，等价于 `SpringApplication.run(App.class, args)` 的第二个参数**：`@SpringBootTest(args = {"--app.mode=test", "--batch.file=input.csv"})`。`--key=value` 形式的参数进入 Environment（优先级低于 `properties`），非 `--` 形式的参数暴露给 `ApplicationArguments` Bean。适用：测试 `ApplicationRunner`/`CommandLineRunner` 读参数的逻辑。**`args` 同样进缓存 key**，且 runner 类 Bean 在容器启动时就执行——传错参数可能导致启动阶段抛异常。

---

# 4. 启动流程（源码视角）

## 4.1 启动链路

**从注解到容器就绪，链路是"注解 → Bootstrapper → Loader → SpringApplication → 容器"：**

```text
@SpringBootTest（元注解：@BootstrapWith(SpringBootTestContextBootstrapper)）
   │  SpringExtension 收到 Jupiter 回调，首次访问上下文时：
   ▼
SpringBootTestContextBootstrapper
   ├─ 解析 classes（未指定则向上查找 @SpringBootConfiguration）
   ├─ 合并 @ActiveProfiles / @TestPropertySource / properties / args
   ├─ 按 webEnvironment 选择 ContextLoader 与容器类型
   └─ 构建 MergedContextConfiguration（MCC）
   │  缓存未命中：
   ▼
SpringBootContextLoader.loadContext(MCC)
   └─ new SpringApplicationBuilder(sources).web(deduce)
        .initializers(...).properties(内联属性).run(args)
   ▼  SpringApplication 标准流程：
prepareEnvironment → createApplicationContext → refreshContext
（profile 合并）      （按 Web 类型选实现）      （自动配置生效、Bean 实例化）
   ▼
ApplicationContext 放入 ContextCache（key = MCC）→ 注入测试实例
```

**测试环境与生产环境的差异点只有三处**：属性源里多了测试属性（优先级最高）；`RANDOM_PORT` 之外不绑定真实端口；容器不注册 shutdown hook、由缓存管理生命周期。其余——自动配置条件评估、Bean 后置处理、`@ConfigurationProperties` 绑定、事件发布——完全一致。

## 4.2 设计意图

- **复用生产启动路径，减少环境差异**：任何"只在生产启动时才暴露"的装配问题（缺 starter、条件不满足、循环依赖）在测试里同样会炸——这是价值而不是麻烦
- **将测试配置融入 SpringApplication 生命周期**：测试属性以标准 PropertySource 形式进入 Environment，`@Value`、`@ConfigurationProperties`、`Environment.getProperty` 三种读取方式结果一致
- **Web 环境按属性延迟决策**：`webEnvironment` 只影响 Bootstrapper 对容器类型与 Loader 的选择

---

# 5. 常见组合

## 5.1 与 `@AutoConfigureMockMvc`

**`MOCK` 环境下容器里有 DispatcherServlet，但 `MockMvc` 实例需要额外自动配置才作为 Bean 提供——这就是 `@AutoConfigureMockMvc` 的职责：**

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest                    // 完整容器 + MOCK Servlet 环境
@AutoConfigureMockMvc              // 装配 MockMvc Bean，含全部过滤器与安全集成
class MockMvcIntegrationTest {

    @Autowired
    private MockMvc mockMvc;       // 走真实装配的过滤器链与异常处理器
}
```

属性可调：`addFilters = false` 跳过过滤器注册；`print` 控制打印策略。**`@WebMvcTest` 已内含 `@AutoConfigureMockMvc`**，只有 `@SpringBootTest` 才需要显式加。

## 5.2 与 `@ActiveProfiles`

`@SpringBootTest` + `@ActiveProfiles("test")` 加载 `application-test.yml` 覆盖默认配置。组合的价值在于**收敛缓存 key**：全工程容器型测试统一一个 profile，比每个类各写 `properties` 少建几十个容器（第 5 篇）。`@ActiveProfiles` 支持 `inheritProfiles = false`（不继承父类 profile）与 `resolver` 自定义。

## 5.3 与 Bean 替换、Testcontainers

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers // Testcontainers 的 Jupiter 扩展管理容器生命周期
class PaymentTest {

    @Container // static → 整个测试类共享一个数据库容器
    @ServiceConnection // Boot 3.1+：自动把连接属性注册进 Environment
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16");

    @MockitoBean // Boot 3.4+；旧版本用 @MockBean
    private PaymentGateway gateway;   // 容器里的真实 Bean 被 Mock 替换

    @MockitoSpyBean
    private AuditLogger auditLogger;  // 真实实现保留，可 verify 也可 stub
}
```

`@ServiceConnection` 替代手写 `@DynamicPropertySource`，注册的属性值稳定（同一容器实例），有利于缓存命中；数据库测试细节见第 12 篇。

> **设计争议**：在 `@SpringBootTest` 里大量使用 Bean 替换是自相矛盾的——花几秒启动完整容器，又把关键组件换成 Mock。行业实践是**分层收敛**：需要 Mock 的层用切片；`@SpringBootTest` 里的替换只用于"确实不能碰的外部世界"（第三方 API、支付网关），数据库与消息队列用 Testcontainers 起真实依赖（第 10 篇）。

---

# 6. Boot 2.x 与 3.x 差异点

**从 2.x 迁移到 3.x，`@SpringBootTest` 相关的改动集中在四处**：**JDK 基线**（17+，测试代码可用 record、文本块）；**Jakarta 包名迁移**（`javax.servlet.*` → `jakarta.servlet.*`、`javax.validation.*` → `jakarta.validation.*`）；**注解换代**（Boot 3.4 起 `@MockBean`/`@SpyBean` 废弃，改用 Spring Framework 6.2 的 `@MockitoBean`/`@MockitoSpyBean`，旧注解将在 Boot 4.0 移除；新注解支持按名字/限定符替换、无匹配 Bean 时新增）；**`@LocalServerPort` 换包**（2.x 的 `org.springframework.boot.web.server` → 3.x 的 `org.springframework.boot.test.web.server`）。

Boot 3.1 的 `@ServiceConnection`、3.4 的 Bean Override 体系是新项目应直接采用的形态——2.x 时代的 `@DynamicPropertySource` 手写样板与 `@MockBean` 都可以退休。

---

# 7. 常见问题

## 7.1 启动过慢

按"先定位再优化"的顺序：**优先改用测试切片**（单笔收益最大）；**精简 `classes` 与自动配置**（`spring.autoconfigure.exclude` 排除重量级自动配置，注意改变缓存 key）；**确认缓存是否命中**（开 `org.springframework.test.context.cache=DEBUG` 看统计，miss 多说明配置发散，收敛配置比优化单容器更有效）；**延迟初始化**（`spring.main.lazy-initialization=true`）；**别把纯逻辑测试塞进容器**（`grep -r "@SpringBootTest" src/test` 数字异常大时，多半有一批本该是纯 Jupiter 的测试）。

## 7.2 Bean 注入失败

- **`Unable to find a @SpringBootConfiguration`**：自动查找失败——测试类的包路径必须在主应用类的包之下；跨模块时用 `classes = Application.class` 显式指定
- **切片测试注入不到 Service/Repository**：切片的定义行为。用 `@MockitoBean` 顶掉或改用全量测试
- **`@Value` 字段是 null/0**：测试类没走 Spring 扩展（漏了 `@SpringBootTest`），静默失败不报错
- **额外 Bean 找不到**：用 `@Import` 加入，不要指望组件扫描捡到 `@TestConfiguration`（它被显式排除在扫描之外）
- **`NoSuchBeanDefinitionException` 但生产能启动**：检查测试 profile 的排除项、`@ConditionalOnProperty` 缺属性、`webEnvironment` 差异（NONE 下没有 Web Bean）

## 7.3 上下文启动成功但断言诡异

- **测试方法之间互相影响**：单例 Bean 状态残留（第 5 篇）；数据残留（缺事务回滚或清理，第 13 篇）
- **`@Transactional` 测试里看不到自己写的数据**：默认回滚导致数据从未提交，另一连接查询看不到——设计行为
- **异步逻辑断言失败**：测试线程不等待异步任务。用 Awaitility 轮询，别用 `Thread.sleep`（第 15 篇）

---

# 8. 小结

| 主题 | 关键结论 |
|------|---------|
| 定位 | 全量容器 + 生产同构装配；判据是"断言是否只有在完整容器里才成立" |
| webEnvironment | MOCK（默认，无真实服务器，配 MockMvc）/ RANDOM_PORT（真 HTTP，配 TestRestTemplate）/ DEFINED_PORT（几乎不用）/ NONE（非 Web 容器）；它是缓存 key 的一部分 |
| classes | 是"配置入口"而非"追加配置"；跨模块必备；追加用 `@Import` |
| properties/args | 最高优先级内联属性，进缓存 key；共享配置应沉到 application-test.yml |
| 启动链路 | `@BootstrapWith` → `SpringBootTestContextBootstrapper` → `SpringBootContextLoader` → `SpringApplication.run`，与生产路径一致 |
| 配套注解 | `@AutoConfigureMockMvc`（`@SpringBootTest` 需显式加，`@WebMvcTest` 已内含）、`@ActiveProfiles`、`@MockitoBean`、`@ServiceConnection` |
| 版本差异 | JDK 17+、jakarta 包名、Boot 3.4 起 `@MockitoBean` 取代 `@MockBean`、`@LocalServerPort` 换包 |
| 常见问题 | 启动慢先查缓存命中再切片降级；注入失败先查包位置与切片边界 |

下一篇逐个拆解测试切片：`@WebMvcTest`/`@DataJpaTest`/`@JsonTest` 等各自装载什么、排除什么，以及自定义切片的实现原理。
