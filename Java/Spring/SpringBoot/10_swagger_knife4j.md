# 1. Swagger/OpenAPI 简介

## 1.1 什么是 Swagger

Swagger 是一套围绕 OpenAPI 规范构建的开源工具，用于设计、构建、记录和使用 REST API。主要包含三大组件：

- **Swagger Editor**：基于浏览器的 API 文档编辑器
- **Swagger UI**：将 OpenAPI 规范渲染成交互式文档页面
- **Swagger Codegen**：根据 OpenAPI 规范生成服务端/客户端代码

SpringBoot 项目中，我们通常不直接使用 Swagger 官方工具，而是引入社区集成库（SpringDoc、Knife4j）让框架自动扫描代码生成 OpenAPI 文档。

## 1.2 Swagger 2 vs OpenAPI 3

| 对比项 | Swagger 2 | OpenAPI 3 |
|--------|-----------|-----------|
| 规范版本 | 2.0 | 3.0 / 3.1 |
| 请求体描述 | body/formData 参数 | Request Body 对象 |
| 参数序列化 | 支持有限 | 强大的 content-type 支持 |
| 安全定义 | securityDefinitions | securitySchemes（支持多认证） |
| 回调 | 不支持 | 支持 Callbacks |
| 链接 | 不支持 | 支持 Links |
| 示例 | 单个 example | examples（支持多个示例） |

> **规范与实现的关系**：OpenAPI 是规范（Spec），Swagger 是早期工具集；SpringFox 是 Swagger 2 的 Spring 实现，SpringDoc 是 OpenAPI 3 的 Spring 实现，Knife4j 是基于 SpringDoc 的增强 UI。

## 1.3 SpringBoot 中 API 文档库的演进

| 库 | 适配规范 | 底层 | 状态 | 说明 |
|----|----------|------|------|------|
| SpringFox | Swagger 2 | 自研 | 已停止维护（2020.7） | 早期主流方案，不支持 OpenAPI 3 |
| SpringDoc | OpenAPI 3 | swagger-core | 活跃维护 | Spring 官方推荐，无缝集成 Spring Boot |
| Knife4j | OpenAPI 3（v4） | 基于 SpringDoc | 活跃维护 | 增强 UI + 增强功能，国内使用广泛 |
| Knife4j | Swagger 2（v3） | 基于 SpringFox | 仅维护 | 老项目兼容用，新项目不推荐 |

## 1.4 技术选型建议

```
新项目（Spring Boot 3.x）  →  SpringDoc OpenAPI 3 + Knife4j 4.x（jakarta）
新项目（Spring Boot 2.7.x）→  SpringDoc OpenAPI 3 + Knife4j 4.x
老项目（SpringFox）        →  迁移到 SpringDoc（见第 9 章）

文档生成用 SpringDoc，文档 UI 用 Knife4j——二者互补，不是替代关系。
```

# 2. SpringDoc OpenAPI 3 快速入门

## 2.1 引入依赖

