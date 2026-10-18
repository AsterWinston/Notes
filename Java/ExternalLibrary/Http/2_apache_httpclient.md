# 1. 概述与简介

## 1.1 Apache HttpClient简介

Apache HttpClient（现更名为Apache HttpComponents Client）是Apache软件基金会旗下的开源HTTP客户端库，是Java生态中最老牌、最成熟的HTTP客户端之一。其历史可以追溯到2001年的Commons HttpClient 1.x，经过20多年的发展，已成为大多数Java企业应用的标配。

### 1.1.1 版本历史

| 版本 | 时间 | 重要变化 |
|------|------|---------|
| Commons HttpClient 1.x | 2001 | 最初的HTTP客户端库，属于Commons项目 |
| Commons HttpClient 2.x/3.x | 2003-2007 | 广泛使用，但API设计较为陈旧 |
| HttpComponents HttpClient 4.x | 2008 | 完全重写，引入I/O模型、连接管理器等 |
| HttpComponents HttpClient 4.5.x | 2015 | 长期维护版本，功能稳定完善 |
| HttpComponents HttpClient 5.x | 2021 | 重大升级，支持HTTP/2，简化API |

目前企业中使用最广泛的是 **4.5.x** 版本，而 **5.x** 版本正在逐步普及。

## 1.2 核心特性

| 特性 | 说明 |
|------|------|
| 协议支持 | HTTP/1.0、HTTP/1.1、HTTP/2（5.x） |
| 连接管理 | 强大的连接池管理，支持连接复用、TTL、过期清理 |
| 请求管道 | 经典的HttpGet/HttpPost等请求对象 |
| 响应处理 | 多种响应处理方式（ResponseHandler） |
| 拦截器 | 请求拦截器（HttpRequestInterceptor）和响应拦截器（HttpResponseInterceptor） |
| 重试机制 | 可配置的重试策略（HttpRequestRetryHandler） |
| Cookie管理 | 丰富的Cookie策略和存储 |
| 认证 | 支持Basic、Digest、NTLM等多种认证方案 |
| SSL/TLS | 全面的SSL配置，支持自定义证书验证 |
| 缓存 | 兼容HTTP缓存规范的缓存模块 |
| 异步支持 | 基于NIO的Async HTTP Client（5.x） |
| 流式API | 5.x提供Fluent API，更简洁易用 |

## 1.3 应用场景

Apache HttpClient广泛应用于：
- 后端微服务之间的HTTP调用
- 与第三方API集成
- 爬虫与数据采集
- 文件上传/下载
- 需要精细控制HTTP连接管理的企业应用
- 与Spring RestTemplate/RestClient集成

# 2. 核心API使用

## 2.1 引入依赖

### 2.1.1 HttpClient 4.5.x（Maven）

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.14</version>
</dependency>
```

### 2.1.2 HttpClient 5.x（Maven）

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.3.1</version>
</dependency>

<!-- 如需缓存 -->
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5-cache</artifactId>
    <version>5.3.1</version>
</dependency>

<!-- 如需异步客户端 -->
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5-fluent</artifactId>
    <version>5.3.1</version>
</dependency>
```

## 2.2 HttpClientBuilder配置

### 2.2.1 基础配置（4.5.x）

```java
CloseableHttpClient httpClient = HttpClients.custom()
    // 连接超时
    .setConnectionTimeToLive(30, TimeUnit.SECONDS)
    // 连接管理器
    .setConnectionManager(poolingConnectionManager)
    // 默认请求配置
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(5000)           // 连接建立超时
        .setSocketTimeout(10000)           // Socket读取超时
        .setConnectionRequestTimeout(3000) // 从连接池获取连接的超时
        .setRedirectsEnabled(true)         // 是否自动重定向
        .setMaxRedirects(5)                // 最大重定向次数
        .build())
    // 重试策略
    .setRetryHandler(retryHandler)
    // 拦截器
    .addInterceptorLast(requestInterceptor)
    .addInterceptorLast(responseInterceptor)
    // Cookie存储
    .setDefaultCookieStore(cookieStore)
    // 认证
    .setDefaultCredentialsProvider(credentialsProvider)
    // SSL
    .setSSLContext(sslContext)
    .setSSLHostnameVerifier(hostnameVerifier)
    .build();
```

### 2.2.2 基础配置（5.x）

```java
CloseableHttpClient httpClient = HttpClientBuilder.create()
    .setConnectionManager(poolingConnectionManager)
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(5000, TimeUnit.MILLISECONDS)
        .setResponseTimeout(10000, TimeUnit.MILLISECONDS)
        .setConnectionRequestTimeout(3000, TimeUnit.MILLISECONDS)
        .setRedirectsEnabled(true)
        .build())
    .setRetryStrategy(retryStrategy)
    .addRequestInterceptorLast(requestInterceptor)
    .addResponseInterceptorLast(responseInterceptor)
    .build();
```

## 2.3 经典API vs Fluent API

### 2.3.1 经典API（4.x/5.x通用风格）

```java
CloseableHttpClient client = HttpClients.createDefault();

// GET请求
HttpGet httpGet = new HttpGet("https://api.example.com/users");
try (CloseableHttpResponse response = client.execute(httpGet)) {
    int statusCode = response.getStatusLine().getStatusCode();
    String responseBody = EntityUtils.toString(response.getEntity());
    System.out.println("状态码: " + statusCode);
    System.out.println("响应体: " + responseBody);
}

// POST JSON请求
HttpPost httpPost = new HttpPost("https://api.example.com/users");
httpPost.setHeader("Content-Type", "application/json");
httpPost.setHeader("Authorization", "Bearer token123");
String json = "{\"name\":\"张三\",\"email\":\"zhangsan@example.com\"}";
httpPost.setEntity(new StringEntity(json, ContentType.APPLICATION_JSON));

try (CloseableHttpResponse response = client.execute(httpPost)) {
    String result = EntityUtils.toString(response.getEntity());
    System.out.println(result);
}
```

### 2.3.2 Fluent API（5.x）

```java
// 5.x的Fluent API更加简洁（类似OkHttp的链式调用）
import org.apache.hc.client5.http.fluent.*;

// GET请求
String result = Request.get("https://api.example.com/users")
    .addHeader("Authorization", "Bearer token123")
    .execute()
    .returnContent()
    .asString();

// POST请求
String postResult = Request.post("https://api.example.com/users")
    .addHeader("Content-Type", "application/json")
    .bodyString("{\"name\":\"张三\"}", ContentType.APPLICATION_JSON)
    .execute()
    .returnContent()
    .asString();

// 处理响应码
Response response = Request.get("https://api.example.com/users")
    .execute();
int statusCode = response.returnResponse().getCode();
String body = response.returnContent().asString();

// 异步Fluent API
Future<String> future = Request.get("https://api.example.com/users")
    .execute()
    .returnContent()
    .asFuture();
String asyncResult = future.get();
```

## 2.4 GET/POST/PUT/DELETE请求示例

### 2.4.1 GET请求

```java
public String doGet(String url) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        HttpGet httpGet = new HttpGet(url);
        httpGet.setHeader("Accept", "application/json");
        
        try (CloseableHttpResponse response = client.execute(httpGet)) {
            int statusCode = response.getStatusLine().getStatusCode();
            if (statusCode >= 200 && statusCode < 300) {
                return EntityUtils.toString(response.getEntity());
            } else {
                throw new IOException("请求失败，状态码: " + statusCode);
            }
        }
    }
}
```

### 2.4.2 POST请求

```java
public String doPost(String url, String jsonBody) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        HttpPost httpPost = new HttpPost(url);
        httpPost.setHeader("Content-Type", "application/json");
        
        // 设置请求体
        StringEntity entity = new StringEntity(jsonBody, StandardCharsets.UTF_8);
        httpPost.setEntity(entity);
        
        try (CloseableHttpResponse response = client.execute(httpPost)) {
            return EntityUtils.toString(response.getEntity());
        }
    }
}

// Multipart表单上传
public String uploadFile(String url, File file) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        HttpPost httpPost = new HttpPost(url);
        
        MultipartEntityBuilder builder = MultipartEntityBuilder.create();
        builder.setMode(HttpMultipartMode.BROWSER_COMPATIBLE);
        builder.addBinaryBody("file", file, ContentType.DEFAULT_BINARY, file.getName());
        builder.addTextBody("description", "文件描述", ContentType.TEXT_PLAIN);
        
        httpPost.setEntity(builder.build());
        
        try (CloseableHttpResponse response = client.execute(httpPost)) {
            return EntityUtils.toString(response.getEntity());
        }
    }
}
```

