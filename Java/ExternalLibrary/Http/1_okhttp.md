# 1. 概述与简介

## 1.1 OkHttp简介

OkHttp是Square公司开发的一款高效的Java/Kotlin HTTP客户端，广泛应用于Android和Java后端服务中。它是目前Java生态中最流行的HTTP客户端之一，被Retrofit等知名框架作为底层网络通信组件。

OkHttp的设计目标包括：
- 高传输效率：支持HTTP/2和连接复用，减少网络延迟
- 接口易用：链式调用API设计，同步/异步请求模式清晰
- 可扩展性强：Interceptor机制允许深度定制请求/响应处理流程
- 协议兼容：支持HTTP/1.x、HTTP/2、WebSocket等多种协议

## 1.2 核心特性

| 特性 | 说明 |
|------|------|
| HTTP/2支持 | 支持HTTP/2多路复用，同一主机共享连接 |
| 连接池 | 自动管理HTTP/1.1和HTTP/2连接复用 |
| Interceptor链 | 基于责任链模式的请求/响应拦截器 |
| 请求/响应体 | 支持多种Body类型（Form、Multipart、JSON等） |
| 超时控制 | 连接超时、读取超时、写入超时可独立配置 |
| 缓存机制 | 内置HTTP缓存，支持Cache-Control等缓存策略 |
| 异步回调 | 支持Callback方式的异步请求 |
| WebSocket | 原生WebSocket支持 |
| Gzip压缩 | 自动解压Gzip响应体 |
| 重定向跟踪 | 自动跟踪重定向请求 |
| 重试机制 | 支持失败重试（默认仅针对路由问题） |

# 2. 核心API使用

## 2.1 引入依赖

Maven：
```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
```

Gradle：
```groovy
implementation 'com.squareup.okhttp3:okhttp:4.12.0'
```

> 注意：OkHttp 4.x是基于Kotlin重写的版本，完全兼容Java调用。

## 2.2 OkHttpClient配置

OkHttpClient是OkHttp的核心入口，所有请求共享同一个OkHttpClient实例以提高连接复用效率。

```java
OkHttpClient client = new OkHttpClient.Builder()
    // 连接超时：建立TCP连接的最大等待时间
    .connectTimeout(10, TimeUnit.SECONDS)
    // 读取超时：接收响应的最大等待时间
    .readTimeout(30, TimeUnit.SECONDS)
    // 写入超时：发送请求体的最大等待时间
    .writeTimeout(30, TimeUnit.SECONDS)
    // 连接池配置
    .connectionPool(new ConnectionPool(5, 5, TimeUnit.MINUTES))
    // 重定向跟踪
    .followRedirects(true)
    .followSslRedirects(true)
    // 缓存目录
    .cache(new Cache(new File("http-cache"), 10 * 1024 * 1024))
    // 失败重试
    .retryOnConnectionFailure(true)
    .build();
```

### 2.2.1 全局超时配置

```java
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(5, TimeUnit.SECONDS)     // 连接建立超时
    .readTimeout(10, TimeUnit.SECONDS)       // 读取响应超时
    .writeTimeout(10, TimeUnit.SECONDS)      // 写入请求超时
    .build();
```

### 2.2.2 自定义Dispatcher线程池

```java
Dispatcher dispatcher = new Dispatcher();
dispatcher.setMaxRequests(64);                    // 最大并发请求数
dispatcher.setMaxRequestsPerHost(5);              // 每个主机的最大并发数
dispatcher.setExecutorService(new ThreadPoolExecutor(
    0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
    new SynchronousQueue<>(), 
    new ThreadFactoryBuilder().setNameFormat("okhttp-dispatcher-%d").build()
));

OkHttpClient client = new OkHttpClient.Builder()
    .dispatcher(dispatcher)
    .build();
```

## 2.3 Request与Response

### 2.3.1 创建Request

```java
// GET请求
Request getRequest = new Request.Builder()
    .url("https://api.example.com/users")
    .header("Authorization", "Bearer token123")
    .header("Accept", "application/json")
    .build();

// POST JSON请求
MediaType JSON = MediaType.parse("application/json; charset=utf-8");
String json = "{\"name\":\"张三\",\"email\":\"zhangsan@example.com\"}";
Request postRequest = new Request.Builder()
    .url("https://api.example.com/users")
    .post(RequestBody.create(json, JSON))
    .build();

// POST Form表单请求
Request formRequest = new Request.Builder()
    .url("https://api.example.com/login")
    .post(new FormBody.Builder()
        .add("username", "admin")
        .add("password", "123456")
        .build())
    .build();

// POST Multipart文件上传
Request multipartRequest = new Request.Builder()
    .url("https://api.example.com/upload")
    .post(new MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("file", "photo.jpg",
            RequestBody.create(new File("/path/to/photo.jpg"),
                MediaType.parse("image/jpeg")))
        .addFormDataPart("description", "旅游照片")
        .build())
    .build();

// DELETE请求
Request deleteRequest = new Request.Builder()
    .url("https://api.example.com/users/123")
    .delete()
    .build();
```

### 2.3.2 发送同步请求

```java
// 同步请求需在子线程中执行，否则会阻塞UI线程（Android环境）
new Thread(() -> {
    try {
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
            .url("https://api.example.com/users")
            .build();

        Response response = client.newCall(request).execute();
        
        // 注意：Response body只能读取一次
        if (response.isSuccessful()) {
            String responseBody = response.body().string();
            System.out.println("Response: " + responseBody);
        } else {
            System.err.println("Request failed: " + response.code());
        }
        
        // 关闭响应释放资源（try-with-resources自动关闭）
        response.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}).start();
```

### 2.3.3 发送异步请求

```java
OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
    .url("https://api.example.com/users")
    .build();

client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        // 请求失败回调（网络异常、超时等）
        System.err.println("Request failed: " + e.getMessage());
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        // 请求成功回调（注意：此回调在子线程中执行）
        try (ResponseBody body = response.body()) {
            if (response.isSuccessful()) {
                String responseBody = body.string();
                System.out.println("Response: " + responseBody);
            }
        }
    }
});
```

### 2.3.4 请求取消

```java
Call call = client.newCall(request);
call.enqueue(callback);

// 在另一个线程或用户操作中取消请求
call.cancel();

// 判断请求是否已取消
if (call.isCanceled()) {
    System.out.println("请求已取消");
}
```

## 2.4 同步与异步调用对比

