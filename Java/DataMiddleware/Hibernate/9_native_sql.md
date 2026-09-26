# 1. 原生 SQL 概述

## 1.1 为什么需要原生 SQL

HQL/JPQL 和 Criteria API 虽然提供了类型安全、数据库无关的查询能力，但在以下场景中仍需要直接编写原生 SQL：

### 1.1.1 HQL/JPQL 无法表达的复杂查询

某些 SQL 特性无法通过 HQL/JPQL 表达，必须使用原生 SQL：

- **窗口函数（Window Functions）**：`ROW_NUMBER()`、`RANK()`、`DENSE_RANK()`、`LEAD()`、`LAG()`、`SUM() OVER()` 等
- **递归 CTE（Common Table Expression）**：`WITH RECURSIVE` 的组织树、图遍历等
- **数据库特有函数**：MySQL 的 `GROUP_CONCAT`、PostgreSQL 的 `string_agg`、Oracle 的 `CONNECT BY` 等
- **MERGE/UPSERT 操作**：`INSERT ... ON DUPLICATE KEY UPDATE`（MySQL）、`MERGE INTO`（Oracle）、`INSERT ... ON CONFLICT`（PostgreSQL）
- **复杂的子查询和派生表**：FROM 子句中的子查询（Hibernate 6+ 的 HQL 有限支持）

```sql
-- 窗口函数示例：每个部门内按工资排名（JPQL 无法表达）
SELECT 
    e.id, e.name, e.salary, e.dept_id,
    ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rank
FROM employees e;

-- 递归 CTE 示例：查询组织树（JPQL 无法表达）
WITH RECURSIVE org_tree AS (
    SELECT id, name, parent_id, 1 AS level FROM departments WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, d.parent_id, t.level + 1
    FROM departments d
    INNER JOIN org_tree t ON d.parent_id = t.id
)
SELECT * FROM org_tree ORDER BY level, name;

-- MySQL 特有函数：分组拼接（JPQL 无法表达）
SELECT dept_id, GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS employee_names
FROM employees GROUP BY dept_id;
```

### 1.1.2 遗留 SQL 复用

企业级应用中存在大量经过优化和测试的遗留 SQL 语句（存储过程、视图、报表 SQL），使用原生 SQL 可以直接复用这些资产：

```java
// 直接将现有复杂报表 SQL 注册为 NativeQuery
String reportSql = "SELECT p.name, SUM(oi.quantity * oi.unit_price) AS total_revenue "
    + "FROM products p "
    + "JOIN order_items oi ON p.id = oi.product_id "
    + "JOIN orders o ON oi.order_id = o.id "
    + "WHERE o.order_date BETWEEN :startDate AND :endDate "
    + "GROUP BY p.id, p.name "
    + "HAVING SUM(oi.quantity * oi.unit_price) > :minRevenue "
    + "ORDER BY total_revenue DESC";

List<Object[]> report = session.createNativeQuery(sql, Object[].class)
    .setParameter("startDate", startDate)
    .setParameter("endDate", endDate)
    .setParameter("minRevenue", minRevenue)
    .getResultList();
```

### 1.1.3 存储过程调用

许多企业的核心业务逻辑封装在数据库存储过程中，HQL/JPQL 无法调用存储过程：

```java
StoredProcedureQuery query = entityManager.createStoredProcedureQuery("calculate_bonus");
query.registerStoredProcedureParameter("dept_id", Long.class, ParameterMode.IN);
query.registerStoredProcedureParameter("bonus_pool", BigDecimal.class, ParameterMode.OUT);
query.setParameter("dept_id", 100L);
query.execute();
BigDecimal bonus = (BigDecimal) query.getOutputParameterValue("bonus_pool");
```

### 1.1.4 性能极致优化

当 HQL 生成的 SQL 不够高效时，通过原生 SQL 进行手写优化：

- **索引提示**：`SELECT /*+ INDEX(e idx_emp_dept) */ * FROM employees e`
- **指定 JOIN 算法**：MySQL `STRAIGHT_JOIN`、PostgreSQL `JOIN ... ON` 的精确控制
- **避免 Hibernate 自动生成的冗余 JOIN**：手动精简 SQL 减少不必要的数据加载
- **数据库特定执行计划提示**：Oracle Hint、MySQL Optimizer Hint 等

```sql
-- HQL 自动生成 SQL 可能包含不必要的 JOIN，通过原生 SQL 精确控制：
SELECT /*+ LEADING(u) USE_NL(p a) INDEX(u idx_user_status) */
    u.id, u.name, u.email, p.bio, p.avatar_url, a.street, a.city
FROM users u
INNER JOIN profiles p ON u.id = p.user_id
LEFT JOIN addresses a ON u.id = a.user_id AND a.is_default = true
WHERE u.status = :status;
```

### 1.1.5 数据库特性利用

利用数据库的特定高级功能：

- **全文索引**：MySQL `MATCH ... AGAINST`、PostgreSQL `tsvector` / `tsquery`
- **JSON 查询**：MySQL `JSON_EXTRACT`、PostgreSQL `->` / `->>` / `@>` 操作符
- **地理空间查询**：PostGIS `ST_DWithin`、`ST_Contains`、MySQL `ST_Distance_Sphere`
- **数组操作**：PostgreSQL 数组类型操作符 `ANY`、`@>`、`&&`

```sql
-- PostgreSQL 全文搜索（HQL 无法使用 tsvector/tsquery 类型）
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM articles, plainto_tsquery('english', :keyword) AS query
WHERE search_vector @@ query
ORDER BY rank DESC LIMIT 20;

-- PostgreSQL JSONB 查询（HQL 无法使用 JSONB 操作符）
SELECT id, data->>'name' AS name
FROM events
WHERE data @> '{"type": "login"}'
  AND (data->>'timestamp')::timestamptz > :since;

-- MySQL 8+ 窗口函数 + CTE 组合
WITH ranked_orders AS (
    SELECT customer_id, order_date, total_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders WHERE order_date >= :startDate
)
SELECT c.id, c.name, ro.order_date, ro.total_amount
FROM customers c
JOIN ranked_orders ro ON c.id = ro.customer_id AND ro.rn = 1;
```

## 1.2 原生 SQL 的代价

使用原生 SQL 是**功能与可移植性的权衡**，需要清醒认识其代价：

### 1.2.1 失去数据库可移植性

原生 SQL 绑定了特定数据库方言，切换数据库（如 MySQL → PostgreSQL）时需要重写所有原生 SQL：

```java
// MySQL 特有语法 — 无法移植到 PostgreSQL
String mysqlSql = "INSERT INTO users (name, score) VALUES (:name, :score) "
    + "ON DUPLICATE KEY UPDATE score = score + :score, updated_at = NOW()";

// PostgreSQL 等价语法
String pgSql = "INSERT INTO users (name, score) VALUES (:name, :score) "
    + "ON CONFLICT (name) DO UPDATE SET score = users.score + :score, updated_at = NOW()";
```

### 1.2.2 需要手动处理结果映射

HQL/JPQL 自动完成结果集到实体/DTO 的映射，而原生 SQL 需要手动定义映射规则：

```java
// HQL：自动映射，零配置
List<User> users = session.createQuery("FROM User WHERE active = true", User.class).list();

// 原生 SQL：需要手动定义列别名或 @SqlResultSetMapping
List<Object[]> rows = session.createNativeQuery(
    "SELECT id, user_name, email_address, created_at FROM t_users WHERE is_active = 1"
).list();
// 需要手动将 Object[] 转换为 User 对象
List<User> users = rows.stream().map(row -> {
    User u = new User();
    u.setId((Long) row[0]);
    u.setUsername((String) row[1]);
    u.setEmail((String) row[2]);
    u.setCreatedAt((LocalDateTime) row[3]);
    return u;
}).collect(Collectors.toList());
```

### 1.2.3 绕过一级缓存（部分场景）

原生 SQL 修改数据（`INSERT`/`UPDATE`/`DELETE`）**不会自动更新 Hibernate 的一级缓存**，可能导致内存中的实体与数据库状态不一致：

```java
// 原生 UPDATE 绕过一级缓存
session.createNativeMutationQuery("UPDATE accounts SET balance = balance - 100 WHERE id = :id")
    .setParameter("id", accountId)
    .executeUpdate();

// account 实体仍持有旧的 balance 值
Account account = session.get(Account.class, accountId);

// 解决方案：手动清除或刷新
session.refresh(account);  // 从数据库重新加载
session.clear();           // 清除整个一级缓存
```

### 1.2.4 SQL 注入风险

拼接字符串构建原生 SQL 是极其危险的行为，必须使用**参数绑定**：

```java
// 危险：字符串拼接（SQL 注入风险）
String username = request.getParameter("username"); // 用户输入: ' OR '1'='1' --
List<?> results = session.createNativeQuery(
    "SELECT * FROM users WHERE username = '" + username + "'"
).list();

// 安全：参数绑定
String username = request.getParameter("username");
List<?> results = session.createNativeQuery(
    "SELECT * FROM users WHERE username = :username"
).setParameter("username", username).list();
```

---

# 2. 原生 SQL 查询

## 2.1 createNativeQuery 基础用法

### 2.1.1 Hibernate Session API

```java
// 方式一：不指定结果类型，返回 List<Object[]>
String sql = "SELECT id, username, email FROM t_users WHERE status = :status";
NativeQuery<Object[]> query = session.createNativeQuery(sql, Object[].class);
List<Object[]> results = query
    .setParameter("status", "ACTIVE")
    .getResultList();

for (Object[] row : results) {
    Long id = (Long) row[0];
    String username = (String) row[1];
    String email = (String) row[2];
    System.out.printf("id=%d, username=%s, email=%s%n", id, username, email);
}

// 方式二：指定实体类型，返回 List<Entity>（自动映射列名到实体属性）
String sql2 = "SELECT id, username, email, created_at FROM t_users WHERE status = :status";
NativeQuery<User> query2 = session.createNativeQuery(sql2, User.class);
List<User> users = query2
    .setParameter("status", "ACTIVE")
    .getResultList();

// 方式三：使用已命名的结果集映射
String sql3 = "SELECT u.id AS user_id, u.username, u.email, o.id AS order_id, o.total_amount "
    + "FROM t_users u JOIN t_orders o ON u.id = o.user_id";
NativeQuery<?> query3 = session.createNativeQuery(sql3, "UserOrderMapping");
List<?> results3 = query3.getResultList();
```

### 2.1.2 JPA EntityManager API

```java
// JPA 标准 API：不指定结果类型
String sql = "SELECT id, username, email FROM t_users";
Query query = entityManager.createNativeQuery(sql);
List<?> results = query.getResultList();  // 默认返回 List<Object[]>

// JPA 标准 API：指定实体类型
Query query2 = entityManager.createNativeQuery(
    "SELECT id, username, email, created_at FROM t_users", User.class
);
List<User> users = query2.getResultList();

// JPA 标准 API：使用命名结果集映射
Query query3 = entityManager.createNativeQuery("SELECT ...", "UserOrderMapping");
```

### 2.1.3 Hibernate 5 vs Hibernate 6 API 变化

```java
// Hibernate 5.x：使用 NativeQuery<T>，继承自 Query<T>
NativeQuery<User> query = session.createNativeQuery(sql)
    .addEntity(User.class)
    .addJoin(Order.class, "o")
    .addScalar("total_amount", BigDecimalType.INSTANCE);

// Hibernate 6.x：统一使用 SelectionQuery / MutationQuery
// 查询使用 NativeQuery（SelectionQuery 子接口）
NativeQuery<User> query = session.createNativeQuery(sql, User.class);

// 变更操作（UPDATE/DELETE）使用 NativeMutationQuery
NativeMutationQuery mutationQuery = session.createNativeMutationQuery(
    "UPDATE t_users SET status = :status WHERE last_login < :threshold"
);
int updated = mutationQuery
    .setParameter("status", "INACTIVE")
    .setParameter("threshold", LocalDate.now().minusMonths(6))
    .executeUpdate();
```

### 2.1.4 getSingleResult 与异常处理

```java
String sql = "SELECT COUNT(*) FROM t_users WHERE status = :status";

// getSingleResult()：期望返回唯一结果
// 无结果 -> NoResultException；多条结果 -> NonUniqueResultException
try {
    Number count = (Number) session.createNativeQuery(sql)
        .setParameter("status", "ACTIVE")
        .getSingleResult();
    long activeUsers = count.longValue();
} catch (NoResultException e) {
    System.out.println("No active users found");
}

// 安全封装：getResultList + 手动判断
List<?> list = session.createNativeQuery(sql)
    .setParameter("status", "ACTIVE")
    .getResultList();
long activeUsers = list.isEmpty() ? 0L : ((Number) list.get(0)).longValue();
```

## 2.2 实体映射结果

### 2.2.1 自动列名匹配

当使用 `createNativeQuery(sql, User.class)` 时，Hibernate 自动将 SQL 结果集的列名与实体属性名进行匹配：

```java
@Entity
@Table(name = "t_users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_name")  // 数据库列名为 user_name
    private String username;

    @Column(name = "email_address")
    private String email;

    @Column(name = "created_at")
    private LocalDateTime createdAt;
}
```

匹配规则：
- **默认行为**：Hibernate 按 `@Column(name)` 的值匹配 SQL 列名
- **命名策略**：若配置了 `PhysicalNamingStrategy`，SQL 列名会受命名策略影响
- **隐式命名**：未配置 `@Column(name)` 时，默认使用属性名（根据命名策略转换为数据库列名）

```sql
-- 正确：列名与 @Column(name) 完全匹配
SELECT id, user_name, email_address, created_at FROM t_users;

-- 错误：列名不匹配（username 不是 user_name）
SELECT id, username, email, created_at FROM t_users;  -- 映射失败！
```

