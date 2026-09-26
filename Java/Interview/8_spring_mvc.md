# 1. 知识点
- **DispatcherServlet**：Spring MVC 前端控制器，所有 HTTP 请求统一入口，负责协调各组件完成请求处理，本身不参与业务逻辑。
- **Spring MVC 请求处理流程**：请求 → DispatcherServlet → HandlerMapping 查找处理器 → HandlerAdapter 执行处理器 → Controller 返回 ModelAndView → ViewResolver 视图解析 → 渲染视图返回响应。返回 JSON 时使用 `HttpMessageConverter` 直接写入响应体，跳过视图解析。
- **HandlerMapping**：请求到处理器的映射组件，根据请求 URL、方法等信息查找匹配的 Handler（Controller 中的方法）。常用实现：`RequestMappingHandlerMapping`。
- **HandlerAdapter**：调用 Handler 的适配器，支持不同类型的处理器（注解 Controller、HttpRequestHandler 等）。常用实现：`RequestMappingHandlerAdapter`。
- **HandlerInterceptor**：拦截器，在 Handler 执行前后插入自定义逻辑，可实现权限校验、日志记录、性能监控。
- **HandlerExceptionResolver**：全局异常处理器，将 Controller 抛出的异常映射到特定的视图或错误响应。常用实现：`ExceptionHandlerExceptionResolver`。
- **ViewResolver**：视图解析器，将逻辑视图名转换为实际的 View 对象（如 JSP、Thymeleaf、FreeMarker 模板）。
- **HttpMessageConverter**：HTTP 消息转换器，在请求体和 Java 对象之间进行序列化/反序列化。`@RequestBody` 和 `@ResponseBody` 依赖此组件工作。
- **MultipartResolver**：文件上传解析器，将 `multipart/form-data` 请求解析为 `MultipartFile`。常用实现：`CommonsMultipartResolver` 或内置 StandardServletMultipartResolver。
- **HandlerMethodArgumentResolver**：参数解析器，将 HTTP 请求中的参数解析为 Controller 方法的参数。Spring 内置 47 种，支持 `@RequestParam`、`@PathVariable`、`@RequestBody` 等注解驱动的参数绑定。
- **HandlerMethodReturnValueHandler**：返回值处理器，将 Controller 方法的返回值转换为响应数据。支持 `@ResponseBody`（JSON）、`ModelAndView`、`String`（视图名）、`ResponseEntity` 等多种返回值。
- **`@RequestMapping`**：将 HTTP 请求映射到 Controller 方法，可指定 URL、Method、Params、Headers、Consumes、Produces 等约束条件。
- **`@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping`**：`@RequestMapping` 的快捷注解，指定了对应的 HTTP Method，语义更清晰。
- **`@PathVariable`**：将 URL 路径中的占位符 `{xxx}` 绑定到方法参数。
- **`@RequestParam`**：将 URL 查询参数（?key=value）或表单参数绑定到方法参数，可设置 required/defaultValue。
- **`@RequestBody`**：将 HTTP 请求体（JSON/XML）反序列化为 Java 对象，通过 `HttpMessageConverter` 实现。
- **`@ResponseBody`**：将 Controller 返回值序列化为 JSON/XML 写入 HTTP 响应体。`@RestController` 组合了 `@Controller` + `@ResponseBody`。
- **`@ModelAttribute`**：将请求参数绑定到 Model 对象，并添加到 Model 中供视图使用。可标注方法参数或单独的方法。
- **`@RequestHeader`**：获取指定的 HTTP 请求头信息。`@CookieValue` 获取 Cookie 值。
- **RESTful 设计规范**：URL 面向资源命名（用名词复数），HTTP Method 表示操作（GET 查询、POST 新增、PUT 全量更新、PATCH 部分更新、DELETE 删除），状态码表示结果，HATEOAS 实现超媒体驱动。
- **Filter**：Servlet 级别的过滤器，在请求进入 Servlet 之前和响应返回客户端之前执行。可链式调用，处理编码、鉴权、日志等横切关注点。
- **Interceptor vs Filter**：Filter 依赖 Servlet 容器，实现在容器级别，执行顺序为 Filter → DispatcherServlet → Interceptor → Controller。Interceptor 是 Spring 级别，可访问 Spring 上下文中的 Bean。
- **`@ControllerAdvice`**：全局异常处理和全局数据绑定的增强器，配合 `@ExceptionHandler` 统一处理异常、`@ModelAttribute` 全局数据绑定、`@InitBinder` 全局数据预处理。
- **`@ExceptionHandler`**：定义在 Controller 内或 `@ControllerAdvice` 类中，处理指定类型的异常，返回错误响应。支持多个方法按异常继承关系匹配最近处理器。
- **`@Valid`/`@Validated`**：JSR-303 数据校验注解，配合校验注解如 `@NotNull`、`@NotEmpty`、`@Size` 等，对请求参数进行校验，校验失败抛出 `MethodArgumentNotValidException`。
- **JSR-303 常用校验注解**：`@NotNull`（不为 null）、`@NotEmpty`（字符串/集合不为空）、`@NotBlank`（字符串不为空白）、`@Size(min,max)`（长度范围）、`@Min`/`@Max`（数值范围）、`@Email`（邮箱格式）、`@Pattern`（正则匹配）、`@Positive`/`@Negative`（正负值）、`@Past`/`@Future`（日期）。
- **自定义校验**：实现 `ConstraintValidator` 接口，定义校验逻辑，并用 `@Constraint` 注解标记自定义注解。
- **跨域（CORS）**：浏览器同源策略限制跨域请求。Spring MVC 通过 `@CrossOrigin` 注解或 `WebMvcConfigurer.addCorsMappings()` 配置允许的域名、方法、请求头等。
- **异步处理**：Controller 返回 `Callable`（Spring 管理的异步任务线程池执行）或 `DeferredResult`（由外部事件触发完成），释放 Servlet 容器工作线程，提升系统吞吐量。
- **`@Async`**：标注方法为异步执行，由 `AsyncTaskExecutor` 线程池执行，方法返回 `Future`/`CompletableFuture`。需在主类或配置类上添加 `@EnableAsync`。
- **内容协商（Content Negotiation）**：客户端通过请求头 `Accept` 或 URL 参数（如 `?format=xml`）指明期望的响应数据格式，服务端选择对应的 `HttpMessageConverter` 处理。配置 `ContentNegotiationConfigurer` 可定制策略。
- **WebMvcConfigurer**：Spring MVC 配置接口，通过实现该接口可自定义拦截器、跨域、资源处理、参数解析器、消息转换器等，取代 XML 配置。
- **`@ResponseStatus`**：标注在异常类或 Controller 方法上，指定 HTTP 响应状态码和原因短语。
- **`ResponseEntity`**：包装返回值和 HTTP 状态码、响应头的响应实体，提供更精细的 HTTP 响应控制。
- **`@MatrixVariable`**：获取 URL 路径中以分号分隔的矩阵变量参数，需在配置中启用。
- **文件上传**：客户端 `multipart/form-data`，服务端通过 `MultipartFile` 接收。需配置 `MultipartResolver`，`@RequestParam("file") MultipartFile file` 绑定文件。
- **`@InitBinder`**：Controller 级别的数据绑定预处理，可注册自定义的 `PropertyEditor` 或 `Validator`，用于格式化日期、trim 字符串等。

