Java 生态里有多种 HTTP 客户端，初学者最容易在不同 API 之间反复切换却说不清差异。本篇对比 `HttpURLConnection`、Java 11 `HttpClient`、OkHttp 和 Apache HttpClient，并用完整示例演示同步、异步、连接池和统一异常封装。

# 1. 客户端选型

## 1.1 HttpURLConnection

`HttpURLConnection` 是 JDK 自带的最老客户端：

```java
URL url = new URL("https://example.com/api/users");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setConnectTimeout(3000);
conn.setReadTimeout(5000);

int status = conn.getResponseCode();
try (InputStream in = status >= 400
        ? conn.getErrorStream()
        : conn.getInputStream()) {
    String body = new String(in.readAllBytes(), StandardCharsets.UTF_8);
    System.out.println(status + " " + body);
} finally {
    conn.disconnect();
}
```

常见坑：

- 必须关闭输入流，否则连接可能无法复用。
- 错误响应要从 `getErrorStream()` 读取。
- 重定向、代理、Cookie 的处理较繁琐。
- API 较老，不推荐新项目使用。

## 1.2 Java 11 HttpClient

JDK 11 引入的 `java.net.http.HttpClient` 是现代 Java 项目的首选：

- 支持同步和异步请求。
- 支持 HTTP/2。
- 内置连接池和 Cookie 管理。
- API 设计比 `HttpURLConnection` 清晰。

## 1.3 OkHttp

OkHttp 是 Square 开源的客户端，特点是：

- 连接池、拦截器、缓存、重试机制完善。
- Android 生态使用广泛，服务端也常见。
- 拦截器适合统一加 Header、日志、鉴权、重试。

```java
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(3, TimeUnit.SECONDS)
    .readTimeout(10, TimeUnit.SECONDS)
    .addInterceptor(chain -> {
        Request request = chain.request().newBuilder()
            .header("Authorization", "Bearer token")
            .build();
        return chain.proceed(request);
    })
    .build();
```

## 1.4 Apache HttpClient

Apache HttpClient 常见于企业项目和老系统：

- 连接池、路由、重试、代理配置成熟。
- API 较冗长，但控制粒度细。
- Spring 生态中常与 `RestTemplate` 一起出现。

新项目如果没有历史包袱，可以优先 Java 11 `HttpClient` 或 OkHttp；需要复杂连接管理时再考虑 Apache HttpClient。

# 2. Java HttpClient 实战

## 2.1 同步请求

```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(3))
    .version(HttpClient.Version.HTTP_2)
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://example.com/api/users"))
    .timeout(Duration.ofSeconds(10))
    .header("Accept", "application/json")
    .GET()
    .build();

HttpResponse<String> response = client.send(
    request,
    HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8)
);

System.out.println(response.statusCode());
System.out.println(response.body());
```

POST JSON：

```java
String json = """
    {"name":"alice","email":"alice@example.com"}
    """;

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://example.com/api/users"))
    .timeout(Duration.ofSeconds(10))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(json, StandardCharsets.UTF_8))
    .build();
```

## 2.2 异步请求

```java
CompletableFuture<HttpResponse<String>> future = client.sendAsync(
    request,
    HttpResponse.BodyHandlers.ofString()
);

future.thenApply(HttpResponse::body)
      .thenAccept(System.out::println)
      .exceptionally(ex -> {
          ex.printStackTrace(); // 记录日志，不要只打印
          return null;
      });
```

异步请求的异常不会直接抛出，而是包在 `CompletionException` 中。处理时要区分：

- 连接失败、超时等网络异常。
- 4xx/5xx 业务响应，通常不会抛异常，需要检查状态码。
- 响应体读取失败。

## 2.3 连接池与超时

Java 11 `HttpClient` 的连接池由实现内部管理，常用配置：

```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(3)) // 建立连接超时
    .executor(Executors.newFixedThreadPool(16)) // 异步请求线程池
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();
```

单次请求超时：

