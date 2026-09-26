
# 1. 过滤器工厂架构

## 1.1 核心接口

```java
// 过滤器工厂接口
public interface GatewayFilterFactory<C> extends ShortcutConfigurable, Configurable<C> {
    // 将配置对象转换为 GatewayFilter
    GatewayFilter apply(C config);

    // 创建默认配置实例
    default C newConfig() {
        return BeanUtils.instantiateClass(getConfigClass());
    }

    Class<C> getConfigClass();
}

// GatewayFilter 接口
@FunctionalInterface
public interface GatewayFilter {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// 带 Order 的过滤器包装器
public class OrderedGatewayFilter implements GatewayFilter, Ordered {
    private final GatewayFilter delegate;
    private final int order;

    public OrderedGatewayFilter(GatewayFilter delegate, int order) {
        this.delegate = delegate;
        this.order = order;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return delegate.filter(exchange, chain);
    }

    @Override
    public int getOrder() {
        return order;
    }
}
```

## 1.2 工厂命名规范

`XxxGatewayFilterFactory` → 配置时使用 `Xxx`。

```yaml
filters:
  - AddRequestHeader=X-Custom, Value        # AddRequestHeaderGatewayFilterFactory
  - StripPrefix=1                            # StripPrefixGatewayFilterFactory
  - name: CircuitBreaker                     # CircuitBreakerGatewayFilterFactory
    args:
      name: myCB
```

## 1.3 Shortcut 配置与完全配置

**Shortcut 方式**（简洁）：

```yaml
filters:
  - AddRequestHeader=X-Request-Id, 123
  - RewritePath=/api/(?<segment>.*), /$\{segment}
  - SetPath=/{segment}
```

**完全配置方式**（参数明确）：

```yaml
filters:
  - name: AddRequestHeader
    args:
      name: X-Request-Id
      value: "123"
  - name: RewritePath
    args:
      regexp: /api/(?<segment>.*)
      replacement: /${segment}
```

# 2. 内置 GatewayFilter 工厂详解

## 2.1 请求/响应头操作过滤器

**AddRequestHeaderGatewayFilterFactory**：

```yaml
filters:
  - AddRequestHeader=X-Gateway-Source, spring-cloud
  - AddRequestHeader=X-Trace-Id, #{T(java.util.UUID).randomUUID()}
```

```java
public class AddRequestHeaderGatewayFilterFactory
        extends AbstractNameValueGatewayFilterFactory {

    @Override
    public GatewayFilter apply(NameValueConfig config) {
        return (exchange, chain) -> {
            // 构造新的请求头
            ServerHttpRequest request = exchange.getRequest().mutate()
                .header(config.getName(), config.getValue())
                .build();
            return chain.filter(exchange.mutate().request(request).build());
        };
    }
}
```

**AddResponseHeaderGatewayFilterFactory**：

```yaml
filters:
  - AddResponseHeader=X-Response-Time, #{T(System).currentTimeMillis()}
```

**RemoveRequestHeaderGatewayFilterFactory / RemoveResponseHeaderGatewayFilterFactory**：

```yaml
filters:
  - RemoveRequestHeader=Cookie          # 移除 Cookie 头
  - RemoveResponseHeader=Server         # 移除后端暴露的 Server 头
```

**SetRequestHeaderGatewayFilterFactory / SetResponseHeaderGatewayFilterFactory**：

替换（而非新增）指定的 Header：

```yaml
filters:
  - SetRequestHeader=Host, api.example.com
  - SetResponseHeader=X-Powered-By, Spring-Gateway
```

**MapRequestHeaderGatewayFilterFactory**：

将某个 Header 的值复制到另一个 Header：

```yaml
filters:
  - MapRequestHeader=X-Original-Ip, X-Forwarded-For
```

**SetRequestHostHeaderGatewayFilterFactory**：

```yaml
filters:
  - SetRequestHostHeader=internal-service.local
```

**DedupeResponseHeaderGatewayFilterFactory**：

去重响应头（避免重复的 CORS 头等）：

```yaml
filters:
  - DedupeResponseHeader=Access-Control-Allow-Origin, RETAIN_LAST
```

去重策略：

| 策略 | 说明 |
|------|------|
| `RETAIN_FIRST` | 保留第一个 |
| `RETAIN_LAST`（默认） | 保留最后一个 |
| `RETAIN_UNIQUE` | 保留所有唯一值 |

**RewriteResponseHeaderGatewayFilterFactory**：

```yaml
filters:
  - RewriteResponseHeader=Location, http://internal(.*), http://external$1
```

