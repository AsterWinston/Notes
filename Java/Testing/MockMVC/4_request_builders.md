# 1. 请求构造器概览

MockMvc 的请求由 `MockMvcRequestBuilders` 创建，返回一个 `MockHttpServletRequestBuilder`。这个 builder 采用链式设计，把 HTTP 方法、路径、查询参数、请求头、Cookie、Session、请求体逐项拼装，最后交给 `perform` 执行。掌握它的 API，等于掌握了 MockMvc 的"输入侧"。

## 1.1 常用静态工厂

| 方法 | 对应 HTTP | 典型用途 |
|------|-----------|----------|
| `get(url, vars...)` | GET | 查询 |
| `post(url, vars...)` | POST | 创建 |
| `put(url, vars...)` | PUT | 全量更新 |
| `patch(url, vars...)` | PATCH | 部分更新 |
| `delete(url, vars...)` | DELETE | 删除 |
| `head(url, vars...)` | HEAD | 只取响应头 |
| `options(url, vars...)` | OPTIONS | 探测支持的方法 |
| `multipart(url, vars...)` | POST | 文件上传 |
| `request(method, url, vars...)` | 任意 | 自定义方法 |

所有工厂都支持 URI 模板，`{id}` 占位符会按参数顺序替换，并且会自动做 URL 编码：

```java
mockMvc.perform(get("/api/users/{id}/orders/{orderId}", 1L, 99L))
        .andExpect(status().isOk());
```

**设计考量**：使用模板而不是字符串拼接，可以避免路径参数里的空格、斜杠、中文导致的编码问题。测试里很容易出现"传了 `a b` 却匹配不到"的情况，模板加可变参数是最省心的写法。

## 1.2 请求构建的基本结构

一次完整的请求通常包含四部分：

```text
HTTP 方法 + URL
  ├── 查询参数 / 表单参数  param(...)
  ├── 请求头 / Cookie      header(...) / cookie(...)
  ├── 会话与属性           session(...) / requestAttr(...)
  └── 请求体               content(...) + contentType(...)
```

后面的小节逐项展开。

# 2. HTTP 方法与路径

## 2.1 基础方法

```java
mockMvc.perform(get("/api/users/1"));
mockMvc.perform(post("/api/users"));
mockMvc.perform(put("/api/users/1"));
mockMvc.perform(patch("/api/users/1"));
mockMvc.perform(delete("/api/users/1"));
```

如果后端使用了自定义 HTTP 方法，可以用通用入口：

```java
mockMvc.perform(request(HttpMethod.valueOf("LOCK"), "/api/docs/1"));
```

## 2.2 路径变量与查询参数

路径变量通过工厂方法传入，查询参数通过 `param` 追加：

```java
mockMvc.perform(get("/api/users/{id}", 42L)
                .param("fields", "id", "name")
                .param("verbose", "true"))
        .andExpect(status().isOk());
```

`param` 有两种重载：`param(String name, String... values)` 支持同名多值，适合复选框、多选标签这类场景。Controller 端用 `@RequestParam List<String> fields` 接收时，两个值都会出现。

**注意**：`param` 同时服务于查询参数和表单参数。当请求方法是 POST 且 `contentType` 为 `application/x-www-form-urlencoded` 时，`param` 设置的值会进入请求体；其余情况下进入查询字符串。这是 MockMvc 的便利设计，但也容易让人误以为表单提交必须手工拼字符串。

# 3. 请求头、Cookie 与 Session

## 3.1 请求头

```java
mockMvc.perform(get("/api/users/1")
                .header("Authorization", "Bearer token-123")
                .header("X-Trace-Id", "trace-001")
                .accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk());
```

`accept` 是 `header("Accept", ...)` 的快捷方式。内容协商相关的测试应优先用它，语义更清晰。

同名多值请求头用 `headers`：

```java
mockMvc.perform(get("/api/search")
                .headers(new HttpHeaders() {{
                    add("X-Tag", "a");
                    add("X-Tag", "b");
                }}));
```

