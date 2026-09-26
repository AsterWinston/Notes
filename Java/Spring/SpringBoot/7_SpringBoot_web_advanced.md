# 1.文件上传

文件上传是Web开发中的常见需求。Spring Boot对文件上传提供了自动配置支持。

## 1.1Spring Boot文件上传的自动配置

文件上传的自动配置类是 `MultipartAutoConfiguration`，它自动配置了 `MultipartConfigElement` 和 `MultipartResolver` 组件。

对应的属性类是 `MultipartProperties`，配置前缀为 `spring.servlet.multipart`：

```java
@ConfigurationProperties(prefix = "spring.servlet.multipart")
public class MultipartProperties {
    private boolean enabled = true;       // 是否启用文件上传
    private String location;               // 上传文件的临时目录
    private String maxFileSize = "1MB";    // 单个文件最大大小
    private String maxRequestSize = "10MB";// 整个请求最大大小
    private String fileSizeThreshold = "0B"; // 文件写入磁盘的阈值
}
```

## 1.2单文件上传

### 1.2.1前端页面
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
<h2>单文件上传</h2>
<form method="post" enctype="multipart/form-data" th:action="@{/upload}">
    <input type="file" name="file" />
    <button type="submit">上传</button>
</form>
</body>
</html>
```

### 1.2.2Controller代码
```java
@Controller
public class FileUploadController {

    @PostMapping("/upload")
    @ResponseBody
    public String upload(@RequestParam("file") MultipartFile file) {
        if (file.isEmpty()) {
            return "请选择文件";
        }
        // 获取原始文件名
        String originalFilename = file.getOriginalFilename();
        // 获取文件大小
        long size = file.getSize();
        // 保存文件
        try {
            file.transferTo(new File("D:/uploads/" + originalFilename));
            return "上传成功：" + originalFilename + "，大小：" + size + " 字节";
        } catch (IOException e) {
            return "上传失败：" + e.getMessage();
        }
    }
}
```

## 1.3多文件上传

### 1.3.1前端页面
```html
<form method="post" enctype="multipart/form-data" th:action="@{/upload/multi}">
    <input type="file" name="files" multiple />
    <button type="submit">批量上传</button>
</form>
```

### 1.3.2Controller代码
```java
@PostMapping("/upload/multi")
@ResponseBody
public String uploadMulti(@RequestParam("files") MultipartFile[] files) {
    if (files == null || files.length == 0) {
        return "请选择文件";
    }
    StringBuilder result = new StringBuilder();
    for (MultipartFile file : files) {
        if (!file.isEmpty()) {
            try {
                file.transferTo(new File("D:/uploads/" + file.getOriginalFilename()));
                result.append(file.getOriginalFilename()).append(" 上传成功\n");
            } catch (IOException e) {
                result.append(file.getOriginalFilename()).append(" 上传失败\n");
            }
        }
    }
    return result.toString();
}
```

## 1.4文件上传大小限制配置
```properties
# 启用文件上传（默认true）
spring.servlet.multipart.enabled=true
# 单个文件最大大小
spring.servlet.multipart.max-file-size=10MB
# 整个请求最大大小
spring.servlet.multipart.max-request-size=50MB
```

# 2.文件下载

## 2.1使用 ResponseEntity 实现下载
```java
@RestController
public class FileDownloadController {

