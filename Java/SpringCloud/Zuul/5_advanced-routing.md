
# 1. 自定义 RouteLocator

Spring Cloud Zuul 允许通过编程方式扩展路由规则，所有路由的查找最终由 `RouteLocator` 接口定义：

```java
public interface RouteLocator {
    // 获取当前所有路由
    List<Route> getRoutes();

    // 根据请求路径匹配路由
    Route getMatchingRoute(String path);

    // 刷新路由（实现 RefreshRouteLocator 时需要）
    default void refresh() {}
}
```

## 1.1 编程式路由定义

```java
@Configuration
public class CustomRouteConfig {

    @Bean
    public RouteLocator customRouteLocator(DiscoveryClient discoveryClient,
                                           ZuulProperties zuulProperties,
                                           ServiceRouteMapper serviceRouteMapper) {
        return new DiscoveryClientRouteLocator("/", discoveryClient, zuulProperties, serviceRouteMapper) {
            @Override
            public List<Route> getRoutes() {
                List<Route> routes = new ArrayList<>();

                // 自定义路由规则 1：灰度路由
                routes.add(new Route("gray-user", "/gray/user/**",
                        "http://gray-user-service:8080", "/user",
                        true, new LinkedHashSet<>()));

                // 自定义路由规则 2：读取数据库动态路由
                List<DbRoute> dbRoutes = loadRoutesFromDatabase();
                for (DbRoute dbRoute : dbRoutes) {
                    routes.add(new Route(
                        dbRoute.getId(),
                        dbRoute.getPath(),
                        dbRoute.getUrl(),
                        dbRoute.getPrefix(),
                        dbRoute.isStripPrefix(),
                        dbRoute.getSensitiveHeaders()
                    ));
                }
                return routes;
            }
        };
    }

    private List<DbRoute> loadRoutesFromDatabase() {
        // 从数据库或配置中心加载动态路由
        return Collections.emptyList();
    }
}
```

## 1.2 Route 对象结构

```java
public class Route {
    private String id;               // 路由唯一标识
    private String fullPath;         // Ant 路径表达式，如 "/user/**"
    private String location;         // 目标地址，如 "http://..." 或 "user-service"
    private String prefix;           // 前缀路径
    private boolean prefixStripped;  // 是否剥离前缀
    private Set<String> sensitiveHeaders; // 敏感 Header 集合
    private boolean customSensitiveHeaders; // 是否使用自定义敏感 Header 配置
}
```

# 2. 路由前缀（prefix）

## 2.1 全局前缀

```yaml
zuul:
  prefix: /api                # 所有路由路径自动加上 /api 前缀
  stripPrefix: true           # 转发时剥离 /api
  routes:
    user-service:
      path: /user/**
      serviceId: user-service
```

效果：

```
客户端请求:  http://gateway:8080/api/user/profile
匹配的路由:  /api/user/** → user-service
转发到后端:  http://user-service/user/profile   (stripPrefix=true 时剥离 /api)
```

## 2.2 局部前缀与全局前缀的组合

```yaml
zuul:
  prefix: /api
  routes:
    user-service:
      path: /v1/user/**
      serviceId: user-service
```

匹配规则：`/api` + `/v1/user/**` = `/api/v1/user/**`

当 `stripPrefix=true`（默认）时：
- 先剥离全局前缀 `/api`
- 再剥离路由前缀 `/v1/user`

# 3. 路由排序

## 3.1 配置式路由排序

`SimpleRouteLocator` 使用 `LinkedHashMap` 保存路由，**按配置写入顺序**进行匹配。可以通过定义 Bean 的方式控制路由排序：

```java
@Configuration
public class RouteOrderConfig {

    @Primary
    @Bean
    public SimpleRouteLocator simpleRouteLocator(
            ServerProperties server,
            ZuulProperties zuulProperties) {
        // 自定义路由加载顺序
        zuulProperties.getRoutes().putIfAbsent("high-priority", new ZuulRoute(
            "high-priority", "/important/**", null,
            "http://high-priority-service:8080", true, null, null
        ));
        return new SimpleRouteLocator(server.getServlet().getContextPath(),
            zuulProperties, "/");
    }
}
```

## 3.2 编程式路由排序

通过自定义 `RouteLocator` 并覆盖 `getMatchingRoute()` 方法：

```java
@Component
public class SortedRouteLocator extends SimpleRouteLocator
        implements RefreshableRouteLocator {

    public SortedRouteLocator(ServerProperties server,
                               ZuulProperties zuulProperties) {
        super(server.getServlet().getContextPath(), zuulProperties);
    }

    @Override
    protected Route getZuulRoute(String adjustedPath) {
        // 自定义排序：按路径长度降序（更具体的路由优先匹配）
        List<Map.Entry<String, ZuulRoute>> sortedRoutes =
            new ArrayList<>(this.getRoutesMap().entrySet());
        sortedRoutes.sort((a, b) ->
            Integer.compare(
                b.getValue().getPath().length(),
                a.getValue().getPath().length()
            ));

        for (Map.Entry<String, ZuulRoute> entry : sortedRoutes) {
            ZuulRoute route = entry.getValue();
            if (this.pathMatcher.match(route.getPath(), adjustedPath)) {
                return route;
            }
        }
        return null;
    }
}
```

