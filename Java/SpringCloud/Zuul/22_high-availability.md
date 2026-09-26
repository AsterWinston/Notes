
# 1. 高可用架构

Zuul 自身无状态，天然支持水平扩展。典型的 HA 架构：

```
                        DNS / LVS / SLB
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Nginx   │    │  Nginx   │    │  Nginx   │
        │ (Keepalived HA)  │          │          │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
      ┌──────┼──────┐ ┌──────┼──────┐ ┌──────┼──────┐
      ▼      ▼      ▼ ▼      ▼      ▼ ▼      ▼      ▼
  ┌──────┐┌──────┐┌──────┐ ...  (每个 Nginx 下挂多个 Zuul 实例)
  │ Zuul ││ Zuul ││ Zuul │
  │实例1 ││实例2 ││实例3 │    ← 无状态，无限扩展
  └──┬───┘└──┬───┘└──┬───┘
     │       │       │
     └───────┼───────┘
             ▼
     ┌───────────────┐
     │   后端微服务    │
     └───────────────┘
```

## 1.1 Nginx 反向代理配置

```nginx
upstream zuul_gateway {
    # 使用 IP Hash 保持会话（如果需要）
    # ip_hash;

    server 192.168.1.11:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 192.168.1.13:8080 weight=3 max_fails=3 fail_timeout=30s backup;
}

server {
    listen 80;
    server_name api.example.com;

    # 请求体大小限制
    client_max_body_size 100m;

    # 超时配置
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    # 代理请求到 Zuul
    location / {
        proxy_pass http://zuul_gateway;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 连接池
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

## 1.2 Keepalived 双机热备

```conf
# keepalived.conf - Master 节点
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        192.168.1.100/24  # VIP (虚拟IP)
    }

    # 健康检查：检测 Nginx 是否存活
    track_script {
        chk_nginx
    }
}

vrrp_script chk_nginx {
    script "/usr/bin/killall -0 nginx"
    interval 2
    weight -20
}
```

# 2. Zuul 自身配置

## 2.1 健康检查端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
```

Kubernetes 探针配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zuul-gateway
spec:
  template:
    spec:
      containers:
        - name: zuul-gateway
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

## 2.2 优雅关闭

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

配合 K8s 的 `terminationGracePeriodSeconds`：

```yaml
spec:
  terminationGracePeriodSeconds: 40  # 必须 > spring.lifecycle.timeout-per-shutdown-phase
```

# 3. 灰度发布

Zuul 通过自定义过滤器实现灰度路由：

```java
@Component
public class GrayscaleFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return PreDecorationFilter.FILTER_ORDER - 1; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // 灰度策略：
        // 1. 根据用户 ID 的 Hash 值匹配
        String userId = request.getHeader("X-User-Id");
        if (userId != null) {
            int hash = Math.abs(userId.hashCode()) % 100;
            if (hash < 10) {  // 前 10% 的用户走灰度
                String serviceId = (String) ctx.get("serviceId");
                ctx.set("serviceId", serviceId + "-canary");
                ctx.set("grayscale", true);
            }
        }

        // 2. 根据 Header 标记
        String grayTag = request.getHeader("X-Grayscale-Tag");
        if ("true".equals(grayTag)) {
            String serviceId = (String) ctx.get("serviceId");
            ctx.set("serviceId", serviceId + "-canary");
            ctx.set("grayscale", true);
        }

        // 3. 根据请求参数
        if ("canary".equals(request.getParameter("version"))) {
            String serviceId = (String) ctx.get("serviceId");
            ctx.set("serviceId", serviceId + "-canary");
            ctx.set("grayscale", true);
        }

        return null;
    }
}
```

# 4. 部署检查清单

## 4.1 启动前检查

```
□ 依赖的 Eureka / Nacos 注册中心已启动
□ 后端服务至少有一个实例已注册
□ 配置文件（application.yml）中的路由规则已确认
□ 敏感 Header 配置正确（Cookie/Authorization 的传递策略）
□ Hystrix 超时 > Ribbon 超时总和
□ 连接池大小已根据预期 QPS 计算
□ 日志级别已确认（生产环境使用 WARN 或 ERROR）
□ Tomcat 线程池大小已根据预期并发量计算
```

## 4.2 运行中检查

```
□ /actuator/health 返回 UP
□ /actuator/routes 显示的路由规则正确
□ /actuator/filters 显示的过滤器顺序正确
□ Hystrix 仪表盘中无异常打开的熔断器
□ GC 日志正常，无频繁 Full GC
□ 线程池无长时间排队（Hystrix metrics 中 queueSize 保持在低位）
```

## 4.3 灰度上线策略

```
□ 先在 1 台实例上部署新版本
□ 观察 N 分钟，确认无异常
□ 逐步替换其他实例（每次 1 台）
□ 保留 1 台旧版本作为快速回滚备机
□ 全部部署完成后观察 1 小时
□ 通知下游服务的负责团队
```

