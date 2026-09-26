
# 1. 概述

## 1.1 什么是 JAX-RS

JAX-RS（Jakarta RESTful Web Services）是 JavaEE 的 **RESTful Web 服务规范**（JSR-311 → 339），用注解声明 HTTP 资源与方法的映射。

- 参考实现（RI）：**Jersey**
- 其他实现：**RESTEasy**（Red Hat）、Apache CXF

```
HTTP 方法（GET/POST/PUT/DELETE/PATCH）
    │
    ▼
JAX-RS 注解（@GET @Path @Produces...）
    │
    ▼
资源类（POJO）
```

## 1.2 REST 核心概念

- **资源**：通过 URI 标识（如 `/users/1`）。
- **HTTP 方法语义**：GET 读取、POST 创建、PUT 全量更新、PATCH 部分更新、DELETE 删除。
- **无状态**：服务器不保存客户端会话状态。
- **表现层**：JSON/XML 等表示形式（Content-Type）。

# 2. 核心注解

| 注解 | 作用 |
|------|------|
| `@Path` | 资源/方法的 URI 路径 |
| `@GET` `@POST` `@PUT` `@DELETE` `@PATCH` `@HEAD` `@OPTIONS` | HTTP 方法绑定 |
| `@Produces` | 响应内容类型（如 `application/json`） |
| `@Consumes` | 请求内容类型 |
| `@PathParam` | 路径参数 `{id}` |
| `@QueryParam` | 查询参数 `?page=1` |
| `@MatrixParam` | 矩阵参数 `;a=b` |
| `@HeaderParam` | Header 参数 |
| `@CookieParam` | Cookie 参数 |
| `@FormParam` | 表单参数 |
| `@BeanParam` | 聚合参数到对象 |
| `@Context` | 注入上下文（UriInfo、HttpHeaders 等） |
| `@ApplicationPath` | 应用根路径 |

# 3. 资源类示例

```java
@Path("/users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    @GET
    public List<User> list(@QueryParam("page") @DefaultValue("1") int page) {
        return userService.findAll(page);
    }

    @GET
    @Path("/{id}")
    public User get(@PathParam("id") Long id) {
        return userService.findById(id);
    }

    @POST
    public Response create(User user) {
        userService.save(user);
        return Response.status(Response.Status.CREATED)
                .entity(user).build();
    }

    @PUT
    @Path("/{id}")
    public User update(@PathParam("id") Long id, User user) {
        user.setId(id);
        return userService.update(user);
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("id") Long id) {
        userService.delete(id);
        return Response.noContent().build();
    }
}
```

# 4. 参数绑定细节

## 4.1 路径与查询参数

```
GET /users/1001?page=2&size=20

@Path("/{id}")
public User get(@PathParam("id") Long id,
                @QueryParam("page") @DefaultValue("1") int page,
                @QueryParam("size") @DefaultValue("20") int size)
```

## 4.2 正则约束路径

```java
@Path("/{id: \\d+}")   // id 只能是数字
```

# 5. 响应构建（Response）

```java
// 200 OK
return Response.ok(user).build();

// 201 Created
return Response.status(Status.CREATED).entity(user).build();

// 202 Accepted
return Response.accepted().build();

// 204 No Content
return Response.noContent().build();

// 302 重定向
return Response.seeOther(URI.create("/login")).build();

// 404
return Response.status(Status.NOT_FOUND).build();
```

# 6. 异常处理

## 6.1 自定义异常与映射

```java
public class NotFoundException extends RuntimeException { }

@Provider
public class NotFoundMapper implements ExceptionMapper<NotFoundException> {
    @Override
    public Response toResponse(NotFoundException e) {
        return Response.status(Status.NOT_FOUND)
                .entity(Map.of("error", "资源不存在"))
                .type(MediaType.APPLICATION_JSON)
                .build();
    }
}
```

## 6.2 WebApplicationException

```java
throw new WebApplicationException(
        Response.status(Status.BAD_REQUEST)
                .entity(Map.of("error", "参数错误"))
                .build());
```

