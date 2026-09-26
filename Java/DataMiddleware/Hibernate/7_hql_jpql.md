# 1. HQL 与 JPQL 概述

## 1.1 什么是 HQL

HQL（Hibernate Query Language）是 Hibernate 提供的**面向对象查询语言**。与 SQL 操作表名和列名不同，HQL 操作的是**实体类名和实体属性名**，语法与 SQL 类似但完全工作在对象模型层面。

核心特征：
- **面向对象**：查询基于实体名称和属性名称，而非数据库表名和列名
- **类 SQL 语法**：关键字风格类似 SQL，降低学习成本
- **数据库无关**：Hibernate 将 HQL 翻译为对应方言的 SQL，屏蔽数据库差异
- **完全支持多态**：查询父类时自动包含子类实例
- **实体概念完整**：支持关联导航、Fetch 策略、继承策略等 ORM 概念

```hql
-- SQL：操作表名和列名
SELECT u.username, u.age, a.city
FROM t_user u
JOIN t_address a ON u.address_id = a.id
WHERE u.age > 18;

-- HQL：操作实体名和属性名
SELECT u.username, u.age, u.address.city
FROM User u
WHERE u.age > 18;
```

与 SQL 的核心区别：

| 维度 | SQL | HQL |
|------|-----|-----|
| 查询目标 | 表名、列名 | 实体名、属性名 |
| 关系表示 | 外键 JOIN | 关联属性导航 / JOIN |
| 结果形式 | 标量值列表 | 实体对象 / Object[] / DTO |
| 数据库依赖 | 强依赖 | 通过 Dialect 屏蔽 |
| 继承 | 不感知 | 自动包含子类 |
| 关联懒加载 | 无法控制 | FETCH JOIN 控制抓取 |

HQL 是 **JPQL 的超集**，即 HQL 包含了 JPQL 全部功能并增加了额外的扩展语法（如 INSERT ... SELECT、RIGHT JOIN、FROM 子查询、WITH 子句等）。

## 1.2 什么是 JPQL

JPQL（Java/Jakarta Persistence Query Language）是 **JPA 规范定义的查询语言**，与 HQL 设计理念相同，但提供了有限的查询能力以保持实现无关性。

```java
// 标准 JPQL 用法
TypedQuery<User> query = entityManager.createQuery(
    "SELECT u FROM User u WHERE u.age > :minAge", User.class
);
List<User> users = query.setParameter("minAge", 18).getResultList();
```

JPQL 与 HQL 的兼容关系：
- JPQL 查询可以直接在 Hibernate 中运行（Hibernate 完整实现了 JPQL 规范）
- HQL 查询**不能保证**在其他 JPA 实现中运行（如 EclipseLink）
- 若需要 JPA 厂商无关性，应该使用 JPQL + Criteria API

JPQL 的局限性（HQL 具备但 JPQL 不支持）：
- **INSERT ... SELECT**：仅 HQL 支持
- **RIGHT OUTER JOIN**：JPQL 仅支持 LEFT OUTER JOIN
- **FROM 子句中的子查询**：Hibernate 6+ 支持
- **WITH 子句**：HQL 可附加 JOIN 条件
- **日期/时间算数**：HQL 内置 date_add、date_sub 等扩展函数
- **类型扩展函数**：HQL 提供更多内置函数（100+）

HQL 额外保留字（JPQL 无）：
```
BETWEEN, CASE, ELSE, END, ESCAPE, FULL, INNER, KEY, LEFT, LOAD, REPLACE, RIGHT, 
TRAILING, UNION, WHEN, WITH
```

## 1.3 查询 API 核心接口

Hibernate 6 的查询 API 分为 JPA 标准接口和 Hibernate 扩展接口两个层级：

**JPA 标准接口：**

```java
// jakarta.persistence.Query — 无类型查询
Query query = entityManager.createQuery("SELECT u.name, u.age FROM User u");
List<Object[]> resultList = query.getResultList();

// jakarta.persistence.TypedQuery<X> — 泛型类型安全查询
TypedQuery<User> typedQuery = entityManager.createQuery(
    "SELECT u FROM User u WHERE u.age > 18", User.class
);
List<User> users = typedQuery.getResultList();
```

**Hibernate 扩展接口（org.hibernate.query）：**

```java
// org.hibernate.query.Query<R> — 扩展 JPA，提供 Hibernate 专有方法
// 继承自 jakarta.persistence.TypedQuery<R>
org.hibernate.query.Query<User> hibernateQuery = session.createQuery(
    "FROM User u", User.class
);

// org.hibernate.query.SelectionQuery<R> — 只读选择查询（Hibernate 6+）
SelectionQuery<User> selQuery = session.createSelectionQuery(
    "FROM User u WHERE u.status = :status", User.class
);
List<User> list = selQuery
    .setParameter("status", Status.ACTIVE)
    .setTimeout(30)
    .getResultList();

// org.hibernate.query.MutationQuery — 批量操作（Hibernate 6+）
MutationQuery mutationQuery = session.createMutationQuery(
    "UPDATE User u SET u.status = :newStatus WHERE u.lastLogin < :date"
);
int rows = mutationQuery
    .setParameter("newStatus", Status.INACTIVE)
    .setParameter("date", LocalDate.now().minusYears(1))
    .executeUpdate();
```

**Query 接口常用方法详解：**

| 方法 | 所属接口 | 说明 |
|------|---------|------|
| `getResultList()` | JPA Query | 返回结果列表，无结果则空列表 |
| `getSingleResult()` | JPA Query | 返回单个结果，无结果抛 NoResultException，多条抛 NonUniqueResultException |
| `getSingleResultOrNull()` | Hibernate Query | 返回单个结果或 null（Hibernate 6 新增）|
| `executeUpdate()` | JPA Query | 执行 UPDATE/DELETE，返回受影响行数 |
| `setParameter(name, value)` | JPA Query | 绑定命名参数 |
| `setParameter(position, value)` | JPA Query | 绑定位置参数 |
| `setMaxResults(int)` | JPA Query | 设置最大返回行数 |
| `setFirstResult(int)` | JPA Query | 设置起始偏移 |
| `setHint(name, value)` | JPA Query | 设置查询 Hint |
| `setFlushMode(FlushModeType)` | JPA Query | 设置刷新模式 |
| `setLockMode(LockModeType)` | JPA Query | 设置悲观锁 |
| `stream()` | JPA Query | 返回 Java 8 Stream（Hibernate 5.2+）|
| `getResultStream()` | JPA Query | 返回 Stream，自动关闭 Session 时不报错 |

# 2. 基本查询语法

## 2.1 SELECT 子句

完整 HQL SELECT 语法：

```hql
SELECT [DISTINCT] <select_expression>
FROM <entity_name> [[AS] <identification_variable>]
    [, <entity_name> [[AS] <identification_variable>]]*
[WHERE <conditional_expression>]
[GROUP BY <group_by_expression>]
[HAVING <conditional_expression>]
[ORDER BY <order_by_expression>]
```

### 查询实体

三种等价写法，效果一致：

```java
// 写法一：省略 SELECT（HQL 特有，JPQL 必须带 SELECT）
List<User> users1 = session.createQuery("FROM User", User.class).list();

// 写法二：完整 SELECT + 别名
List<User> users2 = session.createQuery("SELECT u FROM User u", User.class).list();

// 写法三：多态查询（查询父类，返回包含所有子类实例）
List<BaseEntity> entities = session.createQuery("FROM BaseEntity", BaseEntity.class).list();
```

### 查询属性（投影）

```java
// 单个属性 → 返回属性类型
TypedQuery<String> nameQuery = session.createQuery(
    "SELECT u.username FROM User u", String.class
);
List<String> names = nameQuery.getResultList();

// 多个属性 → 返回 Object[]
List<Object[]> rows = session.createQuery(
    "SELECT u.username, u.age, u.email FROM User u"
).getResultList();
for (Object[] row : rows) {
    String username = (String) row[0];
    Integer age = (Integer) row[1];
    String email = (String) row[2];
}

// DISTINCT 去重
List<String> distinctNames = session.createQuery(
    "SELECT DISTINCT u.username FROM User u", String.class
).getResultList();
```

### SELECT NEW 构造器表达式（DTO 投影）

