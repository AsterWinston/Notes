
# 1. 概述

Feign 的 Decoder 需要根据返回值类型（含泛型信息）来正确反序列化 HTTP 响应。对于复杂类型（泛型、多态、嵌套泛型等），需要特殊处理。

# 2. 泛型返回类型

## 2.1 简单泛型：List<T>、Map<K,V>

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/api/users")
    List<User> listUsers();                    // ✓ 类型擦除后仍是 List

    @GetMapping("/api/users/map")
    Map<Long, String> getUserMap();            // ✓ 能正确解析
}
```

Spring Decoder 基于 `ParameterizedTypeReference` 保留泛型信息，能正确处理。

## 2.2 复杂泛型：带泛型包裹类

```java
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
}

@FeignClient(name = "user-service")
public interface UserClient {

    // ❌ 问题：类型擦除后泛型信息丢失
    @GetMapping("/api/users/{id}")
    ApiResponse<User> getUser(@PathVariable Long id);

    // ✓ 正确做法：使用具体类型
    @GetMapping("/api/users/{id}")
    UserResponse getUser(@PathVariable Long id);
}

// 定义具体类型
public class UserResponse extends ApiResponse<User> { }
```

# 3. 解决方案：ParameterizedTypeReference

针对泛型包裹类的场景，有几种方案：

## 3.1 方案 1：定义具体返回类型（推荐）

```java
public class UserListResponse extends ApiResponse<List<User>> { }
public class UserDetailResponse extends ApiResponse<User> { }
public class OrderResponse extends ApiResponse<Order> { }

@GetMapping("/api/users/{id}")
UserDetailResponse getUser(@PathVariable Long id);

@GetMapping("/api/users")
UserListResponse listUsers();
```

## 3.2 方案 2：自定义 Decoder 处理泛型

```java
public class GenericAwareDecoder implements Decoder {

    private final ObjectMapper objectMapper;
    private final Decoder delegate;

    @Override
    public Object decode(Response response, Type type) throws IOException {
        if (type instanceof ParameterizedType) {
            ParameterizedType pType = (ParameterizedType) type;
            Class<?> rawType = (Class<?>) pType.getRawType();

            // 处理 ApiResponse<T> 泛型解析
            if (rawType == ApiResponse.class) {
                Type dataType = pType.getActualTypeArguments()[0];
                String body = Util.toString(response.body().asReader(UTF_8));
                JsonNode root = objectMapper.readTree(body);
                ApiResponse<Object> apiResponse = new ApiResponse<>();
                apiResponse.setCode(root.get("code").asInt());
                apiResponse.setMessage(root.get("message").asText());
                // 按泛型参数类型反序列化 data
                Object data = objectMapper.readValue(
                        root.get("data").toString(),
                        objectMapper.constructType(dataType)
                );
                apiResponse.setData(data);
                return apiResponse;
            }
        }
        return delegate.decode(response, type);
    }
}
```

## 3.3 方案 3：使用 feign.Response + 手动解析

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/api/users/{id}")
    feign.Response getUserRaw(@PathVariable Long id);
}

// 手动解析
feign.Response response = userClient.getUserRaw(1L);
String json = Util.toString(response.body().asReader(UTF_8));
ApiResponse<User> apiResp = objectMapper.readValue(json,
        new TypeReference<ApiResponse<User>>() {});
```

# 4. 分页结果映射

## 4.1 场景：Spring Data 分页

```java
// 服务端返回
@GetMapping("/api/users")
public Page<User> listUsers(Pageable pageable) { ... }
```

```json
{
  "content": [{ "id": 1, "name": "John" }],
  "pageable": { ... },
  "totalElements": 100,
  "totalPages": 10,
  "size": 10,
  "number": 0
}
```

```java
// Feign 客户端
public class CustomPage<T> {
    private List<T> content;
    private long totalElements;
    private int totalPages;
    private int size;
    private int number;
    // getter/setter...
}
```

