# 1. 测试切片的概念

`@SpringBootTest` 会加载完整应用上下文，速度慢、依赖重。Spring Boot 把应用按功能横切成若干"切片"，每个切片只加载与该功能相关的自动配置和 Bean，这就是 `@WebMvcTest`、`@DataJpaTest`、`@JsonTest` 等注解的由来。`@WebMvcTest` 专注 Web 层，是 MockMvc 最常用的入口。

## 1.1 为什么需要测试切片

一个中型应用完整启动可能要几秒甚至几十秒，如果每个 Controller 测试都启动全部 Bean，测试套件会变得不可接受地慢。切片的思路是：**只加载被测层需要的组件，其他依赖用替身补齐**。这样既保留了框架的真实行为，又把启动成本压到最低。

```text
完整上下文
┌────────────────────────────────────────────┐
│ Controller  Service  Repository  Security  │
│ Jackson     DataSource  Cache   ...        │
└────────────────────────────────────────────┘

@WebMvcTest 切片
┌────────────────────────────────────────────┐
│ Controller  ControllerAdvice  Filter       │
│ Jackson     MockMvc   WebMvcConfigurer     │
└────────────────────────────────────────────┘
        Service / Repository 用 @MockitoBean 补齐
```

**设计考量**：切片不是"更小的 `@SpringBootTest`"，而是"按自动配置类别裁剪"。它加载的 Bean 由 Spring Boot 决定，不由开发者随意指定。理解这一点，才能解释为什么有些 Bean 明明写了却不在容器里。

## 1.2 @WebMvcTest 加载什么

`@WebMvcTest` 会扫描并加载以下类型：

| 类别 | 示例 |
|------|------|
| Controller | `@Controller`、`@RestController` |
| 异常处理 | `@ControllerAdvice`、`@RestControllerAdvice` |
| Web 配置 | `WebMvcConfigurer`、`WebMvcRegistrations` |
| 转换器 | `Converter`、`GenericConverter`、`Formatter` |
| 序列化 | `@JsonComponent`、`Jackson2ObjectMapperBuilderCustomizer` |
| 参数解析 | `HandlerMethodArgumentResolver` |
| 过滤器 | `Filter` |
| 拦截器 | `HandlerInterceptor` |
| 基础设施 | `MockMvc`、Jackson、Validation、Security（若在类路径） |

不在切片内的典型 Bean：

- `@Service`、`@Component`、`@Repository`
- 数据源、JPA、事务管理器
- 自定义业务配置类（除非用 `@Import` 引入）

因此，只要 Controller 依赖 `UserService`，测试就必须提供替身，否则启动时直接报 `No qualifying bean`。

# 2. 基本用法

## 2.1 指定 Controller

最稳妥的写法是在注解里列出本测试关心的 Controller：

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private UserService userService;
}
```

只指定一个 Controller 有两个好处：切片最小、上下文缓存命中率最高。如果写成 `@WebMvcTest` 不带参数，Spring 会加载所有 Controller，启动变慢，还容易因为无关 Controller 的依赖缺失而启动失败。

## 2.2 自动装配 MockMvc

`@WebMvcTest` 自带 `@AutoConfigureMockMvc`，可以直接 `@Autowired MockMvc`。如果使用 `@SpringBootTest`，则必须显式加 `@AutoConfigureMockMvc`，否则容器里没有 `MockMvc` 实例。

```java
@SpringBootTest
@AutoConfigureMockMvc
class FullContextWebTest {

    @Autowired
    private MockMvc mockMvc;
}
```

两种写法对 MockMvc 的初始化方式相同，区别只在上下文规模。

## 2.3 一个完整示例

生产代码：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public UserResponse findById(@PathVariable Long id) {
        return UserResponse.from(userService.findById(id));
    }
}
```

测试代码：

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private UserService userService;

    @Test
    void should_return_user_json() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "Ada"));

        mockMvc.perform(get("/api/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Ada"));
    }
}
```

这段测试同时覆盖了 URL 映射、路径变量、Service 调用、DTO 转换和 JSON 序列化。它没有加载数据库，执行时间通常在几百毫秒以内。

# 3. Bean 过滤与补充

## 3.1 默认过滤规则

`@WebMvcTest` 内部使用一组 `TypeExcludeFilter` 来限制扫描范围。默认只保留前面列出的 Web 相关类型，其余组件一律排除。这正是"切片"的实现方式。

## 3.2 包含与排除过滤器

当项目把某个辅助组件放在 `@Component` 下，却又希望切片加载它时，可以调整过滤规则：

```java
@WebMvcTest(
        controllers = UserController.class,
        useDefaultFilters = false,
        includeFilters = @ComponentScan.Filter(
                type = FilterType.ASSIGNABLE_TYPE,
                classes = UserMapper.class
        )
)
class UserControllerWithMapperTest {
    // 手工控制扫描范围
}
```

更常见的做法是不动过滤器，直接用 `@Import` 引入需要的类：

```java
@WebMvcTest(UserController.class)
@Import({UserMapper.class, WebConfig.class})
class UserControllerImportTest {
}
```

`@Import` 的好处是意图明确、影响范围可控；`useDefaultFilters = false` 则容易把 MVC 基础设施也排除掉，除非确有需要，不建议使用。

## 3.3 用 MockitoBean 替换依赖

Spring Boot 3.4 起，容器内替换 Bean 的推荐注解是 `@MockitoBean`：

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@MockitoBean
private UserService userService;
```

旧版本使用 `@MockBean`：