### 2.2.2 使用列别名匹配

当 SQL 列名与实体属性名不匹配时，使用 `AS` 别名使列名与属性名匹配：

```java
String sql = "SELECT u.user_id AS id, "
    + "u.user_name AS username, "
    + "u.email_addr AS email, "
    + "u.reg_time AS createdAt "
    + "FROM user_table u WHERE u.status = :status";

NativeQuery<User> query = session.createNativeQuery(sql, User.class);
List<User> users = query.setParameter("status", "ACTIVE").getResultList();
```

### 2.2.3 @Column 注解对原生查询的影响

```java
@Entity
@Table(name = "t_employees")
public class Employee {

    @Id
    private Long id;

    @Column(name = "first_name")
    private String firstName;

    @Column(name = "last_name")
    private String lastName;

    @Column(name = "annual_salary", columnDefinition = "DECIMAL(12,2)")
    private BigDecimal salary;

    @Transient  // 仅存在于 Java 对象中，无对应数据库列
    private String fullName;

    @Column(name = "status_code")
    @Convert(converter = StatusConverter.class)
    private EmployeeStatus status;
}

// 原生 SQL 列名必须遵循 @Column(name) 配置
String sql = "SELECT emp_id AS id, first_name, last_name, annual_salary, status_code "
    + "FROM t_employees WHERE annual_salary > :minSalary";
```

### 2.2.4 复合主键实体的映射

```java
@Embeddable
public class OrderItemId implements Serializable {
    @Column(name = "order_id")
    private Long orderId;
    @Column(name = "product_id")
    private Long productId;
}

@Entity
@Table(name = "t_order_items")
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
    @Column(name = "quantity")
    private int quantity;
    @Column(name = "unit_price")
    private BigDecimal unitPrice;
}

// 原生 SQL：需要提供所有主键列
String sql = "SELECT order_id, product_id, quantity, unit_price "
    + "FROM t_order_items WHERE order_id = :orderId";

List<OrderItem> items = session.createNativeQuery(sql, OrderItem.class)
    .setParameter("orderId", 1001L)
    .getResultList();
```

## 2.3 标量查询

### 2.3.1 单值查询

```java
// 1. 查询计数
String sql = "SELECT COUNT(*) FROM t_users WHERE status = :status";
Long count = ((Number) session.createNativeQuery(sql)
    .setParameter("status", "ACTIVE")
    .getSingleResult()).longValue();

// 2. 查询聚合值
String sql2 = "SELECT MAX(salary) FROM t_employees WHERE dept_id = :deptId";
BigDecimal maxSalary = (BigDecimal) session.createNativeQuery(sql2)
    .setParameter("deptId", 100L)
    .getSingleResult();

// 3. 查询字符串
String sql3 = "SELECT email FROM t_users WHERE id = :id";
String email = (String) session.createNativeQuery(sql3)
    .setParameter("id", 1L)
    .getSingleResult();

// 4. 查询布尔值
String sql4 = "SELECT EXISTS(SELECT 1 FROM t_users WHERE email = :email)";
boolean exists = ((Number) session.createNativeQuery(sql4)
    .setParameter("email", "test@example.com")
    .getSingleResult()).intValue() == 1;
```

### 2.3.2 多列查询（List<Object[]>）

```java
String sql = "SELECT "
    + "d.name AS dept_name, "
    + "COUNT(e.id) AS emp_count, "
    + "AVG(e.salary) AS avg_salary, "
    + "SUM(e.salary) AS total_salary "
    + "FROM t_departments d "
    + "LEFT JOIN t_employees e ON d.id = e.dept_id "
    + "GROUP BY d.id, d.name "
    + "HAVING COUNT(e.id) > :minCount "
    + "ORDER BY total_salary DESC";

NativeQuery<Object[]> query = session.createNativeQuery(sql, Object[].class);
List<Object[]> stats = query.setParameter("minCount", 5).getResultList();

for (Object[] row : stats) {
    String deptName = (String) row[0];
    Number empCount = (Number) row[1];
    Double avgSalary = row[2] != null ? ((Number) row[2]).doubleValue() : 0.0;
    BigDecimal totalSalary = row[3] != null ? (BigDecimal) row[3] : BigDecimal.ZERO;
    System.out.printf("部门: %s, 人数: %d, 均薪: %.2f, 总薪: %s%n",
        deptName, empCount.longValue(), avgSalary, totalSalary.toPlainString());
}
```

### 2.3.3 使用元组查询（Hibernate 6+）

Hibernate 6 提供了 `Tuple` 支持，可以通过名称或位置获取值，类型更安全：

```java
String sql = "SELECT "
    + "d.name AS dept_name, "
    + "COUNT(e.id) AS emp_count, "
    + "AVG(e.salary) AS avg_salary "
    + "FROM t_departments d "
    + "LEFT JOIN t_employees e ON d.id = e.dept_id "
    + "GROUP BY d.id, d.name";

@SuppressWarnings("unchecked")
List<Tuple> tuples = session.createNativeQuery(sql, Tuple.class).getResultList();

for (Tuple tuple : tuples) {
    String deptName = tuple.get("dept_name", String.class);
    Long empCount = tuple.get("emp_count", Long.class);
    Double avgSalary = tuple.get("avg_salary", Double.class);
    System.out.printf("部门: %s, 人数: %d, 均薪: %.2f%n",
        deptName, empCount, avgSalary != null ? avgSalary : 0.0);
}
```

## 2.4 参数绑定

### 2.4.1 位置参数绑定（JDBC 风格）

位置参数使用 `?` 占位符，索引从 **1** 开始（与 JDBC 一致）：

```java
String sql = "SELECT id, username, email FROM t_users "
    + "WHERE status = ?1 AND created_at > ?2";

List<User> users = session.createNativeQuery(sql, User.class)
    .setParameter(1, "ACTIVE")
    .setParameter(2, LocalDate.of(2024, 1, 1))
    .getResultList();

// 批量设置
String sql2 = "SELECT * FROM t_products WHERE category = ?1 AND price BETWEEN ?2 AND ?3";
Query query = session.createNativeQuery(sql2, Product.class)
    .setParameters(new Object[]{"ELECTRONICS", 100.0, 5000.0});
```

### 2.4.2 命名参数绑定（推荐）

命名参数使用 `:paramName` 前缀，可读性和可维护性更好：

```java
String sql = "SELECT id, username, email, created_at "
    + "FROM t_users "
    + "WHERE status = :status "
    + "AND email LIKE :keyword "
    + "AND created_at BETWEEN :startDate AND :endDate";

NativeQuery<User> query = session.createNativeQuery(sql, User.class)
    .setParameter("status", "ACTIVE")
    .setParameter("keyword", "%@example.com")
    .setParameter("startDate", LocalDate.of(2024, 1, 1))
    .setParameter("endDate", LocalDate.of(2025, 12, 31));

List<User> users = query.getResultList();

// 批量设置参数（通过 Map）
Map<String, Object> params = new HashMap<>();
params.put("status", "ACTIVE");
params.put("keyword", "%@example.com");
params.put("startDate", LocalDate.of(2024, 1, 1));
params.put("endDate", LocalDate.of(2025, 12, 31));

NativeQuery<User> query2 = session.createNativeQuery(sql, User.class);
for (Map.Entry<String, Object> entry : params.entrySet()) {
    query2.setParameter(entry.getKey(), entry.getValue());
}
```

### 2.4.3 setParameterList（IN 子句集合参数）

```java
List<String> statusList = Arrays.asList("ACTIVE", "PENDING", "SUSPENDED");
List<Long> deptIds = Arrays.asList(100L, 101L, 102L, 103L);

String sql = "SELECT id, username, email, status "
    + "FROM t_users "
    + "WHERE status IN :statuses "
    + "AND dept_id IN :deptIds "
    + "AND created_at > :since";

// Hibernate 自动展开参数列表为多个占位符
NativeQuery<User> query = session.createNativeQuery(sql, User.class)
    .setParameterList("statuses", statusList)
    .setParameterList("deptIds", deptIds)
    .setParameter("since", LocalDate.of(2024, 1, 1));

List<User> users = query.getResultList();
// 等价于：WHERE status IN (?, ?, ?) AND dept_id IN (?, ?, ?, ?) AND created_at > ?
```

### 2.4.4 参数类型推断与显式指定

Hibernate 通过 Java 类型自动推断 SQL 参数类型，但有时需要显式指定：

```java
// 自动类型推断（绝大多数场景适用）
query.setParameter("age", 25);                           // -> INTEGER
query.setParameter("name", "Alice");                     // -> VARCHAR
query.setParameter("active", true);                      // -> BOOLEAN
query.setParameter("createdAt", LocalDate.now());        // -> DATE
query.setParameter("amount", new BigDecimal("99.99"));   // -> NUMERIC

// 显式指定 Hibernate Type（处理 null 值或特殊类型）
import org.hibernate.type.StandardBasicTypes;

query.setParameter("age", 25, StandardBasicTypes.INTEGER);
query.setParameter("amount", null, StandardBasicTypes.BIG_DECIMAL);  // null 值必须显式指定

// Hibernate 6 的 TemporalType 用法
query.setParameter("createdAt", LocalDateTime.now(), TemporalType.TIMESTAMP);
query.setParameter("createdDate", LocalDate.now(), TemporalType.DATE);
```

## 2.5 分页

### 2.5.1 基本分页 API

```java
String sql = "SELECT id, username, email, created_at FROM t_users ORDER BY created_at DESC";
int pageNumber = 2;
int pageSize = 20;

NativeQuery<User> query = session.createNativeQuery(sql, User.class)
    .setFirstResult((pageNumber - 1) * pageSize)  // 起始位置（0-based）
    .setMaxResults(pageSize);                      // 最大返回行数

List<User> users = query.getResultList();

// 通用分页工具方法
public <T> List<T> paginate(Session session, String sql, Class<T> resultClass,
                             int pageNumber, int pageSize, Map<String, Object> params) {
    NativeQuery<T> query = session.createNativeQuery(sql, resultClass)
        .setFirstResult((pageNumber - 1) * pageSize)
        .setMaxResults(pageSize);
    if (params != null) {
        params.forEach(query::setParameter);
    }
    return query.getResultList();
}
```

### 2.5.2 Hibernate 自动包装分页 SQL

Hibernate 在底层自动为原始 SQL 包装分页语法，根据 Dialect 生成数据库特定的分页 SQL：

```java
// 用户编写的 SQL（无分页语法）
String sql = "SELECT id, username, email FROM t_users ORDER BY created_at DESC";

// 根据 Dialect 自动转换为：
// MySQL / PostgreSQL:
// SELECT id, username, email FROM t_users ORDER BY created_at DESC LIMIT ? OFFSET ?

// Oracle:
// SELECT * FROM (
//     SELECT a.*, ROWNUM rnum FROM (
//         SELECT id, username, email FROM t_users ORDER BY created_at DESC
//     ) a WHERE ROWNUM <= ?
// ) WHERE rnum > ?

// SQL Server:
// SELECT id, username, email FROM t_users ORDER BY created_at DESC
// OFFSET ? ROWS FETCH NEXT ? ROWS ONLY
```

### 2.5.3 分页 SQL 兼容性注意

有些原生 SQL 无法被自动包装为分页，需要手动处理：

```java
// 问题：UNION 查询可能无法被某些 Dialect 自动包装分页
String sql = "SELECT id, name, 'CUSTOMER' AS type FROM customers WHERE name LIKE :keyword "
    + "UNION ALL "
    + "SELECT id, name, 'SUPPLIER' AS type FROM suppliers WHERE name LIKE :keyword "
    + "ORDER BY name";

// 解决方案一：手动包装子查询（针对特定数据库）
String paginatedSql = "SELECT * FROM (" + sql + ") AS combined_result "
    + "ORDER BY name LIMIT :limit OFFSET :offset";

// 解决方案二：使用 CTE（兼容更好的数据库）
String cteSql = "WITH combined AS ("
    + "  SELECT id, name, 'CUSTOMER' AS type FROM customers WHERE name LIKE :keyword "
    + "  UNION ALL "
    + "  SELECT id, name, 'SUPPLIER' AS type FROM suppliers WHERE name LIKE :keyword "
    + ") SELECT * FROM combined ORDER BY name";
```

> **注意**：Hibernate 对原始 SQL 的分页包装是修改原始 SQL 字符串。如果 SQL 非常复杂（嵌套的 UNION、CTE 等），可能包装失败。建议在复杂场景下手动编写分页 SQL。

---

# 3. 结果集映射（重点）

## 3.1 默认映射

### 3.1.1 默认映射规则

不指定 `@SqlResultSetMapping` 时，Hibernate 使用默认映射规则：

```java
// 匹配顺序：@Column(name) 的值 > 命名策略转换后的属性名 > 忽略大小写匹配
@Entity
@Table(name = "t_users")
public class User {
    @Id
    @Column(name = "user_id")
    private Long id;
    @Column(name = "user_name")
    private String username;
    @Column(name = "email_address")
    private String email;
    @Column(name = "created_at")
    private LocalDateTime createdAt;
}

// SQL 列名必须与 @Column(name) 一致
String sql = "SELECT user_id, user_name, email_address, created_at FROM t_users";
List<User> users = session.createNativeQuery(sql, User.class).getResultList();
```

### 3.1.2 使用 setResultTransformer 自定义（Hibernate 5）

Hibernate 5 中可通过 `setResultTransformer` 自定义映射逻辑：

