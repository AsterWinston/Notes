# 1. 概述

## 1.1 超时配置的两个层面

Feign 超时配置涉及两个层面：
1. **Feign 层面的超时**：`Options` 对象管理 connectTimeout 和 readTimeout
2. **底层 HTTP 客户端的超时**：不同引擎（HttpURLConnection / Apache HttpClient / OkHttp）有各自的超时机制

两层的关系是"请求级参数"与"引擎级设施"：`Options` 随每次调用传入，绝大多数引擎在 `execute` 时会用它（Apache 转成 per-request 的 `RequestConfig`、OkHttp 经 `newBuilder()` 克隆生效、HttpURLConnection 直接 set）；而连接池、TTL、空闲回收属于引擎全局，不受 Options 影响。因此"改 read-timeout 却全链路没变化"通常说明真正卡点在引擎层或网络层，而不是 Feign 配置层——先分清你在调哪一层。

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

## 2.1 第三种超时入口：方法参数

除 yml 与 `Options` Bean 外，Feign 还支持**在客户端方法签名里带一个 `Request.Options` 参数**：

```java
@FeignClient(name = "user-service")
public interface ReportClient {
    @GetMapping("/reports/{id}")
    Report getReport(@PathVariable("id") Long id, Request.Options options);
}
```

该参数被 Contract 识别后不参与 URL 组装，而是作为**本次调用的超时覆盖**传入 `Client.execute`——适合"导出类慢接口临时放宽、其余接口保持默认"的差异化场景，优先于全局配置（§3.3 的优先级表之外的最顶层）。注意它是运行期行为，接口文档与调用方都要约定默认值来源，避免每个调用点各写一套。

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

超时取值优先级汇总（方法参数 > 单客户端 Bean/yml > default）：

```text
调用级: 方法签名里的 Request.Options          (最高)
客户端级: configuration 类的 Options Bean
配置级: yml client.config.<name>.connect/read-timeout
默认:   yml default / Options 默认值 10s/60s    (最低)
```

`<name>` 按 contextId 优先匹配的规则见 `4_configuration.md` §3.1；主容器里的全局 Options Bean 是"客户端没有专属配置时向上找到它"，并非压过专属配置——方向别记反。

# 4. 超时生效过程

```text
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

三个名词要对齐：Apache 的 `socketTimeout`、OkHttp 的 `readTimeout`、JDK 的 `setReadTimeout` 都是"两次数据包之间的最大间隔"，**不是响应总耗时上限**——一个每 3s 吐一点数据的流式响应可以永远不触发读超时。若需要"整次调用的硬上限"，那是熔断器 TimeLimiter（或 Resilience4j）的职责，见 `14_circuit-breaker.md` 与 `4_configuration.md` §8 的叠加关系说明。

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

（此段是简化还原：真实自动配置里 `time-to-live` 与"空闲多久后复用前先校验"的 `validate-after-inactivity` 是两个独立参数，分别落到连接管理器的 TTL 与校验配置上，不是一行换算。）两个参数各司其职：TTL 决定连接"最多活多久"（到龄强制重建），validateAfterInactivity 决定"闲置超过这个时长，借出前先探测死活"——对付中间设备静默断开半连接，后者才是关键项。`max-connections-per-route` 按**目标主机:端口**计，直连 IP 池化好、走域名经网关时容易撞上限，容量规划口径不同（`26_performance-tuning.md`）。

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

注意 OkHttp 池参数的语义是"**空闲**连接上限"而非并发上限：活跃连接不受 `maxIdleConnections` 限制，它只决定闲置回收的水位——和 Apache 的 maxPerRoute"硬上限排队"行为完全不同，从 Apache 迁到 OkHttp 时容量模型要重写。另外一旦自定义了 `OkHttpClient` Bean，其上的 `connectTimeout/readTimeout` 会与 Feign `Options` 并存：Feign 每次调用经 `newBuilder()` 用 Options 覆盖，所以 Bean 里写死的超时实际会被 per-call 值取代，两处不一致时以 Options 侧为准。

## 5.3 HttpURLConnection 注意

默认引擎 `Client.Default` 使用 `HttpURLConnection`，**没有连接池**，每次请求新建连接。**生产环境强烈建议替换为 Apache HttpClient 或 OkHttp**。

准确地说，JDK 自带一个按目标主机的 keep-alive **空闲连接缓存**（`http.maxConnections` 默认仅 5），但它没有借还语义、没有等待队列，并发一高表现为连接反复建拆、TIME_WAIT 堆积；且它没有独立线程做空闲回收，连接死活要到下次用才知道。"无池"是相对 Apache/OkHttp 的工程含义，不是字面零复用。

# 6. 超时与重试的关系

```text
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

