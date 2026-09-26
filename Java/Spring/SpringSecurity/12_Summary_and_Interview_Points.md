# 1. 核心组件理解

## 速览表

| 组件 | 职责 | 关键接口/类 |
|------|------|------------|
| SecurityFilterChain | 过滤器链，安全入口 | `HttpSecurity.build()` |
| AuthenticationManager | 认证管理 | `ProviderManager` |
| AuthenticationProvider | 具体认证策略 | `DaoAuthenticationProvider` |
| UserDetailsService | 加载用户信息 | `loadUserByUsername()` |
| PasswordEncoder | 密码加密校验 | `BCryptPasswordEncoder` |
| SecurityContextHolder | 存储当前用户的认证信息 | ThreadLocal 存储 |
| GrantedAuthority | 用户权限 | `SimpleGrantedAuthority` |
| AccessDecisionManager | 授权投票决策 | `AffirmativeBased` |
| FilterSecurityInterceptor | 授权拦截器 | 旧版，6.x 仍可用 |

---

# 2. 过滤器链执行顺序

```
1.  DisableEncodeUrlFilter          — 禁止 URL JSessionID
2.  WebAsyncManagerIntegrationFilter — 异步请求传递 SecurityContext
3.  SecurityContextHolderFilter      — 加载/清空 SecurityContext
4.  HeaderWriterFilter               — 安全响应头
5.  CorsFilter                       — 跨域
6.  CsrfFilter                       — CSRF 防护
7.  LogoutFilter                     — 登出
8.  UsernamePasswordAuthenticationFilter — 表单登录
9.  BasicAuthenticationFilter        — HTTP Basic
10. RequestCacheAwareFilter          — 恢复请求
11. SecurityContextHolderAwareRequestFilter — 包装 Request
12. RememberMeAuthenticationFilter   — 记住我
13. AnonymousAuthenticationFilter    — 匿名用户
14. ExceptionTranslationFilter       — 异常转换
15. AuthorizationFilter              — 授权
```

**面试常见追问：**
- JWT 过滤器应该加在哪里？→ 第 8 步之前（`addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)`）
- 为什么 `ExceptionTranslationFilter` 在最后？→ 捕获前面的 `AuthenticationException` 和 `AccessDeniedException` 统一处理

---

# 3. 方法级安全与 URL 级安全区别

| 维度 | URL 级安全 | 方法级安全 |
|------|-----------|-----------|
| 配置位置 | `SecurityFilterChain` | `@PreAuthorize` / `@Secured` |
| 粒度 | 粗（URL 路径） | 细（方法/参数） |
| 实现方式 | Filter 层 | AOP 代理 |
| 性能 | 请求入口直接过滤 | 方法调用时拦截 |
| 适用场景 | 页面/接口级别隔离 | 业务逻辑细粒度控制 |
| 需启用的注解 | 无（默认生效） | `@EnableMethodSecurity` |

**最佳实践：两者结合**
- URL 级：大的路由分类（`/admin/**`, `/api/**`）
- 方法级：具体业务操作（增删改查权限）

---

# 4. Spring Boot 自动配置与手动配置差异

| 特性 | 自动配置 | 手动配置 |
|------|---------|---------|
| 登录页 | 默认生成 | 自定义 `/login.html` |
| 用户 | `user` + 随机密码 | 自定义 `UserDetailsService` |
| 密码编码 | `DelegatingPasswordEncoder` | 自定义 `PasswordEncoder` Bean |
| CSRF | 默认启用 | 可禁用 |
| Session | 默认管理 | `SessionCreationPolicy.STATELESS` |
| 自定义过滤器 | 无 | 手动 `addFilterBefore/After` |

**覆盖规则：定义一个同类型的 Bean 即可覆盖默认配置**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    // 自定义一条 FilterChain → 默认的会被替换
    return http.build();
}
```

---

# 5. 实战中常见安全漏洞与防护

| 漏洞 | 原理 | 防护 |
|------|------|------|
| **CSRF** | 利用用户已登录的Session伪造请求 | CSRF Token / 无状态JWT可禁用 |
| **Session Fixation** | 攻击者指定Session ID让受害者用 | 登录后换Session ID（默认启用） |
| **XSS** | 注入恶意脚本到页面 | CSP 响应头 + 输出HTML编码 |
| **Clickjacking** | iframe 嵌入诱导点击 | `X-Frame-Options: DENY` |
| **密码暴力破解** | 重复尝试不同密码 | 登录失败计数 + 账户锁定 + 验证码 |
| **SQL 注入** | 恶意SQL拼接 | MyBatis/Spring Data 参数化查询 |
| **未加密传输** | HTTP明文传输 | 强制 HTTPS（HSTS） |

---

# 6. 面试高频问题

## Q1: Spring Security 的工作原理？

```
1. DelegatingFilterProxy 将请求委托给 Spring Security 的 FilterChainProxy
2. FilterChainProxy 管理多个 SecurityFilterChain
3. 每个 SecurityFilterChain 包含一系列 Filter
4. 核心 Filter 包括：
   - 认证 Filter（UsernamePasswordAuthenticationFilter等）
   - 授权 Filter（AuthorizationFilter/FilterSecurityInterceptor）
   - 异常处理（ExceptionTranslationFilter）
