# 1. 登录检验流程

## 1.1 用户提交用户名/密码

前端发送 `POST /login`，参数为 `username`、`password`：

```javascript
// 前端 AJAX 提交（前后端分离）
fetch('/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({ username: 'admin', password: '123456' })
})
```

## 1.2 SecurityFilterChain 拦截请求

`UsernamePasswordAuthenticationFilter` 匹配 `POST /login`，提取参数构造 `UsernamePasswordAuthenticationToken`。

## 1.3 AuthenticationManager 验证用户

`ProviderManager` 依次尝试注册的 `AuthenticationProvider`，默认情况下 `DaoAuthenticationProvider` 接手，调用 `UserDetailsService.loadUserByUsername()` 加载用户，再用 `PasswordEncoder.matches()` 比对密码。

## 1.4 Authentication 对象存入 SecurityContextHolder

认证通过后，`SecurityContextPersistenceFilter`（6.x 为 `SecurityContextHolderFilter`）将 Authentication 存入 `SecurityContextHolder`，后续请求可直接获取当前用户信息：

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
```

---

# 2. 原理初探

## 2.1 Spring Security 完整流程

```
HTTP 请求
    │
    ▼
┌───────────────────────────────────────────────────┐
│  DelegatingFilterProxy                          │
│  (Servlet Filter 代理)                          │
└─────────────┬───────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────────────────┐
│  SecurityFilterChain                             │
│  ┌─────────────────────────────────────────────┐ │
│  │ SecurityContextHolderFilter                 │ │
│  │   (从 Repository 加载上下文)                │ │
│  ├─────────────────────────────────────────────┤ │
│  │ UsernamePasswordAuthenticationFilter       │ │
│  │   (解析参数 → 构造待认证 Token)             │ │
│  ├─────────────────────────────────────────────┤ │
│  │ BasicAuthenticationFilter                   │ │
│  ├─────────────────────────────────────────────┤ │
│  │ AuthorizationFilter                         │ │
│  │   (授权检查)                                │ │
│  └─────────────────────────────────────────────┘ │
└─────────────┬───────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────────────────┐
│  AuthenticationManager (ProviderManager)          │
│  ┌─────────────────────────────────────────────┐ │
│  │ DaoAuthenticationProvider                   │ │
│  │  ├─ UserDetailsService.loadUserByUsername() │ │
│  │  │   → SQL: SELECT * FROM user WHERE ...    │ │
│  │  ├─ PasswordEncoder.matches(raw, encoded)   │ │
│  │  │   → BCrypt.checkpw(...)                  │ │
│  │  └─ 返回已认证 Authentication              │ │
│  └─────────────────────────────────────────────┘ │
└─────────────┬───────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────────────────┐
│  SecurityContextHolder.setContext(context)        │
│  ThreadLocal存储 → 当前线程全局可访问              │
└───────────────────────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────────────────┐
│  Controller → @PreAuthorize → Service → Response  │
└───────────────────────────────────────────────────┘
```

## 2.2 认证流程详解

### 2.2.1 UsernamePasswordAuthenticationFilter

```java
// 继承关系
AbstractAuthenticationProcessingFilter
    └── UsernamePasswordAuthenticationFilter

// 核心方法 attemptAuthentication
@Override
public Authentication attemptAuthentication(HttpServletRequest request,
        HttpServletResponse response) throws AuthenticationException {
    if (postOnly && !request.getMethod().equals("POST")) {
        throw new AuthenticationServiceException("不支持" + request.getMethod());
    }
    String username = obtainUsername(request);  // request.getParameter("username")
    String password = obtainPassword(request);  // request.getParameter("password")

    UsernamePasswordAuthenticationToken authRequest =
        UsernamePasswordAuthenticationToken.unauthenticated(username, password);

    setDetails(request, authRequest); // 保存 IP、SessionId 等
    return this.getAuthenticationManager().authenticate(authRequest);
}
```

### 2.2.2 DaoAuthenticationProvider

```java
@Override
protected void additionalAuthenticationChecks(UserDetails userDetails,
        UsernamePasswordAuthenticationToken authentication) {
    if (authentication.getCredentials() == null) {
        throw new BadCredentialsException("密码不能为空");
    }
    String presentedPassword = authentication.getCredentials().toString();
    if (!this.passwordEncoder.matches(presentedPassword,
                                       userDetails.getPassword())) {
        throw new BadCredentialsException("密码错误");
    }
}

