# 1. 概述与简介

## 1.1 什么是 Druid

Druid（德鲁伊）是阿里巴巴开源的一款数据库连接池组件，它不仅仅是连接池，更是一个集**连接池管理、监控、安全防护**于一体的综合性数据库中间件。Druid 在阿里内部经历了大规模高并发场景的考验，是目前国内 Java 生态中使用最广泛的数据库连接池之一。

## 1.2 Druid 的三大核心能力

```text
Druid = 连接池 + 监控 + 安全
         │        │       │
         │        │       └─ WallFilter: SQL 注入检测
         │        │       └─ ConfigFilter: 密码加密
         │        │
         │        └─ StatFilter: SQL 执行统计
         │        └─ DruidStatViewServlet: Web 监控页面
         │        └─ WebStatFilter: Web 应用监控
         │
         └─ DruidDataSource: 高性能连接池
```

## 1.3 设计理念

Druid 的设计理念与 HikariCP 不同：HikariCP 追求极致的性能（零开销），而 Druid 追求**功能全面**和**可观测性**。Druid 认为在实际生产环境中，可观测性（监控、日志、诊断）带来的价值远超微小的性能差异。

## 1.4 发展历史

- 2011年：阿里巴巴内部开发，解决淘宝、天猫的高并发数据库连接问题
- 2013年：开源到 GitHub，迅速成为国内最流行的连接池
- 持续更新至今：支持 JDBC 4.x、Spring Boot、JPA、MyBatis 等主流框架

# 2. 核心概念与架构

## 2.1 DruidDataSource 架构

```text
Application
      ↓
DruidDataSource（核心类）
      ├── 连接池（DruidConnectionHolder[]）
      ├── 过滤器链（FilterChain）
      │    ├── StatFilter          — 统计过滤器
      │    ├── WallFilter          — SQL 注入防火墙
      │    ├── LogFilter           — 日志过滤器
      │    ├── ConfigFilter        — 配置过滤器（密码解密）
      │    └── 自定义 Filter       — 扩展点
      ├── 监控 JMX MBean
      ├── 连接泄漏检测
      └── 心跳检测线程
```

## 2.2 过滤器链架构

Druid 的过滤器链是其精髓所在，采用职责链模式（Chain of Responsibility）：

```java
// Filter 接口定义
public interface Filter {
    
    // 拦截 Connection 操作
    ConnectionProxy connection_connect(
        FilterChain chain, DruidDataSource dataSource, 
        long maxWaitMillis) throws SQLException;
    
    // 拦截 Statement 创建
    StatementProxy createStatement(
        FilterChain chain, ConnectionProxy connection) throws SQLException;
    
    // 拦截 SQL 执行
    ResultSetProxy executeQuery(
        FilterChain chain, StatementProxy statement, String sql) throws SQLException;
    
    // 约 70+ 个拦截点
}
```

### 2.2.1 过滤器链执行流程

```text
DruidDataSource.getConnection()
      ↓
FilterChainImpl.dataSource_connect()
      ↓
  ┌─────────────────────────────────┐
  │ Filter-1: StatFilter            │ ← SQL 执行统计
  │ Filter-2: WallFilter            │ ← SQL 注入检测
  │ Filter-3: LogFilter             │ ← SQL 日志记录
  │ Filter-4: ConfigFilter          │ ← 密码解密
  │ Filter-5: 自定义 Filter         │ ← 用户扩展
  └─────────────────────────────────┘
      ↓
实际 JDBC 连接
```

## 2.3 DruidConnectionHolder — 连接持有者

每个物理连接被包装为 `DruidConnectionHolder`：

```java
public class DruidConnectionHolder {
    private final DruidDataSource dataSource;    // 所属数据源
    private final Connection conn;                // 物理连接
    private final long connectionId;              // 连接 ID（自增）
    private final long createTimeNano;            // 创建时间（纳秒）
    private final long createStackTrace;          // 创建时的堆栈（用于泄漏检测）
    
    private volatile boolean discard;             // 是否已废弃
    private volatile long lastActiveTimeMillis;   // 最后活跃时间
    private volatile long lastExecTimeMillis;     // 最后执行时间
    private long useCount;                        // 使用次数
    private DruidStatementProxy lastStmt;         // 最后的 Statement
}
```

## 2.4 DruidPooledConnection — 连接代理

```java
public class DruidPooledConnection extends PoolableWrapper implements Connection {
    
    private final DruidConnectionHolder holder;
    private final Connection conn;                    // 被代理连接
    private volatile boolean closed;                  // 是否已关闭（逻辑）
    private volatile boolean disable;                 // 是否禁用
    private volatile long connectTime;                // 连接创建时间
    private volatile long connectionNanoSpan;         // 连接耗时
    
    private final Map<Object, Object> userConnectionInfo;  // 用户信息
    
    @Override
    public void close() throws SQLException {
        if (!closed) {
            closed = true;
            // 归还连接：不是真正关闭，而是回收
            holder.getDataSource().recycle(this);
        }
    }
}
```

# 3. 配置详解

