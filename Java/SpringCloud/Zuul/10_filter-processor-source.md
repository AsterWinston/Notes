
# 1. 概述

`FilterProcessor` 是 Zuul 过滤器链的执行引擎，负责管理四种类型过滤器的执行、异常处理和结果传播。它采用**单例模式**，是连接 ZuulServlet 与具体过滤器之间的桥梁。

```java
package com.netflix.zuul;

public class FilterProcessor {
    static FilterProcessor instance = new FilterProcessor();  // 单例

    public static FilterProcessor getInstance() {
        return instance;
    }
}
```

# 2. 核心方法

## 2.1 runFilters() —— 按类型批量执行

```java
public Object runFilters(String sType) throws Throwable {
    // 返回 Boolean，表示是否有过滤器返回了非 null 的 Boolean 值
    boolean bResult = false;

    // 1. 从 FilterLoader 获取该类型的所有已排序过滤器
    List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);

    if (list != null) {
        for (int i = 0; i < list.size(); i++) {
            ZuulFilter zuulFilter = list.get(i);
            // 2. 依次处理每个过滤器
            Object result = processZuulFilter(zuulFilter);
            if (result != null && result instanceof Boolean) {
                bResult |= ((Boolean) result);
            }
        }
    }
    return bResult;
}
```

关键设计点：

1. **过滤器来源**：`FilterLoader.getInstance().getFiltersByType(sType)`，每次调用都会重新获取列表（支持动态加载）
2. **顺序保证**：`FilterLoader.getFiltersByType()` 已按 `filterOrder()` 升序排序，此处按序迭代
3. **返回值累加**：`bResult |= result` 用于合并多个过滤器的布尔返回值
4. **异常传播**：单个过滤器抛出的异常不会立即中断整个链，而是由 `processZuulFilter()` 捕获后重新抛出

## 2.2 processZuulFilter() —— 单个过滤器执行

```java
public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
    RequestContext ctx = RequestContext.getCurrentContext();
    long s = System.currentTimeMillis();  // 记录开始时间
    long st = -1;
    Object o = null;
    Throwable t = null;

    // 1. 判断是否应该执行
    if (!filter.shouldFilter()) {
        // 统计信息：未执行的过滤器计数
        return null;
    }

    try {
        // 2. 执行过滤器
        o = filter.run();
    } catch (Throwable e) {
        t = e;
        // 3. 将异常存入 RequestContext
        ctx.setThrowable(e);
        throw e;  // 重新抛出，由上层处理
    } finally {
        // 4. 记录过滤器执行统计
        st = System.currentTimeMillis();
        // 统计信息：记录执行时间、成功/失败状态
    }
    return o;
}
```

该方法的执行流程：

```
processZuulFilter(zuulFilter)
    │
    ├─ step 1: shouldFilter() == false? → 返回 null（跳过）
    │
    ├─ step 2: run() → 执行过滤器业务逻辑
    │   │
    │   ├── 正常返回 → 记录统计 → return result
    │   │
    │   └── 抛出异常 → ctx.setThrowable(e) → 记录统计 → throw e
    │
    └─ finally: 记录执行耗时
```

# 3. 四个阶段的执行入口

## 3.1 preRoute()

```java
public void preRoute() throws ZuulException {
    try {
        runFilters("pre");
    } catch (ZuulException e) {
        throw e;
    } catch (Throwable e) {
        // 非 ZuulException 包装为 ZuulException
        throw new ZuulException(e, 500, e.getMessage());
    }
}
```

Pre 阶段执行所有 `filterType() = "pre"` 的过滤器。异常处理策略：
- `ZuulException` 直接抛出
- 其他 `Throwable` 包装为 `ZuulException(nStatusCode=500)` 后抛出

## 3.2 route()

```java
public void route() throws ZuulException {
    try {
        runFilters("route");
    } catch (ZuulException e) {
        throw e;
    } catch (Throwable e) {
        throw new ZuulException(e, 500, e.getMessage());
    }
}
```

Route 阶段执行所有 `filterType() = "route"` 的过滤器。此处执行实际的 HTTP 请求转发。

## 3.3 postRoute()

```java
public void postRoute() throws ZuulException {
    try {
        runFilters("post");
    } catch (ZuulException e) {
        throw e;
    } catch (Throwable e) {
        throw new ZuulException(e, 500, e.getMessage());
    }
}
```

Post 阶段执行所有 `filterType() = "post"` 的过滤器。响应处理和输出在此阶段完成。

## 3.4 error()

```java
public void error() {
    try {
        runFilters("error");
    } catch (Throwable e) {
        // 静默处理：Error 过滤器自身的异常不向上传播
    }
}
```

Error 阶段的异常处理与前三阶段不同——**异常被静默吞掉**。设计理由：
1. Error 阶段已是兜底机制，其内部异常不应再触发新一轮 error
2. 避免无限递归（error 中的 error 中的 error...）

# 4. FilterLoader 过滤器加载机制

