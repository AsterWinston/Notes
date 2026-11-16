# 1. 真 HTTP 调用测试的定位

MockMvc 在进程内直接调用 DispatcherServlet（第 8 篇），验证的是"MVC 管线对不对"；本篇的工具验证的是另一件事——**请求经过真实的网络栈（socket、Tomcat 连接器、真实 HTTP 客户端）之后，系统还对不对**。这要求 `@SpringBootTest(webEnvironment = RANDOM_PORT)` 启动真实嵌入式容器，然后用 `TestRestTemplate` 或 `WebTestClient` 从客户端侧发起调用。这一层的独有价值：完整的序列化/反序列化（客户端与服务端两侧的 Jackson）、Content-Type 协商、过滤器与安全的真实注册顺序、重定向与 Cookie 的传输语义、HTTP 客户端自身的配置（超时、连接池）。

```text
MockMvc（MOCK 环境）                 TestRestTemplate / WebTestClient（RANDOM_PORT）
┌────────────┐                      ┌────────────┐        ┌──────────────────┐
│ 测试线程     │                      │ 测试线程     │  HTTP  │ 嵌入式 Tomcat     │
│    │ perform │                      │    │ 客户端   │───────▶│  真实端口          │
│    ▼         │                      │    ▼         │ socket │    ▼              │
│ DispatcherServlet                   │ RestTemplate │        │ DispatcherServlet │
│（进程内直调，无网络）                  │（真实网络栈） │◀───────│  （完整请求管线）   │
└────────────┘                      └────────────┘        └──────────────────┘
```

---

# 2. TestRestTemplate

## 2.1 定位与特点

**`TestRestTemplate` 是 Spring Boot 对 `RestTemplate` 的测试专用包装，在 `RANDOM_PORT`/`DEFINED_PORT` 环境下作为 Bean 自动提供，且已预置指向本地随机端口的 baseUrl。** 与裸 `RestTemplate` 的差异：

- **不抛异常于 4xx/5xx**：`RestTemplate` 默认对错误状态码抛 `HttpClientErrorException`，而测试经常需要断言错误响应——`TestRestTemplate` 关掉了这个行为，把错误响应原样返回供断言
- **自动配置 `HttpMessageConverter`**：与容器侧一致的 Jackson 定制
- **支持基础认证快捷方法**：`withBasicAuth(...)`
- **Cookie 处理宽松**：不会像浏览器客户端那样自动跟随重定向丢上下文，适合断言 3xx 行为

> **注意**：`TestRestTemplate` 只在 `webEnvironment = RANDOM_PORT/DEFINED_PORT` 下可用——MOCK 环境没有真实端口，注入它会启动失败。它内部包装的仍是 `RestTemplate`（同步、`java.net.http`/HttpComponents 底层），在 Boot 3.x 里 `RestTemplate` 虽被 `RestClient` 取代为推荐生产客户端，但 `TestRestTemplate` 依然是测试侧的正规工具；追求新 API 风格可改用 `WebTestClient`（见第 3 节，两者在 servlet 应用上都可用）。

## 2.2 典型用法

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort; // Boot 3.x 包位置
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpApiTest {

    @LocalServerPort
    private int port;                 // 实际绑定的随机端口

    @Autowired
    private TestRestTemplate rest;    // 已预置 http://localhost:{port} 前缀

    @Test
    void should_get_user() {
        // 相对路径即可，TestRestTemplate 自动拼接本地端口
        ResponseEntity<User> resp = rest.getForEntity("/users/1", User.class);

        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(resp.getBody()).isNotNull();
        assertThat(resp.getBody().getName()).isEqualTo("tom");
    }

    @Test
    void should_return_404_when_missing() {
        ResponseEntity<String> resp = rest.getForEntity("/users/999", String.class);

        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND); // 不抛异常
    }

    @Test
    void should_create_user() {
        ResponseEntity<User> resp = rest.postForEntity("/users",
                new CreateUserRequest("jerry"), User.class);

        assertThat(resp.getStatusCode()).is2xxSuccessful();
    }
}
```

需要完整控制（自定义请求头、PUT/DELETE 带 body）时用 `exchange`：

```java
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;

HttpHeaders headers = new HttpHeaders();
headers.set("X-Trace-Id", "t-1");
HttpEntity<Void> entity = new HttpEntity<>(headers);