# 4. 路径重写与转换

## 4.1 Pattern 提取与拼接

Zuul 在 `PreDecorationFilter` 中进行路径转换，核心方法：

```java
// PreDecorationFilter 中路径处理逻辑
protected Object run() {
    RequestContext ctx = RequestContext.getCurrentContext();
    // ...
    String location = route.getLocation();
    String path = getPath(ctx.getRequest());
    String targetPath = path;

    // 1. 剥离前缀
    if (route.getPrefix() != null && !route.getPrefix().isEmpty()) {
        targetPath = stripPrefix(path, route.getPrefix());
    }

    // 2. 拼接目标 URL
    if (route.getLocation().startsWith("http")) {
        // URL 路由：直接拼接
        location = location + targetPath;
    } else {
        // serviceId 路由：保留 serviceId
        // 由 RibbonRoutingFilter 处理
    }
    ctx.set("requestURI", targetPath);
    // ...
}
```

## 4.2 自定义路径转换过滤器

```java
@Component
public class CustomPathRewriteFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        // 在 PreDecorationFilter (order=5) 之后执行
        return 10;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String requestUri = ctx.getRequest().getRequestURI();
        return requestUri.startsWith("/v1/") || requestUri.startsWith("/v2/");
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String requestUri = ctx.getRequest().getRequestURI();

        // /v1/user/list → /api/v1/user/list
        String targetUri = "/api" + requestUri;
        ctx.set("requestURI", targetUri);

        // 也可以将版本号转换为 Header
        if (requestUri.startsWith("/v1/")) {
            ctx.addZuulRequestHeader("X-API-Version", "v1");
        } else if (requestUri.startsWith("/v2/")) {
            ctx.addZuulRequestHeader("X-API-Version", "v2");
        }

        return null;
    }
}
```

# 5. 多注册中心的路由

当需要路由到不同注册中心的服务时：

```yaml
zuul:
  routes:
    # 路由规则仍然在 zuul.routes 下定义
    biz-service:
      path: /biz/**
      serviceId: biz-service

    ops-service:
      path: /ops/**
      serviceId: ops-service
      # 注意：跨注册中心需要额外配置 Ribbon 的 NIWSServerList 实现

ribbon:
  eureka:
    enabled: true

# 为不同 serviceId 指定不同的注册中心（通过自定义 Zone）
biz-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
    NIWSServerListClassName: com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList

ops-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
    # 指向其他注册中心需要自定义实现
```

对于多注册中心场景，更推荐的方式是使用 Spring Cloud Gateway 或自研方案，因为 Zuul + Eureka 的多集群方案较为复杂。

# 6. 服务路由映射器

`ServiceRouteMapper` 允许自定义 Eureka serviceId 到 Zuul 路由路径的映射规则：

```java
@Bean
public PatternServiceRouteMapper serviceRouteMapper() {
    // 正则匹配 serviceId 并转换为路由路径
    // 第一个参数：匹配 serviceId 的正则（捕获组）
    // 第二个参数：路由路径模板
    return new PatternServiceRouteMapper(
        "(?<name>^.+)-(?<version>v.+$)",
        "${version}/${name}"
    );
}
```

效果：

```
serviceId: user-service-v1
自动路由路径: /v1/user-service/**

serviceId: order-service-v2
自动路由路径: /v2/order-service/**
```

# 7. 路由信息 API

Zuul 暴露了 `/routes` 端点来查看当前生效的路由：

```bash
GET /actuator/routes
```

返回示例：

```json
{
    "/user/**": "user-service",
    "/order/**": "order-service",
    "/product/**": "product-service",
    "/external/**": "http://external-api.example.com"
}
```

## 7.1 路由详情端点

需要在 `application.yml` 中开启：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: routes, filters
  endpoint:
    routes:
      enabled: true
```

# 8. 完整配置示例

```yaml
server:
  port: 8080

spring:
  application:
    name: zuul-gateway

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

zuul:
  prefix: /api
  stripPrefix: true
  retryable: true

  sensitive-headers:
    - Cookie
    - Set-Cookie
    - Authorization

  ignored-services: "*"
  ignored-patterns: "/actuator/**"

  add-host-header: true
  add-proxy-headers: true

  routes:
    # 面向服务路由
    user-service:
      path: /user/**
      serviceId: user-service
      stripPrefix: false
      sensitive-headers:

    order-service:
      path: /order/**
      serviceId: order-service

    # URL 路由
    legacy-api:
      path: /legacy/**
      url: http://legacy-server:8080

    # Forward 路由
    health:
      path: /health
      url: forward:/actuator/health

# Hystrix 全局超时
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000

# Ribbon 全局配置
ribbon:
  ReadTimeout: 5000
  ConnectTimeout: 2000
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 1
  OkToRetryOnAllOperations: false
```
