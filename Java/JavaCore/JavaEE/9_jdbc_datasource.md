
# 1. 概述

JDBC（Java Database Connectivity）是 Java 访问关系型数据库的标准 API。本笔记从 **JavaEE 集成视角**讲述 JDBC 规范及其在容器中的使用（DataSource、连接池、JTA 协作），基础 JDBC 用法可参考 JavaSE/数据库相关笔记。

# 2. JDBC 规范层次

```
┌───────────────────────────────────────┐
│   Java 应用（面向 JDBC 接口编程）       │
├───────────────────────────────────────┤
│   JDBC API（java.sql / javax.sql）     │
├───────────────────────────────────────┤
│   JDBC 驱动管理器 / DataSource          │
├───────────────────────────────────────┤
│   JDBC 驱动（JDBC Driver）             │
│   ┌─────────┬─────────┬────────────┐   │
│   │ MySQL   │ Oracle  │ PostgreSQL │   │
│   └─────────┴─────────┴────────────┘   │
└───────────────────────────────────────┘
```

- `java.sql`：基础 JDBC API（`Connection`、`Statement`、`ResultSet`、`PreparedStatement` 等）。
- `javax.sql`：**扩展 JDBC API**（`DataSource`、`ConnectionPoolDataSource`、`XAConnection`、`RowSet` 等）。

# 3. JDBC 驱动类型

| 类型 | 说明 | 现状 |
|------|------|------|
| Type 1 | JDBC-ODBC 桥 | 已废弃（JDK 8 移除 ODBC 桥） |
| Type 2 | 本地 API 驱动 | 依赖本地库，少用 |
| Type 3 | 网络协议驱动 | 中间件模式 |
| **Type 4** | **纯 Java 驱动** | **主流**（MySQL Connector/J、Oracle JDBC、PostgreSQL JDBC） |

- Type 4 驱动直接通过 TCP 与数据库通信，纯 Java 实现，无需本地依赖，跨平台。

# 4. 基础 JDBC 编程模型

```java
String url = "jdbc:mysql://localhost:3306/db?useSSL=false&serverTimezone=Asia/Shanghai";
String user = "root";
String password = "123456";

// 1. 加载驱动（JDBC 4.0+ 可省略，SPI 自动发现）
Class.forName("com.mysql.cj.jdbc.Driver");

try (Connection conn = DriverManager.getConnection(url, user, password);
     PreparedStatement ps = conn.prepareStatement(
             "INSERT INTO user(name, age) VALUES (?, ?)")) {

    ps.setString(1, "张三");
    ps.setInt(2, 25);
    ps.executeUpdate();
} // try-with-resources 自动关闭
```

- **必须用 `PreparedStatement`**：预编译 + 参数化，防止 SQL 注入，提升性能。
- 通过 `Connection`、`Statement`、`ResultSet` 等接口编程，驱动替换不影响业务代码。

# 5. 连接池（Connection Pool）

## 5.1 为什么要连接池

```
无连接池：每次访问都新建 TCP 连接
  建立连接（耗时 30~100ms）→ 执行 SQL → 断开连接

有连接池：复用预先建立的连接
  池中取连接（耗时 <1ms）→ 执行 SQL → 归还连接
```

- 数据库连接创建开销大（网络握手、认证、会话初始化）。
- 高并发下无连接池会导致连接数爆炸、拖垮数据库。
- 连接池管理**最小连接数**与**最大连接数**。

## 5.2 主流连接池实现

| 连接池 | 特点 |
|--------|------|
| **HikariCP** | 性能极佳，Spring Boot 2+ 默认 |
| Tomcat JDBC Pool | Tomcat 内置 |
| DBCP / DBCP2 | Apache 老牌连接池 |
| C3P0 | 老牌，已逐渐淘汰 |
| Druid | 阿里出品，内置监控、防 SQL 注入 |

## 5.3 连接池关键参数

| 参数 | 说明 |
|------|------|
| `initialSize` | 初始化连接数 |
| `minIdle` | 最小空闲连接数 |
| `maxActive` / `maximumPoolSize` | 最大连接数 |
| `maxWait` | 获取连接超时 |
| `validationQuery` | 校验连接是否有效（如 `SELECT 1`） |
| `idleTimeout` | 空闲连接回收时间 |
| `maxLifetime` | 连接最大存活时间（应小于数据库超时） |

# 6. DataSource