# 5. 生产环境完整配置

```yaml
server:
  port: 8080
  shutdown: graceful
  tomcat:
    threads:
      max: 200
      min-spare: 20
    accept-count: 200
    max-connections: 20000
    connection-timeout: 20000

spring:
  application:
    name: zuul-gateway
  lifecycle:
    timeout-per-shutdown-phase: 30s

# Eureka 配置
eureka:
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
  client:
    registry-fetch-interval-seconds: 10
    service-url:
      defaultZone: http://eureka1:8761/eureka/,http://eureka2:8762/eureka/

# Zuul 路由配置
zuul:
  retryable: true
  prefix: /api
  stripPrefix: false

  sensitive-headers:
    - Cookie
    - Set-Cookie
    - Authorization

  ignored-services: config-server,monitor-service

  host:
    max-total-connections: 500
    max-per-route-connections: 200
    socket-timeout-millis: 10000
    connect-timeout-millis: 2000
    time-to-live: 600000
    time-unit: MILLISECONDS

  ribbon-isolation-strategy: SEMAPHORE
  semaphore:
    max-semaphores: 200

  routes:
    user-service:
      path: /user/**
      serviceId: user-service
    order-service:
      path: /order/**
      serviceId: order-service

# Ribbon 配置
ribbon:
  eager-load:
    enabled: true
    clients:
      - user-service
      - order-service
  ReadTimeout: 8000
  ConnectTimeout: 2000
  MaxAutoRetries: 0
  MaxAutoRetriesNextServer: 1

# Hystrix 配置
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 15000
        semaphore:
          maxConcurrentRequests: 200
      circuitBreaker:
        requestVolumeThreshold: 20
        errorThresholdPercentage: 50
        sleepWindowInMilliseconds: 5000
  threadpool:
    default:
      coreSize: 20
      maximumSize: 40
      maxQueueSize: 200
      queueSizeRejectionThreshold: 100

# 监控端点
management:
  endpoints:
    web:
      exposure:
        include: health,info,routes,filters,prometheus
  endpoint:
    health:
      show-details: never
  metrics:
    export:
      prometheus:
        enabled: true

# 日志配置
logging:
  level:
    root: WARN
    com.example.zuul: INFO
    com.netflix.zuul: WARN
    org.springframework.cloud.netflix.zuul: DEBUG
    ACCESS_LOG: INFO
  file:
    name: /var/log/zuul-gateway/application.log

# 错误页配置
server:
  error:
    include-stacktrace: never
    whitelabel:
      enabled: false
```

# 6. 从 Zuul 迁移到 Gateway 的建议

如果计划从 Zuul 1.x 迁移到 Spring Cloud Gateway，需要注意：

| 迁移点 | Zuul 1.x | Spring Cloud Gateway |
|--------|----------|---------------------|
| 路由 `zuul.routes.*` | `spring.cloud.gateway.routes[*]` |
| `sensitiveHeaders` | `filters[].RemoveRequestHeader` / `RemoveResponseHeader` |
| Pre 过滤器 | `GatewayFilter` 作用在路由或全局 |
| Route 过滤器 | 内置 `lb://` 路由 |
| Post 过滤器 | `GatewayFilter` 或 `GlobalFilter` |
| Error 过滤器 | `@ExceptionHandler` 或 `WebExceptionHandler` |
| `@EnableZuulProxy` | `@EnableGateway`（Spring Cloud Gateway 自动配置） |
| `ZuulFilter` | `GatewayFilter` / `GlobalFilter` + `Ordered` 接口 |
| `RequestContext` (ThreadLocal) | `ServerWebExchange` (Reactor Context) |
| Hystrix 熔断 | Resilience4j |
| Ribbon 负载均衡 | Spring Cloud LoadBalancer |
| URL 路由 | `lb://serviceId` |
| 请求体多次读取 | `ModifyRequestBody` / `CacheRequestBody` |

迁移工具函数映射：

```java
// Zuul 的 RequestContext 操作 → Gateway 的 ServerWebExchange 操作

// Zuul: ctx.addZuulRequestHeader("X-User-Id", userId)
// Gateway 等价:
exchange.getRequest().mutate()
    .header("X-User-Id", userId).build();

// Zuul: ctx.setSendZuulResponse(false)
// Gateway 等价:
exchange.getResponse().setComplete();  // 终止请求链

// Zuul: ctx.setResponseStatusCode(401)
// Gateway 等价:
exchange.getResponse().setRawStatusCode(401);
```

# 7. 总结

Zuul 1.x 作为 Spring Cloud 第一代网关，虽然技术架构上存在阻塞 I/O 的局限，但其**过滤器链模式**、**动态路由机制**和**与 Netflix OSS 的深度集成**在特定的技术时代中扮演了重要角色。对于维护存量项目或学习网关设计模式的开发者，深入理解 Zuul 仍有价值。

对于新项目或需要高性能、响应式编程的场景，建议直接选择 **Spring Cloud Gateway**。