```java
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://example.com"))
    .timeout(Duration.ofSeconds(10)) // 整个请求的超时
    .GET()
    .build();
```

⚠ `HttpClient` 的 `connectTimeout` 只覆盖连接建立，不覆盖服务端处理时间。必须同时设置请求级 `timeout`，否则服务端迟迟不返回时仍可能长时间等待。

# 3. 请求封装

## 3.1 统一异常

```java
public class HttpCallException extends RuntimeException {
    private final int statusCode;

    public HttpCallException(String message, int statusCode, Throwable cause) {
        super(message, cause);
        this.statusCode = statusCode;
    }

    public int getStatusCode() {
        return statusCode;
    }
}
```

调用层统一转换：

```java
try {
    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    if (response.statusCode() >= 400) {
        throw new HttpCallException("下游返回错误", response.statusCode(), null);
    }
    return response.body();
} catch (IOException e) {
    throw new HttpCallException("网络调用失败", 0, e);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断标记
    throw new HttpCallException("请求被中断", 0, e);
}
```

★ **关键认知**：网络错误和业务错误要分开处理。连接超时、DNS 失败、TLS 失败属于调用失败；4xx/5xx 是拿到了响应但业务不成功。混在一起会让上层无法决定是否重试。

## 3.2 重试与日志

只对幂等请求重试：

```java
public String getWithRetry(String url) {
    IOException last = null;
    for (int i = 0; i < 3; i++) {
        try {
            return doGet(url);
        } catch (IOException e) {
            last = e;
            sleep(100L * (1L << i) + ThreadLocalRandom.current().nextLong(50));
        }
    }
    throw new HttpCallException("重试失败", 0, last);
}
```

日志至少记录：

- 请求方法、URL、状态码、耗时。
- 请求 ID、下游服务名。
- 错误类型和简短原因。
- 不要记录完整 Token、Cookie、密码。

# 4. 对比与选型建议

| 客户端 | 依赖 | 同步/异步 | HTTP/2 | 连接池 | 适用场景 |
|--------|------|-----------|--------|--------|----------|
| HttpURLConnection | JDK | 同步 | 否 | 有限 | 老项目、简单请求 |
| Java 11 HttpClient | JDK | 同步/异步 | 是 | 内置 | 新项目首选 |
| OkHttp | 第三方 | 同步/异步 | 是 | 完善 | Android、拦截器需求 |
| Apache HttpClient | 第三方 | 同步/异步 | 部分 | 完善 | 企业项目、复杂连接管理 |

# 5. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 响应体未关闭 | 连接泄漏 | try-with-resources |
| 只设连接超时 | 请求可能无限等待 | 同时设置请求超时 |
| 每次请求新建客户端 | 连接池和线程池反复创建 | 复用客户端 |
| 对 POST 盲目重试 | 重复下单 | 只重试幂等请求 |
| 忽略状态码 | 把错误响应当成功 | 检查 4xx/5xx |
| 日志打印完整请求体 | 泄露密码和 Token | 脱敏后再记录 |

# 6. 动手练习

封装一个 `HttpClient` 工具类，要求：

1. 支持 GET 和 POST JSON。
2. 连接超时 3 秒，请求超时 10 秒。
3. 4xx/5xx 抛出统一异常，包含状态码。
4. 网络异常和中断异常分开处理。
5. 只对 GET 自动重试 2 次，并带随机退避。
6. 记录方法、URL、状态码和耗时，不记录敏感信息。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 客户端选型 | 新项目优先 Java 11 HttpClient，Android/拦截器选 OkHttp |
| 同步异步 | 异步异常要显式处理，检查状态码 |
| 超时 | 连接超时和请求超时都要设置 |
| 封装 | 统一异常、重试、日志和脱敏 |
| 复用 | 客户端实例应复用，避免连接池反复创建 |

下一篇进入跨域与安全，讲清同源策略、CORS 预检、XSS/CSRF/SSRF 和安全响应头。
