
# 1. 总体流程图

Zuul 中一个 HTTP 请求的完整生命周期可以分为 8 个阶段：

```
                               ┌─────────────────┐
                               │   客户端发起请求   │
                               └────────┬────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 1. Tomcat 接收请求                                                │
  │    - 从连接池获取线程                                             │
  │    - 构建 HttpServletRequest/HttpServletResponse                 │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 2. Spring MVC 请求分发                                            │
  │    - DispatcherServlet 接收请求                                   │
  │    - ZuulHandlerMapping.lookupHandler(urlPath) 匹配路由            │
  │    - 路由匹配 → 交给 ZuulController.handleRequest()               │
  │    - 路由未匹配 → 404 Not Found                                   │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 3. ZuulController → ZuulServlet.service()                         │
  │    - 初始化 RequestContext（当前线程的 ThreadLocal）               │
  │    - 将原始 Request/Response 存入 RequestContext                  │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 4. Pre 阶段：执行所有 PreFilter（按 filterOrder 升序）              │
  │    典型工作：鉴权 → 限流 → 路由决定 → 参数处理                    │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 5. Route 阶段：执行所有 RouteFilter（按 filterOrder 升序）          │
  │    典型工作：通过 Ribbon 或 HttpClient 转发请求到后端服务          │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 6. Post 阶段：执行所有 PostFilter（按 filterOrder 升序）            │
  │    典型工作：添加统一响应头 → 记录日志 → 输出响应流 → 清理资源     │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 7. 任意阶段异常 → Error 阶段                                      │
  │    - 执行所有 ErrorFilter                                         │
  │    - SendErrorFilter 作为兜底，返回统一错误响应                    │
  └──────────────────────────────────────────────────────────────────┘
                                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ 8. 清理阶段                                                        │
  │    - RequestContext.getCurrentContext().unset()                   │
  │    - 释放 ThreadLocal 引用，防止内存泄漏                           │
  │    - Tomcat 线程归还线程池                                        │
  └──────────────────────────────────────────────────────────────────┘
```

# 2. 阶段一：Tomcat 接收请求

Zuul 基于 Servlet 容器（默认 Tomcat），所有请求先由 Tomcat 的 `CoyoteAdapter` 处理，构建标准的 `HttpServletRequest` 和 `HttpServletResponse` 对象，然后交给 Servlet 容器中的 Filter Chain 和 DispatcherServlet。

Tomcat 的线程池配置直接影响 Zuul 的并发能力：

```yaml
server:
  tomcat:
    threads:
      max: 200          # 最大工作线程数
      min-spare: 10     # 最小空闲线程数
    accept-count: 100    # 最大等待队列长度
    max-connections: 10000  # 最大连接数
    connection-timeout: 20000  # 连接超时（ms）
```

# 3. 阶段二：Spring MVC 请求分发

## 3.1 ZuulHandlerMapping 匹配过程

```java
// ZuulHandlerMapping 的核心匹配逻辑
@Override
protected Object lookupHandler(String urlPath, HttpServletRequest request)
        throws Exception {

    // 1. 检查是否在忽略路径列表中
    if (isIgnoredPath(urlPath, this.zuulProperties.getIgnoredPatterns())) {
        return null;  // 返回 null，交给 Spring MVC 默认处理
    }

    // 2. 启用 Zuul 请求上下文跟踪（用于 Sleuth 链路追踪）
    addRequestContext();

    // 3. 路由匹配
    RequestContext ctx = RequestContext.getCurrentContext();
    Route route = this.routeLocator.getMatchingRoute(urlPath);
    if (route != null) {
        // 匹配成功：交给 ZuulController 处理
        ctx.set("matchedRoute", route);
        return this.zuulController;  // ZuulController 实例
    } else {
        // 匹配失败：交给其他 HandlerMapping 处理
        return null;
    }
}
```

这一步的关键目标：**判断请求是否应该由 Zuul 处理**。判断依据是请求路径是否与任何一个已配置的路由规则匹配。

