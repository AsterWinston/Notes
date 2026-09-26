
# 1. 网关在认证体系中的角色

```
客户端 → Gateway → 认证服务器 → 颁发 Token
    │                  │
    │  携带 Token      │
    ▼                  │
 Gateway ───→ 验证 Token ──→ 转发到下游服务
    │
    ▼
 下游服务（信任网关传递的用户信息）
```

**网关的职责**：

1. **Token 验证**：在入口层统一验证 JWT Token 的有效性
2. **Token 转发**：将认证后的 Token 或用户信息转发给下游服务
3. **权限校验**：按路由维度校验请求的权限范围
4. **Token 续期**：在 Token 即将过期时自动续期

# 2. TokenRelay 过滤器

## 2.1 TokenRelay 原理

`TokenRelayGatewayFilterFactory` 将当前请求中的 OAuth2 访问令牌转发到下游服务：

```java
public class TokenRelayGatewayFilterFactory
        extends AbstractGatewayFilterFactory<Object> {

    private final ServerOAuth2AuthorizedClientRepository authorizedClientRepository;

    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> exchange.getPrincipal()
            .defaultIfEmpty(new AnonymousAuthenticationToken("anonymous",
                "anonymous", AuthorityUtils.NO_AUTHORITIES))
            .filter(principal -> principal instanceof OAuth2AuthenticationToken)
            .cast(OAuth2AuthenticationToken.class)
            .flatMap(authentication ->
                authorizedClientRepository.loadAuthorizedClient(
                    authentication.getAuthorizedClientRegistrationId(),
                    authentication, exchange))
            .map(client -> client.getAccessToken().getTokenValue())
            .map(token -> {
                // 将 access_token 添加到请求头
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                    .build();
                return exchange.mutate().request(request).build();
            })
            .defaultIfEmpty(exchange)
            .flatMap(chain::filter);
    }
}
```

