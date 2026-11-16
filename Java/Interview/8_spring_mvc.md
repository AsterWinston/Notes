本篇覆盖 Spring MVC（Web 层框架）的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：请求处理链路、消息转换与视图、拦截器/过滤器与异常处理、参数校验/跨域/上传与 REST 设计、进阶特性。IoC/AOP 基础见 7_spring.md，HTTP 协议本身见 28_http.md。

# 1. 请求处理链路

## 1.1 DispatcherServlet 与总流程

### 1.1.1 描述一个 HTTP 请求在 Spring MVC 中的完整处理流程

**结论**：一句话——"前端控制器 DispatcherServlet 收到请求后，依次问**找谁处理（HandlerMapping）→ 怎么执行（HandlerAdapter + 参数解析器）→ 结果怎么变成响应（返回值处理器/消息转换器或 ViewResolver）**，全程穿插拦截器与异常解析器"。

十二步展开：

1. 请求到达 Servlet 容器（Tomcat），先过 **Filter 链**（编码、安全、网关埋点）；
2. `DispatcherServlet.doService() → doDispatch()` 统一入口；
3. 遍历 **HandlerMapping**（按 order），URL/Method/Header/Params 匹配出 HandlerMethod（Controller 方法）+ 拦截器链；
4. 找到支持该 Handler 的 **HandlerAdapter**（策略模式：注解 Controller、HttpRequestHandler 各有适配器）；
5. **拦截器 preHandle()** 正序执行，任一返回 false 立即中断（但已执行过的拦截器仍会走 afterCompletion）；
6. Adapter 内经 **HandlerMethodArgumentResolver** 解析每个参数（@RequestParam/@RequestBody/PathVariable/Model…），反射调用目标方法；
7. 返回值交给 **HandlerMethodReturnValueHandler** 处理：
   - 有 `@ResponseBody`/ResponseEntity → 选合适 **HttpMessageConverter** 序列化直接写响应体，**视图流程整体跳过**；
   - 返回 ModelAndView/String 视图名 → 后续走视图；
8. 拦截器 **postHandle()**（仅视图模式有意义）；
9. 异常则进 **HandlerExceptionResolver** 链（3.1.4）；
10. **ViewResolver** 逻辑视图名 → View 对象，`view.render()` 渲染输出；
11. 拦截器 **afterCompletion()**（无论成败都执行，资源收尾）；
12. 响应返回客户端。

> **面试加分**：异步请求（Callable/DeferredResult，5.1.1）会在第 7 步后挂起"当前 Servlet 线程"，容器线程释放，结果就绪后 dispatch 回来续跑第 8~12 步。

### 1.1.2 DispatcherServlet 是如何初始化和工作的？为什么它能做"前端控制器"？

**前端控制器模式**：把"所有请求的统一入口"收敛为一个 Servlet，内部流程编排标准化，各处理环节抽象成可替换策略——客户端、Controller、视图三方解耦。

**初始化（initStrategies()）**：`HttpServletBean.init()` 绑定配置参数 → `initWebApplicationContext()` 创建/引用 WebApplicationContext → `initStrategies()` 从容器**按类型加载九大组件**（MultipartResolver、HandlerMapping 列表、HandlerAdapter 列表、HandlerExceptionResolver 列表、ViewResolver、View、RequestToViewNameTranslator、LocaleResolver、FlashMapManager），按 order 排序存字段。

**运行（doDispatch()）**：`getHandler()` → `getHandlerAdapter()` → 拦截器链 → `ma.handle()` 执行并被 `applyPreHandle/applyPostHandle/applyAfterCompletion` 包裹；异常走 `processHandlerException()`，视图渲染/错误页走 `processDispatchResult()`。

Spring Boot 下由 `DispatcherServletAutoConfiguration` 自动注册并映射 `/`（兜底路由），静态资源走 `ResourceHttpRequestHandler` 仍在同一 Servlet 内。

**为什么成为前端控制器**：策略模式组合九大组件，每个环节可插拔替换；单入口统一做安全、埋点、异常兜底。父子容器分工见 7_spring.md 2.3.3。

### 1.1.3 Spring MVC 的九大组件各管什么？

| 组件 | 职责 | 常用实现 |
|------|------|---------|
| MultipartResolver | multipart 请求解析（文件上传） | StandardServletMultipartResolver（Boot 默认） |
| HandlerMapping | 请求 → Handler + 拦截器链 | RequestMappingHandlerMapping、Resources（静态） |
| HandlerAdapter | 适配并执行 Handler | RequestMappingHandlerAdapter |
| HandlerExceptionResolver | 异常 → 错误响应/视图 | ExceptionHandlerExceptionResolver、ResponseEntityExceptionHandler、DefaultHandlerExceptionResolver |
| ViewResolver | 逻辑视图名 → View | InternalResourceViewResolver、ThymeleafViewResolver |
| View | 模型渲染输出 | JstlView、ThymeleafView、MappingJackson2JsonView |
| RequestToViewNameTranslator | 无视图名时按 URL 推断 | DefaultRequestToViewNameTranslator |
| LocaleResolver | 国际化区域解析 | AcceptHeaderLocaleResolver |
| FlashMapManager | 重定向一次性参数暂存 | SessionFlashMapManager |

**易错**：`@ResponseBody` 的 JSON 输出不经过 ViewResolver——返回 JSON 的接口"视图组件不存在"是正常架构（前后端分离下 ViewResolver 只服务服务端模板场景）。

## 1.2 参数解析与返回值处理