## 6.1 为什么用 DataSource 而不是 DriverManager

- `DriverManager`：每次 `getConnection` 都要从连接池外新建连接，无法统一管理。
- `DataSource`：**标准的连接获取入口**，实现内部可以封装连接池、事务、XA。

| 接口 | 用途 |
|------|------|
| `javax.sql.DataSource` | 普通连接池数据源（最常见） |
| `javax.sql.ConnectionPoolDataSource` | 连接池的底层来源，供连接池使用 |
| `javax.sql.XADataSource` | 分布式事务（XA）数据源，配合 JTA |

## 6.2 获取 DataSource

**方式一：容器注入（JavaEE 推荐）**

```java
@Resource(lookup = "java:comp/env/jdbc/MyDS")
private DataSource dataSource;
```

**方式二：程序化创建**

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/db");
config.setUsername("root");
config.setPassword("123456");
config.setMaximumPoolSize(20);
DataSource ds = new HikariDataSource(config);
```

**方式三：服务器管理控制台配置后 JNDI 查找**（见 `8_jndi.md`）

# 7. 事务与 JDBC

## 7.1 JDBC 本地事务

```java
conn.setAutoCommit(false);
try {
    ps1.executeUpdate();
    ps2.executeUpdate();
    conn.commit();
} catch (Exception e) {
    conn.rollback();
    throw e;
}
```

- 单个数据库连接内的事务，只覆盖**一个数据源**。

## 7.2 JDBC + JTA 全局事务

- 在 JavaEE 容器中，数据源连接自动纳入容器管理的全局事务（`REQUIRED`）。
- 通过 `UserTransaction` 或声明式事务，可把**多个数据源**、JMS 等纳入同一事务（XA 两阶段提交）。
- 详见 `12_jta_transactions.md`。

# 8. 容器管理的连接获取与归还

在 JavaEE 容器中：

- `res-auth=Container`：连接由容器自动获取并**在事务结束/方法返回后自动归还连接池**。
- 应用无需显式 `close()`（但显式关闭也是安全的幂等操作）。
- 连接与事务上下文（`Transaction Context`）绑定：同一事务内拿到同一个连接，保证事务一致性。

# 9. RowSet

`RowSet` 是 `ResultSet` 的扩展，支持离线操作、事件、Web 使用：

| RowSet 类型 | 特点 |
|-------------|------|
| `JdbcRowSet` | 在线，包装 ResultSet |
| `CachedRowSet` | 离线，断开连接后仍可操作 |
| `WebRowSet` | XML 表示 |
| `JoinRowSet` | 内存中合并多个 RowSet |
| `FilteredRowSet` | 带过滤条件 |

```java
CachedRowSet crs = new com.sun.rowset.CachedRowSetImpl();
crs.setUrl("jdbc:mysql://localhost:3306/db");
crs.setUsername("root");
crs.setPassword("123456");
crs.setCommand("SELECT * FROM user WHERE age > ?");
crs.setInt(1, 18);
crs.execute();
// 连接已释放，可以离线遍历
while (crs.next()) {
    System.out.println(crs.getString("name"));
}
```

# 10. JavaEE 实践要点

| 实践 | 说明 |
|------|------|
| 使用连接池 | 禁止每次新建裸连接 |
| 使用 `PreparedStatement` | 防 SQL 注入 |
| try-with-resources | 自动关闭资源，防止泄漏 |
| 配置连接参数 | `connectionTimeout`、`maxLifetime`、`validationQuery` |
| 监控 | Druid 或数据库连接池监控面板查看活跃/空闲连接 |
| 敏感信息加密 | 数据库密码通过加密配置/环境变量注入，不硬编码 |
| 事务边界清晰 | 明确 `REQUIRED` / `REQUIRES_NEW` 的边界，避免长事务 |

# 11. 与 ORM 框架的关系

```
┌──────────┐      ┌──────────────┐      ┌──────────────┐
│  应用    │ ───► │  ORM 框架     │ ───► │    JDBC      │
│  Service │      │ JPA/MyBatis  │      │   + 连接池    │
└──────────┘      └──────────────┘      └──────┬───────┘
                                               │
                                           数据库
```

- JPA/Hibernate、MyBatis 底层都是 JDBC。
- ORM 框架解决对象映射，但连接获取最终仍走 `DataSource`。
- 连接池（如 HikariCP）对 ORM 透明，只需正确配置。
