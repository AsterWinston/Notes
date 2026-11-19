# 1. 管道架构

Feign 的请求处理管道是一个**链式处理模型**，从用户方法调用到最终 HTTP 响应经过多个处理阶段：

```text
用户调用 (proxy.method(args))
    │
    ▼
[1] InvocationHandler.invoke()
    │
    ▼
[2] RequestTemplate 构建
    │  Contract 之前解析的 MethodMetadata
    │  方法参数注入到 URL / Header / Query / Body
    │
    ▼
[3] RequestInterceptor 链
    │  添加认证头、TraceId、版本标记等
    │
    ▼
[4] Target.apply() → Request
    │  URL assembled
    │
    ▼
[5] Encoder.encode()
    │  请求体序列化（JSON / XML / Form）
    │
    ▼
[6] Client.execute()
    │  ┌─ FeignBlockingLoadBalancerClient (LB 层)
    │  │   └─ LoadBalancerClient.choose(serviceId)
    │  └─ ApacheHttpClient / OkHttpClient / Client.Default
    │       └─ 实际 socket I/O
    │
    ▼
[7] Response 返回
    │
    ▼
[8] Logger.logAndRebufferResponse()
    │  记录日志 + rebuffer body
    │
    ▼
[9] Decoder.decode()
    │  响应体反序列化
    │
    ▼
[10] ErrorDecoder.decode()
     (仅在非 2xx 时)
    │
    ▼
返回业务对象
```

## 1.1 阶段次序的一处勘误：Encoder 藏在 [4] 内部

上图把 [5] Encoder 画在 [4] target.apply **之后**，真实次序相反：**序列化发生在 Request 物化时**，即 `target.apply(template)` → `template.request()` 内部触发 `DeferredEncoder` 调用 encoder（§2 源码第 4 步只把"对象+类型"挂进模板，没序列化）。图上 [3]→[5] 的相对关系是对的（拦截器先于序列化），只是 [5] 的框应嵌在 [4] 里。这个修正有两处实际收益：解释了拦截器为什么"改 header 便宜、改 body 费劲"（它执行时 body 还是待序列化的对象）；也解释了 §8 时间线里 T7 晚于 T6 的原图标注同样需要按此更正。

## 1.2 管道的两态分界

整条线以 [4]/[6] 为界分成两种世界：**模板态**（[2]-[4]，可变对象、未定址、body 还是 Java 对象，出错表现为畸形 URL/丢失参数）与**报文态**（[6] 之后，不可变 Request/Response、字节流、只读一次的 body，出错表现为超时/解码错/状态码）。拦截器与 Contract 的 bug 永远在前半段，Encoder/Decoder/Client/压缩/日志 rebuffer 的坑全在后半段——`22_source-contract-methodhandler.md` §7.1 找栈帧定位法，先判异常属于哪半段，可再砍一半搜索空间。

# 2. RequestTemplate 构建细节

```java
// SynchronousMethodHandler 中：
final class Factory implements RequestTemplate.Factory {

    private final MethodMetadata metadata;
    private final Target<?> target;

    @Override
    public RequestTemplate create(Object[] argv) {
        RequestTemplate mutable = RequestTemplate.from(
                metadata.template());   // 从 MethodMetadata 拷贝模板

        // ====== 1. URL 变量替换 ======
        // indexToName: {0 → ["id"], 2 → ["name"]}
        for (Map.Entry<Integer, Collection<String>> entry :
                metadata.indexToName().entrySet()) {
            int paramIndex = entry.getKey();
            Object value = argv[paramIndex];
            if (value == null) continue;

            for (String name : entry.getValue()) {
                // 根据 name 类型决定替换位置
                if (isPathVariable(name)) {
                    // 替换 URL 中的 {name}
                    mutable.resolve(name, value.toString());
                } else if (isQueryParam(name)) {
                    // 用 Param.Expander 展开值并拼接到 query
                    Param.Expander expander = metadata.indexToExpander()
                            .get(paramIndex);
                    if (expander != null) {
                        mutable.query(name, expander.expand(value));
                    } else {
                        // 简单值直接拼接，数组/集合展开
                        mutable.query(name, value);
                    }
                } else if (isHeaderParam(name)) {
                    mutable.header(name, value.toString());
                }
            }
        }

        // ====== 2. @SpringQueryMap 展开 ======
        if (metadata.queryMapIndex() != null) {
            Object queryMap = argv[metadata.queryMapIndex()];
            if (queryMap != null) {
                // 反射读取 getter，将每个属性作为 query param
                addQueryMap(mutable, queryMap);
            }
        }

        // ====== 3. @HeaderMap 展开 ======
        if (metadata.headerMapIndex() != null) {
            Object headerMap = argv[metadata.headerMapIndex()];
            if (headerMap instanceof Map) {
                ((Map<?, ?>) headerMap).forEach((k, v) ->
                    mutable.header(k.toString(), v.toString()));
            }
        }

        // ====== 4. @RequestBody 编码 ======
        if (metadata.bodyIndex() != null) {
            Object body = argv[metadata.bodyIndex()];
            // 此处只标记 body 存在，实际的序列化在 Encoder 阶段
            mutable.body(body, metadata.bodyType());
        }

        return mutable;
    }
}
```

