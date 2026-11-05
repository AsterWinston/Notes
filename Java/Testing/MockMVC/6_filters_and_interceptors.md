# 1. Filter 与 Interceptor 的区别

Filter 属于 Servlet 规范，位于 `DispatcherServlet` 之前；HandlerInterceptor 属于 Spring MVC，位于 `DispatcherServlet` 之后、Controller 之前。两者都能做鉴权、日志、限流，但作用范围和可访问的上下文不同。测试时如果分不清它们，很容易出现"过滤器写了却没执行"或"拦截器拿不到请求体"的问题。

## 1.1 核心差异

| 维度 | Filter | HandlerInterceptor |
|------|--------|--------------------|
| 规范 | Servlet | Spring MVC |
| 执行位置 | DispatcherServlet 之前 | DispatcherServlet 之后 |
| 能否拿到 Handler | 不能 | 能，`preHandle` 的第三个参数 |
| 能否包装请求/响应 | 能 | 不能，只能读取 |
| 能否短路 | 能，不调用 `chain.doFilter` | 能，`preHandle` 返回 `false` |
| 典型用途 | 编码、CORS、TraceId、安全过滤链 | 登录校验、权限注解、Controller 级日志 |
| 测试注册方式 | `addFilters` 或容器 Bean | `addInterceptors` 或 `WebMvcConfigurer` |

**设计考量**：需要修改请求体、包装响应、处理跨域或接入安全框架时，用 Filter；需要知道"即将执行哪个 Controller 方法"、需要读取 `HandlerMethod` 注解时，用 Interceptor。把权限校验放在 Filter 里，往往拿不到方法级注解；放在 Interceptor 里，又无法覆盖静态资源和非 MVC 请求。

## 1.2 执行顺序

```text
HTTP 请求
  │
  ▼
Filter 1 ──► Filter 2 ──► DispatcherServlet
                              │
                              ▼
                     Interceptor 1.preHandle
                              │
                              ▼
                     Interceptor 2.preHandle
                              │
                              ▼
                          Controller
                              │
                              ▼
                     Interceptor 2.postHandle
                              │
                              ▼
                     Interceptor 1.postHandle
                              │
                              ▼
                          视图渲染
                              │
                              ▼
                  Interceptor 2.afterCompletion
                              │
                              ▼
                  Interceptor 1.afterCompletion
                              │
                              ▼
Filter 2 后置逻辑 ◄── Filter 1 后置逻辑
  │
  ▼
HTTP 响应
```

记住两条规律：Filter 的入口正序、出口逆序；Interceptor 的 `preHandle` 正序，`postHandle` 和 `afterCompletion` 逆序。

# 2. 测试 Filter

## 2.1 定义过滤器

以一个写入 TraceId 的过滤器为例：

```java
public class TraceIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString();
        }
        request.setAttribute("traceId", traceId);
        response.setHeader("X-Trace-Id", traceId);
        chain.doFilter(request, response);   // 放行到下一个过滤器或 DispatcherServlet
    }
}
```

这个过滤器同时做了三件事：读取请求头、写入请求属性、回写响应头。测试可以分别验证这三点。

## 2.2 在 standalone 中注册

```java
@BeforeEach
void setUp() {
    mockMvc = MockMvcBuilders
            .standaloneSetup(new EchoController())
            .addFilters(new TraceIdFilter())
            .build();
}

@Test
void should_propagate_trace_id() throws Exception {
    mockMvc.perform(get("/echo/trace").header("X-Trace-Id", "trace-001"))
            .andExpect(status().isOk())
            .andExpect(header().string("X-Trace-Id", "trace-001"))
            .andExpect(content().string("trace-001"));
}
```

`addFilters` 的参数顺序就是过滤器链顺序。需要多个过滤器时，按"先编码、再鉴权、再业务"的顺序传入。

## 2.3 在 @WebMvcTest 中注册

`@WebMvcTest` 会加载 `Filter` 类型的 Bean，但不会自动扫描所有 `@Component`。如果过滤器是 `@Component`，最稳妥的方式是显式导入：

