# 1. Spring Security 测试支持

Web 应用一旦接入 Spring Security，每个 HTTP 测试都要回答三个问题：**当前请求以谁的身份发出？CSRF 令牌带了吗？权限够不够？** 手工构造 `SecurityContext` 和伪造 CSRF 头既繁琐又易碎，`spring-security-test` 提供的就是一整套"以任意身份、带齐凭证发请求"的测试工具。

## 1.1 依赖准备

`spring-boot-starter-test` 默认不包含安全测试模块，需要显式引入：

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

Gradle：

```groovy
testImplementation 'org.springframework.security:spring-security-test'
```

版本由 Boot BOM 管理，无需手写。该库随 Security 6.x 提供 `@WithMockUser`、`SecurityMockMvcRequestPostProcessors`（`user()`/`csrf()`/`jwt()`/`oidcLogin()`）与 `@WithUserDetails`、`@WithSecurityContext` 体系。

## 1.2 典型作用

- 构造认证用户（匿名 → 已认证主体的切换）
- 模拟 CSRF 令牌（写操作请求默认需要）
- 模拟 OAuth2/JWT 认证结果
- 验证方法级权限（`@PreAuthorize` 是否按预期放行/拒绝）

**认证主体注入的层次**决定了工具的选择：

```text
请求 → SecurityContextPersistenceFilter → AuthorizationFilter → Controller → @PreAuthorize
         ▲                                ▲                                ▲
         │                                │                                │
    with(user(...))               角色/权限在这里被检查             方法级安全独立检查
    @WithMockUser 直接填充 SecurityContext（绕过过滤器链的身份来源）
```

> **注意**：`@WithMockUser` 是**直接设置 SecurityContextHolder**，它不经过你配置的真实认证流程（不查库、不验 JWT 签名）。所以它测的是"授权规则"，测不到"认证流程本身"。要测登录端点、JWT 解析、OAuth2 code 交换，得走真实认证路径或对应的专用工具（4.x 节的 `jwt()`/`oidcLogin()` 也是模拟认证**结果**，不是模拟签名校验过程）。

---

# 2. MockMvc 与安全

## 2.1 `@WithMockUser`

最常用的注解，在测试方法执行前把一个 Mock 用户放进 `SecurityContext`：

```java
@WithMockUser(username = "admin", roles = "ADMIN")   // roles 会自动加 ROLE_ 前缀
@Test
void adminOnlyApi_shouldPass() throws Exception {
    mockMvc.perform(get("/admin"))
        .andExpect(status().isOk());
}
```

参数细节：

```java
@WithMockUser(
    username = "alice",
    roles = {"USER", "AUDITOR"},                 // 等价于权限 ROLE_USER、ROLE_AUDITOR
    authorities = "order:write",                 // 与 roles 互斥；直接给 GrantedAuthority
    password = "ignored"                         // 认证测试里几乎不用管
)
```

`roles` 与 `authorities` 的区别是高频坑：`roles = "ADMIN"` 生成的是 `ROLE_ADMIN`；如果你的授权规则写 `hasAuthority('ADMIN')`（无前缀），`@WithMockUser(roles="ADMIN")` 会 403——这时要用 `authorities = "ADMIN"`。默认值 `@WithMockUser`（不带参）是用户名 `user`、角色 `ROLE_USER`、已认证。

## 2.2 请求后处理器

注解作用于"测试方法级"，`RequestPostProcessor` 作用于"单个请求级"，粒度更细，可在一个测试里发多个不同身份的请求：

```java
mockMvc.perform(post("/orders")
        .with(user("u1").roles("USER"))     // 本请求以 u1 身份
        .with(csrf())                        // 附带合法 CSRF 令牌
)
.andExpect(status().isOk());

// 同一测试里换身份再发一次
mockMvc.perform(get("/orders/mine").with(user("u2").roles("USER")))
       .andExpect(status().isForbidden());
```

