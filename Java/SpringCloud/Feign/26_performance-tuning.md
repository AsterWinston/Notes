
# 1. 调优全景图

```
Feign 性能调优涉及 5 个层面：

┌──────────────────────────────────────────┐
│ 1. 应用层    │ Feign 配置、接口设计、重试策略   │
├──────────────────────────────────────────┤
│ 2. HTTP 层   │ 引擎选择、连接池、超时、压缩     │
├──────────────────────────────────────────┤
│ 3. 序列化层   │ JSON 框架选择、编解码优化       │
├──────────────────────────────────────────┤
│ 4. 网络层    │ Keep-Alive、连接复用、SSL 缓存  │
├──────────────────────────────────────────┤
│ 5. JVM 层    │ GC、线程池、堆内存              │
└──────────────────────────────────────────┘
```

# 2. HTTP 引擎选择

```yaml
# 推荐：OkHttp
spring:
  cloud:
    openfeign:
      okhttp:
        enabled: true
      httpclient:
        enabled: false    # 确保仅一个引擎生效
```

**性能基准参考**（1000 次请求，并发 50）：

| 引擎 | QPS | P99 延迟 | 内存占用 |
|------|-----|----------|----------|
| HttpURLConnection | ~450 | 25ms | 高（频繁创建连接） |
| Apache HttpClient | ~1200 | 10ms | 中 |
| OkHttp | ~1400 | 8ms | 低 |

# 3. 连接池调优

## 3.1 OkHttp 连接池

```java
@Bean
public OkHttpClient okHttpClient() {
    return new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(
                    20,                              // 空闲连接数
                    10, TimeUnit.MINUTES            // 空闲存活时间
            ))
            .build();
}
```

**调优原则**：
- `maxIdleConnections` ≥ 下游服务实例数 × 每实例并发数
- `keepAliveDuration` 略大于服务端 Keep-Alive 超时

## 3.2 Apache HttpClient 连接池

```java
@Bean
public HttpClient apacheHttpClient() {
    PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(300);
    cm.setDefaultMaxPerRoute(30);
    cm.setValidateAfterInactivity(3000);

    return HttpClientBuilder.create()
            .setConnectionManager(cm)
            .evictIdleConnections(30, TimeUnit.SECONDS)  // 主动清理
            .build();
}
```

**关键参数**：

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `maxTotal` | 200-500 | 取决于下游服务数量和并发 |
| `defaultMaxPerRoute` | 20-50 | 单个目标的连接数上限 |
| `validateAfterInactivity` | 3000ms | 空闲超过此值后验证连接有效性 |
| `evictIdleConnections` | 30s | 清理空闲连接周期 |

# 4. 超时配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 2000       # 连接超时：不宜过大（快速失败）
            read-timeout: 5000          # 读取超时：根据 SLA 设定
```

**超时建议**：

| 服务类型 | connectTimeout | readTimeout |
|----------|---------------|-------------|
| 内部微服务（RPC） | 500-2000ms | 2000-5000ms |
| 外部 API 调用 | 3000-5000ms | 10000-30000ms |
| 文件上传/下载 | 3000ms | 60000-300000ms |

# 5. 序列化优化

## 5.1 Jackson 优化

```java
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    // 1. 避免日期序列化为 timestamp
    mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    // 2. 忽略未知属性（避免 Abort）
    mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
    // 3. 使用 Afterburner 模块（大幅提升序列化性能）
    mapper.registerModule(new AfterburnerModule());
    return mapper;
}
```

## 5.2 编译时序列化（Protobuf）

对于高性能场景，可以考虑 Protobuf 替代 JSON：

```java
@FeignClient(name = "user-service", configuration = ProtobufConfig.class)
public interface UserClient {

    @PostMapping(value = "/api/users",
                 consumes = "application/x-protobuf",
                 produces = "application/x-protobuf")
    UserProto.User getUser(@RequestBody UserProto.UserRequest request);
}
```

```java
public class ProtobufConfig {
    @Bean
    public Encoder feignEncoder() {
        return new ProtobufEncoder();
    }

    @Bean
    public Decoder feignDecoder() {
        return new ProtobufDecoder();
    }
}
```

**序列化性能对比**：

| 格式 | 速度 | 体积 | 可读性 |
|------|------|------|--------|
| JSON (Jackson) | 基准 | 基准 | 好 |
| JSON (Afterburner) | +30% | 相同 | 好 |
| Protobuf | +200% | -50% | 差 |
| MessagePack | +150% | -30% | 差 |

# 6. 请求压缩

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          mime-types: application/json
          min-request-size: 2048     # 低于 2KB 不压缩
        response:
          enabled: true
```

