
# 1. Contract 是什么

`Contract` 是 Feign 的核心组件之一，负责将 Java 接口上的注解**解析为 `MethodMetadata`**（方法元数据），包含：

- HTTP 方法（GET/POST/PUT/DELETE 等）
- 请求路径模板（如 `/users/{id}`）
- 请求头（Header）映射
- 查询参数映射
- 请求体（Body）位置
- 返回值类型信息

# 2. 两种内置 Contract

| Contract | 使用场景 | 支持的注解 |
|----------|----------|------------|
| `Contract.Default` | 纯 OpenFeign 环境 | Feign 原生注解（`@RequestLine`, `@Param`, `@Headers`, `@Body`） |
| `SpringMvcContract` | Spring Cloud 环境（**默认**） | Spring MVC 注解（`@RequestMapping`, `@PathVariable`, `@RequestParam` 等） |

Spring Cloud OpenFeign 默认使用 `SpringMvcContract`，通过 `FeignClientsRegistrar` 自动注入。

# 3. SpringMvcContract 解析流程

```
@FeignClient 接口
  │
  ▼
SpringMvcContract.parseAndValidateMetadata(Class<?> targetType, Method method)
  │
  ├── 解析 @RequestMapping / @GetMapping / @PostMapping ...
  │     ├── 提取 HTTP 方法 (method)
  │     ├── 提取 URL 模板 (value/path)
  │     ├── 提取 produces → Accept 头
  │     └── 提取 consumes → Content-Type 头
  │
  ├── 解析参数注解
  │     ├── @PathVariable → indexToName 映射 + path variable
  │     ├── @RequestParam → indexToName 映射 + query parameter
  │     ├── @RequestHeader → indexToName 映射 + header
  │     ├── @RequestBody → bodyIndex
  │     ├── @RequestPart → formFieldIndex
  │     └── @SpringQueryMap → queryMapIndex
  │
  ├── 解析 @Headers 注解（类级别和方法级别）
  │
  └── 返回 MethodMetadata
        ├── configKey (类名#方法名(参数类型))
        ├── template (RequestTemplate)
        ├── returnType
        ├── headerMapIndex
        ├── queryMapIndex
        └── indexToName / indexToExpander
```

# 4. 自定义 Contract

## 4.1 扩展 SpringMvcContract

```java
public class MySpringMvcContract extends SpringMvcContract {

    public MySpringMvcContract() {
        // parameterAnnotationsProcessors 默认会处理 @PathVariable 等
        // 可以在此注册自定义参数注解处理器
    }

    @Override
    protected void processAnnotationOnMethod(MethodMetadata data,
                                             Annotation methodAnnotation,
                                             Method method) {
        // 处理自定义注解
        if (methodAnnotation instanceof MyApiVersion) {
            MyApiVersion version = (MyApiVersion) methodAnnotation;
            data.template().header("X-Api-Version", version.value());
            return;
        }
        super.processAnnotationOnMethod(data, methodAnnotation, method);
    }

    @Override
    protected boolean processAnnotationsOnParameter(MethodMetadata data,
                                                    Annotation[] annotations,
                                                    int paramIndex) {
        boolean processed = super.processAnnotationsOnParameter(data, annotations, paramIndex);
        // 自定义参数注解处理...
        return processed;
    }
}
```

## 4.2 注册自定义 Contract

```java
@FeignClient(name = "user-service",
             configuration = MyContractConfig.class)
public interface UserClient { }
```

```java
public class MyContractConfig {
    @Bean
    public Contract feignContract() {
        return new MySpringMvcContract();
    }
}
```

# 5. MethodMetadata 关键字段

```java
public final class MethodMetadata implements Serializable {

    // 方法唯一标识：ClassName#methodName(ParamType1,ParamType2)
    private String configKey;

    // 返回值类型
    private Type returnType;

    // 请求模板（URL, Headers, Query 等）
    private RequestTemplate template;

    // 参数索引 → 参数名称映射
    private Map<Integer, Collection<String>> indexToName;

    // 参数索引 → 参数值展开器（处理数组/集合等）
    private Map<Integer, Param.Expander> indexToExpander;

    // @SpringQueryMap 参数索引
    private Integer queryMapIndex;

    // @RequestHeader @HeaderMap 参数索引
    private Integer headerMapIndex;

    // 请求体参数索引
    private int bodyIndex;

    // 表单字段索引
    private int formFieldIndex;

    // 先序列化为 JSON 再编码为 URL 的查询参数
    private Map<Integer, EncodedQueryParam> encodedQueryParams;

    // 可忽略的类型（用于继承接口的场景）
    private Set<Integer> ignoreParams;

    // ...
}
```

# 6. 自定义参数注解处理器

```java
@Component
public class MyRequestHeaderParameterProcessor
        extends AnnotatedParameterProcessor {

    @Override
    public Class<? extends Annotation> getAnnotationType() {
        return MyHeader.class;
    }

    @Override
    public boolean processArgument(AnnotatedParameterContext context,
                                   Annotation annotation,
                                   Method method) {
        MyHeader myHeader = AnnotatedParameterProcessor
                .findMethodAnnotation(MyHeader.class, context.getParameterIndex(), method);
        if (myHeader != null) {
            context.setParameterSpecification(
                    new SimpleParameterSpecification(
                            Header.class.getSimpleName(),
                            myHeader.value()
                    )
            );
        }
        return true;
    }
}
```

使用时在 `application.yml` 声明（或在 Feign Builder 中注册）：

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            request-interceptors:
              - com.example.MyHeaderParamProcessor  # 实际上需要在 Builder 注册
```

# 7. Contract.Default 解析过程（对比）

```java
Contract.Default 对应的注解体系：

@RequestLine("GET /users/{id}")        → HTTP method + URL template
@Param("id")                           → named parameter
@Headers("X-Token: {token}")           → header template
@Body("%7B\"name\": \"{name}\"%7D")   → body template
@QueryMap(encoded = true)              → query parameter map
```

与 `SpringMvcContract` 本质相同，只是注解类型/解析规则不同。

# 8. 常见问题

**Q：为什么 `@RequestParam` 不传 value 会报错？**

`SpringMvcContract` 解析 `@RequestParam` 时强制要求 `name()` 不为空。这是因为编译后的接口方法参数名默认不保留（除非加 `-parameters` 编译参数），无法通过反射获取。

**Q：`@PostMapping` 同时有 `@RequestParam` 和 `@RequestBody`，body 会被如何处理？**

```java
@PostMapping("/users")
User create(@RequestParam String source, @RequestBody User user);
```

- `@RequestParam` → Query parameter
- `@RequestBody` → HTTP body

生成的请求：`POST /users?source=web` + `{"name":"xxx","age":20}`

两者可以共存，互不影响。
