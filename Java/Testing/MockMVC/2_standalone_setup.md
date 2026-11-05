# 1. standaloneSetup 的定位

`@WebMvcTest` 会启动一个缩小的 Spring 容器，自动装配 MVC 基础设施。`standaloneSetup` 则更进一步：**完全不启动容器**，由测试代码手工组装 Controller、消息转换器、异常处理器和拦截器，再交给 `MockMvcBuilders` 构建。它适合验证单个 Controller 的输入输出契约，也适合在没有 Spring Boot 上下文的环境里快速写测试。

## 1.1 与容器测试的区别

两者的核心差异在于"谁来装配依赖"：

| 维度 | `standaloneSetup` | `@WebMvcTest` |
|------|-------------------|----------------|
| 是否启动容器 | 否 | 是，仅 Web 切片 |
| Bean 由谁创建 | 测试代码手工 new 或 mock | Spring 容器 |
| 消息转换器 | 默认一套，可覆盖 | 与自动配置一致 |
| 异常处理器 | 需显式 `setControllerAdvice` | 自动扫描 `@ControllerAdvice` |
| 拦截器 | 需显式 `addInterceptors` | 自动加载 `WebMvcConfigurer` |
| 过滤器 | 默认无，需 `addFilters` | 会加载 `Filter` Bean |
| 启动速度 | 最快 | 较快 |
| 适用场景 | Controller 纯单元测试 | Web 层集成测试 |

`standaloneSetup` 的关键词是"手工"：少了自动配置的便利，也少了它的隐性行为。测试里出现的每个组件都能在源码里找到对应的注册语句，排错时不需要猜容器加载了什么。

## 1.2 何时选择 standaloneSetup

**推荐使用**：

- Controller 逻辑复杂，但依赖很少，希望测试毫秒级完成。
- 项目不是 Spring Boot，没有 `@WebMvcTest` 这类切片注解。
- 需要精确控制消息转换器、异常处理器，验证特定配置下的行为。
- 想单独验证某个 `@ControllerAdvice` 的映射规则。

**不推荐使用**：

- 需要验证 Spring Boot 自动配置、安全过滤器链、真实 Bean 协作。
- Controller 依赖大量容器基础设施（参数解析器、类型转换服务）。
- 团队已经统一用 `@WebMvcTest`，单独引入一套风格会增加维护成本。

> **行业实践**：大多数 Spring Boot 项目以 `@WebMvcTest` 为主，`standaloneSetup` 只用于个别"想彻底隔离容器"的测试。不要为了追求速度把所有 Controller 测试都改成 standalone，否则会逐渐失去对真实配置的覆盖。

# 2. 最小的 standalone 测试

## 2.1 手工构建 MockMvc