```java
// Hibernate 5.x：aliasToBean 映射到 DTO
List<UserDTO> dtos = session.createNativeQuery(
    "SELECT u.user_name AS username, u.email_address AS email, d.dept_name AS deptName "
    + "FROM t_users u JOIN t_departments d ON u.dept_id = d.id"
)
.setResultTransformer(Transformers.aliasToBean(UserDTO.class))
.list();

// ALIAS_TO_ENTITY_MAP：返回 Map<String, Object>
List<Map<String, Object>> rows = session.createNativeQuery(
    "SELECT user_name, email_address FROM t_users"
)
.setResultTransformer(Transformers.ALIAS_TO_ENTITY_MAP)
.list();
// rows[0]: {"user_name": "Alice", "email_address": "alice@example.com"}

// TO_LIST：默认行为
List list = session.createNativeQuery("SELECT COUNT(*) FROM t_users")
    .setResultTransformer(Transformers.TO_LIST)
    .list();
```

> **注意**：`ResultTransformer` 在 Hibernate 6 中被标记为废弃，推荐使用 `TupleTransformer`（见 3.5 节）。

## 3.2 @SqlResultSetMapping

`@SqlResultSetMapping` 是 JPA 规范定义的结果集映射注解，定义 Java 对象与 SQL 列之间的映射关系，可重用且类型安全。

### 3.2.1 实体映射（@EntityResult）

```java
@Entity
@Table(name = "t_users")
@SqlResultSetMapping(
    name = "UserMapping",
    entities = @EntityResult(
        entityClass = User.class,
        fields = {
            @FieldResult(name = "id", column = "user_id"),
            @FieldResult(name = "username", column = "user_name"),
            @FieldResult(name = "email", column = "email_address"),
            @FieldResult(name = "createdAt", column = "created_at")
        }
    )
)
public class User {
    @Id private Long id;
    private String username;
    private String email;
    private LocalDateTime createdAt;
}

// 使用映射
String sql = "SELECT user_id, user_name, email_address, created_at FROM t_users";
List<User> users = (List<User>) session
    .createNativeQuery(sql, "UserMapping")
    .getResultList();
```

**多个实体映射（JOIN 查询同时映射两个实体）：**

```java
@Entity
@Table(name = "t_orders")
@SqlResultSetMappings({
    @SqlResultSetMapping(
        name = "OrderWithCustomerMapping",
        entities = {
            @EntityResult(entityClass = Order.class, fields = {
                @FieldResult(name = "id", column = "order_id"),
                @FieldResult(name = "totalAmount", column = "order_total"),
                @FieldResult(name = "orderDate", column = "order_date")
            }),
            @EntityResult(entityClass = Customer.class, fields = {
                @FieldResult(name = "id", column = "customer_id"),
                @FieldResult(name = "name", column = "customer_name"),
                @FieldResult(name = "email", column = "customer_email")
            })
        }
    ),
    @SqlResultSetMapping(
        name = "OrderCountMapping",
        columns = @ColumnResult(name = "order_count", type = Long.class)
    )
})
public class Order {
    @Id private Long id;
    @ManyToOne private Customer customer;
    private BigDecimal totalAmount;
    private LocalDate orderDate;
}
```

### 3.2.2 构造器映射（@ConstructorResult）

将查询结果映射到 DTO 对象，要求 DTO 有完全匹配的构造方法（参数类型和顺序一致）。

```java
// ============ DTO 定义 ============
public class UserDTO {
    private String username;
    private String email;
    private String deptName;
    private long orderCount;

    public UserDTO(String username, String email, String deptName, long orderCount) {
        this.username = username;
        this.email = email;
        this.deptName = deptName;
        this.orderCount = orderCount;
    }

    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public String getDeptName() { return deptName; }
    public long getOrderCount() { return orderCount; }
}

public class OrderSummaryDTO {
    private String customerName;
    private long totalOrders;
    private BigDecimal totalRevenue;
    private BigDecimal avgOrderValue;

    public OrderSummaryDTO(String customerName, long totalOrders,
                           BigDecimal totalRevenue, BigDecimal avgOrderValue) {
        this.customerName = customerName;
        this.totalOrders = totalOrders;
        this.totalRevenue = totalRevenue;
        this.avgOrderValue = avgOrderValue;
    }
}
```

```java
// ============ @SqlResultSetMapping 定义 ============
@SqlResultSetMappings({
    @SqlResultSetMapping(
        name = "UserDTOMapping",
        classes = @ConstructorResult(
            targetClass = UserDTO.class,
            columns = {
                @ColumnResult(name = "username", type = String.class),
                @ColumnResult(name = "email", type = String.class),
                @ColumnResult(name = "dept_name", type = String.class),
                @ColumnResult(name = "order_count", type = Long.class)
            }
        )
    ),
    @SqlResultSetMapping(
        name = "OrderSummaryMapping",
        classes = @ConstructorResult(
            targetClass = OrderSummaryDTO.class,
            columns = {
                @ColumnResult(name = "customer_name", type = String.class),
                @ColumnResult(name = "total_orders", type = Long.class),
                @ColumnResult(name = "total_revenue", type = BigDecimal.class),
                @ColumnResult(name = "avg_order_value", type = BigDecimal.class)
            }
        )
    )
})
@Entity
public class SomeEntityForMapping {}

// ============ 使用映射 ============
// 查询 UserDTO
String sql1 = "SELECT u.username, u.email, d.dept_name, "
    + "COUNT(o.id) AS order_count "
    + "FROM t_users u "
    + "LEFT JOIN t_departments d ON u.dept_id = d.id "
    + "LEFT JOIN t_orders o ON u.id = o.user_id "
    + "GROUP BY u.id, u.username, u.email, d.dept_name";

List<UserDTO> userDTOs = (List<UserDTO>) session
    .createNativeQuery(sql1, "UserDTOMapping")
    .getResultList();

// 查询 OrderSummaryDTO（客户订单汇总）
String sql2 = "SELECT "
    + "c.name AS customer_name, "
    + "COUNT(o.id) AS total_orders, "
    + "COALESCE(SUM(o.total_amount), 0) AS total_revenue, "
    + "COALESCE(AVG(o.total_amount), 0) AS avg_order_value "
    + "FROM t_customers c "
    + "LEFT JOIN t_orders o ON c.id = o.customer_id "
    + "WHERE o.order_date BETWEEN :startDate AND :endDate OR o.order_date IS NULL "
    + "GROUP BY c.id, c.name "
    + "ORDER BY total_revenue DESC";

List<OrderSummaryDTO> summaries = (List<OrderSummaryDTO>) session
    .createNativeQuery(sql2, "OrderSummaryMapping")
    .setParameter("startDate", LocalDate.of(2024, 1, 1))
    .setParameter("endDate", LocalDate.of(2024, 12, 31))
    .getResultList();
```

### 3.2.3 列映射（@ColumnResult）

```java
// @ColumnResult 用于映射标量列，可配合 @EntityResult 或单独使用
@SqlResultSetMappings({
    // 纯标量映射：仅包含列结果
    @SqlResultSetMapping(
        name = "ScalarStatsMapping",
        columns = {
            @ColumnResult(name = "total_users", type = Long.class),
            @ColumnResult(name = "active_users", type = Long.class),
            @ColumnResult(name = "avg_age", type = Double.class),
            @ColumnResult(name = "latest_reg_date", type = LocalDate.class)
        }
    ),
    // 标量 + 实体混合映射
    @SqlResultSetMapping(
        name = "DeptEmployeeStatsMapping",
        entities = @EntityResult(entityClass = Department.class),
        columns = {
            @ColumnResult(name = "emp_count", type = Long.class),
            @ColumnResult(name = "total_salary", type = BigDecimal.class),
            @ColumnResult(name = "avg_salary", type = Double.class)
        }
    )
})

// 使用纯标量映射
String sql = "SELECT "
    + "COUNT(*) AS total_users, "
    + "SUM(CASE WHEN status = 'ACTIVE' THEN 1 ELSE 0 END) AS active_users, "
    + "AVG(EXTRACT(YEAR FROM AGE(NOW(), birth_date))) AS avg_age, "
    + "MAX(created_at)::date AS latest_reg_date "
    + "FROM t_users";

List<Object[]> stats = (List<Object[]>) session
    .createNativeQuery(sql, "ScalarStatsMapping")
    .getResultList();

Object[] row = stats.get(0);
Long totalUsers = (Long) row[0];
Long activeUsers = (Long) row[1];
Double avgAge = (Double) row[2];
LocalDate latestRegDate = (LocalDate) row[3];
```

### 3.2.4 组合映射（实体 + 标量）

将 JOIN 查询结果同时映射到实体和标量统计值：

```java
@Entity
@SqlResultSetMapping(
    name = "DepartmentWithStatsMapping",
    entities = @EntityResult(
        entityClass = Department.class,
        fields = {
            @FieldResult(name = "id", column = "dept_id"),
            @FieldResult(name = "name", column = "dept_name"),
            @FieldResult(name = "location", column = "dept_location")
        }
    ),
    columns = {
        @ColumnResult(name = "emp_count", type = Long.class),
        @ColumnResult(name = "total_salary", type = BigDecimal.class),
        @ColumnResult(name = "avg_salary", type = Double.class),
        @ColumnResult(name = "budget_utilization", type = Double.class)
    }
)
public class Department {
    @Id private Long id;
    private String name;
    private String location;
    private BigDecimal budget;
}

// 使用组合映射
String sql = "SELECT "
    + "d.id AS dept_id, d.name AS dept_name, d.location AS dept_location, "
    + "COUNT(e.id) AS emp_count, "
    + "SUM(e.salary) AS total_salary, "
    + "AVG(e.salary) AS avg_salary, "
    + "ROUND(SUM(e.salary) / d.budget * 100, 2) AS budget_utilization "
    + "FROM t_departments d "
    + "LEFT JOIN t_employees e ON d.id = e.dept_id "
    + "GROUP BY d.id, d.name, d.location, d.budget";

List<Object[]> results = (List<Object[]>) session
    .createNativeQuery(sql, "DepartmentWithStatsMapping")
    .getResultList();

for (Object[] row : results) {
    Department dept = (Department) row[0];
    Long empCount = (Long) row[1];
    BigDecimal totalSalary = (BigDecimal) row[2];
    Double avgSalary = (Double) row[3];
    Double budgetUtil = (Double) row[4];
    System.out.printf("部门: %s, 人数: %d, 总薪资: %s, 均薪: %.2f, 预算利用率: %.2f%%%n",
        dept.getName(), empCount, totalSalary.toPlainString(), avgSalary, budgetUtil);
}
```

## 3.3 @NamedNativeQuery

### 3.3.1 基本定义

```java
@Entity
@NamedNativeQueries({
    @NamedNativeQuery(
        name = "User.findActiveUsers",
        query = "SELECT id, username, email, created_at "
            + "FROM t_users WHERE status = :status AND created_at > :since",
        resultClass = User.class
    ),
    @NamedNativeQuery(
        name = "User.findUsersByDepartment",
        query = "SELECT u.id, u.username, u.email, u.created_at "
            + "FROM t_users u INNER JOIN t_departments d ON u.dept_id = d.id "
            + "WHERE d.name = :deptName",
        resultClass = User.class
    ),
    @NamedNativeQuery(
        name = "User.countByStatus",
        query = "SELECT COUNT(*) FROM t_users WHERE status = :status"
    ),
    @NamedNativeQuery(
        name = "User.findUserStats",
        query = "SELECT d.name AS dept_name, COUNT(u.id) AS user_count, "
            + "MAX(u.created_at) AS latest_registration "
            + "FROM t_departments d LEFT JOIN t_users u ON d.id = u.dept_id "
            + "GROUP BY d.id, d.name",
        resultSetMapping = "UserStatsMapping"
    )
})
@SqlResultSetMapping(
    name = "UserStatsMapping",
    columns = {
        @ColumnResult(name = "dept_name", type = String.class),
        @ColumnResult(name = "user_count", type = Long.class),
        @ColumnResult(name = "latest_registration", type = LocalDateTime.class)
    }
)
public class User {
    @Id private Long id;
    private String username;
    private String email;
    private LocalDateTime createdAt;
}
```

### 3.3.2 执行命名原生查询

```java
// Hibernate Session API
List<User> activeUsers = session.createNamedQuery("User.findActiveUsers", User.class)
    .setParameter("status", "ACTIVE")
    .setParameter("since", LocalDate.of(2024, 1, 1).atStartOfDay())
    .getResultList();

// JPA EntityManager API
List<User> users = entityManager
    .createNamedQuery("User.findUsersByDepartment", User.class)
    .setParameter("deptName", "Engineering")
    .getResultList();

// 标量命名查询
Long totalActive = ((Number) session.createNamedQuery("User.countByStatus")
    .setParameter("status", "ACTIVE")
    .getSingleResult()).longValue();

// 结果集映射命名查询
List<Object[]> stats = (List<Object[]>) session
    .createNamedQuery("User.findUserStats")
    .getResultList();

// 分页 + 命名查询
List<User> pagedUsers = session.createNamedQuery("User.findActiveUsers", User.class)
    .setParameter("status", "ACTIVE")
    .setParameter("since", LocalDate.of(2024, 1, 1).atStartOfDay())
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();
```

### 3.3.3 XML 映射定义命名原生查询

```xml
<!-- orm.xml -->
<entity-mappings xmlns="https://jakarta.ee/xml/ns/persistence/orm" version="3.0">
    <named-native-query name="Order.findHighValueOrders" result-class="com.example.Order">
        <query>
            <![CDATA[
                SELECT id, order_number, total_amount, order_date
                FROM t_orders WHERE total_amount > ?1
                ORDER BY total_amount DESC
            ]]>
        </query>
    </named-native-query>

    <sql-result-set-mapping name="OrderWithCustomerMapping">
        <entity-result entity-class="com.example.Order">
            <field-result name="id" column="order_id"/>
            <field-result name="orderNumber" column="order_number"/>
            <field-result name="totalAmount" column="total_amount"/>
        </entity-result>
        <entity-result entity-class="com.example.Customer">
            <field-result name="id" column="customer_id"/>
            <field-result name="name" column="customer_name"/>
        </entity-result>
    </sql-result-set-mapping>
</entity-mappings>
```