## 3.1 Spring Boot 配置

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
    
    druid:
      # ==== 连接池基础配置 ====
      # 初始化连接数
      initial-size: 5
      # 最小空闲连接数
      min-idle: 5
      # 最大活跃连接数
      max-active: 20
      # 获取连接超时（毫秒）
      max-wait: 60000
      
      # ==== 连接有效性检测 ====
      # 检测 SQL（MySQL 用 SELECT 1，Oracle 用 SELECT 1 FROM DUAL）
      validation-query: SELECT 1
      # 检测超时（秒）
      validation-query-timeout: 3
      # 申请连接时检测（影响性能）
      test-on-borrow: false
      # 归还连接时检测（影响性能）
      test-on-return: false
      # 空闲时检测（推荐，异步线程检测）
      test-while-idle: true
      
      # ==== 空闲连接检测 ====
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      max-evictable-idle-time-millis: 600000
      
      # ==== 连接池统计 ====
      # 通过 connectProperties 属性打开 mergeSql 功能
      connection-properties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=3000
      
      # ==== 过滤器 ====
      # 配置 StatFilter（监控统计）
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
        session-stat-enable: true
        session-stat-max-count: 1000
      
      # 配置 StatViewServlet（监控页面）
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        reset-enable: true
        login-username: admin
        login-password: admin123
        allow: 127.0.0.1        # 允许的 IP
        deny: 10.0.0.1          # 拒绝的 IP
      
      # ==== 防火墙（WallFilter）====
      filter:
        wall:
          enabled: true
          config:
            delete-allow: false            # 禁止 DELETE
            drop-table-allow: false        # 禁止 DROP TABLE
            truncate-allow: false          # 禁止 TRUNCATE
            multi-statement-allow: true    # 允许多语句（批量操作）
            # 其他条件
            condition-and-alway-true-check: false
            selelct-into-outfile-check: true
            select-into-outfile-allow: false
            create-table-select-allow: true
```

## 3.2 核心配置参数详解

| 参数 | 默认值 | 说明 | 建议 |
|------|--------|------|------|
| `initialSize` | 0 | 初始化连接数 | 建议 5-10 |
| `minIdle` | 0 | 最小空闲连接数 | 等于 initialSize |
| `maxActive` | 8 | 最大活跃连接数 | 根据并发和数据库容量调整 |
| `maxWait` | -1（无限等待） | 获取连接超时 | 建议 5000-60000ms |
| `timeBetweenEvictionRunsMillis` | 60000ms | 空闲检测周期 | 保持默认 |
| `minEvictableIdleTimeMillis` | 300000ms（5分钟） | 最小空闲时间 | 保持默认 |
| `maxEvictableIdleTimeMillis` | 600000ms（10分钟） | 最大空闲时间 | 保持默认 |
| `testOnBorrow` | false | 借出时检测 | 保持 false（性能） |
| `testOnReturn` | false | 归还时检测 | 保持 false |
| `testWhileIdle` | true | 空闲时检测 | 保持 true |
| `validationQuery` | — | 检测 SQL | MySQL: SELECT 1 |
| `poolPreparedStatements` | false | PSCache 是否启用 | 推荐 true |
| `maxPoolPreparedStatementPerConnectionSize` | 10 | 每个连接的 PSCache 大小 | 建议 20-50 |

## 3.3 编程式配置

```java
@Configuration
public class DruidConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druidDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        
        // 基础配置
        dataSource.setUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        dataSource.setPassword("secret");
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        
        // 连接池配置
        dataSource.setInitialSize(5);
        dataSource.setMinIdle(5);
        dataSource.setMaxActive(20);
        dataSource.setMaxWait(60000);
        
        // 检测配置
        dataSource.setValidationQuery("SELECT 1");
        dataSource.setTestWhileIdle(true);
        dataSource.setTestOnBorrow(false);
        dataSource.setTestOnReturn(false);
        dataSource.setTimeBetweenEvictionRunsMillis(60000);
        
        // PSCache
        dataSource.setPoolPreparedStatements(true);
        dataSource.setMaxPoolPreparedStatementPerConnectionSize(20);
        
        // 过滤器
        try {
            dataSource.setFilters("stat,wall,log4j2");
        } catch (SQLException e) {
            throw new RuntimeException("Init druid filter error", e);
        }
        
        // 连接属性
        dataSource.setConnectionProperties(
            "druid.stat.mergeSql=true;druid.stat.slowSqlMillis=3000");
        
        return dataSource;
    }
}
```

# 4. 高级特性

## 4.1 监控特性

### 4.1.1 StatFilter — SQL 统计过滤器

`StatFilter` 是 Druid 最核心的监控过滤器，收集所有 SQL 执行数据：

```yaml
spring:
  datasource:
    druid:
      filter:
        stat:
          enabled: true
          log-slow-sql: true           # 打印慢 SQL
          slow-sql-millis: 3000         # 慢 SQL 阈值（3秒）
          merge-sql: true              # 合并相似 SQL（参数化）