### 2.4.3 PUT请求

```java
public String doPut(String url, String jsonBody) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        HttpPut httpPut = new HttpPut(url);
        httpPut.setHeader("Content-Type", "application/json");
        httpPut.setEntity(new StringEntity(jsonBody, StandardCharsets.UTF_8));
        
        try (CloseableHttpResponse response = client.execute(httpPut)) {
            return EntityUtils.toString(response.getEntity());
        }
    }
}
```

### 2.4.4 DELETE请求

```java
public String doDelete(String url) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        HttpDelete httpDelete = new HttpDelete(url);
        
        try (CloseableHttpResponse response = client.execute(httpDelete)) {
            return EntityUtils.toString(response.getEntity());
        }
    }
}
```

### 2.4.5 带查询参数的GET请求

```java
public String doGetWithParams(String baseUrl, Map<String, String> params) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        URIBuilder uriBuilder = new URIBuilder(baseUrl);
        for (Map.Entry<String, String> entry : params.entrySet()) {
            uriBuilder.addParameter(entry.getKey(), entry.getValue());
        }
        
        HttpGet httpGet = new HttpGet(uriBuilder.build());
        
        try (CloseableHttpResponse response = client.execute(httpGet)) {
            return EntityUtils.toString(response.getEntity());
        }
    }
}

// 使用
Map<String, String> params = new HashMap<>();
params.put("page", "1");
params.put("size", "20");
params.put("sort", "createdAt,desc");
String result = doGetWithParams("https://api.example.com/users", params);
```

## 2.5 响应处理

### 2.5.1 使用ResponseHandler

ResponseHandler是Apache HttpClient推荐的安全处理响应的方式，它会自动管理连接的释放。

```java
// 基础ResponseHandler
public String executeWithHandler(String url) throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        HttpGet httpGet = new HttpGet(url);
        
        return client.execute(httpGet, response -> {
            int status = response.getStatusLine().getStatusCode();
            if (status >= 200 && status < 300) {
                return EntityUtils.toString(response.getEntity());
            } else {
                throw new ClientProtocolException("Unexpected response status: " + status);
            }
        });
    }
}
```

### 2.5.2 泛型ResponseHandler

```java
public class JsonResponseHandler<T> implements ResponseHandler<T> {
    private final Class<T> clazz;
    private final ObjectMapper objectMapper;
    
    public JsonResponseHandler(Class<T> clazz) {
        this.clazz = clazz;
        this.objectMapper = new ObjectMapper();
    }
    
    @Override
    public T handleResponse(HttpResponse response) throws IOException {
        int status = response.getStatusLine().getStatusCode();
        String json = EntityUtils.toString(response.getEntity());
        
        if (status >= 200 && status < 300) {
            return objectMapper.readValue(json, clazz);
        } else {
            throw new IOException("Request failed with status " + status + ": " + json);
        }
    }
}

// 使用
User user = client.execute(httpGet, new JsonResponseHandler<>(User.class));
```

### 2.5.3 原始响应处理（需手动管理资源）

```java
CloseableHttpClient client = HttpClients.createDefault();
HttpGet httpGet = new HttpGet("https://api.example.com/file");
CloseableHttpResponse response = null;

try {
    response = client.execute(httpGet);
    HttpEntity entity = response.getEntity();
    
    if (entity != null) {
        // 将响应体写入文件
        try (FileOutputStream out = new FileOutputStream("downloaded_file.zip")) {
            entity.writeTo(out);
        }
        
        // 或使用BufferedStream逐行处理大文件
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(entity.getContent(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                processLine(line);
            }
        }
    }
} finally {
    if (response != null) {
        response.close(); // 释放连接
    }
    client.close();
}
```

# 3. 高级特性

## 3.1 请求拦截器与响应拦截器

Apache HttpClient的拦截器机制类似于OkHttp的Interceptor，但分为`HttpRequestInterceptor`和`HttpResponseInterceptor`两种。

### 3.1.1 请求拦截器

```java
// 日志记录请求拦截器
public class LoggingRequestInterceptor implements HttpRequestInterceptor {
    private static final Logger log = LoggerFactory.getLogger(LoggingRequestInterceptor.class);
    
    @Override
    public void process(HttpRequest request, HttpContext context) 
            throws HttpException, IOException {
        // 记录请求信息
        log.info("→ {} {}", request.getRequestLine().getMethod(), 
                 request.getRequestLine().getUri());
        
        // 添加请求头
        request.addHeader("X-Request-Id", UUID.randomUUID().toString());
        request.addHeader("X-Source", "my-application");
        
        // 获取请求体（如果有）
        if (request instanceof HttpEntityEnclosingRequest) {
            HttpEntity entity = ((HttpEntityEnclosingRequest) request).getEntity();
            if (entity != null) {
                log.debug("Request body length: {}", entity.getContentLength());
            }
        }
    }
}
```

### 3.1.2 响应拦截器

```java
// 响应日志拦截器
public class LoggingResponseInterceptor implements HttpResponseInterceptor {
    private static final Logger log = LoggerFactory.getLogger(LoggingResponseInterceptor.class);
    
    @Override
    public void process(HttpResponse response, HttpContext context) 
            throws HttpException, IOException {
        // 记录响应状态码
        log.info("← {} {}", response.getStatusLine().getStatusCode(),
                 response.getStatusLine().getReasonPhrase());
        
        // 记录响应头
        for (Header header : response.getAllHeaders()) {
            log.debug("  {}: {}", header.getName(), header.getValue());
        }
    }
}
```

### 3.1.3 注册拦截器

```java
CloseableHttpClient client = HttpClients.custom()
    .addInterceptorLast(new LoggingRequestInterceptor())
    .addInterceptorLast(new LoggingResponseInterceptor())
    // 也可以加在最前面
    .addInterceptorFirst(new SecurityHeadersInterceptor())
    .build();
```

### 3.1.4 5.x版本拦截器（略有不同）

```java
// 5.x的拦截器接口改为 HttpRequestInterceptor 和 HttpResponseInterceptor
// 位于 org.apache.hc.core5.http 包下

CloseableHttpClient client = HttpClientBuilder.create()
    .addRequestInterceptorLast((request, entity, context) -> {
        request.setHeader("X-Request-Id", UUID.randomUUID().toString());
    })
    .addResponseInterceptorLast((response, entity, context) -> {
        int status = response.getCode();
        if (status >= 400) {
            log.warn("Request failed with status: {}", status);
        }
    })
    .build();
```

## 3.2 重试机制

### 3.2.1 HttpRequestRetryHandler（4.x）

```java
HttpRequestRetryHandler retryHandler = (exception, executionCount, context) -> {
    // 超过最大重试次数则不再重试
    if (executionCount >= 3) {
        log.warn("已达到最大重试次数: {}", executionCount);
        return false;
    }
    
    // 对于InterruptedIOException不重试（超时已被中断）
    if (exception instanceof InterruptedIOException) {
        return false;
    }
    
    // 对于SSL异常不重试
    if (exception instanceof SSLException) {
        return false;
    }
    
    // 对于UnknownHostException不重试（DNS解析失败）
    if (exception instanceof UnknownHostException) {
        return false;
    }
    
    // 其他IO异常重试
    // 休眠后进行重试（指数退避）
    try {
        long waitMs = (long) Math.pow(2, executionCount - 1) * 1000;
        Thread.sleep(waitMs);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return false;
    }
    
    log.info("第{}次重试...", executionCount);
    return true;
};

CloseableHttpClient client = HttpClients.custom()
    .setRetryHandler(retryHandler)
    .build();
```

