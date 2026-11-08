# 1. MockMvc 的定位

**MockMvc 用于在"不启动真实 HTTP 服务"的前提下测试 Spring MVC 的请求处理管线：它构造 `MockHttpServletRequest` 直接调用 DispatcherServlet，走完参数解析、处理器调用、消息转换、异常处理的全流程，唯独不经过网络传输层。** 它比直接调用 Controller 方法真实（覆盖了 Spring MVC 的全部机制），又比 `RANDOM_PORT` 端到端轻。本篇讲装配方式、请求构建与断言、安全集成、异步测试与高频错误。

## 1.1 工作原理

**MockMvc 的本质是"对 DispatcherServlet 的直接方法调用 + Mock 的 Servlet API 对象"：**

```text
测试代码
   │  MockMvcRequestBuilders.get("/users/1")
   ▼
MockHttpServletRequest / MockHttpServletResponse（spring-test 的 Mock 实现）
   │  mockMvc.perform(...)
   ▼
DispatcherServlet.service(request, response)   ← 真实执行，无任何 Mock
   ├─ HandlerMapping / HandlerAdapter   路由匹配、参数绑定、@Valid 校验
   ├─ HandlerInterceptor                拦截器 preHandle/postHandle
   ├─ Controller 方法                    被测代码
   ├─ 异常处理                           @ExceptionHandler / ControllerAdvice
   └─ HttpMessageConverter              响应序列化（真实 Jackson）
   │
   ▼
ResultActions → andExpect(...) 对 response 断言
```

**被覆盖的**：URL 映射、路径变量与查询参数绑定、请求体反序列化、校验、内容协商、响应序列化、异常到状态码/错误体的映射、拦截器。**不被覆盖的**（盲区，需 `RANDOM_PORT` + 真客户端，见第 9 篇）：Servlet 容器层面行为（URL 解码、`server.*` 连接配置、gzip 压缩）；真实的过滤器注册顺序；网络层语义（超时、chunked 传输、重定向跟随）；监听真实端口的组件（actuator 独立端口）。

---

# 2. 配置方式

**三种装配方式按"真实度递增、速度递减"排列：**

| 方式 | 容器 | 过滤器/安全 | 适用 |
|------|------|-----------|------|
| `standaloneSetup` | 无 | 手动注册 | 纯 Controller 单元测试 |
| `@WebMvcTest` | Web 切片 | 自动（含 Security） | 控制器测试主力 |
| `@SpringBootTest` + `@AutoConfigureMockMvc` | 全量 | 自动（含全部 Filter Bean） | 跨层集成验证 |

## 2.1 `@WebMvcTest`

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(UserController.class) // 只装配 Web 层；已内含 @AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;      // 由切片自动配置提供

    @MockitoBean                  // Boot 3.4+；旧版本用 @MockBean
    private UserService userService;

    @Test
    void should_return_ok() throws Exception {
        mockMvc.perform(get("/users/1"))
                .andExpect(status().isOk());
    }
}
```

优点：启动快（百毫秒级），自动加载 MVC 相关 Bean、`@ControllerAdvice`、Jackson 定制与 Spring Security 集成；失败定位锁定在 Web 层。这是控制器测试的默认选择（切片机制见第 7 篇）。

## 2.2 `@SpringBootTest` + `@AutoConfigureMockMvc`

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest                    // 完整容器：真实 Service、真实数据库（或 Testcontainers）
@AutoConfigureMockMvc              // 在全量容器上追加 MockMvc 自动配置
class WebIntegrationTest {

    @Autowired
    private MockMvc mockMvc;       // 管线后面接的是真实业务栈
}
```

优点：上下文完整，请求穿过真实的 Service/Repository，适合多组件协作验证与"配置是否生效"类断言。代价：全量容器启动成本；且 **`webEnvironment` 必须是默认的 MOCK**——`RANDOM_PORT` 下没有 Mock Servlet 环境，`@AutoConfigureMockMvc` 会失败（真 HTTP 场景用 `WebTestClient`/`TestRestTemplate`）。常用属性：`addFilters = false` 跳过 Filter 注册；`print` 控制打印策略。

## 2.3 `standaloneSetup`

**完全不依赖 Spring 容器的构建方式：手工把 Controller 塞进一个最小 MVC 管线**，适合纯 Controller 单元测试（依赖用 Mockito 手工构造）：

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;

