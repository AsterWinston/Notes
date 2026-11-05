# 1. 断言体系概览

请求发出去之后，`andExpect(...)` 负责检查响应。MockMvc 的断言分为几组：状态码用 `status()`，响应头用 `header()`，响应体用 `content()`，JSON 字段用 `jsonPath()`，Cookie 用 `cookie()`，转发与重定向有专门的匹配器。它们都来自 `MockMvcResultMatchers`，通常静态导入后使用。

## 1.1 一次请求可以挂多个断言

```java
mockMvc.perform(get("/api/users/1"))
        .andExpect(status().isOk())
        .andExpect(header().string("Content-Type", "application/json"))
        .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
        .andExpect(jsonPath("$.id").value(1))
        .andExpect(jsonPath("$.name").value("Ada"));
```

断言按书写顺序执行，第一个失败会立即抛出 `AssertionError`，后续断言不再执行。因此把最能定位问题的断言放在前面，比如状态码。

## 1.2 失败信息怎么读

MockMvc 的失败信息通常会给出实际值、期望值和响应体片段。例如：

```text
java.lang.AssertionError: Status expected:<200> but was:<404>
```

看到 404 先查映射和包扫描；看到 400 查参数绑定；看到 415 查 `contentType`；看到 406 查 `accept` 与 `produces`。状态码是最便宜的分诊信号。

# 2. 状态码断言

## 2.1 语义化方法

```java
.andExpect(status().isOk())                    // 200
.andExpect(status().isCreated())               // 201
.andExpect(status().isNoContent())             // 204
.andExpect(status().isBadRequest())            // 400
.andExpect(status().isUnauthorized())          // 401
.andExpect(status().isForbidden())             // 403
.andExpect(status().isNotFound())              // 404
.andExpect(status().isMethodNotAllowed())      // 405
.andExpect(status().isUnsupportedMediaType())  // 415
.andExpect(status().is4xxClientError())        // 4xx 任意
.andExpect(status().is5xxServerError())        // 5xx 任意
.andExpect(status().is3xxRedirection())        // 3xx 任意
```

语义化方法比 `status().value(200)` 可读性更好，失败时也能直接看出意图。

## 2.2 精确值断言

当接口返回非标准状态码，或者需要精确验证某个 4xx 时：

```java
.andExpect(status().value(422));
```

**设计考量**：能用语义化方法就不要写数字。数字断言在重构时不会提示语义变化，而 `isUnprocessableEntity()` 这类方法名本身就是文档。

# 3. 响应头断言

## 3.1 单个头

```java
.andExpect(header().string("Location", "/api/users/1"))
.andExpect(header().string("X-Trace-Id", "trace-001"))
.andExpect(header().exists("ETag"))
.andExpect(header().doesNotExist("X-Deprecated"));
```

`header().string` 是精确匹配。如果只想验证包含关系，可以传 Hamcrest 匹配器：

```java
.andExpect(header().string("Location", containsString("/api/users/")));
```

## 3.2 多值头

```java
.andExpect(header().stringValues("Set-Cookie",
        "SESSION=abc; Path=/; HttpOnly",
        "locale=zh-CN; Path=/"));
```

`stringValues` 会按顺序精确比较所有值，适合验证多个 `Set-Cookie` 的场景。

## 3.3 Cookie 断言

```java
.andExpect(cookie().value("SESSION", "abc123"))
.andExpect(cookie().httpOnly("SESSION", true))
.andExpect(cookie().secure("SESSION", true))
.andExpect(cookie().path("SESSION", "/"))
.andExpect(cookie().maxAge("SESSION", 3600));
```

Cookie 的 `HttpOnly`、`Secure`、`SameSite` 是安全测试的高频检查点，MockMvc 能直接验证响应中的设置，不需要浏览器。

# 4. Content-Type 与响应体

## 4.1 内容类型

```java
.andExpect(content().contentType(MediaType.APPLICATION_JSON))
.andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
```

两者的区别：`contentType` 要求完全相等，包括字符集参数；`contentTypeCompatibleWith` 只比较主类型和子类型，忽略 `charset` 等参数。多数测试应使用后者，避免因为 `application/json;charset=UTF-8` 而误报。

> **注意**：Spring Boot 3 默认不再强制给 JSON 响应附加 `charset=UTF-8`。如果旧测试写了精确的 `contentType(MediaType.APPLICATION_JSON_UTF8)`，升级后会失败，应改为 `contentTypeCompatibleWith`。

