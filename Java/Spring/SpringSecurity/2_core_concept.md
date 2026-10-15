# 1. 身份认证（Authentication）

## 1.1 Principal / UserDetails

- **Principal**：代表当前用户的主体标识，是 Java Security 中的通用概念
- **UserDetails**：Spring Security 扩展的 Principal，包含用户名、密码、权限集合、账户状态等信息

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities(); // 权限集合
    String getPassword();                                    // 密码
    String getUsername();                                    // 用户名
    boolean isAccountNonExpired();                           // 账户是否过期
    boolean isAccountNonLocked();                            // 是否锁定
    boolean isCredentialsNonExpired();                       // 凭证是否过期
    boolean isEnabled();                                     // 是否启用
}
```

`User` 是 `UserDetails` 的标准实现类：
```java
UserDetails user = User.builder()
    .username("admin")
    .password(passwordEncoder.encode("123456"))
    .roles("ADMIN", "USER")
    .build();
```

---

## 1.2 Authentication 对象

`Authentication` 是 Spring Security 中表示认证信息的核心接口，贯穿整个认证流程：

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities(); // 权限
    Object getCredentials();                // 密码/凭证（认证后清除）
    Object getDetails();                    // 额外信息（如IP、SessionId等）
    Object getPrincipal();                  // 主体（认证前是username，认证后是UserDetails）
    boolean isAuthenticated();              // 是否已认证
    void setAuthenticated(boolean isAuthenticated);
}
```

**认证过程中 `Authentication` 的状态变化：**

| 阶段 | principal | credentials | isAuthenticated |
|------|-----------|-------------|-----------------|
| 未认证 | username (String) | password | false |
| 已认证 | UserDetails | null | true |

**`SecurityContext`** 是持有 `Authentication` 的容器：

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);
```

**`SecurityContextHolder`** — 使用 `ThreadLocal` 存储当前线程的 `SecurityContext`：

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
```

三种存储策略（默认 `MODE_THREADLOCAL`）：
- `MODE_THREADLOCAL`：线程绑定的默认模式
- `MODE_INHERITABLETHREADLOCAL`：子线程可继承
- `MODE_GLOBAL`：全局模式，所有线程共享

---

## 1.3 ProviderManager 与 AuthenticationProvider

```
AuthenticationManager (接口)
    └── ProviderManager (实现)
            ├── DaoAuthenticationProvider
            ├── JwtAuthenticationProvider
            └── 自定义 AuthenticationProvider
```

