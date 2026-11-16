# 1. Spring Security 测试概述

只要类路径上存在 Spring Security，`@WebMvcTest` 就会自动装配安全过滤器链。原本返回 200 的测试可能突然变成 401 或 403。理解安全测试的三种姿态——**模拟已认证用户、携带 CSRF 令牌、显式关闭过滤器**——是写好 Web 层测试的前提。

## 1.1 依赖

除了 `spring-boot-starter-test`，还需要安全测试支持：

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

它提供 `@WithMockUser`、`@WithUserDetails`、`user()`、`csrf()` 等工具。

## 1.2 安全过滤器链的位置

```text
请求
  │
  ▼
SecurityFilterChain
  ├── SecurityContextPersistenceFilter
  ├── CsrfFilter
  ├── UsernamePasswordAuthenticationFilter
  ├── AuthorizationFilter
  └── ...
  │
  ▼
DispatcherServlet → Controller
```

MockMvc 会把这套过滤器链按真实顺序加入执行流程，因此安全规则、CSRF 校验、认证入口点都能被测试覆盖。

# 2. 用户模拟

## 2.1 @WithMockUser

最简单的模拟方式是在测试方法或类上声明一个虚拟用户：

```java
@WebMvcTest(AdminController.class)
@Import(SecurityConfig.class)
class AdminControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(username = "ada", roles = "ADMIN")
    void admin_can_access_dashboard() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "bob", roles = "USER")
    void normal_user_is_forbidden() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isForbidden());
    }
}
```

`roles = "ADMIN"` 会自动加上 `ROLE_` 前缀，等价于 `authorities = "ROLE_ADMIN"`。如果配置里写的是 `hasAuthority("ADMIN")`，则应使用 `authorities = "ADMIN"`，两者不要混用。

## 2.2 user() 与匿名用户

`@WithMockUser` 作用于整个测试方法，`user()` 只作用于一次请求，适合在同一方法里切换身份：

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;