## 2.1 null 的三种命运，全在参数路由表里发生

`if (value == null) continue;` 一行让**null 参数被静默跳过**，但三类去向的后果完全不同：query/header 为 null → 参数干脆消失（下游拿到"没传"而非"传了空"，`@RequestParam(required=false)` 语义下才安全）；**path 变量为 null → `{id}` 占位符留在模板里**，`request()` 物化时抛 `IllegalStateException: Missing variable {id}`——报错文案指向模板不像参数问题，是高频迷惑源；body 为 null → 无 body 请求（`6_encoder-decoder.md` §2）。写客户端方法时对 path 参数做前置判空，永远不要把"可能为 null"传进 URL 模板。

## 2.2 Expander：日期与自定义类型的序列化出口

`@RequestParam`/`@PathVariable` 注解的 `valueFormatter` 属性绑定的就是 `Param.Expander`：默认只有 `Object.toString()`（日期会变成 `Wed Oct 01 08:00:00 CST 2025` 这类 Locale 相关字符串，跨服务解析必炸）。正解是注册格式化器：`@RequestParam(value = "since", valueFormatter = DateFormatter.class)` 配一个实现 `Param.Expander`/Spring 侧 `DateFormat` 的类，或在 Contract 层用 `SpringQueryMap` 的对象路径（走 HttpMessageConverter 风格的转换，但集合/日期规则另一套）。另外 `queryMap`/`headerMap` 的展开只读**一层 getter**：嵌套对象字段不会变成 `filter.name=x`，Map 的 value 也不二次展开——复杂条件对象进 query 前自己摊平。

# 3. Encoder 编码阶段

```java
// 在 executeAndDecode() 中，RequestTemplate 转为 Request 之前：
// Encoder 实际上是在 targetRequest() → target.apply(template) 内部被调用的

// Feign 内部流程（简化）：
RequestTemplate template = buildTemplateFromArgs.create(args);
// ...

// Encoder 对 RequestTemplate 进行编码（注入 serialized body）
if (template.body() != null) {
    encoder.encode(template.body(), template.bodyType(), template);
    // 此处的 encoder 可能是：
    // - SpringEncoder → 使用 HttpMessageConverters
    // - Encoder.Default → toString().getBytes(UTF-8)
    // - FormEncoder → multipart/form-data 编码
}

// 然后转为不可变 Request
Request request = target.apply(template);
```

## 3.1 DeferredEncoder：一次挂载、每尝试执行

§2 的 `mutable.body(body, bodyType)` 实际写入的是**惰性体**：模板记住"对象 + 类型 + 注入的 encoder"（`RequestTemplate.DeferredEncoder`），直到 `request()` 被调用才真正 `encode`。两个推论与本篇其他章节闭环：拦截器执行时（`targetRequest` 里先 apply 后 request）body 尚未序列化——签名场景的时序约束（`7_interceptor.md` §2）；**每次重试的 `targetRequest` 都会重新物化 Request，即 Encoder 每次尝试重跑一遍**（`7_interceptor.md` §8.1"每次尝试 = 拦截器→编码→发送"的确切出处）。"body 序列化一次"的直觉是错的：序列化成本 ×尝试次数，大 body 高重试配置会明显放大 CPU。