```

```java
// StatFilter 采集的核心指标
public class JdbcSqlStat {
    private String sql;                            // SQL 语句
    private long executeCount;                      // 执行次数
    private long executeTimeNano;                   // 总执行时间（纳秒）
    private long executeTimeMax;                    // 最大执行时间
    private long runningCount;                      // 当前正在执行的数
    private long concurrentMax;                     // 最大并发数
    
    private long fetchRowCount;                     // 读取行数
    private long updateCount;                       // 更新行数
    
    private long errorCount;                        // 错误次数
    private List<Throwable> errorExceptions;        // 错误异常列表
    
    private long histogram_0_1;                     // 执行时间分布：0-1ms
    private long histogram_1_10;                    // 1-10ms
    private long histogram_10_100;                  // 10-100ms
    private long histogram_100_1000;                // 100-1000ms
    private long histogram_1000_10000;              // 1-10秒
}
```

### 4.1.2 StatViewServlet — Web 监控页面

Druid 内置了一个 Web 监控页面，通过 `StatViewServlet` 提供：

```yaml
spring:
  datasource:
    druid:
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        reset-enable: true
        login-username: admin
        login-password: admin123
        allow: 127.0.0.1
```

访问 `http://localhost:8080/druid/` 进入监控页面，提供以下功能：

```text
数据源（DataSource）
  └─ 连接池状态（活跃/空闲/等待线程数...）
  
SQL 监控（SQL Monitor）
  ├─ 所有 SQL 执行统计（执行次数、耗时、行数）
  ├─ SQL 执行时间分布（0-1ms, 1-10ms, 10-100ms...）
  └─ 慢 SQL 记录
  
URI 监控（URI Monitor）—— 需要 WebStatFilter
  ├─ 每个 REST API 的 SQL 执行统计
  └─ 每个 API 的请求次数、耗时
  
Session 监控（Session Monitor）
  └─ 活跃 Session 及关联的 SQL 执行
  
Spring 监控（Spring Monitor）
  └─ Spring Bean 方法调用统计（需 spring-druid 集成）
```

### 4.1.3 WebStatFilter — Web 请求监控

`WebStatFilter` 将 Web 请求与 SQL 执行关联起来：

```yaml
spring:
  datasource:
    druid:
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
        session-stat-enable: true
        session-stat-max-count: 1000
        profile-enable: true
```

通过 `profile-enable: true` 可以查看每个请求执行的 SQL 明细：

```text
请求: GET /api/users/1001
  ↓
  SQL: SELECT * FROM users WHERE id = ?  → 耗时 5ms
  SQL: SELECT * FROM orders WHERE user_id = ?  → 耗时 120ms
  ↓
总耗时: 130ms
```

## 4.2 WallFilter — SQL 注入防火墙

`WallFilter` 是 Druid 独有的安全特性，提供 SQL 注入攻击检测与防护：

```yaml
spring:
  datasource:
    druid:
      filter:
        wall:
          enabled: true
          config:
            # 允许/禁止特定操作
            delete-allow: false                     # 禁止 DELETE
            drop-table-allow: false                 # 禁止 DROP TABLE
            truncate-allow: false                   # 禁止 TRUNCATE
            alter-table-allow: true                 # 允许 ALTER TABLE
            
            # SQL 注入防护
            condition-and-alway-true-check: true    # 检测 WHERE 1=1
            condition-double-constant-check: true   # 检测 WHERE a=1 AND b=2
            condition-op-xor-check: true            # 检测 XOR 注入
            
            # 其他安全设置
            select-into-outfile-allow: false        # 禁止 INTO OUTFILE
            multi-statement-allow: false             # 禁止多语句执行
            none-base-statement-allow: false          # 禁止非标准语句
            
            # 函数白名单
            function-check: true
            variant-check: true
            
            # 元数据访问控制
            describe-table-allow: true
            show-table-allow: true
```

### 4.2.1 WallFilter 内部原理

```java
// WallFilter 的请求处理流程（简化）
public class WallFilter extends FilterAdapter {
    
    private WallProvider provider;  // 防火墙规则引擎
    
    @Override
    public StatementProxy createStatement(FilterChain chain, ConnectionProxy conn) {
        // 创建 Statement 时初始化安全检查
        return chain.createStatement(conn);
    }
    
    @Override
    public ResultSetProxy executeQuery(FilterChain chain, StatementProxy statement, 
                                       String sql) throws SQLException {
        // SQL 执行前进行安全检查
        WallCheckResult result = provider.check(sql);
        
        if (result.getViolate()) {
            // 检测到 SQL 注入，抛出异常
            throw new SQLException("SQL injection detected: " + sql);
        }
        
        // 通过安全检查，执行 SQL
        return chain.executeQuery(statement, sql);
    }
}
```

WallProvider 的检查过程：