**RewriteLocationResponseHeaderGatewayFilterFactory**：

```yaml
filters:
  - name: RewriteLocationResponseHeader
    args:
      stripVersionOn: /api/.*
      hostValue: gateway.example.com
      protocolsRegex: http|https
```

**AddRequestHeadersIfNotPresentGatewayFilterFactory**：

```yaml
filters:
  - name: AddRequestHeadersIfNotPresent
    args:
      headers:
        - X-Custom-Header:value1
        - X-Another-Header:value2
```

**RemoveJsonAttributesResponseBodyGatewayFilterFactory**：

```yaml
filters:
  - name: RemoveJsonAttributesResponseBody
    args:
      attributeNames: password,secret,token
      deleteArgs: true
```

**RequestHeaderSizeGatewayFilterFactory**：

```yaml
filters:
  - name: RequestHeaderSize
    args:
      maxNameLength: 64
      maxValueLength: 512
```

## 2.2 路径操作过滤器

**RewritePathGatewayFilterFactory**：

```yaml
filters:
  - RewritePath=/api/v1/(?<segment>.*), /$\{segment}
```

**请求** `/api/v1/users/123` → **转发** `/users/123`

```java
public class RewritePathGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RewritePathGatewayFilterFactory.Config> {

    public static class Config {
        private String regexp;       // 正则表达式
        private String replacement;  // 替换字符串（支持 $ 引用分组）
    }

    @Override
    public GatewayFilter apply(Config config) {
        String regexp = config.getRegexp();
        String replacement = config.getReplacement()
            .replace("$\\", "\\$"); // 处理 YAML 中的转义

        return (exchange, chain) -> {
            ServerHttpRequest req = exchange.getRequest();
            String path = req.getURI().getRawPath();
            String newPath = path.replaceAll(regexp, replacement);
            ServerHttpRequest mutatedReq = req.mutate().path(newPath).build();
            return chain.filter(exchange.mutate().request(mutatedReq).build());
        };
    }
}
```

**SetPathGatewayFilterFactory**：

```yaml
filters:
  - SetPath=/{segment}
```

使用 Path 断言提取的 `{segment}` 变量重新构造路径。

**StripPrefixGatewayFilterFactory**：

```yaml
filters:
  - StripPrefix=2
```

**请求** `/api/v1/users`（parts=2）→ **转发** `/users`

**PrefixPathGatewayFilterFactory**：

```yaml
filters:
  - PrefixPath=/api
```

**请求** `/users` → **转发** `/api/users`

**PreserveHostHeaderGatewayFilterFactory**：

```yaml
filters:
  - PreserveHostHeader=
```

保留客户端原始 `Host` 头，不修改为后端地址。

## 2.3 状态码操作过滤器

**SetStatusGatewayFilterFactory**：

```yaml
filters:
  - SetStatus=UNAUTHORIZED         # Spring HttpStatus 枚举名称
  - SetStatus=401                  # 数字状态码
```

支持方式：
- HTTP 状态码数字：`401`, `200`, `503`
- `HttpStatus` 枚举名：`UNAUTHORIZED`, `OK`, `SERVICE_UNAVAILABLE`

**RedirectToGatewayFilterFactory**：

```yaml
filters:
  - RedirectTo=302, https://www.example.com/new-path
```

## 2.4 请求参数操作过滤器

**AddRequestParameterGatewayFilterFactory**：

```yaml
filters:
  - AddRequestParameter=source, gateway
```

**RemoveRequestParameterGatewayFilterFactory**：

```yaml
filters:
  - RemoveRequestParameter=token
```

**RewriteRequestParameterGatewayFilterFactory**：

```yaml
filters:
  - name: RewriteRequestParameter
    args:
      name: redirectUrl
      replacement: http://safe.example.com
```

**RequestHeaderToRequestUriGatewayFilterFactory**：

```yaml
filters:
  - name: RequestHeaderToRequestUri
    args:
      headerName: X-Gateway-Destination
```

使用请求头的值作为转发目标 URI。

## 2.5 请求体操作过滤器

**ModifyRequestBodyGatewayFilterFactory**：

```yaml
filters:
  - name: ModifyRequestBody
    args:
      inClass: java.lang.String
      outClass: java.lang.String
      newBody: "#{@modifyBodyFunction}"
```

```java
@Bean
public Function<String, String> modifyBodyFunction() {
    return body -> {
        // 修改请求体内容
        JsonObject json = JsonParser.parseString(body).getAsJsonObject();
        json.addProperty("modifiedBy", "gateway");
        return json.toString();
    };
}
```