| 特性 | 同步execute() | 异步enqueue() |
|------|---------------|---------------|
| 线程阻塞 | 阻塞当前线程 | 不阻塞（内部线程池执行） |
| 返回值 | 直接返回Response | Callback回调 |
| 关闭响应 | 必须手动关闭 | 建议在onResponse中关闭 |
| 适用场景 | 非UI线程、批量操作 | UI线程、Web层 |
| 异常处理 | try-catch捕获 | onFailure接收 |

# 3. 高级特性

## 3.1 Interceptor拦截器

Interceptor是OkHttp最强大的扩展机制，基于责任链（Chain of Responsibility）设计模式，允许在请求发送前和响应返回后进行拦截处理。

### 3.1.1 Application Interceptor vs Network Interceptor

```java
OkHttpClient client = new OkHttpClient.Builder()
    // Application Interceptor —— 应用层拦截器
    // 只会被调用一次（即使发生重定向或重试）
    .addInterceptor(new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            
            // 预处理：添加公共请求头
            Request newRequest = request.newBuilder()
                .header("X-Request-ID", UUID.randomUUID().toString())
                .header("User-Agent", "MyApp/1.0")
                .build();
            
            long startTime = System.currentTimeMillis();
            Response response = chain.proceed(newRequest);
            long endTime = System.currentTimeMillis();
            
            // 后处理：日志记录
            System.out.printf("请求 %s %s 耗时 %dms，状态码 %d%n",
                newRequest.method(), newRequest.url(), 
                endTime - startTime, response.code());
            
            return response;
        }
    })
    // Network Interceptor —— 网络层拦截器
    // 每次网络请求都会被调用（包括重定向和重试）
    .addNetworkInterceptor(new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            System.out.println("Network Interceptor - 发起网络请求: " + request.url());
            
            Response response = chain.proceed(request);
            System.out.println("Network Interceptor - 收到网络响应: " + response.code());
            
            return response;
        }
    })
    .build();
```

**Application Interceptor vs Network Interceptor 核心区别：**

| 对比维度 | Application Interceptor | Network Interceptor |
|---------|------------------------|--------------------|
| 调用时机 | 在请求组装完成后立即调用 | 在真正发起网络请求前调用 |
| 重定向 | 只调用一次（看不到重定向） | 每次重定向都会调用 |
| 重试 | 只调用一次 | 每次重试都会调用 |
| 缓存 | 可以看到缓存的响应（如果有） | 看不到缓存（只处理真实网络请求） |
| 短路操作 | 可以拦截请求不发送（返回Mock响应） | 不能短路 |

### 3.1.2 典型Interceptor实现

**日志拦截器：**
```java
public class LoggingInterceptor implements Interceptor {
    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        
        log.info("→ {} {}", request.method(), request.url());
        
        // 打印请求头
        Headers headers = request.headers();
        for (String name : headers.names()) {
            log.debug("  Header: {}: {}", name, headers.get(name));
        }
        
        // 读取请求体（注意需保留原始内容）
        RequestBody requestBody = request.body();
        if (requestBody != null) {
            Buffer buffer = new Buffer();
            requestBody.writeTo(buffer);
            log.debug("  Body: {}", buffer.readUtf8());
        }
        
        long startNs = System.nanoTime();
        Response response;
        try {
            response = chain.proceed(request);
        } catch (IOException e) {
            log.error("请求异常: {}", e.getMessage());
            throw e;
        }
        long elapsedMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startNs);
        
        log.info("← {} {} ({}ms)", response.code(), request.url(), elapsedMs);
        
        return response;
    }
}
```

**重试拦截器：**
```java
public class RetryInterceptor implements Interceptor {
    private final int maxRetryCount;
    
    public RetryInterceptor(int maxRetryCount) {
        this.maxRetryCount = maxRetryCount;
    }
    
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = null;
        IOException lastException = null;
        
        for (int retryCount = 0; retryCount < maxRetryCount; retryCount++) {
            try {
                response = chain.proceed(request);
                // 如果响应成功或非5xx错误则直接返回
                if (response.isSuccessful() || response.code() < 500) {
                    return response;
                }
                // 5xx错误时关闭响应继续重试
                response.close();
            } catch (IOException e) {
                lastException = e;
                // 连接异常时继续重试
                if (!shouldRetry(e)) {
                    throw e;
                }
            }
            
            // 指数退避等待
            try {
                long waitMs = (long) Math.pow(2, retryCount) * 1000;
                Thread.sleep(waitMs);
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new IOException("重试被中断", ie);
            }
        }
        
        throw lastException != null 
            ? lastException 
            : new IOException("重试" + maxRetryCount + "次后仍然失败，HTTP状态码: " + response.code());
    }
    
    private boolean shouldRetry(IOException e) {
        return e instanceof SocketTimeoutException || 
               e instanceof ConnectException ||
               e instanceof UnknownHostException;
    }
}
```

**Token自动刷新拦截器：**
```java
public class TokenInterceptor implements Interceptor {
    private final TokenProvider tokenProvider;
    
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request originalRequest = chain.request();
        
        // 添加Token
        String token = tokenProvider.getToken();
        Request request = originalRequest.newBuilder()
            .header("Authorization", "Bearer " + token)
            .build();
        
        Response response = chain.proceed(request);
        
        // 如果返回401，尝试刷新Token后重试
        if (response.code() == 401) {
            response.close();
            
            // 同步刷新Token
            String newToken = tokenProvider.refreshToken();
            if (newToken != null) {
                Request retryRequest = originalRequest.newBuilder()
                    .header("Authorization", "Bearer " + newToken)
                    .build();
                return chain.proceed(retryRequest);
            }
        }
        
        return response;
    }
}
```

## 3.2 连接池管理

OkHttp的连接池是提升性能的核心机制，通过复用TCP连接减少连接建立的开销。

```java
// 自定义连接池
ConnectionPool connectionPool = new ConnectionPool(
    10,                          // maxIdleConnections: 最大空闲连接数
    5,                           // keepAliveDuration: 空闲连接存活时间
    TimeUnit.MINUTES             // 时间单位
);

OkHttpClient client = new OkHttpClient.Builder()
    .connectionPool(connectionPool)
    .build();
```

### 3.2.1 连接池内部实现

ConnectionPool维护一个 `Deque<RealConnection>` 双向队列。工作流程如下：

1. **连接复用**：当发起新请求时，OkHttp会从连接池中查找可复用的连接（匹配主机、端口、协议等）
2. **连接归还**：请求完成后，连接不会被立即关闭，而是放回连接池中等待复用
3. **空闲连接清理**：后台线程（`CleanupRunnable`）定期扫描连接池，关闭超出keepAliveDuration的空闲连接

