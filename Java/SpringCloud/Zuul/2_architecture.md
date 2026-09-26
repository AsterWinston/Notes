
# 1. 整体架构概览

Spring Cloud Zuul 的架构核心是**过滤器链（Filter Chain）**，整个网关可以被视为一系列过滤器的有序组合。每一组过滤器负责请求处理流程中某个特定阶段的工作。

```
                             ┌──────────────────────┐
                             │   Servlet Container   │
                             │   (Tomcat / Jetty)    │
                             └──────────┬───────────┘
                                        │ HTTP Request
                                        ▼
                             ┌──────────────────────┐
                             │    ZuulServlet        │
                             │    (入口调度器)        │
                             └──────────┬───────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
              ┌──────────┐        ┌──────────┐       ┌──────────┐
              │ PreFilter│        │RouteFilter│       │PostFilter│
              │ 前拦截器  │        │ 路由拦截器 │       │ 后拦截器  │
              └──────────┘        └──────────┘       └──────────┘
                    │                   │                   │
                    │           ┌───────┴───────┐           │
                    │           ▼               ▼           │
                    │     ┌──────────┐    ┌──────────┐     │
                    │     │ Ribbon   │    │ Simple   │     │
                    │     │ Routing  │    │ Host     │     │
                    │     │          │    │ Routing  │     │
                    │     └────┬─────┘    └────┬─────┘     │
                    │          │               │            │
                    │          ▼               ▼            │
                    │     ┌─────────────────────────────┐   │
                    │     │       后端微服务集群          │   │
                    │     └─────────────────────────────┘   │
                    │                                       │
                    ▼                                       ▼
              ┌──────────────────────────────────────────────────┐
              │                  ErrorFilter                      │
              │  (任意阶段异常时触发，Normal 类型不可执行时执行)     │
              └──────────────────────────────────────────────────┘
```

# 2. 核心组件

Spring Cloud Zuul 的核心组件均在 `com.netflix.zuul` 包下，Spring Cloud 的封装层在 `org.springframework.cloud.netflix.zuul` 包下。

## 2.1 组件关系图

```
┌───────────────────────────────────────────────────────────────────────┐
│                         Spring Cloud Zuul                             │
│                                                                       │
│  @EnableZuulProxy ────激活──→ ZuulProxyAutoConfiguration              │
│                                       │                               │
│                     ┌─────────────────┼─────────────────┐             │
│                     ▼                 ▼                  ▼             │
│            ZuulConfiguration  ZuulServerAutoConf  ZuulProxyConfig    │
│            (ZuulServlet注册)  (默认Filter注册)    (Ribbon/Hystrix)    │
│                     │                 │                  │             │
│                     ▼                 ▼                  ▼             │
│               ZuulServlet       SimpleRouteLocator  DiscoveryClient   │
│                     │                                RouteLocator     │
│                     ▼                                                  │
│               ZuulRunner                                              │
│                     │                                                  │
│                     ▼                                                  │
│              FilterProcessor  ──→  FilterLoader  ──→  FilterRegistry  │
│                (过滤器    )        (过滤器加载器)      (过滤器注册表)   │
│                 执行引擎 )                                            │
└───────────────────────────────────────────────────────────────────────┘
```

## 2.2 ZuulServlet

`ZuulServlet` 是整个请求处理的入口，继承自 `HttpServlet`，由 Tomcat 容器管理其生命周期。所有到达 Zuul 的 HTTP 请求最终都会进入 `ZuulServlet.service()` 方法。

```java
public class ZuulServlet extends HttpServlet {
    private ZuulRunner zuulRunner;

    @Override
    public void init(ServletConfig config) throws ServletException {
        // 初始化 ZuulRunner，它是 FilterProcessor 的包装器
        String bufferReqsStr = config.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && Boolean.parseBoolean(bufferReqsStr);
        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) {
        // 委托给 ZuulRunner 执行完整的过滤器生命周期
        zuulRunner.init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
        try {
            preRoute();
            route();
            postRoute();
        } catch (Throwable e) {
            error(e); // new ZuulException(e, 500, e.getMessage())
        } finally {
            RequestContext.getCurrentContext().unset(); // 防止内存泄漏
        }
    }
}
```

关键设计点：
- `ZuulRunner` 是 `FilterProcessor` 的外观（Facade），简化调用
- `RequestContext.getCurrentContext()` 使用 `ThreadLocal` 存储请求上下文，同一线程内的所有过滤器共享
- `finally` 块中的 `unset()` 至关重要，用于释放 `ThreadLocal` 中的引用，防止内存泄漏

## 2.3 FilterProcessor

`FilterProcessor` 是过滤器链的执行引擎，负责按类型和顺序执行所有已注册的过滤器。