```xml
<!-- SpringDoc OpenAPI 3 -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

> Spring Boot 3.x 项目需使用 `springdoc-openapi-starter-webmvc-ui`（jakarta 版本）。
> Spring Boot 2.x 项目需使用 `springdoc-openapi-ui`（javax 版本）。

## 2.2 基础配置

### 2.2.1 配置类方式

```java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.ExternalDocumentation;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("项目API文档")
                        .version("1.0.0")
                        .description("Spring Boot API接口文档")
                        .contact(new Contact()
                                .name("开发团队")
                                .email("dev@example.com")
                                .url("https://example.com"))
                        .license(new License()
                                .name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("项目Wiki")
                        .url("https://github.com/xxx/wiki"));
    }
}
```

### 2.2.2 application.yml 配置

```yaml
springdoc:
  api-docs:
    enabled: true                        # 启用 /v3/api-docs 端点
    path: /v3/api-docs                   # API文档json路径
  swagger-ui:
    enabled: true                        # 启用 Swagger UI
    path: /swagger-ui.html               # Swagger UI 访问路径
    tags-sorter: alpha                   # 标签排序（alpha=字母排序）
    operations-sorter: alpha             # 接口排序
    display-request-duration: true       # 显示请求耗时
    doc-expansion: list                  # 文档默认展开 (list/full/none)
    filter: true                         # 是否显示过滤搜索
  packages-to-scan: com.example.controller   # 指定扫描的包
  paths-to-match: /api/**                 # 指定匹配的路径
```

## 2.3 验证和访问

启动应用后访问以下地址：

| 地址 | 说明 |
|------|------|
| `http://localhost:8080/swagger-ui/index.html` | Swagger 原生 UI |
| `http://localhost:8080/v3/api-docs` | OpenAPI JSON 文档 |
| `http://localhost:8080/v3/api-docs.yaml` | OpenAPI YAML 文档 |

# 3. Controller 接口文档注解

## 3.1 @Tag — 接口分类标签

注解在 Controller 类上，用于给接口分组：

```java
@RestController
@RequestMapping("/user")
@Tag(name = "用户管理", description = "用户的增删改查接口")
public class UserController {
    // ...
}
```

支持一个类上使用多个 `@Tags` 标注多个分组：

```java
@RestController
@RequestMapping("/user")
@Tags({
    @Tag(name = "用户管理", description = "用户增删改查"),
    @Tag(name = "认证管理", description = "登录注册")
})
public class UserController {
    // ...
}
```

## 3.2 @Operation — 接口说明

注解在 Controller 方法上，描述单个接口：

```java
@GetMapping("/{id}")
@Operation(
    summary = "根据ID查询用户",
    description = "传入用户ID，返回用户详细信息。如果用户不存在返回null。",
    method = "GET"
)
public R<User> getById(@PathVariable Long id) {
    return R.ok(userService.getById(id));
}
```

| 属性 | 说明 |
|------|------|
| `summary` | 接口简短摘要 |
| `description` | 接口详细描述 |
| `method` | HTTP请求方法 |
| `tags` | 接口所属标签（可覆盖类上的 @Tag） |
| `deprecated` | 标记接口已废弃 |
| `hidden` | 是否隐藏 |
| `operationId` | 操作唯一ID |
| `parameters` | 参数列表（可使用 @Parameter 代替） |
| `responses` | 响应列表（可使用 @ApiResponses 代替） |

**标记废弃接口**：

```java
@GetMapping("/old-api")
@Operation(summary = "旧接口", deprecated = true)
public R<String> oldApi() {
    return R.ok("此接口已废弃");
}
```

## 3.3 @Parameter — 参数说明

注解在方法参数上，描述请求参数：

```java
@GetMapping("/list")
@Operation(summary = "分页查询用户列表")
public R<PageResult<User>> list(
        @Parameter(description = "用户名关键字", example = "admin")
        @RequestParam(required = false) String username,

        @Parameter(description = "页码", required = true, example = "1")
        @RequestParam(defaultValue = "1") Integer pageNo,

        @Parameter(description = "每页条数", required = true, example = "10")
        @RequestParam(defaultValue = "10") Integer pageSize) {
    return R.ok(userService.list(username, pageNo, pageSize));
}
```

**路径参数**：

```java
@DeleteMapping("/{id}")
@Operation(summary = "删除用户")
public R<Void> delete(
        @Parameter(description = "用户ID", required = true, example = "1001")
        @PathVariable Long id) {
    userService.deleteById(id);
    return R.ok();
}
```

**请求头参数**：

```java
@GetMapping("/info")
@Operation(summary = "获取用户信息")
public R<User> info(
        @Parameter(description = "认证Token", required = true)
        @RequestHeader("Authorization") String token) {
    return R.ok(userService.getByToken(token));
}
```

**多参数使用 @Parameters**：

```java
@PostMapping("/search")
@Operation(summary = "高级搜索用户")
@Parameters({
    @Parameter(name = "name", description = "用户名"),
    @Parameter(name = "age", description = "年龄"),
    @Parameter(name = "email", description = "邮箱")
})
public R<List<User>> search(@RequestBody Map<String, Object> params) {
    return R.ok(userService.search(params));
}
```

## 3.4 @ApiResponses / @ApiResponse — 响应描述

描述接口的各种响应状态码和返回内容：

```java
@GetMapping("/{id}")
@Operation(summary = "根据ID查询用户")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "查询成功",
            content = @Content(schema = @Schema(implementation = User.class))),
    @ApiResponse(responseCode = "400", description = "参数错误",
            content = @Content(schema = @Schema(implementation = R.class))),
    @ApiResponse(responseCode = "404", description = "用户不存在",
            content = @Content(schema = @Schema(implementation = R.class))),
    @ApiResponse(responseCode = "500", description = "服务器内部错误")
})
public R<User> getById(
        @Parameter(description = "用户ID", required = true)
        @PathVariable Long id) {
    return R.ok(userService.getById(id));
}
```

**快捷方式 — @ApiResponse 直接在 @Operation 中**：

```java
@GetMapping("/{id}")
@Operation(summary = "根据ID查询用户",
    responses = {
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "404", description = "用户不存在")
    })
public R<User> getById(@PathVariable Long id) {
    return R.ok(userService.getById(id));
}
```

## 3.5 @Hidden — 隐藏接口

隐藏某个接口或整个 Controller，使其不出现在文档中：

```java
// 隐藏单个接口
@GetMapping("/internal")
@Hidden
@Operation(summary = "内部接口，不对外暴露")
public R<String> internal() {
    return R.ok("internal data");
}

// 隐藏整个Controller
@RestController
@RequestMapping("/admin")
@Hidden
public class AdminController {
    // 所有方法都不会出现在文档中
}
```

# 4. 实体类文档注解

## 4.1 @Schema 基本使用

注解在实体类和属性上，描述请求/响应的数据结构：

```java
@Data
@Schema(description = "用户实体")
public class User {

    @Schema(description = "用户ID", example = "1001")
    private Long id;

    @Schema(description = "用户名", example = "admin", required = true)
    private String username;

    @Schema(description = "密码", accessMode = Schema.AccessMode.WRITE_ONLY)
    private String password;

    @Schema(description = "邮箱", example = "admin@example.com")
    private String email;

    @Schema(description = "年龄", example = "25", minimum = "0", maximum = "150")
    private Integer age;

    @Schema(description = "性别", allowableValues = {"MALE", "FEMALE", "OTHER"})
    private String gender;

    @Schema(description = "创建时间", accessMode = Schema.AccessMode.READ_ONLY)
    private LocalDateTime createTime;
}
```

## 4.2 @Schema 常用属性详解

| 属性 | 类型 | 说明 |
|------|------|------|
| `description` | String | 字段描述 |
| `example` | String | 示例值 |
| `required` | boolean | 是否必填 |
| `hidden` | boolean | 是否隐藏 |
| `accessMode` | AccessMode | 访问模式：READ_ONLY / WRITE_ONLY / READ_WRITE |
| `minimum` / `maximum` | String | 数值范围 |
| `minLength` / `maxLength` | String | 字符串长度范围 |
| `allowableValues` | String[] | 允许的取值列表 |
| `pattern` | String | 正则表达式校验 |
| `defaultValue` | String | 默认值 |
| `nullable` | boolean | 是否可为 null |
| `implementation` | Class | 指定实现类（用于多态） |

**枚举字段**：

```java
@Schema(description = "用户状态")
public enum UserStatus {
    @Schema(description = "启用")
    ACTIVE,

    @Schema(description = "禁用")
    DISABLED,

    @Schema(description = "已删除")
    DELETED;
}

@Data
@Schema(description = "用户实体")
public class User {
    @Schema(description = "用户状态", example = "ACTIVE")
    private UserStatus status;
}
```

## 4.3 嵌套对象与泛型

```java
@Data
@Schema(description = "统一响应结果")
public class R<T> {

    @Schema(description = "状态码", example = "200")
    private Integer code;

    @Schema(description = "提示信息", example = "操作成功")
    private String msg;

    @Schema(description = "响应数据")
    private T data;
}
```

**嵌套对象示例**：

```java
@Data
@Schema(description = "用户详情（包含地址信息）")
public class UserDetail {

    @Schema(description = "用户ID")
    private Long id;

    @Schema(description = "用户名")
    private String username;

    @Schema(description = "地址信息")
    private Address address;   // Swagger会自动解析嵌套类型

    @Schema(description = "角色列表")
    private List<Role> roles;  // 集合类型也会自动解析
}

@Data
@Schema(description = "地址")
public class Address {
    @Schema(description = "省份", example = "广东省")
    private String province;

    @Schema(description = "城市", example = "深圳市")
    private String city;

    @Schema(description = "详细地址", example = "南山区科技园")
    private String detail;
}

@Data
@Schema(description = "角色")
public class Role {
    @Schema(description = "角色ID")
    private Long id;

    @Schema(description = "角色名称", example = "管理员")
    private String name;
}
```

# 5. 全局参数配置

## 5.1 添加全局请求头（如 Token）

在配置类中使用 `OperationCustomizer` 或全局 `Parameter`：

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info().title("项目API文档").version("1.0.0"));
    }

    /**
     * 给所有接口添加公共请求头参数（如 Token）
     */
    @Bean
    public OperationCustomizer operationCustomizer() {
        return (operation, handlerMethod) -> operation
                .addParametersItem(
                        new Parameter()
                                .name("Authorization")
                                .description("认证Token")
                                .in(ParameterIn.HEADER.toValue())
                                .required(false)
                                .example("Bearer eyJhbGciOiJIUzI1NiJ9...")
                                .schema(new StringSchema())
                );
    }
}
```

> 若使用 Knife4j，全局参数的输入值会**持久化缓存**在浏览器，无需每次重新输入（见 8.4 `enable-request-cache`）。

## 5.2 添加全局响应状态码

```java
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
            .info(new Info().title("项目API文档").version("1.0.0"))
            // 添加全局错误响应
            .components(new Components()
                    .addResponses("400", new ApiResponse().description("请求参数错误"))
                    .addResponses("401", new ApiResponse().description("未授权"))
                    .addResponses("403", new ApiResponse().description("无权限"))
                    .addResponses("500", new ApiResponse().description("服务器内部错误"))
            );
}
```

# 6. 认证与鉴权集成

## 6.1 JWT Bearer Token 配置

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info().title("项目API文档").version("1.0.0"))
                // 添加安全方案
                .addSecurityItem(new SecurityRequirement().addList("BearerToken"))
                .components(new Components()
                        .addSecuritySchemes("BearerToken",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")
                                        .description("在下方输入JWT Token（不含Bearer前缀）")
                        )
                );
    }
}
```

