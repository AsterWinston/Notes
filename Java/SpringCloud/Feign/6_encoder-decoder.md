# 1. 概述

## 1.1 编解码是 Feign 与 Spring 生态最厚的耦合点

`Encoder` 和 `Decoder` 是 Feign 请求/响应**编解码**的核心接口，负责 Java 对象与 HTTP 报文之间的相互转换。

```text
请求方向： Java Object  ──[Encoder]──→  HTTP Body (JSON/XML/Form)
响应方向： HTTP Body    ──[Decoder]──→  Java Object
```

两个默认实现（`SpringEncoder`/`SpringDecoder`）都把工作委托给 Spring MVC 的 `HttpMessageConverters`——这意味着**你的 Web 层配了什么 Converter/ObjectMapper，Feign 就享受什么**：日期格式、驼峰策略、`Long` 转 String 防 JS 精度丢失等全局 Jackson 定制会自动作用于远程调用，反过来排查"序列化行为两边不一致"时也应先查转换器配置而不是怀疑 Feign。

# 2. Encoder 接口

```java
public interface Encoder {

    /** 将对象编码为 HTTP 请求体 */
    void encode(Object object, Type bodyType, RequestTemplate template)
            throws EncodeException;
}
```

**参数说明**：

| 参数 | 说明 |
|------|------|
| `object` | 待编码的对象（`@RequestBody` 参数值） |
| `bodyType` | 方法声明的参数类型（含泛型信息） |
| `template` | 目标 `RequestTemplate`，通过 `template.body(...)` 写入 |

两个容易忽略的约定：`object` 为 `null` 时**默认实现不会写入 body**（GET 带 @RequestBody 却传 null 会退化成无 body 请求）；`Content-Type` 不是由 Encoder 决定的——它来自 Contract 解析的 `consumes` 或 `@Headers`，Encoder 只是按它选 Converter，**忘了声明 consumes 时 SpringEncoder 默认按 `application/json` 处理**。

# 3. Decoder 接口

```java
public interface Decoder {

    /** 将 HTTP 响应体解码为对象 */
    Object decode(Response response, Type type)
            throws IOException, DecodeException, FeignException;
}
```

**参数说明**：

| 参数 | 说明 |
|------|------|
| `response` | HTTP 响应（含状态码、头、body） |
| `type` | 方法声明的返回值类型（含泛型信息） |

## 3.1 Decoder 的触发前提：不是每个响应都走它

只有**成功状态（2xx）**的响应才会交给 Decoder；非 2xx（且未开 `decode404`）直接路由到 `ErrorDecoder`（见 `8_error-handling.md`）。这决定了一个设计原则：Decoder 里看到的"业务错误码"只可能是 200 包装的错误（统一响应体 `code != 200` 那类，§7 示例），真正的 HTTP 层错误永远轮不到 Decoder 处理。`type` 携带完整泛型（`List<User>`、`Page<User>`），Decoder 必须靠它而不是擦除后的 `List.class` 还原元素类型——自定义 Decoder 里"列表解码成 `LinkedHashMap`"的著名坑，根源就是用 `Class` 而非 `Type` 调 Jackson。

# 4. SpringEncoder 源码解析

```java
public class SpringEncoder implements Encoder {

    private final ObjectFactory<HttpMessageConverters> messageConverters;

    public SpringEncoder(ObjectFactory<HttpMessageConverters> messageConverters) {
        this.messageConverters = messageConverters;
    }

    @Override
    public void encode(Object requestBody, Type bodyType, RequestTemplate request) {
        if (requestBody != null) {
            // 获取 Content-Type
            Collection<String> contentTypes = request.headers()
                    .get(HttpHeaders.CONTENT_TYPE);
            MediaType mediaType = getMediaType(contentTypes);

            // 遍历 HttpMessageConverters，找到匹配的
            for (HttpMessageConverter<?> converter :
                    this.messageConverters.getObject().getConverters()) {
                if (converter instanceof GenericHttpMessageConverter
                        && ((GenericHttpMessageConverter) converter)
                            .canWrite((Type) bodyType, bodyType.getClass(), mediaType)) {
                    // 使用 Spring 的 GenericHttpMessageConverter 写出
                    // ...
                } else if (converter.canWrite(requestBody.getClass(), mediaType)) {
                    // 使用标准 HttpMessageConverter 写出
                    // ...
                }
            }
        }
    }
}
```

