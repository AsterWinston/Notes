# 1. 连接池概述

数据库连接池（Connection Pool）是预先创建一定数量的数据库连接并放入"池"中管理，当应用需要访问数据库时，从池中获取已创建好的连接；使用完毕后，将连接**归还**池中而非销毁。

## 1.1 为什么需要连接池

| 无连接池 | 有连接池 |
|---------|---------|
| 每次请求新建连接（TCP 三次握手 + MySQL 认证 + 线程创建） | 复用已有连接，无需重复创建 |
| 高并发时连接数爆炸（可能打满 MySQL max_connections） | 受限连接数，保护数据库 |
| 频繁创建/销毁连接消耗大量 CPU + 内存 | 连接复用，资源消耗恒定 |
| 连接无管理（可能泄露、僵死） | 池化管理：空闲回收、心跳保活、连接校验 |

**典型耗时对比**：
- 创建一次 TCP + MySQL 连接：约 20~50ms
- 从连接池获取连接：约 1~5μs（微秒级）

# 2. 常见连接池实现

| 连接池 | 特点 | 性能 |
|--------|------|------|
| **HikariCP** | Spring Boot 2.x 默认；极快、轻量、可靠性高 | **最快（推荐）** |
| **Druid** | 阿里开源；功能丰富（监控、SQL 防火墙、可视化控制台） | 略低于 HikariCP |
| **C3P0** | 老牌连接池；性能一般、配置复杂 | 较慢（已淘汰） |
| **Tomcat JDBC** | Tomcat 内置；性能尚可 | 中等 |
| **DBCP2** | Apache 项目；Spring Boot 1.x 默认 | 中等（已被 HikariCP 取代） |

> **推荐**：Java 项目首选 **HikariCP**（性能）或 **Druid**（监控）。

# 3. HikariCP（Spring Boot 默认）

## 3.1 Spring Boot配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf-8
    username: root
    password: root123
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      # 连接池核心参数
      minimum-idle: 10              # 最小空闲连接数（默认 = maximum-pool-size）
      maximum-pool-size: 20         # 最大连接数（默认 10）
      idle-timeout: 600000          # 空闲连接最大存活时间（ms，默认 10 分钟）
      max-lifetime: 1800000         # 连接最大存活时间（ms，默认 30 分钟，应比 MySQL wait_timeout 短）
      connection-timeout: 30000     # 获取连接超时（ms，默认 30 秒）
      connection-test-query: SELECT 1  # 连接校验 SQL（MySQL 8.0 不需要，驱动自带心跳）
      pool-name: HikariPool-MyDB    # 连接池名称
```

## 3.2 连接池大小公式

连接池大小没有万能公式，与以下因素相关：

```
连接数 = ((核心数 * 2) + 有效磁盘数)
```

| 因素 | 影响 |
|------|------|
| CPU 核心数 | 每个连接需要线程处理，但 MySQL 服务器处理能力有限 |
| 磁盘 IO | 数据库瓶颈通常在磁盘 IO，过多连接徒增上下文切换 |
| 业务特性 | IO 密集型 > CPU 密集型所需的连接数 |

> **经验值**：中小项目 10~20 连接，大型服务 20~50。**不是连接数越多越好**，过多连接导致 MySQL 上下文切换开销超过收益。

## 3.3 连接池调优要点

```yaml
hikari:
  # 1. 如果响应时间不稳定，考虑缩小连接池（保持 CPU 核心数 × 2 左右）
  maximum-pool-size: 10

  # 2. 连接超时不能太短（极端情况需要排队等待连接）
  connection-timeout: 30000

  # 3. max-lifetime 必须比 MySQL wait_timeout 短 2~3 分钟
  #    MySQL wait_timeout = 28800s (8h)
  #    HikariCP max-lifetime = 1800000ms (30min)  ✓
  max-lifetime: 1800000

  # 4. 开启 JMX 监控
  register-mbeans: true
```

## 3.4 连接泄露检测

```yaml
hikari:
  leak-detection-threshold: 10000  # 连接超过 10 秒未归还则打印警告日志
```

```log
[HikariPool-1 housekeeper] WARN  HikariPool-1 - Connection leak detection triggered
for conn5 on thread http-nio-8080-exec-3, stack trace follows
java.lang.Exception: Apparent connection leak detected
    at org.example.UserService.getUser(UserService.java:42)
