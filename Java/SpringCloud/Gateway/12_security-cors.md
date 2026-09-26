
# 1. CORS 配置

## 1.1 全局 CORS 配置

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            # 允许的来源
            allowedOrigins:
              - "https://example.com"
              - "https://*.example.com"
              # 注意: 不要在生产环境使用 "*"
            # 允许的 HTTP 方法
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            # 允许的请求头
            allowedHeaders:
              - Content-Type
              - Authorization
              - X-Requested-With
            # 是否允许携带 Cookie
            allowCredentials: true
            # 暴露给客户端的响应头
            exposedHeaders:
              - X-Custom-Header
              - X-Total-Count
            # 预检请求缓存时间（秒）
            maxAge: 3600
        # 是否添加到简单请求的 HandlerMapping（默认 false）
        add-to-simple-url-handler-mapping: true
```

## 1.2 按路由 CORS 配置

通过路由元数据设置 CORS：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: cors_route
          uri: http://backend
          predicates:
            - Path=/api/**
          metadata:
            cors:
              allowedOrigins: "https://frontend.example.com"
              allowedMethods: GET,POST
              allowedHeaders: Content-Type
              allowCredentials: "true"
              maxAge: "1800"
```

## 1.3 CORS 过滤器的实现原理

```java
public class CorsGatewayFilter implements GatewayFilter {

    private final CorsConfiguration configuration;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 检查是否有共享的 CORS 配置
        CorsConfiguration globalCors = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_GLOBAL_CORS_ATTRIBUTE);
        CorsConfiguration routeCors = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_ROUTE_CORS_ATTRIBUTE);

        // 合并配置
        CorsConfiguration corsConfig = globalCors != null ? globalCors : routeCors;
        if (corsConfig == null) {
            return chain.filter(exchange);
        }

        // 获取 CORS 处理器
        DefaultCorsProcessor processor = new DefaultCorsProcessor();
        // 处理跨域请求
        boolean valid = processor.processRequest(corsConfig, exchange);
        if (!valid || CorsUtils.isPreFlightRequest(exchange.getRequest())) {
            // 预检请求直接返回
            return Mono.empty();
        }

        return chain.filter(exchange);
    }
}
```

## 1.4 CORS 配置合并逻辑

```java
// 源码：GatewayAutoConfiguration 中的 CORS 配置合并
@Bean
public WebFilter corsWebFilter() {
    return (exchange, chain) -> {
        // 从路由元数据中提取 CORS 配置
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route != null && route.getMetadata().containsKey("cors")) {
            Object corsMeta = route.getMetadata().get("cors");
            CorsConfiguration routeCorsConfig = parseCorsConfiguration(corsMeta);
            exchange.getAttributes().put(
                ServerWebExchangeUtils.GATEWAY_ROUTE_CORS_ATTRIBUTE, routeCorsConfig);
        }

        // 合并全局 CORS 配置
        CorsConfiguration globalCorsConfig = this.globalCorsProperties
            .getCorsConfigurations()
            .entrySet().stream()
            .filter(entry -> pathMatcher.match(entry.getKey(),
                exchange.getRequest().getPath().value()))
            .map(Map.Entry::getValue)
            .findFirst()
            .orElse(null);

        if (globalCorsConfig != null) {
            exchange.getAttributes().put(
                ServerWebExchangeUtils.GATEWAY_GLOBAL_CORS_ATTRIBUTE, globalCorsConfig);
        }

        return chain.filter(exchange);
    };
}
```

# 2. SecureHeaders 过滤器

## 2.1 默认安全头

Spring Cloud Gateway 的 `SecureHeadersGatewayFilterFactory` 默认添加如下安全响应头：

| 响应头 | 默认值 |
|--------|--------|
| `X-Xss-Protection` | `1; mode=block` |
| `Strict-Transport-Security` | `max-age=631138519` |
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `no-referrer` |
| `Content-Security-Policy` | `default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline'` |
| `X-Download-Options` | `noopen` |
| `X-Permitted-Cross-Domain-Policies` | `none` |

## 2.2 自定义 SecureHeaders

```yaml
spring:
  cloud:
    gateway:
      filter:
        secure-headers:
          xss-protection-header: "1; mode=block"
          strict-transport-security: "max-age=31536000; includeSubDomains; preload"
          frame-options: "SAMEORIGIN"
          content-type-options: "nosniff"
          referrer-policy: "strict-origin-when-cross-origin"
          content-security-policy: "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
          download-options: "noopen"
          permitted-cross-domain-policies: "none"
```