```java
@WebMvcTest(EchoController.class)
@Import(TraceIdFilter.class)
class EchoControllerFilterTest {

    @Autowired
    private MockMvc mockMvc;
}
```

如果过滤器已经在切片内，又用 `@Import` 导入了同一个类，会出现重复注册，导致响应头被写两次。排查时看响应头是否重复即可确认。

**注意**：`@WebMvcTest` 默认不会加载 `FilterRegistrationBean` 之外的完整 Servlet 容器配置，某些依赖容器初始化的过滤器可能行为不完整。涉及真实过滤器链顺序的测试，应放到 `@SpringBootTest` 或真实端口测试中。

# 3. 测试 HandlerInterceptor

## 3.1 定义拦截器

```java
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        Object user = request.getAttribute("currentUser");
        if (user == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setCharacterEncoding("UTF-8");
            response.getWriter().write("{\"code\":\"UNAUTHORIZED\"}");
            return false;   // 不再调用 Controller
        }
        return true;
    }
}
```

返回 `false` 后，Controller 不会执行，`postHandle` 也不会执行，但已经通过的拦截器的 `afterCompletion` 仍会执行。

## 3.2 在 standalone 中注册

```java
mockMvc = MockMvcBuilders
        .standaloneSetup(new EchoController())
        .addInterceptors(new AuthInterceptor())
        .build();

@Test
void should_reject_request_without_user() throws Exception {
    mockMvc.perform(get("/echo/user"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.code").value("UNAUTHORIZED"));
}

@Test
void should_allow_request_with_user() throws Exception {
    mockMvc.perform(get("/echo/user")
                    .requestAttr("currentUser", new User(1L, "Ada")))
            .andExpect(status().isOk())
            .andExpect(content().string("Ada"));
}
```

## 3.3 在 @WebMvcTest 中注册

拦截器通常通过 `WebMvcConfigurer` 注册：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

`@WebMvcTest` 会加载 `WebMvcConfigurer`，但 `@Configuration` 类本身是否被扫描取决于包结构，更明确的写法是在测试类上 `@Import(WebConfig.class)`。

**设计考量**：拦截器测试的重点不是"拦截器代码本身"，而是"路径匹配规则是否生效"。`addPathPatterns` 和 `excludePathPatterns` 是配置错误的高发区，应该用"命中路径"和"排除路径"两条测试分别验证。

# 4. 请求属性与上下文传递

## 4.1 请求属性

Filter 和 Interceptor 都可以向 `HttpServletRequest` 写属性，Controller 通过 `@RequestAttribute` 读取：

```java
@GetMapping("/echo/trace")
public String echoTrace(@RequestAttribute("traceId") String traceId) {
    return traceId;
}
```

测试时既可以依赖真实过滤器写入，也可以直接设置：

```java
mockMvc.perform(get("/echo/trace").requestAttr("traceId", "manual-001"))
        .andExpect(content().string("manual-001"));
```

前者验证链路，后者验证 Controller 契约，按测试目的选择。

## 4.2 ThreadLocal 与 SecurityContext

有些上下文不放在 request 里，而是放在 `ThreadLocal`，例如 `SecurityContextHolder`、MDC、自定义用户上下文。MockMvc 在同一个线程里执行请求，所以这些上下文通常能生效，但有两个坑：

- **请求结束后必须清理**，否则会污染同一个线程里的后续测试。
- **异步请求会切换线程**，`ThreadLocal` 不一定能传过去，除非使用 `DelegatingSecurityContextExecutor` 之类的包装。

直接操作 `SecurityContextHolder` 时，必须在 `finally` 中调用 `clearContext()`，否则会污染同一线程里的后续测试。异步请求还会切换线程，ThreadLocal 不会自动传播，测试时如果 `currentUser` 突然变成 `null`，先检查上下文传播。

> **注意**：直接操作 `SecurityContextHolder` 只适合理解机制。真正的安全测试应使用 Spring Security Test 提供的 `@WithMockUser` 或 `user()`，它们会自动处理上下文创建和清理。