```java
// ConnectionPool源码核心逻辑
public final class ConnectionPool {
    private final int maxIdleConnections;
    private final long keepAliveDurationNs;
    private final Deque<RealConnection> connections = new ArrayDeque<>();
    
    // 归还连接
    void put(RealConnection connection) {
        connections.add(connection);
        // 触发清理任务
    }
    
    // 获取复用连接
    @Nullable
    RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
        for (RealConnection connection : connections) {
            if (connection.isEligible(address, route)) {
                streamAllocation.acquire(connection, true);
                return connection;
            }
        }
        return null;
    }
    
    // 清理空闲连接的后台任务
    long cleanup(long now) {
        // 遍历所有连接，标记空闲连接
        // 关闭超时的空闲连接
        // 返回下次执行清理的等待时间
    }
}
```

## 3.3 HTTP/2与多路复用

OkHttp是Java端最早支持HTTP/2的客户端之一。HTTP/2的核心优势是**多路复用**——同一连接可以并行处理多个请求，解决了HTTP/1.1的队头阻塞问题。

### 3.3.1 HTTP/2协议协商

OkHttp通过ALPN（Application-Layer Protocol Negotiation）自动协商协议版本：

```
客户端 -> 服务端: 你好，我支持HTTP/2和HTTP/1.1
服务端 -> 客户端: 好的，我们使用HTTP/2（如果服务端支持）
```

对于HTTPS连接，OkHttp默认优先使用HTTP/2；对于HTTP明文连接，OkHttp使用HTTP/1.1。

### 3.3.2 多路复用示例

```java
// HTTP/2多路复用自动生效，无需额外配置
OkHttpClient client = new OkHttpClient.Builder()
    .build();

// 对同一主机的多个请求将共享一个TCP连接
Call call1 = client.newCall(new Request.Builder().url("https://api.example.com/users").build());
Call call2 = client.newCall(new Request.Builder().url("https://api.example.com/posts").build());
Call call3 = client.newCall(new Request.Builder().url("https://api.example.com/comments").build());

// 并发执行，HTTP/2下共享连接
call1.enqueue(callback1);
call2.enqueue(callback2);
call3.enqueue(callback3);
```

### 3.3.3 HTTP/2的特性与优势

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 连接复用 | 串行（需等待上一个请求完成） | 多路复用（Stream并行） |
| 头部压缩 | 无 | HPACK算法压缩 |
| 二进制传输 | 文本 | 二进制帧 |
| 服务端推送 | 不支持 | 支持 |
| 队头阻塞 | 有 | 仅TCP层存在 |

## 3.4 WebSocket支持

OkHttp内置WebSocket客户端支持。

```java
OkHttpClient client = new OkHttpClient.Builder()
    .readTimeout(0, TimeUnit.MILLISECONDS)  // WebSocket需要无限超时
    .build();

Request request = new Request.Builder()
    .url("ws://echo.websocket.org")
    .build();

WebSocket ws = client.newWebSocket(request, new WebSocketListener() {
    @Override
    public void onOpen(WebSocket webSocket, Response response) {
        System.out.println("WebSocket连接已建立");
        webSocket.send("Hello, WebSocket!");
    }

    @Override
    public void onMessage(WebSocket webSocket, String text) {
        System.out.println("收到消息: " + text);
    }

    @Override
    public void onMessage(WebSocket webSocket, ByteString bytes) {
        System.out.println("收到二进制消息: " + bytes.hex());
    }

    @Override
    public void onClosing(WebSocket webSocket, int code, String reason) {
        System.out.println("WebSocket正在关闭: " + reason);
        webSocket.close(1000, null);
    }

    @Override
    public void onClosed(WebSocket webSocket, int code, String reason) {
        System.out.println("WebSocket已关闭: " + code + " " + reason);
    }

    @Override
    public void onFailure(WebSocket webSocket, Throwable t, Response response) {
        System.err.println("WebSocket异常: " + t.getMessage());
    }
});

// 注意：需要让主线程保持存活
TimeUnit.SECONDS.sleep(30);
```

## 3.5 缓存机制

OkHttp根据HTTP缓存规范（RFC 7234）实现缓存策略。

### 3.5.1 启用缓存

```java
// 创建缓存目录
File cacheDir = new File(System.getProperty("java.io.tmpdir"), "okhttp-cache");
// 设置缓存大小（单位：字节）
Cache cache = new Cache(cacheDir, 50 * 1024 * 1024); // 50MB

OkHttpClient client = new OkHttpClient.Builder()
    .cache(cache)
    .build();

Request request = new Request.Builder()
    .url("https://api.example.com/users")
    .build();

// 第一次请求——实际网络请求
Response response1 = client.newCall(request).execute();
System.out.println("第一次请求，是否来自缓存: " + response1.cacheResponse());

// 第二次请求——可能来自缓存（取决于服务端的Cache-Control头）
Response response2 = client.newCall(request).execute();
System.out.println("第二次请求，是否来自缓存: " + response2.cacheResponse());
System.out.println("第二次请求，网络响应: " + response2.networkResponse());
```

### 3.5.2 缓存策略控制

OkHttp遵循服务端返回的缓存控制头：

```
Cache-Control: max-age=3600         // 缓存1小时
Cache-Control: no-cache              // 不能直接使用缓存，需先验证
Cache-Control: no-store              // 禁止缓存
Cache-Control: public               // 任何缓存都可以缓存
Cache-Control: private              // 仅浏览器/客户端缓存
```

客户端也可通过Request添加缓存控制：

```java
// 强制使用缓存（即使缓存已过期）
Request cacheRequest = new Request.Builder()
    .url("https://api.example.com/users")
    .header("Cache-Control", "only-if-cached")
    .header("Cache-Control", "max-stale=86400")  // 接受过期1天内的缓存
    .build();

// 强制刷新缓存
Request freshRequest = new Request.Builder()
    .url("https://api.example.com/users")
    .header("Cache-Control", "no-cache")
    .build();
```

### 3.5.3 缓存流程

```
请求 -> 检查是否有缓存？
   |-- 有缓存且未过期 -> 直接返回缓存响应
   |-- 有缓存但已过期 -> 条件请求（If-Modified-Since / If-None-Match）
   |     |-- 服务端返回304 Not Modified -> 更新缓存有效期后返回缓存
   |     |-- 服务端返回200 -> 更新缓存并返回新响应
   |-- 无缓存 -> 发起网络请求，根据响应头决定是否缓存
```