先看被测 Controller。它依赖一个 `UserService`，并暴露两个接口：

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public User findById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    public ResponseEntity<User> create(@RequestBody CreateUserCommand command) {
        User created = userService.create(command);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

测试代码在 `@BeforeEach` 里组装 `MockMvc`：

```java
class UserControllerStandaloneTest {

    private MockMvc mockMvc;
    private UserService userService;

    @BeforeEach
    void setUp() {
        userService = mock(UserService.class);
        mockMvc = MockMvcBuilders
                .standaloneSetup(new UserController(userService))
                .build();
    }

    @Test
    void should_return_user_by_id() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "Ada"));

        mockMvc.perform(get("/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("Ada"));
    }
}
```

`standaloneSetup` 可以接收多个 Controller，也可以直接传 `new UserController(...)`。它返回一个 `StandaloneMockMvcBuilder`，后续所有定制都链式挂在这个 builder 上。

## 2.2 依赖注入的处理

standalone 测试没有容器，构造器参数必须自己提供。这带来一个好处：依赖关系在测试里一目了然。坏处是 Controller 依赖一多，`setUp` 就会变长。

**设计考量**：如果一个 Controller 需要注入五六个依赖才能构造，通常说明它承担了过多职责。测试的装配成本是设计耦合度的一面镜子，值得在重构时参考，而不是简单地用反射或字段注入绕过去。

# 3. 配置消息转换器

## 3.1 默认转换器

`standaloneSetup` 默认会注册一套常用转换器，包括 `MappingJackson2HttpMessageConverter`。如果项目使用标准 Jackson 配置，通常不需要额外设置。但默认实例与 Spring Boot 自动配置的 `ObjectMapper` 不是同一个，差异会体现在日期格式、命名策略、未知字段处理上。

## 3.2 自定义 ObjectMapper

当接口契约依赖特定序列化规则时，应显式传入转换器：

```java
ObjectMapper objectMapper = JsonMapper.builder()
        .addModule(new JavaTimeModule())
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
        .build();

mockMvc = MockMvcBuilders
        .standaloneSetup(new OrderController(orderService))
        .setMessageConverters(new MappingJackson2HttpMessageConverter(objectMapper))
        .build();
```

对应的响应字段会从 `createdAt` 变成 `created_at`，时间从时间戳变成 ISO-8601 字符串。测试断言必须与这套配置一致：

```java
mockMvc.perform(get("/orders/1"))
        .andExpect(jsonPath("$.created_at").value("2026-11-06T10:00:00"))
        .andExpect(jsonPath("$.createdAt").doesNotExist());
```

**注意**：`setMessageConverters` 会替换整套转换器，而不是追加。调用之后，默认的字符串、字节数组、表单转换器都不再生效。如果还需要它们，应把默认列表取出来再追加，或者只注册需要的组合。

# 4. 异常处理器

## 4.1 setControllerAdvice

standalone 模式不会扫描 `@RestControllerAdvice`，必须手工注册：

```java
mockMvc = MockMvcBuilders
        .standaloneSetup(new OrderController(orderService))
        .setControllerAdvice(new GlobalExceptionHandler())
        .build();
```

异常处理器本身可以是一个普通实例：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ApiError("ORDER_NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ApiError> handleBadRequest(IllegalArgumentException ex) {
        return ResponseEntity.badRequest()
                .body(new ApiError("INVALID_ARGUMENT", ex.getMessage()));
    }
}
```

注册后，Controller 抛出的异常会走与生产一致的解析流程，测试可以直接断言错误结构：

```java
@Test
void should_return_404_when_order_missing() throws Exception {
    when(orderService.find(99L)).thenThrow(new OrderNotFoundException(99L));

    mockMvc.perform(get("/orders/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("ORDER_NOT_FOUND"));
}
```

## 4.2 多个 advice 的注册顺序

`setControllerAdvice` 支持可变参数，也支持传入多个实例。顺序会影响异常匹配：Spring 会遍历 advice 列表，找到第一个能处理该异常的方法。如果两个 advice 都声明了 `@ExceptionHandler(Exception.class)`，排在前面的会生效。

> **注意**：standalone 模式按传入顺序处理 advice，不经过包扫描。需要控制优先级时，把更具体的 advice 放在前面，或者合并成一个 advice，用方法级 `@ExceptionHandler` 精确匹配。

# 5. 其他定制

## 5.1 拦截器

拦截器通过 `addInterceptors` 注册：

```java
mockMvc = MockMvcBuilders
        .standaloneSetup(new OrderController(orderService))
        .addInterceptors(new AuthInterceptor())
        .build();
```

注册顺序就是执行顺序。`preHandle` 按注册顺序正序执行，`postHandle` 和 `afterCompletion` 逆序执行。测试拦截器时，通常关注两件事：请求是否被放行，以及拦截器写入的请求属性是否被 Controller 读到。

## 5.2 过滤器

standalone 默认不加载任何过滤器。需要时显式添加：

```java
mockMvc = MockMvcBuilders
        .standaloneSetup(new OrderController(orderService))
        .addFilters(new TraceIdFilter())
        .build();
```

`addFilters` 的参数顺序就是过滤器链顺序。过滤器在 `DispatcherServlet` 之前执行，因此它能修改请求头、包装请求体、直接短路响应，这些行为在测试里都能观察到。

## 5.3 默认请求与响应

当测试需要统一的请求头或字符集时，可以设置默认值：

```java
mockMvc = MockMvcBuilders
        .standaloneSetup(new OrderController(orderService))
        .defaultRequest(get("/").header("X-Client", "test"))
        .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
        .build();
```

`defaultRequest` 只影响未显式覆盖的部分。如果某次 `perform` 里又设置了同名头，以 `perform` 为准。

# 6. standaloneSetup 的限制

standalone 模式省掉了容器，也省掉了容器提供的能力。以下内容不会自动生效：

- **自动配置**：Jackson、Validation、ContentNegotiation 都按默认规则，不读取 `application.yml`。
- **组件扫描**：`@ControllerAdvice`、`WebMvcConfigurer`、`Filter` 都不会被自动发现。
- **参数解析器**：自定义 `HandlerMethodArgumentResolver` 需要手工注册。
- **安全过滤器链**：Spring Security 不会参与，`@WithMockUser` 也无效。
- **真实 Bean 协作**：所有依赖都要 mock 或手工构造。

**设计考量**：这些限制让 standalone 测试非常"白盒"，适合验证 Controller 自身的契约，不适合验证框架集成。如果测试目标是"应用在真实配置下能否正常工作"，就应该换用 `@WebMvcTest` 或 `@SpringBootTest`。

# 7. 常见错误

| 现象 | 原因 | 修正 |
|------|------|------|
| 异常直接抛出测试 | 没有注册 `setControllerAdvice` | 加 `.setControllerAdvice(new GlobalExceptionHandler())` |
| 日期格式与生产不一致 | 使用了默认 ObjectMapper | 显式 `setMessageConverters` |
| 自定义转换器失效 | `setMessageConverters` 覆盖了默认集合 | 手工组装完整列表 |
| 过滤器没执行 | standalone 不自动加载 | 用 `addFilters` 注册 |
| 拦截器顺序不对 | 注册顺序即执行顺序 | 调整 `addInterceptors` 参数顺序 |
| 安全注解无效 | 没有 Spring Security 上下文 | 改用 `@WebMvcTest` + Spring Security Test |
| 校验不生效 | 没有可用的 Validator | 显式配置 `setValidator` |

# 8. 练习

1. 用 `standaloneSetup` 为 `OrderController` 写测试，要求覆盖成功查询与 `OrderNotFoundException` 两条路径。
2. 给 `MockMvc` 配置一个使用 `SNAKE_CASE` 的 `ObjectMapper`，并调整断言验证字段名变化。
3. 注册一个记录请求耗时的拦截器，验证它写入的请求属性能在 Controller 中读到。

# 9. 小结

`standaloneSetup` 用测试代码手工组装 MVC 组件，完全不启动 Spring 容器。它启动极快、控制力强，适合 Controller 单元测试；代价是所有基础设施都要显式配置，无法验证自动配置与安全链路。

| 配置项 | API | 说明 |
|--------|-----|------|
| Controller | `standaloneSetup(...)` | 可传多个实例 |
| 异常处理 | `setControllerAdvice(...)` | 不自动扫描 |
| 消息转换 | `setMessageConverters(...)` | 整体替换 |
| 拦截器 | `addInterceptors(...)` | 顺序即执行顺序 |
| 过滤器 | `addFilters(...)` | 默认不加载 |
| 默认请求 | `defaultRequest(...)` | 可被单次请求覆盖 |

下一篇 `3_webmvctest.md` 回到容器内测试，讨论 `@WebMvcTest` 的切片边界、Bean 过滤、`@MockitoBean` 的用法，以及测试上下文缓存对执行速度的影响。
