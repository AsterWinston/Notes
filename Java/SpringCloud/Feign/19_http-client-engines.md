# 1. 概述

Feign 的 `Client` 接口抽象了底层 HTTP 通信。Spring Cloud OpenFeign 支持三种 HTTP 引擎：

| 引擎 | 类 | 连接池 | 默认 |
|------|-----|--------|------|
| **HttpURLConnection** | `Client.Default` | 无 | 是 |
| **Apache HttpClient** | `ApacheHttpClient` | 有 (PoolingHttpClientConnectionManager) | — |
| **OkHttp** | `OkHttpClient` | 有 (ConnectionPool) | — |

## 1.1 Client 接口的契约只有一行，但坑全在这一行里

```java
public interface Client {
    Response execute(Request request, Request.Options options) throws IOException;
}
```

两个契约要素决定了本篇一切行为差异：`options` 是**每次调用传入**的——引擎实现要么每次应用它（OkHttp 经 `newBuilder()` 派生、Apache 转成 per-request `RequestConfig`），要么忽略它让 Bean 上的全局值说了算（老版本 Apache 适配即如此），`11_timeout-connection.md` §5.2 说的"两处超时不一致以谁为准"归根结底是各 Client 实现的选择；返回的 `Response.body()` 是**活流**，持有连接——`18_file-upload-download.md` §6.2 的关闭纪律、FULL 日志的 rebuffer 都源于此。换引擎换的是"连接怎么管、超时怎么落、编码谁来解压"，Feign 层语义完全不变。

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

"新建连接"要说得精确些：JDK 自带一个按目标的空闲连接缓存（`http.maxConnections` 默认 5），但它没有借还排队语义、空闲回收惰性到下次使用才探测（`11_timeout-connection.md` §5.3）。源码还原里还有一个细节是排障常用的：4xx/5xx 时读 `getErrorStream()`——这就是为什么**服务端错误响应的 body 能被 ErrorDecoder 拿到**而某些中间件吞错误流时拿不到；另外它对 HEAD 之外所有方法统一 `setDoOutput` 的写法、以及不支持 per-route 池上限，都意味着它的容量天花板非常低，压测里 QPS 上不去先看引擎（§6 的 450 vs 1400 就是这个差距的量级感）。

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

## 3.4  Apache 的"三个超时"模型

这段配置真正的精华在三个超时各管一段，漏配任何一个都会产生一类"想不通"的故障：`connectTimeout` 管 TCP+TLS 建连；`socketTimeout` 管数据包间隔（即 Feign 的 read-timeout 语义）；**`connectionRequestTimeout` 管"从池里等一条连接"**——它不属于任何网络异常类别，池满/下游慢导致的排队在 3s 抛 `ConnectionPoolTimeoutException`，栈里没有 SocketTimeout 字样，很多团队因此从没把它和"超时"联系起来（`11_timeout-connection.md` §8.1 的池耗尽定性靠它落地）。默认值是无限等待：池满时业务线程**静默排队**，Tomcat 线程池被慢慢抽干，表现为"整体接口雪崩但下游毫无异常"——生产必须显式设成秒级。另外 `DefaultHttpRequestRetryHandler(0, false)` 把引擎层的重试关掉是刻意的正确姿势：0 次 + 不重试非幂等请求，把重试权完整交还给 Feign Retryer / LoadBalancer，避免第三层放大（`13_loadbalancer.md` §10 的乘法表里，引擎层是最隐蔽的乘数）。

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

## 4.4 三个必须知道的 OkHttp 行为

**per-call 超时的实现**：`feign-okhttp` 拿到 Feign 的 `Options` 后并不直接用注入的 Bean，而是 `client.newBuilder().connectTimeout(...).readTimeout(...).build()` 派生——派生实例**共享原池与 Dispatcher**，成本可控，部分版本还按超时组合缓存。实践含义：Bean 里写死的超时只是"没被 Options 覆盖时的底"，两处不一致以 per-call 为准（`11_timeout-connection.md` §5.2 的机制版）。**写超时是它独有的第四段**（Apache 没有独立的 write timeout）：大 body 上传卡在发送队列的时间由 `writeTimeout` 管，配了文件上传接口（`18_file-upload-download.md` §9）时这是 OkHttp 路线必须显式定档的参数。**maxIdleConnections 不是并发上限**：它是空闲回收水位，活跃连接数实际由 Dispatcher（默认 maxRequests=64 / 64）与池的按需建连决定——从 Apache 的 `maxPerRoute` 心智迁过来会把"排队保护"配丢：OkHttp 不排队，它会**直接建更多连接**，保护下游要靠 Bulkhead 而不是池参数。§4.3 里 `retryOnConnectionFailure(false)` 的正确性同理于 §3.4：OkHttp 的自动重试会在"连接失败换路由/换地址"层面偷偷重放，与上层重试叠加前必须掐掉。

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