```java
public class FilterProcessor {
    private static final FilterProcessor instance = new FilterProcessor(); // 单例

    // 按类型执行过滤器
    public Object runFilters(String sType) throws Throwable {
        boolean bResult = false;
        // 从 FilterLoader 获取该类型的所有过滤器（已排序）
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (ZuulFilter zuulFilter : list) {
                // 处理每个过滤器的执行结果
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }

    // 执行单个过滤器
    public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        try {
            // 1. 判断是否应该执行
            if (!filter.shouldFilter()) {
                return null;
            }
            // 2. 执行过滤器业务逻辑
            Object o = filter.run();
            return o;
        } catch (Throwable e) {
            // 3. 设置异常并重新抛出，供 error 过滤器处理
            ctx.setThrowable(e);
            throw e;
        }
    }
}
```

`FilterProcessor` 的设计值得关注的三点：
1. **单例模式**：全局唯一实例，避免多实例导致过滤器链混乱
2. **异常捕获策略**：捕获异常后写入 `RequestContext` 再重新抛出，这样 `error()` 方法可以从 `RequestContext` 中获取异常信息
3. **布尔返回值累加**：`bResult |= result` 用于通知上层是否有过滤器返回了非 null 结果

## 2.4 FilterLoader

`FilterLoader` 负责过滤器的加载与管理，支持**运行时动态加载**：

```java
public class FilterLoader {
    private static FilterLoader instance = new FilterLoader();

    private final ConcurrentHashMap<String, Long> filterClassLastModified = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> filterClassCode = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> filterCheck = new ConcurrentHashMap<>();
    private FilterRegistry filterRegistry = FilterRegistry.instance();

    // 获取指定类型的所有过滤器
    public List<ZuulFilter> getFiltersByType(String filterType) {
        // 1. 先尝试通过 ClassLoader 加载 filter 类
        loadFiltersFromClassLoader();
        // 2. 从 FilterRegistry 中获取并排序
        List<ZuulFilter> list = filterRegistry.getAllFilters();
        List<ZuulFilter> typedList = new ArrayList<>();
        for (ZuulFilter filter : list) {
            if (filter.filterType().equals(filterType)) {
                typedList.add(filter);
            }
        }
        // 3. 按 filterOrder() 升序排序
        Collections.sort(typedList, new Comparator<ZuulFilter>() {
            public int compare(ZuulFilter o1, ZuulFilter o2) {
                return Integer.compare(o1.filterOrder(), o2.filterOrder());
            }
        });
        return typedList;
    }
}
```

动态加载机制依赖两个关键配置：
- `zuul.filters.root`：过滤器文件根目录（如 `"zuul.filters"`）
- `zuul.filters.interval`：检查过滤器文件变更的时间间隔（单位秒）

> 动态加载机制在 Groovy 过滤器场景下使用较多，Java 类过滤器通常通过 Spring Bean 方式注册。

## 2.5 FilterRegistry

`FilterRegistry` 是一个过滤器注册表，提供增删查功能：

```java
public class FilterRegistry {
    private static final FilterRegistry instance = new FilterRegistry();
    private final ConcurrentHashMap<String, ZuulFilter> filters = new ConcurrentHashMap<>();

    public ZuulFilter remove(String key) { return this.filters.remove(key); }
    public ZuulFilter get(String key) { return this.filters.get(key); }
    public void put(String key, ZuulFilter filter) { this.filters.putIfAbsent(key, filter); }
    public int size() { return filters.size(); }
    public Collection<ZuulFilter> getAllFilters() { return this.filters.values(); }
}
```

设计简单直接，使用 `ConcurrentHashMap` 保证线程安全，其中 Key 是 `filterType() + ":" + filterName()` 的组合。

## 2.6 RequestContext

`RequestContext` 是 Zuul 中最重要的数据传递载体，继承自 `ConcurrentHashMap<String, Object>`：

```java
public class RequestContext extends ConcurrentHashMap<String, Object> {
    protected static final ThreadLocal<? extends RequestContext> threadLocal =
        new ThreadLocal<RequestContext>() {
            @Override
            protected RequestContext initialValue() {
                return new RequestContext(); // 每个线程独立的上下文实例
            }
        };

    public static RequestContext getCurrentContext() {
        RequestContext context = threadLocal.get();
        return context;
    }

    // 便捷方法
    public HttpServletRequest getRequest() { ... }
    public HttpServletResponse getResponse() { ... }
    public Throwable getThrowable() { ... }
    public void setResponseDataStream(InputStream stream) { ... }
    public boolean sendZuulResponse() { ... }
}
```

核心键值含义：

| Key | 类型 | 含义 |
|-----|------|------|
| `requestURI` | String | 当前请求 URI |
| `proxy` | String | 路由目标标识（如 serviceId） |
| `routeHost` | URL | 路由目标地址 |
| `throwable` | Throwable | 过滤器执行异常 |
| `errorStatusCode` | String | HTTP 错误状态码 |
| `errorException` | Exception | 错误异常对象 |
| `sendZuulResponse` | Boolean | 是否继续发送响应（false 表示终止） |
| `requestBody` | byte[] | 请求体内容 |
| `responseDataStream` | InputStream | 后端响应流 |
| `zuulResponseHeaders` | List | 响应头列表 |

# 3. Spring Cloud 封装层

Spring Cloud 对 Zuul 的封装通过自动配置类完成，几个关键类：