    @GetMapping("/download")
    public ResponseEntity<Resource> download() throws IOException {
        // 文件路径
        Path path = Paths.get("D:/uploads/report.pdf");
        // 创建资源对象
        Resource resource = new InputStreamResource(Files.newInputStream(path));

        return ResponseEntity.ok()
                // 设置Content-Type
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                // 设置Content-Disposition（触发浏览器下载）
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"" + URLEncoder.encode("报表.pdf", "UTF-8") + "\"")
                .body(resource);
    }
}
```

## 2.2使用 HttpServletResponse 实现下载
```java
@GetMapping("/download2")
public void download2(HttpServletResponse response) throws IOException {
    // 设置响应头
    response.setContentType("application/octet-stream");
    response.setHeader("Content-Disposition",
            "attachment; filename=\"" + URLEncoder.encode("数据.xlsx", "UTF-8") + "\"");

    // 获取文件输入流
    try (InputStream inputStream = new FileInputStream("D:/uploads/data.xlsx");
         OutputStream outputStream = response.getOutputStream()) {
        byte[] buffer = new byte[1024];
        int len;
        while ((len = inputStream.read(buffer)) != -1) {
            outputStream.write(buffer, 0, len);
        }
        outputStream.flush();
    }
}
```

# 3.拦截器配置

拦截器（Interceptor）是Spring MVC提供的功能，可以在请求到达Controller之前或之后执行逻辑。

## 3.1自定义拦截器
实现 `HandlerInterceptor` 接口：

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        // 在Controller方法执行前调用
        // 返回true：放行；返回false：拦截
        Object user = request.getSession().getAttribute("user");
        if (user == null) {
            response.sendRedirect("/login");
            return false;
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        // 在Controller方法执行后、视图渲染前调用
        System.out.println("postHandle 执行了...");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        // 在视图渲染完成后调用（整个请求完成后）
        System.out.println("afterCompletion 执行了...");
    }
}
```