## 3.4 Hibernate ResultTransformer（旧 API）

Hibernate 5 的 `ResultTransformer` 在 Hibernate 6 中已被标记为 `@Deprecated`：

```java
// ==================== Hibernate 5.x ====================

// 1. Transformers.aliasToBean(Class)：别名映射到 JavaBean 属性
List<UserDTO> dtos = session.createNativeQuery(
    "SELECT u.user_name AS username, u.email_address AS email, d.dept_name AS deptName "
    + "FROM t_users u JOIN t_departments d ON u.dept_id = d.id"
)
.setResultTransformer(Transformers.aliasToBean(UserDTO.class))
.list();

// 2. Transformers.ALIAS_TO_ENTITY_MAP：返回 Map 列表
List<Map<String, Object>> maps = session.createNativeQuery(
    "SELECT user_name, email_address, created_at FROM t_users"
)
.setResultTransformer(Transformers.ALIAS_TO_ENTITY_MAP)
.list();

// 3. 自定义 ResultTransformer
List<UserDTO> customList = session.createNativeQuery(
    "SELECT u.id, u.user_name, u.email_address, d.dept_name "
    + "FROM t_users u LEFT JOIN t_departments d ON u.dept_id = d.id"
)
.setResultTransformer(new BasicTransformerAdapter() {
    @Override
    public Object transformTuple(Object[] tuple, String[] aliases) {
        UserDTO dto = new UserDTO();
        dto.setId((Long) tuple[0]);
        dto.setUsername((String) tuple[1]);
        dto.setEmail((String) tuple[2]);
        dto.setDeptName((String) tuple[3]);
        return dto;
    }
})
.list();
```

### ResultTransformer vs JPA @SqlResultSetMapping 对比

| 对比维度 | ResultTransformer（Hibernate 5） | @SqlResultSetMapping（JPA 标准） |
|----------|----------------------------------|----------------------------------|
| **规范** | Hibernate 专有 | JPA 标准（厂商无关） |
| **定义位置** | 代码中调用时设置 | 注解在实体类上（集中管理） |
| **重用性** | 每次查询都需设置 | 通过 name 引用，可复用 |
| **类型安全** | 无编译期检查 | @ColumnResult.type 部分检查 |
| **Hibernate 6** | 标记 @Deprecated | 完全支持 |
| **灵活性** | 可动态切换 | 静态绑定 |
| **复杂映射** | 需手动实现 | 内置支持 实体/构造器/组合映射 |

## 3.5 Hibernate 6 TupleTransformer（新 API）

Hibernate 6 引入 `TupleTransformer` 函数式接口，替代废弃的 `ResultTransformer`：

```java
import org.hibernate.query.TupleTransformer;

// 1. 简洁的函数式映射（Lambda 风格）
String sql = "SELECT u.user_name, u.email_address, d.dept_name "
    + "FROM t_users u LEFT JOIN t_departments d ON u.dept_id = d.id "
    + "WHERE u.status = :status";

List<UserDTO> dtos = session.createNativeQuery(sql, Object[].class)
    .setParameter("status", "ACTIVE")
    .setTupleTransformer((tuple, aliases) -> {
        String username = (String) tuple[aliasIndex(aliases, "user_name")];
        String email = (String) tuple[aliasIndex(aliases, "email_address")];
        String deptName = (String) tuple[aliasIndex(aliases, "dept_name")];
        return new UserDTO(username, email, deptName);
    })
    .getResultList();

// 2. 使用 Tuple 获取值
List<UserDTO> dtos2 = session.createNativeQuery(sql, Tuple.class)
    .setParameter("status", "ACTIVE")
    .setTupleTransformer((tuple, aliases) -> {
        Tuple t = (Tuple) tuple;
        return new UserDTO(
            t.get("user_name", String.class),
            t.get("email_address", String.class),
            t.get("dept_name", String.class)
        );
    })
    .getResultList();

// 3. 完整实现的 TupleTransformer
public class UserDepartmentProjection implements TupleTransformer<UserDTO> {
    @Override
    public UserDTO transformTuple(Object[] tuple, String[] aliases) {
        String username = null, email = null, deptName = null;
        for (int i = 0; i < aliases.length; i++) {
            switch (aliases[i]) {
                case "user_name": username = (String) tuple[i]; break;
                case "email_address": email = (String) tuple[i]; break;
                case "dept_name": deptName = (String) tuple[i]; break;
            }
        }
        return new UserDTO(username, email, deptName);
    }
}

List<UserDTO> dtos3 = session.createNativeQuery(sql, Object[].class)
    .setTupleTransformer(new UserDepartmentProjection())
    .setParameter("status", "ACTIVE")
    .getResultList();

// 辅助方法：根据别名查找索引
private static int aliasIndex(String[] aliases, String target) {
    for (int i = 0; i < aliases.length; i++) {
        if (target.equalsIgnoreCase(aliases[i])) return i;
    }
    throw new IllegalArgumentException("Alias not found: " + target);
}
```

### TupleTransformer vs ResultTransformer 差异

| 特性 | ResultTransformer（H5） | TupleTransformer（H6） |
|------|-------------------------|------------------------|
| **接口形式** | 抽象类（BasicTransformerAdapter） | 函数式接口（@FunctionalInterface） |
| **Lambda 支持** | 需匿名类 | 原生 Lambda 支持 |
| **方法签名** | `transformTuple(Object[], String[])` | `transformTuple(Object[], String[])` |
| **返回类型** | Object | 泛型 T |
| **列表转换** | 另需 transformList | 不再支持列表级转换 |
| **设置方式** | `setResultTransformer()` | `setTupleTransformer()` |

---

# 4. 存储过程与函数调用

## 4.1 数据库函数调用

### 4.1.1 通过 session.doWork 调用数据库函数

```java
// session.doWork()：获取 JDBC Connection 执行无返回值操作
session.doWork(connection -> {
    String sql = "SELECT log_audit_event(?, ?)";
    try (PreparedStatement ps = connection.prepareStatement(sql)) {
        ps.setLong(1, userId);
        ps.setString(2, "PROFILE_UPDATE");
        ps.execute();
    }
});

// doReturningWork：执行有返回值的操作
Long bonus = session.doReturningWork(connection -> {
    String sql = "SELECT calculate_bonus(?, ?)";
    try (PreparedStatement ps = connection.prepareStatement(sql)) {
        ps.setLong(1, employeeId);
        ps.setBigDecimal(2, performanceScore);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                return rs.getLong(1);
            }
            return 0L;
        }
    }
});
```

### 4.1.2 JDBC CallableStatement 调用存储函数

```java
// PostgreSQL 函数：获取部门总薪资
// CREATE FUNCTION get_dept_total_salary(dept_id BIGINT) RETURNS NUMERIC ...
BigDecimal totalSalary = session.doReturningWork(connection -> {
    String sql = "{ ? = call get_dept_total_salary(?) }";
    try (CallableStatement cs = connection.prepareCall(sql)) {
        cs.registerOutParameter(1, Types.NUMERIC);
        cs.setLong(2, 100L);
        cs.execute();
        return cs.getBigDecimal(1);
    }
});

// MySQL 函数：获取用户名
// CREATE FUNCTION get_user_name(uid BIGINT) RETURNS VARCHAR(100) ...
String userName = session.doReturningWork(connection -> {
    String sql = "{ ? = call get_user_name(?) }";
    try (CallableStatement cs = connection.prepareCall(sql)) {
        cs.registerOutParameter(1, Types.VARCHAR);
        cs.setLong(2, userId);
        cs.execute();
        return cs.getString(1);
    }
});
```

## 4.2 @NamedStoredProcedureQuery（JPA 2.1+）

### 4.2.1 注解定义

```java
@Entity
@NamedStoredProcedureQueries({
    @NamedStoredProcedureQuery(
        name = "Employee.calculateBonus",
        procedureName = "sp_calculate_employee_bonus",
        parameters = {
            @StoredProcedureParameter(name = "emp_id", mode = ParameterMode.IN, type = Long.class),
            @StoredProcedureParameter(name = "bonus_year", mode = ParameterMode.IN, type = Integer.class),
            @StoredProcedureParameter(name = "bonus_amount", mode = ParameterMode.OUT, type = BigDecimal.class),
            @StoredProcedureParameter(name = "bonus_percentage", mode = ParameterMode.OUT, type = Double.class)
        }
    ),
    @NamedStoredProcedureQuery(
        name = "Department.getEmployees",
        procedureName = "sp_get_department_employees",
        resultClasses = Employee.class,
        parameters = {
            @StoredProcedureParameter(name = "dept_id", mode = ParameterMode.IN, type = Long.class),
            @StoredProcedureParameter(name = "status_filter", mode = ParameterMode.IN, type = String.class)
        }
    ),
    @NamedStoredProcedureQuery(
        name = "Report.generateMonthlyReport",
        procedureName = "sp_generate_monthly_report",
        resultSetMappings = "MonthlyReportMapping",
        parameters = {
            @StoredProcedureParameter(name = "report_month", mode = ParameterMode.IN, type = String.class)
        }
    )
})
@SqlResultSetMapping(
    name = "MonthlyReportMapping",
    classes = @ConstructorResult(
        targetClass = MonthlyReportDTO.class,
        columns = {
            @ColumnResult(name = "dept_name", type = String.class),
            @ColumnResult(name = "total_revenue", type = BigDecimal.class),
            @ColumnResult(name = "total_expense", type = BigDecimal.class),
            @ColumnResult(name = "net_profit", type = BigDecimal.class)
        }
    )
)
public class Employee {
    @Id private Long id;
    private String name;
    private BigDecimal salary;
}
```

### 4.2.2 执行命名存储过程

```java
// 1. 执行带 IN/OUT 参数的存储过程
StoredProcedureQuery spQuery = entityManager
    .createNamedStoredProcedureQuery("Employee.calculateBonus");
spQuery.setParameter("emp_id", 1001L);
spQuery.setParameter("bonus_year", 2024);
spQuery.execute();

BigDecimal bonusAmount = (BigDecimal) spQuery.getOutputParameterValue("bonus_amount");
Double bonusPercentage = (Double) spQuery.getOutputParameterValue("bonus_percentage");
System.out.printf("奖金: %s, 比例: %.2f%%%n", bonusAmount.toPlainString(), bonusPercentage);

// 2. 执行返回结果集的存储过程
StoredProcedureQuery spQuery2 = entityManager
    .createNamedStoredProcedureQuery("Department.getEmployees");
spQuery2.setParameter("dept_id", 101L);
spQuery2.setParameter("status_filter", "ACTIVE");
List<Employee> employees = spQuery2.getResultList();

// 3. 执行使用 resultSetMappings 的存储过程
StoredProcedureQuery spQuery3 = entityManager
    .createNamedStoredProcedureQuery("Report.generateMonthlyReport");
spQuery3.setParameter("report_month", "2024-06");
List<MonthlyReportDTO> reports = spQuery3.getResultList();
```

## 4.3 StoredProcedureQuery API（编程式）

```java
// 不需要注解，完全通过编程 API 构建存储过程调用
StoredProcedureQuery spQuery = entityManager
    .createStoredProcedureQuery("sp_process_order");

// 注册参数（按位置）
spQuery.registerStoredProcedureParameter(1, Long.class, ParameterMode.IN);
spQuery.registerStoredProcedureParameter(2, Long.class, ParameterMode.IN);
spQuery.registerStoredProcedureParameter(3, String.class, ParameterMode.OUT);
spQuery.registerStoredProcedureParameter(4, Integer.class, ParameterMode.OUT);

// 设置输入参数
spQuery.setParameter(1, 1001L);  // order_id
spQuery.setParameter(2, 2001L);  // customer_id

// 执行
spQuery.execute();

// 获取输出参数
String status = (String) spQuery.getOutputParameterValue(3);
Integer resultCode = (Integer) spQuery.getOutputParameterValue(4);

// 按名称注册（Hibernate 扩展）
StoredProcedureQuery spQuery2 = entityManager
    .createStoredProcedureQuery("sp_update_inventory");
spQuery2.registerStoredProcedureParameter("product_id", Long.class, ParameterMode.IN);
spQuery2.registerStoredProcedureParameter("quantity", Integer.class, ParameterMode.IN);
spQuery2.registerStoredProcedureParameter("new_stock", Integer.class, ParameterMode.OUT);

spQuery2.setParameter("product_id", 5001L);
spQuery2.setParameter("quantity", 10);
spQuery2.execute();
Integer newStock = (Integer) spQuery2.getOutputParameterValue("new_stock");
```

## 4.4 存储过程结果映射

### 4.4.1 多结果集处理

多结果集的复杂场景建议直接使用 JDBC：

```java
session.doWork(connection -> {
    try (CallableStatement cs = connection.prepareCall("{call sp_monthly_report(?, ?)}")) {
        cs.setInt(1, 2024);
        cs.setInt(2, 6);
        cs.execute();

        // 处理第一个结果集：部门统计
        System.out.println("=== 部门统计 ===");
        try (ResultSet rs = cs.getResultSet()) {
            while (rs.next()) {
                String deptName = rs.getString("dept_name");
                BigDecimal totalSales = rs.getBigDecimal("total_sales");
                System.out.printf("部门: %s, 销售额: %s%n", deptName, totalSales.toPlainString());
            }
        }

        // 移动到下一个结果集
        if (cs.getMoreResults()) {
            System.out.println("=== 产品统计 ===");
            try (ResultSet rs = cs.getResultSet()) {
                while (rs.next()) {
                    String productName = rs.getString("product_name");
                    int quantitySold = rs.getInt("quantity_sold");
                    System.out.printf("产品: %s, 销量: %d%n", productName, quantitySold);
                }
            }
        }
    }
});
```