`user()` 的完整能力：

```java
with(user("bob").password("x").roles("USER", "ADMIN").accountExpired(true))
with(user("bob").authorities(new SimpleGrantedAuthority("order:read")))
```

`csrf()` 生成与当前请求匹配的令牌，替代手写 `X-CSRF-TOKEN` 头：

```java
with(csrf())                                  // 默认 POST 用 header 传递
with(csrf().asHeader())                       // 显式放 header
with(csrf().useInvalidToken())                // 故意用非法令牌，断言 403
```

> **设计争议**：CSRF 到底要不要在测试里关掉？很多团队图省事在测试 profile 里 `http.csrf().disable()`——但这等于放弃了"生产开着 CSRF 时控制器能否正常工作"的验证。更稳的做法是**测试环境保留 CSRF，用 `with(csrf())` 显式带令牌**：既测到了真实配置，又让"漏带 CSRF 会 403"这条规则也被覆盖。

---

# 3. 方法级安全测试

## 3.1 `@WithUserDetails`

`@WithMockUser` 的用户是凭空捏造的；`@WithUserDetails` 则**从真实的 `UserDetailsService` 加载用户**：

```java
@WithUserDetails("tom")          // 调用容器里的 UserDetailsService.loadUserByUsername("tom")
@Test
void methodSecurityTest() {
    // tom 的真实角色/权限来自数据库或内存用户配置
}
```

可指定用哪个 service（多 UserDetailsService 时）：

```java
@WithUserDetails(value = "tom", userDetailsServiceBeanName = "jdbcUserDetailsService")
```

适合验证**基于数据库用户加载的权限**——比如"从库里读出的角色列表能否正确映射到 `hasRole`"。代价是它依赖真实的 `UserDetailsService` 和数据，切片测试里若该 Bean 没加载会启动失败，此时要么 `@Import` 进来，要么改用 `@WithMockUser`。

## 3.2 自定义 SecurityContext

当身份模型超出"用户名 + 角色"（比如带租户 ID、带自定义 claim 的主体），用 `@WithSecurityContext` + 工厂：

```java
@WithSecurityContext(factory = TenantUserSecurityContextFactory.class)
@Test
void customContextTest() {
    // SecurityContext 里是工厂造出来的自定义 Authentication
}
```

工厂实现 `WithSecurityContextFactory<A extends Annotation>`：

```java
public class TenantUserSecurityContextFactory
        implements WithSecurityContextFactory<WithTenantUser> {

    @Override
    public SecurityContext createSecurityContext(WithTenantUser annotation) {
        TenantUser principal = new TenantUser(annotation.username(), annotation.tenantId());
        Authentication auth = UsernamePasswordAuthenticationToken.authenticated(
                principal, null,
                List.of(new SimpleGrantedAuthority("ROLE_USER")));
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(auth);            // 放入自定义主体
        return context;
    }
}

@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = TenantUserSecurityContextFactory.class)
public @interface WithTenantUser {
    String username() default "tester";
    String tenantId() default "t-1";                // 自定义维度随注解参数传入
}
```

这套机制的本质：**任何测试身份都可以表达为一个 `SecurityContext`**，`@WithMockUser`/`@WithUserDetails` 只是官方预置的两个工厂。多租户、RBAC 扩展字段、自定义 claim 的场景一律走自定义工厂，比在每个测试里手动 `SecurityContextHolder.setContext(...)` 干净得多。

## 3.3 方法级安全的独立测试

`@PreAuthorize` 可以在不启 Web 层的情况下测——用 Spring Security 的方法安全切片或直接 `@SpringBootTest` 调 Service：