### 1.2.1 Controller 获取请求数据有哪些注解？分别对应什么位置？

| 注解 | 数据来源 | 备注 |
|------|---------|------|
| @RequestParam | query string / 表单字段 | required、defaultValue；文件参数用 MultipartFile |
| @PathVariable | URL 模板占位 `/users/{id}` | 路径变量 |
| @RequestBody | HTTP 请求体 | HttpMessageConverter 反序列化（不走 DataBinder） |
| @RequestHeader | 请求头 | name 可省略匹配头名 |
| @CookieValue | Cookie | |
| @MatrixVariable | 路径矩阵参数 `/cars;color=red` | 需开启 `setRemoveSemicolonContent(false)`，用得少 |
| @ModelAttribute | 对象字段与请求参数绑定 | query/表单 → POJO，见 1.2.3 |
| @SessionAttribute / @RequestAttribute | 会话/请求属性 | 显式声明依赖，替代裸用 HttpServletRequest |

**隐式参数**：方法签名直接写 `HttpServletRequest/HttpServletResponse/HttpSession/Model/RedirectAttributes/Principal` 均可注入，无需注解（ArgumentResolver 内置支持）。

### 1.2.2 @RequestParam 和 @PathVariable 分别用于什么场景？

- **@PathVariable**：URL **资源标识**——`/users/{userId}/orders` 里 userId 是"定位资源"的一部分，REST 风格；
- **@RequestParam**：**过滤/分页/排序/可选条件**——`?page=1&status=active`，不改变资源定位语义。

```java
@GetMapping("/users/{userId}/orders")
public List<Order> orders(@PathVariable Long userId,
                          @RequestParam(defaultValue = "1") int page,
                          @RequestParam(required = false) String status) { ... }
```

**易错**：@RequestParam 默认 required=true，缺参直接 400；@PathVariable 同理。路径变量拿不到值先检查是否被网关重写 URL。参数名解析依赖 `-parameters` 编译标记（Boot 默认开），否则编译后字节码丢失形参名。

### 1.2.3 @ModelAttribute 有哪些用法？

1. **标方法参数**：请求参数（query/表单）按字段名绑定到对象并放入 Model（key=类名首字母小写或 value 指定）。`@ModelAttribute("user") User user` 可省——不标注解的 POJO 参数默认就走这个处理器；
2. **标方法**（同 Controller 内）：任意方法标注后，**每个请求执行 Controller 方法前**先调用它，返回值入 Model——做"下拉选项预载"：

```java
@ModelAttribute("genders")
public List<String> genders() { return List.of("男", "女"); }
```

**绑定行为**：按 setter 名逐字段绑定，类型不符进 BindingResult 不抛异常；配合 `bindingResult.hasErrors()` 判校验结果；全字段绑定有**Mass Assignment 风险**（攻击者加 `enabled=true` 字段提权）——用 `setAllowedFields/disallowedFields`（@InitBinder）或改用 DTO 白名单字段。

### 1.2.4 HandlerMethodArgumentResolver 的作用？如何自定义？

**作用**：把"HTTP 请求的某个位置"翻译成"Controller 方法的一个参数"——内置 40+ 种（RequestParamMethodArgumentResolver、PathVariableMethodArgumentResolver、RequestResponseBodyMethodProcessor、ModelAttributeMethodProcessor、ServletRequestArgumentResolver…），按 `supportsParameter(MethodParameter)` 逐个询问，**第一个认领的生效**。

自定义示例——登录态注入 `@CurrentUser User user`：

```java
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter p) {
        return p.hasParameterAnnotation(CurrentUser.class)
            && User.class.isAssignableFrom(p.getParameterType());
    }
    @Override
    public Object resolveArgument(MethodParameter p, ModelAndViewContainer mvc,
                                  NativeWebRequest webRequest, WebDataBinderFactory f) {
        return userService.byToken(webRequest.getHeader("Authorization"));
    }
}
// 注册：WebMvcConfigurer#addArgumentResolvers 追加
```

**价值**：把"每个方法重复解析 token/用户/租户上下文"从业务代码里消灭——写框架级扩展、Controller 方法保持纯净。自定义解析器排在内置链尾（不覆盖默认行为），注解驱动是最安全的接入方式。

### 1.2.5 Controller 返回值是怎么被处理的？ResponseEntity 和 @ResponseBody 怎么选？

返回值由 **HandlerMethodReturnValueHandler** 分发：ModelAndView、String（视图名，`redirect:`/`forward:` 前缀特殊处理）、@ResponseBody/ResponseEntity 走消息转换器、View、HttpEntity、Callable/DeferredResult（异步）、`ResponseHeader`、`ProblemDetail`（Boot3）……

- **@ResponseBody**：序列化返回体即可、状态码靠 @ResponseStatus 或全局默认——标准 REST 接口用；
- **ResponseEntity**：需要**动态状态码/响应头/Location** 时用——创建资源返回 201 + Location：

```java
@PostMapping("/user")
public ResponseEntity<User> create(@RequestBody User user) {
    User saved = userService.save(user);
    URI loc = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(saved.getId()).toUri();
    return ResponseEntity.created(loc).body(saved);   // 201 + Location
}
```

**注意**：返回 null 配 @ResponseBody 会输出空 body（204?实际 200 空），明确语义改 `ResponseEntity<Void> noContent()`。

## 1.3 数据绑定与类型转换

### 1.3.1 Spring MVC 的数据绑定（DataBinder）流程是怎样的？