```text
SQL 语句
   ↓
词法分析（Lexer）—— 解析 SQL 为 Token 序列
   ↓
语法分析（Parser）—— 构建抽象语法树（AST）
   ↓
安全检查（Visitor）—— 遍历 AST，匹配规则
   │
   ├─ 检测危险函数（load_file、into outfile 等）
   ├─ 检测危险操作（DROP、TRUNCATE 等）
   ├─ 检测永真条件（1=1、1=2 等）
   ├─ 检测注入特征（UNION SELECT、OR 1=1 等）
   ├─ 检测注释注入（/**/巧妙绕过）
   └─ 检测超长 SQL（缓冲区溢出攻击）
   ↓
放行 / 阻断
```

### 4.2.2 常见 SQL 注入检测场景

```text
# 被 WallFilter 拦截的典型场景

1. 永真条件注入
   SELECT * FROM users WHERE id = 1 OR 1=1
   → WallFilter 检测到 WHERE 条件中存在永真

2. UNION SELECT 注入
   SELECT * FROM users WHERE id = 1 UNION SELECT * FROM admin
   → WallFilter 检测到 UNION 后的 SELECT 表不匹配

3. 注释绕过
   SELECT * FROM users WHERE username = 'admin'/*' AND password = 'x'
   → WallFilter 检测到注释被恶意使用

4. 多语句注入
   SELECT * FROM users; DROP TABLE orders;
   → WallFilter 检测到多语句执行（若 multi-statement-allow=false）

5. 大量数据导出
   SELECT * INTO OUTFILE '/tmp/backup.txt' FROM users
   → WallFilter 检测到 INTO OUTFILE
```

## 4.3 ConfigFilter — 密码加密

Druid 支持数据库密码加密存储，通过 `ConfigFilter` 实现：

### 4.3.1 生成加密密码

```java
// 使用 Druid 工具类生成加密密码
public class PasswordGenerator {
    public static void main(String[] args) throws Exception {
        String password = "my_database_password";
        
        // 生成公钥和加密密码
        String[] result = ConfigTools.genKeyPair(512);
        String privateKey = result[0];
        String publicKey = result[1];
        String encryptedPassword = ConfigTools.encrypt(privateKey, password);
        
        System.out.println("Private Key: " + privateKey);
        System.out.println("Public Key: " + publicKey);
        System.out.println("Encrypted Password: " + encryptedPassword);
    }
}
```

### 4.3.2 配置加密密码

```yaml
spring:
  datasource:
    druid:
      # 使用加密后的密码
      password: Nc4n6B1q8x2Yz...
      
      # 启用 ConfigFilter
      filter:
        config:
          enabled: true
      
      # 连接属性中配置公钥
      connection-properties: config.decrypt=true;config.decrypt.key=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBA...
```

### 4.3.3 使用 Spring 的自动解密

```java
@Configuration
public class DruidConfig {
    
    @Value("${database.password.encrypted}")
    private String encryptedPassword;
    
    @Value("${database.password.publicKey}")
    private String publicKey;
    
    @Bean
    public DataSource dataSource() throws Exception {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        
        // 解密密码
        String password = ConfigTools.decrypt(publicKey, encryptedPassword);
        dataSource.setPassword(password);
        
        return dataSource;
    }
}
```

## 4.4 LogFilter — 日志过滤器

Druid 提供了多种日志过滤器，用于记录 SQL 执行日志：

```yaml
spring:
  datasource:
    druid:
      filter:
        slf4j:
          enabled: true
          statement-create-after-log-enabled: false
          statement-close-after-log-enabled: false
          result-set-open-after-log-enabled: false
          result-set-close-after-log-enabled: false
          data-source-log-enabled: false
          connection-log-enabled: false
          statement-log-enabled: true
          statement-executable-sql-log-enable: true
```

支持三种日志框架：

| 过滤器 | 说明 |
|--------|------|
| `log4j` | Log4j 1.x 支持 |
| `log4j2` | Log4j 2.x 支持 |
| `slf4j` | SLF4J 门面（推荐） |

## 4.5 连接泄漏检测

Druid 内置了连接泄漏检测机制：

```yaml
spring:
  datasource:
    druid:
      # 连接泄漏检测
      remove-abandoned: true                    # 是否开启
      remove-abandoned-timeout: 1800            # 超时时间（秒，30分钟）
      log-abandoned: true                       # 打印告警日志
```

```text
启用后，Druid 会在以下情况输出告警：
WARN - abandon connection, owner thread: http-nio-8080-exec-3, 
        connected at: 2024-01-15 10:30:00, 
        use stack: com.example.service.UserService.getUser(...)
```

## 4.6 慢 SQL 日志

```yaml
spring:
  datasource:
    druid:
      filter:
        stat:
          enabled: true
          log-slow-sql: true
          slow-sql-millis: 2000      # SQL 执行超过 2 秒即视为慢 SQL
          merge-sql: true            # 合并相似的 SQL（? 参数化）
```

输出示例：

```text
WARN - Slow SQL: 
    SQL: SELECT * FROM orders o 
         JOIN order_items oi ON o.id = oi.order_id 
         WHERE o.user_id = ? 
         AND o.status = ? 
         ORDER BY o.created_at DESC
    ExecuteTime: 3,256ms
    FetchRowCount: 1,024
    StartTime: 2024-01-15 10:30:00
```

# 5. 底层原理