# 5. 鉴权上下文的测试策略

## 5.1 三种模拟方式

| 方式 | 适用 | 优点 | 局限 |
|------|------|------|------|
| 设置 `requestAttr` | 自定义拦截器 | 简单、无依赖 | 不覆盖真实过滤器 |
| 操作 `SecurityContextHolder` | 临时验证 | 接近真实机制 | 需手动清理 |
| `@WithMockUser` / `user()` | Spring Security | 官方支持、自动清理 | 需要安全测试依赖 |

## 5.2 自定义鉴权头的例子

如果系统用 `Authorization` 头解析用户，测试应同时覆盖"有头"和"无头"：无头时断言 401，有效 token 时断言 200 并检查关键字段，必要时再补一条角色不足的 403。**设计考量**：鉴权测试不要只测成功路径，401 和 403 的区分、过期 token、角色不足，才是线上事故的高发点。

# 6. 执行顺序的验证

## 6.1 用响应头记录顺序

验证过滤器顺序最直接的方式是让每个过滤器往响应头追加标记：进入时写 `name-before`，调用 `chain.doFilter` 后再写 `name-after`。测试断言 `X-Order` 的多个值顺序：

```java
mockMvc.perform(get("/echo/order"))
        .andExpect(header().stringValues("X-Order",
                "first-before", "second-before", "second-after", "first-after"));
```

## 6.2 短路与清理

如果某个过滤器不调用 `chain.doFilter`，后续过滤器和 Controller 都不会执行，但前面已经执行的过滤器仍会走自己的后置逻辑。测试短路行为时，除了断言状态码，还应断言响应头里没有后续过滤器的标记。

# 7. 常见错误

| 现象 | 原因 | 修正 |
|------|------|------|
| 过滤器没执行 | 未注册，或不是切片内 Bean | `addFilters` 或 `@Import` |
| 响应头重复 | 过滤器被注册两次 | 去掉重复的 `@Import` 或 Bean 声明 |
| 拦截器没执行 | standalone 未 `addInterceptors` | 显式注册，或检查路径匹配 |
| `@RequestAttribute` 为 `null` | 上游没有写入属性 | 检查过滤器/拦截器逻辑 |
| 测试之间互相污染 | ThreadLocal 未清理 | `finally` 中 `clear()` |
| 异步请求拿不到用户 | ThreadLocal 未传播 | 使用上下文包装器 |
| 顺序与预期不符 | 注册顺序理解反了 | 过滤器和拦截器分别记忆正序/逆序 |

# 8. 练习

1. 写一个 `TraceIdFilter`，当请求头没有 `X-Trace-Id` 时生成 UUID 并写回响应头，测试"有值"和"无值"两种情况。
2. 写一个 `AuthInterceptor`，对 `/api/**` 校验 `requestAttr("currentUser")`，对 `/api/public/**` 放行，分别测试三条路径。
3. 注册两个过滤器，用响应头记录执行顺序，验证入口正序、出口逆序。

# 9. 小结

Filter 位于 Servlet 层，能包装请求响应；Interceptor 位于 MVC 层，能感知 Handler。测试 Filter 用 `addFilters` 或 `@Import`，测试 Interceptor 用 `addInterceptors` 或 `WebMvcConfigurer`。请求属性是最轻量的上下文传递方式，ThreadLocal 则要谨慎处理清理和异步传播。

| 主题 | 要点 |
|------|------|
| Filter 注册 | standalone `addFilters`，切片内 `@Import` |
| Interceptor 注册 | `addInterceptors` 或 `WebMvcConfigurer` |
| 执行顺序 | Filter 入口正序出口逆序，Interceptor 同理 |
| 上下文传递 | 优先 `requestAttr`，ThreadLocal 要清理 |
| 鉴权测试 | 成功、401、403 三条路径都要覆盖 |
| 异步注意 | ThreadLocal 不会自动跨线程 |

下一篇 `7_async_and_exception.md` 讨论异步请求的 `asyncDispatch`、统一异常处理、错误响应结构与超时场景。