import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@ExtendWith(MockitoExtension.class) // 纯 Jupiter + Mockito，无 Spring 容器
class UserControllerStandaloneTest {

    @Mock
    private UserService userService;  // 手工 Mock，不进任何容器

    private MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders
                .standaloneSetup(new UserController(userService)) // 直接 new 被测对象
                .setControllerAdvice(new GlobalExceptionHandler()) // 手动挂异常处理器
                .build();
    }

    @Test
    void should_return_user() throws Exception {
        given(userService.findById(1L)).willReturn(new User(1L, "tom"));

        mockMvc.perform(get("/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("tom"));
    }
}
```
**与容器方式的差异清单**（选 standalone 前必须知道放弃了什么）：不加载 `application.yml` 的任何配置（context-path、Jackson 定制全部失效）；`@ControllerAdvice`、拦截器、转换器、过滤器都要手动 `setXxx(...)` 注册，漏注册就静默不生效；`@Valid` 校验需手动配置 Validator。

> **设计争议**：standaloneSetup 评价两极。支持者认为它是"最快的 MVC 测试"；批评者指出它测的管线与生产管线是两个东西——手动注册的 Advice/拦截器清单一旦与生产脱节，测试绿灯毫无意义。行业折中：**新代码默认 `@WebMvcTest`（切片足够快），standalone 只留给对启动时间极度敏感的大型遗留套件**。

---

# 3. 请求构造与断言

## 3.1 构造请求与 JSON 请求体

**`MockMvcRequestBuilders` 为每种 HTTP 方法提供静态工厂（`get/post/put/patch/delete/multipart`），返回可链式配置的 `MockHttpServletRequestBuilder`；路径变量写在 URI 模板里（`get("/users/{id}", 1L)`）：**

```java
import org.springframework.http.MediaType;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;

// 查询参数、请求头、Cookie
mockMvc.perform(get("/users")
        .param("page", "1")                       // 查询参数（可多值）
        .header("X-Trace-Id", "t-1")              // 请求头
        .cookie(new Cookie("sid", "s1")))         // Cookie（jakarta.servlet.http.Cookie）

// JSON 请求体：ObjectMapper 应从容器注入，保证与生产序列化配置一致
mockMvc.perform(post("/users")
        .contentType(MediaType.APPLICATION_JSON)   // 缺这行会 415
        .content(objectMapper.writeValueAsString(cmd))) // 对象 → JSON 串
```

也可以内联 JSON 字符串（简单场景更直观）；表单用 `contentType(APPLICATION_FORM_URLENCODED)` + `param(...)`；文件上传用 `multipart(...)`（第 7 节）。

## 3.2 断言响应

**`andExpect` 接收 `ResultMatcher`，`MockMvcResultMatchers` 提供四大类匹配器：status / header / content / jsonPath（另有 cookie、redirect、model 等）**：

```java
import static org.hamcrest.Matchers.containsString;
import static org.hamcrest.Matchers.hasSize;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

mockMvc.perform(get("/users/1"))
        .andExpect(status().isOk())                                  // 状态码
        .andExpect(header().string("X-Trace-Id", "t-1"))             // 响应头精确匹配
        .andExpect(content().contentType(MediaType.APPLICATION_JSON)) // Content-Type
        .andExpect(content().string(containsString("tom")))           // 响应体子串（Hamcrest）
        .andExpect(jsonPath("$.id").value(1))                         // JsonPath 取值
        .andExpect(jsonPath("$.roles", hasSize(2)));                  // JsonPath + Hamcrest
```

复杂响应体的整体比对：`content().json("{\"id\": 1, \"name\": \"tom\"}")` 用 JSONassert 做结构化比对（默认宽松模式：多余字段不报错；传第二参 `true` 为严格模式）。或者 `andReturn().getResponse().getContentAsString()` 拿到响应体后反序列化成对象转 AssertJ 断言。

> **行业实践**：`jsonPath` 逐字段断言与 `content().json` 整体断言的取舍——前者失败信息精确到字段，后者写起来短但失败时只报"不匹配"。字段少用 jsonPath；响应体大且结构稳定时用 `content().json` + 期望文件快照，配合严格模式还能当"契约快照"用。

---

# 4. 异常处理与全局拦截

## 4.1 ControllerAdvice 验证

**全局异常处理器的行为是 MockMvc 测试的高价值断言点：异常 → 状态码 → 错误体格式的映射只有在管线测试里才能验证。** 设 `@RestControllerAdvice` 里有 `@ExceptionHandler(UserNotFoundException.class)` 返回 404 + `ErrorResponse("USER_404", msg)`，测试里 `given(userService.findById(99L)).willThrow(...)` 后断言 `status().isNotFound()` 与 `jsonPath("$.code").value("USER_404")`。

要点：**异常必须被 Advice 捕获才走这个断言路径**；若没人处理，MockMvc 会把异常包成 `ServletException` 直接抛给测试方法——测试报错而非断言失败，看到这种形态先检查 Advice 是否被加载（standalone 模式常因漏注册 `setControllerAdvice` 出这个问题）。

## 4.2 过滤器与拦截器

**"拦截器"和"过滤器"的加载行为不同，这是高频混淆点**：`HandlerInterceptor`（拦截器）属于 MVC 配置（`WebMvcConfigurer#addInterceptors`），容器方式随配置类自动生效，standalone 需 `addInterceptors(...)` 手动挂；Servlet `Filter`（过滤器）容器方式下注册为 Bean 的 Filter 自动加入管线（含 Spring Security 过滤器链），`addFilters = false` 可整体关闭，standalone 需 `addFilters(...)` 手动挂；`FilterRegistrationBean` 注册的过滤器 Mock 环境按 order 加载，但个别容器级特性与真实 Tomcat 可能有出入。

---

# 5. 异步请求测试

**Controller 返回 `Callable`/`DeferredResult`/`WebAsyncTask` 时，MockMvc 的第一次 `perform` 只启动异步处理，需要二次 dispatch 才能拿到最终结果：**

```java
import org.junit.jupiter.api.Test;
import org.springframework.test.web.servlet.MvcResult;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.asyncDispatch;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@Test
void should_complete_async_request() throws Exception {
    MvcResult result = mockMvc.perform(get("/async"))
            .andExpect(request().asyncStarted())          // 断言确实进入了异步
            .andReturn();

    mockMvc.perform(asyncDispatch(result))              // 触发异步完成后的 dispatch
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.done").value(true));
}
```

`asyncDispatch` 内部等待异步结果完成再重新走一遍管线。**`SseEmitter`/流式响应**用 MockMvc 能力有限，完整验证建议 `RANDOM_PORT` + `WebTestClient`。返回 `CompletableFuture`/`Mono` 的 MVC Controller 同样走异步管线，测法一致。

---

# 6. 安全集成

**classpath 上有 Spring Security 时，容器方式的 MockMvc 自动套用完整安全过滤器链；`spring-security-test`（单独加依赖，scope=test）提供 RequestPostProcessor 简化"带身份发请求"：**

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestBuilders.formLogin;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;

// with(user(...))：伪造已认证用户，绕过登录流程直接测授权
mockMvc.perform(get("/admin/users")
        .with(user("admin").roles("ADMIN")))           // 具备 ROLE_ADMIN
        .andExpect(status().isOk());

// with(csrf())：为 POST/PUT/DELETE 附加合法 CSRF token（缺它会被 403）
mockMvc.perform(post("/users").with(csrf())
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"name\": \"tom\"}"))
        .andExpect(status().isOk());