@Override
protected UserDetails retrieveUser(String username,
        UsernamePasswordAuthenticationToken authentication) {
    UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
    if (loadedUser == null) {
        throw new InternalAuthenticationServiceException(
            "UserDetailsService 返回 null");
    }
    return loadedUser;
}
```

### 2.2.3 UserDetailsService

```java
// 框架提供的实现
InMemoryUserDetailsManager          // 内存用户
JdbcUserDetailsManager              // JDBC 数据库
LdapUserDetailsManager              // LDAP
CachingUserDetailsService           // 缓存装饰器

// 自定义实现（实际项目必用）
@Service
public class CustomUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) {
        // 从数据库加载用户
        // → 构建 UserDetails 返回
    }
}
```

### 2.2.4 PasswordEncoder

```
BCryptPasswordEncoder
  encode("123456") → "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p..."
  matches("123456", "$2a$10$N9qo...") → true
```

BCrypt 的 `$2a$10$` 含义：
- `2a` = BCrypt 版本
- `10` = 强度因子（2^10 = 1024 轮哈希迭代）
- 后面是 22 字符 Salt + 31 字符 Hash

---

# 3. 解决问题（完整实现）

## 3.1 思路分析

### 3.1.1 核心目标
- 用户名 + 密码 验证身份
- 支持数据库用户表
- 密码需加密存储

### 3.1.2 扩展目标
- 返回 JWT Token（前后端分离）
- 自定义 REST 登录接口
- 退出登录 Token 失效
- 支持 `GET /api/user/info` 获取当前用户信息

## 3.2 准备工作

### 3.2.1 数据库表设计

```sql
-- 用户表
CREATE TABLE sys_user (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    username    VARCHAR(50)  NOT NULL UNIQUE,
    password    VARCHAR(200) NOT NULL,
    real_name   VARCHAR(50),
    phone       VARCHAR(20),
    enabled     TINYINT DEFAULT 1,      -- 1:启用 0:禁用
    create_time DATETIME DEFAULT NOW(),
    update_time DATETIME DEFAULT NOW()
);

-- 角色表
CREATE TABLE sys_role (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    role_code   VARCHAR(50) NOT NULL UNIQUE, -- ROLE_ADMIN, ROLE_USER
    role_name   VARCHAR(50) NOT NULL,
    description VARCHAR(200)
);

-- 用户-角色关联表
CREATE TABLE sys_user_role (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id)
);

-- 权限表（细粒度权限）
CREATE TABLE sys_permission (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    permission_code VARCHAR(50) NOT NULL UNIQUE, -- user:read, user:delete
    permission_name VARCHAR(50) NOT NULL,
    description     VARCHAR(200)
);

-- 角色-权限关联表
CREATE TABLE sys_role_permission (
    role_id       BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id)
);
```

### 3.2.2 创建实体类

```java
@Data
@TableName("sys_user")
public class UserEntity {
    private Long id;
    private String username;
    private String password;
    private String realName;
    private String phone;
    private Boolean enabled;
    private List<RoleEntity> roles;  // 关联角色
}

@Data
@TableName("sys_role")
public class RoleEntity {
    private Long id;
    private String roleCode;
    private String roleName;
    private List<PermissionEntity> permissions; // 关联权限
}

@Data
@TableName("sys_permission")
public class PermissionEntity {
    private Long id;
    private String permissionCode;
    private String permissionName;
}
```

## 3.3 实现

### 3.3.1 UserDetailsService 实现类

```java
@Service
public class UserDetailServiceImpl implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) 
            throws UsernameNotFoundException {
        // 1. 查询用户
        UserEntity user = userMapper.findByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在: " + username);
        }

        // 2. 查询角色和权限
        List<RoleEntity> roles = userMapper.findRolesByUserId(user.getId());
        List<GrantedAuthority> authorities = new ArrayList<>();

        for (RoleEntity role : roles) {
            // 添加角色
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getRoleCode()));
            // 添加权限
            List<PermissionEntity> permissions = 
                userMapper.findPermissionsByRoleId(role.getId());
            for (PermissionEntity perm : permissions) {
                authorities.add(new SimpleGrantedAuthority(perm.getPermissionCode()));
            }
        }

        // 3. 构建 UserDetails
        return new User(user.getUsername(),   // principal
                       user.getPassword(),    // 加密后的密码
                       user.getEnabled(),     // 是否启用
                       true,                  // accountNonExpired
                       true,                  // credentialsNonExpired
                       true,                  // accountNonLocked
                       authorities);
    }
}
```

**Mapper 接口（MyBatis-Plus 示例）：**

```java
@Mapper
public interface UserMapper extends BaseMapper<UserEntity> {