```java
@Service
class InvoiceService {
    @PreAuthorize("hasRole('FINANCE')")
    public void approve(Long id) { }
}

@SpringBootTest
@Import(InvoiceService.class)
@EnableMethodSecurity                      // 确保方法级安全被激活
class InvoiceServiceSecurityTest {

    @Autowired
    private InvoiceService invoiceService;

    @Test
    @WithMockUser(roles = "USER")          // 权限不足
    void should_deny_for_non_finance() {
        assertThatThrownBy(() -> invoiceService.approve(1L))
                .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(roles = "FINANCE")       // 权限足够
    void should_allow_for_finance() {
        assertThatCode(() -> invoiceService.approve(1L)).doesNotThrowAnyException();
    }
}
```

> **注意**：方法级安全测试的前提是 `@EnableMethodSecurity`（Security 6.x）生效。切片测试若不加载安全自动配置，`@PreAuthorize` 会被静默忽略——**测试全绿但其实没鉴权**。断言"应该被拒绝"的用例（`AccessDeniedException`）是防这个假绿的护栏：如果注解没生效，拒绝用例反而会失败暴露问题。

---

# 4. OAuth2 / JWT 测试

## 4.1 JWT（资源服务器）

被保护方是 Resource Server 时，用 `jwt()` 后处理器伪造一个已认证的 JWT：

```java
mockMvc.perform(get("/api")
        .with(jwt().jwt(jwt -> jwt
                .claim("scope", "read")             // 自定义 claim
                .subject("user-123")
                .header("alg", "none")))            // 不验签，纯模拟认证结果
)
.andExpect(status().isOk());
```

配合权限断言：

```java
with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_read")))
```

`jwt()` 默认造一个 subject 为 "user"、带 `SCOPE_read` 的 token。它模拟的是"JWT 已校验通过后的 `JwtAuthenticationToken`"，**不测签名验证逻辑**——签名/issuer 校验属于认证流程测试，需要 `@SpringBootTest(webEnvironment = RANDOM_PORT)` 走真实 `SecurityFilterChain` + 真实公钥。

## 4.2 OIDC 登录

客户端（走授权码登录）场景用 `oidcLogin()`：

```java
mockMvc.perform(get("/profile").with(oidcLogin()))
    .andExpect(status().isOk());

// 定制 OIDC 用户
with(oidcLogin().idToken(t -> t.subject("google-1").claim("email", "a@b.com")))
with(oidcLogin().userInfoToken(u -> u.claim("name", "Alice")))
```

## 4.3 客户端凭证与令牌中继

调用下游需要 access token 时，`@WithMockUser` 造不出 OAuth2 token。用 `oauth2Client()` / `oauth2Login()`：

```java
with(oauth2Login().accessToken(new OAuth2AccessToken(BEARER, "tok", null, null)))
with(oauth2Client("my-client"))        // 注册一个 OAuth2AuthorizedClient，供 token 中继使用
```

选型速查：

| 场景 | 工具 |
|------|------|
| 资源服务器校验 JWT | `with(jwt())` |
| OIDC 授权码登录 | `with(oidcLogin())` |
| 需要真实 access token（中继） | `with(oauth2Client())` / `oauth2Login()` |
| 简单用户名密码角色 | `@WithMockUser` / `with(user())` |
| 从库加载真实用户 | `@WithUserDetails` |

---

# 5. 切片里安全配置的装载

安全测试最容易翻车的地方是**切片默认不加载完整安全配置**：

- `@WebMvcTest` **会**加载 `SecurityFilterChain`、方法安全等 Web 安全相关 Bean（它 imports 了 `SecurityAutoConfiguration`/`UserDetailsServiceAutoConfiguration` 等），所以控制器层的授权规则、CSRF 都能测到；
- 但**自定义的、非 Web 层的安全配置类**（比如你手写的 `@Configuration` 里的密码编码器、自定义 `AuthenticationProvider`）不一定被 `@WebMvcTest` 扫描到，需要 `@Import(MySecurityConfig.class)`；
- `@DataJpaTest`/`@JsonTest` 等非 Web 切片**完全不加载安全过滤器链**，`@WithUserDetails` 依赖的 `UserDetailsService` 也不在——在这些切片里测方法安全要显式 `@EnableMethodSecurity` + `@Import`。