// formLogin()：模拟表单登录；Basic 认证用 with(httpBasic("u", "p"))
mockMvc.perform(formLogin("/login").user("u1").password("p1"))
        .andExpect(status().is3xxRedirection());        // 登录成功通常重定向
```

**方法级安全**（`@PreAuthorize`）在 `@WebMvcTest` 里默认不启用（切片不加载完整代理），需要 `@Import(MethodSecurityConfig.class)` 或改在全量容器里测。安全测试完整话题见第 14 篇。

---

# 7. 文件上传

**`multipart(...)` 构建 multipart 请求，`MockMultipartFile`（`org.springframework.mock.web` 包）表示上传的文件内容：**

```java
import org.springframework.mock.web.MockMultipartFile;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@Test
void should_upload_file() throws Exception {
    MockMultipartFile file = new MockMultipartFile(
            "file", "a.txt", "text/plain",            // 参数名、原始文件名、Content-Type
            "hello".getBytes(StandardCharsets.UTF_8)); // 内容

    mockMvc.perform(multipart("/upload")
            .file(file)
            .param("category", "doc"))                 // 混合普通表单字段
            .andExpect(status().isOk());
}
```

大文件与 `spring.servlet.multipart.max-file-size` 限制在 Mock 环境**不生效**（没有真实容器解析），要测上传大小限制得用 `RANDOM_PORT`。`@RequestPart` 接收 JSON 部分：`new MockMultipartFile("meta", null, MediaType.APPLICATION_JSON_VALUE, jsonBytes)`。

---

# 8. 打印调试与常见坑

## 8.1 打印请求响应

`mockMvc.perform(...).andDo(print())` 打印完整请求/响应/处理器/异常信息（静态导入 `MockMvcResultHandlers.print`）。全局打印：类级 `alwaysDo(print())`（standalone）或 `@AutoConfigureMockMvc(print = MockMvcPrint.ALWAYS)`（容器方式）；配置项 `spring.test.mockmvc.print=always` 作用于全部 MockMvc 测试。`print()` 输出包含请求/响应的全部字段、Handler 与异常栈，是排查 4xx/5xx 的第一工具。

## 8.2 常见错误对照

| 问题 | 原因 | 解决 |
|------|------|------|
| 返回 415 | 请求没设 Content-Type | `.contentType(MediaType.APPLICATION_JSON)` |
| 返回 406 | Accept 头与可生产类型不匹配 | `.accept(...)` 或检查转换器配置 |
| JSON 断言失败但响应"看起来对" | 命名策略、日期格式与预期不符 | 确认注入的是容器定制后的 ObjectMapper；先 `print()` 看实际响应 |
| 过滤器未生效 | standalone 没 `addFilters`；或 `addFilters = false` | 容器方式自动加载；standalone 手动注册 |
| 403 且没配安全 | CSRF 拦截了 POST | `.with(csrf())` 或测试配置关闭 CSRF |
| `ServletException` 直接抛出 | 异常没被任何 Advice 处理 | 检查 `@ControllerAdvice` 是否被加载 |
| 异步接口断言拿到空响应 | 没做二次 dispatch | `asyncDispatch(result)`（第 5 节） |
| `@Value`/配置在 standalone 不生效 | standalone 不读任何配置文件 | 换 `@WebMvcTest` + `properties` |
| 中文响应乱码 | Mock 响应默认字符集 | `getContentAsString(StandardCharsets.UTF_8)` |

> **注意**：`andReturn()` 拿到 `MvcResult` 后可以脱离匹配器体系自由断言（`getResponse()`、`getResolvedException()`、`getHandler()`）——排查疑难时 `getResolvedException()` 直接告诉你异常被谁处理成了什么，比从响应体反推快得多。

---

# 9. 小结

| 主题 | 关键结论 |
|------|---------|
| 原理 | Mock 的 Servlet API 对象 + 真实的 DispatcherServlet 管线；无网络传输层 |
| 覆盖范围 | 路由、绑定、校验、序列化、异常映射、拦截器 ✓；Tomcat 行为、真实过滤器顺序、网络语义 ✗ |
| 三种装配 | standaloneSetup（无容器，手动挂一切）→ `@WebMvcTest`（切片，默认选择）→ `@SpringBootTest + @AutoConfigureMockMvc`（全量，跨层验证） |
| 请求构建 | `MockMvcRequestBuilders` 静态工厂 + 链式 param/header/cookie/content |
| 断言 | status/header/content/jsonPath 四类匹配器；大响应体用 `content().json` 快照比对 |
| 异步 | 第一次 perform 只启动异步，`asyncDispatch(result)` 二次分发拿最终结果 |
| 安全 | `user()`/`csrf()`/`httpBasic()`/`formLogin()`；方法级安全在切片里默认不生效 |
| 调试 | `print()` / `spring.test.mockmvc.print=always`；`getResolvedException()` 直查异常处理结果 |

下一篇讲真 HTTP 调用侧的工具：`RANDOM_PORT` 模式下的 `TestRestTemplate` 与 `WebTestClient`，以及测试外部 API 的 WireMock。
