# 1. Spring Security 与 JWT 集成概述

Spring Security 是目前 Java 生态中最广泛使用的安全框架。其默认基于 Session 的认证方式需要调整，才能支持 JWT 无状态认证。

## 1.1 集成目标

- 替换默认的 Session 认证为 JWT 无状态认证
- 登录接口签发 JWT（Access Token + Refresh Token）
- 每次请求通过 Filter 验证 JWT
- 禁用 CSRF（JWT 本身不需要）
- 配置无状态 Session 管理

---

# 2. Spring Boot 3.x + Spring Security 6.x + JJWT 0.12.x 完整集成

## 2.1 依赖配置

```xml
<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Boot Starter Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    
    <!-- JJWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Lombok（可选） -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## 2.2 application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: jwt-security-demo

# JWT 配置
jwt:
  access-token:
    expiration: 900000        # 15 分钟
    private-key: classpath:keys/access-private.pem
    public-key: classpath:keys/access-public.pem
  refresh-token:
    expiration: 2592000000    # 30 天
    secret: ${JWT_REFRESH_SECRET:change-me-in-production}
```

## 2.3 密钥配置

### 2.3.1 生成 RSA 密钥对

```bash
# 生成 RSA 私钥（PKCS#8 格式）
openssl genpkey -algorithm RSA -out access-private.pem -pkeyopt rsa_keygen_bits:2048

# 提取公钥
openssl rsa -pubout -in access-private.pem -out access-public.pem
```

### 2.3.2 JWT 配置属性类

```java
@ConfigurationProperties(prefix = "jwt")
@Validated
public record JwtProperties(
    @NotNull AccessTokenProperties accessToken,
    @NotNull RefreshTokenProperties refreshToken
) {
    public record AccessTokenProperties(
        @Positive long expiration,
        @NotNull Resource privateKey,
        @NotNull Resource publicKey
    ) {}
    
    public record RefreshTokenProperties(
        @Positive long expiration,
        @NotEmpty String secret
    ) {}
}
```

### 2.3.3 JWT 工具类

```java
@Component
public class JwtTokenProvider {
    
    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;
    private final SecretKey refreshSecretKey;
    private final JwtProperties properties;
    private final JwtParser accessTokenParser;
    
    public JwtTokenProvider(JwtProperties properties) {
        this.properties = properties;
        this.privateKey = loadPrivateKey(properties.accessToken().privateKey());
        this.publicKey = loadPublicKey(properties.accessToken().publicKey());
        this.refreshSecretKey = Keys.hmacShaKeyFor(
            properties.refreshToken().secret().getBytes(StandardCharsets.UTF_8)
        );
        
        // 预构建 Parser（线程安全）
        this.accessTokenParser = Jwts.parser()
            .verifyWith(publicKey)
            .build();
    }
    
    public String generateAccessToken(UserDetails userDetails) {
        long now = System.currentTimeMillis();
        
        return Jwts.builder()
            .issuer("self")
            .subject(userDetails.getUsername())
            .issuedAt(new Date(now))
            .expiration(new Date(now + properties.accessToken().expiration()))
            .id(UUID.randomUUID().toString())
            .claim("authorities", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList())
            .signWith(privateKey)
            .compact();
    }
    
    public String generateRefreshToken(UserDetails userDetails) {
        long now = System.currentTimeMillis();
        
        return Jwts.builder()
            .issuer("self")
            .subject(userDetails.getUsername())
            .issuedAt(new Date(now))
            .expiration(new Date(now + properties.refreshToken().expiration()))
            .id(UUID.randomUUID().toString())
            .claim("type", "refresh")
            .signWith(refreshSecretKey)
            .compact();
    }
    
    public Authentication getAuthentication(String token) {
        Claims claims = accessTokenParser
            .parseSignedClaims(token)
            .getPayload();
        
        @SuppressWarnings("unchecked")
        List<String> authorities = claims.get("authorities", List.class);
        
        List<GrantedAuthority> grantedAuthorities = authorities.stream()
            .map(SimpleGrantedAuthority::new)
            .collect(Collectors.toList());
        
        UserDetails principal = new User(claims.getSubject(), "", grantedAuthorities);
        
        return new UsernamePasswordAuthenticationToken(principal, token, grantedAuthorities);
    }
    
    public boolean validateAccessToken(String token) {
        try {
            accessTokenParser.parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
    
    public Claims parseRefreshToken(String token) {
        return Jwts.parser()
            .verifyWith(refreshSecretKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
    
    private RSAPrivateKey loadPrivateKey(Resource resource) {
        try {
            String key = new String(resource.getInputStream().readAllBytes())
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replaceAll("\\s", "");
            byte[] keyBytes = Base64.getDecoder().decode(key);
            PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(keyBytes);
            KeyFactory kf = KeyFactory.getInstance("RSA");
            return (RSAPrivateKey) kf.generatePrivate(spec);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load private key", e);
        }
    }
    
    private RSAPublicKey loadPublicKey(Resource resource) {
        try {
            String key = new String(resource.getInputStream().readAllBytes())
                .replace("-----BEGIN PUBLIC KEY-----", "")
                .replace("-----END PUBLIC KEY-----", "")
                .replaceAll("\\s", "");
            byte[] keyBytes = Base64.getDecoder().decode(key);
            X509EncodedKeySpec spec = new X509EncodedKeySpec(keyBytes);
            KeyFactory kf = KeyFactory.getInstance("RSA");
            return (RSAPublicKey) kf.generatePublic(spec);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load public key", e);
        }
    }
}
```