# 2. 面试题

## 2.1 描述一个 HTTP 请求在 Spring MVC 中的完整处理流程
1. 客户端发送 HTTP 请求到达 Servlet 容器（Tomcat）。
2. 请求经过 Filter 链（如编码过滤器、安全过滤器）。
3. `DispatcherServlet` 接收请求，调用 `doDispatch()` 方法。
4. `DispatcherServlet` 根据请求 URL 调用所有 `HandlerMapping`，找到匹配的 Handler（Controller 方法）和对应的 HandlerInterceptor 链。
5. `DispatcherServlet` 根据 Handler 类型找到支持该 Handler 的 `HandlerAdapter`。
6. 执行拦截器的 `preHandle()` 方法（任一返回 false 则中断请求）。
7. `HandlerAdapter` 调用 `HandlerMethodArgumentResolver` 解析 Handler 方法的参数（如 @RequestParam、@RequestBody 等）。
8. 反射调用 Controller 的目标方法，得到返回值。
9. 通过 `HandlerMethodReturnValueHandler` 处理返回值：
   - 如果标注 `@ResponseBody`：使用 `HttpMessageConverter` 将对象序列化为 JSON/XML，直接写入响应体。
   - 如果返回 ModelAndView：调用 ViewResolver 解析为 View，渲染后写入响应体。