配置后，Swagger UI 右上角会出现 `Authorize` 按钮，点击后可输入 Token。所有标记了安全方案的接口会自动携带 `Authorization: Bearer xxx` 请求头。

**按接口指定认证**：

```java
@GetMapping("/profile")
@Operation(summary = "获取个人信息", security = { @SecurityRequirement(name = "BearerToken") })
public R<User> profile() {
    // 此接口需要认证
    return R.ok(currentUser());
}
```

## 6.2 Basic Auth 配置

```java
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
            .info(new Info().title("项目API文档").version("1.0.0"))
            .addSecurityItem(new SecurityRequirement().addList("BasicAuth"))
            .components(new Components()
                    .addSecuritySchemes("BasicAuth",
                            new SecurityScheme()
                                    .type(SecurityScheme.Type.HTTP)
                                    .scheme("basic")
                                    .description("使用用户名和密码进行Basic认证")
                    )
            );
}
```

## 6.3 OAuth2 配置

```java
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
            .info(new Info().title("项目API文档").version("1.0.0"))
            .addSecurityItem(new SecurityRequirement().addList("OAuth2"))
            .components(new Components()
                    .addSecuritySchemes("OAuth2",
                            new SecurityScheme()
                                    .type(SecurityScheme.Type.OAUTH2)
                                    .flows(new OAuthFlows()
                                            .password(new OAuthFlow()
                                                    .tokenUrl("http://localhost:8080/oauth/token")
                                                    .scopes(new Scopes()
                                                            .addString("read", "读权限")
                                                            .addString("write", "写权限")
                                                    )
                                            )
                                    )
                    )
            );
}
```

# 7. API 分组

## 7.1 GroupedOpenApi 分组配置

当项目接口较多时，可以按模块分组生成多个文档：