这种双花括号初始化会创建匿名子类，生产代码里不推荐，但测试里偶尔用于快速构造。更稳妥的写法是显式创建 `HttpHeaders` 再调用 `add`。

## 3.2 Cookie

```java
import jakarta.servlet.http.Cookie;

mockMvc.perform(get("/api/profile")
                .cookie(new Cookie("SESSION", "abc123"))
                .cookie(new Cookie("locale", "zh-CN")))
        .andExpect(status().isOk());
```

注意包名是 `jakarta.servlet.http.Cookie`，不是旧的 `javax.servlet.http.Cookie`。Spring Boot 3 已经全面迁移到 Jakarta EE。

## 3.3 Session

MockMvc 提供 `MockHttpSession`，可以模拟已登录会话：

```java
import org.springframework.mock.web.MockHttpSession;

MockHttpSession session = new MockHttpSession();
session.setAttribute("userId", 7L);

mockMvc.perform(get("/api/cart").session(session))
        .andExpect(status().isOk());

mockMvc.perform(get("/api/cart").sessionAttr("userId", 7L))
        .andExpect(status().isOk());
```

两种写法等价：`session(session)` 复用同一个会话对象，`sessionAttr(name, value)` 在每次请求时新建会话并写入属性。需要验证"会话中的属性是否被修改"时，用前者并在请求后检查 `session.getAttribute(...)`。

## 3.4 请求属性与 Flash 属性

请求属性只存在于当前请求，常用于过滤器向 Controller 传值：

```java
mockMvc.perform(get("/api/orders")
                .requestAttr("currentUser", new User(1L, "Ada")))
        .andExpect(status().isOk());
```

Flash 属性用于重定向后的临时数据传递：

```java
mockMvc.perform(get("/api/orders/1")
                .flashAttr("message", "操作成功"))
        .andExpect(status().isOk());
```

**设计考量**：请求属性是 Filter 与 Controller 之间最轻量的契约。它不需要线程本地变量，也不会污染全局状态。测试里显式设置 `requestAttr`，比启动一个真实过滤器更聚焦。

# 4. 请求体

## 4.1 JSON 请求体

```java
ObjectMapper objectMapper = new ObjectMapper();
CreateUserCommand command = new CreateUserCommand("Ada", "ada@example.com");

mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(command)))
        .andExpect(status().isCreated());
```

三个要点：

- `contentType` 决定 Spring 选择哪个 `HttpMessageConverter`，漏掉会返回 415。
- `accept` 决定响应体的协商类型，漏掉通常不影响 JSON，但显式声明更贴近真实客户端。
- `content` 接收字符串，对象序列化交给 `ObjectMapper`，不要在测试里手写易错的 JSON 字符串。

如果希望测试代码更直观，可以用文本块：

```java
mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {
                          "name": "Ada",
                          "email": "ada@example.com"
                        }
                        """))
        .andExpect(status().isCreated());
```

文本块适合字段少、结构稳定的场景；字段一多，还是用 `ObjectMapper` 更安全，因为编译器不会帮你检查 JSON 拼写。

## 4.2 表单请求体

```java
mockMvc.perform(post("/api/login")
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .param("username", "ada")
                .param("password", "secret"))
        .andExpect(status().isOk());
```

表单提交不需要手工拼接 `username=ada&password=secret`，`param` 会自动编码。

XML 与纯文本请求同理，只要容器里存在对应的转换器，MockMvc 就会走与生产一致的解析流程。

# 5. 文件上传

## 5.1 单文件上传

```java
import org.springframework.mock.web.MockMultipartFile;

MockMultipartFile file = new MockMultipartFile(
        "file",                       // 表单字段名
        "avatar.png",                 // 原始文件名
        MediaType.IMAGE_PNG_VALUE,    // 内容类型
        "fake-image-bytes".getBytes(StandardCharsets.UTF_8)
);

mockMvc.perform(multipart("/api/files").file(file))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.filename").value("avatar.png"));
```

`multipart` 默认使用 POST。如果需要 PUT 上传，可以调用 `.with(request -> { request.setMethod("PUT"); return request; })`，或者直接使用 `multipart(HttpMethod.PUT, url)` 的重载。