```java
@GetMapping("/api/users")
CustomPage<User> listUsers(@SpringQueryMap PageQuery query);
```

# 5. 不同 Content-Type 的 Decoder 选择

```java
public class MultiTypeDecoder implements Decoder {

    private final Map<String, Decoder> decoders = new HashMap<>();
    private final Decoder defaultDecoder;

    public MultiTypeDecoder(Decoder defaultDecoder) {
        this.defaultDecoder = defaultDecoder;
        decoders.put("application/json", new JacksonDecoder());
        decoders.put("application/xml", new JAXBDecoder());
        decoders.put("text/plain", new StringDecoder());
        decoders.put("application/octet-stream", new ByteArrayDecoder());
    }

    @Override
    public Object decode(Response response, Type type) throws IOException {
        String contentType = getContentType(response);
        Decoder decoder = decoders.getOrDefault(contentType, defaultDecoder);
        return decoder.decode(response, type);
    }

    private String getContentType(Response response) {
        Collection<String> values = response.headers().get("Content-Type");
        return values != null ? values.iterator().next().split(";")[0].trim() : "application/json";
    }
}
```

# 6. 响应为空 / null 的处理

```java
@FeignClient(name = "user-service")
public interface UserClient {

    // 可能返回空对象 {} 或 HTTP 204
    @GetMapping("/api/users/{id}")
    Optional<User> getOptionalUser(@PathVariable Long id);

    @GetMapping("/api/users/{id}")
    @Nullable
    User getNullableUser(@PathVariable Long id);
}
```

```java
// 自定义 Decoder 中处理空响应
@Override
public Object decode(Response response, Type type) throws IOException {
    if (response.status() == 204 || response.body() == null) {
        if (type == Optional.class) {
            return Optional.empty();
        }
        return null;
    }
    // ...
}
```

# 7. WebClient vs Feign 类型解析对比

| | Feign | WebClient |
|------|-------|-----------|
| 泛型保留 | 方法签名保留泛型信息 → Decoder | `bodyToMono(typeRef)` |
| 复杂泛型 | 需定义为具体子类或自定义 Decoder | `ParameterizedTypeReference` |
| 响应式 | 不原生支持 | 原生 `Mono<T>` / `Flux<T>` |
| 错误响应 | ErrorDecoder | `onStatus` / `onErrorMap` |

# 8. 实践：统一 API 响应处理

```java
// 1. 定义统一响应格式
public class ApiResult<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp;
}

// 2. 自定义 Decoder
public class ApiResultDecoder implements Decoder {

    private final ObjectMapper mapper;
    private final Decoder delegate;

    @Override
    public Object decode(Response response, Type type) throws IOException {
        // 解析出 data 的 JSON
        String body = Util.toString(response.body().asReader(UTF_8));
        JsonNode root = mapper.readTree(body);

        int code = root.get("code").asInt();
        if (code != 200) {
            throw new BusinessException(code, root.get("message").asText());
        }

        // 将 data 字段按期望类型反序列化
        JsonNode dataNode = root.get("data");
        if (dataNode == null || dataNode.isNull()) {
            return null;
        }
        return mapper.readValue(dataNode.toString(),
                mapper.constructType(type));
    }
}

// 3. Feign 接口直接写业务对象，无需 ApiResult 包裹
@FeignClient(name = "user-service",
             configuration = ApiResultConfig.class)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id);   // 直接返回 User，ApiResult 由 Decoder 剥离
}
```

# 9. 最佳实践

1. **优先定义具体返回类型** — 避免直接使用 `ApiResponse<T>` 作为 Feign 方法返回类型
2. **统一错误码在 Decoder 中处理** — 将 ApiResult 的 code/message 剥离，Feign 接口只关心 data
3. **大响应使用 `feign.Response`** — 手动按需读取，避免一次性加载
4. **配合 `@Nullable` / `Optional`** — 明确表达可能为空的语义
5. **分页响应定义为明确的 Page 类** — 不要依赖 Spring Data Page 接口
