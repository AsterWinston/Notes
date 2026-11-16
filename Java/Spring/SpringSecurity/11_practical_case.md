# 1. 表单登录 + 权限控制（案例）

## 1.1 场景描述
传统 MVC 项目，用户通过浏览器表单登录，不同角色看到不同页面。

## 1.2 完整代码

**登录页 (login.html)：**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>用户管理系统 - 登录</title>
    <style>
        body { font-family: Arial; display: flex; justify-content: center; 
               align-items: center; height: 100vh; margin: 0; 
               background: #f5f5f5; }
        .login-box { background: white; padding: 40px; border-radius: 8px; 
                     box-shadow: 0 2px 10px rgba(0,0,0,0.1); width: 360px; }
        input { width: 100%; padding: 10px; margin: 8px 0; border: 1px solid #ddd; 
                border-radius: 4px; box-sizing: border-box; }
        button { width: 100%; padding: 12px; background: #1890ff; color: white; 
                 border: none; border-radius: 4px; cursor: pointer; }
        .error { color: red; margin-bottom: 10px; }
    </style>
</head>
<body>
<div class="login-box">
    <h2>用户管理系统</h2>
    <div class="error" th:if="${param.error}">用户名或密码错误</div>
    <div th:if="${param.logout}" style="color:green;margin-bottom:10px;">
        已成功退出登录
    </div>
    <form th:action="@{/login}" method="post">
        <input type="text" name="username" placeholder="用户名" required>
        <input type="password" name="password" placeholder="密码" required>
        <button type="submit">登 录</button>
    </form>
</div>
</body>
</html>
```

**SecurityConfig.java：**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/css/**", "/js/**", "/images/**").permitAll()
                .requestMatchers("/login", "/register").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasAnyRole("ADMIN", "USER")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .loginProcessingUrl("/login")
                .defaultSuccessUrl("/index", true)
                .failureUrl("/login?error")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
                .permitAll()
            )
            .rememberMe(remember -> remember
                .key("uniqueAndSecret")
                .tokenValiditySeconds(604800)
            );

        return http.build();
    }
}
```

**Controller.java：**

```java
@Controller
public class PageController {

    @GetMapping("/")
    public String root() {
        return "redirect:/index";
    }

    @GetMapping("/index")
    public String index(Model model,
            @AuthenticationPrincipal UserDetails userDetails) {
        model.addAttribute("username", userDetails.getUsername());
        model.addAttribute("roles", userDetails.getAuthorities());
        return "index";
    }

    @GetMapping("/admin/dashboard")
    @PreAuthorize("hasRole('ADMIN')")
    public String adminDashboard(Model model) {
        model.addAttribute("userCount", userService.count());
        model.addAttribute("activeSessions", sessionRegistry.getAllSessions(...));
        return "admin/dashboard";
    }

    @GetMapping("/user/profile")
    public String userProfile(Model model,
            @AuthenticationPrincipal UserDetails userDetails) {
        model.addAttribute("user", userService.getByUsername(userDetails.getUsername()));
        return "user/profile";
    }
}
```

---

# 2. REST API 安全（案例）

## 2.1 场景描述
前后端分离项目，前端 Vue/React，后端 Spring Boot + JWT 认证。

## 2.2 完整代码

**登录接口：**

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;
    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    @PostMapping("/login")
    public Result<LoginVO> login(@Valid @RequestBody LoginRequest request) {
        // 1. 执行认证
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword())
        );

        // 2. 生成 Token
        String accessToken = jwtTokenProvider.generateAccessToken(authentication);
        String refreshToken = jwtTokenProvider.generateRefreshToken(authentication);

        // 3. 获取用户信息
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();

        LoginVO vo = LoginVO.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .tokenType("Bearer")
            .expiresIn(jwtTokenProvider.getExpiration())
            .username(userDetails.getUsername())
            .authorities(userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .build();

        return Result.success(vo);
    }

    @PostMapping("/refresh")
    public Result<TokenVO> refresh(@RequestBody RefreshRequest request) {
        if (!jwtTokenProvider.validateToken(request.getRefreshToken())) {
            return Result.error(401, "refresh token 无效或已过期");
        }
        String newAccessToken = jwtTokenProvider.refreshAccessToken(
            request.getRefreshToken());
        return Result.success(new TokenVO(newAccessToken));
    }

    @PostMapping("/logout")
    @PreAuthorize("isAuthenticated()")
    public Result<Void> logout(
            @RequestHeader("Authorization") String token) {
        // 可选：Token加入黑名单
        tokenBlacklist.add(token.substring(7));
        return Result.success();
    }
}
```

**JWT 工具类（完整版）：**

```java
@Slf4j
@Component
public class JwtTokenProvider {

    private final SecretKey key;
    private final long accessTokenValidity;
    private final long refreshTokenValidity;

    public JwtTokenProvider(@Value("${jwt.secret}") String secret,
            @Value("${jwt.access-token-validity}") long accessValidity,
            @Value("${jwt.refresh-token-validity}") long refreshValidity) {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
        this.accessTokenValidity = accessValidity;
        this.refreshTokenValidity = refreshValidity;
    }

    public String generateAccessToken(Authentication authentication) {
        return buildToken(authentication, accessTokenValidity);
    }

    public String generateRefreshToken(Authentication authentication) {
        return buildToken(authentication, refreshTokenValidity);
    }

