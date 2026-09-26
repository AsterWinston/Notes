
# 1. 项目结构最佳实践

## 1.1 分包策略

```
my-project/
├── my-service-api/                 # API 模块（Consumer 和 Provider 共享）
│   ├── src/main/java/
│   │   └── com/example/service/
│   │       ├── api/                # 服务接口
│   │       │   ├── UserService.java
│   │       │   └── OrderService.java
│   │       ├── dto/                # 数据传输对象
│   │       │   ├── UserDTO.java
│   │       │   └── OrderDTO.java
│   │       ├── constants/          # 常量
│   │       │   └── ErrorCode.java
│   │       └── enums/              # 枚举
│   │           └── OrderStatus.java
│   └── pom.xml
│
├── my-service-provider/            # Provider 实现模块
│   ├── src/main/java/
│   │   └── com/example/service/
│   │       ├── provider/
│   │       │   ├── UserServiceImpl.java
│   │       │   └── OrderServiceImpl.java
│   │       ├── mapper/             # MyBatis Mapper
│   │       ├── repository/
│   │       └── config/
│   └── pom.xml
│
├── my-service-consumer/            # Consumer 模块
│   ├── src/main/java/
│   │   └── com/example/web/
│   │       ├── controller/
│   │       ├── config/
│   │       └── ConsumerApplication.java
│   └── pom.xml
│
└── pom.xml                         # 父 POM
```

## 1.2 依赖管理

```xml
<!-- 父 POM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.example</groupId>
            <artifactId>my-service-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- 统一管理第三方依赖版本 -->
    </dependencies>
</dependencyManagement>

<!-- API 模块：尽量少依赖 -->
<dependencies>
    <!-- 仅依赖必要的序列化和注解 -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo</artifactId>
        <scope>provided</scope>  <!-- 运行期由 Provider/Consumer 提供 -->
    </dependency>
</dependencies>
```

# 2. 接口设计最佳实践

## 2.1 接口粒度

```java
// 不推荐：过粗粒度
public interface DataService {
    Map<String, Object> process(String operation, Map<String, Object> params);
}

// 推荐：合适的粒度（单一职责）
public interface UserService {
    UserDTO getUser(Long userId);
    List<UserDTO> listUsers(UserQuery query);
    Long createUser(CreateUserRequest request);
    void updateUser(Long userId, UpdateUserRequest request);
    void deleteUser(Long userId);
}

public interface OrderService {
    OrderDTO createOrder(CreateOrderRequest request);
    OrderDTO getOrder(Long orderId);
    List<OrderDTO> listUserOrders(Long userId);
}
```

## 2.2 请求/响应对象

```java
// 明确的请求对象
public class CreateUserRequest implements Serializable {
    private static final long serialVersionUID = 1L;
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 50)
    private String name;
    
    @Email
    private String email;
    
    @Min(0)
    @Max(150)
    private Integer age;
    
    // getters/setters
}

// 明确的响应对象
public class UserDTO implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private Long id;
    private String name;
    private String email;
    private Integer age;
    private Date createTime;
    
    // getters/setters
}
```

## 2.3 返回值规范

```java
// 推荐：明确返回类型 + 异常表示错误
public interface UserService {
    // 成功返回 UserDTO，失败抛异常
    UserDTO getUser(Long userId) throws UserNotFoundException;
    
    // 批量操作返回明确的结果对象
    BatchResult createUsers(List<CreateUserRequest> requests);
}

// 不推荐：用 Map/JSON 表示结果
public interface UserService {
    Map<String, Object> getUser(Long id);  // 类型不明确
}
```

## 2.4 异常设计

```java
// 自定义业务异常
public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(Long userId) {
        super(ErrorCode.USER_NOT_FOUND, "User not found: " + userId);
    }
}

public class BusinessException extends RuntimeException {
    private final String errorCode;
    private final String errorMessage;
    
    // ...
}

// Provider 端抛出自定义异常
@DubboService
public class UserServiceImpl implements UserService {
    
    @Override
    public UserDTO getUser(Long userId) {
        User user = userMapper.selectById(userId);
        if (user == null) {
            throw new UserNotFoundException(userId);
        }
        return convertToDTO(user);
    }
}

// Consumer 端捕获
try {
    UserDTO user = userService.getUser(1L);
} catch (UserNotFoundException e) {
    log.warn("User not found: {}", e.getErrorCode());
}
```