```java
// 定义 DTO
public record UserDTO(String username, Integer age, String city) {
    public UserDTO(String username, Integer age, String city) {
        this.username = username;
        this.age = age;
        this.city = city;
    }
}

// 使用 SELECT NEW 创建 DTO
List<UserDTO> dtos = session.createQuery(
    "SELECT NEW com.example.dto.UserDTO(u.username, u.age, u.address.city) " +
    "FROM User u WHERE u.age > :minAge",
    UserDTO.class
).setParameter("minAge", 18).getResultList();
```

注意事项：
- 构造器表达式必须使用 **完全限定类名**
- 构造器参数的顺序和类型必须匹配
- DTO 类不需要是实体

### 别名的使用规范

```hql
-- AS 关键字可省略，建议保留以提高可读性
SELECT u FROM User AS u        -- 推荐（显式 AS）
SELECT u FROM User u           -- 允许（省略 AS）

-- 多实体查询时需要别名区分
SELECT u, o FROM User u, Order o WHERE u.id = o.user.id
```

## 2.2 FROM 子句

FROM 子句指定查询的实体路径，支持多实体查询：

```java
// 单实体查询
List<User> users = session.createQuery("FROM User", User.class).list();

// 多实体查询（笛卡尔积 → 通常配合 WHERE 过滤）
List<Object[]> result = session.createQuery(
    "SELECT u, c FROM User u, Company c WHERE u.company.id = c.id"
).getResultList();

// 多态查询：查询基类时自动 UNION 所有子类
List<Payment> payments = session.createQuery(
    "FROM Payment p WHERE p.amount > 1000", Payment.class
).getResultList();
// 生成 SQL: SELECT ... FROM payment p0_ UNION ALL SELECT ... FROM credit_card_payment cc1_ UNION ALL SELECT ... FROM bank_transfer_payment bt2_
```

## 2.3 WHERE 子句

### 2.3.1 比较运算符

```hql
-- 基本比较：=, <>, !=, >, >=, <, <=
SELECT u FROM User u WHERE u.age >= 18 AND u.status <> 'DELETED'

-- IS NULL / IS NOT NULL
SELECT o FROM Order o WHERE o.shippedDate IS NULL
SELECT o FROM Order o WHERE o.shippedDate IS NOT NULL

-- IS EMPTY / IS NOT EMPTY（检查集合属性是否为空）
SELECT u FROM User u WHERE u.orders IS EMPTY          -- 没有订单的用户
SELECT u FROM User u WHERE u.orders IS NOT EMPTY      -- 有订单的用户

-- BETWEEN ... AND ...
SELECT o FROM Order o WHERE o.amount BETWEEN 100 AND 500
SELECT o FROM Order o WHERE o.createdDate BETWEEN :start AND :end

-- [NOT] IN
SELECT u FROM User u WHERE u.status IN ('ACTIVE', 'PENDING')
SELECT u FROM User u WHERE u.id NOT IN (1L, 2L, 3L)
SELECT u FROM User u WHERE u.id IN :idList              -- 绑定集合参数

-- [NOT] LIKE
SELECT u FROM User u WHERE u.username LIKE 'admin%'     -- 以 admin 开头
SELECT u FROM User u WHERE u.email LIKE '%@gmail.com'   -- Gmail 邮箱
SELECT u FROM User u WHERE u.phone LIKE '138________'   -- 手机号（_ 匹配单字）
SELECT u FROM User u WHERE u.code LIKE '%\%OO' ESCAPE '\' -- 转义字符

-- [NOT] MEMBER [OF]（检查元素是否在集合中）
SELECT u FROM User u WHERE :targetRole MEMBER OF u.roles
SELECT u FROM User u WHERE :product NOT MEMBER OF u.favoriteProducts
```

### 2.3.2 逻辑运算符

```hql
-- AND / OR / NOT，优先级：NOT > AND > OR
SELECT u FROM User u
WHERE u.age >= 18
  AND (u.status = 'ACTIVE' OR u.vipLevel > 3)
  AND NOT u.deleted

-- 建议使用括号明确优先级，避免歧义
```

### 2.3.3 函数

**字符串函数：**

```hql
-- CONCAT 字符串拼接
SELECT CONCAT(u.firstName, ' ', u.lastName) FROM User u

-- SUBSTRING 截取
SELECT SUBSTRING(u.email, 1, 5) FROM User u              -- 前5个字符
SELECT SUBSTRING(u.email, LOCATE('@', u.email) + 1) FROM User u  -- 域名

-- TRIM / LOWER / UPPER
SELECT TRIM(u.username) FROM User u
SELECT LOWER(u.email), UPPER(u.code) FROM User u

-- LENGTH 长度
SELECT u FROM User u WHERE LENGTH(u.username) < 20

-- LOCATE 查找位置（1-based）
SELECT u FROM User u WHERE LOCATE('@', u.email) > 0

-- REPLACE 替换
SELECT REPLACE(u.email, '@old.com', '@new.com') FROM User u
```

**数值函数：**

```hql
SELECT ABS(o.amount) FROM Order o                         -- 绝对值
SELECT SQRT(o.amount) FROM Order o                        -- 平方根
SELECT MOD(o.amount, 10) FROM Order o                     -- 取模
SELECT SIZE(u.orders) FROM User u                         -- 集合大小
```

**日期时间函数：**

```hql
-- 获取当前时间
SELECT u FROM User u WHERE u.createdDate > CURRENT_DATE
SELECT o FROM Order o WHERE o.orderTime < CURRENT_TIME
SELECT e FROM Event e WHERE e.timestamp > CURRENT_TIMESTAMP

-- Hibernate 扩展日期函数需通过 Dialect 注册
-- 常见扩展（是否可用取决于 Hibernate 版本和数据库方言）：
-- year(date), month(date), day(date), hour(date), minute(date), second(date)
-- date_add(date, amount, unit), date_sub(date, amount, unit)
```

**条件函数：**

```hql
-- COALESCE：返回第一个非空值
SELECT COALESCE(u.nickname, u.username, 'Anonymous') FROM User u

-- NULLIF：两个参数相等时返回 NULL
SELECT NULLIF(u.counter, -1) FROM User u
```

**集合函数：**

```hql
-- SIZE：返回集合的元素数量
SELECT u FROM User u WHERE SIZE(u.orders) > 5

-- INDEX：配合 @OrderColumn 获取列表索引
SELECT o FROM Order o JOIN o.items i WHERE INDEX(i) = 0   -- 第一个子项
```

**类型转换：**

```hql
-- CAST：类型转换
SELECT CAST(o.amount AS string) FROM Order o
SELECT CAST(u.status AS java.lang.String) FROM User u

-- TREAT：窄化类型（多态场景）
SELECT a.accountNumber FROM Account a
WHERE TREAT(a AS CreditAccount).creditLimit > 10000
```

### 2.3.4 集合相关表达式

```hql
-- EMPTY / NOT EMPTY：检查集合属性是否为空
SELECT d FROM Department d WHERE d.employees IS EMPTY     -- 空部门
SELECT d FROM Department d WHERE d.employees IS NOT EMPTY -- 有员工的部门

-- MEMBER OF / NOT MEMBER OF：检查元素是否在集合中
SELECT t FROM Team t WHERE :specificUser MEMBER OF t.members
SELECT u FROM User u WHERE :adminRole NOT MEMBER OF u.roles

-- SIZE：获取集合元素数量（可在 WHERE / ORDER BY 中使用）
SELECT u FROM User u ORDER BY SIZE(u.orders) DESC
SELECT u FROM User u WHERE SIZE(u.comments) >= 10
```

## 2.4 ORDER BY 子句

```hql
-- 单字段排序
SELECT u FROM User u ORDER BY u.age DESC
SELECT u FROM User u ORDER BY u.username ASC              -- ASC 可省略（默认）

-- 多字段排序
SELECT o FROM Order o ORDER BY o.createdDate DESC, o.amount ASC

-- NULL 排序控制（HQL 扩展，JPQL 标准无此语法）
SELECT u FROM User u ORDER BY u.lastLoginDate ASC NULLS FIRST
SELECT u FROM User u ORDER BY u.email DESC NULLS LAST

-- 关联属性排序
SELECT o FROM Order o ORDER BY o.user.username, o.amount

-- 函数排序
SELECT u FROM User u ORDER BY LENGTH(u.username) DESC
```

## 2.5 GROUP BY / HAVING 子句

**聚合函数：**