## 3.2 ZuulController 的转发

当 `ZuulHandlerMapping` 返回 `ZuulController` 后，Spring MVC 调用 `ZuulController.handleRequest()`：

```java
public class ZuulController extends ServletWrappingController {
    public ZuulController() {
        setServletClass(ZuulServlet.class);   // 包装 ZuulServlet
        setServletName("zuul");
        setSupportedMethods(null);            // 支持所有 HTTP 方法
    }

    @Override
    public ModelAndView handleRequest(HttpServletRequest request,
                                       HttpServletResponse response) {
        try {
            // 委托给 ZuulServlet.service() 处理
            return super.handleRequestInternal(request, response);
        } finally {
            // 确保清理 ThreadLocal（双重保险）
            RequestContext.getCurrentContext().unset();
        }
    }
}
```

`ZuulController` 继承自 `ServletWrappingController`，本质上是一个将 ZuulServlet 包装进 Spring MVC 处理链的适配器。

# 4. 阶段三：ZuulServlet 初始化上下文

```java
public class ZuulServlet extends HttpServlet {
    private ZuulRunner zuulRunner;

    @Override
    public void service(ServletRequest req, ServletResponse res) {
        // 1. 初始化 RequestContext（绑定到当前线程的 ThreadLocal）
        this.zuulRunner.init((HttpServletRequest) req, (HttpServletResponse) res);
        try {
            preRoute();   // Pre 阶段
            route();      // Route 阶段
            postRoute();  // Post 阶段
        } catch (Throwable e) {
            error(e);     // Error 阶段
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }
}
```

`ZuulRunner.init()` 的作用：

```java
public void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
    RequestContext ctx = RequestContext.getCurrentContext();
    // 绑定原始 Request 和 Response
    ctx.setRequest(servletRequest);
    ctx.setResponse(servletResponse);
    // 记录请求信息
    ctx.set("requestURI", servletRequest.getRequestURI());
    ctx.set("httpMethod", servletRequest.getMethod());
    // ...
}
```

# 5. 阶段四：Pre 阶段详解

Pre 阶段是请求进入后的第一个处理阶段，核心过滤器为 `PreDecorationFilter`（order=5）。

此阶段的执行流程：

```
ZuulServlet.preRoute()
    │
    ▼
FilterProcessor.runFilters("pre")
    │
    ├──→ ServletDetectionFilter     (-3)  检测请求来源
    ├──→ Servlet30WrapperFilter     (-2)  包装 Servlet 3.0 请求
    ├──→ FormBodyWrapperFilter      (-1)  包装表单请求体
    ├──→ DebugFilter                (1)   调试标记
    ├──→ PreDecorationFilter        (5)   ★ 路由决定
    ├──→ 用户自定义 PreFilter...     (10+)
    │
    ▼ (Pre 阶段完成)
```

`PreDecorationFilter` 决定了请求的路由目标：

```
请求 /user/profile/123
    │
    ▼  PreDecorationFilter
    │
    ├── 匹配到 Route{path="/user/**", location="user-service"}
    │
    ├── location 是 serviceId → 设置 ctx["serviceId"] = "user-service"
    │    ctx["requestURI"] = "/profile/123" (stripPrefix 后)
    │
    ├── location 是 URL → 设置 ctx.setRouteHost(new URL("http://..."))
    │
    └── 设置 ctx["proxy"] = "user-service" (路由 ID)
```

# 6. 阶段五：Route 阶段详解

Route 阶段执行实际的请求转发。核心逻辑是选择一个合适的 RouteFilter 发送请求到后端。