ResponseEntity<String> resp = rest.exchange("/users/1", HttpMethod.GET, entity, String.class);
```

`@LocalServerPort` 是 `@Value("${local.server.port}")` 的类型安全等价物——`RANDOM_PORT` 启动后，实际端口以 `local.server.port` 属性暴露，注解从环境读取并注入。**只在 `RANDOM_PORT`/`DEFINED_PORT` 下有意义**。

## 2.3 认证方式

```java
// Basic 认证：返回带认证头的新实例（不修改原 Bean）
TestRestTemplate authRest = rest.withBasicAuth("user", "pwd");
ResponseEntity<String> resp = authRest.getForEntity("/api/me", String.class);
```

其他认证形态（Bearer token、OAuth2）通过 `exchange` + 手工设置 `Authorization` 头完成；完整的认证测试策略见第 14 篇。

---

# 3. WebTestClient

## 3.1 响应式测试的首选

**`WebTestClient` 是 Spring 5 引入的非阻塞 HTTP 测试客户端，最初为 WebFlux 设计，提供"请求 → exchange → 链式断言"的流式 API：**

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ReactiveApiTest {

    @Autowired
    private WebTestClient webTestClient; // RANDOM_PORT 下自动指向本地端口

    @Test
    void should_return_user() {
        webTestClient.get()
                .uri("/users/1")
                .exchange()                       // 发送请求，进入断言链
                .expectStatus().isOk()
                .expectHeader().contentTypeCompatibleWith("application/json")
                .expectBody()
                .jsonPath("$.id").isEqualTo(1)    // 内置 JsonPath 断言
                .jsonPath("$.name").isEqualTo("tom");
    }

    @Test
    void should_deserialize_body_to_object() {
        User user = webTestClient.get()
                .uri("/users/1")
                .exchange()
                .expectStatus().isOk()
                .expectBody(User.class)           // 反序列化为目标类型
                .returnResult()
                .getResponseBody();

        assertThat(user.getName()).isEqualTo("tom"); // 转 AssertJ 继续断言
    }
}
```

断言体系三段式：`expectStatus()`（状态码）、`expectHeader()`（响应头）、`expectBody()`（响应体——JsonPath、整体 JSON、反序列化对象、字节数组）。对流式端点（SSE、`Flux` 响应）用 `returnFlux(User.class)` + `StepVerifier`（reactor-test）验证元素序列——这是 `TestRestTemplate` 做不到的。

## 3.2 在 MVC（Servlet）应用中使用

**Boot 2.2 起 `WebTestClient` 也能测试 servlet 应用：`RANDOM_PORT` 下它就是"另一个 HTTP 客户端"，与底层是 WebFlux 还是 MVC 无关。** 统一用它的好处是整个工程（无论 MVC 还是响应式模块）共享一套测试客户端 API 与断言风格：

```java
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient // 显式触发 WebTestClient 自动配置（servlet 应用建议加上）
class MvcWebTestClientTest {

    @Autowired
    private WebTestClient webTestClient; // 对 MVC 应用发真实 HTTP
}
```

两个注意点：**MOCK 环境下也可用**——此时 `WebTestClient` 通过 `bindToApplicationContext(...)` 绑定 Mock 环境（Boot 自动完成），行为类似 MockMvc 但 API 是响应式风格，同样测不到真实网络栈；**依赖要求**——classpath 需要 `spring-webflux`（哪怕应用是 MVC 的），starter-test 不默认带它，需在 `testImplementation` 里补 `spring-webflux` + `reactor-test`。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webflux</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 3.3 绑定方式一览

`WebTestClient` 有四种绑定入口，决定测试的"真实度"：

| 绑定方式 | 容器 | 网络 | 适用 |
|----------|------|------|------|
| `@SpringBootTest(RANDOM_PORT)` 注入 | 全量 + 真实端口 | 真实 HTTP | 端到端 |
| `@WebFluxTest` 注入 | WebFlux 切片 | Mock | 响应式控制器切片测试 |
| `WebTestClient.bindToController(...)` | 无 | Mock | 纯控制器单元测试（对应 standaloneSetup） |
| `WebTestClient.bindToApplicationContext(...)` | 已有容器 | Mock | 对任意已建容器做 Mock 式请求 |

---

# 4. 三者对比与选择

## 4.1 MockMvc vs TestRestTemplate vs WebTestClient