    private String buildToken(Authentication authentication, long validity) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + validity);

        return Jwts.builder()
            .setSubject(authentication.getName())
            .claim("authorities", authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .setIssuedAt(now)
            .setExpiration(expiry)
            .signWith(key)
            .compact();
    }

    public Authentication getAuthentication(String token) {
        Claims claims = parseClaims(token);
        Collection<? extends GrantedAuthority> authorities =
            ((List<String>) claims.get("authorities")).stream()
                .map(SimpleGrantedAuthority::new).toList();
        return new UsernamePasswordAuthenticationToken(
            claims.getSubject(), null, authorities);
    }

    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (Exception e) {
            log.warn("JWT验证失败: {}", e.getMessage());
            return false;
        }
    }

    public String refreshAccessToken(String refreshToken) {
        Claims claims = parseClaims(refreshToken);
        Authentication auth = new UsernamePasswordAuthenticationToken(
            claims.getSubject(), null, List.of());
        Authentication refreshed = authenticationManager.authenticate(auth); // 重新授权
        return generateAccessToken(refreshed);
    }

    private Claims parseClaims(String token) {
        return Jwts.parserBuilder().setSigningKey(key).build()
            .parseClaimsJws(token).getBody();
    }
}
```

**统一返回结果类：**

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> {
    private int code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "操作成功", data);
    }

    public static <T> Result<T> error(int code, String message) {
        return new Result<>(code, message, null);
    }
}
```

**application.yml：**

```yaml
jwt:
  secret: dGhpcyBpcyBhIHNlY3JldCBrZXkgZm9yIGp3dCB0b2tlbiBnZW5lcmF0aW9u
  access-token-validity: 3600000      # 1小时
  refresh-token-validity: 604800000   # 7天
```

---

# 3. 多角色、多权限管理（案例）

## 3.1 角色继承

```java
@Bean
static RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy(
        "ROLE_SUPER_ADMIN > ROLE_ADMIN\n" +
        "ROLE_ADMIN > ROLE_MANAGER\n" +
        "ROLE_MANAGER > ROLE_EDITOR\n" +
        "ROLE_EDITOR > ROLE_USER"
    );
    return hierarchy;
}
```

## 3.2 方法级安全控制示例

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    // 列表：需要 user:read 权限
    @GetMapping
    @PreAuthorize("hasAuthority('user:read')")
    public Result<List<UserVO>> list(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size) {
        return Result.success(userService.listUsers(page, size));
    }

    // 创建：需要 user:create
    @PostMapping
    @PreAuthorize("hasAuthority('user:create')")
    public Result<UserVO> create(@Valid @RequestBody CreateUserRequest request) {
        return Result.success(userService.createUser(request));
    }

    // 更新：管理员 或 修改自己的信息
    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('user:update') or #id == authentication.principal.id")
    public Result<UserVO> update(@PathVariable Long id,
                                 @Valid @RequestBody UpdateUserRequest request) {
        return Result.success(userService.updateUser(id, request));
    }

    // 删除：需要 user:delete 且不能删除自己
    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('user:delete') and #id != authentication.principal.id")
    public Result<Void> delete(@PathVariable Long id) {
        userService.deleteUser(id);
        return Result.success(null);
    }
}
```

---

# 4. 前后端分离安全（案例）

## 4.1 完整 SecurityConfig（JWT + CORS）

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final CustomAuthenticationEntryPoint authEntryPoint;
    private final CustomAccessDeniedHandler accessDeniedHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // 2. 无状态，关 CSRF
            .csrf(AbstractHttpConfigurer::disable)

            // 3. 无状态 Session
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // 4. URL 权限
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/api/auth/login",
                    "/api/auth/register",
                    "/api/auth/refresh"
                ).permitAll()
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // 5. JWT 过滤器
            .addFilterBefore(jwtAuthFilter, 
                UsernamePasswordAuthenticationFilter.class)

            // 6. 异常处理
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            );

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:3000", "http://localhost:5173"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

## 4.2 前端传递 Token

```javascript
// 前端 Axios 拦截器示例（Vue 3）
import axios from 'axios'

const api = axios.create({
    baseURL: 'http://localhost:8080/api',
    timeout: 10000
})

// 请求拦截：自动携带 Token
api.interceptors.request.use(config => {
    const token = localStorage.getItem('accessToken')
    if (token) {
        config.headers.Authorization = `Bearer ${token}`
    }
    return config
})

// 响应拦截：处理 Token 过期
api.interceptors.response.use(
    response => response,
    async error => {
        const originalRequest = error.config

        // 401 且不是登录/刷新请求 → 尝试刷新 Token
        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true

            try {
                const refreshToken = localStorage.getItem('refreshToken')
                const res = await axios.post('/api/auth/refresh', { refreshToken })

                localStorage.setItem('accessToken', res.data.data.accessToken)
                originalRequest.headers.Authorization = 
                    `Bearer ${res.data.data.accessToken}`

                return api(originalRequest)  // 重试原请求
            } catch {
                // 刷新失败 → 跳转登录
                localStorage.clear()
                window.location.href = '/login'
            }
        }

        return Promise.reject(error)
    }
)
```

## 4.3 防止 CSRF / XSS 攻击

**后端安全响应头配置：**

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())                    // 防止点击劫持
    .xssProtection(xss -> xss.disable())                    // XSS 过滤（浏览器已废弃）
    .contentSecurityPolicy(csp -> csp                       // CSP
        .policyDirectives(
            "default-src 'self'; " +
            "script-src 'self'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data:;"
        ))
    .httpStrictTransportSecurity(hsts -> hsts               // HSTS
        .includeSubDomains(true).maxAgeInSeconds(31536000))
);

// JWT 存储在 httpOnly Cookie（比 localStorage 更安全）
// 但注意：httpOnly Cookie 不能通过 JS 访问，需要后端配合设置
ResponseCookie cookie = ResponseCookie.from("token", jwt)
    .httpOnly(true).secure(true).sameSite("Strict")
    .path("/").maxAge(86400).build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```