## 2.2 配置 TokenRelay

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: secured_service
          uri: lb://secured-service
          predicates:
            - Path=/api/secured/**
          filters:
            - TokenRelay=            # 自动转发 OAuth2 Token
```

# 3. 集成 Spring Security OAuth2 Resource Server

## 3.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

## 3.2 基础配置

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # JWT 签发者 URI（用于获取公钥以验签）
          issuer-uri: https://auth.example.com/auth/realms/gateway-realm
          # 直接指定 JWK Set URI（可选，优先级高于 issuer-uri）
          jwk-set-uri: https://auth.example.com/auth/realms/gateway-realm/protocol/openid-connect/certs
```

## 3.3 安全配置

```java
@Configuration
@EnableWebFluxSecurity
public class ResourceServerConfiguration {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(
            ServerHttpSecurity http) {

        return http
            // 授权配置
            .authorizeExchange(exchanges -> exchanges
                // 放行路径
                .pathMatchers("/actuator/**", "/login", "/register", "/public/**")
                    .permitAll()
                // 需要特定角色
                .pathMatchers("/api/admin/**")
                    .hasAuthority("SCOPE_admin")
                // 需要认证
                .anyExchange()
                    .authenticated()
            )
            // 配置 OAuth2 Resource Server
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtDecoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            // 禁用 CSRF（API 网关场景）
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .build();
    }

    @Bean
    public ReactiveJwtDecoder jwtDecoder() {
        return ReactiveJwtDecoders.fromIssuerLocation(
            "https://auth.example.com/auth/realms/gateway-realm");
    }

    @Bean
    public Converter<Jwt, ? extends Mono<? extends AbstractAuthenticationToken>>
            jwtAuthenticationConverter() {
        // 自定义 JWT 到认证信息的转换
        ReactiveJwtAuthenticationConverter converter =
            new ReactiveJwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(
            new ReactiveJwtGrantedAuthoritiesConverter());
        return converter;
    }
}
```

# 4. JWT 令牌校验

## 4.1 自定义 JWT 解码器

```java
@Component
public class CustomJwtDecoder {

    private final ReactiveJwtDecoder decoder;

    public CustomJwtDecoder() {
        // Nimbus JWT 解码器
        NimbusReactiveJwtDecoder decoder = new NimbusReactiveJwtDecoder(
            "https://auth.example.com/.well-known/jwks.json");

        // 配置 JWT 校验
        decoder.setJwtValidator(ReactiveJwtValidators.create(
            // 自定义校验器
            withCustomValidator()
        ));

        // 配置时钟偏移（允许 30 秒的时钟偏差）
        decoder.setClockSkew(Duration.ofSeconds(30));

        this.decoder = decoder;
    }

    public Mono<Jwt> decode(String token) {
        return decoder.decode(token)
            .onErrorMap(JwtException.class, e ->
                new ResponseStatusException(
                    HttpStatus.UNAUTHORIZED, "Invalid token", e));
    }

    private JwtValidator withCustomValidator() {
        return new JwtValidator() {
            @Override
            public void validate(Jwt jwt) throws JwtValidationException {
                // 自定义校验逻辑
                String issuer = jwt.getClaimAsString("iss");
                if (!"https://auth.example.com".equals(issuer)) {
                    throw new JwtValidationException("Invalid issuer",
                        List.of(new OAuth2Error("invalid_issuer", "Issuer mismatch", null)));
                }

                // 校验 audience
                List<String> audience = jwt.getAudience();
                if (!audience.contains("gateway-client")) {
                    throw new JwtValidationException("Invalid audience",
                        List.of(new OAuth2Error("invalid_audience", "Audience mismatch", null)));
                }
            }
        };
    }
}
```

## 4.2 在过滤器中手动验证 JWT

```java
@Component
@Order(-300)
public class JwtAuthGlobalFilter implements GlobalFilter {

    private final CustomJwtDecoder jwtDecoder;

    public JwtAuthGlobalFilter(CustomJwtDecoder jwtDecoder) {
        this.jwtDecoder = jwtDecoder;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);

        return jwtDecoder.decode(token)
            .flatMap(jwt -> {
                // 将 JWT 信息注入到下游请求头
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header("X-User-Id", jwt.getSubject())
                    .header("X-User-Name",
                        jwt.getClaimAsString("preferred_username"))
                    .header("X-User-Email",
                        jwt.getClaimAsString("email"))
                    .header("X-User-Roles",
                        String.join(",",
                            jwt.getClaimAsStringList("roles") != null
                                ? jwt.getClaimAsStringList("roles")
                                : List.of()))
                    .build();

                return chain.filter(exchange.mutate()
                    .request(request).build());
            })
            .onErrorResume(e -> unauthorized(exchange, "Invalid token: " + e.getMessage()));
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders()
            .setContentType(MediaType.APPLICATION_JSON);
        byte[] body = ("{\"code\":401,\"message\":\"" + message + "\"}").getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

# 5. RBAC 权限校验

## 5.1 按路由配置权限

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: admin_route
          uri: lb://admin-service
          predicates:
            - Path=/api/admin/**
          metadata:
            required-roles: admin
            required-scope: write

        - id: user_route
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          metadata:
            required-roles: user,admin
```

## 5.2 权限校验过滤器

```java
@Component
@Order(-200)
public class AuthorizationGlobalFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route == null) {
            return chain.filter(exchange);
        }

        // 获取路由所需的角色
        List<String> requiredRoles = getRequiredRoles(route);
        if (requiredRoles.isEmpty()) {
            // 路由无需特定角色
            return chain.filter(exchange);
        }

        // 获取用户角色
        List<String> userRoles = extractUserRoles(exchange);
        if (userRoles == null) {
            return forbidden(exchange, "No user roles found");
        }

        // 检查角色交集
        boolean hasRole = requiredRoles.stream()
            .anyMatch(userRoles::contains);
        if (!hasRole) {
            return forbidden(exchange,
                "Insufficient permissions. Required: " + requiredRoles);
        }

        return chain.filter(exchange);
    }

    private List<String> getRequiredRoles(Route route) {
        Object roles = route.getMetadata().get("required-roles");
        if (roles instanceof String) {
            return Arrays.asList(((String) roles).split(","));
        }
        return List.of();
    }

    private List<String> extractUserRoles(ServerWebExchange exchange) {
        // 从 JWT 验证时设置的请求头中提取
        String rolesHeader = exchange.getRequest().getHeaders()
            .getFirst("X-User-Roles");
        if (rolesHeader == null || rolesHeader.isEmpty()) {
            return null;
        }
        return Arrays.asList(rolesHeader.split(","));
    }

    private Mono<Void> forbidden(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        byte[] body = ("{\"code\":403,\"message\":\"" + message + "\"}").getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

# 6. 多认证方式支持

## 6.1 API Key 认证

```java
@Component
@Order(-400)
public class ApiKeyGlobalFilter implements GlobalFilter {

    private final Map<String, ApiKeyInfo> validApiKeys = Map.of(
        "key-123", new ApiKeyInfo("client-a", List.of("read")),
        "key-456", new ApiKeyInfo("client-b", List.of("read", "write"))
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 检查是否已经有 JWT 认证（JWT 优先级更高）
        if (exchange.getRequest().getHeaders().containsKey(HttpHeaders.AUTHORIZATION)
            && exchange.getRequest().getHeaders()
                .getFirst(HttpHeaders.AUTHORIZATION).startsWith("Bearer ")) {
            return chain.filter(exchange);
        }

        // 尝试 API Key 认证
        String apiKey = exchange.getRequest().getHeaders()
            .getFirst("X-API-Key");

        if (apiKey == null) {
            // 两种认证方式都不携带时，走匿名或返回 401
            return chain.filter(exchange);
        }

        ApiKeyInfo keyInfo = validApiKeys.get(apiKey);
        if (keyInfo == null) {
            return unauthorized(exchange, "Invalid API Key");
        }

        // 将客户端信息注入请求头
        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-Client-Id", keyInfo.clientId())
            .header("X-Auth-Method", "api-key")
            .build();
        return chain.filter(exchange.mutate().request(request).build());
    }

    record ApiKeyInfo(String clientId, List<String> scopes) {}
}
```

# 7. Token 刷新

## 7.1 自动 Token 刷新过滤器

```java
@Component
@Order(-250)
public class TokenRefreshGlobalFilter implements GlobalFilter {

    private static final long REFRESH_THRESHOLD = 60; // 剩余 60 秒时刷新

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return chain.filter(exchange);
        }

        String token = authHeader.substring(7);

        try {
            // 解码 JWT（不验证签名）检查过期时间
            Jwt jwt = decodeJwtWithoutValidation(token);
            Date expiresAt = jwt.getExpiresAt();
            Date now = new Date();

            // 检查是否即将过期
            if (expiresAt != null && expiresAt.getTime() - now.getTime()
                    < REFRESH_THRESHOLD * 1000) {
                // 调用认证服务刷新 Token
                return refreshToken(token)
                    .flatMap(newToken -> {
                        ServerHttpRequest request = exchange.getRequest()
                            .mutate()
                            .header(HttpHeaders.AUTHORIZATION, "Bearer " + newToken)
                            .build();
                        return chain.filter(
                            exchange.mutate().request(request).build());
                    })
                    .onErrorResume(e -> {
                        // 刷新失败，使用原 Token
                        log.warn("Token refresh failed", e);
                        return chain.filter(exchange);
                    });
            }
        } catch (Exception e) {
            log.warn("Failed to decode JWT", e);
        }

        return chain.filter(exchange);
    }

    private Jwt decodeJwtWithoutValidation(String token) {
        // 仅解码不验签
        return new NimbusReactiveJwtDecoder()
            .decode(token).block();
    }

    private Mono<String> refreshToken(String token) {
        // 调用认证服务器的刷新 Token 接口
        return WebClient.create()
            .post()
            .uri("https://auth.example.com/auth/realms/gateway-realm/protocol/openid-connect/token")
            .body(BodyInserters.fromFormData("grant_type", "refresh_token")
                .with("refresh_token", token))
            .retrieve()
            .bodyToMono(JsonNode.class)
            .map(node -> node.get("access_token").asText());
    }
}
```

# 8. 最佳实践

## 8.1 安全配置清单

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/auth/realms/gateway-realm

spring:
  cloud:
    gateway:
      default-filters:
        - RemoveRequestHeader=Cookie           # 不转发 Cookie
        - RemoveResponseHeader=WWW-Authenticate # 不暴露认证头

      routes:
        - id: public_api
          uri: lb://public-service
          predicates:
            - Path=/api/public/**
          # 公共 API 不需要认证

        - id: secured_api
          uri: lb://secured-service
          predicates:
            - Path=/api/secured/**
          filters:
            - TokenRelay=                       # 转发 Token
            - StripPrefix=1
          metadata:
            required-roles: user
```

## 8.2 常见问题

1. **JWT 与 Session 的选择**：
   - JWT 适合分布式、无状态场景
   - Session 适合有状态、集中式场景

2. **Token 存储**：
   - 前端：HttpOnly Cookie 或 Authorization Header
   - 后端：Redis 缓存 Token 黑名单

3. **安全注意事项**：
   - 不要在 JWT 中存储敏感信息（JWT 只签名不加密）
   - 合理设置 Token 过期时间（access_token 15 分钟，refresh_token 7 天）
   - 实现 Token 吊销机制（黑名单）

4. **性能优化**：
   - 缓存 JWK Set（公钥信息），避免每次请求都远程获取
   - JWT 验签比 OAuth2 Token 内省（Introspection）更高效