## 4.2 字符串与字节

```java
.andExpect(content().string("OK"))
.andExpect(content().string(containsString("Ada")))
.andExpect(content().string(startsWith("{")))
.andExpect(content().bytes("hello".getBytes(StandardCharsets.UTF_8)));
```

读取中文响应时，优先使用 `MvcResult` 并指定字符集：

```java
MvcResult result = mockMvc.perform(get("/api/greetings/张三"))
        .andExpect(status().isOk())
        .andReturn();

String body = result.getResponse().getContentAsString(StandardCharsets.UTF_8);
assertThat(body).contains("张三");
```

`getContentAsString()` 无参版本依赖响应的字符集，有时会得到乱码。显式传 `UTF-8` 更稳妥。

# 5. jsonPath 断言

## 5.1 基本语法

`jsonPath` 使用 Jayway JsonPath 语法，常用的几种表达式：

| 表达式 | 含义 |
|--------|------|
| `$.id` | 根对象的 `id` 字段 |
| `$.user.name` | 嵌套字段 |
| `$.items[0]` | 数组第一个元素 |
| `$.items[*].name` | 数组所有元素的 `name` |
| `$..name` | 任意层级的 `name` |
| `$.tags` | 整个数组 |

## 5.2 值断言

```java
.andExpect(jsonPath("$.id").value(1))
.andExpect(jsonPath("$.name").value("Ada"))
.andExpect(jsonPath("$.active").value(true))
.andExpect(jsonPath("$.user.address.city").value("Shanghai"))
.andExpect(jsonPath("$.items[0].name").value("book"))
.andExpect(jsonPath("$.tags[1]").value("backend"));
```

数值类型有一个常见坑：JsonPath 会把小整数解析成 `Integer`，把小数解析成 `Double`。写 `.value(1L)` 可能匹配失败，写 `.value(1)` 才稳定。浮点数不要用精确相等，改用 `closeTo`：

```java
.andExpect(jsonPath("$.price").value(closeTo(19.99, 0.001)));
```

## 5.3 数组与集合断言

```java
.andExpect(jsonPath("$.tags", hasSize(2)))
.andExpect(jsonPath("$.tags", contains("java", "spring")))
.andExpect(jsonPath("$.tags", hasItem("java")))
.andExpect(jsonPath("$.items[*].name", contains("book", "pen")))
.andExpect(jsonPath("$.items").isArray())
.andExpect(jsonPath("$.items").isNotEmpty())
.andExpect(jsonPath("$.tags").isEmpty());
```

`contains` 要求顺序一致，`containsInAnyOrder` 忽略顺序。接口契约如果没承诺顺序，应使用后者，避免测试过于脆弱。

## 5.4 存在性断言

```java
.andExpect(jsonPath("$.email").exists())
.andExpect(jsonPath("$.deletedAt").doesNotExist())
.andExpect(jsonPath("$.password").doesNotExist())
```

`doesNotExist` 是安全测试的常用断言，用来确保敏感字段没有泄漏到响应里。`jsonPath("$.password").isEmpty()` 与 `doesNotExist` 语义不同：前者是字段存在但值为空，后者是字段完全不存在。

`value` 也可以接收 `containsString`、`startsWith`、`greaterThan` 等 Hamcrest 匹配器。**设计考量**：不要把所有字段都写成精确值，时间戳、ID、随机 token 适合用 `exists` 或匹配器，过度精确的断言会在无关变更时频繁失败。

# 6. 整体 JSON 比较

## 6.1 宽松比较

`content().json(expected)` 默认是宽松模式：期望 JSON 里的字段必须存在且相等，但实际响应可以多出字段。这适合只关心核心字段的契约测试。

## 6.2 严格比较

```java
.andExpect(content().json("""
        {
          "id": 1,
          "name": "Ada"
        }
        """, true));
```

第二个参数为 `true` 时，字段集合必须完全一致，多一个少一个都会失败。严格模式适合验证 DTO 的完整输出，但也最容易因新增字段而失败。

**建议**：大部分测试用宽松模式加 `jsonPath` 精确断言关键字段；只有在明确要锁死 DTO 结构时，才使用严格比较。

# 7. 转发与重定向

## 7.1 重定向