### 4.4.2 返回实体的存储过程

```java
// MySQL 存储过程直接返回 SELECT 结果集
// CREATE PROCEDURE sp_get_active_orders(IN customer_id BIGINT)
// BEGIN
//     SELECT id, order_number, total_amount, order_date, status
//     FROM t_orders WHERE customer_id = customer_id
//         AND status IN ('PENDING', 'PROCESSING', 'SHIPPED')
//     ORDER BY order_date DESC;
// END;

StoredProcedureQuery spQuery = entityManager
    .createStoredProcedureQuery("sp_get_active_orders", Order.class);
spQuery.registerStoredProcedureParameter(1, Long.class, ParameterMode.IN);
spQuery.setParameter(1, 2001L);

List<Order> orders = spQuery.getResultList();
for (Order order : orders) {
    System.out.printf("#%s: %s, Amount: %s%n",
        order.getOrderNumber(), order.getStatus(), order.getTotalAmount().toPlainString());
}
```

---

# 5. 高级原生查询技巧

## 5.1 动态原生 SQL

### 5.1.1 基于条件的 SQL 构建

```java
public class DynamicNativeQueryBuilder {

    private StringBuilder sql = new StringBuilder();
    private Map<String, Object> params = new HashMap<>();
    private boolean hasWhere = false;

    public DynamicNativeQueryBuilder select(String selectClause) {
        sql.append("SELECT ").append(selectClause);
        return this;
    }

    public DynamicNativeQueryBuilder from(String fromClause) {
        sql.append(" FROM ").append(fromClause);
        return this;
    }

    public DynamicNativeQueryBuilder where(String condition, String paramName, Object value) {
        if (value == null) return this;
        if (!hasWhere) { sql.append(" WHERE "); hasWhere = true; }
        else { sql.append(" AND "); }
        sql.append(condition);
        params.put(paramName, value);
        return this;
    }

    public DynamicNativeQueryBuilder andIf(boolean condition, String sqlCondition,
                                           String paramName, Object value) {
        if (condition) where(sqlCondition, paramName, value);
        return this;
    }

    public DynamicNativeQueryBuilder orderBy(String orderClause) {
        sql.append(" ORDER BY ").append(orderClause);
        return this;
    }

    public String build() { return sql.toString(); }
    public Map<String, Object> getParams() { return params; }
}

// ========== 使用示例 ==========
public List<User> searchUsers(String username, String email, String status,
                               LocalDate fromDate, LocalDate toDate,
                               int pageNumber, int pageSize) {

    DynamicNativeQueryBuilder builder = new DynamicNativeQueryBuilder()
        .select("id, user_name AS username, email_address AS email, created_at, status")
        .from("t_users")
        .where("user_name LIKE :username", "username",
            username != null ? "%" + username + "%" : null)
        .andIf(email != null, "email_address LIKE :email", "email", "%" + email + "%")
        .andIf(status != null, "status = :status", "status", status)
        .andIf(fromDate != null, "created_at >= :fromDate", "fromDate", fromDate)
        .andIf(toDate != null, "created_at <= :toDate", "toDate", toDate)
        .orderBy("created_at DESC");

    NativeQuery<User> query = session.createNativeQuery(builder.build(), User.class)
        .setFirstResult((pageNumber - 1) * pageSize)
        .setMaxResults(pageSize);

    builder.getParams().forEach(query::setParameter);
    return query.getResultList();
}
```

### 5.1.2 SQL 注入防护

```java
/**
 * 原生 SQL 注入防护核心原则：
 * 1. 永远不要拼接用户输入到 SQL 字符串
 * 2. 始终使用参数绑定（setParameter）
 * 3. 动态表名/列名/ORDER BY 必须白名单验证
 */

// 动态 ORDER BY 需要白名单验证
private static final Set<String> ALLOWED_SORT_COLUMNS = Set.of(
    "username", "email", "created_at", "status"
);
private static final Set<String> ALLOWED_SORT_DIRECTIONS = Set.of("ASC", "DESC");

public List<User> findUsersWithSort(String sortBy, String sortDir) {
    if (sortBy == null || !ALLOWED_SORT_COLUMNS.contains(sortBy)) {
        throw new IllegalArgumentException("Invalid sort column: " + sortBy);
    }
    if (sortDir == null || !ALLOWED_SORT_DIRECTIONS.contains(sortDir.toUpperCase())) {
        throw new IllegalArgumentException("Invalid sort direction: " + sortDir);
    }

    String sql = "SELECT id, user_name AS username, email_address AS email, created_at, status "
        + "FROM t_users ORDER BY " + sortBy + " " + sortDir.toUpperCase();

    return session.createNativeQuery(sql, User.class).getResultList();
}

// 动态表名也需要白名单验证
private static final Set<String> ALLOWED_TABLES = Set.of(
    "t_users", "t_orders", "t_products", "t_departments"
);

public List<Object[]> queryTable(String tableName) {
    if (!ALLOWED_TABLES.contains(tableName)) {
        throw new IllegalArgumentException("Invalid table: " + tableName);
    }
    return session.createNativeQuery(
        "SELECT * FROM " + tableName + " LIMIT 100", Object[].class
    ).getResultList();
}
```

### 5.1.3 参数化与防注入总结

```java
// 正确：参数绑定
session.createNativeQuery("SELECT * FROM t_users WHERE username = :username")
    .setParameter("username", inputUsername);

// LIKE 查询需转义通配符 %
String escaped = keyword.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_");
session.createNativeQuery("SELECT * FROM t_users WHERE user_name LIKE :keyword ESCAPE '\\'")
    .setParameter("keyword", "%" + escaped + "%");

// IN 子句使用 setParameterList
session.createNativeQuery("SELECT * FROM t_users WHERE dept_id IN :deptIds")
    .setParameterList("deptIds", deptList);

// 禁止：字符串拼接
// session.createNativeQuery("SELECT * FROM t_users WHERE username = '" + input + "'");
```

## 5.2 Fetch 关联

### 5.2.1 原生 SQL 的 JOIN FETCH 限制

原生 SQL 无法直接使用 `JOIN FETCH` 语法。需要通过 `@SqlResultSetMapping` 实现关联实体加载，或采用分步查询 + 代码组装：

```java
// 方案一：@SqlResultSetMapping 映射多个实体
@SqlResultSetMapping(
    name = "OrderWithCustomerAndItemsMapping",
    entities = {
        @EntityResult(entityClass = Order.class, fields = {
            @FieldResult(name = "id", column = "order_id"),
            @FieldResult(name = "orderNumber", column = "order_number"),
            @FieldResult(name = "totalAmount", column = "total_amount")
        }),
        @EntityResult(entityClass = Customer.class, fields = {
            @FieldResult(name = "id", column = "customer_id"),
            @FieldResult(name = "name", column = "customer_name")
        })
    }
)

// 注意：JOIN 会导致重复行（订单 x 订单项），需手动合并

// 方案二（推荐）：分步查询 + 代码组装
public List<Order> getOrdersWithCustomer(Long customerId) {
    // 第一步：查订单
    List<Order> orders = session.createNativeQuery(
        "SELECT id, order_number, total_amount FROM t_orders WHERE customer_id = :customerId",
        Order.class
    ).setParameter("customerId", customerId).getResultList();

    // 第二步：查客户
    Customer customer = session.get(Customer.class, customerId);

    // 第三步：组装
    orders.forEach(o -> o.setCustomer(customer));
    return orders;
}
```

## 5.3 查询 Hint

### 5.3.1 Hibernate 原生查询 Hint

```java
String sql = "SELECT id, username, email FROM t_users WHERE status = :status";

NativeQuery<User> query = session.createNativeQuery(sql, User.class)
    .setParameter("status", "ACTIVE")

    // 超时设置
    .setHint("org.hibernate.timeout", 30)                       // 秒
    .setHint("jakarta.persistence.query.timeout", 30000)        // 毫秒

    // 读取大小
    .setHint("org.hibernate.fetchSize", 100)

    // SQL 注释（方便数据库端排查）
    .setHint("org.hibernate.comment", "查询活跃用户 - JobA")

    // 只读
    .setHint("org.hibernate.readOnly", true)

    // 缓存相关
    .setHint("org.hibernate.cacheable", false)
    .setHint("org.hibernate.cacheMode", "IGNORE")

    // Flush 模式
    .setHint("org.hibernate.flushMode", "COMMIT");

List<User> users = query.getResultList();
```

### 5.3.2 常用 Hint 汇总

| Hint 名称 | 类型 | 说明 |
|-----------|------|------|
| `org.hibernate.timeout` | Integer (秒) | Hibernate 查询超时 |
| `jakarta.persistence.query.timeout` | Integer (毫秒) | JPA 标准查询超时 |
| `org.hibernate.fetchSize` | Integer | JDBC Statement fetchSize |
| `org.hibernate.comment` | String | SQL 注释（标注来源） |
| `org.hibernate.readOnly` | Boolean | 结果实体为只读，跳过脏检查 |
| `org.hibernate.cacheable` | Boolean | 是否缓存查询结果 |
| `org.hibernate.cacheMode` | CacheMode | 与二级缓存交互模式 |
| `org.hibernate.flushMode` | FlushMode | 查询时的 flush 行为 |
| `org.hibernate.cacheRegion` | String | 缓存区域名称 |

## 5.4 Flush 与原生查询

### 5.4.1 默认行为：原生 SQL 不触发自动 Flush

```java
// FlushMode 为 AUTO 时的行为差异：
// - HQL/JPQL：自动触发 flush，持久化上下文变更同步到数据库
// - 原生 SQL：不触发 flush，可能查到过时数据

session.setFlushMode(FlushModeType.AUTO);

// 1. 通过实体修改
User user = session.get(User.class, 1L);
user.setEmail("newemail@example.com");  // 修改在一级缓存中，尚未 flush

// 2. HQL 查询：触发 flush，能查到更新后的数据
List<User> hqlResult = session.createQuery(
    "FROM User WHERE email = :email", User.class
).setParameter("email", "newemail@example.com").getResultList();  // 包含该用户

// 3. 原生 SQL 查询：不触发 flush
List<User> nativeResult = session.createNativeQuery(
    "SELECT * FROM t_users WHERE email_address = :email"
).setParameter("email", "newemail@example.com").getResultList();  // 可能不包含该用户
```

### 5.4.2 解决方案

```java
// 方案一：原生查询前手动 flush
session.flush();
List<User> users = session.createNativeQuery("...").getResultList();

// 方案二：通过 Hint 设置 flush mode
List<User> users2 = session.createNativeQuery("...", User.class)
    .setHint("org.hibernate.flushMode", FlushModeType.ALWAYS)
    .getResultList();

// 方案三：全局配置（application.properties）
// spring.jpa.properties.hibernate.query.native.flush_mode=ALWAYS

// 原生 UPDATE/DELETE 后的缓存一致性
session.createNativeMutationQuery("UPDATE t_users SET status = :s WHERE id = :id")
    .setParameter("s", "INACTIVE")
    .setParameter("id", userId)
    .executeUpdate();

// 需手动处理一级缓存：
session.evict(cachedUser);   // 清除特定实体
session.refresh(cached);     // 重新加载
session.clear();             // 清除整个 Session 缓存
```

## 5.5 只读查询优化

```java
// 通过设置只读标记，Hibernate 跳过脏检查，提升查询性能

// 方式一：通过 Hint 设置
NativeQuery<User> query = session.createNativeQuery(sql, User.class)
    .setHint("org.hibernate.readOnly", true);  // 结果实体为只读
List<User> users = query.getResultList();

// 方式二：Session 级别设置
session.setDefaultReadOnly(true);

// 方式三：逐实体设置
session.setReadOnly(user, true);

// 只读模式适用场景：
// - 报表查询、数据导出、只读 API、无状态数据展示
// 不适用于后续需要修改并持久化的实体
```

---

# 6. JDBC 直接访问

## 6.1 Session.doWork / doReturningWork

### 6.1.1 基本用法

```java
// doWork：执行无返回值的 JDBC 操作
session.doWork(connection -> {
    // connection 是 Hibernate 管理的 JDBC Connection
    // 事务由 Hibernate 管理，不要手动 commit/rollback
    // 不要关闭 connection（由 Hibernate 管理）

    try (PreparedStatement ps = connection.prepareStatement(
            "INSERT INTO audit_log (user_id, action, timestamp) VALUES (?, ?, NOW())")) {
        ps.setLong(1, userId);
        ps.setString(2, "LOGIN");
        ps.executeUpdate();
    }
    // PreparedStatement 自己关闭，Connection 不由我们关闭
});

// doReturningWork：执行有返回值的 JDBC 操作
Long nextSequenceValue = session.doReturningWork(connection -> {
    try (PreparedStatement ps = connection.prepareStatement(
            "SELECT nextval('order_number_seq')")) {
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                return rs.getLong(1);
            }
            throw new RuntimeException("序列值获取失败");
        }
    }
});
```

### 6.1.2 Connection 生命周期与注意事项