## 5.1 连接池内部实现

### 5.1.1 连接池数据结构

```java
public class DruidDataSource extends DruidAbstractDataSource {
    
    // 连接池数组（核心存储）
    private volatile DruidConnectionHolder[] connections;
    
    // 连接池计数器
    private int poolingCount;        // 池中可用连接数
    private int activeCount;         // 活跃连接数
    private int activePeak;          // 活跃连接峰值
    private int creatingCount;       // 正在创建的连接数
    private int createErrorCount;    // 创建错误次数
    private int closeCount;          // 关闭次数
    
    // 锁和条件
    private final ReentrantLock lock;
    private final Condition notEmpty;     // 连接池非空条件
    private final Condition empty;        // 连接池为空条件
    
    // 等待线程计数器
    private int notEmptyWaitCount;
    private long notEmptyWaitNanos;
    private long notEmptySignalCount;
    
    // 后台线程
    private DestroyConnectionThread destroyConnectionThread;  // 销毁线程
    private CreateConnectionThread createConnectionThread;    // 创建线程
    private LogStatsThread logStatsThread;                    // 统计日志线程
}
```

### 5.1.2 获取连接流程

```java
public DruidPooledConnection getConnection(long maxWaitMillis) throws SQLException {
    // 1. 安全检查连接池状态
    checkState();
    
    // 2. 尝试获取连接（阻塞队列方式）
    DruidConnectionHolder holder = getConnectionInternal(maxWaitMillis);
    
    // 3. 包装为 DruidPooledConnection
    DruidPooledConnection poolConn = new DruidPooledConnection(holder);
    
    // 4. 加入 trace 列表（用于泄漏检测）
    connTrace.put(poolConn, new Exception("Connection trace"));
    
    return poolConn;
}

private DruidConnectionHolder getConnectionInternal(long maxWait) {
    // 加锁
    lock.lock();
    try {
        // 循环等待
        while (poolingCount == 0) {
            // 尝试创建新连接
            if (createScheduled) {
                // 等待条件
                notEmpty.await(maxWait, TimeUnit.MILLISECONDS);
            }
        }
        
        // 从池中取出连接
        DruidConnectionHolder holder = connections[--poolingCount];
        connections[poolingCount] = null;
        
        activeCount++;
        return holder;
        
    } finally {
        lock.unlock();
    }
}
```

### 5.1.3 归还连接流程

```java
public void recycle(DruidPooledConnection pooledConnection) {
    DruidConnectionHolder holder = pooledConnection.getConnectionHolder();
    
    // 1. 还原 autoCommit 设置
    resetAutoCommit(holder);
    
    // 2. 重置只读状态
    resetReadOnly(holder);
    
    // 3. 清空未关闭的 Statement
    clearStatement(holder);
    
    // 4. 从 trace 列表移除
    connTrace.remove(pooledConnection);
    
    // 5. 放回池中
    lock.lock();
    try {
        connections[poolingCount++] = holder;
        activeCount--;
        
        // 通知等待线程
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
}
```

## 5.2 创建线程（CreateConnectionThread）

Druid 使用独立的后台线程创建连接：

```java
public class CreateConnectionThread extends Thread {
    
    public void run() {
        while (true) {
            // 检查是否需要创建连接
            if (poolingCount >= minIdle) {
                // 等待需要
                waitNeeded();
            }
            
            // 物理创建连接
            Connection conn = DriverManager.getConnection(url, props);
            
            // 包装为 DruidConnectionHolder
            DruidConnectionHolder holder = new DruidConnectionHolder(holder);
            
            // 放入连接池
            lock.lock();
            try {
                connections[poolingCount++] = holder;
                notEmpty.signal();  // 通知等待线程
            } finally {
                lock.unlock();
            }
        }
    }
}
```

## 5.3 销毁线程（DestroyConnectionThread）

定期清理超时空闲连接：

```java
public class DestroyConnectionThread extends Thread {
    
    public void run() {
        while (true) {
            try {
                Thread.sleep(timeBetweenEvictionRunsMillis);
            } catch (InterruptedException e) {
                break;
            }
            
            // 1. 检查空闲连接是否超时
            evictIdleConnections();
            
            // 2. 检查物理连接是否有效
            checkValidConnections();
            
            // 3. 输出统计日志
            logStats();
        }
    }
}
```

## 5.4 ExpandPollThread（连接池扩容线程）

当 `maxActive` 被占满而等待线程较多时，Druid 会动态扩容：

```text
触发条件：activeCount >= maxActive 且 有线程等待

扩容策略：
1. 每次扩容：maxActive × 0.5（向上取整）
2. 上限：不超过预设的最大值
3. 收缩：空闲连接超过 minIdle 时逐步回收
```

# 6. 性能优化

## 6.1 PSCache（PreparedStatement Cache）优化

```yaml
spring:
  datasource:
    druid:
      # 启用 PSCache
      pool-prepared-statements: true
      # 每个连接上缓存的 PreparedStatement 数量
      max-pool-prepared-statement-per-connection-size: 20
      # MySQL 驱动也需要配合开启
      connection-properties: 
        cachePrepStmts=true;
        prepStmtCacheSize=250;
        prepStmtCacheSqlLimit=2048;
        useServerPrepStmts=true;
```

