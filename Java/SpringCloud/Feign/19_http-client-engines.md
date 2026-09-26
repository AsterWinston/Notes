
# 1. 概述

Feign 的 `Client` 接口抽象了底层 HTTP 通信。Spring Cloud OpenFeign 支持三种 HTTP 引擎：

| 引擎 | 类 | 连接池 | 默认 |
|------|-----|--------|------|
| **HttpURLConnection** | `Client.Default` | 无 | 是 |
| **Apache HttpClient** | `ApacheHttpClient` | 有 (PoolingHttpClientConnectionManager) | — |
| **OkHttp** | `OkHttpClient` | 有 (ConnectionPool) | — |

# 2. Client.Default（JRE 原生）

```java
public static class Default implements Client {

    @Override
    public Response execute(Request request, Options options) throws IOException {
        HttpURLConnection connection = (HttpURLConnection) new URL(request.url())
                .openConnection();
        connection.setConnectTimeout(options.connectTimeoutMillis());
        connection.setReadTimeout(options.readTimeoutMillis());
        connection.setRequestMethod(request.httpMethod().name());

        // 写请求头
        for (Map.Entry<String, Collection<String>> header : request.headers().entrySet()) {
            connection.setRequestProperty(header.getKey(),
                    String.join(",", header.getValue()));
        }

        // 写请求体
        if (request.body() != null) {
            connection.setDoOutput(true);
            connection.getOutputStream().write(request.body());
        }

        // 读响应
        int status = connection.getResponseCode();
        InputStream body = status >= 400
                ? connection.getErrorStream()
                : connection.getInputStream();
        // ...
    }
}
```

**特点**：
- 无需额外依赖
- **每次请求新建 TCP 连接**（无连接池）
- HTTP/1.1 Keep-Alive 支持有限
- 不支持 HTTP/2
- **生产环境不推荐**

# 3. Apache HttpClient

## 3.1 启用

```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        enabled: true
        max-connections: 200
        max-connections-per-route: 50
        time-to-live: 900
        time-to-live-unit: seconds
```

Maven 自动依赖（Spring Cloud 已包含）：

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
</dependency>
```

## 3.2 特性

- **连接池**：`PoolingHttpClientConnectionManager`，避免频繁建连/断连
- **HTTP/2 支持**（Apache HttpClient 5.x）
- **连接复用**：Keep-Alive 策略
- **SSL 可控**：支持双向认证
- **连接驱逐**：后台线程定期清理过期连接

## 3.3 连接池参数调优

```java
@Bean
public HttpClient apacheHttpClient() {
    PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(300);                        // 总连接数
    cm.setDefaultMaxPerRoute(30);               // 每路由连接数
    cm.setValidateAfterInactivity(5000);        // 空闲 5s 后验证连接

    RequestConfig config = RequestConfig.custom()
            .setConnectTimeout(5000)
            .setSocketTimeout(30000)
            .setConnectionRequestTimeout(3000)  // 从池中获取连接超时
            .build();

    return HttpClientBuilder.create()
            .setConnectionManager(cm)
            .setDefaultRequestConfig(config)
            .evictIdleConnections(60, TimeUnit.SECONDS)  // 60s 清理空闲连接
            .setRetryHandler(new DefaultHttpRequestRetryHandler(0, false))
            .build();
}
```

# 4. OkHttp

## 4.1 启用

```yaml
spring:
  cloud:
    openfeign:
      okhttp:
        enabled: true
