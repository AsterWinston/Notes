
# 1. 过滤器概述

Zuul 的核心是**过滤器链（Filter Chain）**。所有请求的处理逻辑——鉴权、限流、路由、日志等——都通过过滤器实现。Zuul 的设计哲学是：**一切皆过滤器**。

```java
public abstract class ZuulFilter implements IZuulFilter, Comparable<ZuulFilter> {

    // 1. 过滤器类型：pre / route / post / error
    public abstract String filterType();

    // 2. 过滤器执行顺序（数字越小越先执行）
    public abstract int filterOrder();

    // 3. 是否执行此过滤器（运行时判断）
    public abstract boolean shouldFilter();

    // 4. 过滤器业务逻辑
    public abstract Object run() throws ZuulException;

    // 5. 过滤器名称
    public String filterName() { return getClass().getSimpleName(); }
}
```

# 2. 四种过滤器类型

```
HTTP 请求
    │
    ▼
┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│  PreFilter │ ──→ │ RouteFilter│ ──→ │PostFilter  │ ──→ │  返回响应   │
│  (前置过滤) │     │  (路由过滤) │     │ (后置过滤)  │     │            │
└────────────┘     └────────────┘     └────────────┘     └────────────┘
    │    │               │     │            │    │
    │    └───────────────┼─────┼────────────┼────┘
    │                    │     │            │
    │         ┌──────────┼─────┼────────────┘
    │         │          │     │
    ▼         ▼          ▼     ▼
┌─────────────────────────────────────────┐
│           ErrorFilter (异常处理)          │
│  任何阶段抛出的异常都会被 ErrorFilter 捕获  │
└─────────────────────────────────────────┘
```

## 2.1 Pre 过滤器（前置）

**执行时机**：请求到达 Zuul 后、路由转发之前。

**典型用途**：
- 身份认证与鉴权
- 请求日志记录
- 限流计数
- 请求参数校验
- 请求头注入（如 TraceId、Auth Token）
- 请求体解析与修改

```java
public abstract class PreFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }
}
```

## 2.2 Route 过滤器（路由）

**执行时机**：Pre 过滤器执行完毕之后。

**典型用途**：
- 将请求转发到后端服务（核心功能）
- Ribbon 负载均衡选择服务实例
- HTTP 客户端发起请求
- WebSocket 代理

**注意**：Route 阶段的过滤器通常只有一个会实际发起请求。Zuul 内置的 `RibbonRoutingFilter`（order=10）和 `SimpleHostRoutingFilter`（order=100）会根据路由类型选择执行。

```java
public abstract class RouteFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "route";
    }
}
```

## 2.3 Post 过滤器（后置）

**执行时机**：Route 过滤器执行完毕、响应返回之后。

**典型用途**：
- 给响应添加标准 HTTP 头（如 CORS 头、Server 头）
- 收集响应统计信息（耗时、状态码）
- 响应内容转换（如压缩、加密）
- 审计日志

```java
public abstract class PostFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "post";
    }
}
```

## 2.4 Error 过滤器（错误处理）

**执行时机**：任何一个阶段（pre/route/post）的过滤器抛出异常时。

**特殊机制**：Error 过滤器分为两种——
- **Type 级别的 Error 过滤器**：`filterType()` 返回 `"error"`，仅当请求处理出错时执行
- **Normal 不可执行时的兜底**：一个特殊的 error 过滤器（`SendErrorFilter`）用于返回错误响应

```java
public abstract class ErrorFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "error";
    }
}
```

# 3. 过滤器执行顺序

Zuul 内部通过 `filterOrder()` 返回值来排序，**数字越小越先执行**。

```
Pre 阶段：
    -3  ServletDetectionFilter        ← 检测是否通过 Spring DispatcherServlet
    -2  Servlet30WrapperFilter        ← 包装 Servlet 3.0 请求
    -1  FormBodyWrapperFilter          ← 包装表单请求体
     1  DebugFilter                    ← 调试信息记录
     5  PreDecorationFilter            ← ★ 路由决定（核心）
    10  用户自定义 Pre Filter...
```

```
Route 阶段：
    10  RibbonRoutingFilter            ← ★ Ribbon 路由
   100  SimpleHostRoutingFilter        ← URL 路由
   500  SendForwardFilter              ← Forward 本地转发
```

```
Post 阶段：
   900  LocationRewriteFilter          ← 重定向 Location 头
  1000  SendResponseFilter             ← ★ 输出响应到客户端
  1100  用户自定义 Post Filter...
```

```
Error 阶段：
     0  SendErrorFilter                ← ★ 统一错误响应
```

## 3.1 PreDecorationFilter 的作用

`PreDecorationFilter`（order=5）是 Pre 阶段最核心的过滤器，完成了路由决定的全部逻辑：