| 函数 | 说明 | 示例 |
|------|------|------|
| `COUNT(*)` | 统计行数（包括 null） | `COUNT(*)` |
| `COUNT(expr)` | 统计非 null 值数量 | `COUNT(u.email)` |
| `COUNT(DISTINCT expr)` | 统计去重后数量 | `COUNT(DISTINCT u.status)` |
| `SUM(expr)` | 求和 | `SUM(o.amount)` |
| `AVG(expr)` | 平均值 | `AVG(o.amount)` |
| `MIN(expr)` | 最小值 | `MIN(o.amount)` |
| `MAX(expr)` | 最大值 | `MAX(o.amount)` |

```java
// 单字段分组
List<Object[]> result1 = session.createQuery(
    "SELECT u.status, COUNT(u) FROM User u GROUP BY u.status"
).getResultList();
// 结果：[ ["ACTIVE", 150], ["INACTIVE", 30], ["BANNED", 5] ]

// 多字段分组
List<Object[]> result2 = session.createQuery(
    "SELECT u.status, u.vipLevel, COUNT(u), AVG(u.age) " +
    "FROM User u GROUP BY u.status, u.vipLevel"
).getResultList();

// HAVING 过滤分组（HAVING 在分组后过滤）
List<Object[]> result3 = session.createQuery(
    "SELECT u.status, COUNT(u) FROM User u " +
    "GROUP BY u.status " +
    "HAVING COUNT(u) > 10 " +              -- 人数超过10的状态
    "ORDER BY COUNT(u) DESC"
).getResultList();

// 聚合 + 条件（WHERE 在分组前，HAVING 在分组后）
List<Object[]> result4 = session.createQuery(
    "SELECT d.name, COUNT(e), AVG(e.salary) " +
    "FROM Department d JOIN d.employees e " +
    "WHERE e.status = 'ACTIVE' " +          -- 分组前过滤员工
    "GROUP BY d.name " +
    "HAVING AVG(e.salary) > 5000 " +        -- 分组后过滤部门
    "ORDER BY AVG(e.salary) DESC"
).getResultList();
```

# 3. 关联查询

## 3.1 隐式关联（Implicit Join）

HQL 通过导航属性自动生成 JOIN，无需显式声明，称为隐式关联：

```java
// 通过导航属性 u.address.city 访问关联实体
List<Object[]> result = session.createQuery(
    "SELECT u.username, u.address.city FROM User u WHERE u.address.city = 'Beijing'"
).getResultList();

// 隐式多级导航
List<Object[]> result2 = session.createQuery(
    "SELECT o.id, o.user.department.name FROM Order o"
).getResultList();
```

生成 SQL：Hibernate 自动将 `u.address.city` 转换为 `INNER JOIN address`：

```sql
SELECT u.username, a.city
FROM t_user u
INNER JOIN t_address a ON u.address_id = a.id
WHERE a.city = 'Beijing';
```

注意事项：
- 隐式关联**默认使用内连接**（INNER JOIN），若需要外连接必须显式声明
- 隐式连接无法添加额外的过滤条件（需用 `WITH` 或显式 JOIN）
- 建议对简单导航使用隐式关联，对需要控制连接类型的场景使用显式关联

## 3.2 显式关联（Explicit Join）

### 3.2.1 INNER JOIN

```java
// JOIN / INNER JOIN：仅返回有匹配的实体
List<User> users = session.createQuery(
    "SELECT u FROM User u INNER JOIN u.orders o WHERE o.amount > 1000",
    User.class
).getResultList();

// 等价于 ON 条件：
// SELECT u.* FROM t_user u INNER JOIN t_order o ON u.id = o.user_id WHERE o.amount > 1000
```

**ON 条件 vs WITH 条件：**

```hql
-- JPA 标准：ON 子句（JPA 2.1+）
SELECT u FROM User u JOIN u.orders o ON o.amount > 1000

-- HQL 扩展：WITH 子句（等价的 HQL 写法）
SELECT u FROM User u JOIN u.orders o WITH o.amount > 1000

-- 区别：ON/WITH 只是在 JOIN 时附加过滤，不影响最终结果集
--       WHERE 在整表结果上过滤，可能改变结果语义（LEFT JOIN 时尤为明显）
```

```java
// ON 与 WHERE 的语义区别示例
// ON：返回所有用户，但只连接金额>1000的订单
List<Object[]> r1 = session.createQuery(
    "SELECT u, o FROM User u LEFT JOIN u.orders o ON o.amount > 1000"
).getResultList();

// WHERE：仅返回有金额>1000订单的用户（LEFT JOIN 退化为 INNER JOIN）
List<Object[]> r2 = session.createQuery(
    "SELECT u, o FROM User u LEFT JOIN u.orders o WHERE o.amount > 1000"
).getResultList();
```

### 3.2.2 LEFT OUTER JOIN

```hql
-- LEFT JOIN / LEFT OUTER JOIN：返回左侧所有实体，右侧无匹配时为 NULL
SELECT u FROM User u LEFT JOIN u.orders o

-- 典型场景：查找没有订单的用户
SELECT u FROM User u LEFT JOIN u.orders o WHERE o.id IS NULL

-- 查找有订单的用户及其订单数
SELECT u, COUNT(o) FROM User u LEFT JOIN u.orders o GROUP BY u
```

```java
// 查找每个用户及其最新订单
List<Object[]> result = session.createQuery(
    "SELECT u, o FROM User u LEFT JOIN u.orders o " +
    "WHERE o IS NULL OR o.createdDate = (SELECT MAX(o2.createdDate) FROM u.orders o2)"
).getResultList();
```

### 3.2.3 RIGHT OUTER JOIN（HQL 扩展）

```hql
-- 仅 HQL 支持，JPQL 不支持
SELECT o, u FROM Order o RIGHT JOIN o.user u

-- 等价于 LEFT JOIN 换边
SELECT u, o FROM User u LEFT JOIN u.orders o
```

### 3.2.4 FETCH JOIN（抓取连接）

FETCH JOIN 是最强大的 HQL 特性之一，**在一次查询中同时加载关联实体，解决 N+1 懒加载问题**。

```java
// 不使用 FETCH JOIN：每个用户的 orders 属性触发一次查询（N+1 问题）
List<User> users = session.createQuery("FROM User u", User.class).list();
for (User u : users) {
    u.getOrders().size();  // 每次访问触发一次 SQL 查询
}

// 使用 FETCH JOIN：一次查询同时加载 User 和 Order
List<User> users = session.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders", User.class
).getResultList();
```

生成 SQL：

```sql
SELECT u.*, o.*
FROM t_user u
INNER JOIN t_order o ON u.id = o.user_id;
```

**LEFT JOIN FETCH：**

```hql
-- 加载所有用户，同时预先加载他们的订单（即使没有订单也返回用户）
SELECT u FROM User u LEFT JOIN FETCH u.orders
```

**FETCH JOIN 对集合的限制：**

```java
// 问题：同时 FETCH 两个 List 类型的关联（BAG）
List<User> users = session.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders JOIN FETCH u.comments",
    User.class
).getResultList();
// 抛出：org.hibernate.loader.MultipleBagFetchException
// 原因：Hibernate 无法在一条 SQL 中处理两个笛卡尔积集合

// 解决方案一：将 BAG 改为 SET（使用 @OrderBy 替代 @OrderColumn）
@Entity
public class User {
    @OneToMany
    @OrderBy("createdDate DESC")
    private Set<Comment> comments;  // Set 替代 List
}

// 解决方案二：分两次查询（推荐）
List<User> users = session.createQuery(
    "SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.orders", User.class
).getResultList();
// 再在需要时加载 comments（可配合 @BatchSize 批量加载）
```

**FETCH JOIN 与 Pagination 的冲突：**

```java
// 问题：JOIN FETCH + setMaxResults()
// Hibernate 5.x 及之前：先全量加载到内存再分页（大表危险）
// Hibernate 6.x：自动拆分查询，先查 ID 再根据 ID FETCH

// 手动解决方案（两步查询法）：
// 第一步：只查 ID + 分页
Page<Long> idPage = session.createQuery(
    "SELECT u.id FROM User u ORDER BY u.createdDate DESC", Long.class
)
.setFirstResult(0)
.setMaxResults(20)
.getResultList();

// 第二步：根据 ID 查详情 + FETCH JOIN
List<User> users = session.createQuery(
    "SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.orders " +
    "WHERE u.id IN :ids ORDER BY u.createdDate DESC",
    User.class
)
.setParameter("ids", idPage)
.getResultList();
```

