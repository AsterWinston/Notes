# 1. 知识点
- **API文档价值**: 前后端协作契约、接口调试工具、新人快速了解系统、代码生成基础(根据文档生成客户端代码)
- **Swagger/OpenAPI**: 最早由SmartBear开发的REST API文档工具，后来贡献给OpenAPI Initiative成为OpenAPI规范(OpenAPI Specification)
- **OpenAPI 3.x规范**: 描述REST API的标准规范(YAML/JSON格式)。包含info(API信息)、servers(服务器地址)、paths(路径和操作)、components(可复用组件如Schema)、security(安全认证)
- **OpenAPI注解**: @Operation(描述操作摘要和详情)、@ApiResponse(描述响应信息)、@Parameter(描述参数)、@Schema(描述模型)、@RequestBody(描述请求体)
- **Swagger 2 vs SpringFox**: SpringFox是Swagger2的Spring社区实现，已停止维护(最后更新2019年)，不兼容Spring Boot 3.x
- **Springdoc OpenAPI**: Spring Boot 3.x推荐的API文档框架，基于OpenAPI 3.x规范，自动生成文档，代替SpringFox Swagger2
- **Springdoc集成**: 引入springdoc-openapi-starter-webmvc-ui依赖即可，访问/swagger-ui.html查看UI，访问/v3/api-docs获取OpenAPI JSON
- **Springdoc自动配置**: 自动扫描@RequestMapping注解的Controller，根据注解生成文档。零配置即可使用
- **Swagger UI**: Swagger提供的可视化文档界面，可直接在浏览器中查看API列表、请求参数、响应示例，支持在线调用测试
- **Knife4j**: 国人开发的增强版Swagger UI，基于Swagger前端UI加强，功能更丰富
- **Knife4j增强功能**: 离线导出API文档(Markdown/Word/Html/OpenAPI)、全局参数(如统一添加Token)、分组(按模块分组)、权限控制、API排序、多文件上传支持
- **Knife4j集成**: 引入knife4j-openapi3-spring-boot-starter依赖，访问/doc.html即可查看
- **接口文档规范**: 每个接口需明确请求路径(Path)、请求方法(GET/POST/PUT/DELETE)、请求参数(Query/Path/Body)是否必填、响应格式(code/msg/data)、示例值、错误码说明
- **错误码规范**: 统一错误码体系，如1xxxx参数错误、2xxxx业务错误、3xxxx权限错误、9xxxx系统错误。每个错误码配中文解释
- **YApi**: 去哪儿开源的API管理平台，支持可视化接口管理、Mock数据生成、自动化测试、权限管理
- **DocWay**: 类似YApi的接口管理平台，支持接口设计、文档生成、Mock服务
- **ApiFox**: 国产一体化API协作平台(Postman+Swagger+Mock+JMeter四合一)。支持API设计、文档生成、接口调试、Mock数据、自动化测试、团队协作
- **ApiFox特点**: 在线/客户端均可、团队协作、Mock引擎(ApiFox自动生成Mock数据或自定义)、前后置脚本、环境变量、断言、数据驱动
- **ApiFox vs Postman**: Postman侧重于调试和测试，ApiFox增加了API设计和文档管理功能，更适合国内开发团队的全流程协作
- **API文档驱动开发**: 先定义API文档(前后端共同确认接口契约)，再各自开发，减少联调成本。ApiFox/Swagger都支持该模式
- **Springdoc全局配置**: 通过OpenAPI Bean自定义API文档信息(标题/描述/版本/联系人/许可)、全局安全方案、全局参数
- **Springdoc分组**: 使用GroupedOpenApi对API进行分组(按包路径/路径匹配)，不同分组在Swagger UI下拉切换
- **API文档安全**: 线上环境关闭Swagger UI(通过profiles控制，生产环境不启用)，或加Basic Auth/JWT认证

# 2. 面试题

## 2.1 如何从Swagger2(SpringFox)迁移到Springdoc OpenAPI？
SpringFox已停止维护，Spring Boot 3.x不兼容，迁移步骤：
1) 移除SpringFox依赖(springfox-swagger2、springfox-swagger-ui)
2) 添加Springdoc依赖：springdoc-openapi-starter-webmvc-ui
3) 替换注解：@Api → @Tag、@ApiOperation → @Operation、@ApiParam → @Parameter、@ApiModel → @Schema、@ApiModelProperty → @Schema、@ApiIgnore → @Hidden
4) 替换配置Bean：移除Swagger的Docket Bean，添加OpenAPI Bean：
```java
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
        .info(new Info().title("API Docs").version("1.0.0"));
}
```
5) 访问地址变化：swagger-ui.html → /swagger-ui/index.html(自动映射)
6) API JSON地址：v2/api-docs → /v3/api-docs