- **AuthenticationManager** 是顶层认证入口，只有一个方法 `authenticate()`
- **ProviderManager** 持有 `List<AuthenticationProvider>`，**遍历**各 Provider 尝试认证
- 每个 Provider 通过 `supports()` 声明自己能处理的 `Authentication` 类型

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {
    @Override
    public Authentication authenticate(Authentication auth) 
            throws AuthenticationException {
        String username = auth.getName();
        String password = auth.getCredentials().toString();
        // 自定义验证逻辑
        List<GrantedAuthority> authorities = List.of(new SimpleGrantedAuthority("ROLE_USER"));
        return new UsernamePasswordAuthenticationToken(username, null, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

---

# 2. 权限授权（Authorization）

## 2.1 GrantedAuthority

表示用户拥有的权限，是一个简单接口：

```java
public interface GrantedAuthority extends Serializable {
    String getAuthority();  // 返回如 "ROLE_ADMIN" 或 "user:delete"
}
```

- **角色**以 `ROLE_` 前缀开头：`ROLE_ADMIN`、`ROLE_USER`
- **权限**没有固定前缀：`user:read`、`order:create`

`.hasRole("ADMIN")` 实际校验的是 `ROLE_ADMIN`（会自动加前缀），而 `.hasAuthority("ROLE_ADMIN")` 需要写完整前缀。

---

## 2.2 基于 URL 的授权

在 `SecurityFilterChain` 中配置，拦截 HTTP 请求级别：

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/", "/login", "/public/**").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/user/**").hasAnyRole("ADMIN", "USER")
    .requestMatchers("/api/**").hasAuthority("api:access")
    .anyRequest().authenticated()
);
```

**Spring Security 6.x 重要变更：** `antMatchers()` 已废弃，统一使用 `requestMatchers()`。

---

## 2.3 基于方法的授权

通过 AOP 实现，需要在配置类上加 `@EnableMethodSecurity`：

```java
@EnableMethodSecurity  // Spring Security 6.x 替代 @EnableGlobalMethodSecurity
@Configuration
public class SecurityConfig { ... }
```

### 2.3.1 常用注解

| 注解 | 作用 | 时机 |
|------|------|------|
| `@PreAuthorize` | 方法执行前验证 | 最常用 |
| `@PostAuthorize` | 方法执行后验证 | 可访问返回值 |
| `@PreFilter` | 方法执行前过滤参数集合 | 较少用 |
| `@PostFilter` | 方法执行后过滤返回值集合 | 可过滤数据 |
| `@Secured` | 基于角色的方法级安全 | 早期注解，仅支持 ROLE_ |

```java
@PreAuthorize("hasRole('ADMIN')")
public List<User> getAllUsers() { ... }

@PreAuthorize("hasAuthority('user:write') and #user.id == authentication.principal.id")
public void updateUser(User user) { ... }

@PostAuthorize("returnObject.username == authentication.name")
public User getUser(Long id) { ... }
```

---

# 3. 用户管理

## 3.1 内存用户（InMemoryUserDetailsManager）

仅适合开发和测试：

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails admin = User.builder()
        .username("admin")
        .password(passwordEncoder().encode("123456"))
        .roles("ADMIN", "USER")
        .build();

    UserDetails user = User.builder()
        .username("user")
        .password(passwordEncoder().encode("123456"))
        .roles("USER")
        .build();

    return new InMemoryUserDetailsManager(admin, user);
}
```

## 3.2 数据库用户（JdbcUserDetailsManager）

基于 Spring Security 内置表结构（`users`、`authorities`）：

```sql
CREATE TABLE users (
    username VARCHAR(50) PRIMARY KEY,
    password VARCHAR(500) NOT NULL,
    enabled  BOOLEAN NOT NULL
);

CREATE TABLE authorities (
    username  VARCHAR(50) NOT NULL,
    authority VARCHAR(50) NOT NULL
);
```

```java
@Bean
public UserDetailsService userDetailsService(DataSource dataSource) {
    JdbcUserDetailsManager manager = new JdbcUserDetailsManager(dataSource);
    // 可选：自定义查询 SQL
    manager.setUsersByUsernameQuery(
        "SELECT username, password, enabled FROM sys_user WHERE username = ?");
    manager.setAuthoritiesByUsernameQuery(
        "SELECT u.username, r.role_code FROM sys_user u " +
        "JOIN user_role ur ON u.id=ur.user_id " +
        "JOIN role r ON ur.role_id=r.id WHERE u.username = ?");
    return manager;
}
```

## 3.3 自定义 UserDetailsService

**实际项目最常用方式**：

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) 
            throws UsernameNotFoundException {
        UserEntity user = userMapper.findByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在: " + username);
        }
        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getCode()))
            .collect(Collectors.toList());

        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            user.getEnabled(),
            true, true, true,
            authorities
        );
    }
}
```

---

# 4. 密码处理

## 4.1 PasswordEncoder 接口

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);          // 加密
    boolean matches(CharSequence rawPassword,         // 比对
                    String encodedPassword);
    default boolean upgradeEncoding(String encoded) { // 是否需要升级编码
        return false;
    }
}
```

> **安全铁律：** 永远不要在代码中硬编码明文密码，永远不要使用 MD5/SHA-1。

## 4.2 常用实现

| 实现类 | 说明 | 推荐度 |
|--------|------|--------|
| **BCryptPasswordEncoder** | 基于 bcrypt 算法，最广泛使用 | ★★★★★ |
| **Pbkdf2PasswordEncoder** | 基于 PBKDF2 (NIST 标准) | ★★★★☆ |
| **SCryptPasswordEncoder** | 基于 scrypt (内存敏感) | ★★★★☆ |
| **Argon2PasswordEncoder** | 密码哈希竞赛优胜者 | ★★★★☆ |
| **NoOpPasswordEncoder** | 明文存储，仅测试用 | ☆☆☆☆☆ 禁止生产 |
| **DelegatingPasswordEncoder** | 支持多种编码混合验证 | ★★★★★ |

### 4.2.1 BCryptPasswordEncoder

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // strength: 4~31，默认10，越大越安全但越慢
    return new BCryptPasswordEncoder(10);
}
```

密码密文格式：`$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy`

### 4.2.2 DelegatingPasswordEncoder（推荐）

```java
@Bean
public PasswordEncoder passwordEncoder() {
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put("bcrypt", new BCryptPasswordEncoder());
    encoders.put("pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

密码存储格式：`{bcrypt}$2a$10$...`（带前缀标识编码算法），便于平滑升级编码方案。