### 3.2.5 CROSS JOIN

```hql
-- 笛卡尔积（所有组合）
-- 通常不推荐直接使用，除非明确需要
SELECT u, c FROM User u, Company c

-- 等价 SQL：SELECT u.*, c.* FROM t_user u, t_company c
```

## 3.3 WITH 子句（HQL 扩展）

```hql
-- WITH：在 JOIN 时附加过滤条件（与 ON 等价）
SELECT u FROM User u JOIN u.orders o WITH o.status = 'PAID'

-- WITH 与 WHERE 在 LEFT JOIN 中的语义区别：
-- WITH：不影响左侧实体是否返回
SELECT u, o FROM User u LEFT JOIN u.orders o WITH o.amount > 1000
-- 即便订单金额 <= 1000，用户仍然返回，订单字段为 NULL

-- WHERE：会影响左侧实体
SELECT u, o FROM User u LEFT JOIN u.orders o WHERE o.amount > 1000
-- 订单金额 <= 1000 的行被过滤，等同于 INNER JOIN

-- 多条件 WITH
SELECT u FROM User u
JOIN u.orders o WITH o.amount > 1000 AND o.status = 'CONFIRMED'
LEFT JOIN u.addresses a WITH a.isDefault = true
```

# 4. 子查询

## 4.1 WHERE 子句中的子查询

**EXISTS / NOT EXISTS：**

```java
// 查询有订单的用户
List<User> users1 = session.createQuery(
    "SELECT u FROM User u WHERE EXISTS " +
    "(SELECT 1 FROM Order o WHERE o.user = u AND o.amount > 1000)",
    User.class
).getResultList();

// 查询没有订单的用户
List<User> users2 = session.createQuery(
    "SELECT u FROM User u WHERE NOT EXISTS " +
    "(SELECT 1 FROM Order o WHERE o.user = u)",
    User.class
).getResultList();
```

**IN / NOT IN：**

```java
// 查询有金额大于1000的订单的用户
List<User> users3 = session.createQuery(
    "SELECT u FROM User u WHERE u IN " +
    "(SELECT o.user FROM Order o WHERE o.amount > 1000)",
    User.class
).getResultList();

// NOT IN 查找无订单用户
List<User> users4 = session.createQuery(
    "SELECT u FROM User u WHERE u NOT IN " +
    "(SELECT o.user FROM Order o)",
    User.class
).getResultList();
```

**ALL / ANY / SOME 比较：**

```java
// 查询比所有北京用户年龄都大的用户
List<User> users5 = session.createQuery(
    "SELECT u FROM User u WHERE u.age > ALL " +
    "(SELECT u2.age FROM User u2 WHERE u2.address.city = 'Beijing')",
    User.class
).getResultList();

// 查询金额大于该类目下任一订单金额的订单
// ANY 与 SOME 等价
List<Order> orders = session.createQuery(
    "SELECT o FROM Order o WHERE o.amount > ANY " +
    "(SELECT o2.amount FROM Order o2 WHERE o2.category = o.category AND o2.id <> o.id)",
    Order.class
).getResultList();
```

**关联子查询与独立子查询：**

```java
// 关联子查询：子查询引用了外层查询的别名
List<Product> products = session.createQuery(
    "SELECT p FROM Product p WHERE p.price > " +
    "(SELECT AVG(p2.price) FROM Product p2 WHERE p2.category = p.category)",
    Product.class
).getResultList();

// 独立子查询：子查询独立于外层
List<Product> expensive = session.createQuery(
    "SELECT p FROM Product p WHERE p.price > (SELECT AVG(p2.price) FROM Product p2)",
    Product.class
).getResultList();
```

## 4.2 SELECT 子句中的子查询

```java
// 子查询作为查询列：必须返回单个值（标量子查询）
List<Object[]> result = session.createQuery(
    "SELECT u.username, " +
    "(SELECT COUNT(o) FROM Order o WHERE o.user = u), " +
    "(SELECT SUM(o.amount) FROM Order o WHERE o.user = u) " +
    "FROM User u"
).getResultList();
```

## 4.3 FROM 子句中的子查询（HQL 扩展）

Hibernate 6 支持在 FROM 子句中使用子查询，JPQL 不支持此特性：

```java
// HQL 扩展：子查询作为临时表
List<Object[]> result = session.createQuery(
    "SELECT stats.username, stats.orderCount " +
    "FROM (SELECT u.username AS username, COUNT(o) AS orderCount " +
    "      FROM User u JOIN u.orders o GROUP BY u.username " +
    "      HAVING orderCount > 5) AS stats " +
    "ORDER BY stats.orderCount DESC"
).getResultList();
```

# 5. 批量操作

## 5.1 UPDATE

HQL 批量 UPDATE 直接操作数据库，**不经过持久化上下文**：

```java
// 批量更新所有过期用户状态
int updatedCount = session.createMutationQuery(
    "UPDATE User u SET u.status = 'EXPIRED' WHERE u.expireDate < :today"
)
.setParameter("today", LocalDate.now())
.executeUpdate();

System.out.println("更新了 " + updatedCount + " 条记录");
```

**批量更新与 Session 缓存的一致性问题：**

```java
// 演示：更新前的数据已在 Session 缓存中
User user = session.get(User.class, 1L);  // 缓存中有 id=1 的用户，status='ACTIVE'

// 批量 UPDATE 直接操作数据库，不更新一级缓存
session.createMutationQuery("UPDATE User u SET u.status = 'BANNED' WHERE u.id = 1L")
    .executeUpdate();

// 此时数据库 status 是 'BANNED'，但 Session 中的实体 status 仍是 'ACTIVE'！
// 缓存中的版本号、脏检查都需要注意

// 解决方案一：更新后清除 Session 缓存
session.clear();
// 或
session.evict(user);

// 解决方案二：执行更新后强制刷新（确保更新 SQL 先于其他操作执行）
session.flush();
session.clear();

// 解决方案三：使用 StatelessSession 执行批量操作（无缓存干扰）
```

## 5.2 DELETE

```java
// 批量删除
int deletedCount = session.createMutationQuery(
    "DELETE FROM LogEntry l WHERE l.createdDate < :cutoffDate"
)
.setParameter("cutoffDate", LocalDate.now().minusMonths(6))
.executeUpdate();
```

**注意事项：**

```java
// 问题一：批量删除不触发级联
// 即便定义了 @OneToMany(cascade = CascadeType.REMOVE)，批量删除不会触发
// 需要先手动删除关联数据
session.createMutationQuery("DELETE FROM OrderItem oi WHERE oi.order.user.id = :userId")
    .setParameter("userId", 1001L)
    .executeUpdate();
session.createMutationQuery("DELETE FROM Order o WHERE o.user.id = :userId")
    .setParameter("userId", 1001L)
    .executeUpdate();
session.createMutationQuery("DELETE FROM User u WHERE u.id = :userId")
    .setParameter("userId", 1001L)
    .executeUpdate();

// 问题二：生命周期回调不会被调用
// @PreRemove、@PostRemove 不会执行

// 问题三：版本号不更新
// 如果有 @Version 字段，批量 UPDATE/DELETE 不会更新版本号
// 需要手动在 UPDATE 中递增版本号：
// "UPDATE User u SET u.status = 'DELETED', u.version = u.version + 1 WHERE ..."

// 推荐做法：批量操作后清除 Session
session.flush();
session.clear();
```

## 5.3 INSERT（HQL 扩展）

HQL 支持 INSERT INTO ... SELECT 语法，JPQL 不支持：

```java
// 将过期日志归档到归档表
int insertedCount = session.createMutationQuery(
    "INSERT INTO LogArchive (id, message, level, createdAt) " +
    "SELECT l.id, l.message, l.level, l.createdAt " +
    "FROM LogEntry l WHERE l.createdAt < :cutoffDate"
)
.setParameter("cutoffDate", LocalDate.now().minusMonths(6))
.executeUpdate();
```

使用场景：
- 数据归档/备份
- 批量数据迁移
- 避免应用层逐条处理大量数据

# 6. 命名查询

## 6.1 @NamedQuery / @NamedQueries

静态查询定义，在实体类上声明，Hibernate 启动时可预处理：

