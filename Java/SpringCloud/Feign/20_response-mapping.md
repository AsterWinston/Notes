# 1. 概述

Feign 的 Decoder 需要根据返回值类型（含泛型信息）来正确反序列化 HTTP 响应。对于复杂类型（泛型、多态、嵌套泛型等），需要特殊处理。

## 1.1 先把"类型信息从哪来"这条线理清

本篇所有讨论的地基是一个常被讲错的事实：**Feign 传给 Decoder 的 `type` 参数是方法签名的完整 `java.lang.reflect.Type`，泛型信息全程不丢**。`MethodMetadata` 在 Contract 解析期就把 `method.getGenericReturnType()` 存好（`5_contract.md` §3），运行期原样交给 Decoder（`6_encoder-decoder.md` §3.1）。所以"类型擦除导致 `ApiResponse<User>` 解不出 User"的说法是讹传——擦除发生在**Decoder 实现自己把 `Type` 降级成 `Class`** 的那一刻（`readValue(json, (Class) type)` 写法）。真正需要本篇技巧的场景是三类：Decoder 实现不称职（裸 Jackson 按 rawClass 解）、想要**换返回形态**（拆壳后接口只写业务类型）、以及多态字段（JSON 里子类不定，泛型也帮不了你）。分清是"信息丢了"还是"信息没用上"，排障方向完全不同。

# 2. 泛型返回类型

## 2.1 简单泛型：List`<T>`、Map`<K,V>`

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

批注纠偏：正确的表述是"签名里的泛型经 `getGenericReturnType` 保留为 `ParameterizedType` 传给 Decoder"。`List<User>` 能解出**元素是 User 而不是 LinkedHashMap**，靠的就是这个 Type 被 Jackson 的 `constructType` 消费（`6_encoder-decoder.md` §3.1 的名著名坑：元素变 LinkedHashMap 一定是某个环节把 Type 丢了，不是擦除的宿命）。

## 2.2 带泛型的包裹类：能用，但有代价

```java
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
}

@FeignClient(name = "user-service")
public interface UserClient {

    // ✓ 可以正确解码：Feign 传的是 ApiResponse<User> 的完整 Type
    @GetMapping("/api/users/{id}")
    ApiResponse<User> getUser(@PathVariable Long id);

    // ✓ 具体子类型：另一种风格
    @GetMapping("/api/users/{id}")
    UserResponse getUserV2(@PathVariable Long id);
}

// 定义具体类型
public class UserResponse extends ApiResponse<User> { }
```

（原文此处把 `ApiResponse<User>` 标为"类型擦除信息丢失 ❌"，是流传很广的讹误，已按 §1.1 修正。）两种写法都能工作，差异在别处：`ApiResponse<User>` 版的真实成本是**每个调用点都要拆壳**——`response.getData()` 前得先判 code，判错逻辑散落在每个 service 里；`extends` 具体子类版把壳固化成类型，配合拆壳 Decoder 后接口甚至可以直接写 `User`（§8）。还有一个会踩的场景：`ApiResponse<List<User>>` 这类**嵌套泛型**在自定义 Decoder 里若用 `pType.getActualTypeArguments()[0]` 逐层手工下钻，第二层就丢了——正确姿势永远是 `mapper.constructType(完整Type)` 一次到位（§3.2 的写法保留了这个坑的解法）。

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

推荐理由不是"泛型解不出"（§2.2 已澄清），而是**契约显式化**：返回类型即文档，拆壳/告警逻辑可以按具体类型在统一处消费；对 Decoder 实现的要求也最低（任何称职的实现都吃 Type，具体类连 Type 都不用传）。代价是类数量随接口数膨胀，常见折中是只给**高频壳**建具体类。

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

