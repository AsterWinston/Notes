
# 1. 管道架构

Feign 的请求处理管道是一个**链式处理模型**，从用户方法调用到最终 HTTP 响应经过多个处理阶段：

```
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

# 8. 请求/响应生命周期（完整时间线）

```
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
