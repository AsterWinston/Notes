
# 1. 类结构总览

`ZuulServlet` 是 Zuul 1.x 的请求处理总入口，位于 `com.netflix.zuul` 包中。它继承自 `javax.servlet.http.HttpServlet`，实现了标准的 Servlet 生命周期方法。

```java
package com.netflix.zuul;

public class ZuulServlet extends HttpServlet {
    private static final long serialVersionUID = -3374242278843351500L;
    private ZuulRunner zuulRunner;

    // 生命周期方法
    @Override public void init(ServletConfig config) throws ServletException { }
    @Override public void service(ServletRequest req, ServletResponse res) { }

    // 阶段方法
    void preRoute() throws ZuulException { }
    void route() throws ZuulException { }
    void postRoute() throws ZuulException { }
    void error(Throwable e) { }

    // Spring Cloud 扩展（在 cloud 包中覆盖）
    // - 添加 Servlet 3.0 异步支持
    // - 可选地设置 Content-Length 头
}
```

## 1.1 与 Spring Cloud ZuulServlet 的关系

Spring Cloud Netflix 在 `org.springframework.cloud.netflix.zuul.web` 包中提供了 `ZuulServlet` 的子类，类名同为 `ZuulServlet`：

```java
package org.springframework.cloud.netflix.zuul.web;

public class ZuulServlet extends com.netflix.zuul.ZuulServlet {
    // Spring Cloud 扩展：支持异步处理、Content-Length 设置等
}
```

本文以 Netflix 原版 `ZuulServlet` 为分析对象，核心逻辑两者一致。

# 2. init() 方法分析

```java
@Override
public void init(ServletConfig config) throws ServletException {
    super.init(config);

    // 读取 web.xml 中的 buffer-requests 参数
    String bufferReqsStr = config.getInitParameter("buffer-requests");
    boolean bufferReqs = bufferReqsStr != null
        && Boolean.parseBoolean(bufferReqsStr);

    // 创建 ZuulRunner 实例
    // bufferReqs 传入：true 表示请求体需要缓存以便多次读取
    zuulRunner = new ZuulRunner(bufferReqs);
}
```

## 2.1 bufferReqs 参数的作用

`bufferReqs = true` 时，请求体的内容会被缓存在 `RequestContext` 中，允许后续过滤器多次读取。默认值为 `false`（Spring Cloud 未显式设置此参数）。

这个参数影响 `FormBodyWrapperFilter` 的行为，该过滤器会包装 `HttpServletRequest` 以支持多次读取表单体。

## 2.2 ZuulRunner 初始化

```java
public class ZuulRunner {
    private boolean bufferRequests;

    public ZuulRunner(boolean bufferRequests) {
        this.bufferRequests = bufferRequests;
    }

    public void init(HttpServletRequest servletRequest,
                     HttpServletResponse servletResponse) {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.setRequest(servletRequest);
        ctx.setResponse(servletResponse);

        // 如果启用了请求缓冲，将请求体读入 RequestContext
        if (bufferRequests) {
            ctx.set("requestBody", readRequestBody(servletRequest));
        }
    }
}
```

ZuulRunner 是 FilterProcessor 的外观（Facade），提供简化的调用接口。`init()` 方法的核心职责是将 Servlet 原生的 Request/Response 绑定到当前线程的 `RequestContext` 中。

# 3. service() 方法核心流程

```java
@Override
public void service(ServletRequest servletRequest, ServletResponse servletResponse)
        throws ServletException, IOException {

    try {
        // Step 1: 初始化上下文
        init((HttpServletRequest) servletRequest,
             (HttpServletResponse) servletResponse);

        // Step 2: 标记"已进入 ZuulServlet"（防止重复处理）
        RequestContext context = RequestContext.getCurrentContext();
        context.setZuulEngineRan();  // 设置 zuulEngineRan = true

        // Step 3: Pre 阶段
        try {
            preRoute();
        } catch (ZuulException e) {
            error(e);
            postRoute();
            return;  // ← 注意：Pre 阶段异常直接返回，不执行 Route
        }

        // Step 4: Route 阶段
        try {
            route();
        } catch (ZuulException e) {
            error(e);
            postRoute();
            return;  // ← Route 阶段异常也不执行后续 Post
        }

        // Step 5: Post 阶段（只有 Pre + Route 都成功才会执行到这里）
        try {
            postRoute();
        } catch (ZuulException e) {
            error(e);  // Post 异常仍执行 error
            return;
        }

    } catch (Throwable e) {
        // 兜底异常处理
        error(new ZuulException(e, 500,
            "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
    } finally {
        // Step 6: 清理 ThreadLocal
        RequestContext.getCurrentContext().unset();
    }
}
```

