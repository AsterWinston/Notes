
# 1. 环境准备

| 组件 | 版本要求 |
|------|---------|
| JDK | 17+ |
| Spring Boot | 3.2+ |
| Spring Cloud | 2023.x |
| Spring Cloud Alibaba | 2023.0.1+ |
| Seata | 1.7.0+ |
| Nacos | 2.2+ |

# 2. 部署 TC Server

## 2.1 下载启动

```bash
wget https://github.com/apache/incubator-seata/releases/download/v1.7.0/seata-server-1.7.0.zip
unzip seata-server-1.7.0.zip
cd seata

# 修改配置：conf/application.yml
# 1. 注册中心选 nacos
# 2. 配置中心选 nacos
# 3. 存储模式选 db

# 启动
sh bin/seata-server.sh -p 8091 -h 127.0.0.1
```

## 2.2 application.yml（TC Server）

```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml

seata:
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP
      data-id: seata.properties

  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP

  store:
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
      user: root
      password: root
```

## 2.3 初始化数据库

```sql
-- 创建 seata 数据库
CREATE DATABASE seata;
USE seata;

-- 执行官方 SQL：
-- https://github.com/apache/incubator-seata/blob/v1.7.0/script/server/db/mysql.sql

-- 核心表
-- global_table    - 全局事务表
-- branch_table    - 分支事务表
-- lock_table      - 全局锁表（AT 模式）
-- distributed_lock- 分布式锁（TC 集群）
```

# 3. 业务服务集成

## 3.1 依赖引入

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <!-- Spring Cloud Alibaba Seata -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    </dependency>

    <!-- Nacos 注册/配置 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- 数据库 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.5.5</version>
    </dependency>
</dependencies>
```

## 3.2 application.yml（业务服务）

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

## 3.3 undo_log 表（每个业务库执行）

```sql
CREATE TABLE `undo_log` (
  `id`            BIGINT(20)   NOT NULL AUTO_INCREMENT,
  `branch_id`     BIGINT(20)   NOT NULL COMMENT '分支事务 ID',
  `xid`           VARCHAR(100) NOT NULL COMMENT '全局事务 ID',
  `context`       VARCHAR(128) NOT NULL COMMENT '上下文',
  `rollback_info` LONGBLOB     NOT NULL COMMENT '回滚信息（镜像数据）',
  `log_status`    INT(11)      NOT NULL COMMENT '状态：0-正常 1-已提交',
  `log_created`   DATETIME     NOT NULL COMMENT '创建时间',
  `log_modified`  DATETIME     NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

# 4. 第一个分布式事务

## 4.1 订单服务（TM 发起方）

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private AccountClient accountClient;  // Feign 调用

    @GlobalTransactional(
        name = "create-order",
        timeoutMills = 300000,
        rollbackFor = Exception.class
    )
    public void createOrder(Order order) {
        // 1. 本地操作：创建订单
        orderMapper.insert(order);

        // 2. 远程调用：扣减账户余额
        //    Seata 自动在 Feign 请求中传播 xid
        accountClient.debit(order.getUserId(), order.getAmount());
    }
}
```

## 4.2 账户服务（RM 参与方）

```java
@RestController
@RequestMapping("/api/account")
public class AccountController {

    @Autowired
    private AccountMapper accountMapper;

    @PostMapping("/debit")
    public void debit(@RequestParam Long userId,
                      @RequestParam BigDecimal amount) {
        // Seata 自动代理数据源，生成 undo_log
        accountMapper.debit(userId, amount);

        // 模拟业务异常：余额不足触发全局回滚
        BigDecimal balance = accountMapper.getBalance(userId);
        if (balance.compareTo(BigDecimal.ZERO) < 0) {
            throw new InsufficientBalanceException(userId);
        }
    }
}
```

# 5. 验证

```bash
# 1. 正常调用
curl -X POST http://localhost:8081/api/order \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"productId":100,"amount":50}'
# → 订单创建成功，账户扣减成功

# 2. 异常触发回滚（余额不足）
# → 订单插入被回滚（undo_log 自动补偿）
# → 账户扣减被回滚
# → 两库数据一致
```

# 6. 关键注解

| 注解 | 作用 | 位置 |
|------|------|------|
| `@GlobalTransactional` | 声明全局事务，标记 TM 入口 | 发起方 Service 方法 |
| `@GlobalLock` | 仅加全局锁，不参与全局事务（读操作） | RM 方法 |

# 7. 配置要点

```yaml
seata:
  tx-service-group: default_tx_group      # 事务分组（映射到 TC 集群）
  
  # 事务分组 → TC 集群的映射（Nacos/PaaS 中配置）
  # service.vgroupMapping.default_tx_group=default
  
  data-source-proxy-mode: AT             # 数据源代理模式：AT（自动）/XA
  
  client:
    tm:
      commit-retry-count: 5              # 提交重试次数
      rollback-retry-count: 5            # 回滚重试次数
    rm:
      async-commit-buffer-limit: 10000   # 异步提交缓冲区
      report-retry-count: 5              # 上报重试次数
      lock:
        retry-interval: 10               # 全局锁重试间隔(ms)
        retry-times: 30                  # 全局锁重试次数
```