```java
@Configuration
public class OpenApiGroupConfig {

    /**
     * 用户模块分组
     */
    @Bean
    public GroupedOpenApi userApiGroup() {
        return GroupedOpenApi.builder()
                .group("用户管理")
                .displayName("用户管理模块")
                .packagesToScan("com.example.controller.user")
                .build();
    }

    /**
     * 订单模块分组
     */
    @Bean
    public GroupedOpenApi orderApiGroup() {
        return GroupedOpenApi.builder()
                .group("订单管理")
                .displayName("订单管理模块")
                .packagesToScan("com.example.controller.order")
                .build();
    }

    /**
     * 系统管理分组（按路径匹配）
     */
    @Bean
    public GroupedOpenApi adminApiGroup() {
        return GroupedOpenApi.builder()
                .group("系统管理")
                .displayName("系统管理模块")
                .pathsToMatch("/admin/**")
                .build();
    }

    /**
     * 所有接口分组
     */
    @Bean
    public GroupedOpenApi allApiGroup() {
        return GroupedOpenApi.builder()
                .group("全部接口")
                .displayName("全部接口")
                .packagesToScan("com.example.controller")
                .build();
    }
}
```

配置后 Swagger UI / Knife4j 右上角会出现下拉框切换不同分组的文档。

## 7.2 分组配置项说明

| 配置方法 | 说明 |
|----------|------|
| `group(String)` | 分组标识（唯一，下拉选项的 value） |
| `displayName(String)` | 分组显示名称（下拉选项的 label） |
| `packagesToScan(String)` | 扫描的包路径 |
| `pathsToMatch(String)` | 匹配的接口路径（支持 Ant 风格通配符） |
| `pathsToExclude(String)` | 排除的接口路径 |
| `packagesToExclude(String)` | 排除的包路径 |
| `addOpenApiCustomizer()` | 添加自定义配置 |

# 8. Knife4j 增强文档 UI

## 8.1 Knife4j 简介与定位

Knife4j（发音同"奈夫4j"，原 swagger-bootstrap-ui）是国人 xiaoymin 开发的一套 **Swagger/OpenAPI 增强文档 UI**。

**定位澄清**：

```
Knife4j 不负责"生成"OpenAPI 文档——文档仍由底层 springdoc 扫描生成。
Knife4j 只做两件事：
  1. 提供更美观、更强大的文档 UI（访问路径 /doc.html）
  2. 提供一堆原生 Swagger UI 没有的增强功能
     （离线导出、全局参数持久化、接口排序、分组、权限控制、多文件上传、i18n 等）
```

**与 SpringDoc 的关系**：Knife4j 4.x 构建在 springdoc 之上。引入 Knife4j starter 后会自动装配，原有 `OpenAPI` Bean、`GroupedOpenApi`、注解全部继续生效，只是多了一个 `/doc.html` 入口。

**增强能力一览**：

| 能力 | Swagger UI | Knife4j |
|------|-----------|---------|
| 界面风格 | 左参数右响应 | 更美观、国产化设计 |
| 离线导出 | 不支持 | Markdown / Word / HTML / OpenAPI |
| 全局参数 | 需手动添加 | 持久化缓存（如 Token） |
| API 分组 | 基础分组 | 可视化分组、排序 |
| 接口排序 | 按方法名 | 自定义排序（`@ApiOperationSupport`） |
| 访问权限 | 无 | 生产环境屏蔽调试 + Basic 密码 |
| 多文件上传 | 不友好 | 原生支持 |
| 调试增强 | 基础 | 复制 cURL、AfterScript 等 |

## 8.2 版本选型与依赖

### 8.2.1 版本对照

| Knife4j 版本 | 底层实现 | Spring Boot | 规范 | 依赖 artifact |
|--------------|----------|-------------|------|---------------|
| 4.x | SpringDoc | 3.x | OpenAPI 3 | `knife4j-openapi3-jakarta-spring-boot-starter` |
| 4.x | SpringDoc | 2.x | OpenAPI 3 | `knife4j-openapi3-spring-boot-starter` |
| 3.x | SpringFox | 2.x | Swagger 2 | `knife4j-spring-boot-starter`（legacy） |

> **选型**：新项目（Spring Boot 3.x）用第一行。仔细区分 **jakarta**——Spring Boot 3 使用 Jakarta EE 9+（`jakarta.*` 包名），依赖也带 `jakarta` 后缀，用错会出现类找不到或文档空白。

### 8.2.2 依赖引入

```xml
<!-- Spring Boot 3.x：jakarta 版本 -->
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>
    <version>4.5.0</version>
</dependency>

<!-- Spring Boot 2.x：非 jakarta 版本 -->
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-spring-boot-starter</artifactId>
    <version>4.5.0</version>
</dependency>
```

> Knife4j 4.x 的 starter 内部已经传递引入了 `springdoc-openapi-starter-webmvc-ui`，通常**无需再单独声明 springdoc 依赖**；若项目已单独引入，注意版本兼容。

## 8.3 快速集成与访问

引入依赖后**零配置即可使用**，原有的 `OpenApiConfig` 与接口注解无需改动：

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info().title("项目API文档").version("1.0.0")
                        .description("Knife4j 增强文档"));
    }
}
```

访问地址：

| 地址 | 说明 |
|------|------|
| `http://localhost:8080/doc.html` | **Knife4j 增强 UI**（推荐） |
| `http://localhost:8080/swagger-ui/index.html` | Swagger 原生 UI（与 Knife4j 共存） |
| `http://localhost:8080/v3/api-docs` | OpenAPI JSON |
| `http://localhost:8080/v3/api-docs/default` | 指定分组的 JSON |

> `/doc.html` 是 Knife4j 的核心入口。它读取的仍是 `/v3/api-docs` 的 JSON，只是渲染层替换成了增强 UI。

## 8.4 knife4j.setting 配置项全表