# 4. URL 解析与构建

```java
// 假设 @FeignClient(name = "user-service")
// 方法定义：@GetMapping("/users/{id}")

// 1. MethodMetadata.template().url() = "/users/{id}"
// 2. buildTemplateFromArgs → resolve("id", "1") → "/users/1"
// 3. target.apply(template):
//    - url = "http://user-service"
//    - 拼接 → "http://user-service/users/1"
// 4. FeignBlockingLoadBalancerClient:
//    - serviceId = "user-service"
//    - 通过 LB 选择实例 → 192.168.1.10:8080
//    - 替换 → "http://192.168.1.10:8080/users/1"

// 最终 HTTP 请求：
// GET http://192.168.1.10:8080/users/1 HTTP/1.1
```

## 4.1 三段拼接的全部规则

URL 由三处合成：`@FeignClient` 的 `url`/服务名（进 `HardCodedTarget`）、`path` 属性（注册期就拼进模板前缀）、方法注解路径。易错点集中在斜杠与占位：`@GetMapping("users/{id}")`（无前导 `/`）与 `path="/api/v2/"`（有尾 `/`）组合会拼出 `/api/v2users/...` 之类的粘连——**模板路径统一前导 `/`、path 统一无前导或有导但要实测一次**，Feign 的 `TemplateUrl` 对重复 `/` 会归一但不会替漏掉的斜杠补位；变量含特殊字符时 `resolve` 默认**不做 URL 编码**（值里带 `?`/`&`/中文会破坏结构），需要编码的值用 `@SpringQueryMap` 走 query 通道或自定义 Expander；服务名当 host 的合法性要求小写/无点开头（`user_service` 下划线部分注册中心可用但 DNS 直连不可用），混用 `url` 与服务名的组合模式见 `13_loadbalancer.md` §8。

# 5. Client.execute() 的请求包装链

```java
// 从外到内的 Client 嵌套（装饰器模式）：

// 最外层：熔断
// FeignCircuitBreakerInvocationHandler
//   └── CircuitBreaker.wrap(client)

// 中间层：负载均衡
// FeignBlockingLoadBalancerClient
//   ├── LoadBalancerClient.choose(serviceId)
//   └── delegate.execute(newRequest, options)

// 最内层：HTTP 引擎
// ApacheHttpClient / OkHttpClient / Client.Default
//   └── 打开 Socket → 发送字节 → 接收字节 → 构建 Response
```

## 5.1 勘误：熔断不在 Client 链上，在方法层外

"最外层熔断"这个画法不精确：`CircuitBreaker` 装饰的**不是 Client**，而是整个 `SynchronousMethodHandler.invoke`（双层代理结构见 `21_source-dynamic-proxy.md` §8.1）。修正后的嵌套从外到内是：

```text
熔断/TimeLimiter/Bulkhead/fallback   ← 方法调用粒度
  └ Retryer 循环                     ← 单次 HTTP 尝试的容错
      └ FeignBlockingLoadBalancerClient  ← 每次尝试重新选址
          └ HTTP 引擎                ← 连接与字节
```

这个纠正的实用价值：统计口径上熔断看到的是"重试后的最终结果"（3 次尝试失败 1 次不进熔断统计）；TimeLimiter 的预算要覆盖整个 Retryer 循环（`9_retry.md` §9.1 的乘积不等式）；而"换实例重试"发生在熔断内侧（§10 的 `24` 篇主线）。Client 接口上真正的装饰器只有 LB 一层。

# 6. Response 反序列化

```java
// executeAndDecode() 中：
if (response.status() >= 200 && response.status() < 300) {
    if (response.status() == 204 || response.body() == null) {
        return null;    // 空响应
    }

    // Decoder 解码
    Object result = decoder.decode(response, metadata.returnType());

    // Decoder 可能是：
    // - SpringDecoder → HttpMessageConverters
    // - JacksonDecoder → ObjectMapper.readValue()
    // - 自定义 Decoder

    return result;
}
```

**注意类型信息传递**：
```java
// metadata.returnType() 保留了完整泛型信息
// 例如方法：List<User> listUsers()
// returnType = ParameterizedTypeImpl{List<User>}
// decoder 可以利用此信息正确反序列化
```

