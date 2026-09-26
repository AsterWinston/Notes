
# 1. ZuulFilter 抽象类

`ZuulFilter` 是所有过滤器的基类，继承自 `BaseFilter` 并实现 `IZuulFilter` 接口。

```java
package com.netflix.zuul;

public abstract class ZuulFilter implements IZuulFilter, Comparable<ZuulFilter> {

    // 核心生命周期方法
    abstract public String filterType();     // 过滤器类型
    abstract public int filterOrder();       // 执行顺序
    abstract public boolean shouldFilter();  // 是否执行
    abstract public Object run() throws ZuulException;  // 业务逻辑
}
```

## 1.1 IZuulFilter 接口

```java
public interface IZuulFilter {
    // 过滤器是否被禁用（保留接口，大部分实现返回 false）
    boolean isFilterDisabled();

    // 过滤器类型：pre / route / post / error
    String filterType();

    // 过滤器执行顺序，数字越小越先执行
    int filterOrder();

    // 过滤器名称，默认取类名
    String filterName();

    // 运行时判断是否执行此过滤器
    boolean shouldFilter();

    // 过滤器核心逻辑
    Object run() throws ZuulException;
}
```

## 1.2 默认方法实现

```java
public abstract class ZuulFilter implements IZuulFilter {

    // 默认不禁用
    @Override
    public boolean isFilterDisabled() {
        return false;
    }

    // 过滤器名称默认为类名
    @Override
    public String filterName() {
        return getClass().getSimpleName();
    }

    // 按 filterOrder 排序
    @Override
    public int compareTo(ZuulFilter o) {
        return Integer.compare(this.filterOrder(), o.filterOrder());
    }
}
```

`compareTo` 的实现使得 `FilterLoader.getFiltersByType()` 中可以通过 `Collections.sort()` 按 order 排序。

# 2. BaseFilter 类（Netflix 内部基类）

```java
public abstract class BaseFilter<T, Z> {
    // FilterProcessor 引用
    FilterProcessor filterProcessor;

    // 归档（Archives）信息：存储过滤器的历史统计数据
    private final ConcurrentHashMap<String, Object> filterExecutionSummary =
        new ConcurrentHashMap<>();

    // 静态上下文（Static Context）：过滤器级别的配置/状态
    private static ConcurrentHashMap<String, Object> staticContext =
        new ConcurrentHashMap<>();

    // Hook 方法：执行前的准备工作
    // 子类可覆盖来实现初始化逻辑
    protected void init() {}

    // Hook 方法：执行后的清理工作
    protected void cleanup() {}

    // 获取静态上下文
    public static ConcurrentHashMap<String, Object> getStaticContext() {
        return staticContext;
    }
}
```

`BaseFilter` 提供了过滤器的基础设施——如执行统计、静态上下文等。`ZuulFilter` 继承了这些能力。

# 3. 内置过滤器类层次结构

```
IZuulFilter (接口)
    ↑
ZuulFilter (抽象类)
    ↑
    ├── ServletDetectionFilter      pre   (-3)
    ├── Servlet30WrapperFilter      pre   (-2)
    ├── FormBodyWrapperFilter       pre   (-1)
    ├── DebugFilter                 pre   (1)
    ├── PreDecorationFilter         pre   (5)
    ├── RibbonRoutingFilter         route (10)
    ├── SimpleHostRoutingFilter     route (100)
    ├── SendForwardFilter           route (500)
    ├── LocationRewriteFilter       post  (900)
    ├── SendResponseFilter          post  (1000)
    └── SendErrorFilter             error (0)
```

# 4. 关键内置过滤器源码分析

## 4.1 PreDecorationFilter（路由装饰器）

作用：**在请求到达后端之前，确定路由目标并装饰请求上下文**。

```java
public class PreDecorationFilter extends ZuulFilter {

    private RouteLocator routeLocator;
    private ZuulProperties zuulProperties;
    private ProxyRequestHelper proxyRequestHelper;

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 5; }

    @Override
    public boolean shouldFilter() { return true; }  // 总是执行

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String requestURI = ctx.getRequest().getRequestURI();

        // 查找匹配的路由
        Route route = this.routeLocator.getMatchingRoute(requestURI);
        if (route != null) {
            // ★ 核心：设置代理信息
            String location = route.getLocation();

            // 设置路由路径（用于 stripPrefix 后的最终 URI）
            ctx.put(REQUEST_URI_KEY, route.getPath());

            // 设置路由 ID
            ctx.put(PROXY_KEY, route.getId());

            if (location != null) {
                if (location.startsWith(HTTP_SCHEME) || location.startsWith(HTTPS_SCHEME)) {
                    // === URL 路由 ===
                    URL url = getUrl(location);
                    ctx.setRouteHost(url);
                    // 设置剥离前缀后的 URI
                    ctx.put(REQUEST_URI_KEY,
                        strippedPath(requestURI, route.getPath(), location));
                } else {
                    // === serviceId 路由 ===
                    ctx.set(ServiceRouteMapper.SERVICE_ID_KEY, location);
                    ctx.setRouteHost(null);
                    ctx.addOriginResponseHeader("X-Zuul-ServiceId", location);
                }
            }

            // 处理敏感 Header
            if (route.getSensitiveHeaders() != null) {
                ctx.put("sensitiveHeaders", route.getSensitiveHeaders());
            }

            // 处理 retryable
            if (route.getRetryable() != null) {
                ctx.put("retryable", route.getRetryable());
            }
        } else {
            // 路由未匹配：记录日志，允许请求通过（可能由后续过滤器处理）
            if (log.isDebugEnabled()) {
                log.debug("No route found for uri: " + requestURI);
            }
        }
        return null;
    }
}
```