## 2.4 JWT 认证过滤器

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";
    
    private final JwtTokenProvider jwtTokenProvider;
    
    public JwtAuthenticationFilter(JwtTokenProvider jwtTokenProvider) {
        this.jspTokenProvider = jwtTokenProvider;
    }
    
    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain)
            throws ServletException, IOException {
        
        String token = resolveToken(request);
        
        if (token != null && jwtTokenProvider.validateAccessToken(token)) {
            Authentication auth = jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
        if (bearerToken != null && bearerToken.startsWith(BEARER_PREFIX)) {
            return bearerToken.substring(BEARER_PREFIX.length());
        }
        return null;
    }
}
```

## 2.5 认证入口点（处理 401）

```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    
    private static final ObjectMapper mapper = new ObjectMapper();
    
    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException)
            throws IOException {
        
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("status", 401);
        body.put("error", "Unauthorized");
        body.put("message", authException.getMessage());
        body.put("path", request.getServletPath());
        body.put("timestamp", Instant.now().toString());
        
        mapper.writeValue(response.getOutputStream(), body);
    }
}
```

## 2.6 安全配置（SecurityConfig）

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final JwtAuthenticationEntryPoint jwtAuthEntryPoint;
    
    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter,
                          JwtAuthenticationEntryPoint jwtAuthEntryPoint) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.jwtAuthEntryPoint = jwtAuthEntryPoint;
    }
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 禁用 CSRF（JWT 不使用 Cookie，天然免疫 CSRF）
            .csrf(AbstractHttpConfigurer::disable)
            
            // 无状态会话管理
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            
            // 异常处理
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(jwtAuthEntryPoint)
            )
            
            // 路由权限配置
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            
            // JWT 过滤器（在 UsernamePasswordAuthenticationFilter 之前）
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

## 2.7 认证控制器

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;
    private final RefreshTokenService refreshTokenService;
    
    public AuthController(AuthenticationManager authenticationManager,
                          JwtTokenProvider jwtTokenProvider,
                          RefreshTokenService refreshTokenService) {
        this.authenticationManager = authenticationManager;
        this.jspTokenProvider = jwtTokenProvider;
        this.refreshTokenService = refreshTokenService;
    }
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest request) {
        // 1. 认证
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.username(), request.password()
            )
        );
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        
        // 2. 获取 UserDetails
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        
        // 3. 签发 Token
        String accessToken = jwtTokenProvider.generateAccessToken(userDetails);
        String refreshToken = jwtTokenProvider.generateRefreshToken(userDetails);
        
        // 4. 存储 Refresh Token（用于吊销和轮换）
        refreshTokenService.store(userDetails.getUsername(), refreshToken);
        
        return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(
            @RequestBody @Valid RefreshRequest request) {
        
        String newAccessToken = refreshTokenService.refreshAccessToken(
            request.refreshToken()
        );
        
        return ResponseEntity.ok(new AuthResponse(newAccessToken, null));
    }
    
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            @RequestHeader("Authorization") String authHeader) {
        
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            refreshTokenService.revokeToken(token);
        }
        
        SecurityContextHolder.clearContext();
        return ResponseEntity.noContent().build();
    }
}

// 请求/响应 DTO
record LoginRequest(@NotBlank String username, @NotBlank String password) {}
record RefreshRequest(@NotBlank String refreshToken) {}
record AuthResponse(String accessToken, String refreshToken) {}
```

