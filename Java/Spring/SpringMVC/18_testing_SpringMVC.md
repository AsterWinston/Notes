# 1. 测试概述
```
SpringMVC提供了强大的测试支持，通过MockMvc可以在不启动Servlet容器（Tomcat）的情况下，
模拟HTTP请求来测试Controller层的各种行为。

使用MockMvc进行测试的优势：
● 无需启动服务器，测试速度快
● 无需部署到Tomcat，纯单元测试
● 可以精确控制请求参数、请求头、Session等
● 可以验证响应状态码、响应体、模型数据、视图名称等

核心依赖（SpringMVC 6已传递依赖引入）：
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>6.1.4</version>
    <scope>test</scope>
</dependency>

<!-- JUnit 5 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

# 2. 搭建MockMvc测试环境
```
MockMvc通过MockMvcBuilders来构建，支持两种方式：

方式一：独立模式（standaloneSetup）—— 不加载Spring容器
只测试指定的Controller，需要手动注册Controller和配置。

方式二：Web应用上下文模式（webAppContextSetup）—— 加载Spring容器（推荐）
加载完整的SpringMVC配置，行为最接近真实环境。

测试基类（推荐做法，抽取公共配置）：

package com.powernode.springmvc.controller;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.context.web.WebAppConfiguration;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

/**
 * 测试基类，所有Controller测试类继承此类
 */
@ExtendWith(SpringExtension.class)           // 集成Spring测试支持
@ContextConfiguration("classpath:springmvc.xml")  // 加载Spring配置文件
@WebAppConfiguration                          // 声明为Web应用上下文
public abstract class BaseControllerTest {

    @Autowired
    protected WebApplicationContext wac;

    protected MockMvc mockMvc;

    @BeforeEach
    public void setUp() {
        // 使用WebApplicationContext构建MockMvc
        mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
    }
}

注解说明：
● @ExtendWith(SpringExtension.class)：JUnit5中集成Spring测试
● @ContextConfiguration：指定Spring配置文件的位置
● @WebAppConfiguration：声明使用WebApplicationContext（而非普通的ApplicationContext）
● @BeforeEach：在每个测试方法执行之前运行（JUnit5）
```

# 3. 测试GET请求（返回视图）
```
测试一个返回Thymeleaf视图的Controller：

// 要被测试的Controller
@Controller
public class UserController {

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable("id") Integer id, Model model) {
        model.addAttribute("userId", id);
        model.addAttribute("userName", "张三");
        return "user_detail";
    }
}

测试类：
package com.powernode.springmvc.controller;

import org.junit.jupiter.api.Test;
import org.springframework.test.web.servlet.MvcResult;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

public class UserControllerTest extends BaseControllerTest {

    @Test
    public void testGetUser() throws Exception {
        mockMvc.perform(get("/user/123"))                                  // 发送GET请求
                .andExpect(status().isOk())                                 // 断言 HTTP 200
                .andExpect(view().name("user_detail"))                       // 断言视图名称
                .andExpect(model().attributeExists("userId"))                // 断言Model中有userId
                .andExpect(model().attributeExists("userName"))              // 断言Model中有userName
                .andExpect(model().attribute("userId", 123))                 // 断言userId的值
                .andExpect(model().attribute("userName", "张三"))            // 断言userName的值
                .andExpect(forwardedUrl("/WEB-INF/thymeleaf/user_detail.html")); // 断言转发路径
    }

    @Test
    public void testGetUserNotExists() throws Exception {
        // 测试用户不存在的场景
        mockMvc.perform(get("/user/999"))
                .andExpect(status().isNotFound());  // 断言返回404
    }
}

常用断言方法：
● status().isOk()                 — HTTP 200
● status().isNotFound()           — HTTP 404
● status().isBadRequest()         — HTTP 400
● status().isInternalServerError() — HTTP 500
● view().name("xxx")              — 断言逻辑视图名
● model().attributeExists("key")  — 断言Model中某个属性存在
● model().attribute("key", value) — 断言属性值
● model().attributeDoesNotExist() — 断言属性不存在
● forwardedUrl()                  — 断言转发URL
● redirectedUrl()                 — 断言重定向URL
● content().string()              — 断言响应体字符串
```

# 4. 测试POST请求（表单提交）
```

@Controller
public class UserController {

    @PostMapping("/user/save")
    public String save(@RequestParam("username") String username,
                       @RequestParam("age") Integer age,
                       Model model) {
        model.addAttribute("msg", "保存成功");
        model.addAttribute("username", username);
        return "ok";
    }
}

测试类：
@Test
public void testSaveUser() throws Exception {
    mockMvc.perform(post("/user/save")
            .param("username", "张三")      // 设置请求参数
            .param("age", "25")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED))  // 表单编码
            .andExpect(status().isOk())
            .andExpect(view().name("ok"))
            .andExpect(model().attribute("msg", "保存成功"))
            .andExpect(model().attribute("username", "张三"));
}