```java
mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"Ada\"}"))
        .andExpect(status().is3xxRedirection())
        .andExpect(redirectedUrl("/api/users/1"))
        .andExpect(header().string("Location", "/api/users/1"));
```

`redirectedUrl` 精确匹配 `Location`，`redirectedUrlPattern` 支持 Ant 风格通配：

```java
.andExpect(redirectedUrlPattern("**/login"))
.andExpect(redirectedUrlPattern("https://*/callback*"));
```

## 7.2 转发

服务端转发不会改变浏览器 URL，响应状态仍是 200：

```java
mockMvc.perform(get("/old-path"))
        .andExpect(status().isOk())
        .andExpect(forwardedUrl("/WEB-INF/views/home.jsp"))
        .andExpect(forwardedUrlPattern("/WEB-INF/views/*.jsp"));
```

**注意**：在 `@RestController` 项目中，转发和重定向都比较少见；如果测试一直拿不到 `forwardedUrl`，先确认 Controller 返回的是 `"forward:/..."` 或 `ModelAndView`，而不是直接返回 JSON。

# 8. 自定义断言与调试输出

## 8.1 用 MvcResult 做后置断言

内置匹配器不够用时，可以调用 `andReturn()` 取回 `MvcResult`，再用 `getResponse().getContentAsString(StandardCharsets.UTF_8)` 拿到响应体，交给 AssertJ 或 Jackson 做跨字段校验。这种写法适合复杂结构和需要复用生产端 `ObjectMapper` 的场景。

## 8.2 打印请求与响应

调试阶段可以在链末加 `andDo(print())`：

```java
mockMvc.perform(get("/api/users/1"))
        .andDo(print())
        .andExpect(status().isOk());
```

它会输出请求方法、URL、头、请求体、状态码、响应头和响应体，是排查"为什么断言失败"最快的手段。

# 9. 常见错误

| 现象 | 原因 | 修正 |
|------|------|------|
| `contentType` 断言失败 | 响应带 `charset` 参数 | 改用 `contentTypeCompatibleWith` |
| `.value(1L)` 失败 | JsonPath 把小整数解析为 `Integer` | 改成 `.value(1)` |
| 数组顺序断言不稳定 | 用了 `contains` | 改用 `containsInAnyOrder` |
| 严格 JSON 比较频繁失败 | 接口新增字段 | 改用宽松模式或 `jsonPath` |
| 重定向断言取不到 URL | 响应不是 3xx | 先断言 `is3xxRedirection()` |
| 中文断言乱码 | 未指定 UTF-8 | `getContentAsString(StandardCharsets.UTF_8)` |
| `doesNotExist` 与预期不符 | 字段存在但值为 `null` | 用 `value(nullValue())` 或检查序列化配置 |

# 10. 练习

1. 为一个返回用户列表的接口写断言：状态 200、`$.items` 长度为 2、第一个元素的 `name` 为 `"Ada"`。
2. 用 `content().json(...)` 的宽松与严格两种模式分别断言同一个响应，观察新增字段时的差异。
3. 写一个登录接口测试，验证成功时返回 302、`Location` 为 `/dashboard`，失败时返回 401 且响应体包含错误码。

# 11. 小结

MockMvc 的断言覆盖状态码、响应头、Cookie、内容类型、响应体和 JSON 结构。`jsonPath` 是最常用的工具，配合 Hamcrest 匹配器可以表达复杂的结构断言；`content().json` 适合整体比较，但要注意宽松与严格模式的区别。

| 断言目标 | 常用 API |
|----------|----------|
| 状态码 | `status().isOk()` 等 |
| 响应头 | `header().string` / `exists` / `stringValues` |
| Cookie | `cookie().value` / `httpOnly` / `secure` |
| 内容类型 | `content().contentTypeCompatibleWith` |
| JSON 字段 | `jsonPath("$.x").value(...)` |
| 数组 | `hasSize` / `contains` / `containsInAnyOrder` |
| 整体 JSON | `content().json(...)` |
| 重定向 | `redirectedUrl` / `redirectedUrlPattern` |
| 转发 | `forwardedUrl` / `forwardedUrlPattern` |
| 调试 | `andDo(print())` |

下一篇 `6_filters_and_interceptors.md` 进入请求处理链，讨论 Filter 与 HandlerInterceptor 的区别、注册方式、执行顺序，以及如何验证鉴权上下文和请求属性。
