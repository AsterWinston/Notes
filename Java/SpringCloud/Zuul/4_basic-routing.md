
# 1. 路由概述

Zuul 的路由机制是其最核心的功能。路由决定了"一个外部请求应该被转发到哪个后端服务"。Spring Cloud Zuul 提供三种路由方式：

| 路由方式 | 配置字段 | 适用场景 |
|---------|---------|---------|
| URL 路由 | `url` | 后端服务为固定 IP/域名，无注册中心 |
| serviceId 路由 | `serviceId` | 通过注册中心（Eureka）发现服务 |
| 自动路由 | 无需配置 | Eureka 中注册的服务自动创建路由规则 |
| Forward 路由 | `url: forward:/xxx` | 转发到 Zuul 自身的本地接口 |

# 2. URL 路由（物理路由）

URL 路由直接将请求转发到指定的物理地址，不依赖服务注册中心。

## 2.1 单节点 URL 路由

```yaml
zuul:
  routes:
    legacy-api:
      path: /legacy/**
      url: http://legacy-server.example.com:8080
```

请求示例：

```
客户端请求:  GET http://gateway:8080/legacy/api/v1/orders

Zuul 转发:  GET http://legacy-server.example.com:8080/api/v1/orders

（注意：/legacy 前缀默认被剥离，因为 stripPrefix 默认为 true）
```

## 2.2 URL 路由 + Ribbon 负载均衡

当 URL 路由需要负载均衡时，可以通过配置 Ribbon 服务列表来实现：

```yaml
zuul:
  routes:
    legacy-api:
      path: /legacy/**
      serviceId: legacy-service  # 通过 serviceId 间接使用 Ribbon

# 关闭 Eureka（Ribbon 不从注册中心获取服务列表）
ribbon:
  eureka:
    enabled: false

# 手动指定 Ribbon 的服务列表
legacy-service:
  ribbon:
    listOfServers: http://server1:8080,http://server2:8080
    ServerListRefreshInterval: 5000  # 刷新间隔（毫秒）
```

## 2.3 URL 路由 + Hystrix

通过 `url` 直接配置的 URL 路由**不会**经过 Hystrix。如果需要对 URL 路由使用 Hystrix，需要改用 `serviceId` 方式并在 Ribbon 中指定 `listOfServers`：

```yaml
zuul:
  routes:
    legacy-api:
      path: /legacy/**
      serviceId: legacy-service
      stripPrefix: false

legacy-service:
  ribbon:
    listOfServers: http://server1:8080,http://server2:8080
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList

hystrix:
  command:
    legacy-service:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000
```

## 2.4 URL 路由的底层实现

URL 路由的底层通过 `SimpleHostRoutingFilter` 执行：

```java
public class SimpleHostRoutingFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "route";  // 属于 route 类型过滤器
    }

    @Override
    public int filterOrder() {
        return 100;  // RibbonRoutingFilter(order=10) 之后执行
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();

        // 从 RequestContext 获取目标 URL
        String uri = request.getRequestURI();
        URL host = context.getRouteHost();
        URL targetUrl = new URL(host, uri);

        // 使用 Apache HttpClient 发起请求
        CloseableHttpClient httpClient = getHttpClient();
        HttpRequest httpRequest = buildHttpRequest(request, targetUrl);

        // 执行请求并获取响应
        CloseableHttpResponse response = httpClient.execute(
            (HttpUriRequest) httpRequest);
        // 将响应写入 RequestContext
        context.setResponseDataStream(response.getEntity().getContent());

        return null;
    }
}
```

核心类：`SimpleHostRoutingFilter` 位于 `org.springframework.cloud.netflix.zuul.filters.route` 包。

# 3. serviceId 路由（面向服务路由）

serviceId 路由依赖服务注册中心，Zuul 通过 Ribbon 实现负载均衡和 Hystrix 熔断。

## 3.1 基本配置

```yaml
zuul:
  routes:
    user-service:
      path: /user/**
      serviceId: user-service
```

spring.application.name 与 serviceId 之间的关系：

```
Eureka 注册名: USER-SERVICE (默认大写)
Zuul 配置:     serviceId: user-service (不区分大小写)
```

## 3.2 简化写法

当 `path` 的 Map Key 与 `serviceId` 相同时，可以省略 `serviceId`：