```java
@Entity
@NamedQueries({
    @NamedQuery(
        name = "User.findByStatus",
        query = "SELECT u FROM User u WHERE u.status = :status ORDER BY u.createdDate DESC"
    ),
    @NamedQuery(
        name = "User.findByAgeRange",
        query = "SELECT u FROM User u WHERE u.age BETWEEN :minAge AND :maxAge"
    ),
    @NamedQuery(
        name = "User.countByStatus",
        query = "SELECT u.status, COUNT(u) FROM User u GROUP BY u.status"
    )
})
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;
    private Integer age;
    private String status;
    // ...
}

// 使用命名查询
TypedQuery<User> query = entityManager.createNamedQuery("User.findByStatus", User.class);
List<User> users = query.setParameter("status", "ACTIVE").getResultList();

// 配合 Hint 使用
List<User> readOnlyUsers = entityManager.createNamedQuery("User.findByStatus", User.class)
    .setParameter("status", "ACTIVE")
    .setHint("org.hibernate.readOnly", true)
    .setHint("org.hibernate.cacheable", true)
    .getResultList();
```

**Hibernate 命名查询预编译配置：**

```java
// application.properties（Spring Boot）
spring.jpa.properties.hibernate.query.startup_check=true
spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch=true

// 或 hibernate.cfg.xml
<property name="hibernate.query.startup_check">true</property>
```

设置 `hibernate.query.startup_check=true` 后，Hibernate 在启动时会验证所有命名查询的语法和实体映射正确性，避免运行时才暴露 HQL 语法错误。

## 6.2 @NamedNativeQuery / @NamedNativeQueries

用于定义命名原生 SQL 查询，可配合结果集映射：

```java
@Entity
@NamedNativeQueries({
    @NamedNativeQuery(
        name = "User.findTopSpenders",
        query = "SELECT u.id, u.username, SUM(o.amount) as total_spent " +
                "FROM t_user u JOIN t_order o ON u.id = o.user_id " +
                "GROUP BY u.id, u.username " +
                "HAVING SUM(o.amount) > :minAmount " +
                "ORDER BY total_spent DESC",
        resultSetMapping = "UserSpendingMapping"
    )
})
public class User { /* ... */ }

// 结果集映射定义（可在任意实体的注解中定义）
@SqlResultSetMapping(
    name = "UserSpendingMapping",
    entities = {
        @EntityResult(entityClass = User.class)
    },
    columns = {
        @ColumnResult(name = "total_spent", type = BigDecimal.class)
    }
)
// 或配合 @ConstructorResult 映射到 DTO
@SqlResultSetMapping(
    name = "UserSpendingDTOMapping",
    classes = @ConstructorResult(
        targetClass = UserSpendingDTO.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "username", type = String.class),
            @ColumnResult(name = "total_spent", type = BigDecimal.class)
        }
    )
)
public class SomeEntity { /* 可放在任意实体上 */ }

// 使用
List<Object[]> results = entityManager.createNamedQuery("User.findTopSpenders")
    .setParameter("minAmount", new BigDecimal("10000"))
    .getResultList();
```

## 6.3 @NamedEntityGraph

实体图（EntityGraph）定义特定查询场景下需要立即加载的关联属性，避免 N+1 问题和 FETCH JOIN 的副作用：

```java
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "User.withOrders",
        attributeNodes = {
            @NamedAttributeNode("orders")
        }
    ),
    @NamedEntityGraph(
        name = "User.withOrdersAndAddress",
        attributeNodes = {
            @NamedAttributeNode("orders"),
            @NamedAttributeNode("address")
        }
    ),
    // 带子图的实体图
    @NamedEntityGraph(
        name = "User.withOrdersAndItems",
        attributeNodes = {
            @NamedAttributeNode(value = "orders", subgraph = "orderItems")
        },
        subgraphs = {
            @NamedSubgraph(
                name = "orderItems",
                attributeNodes = @NamedAttributeNode("items")
            )
        }
    )
})
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;

    @ManyToOne
    private Address address;
    // ...
}

// 使用方式一：EntityManager.find() 中应用 EntityGraph
EntityGraph<?> graph = entityManager.getEntityGraph("User.withOrders");
Map<String, Object> hints = new HashMap<>();
hints.put("jakarta.persistence.fetchgraph", graph);
User user = entityManager.find(User.class, 1L, hints);

// 使用方式二：Query.setHint() 应用 EntityGraph
List<User> users = entityManager.createQuery("SELECT u FROM User u", User.class)
    .setHint("jakarta.persistence.fetchgraph",
             entityManager.getEntityGraph("User.withOrders"))
    .getResultList();

// 使用方式三：Spring Data JPA @EntityGraph 注解
// @EntityGraph(value = "User.withOrders", type = EntityGraphType.FETCH)
// List<User> findAll();
```

**fetchgraph vs loadgraph 的区别：**

| 特性 | fetchgraph | loadgraph |
|------|-----------|-----------|
| 指定节点 | EAGER 加载 | EAGER 加载 |
| 未指定节点的默认字段 | **LAZY** 加载 | 按实体映射的 FetchType 处理 |
| 实体图中未列出的 `@Basic(fetch = EAGER)` | LAZY 加载 | EAGER 加载 |
| 实体图中未列出的 `@ManyToOne(fetch = EAGER)` | LAZY 加载 | EAGER 加载 |
| 典型场景 | 精确控制加载哪些属性 | 在现有 EAGER 基础上追加关联 |

```java
// fetchgraph：只加载 orders，address 即使 EAGER 也不会立即加载
entityManager.createQuery("SELECT u FROM User u", User.class)
    .setHint("jakarta.persistence.fetchgraph", graphWithOrdersOnly);
// 行为：User 的基本字段 + orders 集合加载，其他关联不加载

// loadgraph：加载 orders，address 按映射的 EAGER/LAZY 处理
entityManager.createQuery("SELECT u FROM User u", User.class)
    .setHint("jakarta.persistence.loadgraph", graphWithOrdersOnly);
// 行为：若 address 是 EAGER，则 address 也会一起加载
```

# 7. 分页查询

## 7.1 setFirstResult / setMaxResults

JPA 标准分页 API，Hibernate 自动生成对应数据库方言的分页 SQL：

```java
// 标准分页
TypedQuery<User> query = session.createQuery(
    "SELECT u FROM User u ORDER BY u.createdDate DESC", User.class
);
query.setFirstResult(0);    // 偏移量（页码 * 每页大小）
query.setMaxResults(20);    // 每页大小
List<User> page = query.getResultList();

// 不同数据库生成的 SQL 不同
// MySQL:       SELECT ... FROM t_user u ORDER BY ... LIMIT 20 OFFSET 0
// PostgreSQL:  SELECT ... FROM t_user u ORDER BY ... LIMIT 20 OFFSET 0
// Oracle:      SELECT * FROM (SELECT rownum r_, ... FROM t_user u ORDER BY ...) WHERE r_ BETWEEN 1 AND 20
// SQL Server:  SELECT ... FROM t_user u ORDER BY ... OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY

// Spring Data JPA 分页
Page<User> userPage = userRepository.findAll(
    PageRequest.of(0, 20, Sort.by("createdDate").descending())
);
```

## 7.2 分页与集合抓取的冲突

**核心问题：** 当 HQL 包含 JOIN FETCH 子句时，Hibernate 无法直接在 SQL 层面做 LIMIT/OFFSET。

```java
// 问题代码：JOIN FETCH + setMaxResults
TypedQuery<User> query = session.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders ORDER BY u.createdDate DESC",
    User.class
);
query.setFirstResult(0).setMaxResults(20);
// 危险：Hibernate 5.x 会将所有数据加载到内存，然后内存分页
// 若 User 表有 100 万行，内存溢出
```

**Hibernate 6 的改进：**

Hibernate 6 自动检测此场景并将查询拆分为两步：
1. 先查询 ID 分页
2. 再根据 ID 做 FETCH JOIN

无需手动干预，但需注意控制台可能出现两条 SQL。

**手动解决方案（两步查询法，兼容 Hibernate 5.x）：**

```java
// 第一步：只查 ID 分页
List<Long> ids = session.createQuery(
    "SELECT u.id FROM User u ORDER BY u.createdDate DESC", Long.class
)
.setFirstResult(0)
.setMaxResults(20)
.getResultList();

// 第二步：根据 ID 查询 + FETCH JOIN（无分页限制）
List<User> users = session.createQuery(
    "SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.orders " +
    "WHERE u.id IN :ids ORDER BY u.createdDate DESC",
    User.class
)
.setParameter("ids", ids)
.getResultList();
```