```java
// 正确用法：在 doWork/doReturningWork 闭包内使用 Connection
session.doWork(connection -> {
    try (PreparedStatement ps = connection.prepareStatement("...")) {
        // Connection 由 Hibernate 管理事务和生命周期
        // 闭包执行完毕后 Connection 归还给连接池
    }
});

// 错误用法一：在闭包外持有 Connection 引用
// Connection conn;
// session.doWork(c -> { conn = c; });  // 离开闭包后 conn 可能已被回收

// 错误用法二：手动关闭 Connection
// session.doWork(connection -> {
//     connection.close();  // 不要关闭！否则连接池不可用
// });

// 错误用法三：手动提交/回滚事务
// session.doWork(connection -> {
//     connection.commit();  // 不要！事务由 Hibernate 统一管理
// });

// 错误用法四：修改 Connection 的属性
// session.doWork(connection -> {
//     connection.setAutoCommit(false);  // 不要修改！破坏 Hibernate 事务管理
// });

// Connection 的正确使用范围：仅用于执行 SQL 语句
session.doWork(connection -> {
    try (Statement stmt = connection.createStatement()) {
        stmt.execute("CREATE TEMPORARY TABLE tmp_export (name VARCHAR(100), value NUMERIC)");
    }
});
```

## 6.2 Session.doReturningWork 的 Lambda 形式（Hibernate 5.2+）

Hibernate 5.2+ 支持 Lambda 表达式：

```java
// 1. 查询标量值
Long userCount = session.doReturningWork(connection -> {
    try (PreparedStatement ps = connection.prepareStatement(
            "SELECT COUNT(*) FROM t_users WHERE status = ?")) {
        ps.setString(1, "ACTIVE");
        try (ResultSet rs = ps.executeQuery()) {
            return rs.next() ? rs.getLong(1) : 0L;
        }
    }
});

// 2. 查询实体列表（手动映射）
List<User> users = session.doReturningWork(connection -> {
    List<User> list = new ArrayList<>();
    try (PreparedStatement ps = connection.prepareStatement(
            "SELECT id, user_name, email_address, created_at "
            + "FROM t_users WHERE status = ? ORDER BY created_at DESC")) {
        ps.setString(1, "ACTIVE");
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                User u = new User();
                u.setId(rs.getLong("id"));
                u.setUsername(rs.getString("user_name"));
                u.setEmail(rs.getString("email_address"));
                u.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
                list.add(u);
            }
        }
    }
    return list;
});

// 3. 调用存储过程获取多个输出参数
Map<String, Object> results = session.doReturningWork(connection -> {
    Map<String, Object> map = new HashMap<>();
    try (CallableStatement cs = connection.prepareCall(
            "{call sp_process_refund(?, ?, ?, ?)}")) {
        cs.setLong(1, orderId);
        cs.setBigDecimal(2, refundAmount);
        cs.registerOutParameter(3, Types.VARCHAR);  // 状态
        cs.registerOutParameter(4, Types.BIGINT);    // 退款单 ID
        cs.execute();
        map.put("status", cs.getString(3));
        map.put("refundId", cs.getLong(4));
    }
    return map;
});

System.out.printf("退款状态: %s, 退款单ID: %d%n",
    results.get("status"), results.get("refundId"));
```

---

# 7. 批量操作

## 7.1 原生 SQL 批量更新/删除

### 7.1.1 createNativeMutationQuery（Hibernate 6+）

Hibernate 6 将查询（SELECT）和变更（INSERT/UPDATE/DELETE）操作分离为不同的 API：

```java
// 原生批量 UPDATE
NativeMutationQuery mutation = session.createNativeMutationQuery(
    "UPDATE t_users SET status = :newStatus "
    + "WHERE status = :oldStatus AND last_login < :threshold"
);

int updatedCount = mutation
    .setParameter("newStatus", "INACTIVE")
    .setParameter("oldStatus", "ACTIVE")
    .setParameter("threshold", LocalDate.now().minusMonths(12).atStartOfDay())
    .executeUpdate();

System.out.println("已更新 " + updatedCount + " 条用户记录");

// 原生批量 DELETE
int deletedCount = session.createNativeMutationQuery(
    "DELETE FROM t_audit_logs WHERE created_at < :threshold AND archived = true"
)
.setParameter("threshold", LocalDate.now().minusYears(3).atStartOfDay())
.executeUpdate();

System.out.println("已删除 " + deletedCount + " 条审计日志");

// 条件批量操作
int resetCount = session.createNativeMutationQuery(
    "UPDATE t_employees SET bonus_pool = bonus_pool * :multiplier "
    + "WHERE dept_id = :deptId AND performance_rating >= :minRating"
)
.setParameter("multiplier", new BigDecimal("1.15"))
.setParameter("deptId", 100L)
.setParameter("minRating", 4)
.executeUpdate();
```

### 7.1.2 批量操作后处理缓存

```java
// 批量 UPDATE/DELETE 绕过一级缓存，必须手动处理一致性

@Transactional
public void batchUpdateUserStatus(List<Long> userIds, String newStatus) {
    // 1. 从一级缓存中清除即将更新的实体
    for (Long userId : userIds) {
        User cached = session.get(User.class, userId);
        if (cached != null) {
            session.evict(cached);  // 移除缓存中的实体
        }
    }

    // 2. 执行批量更新
    int count = session.createNativeMutationQuery(
        "UPDATE t_users SET status = :newStatus WHERE id IN (:ids)"
    )
    .setParameter("newStatus", newStatus)
    .setParameterList("ids", userIds)
    .executeUpdate();

    // 3. 如果后续需要操作这些实体，建议清除整个一级缓存
    session.clear();

    System.out.printf("批量更新完成: %d/%d 条记录%n", count, userIds.size());
}

// 批量操作最佳实践：
// 1. 批量操作前：evict() 或 clear() 一级缓存中的关联实体
// 2. 批量操作后：如果不再需要继续操作实体，clear() 一级缓存
// 3. 大量数据分批执行：每 50~100 条 flush() + clear() 一次
// 4. 批量操作放在独立事务中，避免长事务
```

## 7.2 批量插入

### 7.2.1 JDBC Batch + 原生 SQL

Hibernate 的 `hibernate.jdbc.batch_size` 配置只对通过实体持久化的 INSERT 生效，对原生 SQL `INSERT` 不生效。需要通过 JDBC 批处理：

```java
// 使用 JDBC Batch 批量插入
@Transactional
public void batchInsertAuditLogs(List<AuditLog> logs) {
    session.doWork(connection -> {
        String sql = "INSERT INTO t_audit_logs (user_id, action, detail, ip_address, created_at) "
            + "VALUES (?, ?, ?, ?, ?)";

        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            int batchCount = 0;

            for (AuditLog log : logs) {
                ps.setLong(1, log.getUserId());
                ps.setString(2, log.getAction());
                ps.setString(3, log.getDetail());
                ps.setString(4, log.getIpAddress());
                ps.setTimestamp(5, Timestamp.valueOf(log.getCreatedAt()));

                ps.addBatch();
                batchCount++;

                // 分批提交，避免单批过大导致 OOM
                if (batchCount % 500 == 0) {
                    int[] results = ps.executeBatch();
                    System.out.printf("已插入 %d 条记录%n", batchCount);
                }
            }

            // 提交最后一批
            if (batchCount % 500 != 0) {
                int[] results = ps.executeBatch();
                System.out.printf("最后一批已插入，总计 %d 条记录%n", batchCount);
            }
        }
    });
}
```

### 7.2.2 数据库特定的 UPSERT 批量操作

```java
// MySQL：INSERT ... ON DUPLICATE KEY UPDATE
public void batchUpsertProductsMySQL(List<Product> products) {
    session.doWork(connection -> {
        String sql = "INSERT INTO t_products (sku, name, price, stock, updated_at) "
            + "VALUES (?, ?, ?, ?, NOW()) "
            + "ON DUPLICATE KEY UPDATE "
            + "name = VALUES(name), price = VALUES(price), "
            + "stock = stock + VALUES(stock), updated_at = NOW()";

        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            for (Product product : products) {
                ps.setString(1, product.getSku());
                ps.setString(2, product.getName());
                ps.setBigDecimal(3, product.getPrice());
                ps.setInt(4, product.getStock());
                ps.addBatch();
            }
            ps.executeBatch();
        }
    });
}

// PostgreSQL：INSERT ... ON CONFLICT
public void batchUpsertProductsPostgreSQL(List<Product> products) {
    session.doWork(connection -> {
        String sql = "INSERT INTO t_products (sku, name, price, stock, updated_at) "
            + "VALUES (?, ?, ?, ?, NOW()) "
            + "ON CONFLICT (sku) DO UPDATE SET "
            + "name = EXCLUDED.name, price = EXCLUDED.price, "
            + "stock = t_products.stock + EXCLUDED.stock, updated_at = NOW()";

        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            for (Product product : products) {
                ps.setString(1, product.getSku());
                ps.setString(2, product.getName());
                ps.setBigDecimal(3, product.getPrice());
                ps.setInt(4, product.getStock());
                ps.addBatch();
            }
            ps.executeBatch();
        }
    });
}
```

### 7.2.3 批量插入性能优化建议

```java
// 性能对比（插入 10,000 条记录）：
// 逐条 session.persist() + hibernate.jdbc.batch_size=50    ≈ 2-5 秒
// JDBC Batch（直接 PreparedStatement.addBatch）            ≈ 0.5-1 秒
// JDBC Batch + rewriteBatchedStatements=true（MySQL）       ≈ 0.2-0.5 秒

// 优化总结：
// 1. 使用 JDBC Batch 而非 Hibernate 的 session.persist 循环
// 2. MySQL 添加 jdbc URL 参数: rewriteBatchedStatements=true
// 3. PostgreSQL 添加 jdbc URL 参数: reWriteBatchedInserts=true
// 4. 每 500-1000 条记录执行一次 executeBatch + clear
// 5. 在独立事务中执行批量操作
// 6. 关闭二级缓存
```

---

# 8. 数据库特定功能

## 8.1 PostgreSQL 特性

### 8.1.1 JSONB 查询

PostgreSQL JSONB 类型支持高效的索引和查询操作符：

```sql
-- 建表
-- CREATE TABLE events (
--     id BIGSERIAL PRIMARY KEY,
--     event_type VARCHAR(50),
--     payload JSONB,
--     created_at TIMESTAMPTZ DEFAULT NOW()
-- );
-- CREATE INDEX idx_events_payload ON events USING GIN (payload);
```

```java
// 1. JSONB 字段查询：@> 包含操作符
String sql1 = "SELECT id, event_type, payload, created_at "
    + "FROM events "
    + "WHERE payload @> :conditions "
    + "ORDER BY created_at DESC LIMIT :limit";

String conditions = "{\"user_id\": 1001, \"action\": \"login\"}";
List<Object[]> events = session.createNativeQuery(sql1, Object[].class)
    .setParameter("conditions", conditions)
    .setParameter("limit", 100)
    .getResultList();

// 2. JSONB 路径提取：->> 文本提取
String sql2 = "SELECT "
    + "payload->>'user_id' AS user_id, "
    + "payload->>'action' AS action, "
    + "payload->>'ip_address' AS ip, "
    + "COUNT(*) AS event_count "
    + "FROM events "
    + "WHERE created_at BETWEEN :startDate AND :endDate "
    + "GROUP BY payload->>'user_id', payload->>'action', payload->>'ip_address' "
    + "HAVING COUNT(*) > :threshold";

// 3. JSONB 数组查询：jsonb_array_elements
String sql3 = "SELECT "
    + "e.id, "
    + "items.item->>'product_id' AS product_id, "
    + "items.item->>'quantity' AS quantity "
    + "FROM t_orders e, "
    + "jsonb_array_elements(e.order_data->'items') AS items(item) "
    + "WHERE e.order_data->>'status' = 'PROCESSING'";
```

### 8.1.2 全文搜索

PostgreSQL 全文搜索：tsvector（文本向量）、tsquery（查询向量）：

```sql
-- 建表和索引
-- CREATE TABLE articles (
--     id BIGSERIAL PRIMARY KEY,
--     title VARCHAR(500),
--     content TEXT,
--     search_vector TSVECTOR,
--     published_at TIMESTAMPTZ
-- );
-- CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);
```

```java
// 1. 基本全文搜索 + 相关性排序
String searchSql = "SELECT "
    + "a.id, a.title, a.published_at, "
    + "ts_rank(a.search_vector, query) AS rank "
    + "FROM articles a, plainto_tsquery('english', :keyword) AS query "
    + "WHERE a.search_vector @@ query "
    + "ORDER BY rank DESC LIMIT :limit";

// 2. 搜索结果高亮
String highlightSql = "SELECT "
    + "a.id, a.title, "
    + "ts_headline('english', a.content, "
    + "    plainto_tsquery('english', :keyword), "
    + "    'MaxWords=50, MinWords=20, StartSel=<mark>, StopSel=</mark>') AS headline, "
    + "ts_rank(a.search_vector, plainto_tsquery('english', :keyword)) AS rank "
    + "FROM articles a "
    + "WHERE a.search_vector @@ plainto_tsquery('english', :keyword) "
    + "ORDER BY rank DESC LIMIT :limit";

// 3. 高级搜索（支持逻辑操作符 &、|、!）
String advancedSearch = "SELECT id, title, ts_rank(search_vector, query) AS rank "
    + "FROM articles, to_tsquery('english', :queryText) AS query "
    + "WHERE search_vector @@ query "
    + "ORDER BY rank DESC LIMIT :limit";
// :queryText 示例: "java & hibernate & !eclipselink"
```

### 8.1.3 窗口函数