### 3.2.2 RetryStrategy（5.x）

```java
// 5.x使用更清晰的RetryStrategy
HttpRetryStrategy retryStrategy = (response, executionCount, context) -> {
    if (executionCount >= 3) {
        return false;
    }
    
    // 根据状态码决定是否重试（如5xx服务器错误）
    int statusCode = response.getCode();
    if (statusCode >= 500) {
        // 指数退避：1s, 2s, 4s
        long waitMs = (long) Math.pow(2, executionCount - 1) * 1000;
        try {
            Thread.sleep(waitMs);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
        return true;
    }
    
    return false;
};

CloseableHttpClient client = HttpClientBuilder.create()
    .setRetryStrategy(retryStrategy)
    .build();
```

### 3.2.3 StandardHttpRequestRetryHandler

```java
// Apache HttpClient内置的标准重试处理器
// 默认重试3次，仅重试幂等请求（GET, HEAD, PUT, DELETE, OPTIONS, TRACE）
// 不重试非幂等请求（POST, PATCH）
HttpRequestRetryHandler standardRetry = new StandardHttpRequestRetryHandler(3, true);
// 第二个参数 requestSentRetryEnabled：如果请求已发送但未收到响应时是否重试
```

## 3.3 Cookie管理

```java
// 创建Cookie存储
BasicCookieStore cookieStore = new BasicCookieStore();

// 手动添加Cookie
BasicClientCookie cookie = new BasicClientCookie("sessionId", "abc123");
cookie.setDomain(".example.com");
cookie.setPath("/");
cookie.setSecure(false);
cookie.setExpiryDate(new Date(System.currentTimeMillis() + 3600 * 1000)); // 1小时后过期
cookieStore.addCookie(cookie);

// Cookie策略
// 4.x支持的CookieSpecs:
//   - CookieSpecs.STANDARD: RFC 6265规范（推荐）
//   - CookieSpecs.STANDARD_STRICT: 严格RFC 6265
//   - CookieSpecs.NETSCAPE: 兼容老旧的Netscape规范
//   - CookieSpecs.DEFAULT: 默认兼容模式

RequestConfig globalConfig = RequestConfig.custom()
    .setCookieSpec(CookieSpecs.STANDARD)
    .build();

CloseableHttpClient client = HttpClients.custom()
    .setDefaultCookieStore(cookieStore)
    .setDefaultRequestConfig(globalConfig)
    .build();

// 发起请求后，响应中的Set-Cookie会自动存入cookieStore
HttpGet httpGet = new HttpGet("https://api.example.com/login");
try (CloseableHttpResponse response = client.execute(httpGet)) {
    // CookieStore自动更新
    List<Cookie> cookies = cookieStore.getCookies();
    cookies.forEach(c -> 
        System.out.println(c.getName() + " = " + c.getValue()));
}
```

## 3.4 认证支持

### 3.4.1 Basic认证

```java
CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
credentialsProvider.setCredentials(
    new AuthScope("api.example.com", 443, AuthScope.ANY_REALM),
    new UsernamePasswordCredentials("username", "password".toCharArray())
);

CloseableHttpClient client = HttpClients.custom()
    .setDefaultCredentialsProvider(credentialsProvider)
    .build();

// 或者在请求中添加Authorization头
// 注意：Apache HttpClient在处理认证时，会先发送不带认证的请求
// 收到401后，根据AuthScope查找Credentials，然后重新发送带认证的请求
// 如果不想多一次往返，可以直接添加请求头
HttpGet httpGet = new HttpGet("https://api.example.com/protected");
String auth = "username:password";
byte[] encodedAuth = Base64.getEncoder().encode(auth.getBytes(StandardCharsets.UTF_8));
httpGet.setHeader("Authorization", "Basic " + new String(encodedAuth));
```

### 3.4.2 Bearer Token认证

```java
// 使用拦截器自动添加Token
CloseableHttpClient client = HttpClients.custom()
    .addInterceptorLast((HttpRequestInterceptor) (request, context) -> {
        if (!(request instanceof HttpUriRequest)) {
            return;
        }
        request.setHeader("Authorization", "Bearer " + getAccessToken());
    })
    .build();

// Token刷新逻辑
public class TokenRefreshInterceptor implements HttpRequestInterceptor {
    private volatile String accessToken;
    private volatile long expiresAt;
    
    @Override
    public void process(HttpRequest request, HttpContext context) {
        // 检查Token是否即将过期，提前刷新
        if (accessToken == null || System.currentTimeMillis() > expiresAt - 60000) {
            refreshToken();
        }
        request.setHeader("Authorization", "Bearer " + accessToken);
    }
    
    private synchronized void refreshToken() {
        // 调用认证服务获取新Token
        // this.accessToken = ...
        // this.expiresAt = System.currentTimeMillis() + expiresIn * 1000;
    }
}
```

### 3.4.3 NTLM认证

```java
// NTLM认证（用于Windows集成认证）
CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
credentialsProvider.setCredentials(
    new AuthScope("intranet.example.com", 80),
    new NTCredentials("username", "password".toCharArray(), "workstation", "domain")
);

CloseableHttpClient client = HttpClients.custom()
    .setDefaultCredentialsProvider(credentialsProvider)
    .build();
```

## 3.5 SSL/TLS配置

### 3.5.1 自定义SSL上下文

```java
// 加载信任证书
KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
try (InputStream instream = new FileInputStream("my-truststore.jks")) {
    trustStore.load(instream, "password".toCharArray());
}

SSLContext sslContext = SSLContexts.custom()
    .loadTrustMaterial(trustStore, null)
    .build();

CloseableHttpClient client = HttpClients.custom()
    .setSSLContext(sslContext)
    .setSSLHostnameVerifier(new StrictHostnameVerifier())
    .build();
```

### 3.5.2 跳过SSL验证（仅开发/测试环境）

```java
@Deprecated
public CloseableHttpClient createInsecureHttpClient() throws Exception {
    // 信任所有证书
    SSLContext sslContext = SSLContexts.custom()
        .loadTrustMaterial(null, (chain, authType) -> true)
        .build();
    
    return HttpClients.custom()
        .setSSLContext(sslContext)
        .setSSLHostnameVerifier(NoopHostnameVerifier.INSTANCE)
        .build();
}
```

### 3.5.3 5.x SSL配置

```java
// 5.x SSL配置方式类似，但包名变更
SSLContext sslContext = SSLContextBuilder.create()
    .loadTrustMaterial(trustStore, null)
    .build();

CloseableHttpClient client = HttpClientBuilder.create()
    .setDefaultTlsConfig(TlsConfig.custom()
        .setVersion(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
        .setHandshakeTimeout(10, TimeUnit.SECONDS)
        .build())
    .build();
```

## 3.6 缓存模块

Apache HttpClient提供了兼容HTTP缓存规范的缓存模块。

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient-cache</artifactId>
    <version>4.5.14</version>
</dependency>
```

```java
// 创建缓存HttpClient
CacheConfig cacheConfig = CacheConfig.custom()
    .setMaxCacheEntries(1000)          // 最大缓存条目数
    .setMaxObjectSize(8192)            // 单个缓存最大字节数
    .setSharedCache(false)             // 是否是共享缓存（CDN场景）
    .build();

CloseableHttpClient cachingClient = CachingHttpClients.custom()
    .setCacheConfig(cacheConfig)
    .setCacheDir(new File("/tmp/http-cache"))
    .build();

// 使用缓存客户端
HttpGet httpGet = new HttpGet("https://api.example.com/users");

// 第一次请求：从服务端获取并缓存
try (CloseableHttpResponse response = cachingClient.execute(httpGet)) {
    System.out.println("是否来自缓存: " + isFromCache(response));
}

// 第二次请求：可能从缓存直接返回
try (CloseableHttpResponse response = cachingClient.execute(httpGet)) {
    System.out.println("是否来自缓存: " + isFromCache(response));
}

// 判断是否来自缓存
private boolean isFromCache(HttpResponse response) {
    return response.containsHeader("Age") || 
           response.getFirstHeader("Cache-Control") != null &&
           response.getFirstHeader("Cache-Control").getValue().contains("public");
}
```

## 3.7 异步HTTP客户端（5.x）

### 3.7.1 引入依赖

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.3.1</version>
</dependency>
```