## 2.8 UserDetailsService 实现

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    private final UserRepository userRepository;
    
    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        
        return userRepository.findByUsername(username)
            .map(this::toUserDetails)
            .orElseThrow(() -> 
                new UsernameNotFoundException("User not found: " + username));
    }
    
    private UserDetails toUserDetails(UserEntity user) {
        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());
        
        return new User(
            user.getUsername(),
            user.getPasswordHash(),
            user.isEnabled(),
            true,   // accountNonExpired
            true,   // credentialsNonExpired
            true,   // accountNonLocked
            authorities
        );
    }
}
```

---

# 3. Nimbus JOSE+JWT 与 Spring Security 集成（OAuth2 Resource Server 模式）

Spring Security 内置了 OAuth2 Resource Server 支持，底层使用 Nimbus JOSE+JWT：

## 3.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

## 3.2 配置

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # 或
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

## 3.3 最小 SecurityConfig

```java
@Configuration
@EnableWebSecurity
public class OAuth2ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            );
        
        return http.build();
    }
}
```

## 3.4 自定义 JWT 验证器

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .build();
    
    // 添加自定义验证规则
    OAuth2TokenValidator<Jwt> audienceValidator = new JwtClaimValidator<List<String>>(
        "aud", aud -> aud.contains("api.example.com")
    );
    
    OAuth2TokenValidator<Jwt> withIssuer = 
        JwtValidators.createDefaultWithIssuer("https://auth.example.com");
    
    OAuth2TokenValidator<Jwt> combined = 
        new DelegatingOAuth2TokenValidator<>(withIssuer, audienceValidator);
    
    decoder.setJwtValidator(combined);
    
    return decoder;
}
```

## 3.5 自定义 JWT 权限映射

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter = 
        new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthorityPrefix("ROLE_");
    authoritiesConverter.setAuthoritiesClaimName("roles");
    
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
    
    return converter;
}
```

---

# 4. 方法级权限控制

## 4.1 注解方式

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // 角色控制
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }
    
    // 权限控制
    @PreAuthorize("hasAuthority('user:write')")
    @PostMapping
    public User createUser(@RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
    
    // 资源所有权控制
    @PreAuthorize("hasRole('USER') and #userId == authentication.principal.username")
    @GetMapping("/{userId}")
    public User getUser(@PathVariable String userId) {
        return userService.findById(userId);
    }
    
    // 从 JWT Claims 中提取值进行判断
    @PreAuthorize("@authService.hasPermission(#userId, 'READ')")
    @GetMapping("/{userId}/profile")
    public UserProfile getProfile(@PathVariable String userId) {
        return userService.getProfile(userId);
    }
}
```

## 4.2 自定义权限评估

```java
@Component("authService")
public class AuthorizationService {
    
    public boolean hasPermission(String userId, String permission) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            Map<String, Object> claims = jwtAuth.getToken().getClaims();
            // 从 JWT Claims 中获取权限信息进行判断
            return hasRequiredPermission(claims, permission);
        }
        
        return false;
    }
}
```

---

# 5. Refresh Token 管理

## 5.1 Refresh Token 存储

```java
@Entity
@Table(name = "refresh_tokens")
public class RefreshTokenEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;
    
    @Column(nullable = false, unique = true)
    private String tokenHash;  // 存储哈希值而非原始 Token
    
    @Column(nullable = false)
    private String username;
    
    @Column(nullable = false)
    private Instant expiresAt;
    
    private boolean revoked;
    
    private long version;  // 用于家族吊销
    
    @Column(nullable = false)
    private Instant createdAt;
}
```

## 5.2 Refresh Token 服务

