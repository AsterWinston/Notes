
# 1. @FeignClient

```java
@FeignClient(
    name = "user-service",           // 服务名（必填），同时也作为 spring bean name
    path = "/api/users",             // 统一路径前缀
    url = "http://localhost:8080",   // 直连 URL（不经过注册中心）
    configuration = UserFeignConfig.class,  // 自定义配置类
    fallback = UserClientFallback.class,     // 降级类
    fallbackFactory = UserClientFallbackFactory.class, // 降级工厂（可获取异常）
    primary = true                   // 当有多个同类型 bean 时，是否作为首选
)
public interface UserClient {
    // ...
}
```

| 属性 | 说明 | 默认值 |
|------|------|--------|
| `name` / `value` | 服务名（bean name 和 serviceId） | 必填 |
| `path` | 所有方法的统一路径前缀 | `""` |
| `url` | 直连 URL，不经过负载均衡 | `""` |
| `configuration` | 自定义配置类（注意不要被 Spring 扫描到） | `{}` |
| `fallback` | 降级类，必须实现本接口 | `void.class` |
| `fallbackFactory` | 降级工厂，可获取 `Throwable` | `void.class` |
| `primary` | 是否作为主 bean | `true` |
| `contextId` | 上下文 ID，用于区分同名 FeignClient 的不同配置 | `name` |
| `qualifiers` / `qualifier` | 限定符别名 | `{}` |
| `dismiss404` | 404 时是否不抛异常返回 null | `false` |

# 2. Spring MVC 映射注解

## 2.1 @RequestMapping / @GetMapping / @PostMapping / @PutMapping / @DeleteMapping

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @RequestMapping(method = RequestMethod.GET, value = "/users/{id}")
    User getById(@PathVariable Long id);

    @GetMapping("/users")
    List<User> list();

    @PostMapping("/users")
    User create(@RequestBody User user);

    @PutMapping("/users/{id}")
    User update(@PathVariable Long id, @RequestBody User user);

    @DeleteMapping("/users/{id}")
    void delete(@PathVariable Long id);
}
```

Feign 在 Spring Cloud 下支持所有 Spring MVC 的 `@RequestMapping` 衍生注解，因为默认使用 `SpringMvcContract` 来解析。

## 2.2 @RequestMapping 的 produces / consumes

```java
@PostMapping(value = "/users", produces = "application/json", consumes = "application/json")
User create(@RequestBody User user);
```

这些属性会被转换为 HTTP 请求头：
- `produces` → `Accept` 头
- `consumes` → `Content-Type` 头

# 3. 参数注解

## 3.1 @PathVariable — 路径参数

```java
@GetMapping("/users/{userId}/orders/{orderId}")
Order getOrder(@PathVariable("userId") Long userId,
               @PathVariable("orderId") Long orderId);
```

- **必须指定 value**（Feign 接口中不能省略）
- 参数会替换 URL 模板中的 `{name}` 占位符

## 3.2 @RequestParam — 查询参数

```java
@GetMapping("/users")
List<User> search(@RequestParam("name") String name,
                  @RequestParam("age") Integer age,
                  @RequestParam(value = "page", defaultValue = "1") int page);
```

- **必须指定 value**
- 生成的 URL：`/users?name=xxx&age=yyy&page=1`
- 支持 `defaultValue`，参数为 `null` 时使用默认值

## 3.3 @RequestHeader — 请求头

```java
@GetMapping("/users")
User getByToken(@RequestHeader("X-Auth-Token") String token);
```

- 将方法参数值放入请求头
- 常用于传递认证 Token、TraceId 等

## 3.4 @RequestBody — 请求体

```java
@PostMapping("/users")
User create(@RequestBody User user);
```

- 每个方法**最多只能有一个** `@RequestBody`
- 参数会被 `Encoder` 序列化为 JSON / XML 等

## 3.5 @SpringQueryMap — 对象展开为查询参数

```java
public class UserQuery {
    private String name;
    private Integer age;
    private List<Long> ids;
    // getter/setter...
}

@GetMapping("/users")
List<User> search(@SpringQueryMap UserQuery query);
```

相当于生成：`/users?name=xxx&age=yyy&ids=1&ids=2&ids=3`

这是 Spring Cloud OpenFeign 提供的扩展，**不是** Feign 原生或 Spring MVC 标准注解。

## 3.6 @RequestPart — 文件上传

```java
@PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
String upload(@RequestPart("file") MultipartFile file);
```

需要额外引入 `feign-form` 和 `feign-form-spring` 依赖，详见文件上传专题。

# 4. Feign 原生注解（可选）

除了 Spring MVC 注解，Feign 还有自己的一套注解体系（通过默认的 `Contract.Default` 解析）：

| Feign 原生注解 | 等效的 Spring MVC 注解 |
|----------------|----------------------|
| `@RequestLine("GET /users/{id}")` | `@GetMapping("/users/{id}")` |
| `@Param("id")` | `@PathVariable("id")` |
| `@Headers("Content-Type: application/json")` | `@RequestMapping(consumes=...)` |
| `@Body("{body}")` | `@RequestBody` |
| `@QueryMap` | `@SpringQueryMap` |

**不推荐混用**：在 Spring Cloud 环境下应统一使用 Spring MVC 注解，除非你显式指定了 `contract = Contract.Default.class`。

# 5. 注解继承

Feign 支持**接口继承**：

```java
public interface BaseApi<T> {
    @GetMapping("/{id}")
    T getById(@PathVariable Long id);

    @PostMapping
    T create(@RequestBody T entity);
}

@FeignClient(name = "user-service")
public interface UserClient extends BaseApi<User> {
    @GetMapping("/search")
    List<User> search(@RequestParam String name);
}
```

`UserClient` 会继承 `BaseApi` 中的 `@GetMapping` 和 `@PostMapping` 方法定义。

# 6. 复杂 URL 模板

```java
@GetMapping("/search{?name,age,status*}")
List<User> complexSearch(@RequestParam String name,
                         @RequestParam Integer age,
                         @RequestParam Map<String, String> status);

// 或使用数组参数
@GetMapping("/users")
List<User> findByIds(@RequestParam("ids") Long[] ids);
// 生成: /users?ids=1&ids=2&ids=3
```

- `{?name,age}` — 可选查询参数（v10.x+ 支持 RFC 6570 模板）
- `{?status*}` — Map 展开，`*` 表示展开（explode）

# 7. 注意事项

1. **`@PathVariable` 必须指定 value** — 不指定会导致 `IllegalStateException`
2. **`@RequestParam` 必须指定 value** — 否则会报错 `name() must not be empty`
3. **参数为 null 时的行为** — `@RequestParam` 为 null 时参数会被忽略；`@RequestBody` 为 null 时 body 为空
4. **`@SpringQueryMap` 仅用于参数编码** — 不会触发 HTTP body 请求体编码
5. **不要混用 Spring MVC 注解与 Feign 原生注解** — 会导致 MethodMetadata 解析冲突
