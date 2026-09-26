# 1. TestRestTemplate

## 1.1 定位与特点

TestRestTemplate 是 Spring Boot 对 `RestTemplate` 的测试封装，适合端到端 HTTP 测试：

- 自动配置 `HttpMessageConverter`
- 支持基础认证
- 与 `@SpringBootTest(webEnvironment = RANDOM_PORT)` 配合使用

## 1.2 典型用法

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpApiTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate rest;

    @Test
    void getUser() {
        ResponseEntity<String> resp = rest.getForEntity("http://localhost:" + port + "/users/1", String.class);
        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

## 1.3 认证方式

```java
TestRestTemplate authRest = rest.withBasicAuth("user", "pwd");
```

---

# 2. WebTestClient

## 2.1 响应式测试的首选

WebTestClient 最初用于 WebFlux，提供链式断言风格：

```java
webTestClient.get()
    .uri("/users/1")
    .exchange()
    .expectStatus().isOk()
    .expectBody()
    .jsonPath("$.id").isEqualTo(1);
```

## 2.2 在 MVC 中使用

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class MvcWebTestClientTest {

    @Autowired
    private WebTestClient webTestClient;
}
```

适合统一测试风格，但启动成本略高。

---

# 3. 选择建议

| 场景 | 推荐 |
|------|------|
| 传统 MVC | `MockMvc` / `TestRestTemplate` |
| WebFlux | `WebTestClient` |
| 需要真实端口 | `TestRestTemplate` / `WebTestClient` |
| 只测控制器 | `MockMvc` |

---

# 4. 实践注意点

## 4.1 端口管理

- 使用 `RANDOM_PORT` 避免冲突
- 通过 `@LocalServerPort` 注入端口

## 4.2 Cookie 与 Header

```java
rest.getRestTemplate().setInterceptors(List.of((req, body, ex) -> {
    req.getHeaders().add("X-Trace-Id", "t-1");
    return ex.execute(req, body);
}));
```

## 4.3 断言风格统一

建议统一使用 AssertJ 或 WebTestClient 自带断言链，避免风格混杂。