测试POST请求配合@Valid校验：
@Test
public void testSaveUserWithValidation() throws Exception {
    // 测试用户名过短的情况
    mockMvc.perform(post("/user/save")
            .param("username", "张")   // 用户名长度1 < 2，应触发校验失败
            .param("age", "25"))
            .andExpect(view().name("user_form"))       // 回到表单页面
            .andExpect(model().attributeHasErrors());   // Model中存在错误

    // 测试正常情况
    mockMvc.perform(post("/user/save")
            .param("username", "张三丰")
            .param("age", "25"))
            .andExpect(view().name("ok"))
            .andExpect(model().attributeHasNoErrors());
}
```

# 5. 测试RESTful API（JSON请求和响应）
```
测试返回JSON的@RestController：

@RestController
@RequestMapping("/api/user")
public class UserApiController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getById(@PathVariable("id") Long id) {
        User user = new User(id, "张三", "123456", "zhangsan@test.com");
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<User> save(@RequestBody User user) {
        user.setId(1L);  // 模拟数据库生成ID
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable("id") Long id) {
        return ResponseEntity.noContent().build();  // 204 No Content
    }
}

测试类：
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.*;
import static org.hamcrest.Matchers.*;
import static org.springframework.http.MediaType.*;

@Test
public void testGetUserApi() throws Exception {
    mockMvc.perform(get("/api/user/1")
            .accept(MediaType.APPLICATION_JSON))           // 期望返回JSON
            .andDo(print())                                 // 打印请求和响应详情（调试用）
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.id").value(1))           // 断言JSON字段
            .andExpect(jsonPath("$.name").value("张三"))
            .andExpect(jsonPath("$.email").value("zhangsan@test.com"));
}

@Test
public void testSaveUserApi() throws Exception {
    // 构造JSON请求体
    String userJson = "{ \"name\": \"李四\", \"password\": \"123456\", \"email\": \"lisi@test.com\" }";

    mockMvc.perform(post("/api/user")
            .contentType(MediaType.APPLICATION_JSON)       // 请求体是JSON
            .content(userJson)                              // 设置请求体
            .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isCreated())                 // 断言201 Created
            .andExpect(jsonPath("$.id").exists())            // 断言id字段存在
            .andExpect(jsonPath("$.name").value("李四"));
}

常用jsonPath断言：
● jsonPath("$.name").value("张三")     — 断言JSON字段值
● jsonPath("$.name").exists()          — 断言字段存在
● jsonPath("$.list").isArray()         — 断言是数组
● jsonPath("$.list", hasSize(3))       — 断言数组长度
● jsonPath("$[0].name").value("张三")  — 断言数组第一个元素的name

使用andDo(print())可以打印完整的请求和响应信息，方便调试。
```

# 6. 测试文件上传
```
Controller：
@PostMapping("/file/up")
public String fileUp(@RequestParam("fileName") MultipartFile file) throws IOException {
    if (!file.isEmpty()) {
        String filename = UUID.randomUUID().toString() + 
            file.getOriginalFilename().substring(file.getOriginalFilename().lastIndexOf("."));
        file.transferTo(new File("D:/upload/" + filename));
    }
    return "ok";
}

测试类：
@Test
public void testFileUpload() throws Exception {
    // 创建模拟的文件
    MockMultipartFile file = new MockMultipartFile(
        "fileName",                              // 表单字段名（对应@RequestParam）
        "test.jpg",                              // 原始文件名
        "image/jpeg",                            // MIME类型
        "mock file content".getBytes()           // 文件内容字节
    );

    mockMvc.perform(multipart("/file/up")        // multipart请求
            .file(file))                         // 添加文件
            .andExpect(status().isOk())
            .andExpect(view().name("ok"));
}

测试多文件上传：
@Test
public void testMultiFileUpload() throws Exception {
    MockMultipartFile file1 = new MockMultipartFile("fileNames", "1.jpg", "image/jpeg", "content1".getBytes());
    MockMultipartFile file2 = new MockMultipartFile("fileNames", "2.png", "image/png", "content2".getBytes());
    MockMultipartFile file3 = new MockMultipartFile("fileNames", "3.pdf", "application/pdf", "content3".getBytes());

    mockMvc.perform(multipart("/file/multiUp")
            .file(file1).file(file2).file(file3))   // 添加多个文件
            .andExpect(status().isOk())
            .andExpect(view().name("ok"));
}
```

# 7. 测试异常处理
```
测试@ControllerAdvice异常处理器：

异常处理器：
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public String handleUserNotFound(UserNotFoundException e, Model model) {
        model.addAttribute("msg", e.getMessage());
        return "error_404";
    }
}