```java
// 源码核心逻辑
public class ModifyRequestBodyGatewayFilterFactory
        extends AbstractGatewayFilterFactory<ModifyRequestBodyGatewayFilterFactory.Config> {

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 读取已有缓存的 body 或重新读取
            return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange, cachedBody -> {
                Class<?> inClass = config.getInClass();
                Class<?> outClass = config.getOutClass();
                // 反序列化
                Object body = deserialize(cachedBody, inClass);
                // 应用转换函数
                Mono<?> modifiedBody = applyTransform(config.getNewBody(), body, outClass);
                return modifiedBody.flatMap(newBody -> {
                    // 构造新请求
                    ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                        .headers(httpHeaders -> httpHeaders.remove(HttpHeaders.CONTENT_LENGTH)) // 移除 Content-Length
                        .build();
                    return chain.filter(exchange.mutate()
                        .request(mutatedRequest)
                        .build());
                });
            });
        };
    }
}
```

**ModifyResponseBodyGatewayFilterFactory**：

```yaml
filters:
  - name: ModifyResponseBody
    args:
      inClass: java.lang.String
      outClass: java.lang.String
      rewrite: "#{@modifyResponseBodyFunction}"
```

```java
@Bean
public Function<String, String> modifyResponseBodyFunction() {
    return body -> {
        // 包装响应体
        return "{\"code\":0,\"data\":" + body + "}";
    };
}
```

**CacheRequestBodyGatewayFilterFactory**：

```yaml
filters:
  - name: CacheRequestBody
    args:
      bodyClass: java.lang.String
```

用于提前缓存请求体供后续过滤器（如 `ModifyRequestBody`）使用。

**LocalResponseCacheGatewayFilterFactory**：

```yaml
filters:
  - name: LocalResponseCache
    args:
      timeToLive: 1m
      size: 5MB
```

基于 Caffeine 的本地响应缓存。

## 2.6 流量控制过滤器

**RequestSizeGatewayFilterFactory**：

```yaml
filters:
  - RequestSize=10MB
  - name: RequestSize
    args:
      maxSize: 10485760
```

超过大小限制返回 `413 PAYLOAD_TOO_LARGE`。

**RequestRateLimiterGatewayFilterFactory**：

详见 [10_rate-limiting.md] 的详细说明。

**CircuitBreakerGatewayFilterFactory**：

详见 [9_circuit-breaker.md] 的详细说明。

**RetryGatewayFilterFactory**：

```yaml
filters:
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE,GATEWAY_TIMEOUT
      methods: GET
      series: SERVER_ERROR
      exceptions: java.io.IOException,java.util.concurrent.TimeoutException
      backoff:
        firstBackoff: 100ms
        maxBackoff: 5s
        factor: 2
        basedOnPreviousValue: false
```

## 2.7 会话和安全过滤器

**SaveSessionGatewayFilterFactory**：

```yaml
filters:
  - SaveSession=
```

强制在转发前保存 `WebSession`（用于 Spring Session 的场景）。

```java
public class SaveSessionGatewayFilterFactory
        extends AbstractGatewayFilterFactory<Object> {

    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> exchange.getSession()
            .flatMap(WebSession::save)
            .then(chain.filter(exchange));
    }
}
```

**SecureHeadersGatewayFilterFactory**：

添加安全相关的响应头：

```yaml
filters:
  - SecureHeaders=
```

默认添加的头：

```java
public class SecureHeadersGatewayFilterFactory {
    public static final String X_XSS_PROTECTION_HEADER = "X-Xss-Protection";
    public static final String STRICT_TRANSPORT_SECURITY_HEADER = "Strict-Transport-Security";
    public static final String X_FRAME_OPTIONS_HEADER = "X-Frame-Options";
    public static final String X_CONTENT_TYPE_OPTIONS_HEADER = "X-Content-Type-Options";
    public static final String REFERRER_POLICY_HEADER = "Referrer-Policy";
    public static final String CONTENT_SECURITY_POLICY_HEADER = "Content-Security-Policy";
    public static final String X_DOWNLOAD_OPTIONS_HEADER = "X-Download-Options";
    public static final String X_PERMITTED_CROSS_DOMAIN_POLICIES_HEADER = "X-Permitted-Cross-Domain-Policies";

    // 默认值
    // X-Xss-Protection: 1; mode=block
    // Strict-Transport-Security: max-age=631138519
    // X-Frame-Options: DENY
    // X-Content-Type-Options: nosniff
    // Referrer-Policy: no-referrer
    // Content-Security-Policy: default-src 'self' https:; font-src 'self' https: data:; ...
    // X-Download-Options: noopen
    // X-Permitted-Cross-Domain-Policies: none
}
```