```yaml
knife4j:
  enable: true                               # 启用 Knife4j 增强
  setting:
    language: zh_cn                          # 界面语言（zh_cn / en）
    enable-swagger-models: true              # 是否显示"Swagger 模型"标签页
    enable-document-manage: true             # 是否显示"文档管理"（离线导出入口）
    enable-version: false                    # 是否在页脚显示 Knife4j 版本号
    enable-request-cache: true               # ★ 请求参数缓存（全局参数持久化）
    enable-filter-multipart-apis: false      # 是否过滤 Multipart 接口
    enable-footer: false                     # 是否显示默认页脚
    enable-footer-custom: true               # 是否自定义页脚
    footer-custom-content: "Copyright © 2024 开发团队"
    enable-group: true                       # 是否显示分组下拉
    enable-open-api: true                    # 是否显示 OpenAPI 原始 JSON
    enable-after-script: true                # 是否启用调试的 AfterScript
    enable-host: false                       # 是否显示 Host 信息
    enable-home-custom: false                # 是否自定义首页
    home-custom-path: classpath:markdown/home.md   # 自定义首页 Markdown 路径
    enable-dynamic-parameter: true           # 是否启用动态参数调试
    enable-debug: true                       # 是否允许在线调试
```

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `language` | `zh_cn` | 界面语言 |
| `enable-swagger-models` | `true` | 显示模型定义标签页 |
| `enable-document-manage` | `true` | 显示文档管理（离线导出） |
| `enable-version` | `true` | 页脚显示 Knife4j 版本 |
| `enable-request-cache` | `true` | **缓存全局参数**，刷新后不丢 Token |
| `enable-filter-multipart-apis` | `false` | 过滤文件上传类接口 |
| `enable-footer` | `false` | 默认页脚 |
| `enable-footer-custom` | `false` | 自定义页脚 |
| `footer-custom-content` | 空 | 自定义页脚内容 |
| `enable-group` | `true` | 分组下拉框 |
| `enable-open-api` | `true` | 显示 OpenAPI 原始 JSON |
| `enable-after-script` | `true` | 请求后脚本（生产应关闭） |
| `enable-home-custom` | `false` | 自定义首页 |
| `enable-debug` | `true` | 在线调试开关（生产应关闭） |

## 8.5 接口与 Tag 排序

### 8.5.1 接口排序 @ApiOperationSupport(order)

`@ApiOperationSupport` 来自 `com.github.xiaoymin.knife4j.annotations`，`order` 越小越靠前：

```java
import com.github.xiaoymin.knife4j.annotations.ApiOperationSupport;

@RestController
@RequestMapping("/user")
@Tag(name = "用户管理")
public class UserController {

    @ApiOperationSupport(order = 1)
    @GetMapping("/{id}")
    @Operation(summary = "查询用户")
    public R<User> getById(@PathVariable Long id) {
        return R.ok(userService.getById(id));
    }

    @ApiOperationSupport(order = 2)
    @PostMapping
    @Operation(summary = "新增用户")
    public R<Void> add(@RequestBody User user) {
        userService.add(user);
        return R.ok();
    }
}
```

**忽略/包含嵌套参数**（常用于 DTO 字段过多时）：

```java
@ApiOperationSupport(order = 3, ignoreParameters = {"user.id", "user.createTime"})
@PostMapping("/save")
@Operation(summary = "保存用户")
public R<Void> save(@RequestBody UserDTO dto) { ... }
```

### 8.5.2 Tag 排序 @ApiSort

`@ApiSort` 标注在 Controller 类上，控制分组（Tag）的排序，数字越小越靠前：

```java
import com.github.xiaoymin.knife4j.annotations.ApiSort;

@ApiSort(1)
@Tag(name = "用户管理")
@RestController
@RequestMapping("/user")
public class UserController { ... }
```

## 8.6 离线文档导出

**方式一：UI 导出（推荐）**

访问 `http://localhost:8080/doc.html`，右上角「文档管理」→「离线文档」，可导出：

```
Markdown   导出为 .md 文件
HTML       导出为单页 HTML
Word       导出为 .doc 文件
OpenAPI    导出标准 OpenAPI JSON（可用于代码生成）
```

**方式二：直接下载 OpenAPI JSON**

```bash
# 原始 OpenAPI 文档，可交给 OpenAPI Generator 生成客户端代码
curl http://localhost:8080/v3/api-docs -o openapi.json
```

> 导出的 Markdown/Word 文档适合交付给前端或作为离线接口手册；OpenAPI JSON 适合接入 CI 做代码生成、契约测试。

## 8.7 全局参数与鉴权

- **全局参数**：沿用第 5 章的 `OperationCustomizer` 给所有接口加公共头；Knife4j 会把用户填写的参数值**缓存在本地**（`enable-request-cache: true`），调试时无需反复输入 Token。
- **鉴权**：沿用第 6 章的 `SecurityScheme`（JWT/Basic/OAuth2）。Knife4j 界面会在顶部显示「Authorize」区域，输入后自动附加到请求。
- **分组级参数**：可在 `GroupedOpenApi` 的 `addOpenApiCustomizer` 中为某个分组单独追加参数。

```java
@Bean
public GroupedOpenApi userApiGroup() {
    return GroupedOpenApi.builder()
            .group("用户管理")
            .packagesToScan("com.example.controller.user")
            .addOpenApiCustomizer(openApi ->
                    openApi.addSecurityItem(new SecurityRequirement().addList("BearerToken")))
            .build();
}
```

## 8.8 访问权限控制

### 8.8.1 生产环境屏蔽调试（production）

```yaml
knife4j:
  production: true    # ★ 生产环境开启
```

`production: true` 会**只保留文档查看**，屏蔽：

```
- 在线调试（调试按钮消失）
- AfterScript 脚本
- 文档管理（离线导出入口）
- 个性化设置（全局参数、缓存等）
- OpenAPI 原始文档的编辑能力
```

### 8.8.2 登录密码保护（basic）