**适用面**：@ModelAttribute/表单/query 参数 → POJO 的绑定；**@RequestBody 的 JSON 反序列化走 HttpMessageConverter，不走 DataBinder**（高频误区）。

流程：

1. ArgumentResolver 确定目标对象（必要时先实例化）；
2. 创建 **WebDataBinder**：参数名值对 ↔ 对象属性；
3. 逐属性填充：set 方法反射调用，**ConversionService/PropertyEditor** 完成 String→目标类型转换（"123"→Integer、"2024-01-01"→LocalDate）；
4. 错误记录进 BindingResult（不立刻抛）；
5. 有 @Valid 则触发 JSR-303 校验，错误同样入 BindingResult；
6. 方法参数紧随 BindingResult/Errors 时，错误注入其中，否则校验失败直接抛 MethodArgumentNotValidException/BindException。

扩展点：

```java
@InitBinder                       // Controller 级：绑定预处理
public void initBinder(WebDataBinder binder) {
    binder.registerCustomEditor(Date.class,
        new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
    binder.setAllowedFields("name", "age");   // Mass Assignment 白名单
}
```

全局方案：`WebBindingInitializer` Bean；日期格式优先 `@DateTimeFormat(pattern="yyyy-MM-dd")` 或 Boot `spring.mvc.format.date`——比手写 Editor 优雅。

### 1.3.2 PropertyEditor 和 ConversionService 有什么区别？

| 维度 | PropertyEditor（JDK 老机制） | ConversionService（Spring 3+） |
|------|------------------------------|-------------------------------|
| 定位 | BeanInfo 规范，单例非线程安全 | Spring 通用转换体系，**线程安全** |
| 方向 | String ↔ Object 为主 | 任意类型互转（Object→Object 也行） |
| 注册 | `registerCustomEditor` | 实现 `Converter<S,T>`/`ConverterFactory`/`GenericConverter` 注册进 ConversionService |
| 格式化 | 无概念 | Formatter SPI（`@DateTimeFormat` 背后就是它） |

WebDataBinder 两者都兼容，**优先走 ConversionService**；新代码一律 Converter/Formatter（可全局复用、可注解驱动），PropertyEditor 只在兼容遗留配置时遇到。Boot 默认注册 `ApplicationConversionService`（含 String→枚举/InetSocketAddress 等）。

# 2. 消息转换与视图

## 2.1 HttpMessageConverter 与内容协商

### 2.1.1 HttpMessageConverter 的工作原理？如何自定义？

**职责**：HTTP 报文体 ↔ Java 对象的序列化桥梁——请求侧 `@RequestBody` 用 `read()` 反序列化，响应侧 `@ResponseBody` 用 `write()` 序列化。

**选择机制**：遍历注册的转换器列表（ByteArray、String、Resource、Jackson JSON、XML、Form 等默认集），依次问三件事：**能否支持该媒体类型（可被 produces/Accept 约束）→ 能否支持该 Class（`canRead/canWrite`）**，第一个全命中的胜出。没命中→415/406。

自定义（继承 AbstractHttpMessageConverter）：

```java
public class ProtobufHttpMessageConverter extends AbstractHttpMessageConverter<Message> {
    public ProtobufHttpMessageConverter() { super(MediaType.APPLICATION_OCTET_STREAM); }
    @Override protected boolean supports(Class<?> c) { return Message.class.isAssignableFrom(c); }
    @Override protected Message readInternal(Class<? extends Message> c, HttpInputMessage in) throws IOException {
        return Message.parseFrom(in.getBody());
    }
    @Override protected void writeInternal(Message msg, HttpOutputMessage out) throws IOException {
        msg.writeTo(out.getBody());
    }
}
```

注册二选一：`configureMessageConverters()`（**清空默认**只留你加的）vs `extendMessageConverters()`（默认基础上追加）——多数场景用后者；Boot 更推荐定制 ObjectMapper（Jackson 25 题）而不是换整个转换器。

### 2.1.2 内容协商（Content Negotiation）的工作机制？

**目标**：同一资源按客户端期望返回 JSON/XML 等格式。`ContentNegotiationManager` 按**注册顺序**收集候选媒体类型：

1. **固定声明**：`@RequestMapping(produces="application/json")` ——最硬约束；
2. **URL 参数**：`?format=json`（需显式开启 favorParameter）；
3. **路径后缀**：`.json`/`.xml`（Spring 4.3+ 默认关闭，6.0 移除——路径后缀可被伪造，安全争议大）；
4. **Accept 头 + q 权重**：`Accept: application/xml;q=0.9, application/json;q=0.8` → 选 XML。

最终在候选集合里挑"客户端想要 ∩ 服务器 produces ∩ 存在对应 Converter"的交集最优者；无交集返回 **406 Not Acceptable**。

**Boot 实践**：前后端分离默认只注册 JSON，内容协商形同虚设；API 网关/开放平台提供多格式时才真正启用。

### 2.1.3 @RequestMapping 的 produces 和 consumes 有什么用？

- **consumes**：约束**请求体 Content-Type**——`consumes="application/json"` 只接受 JSON 请求体，不符返回 **415 Unsupported Media Type**；
- **produces**：约束**响应 Content-Type**——参与内容协商，客户端 Accept 不匹配返回 **406 Not Acceptable**；还用于**同 URL 按类型路由到不同方法**（`/orders` produces=JSON → jsonHandler，produces=XML → xmlHandler）。

**易混**：produces 不等于"强制转换格式"——它声明"我能提供什么"，真正序列化仍由 Converter 链完成；乱码问题常因缺 `;charset=UTF-8`（Boot 默认已带）。