## 3.1 异常处理路径分析

```
正常路径：
  preRoute() 成功 → route() 成功 → postRoute() 成功

Pre 阶段异常路径：
  preRoute() 抛出异常 → error() → postRoute() → return（不执行 route）

Route 阶段异常路径：
  preRoute() 成功 → route() 抛出异常 → error() → postRoute() → return

Post 阶段异常路径：
  preRoute() 成功 → route() 成功 → postRoute() 抛出异常 → error() → return

完全兜底路径：
  service() 主 try 块中未捕获异常 → 外层 catch → error(new ZuulException(...))
```

核心设计原则：
1. **Pre 失败** → 不执行 Route → 执行 Error + Post
2. **Route 失败** → 不执行正常 Post → 执行 Error + Post
3. **Post 失败** → 只执行 Error
4. Post 阶段的 `SendResponseFilter` 在 error 路径下也会执行，以便将错误响应发送给客户端

# 4. preRoute() 方法

```java
void preRoute() throws ZuulException {
    // 委托给 FilterProcessor 执行所有 "pre" 类型过滤器
    zuulRunner.preRoute();
}
```

`ZuulRunner.preRoute()` 内部：

```java
public void preRoute() throws ZuulException {
    FilterProcessor.getInstance().preRoute();
}
```

`FilterProcessor.preRoute()` 内部：

```java
public void preRoute() throws ZuulException {
    try {
        runFilters("pre");  // 执行所有 filterType = "pre" 的过滤器
    } catch (Throwable e) {
        if (e instanceof ZuulException) {
            throw (ZuulException) e;
        }
        // 包装为非 ZuulException 的统一异常
        throw new ZuulException(e, 500, e.getMessage());
    }
}
```

# 5. route() 方法

```java
void route() throws ZuulException {
    zuulRunner.route();
}
```

`FilterProcessor.route()` 同理，执行所有 `filterType = "route"` 的过滤器。

值得注意的是 Route 阶段的设计：同一个请求**通常只有一个 RouteFilter 实际发起 HTTP 请求**，其他 RouteFilter 通过 `shouldFilter()` 返回 `false` 来跳过。

```java
// RibbonRoutingFilter.shouldFilter()
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    return ctx.getRouteHost() == null        // URL 路由未设置 routeHost
        && ctx.get("serviceId") != null      // 但设置了 serviceId
        && ctx.sendZuulResponse();           // 且允许发送响应
}

// SimpleHostRoutingFilter.shouldFilter()
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    return ctx.getRouteHost() != null         // URL 路由已设置 routeHost
        && ctx.sendZuulResponse();            // 且允许发送响应
}

// SendForwardFilter.shouldFilter()
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    return ctx.containsKey("forward.to");     // 设置了 forward.to
}
```

这三个 RouteFilter 通过 `shouldFilter()` 的互斥条件确保每次请求只有一个会执行。

# 6. postRoute() 方法

```java
void postRoute() throws ZuulException {
    zuulRunner.postRoute();
}
```

Post 阶段的关键过滤器：

| Order | 过滤器 | 作用 |
|-------|--------|------|
| 900 | `LocationRewriteFilter`（Spring Cloud 添加） | 处理重定向 URL 改写 |
| 1000 | `SendResponseFilter` | 将后端响应流写入客户端 |
| 1100+ | 用户自定义 PostFilter | 日志、监控等 |

`SendResponseFilter` 是 Post 阶段的最后一道工序——将后端服务的响应数据（存储在 `RequestContext.responseDataStream` 中）复制到客户端的 `HttpServletResponse.getOutputStream()`。