**使用 @NamedEntityGraph 替代 FETCH JOIN + 分页：**

```java
// 更优雅的方案：EntityGraph + 分页
EntityGraph<?> graph = entityManager.getEntityGraph("User.withOrders");
List<User> users = entityManager.createQuery(
    "SELECT u FROM User u ORDER BY u.createdDate DESC", User.class
)
.setHint("jakarta.persistence.fetchgraph", graph)
.setFirstResult(0)
.setMaxResults(20)
.getResultList();
// EntityGraph 在加载 User 后以批量 SQL 方式加载关联，不干扰分页
```

## 7.3 游标分页

```java
// ScrollableResults：JDBC 可滚动结果集
// 适用场景：大结果集，需要逐条处理，或需要无偏移分页

try (ScrollableResults<User> scroll = session.createQuery(
    "SELECT u FROM User u ORDER BY u.id", User.class
)
.setFetchSize(50)            // JDBC FetchSize，每次取50条
.setReadOnly(true)           // 只读模式（跳过脏检查）
.scroll(ScrollMode.FORWARD_ONLY)  // 仅向前滚动
) {
    while (scroll.next()) {
        User user = scroll.get();
        // 逐条处理，不占用大量内存
        processUser(user);

        // 定期清除 Session 避免内存膨胀
        if (scroll.getRowNumber() % 100 == 0) {
            session.flush();
            session.clear();
        }
    }
}
```

ScrollableResults 与 setFirstResult/setMaxResults 的对比：

| 特性 | setFirstResult/setMaxResults | ScrollableResults |
|------|------------------------------|-------------------|
| 内存占用 | 一次性加载当前页 | 逐条/批量加载 |
| 大结果集 | 依赖 OFFSET 性能差 | 游标遍历，性能稳定 |
| 跳页 | 支持任意页码 | 仅支持顺序滚动 |
| 实时性 | 每次查询新数据 | 结果集快照 |
| 数据库连接 | 查询后释放 | 持有连接直到关闭 |
| 使用复杂度 | 简单 | 需手动管理资源 |

# 8. 参数绑定

## 8.1 位置参数

```java
// 位置参数：?1, ?2, ?3 ...（序号从 1 开始）
// 不推荐：代码变更时序号易出错，可读性差
TypedQuery<User> query = session.createQuery(
    "SELECT u FROM User u WHERE u.age > ?1 AND u.status = ?2", User.class
);
List<User> users = query
    .setParameter(1, 18)
    .setParameter(2, "ACTIVE")
    .getResultList();

// 位置参数不支持 IN 集合绑定（需用命名参数）
```

## 8.2 命名参数

```java
// 命名参数：:paramName（推荐方式）
TypedQuery<User> query = session.createQuery(
    "SELECT u FROM User u WHERE u.age BETWEEN :minAge AND :maxAge " +
    "AND u.status = :status",
    User.class
);
List<User> users = query
    .setParameter("minAge", 18)
    .setParameter("maxAge", 65)
    .setParameter("status", "ACTIVE")
    .getResultList();

// setParameterList：绑定集合参数（IN 子句）
List<String> statusList = Arrays.asList("ACTIVE", "PENDING", "VERIFIED");
TypedQuery<User> query2 = session.createQuery(
    "SELECT u FROM User u WHERE u.status IN :statusList", User.class
)
.setParameterList("statusList", statusList);
// 也可用 setParameter("statusList", Arrays.asList("ACTIVE", "PENDING"))

// TypedParameterValue：显式指定参数类型
TypedQuery<User> query3 = session.createQuery(
    "SELECT u FROM User u WHERE u.enumField = :enumVal", User.class
)
.setParameter("enumVal",
    new TypedParameterValue(StandardBasicTypes.STRING, "ACTIVE"));
```

## 8.3 参数类型推断

Hibernate 能够从实体映射自动推断参数类型，但特定场景需显式指定：

```java
// 自动推断：根据实体字段的 @Enumerated、@Temporal 等自动判断
query.setParameter("status", Status.ACTIVE);          // 自动推断为枚举类型
query.setParameter("date", LocalDate.now());          // 自动推断为 DATE

// 需要显式指定的场景：

// 场景一：null 值参数（Hibernate 无法推断 null 的类型）
query.setParameter("nullableField", null, StandardBasicTypes.STRING);

// 场景二：集合成员类型（List<Enum> 需要指定元素类型）
// Hibernate 5.x 推荐用 TypedParameterValue，6.x 多数情况自动推断
query.setParameter("statusList", statusList);

// 场景三：使用 JPA TemporalType 指定日期精度
query.setParameter("datetime", date, TemporalType.DATE);       // 仅日期
query.setParameter("datetime", date, TemporalType.TIMESTAMP);  // 日期+时间
```

# 9. 查询 Hint

## 9.1 JPA 查询 Hint

JPA 规范定义了一组标准 Hint（通过 `jakarta.persistence.*` 前缀）：

```java
TypedQuery<User> query = entityManager.createQuery(
    "SELECT u FROM User u", User.class
);

// 查询超时（毫秒）
query.setHint("jakarta.persistence.query.timeout", 5000);

// 实体图
query.setHint("jakarta.persistence.fetchgraph", entityGraph);
query.setHint("jakarta.persistence.loadgraph", entityGraph);

// 缓存控制
query.setHint("jakarta.persistence.cache.retrieveMode",
              CacheRetrieveMode.BYPASS);  // 跳过二级缓存读取
query.setHint("jakarta.persistence.cache.storeMode",
              CacheStoreMode.BYPASS);     // 跳过二级缓存写入

// 悲观锁超时（毫秒）
query.setHint("jakarta.persistence.lock.timeout", 3000);
```

## 9.2 Hibernate 查询 Hint

Hibernate 提供更丰富的查询 Hint（`org.hibernate.*` 前缀）：

```java
// 只读查询（跳过脏检查快照，性能更好）
query.setHint("org.hibernate.readOnly", true);

// 查询缓存（二级缓存，需配合 @Cacheable 和缓存策略配置）
query.setHint("org.hibernate.cacheable", true);
query.setHint("org.hibernate.cacheRegion", "user-queries");

// JDBC FetchSize（每次从数据库拉取的行数，适合大结果集流式处理）
query.setHint("org.hibernate.fetchSize", 50);

// SQL 注释（在生成的 SQL 前添加注释，便于数据库监控/日志分析）
query.setHint("org.hibernate.comment", "Find active users for dashboard");

// 刷新模式
query.setHint("org.hibernate.flushMode", FlushMode.COMMIT);

// 查询超时（秒，与 JPA 的毫秒不同）
query.setHint("org.hibernate.timeout", 30);

// 悲观锁范围
query.setHint("org.hibernate.lockOptions", LockOptions.SKIP_LOCKED);

// 批量加载策略
query.setHint("org.hibernate.batch_fetch_style", BatchFetchStyle.PADDED);
```

**完整 Hint 速查表：**

| Hint Key | 类型 | 说明 |
|----------|------|------|
| `jakarta.persistence.query.timeout` | int (ms) | 查询超时 |
| `jakarta.persistence.fetchgraph` | EntityGraph | 精确控制加载图 |
| `jakarta.persistence.loadgraph` | EntityGraph | 追加加载图 |
| `jakarta.persistence.cache.retrieveMode` | CacheRetrieveMode | 缓存读取策略 |
| `jakarta.persistence.cache.storeMode` | CacheStoreMode | 缓存存储策略 |
| `jakarta.persistence.lock.timeout` | int (ms) | 锁等待超时 |
| `org.hibernate.readOnly` | boolean | 只读模式 |
| `org.hibernate.cacheable` | boolean | 查询缓存 |
| `org.hibernate.cacheRegion` | String | 缓存区域名 |
| `org.hibernate.fetchSize` | int | JDBC FetchSize |
| `org.hibernate.comment` | String | SQL 注释 |
| `org.hibernate.flushMode` | FlushMode | 刷新模式 |
| `org.hibernate.timeout` | int (s) | 超时秒数 |
| `org.hibernate.lockOptions` | LockOptions | 锁选项 |
| `org.hibernate.batch_fetch_style` | BatchFetchStyle | 批量抓取样式 |

# 10. 动态查询构建