# 7. 内容协商与 Provider

## 7.1 `@Produces` / `@Consumes`

```java
@GET
@Produces({MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML})
public User get(@PathParam("id") Long id) {
    // 根据 Accept 头自动选择 JSON/XML
}
```

## 7.2 MessageBodyReader/Writer

- JAX-RS 通过 `MessageBodyReader` / `MessageBodyWriter` 实现对象与消息体互转。
- Jersey 内置 JSON（Jackson/JAXB）与 XML Provider。

```java
@Provider
@Produces("application/x-myformat")
public class MyWriter implements MessageBodyWriter<MyType> {
    @Override
    public void writeTo(MyType t, Class<?> type, Type genericType,
            Annotation[] annotations, MediaType mediaType,
            MultivaluedMap<String, Object> httpHeaders,
            OutputStream entityStream) throws IOException {
        entityStream.write(t.toString().getBytes());
    }
}
```

# 8. 子资源定位

```java
@Path("/users")
public class UserResource {
    @Path("{id}/orders")
    public OrderResource orderResource() {
        return new OrderResource();   // 子资源
    }
}
```

# 9. JAX-RS 与 CDI 集成

```java
@Path("/orders")
@RequestScoped
public class OrderResource {

    @Inject
    private OrderService orderService;

    @Context
    private UriInfo uriInfo;
}
```

- 资源类默认每个请求创建（`@RequestScoped`）。
- 可注入 EJB、CDI Bean、JPA EntityManager。

# 10. 应用引导

## 10.1 编程式（Application 子类）

```java
@ApplicationPath("/api")
public class MyApplication extends Application {
    @Override
    public Set<Class<?>> getClasses() {
        return Set.of(UserResource.class, NotFoundMapper.class);
    }
}
```

## 10.2 注解扫描

- 默认自动扫描 `@Path` 资源与 `@Provider`。
- 需要 `web.xml` 注册（或使用 Jersey Servlet 容器）：

```xml
<servlet>
    <servlet-name>Jersey</servlet-name>
    <servlet-class>org.glassfish.jersey.servlet.ServletContainer</servlet-class>
    <init-param>
        <param-name>jersey.config.server.provider.packages</param-name>
        <param-value>com.example.rest</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>Jersey</servlet-name>
    <url-pattern>/api/*</url-pattern>
</servlet-mapping>
```

# 11. JAX-RS vs Spring MVC

| 对比 | JAX-RS | Spring MVC |
|------|--------|-----------|
| 资源注解 | `@Path` | `@RequestMapping` / `@GetMapping` |
| 路径参数 | `@PathParam` | `@PathVariable` |
| 查询参数 | `@QueryParam` | `@RequestParam` |
| 响应包装 | `Response` | `ResponseEntity` |
| 参数对象 | `@BeanParam` | `@ModelAttribute` |
| 异常映射 | `ExceptionMapper` | `@ExceptionHandler` |
| 内容协商 | `@Produces`/`@Consumes` | `produces`/`consumes` |
| 生态 | Jersey/RESTEasy | Spring Boot 全家桶 |

> Spring MVC 与 JAX-RS 思想一致，现代 Java 项目多数用 Spring MVC（`@RestController`），JAX-RS 主要存在于纯 Jakarta EE 应用。

# 12. RESTful 设计规范要点

| 要点 | 说明 |
|------|------|
| 资源命名 | 复数名词：`/users`、`/orders` |
| HTTP 方法语义 | GET 无副作用、POST 创建、PUT 幂等 |
| 状态码使用 | 200/201/204/400/401/403/404/409/500 |
| 版本控制 | `/api/v1/users` 或 Header 版本 |
| 过滤/分页 | Query 参数 `?page=1&size=20&sort=id` |
| 错误结构 | 统一 `{code, message, detail}` 结构 |
| 安全 | HTTPS、认证（JWT/OAuth2）、限流、幂等键 |