```

Maven 依赖：

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

## 4.2 特性

- **连接池**：默认 5 空闲连接，5 分钟存活
- **HTTP/2 原生支持**
- **自动 Gzip 解压**
- **连接复用 + 自动恢复**
- **拦截器链**：可插入应用层拦截器
- **DNS 缓存**

## 4.3 自定义配置

```java
@Bean
public okhttp3.OkHttpClient okHttpClient() {
    return new okhttp3.OkHttpClient.Builder()
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .connectionPool(new ConnectionPool(
                    20,                              // maxIdleConnections
                    10, TimeUnit.MINUTES            // keepAliveDuration
            ))
            .addInterceptor(new LoggingInterceptor())
            .addInterceptor(chain -> {
                Request original = chain.request();
                Request request = original.newBuilder()
                        .header("X-Service", "order-service")
                        .build();
                return chain.proceed(request);
            })
            .followRedirects(true)
            .followSslRedirects(true)
            .retryOnConnectionFailure(false)         // 关闭自动重试（由 Feign Retryer 控制）
            .build();
}
```

# 5. 引擎对比矩阵

| 特性 | HttpURLConnection | Apache HttpClient 4.x | Apache HttpClient 5.x | OkHttp 3.x / 4.x |
|------|-------------------|----------------------|----------------------|-------------------|
| 连接池 | 无 | 有 | 有 | 有 |
| HTTP/2 | 不支持 | 通过升级支持 | 原生支持 | 原生支持 |
| 连接复用 | 有限 | 完善 | 完善 | 完善 |
| 异步支持 | 无 | 有（5.x） | 原生 | 原生 |
| 拦截器 | 无 | 有 | 有 | 有 |
| 自动重试 | 无 | 可配置 | 可配置 | 可配置 |
| Gzip 自动解压 | 无 | 手动 | 手动 | 自动 |
| 包大小 | JRE 内置 | ~700KB | ~1MB | ~500KB |
| 社区活跃度 | — | 高 | 高 | 高 |
| Spring 默认 | 是 | 否 | 否 | 否 |

# 6. 性能测试参考

```
场景：1000 次请求，并发 50，本地回环

引擎                  平均响应时间     P99         QPS
HttpURLConnection      8ms            25ms        450
Apache HttpClient      3ms            10ms        1200
OkHttp                 2ms            8ms         1400
```

> 数据仅供参考，实际性能取决于网络环境、请求/响应大小、SSL 等因素。

# 7. 自定义 SSL

## 7.1 Apache HttpClient SSL

```java
@Bean
public HttpClient sslHttpClient() throws Exception {
    // 加载证书
    SSLContext sslContext = SSLContexts.custom()
            .loadTrustMaterial(new File("truststore.jks"), "password".toCharArray())
            .build();

    return HttpClientBuilder.create()
            .setSSLContext(sslContext)
            .setSSLHostnameVerifier(NoopHostnameVerifier.INSTANCE) // 仅开发环境
            .build();
}
```

## 7.2 OkHttp SSL

```java
@Bean
public okhttp3.OkHttpClient sslOkHttpClient() throws Exception {
    TrustManager[] trustAllCerts = new TrustManager[]{
        new X509TrustManager() {
            public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
            public void checkClientTrusted(X509Certificate[] certs, String authType) { }
            public void checkServerTrusted(X509Certificate[] certs, String authType) { }
        }
    };

    SSLContext sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, trustAllCerts, new SecureRandom());

    return new okhttp3.OkHttpClient.Builder()
            .sslSocketFactory(sslContext.getSocketFactory(),
                    (X509TrustManager) trustAllCerts[0])
            .hostnameVerifier((hostname, session) -> true) // 仅开发环境
            .build();
}
```

# 8. 引擎切换

## 8.1 方式 1：YAML 配置

```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        enabled: false    # 关闭 Apache
      okhttp:
        enabled: true      # 开启 OkHttp
```

当两者都启用时，OkHttp 优先。

## 8.2 方式 2：排除依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-httpclient</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

# 9. 选择建议

| 场景 | 推荐 |
|------|------|
| **开发/测试** | HttpURLConnection（默认，零配置） |
| **生产（通用）** | OkHttp（性能好，HTTP/2 原生支持） |
| **生产（需大量调优）** | Apache HttpClient（连接池调优空间大） |
| **已有 Feign 项目优化** | OkHttp（引入即可，即时收益） |
| **需要 HTTP/2 + gRPC 互通** | OkHttp（基于相同的底层协议栈） |
