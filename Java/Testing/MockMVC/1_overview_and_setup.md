# 1. MockMvc 的定位

在 Spring MVC 应用里，Controller 是最容易被改坏、也最值得被测试的一层：URL 映射、参数绑定、JSON 序列化、状态码、异常处理全部集中在这里。**MockMvc** 是 Spring Test 提供的 Web 层测试工具，它在**不启动真实 HTTP 服务器**的前提下，模拟一次完整的 `DispatcherServlet` 调用，让开发者用接近 HTTP 的语法构造请求、断言响应。理解它的边界，比记住几个断言方法更重要。

## 1.1 什么是 MockMvc

MockMvc 的核心是一套 `perform(...)` 加 `andExpect(...)` 的流式 API。测试代码发出一个"请求"，MockMvc 把它包装成 `MockHttpServletRequest`，交给模拟的 `DispatcherServlet`，走完 `HandlerMapping → HandlerAdapter → Controller → HttpMessageConverter` 的完整 MVC 处理链，最后把结果放进 `MockHttpServletResponse` 供断言。

```java
mockMvc.perform(get("/api/users/1"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.name").value("Ada"));
```

这段代码读起来几乎就是一句 HTTP 描述，这正是 MockMvc 的设计目标：**让 Web 层测试保持声明式**，而不是手工 new 一个 Controller 再检查返回值。手工调用 Controller 方法只能验证 Java 方法语义，验证不了注解映射、类型转换、消息转换和异常处理；而这些恰恰是 Web 层最容易出错的部分。

**设计考量**：MockMvc 不是 Mock 框架。它不生成替身，也不替代真实对象，而是**替代 Servlet 容器**。被测试的 Controller、过滤器、消息转换器通常都是真实实例，只有外部依赖（Service、Repository、远程客户端）才需要用 Mockito 替换。把 MockMvc 和"Mock 掉一切"混为一谈，是初学者最常见的误解。

## 1.2 在测试分层中的位置

把测试按范围从窄到宽排列，MockMvc 位于"Web 层切片"这一档：

| 层次 | 典型工具 | 启动内容 | 速度 | 覆盖重点 |
|------|----------|----------|------|----------|
| 纯单元测试 | JUnit 5 + Mockito | 无容器 | 毫秒级 | 业务规则 |
| Web 层切片 | `@WebMvcTest` + MockMvc | MVC 基础设施 | 百毫秒级 | 映射、参数、JSON、异常 |
| 完整上下文 | `@SpringBootTest` + MockMvc | 整个应用 | 秒级 | 跨层协作 |
| 真实端口 | `@SpringBootTest(RANDOM_PORT)` | 内嵌容器 + 网络 | 秒级 | 网络、容器、端到端 |

MockMvc 在第二、第三档都能用。区别只在于容器启动范围：`@WebMvcTest` 只加载 Web 相关 Bean，`@SpringBootTest` 加载全部 Bean。前者快但需要手动补齐依赖，后者真实但更慢。

> **行业实践**：多数团队把 Controller 契约测试放在 `@WebMvcTest` 层，把少量关键链路放在 `@SpringBootTest` 层。不要为了"更像生产"而让每个 Controller 测试都启动完整上下文，测试套件的执行时间会迅速失控。

## 1.3 MockMvc 与真实端口测试的差异

真实端口测试会启动内嵌 Tomcat，监听随机端口，通过 HTTP 客户端发真实网络请求。MockMvc 则完全在进程内完成调用，没有 socket、没有线程池、没有连接超时。两者覆盖的范围不同：

| 维度 | MockMvc | 真实端口测试 |
|------|---------|--------------|
| 是否监听端口 | 否 | 是 |
| 是否经过网络栈 | 否 | 是 |
| 是否经过 Servlet 容器 | 模拟 | 真实 |
| 过滤器与拦截器 | 可配置加载 | 完整加载 |
| HTTP/2、TLS | 不覆盖 | 可覆盖 |
| 启动速度 | 快 | 慢 |
| 适合场景 | Web 层契约 | 端到端冒烟 |