读表两个纠偏："Gzip 自动解压：手动"对 Apache 不准确——4.x/5.x 默认注册内容解压拦截器（`17_compression.md` §6.2），表意应为"可关闭、行为受配置面控制"；"自动重试：可配置"三家都有，差别在默认值——OkHttp 默认开（所以要显式关，§4.3），Apache 默认关（所以要显式开才危险）。**默认值差异比能力差异更容易坑人**，选型时把两列换成"出厂行为"再读一遍。

# 6. 性能测试参考

```text
场景：1000 次请求，并发 50，本地回环

引擎                  平均响应时间     P99         QPS
HttpURLConnection      8ms            25ms        450
Apache HttpClient      3ms            10ms        1200
OkHttp                 2ms            8ms         1400
```

> 数据仅供参考，实际性能取决于网络环境、请求/响应大小、SSL 等因素。

这组回环数据的意义只在**相对排序**（默认引擎的建连开销在并发下被放大到 2-3 倍差距），绝对值没有移植性：本地回环 0ms 网络延迟下测的是纯客户端开销，真实内网里服务端 RT 占大头后差距收敛到几个百分点。自测时的三个方法学陷阱：没预热先打点（首分钟数据全是建连与 JIT）、压测端自身成为瓶颈（50 并发用业务线程模拟时，先确认发出端 CPU 没满）、小 JSON 外推大报文（引擎的缓冲策略差异在 KB 级报文上不显现，`18` 那类 10MB 场景结论会翻转）。真要对比，按你的**报文尺寸分布 × 目标 RT × 并发**建场景，看 P99 与错误率而不是平均数。

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

两段的"仅开发环境"注释要当真执行：`NoopHostnameVerifier` / trust-all 关掉的是**中间人防线**，代码里常见的悲剧是开发便利写法随配置一起进了生产。生产形态的正确层次：内部 CA 签发的证书直接进 JVM 默认 truststore（`-Djavax.net.ssl.trustStore` 或 keytool 导入），一行代码都不用写；双向认证（mTLS）才是本节的正当用途——`loadKeyMaterial`(Apache)/`sslSocketFactory`+`X509ExtendedKeyManager`(OkHttp) 挂客户端证书，且 hostname 校验保持开启。`disable-ssl-validation` 那个 yml 开关（`11_timeout-connection.md` §5.1）同理：它只配出现在本地 profile 里。

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

这行"谁优先"不建议当架构依赖：多引擎同 classpath 时的胜出者由自动配置的注册与 `@ConditionalOnMissingBean` 顺序决定，跨小版本有过变化，而且**胜出者的池参数来自哪套配置也跟着变**（Apache 赢时 `httpclient.*` 生效、OkHttp 赢时 `ConnectionPool` 用 Bean/默认值——两边配置半套失效半套生效是最混乱的状态）。工程纪律：留一个 `enabled: true`，另一个显式 false，甚至把不用的 `feign-*` 依赖从 pom 里排掉，让"只有一个引擎"成为编译期事实。

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

## 9.1 表之外，真正的决策变量是三个

**池模型与你需要的保护语义匹配吗**：要"池满排队 + 排队超时快速失败"的背压选 Apache（maxPerRoute + connectionRequestTimeout 三件套），要"轻快复用、不设硬顶"选 OkHttp，反过来说 OkHttp 路线必须补一层 Bulkhead 才等价安全；**可观测与故障史**：团队已有 Apache 的调参/排障经验（`ConnectionPoolTimeoutException` 的处置手册），换引擎的隐性成本比性能表上的几个百分点贵；**生态联动**：压缩默认行为（§5 表）、per-call 超时应用方式（§4.4）、与 Sentinel/OTel 的插桩兼容度都跟引擎绑定，选型时按 `17`（压缩）与 `21_logging_monitoring.md`（埋点）的章节各过一遍清单。没有历史包袱的新项目，表里"生产通用 → OkHttp"是合理默认；有 Apache 资产的团队维持 Apache 也完全成立——**两强的差距在调优是否到位，不在引擎本身**。