### 3.7.2 基础使用

```java
// 创建异步客户端
CloseableHttpAsyncClient asyncClient = HttpAsyncClients.createDefault();
asyncClient.start();

// 异步GET请求
SimpleHttpRequest request = SimpleRequestBuilder.get()
    .setUri("https://api.example.com/users")
    .build();

asyncClient.execute(request, new FutureCallback<SimpleHttpResponse>() {
    @Override
    public void completed(SimpleHttpResponse response) {
        System.out.println("状态码: " + response.getCode());
        System.out.println("响应体: " + response.getBodyText());
    }

    @Override
    public void failed(Exception ex) {
        System.err.println("请求失败: " + ex.getMessage());
    }

    @Override
    public void cancelled() {
        System.out.println("请求已取消");
    }
});

// 或者使用CompletableFuture
CompletableFuture<SimpleHttpResponse> future = 
    asyncClient.execute(request, null);
future.thenAccept(response -> {
    System.out.println(response.getBodyText());
}).exceptionally(ex -> {
    System.err.println("Error: " + ex.getMessage());
    return null;
});
```

### 3.7.3 异步客户端配置

```java
// 配置并发的异步客户端
PoolingAsyncClientConnectionManager asyncConnManager = 
    PoolingAsyncClientConnectionManagerBuilder.create()
        .setMaxConnTotal(200)
        .setMaxConnPerRoute(50)
        .build();

CloseableHttpAsyncClient asyncClient = HttpAsyncClients.custom()
    .setConnectionManager(asyncConnManager)
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(5000, TimeUnit.MILLISECONDS)
        .setResponseTimeout(10000, TimeUnit.MILLISECONDS)
        .build())
    .build();

asyncClient.start();

// 批量并发请求
List<String> urls = Arrays.asList(
    "https://api.example.com/users",
    "https://api.example.com/posts",
    "https://api.example.com/comments"
);

List<CompletableFuture<SimpleHttpResponse>> futures = urls.stream()
    .map(url -> {
        SimpleHttpRequest req = SimpleRequestBuilder.get().setUri(url).build();
        return asyncClient.execute(req, null);
    })
    .collect(Collectors.toList());

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenAccept(v -> {
        futures.forEach(f -> {
            try {
                SimpleHttpResponse resp = f.get();
                System.out.println(resp.getCode() + ": " + resp.getBodyText().length() + " bytes");
            } catch (Exception e) {
                System.err.println("请求异常: " + e.getMessage());
            }
        });
    })
    .join();
```

# 4. 底层原理与架构

## 4.1 整体架构

Apache HttpClient的架构层次清晰，核心组件包括：

```
┌─────────────────────────────────────────────┐
│  User Layer                                  │
│  (HttpGet, HttpPost, CloseableHttpResponse) │
├─────────────────────────────────────────────┤
│  Execution Pipeline                          │
│  (HttpRequestInterceptor链 → 请求执行 →     │
│   HttpResponseInterceptor链)                │
├─────────────────────────────────────────────┤
│  HttpClient                                  │
│  (InternalHttpClient / MinimalHttpClient)   │
│  - 路由计算 (RoutePlanner)                   │
│  - 连接管理 (ConnectionManager)              │
│  - 认证处理 (CredentialsProvider)            │
│  - 重定向处理 (RedirectStrategy)             │
├─────────────────────────────────────────────┤
│  Connection Manager                          │
│  (PoolingHttpClientConnectionManager)        │
│  - 连接池管理                                │
│  - 连接生命周期管理                          │
│  - 连接复用/过期/驱逐                        │
├─────────────────────────────────────────────┤
│  Connection Layer                            │
│  (HttpClientConnection / Socket)             │
│  - TCP连接                                   │
│  - TLS/SSL握手                               │
│  - 请求/响应序列化与解析                     │
└─────────────────────────────────────────────┘
```

## 4.2 请求执行流程

```
HttpClient.execute(request)
    ↓
HttpRequestExecutor.execute(request, conn, context)
    ↓
1. 检查请求的合法性（scheme、uri等）
2. 路由计算（RoutePlanner.determineRoute()）
3. 从连接管理器获取连接（ConnectionManager.requestConnection()）
4. 执行请求拦截器链（HttpRequestInterceptor）
5. 发送HTTP请求到服务端
6. 接收HTTP响应
7. 执行响应拦截器链（HttpResponseInterceptor）
8. 处理重定向（如果启用）
9. 处理认证挑战（401/407）
10. 返回response给用户
```

## 4.3 连接管理原理

### 4.3.1 PoolingHttpClientConnectionManager

连接管理器是Apache HttpClient最核心的组件之一。

```java
PoolingHttpClientConnectionManager connManager = 
    new PoolingHttpClientConnectionManager();

// 总连接数限制
connManager.setMaxTotal(200);
// 每个路由（host:port）的最大连接数
connManager.setDefaultMaxPerRoute(20);
// 特定路由的定制（例如对重要API提高连接数上限）
HttpHost apiHost = new HttpHost("api.example.com", 443, "https");
connManager.setMaxPerRoute(new HttpRoute(apiHost), 50);
```

### 4.3.2 连接池数据结构

```java
// 核心数据结构（简化表示）
public class PoolingHttpClientConnectionManager {
    // 连接池：CPool（线程安全的连接池实现）
    private final CPool pool;
    
    // 默认连接配置
    private SocketConfig defaultSocketConfig;
    private ConnectionConfig defaultConnectionConfig;
    
    // 连接TTL（Time To Live）
    private long timeToLive;
    private TimeUnit timeToLiveUnit;
    
    // 是否自动验证连接有效性
    private boolean validateAfterInactivity;
}

// CPool内部
// public class CPool implements ConnPool<HttpRoute, ManagedHttpClientConnection> {
//     private final AtomicLong count;            // 连接计数器
//     private final LinkedList<CPoolEntry> leased; // 已租出的连接
//     private final LinkedList<CPoolEntry> available; // 可用（空闲）连接
//     private final LinkedList<Future<CPoolEntry>> pending; // 等待中的请求
// }
```

### 4.3.3 连接生命周期

```
创建连接 (ConnectionRequest) 
   ↓
借出连接 (lease)
   ├── 有空闲连接且有效 → 直接复用
   └── 无空闲连接
       ├── 未达到最大连接数 → 创建新连接
       └── 达到最大连接数 → 阻塞等待（直到有连接归还）
   ↓
使用连接 (执行请求)
   ↓
归还连接 (release)
   ├── 连接有效 → 放回空闲队列
   └── 连接无效或已关闭 → 销毁连接
```

### 4.3.4 连接配置

```java
// 4.x
PoolingHttpClientConnectionManager connManager = 
    new PoolingHttpClientConnectionManager();

// Socket配置
connManager.setDefaultSocketConfig(SocketConfig.custom()
    .setSoTimeout(10000)           // Socket读取超时
    .setSoKeepAlive(true)          // 启用TCP Keep-Alive
    .setTcpNoDelay(true)           // 禁用Nagle算法
    .build());

// 连接配置（HTTP层面）
connManager.setDefaultConnectionConfig(ConnectionConfig.custom()
    .setBufferSize(8192)           // 缓冲区大小
    .setCharset(StandardCharsets.UTF_8)
    .build());

// 设置空闲连接验证时间间隔
// 在借出连接时，如果连接空闲时间超过此值，会先验证连接有效性
connManager.setValidateAfterInactivity(2000); // 2秒
```

## 4.4 线程模型

```
同步请求线程模型：
Thread (用户线程) → HttpClient.execute() → 阻塞直到响应返回

异步请求线程模型（5.x）：
Reactor I/O Thread (NIO事件循环)
    ├── 接收请求 → 解析HTTP协议
    ├── 处理I/O事件（连接、读取、写入）
    └── 回调用户代码（completed/failed/cancelled）

连接管理线程模型：
IdleConnectionMonitorThread (用户启动的后台线程)
    └── 定期扫描连接池，关闭过期/空闲过久的连接
```

