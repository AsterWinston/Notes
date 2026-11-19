# 1. 调优全景图

```text
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

## 1.1 用法：先定位，再选层

这张图是**坐标系不是清单**——直接照着逐层调，80% 的力气会花在没病的器官上。正确的入场姿势是 `23_source-request-pipeline.md` §8.1 的分段归因：用 Feign 日志 `(45ms)`、APM span、下游服务自身 RT 三个口径对齐，把耗时钉进"发送前准备 / 网络 / 下游执行 / 解码"某一段，再进对应层。**经验分布**：多数"Feign 慢"的真凶在第 1 层（N+1、循环调用、串行可并行——`25` §1.3/§5.1）与下游服务本身；第 2、3 层的参数级优化通常只有 10-30% 的边际收益。优先级永远是：**改接口设计 > 改调用模式 > 改引擎与池 > 换序列化 > 调 JVM**。

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

数字只看相对排序、绝对值不可移植的读法见 `19_http-client-engines.md` §6；"确保仅一个引擎生效"的双开关写法是正确纪律（谁优先由装配顺序决定，别赌——`19` §8.1）。选型层面的真正分水岭在**池模型语义**：要"每路由硬上限 + 排队 + 排队超时"的背压保护选 Apache（并配 connectionRequestTimeout，`19` §3.4），要"轻快复用无硬顶"选 OkHttp（并发保护改由 Bulkhead 承担，`19` §4.4）。

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

两条都要加限定再执行：`maxIdleConnections` 是**空闲水位不是并发上限**（活跃连接不受它限），按"实例数×每实例并发"放大它保护的其实是"空闲期后重建连的成本"——数值意义上更多是避免复用断连；`keepAliveDuration` 与"略大于服务端"方向正确但要**小于链路上最短的中间设备断连阈值**（网关/LB 的 idle timeout，`11_timeout-connection.md` §7.3），"大于服务端、小于最短设备"两头夹。另注意本 Bean 的超时字段会被 per-call Options 覆盖（`19` §4.4），池参数则完整生效。

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

表外必须补的第 5 个参数是 `connectionRequestTimeout`（本示例没配，默认无限等待——池满时业务线程静默排队抽干 Tomcat 线程池，`19` §3.4 的雪崩形态），生产建议 1-3s 快速失败。route 口径的坑顺带记住：经网关/域名调用时几十个下游服务可能共用 1 个 route 名额，直连 IP 时又是另一算法——`maxPerRoute` 换算前先在故障日志里确认 `ConnectionPoolTimeoutException` 报的是哪个 route。

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

"根据 SLA 设定"的操作定义：read-timeout ≈ 下游 **P99 × 1.5~2**（吸收抖动又不至于吊死线程），不是 P50、更不是拍的整数；connect-timeout 独立定短（1-2s 判得出网络级故障，`11` §6 的分工）。表里的文件行属于"知道有这档，但优先按 `18` §10.5 不走 Feign"。

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

三个修正必要。**定义裸 `ObjectMapper` Bean 是全局改动**：它会顶掉 Boot 自动装配的定制版，Spring MVC 的响应序列化、Actuator、以及 Feign 的 SpringEncoder（都吃 `HttpMessageConverters` 持有的 mapper）一起换血——Boot 工程应改在 `Jackson2ObjectMapperBuilderCustomizer` 上叠加配置而不是造新 Bean。Afterburner（字节码生成器注入）对 Jackson 2.10+ **无效且已停止维护**，等效提速用 `jackson-module-blackbird`；Jackson 3 则内置。性能收益也要有预期管理：序列化通常只占调用 RT 的个位数百分比，除非报文大（>100KB）或 QPS 极高，这层优化在 §1.1 优先级表里排倒数。

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

换表的现代版：Afterburner 行换成 Blackbird（幅度类似）。换格式的**系统成本**在表外：proto 文件与 IDL 的版本管理、两端同步发布、`curl` 排障能力清零（可读性"差"在生产意味着事故处理时长翻倍）、日志/监控里的报文全部二进制化（FULL 日志、ES 采集、网关审计都得适配）、第三方接入成本。它的正当场景是**高频小报文的核心 RPC**（QPS 万级、包体 <5KB、团队有 gRPC/protobuf 资产）；一般业务微服务间 JSON 的运维红利远大于 2 倍序列化性能。若确实要上，`21_logging_monitoring.md` 那类观测配套先落地。

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

收益公式与逐跳纪律在 `17_compression.md` §8.1/§10：决策变量是"带宽 × 报文大小"的乘积；开启前确认引擎配套（OkHttp 请求方向不代压 body、显式 Accept-Encoding 会关掉它的透明解压）与提供方的请求解压能力。作为**性能手段**它专治"跨域大报文"这一种病，别把它当通用开关。

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

```text
第 1 次：0ms
第 2 次：200ms
第 3 次：400ms
总耗时 ≈ max(readTimeout, totalRetryDelay) ≈ 5600ms (假设 readTimeout=5000ms)
```

【勘误】上面这个算式把"退避间隔之和"当成了主线，**漏乘了每次尝试各自的超时**：三次尝试每次都等满 read-timeout 时，最坏总耗时 ≈ `3 × 5000ms(读超时) + (200 + 400ms)(退避) = 15.6s`，是原数值的近 3 倍——重试预算永远按"尝试数 × 单次超时"上界估（`9_retry.md` §9.1 的不等式在这里从"建议"变成"算术"）。这组 `200/2000/3` 参数成立的前提也因此要说全：单次调用预算 ≥ 16s 的离线/任务型链路才配得起 3 次读超时重试；同步 Web 链路若预算只有 5s，`maxAttempts=3` 应只针对 **connect 类失败**（1-2s 判负）而非 read 超时，或者干脆把重试交给 LB 层换实例语义。

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

参数之外对**性能**真正起作用的是三件：`slow-call-rate`+`slow-call-duration-threshold` 没配——"下游没报错但变慢"的故障只会拖线程不会开熔断，慢调用维度是响应时间劣化的自动闸门（`14_circuit-breaker.md` §3.1）；`sliding-window-type: TIME_BASED` 在高 QPS 下比 COUNT_BASED 语义更稳（窗口不随流量忽大忽小）；`cancel-running-future` 在同步链路里只是**尽力**取消——TimeLimiter 超时后底层 HTTP 仍占着连接直到 read-timeout（`14` §7.1），"3s 超时"并不等于 3s 归还容量，容量账要按两段的较大者估。Bulkhead 的 20 并发按利特尔法则（QPS×RT）核算过才写进配置，不是默认抄值。

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

100MB/s 的分配率对 G1 来说尚可，但三个乘数决定它会不会变成事故：**重试**（每尝试重跑序列化，分配翻倍——`23` §3.1）、**FULL 日志 rebuffer**（响应体全量多一份堆拷贝——`10` §6）、**大响应**（byte[] 路径整包进堆）。所以 JVM 之前的免费优化是把这三个乘数关掉；堆内对象分配热点要用分配剖析（async-profiler alloc 模式）确认是 Jackson char[]/byte[] 再谈调参。`InitiatingHeapOccupancyPercent=45` 这类参数的合理性与 JDK 版本强相关（JDK 17+ 的 G1 自适应IHOP 基本不需要手设），照抄互联网参数本身就是一种性能风险。线程维度同样属于本层：Feign 同步调用占的是**发起线程**，重试睡眠也在其上（`9_retry.md` §2.1）——池大小按"入口 QPS × 链路 RT"核算，调 GC 之前先把线程池账算平。

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

装饰 Client 的口径是**每次尝试**（重试内侧，`23` §10.1：回答"下游实际收到多少"），但先确认标准件：Spring Cloud OpenFeign 自带的 Micrometer 指标（Timer/Counter 按 configKey 维度）+ 熔断层指标（按最终结果维度，`14` §12）已经覆盖绝大多数看板，手搓版只在需要"尝试 vs 调用"双口径对照时补 Client 层。示例实现的两处工程化欠账：`Timer.builder().register()` 每次调用都查/建（应缓存 Meter 引用，高频下注册表查找本身是开销）；`extractService` 从 URL 截 host 在直连场景会得到真实域名——标签基数要按服务名归一，否则 Prometheus 时间线爆炸。

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

清单的"优先级"列按 §1.1 的经验分布成立，但**预期收益列的数字只在"该层确实有病"时才存在**：一个已经用 OkHttp + 池化良好的系统，"切换引擎"收益是 0；N+1 的行永远应该先于所有参数行被检查。建议把表头改造成决策格式使用时加两列——"怎么确认有这病"（指标名：建连频率/池等待/N+1 看下游 QPS 与入口 QPS 比值/序列化耗时占比）与"验证治好"（同一指标回归），它就从愿望清单变成了诊断流程。

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

wrk 直打下游测的是**服务端裸能力**，与"经 Feign 调用"之间隔着客户端全部变量——它用于建立下游容量基线（后续所有池/并发参数以它为参照），不能用来评估 Feign 调优效果。评客户端要压**经过调用方的入口**，观察清单补三项 Feign 特有信号：`ConnectionPoolTimeoutException` 计数（池上限是否真够用）、`RetryableException`/重试放大率（入口 QPS × 下游 QPS 的比值，验证 §7 预算）、下游连接数与 TIME_WAIT（§3.1/§3.2 池参数与 keep-alive 策略的体检报告）。方法学陷阱三条在 `19` §6：预热、压测端自身瓶颈甄别、按真实报文尺寸分布建场景——**调优类改动一次只动一个变量并留存同场景对照数据**，否则永远无法知道收益属于哪一行清单。

# 13. 调优流程（收束）

```text
1. 定位   日志/APM/下游 RT 三口径对齐 → 钉住耗时所属段（23 §8.1）
2. 设计层 接口粒度与调用模式检查（25 §1.3/§5）      ← 收益最大
3. HTTP层 引擎唯一启用 + 池参数 + 超时/重试预算（本文 §2-§4、§7）
4. 治理层 熔断/慢调用率/Bulkhead 与统计口径配套（§8、14）
5. 编码层 压缩按病开（§6、17）、序列化最后（§5、§9）
6. 验证   同场景压测对照（§12），一次一个变量
7. 固化   结论写进配置注释与 25 §9 检查清单
```

流程里最重要的其实是第 1 步和第 6 步：**没有归因的调优是抽奖，没有对照的调优是玄学**。至此 Feign 系列收官——参数与机制的每一条都能沿本系列交叉引用回溯到源码篇（21-24）的对应帧，配置改动的副作用沿 `4_configuration.md` 的作用域规则收敛。
