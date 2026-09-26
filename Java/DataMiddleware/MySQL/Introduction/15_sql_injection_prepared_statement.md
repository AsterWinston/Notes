# 1.SQL注入概述

SQL 注入（SQL Injection）是攻击者通过在应用程序的输入参数中**插入恶意的 SQL 代码**，从而欺骗数据库服务器执行非预期 SQL 命令的攻击方式。它是 Web 安全领域最常见的漏洞之一（OWASP Top 10）。

## 1.1注入原理

**不安全的代码**（拼接 SQL）：

```java
// ✗ 危险：直接拼接用户输入
String username = request.getParameter("username");
String sql = "SELECT * FROM user WHERE username = '" + username + "'";
// 用户输入：' OR '1'='1' --
// 最终 SQL：SELECT * FROM user WHERE username = '' OR '1'='1' --'
// 结果：返回所有用户数据！
```

**安全的代码**（参数化查询）：

```java
// ✓ 安全：PreparedStatement 参数化
String sql = "SELECT * FROM user WHERE username = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setString(1, username);  // 参数绑定
```

## 1.2常见注入方式

| 注入方式 | 输入示例 | 效果 |
|---------|---------|------|
| **OR 永真** | `' OR '1'='1` | 绕过认证，返回所有数据 |
| **注释符截断** | `admin' --` | 忽略后续条件 |
| **UNION 注入** | `' UNION SELECT 1,2,3 --` | 窃取其他表数据 |
| **批量执行** | `'; DROP TABLE user; --` | 执行任意 SQL（需多语句支持） |
| **堆叠查询** | `'; UPDATE user SET balance=99999 WHERE id=1; --` | 修改数据 |
| **盲注（时间/布尔）** | `' AND SLEEP(5) --` | 根据响应时间推断数据 |
| **报错注入** | `' AND extractvalue(1, concat(0x7e, database())) --` | 通过错误信息泄露数据 |

# 2.MySQL中的预编译（Prepared Statement）

## 2.1原理

预编译语句（Prepared Statement）将 SQL 语句的**结构**和**数据**分两次发送到 MySQL 服务器：

```
步骤 1：发送 SQL 模板 → 服务器解析 & 编译
步骤 2：发送参数值 → 服务器执行
```

**为什么能防注入？**
- SQL 模板在步骤 1 中已完成解析和编译
- 步骤 2 的参数值**仅被视为数据**，不会被当作 SQL 代码解析
- 无论参数值中包含哪些特殊字符，都不会改变 SQL 的语义结构

## 2.2MySQL 服务端预编译

```sql
-- 使用服务端预编译
PREPARE stmt FROM 'SELECT * FROM user WHERE id = ?';
SET @id = 1;
EXECUTE stmt USING @id;
DEALLOCATE PREPARE stmt;
```

## 2.3JDBC中的PreparedStatement

```java
// 基础用法
String sql = "SELECT * FROM user WHERE username = ? AND status = ?";
PreparedStatement ps = connection.prepareStatement(sql);
ps.setString(1, username);
ps.setInt(2, status);
ResultSet rs = ps.executeQuery();

// 批量执行
String sql = "INSERT INTO user (name, email) VALUES (?, ?)";
PreparedStatement ps = connection.prepareStatement(sql);
for (User user : users) {
    ps.setString(1, user.getName());
    ps.setString(2, user.getEmail());
    ps.addBatch();
}
ps.executeBatch();
```

### JDBC 预编译的两面性

JDBC 的 `PreparedStatement` 默认是**客户端模拟预编译**：
- `useServerPrepStmts=false`（默认）：JDBC 驱动在客户端拼接参数值，发给数据库的是完整 SQL
- `useServerPrepStmts=true`：使用 MySQL 服务端真正的预编译功能，减少 SQL 解析开销

```yaml
# 开启服务端预编译（推荐）
spring.datasource.url: jdbc:mysql://localhost:3306/mydb?useServerPrepStmts=true&cachePrepStmts=true
```

