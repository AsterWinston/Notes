# 1. 定义与作用

Spring Security 是 Spring 生态中**基于过滤器链的安全框架**，核心解决两个问题：

- **身份认证 (Authentication)** — "你是谁"
- **权限授权 (Authorization)** — "你能做什么"

同时提供对常见 Web 攻击的**内置防护**（CSRF、Session Fixation、XSS 等），以及登录、注销、记住我等开箱即用的功能。

> 版本说明：本文档基于 **Spring Security 6.x + Spring Boot 3.x**，JDK 17+。

---

## 1.1 身份认证（Authentication）

验证用户身份的合法性。常见认证方式：

| 认证方式 | 说明 |
|----------|------|
| 表单登录 | 账号密码通过 HTML 表单提交 |
| HTTP Basic | 浏览器弹出框输入用户/密码，每次请求携带 `Authorization: Basic xxx` |
| JWT / Token | 无状态认证，前后端分离首选 |
| OAuth2 / OIDC | 第三方授权登录（GitHub、微信等） |
| LDAP | 企业目录认证 |
| 证书登录 | X.509 客户端证书认证 |

Spring Security 会将认证后的用户信息封装为 **`Authentication`** 对象存入 **`SecurityContextHolder`**。

---

## 1.2 权限授权（Authorization）

确定已认证用户能访问哪些资源。核心模型：

- **RBAC** (Role-Based Access Control)：用户 → 角色 → 权限
- **URL 级授权**：`.requestMatchers("/admin/**").hasRole("ADMIN")`
- **方法级授权**：`@PreAuthorize("hasAuthority('user:delete')")`

---

## 1.3 防止常见安全攻击

| 攻击类型 | 防护机制 |
|----------|----------|
| **CSRF** (跨站请求伪造) | CsrfFilter 生成随机 Token 校验 |
| **Session Fixation** | 登录成功后自动更换 Session ID |
| **XSS** | Content-Security-Policy 响应头 + 输出编码 |
| **Clickjacking** | `X-Frame-Options: DENY` |
| **Session 并发劫持** | `maximumSessions(1)` 控制会话数 |

---

## 1.4 集成登录、注销、记住我功能

Spring Security 提供开箱即用的：
- **表单登录**：默认 `/login` 页面和 `POST /login` 接口
- **注销**：`/logout`，可清理 Session 和 Cookie
- **Remember-Me**：基于 Token 或持久化 Token 实现

---

# 2. Spring Security 架构

## 2.1 核心组件

```
┌──────────────────────────────────────────────────────┐
│              SecurityFilterChain                      │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌───────────┐ │
│  │ Csrf    │ │ Auth    │ │ Header   │ │ Session   │ │
│  │ Filter  │→│ Filter  │→│ Writer   │→│ Mgmt      │→│...
│  └─────────┘ └─────────┘ └──────────┘ └───────────┘ │
└──────────────────────────────────────────────────────┘
                         ↓
          ┌──────────────────────────┐
          │   AuthenticationManager   │
          │  ┌────────────────────┐  │
          │  │  ProviderManager   │  │
          │  │  ┌──────────────┐  │  │
          │  │  │ DaoAuthProv  │  │  │
          │  │  └──────────────┘  │  │
          │  └────────────────────┘  │
          └──────────────────────────┘
                         ↓
          ┌──────────────────────────┐
          │   UserDetailsService      │
          └──────────────────────────┘
                         ↓
          ┌──────────────────────────┐
          │   SecurityContextHolder   │
          └──────────────────────────┘
```

### 2.1.1 SecurityFilterChain

一系列 Servlet Filter 组成的链，是 Spring Security 的核心入口。每个 HTTP 请求都会经过这条链。

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

### 2.1.2 AuthenticationManager

认证管理的顶层接口，通常由 `ProviderManager` 实现。内部持有多个 `AuthenticationProvider`，依次尝试认证。

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) 
        throws AuthenticationException;
}
```

### 2.1.3 UserDetailsService

加载用户信息的核心接口。`loadUserByUsername(String)` 返回 `UserDetails`。

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) 
        throws UsernameNotFoundException;
}
```

### 2.1.4 GrantedAuthority

表示用户拥有的权限，存储在 `Authentication` 中。格式通常为 `ROLE_XXX` 或 `xxx:yyy`。

```java
// 角色：ROLE_ADMIN, ROLE_USER
// 权限：user:read, user:write, order:delete
```

### 2.1.5 PasswordEncoder

密码加密接口，用于存储和验证密码。

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}
```

## 2.2 核心流程

```
用户请求 --> SecurityFilterChain --> 认证过滤器 --> AuthenticationManager
    --> AuthenticationProvider --> UserDetailsService (加载用户)
    --> PasswordEncoder (比对密码) --> Authentication对象
    --> SecurityContextHolder.setContext()
    --> FilterSecurityInterceptor (授权判断)
    --> 访问资源 / 拒绝访问
```

### 2.2.1 请求过滤 → 身份验证 → 授权 → 访问资源

1. **请求拦截**：`DelegatingFilterProxy` 将请求委托给 Spring Security 的 Filter Chain
2. **身份验证**：提取凭证，调用 `AuthenticationManager` 做认证
3. **认证成功后**：将 `Authentication` 存入 `SecurityContextHolder`
4. **授权判断**：`FilterSecurityInterceptor` 调用 `AccessDecisionManager` 投票
5. **访问资源**：通过则到达 Controller，拒绝则抛 `AccessDeniedException`

---

# 3. 发展历程

## 3.1 Spring Security 版本演变

| 版本 | 里程碑 | 说明 |
|------|--------|------|
| **3.x** | 2008 | 由 Acegi Security 更名而来 |
| **4.x** | 2015 | 全面拥抱 Java Config，废弃 XML |
| **5.x** | 2017 | 引入 OAuth2 支持，响应式安全（WebFlux） |
| **6.x** | 2022 | JDK 17+ 基线；废弃 `antMatchers()`，统一使用 `requestMatchers()` |

**Spring Boot 与 Spring Security 版本对应：**

| Spring Boot | Spring Security |
|-------------|-----------------|
| 2.7.x | 5.7.x |
| 3.0.x | 6.0.x |
| 3.2.x | 6.2.x |

## 3.2 配置方式演变

| 方式 | 特点 | 现状 |
|------|------|------|
| **XML 配置** | 早期方式，`<http>` `<authentication-manager>` | 已淘汰 |
| **Java Config** | `@Configuration` + `SecurityFilterChain` Bean | **当前主流** |
| **注解驱动** | `@EnableWebSecurity`、`@EnableMethodSecurity` | 配合 Java Config 使用 |