10. 执行拦截器的 `postHandle()` 方法。
11. 执行拦截器的 `afterCompletion()` 方法（即使发生异常也会执行）。
12. 响应返回客户端。

## 2.2 Filter 和 Interceptor 有什么区别？
| 维度 | Filter | Interceptor |
|------|--------|-------------|
| 规范 | Servlet 规范 | Spring 框架 |
| 容器依赖 | 依赖 Servlet 容器 | 依赖 Spring 容器，可注入 Bean |
| 作用范围 | 对所有请求生效，可拦截静态资源 | 只拦截 Controller 请求（DispatcherServlet 内） |
| 执行顺序 | Filter → DispatcherServlet → Interceptor → Controller | 在 Filter 之后 |
| 拦截粒度 | 只能获取请求和响应对象 | 可获取 Handler（具体 Controller 方法）、ModelAndView |
| 生命周期方法 | init、doFilter、destroy | preHandle、postHandle、afterCompletion |

Filter 用于编码设置、安全鉴权（如 Spring Security 底层就是 Filter 链）、XSS 过滤等。Interceptor 用于权限校验、日志记录、性能统计（可精确到具体方法）。

## 2.3 @ControllerAdvice 和 @ExceptionHandler 如何实现统一异常处理？
`@ControllerAdvice` 是一个增强型 Controller，结合 `@ExceptionHandler` 可以实现全局异常处理：

```java
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result handleBusinessException(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Result handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "服务器内部错误");
    }
}
```

**工作原理**：
1. Spring MVC 使用 `ExceptionHandlerExceptionResolver` 处理异常。
2. 当 Controller 抛出异常时，Spring 首先在出错的 Controller 内部查找 `@ExceptionHandler` 方法。
3. 如果未找到，则在所有 `@ControllerAdvice` 标注的类中查找。
4. 按异常继承关系匹配最接近的处理方法（子类异常优先匹配）。如果有多个匹配的方法且无法确定优先级，根据 `@Order` 或实现 `Ordered` 接口决定。

**`@ExceptionHandler` 方法的参数**：可以注入异常对象、原始请求（HttpServletRequest）、ModelAndView、Session 等。

## 2.4 RESTful API 的设计原则有哪些？
1. **资源导向的 URL**：URL 表示资源而非操作，使用名词复数。如 `/users`、`/orders/123`。
2. **HTTP Method 表示操作**：GET（查询）、POST（新增）、PUT（全量替换）、PATCH（部分更新）、DELETE（删除）。
3. **状态码正确使用**：200（成功）、201（创建成功）、204（无内容）、400（请求错误）、401（未认证）、403（无权限）、404（不存在）、500（服务器错误）。
4. **无状态**：每个请求包含所有必要信息，服务端不存储客户端状态。
5. **资源嵌套层级不宜过深**：一般不超过两层，如 `/users/1/orders`，更深层的资源通过查询参数获取。
6. **版本控制**：通过 URL 路径（`/api/v1/users`）或请求头（`Accept: application/vnd.api.v1+json`）区分版本。
7. **分页与过滤**：使用查询参数 `?page=1&size=20&sort=createdAt,desc&status=active`。
8. **响应格式统一**：返回 JSON，包含统一的响应结构（code、message、data）。

