
# 1. 测试体系总览

Dubbo 的测试可以分为四个层次：

```
┌─────────────────────────────────────────────┐
│           E2E 测试（全链路验证）                │
│  从 Consumer HTTP 入口 → Dubbo → Provider    │
├─────────────────────────────────────────────┤
│          集成测试（多模块联调）                  │
│  启动真实 Nacos/Provider，验证注册与 RPC       │
├─────────────────────────────────────────────┤
│            Consumer 单测                      │
│  Mock Dubbo 服务，测试 Consumer 业务逻辑       │
├─────────────────────────────────────────────┤
│            Provider 单测                      │
│  直接调用 Service 实现，验证业务逻辑            │
└─────────────────────────────────────────────┘
```

# 2. Provider 单元测试

Provider 端测试核心：**不经过 Dubbo 框架，直接验证业务逻辑**。

```java
// UserServiceImplTest.java
package com.example.provider;

import static org.assertj.core.api.Assertions.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class UserServiceImplTest {

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    void shouldReturnUserWhenIdExists() {
        User mockUser = new User(1L, "test", 25);
        when(userMapper.selectById(1L)).thenReturn(mockUser);

        UserDTO result = userService.getUser(1L);

        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("test");
    }

    @Test
    void shouldThrowExceptionWhenUserNotFound() {
        when(userMapper.selectById(999L)).thenReturn(null);

        assertThatThrownBy(() -> userService.getUser(999L))
            .isInstanceOf(UserNotFoundException.class)
            .hasMessageContaining("999");
    }

    @Test
    void shouldHandleBusinessValidation() {
        User invalidUser = new User(null, "", -1);

        assertThatThrownBy(() -> userService.createUser(invalidUser))
            .isInstanceOf(ValidationException.class);
    }
}
```

**要点**：
- Mock 所有外部依赖（数据库、缓存、消息队列）
- 使用 Arrange-Act-Assert 模式
- 覆盖正常路径、异常路径和边界条件
- 不启动 Spring 容器，纯单元测试

# 3. Consumer 单元测试

Consumer 端测试核心：**Mock 远程 Dubbo 服务，验证 Consumer 自身的业务编排逻辑**。

```java
// UserControllerTest.java — 使用 MockMvc 测试 HTTP 层
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean          // Spring Boot 3.4+ 的 Mock Bean
    @DubboReference
    private UserService userService;  // Mock 掉 Dubbo 远程调用

    @Test
    void shouldReturnUserViaHttp() throws Exception {
        when(userService.getUser(1L))
            .thenReturn(new User(1L, "test-user", 25));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("test-user"));
    }

    @Test
    void shouldHandleDubboTimeout() throws Exception {
        when(userService.getUser(anyLong()))
            .thenThrow(new RpcException(RpcException.TIMEOUT_EXCEPTION, "timeout"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().is5xxServerError())
            .andExpect(jsonPath("$.message").value(containsString("timeout")));
    }
}
```

```java
// ConsumerServiceTest.java — 直接测试 Dubbo 消费者的编排逻辑
@ExtendWith(MockitoExtension.class)
class OrderFacadeTest {

    @Mock
    private UserService userService;      // Mock Dubbo 远程服务

    @Mock
    private InventoryService inventoryService;

    @InjectMocks
    private OrderFacade orderFacade;       // Consumer 端的业务编排类

    @Test
    void shouldCompleteOrderWorkflow() {
        when(userService.getUser(1L))
            .thenReturn(new UserDTO(1L, "test", 30));
        when(inventoryService.checkStock(100L, 5))
            .thenReturn(true);

        OrderResult result = orderFacade.placeOrder(1L, 100L, 5);

        assertThat(result.isSuccess()).isTrue();
        verify(userService).getUser(1L);
        verify(inventoryService).checkStock(100L, 5);
    }
}
```

# 4. 集成测试（Spring + Dubbo 框架）

## 4.1 使用 Testcontainers + Nacos

Testcontainers 可以在测试中启动真实的 Nacos 容器，验证 Dubbo 的完整注册与调用链路。

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// DubboIntegrationTest.java
@SpringBootTest(
    classes = {
        ProviderApplication.class,
        ConsumerApplication.class
    },
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
)
@Testcontainers
class DubboIntegrationTest {

    @Container
    static GenericContainer<?> nacos = new GenericContainer<>(
        DockerImageName.parse("nacos/nacos-server:v2.3.0")
    )
        .withExposedPorts(8848)
        .withEnv("MODE", "standalone");

    @DynamicPropertySource
    static void configureRegistry(DynamicPropertyRegistry registry) {
        String nacosAddr = nacos.getHost() + ":" + nacos.getMappedPort(8848);
        registry.add("dubbo.registry.address",
            () -> "nacos://" + nacosAddr);
        registry.add("spring.cloud.nacos.discovery.server-addr",
            () -> nacosAddr);
    }

    @DubboReference
    private UserService userService;