# 5. 性能优化

## 5.1 连接池调优

```java
// 4.x 连接池优化
PoolingHttpClientConnectionManager connManager = 
    new PoolingHttpClientConnectionManager();
connManager.setMaxTotal(400);                      // 总连接数
connManager.setDefaultMaxPerRoute(100);            // 每路由最大连接数

// 为高并发API单独设置
HttpHost crmApi = new HttpHost("crm.example.com", 443);
connManager.setMaxPerRoute(new HttpRoute(crmApi), 200);

// 5.x 连接池优化
PoolingHttpClientConnectionManager connManager5 = 
    PoolingHttpClientConnectionManagerBuilder.create()
        .setMaxConnTotal(400)
        .setMaxConnPerRoute(100)
        .build();
```

### 5.1.1 连接TTL设置

```java
// 设置连接最大存活时间（避免使用过时连接）
// 4.x
PoolingHttpClientConnectionManager connManager = 
    new PoolingHttpClientConnectionManager(30, TimeUnit.SECONDS);

// 5.x
PoolingHttpClientConnectionManager connManager5 = 
    PoolingHttpClientConnectionManagerBuilder.create()
        .setMaxConnPerRoute(100)
        .setMaxConnTotal(400)
        .setTimeToLive(30, TimeUnit.SECONDS)
        .build();
```

### 5.1.2 空闲连接监控与驱逐

```java
// 启动空闲连接监控线程
public class IdleConnectionMonitor {
    private final PoolingHttpClientConnectionManager connManager;
    private volatile boolean shutdown;
    
    public IdleConnectionMonitor(PoolingHttpClientConnectionManager connManager) {
        this.connManager = connManager;
    }
    
    public void start() {
        Thread monitorThread = new Thread(() -> {
            try {
                while (!shutdown) {
                    synchronized (this) {
                        wait(5000); // 每5秒检查一次
                        
                        // 关闭空闲超过30秒的连接
                        connManager.closeIdleConnections(30, TimeUnit.SECONDS);
                        // 关闭过期连接
                        connManager.closeExpiredConnections();
                        
                        log.debug("连接池状态 - 总连接: {}, 已租出: {}, 空闲: {}",
                            connManager.getTotalStats().getAvailable(), // 这里需要细分统计
                            connManager.getTotalStats().getLeased(),
                            connManager.getTotalStats().getPending());
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        monitorThread.setDaemon(true);
        monitorThread.setName("idle-connection-monitor");
        monitorThread.start();
    }
    
    public void shutdown() {
        shutdown = true;
        synchronized (this) {
            notifyAll();
        }
    }
}
```

### 5.1.3 5.x连接监控

```java
// 5.x提供了更便捷的连接监控方式
PoolingHttpClientConnectionManager connManager = 
    PoolingHttpClientConnectionManagerBuilder.create()
        .setMaxConnTotal(200)
        .setMaxConnPerRoute(50)
        .build();

// 获取连接池状态
PoolStats totalStats = connManager.getTotalStats();
System.out.printf("总连接数: %d, 已租出: %d, 空闲: %d, 等待中: %d%n",
    totalStats.getMax(), totalStats.getLeased(), 
    totalStats.getAvailable(), totalStats.getPending());
```

## 5.2 超时配置优化

### 5.2.1 三阶段超时（4.x）

```java
RequestConfig requestConfig = RequestConfig.custom()
    .setConnectTimeout(5000)             // 1. 建立TCP连接的超时时间
    .setConnectionRequestTimeout(3000)   // 2. 从连接池获取连接的超时时间
    .setSocketTimeout(10000)             // 3. Socket读取数据的超时时间（等待数据到达的最大时间）
    .build();

CloseableHttpClient client = HttpClients.custom()
    .setDefaultRequestConfig(requestConfig)
    .build();
```

### 5.2.2 5.x超时配置

```java
RequestConfig requestConfig = RequestConfig.custom()
    .setConnectTimeout(5000, TimeUnit.MILLISECONDS)       // 连接超时
    .setConnectionRequestTimeout(3000, TimeUnit.MILLISECONDS) // 连接池获取超时
    .setResponseTimeout(10000, TimeUnit.MILLISECONDS)     // 响应超时（替代SocketTimeout）
    .build();
```

## 5.3 复用CloseableHttpClient

与OkHttp类似，必须复用HttpClient实例。

```java
// ❌ 错误：每次创建新实例
public void badPractice() throws IOException {
    CloseableHttpClient client = HttpClients.createDefault();
    client.execute(new HttpGet("https://api.example.com"));
    client.close(); // 频繁创建和关闭时，连接池反复创建销毁
}

// ✅ 正确：复用单例
@Component
public class HttpClientPool {
    private final CloseableHttpClient httpClient;
    
    public HttpClientPool() {
        PoolingHttpClientConnectionManager connManager = 
            new PoolingHttpClientConnectionManager();
        connManager.setMaxTotal(200);
        connManager.setDefaultMaxPerRoute(50);
        
        this.httpClient = HttpClients.custom()
            .setConnectionManager(connManager)
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(5000)
                .setSocketTimeout(10000)
                .setConnectionRequestTimeout(3000)
                .build())
            .build();
    }
    
    public CloseableHttpClient getClient() {
        return httpClient;
    }
    
    @PreDestroy
    public void destroy() {
        try {
            httpClient.close();
        } catch (IOException e) {
            log.error("关闭HttpClient异常", e);
        }
    }
}
```

## 5.4 响应体释放与连接回收

### 5.4.1 推荐方式：EntityUtils.consume()

```java
// 确保Entity被完全消费，才能正确回收连接
CloseableHttpResponse response = client.execute(httpGet);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        String content = EntityUtils.toString(entity, StandardCharsets.UTF_8);
        processContent(content);
    }
    // EntityUtils.toString()内部已调用EntityUtils.consume()消费entity
} finally {
    // 关闭响应，释放连接
    response.close();
}

// 如果不需要读取实体内容，直接消费
response.close(); // 会自动调用EntityUtils.consume()
```

### 5.4.2 流式处理必须完全消费

```java
// 流式处理大文件时必须读取完所有内容
CloseableHttpResponse response = client.execute(httpGet);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        try (InputStream instream = entity.getContent()) {
            byte[] buffer = new byte[4096];
            int bytesRead;
            while ((bytesRead = instream.read(buffer)) != -1) {
                // 处理数据块
                processChunk(buffer, bytesRead);
            }
            // 必须读到底（InputStream返回-1），连接才会被回收
        }
    }
} finally {
    response.close();
}
```

## 5.5 使用缓冲实体提高性能

```java
// 使用BufferedHttpEntity包装，将响应体缓存到内存中
// 可以多次读取响应体
HttpEntity entity = response.getEntity();
BufferedHttpEntity bufferedEntity = new BufferedHttpEntity(entity);

// 可以多次调用toString
String content1 = EntityUtils.toString(bufferedEntity);
String content2 = EntityUtils.toString(bufferedEntity); // 不会报错
```

## 5.6 HTTP/2优化（5.x）

```java
// 启用HTTP/2
CloseableHttpClient http2Client = HttpClients.custom()
    .setConnectionManager(PoolingHttpClientConnectionManagerBuilder.create()
        .setMaxConnTotal(200)
        .setMaxConnPerRoute(50)
        .build())
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(5000, TimeUnit.MILLISECONDS)
        .setResponseTimeout(10000, TimeUnit.MILLISECONDS)
        .build())
    .setVersionPolicy(HttpVersionPolicy.NEGOTIATE) // 自动协商HTTP/2
    .build();
```

## 5.7 合理使用Keep-Alive

```java
// 服务端可能会在响应中告知Keep-Alive时间
// Connection: keep-alive
// Keep-Alive: timeout=60, max=500

// 客户端应设置合理的连接TTL，避免使用已关闭的连接
// 同时建议启用空闲连接监控，主动清理无效连接
```

# 6. 与Spring Boot集成

## 6.1 RestTemplate集成

