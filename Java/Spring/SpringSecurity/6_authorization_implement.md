# 1. 权限系统的作用

## 1.1 控制 URL 访问

不同 URL 路径需要不同权限：

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")    // 仅管理员
    .requestMatchers("/user/**").hasAnyRole("ADMIN", "USER") // 管理员和普通用户
    .requestMatchers("/api/public/**").permitAll()     // 公开访问
    .anyRequest().authenticated()                      // 其余需登录
);
```

## 1.2 控制方法访问

在 Service 或 Controller 层做细粒度控制：

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

@PreAuthorize("hasAuthority('order:create')")
public Order createOrder(Order order) { ... }
```

## 1.3 支持 RBAC 模型

```
用户(User) ───多对多─── 角色(Role) ───多对多─── 权限(Permission)
                                                      │
                      ┌───────────────────────────────┘
                      ├─ user:read
                      ├─ user:create
                      ├─ order:read
                      └─ order:delete
```

---

# 2. 授权基本流程

## 2.1 用户认证通过

```java
Authentication authentication = SecurityContextHolder.getContext()
    .getAuthentication();
// authentication 包含用户名和权限列表
```

## 2.2 SecurityContextHolder 获取权限

```java
// 获取当前用户的所有权限
Collection<? extends GrantedAuthority> authorities =
    authentication.getAuthorities();

// 检查是否有某权限
boolean hasAdminRole = authorities.stream()
    .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
```

## 2.3 AccessDecisionManager 判定

`AuthorizationFilter`（6.x 新版）或 `FilterSecurityInterceptor`（旧版）调用 `AccessDecisionManager` 进行投票决策：

| 决策管理器 | 策略 |
|-----------|------|
| `AffirmativeBased` | 一票通过制（默认） |
| `ConsensusBased` | 多数通过制 |
| `UnanimousBased` | 全票通过制 |

```java
@Bean
public AccessDecisionManager accessDecisionManager() {
    List<AccessDecisionVoter<?>> voters = List.of(
        new WebExpressionVoter(),
        new RoleVoter(),
        new AuthenticatedVoter()
    );
    return new AffirmativeBased(voters); // 任一Voter通过即通过
}
```

---

# 3. 授权实现

## 3.1 限制访问资源所需权限

### 3.1.1 URL 级权限（requestMatchers）

**HttpSecurity 授权 API 速查表：**

| 方法 | 说明 |
|------|------|
| `permitAll()` | 所有人（含匿名）可访问 |
| `denyAll()` | 所有人拒绝 |
| `authenticated()` | 需登录即可（不限角色） |
| `fullyAuthenticated()` | 需完全认证（非记住我） |
| `rememberMe()` | 需记住我 |
| `anonymous()` | 仅匿名用户 |
| `hasRole("ADMIN")` | 需 ROLE_ADMIN |
| `hasAnyRole("ADMIN", "USER")` | 需 ROLE_ADMIN 或 ROLE_USER |
| `hasAuthority("user:delete")` | 需 user:delete 权限 |
| `hasAnyAuthority("user:read","user:write")` | 多权限之一 |
| `hasIpAddress("192.168.1.0/24")` | IP 限制 |

```java
http.authorizeHttpRequests(auth -> auth
    // 公共资源
    .requestMatchers("/", "/login", "/css/**", "/js/**").permitAll()

    // 需要特定角色
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/user/**").hasAnyRole("ADMIN", "USER")

    // 需要特定权限
    .requestMatchers(HttpMethod.GET, "/api/users/**")
        .hasAuthority("user:read")
    .requestMatchers(HttpMethod.POST, "/api/users/**")
        .hasAuthority("user:create")
    .requestMatchers(HttpMethod.DELETE, "/api/users/**")
        .hasAuthority("user:delete")

    // IP 限制
    .requestMatchers("/internal/**").hasIpAddress("10.0.0.0/8")

    .anyRequest().authenticated()
);
```

### 3.1.2 方法级权限

**启用配置：**

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,   // @PreAuthorize / @PostAuthorize
    securedEnabled = true,   // @Secured
    jsr250Enabled = true     // @RolesAllowed
)
public class MethodSecurityConfig {
}
```

**常用 SpEL 表达式：**

| 表达式 | 说明 |
|--------|------|
| `hasRole('ADMIN')` | 有 ADMIN 角色 |
| `hasAnyRole('ADMIN', 'USER')` | 有任一角色 |
| `hasAuthority('user:write')` | 有特定权限 |
| `hasAnyAuthority('user:read','user:write')` | 有多权限之一 |
| `isAuthenticated()` | 已认证 |
| `isAnonymous()` | 匿名 |
| `isFullyAuthenticated()` | 完全认证 |
| `permitAll` | 允许所有 |
| `denyAll` | 拒绝所有 |
| `#paramName` | 引用方法参数 |
| `#object.property` | 引用参数属性 |
| `returnObject` | 引用返回值（@PostAuthorize） |
| `authentication` | 当前认证对象 |