**关键点**：
- SpringEncoder 委托给 Spring MVC 的 `HttpMessageConverters` 体系
- 默认包含 `MappingJackson2HttpMessageConverter`（JSON）、`StringHttpMessageConverter` 等
- 根据 `Content-Type` 请求头匹配合适的 Converter

`ObjectFactory` 注入而非直接持有 `HttpMessageConverters` 实例，是因为编解码器可能配置在**客户端子上下文**（`4_configuration.md` §1.1）：延迟到首次使用时从当前上下文解析，既解决子上下文的 Bean 可见性，也避免主容器转换器还没就绪的启动顺序问题。"遍历取第一个 canWrite"意味着**Converter 顺序即优先级**——注册自定义 Converter 时用 `@Order` 或 `configurePreferredConverters` 明确位置。

# 5. SpringDecoder 源码解析

```java
public class SpringDecoder implements Decoder {

    private final ObjectFactory<HttpMessageConverters> messageConverters;

    @Override
    public Object decode(Response response, Type type) throws IOException {
        // 获取响应 Content-Type
        MediaType mediaType = getMediaType(response);

        // 将 Feign Response 包装为 Spring 的 Message
        FeignResponseAdapter adapter = new FeignResponseAdapter(response);

        // 委托给 HttpMessageConverters 读取
        for (HttpMessageConverter<?> converter :
                this.messageConverters.getObject().getConverters()) {
            if (converter instanceof GenericHttpMessageConverter
                    && ((GenericHttpMessageConverter) converter)
                        .canRead((Class) type, type.getClass(), mediaType)) {
                return ((GenericHttpMessageConverter) converter)
                        .read((Class<?>) type, adapter);
            }
            // ...
        }
    }
}
```

`FeignResponseAdapter` 把 Feign 的 `Response` 适配成 Spring 的 `HttpInputMessage`，代价是**响应流只能被消费一次**：自定义 Decoder 想"先看原始 JSON 再决定怎么解"必须显式 `response.body().asInputStream()` 读全量并缓存字节（`IOUtils.toByteArray` 思路），二次调用 `decode` 拿到的是空流。这也解释了 RetryableException 重试时响应体重新拉取的原因——每次尝试都是全新的 `Response` 对象（重试机制见 `9_retry.md`）。

# 6. 自定义 Encoder

## 6.1 示例：加密请求体

```java
public class AesEncoder implements Encoder {

    private final Encoder delegate;
    private final String secretKey;

    public AesEncoder(Encoder delegate, String secretKey) {
        this.delegate = delegate;
        this.secretKey = secretKey;
    }

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template) {
        // 先序列化到 byte[]
        byte[] bytes = serializeToBytes(object);
        // 加密
        byte[] encrypted = AesUtils.encrypt(bytes, secretKey);
        // Base64 编码写入 body
        template.body(Base64.getEncoder().encodeToString(encrypted));
        template.header("X-Encrypted", "AES");
    }
}
```

示例里 `delegate` 字段没有用到是刻意的对比：两种正确的组合姿势是——**包装型**（如本例，完全接管 body 写入）与**委托型**（`delegate.encode(...)` 后再对 `template.body()` 做二次加工）。委托型保留 Spring 的 Content-Type 协商能力，通常更符合预期；接管型一切序列化细节自负。无论哪种，都要处理 `object == null` 直通返回。

## 6.2 注册

```java
public class FeignEncoderConfig {
    @Bean
    public Encoder feignEncoder(ObjectFactory<HttpMessageConverters> converters) {
        return new AesEncoder(new SpringEncoder(converters), "my-secret-128");
    }
}
```

Bean 名建议就叫 `feignEncoder`/`feignDecoder` 之一并**只注册一个 Encoder Bean**：容器里出现多个 `Encoder` 候选时，客户端上下文的按类型解析会直接抛 `NoUniqueBeanDefinition`，报错却发生在第一次调用远程方法时，堆栈里全是 Feign 内部类，很难一眼关联到"重复注册"。

# 7. 自定义 Decoder

## 7.1 示例：解密响应体 + 统一响应包装