`maxAutoRetries` 是 Ribbon 时代的键名，现在的重试预算来自三处：Feign `Retryer`（§2 的 `9_retry.md`）、负载均衡的实例重试、熔断层 Retry——三者相乘才是下游看到的真实放大系数（`13_loadbalancer.md` 有换算表）。最实用的硬约束是一条链式不等式：**本层（重试×超时）总预算 ≤ 调用方预算 × 0.7**，且 connect-timeout 与 read-timeout 分开定——前者是"有没有连上"的故障，短（1-3s）即可判定重试；后者是"对方多久干活"的 SLA，跟着下游 P99 走。

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

## 7.3 保活时长必须与最短的一环对齐

keep-alive 不是客户端单方决定：**网关/LB/云 LB 的空闲断开时间**（常见 60s，Nginx `keepalive_timeout` 默认 75s/60s）通常比连接池 TTL（900s）短得多。链路空闲超过中间设备的断连阈值后，设备静默丢弃半开连接，客户端下次借出这条"僵尸连接"写请求即 `Connection reset`/`no output`，且这类错误只在低峰后的第一个请求出现、重试又成功，极难归因。治理优先级：`validate-after-inactivity`（借出前探测）> 把空闲回收调到**小于**链路上最短断连阈值 > `evictIdleConnections` 定时驱逐。`setKeepAliveStrategy` 返回 30s 意味着服务端还额外享有响应头 `Keep-Alive: timeout=` 协商权，示例写法是无条件 30s，需确认与下游一致。

# 8. 超时问题排查

## 8.1 连接超时排查

```text
检查点：
1. 目标服务端口是否可达 (telnet host port)
2. 防火墙 / 安全组规则
3. 负载均衡器是否正确路由
4. DNS 解析是否正确
5. 连接池是否耗尽（PoolEmptyException / ConnectionPoolTimeoutException）
```

第 5 项和第 1-4 项要分开定性：池耗尽不是网络故障而是**容量/泄漏信号**——表现为 `ConnectionPoolTimeoutException: Timeout waiting for connection`（等待借出超时），伴随堆栈里没有任何 IO 异常。若下游 RT 正常而此类错误增长，查两件事：`maxPerRoute` 是否小于该路由的实际并发、是否有客户端用完不归还（连接被业务线程攥着 sleep——响应体流未读完/未关闭是隐性占用的经典来源，与 `6_encoder-decoder.md` 的"流只读一次"相关）。

## 8.2 读取超时排查

```text
检查点：
1. 目标服务处理时间是否过长
2. 网络延迟是否异常
3. 请求体 / 响应体是否过大
4. 是否存在慢 SQL / 慢缓存 / 死锁
```

补一条定位顺序上的技巧：先用 Feign BASIC 日志的 `(45ms)` 或 APM 分段，把"慢"钉在**发送/等待首包/接收**三段之一——读超时几乎总落在等待首包（下游执行慢），若三段都快但仍超时，才轮到查网络重传与中间件缓冲。

# 9. 最佳实践

| 建议 | 值 | 说明 |
|------|-----|------|
| 连接超时 | 2-5s | 建连是轻量操作，不需要太长 |
| 读取超时 | 5-30s | 取决于下游服务的 SLA |
| 连接池大小 | per-route × 下游实例数 | 每路由 20-50 即可 |
| 连接空闲回收 | 30-60s | 避免长连接被中间网络设备断开 |

再加三条纪律：**任何超时值都要能回答"依据是哪个 SLA/P99"**，拍脑袋的 60s 默认值（Options 缺省）出现在生产 yml 里本身就是故障预案缺失；读超时放宽前先看它是不是在掩盖一个更该修的慢点；每条对外链路把 `超时 × 三层重试` 的乘积写进配置注释，容量评审与故障演练直接引用该行。
