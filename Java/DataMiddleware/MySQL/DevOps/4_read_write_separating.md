# 1. 读写分离概述

读写分离是数据库架构中常见的扩展方案：**主库（Master）负责处理写操作（INSERT/UPDATE/DELETE），从库（Slave）负责处理读操作（SELECT）**。

## 1.1 为什么要读写分离

大多数业务的读写比例通常在 **8:2 到 9:1**（读多写少）：

- **分摊负载**：将大量读请求分散到多个从库，减轻主库压力
- **提升并发**：主库专注处理写入，从库提供并行读取能力
- **提高可用性**：主库故障时，从库可接管读流量（甚至提升为新主库）
- **数据安全**：可在从库执行备份、报表查询等重操作，不影响核心业务

## 1.2 实现层面

读写分离可在不同层面实现：

| 层面 | 实现方式 | 优缺点 |
|------|---------|--------|
| **应用层** | 代码中手动路由（多数据源 + AOP） | 灵活可控，但代码侵入性强 |
| **中间件层** | ShardingSphere、Mycat | 透明、集中管理，但增加维护节点 |
| **驱动层** | MySQL Router、JDBC 路由 | 轻量，但功能有限 |

# 2. 一主一从架构

## 2.1 架构拓扑

```
          ┌──────────────┐
          │   应用服务器   │
          └──┬────────┬──┘
             │        │
        Write│        │Read
             │        │
             ▼        ▼
       ┌─────────┐ ┌─────────┐
       │  Master │ │  Slave  │
       │  (写)   │ │  (读)   │
       └────┬────┘ └─────────┘
            │ Binlog
            └──────────────┘ 主从复制
```

## 2.2 代码层实现（Spring + Dynamic DataSource）

### 2.2.1 数据源配置

```yaml
spring:
  datasource:
    master:
      url: jdbc:mysql://192.168.1.100:3306/mydb
      username: root
      password: master123
    slave:
      url: jdbc:mysql://192.168.1.101:3306/mydb
      username: root
      password: slave123
```

### 2.2.2 动态数据源 + AOP

```java
// 数据源上下文持有者
public class DynamicDataSourceHolder {
    private static final ThreadLocal<String> CONTEXT = new ThreadLocal<>();

    public static void setDataSource(String ds) { CONTEXT.set(ds); }
    public static String getDataSource() { return CONTEXT.get(); }
    public static void clear() { CONTEXT.remove(); }
}

// AOP 切面：@ReadOnly 注解 → 从库
@Aspect
@Component
@Order(-1)
public class DataSourceAspect {
    @Around("@annotation(readOnly)")
    public Object around(ProceedingJoinPoint pjp, ReadOnly readOnly) {
        try {
            DynamicDataSourceHolder.setDataSource("slave");
            return pjp.proceed();
        } finally {
            DynamicDataSourceHolder.clear();
        }
    }
}

// Service 层使用
@Service
public class UserService {
    @ReadOnly
    public User getUser(Long id) { ... }    // 读从库

    @Transactional
    public void createUser(User user) { ... } // 写主库
}
```

> **基于 AOP + 注解的读写分离**适用于中小型项目。大型系统建议使用 ShardingSphere 等中间件。

## 2.3 主从延时问题及处理

主从复制存在延迟（通常毫秒到秒级），可能引发"写完立即读"的数据不一致问题：

```java
// 问题：刚写入的数据，从库还没同步完成，立刻读取读不到
userService.createUser(user);        // 写主库
User u = userService.getUser(id);    // 读从库 → 可能为 null！
```

**解决方案**：

| 方案 | 适用场景 |
|------|---------|
| **强制读主库** | 关键操作后立即读（如注册后立即展示个人信息） |
| **延迟双删** | 更新缓存时，等待从库同步后再删缓存 |
| **半同步复制** | 确保至少一个从库收到 binlog 后才返回成功 |
| **缓存补偿** | 写入时同时更新 Redis，读从库 + Redis 兜底 |

```java
// 方案一：关键操作后强制走主库
@Transactional
public void register(User user) {
    userMapper.insert(user);              // 写主库
}

// 紧接着的读操作走主库而非从库
public User getUserAfterRegister(Long id) {
    DynamicDataSourceHolder.setDataSource("master");
    return userMapper.selectById(id);
}
```