### 6.1.1 配置依赖

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 6.1.2 通过ClientHttpRequestFactory集成

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public PoolingHttpClientConnectionManager poolingConnectionManager() {
        PoolingHttpClientConnectionManager connManager = 
            new PoolingHttpClientConnectionManager();
        connManager.setMaxTotal(200);
        connManager.setDefaultMaxPerRoute(50);
        
        // Socket配置
        connManager.setDefaultSocketConfig(SocketConfig.custom()
            .setSoTimeout(10000)
            .setSoKeepAlive(true)
            .setTcpNoDelay(true)
            .build());
        
        return connManager;
    }

    @Bean
    public CloseableHttpClient httpClient(PoolingHttpClientConnectionManager connManager) {
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectTimeout(5000)
            .setConnectionRequestTimeout(3000)
            .setSocketTimeout(10000)
            .build();
        
        return HttpClients.custom()
            .setConnectionManager(connManager)
            .setDefaultRequestConfig(requestConfig)
            .setRetryHandler(new DefaultHttpRequestRetryHandler(0, false)) // 禁止默认重试
            .build();
    }

    @Bean
    public RestTemplate restTemplate(CloseableHttpClient httpClient) {
        // 使用HttpComponentsClientHttpRequestFactory桥接
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        factory.setHttpClient(httpClient);
        
        RestTemplate restTemplate = new RestTemplate(factory);
        
        // 添加自定义MessageConverter
        restTemplate.getMessageConverters().add(0, 
            new StringHttpMessageConverter(StandardCharsets.UTF_8));
        
        return restTemplate;
    }
}
```

### 6.1.3 RestTemplate使用示例

```java
@Service
public class ApiService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    // GET请求——返回对象
    public User getUser(Long id) {
        return restTemplate.getForObject(
            "https://api.example.com/users/{id}", 
            User.class, id);
    }
    
    // GET请求——返回ResponseEntity
    public ResponseEntity<List<User>> getUsers() {
        ParameterizedTypeReference<List<User>> typeRef = 
            new ParameterizedTypeReference<List<User>>() {};
        
        return restTemplate.exchange(
            "https://api.example.com/users",
            HttpMethod.GET,
            null,
            typeRef);
    }
    
    // POST请求
    public User createUser(User user) {
        return restTemplate.postForObject(
            "https://api.example.com/users", 
            user, 
            User.class);
    }
    
    // PUT请求
    public void updateUser(Long id, User user) {
        restTemplate.put("https://api.example.com/users/{id}", user, id);
    }
    
    // DELETE请求
    public void deleteUser(Long id) {
        restTemplate.delete("https://api.example.com/users/{id}", id);
    }
    
    // 自定义请求头
    public String getWithAuth() {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth("access-token");
        headers.set("X-Custom-Header", "value");
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        ResponseEntity<String> response = restTemplate.exchange(
            "https://api.example.com/protected",
            HttpMethod.GET,
            entity,
            String.class);
        
        return response.getBody();
    }
    
    // POST JSON + 自定义请求头
    public User createWithAuth(User user) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth("access-token");
        
        HttpEntity<User> entity = new HttpEntity<>(user, headers);
        
        ResponseEntity<User> response = restTemplate.postForEntity(
            "https://api.example.com/users", 
            entity, 
            User.class);
        
        return response.getBody();
    }
    
    // 错误处理
    public String getWithErrorHandling() {
        try {
            return restTemplate.getForObject(
                "https://api.example.com/users", String.class);
        } catch (HttpClientErrorException e) {
            // 4xx错误
            log.error("客户端错误: {} {}", e.getStatusCode(), e.getResponseBodyAsString());
            throw new BusinessException("请求参数错误");
        } catch (HttpServerErrorException e) {
            // 5xx错误
            log.error("服务端错误: {}", e.getStatusCode());
            throw new BusinessException("服务暂不可用");
        } catch (ResourceAccessException e) {
            // 连接超时等IO异常
            log.error("网络异常: {}", e.getMessage());
            throw new BusinessException("网络异常");
        }
    }
}
```

### 6.1.4 5.x版本集成RestTemplate

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.3.1</version>
</dependency>
```

```java
@Configuration
public class RestTemplateConfig5x {

    @Bean
    public PoolingHttpClientConnectionManager poolingConnectionManager() {
        return PoolingHttpClientConnectionManagerBuilder.create()
            .setMaxConnTotal(200)
            .setMaxConnPerRoute(50)
            .setDefaultSocketConfig(SocketConfig.custom()
                .setSoTimeout(10000, TimeUnit.MILLISECONDS)
                .setSoKeepAlive(true)
                .setTcpNoDelay(true)
                .build())
            .build();
    }

    @Bean
    public CloseableHttpClient httpClient(PoolingHttpClientConnectionManager connManager) {
        return HttpClientBuilder.create()
            .setConnectionManager(connManager)
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(5000, TimeUnit.MILLISECONDS)
                .setConnectionRequestTimeout(3000, TimeUnit.MILLISECONDS)
                .setResponseTimeout(10000, TimeUnit.MILLISECONDS)
                .build())
            .build();
    }

    @Bean
    public RestTemplate restTemplate(CloseableHttpClient httpClient) {
        // Spring 6.x / Spring Boot 3.x 使用 HttpComponentsClientHttpRequestFactory
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        factory.setHttpClient(httpClient);
        
        return new RestTemplate(factory);
    }
}
```

## 6.2 集成Spring Cloud OpenFeign

```java
@Configuration
public class FeignConfig {

    @Bean
    public Client feignClient() {
        // 使用Apache HttpClient作为Feign的底层客户端
        return new ApacheHttpClient(
            HttpClients.custom()
                .setConnectionManager(new PoolingHttpClientConnectionManager())
                .setDefaultRequestConfig(RequestConfig.custom()
                    .setConnectTimeout(5000)
                    .setSocketTimeout(10000)
                    .build())
                .build());
    }
}

// Feign客户端接口
@FeignClient(name = "user-service", url = "https://api.example.com", 
             configuration = FeignConfig.class)
public interface UserFeignClient {
    
    @GetMapping("/users/{id}")
    User getUser(@PathVariable("id") Long id);
    
    @PostMapping("/users")
    User createUser(@RequestBody User user);
    
    @GetMapping("/users")
    List<User> getUsers();
}
```

## 6.3 集成Spring Boot Actuator监控

```java
@Configuration
public class HttpClientMetricsConfig {
    
    @Bean
    public ConnectionPoolMetrics connectionPoolMetrics(
            PoolingHttpClientConnectionManager connManager) {
        return new ConnectionPoolMetrics(connManager);
    }
}

// 自定义Metrics收集
public class ConnectionPoolMetrics {
    private final PoolingHttpClientConnectionManager connManager;
    private final MeterRegistry meterRegistry;
    
    public ConnectionPoolMetrics(PoolingHttpClientConnectionManager connManager,
                                  MeterRegistry meterRegistry) {
        this.connManager = connManager;
        this.meterRegistry = meterRegistry;
        
        // 注册Gauge指标
        Gauge.builder("httpclient.connections.total", connManager,
                cm -> cm.getTotalStats().getMax())
            .description("连接池总连接数")
            .register(meterRegistry);
        
        Gauge.builder("httpclient.connections.leased", connManager,
                cm -> cm.getTotalStats().getLeased())
            .description("已租出连接数")
            .register(meterRegistry);
        
        Gauge.builder("httpclient.connections.available", connManager,
                cm -> cm.getTotalStats().getAvailable())
            .description("空闲连接数")
            .register(meterRegistry);
        
        Gauge.builder("httpclient.connections.pending", connManager,
                cm -> cm.getTotalStats().getPending())
            .description("等待中的请求数")
            .register(meterRegistry);
    }
}
```

