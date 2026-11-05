# 1. Spring 测试中的 Mockito

纯单元测试用 `@ExtendWith(MockitoExtension.class)` 就够了，但 Spring 项目还有另一类测试：需要启动 Web 层、验证 JSON 序列化、检查 URL 映射和过滤器链。这类测试既不想启动完整容器，又需要 Spring 的 MVC 基础设施。Spring Boot 的**测试切片（Test Slice）**配合 Mockito 的 Bean 替换能力，正好填补这个区间。

## 1.1 单元测试与容器测试的分界

`MockitoExtension` 管理的是测试类里的 `@Mock` 字段，它不启动 Spring 容器，也不知道 Bean 是什么。Spring 测试则相反：容器负责创建 Bean，Mockito 负责把容器里的某个 Bean 替换成替身。两者的生命周期和注入方式完全不同。

| 维度 | `MockitoExtension` | Spring 测试切片 |
|------|--------------------|-----------------|
| 是否启动容器 | 否 | 是，但只加载切片 |
| 替身注解 | `@Mock` / `@Spy` | `@MockitoBean` / `@MockitoSpyBean` |
| 依赖注入者 | Mockito | Spring TestContext |
| 适用对象 | 普通业务类 | Controller、Repository、Bean 协作 |
| 常见错误 | 忘记扩展注解 | 用 `@Mock` 代替 Bean 覆盖 |

理解这条分界后，下面的切片选择和 Bean 替换就不会混在一起。

## 1.2 测试层次与切片选择

Spring Boot 提供了一组以 `@...Test` 结尾的切片注解，每种只加载应用的一部分配置：

| 注解 | 加载内容 | 典型用途 |
|------|----------|----------|
| `@WebMvcTest` | Controller、`@ControllerAdvice`、Filter、Jackson、MockMvc | Web 层契约 |
| `@DataJpaTest` | JPA、DataSource、Repository | 持久层查询 |
| `@JsonTest` | Jackson、JSON 序列化器 | DTO 序列化 |
| `@SpringBootTest` | 完整应用上下文 | 集成测试、端到端 |
| `@RestClientTest` | REST 客户端、JSON | 外部 API 客户端 |

```text
┌──────────────────────────────────────────────────────────┐
│                      测试范围由宽到窄                       │
│                                                          │
│  @SpringBootTest  ─────────────────────────── 完整上下文    │
│        │                                                 │
│        ├── @WebMvcTest        ────────────── 只加载 Web 层  │
│        ├── @DataJpaTest       ────────────── 只加载 JPA 层  │
│        ├── @JsonTest          ────────────── 只加载 JSON 层 │
│        └── @RestClientTest    ────────────── 只加载客户端   │
│                                                          │
│  MockitoExtension  ────────────────────────── 纯单元测试    │
└──────────────────────────────────────────────────────────┘
```

切片越窄，启动越快、反馈越早，但也越依赖 Mock 来补齐缺失的 Bean。**设计考量**：Controller 逻辑用 `@WebMvcTest`，Repository 查询用 `@DataJpaTest`，真正的跨层协作再交给 `@SpringBootTest`。不要为了"测试像生产"而把所有测试都写成完整上下文。

## 1.3 @MockBean 与 Spring Boot 3.4 的变化

在 Spring Boot 3.3 及更早版本中，替换容器里的 Bean 通常用 `@MockBean` 和 `@SpyBean`：

```java
// Spring Boot 3.3 及以前的写法，3.4 起标记为废弃
@MockBean
private UserService userService;

@SpyBean
private PriceCalculator priceCalculator;
```

从 **Spring Boot 3.4 / Spring Framework 6.2** 开始，官方推荐改为 `@MockitoBean` 和 `@MockitoSpyBean`：

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;

@MockitoBean
private UserService userService;

