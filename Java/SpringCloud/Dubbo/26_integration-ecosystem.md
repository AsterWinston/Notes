
# 1. Spring Cloud Alibaba 体系

```
┌──────────────────────────────────────────────────────┐
│               Spring Cloud Alibaba                     │
├──────────────────────────────────────────────────────┤
│  Nacos (注册/配置)                                     │
│    ├── 服务注册与发现                                   │
│    └── 动态配置管理                                     │
│                                                       │
│  Dubbo (RPC)                                          │
│    ├── 高性能服务间调用                                  │
│    └── 服务治理                                         │
│                                                       │
│  Sentinel (流量防护)                                    │
│    ├── 限流 / 熔断 / 降级                              │
│    └── 系统自适应保护                                    │
│                                                       │
│  Seata (分布式事务)                                     │
│    ├── AT / TCC / Saga / XA 模式                      │
│    └── 事务协调                                         │
│                                                       │
│  RocketMQ (消息)                                       │
│    └── 异步解耦 / 削峰填谷                              │
└──────────────────────────────────────────────────────┘
```

## 1.1 依赖管理

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 1.2 基础依赖

```xml
<!-- Spring Cloud Alibaba Nacos Discovery -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<!-- Spring Cloud Alibaba Nacos Config -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>

<!-- Dubbo Spring Boot Starter -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
</dependency>
```

# 2. Dubbo + Sentinel 集成

Sentinel 为 Dubbo 提供限流、熔断、降级能力。

## 2.1 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-sentinel-spring-cloud-starter</artifactId>
</dependency>
```

## 2.2 配置

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      eager: true
```

## 2.3 限流规则

```java
// 编程式定义限流规则
@PostConstruct
public void initFlowRules() {
    List<FlowRule> rules = new ArrayList<>();
    
    FlowRule rule = new FlowRule();
    rule.setResource("org.example.UserService:getUser");
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    rule.setCount(100); // QPS 100
    rules.add(rule);
    
    FlowRuleManager.loadRules(rules);
}
```

## 2.4 熔断降级

```java
// DegradeRule 熔断规则
DegradeRule rule = new DegradeRule();
rule.setResource("org.example.UserService:getUser");
rule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
rule.setCount(0.5);     // 异常率 > 50%
rule.setMinRequestAmount(10);
rule.setTimeWindow(10);  // 熔断 10s
DegradeRuleManager.loadRules(Collections.singletonList(rule));
```

Dubbo 的 Sentinel Filter：

```java
@Activate(group = {PROVIDER, CONSUMER})
public class SentinelDubboProviderFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        String resourceName = invoker.getInterface().getName() 
            + ":" + invocation.getMethodName();
        
        Entry entry = null;
        try {
            entry = SphU.entry(resourceName);
            return invoker.invoke(invocation);
        } catch (BlockException e) {
            // 限流/熔断
            throw new RpcException("Sentinel blocked: " + resourceName);
        } catch (RpcException e) {
            Tracer.trace(e);
            throw e;
        } finally {
            if (entry != null) {
                entry.exit();
            }
        }
    }
}
```

# 3. Dubbo + Seata 集成

## 3.1 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-seata</artifactId>
</dependency>
```

## 3.2 配置

```yaml
seata:
  enabled: true
  tx-service-group: my_tx_group
  service:
    vgroup-mapping:
      my_tx_group: default
    grouplist:
      default: 127.0.0.1:8091
```

## 3.3 分布式事务示例

```java
@Service
public class OrderService {

    @DubboReference
    private AccountService accountService;

    @DubboReference
    private StorageService storageService;

    @GlobalTransactional  // Seata 全局事务
    public void createOrder(Order order) {
        // 1. 创建订单
        orderMapper.insert(order);
        
        // 2. 扣减库存 (Dubbo RPC)
        storageService.deduct(order.getProductId(), order.getQuantity());
        
        // 3. 扣减账户余额 (Dubbo RPC)
        accountService.debit(order.getUserId(), order.getAmount());
    }
}
```

## 3.4 Seata AT 模式与 Dubbo 集成原理

```
BusinessService (TM)
    │
    ├── OrderService.insert()       → 本地 RM, 写 undo_log
    │
    ├── StorageService.deduct()     → Dubbo RPC → Storage RM, 写 undo_log
    │   (xid 通过 Dubbo attachments 透传)
    │
    └── AccountService.debit()      → Dubbo RPC → Account RM, 写 undo_log
        (xid 通过 Dubbo attachments 透传)

全局提交/回滚由 Seata TC 协调
```

Dubbo 事务上下文透传 Filter：

```java
@Activate(group = {PROVIDER, CONSUMER})
public class TransactionPropagationFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        // Consumer: 将 Seata XID 放入 attachments
        String xid = RootContext.getXID();
        if (xid != null) {
            invocation.setAttachment(RootContext.KEY_XID, xid);
        }
        
        // Provider: 从 attachments 恢复 XID
        String rpcXid = invocation.getAttachment(RootContext.KEY_XID);
        if (rpcXid != null) {
            RootContext.bind(rpcXid);
        }
        
        try {
            return invoker.invoke(invocation);
        } finally {
            if (rpcXid != null) {
                RootContext.unbind();
            }
        }
    }
}
```

# 4. Dubbo + Spring Boot 3

## 4.1 Jakarta 迁移

Dubbo 3.2+ 全面支持 Spring Boot 3 和 Jakarta EE：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>3.3.0</version>
</dependency>
```