## 6.4 使用Spring RestClient（Spring 6.1+）

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient restClient(CloseableHttpClient httpClient) {
        return RestClient.builder()
            .baseUrl("https://api.example.com")
            .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
public class UserService {
    @Autowired
    private RestClient restClient;
    
    public List<User> getUsers() {
        return restClient.get()
            .uri("/users")
            .retrieve()
            .body(new ParameterizedTypeReference<List<User>>() {});
    }
    
    public User getUser(Long id) {
        return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new BusinessException("用户不存在");
            })
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new BusinessException("服务异常");
            })
            .body(User.class);
    }
    
    public User createUser(User user) {
        return restClient.post()
            .uri("/users")
            .body(user)
            .retrieve()
            .body(User.class);
    }
}
```

# 7. 最佳实践与常见问题

## 7.1 最佳实践

### 7.1.1 HttpClient单例复用

```java
// ✅ 正确：全局单例
@Component
public class HttpClientManager {
    private static CloseableHttpClient httpClient;
    
    static {
        PoolingHttpClientConnectionManager connManager = 
            new PoolingHttpClientConnectionManager();
        connManager.setMaxTotal(200);
        connManager.setDefaultMaxPerRoute(50);
        
        httpClient = HttpClients.custom()
            .setConnectionManager(connManager)
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(5000)
                .setSocketTimeout(10000)
                .setConnectionRequestTimeout(3000)
                .build())
            .build();
    }
    
    public static CloseableHttpClient getClient() {
        return httpClient;
    }
}

// ❌ 错误：每次创建新Client
public void wrongWay() throws IOException {
    try (CloseableHttpClient client = HttpClients.createDefault()) {
        client.execute(new HttpGet("...")); // 每次创建新连接池
    }
}
```

### 7.1.2 正确释放连接

```java
// ✅ 推荐使用ResponseHandler自动管理连接
String result = httpClient.execute(httpGet, response -> {
    return EntityUtils.toString(response.getEntity());
});

// ✅ 手动管理时必须确保finally关闭
CloseableHttpResponse response = null;
try {
    response = httpClient.execute(httpGet);
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        EntityUtils.consume(entity); // 确保完全消费
    }
} finally {
    if (response != null) {
        response.close();
    }
}

// ✅ try-with-resources（5.x）
try (CloseableHttpResponse response = httpClient.execute(httpGet)) {
    String body = EntityUtils.toString(response.getEntity());
}
```

### 7.1.3 EntityUtils的正确使用

```java
// EntityUtils.toString()会同时消费entity并关闭内部流
// 但注意：对大响应体要先确认ContentLength

// 对大文件的处理应使用流式方式，避免OOM
CloseableHttpResponse response = httpClient.execute(httpGet);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        long contentLength = entity.getContentLength();
        if (contentLength > 100 * 1024 * 1024) { // 超过100MB
            // 流式写入文件
            try (InputStream in = entity.getContent();
                 FileOutputStream out = new FileOutputStream("large-file.zip")) {
                byte[] buffer = new byte[8192];
                int len;
                while ((len = in.read(buffer)) != -1) {
                    out.write(buffer, 0, len);
                }
            }
        } else {
            // 内存加载
            String content = EntityUtils.toString(entity);
        }
    }
} finally {
    response.close();
}
```

### 7.1.4 配置合理的超时时间

```java
// 不同场景使用不同的超时配置
// 内部服务：短超时，快速失败后降级
RequestConfig internalConfig = RequestConfig.custom()
    .setConnectTimeout(1000)
    .setConnectionRequestTimeout(500)
    .setSocketTimeout(3000)
    .build();

// 外部服务：适中超时
RequestConfig externalConfig = RequestConfig.custom()
    .setConnectTimeout(5000)
    .setConnectionRequestTimeout(3000)
    .setSocketTimeout(10000)
    .build();

// 文件上传：长写入超时
RequestConfig uploadConfig = RequestConfig.custom()
    .setConnectTimeout(10000)
    .setConnectionRequestTimeout(5000)
    .setSocketTimeout(60000)
    .build();
```

### 7.1.5 使用连接池监控

```java
// 定期打印连接池状态
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(() -> {
    PoolStats stats = connManager.getTotalStats();
    log.info("HttpClient连接池 - 总容量: {}, 已租出: {}, 空闲: {}, 等待: {}",
        stats.getMax(), stats.getLeased(), 
        stats.getAvailable(), stats.getPending());
    
    // 针对每个路由的统计
    Set<HttpRoute> routes = connManager.getRoutes();
    for (HttpRoute route : routes) {
        PoolStats routeStats = connManager.getStats(route);
        log.debug("路由 {} - 容量: {}, 已租出: {}, 空闲: {}",
            route, routeStats.getMax(), 
            routeStats.getLeased(), routeStats.getAvailable());
    }
}, 10, 10, TimeUnit.SECONDS);
```

### 7.1.6 使用简化的Fluent API（5.x）

```java
// 对于简单的请求，使用Fluent API减少样板代码
import org.apache.hc.client5.http.fluent.Request;

// 简单GET
String content = Request.get("https://api.example.com/users")
    .connectTimeout(5000)
    .responseTimeout(10000)
    .addHeader("Authorization", "Bearer token")
    .execute()
    .returnContent()
    .asString();

// 简单POST
String result = Request.post("https://api.example.com/users")
    .bodyString("{\"name\":\"张三\"}", ContentType.APPLICATION_JSON)
    .execute()
    .returnContent()
    .asString();
```

## 7.2 常见问题与解决方案

### 7.2.1 连接池耗尽

**现象**：`ConnectionPoolTimeoutException: Timeout waiting for connection from pool`

**原因**：
1. 连接池太小，无法支撑并发量
2. 响应体未正确消费，连接未归还
3. 连接泄漏

**解决方案**：

```java
// 1. 增大连接池
connManager.setMaxTotal(500);
connManager.setDefaultMaxPerRoute(100);

// 2. 确保实体完全消费
EntityUtils.consume(entity); // 或 EntityUtils.toString(entity)

// 3. 使用ResponseHandler（自动管理）
String result = client.execute(request, response -> 
    EntityUtils.toString(response.getEntity()));

// 4. 设置连接池获取超时，避免无限等待
RequestConfig config = RequestConfig.custom()
    .setConnectionRequestTimeout(5000) // 从连接池获取连接最多等5秒
    .build();
```

### 7.2.2 Socket超时

**现象**：`SocketTimeoutException: Read timed out`

**原因**：
1. 服务端处理过慢
2. 网络带宽不足
3. 响应数据量过大

**解决方案**：

```java
// 1. 适当增大超时
RequestConfig config = RequestConfig.custom()
    .setSocketTimeout(30000) // 增大到30秒
    .build();

// 2. 针对慢接口单独使用长超时
// 3. 考虑使用异步客户端避免线程阻塞
```

### 7.2.3 连接池泄漏

**现象**：连接池连接数持续增长，不释放

**原因**：
1. 响应体未消费（最常见原因）
2. Entity的InputStream未关闭
3. 异常分支未关闭响应

**解决方案**：

```java
// ✅ 确保所有分支都关闭
CloseableHttpResponse response = null;
try {
    response = client.execute(request);
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        // 消费entity
        EntityUtils.consume(entity);
    }
} catch (IOException e) {
    log.error("请求异常", e);
    throw e;
} finally {
    if (response != null) {
        try {
            response.close();
        } catch (IOException e) {
            log.error("关闭响应异常", e);
        }
    }
}

// ✅ 或使用ResponseHandler（最安全）
```

### 7.2.4 SSL/TLS握手失败

**现象**：`SSLHandshakeException` 或 `javax.net.ssl.SSLException`

**原因**：
1. 证书过期或无效
2. 服务器使用不支持的TLS版本
3. 证书链不完整

**解决方案**：

```java
// 配置TLS版本
SSLContext sslContext = SSLContexts.custom()
    .loadTrustMaterial(trustStore, null)
    .build();

CloseableHttpClient client = HttpClients.custom()
    .setSSLContext(sslContext)
    .setSSLHostnameVerifier(NoopHostnameVerifier.INSTANCE) // 跳过主机名验证（不推荐生产）
    .build();