```java
// 1. 每个部门内的工资排名
String rankSql = "SELECT "
    + "e.id, e.name, e.salary, d.name AS dept_name, "
    + "RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS salary_rank, "
    + "DENSE_RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dense_rank, "
    + "NTILE(4) OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS quartile "
    + "FROM t_employees e "
    + "JOIN t_departments d ON e.dept_id = d.id "
    + "ORDER BY d.name, salary_rank";

// 2. 月度销售额累计 + 移动平均
String cumulativeSql = "SELECT "
    + "DATE_TRUNC('month', order_date) AS month, "
    + "SUM(total_amount) AS monthly_sales, "
    + "SUM(SUM(total_amount)) OVER (ORDER BY DATE_TRUNC('month', order_date)) "
    + "    AS cumulative_sales, "
    + "ROUND(AVG(SUM(total_amount)) OVER (ORDER BY DATE_TRUNC('month', order_date) "
    + "    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3month "
    + "FROM t_orders "
    + "WHERE order_date >= :startDate "
    + "GROUP BY DATE_TRUNC('month', order_date) "
    + "ORDER BY month";

// 3. LAG/LEAD：与前后行比较
String lagLeadSql = "SELECT "
    + "DATE_TRUNC('day', order_date) AS day, "
    + "COUNT(*) AS order_count, "
    + "LAG(COUNT(*), 1) OVER (ORDER BY DATE_TRUNC('day', order_date)) AS prev_day_count, "
    + "LEAD(COUNT(*), 1) OVER (ORDER BY DATE_TRUNC('day', order_date)) AS next_day_count, "
    + "ROUND((COUNT(*) - LAG(COUNT(*), 1) OVER (ORDER BY DATE_TRUNC('day', order_date))) "
    + "    * 100.0 / NULLIF(LAG(COUNT(*), 1) OVER (ORDER BY DATE_TRUNC('day', order_date)), 0), 2) "
    + "    AS growth_pct "
    + "FROM t_orders "
    + "WHERE order_date >= :startDate "
    + "GROUP BY DATE_TRUNC('day', order_date) "
    + "ORDER BY day";
```

### 8.1.4 CTE（WITH RECURSIVE）

```java
// 1. 组织树（部门层级递归）
String orgTreeSql = "WITH RECURSIVE dept_tree AS ( "
    + "    SELECT id, name, parent_id, 1 AS level, name AS path "
    + "    FROM t_departments "
    + "    WHERE parent_id IS NULL "
    + "    UNION ALL "
    + "    SELECT d.id, d.name, d.parent_id, t.level + 1, "
    + "        t.path || ' > ' || d.name "
    + "    FROM t_departments d "
    + "    INNER JOIN dept_tree t ON d.parent_id = t.id "
    + "    WHERE t.level < :maxLevel "
    + ") "
    + "SELECT id, name, parent_id, level, path FROM dept_tree ORDER BY path";

// 2. 物料清单（BOM）展开
String bomSql = "WITH RECURSIVE bom_tree AS ( "
    + "    SELECT product_id, component_id, quantity, 1 AS level, "
    + "        CAST(component_id AS VARCHAR(500)) AS path "
    + "    FROM t_bom WHERE product_id = :productId "
    + "    UNION ALL "
    + "    SELECT b.product_id, b.component_id, b.quantity * bt.quantity, "
    + "        bt.level + 1, bt.path || ',' || b.component_id "
    + "    FROM t_bom b "
    + "    INNER JOIN bom_tree bt ON b.product_id = bt.component_id "
    + "    WHERE bt.level < :maxLevel "
    + "        AND bt.path NOT LIKE '%' || b.component_id || '%' "
    + ") "
    + "SELECT product_id, component_id, quantity, level "
    + "FROM bom_tree ORDER BY level, product_id";

// 3. 日期序列生成（补全缺失日期）
String dateSeriesSql = "WITH RECURSIVE date_series AS ( "
    + "    SELECT :startDate::date AS report_date "
    + "    UNION ALL "
    + "    SELECT report_date + 1 FROM date_series "
    + "    WHERE report_date < :endDate::date "
    + ") "
    + "SELECT ds.report_date, COALESCE(o.order_count, 0) AS order_count, "
    + "    COALESCE(o.total_amount, 0) AS total_amount "
    + "FROM date_series ds "
    + "LEFT JOIN ( "
    + "    SELECT DATE(order_date) AS order_date, COUNT(*) AS order_count, "
    + "        SUM(total_amount) AS total_amount "
    + "    FROM t_orders "
    + "    WHERE order_date BETWEEN :startDate AND :endDate "
    + "    GROUP BY DATE(order_date) "
    + ") o ON ds.report_date = o.order_date "
    + "ORDER BY ds.report_date";
```

### 8.1.5 地理信息（PostGIS）

```java
// PostGIS 地理空间查询：WGS84 坐标系 (SRID=4326)

// 建表和索引
// CREATE TABLE stores (
//     id BIGSERIAL PRIMARY KEY, name VARCHAR(200), address VARCHAR(500),
//     location GEOMETRY(Point, 4326)
// );
// CREATE INDEX idx_stores_location ON stores USING GIST (location);

// 1. 按距离查找附近门店
String nearbySql = "SELECT "
    + "s.id, s.name, s.address, "
    + "ST_Distance(s.location::geography, "
    + "    ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography) AS distance_meters "
    + "FROM stores s "
    + "WHERE ST_DWithin(s.location::geography, "
    + "    ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography, :radiusMeters) "
    + "ORDER BY distance_meters";

// 2. 范围查询（判断点是否在配送区域内）
String containsSql = "SELECT id, name "
    + "FROM delivery_zones "
    + "WHERE ST_Contains(zone_polygon, "
    + "    ST_SetSRID(ST_MakePoint(:lng, :lat), 4326))";

// 3. 地理聚类分析
String clusterSql = "SELECT "
    + "ST_ClusterKMeans(location::geometry, :numClusters) OVER () AS cluster_id, "
    + "COUNT(*) AS customer_count, "
    + "ST_Centroid(ST_Collect(location::geometry)) AS cluster_center "
    + "FROM t_customers "
    + "WHERE location IS NOT NULL "
    + "GROUP BY cluster_id";
```

## 8.2 MySQL 特性

### 8.2.1 全文搜索

MySQL FULLTEXT 索引 + MATCH ... AGAINST 语法：

```sql
-- 建表和索引
-- CREATE TABLE articles (
--     id BIGINT AUTO_INCREMENT PRIMARY KEY,
--     title VARCHAR(500), content TEXT,
--     FULLTEXT INDEX ft_articles (title, content)
-- ) ENGINE=InnoDB;
```

```java
// 1. 自然语言搜索
String naturalLangSql = "SELECT "
    + "a.id, a.title, "
    + "MATCH(a.title, a.content) AGAINST (:keyword IN NATURAL LANGUAGE MODE) AS relevance "
    + "FROM articles a "
    + "WHERE MATCH(a.title, a.content) AGAINST (:keyword IN NATURAL LANGUAGE MODE) "
    + "ORDER BY relevance DESC LIMIT :limit";

// 2. 布尔模式搜索（支持 +、-、"..."、* 操作符）
String booleanSql = "SELECT id, title, content "
    + "FROM articles "
    + "WHERE MATCH(title, content) AGAINST (:keywords IN BOOLEAN MODE)";
// :keywords 示例: "+Java +Hibernate -EclipseLink"

// 3. 查询扩展模式
String expansionSql = "SELECT id, title "
    + "FROM articles "
    + "WHERE MATCH(title, content) AGAINST (:keyword WITH QUERY EXPANSION) LIMIT 20";
```

### 8.2.2 JSON 函数（MySQL 5.7+/8.0）

```java
// 1. JSON_EXTRACT 提取值 + JSON_UNQUOTE 去掉引号
String jsonExtractSql = "SELECT "
    + "id, "
    + "JSON_EXTRACT(attributes, '$.color') AS color, "
    + "JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.brand')) AS brand "
    + "FROM t_products "
    + "WHERE JSON_CONTAINS(attributes->'$.tags', '\"premium\"')";

// 2. JSON_SEARCH 搜索
String jsonSearchSql = "SELECT id, name, attributes "
    + "FROM t_products "
    + "WHERE JSON_EXTRACT(attributes, '$.category') = :category "
    + "AND JSON_SEARCH(attributes, 'one', :tagName, NULL, '$.tags[*]') IS NOT NULL";

// 3. JSON_TABLE：将 JSON 数组转换为关系表（MySQL 8.0.4+）
String jsonTableSql = "SELECT "
    + "o.id AS order_id, o.customer_name, "
    + "jt.product_id, jt.product_name, jt.quantity, jt.unit_price "
    + "FROM t_orders o, "
    + "JSON_TABLE(o.order_items, '$.items[*]' COLUMNS ( "
    + "    product_id BIGINT PATH '$.product_id', "
    + "    product_name VARCHAR(200) PATH '$.product_name', "
    + "    quantity INT PATH '$.quantity', "
    + "    unit_price DECIMAL(10,2) PATH '$.unit_price' "
    + ")) AS jt "
    + "WHERE o.status = 'CONFIRMED'";
```

### 8.2.3 窗口函数（MySQL 8+）

```java
// 1. 用户订单排名 + 累计值
String rankingSql = "SELECT "
    + "customer_id, order_id, order_date, total_amount, "
    + "ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn, "
    + "SUM(total_amount) OVER (PARTITION BY customer_id) AS lifetime_value "
    + "FROM t_orders WHERE order_date >= :startDate";

// 2. 百分比排名
String percentRankSql = "SELECT "
    + "product_id, product_name, sales_quantity, "
    + "ROUND(PERCENT_RANK() OVER (ORDER BY sales_quantity DESC), 4) AS percentile_rank "
    + "FROM t_product_sales WHERE year = :year";

// 3. 移动平均（7天 / 30天）
String movingAvgSql = "SELECT "
    + "order_date, total_amount, "
    + "ROUND(AVG(total_amount) OVER (ORDER BY order_date "
    + "    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS avg_7day "
    + "FROM ("
    + "    SELECT DATE(order_date) AS order_date, SUM(total_amount) AS total_amount "
    + "    FROM t_orders GROUP BY DATE(order_date)"
    + ") t ORDER BY order_date";
```

### 8.2.4 GROUP_CONCAT

```java
// GROUP_CONCAT：将分组内的值拼接为字符串（仅 MySQL / SQLite）

String groupConcatSql = "SELECT "
    + "customer_id, "
    + "GROUP_CONCAT(DISTINCT product_name ORDER BY product_name SEPARATOR ', ') "
    + "    AS products "
    + "FROM t_orders o "
    + "JOIN t_order_items oi ON o.id = oi.order_id "
    + "WHERE o.order_date >= :startDate "
    + "GROUP BY customer_id";

// 完整的部门员工汇总
String summarySql = "SELECT "
    + "d.name AS dept_name, "
    + "COUNT(e.id) AS employee_count, "
    + "GROUP_CONCAT(e.name ORDER BY e.salary DESC SEPARATOR ', ') AS employee_names, "
    + "ROUND(AVG(e.salary), 2) AS avg_salary, "
    + "SUM(e.salary) AS total_salary "
    + "FROM t_departments d "
    + "LEFT JOIN t_employees e ON d.id = e.dept_id "
    + "GROUP BY d.id, d.name "
    + "ORDER BY total_salary DESC";

// 注意：GROUP_CONCAT 有最大长度限制（默认 1024 字节）
// SET SESSION group_concat_max_len = 1000000;
```

## 8.3 Oracle 特性

### 8.3.1 CONNECT BY（层次查询）

```java
// Oracle CONNECT BY：层次结构查询（MySQL/PostgreSQL 无此语法，需用 CTE 替代）

// 1. 部门组织树（从上到下）
String orgTreeSql = "SELECT "
    + "id, name, parent_id, LEVEL, "
    + "SYS_CONNECT_BY_PATH(name, ' > ') AS path "
    + "FROM t_departments "
    + "START WITH parent_id IS NULL "
    + "CONNECT BY PRIOR id = parent_id "
    + "ORDER SIBLINGS BY name";

// 2. 员工管理层级（从下到上：找到某个员工的所有上级）
String hierarchySql = "SELECT "
    + "e.id, e.name, e.manager_id, LEVEL AS management_level, "
    + "SYS_CONNECT_BY_PATH(e.name, ' -> ') AS reporting_line "
    + "FROM t_employees e "
    + "START WITH e.id = :employeeId "
    + "CONNECT BY PRIOR e.manager_id = e.id";
```

### 8.3.2 分析函数（LEAD、LAG、RANK 等）

```java
// 1. LEAD/LAG
String leadLagSql = "SELECT "
    + "employee_id, hire_date, salary, "
    + "LAG(salary, 1, 0) OVER (ORDER BY hire_date) AS prev_hire_salary, "
    + "LEAD(salary, 1, 0) OVER (ORDER BY hire_date) AS next_hire_salary "
    + "FROM t_employees ORDER BY hire_date";

// 2. RANK/DENSE_RANK/ROW_NUMBER
String rankSql = "SELECT "
    + "dept_id, employee_id, name, salary, "
    + "RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS salary_rank, "
    + "DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rank, "
    + "ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num "
    + "FROM t_employees";

// 3. FIRST_VALUE / LAST_VALUE
String firstLastSql = "SELECT "
    + "employee_id, name, dept_id, salary, "
    + "FIRST_VALUE(name) OVER (PARTITION BY dept_id ORDER BY salary DESC) AS highest_paid, "
    + "LAST_VALUE(name) OVER (PARTITION BY dept_id ORDER BY salary DESC "
    + "    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS lowest_paid "
    + "FROM t_employees";
```

### 8.3.3 MERGE 操作