### 2.1.4 视图解析（ViewResolver）的流程？前后端分离下还需要吗？

流程：逻辑视图名（"user/list"）→ ViewResolver 链按 order 解析 → 得到 View（或 null 交给下一个）→ `view.render(model, request, response)` 输出。

常用实现：`InternalResourceViewResolver`（prefix+suffix 拼 JSP/HTML 路径，转发渲染）、ThymeleafViewResolver、FreeMarkerConfigurer；`BeanNameViewResolver` 按视图 beanName 查找（配合 MappingJackson2JsonView 可做"视图即组件"的 JSONP/CSV 输出）。

**结论**：REST API 场景（@ResponseBody）**完全绕过视图链**；服务端渲染/混合页面场景仍需要。RequestToViewNameTranslator 只在"返回 void 且没指定视图名"时按 URL 推断视图名，现代项目少见。

# 3. 拦截器、过滤器与异常处理

## 3.1 Filter 与 Interceptor

### 3.1.1 Filter 和 Interceptor 有什么区别？

| 维度 | Filter | HandlerInterceptor |
|------|--------|--------------------|
| 规范/归属 | Servlet 规范，容器层 | Spring MVC，DispatcherServlet 内 |
| 生效范围 | **所有请求**（含静态资源、错误页、非 Spring 资源） | 仅命中 Handler 的请求（静态资源走 ResourceHttpRequestHandler 也可能被拦） |
| 时序 | Filter → DispatcherServlet → Interceptor → Controller | 晚于 Filter |
| 能力 | 只有 request/response 裸对象 | 拿得到 **handler（具体方法）**、ModelAndView、Spring 容器 Bean |
| 中断方式 | 不调 `chain.doFilter()` | preHandle 返回 false |
| 典型用途 | 编码、鉴权（Security 过滤器链）、XSS 清洗、CORS 响应头 | 登录态细校验、接口耗时埋点、审计日志 |

**链上协作**：Filter 适合"与 Spring 无关的协议层处理"；Interceptor 适合"知道是哪个 Controller 方法后做的事"。Boot 注册 Filter：`FilterRegistrationBean`（可配 order/urlPattern，@ServletComponentScan+@WebFilter 也可）。

### 3.1.2 Interceptor 的生命周期方法有哪些？多个拦截器如何排序？

单个请求三个钩子：

1. `preHandle(req, res, handler)`：Controller 执行前；**false 中断**，剩余拦截器与 Controller 都不执行；
2. `postHandle(req, res, handler, mav)`：Controller 执行后、**视图渲染前**（可改 ModelAndView）；@ResponseBody 场景 mav 为 null，改响应体力不从心；异步请求该阶段在结果返回后才触发；
3. `afterCompletion(req, res, handler, ex)`：视图渲染后**必然执行**（哪怕异常；但 preHandle 返回 false 时自己不会被调），做资源释放与全链路计时。

**多拦截器**：注册顺序正序 preHandle → 逆序 postHandle/afterCompletion（洋葱模型，同 AOP）。`Ordered`/`@Order` 参与排序。

