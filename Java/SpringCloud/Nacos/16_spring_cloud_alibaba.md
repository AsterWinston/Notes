
# 1. 全家桶全景

Spring Cloud Alibaba 提供了一套完整的微服务解决方案，Nacos 是其中的核心基础设施。

```
        Spring Cloud Alibaba 微服务体系

┌─────────────────────────────────────────────────────────┐
│                    接入层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Gateway  │  │  OpenAPI │  │  Mobile  │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       └──────────────┼─────────────┘                     │
│                      │                                    │
├──────────────────────┼────────────────────────────────────┤
│                      ▼                治理层              │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                    Nacos                             │ │
│  │  ┌────────────────┐  ┌──────────────────┐          │ │
│  │  │  服务注册发现   │  │  配置中心         │          │ │
│  │  └────────────────┘  └──────────────────┘          │ │
│  └─────────────────────────────────────────────────────┘ │
│                      │                                    │
├──────────────────────┼────────────────────────────────────┤
│                      ▼                中间件层            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Sentinel │  │  Seata   │  │ RocketMQ │              │
│  │ (流量防卫)│  │(分布式事务)│  │ (消息队列) │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                    业务服务层                              │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────────┐   │
│  │Order   │ │User    │ │Payment │ │Inventory       │   │
│  │Service │ │Service │ │Service │ │Service         │   │
│  │(Dubbo) │ │(Feign) │ │(Dubbo) │ │(Feign/RestTpl) │   │
│  └────────┘ └────────┘ └────────┘ └────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

# 2. 组件版本对应关系

| Spring Cloud Alibaba | Spring Cloud | Spring Boot | Nacos | Sentinel | Seata |
|---------------------|-------------|-------------|-------|----------|-------|
| 2021.0.5.0 | 2021.0.5 | 2.6.13 | 2.2.0 | 1.8.6 | 1.6.1 |
| 2022.0.0.0 | 2022.0.0 | 3.0.2 | 2.2.1 | 1.8.6 | 1.7.0 |
| 2023.0.1.0 | 2023.0.1 | 3.2.2 | 2.3.0 | 1.8.6 | 2.0.0 |

# 3. Maven 依赖管理

```xml
<properties>
    <spring-cloud.version>2021.0.5</spring-cloud.version>
    <spring-cloud-alibaba.version>2021.0.5.0</spring-cloud-alibaba.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

# 4. Nacos + Dubbo 集成

## 4.1 服务提供者

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-dubbo</artifactId>
</dependency>
```

```yaml
# application.yml
dubbo:
  protocol:
    name: dubbo
    port: 20880
  registry:
    address: nacos://127.0.0.1:8848
  scan:
    base-packages: com.example.service
```

```java
// 服务接口
public interface OrderService {
    String createOrder(OrderDTO order);
}

// 服务实现
@DubboService(version = "1.0.0", group = "order-group")
public class OrderServiceImpl implements OrderService {
    @Override
    public String createOrder(OrderDTO order) {
        return "Order created: " + order.getOrderId();
    }
}
```

## 4.2 服务消费者

```java
// 方式一：@DubboReference 注解
@RestController
public class OrderController {

    @DubboReference(version = "1.0.0", group = "order-group", 
                    check = false, timeout = 3000)
    private OrderService orderService;

    @PostMapping("/order")
    public String create(@RequestBody OrderDTO order) {
        return orderService.createOrder(order);
    }
}

// 方式二：Dubbo Generic 泛化调用
@Component
public class GenericConsumer {

    public Object callDubboService() {
        ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
        reference.setInterface("com.example.service.OrderService");
        reference.setVersion("1.0.0");
        reference.setGeneric(true); // 泛化调用

        GenericService genericService = reference.get();
        return genericService.$invoke("createOrder", 
            new String[]{"com.example.dto.OrderDTO"}, 
            new Object[]{buildOrderDTO()});
    }
}
```

## 4.3 Dubbo + Nacos 高级配置

```yaml
dubbo:
  application:
    name: order-service
    qos-enable: true
    qos-port: 22222
  registry:
    address: nacos://127.0.0.1:8848
    parameters:
      namespace: dev
      group: DUBBO_GROUP
  protocol:
    name: dubbo
    port: 20880
    threads: 200
  consumer:
    check: false
    timeout: 5000
    retries: 2
    loadbalance: leastactive    # 负载均衡策略
    cluster: failover           # 集群容错策略
  provider:
    timeout: 3000
    retries: 1
    loadbalance: random
    weight: 100
```

# 5. Nacos + Feign 集成

## 5.1 基础配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

## 5.2 服务调用示例

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

@FeignClient(name = "user-service", path = "/user")
public interface UserFeignClient {

    @GetMapping("/{userId}")
    Result<UserDTO> getUser(@PathVariable("userId") Long userId);

    @PostMapping("/batch")
    Result<List<UserDTO>> getUsers(@RequestBody List<Long> userIds);
}

@RestController
public class OrderController {

    @Autowired
    private UserFeignClient userFeignClient;

    @GetMapping("/order/{orderId}/user")
    public Result<UserDTO> getOrderUser(@PathVariable Long orderId) {
        Long userId = getUserIdFromOrder(orderId);
        return userFeignClient.getUser(userId);
    }
}
```