自定义配置：

```yaml
spring:
  cloud:
    gateway:
      filter:
        secure-headers:
          xss-protection-header: "1; mode=block"
          strict-transport-security: "max-age=31536000;includeSubdomains"
          frame-options: "SAMEORIGIN"
```

**TokenRelayGatewayFilterFactory**：

```yaml
filters:
  - TokenRelay=
```

将当前请求中的 OAuth2 访问令牌转发到下游服务。需要配合 `spring-boot-starter-oauth2-client` 使用。

## 2.8 gRPC 转换过滤器

**JsonToGrpcGatewayFilterFactory**：

```yaml
filters:
  - name: JsonToGrpc
    args:
      protoDescriptor: classpath:service.desc
      protoFile: service.proto
      service: com.example.Service
      method: handleRequest
```

将 JSON 请求转换为 gRPC 调用，使 gRPC 服务可以暴露为 HTTP API。

## 2.9 FallbackHeaders 过滤器

**FallbackHeadersGatewayFilterFactory**：

```yaml
filters:
  - name: FallbackHeaders
    args:
      executionExceptionTypeHeaderName: X-Exception-Type
      executionExceptionMessageHeaderName: X-Exception-Message
      rootCauseExceptionTypeHeaderName: X-Root-Cause-Type
      rootCauseExceptionMessageHeaderName: X-Root-Cause-Message
```

配合 `CircuitBreaker` 使用，将熔断异常信息以 Header 形式传递给 fallback 处理器。

# 3. 过滤器的执行顺序

## 3.1 Order 排序机制

路由级过滤器默认 order = 0，可以通过 `OrderedGatewayFilter` 自定义：

```java
@Component
public class CustomFilterFactory extends AbstractGatewayFilterFactory<Object> {
    @Override
    public GatewayFilter apply(Object config) {
        return new OrderedGatewayFilter((exchange, chain) -> {
            System.out.println("Pre: " + exchange.getRequest().getURI());
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                System.out.println("Post: " + exchange.getResponse().getStatusCode());
            }));
        }, -1); // 指定 order
    }
}
```

## 3.2 PRE 和 POST 执行阶段

```java
return chain.filter(exchange)
    .then(Mono.fromRunnable(() -> {
        // POST 阶段：响应返回后执行
    }));
```

完整过滤器链执行顺序：

```
Order = -∞                          Order = +∞
    │                                   ▲
    │   PRE 阶段（正向执行）             │
    ├───────────────────────────────────┤
    │  Lower order first                │
    │                                   │
    │   ↓ chain.filter() ↓              │
    │   （调用下一个过滤器）              │
    │                                   │
    │  POST 阶段（反向执行）             │
    │  Higher order first               │
    └───────────────────────────────────┘
```

# 4. 过滤器工厂的扩展机制

## 4.1 ShortcutType

控制快捷配置参数的解析方式：

```java
public enum ShortcutType {
    DEFAULT {           // "key=value1,value2" → {key: [value1, value2]}
        @Override
        public Map<String, Object> normalize(Map<String, String> args, ...) {
            return args;
        }
    },
    GATHER_LIST {       // "key=v1,v2" → {key: "v1,v2"} 列表形式
        @Override
        ...
    },
    GATHER_LIST_TAIL_FLAG { // 拼接成列表参数
        @Override
        ...
    }
}
```

## 4.2 自定义配置映射

```java
@Override
public Map<String, Object> shortcutFieldOrder() {
    // 定义快捷配置中参数的顺序
    return Collections.singletonMap("class", "inClass");
}
```

# 5. 过滤器配置最佳实践

## 5.1 按功能拆分过滤器

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - AddResponseHeader=X-Gateway, Spring-Cloud-Gateway
        - SecureHeaders=
      routes:
        - id: user_service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=2
            - name: Retry
              args:
                retries: 2
                statuses: BAD_GATEWAY
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

## 5.2 请求体修改的注意事项

1. `ModifyRequestBody` 需要配合 `CacheRequestBody` 或 `ReadBody` 使用
2. 修改请求体后需移除 `Content-Length` Header（框架自动处理）
3. 修改响应体需考虑性能影响，特别是大响应体

## 5.3 安全过滤器的推荐配置

```yaml
filters:
  - RemoveResponseHeader=Server           # 隐藏服务器信息
  - RemoveResponseHeader=X-Powered-By     # 隐藏技术栈
  - SecureHeaders=                        # OWASP 安全头
  - RequestSize=5MB                       # 限制请求大小
```