# 3. 版本管理最佳实践

## 3.1 接口版本化

```java
// 多版本共存
@DubboService(version = "1.0.0")
public class UserServiceImplV1 implements UserService { }

@DubboService(version = "2.0.0")
public class UserServiceImplV2 implements UserService { }

// Consumer 指定版本
@DubboReference(version = "1.0.0")
private UserService userServiceV1;

@DubboReference(version = "2.0.0")
private UserService userServiceV2;
```

## 3.2 接口兼容性

```
兼容升级（可安全升级）:
  ✅ 新增方法
  ✅ 新增字段（请求/响应对象）
  ✅ 字段新增默认值

不兼容升级（需重建 Consumer）:
  ❌ 修改方法签名（参数类型、数量、返回值）
  ❌ 删除方法
  ❌ 修改 serialVersionUID
  ❌ 修改字段名
  ❌ 修改集合/泛型类型
```

## 3.3 灰度发布流程

```
1. Provider v2 以低权重上线 (weight=10)
2. 观察监控（错误率、RT）
3. 逐步提升 v2 权重，降低 v1 权重
4. 全部切到 v2
5. 观察稳定后，下线 v1
```

# 4. 超时与重试最佳实践

## 4.1 超时设置

```java
// 按接口特点分级设置超时

// 快速查询接口：短超时
@DubboReference(timeout = 500)
private UserService userService;

// 普通业务接口：中等超时
@DubboReference(timeout = 3000)
private OrderService orderService;

// 批量处理接口：长超时
@DubboReference(timeout = 30000)
private BatchService batchService;
```

## 4.2 重试策略

```java
// 幂等读操作：允许重试
@DubboReference(retries = 2)
private UserService userService;

// 非幂等写操作：禁止重试
@DubboReference(retries = 0, cluster = "failfast")
private OrderService orderService;

// 最终一致性：缓慢重试
@DubboReference(cluster = "failback", retries = 5)
private NotificationService notificationService;
```

## 4.3 超时与重试的关系

```
Consumer timeout = 3000ms
Provider timeout = 2000ms
Consumer retries = 2

最长等待时间 = timeout × (retries + 1) = 3000 × 3 = 9000ms

建议：
- Provider timeout < Consumer timeout
- 总等待时间 < 上游 Nginx/Gateway 的超时
```

# 5. 性能最佳实践

```yaml
# 高性能配置模板
dubbo:
  protocol:
    name: dubbo
    serialization: kryo          # Kryo 性能最高
    threadpool: fixed
    threads: 200
    iothreads: 8
    dispatcher: message
    queues: 100

  provider:
    timeout: 3000
    retries: 0                   # Provider 不重试（Consumer 控制）
    delay: -1                    # 延迟到 Spring 初始化完成后导出
    weight: 100

  consumer:
    check: false                 # 不强制启动检查
    timeout: 3000
    retries: 2
    connections: 1               # 默认单连接足够

  application:
    register-mode: instance      # 应用级发现
```

# 6. 可观测性最佳实践

```yaml
# 1. 访问日志
dubbo:
  provider:
    accesslog: /var/log/dubbo/access.log

# 2. Metrics
dubbo:
  metrics:
    protocol: prometheus
    port: 9090

# 3. 健康检查
dubbo:
  application:
    qos-enable: true
    qos-port: 22222

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

# 7. 异常处理最佳实践

```java
// 区分业务异常和系统异常
@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public UserDTO getUser(Long userId) {
        try {
            User user = userMapper.selectById(userId);
            if (user == null) {
                // 业务异常：明确、可处理
                throw new UserNotFoundException(userId);
            }
            return convertToDTO(user);
        } catch (BusinessException e) {
            throw e; // 业务异常直接抛出（不被 Dubbo 包装）
        } catch (Exception e) {
            log.error("Unexpected error in getUser", e);
            throw new ServiceException("Service internal error", e);
        }
    }
}