## 2.5 Spring MVC 如何实现跨域配置？
**方式一：`@CrossOrigin` 注解**
```java
// 方法级别
@CrossOrigin(origins = "http://example.com", maxAge = 3600)
@GetMapping("/users")
public List<User> list() { ... }

// 类级别（该 Controller 所有方法允许跨域）
@CrossOrigin(origins = "*")
@RestController
public class UserController { ... }
```

**方式二：全局配置（WebMvcConfigurer）**
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

**方式三：CorsFilter（Filter 级别，最底层）**
```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("*");
    config.addAllowedMethod("*");
    config.addAllowedHeader("*");
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
}
```

`allowCredentials(true)` 时不能同时设置 `allowedOrigins("*")`，需指定具体域名。

## 2.6 Spring MVC 中如何处理文件上传？
1. **MultipartResolver 配置**（Spring Boot 已自动配置 StandardServletMultipartResolver）。
2. **Controller 端接收**：
```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) return "文件为空";
    String fileName = file.getOriginalFilename();
    String filePath = "/upload/" + fileName;
    file.transferTo(new File(filePath));
    return "上传成功";
}
```
3. **多文件上传**：参数改为 `@RequestParam("files") MultipartFile[] files` 或 `List<MultipartFile>`。
4. **限制大小**：
   - application.properties: `spring.servlet.multipart.max-file-size=10MB`，`spring.servlet.multipart.max-request-size=20MB`
   - 通过 `MultipartConfigElement` 编程配置

## 2.7 HandlerMethodArgumentResolver 的作用？如何自定义？
`HandlerMethodArgumentResolver`（方法参数解析器）负责将 HTTP 请求中的数据转换为 Controller 方法的参数。Spring MVC 内置了 47 种解析器，支持 `@RequestParam`、`@PathVariable`、`@RequestBody`、`@ModelAttribute` 等。

**自定义参数解析器**：
```java
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class);
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter, ...) {
        // 从 Token 或 Session 获取当前用户信息
        return userService.getCurrentUser();
    }
}
```
注册到 WebMvcConfigurer 的 `addArgumentResolvers()` 方法中。

## 2.8 Spring MVC 中的异步请求处理有什么优势？如何实现？
**优势**：Servlet 容器（如 Tomcat）的工作线程是有限的（默认 200）。传统同步处理中一个请求占用一个线程直到响应返回。异步处理允许将耗时操作交给独立线程池，释放 Servlet 容器线程去处理其他请求，提升吞吐量。

**方式一：返回 Callable**
```java
@GetMapping("/async")
public Callable<String> async() {
    return () -> {
        Thread.sleep(3000); // 模拟耗时
        return "result";
    };
}
```
Spring MVC 通过 `TaskExecutor` 执行 Callable，Servlet 线程立即释放。

**方式二：返回 DeferredResult**
```java
@GetMapping("/deferred")
public DeferredResult<String> deferred() {
    DeferredResult<String> result = new DeferredResult<>(5000L);
    // 由其他线程（如消息队列、外部事件）完成
    executorService.submit(() -> result.setResult("async result"));
    return result;
}
```
DeferredResult 可以由任意线程调用 `setResult()` 完成，适用于跨线程通知场景。

**配置**：需配置 `AsyncTaskExecutor`，可自定义线程池参数、超时回调等。

## 2.9 @Valid 和 @Validated 的区别？
| 维度 | @Valid | @Validated |
|------|--------|------------|
| 来源 | javax.validation（JSR-303） | Spring 框架 |
| 分组校验 | 不支持 | 支持（通过 groups 属性） |
| 嵌套校验 | 支持 | 不支持（需结合 @Valid） |
| 使用位置 | 方法参数、字段 | 方法参数、类级别 |
| 校验提供 | Hibernate Validator | Hibernate Validator |

**分组校验示例**：
```java
// 定义分组接口
public interface Create {}
public interface Update {}

// 实体类
public class User {
    @NotNull(groups = Update.class)
    @Null(groups = Create.class)
    private Long id;
    @NotBlank(groups = {Create.class, Update.class})
    private String name;
}

// Controller
@PostMapping("/user")
public Result create(@Validated(Create.class) @RequestBody User user) { ... }
```