### 3.1.3 如何自定义拦截器并配置拦截路径？

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        String token = req.getHeader("Authorization");
        if (token == null || !JwtUtil.verify(token)) {
            res.setStatus(401); return false;      // 或统一抛 UnauthorizedException
        }
        return true;
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private AuthInterceptor authInterceptor;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login", "/api/register", "/actuator/**")
                .order(1);
    }
}
```

**实践要点**：拦截器是普通 Bean 可注入依赖；放行名单用 exclude 集中管理，别在 preHandle 里写 if-else 路径；"从 header 拿用户上下文"后**务必放 request attribute 或 ThreadLocal 并在 afterCompletion 清理**（线程池复用防串号，参照 1_java.md 3.4 ThreadLocal 坑）。

### 3.1.4 Filter、Interceptor、AOP 三者有什么区别？如何选择？

| 维度 | Filter | Interceptor | AOP |
|------|--------|-------------|-----|
| 层级 | Servlet 容器 | Spring MVC | Spring Bean 代理层 |
| 拦截对象 | 一切进出容器的请求 | Controller 方法前后 | **任意 Spring Bean 的方法**（含 Service/Repository） |
| 粒度 | 协议级（request/response） | 知道 handler 和视图 | 方法级、参数/返回值可改 |
| 生效前提 | 容器支持 | 必须走 DispatcherServlet | 必须走代理（自调用失效，见 7_spring.md 4.3.2） |
| 典型用途 | 编码/安全头/XSS/压缩 | 登录校验、耗时统计 | 事务、日志、权限、缓存 |

**选型口诀**：动请求/响应的进 Filter；"哪个接口"级别的横切进 Interceptor；业务方法级别的横切用 AOP。

## 3.2 统一异常处理

### 3.2.1 Spring MVC 如何处理异常？@ControllerAdvice + @ExceptionHandler 的原理？

**HandlerExceptionResolver 链**（按 order）：

1. `ExceptionHandlerExceptionResolver`：找 **@ExceptionHandler**——先在出错的 Controller 类内找，再遍历所有 **@ControllerAdvice** 类（@Order 决定先后）；匹配规则：异常继承树**就近优先**（BusinessException 处理器压过 Exception 处理器）；
2. `ResponseEntityExceptionHandler`：专门映射 MVC 标准异常（400/404/405/406…）为 ProblemDetail/ResponseEntity；
3. `DefaultHandlerExceptionResolver`：把标准异常转 servlet `sendError`（容器错误页）；
4. 全没命中 → 异常抛回容器 → Boot 的 `/error`（BasicErrorController 白页/JSON）。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)   // @RequestBody 校验失败
    public Result<?> valid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ":" + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return Result.error(400, msg);
    }
    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result<?> biz(BusinessException e) { return Result.error(e.getCode(), e.getMessage()); }
    @ExceptionHandler(Exception.class)
    public Result<?> other(Exception e) {
        log.error("未处理异常", e);                     // 兜底必打堆栈
        return Result.error(500, "服务器内部错误");      // 对外不泄露细节
    }
}
```

`@RestControllerAdvice` = @ControllerAdvice + @ResponseBody。**参数可注入**：异常对象、HttpServletRequest、MethodArgumentNotValidException 的 BindingResult 等。

**易错**：@ExceptionHandler 只对"**已经抛到 MVC 层**"的异常生效——Interceptor 里抛的异常也能进（afterCompletion 之前），但**过滤器里抛的进不了**（在 DispatcherServlet 外）；`@ExceptionHandler(Exception.class)` 若吞掉 404/405 等状态语义会破坏 REST 契约，优先放行 ResponseEntityExceptionHandler 已覆盖的类型。

### 3.2.2 @ResponseStatus 和 ProblemDetail（RFC 7807）是什么关系？

- **@ResponseStatus**：标在**异常类**（"抛这个异常→HTTP 状态码 X"）或 @ExceptionHandler 方法上，最简单粗暴的状态映射，无响应体细节。
- **ProblemDetail**（Spring 6/Boot 3）：`application/problem+json` 标准错误体——type/title/status/detail/instance + 扩展字段，替代各家自造 `Result.error(code,msg)`：

```java
@ExceptionHandler(ValidationException.class)
public ProblemDetail invalid(ValidationException e) {
    ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, e.getMessage());
    pd.setTitle("参数校验失败");
    pd.setProperty("errors", e.getFieldErrors());
    return pd;
}
```

**选型**：内部统一 Result 结构延续现状即可；对外开放 API 建议向 ProblemDetail 靠——状态码语义化 + 标准化错误体，网关/SDK 友好。

# 4. 校验、跨域、上传与 REST 设计

## 4.1 参数校验

### 4.1.1 @Valid 和 @Validated 的区别？校验注解有哪些？

**结论**：两者都触发 **JSR-380（Bean Validation）** 校验，实现由 Hibernate Validator 提供；差别在能力面。

| 维度 | @Valid | @Validated |
|------|--------|-----------|
| 来源 | JDK 标准（javax/jakarta.validation） | Spring 扩展 |
| 分组校验 | 不支持 | 支持 `groups = X.class` |
| 嵌套级联 | `@Valid` 标在字段上才级联 | 同左（级联仍靠 @Valid） |
| 作用位置 | 方法参数、字段 | 参数、**类上**（方法级校验代理） |

**失败异常的路径**（高频追问）：
- `@RequestBody` 参数校验失败 → **MethodArgumentNotValidException**（400）；
- 表单/ query 绑定对象（@ModelAttribute）失败 → **BindException**；
- 类上 @Validated + 方法参数约束（`@Validated` 触发 MethodValidationPostProcessor 代理）→ **ConstraintViolationException**。
统一异常处理要把三种都接住（见 3.2.1）。

常用注解速查：

| 注解 | 语义 | 备注 |
|------|------|------|
| @NotNull / @NotBlank / @NotEmpty | 非 null / 非空白串 / 非空集合 | 三者区别是经典追问 |
| @Size(min,max) | 字符串/集合长度 | |
| @Min/@Max/@DecimalMin/@DecimalMax | 数值范围 | |
| @Pattern(regexp) | 正则 | |
| @Email/@URL/@Past/@Future/@CreditCardNumber | 语义化格式 | |
| @Positive/@Negative 系列 | 正负数 | |
| @AssertTrue/@AssertFalse | 布尔方法断言 | 跨字段规则可用自定义 @ScriptAssert（HV 扩展） |

```java
@PostMapping("/users")
public Result<?> add(@RequestBody @Valid UserCreateDTO dto) { ... }

public class UserCreateDTO {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20)
    private String name;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式错误")
    private String phone;

    @Min(0) @Max(150)
    private Integer age;
}
```

**分组校验**（新增必填 id、修改必带 id 之类的差异规则）：

```java
public interface OnCreate {}
public interface OnUpdate {}

public class UserDTO {
    @NotNull(groups = OnUpdate.class)
    private Long id;
    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String name;
}

// 接口上按组触发
@PostMapping("/create")
public Result<?> create(@RequestBody @Validated(OnCreate.class) UserDTO dto) {...}
```

注意：**Default 组不会级联**——指定 groups 后不带 `Default.class` 的话，未标 groups 的约束全部失效，通常写 `groups = {Default.class, OnCreate.class}`。

### 4.1.2 如何自定义校验注解？

三步：定义注解 + 实现 `ConstraintValidator<A, T>` + 绑定。

```java
@Target({METHOD, FIELD, ANNOTATION_TYPE, PARAMETER})
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = PhoneValidator.class)
public @interface ChinaMobile {
    String message() default "{invalid.phone}";   // 支持 i18n 资源键
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<ChinaMobile, String> {
    private Pattern pattern;
    @Override
    public void initialize(ChinaMobile a) { pattern = Pattern.compile("^1[3-9]\\d{9}$"); }
    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null || value.isEmpty()) return true;   // null 交给 @NotBlank
        return pattern.matcher(value).matches();
    }
}
```

**要点**：

1. `message/groups/payload` 三个属性是规范强制的（三件套少一个启动报错）；
2. **isValid 里 null 返回 true**——"非空"语义留给 @NotNull，保持注解职责单一（Hibernate Validator 约定）；
3. 跨字段校验：注解标在**类**上，validator 校验整个对象（如"开始时间早于结束时间"），并用 `ctx.buildConstraintViolationWithTemplate(...).addPropertyNode("endTime")` 把错误挂到具体字段；
4. 复用第三方库的 validator 可直接在注解里 `@Constraint(validatedBy = {})` + 实现 `Constraint` 组合注解（`@List` 式组合）。

### 4.1.3 校验应该放在 Controller、Service 还是 DB？

**分层职责**：

1. **Controller 层（格式校验）**：非空、长度、正则、范围——快速失败挡掉坏请求，Bean Validation 主战场；
2. **Service 层（业务规则校验）**：唯一性、库存是否够、状态机是否允许流转——与领域模型绑定，不依赖 HTTP 上下文；
3. **数据库（最后防线）**：唯一索引、外键、CHECK 约束兜底并发下的最终一致（应用层"先查后插"防不住并发，唯一索引才防得住）。

> **易错**："只在 Service 校验"的接口没有格式防御，脏参数打到 DB 才报错；"只在 Controller 校验"的 Service 被 RPC/MQ 消费者复用时无保护——**校验属于领域，格式属于边界**，两层都要有但内容不同。

## 4.2 跨域（CORS）

### 4.2.1 什么是跨域？Spring 提供哪几种解决方式？

**同源策略**：协议 + 域名 + 端口三者相同才允许脚本读取跨源响应。跨域请求**会发出去**，只是响应被浏览器拦截——这是"发了收不到"而不是"没发"。

**Spring MVC 三种方案**：

1. **@CrossOrigin**（Controller/方法级）：
   `@CrossOrigin(origins = "https://a.com", methods = RequestMethod.GET)`——简单但无法集中管理；
2. **全局 WebMvcConfigurer**（推荐单体）：

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
            .allowedOriginPatterns("https://*.example.com")   // 带凭证时用 * 的替代
            .allowedMethods("GET","POST","PUT","DELETE","OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);        // 预检缓存秒数，减少 OPTIONS 轰炸
}
```

3. **CorsFilter**（Filter 级）：Spring Cloud Gateway 场景或要在 **Servlet 容器层（含 404/错误页/非 MVC 资源）** 也带 CORS 头时用——Filter 优先级高于 MVC 异常处理，"错误响应也要能跨域读"就靠它。

底层同一套：`CorsConfiguration` + `DefaultCorsProcessor` 负责拼响应头（`Access-Control-Allow-Origin` 等）。

### 4.2.2 简单请求和预检请求的区别？allowCredentials 怎么配？

**预检触发条件**（任一满足即先发 `OPTIONS` 询问）：方法非 GET/HEAD/POST；自定义头（Authorization、token）；Content-Type 是 application/json 等三类之外……实际项目里 **带 Bearer token 的 JSON 请求 100% 走预检**。

预检往返：浏览器发 `Origin: https://a.com` + `Access-Control-Request-Method` + `Access-Control-Request-Headers` → 服务端回 `Allow-Methods/Allow-Headers/Max-Age` → 通过后浏览器才发真实请求。**maxAge 缓存预检结果**，没配它会每个请求多一次 RTT。

**凭证规则**：`allowCredentials(true)` 时 `Access-Control-Allow-Origin` **不能为 `*`**（规范禁止），必须回具体 Origin——配 `allowedOriginPatterns` + Spring 动态回显；同时前端要 `fetch(..., {credentials:'include'})`。响应头 `Vary: Origin` 要留意——CDN 缓存了"给 A 站的响应"复用在 B 站会出安全问题，Spring 的 `DefaultCorsProcessor` 会自动加 Vary 防御。

### 4.2.3 前后端分离登录态怎么跨域维持？Cookie 不通了怎么办？

1. **Cookie 方案**：`allowCredentials` + 固定 Origin + 前端 credentials:include + SameSite；但**顶级域名不同**（a.com 与 b.com）时 Cookie 域无法互认——跨主域场景 Cookie 天然不可用；
2. **Token 方案（主流）**：登录后发 JWT/自管 token，存 localStorage/sessionStorage，请求带 `Authorization: Bearer xxx` 头——与 Cookie 解耦，网关/移动端复用同一协议；后端配合：CORS 放行 Authorization 头 + 拦截器解析 token（3.1.3）；
3. **同域代理**：Nginx 把 `/api` 反代到后端——浏览器视角根本没跨域，最省心的生产部署形态（前端开发环境用 devServer proxy 同理）。

## 4.3 文件上传

### 4.3.1 Spring MVC 文件上传的实现原理？如何接收大文件？

**解析链路**：multipart/form-data 请求 → **MultipartResolver** 提前把 HttpServletRequest 包装成 **MultipartHttpServletRequest**（Boot 默认 `StandardServletMultipartResolver`，走 Servlet 3.0 原生 `request.getParts()`；老方案 CommonsMultipartResolver 已随 Spring 5 移除）→ Controller 用 `@RequestPart`/`MultipartFile` 直接绑定文件部分。

```java
@PostMapping("/upload")
public Result<?> upload(@RequestPart("file") MultipartFile file,
                       @RequestPart("meta") FileMetaDTO meta) throws IOException {
    if (file.isEmpty()) return Result.error("空文件");
    Path target = uploadDir.resolve(UUID.randomUUID() + ext(file));
    file.transferTo(target);        // 落盘
    return Result.ok(meta);
}
```

**配置与坑**：

1. Boot 默认 `spring.servlet.multipart.enabled=true`、单文件/总大小上限 **1MB**——大文件必调 `max-file-size`/`max-request-size`；Servlet 容器层 Tomcat `maxSwallowSize` 与 `connection.multipart` 也要匹配，Nginx 还有 `client_max_body_size`（413 报错先查它）；
2. 超过阈值时解析在**进入 Controller 之前**就抛 `MaxUploadSizeExceededException`——异常处理要在 @ControllerAdvice 接住；
3. 临时文件落磁盘（`java.io.tmpdir`），transferTo 之后要**移动或改名**，同名覆盖有安全隐患；
4. 安全清单：校验扩展名白名单 + Magic Number、重命名存储、存对象存储（OSS/S3）而非应用磁盘、回显下载链接不带原文件名。

> **追问**：大文件分片上传怎么做？——前端切片（每片独立请求）+ 服务端按 `uploadId+chunkIndex` 缓存、全部到齐调"合并"接口；秒传 = 先查文件 hash；断点续传 = 问服务端已有分片列表。本质是业务协议设计，与 MultipartResolver 无关。

### 4.3.2 如何设计文件下载接口？

```java
@GetMapping("/files/{id}")
public ResponseEntity<InputStreamResource> download(@PathVariable Long id) {
    StoredFile f = fileService.get(id);
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + f.safeName() + "\"; filename*=UTF-8''" + enc(f.name()))
        .body(new InputStreamResource(f.stream()));
}
```

**细节**：`Content-Disposition` 中文文件名要 URL 编码 + `filename*`（RFC 5987）兼容老浏览器；静态资源直接 `ResourceHttpRequestHandler`/`/error` 之外的 `webjars`、`classpath:/static/` 即可，不必手写流拷贝；Range 请求（断点续传/视频拖动）Spring MVC 原生支持（`ResponseEntity<Resource>` 返回 Resource 时自动处理 Range 头）。

## 4.4 RESTful 设计

### 4.4.1 RESTful API 的设计原则？@RequestMapping 风格怎么算规范？

**REST 六约束的落地口径**（面试答这四条即可）：

1. **资源为名词、URL 表层级**：`/users/{id}/orders`，动词不进路径（`/getUserList` 不是 REST）；
2. **HTTP 方法表操作语义**：GET 查（幂等安全）、POST 建、PUT 全量改（幂等）、PATCH 局部改、DELETE 删（幂等）；
3. **状态码表达结果**：200/201(+Location)/204、400/401/403/404/405/409/422、500；不要"永远 200 + body 里 code"；
4. **无状态**：服务端不存会话上下文，认证走 token——天然利于水平扩展。

Spring MVC 对应习惯：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping("/{id}")                 // 200 详情
    public User get(@PathVariable Long id) {...}
    @GetMapping                          // 200 列表 + 分页参数
    public Page<User> list(@RequestParam int page) {...}
    @PostMapping                         // 201 + Location（见 1.2.5）
    public ResponseEntity<User> create(@Valid @RequestBody UserDTO dto) {...}
    @PutMapping("/{id}")                 // 204 全量更新
    @PatchMapping("/{id}")               // 部分更新
    @DeleteMapping("/{id}")              // 204
}
```

**语义陷阱**：GET 不写副作用（爬虫/预取会乱触发）；PUT 幂等要求"重复提交结果一致"——实现上要么覆盖写要么乐观锁（见 16_mysql.md）；版本化 `/api/v1/...` 与兼容性策略比"过度 REST 纯洁性"更重要。

# 5. 进阶特性

## 5.1 异步请求处理

### 5.1.1 Spring MVC 如何支持异步请求？Callable 和 DeferredResult 的区别？

**动机**：同步模型下"慢调用（RPC/大查询）占着 Servlet 线程干等"——Tomcat 200 线程池被 I/O 等待拖死。异步处理**先释放容器线程**，结果就绪再拿回来渲染。

实现依赖 Servlet 3.0 `AsyncContext`。两种入口：

**Callable（简单场景）**：

```java
@GetMapping("/report")
public Callable<String> report() {
    return () -> heavyReportService.build();   // 提交给 AsyncTaskExecutor 执行
}
// 返回 "done" 字符串后经正常消息转换器写响应
```

MVC 内部用 `taskExecutor`（默认 `SimpleAsyncTaskExecutor`——**每次新建线程，生产必须换成 ThreadPoolTaskExecutor**）跑任务，完成后再 dispatch 回 `/async` 续跑响应。

**DeferredResult（复杂/回调场景）**：

```java
@GetMapping("/order/{id}/pay-status")
public DeferredResult<String> status(@PathVariable Long id) {
    DeferredResult<String> dr = new DeferredResult<>(3000L, "timeout");
    payService.watch(id, result -> dr.setResult(result)); // 外部线程/回调完成时 set
    return dr;
}
```

适合结果由**另一个线程/中间件**决定：MQ 回执、异步网关回调、长轮询、SSE 轮询模拟。

**区别**：Callable = 服务端自己"起线程算一个返回值"；DeferredResult = "线程随便谁，我只要往容器里 set 结果"——后者可跨请求/跨线程管理（配合 `onTimeout/onCompletion` 清理注册表，做推送网关的标准姿势）。异步结束后会**重新进入 DispatcherServlet**（拦截器链会再走一遍），`WebAsyncManager` 负责挂接。

> **追问**：Boot 默认 `spring.mvc.async.request-timeout`（-1 无限）；异步链路里 ThreadLocal 上下文（用户、租户）会丢——要么用 DeferredResult 回调显式传递，要么用可继承的上下文传递工具（见 1_java.md 3.4）。

### 5.1.2 SSE 和 WebSocket 在 Spring MVC 里怎么做？

- **SSE**：`produces = MediaType.TEXT_EVENT_STREAM_VALUE` 返回 `SseEmitter`，超时/完成回调管理连接集合——单向推送首选（行情、进度条），走普通 HTTP 兼容网关；
- **WebSocket**：MVC 只做握手（`HandshakeHandler`/`WebSocketHandler` 注册），消息协议属于容器/独立模块——细节见 36_websocket.md。

## 5.2 静态资源与路径匹配

### 5.2.1 静态资源如何处理？资源映射的机制？

`initStrategies` 里 HandlerMapping 链包含 **SimpleUrlHandlerMapping（资源）**：默认由 `ResourceHandlerRegistry` 配置——

```java
registry.addResourceHandler("/static/**")
        .addResourceLocations("classpath:/static/", "file:/data/uploads/")
        .setCachePeriod(3600)
        .resourceChain(true);          // 启用 ResourceResolver/Transformer 链（版本指纹、gzip）
```

**Boot 默认**：`classpath:/static|public|resources|META-INF/resources`、`/webjars/**` 自动映射；命中静态资源走 `ResourceHttpRequestHandler`，**支持 Last-Modified/ETag/Range** 且跳过 @ResponseBody 流程。

常见坑：① Controller 映射 `/api/**` 兜底 `/**` 会吃掉静态资源；② 拦截器默认也拦资源 handler（exclude 路径）；③ CDN/缓存导致更新不生效——`version()` 资源指纹 + 短 maxAge。

### 5.2.2 AntPathMatcher 和 PathPattern 有什么区别？

| 维度 | AntPathMatcher（传统） | PathPattern（Spring 5.3+ 默认） |
|------|------------------------|--------------------------------|
| 匹配 | 分段解析、逐段比较 | 预编译成 token 状态机，**匹配更快、分配更少** |
| `**` 语义 | 位置随意 | 显式约束（只允许在末尾/特定段），歧义更少 |
| 捕获 | 无 | `PathContainer` 解析一次可复用拿变量 |
| 开关 | — | `spring.mvc.pathmatch.matching-strategy=path-pattern-parser`（Boot 2.6 起默认） |

面试点：改默认后少数自定义 Matcher/老写法（`/**/*` 中间 `**`）不兼容报错——排查"升级后路径突然 404" 先看 matching-strategy；PathPattern 不支持正则段（要正则仍回 Ant）。

## 5.3 转发、重定向与一次性消息

### 5.3.1 forward 和 redirect 的区别？`forward:` / `redirect:` 前缀怎么用？

| 维度 | forward（转发） | redirect（重定向） |
|------|-----------------|---------------------|
| 实现 | 服务端内部 `RequestDispatcher.forward` | 302/303 + `Location` 头，浏览器再发一次请求 |
| 浏览器 URL | 不变 | 变成新地址 |
| 请求次数 | 1 | 2（新请求，**request 属性丢**） |
| 数据传递 | 同一 request，attribute 照用 | 只能 URL 参数或 FlashMap/会话 |
| 跨域能力 | 否（应用内） | 是（任意 URL） |

MVC 写法：`return "forward:/other/path"`（仍在当前 Servlet 内重新走资源路径）、`return "redirect:/list"`（Post-Redirect-Get 防重复提交的标准方案）、`return "redirect:https://..."` 跨站。

**PRG 模式**：POST 提交成功 → `redirect:` 到列表页——刷新不再重复提交；POST 里的校验错误/提示消息靠 **FlashMap** 跨重定向传递：

```java
public String create(@Valid User u, BindingResult br, RedirectAttributes ra) {
    if (br.hasErrors()) return "user/form";
    repo.save(u);
    ra.addFlashAttribute("msg", "创建成功");   // 存 FlashMapManager（默认 session），重定向后取一次即清
    return "redirect:/users";
}
```

FlashMap 原理：请求结束时按"目标路径 + 查询条件"匹配存入会话，下一个请求匹配取出并删除——**一次性**。`@SessionAttribute`/`@SessionAttributes` 是"常驻会话属性"的声明式包装，别混用场景。

### 5.3.2 Spring MVC 和 Struts2 的区别？（了解即可）

| 维度 | Spring MVC | Struts2 |
|------|-----------|---------|
| 入口 | 单 Servlet DispatcherServlet | Filter FilterDispatcher/StrutsPrepareAndExecute |
| 参数模型 | 方法级，参数绑定到方法形参 | 动作类字段（ognl 读写成员变量）→ **有状态动作需 prototype 包装** |
| 拦截器 | HandlerInterceptor | 拦截器栈 + OGNL 值栈 |
| 功能面 | REST/异步/校验/消息转换全家桶，生态延续 | 老项目维护，CVE 历史多（ognl 系列） |

一句话答法：MVC 是"方法即端点"的无状态模型 + 可插拔策略组件，Struts2 是"类即动作 + 值栈"——当年 Struts 的字段线程安全与 OGNL 注入是后者的名声包袱；新项目没有选择 Struts2 的理由。