## 3.6 EventListener事件监听

EventListener允许监听请求的各个生命周期阶段，用于性能监控和问题诊断。

```java
OkHttpClient client = new OkHttpClient.Builder()
    .eventListener(new EventListener() {
        @Override
        public void callStart(Call call) {
            System.out.println("请求开始: " + call.request().url());
        }

        @Override
        public void dnsStart(Call call, String domainName) {
            System.out.println("DNS解析开始: " + domainName);
        }

        @Override
        public void dnsEnd(Call call, String domainName, List<InetAddress> inetAddressList) {
            System.out.println("DNS解析结束: " + domainName + " -> " + inetAddressList);
        }

        @Override
        public void connectStart(Call call, InetSocketAddress inetSocketAddress, Proxy proxy) {
            System.out.println("连接开始: " + inetSocketAddress);
        }

        @Override
        public void connectEnd(Call call, InetSocketAddress inetSocketAddress, 
                              Proxy proxy, @Nullable Protocol protocol) {
            System.out.println("连接建立: " + protocol);
        }

        @Override
        public void connectionAcquired(Call call, Connection connection) {
            System.out.println("获取到连接: " + connection);
        }

        @Override
        public void requestHeadersStart(Call call) {
            System.out.println("发送请求头开始");
        }

        @Override
        public void requestHeadersEnd(Call call, Request request) {
            System.out.println("发送请求头结束");
        }

        @Override
        public void responseHeadersStart(Call call) {
            System.out.println("接收响应头开始");
        }

        @Override
        public void responseHeadersEnd(Call call, Response response) {
            System.out.println("接收响应头结束, 状态码: " + response.code());
        }

        @Override
        public void responseBodyStart(Call call) {
            System.out.println("接收响应体开始");
        }

        @Override
        public void responseBodyEnd(Call call, long byteCount) {
            System.out.println("接收响应体结束, 大小: " + byteCount + " bytes");
        }

        @Override
        public void callEnd(Call call) {
            System.out.println("请求结束");
        }

        @Override
        public void callFailed(Call call, IOException ioe) {
            System.err.println("请求失败: " + ioe.getMessage());
        }
    })
    .build();
```

## 3.7 DNS自定义

OkHttp允许自定义DNS解析逻辑，可用于服务发现、流量调度等场景。

```java
OkHttpClient client = new OkHttpClient.Builder()
    .dns(new Dns() {
        private final Dns SYSTEM = Dns.SYSTEM;
        private final Map<String, List<InetAddress>> mockDns = new HashMap<>();
        
        {
            try {
                mockDns.put("internal-service.example.com", 
                    Arrays.asList(
                        InetAddress.getByName("10.0.1.1"),
                        InetAddress.getByName("10.0.1.2")
                    ));
            } catch (UnknownHostException e) {
                // ignore
            }
        }

        @Override
        public List<InetAddress> lookup(String hostname) throws UnknownHostException {
            // 先查询自定义DNS映射
            if (mockDns.containsKey(hostname)) {
                System.out.println("自定义DNS解析: " + hostname);
                return mockDns.get(hostname);
            }
            // 使用系统DNS
            List<InetAddress> addresses = SYSTEM.lookup(hostname);
            System.out.println("系统DNS解析: " + hostname + " -> " + addresses);
            return addresses;
        }
    })
    .build();
```

## 3.8 认证支持

OkHttp支持基本的认证方案，也可以自定义Authenticator。

```java
// Basic认证
OkHttpClient client = new OkHttpClient.Builder()
    .authenticator(new Authenticator() {
        @Override
        public Request authenticate(Route route, Response response) throws IOException {
            if (response.request().header("Authorization") != null) {
                return null; // 已尝试认证且失败，不再重试
            }
            String credential = Credentials.basic("username", "password");
            return response.request().newBuilder()
                .header("Authorization", credential)
                .build();
        }
    })
    .build();

// 拦截器方式添加Token
OkHttpClient client2 = new OkHttpClient.Builder()
    .addInterceptor(chain -> {
        Request original = chain.request();
        Request authenticated = original.newBuilder()
            .header("Authorization", "Bearer " + getToken())
            .build();
        return chain.proceed(authenticated);
    })
    .build();
```

# 4. 底层原理与架构

## 4.1 整体架构

OkHttp的架构可以看作一个多层处理管道：

```
┌─────────────────────────────────────────────────┐
│  Application Layer                              │
│  (用户代码: OkHttpClient, Request, Call)        │
├─────────────────────────────────────────────────┤
│  Application Interceptors                       │
│  (addInterceptor: 日志、Token、重试等)          │
├─────────────────────────────────────────────────┤
│  Core Pipeline                                  │
│  (重定向处理, 缓存处理, 连接复用等)            │
├─────────────────────────────────────────────────┤
│  Network Interceptors                           │
│  (addNetworkInterceptor: 监控、压缩等)          │
├─────────────────────────────────────────────────┤
│  Connection Management                          │
│  (连接池、HTTP/2多路复用、WebSocket)           │
├─────────────────────────────────────────────────┤
│  Network Layer                                  │
│  (Socket连接、TLS握手、数据传输)               │
└─────────────────────────────────────────────────┘
```

## 4.2 Interceptor链与责任链模式

OkHttp的核心设计模式是责任链模式。每个Interceptor可以决定是否将请求继续向下传递，并在返回时处理响应。

### 4.2.1 RealInterceptorChain源码分析

```java
// OkHttp核心拦截器链
public final class RealInterceptorChain implements Interceptor.Chain {
    private final List<Interceptor> interceptors;
    private final int index;
    private final Request request;
    
    @Override
    public Response proceed(Request request) throws IOException {
        // 到达拦截器链末尾
        if (index >= interceptors.size()) {
            throw new AssertionError();
        }
        
        // 创建下一个拦截器链
        RealInterceptorChain next = new RealInterceptorChain(
            interceptors, transmitter, null, index + 1, request, ...);
        
        // 获取当前拦截器
        Interceptor interceptor = interceptors.get(index);
        
        // 执行当前拦截器（传入下一个链）
        Response response = interceptor.intercept(next);
        
        return response;
    }
}
```

### 4.2.2 OkHttp内置拦截器

OkHttp内部维护了一个不可见的拦截器链，包含以下核心组件：