测试类：
@Test
public void testUserNotFoundException() throws Exception {
    // 假设 /user/999 会抛出 UserNotFoundException
    mockMvc.perform(get("/user/999"))
            .andExpect(status().isNotFound())                        // 断言404
            .andExpect(view().name("error_404"))                     // 断言错误视图
            .andExpect(model().attributeExists("msg"));
}

测试RESTful API的异常返回JSON：
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<Map<String, Object>> handleBusiness(BusinessException e) {
        Map<String, Object> body = new HashMap<>();
        body.put("code", e.getCode());
        body.put("message", e.getMessage());
        return ResponseEntity.badRequest().body(body);
    }
}

测试：
@Test
public void testBusinessExceptionReturnsJson() throws Exception {
    mockMvc.perform(get("/api/user/-1"))   // 触发业务异常
            .andExpect(status().isBadRequest())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.code").value(400))
            .andExpect(jsonPath("$.message").exists());
}
```

# 8. 测试拦截器
```
测试拦截器是否对指定路径生效：

拦截器配置：
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        if (request.getSession().getAttribute("user") == null) {
            response.sendRedirect("/login");
            return false;
        }
        return true;
    }
}

测试类：

@Test
public void testAuthInterceptorRedirectWhenNotLoggedIn() throws Exception {
    // 未登录时访问受保护页面，应重定向到登录页
    mockMvc.perform(get("/admin/dashboard"))
            .andExpect(status().is3xxRedirection())        // 断言3xx重定向
            .andExpect(redirectedUrl("/login"));            // 断言重定向URL
}

@Test
public void testAuthInterceptorPassWhenLoggedIn() throws Exception {
    // 模拟已登录用户（设置Session）
    mockMvc.perform(get("/admin/dashboard")
            .sessionAttr("user", new User(1L, "张三", "", "")))  // 设置Session属性
            .andExpect(status().isOk());
}

测试拦截器的排除路径：
@Test
public void testInterceptorExcludedPath() throws Exception {
    // 排除了 /login 路径，应该能正常访问
    mockMvc.perform(get("/login"))
            .andExpect(status().isOk());
}
```

# 9. 设置请求头和Session
```
5.1 设置请求头：
mockMvc.perform(get("/api/user/1")
        .header("Authorization", "Bearer xxx-token")   // 设置Authorization头
        .header("User-Agent", "Mozilla/5.0")           // 设置User-Agent
        .header("Referer", "http://example.com"))       // 设置Referer
        .andExpect(status().isOk());

5.2 设置Session属性：
mockMvc.perform(get("/user/profile")
        .sessionAttr("user", userObject)                // 设置Session中已有属性
        .sessionAttr("loginTime", System.currentTimeMillis()))
        .andExpect(status().isOk())
        .andExpect(request().sessionAttribute("user", notNullValue())); // 断言Session属性

5.3 设置Cookie：
mockMvc.perform(get("/user/info")
        .cookie(new Cookie("token", "abc123"))          // 设置Cookie
        .cookie(new Cookie("lang", "zh_CN")))
        .andExpect(status().isOk())
        .andExpect(cookie().value("token", "abc123"));  // 断言Cookie值

5.4 设置PathVariable（通过URL路径）：
mockMvc.perform(get("/user/123"))                       // URL中的123就是PathVariable
        .andExpect(status().isOk());
```

# 10. 测试速查表
```
| 测试目标 | 示例代码 |
|----------|----------|
| GET请求 | mockMvc.perform(get("/path")) |
| POST请求 | mockMvc.perform(post("/path").param("key","val")) |
| PUT请求 | mockMvc.perform(put("/path")) |
| DELETE请求 | mockMvc.perform(delete("/path")) |
| JSON请求体 | .contentType(APPLICATION_JSON).content(jsonStr) |
| 表单参数 | .param("key", "value") |
| 路径变量 | 直接写在URL中：get("/user/123") |
| 请求头 | .header("Key", "Value") |
| Session | .sessionAttr("key", obj) |
| Cookie | .cookie(new Cookie("k", "v")) |
| 断言200 | .andExpect(status().isOk()) |
| 断言404 | .andExpect(status().isNotFound()) |
| 断言JSON | .andExpect(jsonPath("$.key").value("val")) |
| 断言视图名 | .andExpect(view().name("xxx")) |
| 断言Model | .andExpect(model().attribute("key","val")) |
| 断言重定向 | .andExpect(redirectedUrl("/path")) |
| 断言转发 | .andExpect(forwardedUrl("/path")) |
| 打印请求响应 | .andDo(print()) |
| 独立模式MockMvc | MockMvcBuilders.standaloneSetup(controller).build() |
| Web上下文MockMvc | MockMvcBuilders.webAppContextSetup(wac).build() |
```
