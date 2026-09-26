
# 1. Zuul 文件上传的特殊性

Zuul 网关中处理文件上传与普通请求有本质区别：

```
普通请求流程：
  DispatcherServlet → ZuulController → ZuulServlet → RouteFilter → 后端

文件上传流程：
  DispatcherServlet → ZuulController → ZuulServlet → 绕过 DispatcherServlet → 后端
```

核心差异在于：Zuul 需要绕过 Spring MVC 的 `DispatcherServlet` 对请求体（multipart）的解析，将文件流直接透传到后端服务。

# 2. ServletDetectionFilter 与文件上传

`ServletDetectionFilter`（order=-3）判断请求是否通过 Spring DispatcherServlet：

```java
public class ServletDetectionFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return -3; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // 判断是否由 DispatcherServlet 处理
        boolean isDispatcherServletRequest =
            request instanceof DispatcherServlet
            || request.getAttribute(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null;

        ctx.set("isDispatcherServletRequest", isDispatcherServletRequest);

        return null;
    }
}
```

如果请求经过了 `DispatcherServlet`，它可能已经被 `MultipartResolver` 预处理过（文件已写入磁盘/内存），这会导致 Zuul 无法正确获取原始的文件流。

# 3. 文件上传配置

## 3.1 基础配置

```yaml
spring:
  servlet:
    multipart:
      enabled: true           # 启用 multipart 解析
      max-file-size: 50MB     # 单个文件最大大小
      max-request-size: 100MB # 整个请求最大大小

zuul:
  routes:
    file-service:
      path: /file/**
      serviceId: file-service
      sensitive-headers:

# 文件上传需要关闭 Spring MultipartResolver 的自动解析
# 方法一：在路由配置中忽略特定路径
zuul:
  ignored-patterns: /file/upload/**
```

## 3.2 禁用 DispatcherServlet 对上传路径的处理

```yaml
zuul:
  servlet-path: /zuul

  routes:
    file-service:
      path: /file/**
      serviceId: file-service
```

将 `zuul.servlet-path` 设置为非根路径（如 `/zuul`），这样匹配 `/file/**` 的请求就不会经过 `DispatcherServlet`，文件流可以原样传递。

## 3.3 代码方式配置

```java
@Configuration
public class FileUploadConfig {

    @Bean
    public MultipartResolver multipartResolver() {
        // 返回 null 可禁用 Spring 的 MultipartResolver
        // 但在 Zuul 网关环境下不推荐（会影响其他非代理的业务）
        return null;
    }

    @Bean
    public ServletRegistrationBean<ZuulServlet> zuulServlet() {
        ServletRegistrationBean<ZuulServlet> registration =
            new ServletRegistrationBean<>(new ZuulServlet(), "/zuul/*");
        registration.addInitParameter("buffer-requests", "false");
        return registration;
    }
}
```

# 4. 文件上传的流式传输原理

Zuul 通过 `SimpleHostRoutingFilter`（URL 路由）或 `RibbonRoutingFilter`（serviceId 路由）转发文件上传请求：

```java
// SimpleHostRoutingFilter 中的请求体处理
private void copyRequestBody(HttpServletRequest request,
                              HttpClientRequest httpClientRequest)
        throws IOException {
    // 1. 获取原始请求体流
    InputStream inputStream = request.getInputStream();

    // 2. 如果请求体已被 DispatcherServlet 预处理，
    //    则已经缓存在 RequestContext 中
    RequestContext ctx = RequestContext.getCurrentContext();
    if (ctx.get("requestEntity") != null) {
        // 从缓存中读取（非文件上传场景）
        // ...
    } else {
        // 直接从原始流读取（文件上传场景）
        httpClientRequest.setEntity(new InputStreamEntity(
            inputStream, request.getContentLength()));
    }
}
```

## 4.1 流式传输的优势

- **内存友好**：文件不会完全加载到内存，而是边读边转发
- **延迟低**：不需要等待文件完全接收后才开始转发
- **支持大文件**：不受 JVM 堆内存限制

## 4.2 流式传输的限制

- **不支持重试**：流式传输中如果转发失败，客户端已经发送了一部分数据，无法重新读取
- **超时控制难**：大文件上传持续时间长，需合理设置超时

# 5. 大文件上传调优

## 5.1 超时配置

```yaml
# Zuul 层超时
zuul:
  host:
    socket-timeout-millis: 300000    # 5分钟读超时
    connect-timeout-millis: 5000    # 5秒连接超时

# Ribbon 层超时
file-service:
  ribbon:
    ConnectTimeout: 5000
    ReadTimeout: 300000  # 5分钟读超时

# Hystrix 层超时（必须大于 Ribbon 超时）
hystrix:
  command:
    file-service:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 360000  # 6分钟

# Tomcat 连接超时
server:
  tomcat:
    connection-timeout: 360000  # 6分钟
```