# 7. error() 方法

```java
void error(Throwable e) {
    RequestContext.getCurrentContext().setThrowable(e);
    zuulRunner.error();
}
```

`zuulRunner.error()` → `FilterProcessor.error()`：

```java
public void error() {
    try {
        runFilters("error");  // 执行所有 filterType = "error" 的过滤器
    } catch (Throwable e) {
        // 错误处理阶段的异常：极端情况，记录日志并吞掉
    }
}
```

Error 阶段的设计亮点：
1. Error 过滤器执行时的异常**不会**向上抛出（catch 并吞掉），避免无限递归
2. `RequestContext.getThrowable()` 保存原始异常，供 Error 过滤器读取
3. `SendErrorFilter` 通过检查 `ctx.getBoolean("sendErrorFilter.ran", false)` 来避免重复执行

# 8. 线程模型深入分析

ZuulServlet 通过 Servlet 容器（Tomcat）运行，每个请求绑定一个工作线程。关键影响：

## 8.1 ThreadLocal 的使用

```java
// RequestContext 中的 ThreadLocal
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

public static RequestContext getCurrentContext() {
    if (testContext != null) return testContext;  // 测试模式
    RequestContext context = threadLocal.get();
    return context;
}
```

每个 Tomcat 线程拥有独立的 `RequestContext` 实例，保证了并发安全。但这也意味着：
- **同一个请求的所有过滤器必须在同一个线程中执行**
- 如果在过滤器中使用异步操作（如 `CompletableFuture`），需要注意 `ThreadLocal` 的上下文传递
- `finally` 块中的 `unset()` 是防止内存泄漏的**最后防线**

## 8.2 阻塞点分析

```java
// Pre 阶段：通常轻量级，阻塞时间短
preRoute();    // 鉴权、限流、路由决定（通常 < 10ms）

// Route 阶段：主要阻塞点
route();       // ↓ 阻塞等待后端响应（可能达到秒级）
               //   HttpClient 的 I/O 操作阻塞当前 Tomcat 线程
               //   直到收到完整响应或超时

// Post 阶段：轻量级
postRoute();   // 输出响应、记录日志（通常 < 10ms）
```

这就是 Zuul 1.x 阻塞模型的本质：**Tomcat 线程在 route() 阶段被阻塞，等待后端响应**。高并发时，线程池很快耗尽。

## 8.3 信号量隔离 vs 线程隔离的影响

```yaml
# 信号量隔离（默认）
zuul:
  ribbon-isolation-strategy: SEMAPHORE
  semaphore:
    max-semaphores: 100

# 线程隔离
zuul:
  ribbon-isolation-strategy: THREAD
  thread-pool:
    use-separate-thread-pools: true
    thread-pool-key-prefix: zuul-
```

信号量隔离下，`RibbonRoutingFilter` 在当前 Tomcat 线程中执行 HTTP 调用，阻塞当前线程。线程隔离下，HTTP 调用在 Hystrix 线程池中执行，Tomcat 线程可以释放。

# 9. ZuulServlet 的 Spring Cloud 扩展

Spring Cloud 对 ZuulServlet 做了几个关键扩展：

## 9.1 异步处理支持

```java
// Spring Cloud 扩展的 ZuulServlet
public class ZuulServlet extends com.netflix.zuul.ZuulServlet {

    @Override
    public void service(ServletRequest servletRequest,
                        ServletResponse servletResponse) {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        // 支持 Servlet 3.0 异步
        if (request.isAsyncSupported()) {
            super.service(servletRequest, servletResponse);
        } else {
            super.service(servletRequest, servletResponse);
        }
    }
}
```

## 9.2 Content-Length 处理

Spring Cloud 在 Post 阶段自动处理 `Content-Length` 头的设置，避免客户端因缺少该头而卡住：

```java
// SendResponseFilter 中的 Content-Length 处理
if (context.getOriginContentLength() != null) {
    servletResponse.setContentLength(
        context.getOriginContentLength().intValue());
} else if (context.getResponseContentLength() != null) {
    servletResponse.setContentLength(
        context.getResponseContentLength().intValue());
}
```