## 2.4MyBatis中的防注入

MyBatis 使用 `#{}`（参数绑定）和 `${}`（字符串替换）区分安全和危险方式：

```xml
<!-- ✓ 安全：#{} 使用 PreparedStatement，自动防注入 -->
<select id="getUser" resultType="User">
  SELECT * FROM user WHERE username = #{username}
</select>

<!-- ✗ 危险：${} 是字符串替换，存在注入风险 -->
<select id="getUser" resultType="User">
  SELECT * FROM user WHERE username = '${username}'
</select>
```

**`${}` 的合理使用场景**（仅限非用户输入的场景）：

```xml
<!-- ✓ ORDER BY 字段和表名无法用 ? 占位，动态但安全（值来自代码而非用户输入） -->
<select id="listUsers" resultType="User">
  SELECT * FROM user ORDER BY ${orderColumn}
</select>

<!-- 如果必须用动态表名/字段名，务必在 Java 代码中白名单校验 -->
```

# 3.其他防注入策略

## 3.1输入校验与过滤

```java
// 白名单校验（推荐：只允许合法的值）
private static final Set<String> ALLOWED_COLUMNS = Set.of("id", "username", "create_time");

public List<User> listByOrder(String orderBy) {
    if (!ALLOWED_COLUMNS.contains(orderBy)) {
        throw new IllegalArgumentException("Invalid column: " + orderBy);
    }
    return mapper.listUsers(orderBy);
}
```

## 3.2最小权限原则

数据库用户只授予**需要的最小权限**：

```sql
-- 应用账号：只给 CRUD 权限（不给 DDL、DROP 等）
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_user'@'%';

-- 读写分离：从库账号只给 SELECT
GRANT SELECT ON mydb.* TO 'readonly'@'%';
```

> 即使 SQL 注入发生，受限于权限，攻击者无法 `DROP TABLE` 或读取其他库数据。

## 3.3SQL防火墙 / WAF

- **数据库防火墙**：MySQL Enterprise Firewall 记录白名单 SQL 模式，拒绝异常 SQL
- **Web 防火墙（WAF）**：在 HTTP 层拦截注入类流量（如阿里云 WAF、Cloudflare）
- **RASP**：运行时应用自我保护，检测并拦截 SQL 注入行为

## 3.4代码审查清单

| 检查项 | 说明 |
|--------|------|
| 所有 SQL 使用参数化查询 | 禁止拼接字符串 |
| `${}` 确认无用户输入 | MyBatis 中 `${}` 只用于系统内部值 |
| 表名/列名动态时使用白名单 | 不可直接使用前端传来的字段名 |
| LIKE 查询的参数转义 | `%` 和 `_` 在 LIKE 中是通配符，需转义 |
| 存储过程参数绑定 | 不要拼接 SQL 到存储过程调用 |
| 错误消息不暴漏 SQL 结构 | `try-catch` 中返回通用错误页，不返回 `SQLException` 详情 |

```java
// LIKE 查询的参数转义
String search = request.getParameter("search");
// 转义 LIKE 通配符
search = search.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_");
String sql = "SELECT * FROM article WHERE title LIKE CONCAT('%', ?, '%') ESCAPE '\\'";
```

# 4.总结

| 方法 | 防护程度 | 推荐度 |
|------|---------|--------|
| **PreparedStatement 参数绑定** | ★★★★★ | 必须实施 |
| MyBatis `#{}` | ★★★★★ | 必须实施 |
| 输入白名单校验 | ★★★★☆ | 强烈推荐 |
| 最小权限原则 | ★★★★☆ | 强烈推荐 |
| SQL 防火墙 / WAF | ★★★☆☆ | 深度防御 |
| 仅靠输入黑名单过滤 | ★☆☆☆☆ | 不靠谱（绕过多） |
| 仅靠前端校验 | ☆☆☆☆☆ | 防君子不防小人 |