```
Application Interceptors (用户注册的拦截器)

    ↓
    
RetryAndFollowUpInterceptor
    - 处理请求重试（连接失败时的自动重试）
    - 处理重定向（301/302等）
    - 最多重定向20次

    ↓
    
BridgeInterceptor
    - 添加请求头：Content-Type, Content-Length, Transfer-Encoding, Host, Connection, Accept-Encoding, User-Agent, Cookie等
    - Gzip响应自动解压

    ↓
    
CacheInterceptor
    - 根据HTTP缓存规范处理缓存
    - 条件请求（If-Modified-Since/If-None-Match）

    ↓
    
ConnectInterceptor
    - 从连接池获取连接
    - 如果没有可用连接则创建新连接

    ↓
    
Network Interceptors (用户注册的网络拦截器)

    ↓
    
CallServerInterceptor
    - 真正的网络I/O操作
    - 发送请求数据到服务端
    - 接收服务端响应数据
```

### 4.2.3 责任链模式优势

- **可插拔**：每个Interceptor职责单一，可独立添加或移除
- **顺序可控**：Interceptor的注册顺序影响执行顺序
- **短路机制**：Application Interceptor可以直接返回Mock响应，跳过网络请求
- **嵌套处理**：在Request/Response的多个层次上进行处理

## 4.3 连接管理原理

### 4.3.1 连接获取流程

```
Call.execute() / enqueue()
    ↓
Dispatcher 调度请求
    ↓
RetryAndFollowUpInterceptor
    ↓  streamAllocation.newStream(...)
ConnectInterceptor
    ↓  findHealthyConnection(...)
ConnectionPool.get(address, streamAllocation, route)
    ├── 找到复用连接 → 直接使用
    └── 未找到连接 → RouteSelector.selectRoute()
                          ↓
                    RealConnection.connect()
                          ↓
                    TCP连接 + TLS握手 + HTTP/2协商
                          ↓
                    ConnectionPool.put(connection)
```

### 4.3.2 StreamAllocation

StreamAllocation是连接管理中的关键类，负责协调Connections、Streams和Calls之间的关系：

```java
public final class StreamAllocation {
    // 关联的Connection
    private RealConnection connection;
    
    // 在连接上创建一个新的Stream（HTTP/2中是Stream，HTTP/1.1中独占连接）
    public HttpCodec newStream(...) {
        // HTTP/2: 从连接上分配一个新的Stream ID
        // HTTP/1.1: 如果连接已被占用则阻塞等待
    }
    
    // 获取健康连接
    public RealConnection findHealthyConnection(...) {
        // 检查连接是否健康（Socket未关闭、未过期等）
    }
    
    // 释放连接
    public void release() {
        // 归还连接到连接池
        connectionPool.put(connection);
    }
}
```

## 4.4 连接池管理

### 4.4.1 连接池数据结构

```java
public final class ConnectionPool {
    // 双向队列存储所有连接
    private final Deque<RealConnection> connections = new ArrayDeque<>();
    
    // 连接清理的路由统计
    private final RouteDatabase routeDatabase = new RouteDatabase();
    
    // 后台清理线程
    private boolean cleanupRunning;
    
    // 最大空闲连接数（默认5）
    private final int maxIdleConnections;
    
    // 空闲连接存活时间（默认5分钟）
    private final long keepAliveDurationNs;
}
```

### 4.4.2 空闲连接清理算法

```java
// 清理算法的核心逻辑
long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // 遍历所有连接，分类统计
    for (RealConnection connection : connections) {
        if (connection.isInUse()) {
            inUseConnectionCount++;
        } else {
            idleConnectionCount++;
            long idleDurationNs = now - connection.idleAtNanos;
            if (idleDurationNs > longestIdleDurationNs) {
                longestIdleDurationNs = idleDurationNs;
                longestIdleConnection = connection;
            }
        }
    }

    // 检查是否需要清理
    if (longestIdleDurationNs >= keepAliveDurationNs
        || idleConnectionCount > maxIdleConnections) {
        // 关闭最长时间的空闲连接
        connections.remove(longestIdleConnection);
        return 0; // 立即再次检查
    }

    if (idleConnectionCount > 0) {
        // 返回下次需要清理的时间
        return keepAliveDurationNs - longestIdleDurationNs;
    }

    // 没有空闲连接，等待直到有新连接被添加
    return keepAliveDurationNs;
}
```

## 4.5 线程模型

```
Dispatcher (调度器)
    ├── executorService: ThreadPoolExecutor      ← 异步执行任务的线程池
    ├── runningAsyncCalls: Deque<AsyncCall>      ← 正在执行的异步请求
    ├── readyAsyncCalls: Deque<AsyncCall>        ← 等待执行的异步请求
    └── runningSyncCalls: Deque<RealCall>        ← 正在执行的同步请求

ConnectionPool
    └── cleanupRunning: boolean                   ← 后台清理线程（最多一个）
         └── executor: ThreadPoolExecutor          ← 专门用于连接池清理
```

# 5. 性能优化

## 5.1 连接池调优

```java
// 根据应用场景调整连接池参数
ConnectionPool connectionPool = new ConnectionPool(
    20,                    // maxIdleConnections: 根据并发量调整
    10,                    // keepAliveDuration: 建议5-10分钟
    TimeUnit.MINUTES
);

// 监控连接池状态
System.out.println("空闲连接数: " + connectionPool.idleConnectionCount());
System.out.println("已用连接数: " + connectionPool.connectionCount());
```

## 5.2 超时配置优化

```java
// 不同场景的超时配置
// 内部服务调用（内网）
OkHttpClient internalClient = new OkHttpClient.Builder()
    .connectTimeout(1, TimeUnit.SECONDS)
    .readTimeout(5, TimeUnit.SECONDS)
    .writeTimeout(5, TimeUnit.SECONDS)
    .build();

// 外部API调用（公网）
OkHttpClient externalClient = new OkHttpClient.Builder()
    .connectTimeout(5, TimeUnit.SECONDS)
    .readTimeout(10, TimeUnit.SECONDS)
    .writeTimeout(10, TimeUnit.SECONDS)
    .build();

// 文件上传（大文件）
OkHttpClient uploadClient = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(60, TimeUnit.SECONDS)  // 上传需要更长的写入超时
    .build();
```

## 5.3 单例模式复用OkHttpClient

```java
public class HttpClientManager {
    private static volatile OkHttpClient instance;
    
    private HttpClientManager() {}
    
    public static OkHttpClient getInstance() {
        if (instance == null) {
            synchronized (HttpClientManager.class) {
                if (instance == null) {
                    instance = createClient();
                }
            }
        }
        return instance;
    }
    
    private static OkHttpClient createClient() {
        ConnectionPool pool = new ConnectionPool(20, 5, TimeUnit.MINUTES);
        
        return new OkHttpClient.Builder()
            .connectionPool(pool)
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(10, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .addInterceptor(new LoggingInterceptor())
            .eventListener(new MetricsEventListener())
            .build();
    }
}

// 使用
OkHttpClient client = HttpClientManager.getInstance();
```

