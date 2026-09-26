
# 1. Spring Cloud Gateway 集成

SkyWalking Agent 自动拦截 Gateway 的请求处理：

```
客户端请求 /api/user/1
  │
  ▼
Spring Cloud Gateway
  │
  ├── Entry Span: GET:/api/user/{id}    ← 网关层入口
  │
  ├── Route → user-service
  │     └── Exit Span: user-service
  │           │
  │           ▼ (Feign/LB HTTP 调用)
  │     user-service
  │       └── Entry Span: GET:/api/user/{id}
  │
  └── Response
```

# 2. Gateway 自动化

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
```

Agent 自动为每个路由创建 Exit Span，peer 为路由的目标服务名。

# 3. Sentinel 集成

Sentinel 插件在流控/熔断发生时自动标记 Span：

```java
@RestController
public class UserController {

    @GetMapping("/user/{id}")
    @SentinelResource(value = "getUser", blockHandler = "handleBlock")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }

    public User handleBlock(Long id, BlockException ex) {
        // SkyWalking 自动在此 Span 中标记流控事件
        // Span 标签: sentinel.block=true
        return new User(id, "Blocked", 0);
    }
}
```

## 3.1 Sentinel Span 标签

| 标签 | 值 | 含义 |
|------|-----|------|
| `sentinel.block` | true/false | 是否被 Sentinel 拦截 |
| `sentinel.rule` | flow/degrade/system | 拦截规则类型 |
| `sentinel.resource` | resourceName | 被保护的资源名 |

# 4. Trace + 流量防护联动

```
Trace: /api/user/{id} (200ms)
  ├── Span: Sentinel Entry (1ms)
  │     └── 流控检查通过
  ├── Span: UserService.getById (150ms)
  │     └── Span: JDBC select (100ms)
  └── Span: Sentinel Exit (1ms)

如果被 Sentinel 流控：
Trace: /api/user/{id} (2ms)
  └── Span: Sentinel Block (2ms)
        ├── Tags: sentinel.block=true, sentinel.rule=flow
        └── 直接返回 blockHandler 结果
```

# 5. Gateway + Sentinel 联合

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      scg:
        fallback:
          mode: response
          response-body: '{"code":429}'
```

```
Gateway 被限流时:
  Entry Span → 直接标记为 sentinel.block=true
  不再有下游服务的 Span
  整个 Trace 耗时仅包含 Gateway 的 Sentinel 检查时间
```

# 6. Nginx + SkyWalking

```nginx
# Nginx Lua 模块集成
# 安装 skywalking-nginx-lua
http {
    lua_shared_dict tracing_buffer 100m;

    server {
        listen 80;

        location / {
            # SkyWalking Nginx 插件自动创建 Nginx Span
            # 将 sw8 头传播给上游服务
            proxy_pass http://gateway;
            proxy_set_header sw8 $sw8_header;
        }
    }
}
```

# 7. 最佳实践

1. **网关层保留原始 TraceId**：确保 Nginx → Gateway → Service 的 TraceId 不中断
2. **Sentinel 流控事件追踪**：在 Dashboard 中过滤 `sentinel.block=true` 的 Trace
3. **Gateway 端点规范**：Gateway 层端点 = 路由 Pattern（如 `/api/user/**`）