## 5.3 Feign 高级配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:      # 针对特定服务的配置
            connect-timeout: 3000
            read-timeout: 5000
            logger-level: HEADERS
            request-interceptors:
              - com.example.interceptor.AuthRequestInterceptor
          default:           # 全局默认配置
            connect-timeout: 2000
            read-timeout: 3000
      compression:
        request:
          enabled: true
          mime-types: application/json
          min-request-size: 2048
        response:
          enabled: true
```

# 6. Nacos + Sentinel 集成

## 6.1 流量控制

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080
        port: 8719
      datasource:
        flow:
          nacos:
            server-addr: 127.0.0.1:8848
            data-id: ${spring.application.name}-flow-rules.json
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow
        degrade:
          nacos:
            server-addr: 127.0.0.1:8848
            data-id: ${spring.application.name}-degrade-rules.json
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: degrade
```

## 6.2 Sentinel 规则（存储在 Nacos）

```json
// order-service-flow-rules.json（流控规则）
[
  {
    "resource": "createOrder",
    "grade": 1,
    "count": 100,
    "strategy": 0,
    "controlBehavior": 0,
    "limitApp": "default"
  },
  {
    "resource": "getOrder",
    "grade": 0,
    "count": 50,
    "strategy": 0,
    "controlBehavior": 1,
    "warmUpPeriodSec": 10,
    "maxQueueingTimeMs": 500
  }
]

// order-service-degrade-rules.json（降级规则）
[
  {
    "resource": "createOrder",
    "grade": 0,
    "count": 50,
    "timeWindow": 10,
    "minRequestAmount": 5,
    "statIntervalMs": 1000,
    "slowRatioThreshold": 0.5
  }
]
```

## 6.3 使用 Sentinel 注解

```java
@Service
public class OrderService {

    @SentinelResource(
        value = "createOrder",
        blockHandler = "createOrderBlockHandler",
        fallback = "createOrderFallback"
    )
    public OrderDTO createOrder(OrderRequest request) {
        // 业务逻辑
        return new OrderDTO();
    }

    // 限流/降级时执行
    public OrderDTO createOrderBlockHandler(OrderRequest request, BlockException e) {
        return OrderDTO.fallback("Service is busy, please try again later");
    }

    // 业务异常时执行
    public OrderDTO createOrderFallback(OrderRequest request, Throwable e) {
        return OrderDTO.fallback("Service temporarily unavailable");
    }
}
```

# 7. Nacos + Seata 集成

## 7.1 分布式事务配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: default_tx_group
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP
```

## 7.2 分布式事务使用

```java
@Service
public class CreateOrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private InventoryFeignClient inventoryClient;

    @Autowired
    private AccountFeignClient accountClient;

    @GlobalTransactional(name = "create-order", timeoutMills = 300000)
    public void createOrder(OrderRequest request) {
        // 1. 创建订单（本地事务）
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setAmount(request.getAmount());
        orderMapper.insert(order);

        // 2. 扣减库存（远程调用）
        inventoryClient.deduct(request.getProductId(), request.getQuantity());

        // 3. 扣减账户余额（远程调用）
        accountClient.debit(request.getUserId(), request.getAmount());

        // 任何一步失败，Seata 自动回滚前面已完成的操作
    }
}
```

# 8. 完整项目结构

```
microservice-parent/
├── pom.xml (父 POM)
├── common/
│   └── common-api/          # 公共 API 定义
├── gateway/
│   └── api-gateway/          # 网关服务
├── services/
│   ├── order-service/        # 订单服务 (Dubbo Provider)
│   ├── user-service/         # 用户服务 (Feign)
│   ├── payment-service/      # 支付服务 (Dubbo + Feign)
│   └── inventory-service/    # 库存服务 (Dubbo)
└── config/
    ├── nacos/                # Nacos 配置导出
    └── sentinel/             # Sentinel 规则导出
```

# 9. Docker Compose 全家桶部署

```yaml
version: '3.8'
services:
  nacos:
    image: nacos/nacos-server:latest
    environment:
      - MODE=standalone
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root
    ports:
      - "8848:8848"
      - "9848:9848"

  sentinel-dashboard:
    image: bladex/sentinel-dashboard:1.8.6
    environment:
      - SERVER_PORT=8080
    ports:
      - "8080:8080"

  seata-server:
    image: seataio/seata-server:1.6.1
    environment:
      - SEATA_PORT=8091
      - STORE_MODE=db
    ports:
      - "8091:8091"

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"
    volumes:
      - ./sql:/docker-entrypoint-initdb.d
```

# 10. 生产建议

1. **分层隔离**：不同 Namespace 隔离不同环境，不同 Group 组织不同维度的配置
2. **统一日志**：所有服务接入 ELK/SkyWalking 链路追踪
3. **统一监控**：所有服务接入 Prometheus + Grafana
4. **灰度发布**：利用 Nacos 元数据 + Sentinel 流控实现平滑灰度
5. **配置治理**：敏感配置加密，配置变更审批
6. **版本锁定**：严格锁定 Spring Cloud Alibaba 版本与组件版本对应关系
