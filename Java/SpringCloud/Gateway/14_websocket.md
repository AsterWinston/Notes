
# 1. WebSocket 代理概述

Spring Cloud Gateway 支持 WebSocket 代理，可以将客户端的 WebSocket 请求转发到后端服务。底层基于 Reactor Netty 的 WebSocket 支持。

```
客户端 (WS) ──→ Gateway ──→ 后端服务 (WS/WSS)
    ↑                    ↑
    │  升级请求           │ 握手转发
    └─────────────────────┘
        全双工通信
```

# 2. WebsocketRoutingFilter 源码解析

## 2.1 核心过滤器

```java
public class WebsocketRoutingFilter implements GlobalFilter, Ordered {
    public static final int WEBSOCKET_ROUTING_FILTER_ORDER = Ordered.LOWEST_PRECEDENCE - 2;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取目标 URL
        URI requestUrl = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String scheme = requestUrl.getScheme();

        // 只处理 ws/wss 协议
        if (!"ws".equals(scheme) && !"wss".equals(scheme)) {
            return chain.filter(exchange);
        }

        // 标记已路由
        ServerWebExchangeUtils.setAlreadyRouted(exchange);

        // 获取请求头（包含 Upgrade: websocket）
        HttpHeaders headers = exchange.getRequest().getHeaders();

        // 获取子协议（Sec-WebSocket-Protocol）
        List<String> subprotocols = headers.get("Sec-WebSocket-Protocol");

        // 构建 WebSocket URI
        URI wsUri = UriComponentsBuilder.fromUri(requestUrl)
            .scheme(scheme)
            .host(requestUrl.getHost())
            .port(requestUrl.getPort())
            .path(exchange.getRequest().getPath().pathWithinApplication().value())
            .query(exchange.getRequest().getURI().getRawQuery())
            .build().toUri();

        // 获取响应对象
        ServerHttpResponse response = exchange.getResponse();

        // 创建 WebSocket 会话
        Mono<Void> sessionMono = this.webSocketClient.execute(
            wsUri,
            // WebSocket 会话处理
            session -> {
                // 服务端到客户端
                Flux<WebSocketMessage> serverToClient = session
                    .receive()
                    .doOnNext(message -> {
                        // 处理接收到的消息（可选）
                    });

                // 客户端到服务端
                Flux<WebSocketMessage> clientToServer = ...;

                return session.send(serverToClient);
            }
        );

        return sessionMono;
    }
}
```

## 2.2 请求头处理

WebSocket 握手需要正确传递升级相关的请求头：

```java
// 核心请求头处理方法
private HttpHeaders filterHeaders(HttpHeaders headers) {
    HttpHeaders filtered = new HttpHeaders();
    // 复制允许传递的请求头
    headers.entrySet().stream()
        .filter(entry -> !entry.getKey().toLowerCase().startsWith("sec-websocket"))
        .forEach(entry -> filtered.addAll(entry.getKey(), entry.getValue()));

    // 添加必要的 WebSocket 头
    if (headers.containsKey("Sec-WebSocket-Protocol")) {
        filtered.addAll("Sec-WebSocket-Protocol",
            headers.get("Sec-WebSocket-Protocol"));
    }
    return filtered;
}
```

# 3. WebSocket 路由配置