@MockitoSpyBean
private PriceCalculator priceCalculator;
```

两者的关键差异如下：

| 维度 | `@MockBean` / `@SpyBean` | `@MockitoBean` / `@MockitoSpyBean` |
|------|--------------------------|-------------------------------------|
| 所属模块 | Spring Boot Test | Spring Framework Test |
| 状态 | 3.4 起废弃 | 推荐使用 |
| 与切片集成 | 支持 | 支持 |
| 底层机制 | Spring Boot 的上下文定制器 | Spring Framework 的 Bean 覆盖机制 |
| 对缓存的影响 | 会影响测试上下文缓存 | 仍应避免在同一测试类中频繁变化 |

> **行业实践**：如果项目还在 Spring Boot 3.2 或 3.3，升级前可以继续使用 `@MockBean`，但应把新测试写成 `@MockitoBean` 的目标形态；升级到 3.4 后统一迁移，避免编译告警堆积。

### 1.3.1 三个容易混淆的注解

| 注解 | 来源 | 作用范围 | 是否会注册到 Spring 容器 |
|------|------|----------|--------------------------|
| `@Mock` | Mockito | 测试类字段 | 否 |
| `@MockBean` | Spring Boot Test | Spring 上下文 | 是，替换或新增 Bean |
| `@MockitoBean` | Spring Framework Test | Spring 上下文 | 是，覆盖或新增 Bean |

在 `@SpringBootTest` 里混用 JUnit 的 `@Mock` 和 Spring 的 `@MockitoBean` 是常见错误。`@Mock` 只被 `MockitoExtension` 处理，不会进入 Spring 容器；Controller 里注入的仍然是真实 Bean。看到"我明明 Mock 了，为什么还是调用了真实服务"，先检查注解来源。

## 1.4 @WebMvcTest 与 MockMvc 配合

下面用一个最小 Controller 演示完整流程。先看生产代码：

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

测试只加载 Web 层，用 `@MockitoBean` 替换业务服务：

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private UserService userService;

    @Test
    void should_return_user_json() throws Exception {
        when(userService.getUser(1L)).thenReturn(new User(1L, "Ada"));

        mockMvc.perform(get("/users/{id}", 1L))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Ada"));
    }
}
```

这段测试同时验证了 URL 映射、路径参数绑定、JSON 序列化和 HTTP 状态码。它没有启动数据库，也没有加载 `@Service`，所以执行速度通常比 `@SpringBootTest` 快一个数量级。

如果项目还在 Spring Boot 3.3，把 `@MockitoBean` 换成 `@MockBean` 即可，其余代码不变。

### 1.4.1 MockMvc 常用断言

| 断言 | 作用 |
|------|------|
| `status().isOk()` | 状态码 200 |
| `status().isNotFound()` | 状态码 404 |
| `jsonPath("$.id").value(1)` | JSON 字段值 |
| `jsonPath("$.items").isArray()` | 字段是数组 |
| `content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON)` | 响应类型 |
| `header().string("Location", "/users/1")` | 响应头 |
| `content().json("{...}")` | 整体 JSON 比较 |

### 1.4.2 请求体与自定义参数

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.http.MediaType.APPLICATION_JSON;

mockMvc.perform(post("/users")
                .contentType(APPLICATION_JSON)
                .content("""
                        {"name": "Ada"}
                        """))
        .andExpect(status().isCreated());
```

`MockMvc` 默认不启动真实 HTTP 服务器，而是直接调用 DispatcherServlet。因此它验证的是 Spring MVC 的处理链，不是网络层、TLS 或真实的 Tomcat 行为。需要真实端口时，应使用 `@SpringBootTest(webEnvironment = RANDOM_PORT)` 配合 `TestRestTemplate` 或 `WebTestClient`。

## 1.5 测试切片与上下文缓存

Spring Test 会缓存 `ApplicationContext`，避免每个测试类都重新启动容器。缓存键由一组配置共同决定：测试切片类型、`@TestPropertySource`、`@MockitoBean` 定义、`@Import`、`@ActiveProfiles` 等。配置不同，通常就会创建新的上下文。

```text
┌──────────────────────┐
│  UserControllerTest  │ ── @WebMvcTest(UserController.class)
│                      │ ── @MockitoBean UserService
└──────────┬───────────┘
           │ 配置相同
           ▼
┌──────────────────────┐
│  OrderControllerTest │ ── 复用同一个缓存上下文
└──────────────────────┘