## 4.2 RibbonRoutingFilter（Ribbon 路由过滤器）

作用：**通过 Ribbon 负载均衡选择后端实例并转发请求**。

```java
public class RibbonRoutingFilter extends ZuulFilter {

    private final RibbonCommandFactory<?> ribbonCommandFactory;
    private final List<RibbonRequestCustomizer> requestCustomizers;

    @Override
    public String filterType() { return "route"; }

    @Override
    public int filterOrder() { return 10; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getRouteHost() == null        // 不是 URL 路由
            && ctx.get(SERVICE_ID_KEY) != null   // 是 serviceId 路由
            && ctx.sendZuulResponse();           // 允许继续处理
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();

        try {
            // 1. 创建 RibbonCommand（Hystrix 包装）
            RibbonCommand command = this.ribbonCommandFactory.create(context);

            // 2. 执行命令 —— 这里会：
            //    a. Hystrix 熔断检查
            //    b. Ribbon 负载均衡选择实例
            //    c. HttpClient 发起 HTTP 请求
            //    d. 获取响应
            ClientHttpResponse response = command.execute();

            // 3. 将响应写入 RequestContext
            context.set("zuulResponse", response);
            context.setResponseDataStream(response.getBody());

            // 4. 设置响应信息
            context.setResponseStatusCode(
                response.getRawStatusCode());
            context.setResponseContentType(
                response.getHeaders().getContentType().toString());

            // 5. 收集响应头
            for (Map.Entry<String, List<String>> entry :
                    response.getHeaders().entrySet()) {
                for (String value : entry.getValue()) {
                    context.addOriginResponseHeader(entry.getKey(), value);
                }
            }

        } catch (HystrixRuntimeException e) {
            // Hystrix 异常（熔断/超时/拒绝）→ 跳转 error 过滤器
            throw new ZuulException(e, 500,
                "RibbonRoutingFilter - Hystrix exception");
        } catch (Exception e) {
            throw new ZuulException(e, 500,
                "RibbonRoutingFilter - Unexpected exception");
        }
        return null;
    }
}
```

## 4.3 SimpleHostRoutingFilter（URL 路由过滤器）

作用：**通过 Apache HttpClient 直接将请求转发到指定 URL**。

```java
public class SimpleHostRoutingFilter extends ZuulFilter {

    // Apache HttpClient 实例（连接池管理）
    private final CloseableHttpClient httpClient;

    @Override
    public String filterType() { return "route"; }

    @Override
    public int filterOrder() { return 100; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getRouteHost() != null      // 是 URL 路由
            && ctx.sendZuulResponse();         // 允许继续处理
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();

        // 1. 构建目标 URL
        URL host = context.getRouteHost();
        String uri = (String) context.get(REQUEST_URI_KEY);
        String verb = getVerb(request);

        // 2. 构建 HttpClient 请求
        HttpClientRequest httpClientRequest =
            HttpClientRequest.create(host, uri, verb);

        // 3. 复制原始请求头
        copyRequestHeaders(request, httpClientRequest);

        // 4. 复制请求体
        copyRequestBody(request, httpClientRequest);

        // 5. 执行请求（阻塞）
        CloseableHttpResponse response = forward(httpClientRequest);

        // 6. 写入响应到 RequestContext
        setResponse(response);
        return null;
    }
}
```

## 4.4 SendResponseFilter（响应输出过滤器）

作用：**将后端返回的响应流写入客户端输出流**。

```java
public class SendResponseFilter extends ZuulFilter {

    @Override
    public String filterType() { return "post"; }

    @Override
    public int filterOrder() { return 1000; }

    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        return context.getThrowable() == null           // 无异常
            && context.getResponseDataStream() != null  // 有响应流
            && context.sendZuulResponse();              // 允许发送
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletResponse servletResponse = ctx.getResponse();
        InputStream responseStream = ctx.getResponseDataStream();

        // 1. 设置响应头
        addResponseHeaders();

        // 2. 设置状态码
        servletResponse.setStatus(ctx.getResponseStatusCode());

        // 3. Content-Length 特殊处理
        // 如果是 GZIP 压缩，需要移除原始 Content-Length
        // 因为解压后的长度与压缩长度不同

        // 4. 流复制
        OutputStream outStream = servletResponse.getOutputStream();
        IOUtils.copy(responseStream, outStream);
        outStream.flush();

        return null;
    }
}
```

## 4.5 SendErrorFilter（错误响应过滤器）