```
ZuulServlet.route()
    │
    ▼
FilterProcessor.runFilters("route")
    │
    ├── shouldFilter() = true for ctx.getRouteHost() == null && serviceId != null
    │   │
    │   ├──→ RibbonRoutingFilter (10)  → RibbonCommand → Hystrix → Ribbon → HTTP → 后端
    │   │
    ├── shouldFilter() = true for ctx.getRouteHost() != null
    │   │
    │   ├──→ SimpleHostRoutingFilter (100) → HttpClient → URL → 后端
    │   │
    ├── shouldFilter() = true for ctx["forward.to"] != null
    │   │
    │   └──→ SendForwardFilter (500) → RequestDispatcher.forward()
    │
    ▼ (Route 阶段完成)
```

## 6.1 RibbonRoutingFilter 详细流程

```
RibbonRoutingFilter.run()
    │
    ├─ 1. 从 RequestContext 获取 serviceId
    │     ctx.get("serviceId") → "user-service"
    │
    ├─ 2. 构建 RibbonCommand (Hystrix 包装的 Ribbon 请求)
    │     RibbonCommand command = ribbonCommandFactory.create(context)
    │
    ├─ 3. 执行命令
    │     │
    │     ├── 线程隔离模式:
    │     │   HystrixCommand.execute()
    │     │   → Hystrix 线程池中的线程执行
    │     │   → Ribbon LoadBalancer 选择服务实例
    │     │   → HttpClient 发起 HTTP 请求到选定实例
    │     │
    │     └── 信号量隔离模式:
    │         HystrixCommand.execute()
    │         → 当前 Tomcat 线程执行（需获取信号量许可）
    │         → Ribbon LoadBalancer 选择服务实例
    │         → HttpClient 发起 HTTP 请求到选定实例
    │
    ├─ 4. 获取响应
    │     ClientHttpResponse response = command.execute()
    │     ctx.setResponseStatusCode(response.getRawStatusCode())
    │     ctx.setResponseDataStream(response.getBody())
    │
    └─ 5. 异常降级
          Hystrix 熔断/超时/拒绝 → fallback → FallbackProvider 处理
```

## 6.2 SimpleHostRoutingFilter 详细流程

```
SimpleHostRoutingFilter.run()
    │
    ├─ 1. 从 RequestContext 获取 routeHost
    │     ctx.getRouteHost() → http://external-api.example.com:8080
    │
    ├─ 2. 拼接完整的后端 URL
    │     URL targetUrl = new URL(routeHost, requestURI)
    │     → http://external-api.example.com:8080/api/data
    │
    ├─ 3. 构建 HttpClient 请求
    │     - 复制原始请求的 Method、Headers、Body
    │     - 添加 zuulRequestHeaders（自定义请求头）
    │
    ├─ 4. 发送请求（阻塞 I/O）
    │     CloseableHttpResponse response = httpClient.execute(request)
    │
    └─ 5. 写入响应到 RequestContext
          ctx.setResponseDataStream(response.getEntity().getContent())
          ctx.setResponseStatusCode(response.getStatusLine().getStatusCode())
```

# 7. 阶段六：Post 阶段详解

Route 阶段完成后（无论成功或失败），Post 阶段的过滤器开始执行。

```
ZuulServlet.postRoute()
    │
    ▼
FilterProcessor.runFilters("post")
    │
    ├──→ 用户自定义 PostFilter...    (900-999) 日志、指标、响应头处理
    ├──→ SendResponseFilter         (1000)    ★ 输出响应流到客户端
    │
    ▼ (Post 阶段完成)
```

## 7.1 SendResponseFilter 源码分析

```java
public class SendResponseFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "post";
    }

    @Override
    public int filterOrder() {
        return 1000;  // 最后执行（数值足够大）
    }

    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        return context.getThrowable() == null       // 没有异常
            && context.getResponseDataStream() != null // 有响应流
            && context.sendZuulResponse();             // 允许发送响应
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletResponse servletResponse = context.getResponse();
        InputStream responseStream = context.getResponseDataStream();

        // 1. 设置响应头（来自 zuulResponseHeaders）
        addResponseHeaders();

        // 2. 设置 Content-Type
        if (servletResponse.getContentType() == null
                && context.getResponseContentType() != null) {
            servletResponse.setContentType(context.getResponseContentType());
        }

        // 3. 设置 Content-Length
        // ...

        // 4. 将响应流写入客户端输出流
        OutputStream outStream = servletResponse.getOutputStream();
        IOUtils.copy(responseStream, outStream);  // 阻塞 I/O 复制
        outStream.flush();

        return null;
    }
}
```

