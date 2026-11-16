# 1. 测试切片概览

测试切片（Slice Test）是 Spring Boot 对"容器成本与测试信心"矛盾的官方回答：**当断言只关于某一层时，没必要为它启动全部装配。** 本篇给出各切片的装载/排除对照、机制原理（`@TypeExcludeFilters` + `@ImportAutoConfiguration`）、切片与全量的选择准则，以及自定义切片的实现。

## 1.1 为什么需要切片

**完整启动 `@SpringBootTest` 的成本与"被测的那一层"完全不成比例**：测一个 Controller 的路由映射，却要等 JPA 扫描实体、连接池初始化、消息监听器注册——既拖慢启动，又引入无关失败模式（数据库连不上导致 Web 测试红）。切片测试**只加载某一层相关 Bean**：启动更快（几百毫秒，是全量的几分之一到十分之一）；边界更清晰（容器里没有 DataSource，就不可能死于数据库问题）；问题定位更容易（`@WebMvcTest` 失败就是 Web 层的事）。

## 1.2 切片的本质

**切片不是"启动全量容器后过滤 Bean"，而是在装配发生之前就裁剪了候选集。** 两个机制配合：**组件扫描裁剪**——切片注解携带 `@TypeExcludeFilters`，扫描阶段只放行本层的构造型注解（Web 切片只放行 `@Controller`、`@ControllerAdvice`、`@JsonComponent`、`WebMvcConfigurer` 等），`@Service`/`@Repository`/`@Component` 被拒绝注册；**自动配置裁剪**——切片注解携带 `@ImportAutoConfiguration`，从 `.imports` 清单只导入相关的自动配置，数据库、消息队列等**根本不参与条件评估**。

```text
全量 @SpringBootTest                 切片 @WebMvcTest
┌──────────────────────────┐        ┌──────────────────────────┐
│ 扫描：全部 @Component 系  │        │ 扫描：仅 Web 构造型注解    │
│ 自动配置：140+ 候选全评估 │        │ 自动配置：仅 WebMvc 相关   │
│ DataSource / JPA / MQ ✓  │        │ DataSource / JPA / MQ ✗   │
│ 启动：秒级                │        │ 启动：百毫秒级             │
└──────────────────────────┘        └──────────────────────────┘
```

**推论：切片容器"缺 Bean"是常态而不是异常。** Controller 依赖的 Service 不在扫描范围里，必须用 `@MockitoBean`（Boot 3.4+，旧版 `@MockBean`）顶掉或 `@Import` 显式引入——这是切片测试的固定搭配，第 2 节有完整示例。

---

# 2. `@WebMvcTest`

## 2.1 默认加载内容

**`@WebMvcTest` 装配的是"Spring MVC 的请求处理管线"及其直接协作者**：`@Controller`、`@ControllerAdvice`、`@JsonComponent`；`@WebMvcConfigurer`（拦截器、跨域、消息转换器）；`MockMvc` 与基础设施（DispatcherServlet、HandlerMapping、参数解析器、`HttpMessageConverter`）；Jackson 的 `ObjectMapper`、`Validator`；Servlet `Filter` Bean 与 Spring Security（若 classpath 存在，自动集成进 MockMvc）。

**不会加载**：`@Service`、`@Repository`、普通 `@Component`、DataSource/JPA 配置、消息队列、定时任务。指定 `@WebMvcTest(UserController.class)` 时进一步收窄：**只注册这一个 Controller**，但 `@ControllerAdvice`、配置类仍全量加载——全局异常处理器的行为照常生效。