// 5.x TLS配置
TlsConfig tlsConfig = TlsConfig.custom()
    .setSupportedProtocols(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
    .build();
```

### 7.2.5 重试导致非幂等请求重复执行

**问题**：POST/PATCH等非幂等请求在重试时可能导致重复提交

**解决方案**：

```java
// 禁止重试非幂等请求
HttpRequestRetryHandler noRetryForPost = (exception, executionCount, context) -> {
    if (executionCount >= 3) {
        return false;
    }
    
    // 仅对GET等幂等方法重试
    HttpClientContext clientContext = HttpClientContext.adapt(context);
    HttpRequest request = clientContext.getRequest();
    if (request instanceof HttpEntityEnclosingRequest) {
        // POST/PATCH等包含请求体的方法不重试
        return false;
    }
    
    return true;
};

// 或使用自带StandardHttpRequestRetryHandler（默认只重试幂等方法）
HttpRequestRetryHandler retryHandler = new StandardHttpRequestRetryHandler(3, false);
```

### 7.2.6 大量TIME_WAIT连接

**现象**：系统产生大量`TIME_WAIT`状态的TCP连接

**原因**：
1. 短连接模式下频繁创建和关闭连接
2. Keep-Alive未正确启用

**解决方案**：

```java
// 1. 正确使用连接池（重用连接）
// 2. 启用TCP Keep-Alive
SocketConfig socketConfig = SocketConfig.custom()
    .setSoKeepAlive(true)
    .build();

// 3. 设置合理的连接TTL，避免连接过快过期
PoolingHttpClientConnectionManager connManager = 
    new PoolingHttpClientConnectionManager(60, TimeUnit.SECONDS);
```

# 8. 与其他HTTP客户端对比

## 8.1 Apache HttpClient vs OkHttp

| 对比维度 | Apache HttpClient | OkHttp |
|---------|------------------|--------|
| **诞生时间** | 2001年（最早的Java HTTP客户端之一） | 2013年 |
| **包体积** | 较大（约2MB+） | 较小（约700KB） |
| **API设计** | 经典风格，配置较为繁琐 | 链式Builder，简洁优雅 |
| **HTTP/2** | 4.x需额外依赖，5.x原生支持 | 4.x即原生支持 |
| **WebSocket** | 5.x开始原生支持 | 原生支持 |
| **连接池** | 强大的连接池，配置丰富 | 自动管理，配置简单 |
| **拦截器链** | Interceptor模式（请求/响应分离） | 责任链模式，更灵活 |
| **重试机制** | 完善的重试策略配置 | 内置自动重试（有限） |
| **Cookie管理** | 丰富的Cookie策略 | 需自行实现CookieJar |
| **认证方案** | 多种认证方案（Basic/Digest/NTLM） | 需通过Interceptor实现 |
| **流式API** | 5.x提供Fluent API | 原生链式调用 |
| **Android** | 不推荐（包大） | 官方推荐 |
| **社区活跃度** | 较慢更新，足够成熟稳定 | 活跃更新 |
| **Spring Boot集成** | RestTemplate原生支持（HttpComponents） | 需手动配置 |

**选型建议**：
- **企业遗留系统**：选择Apache HttpClient（兼容性好，文档丰富）
- **新项目/微服务**：选择OkHttp（更现代，性能更好）
- **Android项目**：必须选择OkHttp（官方推荐）
- **需要强认证方案**：选择Apache HttpClient（NTLM等支持更完善）
- **追求简洁API**：选择OkHttp

## 8.2 Apache HttpClient vs Java 11 HttpClient

| 对比维度 | Apache HttpClient | Java 11 HttpClient |
|---------|------------------|-------------------|
| JDK版本 | 无要求 | JDK 11+ |
| 额外依赖 | 需要引入 | 内置 |
| HTTP/2 | 5.x支持 | 原生支持 |
| 同步/异步 | 都支持 | 都支持（CompletableFuture） |
| 连接池 | 强大 | 内置但配置有限 |
| 拦截器 | 支持 | 支持 |
| API风格 | 经典/繁琐 | 较现代（Builder模式） |
| 社区资源 | 丰富 | 较新，资源较少 |
| 第三方集成 | 广泛 | 有限 |

## 8.3 Apache HttpClient vs RestTemplate

RestTemplate基于模板方法模式，是更上层的封装，底层可使用Apache HttpClient。

关系：`RestTemplate → HttpComponentsClientHttpRequestFactory → Apache HttpClient`

| 对比维度 | Apache HttpClient | RestTemplate |
|---------|------------------|-------------|
| 定位 | 底层HTTP客户端 | 上层HTTP访问模板 |
| 抽象层次 | 协议层 | 应用层 |
| 序列化 | 手动处理 | MessageConverter自动转换 |
| 异常体系 | IOException | RestClientException |
| 代码量 | 需要更多代码 | 简洁 |
| 灵活性 | 高（可精细控制每个环节） | 中（封装常见操作） |

# 9. 面试要点

## 9.1 基础知识

1. **Apache HttpClient的核心组件有哪些？**
   - CloseableHttpClient：HTTP客户端入口
   - HttpGet/HttpPost/HttpPut/HttpDelete：请求对象
   - CloseableHttpResponse：响应对象
   - HttpEntity：请求/响应体
   - RequestConfig：请求配置（超时、重定向等）
   - PoolingHttpClientConnectionManager：连接池管理器
   - HttpRequestInterceptor/HttpResponseInterceptor：拦截器

2. **Apache HttpClient的发展历程是怎样的？**
   - Commons HttpClient 1.x/2.x/3.x → HttpComponents HttpClient 4.x → 5.x
   - 4.x是Java企业中最广泛使用的版本
   - 5.x是重大升级，支持HTTP/2，引入异步客户端

3. **什么是连接池？为什么使用连接池？**
   - 连接池维护一组TCP连接的复用
   - 减少TCP三次握手和TLS握手的开销
   - 控制并发连接数，避免资源耗尽
   - 提高请求响应速度

## 9.2 高级面试题

4. **PoolingHttpClientConnectionManager的工作原理？**
   - 内部维护CPool连接池，包含leased（已租出）、available（空闲）、pending（等待队列）
   - 请求连接时优先从available队列获取复用连接
   - 无可用连接且未达上限时创建新连接
   - 达到上限时请求进入pending队列等待
   - 连接归还时放回available队列，并通知pending中的请求
   - 后台线程定期清理过期/超时空闲连接

5. **Apache HttpClient的超时时间有哪些？分别含义是什么？**
   - ConnectionTimeout：建立TCP连接的超时时间
   - SocketTimeout（ReadTimeout）：等待服务端返回数据的超时时间
   - ConnectionRequestTimeout：从连接池获取连接的超时时间
   - 这三个超时分别对应网络通信的不同阶段

6. **如何确保连接正确释放？**
   - 使用ResponseHandler自动管理（推荐）
   - 使用try-with-resources或try-catch-finally确保response.close()
   - 使用EntityUtils.consume()消费实体
   - 流式读取时必须读到底（InputStream返回-1）

7. **Apache HttpClient支持哪些认证方式？**
   - Basic认证（base64编码用户名密码）
   - Digest认证（MD5摘要挑战）
   - NTLM认证（Windows集成认证）
   - Bearer Token（通过拦截器或请求头实现）
   - 自定义CredentialsProvider/AuthScheme

8. **HttpRequestRetryHandler的重试策略如何配置？**
   - StandardHttpRequestRetryHandler：默认重试3次，仅重试幂等方法
   - 自定义：根据异常类型、执行次数、上下文决定是否重试
   - 重试可能导致非幂等请求重复执行，需谨慎
   - 可设置指数退避避免重试风暴

9. **Apache HttpClient 4.x和5.x的主要区别？**
   - 5.x支持HTTP/2（4.x需要额外依赖）
   - 5.x引入Fluent API和Async Client
   - 5.x包名从org.apache.http改为org.apache.hc
   - 5.x简化了API设计，移除了部分废弃API
   - 5.x的连接管理器和请求配置有所不同

10. **Apache HttpClient与Spring RestTemplate如何集成？**
    - 通过HttpComponentsClientHttpRequestFactory桥接
    - RestTemplate接受ClientHttpRequestFactory作为底层HTTP引擎
    - 可配置连接池、超时、拦截器后传给RestTemplate
    - Spring 6.1+引入RestClient替代RestTemplate