## 2.3 仅对指定路由启用

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: secured_route
          uri: http://backend
          predicates:
            - Path=/secure/**
          filters:
            - SecureHeaders=
```

# 3. CSRF 防护

## 3.1 WebFlux CSRF 配置

由于 Spring Cloud Gateway 基于 WebFlux，需要手动配置 CSRF：

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfiguration {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(
            ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieServerCsrfTokenRepository.withHttpOnlyFalse())
                .requireCsrfProtectionMatcher(
                    new NegatedServerWebExchangeMatcher(
                        ServerWebExchangeMatchers.pathMatchers(
                            "/webhook/**",   // Webhook 回调不需要 CSRF
                            "/actuator/**"   // Actuator 端点不需要 CSRF
                        )))
            )
            .build();
    }
}
```

## 3.2 API 场景禁用 CSRF

对于纯 API 的网关场景，通常可以禁用 CSRF：

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfiguration {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(
            ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .build();
    }
}
```

# 4. IP 黑白名单

## 4.1 基于 RemoteAddr 断言

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: internal_api
          uri: lb://internal-service
          predicates:
            - Path=/internal/**
            - RemoteAddr=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```

## 4.2 自定义 IP 过滤全局过滤器

```java
@Component
@Order(-1000)
public class IpWhitelistGlobalFilter implements GlobalFilter {

    private static final List<String> WHITE_LIST = Arrays.asList(
        "192.168.0.0/16",
        "10.0.0.0/8"
    );

    private static final List<String> BLACK_LIST = Arrays.asList(
        "1.2.3.4"
    );

    private final List<IpSubnetFilterRule> whiteRules;
    private final List<String> blackList;

    public IpWhitelistGlobalFilter() {
        this.whiteRules = WHITE_LIST.stream()
            .map(cidr -> {
                String[] parts = cidr.split("/");
                return new IpSubnetFilterRule(parts[0],
                    Integer.parseInt(parts[1]), IpFilterRuleType.ACCEPT);
            })
            .collect(Collectors.toList());
        this.blackList = BLACK_LIST;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        InetSocketAddress remoteAddr = exchange.getRequest().getRemoteAddress();
        if (remoteAddr == null) {
            return forbidden(exchange, "Unknown remote address");
        }

        String ip = remoteAddr.getAddress().getHostAddress();

        // 黑名单检查
        if (blackList.contains(ip)) {
            return forbidden(exchange, "IP blocked: " + ip);
        }

        // 白名单检查（白名单模式：不在白名单内的拒绝）
        boolean inWhiteList = whiteRules.stream()
            .anyMatch(rule -> rule.matches(remoteAddr.getAddress()));
        if (!inWhiteList) {
            return forbidden(exchange, "IP not allowed: " + ip);
        }

        return chain.filter(exchange);
    }

    private Mono<Void> forbidden(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        exchange.getResponse().getHeaders()
            .setContentType(MediaType.APPLICATION_JSON);
        byte[] body = ("{\"code\":403,\"message\":\"" + message + "\"}").getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

# 5. 请求验签

## 5.1 实现一个验签过滤器

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api_route
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - name: SignatureVerify
              args:
                secretKey: your-secret-key
```

```java
@Component
public class SignatureVerifyGatewayFilterFactory
        extends AbstractGatewayFilterFactory<SignatureVerifyGatewayFilterFactory.Config> {

    public SignatureVerifyGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // 获取签名相关头
            String timestamp = request.getHeaders()
                .getFirst("X-Timestamp");
            String nonce = request.getHeaders()
                .getFirst("X-Nonce");
            String signature = request.getHeaders()
                .getFirst("X-Signature");

            if (timestamp == null || nonce == null || signature == null) {
                return unauthorized(exchange, "Missing signature headers");
            }

            // 验证时间戳（防重放攻击，5 分钟内有效）
            long ts = Long.parseLong(timestamp);
            if (Math.abs(System.currentTimeMillis() - ts) > 5 * 60 * 1000) {
                return unauthorized(exchange, "Request expired");
            }

            // 构造签名字符串
            String method = request.getMethod().name();
            String path = request.getPath().value();
            String query = request.getURI().getRawQuery();
            String body = ...; // 获取请求体

            String signStr = method + "\n" + path + "\n" + query + "\n"
                + body + "\n" + timestamp + "\n" + nonce;

            // 验证签名
            String expectedSign = hmacSha256(signStr, config.getSecretKey());
            if (!expectedSign.equals(signature)) {
                return unauthorized(exchange, "Invalid signature");
            }

            // 验签通过，将用户信息放入请求头
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-Authenticated", "true")
                .build();
            return chain.filter(exchange.mutate()
                .request(mutatedRequest).build());
        };
    }

    private String hmacSha256(String data, String key) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKey = new SecretKeySpec(
                key.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
            mac.init(secretKey);
            byte[] bytes = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(bytes);
        } catch (Exception e) {
            throw new RuntimeException("HMAC-SHA256 failed", e);
        }
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        byte[] body = ("{\"code\":401,\"message\":\"" + message + "\"}").getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }

    public static class Config {
        private String secretKey;
        public String getSecretKey() { return secretKey; }
        public void setSecretKey(String secretKey) { this.secretKey = secretKey; }
    }
}
```

# 6. 请求头清理

## 6.1 清理敏感请求头

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        # 防止内部信息泄露到外部
        - RemoveRequestHeader=Cookie                    # 不转发 Cookie
        - RemoveRequestHeader=X-Internal-Token          # 移除内部 Token
        - RemoveResponseHeader=Server                   # 隐藏后端服务器信息
        - RemoveResponseHeader=X-Powered-By             # 隐藏技术栈
        - RemoveResponseHeader=WWW-Authenticate
        - RemoveResponseHeader=Transfer-Encoding
```

## 6.2 添加转发头

```yaml
spring:
  cloud:
    gateway:
      x-forwarded:
        enabled: true
        proto-enabled: true          # 添加 X-Forwarded-Proto
        host-enabled: true           # 添加 X-Forwarded-Host
        port-enabled: true           # 添加 X-Forwarded-Port
        for-enabled: true            # 添加 X-Forwarded-For
        for-append: true             # 追加到现有值
```

# 7. HTTPS 强制跳转

## 7.1 全局 HTTPS 重定向

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: https_redirect
          uri: https://${target.host}
          predicates:
            - Path=/**
          filters:
            - RedirectTo=301, https://example.com
```

## 7.2 使用过滤器实现

```java
@Component
public class HttpsRedirectGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        URI uri = request.getURI();
        String scheme = uri.getScheme();

        // 如果已经是 HTTPS 或者不需要 HTTPS 的路径，跳过
        if ("https".equals(scheme) || uri.getPath().startsWith("/actuator")) {
            return chain.filter(exchange);
        }

        // 重定向到 HTTPS
        URI httpsUri = URI.create("https://" + uri.getHost()
            + (uri.getPort() > 0 ? ":" + uri.getPort() : "")
            + uri.getRawPath()
            + (uri.getRawQuery() != null ? "?" + uri.getRawQuery() : ""));

        exchange.getResponse().setStatusCode(HttpStatus.MOVED_PERMANENTLY);
        exchange.getResponse().getHeaders()
            .setLocation(httpsUri);
        return exchange.getResponse().setComplete();
    }

    @Override
    public int getOrder() {
        return -2000;
    }
}
```

# 8. 安全最佳实践

## 8.1 安全配置清单

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        # 安全头
        - SecureHeaders=
        # 隐藏后端信息
        - RemoveResponseHeader=Server
        - RemoveResponseHeader=X-Powered-By
        # 限制请求大小
        - name: RequestSize
          args:
            maxSize: 10MB
        # 转发头
      x-forwarded:
        enabled: true
        for-enabled: true

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins:
              - "https://trusted-frontend.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
            allowedHeaders:
              - Content-Type
              - Authorization
            allowCredentials: true
            maxAge: 3600
```

## 8.2 安全检查清单

1. **CORS 限制**：不设置 `Access-Control-Allow-Origin: *`
2. **敏感头清理**：移除 `Cookie`、`Authorization`、内部 Token
3. **安全头设置**：启用 `SecureHeaders` 过滤器
4. **请求大小限制**：设置合理的 `RequestSize` 限制
5. **限流防护**：配置合理的限流策略
6. **HTTPS**：生产环境强制 HTTPS
7. **IP 白名单**：管理后台限制内网访问
8. **请求验签**：对外 API 开启签名验证