┌──────────────────────┐
│  AdminControllerTest │ ── @WebMvcTest(AdminController.class)
│                      │ ── 不同的切片配置，创建新上下文
└──────────────────────┘
```

**设计考量**：上下文缓存是 Spring 测试性能的关键。常见的"测试越跑越慢"往往不是 Mockito 的问题，而是缓存失效：

- 每个测试类都加不同的 `@TestPropertySource` 或 `@DynamicPropertySource`。
- 同一个切片里，`@MockitoBean` 的字段集合不同。
- 随意使用 `@DirtiesContext`，导致上下文被销毁重建。
- 不同测试类使用不同的 `@ActiveProfiles`。

> **注意**：`@DirtiesContext` 应该作为最后手段。它会清空缓存，显著拖慢测试套件。大多数"测试之间状态污染"可以通过每个测试独立的数据库清理、事务回滚或独立 Mock 解决，而不是重建整个容器。

## 1.6 Spring Boot 3 的关键变化

从 Spring Boot 2.x 升级到 3.x 时，测试代码需要关注以下变化：

| 变化 | 影响 | 处理方式 |
|------|------|----------|
| Java 17 基线 | 旧 JDK 无法运行 | 升级 JDK 与编译目标 |
| Jakarta EE | `javax.servlet` 改为 `jakarta.servlet` | 替换导入包 |
| Spring Framework 6.2 | `@MockBean` 废弃 | 迁移到 `@MockitoBean` |
| Spring Security 6 | 默认安全配置更严格 | 使用 `@WithMockUser` 或关闭过滤器 |
| Mockito 5 | 默认 inline mock maker | final/static Mock 不再需要额外依赖 |
| `@WebMvcTest` | 仍只加载 Web 相关 Bean | 业务依赖必须替换或导入 |

Spring Security 6 下，`@WebMvcTest` 默认会加载安全过滤器链。如果测试没有配置认证，可能得到 401 或 403，而不是预期的 200：

```java
import org.springframework.security.test.context.support.WithMockUser;

@Test
@WithMockUser(username = "admin", roles = "ADMIN")
void should_allow_authenticated_user() throws Exception {
    mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk());
}
```

如果测试根本不关心安全，可以关闭过滤器链：

```java
@WebMvcTest(UserController.class)
@AutoConfigureMockMvc(addFilters = false)
class UserControllerNoSecurityTest {
    // 只验证 MVC 映射，不验证安全行为
}
```

但**不要**为了省事在所有 Web 测试里关闭安全过滤器。安全行为本身也需要测试，关闭过滤器应该只用于"这个测试与安全无关"的明确场景。

## 1.7 常见错误与排错

| 现象 | 原因 | 修正 |
|------|------|------|
| `No qualifying bean of type UserService` | `@WebMvcTest` 不加载 `@Service` | 加 `@MockitoBean UserService` |
| Mock 不生效，仍调用真实服务 | 用了 Mockito 的 `@Mock` 而非 Spring 的 Bean 覆盖 | 改用 `@MockitoBean` |
| `@MockBean` 编译告警 | Spring Boot 3.4 已废弃 | 迁移到 `@MockitoBean` |
| `MockMvc` 为 `null` | 没有 `@WebMvcTest` 或 `@AutoConfigureMockMvc` | 补注解 |
| 所有请求都 403 | Spring Security 默认拦截 | 用 `@WithMockUser` 或 `addFilters = false` |
| 404 找不到 Controller | 启动类包路径不包含 Controller | 检查包结构与 `@SpringBootApplication` 扫描范围 |
| JSON 字段缺失 | DTO getter/record 组件不匹配 | 检查序列化配置与字段名 |
| 测试上下文数量暴涨 | 每个类使用不同属性或 Mock 组合 | 统一测试配置，减少 `@DirtiesContext` |
| `@SpyBean` 验证失败 | 拿到的是 AOP 代理 | 用 `AopTestUtils.getTargetObject` 取目标对象 |

# 2. 练习与自检

1. 用 `@WebMvcTest` 和 `@MockitoBean` 写一个测试，验证 `GET /users/1` 返回 200 和正确的 `$.name`。
2. Spring Boot 3.4 为什么把 `@MockBean` 标记为废弃？迁移后的注解是什么？
3. 列出三种会导致 Spring 测试上下文缓存失效的常见写法，并说明如何减少缓存数量。

# 3. 小结

Spring 测试里的 Mockito 有两套用法：纯单元测试用 `MockitoExtension` + `@Mock`，容器内测试用 `@MockitoBean` / `@MockitoSpyBean`。`@WebMvcTest` 负责 Web 层切片，`MockMvc` 负责在不启动真实服务器的情况下验证 MVC 处理链；两者结合可以用较低成本覆盖 URL、参数、JSON 和状态码。

| 场景 | 推荐组合 |
|------|----------|
| Controller 契约测试 | `@WebMvcTest` + `@MockitoBean` + `MockMvc` |
| 完整链路测试 | `@SpringBootTest` + `@AutoConfigureMockMvc` |
| 只测 JSON | `@JsonTest` |
| 只测 Repository | `@DataJpaTest` |
| 纯业务单元测试 | `@ExtendWith(MockitoExtension.class)` + `@Mock` |
| 部分真实 Bean | `@MockitoSpyBean`，谨慎使用 |

下一篇 `8_anti_patterns_and_summary.md` 收束整个模块，系统讨论过度 Mock、验证实现细节、Mock 第三方库等反模式，并给出 Mockito 的使用决策清单。