    @Test
    void shouldInvokeDubboServiceSuccessfully() {
        // 等待服务注册
        await().atMost(30, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                User user = userService.getUser(1L);
                assertThat(user).isNotNull();
                assertThat(user.getId()).isEqualTo(1L);
            });
    }

    @Test
    void shouldHandleServiceNotAvailable() {
        assertThatThrownBy(() -> userService.getUser(-1L))
            .isInstanceOf(RpcException.class);
    }
}
```

## 4.2 使用 Spring Test + 简化注册中心

对于不需要真实 Nacos 的场景，可使用 Dubbo 的内存注册中心或直连模式：

```java
@SpringBootTest
@ActiveProfiles("test")
@TestPropertySource(properties = {
    "dubbo.registry.address=N/A",          // 不使用注册中心
    "dubbo.consumer.url=dubbo://127.0.0.1:20880"  // 直连 Provider
})
class DubboDirectConnectTest {

    @DubboReference
    private UserService userService;

    @Test
    void directConnectInvocation() {
        User user = userService.getUser(1L);
        assertThat(user).isNotNull();
    }
}
```

## 4.3 Provider 端 Dubbo 测试

验证 Provider 的服务导出是否正常，Dubbo 官方提供了 `dubbo-spring-boot-starter-test`：

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter-test</artifactId>
    <version>${dubbo.version}</version>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@EnableDubbo
@TestPropertySource(properties = {
    "dubbo.registry.address=N/A",
    "dubbo.protocol.port=-1"     // 随机端口
})
class ProviderExportTest {

    @Autowired
    private DubboServiceExporter exporter;

    @Test
    void shouldExportServices() {
        List<String> exportedServices = exporter.getExportedServices();
        assertThat(exportedServices)
            .contains("org.example.UserService");
    }
}
```

# 5. Dubbo 专用测试工具

## 5.1 Dubbo QOS — 运行时验证

```bash
# 查看已导出的服务列表
curl http://localhost:22222/ls

# 查看指定服务的详细信息和提供者
curl http://localhost:22222/ls?service=org.example.UserService

# 测试调用（不经过 Consumer）
curl "http://localhost:22222/invoke?service=org.example.UserService&method=getUser&param-types=java.lang.Long&params=[1]"
```

## 5.2 使用 Apidog / Postman 调用 Triple 协议

Triple 协议兼容 HTTP，可直接用 HTTP 工具测试：

```bash
# Triple 协议以 HTTP POST 方式调用
curl -X POST \
  -H "Content-Type: application/json" \
  -d '["1"]' \
  http://localhost:50051/org.example.UserService/getUser

# 带参数名调用
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"id":1}' \
  http://localhost:50051/org.example.UserService/getUser
```

## 5.3 Dubbo Admin — 可视化测试

在 Dubbo Admin 管理后台中：
1. 进入「服务测试」模块
2. 选择目标服务和 Provider
3. 填写方法参数，点击「调用」
4. 查看返回结果和耗时

# 6. Mock 策略对比

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **Mockito Mock** | Provider/Consumer 单测 | 快速、纯内存、无外部依赖 | 不验证 Dubbo 框架行为 |
| **本地直连** | Consumer 集成测试 | 验证 Dubbo 协议调用 | 需启动 Provider |
| **Testcontainers** | 全链路集成测试 | 真实注册中心，最接近生产 | 启动慢，资源消耗大 |
| **Dubbo QOS** | 快速验证、问题排查 | 无需 Consumer，即时调用 | 手动操作，不适合自动化 |
| **内嵌 Dubbo** | 多模块单仓库项目 | 不依赖注册中心 | 只适用于同 JVM 场景 |

# 7. 测试最佳实践

## 7.1 测试分层原则

```
E2E 测试       ← 少量（冒烟测试）
  ▲
集成测试       ← 适量（关键链路验证）
  ▲
单测 (Consumer) ← 大量（业务逻辑覆盖）
  ▲
单测 (Provider) ← 最多（核心业务逻辑）
```

## 7.2 异步调用测试

```java
@Test
@Timeout(5)  // JUnit 5 超时控制
void shouldCompleteAsyncCallWithinTimeout() throws Exception {
    CompletableFuture<User> future = userService.getUserAsync(1L);

    User user = future.get(3, TimeUnit.SECONDS);

    assertThat(user).isNotNull();
}
```

## 7.3 泛化调用测试

```java
@Test
void genericServiceInvocationTest() {
    GenericService genericService = ApplicationModel.defaultModel()
        .getDefaultExtension(GenericService.class);

    Object result = genericService.$invoke(
        "org.example.UserService",
        "getUser",
        new String[]{"java.lang.Long"},
        new Object[]{1L}
    );

    assertThat(result).isNotNull();
}
```

## 7.4 测试覆盖率目标

| 层级 | 覆盖率目标 | 说明 |
|------|-----------|------|
| API 模块（DTO） | 90%+ | 序列化、equals/hashCode、构造器 |
| Provider ServiceImpl | 85%+ | 核心业务逻辑 |
| Consumer Controller | 80%+ | HTTP → Dubbo 转换逻辑 |
| 集成测试 | 关键路径 100% | 核心业务的端到端验证 |
