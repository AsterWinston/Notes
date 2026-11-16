# 1. Spring Boot Actuator

Spring Boot Actuator 提供了生产级别的应用监控和管理能力，通过HTTP端点或JMX来暴露应用的运行状况。

## 1.1 引入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## 1.2 常用端点

Actuator提供了很多内置端点，每个端点都有其特定的功能：

| 端点 | 说明 |
|------|------|
| `/actuator/health` | 应用健康状态 |
| `/actuator/info` | 应用基本信息 |
| `/actuator/metrics` | 应用指标信息 |
| `/actuator/metrics/{name}` | 具体指标详情 |
| `/actuator/env` | 环境属性（配置信息） |
| `/actuator/beans` | 容器中所有的Bean |
| `/actuator/mappings` | 所有URL映射 |
| `/actuator/loggers` | 日志配置 |
| `/actuator/loggers/{name}` | 查看/修改指定包的日志级别 |
| `/actuator/threaddump` | 线程转储 |
| `/actuator/heapdump` | 堆转储文件 |
| `/actuator/conditions` | 自动配置条件报告 |

## 1.3 端点暴露配置

默认情况下，只有 `/actuator/health` 端点是通过HTTP暴露的。其他端点需要配置：

```properties
# 暴露所有端点（生产环境慎用）
management.endpoints.web.exposure.include=*

# 只暴露指定的端点
management.endpoints.web.exposure.include=health,info,metrics,loggers

# 排除某些端点
management.endpoints.web.exposure.exclude=env,beans

# 启用shutdown端点
management.endpoint.shutdown.enabled=true
```

## 1.4 健康检查（Health）

访问 `/actuator/health` 返回应用健康状态：

```json
{
    "status": "UP",
    "components": {
        "diskSpace": {"status": "UP", "details": {"free": "..."}},
        "db": {"status": "UP", "details": {"database": "MySQL"}},
        "redis": {"status": "UP"}
    }
}
```

显示健康详情：
```properties
management.endpoint.health.show-details=always
management.endpoint.health.show-components=always
```

## 1.5 自定义健康指示器

实现 `HealthIndicator` 接口：

```java
@Component
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // 模拟检查某个外部服务
        boolean serviceUp = checkExternalService();
        if (serviceUp) {
            return Health.up()
                    .withDetail("message", "外部服务正常")
                    .withDetail("responseTime", "50ms")
                    .build();
        }
        return Health.down()
                .withDetail("message", "外部服务不可用")
                .build();
    }

    private boolean checkExternalService() {
        return true; // 实际项目中去真的检查
    }
}
```

## 1.6 自定义Info端点

在 `application.properties` 中定义：
```properties
management.info.env.enabled=true
management.info.os.enabled=true
management.info.java.enabled=true
```

通过 `InfoContributor` 编程添加：
```java
@Component
public class MyInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("appName", "MyApplication");
        builder.withDetail("version", "1.0.0");
        builder.withDetail("description", "这是一个示例应用");
    }
}
```

## 1.7 调整Actuator路径
```properties
# 修改Actuator的基础路径
management.endpoints.web.base-path=/monitor
# 修改后访问路径变为 /monitor/health, /monitor/info 等

# 修改服务器端口（让监控端口与业务端口分开）
management.server.port=8081
```

# 2. Spring Boot DevTools

DevTools可以提升开发体验，提供自动重启、LiveReload等功能。

## 2.1 引入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

注意：`scope=runtime` 和 `optional=true` 确保DevTools不会传递到其他模块，且打包时不会包含。

## 2.2 自动重启

当classpath下的文件发生变化时，DevTools会自动重启应用（比手动重启快，因为它使用两个ClassLoader，只重启开发类的ClassLoader）。

默认触发重启的目录：`/META-INF/maven`、`/META-INF/resources`、`/resources`、`/static`、`/public`、`/templates`

排除某些目录不触发重启：
```properties
spring.devtools.restart.exclude=static/**,public/**
```

## 2.3 LiveReload

DevTools内置了一个LiveReload服务器，浏览器安装LiveReload插件后，静态资源变化时浏览器会自动刷新。

如果不需要LiveReload：
```properties
spring.devtools.livereload.enabled=false
```

## 2.4 全局配置

在用户目录下创建 `~/.spring-boot-devtools.properties` 文件，所有Spring Boot项目共享该配置：

```properties
# 全局关闭自动重启
spring.devtools.restart.enabled=false
```

## 2.5 IDEA中配合使用

1. File → Settings → Build, Execution, Deployment → Compiler → 勾选 "Build project automatically"
2. 按 `Ctrl+Shift+Alt+/` → Registry → 勾选 "compiler.automake.allow.when.app.running"

这样修改代码后，IDEA自动编译，DevTools检测到变化后自动重启。