`SendResponseFilter` 的 `shouldFilter()` 有三个条件缺一不可：
- `context.getThrowable() == null`：没有异常（有异常时走 Error 阶段）
- `context.getResponseDataStream() != null`：后端返回了响应数据
- `context.sendZuulResponse()`：没有在 Pre 阶段被主动截断

# 8. 阶段七：Error 阶段详解

当 pre/route/post 任一阶段的过滤器抛出异常时，流程跳转到 Error 阶段。

```java
// ZuulServlet.service() 中的异常处理
try {
    preRoute();
    route();
    postRoute();
} catch (Throwable e) {
    error(e);  // 跳转到 Error 阶段
}
```

`error()` 方法会执行所有 `filterType()` 返回 `"error"` 的过滤器：

```
ZuulServlet.error(e)
    │
    ├─ 设置 ctx.set("error.status_code", ...)
    ├─ 设置 ctx.setThrowable(e)
    │
    ▼
FilterProcessor.runFilters("error")
    │
    ├──→ 用户自定义 ErrorFilter...   (-1)  记录异常日志、发送告警
    ├──→ SendErrorFilter            (0)   ★ 返回统一错误响应
    │
    ▼ (Error 阶段完成)
```

## 8.1 SendErrorFilter 源码分析

```java
public class SendErrorFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getThrowable() != null                          // 存在异常
            && !ctx.getBoolean("sendErrorFilter.ran", false);      // 未被执行过
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        HttpServletResponse response = ctx.getResponse();

        // 1. 提取异常信息
        Throwable throwable = ctx.getThrowable();
        int statusCode = extractStatusCode(throwable);

        // 2. 设置响应状态码
        response.setStatus(statusCode);

        // 3. 设置错误信息
        request.setAttribute("javax.servlet.error.status_code", statusCode);
        request.setAttribute("javax.servlet.error.exception", throwable);
        request.setAttribute("javax.servlet.error.message", throwable.getMessage());

        // 4. 使用 Tomcat 的 ErrorPage 机制渲染错误页面
        //    或者由自定义 ErrorFilter 输出 JSON 错误响应

        return null;
    }
}
```

## 8.2 异常传播链路

```
Pre 阶段某个 Filter.run()
    │
    ├─ 抛出异常
    │
    ▼
FilterProcessor.processZuulFilter()
    │
    ├─ catch (Throwable e)
    │   ├─ ctx.setThrowable(e)     ← 存入 RequestContext
    │   └─ throw e                  ← 重新抛出
    │
    ▼
FilterProcessor.runFilters()
    │
    ├─ catch (Throwable e)         ← 重新捕获
    │   └─ throw e
    │
    ▼
ZuulRunner.preRoute() / route() / postRoute()
    │
    ├─ catch (Throwable e)
    │   └─ throw e
    │
    ▼
ZuulServlet.service()
    │
    ├─ catch (Throwable e)
    │   └─ error(e)                ← 进入 Error 阶段
```

# 9. 阶段八：清理

无论请求处理成功还是失败，`finally` 块始终执行：

```java
finally {
    RequestContext.getCurrentContext().unset();
}
```

`unset()` 方法调用 `ThreadLocal.remove()` 清除当前线程的引用，防止：
- **内存泄漏**：`RequestContext` 持有 `HttpServletRequest`、`HttpServletResponse`、`InputStream` 等对象，如果不清理，这些对象将随线程存活
- **数据串扰**：线程被复用后，下一个请求可能读取到上一个请求的上下文数据