这个实现的适用范围要想清楚：**只有"想在 ApiResponse 壳处停下、但仍要壳里的元信息"时才需要它**——按 Type 直通给 Jackson 本来就能解整个壳，手写逐层拆解是为了拿 `code/message` 参与决策（不 throw 而返回带码对象）。`root.get("data").toString()` 再 readValue 是**序列化往返两遍**的低效写法，`mapper.treeToValue(dataNode, ...)` 一步到位（§8 已改用它）；`getActualTypeArguments()[0]` 只钻一层，T 本身是泛型（`ApiResponse<List<User>>`）时 dataType 仍是 ParameterizedType，`constructType` 恰好能吃下——这是这段代码唯一"侥幸正确"的地方，值得注释固化。

## 3.3 方案 3：使用 feign.Response + 手动解析

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/api/users/{id}")
    feign.Response getUserRaw(@PathVariable Long id);
}
```

```java
// 手动解析
feign.Response response = userClient.getUserRaw(1L);
String json = Util.toString(response.body().asReader(UTF_8));
ApiResponse<User> apiResp = objectMapper.readValue(json,
        new TypeReference<ApiResponse<User>>() {});
```

TypeReference 匿名子类是 Jackson 对抗擦除的经典招（用 `class{}` 捕获泛型超类签名）。Feign 场景把返回类型降级成 `feign.Response` 意味着**放弃整条声明式管线的下游部分**：熔断统计看不到业务失败、要自己判 `status()`（非 2xx 仍会先过 ErrorDecoder 抛异常，通常只有成功响应与 decode404 的 404 会进到你手里）、`response` 需手动 close（`18_file-upload-download.md` §6.2 的泄漏纪律同样适用）。它的正当用途只剩两类：需要原始报文做验签/审计，和**跨版本灰度期**对新字段容错解析。别让它成为日常风格。

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

不直接反序列化成 `Page<User>` 接口类型的理由是硬的：`Page` 是接口、`PageImpl` 的 JSON 形态**从来不是稳定的对外契约**——Spring Boot 2.3 起默认不再暴露 `PageImpl` 序列化（`spring.jackson.serialization.fail-on-empty-beans` 之外的 `@JsonUnwrapped` pageable 结构变化），Boot 3 / Jackson 配置差异还会带来 `paged/list` 字段有无。所以提供方要么原样透传这套"演化中的结构"（脆弱），要么自定义 `CustomPage`（本方案，只取用得着的字段——防御性契约）；更规范的做法是提供方自己定义分页 DTO（§9.5 的精神：分页结构属于**你们的 API 契约**，不属于 Spring Data）。字段映射要核对一处易错点：JSON 里当前页码字段是 `number`（0 起），DTO 若命名 `pageIndex` 需 `@JsonProperty` 对齐。

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

路由器模式三个必须补的细节：header 名匹配要**大小写不敏感**（Feign 的 headers Map 本身大小写归一，但 `get("Content-Type")` 之外的自定义网关可能不保证，拿值前先当它会缺省处理）；`split(";")[0]` 剥参数正确，但媒体类型子型（`application/hal+json`、`application/problem+json`）落不进精确 Map——真实路由要用 `MediaType.parseMediaType` + `includes/isCompatibleWith` 匹配，否则 Spring HATEOAS 响应会静默走 defaultDecoder；**每个分支 Decoder 都得自己处理 `type`**——`new JacksonDecoder()` 用默认 ObjectMapper，不吃 Spring 容器的定制（日期/驼峰，`6_encoder-decoder.md` §1.1），一致性要求高时分支应包 SpringDecoder 而不是裸 Jackson。多数团队其实不需要这个类：Content-Type 协商本来就是 `HttpMessageConverters` 的职责（SpringDecoder 内部就在做 §5 的事），手写路由通常只出现在"非 JSON 媒体类型 + 纯 OpenFeign"组合里。

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

"空"在 HTTP 里有四种形态，处理矩阵要背下来：

```text
204 No Content      → 无 body；Jackson 对空流返回 null（对象目标）或抛 MismatchedInputException（int/boolean 等基本类型目标！）
200 + 空 body       → 同上，但 decode404 体系管不到它
200 + "null" 字面量 → Jackson 解出 null，正常路径
404                 → ErrorDecoder / decode404（8_error-handling.md §7）
```

两个高危组合：返回类型是 `int`/`boolean` 等基本类型包装时遇到 204 会炸（签名改 `Integer` 或在 Decoder 里按 §6 短路）；`Optional<User>` 需要 `type == Optional.class` 的**逐层判断**——真实 `Optional` 支持在 SpringDecoder 路径由 `HttpMessageConverters` 处理，自定义实现里 `type` 是 `ParameterizedType`，示例的 `== Optional.class` 对 `Optional<User>` 永不成立，应比对 `getRawType()`。

# 7. WebClient vs Feign 类型解析对比

| | Feign | WebClient |
|------|-------|-----------|
| 泛型保留 | 方法签名保留泛型信息 → Decoder | `bodyToMono(typeRef)` |
| 复杂泛型 | 需定义为具体子类或自定义 Decoder | `ParameterizedTypeReference` |
| 响应式 | 不原生支持 | 原生 `Mono<T>` / `Flux<T>` |
| 错误响应 | ErrorDecoder | `onStatus` / `onErrorMap` |

"复杂泛型"一行的 Feign 侧描述按 §1.1 更新：**Feign 的签名泛型同样完整可用**，与 WebClient 的差别不在擦除而在**表达位置**——WebClient 在调用点用 `new ParameterizedTypeReference<>() {}` 现场构造类型，Feign 在声明点由签名静态决定；前者灵活（同一段代码可换类型）、后者可被契约工具消费（文档生成、拆壳 Decoder 都靠声明式形态）。"响应式不原生支持"也有边界：Feign 的方法可以返回 `Optional`/自定义包装，WebFlux 工程里 Feign 仍是阻塞 Client（需要配 `Schedulers` 隔离或干脆该场景选 WebClient）。

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

这是 §3 三方案的合体终态（接口写 `User` = 拆壳后类型即业务类型；Type 直通 `constructType` = 泛型不丢），补四个生产细节：`type == void.class`（方法返回 `void`/`Task`）要短路返回 null，否则对 `data=null` 的响应继续解必炸（`6_encoder-decoder.md` §7 同款）；`root.get("code")` 判 null——网关截断返回非壳 JSON（如 Nginx 的 502 错误页）时直接 NPE，规范做法是"不是壳"原样交 delegate/或抛带原始 body 的解析异常；`treeToValue(dataNode, mapper.constructType(type))` 替代 `toString()+readValue` 省一次序列化；**Decoder 里 throw 的业务异常参与熔断统计口径**（`8_error-handling.md` §9.7 / `14_circuit-breaker.md` §3.2 的 record-exceptions 要认识 `BusinessException`——业务码错误默认会被计入失败率）。注册后别忘了对**不用壳的老服务**留豁免：拆壳 Decoder 挂到全局配置会误伤返回裸对象的第三方 API，它必须只出现在 `configuration` 专属挂载里。

# 9. 最佳实践

1. **优先定义具体返回类型** — 避免直接使用 `ApiResponse<T>` 作为 Feign 方法返回类型（动机是契约显式化与拆壳统一，不是泛型解不出——§2.2）
2. **统一错误码在 Decoder 中处理** — 将 ApiResult 的 code/message 剥离，Feign 接口只关心 data（配套的统计口径见 §8 第三条）
3. **大响应使用 `feign.Response`** — 手动按需读取，避免一次性加载
4. **配合 `@Nullable` / `Optional`** — 明确表达可能为空的语义（基本类型返回值是空响应矩阵里唯一的硬坑，§6）
5. **分页响应定义为明确的 Page 类** — 不要依赖 Spring Data Page 接口（`PageImpl` 的 JSON 形态不受你们契约控制，§4.1）
6. **多态 data 字段提前设计好路由键** — `T data` 的具体实现类不定（如回调载荷）时，Jackson 泛型帮不了你，需要在壳里放 `dataType` 判别字段 + `@JsonTypeInfo` 或拆壳后二次解析（§8 的 treeToValue 按判别字段选类是标准做法）。