```yaml
knife4j:
  basic:
    enable: true
    username: admin
    password: 123456    # 访问 /doc.html 需先登录
```

> ⚠ Knife4j 自带的 basic 密码保护强度有限（会话级），生产环境建议**叠加 Spring Security**（见 8.10）对 `/doc.html`、`/v3/api-docs/**` 做统一鉴权，或直接用 profile 关闭文档端点。

## 8.9 UI 个性化

### 8.9.1 自定义 Footer

```yaml
knife4j:
  setting:
    enable-footer: false
    enable-footer-custom: true
    footer-custom-content: "Copyright © 2024 某某公司 | 内部文档"
```

### 8.9.2 自定义首页

准备一份 Markdown 放到 `resources/markdown/home.md`：

```markdown
# 欢迎使用 XX 系统 API 文档
- 环境：开发环境
- 联系人：dev@example.com
```

```yaml
knife4j:
  setting:
    enable-home-custom: true
    home-custom-path: classpath:markdown/home.md
```

### 8.9.3 其他 UI 开关

通过 8.4 的 `knife4j.setting.*` 控制显示/隐藏各模块（模型、分组、OpenAPI 原始 JSON、版本号等）。若需要更深度定制（自定义 JS/CSS），可通过覆盖静态资源或使用 `WebMvcConfigurer` 注册自定义资源处理器实现。

## 8.10 与 Spring Security 集成