```java
@WebMvcTest(OrderController.class)
@Import(SecurityConfig.class)              // 把项目自定义安全配置显式带进切片
@WithMockUser(roles = "USER")
class OrderControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;     // 切片外依赖照例 Mock

    @Test
    void user_cannot_delete() throws Exception {
        mockMvc.perform(delete("/orders/1").with(csrf()))
               .andExpect(status().isForbidden());   // 删除需要 ADMIN
    }
}
```

装载检查清单：

| 想测 | 用什么切片 | 需要额外装载 |
|------|-----------|--------------|
| URL 授权规则、CSRF | `@WebMvcTest` | 自定义 SecurityConfig（`@Import`） |
| 方法级 `@PreAuthorize` | `@SpringBootTest` 或切片 + `@EnableMethodSecurity` | Service Bean |
| JWT 签名/issuer 校验 | `@SpringBootTest(RANDOM_PORT)` | 完整链 + 公钥配置 |
| 登录表单/端点 | `@SpringBootTest(RANDOM_PORT)` 或 `@AutoConfigureMockMvc` | 完整 SecurityFilterChain |

> **行业实践**：安全测试分两层写——**切片层（`@WebMvcTest` + `@WithMockUser`/`jwt()`）覆盖每条 URL 和每个 `@PreAuthorize` 的放行/拒绝矩阵**，快且能穷举角色组合；**端到端层（少量 `RANDOM_PORT`）验证真实登录、真实 JWT 校验、真实重定向**。只做前者会漏掉"认证流程本身坏了"，只做后者则角色组合覆盖不全、跑得慢。

---

# 6. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 写操作 403 | 缺少 CSRF 令牌 | `with(csrf())`，或确认测试环境未误关 CSRF |
| 401 | 未注入认证主体 | `@WithMockUser` 或 `with(user())`；匿名请求默认无身份 |
| 方法级 403 但预期放行 | `roles` 与 `authorities` 前缀不匹配 | `roles="ADMIN"` → `ROLE_ADMIN`；`hasAuthority('ADMIN')` 要用 `authorities` |
| `@PreAuthorize` 完全没拦截 | 方法安全未激活 | 确认 `@EnableMethodSecurity`；切片里需显式开启 |
| `@WithUserDetails` 启动失败 | 切片未加载 `UserDetailsService` | `@Import` 配置或改用 `@WithMockUser` |
| JWT 测试没测到验签 | `jwt()` 只模拟认证结果 | 验签走 `RANDOM_PORT` + 真实公钥的端到端测试 |
| 登录后不重定向 | MockMvc 不走完整 servlet 转发链 | 用 `RANDOM_PORT` + `TestRestTemplate` 跟随重定向 |

# 7. 小结

| 主题 | 关键结论 |
|------|---------|
| 依赖 | `spring-security-test` 需显式引入，不在 starter-test 内 |
| 身份注入 | `@WithMockUser`（捏造）/ `@WithUserDetails`（查库）/ `@WithSecurityContext`（自定义工厂）/ `with(user())`（请求级） |
| CSRF | 测试环境保留 CSRF，用 `with(csrf())` 显式带令牌 |
| 方法级安全 | 需 `@EnableMethodSecurity` 生效；断言 `AccessDeniedException` 防假绿 |
| OAuth2/JWT | `jwt()`/`oidcLogin()`/`oauth2Client()` 模拟认证结果，不测验签流程 |
| 切片装载 | `@WebMvcTest` 自动带 Web 安全；自定义配置与 Service 需 `@Import` |
| 分层 | 切片穷举授权矩阵 + 少量端到端验真实认证流程 |

下一篇进入异步、事件与调度：`@Async`、`ApplicationEvent`、`@Scheduled` 在测试里如何同步化、捕获与手动触发。