## 5.2 请求体大小限制

```yaml
# 方式一：Spring Multipart 限制
spring:
  servlet:
    multipart:
      max-file-size: 500MB
      max-request-size: 600MB

# 方式二：Tomcat 限制
server:
  tomcat:
    max-http-form-post-size: 600MB
    max-swallow-size: 600MB
```

## 5.3 连接池配置（URL 路由）

```yaml
zuul:
  host:
    max-total-connections: 500       # 最大连接数
    max-per-route-connections: 100   # 每个 route 的最大连接数
    time-to-live: 300000            # 连接最大存活时间(ms)
    time-unit: MILLISECONDS
```

# 6. 文件上传的自定义过滤器

## 6.1 文件类型校验过滤器

```java
@Component
public class FileValidationFilter extends ZuulFilter {

    // 允许的文件类型
    private static final Set<String> ALLOWED_CONTENT_TYPES = Set.of(
        "image/jpeg", "image/png", "image/gif",
        "application/pdf",
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "text/plain"
    );

    // 最大文件大小（10MB）
    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024;

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 0; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        String contentType = ctx.getRequest().getContentType();
        return contentType != null
            && contentType.startsWith("multipart/form-data");
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // Content-Length 校验
        long contentLength = request.getContentLengthLong();
        if (contentLength > MAX_FILE_SIZE) {
            reject(ctx, 413, "文件大小超过" + (MAX_FILE_SIZE / 1024 / 1024) + "MB限制");
            return null;
        }

        return null;
    }

    private void reject(RequestContext ctx, int status, String message) {
        ctx.setSendZuulResponse(false);
        ctx.setResponseStatusCode(status);
        ctx.setResponseBody(
            String.format("{\"code\":%d,\"message\":\"%s\"}", status, message));
        ctx.getResponse().setContentType("application/json;charset=UTF-8");
    }
}
```

## 6.2 文件存储路径注入

```java
@Component
public class FileUploadPathFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 5; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return ctx.getRequest().getRequestURI().startsWith("/file/upload");
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 注入用户信息到请求头，后端用于确定文件存储路径
        String userId = ctx.getRequest().getHeader("X-User-Id");
        if (userId != null) {
            ctx.addZuulRequestHeader("X-File-Path", "/uploads/" + userId);
        }

        return null;
    }
}
```

# 7. 常见问题

## 7.1 文件上传返回 413 Request Entity Too Large

这是最常见的文件上传问题，原因是文件大小超过了某个环节的限制：

```yaml
# 三层限制都需要调整
spring.servlet.multipart.max-file-size: 500MB      # Spring
server.tomcat.max-http-form-post-size: 500MB        # Tomcat
zuul.host.max-per-route-connections: 100
```

## 7.2 文件损坏（Content-Length 不匹配）

原因：Spring MVC 的 `MultipartResolver` 处理后修改了请求体，但 Content-Length 没有同步更新。

解决方案：

```java
@Configuration
public class FileUploadConfig {

    @Bean
    @Primary
    public MultipartResolver multipartResolver() {
        StandardServletMultipartResolver resolver =
            new StandardServletMultipartResolver();
        resolver.setResolveLazily(true);  // 延迟解析
        return resolver;
    }
}
```

或者在 `application.yml` 中：

```yaml
spring:
  servlet:
    multipart:
      resolve-lazily: true   # 延迟解析（访问时才解析）
```

## 7.3 大文件上传超时

大文件上传中，如果使用信号量隔离（默认），Tomcat 线程全程阻塞。可以通过配置线程隔离改善：

```yaml
zuul:
  ribbon-isolation-strategy: THREAD
  thread-pool:
    use-separate-thread-pools: true
```

# 8. 分片上传支持

对于超大文件（>500MB），推荐在客户端做分片上传，Zuul 网关透传分片请求：

```java
@Component
public class ChunkUploadFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 5; }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        return request.getRequestURI().startsWith("/file/chunk")
            && request.getParameter("chunk") != null;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // 将分片信息传递给后端
        ctx.addZuulRequestHeader("X-Chunk-Index",
            request.getParameter("chunk"));
        ctx.addZuulRequestHeader("X-Chunk-Total",
            request.getParameter("totalChunks"));
        ctx.addZuulRequestHeader("X-File-Id",
            request.getParameter("fileId"));
        ctx.addZuulRequestHeader("X-File-Name",
            new String(request.getParameter("fileName").getBytes(
                StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8));

        return null;
    }
}
```