```java
// 方式一：字符串拼接（不推荐 — SQL 注入风险 + 代码难以维护）
StringBuilder hql = new StringBuilder("SELECT u FROM User u WHERE 1=1");
if (name != null) {
    hql.append(" AND u.username LIKE '%").append(name).append("%'");  // 安全隐患！
}
if (status != null) {
    hql.append(" AND u.status = '").append(status).append("'");      // 安全隐患！
}

// 方式二：字符串拼接 + 参数绑定（防注入但代码臃肿）
StringBuilder hql = new StringBuilder("SELECT u FROM User u WHERE 1=1");
Map<String, Object> params = new HashMap<>();
if (name != null) {
    hql.append(" AND u.username LIKE :name");
    params.put("name", "%" + name + "%");
}
if (status != null) {
    hql.append(" AND u.status = :status");
    params.put("status", status);
}
TypedQuery<User> query = session.createQuery(hql.toString(), User.class);
params.forEach(query::setParameter);

// 方式三：Criteria API（类型安全，见后续笔记）
// 方式四：Querydsl（编译时类型安全，需引入 querydsl-jpa 依赖）

// 方式五：Spring Data JPA Specification（推荐用于复杂动态查询）
public class UserSpecification {
    public static Specification<User> nameContains(String name) {
        return (root, query, cb) ->
            name == null ? null : cb.like(root.get("username"), "%" + name + "%");
    }

    public static Specification<User> statusEquals(Status status) {
        return (root, query, cb) ->
            status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<User> ageBetween(Integer min, Integer max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("age"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("age"), min);
            return cb.between(root.get("age"), min, max);
        };
    }
}

// 组合使用 Specification
Specification<User> spec = Specification.where(null);
if (nameFilter != null) spec = spec.and(UserSpecification.nameContains(nameFilter));
if (statusFilter != null) spec = spec.and(UserSpecification.statusEquals(statusFilter));
if (minAge != null || maxAge != null) spec = spec.and(UserSpecification.ageBetween(minAge, maxAge));

List<User> users = userRepository.findAll(spec, PageRequest.of(0, 20, Sort.by("createdDate").desc()));
```

# 11. Hibernate 6 SQM（Semantic Query Model）

## 11.1 SQM 概述

Hibernate 6 引入了全新的查询引擎 **SQM（Semantic Query Model）**，这是 HQL/JPQL 执行路径的根本性重构：

- **统一中间表示**：HQL、JPQL、Criteria API 三种查询方式在内部被编译为同一棵 SQM 语法树
- **查询编译器架构**：`HQL/JPQL String -> SQM Tree -> SQL (via SQMTranslator)`
- **优于 Hibernate 5 的 ANTLR 解析器**：解析速度更快，语义分析更精确
- **查询优化器**：SQM 层面做语义优化（如隐式 JOIN 合并、冗余子查询消除）
- **更好的错误信息**：提供带有源位置的精确错误报告

```
旧架构（Hibernate 5）：
  HQL String -> ANTLR AST -> HQLSqlWalker -> SQL
  JPQL String -> ANTLR AST -> HQLSqlWalker -> SQL
  Criteria -> CriteriaQueryTranslator -> SQL
  （三条路径各自实现，维护困难）

新架构（Hibernate 6）：
  HQL String -> SQM Tree ┐
  JPQL String -> SQM Tree ├──> SQMTranslator -> SQL
  Criteria API -> SQM Tree ┘
  （统一中间表示，同一套 SQL 生成逻辑）
```

## 11.2 SQM 的新能力

**FROM 子句中的子查询：**

```java
// Hibernate 6 支持，JPQL 标准不支持
List<Object[]> result = session.createQuery(
    "SELECT sub.name, sub.total " +
    "FROM (SELECT d.name AS name, SUM(e.salary) AS total " +
    "      FROM Department d JOIN d.employees e GROUP BY d.name) AS sub " +
    "WHERE sub.total > 100000 " +
    "ORDER BY sub.total DESC"
).getResultList();
```

**更优的 JOIN 处理：**

- SQM 自动识别隐式 JOIN 和显式 JOIN 的重合，合并为同一 SQL JOIN
- 减少不必要的 JOIN 生成

```java
// Hibernate 5：u.address 和 u.address.city 可能生成两次 address JOIN
// Hibernate 6：SQM 智能合并为一次 JOIN
List<Object[]> result = session.createQuery(
    "SELECT u.username, u.address.city FROM User u " +
    "WHERE u.address.country = 'CN'"
).getResultList();
// 生成 SQL：SELECT u0_.username, a1_.city FROM t_user u0_ INNER JOIN t_address a1_ ON u0_.address_id = a1_.id WHERE a1_.country = 'CN'
```

**CTE（Common Table Expression）支持（部分版本）：**

```java
// Hibernate 6.2+ 通过 org.hibernate.dialect.Dialect.supportsCTE() 控制
// 是否可用取决于数据库方言是否实现了 CTE 语法

// 部分方言支持 WITH RECURSIVE
// 对于复杂的递归查询，目前更稳妥的方式是使用原生 SQL Query
String sql = """
    WITH RECURSIVE category_tree AS (
        SELECT id, name, parent_id, 0 AS depth FROM category WHERE parent_id IS NULL
        UNION ALL
        SELECT c.id, c.name, c.parent_id, ct.depth + 1
        FROM category c JOIN category_tree ct ON c.parent_id = ct.id
    )
    SELECT * FROM category_tree ORDER BY depth, name
    """;
List<Object[]> tree = session.createNativeQuery(sql).getResultList();
```

**窗口函数（通过原生 SQL 更稳定）：**

```java
// HQL 层面部分版本支持，但推荐使用原生 SQL
String sql = """
    SELECT username, amount,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rank
    FROM t_order
    """;
```

# 12. 函数速查表

## 12.1 JPA 标准函数（所有 JPA 实现均支持）

**字符串函数：**

| 函数 | 语法 | 返回值 | 示例 |
|------|------|--------|------|
| CONCAT | `CONCAT(s1, s2, ...)` | String | `CONCAT(u.first, ' ', u.last)` |
| SUBSTRING | `SUBSTRING(s, start, length)` | String | `SUBSTRING(u.email, 1, 5)` |
| TRIM | `TRIM([LEADING\|TRAILING\|BOTH] [char] FROM s)` | String | `TRIM(u.username)` |
| LOWER | `LOWER(s)` | String | `LOWER(u.email)` |
| UPPER | `UPPER(s)` | String | `UPPER(u.code)` |
| LENGTH | `LENGTH(s)` | Integer | `LENGTH(u.username)` |
| LOCATE | `LOCATE(search, s [, start])` | Integer | `LOCATE('@', u.email)` |

**数值函数：**

| 函数 | 语法 | 返回值 | 示例 |
|------|------|--------|------|
| ABS | `ABS(n)` | Number | `ABS(o.amount)` |
| SQRT | `SQRT(n)` | Double | `SQRT(o.amount)` |
| MOD | `MOD(n1, n2)` | Integer | `MOD(o.id, 2)` |
| SIZE | `SIZE(collection)` | Integer | `SIZE(u.orders)` |

**日期时间函数：**

| 函数 | 说明 |
|------|------|
| `CURRENT_DATE` | 当前日期（数据库日期，无时间部分）|
| `CURRENT_TIME` | 当前时间（数据库时间，无日期部分）|
| `CURRENT_TIMESTAMP` | 当前日期时间（数据库完整时间戳）|

**集合与条件函数：**

| 函数 | 语法 | 说明 |
|------|------|------|
| COALESCE | `COALESCE(v1, v2, ...)` | 返回第一个非 null 值 |
| NULLIF | `NULLIF(v1, v2)` | v1 == v2 时返回 null，否则返回 v1 |

**类型转换：**

| 函数 | 语法 | 说明 |
|------|------|------|
| CAST | `CAST(expr AS type)` | 类型转换 |
| TREAT | `TREAT(expr AS SubType)` | 子类型窄化 |

**聚合函数：** `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`

## 12.2 Hibernate 扩展函数（HQL 专有）

Hibernate 6 提供 100+ 内置扩展函数，以下是常用分类：