```java
import org.springframework.boot.test.mock.mockito.MockBean;

@MockBean
private UserService userService;
```

| 维度 | `@MockBean` | `@MockitoBean` |
|------|-------------|----------------|
| 来源 | Spring Boot Test | Spring Framework Test |
| 状态 | Spring Boot 3.4 起废弃 | 推荐 |
| 行为 | 替换或新增 Bean | 覆盖或新增 Bean |
| 对缓存的影响 | 会影响上下文键 | 同样会影响上下文键 |

**注意**：不要用 Mockito 的 `@Mock` 替代 `@MockitoBean`。`@Mock` 只由 `MockitoExtension` 处理，不会进入 Spring 容器，Controller 里注入的仍然是真实 Bean。看到"明明 Mock 了却调用了真实服务"，先检查注解来源。

## 3.4 MockitoBean 的匹配规则

`@MockitoBean` 默认按字段类型匹配容器中的 Bean。如果同一个类型有多个实例，需要指定名称：

```java
@MockitoBean(types = PaymentGateway.class, name = "alipayGateway")
private PaymentGateway alipayGateway;
```

**设计考量**：按类型覆盖简单直接，但会隐藏"到底替换了哪个 Bean"的信息。当一个接口有多个实现时，显式写 `name` 比依赖类型匹配更安全，也能避免测试因新增实现而突然失效。

# 4. 上下文缓存

## 4.1 缓存如何工作

Spring Test 会缓存 `ApplicationContext`，相同配置的测试类复用同一个上下文，避免重复启动。缓存键由一组配置组成，包括：

- 测试切片类型（`@WebMvcTest` / `@SpringBootTest`）
- `@TestPropertySource`、`@DynamicPropertySource`
- `@ActiveProfiles`
- `@Import`、`@MockitoBean` 的字段集合
- `@AutoConfigureMockMvc` 的参数
- 自定义 `ContextCustomizer`

配置相同就命中，配置不同就新建。上下文数量直接决定测试套件的总耗时。

```text
UserControllerTest   ─┐
                      ├─ 配置相同 → 共用 Context A
OrderControllerTest  ─┘

AdminControllerTest  ─── 配置不同 → 新建 Context B
```

## 4.2 减少上下文数量

常见做法：

1. **统一切片范围**：不要一个测试写 `@WebMvcTest(UserController.class)`，另一个写不带参数的 `@WebMvcTest`。
2. **统一属性来源**：把测试属性集中到 `src/test/resources/application-test.yml`，避免每个类写不同的 `@TestPropertySource`。
3. **统一 Mock 集合**：同一组 Controller 的测试尽量使用相同的 `@MockitoBean` 字段集合。
4. **慎用 `@DirtiesContext`**：它会销毁上下文，显著拖慢后续测试。
5. **慎用 `@ActiveProfiles`**：每多一个 profile，通常就多一套上下文。

> **行业实践**：大型项目会把测试配置收敛到少数几个 profile，并在 CI 中打印上下文缓存统计。如果发现测试上下文数量达到几十个，通常不是 Spring 的问题，而是测试配置过于分散。

# 5. 常见错误

| 现象 | 原因 | 修正 |
|------|------|------|
| 启动报缺少 `UserService` | `@WebMvcTest` 不加载 `@Service` | 加 `@MockitoBean UserService` |
| Mock 不生效 | 用了 `@Mock` 而非容器覆盖注解 | 改用 `@MockitoBean` |
| 所有请求 401/403 | Spring Security 自动配置生效 | 用 `@WithMockUser` 或关闭过滤器 |
| 返回 404 | Controller 未被切片包含 | 在 `@WebMvcTest` 中显式列出 |
| 测试越跑越慢 | 上下文缓存未命中 | 统一属性、切片和 Mock 配置 |
| `@Import` 后仍缺 Bean | 被导入类自身还有未满足依赖 | 继续补充 Mock 或导入配置 |
| 过滤器重复执行 | 既被切片扫描又手动 `@Import` | 去掉重复注册 |

# 6. 练习

1. 用 `@WebMvcTest(UserController.class)` 写一个测试，验证 `GET /api/users/1` 返回 200 和正确的 `$.name`，用 `@MockitoBean` 替换 `UserService`。
2. 给同一个 Controller 增加一个 `@Component` 类型的 `UserMapper`，分别用 `@Import` 和 `includeFilters` 两种方式让切片加载它，比较可读性。
3. 列出三种会导致上下文缓存失效的写法，并说明如何把上下文数量降到最低。

# 7. 小结

`@WebMvcTest` 是 Spring Boot 的 Web 层测试切片，只加载 Controller、`@ControllerAdvice`、转换器、过滤器和 MockMvc 等 Web 相关组件，业务依赖用 `@MockitoBean` 替换。它比完整上下文快得多，但要求测试显式补齐缺失 Bean。

| 主题 | 要点 |
|------|------|
| 切片范围 | 仅 Web 层，不含 Service、Repository |
| 指定 Controller | `@WebMvcTest(XxxController.class)` |
| 替换依赖 | `@MockitoBean`，3.4 起替代 `@MockBean` |
| 补充 Bean | 优先 `@Import`，慎用 `useDefaultFilters = false` |
| 性能关键 | 统一配置，提高上下文缓存命中率 |
| 安全 | 类路径有 Security 时会生效，需模拟用户 |

下一篇 `4_request_builders.md` 系统梳理请求构造 API，包括 HTTP 方法、参数、请求头、Cookie、Session、JSON 请求体和文件上传。