```java
public class PreDecorationFilter extends ZuulFilter {
    @Override
    public int filterOrder() {
        return 5;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String requestURI = ctx.getRequest().getRequestURI();

        // 1. 查找匹配的路由
        Route route = this.routeLocator.getMatchingRoute(requestURI);

        if (route != null) {
            String location = route.getLocation(); // 目标 URL 或 serviceId
            ctx.put(REQUEST_URI_KEY, route.getPath());  // 设置路由路径
            ctx.put(PROXY_KEY, route.getId());          // 设置路由 ID
            ctx.setRouteHost(null);                     // URL路由使用

            if (location.startsWith("http") || location.startsWith("https")) {
                // URL 路由：直接设置 routeHost
                ctx.setRouteHost(getUrl(location));
                // 将前缀剥离后的路径存入 requestURI
                ctx.put(REQUEST_URI_KEY, strippedPath);
            } else {
                // serviceId 路由：设置 serviceId
                ctx.set("serviceId", location);
                ctx.setRouteHost(null);
                // 交由 RibbonRoutingFilter 后续处理
                ctx.addOriginResponseHeader("X-Zuul-ServiceId", location);
            }
        }

        return null;
    }
}
```

# 4. 内置过滤器一览

Spring Cloud Zuul 开箱即用内置了以下过滤器：

| 过滤器 | 类型 | Order | 功能 |
|--------|------|-------|------|
| `ServletDetectionFilter` | pre | -3 | 判断请求是否通过 Spring DispatcherServlet |
| `Servlet30WrapperFilter` | pre | -2 | 包装 HttpServletRequest（Servlet 3.0） |
| `FormBodyWrapperFilter` | pre | -1 | 包装表单请求体以便多次读取 |
| `DebugFilter` | pre | 1 | 根据 `zuul.debug.request=true` 设置调试标记 |
| `PreDecorationFilter` | pre | 5 | **路由决定**：匹配路由、剥离前缀、设置目标地址 |
| `RibbonRoutingFilter` | route | 10 | Ribbon 负载均衡路由（serviceId 路由） |
| `SimpleHostRoutingFilter` | route | 100 | URL 直接路由（url 路由） |
| `SendForwardFilter` | route | 500 | Forward 本地转发 |
| `SendResponseFilter` | post | 1000 | 将后端响应写入客户端输出流 |
| `SendErrorFilter` | error | 0 | 统一错误响应处理 |

# 5. shouldFilter() 的执行判断

`shouldFilter()` 在每个过滤器执行前被调用，用于运行时条件判断。如果返回 `false`，该过滤器在本轮请求中跳过：

```java
// RibbonRoutingFilter 的判断逻辑
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    // 只有 serviceId 路由时才执行
    return (ctx.getRouteHost() == null
            && ctx.get("serviceId") != null
            && ctx.sendZuulResponse());
}

// SimpleHostRoutingFilter 的判断逻辑
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    // 只有 URL 路由（routeHost 不为空）时才执行
    return ctx.getRouteHost() != null && ctx.sendZuulResponse();
}

// SendErrorFilter 的判断逻辑
public boolean shouldFilter() {
    RequestContext ctx = RequestContext.getCurrentContext();
    // 只有当请求上下文中有异常时才执行
    return ctx.getThrowable() != null
            && !ctx.getBoolean("sendErrorFilter.ran", false);
}
```

# 6. RequestContext 中的关键标志位

过滤器之间通过 `RequestContext` 传递状态和标志位：

```java
// Pre 阶段设置路由信息
ctx.set("serviceId", "user-service");        // RibbonRoutingFilter 检查
ctx.setRouteHost(new URL("http://..."));      // SimpleHostRoutingFilter 检查
ctx.set("requestURI", "/api/user/list");      // 最终转发的 URI

// Route 阶段设置响应信息
ctx.setResponseDataStream(inputStream);        // 后端响应流
ctx.setResponseStatusCode(200);               // 响应状态码
ctx.addZuulResponseHeader("X-TraceId", id);   // 自定义响应头

// 终止处理链
ctx.setSendZuulResponse(false);               // 设为 false 后不再继续

// Post 阶段检查
ctx.getResponseDataStream();                  // 供 SendResponseFilter 读取
ctx.getResponseStatusCode();                  // 供 LocationRewriteFilter 检查
```

# 7. 过滤器生命周期完整流程图

```
                    HTTP Request
                         │
                         ▼
              ┌─────────────────────┐
              │  ZuulServlet        │
              │  service() 方法      │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  init(request, rsp) │  初始化 RequestContext
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  preRoute()         │  执行所有 Pre 过滤器
              │  runFilters("pre")  │  (按 filterOrder 升序)
              └──────┬──────┬───────┘
                     │      │
                成功 │      │ 异常
                     │      └──────────────────────┐
              ┌──────▼──────┐                       │
              │  route()    │  执行所有 Route 过滤器 │
              │ runFilters  │  (按 filterOrder 升序) │
              │  ("route")  │                       │
              └──────┬──────┬───────────────────┐   │
                     │      │                   │   │
                成功 │      │ 异常               │   │
                     │      └──────────┐        │   │
              ┌──────▼──────┐           │        │   │
              │  postRoute() │          │        │   │
              │ runFilters   │          │        │   │
              │  ("post")    │          │        │   │
              └──────┬───────┘          │        │   │
                     │                  │        │   │
                     ▼                  ▼        ▼   ▼
              ┌──────────────────────────────────────────┐
              │  error(e)     执行所有 Error 过滤器       │
              │  runFilters   最后兜底执行                 │
              │  ("error")    SendErrorFilter            │
              └──────────────────────────────────────────┘
                                    │
                                    ▼
              ┌──────────────────────────────────────────┐
              │  RequestContext.getCurrentContext().unset │
              │  清除 ThreadLocal，防止内存泄漏           │
              └──────────────────────────────────────────┘
```