// Consumer 端防御性编程
try {
    UserDTO user = userService.getUser(1L);
} catch (UserNotFoundException e) {
    // 可预期的业务异常：正常处理
    return ResponseEntity.notFound().build();
} catch (RpcException e) {
    // Dubbo RPC 异常：超时、网络等问题
    if (e.isTimeout()) {
        // 超时降级
        return fallbackGetUser(1L);
    }
    throw e;
}
```

# 8. 安全最佳实践

```yaml
# 生产环境安全配置
dubbo:
  application:
    qos-accept-foreign-ip: false   # 禁止外部 IP 访问 QOS
  
  protocol:
    serialization-security:
      check-serializable: true
      trusted-classes:
        - com.example.api.*
        - java.util.*
        - java.lang.*

  provider:
    token: "production-secret-token"
```

# 9. 配置管理最佳实践

```yaml
# 生产环境完整配置
spring:
  profiles: prod
  application:
    name: user-service

dubbo:
  application:
    name: ${spring.application.name}
    version: 1.0.0
    qos-enable: true
    qos-port: 22222

  registry:
    address: ${DUBBO_REGISTRY_ADDR:nacos://nacos-cluster:8848}
    timeout: 30000
    parameters:
      namespace: prod

  config-center:
    address: ${DUBBO_CONFIG_CENTER:nacos://nacos-cluster:8848}
    namespace: prod

  protocol:
    name: dubbo
    port: ${DUBBO_PROTOCOL_PORT:20880}
    serialization: kryo

  provider:
    timeout: 3000
    retries: 0
    threads: 200
    delay: -1
    weight: 100
    accesslog: /var/log/dubbo/access.log

  consumer:
    timeout: 3000
    retries: 2
    check: false

# 敏感信息使用环境变量
# ${DUBBO_REGISTRY_ADDR} 等通过 K8s ConfigMap/Secret 注入
```

# 10. 开发流程最佳实践

```
1. 接口先行：
   - 先定义 API 模块中的接口和 DTO
   - Provider 和 Consumer 并行开发

2. 本地开发：
   - Consumer 直连 Provider (dubbo://localhost:20880)
   - 使用 @DubboReference(url="...") 绕过注册中心

3. 联调测试：
   - 使用测试注册中心 (dev 环境 Nacos/ZK)
   - 统一 version/group 隔离

4. 集成测试：
   - 自动化测试验证接口契约
   - 使用 Spring Boot Test + Dubbo 测试框架

5. 上线发布：
   - 灰度发布（低权重 → 逐步提升）
   - 观察监控（错误率、RT、线程池）
   - 做好回滚准备
```

# 11. 测试最佳实践

```java
// 1. Provider 单元测试
@SpringBootTest
@EnableDubbo
class UserServiceImplTest {

    @Autowired
    private UserService userService;

    @Test
    void testGetUser() {
        UserDTO user = userService.getUser(1L);
        assertNotNull(user);
    }
}

// 2. Consumer 集成测试（直连模式）
@SpringBootTest
class UserServiceConsumerTest {

    // 直连测试 Provider，不依赖注册中心
    @DubboReference(url = "dubbo://localhost:20880")
    private UserService userService;

    @Test
    void testGetUser() {
        UserDTO user = userService.getUser(1L);
        assertEquals("user-1", user.getName());
    }
}

// 3. 泛化调用测试（无 API jar）
@Test
void testGenericInvoke() {
    ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
    reference.setInterface("org.example.api.UserService");
    reference.setUrl("dubbo://localhost:20880");
    reference.setGeneric("true");

    GenericService service = reference.get();
    Object result = service.$invoke("getUser",
        new String[]{"java.lang.Long"}, new Object[]{1L});

    assertNotNull(result);
}
```

# 12. 不要做的事

| 事项 | 原因 |
|------|------|
| 不要用 `Map<String, Object>` 作为接口参数/返回值 | 类型不安全，维护困难 |
| 不要在 Dubbo 接口中声明受检异常（Checked Exception） | 不符合 RPC 语义，会被 ExceptionFilter 包装 |
| 不要把 Service 接口和实现混合在同一个模块 | API 变更影响 Consumer 编译 |
| 不要在一个接口中定义过多方法 | 影响服务拆分和维护 |
| 不要在 Dubbo 线程池中执行长阻塞操作 | 线程池耗尽 |
| 不要使用 JDK 自带序列化 | 性能和安全性问题 |
| 不要忽略 `serialVersionUID` | 不同编译版本可能导致不兼容 |
| 不要在 Consumer 启动时依赖 Provider（check=true） | 启动顺序依赖 |
| 不要在 `@DubboService` 上同时加 `@Transactional` | RPC 和本地事务混合，不利于分布式事务 |