```

> **生产环境务必开启 `leak-detection-threshold`**，及时发现忘关连接的代码 BUG。

# 4. Druid（适合监控场景）

Druid 提供了丰富的监控和管理功能，适合需要深度监控 SQL 执行情况的团队。

## 4.1 Spring Boot配置

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      url: jdbc:mysql://localhost:3306/mydb
      username: root
      password: root123
      initial-size: 5
      min-idle: 10
      max-active: 20
      max-wait: 30000
      test-on-borrow: false
      test-while-idle: true
      validation-query: SELECT 1
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000

      # 监控配置
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        login-username: admin
        login-password: admin123

      # SQL 防火墙
      filters: stat,wall,slf4j

      # 慢 SQL 记录
      filter:
        stat:
          slow-sql-millis: 1000
          log-slow-sql: true
```

访问 `http://localhost:8080/druid/` 查看监控面板（数据源监控、SQL 监控、URI 监控等）。

# 5. MySQL连接相关参数

## 5.1 JDBC URL推荐参数

```
jdbc:mysql://localhost:3306/mydb
  ?useSSL=false                          # 内网关 SSL（外网必须开）
  &serverTimezone=Asia/Shanghai          # 指定时区
  &characterEncoding=utf-8               # 编码
  &useServerPrepStmts=true               # 开启服务端预编译
  &cachePrepStmts=true                   # 缓存预编译语句
  &rewriteBatchedStatements=true         # 批量 INSERT 优化（将多条合并为一条）
  &allowMultiQueries=false               # 禁止多语句（安全）
  &connectTimeout=5000                   # 建立连接超时(ms)
  &socketTimeout=60000                   # 读取超时(ms)
```

## 5.2 MySQL服务端连接参数

```sql
-- 查看关键连接参数
SHOW VARIABLES LIKE 'max_connections';         -- 最大连接数（默认 151）
SHOW VARIABLES LIKE 'wait_timeout';            -- 非交互连接超时（默认 28800s）
SHOW VARIABLES LIKE 'interactive_timeout';     -- 交互连接超时
SHOW STATUS LIKE 'Threads_connected';          -- 当前连接数
SHOW STATUS LIKE 'Max_used_connections';       -- 历史最大连接数
```

**生产环境建议**：

```
[mysqld]
max_connections = 1000          # 根据业务量设置
wait_timeout = 600              # 10 分钟，及时回收空闲连接
interactive_timeout = 600
```

> `wait_timeout` 设置过短可能导致连接池中空闲连接被 MySQL 断开 → 应用获取到死连接 → `CommunicationsException`。连接池的 `max-lifetime` 必须小于 `wait_timeout`。

# 6. 连接池常见问题

## 6.1 连接耗尽

```
java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms.
```

**原因**：
- 连接池大小配置不足
- 连接泄露（获取连接后未关闭）
- 慢 SQL 持有连接时间过长
- 数据库响应慢

**排查**：
```sql
-- MySQL 端查看连接
SHOW PROCESSLIST;  -- 查看所有连接的当前状态
-- 状态为 Sleep 的连接过多 → 连接池闲置连接太多
-- 状态为 Sending data / Creating sort index → 存在慢查询

-- 按状态统计
SELECT state, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY state;
```

## 6.2 连接泄露

```java
// ✗ 连接泄露：没有在 finally 中关闭
public User getUser(Long id) {
    Connection conn = dataSource.getConnection();
    // ... 使用 conn ...
    // 忘记归还！→ 连接泄露
    return user;
}

// ✓ 正确做法：try-with-resources 自动关闭
public User getUser(Long id) throws SQLException {
    try (Connection conn = dataSource.getConnection()) {
        // ... 使用 conn ...
        return user;
    } // 自动关闭
}

// ✓ Spring JdbcTemplate：框架自动管理连接
@Autowired
private JdbcTemplate jdbcTemplate;
// 框架自动从连接池获取和归还，无需手动管理
```

## 6.3 连接池监控指标

| 指标 | 说明 |
|------|------|
| Active Connections | 当前活跃连接数（接近 max = 需扩容） |
| Idle Connections | 空闲连接数 |
| Pending Connections | 等待获取连接的请求数（> 0 = 池太小） |
| Connection Timeout Count | 获取连接超时次数 |
| Connection Creation Rate | 连接创建速率（突增 = 池不够用） |