| 维度 | MockMvc | TestRestTemplate | WebTestClient |
|------|---------|------------------|---------------|
| 运行环境要求 | MOCK | RANDOM_PORT/DEFINED_PORT | 两者皆可 |
| 真实 HTTP/网络栈 | ✗ | ✓ | ✓（绑定容器时） |
| 技术栈 | Servlet MVC | 任意（同步客户端） | WebFlux 与 MVC 皆可 |
| API 风格 | perform + ResultMatcher | RestTemplate 式（getForEntity/exchange） | 流式断言链（expectStatus/expectBody） |
| 异步/流式响应 | 需 asyncDispatch 二次分发，流式支持弱 | 不支持流式断言 | 原生支持（returnFlux + StepVerifier） |
| 4xx/5xx 行为 | 正常返回供断言 | 不抛异常，返回响应 | 不抛异常，进入断言链 |
| 典型用途 | 控制器切片测试、MVC 管线验证 | 传统 MVC 端到端 | 响应式端到端；MVC 端到端的新选择 |

选择准则：**测管线与切片 → MockMvc；测真实 HTTP 的 MVC 应用 → TestRestTemplate（存量）或 WebTestClient（新代码、想统一风格）；测 WebFlux → WebTestClient 无争议**。Boot 官方文档近年示例明显偏向 WebTestClient——它一套 API 覆盖两种技术栈、断言链可读性好、支持流式，是新项目的合理默认。

> **注意**：`WebTestClient` 测 servlet 应用需要 classpath 上有 `spring-webflux`（starter-test 不默认带），只加依赖不改应用代码——它此时只是一个恰好可用的非阻塞 HTTP 客户端，不会把应用变成响应式的。

## 4.2 与 MockMvc 的分工

**两层不是竞争关系而是互补关系：MockMvc 数量多、跑得快、定位准；真 HTTP 测试数量少、验证"网络栈与客户端配置"这最后一环。** 典型分配：每个 Controller 的路由/校验/异常映射用 `@WebMvcTest` + MockMvc 全覆盖；每条核心业务流程补 1-2 个 `RANDOM_PORT` 用例，走真实的登录 → 调用 → 落库链路。真 HTTP 用例多了套件会明显变慢（每个 RANDOM_PORT 容器占一个 Tomcat），优先检查这些容器是否被上下文缓存共享（`webEnvironment` 进缓存 key，第 5 篇）。

---

# 5. 实践注意点

## 5.1 端口管理

- **使用 `RANDOM_PORT` 避免冲突**：并行执行、CI 多任务、开发机多项目同时跑都安全；`DEFINED_PORT` 只留给"外部系统必须回调固定地址"的场景
- **通过 `@LocalServerPort` 注入端口**：不要写死端口号或从配置读 `server.port`（`RANDOM_PORT` 下配置值仍是 0/原值，实际端口在 `local.server.port`）
- **优先用注入的客户端而非手拼 URL**：`TestRestTemplate`/`WebTestClient` Bean 已预置 baseUrl，手拼 `http://localhost:" + port` 容易在 SSL/代理配置下失效

## 5.2 Cookie、Header 与拦截器

```java
import java.util.List;

// 给 TestRestTemplate 底层 RestTemplate 加拦截器：统一注入链路追踪头
rest.getRestTemplate().setInterceptors(List.of((req, body, ex) -> {
    req.getHeaders().add("X-Trace-Id", "t-1"); // 每个请求都带上
    return ex.execute(req, body);              // 放行执行
}));

// WebTestClient 等价手段：mutate() 派生带默认头的实例
WebTestClient withHeader = webTestClient.mutate()
        .defaultHeader("X-Trace-Id", "t-1")
        .build();
```

**Cookie/Session 状态默认不跨请求保持**——两个客户端都无状态，模拟登录态要么每次带认证头，要么手工把上一个响应的 `Set-Cookie` 塞进下一个请求（`TestRestTemplate` 需自己管理；`WebTestClient` 用 `exchange()` 返回的 `Cookie` 列表）。需要会话保持的场景考虑 `CookieManager` 配置或改用带 CookieStore 的 HttpClient。

## 5.3 断言风格统一

**建议全工程统一一种断言风格，避免 MockMvc 的 Matcher、TestRestTemplate 的 AssertJ、WebTestClient 的链式断言三种写法混杂。** 具体做法：Web 切片层用 MockMvc + `jsonPath`/`content().json`；端到端层收敛到 WebTestClient 的 `expectStatus()/expectBody()` 链，复杂响应体 `expectBody(Xxx.class)` 反序列化后转 AssertJ。