```text
PSCache 优化效果：
  开启前：每次 SQL 执行都经历 解析 → 编译 → 优化 → 执行
  开启后：解析/编译结果缓存，SQL 执行仅需绑定参数
  性能提升：复用率越高，效果越明显（通常 30%-60%）
  
注意：
  - 仅对参数化查询有效（PreparedStatement）
  - 内存消耗：每个连接缓存 N 个 PreparedStatement
  - maxPoolPreparedStatementPerConnectionSize 过大可能导致内存溢出
  - MySQL 8.0.20+ 已弃用服务端 Prepared Statement 缓存
```

## 6.2 连接验证策略优化

```yaml
spring:
  datasource:
    druid:
      # 推荐配置（性能最优）
      test-on-borrow: false         # 不验证（避免每次获取连接都验证）
      test-on-return: false         # 不验证
      test-while-idle: true         # 空闲时验证（推荐）
      validation-query: SELECT 1    # 验证 SQL
      validation-query-timeout: 3   # 验证超时
      time-between-eviction-runs-millis: 30000  # 30秒空闲检测间隔
```

三种验证方式的性能开销：

```text
test-on-borrow = true
  └─ 每次 getConnection() 都执行 SELECT 1
  └─ 并发高时，大量额外的数据库查询
  └─ 性能影响：大（频繁验证）

test-while-idle = true（推荐）
  └─ 只在连接空闲时验证
  └─ 固定周期检测（如30秒）
  └─ 性能影响：小（低频率）

test-on-return = true
  └─ 归还连接时验证
  └─ 非高并发场景可用
  └─ 性能影响：中
```

## 6.3 监控对性能的影响

```yaml
spring:
  datasource:
    druid:
      filter:
        stat:
          enabled: true
          # 关闭不必要的数据采集以提升性能
          connection-stack-start-enable: false     # 关闭连接创建堆栈采集
          slow-sql-millis: 5000                    # 提高慢 SQL 阈值
```

```text
监控开销：
  StatFilter 采集 ≈ 每条 SQL 耗时 0.01-0.1ms
  WallFilter 检查 ≈ 每条 SQL 耗时 0.1-1ms（取决于 SQL 复杂度）
  
在极高并发场景（QPS > 10000）：
  - 可关闭 StatFilter 的 SQL 堆栈采集
  - 可关闭 WallFilter（如果已有其他安全网关）
  - 保留基础指标采集（执行次数、耗时）
```

# 7. 与 Spring Boot 集成

## 7.1 引入依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.21</version>
</dependency>

<!-- 或者引入独立的 druid，手动配置 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.21</version>
</dependency>
```

## 7.2 自动配置原理

`druid-spring-boot-starter` 提供了以下自动配置：

```java
@Configuration
@ConditionalOnClass(DruidDataSource.class)
@EnableConfigurationProperties(DruidStatProperties.class)
public class DruidDataSourceAutoConfigure {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid")
    public DruidDataSource dataSource() {
        return new DruidDataSource();
    }
    
    // StatViewServlet 自动注册
    @Bean
    public ServletRegistrationBean<StatViewServlet> statViewServlet(
            DruidStatProperties properties) {
        // ...
    }
    
    // WebStatFilter 自动注册
    @Bean
    public FilterRegistrationBean<WebStatFilter> webStatFilter(
            DruidStatProperties properties) {
        // ...
    }
}
```

## 7.3 多数据源配置

```java
@Configuration
public class MultiDruidConfig {
    
    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary.druid")
    public DataSource primaryDataSource() {
        return DruidDataSourceBuilder.create().build();
    }
    
    @Bean(name = "secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary.druid")
    public DataSource secondaryDataSource() {
        return DruidDataSourceBuilder.create().build();
    }
    