## 2.10 如何自定义一个校验注解？
```java
// 1. 定义注解
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface Phone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. 实现校验器
public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final Pattern PATTERN = Pattern.compile("^1[3-9]\\d{9}$");
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || PATTERN.matcher(value).matches();
    }
}
```

## 2.11 Spring MVC 中 @RequestMapping 的 produces 和 consumes 属性有什么用？
- **consumes**：限制请求的 Content-Type，只有匹配的请求才会被该方法处理。如 `consumes = "application/json"` 表示只接受 JSON 格式的请求体。
- **produces**：限制响应的 Accept 类型，配合 `HttpMessageConverter` 决定输出格式。如 `produces = "application/json;charset=UTF-8"` 表示返回 JSON。
- 这两个属性参与**内容协商**过程。当请求的 Content-Type 与 consumes 不匹配时，返回 415 Unsupported Media Type。当客户端 Accept 与 produces 不匹配时，返回 406 Not Acceptable。

## 2.12 如何处理 Spring MVC 中的静态资源？
Spring Boot 默认从以下路径加载静态资源：
- `classpath:/static/`
- `classpath:/public/`
- `classpath:/resources/`
- `classpath:/META-INF/resources/`

自定义配置：
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/myres/**")
                .addResourceLocations("classpath:/my-custom/", "file:/external/");
    }
}
```
第一个参数是 URL 映射路径（浏览器访问地址），第二个是资源实际存放位置。优先级：META-INF/resources > resources > static > public。

## 2.13 Interceptor 的生命周期方法执行顺序是怎样的？
在一个请求的处理过程中：

1. `preHandle(request, response, handler)` — Controller 执行前
   - 返回 `true` 则继续执行；返回 `false` 则中断，不再调用后续拦截器和 Controller。
2. Controller 方法执行
3. `postHandle(request, response, handler, modelAndView)` — Controller 执行后、视图渲染前
   - 可修改 ModelAndView。
4. 视图解析和渲染
5. `afterCompletion(request, response, handler, exception)` — 请求完成后（afterCompletion 的 completion）
   - 无论是否发生异常都会执行（除非 preHandle 返回 false）。
   - exception 参数包含异常信息，如果正常执行则为 null。

**多个拦截器链**时的执行顺序：preHandle 按注册顺序正序执行，postHandle 和 afterCompletion 按注册顺序逆序执行。

## 2.14 内容协商（ContentNegotiation）的工作机制？
内容协商决定请求的响应数据格式（JSON、XML 等）。Spring MVC 通过 `ContentNegotiationManager` 管理协商策略：

1. **请求头 Accept**（默认策略）：如 `Accept: application/json` → 返回 JSON；`Accept: application/xml` → 返回 XML。
2. **URL 参数**：`/users?format=json` → 策略 `parameterName=format`。
3. **URL 后缀**（已废弃不推荐）：`/users.json` → 策略 `favorPathExtension=true`。
4. **固定 Content-Type**：`produces = "application/json"` 的优先级最高。

当有多个 `HttpMessageConverter` 都能处理时，根据 `Accept` 权重（q 值）选择最合适的转换器。

## 2.15 @ModelAttribute 的用法有哪些？
1. **标注在方法参数上**：将请求参数自动绑定到对象，并添加到 Model。
```java
@PostMapping("/user")
public String save(@ModelAttribute("user") User user) { ... }
```
2. **标注在方法上**（无 @RequestMapping 的方法）：在每个 Controller 方法执行前，将方法的返回值放入 Model。
```java
@ModelAttribute("genders")
public List<String> genders() {
    return Arrays.asList("男", "女");
}
```
3. 标注在方法参数时，Spring MVC 会以参数名（或 @ModelAttribute 的 value）为 key，将对象存入请求域（request scope）。

主要用于表单对象绑定，以及向视图提供通用数据（如下拉列表选项）。

## 2.16 ResponseEntity 和 @ResponseBody 的使用场景区别？
- `@ResponseBody` + `@ResponseStatus`：返回简单的 JSON 对象和固定状态码。适合大多数标准 REST 接口。
- `ResponseEntity`：返回完整的 HTTP 响应，包括状态码、响应头、响应体。适合需要动态状态码（如 201 Created 带 Location 头）、设置 Cookie、缓存控制等复杂场景。

```java
@PostMapping("/user")
public ResponseEntity<User> create(@RequestBody User user) {
    User saved = userService.save(user);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(saved.getId()).toUri();
    return ResponseEntity.created(location).body(saved);
}
```

## 2.17 @RequestParam 和 @PathVariable 各用于什么场景？
- `@PathVariable`：获取 URL 路径中的占位符变量。用于标识资源的唯一 ID，RESTful API 中 `/users/{id}` 的 `{id}` 部分。
- `@RequestParam`：获取 URL 查询参数（?key=value）或 POST 表单参数。用于过滤、分页、排序等非资源标识的参数。

```java
@GetMapping("/users/{userId}/orders")
public List<Order> orders(
    @PathVariable Long userId,          // 资源标识：/users/123/orders
    @RequestParam int page,             // 分页参数：?page=1
    @RequestParam(defaultValue = "10") int size,  // 带默认值：?size=20
    @RequestParam(required = false) String status  // 可选参数
) { ... }
```

## 2.18 DispatcherServlet 是如何工作的？为什么它能成为前端控制器？
`DispatcherServlet` 是 Spring MVC 的**核心调度器**，继承自 `HttpServlet`，在 web.xml 中配置 URL 映射为 `/`（拦截所有请求）。

工作原理：
1. 在 `init()` 阶段初始化 Spring 容器（WebApplicationContext），扫描并注册所有 MVC 组件。
2. 在 `service()` 阶段收到请求后，调用 `doDispatch()`：
   - 遍历 HandlerMapping，找到匹配的 Handler 和拦截器。
   - 遍历 HandlerAdapter，找到支持当前 Handler 的适配器。
   - 通过拦截器链和适配器执行 Handler。
   - 处理异常（HandlerExceptionResolver）。
   - 使用 ViewResolver 解析视图或 HttpMessageConverter 序列化结果。

它能成为前端控制器是因为它通过**策略模式**组合了 HandlerMapping、HandlerAdapter、ViewResolver 等组件，将请求处理各环节解耦，使得每个环节都可由不同的策略实现替代。

## 2.19 如何在 Spring MVC 中拦截所有的异常并返回统一格式？
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handleValidException(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return Result.error(400, msg);
    }
    
    @ExceptionHandler(BindException.class)
    public Result handleBindException(BindException e) {
        return Result.error(400, "参数绑定异常");
    }
    
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        log.error("未处理的异常", e);
        return Result.error(500, "服务器内部错误");
    }
}
```

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`，省去方法上标注 `@ResponseBody`。

## 2.20 如何自定义拦截器并配置它的拦截路径？
```java
// 1. 定义拦截器
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                             Object handler) {
        // 从请求头获取 token 验证
        String token = request.getHeader("Authorization");
        if (token == null || !JwtUtil.verify(token)) {
            throw new UnauthorizedException("未登录或 token 已过期");
        }
        return true;
    }
}

// 2. 注册拦截器
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private AuthInterceptor authInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")           // 拦截路径
                .excludePathPatterns("/api/login", "/api/register"); // 排除路径
    }
}
```

## 2.21 Spring MVC 的数据绑定（DataBinder）流程
`DataBinder` 是 Spring MVC 将 HTTP 请求参数绑定到 Java 对象的核心机制：

**绑定流程**：
1. 请求到达 Controller 方法 → `MethodParameter` 描述目标参数（类型、注解）。
2. `HandlerMethodArgumentResolver`（如 `RequestParamMethodArgumentResolver`、`ModelAttributeMethodProcessor`）选择合适的处理方式。
3. 创建 `WebDataBinder`：将请求参数名值对与目标对象关联。
4. `ConversionService` 执行类型转换：字符串 "123" → `Integer 123`；"2024-01-01" → `LocalDate`。
5. `Validator` 执行校验（`@Valid` 触发 JSR-303）。
6. 绑定完成 → 目标参数对象传递给 Controller 方法。

**常见扩展点**：
- `@InitBinder`：在 Controller 中定义数据绑定预处理（如日期格式、字段白名单）。
  ```java
  @InitBinder
  public void initBinder(WebDataBinder binder) {
      binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
  }
  ```
- `WebBindingInitializer`：全局级别的 DataBinder 初始化。
- `@DateTimeFormat`：声明日期格式，无需 `@InitBinder` 手动注册（推荐）。

**注意**：DataBinder 只在处理 `@ModelAttribute` 或表单提交时使用。`@RequestBody` 的 JSON 反序列化使用 `HttpMessageConverter`，不走 DataBinder——这通常是常见的误区。

## 2.22 如何自定义 HttpMessageConverter？
`HttpMessageConverter` 负责 HTTP 请求/响应体与 Java 对象的序列化/反序列化（JSON ↔ Object、XML ↔ Object 等）。

**自定义步骤**：
1. 实现 `HttpMessageConverter<T>` 接口（或继承 `AbstractHttpMessageConverter<T>`）。
2. 重写 `supports()`、`readInternal()`、`writeInternal()` 三个方法。
3. 注册到 Spring MVC 配置。

**示例——自定义 Protobuf 消息转换器**：
```java
public class ProtobufHttpMessageConverter extends AbstractHttpMessageConverter<Message> {
    public ProtobufHttpMessageConverter() {
        super(MediaType.APPLICATION_OCTET_STREAM);
    }

    @Override
    protected boolean supports(Class<?> clazz) { return Message.class.isAssignableFrom(clazz); }

    @Override
    protected Message readInternal(Class<? extends Message> clazz, HttpInputMessage inputMessage) throws IOException {
        return Message.parseFrom(inputMessage.getBody());
    }

    @Override
    protected void writeInternal(Message message, HttpOutputMessage outputMessage) throws IOException {
        message.writeTo(outputMessage.getBody());
    }
}
```

**注册到 Spring MVC**：
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new ProtobufHttpMessageConverter());  // 自定义 Converter
    }
}
```
或使用 `extendMessageConverters()` 来添加（而不是覆盖）默认转换器。

**注意**：`configureMessageConverters()` 会移除所有默认 Converter，`extendMessageConverters()` 保留默认 Converter 并追加自定义的。

## 2.23 PathPattern 和 AntPathMatcher 的区别
| 维度 | AntPathMatcher（旧） | PathPattern（新，Spring 5.3+ / Spring Boot 2.6+） |
|------|-------------------|--------------------------------------------------|
| 引入版本 | Spring 1.0 | Spring 5.3 |
| 解析方式 | 每次请求都重新解析路径匹配 | 预解析路径模式，构建 PathPattern 对象（可复用） |
| 性能 | 较慢（每次字符串解析） | 更快（预编译 + 纯规则匹配） |
| 内存占用 | 低（无预编译开销） | 略高（缓存已解析的 Pattern） |
| 语法差异 | `**` 多级路径、`?` 单字符 | 同 AntPathMatcher，但更严格（末尾 `/` 差异） |
| 推荐度 | 旧项目兼容 | 官方推荐，默认开启 |

**切换方式**（Spring Boot 2.6+ 默认已使用 PathPattern）：
```yaml
spring:
  mvc:
    pathmatch:
      matching-strategy: path_pattern_parser  # path_pattern_parser / ant_path_matcher
```

**注意**：PathPattern 不支持某些 AntPathMatcher 的特殊用法，如通过 `AntPathMatcher` 的工具方法直接比较路径。`spring.mvc.pathmatch.matching-strategy=ant_path_matcher` 适用于需要兼容旧代码的场景。Swagger 2 (SpringFox) 在 PathPattern 下会报错，需切换到 AntPathMatcher 或升级到 SpringDoc OpenAPI。