## 2.2 典型用法

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class) // 只加载 UserController 及 Web 层协作者
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;          // MOCK 环境自动配置

    @MockitoBean                      // Boot 3.4+；旧版本用 @MockBean
    private UserService userService;  // 切片之外的 Bean 必须 Mock，否则启动失败

    @Test
    void should_create_user() throws Exception {
        given(userService.create(any())).willReturn(new User(1L, "tom"));

        mockMvc.perform(post("/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\": \"tom\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1)); // JsonPath 断言响应体
    }
}
```

启动失败的典型报错 `NoSuchBeanDefinitionException: No qualifying bean of type 'UserService'` 就是"切片外依赖没 Mock"——**修法是加 `@MockitoBean` 而不是把 Service 也 `@Import` 进来**（后者会把依赖树整条拉进容器，切片就名存实亡了）。

## 2.3 扩展方式

- **`@Import` 引入自定义配置**：需要真实的转换器、拦截器配置时用 `@Import(WebConfig.class)`；`@Import` 的类不受 TypeExcludeFilter 限制
- **`@MockitoBean` 替换依赖**：Service、外部客户端一律 Mock；Mock 字段集合参与缓存 key，多个测试类保持相同组合才能共享容器
- **`properties` 微调**：`@WebMvcTest(properties = "app.upload.max-size=1MB")` 覆盖 Web 相关配置
- **`excludeFilters`/`excludeAutoConfiguration`**：对扫描范围或自动配置做微调（少用，通常说明切片选择不对）

> **注意**：`@WebMvcTest` 已内含 `@AutoConfigureMockMvc` 与 `@ExtendWith(SpringExtension.class)`，不需要重复声明；它默认 `MOCK` Web 环境，**测不到真实 HTTP 栈**——那属于 `RANDOM_PORT` 端到端测试的职责（第 9 篇）。

---

# 3. `@WebFluxTest`

**响应式栈的对应切片：只加载 WebFlux 的 `@Controller`（注解式）或 `RouterFunction`（函数式路由）及其协作者，注入的客户端是 `WebTestClient` 而不是 MockMvc。** 写法与 `@WebMvcTest` 完全同构——`@WebFluxTest(UserController.class)` 收窄到单个控制器、层外依赖同样必须 `@MockitoBean`、断言换成 `webTestClient.get().uri(...).exchange().expectStatus().isOk()` 链（完整示例见第 9 篇）。

与 `@WebMvcTest` 的对称差异：装配范围换成 WebFlux 基础设施（`WebExceptionHandler`、`RouterFunctionMapping`、编解码器）；Servlet 组件全部不存在；`WebTestClient` 不绑定真实端口。

---

# 4. `@DataJpaTest`

**专注 JPA Repository 测试：自动配置 DataSource、EntityManagerFactory、Spring Data JPA 仓储与事务管理器。** 要点：扫描注册 `@Entity` 实体与 `JpaRepository` 接口；提供 `TestEntityManager`（`persistAndFlush` 一步到位）；默认 `@Transactional` 回滚（内置，不需要再标）；默认用内嵌内存数据库替换真实数据源。

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest // 默认内嵌库 + @Transactional 回滚，均无需再声明
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;   // Spring Data 生成的代理

    @Autowired
    private TestEntityManager entityManager; // 切片提供的便捷入口

    @Test
    void should_find_by_email() {
        entityManager.persistAndFlush(new User("tom@x.com")); // 同一事务内可见

        assertThat(userRepository.findByEmail("tom@x.com")).isPresent();
    }
}
```

两个高频决策点：**要用真实数据库**（验证 MySQL 方言、JSON 列、锁行为）时加 `@AutoConfigureTestDatabase(replace = Replace.NONE)` 关闭内嵌替换并配 Testcontainers；**要加载切片外 Bean** 时用 `@Import`——但先问一句是不是该换成 `@SpringBootTest`。事务回滚细节见第 13 篇。

---

# 5. `@JdbcTest`

**纯 JDBC 场景的更窄切片：自动配置 DataSource、`JdbcTemplate`、事务管理器，不加载 JPA 相关 Bean**（没有 EntityManagerFactory）。默认同样替换为内嵌库并回滚事务。适用：`JdbcTemplate` 手写 SQL 的验证、Flyway/Liquibase 迁移脚本执行、不需要 ORM 的数据访问测试。与 `@DataJpaTest` 的选择标准就一条：**项目持久层用的是什么**。注意它不初始化 Hibernate，测不到 JPA 与 SQL 的交互（二级缓存、flush 时机）。相关的还有 `@DataJdbcTest` 与 `@JooqTest`。

---

# 6. `@JsonTest`

**用于 JSON 序列化与反序列化测试：只装配 Jackson（或 Gson/JSON-B）相关的 Bean，提供类型安全的 `JacksonTester` 工具。**

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.boot.test.json.JacksonTester;

import static org.assertj.core.api.Assertions.assertThat;

@JsonTest
class UserJsonTest {

    @Autowired
    private JacksonTester<User> json; // 泛型化 tester，自动绑定 ObjectMapper

    @Test
    void should_serialize_with_snake_case() throws Exception {
        assertThat(json.write(new User(1L, "tom cat")))     // 序列化
                .extractingJsonPathStringValue("$.user_name")
                .isEqualTo("tom cat");                       // 验证命名策略生效
    }
}
```

`JacksonTester` 使用的 `ObjectMapper` 是 Boot 定制后的版本（`Jackson2ObjectMapperBuilder` 产物）——**这正是它的价值**：验证 `spring.jackson.*` 配置（命名策略、日期格式、null 处理）与自定义 `@JsonComponent`、Module 的真实效果。单元测试里 `new ObjectMapper()` 测的是裸 Jackson，测不到 Boot 定制层。对应物：`GsonTester`、`JsonbTester`。

---

# 7. `@RestClientTest`

**测试"调用外部 HTTP 服务"的客户端组件：装配 `RestTemplate`/`RestClient` 相关 Bean，并提供 `MockRestServiceServer` 在客户端层拦截请求。**

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.client.RestClientTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;

@RestClientTest(UserApiClient.class) // 被测客户端组件
class UserApiClientTest {

    @Autowired
    private UserApiClient userApiClient;

    @Autowired
    private MockRestServiceServer server; // 拦截 RestTemplate 的真实请求

    @Test
    void should_fetch_user_from_remote() {
        server.expect(requestTo("https://api.example.com/users/1"))
              .andRespond(withSuccess("{\"id\": 1, \"name\": \"tom\"}",
                      MediaType.APPLICATION_JSON));     // 预设响应

        assertThat(userApiClient.findById(1L).getName()).isEqualTo("tom");
        server.verify();                                // 校验期望的请求确实发生
    }
}
```

拦截发生在 `RestTemplate` 的请求工厂层——**URL 构造、请求头、序列化、错误处理（4xx/5xx 映射）都被真实执行**，只有网络传输被替身。与 WireMock 的分工：`MockRestServiceServer` 更快更轻（进程内、无端口）；WireMock 起真实 HTTP 服务，覆盖连接超时等传输层行为（第 9 篇）。

---

# 8. 其他常见切片

**Boot 为每种主流技术栈都提供了对应切片，命名规律是 `@Data{技术}Test`（数据访问）与 `@{层}Test`（应用层）：**

| 注解 | 说明 |
|------|------|
| `@DataMongoTest` | MongoDB Repository 测试（默认内嵌 Mongo 或真实连接） |
| `@DataRedisTest` | Redis 测试（Boot 3.x），装配 `RedisTemplate` 与连接工厂 |
| `@DataNeo4jTest` / `@DataJdbcTest` | Neo4j / Spring Data JDBC 测试（后者无 EntityManager） |
| `@JooqTest` / `@GraphQlTest` | jOOQ（DSLContext）/ GraphQL 控制器测试（Boot 3.x，`GraphQlTester`） |
| `@CacheTest` | 缓存抽象测试（CacheManager，不含数据访问） |
| `@WebServiceClientTest` / `@WebServiceServerTest` | SOAP 客户端/服务端 |

> **注意**：切片清单随 Boot 版本演进——`@DataRedisTest`、`@GraphQlTest` 是 Boot 3.x 才有的。没有官方切片的中间件（如 Elasticsearch、Kafka）要么用 `@SpringBootTest` + 排除，要么自定义切片（第 10 节）。

---

# 9. 切片还是 `@SpringBootTest`

## 9.1 选择原则

**判断标准是"断言需要几层装配才成立"，而不是"被测类属于哪一层"：** 只测某一层（路由与参数绑定、Repository 查询、JSON 映射、客户端调用——断言在本层内闭合）→ 切片；多层协作/配置验证（跨 Service 与 Repository 的事务行为、事件发布被监听器消费、"应用能启动"冒烟）→ `@SpringBootTest`；真实 HTTP 栈（过滤器顺序、Content-Type 协商、安全完整链路）→ `@SpringBootTest(RANDOM_PORT)`。实用的组合策略：**日常开发用切片获得快速反馈，主干合并前用少量 `@SpringBootTest` 验证"接起来也对"**——这正是第 2 篇金字塔在注解层的落地。

## 9.2 常见误区

- **切片中引入过多 `@Import` 反而让测试变慢**：每 `@Import` 一个 `@Service`，它的依赖树都要跟着进容器或挨个 Mock——最后等于手工组装了一个迷你全量容器。`@Import` 只用于"少量、必需、依赖树浅"的配置类
- **切片失败时，优先检查"缺失依赖"而不是业务逻辑**：第一类失败是启动期的 `NoSuchBeanDefinitionException`（忘 Mock），读懂报错里的 Bean 名字通常十秒定位
- **把切片当单元测试用**：切片仍是容器型测试，纯逻辑断言永远优先纯 Jupiter
- **Mock 组合发散**：每个测试类 Mock 不同的字段组合 → 每类一个缓存容器。同一 Controller 的多个测试类应保持相同的 `@MockitoBean` 集合（第 5 篇）

---

# 10. 自定义切片

## 10.1 `@AutoConfigure` 系列注解的原理

**官方切片的实现是三件套：`@BootstrapWith`（接入 TestContext）+ `@TypeExcludeFilters`（裁剪扫描）+ `@ImportAutoConfiguration`（裁剪自动配置）。** 以 `@WebMvcTest` 的注解定义为例（简化）：

```java
@BootstrapWith(WebMvcTestContextBootstrapper.class) // 决定 Loader 与 MCC 构建
@ExtendWith(SpringExtension.class)                  // 接入 Jupiter
@ImportAutoConfiguration                            // 导入清单在配套 .imports 文件里
@TypeExcludeFilters(WebMvcTypeExcludeFilter.class)  // 扫描期过滤非 Web Bean
@AutoConfigureWebMvc                                // "自动配置 Xxx"型注解
@AutoConfigureMockMvc                               // 可叠加多个 AutoConfigure 注解
public @interface WebMvcTest { /* ... */ }
```

其中 `@AutoConfigureXxx` 是一族**可组合的自动配置开关**（`@AutoConfigureMockMvc`、`@AutoConfigureTestDatabase`、`@AutoConfigureCache` 等），既能被官方切片组合使用，也能**单独叠加到 `@SpringBootTest` 上**（第 6 篇）。

## 10.2 手写一个切片

**为没有官方切片的技术（如 Kafka）自定义切片，就是仿造三件套**：声明组合注解（`@BootstrapWith(SpringBootTestContextBootstrapper.class)` + `@ExtendWith(SpringExtension.class)` + `@TypeExcludeFilters(KafkaTypeExcludeFilter.class)` + `@ImportAutoConfiguration`）；配套两步——写 `KafkaTypeExcludeFilter`（继承 `TypeExcludeFilter`，声明放行的注解类型集合），在 `META-INF/spring/com.example.KafkaListenerTest.imports` 里列出要导入的自动配置类。

> **行业实践**：自定义切片是"框架组/平台组"的活儿，业务团队通常不值得为单个中间件手写。更常见的轻量替代：`@SpringBootTest(classes = KafkaTestConfig.class)` 用最小配置类圈定装配范围——一行注解拿到切片八成收益。

---

# 11. 小结

| 主题 | 关键结论 |
|------|---------|
| 切片本质 | 装配前裁剪：`@TypeExcludeFilters` 限扫描 + `@ImportAutoConfiguration` 限自动配置 |
| `@WebMvcTest` | Controller/Advice/Filter/Jackson/Security + MockMvc；Service 必须 `@MockitoBean`；指定 value 只注册单个 Controller |
| `@WebFluxTest` | 响应式对称物，客户端是 `WebTestClient` |
| `@DataJpaTest` | Entity + Repository + TestEntityManager；默认内嵌库 + 事务回滚；真实库用 `@AutoConfigureTestDatabase(replace = NONE)` |
| `@JdbcTest` | 无 JPA 的更窄数据切片；选择标准是持久层技术 |
| `@JsonTest` | 验证 Boot 定制后的 ObjectMapper，`new ObjectMapper()` 测不到 |
| `@RestClientTest` | `MockRestServiceServer` 进程内拦截；传输层行为归 WireMock |
| 切片 vs 全量 | 断言在本层闭合用切片；跨层协作、配置验证、真实 HTTP 用 `@SpringBootTest` |
| 自定义切片 | 三件套：Bootstrapper + TypeExcludeFilter + `.imports` 清单 |

下一篇深入切片测试中最常用的工具 MockMvc：它如何不起真实 HTTP 就走完 DispatcherServlet，standalone 与自动配置两种模式的差异，以及请求构建、断言与调试技巧。