```hql
-- 日期时间操作
-- year(date), month(date), day(date), hour(date), minute(date), second(date)
SELECT u FROM User u WHERE year(u.createdDate) = 2024
SELECT u FROM User u WHERE month(u.createdDate) = 12
SELECT u FROM User u WHERE hour(u.loginTime) BETWEEN 9 AND 18

-- 日期差值
-- format(datetime, pattern)  -- 格式化日期
-- date_trunc(field, datetime)

-- 字符串操作
-- str(expr)                   -- 转换为字符串
-- left(string, length)        -- 左侧截取
-- right(string, length)       -- 右侧截取
-- replace(string, old, new)   -- 替换
-- collate(string, collation)  -- 指定排序规则

-- 数学函数
-- round(number, places)       -- 四舍五入
-- ceiling(number)             -- 向上取整
-- floor(number)               -- 向下取整
-- sign(number)                -- 正负号
-- log(number), ln(number)     -- 对数
-- power(base, exp)            -- 幂运算
-- radians(degrees), degrees(radians)  -- 角度转换

-- 类型转换
-- cast(expr as Type)
-- str(expr)

-- 集合函数
-- elements(collection)        -- 展开集合为元素
-- indices(collection)         -- 获取 @OrderColumn 索引
-- maxindex(collection)        -- 最大索引
-- minindex(collection)        -- 最小索引
-- maxelement(collection)      -- 最大元素
-- minelement(collection)      -- 最小元素

-- 其他
-- bit_length(string)          -- 位长度
-- extract(field FROM date)    -- 提取日期部分
-- 区分大小写的 LIKE（部分方言）
-- 正则匹配（部分方言）
```

**完整函数可用性查询：**

```java
// 通过 Metadata 获取当前 Dialect 注册的所有 SQL 函数
SessionFactory sf = session.getSessionFactory();
Metadata metadata = sf.unwrap(SessionFactoryImplementor.class).getMetadata();
Set<String> functionNames = metadata.getDatabase().getDialect().getFunctions()
    .keySet();
for (String name : functionNames.stream().sorted().toList()) {
    System.out.println(name);
}
```

## 12.3 自定义函数注册

**方式一：通过自定义 Dialect 注册：**

```java
public class MyPostgreSQLDialect extends PostgreSQLDialect {
    public MyPostgreSQLDialect() {
        super();
        // 注册无参数函数
        registerFunction("my_random",
            new StandardSQLFunction("random", StandardBasicTypes.DOUBLE)
        );

        // 注册有参数函数
        registerFunction("group_concat",
            new SQLFunctionTemplate(
                StandardBasicTypes.STRING,
                "string_agg(?1, ',' ORDER BY ?1)"
            )
        );

        // 注册自定义函数
        registerFunction("date_diff",
            new SQLFunctionTemplate(
                StandardBasicTypes.INTEGER,
                "EXTRACT(DAY FROM (?2 - ?1))"
            )
        );
    }
}
```

**方式二：通过 MetadataBuilderContributor 注册：**

```java
public class CustomFunctionContributor implements MetadataBuilderContributor {
    @Override
    public void contribute(MetadataBuilder metadataBuilder) {
        metadataBuilder.applySqlFunction("date_diff",
            new SQLFunctionTemplate(
                StandardBasicTypes.INTEGER,
                "EXTRACT(DAY FROM (?2 - ?1))"
            )
        );
        metadataBuilder.applySqlFunction("group_concat",
            new SQLFunctionTemplate(
                StandardBasicTypes.STRING,
                "string_agg(distinct ?1, ',' ORDER BY ?1)"
            )
        );
    }
}

// 在配置中引用
// hibernate.metadata_builder_contributor=com.example.CustomFunctionContributor
```

**使用自定义函数：**

```java
// 注册后可在 HQL 中使用
List<Object[]> result = session.createQuery(
    "SELECT d.name, group_concat(e.name), date_diff(d.foundedDate, CURRENT_DATE) " +
    "FROM Department d JOIN d.employees e GROUP BY d.name, d.foundedDate"
).getResultList();
```

# 13. 常见错误与调试

## 13.1 实体名大小写

```java
// 默认实体名 = 类名（大小写敏感）
@Entity
public class UserAccount { ... }
// HQL：FROM UserAccount（必须与类名完全一致，包括大小写）

// 自定义实体名
@Entity(name = "UserAcct")
public class UserAccount { ... }
// HQL：FROM UserAcct（使用自定义名称）

// 常见错误：
// FROM userAccount          — 错误：大小写不匹配
// FROM User_Account         — 错误：非下划线风格
// FROM User                 — 错误：非类名
```

## 13.2 属性名拼写错误

```java
// 属性名必须与实体字段名一致（Java 字段名，非数据库列名）
@Entity
public class User {
    @Column(name = "user_name")
    private String username;
}

// HQL 中使用：
// SELECT u.username FROM User u  — 正确：使用属性名
// SELECT u.userName FROM User u — 错误：属性名是 username，不是 userName
// SELECT u.user_name FROM User u— 错误：属性名不是下划线风格
```

## 13.3 别名使用不当

```java
// 错误：SELECT 了别名但 WHERE 中未用别名
session.createQuery("SELECT u.username FROM User u WHERE username = 'admin'");
// 应写成：WHERE u.username = 'admin'

// 错误：不使用别名直接引用属性
session.createQuery("SELECT username FROM User");
// 应写成：SELECT u.username FROM User u

// 错误：子查询中外层别名不可见
session.createQuery("SELECT u FROM User u WHERE ... AND u IN (SELECT u2 FROM User u2 WHERE ...)");
// 子查询内部的 u2 独立于外层 u
```

## 13.4 MultipleBagFetchException

```java
// 错误原因：同时 FETCH 两个 List 类型关联（Hibernate 内部称 List 为 Bag）
session.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders JOIN FETCH u.comments"
);
// 抛出 MultipleBagFetchException

// 解决方案见 3.2.4 节
```

## 13.5 查询日志分析

**启用 SQL 日志（开发调试）：**

```properties
# application.properties（Spring Boot）
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true

# logback / log4j2 配置
# 打印 SQL 语句
logging.level.org.hibernate.SQL=DEBUG

# 打印 SQL 绑定参数值（极其重要！）
logging.level.org.hibernate.orm.jdbc.bind=TRACE

# 打印 SQL 提取结果集
logging.level.org.hibernate.orm.jdbc.extract=TRACE

# 打印统计信息
logging.level.org.hibernate.stat=DEBUG
```

**日志输出示例：**

```
-- Hibernate SQL 日志
Hibernate:
    select
        u1_0.id,
        u1_0.username,
        u1_0.age,
        u1_0.status
    from
        t_user u1_0
    where
        u1_0.age>?
        and u1_0.status in (?,?,?)

-- 绑定参数日志
2024-01-15 10:30:45 TRACE o.h.o.j.bind - binding parameter [1] as [INTEGER] - [18]
2024-01-15 10:30:45 TRACE o.h.o.j.bind - binding parameter [2] as [VARCHAR] - [ACTIVE]
2024-01-15 10:30:45 TRACE o.h.o.j.bind - binding parameter [3] as [VARCHAR] - [PENDING]
2024-01-15 10:30:45 TRACE o.h.o.j.bind - binding parameter [4] as [VARCHAR] - [VERIFIED]
```

**日志级别说明：**

| 日志分类 | 说明 |
|---------|------|
| `org.hibernate.SQL` | 打印生成的 SQL 语句 |
| `org.hibernate.orm.jdbc.bind` | 打印参数绑定值（诊断数据相关问题）|
| `org.hibernate.orm.jdbc.extract` | 打印结果集提取（诊断返回数据问题）|
| `org.hibernate.type.descriptor.sql` | 打印参数绑定和结果提取组合（Hibernate 5.x）|
| `org.hibernate.stat` | 打印 SessionFactory 统计信息（缓存命中率等）|
| `org.hibernate.engine.internal.StatisticalLoggingSessionEventListener` | 会话级统计 |

**Hibernate 6 新增调试特性：**

```java
// 获取当前查询对应的原生 SQL（调试用）
org.hibernate.query.Query<?> hibernateQuery = session.createQuery(
    "SELECT u FROM User u WHERE u.age > 18", User.class
);
String sql = hibernateQuery.getQueryString();           // HQL 原文
String nativeSql = hibernateQuery.unwrap(org.hibernate.query.spi.QueryImplementor.class)
    .getQueryString();                                    // 不直接提供原生 SQL

// 使用 org.hibernate.SQL 日志是最直接的方式

// 检查查询是否被缓存
// 当 hibernate.cache.use_query_cache=true 且 hint cacheable=true
// 留意统计日志中的缓存命中率
```