    @Bean(name = "primaryJdbcTemplate")
    public JdbcTemplate primaryJdbcTemplate(
            @Qualifier("primaryDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
    
    @Bean(name = "secondaryJdbcTemplate")
    public JdbcTemplate secondaryJdbcTemplate(
            @Qualifier("secondaryDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

```yaml
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/db1
      username: root
      password: secret
      druid:
        initial-size: 5
        max-active: 20
        stat-view-servlet:
          enabled: true
          url-pattern: /druid/primary/*
    secondary:
      url: jdbc:mysql://localhost:3306/db2
      username: root
      password: secret
      druid:
        initial-size: 3
        max-active: 10
        stat-view-servlet:
          enabled: true
          url-pattern: /druid/secondary/*
```

## 7.4 结合 MyBatis 使用

```yaml
# application.yml 完整配置示例
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf8&useSSL=false
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
    
    druid:
      initial-size: 5
      min-idle: 5
      max-active: 20
      max-wait: 60000
      test-while-idle: true
      validation-query: SELECT 1
      time-between-eviction-runs-millis: 60000
      
      # 监控配置
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        login-username: admin
        login-password: admin123
      
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.css,/druid/*"
      
      filter:
        stat:
          enabled: true
          log-slow-sql: true
          slow-sql-millis: 3000
          merge-sql: true
        wall:
          enabled: true
          config:
            delete-allow: false
            drop-table-allow: false
```

# 8. 最佳实践与常见问题

## 8.1 最佳实践

### 8.1.1 生产环境推荐配置

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      # 连接池基础
      initial-size: 10
      min-idle: 10
      max-active: 30
      max-wait: 10000
      
      # 连接有效性
      validation-query: SELECT 1
      test-while-idle: true
      time-between-eviction-runs-millis: 30000
      min-evictable-idle-time-millis: 300000
      
      # 连接泄漏检测
      remove-abandoned: true
      remove-abandoned-timeout: 1800
      log-abandoned: true
      
      # 监控（开发环境打开，生产按需）
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        login-username: admin
        login-password: ${DRUID_MONITOR_PASSWORD}
        allow: 10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
        reset-enable: false
      
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
      
      filter:
        stat:
          enabled: true
          log-slow-sql: true
          slow-sql-millis: 2000
        wall:
          enabled: true
          config:
            delete-allow: false
            drop-table-allow: false
            truncate-allow: false
            multi-statement-allow: true
```

### 8.1.2 安全配置

1. **密码加密**：使用 ConfigFilter 加密数据库密码
2. **监控页面保护**：设置 `login-username` 和 `login-password`
3. **IP 白名单**：限制监控页面访问 IP
4. **禁用危险操作**：通过 WallFilter 禁用 DELETE 无 WHERE 条件
5. **最小权限原则**：数据库账号只授予应用所需的最小权限

### 8.1.3 监控告警配置

```java
@Component
public class DruidMonitorAlert {
    
    @EventListener
    public void onDruidDataSourceStart(DruidDataSourceStartedEvent event) {
        DruidDataSource dataSource = event.getDataSource();
        
        // 定时检查连接池状态
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        scheduler.scheduleAtFixedRate(() -> {
            int activeCount = dataSource.getActiveCount();
            int maxActive = dataSource.getMaxActive();
            int waitCount = dataSource.getWaitThreadCount();
            
            // 活跃连接数超过 80% 告警
            if (activeCount > maxActive * 0.8) {
                log.warn("Connection pool high usage: {}/{}", activeCount, maxActive);
            }
            
            // 有线程在等待连接
            if (waitCount > 0) {
                log.warn("Threads waiting for connection: {}", waitCount);
            }
            
        }, 0, 30, TimeUnit.SECONDS);
    }
}
```

## 8.2 常见问题

### 8.2.1 连接池初始化失败

```text
异常：com.alibaba.druid.pool.DruidDataSource - 
       {dataSource-1} init error

原因：
1. 数据库连接不上（URL/IP/端口/防火墙错误）
2. 用户名或密码错误
3. 驱动未加载或版本不兼容

解决方案：
1. 检查网络连通性：telnet host port
2. 验证凭据：直接使用 SQL 客户端连接
3. 检查依赖版本兼容性
```

### 8.2.2 连接获取超时

```text
异常：com.alibaba.druid.pool.GetConnectionTimeoutException: 
       wait millis 60000, active 20, maxActive 20

原因：
1. 并发超过 maxActive
2. 慢查询导致连接被长期占用
3. 连接泄漏

排查步骤：
1. 查看 Druid 监控页面的 SQL 监控
2. 查找执行次数多/耗时长的 SQL
3. 查看 activeCount 是否持续在高位
4. 检查 remove-abandoned 日志
```

### 8.2.3 连接被 WallFilter 拦截

```text
异常：sql injection violation, part of the query...

原因：
WallFilter 检测到可疑 SQL，根据配置规则拦截。

解决方案：
1. 确认 SQL 是否确实有安全风险
2. 如为误判，调整 wall 配置：
   - 关闭特定检查规则
   - 添加 SQL 白名单
3. 绝不可为了绕过检测而关闭 WallFilter
```

### 8.2.4 Druid 监控页面无法访问

```text
排查步骤：
1. 检查依赖是否包含 druid-spring-boot-starter
2. 检查配置 stat-view-servlet.enabled: true
3. 确认 url-pattern 是否正确
4. 检查 exclusions 是否排除了 /druid/*
5. 检查是否有 Spring Security 拦截了监控页面
6. 访问 http://localhost:8080/druid/（注意末尾斜杠）
```

### 8.2.5 连接池内存泄漏

```text
场景：持续创建新连接但未正确释放

排查：
1. 在监控页面上观察 activeCount 是否持续增长
2. 开启 remove-abandoned 检测泄漏连接
3. 使用 jstack 查看线程堆栈，定位未关闭连接的位置

根因分析：
- 代码中未使用 try-with-resources
- 异常路径中未关闭连接
- 事务未正确提交/回滚
- 自定义 Connection 包装类未委托 close()
```

## 8.3 Druid vs HikariCP

| 维度 | Druid | HikariCP |
|------|-------|----------|
| 核心定位 | 连接池 + 监控 + 安全 | 高性能连接池 |
| 性能 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 功能丰富度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| SQL 监控 | 内置 StatViewServlet 可视化界面 | 仅 JMX/Micrometer 指标 |
| SQL 注入防护 | WallFilter | 无 |
| 密码加密 | ConfigFilter | 无 |
| 连接泄漏检测 | 内置（remove-abandoned） | 内置（leakDetectionThreshold） |
| 慢 SQL 记录 | 内置（SlowSqlMillis） | 需自行实现 |
| 代码量 | ~10万行 | ~1万行 |
| 配置复杂度 | 复杂 | 简单 |
| Spring Boot 默认 | 否 | 是 |
| 社区（国内） | 非常活跃 | 活跃 |
| 适合场景 | 国内企业级，需要监控和安全 | 极简高性能，国际项目 |

选择建议：
- 追求极致性能、配置简洁、国际化项目 → HikariCP
- 需要全面监控、SQL 防护、国内企业级 → Druid
- 两者都需要的 → 使用 HikariCP + 额外的监控工具（如 Arthas、SkyWalking）

# 9. 面试要点

## 9.1 基础问题

**Q: Druid 相比于普通连接池有哪些独特功能？**

A: Druid 不仅是连接池，还集成了三大核心能力：1）监控（StatFilter + StatViewServlet + WebStatFilter）2）安全（WallFilter SQL 注入检测）3）扩展性（Filter 链架构支持自定义插件）。此外还有密码加密、慢 SQL 记录、连接泄漏检测等功能。

**Q: WallFilter 是如何检测 SQL 注入的？**

A: WallFilter 内部包含一个 SQL 词法分析器和语法分析器。将 SQL 语句解析为抽象语法树（AST），然后遍历 AST 节点匹配安全规则。规则包括：检测永真条件（1=1）、危险函数（load_file）、危险操作（DROP TABLE）、注释注入、UNION SELECT 注入等。

**Q: Druid 的过滤器链架构是怎样的？**

A: 基于职责链（Chain of Responsibility）模式。Filter 接口定义了约 70+ 个拦截点（连接创建、Statement 执行、ResultSet 读取等）。过滤器串联执行，每个 Filter 可以在执行前后添加自定义逻辑。典型的 Filter 链：StatFilter → WallFilter → LogFilter → ConfigFilter → 用户自定义 Filter。

**Q: Druid 如何实现数据库密码加密？**

A: 通过 ConfigFilter 实现。使用 Druid 提供的 ConfigTools 工具类生成 RSA 密钥对，用私钥加密密码，配置文件中存储公钥和加密后的密码。应用启动时 ConfigFilter 使用公钥解密还原真实密码。

## 9.2 进阶问题

**Q: Druid 的连接池缩扩容机制是怎样的？**

A: 创建连接由 `CreateConnectionThread` 后台线程执行，当池中连接数小于 `minIdle` 时自动创建。销毁由 `DestroyConnectionThread` 定期执行，清理超过 `minEvictableIdleTimeMillis` 的空闲连接。当活跃连接达到 `maxActive` 且有线程等待时触发扩容，扩容量 = `maxActive * 0.5`。

**Q: Druid 的监控性能开销有多大？**

A: StatFilter 的每条 SQL 统计基准开销约 0.01-0.1ms。WallFilter 的 SQL 解析和检查开销约 0.1-1ms（取决于 SQL 长度和复杂度）。在 QPS < 5000 的场景下，监控开销可忽略不计。极高并发场景可关闭堆栈采集和部分检查规则。

**Q: Druid 如何实现连接泄漏检测？**

A: Druid 内部维护一个 `connTrace` 的 `ConcurrentHashMap<DruidPooledConnection, Exception>`，在借出连接时记录堆栈。`removeAbandoned` 定时任务检查连接是否超时未归还（超过 `removeAbandonedTimeout`），如果发现则将连接强制回收，并打印借出时的堆栈信息。

## 9.3 设计思路题

**Q: 如果让你设计一个包含监控功能的数据库连接池，你会如何设计？**

A:
1. **核心连接池**：基于数组的池化管理，支持动态缩扩容
2. **Filter 拦截器**：职责链模式，可插拔的 Filter 架构
3. **监控数据采集**：JDBC 代理拦截所有 Statement 执行，采集 SQL、耗时、行数
4. **监控数据存储**：使用环形缓冲区或分段计数器，减少对性能的影响
5. **Web 可视化**：内嵌 Servlet，展示 SQL 统计、连接池状态、URI 关联分析
6. **安全扩展**：提供 SQL 解析和规则引擎接口，便于集成安全模块

**Q: Druid 和 HikariCP 的设计理念差异？**

A: HikariCP 追求「零开销」，认为连接池不应产生可感知的性能影响，代码精简到极致。Druid 追求「功能全面」，认为连接池的价值不仅在于管理连接，更在于让数据库访问变得可观测、可控制。这是两种不同的哲学——极简 vs 全面，没有绝对优劣，取决于项目需求。