```yaml
zuul:
  routes:
    user-service:
      path: /user/**
    # 等价于:
    # user-service:
    #   path: /user/**
    #   serviceId: user-service

    order-service:
      path: /order/**
```

## 3.3 多路径映射到同一服务

```yaml
zuul:
  routes:
    user-api-v1:
      path: /api/v1/user/**
      serviceId: user-service
    user-api-v2:
      path: /api/v2/user/**
      serviceId: user-service
    user-api-v3:
      path: /api/v3/user/**
      serviceId: user-service
```

## 3.4 serviceId 路由的底层实现

serviceId 路由通过 `RibbonRoutingFilter` 执行：

```java
public class RibbonRoutingFilter extends ZuulFilter {
    // Ribbon 命令工厂，封装 Hystrix
    private final RibbonCommandFactory<?> ribbonCommandFactory;

    @Override
    public String filterType() {
        return "route";
    }

    @Override
    public int filterOrder() {
        return 10;  // 在 SimpleHostRoutingFilter(100) 之前执行
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        // 构建 RibbonCommand（Hystrix 命令）
        RibbonCommand command = this.ribbonCommandFactory.create(context);
        // 执行命令（含 Hystrix 熔断和 Ribbon 负载均衡）
        ClientHttpResponse response = command.execute();
        // 将响应流写入 RequestContext
        context.setResponseDataStream(response.getBody());
        return null;
    }
}
```

`RibbonRoutingFilter` 的 `filterOrder()` 为 10，`SimpleHostRoutingFilter` 为 100，这意味着**对于同一个请求，Ribbon 路由优先于 URL 路由**。

# 4. Forward 路由（本地转发）

Forward 路由将请求转发到 Zuul 网关自身的 Controller 端点，适用于需要在网关层直接返回数据（如聚合接口、健康检查等）的场景。

```yaml
zuul:
  routes:
    # 将 /health-check 转发到网关本地的 /actuator/health 端点
    health:
      path: /health-check
      url: forward:/actuator/health
```

Forward 路由通过 `SendForwardFilter` 实现：

```java
public class SendForwardFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "route";
    }

    @Override
    public int filterOrder() {
        return 500;  // 在 Ribbon 和 SimpleHost 之后执行
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.containsKey("forward.to");  // 存在 forward.to 时才执行
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String forwardUri = (String) ctx.get("forward.to");
        // 使用 RequestDispatcher 内部转发
        RequestDispatcher dispatcher = ctx.getRequest()
            .getRequestDispatcher(forwardUri);
        dispatcher.forward(ctx.getRequest(), ctx.getResponse());
        return null;
    }
}
```

# 5. 自动路由

当 Eureka 中注册了服务，Zuul 会自动创建路由规则（可在 `/actuator/routes` 中查看）。

## 5.1 自动路由规则

```yaml
zuul:
  # 忽略指定的 serviceId，不为其创建自动路由
  ignored-services: "config-server,monitor-service"

  # 忽略所有自动路由（仅使用显式配置的路由）
  # ignored-services: "*"

  # 忽略指定路径的请求（匹配的请求直接由 Tomcat 处理，不进入 Zuul）
  ignored-patterns: "/actuator/**,/static/**"
```

## 5.2 自动路由的生成源码

在 `ZuulServerAutoConfiguration` 中，通过 `routeLocator()` 方法创建路由定位器：

```java
@Bean
@ConditionalOnMissingBean(CompositeRouteLocator.class)
public CompositeRouteLocator primaryRouteLocator(
        Collection<RouteLocator> routeLocators) {
    return new CompositeRouteLocator(routeLocators);
}
```

`CompositeRouteLocator` 组合了以下 `RouteLocator`：

1. `SimpleRouteLocator`：解析 `zuul.routes.*` 中配置的路由
2. `DiscoveryClientRouteLocator`：从 Eureka 服务列表生成路由
3. 用户自定义的 `RouteLocator` Bean

# 6. 路由匹配规则

## 6.1 AntPathMatcher

Zuul 使用 Spring 的 `AntPathMatcher` 进行路径匹配：

