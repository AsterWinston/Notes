# 1. MockMvc 的定位

MockMvc 用于在**不启动真实 HTTP 服务**的情况下测试 Spring MVC。它模拟 DispatcherServlet 调用流程，适用于控制器层测试与异常处理验证。

---

# 2. 配置方式

## 2.1 `@WebMvcTest`

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
}
```

优点：启动快，加载 MVC 相关 Bean。

## 2.2 `@SpringBootTest` + `@AutoConfigureMockMvc`

```java
@SpringBootTest
@AutoConfigureMockMvc
class WebIntegrationTest {
    @Autowired
    private MockMvc mockMvc;
}
```

优点：上下文完整，适合多组件协作验证。

## 2.3 `standaloneSetup`

适用于纯 Controller 单元测试，不依赖 Spring 容器。

```java
MockMvc mockMvc = MockMvcBuilders.standaloneSetup(new UserController()).build();
```

---

# 3. 请求构造与断言

## 3.1 构造请求

```java
mockMvc.perform(get("/users")
        .param("page", "1")
        .header("X-Trace-Id", "t-1")
        .cookie(new Cookie("sid", "s1"))
)
```

## 3.2 JSON 请求体

```java
mockMvc.perform(post("/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(cmd))
)
```

## 3.3 断言响应

```java
mockMvc.perform(get("/users/1"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.id").value(1))
    .andExpect(content().contentType(MediaType.APPLICATION_JSON));
```

---

# 4. 异常处理与全局拦截

## 4.1 ControllerAdvice 验证

可通过 MockMvc 验证统一异常处理结果。

## 4.2 过滤器与拦截器

- `@WebMvcTest` 会加载 `@ControllerAdvice`
- 过滤器默认不加载，需显式配置

---

# 5. 异步请求测试

```java
MvcResult result = mockMvc.perform(get("/async"))
    .andExpect(request().asyncStarted())
    .andReturn();

mockMvc.perform(asyncDispatch(result))
    .andExpect(status().isOk());
```

---

# 6. 安全集成

```java
mockMvc.perform(post("/login")
        .with(csrf())
        .param("username", "u1")
        .param("password", "p1")
)
```

Spring Security Test 提供 `user()`、`csrf()` 等请求后处理器。

---

# 7. 文件上传

```java
mockMvc.perform(multipart("/upload")
        .file(new MockMultipartFile("file", "a.txt", "text/plain", "hello".getBytes()))
)
.andExpect(status().isOk());
```

---

# 8. 常见坑

| 问题 | 原因 | 解决 |
|------|------|------|
| 返回 415 | Content-Type 不匹配 | 设置 `contentType` |
| JSON 断言失败 | 字段命名策略不同 | 确认 Jackson 配置 |
| 过滤器未生效 | `@WebMvcTest` 不加载 | 使用 `@Import` 或 `@SpringBootTest` |
