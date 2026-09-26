
# 1. 概述

SkyWalking Java Agent 提供 Spring Boot 的**全自动集成**，无需修改代码。Agent 自动识别 Spring MVC、WebFlux、Scheduled、Async 等组件。

```
Spring Boot 应用
  │
  ├── Spring MVC / WebFlux → Entry Span (HTTP 端点)
  ├── @Service / @Repository → Local Span (方法级，需 optional 插件)
  ├── @Scheduled → Local Span (定时任务)
  ├── @Async → 跨线程 Trace 传播
  ├── RestTemplate / Feign → Exit Span (HTTP 调用)
  └── JDBC / Redis / Kafka → Exit Span (中间件访问)
```

# 2. 启动配置

```bash
java -javaagent:skywalking-agent.jar \
     -DSW_AGENT_NAME=order-service \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800 \
     -jar order-service.jar
```

```properties
# agent.config
agent.service_name=order-service
collector.backend_service=oap:11800
plugin.springmvc.collect_http_params=true
```

# 3. Spring MVC 自动埋点

Agent 自动拦截 `DispatcherServlet` 和相关 Controller 方法：

```
HTTP 请求 → DispatcherServlet.doDispatch()
  │
  ├── 拦截点 1: HandlerMapping → 提取 URL Pattern
  ├── 拦截点 2: HandlerAdapter.handle() → 执行 Controller
  │     └── 创建 Local Span: {ClassName}.{methodName}
  └── 拦截点 3: 响应 → 记录 HTTP 状态码
```

**自动记录的标签**：

| 标签 | 说明 |
|------|------|
| `url` | 请求 URL |
| `http.method` | HTTP 方法 |
| `status_code` | 响应状态码 |
| `http.params` | 请求参数（需配置开启） |

# 4. @Trace 注解增强

```java
@Service
public class OrderService {

    @Trace                    // 方法级 Span
    @Tag(key = "userId", value = "arg[0]")
    @Tag(key = "result", value = "returnedObj.status")
    public Order createOrder(Long userId, CreateOrderRequest req) {
        // ...
    }
}
```

# 5. 端点名称规范化

```properties
# 自定义端点名称映射
# 将 Restful 风格的 URL 参数替换为 {param}
# 例：/api/user/123 → /api/user/{id}
plugin.springmvc.urlSuffixRestPath=/api/user/\d+
```

# 6. Spring Cloud Gateway 集成

Agent 自动识别 Gateway 路由：

```
请求 /api/user/1 → Gateway 路由 → user-service
  │
  ├── Gateway Entry Span: /api/user/**
  ├── Route Exit Span: user-service
  └── user-service Entry Span: GET:/api/user/{id}
```

# 7. 忽略端点

```properties
# 不追踪健康检查和静态资源
trace.ignore_path=/health,/actuator/**,/*.ico,/*.js,/*.css

# 或通过 Optional 插件
# agent/optional-plugins/apm-trace-ignore-plugin.jar
```

# 8. Spring Boot Actuator 集成

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

Agent 自动标记 `/actuator` 端点，可在 SkyWalking UI 中过滤。

# 9. 最佳实践

1. **agent.service_name 唯一**：每个服务使用不同的 service_name
2. **开启 HTTP 参数需谨慎**：`collect_http_params` 可能泄露敏感信息
3. **端点规范化**：避免 Restful ID 导致端点爆炸（`/user/1` `/user/2` → `/user/{id}`）
4. **健康检查忽略**：避免 `/health` 请求污染 Trace 数据