## 3.1 基本 WebSocket 路由

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: websocket_route
          uri: ws://localhost:3001                            # WebSocket 后端
          predicates:
            - Path=/websocket/**
          filters:
            - StripPrefix=1
```

## 3.2 带负载均衡的 WebSocket

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ws_lb_route
          uri: lb:ws://websocket-service                      # 协议前缀 lb:ws
          predicates:
            - Path=/ws/**
          filters:
            - StripPrefix=1
```

**说明**：`lb:ws://service-name` 中 `lb` 是负载均衡协议前缀，`ws` 是实际采用的协议。

## 3.3 SockJS 降级

SockJS 是 WebSocket 的回退方案，当浏览器不支持 WebSocket 时使用 HTTP 长轮询：

```yaml
spring:
  cloud:
    gateway:
      routes:
        # WebSocket 路由
        - id: ws_route
          uri: ws://localhost:3001
          predicates:
            - Path=/ws/**

        # SockJS 降级路由（HTTP）
        - id: sockjs_route
          uri: http://localhost:3001
          predicates:
            - Path=/ws/info/**
```

# 4. WebSocket 配置参数

## 4.1 完整配置

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        websocket:
          # 最大帧长度（字节）
          max-frame-payload-length: 65536

          # 是否代理 ping/pong 帧
          proxy-ping: true

      httpserver:
        websocket:
          # 服务端最大帧长度
          max-frame-payload-length: 65536

          # 握手超时
          handshake-timeout-millis: 10000
```

## 4.2 自定义 WebSocket 客户端配置

```java
@Configuration
public class WebSocketConfiguration {

    @Bean
    public WebSocketClient webSocketClient() {
        return new ReactorNettyWebSocketClient(
            HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
                .doOnConnected(conn -> conn
                    .addHandlerLast(new WebSocketClientCompressionHandler()))
        );
    }
}
```

# 5. WebSocket 过滤器和拦截器

## 5.1 自定义 WebSocket 全局过滤器

```java
@Component
@Order(Ordered.LOWEST_PRECEDENCE - 3)
public class WebSocketAuthGlobalFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI requestUrl = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        if (requestUrl == null) return chain.filter(exchange);

        String scheme = requestUrl.getScheme();
        if (!"ws".equals(scheme) && !"wss".equals(scheme)) {
            return chain.filter(exchange);
        }

        // WebSocket 认证处理
        String token = exchange.getRequest().getHeaders()
            .getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            return unauthorized(exchange);
        }

        // 验证 Token（异步）
        return validateToken(token.substring(7))
            .flatMap(valid -> {
                if (!valid) {
                    return unauthorized(exchange);
                }
                // 添加自定义头到 WebSocket 握手请求
                ServerHttpRequest request = exchange.getRequest()
                    .mutate()
                    .header("X-Authenticated", "true")
                    .build();
                return chain.filter(
                    exchange.mutate().request(request).build());
            });
    }

    private Mono<Boolean> validateToken(String token) {
        return Mono.fromCallable(() -> {
            // JWT 验证逻辑
            return true;
        });
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
}
```

## 5.2 连接计数过滤器

```java
@Component
public class WebSocketConnectionCounter {

    private final AtomicInteger connectionCount = new AtomicInteger(0);
    private final int maxConnections = 10000;

    public boolean tryAcquire() {
        int current;
        do {
            current = connectionCount.get();
            if (current >= maxConnections) return false;
        } while (!connectionCount.compareAndSet(current, current + 1));
        return true;
    }

    public void release() {
        connectionCount.decrementAndGet();
    }

    public int getActiveConnections() {
        return connectionCount.get();
    }
}

// WebSocket 连接管理过滤器
@Component
@Order(Ordered.LOWEST_PRECEDENCE - 4)
public class WebSocketConnectionManagerFilter implements GlobalFilter {

    private final WebSocketConnectionCounter counter;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        if (url == null || !"ws".equals(url.getScheme())) {
            return chain.filter(exchange);
        }

        // 检查连接数限制
        if (!counter.tryAcquire()) {
            exchange.getResponse().setStatusCode(HttpStatus.SERVICE_UNAVAILABLE);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange)
            .doFinally(signalType -> counter.release());
    }
}
```

# 6. 协议升级源码分析

WebSocket 协议升级的核心在于 HTTP 升级握手：

```
客户端请求：                         网关转发：
GET /ws HTTP/1.1                    GET /ws HTTP/1.1
Host: gateway:8080                  Host: backend:3001
Upgrade: websocket                  Upgrade: websocket
Connection: Upgrade                 Connection: Upgrade
Sec-WebSocket-Key: xxx              Sec-WebSocket-Key: xxx
Sec-WebSocket-Version: 13           Sec-WebSocket-Version: 13
```

```java
// Netty 层握手源码简化
public class WebsocketRoutingFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // ...

        return this.webSocketClient.execute(wsUri, (wsSession) -> {
            // 连接建立后的双向消息转发
            Flux<WebSocketMessage> input = exchange.getRequest()
                .getBody()
                .map(dataBuffer -> wsSession
                    .bufferFactory()
                    .wrap(dataBuffer.asByteBuffer()));

            return wsSession.send(input)
                .and(wsSession.receive()
                    .map(webSocketMessage -> {
                        DataBuffer dataBuffer = exchange.getResponse()
                            .bufferFactory()
                            .wrap(webSocketMessage.getPayload());
                        return dataBuffer;
                    })
                    .doOnNext(dataBuffer -> {
                        // 写入响应
                    }));
        });
    }
}
```

# 7. WebSocket 安全性

## 7.1 同源检查

```java
@Component
@Order(Ordered.LOWEST_PRECEDENCE - 5)
public class WebSocketOriginCheckFilter implements GlobalFilter {

    private static final List<String> ALLOWED_ORIGINS = List.of(
        "https://example.com",
        "https://admin.example.com"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI requestUrl = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        if (requestUrl == null || !"ws".equals(requestUrl.getScheme())
                && !"wss".equals(requestUrl.getScheme())) {
            return chain.filter(exchange);
        }

        // 检查 Origin 头
        String origin = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.ORIGIN);
        if (origin != null && !ALLOWED_ORIGINS.contains(origin)) {
            return forbidden(exchange, "Origin not allowed: " + origin);
        }

        return chain.filter(exchange);
    }

    private Mono<Void> forbidden(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        return exchange.getResponse().setComplete();
    }
}
```

# 8. 监控与指标

## 8.1 连接监控

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true
```

自定义 WebSocket 连接指标：

```java
@Component
public class WebSocketMetrics {

    private final MeterRegistry meterRegistry;
    private final AtomicInteger activeConnections = new AtomicInteger(0);

    public WebSocketMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        // 注册 Gauge 指标
        Gauge.builder("gateway.websocket.connections", activeConnections,
                AtomicInteger::get)
            .description("Active WebSocket connections")
            .register(meterRegistry);
    }

    public void incrementConnections() {
        activeConnections.incrementAndGet();
    }

    public void decrementConnections() {
        activeConnections.decrementAndGet();
    }
}
```

# 9. 最佳实践

## 9.1 生产配置推荐

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        websocket:
          max-frame-payload-length: 65536
          proxy-ping: true

      httpserver:
        websocket:
          max-frame-payload-length: 65536
          handshake-timeout-millis: 10000
```

## 9.2 注意事项

1. **连接数管理**：WebSocket 是长连接，需要合理限制并发连接数
2. **心跳检测**：启用 `proxy-ping: true` 让网关代理心跳帧
3. **超时设置**：WebSocket 连接的超时应该比 HTTP 请求更长
4. **后端负载均衡**：WebSocket 的会话保持要求负载均衡器支持 session affinity（粘滞会话）
5. **身份认证**：WebSocket 握手阶段的认证 Token 建议放在 URL 参数或首个消息中
6. **监控告警**：关注 WebSocket 连接数和异常断连事件