```java
public class SendErrorFilter extends ZuulFilter {

    @Override
    public String filterType() { return "error"; }

    @Override
    public int filterOrder() { return 0; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getThrowable() != null                          // 存在异常
            && !ctx.getBoolean("sendErrorFilter.ran", false);      // 未执行过
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        // 标记已执行，避免因重试或重复调用导致死循环
        ctx.set("sendErrorFilter.ran", true);

        // 设置错误属性到 Request（供 Tomcat ErrorPage 机制使用）
        setErrorAttributes(ctx);

        // 如果配置了 error.path，转发到该路径
        // 否则使用 Tomcat 默认错误页面
        ctx.getResponse().sendError(ctx.getResponseStatusCode());
        return null;
    }
}
```

# 5. FilterLoader 生命周期管理

## 5.1 初始化流程

```java
public class FilterLoader {
    // 类加载时自动初始化
    static {
        instance = new FilterLoader();

        // 从系统属性读取配置
        zuulFilterRoot = System.getProperty("zuul.filters.root", "");
        zuulFilterRefreshInterval =
            Integer.getInteger("zuul.filters.interval", -1);
    }
}
```

## 5.2 Groovy 过滤器编译

```java
private Class<?> compileGroovy(String className, String sourceCode) {
    GroovyClassLoader loader = new GroovyClassLoader(
        Thread.currentThread().getContextClassLoader());
    Class<?> clazz = loader.parseClass(sourceCode);
    return clazz;
}
```

Groovy 过滤器编译使用了 `GroovyClassLoader`，源码在内存中编译为 class。编译结果会被缓存在 JVM 的永久代（JDK 7）或元空间（JDK 8+）中。

## 5.3 过滤器卸载机制

`FilterLoader` 没有提供显式的卸载机制。如果需要替换过滤器：

1. 修改 Groovy 源码文件
2. `FilterLoader` 检测到文件修改后重新编译
3. `FilterRegistry.put()` 使用 `putIfAbsent`，因此在 Spring Cloud 环境下，**已存在同名的过滤器不会被覆盖**

这意味着在 Spring Cloud 中通过 `@Component` 注册的 Java 过滤器**无法**通过 Groovy 热更新替换（`putIfAbsent` 特性导致）。

# 6. RequestContext 的 ThreadLocal 机制

```java
public class RequestContext extends ConcurrentHashMap<String, Object> {

    // ★ 核心：每个线程独立的上下文
    protected static final ThreadLocal<? extends RequestContext> threadLocal =
        new ThreadLocal<RequestContext>() {
            @Override
            protected RequestContext initialValue() {
                try {
                    return contextClass.getDeclaredConstructor().newInstance();
                } catch (Throwable e) {
                    throw new RuntimeException(e);
                }
            }
        };

    // 获取当前线程的上下文
    public static RequestContext getCurrentContext() {
        if (testContext != null) {
            return testContext;  // 单元测试模式
        }
        RequestContext context = threadLocal.get();
        return context;
    }

    // 释放当前线程的上下文（防止内存泄漏）
    public void unset() {
        threadLocal.remove();
    }
}
```

## 6.1 ThreadLocal 的两大问题

1. **异步调用上下文丢失**：如果在过滤器中启动异步任务，子线程无法访问父线程的 `RequestContext`
2. **内存泄漏风险**：线程池（Tomcat）复用线程时，如果忘记 `unset()`，旧的 `RequestContext` 会一直存活

## 6.2 Spring Cloud 的双重清理

```java
// ZuulController.handleRequest()
public ModelAndView handleRequest(HttpServletRequest request,
                                   HttpServletResponse response) {
    try {
        return super.handleRequestInternal(request, response);
    } finally {
        RequestContext.getCurrentContext().unset();  // ← 兜底清理
    }
}
```

Spring Cloud 在 `ZuulController.handleRequest()` 的 `finally` 块中也加入了 `unset()`，与 `ZuulServlet.service()` 的 `finally` 形成双重保障。

# 7. 过滤器执行全景总结

```
┌──────────────────────────────────────────────────────┐
│                    应用启动阶段                         │
│                                                      │
│  @Component 过滤器 → Spring 扫描 → 注入到             │
│  ZuulServerAutoConfiguration.filters Map              │
│       │                                              │
│       ├──→ @PostConstruct: configureFilters()        │
│       │    → FilterRegistry.instance().put(          │
│       │        filterName, filterInstance)           │
│       │                                              │
│       └──→ Groovy 动态过滤器（如果配置）：             │
│            FilterLoader 定期扫描文件 → 编译 → put     │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│                    请求处理阶段                         │
│                                                      │
│  1. ZuulServlet.service()                            │
│  2. RequestContext 绑定 ThreadLocal                  │
│  3. preRoute()  → FilterProcessor.runFilters("pre")  │
│       └── FilterLoader.getFiltersByType("pre")       │
│           └── FilterRegistry 中筛选 + 排序            │
│  4. route()     → FilterProcessor.runFilters("route")│
│  5. postRoute() → FilterProcessor.runFilters("post") │
│  6. error()     → FilterProcessor.runFilters("error")│
│  7. RequestContext.unset()                           │
└──────────────────────────────────────────────────────┘
```