## 5.4 缓存优化

```java
// 合理设置缓存大小
// 对于资源类API（图片、CSS等），可以设置较大的缓存
Cache resourceCache = new Cache(new File("/tmp/resource-cache"), 100 * 1024 * 1024);

// 对于数据类API，设置较小缓存
Cache dataCache = new Cache(new File("/tmp/data-cache"), 10 * 1024 * 1024);

// 监控缓存命中率
Cache cache = client.cache();
System.out.println("缓存请求次数: " + cache.requestCount());
System.out.println("缓存命中次数: " + cache.hitCount());
System.out.println("缓存网络请求次数: " + cache.networkCount());
System.out.println("缓存命中率: " + (double) cache.hitCount() / cache.requestCount() * 100 + "%");
```

## 5.5 禁用无用特性

```java
OkHttpClient client = new OkHttpClient.Builder()
    // 如果不需要自动重定向
    .followRedirects(false)
    .followSslRedirects(false)
    // 如果不需要缓存
    .cache(null)
    // 根据需要对Cookie做处理
    .cookieJar(new NoopCookieJar())
    .build();
```

## 5.6 监控与度量

```java
// 集成Micrometer等监控工具
public class MicrometerEventListener extends EventListener {
    private final MeterRegistry registry;
    
    public MicrometerEventListener(MeterRegistry registry) {
        this.registry = registry;
    }
    
    @Override
    public void callStart(Call call) {
        registry.counter("okhttp.call.start").increment();
    }
    
    @Override
    public void dnsEnd(Call call, String domainName, List<InetAddress> inetAddressList) {
        registry.timer("okhttp.dns.latency")
            .record(Duration.between(start, Instant.now()));
    }
    
    @Override
    public void connectEnd(Call call, InetSocketAddress inetSocketAddress, 
                          Proxy proxy, Protocol protocol) {
        registry.timer("okhttp.connect.latency")
            .record(Duration.between(start, Instant.now()));
    }
    
    @Override
    public void callEnd(Call call) {
        registry.counter("okhttp.call.end", "status", "success").increment();
    }
    
    @Override
    public void callFailed(Call call, IOException ioe) {
        registry.counter("okhttp.call.end", "status", "failed").increment();
    }
}
```

# 6. 与Spring Boot集成

## 6.1 配置RestTemplate使用OkHttp

### 6.1.1 添加依赖

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
</dependency>
```

### 6.1.2 Java配置

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(20, 5, TimeUnit.MINUTES))
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(10, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .addInterceptor(new LoggingInterceptor())
            .build();
    }

    @Bean
    public RestTemplate restTemplate(OkHttpClient okHttpClient) {
        // 使用OkHttp作为RestTemplate的底层HTTP引擎
        ClientHttpRequestFactory requestFactory = 
            new OkHttp3ClientHttpRequestFactory(okHttpClient);
        
        RestTemplate restTemplate = new RestTemplate(requestFactory);
        
        // 添加自定义消息转换器
        restTemplate.getMessageConverters().add(new MappingJackson2HttpMessageConverter());
        
        return restTemplate;
    }
}
```

### 6.1.3 使用示例

```java
@Service
public class UserService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public List<User> getUsers() {
        ResponseEntity<List<User>> response = restTemplate.exchange(
            "https://api.example.com/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<User>>() {}
        );
        return response.getBody();
    }
    
    public User createUser(User user) {
        ResponseEntity<User> response = restTemplate.postForEntity(
            "https://api.example.com/users",
            user,
            User.class
        );
        return response.getBody();
    }
    
    public void deleteUser(Long id) {
        restTemplate.delete("https://api.example.com/users/{id}", id);
    }
}
```

## 6.2 配置WebClient使用OkHttp（Spring WebFlux）

### 6.2.1 添加依赖

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### 6.2.2 Java配置

```java
@Configuration
public class WebClientConfig {

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(50, 10, TimeUnit.MINUTES))
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(10, TimeUnit.SECONDS)
            .protocols(Arrays.asList(Protocol.HTTP_2, Protocol.HTTP_1_1))
            .build();
    }

    @Bean
    public WebClient webClient(OkHttpClient okHttpClient) {
        // 通过Reactor Netty的Connector使用OkHttp
        // 注意：此处用RestTemplate的ClientHttpRequestFactory示例
        // 实际上WebClient底层默认使用Reactor Netty
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create().wiretap(true)
            ))
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    /**
     * 使用OkHttp作为WebClient底层引擎
     * 需要引入 okhttp-client-adaptor 或使用 OkHttpReactorClientHttpConnector
     */
    @Bean
    public WebClient okHttpWebClient() {
        return WebClient.builder()
            .clientConnector(new OkHttpReactorClientHttpConnector())
            .build();
    }
}

// 自定义Connector适配器
class OkHttpReactorClientHttpConnector implements ClientHttpConnector {
    private final OkHttpClient okHttpClient;
    
    public OkHttpReactorClientHttpConnector() {
        this.okHttpClient = new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(20, 5, TimeUnit.MINUTES))
            .build();
    }
    
    @Override
    public Mono<ClientHttpResponse> connect(HttpMethod method, URI uri,
                                            Function<? super ClientHttpRequest, Mono<Void>> requestCallback) {
        return Mono.create(sink -> {
            Request.Builder requestBuilder = new Request.Builder()
                .url(uri.toString())
                .method(method.name(), null);
            
            okHttpClient.newCall(requestBuilder.build()).enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    sink.error(e);
                }
                
                @Override
                public void onResponse(Call call, Response response) throws IOException {
                    sink.success(new OkHttpClientHttpResponse(response));
                }
            });
        });
    }
}
```

## 6.3 通过Retrofit集成Spring Boot

Retrofit是Square公司的另一个库，默认使用OkHttp作为底层HTTP引擎。

```xml
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>retrofit</artifactId>
    <version>2.9.0</version>
</dependency>
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>converter-jackson</artifactId>
    <version>2.9.0</version>
</dependency>
```