这段的完整分支表（204/空 body、decode404、ErrorDecoder 路由）在 `22_source-contract-methodhandler.md` §4 源码已展开，类型信息链（`getGenericReturnType` → `Types.resolveReturnType` → Decoder 的 `Type` 参数）的"泛型不丢"结论见 `20_response-mapping.md` §1.1——本节图上补两个盲区：`return null` 分支对**基本类型返回值**直接 NPE（拆箱在调用侧发生，`20_response-mapping.md` §6 矩阵），以及 `Task<Void>`/`void` 方法同样走这条 null 路径而不是"跳过 Decoder"，所以自定义 Decoder 必须先判 `type == void.class`（`6_encoder-decoder.md` §7.1）。

# 7. 异常处理管线

```java
// executeAndDecode() 中：
if (response.status() == 404 && decode404) {
    // 404 当正常响应处理
    return decoder.decode(response, metadata.returnType());
}

// 其他错误 → ErrorDecoder
try {
    throw errorDecoder.decode(configKey, response);
} catch (FeignException e) {
    // e.status() == 503 && e instanceof RetryableException?
    // 如果是 RetryableException，外层 while(true) 会重试
    throw e;
}
```

ErrorDecoder 与重试的握手只有一条线：**产出的异常 `instanceof RetryableException` 才被外层循环接住**。示意里 503 分支的注释要落到实现上就是 `8_error-handling.md` §6 的 `RetryableErrorDecoder`——而 catch 块本身只是注释性写法（真实代码直接 `throw`，是否重试由外层判定）。补一条容易漏的闭环：ErrorDecoder 抛**非** FeignException 的自定义异常（BusinessException）时，重试循环的 `catch (RetryableException)` 接不住，异常**穿透整条管线**直达熔断层计失败——这是"业务翻译异常参与熔断统计"（`14_circuit-breaker.md` §3.2）的路径依据。

# 8. 请求/响应生命周期（完整时间线）

```text
T1: userClient.getUser(1L)                    ← 用户代码调用
T2: FeignInvocationHandler.invoke(...)          ← JDK 代理拦截
T3: SynchronousMethodHandler.invoke([1L])       ← 方法处理器
T4: buildTemplateFromArgs.create([1L])          ← 构建 RequestTemplate
T5: RequestInterceptor.apply(template) × N      ← 拦截器链
T6: target.apply(template) → Request            ← 生成不可变 Request
T7: Encoder.encode(body, bodyType, template)    ← 请求体编码
T8: logger.logRequest(...)                      ← 日志：请求
T9: LoadBalancer.choose(serviceId)             ← 服务实例选择
T10: TCP connect(192.168.1.10:8080)             ← 建立连接
T11: Send HTTP request                          ← 发送字节
T12: Wait for response                          ← 等待响应
T13: Receive HTTP response                      ← 接收字节
T14: logger.logAndRebufferResponse(...)          ← 日志：响应
T15: Decoder.decode(response, returnType)       ← 响应体反序列化
T16: return user                                ← 返回业务对象

总耗时 = T16 - T1
网络耗时 = T13 - T10
服务端处理耗时 = T12 - T11
```

## 8.1 按 §1.1/§5.1 更正后的口径与归因用法

T6/T7 次序按 §1.1 修正（编码在物化内），且 T4-T7 在重试时整体重复一轮。三条计时线段的**测量者各不相同**，这是口径混淆的总根源：Feign 日志的 `(45ms)` = `client.execute` 全程（T9-T13，含 LB 选择）——**不含 T4-T7 与 T15**（`10_logging.md` §6"日志 RT 比 APM 低"的答案）；APM 客户端 span 通常从 T2 起画到 T16（含编解码）；熔断层 TimeLimiter 计时含全部重试轮次。归因套路：日志 RT 高而 APM 分段显示发送前就慢 → T5-T7（拦截器重活/序列化大 body）；日志 RT 高在 `(45ms)` 本身 → T9-T13，再按 connect/wait-first-byte/receive 三段细分（`11_timeout-connection.md` §8.2）；都正常但业务方说慢 → T15 解码大响应（FULL 日志的 rebuffer 也在 T14 加一笔）。