**结论**：MockMvc 验证的是"Spring MVC 如何处理这个请求"，不是"网络和容器是否正常"。如果问题出在序列化配置、路由、参数绑定、异常映射，MockMvc 能快速定位；如果问题出在端口、连接池、TLS、容器行为，就必须上真实端口测试。

# 2. 环境准备

## 2.1 依赖

Spring Boot 项目只需要一个测试 starter，它已经聚合了 JUnit 5、Mockito、AssertJ、Spring Test、JsonPath：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

如果使用 Gradle：

```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

需要特别注意版本基线：**Java 17、Spring Boot 3.x、JUnit 5**。Spring Boot 3 把 `javax.servlet` 全部迁移到了 `jakarta.servlet`，复制旧代码时如果看到 `javax.servlet.http.Cookie`，编译会直接失败。

## 2.2 项目结构

MockMvc 测试通常放在 `src/test/java` 下，包名与被测 Controller 保持一致：

```text
src/
├── main/java/com/example/demo/
│   ├── DemoApplication.java
│   └── web/GreetingController.java
└── test/java/com/example/demo/
    └── web/GreetingControllerTest.java
```

包结构不是形式主义。`@WebMvcTest` 默认从测试类所在包向上寻找 `@SpringBootApplication`，包放错位置会出现"找不到配置类"或"扫描不到 Controller"的错误。

# 3. 第一个 MockMvc 测试

## 3.1 生产代码

先准备一个最小可用的 Controller：

```java
package com.example.demo.web;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/greetings")
public class GreetingController {

    @GetMapping("/{name}")
    public Greeting greet(@PathVariable String name) {
        return new Greeting("Hello, " + name);
    }

    public record Greeting(String message) {
    }
}
```

这个 Controller 同时用到了路径映射、路径变量绑定和 JSON 序列化，足够验证 MockMvc 的基本流程。

## 3.2 测试代码

```java
package com.example.demo.web;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(GreetingController.class)
class GreetingControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void should_return_greeting_json() throws Exception {
        mockMvc.perform(get("/api/greetings/{name}", "Ada"))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.message").value("Hello, Ada"));
    }
}
```

三个断言分别验证了状态码、响应类型和 JSON 内容。请求路径中的 `{name}` 是 URI 模板，后面的 `"Ada"` 会按顺序填充，避免手工拼接字符串。

## 3.3 运行与解读

用 Maven 执行单个测试类：

```bash
mvn -q -Dtest=GreetingControllerTest test
```

用 Gradle：

```bash
./gradlew test --tests "com.example.demo.web.GreetingControllerTest"
```

测试通过说明三件事同时成立：`GET /api/greetings/{name}` 被正确映射、路径变量被成功绑定、返回值被 Jackson 序列化成 JSON。任何一环出错都会给出不同的失败信号：404 指向映射或包扫描，400 指向参数绑定，406 指向内容协商，500 往往指向序列化或异常处理。

# 4. MockMvc 的工作机制

## 4.1 调用链

下面这张图展示了一次 `perform` 的完整路径：

```text
测试代码
  │  mockMvc.perform(get("/api/greetings/Ada"))
  ▼
MockMvc
  │  构造 MockHttpServletRequest
  ▼
DispatcherServlet（模拟实例）
  │  HandlerMapping 找到 GreetingController
  │  HandlerAdapter 调用 greet("Ada")
  ▼
HttpMessageConverter
  │  Greeting → application/json
  ▼
MockHttpServletResponse
  │  andExpect(...) 逐项断言
  ▼