## 3.2注册拦截器
实现 `WebMvcConfigurer` 接口，重写 `addInterceptors` 方法：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/**")          // 拦截所有路径
                .excludePathPatterns(             // 排除不需要拦截的路径
                        "/login",
                        "/register",
                        "/css/**",
                        "/js/**",
                        "/images/**"
                );
    }
}
```

## 3.3多个拦截器的执行顺序
```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    // 注册顺序就是执行顺序
    registry.addInterceptor(new FirstInterceptor());   // 先执行
    registry.addInterceptor(new SecondInterceptor());  // 后执行
}
```

执行流程：
```
FirstInterceptor.preHandle()
  → SecondInterceptor.preHandle()
    → Controller.method()
  → SecondInterceptor.postHandle()
→ FirstInterceptor.postHandle()
→ SecondInterceptor.afterCompletion()
→ FirstInterceptor.afterCompletion()
```

# 4.过滤器配置

过滤器（Filter）是Servlet规范中的组件，工作在请求进入Servlet容器、但未到达Servlet/DispatcherServlet之前。

## 4.1使用 @WebFilter 注解（需配合 @ServletComponentScan）
```java
@WebFilter(urlPatterns = "/*")
public class LoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        System.out.println("请求路径：" + httpRequest.getRequestURI());
        long start = System.currentTimeMillis();

        // 放行，继续执行后续的Filter或Servlet
        chain.doFilter(request, response);

        long end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start) + "ms");
    }
}
```

启动类上添加：
```java
@SpringBootApplication
@ServletComponentScan(basePackages = "com.example.filter")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 4.2使用 FilterRegistrationBean 注册
```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<LoggingFilter> loggingFilter() {
        FilterRegistrationBean<LoggingFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new LoggingFilter());
        registrationBean.addUrlPatterns("/*");           // 过滤所有路径
        registrationBean.setOrder(1);                     // 设置执行顺序（数字越小越优先）
        return registrationBean;
    }
}
```

## 4.3拦截器 vs 过滤器的区别

| 对比项 | 过滤器（Filter） | 拦截器（Interceptor） |
|--------|-----------------|----------------------|
| 规范 | Servlet规范 | Spring MVC |
| 作用范围 | 所有请求（包括静态资源） | 只对Controller请求生效 |
| 粒度 | 只能拿到request/response | 可以拿到Handler、ModelAndView |
| 依赖注入 | 不能直接注入Spring Bean | 可以直接注入Spring Bean |
| 生命周期 | 由Servlet容器管理 | 由Spring IOC容器管理 |

# 5.数据校验

Spring Boot支持Bean Validation（JSR-380）规范，默认使用Hibernate Validator作为实现。

## 5.1引入依赖
`spring-boot-starter-web` 已经默认引入了 `spring-boot-starter-validation`。如果单独使用需手动添加：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## 5.2常用校验注解

| 注解 | 说明 |
|------|------|
| @NotNull | 不能为null |
| @NotEmpty | 不能为null且不能为空字符串/空集合 |
| @NotBlank | 不能为null且trim后不能为空字符串 |
| @Size(min, max) | 字符串/集合长度范围 |
| @Min / @Max | 数值最小/最大值 |
| @Email | 邮箱格式 |
| @Pattern(regexp) | 正则匹配 |
| @Past / @Future | 过去/未来日期 |
| @Positive / @PositiveOrZero | 正数/非负数 |
| @Range(min, max) | 数值范围 |

## 5.3基本使用

### 5.3.1实体类添加校验注解
```java
@Data
public class User {
    private Long id;

    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度必须在2-20之间")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Size(min = 6, max = 30, message = "密码长度必须在6-30之间")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 0, message = "年龄不能小于0")
    @Max(value = 150, message = "年龄不能大于150")
    private Integer age;
}
```

### 5.3.2Controller中使用 @Valid
```java
@RestController
public class UserController {

    @PostMapping("/user/save")
    public R<String> save(@Valid @RequestBody User user, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            // 获取第一个校验错误信息
            String errorMsg = bindingResult.getFieldErrors().stream()
                    .map(FieldError::getDefaultMessage)
                    .collect(Collectors.joining(", "));
            return R.FAIL(400, errorMsg);
        }
        // 业务逻辑...
        return R.OK("保存成功");
    }
}
```

注意：
- `@Valid` 注解触发校验
- `BindingResult` 参数必须紧跟在被校验对象后面，否则Spring会直接抛出异常
- 如果不接收 `BindingResult`，校验失败时会抛出 `MethodArgumentNotValidException`

## 5.4全局异常处理校验错误
不在每个Controller中处理 `BindingResult`，使用全局异常处理器统一处理：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public R<String> handleValidationException(MethodArgumentNotValidException e) {
        String errorMsg = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return R.FAIL(400, errorMsg);
    }
}
```

这样Controller就只需写：
```java
@PostMapping("/user/save")
public R<String> save(@Valid @RequestBody User user) {
    // 校验失败会自动被全局异常处理器捕获
    return R.OK("保存成功");
}
```

## 5.5分组校验
不同场景使用不同的校验规则（如：新增时不需要id，修改时需要id）：

### 5.5.1定义分组接口
```java
public interface AddGroup {}
public interface UpdateGroup {}
```

### 5.5.2实体类指定分组
```java
@Data
public class User {

    @NotNull(message = "修改时ID不能为空", groups = UpdateGroup.class)
    private Long id;

    @NotBlank(message = "用户名不能为空", groups = {AddGroup.class, UpdateGroup.class})
    private String username;

    @NotBlank(message = "密码不能为空", groups = AddGroup.class)
    private String password;
}
```

### 5.5.3Controller切换 @Validated
```java
// 使用 spring 的 @Validated 来指定分组
@PostMapping("/user/add")
public R<String> add(@Validated(AddGroup.class) @RequestBody User user) {
    return R.OK("新增成功");
}

@PutMapping("/user/update")
public R<String> update(@Validated(UpdateGroup.class) @RequestBody User user) {
    return R.OK("修改成功");
}
```

注意：需要使用 `org.springframework.validation.annotation.Validated`，而不是 `jakarta.validation.Valid`。

## 5.6自定义校验注解

### 5.6.1定义注解
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface Phone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### 5.6.2实现校验器
```java
public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final Pattern PHONE_PATTERN = Pattern.compile("^1[3-9]\\d{9}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isEmpty()) {
            return true; // null和空串由 @NotBlank 处理
        }
        return PHONE_PATTERN.matcher(value).matches();
    }
}
```

### 5.6.3使用
```java
@Phone(message = "请输入正确的手机号")
private String phone;
```

# 6.定时任务

## 6.1启用定时任务
在启动类上添加 `@EnableScheduling` 注解：

```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 6.2使用 @Scheduled 注解
```java
@Component
public class ScheduledTasks {

    // 固定延迟（上一次执行完毕后再等3秒）
    @Scheduled(fixedDelay = 3000)
    public void task1() {
        System.out.println("task1 执行时间：" + LocalDateTime.now());
    }

    // 固定频率（每5秒执行一次，不管上次是否完成）
    @Scheduled(fixedRate = 5000)
    public void task2() {
        System.out.println("task2 执行时间：" + LocalDateTime.now());
    }

    // 初始延迟 + 固定频率
    @Scheduled(initialDelay = 2000, fixedRate = 10000)
    public void task3() {
        System.out.println("task3 执行时间：" + LocalDateTime.now());
    }

    // Cron表达式（最常用）
    // 秒 分 时 日 月 周
    @Scheduled(cron = "0 0 3 * * ?")  // 每天凌晨3点
    public void task4() {
        System.out.println("task4 执行时间：" + LocalDateTime.now());
    }
}
```

## 6.3Cron表达式说明
```
┌────────── 秒 (0-59)
│ ┌──────── 分 (0-59)
│ │ ┌────── 时 (0-23)
│ │ │ ┌──── 日 (1-31)
│ │ │ │ ┌── 月 (1-12)
│ │ │ │ │ ┌ 周 (0-7, 0和7都是周日)
│ │ │ │ │ │
* * * * * *
```

常用示例：
| 表达式 | 说明 |
|--------|------|
| `0 0/5 * * * ?` | 每5分钟 |
| `0 0 8,12,18 * * ?` | 每天8点、12点、18点 |
| `0 30 9 * * MON-FRI` | 工作日9:30 |
| `0 0 0 1 * ?` | 每月1号0点 |
| `0 0 0 L * ?` | 每月最后一天0点 |

## 6.4定时任务线程池配置
默认情况下，所有 `@Scheduled` 任务共用一个单线程池，如果有多个任务会排队执行。可以通过配置修改：

```properties
# 定时任务线程池大小
spring.task.scheduling.pool.size=5
# 线程名称前缀
spring.task.scheduling.thread-name-prefix=scheduled-
```

# 7.异步任务

## 7.1启用异步任务
在启动类上添加 `@EnableAsync` 注解：

```java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 7.2使用 @Async 注解
```java
@Service
public class AsyncService {

    // 同步方法
    public void syncMethod() {
        System.out.println("syncMethod 开始：" + LocalDateTime.now());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("syncMethod 结束：" + LocalDateTime.now());
    }

    // 异步方法
    @Async
    public void asyncMethod() {
        System.out.println("asyncMethod 开始：" + LocalDateTime.now());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("asyncMethod 结束：" + LocalDateTime.now());
    }

    // 带有返回值的异步方法
    @Async
    public CompletableFuture<String> asyncWithResult() {
        System.out.println("asyncWithResult 开始：" + LocalDateTime.now());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("asyncWithResult 结束：" + LocalDateTime.now());
        return CompletableFuture.completedFuture("异步任务结果");
    }
}
```

## 7.3获取异步任务返回值
```java
@RestController
public class AsyncController {

    @Autowired
    private AsyncService asyncService;

    @GetMapping("/async")
    public String testAsync() throws Exception {
        System.out.println("Controller开始：" + LocalDateTime.now());

        // 调用异步任务，返回Future对象
        CompletableFuture<String> future = asyncService.asyncWithResult();

        // 主线程可以继续做其他事情...

        // 获取异步任务结果（阻塞等待）
        String result = future.get();
        System.out.println("获取结果：" + result);

        return "完成：" + result;
    }
}
```

## 7.4配置异步任务线程池
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);        // 核心线程数
        executor.setMaxPoolSize(50);         // 最大线程数
        executor.setQueueCapacity(100);      // 队列容量
        executor.setKeepAliveSeconds(60);    // 线程空闲时间
        executor.setThreadNamePrefix("async-"); // 线程名前缀
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            System.out.println("异步任务异常：方法=" + method.getName() + ", 异常=" + ex.getMessage());
        };
    }
}
```

## 7.5@Async 使用注意事项
- 必须通过Spring容器获取的Bean来调用，不能直接在类内部调用（this.xxx()不走代理）
- 方法和调用方必须在不同的类中（避免AOP代理失效）
- 返回值支持void和Future<T>（以及CompletableFuture<T>）
- 异常需要自定义 AsyncUncaughtExceptionHandler 来处理
