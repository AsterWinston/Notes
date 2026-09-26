
# 1. 概述

`Encoder` 和 `Decoder` 是 Feign 请求/响应**编解码**的核心接口，负责 Java 对象与 HTTP 报文之间的相互转换。

```
请求方向： Java Object  ──[Encoder]──→  HTTP Body (JSON/XML/Form)
响应方向： HTTP Body    ──[Decoder]──→  Java Object
```

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

            // 遍历 HttpMessageConverter，找到匹配的
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

## 6.2 注册

```java
public class FeignEncoderConfig {
    @Bean
    public Encoder feignEncoder(ObjectFactory<HttpMessageConverters> converters) {
        return new AesEncoder(new SpringEncoder(converters), "my-secret-128");
    }
}
```

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

# 10. 最佳实践

1. **始终返回标准响应体类型**——避免 Decoder 中做过多类型转换
2. **统一 API 响应格式**——通过 Decoder 统一处理 `code/message/data` 结构
3. **处理 404 场景**——`decode404: true` + Decoder 中对空 body 做防御
4. **异常处理在 Decoder 中做**——不要在 Encoder 的 try-catch 中吞掉异常