```java
// Oracle MERGE INTO：根据条件插入或更新

// 1. 同步外部数据到主表
String mergeSql = "MERGE INTO t_products p "
    + "USING (SELECT :sku AS sku, :name AS name, :price AS price, :stock AS stock FROM dual) s "
    + "ON (p.sku = s.sku) "
    + "WHEN MATCHED THEN "
    + "    UPDATE SET p.name = s.name, p.price = s.price, "
    + "        p.stock = p.stock + s.stock, p.updated_at = SYSTIMESTAMP "
    + "WHEN NOT MATCHED THEN "
    + "    INSERT (sku, name, price, stock, created_at, updated_at) "
    + "    VALUES (s.sku, s.name, s.price, s.stock, SYSTIMESTAMP, SYSTIMESTAMP)";

int affected = session.createNativeMutationQuery(mergeSql)
    .setParameter("sku", "PROD-001")
    .setParameter("name", "New Product")
    .setParameter("price", new BigDecimal("199.99"))
    .setParameter("stock", 100)
    .executeUpdate();
```

---

# 9. 常见问题

## 9.1 原生查询与缓存的一致性问题

### 问题一：一级缓存脏数据

```java
// 场景：原生 UPDATE 后，一级缓存中的实体过期

User user = session.get(User.class, 1L);
System.out.println("修改前: " + user.getEmail());  // old@example.com

// 通过原生 SQL 修改
session.createNativeMutationQuery(
    "UPDATE t_users SET email_address = 'new@example.com' WHERE id = :id"
).setParameter("id", 1L).executeUpdate();

// 再次访问同一实体：返回一级缓存的旧对象
User sameUser = session.get(User.class, 1L);
System.out.println("修改后: " + sameUser.getEmail());  // 仍是 old@example.com

// 解决方案：
// 1. 操作前清除
session.evict(user);
session.createNativeMutationQuery("UPDATE t_users SET ...").executeUpdate();
User fresh = session.get(User.class, 1L);  // 从数据库重新加载

// 2. 操作后刷新
session.createNativeMutationQuery("UPDATE ...").executeUpdate();
session.refresh(user);

// 3. 操作后清除所有
session.createNativeMutationQuery("UPDATE ...").executeUpdate();
session.clear();
```

### 问题二：二级缓存不更新

```java
// 原生 SQL 操作不会更新二级缓存

@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@Entity
public class Product { ... }

// 原生 UPDATE 后，二级缓存仍持有旧数据
session.createNativeMutationQuery(
    "UPDATE t_products SET price = :newPrice WHERE id = :id"
).setParameter("newPrice", new BigDecimal("129.99")).setParameter("id", 100L).executeUpdate();

// 解决方案：使用 sessionFactory.getCache().evict() 清除二级缓存
sessionFactory.getCache().evictEntityData(Product.class, 100L);    // 清除单个
sessionFactory.getCache().evictEntityData(Product.class);           // 清除全部 Product
sessionFactory.getCache().evictAllRegions();                         // 清除全部缓存
```

## 9.2 SQL 注入防范

```java
// ========== SQL 注入防范清单 ==========

// 1. 始终使用参数绑定
//    session.createNativeQuery("... WHERE name = :name").setParameter("name", input)
//    session.createNativeQuery("... WHERE name = '" + input + "'")  // 禁止！

// 2. 动态表名/列名/排序字段必须白名单验证
private static final Set<String> ALLOWED_COLUMNS =
    Set.of("id", "username", "email", "created_at");

public List<User> getUsersByOrder(String orderBy, String direction) {
    if (!ALLOWED_COLUMNS.contains(orderBy)) {
        throw new IllegalArgumentException("Invalid column: " + orderBy);
    }
    if (!"ASC".equalsIgnoreCase(direction) && !"DESC".equalsIgnoreCase(direction)) {
        throw new IllegalArgumentException("Invalid direction: " + direction);
    }
    return session.createNativeQuery(
        "SELECT * FROM t_users ORDER BY " + orderBy + " " + direction, User.class
    ).getResultList();
}

// 3. LIMIT / OFFSET 值需要范围校验
public List<User> getPagedUsers(int page, int size) {
    if (page < 1) page = 1;
    if (size < 1 || size > 100) size = 20;
    int offset = (page - 1) * size;
    return session.createNativeQuery(
        "SELECT * FROM t_users ORDER BY id LIMIT :limit OFFSET :offset", User.class
    )
    .setParameter("limit", size)
    .setParameter("offset", offset)
    .getResultList();
}

// 4. LIKE 查询需要对 % 和 _ 进行转义
public List<User> searchByUsername(String keyword) {
    String escaped = keyword.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_");
    return session.createNativeQuery(
        "SELECT * FROM t_users WHERE user_name LIKE :keyword ESCAPE '\\'", User.class
    )
    .setParameter("keyword", "%" + escaped + "%")
    .getResultList();
}
```

## 9.3 结果集映射失败排查

### 常见问题与排查步骤

```java
// === 问题 1：列名不匹配 ===
// 症状：查询执行成功但实体属性全部为 null 或默认值
// 原因：SQL 列名与 @Column(name) / 属性名不匹配

// 排查：启用 SQL 日志
// hibernate.show_sql=true
// logging.level.org.hibernate.SQL=DEBUG
// logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

// === 问题 2：类型无法转换 ===
// 症状：ClassCastException 或 NumberFormatException
// 原因：数据库列类型与 Java 属性类型不匹配

// 排查：检查 Object[] 中每个元素的运行时类型
List<Object[]> rows = session.createNativeQuery(
    "SELECT id, username, created_at FROM t_users", Object[].class
).getResultList();
for (Object[] row : rows) {
    System.out.println("id class: " + row[0].getClass().getName());
    System.out.println("username class: " + row[1].getClass().getName());
    System.out.println("created_at class: " + row[2].getClass().getName());
}

// === 问题 3：@ConstructorResult 参数不匹配 ===
// 症状：MappingException 或查询结果全部为 null
// 原因：DTO 构造方法参数与 @ColumnResult 顺序/类型不一致

// 排查：
// 1. 确认 @ColumnResult 的 name 与 SQL AS 别名一致
// 2. 确认构造方法参数顺序与 @ColumnResult 声明顺序一致
// 3. 确认构造方法参数类型与 @ColumnResult.type 一致

// === 问题 4：关联实体代理初始化失败 ===
// 症状：LazyInitializationException
// 原因：原生查询加载的实体没有初始化代理（proxy）

// 解决：
// 1. 在 Session 打开时访问惰性属性
// 2. 使用 @SqlResultSetMapping 一次性加载关联实体
// 3. Hibernate.initialize(entity.getAssociation())
```

### 调试辅助工具类

```java
public class NativeQueryDebugger {

    /**
     * 调试原生查询：打印列名和值的类型
     */
    public static void debugNativeQuery(Session session, String sql,
                                         Map<String, Object> params) {
        NativeQuery<Object[]> query = session.createNativeQuery(sql, Object[].class);
        if (params != null) {
            params.forEach(query::setParameter);
        }

        List<Object[]> results = query.getResultList();

        if (results.isEmpty()) {
            System.out.println("[NativeQueryDebugger] 查询无结果");
            return;
        }

        // 通过 JDBC 获取列元数据
        session.doWork(connection -> {
            try (PreparedStatement ps = connection.prepareStatement(sql)) {
                if (params != null) {
                    int idx = 1;
                    for (Object val : params.values()) {
                        ps.setObject(idx++, val);
                    }
                }
                try (ResultSet rs = ps.executeQuery()) {
                    ResultSetMetaData meta = rs.getMetaData();
                    int columnCount = meta.getColumnCount();

                    System.out.println("========== ResultSet 列信息 ==========");
                    for (int i = 1; i <= columnCount; i++) {
                        System.out.printf("列 %d: name=%s, label=%s, type=%s, className=%s%n",
                            i, meta.getColumnName(i), meta.getColumnLabel(i),
                            meta.getColumnTypeName(i), meta.getColumnClassName(i));
                    }

                    if (rs.next()) {
                        System.out.println("========== 第一行数据 ==========");
                        for (int i = 1; i <= columnCount; i++) {
                            Object value = rs.getObject(i);
                            System.out.printf("列 %d (%s): value=%s, type=%s%n",
                                i, meta.getColumnLabel(i), value,
                                value != null ? value.getClass().getName() : "null");
                        }
                    }
                }
            }
        });

        System.out.println("========== Hibernate 映射结果 ==========");
        System.out.println("结果行数: " + results.size());
        if (!results.isEmpty()) {
            Object[] firstRow = results.get(0);
            for (int i = 0; i < firstRow.length; i++) {
                System.out.printf("列 %d: value=%s, type=%s%n",
                    i, firstRow[i],
                    firstRow[i] != null ? firstRow[i].getClass().getName() : "null");
            }
        }
    }
}
```

## 9.4 分页数据库兼容性

```java
// 不同数据库分页语法完全不同，Hibernate 自动包装分页依赖 Dialect

// MySQL / PostgreSQL / SQLite / H2
// SELECT ... LIMIT :limit OFFSET :offset

// Oracle 11g（不支持 OFFSET FETCH）
// SELECT * FROM (
//     SELECT a.*, ROWNUM rnum FROM (
//         SELECT ... ORDER BY ...
//     ) a WHERE ROWNUM <= :endRow
// ) WHERE rnum > :startRow

// Oracle 12c+ / SQL Server 2012+
// SELECT ... OFFSET :offset ROWS FETCH NEXT :limit ROWS ONLY

// SQL Server 2008
// SELECT * FROM (
//     SELECT ROW_NUMBER() OVER (ORDER BY ...) AS row_num, *
//     FROM table_name
// ) AS t WHERE row_num BETWEEN :startRow AND :endRow

// 兼容性方案一：让 Hibernate 自动包装（推荐）
query.setFirstResult(0).setMaxResults(20);

// 方案二：复杂场景（UNION、CTE）使用窗口函数手写分页
String compatibleSql = "SELECT * FROM ( "
    + "SELECT t.*, ROW_NUMBER() OVER (ORDER BY created_at DESC) AS rn "
    + "FROM ("
    + "    SELECT id, name, created_at FROM t_users "
    + "    UNION ALL "
    + "    SELECT id, name, created_at FROM t_archived_users "
    + ") t"
    + ") subquery WHERE rn BETWEEN :startRow AND :endRow";
```

## 9.5 事务与原生查询

### 事务传播与隔离级别

```java
// 1. 原生查询在当前事务上下文中执行
//    Connection 由当前 Hibernate Session 的事务管理

@Transactional
public void mixedOperations() {
    // 实体操作
    User user = new User();
    user.setUsername("Alice");
    session.persist(user);

    // 原生查询能否看到上面的未提交变更？
    // 取决于 FlushMode：
    //   AUTO：原生 SQL 不触发 flush，看不到未提交变更
    //   ALWAYS：触发 flush，能看到
    //   COMMIT：提交时才 flush

    // 最佳实践：
    // - 混用实体操作和原生查询时，先手动 flush
    // - 或通过 @Transactional 控制边界
}

// 2. 存储过程的独立事务（自治事务）
// 数据库存储过程内部可能使用 PRAGMA AUTONOMOUS_TRANSACTION（Oracle）
// 或独立事务控制（PostgreSQL dblink），其提交不受 Hibernate 事务管理

// 3. 只读事务优化
@Transactional(readOnly = true)
public List<User> getReportData() {
    return session.createNativeQuery(
        "SELECT * FROM t_users WHERE status = :status", User.class
    )
    .setParameter("status", "ACTIVE")
    .setHint("org.hibernate.readOnly", true)
    .getResultList();
}

// 4. 事务超时控制
@Transactional(timeout = 30)  // 30秒
public void longRunningQuery() {
    session.createNativeQuery("SELECT * FROM huge_table")
        .setHint("jakarta.persistence.query.timeout", 25000)
        .getResultList();
}

// 5. 原生 SQL 在事务回滚时的行为
// - 原生 UPDATE/DELETE 会随 Hibernate 事务回滚（由 JDBC Connection 保证）
// - 但一级缓存不会自动回滚，需要手动清理
@Transactional
public void safeBatchUpdate() {
    try {
        session.createNativeMutationQuery("UPDATE ...").executeUpdate();
        session.clear();  // 清理一级缓存
        // 其他业务操作 ...
    } catch (Exception e) {
        session.clear();  // 异常时也清除缓存
        throw e;
    }
}
```

### 事务隔离级别与原生 SQL

```java
// 不同隔离级别下原生 SQL 的可见性差异

// READ_COMMITTED（默认，大多数数据库）
// - 每次查询看到已提交的数据
// - 同事务内多次查询可能看到不同数据（不可重复读）

// REPEATABLE_READ（MySQL InnoDB 默认）
// - 同事务内多次查询看到相同快照
// - 原生 UPDATE 后查询返回新值，但 SELECT 仍看到快照

// 原生查询与悲观锁
List<User> users = session.createNativeQuery(
    "SELECT * FROM t_users WHERE id = :id FOR UPDATE", User.class
)
.setParameter("id", 1001L)
.setHint("jakarta.persistence.lock.timeout", 5000)
.getResultList();
// FOR UPDATE 在提交/回滚前锁定行，其他事务无法修改
```

---

> **总结**：原生 SQL 是 Hibernate ORM 的重要补充，在复杂查询、存储过程调用、性能优化和数据库特性利用等场景中不可替代。使用时应遵循以下原则：
>
> 1. **能用 HQL/JPQL/Criteria 时优先使用**，仅在必要时用原生 SQL
> 2. **始终使用参数绑定**，绝不拼接用户输入到 SQL 字符串
> 3. **关注缓存一致性**，原生 DML 操作后及时清理 Session 缓存
> 4. **优先使用 `@SqlResultSetMapping`**，将映射集中管理并复用
> 5. **利用 `@NamedNativeQuery`** 将 SQL 与业务逻辑分离
> 6. **Hibernate 6 迁移时**，用 `TupleTransformer` 替代 `ResultTransformer`
> 7. **批量操作使用 JDBC Batch**，比 Hibernate 逐条 persist 效率高 5-10 倍
> 8. **数据库特定 SQL 需文档化标注**，明确移植影响范围