测试结果
```

注意图中的 `DispatcherServlet` 是 Spring Test 构造的模拟对象，不是容器里的那个 Servlet。它复用同一套 MVC 组件，所以映射、转换、异常解析的行为与生产一致，但没有真实的 Servlet 容器生命周期。

## 4.2 MockMvc 不覆盖什么

MockMvc 的"轻"来自取舍，它明确不覆盖以下内容：

- **真实网络行为**：连接建立、超时、重试、TLS 握手、HTTP/2。
- **Servlet 容器特性**：线程池、会话持久化、容器级错误页、部署描述符。
- **完整过滤器链的自动装配**：`@WebMvcTest` 只加载部分 Bean，自定义 Filter 需要显式引入。
- **真实外部依赖**：数据库、缓存、消息队列都必须替换或使用 Testcontainers。

**设计考量**：这些"不覆盖"不是缺陷，而是测试金字塔的分工。用 MockMvc 做 Controller 契约测试，用真实端口测试做冒烟，用 Testcontainers 验证基础设施集成，三层各司其职，才能兼顾速度与信心。

# 5. 与真实端口测试的对比

## 5.1 真实端口测试示例

同一段业务逻辑，用真实端口测试写出来是这样的：

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GreetingE2ETest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void should_return_greeting_over_http() {
        ResponseEntity<String> response =
                restTemplate.getForEntity("/api/greetings/Ada", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).contains("Hello, Ada");
    }
}
```

这段测试会真正启动 Tomcat、分配随机端口、走一次完整的 HTTP 往返。它能发现 MockMvc 发现不了的问题，比如端口配置、容器初始化、真实的字符集处理；代价是启动慢、调试成本高。

## 5.2 选型建议

| 你想验证 | 推荐方式 |
|----------|----------|
| URL 是否正确映射 | MockMvc |
| 参数绑定与校验 | MockMvc |
| JSON 字段与状态码 | MockMvc |
| 统一异常处理 | MockMvc |
| 安全过滤器链 | MockMvc + Spring Security Test |
| 真实 HTTP 往返 | 真实端口测试 |
| TLS、连接超时 | 真实端口测试 |
| 数据库与消息队列集成 | Testcontainers |

**建议顺序**：先用 MockMvc 把 Web 层契约锁住，再补少量真实端口冒烟测试。反过来做，测试会又慢又难定位。

# 6. 常见错误

| 现象 | 可能原因 | 修正 |
|------|----------|------|
| `mockMvc` 为 `null` | 缺少 `@WebMvcTest` 或 `@AutoConfigureMockMvc` | 补注解，并确认字段用 `@Autowired` |
| 所有请求返回 404 | Controller 不在扫描范围 | 调整包结构或显式指定 `@WebMvcTest(XxxController.class)` |
| 返回 415 | 请求体没有设置 `contentType` | 加 `.contentType(MediaType.APPLICATION_JSON)` |
| 返回 406 | `Accept` 与产出类型不匹配 | 检查 `produces` 与 `.accept(...)` |
| 中文乱码 | 读取响应时未指定字符集 | 用 `getContentAsString(StandardCharsets.UTF_8)` |
| 启动报缺少 Bean | `@WebMvcTest` 不加载 `@Service` | 用 `@MockitoBean` 替换依赖 |
| 所有请求 401/403 | 引入了 Spring Security | 用 `@WithMockUser` 或关闭过滤器 |

# 7. 练习

1. 写一个 `GET /api/books/{isbn}` 的 Controller，返回包含 `isbn` 和 `title` 的 JSON，用 `@WebMvcTest` 验证 200 与字段值。
2. 把同一个测试改成真实端口测试，对比两者的启动时间与代码复杂度。
3. 故意把 `@WebMvcTest` 的 Controller 参数写错，观察返回的 404 或 400，并解释失败原因。

# 8. 小结

MockMvc 是 Spring MVC 的进程内测试工具，用模拟的 `DispatcherServlet` 验证 Web 层处理链，不启动真实容器。它适合覆盖映射、参数绑定、JSON 序列化、状态码和异常处理，不适合验证网络与容器行为。

| 关键点 | 结论 |
|--------|------|
| 本质 | 替代 Servlet 容器，不替代真实业务对象 |
| 常用入口 | `@WebMvcTest` + `@Autowired MockMvc` |
| 核心 API | `perform` + `andExpect` |
| 测试范围 | Web 层契约 |
| 不覆盖 | 网络、TLS、容器生命周期 |
| 对比真实端口 | 快、隔离好，但不够端到端 |

下一篇 `2_standalone_setup.md` 会把容器也去掉，讨论如何用 `standaloneSetup` 在纯单元测试里驱动 MockMvc，以及它带来的自由与限制。