```java
@Service
public class UserService {

    // 角色控制
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> findAllUsers() {
        return userRepository.findAll();
    }

    // 权限控制
    @PreAuthorize("hasAuthority('user:delete')")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }

    // 结合参数：只能修改自己的信息
    @PreAuthorize("#user.id == authentication.principal.id")
    public void updateProfile(User user) {
        userRepository.update(user);
    }

    // 返回值过滤
    @PostAuthorize("returnObject.owner == authentication.name")
    public Order getOrder(Long orderId) {
        return orderRepository.findById(orderId);
    }

    // 过滤集合返回值
    @PostFilter("filterObject.owner == authentication.name")
    public List<Order> getUserOrders() {
        return orderRepository.findAll();
    }
}
```

## 3.2 封装权限信息

### 3.2.1 GrantedAuthority

权限的两种常见命名模式：

```java
// 模式一：角色（Spring Security 内置 ROLE_ 前缀）
new SimpleGrantedAuthority("ROLE_ADMIN")
// .hasRole("ADMIN")  → 实际检查 "ROLE_ADMIN"

// 模式二：细粒度权限（无前缀）
new SimpleGrantedAuthority("user:read")
new SimpleGrantedAuthority("order:create")
// .hasAuthority("user:read") → 原样检查
```

### 3.2.2 Authentication 中存储权限

```java
UsernamePasswordAuthenticationToken auth =
    new UsernamePasswordAuthenticationToken(
        userDetails,    // principal
        null,           // credentials (认证后为 null)
        authorities     // 权限列表
    );
```

---

## 3.3 从数据库查询权限信息

### 3.3.1 RBAC 权限模型

```
┌──────────┐     ┌──────────────┐     ┌────────────────┐
│   User   │────→│  UserRole    │←────│     Role       │
│  id      │     │  user_id     │     │  id            │
│  username│     │  role_id     │     │  role_code     │
│  password│     └──────────────┘     │  role_name     │
└──────────┘                          └────────┬───────┘
                                                │
                                        ┌───────────────┐
                                        │ RolePermission│
                                        │  role_id      │
                                        │  permission_id│
                                        └───────┬───────┘
                                                │
                                        ┌───────┴───────┐
                                        │  Permission   │
                                        │  id           │
                                        │  code         │
                                        │  name         │
                                        └───────────────┘
```

### 3.3.2 准备工作

数据库建表 SQL 见 **[5_authentication_implement.md §3.2.1](#321-数据库表设计)**。

### 3.3.3 代码实现

**UserDetailsService 加载完整权限：**

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) {
        UserEntity user = userMapper.findWithRolesAndPermissions(username);

        Set<GrantedAuthority> authorities = new HashSet<>();

        // 添加角色
        user.getRoles().forEach(role -> {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getRoleCode()));

            // 添加角色下的权限
            role.getPermissions().forEach(perm ->
                authorities.add(new SimpleGrantedAuthority(perm.getPermissionCode()))
            );
        });

        return new User(user.getUsername(), user.getPassword(),
            user.getEnabled(), true, true, true, authorities);
    }
}
```

**MyBatis 联合查询（一次查出所有）：**

```xml
<resultMap id="UserWithRolesAndPermissions" type="UserEntity">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="password" column="password"/>
    <collection property="roles" ofType="RoleEntity">
        <id property="id" column="role_id"/>
        <result property="roleCode" column="role_code"/>
        <collection property="permissions" ofType="PermissionEntity">
            <id property="id" column="perm_id"/>
            <result property="permissionCode" column="perm_code"/>
        </collection>
    </collection>
</resultMap>

<select id="findWithRolesAndPermissions" resultMap="UserWithRolesAndPermissions">
    SELECT
        u.id, u.username, u.password, u.enabled,
        r.id AS role_id, r.role_code,
        p.id AS perm_id, p.permission_code AS perm_code
    FROM sys_user u
    LEFT JOIN sys_user_role ur ON u.id = ur.user_id
    LEFT JOIN sys_role r ON ur.role_id = r.id
    LEFT JOIN sys_role_permission rp ON r.id = rp.role_id
    LEFT JOIN sys_permission p ON rp.permission_id = p.id
    WHERE u.username = #{username}
</select>
```

### 3.3.4 角色继承配置

```java
@Bean
static RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy(
        "ROLE_ADMIN > ROLE_MANAGER\n" +
        "ROLE_MANAGER > ROLE_USER"
    );
    return hierarchy;
}
```

表示 ADMIN 自动拥有 MANAGER 和 USER 的所有权限。