    @Select("SELECT * FROM sys_user WHERE username = #{username}")
    UserEntity findByUsername(@Param("username") String username);

    @Select("""
        SELECT r.* FROM sys_role r
        JOIN sys_user_role ur ON r.id = ur.role_id
        WHERE ur.user_id = #{userId}
    """)
    List<RoleEntity> findRolesByUserId(@Param("userId") Long userId);

    @Select("""
        SELECT p.* FROM sys_permission p
        JOIN sys_role_permission rp ON p.id = rp.permission_id
        WHERE rp.role_id = #{roleId}
    """)
    List<PermissionEntity> findPermissionsByRoleId(@Param("roleId") Long roleId);
}
```

### 3.3.2 密码加密存储

**配置 PasswordEncoder Bean：**

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

**注册/添加用户时加密：**

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;
    @Autowired
    private PasswordEncoder passwordEncoder;

    public void createUser(String username, String rawPassword) {
        UserEntity user = new UserEntity();
        user.setUsername(username);
        user.setPassword(passwordEncoder.encode(rawPassword)); // 加密存储
        user.setEnabled(true);
        userMapper.insert(user);
    }
}
```

### 3.3.3 登录接口

**方案一：使用默认表单登录 URL**

```java
// SecurityConfig 中保持默认设置即可
http.formLogin(form -> form
    .loginProcessingUrl("/login")    // 处理 POST /login
    .successHandler(customSuccessHandler)
    .failureHandler(customFailureHandler)
);
```

**方案二：自定义 REST 登录接口 + JWT**

```java
@RestController
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;
    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    @PostMapping("/api/auth/login")
    public Result<LoginVO> login(@RequestBody LoginRequest request) {
        // 1. 封装认证 Token
        UsernamePasswordAuthenticationToken authToken =
            UsernamePasswordAuthenticationToken.unauthenticated(
                request.getUsername(), request.getPassword());

        // 2. 执行认证
        Authentication authentication = authenticationManager.authenticate(authToken);

        // 3. 生成 JWT
        String jwt = jwtTokenProvider.generateToken(authentication);

        // 4. 返回
        return Result.success(new LoginVO(jwt));
    }
}
```

### 3.3.4 JWT 认证过滤器

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String token = resolveToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            Authentication auth = jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}

@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secretKey;
    @Value("${jwt.expiration}")
    private long validityInMs; // 3600000 = 1小时

    public String generateToken(Authentication authentication) {
        Date now = new Date();
        Date validity = new Date(now.getTime() + validityInMs);

        return Jwts.builder()
            .setSubject(authentication.getName())
            .claim("authorities", authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).collect(Collectors.toList()))
            .setIssuedAt(now)
            .setExpiration(validity)
            .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()))
            .compact();
    }

    public Authentication getAuthentication(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(secretKey.getBytes())
            .build()
            .parseClaimsJws(token)
            .getBody();

        String username = claims.getSubject();
        List<String> authorities = claims.get("authorities", List.class);
        List<GrantedAuthority> grantedAuthorities = authorities.stream()
            .map(SimpleGrantedAuthority::new)
            .collect(Collectors.toList());

        return new UsernamePasswordAuthenticationToken(username, null, grantedAuthorities);
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder().setSigningKey(secretKey.getBytes())
                .build().parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

**JWT 依赖（jjwt）：**

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

### 3.3.5 SecurityConfig 中注册 JWT 过滤器

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)       // JWT 无状态，关闭 CSRF
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // 无状态
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### 3.3.6 退出登录

```java
// SecurityConfig 中配置
http.logout(logout -> logout
    .logoutUrl("/api/auth/logout")
    .addLogoutHandler((request, response, auth) -> {
        // JWT 场景：将 Token 加入黑名单或清理客户端存储
        String token = request.getHeader("Authorization");
        if (token != null) {
            tokenBlacklistService.addBlacklist(token);
        }
    })
    .logoutSuccessHandler((request, response, auth) -> {
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":200,\"msg\":\"已退出\"}");
    })
);
```