```java
@Service
@Transactional
public class RefreshTokenService {
    
    private final RefreshTokenRepository repository;
    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;
    
    public void store(String username, String refreshToken) {
        RefreshTokenEntity entity = new RefreshTokenEntity();
        entity.setTokenHash(hashToken(refreshToken));
        entity.setUsername(username);
        entity.setExpiresAt(
            Instant.now().plusMillis(jwtProperties.refreshToken().expiration())
        );
        entity.setRevoked(false);
        entity.setCreatedAt(Instant.now());
        
        repository.save(entity);
    }
    
    public String refreshAccessToken(String refreshToken) {
        String tokenHash = hashToken(refreshToken);
        RefreshTokenEntity stored = repository.findByTokenHash(tokenHash)
            .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));
        
        if (stored.isRevoked()) {
            throw new InvalidTokenException("Refresh token has been revoked");
        }
        
        if (stored.getExpiresAt().isBefore(Instant.now())) {
            throw new InvalidTokenException("Refresh token has expired");
        }
        
        // 生成新的 Access Token
        UserDetails userDetails = userDetailsService.loadUserByUsername(stored.getUsername());
        return jwtTokenProvider.generateAccessToken(userDetails);
    }
    
    public void revokeToken(String accessToken) {
        Claims claims = jwtTokenProvider.parseAccessToken(accessToken);
        String username = claims.getSubject();
        
        // 吊销该用户的所有 Refresh Token
        repository.revokeAllByUsername(username);
    }
    
    private String hashToken(String token) {
        return DigestUtils.sha256Hex(token);
    }
}
```

---

# 6. 生产环境配置清单

## 6.1 安全配置检查

```java
// SecurityConfig 中必须有的配置项：
□ .csrf(AbstractHttpConfigurer::disable)          // JWT 无需 CSRF
□ .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // 无状态
□ .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
□ AuthenticationEntryPoint（统一 401 响应格式）
□ AccessDeniedHandler（统一 403 响应格式）

// JwtTokenProvider 中必须有的检查：
□ 签名验证（parseSignedClaims 而非 parse）
□ exp 验证（JJWT 自动处理）
□ 算法白名单（仅接受预期算法）
□ 密钥不为 null（启动时检查）
```

## 6.2 CORS 配置

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("https://your-frontend-domain.com"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    configuration.setExposedHeaders(List.of("X-Request-Id"));
    configuration.setAllowCredentials(false);  // JWT 模式下不需要 credentials
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

---

# 7. 测试 JWT 认证

## 7.1 集成测试

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class AuthControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void shouldLoginAndAccessProtectedEndpoint() throws Exception {
        // 1. 登录获取 JWT
        String loginJson = """
            {"username": "user", "password": "password"}
        """;
        
        MvcResult loginResult = mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(loginJson))
            .andExpect(status().isOk())
            .andReturn();
        
        String accessToken = JsonPath.read(
            loginResult.getResponse().getContentAsString(),
            "$.accessToken"
        );
        
        // 2. 使用 JWT 访问受保护接口
        mockMvc.perform(get("/api/users")
                .header("Authorization", "Bearer " + accessToken))
            .andExpect(status().isOk());
    }
    
    @Test
    void shouldRejectExpiredToken() throws Exception {
        // 构造过期 JWT 或等待过期后测试
        // ...
    }
}
```

## 7.2 单元测试 JwtTokenProvider

```java
class JwtTokenProviderTest {
    
    private JwtTokenProvider provider;
    
    @BeforeEach
    void setUp() {
        KeyPair keyPair = Jwts.SIG.RS256.keyPair().build();
        // 注入测试密钥
    }
    
    @Test
    void shouldGenerateAndValidateToken() {
        UserDetails userDetails = new User("test", "",
            List.of(new SimpleGrantedAuthority("ROLE_USER")));
        
        String token = provider.generateAccessToken(userDetails);
        
        assertNotNull(token);
        assertTrue(provider.validateAccessToken(token));
    }
    
    @Test
    void shouldRejectTamperedToken() {
        String token = provider.generateAccessToken(mockUserDetails());
        // 篡改 token
        String tampered = token.substring(0, token.lastIndexOf('.'));
        
        assertFalse(provider.validateAccessToken(tampered));
    }
}
```