引入 Spring Security 后，`/doc.html`、`/webjars/**`、`/v3/api-docs/**` 等默认会被拦截，需放行：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                // 放行 Knife4j & Swagger
                .requestMatchers(
                        "/doc.html",
                        "/webjars/**",
                        "/v3/api-docs/**",
                        "/swagger-ui/**",
                        "/swagger-ui.html",
                        "/swagger-resources/**"
                ).permitAll()
                .anyRequest().authenticated()
        );
        return http.build();
    }
}
```

> 若还配置了自定义 `HandlerInterceptor`，也要在 `excludePathPatterns` 中排除上述路径（见 11.1）。

## 8.11 注解说明与纠错

**关键澄清**：Knife4j **4.x 没有自己的注解体系**——它复用 springdoc 的 OpenAPI 3 注解（`@Tag`/`@Operation`/`@Parameter`/`@Schema`），只额外提供了**排序相关**的 `@ApiOperationSupport`、`@ApiSort`。而在 Knife4j **3.x（基于 SpringFox）** 中，才使用 `@Api`/`@ApiOperation`/`@ApiModelProperty` 这类注解。

> ⚠ 常见误区：把 `@Api`、`@ApiOperation`、`@ApiParam`、`@ApiModel` 当成 "Knife4j 注解"。它们其实是 **SpringFox（Swagger 2）** 的注解。在 Spring Boot 3 + Knife4j 4.x 下，这些注解**不会生效**，必须改用 OpenAPI 3 注解。

**注解对照（务必区分两代）**：

| 需求 | OpenAPI 3（SpringDoc / Knife4j 4.x） | Swagger 2（SpringFox / Knife4j 3.x） |
|------|--------------------------------------|--------------------------------------|
| 分组标签 | `@Tag(name=...)` | `@Api(tags=...)` |
| 接口描述 | `@Operation(summary=...)` | `@ApiOperation(value=...)` |
| 参数描述 | `@Parameter(description=...)` | `@ApiParam(value=...)` |
| 模型描述 | `@Schema(description=...)` | `@ApiModel` / `@ApiModelProperty` |
| 多参数 | `@Parameters` | `@ApiImplicitParams` / `@ApiImplicitParam` |
| 隐藏 | `@Hidden` | `@ApiIgnore` |
| 响应 | `@ApiResponses` / `@ApiResponse` | `@ApiResponses` / `@ApiResponse` |
| Knife4j 排序（独有） | `@ApiOperationSupport(order)` / `@ApiSort` | `@ApiOperationSupport(order)` / `@ApiSort` |

## 8.12 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `/doc.html` 404 | 未引入 Knife4j 依赖 | 添加对应版本的 starter |
| `/doc.html` 404（依赖已加） | 静态资源被拦截 | 放行 `/doc.html`、`/webjars/**`、`/v3/api-docs/**` |
| 页面空白 / 增强不生效 | 依赖与 Spring Boot 版本不匹配 | Boot 3 用 `jakarta` 版，Boot 2 用非 jakarta 版 |
| 排序不生效 | 注解用错版本 | 4.x 用 `@ApiOperationSupport`，勿用 SpringFox 注解 |
| `@Api` 等注解不生效 | 误用 SpringFox 注解 | 改用 OpenAPI 3 的 `@Tag`/`@Operation` |
| 生产环境暴露调试入口 | 未做权限控制 | `knife4j.production=true` + Spring Security / profile 关闭 |
| 全局参数每次都要重填 | 未开启缓存 | 设置 `knife4j.setting.enable-request-cache: true` |

# 9. SpringFox (Swagger 2) 兼容与迁移

## 9.1 SpringFox 简介

SpringFox 是 SpringBoot 早期的 Swagger 2 实现，已于 2020 年停止维护，且不兼容 Spring Boot 3.x。若项目仍在使用 SpringFox，建议迁移到 SpringDoc（+ 可选 Knife4j）。

## 9.2 SpringFox → SpringDoc 注解对照表

| SpringFox (Swagger 2) | SpringDoc / Knife4j 4.x (OpenAPI 3) | 说明 |
|------------------------|-------------------------------------|------|
| `@Api(tags = "xxx")` | `@Tag(name = "xxx")` | Controller 分类 |
| `@ApiOperation(value = "xxx")` | `@Operation(summary = "xxx")` | 接口描述 |
| `@ApiParam(value = "xxx")` | `@Parameter(description = "xxx")` | 参数描述 |
| `@ApiImplicitParams` | `@Parameters` | 多参数描述 |
| `@ApiImplicitParam` | `@Parameter` | 单个参数描述 |
| `@ApiModel(value = "xxx")` | `@Schema(description = "xxx")` | 模型描述 |
| `@ApiModelProperty(value = "xxx")` | `@Schema(description = "xxx")` | 属性描述 |
| `@ApiIgnore` | `@Hidden` | 隐藏接口/字段 |
| `@ApiResponses` | `@ApiResponses` | 一致 |
| `@ApiResponse` | `@ApiResponse` | 一致 |
| `Docket` | `GroupedOpenApi` | 分组配置 |

## 9.3 依赖替换

```xml
<!-- SpringFox (旧) - 移除 -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>

<!-- SpringDoc (新) - 替换 -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

## 9.4 配置迁移示例

**SpringFox 的 Docket 配置**：

```java
@Configuration
public class Swagger2Config {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("项目API文档")
                .version("1.0.0")
                .description("接口文档")
                .build();
    }
}
```

**迁移到 SpringDoc**：

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("项目API文档")
                        .version("1.0.0")
                        .description("接口文档"));
    }

    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
                .group("default")
                .packagesToScan("com.example.controller")
                .build();
    }
}
```

**访问地址变化**：

| 用途 | SpringFox | SpringDoc / Knife4j |
|------|-----------|---------------------|
| 文档 UI | `/swagger-ui.html` | `/swagger-ui/index.html` 或 `/doc.html` |
| JSON | `/v2/api-docs` | `/v3/api-docs` |

# 10. 常用配置项汇总

## 10.1 springdoc 完整配置项

```yaml
springdoc:
  api-docs:
    enabled: true                 # 是否启用 API docs 端点
    path: /v3/api-docs            # API docs 路径
    groups:
      enabled: true               # 是否启用分组

  swagger-ui:
    enabled: true                 # 是否启用 Swagger UI
    path: /swagger-ui.html        # Swagger UI 路径
    config-url: /v3/api-docs/swagger-config   # 外部配置URL
    urls:                         # 配置多个API文档源
      - name: user-api
        url: /v3/api-docs/user-api-group
      - name: order-api
        url: /v3/api-docs/order-api-group
    tags-sorter: alpha            # Tag排序 (alpha/order)
    operations-sorter: alpha      # 操作排序 (alpha/method/order)
    display-request-duration: true    # 显示请求耗时
    display-operation-id: false       # 显示 operationId
    default-models-expand-depth: 1    # 默认模型展开深度
    deep-linking: true            # 启用深度链接
    doc-expansion: list           # 文档展开级别 (list/full/none)
    filter: true                  # 启用过滤搜索
    show-common-extensions: false # 是否显示通用扩展
    syntax-highlight:
      activated: true             # 启用语法高亮
    try-it-out-enabled: true      # 是否启用 try-it-out
    use-root-path: false          # 是否使用根路径
    urls-primary-name: default    # 主分组名称

  packages-to-scan:               # 指定扫描包路径
  packages-to-exclude:            # 排除包路径
  paths-to-match: /api/**         # 匹配路径
  paths-to-exclude:               # 排除路径
  show-actuator: false            # 是否显示 actuator 端点
  writer-with-default-pretty-printer: true  # 格式化输出

  auto-tag-classes: true          # 自动将类名作为Tag
  default-flat-param-object: false  # 是否扁平化参数对象
  model-and-view-allowed: false   # 是否支持 ModelAndView
  override-with-generic-response: true  # 是否覆盖泛型响应

  cache:
    disabled: false               # 是否禁用缓存

  group-configs:                  # 分组配置（多文档）
    - group: user
      display-name: 用户模块
      packages-to-scan:
        - com.example.controller.user
    - group: order
      display-name: 订单模块
      packages-to-scan:
        - com.example.controller.order
```

## 10.2 knife4j 完整配置项

```yaml
knife4j:
  enable: true                     # 启用 Knife4j 增强
  production: false                # 生产环境：true 则屏蔽调试/文档管理等
  basic:
    enable: false                  # 启用登录密码保护
    username: admin
    password: 123456
  setting:
    language: zh_cn
    enable-swagger-models: true
    enable-document-manage: true
    enable-version: false
    enable-request-cache: true
    enable-filter-multipart-apis: false
    enable-footer: false
    enable-footer-custom: true
    footer-custom-content: "Copyright © 2024 开发团队"
    enable-group: true
    enable-open-api: true
    enable-after-script: true
    enable-host: false
    enable-home-custom: false
    home-custom-path: classpath:markdown/home.md
    enable-dynamic-parameter: true
    enable-debug: true
```

## 10.3 按环境隔离文档

```yaml
# application-dev.yml (开发环境 - 开启文档)
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    enabled: true
knife4j:
  enable: true
  production: false

# application-prod.yml (生产环境 - 关闭文档或仅可查看)
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
knife4j:
  enable: false
```

**使用 profile 条件编写配置类**：

```java
@Configuration
@Profile({"dev", "test"})   // 仅在开发和测试环境生效
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info().title("项目API文档(测试环境)").version("1.0.0"));
    }
}
```

> 生产环境若仍想保留只读文档，可保留 `knife4j.enable=true` 并设置 `production=true`、同时关闭 springdoc 的调试能力；若完全不需要，则用 profile 直接关闭文档端点。

# 11. 常见问题与注意事项

## 11.1 静态资源被拦截导致 404

如果项目自定义了拦截器或 Spring Security 配置，Swagger/Knife4j 的静态资源可能被拦截。

**Spring Security 放行配置**：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                // 放行 Swagger & Knife4j
                .requestMatchers(
                        "/swagger-ui/**",
                        "/swagger-ui.html",
                        "/v3/api-docs/**",
                        "/doc.html",
                        "/webjars/**"
                ).permitAll()
                .anyRequest().authenticated()
        );
        return http.build();
    }
}
```

**自定义拦截器放行**：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns(
                        "/swagger-ui/**",
                        "/swagger-ui.html",
                        "/v3/api-docs/**",
                        "/doc.html",
                        "/webjars/**"
                );
    }
}
```

## 11.2 与 Spring Security OAuth2 资源服务器集成

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info().title("项目API文档").version("1.0.0"))
                .addSecurityItem(new SecurityRequirement().addList("OAuth2"))
                .components(new Components()
                        .addSecuritySchemes("OAuth2",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.OAUTH2)
                                        .flows(new OAuthFlows()
                                                .authorizationCode(new OAuthFlow()
                                                        .authorizationUrl("/oauth2/authorize")
                                                        .tokenUrl("/oauth2/token")
                                                        .scopes(new Scopes()
                                                                .addString("read", "读权限")
                                                                .addString("write", "写权限")
                                                        )
                                                )
                                        )
                        )
                );
    }
}
```

## 11.3 文件上传接口文档化

```java
@PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
@Operation(summary = "上传文件")
public R<String> upload(
        @Parameter(description = "文件", required = true)
        @RequestPart("file") MultipartFile file) {
    return R.ok("上传成功: " + file.getOriginalFilename());
}
```

多文件上传：

```java
@PostMapping(value = "/upload/multi", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
@Operation(summary = "批量上传文件")
public R<String> uploadMulti(
        @Parameter(description = "文件列表", required = true)
        @RequestPart("files") List<MultipartFile> files) {
    return R.ok("成功上传 " + files.size() + " 个文件");
}
```

> Knife4j 对多文件上传界面做了增强，可直接选择多个文件调试。

## 11.4 @RequestParam 与 @PathVariable 注意事项

SpringDoc 能自动识别 `@RequestParam` 和 `@PathVariable`，无需显式添加 `@Parameter` 注解即可生成文档。但显式添加 `@Parameter` 可以提供更详细的中文说明：

```java
// 自动识别，但文档中参数名为"id"，无中文说明
@GetMapping("/{id}")
public R<User> getById(@PathVariable Long id) { ... }

// 显式添加 Parameter 注解，文档中有中文说明和示例值
@GetMapping("/{id}")
@Operation(summary = "查询用户")
public R<User> getById(
        @Parameter(description = "用户ID", required = true, example = "1001")
        @PathVariable Long id) { ... }
```

## 11.5 使用 @ParameterObject 简化分页参数

对于分页查询中参数较多的情况，可以封装为一个对象并用 `@ParameterObject` 注解：

```java
@Data
@Schema(description = "用户分页查询条件")
public class UserPageQuery {

    @Parameter(description = "用户名关键字")
    private String username;

    @Parameter(description = "邮箱")
    private String email;

    @Parameter(description = "页码", example = "1")
    private Integer pageNo = 1;

    @Parameter(description = "每页条数", example = "10")
    private Integer pageSize = 10;
}

@GetMapping("/page")
@Operation(summary = "分页查询用户")
public R<PageResult<User>> page(@ParameterObject UserPageQuery query) {
    return R.ok(userService.page(query));
}
```

## 11.6 分组后访问文档 404

使用 `GroupedOpenApi` 分组后，Swagger UI 默认访问的是 `default` 分组。如果没定义 `default` 分组，需要手动切换右上角下拉框，或者在 URL 上指定分组参数：

```
# 访问指定分组
http://localhost:8080/swagger-ui/index.html?group=用户管理
http://localhost:8080/doc.html#/用户管理

# 或者在配置中指定默认显示的分组
springdoc.swagger-ui.urls-primary-name=用户管理
```

## 11.7 Knife4j 相关常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `doc.html` 打不开 | 依赖缺失或版本不匹配 | 确认引入 starter，Boot 3 用 jakarta 版 |
| 文档 UI 能开但无接口 | 分组未匹配到 | 检查 `packages-to-scan` / `paths-to-match` |
| 生产环境不想暴露调试 | 未配置 | `knife4j.production=true`，或用 profile 关闭 |
| 调试时要反复输入 Token | 未开启缓存 | `enable-request-cache: true` |
| 排序注解无效 | 误用 SpringFox 注解 | 用 `@ApiOperationSupport` / `@ApiSort` |
| 中文乱码 / 界面英文 | 未设置语言 | `knife4j.setting.language: zh_cn` |

## 11.8 常见注意事项总结

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Swagger UI 页面空白 | 静态资源被拦截 | 放行 `/swagger-ui/**`、`/v3/api-docs/**`、`/doc.html`、`/webjars/**` |
| 接口分组后 404 | 未指定默认分组 | 配置 `urls-primary-name` 或 URL 传参 |
| 注解不生效 | 未开启相关配置 / 注解版本不对 | 检查 `springdoc.api-docs.enabled=true`，Spring Boot 3 用 OpenAPI 3 注解 |
| Knife4j 增强不生效 | 缺少依赖 | 添加对应版本 `knife4j-...-spring-boot-starter` |
| 泛型类型丢失 | 编译时类型擦除 | 使用 `@ApiResponse` 显式指定 `implementation` |
| 文件上传无文档 | 未指定 consumes | 添加 `consumes = MediaType.MULTIPART_FORM_DATA_VALUE` |
| 生产环境暴露文档 | 未做环境隔离 | profile 关闭端点，或 `knife4j.production=true` + 鉴权 |