| 模式 | 说明 | 匹配 |
|------|------|------|
| `*` | 匹配单层路径中的一个或多个字符（不含 `/`） | `/user/a` 匹配 `/user/*` |
| `**` | 匹配多层路径 | `/user/a/b/c` 匹配 `/user/**` |
| `?` | 匹配单个字符 | `/user/a` 匹配 `/user/?` |

## 6.2 匹配过程源码

在 `SimpleRouteLocator` 中：

```java
// SimpleRouteLocator.getMatchingRoute()
protected ZuulRoute getZuulRoute(String adjustedPath) {
    if (!matchesIgnoredPatterns(adjustedPath)) {
        for (Entry<String, ZuulRoute> entry : this.routes.get().entrySet()) {
            ZuulRoute route = entry.getValue();
            String pattern = route.getPath();
            if (this.pathMatcher.match(pattern, adjustedPath)) {
                return route;
            }
        }
    }
    return null;
}
```

`routes` 是一个 `LinkedHashMap`，保持配置时的插入顺序。匹配时**按配置顺序遍历**，第一个匹配成功的路由即为最终路由。

# 7. stripPrefix 详解

`stripPrefix` 控制是否在转发前删除匹配的前缀路径。

## 7.1 默认行为

```yaml
zuul:
  routes:
    user-service:
      path: /user/**
      serviceId: user-service
      # stripPrefix 默认为 true
```

**stripPrefix = true（默认）**：

```
客户端请求:  /user/profile/123
Zuul 匹配:   /user/** → user-service
转发到后端:  /profile/123          ← /user 前缀被剥除
```

**stripPrefix = false**：

```
客户端请求:  /user/profile/123
Zuul 匹配:   /user/** → user-service
转发到后端:  /user/profile/123     ← /user 前缀保留
```

## 7.2 全局 stripPrefix

```yaml
zuul:
  stripPrefix: false  # 全局关闭前缀剥离
  routes:
    user-service:
      path: /user/**
      serviceId: user-service
      # 此路由继承全局配置 stripPrefix = false
```

## 7.3 源码实现

`stripPrefix` 逻辑在 `PreDecorationFilter` 中实现：

```java
// PreDecorationFilter.run()
public Object run() {
    RequestContext ctx = RequestContext.getCurrentContext();
    // ...
    if (route != null) {
        String location = route.getLocation();
        // 如果配置了前缀且需要剥离
        if (location != null && zuulProperties.isStripPrefix()) {
            String path = getPath(ctx.getRequest());
            String strippedPath = stripPrefix(path, route.getPath());
            ctx.put(REQUEST_URI_KEY, strippedPath);
        }
    }
    // ...
}

// 剥离前缀
private String stripPrefix(String path, String prefix) {
    // 将 Ant 风格的模式转换为路径前缀
    // 例如 /user/** → /user
    String normalizedPrefix = prefix.replaceAll("/\\*\\*$", "");
    if (!"/".equals(normalizedPrefix)) {
        path = path.replaceFirst(normalizedPrefix, "");
    }
    return path;
}
```

# 8. 路由配置优先级

当多个路由规则都能匹配同一个请求时，匹配顺序由 `ZuulRoute` 的位置决定：

```yaml
zuul:
  routes:
    user-one:
      path: /user/v1/**
      serviceId: user-service-v1
    user-two:              # 这个规则会匹配 /user/v1/list
      path: /user/**
      serviceId: user-service-v2
```

以上配置中，`/user/v1/list` 会匹配到 `user-one`，因为 `user-one` 先配置且也匹配。如果交换配置顺序，则 `user-two` 先匹配。

`LinkedHashMap` 保证的插入顺序决定了匹配优先级——**先定义的先匹配**。

# 9. 忽略路由

```yaml
zuul:
  # 忽略路径：匹配的请求不进入 Zuul 处理，由 Tomcat 直接响应
  ignored-patterns:
    - /actuator/**
    - /static/**
    - /public/**

  # 忽略服务：不为此 serviceId 创建自动路由
  ignored-services:
    - config-server
    - internal-service
```

`ignored-patterns` 在请求进入 `ZuulHandlerMapping` 之前就被过滤：

```java
protected Object lookupHandler(String urlPath, HttpServletRequest request) {
    // ...
    if (isIgnoredPath(urlPath)) {
        return null;  // 返回 null，MVC 不会将此请求交给 Zuul 处理
    }
    // ...
}
```