```java
public class FilterLoader {
    static FilterLoader instance = new FilterLoader();  // 单例

    // 记录每个过滤器文件的最后修改时间
    private final ConcurrentHashMap<String, Long> filterClassLastModified =
        new ConcurrentHashMap<>();

    // 缓存过滤器的 Groovy/Java 源码
    private final ConcurrentHashMap<String, String> filterClassCode =
        new ConcurrentHashMap<>();

    // 记录已检查过的过滤器文件名
    private final ConcurrentHashMap<String, String> filterCheck =
        new ConcurrentHashMap<>();

    // 引用 FilterRegistry
    FilterRegistry filterRegistry = FilterRegistry.instance();

    // 过滤器文件根目录
    static String zuulFilterRoot = System.getProperty("zuul.filters.root", "");

    // 检查间隔（秒）
    static int zuulFilterRefreshInterval =
        Integer.getInteger("zuul.filters.interval", -1);
}
```

## 4.1 getFiltersByType()

```java
public List<ZuulFilter> getFiltersByType(String filterType) {
    return getFiltersByType(filterType, true);
}

public List<ZuulFilter> getFiltersByType(String filterType, boolean refresh) {
    if (refresh) {
        // 1. 检查并加载新的过滤器文件
        loadFiltersFromClassLoader();
    }

    // 2. 从 FilterRegistry 获取所有已注册过滤器
    List<ZuulFilter> list = new ArrayList<>();
    Collection<ZuulFilter> filters = filterRegistry.getAllFilters();

    // 3. 按类型筛选
    for (ZuulFilter filter : filters) {
        if (filter.filterType().equals(filterType)) {
            list.add(filter);
        }
    }

    // 4. 按 filterOrder 升序排序
    Collections.sort(list, new Comparator<ZuulFilter>() {
        @Override
        public int compare(ZuulFilter o1, ZuulFilter o2) {
            return Integer.compare(o1.filterOrder(), o2.filterOrder());
        }
    });

    return list;
}
```

执行步骤：
1. `loadFiltersFromClassLoader()`——触发动态加载检查
2. 从 `FilterRegistry` 获取全部已注册过滤器
3. 按 `filterType` 筛选
4. 按 `filterOrder` 排序

## 4.2 loadFiltersFromClassLoader()

```java
private void loadFiltersFromClassLoader() throws Exception {
    // 1. 读取配置的过滤器根目录下的 .class / .groovy 文件
    String rootPath = getZuulFilterRoot();

    if (rootPath == null || rootPath.isEmpty()) {
        // 没有配置动态过滤器路径，仅使用 Spring Bean 注册
        return;
    }

    // 2. 检查文件是否有变更
    File rootDir = new File(rootPath);
    File[] files = rootDir.listFiles();
    if (files == null) return;

    for (File file : files) {
        String fileName = file.getName();
        Long lastModified = filterClassLastModified.get(fileName);

        // 3. 如果文件有更新，重新编译/加载
        if (lastModified == null || file.lastModified() > lastModified) {
            Class<?> clazz = compileGroovyOrLoadClass(file);
            ZuulFilter filter = (ZuulFilter) clazz.newInstance();
            // 4. 注册到 FilterRegistry
            filterRegistry.put(fileName, filter);
            // 5. 更新修改时间
            filterClassLastModified.put(fileName, file.lastModified());
        }
    }
}
```

动态加载支持两种格式：
- `.class` 文件：通过 ClassLoader 直接加载
- `.groovy` 文件：通过 GroovyClassLoader 编译后加载

# 5. FilterRegistry 注册机制

```java
public class FilterRegistry {
    private static final FilterRegistry instance = new FilterRegistry();

    // 存储所有已注册的过滤器
    // Key: filterType + ":" + filterName(), 如 "pre:AuthFilter"
    private final ConcurrentHashMap<String, ZuulFilter> filters =
        new ConcurrentHashMap<>();

    public static final FilterRegistry instance() {
        return instance;
    }

    public ZuulFilter remove(String key) {
        return this.filters.remove(key);
    }

    public ZuulFilter get(String key) {
        return this.filters.get(key);
    }

    public void put(String key, ZuulFilter filter) {
        this.filters.putIfAbsent(key, filter);  // 仅当不存在时才放入
    }

    public int size() {
        return this.filters.size();
    }

    public Collection<ZuulFilter> getAllFilters() {
        return this.filters.values();
    }
}
```

## 5.1 Spring Cloud 的注册时机

Spring Cloud 通过 `ZuulServerAutoConfiguration` 在应用启动时自动注册所有 `ZuulFilter` 类型的 Bean：

```java
@Configuration
public class ZuulServerAutoConfiguration {
    // 自动注入所有 ZuulFilter Bean
    @Autowired(required = false)
    private Map<String, ZuulFilter> filters;

    @PostConstruct
    public void configureFilters() {
        // 将所有注入的过滤器注册到 FilterRegistry
        for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
            FilterRegistry.instance()
                .put(entry.getKey(), entry.getValue());
        }
    }
}
```