## 2.2 Knife4j相比原版Swagger UI有什么增强？
| 功能 | Swagger UI | Knife4j |
|------|-----------|---------|
| 界面风格 | 左参数右响应 | 更美观，国产化设计 |
| 离线导出 | 不支持 | Markdown/Word/HTML/OpenAPI |
| 全局参数 | 需手动添加 | 持久化全局参数(如Token) |
| API分组 | 基础分组 | 可视化分组、排序 |
| 接口排序 | 按方法名 | 自定义排序规则 |
| 权限控制 | 无 | 生产环境可加访问密码 |
| 多文件上传 | 不友好 | 原生支持 |
| 调试增强 | 基础 | 支持cURL命令复制等 |

## 2.3 REST API文档应该包含哪些内容？
标准接口文档内容：
1) **接口路径和方法**：POST /api/users
2) **接口描述**：创建用户
3) **请求参数**：参数名、类型、是否必填、说明、示例值
4) **请求头**：Content-Type、Authorization等
5) **请求体示例**：JSON/XML完整示例
6) **响应格式**：统一的响应体结构(code/msg/data)
7) **成功响应示例**：各字段含义注释
8) **错误码说明**：code=10001 用户名已存在/code=10002 邮箱格式错误
9) **业务说明**：接口的特殊逻辑(如幂等性、重试规则、异步处理)

## 2.4 如何控制Swagger在生产环境不可访问？
方案1(Profile控制)：
```java
@Configuration
@Profile({"dev", "test"})  // 非prod环境启用
public class SwaggerConfig { ... }
```

方案2(条件注入)：
```yaml
# application-prod.yml
springdoc:
  swagger-ui:
    enabled: false
  api-docs:
    enabled: false
```

方案3(Security拦截)：配置Spring Security，对/swagger-ui/**和/v3/api-docs/**路径要求admin角色或完全禁用。

方案4(Knife4j)：配置knife4j.production=true，生产环境仅允许访问文档查看，不允许调试。

## 2.5 ApiFox这类一体化平台相比传统Swagger有什么优势？
传统Swagger及其UI侧重API文档生成和在线测试。ApiFox整合了：
1) API设计：可视化的接口设计器，前后端通过同一平台确认接口契约
2) 文档生成：基于设计自动生成在线文档
3) 接口调试：内置Postman级别的请求构造(支持环境变量、前后脚本、断言)
4) Mock服务：根据文档自动生成Mock数据(支持智能Mock如中文名/邮箱规则)
5) 自动化测试：测试用例编排、数据驱动、断言、CI集成
6) 团队协作：多人编辑、评论、版本管理、权限控制
一站式解决API开发全流程(设计→开发→测试→文档→Mock)，减少工具切换。

## 2.6 如何在Springdoc中配置全局安全认证参数？
```java
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
        .info(new Info().title("API").version("1.0"))
        .addSecurityItem(new SecurityRequirement().addList("BearerAuth"))
        .components(new Components()
            .addSecuritySchemes("BearerAuth",
                new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .description("输入JWT Token")));
}
```
配置后Swagger UI自动出现Authorize按钮，填入Token后所有接口自动携带Authorization: Bearer xxx请求头。Knife4j支持类似的全局参数设置且界面更友好。

## 2.7 接口文档中的错误码规范如何设计？
推荐分段设计：
- 1xxxx 参数错误：10001 缺少必填参数、10002 参数格式错误、10003 参数值超出范围
- 2xxxx 业务错误：20001 用户不存在、20002 库存不足、20003 订单状态不允许操作、20004 重复提交
- 3xxxx 权限错误：30001 未登录、30002 Token过期、30003 无权限
- 4xxxx 依赖服务错误：40001 调用支付服务失败、40002 短信发送失败
- 9xxxx 系统错误：90001 系统繁忙、90002 数据库错误、99999 未知错误

统一响应格式：
```json
{
  "code": 10001,
  "msg": "用户名已存在",
  "data": null,
  "traceId": "abc123"
}
```

## 2.8 如何基于API文档生成客户端代码？
1) **Swagger Codegen**：根据OpenAPI JSON/YAML生成多种语言的SDK(Java/Go/TypeScript/Python等)。使用swagger-codegen-cli或swagger-codegen-maven-plugin
2) **OpenAPI Generator**：Swagger Codegen的分叉，更活跃的社区版本，支持更多语言和框架
3) **ApiFox**：支持生成多语言代码(Request代码和模型代码)
4) **前端代码生成**：根据OpenAPI生成TypeScript类型定义和API调用函数(如openapi-typescript-codegen)
流程：后端维护好OpenAPI文档 → 自动或手动生成客户端代码 → 前后端按契约并行开发