5. 认证通过后 Authentication 存入 SecurityContextHolder
6. 后续请求可通过 SecurityContextHolder 获取用户信息
```

## Q2: AuthenticationManager、ProviderManager、AuthenticationProvider 的关系？

```
AuthenticationManager (顶层接口)
    └── ProviderManager (持有多个 Provider)
            ├── DaoAuthenticationProvider
            ├── JwtAuthenticationProvider
            └── Custom Provider

ProviderManager 遍历所有 Provider：
  - supports() → true 才尝试 authenticate()
  - 认证成功 → 返回已认证 Authentication
  - AccountStatusException → 直接抛出
  - BadCredentialsException → 继续下一个 Provider
  - 所有都不匹配 → 抛出最后一个异常
```

## Q3: JWT 和 Session 认证的区别？

| 维度 | Session | JWT |
|------|---------|-----|
| 状态 | 有状态（服务端存） | 无状态（客户端存） |
| 存储 | 服务端内存/Redis | 客户端 Cookie/LocalStorage |
| 扩展性 | 需 Session 共享 | 天然支持分布式 |
| 失效 | 服务端删除即可 | 需 Token 黑名单/短有效期 |
| 跨域 | 依赖 Cookie，有 CSRF 风险 | 无 CSRF 风险 |
| 大小 | 仅 Session ID | Token 包含用户信息 |
| 适用 | 传统 MVC | 前后端分离/微服务 |

## Q4: 如何实现"记住我"功能？

```
方案一：Hash 方式（简单，不推荐生产）
  Cookie = base64(username + ":" + expiry + ":" + MD5(username + expiry + password + key))
  
方案二：持久化 Token（推荐）
  - 每次登录生成新的 series 和 token
  - Cookie 存储 series:token
  - 每次自动登录更新 token
  - 可用于检测 Cookie 盗用（token 不匹配 → 清除所有记住我）
```

## Q5: 如何实现同账号只能登录一个？

```java
http.sessionManagement(session -> session
    .maximumSessions(1)
    .maxSessionsPreventsLogin(false)  // 踢掉旧会话
);
```

## Q6: `@PreAuthorize` 和 `@Secured` 有什么区别？

| 特性 | @PreAuthorize | @Secured |
|------|--------------|---------|
| SpEL 支持 | 支持 | 不支持 |
| 参数引用 | `#paramName` | 不支持 |
| 返回值引用 | `returnObject`（@PostAuthorize） | 不支持 |
| 权限前缀 | 需写完整 `hasRole('ADMIN')` | 数组 `@Secured({"ROLE_ADMIN"})` |
| 推荐度 | ★★★★★ | ★★☆☆☆ |

## Q7: Spring Security 6.x 相比 5.x 有哪些重要变更？

| 变更 | 5.x | 6.x |
|------|-----|-----|
| `antMatchers()` | 支持 | 废弃 → `requestMatchers()` |
| `authorizeRequests()` | 支持 | 废弃 → `authorizeHttpRequests()` |
| `@EnableGlobalMethodSecurity` | 启用方法安全 | 废弃 → `@EnableMethodSecurity` |
| JDK 基线 | JDK 8+ | JDK 17+ |
| 授权 Filter | FilterSecurityInterceptor | AuthorizationFilter（默认） |

---

# 7. 常见开发陷阱

1. **忘记禁用 CSRF**：JWT 无状态项目忘了 `http.csrf(AbstractHttpConfigurer::disable)`，导致 POST/PUT 返回 403
2. **`SecurityContextHolder` 跨线程丢失**：异步方法中获取不到用户 → 配置 `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)`
3. **`@PreAuthorize` 不生效**：没加 `@EnableMethodSecurity` 注解
4. **自定义 Filter 顺序不对**：JWT Filter 放在认证 Filter 后面，请求永远走不到
5. **密码编码不一致**：注册用 BCrypt，另一个地方用了 NoOp，导致永远登录失败
6. **`permitAll()` 仍然经过 Filter 链**：只是授权放行，认证 Filter 仍然会尝试解析 Token → 需要在认证 Filter 中跳过公开 URL
7. **`DelegatingPasswordEncoder` 前缀问题**：数据库密码没有 `{bcrypt}` 前缀，使用 `BCryptPasswordEncoder` 校验
