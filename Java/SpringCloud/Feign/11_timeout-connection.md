
# 1. 概述

Feign 超时配置涉及两个层面：
1. **Feign 层面的超时**：`Options` 对象管理 connectTimeout 和 readTimeout
2. **底层 HTTP 客户端的超时**：不同引擎（HttpURLConnection / Apache HttpClient / OkHttp）有各自的超时机制

# 2. Feign Options

```java
public static class Options {
    // 连接超时 (ms)
    private final long connectTimeout;
    // 连接超时时间单位
    private final TimeUnit connectTimeoutUnit;
    // 读取超时 (ms)
    private final long readTimeout;
    // 读取超时时间单位
    private final TimeUnit readTimeoutUnit;
    // 是否跟随重定向
    private final boolean followRedirects;

    public Options(long connectTimeout, TimeUnit connectTimeoutUnit,
                   long readTimeout, TimeUnit readTimeoutUnit,
                   boolean followRedirects) { ... }

    // 便捷构造
    public Options(long connectTimeoutMillis, long readTimeoutMillis) {
        this(connectTimeoutMillis, MILLISECONDS, readTimeoutMillis, MILLISECONDS, true);
    }
}
```

# 3. 配置超时

## 3.1 YAML 配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 5000       # 连接超时 (ms)
            read-timeout: 10000          # 读取超时 (ms)
            follow-redirects: true       # 是否跟随重定向（部分版本支持）
          user-service:
            connect-timeout: 3000
            read-timeout: 5000
```

## 3.2 代码配置（Request.Options Bean）

```java
@Bean
public Request.Options requestOptions() {
    return new Request.Options(
            5, TimeUnit.SECONDS,   // connectTimeout
            10, TimeUnit.SECONDS,  // readTimeout
            true                   // followRedirects
    );
}
```

## 3.3 全局 vs 特定 FeignClient

```java
// 全局
@Configuration
public class GlobalFeignConfig {
    @Bean
    public Request.Options options() {
        return new Request.Options(3000, 5000);
    }
}

// 特定 FeignClient
@FeignClient(name = "user-service", configuration = UserFeignConfig.class)
public interface UserClient {}
```

```java
public class UserFeignConfig {
    @Bean
    public Request.Options options() {
        return new Request.Options(2000, 3000);
    }
}
```

# 4. 超时生效过程

```
FeignClient 调用
  │
  │  Options 对象在 Feign.Builder 或 Spring 容器中
  │
  ▼
SynchronousMethodHandler.invoke()
  │
  │  client.execute(request, options)
  │
  ▼
底层 HTTP 引擎
  ├── Apache HttpClient  →  HttpRequestConfig.setConnectTimeout() / .setSocketTimeout()
  ├── OkHttp              →  OkHttpClient.Builder.connectTimeout() / .readTimeout()
  └── HttpURLConnection   →  URLConnection.setConnectTimeout() / .setReadTimeout()
```

# 5. 连接池管理

## 5.1 Apache HttpClient 连接池

```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        enabled: true
        max-connections: 200                  # 总连接数
        max-connections-per-route: 50          # 每个路由的最大连接数
        time-to-live: 900                      # 连接存活时间
        time-to-live-unit: seconds
        connection-timer-repeat: 3000          # 连接清理间隔 (ms)
        disable-ssl-validation: false          # 是否禁用 SSL 校验
```

**底层原理**：

```java
@Configuration
@ConditionalOnClass(ApacheHttpClient.class)
public class HttpClientFeignConfiguration {

    @Bean
    @ConditionalOnMissingBean(org.apache.http.client.HttpClient.class)
    public HttpClient httpClient(FeignHttpClientProperties properties) {
        // 1. 创建连接管理器
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(properties.getMaxConnections());
        cm.setDefaultMaxPerRoute(properties.getMaxConnectionsPerRoute());

        // 2. 设置连接存活时间
        cm.setValidateAfterInactivity((int) properties.getTimeToLiveUnit()
                .toMillis(properties.getTimeToLive()));

        // 3. 配置 RequestConfig
        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(connectTimeout)
                .setSocketTimeout(readTimeout)
                .setRedirectsEnabled(followRedirects)
                .build();

        // 4. 构建 CloseableHttpClient
        return HttpClients.custom()
                .setConnectionManager(cm)
                .setDefaultRequestConfig(requestConfig)
                .build();
    }
}
```

## 5.2 OkHttp 连接池

```yaml
spring:
  cloud:
    openfeign:
      okhttp:
        enabled: true
```

```java
// OkHttp 自带连接池，默认 5 个空闲连接，存活 5 分钟
// 可通过自定义 OkHttpClient Bean 覆盖

@Bean
public okhttp3.OkHttpClient okHttpClient() {
    return new okhttp3.OkHttpClient.Builder()
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .connectionPool(new ConnectionPool(
                    50,                              // maxIdleConnections
                    5, TimeUnit.MINUTES              // keepAliveDuration
            ))
            .addInterceptor(new OkHttpLoggingInterceptor())
            .build();
}
```

## 5.3 HttpURLConnection 注意

默认引擎 `Client.Default` 使用 `HttpURLConnection`，**没有连接池**，每次请求新建连接。**生产环境强烈建议替换为 Apache HttpClient 或 OkHttp**。

# 6. 超时与重试的关系

```
第 1 次尝试: connectTimeout=3000, readTimeout=5000
  失败 (RetryableException)
第 2 次尝试: connectTimeout=3000, readTimeout=5000  ← 每次尝试独立计时
  失败 (RetryableException)
第 3 次尝试: connectTimeout=3000, readTimeout=5000
  成功 / 最终失败

总耗时上限 ≈ maxAttempts × max(readTimeout, connectTimeout)
```

**风险**：大重试次数 + 大超时值 = 请求堆积。例如 `maxAttempts=5 + readTimeout=10s = 最坏 50s`。

**建议**：配合 `maxAutoRetries`、`readTimeout` 和熔断器超时设置，避免级联超时。

# 7. 连接保活（Keep-Alive）

## 7.1 Apache HttpClient

```java
@Bean
public HttpClient httpClient() {
    return HttpClients.custom()
            .setKeepAliveStrategy((response, context) -> 30_000)  // 30s
            .setConnectionManager(connectionManager())
            .build();
}
```

## 7.2 OkHttp

```java
new OkHttpClient.Builder()
    .connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES))
    .build();
```

OkHttp 默认自动开启 Keep-Alive，且有自己的连接保活检查。

# 8. 超时问题排查

## 8.1 连接超时排查

```
检查点：
1. 目标服务端口是否可达 (telnet host port)
2. 防火墙 / 安全组规则
3. 负载均衡器是否正确路由
4. DNS 解析是否正确
5. 连接池是否耗尽（PoolEmptyException / ConnectionPoolTimeoutException）
```

## 8.2 读取超时排查

```
检查点：
1. 目标服务处理时间是否过长
2. 网络延迟是否异常
3. 请求体 / 响应体是否过大
4. 是否存在慢 SQL / 慢缓存 / 死锁
```

# 9. 最佳实践

| 建议 | 值 | 说明 |
|------|-----|------|
| 连接超时 | 2-5s | 建连是轻量操作，不需要太长 |
| 读取超时 | 5-30s | 取决于下游服务的 SLA |
| 连接池大小 | per-route × 下游实例数 | 每路由 20-50 即可 |
| 连接空闲回收 | 30-60s | 避免长连接被中间网络设备断开 |