```java
public class UnifiedApiDecoder implements Decoder {

    private final Decoder delegate;

    public UnifiedApiDecoder(Decoder delegate) {
        this.delegate = delegate;
    }

    @Override
    public Object decode(Response response, Type type) throws IOException {
        // 先解析为统一响应格式
        ApiResponse<?> apiResp = (ApiResponse<?>) delegate.decode(response, ApiResponse.class);
        if (apiResp.getCode() != 200) {
            throw new BusinessException(apiResp.getCode(), apiResp.getMessage());
        }
        // 提取 data 字段
        Object data = apiResp.getData();
        // 如果 data 是 JSON 对象，使用 ObjectMapper 转换
        if (data != null && !type.equals(Object.class)) {
            String json = objectMapper.writeValueAsString(data);
            return objectMapper.readValue(json, objectMapper.constructType(type));
        }
        return data;
    }
}
```

这是社区最流行的"拆壳"写法，三个工程细节要补上：`type == void`（返回 `null` 的 `Task<Void>` 场景）必须先短路，否则第二次反序列化必炸；"data 转 JSON 再转目标类型"**丢失泛型**——`List<User>` 会退化成 `List<LinkedHashMap>`，正确做法是 `objectmapper.convertValue(data, factory.constructType(type))` 或保留原始节点用 `treeToValue`；在 Decoder 里抛业务异常要意识到它会**参与重试判定**（非 `RetryableException` 直接上抛，但熔断器会把它计为失败，见 `14_circuit-breaker.md` 的异常统计口径）。

# 8. 多类型 Encoder 处理

```java
public class SmartEncoder implements Encoder {

    private final Map<String, Encoder> encoders = new HashMap<>();

    public SmartEncoder() {
        encoders.put("application/json", new SpringEncoder(converters));
        encoders.put("application/xml", new JAXBEncoder());
        encoders.put("multipart/form-data", new FormEncoder());
        encoders.put("application/x-www-form-urlencoded", new FormEncoder());
    }

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template) {
        String contentType = getContentType(template);
        Encoder encoder = encoders.getOrDefault(contentType, encoders.get("application/json"));
        encoder.encode(object, bodyType, template);
    }
}
```

按 Content-Type 分发的路由器模式有一个不可省略的细节：`FormEncoder` 处理表单/multipart 时同样需要包装 `SpringEncoder` 作为 delegate（直接 `new FormEncoder()` 会用 Feign 内置编码器，JSON 字段会退化成 toString）。Spring Cloud 环境直接用现成的 `SpringFormEncoder`（包装 SpringEncoder），不要重复造这个轮子——文件上传专题见 `18_file-upload-download.md`。

# 9. 常用 Encoder/Decoder 实现

| 实现 | 说明 |
|------|------|
| `Encoder.Default` | 简单的 toString + UTF-8 编码（Feign 内置） |
| `SpringEncoder` | Spring MVC `HttpMessageConverters` 代理 |
| `JacksonEncoder` | Jackson 直接序列化 |
| `GsonEncoder` | Gson 序列化 |
| `FormEncoder` | 表单编码（multipart/form-data, application/x-www-form-urlencoded） |
| `Decoder.Default` | 简单的 String 解码（Feign 内置） |
| `SpringDecoder` | Spring MVC `HttpMessageConverters` 代理 |
| `JacksonDecoder` | Jackson 直接反序列化 |
| `GsonDecoder` | Gson 反序列化 |

`Encoder.Default`/`Decoder.Default` 是纯 OpenFeign 的兜底，只处理 String；看到"对象被编码成 `User@1a2b3c`"这类报文，几乎可以断定 Encoder 被意外换回或没生效（常见于自定义 Contract/Builder 时把组件带丢了，对应 `4_configuration.md` §6.1 的污染面）。

# 10. 最佳实践

1. **始终返回标准响应体类型**——避免 Decoder 中做过多类型转换
2. **统一 API 响应格式**——通过 Decoder 统一处理 `code/message/data` 结构
3. **处理 404 场景**——`decode404: true` + Decoder 中对空 body 做防御
4. **异常处理在 Decoder 中做**——不要在 Encoder 的 try-catch 中吞掉异常
5. **自定义编解码用"包装 + 委托"而非"替换"**——保留 `SpringEncoder`/`SpringDecoder` 作为 delegate，只加工自己关心的那一段（加密、拆壳、字段映射），Spring 生态的 Content-Type 协商与泛型还原能力才不会丢；
6. **Decoder 保持无状态**——它与代理一样是多线程共享单例，ObjectMapper 可以是 final 字段，但任何"上次解析结果"式的成员变量都是竞态源。