# 9. Client 接口的职责边界

```java
public interface Client {
    Response execute(Request request, Options options) throws IOException;
}

// Client 的职责：
// 1. 打开到目标主机的连接
// 2. 发送 Request（headers + body）
// 3. 读取 Response（status + headers + body）
// 4. 返回不可变的 feign.Response 对象

// Client 不负责：
// - URL 解析（Target 负责）
// - 编码/解码（Encoder/Decoder 负责）
// - 重试（Retryer 负责）
// - 日志（Logger 负责）
// - 负载均衡（由 FeignBlockingLoadBalancerClient 装饰）
```

职责表里"URL 解析归 Target"对 **Spring Cloud 语境要打折**：目标地址最终由 LB Client 改写（§5 中间层），`Target.apply` 只保证模板拿到一个可解析的 host——这个"两处都碰 URL"的分工正是 `url=` 直连时服务名解析失效的机制解释（`13_loadbalancer.md` §7 的 host 即 serviceId）。`throws IOException` 是 Client 唯一的受检异常承诺：LB 选址失败抛的是 `IllegalStateException`（`No instances available`，§2 of `24` 篇），它不合规约却照样穿过 `executeAndDecode` 的 `catch (IOException)`——于是不进重试、直达熔断，异常类型选择决定了故障走哪条治理通道，自定义 Client 时值得记住这一点。

# 10. 可观测性注入点

在管线中有几个理想的**可观测性注入点**：

```java
// 1. RequestInterceptor → 注入 TraceId
@Component
public class TracingInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        Span span = tracer.nextSpan();
        template.header("X-B3-TraceId", span.context().traceId());
        template.header("X-B3-SpanId", span.context().spanId());
    }
}

// 2. Client 装饰器 → 记录耗时和状态
public class MetricsClient implements Client {
    private final Client delegate;
    private final MeterRegistry meterRegistry;

    @Override
    public Response execute(Request request, Options options) throws IOException {
        long start = System.currentTimeMillis();
        try {
            Response response = delegate.execute(request, options);
            meterRegistry.counter("feign.call.success", "service", extractService(request)).increment();
            meterRegistry.timer("feign.call.latency", "service", extractService(request))
                    .record(System.currentTimeMillis() - start, TimeUnit.MILLISECONDS);
            return response;
        } catch (Exception e) {
            meterRegistry.counter("feign.call.failure", "service", extractService(request)).increment();
            throw e;
        }
    }
}

// 3. ErrorDecoder → 区分业务异常和系统异常
// 4. Retryer → 记录重试次数
```

## 10.1 手搓指标之前：框架已给的两套

四个注入点都成立，但**指标类需求先确认现成的**：`feign-micrometer` 的 `MicrometerCapability`/Spring Cloud OpenFeign 4.x 默认开启的 `spring.cloud.openfeign.metrics.*`（按 configKey/method/uri 模板打标签的 Timer/Counter），以及 Micrometer Tracing 自动完成的 [1] 注入（`7_interceptor.md` §4 的"不要手写"同样适用于指标）。手搓 `MetricsClient` 的正当理由只剩两种：口径要**每次尝试**而非每次调用（装饰 Client 在重试内侧，天然统计尝试数——与熔断层"只见最终结果"正好互补，两个口径各回答"下游收了多少请求"与"业务失败率"）；或要把指标挂到非 Micrometer 体系。位置选择经验：**统计"尝试"放 Client 层，统计"结果"放熔断层，改元信息放拦截器层，翻译错误放 ErrorDecoder**——四个注入点不是四个方案，是一条流水线上的四个探针。

## 10.2 与本系列源码篇的收束关系

本篇把 [2] 的细节（参数路由）交还给 `22` 的 Contract 输出表，把 [6] 的内脏（选址与 URL 改写）交还给 `24` 的装饰器链，把 [1]/双层代理的出生过程交还给 `21` 的装配链。三篇连读顺序建议 `21 → 22 → 24 → 23`：先知道代理怎么造出来、元数据里装了什么、LB 在哪个对象里改 URL，再回看这条时间线，图上每一帧都能说出"这是哪个类的哪一行"。