注入的 Map 类型是 `Map<String, ZuulFilter>`，Spring 自动将 Bean 的名称作为 Key：
- Bean 名由 `@Component("name")` 或默认的类名首字母小写决定
- 例如 `AuthFilter` Bean → Key = "authFilter"

但在 `FilterRegistry` 中存储时的 Key 并非直接使用 Bean 名。实际上 `putIfAbsent` 使用的是调用者指定的 Key。Netflix 原版的 `BasicFilterUsage` 中使用 `filterName()` 作为 Key。

## 5.2 ConcurrentHashMap 的线程安全保证

```
FilterRegistry.filters: ConcurrentHashMap<String, ZuulFilter>

多线程并发场景：
Thread-A: getFiltersByType("pre")  → 读取 filters.values()
Thread-B: put("newFilter", filter) → 向 filters 写入新条目
Thread-C: remove("oldFilter")      → 从 filters 移除条目

ConcurrentHashMap 保证：
- 读操作不加锁，返回的值是"当时那一刻"的有效视图
- 写操作保证原子性
- 不会出现死锁或数据破坏
```

# 6. 过滤器链的执行终止机制

Zuul 提供了三种方式终止过滤器链的继续执行：

## 6.1 通过 setSendZuulResponse(false)

```java
// 在 Pre 过滤器中
ctx.setSendZuulResponse(false);  // 终止请求处理
```

这会导致后续 Route 和 Post 过滤器的 `shouldFilter()` 返回 false：

```java
// RibbonRoutingFilter.shouldFilter()
public boolean shouldFilter() {
    return ctx.getRouteHost() == null
        && ctx.get("serviceId") != null
        && ctx.sendZuulResponse();  // ← false，跳过
}
```

## 6.2 通过抛出异常

```java
// 在任意过滤器中
throw new ZuulException("认证失败", 401, "Unauthorized");
```

异常被 `FilterProcessor.processZuulFilter()` 捕获后存入 `RequestContext`，然后重新抛出。上层 `ZuulServlet.service()` 捕获后跳转到 `error()` 方法：

```java
// ZuulServlet 中的异常处理
try {
    preRoute();
} catch (ZuulException e) {
    error(e);      // 执行 Error 过滤器
    postRoute();   // 执行 Post 过滤器（包括 SendResponseFilter）
    return;        // 不执行 route()
}
```

## 6.3 通过 shouldFilter() 的逻辑

这是隐式的终止方式。过滤器通过检查上下文状态来决定是否执行：

```java
// 例如 SimpleHostRoutingFilter 只在 URL 路由时执行
public boolean shouldFilter() {
    return ctx.getRouteHost() != null && ctx.sendZuulResponse();
}

// RibbonRoutingFilter 只在 serviceId 路由时执行
public boolean shouldFilter() {
    return ctx.getRouteHost() == null
        && ctx.get("serviceId") != null
        && ctx.sendZuulResponse();
}
```

两者互斥，确保每次请求只有一个 RouteFilter 实际发起 HTTP 请求。

# 7. 统计与调试支持

## 7.1 DebugFilter

```java
public class DebugFilter extends ZuulFilter {
    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 1; }

    @Override
    public boolean shouldFilter() {
        HttpServletRequest request = RequestContext.getCurrentContext().getRequest();
        // 只有请求参数中带了 debug=true 时才启用
        return "true".equals(request.getParameter("debug"));
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.setDebugRequest(true);   // 标记调试模式
        ctx.setDebugRouting(true);   // 记录路由信息
        return null;
    }
}
```

启用方式：

```
GET http://gateway:8080/api/user/list?debug=true
```

## 7.2 过滤器执行统计

`FilterProcessor` 在 `processZuulFilter()` 中记录了每个过滤器的执行情况：

```java
public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
    long l = System.currentTimeMillis();
    // ... 执行过滤器 ...
    long st = System.currentTimeMillis();
    long elapsed = st - l;

    // 记录执行时间
    // 可通过扩展点获取这些统计数据
}
```

这些统计数据可以通过 `/actuator/filters` 端点查看（需开启 Actuator 相关端点）。

# 8. 自定义 FilterProcessor 扩展

Zuul 允许通过配置类替换默认的 `FilterProcessor`：

```java
@Configuration
public class CustomFilterProcessorConfig {

    @PostConstruct
    public void customizeFilterProcessor() {
        FilterProcessor.setProcessor(new CustomFilterProcessor());
    }

    static class CustomFilterProcessor extends FilterProcessor {
        @Override
        public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
            // 自定义处理逻辑：例如添加告警
            try {
                Object result = super.processZuulFilter(filter);
                return result;
            } catch (ZuulException e) {
                // 发送告警通知
                alertService.send("过滤器执行异常: " + filter.filterName());
                throw e;
            }
        }
    }
}
```

注意：`FilterProcessor.setProcessor()` 不是 Netflix 原版的公共 API，需要通过反射或继承方式实现。在实际项目中，推荐通过自定义过滤器装饰器或 AOP 实现类似功能。