```java
@Configuration
public class RetrofitConfig {

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(20, 5, TimeUnit.MINUTES))
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .addInterceptor(new RetryInterceptor(3))
            .build();
    }

    @Bean
    public Retrofit retrofit(OkHttpClient okHttpClient) {
        return new Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(JacksonConverterFactory.create())
            .addCallAdapterFactory(RxJava3CallAdapterFactory.create()) // 可选
            .build();
    }

    @Bean
    public UserApi userApi(Retrofit retrofit) {
        return retrofit.create(UserApi.class);
    }
}

// 定义API接口
interface UserApi {
    @GET("users")
    Call<List<User>> getUsers();
    
    @GET("users/{id}")
    Call<User> getUser(@Path("id") Long id);
    
    @POST("users")
    @Headers("Content-Type: application/json")
    Call<User> createUser(@Body User user);
    
    @PUT("users/{id}")
    Call<User> updateUser(@Path("id") Long id, @Body User user);
    
    @DELETE("users/{id}")
    Call<Void> deleteUser(@Path("id") Long id);
}

// 在Service中使用
@Service
public class UserService {
    
    @Autowired
    private UserApi userApi;
    
    public List<User> getAllUsers() throws IOException {
        Response<List<User>> response = userApi.getUsers().execute();
        if (response.isSuccessful()) {
            return response.body();
        }
        throw new RuntimeException("请求失败: " + response.code());
    }
    
    public void getAllUsersAsync() {
        userApi.getUsers().enqueue(new Callback<List<User>>() {
            @Override
            public void onResponse(Call<List<User>> call, Response<List<User>> response) {
                // 处理响应
            }
            
            @Override
            public void onFailure(Call<List<User>> call, Throwable t) {
                // 处理失败
            }
        });
    }
}
```

## 6.4 Spring Boot自动配置（YAML配置）

```yaml
okhttp:
  connect-timeout: 5s
  read-timeout: 10s
  write-timeout: 10s
  max-idle-connections: 20
  keep-alive-duration: 5m
  retry-on-connection-failure: true

spring:
  rest-template:
    okhttp:
      enabled: true
```

```java
@ConfigurationProperties(prefix = "okhttp")
@Configuration
public class OkHttpProperties {
    private Duration connectTimeout = Duration.ofSeconds(5);
    private Duration readTimeout = Duration.ofSeconds(10);
    private Duration writeTimeout = Duration.ofSeconds(10);
    private int maxIdleConnections = 20;
    private Duration keepAliveDuration = Duration.ofMinutes(5);
    private boolean retryOnConnectionFailure = true;

    public OkHttpClient createOkHttpClient() {
        return new OkHttpClient.Builder()
            .connectTimeout(connectTimeout.toMillis(), TimeUnit.MILLISECONDS)
            .readTimeout(readTimeout.toMillis(), TimeUnit.MILLISECONDS)
            .writeTimeout(writeTimeout.toMillis(), TimeUnit.MILLISECONDS)
            .connectionPool(new ConnectionPool(maxIdleConnections, 
                keepAliveDuration.toMillis(), TimeUnit.MILLISECONDS))
            .retryOnConnectionFailure(retryOnConnectionFailure)
            .build();
    }
    
    // getters and setters
}
```

# 7. 最佳实践与常见问题

## 7.1 最佳实践

### 7.1.1 OkHttpClient复用

**必须复用OkHttpClient实例**。每次创建新的OkHttpClient会创建新的连接池和线程池，导致资源泄露和性能下降。

```java
// ❌ 错误做法：每次请求都创建新实例
public void badPractice() {
    OkHttpClient client = new OkHttpClient();
    // ...
}

// ✅ 正确做法：单例复用
public class HttpClientHolder {
    public static final OkHttpClient CLIENT = new OkHttpClient.Builder()
        .connectionPool(new ConnectionPool(20, 5, TimeUnit.MINUTES))
        .build();
}
```

### 7.1.2 ResponseBody正确关闭

```java
// ✅ 正确做法：使用try-with-resources或finally确保关闭
try (Response response = client.newCall(request).execute()) {
    String body = response.body().string();
    // 处理body
} catch (IOException e) {
    // 处理异常
    // Response和ResponseBody自动关闭
}

// ✅ 也可以手动关闭
Response response = client.newCall(request).execute();
try {
    String body = response.body().string();
    // 处理body
} finally {
    response.close();  // 重要：关闭响应以释放连接
}
```

### 7.1.3 ResponseBody只读一次

```java
Response response = client.newCall(request).execute();

// 第一次读取成功
String body1 = response.body().string();
System.out.println(body1);

// ❌ 第二次读取失败：body已被消费，返回空字符串
String body2 = response.body().string();
System.out.println(body2); // ""

// ✅ 正确做法：读取后保存到变量
String body = response.body().string();
```

### 7.1.4 超时设置的合理性

```java
// 连接超时不要太长（通常1-10秒）
// 读取超时根据API响应时间设定
// 写入超时根据请求体大小设定

// 对于长时间轮询的API，使用单独的client实例
OkHttpClient longPollingClient = client.newBuilder()
    .readTimeout(0, TimeUnit.SECONDS)  // 无限超时
    .build();
```

### 7.1.5 使用日志框架打印完整请求响应

```java
public class CurlLoggingInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        
        // 打印类似curl的格式
        StringBuilder curl = new StringBuilder("curl -v");
        curl.append(" -X ").append(request.method());
        curl.append(" '").append(request.url()).append("'");
        
        for (String headerName : request.headers().names()) {
            curl.append(" -H '").append(headerName)
                .append(": ").append(request.header(headerName)).append("'");
        }
        
        RequestBody body = request.body();
        if (body != null && body.contentLength() > 0) {
            Buffer buffer = new Buffer();
            body.writeTo(buffer);
            curl.append(" -d '").append(buffer.readUtf8()).append("'");
        }
        
        log.debug(curl.toString());
        
        return chain.proceed(request);
    }
}
```

## 7.2 常见问题与解决方案

### 7.2.1 连接池耗尽

**现象**：请求被阻塞或超时
**原因**：并发请求超过连接池容量，或连接未被正确释放
**解决方案**：

```java
// 1. 增大连接池
OkHttpClient client = new OkHttpClient.Builder()
    .connectionPool(new ConnectionPool(100, 10, TimeUnit.MINUTES))
    .build();

// 2. 增加Dispatcher的并发限制
Dispatcher dispatcher = new Dispatcher();
dispatcher.setMaxRequests(100);
dispatcher.setMaxRequestsPerHost(10);

// 3. 确保响应正确关闭
// 4. 监控连接池使用情况
```

### 7.2.2 读取超时

**现象**：SocketTimeoutException
**原因**：服务端处理过慢或响应内容过大
**解决方案**：