> ShardingSphere 提供 `HintManager` 机制强制路由到主库，类似：
> ```java
> HintManager.getInstance().setPrimaryRouteOnly();
> ```

# 3. ShardingSphere读写分离

**Apache ShardingSphere** 是一个分布式数据库生态系统，ShardingSphere-JDBC 可嵌入应用实现读写分离。

## 3.1 ShardingSphere-JDBC配置

```yaml
spring:
  shardingsphere:
    datasource:
      names: master, slave0, slave1
      master:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.1.100:3306/mydb
        username: root
        password: root123
      slave0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.1.101:3306/mydb
        username: root
        password: root123
      slave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.1.102:3306/mydb
        username: root
        password: root123
    rules:
      readwrite-splitting:
        data-sources:
          ms-ds:
            type: Static
            props:
              write-data-source-name: master
              read-data-source-names: slave0, slave1
            load-balancer-name: round_robin
        load-balancers:
          round_robin:
            type: ROUND_ROBIN
            props:
              workId: 1
```

**优点**：
- 代码零侵入：应用无感知，JDBC 层自动路由
- 内置负载均衡：轮询、随机等策略选择从库
- 支持强制主库路由（`HintManager`）

## 3.2 强制走主库

```java
// 场景：注册后立即读取用户信息，必须走主库
@Transactional
public void register(User user) {
    userMapper.insert(user);                     // 自动路由到主库（INSERT）
    HintManager.getInstance().setWriteRouteOnly();  // 强制后续查询走主库
    User inserted = userMapper.selectById(user.getId());  // 走主库
    HintManager.clear();
}
```

# 4. 一主多从架构

```
          ┌──────────────┐
          │   应用服务器   │
          │ (ShardingJdbc)│
          └──┬──┬──┬──┬──┘
        W    │  │  │  │   R（负载均衡）
             ▼  │  │  ▼
       ┌─────────┐ ┌─────────┐ ┌─────────┐
       │  Master │ │ Slave-0 │ │ Slave-1 │
       └─────────┘ └─────────┘ └─────────┘
```

**优势**：
- 更多从库分担读负载
- 单从库故障不影响整体服务（剩余从库接管）
- 可以按业务维度指定不同从库（如报表从库、搜索从库）

# 5. 双主架构

```
          ┌──────────────┐
          │   应用服务器   │
          └──┬──┬──┬──┬──┘
             │  │  │  │
             ▼  │  │  ▼
       ┌─────────┐ ┌─────────┐
       │ Master-A│◄┼►Master-B│
       └────┬────┘ └────┬────┘
            │           │
            ▼           ▼
       ┌─────────┐ ┌─────────┐
       │ Slave-A │ │ Slave-B │
       └─────────┘ └─────────┘
```

双主 + 双从架构提供更高的可用性：

- 正常情况：A 写、B 写，各自挂从库
- 一台主库故障：另一台接管全部写入
- 故障恢复后：数据自动同步回旧主库

**配置要点**：

```sql
-- Master-A
server-id = 1
log-bin = mysql-bin
auto_increment_increment = 2
auto_increment_offset = 1

-- Master-B
server-id = 2
log-bin = mysql-bin
auto_increment_increment = 2
auto_increment_offset = 2
```

# 6. 读写分离的监控

```sql
-- 主库查看从库连接
SHOW REPLICAS;

-- 从库查看复制状态
SHOW REPLICA STATUS\G;

-- 查看复制延迟（秒）
SELECT
  TIME_TO_SEC(TIMEDIFF(NOW(), ts)) AS delay_seconds
FROM (
  SELECT MAX(create_time) AS ts FROM your_table
) t;
```

> **监控指标**：主从延迟（> 1s 告警）、I/O 线程状态、SQL 线程状态、从库连接数。

# 7. 架构演进路径

```
阶段 1：单机 MySQL
         ↓
阶段 2：一主一从（读写分离）
         ↓
阶段 3：一主多从（读负载均衡）
         ↓
阶段 4：分库分表 + 读写分离（水平拆分）
         ↓
阶段 5：多机房部署（异地多活）
```

> **不要过度设计**：一主一从已能满足大多数百万 DAU 以下的应用。读写分离是成本最低、收益最高的第一步。