## 5.2 多文件与附加参数

```java
MockMultipartFile front = new MockMultipartFile(
        "files", "front.jpg", MediaType.IMAGE_JPEG_VALUE, "front".getBytes());
MockMultipartFile back = new MockMultipartFile(
        "files", "back.jpg", MediaType.IMAGE_JPEG_VALUE, "back".getBytes());

mockMvc.perform(multipart("/api/files/batch")
                .file(front)
                .file(back)
                .param("bucket", "avatars"))
        .andExpect(status().isOk());
```

同名 `file` 字段对应 Controller 的 `@RequestParam MultipartFile[] files` 或 `List<MultipartFile>`。附加参数继续用 `param`，MockMvc 会把它们放进 multipart 表单。

**注意**：`MockMultipartFile` 只模拟文件元数据和字节内容，不做真实磁盘写入。它能验证 Controller 的接收逻辑、文件名校验、大小限制，但不能验证存储层是否真的写成功。涉及真实存储的测试应交给集成测试。

# 6. 请求后处理器

`with(...)` 接收一个 `RequestPostProcessor`，可以在请求执行前做任意修改。Spring Security Test 的 `user()`、`csrf()` 就是通过这个机制接入的：

```java
mockMvc.perform(get("/api/admin")
                .with(request -> {
                    request.addHeader("X-Internal-Call", "true");
                    return request;
                }))
        .andExpect(status().isOk());
```

如果某个请求头需要在多个测试里重复添加，可以封装一个 `RequestPostProcessor` 实现，再通过 `.with(new TraceIdPostProcessor())` 复用。

# 7. 常见错误

| 现象 | 原因 | 修正 |
|------|------|------|
| 415 Unsupported Media Type | 没有设置 `contentType` | 加 `.contentType(MediaType.APPLICATION_JSON)` |
| 400 参数缺失 | `param` 名称与 `@RequestParam` 不一致 | 对照 Controller 签名检查 |
| 路径变量未替换 | 直接拼了 `{id}` 却没传参数 | 用 `get("/users/{id}", 1L)` |
| 中文乱码 | 读取响应未指定 UTF-8 | `getContentAsString(StandardCharsets.UTF_8)` |
| Session 不生效 | 每次请求新建了不同会话 | 复用同一个 `MockHttpSession` |
| 文件上传 400 | 字段名与 `@RequestParam` 不匹配 | 检查 `MockMultipartFile` 的第一个参数 |
| Cookie 类型错误 | 用了 `javax.servlet` 包 | 改为 `jakarta.servlet.http.Cookie` |

# 8. 练习

1. 构造一个带路径变量、两个查询参数、一个自定义请求头的 GET 请求，并断言 200。
2. 用 `ObjectMapper` 构造 JSON 请求体，测试 `POST /api/users` 返回 201 和 `Location` 头。
3. 写一个多文件上传测试，同时提交两个文件和一个普通表单参数，验证 Controller 收到的文件数量。

# 9. 小结

请求构造器把 HTTP 请求拆成方法、路径、参数、头、Cookie、会话、属性和请求体几部分，每部分都有对应的链式方法。掌握这些 API 后，绝大多数 Web 层输入场景都能用几行代码表达。

| 需求 | API |
|------|-----|
| 选择方法 | `get` / `post` / `put` / `patch` / `delete` |
| 路径变量 | 工厂方法可变参数 |
| 查询或表单参数 | `param(name, values...)` |
| 请求头 | `header` / `headers` / `accept` |
| Cookie | `cookie(new Cookie(...))` |
| 会话 | `session(...)` / `sessionAttr(...)` |
| 请求属性 | `requestAttr(...)` / `flashAttr(...)` |
| 请求体 | `content(...)` + `contentType(...)` |
| 文件上传 | `multipart(...).file(...)` |
| 自定义处理 | `with(RequestPostProcessor)` |

下一篇 `5_assertions_and_jsonpath.md` 转向输出侧，系统讲解状态码、响应头、Content-Type、jsonPath、整体 JSON 比较，以及转发与重定向的断言方式。