## 4.2 自动配置

```java
@SpringBootApplication
@EnableDubbo
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@EnableDubbo` 注解背后：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
}

// EnableDubboConfig → 启动 Dubbo 配置加载
// DubboComponentScan → 扫描 @DubboService 和 @DubboReference
```

# 5. Dubbo + MyBatis

```java
@DubboService
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;  // MyBatis 注入

    @Override
    @Transactional  // Spring 本地事务
    public User createUser(User user) {
        userMapper.insert(user);
        return user;
    }
}
```

**分布式事务注意事项**：
- 同一 Provider 内的多个 DB 操作用 Spring `@Transactional` 管理
- 跨 Provider 的分布式事务用 Seata

# 6. Dubbo + Gateway

网关层接收 HTTP 请求，转发为 Dubbo 调用：

```java
@RestController
@RequestMapping("/api")
public class GatewayController {

    @DubboReference
    private UserService userService;

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

或使用 Dubbo 3 的 Triple 协议直接访问（无需网关）：

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '["1"]' \
  http://localhost:50051/org.example.UserService/getUser
```

# 7. Dubbo + Kubernetes

## 7.1 服务发现

```yaml
# 使用 Kubernetes API 作为注册中心
dubbo:
  registry:
    address: kubernetes://default
  application:
    name: user-service
```

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-registry-kubernetes</artifactId>
</dependency>
```

## 7.2 部署 YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:latest
          ports:
            - containerPort: 20880
            - containerPort: 50051
          env:
            - name: JAVA_OPTS
              value: "-Xms2g -Xmx2g"
          livenessProbe:
            httpGet:
              path: /live
              port: 22222
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8081
            initialDelaySeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - curl -X POST http://localhost:22222/offline && sleep 15
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - name: dubbo
      port: 20880
      targetPort: 20880
```

# 8. Dubbo + SkyWalking（链路追踪）

Apache SkyWalking 是开源的 APM（应用性能管理）系统，通过 Java Agent 无侵入方式集成 Dubbo。

## 8.1 部署方式

```bash
# 下载 SkyWalking Java Agent
wget https://dlcdn.apache.org/skywalking/java-agent/9.1.0/apache-skywalking-java-agent-9.1.0.tgz
tar -xzf apache-skywalking-java-agent-9.1.0.tgz

# JVM 启动参数
java -javaagent:/path/to/skywalking-agent.jar \
     -Dskywalking.agent.service_name=dubbo-provider \
     -Dskywalking.collector.backend_service=127.0.0.1:11800 \
     -jar provider.jar
```

## 8.2 自动追踪的 Dubbo 组件

SkyWalking Agent 自动拦截 Dubbo 的以下组件：

| 组件 | 自动埋点位置 | 生成的 Span |
|------|-------------|------------|
| `DubboInvoker` | Consumer 端发起 RPC 调用 | Client Span |
| `DubboProtocol` | Provider 端接收 RPC 请求 | Server Span |
| `DubboMonitorFilter` | Filter 链中的调用监控 | 内部 Span |

无需任何代码修改或配置，Agent 自动创建调用链。

## 8.3 Dubbo 调用链示例

```
traceId: 1a2b3c4d5e6f

HTTP Request (gateway)
  ├── POST /api/orders                                    [100ms]
  │   └── Dubbo: OrderService.createOrder()               [80ms]
  │       ├── MySQL: INSERT INTO orders                   [10ms]
  │       └── Dubbo: AccountService.debit()               [60ms]
  │           ├── MySQL: UPDATE accounts                  [5ms]
  │           └── Dubbo: InventoryService.deduct()        [45ms]
  │               ├── Redis: DECR stock:1001              [3ms]
  │               └── MySQL: INSERT INTO inventory_log    [5ms]
```

## 8.4 自定义 Trace 参数过滤

在 Dubbo RPC 调用中可能存在敏感参数（手机号、密码等）。SkyWalking 支持参数脱敏：

```properties
# agent/config/agent.config
plugin.dubbo.collect_provider_arguments=true
plugin.dubbo.hide_provider_arguments=true

# 自定义脱敏
# 参数超过此长度自动截断
plugin.dubbo.arguments_length_threshold=256
```

## 8.5 集成 Metrics

SkyWalking 与 Prometheus 联合使用：

```yaml
# 在 Dubbo 端配置 Exposure
dubbo:
  metrics:
    protocol: prometheus
    port: 9090

# SkyWalking OAP 的 Prometheus Fetcher
# 在 SkyWalking backend 配置中添加：
prometheus-fetcher:
  enabled: true
  host: http://localhost:9090
  interval: 30
```