---

# 6. 测试外部 API：WireMock 简介

**前面的工具都在测"别人调我"；当我的应用调用外部 API（支付、短信、第三方数据）时，需要反方向的替身——WireMock 在本地起一个真实 HTTP 服务，按预设规则应答，让集成测试不依赖外网。**

```xml
<dependency>
    <groupId>org.wiremock.integrations</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>3.13.0</version>
    <scope>test</scope>
</dependency>
```

```java
import com.maciejwalkowiak.wiremock.spring.ConfigureWireMock;
import com.maciejwalkowiak.wiremock.spring.EnableWireMock;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest
@EnableWireMock({
    // 启动 WireMock 并把其地址自动写入 payment.gateway.url 属性
    @ConfigureWireMock(name = "payment-gateway", property = "payment.gateway.url")
})
class PaymentGatewayIT {

    @Autowired
    private PaymentService paymentService; // 内部通过 RestClient 调外部网关

    @Test
    void should_handle_successful_payment() {
        stubFor(post(urlEqualTo("/charge"))          // 预设：POST /charge
                .willReturn(okJson("{\"status\": \"SUCCESS\", \"txn\": \"t-1\"}")));

        PaymentResult result = paymentService.charge(new BigDecimal("100"));

        assertThat(result.status()).isEqualTo(PaymentStatus.SUCCESS);
        verify(postRequestedFor(urlEqualTo("/charge")) // 校验请求确实发出
                .withRequestBody(matchingJsonPath("$.amount", equalTo("100"))));
    }

    @Test
    void should_fallback_when_gateway_500() {
        stubFor(post(urlEqualTo("/charge"))
                .willReturn(serverError()));          // 预设外部服务故障

        assertThat(paymentService.charge(new BigDecimal("100")).status())
                .isEqualTo(PaymentStatus.FAILED);      // 验证降级逻辑
    }
}
```

WireMock 的核心能力：**请求匹配**（URL/方法/头/body 的 JsonPath 匹配）、**响应预设**（状态码、body、延迟、fault 注入——模拟超时与连接重置）、**请求校验**（`verify` 断言调用发生及次数）、**录制回放**（proxy 到真实服务录制桩数据）。与 `MockRestServiceServer`（第 7 篇 `@RestClientTest`）的分工：后者在客户端进程内拦截，更快但测不到真实 HTTP 传输；**WireMock 起真实服务，连客户端的超时与重试行为都能验证**——集成测试选 WireMock，客户端逻辑单测选 MockRestServiceServer。

> **行业实践**：WireMock 地址注入优先用 `@EnableWireMock`/`@ConfigureWireMock`（wiremock-spring-boot）或 Testcontainers 化的 WireMock 镜像，避免每个测试类手写 `@DynamicPropertySource`——注册方法的差异参与上下文缓存 key，写法不一致会导致容器无法共享（第 5 篇）。

---

# 7. 小结

| 主题 | 关键结论 |
|------|---------|
| 定位 | RANDOM_PORT + 真客户端验证网络栈、两侧序列化、过滤器真实顺序、客户端配置——MockMvc 的盲区 |
| TestRestTemplate | Boot 包装的 RestTemplate：错误状态码不抛异常、预置 baseUrl、`withBasicAuth`；仅 RANDOM_PORT/DEFINED_PORT 可用 |
| @LocalServerPort | 实际端口在 `local.server.port`，注解是它的类型安全注入 |
| WebTestClient | 流式断言链，WebFlux 与 MVC 通用；servlet 应用需补 spring-webflux 测试依赖；原生支持流式响应断言 |
| 选择 | 切片/MockMvc，端到端/WebTestClient（新代码）或 TestRestTemplate（存量）；两层互补不竞争 |
| 实践 | 随机端口 + 注入客户端 + 统一断言风格；Cookie/Session 默认不保持 |
| WireMock | 测"我调外部"：真实本地 HTTP 替身，支持匹配/预设/校验/故障注入；比 MockRestServiceServer 多覆盖传输层 |

至此 SpringBootTest 专题的工具链（容器、切片、MockMvc、真 HTTP）已闭环。下一篇进入 Mock 与测试替身的系统讨论：`@MockitoBean` 的机制、Mockito 用法边界与 Fake 的取舍。