**压缩收益评估**：
- JSON 文本数据压缩比约 5:1 ~ 10:1
- 内网延迟 < 1ms 时，压缩的 CPU 开销可能高于收益
- 跨机房 / 跨区域调用强烈建议开启

# 7. 重试策略调优

```java
// 避免无效重试堆积
@Bean
public Retryer feignRetryer() {
    return new Retryer.Default(
            200,    // period：初始间隔 200ms（不宜过大）
            2000,   // maxPeriod：最大间隔 2s
            3       // maxAttempts：最多 3 次
    );
}
```

**重试风险**：

```
第 1 次：0ms
第 2 次：200ms
第 3 次：400ms
总耗时 ≈ max(readTimeout, totalRetryDelay) ≈ 5600ms (假设 readTimeout=5000ms)
```

建议：
- `maxAttempts` 不要超过 3
- 仅在 GET 请求上重试
- 配合熔断器限制最大并发

# 8. 熔断器参数

```yaml
resilience4j:
  timelimiter:
    configs:
      default:
        timeout-duration: 3s         # 超时时间（应 < readTimeout）
        cancel-running-future: true  # 超时后取消任务

  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20      # 窗口大小（不宜过小）
        minimum-number-of-calls: 10  # 最小调用数后开始统计
        failure-rate-threshold: 50   # 失败率阈值
        wait-duration-in-open-state: 15s

  bulkhead:
    configs:
      default:
        max-concurrent-calls: 20     # 最大并发数
        max-wait-duration: 500ms    # 等待超时
```

# 9. GC 与内存

```java
// 大量请求时，Feign 的解码会产生大量临时对象
// JVM 参数建议：
// -XX:+UseG1GC
// -XX:MaxGCPauseMillis=200
// -XX:InitiatingHeapOccupancyPercent=45

// 监控指标：
// feign_request_bytes_sum
// feign_response_bytes_sum
// 如单次请求 100KB，QPS 1000 → 100MB/s 的临时对象创建
```

# 10. 监控关键指标

```java
// 自定义 Client 装饰器收集指标
public class MonitoredClient implements Client {
    private final Client delegate;
    private final MeterRegistry registry;

    @Override
    public Response execute(Request request, Options options) throws IOException {
        long start = System.nanoTime();
        try {
            Response response = delegate.execute(request, options);
            recordSuccess(request, System.nanoTime() - start);
            return response;
        } catch (IOException e) {
            recordFailure(request, System.nanoTime() - start, e);
            throw e;
        }
    }

    private void recordSuccess(Request request, long nanos) {
        Timer.builder("feign.client.requests")
                .tag("service", extractService(request))
                .tag("method", request.httpMethod().name())
                .register(registry)
                .record(nanos, TimeUnit.NANOSECONDS);
    }
}
```

# 11. 调优清单

| 优化项 | 优先级 | 预期收益 | 成本 |
|--------|--------|----------|------|
| 切换 HTTP 引擎 (OkHttp) | 高 | 30-50% 吞吐提升 | 低（一行配置） |
| 连接池参数调优 | 高 | 减少建连开销 | 低 |
| 超时合理设置 | 高 | 避免线程堆积 | 低 |
| 请求/响应压缩 | 中 | 50-80% 带宽节省 | 低（CPU 略增） |
| JSON 序列化优化 | 中 | 10-30% 序列化提升 | 低 |
| 避免 N+1 调用 | 高 | 数倍延迟降低 | 中（需改接口） |
| 重试策略调优 | 中 | 提升可靠性 | 低 |
| 熔断器参数调优 | 中 | 防止雪崩 | 低 |
| Protobuf 序列化 | 低 | 50-200% 性能提升 | 高（全链路改造） |
| JVM GC 调优 | 低 | 5-15% 延迟降低 | 中 |

# 12. 压测建议

```bash
# 使用 wrk 或 ab 对下游服务进行压测
wrk -t4 -c100 -d30s http://localhost:8080/api/users/1

# 使用 JMeter / Gatling 进行全链路压测
# 观察指标：
#  - feign.client.requests P99 延迟
#  - resilience4j_circuitbreaker_state
#  - jvm_gc_pause_seconds
#  - http_connections_active
```