# 3. Swagger/Knife4j接口文档

接口文档（Swagger/OpenAPI、SpringDoc、Knife4j 增强 UI）的完整内容已独立成篇，请查阅 [10_swagger_knife4j.md](./10_swagger_knife4j.md)。

> 本章聚焦 Actuator 监控与测试；接口文档的依赖引入、注解用法、分组、鉴权、Knife4j 配置与常见问题均见上述专门章节。

# 4. 测试进阶

## 4.1 MockMvc测试Controller

MockMvc可以模拟HTTP请求，对Controller进行测试，无需启动真实服务器：

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testGetById() throws Exception {
        // 构造GET请求
        mockMvc.perform(get("/user/{id}", 1L)
                        .contentType(MediaType.APPLICATION_JSON))
                // 断言HTTP状态码200
                .andExpect(status().isOk())
                // 断言返回JSON中code字段为200
                .andExpect(jsonPath("$.code").value(200))
                // 断言返回JSON中data不为空
                .andExpect(jsonPath("$.data").isNotEmpty())
                // 打印请求和响应信息
                .andDo(print());
    }

    @Test
    void testAdd() throws Exception {
        // 构造JSON请求体
        String jsonStr = """
                {
                    "username": "test",
                    "password": "123456",
                    "email": "test@example.com"
                }
                """;

        mockMvc.perform(post("/user")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(jsonStr))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.msg").value("新增成功"))
                .andDo(print());
    }

    @Test
    void testAdd_ValidationFailed() throws Exception {
        // 发送不合法的数据测试校验
        String jsonStr = """
                {
                    "username": "",
                    "password": "12",
                    "email": "invalid-email"
                }
                """;

        mockMvc.perform(post("/user")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(jsonStr))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(400))
                .andDo(print());
    }
}
```

## 4.2 @WebMvcTest 切片测试

只加载Web层相关的Bean，不加载Service和Repository，测试更快更轻量：

```java
@WebMvcTest(UserController.class)
class UserControllerSliceTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean   // 模拟并注入一个假的Bean
    private UserService userService;

    @Test
    void testGetById() throws Exception {
        User user = new User();
        user.setId(1L);
        user.setUsername("admin");

        // 设置模拟行为
        when(userService.getById(1L)).thenReturn(user);

        mockMvc.perform(get("/user/{id}", 1L))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.username").value("admin"));

        // 验证userService.getById(1L)被调用过
        verify(userService, times(1)).getById(1L);
    }
}
```

## 4.3 @DataJpaTest 切片测试

只加载JPA相关的Bean，使用内嵌数据库（默认H2）：

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    void testFindByUsername() {
        User user = new User();
        user.setUsername("admin");
        entityManager.persist(user);
        entityManager.flush();

        Optional<User> found = userRepository.findByUsername("admin");
        assertTrue(found.isPresent());
        assertEquals("admin", found.get().getUsername());
    }
}
```

## 4.4 @MockBean 和 @SpyBean

| 注解 | 说明 |
|------|------|
| @MockBean | 创建一个Mock对象替换Spring容器中的Bean，默认不执行真实方法 |
| @SpyBean | 创建一个Spy对象包裹Spring容器中的Bean，默认执行真实方法，可以部分Mock |

```java
@SpringBootTest
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @MockBean
    private SmsService smsService;    // 不想真发短信，用Mock

    @SpyBean
    private InventoryService inventoryService; // 想用真实逻辑但部分Mock

    @Test
    void testCreateOrder() {
        // Mock外部SMS服务
        doNothing().when(smsService).send(anyString());

        // 部分Mock库存服务（扣减库存执行真实逻辑，但监控告警不执行）
        doNothing().when(inventoryService).alertLowStock(anyString());

        orderService.create(new Order());
        verify(smsService, times(1)).send(anyString());
    }
}
```

## 4.5 JsonPath 常用表达式

| 表达式 | 说明 |
|--------|------|
| `$.name` | 根路径下name属性 |
| `$.data.id` | 嵌套属性 |
| `$.items[0]` | 数组第一个元素 |
| `$.items[*].name` | 数组中所有name属性 |
| `$.items.length()` | 数组长度 |
| `$..name` | 递归查找所有name属性 |

```java
// 验证数组长度
.andExpect(jsonPath("$.data.length()").value(3))
// 验证嵌套属性
.andExpect(jsonPath("$.data.items[0].name").value("商品A"))
```

## 4.6 测试事务回滚

Spring Boot测试中，默认会对测试方法进行事务回滚，避免测试数据污染数据库：

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    // 默认回滚：测试数据不会写入数据库
    @Test
    void testAdd() { /* ... */ }

    // 不回滚：需要保留测试数据时
    @Test
    @Rollback(false)
    @Commit
    void testAddAndKeep() { /* ... */ }
}
```