## 3.1 @EnableZuulProxy

```java
@SpringBootApplication
@EnableZuulProxy  // 激活 Zuul 网关代理功能
public class ZuulGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulGatewayApplication.class, args);
    }
}
```

`@EnableZuulProxy` 与 `@EnableZuulServer` 的区别：

| 注解 | 功能 |
|------|------|
| `@EnableZuulServer` | 基础网关能力（ZuulServlet + 基础过滤器），不含 Ribbon/Hystrix |
| `@EnableZuulProxy` | `@EnableZuulServer` + Ribbon 路由 + Hystrix 熔断 + 服务发现路由 |

`@EnableZuulProxy` 实际上是 `@EnableZuulServer` 的超集：

```java
@EnableCircuitBreaker        // 激活 Hystrix
@EnableDiscoveryClient       // 激活服务发现
@EnableZuulServer            // 激活基础 Zuul 功能
public @interface EnableZuulProxy { }
```

## 3.2 ZuulServerAutoConfiguration

这是基础配置类，注册 Zuul 的核心 Bean：

```java
@Configuration
@EnableConfigurationProperties({ ZuulProperties.class })
// 当 org.springframework.cloud.netflix.zuul.filters.ZuulProperties 在 classpath 时生效
public class ZuulServerAutoConfiguration {
    // 注入所有 ZuulFilter 类型的 Bean
    @Autowired(required = false)
    private List<ZuulFilter> filters;

    // 注册 ZuulServlet
    @Bean
    @ConditionalOnMissingBean
    public ServletRegistrationBean<ZuulServlet> zuulServletRegistration() {
        ServletRegistrationBean<ZuulServlet> registration =
            new ServletRegistrationBean<>(new ZuulServlet(), "/zuul/*");
        registration.addInitParameter("buffer-requests", "false");
        return registration;
    }

    // 注册所有内置过滤器到 FilterRegistry
    @PostConstruct
    public void configureFilters() {
        for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
            FilterRegistry.instance().put(entry.getKey(), entry.getValue());
        }
    }
}
```

注意这里 `zuulServletRegistration` 的 URL mapping 默认是 `/zuul/*`，这意味着只有匹配 `/zuul/*` 的请求才会被 ZuulServlet 处理。Spring Cloud 通过 `ZuulHandlerMapping` 覆盖了这个行为，将所有请求都映射到 ZuulServlet。

## 3.3 ZuulHandlerMapping

`ZuulHandlerMapping` 通过 Spring MVC 的 `HandlerMapping` 机制将请求路由到 `ZuulController`：

```java
public class ZuulHandlerMapping extends AbstractUrlHandlerMapping {
    private RouteLocator routeLocator;

    public ZuulHandlerMapping(RouteLocator routeLocator, ZuulController zuulController) {
        this.routeLocator = routeLocator;
        setInterceptors(...); // 未配置拦截器，实际不起作用
        // 将 ZuulController 注册为默认 Handler
        registerHandlers();
    }

    @Override
    protected Object lookupHandler(String urlPath, HttpServletRequest request) {
        // 请求路径匹配路由表，匹配成功则交给 ZuulController 处理
        RequestContext ctx = RequestContext.getCurrentContext();
        // ... 路由匹配逻辑 ...
        return super.lookupHandler(urlPath, request);
    }
}
```

# 4. 配置体系

Spring Cloud Zuul 的配置主要通过 `ZuulProperties` 管理：

```java
@ConfigurationProperties("zuul")
public class ZuulProperties {
    // 路由配置
    private Map<String, ZuulRoute> routes = new LinkedHashMap<>();

    // 忽略的服务列表（不代理这些服务Id的请求）
    private Set<String> ignoredServices = new LinkedHashHashSet<>();

    // 忽略的请求路径前缀
    private Set<String> ignoredPatterns = new LinkedHashSet<>();

    // 路由前缀
    private String prefix = "";

    // 是否去除路由前缀
    private boolean stripPrefix = true;

    // 重试
    private Boolean retryable = false;

    // 敏感 Headers（默认包含 Cookie, Set-Cookie, Authorization）
    private Set<String> sensitiveHeaders = new LinkedHashSet<>(
        Arrays.asList("Cookie", "Set-Cookie", "Authorization"));

    // Host 头相关
    private String hostHeader;
    private boolean addHostHeader = false;
    private boolean addProxyHeaders = true;

    // Ribbon 相关
    private Host host;
    private Semaphore semaphore = new Semaphore();

    // 静态内部类 ZuulRoute
    public static class ZuulRoute {
        private String id;
        private String path;        // Ant 风格路径匹配表达式
        private String serviceId;    // 服务发现中的服务名
        private String url;         // 物理 URL
        private boolean stripPrefix = true;
        private Boolean retryable;
        private Set<String> sensitiveHeaders = new LinkedHashSet<>();
        // ... getters/setters
    }
}
```

这个配置类的设计体现了 Spring Boot 的约定优于配置思想：大部分属性都有合理的默认值，只有路由规则需要使用者在 `application.yml` 中显式配置。