@Test
void should_allow_owner_but_reject_other_user() throws Exception {
    mockMvc.perform(get("/api/orders/1").with(user("ada").roles("USER")))
            .andExpect(status().isOk());

    mockMvc.perform(get("/api/orders/1").with(user("bob").roles("USER")))
            .andExpect(status().isForbidden());
}
```

验证未登录用户被拦截时，用 `@WithAnonymousUser`。状态码取决于认证入口点：HTTP Basic 通常返回 401，表单登录通常返回 302 跳转，自定义入口点可能返回 JSON。

## 2.3 @WithUserDetails

需要真实的 `UserDetails` 对象时，用 `@WithUserDetails` 指定 Bean 名称：

```java
@Test
@WithUserDetails(value = "ada", userDetailsServiceBeanName = "userDetailsService")
void should_load_real_user_details() throws Exception {
    mockMvc.perform(get("/api/profile"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("ada"));
}
```

它会在执行前调用 `UserDetailsService.loadUserByUsername("ada")`，因此测试数据必须已存在。相比 `@WithMockUser`，它更接近真实登录，但依赖更多。

# 3. CSRF 测试

## 3.1 为什么 POST 会失败

Spring Security 默认开启 CSRF 防护。使用 MockMvc 发 POST 时，如果没有携带令牌，`CsrfFilter` 会直接返回 403。很多"我的 POST 测试莫名 403"都源于此。

## 3.2 携带令牌与验证失败

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;

@Test
@WithMockUser(username = "ada", roles = "USER")
void should_create_order_only_with_valid_csrf() throws Exception {
    mockMvc.perform(post("/api/orders")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"itemId\":1}"))
            .andExpect(status().isCreated());

    mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"itemId\":1}"))
            .andExpect(status().isForbidden());
}
```

`csrf()` 会同时写入请求参数和请求头，模拟浏览器提交隐藏字段的行为；`csrf().asHeader()` 只放请求头，`csrf().useInvalidToken()` 可以模拟非法令牌。只测"带令牌成功"不够，必须再测"不带令牌失败"，否则安全配置被误关也不会有人发现。

## 3.3 关闭 CSRF 的边界

纯 API 服务常见做法是关闭 CSRF：

```java
http.csrf(csrf -> csrf.disable());
```

> **设计考量**：CSRF 防护针对的是基于 Cookie 的会话认证。无状态 JWT API 不依赖浏览器自动携带凭证，关闭 CSRF 是合理的；但只要使用 Session 或 Cookie 认证，就必须保留 CSRF，并在测试里覆盖成功与失败两条路径。

# 4. 认证与授权的断言

## 4.1 401 与 403 的区别

| 状态码 | 含义 | 常见原因 |
|--------|------|----------|
| 401 | 未认证 | 没有登录、令牌无效、认证入口点触发 |
| 403 | 已认证但无权限 | 角色不足、CSRF 校验失败、访问被拒绝 |

测试时要明确区分两者。没登录却断言 403，往往说明认证入口点配置不对；已登录却被断言 401，则可能是会话没有正确建立。

## 4.2 安全配置与测试

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/public/**").permitAll()
                        .requestMatchers("/api/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated())
                .csrf(csrf -> csrf.disable())
                .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

测试类导入这份配置后，分别验证公开接口、管理员访问、普通用户被拒三条路径：

```java
@WebMvcTest(AdminController.class)
@Import(SecurityConfig.class)
class AdminSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void public_endpoint_is_open() throws Exception {
        mockMvc.perform(get("/api/public/ping"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void admin_endpoint_allows_admin() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void admin_endpoint_rejects_normal_user() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isForbidden());
    }
}
```

如果使用了 `@PreAuthorize`，还需在测试配置上开启 `@EnableMethodSecurity`，否则注解不会生效，测试会误以为权限校验已经通过。

# 5. 关闭安全过滤器

只关心 MVC 映射、不关心安全的测试，可以关闭过滤器：

```java
@WebMvcTest(UserController.class)
@AutoConfigureMockMvc(addFilters = false)
class UserControllerNoSecurityTest {
}
```

也可以排除安全自动配置：

```java
@WebMvcTest(controllers = UserController.class,
        excludeAutoConfiguration = SecurityAutoConfiguration.class)
class UserControllerNoSecurityAutoConfigTest {
}
```

**注意**：不要在项目里全局关闭过滤器。安全行为本身也需要测试，`addFilters = false` 只应用于"这个测试与安全无关"的明确场景。

# 6. 反模式

| 反模式 | 问题 | 改进 |
|--------|------|------|
| 只测成功路径 | 权限放宽无法被发现 | 覆盖未认证、角色不足、资源不属于当前用户 |
| 用 `@Mock` 替代 `@MockitoBean` | 替身不进入容器，真实 Bean 仍被调用 | 改用容器覆盖注解 |
| 用 `verify` 锁死内部调用 | 测试绑定实现细节，重构即失败 | 断言 HTTP 契约 |
| 用 `Thread.sleep` 等异步 | 慢且不稳定 | `asyncDispatch` 或 Awaitility |
| 每个测试都上完整上下文 | 测试套件越来越慢 | 优先 `@WebMvcTest` 切片 |
| 依赖测试执行顺序 | 失败难以复现 | 每个测试独立准备和清理 |
| 严格比较整个 JSON | 新增字段就失败 | 用 `jsonPath` 断言关键字段 |

# 7. 最佳实践

- **命名表达行为**：`should_return_404_when_order_not_found` 比 `testFindById` 更能说明业务规则。
- **一个测试一个行为**：创建成功、参数校验、权限拒绝分开写，失败时能直接定位。
- **用对象构造请求体**：`objectMapper.writeValueAsString(command)` 比手写 JSON 安全，字段改名时编译器会提示。
- **断言契约而非实现**：优先状态码、关键字段、错误码、响应头，不断言日志和私有方法。
- **控制上下文数量**：统一切片、属性来源和 Mock 集合，提高缓存命中率。
- **覆盖错误路径**：参数非法、资源不存在、无权限、依赖失败，至少各有一条测试。
- **安全测试常态化**：把 401、403、CSRF 缺失、敏感字段不泄漏纳入日常测试。

> **行业实践**：Spring Framework 6.2 引入了 `MockMvcTester`，用 AssertJ 风格替代部分 `andExpect`。新项目可以评估统一到这种写法，老项目继续用 `MockMvc` 完全没有问题。

# 8. 练习

1. 为受保护的 `GET /api/admin/dashboard` 写三条测试：匿名 401、普通用户 403、管理员 200。
2. 为 `POST /api/orders` 写两条测试：带 `csrf()` 返回 201，不带返回 403。
3. 审查一个现有测试类，找出至少两个反模式，并改写成"断言 HTTP 契约"的形式。

# 9. 小结

Spring Security 一旦进入类路径，就会参与 `@WebMvcTest`。用 `@WithMockUser` 或 `user()` 模拟身份，用 `csrf()` 通过防护，用 401/403 区分认证与授权。关闭过滤器只应作为个别测试的显式选择，而不是全局习惯。

| 主题 | 推荐做法 |
|------|----------|
| 用户模拟 | `@WithMockUser`、`user()`、`@WithUserDetails` |
| CSRF | 成功与失败两条路径都测 |
| 权限断言 | 明确 401 与 403 的语义 |
| Bean 替换 | `@MockitoBean`，不要用 `@Mock` |
| 断言重点 | 状态码、关键字段、错误码 |
| 性能 | 控制上下文数量，优先测试切片 |
| 安全 | 关闭过滤器要局部、有理由 |

八篇文章从定位到安全，构成一条完整的 MockMvc 学习路径：

| 篇目 | 主题 | 核心结论 |
|------|------|----------|
| `1_overview_and_setup.md` | 定位与环境 | MockMvc 替代容器，不替代业务对象 |
| `2_standalone_setup.md` | 独立搭建 | 手工组装，快但缺少自动配置 |
| `3_webmvctest.md` | 测试切片 | 只加载 Web 层，依赖用 `@MockitoBean` |
| `4_request_builders.md` | 请求构造 | 方法、参数、头、会话、请求体、上传 |
| `5_assertions_and_jsonpath.md` | 响应断言 | 状态码、头、jsonPath、整体 JSON |
| `6_filters_and_interceptors.md` | 处理链 | Filter 在 Servlet 层，Interceptor 在 MVC 层 |
| `7_async_and_exception.md` | 异步与异常 | `asyncStarted` + `asyncDispatch`，统一错误结构 |
| `8_security_and_best_practices.md` | 安全与规范 | 用户模拟、CSRF、契约优先 |

至此 MockMvc 模块结束。下一步可以进入 `Java/Testing/TestContainers`，学习如何用真实数据库和中间件补上 Web 层测试覆盖不到的集成场景。
