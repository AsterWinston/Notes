
# 1. 概述

Seata 通过 `spring-cloud-starter-alibaba-seata` 与 Spring Cloud Alibaba 生态深度集成，主要能力包括：

- **自动配置**：自动代理数据源、注册 TM/RM
- **Nacos 集成**：服务发现 + 配置中心
- **Feign 集成**：自动在 Feign 请求头中传播 xid
- **RestTemplate 集成**：通过 Interceptor 传播 xid

# 2. 项目结构

```
order-service/
├── pom.xml
├── src/main/resources/
│   └── application.yml
└── src/main/java/com/example/
    ├── OrderApplication.java
    ├── controller/OrderController.java
    ├── service/OrderService.java
    ├── client/AccountClient.java       (Feign)
    └── mapper/OrderMapper.java

account-service/
├── pom.xml
├── src/main/resources/
│   └── application.yml
└── src/main/java/com/example/
    ├── AccountApplication.java
    ├── controller/AccountController.java
    └── mapper/AccountMapper.java
```

# 3. 完整配置

## 3.1 订单服务（TM）

```yaml
server:
  port: 8081

spring:
  application:
    name: order-service

  datasource:
    url: jdbc:mysql://localhost:3306/order_db?useUnicode=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: seata
      config:
        server-addr: localhost:8848
        namespace: seata

seata:
  tx-service-group: default_tx_group
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: seata
      application: seata-server
      group: SEATA_GROUP
  config:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: seata
      group: SEATA_GROUP
      data-id: seata.properties
  data-source-proxy-mode: AT
```

## 3.2 账户服务（RM）

```yaml
server:
  port: 8082

spring:
  application:
    name: account-service
  # ... 其他配置同上
```

# 4. Feign 集成

## 4.1 xid 自动传播

Seata 内置 `SeataFeignClientAutoConfiguration`，自动为每个 Feign 请求添加 `TX_XID` 请求头：

```
订单服务 (TM)                    账户服务 (RM)
     │                                │
     │ Feign Request                  │
     │ Header: TX_XID = xxx           │
     │ ─────────────────────────→     │
     │                                │ SeataHandlerInterceptor
     │                                │ 提取 TX_XID → RootContext.bind(xid)
     │                                │ 参与全局事务
```

**无需额外配置**，依赖 spring-cloud-starter-alibaba-seata 即自动生效。

## 4.2 Feign Client 定义

```java
@FeignClient(name = "account-service")
public interface AccountClient {

    @PostMapping("/api/account/debit")
    String debit(@RequestParam Long userId,
                 @RequestParam BigDecimal amount);
}
```

## 4.3 RestTemplate 传播

```java
// RestTemplate 也需要添加 Seata 拦截器
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        // Seata 提供 RequestInterceptor 传播 xid
        template.getInterceptors().add(
            (request, body, execution) -> {
                String xid = RootContext.getXID();
                if (xid != null) {
                    request.getHeaders().add(
                        RootContext.KEY_XID, xid);
                }
                return execution.execute(request, body);
            }
        );
        return template;
    }
}
```

# 5. 全局事务示例

```java
@RestController
@RequestMapping("/api/order")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public Result createOrder(@RequestBody OrderRequest req) {
        orderService.createOrder(req);
        return Result.success();
    }
}

@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private AccountClient accountClient;

    @Autowired
    private InventoryClient inventoryClient;

    @GlobalTransactional(
        name = "create-order",
        timeoutMills = 300000
    )
    public void createOrder(OrderRequest req) {
        // 1. 本地：创建订单 (AT)
        Order order = Order.from(req);
        orderMapper.insert(order);

        // 2. 远程：扣减库存 (Feign → AT)
        inventoryClient.deduct(req.getProductId(), req.getQuantity());

        // 3. 远程：扣减余额 (Feign → AT)
        accountClient.debit(req.getUserId(), req.getAmount());
    }
}
```

# 6. 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        // Seata 异常会自动触发回滚
        log.error("Business error, Seata will rollback", e);
        return Result.error(e.getMessage());
    }

    @ExceptionHandler(NotEnoughBalanceException.class)
    public Result handleBalance(NotEnoughBalanceException e) {
        // 业务异常 → Seata 检测到 → TC 驱动全局回滚
        return Result.error("余额不足");
    }
}
```

# 7. Nacos 配置管理

## 7.1 seata.properties（Nacos 中）

```properties
# 事务分组映射
service.vgroupMapping.default_tx_group=default

# 存储模式
store.mode=db
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://localhost:3306/seata?useUnicode=true
store.db.user=root
store.db.password=root

# 服务端属性
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
server.maxCommitRetryTimeout=120000
server.maxRollbackRetryTimeout=120000

# 客户端属性
client.undo.logTable=undo_log
client.undo.onlyCareUpdateColumns=true
```

# 8. 验证流程

```bash
# 1. 启动 TC Server
sh seata-server.sh

# 2. 启动 Account Service (8082)
# 3. 启动 Order Service (8081)

# 4. 正常下单
curl -X POST http://localhost:8081/api/order \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"productId":100,"amount":50}'

# 5. 观察日志
# [SEATA] begin new global transaction: xid=...
# [SEATA] register branch transaction: xid=..., branchId=...
# [SEATA] commit global transaction: xid=...

# 6. 模拟异常（余额不足）
# account-service 抛出异常 → order-service 回滚
# 检查 order_db → 订单回滚
# 检查 account_db → 扣款回滚
```