```java
// 1. 合理增大超时时间
OkHttpClient slowClient = client.newBuilder()
    .readTimeout(60, TimeUnit.SECONDS)
    .build();

// 2. 使用流式读取大响应（避免一次性加载到内存）
try (Response response = client.newCall(request).execute()) {
    BufferedSource source = response.body().source();
    while (!source.exhausted()) {
        String line = source.readUtf8Line();
        // 逐行处理
    }
}
```

### 7.2.3 DNS解析失败

**现象**：UnknownHostException
**解决方案**：

```java
// 自定义DNS，增加备用IP
OkHttpClient client = new OkHttpClient.Builder()
    .dns(hostname -> {
        try {
            return Dns.SYSTEM.lookup(hostname);
        } catch (UnknownHostException e) {
            // 备用DNS解析
            if ("api.example.com".equals(hostname)) {
                return Arrays.asList(InetAddress.getByName("203.0.113.1"));
            }
            throw e;
        }
    })
    .build();
```

### 7.2.4 SSL/TLS握手问题

```java
// 跳过SSL验证（仅开发环境使用！）
OkHttpClient debugClient = client.newBuilder()
    .hostnameVerifier((hostname, session) -> true)
    .sslSocketFactory(createInsecureSslSocketFactory(), createInsecureTrustManager())
    .build();

// 生产环境使用自定义信任证书
OkHttpClient secureClient = client.newBuilder()
    .sslSocketFactory(sslSocketFactory, trustManager)
    .hostnameVerifier(new StrictHostnameVerifier())
    .build();
```

### 7.2.5 HTTP/2连接异常

```java
// 如果服务端HTTP/2支持不完善，可降级到HTTP/1.1
OkHttpClient http11Client = client.newBuilder()
    .protocols(Arrays.asList(Protocol.HTTP_1_1))
    .build();
```

# 8. 与其他HTTP客户端对比

## 8.1 OkHttp vs Apache HttpClient

| 对比维度 | OkHttp | Apache HttpClient |
|---------|--------|-------------------|
| 包大小 | 较小（约700KB） | 较大（约2MB+） |
| API设计 | 链式调用，简洁优雅 | 较为繁琐，配置复杂 |
| HTTP/2 | 原生支持，自动协商 | 5.x支持，4.x需额外配置 |
| WebSocket | 原生支持 | 需扩展库 |
| 性能 | 更高（连接复用优化好） | 略低 |
| 连接池 | 自动管理 | 需手动配置ConnectionManager |
| Interceptor机制 | 责任链模式，灵活强大 | 类似但略弱 |
| Android支持 | 官方推荐，完美支持 | 不推荐（包体积大） |
| Spring Boot集成 | RestTemplate/WebClient | RestTemplate/RestClient |
| 文档 | 较好 | 丰富但分散 |

## 8.2 OkHttp vs Java原生HttpURLConnection

| 对比维度 | OkHttp | HttpURLConnection |
|---------|--------|-------------------|
| API易用性 | 优秀（Builder模式） | 较差（API古老） |
| HTTP/2 | 支持 | JDK 9+支持 |
| 连接池 | 内置 | 无 |
| 拦截器 | 支持 | 不支持 |
| 性能 | 高 | 中 |
| 维护活跃度 | 活跃 | 不活跃 |

## 8.3 OkHttp vs RestTemplate（Spring）

RestTemplate本身是一个高层API封装，底层依赖于具体的HTTP客户端实现。OkHttp可以作为RestTemplate的底层引擎。

| 对比维度 | OkHttp | RestTemplate |
|---------|--------|-------------|
| 定位 | HTTP客户端库 | HTTP访问模板 |
| 抽象层次 | 底层 | 上层封装 |
| 序列化 | 不处理 | 支持多种MessageConverter |
| 异常处理 | IOException | RestClientException体系 |
| 异步 | Callback | ListenableFuture/CompletableFuture |

# 9. 面试要点

## 9.1 基础知识

1. **OkHttp的核心组件有哪些？**
   - OkHttpClient：配置和创建Call的工厂
   - Request/Response：请求和响应的不可变对象
   - Call：代表一个待执行的请求
   - Interceptor：拦截器，基于责任链模式
   - ConnectionPool：连接池，管理TCP连接复用

2. **OkHttp的优势是什么？**
   - HTTP/2多路复用
   - 连接池自动管理
   - Interceptor责任链模式
   - 简洁的API设计
   - 优秀的性能表现

3. **如何实现OkHttp的全局单例？**
   - 使用双重检查锁定（DCL）或静态内部类方式
   - 配置连接池、超时、拦截器等参数

## 9.2 高级面试题

4. **OkHttp的Interceptor和NetworkInterceptor有何区别？**
   - Application Interceptor：在应用层拦截，只执行一次，可看到缓存响应
   - Network Interceptor：在网络层拦截，重定向和重试都会执行，只看到真实网络响应

5. **OkHttp连接池的工作原理？**
   - 使用Deque<RealConnection>存储连接
   - 后台线程定期清理空闲连接
   - 根据host、port、protocol匹配复用连接
   - HTTP/1.1：一个连接同时只能处理一个请求
   - HTTP/2：一个连接可同时处理多个Stream

6. **OkHttp的线程模型是怎样的？**
   - Dispatcher管理请求调度
   - 异步请求使用内部线程池执行
   - 连接池使用独立的后台线程清理
   - EventListener的回调在请求执行的线程中

7. **OkHttp如何支持HTTP/2？**
   - 通过ALPN协议协商
   - HTTPS默认尝试HTTP/2
   - HTTP/2下共享TCP连接，多路复用
   - 每个Stream独立收发，互不阻塞

8. **如何监控OkHttp的性能？**
   - EventListener监听请求各阶段耗时
   - ConnectionPool的idleConnectionCount()和connectionCount()
   - Cache的hitCount()和networkCount()
   - 自定义Interceptor记录请求耗时

9. **OkHttp的缓存策略是怎样的？**
   - 遵循RFC 7234缓存规范
   - 根据Cache-Control头决定缓存行为
   - 支持条件请求（If-Modified-Since/If-None-Match）
   - 可通过Request请求头控制缓存策略

10. **使用OkHttp有哪些常见陷阱？**
    - 未复用OkHttpClient导致连接池失效
    - ResponseBody未关闭导致连接泄露
    - ResponseBody重复读取导致空字符串
    - 同步请求在UI线程执行导致ANR（Android）
    - 未设置合理超时导致线程阻塞
